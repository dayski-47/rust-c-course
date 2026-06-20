# Doc 14 — Tricks from the Trenches

🟡 These are the patterns that don't fit neatly into any chapter but come up repeatedly in production Rust codebases. Each one is self-contained — read them in any order.

By section 8, you've built production-grade async services, type-safe APIs, and a storage engine with unsafe internals. The techniques in this doc are the accumulated friction-reduction that experienced Rust engineers apply without thinking. Each one solves a real problem you'll encounter.

---

## 1. The `#![deny(warnings)]` Trap

**Problem:** Adding `#![deny(warnings)]` to `lib.rs` causes builds to break when Clippy adds new lints. Your code that compiled yesterday fails today when you update Rust.

```rust
// ❌ Don't put this in source code
#![deny(warnings)]
```

**Fix:** Use `CARGO_ENCODED_RUSTFLAGS` in CI instead. This treats warnings as errors only in your crate (not build scripts or proc-macros), and only in CI:

```yaml
# .github/workflows/ci.yml
env:
  CARGO_ENCODED_RUSTFLAGS: "-Dwarnings"
```

Or use workspace lints for finer control:

```toml
# Cargo.toml
[workspace.lints.rust]
unused_imports = "deny"
unused_variables = "warn"

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
```

`CARGO_ENCODED_RUSTFLAGS` (unlike `RUSTFLAGS`) only applies to the workspace's own code — not to build scripts or proc-macros from third parties. This avoids false failures when a dependency's build script has a new warning.

---

## 2. Debug Builds with Optimized Dependencies

**Problem:** Debug builds of ironkv are painfully slow because serde, zstd, and other dependencies run unoptimized.

**Fix:** Optimize all dependencies in dev mode without affecting your own code's debuggability:

```toml
# Cargo.toml
[profile.dev.package."*"]
opt-level = 2  # Optimize all dependencies in dev builds
```

First build with this setting takes a few minutes longer. All subsequent builds are fast because dependencies are cached. The runtime performance of debug builds improves dramatically — serde in particular is 10–50× faster optimized than debug. You keep full debugging symbols in your own code.

---

## 3. Commit `Cargo.lock` for Binaries, Not Libraries

| Crate type | Commit `Cargo.lock`? | Why |
|------------|---------------------|-----|
| Binary / application | **Yes** | Reproducible builds — every CI run uses identical deps |
| Library (published) | **No** | Let downstream projects choose compatible versions |
| Workspace with both | **Yes** | The binary crates win |

Add a CI check that fails when `Cargo.lock` is out of date:

```yaml
- name: Check Cargo.lock is current
  run: cargo fetch --locked
```

`cargo fetch --locked` fails if `Cargo.lock` doesn't match `Cargo.toml`. This catches the case where someone bumped a version in `Cargo.toml` but forgot to run `cargo update` before committing.

---

## 4. CI Cache Thrashing

**Problem:** `Swatinem/rust-cache@v2` saves a new cache entry on every PR branch. With 50 active PRs, you have 50 cache entries that are never reused (each PR branch is unique). Restore times grow as the cache fills with stale data.

**Fix:** Only save cache from `main`:

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    save-if: ${{ github.ref == 'refs/heads/main' }}
```

PR builds still benefit from the `main` branch cache — they restore from it but don't save their own. The cache stays small and fast.

For workspaces with multiple targets, add a `shared-key` so caches from different target configurations don't interfere:

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    shared-key: "ci-${{ matrix.target }}"
    save-if: ${{ github.ref == 'refs/heads/main' }}
```

---

## 5. `RUSTFLAGS` vs `CARGO_ENCODED_RUSTFLAGS`

Both set flags passed to `rustc`. The difference:

- `RUSTFLAGS` applies to **everything**: your code, build scripts, proc-macros, dependencies
- `CARGO_ENCODED_RUSTFLAGS` applies **only to the workspace crates** (top-level crates), not to build scripts or proc-macros

For `-Dwarnings`, always use `CARGO_ENCODED_RUSTFLAGS`:

```yaml
# ❌ Can fail because serde_derive's build.rs has a new warning
RUSTFLAGS: "-Dwarnings"

# ✅ Only applies to ironkv's own crates
CARGO_ENCODED_RUSTFLAGS: "-Dwarnings"
```

`CARGO_ENCODED_RUSTFLAGS` uses a special encoding (flags separated by `\x1f`) to handle flags with spaces correctly. Most common flags work as expected.

---

## 6. Pattern: `todo!()` as a Compilation Gate

When designing an interface before implementing it, use `todo!()` to make the code compile while leaving explicit markers of missing implementation:

```rust
impl IronKV {
    pub fn compact(&mut self) -> Result<(), Error> {
        todo!("compaction: merge small segment files into large ones")
    }

    pub fn snapshot(&self) -> Result<Snapshot, Error> {
        todo!("snapshot: copy-on-write read view at current sequence number")
    }
}
```

`todo!()` is distinct from `unimplemented!()`:
- `todo!()`: "I know what this should do, I haven't done it yet"
- `unimplemented!()`: "I'm not sure if this should ever be called"

Both panic at runtime. `todo!()` is the right choice during implementation. Before shipping, a CI check that greps for `todo!()` ensures they all get resolved:

```yaml
- name: Check for unresolved todo!()
  run: |
    if grep -r "todo!" src/; then
      echo "ERROR: Unresolved todo!() in source"
      exit 1
    fi
```

---

## 7. Feature Flags That Affect Behavior Are Hard to Test

**Problem:** A feature flag that changes algorithm behavior (e.g., `compression`) means you need to test both the compressed and uncompressed paths. `cargo test` only runs with the default features.

**Fix:** Test explicitly with all relevant feature combinations:

```yaml
# In CI
- name: Test with compression feature
  run: cargo test --workspace --features compression

- name: Test with all features
  run: cargo test --workspace --all-features

- name: Test with no features
  run: cargo test --workspace --no-default-features
```

Or use `cargo-hack` to automate this:

```bash
cargo hack test --each-feature --workspace
```

---

## 8. The `eprintln!` Debugging Hack

When you need a quick debug print in a complex async or storage context where a proper debugger is hard to attach:

```rust
// Quick: shows value with its Debug format
eprintln!("[DEBUG {}:{}] value = {:?}", file!(), line!(), my_value);

// Better: log to stderr with context
eprintln!("[DEBUG] {:?} at seq={}", entry, sequence_number);
```

`eprintln!` writes to stderr, which doesn't interfere with stdout-based output and doesn't buffer like `println!`. For async code where the panic location is hard to track, `eprintln!` statements that fire before the panic often reveal the cause faster than a debugger.

Always remove `eprintln!` debug statements before committing. The workspace lint `unused_must_use = "deny"` doesn't catch them — use a pre-commit hook:

```bash
# .git/hooks/pre-commit
if grep -r 'eprintln!\|dbg!' src/; then
    echo "Found debug prints — remove before committing"
    exit 1
fi
```

---

## 9. `Rc` vs `Arc`: Know When to Upgrade

`Rc<T>` is a single-threaded reference-counted pointer — cheaper than `Arc<T>` because it doesn't use atomic operations. For ironkv's internal index structures (which run on a single thread), `Rc` is appropriate and measurably faster.

```rust
use std::rc::Rc;

// Single-threaded — Rc is fine and faster
struct IndexNode {
    parent: Option<Rc<IndexNode>>,
    // ...
}
```

When you add async or threading and the compiler complains `Rc is not Send`, that's the signal to upgrade to `Arc`. Don't proactively use `Arc` everywhere — it's unnecessary overhead for single-threaded structures.

The compiler error is explicit:
```
error[E0277]: `Rc<IndexNode>` cannot be sent between threads safely
```

At that point, change `Rc` to `Arc` in the affected type. Nowhere else.

---

## 10. Using `cargo-audit` for Supply Chain Safety

```bash
cargo install cargo-audit

# Check all dependencies for known vulnerabilities
cargo audit

# In CI — fail the build on any advisory
cargo audit --deny warnings
```

Add to CI:

```yaml
- name: Security audit
  run: cargo audit --deny warnings
```

`cargo audit` checks against the RustSec advisory database. When a dependency has a known vulnerability, it reports the advisory, the affected versions, and the fix version. For ironkv, the most likely advisories are in the zstd C library bindings or in cryptographic dependencies (if encryption is enabled).

---

## 11. The Shrinking Refactor: How to Remove Code Safely

When removing a feature or simplifying ironkv, work in this order:

1. **Add a deprecation notice** — mark the old API `#[deprecated]` and compile; everything still works
2. **Update call sites** — change all callers to the new API; the deprecated items are now unused
3. **Remove the deprecated items** — the compiler tells you if you missed any callers
4. **Run tests** — confirm nothing broke

Never delete code without checking that its callers still exist. The `#[deprecated]` + compiler-found-usages pattern is safer than grep.

---

## 12. Workspace `Cargo.toml` as the Source of Truth

Use the workspace `Cargo.toml` for all shared configuration. It's the single place to make workspace-wide changes:

```toml
# Cargo.toml (workspace root)
[workspace]
members = ["ironkv-core", "ironkv-cli"]

# Version all workspace crates together
[workspace.package]
version = "0.3.0"
edition = "2021"
license = "MIT OR Apache-2.0"
authors = ["Your Name"]
repository = "https://github.com/you/ironkv"

# All deps from one place — no version duplication
[workspace.dependencies]
bytes = "1"
serde = { version = "1", features = ["derive"] }
thiserror = "1"
zstd = { version = "0.13", optional = true }

[workspace.lints.rust]
unsafe_code = "warn"

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
```

Member crates inherit version from workspace:

```toml
# ironkv-core/Cargo.toml
[package]
name = "ironkv-core"
version.workspace = true    # inherits 0.3.0
edition.workspace = true    # inherits 2021
license.workspace = true    # inherits "MIT OR Apache-2.0"

[dependencies]
bytes = { workspace = true }          # inherits version "1"
serde = { workspace = true }          # inherits version + features

[lints]
workspace = true                      # inherits all lint configuration
```

Bumping the version or updating a dependency becomes a one-line change in the workspace `Cargo.toml` instead of a change to every member crate.

---

## 13. The `Makefile.toml` Shortcut

```toml
# Makefile.toml (requires cargo-make)
[tasks.bench]
command = "cargo"
args = ["bench", "--", "--baseline", "main"]

[tasks.audit]
command = "cargo"
args = ["audit", "--deny", "warnings"]

[tasks.udeps]
command = "cargo"
args = ["+nightly", "udeps", "--workspace"]

[tasks.ci]
dependencies = ["fmt-check", "clippy", "test", "audit"]
```

```bash
cargo install cargo-make
cargo make ci      # Run the full CI check locally
cargo make bench   # Run benchmarks with baseline comparison
```

`cargo make ci` being exactly what CI runs is the goal. "It works on my machine" stops being a valid excuse when your machine runs the same commands as CI.
