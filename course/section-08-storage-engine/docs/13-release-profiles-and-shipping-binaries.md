# Doc 13 - Release Profiles and Shipping Binaries

🟡 The difference between a debug build and a well-tuned release build is typically 3–5× faster runtime and 60–80% smaller binary. For ironkv - a storage engine that processes millions of operations - that gap matters.

This doc covers the knobs in Cargo's release profiles, how to measure binary size, how to strip unused code, and how to produce distributable binaries.

---

## Release Profile Anatomy

The default `cargo build --release` is conservative. Cargo's defaults balance compile time against optimization:

```toml
# What you get by default with [profile.release]
opt-level = 3        # Aggressive optimization
lto = false          # Link-time optimization OFF (faster compile)
codegen-units = 16   # Parallel compilation (faster compile, less optimization opportunity)
panic = "unwind"     # Stack unwinding (larger binary, but catch_unwind works)
strip = "none"       # Keep all symbols (useful for debugging crashes)
overflow-checks = false
debug = false
```

For ironkv production builds:

```toml
# Cargo.toml
[profile.release]
lto = true           # Cross-crate optimization - biggest performance win
codegen-units = 1    # Single codegen unit - lets LTO see everything
panic = "abort"      # No unwinding overhead; slightly smaller binary
strip = true         # Remove symbols - 50-70% binary size reduction
overflow-checks = true   # Keep overflow checks - safety > micro-optimization
```

**The impact of each setting:**

| Setting | Binary Size | Runtime Speed | Compile Time |
|---------|-------------|---------------|--------------|
| `lto = false → true` | −10 to −20% | +5 to +20% | 2–5× longer |
| `codegen-units = 16 → 1` | −5 to −10% | +5 to +10% | 1.5–2× longer |
| `panic = unwind → abort` | −5 to −10% | Negligible | Negligible |
| `strip = none → true` | −50 to −70% | None | Negligible |

The combination of LTO + single codegen unit + strip typically produces a binary 40–60% smaller and 15–30% faster than the default release build, at the cost of 3–5× longer compile time. This is the right trade-off for shipping to production.

---

## LTO Variants

Three levels of link-time optimization, with different trade-offs:

```toml
# Full LTO: maximum optimization, slowest compile
[profile.release]
lto = true

# Thin LTO: most of the benefit, much faster than full
[profile.release]
lto = "thin"

# No LTO: fastest compile, least optimization
[profile.release]
lto = false
```

**Recommendation for ironkv:**
- Development: `lto = false` (fast iteration)
- Release builds in CI: `lto = "thin"` (good optimization, reasonable CI time)
- Tagged releases for distribution: `lto = true` (maximum optimization, slower CI acceptable)

Cross-language LTO (between Rust and C) is possible when the zstd compression feature is enabled, but requires matching LLVM versions between rustc and the C compiler - not worth the complexity for ironkv.

---

## Per-Crate Profile Overrides

For debug builds, you want fast iteration on your code but don't mind if dependencies are slow to compile once:

```toml
# Cargo.toml
# Development: optimize all dependencies, keep your own code debug-friendly
[profile.dev.package."*"]
opt-level = 2  # Optimize all dependencies in dev mode (not your crate)

# Release: maximize optimization for the JSON parser specifically
[profile.release.package.serde_json]
opt-level = 3
codegen-units = 1
```

The `[profile.dev.package."*"]` trick is high-value: dependencies like `serde_json`, `regex`, and `zstd` compile in optimized form during development, making debug builds of ironkv run 3–10× faster without slowing your own code's recompilation.

---

## Binary Size Analysis

Measure before optimizing. `cargo-bloat` shows what's taking space:

```bash
cargo install cargo-bloat

# Analyze the release binary
cargo bloat --release

# Output:
# File  .text
#  6.3%   5.7%   serde_json::ser::Serializer::collect_str
#  5.1%   4.6%   ironkv::storage::write_entry
#  3.8%   3.4%   ironkv::index::BTreeMap::insert
#  ...

# Show functions from a specific crate
cargo bloat --release --crates

# Output:
# File  .text  Name
# 18.3%  16.5%  ironkv
#  9.2%   8.3%  serde_json
#  6.1%   5.5%  zstd
#  ...
```

The `.text` column is the compiled code size. If `serde_json` takes 8% of your binary but you're only using it for debug output, that's a candidate for removal or feature-gating.

---

## Removing Dead Code and Unused Dependencies

```bash
# Find unused dependencies
cargo install cargo-udeps
cargo +nightly udeps --workspace

# Alternative: cargo-machete (doesn't require nightly)
cargo install cargo-machete
cargo machete

# Find dead code
cargo clean
cargo build --release 2>&1 | grep "warning: unused"
```

Unused dependencies are silent binary bloat. A dependency you added three months ago and stopped using still compiles and links into the binary. `cargo-udeps` or `cargo-machete` catch them.

For ironkv, common culprits:
- Test-only crates that ended up in `[dependencies]` instead of `[dev-dependencies]`
- Debug utilities left in after resolving a production issue
- Optional features that are always disabled but still listed as non-optional

---

## Size-Optimized Profile for Distribution

When binary size is the primary concern (embedded deployments, WASM targets):

```toml
# Cargo.toml
[profile.release-tiny]
inherits = "release"
opt-level = "z"       # Optimize aggressively for size
lto = true
codegen-units = 1
panic = "abort"
strip = true

# Use: cargo build --profile release-tiny
```

`opt-level = "z"` is more aggressive than `"s"` - it may disable some loop unrolling and inlining that would make code larger. For a storage engine, this typically costs 5–15% runtime throughput to achieve another 10–20% size reduction.

Don't use `opt-level = "z"` for the default release build. It's for specific size-constrained deployment contexts.

---

## Shipping Statically Linked Binaries

The safest distribution format for Linux: a statically linked musl binary that runs on any Linux distribution regardless of glibc version:

```bash
# Build for musl - produces a static binary
cargo build --release --target x86_64-unknown-linux-musl

# Verify static linking
ldd target/x86_64-unknown-linux-musl/release/ironkv
# Output: not a dynamic executable

# Check the final size
ls -lh target/x86_64-unknown-linux-musl/release/ironkv
```

Note: if ironkv links against native zstd (not the Rust port), the musl build requires a statically compiled zstd library. In this case, either use the pure-Rust `zstd` crate (which compiles with cargo, no system dependency), or build a static zstd and point the linker at it via `build.rs`.

---

## Automated Releases with `cargo-dist`

`cargo-dist` builds release binaries for all target platforms and creates GitHub Releases automatically:

```bash
cargo install cargo-dist

# Initialize cargo-dist configuration
cargo dist init

# This modifies Cargo.toml with:
# [workspace.metadata.dist]
# cargo-dist-version = "0.x.y"
# targets = ["x86_64-pc-windows-msvc", "x86_64-unknown-linux-musl", "x86_64-apple-darwin"]
# installers = ["shell", "powershell"]
```

In CI, triggered by a version tag:

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  dist:
    name: Build and release
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2

      - name: Install cargo-dist
        run: cargo install cargo-dist

      - name: Build release artifacts
        run: cargo dist build --tag ${{ github.ref_name }} --output-format=json

      - name: Create GitHub Release
        run: cargo dist publish --tag ${{ github.ref_name }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

When you push `v0.3.0`, CI builds binaries for Linux (musl), macOS (Intel + ARM), and Windows, creates a GitHub Release, and attaches the artifacts. Users download and run without installing Rust.

---

## The Release Checklist

Before tagging a release:

- [ ] `cargo test --workspace` passes
- [ ] `cargo clippy --workspace -- -D warnings` passes
- [ ] `cargo build --release --target x86_64-unknown-linux-musl` succeeds
- [ ] Binary size is within expectations (`cargo bloat --release`)
- [ ] No unused dependencies (`cargo machete`)
- [ ] CHANGELOG.md updated with this version's changes
- [ ] Version bumped in `Cargo.toml` and `Cargo.lock` is committed

A versioned release is a promise: users who pin to `ironkv 0.3.0` expect it to behave consistently. Test before tagging. The musl build check catches the common case where a dependency added native linking that breaks static builds.
