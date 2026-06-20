# Doc 09 - Rust Idioms and Best Practices

🟢 The section 06 idioms are where async, systems, and type discipline intersect.

The music queue is the most complex project yet: it has async WebSocket connections, Redis message streams, JWT authentication, connection pooling, graceful shutdown, rate limiting, and now single-use type guarantees. This doc names the patterns that made it work - the idioms that experienced async Rust developers apply instinctively.

---

## Never Block the Async Thread

The single most important rule in async Rust: **never call a blocking operation on an async task without moving it off the async runtime's thread pool.**

The async runtime expects every task to yield back to the executor quickly (within microseconds). If a task blocks (sleeping, waiting on I/O with sync primitives, doing heavy computation), it starves every other task on that worker thread:

```rust
// WRONG - blocks the Tokio thread, starves all other tasks on this worker
async fn hash_password(password: &str) -> String {
    // bcrypt is intentionally slow (that's the point) - 100-300ms of CPU
    bcrypt::hash(password, 12).unwrap()
}

// CORRECT - moves the blocking work to a dedicated thread pool
async fn hash_password(password: &str) -> Result<String, Error> {
    let password = password.to_owned();
    tokio::task::spawn_blocking(move || {
        bcrypt::hash(&password, 12)
            .map_err(|e| Error::Hash(e.to_string()))
    })
    .await
    .map_err(|e| Error::TaskJoin(e.to_string()))?
}
```

**`tokio::task::spawn_blocking`** moves the closure to a dedicated thread pool for blocking operations. The async task immediately yields to the executor while the blocking work runs on a separate thread. Results come back through the returned `JoinHandle`.

Use `spawn_blocking` for:
- Password hashing (bcrypt, argon2, scrypt)
- CPU-intensive computation (audio processing, image resizing, compression)
- Synchronous file I/O in contexts where `tokio::fs` isn't sufficient
- Any third-party library that does blocking I/O internally

---

## Use `Arc` for Shared State, Not Global Variables

The music queue needs multiple tasks to share the connection pool, Redis client, and queue state. The idiomatic approach is `Arc<T>` with clones passed into each task:

```rust
use std::sync::Arc;

// Instead of globals:
// static REDIS_CLIENT: Mutex<Redis> = ...  ← avoid this

// Use Arc with shared state
#[derive(Clone)]
pub struct AppState {
    pub db_pool: Arc<sqlx::SqlitePool>,
    pub redis: Arc<tokio::sync::Mutex<redis::aio::Connection>>,
    pub queue_broadcast: Arc<tokio::sync::broadcast::Sender<QueueEvent>>,
}

impl AppState {
    pub async fn new(config: &Config) -> Result<Self, Error> {
        let pool = sqlx::SqlitePool::connect(&config.database_url).await?;
        let redis_conn = redis::Client::open(config.redis_url.as_str())?
            .get_async_connection().await?;
        let (broadcast_tx, _) = tokio::sync::broadcast::channel(1000);

        Ok(AppState {
            db_pool: Arc::new(pool),
            redis: Arc::new(tokio::sync::Mutex::new(redis_conn)),
            queue_broadcast: Arc::new(broadcast_tx),
        })
    }
}

// Each task gets a clone of AppState - cheap (just clones the Arc pointers)
async fn start_all_workers(state: AppState, shutdown: watch::Receiver<bool>) {
    let state2 = state.clone();
    let state3 = state.clone();

    tokio::spawn(run_queue_processor(state, shutdown.clone()));
    tokio::spawn(run_websocket_server(state2, shutdown.clone()));
    tokio::spawn(run_metadata_fetcher(state3, shutdown));
}
```

The `#[derive(Clone)]` on `AppState` derives `Clone` automatically because every field is `Arc<T>`. Cloning `AppState` is cheap - it just increments reference counts on the inner `Arc`s.

---

## Choose the Right Channel for the Job

Four channel types, four different shapes of communication:

```rust
use tokio::sync::{mpsc, oneshot, broadcast, watch};

// mpsc: one task sending work to one worker
// Pattern: producer → worker queue
let (task_tx, mut task_rx) = mpsc::channel::<MetadataFetchTask>(100);
tokio::spawn(async move {
    while let Some(task) = task_rx.recv().await {
        fetch_and_store_metadata(task).await;
    }
});

// oneshot: request-response pattern
// Pattern: ask a task for a single result and await it
let (tx, rx) = oneshot::channel::<QueueState>();
status_request_tx.send(tx).await?;
let state = rx.await?;

// broadcast: one value goes to every subscriber
// Pattern: queue events that all connected WebSocket clients should see
let (event_tx, _) = broadcast::channel::<QueueEvent>(1000);
// Each WebSocket connection subscribes:
let mut event_rx = event_tx.subscribe();
// When a track changes:
event_tx.send(QueueEvent::TrackChanged { track_id: 42 })?;
// Every subscriber gets a copy

// watch: current value with change notification
// Pattern: configuration that needs to be read by many tasks, updated rarely
let (config_tx, config_rx) = watch::channel(AppConfig::default());
// Multiple readers:
let current_config = config_rx.borrow().clone();
// One writer:
config_tx.send(new_config)?;
```

**Choosing:**
- `mpsc` when one producer sends to one consumer
- `oneshot` for request-reply (send one thing, get one thing back)
- `broadcast` when every subscriber needs every message (event fan-out)
- `watch` when you want "the current value" (only the latest, not history)

---

## Use `tokio::select!` at Logical Breakpoints Only

`tokio::select!` is powerful but has subtle cancellation behavior (doc 08 of section 04 covers this). Apply it at clean logical boundaries:

```rust
// GOOD: select! at the top of the event loop - all state is consistent here
loop {
    tokio::select! {
        biased;  // Check shutdown first - highest priority

        _ = shutdown.changed() => {
            if *shutdown.borrow() { break; }
        }
        Some(msg) = ws_receiver.next() => {
            handle_message(msg).await;
        }
        Some(event) = event_queue.recv() => {
            broadcast_event(event).await;
        }
    }
}

// BAD: select! in the middle of a multi-step operation
async fn update_queue(db: &Db, queue_id: i64, track: Track) {
    let tx = db.begin().await.unwrap();
    
    tokio::select! {
        // If cancelled HERE, tx is rolled back (via Drop), but the
        // caller might not know whether the update happened or not
        result = tx.insert_track(track) => {
            tx.commit().await.unwrap();
        }
        _ = shutdown.changed() => {
            // tx rolls back via Drop - this is correct for the DB
            // but the caller has no way to know the state changed
        }
    }
}
```

Keep `select!` at the outermost event loop, where you're choosing between independent work items. Inside a multi-step operation, complete the operation fully or roll it back explicitly.

---

## `impl Trait` for Async Function Return Types

Returning futures from non-async functions requires naming the future type. Use `impl Future` to avoid the complexity:

```rust
use std::future::Future;

// Instead of:
// fn fetch_track<'a>(&'a self, id: i64) -> Pin<Box<dyn Future<Output = Track> + 'a>>

// Use:
fn fetch_track(&self, id: i64) -> impl Future<Output = Track> + '_ {
    async move {
        self.db.query_track(id).await
    }
}

// Or just make the function async (preferred when possible):
async fn fetch_track(&self, id: i64) -> Track {
    self.db.query_track(id).await
}
```

When you need to store futures in a collection (different concrete future types), use `Pin<Box<dyn Future>>`:

```rust
type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;

// Store heterogeneous futures in a Vec
let tasks: Vec<BoxFuture<'static, ()>> = subscribers
    .iter()
    .map(|sub| -> BoxFuture<'static, ()> {
        let sub = sub.clone();
        Box::pin(async move { sub.send_event(event.clone()).await })
    })
    .collect();

futures::future::join_all(tasks).await;
```

---

## Clone the Minimum, Move What You Can

In async code, closures passed to `tokio::spawn` must be `'static`. This means you can't capture references - you must either own the data or clone it before the spawn:

```rust
// Pattern: clone only what the task needs, before spawning
async fn spawn_track_fetch(
    state: AppState,
    track_ids: Vec<i64>,
    result_tx: mpsc::Sender<TrackMetadata>,
) {
    for track_id in track_ids {
        // Clone only the pieces this task needs, not the entire state
        let client = state.http_client.clone();  // clone the Arc (cheap)
        let tx = result_tx.clone();              // clone the sender (cheap)
        
        tokio::spawn(async move {
            // client and tx are owned by this task - no shared reference
            if let Ok(metadata) = fetch_metadata(&client, track_id).await {
                let _ = tx.send(metadata).await;
            }
        });
    }
}
```

**Clone `Arc`s, not the data behind them.** Cloning an `Arc<T>` is an atomic reference count increment - nanoseconds. Cloning the `T` itself might be expensive (a `Vec`, a connection pool, a large config struct). Always clone the `Arc`, never the underlying data.

---

## Test Async Code With `tokio::test`

Testing async code is simple with the `#[tokio::test]` attribute:

```rust
#[tokio::test]
async fn test_queue_event_processing() {
    // Setup
    let (tx, mut rx) = mpsc::channel::<QueueEvent>(10);
    let processor = QueueProcessor::new(tx);

    // Send a test event
    processor.enqueue(TrackId(42)).await;

    // Verify it was processed
    let event = rx.recv().await.expect("Should receive an event");
    assert_eq!(event, QueueEvent::TrackQueued { track_id: 42 });
}

#[tokio::test]
async fn test_websocket_receives_queue_updates() {
    let state = AppState::new_for_testing().await;
    let (client_ws, server_ws) = mock_websocket_pair();
    
    // Connect to the server-side handler
    let handle = tokio::spawn(handle_websocket(server_ws, state.clone()));
    
    // Trigger a queue update from another task
    state.queue_broadcast.send(QueueEvent::TrackChanged { track_id: 1 });
    
    // Verify the WebSocket client received the update
    let msg = tokio::time::timeout(
        Duration::from_secs(1),
        client_ws.next(),
    ).await
    .expect("Should receive within 1 second")
    .expect("Channel should not close");
    
    assert!(msg.to_text().unwrap().contains("TrackChanged"));
    
    handle.abort();
}
```

For tests that need a real timeout (not infinite wait), always use `tokio::time::timeout`. A hanging test is worse than a failing test - it blocks the entire CI run.

---

## Structured Logging in Async Code

In multi-task async services, `println!` is not enough - you need to know **which task** logged each message. The `tracing` crate handles this:

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

```rust
use tracing::{info, error, instrument, Span};

// #[instrument] automatically logs function entry/exit with all parameters
#[instrument(skip(db, state), fields(queue_id = queue_id))]
async fn process_queue_event(
    event: QueueEvent,
    queue_id: i64,
    db: &sqlx::Pool<sqlx::Sqlite>,
    state: &AppState,
) -> Result<(), Error> {
    info!("Processing event");
    
    match &event {
        QueueEvent::TrackQueued { track_id } => {
            info!(track_id, "Track queued");
            db.add_to_queue(queue_id, *track_id).await?;
        }
        QueueEvent::TrackSkipped => {
            info!("Track skipped");
            db.advance_queue(queue_id).await?;
        }
    }
    
    info!("Event processed successfully");
    Ok(())
}

// In main: initialize the subscriber
fn main() {
    tracing_subscriber::fmt()
        .with_env_filter("music_queue=debug,tower_http=info")
        .init();
    
    // Now every tracing! macro call goes through the subscriber
}
```

With `tracing`, you get:
- Automatic span nesting (task → handler → db query)
- Structured fields (`track_id = 42`) searchable in log aggregation systems
- Filtering by module and level at runtime via `RUST_LOG=music_queue=debug`
- Integration with Jaeger, Zipkin, and other distributed tracing systems

---

## The Music Queue: Idioms Checklist

Before marking the music queue project complete:

- [ ] No blocking operations on async tasks - bcrypt and heavy computation use `spawn_blocking`
- [ ] Shutdown propagated via `watch::channel` - all tasks check it in their event loops
- [ ] Channel types chosen deliberately - `mpsc` for work queues, `broadcast` for WebSocket events, `watch` for config
- [ ] `Arc<T>` for shared state - not globals, not thread-local statics
- [ ] Single-use session tokens enforce one-connection-per-token at compile time
- [ ] Message acknowledgments are single-use - `PendingMessage` moves to `ack()` or `nack()`
- [ ] `#[tokio::test]` for all async tests - with `timeout` wrappers where infinite wait is possible
- [ ] `tracing` for structured logging - not `println!` in async handlers
- [ ] `JoinSet` manages WebSocket connection lifetimes - cleans up on disconnect and on shutdown
- [ ] `tokio::select!` only at top-level event loop boundaries - not inside multi-step operations

These aren't arbitrary rules. Each one exists because the alternative causes a class of bugs: blocking hangs the runtime, missing shutdown makes deployments painful, wrong channel causes data loss, globals make testing hard, and so on. The idioms encode hard-won operational experience into patterns that are easy to check and verify.
