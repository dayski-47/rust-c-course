# Doc 10 - Wrapping and Replacing C Incrementally

🟡 No one rewrites a large C system in one shot. The successful strategy is incremental: wrap existing C in safe Rust interfaces, test that the interfaces work, then replace the implementation piece by piece. This doc shows how to do it safely.

ironkv uses zstd for optional compression - a real-world case of calling a battle-tested C library from Rust. Beyond that, the techniques here apply to any scenario where you inherit C code and want to introduce Rust without breaking everything.

---

## The Basic FFI Mechanics

Rust's `extern "C"` block declares C functions for the linker to resolve:

```rust
// The raw, unsafe FFI binding
extern "C" {
    fn zstd_compress(
        dst: *mut u8, dst_capacity: usize,
        src: *const u8, src_size: usize,
        compression_level: i32,
    ) -> usize;  // Returns compressed size, or error code (negative)

    fn zstd_decompress(
        dst: *mut u8, dst_capacity: usize,
        src: *const u8, src_size: usize,
    ) -> usize;  // Returns decompressed size, or error code

    fn zstd_is_error(code: usize) -> u32;  // Nonzero if code is an error
    fn zstd_get_error_name(code: usize) -> *const u8;  // C string

    fn zstd_compress_bound(src_size: usize) -> usize;  // Max compressed size
    fn zstd_default_c_level() -> i32;  // Default compression level
}
```

This is the *unsafety surface*: everything in `extern "C"` is `unsafe` to call, because the compiler can't verify the C code follows Rust's safety rules. The goal is to wrap all of it so callers see only safe Rust.

---

## Writing a Safe Wrapper

The pattern: keep `unsafe` in one module, expose only safe interfaces:

```rust
/// Safe wrapper around the zstd C library.
///
/// All `unsafe` calls are encapsulated here - callers see only safe Rust.
pub mod zstd {
    use super::*;

    #[derive(Debug)]
    pub struct ZstdError(String);

    impl std::fmt::Display for ZstdError {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            write!(f, "zstd error: {}", self.0)
        }
    }

    impl std::error::Error for ZstdError {}

    fn check_error(code: usize) -> Result<usize, ZstdError> {
        // Safety: zstd_is_error is always safe to call (no preconditions)
        if unsafe { zstd_is_error(code) } != 0 {
            // Safety: zstd_get_error_name returns a valid C string for any error code
            let name_ptr = unsafe { zstd_get_error_name(code) };
            let name = unsafe { std::ffi::CStr::from_ptr(name_ptr as *const i8) }
                .to_string_lossy()
                .into_owned();
            Err(ZstdError(name))
        } else {
            Ok(code)
        }
    }

    /// Compress `data` at the given compression level.
    /// Level 0 uses zstd's default. Levels 1–22, higher = smaller but slower.
    pub fn compress(data: &[u8], level: i32) -> Result<Vec<u8>, ZstdError> {
        let level = if level == 0 {
            unsafe { zstd_default_c_level() }
        } else {
            level
        };

        // Safety: zstd_compress_bound has no preconditions
        let max_size = unsafe { zstd_compress_bound(data.len()) };
        let mut output = vec![0u8; max_size];

        // Safety: output and data are valid byte slices for their lengths.
        // zstd_compress reads src_size bytes from src and writes at most
        // dst_capacity bytes to dst. Both pointers are non-null and valid.
        let compressed_size = unsafe {
            zstd_compress(
                output.as_mut_ptr(),
                output.len(),
                data.as_ptr(),
                data.len(),
                level,
            )
        };

        check_error(compressed_size)?;
        output.truncate(compressed_size);
        Ok(output)
    }

    /// Decompress `data` to at most `max_output_size` bytes.
    pub fn decompress(data: &[u8], max_output_size: usize) -> Result<Vec<u8>, ZstdError> {
        let mut output = vec![0u8; max_output_size];

        // Safety: same argument as compress - valid byte slices, correct sizes.
        let decompressed_size = unsafe {
            zstd_decompress(
                output.as_mut_ptr(),
                output.len(),
                data.as_ptr(),
                data.len(),
            )
        };

        check_error(decompressed_size)?;
        output.truncate(decompressed_size);
        Ok(output)
    }
}
```

The `// Safety:` comment on every `unsafe` block is a project convention: it documents the invariants the programmer verified manually. Without it, unsafe blocks are unjustifiable - someone reading the code later has no idea whether the unsafe was carefully considered or just hacked in.

---

## The `bindgen` Tool: Automating FFI Declarations

For large C libraries with many functions, writing `extern "C"` blocks by hand is error-prone. `bindgen` generates them from the C header file:

```toml
# Cargo.toml
[build-dependencies]
bindgen = "0.69"
cc = "1"
```

```rust
// build.rs
fn main() {
    // Tell cargo to link the zstd library
    println!("cargo::rustc-link-lib=zstd");
    println!("cargo::rerun-if-changed=wrapper.h");

    // Generate bindings from the header
    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        .allowlist_function("ZSTD_.*")  // Only generate functions starting with ZSTD_
        .parse_callbacks(Box::new(bindgen::CargoCallbacks::new()))
        .generate()
        .expect("Unable to generate bindings");

    let out_path = std::path::PathBuf::from(std::env::var("OUT_DIR").unwrap());
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("Couldn't write bindings");
}
```

```c
// wrapper.h - the include file for bindgen
#include <zstd.h>
```

```rust
// src/ffi.rs - use the generated bindings
#![allow(non_upper_case_globals, non_camel_case_types, non_snake_case)]
include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

`bindgen` generates the correct `extern "C"` signatures, including the right types for platform-specific C types (`size_t`, `int32_t`, etc.). It's the right tool when you're wrapping a large C API rather than a small one.

---

## The Incremental Replacement Strategy

When you have an existing C codebase and want to introduce Rust:

**Phase 1: Create a safe Rust wrapper for the C code**

Start by calling the existing C code from Rust with a safe wrapper. Don't change the C yet. Tests verify the wrapper works identically to calling the C directly.

```rust
// Phase 1: Safe Rust wrapper that calls the original C
pub fn compress_value(data: &[u8]) -> Vec<u8> {
    // Calls the C zstd library
    zstd::compress(data, 3).expect("compression failed")
}
```

**Phase 2: Write Rust tests against the wrapper's interface**

The tests test behavior, not implementation:

```rust
#[test]
fn compress_roundtrip() {
    let original = b"hello world! ".repeat(100);
    let compressed = compress_value(&original);
    let decompressed = decompress_value(&compressed, original.len() * 2).unwrap();
    assert_eq!(original.as_slice(), decompressed.as_slice());
}

#[test]
fn compress_achieves_reasonable_ratio() {
    let compressible = b"aaaa".repeat(1000);
    let compressed = compress_value(&compressible);
    assert!(compressed.len() < compressible.len() / 2, "compression ratio too low");
}
```

**Phase 3: Replace the C implementation with Rust (optional)**

Once tests exist, replace the C call with a pure Rust implementation that passes the same tests:

```rust
// Phase 3: Pure Rust implementation (using the `zstd` Rust crate)
pub fn compress_value(data: &[u8]) -> Vec<u8> {
    zstd::encode_all(data, 3).expect("compression failed")
}
// Same tests pass - behavior is identical, implementation has changed
```

The tests from Phase 2 are the regression barrier. As long as they pass, the replacement is safe.

---

## CString and CStr: Crossing the String Boundary

C strings are null-terminated and use raw pointers. Rust's `CString` and `CStr` bridge the gap:

```rust
use std::ffi::{CString, CStr};
use std::os::raw::c_char;

extern "C" {
    // Returns a C string that the caller must NOT free (static lifetime)
    fn ironkv_version() -> *const c_char;

    // Takes a file path as a C string
    fn open_legacy_db(path: *const c_char) -> *mut core::ffi::c_void;
    fn close_legacy_db(handle: *mut core::ffi::c_void);
}

/// Get the version string from the C library.
pub fn version() -> &'static str {
    // Safety: ironkv_version() returns a valid, static, null-terminated C string.
    // The 'static lifetime is correct because the string lives for the program's duration.
    let ptr = unsafe { ironkv_version() };
    let c_str = unsafe { CStr::from_ptr(ptr) };
    c_str.to_str().expect("version string is not valid UTF-8")
}

/// Open a database, passing the path as a C string.
pub fn open_legacy(path: &str) -> *mut core::ffi::c_void {
    // CString adds the null terminator; panics if path contains interior nulls
    let c_path = CString::new(path).expect("path contains null bytes");

    // Safety: c_path is valid and null-terminated. open_legacy_db is a C function
    // that reads the path string but does not retain a pointer to it after returning.
    unsafe { open_legacy_db(c_path.as_ptr()) }
    // c_path is dropped here - the C function has already copied the path
}
```

**Critical rule:** Never hold onto a `*const c_char` after the `CString` that provided it is dropped. The `CString` owns the allocation; when it drops, the pointer becomes dangling. If the C function needs to retain the string, clone it on the C side.

---

## RAII Wrappers for C Resources

C code that uses `open`/`close` pairs maps naturally to Rust's RAII:

```rust
extern "C" {
    fn legacy_db_open(path: *const i8) -> *mut LegacyDb;
    fn legacy_db_close(db: *mut LegacyDb);
    fn legacy_db_get(db: *mut LegacyDb, key: *const u8, key_len: usize,
                     value_out: *mut *mut u8, value_len_out: *mut usize) -> i32;
    fn legacy_db_free_value(value: *mut u8);
}

pub struct LegacyDatabase {
    handle: *mut LegacyDb,
}

impl LegacyDatabase {
    pub fn open(path: &str) -> Result<Self, std::io::Error> {
        let c_path = std::ffi::CString::new(path).unwrap();
        // Safety: legacy_db_open creates a new DB object; returns null on failure
        let handle = unsafe { legacy_db_open(c_path.as_ptr()) };
        if handle.is_null() {
            return Err(std::io::Error::new(
                std::io::ErrorKind::Other,
                "Failed to open legacy database",
            ));
        }
        Ok(LegacyDatabase { handle })
    }

    pub fn get(&self, key: &[u8]) -> Option<Vec<u8>> {
        let mut value_ptr: *mut u8 = std::ptr::null_mut();
        let mut value_len: usize = 0;

        // Safety: handle is non-null (checked in open). key is valid for key.len().
        // The C function writes a pointer and length if found, or null if not found.
        let found = unsafe {
            legacy_db_get(
                self.handle,
                key.as_ptr(), key.len(),
                &mut value_ptr, &mut value_len,
            )
        };

        if found == 0 || value_ptr.is_null() {
            return None;
        }

        // Safety: value_ptr is a valid allocation of value_len bytes, returned by C.
        // We copy the bytes into a Vec, then free the C allocation.
        let value = unsafe {
            let slice = std::slice::from_raw_parts(value_ptr, value_len);
            let owned = slice.to_vec();  // Copy bytes into Rust-owned memory
            legacy_db_free_value(value_ptr);  // Free the C allocation
            owned
        };

        Some(value)
    }
}

impl Drop for LegacyDatabase {
    fn drop(&mut self) {
        // Safety: handle is non-null and not yet closed.
        // Drop is called exactly once (Rust guarantees this).
        unsafe { legacy_db_close(self.handle) };
    }
}
```

The RAII wrapper guarantees:
- `legacy_db_close` is called exactly once - no leaks, no double-free
- The handle is non-null when any method is called - methods on `LegacyDatabase` cannot be called after `drop`
- The C allocation for returned values is freed before returning - no leaks in the get path

This is the exact pattern for wrapping any C resource that has open/close lifecycle semantics.

---

## The Safety Comment Convention

Every `unsafe` block in ironkv should have a `// Safety:` comment explaining:

1. **What invariants are being assumed** - e.g., "pointer is non-null and valid for N bytes"
2. **Why those invariants hold** - e.g., "because we checked for null in the constructor"
3. **What would go wrong if they didn't** - e.g., "null dereference / UB"

This isn't bureaucracy. These comments are the audit trail for future maintainers who need to verify that the unsafe code is correct. Without them, every `unsafe` block is a mystery that requires re-deriving the proof from scratch.

---

## When to Use the `zstd` Crate vs Raw FFI

For ironkv's compression feature, use the [`zstd` crate](https://crates.io/crates/zstd) rather than raw FFI:

```toml
[dependencies]
zstd = { version = "0.13", optional = true }
```

```rust
#[cfg(feature = "compression")]
pub fn compress(data: &[u8]) -> Vec<u8> {
    zstd::encode_all(data, 3).expect("zstd compression failed")
}
```

The `zstd` crate:
- Handles the FFI binding, including `bindgen`-generated signatures
- Provides a safe Rust API
- Is tested by thousands of users
- Gets security updates when the underlying C library does

Raw FFI is appropriate when no good binding crate exists, or when the binding crate doesn't expose the API shape you need. For zstd (and most well-known C libraries), the crate is almost always the right choice.

The raw FFI examples in this doc exist to teach the mechanics. In production ironkv code, use the crate.

---

## Exercises

**Exercise 1 - Write the Safe zstd Wrapper**

Using the raw `extern "C"` declarations from this doc, implement the `zstd::compress()`
and `zstd::decompress()` functions with proper `// Safety:` comments. Then write a
roundtrip test:

```rust
#[test]
fn compress_decompress_roundtrip() {
    let original = b"hello, world! ".repeat(1000);
    let compressed = zstd::compress(&original, 3).unwrap();
    assert!(compressed.len() < original.len(), "should be smaller");
    let decompressed = zstd::decompress(&compressed, original.len() * 2).unwrap();
    assert_eq!(original.as_slice(), decompressed.as_slice());
}
```

Run under Miri: `cargo +nightly miri test compress_decompress_roundtrip`. If Miri
complains about foreign functions (it can't execute C code), use `MIRIFLAGS="-Zmiri-disable-isolation"`.

**Exercise 2 - RAII Wrapper**

Implement the `LegacyDatabase` RAII wrapper from this doc. Write a test that:
1. Opens a `LegacyDatabase`
2. Calls `get()` for a key that doesn't exist - asserts `None` is returned
3. Drops the `LegacyDatabase` (let it go out of scope)
4. Verifies (with a counter or log) that `legacy_db_close` was called exactly once

For the test database, use a mock C library implemented in Rust using `#[no_mangle]`
extern functions that store state in a global `Mutex<HashMap>`.

**Exercise 3 - Incremental Replacement**

Implement the three-phase replacement for ironkv's compression:

- **Phase 1**: `compress_value(data: &[u8]) -> Vec<u8>` calls the raw C zstd binding
- **Phase 2**: Write roundtrip tests for `compress_value`
- **Phase 3**: Replace the C binding with `zstd::encode_all(data, 3)?` using the Rust crate

Verify that the Phase 2 tests still pass after Phase 3. This is the value of the
interface abstraction: you replaced the implementation without changing the tests.
Run `cargo bench` before and after Phase 3 to verify no performance regression.
