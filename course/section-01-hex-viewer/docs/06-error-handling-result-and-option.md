# Error Handling: Result and Option

🟡 Think about it - the mechanics are simple, knowing when to use each takes practice.

In C, errors come back as return codes (`-1`, `NULL`, `errno`). It's easy to ignore a return code. The compiler won't stop you. In Rust, errors are types - and the type system forces you to deal with them.

---

## No NULL, No Exceptions

Rust has no null pointers in safe code. It has no exceptions. These are features, not oversights.

**Why no null?** Because null is a value that means "I have a variable of type T but there's no T here." That ambiguity causes an enormous number of crashes. Rust replaces it with a type that makes the "might be absent" case explicit.

**Why no exceptions?** Because exceptions are hidden control flow. A C++ function that calls `new` might throw. A function that looks like it returns `int` might actually throw at any point. Rust puts errors in the return type, where they're visible in the function signature and the compiler forces you to handle them.

---

## `Option<T>`: When Something Might Not Be There

`Option<T>` replaces nullable pointers. It's an enum with two variants:

```rust
enum Option<T> {
    Some(T),   // there is a value, and it's T
    None,      // there is no value
}
```

You've already seen this if you've used vectors: `vec.get(index)` returns `Option<&T>` instead of panicking or returning null on out-of-bounds.

```rust
fn main() {
    let offsets = vec![0u64, 16, 32];

    // get() returns Option<&u64>
    match offsets.get(1) {
        Some(offset) => println!("Found offset: {offset}"),
        None => println!("Index out of bounds"),
    }

    // Compared to: if you used offsets[5], it would panic
    // Compared to C: arr[5] is undefined behavior
}
```

**Rule of thumb for `Option`:** Use it when absence is normal - when "no value" is a valid, expected outcome. Looking something up in a table. Finding a substring. Reading an optional config value.

---

## `Result<T, E>`: When Something Might Fail With an Explanation

`Result<T, E>` is for operations that either succeed or fail with an error:

```rust
enum Result<T, E> {
    Ok(T),    // success - contains the value
    Err(E),   // failure - contains the error
}
```

Most I/O in Rust returns `Result`. File opens, reads, seeks - all of them.

```rust
use std::fs::File;

fn main() {
    let result = File::open("/etc/hostname");
    
    match result {
        Ok(file) => println!("Opened file: {:?}", file),
        Err(e) => println!("Failed to open: {e}"),
    }
}
```

In C, `fopen()` returns `NULL` on failure and you check `errno`. You can forget to check. In Rust, `File::open()` returns `Result<File, io::Error>`. You get a warning if you ignore it entirely, and a compile error if you try to use the `File` without checking whether you actually got one.

---

## Pattern Matching With `match`

`match` is the primary tool for handling `Option` and `Result`. It's exhaustive - you must handle all variants:

```rust
fn parse_offset(s: &str) -> Option<u64> {
    s.parse::<u64>().ok()   // converts Result -> Option, discarding error details
}

fn main() {
    match parse_offset("256") {
        Some(offset) => println!("Starting at byte offset {offset}"),
        None => println!("Invalid offset - must be a non-negative integer"),
    }

    match parse_offset("abc") {
        Some(offset) => println!("Starting at byte offset {offset}"),
        None => println!("Invalid offset"),  // "abc" can't parse as u64 -> None
    }
}
```

`if let` is a shorter form when you only care about one variant:

```rust
fn main() {
    if let Some(offset) = parse_offset("1024") {
        println!("Skipping first {offset} bytes");
    }
    // If None, nothing happens - we'd want to print an error in real code
}
```

---

## The `?` Operator: Idiomatic Error Propagation

This is the most important operator for writing clean Rust code.

In C, propagating errors looks like:

```c
int open_and_read(const char *path, size_t offset, unsigned char *out, size_t len) {
    FILE *f = fopen(path, "rb");
    if (!f) return -1;

    if (fseek(f, offset, SEEK_SET) != 0) { fclose(f); return -1; }

    size_t n = fread(out, 1, len, f);
    if (n == 0) { fclose(f); return -1; }

    fclose(f);
    return (int)n;
}
```

Every call checks for failure and propagates it. In Rust, `?` does this automatically:

```rust
use std::fs::File;
use std::io::{self, BufReader, Read};

fn read_first_bytes(path: &str) -> Result<Vec<u8>, io::Error> {
    let file = File::open(path)?;          // returns early with Err if file missing
    let mut reader = BufReader::new(file);
    let mut buf = vec![0u8; 16];
    let n = reader.read(&mut buf)?;        // returns early with Err if read fails
    buf.truncate(n);
    Ok(buf)                                // success: return the bytes read
}
```

What `?` does: if the expression is `Ok(value)`, unwrap it and continue. If it's `Err(e)`, **immediately return** from the current function with that error.

It's exactly the pattern you'd write manually in C, but in one character. The function's return type must be `Result` (or `Option`) for `?` to work.

---

## `unwrap()` and `expect()`: Panic on Failure

Sometimes you want to say "if this fails, the program should crash":

```rust
fn main() {
    // unwrap(): panics with a generic message if Err or None
    let offset: u64 = "256".parse().unwrap();

    // expect(): panics with your custom message - better for debugging
    let offset: u64 = "256".parse().expect("offset must be a valid u64");
}
```

**When `unwrap()`/`expect()` is OK:**
- Tests (panicking on test failure is fine)
- Prototyping / early development
- Cases that are genuinely impossible in correct code (and you want to crash fast if you're wrong)

**When to avoid them:**
- Production code handling external input (user input, file paths, CLI arguments)
- Anything where the error is recoverable

For hexview: it's fine to `unwrap()` during milestones 1-2. Replace with proper error handling in milestones 5-6.

---

## How This Connects to the Hex Viewer

File I/O is exactly the thing `Result` was designed for. Opening a file, reading bytes, parsing arguments - all of them can fail in distinct ways, and `Result` captures all of them:

```rust
use std::fs::File;
use std::io::{self, BufReader, Read};

fn read_bytes(path: &str, offset: u64, limit: usize) -> io::Result<Vec<u8>> {
    let file = File::open(path)?;        // Err if file doesn't exist or no permission
    let mut reader = BufReader::new(file);
    
    // Seek to offset if needed
    use std::io::Seek;
    reader.seek(io::SeekFrom::Start(offset))?;  // Err if offset beyond file
    
    let mut buf = vec![0u8; limit];
    let n = reader.read(&mut buf)?;      // Err on I/O failure; Ok(0) means EOF
    buf.truncate(n);
    Ok(buf)
}
```

`File::open` failing (`ENOENT`, `EACCES` in C) comes back as `Err`. Seeking past the end comes back as `Err`. Reading succeeds but returns 0 bytes at EOF - that's `Ok(0)`, not an error. No `errno`. No null file handles. The type system forces you to think about failure at every step.

---

## Quick Reference: Handling Option and Result

```
Option<T>
  .unwrap()          -> T or panic
  .unwrap_or(x)      -> T or x (safe fallback)
  .expect("msg")     -> T or panic with "msg"
  .is_some()         -> bool
  .is_none()         -> bool
  ?                  -> unwrap or return None from calling function

Result<T, E>
  .unwrap()          -> T or panic
  .unwrap_or(x)      -> T or x (safe fallback)
  .expect("msg")     -> T or panic with "msg"
  .is_ok()           -> bool
  .is_err()          -> bool
  .ok()              -> Option<T> (discards error)
  ?                  -> unwrap or return Err(...) from calling function
```

---

## How It Breaks

**`unwrap()` panicking in production.**
`unwrap()` panics if the value is `Err` or `None`. In development, a panic with a line number is useful. In production, it's a crash that exposes internals to users and produces no useful error message for them. Never call `unwrap()` on anything that can be caused by user input, network conditions, or file system state. That covers: argument parsing, file paths, byte offset parsing, file reads, and JSON parsing. Reserve `unwrap()` for things you can *prove* at the call site are always `Ok` - and add a comment explaining why.

**The `?` operator silently converting error types.**
`?` works by calling `From` to convert the error into the function's return type. If your function returns `Result<T, io::Error>` and you use `?` on something that returns `Result<T, ParseIntError>`, the compiler needs `ParseIntError: From<io::Error>` to exist. It doesn't. You'll get a type mismatch error that points at the `?` but the real problem is that your error types are incompatible. Solutions: use `Box<dyn Error>` as your error type for flexibility, use the `anyhow` crate (common in applications), or define your own error enum with variants for each error type and implement `From` for each.

**Error messages that hide what went wrong.**
This produces a useless error:
```rust
offset_str.parse::<u64>().map_err(|_| "parse failed")?;
```
Always chain context. The user needs to know *what* failed and *why*:
```rust
offset_str.parse::<u64>()
    .map_err(|e| format!("invalid offset '{}': {}", offset_str, e))?;
```
Good error messages include the value that was invalid, the operation that failed, and the underlying cause.

**Forgetting that `Ok(0)` from `read()` means EOF, not an error.**
When you loop reading a file, the loop ends when `read()` returns `Ok(0)`. If you treat `Ok(0)` like a real read, you'll spin forever doing nothing. The correct loop check is `let n = reader.read(&mut buf)?; if n == 0 { break; }`. This is the same as `read()` returning 0 in C - it means end of file, not failure.

---

## Common Mistakes C Developers Make

**Using `unwrap()` on user input.** User input fails. Always. Handle it properly with `match` or `?`.

**Ignoring `Result` return values.** Rust will warn you if you call a function that returns `Result` and you don't use the return value. Don't suppress the warning with `let _ = ...` unless you genuinely don't care about the error.

**Confusing `Option` and `Result`.** `Option` is "might not exist" (absence is normal). `Result` is "might have failed" (failure needs explanation). You can convert between them: `.ok()` turns `Result` into `Option`, `.ok_or(err)` turns `Option` into `Result`.

**Nesting `match` when `?` would work.** If you're writing `match x { Ok(v) => { ... }, Err(e) => return Err(e) }`, that's exactly what `?` does. Use `?`.

**Forgetting that `?` requires the function to return `Result` or `Option`.** You can't use `?` in `main()` unless you change its signature to `fn main() -> Result<(), Box<dyn Error>>`.
