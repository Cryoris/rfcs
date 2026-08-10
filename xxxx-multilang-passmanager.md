# Qiskit pass manager in rust

| **Status**        | **Proposed/Accepted/Deprecated** |
|:------------------|:---------------------------------------------|
| **RFC #**         | ####                                         |
| **Authors**       | Julien Gacon (jul@zurich.ibm.com)            |
| **Submitted**     | YYYY-MM-DD                                   |
| **Updated**       | YYYY-MM-DD                                   |

## Summary

We will implement a Qiskit pass manager in Rust, which is exposed to both the Python and C APIs.
The new pass manager supersedes the existing Python pass manager in a backward compatible fashion.
It will share the same key concepts, such as a pass, task, property set and pass manager
which facilitates backwards compatibility and adoptation. Newly, the pass manager is solely 
responsible for workflow execution, multi-IR pipelines can be verified upon construction (if
type information is available), and callbacks can be registered at different entry points.

## Motivation

Compilation of a quantum computing program is a complex process. Qiskit's existing pass manager
system served the purpose for near-term compiler pipelines, which were written in Python and mainly
using a single IR, the `DAGCircuit`, with a single builder-interface, the `QuantumCircuit`.
Even though the pass manager infrastructure is written in Python, the compiler is highly performant 
as the passes themselves, where the work is, are written in Rust.

Nevertheless, there are two main reasons to write a new pass manager in Rust:

1. Qiskit's C API requires a pass manager. For compatibility with Python extensions and for reasons
   of consistency and code maintainability, both the Python and C pass manager should be backed by
   the same Rust datastructure.

   This will also enable writing compiler passes in other languages, such as third party Rust 
   (frequently requested), by means of a C FFI.

2. Several compiler workflows are arising that use multiple IRs, most notably in the fault-tolerant
   compiler frameworks. Python does not provide adequate tooling to ensure the pipelines we are
   building have correct types and lowerings. Rust, on the other hand, does.

   This becomes especially important as we are developing a fault-tolerant compiler pipeline
   in Qiskit itself. The Rust pass manager enables us to build the entire pipeline directly in 
   Rust, with compatibility checks.

## User benefit

- The C API will get a pass manager. 

- By the C FFI, passes and pass managers can also be written 
  in other languages, such as third party Rust code. Users will also be able to write passes for their
  Python compiler frameworks using compiled Python extensions.

- In the long run, users will benefit from the inherent type checks in our predefined, mutable 
  pipelines, since compatibility errors are more easily detected and helpful errors can be raised.
  The type checks for custom passes is _very_ limited (in Python) or non-existing (in C).


## Design proposal

The core requirements for the pass manager are:

- Enable transformations, analyses and lowerings on multiple IRs
- Support of flow control, such as conditionals, or loops
- Convenient setup using preset pipelines that are mutable and pluggable
- Status information via callbacks
- An analysis manager for context-specific information for executing passes
- Implementing passes in Python and C (and other languages by extension)
- Backward compatibility with Qiskit's existing system 

We propose these four main concepts and responsibilities:

1. `Pass`: An atomic unit of work. A pass is some form of atomic transformation on the input IR, such as 
   a specific optimization or a lowering. This what users should implement when implementing a single 
   compiler pass.
2. `Task`: A single unit of execution flow. This describes how work is executed, including non-linear
   execution such as loops or switches, or grouped execution of task, or named stages.
3. `PassManager`: Orchestrates the execution of a set of tasks. This includes tracking required 
    analysis tasks, providing a runtime context to the tasks, and passing a callback.
4. `PassContext`: Contains and provides context information. Can be thought of the equivalent
   of an analysis manager is the new LLVM pass manager.

### Passes

We define a pass as implementable trait, where the input and output IRs are set via
associated types.  We use associated types over generics since a pass is typically tied 
to an IR and does not need to be generic over it. 
```rust
/// An atomic transformation. Runs on an input IR and produces an output IR.
pub trait Pass {
    // The IRs must be static to store them in a type-erased pass,
    // see the following sections for more detail on this requirement.
    type InputIR: 'static; 
    type OutputIR: 'static;

    /// Run the pass on the IR, given the context.
    fn run(&self, ir: Self::InputIR, context: &mut PassContext) -> anyhow::Result<Self::OutputIR>;

    /// Which analysis does this preserve?
    fn preserves(&self) -> PreservedAnalysis;
}
```
We could add a `type Error` associated type and have `run` returns a
`Result<Self::OutputIR, Self::Error>` instead of using `anyhow::Result`, though for simplicity
we're starting out with the latter.

Under the hood we'll store a type-erased version of the pass, since the associated type is must be
fully specified to use the trait in a type specification.  Since the types are user-specified,
we cannot store a `dyn Pass` object in a task or pass manager, but instead need to use 
a type-erased version that does not have the associated types.  The types can instead be queried
by methods returning the `TypeId`. 
```rust
/// A type-erased version which we can call using dyn Any as input.
trait TypeErasedPass {
    fn input_ir(&self) -> TypeId;
    fn output_ir(&self) -> TypeId;
    fn run(&self, ir: Box<dyn Any>, context: &mut PassContext) -> Box<dyn Any>;
}
```
Note that `TypeId`s are only defined for `'static` types!  This implies that any IR in the 
pass manager is static.
For now, being static is the only requirement for an IR.  We are free to inject an `QiskitIR` 
trait that every IR has to implement, if the need arises.

We provide a blanket implementation for objects with the `Pass` trait
```rust
/// ... plus a blanket implementation for objects with the [Pass] trait
impl<P: Pass> AnyPass for P {
    fn input_type_id(&self)  -> TypeId { TypeId::of::<P::InputIR>() }
    fn output_type_id(&self) -> TypeId { TypeId::of::<P::OutputIR>() }
    fn preserves(&self) -> PreservedAnalysis { Pass::preserves(self) }

    fn run(&self, ir: Box<dyn Any>, context: &mut PassContext) -> anyhow::Result<Box<dyn Any>> {
        let ir = ir
            .downcast::<P::InputIR>()
            .expect("The pipeline construction guarantees the input IR has the correct type.");
        let out_ir = P::run(&self, *ir, context)?;
        Ok(Box::new(out_ir))
    }
}
```

**Questions**

- Do we need to provide a `requires()` method that allows listing passes that must run before this,
  or can we leave this in Python space? Probably we need this here.

### Tasks

A `Task` is implemented as enum, the variants describing different execution units.
```rust
/// An execution unit.
pub enum Task {
    /// A single pass.
    Transformation(Box<dyn AnyPass>), // here we need the type erasure -- <dyn Pass> is not enough!

    /// A task group. Used to conceptually group and to execute a sequence in conditionals.
    Group(Vec<Task>),

    /// A sequence of named tasks.
    Stages(Vec<(String, Task)>),

    /// A conditional execution of a task.
    /// Takes a switch function, that takes the type-erased IR and the pass context,
    /// and returns an index to which case to run.
    Switch {
        switch: fn(&dyn Any, &PassContext) -> usize,
        cases: Vec<Task>,
    },

    /// A looped execution.
    /// Runs the body until the condition function returns false.
    Loop {
        condition: fn(&dyn Any, &PassContext) -> bool,
        body: Box<Task>, // must be boxed to define Task's size
    },
}

impl Task {
    /// Return the input IR and output IR type of the task.
    /// Needed to check type compatibility.
    /// 
    /// # Returns
    /// 
    /// * Ok((TypeId, TypeId)) - The type IDs of a non-empty task.
    /// * Err(PassManagerError) - An error if the task is empty.
    fn io_types(&self) -> Result<(TypeId, TypeId), PassManagerError> { ... }
}
```
There's a design question here on allowing empty `Task`s, such as
```rust
let empty = Task::Group(vec![]);
```
This can come up naturally e.g. if a user removes tasks from a group, but empty tasks 
make checking the types for compatibility cumbersome.  If in the future we would like to enable
workflows where tasks are temporarily empty, but populated later on, then we can enable empty
tasks at a more complex type check cost.

**Questions**

- Python's flow controllers only take the property set as input for controllers, such as 
  the while-loop or conditionals, not the IR itself. Do we want to keep this, or add the IR?

### Pass manager

The pass manager is in charge of executing the tasks.
```rust
pub struct PassManager {
    tasks: Vec<Task>,
}

/// A callback registry.
///
/// Callbacks have different hook points: After pass, after task and after stages.
/// Since we want the callback to handle any IR that we're going through in the compiler
/// pipeline, we're using type-erased `dyn Any` which needs to be downcast by the implementer.
pub struct CallbackRegistry {
    post_pass: Vec<Box<dyn Fn(&dyn Any, &PassContext) -> ()>>,
    post_task: Vec<Box<dyn Fn(&dyn Any, &PassContext) -> ()>>,
    post_stage: Vec<Box<dyn Fn(&dyn Any, &PassContext) -> ()>>,
}

impl PassManager {
    pub fn new() -> Self { Self { tasks: vec![] } }

    pub fn run<IRIn, IROut>(
        &self, ir: IRIn, 
        callback: Option<&CallbackRegistry>
    ) -> Result<IROut, PassManagerError>
    where
        IRIn: 'static,
        IROut: 'static,
    {
        if self.tasks.len() == 0 {
            // If there are no tasks, return the input, but cast to IROut
            return match Box::new(ir).downcast::<OutType>() {
                Ok(out) => return Ok(*out),
                Err(_) => Err(PassManagerError::FailedOutputConversion),
            }
        }

        let (first, _) = self.tasks[0].io_types()?;
        if first != TypeId::of::<IROut>() {
            return Err(PassManagerError::IncompatibleTypes);
        }

        // Erase the type to pass it through the task execution
        let mut ir: Box<dyn Any> = Box::new(ir);

        // Main iteration loop over tasks
        let mut context = PassContext::new();
        for task in self.tasks.iter() {
            ir = execute_task(task, ir, &mut context, callback)?;
        }

        match obj.downcast::<OutType>() {
            Ok(out) => Ok(*out),
            Err(_) => Err(PassManagerError::FailedOutputConversion),
        }
    }
}

/// The task runner. This should not be called standalone, passes should be run
/// via the pass manager.
fn execute_task(
    task: &Task,
    mut ir: Box<dyn Any>,
    context: &mut PassContext,
    callbacks: Option<&CallbackRegistry>,
) -> Result<Box<dyn Any>, PassManagerError> {
    let out = match task {
        Task::Transformation(pass) => {
            let out = pass
                .run(ir, context)
                .map_err(|e| PassManagerError::PassError(e))?;
            callbacks.map(|cb| cb.call_post_pass(&out, context));
            Ok(out)
        }
        Task::Group(tasks) => {
            for task in tasks.iter() {
                ir = execute_task(task, ir, context, callbacks)?;
            }
            Ok(ir)
        }
        Task::Switch { switch, cases } => {
            let index = switch(&ir, context);
            execute_task(&cases[index], ir, context, callbacks)
        }
        Task::Loop { condition, body } => {
            while condition(&ir, &context) {
                ir = execute_task(&body, ir, context, callbacks)?;
            }
            Ok(ir)
        }
        Task::Stages(stages) => {
            for (_name, task) in stages.iter() {
                ir = execute_task(task, ir, context, callbacks)?;
                callbacks.map(|cb| cb.call_post_stage(&ir, context));
            }
            Ok(ir)
        }
    };
    callbacks.map(|cb| cb.call_post_task(&out, context));
    out
}
```

The pass manager provides functionality to modify the tasks; appending, removing, inserting, ...
Every modification must leave the pass manager in a valid state.
```rust
impl PassManager {
    /// Try push a [Task] to the pass manager. Returns an error if the types are not
    /// compatible.
    pub fn try_push_task(&mut self, task: Task) -> Result<(), PassManagerError> {
        // Check that the task types are compatible, if there's an existing task and if
        // neither of the tasks are empty.
        if let Some(last_task) = self.tasks.last() {
            let (_, out_type) = last_task.io_types()?;
            let (in_type, _) = task.io_types()?;
            if in_type != out_type {
                return Err(PassManagerError::IncompatibleTypes);
            }
        }
        self.tasks.push(task);
        Ok(())
    }

    /// ... plus some other mutation methods, like removing or injecting a task.
}
```

### Pass context

The `PassContext` contains context information provided to the passes, akin to 
the existing `PropertySet`.
```rust
/// The key under which analyses results or properties are stored.
pub struct Property(String);

pub struct PassContext {
    data: HashMap<Property, Box<dyn Any>>
}

/// Specify which analyses are preserved by a pass.
pub enum PreservedAnalysis {
    All,
    None,
    Preserves(&[Property]),  // specify which are preserved
    Invalidates(&[Property]),  // specify which are invalidated
}
```

### Extension: Analysis passes

We could add dedicated analysis passes, which only take an IR by reference and does not
produce an output.  This allows to skip type checks and is the formally correct object to
implement analyses.

This is something we likely want since it provides safety guardrails and ensure that analysis
passes cannot modify the IR, but it's not a technical requirement for the first version.

```rust
pub trait AnalysisPass {
    type InputIR;

    /// Run the pass and update the context in-place.
    pub fn run(&self, ir: &Self::InputIR, context: &mut PassContext);

    /// Report which properties this pass computes.
    pub fn properties(&self) -> Vec<Property>;
}
```

In the same fashion as for the transformation `Pass` we will have 
type-erased version and blanket implmentation
```rust
trait AnyAnalysisPass { 
    fn input_type_id(&self) -> TypeId;
    fn run(&self, ir: &dyn Any, context: &mut PassContext) -> anyhow::Result<()>;
    fn properties(&self) -> Vec<Property>;
}

impl<A: AnalysisPass> TypeErasedAnalysisPass for A {
    fn input_type_id(&self)  -> TypeId { TypeId::of::<A::InputIR>() }
    fn properties(&self) -> Vec<Property> { A::properties(self) }

    fn run(&self, ir: &dyn Any, context: &mut PassContext) -> anyhow::Result<()> {
        let ir = ir
            .downcast::<P::InputIR>()
            .expect("The pipeline construction guarantees the input IR has the correct type.");
        A::run(&self, &ir, context)
    }
}
```

The `Task` enum would gain an additional variant
```rust
pub enum Task {
    /// [ ... ]
 
    /// A single pass.
    Analysis(Box<dyn AnyAnalysisPass>),

    /// [ ... ]
}
```

### Extension: Analysis registry

In the current design, analysis passes populating the property set need to be 
scheduled manually. This is inconvenient since the requirements are tied to _passes_ rather
than _properties_ (since there could be different ways of computing the required property).

Another solution is to request a property from the pass context, akin to the 
`AnalysisManager` in LLVM's new pass manager. The pass context would contain a registry of 
analysis passes it can run to compute a required property, and would be in charge of maintaining
the cached data.

```rust
pub struct PassContext {
    data: HashMap<Property, Box<Any>> 

    /// The analysis passes the context knows about.
    registry: HashMap<Property, Box<dyn AnyAnalysisPass>>
}

impl PassContext {
    /// Register an analysis we can run.
    pub fn register(&mut self, analysis: Box<dyn AnyAnalysisPass>) {
        for property in analysis.properties() {
            self.registry.insert(property, analysis);
        }
    }

    /// Update the cache given the preserved analyses.
    pub fn update(&mut self, preserved: &PreservedAnalysis) { ... }

    /// Request an analysis. 
    /// 
    /// Try fetch from the cache, else try compute using the registry, else we cannot do it.
    pub fn request(&mut self, property: &Property) -> Result<Box<dyn Any>, PassContextError> {
        ...
    }

    /// Manually add something to the cache.
    pub fn insert(&mut self, key: &AnalysisKey, value: Box<dyn Any>) {
        self.data.insert(key, value);
    }
}
```

This is a feature Qiskit's pass manager system currently does not have and as such is not
a technical requirement. If there's concrete evidence that this improves performance or quality,
we can add it.

### C API

In C, we will rely on function pointers to implement custom passes and `void`-pointers for 
IRs. We have no way of verifying that the IRs are compatible and it is entirely up to the user to
ensure the pipeline is set up correctly. Inconsistent pipelines will probably panic.

The pass manager, the pass context and the callback registry are structs and will be exposed
to C as opaque pointers. We will provide functions to interact with these objects, e.g.
```c
QkPassManager *qk_passmanager_new(void);  // create a new empty PM
QkCallbackRegistry *qk_passmanager_new_callback(void);  // create a new empty CB registry
void *qk_passmanager_run(void* ir, QkCallbackRegistry* callbacks);  // run a passmanager
```

The pass trait will be exposed to C as struct containing the required methods as elements.
This "trait as struct" approach is also used in the custom gates for C and looks like this:
```c
#[repr(C)]
struct QkPass {
    // function pointer to run the pass, which takes an IR (as void* since it's generic)
    // and a pointer to a pass context
    void (*run)(void*, QkPassContext*); 

    // additional methods in the "trait", such as which analysis are preserved
    QkPreservedAnalysis *preserved_analyses(void);
}
```
when pushing a pass to the pass manager, the struct will be converted into a Rust object that 
implements the `Pass` trait
```rust
// note: all of this below is coded in the markdown, it's Rust pseudo-code
struct PassFromC {
    ptr_run: Box<dyn Fn(c_void, *mut PassContext) -> c_void>,
    ptr_preserved_analyses: Box<dyn Fn(c_void) -> *const PreservedAnalysis>,
}

impl PassFromC {
    fn from_qk_pass(pass: &QkPass) -> Self { ... }
}

impl Pass for PassFromC {
    type InputIR = c_void;
    type OutputIR = c_void;

    fn run(&self, ir: Self::InputIR, context: &mut PassContext) -> anyhow::Result<Self::OutputIR> {
        // call to self.ptr_run ...
    }

    fn preserves(&self) -> PreservedAnalysis {
        // call to self.ptr_preserved_analyses ...
    }
 }

#[unsafe(no_mangle)]
pub extern "C" fn qk_passmanager_append_pass(pm: *mut PassManager, pass: *const QkPass) -> QkExitCode {
    let pm = unsafe { mut_ptr_as_ref(pm) };
    let pass = unsafe { const_ptr_as_ref(pass) };
    let pushable_pass = PassFromC::from_qk_pass(pass);
    pm.try_append_pass(pushable_pass).expect("Failed appending pass.");
}
```
Importantly the lifetime of the Rust pass is bound to the C pass, since we store function
pointers to the method. It is undefined behavior if the C pass is freed before running the 
pass manager.

The task will be exposed as opaque pointer with various functions to construct the variants.
Again, the lifetime will be tied to the C objects it is storing references to. 

### Python API and backwards compatibility

_UNDER CONSTRUCTION: This part is yet to be completed._ 

The Python API is stable and must be backward compatible.  Since some of the class features 
are not supported by PyO3 (Python's `typing.Generic`) we will keep the existing Python 
object in place, where necessary, and simply inject the Rust-backed PyO3 type via composition.

The high-level migration plan is
* `GenericPass` maintains a PyO3 `PyPass` under the hood to keep the `typing.Generic`s working.
  The `PyPass` is what will be stored Rust-side.
* **tbd** `PassManager` could be completely done via PyO3, if it was not for the parallelization.
  How should we approach this? We could turn parallelization off, or keep the Python runner.
* **tbd** `StagedPassManager` is effectively a single `Task::Staged`. But depends on the above.
* `qiskit.passmanager.Task` is an interface we no longer require, we will leave this unchanged.
  We will provide specific objects for flow-control tasks other than a single pass, e.g. a `PyLoopTask`, 
  `PyWhileTask`, `PyStagedTask`, etc. that can be added to a pass manager.
* Python's flow controllers can be implemented using the above flow-control tasks.

