# Doc 05 — Testing: Integration Tests and Mocking

You built a layered architecture (Doc 04) specifically to make testing possible.
This document is about how you actually do it — unit tests for business logic,
integration tests with real Redis, and the tooling to measure how much of the
code your tests actually reach.

The core philosophy: **test behavior, not implementation.** Test what a function
does, not how it does it. This keeps tests from breaking every time you refactor.

---

## Unit Tests: Test the Service, Not the Handler 🟢

Unit tests live in the same file as the code, inside a `#[cfg(test)]` module.
They run in the same process, need no network, and run in milliseconds.

The service layer is where unit tests shine. You give the service an
`InMemoryJobRepository` and test the business logic directly:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::storage::InMemoryJobRepository;
    use std::sync::Arc;

    fn make_service() -> JobService {
        let repo = Arc::new(InMemoryJobRepository::new());
        JobService::new(repo)
    }

    #[tokio::test]
    async fn submit_job_returns_job_id() {
        let service = make_service();
        let payload = serde_json::json!({ "to": "user@example.com" });
        
        let result = service.submit_job("send_email", payload, "normal").await;
        
        assert!(result.is_ok());
        // The returned ID is a valid JobId
        let id = result.unwrap();
        // And we can look it up
        let status = service.get_status(id).await.unwrap();
        assert_eq!(status.status, JobStatus::Pending);
    }

    #[tokio::test]
    async fn cancel_nonexistent_job_returns_not_found() {
        let service = make_service();
        let fake_id = JobId::from(Uuid::new_v4());
        
        let result = service.cancel_job(fake_id).await;
        
        assert!(matches!(result, Err(ServiceError::NotFound(_))));
    }
    
    #[tokio::test]
    async fn cancelled_job_cannot_be_retrieved() {
        let service = make_service();
        let id = service
            .submit_job("test", serde_json::json!({}), "normal")
            .await
            .unwrap();
        
        service.cancel_job(id).await.unwrap();
        
        let result = service.get_status(id).await;
        // Should either be NotFound or show Cancelled status
        // depending on your design — test what makes sense
        assert!(result.is_err() || 
            result.unwrap().status == JobStatus::Cancelled);
    }
}
```

Notice: no HTTP. No Redis. No network. Just pure business logic. These tests
run in under a millisecond.

---

## The InMemoryJobRepository Implementation 🟡

This is the test double that makes the above tests work. It implements
`JobRepository` using in-memory data structures.

```rust
use std::collections::{HashMap, VecDeque};
use std::sync::Arc;
use tokio::sync::Mutex;

pub struct InMemoryJobRepository {
    jobs: Arc<Mutex<HashMap<JobId, JobRecord>>>,
    queues: Arc<Mutex<HashMap<String, VecDeque<SerializedJob>>>>,
}

impl InMemoryJobRepository {
    pub fn new() -> Self {
        InMemoryJobRepository {
            jobs: Arc::new(Mutex::new(HashMap::new())),
            queues: Arc::new(Mutex::new(HashMap::new())),
        }
    }
}

#[async_trait]
impl JobRepository for InMemoryJobRepository {
    async fn push(&self, job: &Job<Pending>) -> Result<(), StorageError> {
        let record = JobRecord {
            id: job.id,
            queue: job.queue.clone(),
            payload: job.payload.clone(),
            status: JobStatus::Pending,
            created_at: job.created_at,
            updated_at: chrono::Utc::now(),
            result: None,
            error: None,
            retry_count: job.retry_count,
        };
        
        let serialized = serde_json::to_string(job)
            .map_err(|e| StorageError::Serialization(e.to_string()))?;
        
        let mut jobs = self.jobs.lock().await;
        jobs.insert(job.id, record);
        
        let mut queues = self.queues.lock().await;
        queues
            .entry(job.queue.clone())
            .or_default()
            .push_back(serialized);
        
        Ok(())
    }
    
    async fn pop(
        &self,
        queue: &str,
        _timeout: Duration,
    ) -> Result<Option<Job<Pending>>, StorageError> {
        let mut queues = self.queues.lock().await;
        
        let serialized = queues
            .get_mut(queue)
            .and_then(|q| q.pop_front());
        
        match serialized {
            None => Ok(None),
            Some(s) => {
                let job = serde_json::from_str(&s)
                    .map_err(|e| StorageError::Deserialization(e.to_string()))?;
                Ok(Some(job))
            }
        }
    }
    
    // ... other methods follow the same pattern
}
```

The `Mutex` here is `tokio::sync::Mutex`, not `std::sync::Mutex`, because
the methods are `async`. Never use `std::sync::Mutex` in async code — holding
a standard mutex across an `.await` point can deadlock.

---

## Integration Tests: The Real Thing 🟡

Integration tests live in the `tests/` directory. They test your crate from
the outside, using only its public API. They run against real Redis.

```
taskforge-core/
├── src/
│   └── lib.rs
└── tests/
    └── integration.rs   <- here
```

```rust
// tests/integration.rs
use taskforge_core::{
    storage::RedisJobRepository,
    service::JobService,
    job::{Job, JobStatus},
};
use std::sync::Arc;

// Helper: connect to the test Redis instance
async fn test_redis() -> RedisJobRepository {
    let client = redis::Client::open("redis://127.0.0.1:6379/1")
        .expect("Could not connect to test Redis (DB 1)");
    let mgr = redis::aio::ConnectionManager::new(client)
        .await
        .expect("Could not create connection manager");
    RedisJobRepository::new(mgr)
}

// Helper: flush the test DB before each test
async fn clean_db(repo: &RedisJobRepository) {
    let mut conn = repo.connection().clone();
    redis::cmd("FLUSHDB")
        .query_async::<()>(&mut conn)
        .await
        .expect("Could not flush test DB");
}

#[tokio::test]
async fn push_and_pop_roundtrip() {
    let repo = test_redis().await;
    clean_db(&repo).await;
    let service = JobService::new(Arc::new(repo));
    
    let id = service
        .submit_job("test_job", serde_json::json!({"x": 1}), "normal")
        .await
        .unwrap();
    
    let status = service.get_status(id).await.unwrap();
    assert_eq!(status.status, JobStatus::Pending);
}
```

**DB number**: Use `redis://127.0.0.1:6379/1` (database 1) for tests, not
database 0 (the default). This prevents tests from stomping on development data.

**FLUSHDB before each test**: Each test starts clean. Without this, leftover data
from a previous failed test causes false positives or false negatives.

---

## testcontainers-rs: No Local Redis Required 🔴

`testcontainers-rs` spins up a real Redis in a Docker container for the duration
of your test suite. This eliminates the "you need Redis running locally to run tests"
problem in CI and on new developer machines.

```toml
[dev-dependencies]
testcontainers = "0.20"
testcontainers-modules = { version = "0.9", features = ["redis"] }
```

```rust
use testcontainers::runners::AsyncRunner;
use testcontainers_modules::redis::Redis;

#[tokio::test]
async fn with_containerized_redis() {
    // Spins up Redis in Docker, cleans up when test exits
    let redis_container = Redis::default().start().await.unwrap();
    let host = redis_container.get_host().await.unwrap();
    let port = redis_container.get_host_port_ipv4(6379).await.unwrap();
    
    let url = format!("redis://{}:{}/", host, port);
    let client = redis::Client::open(url.as_str()).unwrap();
    let mgr = redis::aio::ConnectionManager::new(client).await.unwrap();
    
    let repo = Arc::new(RedisJobRepository::new(mgr));
    let service = JobService::new(repo);
    
    let id = service
        .submit_job("test", serde_json::json!({}), "normal")
        .await
        .unwrap();
    
    assert!(service.get_status(id).await.is_ok());
    // Container stops and is removed when redis_container drops
}
```

This is cleaner for CI because the workflow does not need to install Redis as a
service — just Docker, which most CI runners already have.

---

## Property-Based Testing with proptest 🟡

proptest generates random inputs and verifies invariants. The key insight: you do
not write specific test cases. You write properties that should always hold.

```toml
[dev-dependencies]
proptest = "1"
```

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn job_id_roundtrip_through_string(
        // Generate random UUIDs
        raw_uuid in "[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}"
    ) {
        // Property: parsing a valid UUID string always succeeds
        let uuid = Uuid::parse_str(&raw_uuid).unwrap();
        let job_id = JobId::from(uuid);
        
        // Property: converting to string and back gives the same JobId
        let as_string = job_id.to_string();
        // ...parse back... (depends on your Display/FromStr impls)
        // assert roundtrip
    }
    
    #[test]
    fn job_payload_serialization_roundtrip(
        key in "[a-z]{1,20}",
        value in any::<u64>(),
    ) {
        // Property: any valid JSON payload round-trips through serialization
        let payload = serde_json::json!({ key: value });
        let job = Job::new("test_queue", payload.clone());
        
        let serialized = serde_json::to_string(&job).unwrap();
        let deserialized: Job<Pending> = serde_json::from_str(&serialized).unwrap();
        
        prop_assert_eq!(deserialized.payload, payload);
    }
}
```

When proptest finds a failing case, it shrinks the input to the smallest version
that still fails. This makes the failure easy to understand and debug.

---

## Snapshot Testing with insta 🟡

When your API returns structured JSON, you want to catch unintended changes.
`insta` captures the output on the first run and compares against it on subsequent runs.

```toml
[dev-dependencies]
insta = { version = "1", features = ["json"] }
```

```rust
#[tokio::test]
async fn job_status_response_format() {
    let service = make_service();
    let id = service
        .submit_job("send_email", 
            serde_json::json!({ "to": "test@example.com" }),
            "normal")
        .await
        .unwrap();
    
    let record = service.get_status(id).await.unwrap();
    
    // First run: creates tests/snapshots/job_status_response_format.snap
    // Subsequent runs: compares against that snapshot
    // Run `cargo insta review` to accept changes interactively
    insta::assert_json_snapshot!(record, {
        ".id" => "[job_id]",          // Redact fields that change between runs
        ".created_at" => "[timestamp]",
        ".updated_at" => "[timestamp]",
    });
}
```

If the API response format changes (you add a field, rename something), the test
fails. You run `cargo insta review` to see a diff and decide whether to accept the
change. This is much better than updating a dozen `assert_eq!(record.field, ...)` calls.

---

## Code Coverage: Seeing Your Blind Spots 🟡

You have been running `cargo test`. But which lines are actually executed?

```bash
cargo install cargo-llvm-cov
rustup component add llvm-tools-preview

# Run tests and generate HTML report
cargo llvm-cov --workspace --html

# Open the report
open target/llvm-cov/html/index.html

# In CI: fail if coverage drops below 80%
cargo llvm-cov --workspace --fail-under-lines 80
```

The report shows every line in green (covered) or red (not covered). Red lines
in error handling are common. Red lines in the happy path are a problem.

Do not chase 100% coverage. That requires increasingly contrived tests and
exclusion macros. Aim for:
- 80%+ line coverage
- 70%+ branch coverage (catches untested `if/else` arms)

Pay special attention to error handling branches. In job queue code, the most
dangerous bugs are in the retry logic, the dead letter queue handling, and the
error recovery paths — exactly the paths that are hardest to test manually.

---

## What to Test vs What Not to Test 🟢

**Test these:**
- Business logic in the service layer (retry count limits, queue routing, cancellation rules)
- Serialization roundtrips (job can be serialized, put in Redis, deserialized, and still works)
- Error cases (not found, already cancelled, malformed payload)
- State transitions (pending → running → completed/failed, failed → pending/dead)

**Do not bother testing these:**
- That Redis stores and retrieves data (Redis tests this itself)
- Framework glue (that Axum routes requests correctly)
- Trivial getters/setters with no logic
- Generated code (serde derives, etc.)

The rule: test behavior, not plumbing. If a test is just checking that your code
calls a dependency, you are testing the dependency, not your code.

---

## Test Helpers and Fixtures 🟢

Keep test setup DRY. A `TestFixture` struct owned by each test avoids repeated
setup code:

```rust
#[cfg(test)]
pub mod test_support {
    use super::*;
    
    pub struct TestFixture {
        pub service: JobService,
    }
    
    impl TestFixture {
        pub fn new() -> Self {
            let repo = Arc::new(InMemoryJobRepository::new());
            TestFixture {
                service: JobService::new(repo),
            }
        }
        
        // Pre-built jobs for common scenarios
        pub async fn a_pending_email_job(&self) -> JobId {
            self.service
                .submit_job(
                    "send_email",
                    serde_json::json!({ "to": "test@example.com" }),
                    "normal",
                )
                .await
                .unwrap()
        }
    }
}

#[tokio::test]
async fn test_cancel_pending_job() {
    let fixture = TestFixture::new();
    let id = fixture.a_pending_email_job().await;
    
    let result = fixture.service.cancel_job(id).await;
    assert!(result.is_ok());
}
```

---

## How It Breaks

**Integration tests sharing state.** Two integration tests run in parallel, both using the same Redis database. One test's jobs interfere with the other. Use a unique key prefix or a separate Redis database per test.

**Tests that only test the happy path.** Your job worker is tested when jobs succeed, but not when they fail, timeout, panic, or are duplicated. Add failure case tests before claiming "it's tested."

**Tests that mock too much.** If you mock the database, the job queue, the serializer, and the retry logic, you're testing that your mock returns what you set it to return — not that your code works. Mock at the outermost boundary only.

**Property tests finding edge cases.** When you write a property test like "any valid job can be serialized and deserialized back to the same job," you'll sometimes find that empty strings or special characters break your JSON parsing.

---

## Common Mistakes

**1. Testing the Redis repository instead of the service.** If your tests connect
to Redis to test that `RedisJobRepository::push()` works, you are testing Redis, not
your code. Trust the database. Test your business logic with in-memory stubs.

**2. Using `std::sync::Mutex` in async test code.** If you lock a mutex and then
`.await`, you hold the lock across an await point. This can deadlock. Use
`tokio::sync::Mutex` in async contexts.

**3. Not flushing Redis between tests.** Tests run in unpredictable order
(especially with `cargo test --test-threads 4`). If test A pushes a job and
test B expects an empty queue, the test order determines the result. FLUSHDB
at the start of each integration test.

**4. Writing only happy-path tests.** The job queue's most critical code is
what happens when things go wrong: job execution fails, Redis connection drops,
payload is malformed. These paths are rarely tested and often wrong.

**5. Snapshot testing everything.** Snapshot tests are powerful for large outputs
but painful for fields that legitimately change (timestamps, IDs). Always redact
volatile fields with insta's `{".field" => "[redacted]"}` syntax.
