# Doc 04 - Service Architecture and Dependency Injection

In Section 6 you built an API where handlers talked directly to the database.
That works for simple projects. But once you need to test a handler without
a live Redis instance, or swap the storage backend, or run the same logic from
a background worker and an HTTP handler, you hit the wall. This document explains
the layered architecture that makes Rust services both testable and maintainable.

---

## The Problem With Logic in Handlers

When business logic lives in the handler, several things go wrong:

- **Untestable**: a handler test requires an HTTP request, which requires Axum,
  which requires a running server, which requires infrastructure
- **Duplicated**: two handlers that need the same logic copy and paste it
- **Coupled**: changing how data is stored means touching the handler code
- **Hard to reason about**: HTTP concerns and business concerns are mixed together

The alternative is layers.

---

## The Three-Layer Architecture 🟡

```
HTTP Request
     │
     ▼
┌─────────────────────────┐
│         Handler         │  Parses HTTP, formats HTTP response
│  (knows about HTTP)     │  No business logic
└──────────┬──────────────┘
           │ calls
           ▼
┌─────────────────────────┐
│         Service         │  Business logic
│  (knows nothing about   │  "push this job, return its ID"
│   HTTP or databases)    │  Validates rules, orchestrates
└──────────┬──────────────┘
           │ calls
           ▼
┌─────────────────────────┐
│       Repository        │  Data access
│  (knows about Redis)    │  "store this, retrieve that"
│                         │  No business logic
└─────────────────────────┘
```

**Handler layer**: Deserializes the request body, calls the service, serializes
the response. If the service returns an error, the handler maps it to an HTTP
status code. That is the handler's entire job.

**Service layer**: Contains the "what." Push a job. Check if a job exists. Cancel
a job. This layer has no knowledge of HTTP status codes, Redis commands, or
JSON serialization. It works in terms of domain types.

**Repository layer**: Contains the "how." LPUSH to a Redis list. HSET to a Redis hash.
BRPOP to consume from a queue. This layer has no knowledge of business rules.

The critical insight: **the service depends on the repository through a trait,
not through a concrete type.** This is dependency injection via Rust traits.

---

## The Repository Trait 🟡

```rust
use async_trait::async_trait;

#[async_trait]
pub trait JobRepository: Send + Sync {
    /// Push a job onto its queue
    async fn push(&self, job: &Job<Pending>) -> Result<(), StorageError>;
    
    /// Pop the next job from the specified queue (blocking up to timeout)
    async fn pop(
        &self,
        queue: &str,
        timeout: Duration,
    ) -> Result<Option<Job<Pending>>, StorageError>;
    
    /// Update job status (when running, completed, failed, etc.)
    async fn update_status(
        &self,
        id: JobId,
        status: JobStatus,
    ) -> Result<(), StorageError>;
    
    /// Retrieve a job's full record
    async fn get(&self, id: JobId) -> Result<Option<JobRecord>, StorageError>;
    
    /// List jobs with optional filters
    async fn list(
        &self,
        filter: JobFilter,
    ) -> Result<Vec<JobRecord>, StorageError>;
    
    /// Cancel a pending job
    async fn cancel(&self, id: JobId) -> Result<bool, StorageError>;
}
```

Why `Send + Sync`? Because this trait object will live behind an `Arc` and be
shared between async tasks running on multiple threads. The bounds ensure
thread safety.

Why `async_trait`? Async functions in traits are stable in Rust 1.75+, but
using them with `dyn Trait` still requires the `async_trait` crate or manual
boxing. In taskforge, `async_trait` is the practical choice.

---

## Concrete Implementations 🟢

Two implementations of the same trait:

```rust
// Production: Redis
pub struct RedisJobRepository {
    client: redis::aio::ConnectionManager,
}

#[async_trait]
impl JobRepository for RedisJobRepository {
    async fn push(&self, job: &Job<Pending>) -> Result<(), StorageError> {
        let payload = serde_json::to_string(job)
            .map_err(StorageError::Serialization)?;
        
        let queue_key = format!("queue:{}", job.queue);
        
        let mut conn = self.client.clone();
        redis::cmd("LPUSH")
            .arg(&queue_key)
            .arg(&payload)
            .query_async(&mut conn)
            .await
            .map_err(StorageError::Redis)?;
        
        Ok(())
    }
    
    // ... other methods
}

// Testing: In-memory
pub struct InMemoryJobRepository {
    jobs: Arc<Mutex<HashMap<JobId, JobRecord>>>,
    queues: Arc<Mutex<HashMap<String, VecDeque<Job<Pending>>>>>,
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
        let mut queues = self.queues.lock().await;
        queues
            .entry(job.queue.clone())
            .or_default()
            .push_back(job.clone());
        Ok(())
    }
    
    // ... other methods
}
```

The `InMemoryJobRepository` is fast, deterministic, and requires no external
processes. It is your primary testing tool.

---

## The Service Layer 🟡

The service receives an `Arc<dyn JobRepository>`. It does not know whether
that repository talks to Redis, uses in-memory storage, or does something else.

```rust
pub struct JobService {
    repository: Arc<dyn JobRepository>,
}

impl JobService {
    pub fn new(repository: Arc<dyn JobRepository>) -> Self {
        JobService { repository }
    }
    
    pub async fn submit_job(
        &self,
        job_type: &str,
        payload: serde_json::Value,
        queue: &str,
    ) -> Result<JobId, ServiceError> {
        let job = Job::new(queue, payload);
        let id = job.id;
        
        self.repository
            .push(&job)
            .await
            .map_err(ServiceError::Storage)?;
        
        Ok(id)
    }
    
    pub async fn get_status(&self, id: JobId) -> Result<JobRecord, ServiceError> {
        self.repository
            .get(id)
            .await
            .map_err(ServiceError::Storage)?
            .ok_or(ServiceError::NotFound(id))
    }
    
    pub async fn cancel_job(&self, id: JobId) -> Result<(), ServiceError> {
        let cancelled = self.repository
            .cancel(id)
            .await
            .map_err(ServiceError::Storage)?;
        
        if !cancelled {
            return Err(ServiceError::NotFound(id));
        }
        
        Ok(())
    }
}
```

Notice: no JSON, no HTTP status codes, no Axum types. Pure business logic.

---

## The Handler Layer 🟢

The handler knows about HTTP. That is all it knows.

```rust
use axum::{extract::State, http::StatusCode, Json};

#[derive(Clone)]
pub struct AppState {
    pub job_service: Arc<JobService>,
}

pub async fn submit_job(
    State(state): State<AppState>,
    Json(body): Json<SubmitJobRequest>,
) -> Result<Json<SubmitJobResponse>, (StatusCode, String)> {
    let id = state.job_service
        .submit_job(&body.job_type, body.payload, &body.queue)
        .await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
    
    Ok(Json(SubmitJobResponse { id }))
}

pub async fn get_job_status(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> Result<Json<JobRecord>, (StatusCode, String)> {
    let job_id = JobId::from(id);
    
    state.job_service
        .get_status(job_id)
        .await
        .map(Json)
        .map_err(|e| match e {
            ServiceError::NotFound(_) => (StatusCode::NOT_FOUND, "not found".into()),
            _ => (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()),
        })
}
```

---

## Building the Dependency Graph in main() 🟡

The key moment is `main()` - this is where everything gets assembled.

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Create infrastructure
    let redis_client = redis::Client::open("redis://127.0.0.1/")?;
    let connection_manager = redis::aio::ConnectionManager::new(redis_client).await?;
    
    // 2. Create repository (implements JobRepository)
    let repository: Arc<dyn JobRepository> = Arc::new(
        RedisJobRepository::new(connection_manager)
    );
    
    // 3. Inject repository into service
    let job_service = Arc::new(JobService::new(repository));
    
    // 4. Inject service into handlers via AppState
    let state = AppState { job_service };
    
    // 5. Build router
    let app = Router::new()
        .route("/jobs", post(submit_job))
        .route("/jobs/:id", get(get_job_status))
        .with_state(state);
    
    // 6. Run server
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    axum::serve(listener, app).await?;
    
    Ok(())
}
```

The dependency graph flows downward: main creates Redis → wraps in Repository →
injects into Service → injects into Handler. Each layer only knows about the
layer below it, and only through a trait.

In tests, you swap step 2: `Arc::new(InMemoryJobRepository::new())`.
Everything else is identical.

---

## Feature Flags for Optional Storage Backends 🟡

You can conditionally compile the Redis backend using feature flags:

```toml
# Cargo.toml
[features]
default = ["redis"]
redis = ["dep:redis"]

[dependencies]
redis = { version = "0.25", features = ["tokio-comp", "connection-manager"], optional = true }
```

```rust
#[cfg(feature = "redis")]
pub mod redis_repository;

#[cfg(feature = "redis")]
pub use redis_repository::RedisJobRepository;
```

This means the `taskforge-core` crate compiles without Redis, and only the
`taskforge-worker` and `taskforge-api` crates pull it in. Tests in core can
use `InMemoryJobRepository` with no Redis dependency at all.

---

## The Tradeoff: Dynamic Dispatch Cost 🟡

`Arc<dyn JobRepository>` uses dynamic dispatch. Every call to `push()` or `pop()`
goes through a vtable pointer. The concrete implementation is not known until runtime.

For a job queue, this is completely acceptable. The overhead of a vtable call is
roughly 1-2 nanoseconds. A Redis round-trip is roughly 200-500 microseconds.
The dynamic dispatch overhead is less than 0.001% of the total time.

If you were writing a hot-path math library where millions of calls happen per
second, the tradeoff would be different. For I/O-bound code like a job queue,
`dyn Trait` is the right call because it makes the code dramatically more flexible
and testable.

---

## Structuring the Crate

Where things live:

```
taskforge-core/src/
├── lib.rs                  # pub use of everything
├── job.rs                  # Job<S> type states, JobId newtype
├── traits/
│   ├── mod.rs
│   ├── repository.rs       # pub trait JobRepository
│   └── handler.rs          # pub trait JobHandler
├── service.rs              # JobService (depends on trait, not impl)
├── error.rs                # StorageError, ServiceError
└── models.rs               # JobRecord, JobFilter, JobStatus enums

taskforge-core/src/storage/ # concrete implementations
├── mod.rs
├── memory.rs               # InMemoryJobRepository (always compiled)
└── redis.rs                # RedisJobRepository (cfg feature = "redis")
```

The trait is public. The service is public. Concrete implementations are in
`storage/` - their visibility depends on what you want to expose.

---

## How It Breaks

**The trait object vs generic argument choice.** `Arc<dyn JobRepository>` is flexible but uses virtual dispatch. `JobService<R: JobRepository>` with a generic argument means the concrete type is fixed at compile time. Choose wrong and you either pay for virtual dispatch you don't need, or you lose the flexibility to swap implementations.

**Circular dependencies in the service graph.** `ServiceA` depends on `ServiceB`, `ServiceB` depends on `ServiceA`. You can't construct either. Redesign to introduce an event bus or a mediator.

**State leaking across tests.** If your `InMemoryJobRepository` is shared between tests (e.g., stored in a static), one test's data appears in another test. Use a fresh instance per test.

**Feature flag drift.** You have `#[cfg(feature = "redis")]` around the Redis implementation, but you forget to update the CI to test both feature combinations. The non-Redis code path breaks silently.

---

## Common Mistakes

**1. Putting the repository trait in the same file as a concrete implementation.**
When traits and implementations are co-located, it is tempting to reference
implementation details. Keep the trait in `traits/repository.rs` where it
is pure interface.

**2. Making the service generic over the repository type instead of using `dyn`.**

```rust
// This compiles but creates unnecessary complexity
pub struct JobService<R: JobRepository> {
    repository: R,
}

// Prefer this - cleaner API, acceptable performance for I/O-bound work
pub struct JobService {
    repository: Arc<dyn JobRepository>,
}
```

The generic version forces everyone who touches `JobService` to carry the `R`
type parameter everywhere. For an I/O-bound system, the dynamic dispatch cost is
not worth the complexity.

**3. Cloning the repository for each request instead of sharing it.** The `Arc`
exists for this reason. Store `Arc<dyn JobRepository>` in `AppState`, and Axum
clones the `Arc` (not the underlying repository) for each request.

**4. Putting business logic in the repository.** If you find yourself writing
"if job has been retried 3 times, move to dead letter queue" in `RedisJobRepository`,
stop. That logic belongs in the service. The repository pushes and pops. The service
decides what to push and what to do after a pop.

**5. Not testing the service layer separately.** The point of the trait abstraction
is that you can test `JobService` with `InMemoryJobRepository` without any Redis.
If your only tests are end-to-end with Redis, you have wasted the architecture.
