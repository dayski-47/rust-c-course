# Doc 14 - Rust Idioms and Best Practices

🟢 Section 7 is where Rust's type system stops being a novelty and starts being your most reliable colleague.

taskforge is the most type-sophisticated project in the course: type-state machines for job lifecycles, capability tokens for authorization, const-verified configuration, typed Redis commands, compile-fail tests. This doc names the idioms underneath all of that - the habits that make them work, and the traps that make them fail.

---

## Make Illegal States Unrepresentable

The highest-leverage design decision in any Rust codebase: if an invalid state cannot be constructed, it cannot occur at runtime.

The newtype and type-state patterns both serve this goal. So does choosing `enum` over boolean flags:

```rust
// ❌ Three booleans - 8 possible states, most are illegal
pub struct Job {
    pending: bool,
    running: bool,
    complete: bool,
}
// What does { pending: true, running: true, complete: true } mean?

// ✅ Enum - exactly 4 valid states, all distinct
pub enum JobStatus {
    Pending,
    Running,
    Completed,
    Failed,
}
```

And choosing newtypes over raw types:

```rust
// ❌ Job ID and worker ID are both Uuid - easy to swap
fn assign_job(job: Uuid, worker: Uuid) {}

// ✅ Distinct types - swap is a compile error
fn assign_job(job: JobId, worker: WorkerId) {}
```

Before adding a field or a function, ask: "what invalid state does this allow?" If the answer is "many," reach for enums, newtypes, or type-state.

---

## Propagate Errors with `?`, Define Them with `thiserror`

The `?` operator is Rust's most important ergonomic feature for error handling. It requires that errors implement `From<SourceError> for TargetError`. The `thiserror` crate generates these implementations:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum TaskforgeError {
    #[error("job not found: {0}")]
    NotFound(JobId),

    #[error("invalid job ID: {0}")]
    InvalidJobId(String),

    #[error("queue error: {0}")]
    Queue(#[from] redis::RedisError),  // From<RedisError> generated automatically

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),     // From<sqlx::Error> generated automatically

    #[error("serialization error: {0}")]
    Serialization(#[from] serde_json::Error),

    #[error("authentication failed")]
    Unauthorized,
}

// Usage - ? converts each error to TaskforgeError via the generated From impls
async fn process_job(id: JobId, redis: &TypedRedis, db: &sqlx::Pool<sqlx::Sqlite>)
    -> Result<(), TaskforgeError>
{
    let record = redis.execute(&HGetAll { job_id: id.to_string() })?;  // RedisError → TaskforgeError
    let job = record.ok_or(TaskforgeError::NotFound(id))?;
    
    sqlx::query!("UPDATE jobs SET status = 'running' WHERE id = ?", id.to_string())
        .execute(db)
        .await?;  // sqlx::Error → TaskforgeError
    
    Ok(())
}
```

For library crates that are published externally, prefer explicit `impl From<>` over `#[from]` - it makes the API surface clearer and avoids accidentally exposing third-party error types. For applications like taskforge, `thiserror` with `#[from]` is the right default.

One rule: **never use `unwrap()` in production code paths.** In tests and examples, `unwrap()` is fine (it gives a clear panic on failure). In code that runs in production, every `unwrap()` is a latent bug waiting for a rare input.

---

## Prefer `iter()` Chains for Data Pipelines

Iterator chains communicate *what* is being done without noise from mutable state:

```rust
// ❌ Mutable loop - intent buried in housekeeping
fn highest_priority_jobs(jobs: &[Job], limit: usize) -> Vec<&Job> {
    let mut sorted: Vec<&Job> = Vec::new();
    for job in jobs {
        if job.status == JobStatus::Pending {
            sorted.push(job);
        }
    }
    sorted.sort_by(|a, b| b.priority.cmp(&a.priority));
    sorted.truncate(limit);
    sorted
}

// ✅ Iterator chain - reads like the description
fn highest_priority_jobs(jobs: &[Job], limit: usize) -> Vec<&Job> {
    let mut results: Vec<&Job> = jobs.iter()
        .filter(|j| j.status == JobStatus::Pending)
        .collect();
    results.sort_by(|a, b| b.priority.cmp(&a.priority));
    results.truncate(limit);
    results
}
```

For collecting into a `HashMap`:

```rust
// Frequency count of job types in the queue
let type_counts: HashMap<&str, usize> = jobs.iter()
    .filter(|j| j.status == JobStatus::Pending)
    .map(|j| j.job_type.as_str())
    .fold(HashMap::new(), |mut acc, job_type| {
        *acc.entry(job_type).or_insert(0) += 1;
        acc
    });
```

Use a loop when the operation has side effects that need to be sequenced (updating a database, writing to a channel) or when you need early return semantics inside the iteration body. For pure data transformations, chains are clearer.

---

## Bounds in Impl Blocks, Not Struct Definitions

A common mistake: adding bounds to struct definitions.

```rust
// ❌ Bound on the struct - propagates everywhere
struct JobExecutor<F: Fn() -> Job> {
    factory: F,
}
// Now every impl block and every type annotation for JobExecutor must repeat F: Fn() -> Job

// ✅ Bound on the impl - each impl only carries what it needs
struct JobExecutor<F> {
    factory: F,
}

impl<F: Fn() -> Job> JobExecutor<F> {
    fn execute(&self) -> Job {
        (self.factory)()
    }
}

impl<F> JobExecutor<F> {
    // Methods that don't need the Fn bound can be in their own impl block
    fn name(&self) -> &str { "executor" }
}
```

This reduces unnecessary constraint propagation and keeps individual impl blocks focused.

---

## `impl Trait` for Flexible Interfaces

`impl Trait` in parameter position is syntactic sugar for a bounded generic. In return position, it hides the concrete type:

```rust
// Parameter: accept any iterator of jobs
fn process_batch(jobs: impl Iterator<Item = Job>) -> Vec<JobId> {
    jobs.map(|j| j.id).collect()
}

// Return: caller gets "something implementing Future", not the concrete type
fn fetch_job(&self, id: &JobId) -> impl Future<Output = Result<Job, TaskforgeError>> + '_ {
    async move {
        self.redis.execute(&HGetAll { job_id: id.to_string() })?
            .ok_or(TaskforgeError::NotFound(id.clone()))
    }
}

// When storing futures in collections, use the boxed form:
type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;

fn spawn_all_workers(
    jobs: Vec<Job>,
    executor: &Executor,
) -> Vec<BoxFuture<'_, Result<(), TaskforgeError>>> {
    jobs.into_iter()
        .map(|job| -> BoxFuture<'_, _> {
            Box::pin(executor.run_job(job))
        })
        .collect()
}
```

---

## Derive What You Can, Implement Only What's Custom

```rust
// Derive standard traits - less code, harder to get wrong
#[derive(Debug, Clone, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
pub struct JobId(pub uuid::Uuid);

// Implement manually only when derived behavior is wrong:
impl std::fmt::Display for JobId {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl std::str::FromStr for JobId {
    type Err = TaskforgeError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        s.parse::<uuid::Uuid>()
            .map(JobId)
            .map_err(|_| TaskforgeError::InvalidJobId(s.to_string()))
    }
}
```

Always derive: `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `Serialize`, `Deserialize` (when applicable), `Default` (when zero/empty is a valid default).

Only implement manually when the derived behavior is semantically wrong - for example, `PartialOrd` on a type where comparison should use a specific field ordering.

---

## Use `Default` for Configuration Structs

Configuration with sensible defaults shouldn't require every field to be specified:

```rust
#[derive(Debug, Clone)]
pub struct WorkerConfig {
    pub concurrency: u32,
    pub redis_url: String,
    pub poll_interval_ms: u64,
    pub max_retries: u32,
    pub job_timeout_secs: u64,
}

impl Default for WorkerConfig {
    fn default() -> Self {
        WorkerConfig {
            concurrency: 4,
            redis_url: "redis://127.0.0.1:6379".to_string(),
            poll_interval_ms: 100,
            max_retries: 3,
            job_timeout_secs: 300,
        }
    }
}

// Struct update syntax to override just what you need:
let config = WorkerConfig {
    concurrency: 16,
    redis_url: std::env::var("REDIS_URL").unwrap_or_default(),
    ..WorkerConfig::default()
};
```

Combine `Default` with the builder pattern for more complex configuration: `WorkerConfig::default()` gives a safe starting point, the builder allows ergonomic modification.

---

## Don't Add `Clone` When You Mean `Arc`

A common over-cloning pattern in Rust:

```rust
// ❌ Cloning the entire state struct on every worker spawn
for _ in 0..config.concurrency {
    let state = app_state.clone();  // clones all the data
    tokio::spawn(run_worker(state));
}

// ✅ Wrap expensive state in Arc; clone the Arc (cheap)
let shared_state = Arc::new(app_state);
for _ in 0..config.concurrency {
    let state = Arc::clone(&shared_state);  // atomic reference count increment
    tokio::spawn(run_worker(state));
}
```

If a type needs to be shared across async tasks, put it behind `Arc`. Cloning the `Arc` is a few nanoseconds (an atomic increment). Cloning the underlying struct might be copying megabytes of configuration or a connection pool.

---

## The Type-Level Design Checklist

When adding a new domain type or function to taskforge, ask:

- **Can it be a newtype instead of a raw type?** `JobId` not `Uuid`.
- **Can invalid states be made unrepresentable?** Use enums instead of boolean combinations.
- **Does this function require authority?** Add a capability token parameter.
- **Does this value need to be used exactly once?** Drop `Clone`, add `#[must_use]`.
- **Can this configuration relationship be verified at compile time?** Use `const fn` + `assert!`.
- **Does this struct have a phantom type parameter?** Use `PhantomData<T>` - not just a comment.

These questions take 30 seconds to answer and prevent entire classes of bugs.

---

## Industry Practices Checklist

Before marking the taskforge project complete:

- [ ] `JobHandle<S>` type-state enforces the Pending → Running → Completed/Failed lifecycle
- [ ] `WorkerAdminToken` capability gates all destructive operations
- [ ] `WorkerProcessToken` gates job status transitions - workers can't forge each other's completions
- [ ] `const fn` assertions verify that `REDIS_POOL_SIZE >= WORKER_CONCURRENCY + 4` at build time
- [ ] All Redis commands use typed command structs - no raw `Vec<u8>` parsing at call sites
- [ ] Compile-fail tests verify that the type invariants can't be bypassed
- [ ] `cargo test --workspace` includes proptest runs for all validated boundaries
- [ ] `cargo bench` baseline established for deserialization and routing hot paths
- [ ] `#[must_use]` on `JobReservation` - dropped reservation is almost always a bug
- [ ] All public types derive `Debug` - no mystery values in panic messages
- [ ] `thiserror` used for all error types - `?` works throughout the codebase
- [ ] No `unwrap()` in non-test production paths
- [ ] CI gates on `cargo clippy -- -D warnings` and `cargo fmt --check`

The pattern running through all of these: **express your intentions in the type system**. What can be a compile error should be a compile error. What can be a type instead of a comment should be a type. What can be checked at build time should be checked at build time.

Runtime errors that make it to production are expensive. Compile errors that fire in the developer's editor are free.
