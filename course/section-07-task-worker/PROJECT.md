# Section 7 Project: taskforge

A distributed task worker system. You are building the kind of infrastructure
that powers every high-traffic web application - the background job queue.

---

## What You Are Building

Every real web application has slow operations: send a password reset email,
resize an uploaded image, generate a PDF report, sync data to a third-party API.
You do not run these in the HTTP handler. The handler would time out, the user
would wait, and the system would be fragile.

Instead, the handler pushes a **job** to a **queue** and immediately returns
"accepted." A background **worker** picks up the job and processes it
independently. This is exactly what Sidekiq does for Ruby, Celery for Python,
and BullMQ for Node. You are building the Rust version.

**taskforge** consists of three crates in a workspace:

- `taskforge-core`: the library. The `Job` trait, type-state lifecycle,
  the `JobRepository` trait, storage implementations, and the service layer.
  Any Rust application can depend on this.

- `taskforge-worker`: the binary that runs workers. Pulls jobs from Redis,
  executes them, updates status.

- `taskforge-api`: the HTTP API server. Accepts job submissions, returns status,
  exposes management endpoints.

---

## Architecture

```
[Producer Apps] --POST /jobs--> [taskforge-api :8080]
                                        |
                                        v
                                 [Redis Lists]
                                 queue:high    <- priority 3
                                 queue:normal  <- priority 2
                                 queue:low     <- priority 1
                                        |
                               .--------+--------.
                               v                 v
                          [Worker 1]        [Worker 2]
                         BRPOP queue        BRPOP queue
                         deserialize        deserialize
                         execute job        execute job
                         update status      update status
                               |                 |
                               '--------+--------'
                                        v
                               [job:{id} Redis Hash]
                               status, type, payload,
                               result, error, retry_count,
                               created_at, started_at,
                               completed_at, worker_id
```

Workers use Redis's `BRPOP` command, which **blocks** for up to N seconds
waiting for a job to appear on the queue. This is more efficient than polling.
When a job arrives, `BRPOP` returns it immediately. When there are no jobs,
workers sleep efficiently instead of burning CPU.

Multiple workers can run simultaneously. Redis's `BRPOP` guarantees that each
job is delivered to exactly one worker.

---

## The Job Trait

This is where generics shine. Any type that implements `Job` can be processed
by the worker system.

```rust
use async_trait::async_trait;
use serde::{Serialize, de::DeserializeOwned};
use std::error::Error;

#[async_trait]
pub trait Job: Serialize + DeserializeOwned + Send + Sync + 'static {
    /// Which Redis queue this job goes to
    const QUEUE: &'static str;
    
    /// The type this job produces on success
    type Output: Serialize + Send;
    
    /// The type of error this job can fail with
    type Error: Error + Send;
    
    /// Execute the job. Consumes self.
    async fn execute(self) -> Result<Self::Output, Self::Error>;
}
```

Example implementations:

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct SendEmailJob {
    pub to: String,
    pub subject: String,
    pub body: String,
}

#[async_trait]
impl Job for SendEmailJob {
    const QUEUE: &'static str = "normal";
    type Output = ();
    type Error = EmailError;
    
    async fn execute(self) -> Result<(), EmailError> {
        // In milestone 1, just print instead of actually sending
        println!("Sending email to: {}", self.to);
        Ok(())
    }
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ResizeImageJob {
    pub path: std::path::PathBuf,
    pub width: u32,
    pub height: u32,
}

#[async_trait]
impl Job for ResizeImageJob {
    const QUEUE: &'static str = "high";
    type Output = std::path::PathBuf;  // Path to resized image
    type Error = ImageError;
    
    async fn execute(self) -> Result<std::path::PathBuf, ImageError> {
        // Resize the image at self.path to self.width x self.height
        // Return path to the output file
        todo!()
    }
}

#[derive(Debug, Serialize, Deserialize)]
pub struct GenerateReportJob {
    pub user_id: String,
    pub date_range: (chrono::DateTime<chrono::Utc>, chrono::DateTime<chrono::Utc>),
}

#[async_trait]
impl Job for GenerateReportJob {
    const QUEUE: &'static str = "low";
    type Output = String;  // URL to the generated report
    type Error = ReportError;
    
    async fn execute(self) -> Result<String, ReportError> {
        todo!()
    }
}
```

The worker does not know at compile time which concrete job types exist.
See the Generic Worker section below for how this is handled.

---

## The Type-State Job Lifecycle

Jobs move through states during their lifetime. Using type-state, invalid
transitions are caught at compile time.

```
Job<Pending>
    |
    | .assign(worker_id)          -- worker picks up the job
    v
Job<Running { started_at, worker_id }>
    |                    |
    | .complete(result)  | .fail(error)
    v                    v
Job<Completed>      Job<Failed { error, attempt }>
                         |                    |
                         | .retry()           | .discard()
                         v                    v
                    Job<Pending>         Job<Dead>
                    (back to queue)      (dead letter)
```

State structs:

```rust
pub struct Pending;

pub struct Running {
    pub started_at: std::time::Instant,
    pub worker_id: WorkerId,
}

pub struct Completed {
    pub finished_at: std::time::Instant,
    pub result: serde_json::Value,
}

pub struct Failed {
    pub error: String,
    pub attempt: u32,
    pub failed_at: std::time::Instant,
}

pub struct Dead {
    pub reason: String,
}
```

The struct:

```rust
pub struct Job<S> {
    pub id: JobId,
    pub job_type: String,
    pub queue: String,
    pub payload: serde_json::Value,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub retry_count: u32,
    pub state: S,
}
```

Each transition exists only on the appropriate state:

```rust
impl Job<Pending> {
    pub fn assign(self, worker_id: WorkerId) -> Job<Running> { ... }
}

impl Job<Running> {
    pub fn complete(self, result: serde_json::Value) -> Job<Completed> { ... }
    pub fn fail(self, error: String) -> Job<Failed> { ... }
    pub fn worker_id(&self) -> WorkerId { self.state.worker_id }
}

impl Job<Failed> {
    pub fn retry(self) -> Job<Pending> { ... }
    pub fn discard(self, reason: String) -> Job<Dead> { ... }
    pub fn attempt_number(&self) -> u32 { self.state.attempt }
}
```

Calling `.complete()` on a `Job<Pending>` is a compile error - that method
does not exist on `Job<Pending>`.

---

## API Endpoints

```
POST   /jobs              Submit a new job
                          Body: { "type": "send_email", "payload": {...}, "queue": "normal" }
                          Response 202: { "id": "uuid" }

GET    /jobs/:id          Get job status and result
                          Response 200: { "id": "...", "status": "completed", "result": {...} }
                          Response 404: job not found

GET    /jobs              List jobs (with filters)
                          Query: ?status=failed&queue=high&limit=50&offset=0

DELETE /jobs/:id          Cancel a pending job
                          Response 204: cancelled
                          Response 409: job is already running or completed

GET    /queues            List all queues with current depth
                          Response: [{ "name": "high", "depth": 42 }, ...]

GET    /workers           List active workers with current job
                          Response: [{ "id": "...", "current_job": "...", "started_at": "..." }]

GET    /health            Health check (always 200 if process is running)

GET    /ready             Readiness check (200 if Redis is connected)

GET    /metrics           Prometheus metrics
```

---

## Redis Data Layout

```
queue:high          Redis List (LPUSH to add, BRPOP to consume)
queue:normal        Redis List
queue:low           Redis List

job:{id}            Redis Hash
                    status: "pending" | "running" | "completed" | "failed" | "dead"
                    type: "send_email"
                    queue: "normal"
                    payload: "{...}"      (JSON string)
                    result: "{...}"       (JSON string, set on completion)
                    error: "..."          (set on failure)
                    retry_count: "0"
                    created_at: "2024-01-15T10:30:00Z"
                    started_at: "..."     (set when worker picks up)
                    completed_at: "..."   (set when done)
                    worker_id: "..."      (set when running)

workers:active      Redis Set of worker IDs (UUIDs)

worker:{id}:heartbeat   Redis String with TTL
                        Value: "alive"
                        TTL: 30 seconds (refreshed every 10 seconds by the worker)
                        If TTL expires, the worker is considered dead

queue:dead          Redis List for dead-letter jobs
```

---

## Retry and Failure Logic

When a job fails, the worker does not simply re-queue it immediately.
Exponential backoff prevents hammering a broken external service.

```
Attempt 1 fails -> wait 30 seconds -> retry (attempt 2)
Attempt 2 fails -> wait 5 minutes -> retry (attempt 3)
Attempt 3 fails -> wait 30 minutes -> retry (attempt 4)
Attempt 4 fails -> move to queue:dead (dead letter queue)
```

Dead letter jobs stay in `queue:dead` until a human (or automated recovery
process) decides what to do: discard, fix, or manually re-queue.

The retry delay is stored in Redis as a sorted set (scored by the Unix timestamp
when the job should be retried), and a separate scheduler process moves jobs
from the retry set to the appropriate queue when their time comes.

---

## Cargo.toml

```toml
# Root workspace
[workspace]
members = ["taskforge-core", "taskforge-worker", "taskforge-api"]
resolver = "2"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
async-trait = "0.1"
thiserror = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }


# taskforge-core/Cargo.toml
[package]
name = "taskforge-core"
version = "0.1.0"
edition = "2021"

[features]
default = ["redis"]
redis = ["dep:redis"]

[dependencies]
serde = { workspace = true }
serde_json = { workspace = true }
tokio = { workspace = true }
async-trait = { workspace = true }
thiserror = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
tracing = { workspace = true }
redis = { version = "0.25", features = ["tokio-comp", "connection-manager"], optional = true }

[dev-dependencies]
tokio = { version = "1", features = ["full", "test-util"] }
criterion = { version = "0.5", features = ["html_reports", "async_tokio"] }

[[bench]]
name = "job_queue"
harness = false


# taskforge-api/Cargo.toml
[package]
name = "taskforge-api"
version = "0.1.0"
edition = "2021"

[dependencies]
taskforge-core = { path = "../taskforge-core" }
axum = "0.7"
tower-http = { version = "0.5", features = ["cors", "trace"] }
serde = { workspace = true }
serde_json = { workspace = true }
tokio = { workspace = true }
tracing = { workspace = true }
tracing-subscriber = { workspace = true }
uuid = { workspace = true }


# taskforge-worker/Cargo.toml
[package]
name = "taskforge-worker"
version = "0.1.0"
edition = "2021"

[dependencies]
taskforge-core = { path = "../taskforge-core" }
serde = { workspace = true }
serde_json = { workspace = true }
tokio = { workspace = true }
tracing = { workspace = true }
tracing-subscriber = { workspace = true }
```

---

## Critical Patterns for This Section

### 1. Cargo Workspace Setup

taskforge is a workspace with three crates. The directory structure:

```
taskforge/
├── Cargo.toml           ← workspace root (NOT a package)
├── taskforge-core/      ← shared types, traits, errors
│   ├── Cargo.toml
│   └── src/lib.rs
├── taskforge-worker/    ← the job executor
│   ├── Cargo.toml
│   └── src/main.rs
└── taskforge-api/       ← HTTP management API
    ├── Cargo.toml
    └── src/main.rs
```

The workspace root `Cargo.toml`:
```toml
[workspace]
members = [
    "taskforge-core",
    "taskforge-worker",
    "taskforge-api",
]
resolver = "2"
```

Each member `Cargo.toml` references core:
```toml
# In taskforge-worker/Cargo.toml:
[dependencies]
taskforge-core = { path = "../taskforge-core" }
```

Build everything from the workspace root: `cargo build --workspace`

### 2. Type-State Pattern for Job Lifecycle

The type parameter `S` marks which state the job is in. The type changes at each transition:

```rust
use std::marker::PhantomData;

// State marker types - zero-sized, only exist at the type level:
pub struct Pending;
pub struct Running;
pub struct Completed;
pub struct Failed;

pub struct Job<S> {
    pub id: JobId,
    pub payload: serde_json::Value,
    pub retry_count: u32,
    _state: PhantomData<S>,  // "I carry the type S but no runtime data"
}

// Transition functions - the OLD state is CONSUMED, the NEW state is returned:
impl Job<Pending> {
    pub fn start(self, worker_id: WorkerId) -> Job<Running> {
        Job {
            id: self.id,
            payload: self.payload,
            retry_count: self.retry_count,
            _state: PhantomData,
        }
    }
}

impl Job<Running> {
    pub fn complete(self) -> Job<Completed> { /* ... */ }
    pub fn fail(self, error: String) -> Job<Failed> { /* ... */ }
}
```

Now `Job<Pending>` and `Job<Running>` are DIFFERENT types. Code that expects a pending job won't compile if you pass a running job.

### 3. The Generic Job Trait

```rust
use async_trait::async_trait;

#[async_trait]
pub trait JobHandler: Send + Sync {
    fn job_type(&self) -> &str;
    async fn handle(&self, payload: serde_json::Value) -> Result<serde_json::Value, JobError>;
}

// Register handlers by job type name:
pub struct HandlerRegistry {
    handlers: HashMap<String, Box<dyn JobHandler>>,
}

impl HandlerRegistry {
    pub fn register<H: JobHandler + 'static>(&mut self, handler: H) {
        self.handlers.insert(handler.job_type().to_string(), Box::new(handler));
    }

    pub async fn dispatch(&self, job_type: &str, payload: serde_json::Value)
        -> Result<serde_json::Value, JobError>
    {
        let handler = self.handlers.get(job_type)
            .ok_or_else(|| JobError::UnknownJobType(job_type.to_string()))?;
        handler.handle(payload).await
    }
}
```

### 4. Redis BRPOP - Blocking Queue Read

BRPOP blocks until an item is available in the queue:

```rust
// Push a job (producer side):
redis::cmd("LPUSH")
    .arg("jobs:pending")
    .arg(serde_json::to_string(&job_record)?)
    .query_async::<()>(&mut conn)
    .await?;

// Pop a job - blocks for up to 30 seconds, then returns None:
let result: Option<(String, String)> = redis::cmd("BRPOP")
    .arg("jobs:pending")
    .arg(30)  // timeout in seconds (0 = block forever)
    .query_async(&mut conn)
    .await?;

if let Some((_key, value)) = result {
    let job: JobRecord = serde_json::from_str(&value)?;
    // look up handler, execute, update status
    // ...
}
```

BRPOP returns `(list_name, value)` - that's why it's a tuple. The first element is which queue the item came from (useful when watching multiple queues).

### 5. Exponential Backoff for Retries

```rust
use tokio::time::{sleep, Duration};

pub fn retry_delay(attempt: u32) -> Duration {
    // 2^attempt seconds, capped at 5 minutes:
    let secs = 2u64.pow(attempt).min(300);
    // Add jitter to prevent thundering herd:
    let jitter = rand::random::<u64>() % (secs / 2).max(1);
    Duration::from_secs(secs + jitter)
}
```

---

## Milestone Expected Output

**Milestone: workspace builds:**

```
$ cargo build --workspace
   Compiling taskforge-core v0.1.0
   Compiling taskforge-worker v0.1.0
   Compiling taskforge-api v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 8.42s
```

**Milestone: job can be enqueued and dequeued:**

```bash
# Enqueue a job via the API:
$ curl -X POST http://localhost:3002/jobs \
  -H "Content-Type: application/json" \
  -d '{"type":"send_email","payload":{"to":"user@example.com","subject":"Hello"}}'

{"job_id":"01HQ...","status":"pending"}

# Worker picks it up automatically:
[worker-1] 14:23:01 Received job 01HQ... (type: send_email)
[worker-1] 14:23:01 Executing send_email for user@example.com
[worker-1] 14:23:02 Job 01HQ... completed in 1.02s
```

**Milestone: retry on failure:**

```
[worker-1] 14:23:01 Job 01HQ... failed: connection timeout (attempt 1/3)
[worker-1] 14:23:03 Retrying job 01HQ... in 2s
[worker-1] 14:23:05 Job 01HQ... failed: connection timeout (attempt 2/3)
[worker-1] 14:23:13 Retrying job 01HQ... in 8s
[worker-1] 14:23:21 Job 01HQ... failed: connection timeout (attempt 3/3)
[worker-1] 14:23:21 Job 01HQ... moved to dead letter queue
```

---

## Common Compile Errors

**`the trait JobHandler cannot be made into an object`**
Your `JobHandler` trait isn't object-safe. The most common cause: a method with a generic type parameter. Fix: use `serde_json::Value` as the parameter/return type instead of `T: Serialize + Deserialize`.

**`PhantomData: cannot infer type for type parameter S`**
You need to specify the state when constructing: `Job::<Pending> { ... _state: PhantomData }` or let the return type annotation infer it.

**`async-trait: the method cannot be dispatched on a trait object`**
Add `#[async_trait]` from the `async-trait` crate to both the trait definition and all `impl` blocks. Without it, `async fn` in traits don't work with `Box<dyn Trait>`.

**`BRPOP returns None even though items are in the queue`**
Check that you're PUSHing to `LPUSH` (left push) and POPping with `BRPOP` (which pops from the right = RPOP). If you `RPUSH` and `BRPOP`, it works. If you `LPUSH` and `BLPOP`, it also works. Mixing them empties the wrong end.

---

## The Generic Worker: The Key Design Challenge

The worker binary needs to handle ANY job type - `SendEmailJob`, `ResizeImageJob`,
whatever you register. But the payload stored in Redis is just JSON with a type
name. How does the worker know what to deserialize it into?

The answer: a job handler registry.

```
Worker has:
    handlers: HashMap<String, Box<dyn JobHandler>>

Where JobHandler is:
    #[async_trait]
    trait JobHandler: Send + Sync {
        async fn handle(&self, payload: serde_json::Value) -> Result<serde_json::Value, JobError>;
    }

Each concrete job type implements JobHandler:
    struct SendEmailHandler;

    #[async_trait]
    impl JobHandler for SendEmailHandler {
        async fn handle(&self, payload: serde_json::Value) -> Result<serde_json::Value, JobError> {
            let job: SendEmailJob = serde_json::from_value(payload)
                .map_err(|e| JobError::Deserialization(e.to_string()))?;
            
            // Chain both fallible operations with and_then - no unwrap in production paths
            job.execute()
                .await
                .map_err(|e| JobError::Execution(e.to_string()))
                .and_then(|output| {
                    serde_json::to_value(output)
                        .map_err(|e| JobError::Execution(e.to_string()))
                })
        }
    }

Worker loop:
    1. BRPOP from queue (blocking pop)
    2. Deserialize the envelope: { type: "send_email", payload: {...} }
    3. Look up handler: registry.handlers.get("send_email")
    4. Call handler.handle(payload)
    5. Update job status in Redis
```

The worker is registered at startup:

```rust
let mut registry = JobRegistry::new();
registry.register("send_email", Box::new(SendEmailHandler));
registry.register("resize_image", Box::new(ResizeImageHandler));
// Add more job types here
```

---

## Engineering Approach: Type-Driven Design + Invariants

Define the invariants for taskforge before writing a line of code.

**Job invariants:**
- A Pending job: has no worker assigned, has retry_count 0
- A Running job: has exactly one worker assigned (WorkerId), has a started_at timestamp, is in at most one worker's active jobs
- A Completed job: has a result (can be empty JSON), has a completed_at timestamp, is never retried
- A Failed job: has an error message, retry_count > 0, either becomes Pending (retry) or Dead (max retries exceeded)
- A Dead job: has retry_count = max_retries, is never retried, requires manual intervention

**Invariants that must never be violated:**
- No job transitions directly from Pending to Completed
- No job has retry_count > max_retries while in Pending or Running state
- No two workers claim the same job simultaneously

Now: which of these can you enforce with types (type-state, newtypes), and which require runtime logic?

Types can enforce: state transitions (`Job<Pending>` → `Job<Running>` requires a `WorkerId`), ID mixing (you can't pass a `WorkerId` where a `JobId` is expected).

Runtime logic handles: "no two workers claim the same job" (this requires atomic compare-and-swap in Redis).

This analysis is what you do before writing code.

---

## What You Bring From Sections 1-6

This section is where every previous section's knowledge compounds:

- **S1 Result/error handling:** your custom `JobError` uses the same `thiserror` pattern from S3, with variants you've been building since S1.
- **S2 traits:** the `Collector` trait from S2 was your first trait. The `Job` trait is the same concept, now with async methods and associated types.
- **S3 Arc<Mutex<T>>:** the in-memory job repository uses `Arc<Mutex<HashMap<JobId, JobRecord>>>`. You've been writing this since S3.
- **S4 async/Tokio:** every job execution is an async task. `tokio::spawn`, `JoinSet`, `CancellationToken` - all from S4.
- **S5 REST API:** the HTTP management API uses the same Axum handlers, extractors, and `AppError` pattern as S5.
- **S6 Redis:** `BRPOP` for blocking queue reads, `HSET` for job status, `EXPIRE` for worker heartbeats - same Redis you learned in S6, different usage.
- **S6 pub/sub:** worker-to-coordinator communication uses the same pub/sub pattern from S6.

---

## How This Project Works in Rust - The Full Picture

The task worker has three moving parts that work together.

**The job queue (Redis Lists):** a job is pushed to a Redis list with `LPUSH`. Workers call `BRPOP` to atomically dequeue a job. The "atomic" part matters: `BRPOP` guarantees only one worker gets each job, even with 10 workers running simultaneously. This is the concurrency safety you get from Redis for free.

**The worker loop:** each worker runs a tokio task in a loop. It calls `BRPOP` (which blocks on Redis until a job is available), deserializes the job payload from JSON (using serde), calls the appropriate job handler (looked up by job type name in the handler registry), then updates the job status in Redis (`HSET` on the job's hash key). If the handler returns an error, it increments the retry count and either re-queues the job (`LPUSH` back to the queue) or marks it dead.

**The job registry:** this is the generic piece. The worker doesn't know about specific job types at compile time. Instead, it has a `HashMap<String, Box<dyn JobHandler>>` mapping job type names to handler objects. When a job arrives with type `"send_email"`, it looks up `"send_email"` in the registry, calls `handle(raw_json_payload)`, and gets back a `Result<JsonValue, JobError>`. The generics in the `Job` trait are erased into `Box<dyn JobHandler>` at this boundary - that's the dynamic dispatch tradeoff.

---

## Milestones

Work through these in order. Each builds on the previous.

**Milestone 1**: Define the `Job` trait. Implement `SendEmailJob` that prints
to stdout instead of sending email. Write a unit test that calls `execute()`.

**Milestone 2**: Implement the `JobRepository` trait. Implement
`InMemoryJobRepository` with `HashMap` and `VecDeque`. Write tests using it.

**Milestone 3**: Implement `RedisJobRepository`. Use `LPUSH` to add jobs,
`BRPOP` to consume them, `HSET` to update status. Write integration tests
against real Redis.

**Milestone 4**: Build the `Worker` struct. It holds a `JobRegistry`,
a `JobRepository`, and a worker ID. The main loop: BRPOP → deserialize →
look up handler → execute → update status. Handle errors gracefully.

**Milestone 5**: Implement the type-state `Job<S>` lifecycle. Add transitions:
`assign()`, `complete()`, `fail()`. Make the worker use these transitions.

**Milestone 6**: Add retry logic. On failure, check retry count. If under
the limit, re-queue with backoff delay (use a Redis sorted set for scheduled
retries). After max retries, move to `queue:dead`.

**Milestone 7**: Build the REST API with Axum. `POST /jobs`, `GET /jobs/:id`,
`DELETE /jobs/:id`. Wire handlers → service → repository as described in Doc 04.

**Milestone 8**: Add worker heartbeat. Every 10 seconds, each worker runs
`SET worker:{id}:heartbeat "alive" EX 30`. On startup, detect workers whose
heartbeat has expired and re-queue any jobs they were processing.

**Milestone 9**: Write integration tests using `testcontainers-rs`. The tests
spin up Redis in Docker - no local Redis required. Test the full flow: submit
via API → worker picks up → status updates.

**Milestone 10**: Write Criterion benchmarks for job throughput. Measure
push/pop operations per second. Measure end-to-end latency for the in-memory
repository. Establish a baseline.

**Milestone 11**: Set up GitHub Actions CI with four stages: format+clippy,
tests with Redis service container, security audit, release build.

---

## Hints (Not Solutions)

**BRPOP blocking semantics**: `BRPOP key 5` blocks for up to 5 seconds waiting
for a job. When a job arrives, it returns immediately. This is more efficient
than polling with `RPOP` in a loop. Use 5-second timeouts so workers can check
for shutdown signals periodically.

**Job envelope format**: Store jobs in Redis as a JSON object with a type
discriminator and a payload:
```json
{ "type": "send_email", "id": "...", "payload": { "to": "..." }, "retry_count": 0 }
```

**Serialization for the generic worker**: Use `serde_json::to_value(job)` to
serialize and `serde_json::from_value::<J>(v)` to deserialize. The handler
receives a `Value` and knows what to deserialize it into.

**Heartbeat implementation**: `tokio::spawn` a separate task that runs
`SET worker:{id}:heartbeat "alive" EX 30` every 10 seconds. The TTL (EX 30)
ensures that if the worker crashes, the key disappears within 30 seconds.

**Dead worker detection**: On startup, scan all `worker:*:heartbeat` keys with
`KEYS worker:*:heartbeat` (or better, `SCAN`). Any worker whose heartbeat key
is missing but appears in `workers:active` is dead. Re-queue their in-progress jobs.

**async_trait for dyn compatibility**: Async functions in traits are stable in
Rust 1.75+, but using them behind `dyn Trait` still requires boxing the future.
The `async_trait` crate handles this automatically. Add it to `Cargo.toml` and
annotate your traits with `#[async_trait]`.

---

## Stretch Goals

Once all milestones are complete, consider these extensions:

**Priority within queues**: Instead of separate `queue:high`/`queue:normal`/`queue:low`
lists, use a single Redis Sorted Set where the score combines priority and submission
timestamp: `score = priority * 1e12 - unix_timestamp_ms`. `BZPOPMIN` pops the
highest-priority (lowest score) job. Jobs with equal priority are processed FIFO.

**Scheduled jobs**: A job with `run_at: DateTime` should not execute immediately.
Store it in a sorted set `jobs:scheduled` scored by the Unix timestamp of `run_at`.
A separate scheduler process runs every second, checks for due jobs with
`ZRANGEBYSCORE jobs:scheduled 0 now`, and moves them to the appropriate queue.

**Job dependencies**: Job B should not start until Job A completes. Store
dependencies in Redis. When a job completes, check if any other jobs were
waiting on it and, if so, move them from "waiting" to the queue.

**Web dashboard**: A simple HTML page (no framework needed, just plain HTML + CSS)
served at `/dashboard` showing: current queue depths, active workers and their
current jobs, recent job history with status, and a form to submit test jobs.
Use server-sent events for live updates.

**Rate limiting**: Limit concurrent jobs of a specific type. For example: at most
3 `ResizeImageJob` tasks running simultaneously (to avoid overloading the image
processing library). Use a Redis counter with `INCR`/`DECR` and check before
accepting a job.
