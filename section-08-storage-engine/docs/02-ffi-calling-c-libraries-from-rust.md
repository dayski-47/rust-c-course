# 02 — FFI: Calling C Libraries from Rust

> **Difficulty:** 🟡 think about it  
> **You'll learn:** `extern "C"` blocks, C types in Rust, `CString`/`CStr` for string interop,
> `#[repr(C)]` for struct layout, bindgen for real projects, and complete ownership
> semantics across the FFI boundary.

---

## This Is Your Home Territory

You come from C. FFI is where your existing knowledge pays off most directly.
You understand calling conventions, null-terminated strings, `int` return codes,
and who owns what memory. Rust gives you a structured way to express all of that —
plus it enforces the rules at compile time where possible.

The core truth: **you can call any C library from Rust**. The entire POSIX API,
every compression library, every database driver, every OS-level interface —
all accessible through FFI. This is how Rust becomes a systems language in practice.

---

## The `extern "C"` Block

Declaring a C function in Rust looks like this:

```rust
use std::ffi::{c_int, c_uint, c_char, c_size_t};

extern "C" {
    // strlen from libc
    fn strlen(s: *const c_char) -> c_size_t;

    // open() from POSIX
    fn open(path: *const c_char, flags: c_int, ...) -> c_int;

    // A custom C function from your own library
    fn process_buffer(buf: *const u8, len: c_size_t) -> c_int;
}
```

The `"C"` in `extern "C"` is the ABI (Application Binary Interface). It specifies:
- Arguments are passed according to the C calling convention for your platform
- Names are not mangled (so the linker can find them by the exact name you wrote)
- Return values follow C conventions

Rust has its own calling convention for internal functions — it is unstable and
intentionally not documented. **Never use Rust-to-Rust FFI without `extern "C"`.**
The ABI could change between compiler versions.

---

## C Types: Use the Right Names

Rust's `i32` and C's `int` are usually the same on 64-bit Linux. But "usually" is
not a specification. Use the correct FFI types from `std::ffi`:

```rust
use std::ffi::{
    c_char,    // char
    c_int,     // int
    c_uint,    // unsigned int
    c_long,    // long
    c_ulong,   // unsigned long
    c_size_t,  // size_t
    c_void,    // void (for opaque pointer targets: *mut c_void)
    c_double,  // double
    c_float,   // float
};
```

These types are defined to match the platform's C ABI. They exist precisely because
`int` is 16 bits on some embedded platforms, 32 bits on most others.

For functions that expect fixed-width integers (`uint32_t`, `int64_t`), use
Rust's `u32`, `i64` etc. directly — those are always the right width.

---

## C Strings: `CString` and `CStr`

The fundamental mismatch: Rust strings are UTF-8 byte slices with a stored length.
C strings are null-terminated byte sequences. They cannot be passed to each other directly.

```rust
use std::ffi::{CString, CStr, c_char};

fn demonstrate_c_strings() {
    // CString: Rust-owned, null-terminated. Use to pass a string TO C.
    let message = CString::new("Hello from Rust").expect("interior null byte");
    // message.as_ptr() gives *const c_char — valid for the lifetime of message
    let ptr: *const c_char = message.as_ptr();
    // SAFETY: strlen is a standard C function. ptr is a valid null-terminated
    //         string that lives as long as message.
    let len = unsafe { libc::strlen(ptr) };
    assert_eq!(len, 15);
    // DO NOT use ptr after message is dropped. It's a dangling pointer.

    // CStr: borrowed view of a C string. Use to receive a string FROM C.
    // This is common when C gives you a const char* you don't own.
    let raw_ptr: *const c_char = b"hello\0".as_ptr() as *const c_char;
    // SAFETY: raw_ptr points to a valid null-terminated string; we keep it alive
    let borrowed: &CStr = unsafe { CStr::from_ptr(raw_ptr) };
    let as_str: &str = borrowed.to_str().expect("not valid UTF-8");
    println!("{}", as_str);
}
```

A critical mistake to avoid:

```rust
// WRONG: The CString is created and immediately dropped
//        ptr is a dangling pointer when it reaches the C function
fn bad_example() -> *const c_char {
    let s = CString::new("hello").unwrap();
    s.as_ptr()  // DANGLING — s is dropped at the end of this function
}

// CORRECT: Keep CString alive for the duration of the C call
fn good_example() {
    let s = CString::new("hello").unwrap();
    let ptr = s.as_ptr();  // ptr is valid here
    // SAFETY: ptr is valid and null-terminated; s is alive
    unsafe { some_c_function(ptr); }
    // s drops here, after the call. Fine.
}
```

---

## `#[repr(C)]`: Making Structs C-Compatible

Rust is free to reorder struct fields for optimal alignment. This is great for
pure-Rust code but catastrophic when C expects a specific layout.

```rust
// WITHOUT #[repr(C)]:
// Rust might reorder fields, add hidden padding, or change the size.
// If C reads this struct, it reads garbage.
struct BadForFFI {
    flag: bool,    // C sees this at wrong offset
    count: u32,
    value: f64,
}

// WITH #[repr(C)]:
// Fields are in declaration order. Padding follows C rules.
// This is what C will see.
#[repr(C)]
struct Header {
    magic: [u8; 4],   // bytes 0-3
    version: u32,     // bytes 4-7
    entry_count: u64, // bytes 8-15
    flags: u32,       // bytes 16-19
    _padding: [u8; 4], // bytes 20-23 (explicit padding to reach 24 bytes)
}
```

Use `#[repr(C)]` for:
- Any struct that C code reads or writes directly
- Any struct that is part of a file format where byte layout matters
- File headers, network packet structures, hardware register maps

Do NOT use `#[repr(C)]` for:
- Rust-internal types that never cross an FFI boundary — you pay a size cost for no reason

There is also `#[repr(packed)]` which eliminates all padding between fields.
Useful for dense binary formats, but dangerous: accessing a `u32` field at an
unaligned offset is UB on some architectures. If you use `repr(packed)`, always
copy fields out into a local variable before using them.

---

## Function Pointers Across FFI

C APIs that take callbacks expect function pointers. The `extern "C"` ABI applies here too:

```rust
// This type matches C's: int (*callback)(const char*, int)
type CCallback = extern "C" fn(*const c_char, c_int) -> c_int;

extern "C" {
    fn register_handler(cb: CCallback) -> c_int;
}

extern "C" fn my_rust_handler(msg: *const c_char, code: c_int) -> c_int {
    // SAFETY: msg is a valid null-terminated C string from the C runtime
    let s = unsafe { CStr::from_ptr(msg) };
    println!("C called us: {:?} (code {})", s, code);
    0 // success
}

fn setup() {
    // SAFETY: my_rust_handler has the correct signature and is safe to call from C
    unsafe { register_handler(my_rust_handler); }
}
```

One rule: **never let a Rust panic unwind across an FFI boundary.** If a Rust
function called from C panics, the behavior is undefined. Wrap FFI-exposed functions
with `std::panic::catch_unwind`, or set `panic = "abort"` in your Cargo profile.

---

## Exporting Rust Functions to C

Going the other direction — letting C call Rust:

```rust
// #[no_mangle]: do not mangle the function name. C needs to find "add" exactly.
// extern "C": use the C calling convention
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

In the Cargo.toml for a library that C will link against:

```toml
[lib]
crate-type = ["staticlib"]  # produces libmylibrary.a
# or:
crate-type = ["cdylib"]     # produces libmylibrary.so / .dll
```

---

## Memory Ownership Across FFI: Document It

This is where you earn your keep. C has no ownership system. When a C function
returns a pointer, who frees it? When you pass a pointer to C, can C hold onto it?

Document ownership explicitly in your FFI wrappers:

```rust
extern "C" {
    // C allocates the return value. Caller must free with zstd_free().
    fn ZSTD_compress(dst: *mut c_void, dst_capacity: c_size_t,
                     src: *const c_void, src_size: c_size_t,
                     compression_level: c_int) -> c_size_t;

    // ZSTD_isError: pass the result of ZSTD_compress to check for error
    fn ZSTD_isError(code: c_size_t) -> c_uint;

    // Returns a string literal — do NOT free this pointer
    fn ZSTD_getErrorName(code: c_size_t) -> *const c_char;
}

/// Compress `input` using zstd at the given level.
/// Returns the compressed bytes or an error message.
fn zstd_compress(input: &[u8], level: i32) -> Result<Vec<u8>, String> {
    // Worst-case output size from zstd
    let max_output = input.len() + 128;
    let mut output: Vec<u8> = vec![0u8; max_output];

    // SAFETY: output has max_output bytes; input.as_ptr() is valid for input.len() bytes;
    //         neither slice overlaps; level is a valid compression level (1-22)
    let result = unsafe {
        ZSTD_compress(
            output.as_mut_ptr() as *mut c_void,
            max_output as c_size_t,
            input.as_ptr() as *const c_void,
            input.len() as c_size_t,
            level as c_int,
        )
    };

    // SAFETY: result is the return value of ZSTD_compress
    if unsafe { ZSTD_isError(result) } != 0 {
        // SAFETY: ZSTD_getErrorName returns a valid C string literal, no free needed
        let err_ptr = unsafe { ZSTD_getErrorName(result) };
        let err_str = unsafe { CStr::from_ptr(err_ptr) };
        return Err(err_str.to_string_lossy().into_owned());
    }

    output.truncate(result as usize);
    Ok(output)
}
```

In this section's storage engine, `zstd` compression for large values is one of
your FFI tasks. The `zstd` crate on crates.io already provides safe Rust bindings,
but the exercise asks you to understand what those bindings are doing underneath.

---

## bindgen: The Real Way to Wrap C Libraries

For any non-trivial C library, writing FFI declarations by hand is tedious and
error-prone. `bindgen` reads a C header file and generates Rust FFI declarations automatically.

```bash
cargo install bindgen-cli

# Generate bindings from a C header
bindgen /usr/include/zstd.h -o src/zstd_bindings.rs
```

For build-time generation, add `bindgen` as a build dependency and call it from `build.rs`:

```rust
// build.rs
fn main() {
    let bindings = bindgen::Builder::default()
        .header("src/zstd_wrapper.h")
        .parse_callbacks(Box::new(bindgen::CargoCallbacks::new()))
        .generate()
        .expect("Unable to generate bindings");

    bindings
        .write_to_file("src/bindings.rs")
        .expect("Couldn't write bindings");
}
```

Then in Rust:

```rust
// src/lib.rs
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
include!("bindings.rs");
```

`bindgen` handles complex types, arrays, function pointers, and nested structs.
For any C library you use in production, start with `bindgen`.

---

## Error Handling Across FFI: C Codes to Rust Results

C returns `0` for success and negative or positive non-zero for failure.
Rust uses `Result`. Your FFI wrappers should do the conversion:

```rust
use std::io;

fn check_c_result(code: c_int) -> io::Result<()> {
    if code == 0 {
        Ok(())
    } else {
        // Translate errno into a Rust I/O error
        Err(io::Error::last_os_error())
    }
}

// Usage:
// SAFETY: fd is a valid file descriptor, buf is valid for len bytes
let n = unsafe { libc::read(fd, buf.as_mut_ptr() as *mut c_void, buf.len()) };
if n < 0 {
    return Err(io::Error::last_os_error());
}
```

---

## How It Breaks

**Panic across FFI boundary.** A panic unwinding through a C stack frame is undefined behavior. Wrap the Rust code called from C (or code that calls back into Rust) with `catch_unwind`. Mark `extern "C" fn` functions that could panic as `unsafe` and document the no-panic requirement.

**Null pointer passed to C function expecting a valid pointer.** Most C functions are not documented to handle `NULL` for all parameters. Check before calling.

**C function modifies memory Rust thinks is immutable.** You pass `&T` to C, C modifies the bytes. This is aliasing UB. Only pass `&mut T` to C functions that write to the buffer.

**build.rs running on the wrong platform.** Your `build.rs` compiles `reverb.c`, but the CI runs on a different OS. The `cc` crate handles this, but you need to test on the same platform as deployment.

**Forgetting to link.** You declare `extern "C" { fn foo(); }` but don't tell the linker where to find `foo`. You get "undefined reference to foo" at link time, not compile time.

---

## Common Mistakes

**Using `String::as_ptr()` instead of `CString::as_ptr()`.** A Rust String is NOT
null-terminated. Passing `String::as_ptr()` to a C function that expects a
`const char*` will read past the end of the string into whatever memory follows.
Always use `CString`.

**Forgetting `#[repr(C)]` on shared structs.** If your struct goes over FFI without
`#[repr(C)]`, the C code reads fields at the wrong offsets. The bug may only appear
on certain platforms or compiler versions where Rust chooses a different layout.

**Creating a dangling CString pointer.** `CString::new("hello").unwrap().as_ptr()` —
the CString is created, you get the pointer, then the CString is immediately dropped.
The pointer is dangling. Assign the CString to a variable that lives long enough.

**Letting panics cross FFI boundaries.** A panic in a Rust function called from C
is undefined behavior. Wrap the body in `std::panic::catch_unwind`.

**Mismatched free/alloc across FFI.** If C allocates memory with `malloc`, you
cannot free it with Rust's allocator. You must call `free()` via FFI. And vice
versa: memory Rust allocates cannot be freed by C.

---

## Safety Invariants to Maintain

- **String lifetimes:** A `*const c_char` obtained from `CString::as_ptr()` is
  only valid while the `CString` is alive. Never store the raw pointer; always
  perform the C call while the CString is in scope.

- **Struct layout:** Any struct passed across FFI boundaries must have `#[repr(C)]`.
  Verify this in code review.

- **Null checks:** Always check incoming `*const T` pointers for null before dereferencing.
  C code can pass null; Rust references cannot be null, but raw pointers can.

- **Panic boundary:** FFI-exposed functions (`#[no_mangle] pub extern "C" fn`) must
  never allow a Rust panic to propagate. Use `catch_unwind` or configure `panic = "abort"`.

- **Ownership documentation:** Every FFI boundary must have explicit ownership docs:
  who allocates, who frees, what lifetime the pointer has.
