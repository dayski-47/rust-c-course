# Doc 11 - Testing Type-Level Guarantees

🟡 Testing that valid code works is table stakes. Testing that invalid code doesn't compile is what separates a type-safe system from one that only claims to be.

When you build type-state machines, capability tokens, validated newtypes, and sealed traits, you've made specific promises: "you cannot use a `JobHandle<Pending>` where a `JobHandle<Running>` is expected," "you cannot call `purge_failed` without an `AdminToken`," "you cannot construct a `JobId` that isn't a valid UUID."

Those promises are enforced by the compiler - but only if you *don't accidentally break the type system* in a later refactor. The tools in this doc let you write tests that fail to compile when your type invariants are correct, and *pass* when someone weakens them. This is the inverse of a normal test, and it's the only way to verify compile-time properties.

---

## Compile-Fail Tests with `trybuild`

The [`trybuild`](https://crates.io/crates/trybuild) crate compiles small Rust programs and asserts that they produce specific error messages. A compile-fail test defines code that *should* be rejected by the compiler, plus the expected error, and the test passes only if the code fails to compile with exactly that error.

### Setup

```toml
# taskforge-core/Cargo.toml
[dev-dependencies]
trybuild = "1"
```

### The test runner

```rust
// tests/compile_fail.rs
#[test]
fn type_safety_tests() {
    let t = trybuild::TestCases::new();
    // All .rs files in tests/ui/ must fail to compile
    t.compile_fail("tests/ui/*.rs");
}
```

### Writing compile-fail cases

Each test case is a small Rust file. The test file lives in `tests/ui/`, the expected error in a matching `.stderr` file:

**`tests/ui/job_handle_reuse.rs`** - verify that a consumed `JobHandle<Pending>` can't be used twice:

```rust
use taskforge_core::{JobHandle, job_id};

fn main() {
    let handle = JobHandle::new_pending(job_id!());
    let _running = handle.start();
    let _running2 = handle.start();  // should fail: use of moved value
}
```

**`tests/ui/job_handle_reuse.stderr`** - the expected compiler error:

```text
error[E0382]: use of moved value: `handle`
 --> tests/ui/job_handle_reuse.rs:6:19
  |
4 |     let handle = JobHandle::new_pending(job_id!());
  |         ------ move occurs because `handle` has type `JobHandle<Pending>`, which does not implement the `Copy` trait
5 |     let _running = handle.start();
  |                    ------ value moved here
6 |     let _running2 = handle.start();  // should fail: use of moved value
  |                     ^^^^^^ value used here after move
```

**`tests/ui/missing_admin_token.rs`** - verify that `purge_failed` requires `AdminToken`:

```rust
use taskforge_core::JobQueue;

async fn try_purge(queue: &JobQueue) {
    queue.purge_failed().await;  // should fail: missing required argument
}

fn main() {}
```

**`tests/ui/missing_admin_token.stderr`:**

```text
error[E0061]: this function takes 2 arguments but 1 argument was supplied
 --> tests/ui/missing_admin_token.rs:4:11
  |
4 |     queue.purge_failed().await;
  |           ^^^^^^^^^^^^ -- an argument of type `&WorkerAdminToken` is missing
```

**`tests/ui/wrong_job_state.rs`** - verify that `complete()` isn't callable on a `Pending` handle:

```rust
use taskforge_core::{JobHandle, job_id};

fn main() {
    let handle = JobHandle::new_pending(job_id!());
    handle.complete();  // should fail: method not found on JobHandle<Pending>
}
```

### Running compile-fail tests in CI

```yaml
# .github/workflows/ci.yml
- name: Run compile-fail tests
  run: cargo test --test compile_fail
```

Compile-fail tests run as part of the normal test suite. If someone adds `#[derive(Clone)]` to `JobHandle`, the `job_handle_reuse` test starts *passing to compile* - which fails the compile-fail test runner. The regression is caught automatically.

---

## What Compile-Fail Tests Actually Test

The scenarios worth writing compile-fail tests for in taskforge:

| Invariant | Test case file |
|-----------|---------------|
| `JobHandle<Pending>` can't be `start()`ed twice | `job_handle_reuse.rs` |
| `JobHandle<Pending>` can't be `complete()`d | `wrong_job_state_complete.rs` |
| `purge_failed()` requires `AdminToken` | `missing_admin_token.rs` |
| `cancel_job()` requires either `AdminToken` or `OwnerToken` | `missing_auth_cancel.rs` |
| `WorkerAdminToken` can't be constructed externally | `forge_admin_token.rs` |
| Adding `Celsius` and `Rpm` fails (if using dimensional types) | `unit_mismatch.rs` |

Each test is 5–15 lines. Together they form a regression suite for the type system.

---

## Property-Based Testing of Validated Boundaries

Validated newtypes (Section 5's parse-don't-validate pattern) make a specific promise: "all input that passes validation is safe to use." That promise is hard to verify with hand-written examples - you can only write examples you think of. Property-based testing generates *thousands* of random inputs and verifies that the invariants hold for all of them.

### Setup

```toml
[dev-dependencies]
proptest = "1"
```

### Testing `JobId` validation

```rust
use proptest::prelude::*;
use taskforge_core::JobId;

proptest! {
    /// Every string that parses to JobId must be a valid UUID.
    #[test]
    fn job_id_only_accepts_valid_uuids(s in ".*") {
        let result = s.parse::<JobId>();
        match result {
            Ok(id) => {
                // If it parsed, it must round-trip correctly
                assert_eq!(id.to_string().parse::<JobId>().unwrap(), id);
            }
            Err(_) => {
                // If it failed, that's fine - the input was invalid
            }
        }
    }

    /// JobId must reject strings that aren't UUID v4 format
    #[test]
    fn job_id_rejects_non_uuid(s in "[a-z]{1,20}") {
        // A random lowercase string is almost certainly not a UUID
        // (with overwhelming probability - this tests the constraint)
        let is_valid_uuid = s.parse::<uuid::Uuid>().is_ok();
        let parsed = s.parse::<JobId>();
        assert_eq!(parsed.is_ok(), is_valid_uuid,
            "JobId parse result should match UUID validity for input {:?}", s);
    }
}
```

### Testing `JobSpec` validation

```rust
proptest! {
    /// Job type must be non-empty after trimming.
    #[test]
    fn job_spec_rejects_empty_type(
        spaces in " *",  // zero or more spaces
        extra in ".*",
    ) {
        let job_type = format!("{spaces}{extra}{spaces}");
        if job_type.trim().is_empty() {
            // Empty or whitespace-only job type must be rejected
            let result = JobSpecBuilder::new()
                .job_type(&job_type)
                .build();
            assert!(result.is_err(), "empty job type should be rejected");
        }
    }

    /// Priority roundtrips through serialization.
    #[test]
    fn priority_roundtrips(p in 0u8..=2u8) {
        // 0 = Low, 1 = Normal, 2 = High
        let priority = Priority::from_u8(p).unwrap();
        let serialized = serde_json::to_string(&priority).unwrap();
        let deserialized: Priority = serde_json::from_str(&serialized).unwrap();
        assert_eq!(priority, deserialized);
    }
}
```

Proptest runs each `proptest!` block with hundreds of generated inputs by default. A failing case is automatically shrunk to the minimal reproducer.

---

## Snapshot Testing with `insta`

Some outputs are complex enough that you'd rather capture them and verify they haven't changed than write out expected values by hand. `insta` makes snapshot testing ergonomic:

```toml
[dev-dependencies]
insta = { version = "1", features = ["json"] }
```

```rust
use insta::assert_json_snapshot;
use taskforge_core::{Job, JobStatus, Priority};

#[test]
fn job_serialization_snapshot() {
    let job = Job {
        id: JobId("550e8400-e29b-41d4-a716-446655440000".parse().unwrap()),
        job_type: "email_notification".to_string(),
        status: JobStatus::Pending,
        priority: Priority::Normal,
        retry_count: 0,
        created_at: 1_700_000_000,
    };

    assert_json_snapshot!(job, @r###"
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "job_type": "email_notification",
      "status": "Pending",
      "priority": "Normal",
      "retry_count": 0,
      "created_at": 1700000000
    }
    "###);
}
```

On the first run, `insta` captures the snapshot into a `.snap` file. On subsequent runs, it compares the output to the snapshot. If the serialization format changes (a field renamed, a type changed), the test fails and shows a diff. Run `cargo insta review` to interactively approve or reject changes.

---

## Verifying Zero-Cost Abstractions

The capability token and PhantomData patterns promise zero runtime cost. You can verify this with `cargo-show-asm`:

```bash
cargo install cargo-show-asm

# Show assembly for a function that uses a capability token
cargo asm --lib taskforge_core::JobQueue::cancel_job
```

A function that accepts `_admin: &WorkerAdminToken` where `WorkerAdminToken` is a zero-sized type should generate identical assembly to the same function without the parameter. If you see extra register moves or stack operations for the token, the type isn't truly zero-cost - that's worth investigating.

For the unit-of-measure `Duration<Unit>` type, the arithmetic should compile to the same instructions as operating on a raw `f64`. Compare:

```bash
cargo asm taskforge_core::duration_add
```

against what you'd expect from:

```rust
fn add_f64(a: f64, b: f64) -> f64 { a + b }
```

If they differ, the abstraction has a cost.

---

## RAII Invariant Tests

The `Transaction` type from section 6 promised that dropping a transaction without committing rolls it back. Test the RAII invariant:

```rust
#[tokio::test]
async fn transaction_rollback_on_drop() {
    let mut db = test_db().await;
    let initial_count: i64 = sqlx::query_scalar!("SELECT COUNT(*) FROM jobs")
        .fetch_one(&db)
        .await
        .unwrap();

    {
        let mut tx = db.begin_transaction().await.unwrap();
        sqlx::query!("INSERT INTO jobs (status) VALUES ('pending')")
            .execute(&mut tx)
            .await
            .unwrap();
        // tx dropped here without commit - Drop must roll back
    }

    let final_count: i64 = sqlx::query_scalar!("SELECT COUNT(*) FROM jobs")
        .fetch_one(&db)
        .await
        .unwrap();

    assert_eq!(initial_count, final_count, "transaction must roll back on drop");
}

#[tokio::test]
async fn transaction_commit_persists() {
    let mut db = test_db().await;

    let mut tx = db.begin_transaction().await.unwrap();
    sqlx::query!("INSERT INTO jobs (status) VALUES ('pending')")
        .execute(&mut tx)
        .await
        .unwrap();
    tx.commit().await.unwrap();

    let count: i64 = sqlx::query_scalar!("SELECT COUNT(*) FROM jobs")
        .fetch_one(&db)
        .await
        .unwrap();
    assert_eq!(count, 1);
}
```

These tests verify that the `Drop` implementation is correct - the RAII contract holds at runtime, not just in theory.

---

## The Testing Pyramid for Type-Level Code

```
        ┌────────────────────────┐
        │   Compile-fail tests   │  ← trybuild: "this code must not compile"
        │     (trybuild)         │
        ├────────────────────────┤
        │   Property-based tests │  ← proptest: "these invariants hold for all inputs"
        │     (proptest)         │
        ├────────────────────────┤
        │   Snapshot tests       │  ← insta: "this output hasn't changed"
        │     (insta)            │
        ├────────────────────────┤
        │   Unit tests           │  ← standard: "this behavior is correct"
        │     (std)              │
        └────────────────────────┘
```

Each layer catches different classes of problems:
- **Compile-fail**: the type invariants themselves haven't been accidentally relaxed
- **Property-based**: validation logic handles the full input space, not just your test cases
- **Snapshot**: serialization formats and error messages haven't silently changed
- **Unit**: specific behaviors work correctly

All four layers are in the taskforge test suite. The compile-fail tests are the most unusual, but they're critical - without them, adding `#[derive(Clone)]` to `JobHandle` breaks the type-state guarantee silently.
