# Doc 04 — Protocol State Machines in Types

🔴 nexus connections have strict states: a client that hasn't authenticated cannot
publish or subscribe. In C, this is a runtime enum checked on every operation. In
Rust, it's a compile-time constraint — calling `publish()` on an unauthenticated
connection is a type error, not a runtime error. This doc shows how to encode the
nexus connection lifecycle in the type system.

---

## The C Approach: Runtime State Machine

In C, the natural representation is an enum tracked at runtime:

```c
typedef enum {
    CONN_UNAUTHENTICATED = 0,
    CONN_ACTIVE          = 1,
    CONN_CLOSED          = 2,
} conn_state_t;

typedef struct {
    conn_state_t state;
    int fd;
    uint64_t id;
    char subscribed_topic[256];  // empty if not subscribed
} nexus_conn_t;

int nexus_publish(nexus_conn_t *conn, const char *topic,
                  const uint8_t *payload, size_t len) {
    if (conn->state != CONN_ACTIVE) {  // Runtime check
        return -EACCES;
    }
    // ... publish ...
    return 0;
}
```

**Problems with this approach:**

1. Every function that needs authentication must check state manually
2. If you forget the check, the code compiles — the bug ships
3. The caller has no idea from the type signature which states are valid
4. Tests for "wrong state" behavior require runtime setup, not compile-time verification

---

## The Type-State Pattern

With type-state, each state is a distinct zero-sized type. Methods only exist on
the types that can call them:

```rust
// nexus-core/src/session.rs
use std::marker::PhantomData;

// State marker types — zero-sized, compile-time only
pub struct Unauthenticated;
pub struct Active;
pub struct Closed;

// The connection — generic over its current state
pub struct Connection<S> {
    pub id: u64,
    peer_addr: std::net::SocketAddr,
    // ... transport layer ...
    _state: PhantomData<S>,
}
```

`PhantomData<S>` has zero size at runtime. It carries the state type `S` through
the type system without any runtime cost. `Connection<Unauthenticated>` and
`Connection<Active>` are different types — the compiler distinguishes them even
though their runtime layout is identical.

---

## State Transitions as Consuming Methods

Transitions consume the old state and return the new state:

```rust
impl Connection<Unauthenticated> {
    /// Create a new connection in the unauthenticated state.
    pub fn new(id: u64, peer_addr: std::net::SocketAddr) -> Self {
        Connection {
            id,
            peer_addr,
            _state: PhantomData,
        }
    }

    /// Validate auth token; if valid, transition to Active.
    ///
    /// Consumes `self` — the unauthenticated connection cannot be used after this call.
    pub fn authenticate(
        self,
        token: &str,
        expected_token: &str,
    ) -> Result<Connection<Active>, AuthError> {
        if token == expected_token {
            Ok(Connection {
                id: self.id,
                peer_addr: self.peer_addr,
                _state: PhantomData,
            })
        } else {
            Err(AuthError::InvalidToken)
        }
    }
}

impl Connection<Active> {
    /// Publish a message to a topic. Only valid in the Active state.
    pub async fn publish(
        &self,
        engine: &NexusEngine,
        topic: &str,
        payload: bytes::Bytes,
    ) -> Result<u64, NexusError> {
        let seq = engine.next_message_seq();
        engine.route(topic, seq, payload).await?;
        Ok(seq)
    }

    /// Subscribe to a topic. Only valid in the Active state.
    pub fn subscribe(
        &mut self,
        topic: &str,
        engine: &NexusEngine,
    ) -> Result<tokio::sync::mpsc::Receiver<(u64, bytes::Bytes)>, NexusError> {
        engine.subscribe(topic, self.id)
    }

    /// Close the connection. Consumes self — can't be used after this.
    pub fn close(self) -> Connection<Closed> {
        Connection {
            id: self.id,
            peer_addr: self.peer_addr,
            _state: PhantomData,
        }
    }
}

impl Connection<Closed> {
    /// No operations available — only cleanup is possible.
    pub fn id(&self) -> u64 {
        self.id  // Still readable for logging
    }
}
```

Now the compiler prevents misuse:

```rust
// ❌ Compile error — publish() doesn't exist on Connection<Unauthenticated>
let conn = Connection::new(1, peer_addr);
conn.publish(&engine, "topic", payload).await?;
// error[E0599]: no method named `publish` found for struct `Connection<Unauthenticated>`

// ✅ Correct sequence
let conn = Connection::new(1, peer_addr);
let active = conn.authenticate(token, &expected_token)?;  // Transition
active.publish(&engine, "topic", payload).await?;         // Now allowed

// ❌ Using after close — doesn't compile
let closed = active.close();
closed.publish(&engine, "topic", payload).await?;
// error[E0599]: no method named `publish` found for struct `Connection<Closed>`
```

The error messages point to the wrong usage site. No runtime check, no `assert!`, no
`if state != ACTIVE` — the compiler enforces the protocol.

---

## Async State Transitions

State transitions in async contexts work the same way, but the consuming pattern
interacts with async carefully. Since Rust doesn't allow `self` to be consumed
across an `.await`, structure transitions as:

```rust
impl Connection<Unauthenticated> {
    /// Async transition: read and validate the CONNECT frame.
    pub async fn handle_connect(
        self,
        frame: Frame,
        config: &NexusConfig,
    ) -> Result<Connection<Active>, (Connection<Closed>, AuthError)> {
        let token = parse_connect_payload(&frame.payload)?;

        // Async validation (e.g., check a token database)
        let valid = config.auth.validate(&token).await;

        if valid {
            Ok(Connection {
                id: self.id,
                peer_addr: self.peer_addr,
                _state: PhantomData,
            })
        } else {
            let closed = Connection {
                id: self.id,
                peer_addr: self.peer_addr,
                _state: PhantomData,
            };
            Err((closed, AuthError::InvalidToken))
        }
    }
}
```

The error variant carries a `Connection<Closed>` — the caller knows the connection
is done and should clean up resources, without checking a separate "is closed" field.

---

## The Full Connection Loop

In the actual connection handler, the state machine drives the main loop:

```rust
// nexus-server/src/handler.rs

pub async fn handle_connection(stream: TcpStream, engine: Arc<NexusEngine>) {
    use tokio_util::codec::Framed;
    use futures::StreamExt;

    let peer = stream.peer_addr().unwrap_or("[unknown]".parse().unwrap());
    let conn_id = engine.allocate_connection_id();

    tracing::info!(conn_id, ?peer, "Connection accepted");

    let mut framed = Framed::new(stream, NexusCodec::new(engine.config.max_payload_bytes));

    // Start in unauthenticated state
    let unauth = Connection::new(conn_id, peer);

    // Wait for CONNECT frame
    let active = match wait_for_connect(&mut framed, unauth, &engine).await {
        Ok(active) => active,
        Err(e) => {
            tracing::warn!(conn_id, error = ?e, "Authentication failed");
            return;
        }
    };

    engine.connection_opened();

    // Now in Active state — run the main loop
    run_active(&mut framed, active, engine.clone()).await;

    engine.connection_closed();
    tracing::info!(conn_id, "Connection closed");
}

async fn wait_for_connect(
    framed: &mut Framed<TcpStream, NexusCodec>,
    conn: Connection<Unauthenticated>,
    engine: &NexusEngine,
) -> Result<Connection<Active>, NexusError> {
    // Only accept the first frame; if it's not CONNECT, close.
    match framed.next().await {
        Some(Ok(frame)) if frame.command == Command::Connect => {
            let token = parse_connect_payload(&frame.payload)?;
            let active = conn.authenticate(&token, &engine.config.auth_token)
                .map_err(NexusError::Auth)?;
            // Send CONNACK
            framed.send(Frame::connack_ok(frame.sequence_id)).await?;
            Ok(active)
        }
        Some(Ok(frame)) => {
            framed.send(Frame::error(frame.sequence_id, ErrorCode::MustConnectFirst)).await.ok();
            Err(NexusError::Protocol("Expected CONNECT".into()))
        }
        Some(Err(e)) => Err(e.into()),
        None => Err(NexusError::ClientDisconnected),
    }
}

async fn run_active(
    framed: &mut Framed<TcpStream, NexusCodec>,
    conn: Connection<Active>,
    engine: Arc<NexusEngine>,
) {
    let (msg_tx, mut msg_rx) = tokio::sync::mpsc::channel::<(u64, bytes::Bytes)>(128);

    loop {
        tokio::select! {
            // Handle incoming frames from the client
            frame_result = framed.next() => {
                match frame_result {
                    Some(Ok(frame)) => {
                        if handle_active_frame(framed, &conn, &engine, &msg_tx, frame).await.is_err() {
                            break;
                        }
                    }
                    Some(Err(e)) => {
                        tracing::warn!(conn_id = conn.id, error = ?e, "Frame error");
                        break;
                    }
                    None => break,
                }
            }

            // Deliver incoming messages to the client
            Some((seq, payload)) = msg_rx.recv() => {
                if framed.send(Frame::deliver(seq, payload)).await.is_err() {
                    break;
                }
            }
        }
    }

    let _closed = conn.close();  // Transition to Closed — ensures cleanup
}
```

The flow is explicit: you can't call `run_active` without a `Connection<Active>`.
The type signature enforces the protocol.

---

## When the State Can't Be Encoded

Not all protocols fit cleanly into type-state. nexus has a complication: a client
can subscribe to multiple topics. The "subscribed to topics" state is dynamic — it's
a `HashSet<String>`, not a fixed type.

The rule: encode the **phase** (unauthenticated, active, closed) in types. Encode
**dynamic data** (subscribed topics list, sequence counters) in the struct fields.

```rust
// Active connection — subscriptions are data, not type-state
pub struct Connection<S> {
    pub id: u64,
    peer_addr: std::net::SocketAddr,
    subscriptions: std::collections::HashSet<String>,  // Dynamic — not in type
    _state: PhantomData<S>,
}

impl Connection<Active> {
    pub fn subscribe(&mut self, topic: String) {
        self.subscriptions.insert(topic);
    }

    pub fn unsubscribe(&mut self, topic: &str) {
        self.subscriptions.remove(topic);
    }

    pub fn subscriptions(&self) -> &HashSet<String> {
        &self.subscriptions
    }
}
```

The criterion: if missing a method is a "wrong protocol" error (sending a message
before authenticating), that's type-state territory. If it's a "wrong data" error
(subscribing to the same topic twice, or unsubscribing from a topic you're not
subscribed to), that's runtime validation.

---

## Type-State for the Protocol Codec

The codec also has states. Before negotiating a version, it doesn't know the
maximum payload size. After negotiation, it's configured:

```rust
pub struct Uninitialized;
pub struct Configured { max_payload_bytes: usize }

pub struct NexusCodec<S = Configured> {
    _state: PhantomData<S>,
    // ... other fields
}

impl NexusCodec<Uninitialized> {
    pub fn new() -> Self {
        NexusCodec { _state: PhantomData }
    }

    pub fn configure(self, max_payload_bytes: usize) -> NexusCodec<Configured> {
        NexusCodec { _state: PhantomData }
    }
}

// Only a Configured codec can decode frames
impl tokio_util::codec::Decoder for NexusCodec<Configured> {
    // ...
}
```

The default type parameter `<S = Configured>` means `NexusCodec` (without a type
parameter) is `NexusCodec<Configured>`. This is the public API — most users never
see the `<S>`. The type-state is an internal implementation detail.

---

## When to Use Type-State vs Runtime Enum

**Use type-state when:**
- An operation is *always wrong* in a given phase — calling it should never compile
- The state set is small and fixed at compile time (2–5 phases)
- Incorrect ordering of operations would be a protocol violation or security hole
- You want compile-fail tests to document and verify the invariant

**Use a runtime enum or flag when:**
- The set of possible states is determined at runtime (loaded from config, negotiated)
- You need to store heterogeneous connections in the same collection (`Vec<Connection>`)
- The "wrong state" transitions are expected and need graceful handling, not hard rejection
- The state has too many degrees of freedom (nested, concurrent, dependent on data values)

```rust
// Type-state: small fixed phases, protocol enforcement
//   Connection<Unauthenticated> | Connection<Active> | Connection<Closed>

// Runtime enum: dynamic states or mixed collections
enum ConnectionState { Handshaking { version: u8 }, Active, Draining, Closed }
struct Connection { state: ConnectionState, id: u64 }
// Needed when different connections in a Vec might be in different states
let connections: Vec<Connection> = vec![...];
```

The most common mistake is trying to encode *everything* in types. If the state has
more than ~5 variants or is negotiated at runtime, the type-state overhead (new
types, conversion methods, `PhantomData`) outweighs the benefit. Use it where it
catches real protocol bugs, not as a dogma.

## C Comparison: Before and After

| Aspect | C runtime enum | Rust type-state |
|--------|----------------|-----------------|
| Wrong-state call | Silent at compile time, runtime error | Compile error |
| Test for wrong-state | Requires runtime setup | `compile_fail` in `trybuild` |
| Documentation | Comment on the function | Type signature |
| Performance | Check on every call | Zero cost (no check) |
| Refactoring | Can forget to add check | Adding a state = all callers must handle it |

The runtime check in C is a contract expressed in prose documentation. The type-state
in Rust is a contract expressed in the type system. The compiler enforces it instead
of the programmer.

---

## Testing Type-State with `trybuild`

Verify that the type system actually rejects wrong-state calls:

```toml
# nexus-core/Cargo.toml
[dev-dependencies]
trybuild = "1"
```

```rust
// tests/ui_tests.rs
#[test]
fn compile_fail_tests() {
    let t = trybuild::TestCases::new();
    t.compile_fail("tests/ui/publish_before_auth.rs");
    t.compile_fail("tests/ui/use_after_close.rs");
}
```

```rust
// tests/ui/publish_before_auth.rs
fn main() {
    let conn = nexus_core::session::Connection::new(1, "127.0.0.1:0".parse().unwrap());
    // This must NOT compile:
    let _ = conn.publish; // no such field/method on Connection<Unauthenticated>
}
```

```
// tests/ui/publish_before_auth.stderr
error[E0599]: no method named `publish` found for struct `Connection<Unauthenticated>`
```

The `.stderr` file captures the expected error message. If the code compiles (the
invariant breaks), the test fails. If the error message changes, the test fails.
Type-level invariants have tests.

---

## Exercises

**Exercise 1 — Implement the State Transitions**

Implement the three state types (`Unauthenticated`, `Active`, `Closed`) and the
`Connection<S>` struct. Add:
- `Connection<Unauthenticated>::new(id: u64) -> Self`
- `Connection<Unauthenticated>::authenticate(self, token: &str, expected: &str) -> Result<Connection<Active>, AuthError>`
- `Connection<Active>::close(self) -> Connection<Closed>`

Write a test that demonstrates the happy path (new → authenticate → close) and the
failure path (authenticate with wrong token returns `Err`, the connection is consumed
and can't be used again).

**Exercise 2 — Add Subscriptions**

Add a `subscriptions: HashSet<String>` field to `Connection<S>` (accessible in all states).
Implement on `Connection<Active>`:
- `subscribe(&mut self, topic: String)` — insert into the HashSet
- `unsubscribe(&mut self, topic: &str)` — remove from the HashSet
- `is_subscribed(&self, topic: &str) -> bool`

Write a test that:
1. Creates an authenticated `Connection<Active>`
2. Subscribes to "alerts" and "metrics"
3. Unsubscribes from "alerts"
4. Asserts `is_subscribed("alerts") == false` and `is_subscribed("metrics") == true`

**Exercise 3 — trybuild Compile-Fail Tests**

Add `trybuild` as a dev-dependency. Create the test file
`tests/ui/publish_before_auth.rs` that attempts to call `publish()` on a
`Connection<Unauthenticated>`. Run `cargo test` — it should pass because the file
fails to compile (that's the expected behavior). Create the matching `.stderr` file
with the expected compiler error message.

---

## Checklist

- [ ] Each distinct protocol phase is a separate state type (`Unauthenticated`, `Active`, `Closed`)
- [ ] State transitions are consuming methods (`fn authenticate(self) -> Result<Connection<Active>, _>`)
- [ ] No `pub fn` on `Connection<X>` that belongs to a different state
- [ ] Dynamic data (subscriptions, sequence numbers) lives in struct fields, not in types
- [ ] `trybuild` compile-fail tests verify type constraints hold
- [ ] `PhantomData<S>` used with `_state: PhantomData<S>` naming convention
- [ ] `Connection<Closed>` returned on errors so the caller knows cleanup is needed

---

## Multi-Party Protocols: Both Sides Have a State Machine

The nexus connection is one state machine on the server side. The real world has
protocols where **both sides track state independently**, and the question is how
to prove — at compile time — that they are in sync.

Consider a TLS handshake. The client transitions:
`Idle → HelloSent → KeyExchanged → Established`.
The server transitions:
`Listening → HelloReceived → KeyExchanged → Established`.

They arrive at `Established` through different paths. If the server calls
`send_application_data()` before receiving the finished message, it's a protocol
error — but the server's type-state machine says it's in `Established`, so the
compiler won't catch it.

The solution is **session types**: each side's state is parameterized on what the
*other* side must do next. This is advanced territory. For nexus, the simpler
approach is correct: the server's state machine transitions in lockstep with the
wire protocol messages it receives. If the client sends a message that's only
valid in `Active` state and the server receives it while still in
`Unauthenticated`, the server's state-checking logic (which does the `Err(Closed)`
transition) handles it — the type system enforces state on the server side.

The practical takeaway: **let the protocol messages drive your state transitions**.
Your server's type-state machine advances when a valid message arrives; invalid
messages produce an error state.

---

## Capability Tokens: When Compile-Time State Isn't Enough

Type-state gives you compile-time enforcement of sequential state. But sometimes
you need something different: **proof that a specific authority was granted at
runtime**, not just that certain steps were taken in order.

The canonical example in nexus is subscription authorization. A client subscribes
to a topic, and you want to ensure that only subscribed clients can publish to that
topic. The type-state `Connection<Active>` tells you the connection is authenticated
— it doesn't tell you *which topics* the client subscribed to.

A **capability token** is a zero-sized type that proves a specific capability was
granted. It costs zero bytes at runtime (the compiler eliminates it), but its
presence as a function parameter makes it impossible to call the function without
having been through the granting code path.

```rust
// Zero-sized — compiles to nothing at runtime.
// Only our module can construct this (private _private field).
pub struct SubscriptionToken<'conn> {
    topic: String,
    _conn: std::marker::PhantomData<&'conn ()>,
}

impl<'conn> SubscriptionToken<'conn> {
    // Only constructable inside this module
    fn new(topic: String) -> Self {
        SubscriptionToken {
            topic,
            _conn: std::marker::PhantomData,
        }
    }
    
    pub fn topic(&self) -> &str {
        &self.topic
    }
}

// The connection grants tokens during Active state
impl Connection<Active> {
    pub fn subscribe(&mut self, topic: &str) -> SubscriptionToken<'_> {
        // ... add to self.subscriptions, send SUBSCRIBE_ACK ...
        SubscriptionToken::new(topic.to_string())
    }
    
    // publish requires proof that subscribe was called for this topic
    pub fn publish(
        &mut self,
        token: &SubscriptionToken<'_>,
        payload: &[u8],
    ) -> Result<(), NexusError> {
        // No runtime auth check needed — the token IS the proof
        // Token lifetime is tied to the connection, so we know
        // the connection is still alive and the subscription is valid
        self.send_publish(token.topic(), payload)
    }
}
```

The `'conn` lifetime on `SubscriptionToken<'conn>` ties the token's lifetime to
the connection. When the connection is dropped, all tokens it handed out become
invalid — the borrow checker enforces this at compile time. There is no runtime
expiry check needed.

The pattern in production systems:

| Scenario | Pattern |
|----------|---------|
| Topic subscription authorization | `SubscriptionToken<'conn>` |
| Admin-only operations | `AdminToken` (private constructor, granted on auth) |
| Multi-step sequencing | Chain: `StandbyOn → AuxOn → MainOn` (each proves prev) |
| Role-based access | Trait hierarchy: `CanRead: CanWrite: CanAdmin` |

Private constructors are the key. If `SubscriptionToken::new()` is private, there
is no way to fabricate a token — the caller must go through `subscribe()`.

---

## Async Trait Objects in State Machines

When implementing the nexus connection handler's read loop, you will hit the
**async trait object problem**: `async fn` in a trait is not (yet) object-safe by
default.

The issue is that `async fn` desugars to a method returning `impl Future<Output =
T>`, but `impl Trait` in a trait method's return position creates an unnameable
associated type. You can't put an unnameable type behind `dyn`.

**Option 1: `async_trait` crate**

The `async-trait` crate rewrites each `async fn` to return
`Pin<Box<dyn Future<Output = T> + Send + '_>>`. This makes the trait object-safe:

```rust
use async_trait::async_trait;

#[async_trait]
pub trait ProtocolHandler: Send + Sync {
    async fn on_connect(&mut self, conn_id: u64) -> Result<(), NexusError>;
    async fn on_message(&mut self, msg: WireMessage) -> Result<(), NexusError>;
    async fn on_disconnect(&mut self) -> Result<(), NexusError>;
}

// Now you can store a Box<dyn ProtocolHandler> and call async methods on it:
pub struct ConnectionRouter {
    handlers: HashMap<u64, Box<dyn ProtocolHandler>>,
}
```

The cost: each async method call allocates a `Box`. For a connection handler that
processes thousands of messages per second, this is measurable but usually
acceptable — the TCP I/O cost dominates.

**Option 2: Manual `BoxFuture`**

Without `async_trait`, write the return type explicitly:

```rust
use std::future::Future;
use std::pin::Pin;

// A type alias for clarity — this is the common pattern in tokio internals
type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;

pub trait ProtocolHandler: Send + Sync {
    fn on_message<'a>(
        &'a mut self,
        msg: WireMessage,
    ) -> BoxFuture<'a, Result<(), NexusError>>;
}

// Implement by boxing the async block:
impl ProtocolHandler for NexusHandler {
    fn on_message<'a>(
        &'a mut self,
        msg: WireMessage,
    ) -> BoxFuture<'a, Result<(), NexusError>> {
        Box::pin(async move {
            self.handle_wire_message(msg).await
        })
    }
}
```

`Box::pin(async move { ... })` allocates a boxed future on the heap and pins it in
place. This is what `async_trait` generates for you automatically.

**Option 3: Enum dispatch (zero allocation)**

If the set of handler types is fixed, use an enum instead of a trait object:

```rust
enum Handler {
    Nexus(NexusHandler),
    Legacy(LegacyHandler),
    Admin(AdminHandler),
}

impl Handler {
    async fn on_message(&mut self, msg: WireMessage) -> Result<(), NexusError> {
        match self {
            Handler::Nexus(h) => h.on_message(msg).await,
            Handler::Legacy(h) => h.on_message(msg).await,
            Handler::Admin(h) => h.on_message(msg).await,
        }
    }
}
```

No `Box`, no allocation, no vtable. The compiler monomorphizes each branch.
Downsides: all variants must be known at compile time, and adding a new handler
type requires changing the enum.

**When to use which:**

| Approach | Allocation | Open set? | Use when |
|----------|-----------|-----------|----------|
| `async_trait` | Box per call | Yes | Plugin system, unknown handler types |
| Manual BoxFuture | Box per call | Yes | No proc-macro dependency |
| Enum dispatch | Zero | No | Fixed set of well-known types |

For nexus, enum dispatch is often the right choice because the handler types are
known at compile time and the message hot path benefits from zero allocation.
