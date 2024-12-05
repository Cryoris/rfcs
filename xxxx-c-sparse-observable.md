# Design considerations

* All memory is owned and administered by Rust. 
* Users operate on the observable only via functions (there's no struct exposed).

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

Use ``Complex64`` directly, which is natively C-compatible as it is defined with ``repr(C)``. However, cbindgen doesn't recognize this type 
so we manually have to update the generated header file to map ``Complex64`` to ``double complex``.

Advantage: C users can use their established types.

Disadvantage (but not really): We have to customize cbindgen. @Max will check with the maintainers if the mapping to ``double complex`` can be added.

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

Options
1. Users have to compile from source. We provide a Makefile (which might be very short).
2. We ship the C header and the pre-compiled library for a set of architectures.

# Open questions

* error handling (Current status: `panic!`)
* serialization: do we have a serialization in Rust we can expose? (Current status: serialization is up to the user)
