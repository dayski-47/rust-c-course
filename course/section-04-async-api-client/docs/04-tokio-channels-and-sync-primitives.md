# 04 — Tokio Channels and Sync Primitives 🟡

In section 3 you used `std::sync::mpsc` channels to communicate between threads. Tokio has its own channel implementations that work the same way conceptually, but with a critical difference: they're async-aware. They won't block OS threads when waiting for messages.

This doc covers Tokio's four channel types, async mutexes, and semaphores. By the end, you'll know which synchronization tool to reach for in any situation.

---

## Why Tokio Has Its Own Channels

The channels in `std::sync::mpsc` are perfectly good for threaded code. But in async code, `recv()` is a *blocking* call — it puts the OS thread to sleep until a message arrives. In an async context, that blocks the entire executor thread, starving all other tasks.

Tokio's channels solve this by making `recv()` an async function. Instead of blocking, it yields to the executor when no messages are available, allowing other tasks to run:

```rust
// std channels: recv() BLOCKS the thread
use std::sync::mpsc;
let (tx, rx) = mpsc::channel();
let msg = rx.recv().unwrap();  // Thread is frozen here until message arrives

// Tokio channels: recv() is async, yields to executor
use tokio::sync::mpsc;
let (tx, mut rx) = mpsc::channel(100);
let msg = rx.recv().await.unwrap();  // Other tasks can run while waiting
```

---

## The Four Channel Types

Tokio provides four channel types covering different communication patterns:

```
CHANNEL SELECTION GUIDE
=======================

Use case                              Channel
----------------------------------------------
Work queue, many producers → 1 consumer    mpsc
Request/response, one-time value          oneshot
Broadcast event to all subscribers        broadcast
Share latest config/state change          watch
```

### mpsc — Multi-Producer, Single Consumer

The workhorse. One receiver, many senders. Messages are queued and delivered in order.

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // Buffer size: how many messages can be queued before senders block.
    // Bounded channels provide backpressure — if the receiver falls behind,
    // senders automatically slow down.
    let (tx, mut rx) = mpsc::channel::<String>(100);

    // You can clone the sender to create multiple producers.
    let tx2 = tx.clone();

    tokio::spawn(async move {
        tx.send("message from task 1".to_string()).await.unwrap();
    });

    tokio::spawn(async move {
        tx2.send("message from task 2".to_string()).await.unwrap();
    });

    // Receive until the channel closes (all senders dropped).
    while let Some(msg) = rx.recv().await {
        println!("Received: {}", msg);
    }
}
```

This is the right channel for `ghanalyze`: worker tasks (fetching repo details) send results back to the main task for collection.

### oneshot — Single Value, One Time

A channel that carries exactly one message, from one sender to one receiver. Perfect for request/response patterns.

```rust
use tokio::sync::oneshot;

async fn request_data_from_worker(
    work_tx: tokio::sync::mpsc::Sender<WorkRequest>,
) -> String {
    // Create a response channel for this specific request.
    let (response_tx, response_rx) = oneshot::channel();

    // Send the work request along with the response channel.
    work_tx.send(WorkRequest { response: response_tx }).await.unwrap();

    // Wait for the worker's response.
    response_rx.await.unwrap()
}

struct WorkRequest {
    response: oneshot::Sender<String>,
}
```

The sender doesn't need `.await` — `oneshot::Sender::send()` is a regular function that either succeeds or fails instantly (if the receiver was dropped).

### broadcast — One Publisher, Many Subscribers

Every subscriber receives every message. Messages are *cloned* for each subscriber. If a slow subscriber falls too far behind, it misses messages (the buffer wraps around).

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    // Buffer size: how many messages to keep in the ring buffer.
    let (tx, _) = broadcast::channel::<String>(16);

    // Subscribe to create a new receiver. Each subscriber gets ALL messages.
    let mut rx1 = tx.subscribe();
    let mut rx2 = tx.subscribe();

    tx.send("shutdown signal".to_string()).unwrap();

    let msg1 = rx1.recv().await.unwrap();
    let msg2 = rx2.recv().await.unwrap();

    assert_eq!(msg1, "shutdown signal");
    assert_eq!(msg2, "shutdown signal");  // Both got the same message
}
```

Use broadcast for: shutdown signals, configuration change notifications, log fanout, or any situation where you need "everyone gets the message."

### watch — Latest Value, Multiple Readers

Like broadcast, but only the *latest* value matters. Readers don't miss messages — they always read the current value, even if they're slow. Older values are overwritten.

```rust
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (tx, rx) = watch::channel(0u64);  // Initial value: 0

    tokio::spawn(async move {
        for i in 1..=5 {
            tokio::time::sleep(std::time::Duration::from_millis(10)).await;
            tx.send(i).unwrap();
        }
    });

    let mut rx2 = rx.clone();  // Multiple readers share the same watch channel
    loop {
        // Wait until the value changes.
        rx2.changed().await.unwrap();
        println!("New value: {}", *rx2.borrow());
    }
}
```

Use watch for: rate limit state (is the API throttled?), configuration hot-reload, connection status, or any "current state" that multiple consumers need to observe.

---

## tokio::sync::Mutex — The Async Mutex

You already know `std::sync::Mutex`. Tokio's version works the same way, with one difference: `.lock()` is async. If the mutex is already locked, instead of blocking the OS thread, it yields to the executor.

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0u64));

    let mut handles = vec![];
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = tokio::spawn(async move {
            let mut guard = counter.lock().await;  // Async lock — doesn't block thread
            *guard += 1;
        }); // Guard dropped here, lock released
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }

    println!("Final count: {}", *counter.lock().await);
}
```

### The Critical Pitfall: std::sync::Mutex Across .await 🔴

This is one of the most dangerous mistakes in async Rust. It compiles, it might work sometimes, and it causes subtle deadlocks in production:

```rust
use std::sync::Mutex;  // NOT tokio — this is the std Mutex

async fn dangerous(data: &Mutex<Vec<String>>) {
    let mut guard = data.lock().unwrap();    // Lock acquired
    guard.push("item".to_string());

    slow_network_call().await;               // ← PROBLEM HERE

    guard.push("another".to_string());
}   // Guard dropped here — lock released
```

What goes wrong: while `slow_network_call()` is awaited, the executor might schedule another task on the same OS thread. If that other task also tries to lock `data`, it will try to acquire the std Mutex from the *same thread that already holds it*. On a single-threaded runtime, this is an instant deadlock. On multi-threaded, it's a performance trap that blocks a worker thread.

The fix:

```rust
use tokio::sync::Mutex;  // Tokio's async Mutex

async fn safe(data: &Mutex<Vec<String>>) {
    let mut guard = data.lock().await;       // Async lock

    guard.push("item".to_string());
    slow_network_call().await;               // OK: tokio Mutex yields correctly
    guard.push("another".to_string());
}   // Guard dropped, lock released
```

Or if you don't need the lock held across the await:

```rust
async fn also_safe(data: &std::sync::Mutex<Vec<String>>) {
    {
        let mut guard = data.lock().unwrap();
        guard.push("item".to_string());
    }   // Guard dropped HERE, before the await

    slow_network_call().await;              // Lock is not held during this

    {
        let mut guard = data.lock().unwrap();
        guard.push("another".to_string());
    }
}
```

**Rule of thumb:**
- Use `std::sync::Mutex` for short critical sections with no `.await` inside. It's faster.
- Use `tokio::sync::Mutex` when you need to hold the lock across `.await` points.

---

## tokio::sync::Semaphore — Limiting Concurrency

A semaphore starts with N permits. Tasks acquire a permit to proceed; when they're done, they release it. If no permits are available, the task waits asynchronously.

This is the standard pattern for "fetch up to N things concurrently":

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

async fn fetch_repos_with_limit(repos: Vec<String>, concurrency: usize) {
    // Only `concurrency` fetches can run at the same time.
    let semaphore = Arc::new(Semaphore::new(concurrency));
    let mut handles = vec![];

    for repo in repos {
        let sem = Arc::clone(&semaphore);
        let handle = tokio::spawn(async move {
            // Acquire a permit. If `concurrency` tasks are already running,
            // this awaits (without blocking the thread) until one finishes.
            let _permit = sem.acquire().await.unwrap();
            // _permit is held here. When it's dropped, the slot opens for another task.

            fetch_repo_details(&repo).await
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

The permit is held for as long as `_permit` is in scope. When the async block ends and `_permit` is dropped, the semaphore counter increments back and a waiting task can proceed.

For `ghanalyze`, you'll use a semaphore to avoid hammering GitHub's API. GitHub allows 60 requests/hour unauthenticated and 5000/hour authenticated — you don't want to spawn 500 tasks all firing simultaneously.

---

## Summary Table

```
TOKIO SYNCHRONIZATION PRIMITIVES
==================================

Primitive              Use when
---------------------------------------------------------------------------
mpsc::channel          Many tasks produce, one task consumes (work queue)
oneshot::channel       One task needs a response from one other task
broadcast::channel     All subscribers need every event (shutdown signals)
watch::channel         Many observers need the current state (config)
Mutex                  Shared mutable state across tasks, need to hold across .await
Semaphore              Limit concurrency to N simultaneous operations
RwLock                 Shared data with frequent reads, rare writes
```

---

## Common Mistakes

**Using std::sync::mpsc in async code.** The `recv()` method blocks. Use `tokio::sync::mpsc` instead.

**Holding `std::sync::MutexGuard` across `.await`.** Described in detail above. This is silent in some configurations and causes deadlocks in others. Always use `tokio::sync::Mutex` if the lock needs to be held across an await.

**Making the broadcast buffer too small.** If a subscriber is slow, it can fall behind the ring buffer and get a `RecvError::Lagged(n)` error, meaning it missed `n` messages. Size the buffer according to how far behind the slowest subscriber might fall.

**Forgetting to clone the semaphore before moving it.** A semaphore used across multiple spawned tasks needs `Arc<Semaphore>`. If you move the semaphore into the first task, you can't use it for the rest.

**Creating per-request channels instead of shared channels.** Creating a new channel for each item in a loop is usually wrong — create one channel before the loop, then clone the sender for each task.

---

## How It Breaks

**`mpsc` sender outliving the receiver.** You drop the receiver (perhaps by returning from the function that owns it, or by explicitly dropping it), but senders still exist and are still calling `send()`. Each `send()` returns `Err(SendError)` because there is no receiver. If your code uses `.unwrap()` on sends, every subsequent send panics. If your code uses `.ok()` to ignore send errors, data is silently discarded. The fix depends on the design: either keep the receiver alive as long as senders exist, or check for `SendError` and treat it as a shutdown signal.

**`broadcast` channel lagging.** The broadcast channel stores messages in a fixed-size ring buffer. If a subscriber is slow — it doesn't call `recv()` quickly enough — the ring buffer wraps around and old messages are overwritten before the slow subscriber reads them. The next time that subscriber calls `recv()`, it gets `Err(RecvError::Lagged(n))` where `n` is the number of messages that were dropped. This is not a crash, but it means the subscriber missed data. If missing messages is unacceptable, increase the buffer size, add backpressure, or switch to `mpsc` with per-subscriber channels.

**`watch` channel: holding a borrow across an `.await`.** `rx.borrow()` returns a reference to the current value inside the watch channel. That borrow holds a read lock on the channel's internal state. If you hold that borrow across an `.await` point, the lock is held while the task is suspended. During that time, the sender cannot update the value — `tx.send()` will block or return an error. The fix is simple: grab the value you need, drop the borrow immediately, then do your async work with the extracted value rather than the live reference.
