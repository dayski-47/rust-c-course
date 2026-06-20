# Doc 06 - Structured Logging with Tracing

🟢 `println!` debugging is what you do once. `tracing` is what you do when the
server is in production, handling 10,000 connections, and you need to understand
what went wrong for connection 4,729 three minutes ago without restarting anything.

This doc covers the `tracing` crate: spans, events, the `#[instrument]` macro, and
how to configure structured JSON output that works with any log aggregation system.

---

## Why Not `log`?

The `log` crate is older and simpler: `info!("message")`. It works but loses context.

With `log`:
```
INFO: Message delivered
INFO: Message delivered
WARN: Client disconnected
INFO: Message delivered
```

You can't tell which connection delivered which message, or which connection
disconnected. Three connections were active. One had a problem. You don't know which.

With `tracing`:
```json
{"timestamp":"2025-11-14T10:23:45Z","level":"INFO","target":"nexus::handler","message":"Message delivered","conn_id":4729,"topic":"alerts","seq":8421,"peer":"192.168.1.5:54321"}
{"timestamp":"2025-11-14T10:23:45Z","level":"INFO","target":"nexus::handler","message":"Message delivered","conn_id":4730,"topic":"metrics","seq":19283,"peer":"10.0.0.3:61234"}
{"timestamp":"2025-11-14T10:23:45Z","level":"WARN","target":"nexus::handler","message":"Client disconnected","conn_id":4729,"peer":"192.168.1.5:54321","error":"connection reset"}
```

You can filter for `conn_id=4729` and see the complete history of that connection.

---

## How Tracing Context Propagates

Understanding how span context works in async Rust is important before using it.

In synchronous code, `tracing` stores the current span in thread-local storage.
Every log call reads that storage and adds the current span's fields to the event.
Entering a span sets the thread-local; leaving it restores the previous value.

In async code, a single task may execute across multiple OS threads (the Tokio
thread pool can move tasks between threads at await points). Thread-local storage
is per-thread, not per-task. If a task's span were stored in thread-local storage,
it would be lost or shared incorrectly when the task is moved to a different thread.

`tracing` solves this with `Instrument`. When you use `#[instrument]` or
`.instrument(span)`, the span is attached to the *future* itself, not to the
thread. The Tokio runtime correctly enters and exits the span each time it polls
the future, regardless of which thread runs it.

This is why:
- `#[instrument]` works correctly on async functions - it's sugar for `.instrument(span)`
- You must use `.instrument(span)` on futures passed to `tokio::spawn` - the spawned
  task will run on a thread pool, and without `.instrument()`, the parent span is lost
- You should NOT call `span.enter()` and hold the guard across an `.await` - this
  holds the span on the thread even while the task is parked, corrupting context for
  other tasks that run on the same thread

## The Core Concepts: Events and Spans

`tracing` has two primitives:

**Events** are point-in-time log entries:
```rust
tracing::info!("Connection accepted");
tracing::warn!(error = ?e, "Frame decode error");
tracing::debug!(topic = "alerts", seq = 42, bytes = 128, "Message delivered");
```

**Spans** are durations - they represent "while this was happening":
```rust
let span = tracing::span!(
    tracing::Level::DEBUG,
    "connection",
    conn_id = conn_id,
    peer = %peer_addr,
);
let _guard = span.enter();
// All events inside this block are tagged with conn_id and peer_addr
```

The key: spans attach context to all events inside them. Events inside a span
automatically carry the span's fields.

---

## Setup: Subscriber Configuration

The subscriber is what actually outputs log data. `tracing_subscriber` provides
several formatters:

```rust
// nexus-server/src/main.rs
use tracing_subscriber::{EnvFilter, fmt};

pub fn init_tracing() {
    // Human-readable format for development
    #[cfg(debug_assertions)]
    {
        fmt()
            .with_env_filter(
                EnvFilter::try_from_default_env()
                    .unwrap_or_else(|_| EnvFilter::new("nexus=debug,tokio=warn,info"))
            )
            .with_target(true)
            .with_thread_ids(false)
            .with_file(false)
            .init();
    }

    // JSON for production
    #[cfg(not(debug_assertions))]
    {
        fmt()
            .json()                    // Output as JSON lines
            .with_env_filter(
                EnvFilter::try_from_default_env()
                    .unwrap_or_else(|_| EnvFilter::new("nexus=info,warn"))
            )
            .with_current_span(true)   // Include span context in each log line
            .flatten_event(true)       // Flatten span fields into the event JSON
            .init();
    }
}
```

**`RUST_LOG` environment variable**: `EnvFilter::try_from_default_env()` reads
`RUST_LOG` to set log levels at runtime without recompiling:

```bash
# All nexus output at debug, everything else at warn
RUST_LOG=nexus=debug,warn ./nexus-server

# Specific module at trace
RUST_LOG=nexus::handler=trace,nexus=debug,warn ./nexus-server
```

This is the standard pattern for Rust servers. No configuration file needed for
log level control.

---

## Cargo.toml Setup

```toml
# nexus-core/Cargo.toml
[dependencies]
tracing = "0.1"    # The logging API (no output, just the macros)

# nexus-server/Cargo.toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

The `tracing` crate contains the macros and span types. The `tracing-subscriber`
crate contains the output implementations. Libraries depend only on `tracing`;
applications configure the subscriber.

This separation means: nexus-core can log with `tracing::info!()` without knowing
or caring how the output is formatted. The nexus-server binary decides the output
format. Users of nexus-core can use their own subscriber.

---

## Per-Connection Spans with `#[instrument]`

The `#[instrument]` macro wraps an `async fn` in a span that starts when the
function is called and ends when it returns:

```rust
// nexus-server/src/handler.rs
use tracing::{info, debug, warn, error, instrument};

#[instrument(
    skip(stream, engine),              // Don't log these - too large or not useful
    fields(                            // Add these fields to every log inside this span
        conn_id = tracing::field::Empty,   // Will be set after allocation
        peer = %stream.peer_addr().unwrap_or("[unknown]".parse().unwrap()),
    )
)]
pub async fn handle_connection(stream: TcpStream, engine: Arc<NexusEngine>) {
    let conn_id = engine.allocate_connection_id();

    // Set the conn_id field now that we have it
    tracing::Span::current().record("conn_id", conn_id);

    info!("Connection accepted");   // Logged with conn_id and peer already attached

    // ... rest of handler
    
    info!("Connection closed");     // Same conn_id and peer in every log line
}
```

Every `info!`, `debug!`, `warn!` inside `handle_connection` automatically includes
`conn_id` and `peer`. You write the fields once (on the function), not on every log
statement.

**`skip()`**: Use `skip` for arguments that are:
- Large data structures (expensive to format)
- Not useful for debugging (e.g., internal engine handle)
- Sensitive (credentials, personal data)

**`fields()`**: Use `fields` for computed values that should appear in every log
line from this function. The `tracing::field::Empty` marker lets you set the value
later with `.record()`.

---

## Span Hierarchy: Nested Context

Spans nest. The child span inherits context from the parent:

```rust
#[instrument(skip(conn, engine), fields(conn_id = conn.id))]
async fn handle_publish(
    conn: &Connection<Active>,
    engine: &Arc<NexusEngine>,
    frame: &Frame,
) -> Result<u64, NexusError> {
    let (topic, payload) = parse_publish_payload(&frame.payload)?;

    debug!(topic = topic, payload_bytes = payload.len(), "Handling PUBLISH");

    // This inner span adds topic context to all logs inside store_message
    let span = tracing::span!(
        tracing::Level::DEBUG,
        "route",
        topic = topic,
    );
    let seq = async {
        engine.route(&topic, engine.next_message_seq(), payload.clone()).await
    }
    .instrument(span)
    .await?;

    info!(topic = topic, seq = seq, "Message published");
    Ok(seq)
}
```

Log output in JSON (flattened):
```json
{"level":"DEBUG","message":"Handling PUBLISH","conn_id":4729,"topic":"alerts","payload_bytes":47}
{"level":"INFO","message":"Message published","conn_id":4729,"topic":"alerts","seq":8421}
```

Both lines carry `conn_id` from the outer span, `topic` from the inner span.
When you search for `conn_id=4729` you get the complete story of that connection.

---

## Log Levels: When to Use Each

| Level | When | nexus examples |
|-------|------|---------------|
| `ERROR` | Unexpected failure; requires attention | Store corruption, panic recovery |
| `WARN` | Expected-but-unusual condition | Protocol error from client, slow subscriber |
| `INFO` | Normal operation milestones | Connection accepted/closed, server started |
| `DEBUG` | Detailed normal operation | Each frame received, message routed |
| `TRACE` | Per-iteration details | Polling state, partial reads |

```rust
// ERROR: Server-side failures that need investigation
tracing::error!(
    conn_id = conn.id,
    error = ?e,
    "Store write failed - message may be lost"
);

// WARN: Client did something unusual (not a server bug)
tracing::warn!(
    conn_id = conn.id,
    topic = topic,
    payload_bytes = payload.len(),
    limit = engine.config.max_payload_bytes,
    "Payload too large - rejecting publish"
);

// INFO: Normal lifecycle events
tracing::info!(
    conn_id = conn.id,
    peer = %conn.peer_addr,
    "Connection established"
);

// DEBUG: Per-message events (high volume - disable in production)
tracing::debug!(
    conn_id = conn.id,
    command = ?frame.command,
    seq = frame.sequence_id,
    payload_bytes = frame.payload.len(),
    "Frame received"
);
```

Production servers run at INFO. The WARN and ERROR lines are alerts. DEBUG floods
output and is enabled only for investigation.

---

## Span Timers: Measuring Operation Duration

`#[instrument]` automatically records span duration. With `tracing-subscriber`'s
JSON format and `with_span_events(FmtSpan::CLOSE)`, you get the duration of every
instrumented function call:

```toml
# nexus-server/Cargo.toml
[dependencies]
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

```rust
use tracing_subscriber::fmt::format::FmtSpan;

fmt()
    .json()
    .with_span_events(FmtSpan::CLOSE)  // Log when spans close, including duration
    .init();
```

Output:
```json
{"level":"INFO","target":"nexus::handler","message":"close","time_busy_us":1823,"time_idle_us":24891,"conn_id":4729}
```

`time_busy_us` is the microseconds actually executing. `time_idle_us` is time spent
waiting on async operations. For a connection handler, `time_idle_us` dominates -
connections spend most of their time waiting for the next frame.

---

## Async Spans: The `.instrument()` Pattern

`#[instrument]` handles most cases. When you need manual span control inside an
async block (e.g., in a `tokio::spawn`), use `.instrument()`:

```rust
use tracing::Instrument;

// When spawning a task: attach span context so the task's logs are associated
// with the parent context, even though they run on a different thread/task
let span = tracing::span!(
    tracing::Level::DEBUG,
    "subscriber_delivery",
    conn_id = conn.id,
    topic = topic,
);

tokio::spawn(
    async move {
        deliver_messages(msg_rx, sink).await
    }
    .instrument(span)  // The spawned task logs appear inside this span
);
```

Without `.instrument()`, the spawned task has no span context - its logs appear
unattributed. With it, every log from `deliver_messages` carries `conn_id` and
`topic`.

---

## nexus Tracing in Practice

Here's the complete tracing for a successful PUBLISH flow:

```
[SPAN] connection{conn_id=4729,peer="192.168.1.5:54321"}
  INFO  Connection accepted
  [SPAN] handle_connect{conn_id=4729}
    DEBUG CONNECT frame received, validating token
    DEBUG Token valid, transitioning to Active
  INFO  Authentication successful
  [SPAN] handle_publish{conn_id=4729}
    DEBUG Frame received, command=Publish, seq=1, payload_bytes=47
    DEBUG Routing to topic "alerts"
    [SPAN] route{topic="alerts"}
      DEBUG Subscribers for "alerts": 3
      DEBUG Delivered to subscriber conn_id=4712
      DEBUG Delivered to subscriber conn_id=4731
      DEBUG Subscriber conn_id=4745: channel full, dropping
    WARN  1 subscriber could not receive message (channel full)
    INFO  Message published, topic="alerts", seq=8421
  DEBUG Frame received, command=Disconnect, seq=2
  INFO  Connection closed
```

Every log line carries context from all open spans. When you filter for
`conn_id=4729`, you see the complete connection story. When you filter for
`topic="alerts"`, you see all delivery attempts for that topic.

---

## What Not to Log

**Don't log auth tokens, passwords, or personally identifying data:**

```rust
// ❌ Security violation - token appears in logs
tracing::debug!(token = token, "Validating auth token");

// ✅ Log that validation happened, not the token itself
tracing::debug!("Validating auth token");
tracing::debug!(token_length = token.len(), token_prefix = &token[..4.min(token.len())], "Token received");
```

**Don't log full message payloads at high volume:**

```rust
// ❌ 1 MB payloads × 10k messages/sec = 10 GB of logs/sec
tracing::debug!(payload = ?payload, "Message received");

// ✅ Log metadata, not content
tracing::debug!(payload_bytes = payload.len(), "Message received");
```

**Don't log at DEBUG in production by default** - DEBUG events may be high volume
and expensive to format. They exist for investigation, not for always-on monitoring.

---

## Exercises

**Exercise 1 - Per-Connection Span**

Add `#[instrument]` to `handle_connection`. Configure it to:
- Skip the `stream` and `engine` arguments
- Add `conn_id` and `peer` as span fields
- Set `conn_id` using `.record()` after calling `engine.allocate_connection_id()`

Write a test using the `tracing-test` crate that verifies:
- After the connection is handled, the span was closed
- The `conn_id` field appeared in at least one log event

**Exercise 2 - JSON Output**

Add a `--json` CLI flag to `nexus-server`. When passed:
- Initialize the subscriber with `.json()` format
- Add `.flatten_event(true)` so span fields appear in each log line

Test by running the server with `--json` and piping output to `jq`. Verify that:
- Each line is valid JSON
- Lines inside the connection span have a `conn_id` field
- The `level` field is present as a string

**Exercise 3 - Span Timing**

Enable `FmtSpan::CLOSE` on the subscriber. For each of these spans in the connection
lifecycle, observe what `time_busy_us` and `time_idle_us` report and explain why:
- `handle_connection` span (mostly idle - waiting for TCP)
- `handle_publish` span (mostly busy - routing is CPU work)

Add a Criterion benchmark that measures `handle_publish` span duration vs raw routing
duration (without the span). The overhead of `#[instrument]` should be under 100 ns.

---

## Checklist

- [ ] `tracing` in all library crates, `tracing-subscriber` only in the binary
- [ ] `init_tracing()` called at the start of `main`
- [ ] `#[instrument(skip(...))]` on all major async functions
- [ ] Per-connection span with `conn_id` and `peer` fields established at connection start
- [ ] All logs inside connection handler carry `conn_id` automatically via span
- [ ] Log levels: ERROR for server bugs, WARN for client errors, INFO for lifecycle, DEBUG for hot paths
- [ ] No auth tokens, passwords, or full payloads in log output
- [ ] `RUST_LOG` env var controls log level - no recompile needed
- [ ] JSON format enabled in release builds for log aggregation systems
