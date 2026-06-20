# 08 - Rust Idioms and Best Practices

> **Project:** Hex Viewer - reading binary files, formatting bytes as hex and ASCII, printing aligned output to the terminal.

## In This Section

You are building a hex viewer that opens a binary file, reads it in chunks, and prints each 16-byte line as both hex digits and a printable-ASCII sidebar. Three idioms from this document will appear immediately:

- **Testing in the same file** - `is_printable(byte: u8)` is the perfect first test function to write for this project. It has clear boundaries and no side effects.
- **References in function signatures** - your formatting functions will take `&[u8]` (a slice of bytes), not `Vec<u8>`. Slices are the idiomatic way to accept a sequence of bytes without taking ownership.
- **Error handling hierarchy** - the entry point reads a file path from `argv`. File I/O can fail. Use `anyhow::Result<()>` in `main` or a thin `run()` wrapper from the start.

---

## 1. Naming Conventions

The Rust community follows these naming conventions. Your code looks wrong to any Rust developer if you don't:

| What | Convention | Example |
|------|-----------|---------|
| Variables, functions, methods | `snake_case` | `read_bytes`, `byte_count` |
| Types, structs, enums, traits | `PascalCase` | `ByteOffset`, `HexLine`, `AppError` |
| Constants, statics | `UPPER_SNAKE_CASE` | `MAX_LINE_WIDTH`, `DEFAULT_OFFSET` |
| Modules | `snake_case` | `mod file_io`, `mod error` |
| Enum variants | `PascalCase` | `Ok`, `Err`, `Some`, `None`, `FileNotFound` |

Do not use `camelCase` for variables (that is JavaScript). Do not use `ALL_CAPS` for types.

---

## 2. Prefer Owned Types in Structs, References in Function Signatures

A C developer's instinct: "use a pointer to avoid copying." In Rust, this leads to lifetime headaches.

**Rule: Structs should own their data.** Use `String`, not `&str`. Use `Vec<T>`, not `&[T]`.

**Rule: Functions should borrow.** Function parameters should be `&str` (not `String`), `&[T]` (not `Vec<T>`), `&T` (not `T`) unless you need to store or transfer ownership.

Why this works: `String` can be passed as `&str` automatically (deref coercion). `Vec<T>` can be passed as `&[T]` automatically. The reverse is not true - you cannot pass a `&str` where a `String` is needed without cloning.

In the hex viewer, your `format_hex_line` function takes `&[u8]` so that the caller is not forced to give up its buffer:

```rust
// Good: borrow a slice, return owned output
fn format_hex_line(bytes: &[u8], offset: usize) -> String {
    // ...
}

// Bad: takes ownership of the Vec - caller loses its buffer
fn format_hex_line(bytes: Vec<u8>, offset: usize) -> String {
    // ...
}
```

---

## 3. The Builder Pattern for Complex Construction

When a struct has many optional fields, avoid a constructor with 7 parameters. Use a builder:

```rust
// Bad: hard to read at call site
let cfg = ViewerConfig::new("file.bin", 16, 0, true, false, Some(256));

// Good: self-documenting
let cfg = ViewerConfig::builder()
    .path("file.bin")
    .bytes_per_line(16)
    .show_offset(true)
    .build()?;
```

You will use this pattern when working with `reqwest` clients, Tokio runtimes, and database connection pools in later sections. Learn to recognize it now.

---

## 4. Error Handling Hierarchy

**Do not use:** `unwrap()` or `expect()` in production paths - anything that can be caused by external input, file contents, or command-line arguments.

**Early development:** `Box<dyn std::error::Error>` as the error type. Gets you started without designing an error type first.

**Applications (tools, services):** `anyhow::Error`. Context-aware, stackable. The `anyhow::Context` trait lets you attach messages.

**Libraries:** `thiserror` with a typed enum. Callers need to match on variants; `anyhow` would hide them.

The pattern:

```rust
fn run() -> anyhow::Result<()> {
    let path = std::env::args().nth(1)
        .ok_or_else(|| anyhow::anyhow!("usage: hex-viewer <file>"))?;

    let data = std::fs::read(&path)
        .with_context(|| format!("failed to read {path}"))?;

    print_hex(&data);
    Ok(())
}

fn main() {
    if let Err(e) = run() {
        eprintln!("error: {e:#}");
        std::process::exit(1);
    }
}
```

The `{e:#}` format prints the full error chain - every `.with_context()` layer, in order. This is what users actually need to diagnose problems.

---

## 5. Don't Match on Bool - Use the Right Abstraction

```rust
// Bad: checking is_some() then calling unwrap()
if thing.is_some() {
    let val = thing.unwrap();  // redundant and dangerous
    use(val);
}

// Good: if let destructures in one step
if let Some(val) = thing {
    use(val);
}

// Bad: manually re-wrapping an error
if result.is_err() {
    return Err(result.unwrap_err());
}

// Good: ? propagates automatically
let val = result?;
```

---

## 6. Closures as First-Class Values

In C, function pointers do what closures do, but closures also capture their environment. You will see this in iterator chains throughout the hex viewer:

```rust
// Closure capturing `width` from the outer scope
let lines: Vec<String> = data
    .chunks(width)
    .enumerate()
    .map(|(i, chunk)| format_hex_line(chunk, i * width))
    .collect();
```

Know the three closure traits:

- `Fn` - can be called multiple times, borrows captured variables
- `FnMut` - can be called multiple times, mutably borrows captured variables
- `FnOnce` - can only be called once, takes ownership of captured variables

When you pass a closure to `thread::spawn` or `tokio::spawn` (in later sections), it must be `FnOnce + Send + 'static`. That `'static` requirement is what forces you to use `Arc::clone` instead of passing a raw reference - the thread may outlive the stack frame where the reference lives.

---

## 7. The Type Alias Habit

If you see a complex type written out more than once, create an alias:

```rust
type Result<T> = std::result::Result<T, AppError>;
type HexLines = Vec<(usize, String, String)>;  // (offset, hex, ascii)
```

Now the code reads as intent, not as implementation. You will see this pattern used heavily in the error-handling and async sections.

---

## 8. Zero-Cost Abstractions: When to Use Them

Rust's iterators, closures, and generics compile down to the same assembly as hand-written loops. Prefer them over manual index loops not just for readability, but because the compiler can optimize them better.

```rust
// Avoid: manual index loop
for i in 0..bytes.len() {
    if bytes[i] >= 0x20 && bytes[i] < 0x7f {
        count += 1;
    }
}

// Prefer: iterator chain
let count = bytes.iter().filter(|&&b| is_printable(b)).count();
```

Use generics over `Box<dyn Trait>` when the concrete type is known at compile time. Use `Box<dyn Trait>` when you genuinely need different concrete types at runtime - for example, a list of formatters chosen at startup.

---

## 9. Testing: Write Tests in the Same File

Rust's `#[cfg(test)]` module lets you put unit tests directly in the same file as the code they test. Do not create a separate test file for unit tests.

`is_printable` is the ideal first test in this project. The boundary values are exactly defined by the ASCII table:

```rust
pub fn is_printable(byte: u8) -> bool {
    byte >= 0x20 && byte < 0x7f
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn printable_boundaries() {
        assert!(is_printable(0x20));   // space - printable
        assert!(is_printable(0x7e));   // ~ - printable
        assert!(!is_printable(0x1f));  // control char - not printable
        assert!(!is_printable(0x7f));  // DEL - not printable
    }

    #[test]
    fn format_hex_line_pads_short_chunks() {
        // A chunk shorter than 16 bytes must still print aligned output
        let line = format_hex_line(&[0xde, 0xad], 0);
        assert!(line.contains("de ad"));
    }
}
```

Run with `cargo test`. No extra framework or configuration needed.

---

## 10. Clippy: Your Automated Code Reviewer

`cargo clippy` is Rust's linter. Run it constantly. It catches:

- `if x == true` should be `if x`
- `vec.len() == 0` should be `vec.is_empty()`
- `for i in 0..vec.len() { vec[i] }` should use `.iter()` or `.iter().enumerate()`
- Unnecessary `.clone()` calls
- `match x { true => ..., false => ... }` should be `if x { ... } else { ... }`
- Hundreds more

Add this to `Cargo.toml` to treat Clippy warnings as build errors:

```toml
[lints.clippy]
all = "warn"
```

Run `cargo clippy -- -D warnings` in CI to enforce this automatically.
