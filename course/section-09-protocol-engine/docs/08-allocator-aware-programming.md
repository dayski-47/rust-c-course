# Doc 08 - Allocator-Aware Programming

🟡 Every message nexus routes allocates memory. A tiny message going to 1,000
subscribers might trigger 1,000 `Vec<u8>` allocations, one per subscriber. Or it
might trigger one allocation, shared by all 1,000 subscribers. The difference is
`Bytes` - an Arc-backed reference-counted byte buffer that clones in O(1). This
doc covers memory-aware programming for nexus: when to allocate, how to share, and
how to measure.

---

## How Rust Allocates

Every `Vec`, `Box`, `String`, and `Arc` eventually calls the global allocator -
usually jemalloc or the system malloc. An allocation costs:
- A call into the allocator (~10–100 ns depending on contention)
- A cache miss when accessing newly allocated memory
- A deallocation when the value is dropped (same cost as allocation)

This overhead is invisible for occasional allocations. It becomes significant when
you're routing 100,000 messages per second and each message triggers many allocations.

**In C**, you manage this with custom allocators, arena allocators (`apr_pool_t`
in Apache, `slabs` in nginx), or explicit reuse. In Rust, the same techniques apply,
but the type system helps ensure you don't use freed memory.

---

## The Problem: Cloning Messages for Subscribers

The naive routing implementation clones the message payload for every subscriber:

```rust
// ❌ Allocates once per subscriber
pub fn route(&self, topic: &str, seq: u64, payload: Vec<u8>) {
    let subs = self.subscriptions.get(topic);
    for sub in subs {
        sub.send((seq, payload.clone())).ok();  // clone() allocates a new Vec<u8>
    }
}
```

1000 subscribers × 1 KB message × 100,000 messages/sec = 100 GB/sec of allocations.
This is not achievable on any system.

---

## `bytes::Bytes`: O(1) Clone

The `bytes` crate provides `Bytes` - an Arc-backed reference-counted byte buffer:

```toml
[dependencies]
bytes = "1"
```

```rust
use bytes::Bytes;

// Bytes is like Arc<[u8]> - cloning increments a reference count
let payload = Bytes::from(vec![0u8; 1024]);  // One allocation: 1024 bytes + Arc overhead

let clone1 = payload.clone();  // No allocation: just increments a counter
let clone2 = payload.clone();  // No allocation: increments again

// All three point to the same memory
// Memory freed when the last Bytes is dropped

// Slice without copying
let slice = payload.slice(0..512);  // No allocation: offset + length into the same buffer
```

For nexus routing:

```rust
// ✅ One allocation, no matter how many subscribers
pub async fn route(&self, topic: &str, seq: u64, payload: Bytes) {
    let subs = self.subscriptions.read().await;
    if let Some(subs_for_topic) = subs.get(topic) {
        for sub in subs_for_topic.values() {
            // .clone() on Bytes is O(1) - atomic reference count increment
            sub.try_send((seq, payload.clone())).ok();
        }
    }
}
```

The message payload is allocated once when received from the client, then shared by
reference among all subscribers. Each `Bytes::clone()` is a single atomic increment -
about 5 ns on x86-64.

---

## Building Bytes Efficiently

`Bytes` is immutable after creation. When building frames to send, use `BytesMut`
(the mutable version), then freeze it:

```rust
use bytes::{BufMut, BytesMut, Bytes};

pub fn encode_deliver_frame(seq: u64, payload: &Bytes) -> Bytes {
    let total_len = 16 + payload.len();
    let mut buf = BytesMut::with_capacity(total_len);  // Pre-allocate exact size

    buf.put_u8(0x01);       // version
    buf.put_u8(0x07);       // command: DELIVER
    buf.put_u16(0);         // flags
    buf.put_u32(payload.len() as u32);
    buf.put_u64(seq);
    buf.put_slice(payload);  // No copy if payload is contiguous

    buf.freeze()  // Convert BytesMut → Bytes (O(1), no copy)
}
```

`BytesMut::with_capacity(n)` pre-allocates exactly n bytes. Combined with `buf.freeze()`,
this encodes a frame with one allocation and zero copies of the payload bytes.

---

## Arc-Based Sharing Without bytes

When `Bytes` doesn't fit your use case (e.g., you're sharing a large struct, not raw bytes),
`Arc<T>` provides the same O(1) clone semantics:

```rust
use std::sync::Arc;

pub struct Message {
    pub topic: String,
    pub seq: u64,
    pub payload: Bytes,
    pub published_at: std::time::Instant,
    pub publisher_id: u64,
}

// Wrap in Arc to share without cloning the full struct
pub type SharedMessage = Arc<Message>;

// Publishing: one allocation
let msg = Arc::new(Message {
    topic: "alerts".into(),
    seq: engine.next_message_seq(),
    payload: received_payload,
    published_at: std::time::Instant::now(),
    publisher_id: conn.id,
});

// Routing to 1000 subscribers: 1000 Arc clones = 1000 atomic increments
for sub in subscribers {
    sub.send(Arc::clone(&msg)).ok();
}
```

Each subscriber gets a reference to the same `Message` in memory. When the last
subscriber drops its reference, the message is freed.

---

## Pre-Allocation: Reserving Capacity

Every `Vec::push()` past capacity triggers a reallocation. For collections that grow
predictably, pre-allocate:

```rust
// ❌ Reallocates multiple times as the vec grows
let mut delivered_to: Vec<u64> = Vec::new();
for (id, sub) in subscribers {
    if sub.try_send(msg.clone()).is_ok() {
        delivered_to.push(*id);  // May reallocate
    }
}

// ✅ One allocation for the expected size
let mut delivered_to: Vec<u64> = Vec::with_capacity(subscribers.len());
for (id, sub) in &subscribers {
    if sub.try_send(msg.clone()).is_ok() {
        delivered_to.push(*id);  // Never reallocates
    }
}
```

For nexus's subscriber tracking, pre-allocate the subscriber list based on the
typical topic fanout:

```rust
pub struct TopicState {
    // Typical topics have 1-100 subscribers
    subscribers: HashMap<u64, mpsc::Sender<SharedMessage>>,
}

impl TopicState {
    pub fn new() -> Self {
        Self {
            subscribers: HashMap::with_capacity(16),  // Pre-allocate for typical case
        }
    }
}
```

`HashMap::with_capacity(16)` avoids the first three or four rehashes for topics
with typical subscriber counts.

---

## The Bump Allocator: `bumpalo`

For short-lived allocations in a hot path (e.g., parsing a frame to route it, then
discarding the parsed data), a bump allocator is dramatically faster than the
system malloc:

```toml
[dev-dependencies]  # or dependencies if used in production hot paths
bumpalo = "3"
```

```rust
use bumpalo::Bump;

// nexus-core/src/routing.rs
// For per-request allocations that live only during frame processing:

pub fn process_frame_with_arena(frame: Bytes) {
    // The arena allocates everything in one contiguous block
    let arena = Bump::new();

    // All allocations here come from the arena - no syscalls
    let parsed_topic = arena.alloc_str(&parse_topic(&frame));
    let headers: &[u8] = arena.alloc_slice_copy(&frame[..16]);

    // Route the message...
    route_message(parsed_topic, &frame[16..]);

    // arena drops here: frees everything in one operation (just reset a pointer)
}
```

A bump allocator works by keeping a pointer into a pre-allocated block and bumping
it forward for each allocation. Freeing is O(1) - just reset the pointer to zero.
There's no per-object bookkeeping, no fragmentation, no free list.

**When to use bumpalo:**
- Allocations that all have the same lifetime (created together, freed together)
- Hot paths where allocation cost is measurable
- Request processing (allocate per-request, free at end of request)

**When NOT to use bumpalo:**
- Long-lived objects with different lifetimes
- Objects that need to outlive the arena
- Library code where the caller controls lifetimes

For nexus, the most natural use is parsing: parse a frame into typed data, route it,
then free the parsed data at once.

---

## Measuring Allocations

Before optimizing, measure. The `dhat` profiler (or `heaptrack` on Linux) shows
allocation hotspots:

```toml
# nexus-server/Cargo.toml
[dev-dependencies]
dhat = "0.3"
```

```rust
// nexus-server/src/main.rs
#[cfg(feature = "dhat-heap")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    #[cfg(feature = "dhat-heap")]
    let _profiler = dhat::Profiler::new_heap();

    // ... run server ...
}
```

```bash
cargo run --features dhat-heap &
# Send some messages to the server
kill -SIGTERM $!
# dhat outputs a dhat-heap.json file
# Open it with https://nnethercote.github.io/dh_view/dh_view.html
```

The report shows: how many allocations happened, where they happened (stack trace),
and how long they lived. Hot allocation sites are good candidates for optimization.

---

## When Allocation Doesn't Matter

Not every allocation needs optimization. The rules of thumb:

**Optimize allocation when:**
- The hot path (per-message routing) allocates in a loop
- Profiling shows >10% of time in allocator code
- The allocation count scales with load (N subscribers × M messages)

**Don't optimize allocation when:**
- Startup/initialization (one-time cost)
- Error paths (infrequent)
- Configuration loading (milliseconds, not microseconds)
- Strings for log messages (logging is already the bottleneck)

Premature allocation optimization adds complexity without measurable benefit. Measure
first. `cargo bench` + `dhat` shows where time is actually spent.

---

## nexus Message Lifecycle

Here's the complete allocation story for one nexus message:

```
Client sends: PUBLISH "alerts" 47-byte payload

1. TcpStream → BufReader: bytes in reader buffer (no allocation - buffer pre-allocated)

2. NexusCodec::decode: 
   - Reads header from buffer (no allocation - reads into existing BytesMut)
   - frame.payload = Bytes::from(src.split_to(16 + payload_len).freeze())
     → One allocation: 63 bytes (16 header + 47 payload)
   
3. parse_publish_payload(frame.payload):
   - topic: frame.payload.slice(2..topic_len+2) → No allocation (Bytes slice)
   - payload: frame.payload.slice(topic_len+2..) → No allocation (Bytes slice)

4. engine.route("alerts", seq, payload):
   - Per subscriber: payload.clone() → No allocation (atomic increment)
   - 1000 subscribers → 1000 atomic increments, 0 heap allocations

5. Subscribers receive: (seq: u64, payload: Bytes)
   - Build DELIVER frame: BytesMut::with_capacity(63) → One allocation per subscriber

6. Write to TcpStream: No allocation (writes from BytesMut directly)

Total allocations per message per subscriber: 2 (one for receive, one for send)
Total allocations per message regardless of subscriber count: 1 (for the received payload)
```

Without `Bytes`, step 4 would be 1000 allocations. With `Bytes`, it's 1000 atomic
increments and 1000 allocations in step 5 (which is irreducible - you need one frame
per subscriber).

---

## Global Allocator Replacement

For workloads where allocation is a bottleneck, replacing the system allocator with
jemalloc or mimalloc can improve throughput by 10–30%:

```toml
[dependencies]
tikv-jemallocator = "0.6"
```

```rust
// nexus-server/src/main.rs
#[cfg(not(target_env = "msvc"))]
#[global_allocator]
static GLOBAL: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;
```

jemalloc is tuned for multi-threaded workloads: it uses thread-local caches to
avoid lock contention in the allocator. For nexus, which handles thousands of
connections concurrently, jemalloc typically outperforms the system allocator.

Don't add this until you've profiled and confirmed that allocator contention is
the bottleneck. It adds a dependency and a larger binary.

---

## Exercises

**Exercise 1 - Bytes in the Routing Path**

In `TopicMap::route()`, change the function signature from `payload: Vec<u8>` to
`payload: Bytes`. Update all callers. Run the test suite.

Write a benchmark comparing the old `Vec<u8>` clone approach vs the new `Bytes` clone
approach with 100 subscribers. Use `criterion::black_box()` to prevent the compiler
from optimizing away the clones. The `Bytes` version should be roughly 100× faster.

**Exercise 2 - Pre-Allocate Subscriber Lists**

Add `HashMap::with_capacity(16)` to `TopicState::new()`. Use `dhat` (or count
allocations manually) to verify that inserting the first 16 subscribers triggers zero
rehashes. Then insert 17 subscribers and observe the single rehash.

For a topic with 1000 subscribers, measure the total allocation count with
`with_capacity(0)` vs `with_capacity(1024)`. Report the reduction in rehash operations.

**Exercise 3 - Build Frame without Extra Copy**

Implement `encode_deliver_frame(seq: u64, payload: &Bytes) -> Bytes` using
`BytesMut::with_capacity(16 + payload.len())` and a series of `buf.put_*` calls.
Verify with a unit test that:
1. The output is exactly `16 + payload.len()` bytes
2. The payload bytes appear at offset 16 in the output
3. The sequence number at bytes 8..16 decodes correctly with `u64::from_be_bytes`

---

## Checklist

- [ ] Message payloads use `bytes::Bytes` - not `Vec<u8>` - for sharing across subscribers
- [ ] `Bytes::clone()` used in routing loop (O(1), not O(n))
- [ ] Frame encoding uses `BytesMut::with_capacity(exact_size)` before `freeze()`
- [ ] Long-lived subscriber data uses `Arc<T>` for shared structures
- [ ] `Vec::with_capacity()` for collections of known size
- [ ] Allocation profiled with `dhat` before optimizing
- [ ] `bumpalo` only for proven hotspots with same-lifetime allocations
- [ ] jemalloc only if profiling confirms allocator contention
