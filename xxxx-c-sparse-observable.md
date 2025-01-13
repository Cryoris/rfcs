# C API for ``SparseObservable``

Goal: Construct, modify, and operate on ``SparseObservable``s from C.

## Design considerations

* All memory is owned and administered by Rust. 
* Users operate on the observable only via functions (the ``SparseObservable`` instance itself is not exposed).
* We generate a single ``"qiskit.h"`` header and link to Rust's compiled binary
* Crate management: see discussion below.

## Using Rust functions to C

In a nutshell: we explicitly tell the compiler a Rust function should be exposed to C, generate a C header file to define the signatures, and link the Rust-library to the C program.

#### Exposing the Rust function

We expose functions to C that build an object Rust-side and return a pointer to the memory. This constructor function
on Rust side is

```rust
#[no_mangle]  // <-- tell Rust compiler not to mangle the function name, so C can find it
pub extern "C" fn obs_zero(num_qubits: u32) -> mut* SparseObservable {
    let obs = SparseObservable::zero(num_qubits);  // build object
    Box::into_raw(Box::new(obs))  // return pointer 
}
```
which will allow building the object C side as (the ``SparseObservable`` type is defined as opaque C struct)
```c
SparseObservable *obs = obs_zero(10);
```

Since we're returning a pointer to allocated memory, we also have to ensure the memory is freed. 
It's the responsibility of the C code to call the corresponding destructor, 
```c
obs_free(obs);
```
which is implemented Rust-side as 
```rust
#[no_mangle]  // <-- tell Rust compiler not to mangle the function name, so C can find it
pub extern "C" fn obs_zero(obs: &mut SparseObservable) {
    unsafe {  // reading memory from a random pointer is unsafe
        let _ = Box::from_raw(obs);  // read the memory and let the result go out of scope
    }
}
```

#### Generating the header

The header defines the function signatures and would look something like
```c
// file: qiskit.h
typedef struct SparseObservable SparseObservable;  // define the SparseObservable type

struct SparseObservable *obs_zero(uint32_t num_qubits);

void obs_free(SparseObservable *obs);
```

This header file can be automatically generated with the `cbinding` tool, for which we need to annotate the Rust function as
```rust
#[no_mangle]  
#[cfg(feature = "cbinding")]  // <-- use cbindgen to generate the C header
pub extern "C" fn obs_zero(num_qubits: u32) -> mut* SparseObservable { ... }
```

#### Compiling

With the header we can write a C program using these functions, e.g.
```c
// file: example.c
#include "qiskit.h"

int main() {
    SparseObservable *obs = obs_zero(100);
    obs_free(obs);
    return 0;
}
```

To compile the C program we need to first compile our Rust code (as static or dynamic lib, this can be set in `Cargo.toml`), and then link it against the C program.
For example, if we compile the Rust code as dynamic lib in release mode, the library is stored in `target/release/lib<crate_name>.dylib` and we can compile as
```
gcc example.c -l<crate_name> -L./target/release
```

## Interface

_Disclaimer: **all** names are up for discussion._

#### Types

How are Rust types represented in C? Generally, we expose pointers to Rust objects to C and C may only operate on these objects via functions we expose from Rust.
This way, the memory is owned and operated on by Rust, which provides an opaque interface to C. More concretely:

* C handles only pointers to Rust data
* ``SparseObservable`` and ``SparseTerm`` are only defined Rust-side, C get's access to ``SparseObservable*`` and ``SparseTerm*``
* Any vectors we need are defined Rust-side (since they manage memory)

Some types are compatible with C:

* Rust's ``Complex64`` is mapped to ``complex double`` and passed by reference (since they are memoery compatible but not calling convention compatible, see legacy section)
* ``BitTerm`` is natively usable since it is `repr(u8)`
* Qubit indices/number of qubits is ``uint32_t``
* Index types are ``uint64_t`` (we use sized to ensure all values have a defined size -- should this be larger?)

#### Construction

Now we delve into the syntax: we construct sparse observables with functions that return pointers. Since there are no methods, we need 
to agree upon a standard prefix to all functions acting on/returning sparse observables. Here we suggest ``obs_<function_name>``.

* ``SparseObservable* obs_zero(uint32_t num_qubits)``  (remember C syntax has the return type infront)
* ``SparseObservable* obs_identity(uint32_t num_qubits)``

We need to be able to push new terms to the observable. 
A term consists of a coefficient and the Pauli term (= the bit terms and indices, represented by a `PauliTermVec`).
To push a new term to the observable, we provide a ``copy`` and a ``consume`` variants. The latter frees the memory of the Paulis, given as vector, after adding it to the observable.
  * ``obs_push_copy(SparseObservable*, PauliTermVec*, complex double*)`` -- copy the Pauli vector
  * ``obs_push_consume(SparseObservable*, PauliTermVec*, complex double*)`` -- free the Pauli vector after adding it
      
Memory de-allocation is the user's responsibility.
* ``obs_free(SparseObservable*)``

For example:
```c
#include "qiskit.h"

SparseObservable *obs = obs_zero(num_qubits);

PauliTermVec *paulis = paulis_new(); // could use paulis_with_capacity(3) here
paulis_push(paulis, BitTerm_X, 0);  // enums explicitly include their prefix
paulis_push(paulis, BitTerm_Y, 1);  // (otherwise it would just be a global "X/Y/Z")
paulis_push(paulis, BitTerm_Z, 2);

complex double coeff = 1;
obs_push_copy(obs, paulis, &coeff);  // complex numbers are not calling convention compatible, hence pass by ref
obs_push_consume(obs, paulis, &coeff); // consumes the bits and indices vectors

obs_free(obs);  // once we're done, free the observable (remember, paulis is already freed)
```

#### Arithmetic
* ``SparseObservable* obs_add(SparseObservable *self, SparseObservable *other)``
* ``SparseObservable* obs_mul(SparseObservable *self, complex double *value)``
* ``obs_iadd(SparseObservable *self, SparseObservable *other)``
* ``obs_imul(SparseObservable *self, complex double *value)``
* ... 

#### Data access
* ``obs_term(SparseObservable*, uint64_t*) -> SparseTerm*``
* ``obs_num_terms(SparseObservable*) -> uint64_t`` 
* ``obs_num_qubits(SparseObservable*) -> uint32_t``
* ...

This allows iterating C-side as 
```c
SparseObservable *obs = // your observable ...
for (int i = 0; i < obs_num_terms(obs); i++) {
    SparseTerm *term = obs_term(obs, i);
    // do something with the term

    // free the memory, because Term was constructed ad-hoc
    // remember that we don't store the SparseObservable as list of Terms!
    obsterm_free(term);  
}
obs_free(obs);
```

## Crate management 

There are multiple options for organizing the Qiskit crates:

#### Clean split

We have 3 seperate crates for Rust, Python, and C, respectively. A Rust-only ``core`` crate contains the data structures and functions in Rust (without PyO3 dependencies), and Python/C specific code is in separate ``py_ext`` and ``c_ext`` crates. This structure could be
```
crates/
  core/  // future objects we expose to C also move here
    src/sparse_observable.rs  // contains no pyo3, no libc, only Rust
    Cargo.toml  // no pyo3 or qiskit-accelerate or qiskit-circuit dependencies
  c_ext/  // C API
    src/sparse_observable.rs  // function definitions for C
    Cargo.toml  // depends on cbinding and qiskit-core
  pyext/  // Python API: in future, move Python-specific code here
    src/sparse_observable.rs  // Python interface for SparseObservable
    Cargo.toml // depends on pyo3 and qiskit-core (and currenty also qiskit-accelerate)
  accelerate/* // start moving things out of here into core and pyext
  circuit/* // same; move things
```
Advantages:
* clean split; the core rust functionality is not mixed with Python and PyO3
* we can build the C library without dependencies on `libpython`
* we would like this structure in the end anyways?

Disadvantages:
* more effort to get to, as currently Rust and Python are very entangled

#### Minimal split

We keep the current ``accelerate`` and ``circuit`` crates and split Python/Rust inplace. The crates/files can still have PyO3 dependencies, as long as the objects we expose to C do not. 
To compile for C, we link ``libpython`` (otherwise not all symbols are defined).

Advantages:
* no new crate management

Disadvantages: `!(Advantages of clean split)`

#### Suggestion

We would prefer the "clean split" as it promises to avoid re-engineering what we're building now. 
Currently, Rust and Python are very entangled, but we could more things into ``core`` as we split them off from Python (which we want anyways as we expose them to Rust). This might also be easier if in the future we ever want to expose the Rust crate itself.

However, we might be missing something and this path could maybe be hard to implement for some objects?


## Packaging

In C/C++/Fortran world there is no unified package management solution.
Thus, it is customary for developers to obtain the libraries which they require manually.

The pattern we will follow is to instruct users as follows:
1. obtain the source from Github
2. build the dynamic or static library using the rust compilation toolchain
3. when building their dependent project, they link against the library and header file

Note: we may choose to encapsulate the entire C API in its own crate.

## C API Documentation

So far, we (obviously) only had Python API documentation to deal with.
This is done using Sphinx (as we all know) which automatically extracts the docs from the source code.
Some pages are written directly in the RST files.

### Extract documentation automatically

We need to consider the following points:
- Rust functions which get exposed as the C API (`#[cfg(feature = "cbinding")]`) are automatically gathered in a single C header file using `cbindgen`
- In doing so, docstrings of said functions will be gathered, too.
- If we want to automatically generate C API documentation from this header file, we need:
    - the docstrings in the Rust source code to conform with the C auotdoc tool
    - we need to build the C API rust crate and generate said C header file in order to build the docs
    - we need to amend our documentation toolchain with more tools (a C autodoc tool and a bridge to Sphinx)
- Currently, the above is done using [doxygen](https://www.doxygen.nl/manual/) and [breathe](https://breathe.readthedocs.io/en/stable/).

Advantages:

- documentation and source code are in one place

Disadvantages:

- more complicated documentation tooling
- C crate and header must be built in order to generate docs

### Manual C API Documentation

Alternatively, we can write the whole C API docs directly in RST pages, avoiding the more complicated toolchain.

Advantages:

- no extra tooling required

Disadvantages: `!(Advantages of autodoc)`

#### Custom `rustdoc --json` bridge to Sphinx

It was suggested by Jake to parse the `rustdoc --json` output (which is still in beta but already relied on elsewhere)
and use it to build a bridge to Sphinx "manually"/ourselves.


## Open questions

* error handling (Current status: `panic!`)
* serialization: do we have a serialization in Rust we can expose? (Current status: serialization is up to the user)
* what's the battleplan for things we want to expose to C? What are the priorities? Are there demos we want for 2.0?
* ~should the header file generated by cbindgen be checked into git?~ NO!
* ~why C and not C++?~ C is more general, e.g. has built-in compatibility with Fortran, Python, Rust, ...

-----

## Legacy topics

## Complex numbers

#### Option A

Use a custom-defined struct exposed to C, that represents complex numbers, i.e.
```rust
#[repr(C)]
struct c_complex_double([f64; 2]);
```

Disadvantage: this is not a native C type and C users will have to use Qiskit's custom type here.

#### Option B (current solution)

Use ``Complex64`` directly, which is natively C-compatible as it is defined with ``repr(C)``.
However, cbindgen does not recognize this type by default.
Thus, we have to configure cbindgen to include a type alias in its ``after_includes`` setting.
```c
typedef float complex Complex32;
typedef double complex Complex64;
```

Advantage: C users can use their established types.

Disadvantage (but not really): We have to customize cbindgen.

Sharp bit: Complex numbers are not guaranteed to be calling convention compatible and must be passed by pointers, not by values.
