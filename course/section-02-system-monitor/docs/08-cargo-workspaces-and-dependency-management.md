# Doc 08 - Cargo Workspaces and Dependency Management

🟢 Structural - these are habits you build once and use in every project forever.

The system monitor is large enough to split into two crates: a library (`monitor-core`) containing the data types and logic, and a binary (`monitor-cli`) containing the command-line interface. This is the standard pattern for any non-trivial Rust project. It makes the core logic independently testable, reusable, and eventually publishable. This doc explains workspaces, how to manage dependencies across them, and how to make sure nothing dangerous is hiding in your dependency tree.

---

## What a Workspace Is

A Cargo workspace is a collection of related crates that share a single `Cargo.lock` and a single `target/` directory. All crates in the workspace compile together and use identical versions of every shared dependency.

Without a workspace:
```
project/
├── monitor-core/   ← has its own Cargo.lock, its own target/
│   ├── Cargo.toml
│   └── src/
└── monitor-cli/    ← different Cargo.lock, different target/, separate builds
    ├── Cargo.toml
    └── src/
```

With a workspace:
```
project/
├── Cargo.toml       ← workspace root: lists members
├── Cargo.lock       ← ONE lock file for all crates
├── target/          ← ONE build directory, shared compilation cache
├── monitor-core/
│   ├── Cargo.toml   ← crate manifest, no [dependencies] duplication
│   └── src/
└── monitor-cli/
    ├── Cargo.toml
    └── src/
```

Benefits:
- **Single lock file** - all crates use the same dependency versions. No version drift between your library and binary.
- **Shared build cache** - compile `serde` once, reuse it everywhere.
- **Single `cargo test`** - tests all crates in one command.
- **Dependency deduplication** - `serde 1.0.193` is compiled once, not once per crate.

---

## Setting Up a Workspace

The root `Cargo.toml` declares the workspace:

```toml
# Cargo.toml (workspace root - no [package] section)
[workspace]
members = [
    "monitor-core",
    "monitor-cli",
]
resolver = "2"  # Always use resolver v2 (required for features to work correctly)

# Centralize shared dependency versions here
[workspace.dependencies]
serde        = { version = "1.0", features = ["derive"] }
serde_json   = "1.0"
thiserror    = "1.0"
anyhow       = "1.0"
clap         = { version = "4", features = ["derive"] }
chrono       = "0.4"
```

The library crate:

```toml
# monitor-core/Cargo.toml
[package]
name    = "monitor-core"
version = "0.1.0"
edition = "2021"

[dependencies]
serde     = { workspace = true }       # Inherits from [workspace.dependencies]
thiserror = { workspace = true }
chrono    = { workspace = true }
```

The binary crate:

```toml
# monitor-cli/Cargo.toml
[package]
name    = "monitor-cli"
version = "0.1.0"
edition = "2021"

[dependencies]
monitor-core = { path = "../monitor-core" }   # Local crate reference
serde_json   = { workspace = true }
anyhow       = { workspace = true }
clap         = { workspace = true }
```

Now, from the workspace root:

```bash
cargo build              # Build all crates
cargo test               # Test all crates
cargo run -p monitor-cli -- --help   # Run a specific binary
cargo build -p monitor-core          # Build just the library
```

---

## Workspace Dependency Management

### `[workspace.dependencies]` - single source of truth for versions

Without workspace dependencies, each crate declares its own version of `serde`. If `monitor-core` uses `serde = "1.0.193"` and `monitor-cli` uses `serde = "1.0.150"`, Cargo has to figure out which to use (it picks the highest compatible). This creates confusion and potential for subtle version mismatch bugs.

With `[workspace.dependencies]`, all crates say `serde = { workspace = true }`. Change the version in one place, all crates update. You can also set features in one place:

```toml
# Workspace root
[workspace.dependencies]
tokio = { version = "1", features = ["full"] }  # All crates get "full" features

# But a specific crate can override:
# in monitor-core/Cargo.toml:
[dependencies]
tokio = { workspace = true, features = ["rt", "sync"] }  # Only these features
```

### Feature flags

Dependencies can have optional features. Features are additive - once any crate enables a feature, all crates in the workspace get it (because they share the same compiled artifact).

```toml
# monitor-core/Cargo.toml
[features]
default = []
json-output  = ["dep:serde_json"]  # Only pull in serde_json if json-output is requested
prometheus   = ["dep:metrics"]     # Optional metrics export

[dependencies]
serde_json = { workspace = true, optional = true }
metrics    = { version = "0.21", optional = true }
```

```bash
cargo build --features json-output     # Enable just JSON output
cargo build --all-features             # Enable everything
cargo build --no-default-features      # Bare minimum
```

Feature flags let users of your library pay only for what they use. A user who doesn't need JSON output doesn't compile `serde_json`.

---

## The Dependency Tree

Your `Cargo.lock` might say you have 50 dependencies. Do you know what they all are? `cargo tree` shows you:

```bash
# Full dependency tree
cargo tree

# Just your direct dependencies (depth 1)
cargo tree --depth 1

# Why is a crate in your tree? (trace its origin)
cargo tree --invert --package openssl-sys
# Shows: your-crate → hyper → native-tls → openssl-sys

# Find crates that appear at multiple versions
cargo tree --duplicates
```

### Understanding duplicate versions

If `cargo tree --duplicates` shows:

```
syn v1.0.109
└── serde_derive v1.0.193

syn v2.0.48
├── thiserror-impl v1.0.56
└── tokio-macros v2.2.0
```

You have `syn` compiled twice. This is common (syn 1.x and 2.x coexist during the ecosystem's migration period), but each duplicate adds compile time and binary size. When possible, update your direct dependencies to use a unified version.

---

## Vulnerability Scanning: `cargo audit`

Your binary contains every transitive dependency. A CVE in any one of them is your problem.

```bash
cargo install cargo-audit

cargo audit
# Checks Cargo.lock against the RustSec Advisory Database
# Output:
# Crate:    chrono
# Version:  0.4.19
# Title:    Potential segfault in localtime_r
# Solution: Upgrade to >= 0.4.20
```

Run this on every push. Add it to CI:

```yaml
# .github/workflows/audit.yml
- uses: rustsec/audit-check@v2
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
```

Also schedule it daily - new advisories appear continuously, even for versions you haven't touched:

```yaml
on:
  schedule:
    - cron: '0 0 * * *'
  push:
    paths: ['Cargo.lock']
```

---

## Policy Enforcement: `cargo deny`

`cargo-deny` is more powerful than `cargo-audit` - it enforces four categories of policy simultaneously:

```bash
cargo install cargo-deny
cargo deny init        # Creates deny.toml
cargo deny check       # Runs all checks
```

A reasonable starting configuration for the system monitor:

```toml
# deny.toml

[advisories]
vulnerability = "deny"   # Fail CI on known CVEs
unmaintained  = "warn"   # Flag unmaintained crates
yanked        = "deny"   # Don't use yanked versions

[licenses]
unlicensed = "deny"
allow = [
    "MIT",
    "Apache-2.0",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC",
    "Unicode-DFS-2016",
]
copyleft = "deny"    # No GPL in a library that others might link

[bans]
multiple-versions = "warn"    # Track duplicates without blocking
wildcards         = "deny"    # No path = "*" dependencies
deny = [
    # Prefer rustls over OpenSSL to avoid C dependency
    { name = "openssl-sys" },
]

[sources]
unknown-registry = "deny"    # Only crates.io
unknown-git      = "deny"    # No random git dependencies
```

### What cargo deny catches that cargo audit misses

| Issue | cargo audit | cargo deny |
|-------|-------------|------------|
| Known CVEs | ✅ | ✅ |
| Yanked crate versions | ❌ | ✅ |
| GPL license violations | ❌ | ✅ |
| Duplicate crate versions | ❌ | ✅ |
| Forbidden crates | ❌ | ✅ |
| Non-crates.io sources | ❌ | ✅ |

Use both in CI. `cargo audit` for speed, `cargo deny` for policy.

---

## Lock File Strategy

`Cargo.lock` records the exact version of every dependency. The question is whether to commit it.

**For binaries (applications):** Always commit `Cargo.lock`. You want reproducible builds. When you deploy, you want exactly the same code you tested. Without a committed lock file, `cargo build` might pull in a dependency that was published an hour ago.

**For libraries (published to crates.io):** Do not commit `Cargo.lock`. Library consumers have their own lock files, and committing yours adds noise without benefit. The exception: if your library is also used as a binary (has `[[bin]]` in Cargo.toml), commit the lock file.

For the system monitor workspace:
- `monitor-core` (library) - don't commit lock file if you plan to publish
- `monitor-cli` (binary) - always commit lock file
- Since they share a workspace, commit the workspace `Cargo.lock`

---

## Tracking Updates: `cargo outdated`

```bash
cargo install cargo-outdated

cargo outdated --workspace
# Output:
# Name       Project  Compat  Latest  Kind
# clap       4.3.0    4.5.1   4.5.1   Normal
# serde      1.0.193  1.0.203 1.0.203 Normal
# thiserror  1.0.50   2.0.3   2.0.3   Normal  ← major version (breaking changes likely)
```

Update compatible versions:
```bash
cargo update              # Update Cargo.lock to latest compatible versions
cargo update -p serde     # Update just serde
```

For major version bumps (like `thiserror 1.x` → `2.x`): read the migration guide, update `Cargo.toml` manually, then rebuild and fix any breaking changes.

---

## Workspace Structure for the System Monitor

Here's the target structure for the section 02 project:

```
section-02-system-monitor/
├── Cargo.toml           ← workspace root
├── Cargo.lock
├── deny.toml            ← cargo-deny policy
├── monitor-core/
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs       ← public API
│       ├── sample.rs    ← Sample, RawSample types
│       ├── collector.rs ← reads /proc/stat, /proc/meminfo
│       ├── stats.rs     ← aggregation: avg, max, min
│       └── format.rs    ← Formatter trait + implementations
└── monitor-cli/
    ├── Cargo.toml
    └── src/
        └── main.rs      ← CLI argument parsing + calls monitor-core
```

**What goes in `monitor-core` (library):**
- All data types (`Sample`, `CpuStats`, `MemStats`)
- All `From` and `TryFrom` implementations
- The `Formatter` trait and its implementations
- The collection logic (reading `/proc` files)
- All unit tests

**What goes in `monitor-cli` (binary):**
- `main.rs` - parse CLI args with `clap`, call `monitor-core`, print results
- No business logic - just wiring

This split means you can write integration tests for `monitor-core` without the CLI, and later reuse the library in a TUI, a daemon, or a web endpoint.

---

## Reading Crate Documentation - A Meta-Skill You Need Today

Adding a dependency is one thing. Figuring out what it actually provides is another. Every time you add a new crate, you need to know where its documentation lives and how to navigate it.

**docs.rs** is the official documentation host for all crates published to crates.io. Every crate gets its documentation auto-built and hosted there:

```
https://docs.rs/sysinfo        ← latest version
https://docs.rs/sysinfo/0.30   ← specific version
https://docs.rs/clap            ← the CLI library from Section 1
```

When you add `sysinfo = "0.30"` to your `Cargo.toml`, go to `https://docs.rs/sysinfo/0.30` immediately. This is where you'll find:
- Every struct, enum, and trait the crate provides
- Every method on each type, with its signature
- Examples embedded in the documentation

### Reading the sysinfo Documentation

`sysinfo` is the crate you'll use to query CPU, memory, and disk metrics. Here's how to navigate it:

1. Go to `https://docs.rs/sysinfo`
2. Look at the **Structs** section. You'll see `System`, `Cpu`, `Disk`, `Networks`
3. Click on `System` - this is the entry point
4. Look at the **Methods** section. You'll see `new()`, `refresh_cpu()`, `global_cpu_info()`, `total_memory()`, etc.
5. Each method shows its signature and a brief description. Click through to see examples

The most important thing to find: **what do I call first, and what does it return?**

For sysinfo:
```rust
use sysinfo::System;

let mut sys = System::new_all();  // creates a System, refreshes all components
sys.refresh_all();                // update all metrics (call this before reading)

// CPU
let cpu_usage = sys.global_cpu_info().cpu_usage();  // returns f32 (percent)

// Memory (values are in bytes)
let total = sys.total_memory();   // u64 bytes
let used  = sys.used_memory();    // u64 bytes
let avail = sys.available_memory(); // u64 bytes

// Disks
use sysinfo::Disks;
let disks = Disks::new_with_refreshed_list();
for disk in &disks {
    let name  = disk.name().to_string_lossy();   // disk device name
    let total = disk.total_space();              // u64 bytes
    let avail = disk.available_space();          // u64 bytes
}
```

You didn't have to guess these - they're all in the docs. The pattern: find the main entry-point type (usually named `System` or `Client` or similar), read its constructor, then its refresh methods, then its getters.

### Local Documentation with `cargo doc`

You can also build documentation locally and open it in a browser:

```bash
cargo doc --open
```

This builds docs for your crate AND all its dependencies, then opens them in your browser. The local docs are identical to docs.rs, but they're always for the exact version in your `Cargo.lock`.

Useful flags:
```bash
cargo doc --open --no-deps    # Only your own crate, not dependencies
cargo doc --document-private-items  # Include private functions (for your own code)
```

### How Crate Docs Are Written

Rust documentation is written as doc comments (`///`) in the source code. When you read a function signature like:

```rust
/// Returns the CPU usage in percent
pub fn cpu_usage(&self) -> f32
```

The `///` comment becomes the documentation. Good crates include:
- A one-line summary
- A longer explanation
- At least one `# Example` section with runnable code
- Notes on edge cases or performance

When a crate's documentation is thin or missing examples, that's a signal to look for an alternative crate or to test your assumptions with a small isolated program before building on it.

### The `#[doc(hidden)]` Pattern

Some types appear in the docs but are marked with `#[doc(hidden)]` - they're implementation details that are technically public (so they compile) but not intended for direct use. You'll see this in macros and proc-macro support types. If you see a type with a mysterious name like `__Implicit` or `HiddenHelper`, it's not for you. Stick to the clearly documented public API.

---

## How It Breaks

**Forgetting `resolver = "2"` in the workspace root.**
The default resolver (`"1"`) has buggy feature unification behavior that causes features to "leak" between crates. Always add `resolver = "2"` to the workspace root. The `edition = "2021"` crates enable this automatically, but it's safer to be explicit.

**Depending on a workspace crate with a git path in a published crate.**
If you publish `monitor-core` to crates.io, all its dependencies must also be on crates.io or specify version numbers. A `path = "../monitor-cli"` dependency will cause `cargo publish` to fail. Local path dependencies are only for workspace use.

**Assuming `cargo test` runs all workspace tests.**
`cargo test` from the workspace root runs tests in all workspace members. But `cargo test` from inside a member's directory only runs that member's tests. When in doubt, run from the workspace root with `cargo test --workspace`.

**Multiple versions of the same crate causing subtle type mismatches.**
If two crates in your workspace expose a type from the same dependency but at different versions, those types are incompatible - `serde::Serialize` from `serde 1.0.100` is not the same trait as from `serde 1.0.193` in the compiler's view. This causes confusing errors like "expected `serde::Serialize`, found `serde::Serialize`". Fix with `cargo update` to unify versions.

**Ignoring `cargo audit` output.**
`cargo audit` sometimes produces warnings for unmaintained crates with no known CVE. It's tempting to ignore them. Don't - unmaintained crates don't get security patches. Find an alternative or pin to a known-good version and monitor it.
