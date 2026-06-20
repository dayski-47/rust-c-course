# Doc 04: Modules, Crates, and Project Organization

In C, you split code across files using `.h` headers and `.c` source files. You manage visibility manually by choosing what to put in headers. Dependencies come in through your build system - Makefile, CMake, pkg-config.

Rust has a cleaner system. Every file is automatically a module. Visibility is explicit. Dependencies are declared in one file and Cargo handles the rest.

---

## The mod Keyword

You can define a module inline or in a separate file. They're equivalent:

**Inline module:**

```rust
// main.rs
mod cpu {
    pub fn read_usage() -> f64 {
        // ... reads /proc/stat or uses sysinfo ...
        42.7
    }

    // Private by default - not visible outside this module
    fn parse_stat_line(line: &str) -> f64 {
        // implementation detail
        0.0
    }
}

fn main() {
    let usage = cpu::read_usage();
    println!("CPU: {:.1}%", usage);
    // cpu::parse_stat_line("..."); // Won't compile - it's private
}
```

**File module** (more common for real projects):

When you write `mod cpu;` in `main.rs`, Rust looks for either:
- `src/cpu.rs`, or
- `src/cpu/mod.rs`

```
src/
├── main.rs         <- contains: mod cpu;  mod memory;  mod display;
├── cpu.rs          <- the cpu module
├── memory.rs       <- the memory module
└── display.rs      <- the display module
```

```rust
// main.rs
mod cpu;
mod memory;
mod display;

fn main() {
    let usage = cpu::read_usage();
    display::print_cpu(usage);
}
```

```rust
// cpu.rs
pub fn read_usage() -> f64 {
    42.7  // placeholder
}
```

That's it. No header files. No `#include`. The `mod cpu;` declaration in `main.rs` is the only thing you need.

---

## Visibility: pub vs private

By default, everything in Rust is private to its module. You expose things by adding `pub`:

```rust
// memory.rs

// Private - only accessible within this file
struct RawMemInfo {
    total_bytes: u64,
    free_bytes: u64,
}

// Public struct - visible outside the module
pub struct MemoryMetrics {
    pub total_mb: u64,    // pub field - accessible directly
    pub used_mb: u64,     // pub field - accessible directly
    cached_mb: u64,       // private field - only accessible via methods
}

// Public function - accessible from other modules
pub fn read_memory() -> MemoryMetrics {
    MemoryMetrics {
        total_mb: 16384,
        used_mb: 8192,
        cached_mb: 2048,
    }
}
```

If a struct is `pub` but its fields aren't, callers can use the struct as a type but can't read or write its fields directly. They must go through `pub` methods. This is how you enforce invariants.

---

## use - Importing Names

You can always refer to things by their full path (`cpu::read_usage()`), or bring them into scope with `use`:

```rust
use std::collections::HashMap;
use std::fmt;

// Now you can write HashMap instead of std::collections::HashMap
let mut map: HashMap<String, u64> = HashMap::new();
```

Path prefixes:
- `crate::` - from the root of your current crate
- `super::` - from the parent module
- `self::` - from the current module (rarely needed)

```rust
// src/display.rs
use crate::cpu::CpuMetrics;    // Import from the cpu module in this crate
use super::memory::MemInfo;    // Import from the parent module's memory submodule
```

Importing multiple things from the same path:

```rust
use std::fmt::{self, Display, Formatter};
//             ^^^^ imports `fmt` itself AND Display AND Formatter
```

---

## Adding External Crates

This is where Rust really shines over C. No finding and installing library headers, no pkg-config, no CMake FindPackage. You just add a line to `Cargo.toml`:

```toml
[package]
name = "sysmon"
version = "0.1.0"
edition = "2021"

[dependencies]
sysinfo = "0.30"
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

Then run `cargo build`. Cargo downloads everything, compiles it, and caches the result. The first build is slow. Subsequent builds only recompile what changed.

**SemVer in Cargo.toml:**

| Specifier | Means |
|-----------|-------|
| `"0.30"` | `>= 0.30.0, < 0.31.0` - minor updates OK |
| `"4"` | `>= 4.0.0, < 5.0.0` - patch and minor OK |
| `"=1.2.3"` | Exactly `1.2.3`, nothing else |
| `"*"` | Latest available |

The `Cargo.lock` file pins exact versions for reproducible builds. Commit it for binaries; optionally commit it for libraries.

---

## Feature Flags - Conditional Compilation

Some crates have optional features that pull in extra dependencies. You opt into them:

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
#                        ^^^^^^^^^^^^^^^^^^
#    The "derive" feature enables #[derive(Parser)] macros
```

You can also define features in your own crate:

```toml
[features]
default = ["json"]
json = ["dep:serde_json"]   # Only pull in serde_json if json feature is enabled
color = ["dep:colored"]     # Optional color support
```

```rust
// Code that only compiles when the "json" feature is enabled
#[cfg(feature = "json")]
pub fn export_json(metrics: &Metrics) -> Result<String, serde_json::Error> {
    // Return the Result - let the caller decide how to handle failure.
    // Never unwrap inside a library function: the caller might not want to panic.
    serde_json::to_string_pretty(metrics)
}
```

Feature flags replace C's `#ifdef` for this use case. The advantage: they're declared in one place, they affect dependency resolution, and the compiler checks them consistently.

---

## Refactoring the Hex Viewer Into Modules

Here's how you'd split Section 1's hex viewer into modules. This is exactly the same pattern you'll use for the system monitor.

**Before - everything in main.rs:**

```rust
// main.rs (flat, everything together)
use std::fs::File;
use std::io::Read;

fn read_bytes(path: &str) -> Result<Vec<u8>, std::io::Error> {
    let mut file = File::open(path)?;
    let mut bytes = Vec::new();
    file.read_to_end(&mut bytes)?;
    Ok(bytes)
}

fn format_hex_line(chunk: &[u8], offset: usize) -> String {
    let hex: String = chunk.iter().map(|b| format!("{:02x} ", b)).collect();
    format!("{:08x}: {}", offset, hex)
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let bytes = read_bytes("test.bin")?;
    for (i, chunk) in bytes.chunks(16).enumerate() {
        println!("{}", format_hex_line(chunk, i * 16));
    }
    Ok(())
}
```

**After - split into modules:**

```
src/
├── main.rs
├── reader.rs
└── formatter.rs
```

```rust
// reader.rs
use std::fs::File;
use std::io::Read;

pub fn read_bytes(path: &str) -> Result<Vec<u8>, std::io::Error> {
    let mut file = File::open(path)?;
    let mut bytes = Vec::new();
    file.read_to_end(&mut bytes)?;
    Ok(bytes)
    // Why ? not unwrap: this function is used by main - let main decide how
    // to report the error to the user. unwrap would crash silently with a panic.
}
```

```rust
// formatter.rs
pub fn format_hex_line(chunk: &[u8], offset: usize) -> String {
    let hex: String = chunk.iter().map(|b| format!("{:02x} ", b)).collect();
    format!("{:08x}: {}", offset, hex)
}
```

```rust
// main.rs
mod reader;
mod formatter;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let bytes = reader::read_bytes("test.bin")?;
    for (i, chunk) in bytes.chunks(16).enumerate() {
        println!("{}", formatter::format_hex_line(chunk, i * 16));
    }
    Ok(())
}
```

The split is mechanical: move the implementation into new files, add `pub` to the functions you want visible, declare `mod reader;` and `mod formatter;` in `main.rs`. Each module now has a single responsibility - reading vs. formatting - and can be tested independently.

---

## Workspace Projects - A Brief Preview

For this section's project, one crate is enough. But later in the course you'll see workspaces - a way to group multiple related crates in one repo:

```toml
# workspace Cargo.toml
[workspace]
resolver = "2"
members = [
    "sysmon-core",    # library crate with the collection logic
    "sysmon-cli",     # binary crate with the CLI
    "sysmon-web",     # binary crate with a web API
]
```

Each member is its own directory with its own `Cargo.toml`. They can depend on each other using path dependencies. This is how large Rust projects stay organized.

---

## How the System Monitor Will Be Organized

When you build the project in this section, you'll end up with:

```
src/
├── main.rs       <- argument parsing, main loop, entry point
├── cpu.rs        <- reads CPU metrics
├── memory.rs     <- reads memory metrics
├── disk.rs       <- reads disk metrics
└── display.rs    <- formats and prints everything
```

Each file is a module. `main.rs` declares them with `mod cpu; mod memory; mod disk; mod display;`. The `Metrics` struct and `Collector` trait live in `main.rs` or their own file, whichever feels natural.

This organization keeps each concern separate. If you want to change how CPU data is collected, you only touch `cpu.rs`. If you want to change the display format, you only touch `display.rs`.

---

## Common Mistakes

**1. Forgetting the mod declaration.** Creating `cpu.rs` doesn't make it part of your project. You must add `mod cpu;` to `main.rs` (or `lib.rs`). The file alone does nothing.

**2. Making the struct pub but forgetting the fields.** `pub struct Foo { bar: u32 }` - Foo is visible but `bar` is private. Outside code can hold a `Foo` but can't read `bar`. Add `pub` to the field or add a getter method.

**3. Circular module dependencies.** If `cpu.rs` tries to use something from `display.rs` and `display.rs` tries to use something from `cpu.rs`, you have a cycle. Fix it by extracting shared types to a third module (often called `types.rs` or `metrics.rs`).

**4. Trying to use `use` without a mod declaration.** `use crate::cpu::read_usage;` requires that `cpu` is declared somewhere as a module. If you haven't written `mod cpu;`, the compiler can't find it.

**5. Putting features in the wrong section.** Features for a dependency go in `[dependencies]` as `features = [...]`. Your own crate's features go in `[features]`. They look similar but they're different.
