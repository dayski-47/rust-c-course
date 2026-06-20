# Doc 04 — PhantomData and Zero-Cost Proofs

🔴 This is Rust's most misunderstood feature. By the end of this doc you'll know exactly what it does, why it exists, and how to use it to make bugs impossible at compile time.

Every C developer has seen this class of bug: two things that look identical in the type system but aren't interchangeable at runtime. A 32-bit register and a 16-bit register are both "an integer." A read DMA buffer and a write DMA buffer are both "a pointer." An open file descriptor and a closed one are both "a number." The C type system has no way to distinguish them.

Rust's `PhantomData<T>` gives you a way to attach type-level information to a struct without storing any data. The phantom type parameter exists only during compilation — it compiles to zero bytes — but it makes the compiler enforce distinctions that would otherwise only be caught at runtime (or not at all).

---

## What `PhantomData` Is

`PhantomData<T>` is a zero-sized type defined in `std::marker`. When you put it in a struct, the struct becomes *parameterized* by `T` even though `T` doesn't appear in any actual field:

```rust
use std::marker::PhantomData;

struct JobHandle<S> {
    id: u64,
    _state: PhantomData<S>,  // zero bytes — exists only in the type system
}
```

At runtime, `JobHandle<Pending>` and `JobHandle<Running>` are identical — both are a single `u64`. At compile time they are different types and the compiler won't let you mix them.

**Why does the compiler require `PhantomData` at all?**

Rust's variance rules require that every type parameter appear in at least one field. If you write `struct Foo<T>` with no field that mentions `T`, the compiler rejects it — it can't determine variance or drop behavior for `T`. `PhantomData<T>` declares "this struct is logically related to `T`" without actually storing one.

---

## The Three Jobs of `PhantomData`

| Job | Spelling | Effect |
|-----|----------|--------|
| Lifetime binding | `PhantomData<&'a T>` | Struct is treated as borrowing from `'a` |
| Ownership simulation | `PhantomData<T>` | Drop check assumes struct owns a `T` |
| Variance control | `PhantomData<fn(T)>` | Makes struct contravariant over `T` |

For taskforge, the first two jobs are the most important. The variance table is worth knowing but rarely needs deliberate control — the common phantom patterns pick the right variance automatically.

---

## Job-State Branding for taskforge

The taskforge worker needs to track a `Job`'s lifecycle without letting callers reach into an intermediate state:

```
Pending → Running → Completed
                 ↘ Failed
```

The type-state pattern uses PhantomData to encode which state the handle refers to:

```rust
use std::marker::PhantomData;
use uuid::Uuid;

// State marker types — zero-sized, live only in the type system
pub struct Pending;
pub struct Running;
pub struct Completed;
pub struct Failed;

/// A typed handle to a job in a specific lifecycle state.
/// `PhantomData<S>` costs zero bytes but locks the state into the type.
pub struct JobHandle<S> {
    pub id: Uuid,
    _state: PhantomData<S>,
}

impl JobHandle<Pending> {
    pub fn new(id: Uuid) -> Self {
        JobHandle { id, _state: PhantomData }
    }

    /// Transition from Pending → Running.
    /// Consumes the Pending handle — you can't run a job twice.
    pub fn start(self) -> JobHandle<Running> {
        JobHandle { id: self.id, _state: PhantomData }
    }
}

impl JobHandle<Running> {
    /// Transition from Running → Completed.
    pub fn complete(self) -> JobHandle<Completed> {
        JobHandle { id: self.id, _state: PhantomData }
    }

    /// Transition from Running → Failed.
    pub fn fail(self) -> JobHandle<Failed> {
        JobHandle { id: self.id, _state: PhantomData }
    }
}

// Only Running jobs can be retried — Completed and Failed cannot
impl JobHandle<Failed> {
    pub fn retry(self) -> JobHandle<Pending> {
        JobHandle { id: self.id, _state: PhantomData }
    }
}
```

The caller's code can only follow the valid transitions:

```rust
fn process_job(handle: JobHandle<Pending>) {
    let running = handle.start();
    
    match execute_job_logic(running.id) {
        Ok(_) => {
            let _done = running.complete();
            // job is now Completed
        }
        Err(_) => {
            let failed = running.fail();
            let _retrying = failed.retry();
            // job is now Pending again
        }
    }
    
    // These would not compile:
    // handle.complete();  ← handle was moved into start()
    // running.complete(); ← running was moved into complete() or fail()
}
```

In C, this lifecycle would be a `job_status_t` enum checked with `if` statements. Every caller who forgets to check the status — or checks the wrong field — introduces a bug. The type-state version makes the wrong path a compile error.

---

## Register Width Branding

taskforge-worker communicates with Redis via raw byte manipulation in some paths. The same phantom technique prevents reading a 32-bit Redis counter as if it were 16-bit:

```rust
use std::marker::PhantomData;

// Width markers — zero-sized
pub struct U16;
pub struct U32;
pub struct U64;

/// A buffer whose element width is tracked in the type.
/// PhantomData<W> costs zero bytes.
pub struct RedisIntBuffer<W> {
    bytes: Vec<u8>,
    _width: PhantomData<W>,
}

impl RedisIntBuffer<U64> {
    pub fn from_redis_reply(bytes: Vec<u8>) -> Option<Self> {
        if bytes.len() % 8 != 0 {
            return None;
        }
        Some(RedisIntBuffer { bytes, _width: PhantomData })
    }

    pub fn read(&self, index: usize) -> u64 {
        let offset = index * 8;
        u64::from_le_bytes(self.bytes[offset..offset + 8].try_into().unwrap())
    }
}

impl RedisIntBuffer<U16> {
    pub fn read(&self, index: usize) -> u16 {
        let offset = index * 2;
        u16::from_le_bytes(self.bytes[offset..offset + 2].try_into().unwrap())
    }
}

// Elsewhere in the code:
// fn read_queue_depth(buf: &RedisIntBuffer<U64>) -> u64 { buf.read(0) }
// fn read_slot_mask(buf: &RedisIntBuffer<U16>) -> u16 { buf.read(0) }
//
// read_queue_depth(&u16_buf)  ← compile error: wrong buffer type
```

No runtime check. No assert. The right type forces the right read.

---

## Lifetime Branding: Preventing Handle Escape

Lifetime branding is the most subtle use of `PhantomData`. It brands a handle with an invariant lifetime so handles from one context cannot be used in another:

```rust
use std::marker::PhantomData;

/// A scoped connection handle branded to a specific pool checkout.
/// Invariant over 'pool — handles from one checkout can't be used in another.
pub struct PooledConn<'pool> {
    raw: redis::Connection,
    _brand: PhantomData<*mut &'pool ()>,  // *mut → invariant (not covariant)
}

/// Check out a connection; the handle cannot outlive the pool.
pub fn with_connection<R>(
    pool: &redis::Pool,
    f: impl for<'pool> FnOnce(PooledConn<'pool>) -> R,
) -> R {
    let raw = pool.checkout().expect("pool exhausted");
    let conn = PooledConn { raw, _brand: PhantomData };
    f(conn)
    // conn returned to pool here via Drop
}

// This would be a compile error — handle can't escape the closure:
// let stolen: PooledConn<'_>;
// with_connection(&pool, |conn| { stolen = conn; });
// use_elsewhere(stolen);
```

The `*mut` in `PhantomData<*mut &'pool ()>` makes the struct *invariant* over `'pool`. Invariance means "the lifetime is exact" — the compiler won't let you store a `PooledConn<'short>` where a `PooledConn<'long>` is expected, which prevents the handle from escaping its lexical scope.

---

## Unit-of-Measure Pattern

taskforge tracks job timing in multiple units (milliseconds for user display, microseconds internally, seconds for configuration). PhantomData prevents mixing them:

```rust
use std::marker::PhantomData;
use std::ops::{Add, Sub};

// Unit markers
pub struct Milliseconds;
pub struct Microseconds;
pub struct Seconds;

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Duration<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl<U> Duration<U> {
    pub fn value(self) -> f64 { self.value }
}

impl Duration<Milliseconds> {
    pub const fn ms(value: f64) -> Self {
        Duration { value, _unit: PhantomData }
    }
    pub fn to_micros(self) -> Duration<Microseconds> {
        Duration { value: self.value * 1000.0, _unit: PhantomData }
    }
}

impl Duration<Microseconds> {
    pub const fn us(value: f64) -> Self {
        Duration { value, _unit: PhantomData }
    }
    pub fn to_millis(self) -> Duration<Milliseconds> {
        Duration { value: self.value / 1000.0, _unit: PhantomData }
    }
}

impl<U> Add for Duration<U> {
    type Output = Duration<U>;
    fn add(self, rhs: Duration<U>) -> Duration<U> {
        Duration { value: self.value + rhs.value, _unit: PhantomData }
    }
}

// Usage in taskforge:
fn record_job_timing(execution: Duration<Microseconds>, queue_wait: Duration<Milliseconds>) {
    let total_ms = execution.to_millis() + queue_wait;
    // execution + queue_wait would be a compile error — different units
}
```

The add `impl` only works within a unit — you can't add `Duration<Milliseconds>` and `Duration<Microseconds>` directly. The compiler enforces dimensional consistency.

---

## The `Send` and `Sync` Implications

`PhantomData` affects whether your type implements `Send` and `Sync` automatically:

```rust
// PhantomData<T>:
// - struct is Send if T: Send
// - struct is Sync if T: Sync
// This is correct for "I logically own a T"

struct JobBuffer<T> {
    ptr: *mut T,        // *mut T is !Send, !Sync
    _owner: PhantomData<T>,  // restores Send/Sync based on T's own bounds
}

// Without PhantomData<T>, JobBuffer would always be !Send due to *mut T.
// With PhantomData<T>, JobBuffer<T> is Send when T: Send — which is correct.
// If T were a non-Send type (like Rc), JobBuffer<T> would correctly be !Send.
```

This matters for taskforge's concurrent queue processing: worker threads share handles across thread boundaries. `PhantomData<T>` ensures the compiler verifies thread safety correctly.

---

## C Comparison: What You Replace

| C pattern | C problem | Rust equivalent |
|-----------|-----------|-----------------|
| `typedef uint32_t job_id_t` | `job_id_t x = worker_id;` compiles | `struct JobId(u32)` — newtype |
| `int status;` with `STATUS_PENDING/RUNNING` enum | Forgot to check status | `JobHandle<Pending>` vs `JobHandle<Running>` |
| `void *buf` for different widths | Wrong cast, wrong size | `Buffer<U32>` vs `Buffer<U16>` |
| `int fd;` can be open or closed | Used after close | `OpenFd` vs `ClosedFd` type-state |

All of these C patterns require runtime checks or trust in convention. The Rust phantom type versions are compile-time — they cannot be violated.

---

## When to Use `PhantomData`

Use it when:

- A struct has a type parameter only for compile-time tracking (type-state, unit-of-measure)
- You're wrapping a raw pointer and need to express ownership or lifetime bounds correctly
- You want to brand values with a "session" or "scope" lifetime to prevent them escaping
- You need to control `Send`/`Sync` behavior on a wrapper around raw types

Do NOT use it to:

- Make types more complex without a specific safety goal
- Paper over a design where the state should actually be a runtime enum

The taskforge project uses PhantomData in two places: the `JobHandle<S>` lifecycle type-state (which prevents running a job twice or completing a job that hasn't started), and the `Duration<Unit>` measure type (which prevents queue wait times and execution times from being mixed in arithmetic).

---

## Exercises

**Exercise 1 — JobHandle Type-State**

Implement `JobHandle<S>` with states `Pending`, `Running`, `Completed`. Add:
- `fn start(self: JobHandle<Pending>) -> JobHandle<Running>` — transition from pending to running
- `fn complete(self: JobHandle<Running>, result: String) -> JobHandle<Completed>` — transition from running to completed
- `fn result(self: &JobHandle<Completed>) -> &str` — only available on completed handles

Write a unit test that chains these three calls. Then write a second test that tries to
call `complete()` on a `JobHandle<Pending>` — it should not compile. Use `trybuild` to
verify the compile error.

**Exercise 2 — Duration<Unit> Arithmetic**

Implement `Duration<Milliseconds>` and `Duration<Microseconds>` as newtype wrappers
around `u64`. Add `impl Add` for `Duration<T> + Duration<T>` (same unit) but not
`Duration<Milliseconds> + Duration<Microseconds>` (different units).

Write a test that adds two `Duration<Milliseconds>` values and asserts the result.
Then try to add a `Duration<Milliseconds>` and a `Duration<Microseconds>` — it should
not compile. Verify with `trybuild`.

**Exercise 3 — Verify Zero Cost**

Use `cargo-show-asm` to inspect the generated assembly for a function that creates and
transitions a `JobHandle<Pending>` to `JobHandle<Running>`:

```bash
cargo install cargo-show-asm
cargo show-asm 'taskforge_core::job::JobHandle::start'
```

Confirm that the function compiles to the same assembly as a function that just moves
a `u64`. The PhantomData should produce zero overhead.

---

## Variance: Why PhantomData's Type Parameter Matters

Variance determines whether a generic type can be substituted with a "sub-type." In Rust, "sub-type" means "has a longer lifetime" — a `&'long str` is a sub-type of `&'short str` because it's more specific (lives longer, so usable anywhere a shorter-lived reference is needed).

**The three variances:**

| Variance | Meaning | Example |
|----------|---------|---------|
| **Covariant** | `'long` can substitute for `'short` | `&'a T`, `Box<T>`, `Vec<T>` |
| **Contravariant** | Reverses: `'short` can substitute for `'long` | `fn(T)` in parameter position |
| **Invariant** | No substitution in either direction | `&'a mut T`, `Cell<T>` |

**PhantomData controls which variance your struct gets:**

| PhantomData type | Variance over T | Use when |
|-----------------|----------------|----------|
| `PhantomData<T>` | Covariant | You logically own a T |
| `PhantomData<&'a T>` | Covariant over 'a, covariant over T | You borrow T with lifetime 'a |
| `PhantomData<&'a mut T>` | Covariant over 'a, **invariant over T** | You mutably borrow T |
| `PhantomData<*mut T>` | **Invariant** | Non-owning mutable pointer |
| `PhantomData<fn(T)>` | **Contravariant** | T appears in argument position |

**Why invariance matters — a concrete example:**

```rust
use std::marker::PhantomData;

// If JobHandle<S> were covariant over S, you could substitute
// JobHandle<Running> where JobHandle<Pending> is expected.
// That breaks the state machine — you'd call methods on the wrong state.

// Make it INVARIANT over S to prevent this:
struct JobHandle<S> {
    id: u64,
    _state: PhantomData<fn(S) -> S>,  // invariant: fn(S) -> S cancels out
}

// Compare to covariant (wrong for state machines):
// _state: PhantomData<S>,  // covariant: JobHandle<Running> could flow into JobHandle<Pending>
```

For the taskforge type-state machine, you almost always want `PhantomData<fn(S) -> S>` (invariant) rather than `PhantomData<S>` (covariant), because you don't want state substitution.

## Send + Sync Derivation with PhantomData

When you have raw pointers or PhantomData in a struct, Rust's automatic `Send`/`Sync` derivation stops working — the compiler can't prove the type is safe to send across threads. You have to assert it manually with `unsafe impl`:

```rust
use std::marker::PhantomData;

struct JobQueue<J> {
    ptr: *mut J,
    len: usize,
    _marker: PhantomData<J>,
}

// Raw pointer makes JobQueue<J> not Send or Sync by default.
// If we know J is Send (which all job types should be),
// we can assert the queue is also Send:

// SAFETY: JobQueue<J> owns its data exclusively. If J is Send,
// the queue is safe to transfer to another thread.
unsafe impl<J: Send> Send for JobQueue<J> {}

// SAFETY: We never allow concurrent mutation — &JobQueue<J>
// only provides read access, so sharing a reference is safe.
unsafe impl<J: Send> Sync for JobQueue<J> {}
```

The `unsafe impl` is your contract to the compiler: "I have verified this is safe, please trust me." The comment is required — it explains WHY it's safe so the next person reading the code knows what invariant you're maintaining. If the code ever changes in a way that breaks that invariant, the comment tells them to revisit the `unsafe impl`.

**Rule of thumb:**
- Types made of only `Send + Sync` types → automatically `Send + Sync`
- Types with raw pointers → manually assert `Send` and/or `Sync` with `unsafe impl`
- Types with `Rc<T>` or `Cell<T>` → NOT `Send`, not even with `unsafe impl` (they're genuinely not thread-safe)
