# Doc 07 — Const Fn and Compile-Time Evaluation

🟠 The compiler can run your code before your program exists. Use it to make invariants that can't be violated at runtime, because they're checked before compilation succeeds.

Embedded systems developers and protocol implementors deal with a class of bugs that are entirely preventable: memory maps that overlap, protocol frame sizes that don't fit in the allocated buffer, timeout values that exceed the maximum the hardware supports. In C, these are `#define` constants with no structural relationship — change one, forget to update another, and you get a crash that only happens under production load.

Rust's `const fn` lets you write functions that the compiler executes at compile time. When a `const fn` panics, the panic becomes a compile error. This turns the compiler into a proof engine: the program cannot build unless your invariants hold.

---

## The Difference Between `const` and `const fn`

```rust
// const: a fixed value, evaluated at compile time
const MAX_JOBS: u32 = 1000;

// const fn: a function that CAN run at compile time (also can run at runtime)
const fn jobs_per_worker(total: u32, workers: u32) -> u32 {
    assert!(workers > 0, "division by zero");
    total / workers
}

// Use const fn to compute constants — compiler runs the function:
const DEFAULT_WORKER_LOAD: u32 = jobs_per_worker(MAX_JOBS, 4);  // = 250, at compile time

// Can also call at runtime — same function, same behavior:
fn dynamic_load(total: u32, workers: u32) -> u32 {
    jobs_per_worker(total, workers)
}
```

`const fn` functions can be called in constant contexts (e.g., `const X: T = ...`). When they are, the compiler evaluates them fully. A panic in that evaluation is a compile error with the panic message as the error text.

---

## Turning Invariants into Compile Errors

The key insight: `assert!()` inside a `const fn` is a **proof obligation**. If the assertion fails during compile-time evaluation, the compiler rejects the program.

```rust
pub const fn checked_add(a: u64, b: u64) -> u64 {
    assert!(a <= u64::MAX - b, "overflow");
    a + b
}

// ✅ Fine — 1000 + 500 fits in u64
const TOTAL_CAPACITY: u64 = checked_add(1000, 500);

// ❌ Compile error: "overflow"
// const BROKEN: u64 = checked_add(u64::MAX, 1);
```

---

## taskforge: Validating Worker Configuration at Compile Time

taskforge has configuration constants that must satisfy relationships. Without `const fn`, these are checked at startup (runtime). With `const fn`, they're checked at build time.

```rust
// Configuration constants for the task worker
pub const WORKER_CONCURRENCY: u32 = 16;
pub const JOB_TIMEOUT_SECS: u64 = 300;       // 5 minutes max
pub const REDIS_POOL_SIZE: u32 = 20;
pub const MAX_RETRY_ATTEMPTS: u32 = 5;
pub const RETRY_BACKOFF_SECS: u64 = 30;

// Constraint: pool must be able to serve all concurrent workers plus admin headroom
const fn assert_pool_sufficient(pool_size: u32, concurrency: u32) {
    assert!(
        pool_size >= concurrency + 4,
        "Redis pool must have at least concurrency + 4 connections (4 headroom for admin/monitor)",
    );
}

// Constraint: retry backoff times must not exceed job timeout
const fn assert_retry_budget(
    max_retries: u32,
    backoff_secs: u64,
    timeout_secs: u64,
) {
    let total_backoff = max_retries as u64 * backoff_secs;
    assert!(
        total_backoff < timeout_secs,
        "total retry backoff exceeds job timeout — jobs may never retry before expiring",
    );
}

// These run at compile time. If REDIS_POOL_SIZE < WORKER_CONCURRENCY + 4, the crate won't build.
const _: () = assert_pool_sufficient(REDIS_POOL_SIZE, WORKER_CONCURRENCY);
const _: () = assert_retry_budget(MAX_RETRY_ATTEMPTS, RETRY_BACKOFF_SECS, JOB_TIMEOUT_SECS);
```

The `const _: () = ...` idiom evaluates a constant expression and discards the result. It's the standard way to run compile-time assertions that don't produce a value.

Try changing `REDIS_POOL_SIZE` to 10 — the crate won't compile, and the error message tells you exactly why:

```text
error[E0080]: evaluation of constant value failed
  --> src/config.rs:42:5
   |
42 |     assert!(pool_size >= concurrency + 4, "Redis pool must have at least ...");
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |     the evaluated program panicked at 'Redis pool must have at least concurrency + 4 connections...'
```

---

## Compile-Time Lookup Tables

`const fn` can build lookup tables that the compiler evaluates once and bakes into the binary. No heap allocation, no lazy initialization, no runtime cost:

```rust
/// Maps job priority (0-255) to the Redis queue name index.
/// Built at compile time — zero runtime cost.
const PRIORITY_QUEUE_MAP: [u8; 256] = {
    let mut map = [0u8; 256];
    let mut i = 0usize;
    while i < 256 {
        map[i] = if i >= 200 { 2 }       // High priority: queue index 2
                 else if i >= 100 { 1 }   // Normal priority: queue index 1
                 else { 0 };              // Low priority: queue index 0
        i += 1;
    }
    map
};

const QUEUE_NAMES: [&str; 3] = ["queue:low", "queue:normal", "queue:high"];

pub fn queue_for_priority(priority: u8) -> &'static str {
    QUEUE_NAMES[PRIORITY_QUEUE_MAP[priority as usize] as usize]
}
```

The array `PRIORITY_QUEUE_MAP` is computed once at build time and placed in the binary's read-only data section. The runtime function is a single array lookup with no branching.

Note the `while` loop instead of `for`: `for` loops over iterators aren't yet stable in `const fn` (as of Rust 1.79). Use `while i < len` instead.

---

## Compile-Time Protocol Frame Validation

taskforge serializes job records into Redis hash fields. The field names are protocol constants. With `const fn`, you can validate them at build time:

```rust
/// Validate that a protocol field name is safe for Redis:
/// - Not empty
/// - No spaces (Redis HSET syntax)
/// - Max 64 bytes
pub const fn validate_field_name(name: &str) -> &str {
    let bytes = name.as_bytes();
    assert!(!bytes.is_empty(), "field name must not be empty");
    assert!(bytes.len() <= 64, "field name must be at most 64 bytes");
    let mut i = 0;
    while i < bytes.len() {
        assert!(bytes[i] != b' ', "field name must not contain spaces");
        assert!(bytes[i] != b'\n', "field name must not contain newlines");
        i += 1;
    }
    name
}

// Protocol constants — validated at compile time
pub const FIELD_STATUS:      &str = validate_field_name("status");
pub const FIELD_JOB_TYPE:    &str = validate_field_name("job_type");
pub const FIELD_PAYLOAD:     &str = validate_field_name("payload");
pub const FIELD_RETRY_COUNT: &str = validate_field_name("retry_count");
pub const FIELD_CREATED_AT:  &str = validate_field_name("created_at");
pub const FIELD_STARTED_AT:  &str = validate_field_name("started_at");
pub const FIELD_WORKER_ID:   &str = validate_field_name("worker_id");

// This would fail to compile:
// pub const FIELD_BAD: &str = validate_field_name("bad field");
//    error: "field name must not contain spaces"
```

---

## Const Generics: Configuration Encoded in Types

Beyond `const fn`, Rust supports *const generics* — type parameters that are constant values rather than types. This enables type-safe capacity enforcement:

```rust
use std::collections::VecDeque;

/// A fixed-capacity job buffer.
/// The capacity N is part of the type — can't accidentally create a size-1 buffer
/// where a size-100 buffer is expected.
pub struct JobBuffer<const N: usize> {
    items: VecDeque<String>,
}

impl<const N: usize> JobBuffer<N> {
    pub const fn new() -> Self {
        assert!(N > 0, "buffer capacity must be positive");
        JobBuffer { items: VecDeque::new() }
    }

    pub fn push(&mut self, job: String) -> bool {
        if self.items.len() >= N {
            return false;  // full
        }
        self.items.push_back(job);
        true
    }

    pub fn pop(&mut self) -> Option<String> {
        self.items.pop_front()
    }

    pub fn len(&self) -> usize { self.items.len() }
    pub fn is_full(&self) -> bool { self.items.len() >= N }
}

// The buffer size is in the type — functions can require specific capacities:
fn process_batch(buf: &mut JobBuffer<100>) {
    // This function requires exactly a 100-slot buffer
    // JobBuffer<50> won't be accepted
}

// Worker with a specific-size pre-fetch buffer
pub struct Worker {
    prefetch: JobBuffer<16>,  // 16 jobs pre-fetched per cycle
}
```

---

## What `const fn` Cannot Do (Yet)

Not everything is available in `const fn`. As of stable Rust 1.79:

| Works | Doesn't work yet |
|-------|-----------------|
| `if`/`else`, `loop`, `while`, `match` | `for` loops over iterators |
| Arithmetic, comparisons | Floating-point arithmetic |
| `assert!`, `panic!` | Heap allocation (`Vec`, `Box`, `String`) |
| Array/slice indexing | Trait objects (`dyn Trait`) |
| Calling other `const fn` | `unsafe` blocks (partially available on nightly) |
| Struct construction | `Drop` implementations with side effects |

The limitation on heap allocation is the most impactful. It's why the lookup table example above uses a raw array rather than a `HashMap`. For compile-time data structures that need associative lookup, use sorted arrays and `binary_search`.

---

## `const fn` in Test Assertions

The same `const fn` that validates production constants can validate test fixtures:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // These run at compile time even in test builds
    const _: () = assert_pool_sufficient(50, 16);  // big pool — fine
    // const _: () = assert_pool_sufficient(10, 16);  // would fail to compile

    #[test]
    fn queue_routing_low_priority() {
        assert_eq!(queue_for_priority(0), "queue:low");
        assert_eq!(queue_for_priority(99), "queue:low");
    }

    #[test]
    fn queue_routing_normal_priority() {
        assert_eq!(queue_for_priority(100), "queue:normal");
        assert_eq!(queue_for_priority(199), "queue:normal");
    }

    #[test]
    fn queue_routing_high_priority() {
        assert_eq!(queue_for_priority(200), "queue:high");
        assert_eq!(queue_for_priority(255), "queue:high");
    }
}
```

The compile-time assertions catch configuration bugs. The runtime tests catch routing logic bugs. Together they cover both layers.

---

## C Comparison: `#define` vs `const fn`

In C, compile-time configuration looks like:

```c
#define WORKER_CONCURRENCY  16
#define REDIS_POOL_SIZE     20
// No check that REDIS_POOL_SIZE >= WORKER_CONCURRENCY + 4
// No check that retries won't exceed timeout
// No check that field names are valid Redis identifiers
// Everything discovered at runtime, under load, in production
```

In Rust:

```rust
// Same constants, but with enforced relationships
pub const WORKER_CONCURRENCY: u32 = 16;
pub const REDIS_POOL_SIZE: u32 = 20;
const _: () = assert_pool_sufficient(REDIS_POOL_SIZE, WORKER_CONCURRENCY);
// ↑ If this doesn't hold, the crate doesn't compile.
// The bug is caught by the developer's machine, not in production.
```

The principle extends to the entire configuration surface: every structural relationship between constants can be verified before the binary exists.

---

## When to Reach for `const fn`

Use `const fn` when:

- You have numeric relationships between configuration constants that must hold
- You're building lookup tables that are fixed at compile time
- You want to validate protocol-level constants (field names, frame sizes, magic numbers)
- You're using const generics to encode capacity constraints in types

Don't use `const fn` when:

- The values are genuinely dynamic (read from env vars, config files, databases)
- The computation requires heap allocation or I/O
- You're adding compile-time checks that could just be unit tests — prefer tests when the values aren't truly compile-time constants

For taskforge, the worker configuration constants (`WORKER_CONCURRENCY`, `REDIS_POOL_SIZE`, retry parameters) are known at compile time and have relationships that must hold. Verifying them with `const fn` means these relationships can't drift apart during a config change — the developer sees the error immediately, not hours later in a staging environment.

---

## Exercises

**Exercise 1 — Compile-Time Pool Validation**

Implement the `assert_pool_sufficient(pool_size: usize, concurrency: usize)` `const fn`
that panics at compile time if `pool_size < concurrency + 2`. Add a compile-time
validation in your configuration:

```rust
const _: () = assert_pool_sufficient(REDIS_POOL_SIZE, WORKER_CONCURRENCY);
```

Then set `REDIS_POOL_SIZE = 3` and `WORKER_CONCURRENCY = 10` and verify the build
fails with a compile-time panic message (not a runtime error). Restore valid values.

**Exercise 2 — JobBuffer<N> with Const Generics**

Implement `struct JobBuffer<const N: usize>` with a `jobs: [Option<Job>; N]` array.
Add:
- `fn push(&mut self, job: Job) -> bool` — returns false if full
- `fn pop(&mut self) -> Option<Job>`
- `const CAPACITY: usize = N` — a const associated with the capacity

Write unit tests for `JobBuffer<4>` and `JobBuffer<8>`. Verify that creating a
`JobBuffer<0>` compiles (the type is valid, just immediately full).

**Exercise 3 — Compile-Time Lookup Table**

Create a `const fn priority_weight(priority: u8) -> u32` that maps:
- Priority 1 → weight 100
- Priority 2 → weight 500
- Priority 3 → weight 2500

Then build a `const PRIORITY_WEIGHTS: [u32; 4]` using the function. Write a unit
test that asserts `PRIORITY_WEIGHTS[3] == 2500`. Verify the weights array is in the
binary's rodata section (no runtime computation) using `cargo-show-asm` or inspecting
the binary with `objdump -d`.
