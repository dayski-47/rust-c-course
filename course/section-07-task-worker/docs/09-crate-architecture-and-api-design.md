# Doc 09 — Crate Architecture and API Design

🟡 A well-designed crate is easy to use correctly and hard to use wrong. This doc covers the structural decisions that make the difference between a library that everyone reaches for and one that everyone works around.

taskforge is organized as a workspace of three crates. This is the right time to talk about how Rust workspaces work, how to structure public APIs, and what design choices make a crate feel polished rather than accidental.

---

## Workspace Structure

A Rust workspace is a single repository containing multiple crates that share a `Cargo.lock` and can depend on each other:

```toml
# /Cargo.toml (workspace root)
[workspace]
members = [
    "taskforge-core",
    "taskforge-worker",
    "taskforge-api",
]
resolver = "2"  # Use the v2 feature resolver — required for most modern crates

[workspace.dependencies]
# Declare shared dependency versions once at the workspace level
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["sqlite", "runtime-tokio", "macros"] }
uuid = { version = "1", features = ["v4"] }
thiserror = "1"
tracing = "0.1"
redis = { version = "0.24", features = ["tokio-comp"] }
```

Member crates inherit versions from the workspace:

```toml
# taskforge-core/Cargo.toml
[package]
name = "taskforge-core"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { workspace = true }      # uses workspace version, no duplication
serde = { workspace = true }
sqlx = { workspace = true }
uuid = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }
```

**Benefits of `workspace.dependencies`:**
- Version conflicts are caught at workspace level
- One place to bump a version across all crates
- `cargo update` works uniformly

---

## Module Layout Conventions

Inside `taskforge-core`, organize by domain, not by file type:

```text
taskforge-core/
└── src/
    ├── lib.rs           # Public API: re-exports only
    ├── error.rs         # Error types (all crate errors in one place)
    ├── job/
    │   ├── mod.rs       # pub use job::*; — re-exports the job domain
    │   ├── types.rs     # Job, JobId, JobStatus, Priority enums
    │   ├── lifecycle.rs # JobHandle<S> type-state, transitions
    │   └── repository.rs # JobRepository trait
    ├── queue/
    │   ├── mod.rs
    │   ├── commands.rs  # RedisCmd implementations (doc 06)
    │   └── scheduler.rs # Priority scheduling logic
    ├── worker/
    │   ├── mod.rs
    │   ├── executor.rs  # Job execution engine
    │   └── pool.rs      # Worker pool management
    └── auth/
        ├── mod.rs
        └── tokens.rs    # Capability tokens (doc 05)
```

```rust
// lib.rs — curate the public API with explicit re-exports
mod error;
mod job;
mod queue;
mod worker;
mod auth;

// Only re-export what external callers need:
pub use error::TaskforgeError;
pub use job::{Job, JobId, JobStatus, Priority, JobHandle};
pub use job::repository::JobRepository;
pub use queue::Scheduler;
pub use worker::WorkerPool;
pub use auth::{SessionToken, WorkerAdminToken, WorkerProcessToken};
```

Users write `use taskforge_core::JobId`, not `use taskforge_core::job::types::JobId`. The module tree is an implementation detail.

---

## Visibility Modifiers in Practice

```rust
// Public to everyone
pub struct JobId(pub Uuid);

// Public to this crate only — internal helpers, capabilities construction
pub(crate) struct WorkerAdminToken { _private: () }
pub(crate) fn issue_admin_token() -> WorkerAdminToken { WorkerAdminToken { _private: () } }

// Public to the parent module — fine-grained within a domain
pub(super) fn validate_queue_depth(depth: u64) -> bool { depth < 10_000 }

// Private — implementation detail within this file
fn format_redis_key(job_id: &JobId) -> String { format!("job:{}", job_id.0) }
```

A useful rule of thumb: make things as private as they can be, then loosen access when code in another module legitimately needs it. Starting public and trying to restrict it later is hard — published API is a contract.

---

## Ergonomic Parameter Patterns

Well-designed APIs minimize friction at the call site. Three patterns cover most of it.

### `impl Into<T>`: Accept Anything Convertible

```rust
// ❌ Friction: callers must convert
pub async fn enqueue(&self, queue: String, payload: Vec<u8>) {}
enqueue("queue:high".to_string(), serde_json::to_vec(&job).unwrap()).await;

// ✅ Ergonomic: accept anything that becomes the right type
pub async fn enqueue(
    &self,
    queue: impl Into<String>,
    payload: impl Into<Vec<u8>>,
) {}
enqueue("queue:high", serde_json::to_vec(&job).unwrap()).await;
// &str → String happens inside the function
```

The conversion cost is identical — `impl Into<T>` just removes boilerplate from the call site. Accept `impl Into<T>` when you need ownership. Accept `impl AsRef<T>` when you only need to borrow.

### `impl AsRef<T>`: Borrow Without Owning

```rust
use std::path::Path;

// Accepts &str, String, PathBuf, &Path — all compile
pub fn log_job_artifact(path: impl AsRef<Path>) {
    let path = path.as_ref();
    tracing::info!(?path, "artifact written");
}

// Works for string-like parameters too:
pub fn find_jobs_by_type(job_type: impl AsRef<str>) -> Vec<Job> {
    let t = job_type.as_ref();
    // ... query ...
    vec![]
}

find_jobs_by_type("email_notification");   // &str ✅
find_jobs_by_type(String::from("resize")); // String ✅
```

### `Cow<T>`: Clone on Write

Use `Cow` when most callers pass a borrowed value but occasional callers need to modify it:

```rust
use std::borrow::Cow;

/// Normalize a job type string — lowercase, trim whitespace.
/// Only allocates if the string needs modification.
pub fn normalize_job_type(s: &str) -> Cow<'_, str> {
    let trimmed = s.trim();
    if trimmed.chars().all(|c| c.is_ascii_lowercase() || c == '_') {
        Cow::Borrowed(trimmed)  // No allocation — string already valid
    } else {
        Cow::Owned(trimmed.to_lowercase().replace(' ', "_"))  // Allocate only when needed
    }
}
```

**Quick reference:**

| Pattern | Use when |
|---------|----------|
| `impl Into<T>` | Function takes ownership, caller has a compatible type |
| `impl AsRef<T>` | Function borrows, caller might have `String`, `&str`, `PathBuf`, etc. |
| `Cow<T>` | Function sometimes needs to modify the input, sometimes doesn't |

---

## Public API Design Checklist

Eight principles that distinguish a polished crate from a draft:

**1. Return `Result`, not `panic!`**

```rust
// ❌ Panics on invalid input — caller has no recovery path
pub fn parse_job_id(s: &str) -> JobId {
    JobId(s.parse().expect("invalid UUID"))
}

// ✅ Returns Result — caller decides how to handle errors
pub fn parse_job_id(s: &str) -> Result<JobId, TaskforgeError> {
    let id = s.parse().map_err(|_| TaskforgeError::InvalidJobId(s.to_string()))?;
    Ok(JobId(id))
}
```

**2. Implement standard traits**

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct JobId(pub Uuid);

impl std::fmt::Display for JobId {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl std::str::FromStr for JobId {
    type Err = TaskforgeError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        parse_job_id(s)
    }
}
```

Every type that represents a domain concept should derive `Debug`. Public types that users store in collections need `Clone + PartialEq + Eq + Hash`. Types that serialize over the wire need `Serialize + Deserialize`.

**3. Mark important return types `#[must_use]`**

```rust
#[must_use = "dropping the token immediately cancels the job reservation"]
pub struct JobReservation {
    job_id: JobId,
    // ...
}

#[must_use]
pub async fn try_reserve_job(&self, queue: &str) -> Option<JobReservation> {
    // Returns None if queue is empty — returning Some and ignoring it is a bug
    None
}
```

**4. Seal traits you don't want external code to implement**

```rust
// Internal seal — prevents external implementations
mod private {
    pub trait Sealed {}
}

// External callers can USE this trait but cannot IMPLEMENT it
pub trait StorageBackend: private::Sealed {
    async fn save_job(&self, job: &Job) -> Result<(), TaskforgeError>;
    async fn load_job(&self, id: &JobId) -> Result<Option<Job>, TaskforgeError>;
}

// Only types in this crate can implement Sealed → only we implement StorageBackend
pub struct RedisBackend { /* ... */ }
impl private::Sealed for RedisBackend {}
impl StorageBackend for RedisBackend {
    async fn save_job(&self, job: &Job) -> Result<(), TaskforgeError> { Ok(()) }
    async fn load_job(&self, id: &JobId) -> Result<Option<Job>, TaskforgeError> { Ok(None) }
}
```

Sealed traits let you evolve the interface without breaking callers — add a method to `StorageBackend` without worrying that external implementors break.

**5. Use `#[non_exhaustive]` for public enums**

```rust
#[derive(Debug, Clone, PartialEq)]
#[non_exhaustive]  // Adding variants later is NOT a semver break
pub enum JobStatus {
    Pending,
    Running,
    Completed,
    Failed,
    Cancelled,
}
```

External code must use a wildcard arm in match statements:
```rust
match status {
    JobStatus::Completed => record_success(),
    JobStatus::Failed => schedule_retry(),
    _ => {}  // Required — can't exhaust non_exhaustive enum
}
```

This means adding `JobStatus::Retrying` in a future version is a minor release, not a major one.

**6. Accept references, return owned**

```rust
// ✅ Take reference, return owned — maximizes caller flexibility
pub async fn create_job(&self, spec: &JobSpec) -> Result<JobId, TaskforgeError> {
    // Creates a copy of spec internally if needed — caller keeps theirs
    Ok(JobId(Uuid::new_v4()))
}

// ❌ Taking owned forces the caller to clone if they need spec afterward
pub async fn create_job_bad(&self, spec: JobSpec) -> Result<JobId, TaskforgeError> {
    Ok(JobId(Uuid::new_v4()))
}
```

**7. Builder pattern for complex construction**

```rust
#[derive(Default)]
pub struct JobSpecBuilder {
    job_type: Option<String>,
    payload: Option<Vec<u8>>,
    priority: Priority,
    max_retries: u32,
    timeout_secs: u64,
}

impl JobSpecBuilder {
    pub fn new() -> Self { Self::default() }
    pub fn job_type(mut self, t: impl Into<String>) -> Self { self.job_type = Some(t.into()); self }
    pub fn payload(mut self, p: impl Into<Vec<u8>>) -> Self { self.payload = Some(p.into()); self }
    pub fn priority(mut self, p: Priority) -> Self { self.priority = p; self }
    pub fn max_retries(mut self, n: u32) -> Self { self.max_retries = n; self }
    pub fn timeout_secs(mut self, secs: u64) -> Self { self.timeout_secs = secs; self }

    pub fn build(self) -> Result<JobSpec, TaskforgeError> {
        Ok(JobSpec {
            job_type: self.job_type.ok_or(TaskforgeError::MissingJobType)?,
            payload: self.payload.unwrap_or_default(),
            priority: self.priority,
            max_retries: self.max_retries,
            timeout_secs: self.timeout_secs,
        })
    }
}

// Clean call site:
let spec = JobSpecBuilder::new()
    .job_type("email_notification")
    .payload(serde_json::to_vec(&email_data)?)
    .priority(Priority::High)
    .max_retries(3)
    .build()?;
```

**8. Make invalid states unrepresentable in public types**

```rust
// ❌ Allows construction of invalid job spec
pub struct JobSpec {
    pub job_type: String,  // could be empty
    pub priority: u8,      // could be out of range
}

// ✅ Validates at construction — type carries proof of validity
pub struct JobSpec {
    job_type: ValidJobType,   // newtype with private constructor
    priority: Priority,        // enum — only valid values exist
}
```

---

## Feature Flags for Optional Backends

taskforge-core supports multiple storage backends. Use feature flags to make dependencies optional:

```toml
# taskforge-core/Cargo.toml
[features]
default = ["redis-backend"]
redis-backend = ["dep:redis"]
sqlite-backend = ["dep:sqlx"]
postgres-backend = ["dep:sqlx", "sqlx/postgres"]

[dependencies]
redis = { version = "0.24", optional = true }
sqlx = { version = "0.7", optional = true, features = ["runtime-tokio"] }
```

```rust
// In lib.rs — conditional compilation based on features
#[cfg(feature = "redis-backend")]
pub mod redis_backend;

#[cfg(feature = "sqlite-backend")]
pub mod sqlite_backend;
```

Users who only need Redis don't pay for SQLx compilation. Users who need both can enable both features.

---

## Documentation as Part of the API

Documentation is part of the public API contract. Every public item needs at least one line:

```rust
/// A unique identifier for a job in the task queue.
///
/// Job IDs are UUID v4 values, generated at enqueue time.
/// They are stable across worker restarts and can be used
/// to check job status via the API.
///
/// # Examples
///
/// ```
/// use taskforge_core::JobId;
///
/// let id: JobId = "550e8400-e29b-41d4-a716-446655440000".parse()?;
/// println!("Job ID: {id}");
/// # Ok::<(), taskforge_core::TaskforgeError>(())
/// ```
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct JobId(pub Uuid);
```

Documentation examples in `///` blocks are run as tests by `cargo test --doc`. They are the most valuable documentation: they're always correct (because the CI runs them), and they show exactly how to use the type.

For panic conditions, always document them:

```rust
/// Returns the job with the given ID.
///
/// # Panics
///
/// Panics if called from outside a Tokio async context.
/// Use `try_get_job` from sync contexts.
pub async fn get_job(&self, id: &JobId) -> Result<Job, TaskforgeError> { /* ... */ }
```

---

## The Library / Binary Split

`taskforge-core` is a library — it has no `main` function, no hard-coded configuration, no logging setup. The binaries (`taskforge-worker`, `taskforge-api`) own these:

```rust
// taskforge-worker/src/main.rs
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Initialize tracing — only the binary does this
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
        .init();

    // Load configuration — only the binary knows where config comes from
    let config = Config::from_env()?;

    // Build the library objects
    let core = taskforge_core::WorkerPool::new(&config.redis_url, config.concurrency).await?;

    // Run
    core.run_until_shutdown().await
}
```

This separation means the library can be tested without a running Redis instance (with mock backends), and multiple binaries can share the same core logic with different startup behaviors.
