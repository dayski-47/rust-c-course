# Doc 02 - Streams, AsyncRead, and AsyncWrite

🟡 A TCP connection is a byte stream. The client sends bytes; you read them,
interpret them as protocol frames, and act on each frame. The question is: what
abstractions handle this well in async Rust? This doc covers `AsyncRead`,
`AsyncWrite`, tokio's codec system, and the `Stream` trait that connects them.

---

## The Fundamental Problem: Bytes vs Frames

TCP is a stream protocol - it delivers a sequence of bytes with no inherent message
boundaries. When your client sends two frames back-to-back, they might arrive as:
- Two separate `read()` calls (one frame each)
- One `read()` call containing both frames
- Three `read()` calls (first frame in two reads, second frame in one)

**In C**, this is the "framing problem" and it's where most protocol implementation
bugs live:

```c
// Naive C - will silently fail on partial reads
ssize_t n = read(fd, buf, sizeof(buf));
process_frame(buf, n);  // WRONG: n might be 7, not 16

// Correct C loop for full frame reads:
size_t total_read = 0;
while (total_read < HEADER_SIZE) {
    ssize_t n = read(fd, buf + total_read, HEADER_SIZE - total_read);
    if (n <= 0) break;
    total_read += n;
}
uint32_t payload_len = ntohl(*(uint32_t*)(buf + 4));
// Repeat for the payload...
```

Rust's async ecosystem solves this with `BufReader`, `AsyncReadExt`, and most
importantly `tokio_util::codec` - a battle-tested framing abstraction.

---

## Why Poll-Based I/O Exists

Tokio runs thousands of connections on a small thread pool. For this to work, no
connection's I/O can block a thread. When a `read()` would normally block (no data
available yet), an async read instead yields back to the Tokio runtime and registers
a wake-up callback with the OS. The thread is freed to run other tasks. When the OS
signals that data is available, Tokio reschedules the task.

This is the difference between synchronous `Read`/`Write` (from `std::io`) and
`AsyncRead`/`AsyncWrite` (from `tokio::io`). Synchronous traits block the calling
thread. Async traits return `Poll::Pending` and arrange a wake-up.

## AsyncRead and AsyncWrite

The foundation is two traits from `tokio::io`:

```rust
// Conceptual shapes (simplified)
trait AsyncRead {
    // Poll::Ready(Ok(())) - data written into buf
    // Poll::Pending        - no data yet; this task will be woken when data arrives
    // Poll::Ready(Err(e)) - connection error
    fn poll_read(self: Pin<&mut Self>, cx: &mut Context<'_>, buf: &mut ReadBuf<'_>)
        -> Poll<io::Result<()>>;
}

trait AsyncWrite {
    // Poll::Ready(Ok(n))  - n bytes written (may be less than buf.len())
    // Poll::Pending        - write buffer full; wake when drained
    // Poll::Ready(Err(e)) - connection error
    fn poll_write(self: Pin<&mut Self>, cx: &mut Context<'_>, buf: &[u8])
        -> Poll<io::Result<usize>>;
    fn poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<()>>;
    fn poll_shutdown(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<()>>;
}
```

You almost never call `poll_read` directly. Extension traits like `AsyncReadExt`
add `async fn read()`, `async fn read_exact()`, and others that wrap the poll loop
in an awaitable future. The poll methods are the mechanism; the extension methods
are what you use.

`TcpStream` implements both. Reading from a `TcpStream` in async Rust:

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

async fn read_frame_naive(stream: &mut TcpStream) -> io::Result<Vec<u8>> {
    let mut header = [0u8; 16];
    stream.read_exact(&mut header).await?;  // Reads exactly 16 bytes (handles partial reads)

    let payload_len = u32::from_be_bytes(header[4..8].try_into().unwrap()) as usize;
    let mut payload = vec![0u8; payload_len];
    stream.read_exact(&mut payload).await?;

    let mut frame = header.to_vec();
    frame.extend_from_slice(&payload);
    Ok(frame)
}
```

`read_exact` handles the partial-read problem. It loops internally until the buffer
is full or an error occurs. This is the async equivalent of the C loop above, but
in one line.

---

## BufReader: Buffered Reads

Reading 16 bytes for a header, then N bytes for the payload, makes two syscalls per
frame. For high-throughput servers, syscall overhead adds up. `BufReader` batches
reads:

```rust
use tokio::io::BufReader;
use tokio::net::TcpStream;

let stream = TcpStream::connect("127.0.0.1:9090").await?;
let mut reader = BufReader::with_capacity(65536, stream);  // 64 KB buffer

// Now reads are served from the buffer; syscalls happen only when the buffer empties
let mut header = [0u8; 16];
reader.read_exact(&mut header).await?;
```

`BufReader` reads up to `capacity` bytes per syscall. If the next frame's header is
already in the buffer, `read_exact` returns immediately without a syscall.

For nexus, wrap the reading half of the TCP stream in a `BufReader`:

```rust
use tokio::net::TcpStream;
use tokio::io::{BufReader, BufWriter};

async fn handle_connection(stream: TcpStream) {
    let (read_half, write_half) = stream.into_split();
    let reader = BufReader::with_capacity(64 * 1024, read_half);
    let writer = BufWriter::with_capacity(64 * 1024, write_half);
    // ...
}
```

`into_split()` splits the TCP stream into independent read and write halves. This
lets you read and write concurrently - two async tasks or two sides of a `select!`.

---

## tokio_util::codec: The Framing Abstraction

Manually calling `read_exact` twice per frame is fine but verbose. `tokio_util::codec`
provides a cleaner abstraction: you define how to parse bytes into frames, and the
codec handles buffering and partial reads for you.

Two traits to implement:

```rust
use tokio_util::codec::{Decoder, Encoder};
use bytes::{Bytes, BytesMut};

// nexus-proto/src/codec.rs

pub struct NexusCodec {
    max_payload_bytes: usize,
}

impl Decoder for NexusCodec {
    type Item = Frame;
    type Error = ProtocolError;

    fn decode(&mut self, src: &mut BytesMut) -> Result<Option<Frame>, ProtocolError> {
        // Need at least the 16-byte header
        if src.len() < 16 {
            return Ok(None);  // Signal: "not enough data yet, come back later"
        }

        // Peek at the header without consuming bytes
        let version = src[0];
        if version != 0x01 {
            return Err(ProtocolError::UnsupportedVersion { got: version });
        }

        let command = src[1];
        let payload_len = u32::from_be_bytes(src[4..8].try_into().unwrap()) as usize;

        if payload_len > self.max_payload_bytes {
            return Err(ProtocolError::PayloadTooLarge {
                limit: self.max_payload_bytes,
                got: payload_len,
            });
        }

        let total_len = 16 + payload_len;

        if src.len() < total_len {
            // Not enough data yet - tell the codec framework to reserve more space
            src.reserve(total_len - src.len());
            return Ok(None);
        }

        // We have a complete frame - consume the bytes
        let frame_bytes: Bytes = src.split_to(total_len).freeze();

        let sequence_id = u64::from_be_bytes(frame_bytes[8..16].try_into().unwrap());
        let payload = frame_bytes.slice(16..);

        Ok(Some(Frame {
            version,
            command: Command::from_byte(command)?,
            payload_len: payload_len as u32,
            sequence_id,
            payload,
        }))
    }
}

impl Encoder<Frame> for NexusCodec {
    type Error = ProtocolError;

    fn encode(&mut self, frame: Frame, dst: &mut BytesMut) -> Result<(), ProtocolError> {
        dst.reserve(16 + frame.payload.len());

        dst.put_u8(frame.version);
        dst.put_u8(frame.command.to_byte());
        dst.put_u16(0);  // flags
        dst.put_u32(frame.payload.len() as u32);
        dst.put_u64(frame.sequence_id);
        dst.put_slice(&frame.payload);

        Ok(())
    }
}
```

**How `Framed` buffers internally**: `Framed` owns a `BytesMut` buffer. When you
call `framed.next().await`:

1. The runtime polls the inner `TcpStream` for new data
2. New bytes are appended to the internal `BytesMut`
3. `NexusCodec::decode()` is called with the accumulated buffer
4. If `decode()` returns `Ok(None)`, the runtime waits for more bytes and repeats
5. If `decode()` returns `Ok(Some(frame))`, the bytes are consumed and the frame is returned
6. If `decode()` returns `Err(e)`, the stream ends with an error

You don't write this loop. `Framed` is the loop.

Then in the connection handler:

```rust
use tokio_util::codec::Framed;
use futures::{SinkExt, StreamExt};

async fn handle_connection(stream: TcpStream) {
    let codec = NexusCodec { max_payload_bytes: 1024 * 1024 };
    let mut framed = Framed::new(stream, codec);

    while let Some(result) = framed.next().await {
        match result {
            Ok(frame) => {
                // Frame is fully parsed - handle it
                handle_frame(&mut framed, frame).await;
            }
            Err(ProtocolError::PayloadTooLarge { .. }) => {
                // Send error frame and close
                let _ = framed.send(Frame::error(ErrorCode::PayloadTooLarge)).await;
                break;
            }
            Err(e) => {
                tracing::warn!("Protocol error: {e}");
                break;
            }
        }
    }
}
```

`Framed` wraps the TCP stream with the codec. `framed.next().await` reads bytes
from the stream, runs them through `decode()`, and returns a complete `Frame` when
enough bytes are available. `framed.send(frame).await` encodes and writes.

**The key insight**: you wrote `decode()` once. The framework handles the rest -
buffering, partial reads, backpressure. This is the same abstraction that hyper,
tonic, and tokio-postgres use internally.

---

## The Stream Trait

`framed.next()` works because `Framed` implements `Stream`. The `Stream` trait is
to `Iterator` what `Future` is to a single value:

```rust
// Synchronous, pull-based, multiple values
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// Asynchronous, pull-based, multiple values
trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

The three possible results from `poll_next`:

| Result | Meaning |
|--------|---------|
| `Poll::Ready(Some(item))` | An item is available - process it |
| `Poll::Pending` | No item yet - suspend the task, reschedule when the source has data |
| `Poll::Ready(None)` | Stream is exhausted - no more items will ever be produced |

The `None` case is how `Framed` signals a clean TCP disconnect - the stream ends,
and the connection loop exits.

`Stream` yields items asynchronously - each call to `poll_next` either returns a
value, says "not ready yet" (and schedules a wakeup), or ends the stream.

### StreamExt: The Stream Combinators

The `StreamExt` trait (from `futures`) adds combinator methods, the same way
`Iterator` has `map`, `filter`, etc.:

```rust
use futures::StreamExt;

// Map: transform each frame
let frame_stream = framed.map(|result| result.map(|f| (f.command, f.payload)));

// Filter: only process PUBLISH frames
let publish_stream = framed.filter_map(|result| async move {
    match result {
        Ok(f) if f.command == Command::Publish => Some(Ok(f)),
        Ok(_) => None,   // Discard non-PUBLISH frames
        Err(e) => Some(Err(e)),
    }
});

// Timeout: error if no frame arrives within 30 seconds
use tokio_stream::StreamExt as TokioStreamExt;  // tokio-stream adds timeout
let timed = framed.timeout(Duration::from_secs(30));

// Fold: accumulate statistics
let stats = framed
    .filter_map(|r| futures::future::ready(r.ok()))
    .fold(FrameStats::default(), |mut acc, frame| async move {
        acc.total += 1;
        acc.bytes += frame.payload.len() as u64;
        acc
    })
    .await;
```

For nexus, the connection handler is a stream pipeline:

```
TcpStream bytes
    → NexusCodec::decode (BytesMut → Frame)
    → dispatch on Command type
    → respond with Frame
    → NexusCodec::encode (Frame → BytesMut)
    → TcpStream bytes
```

---

## Generating Streams with `async_stream`

Sometimes you need to produce a stream from async logic. The `async_stream` crate
provides an `async fn`-like syntax for this:

```rust
use async_stream::stream;
use futures::Stream;

// nexus-core/src/topic.rs
// Produce a stream of messages for a subscriber
fn messages_for_subscriber(
    mut rx: tokio::sync::mpsc::Receiver<(u64, Bytes)>,
) -> impl Stream<Item = Frame> {
    stream! {
        while let Some((seq, payload)) = rx.recv().await {
            yield Frame::deliver(seq, payload);
        }
    }
}
```

This is the async equivalent of a generator function. The `yield` keyword suspends
the generator and produces a value. The `while let` loop ends when the channel
closes - the stream ends naturally.

In the connection handler, merge the incoming frame stream with the subscriber
message stream:

```rust
use tokio::select;
use futures::StreamExt;

async fn active_connection_loop(
    mut framed: Framed<TcpStream, NexusCodec>,
    mut message_rx: tokio::sync::mpsc::Receiver<(u64, Bytes)>,
    topic: &str,
) {
    loop {
        select! {
            // Handle incoming frames from the client
            Some(result) = framed.next() => {
                match result {
                    Ok(frame) => { /* process client frame */ }
                    Err(e) => { tracing::warn!("Client error: {e}"); break; }
                }
            }

            // Deliver messages to the client
            Some((seq, payload)) = message_rx.recv() => {
                if framed.send(Frame::deliver(seq, payload)).await.is_err() {
                    break;  // Client disconnected
                }
            }

            else => break,  // Both streams exhausted
        }
    }
}
```

`select!` polls both futures concurrently and acts on whichever is ready first. This
is the correct way to "fan-in" multiple async event sources - no busy-waiting,
no polling interval.

---

## Backpressure: Why Channels Have Sizes

When subscribers can't consume messages fast enough, the server must not buffer
them infinitely (that leads to OOM). Tokio's `mpsc::channel(capacity)` enforces
backpressure:

```rust
// nexus-core/src/topic.rs

pub struct TopicSubscriber {
    tx: tokio::sync::mpsc::Sender<(u64, Bytes)>,
}

impl TopicSubscriber {
    pub fn new(capacity: usize) -> (Self, tokio::sync::mpsc::Receiver<(u64, Bytes)>) {
        let (tx, rx) = tokio::sync::mpsc::channel(capacity);
        (TopicSubscriber { tx }, rx)
    }

    /// Returns false if the subscriber is slow or disconnected.
    pub fn try_deliver(&self, seq: u64, payload: Bytes) -> bool {
        // try_send doesn't wait - returns Err if the channel is full
        match self.tx.try_send((seq, payload)) {
            Ok(()) => true,
            Err(tokio::sync::mpsc::error::TrySendError::Full(_)) => {
                // Subscriber is slow - could drop the message or disconnect them
                tracing::warn!("Subscriber channel full - dropping message");
                false
            }
            Err(tokio::sync::mpsc::error::TrySendError::Closed(_)) => {
                // Subscriber disconnected
                false
            }
        }
    }
}
```

`try_send` is non-blocking - if the channel is full, the message is dropped (or the
subscriber is disconnected). `send().await` would block the publisher, which could
cascade into blocking all publishers waiting for slow subscribers.

The capacity of the channel is the backpressure knob. A subscriber with 1000-message
capacity can burst; a subscriber with 10-message capacity is tightly flow-controlled.

**When to use `try_send` vs `send().await`:**

| Situation | Use |
|-----------|-----|
| Publisher cannot afford to wait (broadcast router) | `try_send` - drop or disconnect slow subscribers |
| Publisher must deliver every message (reliable queue) | `send().await` - apply backpressure to the sender |
| Checking capacity without committing | `sender.capacity()` before `try_send` |

In nexus, the router uses `try_send`: one slow subscriber should not block messages
to all other subscribers on the same topic. For durable delivery (guaranteed
delivery queues), the architecture requires a different design - bounded `send().await`
with the publisher as the flow-controlled party.

---

## Splitting Streams for Reading and Writing

`Framed` doesn't support concurrent read and write from two tasks because `next()`
and `send()` both take `&mut self`. For true concurrent I/O, split:

```rust
use tokio_util::codec::Framed;
use futures::{SinkExt, StreamExt};

async fn handle_connection(stream: TcpStream) {
    let framed = Framed::new(stream, NexusCodec::new());
    let (mut sink, mut stream) = framed.split();

    // Reader task: receive frames from client
    let reader_task = tokio::spawn(async move {
        while let Some(frame) = stream.next().await {
            // Process incoming frames
        }
    });

    // Writer task: send frames to client
    let (tx, mut rx) = tokio::sync::mpsc::channel::<Frame>(32);
    let writer_task = tokio::spawn(async move {
        while let Some(frame) = rx.recv().await {
            if sink.send(frame).await.is_err() {
                break;
            }
        }
    });

    // Hold onto `tx` to send frames from the reader task
    // ...
}
```

This is the "split actor" pattern - one task owns the read half, one owns the write
half. They communicate via an `mpsc` channel. The channel provides backpressure on
writes.

---

## C Comparison: The Framing Loop

The equivalent in C for a single-threaded server using `epoll`:

```c
// C: manual frame accumulation buffer per connection
struct connection {
    int fd;
    uint8_t buf[65536];
    size_t buf_len;
    size_t expected;     // 0 = reading header, >0 = reading payload
};

void on_readable(struct connection *conn) {
    ssize_t n = read(conn->fd, conn->buf + conn->buf_len,
                     sizeof(conn->buf) - conn->buf_len);
    if (n <= 0) { close_connection(conn); return; }
    conn->buf_len += n;

    while (conn->buf_len >= 16) {  // Have header?
        uint32_t payload_len = ntohl(*(uint32_t*)(conn->buf + 4));
        if (conn->buf_len < 16 + payload_len) break;  // Wait for payload

        process_frame(conn->buf, 16 + payload_len);

        // Shift buffer
        memmove(conn->buf, conn->buf + 16 + payload_len,
                conn->buf_len - 16 - payload_len);
        conn->buf_len -= 16 + payload_len;
    }
}
```

Every line of this exists in Rust's `BufReader` + `NexusCodec::decode()`. The `memmove`
is `src.split_to()`. The `while` loop is the decoder's retry logic. The `if (n <= 0)`
is the `Err` branch. The difference is that in Rust, this logic is written once in
`NexusCodec`, tested once, and reused by every nexus connection.

---

## Exercises

**Exercise 1 - Implement NexusCodec**

Implement `Decoder for NexusCodec` so that:
- `decode()` returns `Ok(None)` when fewer than 16 bytes are available
- `decode()` returns `Ok(None)` when the payload is incomplete (but the header is present)
- `decode()` returns `Err(ProtocolError::UnsupportedVersion)` when version != 0x01
- `decode()` returns `Ok(Some(Frame))` for a complete, valid frame

Write a unit test that encodes a frame with the `Encoder` impl, then decodes it in
three separate `decode()` calls (to simulate partial reads): 5 bytes, 11 bytes, and
then the payload bytes.

**Exercise 2 - Timeout on Idle Connections**

Add an inactivity timeout to the connection frame stream. Using `tokio_stream::StreamExt::timeout`,
wrap `framed.next()` so that connections that send no frames in 30 seconds receive
a `DISCONNECT` frame and are closed. Write a test using `tokio::time::pause()` and
`tokio::time::advance()` to simulate the 30-second timeout without waiting.

**Exercise 3 - Split Reader/Writer**

Refactor `handle_connection` to use `framed.split()` so:
- A reader task processes incoming frames and sends messages to a channel
- A writer task receives from the channel and sends frames
- Both tasks are spawned with `tokio::spawn`

Verify that a slow writer (add a `tokio::time::sleep(50ms)` in the writer) does not
block the reader from receiving new frames.

---

## Checklist

- [ ] All TCP reading uses `BufReader` or a codec - no manual partial-read loops
- [ ] `Framed` + codec for all protocol parsing
- [ ] `into_split()` when reading and writing happen in separate tasks
- [ ] Bounded `mpsc` channels for backpressure - never `unbounded()`
- [ ] `try_send` for non-blocking delivery to slow subscribers
- [ ] `select!` to fan-in multiple event sources (incoming frames + outgoing messages)
- [ ] `StreamExt::timeout` on the outer frame stream to detect idle connections
