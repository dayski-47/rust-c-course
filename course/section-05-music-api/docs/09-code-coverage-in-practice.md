# Doc 09 — Code Coverage in Practice

🟢 Coverage is a tool for finding blind spots, not a number to maximize.

You've written tests for the music API. But how many code paths are those tests actually exercising? A function with three `match` arms might have tests for two of them — the third arm has a bug that's been dormant for six months, waiting for a specific input combination. Code coverage makes these gaps visible.

This doc covers the three Rust coverage tools, how to read coverage reports, how to enforce thresholds in CI, and the testing strategy that makes coverage numbers meaningful.

---

## What Coverage Measures (and What It Doesn't)

Coverage tells you **which lines, branches, and functions your tests executed**. It does not tell you whether those tests are correct. A covered line can still contain a bug.

The most useful coverage metric is **branch coverage** — which arms of `if`/`match` were actually taken. Line coverage (was this line hit?) hides the danger: a function can be 100% line-covered but 50% branch-covered, meaning half your conditional logic is untested.

```rust
// 100% line coverage, 50% branch coverage — still dangerous
pub fn categorize_duration(ms: u32) -> &'static str {
    if ms < 30_000 {          // tested (short song)
        "short"
    } else if ms < 300_000 {  // tested (normal song)
        "normal"
    } else if ms < 1_800_000 { // NEVER TESTED — what happens here?
        "long"
    } else {
        "audiobook"           // also never tested
    }
}
```

A test file that only calls `categorize_duration(10_000)` and `categorize_duration(120_000)` gives 100% line coverage (all four return statements are "reachable"), but the `long` and `audiobook` branches are dead code as far as tests are concerned.

---

## Tool 1: `cargo-llvm-cov` (Recommended)

Source-based coverage via LLVM — the most accurate tool available. It instruments the code at compile time, so the measurements are precise:

```bash
# Install
cargo install cargo-llvm-cov
rustup component add llvm-tools-preview

# Run tests and show per-file coverage summary
cargo llvm-cov

# Generate an HTML report (line-by-line highlighting — best for local debugging)
cargo llvm-cov --html
# Output: target/llvm-cov/html/index.html — open in a browser

# LCOV format (for CI integration with Codecov, Coveralls, etc.)
cargo llvm-cov --lcov --output-path lcov.info

# Cover the whole workspace
cargo llvm-cov --workspace

# Fail if coverage drops below threshold (for CI gates)
cargo llvm-cov --workspace --fail-under-lines 80
```

**Reading the HTML report:**

```
Filename              │ Functions │ Lines  │ Branches
───────────────────────┼───────────┼────────┼─────────
src/routes/tracks.rs  │   91.2%   │ 94.1%  │  78.3%   ← branch gap to investigate
src/validation.rs     │  100.0%   │ 100.0% │  96.4%
src/db/queries.rs     │   67.5%   │ 72.0%  │  55.8%   ← low coverage here
src/middleware.rs     │   80.0%   │ 83.3%  │  66.7%
```

Green = covered, Red = not covered, Yellow = partially covered (some branches taken, others not).

**Coverage types explained:**

| Type | What It Measures | Why It Matters |
|------|-----------------|----------------|
| Line | Which source lines were executed | Basic "was this code reached?" |
| Branch | Which `if`/`match` arms were taken | Catches untested conditions |
| Function | Which functions were called | Finds dead code |
| Region | Sub-expression granularity | Most precise |

Always look at **branch coverage** first. Line coverage alone is misleading.

---

## Tool 2: `cargo-tarpaulin` (Linux Only, Quick Setup)

Tarpaulin uses `ptrace` rather than compile-time instrumentation. Less accurate than llvm-cov, but zero setup:

```bash
cargo install cargo-tarpaulin

# Basic
cargo tarpaulin

# HTML output
cargo tarpaulin --out Html

# Workspace-wide with XML output for CI
cargo tarpaulin \
    --workspace \
    --timeout 120 \
    --out Xml \
    --output-dir coverage/ \
    --exclude-files "*/tests/*" "*/benches/*"
```

**When to use each tool:**

| Feature | cargo-llvm-cov | cargo-tarpaulin |
|---------|---------------|-----------------|
| Accuracy | ✅ Source-based | ⚠️ ptrace (some overcounting) |
| Platform | ✅ Any | Linux only |
| Branch coverage | ✅ Yes | Limited |
| Doc tests | ✅ Yes | No |
| Setup complexity | Moderate (`llvm-tools-preview`) | Minimal |
| Speed | Faster | Slower |

Recommendation: use `llvm-cov` for CI and accuracy. Use `tarpaulin` on Linux for a quick local check without installing LLVM tools.

---

## Coverage in CI: GitHub Actions + Codecov

The right CI setup: generate coverage on every push, upload to Codecov for tracking, and fail the build if coverage drops below your threshold.

```yaml
# .github/workflows/coverage.yml
name: Code Coverage

on:
  push:
    branches: [main]
  pull_request:

jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: llvm-tools-preview

      - name: Install cargo-llvm-cov
        uses: taiki-e/install-action@cargo-llvm-cov

      - name: Generate coverage report
        run: cargo llvm-cov --workspace --lcov --output-path lcov.info

      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: lcov.info
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

      - name: Enforce coverage threshold
        run: cargo llvm-cov --workspace --fail-under-lines 80
        # Fails the build if line coverage drops below 80%
```

**Coverage gates** — the threshold you enforce in CI:

```bash
# Multiple threshold types
cargo llvm-cov --workspace --fail-under-lines 80       # >= 80% line coverage
cargo llvm-cov --workspace --fail-under-functions 70   # >= 70% function coverage
cargo llvm-cov --workspace --fail-under-regions 60     # >= 60% region coverage
```

The music API at 80% line coverage is a reasonable starting point. Aim for higher branch coverage (85%+) on validation logic and error handling paths.

---

## Coverage-Guided Testing Strategy

Coverage numbers tell you WHERE the gaps are. The strategy tells you WHAT to do about them.

**Triage by risk, not by number:**

```
High coverage + high risk → ✅ Good — maintain it, add edge cases
High coverage + low risk  → 🔄 Possibly over-tested — these tests may be low value
Low coverage + high risk  → 🔴 WRITE TESTS NOW — bugs are hiding here
Low coverage + low risk   → 🟡 Track, but don't panic
```

**High-risk code in the music API:**
- Validation logic (`src/validation.rs`) — every error path needs a test
- Database query error handling — what happens when the connection drops?
- Pagination edge cases — page 0, page beyond total count, empty result set
- Search query building — invalid inputs, SQL injection attempts
- Middleware — authentication failure paths

**Step 1: Find the low-coverage branches**

Open the HTML report and look for red and yellow lines. Focus on:
- Error handling branches — `Err(_) => { ... }` arms
- Range checks — the boundary conditions (`>`, `<`, `==`)
- None handling — `None => { ... }` arms in `match`
- Early returns in complex functions

**Step 2: Write tests that specifically target gaps**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Happy path — probably already tested
    #[test]
    fn create_track_success() {
        let body = CreateTrackBody {
            title: "Yesterday".to_string(),
            duration_ms: 125_000,
            artist: "The Beatles".to_string(),
            genre: Some("rock".to_string()),
        };
        assert!(ValidatedTrack::try_from(RawTrackRequest::from(body)).is_ok());
    }

    // Branch coverage: empty title (the Err branch of title validation)
    #[test]
    fn create_track_empty_title() {
        let body = CreateTrackBody {
            title: "   ".to_string(),  // whitespace-only — triggers TitleError::Empty
            duration_ms: 125_000,
            artist: "The Beatles".to_string(),
            genre: None,
        };
        assert!(matches!(
            ValidatedTrack::try_from(RawTrackRequest::from(body)),
            Err(ValidationError::Title(TitleError::Empty))
        ));
    }

    // Branch coverage: exactly at the max title length boundary
    #[test]
    fn create_track_title_at_max_length() {
        let long_title = "a".repeat(500);
        // Should succeed — 500 chars is the max
        assert!(TrackTitle::try_from(long_title).is_ok());
    }

    #[test]
    fn create_track_title_over_max_length() {
        let too_long = "a".repeat(501);
        // Should fail — 501 chars exceeds the max
        assert!(matches!(
            TrackTitle::try_from(too_long),
            Err(TitleError::TooLong(501))
        ));
    }

    // Branch coverage: zero duration
    #[test]
    fn create_track_zero_duration() {
        assert!(matches!(
            TrackDuration::try_from(0i64),
            Err(DurationError::NonPositive(0))
        ));
    }

    // Branch coverage: negative duration
    #[test]
    fn create_track_negative_duration() {
        assert!(matches!(
            TrackDuration::try_from(-1i64),
            Err(DurationError::NonPositive(-1))
        ));
    }

    // Branch coverage: duration too long
    #[test]
    fn create_track_too_long_duration() {
        let too_long = 24 * 60 * 60 * 1000 + 1;
        assert!(matches!(
            TrackDuration::try_from(too_long),
            Err(DurationError::TooLong(_))
        ));
    }

    // Branch coverage: unknown genre
    #[test]
    fn create_track_unknown_genre() {
        assert!(Genre::try_from("ska".to_string()).is_err());
    }
}
```

**Step 3: Property-based testing for finding hidden edge cases**

`proptest` generates thousands of random inputs automatically — it finds the edge cases your hand-written tests miss:

```toml
[dev-dependencies]
proptest = "1"
```

```rust
use proptest::prelude::*;

proptest! {
    // Property: parsing should never panic, only succeed or return Err
    #[test]
    fn title_validation_never_panics(input in "\\PC*") {
        let _ = TrackTitle::try_from(input);
    }

    // Property: valid durations should always round-trip through as_display
    #[test]
    fn duration_display_is_consistent(ms in 1u32..=86_400_000u32) {
        let duration = TrackDuration::try_from(ms as i64).unwrap();
        // The display should always be a valid MM:SS or H:MM:SS format
        let display = duration.as_display();
        assert!(display.contains(':'));
    }

    // Property: valid titles preserve their content
    #[test]
    fn valid_title_preserves_non_whitespace_content(
        prefix in "\\PC{1,100}",
        suffix in "\\PC{0,100}"
    ) {
        let s = format!("{prefix}{suffix}");
        if let Ok(title) = TrackTitle::try_from(s.clone()) {
            // Trimmed content should be preserved
            assert_eq!(title.as_str(), s.trim());
        }
    }
}
```

`proptest` is especially powerful for parser and validation code. It's found real bugs in production Rust libraries by generating inputs that human test writers wouldn't think to try.

---

## Excluding Code from Coverage

Not all code needs coverage. Exclude noise:

```bash
# Exclude test files themselves (always "covered" — they're what runs)
cargo llvm-cov --workspace --ignore-filename-regex 'tests?\.rs$'

# Exclude generated code
cargo llvm-cov --workspace --ignore-filename-regex 'target/'

# Exclude benchmarks
cargo llvm-cov --workspace --ignore-filename-regex 'benches/'
```

In code, mark paths that require hardware or specific environments:

```rust
// For paths that only run on specific hardware — mark as intentionally uncovered
fn detect_audio_hardware() -> Option<AudioDevice> {
    #[cfg(not(test))]  // Only run detection in non-test builds
    {
        // This path requires actual audio hardware — don't count against coverage
        probe_alsa_devices()
    }
    #[cfg(test)]
    {
        None  // Test builds get None instead of hardware probe
    }
}
```

---

## Snapshot Testing with `insta`

For large structured outputs (JSON responses, error messages, reports), snapshot testing is faster to write and maintain than hand-coded assertions:

```toml
[dev-dependencies]
insta = { version = "1", features = ["json"] }
```

```rust
#[test]
fn track_list_response_format() {
    let tracks = vec![
        Track { id: 1, title: "Yesterday".to_string(), duration_ms: 125_000, /* ... */ },
        Track { id: 2, title: "Hey Jude".to_string(), duration_ms: 431_000, /* ... */ },
    ];
    
    let response = format_track_list(&tracks);
    
    // First run: creates a snapshot file in snapshots/ directory.
    // Subsequent runs: compares against the snapshot.
    // Run `cargo insta review` to accept intentional changes interactively.
    insta::assert_json_snapshot!(response);
}
```

When the response format changes intentionally, `cargo insta review` shows you what changed and lets you accept or reject the diff. When it changes accidentally, the test fails and the diff points to exactly what broke.

---

## How It Breaks

**Coverage masks bugs in branches you didn't test.**
A test that calls `search_tracks("beatles")` might hit the happy path and nothing else. The "no results found" branch, the "database error" branch, and the "malformed query" branch are all uncovered — even though the function itself is 100% line-covered. Always check branch coverage, not just line coverage.

**`cargo tarpaulin` overcounting on macros.**
Tarpaulin sometimes reports macro-expanded code as covered even when it wasn't actually executed. If your coverage numbers look suspiciously high, run `llvm-cov` to get the accurate picture.

**Setting the threshold too low and never raising it.**
Starting at 80% is fine. But if coverage drifts down to 70% and you just update the threshold, you've defeated the purpose. The threshold should only move upward. When coverage drops, write tests — don't lower the gate.

**Coverage-driven test writing missing semantics.**
Writing tests purely to hit uncovered lines produces tests that cover code but don't verify behavior. A test that calls `categorize_duration(100_000)` to hit the `"long"` branch covers the line, but if the function returns `"normal"` instead of `"long"`, the test passes. Coverage tells you what was touched. Tests tell you what was verified. You need both.
