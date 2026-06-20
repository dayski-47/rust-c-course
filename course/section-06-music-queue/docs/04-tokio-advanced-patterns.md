# 04 — Tokio Advanced Patterns 🔴

You know `tokio::spawn` and `.await`. This chapter covers the patterns that separate a working prototype from a production service: coordinated shutdown, concurrent futures, dynamic task sets, cooperative cancellation, and the actor model. `queuemaster` uses all of these.

## tokio::select! — Take the First Result

`select!` runs multiple futures concurrently and returns as soon as one of them completes. The others are cancelled (dropped). This is the foundational pattern for timeouts, shutdown signals, and any situation where you're waiting for "whichever happens first."

```rust
use tokio::time::{sleep, Duration};

async fn with_timeout() {
    tokio::select! {
        result = fetch_data() => {
            println!("Got data: {:?}", result);
        }
        _ = sleep(Duration::from_secs(5)) => {
            println!("Timed out after 5 seconds");
        }
    }
    // fetch_data() is cancelled if the timeout fires first
}
```

`select!` is also the heart of the shutdown pattern. Each long-running task watches a shutdown signal alongside its main work:

```rust
async fn queue_processor(
    mut queue_rx: tokio::sync::mpsc::Receiver<QueueAction>,
    mut shutdown: tokio::sync::watch::Receiver<bool>,
) {
    loop {
        tokio::select! {
            action = queue_rx.recv() => {
                match action {
                    Some(a) => process_action(a).await,
                    None => break, // channel closed
                }
            }
            _ = shutdown.changed() => {
                if *shutdown.borrow() {
                    println!("Queue processor shutting down");
                    break;
                }
            }
        }
    }
    // Flush any remaining state before exiting
}
```

One subtlety: by default, `select!` randomly picks which branch to check when multiple are ready. This prevents starvation. If you need a specific priority order, add `biased;` at the top of the macro — it will always check branches in order from top to bottom.

## tokio::join! — Wait for All

`join!` runs multiple futures concurrently and waits for all of them. Unlike spawning tasks, these futures run on the same task — but they interleave at `.await` points.

```rust
async fn initialize_services(state: &AppState) {
    // These run concurrently, not sequentially
    let (redis_ok, db_ok, cache_warmed) = tokio::join!(
        ping_redis(&state.redis),
        ping_database(&state.db),
        warm_song_cache(&state.redis, &state.db),
    );

    // All three must complete before we proceed
    assert!(redis_ok && db_ok && cache_warmed);
}
```

The difference from `select!`: join waits for everything. Use it when all results are required. Use `select!` when you want the first result or need to race against a timeout.

For a dynamic number of futures (not known at compile time), use `JoinSet`:

```rust
use tokio::task::JoinSet;

async fn broadcast_to_all_rooms(rooms: Vec<String>, message: String) {
    let mut set = JoinSet::new();

    for room_id in rooms {
        let msg = message.clone();
        set.spawn(async move {
            send_to_room(&room_id, &msg).await
        });
    }

    // Collect results as they complete (not in order)
    while let Some(result) = set.join_next().await {
        match result {
            Ok(Ok(())) => {}
            Ok(Err(e)) => eprintln!("Room send failed: {}", e),
            Err(e) => eprintln!("Task panicked: {}", e),
        }
    }
}
```

When you drop a `JoinSet`, all its tasks are cancelled. This is useful for scoping a batch of work — when the `JoinSet` goes out of scope, cleanup happens automatically.

## CancellationToken — Cooperative Shutdown 🔴

`tokio::task::JoinHandle::abort()` cancels a task immediately, dropping it at the next `.await` point. This is brutal — the task has no chance to clean up. For graceful shutdown, you want cooperative cancellation: you signal the task to stop, and the task cleans up and exits on its own.

`CancellationToken` from `tokio-util` is the standard way to do this.

```toml
tokio-util = "0.7"
```

```rust
use tokio_util::sync::CancellationToken;

// Create a token — when cancelled, all clones see it
let token = CancellationToken::new();

// Clone before moving into tasks
let token_a = token.clone();
let token_b = token.clone();

tokio::spawn(async move {
    loop {
        tokio::select! {
            _ = token_a.cancelled() => {
                // Cleanup: flush buffer, close connection, etc.
                println!("Task A: cleaning up and exiting");
                break;
            }
            result = do_work() => {
                // Normal operation
            }
        }
    }
});

tokio::spawn(async move {
    tokio::select! {
        _ = token_b.cancelled() => {
            println!("Task B: exiting");
        }
        _ = long_running_task() => {
            println!("Task B: completed normally");
        }
    }
});

// When you cancel the token, all tasks using any clone see it
tokio::signal::ctrl_c().await.unwrap();
token.cancel(); // Every clone is notified
```

For `queuemaster`, you'll cancel the token on Ctrl+C, and every WebSocket handler, every room coordinator, and the Redis subscriber all exit cleanly.

## Graceful Shutdown Pattern

The full graceful shutdown sequence:

```rust
use tokio_util::sync::CancellationToken;
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let cancel = CancellationToken::new();
    let mut tasks = JoinSet::new();

    // Spawn all background tasks, passing a clone of the token
    tasks.spawn(run_websocket_server(cancel.clone(), state.clone()));
    tasks.spawn(run_redis_subscriber(cancel.clone(), state.clone()));
    tasks.spawn(run_queue_coordinator(cancel.clone(), state.clone()));

    // Wait for Ctrl+C
    tokio::signal::ctrl_c().await.expect("Failed to install Ctrl+C handler");
    println!("Shutdown signal received");

    // Signal all tasks to stop
    cancel.cancel();

    // Give tasks 30 seconds to finish cleanly
    let shutdown_deadline = tokio::time::sleep(std::time::Duration::from_secs(30));
    tokio::select! {
        _ = async {
            while let Some(result) = tasks.join_next().await {
                if let Err(e) = result {
                    eprintln!("Task error during shutdown: {}", e);
                }
            }
        } => {
            println!("All tasks shut down cleanly");
        }
        _ = shutdown_deadline => {
            eprintln!("Shutdown timed out — forcing exit");
            tasks.abort_all();
        }
    }
}
```

## The Actor Pattern

An actor is a task that owns mutable state and processes messages one at a time through a channel. Because only one task touches the state, there's no mutex needed. It's the async equivalent of a single-threaded state machine.

This pattern works very well for the room coordinator in `queuemaster`. The room owns the queue state; WebSocket handler tasks send it actions.

```rust
use tokio::sync::{mpsc, oneshot};
use serde_json::Value;

// Commands the actor understands
enum RoomCommand {
    AddSong {
        song_id: i64,
        username: String,
    },
    Skip {
        reply: oneshot::Sender<Option<Song>>, // reply with what's now playing
    },
    GetQueue {
        reply: oneshot::Sender<Vec<Song>>,
    },
    Subscribe {
        client_id: uuid::Uuid,
        sender: mpsc::Sender<String>,
    },
    Unsubscribe {
        client_id: uuid::Uuid,
    },
}

// The actor task — owns all room state
async fn room_actor(
    room_id: String,
    mut commands: mpsc::Receiver<RoomCommand>,
    redis: deadpool_redis::Pool,
    cancel: CancellationToken,
) {
    let mut clients: std::collections::HashMap<uuid::Uuid, mpsc::Sender<String>> =
        std::collections::HashMap::new();

    loop {
        tokio::select! {
            _ = cancel.cancelled() => break,
            cmd = commands.recv() => {
                let Some(cmd) = cmd else { break };
                match cmd {
                    RoomCommand::AddSong { song_id, username } => {
                        // Update Redis list
                        // Broadcast update to all clients
                        let event = serde_json::json!({
                            "event": "queue_updated",
                            "added_by": username,
                            "song_id": song_id,
                        }).to_string();
                        for sender in clients.values() {
                            let _ = sender.try_send(event.clone());
                        }
                    }
                    RoomCommand::Skip { reply } => {
                        // Pop from Redis, send back new "now playing"
                        let _ = reply.send(None); // placeholder
                    }
                    RoomCommand::GetQueue { reply } => {
                        let _ = reply.send(vec![]); // placeholder
                    }
                    RoomCommand::Subscribe { client_id, sender } => {
                        clients.insert(client_id, sender);
                    }
                    RoomCommand::Unsubscribe { client_id } => {
                        clients.remove(&client_id);
                    }
                }
            }
        }
    }
}

// Handle: cheap to clone, hides the channel
#[derive(Clone)]
pub struct RoomHandle {
    tx: mpsc::Sender<RoomCommand>,
}

impl RoomHandle {
    pub fn spawn(room_id: String, redis: deadpool_redis::Pool, cancel: CancellationToken) -> Self {
        let (tx, rx) = mpsc::channel(256);
        tokio::spawn(room_actor(room_id, rx, redis, cancel));
        RoomHandle { tx }
    }

    pub async fn add_song(&self, song_id: i64, username: String) {
        let _ = self.tx.send(RoomCommand::AddSong { song_id, username }).await;
    }

    pub async fn get_queue(&self) -> Vec<Song> {
        let (reply_tx, reply_rx) = oneshot::channel();
        let _ = self.tx.send(RoomCommand::GetQueue { reply: reply_tx }).await;
        reply_rx.await.unwrap_or_default()
    }
}
```

The `oneshot` channel is how the actor sends replies back to callers. The caller creates the channel, sends one end with the request, and `.await`s the other end. The actor sends the response and the caller wakes up.

## spawn_blocking — Calling Blocking Code

Some APIs are unavoidably synchronous: CPU-intensive computation, C libraries, file I/O from system calls that don't have async wrappers, password hashing. If you call these directly in an async task, you block the Tokio worker thread and starve other tasks.

`spawn_blocking` runs the closure on a separate thread pool dedicated to blocking work. The async task `.await`s the result without blocking the executor.

```rust
pub async fn hash_password_async(password: String) -> anyhow::Result<String> {
    // argon2 is intentionally slow (100ms+). Never call it directly in async.
    tokio::task::spawn_blocking(move || {
        hash_password(&password)
    })
    .await
    .map_err(|e| anyhow::anyhow!("Thread panicked: {}", e))?
}

pub async fn process_audio_file(path: std::path::PathBuf) -> anyhow::Result<AudioMetadata> {
    // ID3 tag parsing is synchronous
    tokio::task::spawn_blocking(move || {
        read_audio_metadata(&path)
    })
    .await?
}
```

Tokio's blocking thread pool defaults to 512 threads and grows dynamically. It's separate from the async worker thread pool. Don't worry about hitting the limit unless you're spawning thousands of long-running blocking tasks.

## Retry with Exponential Backoff

When a network call fails, don't retry immediately. Wait a bit, then wait longer, then give up.

```rust
pub async fn retry_with_backoff<F, Fut, T, E>(
    operation: F,
    max_retries: u32,
) -> Result<T, E>
where
    F: Fn() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
    E: std::fmt::Display,
{
    let mut delay = std::time::Duration::from_millis(100);

    for attempt in 0..=max_retries {
        match operation().await {
            Ok(val) => return Ok(val),
            Err(e) if attempt == max_retries => {
                eprintln!("All {} retries exhausted: {}", max_retries, e);
                return Err(e);
            }
            Err(e) => {
                eprintln!("Attempt {} failed: {}. Retrying in {:?}", attempt + 1, e, delay);
                tokio::time::sleep(delay).await;
                // Add some jitter to avoid thundering herd
                delay = delay * 2;
                if delay > std::time::Duration::from_secs(30) {
                    delay = std::time::Duration::from_secs(30);
                }
            }
        }
    }
    unreachable!()
}
```

## How It Breaks

- `CancellationToken` not propagated: you create a token, but forget to pass it to every spawned task. Some tasks keep running after shutdown.
- `select!` with unbalanced arms: if one arm's future is much faster than others and you're in a loop, the fast arm dominates and the slow arms starve.
- Graceful shutdown that hangs: shutdown signal received, you wait for all tasks to finish, but one task is stuck in a blocking operation. Set a timeout on the shutdown wait.

## Common Mistakes

**Using `select!` for sequential work.** If you have two futures where the second depends on the first, don't put them in `select!`. That runs them concurrently and cancels one. Use `join!` or await them in sequence.

**Forgetting that `select!` cancels the losing branch.** When the timeout fires, `fetch_data()` is dropped mid-execution. If `fetch_data` was in the middle of a database write, that write is cancelled. Prefer `select!` for reads and idempotent operations. For writes, use `CancellationToken` so the task can complete its current operation before exiting.

**Sending to an actor and ignoring the error.** If the actor task has panicked or exited, the channel is closed. `tx.send(cmd).await` returns `Err`. Always check it. In a web handler, return 500 if the room actor is gone.

**Calling blocking functions without `spawn_blocking`.** This is the most common performance problem. `std::fs::read`, `std::net::TcpStream::connect` (synchronous), CPU-heavy parsing — all will block Tokio worker threads. The symptom is high latency under load even when CPU is not maxed out. Reach for `spawn_blocking` whenever you call anything that doesn't have `async` in its signature.
