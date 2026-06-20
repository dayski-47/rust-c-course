# Doc 08 — Cancellation Safety and tokio::select!

🔴 This is where async bugs hide — code that looks correct but corrupts state on cancellation.

Async Rust has a property C programmers rarely think about explicitly: when you stop waiting for something — by dropping a future, timing out, or handling a shutdown signal — that operation is cancelled *immediately at the next `.await` point*. No cleanup runs. No destructor notifies the underlying work. The state machine is dropped in the middle of what it was doing.

This sounds dangerous. It is, if you're not prepared for it. This doc explains what "cancellation safe" means, how `tokio::select!` works, and how to write async code that handles cancellation without corrupting state.

---

## What Happens When a Future is Cancelled

In Rust, dropping a future cancels it. When you stop polling a future (by dropping it), the async state machine is destroyed at whatever `.await` point it was suspended at.

Consider a simple example:

```rust
async fn risky_transfer(from: &Account, to: &Account, amount: u64) {
    from.debit(amount).await;   // Step 1
    // If dropped HERE, money is debited but not credited
    to.credit(amount).await;    // Step 2
}
```

If this future is cancelled between step 1 and step 2, the account is inconsistent. The money is gone. This is the fundamental challenge of cancellation: **any work done before the last `.await` point in the cancelled code might be partial.**

In C, you'd solve this with mutexes and rollback logic. In Rust, the solution is structural: design your async functions so that partial execution leaves state consistent, or ensure they run to completion before accepting cancellation.

---

## `tokio::select!`: Racing Multiple Futures

`tokio::select!` runs multiple futures simultaneously and completes when **any one of them** resolves. All the others are **immediately dropped** — cancelled.

```rust
use tokio::time::{timeout, Duration};

async fn fetch_with_timeout(url: &str) -> Option<String> {
    tokio::select! {
        response = reqwest::get(url) => {
            // fetch completed first
            response.ok()?.text().await.ok()
        }
        _ = tokio::time::sleep(Duration::from_secs(5)) => {
            // timeout fired first — the fetch future is dropped
            eprintln!("Request to {url} timed out");
            None
        }
    }
}
```

When the sleep fires, `reqwest::get(url)` is dropped — the HTTP connection is closed, any partial response body is discarded, and the future's state machine is freed. This is intentional — but if `reqwest::get` had already partially read a response body that was shared elsewhere, you'd have a problem.

### The Four Patterns of `select!`

```rust
tokio::select! {
    // Pattern 1: plain future
    result = some_future => { /* result is the future's output */ }
    
    // Pattern 2: future with a condition — only enters the arm if the guard is true
    result = fallible_future, if should_attempt => { /* only active when should_attempt */ }
    
    // Pattern 3: receive from a channel
    Some(msg) = rx.recv() => { /* msg is the received value, arm skipped if channel closed */ }
    
    // Pattern 4: else branch — taken when ALL other branches have their guards as false
    else => { /* fallthrough when all guards are false */ }
}
```

---

## What "Cancellation Safe" Means

A function is **cancellation safe** if dropping it at any `.await` point leaves the world in a consistent state. No partial writes, no lost data, no leaked resources.

The standard library documents this property for its primitives. For example:

| Operation | Cancellation Safe? | Why |
|-----------|-------------------|-----|
| `tokio::time::sleep()` | ✅ Yes | Dropping it just cancels the timer — no state changes |
| `channel.recv()` | ✅ Yes | If cancelled before receiving, no message is consumed |
| `channel.send(val)` | ❌ No | If cancelled mid-send, `val` may be partially enqueued |
| `channel.recv()` in a loop | ⚠️ Context-dependent | See below |
| `AsyncRead::read()` | ❌ No | Cancellation mid-read loses the partial buffer state |
| `join!()` | ⚠️ Mixed | Depends on whether any branch has side effects |

### The `recv()` in a Loop Problem

This is the most common cancellation bug in production async Rust:

```rust
// This looks safe but has a hidden bug:
async fn process_batch(rx: &mut mpsc::Receiver<Item>) {
    let mut batch = Vec::new();
    
    while batch.len() < 10 {
        match rx.recv().await {  // ← if dropped HERE...
            Some(item) => batch.push(item),
            None => break,
        }
    }
    
    flush_batch(batch).await;
}
```

If `process_batch` is wrapped in a `select!` and cancelled while `rx.recv().await` is awaiting, no item is lost — `recv()` is cancellation safe. The item never left the channel.

But here's the dangerous version:

```rust
// BUG: If cancelled after recv but before push, the item is silently dropped
async fn process_batch_with_side_effect(rx: &mut mpsc::Receiver<Item>) {
    let mut count = 0;
    
    loop {
        let item = rx.recv().await;  // item received from channel
        count += 1;                  // count incremented
        db.insert(item).await;       // ← if cancelled HERE...
        // count and db are now inconsistent: count says N but db has N-1
    }
}
```

The item was consumed from the channel (it won't be there when the function is resumed). The database write didn't complete. Now count doesn't match the database. This is a cancellation safety violation.

---

## Strategies for Cancellation Safety

### Strategy 1: Make the operation atomic

Use database transactions or other all-or-nothing primitives. Either everything commits, or nothing does:

```rust
async fn safe_batch_process(items: Vec<Item>, db: &Database) -> Result<(), Error> {
    let tx = db.begin_transaction().await?;
    
    for item in items {
        tx.insert(item).await?;
    }
    
    // If cancelled before commit, transaction rolls back automatically
    tx.commit().await?;
    Ok(())
}
```

### Strategy 2: Separate collection from processing

Collect first (doing only cancellation-safe work), then process in a way that can complete:

```rust
async fn process_with_backlog(rx: &mut mpsc::Receiver<Item>) {
    // Phase 1: collect a batch — recv() is cancellation safe
    let mut batch = Vec::new();
    while batch.len() < 10 {
        match rx.recv().await {
            Some(item) => batch.push(item),
            None => break,
        }
    }
    
    // Phase 2: process the batch — not inside a select!, won't be cancelled
    // Run this outside any select! to ensure it completes:
    flush_to_database(&batch).await;
}
```

### Strategy 3: Use `select!` only at safe boundaries

Structure your code so `select!` only appears at points where all active state is consistent:

```rust
async fn rate_limited_fetcher(
    repos: Vec<String>,
    mut shutdown: watch::Receiver<bool>,
) -> Vec<RepoInfo> {
    let mut results = Vec::new();
    
    for repo in &repos {
        tokio::select! {
            // Only race at the TOP of the loop — when state is consistent
            info = fetch_repo_info(repo) => {
                results.push(info);
            }
            _ = shutdown.changed() => {
                // Shutdown — we have a partial result set, but results is consistent
                break;
            }
        }
    }
    
    results  // Partial but consistent — no torn writes
}
```

---

## The GitHub Analyzer: Applying Cancellation Safety

The GitHub analyzer needs to:
1. Fetch data from multiple repositories concurrently (with a rate limit)
2. Handle timeouts on individual requests
3. Support graceful shutdown (Ctrl+C before all repos are fetched)

Each of these is a `select!` use case:

```rust
use tokio::sync::{mpsc, watch};
use tokio::time::{timeout, Duration};
use tokio_util::sync::CancellationToken;

pub struct GitHubAnalyzer {
    client: reqwest::Client,
    rate_limit: Arc<Semaphore>,
    cancel: CancellationToken,
}

impl GitHubAnalyzer {
    /// Fetch one repo — handles its own timeout, cancellation-safe at the entry point
    async fn fetch_repo(
        &self,
        repo: &str,
    ) -> Option<RepoInfo> {
        tokio::select! {
            // Respect global cancellation (user pressed Ctrl+C)
            _ = self.cancel.cancelled() => {
                None
            }
            
            // Try to fetch with a per-request timeout
            result = async {
                let _permit = self.rate_limit.acquire().await.ok()?;
                
                timeout(
                    Duration::from_secs(10),
                    self.do_fetch(repo),
                ).await.ok()?.ok()
            } => result
        }
    }

    async fn do_fetch(&self, repo: &str) -> Result<RepoInfo, Error> {
        let url = format!("https://api.github.com/repos/{repo}");
        let info: RepoInfo = self.client
            .get(&url)
            .send().await?
            .error_for_status()?
            .json().await?;
        Ok(info)
    }

    /// Analyze all repos, stopping on cancellation
    pub async fn analyze_all(&self, repos: Vec<String>) -> Vec<RepoInfo> {
        let mut handles = Vec::new();
        
        for repo in repos {
            let analyzer = self.clone();
            handles.push(tokio::spawn(async move {
                analyzer.fetch_repo(&repo).await
            }));
        }
        
        let mut results = Vec::new();
        for handle in handles {
            if let Ok(Some(info)) = handle.await {
                results.push(info);
            }
        }
        results
    }
}
```

---

## `CancellationToken`: The Clean Way to Signal Shutdown

`tokio_util::sync::CancellationToken` is the idiomatic tool for propagating cancellation signals through a tree of tasks. It's simpler than a `watch::channel` for pure "stop now" semantics:

```rust
use tokio_util::sync::CancellationToken;

async fn run(cancel: CancellationToken) {
    tokio::select! {
        _ = cancel.cancelled() => {
            println!("Cancelled!");
        }
        _ = do_work() => {
            println!("Work completed normally");
        }
    }
}

// A token can be cloned — clones share the same cancellation state
let token = CancellationToken::new();

let child = token.child_token();  // cancelling parent also cancels child
let handle = tokio::spawn(run(child));

// Cancel everything
token.cancel();  // triggers cancelled() on token AND all child tokens
handle.await.unwrap();
```

The tree structure lets you cancel a whole subtree of tasks without tracking each one individually:

```
token (root)
├── cancel token A → spawned task A
├── cancel token B → spawned task B
│   └── cancel token B1 → sub-task of B
└── cancel token C → spawned task C

token.cancel() → all of A, B, B1, C see cancelled()
```

### `CancellationToken` vs `watch::channel`

| | `CancellationToken` | `watch::channel(bool)` |
|---|---|---|
| **Use for** | "Stop now" signals | "Current state" signals |
| **Value** | Binary: not-cancelled / cancelled | Any T |
| **Can set** | Only once (cancel) | Multiple times |
| **Best for** | Shutdown propagation | Config updates, health flags |

Use `CancellationToken` for shutdown. Use `watch` when you need to communicate values over time (not just stop).

---

## `biased` `select!`: Explicit Priorities

By default, `tokio::select!` randomly picks which branch to poll first when multiple are ready. This avoids starvation but can be unpredictable. The `biased` keyword makes priority explicit:

```rust
async fn priority_select(
    mut high_priority: mpsc::Receiver<Event>,
    mut low_priority: mpsc::Receiver<Event>,
    mut shutdown: watch::Receiver<bool>,
) {
    loop {
        tokio::select! {
            biased;  // Arms checked in declaration order
            
            // 1st priority: shutdown — always check this first
            _ = shutdown.changed() => {
                if *shutdown.borrow() { break; }
            }
            
            // 2nd priority: high-priority events
            Some(event) = high_priority.recv() => {
                handle_high_priority(event).await;
            }
            
            // 3rd priority: low-priority events (only if nothing else is ready)
            Some(event) = low_priority.recv() => {
                handle_low_priority(event).await;
            }
        }
    }
}
```

Without `biased`, if `high_priority` and `low_priority` are both ready constantly, `low_priority` gets 50% of CPU. With `biased`, high-priority events always drain before low-priority ones are processed.

**Warning**: `biased` can cause starvation. If `high_priority` produces events faster than you process them, `low_priority` never runs. Use `biased` when you have genuine priority requirements, not as a default.

---

## The `select!` Branch Cancellation Contract

Every branch in a `select!` that doesn't win is cancelled. This has implications:

```rust
let mut buffer = Vec::new();

loop {
    tokio::select! {
        // ⚠️ If this branch wins, recv() is in the middle of building its state
        // It's OK because recv() is cancellation-safe
        Some(data) = socket.read_buf(&mut buffer) => {
            process(&buffer).await;
            buffer.clear();
        }
        _ = timeout => break,
    }
}
```

`socket.read_buf` modifies `buffer` as it reads. If cancelled mid-read, `buffer` might contain partial data from a read that never completed. This is why `read_buf` is not cancellation safe — the buffer's contents are indeterminate after cancellation.

The safe pattern: use `tokio::io::ReadBuf` at the proper boundary level, or use higher-level primitives like `BufReader::read_line()` which are designed to be cancellation safe:

```rust
use tokio::io::{AsyncBufReadExt, BufReader};

let mut reader = BufReader::new(socket);
let mut line = String::new();

loop {
    tokio::select! {
        // read_line is cancellation-safe: if dropped mid-read, the partial
        // data stays in BufReader's internal buffer and will be re-read
        result = reader.read_line(&mut line) => {
            match result {
                Ok(0) => break,  // EOF
                Ok(_) => {
                    process_line(&line).await;
                    line.clear();
                }
                Err(e) => { eprintln!("Error: {e}"); break; }
            }
        }
        _ = shutdown.changed() => break,
    }
}
```

---

## How It Breaks

**Dropping futures in a `JoinSet` mid-way.**
`tokio::task::JoinSet` collects spawned tasks. When the `JoinSet` is dropped, all tasks are cancelled. If your tasks had half-written to a database, those writes are lost. The fix: `join_set.abort_all()` followed by draining completed tasks, or use explicit cancellation tokens so tasks can clean up.

**Assuming timeout means "try again later."**
```rust
// This LOSES the partial response on timeout
let result = timeout(Duration::from_secs(5), fetch(url)).await;
if result.is_err() {
    // The TCP connection is closed, partial response discarded
    // You can retry, but you're starting fresh
    retry_fetch(url).await;
}
```
After timeout, the future is dropped and the TCP connection is closed. Any partial response body is gone. Retry logic must restart from scratch. If the server has already begun processing the request, it may process it twice. Use idempotent operations or implement proper retry logic with deduplication.

**Using `std::sync::Mutex` across a `select!` branch.**
If you hold a `std::sync::Mutex` lock inside a branch and the branch is cancelled, the lock guard drops — the mutex unlocks. That's fine. But if you use a `std::sync::Mutex` across a `.await` inside a branch, the branch is `!Send` and won't compile on a multi-threaded runtime. Use `tokio::sync::Mutex` for locks held across `.await` points.

**Writing to a channel inside a `select!` branch — infinite loop risk.**
```rust
// Subtle bug: if send() is the future that LOSES the race, the message is lost
tokio::select! {
    _ = tx.send(important_data) => {}  // If this loses, data is GONE
    _ = timeout => {}
}

// Safe: collect what needs sending, then send outside select!
let to_send = compute_data();
tokio::select! {
    _ = some_event => {}
    _ = timeout => {}
}
// Now send outside the select — no cancellation risk
tx.send(to_send).await?;
```
