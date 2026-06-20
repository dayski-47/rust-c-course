# Doc 03 — Type-State and Newtype Patterns

These two patterns are where Rust diverges most dramatically from every other
mainstream language. In Go or Python you validate data and then carry the raw
value around. In Rust you can encode the fact that something has been validated
directly into the type. Once you see how this works, you will wonder how you
ever wrote code without it.

---

## The Newtype Pattern: Making Bugs Impossible 🟢

A newtype is a tuple struct with a single field. It creates a brand new type
that is distinct from the inner type at the type-system level, even if it has
the same runtime representation.

```rust
struct JobId(Uuid);
struct WorkerId(Uuid);
```

Both are Uuids at runtime. But the compiler treats them as completely different
types. This matters when you write functions like:

```rust
fn assign(job: JobId, worker: WorkerId) { ... }
```

You cannot call `assign(worker_id, job_id)` — the compiler will reject it.
In a stringly-typed codebase, this kind of argument swap is the source of
real production bugs. With newtypes, it is a compile error.

The runtime cost: zero. The compiler erases the newtype wrapper entirely.
`JobId(uuid)` and `uuid` occupy the same memory layout.

### Implementing Useful Traits

A bare newtype has limited functionality. You usually want:

```rust
use std::fmt;
use std::ops::Deref;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
pub struct JobId(Uuid);

impl JobId {
    pub fn new() -> Self {
        JobId(Uuid::new_v4())
    }
    
    pub fn inner(&self) -> Uuid {
        self.0
    }
}

impl fmt::Display for JobId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "job:{}", self.0)
    }
}

impl From<Uuid> for JobId {
    fn from(id: Uuid) -> Self { JobId(id) }
}
```

Notice there is no `Deref` implementation here. That is intentional.

### When Not to Implement Deref

`Deref` lets a type automatically coerce to its inner type. Implementing it
on `JobId` would mean you can pass a `&JobId` anywhere a `&Uuid` is expected.
That sounds convenient, but it defeats the purpose of the newtype.

If you can pass a `JobId` anywhere a `Uuid` is expected, you have not prevented
the bug where someone passes a `WorkerId` instead — they just convert it to a
`Uuid` first and the compiler shrugs. Keep the boundary explicit. Provide
`.inner()` when you genuinely need the raw value.

The rule: implement `Deref` for smart pointers and transparent wrappers where
the wrapper IS the inner type in every meaningful sense (`Box<T>`, `Arc<T>`).
Do not implement it for domain newtypes where the distinction matters.

---

## Parse, Don't Validate 🟡

This principle sounds simple but changes how you design entire systems.

The bad approach: validate at the call site every time.

```rust
// This is how most code works — validate, then carry the raw value
fn send_email(to: String) {
    if !to.contains('@') {
        panic!("invalid email");
    }
    // ... but now anyone calling this without thinking
    // might pass invalid email, and you find out at runtime
}
```

The good approach: parse at the boundary, carry the proof.

```rust
pub struct Email(String);

#[derive(Debug)]
pub struct EmailError(String);

impl Email {
    pub fn parse(raw: &str) -> Result<Self, EmailError> {
        if raw.contains('@') && raw.len() > 3 {
            Ok(Email(raw.to_string()))
        } else {
            Err(EmailError(format!("'{}' is not a valid email", raw)))
        }
    }
    
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

// Now this function CANNOT receive an invalid email
// If you have an Email value, it is already validated
fn send_email(to: Email) {
    // No validation needed. The type is proof.
}
```

The validation happens exactly once — at the boundary where untrusted data enters
the system (CLI args, HTTP request body, Redis payload). After that, the type
carries the proof. You never need to re-validate.

This connects directly to taskforge's job payloads. Once a job is deserialized
from JSON into a concrete job type, the struct's fields hold validated data.
A `PathBuf` has been validated as a valid path. A custom `Email` type has
been validated as a valid email. You do not re-check.

---

## The Type-State Pattern: Encoding Transitions in Types 🔴

Type-state uses the type system to represent state. Instead of an enum field
that tracks what state an object is in, the state becomes a type parameter.
Invalid state transitions become compiler errors.

Here is the job lifecycle in taskforge:

```
Job<Pending>  --[assign(worker)]-->  Job<Running>
Job<Running>  --[complete(result)]--> Job<Completed>
Job<Running>  --[fail(error)]------> Job<Failed>
Job<Failed>   --[retry()]----------> Job<Pending>
Job<Failed>   --[discard()]--------> Job<Dead>
```

You cannot call `complete()` on a `Job<Pending>`. The method does not exist
on that type. You cannot call `retry()` on a `Job<Completed>`. The compiler
will tell you.

### Implementation

States are unit structs (zero-sized types):

```rust
pub struct Pending;
pub struct Running {
    pub started_at: std::time::Instant,
    pub worker_id: WorkerId,
}
pub struct Completed {
    pub finished_at: std::time::Instant,
}
pub struct Failed {
    pub error: String,
    pub attempt: u32,
}
pub struct Dead;
```

Note that states can carry data. `Running` knows when it started and which
worker picked it up. `Failed` knows the error message and how many times
we have retried.

The job struct uses `PhantomData` to carry the state without storing it:

```rust
use std::marker::PhantomData;

pub struct Job<S> {
    pub id: JobId,
    pub queue: String,
    pub payload: serde_json::Value,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub retry_count: u32,
    state: S,
    _phantom: PhantomData<S>,
}
```

Wait — here we actually store the state in the struct, because states can carry
data (`Running` has `started_at`). The `PhantomData` is only needed when you have
zero-sized state markers. Either way, the pattern is the same.

### State-Specific Methods

Each transition is an `impl` block for one specific state:

```rust
impl Job<Pending> {
    pub fn new(queue: &str, payload: serde_json::Value) -> Self {
        Job {
            id: JobId::new(),
            queue: queue.to_string(),
            payload,
            created_at: chrono::Utc::now(),
            retry_count: 0,
            state: Pending,
            _phantom: PhantomData,
        }
    }
    
    // Only pending jobs can be assigned to a worker
    pub fn assign(self, worker_id: WorkerId) -> Job<Running> {
        Job {
            id: self.id,
            queue: self.queue,
            payload: self.payload,
            created_at: self.created_at,
            retry_count: self.retry_count,
            state: Running {
                started_at: std::time::Instant::now(),
                worker_id,
            },
            _phantom: PhantomData,
        }
    }
}

impl Job<Running> {
    // Only running jobs can be completed
    pub fn complete(self) -> Job<Completed> {
        Job {
            id: self.id,
            queue: self.queue,
            payload: self.payload,
            created_at: self.created_at,
            retry_count: self.retry_count,
            state: Completed {
                finished_at: std::time::Instant::now(),
            },
            _phantom: PhantomData,
        }
    }
    
    // Only running jobs can fail
    pub fn fail(self, error: String) -> Job<Failed> {
        Job {
            id: self.id,
            queue: self.queue,
            payload: self.payload,
            created_at: self.created_at,
            retry_count: self.retry_count + 1,
            state: Failed { error, attempt: self.retry_count + 1 },
            _phantom: PhantomData,
        }
    }
    
    pub fn worker_id(&self) -> WorkerId {
        self.state.worker_id
    }
}

impl Job<Failed> {
    pub fn retry(self) -> Job<Pending> {
        Job {
            state: Pending,
            _phantom: PhantomData,
            ..self
        }
    }
    
    pub fn discard(self) -> Job<Dead> {
        Job {
            state: Dead,
            _phantom: PhantomData,
            ..self
        }
    }
    
    pub fn attempt_number(&self) -> u32 {
        self.state.attempt
    }
}
```

### Why State-Specific Methods Matter

Each transition **consumes self** and returns a new type. Once you call
`job.assign(worker_id)`, the original `Job<Pending>` is gone. You cannot
use it again. The move semantics enforce this.

This is the opposite of runtime state machines where you set `self.state = State::Running`
and nothing prevents you from accidentally calling `mark_complete()` on a `Pending` job.
With type-state, the error is at compile time.

### The Tradeoffs

Type-state has real costs:

- Code duplication: common fields appear in multiple impl blocks
- Storing type-state in Redis requires serialization to an enum or string anyway
- Functions that work on a job regardless of state need either a trait bound or
  you implement the method on each state separately

For taskforge, the workflow is: jobs live as type-state values within the worker process
for the duration of processing. When you write to Redis, you serialize to a plain enum.
When you read from Redis, you reconstruct the appropriate state. The type-state buys you
correctness within the processing logic where it matters most.

---

## The Builder Pattern as Type-State 🟡

Builder pattern using type-state is a clean way to enforce required configuration.
Instead of `Option<String>` fields that panic if not set, missing required fields
are caught at compile time.

```rust
struct NeedsQueue;
struct NeedsPayload;
struct Ready;

struct JobBuilder<S> {
    queue: Option<String>,
    payload: Option<serde_json::Value>,
    priority: u8,
    _state: PhantomData<S>,
}

impl JobBuilder<NeedsQueue> {
    pub fn new() -> Self {
        JobBuilder {
            queue: None,
            payload: None,
            priority: 5,
            _state: PhantomData,
        }
    }
    
    pub fn queue(self, q: &str) -> JobBuilder<NeedsPayload> {
        JobBuilder {
            queue: Some(q.to_string()),
            payload: self.payload,
            priority: self.priority,
            _state: PhantomData,
        }
    }
}

impl JobBuilder<NeedsPayload> {
    pub fn payload(self, p: serde_json::Value) -> JobBuilder<Ready> {
        JobBuilder {
            queue: self.queue,
            payload: Some(p),
            priority: self.priority,
            _state: PhantomData,
        }
    }
}

impl JobBuilder<Ready> {
    pub fn priority(mut self, p: u8) -> Self {
        self.priority = p;
        self
    }
    
    pub fn build(self) -> Job<Pending> {
        Job::new(
            &self.queue.unwrap(),
            self.payload.unwrap(),
        )
    }
}

// Caller is forced to set both required fields before build() is available
let job = JobBuilder::new()
    .queue("high")
    .payload(serde_json::json!({ "to": "user@example.com" }))
    .priority(10)  // optional
    .build();      // only available now
```

`build()` does not exist on `JobBuilder<NeedsQueue>`. Calling it in the wrong
order is a compile error, not a runtime panic with a missing-field message.

---

## How It Breaks

**Type-state becoming unwieldy.** If you have 10 states and 20 methods, you end up with 200 `impl` blocks. At some point, a runtime state enum is cleaner and easier to reason about. Type-state shines for 3-5 states with clear transitions.

**Newtype wrapper forgotten.** You create `Email(String)`, but your database layer accepts `String`. You call `.0` to unwrap everywhere. This defeats the purpose — create `From<Email> for String` and use it at the database boundary.

**Phantom type confusion.** `PhantomData<S>` makes the struct carry a type `S` it doesn't store. But if you're cloning the struct, you need `PhantomData<fn() -> S>` to avoid the compiler adding a spurious `S: Clone` bound.

**Builder state explosion.** If your builder has many independent optional fields, you don't need type-state for each one. Just use `Option<T>` fields and validate at build time.

---

## Common Mistakes

**1. Using type-state when a runtime state machine is simpler.** Type-state is
excellent for protocol correctness within a single function or module. For
tracking job state across Redis round-trips, you need serializable enums anyway.
Do not force type-state where it does not fit.

**2. Making the state parameter public on a newtype-like struct.** If users can
construct `Job<Dead>` directly by building the struct literal, your type-state
provides no protection. Keep the state private and enforce transitions through
the constructors and methods.

**3. Forgetting to implement `Debug` and `Clone` on state types.** When your
state carries data (like `Running { started_at, worker_id }`), you need those
derives on the state structs too, not just on the outer `Job<S>`.

**4. Trying to put `Job<Pending>` and `Job<Running>` in the same `Vec`.** They
are different types. If you need a mixed collection, use an enum:

```rust
pub enum AnyJob {
    Pending(Job<Pending>),
    Running(Job<Running>),
    Completed(Job<Completed>),
    Failed(Job<Failed>),
    Dead(Job<Dead>),
}
```

**5. Over-engineering simple types.** `struct Email(String)` does not need
the full type-state treatment. Newtypes with validated constructors are simpler
and sufficient for most domain values. Reserve type-state for genuine protocol
or lifecycle state machines.

---

## Full Newtype Idiom: Display, From, Deref, repr(transparent)

A newtype wrapping `String` or `u64` is just the start. Here's the full pattern for a production newtype:

```rust
use std::fmt;

#[repr(transparent)]  // same memory layout as the inner type
pub struct JobId(u64);

// Create from the inner type
impl From<u64> for JobId {
    fn from(v: u64) -> Self { JobId(v) }
}

// Convert back to the inner type  
impl From<JobId> for u64 {
    fn from(id: JobId) -> u64 { id.0 }
}

// Human-readable display
impl fmt::Display for JobId {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "job:{}", self.0)
    }
}

// Debug
impl fmt::Debug for JobId {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "JobId({})", self.0)
    }
}

// Allow using job_id.0 without going through .0 everywhere:
impl std::ops::Deref for JobId {
    type Target = u64;
    fn deref(&self) -> &u64 { &self.0 }
}
```

`#[repr(transparent)]` guarantees the newtype has identical memory layout to the inner type. This matters if you're passing the type across FFI boundaries or storing it in shared memory — the ABI is identical.

## Multiple Orthogonal State Dimensions

Real systems often have two independent state dimensions. The trick is to use two type parameters, one for each dimension:

```rust
// State markers for auth
struct Unauthenticated;
struct Authenticated { user_id: u64 }

// State markers for mode  
struct ReadOnly;
struct ReadWrite;

struct Connection<Auth, Mode> {
    socket: TcpStream,
    _auth: PhantomData<Auth>,
    _mode: PhantomData<Mode>,
}

// Start unauthenticated, read-only
impl Connection<Unauthenticated, ReadOnly> {
    pub fn authenticate(self, token: &str) -> Result<Connection<Authenticated, ReadOnly>> {
        let user_id = verify_token(token)?;
        Ok(Connection {
            socket: self.socket,
            _auth: PhantomData,  // Authenticated
            _mode: PhantomData,  // still ReadOnly
        })
    }
}

// After auth, can upgrade to read-write
impl Connection<Authenticated, ReadOnly> {
    pub fn upgrade_to_write(self) -> Connection<Authenticated, ReadWrite> {
        Connection {
            socket: self.socket,
            _auth: PhantomData,
            _mode: PhantomData,  // now ReadWrite
        }
    }
    
    pub fn user_id(&self) -> u64 {
        // Can only access user_id in Authenticated state
        // ...
    }
}

// Only read-write connections can call mutating methods
impl Connection<Authenticated, ReadWrite> {
    pub fn write_record(&mut self, data: &[u8]) -> io::Result<()> { ... }
}
```

This is two type-state machines composed. A `Connection<Unauthenticated, ReadWrite>` is literally impossible to construct — the state machines enforce the ordering.

## When Runtime Enums Are Better

Type-state is not always the right choice. Use a runtime enum when:

- The states are numerous or determined by external data
- You need to store a collection of objects in different states (`Vec<Job>` where each Job might be in any state)
- The code that handles each state isn't significantly different

```rust
// Type-state: good when you have fixed states and state-specific methods
struct Job<S> { state: PhantomData<S>, ... }

// Runtime enum: good when you need a Vec of mixed-state jobs
enum JobState {
    Pending { priority: u8 },
    Running { worker_id: u64, started_at: Instant },
    Completed { result: JobResult, duration: Duration },
    Failed { error: String, retries: u32 },
}

struct Job {
    id: JobId,
    state: JobState,
}

// Runtime dispatch for storage and display
fn display_jobs(jobs: &[Job]) {
    for job in jobs {
        match &job.state {
            JobState::Pending { priority } => println!("pending (p{})", priority),
            JobState::Running { worker_id, .. } => println!("running on {}", worker_id),
            // ...
        }
    }
}
```

In taskforge, you'll use BOTH: the `Job<State>` type-state pattern for the lifecycle of an individual job being processed, and a runtime enum for the queue's view of all jobs (where you need to store them in a `HashMap` or database).
