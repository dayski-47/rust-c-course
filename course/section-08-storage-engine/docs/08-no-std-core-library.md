# Doc 08 — `no_std` and the Core Library

🔴 The standard library is a luxury. For embedded systems, kernel modules, UEFI firmware, and WebAssembly targets, there is no operating system to provide the heap, threads, or file system that `std` depends on. This doc teaches you what remains when you strip `std` — and how to use it.

For ironkv, `no_std` is not a primary requirement. The value of this doc is understanding the *layers* of the Rust standard library — what `core` provides unconditionally, what `alloc` adds with a heap, and what `std` adds with an OS. That understanding changes how you reason about portability and dependencies.

---

## The Three Layers

```
┌─────────────────────────────────────────────────────┐
│  std                                                 │
│  (requires OS: threads, file I/O, sockets, env)     │
├─────────────────────────────────────────────────────┤
│  alloc                                               │
│  (requires heap: Vec, Box, String, Arc, BTreeMap)   │
├─────────────────────────────────────────────────────┤
│  core                                                │
│  (always available: primitives, traits, iterators,  │
│   Option, Result, slice, str, atomics, ptr)          │
└─────────────────────────────────────────────────────┘
```

`core` works everywhere — bare metal, kernel space, WASM, embedded. It has no runtime dependencies and uses no heap. Everything in `core` is available in your `no_std` crate.

`alloc` adds heap-allocated types. It requires a global allocator to be registered (your `#[global_allocator]` annotation). It's available in `no_std` environments that have a heap (custom allocators, embedded with external SRAM).

`std` re-exports everything in `core` and `alloc`, and adds OS-dependent functionality. Most Rust code uses `std` because most code runs on a host OS.

---

## What `core` Provides

Core types you've been using — they all live in `core`:

```rust
// These work in no_std environments:
use core::option::Option;
use core::result::Result;
use core::fmt::Display;
use core::iter::Iterator;
use core::slice;
use core::str;
use core::mem;
use core::ptr;
use core::sync::atomic::{AtomicU64, Ordering};
use core::marker::{PhantomData, Send, Sync};
```

The fundamental numeric types, all iterator adaptors, pattern matching, closures, traits — all in `core`. The things that are NOT in `core`:

- `Vec`, `Box`, `String`, `Arc`, `Rc` — require heap allocation (in `alloc`)
- `File`, `TcpStream`, `Mutex` — require OS primitives (in `std`)
- `thread::spawn` — requires OS threads (in `std`)

---

## Writing a `no_std` Compatible Library

Make ironkv-core optionally `no_std` with a feature flag:

```toml
# ironkv-core/Cargo.toml
[features]
default = ["std"]
std = []

[dependencies]
# no_std-compatible crates exist for most common needs:
```

```rust
// ironkv-core/src/lib.rs

// When the "std" feature is not enabled, use no_std mode
#![cfg_attr(not(feature = "std"), no_std)]

// In no_std mode, we still need alloc for Vec, Box, etc.
#[cfg(not(feature = "std"))]
extern crate alloc;

// Use the right prelude based on whether std is available:
#[cfg(feature = "std")]
use std::{vec::Vec, string::String, boxed::Box};

#[cfg(not(feature = "std"))]
use alloc::{vec::Vec, string::String, boxed::Box};

// Core types are available in both modes via core::
use core::fmt;
use core::slice;
```

The `#![cfg_attr(not(feature = "std"), no_std)]` attribute says: "when the `std` feature is disabled, don't link the standard library." This lets the same crate compile in both environments.

---

## Custom Panic Handlers

In `no_std`, there's no default panic handler. You must provide one:

```rust
// For embedded/bare-metal targets:
#![no_std]
#![no_main]  // No standard main function

use core::panic::PanicInfo;

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    // In real embedded code, write to a debug UART or halt:
    loop {}  // Infinite loop — can't exit without an OS
}
```

For `no_std` libraries (not applications), you don't provide a panic handler — the final application binary is responsible for that. Your library just needs `#![no_std]`.

---

## Verifying Feature Combinations with `cargo-hack`

The most common `no_std` bug: a dependency pulls in `std` transitively, breaking the `no_std` build. Verify systematically:

```bash
cargo install cargo-hack

# Verify the no-std path compiles
cargo hack check --no-default-features --target thumbv7m-none-eabi

# Verify all features compile
cargo hack check --each-feature --workspace
```

For ironkv's `no_std` feature gate, add to CI:

```yaml
- name: Check no_std compatibility
  run: |
    rustup target add thumbv7m-none-eabi
    cargo check --no-default-features --target thumbv7m-none-eabi -p ironkv-core
```

This catches the problem before users report it: "I tried to use ironkv-core on my embedded project and it pulled in `std`."

---

## Writing `no_std` Compatible Code

The patterns for writing code that works in both environments:

### Conditionally use std or core types

```rust
// Instead of:
use std::fmt;

// Use core, which is always available:
use core::fmt;

// For types in alloc vs std, feature-gate them:
#[cfg(feature = "std")]
use std::sync::Arc;

#[cfg(not(feature = "std"))]
use alloc::sync::Arc;
```

### Use the `core::` prefix explicitly

In `no_std` code, use `core::` explicitly to make it clear you're not accidentally using `std`:

```rust
pub fn crc32_entry(key: &[u8], value: &[u8]) -> u32 {
    // core::iter is always available
    key.iter()
        .chain(value.iter())
        .fold(0xFFFF_FFFFu32, |crc, &byte| {
            (crc >> 8) ^ CRC_TABLE[((crc ^ byte as u32) & 0xFF) as usize]
        })
        ^ 0xFFFF_FFFF
}
```

### Avoid `std::io` in `no_std` paths

`std::io::Read` and `std::io::Write` are in `std`, not `core`. In `no_std` code, define your own minimal I/O traits or use a crate like `embedded-io`:

```rust
// In no_std mode, define lightweight I/O traits:
pub trait Write {
    type Error;
    fn write_all(&mut self, data: &[u8]) -> Result<(), Self::Error>;
}

pub trait Read {
    type Error;
    fn read_exact(&mut self, buf: &mut [u8]) -> Result<(), Self::Error>;
}
```

---

## The `no_std` Dependency Ecosystem

Most crates in the Rust ecosystem support `no_std` with a feature flag. When evaluating a dependency for a `no_std` project, look for:

```toml
# In the crate's Cargo.toml:
[features]
default = ["std"]
std = []  # The "std" feature enables std-dependent code
```

Well-maintained `no_std` compatible crates:

| Category | Crate |
|----------|-------|
| Error handling | `thiserror` (with `std` feature off), `snafu` |
| Serialization | `serde` (with `std` feature off) |
| Checksums | `crc32fast` |
| Fixed-size collections | `heapless` |
| Formatting | `ufmt` |
| Logging | `defmt` (embedded), `log` |

When a crate doesn't support `no_std`, you can't use it in `no_std` code. For ironkv's core types (the ones that need `no_std` portability), avoid any dependency that unconditionally links `std`.

---

## Relevance for ironkv

ironkv itself runs on Linux — it needs OS file I/O and memory mapping, which are firmly in `std`. The `no_std` knowledge applies in two ways:

**1. Keeping `ironkv-core` portably designed:**
The index types (`IndexEntry`), the entry format (`EntryHeader`), checksum functions — these don't inherently need `std`. Writing them with `core` primitives means they could be embedded in a future `no_std` variant (e.g., ironkv on a microcontroller with SPI flash storage).

**2. Auditing dependencies:**
`cargo tree --no-dedupe | grep "\[proc-macro\]"` shows your proc-macro dependencies. Proc macros always run on the host machine, so they can use `std` freely — they're compiled for the host, not the target. But regular dependencies that aren't `no_std` compatible pull in `std` transitively, which prevents ever using your crate in `no_std` environments.

The practical takeaway: even if you don't need `no_std` now, write your core types against `core` traits where possible. It costs nothing and leaves doors open.
