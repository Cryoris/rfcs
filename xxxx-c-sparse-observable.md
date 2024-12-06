# Design considerations

* All memory is owned and administered by Rust. 
* Users operate on the observable only via functions (the ``SparseObservable`` struct itself is not exposed).
* Crate management: see discussion below.

# Crate management 

There are multiple options for organizing the Qiskit crates:

#### Clean split

In this scenario, we have a Rust-only (i.e. no C or Python specific code) crate containing the core functionality. Python and C specific code is in separate crates.
This structure is
```
crates/
  core/  // future objects we expose to C also move here
    src/
      sparse_observable.rs  // contains no pyo3, no libc, only Rust
      ... 
    Cargo.toml  // no pyo3 or qiskit-accelerate or qiskit-circuit dependencies
  c_ext/  // C API
    src/
      sparse_observable.rs  // function definitions for C
      lib.rs  // load mods
    Cargo.toml  // depends on cbinding and qiskit-core
  pyext/  // Python API
    src/  // in future, move Python-specific code here
      lib.rs  // loads mods and expose to Python
      sparse_observable.rs  // Python interface for SparseObservable
    Cargo.toml // depends on pyo3 and qiskit-core (and currenty also qiskit-accelerate)
  accelerate/* // start moving things out of here into core and pyext
  circuit/* // same; move things
```
Advantages:
* clean split; the core rust functionality is not mixed with Python and PyO3
* we can build the C library without dependencies on `libpython`
* we would like this structure in the end anyways?

Disadvantages:
* more effort?

#### Link `libpython`

Here we keep the Rust core code in the current crates (`accelerate` and `circuit`) and only have to ensure that the objects we use don't call Python.
To compile, we have to link `libpython`, otherwise not all symbols are defined.

Advantages:
* possibly less coding effort?

Disadvantages: invert all statments of the "clean split" advantages.

#### Suggestion

We would prefer the "clean split" as it promises to avoid re-engineering what we're building now. 
However, we might be missing something and this path could maybe be hard to implement for some objects?

# Interface

_Disclaimer: **all** names are up for discussion._

* Types
  * the core type ``SparseObservable`` is only defined Rust-side
  * ``CTerm`` represents Rust's ``SparseTerm``
  * Rust's ``Complex64`` is mapped to ``double complex``
  * ``BitTerm`` _should_ be natively usable since it is `repr(u8)`
* Construction
  * ``obs_zero(uint32) -> SparseObservable*``
  * ``obs_from_label(char*) -> SparseObservable*``
  * ``obs_add_term(SparseObservable*, CTerm*)``
* Deconstructions
  * ``obs_free(SparseObservable*)``
  * ``obs_term_free(CTerm*)`` 
* Arithmetic
  * ``obs_add(SparseObservable*, SparseObservable*) -> SparseObservable*``
  * ``obs_mul(SparseObservable*, c_complex_double*) -> SparseObservable*``
  * ``obs_iadd(SparseObservable* self, SparseObservable* other)``
  * ``obs_imul(SparseObservable* self, c_complex_double* value)``
* Data access
  * ``obs_get_term(SparseObservable*, usize*) -> CTerm*``
  * ``obs_num_terms(SparseObservable*) -> usize`` 

This allows iterating C-side as 
```c
SparseObservable* obs = // your observable ...
for (int i = 0; i < obs_num_terms(obs); i++) {
    CTerm* term = obs_get_term(obs, i);
    // do something with the term

    // free the memory, because Term was constructed ad-hoc
    // remember that we don't store the SparseObservable as list of Terms!
    obs_term_free(term);  
}
```
 
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

## Memory handling

We expose functions to C that build an object Rust-side and return a pointer to the memory. This constructor function
on Rust side is

```rust
#[no_mangle]  // <-- tell Rust compiler not to mangle the function name, so C can find it
#[cfg(feature = "cbinding")]  // <-- use cbindgen to generate the C header
pub extern "C" fn obs_zero(num_qubits: u32) -> mut* SparseObservable {
    let obs = SparseObservable::zero(num_qubits);  // build object
    Box::into_raw(Box::new(obs))  // return pointer 
}
```

which allows to build the object C side as (the ``SparseObservable`` type is defined as opaque C struct)
```c
SparseObservable* obs = obs_zero(10);
```

Since we're returning a pointer to allocated memory, we also have to ensure the memory is freed. 
It's the responsibility of the C code to call the corresponding destructor, 
```c
obs_free(obs);
```
which is implemented Rust-side as 
```rust
#[no_mangle]  // <-- tell Rust compiler not to mangle the function name, so C can find it
#[cfg(feature = "cbinding")]  // <-- use cbindgen to generate the C header
pub extern "C" fn obs_zero(obs: &mut SparseObservable) {
    unsafe {  // reading memory from a random pointer is unsafe
        let _ = Box::from_raw(obs);  // read the memory and let the result go out of scope
    }
}
```




# Packaging

In C/C++/Fortran world there is no unified package management solution.
Thus, it is customary for developers to obtain the libraries which they require manually.

The pattern we will follow is to instruct users as follows:
1. obtain the source from Github
2. build the dynamic or static library using the rust compilation toolchain
3. when building their dependent project, they link against the library and header file

Note: we may choose to encapsulate the entire C API in its own crate.

# Open questions

* error handling (Current status: `panic!`)
* serialization: do we have a serialization in Rust we can expose? (Current status: serialization is up to the user)
* should the header file generated by cbindgen be checked into git?
