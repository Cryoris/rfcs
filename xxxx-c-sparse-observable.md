# Design considerations

* All memory is owned and administered by Rust.
* Users operate on the observable only via functions.

# Interface

* Types
  * the core type ``SparseObservable`` is only defined Rust-side
  * ``CTerm`` represents Rust's ``SparseTerm``
  * Rust's ``Complex64`` is mapped to ``[f64; 2]`` (what is this as C type?)
* Construction
  * ``obs_zero(uint32) -> SparseObservable*``
  * ``obs_add_term(SparseObservable*, CTerm*)``
  * ``obs_from_label(char*)``
* Arithmetic
  * ``obs_add(SparseObservable*, SparseObservable*) -> SparseObservable*``
  * ``obs_mul(SparseObservable*, c_complex_double*) -> SparseObservable*``
  * ``obs_iadd(SparseObservable* self, SparseObservable* other)``
  * ``obs_imul(SparseObservable* self, c_complex_double* value)``
* Data access
  * ``obs_terms(SparseObservable*) -> (CTerm*, size)``
  * (_in discussion_) ``obs_next_term(SparseObservable*, CTerm*) -> CTerm* | null``
 
## Complex numbers

#### Option A

Use a custom-defined struct exposed to C, that represents complex numbers, i.e.
```rust
#[repr(C)]
struct c_complex_double([f64; 2]);
```

#### Option B

Use ``Complex64`` directly, which is natively C-compatible as it is defined with ``repr(C)``. However, cbindgen doesn't recognize this type 
so we manually have to update the generated header file to map ``Complex64`` to ``complex double``.

# Packaging

Options
1. Users have to compile from source. We provide a Makefile (which might be very short).
2. We ship the C header and the pre-compiled library for a set of architectures.
