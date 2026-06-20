# Doc 07 - Graceful Shutdown and Task Management

🟡 This is what separates a toy server from a production service.

The music queue runs as a long-lived process: it receives WebSocket connections, streams playback state, fetches metadata from external APIs, and processes queue events. When you stop the process - for a deployment, a crash, or a routine restart - what happens to all of that in-flight work?

Without graceful shutdown: WebSocket clients see abrupt disconnects, partially-committed queue operations are lost, and the next version of the service starts with inconsistent state.

With graceful shutdown: clients receive a proper close frame, in-flight database writes complete, metadata fetches either finish or are discarded cleanly, and the process exits only when all of this is done.

This doc teaches the Tokio primitives for structured task management and graceful shutdown.

---

## The Three Layers of Graceful Shutdown

A production service needs shutdown to work at three levels:

1. **Stop accepting new work** - no new WebSocket connections, no new queue entries
2. **Finish in-flight work** - complete database writes, send close frames to connected clients
3. **Enforce a deadline** - if in-flight work takes too long, force-exit anyway

Skipping any layer causes problems. Skipping layer 1 means new work keeps arriving during shutdown. Skipping layer 2 means corrupted state. Skipping layer 3 means the process hangs forever if something doesn't respond.

---

## Signal Handling: Catching Ctrl+C and SIGTERM

In production, processes are stopped by OS signals: `SIGTERM` from systemd, `SIGKILL` from Kubernetes, `SIGINT` from the terminal. Tokio provides async signal handling:

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    // Start the music queue server
    let server = start_server().await;

    // Wait for either Ctrl+C (SIGINT) or SIGTERM
    tokio::select! {
        _ = signal::ctrl_c() => {
            println!("Ctrl+C received - shutting down...");
        }
        _ = signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("Failed to register SIGTERM handler")
            .recv() => {
            println!("SIGTERM received - shutting down...");
        }
    }

    // Both signals reach the same shutdown path:
    server.shutdown().await;
    println!("Music queue stopped.");
}
```

For cross-platform code (Windows doesn't have SIGTERM), use only `ctrl_c`:

```rust
tokio::select! {
    _ = signal::ctrl_c() => {
        println!("Shutdown signal received");
    }
}
```

---

## `watch::channel`: Broadcasting Shutdown to All Tasks

The cleanest way to propagate "shutdown now" to many concurrent tasks is a `watch::channel`. It broadcasts a value to all receivers, and each receiver can independently check whether shutdown was requested:

```rust
use tokio::sync::watch;

async fn run_music_queue() {
    // Create a shutdown channel. Initial value: false (not shutting down).
    let (shutdown_tx, shutdown_rx) = watch::channel(false);

    // Spawn the queue's worker tasks, each receiving a clone of the receiver
    let queue_handle = tokio::spawn(run_queue_worker(shutdown_rx.clone()));
    let ws_handle = tokio::spawn(run_websocket_broadcaster(shutdown_rx.clone()));
    let metadata_handle = tokio::spawn(run_metadata_fetcher(shutdown_rx.clone()));

    // Wait for shutdown signal
    signal::ctrl_c().await.expect("Failed to listen for Ctrl+C");
    println!("Shutting down music queue...");

    // Broadcast shutdown to all workers
    shutdown_tx.send(true).expect("Failed to send shutdown signal");

    // Wait for workers to finish (with a 30-second deadline)
    let shutdown_deadline = tokio::time::Duration::from_secs(30);
    let _ = tokio::time::timeout(
        shutdown_deadline,
        async {
            let _ = tokio::join!(queue_handle, ws_handle, metadata_handle);
        },
    ).await;

    println!("All workers stopped.");
}
```

Each worker checks for shutdown in its event loop:

```rust
async fn run_queue_worker(mut shutdown: watch::Receiver<bool>) {
    loop {
        tokio::select! {
            // Check for shutdown - highest priority
            _ = shutdown.changed() => {
                if *shutdown.borrow() {
                    println!("Queue worker: stopping");
                    // Finish any in-progress operation before returning
                    break;
                }
            }
            
            // Process the next queue event
            Some(event) = queue_event_stream() => {
                process_queue_event(event).await;
            }
        }
    }
    
    // Cleanup: flush any buffered state to the database
    flush_queue_state().await;
    println!("Queue worker: shutdown complete");
}
```

---

## `JoinSet`: Managing a Dynamic Set of Tasks

The music queue spawns one task per WebSocket connection. The number of connections is dynamic - connections come and go. `JoinSet` manages a set of tasks and lets you collect results as they complete:

```rust
use tokio::task::JoinSet;

async fn run_websocket_server(listener: TcpListener, shutdown: watch::Receiver<bool>) {
    let mut connections: JoinSet<Result<(), Error>> = JoinSet::new();

    loop {
        tokio::select! {
            // Accept new connections
            Ok((stream, addr)) = listener.accept() => {
                println!("New connection: {addr}");
                let shutdown_clone = shutdown.clone();
                connections.spawn(handle_websocket(stream, shutdown_clone));
            }

            // Collect completed connection handlers
            Some(result) = connections.join_next() => {
                match result {
                    Ok(Ok(())) => {},  // clean exit
                    Ok(Err(e)) => eprintln!("Connection error: {e}"),
                    Err(e) if e.is_cancelled() => {},  // task was cancelled
                    Err(e) => eprintln!("Task panicked: {e}"),
                }
            }

            // Shutdown signal
            _ = shutdown.clone().changed() => {
                if *shutdown.borrow() {
                    println!("WebSocket server: stopping, closing {} connections", 
                             connections.len());
                    // Abort all remaining connections
                    connections.abort_all();
                    // Wait for them all to finish
                    while connections.join_next().await.is_some() {}
                    break;
                }
            }
        }
    }
}
```

`JoinSet` automatically cancels remaining tasks when it's dropped. This is the right behavior for graceful shutdown: abort all connections when shutdown is requested, rather than waiting indefinitely.

---

## `tokio_util::task::TaskTracker`: Waiting Without Cancelling

Sometimes you want to wait for all tasks to complete naturally rather than cancelling them. `TaskTracker` from `tokio-util` tracks tasks without aborting:

```toml
[dependencies]
tokio-util = { version = "0.7", features = ["rt"] }
```

```rust
use tokio_util::task::TaskTracker;

async fn run_metadata_service(shutdown: watch::Receiver<bool>) {
    let tracker = TaskTracker::new();

    // Spawn metadata fetch tasks using the tracker
    for track_id in queue.pending_metadata() {
        let mut shutdown_clone = shutdown.clone();
        tracker.spawn(async move {
            tokio::select! {
                result = fetch_metadata(&track_id) => {
                    if let Ok(metadata) = result {
                        store_metadata(metadata).await;
                    }
                }
                _ = shutdown_clone.changed() => {
                    // Shutdown requested - abandon this fetch
                }
            }
        });
    }

    // Wait for shutdown
    let mut shutdown_clone = shutdown.clone();
    shutdown_clone.changed().await.ok();

    // Close the tracker - no new tasks can be spawned
    tracker.close();

    // Wait for all in-flight tasks to complete (they'll exit via their shutdown check)
    tracker.wait().await;
    println!("Metadata service: all tasks finished");
}
```

`TaskTracker` vs `JoinSet`:
- `JoinSet`: owns the tasks, can abort them, collects their results
- `TaskTracker`: observes tasks, waits for them without collecting results, tasks can be spawned from anywhere (not just where the tracker is)

---

## The WebSocket Shutdown Protocol

WebSocket connections have a proper close handshake. The music queue should send close frames rather than just dropping connections:

```rust
use tokio_tungstenite::tungstenite::Message;

async fn handle_websocket(
    stream: TcpStream,
    mut shutdown: watch::Receiver<bool>,
) -> Result<(), Error> {
    let mut ws = tokio_tungstenite::accept_async(stream).await?;

    loop {
        tokio::select! {
            // Incoming client messages
            Some(msg) = ws.next() => {
                match msg? {
                    Message::Text(text) => handle_queue_command(&text).await?,
                    Message::Close(_) => {
                        // Client initiated close - respond with close frame
                        ws.close(None).await?;
                        break;
                    }
                    _ => {},
                }
            }

            // Shutdown signal - send close frame to client
            _ = shutdown.changed() => {
                if *shutdown.borrow() {
                    // Tell the client we're closing
                    let close_frame = tokio_tungstenite::tungstenite::protocol::CloseFrame {
                        code: tokio_tungstenite::tungstenite::protocol::frame::coding::CloseCode::Away,
                        reason: "Server shutting down".into(),
                    };
                    ws.close(Some(close_frame)).await?;
                    break;
                }
            }
        }
    }

    Ok(())
}
```

---

## Backpressure: Bounded Channels Prevent Memory Exhaustion

The music queue has a task that processes incoming events and a task that handles WebSocket broadcasts. If events arrive faster than broadcasts can be sent, you need backpressure:

```rust
use tokio::sync::mpsc;

async fn queue_event_pipeline() {
    // Bounded channel - if the broadcaster falls behind, this channel fills up
    // and the event producer naturally slows down (send() will await until space)
    let (tx, mut rx) = mpsc::channel::<QueueEvent>(100);

    // Producer: generates queue events
    let producer = tokio::spawn(async move {
        let mut event_source = connect_to_event_source().await;
        while let Some(event) = event_source.next().await {
            // If channel is full, this await will block until space is available.
            // This is natural backpressure - the producer slows to match consumer speed.
            if tx.send(event).await.is_err() {
                break;  // receiver dropped - shutdown in progress
            }
        }
    });

    // Consumer: broadcasts to WebSocket clients
    let broadcaster = tokio::spawn(async move {
        while let Some(event) = rx.recv().await {
            broadcast_to_clients(event).await;
        }
    });

    let _ = tokio::join!(producer, broadcaster);
}
```

**Why bounded channels?** Unbounded channels accept messages without any backpressure. If your producer is faster than your consumer, an unbounded channel will grow until you run out of memory. A bounded channel with `capacity = 100` means the producer blocks when the buffer is full - this automatically matches the producer's rate to the consumer's processing rate.

For the music queue, a capacity of 100-1000 events is reasonable. If your queue processes events in 10ms bursts and events arrive at 100/s, a buffer of 1000 gives you a 10-second cushion before backpressure kicks in.

---

## Retry with Exponential Backoff

The music queue fetches track metadata from Spotify. When Spotify rate-limits you or has a transient error, you need to retry with backoff:

```rust
use tokio::time::{sleep, Duration};

pub async fn fetch_with_retry<F, Fut, T, E>(
    max_attempts: usize,
    base_delay: Duration,
    operation: F,
) -> Result<T, E>
where
    F: Fn() -> Fut,
    Fut: Future<Output = Result<T, E>>,
    E: std::fmt::Debug,
{
    let mut last_error;
    let mut delay = base_delay;

    for attempt in 0..max_attempts {
        match operation().await {
            Ok(value) => return Ok(value),
            Err(e) => {
                last_error = e;
                if attempt + 1 < max_attempts {
                    eprintln!("Attempt {}/{} failed: {:?}. Retrying in {:?}...",
                             attempt + 1, max_attempts, last_error, delay);
                    sleep(delay).await;
                    // Exponential backoff: double the delay each time, cap at 60s
                    delay = (delay * 2).min(Duration::from_secs(60));
                }
            }
        }
    }

    Err(last_error)
}

// Using it in the metadata fetcher:
async fn fetch_track_metadata(track_id: &str) -> Result<TrackMetadata, SpotifyError> {
    fetch_with_retry(
        3,
        Duration::from_millis(500),
        || spotify_api_call(track_id),
    ).await
}
```

**Exponential backoff** is the industry standard for retrying external API calls. Linear backoff (retry every N seconds) thundering-herd problem: if 1000 clients all get errors at the same time and retry at the same interval, they all hit the server together, causing another error. Exponential backoff spreads the retries out, reducing load on the recovering server.

---

## Putting It All Together: The Music Queue Shutdown Sequence

```rust
use tokio::sync::watch;
use tokio::task::JoinSet;
use tokio::time::{timeout, Duration};

pub struct MusicQueueServer {
    // Internal components
}

impl MusicQueueServer {
    pub async fn run(self) {
        let (shutdown_tx, shutdown_rx) = watch::channel(false);
        let mut tasks: JoinSet<()> = JoinSet::new();

        // Spawn all service tasks
        tasks.spawn(self.run_queue_processor(shutdown_rx.clone()));
        tasks.spawn(self.run_websocket_server(shutdown_rx.clone()));
        tasks.spawn(self.run_metadata_fetcher(shutdown_rx.clone()));
        tasks.spawn(self.run_health_check_server(shutdown_rx.clone()));

        // Wait for shutdown signal
        signal::ctrl_c().await.expect("Signal handler failed");
        println!("[1/3] Signal received - stopping new connections");

        // Broadcast shutdown to all tasks
        shutdown_tx.send(true).ok();
        println!("[2/3] Shutdown broadcast sent - waiting for tasks to finish");

        // Wait up to 30 seconds for tasks to finish
        match timeout(Duration::from_secs(30), async {
            while tasks.join_next().await.is_some() {}
        }).await {
            Ok(_) => println!("[3/3] All tasks finished cleanly"),
            Err(_) => {
                eprintln!("[3/3] Shutdown timed out - aborting remaining tasks");
                tasks.abort_all();
            }
        }

        println!("Music queue stopped.");
    }
}
```

---

## How It Breaks

**Forgetting to drop the shutdown sender when the receiver should see channel closed.**
If you hold `shutdown_tx` alive forever, `shutdown_rx.changed()` will never error out. A task that waits for the channel to be closed (using `recv()` returning `None` or `changed()` returning `Err`) will hang.

**Race condition: checking shutdown AFTER the last `select!` iteration.**
```rust
// BUG: shutdown check only happens in the select!, not after the loop exits
loop {
    tokio::select! {
        Some(event) = rx.recv() => process(event).await,
        _ = shutdown.changed() => break,
    }
    // After the event is processed here, we loop back to select!
    // This is actually correct - but many beginners put cleanup code INSIDE
    // the event arm that should run AFTER the loop
}
// Cleanup goes here, outside the loop
cleanup().await;
```

**Not handling the `JoinHandle`'s `Err(JoinError)` for panics.**
When a spawned task panics, `join().await` returns `Err(JoinError)`. If you ignore this (using `let _ = handle.await`), the panic is silently swallowed. In a production service, a panicking task should be logged and potentially trigger a restart:

```rust
match handle.await {
    Ok(()) => {},
    Err(e) if e.is_panic() => {
        eprintln!("Task panicked: {e}");
        // Decide: continue running, or initiate shutdown?
    }
    Err(e) if e.is_cancelled() => {
        // Task was explicitly cancelled - expected during shutdown
    }
    Err(e) => eprintln!("Task error: {e}"),
}
```
