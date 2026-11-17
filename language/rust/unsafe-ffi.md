# Rust Unsafe Code And FFI

## Policy

- Default to `#![forbid(unsafe_code)]` (or `[lints.rust] unsafe_code = "forbid"`). Relax it only in the small module or crate that needs it.
- Use `unsafe` only when safe Rust cannot express the operation: FFI, raw pointers, implementing `Send`/`Sync`, intrinsics, or a measured performance need that a safe crate cannot meet.
- Prefer a well-audited crate (`bytemuck`, `zerocopy`, `memmap2`, `crossbeam`) over writing your own `unsafe`.
- Wrap every `unsafe` operation in a **safe** API that makes misuse impossible. The `unsafe` block must be small and its justification local.

## Document Every Use

- Every `unsafe` block gets a `// SAFETY:` comment stating which invariants hold and why.
- Every `unsafe fn` gets a `# Safety` section in its docs listing what the caller must guarantee.
- Enable `clippy::undocumented_unsafe_blocks` and `clippy::missing_safety_doc`.

```rust
/// Returns the first byte of `bytes` without bounds checking.
///
/// # Safety
/// `bytes` must not be empty.
pub unsafe fn first_unchecked(bytes: &[u8]) -> u8 {
    // SAFETY: the caller guarantees `bytes` is non-empty, so index 0 is in bounds.
    unsafe { *bytes.get_unchecked(0) }
}

/// Safe wrapper: checks the invariant, then relies on it.
pub fn first(bytes: &[u8]) -> Option<u8> {
    if bytes.is_empty() {
        return None;
    }
    // SAFETY: we just checked that `bytes` is non-empty.
    Some(unsafe { first_unchecked(bytes) })
}

fn main() {
    assert_eq!(first(&[7, 8]), Some(7));
    assert_eq!(first(&[]), None);
}
```

## Edition 2024 Rules

- `unsafe_op_in_unsafe_fn` warns by default: even inside an `unsafe fn`, each unsafe operation needs its own `unsafe { }` block (as above).
- `extern` blocks must be written `unsafe extern`, and each item may be marked `safe` or `unsafe`.
- Attributes that can cause UB are written `#[unsafe(no_mangle)]`, `#[unsafe(export_name = "...")]`, `#[unsafe(link_section = "...")]`.
- Creating references to `static mut` is a deny-by-default error. Use atomics, `Mutex`, `OnceLock`, or `&raw const` / `&raw mut` with extreme care.
- `std::env::set_var` and `remove_var` are `unsafe` (not thread-safe on most platforms). Set environment variables before spawning threads, or pass values explicitly.

## Common Undefined Behaviour

Avoid, and test with Miri:

- Dereferencing dangling, null or misaligned pointers
- Data races; aliasing a `&mut` with any other reference
- Constructing invalid values (`bool` other than 0/1, invalid `enum` discriminant, non-UTF-8 `str`, uninitialised memory via `mem::uninitialized`)
- Violating `Send`/`Sync` contracts or `Pin` guarantees
- Calling a function through the wrong ABI or signature
- Unwinding across an FFI boundary (use `extern "C-unwind"` deliberately or `catch_unwind`)

```bash
rustup +nightly component add miri
cargo +nightly miri test            # interprets tests and reports UB
```

Run Miri in CI for any crate containing `unsafe`. For concurrency code also use `loom` (model checker) and thread sanitizer (`RUSTFLAGS=-Zsanitizer=thread` on nightly).

## FFI: Calling C From Rust

Generate bindings with `bindgen` rather than writing them by hand. Keep raw bindings in a `*-sys` crate or private module and expose a safe wrapper.

```rust
use std::ffi::{CStr, CString, c_char, c_int};

unsafe extern "C" {
    // Mark functions `safe` only when any argument values are sound
    pub safe fn abs(input: c_int) -> c_int;
    fn strlen(s: *const c_char) -> usize;
}

pub fn c_string_len(text: &str) -> Result<usize, std::ffi::NulError> {
    let c = CString::new(text)?;            // appends the NUL, rejects interior NULs
    // SAFETY: `c` is a valid NUL-terminated string that outlives the call.
    Ok(unsafe { strlen(c.as_ptr()) })
}

pub fn from_c(ptr: *const c_char) -> Option<String> {
    if ptr.is_null() {
        return None;
    }
    // SAFETY: caller contract: `ptr` points to a valid NUL-terminated string.
    let s = unsafe { CStr::from_ptr(ptr) };
    s.to_str().ok().map(str::to_owned)
}
```

- Use `#[repr(C)]` for structs crossing the boundary; `#[repr(transparent)]` for newtypes over a single field; `#[repr(u8)]` etc. for C-like enums.
- Own memory on one side only. Memory allocated by C is freed by C's `free`/API; memory from Rust's allocator is freed by Rust. Provide explicit `create`/`destroy` functions.
- Wrap handles in a struct implementing `Drop` so resources are released on every path.
- Translate error codes and `errno` into `Result` at the boundary.

## FFI: Exposing Rust To C

```rust,ignore
// Cargo.toml: [lib] crate-type = ["cdylib", "staticlib"]

#[unsafe(no_mangle)]
pub extern "C" fn counter_new() -> *mut Counter {
    Box::into_raw(Box::new(Counter::default()))
}

/// # Safety
/// `ptr` must come from `counter_new` and must not be used afterwards.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn counter_free(ptr: *mut Counter) {
    if !ptr.is_null() {
        // SAFETY: per the contract above, `ptr` is a live Box from `counter_new`.
        drop(unsafe { Box::from_raw(ptr) });
    }
}
```

- Generate the C header with `cbindgen`.
- Never let a panic unwind into C: wrap entry points in `std::panic::catch_unwind` and return an error code.
- Do not pass Rust types with unspecified layout (`String`, `Vec`, `&str`, trait objects) across the boundary; pass pointer + length pairs.

## Concurrency Primitives

- `unsafe impl Send for T {}` / `Sync` needs a written proof. Prefer composing safe types (`Arc<Mutex<_>>`, atomics).
- Atomics: default to `Ordering::SeqCst`; weaken to `Acquire`/`Release`/`Relaxed` only with a documented reason and a `loom` test.

## Quick Checklist

- [ ] `unsafe_code = "forbid"` everywhere except named, minimal modules
- [ ] Each `unsafe` block has `// SAFETY:`; each `unsafe fn` has `# Safety` docs
- [ ] Unsafe is hidden behind a safe, hard-to-misuse API
- [ ] Miri (and loom for concurrency) run in CI
- [ ] FFI uses `#[repr(C)]`, `CString`/`CStr`, explicit ownership, and `catch_unwind`
