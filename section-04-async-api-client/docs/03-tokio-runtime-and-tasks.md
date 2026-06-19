# 03 — Tokio Runtime and Tasks 🟡

Now that you understand futures conceptually, let's look at the machinery that actually runs them. Tokio is the runtime — the engine that powers everything in this section. It's the most widely used async runtime in the Rust ecosystem and the foundation for most production Rust services.

---

## What a Runtime Is

A Tokio runtime does two things:

1. **Polls futures** — it calls `poll()` on tasks that are ready to make progress, advancing their state machines.
2. **Manages I/O events** — it registers interest in OS-level events (network data arriving, file reads completing) and wakes the right tasks when those events fire.

Under the hood, Tokio uses the OS's native async I/O notification systems: `epoll` on Linux, `kqueue` on macOS, and IOCP on Windows. You don't interact with these directly; Tokio wraps them.

Without a runtime, your futures are just structs sitting in memory doing nothing. The runtime is what makes them actually execute.

---

## #[tokio::main]

The simplest way to start a Tokio runtime:

```rust
#[tokio::main]
async fn main() {
    println!("Running inside Tokio!");
    tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    println!("Done.");
}
```

The `#[tokio::main]` macro expands to something like:

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            // your async main function body goes here
        })
}
```

It creates a multi-threaded runtime with one worker thread per CPU core, enables all I/O and timer features, and runs your async main on it.

### Runtime Flavors

Tokio has two runtime configurations:

```rust
// Default: multi-threaded, work-stealing scheduler.
// One OS thread per CPU core. Tasks can move between threads.
// Requires spawned futures to be Send + 'static.
#[tokio::main]
async fn main() { /* ... */ }

// Single-threaded: everything runs on one OS thread.
// Good for simple tools, WASM, or when you have !Send types.
// Spawned futures don't need to be Send.
#[tokio::main(flavor = "current_thread")]
async fn main() { /* ... */ }
```

For `ghanalyze`, the default multi-threaded runtime is correct. We want concurrent fetches running on multiple cores.

---

## tokio::spawn — Creating Async Tasks

`tokio::spawn` is to async Rust what `thread::spawn` is to threaded Rust: it creates a new independent unit of work.

```rust
#[tokio::main]
async fn main() {
    // Spawn a background task. Returns a JoinHandle immediately.
    let handle = tokio::spawn(async {
        tokio::time::sleep(std::time::Duration::from_millis(500)).await;
        println!("Background task completed");
        42  // The task's return value
    });

    println!("Main task continues while background runs...");

    // Await the handle to get the task's result.
    // This is where we wait for it to finish.
    let result = handle.await.unwrap();
    println!("Task returned: {}", result);
}
```

Unlike `thread::spawn`, spawning a task is very cheap. A Tokio task uses a small amount of heap memory for its state machine, not an 8MB stack. You can spawn thousands of them without issue.

### The 'static Requirement

Here's a difference from `thread::spawn` that will bite you early: spawned tasks must be `'static`.

```rust
async fn example() {
    let data = String::from("hello");

    // WRONG: Borrows data, which is not 'static.
    // What if `data` is dropped before this task finishes?
    // tokio::spawn(async {
    //     println!("{}", data);  // ERROR: `data` does not live long enough
    // });

    // CORRECT: Move ownership into the task.
    let handle = tokio::spawn(async move {
        println!("{}", data);  // data is moved into the future, owned by the task
    });

    handle.await.unwrap();
}
```

The reason for `'static`: a spawned task runs independently. It might outlive the function that created it. The compiler can't prove that borrowed references will stay valid, so it requires ownership.

The practical consequence: **you can't share data by reference across `tokio::spawn` boundaries**. Use `Arc<T>` to share data, or channels to communicate:

```rust
use std::sync::Arc;

async fn process_repos(repos: Vec<String>) {
    // Arc lets multiple tasks share the same data without copying.
    let client = Arc::new(reqwest::Client::new());

    let mut handles = vec![];
    for repo in repos {
        let client = Arc::clone(&client);  // Clone the Arc, not the Client
        let handle = tokio::spawn(async move {
            fetch_repo_details(&client, &repo).await
        });
        handles.push(handle);
    }

    for handle in handles {
        let result = handle.await.unwrap();
        println!("{:?}", result);
    }
}

async fn fetch_repo_details(client: &reqwest::Client, repo: &str) -> String {
    // Use the shared client here
    format!("details for {}", repo)
}
```

### JoinHandle

`tokio::spawn` returns a `JoinHandle<T>`, where `T` is the return type of the spawned future. You `.await` the handle to get the result:

```rust
let handle: tokio::task::JoinHandle<String> = tokio::spawn(async {
    "hello".to_string()
});

let result: Result<String, tokio::task::JoinError> = handle.await;
let value: String = result.unwrap();
```

The outer `Result` is `Err` if the task panicked. Handle it if your task might panic.

### Aborting Tasks

Dropping a `JoinHandle` does NOT cancel the task. The task keeps running as a detached background task. To actually stop it:

```rust
let handle = tokio::spawn(long_running_task());

// Cancel the task:
handle.abort();

// Awaiting after abort returns a JoinError with is_cancelled() == true.
match handle.await {
    Ok(value) => println!("Completed: {:?}", value),
    Err(e) if e.is_cancelled() => println!("Task was cancelled"),
    Err(e) => println!("Task panicked: {}", e),
}
```

---

## Time: sleep, timeout

### tokio::time::sleep — The Async Version

Never use `std::thread::sleep` inside async code. Here's why it's catastrophic:

```
WRONG: std::thread::sleep(Duration::from_secs(1))
=====================================================
Thread 1: [=========BLOCKED FOR 1 SECOND===========]
          ^ No other tasks can run on this thread!
          ^ If you have 4 threads and 4 tasks all call this,
            your entire runtime freezes for 1 second.

CORRECT: tokio::time::sleep(Duration::from_secs(1)).await
==========================================================
Thread 1: [T1 yields]...[T2]...[T3]...[T4]...[T1 resumes after 1s]
          ^ Thread is free to run other tasks while T1 waits.
```

```rust
// WRONG — blocks the OS thread:
async fn bad_rate_limiter() {
    std::thread::sleep(std::time::Duration::from_secs(1));  // Freezes the thread!
}

// CORRECT — yields to the executor:
async fn good_rate_limiter() {
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
}
```

### tokio::time::timeout — Deadline for a Future

Wrap any future with a deadline. If it doesn't complete in time, you get an error:

```rust
use tokio::time::{timeout, Duration};

async fn fetch_with_deadline(url: &str) -> Result<String, String> {
    match timeout(Duration::from_secs(5), make_request(url)).await {
        Ok(response) => Ok(response),
        Err(_elapsed) => Err(format!("Request to {} timed out after 5s", url)),
    }
}
```

`timeout` returns `Result<T, Elapsed>` where `T` is the original future's output type.

---

## tokio::fs — Async File I/O

Like with sleep, using `std::fs` in async code is a problem because file operations can block:

```rust
// WRONG — blocking file I/O on the async thread:
async fn read_config() -> String {
    std::fs::read_to_string("config.json").unwrap()  // Can block!
}

// CORRECT — tokio's async filesystem:
async fn read_config() -> String {
    tokio::fs::read_to_string("config.json").await.unwrap()
}
```

`tokio::fs` mirrors `std::fs` but uses async I/O under the hood.

---

## spawn_blocking — Bridging Sync and Async

Sometimes you have blocking code you can't avoid: a CPU-intensive computation, a database driver that's synchronous, or a third-party library that blocks. `spawn_blocking` runs that code on a separate thread pool dedicated to blocking work, so it doesn't starve your async tasks:

```rust
#[tokio::main]
async fn main() {
    // This runs on a dedicated "blocking thread pool", separate from the
    // async worker threads. The async runtime is not blocked.
    let result = tokio::task::spawn_blocking(|| {
        // Anything in here can block, call std::thread::sleep, do CPU work, etc.
        do_heavy_computation()
    }).await.unwrap();

    println!("Result: {}", result);
}

fn do_heavy_computation() -> u64 {
    // Simulate heavy work that takes time
    let mut sum = 0u64;
    for i in 0..10_000_000 {
        sum += i;
    }
    sum
}
```

```
When to use spawn_blocking:
- Parsing a large file synchronously
- Calling a blocking database driver (sqlite, etc.)
- Heavy cryptographic operations
- Any C library that blocks
- std::fs operations on large files

When NOT to use spawn_blocking:
- Network I/O (use async reqwest instead)
- Tokio's own async file operations
- Anything that already has an async version
```

---

## tokio::task::JoinSet — Managing Many Tasks

When you're spawning a dynamic number of tasks (like fetching details for each of N repos), `JoinSet` is cleaner than managing a `Vec<JoinHandle>`:

```rust
use tokio::task::JoinSet;

async fn fetch_all_repos(repos: Vec<String>) -> Vec<String> {
    let mut set = JoinSet::new();

    for repo in repos {
        set.spawn(async move {
            // Fetch details for this repo
            fetch_repo_details(&repo).await
        });
    }

    let mut results = vec![];
    // Process results as each task completes (in completion order, not spawn order)
    while let Some(result) = set.join_next().await {
        match result {
            Ok(details) => results.push(details),
            Err(e) => eprintln!("Task failed: {}", e),
        }
    }

    results
}
```

`join_next()` returns the result of whichever task completes first, which means you can start processing results as soon as any task finishes rather than waiting for the last one.

---

## Common Mistakes

**Using std::thread::sleep in async code.** This is the most common performance killer. Always use `tokio::time::sleep(...).await`. The compiler will not warn you; it's a silent correctness issue.

**Using std::fs in async code for large files.** For small config files it's usually fine in practice, but for large files or high-throughput code, use `tokio::fs` or `spawn_blocking`.

**Not handling the JoinError.** `handle.await` returns `Result<T, JoinError>`. If you just `.unwrap()` it and the task panics, your program panics too. In production, match on the error.

**Spawning too many tasks at once.** While tasks are cheap, spawning 100,000 tasks all at once before any can run can cause memory issues. For bounded concurrency (fetch N repos but only 10 at a time), use `tokio::sync::Semaphore` — covered in the next doc.

**Thinking that detached tasks clean themselves up.** If you drop a `JoinHandle` without awaiting it, the task runs to completion (or panic) in the background with no way to collect its result or check for errors. This is sometimes what you want, but often it's a resource leak waiting to happen.

---

## How It Breaks

**Spawning a task and never awaiting the `JoinHandle`.** The task runs in the background but its result is lost. If the task returns an important value or error, you'll never see it. More critically: if the task panics, the panic doesn't propagate anywhere — the task simply disappears. The `JoinHandle` would have given you `Err(JoinError)` containing the panic info, but you dropped it. In production, dropped `JoinHandle`s that panic appear as silent failures.

**Running CPU-heavy work on Tokio worker threads.** Tokio's worker threads are shared among all async tasks. If one task runs a tight CPU loop without ever hitting `.await`, it monopolizes its worker thread for the duration of that loop. Other tasks that are ready to make progress can't run because the thread is busy. On a 4-thread runtime, if 4 tasks all do CPU work simultaneously, all async I/O in the entire program stalls. Move CPU-heavy work to `spawn_blocking`, which uses a separate thread pool that doesn't affect the async workers.

**Panicking inside a spawned task.** Panics inside `tokio::spawn` don't propagate to the spawning task. The runtime catches the panic, terminates the task, and records it in the `JoinError`. If you call `handle.await.unwrap()` and the task panicked, the `unwrap()` will re-panic in your code — but only if you actually `await` the handle. If you dropped the handle, the panic is silently swallowed. Always check `JoinHandle` results, especially in long-running servers.

**`'static` requirement biting you.** `tokio::spawn` requires the future to be `'static` because the task may outlive the calling function. This means you cannot borrow local variables into a spawned task. The error message ("does not live long enough" or "borrowed value must be valid for `'static`") can be confusing. The solutions are always one of: `Arc::clone` to share ownership, `move` to transfer ownership, or channel messages to send data into the task. You cannot pass references across `spawn` boundaries.
