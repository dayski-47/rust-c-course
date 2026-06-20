# 06 — Streams and Async Iteration 🟡

An `Iterator` gives you many values synchronously, one at a time. A `Future` gives you one value asynchronously. A `Stream` combines both ideas: many values arriving asynchronously over time.

If you're fetching database rows and want to process them as they arrive without loading everything into memory — that's a Stream. If a Redis pub/sub channel keeps delivering events — that's a Stream. If you want to push server-sent events to a browser — you return a Stream.

## The Stream Trait

```rust
// std::iter::Iterator — synchronous, one value at a time
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// futures::Stream — async, one value at a time
trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

`Poll::Ready(Some(item))` — a value is available.
`Poll::Ready(None)` — the stream has ended.
`Poll::Pending` — nothing yet, wake me when there is something.

You almost never implement this trait directly. You use combinators and adapters from `StreamExt`, which `tokio_stream` and `futures` provide.

## Creating Streams

```toml
tokio-stream = "0.1"
futures = "0.3"
async-stream = "0.3"
```

```rust
use tokio_stream::{self, StreamExt};
use futures::stream::{self, Stream};

// From a vec — good for testing
let s = stream::iter(vec!["song1", "song2", "song3"]);

// From a Tokio channel — the most common pattern in servers
let (tx, rx) = tokio::sync::mpsc::channel::<String>(32);
let s = tokio_stream::wrappers::ReceiverStream::new(rx);

// From a timer
let ticks = tokio_stream::wrappers::IntervalStream::new(
    tokio::time::interval(std::time::Duration::from_secs(1))
);

// From async generator with async-stream
use async_stream::stream;

fn heartbeat_stream(room_id: String) -> impl Stream<Item = String> {
    stream! {
        let mut interval = tokio::time::interval(std::time::Duration::from_secs(30));
        loop {
            interval.tick().await;
            yield serde_json::json!({
                "event": "heartbeat",
                "room": room_id,
            }).to_string();
        }
    }
}

// From unfold — build a stream from a stateful async function
let song_stream = stream::unfold(0u32, |page| async move {
    let songs = fetch_songs_page(page).await.ok()?;
    if songs.is_empty() {
        return None; // Stream ends
    }
    // Yield all songs on this page, advance page counter
    Some((stream::iter(songs), page + 1))
    // Note: this produces a stream of streams; use flatten() to get individual songs
});
```

## Consuming a Stream

```rust
use futures::StreamExt;

async fn consume_examples() {
    let mut s = stream::iter(vec![1u32, 2, 3, 4, 5]);

    // Pattern 1: while let — the most explicit
    while let Some(item) = s.next().await {
        println!("{}", item);
    }

    // Pattern 2: for_each — async closure
    stream::iter(vec!["a", "b", "c"])
        .for_each(|s| async move { println!("{}", s) })
        .await;

    // Pattern 3: collect — gather all items into a Vec
    let all: Vec<u32> = stream::iter(1u32..=10).collect().await;

    // Pattern 4: fold — accumulate
    let sum: u32 = stream::iter(1u32..=10)
        .fold(0, |acc, x| async move { acc + x })
        .await;
}
```

## Stream Adapters

Stream adapters transform streams lazily — they produce a new stream, processing one item at a time as the consumer pulls values. They don't buffer everything upfront.

```rust
use tokio_stream::StreamExt;

async fn adapter_examples() {
    // map: transform each item
    let doubled = stream::iter(1u32..=5)
        .map(|x| x * 2)
        .collect::<Vec<_>>()
        .await;
    // [2, 4, 6, 8, 10]

    // filter: keep items matching a predicate
    let evens = stream::iter(1u32..=10)
        .filter(|x| futures::future::ready(x % 2 == 0))
        .collect::<Vec<_>>()
        .await;

    // take: stop after N items
    let first_three = stream::iter(1u32..).take(3).collect::<Vec<_>>().await;
    // [1, 2, 3]

    // chunks: group items into batches
    let batches = stream::iter(1u32..=10)
        .chunks(3)
        .collect::<Vec<_>>()
        .await;
    // [[1,2,3], [4,5,6], [7,8,9], [10]]

    // throttle: rate-limit item delivery
    let throttled = stream::iter(1u32..=100)
        .throttle(std::time::Duration::from_millis(100));
    // Delivers at most 10 items per second
}
```

## buffer_unordered: Parallel Processing with Backpressure 🔴

`buffer_unordered(N)` is the most powerful stream adapter. It takes a stream of futures and runs up to N of them concurrently. Results are yielded as they complete — not in order. This is the right way to make N concurrent network requests without either spawning N tasks or waiting for each one before starting the next.

```rust
use futures::StreamExt;

async fn fetch_song_metadata_parallel(song_ids: Vec<i64>) -> Vec<SongMetadata> {
    let db = get_db_pool();
    
    stream::iter(song_ids)
        .map(|id| {
            let db = db.clone();
            async move {
                sqlx::query_as!(SongMetadata, "SELECT * FROM songs WHERE id = $1", id)
                    .fetch_one(&db)
                    .await
            }
        })
        .buffer_unordered(10)  // At most 10 concurrent DB queries
        .filter_map(|result| async move { result.ok() }) // Skip errors
        .collect()
        .await
}
```

Without `buffer_unordered`: one query at a time, sequential.
With `buffer_unordered(10)`: up to 10 queries concurrent, results come back as they finish.
With `buffer_unordered(song_ids.len())`: all queries concurrent, same as spawning N tasks.

The N parameter is your backpressure knob. Too low and you leave throughput on the table. Too high and you hammer the database. For Postgres, 10–50 is usually a good range.

## Converting a Channel to a Stream

The pattern for turning an `mpsc::Receiver` into a `Stream` — which you need for SSE and for chaining stream adapters on channel output:

```rust
use tokio::sync::mpsc;
use tokio_stream::wrappers::ReceiverStream;

async fn channel_as_stream() {
    let (tx, rx) = mpsc::channel::<String>(32);

    // Wrap the receiver in a stream adapter
    let mut stream = ReceiverStream::new(rx);

    // Spawn a producer
    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(format!("event {}", i)).await.unwrap();
        }
        // tx drops here, stream ends
    });

    // Consume like any stream
    while let Some(event) = stream.next().await {
        println!("Got: {}", event);
    }
}
```

## Server-Sent Events — Streaming to the Browser

Server-Sent Events (SSE) are a simple alternative to WebSockets for one-directional server-to-client streaming. The browser connects with a regular GET request, and the server sends a stream of events. No bidirectional communication, no upgrade, just a long-running HTTP response.

SSE is useful for things like song progress updates, notifications, or live dashboards where the client doesn't need to send data.

Axum supports SSE natively:

```rust
use axum::{
    extract::{Path, State},
    response::{sse::{Event, Sse}, IntoResponse},
};
use futures::stream::{self, Stream};
use tokio_stream::StreamExt;
use std::convert::Infallible;

pub async fn queue_events_sse(
    Path(room_id): Path<String>,
    State(state): State<AppState>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    // Create a channel for this SSE client
    let (tx, rx) = tokio::sync::mpsc::channel::<String>(32);

    // Register this SSE connection to receive room events
    state.register_sse_client(&room_id, tx).await;

    let stream = tokio_stream::wrappers::ReceiverStream::new(rx)
        .map(|json| {
            Ok(Event::default()
                .event("queue_update")
                .data(json))
        });

    Sse::new(stream).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(std::time::Duration::from_secs(30))
            .text("ping"),
    )
}
```

The browser connects with:
```javascript
const events = new EventSource('/queue/room1/events');
events.addEventListener('queue_update', e => {
    const data = JSON.parse(e.data);
    renderQueue(data.queue);
});
```

## Streaming Database Results

Instead of loading all rows into a `Vec` and returning it, stream them to the response. For large result sets this uses constant memory regardless of row count.

```rust
use futures::TryStreamExt;

pub async fn stream_play_history(
    room_id: String,
    db: sqlx::PgPool,
) -> impl Stream<Item = Result<PlayHistoryEntry, sqlx::Error>> + '_ {
    sqlx::query_as!(
        PlayHistoryEntry,
        "SELECT * FROM play_history WHERE room_id = $1 ORDER BY played_at DESC LIMIT 50",
        room_id
    )
    .fetch(&db) // Returns a Stream, not a Vec
}

// Caller:
let mut history = stream_play_history(room_id, db).await;
while let Some(entry) = history.try_next().await? {
    // process one row at a time
    send_to_client(&entry).await;
}
```

`sqlx::query_as!(...).fetch()` returns a `Stream`. It fetches rows from Postgres as the consumer pulls them, using a cursor under the hood. For 50 rows this doesn't matter much, but for 50,000 rows it's the difference between loading 400MB into RAM or using 4KB.

## Merging Streams

Sometimes you need to combine events from two sources and process them in one loop. `tokio_stream::StreamExt::merge` interleaves two streams item by item:

```rust
use tokio_stream::StreamExt;

async fn combined_events(
    queue_events: impl Stream<Item = String>,
    heartbeat: impl Stream<Item = String>,
) {
    let mut merged = queue_events.merge(heartbeat);
    while let Some(event) = merged.next().await {
        send_to_client(event).await;
    }
}
```

Items arrive in the order they become available. If both streams have items ready, they interleave. Neither stream starves the other.

## Common Mistakes

**Forgetting `.await` on terminal operations.** `stream.map(...).filter(...).collect()` returns a `Future`, not a `Vec`. You must `.await` it. The compiler tells you this, but the error message can be confusing.

**Using `buffer_unordered` with too-large N.** `buffer_unordered(1000)` on a set of database queries will open 1,000 concurrent connections. Most connection pools have a max size far below that. Either size `N` to match your pool size, or use a `Semaphore` inside the map closure to limit actual concurrency independently of the stream size.

**Blocking inside a stream adapter.** If you call a blocking function inside `.map()` without `spawn_blocking`, you'll block the executor thread for every item. Heavy computation in a stream adapter should be offloaded to `spawn_blocking` and awaited.

**Not pinning streams for `next()`.** The `Stream::poll_next` signature requires `Pin<&mut Self>`. To call `.next()` in a loop, the stream must be pinned. Usually the compiler catches this: `use tokio_stream::StreamExt` brings in a `next()` method that handles pinning for you. If you get errors about `Unpin`, try `let mut stream = Box::pin(my_stream)` or `tokio::pin!(my_stream)`.
