# Doc 08 — Single-Use Types and Move as Proof

🟡 This is Rust's ownership system used as a correctness tool, not just a memory tool.

Every Rust developer learns that ownership prevents memory bugs: no dangling pointers, no double-frees. But ownership does something more interesting that most developers miss: it can **prove behavioral constraints** at compile time.

If a type is `!Clone` and `!Copy`, it can only be used **once**. When you pass it to a function, it's gone. The compiler prevents you from using it again — not at runtime, not as a defensive check, not through a lock: at **compile time, before the program runs**.

This is a linear type system. Rust has one built into the language. This doc shows how to use it to enforce "this must happen exactly once" properties in the music queue.

---

## The Motivating Problem: Things That Must Happen Once

The music queue interacts with several single-use resources:

- **WebSocket session tokens** — a handshake token must be consumed to establish a session; using it twice would establish two sessions with the same identity
- **Message delivery receipts** — acknowledging a message twice in a message broker causes duplicate processing
- **Transaction handles** — a database transaction can be committed or rolled back, but not both
- **One-time event triggers** — "start playback" for a new queue should only fire once per queue creation

In C, these are "documented" constraints:

```c
// "Call this exactly once. We have no way to enforce this." 
void start_queue_playback(QueueContext *ctx);
```

Nothing prevents you from calling it twice. The second call might start a duplicate stream, corrupt the playback position, or crash — depending on what `start_queue_playback` happens to do with already-initialized state.

---

## Move Semantics as Linear Types

Rust's ownership rules create a linear type system: a value can be consumed exactly once (moved), unless it implements `Copy`. When you move a value, the original binding is gone. The compiler tracks this statically.

```rust
struct SessionToken {
    token_id: uuid::Uuid,
    user_id: i64,
    // NOT Clone, NOT Copy — deliberate
}

fn establish_websocket_session(token: SessionToken, ws: WebSocket) -> ActiveSession {
    // token is moved here — consumed
    ActiveSession {
        user_id: token.user_id,
        ws,
        // token is gone
    }
}

fn bad_code() {
    let token = SessionToken { token_id: Uuid::new_v4(), user_id: 42 };
    
    let session1 = establish_websocket_session(token, ws1);  // token consumed
    let session2 = establish_websocket_session(token, ws2);  // COMPILE ERROR
    //                                         ^^^^^ use of moved value: `token`
}
```

The compiler proves nonce reuse is impossible — not by checking at runtime, but by making it a type error.

---

## Case Study: One-Time Queue Start Token

The music queue server creates a `QueueSession` when a user starts listening. The first HTTP request authenticates the user and creates a session token. That token — consumed exactly once — initializes the WebSocket connection. If someone tries to reuse the token (replay attack, client bug), the compiler prevents it:

```rust
use uuid::Uuid;

/// A one-time-use token for establishing a WebSocket music queue session.
/// Not Clone, not Copy — can be used exactly once.
pub struct QueueSessionToken {
    pub token_id: Uuid,
    pub user_id: i64,
    pub queue_id: i64,
    pub expires_at: std::time::Instant,
}

/// The active WebSocket session, created by consuming the token.
pub struct ActiveQueueSession {
    pub user_id: i64,
    pub queue_id: i64,
    connection_id: Uuid,  // unique per connection
}

/// Upgrade from a session token to an active session.
/// Consumes the token — it cannot be used again.
pub fn upgrade_to_session(
    token: QueueSessionToken,  // moved, not borrowed
    ws: WebSocket,
) -> Result<ActiveQueueSession, Error> {
    // Validate before consuming
    if token.expires_at < std::time::Instant::now() {
        return Err(Error::TokenExpired);
    }

    // Token is consumed here. Its fields are moved into ActiveQueueSession.
    Ok(ActiveQueueSession {
        user_id: token.user_id,
        queue_id: token.queue_id,
        connection_id: Uuid::new_v4(),
    })
    // token is gone — using it again would be a compile error
}
```

---

## Case Study: Single-Use Message Acknowledgment

Message queues (Redis Streams, RabbitMQ, Kafka) require you to acknowledge each message exactly once. Double-acknowledging is usually harmless, but in some setups it causes duplicate delivery. Single-use types make double-acking impossible:

```rust
/// A Redis Streams message waiting to be processed.
/// Not Clone — the acknowledgment can only happen once.
pub struct PendingMessage {
    pub stream_key: String,
    pub message_id: String,  // Redis message ID (e.g., "1234567890-0")
    pub payload: Vec<u8>,
    pub(crate) _sealed: (),  // prevents outside construction
}

impl PendingMessage {
    /// Acknowledge this message — consumes the token, prevents double-ack.
    pub async fn ack(self, redis: &mut redis::aio::Connection) -> Result<(), redis::RedisError> {
        redis::cmd("XACK")
            .arg(&self.stream_key)
            .arg("music-queue-group")
            .arg(&self.message_id)
            .query_async(redis)
            .await
        // self is consumed — this message can never be acked again
    }

    /// Reject this message (move it back to the pending list).
    /// Also consumes — you can't reject and then ack.
    pub async fn nack(self, redis: &mut redis::aio::Connection) -> Result<(), redis::RedisError> {
        // Move the message back to the head of the pending list
        redis::cmd("XCLAIM")
            .arg(&self.stream_key)
            .arg("music-queue-group")
            .arg("consumer-0")
            .arg(0)  // min-idle-time: 0 (take it back immediately)
            .arg(&self.message_id)
            .query_async(redis)
            .await
        // self is consumed — can't nack twice or ack after nacking
    }
}

// In the queue event processor:
async fn process_queue_event(msg: PendingMessage, db: &Pool) {
    let payload = msg.payload.clone();  // clone the data we need
    
    match parse_queue_command(&payload) {
        Ok(command) => {
            execute_command(command, db).await;
            msg.ack(&mut db.redis()).await.unwrap();  // msg is consumed
        }
        Err(parse_error) => {
            eprintln!("Malformed message: {parse_error}");
            msg.nack(&mut db.redis()).await.unwrap();  // msg is consumed
        }
    }
    // After either branch, msg is gone — no double-ack possible
}
```

---

## Case Study: Database Transaction Handle

A database transaction can be committed or rolled back, but not both — and not committed twice. The transaction handle as a single-use type enforces this:

```rust
/// A database transaction. Must be committed or rolled back exactly once.
/// Not Clone — the commit or rollback action is non-repeatable.
pub struct Transaction<'a> {
    conn: &'a mut sqlx::SqliteConnection,
    committed: bool,
}

impl<'a> Transaction<'a> {
    pub(crate) async fn begin(conn: &'a mut sqlx::SqliteConnection) -> Result<Self, sqlx::Error> {
        sqlx::query("BEGIN").execute(&mut *conn).await?;
        Ok(Transaction { conn, committed: false })
    }

    /// Commit the transaction — consumes it, preventing double-commit or rollback.
    pub async fn commit(mut self) -> Result<(), sqlx::Error> {
        sqlx::query("COMMIT").execute(&mut *self.conn).await?;
        self.committed = true;
        Ok(())
        // self is consumed
    }

    /// Explicit rollback — also consumes the transaction.
    pub async fn rollback(self) -> Result<(), sqlx::Error> {
        sqlx::query("ROLLBACK").execute(&mut *self.conn).await?;
        Ok(())
        // self is consumed — committed is still false, so Drop will NOT rollback again
    }

    /// Execute a query within the transaction
    pub async fn execute(&mut self, sql: &str) -> Result<sqlx::sqlite::SqliteQueryResult, sqlx::Error> {
        sqlx::query(sql).execute(&mut *self.conn).await
    }
}

impl<'a> Drop for Transaction<'a> {
    fn drop(&mut self) {
        if !self.committed {
            // Safety net: if the transaction was dropped without committing
            // (e.g., due to an early return from an error), roll back.
            // Note: we can't .await here (Drop is sync), so we use try_execute.
            // For async safety nets, use explicit rollback() instead.
            let _ = self.conn.execute_many("ROLLBACK");
        }
    }
}

// Using the transaction:
async fn update_queue_atomically(
    db: &mut sqlx::SqliteConnection,
    queue_id: i64,
    new_track: &Track,
) -> Result<(), Error> {
    let mut tx = Transaction::begin(db).await?;
    
    tx.execute("DELETE FROM queue_entries WHERE queue_id = ?1").await?;
    tx.execute("INSERT INTO queue_entries (queue_id, track_id) VALUES (?1, ?2)").await?;
    
    // On success: commit consumes tx
    tx.commit().await?;
    Ok(())
    
    // On any error: the ? returns early, tx is dropped, Drop calls ROLLBACK
}
```

---

## Using `#[must_use]` to Strengthen Single-Use Types

If someone creates a single-use value and immediately drops it without using it, the compiler doesn't warn by default. `#[must_use]` turns this into a warning:

```rust
#[must_use = "Session tokens must be consumed to establish a session"]
pub struct QueueSessionToken {
    pub token_id: Uuid,
    pub user_id: i64,
    // ...
}

// Now this generates a warning:
fn issue_token_and_ignore() {
    let _token = create_session_token(42, 100);
    // warning: unused `QueueSessionToken` that must be used
    //          Session tokens must be consumed to establish a session
}
```

`#[must_use]` doesn't prevent the programmer from intentionally ignoring the value with `let _ = create_session_token(...)`, but it warns in the common case where someone forgot to use the result.

---

## The `Sealed` Pattern: Controlling Construction

A single-use type only works if external code can't construct it directly (bypassing the validation that ensures the "exactly one" invariant). The `sealed` pattern prevents this:

```rust
// In lib.rs or mod.rs — the type is public, but _sealed prevents outside construction
pub struct QueueSessionToken {
    pub token_id: Uuid,
    pub user_id: i64,
    pub queue_id: i64,
    pub expires_at: std::time::Instant,
    pub(crate) _sealed: (),  // zero-size, but only this crate can set it
}

impl QueueSessionToken {
    // The only constructor — validates before creating
    pub(crate) fn issue(user_id: i64, queue_id: i64, ttl: std::time::Duration) -> Self {
        QueueSessionToken {
            token_id: Uuid::new_v4(),
            user_id,
            queue_id,
            expires_at: std::time::Instant::now() + ttl,
            _sealed: (),
        }
    }
}

// External code can READ the public fields but CANNOT construct QueueSessionToken directly:
// QueueSessionToken { token_id: ..., _sealed: () }  
// ERROR: field `_sealed` of struct `QueueSessionToken` is private
```

With a private field, external callers must use the provided constructor (which validates the token issuance policy), and the compiler prevents them from forging a token.

---

## When to Apply Single-Use Types

The pattern fits any situation where a resource or operation must happen exactly once:

| Situation | Single-use type? | Notes |
|-----------|:---------------:|-------|
| Cryptographic nonce | ✅ Always | Nonce reuse breaks authentication |
| Ephemeral DH key | ✅ Always | Reuse weakens forward secrecy |
| Session auth token | ✅ Usually | Prevent replay attacks |
| Message acknowledgment | ✅ Usually | Prevent duplicate processing |
| OTP (one-time password) | ✅ Usually | By definition single-use |
| Transaction handle | ✅ Usually | Commit or rollback, not both |
| Calibration token | ✅ Sometimes | If recalibration is invalid |
| File write lock | ❌ Usually not | Locks are reentrant by design |
| Regular data buffers | ❌ Not applicable | Need repeated access |

---

## How It Breaks

**Adding `#[derive(Clone)]` to a single-use type.**
Once a type is `Clone`, the linear-use guarantee is gone. Anyone can clone the token and use it twice. Single-use types must explicitly NOT derive `Clone` or `Copy`. Code review should flag any `#[derive(Clone)]` added to a type that was previously single-use.

**Moving out of the type to reuse the ID.**
```rust
let token = create_session_token(user_id, queue_id);
let token_id = token.token_id;  // copy the UUID out (UUID is Copy)
establish_session(token);       // token consumed
// Now try to create a second session with the same token_id...
// The type prevented reusing the token, but you extracted the ID first.
```
The single-use guarantee is on the **token**, not on the UUID inside it. If the server validates sessions by UUID, the attacker can still try to use the UUID directly. The type-level guarantee needs to be paired with server-side validation that a UUID is only valid once.

**Ignoring the `#[must_use]` warning with `let _ =`.**
`let _ = issue_token()` satisfies the compiler but means the token is immediately dropped — wasted. Make `#[must_use]` an error in your codebase by adding to `lib.rs`:
```rust
#![deny(unused_must_use)]
```
Now `let _ = issue_token()` is a compilation error, not just a warning.

**Drop order mattering unexpectedly.**
When a function returns early from an error, Rust drops variables in reverse order of declaration. For the `Transaction` type, this means the `drop()` rollback fires when the `?` propagates an error. But if you have two `Transaction`s in scope, the second one might roll back first, which could violate your assumptions about ordering. Structure your code to have at most one active transaction per scope.
