# Zero to Hero Rust — Systems & Backend Engineering Course

**Built for:** You. A C developer who knows pointers, sockets, and manual memory management. You're not starting from zero on programming — you're starting from zero on Rust. That's a huge advantage.

**Goal:** By the end of this course you can build production-grade systems software and backend services in Rust. TCP servers, REST APIs, real-time streaming, Redis, Docker, async runtimes — all of it.

**How this works:** Every section is built around a real project. The documentation teaches you exactly what you need to complete that project — nothing more, nothing less. You read, you build, you struggle a little, you figure it out. That's the loop.

---

## The Curriculum Ladder

| # | Section | Project | Core Concepts |
|---|---------|---------|---------------|
| 01 | [Hex Viewer](./section-01-hex-viewer/) | Binary file hex dump tool (like xxd) | Types, variables, functions, structs, file I/O, slices, iterators, error handling |
| 02 | [System Monitor CLI](./section-02-system-monitor/) | Resource monitor with reporting | Vec, HashMap, enums, pattern matching, closures, iterators, modules, Cargo, CLI |
| 03 | [Multi-Client Chat Server](./section-03-chat-server/) | Real-time terminal chat (TCP) | Ownership deep dive, Arc/Mutex, threads, channels (mpsc), lifetimes, TCP server |
| 04 | [Async API Client](./section-04-async-api-client/) | GitHub repo analyzer tool | async/await, Future trait, Tokio runtime, reqwest, JSON parsing, rate limiting |
| 05 | [Music Library REST API](./section-05-music-api/) | Full CRUD music library backend | Axum, SQLite + SQLx, serde, thiserror/anyhow, Docker, migrations |
| 06 | [Real-Time Music Queue Service](./section-06-music-queue/) | Live play queue with Redis | Redis, WebSockets, JWT auth, pub/sub, tokio channels, connection pooling |
| 07 | [Distributed Task Worker](./section-07-task-worker/) | Job queue and worker system | Generics (full), traits in depth, type-state, service architecture, CI/CD, benchmarking |
| 08 | [Key-Value Storage Engine](./section-08-storage-engine/) | Persistent KV store (mini RocksDB) | Unsafe Rust, FFI (zstd), memory-mapped files, binary formats, Miri, fuzzing |

---

## How to Use This Course

### Each Section Contains

```
section-XX-name/
├── docs/           ← Read these first. In order.
│   ├── 01-...md
│   ├── 02-...md
│   └── ...
└── PROJECT.md      ← Your mission. Read after docs. Then build.
```

### The Process (Proven Method)
1. **Read the docs** in this section. Don't skim. Run every code example.
2. **Re-read the PROJECT.md** carefully. Understand the full scope before writing line one.
3. **Sketch on paper** what you're building before you open an editor. (This is engineering discipline. Do it.)
4. **Build it.** When you're stuck, the docs in that section are your first stop.
5. **Push through the hard parts.** If the borrow checker is fighting you, that's the lesson.

### The Engineering Mindset (Don't Skip This)
Before writing code for any project, answer these questions:
- **What problem am I solving?** (Not "what code will I write" — what problem.)
- **What are the inputs and outputs?** Draw this out.
- **What can go wrong?** (Edge cases, failures, bad inputs)
- **How will I know it works?** (Testable outcomes)

Software engineers who can't answer these before coding write code twice. You'll write it once.

---

## Your Starting Point: C vs. Rust Mental Model

Coming from C, here's how to reframe what you already know:

| C Concept | Rust Equivalent | The Key Difference |
|-----------|-----------------|-------------------|
| `malloc`/`free` | `Box<T>` | Rust frees automatically at end of scope. No leaks. |
| `pthread_t` / `thread_create` | `std::thread::spawn` | Rust prevents data races at compile time |
| `void *` passed around | Generics / `Box<dyn Trait>` | Type-safe. No casts needed. |
| `struct` with function pointers | `impl Trait` | Traits are Rust's version of interfaces |
| `errno` / return codes | `Result<T, E>` | Can't ignore errors. Compiler enforces handling. |
| Header files | `mod` / `use` | Modules. No separate `.h` files. |
| `#include <string.h>` | Built-in `String` / `&str` | Memory-safe strings. Two kinds (you'll learn why). |
| Segfault at runtime | Compiler error | That's the whole point. |

---

## Tools You Need

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Verify
rustc --version
cargo --version

# IDE: VS Code + rust-analyzer extension (highly recommended)
# Or: any editor with rust-analyzer LSP support

# Useful extras (install as you go)
cargo install cargo-watch    # auto-recompile on save
cargo install cargo-expand   # see what macros expand to
cargo install cargo-nextest  # faster test runner (use from Section 7+)
```

---

## Cargo Practices (Apply From Day One)

These aren't optional polish — they're the difference between a project that works on your machine and one that works everywhere.

**Commit your `Cargo.lock`** for binary projects (`[[bin]]`). The lock file pins every dependency to an exact version, so `cargo build` produces the same binary weeks later. Libraries (no `[[bin]]`) conventionally leave `Cargo.lock` out of version control; binaries keep it.

**Faster `cargo build` in debug mode.** Dependencies compile in debug mode too, which is slow. Add this to your `Cargo.toml` to optimize them while keeping your own code debug-friendly:

```toml
[profile.dev.package."*"]
opt-level = 2
```

This one change cuts compilation time significantly on projects with large dependency trees (serde, tokio, etc.).

**In CI, use `CARGO_ENCODED_RUSTFLAGS` instead of `RUSTFLAGS`.** The plain `RUSTFLAGS` env var is overridden by any `[build]` section in `.cargo/config.toml`. The encoded form stacks cleanly:

```bash
# CI script
export CARGO_ENCODED_RUSTFLAGS="-D\x1fwarnings"  # deny warnings without config conflicts
```

**Check for duplicate dependencies** before they cause link errors or bloated binaries:

```bash
cargo tree --duplicates
```

If the same crate appears in two versions, investigate — often a dep pin in `Cargo.toml` resolves it.

---

## Difficulty Key Used in Docs

- 🟢 **Straightforward** — run it, understand it, move on
- 🟡 **Think about it** — pause and understand before continuing
- 🔴 **Challenge** — you're meant to struggle here. That's the learning.

---

## Source Material

This course draws from the following books in this repository:
- **c-cpp-book** — Rust from a C/C++ developer's perspective (your primary reference)
- **async-book** — Async Rust and Tokio in depth
- **rust-patterns-book** — Production Rust patterns and advanced concepts
- **engineering-book** — Real-world engineering: CI/CD, benchmarking, tooling
- **type-driven-correctness-book** — Advanced: making the compiler enforce correctness

---

*Start with Section 01. Don't skip sections. The ladder exists for a reason.*
