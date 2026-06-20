# Doc 05 — Error Type Hierarchies and Design

🟢 nexus has four distinct error domains: protocol errors (bad frames), auth errors
(bad tokens), store errors (failed persistence), and transport errors (TCP problems).
Each domain has different callers, different recovery strategies, and different
information that needs to flow up the stack. This doc shows how to design error
types that are useful at every layer.

---

## Why Error Design Matters

In C, error handling is an afterthought: a function returns -1, sets `errno`, and
the caller decides what to do. This works for simple cases but falls apart for
complex systems:

```c
// Caller can't tell WHY it failed from the return value alone
int result = nexus_publish(conn, topic, payload, len);
if (result < 0) {
    // Is it -EACCES (not authenticated)?
    // -ENOENT (topic doesn't exist)?
    // -ENOSPC (store full)?
    // -EIO (network error)?
    log("publish failed: %d", result);  // Information lost
    return result;
}
```

In Rust, error types carry the information callers need to make decisions:

```rust
match conn.publish(&engine, "alerts", payload).await {
    Err(NexusError::Auth(AuthError::NotAuthenticated)) => {
        send_error_frame(ErrorCode::MustConnectFirst).await
    }
    Err(NexusError::Store(StoreError::TopicNotFound(name))) => {
        send_error_frame(ErrorCode::TopicDoesNotExist).await
    }
    Err(NexusError::Transport(_)) => {
        // Can't send an error — the connection is broken
        break;
    }
    Ok(seq) => send_puback(seq).await,
    Err(e) => {
        tracing::error!(?e, "Unexpected error");
        break;
    }
}
```

The error type is part of the function's contract. The caller matches on it and takes
the right action. This is impossible with `errno` or `Box<dyn Error>`.

---

## The Error Type Is an API Contract

The return type `Result<T, E>` is not just "success or failure". The `E` type tells
the *caller* what decisions they can make. If `E` is `Box<dyn Error>`, the caller
can only log and give up — they cannot pattern-match and recover differently for a
"topic not found" vs a "network error". If `E` is a concrete enum, the caller can
handle each case.

This means error types are part of your public API, held to the same standard as
function signatures. Changing an error type's variants is a breaking change for
callers who match on them.

The corollary: **never use `Box<dyn Error>` or `anyhow::Error` as return types in
library code**. Those types are appropriate in binaries (where the error is being
logged and there are no downstream callers). In libraries, callers need to match.

## Layered Error Types

nexus errors form a hierarchy: domain errors at the bottom, a top-level `NexusError`
at the top:

```
NexusError
├── Auth(AuthError)           — authentication failures
│   ├── InvalidToken
│   └── NotAuthenticated
├── Protocol(ProtocolError)   — wire-level protocol violations
│   ├── UnsupportedVersion
│   ├── UnknownCommand
│   └── PayloadTooLarge
├── Store(StoreError)         — persistence failures
│   ├── TopicNotFound
│   └── Io(io::Error)
└── Transport(io::Error)      — TCP / network failures
```

Each domain has its own error type. The top-level type wraps them all.

---

## Implementing the Error Hierarchy

```toml
# nexus-core/Cargo.toml
[dependencies]
thiserror = "2"
```

```rust
// nexus-core/src/error.rs
use thiserror::Error;

// --- Layer 1: Domain Errors ---

#[derive(Debug, Error)]
pub enum AuthError {
    #[error("invalid auth token")]
    InvalidToken,

    #[error("connection must authenticate before sending {0:?}")]
    NotAuthenticated(crate::proto::Command),
}

#[derive(Debug, Error)]
pub enum ProtocolError {
    #[error("unsupported protocol version {got} (expected 1)")]
    UnsupportedVersion { got: u8 },

    #[error("unknown command byte 0x{code:02X}")]
    UnknownCommand { code: u8 },

    #[error("payload too large: {got} bytes (limit: {limit})")]
    PayloadTooLarge { limit: usize, got: usize },

    #[error("frame truncated: expected {expected} bytes, got {got}")]
    Truncated { expected: usize, got: usize },
}

#[derive(Debug, Error)]
pub enum StoreError {
    #[error("topic not found: {0}")]
    TopicNotFound(String),

    #[error("sequence number overflow")]
    SequenceOverflow,

    #[error("store I/O error: {0}")]
    Io(#[from] std::io::Error),
}

// --- Layer 2: Top-Level Error ---

#[derive(Debug, Error)]
pub enum NexusError {
    #[error("auth error: {0}")]
    Auth(#[from] AuthError),

    #[error("protocol error: {0}")]
    Protocol(#[from] ProtocolError),

    #[error("store error: {0}")]
    Store(#[from] StoreError),

    #[error("transport error: {0}")]
    Transport(#[from] std::io::Error),

    #[error("client disconnected")]
    ClientDisconnected,
}
```

The `#[from]` attribute auto-generates `From<AuthError> for NexusError`, etc. The
`?` operator uses these conversions:

```rust
async fn handle_publish(
    conn: &Connection<Active>,
    engine: &NexusEngine,
    frame: Frame,
) -> Result<u64, NexusError> {
    let (topic, payload) = parse_publish_payload(&frame.payload)?;  // ? converts ProtocolError
    let seq = engine.store.store(&topic, frame.sequence_id, payload).await?;  // ? converts StoreError
    Ok(seq)
}
```

Each `?` converts its source error type into `NexusError` via the `#[from]`-generated
impls. The caller sees `NexusError` and can match on its variants.

---

## Error Conversion Chains

A longer chain shows how errors flow up:

```rust
// Low level: io::Error
// A TCP write fails: std::io::Error

// Mid level: NexusError::Transport(io::Error)
// The handler catches this and logs the connection failure

// This works because:
// NexusError has: #[error("transport error: {0}")] Transport(#[from] std::io::Error)
// Which generates: impl From<std::io::Error> for NexusError

// In code:
async fn send_frame(framed: &mut Framed<TcpStream, NexusCodec>, frame: Frame)
    -> Result<(), NexusError>
{
    framed.send(frame).await?;  // io::Error → NexusError::Transport via ?
    Ok(())
}
```

Now consider `StoreError::Io(#[from] std::io::Error)` — this lets `io::Error`
convert to `StoreError` too. When you `?` in a function that returns `NexusError`,
you need the conversion path to be unambiguous:

```rust
// ❌ Ambiguous — does io::Error become StoreError::Io or NexusError::Transport?
async fn broken(engine: &NexusEngine) -> Result<(), NexusError> {
    let file = std::fs::File::open("data")?;  // io::Error → which NexusError variant?
    Ok(())
}

// ✅ Be explicit about which domain an error belongs to
async fn correct(engine: &NexusEngine) -> Result<(), NexusError> {
    let file = std::fs::File::open("data")
        .map_err(|e| NexusError::Transport(e))?;   // Explicit: this is a transport error
    Ok(())
}

// Or: use a StoreError function that does the conversion internally
async fn store_fn() -> Result<(), StoreError> {
    let file = std::fs::File::open("data")?;  // io::Error → StoreError::Io (no ambiguity here)
    Ok(())
}
```

When `From<io::Error>` is implemented for multiple variants, the `?` operator is
ambiguous and fails to compile. The fix: be explicit with `.map_err()`, or structure
your code so the io::Error only appears in one domain's function.

---

## thiserror for Libraries vs anyhow for Applications

| Context | Use |
|---------|-----|
| `nexus-core` (library) | `thiserror` — callers need to match errors |
| `nexus-proto` (library) | `thiserror` — protocol errors must be inspectable |
| `nexus-server` (application) | `anyhow` — startup errors are logged, not matched |

```rust
// nexus-server/src/main.rs — application code
use anyhow::{Context, Result};

#[tokio::main]
async fn main() -> Result<()> {
    let config = load_config("nexus.toml")
        .context("Failed to load configuration")?;

    let engine = NexusEngine::new(config.engine_config)
        .context("Failed to initialize engine")?;

    nexus_server::serve(engine, &config.listen_addr)
        .await
        .with_context(|| format!("Server failed on {}", config.listen_addr))?;

    Ok(())
}
```

`anyhow::Context` adds a human-readable string to any error. This context is
extremely valuable in production:

```
Error: Server failed on 0.0.0.0:9090

Caused by:
    0: Failed to bind TCP listener
    1: Address already in use (os error 98)
```

vs. the library's opaque error:

```
Error: transport error: Address already in use (os error 98)
```

The `anyhow` chain shows what the application was trying to do at each level. The
`thiserror` chain shows what the underlying operation failed on.

---

## Error Logging vs Error Propagation

Not every error should propagate up to the caller. Some errors are "local" —
the caller can't do anything useful with them, and the right action is to log
and continue:

```rust
// nexus-server/src/handler.rs

async fn run_active_loop(
    framed: &mut Framed<TcpStream, NexusCodec>,
    conn: &Connection<Active>,
    engine: &Arc<NexusEngine>,
) {
    loop {
        match framed.next().await {
            Some(Ok(frame)) => {
                let result = dispatch_frame(conn, engine, &frame).await;
                match result {
                    Ok(response) => {
                        if framed.send(response).await.is_err() {
                            // ⬆️ Transport error: can't send, connection is broken
                            // Don't log an error here — this is a normal disconnection
                            break;
                        }
                    }
                    Err(NexusError::Protocol(e)) => {
                        // ⬇️ Protocol error: client sent bad data
                        // Log a warning (client bug, not server bug), send an error frame
                        tracing::warn!(conn_id = conn.id, error = ?e, "Protocol error");
                        let _ = framed.send(Frame::error(frame.sequence_id, e.into())).await;
                        // Continue — one bad frame doesn't kill the connection
                    }
                    Err(NexusError::Store(e)) => {
                        // ⬇️ Store error: server-side problem
                        // Log an error, send an error frame, continue
                        tracing::error!(conn_id = conn.id, error = ?e, "Store error");
                        let _ = framed.send(Frame::error(frame.sequence_id, ErrorCode::ServerError)).await;
                    }
                    Err(NexusError::Transport(e)) => {
                        // ⬆️ Transport error: log and disconnect
                        tracing::debug!(conn_id = conn.id, error = ?e, "Transport error, closing");
                        break;
                    }
                    Err(NexusError::ClientDisconnected) | Err(NexusError::Auth(_)) => {
                        break;
                    }
                }
            }
            Some(Err(e)) => {
                tracing::warn!(conn_id = conn.id, error = ?e, "Frame decode error");
                break;
            }
            None => break,
        }
    }
}
```

The error handling policy:
- **Protocol errors**: log at WARN, send error frame, continue the connection
- **Store errors**: log at ERROR, send error frame, continue the connection
- **Transport errors**: log at DEBUG (normal disconnection), close the connection
- **Auth errors**: close the connection (shouldn't occur in Active state)

---

## Structured Error Context

In production, "connection closed" is useless. "Connection 1847 from 192.168.1.5
closed after protocol error: payload too large (1.2 MB, limit 1 MB)" is actionable.

Add context at the boundary where information is available:

```rust
// Bad: error loses context when propagated up
async fn publish_inner(topic: &str, payload: bytes::Bytes) -> Result<u64, NexusError> {
    if payload.len() > MAX_PAYLOAD {
        return Err(NexusError::Protocol(ProtocolError::PayloadTooLarge {
            limit: MAX_PAYLOAD,
            got: payload.len(),
        }));
    }
    // ...
}

// Good: the caller adds context before logging or returning
tracing::warn!(
    conn_id = conn.id,
    topic = topic,
    payload_bytes = payload.len(),
    limit = MAX_PAYLOAD,
    "Publish rejected: payload too large"
);
```

Error types carry facts (what went wrong). Tracing context carries situation (who
was doing what). The combination is what makes production debugging possible.

---

## Panics vs Errors

The guideline: use `Result` for expected failure modes, `panic!` for programming
errors that should never happen:

```rust
// ✅ Expected failure — return an error
pub async fn authenticate(self, token: &str) -> Result<Connection<Active>, AuthError> {
    if token != self.expected_token {
        return Err(AuthError::InvalidToken);  // Expected: clients can have wrong tokens
    }
    Ok(Connection { /* ... */ })
}

// ✅ Programming error — panic
fn allocate_slot(&mut self) -> usize {
    assert!(self.slots.len() < self.max_slots, "BUG: slot pool exhausted — capacity was validated at construction");
    let slot = self.next_slot;
    self.next_slot += 1;
    slot
}

// ❌ Wrong: using panic for expected failures
pub fn publish(&self, topic: &str) {
    assert!(self.is_subscribed(topic), "Must be subscribed before publishing");  // DON'T
    // Use Result: the caller can't verify subscription without calling subscribe first
}

// ❌ Wrong: using Result for programming errors
pub fn allocate_slot(&mut self) -> Result<usize, SlotError> {
    if self.slots.len() >= self.max_slots {
        return Err(SlotError::PoolExhausted);  // Callers won't handle this — they'll panic anyway
    }
    // ...
}
```

The distinction: can a correct caller trigger this failure? If yes → `Result`. If
only a buggy caller triggers it → `panic`.

---

## The `#[non_exhaustive]` Pattern

When nexus adds a new error variant in a future version, you don't want to break
downstream code that matches on your error type:

```rust
#[derive(Debug, Error)]
#[non_exhaustive]
pub enum NexusError {
    #[error("auth error: {0}")]
    Auth(#[from] AuthError),

    #[error("protocol error: {0}")]
    Protocol(#[from] ProtocolError),

    // Future versions may add more variants
    // #[error("rate limit exceeded")]
    // RateLimited,
}
```

With `#[non_exhaustive]`, callers must have a `_ => { ... }` arm:

```rust
match err {
    NexusError::Auth(e) => { /* ... */ }
    NexusError::Protocol(e) => { /* ... */ }
    _ => { /* Future variants handled here */ }
}
```

This is forward compatibility for error types. Apply it to error types that ship in
published libraries. For internal error types in a binary-only project, it's optional.

---

## Exercises

**Exercise 1 — Implement the Error Hierarchy**

Create `nexus-core/src/error.rs` with all four error types (`AuthError`, `ProtocolError`,
`StoreError`, `NexusError`). Verify that the `?` operator works correctly:

```rust
// This should compile and return NexusError::Protocol on a bad command byte
fn parse_command(byte: u8) -> Result<Command, NexusError> {
    Command::from_byte(byte).map_err(|e| NexusError::Protocol(e))
}
```

Write a test that calls `parse_command(0xFF)` and matches the result against
`Err(NexusError::Protocol(ProtocolError::UnknownCommand { code: 0xFF }))`.

**Exercise 2 — Error Context with anyhow**

In `nexus-server/src/main.rs`, use `anyhow` for startup errors. Add `.with_context()`
to each fallible call so that a startup failure produces an error message like:

```
Error: Failed to start nexus server

Caused by:
    0: Failed to bind listener
    1: Address already in use (os error 98)
```

Test this by trying to bind the server twice on the same port. The second bind should
fail with the full context chain.

**Exercise 3 — Error Handling Policy**

In `run_active_loop()`, implement the four-way error dispatch:
- `NexusError::Protocol` → log WARN, send ERROR frame, continue
- `NexusError::Store` → log ERROR, send ERROR frame, continue
- `NexusError::Transport` → log DEBUG, break
- `NexusError::ClientDisconnected` → break silently

Write a test that injects a `ProtocolError::PayloadTooLarge` via a mock and verifies:
1. The connection stays open (loop doesn't break)
2. An ERROR frame was sent
3. A WARN log line was emitted (use `tracing_test` or check the return value)

---

## Checklist

- [ ] One `thiserror` enum per domain (`AuthError`, `ProtocolError`, `StoreError`)
- [ ] One top-level `NexusError` that wraps domain errors with `#[from]`
- [ ] No `Box<dyn Error>` in library code — use concrete error enums
- [ ] `anyhow` only in binaries (`nexus-server/src/main.rs`) not in libraries
- [ ] Protocol errors → log at WARN, send error frame, continue connection
- [ ] Transport errors → log at DEBUG, close connection
- [ ] Store errors → log at ERROR, send error frame, continue connection
- [ ] Ambiguous `From<io::Error>` avoided: use `.map_err()` to be explicit
- [ ] `#[non_exhaustive]` on published error types to allow future variant additions
