# Doc 15 — Rust Idioms and Best Practices

🟢 Section 8 is the technical peak of the course. ironkv uses unsafe code with careful invariants, FFI bindings to a C library, `repr(C)` structs for zero-copy binary parsing, memory-mapped files, and custom binary formats. This doc collects the idioms that make that complexity manageable.

---

## Every `unsafe` Block Needs a `// Safety:` Comment

This is the highest-impact practice for unsafe Rust. Without a comment, every `unsafe` block is a mystery that requires re-deriving the proof from scratch. With a comment, future maintainers know what invariants were assumed and why they hold:

```rust
// ❌ No comment — what invariants are we assuming?
let header = unsafe { &*(ptr as *const EntryHeader) };

// ✅ Comment documents the proof
// Safety: ptr points to at least HEADER_SIZE bytes of initialized memory,
// guaranteed by the caller's offset check. EntryHeader is repr(C) with no
// invalid bit patterns (all fields are integer types).
let header = unsafe { &*(ptr as *const EntryHeader) };
```

Make this non-negotiable: no `unsafe` block merges without a `// Safety:` comment. Add a Clippy lint to enforce it:

```toml
# Cargo.toml
[workspace.lints.clippy]
undocumented_unsafe_blocks = "deny"
```

---

## Minimize the Unsafe Surface

Keep `unsafe` isolated in dedicated modules. The rest of the codebase should be entirely safe Rust that calls into those modules through safe interfaces:

```
ironkv-core/src/
├── lib.rs           — safe public API
├── storage/
│   ├── mod.rs       — safe
│   ├── index.rs     — safe
│   ├── wal.rs       — safe
│   └── mmap.rs      — unsafe internals here; safe public interface
└── ffi/
    ├── mod.rs       — safe wrappers
    └── zstd_raw.rs  — raw extern "C" declarations; nothing else
```

The pattern: unsafe code in `mmap.rs` and `zstd_raw.rs`. Everything else calls safe wrapper functions. When you need to audit unsafe code, you know exactly where to look.

---

## `repr(C)` Structs: Explicit Padding and Alignment

Never let the compiler choose padding for on-disk structs:

```rust
// ❌ Compiler-chosen padding — layout may change between Rust versions
#[repr(C)]
struct Header {
    version: u8,    // 1 byte + 3 bytes padding (assumed)
    size: u32,      // 4 bytes
}

// ✅ Explicit padding — layout is documented and stable
#[repr(C)]
struct Header {
    version: u8,
    _pad: [u8; 3],  // Documented: 3 bytes of alignment padding
    size: u32,
}

// Verify the expected size at compile time:
const _: () = assert!(
    std::mem::size_of::<Header>() == 8,
    "Header must be exactly 8 bytes"
);
```

The compile-time `assert!` catches any future change to the struct (added field, changed type) that would alter the binary layout.

---

## Write Invariant Documentation for Data Structures

Complex data structures have invariants — relationships that must always hold. Document them at the type level:

```rust
/// In-memory index mapping keys to their on-disk location.
///
/// Invariants maintained by all methods:
/// - Every offset in `entries` refers to a valid entry in the data file.
/// - `entries` contains no duplicate keys.
/// - `entries` is not modified during an active mmap read (ensured by &mut self on writes).
/// - Entries with `offset == 0` are tombstones (deleted keys) and must not be returned.
pub struct Index {
    entries: std::collections::BTreeMap<Vec<u8>, (u64, u32)>,  // key → (offset, length)
}
```

These comments are not just documentation — they're the specification that `unsafe` code depends on. When someone adds a method, they can check whether their implementation maintains the invariants.

---

## Newtype Wrappers for File Offsets

File offsets and lengths are both integers, but mixing them is a bug:

```rust
// ❌ Raw u64 — easy to pass offset where length is expected
fn read_entry(file: &File, offset: u64, length: u64) {}
read_entry(&file, entry.length, entry.offset);  // Swapped — compiles fine

// ✅ Newtypes — swap is a type error
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct FileOffset(pub u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct EntryLength(pub u32);

fn read_entry(file: &File, offset: FileOffset, length: EntryLength) {}
// read_entry(&file, entry.length, entry.offset);  // Compile error: mismatched types
```

The newtype pattern pays off especially in storage engine code, where offsets, lengths, sequence numbers, and checksums are all integers that must never be confused.

---

## Use `impl Write` and `impl Read` for Testable I/O

Code that takes `impl std::io::Write` can be tested with an in-memory buffer instead of a real file:

```rust
// ❌ Hard to test — must create real files
pub fn write_wal_entry(path: &Path, entry: &WalEntry) -> std::io::Result<()> {
    let mut file = std::fs::OpenOptions::new().append(true).open(path)?;
    write_entry_to(&mut file, entry)
}

// ✅ Testable — inject any writer
pub fn write_entry_to(writer: &mut impl std::io::Write, entry: &WalEntry) -> std::io::Result<()> {
    let encoded = bincode::serialize(entry).unwrap();
    let len = encoded.len() as u32;
    writer.write_all(&len.to_le_bytes())?;
    writer.write_all(&encoded)?;
    Ok(())
}

// In tests — no file system needed
#[test]
fn test_entry_serialization() {
    let entry = WalEntry { sequence: 1, key: b"k".to_vec(), value: Some(b"v".to_vec()), op: EntryOp::Put };
    let mut buf = Vec::new();
    write_entry_to(&mut buf, &entry).unwrap();
    assert!(!buf.is_empty());
}
```

---

## Avoid `Box<dyn Error>` in Library Code

In application code, `Box<dyn Error>` is convenient — it accepts any error type. In library code, it's problematic: callers can't match on the error type, can't get structured error information, and the API is less useful.

```rust
// ❌ Application convenience — bad for libraries
pub fn open(path: &Path) -> Result<IronKV, Box<dyn std::error::Error>> {}

// ✅ Typed error — callers can match and inspect
use thiserror::Error;

#[derive(Debug, Error)]
pub enum OpenError {
    #[error("file not found: {0}")]
    NotFound(std::path::PathBuf),

    #[error("corrupt data file: {0}")]
    Corrupt(String),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
}

pub fn open(path: &Path) -> Result<IronKV, OpenError> {}
```

With a typed error, callers can write:

```rust
match db::open(path) {
    Ok(db) => use_db(db),
    Err(OpenError::NotFound(_)) => create_new_db(path),
    Err(OpenError::Corrupt(msg)) => {
        eprintln!("Attempting WAL recovery: {msg}");
        recover_from_wal(path)
    }
    Err(e) => return Err(e.into()),
}
```

This would be impossible with `Box<dyn Error>`.

---

## The Storage Engine Completion Checklist

Before calling ironkv production-ready:

- [ ] Every `unsafe` block has a `// Safety:` comment explaining the invariants
- [ ] Invariants are documented on all data structures that have them
- [ ] `repr(C)` struct sizes are verified with `const _: () = assert!(size_of::<T>() == N);`
- [ ] Checksums are verified on every entry read, not just on startup
- [ ] WAL replay is tested — write entries, crash-simulate (drop the IronKV), reopen, verify data
- [ ] Concurrent write tests: two writers, verify no corruption (if applicable to your API)
- [ ] Miri clean: `cargo +nightly miri test` passes with no UB reports
- [ ] `cargo audit` passes — no known vulnerabilities in dependencies
- [ ] musl build succeeds: `cargo build --release --target x86_64-unknown-linux-musl`
- [ ] `cargo bench` baseline established and results are in the expected range
- [ ] Public API is fully documented — all `pub` items have doc comments

---

## The Systems Programmer's Mindset

By this section, you've built tools that most developers never touch: a custom binary file format, zero-copy parsing from mmap'd memory, FFI bindings, and a write-ahead log. The mindset shift that makes this work:

**Trust the type system first.** If the type system makes it impossible to call a function in the wrong state, you don't need a runtime check. `FileOffset` and `EntryLength` as newtypes are not overhead — they're documentation that the compiler enforces.

**Minimize the unsafe surface.** Every line of `unsafe` code is code you personally take responsibility for. Make that surface as small as possible, document it rigorously, and test it with Miri.

**Measure before optimizing.** The mmap zero-copy read path is fast — but how fast? Is it actually the bottleneck? `cargo bench` with Criterion answers this. Don't optimize what isn't slow.

**The invariants are load-bearing.** The comment "// entries contains no duplicate keys" isn't decorative — it's the reason the caller doesn't need to check for duplicates. When an invariant breaks, the bugs downstream are hard to trace. Enforce them in constructors and document them where they're assumed.

These are the instincts that distinguish a systems programmer from an application programmer. By section 8, you have them.
