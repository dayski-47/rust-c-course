# 01 - Why Async? When Threads Aren't Enough 🟢

## Engineering Methodology: Resilience and Failure-Mode Thinking

Every external call in this section can fail. The GitHub API can be down. You can be rate-limited. The response can be malformed. The network can drop.

Before writing any code that touches a network, for every external call, write down:
- What does success look like?
- What does failure look like? (and there are multiple kinds of failure)
- What should your program do for each failure case?

This is failure-mode thinking. Engineers who don't practice it write code that works in demos but crashes in production. The GitHub analyzer has at least five distinct failure modes: rate limit exceeded, invalid or nonexistent username, network timeout, malformed JSON in the response, and pagination failure on large accounts. Each one needs a deliberate response from your code - not the same response, specific ones.

The `?` operator is not enough by itself. Using `?` handles the "something went wrong" case generically - it propagates the error upward. But it doesn't let you distinguish which thing went wrong, and it doesn't let you handle each failure differently. A rate limit error should trigger a sleep-and-retry. An invalid username should print a helpful message and exit. A malformed JSON response should log the raw response for debugging. These are different behaviors for different failure modes, and `?` alone can't express them.

---

You've spent the last three sections writing threaded code. You spawned OS threads, passed data through channels, shared state with `Arc<Mutex<T>>`. That all works well for a system monitor or a chat server. So why are we changing approach?

The honest answer: threads are fine for dozens or hundreds of concurrent tasks. But when you need *thousands* of concurrent I/O operations, threads stop scaling. Let's see why.

---

## The Problem: 1000 Concurrent HTTP Requests

Imagine you're writing a tool to check the health of 1000 servers. You want to send an HTTP request to each one and collect the results. With threads, the natural approach is:

```rust
use std::thread;

fn check_all(servers: Vec<String>) {
    let mut handles = vec![];

    for server in servers {
        let handle = thread::spawn(move || {
            // Each thread makes one HTTP request and waits for the response
            let response = make_http_request(&server);
            println!("{}: {}", server, response.status);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

This spawns 1000 OS threads. Each thread blocks while waiting for the HTTP response. What does that cost?

Each OS thread needs a **stack**. On Linux, the default stack size is 8MB, but even with a reduced 512KB stack, 1000 threads means 500MB of memory just for stacks - before you've even stored any data. The OS also has to schedule 1000 threads, context-switching between them constantly, even though almost all of them are just sitting idle waiting for network I/O.

The CPU is not the bottleneck here. The *network* is. You're spending most of your time waiting. And threads are a terrible way to wait.

---

## The Core Idea: Don't Waste the Thread While Waiting

When a thread calls `read()` on a network socket and the data isn't there yet, the OS puts the thread to sleep. The thread's stack sits in memory doing nothing. It can't do any other work. It's just... parked.

Async Rust's key insight is: **instead of blocking a thread while waiting for I/O, yield control back to a shared executor, which can do something useful with that thread in the meantime.**

Here's the same concept with async:

```rust
use tokio::task;

async fn check_all(servers: Vec<String>) {
    let mut handles = vec![];

    for server in servers {
        // This is a lightweight task, not an OS thread
        let handle = task::spawn(async move {
            let response = make_async_http_request(&server).await;
            println!("{}: {}", server, response.status);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

Under the hood, Tokio runs this with maybe 4–8 OS threads (one per CPU core). All 1000 tasks share those threads. When one task is waiting for network I/O, its thread immediately picks up another task and runs it. No thread sits idle.

---

## Threads vs Async Tasks: The Key Differences

```
THREAD MODEL (OS-managed)
==========================

Thread 1:  [====HTTP WAIT==========][work][===HTTP WAIT===][work]
Thread 2:  [=======HTTP WAIT==============][work][==HTTP WAIT==]
Thread 3:  [==HTTP WAIT====][work][=======HTTP WAIT=======][work]
...
Thread 1000: [====HTTP WAIT=========][work]

All these threads exist simultaneously, each consuming stack memory.
Most of them are blocked (=====) most of the time.


ASYNC MODEL (cooperative, executor-managed)
============================================

Thread 1:  [T1][T4][T7][T2][T9][T3][T5][T8][T6][T1][T4]...
           ^ Tasks switch at .await points (I/O wait)

Thread 2:  [T10][T11][T15][T12][T16][T13][T17][T14]...

Only 2-8 OS threads. Tasks hand control back when they hit .await.
```

The critical difference in scheduling:

- **OS threads** are *preemptively* scheduled. The OS can pause a thread at any instruction, mid-loop, mid-function, anywhere. The thread has no say in the matter.
- **Async tasks** are *cooperatively* scheduled. A task runs until it hits a `.await` point and voluntarily yields. Other tasks only get to run at those yield points.

This has an important implication: if an async task never hits a `.await` (say it's doing a long computation), it hogs the thread and starves other tasks. More on that in the pitfalls section.

---

## Rust's Approach: Zero-Cost Async

Languages like Go and JavaScript bake their async runtime directly into the language. When you write async code in Go, the goroutine scheduler is always present. When you write async code in JavaScript, the event loop is always there.

Rust takes a different approach. The `async` keyword is a **compilation strategy**. When you write this:

```rust
async fn fetch_url(url: &str) -> String {
    let response = make_request(url).await;
    response.body
}
```

The compiler transforms it into a state machine - roughly equivalent to:

```rust
enum FetchUrlState {
    Start,
    WaitingForResponse { url: String },
    Done,
}

struct FetchUrlFuture {
    state: FetchUrlState,
}

// The actual Future trait implementation is generated by the compiler.
// Each .await point becomes a state transition.
```

This means there is no runtime overhead unless you are actually doing async work. If you write a simple program with no `.await` calls, you pay nothing for the async machinery. The transformation is done at compile time.

However - and this is critical - **someone still has to drive that state machine**. That's the executor's job. In Rust, the executor is *not* built in. You have to bring one. The most popular is **Tokio**, which is what we'll use throughout this section.

---

## When to Use Async

Async is not always the right tool. Here's a practical guide:

**Use async when:**
- You're making many concurrent network requests (HTTP, gRPC, WebSocket)
- You're talking to databases concurrently
- You're handling many simultaneous client connections in a server
- You're doing file I/O that could overlap with other work
- You need to handle hundreds or thousands of concurrent lightweight tasks

**Don't use async when:**
- You're doing CPU-intensive computation (image processing, compression, encryption). For that, use OS threads or `rayon`. Async doesn't help - the CPU is the bottleneck, not I/O.
- Your program does one thing at a time and doesn't need concurrency at all. Async adds complexity for no benefit.
- You're working with existing blocking code you can't change. You need `spawn_blocking` to bridge the gap (covered in doc 03).
- You're writing a quick script or CLI tool that makes a single request. Async is overkill; it adds a runtime dependency and boilerplate.

Think of it this way: **async is for waiting efficiently. Threads are for working in parallel.**

---

## A Quick Taste of Tokio

You need two things to write async Rust: the `async/await` syntax, and a runtime to execute it. Tokio is the runtime we'll use:

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
// The #[tokio::main] macro sets up the Tokio runtime and
// runs your async main function inside it.
#[tokio::main]
async fn main() {
    // tokio::join! runs multiple futures concurrently.
    // Both "requests" start at the same time.
    let (result_a, result_b) = tokio::join!(
        simulate_http_request("server-a"),
        simulate_http_request("server-b"),
    );

    println!("A: {}, B: {}", result_a, result_b);
}

async fn simulate_http_request(name: &str) -> String {
    // In real code, this would be an actual HTTP call.
    // tokio::time::sleep is the async version of thread::sleep.
    tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    format!("{} responded OK", name)
}
```

Notice `tokio::time::sleep(...).await` - that `.await` is the yield point. While this task is sleeping, the Tokio executor is free to run other tasks. When the sleep completes, this task picks up right where it left off.

---

## Common Mistakes

**Confusing concurrency with parallelism.** Async gives you concurrency - many tasks making progress by taking turns. For *parallelism* (multiple CPU cores doing computation simultaneously), you still need threads. The two can be combined: Tokio's multi-threaded runtime runs async tasks across multiple CPU cores.

**Thinking async is always faster.** For a single HTTP request, async is actually slightly slower than a blocking call because of the state machine overhead. Async wins at scale, not in single-operation benchmarks.

**Blocking the executor.** The most common mistake is calling blocking code (like `std::thread::sleep` or `std::fs::read_to_string`) inside an async function without using `spawn_blocking`. This blocks the OS thread and starves other tasks. This is covered in depth in doc 03 and doc 06.

**Using async when you don't need it.** Not every Rust program needs async. If you're writing a CLI tool that makes one HTTP request and exits, just use a blocking HTTP client. Save async for when you actually need concurrency.

---

## How It Breaks

**Using async where you need CPU parallelism.** Async multiplexes I/O on fewer threads - it does not parallelize CPU work. If you have a task that hashes 10,000 files or runs a compression algorithm, putting it in an async task doesn't speed it up. All async tasks on a Tokio worker thread still take turns - one runs until it hits `.await`, then another gets a turn. If your task never yields (no I/O, no `.await`), it monopolizes the thread. For real CPU parallelism, use `rayon` (which creates a thread pool sized to CPU cores) or `tokio::task::spawn_blocking` (which moves the work to a separate blocking thread pool).

**Mixing sync and async: calling a blocking function from async code.** `std::thread::sleep`, synchronous file reads with `std::fs`, blocking database drivers, any C library that blocks - these all block the OS thread they run on. In an async context, that means the Tokio worker thread is frozen. While it's frozen, no other tasks on that thread can run. If your runtime has 4 worker threads and all 4 are blocked on synchronous calls, your entire async program is frozen. The fix is `spawn_blocking` for blocking operations, and the async equivalents (`tokio::time::sleep`, `tokio::fs`) for I/O.

**Forgetting that the multi-thread runtime changes `Send` requirements.** In a single-threaded runtime (current_thread flavor), your tasks never move between threads, so they can hold non-`Send` types. When you switch to the default multi-threaded runtime, Tokio may move a task from one worker thread to another at any `.await` point. This means everything your task holds across `.await` must implement `Send`. If you were relying on non-`Send` types (like `Rc`, `RefCell`, raw pointers, or certain library types), the compiler will start rejecting your code when you switch runtime flavors. This is caught at compile time, but the error messages can be confusing if you don't know what's driving them.
