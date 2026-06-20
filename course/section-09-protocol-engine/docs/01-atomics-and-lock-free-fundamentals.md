# Doc 01 — Atomics and Lock-Free Fundamentals

🟡 Every high-throughput server counts things: connections, messages, errors. When
that counting happens across threads, you need thread-safe mutation. There are two
correct tools: a `Mutex` (flexible, heavier) and an `Atomic` type (narrow, fast).
Understanding the difference — and when each is correct — starts at the hardware
level.

---

## Why Shared Mutation Is Dangerous Without Synchronization

Consider a simple counter incremented by two threads simultaneously:

```rust
// This looks fine. It is not.
static mut COUNTER: u64 = 0;

fn increment() {
    unsafe { COUNTER += 1; }  // Three machine instructions, not one
}
```

`COUNTER += 1` is not atomic. The CPU executes it as three separate steps:

1. **Load**: read COUNTER from memory into a register
2. **Add**: increment the register value
3. **Store**: write the register back to memory

If two threads both execute step 1 before either executes step 3, they both read
the same value. They both add 1 to it. They both store the result. The counter
advances by 1, not 2. This is a data race — a lost update.

**In C** this was the original problem `_Atomic` was introduced to solve:

```c
#include <stdatomic.h>

// Without atomic: data race on concurrent increment
int counter = 0;
counter++;  // load, add, store — races with other threads

// With atomic: hardware-guaranteed single operation
_Atomic int counter = 0;
atomic_fetch_add(&counter, 1);  // single CPU instruction: LOCK XADD
```

The word "atomic" means *indivisible*. An atomic operation completes in a single
step as seen by all other threads — there is no intermediate state where the load
has happened but the store has not.

---

## Why Mutexes Solve This But Have a Cost

The classic solution is a mutex. A mutex ensures only one thread executes the
critical section at a time:

```rust
use std::sync::Mutex;

let counter = Mutex::new(0u64);

// In the connection handler:
let mut count = counter.lock().unwrap();  // blocks if another thread holds the lock
*count += 1;
// Guard dropped here — lock released
```

This is correct. But a `Mutex` acquires an OS-level lock. On an uncontended x86
CPU, locking a mutex costs roughly 30–100 ns — not because the increment itself is
slow, but because of the kernel syscall, the cache coherency protocol, and the
memory barriers that surround the lock acquisition.

For a counter incremented on every connection accept — potentially thousands of
times per second — this overhead adds up. More importantly, **it serializes threads
unnecessarily**: thread 2 cannot increment the counter while thread 1 holds the
lock, even though "increment a counter" can be done as a single CPU instruction
without a lock at all.

---

## What Atomic Types Are: Hardware Instructions, Not OS Locks

Atomic types bypass the OS entirely. They compile directly to CPU instructions that
the hardware executes as a single, uninterruptible unit.

On x86-64:

| Rust operation | CPU instruction |
|---------------|-----------------|
| `fetch_add(1, ...)` | `LOCK XADD` — lock the cache line, add, return old value |
| `compare_exchange(old, new, ...)` | `LOCK CMPXCHG` — lock the cache line, compare, conditionally write |
| `store(val, ...)` | `MOV` with optional `MFENCE` depending on ordering |
| `load(...)` | `MOV` with optional `LFENCE` depending on ordering |

The `LOCK` prefix causes the CPU to assert exclusive ownership of the relevant
cache line for the duration of that instruction. No other CPU core can read or
write that cache line until the operation completes. This is why atomic operations
are thread-safe without an OS mutex — the synchronization happens in the CPU's
cache coherency hardware, not in kernel space.

This is also why atomics are limited to a single value. The CPU can lock one cache
line for one instruction. It cannot lock two cache lines atomically — that's what
mutexes are for.

---

## The Standard Atomic Types

The standard library provides atomic versions of the primitive integer types:

```rust
use std::sync::atomic::{AtomicBool, AtomicU32, AtomicU64, AtomicUsize, Ordering};
```

The operations on any atomic type:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

let counter = AtomicU64::new(0);

// Load: read the current value
let value = counter.load(Ordering::Relaxed);

// Store: write a new value
counter.store(42, Ordering::Relaxed);

// Fetch-and-add: atomically add, return the *previous* value
let old = counter.fetch_add(1, Ordering::Relaxed);
// old == value before the increment; counter now == old + 1

// Fetch-max: atomically set to max(current, new); returns the previous value
counter.fetch_max(1000, Ordering::Relaxed);

// Compare-and-swap: if current == expected, set to new; returns Result
let result = counter.compare_exchange(
    old,                  // expected current value
    old + 1,              // desired new value
    Ordering::AcqRel,     // ordering on success
    Ordering::Relaxed,    // ordering on failure
);
// Ok(old)         — swap happened: old value was `old`, now `old + 1`
// Err(actual)     — swap did not happen: actual current value returned
```

Every operation takes an `Ordering` argument. This is where Rust differs from C's
`_Atomic`, which defaults to sequential consistency. Rust makes you specify exactly
what you need — we'll explain why this matters next.

---

## Memory Ordering: Why the CPU Lies to You

This is the hardest part of atomics. Understand it once and it becomes mechanical.

**The problem**: CPUs and compilers reorder instructions for performance. The order
your instructions execute in may differ from the order you wrote them.

Here is a concrete example. Without ordering constraints:

```
Thread A (producer):               Thread B (consumer):

message = "hello"  // (1)          while !ready.load() {}  // wait
ready.store(true)  // (2)          println!("{}", message) // read message
```

This looks correct. It is not. The CPU might reorder (1) and (2) in Thread A's
pipeline: the store to `ready` might become visible to other cores before the
store to `message` does. Thread B observes `ready == true`, reads `message`, and
gets garbage — or the old message, or an empty string.

This isn't a bug in the code. It's the hardware working correctly. The CPU
optimizes for throughput; it does not guarantee that stores become visible to other
cores in the order you wrote them — unless you add explicit ordering constraints.

**The four orderings, explained as constraints:**

| Ordering | What you're telling the CPU |
|----------|----------------------------|
| `Relaxed` | No ordering constraint. Just make this operation atomic. Other reads/stores may be reordered freely. |
| `Acquire` | No loads or stores *after* this operation may be moved *before* it. Used on loads. |
| `Release` | No loads or stores *before* this operation may be moved *after* it. Used on stores. |
| `AcqRel` | Both Acquire and Release constraints. Used on read-modify-write operations. |
| `SeqCst` | A globally consistent total order of all SeqCst operations across all threads. Slowest. Rarely needed. |

**The standard pattern for "publish something and signal it's ready":**

```rust
// Thread A: producer
//   - write the data (non-atomic)
//   - then set ready with Release
//   Release: everything written BEFORE the store is visible to any Acquire load
message = "hello";
ready.store(true, Ordering::Release);  // barrier: message is visible before ready=true

// Thread B: consumer
//   - wait for ready with Acquire
//   - then read the data (non-atomic)
//   Acquire: everything written before the Release store is visible after this load
while !ready.load(Ordering::Acquire) {}
println!("{}", message);  // safe: guaranteed to see "hello"
```

The `Release`/`Acquire` pair creates a *happens-before* relationship: everything
Thread A did before the `Release` store is guaranteed to be visible to Thread B
after the `Acquire` load returns `true`.

**Decision rules for nexus:**

```rust
// Relaxed: I just need the number to be consistent.
// Nothing in memory depends on this value being seen in order.
let id = self.connection_counter.fetch_add(1, Ordering::Relaxed);

// AcqRel: I need the message payload to be visible before the sequence number.
// Any subscriber reading this seq number with Acquire will also see the payload.
let seq = self.message_counter.fetch_add(1, Ordering::AcqRel);

// Release / Acquire: shutdown signaling (see ShutdownFlag below)
// The work done before signaling must be visible to threads reading the signal.

// SeqCst: Almost never. Use only when you need a total global order across
// multiple independent atomics. If you think you need SeqCst, reconsider the design.
```

**C parallel:**

```c
// C's _Atomic with no explicit ordering defaults to sequential consistency
// This is like Rust's SeqCst — always correct, often slower than necessary
_Atomic uint64_t counter = 0;
atomic_fetch_add(&counter, 1);  // implicitly memory_order_seq_cst

// Explicit ordering in C matches Rust:
atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
atomic_store_explicit(&ready, true, memory_order_release);
bool r = atomic_load_explicit(&ready, memory_order_acquire);
```

Rust's explicit ordering is a feature: it forces you to think about what each
operation actually requires, preventing the common mistake of defaulting to SeqCst
everywhere (which is correct but slower than necessary on ARM and other weakly-
ordered architectures).

---

## nexus Metrics: Using Atomics for Counters

With the hardware model established, here is how nexus structures its metrics:

```rust
// nexus-core/src/metrics.rs
use std::sync::atomic::{AtomicU64, Ordering};

pub struct NexusMetrics {
    pub total_connections: AtomicU64,
    pub active_connections: AtomicU64,
    pub total_messages_routed: AtomicU64,
    pub total_bytes_sent: AtomicU64,
    pub total_errors: AtomicU64,
}

impl NexusMetrics {
    // const fn: this struct can be a static, initialized at compile time (no heap)
    pub const fn new() -> Self {
        Self {
            total_connections: AtomicU64::new(0),
            active_connections: AtomicU64::new(0),
            total_messages_routed: AtomicU64::new(0),
            total_bytes_sent: AtomicU64::new(0),
            total_errors: AtomicU64::new(0),
        }
    }

    pub fn snapshot(&self) -> MetricsSnapshot {
        // Each load is independent. The snapshot may be slightly inconsistent:
        // active_connections might have changed between the first and last load.
        // For monitoring dashboards, this is acceptable.
        // For transactional consistency, wrap in Mutex<MetricsSnapshot> instead.
        MetricsSnapshot {
            total_connections: self.total_connections.load(Ordering::Relaxed),
            active_connections: self.active_connections.load(Ordering::Relaxed),
            total_messages_routed: self.total_messages_routed.load(Ordering::Relaxed),
            total_bytes_sent: self.total_bytes_sent.load(Ordering::Relaxed),
            total_errors: self.total_errors.load(Ordering::Relaxed),
        }
    }
}

#[derive(Debug, Clone)]
pub struct MetricsSnapshot {
    pub total_connections: u64,
    pub active_connections: u64,
    pub total_messages_routed: u64,
    pub total_bytes_sent: u64,
    pub total_errors: u64,
}

// `const fn new()` enables a zero-cost static — no lazy_static, no Once, no heap
pub static METRICS: NexusMetrics = NexusMetrics::new();
```

The `NexusEngine` uses these in connection handling:

```rust
// nexus-core/src/engine.rs
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;

pub struct NexusEngine {
    metrics: Arc<NexusMetrics>,
    // message_counter shared with topic structs via Arc
    message_counter: Arc<AtomicU64>,
}

impl NexusEngine {
    /// Returns a unique, monotonically increasing ID for this connection.
    pub fn allocate_connection_id(&self) -> u64 {
        // Relaxed: uniqueness is all we need. Nothing in memory is gated on this ID.
        self.metrics.total_connections.fetch_add(1, Ordering::Relaxed)
    }

    pub fn connection_opened(&self) {
        self.metrics.active_connections.fetch_add(1, Ordering::Relaxed);
    }

    pub fn connection_closed(&self) {
        self.metrics.active_connections.fetch_sub(1, Ordering::Relaxed);
    }

    /// Returns a globally unique, monotonically increasing sequence number.
    pub fn next_message_seq(&self) -> u64 {
        // AcqRel: callers writing the message payload before this call can be
        // sure the payload is visible to any reader who sees this sequence number
        // via an Acquire load.
        self.message_counter.fetch_add(1, Ordering::AcqRel)
    }
}
```

---

## AtomicBool for Shutdown Signaling

nexus uses `AtomicBool` to signal shutdown to OS-thread workers. The
`Acquire`/`Release` pairing here is not optional:

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

pub struct ShutdownFlag(Arc<AtomicBool>);

impl ShutdownFlag {
    pub fn new() -> (Self, Self) {
        let flag = Arc::new(AtomicBool::new(false));
        (ShutdownFlag(Arc::clone(&flag)), ShutdownFlag(flag))
    }

    pub fn signal(&self) {
        // Release: all stores executed before this call are guaranteed to be
        // visible to any thread that loads this flag with Acquire.
        self.0.store(true, Ordering::Release);
    }

    pub fn is_set(&self) -> bool {
        // Acquire: we see all stores that happened before the Release store.
        self.0.load(Ordering::Acquire)
    }
}
```

The concrete guarantee this provides:

```
Thread A (shutdown initiator):          Thread B (worker):

write_final_snapshot(&snapshot);  // (1)   loop {
flag.signal();  // Release         // (2)       if flag.is_set() { // Acquire (3)
                                               read_snapshot(&snapshot); // (4)
                                           }
                                       }
```

Because (2) uses `Release` and (3) uses `Acquire`, Thread B's read in (4) is
guaranteed to see the write Thread A did in (1). Without this pairing, the CPU
could reorder (4) before (3) and Thread B would read stale snapshot data.

For nexus's async tasks, Tokio's `watch` channel is cleaner than `AtomicBool` —
it integrates with the async runtime's notification mechanism. `AtomicBool` is
correct for OS threads and for cases where you need the absolute minimum overhead.

---

## How Compare-and-Swap Works

Before showing the rate limiter, understand what CAS is at the hardware level.

`compare_exchange` (CMPXCHG on x86) is a single CPU instruction that does this
atomically:

```
if *address == expected {
    *address = new_value
    return Ok(expected)   // swap happened
} else {
    return Err(*address)  // swap did not happen; returns actual value
}
```

The entire read-compare-write sequence happens under the CPU's cache lock, in one
instruction. No other thread can observe the intermediate state.

This enables **lock-free algorithms**: instead of "lock a mutex, do the work,
unlock", you "read the current state, compute the new state, CAS — retry if
someone changed it first." No OS involvement, no thread blocking.

The retry loop is fundamental:

```rust
loop {
    let current = atomic.load(Ordering::Relaxed);
    let new = compute_next(current);  // your logic

    match atomic.compare_exchange_weak(current, new, Ordering::AcqRel, Ordering::Relaxed) {
        Ok(_) => break,        // CAS succeeded — we updated the value
        Err(_) => continue,    // another thread changed it; reload and retry
    }
}
```

`compare_exchange_weak` may fail spuriously on ARM (the weak variant allows false
failures). This is intentional: the `weak` version maps to `LDREX`/`STREX` on ARM
which are faster than the strong version. The retry loop handles both genuine
conflicts and spurious failures identically.

---

## Rate Limiting Without a Mutex: The Packed CAS Pattern

nexus rate-limits PUBLISH commands: a client may send at most N publishes per
second. The challenge is that both the current window (which second) and the count
within that window must be updated together, atomically.

If we used two separate atomics:

```rust
// WRONG — not atomically consistent
struct BrokenRateLimiter {
    window_start: AtomicU32,  // which second are we in?
    count: AtomicU32,         // how many this second?
}
```

A thread could observe `window_start` reset to the new second but `count` not yet
reset — the two updates cannot be made atomic independently. Another thread might
incorrectly believe the rate limit has been exceeded in the new window.

The fix: **pack both values into a single `u64`** so a single CAS can update both
atomically. The high 32 bits store the window timestamp; the low 32 bits store the
count. One CAS updates both fields in a single indivisible operation.

```rust
// Bit layout of the packed state:
//   bits 63-32: window_start (Unix timestamp in seconds, cast to u32)
//   bits 31-0:  count within the current window
//
// Packing: state = ((window as u64) << 32) | (count as u64)
// Unpacking:
//   window = (state >> 32) as u32
//   count  = (state & 0xFFFF_FFFF) as u32
```

With that understanding, the implementation:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::{SystemTime, UNIX_EPOCH};

pub struct RateLimiter {
    // Packed: [window_start: u32 | count: u32]
    // Single CAS updates both fields atomically
    state: AtomicU64,
    max_per_second: u32,
}

impl RateLimiter {
    pub fn new(max_per_second: u32) -> Self {
        RateLimiter {
            state: AtomicU64::new(0),
            max_per_second,
        }
    }

    fn current_second() -> u32 {
        SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs() as u32
    }

    /// Returns true and increments count if under the rate limit.
    /// Returns false without incrementing if the limit has been reached.
    pub fn check_and_increment(&self) -> bool {
        let now = Self::current_second();

        loop {
            let current = self.state.load(Ordering::Relaxed);
            let window = (current >> 32) as u32;
            let count = (current & 0xFFFF_FFFF) as u32;

            let (new_window, new_count) = if window == now {
                // Same second — check limit before incrementing
                if count >= self.max_per_second {
                    return false;  // rate limited
                }
                (window, count + 1)
            } else {
                // New second — reset counter, start at 1
                (now, 1)
            };

            let new_state = ((new_window as u64) << 32) | (new_count as u64);

            // Try to atomically update both fields.
            // If another thread changed state between our load and this CAS,
            // compare_exchange_weak returns Err with the new actual value.
            // We loop and retry with the fresh value.
            match self.state.compare_exchange_weak(
                current,
                new_state,
                Ordering::AcqRel,  // success: publish the change with ordering
                Ordering::Relaxed, // failure: we'll reload and retry
            ) {
                Ok(_) => return true,
                Err(_) => continue,
            }
        }
    }
}
```

**Why `compare_exchange_weak` instead of `compare_exchange`?**

On ARM, the hardware provides `LDREX`/`STREX` which implement a "load-linked /
store-conditional" pattern. STREX can fail spuriously (the cache line was observed
by another core, even if not written). `compare_exchange_weak` maps directly to
this pattern — slightly faster, with spurious failures handled by the retry loop.
`compare_exchange` (strong) emits extra code on ARM to retry until the failure is
genuine. Since we have a loop anyway, `weak` is always the right choice for CAS
loops.

---

## When to Use Atomics — and When Not To

Atomics are a sharp, narrow tool. The decision rules:

**Use atomics when:**
- Updating a single independent counter (`fetch_add` with `Relaxed`)
- Signaling a boolean state change between threads (`AtomicBool` with `Release`/`Acquire`)
- Implementing a lock-free single-value CAS algorithm
- Performance is critical and the update fits in one integer

**Use a Mutex when:**
- Multiple values must update together (connection count + bytes transferred)
- The update logic is more complex than read-modify-write
- You need to hold a lock across multiple operations (e.g., check-then-act)
- Correctness is more important than microseconds

```rust
// ❌ Two atomics cannot update atomically together
struct BadStats {
    connection_count: AtomicU64,
    bytes_transferred: AtomicU64,
}
// Incrementing both has a window where one is updated and the other isn't.
// A reader observes an inconsistent pair.

// ✅ Mutex when multiple fields must be consistent with each other
struct GoodStats {
    inner: Mutex<StatsInner>,
}

struct StatsInner {
    connection_count: u64,
    bytes_transferred: u64,
}
// Both fields update under a single lock — readers always see a consistent pair.
```

**The key rule**: Atomics operate on *one word*. The packing trick extends this to
*two logical values packed into one word*. Beyond that, reach for a `Mutex`.

---

## Send and Sync: Why Atomics Need No Extra Wrapper

Unlike `Cell<T>` or `RefCell<T>`, atomic types implement `Send + Sync` by
definition — they are designed for multi-threaded access. You need `Arc` to share
the *allocation* across threads, but not to make the access thread-safe:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering};

let counter = Arc::new(AtomicU64::new(0));

for _ in 0..8 {
    let counter = Arc::clone(&counter);
    std::thread::spawn(move || {
        counter.fetch_add(1, Ordering::Relaxed);
    });
}
```

The `Arc` provides shared ownership of the heap allocation. The `AtomicU64`
provides thread-safe mutation of the value. No `Mutex` wrapping needed — that is
the whole point.

**C parallel:**

```c
// C: _Atomic global, no wrapper needed
static _Atomic uint64_t g_counter = 0;
atomic_fetch_add(&g_counter, 1);  // safe from any thread
```

In Rust, prefer `Arc<AtomicU64>` over a `static AtomicU64` in library code.
Global state complicates testing: you cannot reset a static between tests unless
you write reset logic. Wrapping in `Arc` makes the counter testable — each test
creates its own `Arc` and there is no shared global state to clean up.

---

## Exercises

**Exercise 1 — Connection Counter Under Concurrency**

Implement `NexusMetrics::new()` with five `AtomicU64` fields. Add a `snapshot()`
method that returns a `MetricsSnapshot`. Write a test that spawns 100 `tokio::task`
tasks, each calling `metrics.total_connections.fetch_add(1, Ordering::Relaxed)`,
then awaits all tasks and asserts `snapshot().total_connections == 100`. The test
verifies that 100 concurrent increments produce exactly 100, not fewer.

**Exercise 2 — Rate Limiter**

Implement the `RateLimiter` from this doc. Write a unit test that:
1. Creates a `RateLimiter` with `max_per_second = 3`
2. Calls `check_and_increment()` four times in rapid succession
3. Asserts the fourth call returns `false` (rate limited)
4. Waits until the next second using `std::thread::sleep(Duration::from_secs(1))`
5. Asserts the next call returns `true` (new window, counter reset)

Bonus: run the test with `cargo +nightly miri test` to verify no undefined
behavior from the bit-packing operations.

**Exercise 3 — Benchmark: Mutex vs AtomicU64**

Write a Criterion benchmark in `benches/atomic_bench.rs` comparing:
- Incrementing a `Mutex<u64>` from 8 concurrent threads, 100,000 times each
- Incrementing an `AtomicU64` (Relaxed) from 8 concurrent threads, 100,000 times each

Use `std::sync::Arc` to share the value across threads. Measure throughput
(increments/second). The atomic version should be 3–10x faster on x86-64.
Annotate the benchmark with the measured ratio and explain it in a comment: why
is the atomic version faster even though both are correct?

---

## Checklist

- [ ] All monotonically increasing counters use `fetch_add` with `Relaxed` ordering
- [ ] Sequence numbers that gate access to data use `AcqRel`
- [ ] Shutdown flags use `Release` (set) and `Acquire` (read) — not `Relaxed`
- [ ] Complex multi-field updates use a `Mutex`, not multiple atomics
- [ ] CAS loops use `compare_exchange_weak` (handles spurious failure on ARM correctly)
- [ ] Packed CAS values have a comment explaining the bit layout before the code
- [ ] `const fn new()` used on metrics structs to allow static initialization
- [ ] `Arc<AtomicT>` preferred over `static AtomicT` in testable code
