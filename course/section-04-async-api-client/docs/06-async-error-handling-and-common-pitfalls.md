# 06 - Async Error Handling and Common Pitfalls 🔴

Error handling in async Rust is mostly the same as synchronous Rust - `Result<T, E>`, the `?` operator, `thiserror` for defining error types. But there are a handful of async-specific mistakes that are hard to debug because the compiler doesn't always catch them. This doc covers those, plus some tools to help when things go wrong.

---

## The Pitfall Hierarchy: From Annoying to Silent Bug

Not all mistakes are equally dangerous. Here is how to rank them by how hard they are to diagnose:

1. **Compiler error** - caught immediately. Missing `Send` bound, non-`'static` reference into a spawned task. You can't ship broken code. The cost is zero because you fix it before it runs.

2. **Runtime panic** - visible in logs. `unwrap()` on `None` inside a spawned task, `RefCell` borrow conflict, index out of bounds. The program crashes. Loud, easy to find in a stack trace.

3. **Silent hang** - the task runs but never completes. Awaiting a future that never resolves, deadlock, holding a lock that no one releases. The program appears to work but specific operations freeze. No error, no panic. Hard to reproduce under testing if the trigger is a race condition.

4. **Silent data loss** - the task completes but data is wrong or missing. A dropped future cancelled a write mid-operation. A broadcast channel lagged and dropped messages. A `select!` branch was cancelled unexpectedly. The program finishes successfully but produces incorrect output. The hardest to catch because automated tests often only check the happy path.

5. **Memory growth** - no crash yet, but approaching OOM. An unbounded channel filling up because the consumer is slow. Tasks accumulating in a JoinSet because no one calls `join_next`. Arc references not being released because of a cycle. The program is "working" but memory climbs steadily until the OS kills it.

The silent failures (3, 4, 5) are the dangerous ones. The compiler catches category 1. `#[should_panic]` tests and careful error handling catch category 2. Categories 3–5 require deliberately testing failure conditions, not just the happy path.

---

---

## Returning Result from Async Functions

The `?` operator works exactly the same in async functions as in sync ones:

```rust
use thiserror::Error;
use reqwest;

#[derive(Debug, Error)]
pub enum GhanalyzeError {
    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),

    #[error("GitHub API error {status}: {message}")]
    Api { status: u16, message: String },

    #[error("Rate limit exceeded, resets at {reset_time}")]
    RateLimited { reset_time: u64 },

    #[error("JSON parse error: {0}")]
    Json(#[from] serde_json::Error),
}

async fn fetch_user_repos(
    client: &reqwest::Client,
    username: &str,
) -> Result<Vec<GitHubRepo>, GhanalyzeError> {
    let response = client
        .get(&format!(
            "https://api.github.com/users/{}/repos",
            username
        ))
        .header("User-Agent", "ghanalyze/1.0")
        .send()
        .await?;  // ? works fine here - converts reqwest::Error via #[from]

    if response.status() == reqwest::StatusCode::FORBIDDEN {
        let reset = response
            .headers()
            .get("X-RateLimit-Reset")
            .and_then(|v| v.to_str().ok())
            .and_then(|s| s.parse().ok())
            .unwrap_or(0);
        return Err(GhanalyzeError::RateLimited { reset_time: reset });
    }

    if !response.status().is_success() {
        return Err(GhanalyzeError::Api {
            status: response.status().as_u16(),
            message: response.text().await?,
        });
    }

    Ok(response.json::<Vec<GitHubRepo>>().await?)  // ? converts reqwest::Error
}
```

No surprises here. The async-ness doesn't change how `?` works.

---

## Pitfall 1: std::thread::sleep in Async Code 🔴

Already mentioned in doc 03, but it's worth repeating because it's so common and so silent:

```rust
// WRONG: Blocks the OS thread. Every other task on this thread is frozen.
async fn rate_limit_wrong() {
    println!("Waiting 1 second...");
    std::thread::sleep(std::time::Duration::from_secs(1));  // The entire thread freezes!
}

// CORRECT: Yields to the executor. Other tasks run while this one waits.
async fn rate_limit_correct() {
    println!("Waiting 1 second...");
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
}
```

The compiler does not warn you about this. The program will "work" - it just performs terribly because your async concurrency is completely undermined. A program that should handle 100 concurrent requests becomes effectively sequential.

**How to spot it:** If your async program is inexplicably slow and you see `std::thread::sleep` anywhere in async code, that's your problem.

---

## Pitfall 2: std::sync::Mutex Across .await 🔴

Also covered in doc 04, but it deserves emphasis because the failure mode is subtle:

```rust
use std::sync::Mutex;

// This compiles and might even work in simple tests.
// Under load, it causes the executor thread to block, starving other tasks.
async fn dangerous(shared: &Mutex<Vec<String>>, item: String) {
    let mut guard = shared.lock().unwrap();
    guard.push(item);
    slow_network_call().await;    // MutexGuard held across this await!
    guard.push("done".to_string());
}
```

What actually happens: while `slow_network_call()` runs, the executor might try to run another task that also needs this mutex. On a single-threaded runtime, that task tries to acquire the mutex from the same thread - instant deadlock. On multi-threaded, the thread that holds the mutex is blocked, and the worker thread count effectively shrinks.

```rust
use tokio::sync::Mutex;  // Use this when holding across .await

async fn safe_version(shared: &Mutex<Vec<String>>, item: String) {
    let mut guard = shared.lock().await;  // Async lock, yields if contended
    guard.push(item);
    slow_network_call().await;            // OK with tokio::sync::Mutex
    guard.push("done".to_string());
}
```

---

## Pitfall 3: Forgetting to .await a Future 🟡

Calling an async function without `.await` does nothing. The future is created and immediately dropped. This is the async equivalent of calling a function and ignoring its return value, but worse - in sync code, you'd at least see the side effects. With futures, nothing happens at all.

```rust
async fn fetch_and_save(url: &str) {
    let data = download(url);        // BUG: Future created but never run!
    save_to_disk(data);              // This also does nothing - data has the wrong type
}

async fn fetch_and_save_correct(url: &str) {
    let data = download(url).await;  // Now it actually runs
    save_to_disk(data).await;
}
```

The compiler *sometimes* warns about unused futures but not always, especially when the future is passed to another function. The `#[must_use]` attribute on futures helps here, and many libraries use it.

```rust
// If you see this warning, you forgot to .await:
// warning: unused implementor of `Future` that must be used
// note: futures do nothing unless you `.await` or poll them
```

---

## Pitfall 4: Spawned Tasks Must Be 'static 🟡

Covered in doc 03, but the error message is confusing enough that it bears repeating with context:

```rust
async fn process_all(repos: &Vec<GitHubRepo>) {
    for repo in repos {
        tokio::spawn(async {
            // ERROR: `repo` doesn't live long enough.
            // `repos` is borrowed from the function parameter,
            // which might be dropped before this task runs.
            process_repo(repo).await;
        });
    }
}

// Fix: clone the data into the task, or use Arc.
async fn process_all_correct(repos: Vec<GitHubRepo>) {
    for repo in repos {
        tokio::spawn(async move {   // `move` takes ownership of `repo`
            process_repo(&repo).await;
        });
    }
}
```

When you see "the trait `Send` is not implemented" or "does not live long enough" errors related to `tokio::spawn`, the issue is almost always that you're trying to borrow something across a spawn boundary. Either clone the data, or wrap it in `Arc`.

---

## Pitfall 5: Mixing Blocking and Async Without spawn_blocking 🔴

Some libraries are inherently blocking. SQLite drivers, older file operations, CPU-intensive work. You can't just `.await` them. If you call them directly from async code, you block the executor thread:

```rust
// WRONG: Blocks the executor thread while computing
async fn analyze_code(path: &str) -> AnalysisResult {
    // This might take 500ms of CPU time, blocking all other tasks
    run_static_analysis(path)
}

// CORRECT: Run on the blocking thread pool
async fn analyze_code_correct(path: String) -> AnalysisResult {
    tokio::task::spawn_blocking(move || {
        run_static_analysis(&path)
    }).await.unwrap()
}
```

The blocking thread pool is separate from the async worker pool. Blocking operations there don't affect async tasks.

Note that `spawn_blocking` takes ownership (requires `'static`), which is why we change `path: &str` to `path: String` in the corrected version.

---

## Pitfall 6: Sequential Awaits When You Want Concurrency 🟡

This one is subtle because the code looks right:

```rust
// WRONG: Runs sequentially. If each request takes 100ms, this takes 300ms.
async fn fetch_three_repos(client: &reqwest::Client) -> Vec<Repo> {
    let linux = fetch_repo(client, "torvalds/linux").await.unwrap();
    let git = fetch_repo(client, "git/git").await.unwrap();
    let cpython = fetch_repo(client, "python/cpython").await.unwrap();
    vec![linux, git, cpython]
}

// CORRECT: Runs concurrently. Takes ~100ms (the longest single request).
async fn fetch_three_repos_concurrent(client: &reqwest::Client) -> Vec<Repo> {
    let (linux, git, cpython) = tokio::join!(
        fetch_repo(client, "torvalds/linux"),
        fetch_repo(client, "git/git"),
        fetch_repo(client, "python/cpython"),
    );
    vec![linux.unwrap(), git.unwrap(), cpython.unwrap()]
}
```

If you're not seeing the concurrency speedup you expect, check if your awaits are sequential. Each `let x = something().await` line blocks until that specific future completes before moving on.

---

## tokio::task::JoinSet - Handling Dynamic Task Collections

When you have a dynamic number of tasks (not known at compile time), `JoinSet` handles them cleanly, especially around error handling:

```rust
use tokio::task::JoinSet;

async fn analyze_repos(repos: Vec<String>) -> Vec<Result<RepoAnalysis, GhanalyzeError>> {
    let mut set = JoinSet::new();

    for repo in repos {
        set.spawn(async move {
            // Each task independently handles its own errors.
            analyze_single_repo(&repo).await
        });
    }

    let mut results = vec![];
    while let Some(join_result) = set.join_next().await {
        match join_result {
            Ok(analysis_result) => results.push(analysis_result),
            Err(join_error) => {
                // The task panicked. Log it, don't crash.
                eprintln!("Task panicked: {}", join_error);
            }
        }
    }

    results
}
```

This is the pattern you'll use in `ghanalyze`: spawn one task per repo, collect results as they complete (not waiting for all to finish before processing any), and handle individual task failures without crashing the whole analysis.

---

## Tracing: Logging for Async Code 🟢

Standard `println!` and `log` macros don't understand async. If you log inside an async function, you lose track of *which* task logged what. The `tracing` crate is async-aware: it maintains context across `.await` points.

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

```rust
use tracing::{info, warn, error, instrument};

#[instrument]  // Automatically creates a span with the function name and arguments
async fn fetch_repo(username: &str, repo_name: &str) -> Result<GitHubRepo, GhanalyzeError> {
    info!("Fetching repo");  // Logs include the span context: username, repo_name

    let result = make_request(username, repo_name).await;

    match &result {
        Ok(repo) => info!(stars = repo.stargazers_count, "Fetch successful"),
        Err(e) => warn!(error = %e, "Fetch failed"),
    }

    result
}

#[tokio::main]
async fn main() {
    // Set up the tracing subscriber (writes to stderr by default).
    tracing_subscriber::fmt::init();

    fetch_repo("torvalds", "linux").await.unwrap();
}
```

`#[instrument]` is the key attribute. It wraps the function in a tracing *span*, so every log inside the function includes context (like the username and repo_name arguments). When multiple tasks are running concurrently, the logs stay organized by which task produced them.

Set the log level with the `RUST_LOG` environment variable:
```
RUST_LOG=debug cargo run -- torvalds
RUST_LOG=ghanalyze=info cargo run -- torvalds   # Only show your crate's logs
```

---

## Common Mistakes Summary

```
MISTAKE                           SYMPTOM                    FIX
----------------------------------------------------------------------
std::thread::sleep in async       Program slow, low CPU      tokio::time::sleep().await
std::sync::Mutex across .await    Deadlocks under load       tokio::sync::Mutex or scope lock
Forgetting .await                 Nothing happens, no error  Always .await async calls
Borrowing across spawn            Compile error: not 'static clone or Arc
Blocking code in async            Low throughput             spawn_blocking
Sequential awaits                 No concurrency speedup     tokio::join!
```

---

## A Debugging Checklist

When your async program hangs, runs slowly, or produces wrong results:

1. **Is there `std::thread::sleep` in async code?** Replace with `tokio::time::sleep().await`.
2. **Is there `std::sync::Mutex` held across `.await`?** Switch to `tokio::sync::Mutex` or scope the lock.
3. **Are concurrent operations actually concurrent?** Check for sequential awaits where you need `tokio::join!`.
4. **Are tasks completing?** Add tracing logs at task start and end to confirm tasks are running.
5. **Are errors being swallowed?** Check that you're not silently ignoring `Err` variants in `match` or using `.ok()` too aggressively.
6. **Are JoinHandle results being checked?** If you drop a JoinHandle without awaiting it, panics in that task are invisible.
