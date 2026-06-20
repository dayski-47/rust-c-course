# Doc 12 - CI/CD Pipeline from Scratch

🟡 A CI pipeline that takes 20 minutes gives developers enough time to switch context and forget what they were doing. A good pipeline gives fast feedback, catches real bugs, and automates the tedious parts of shipping.

This doc builds a production GitHub Actions CI/CD pipeline for taskforge from first principles - not a copy-paste template, but a design with reasons for each choice.

---

## The Goals

A good CI pipeline for a Rust project must:

1. **Give fast feedback on obvious problems** (formatting, compilation) - under 2 minutes
2. **Catch real bugs** (tests, coverage gates, Clippy errors)
3. **Verify cross-platform behavior** (Linux + Windows minimum)
4. **Gate merges on passing checks** - no broken main branch
5. **Automate releases** - no manual `cargo publish` or binary uploads

---

## Pipeline Structure

Stages run in sequence where they depend on each other, in parallel where they don't:

```
push / PR
    │
    ▼
┌──────────────────────────────────────────────────────┐
│  Stage 1: Check (< 2 min)                            │
│  cargo fmt --check  │  cargo clippy  │  cargo check  │
└──────────────────────────────────────────────────────┘
    │ (on success)
    ├─────────────────────────────────────────────────────────────┐
    │                                                             │
    ▼                                                             ▼
┌─────────────────────────┐                           ┌──────────────────────┐
│  Stage 2: Test           │                           │  Stage 2: Cross      │
│  ubuntu + windows        │                           │  musl, aarch64       │
│  cargo test --workspace  │                           │  build check only    │
└─────────────────────────┘                           └──────────────────────┘
    │ (on success)
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Stage 3: Coverage (< 5 min)                             │
│  cargo llvm-cov --fail-under-lines 80                    │
│  Upload to Codecov                                       │
└─────────────────────────────────────────────────────────┘
    │
    ▼  (on push to main only)
┌─────────────────────────────────────────────────────────┐
│  Stage 4: Release                                        │
│  cargo-dist builds binaries for all platforms            │
│  GitHub Release with artifacts                           │
└─────────────────────────────────────────────────────────┘
```

---

## The Complete Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always
  # Treat warnings as errors only in our own crates, not build scripts or proc-macros.
  # CARGO_ENCODED_RUSTFLAGS does not affect build scripts (unlike RUSTFLAGS).
  CARGO_ENCODED_RUSTFLAGS: "-Dwarnings"

jobs:
  # ─── Stage 1: Fast feedback ───────────────────────────────────────────────
  check:
    name: Check + Clippy + Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt

      # Cache compiled dependencies - avoids recompiling on every push
      - uses: Swatinem/rust-cache@v2

      - name: Check Cargo.lock is committed
        run: cargo fetch --locked

      - name: Check documentation builds without warnings
        run: RUSTDOCFLAGS='-Dwarnings' cargo doc --workspace --no-deps

      - name: Check compilation
        run: cargo check --workspace --all-targets --all-features

      - name: Clippy lints
        run: cargo clippy --workspace --all-targets --all-features -- -D warnings

      - name: Formatting
        run: cargo fmt --all -- --check

  # ─── Stage 2: Tests ───────────────────────────────────────────────────────
  test:
    name: Test (${{ matrix.os }})
    needs: check
    strategy:
      fail-fast: false  # Don't cancel Windows when Linux fails or vice versa
      matrix:
        os: [ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2

      - name: Run unit and integration tests
        run: cargo test --workspace

      - name: Run doc tests
        run: cargo test --workspace --doc

  # ─── Stage 2 (parallel): Cross-compilation check ─────────────────────────
  cross:
    name: Cross (${{ matrix.target }})
    needs: check
    strategy:
      matrix:
        include:
          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
          - target: aarch64-unknown-linux-gnu
            os: ubuntu-latest
            use_cross: true
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install cross (if needed)
        if: matrix.use_cross
        run: cargo install cross --git https://github.com/cross-rs/cross

      - name: Cross-compile check
        run: |
          if [ "${{ matrix.use_cross }}" = "true" ]; then
            cross build --target ${{ matrix.target }}
          else
            cargo build --target ${{ matrix.target }}
          fi

  # ─── Stage 3: Coverage ────────────────────────────────────────────────────
  coverage:
    name: Coverage
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: llvm-tools-preview

      - uses: Swatinem/rust-cache@v2

      - name: Install cargo-llvm-cov
        run: cargo install cargo-llvm-cov

      - name: Generate coverage report
        run: cargo llvm-cov --workspace --lcov --output-path lcov.info --fail-under-lines 80

      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: lcov.info
          fail_ci_if_error: true
          token: ${{ secrets.CODECOV_TOKEN }}

  # ─── Stage 4: Release (main branch only) ──────────────────────────────────
  release:
    name: Release
    needs: [test, coverage]
    if: github.ref == 'refs/heads/main' && startsWith(github.ref, 'refs/tags/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2

      - name: Install cargo-dist
        run: cargo install cargo-dist

      - name: Build release artifacts
        run: cargo dist build --tag ${{ github.ref_name }}

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: dist/**
          generate_release_notes: true
```

---

## Caching Strategy

The `Swatinem/rust-cache@v2` action caches compiled dependencies in `~/.cargo`. Without it, every CI run recompiles all crates from scratch - for a project with many dependencies, that's 5–15 minutes wasted per run.

The cache is keyed on `Cargo.lock`. When you update a dependency, the cache misses (correct behavior - the new dependency needs to compile). Between pushes on the same lockfile, the cache hits and the check stage runs in under a minute.

For large monorepos, add `save-if` to avoid flooding the cache with PR caches that are never reused:

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    save-if: ${{ github.ref == 'refs/heads/main' }}
```

This saves cache only from main branch runs. PR builds benefit from the main branch cache but don't pollute it with short-lived branches.

---

## Treating Warnings as Errors

`CARGO_ENCODED_RUSTFLAGS: "-Dwarnings"` causes `cargo check` and `cargo build` to fail if any warning exists in the workspace's own code. This prevents the gradual accumulation of "I'll fix this later" warnings that eventually number in the dozens and get ignored.

Important distinction: `CARGO_ENCODED_RUSTFLAGS` does *not* affect build scripts or proc-macros. This avoids false failures from warnings in third-party macros you can't control. If you want warnings-as-errors in build scripts too, use `RUSTFLAGS="-Dwarnings"` instead, but expect occasional noise from upstream crates.

For Clippy specifically, pass the flag directly to the invocation:

```bash
cargo clippy --workspace --all-targets -- -D warnings
```

---

## The Compile-Fail Tests in CI

The `trybuild` compile-fail tests (doc 11) run as part of `cargo test --workspace` automatically. No special CI step needed - they're normal tests that invoke the compiler on fixture files.

One subtlety: the `.stderr` snapshot files must match the *exact* error message. Error messages sometimes change between Rust versions. When you update the Rust toolchain in CI (via `dtolnay/rust-toolchain@stable`), compile-fail tests may need `.stderr` file updates. This is expected - run `cargo test` locally with the new toolchain and update the snapshots.

---

## Local Development Parity

Everything that runs in CI should be runnable locally in a single command. Add a `Makefile.toml` using `cargo-make`:

```toml
# Makefile.toml
[config]
default_to_workspace = false

[tasks.ci]
description = "Run the full CI pipeline locally"
dependencies = ["fmt-check", "clippy", "test", "coverage"]

[tasks.fmt-check]
command = "cargo"
args = ["fmt", "--all", "--", "--check"]

[tasks.clippy]
command = "cargo"
args = ["clippy", "--workspace", "--all-targets", "--", "-D", "warnings"]

[tasks.test]
command = "cargo"
args = ["test", "--workspace"]

[tasks.coverage]
command = "cargo"
args = ["llvm-cov", "--workspace", "--fail-under-lines", "80"]

[tasks.fix]
description = "Auto-fix formatting and some Clippy lints"
dependencies = ["fmt", "clippy-fix"]

[tasks.fmt]
command = "cargo"
args = ["fmt", "--all"]

[tasks.clippy-fix]
command = "cargo"
args = ["clippy", "--workspace", "--fix", "--allow-dirty"]
```

```bash
cargo install cargo-make

# Run full CI check:
cargo make ci

# Fix formatting and simple Clippy lints:
cargo make fix
```

This makes "does this match CI?" a one-command question.

---

## Pre-Commit Hooks

Catch formatting issues before they hit CI. Configure hooks in `.pre-commit-config.yaml` or use a simple git hook:

```bash
# .git/hooks/pre-commit (or via git config core.hooksPath)
#!/bin/bash
set -e

# Fast check: formatting (takes < 1s)
if ! cargo fmt --all -- --check 2>/dev/null; then
    echo "❌ Formatting issues. Run: cargo fmt --all"
    exit 1
fi

# Fast check: compilation errors only (not tests)
cargo check --workspace --quiet

echo "✅ Pre-commit checks passed"
```

This catches the two most common push-to-CI-then-fail scenarios: formatting and compilation errors.

---

## Nightly Checks: Miri and Sanitizers

Some checks are expensive enough to run on a schedule rather than every push. Set these up on a weekly or nightly cron:

```yaml
# .github/workflows/nightly.yml
name: Nightly

on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM UTC every day
  workflow_dispatch:       # Manual trigger

jobs:
  miri:
    name: Miri (UB detection)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@nightly
        with:
          components: miri

      - name: Run Miri on unsafe code
        run: cargo miri test --workspace

  sanitizers:
    name: Address Sanitizer
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@nightly

      - name: Run with AddressSanitizer
        run: cargo test --workspace
        env:
          RUSTFLAGS: "-Z sanitizer=address"
          RUSTDOCFLAGS: "-Z sanitizer=address"
```

Miri interprets Rust code in a simulated environment that catches undefined behavior: use-after-free, uninitialized memory reads, data races. It's slow (10–100x slower than native), which is why it runs nightly rather than on every PR.

---

## What to Gate and What Not to Gate

**Gate merges on:**
- `cargo fmt -- --check` (formatting is non-negotiable)
- `cargo clippy -- -D warnings` (Clippy catches real bugs)
- `cargo test --workspace` (tests failing means the code is broken)
- Coverage gate if you have a coverage target

**Don't gate merges on:**
- Benchmark performance (regressions may be statistical noise, or the CI machine is slower)
- Nightly-only features (nightly toolchain can break at any time)
- Miri (too slow for per-PR feedback)

The goal of CI gates is: if CI passes, the PR is safe to merge. False failures from benchmark noise or nightly instability train developers to ignore CI results - defeating the purpose.
