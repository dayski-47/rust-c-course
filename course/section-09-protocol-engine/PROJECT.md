# Project: nexus - A Binary Protocol Engine

> **Section 9 Capstone Project**  
> You are building a custom binary protocol server: the same kind of infrastructure
> that powers Redis, NATS, and Kafka at their core. The project synthesizes every
> major concept from the course into one production-capable system.

---

## What You're Building

`nexus` is a message routing engine with a custom binary protocol over TCP.

Clients connect, authenticate, publish messages to named topics, and subscribe to
receive messages from those topics. The server routes messages between publishers
and subscribers in real time.

This is the core pattern behind:
- **Redis Pub/Sub**: subscribers receive messages published to channels
- **NATS**: lightweight message broker with a text protocol
- **MQTT brokers**: IoT device messaging infrastructure

nexus uses a binary protocol instead of a text protocol. Binary is faster to parse,
more compact on the wire, and makes the state machine obvious.

**The system you're building:**

```
[Client A: Publisher]                    [Client C: Subscriber]
    |                                          ^
    | PUBLISH topic:alerts 0x0042             |
    v                                          |
[nexus server :9090] -------DELIVER---------->|
    ^                                          
    | Shared message routing state
    | AtomicU64 sequence numbers
    | Arc<RwLock<TopicMap>>
    v
[Topic: "alerts"]
  - subscriber list: [Client C, Client D]
  - message counter: AtomicU64
  - last_seq: u64

[Client B: Subscriber to "metrics"]       [Client D: Subscriber to "alerts"]
```

---

## Binary Protocol Design

Every nexus frame has the same envelope:

```
Offset   Size   Type         Field
------   ----   ----         -----
0        1      u8           version (must be 0x01)
1        1      u8           command (see table below)
2        2      u16 BE       flags
4        4      u32 BE       payload_len
8        8      u64 BE       sequence_id  (monotonically increasing, per-connection)
16       ?      [u8]         payload (payload_len bytes)
```

**Command codes:**

| Code | Name        | Direction    | Description |
|------|-------------|--------------|-------------|
| 0x01 | CONNECT     | Client→Server| Initiate connection, send auth token |
| 0x02 | CONNACK     | Server→Client| Connection accepted/rejected |
| 0x03 | PUBLISH     | Client→Server| Publish a message to a topic |
| 0x04 | PUBACK      | Server→Client| Publication acknowledged |
| 0x05 | SUBSCRIBE   | Client→Server| Subscribe to a topic |
| 0x06 | SUBACK      | Server→Client| Subscription acknowledged |
| 0x07 | DELIVER     | Server→Client| Deliver a message to subscriber |
| 0x08 | UNSUBSCRIBE | Client→Server| Cancel a subscription |
| 0x09 | PING        | Client→Server| Keepalive |
| 0x0A | PONG        | Server→Client| Keepalive response |
| 0x0B | DISCONNECT  | Either       | Graceful close |
| 0xFF | ERROR       | Server→Client| Error response |

**Protocol state machine:**

```
    [New TCP connection]
          |
          v
    [Unauthenticated]
          | CONNECT (with auth token)
          v
    [Authenticated]  ←───────────────────┐
          | SUBSCRIBE(topic)              |
          v                              | UNSUBSCRIBE(topic)
    [Active: subscribed to topics] ──────┘
          | DISCONNECT or error
          v
    [Closed]
```

**Why binary over text?**

In C, you'd use a state machine with `switch(state)` and runtime checks. In nexus,
the states are distinct types - `Connection<Unauthenticated>` vs
`Connection<Active>`. The compiler rejects calling `publish()` on an
unauthenticated connection because the method doesn't exist on that type.

---

## Architecture

```
nexus/
├── Cargo.toml           (workspace)
├── nexus-proto/         (protocol types and codec)
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── frame.rs     (Frame struct, parse/write)
│       ├── command.rs   (typed command enums)
│       └── codec.rs     (tokio_util::codec implementation)
├── nexus-core/          (engine: topic routing, subscriptions)
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── topic.rs     (TopicMap, AtomicU64 sequence numbers)
│       ├── session.rs   (type-state connection lifecycle)
│       └── error.rs     (NexusError hierarchy)
└── nexus-server/        (TCP listener, connection handling)
    ├── Cargo.toml
    └── src/
        ├── main.rs
        ├── listener.rs  (accept loop, spawn connection tasks)
        ├── handler.rs   (per-connection state machine)
        └── tracing.rs   (observability setup)
```

---

## Public API (Server Usage)

```rust
use nexus_core::{NexusEngine, EngineConfig};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Set up structured logging
    tracing_subscriber::fmt()
        .with_env_filter("nexus=debug,info")
        .json()
        .init();

    let engine = NexusEngine::new(EngineConfig {
        max_connections: 10_000,
        max_topics: 1_000,
        max_payload_bytes: 1024 * 1024,  // 1 MB per message
        auth_token: std::env::var("NEXUS_AUTH_TOKEN")?,
    });

    // Start TCP listener
    nexus_server::serve(engine, "0.0.0.0:9090").await?;
    Ok(())
}
```

---

## Client Protocol (Wire Bytes)

**CONNECT frame** (client → server):
```
version=0x01, command=0x01, flags=0x0000
payload: { "token": "secret-auth-token" }
```

**PUBLISH frame** (client → server):
```
version=0x01, command=0x03, flags=0x0000
payload: [topic_len: u16 BE][topic bytes][message bytes]
```

**SUBSCRIBE frame** (client → server):
```
version=0x01, command=0x05, flags=0x0000
payload: [topic_len: u16 BE][topic bytes]
```

**DELIVER frame** (server → client):
```
version=0x01, command=0x07, flags=0x0000
payload: [topic_len: u16 BE][topic bytes][msg_seq: u64 BE][message bytes]
```

---

## Cargo.toml (Workspace)

```toml
[workspace]
members = ["nexus-proto", "nexus-core", "nexus-server"]
resolver = "2"

[workspace.package]
version = "0.1.0"
edition = "2021"
license = "MIT OR Apache-2.0"

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
tokio-util = { version = "0.7", features = ["codec"] }
tokio-stream = "0.1"
futures = "0.3"
bytes = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }

[workspace.lints.rust]
unused_imports = "deny"
```

---

## Critical Patterns for This Section

### 1. Typed Frame Parsing with tokio_util::codec

```rust
use tokio_util::codec::{Decoder, Encoder};
use bytes::{Bytes, BytesMut, Buf};

pub struct NexusCodec;

impl Decoder for NexusCodec {
    type Item = Frame;
    type Error = ProtocolError;

    fn decode(&mut self, src: &mut BytesMut) -> Result<Option<Self::Item>, Self::Error> {
        // Need at least 16 bytes for the header
        if src.len() < 16 {
            return Ok(None);  // Wait for more data
        }

        let payload_len = u32::from_be_bytes(src[4..8].try_into().unwrap()) as usize;
        let total_len = 16 + payload_len;

        if src.len() < total_len {
            src.reserve(total_len - src.len());
            return Ok(None);  // Wait for full frame
        }

        let frame_bytes = src.split_to(total_len).freeze();
        Ok(Some(Frame::parse(frame_bytes)?))
    }
}
```

### 2. AtomicU64 for Connection and Message Sequencing

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;

pub struct NexusEngine {
    connection_counter: AtomicU64,
    message_counter: Arc<AtomicU64>,
    // ...
}

impl NexusEngine {
    pub fn next_connection_id(&self) -> u64 {
        // Relaxed: we only need uniqueness, not ordering with other data
        self.connection_counter.fetch_add(1, Ordering::Relaxed)
    }

    pub fn next_message_seq(&self) -> u64 {
        // AcqRel: the message sequence must be visible before the message content
        self.message_counter.fetch_add(1, Ordering::AcqRel)
    }
}
```

### 3. Type-State Connection Lifecycle

```rust
use std::marker::PhantomData;

pub struct Unauthenticated;
pub struct Active;
pub struct Closed;

pub struct Connection<S> {
    id: u64,
    state: PhantomData<S>,
    // inner state here
}

impl Connection<Unauthenticated> {
    pub fn authenticate(self, token: &str) -> Result<Connection<Active>, AuthError> {
        // Validate token; transition to Active state
        todo!()
    }
}

impl Connection<Active> {
    pub fn publish(&self, topic: &str, payload: Bytes) -> Result<u64, NexusError> {
        // Only callable on authenticated connections
        todo!()
    }
    
    pub fn close(self) -> Connection<Closed> {
        todo!()
    }
}
```

### 4. Structured Logging Per-Connection

```rust
use tracing::{info, debug, error, instrument, Span};

#[instrument(skip(self), fields(conn_id = %self.id, remote = %self.peer_addr))]
async fn handle_publish(&self, topic: &str, payload: &Bytes) -> Result<(), NexusError> {
    debug!(
        topic = topic,
        payload_bytes = payload.len(),
        "Publishing message"
    );
    
    let seq = self.engine.next_message_seq();
    
    // Route to subscribers
    let subscriber_count = self.engine.route(topic, seq, payload.clone()).await?;
    
    info!(
        topic = topic,
        seq = seq,
        subscribers = subscriber_count,
        "Message delivered"
    );
    
    Ok(())
}
```

---

## Milestones

### Milestone 1: Frame Parsing

Parse and serialize binary frames. Tests that verify:
- All valid command codes round-trip correctly
- Partial frames return `None` (wait for more data)
- Invalid version byte returns `ProtocolError::UnsupportedVersion`
- payload_len larger than `max_payload_bytes` returns `ProtocolError::PayloadTooLarge`

No networking yet - just the codec.

### Milestone 2: TCP Listener and Basic Echo

Accept TCP connections. For each connection, spawn a Tokio task. Read frames using the codec. Log each frame received. Echo it back.

```bash
$ nc -q1 localhost 9090 | xxd
```

### Milestone 3: CONNECT/CONNACK Authentication

Add CONNECT frame handling. Compare the auth token from the payload against the configured token. Send CONNACK with success or failure code. Track connection state.

Add a tracing span per connection with `connection_id` field.

### Milestone 4: PUBLISH and Topic Registration

When a client in Active state sends a PUBLISH frame, register the topic if it doesn't exist (create a channel). Log the publish with tracing. Send PUBACK.

No delivery to subscribers yet - just persistence of the topic.

### Milestone 5: SUBSCRIBE and DELIVER

On SUBSCRIBE, add the connection's sender to the topic's subscriber list. On PUBLISH, route the message to all subscribers via DELIVER frames. Add an AtomicU64 sequence counter per topic. Messages to a subscriber are delivered in sequence order.

### Milestone 6: UNSUBSCRIBE and PING/PONG

Implement UNSUBSCRIBE (remove from subscriber list). Implement PING/PONG keepalive. Add a per-connection inactivity timeout: if no frame is received in 30 seconds, close the connection.

### Milestone 7: Graceful Shutdown

On SIGTERM:
1. Stop accepting new connections
2. Send DISCONNECT to all active connections
3. Wait up to 5 seconds for in-flight publishes to complete
4. Force-close remaining connections

Use a `tokio::sync::watch` channel to broadcast the shutdown signal.

### Milestone 8: Error Handling and Recovery

Add structured error responses (ERROR frame with error code and message). Test connection error paths:
- Auth failure
- Publish to non-existent topic with `auto_create = false`
- Payload too large
- Malformed frame

### Milestone 9: Property-Based Tests

Use proptest to verify:
- Any sequence of valid frames parses without panicking
- Publish → Deliver preserves message bytes exactly
- Subscribe → Unsubscribe leaves no dangling subscriber references
- AtomicU64 sequence numbers are monotonically increasing under concurrent load

### Milestone 10: Performance Benchmarks

Criterion benchmarks for:
- Frame parsing throughput (frames/sec)
- Message routing latency with 1, 10, 100 subscribers
- Peak throughput: publishers/sec with a saturated subscriber
- Memory: bytes per connection, bytes per subscription

---

## What You Bring to This Section

Every section built toward nexus:

- **S1:** `NexusError` is a `thiserror` enum with `#[from]` conversion chains - same pattern as your first error types.
- **S3/S4:** The async TCP server is the same pattern as the chat server - `tokio::spawn` per connection, `select!` for concurrent events.
- **S6/S7:** Type-state for `Connection<S>` uses `PhantomData` from Section 7's deep dive.
- **S8:** Binary framing is the same `#[repr(C)]` and byte-level parsing mindset from ironkv.

nexus is not a new project. It's the same Rust you've been writing, composed at the systems level.
