# Doc 09 — Putting It All Together

🔴 This doc synthesizes every pattern from Sections 1–9 into a working nexus
protocol engine. It walks through the critical paths — connection establishment,
message routing, graceful shutdown — and shows where each concept from the course
appears in the running system.

---

## The Architecture at a Glance

```
nexus-server (binary)
├── main.rs
│   ├── init_tracing()                      [Doc 06]
│   ├── NexusEngine::new()                  [Docs 01, 08]
│   └── serve(engine, addr)
│
├── listener.rs
│   └── accept_loop()
│       └── tokio::spawn(handle_connection) per connection
│
└── handler.rs
    └── handle_connection(stream, engine)
        ├── Framed::new(stream, NexusCodec)   [Doc 02]
        ├── Connection::<Unauthenticated>::new [Doc 04]
        ├── wait_for_connect()
        │   └── conn.authenticate()           [Doc 04]
        └── run_active_loop()
            ├── select! on {framed, msg_rx}   [Doc 02]
            ├── dispatch_frame()
            │   ├── handle_publish()          [Docs 01, 08]
            │   ├── handle_subscribe()
            │   └── handle_unsubscribe()
            └── error handling                [Doc 05]

nexus-core (library)
├── engine.rs
│   ├── NexusEngine { AtomicU64 counters }   [Doc 01]
│   └── route() → Bytes clone               [Doc 08]
│
├── session.rs
│   └── Connection<S: TypeState>            [Doc 04]
│
├── store.rs
│   └── trait MessageStore (BoxFuture)      [Doc 03]
│
├── topic.rs
│   └── TopicMap { Arc<RwLock<...>> }
│
└── error.rs
    └── NexusError hierarchy                [Doc 05]

nexus-proto (library)
├── codec.rs
│   └── NexusCodec: Decoder + Encoder       [Doc 02]
├── frame.rs
│   └── Frame struct, parse/write
└── command.rs
    └── Command enum

Testing
├── proptest roundtrips                     [Doc 07]
├── trybuild compile-fail state machine     [Doc 04]
└── criterion benchmarks                    [Doc 01, 08]
```

---

## Critical Path 1: Connection Establishment

A client connects. The server must authenticate the connection before any other
commands are processed. Every concept from the course touches this path:

```rust
// nexus-server/src/handler.rs

/// Full connection lifecycle.
///
/// The function signature tells you the state at entry and exit:
/// - At entry: a raw TcpStream (no state machine yet)
/// - At exit: connection is closed (all resources released)
#[tracing::instrument(                         // [Doc 06] Per-connection span
    skip(stream, engine),
    fields(
        conn_id = tracing::field::Empty,
        peer = %stream.peer_addr().unwrap_or("[unknown]".parse().unwrap()),
    )
)]
pub async fn handle_connection(stream: TcpStream, engine: Arc<NexusEngine>) {
    // [Doc 01] Allocate a unique connection ID atomically
    let conn_id = engine.allocate_connection_id();
    tracing::Span::current().record("conn_id", conn_id);

    // [Doc 01] Track active connection count
    engine.metrics.active_connections.fetch_add(1, Ordering::Relaxed);

    tracing::info!("Connection accepted");

    // [Doc 02] Wrap the TCP stream in a codec-framed reader/writer
    let mut framed = Framed::new(
        stream,
        NexusCodec::new(engine.config.max_payload_bytes),
    );

    // [Doc 04] Start in the unauthenticated state — type enforces this
    let unauth: Connection<Unauthenticated> = Connection::new(conn_id);

    // Authentication: must happen before any other command
    let active: Connection<Active> = match wait_for_connect(&mut framed, unauth, &engine).await {
        Ok(conn) => conn,
        Err(e) => {
            tracing::warn!(error = ?e, "Authentication failed");
            // [Doc 05] Send error frame before closing
            let _ = framed.send(Frame::connack_fail(e.as_error_code())).await;
            engine.metrics.active_connections.fetch_sub(1, Ordering::Relaxed);
            return;
        }
    };

    tracing::info!("Authentication successful");

    // [Doc 04] Type transition: unauth consumed, active returned
    // It is now impossible to call unauthenticated-only methods
    run_active_loop(&mut framed, active, engine.clone()).await;

    engine.metrics.active_connections.fetch_sub(1, Ordering::Relaxed);
    tracing::info!("Connection closed");
}
```

---

## Critical Path 2: Message Routing

A publish from one client reaches all subscribers. This is the hot path — every
optimization matters here:

```rust
// nexus-core/src/engine.rs

#[tracing::instrument(skip(self, payload), fields(topic = topic, seq = seq))]
pub async fn route(
    &self,
    topic: &str,
    seq: u64,
    payload: Bytes,    // [Doc 08] Bytes: O(1) clone for all subscribers
) -> Result<usize, NexusError> {
    let topics = self.topics.read().await;

    let topic_state = match topics.get(topic) {
        Some(t) => t,
        None => return Ok(0),
    };

    let mut delivered = 0;
    let mut disconnected = Vec::new();

    for (conn_id, sub) in &topic_state.subscribers {
        // [Doc 08] .clone() on Bytes is an atomic reference count increment
        // Not a memory copy — all 1000 subscribers share the same bytes
        match sub.try_send((seq, payload.clone())) {
            Ok(()) => {
                delivered += 1;
            }
            Err(tokio::sync::mpsc::error::TrySendError::Full(_)) => {
                // [Doc 06] Log at debug — slow subscribers are expected at scale
                tracing::debug!(subscriber_id = conn_id, "Subscriber channel full, dropping message");
            }
            Err(tokio::sync::mpsc::error::TrySendError::Closed(_)) => {
                // Subscriber disconnected — clean up on next write
                disconnected.push(*conn_id);
            }
        }
    }

    // [Doc 06] Include delivery metrics in the log
    tracing::debug!(
        delivered = delivered,
        dropped = topic_state.subscribers.len() - delivered,
        "Message routed"
    );

    // [Doc 01] Atomic increment for total message count
    self.metrics.total_messages_routed.fetch_add(1, Ordering::Relaxed);
    self.metrics.total_bytes_sent.fetch_add(
        payload.len() as u64 * delivered as u64,
        Ordering::Relaxed,
    );

    Ok(delivered)
}
```

---

## Critical Path 3: Graceful Shutdown

Shutdown must be orderly: in-flight messages complete, connections close cleanly,
resources are freed.

```rust
// nexus-server/src/listener.rs
use tokio::sync::watch;

pub async fn serve(engine: Arc<NexusEngine>, addr: &str) -> Result<(), anyhow::Error> {
    let listener = tokio::net::TcpListener::bind(addr)
        .await
        .with_context(|| format!("Failed to bind to {addr}"))?;

    // [Doc 05] anyhow context: adds "what we were doing" to errors
    tracing::info!(addr = addr, "Server listening");

    // Shutdown channel: watch lets all tasks see the signal
    let (shutdown_tx, shutdown_rx) = watch::channel(false);

    // Ctrl+C or SIGTERM triggers shutdown
    tokio::spawn(async move {
        tokio::signal::ctrl_c().await.ok();
        tracing::info!("Shutdown signal received");
        shutdown_tx.send(true).ok();
    });

    loop {
        tokio::select! {
            accept_result = listener.accept() => {
                match accept_result {
                    Ok((stream, _peer)) => {
                        let engine = Arc::clone(&engine);
                        let shutdown = shutdown_rx.clone();
                        tokio::spawn(handle_connection_with_shutdown(stream, engine, shutdown));
                    }
                    Err(e) => {
                        tracing::error!(error = ?e, "Accept error");
                    }
                }
            }

            // Stop accepting when shutdown is signaled
            _ = async {
                let mut rx = shutdown_rx.clone();
                rx.changed().await.ok();
            } => {
                if *shutdown_rx.borrow() {
                    tracing::info!("Stopping accept loop");
                    break;
                }
            }
        }
    }

    // Wait for all active connections to finish (with timeout)
    let active = engine.metrics.active_connections.load(Ordering::Relaxed);
    if active > 0 {
        tracing::info!(active_connections = active, "Waiting for connections to close");
        tokio::time::sleep(std::time::Duration::from_secs(5)).await;
    }

    tracing::info!("Server shut down cleanly");
    Ok(())
}
```

---

## The Testing Pyramid for nexus

Each level tests different aspects:

```
           ╔════════════════╗
           ║  Integration   ║   3 tests: full connect/publish/subscribe cycle
           ╚════════════════╝
         ╔══════════════════════╗
         ║   Property-Based     ║   5 tests: codec roundtrip, routing consistency
         ╚══════════════════════╝
       ╔══════════════════════════════╗
       ║        Unit Tests            ║   20 tests: each function in isolation
       ╚══════════════════════════════╝
     ╔══════════════════════════════════════╗
     ║     Compile-Fail Tests (trybuild)    ║   3 tests: type-state enforced
     ╚══════════════════════════════════════╝
```

```rust
// Unit tests: codec, frame parsing, routing logic
#[tokio::test]
async fn test_subscribe_then_unsubscribe() {
    let map = TopicMap::new();
    let (tx, _rx) = tokio::sync::mpsc::channel(16);
    map.subscribe("test", 1, tx);
    assert_eq!(map.subscriber_count("test"), 1);
    map.unsubscribe("test", 1);
    assert_eq!(map.subscriber_count("test"), 0);
}

// Integration tests: full server lifecycle
#[tokio::test]
async fn test_publish_reaches_subscriber() {
    let engine = Arc::new(NexusEngine::new(test_config()));
    let server_addr = start_test_server(engine.clone()).await;

    // Client A: subscribe
    let mut client_a = TestClient::connect(server_addr).await;
    client_a.authenticate("test-token").await;
    client_a.subscribe("alerts").await;

    // Client B: publish
    let mut client_b = TestClient::connect(server_addr).await;
    client_b.authenticate("test-token").await;
    client_b.publish("alerts", b"test message").await;

    // Client A should receive the message
    let msg = tokio::time::timeout(
        std::time::Duration::from_secs(1),
        client_a.next_message(),
    ).await.expect("Timeout waiting for message").expect("Error reading");

    assert_eq!(msg.payload, b"test message");
}

// Compile-fail tests: type-state enforced
// tests/ui/publish_before_auth.rs — must not compile
// tests/ui/use_after_close.rs — must not compile
```

---

## Performance Expectations

After completing nexus, run these benchmarks to understand where it stands:

```
Connection throughput:    ~50,000 connections/sec  (limited by TCP handshake)
Message routing:          ~1,000,000 messages/sec  (with 10 subscribers per topic)
Per-message latency:      ~5 µs (from publish receipt to subscriber delivery)
Memory per connection:    ~8 KB (stack + channel buffer)
Memory per subscription:  ~64 bytes (channel sender + Arc overhead)
```

These numbers are achievable with the patterns in this course. The critical path:
- Bytes (O(1) clone) for payload sharing
- AtomicU64 (lockless) for sequence numbers
- BufReader + NexusCodec (zero-copy framing) for I/O

The bottleneck in most deployments is not the server but the clients: how fast can
they produce and consume messages.

---

## What You've Built Across the Course

This is the moment to look back at the full journey:

**Section 1:** Types and ownership. `let name: String = String::from("Alice")` and
the difference between `&str` and `String`. Now in Section 9, you use `Bytes` (which
is Arc-backed), `&[u8]` (borrowed slices), and `Vec<u8>` (owned allocations) with
the precision of someone who knows exactly what each costs.

**Section 2:** Structs and traits. `impl Point { fn distance(&self) -> f64 }`. Now
you write `impl Connection<Active>` and `impl MessageStore for InMemoryStore`, where
the type parameter and trait bound are doing real work.

**Section 3:** Error handling. `Result<T, E>`, `?` operator, `match` on `Err`. Now
you design entire error hierarchies with `thiserror`, understand which errors should
propagate and which should be handled locally, and add context with `anyhow`.

**Section 4:** Async/await. `tokio::spawn`, `select!`, `TcpStream`. Now you use
`select!` to fan-in frames from a client and messages from subscribers simultaneously,
and understand why `std::thread::sleep` in async code is wrong.

**Section 5:** HTTP APIs with Axum. REST endpoints, `Json<T>`, middleware. Now you're
building below the HTTP layer — the binary protocol that HTTP itself runs over.

**Section 6:** Larger async systems. Channels, backpressure, structured concurrency.
Now `mpsc::channel(128)` is a deliberate design choice: 128 is the backpressure point
where a slow subscriber stops the publisher rather than accumulating unbounded data.

**Section 7:** Type-level programming. `PhantomData`, type-state, capability tokens.
`Connection<Unauthenticated>` and `Connection<Active>` are the same pattern as
`JobHandle<Pending>` — illegal states cannot be represented.

**Section 8:** Systems programming. Unsafe, FFI, memory-mapped files, binary formats.
The `NexusCodec::decode` function reads bytes with the same care you brought to
parsing ironkv's binary file format.

**Section 9:** All of the above, composing into a real protocol engine.

---

## What Comes Next

nexus is a foundation, not a ceiling. Real production systems built on these patterns:

**Add persistence.** Wire in ironkv (Section 8) as the `MessageStore`. Messages
survive server restarts. Subscribers can replay from any sequence number.

**Add clustering.** Multiple nexus instances, connected by a gossip protocol. When
a client publishes to instance A, instance B's subscribers also receive it.

**Add the Redis protocol.** RESP2 (Redis Serialization Protocol) is a text protocol.
Implement a `RedisCodec` alongside `NexusCodec`. Now nexus accepts Redis clients
and routes their pub/sub traffic.

**Add observability.** Export `NexusMetrics` in Prometheus format. Wire Grafana to
the `/metrics` endpoint. Set an alert when `active_connections` exceeds capacity.

**Build something unrelated.** The skills you have now — ownership, async, type-state,
binary protocols, structured logging, property-based testing — transfer directly to
embedded Rust, WebAssembly, game development, scientific computing, compiler development.

The Rust ecosystem is vast. Pick the direction that interests you and go deeper.

---

## Exercises: Building nexus End-to-End

Work through the milestones in order. Each milestone produces a running, testable system.

**Milestone 1 — Frame Parsing (no networking)**
Implement `NexusCodec` (`Decoder` + `Encoder`). Write unit tests for all command types.
No TCP yet.

**Milestone 2 — TCP Echo Server**
Accept connections with `tokio::net::TcpListener`. For each connection, spawn a task,
wrap the stream in `Framed::new(stream, NexusCodec::new(...))`, and echo every frame back.
Test with `nc` or a test client.

**Milestone 3 — Authentication**
Add CONNECT/CONNACK handling. Implement `Connection<Unauthenticated>::authenticate()`.
Add tracing with `#[instrument]` on `handle_connection`. Write an integration test
that connects, sends CONNECT with the right token, and verifies CONNACK success.

**Milestone 4 — Publish and Subscribe**
Implement `TopicMap`. On PUBLISH, route to all subscribers. On SUBSCRIBE, register a
channel receiver. On DELIVER, send from the channel to the framed sink.

Write the property-based routing consistency test from Doc 07.

**Milestone 5 — Graceful Shutdown**
Add SIGTERM / Ctrl+C handling with `tokio::sync::watch`. Verify that in-flight messages
complete before the server exits. Integration test: publish 100 messages, send SIGTERM
mid-publish, verify no messages are lost.

**Stretch — Message Persistence**
Wire in an `InMemoryStore` implementing `MessageStore`. Persist every published message.
On SUBSCRIBE, replay all stored messages from seq 0. Write a test that subscribes after
publishing, and verifies all prior messages are received.

---

## Course Completion Checklist

Before calling nexus production-ready:

- [ ] All frames parse correctly (proptest roundtrip passes)
- [ ] Arbitrary bytes don't panic the decoder (parser robustness test passes)
- [ ] Compile-fail tests verify type-state invariants
- [ ] Integration test: publish reaches subscriber end-to-end
- [ ] Graceful shutdown: all in-flight messages delivered before exit
- [ ] Backpressure: slow subscriber is dropped, not blocked indefinitely
- [ ] Structured logging: every connection has a `conn_id` span
- [ ] No auth tokens, passwords, or full payloads in log output
- [ ] `cargo clippy --workspace -- -D warnings` passes clean
- [ ] `cargo audit` passes (no known vulnerabilities in deps)
- [ ] Release binary is statically linked musl: `cargo build --release --target x86_64-unknown-linux-musl`
- [ ] README explains: what nexus does, how to run it, the protocol format, configuration

---

## The Systems Programmer's Mindset: Final Edition

You started this course as a C developer who knew how memory works. You finish it
as a Rust developer who has the compiler verify what you used to verify with
documentation and discipline.

The patterns you've learned:
- **Make illegal states unrepresentable.** `Connection<Unauthenticated>` has no
  `publish()` method. The type system is the spec.
- **Share, don't copy.** `Bytes` and `Arc` let 1000 subscribers receive the same
  message without 1000 copies. O(1) clone is always better than O(n).
- **Measure before optimizing.** `dhat` and Criterion show where time actually goes.
  The bottleneck is almost never where you think.
- **Log what you need, not what you have.** `conn_id` in every log line is worth
  more than the full frame payload in some log lines.
- **The invariants are load-bearing.** "Messages are delivered in sequence order"
  is not a wish. It's a property that your `AtomicU64` sequence counter plus
  ordered delivery enforces. Property-based tests verify it.

These instincts transfer to every systems project you'll work on.

```
$ nexus-server --bind 0.0.0.0:9090
{"level":"INFO","message":"Server listening","addr":"0.0.0.0:9090"}
{"level":"INFO","message":"Connection accepted","conn_id":1,"peer":"127.0.0.1:54321"}
{"level":"INFO","message":"Authentication successful","conn_id":1}
{"level":"DEBUG","message":"Message routed","topic":"alerts","seq":1,"delivered":3}
{"level":"INFO","message":"Connection closed","conn_id":1}
```

Ship it.
