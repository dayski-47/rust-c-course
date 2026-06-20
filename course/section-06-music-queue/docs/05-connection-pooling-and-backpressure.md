# 05 - Connection Pooling and Backpressure 🟡

Every time you establish a new database or Redis connection, you pay a cost: TCP handshake, TLS negotiation, database authentication, memory allocation on the server. For PostgreSQL, this is typically 50–150ms. If every HTTP request created a new connection, your service would spend more time connecting than actually doing work.

Connection pooling solves this by maintaining a set of reusable connections. When a handler needs a connection, it borrows one from the pool. When the handler finishes, the connection goes back. The pool maintains persistent connections that stay open for the lifetime of the process.

## How deadpool Works

`deadpool` is the async connection pool library that works with both `sqlx` and Redis. It's what `deadpool-redis` wraps.

The mental model is simple: the pool holds N connections. When you call `pool.get().await`, you get a managed guard that wraps one of those connections. When the guard drops, the connection is returned to the pool. If all N connections are checked out, `pool.get().await` blocks until one becomes available.

```toml
deadpool-redis = "0.14"
```

```rust
use deadpool_redis::{Config, Pool, Runtime};

pub fn create_pool(redis_url: &str, max_size: usize) -> Pool {
    let mut cfg = Config::from_url(redis_url);
    cfg.pool = Some(deadpool_redis::PoolConfig {
        max_size,
        ..Default::default()
    });
    cfg.create_pool(Some(Runtime::Tokio1))
        .expect("Failed to create Redis pool")
}

async fn use_pool(pool: &Pool) {
    // Borrow a connection from the pool
    let mut conn = pool.get().await.unwrap();

    // Use it
    use redis::AsyncCommands;
    let _: () = conn.set("key", "value").await.unwrap();

    // conn drops here - automatically returned to pool
}
```

The connection is returned even if your function returns early via `?` or panics. Rust's drop semantics guarantee it. This is the same pattern as `MutexGuard` - the borrow returns when the guard leaves scope.

## Pool Configuration

```rust
use deadpool_redis::{Config, PoolConfig, Runtime};

let mut cfg = Config::from_url("redis://127.0.0.1:6379");
cfg.pool = Some(PoolConfig {
    max_size: 16,           // maximum connections open at once
    timeouts: deadpool::managed::Timeouts {
        wait: Some(std::time::Duration::from_secs(5)),   // max wait for available connection
        create: Some(std::time::Duration::from_secs(2)), // max time to create a connection
        recycle: Some(std::time::Duration::from_secs(1)),// max time to validate recycled conn
    },
    ..Default::default()
});
```

**max_size**: How many connections to keep open. For a service with up to 100 concurrent requests that each make a few Redis calls, `max_size = 16` is plenty. Redis operations are fast; connections don't stay checked out for long.

**wait timeout**: How long to wait when all connections are in use. If a connection doesn't become available in time, `pool.get().await` returns an error. This is better than waiting forever.

**When to increase max_size**: When you see `pool.get()` timing out or taking long under load. A good starting point is `cpu_count * 2` for Redis. Monitor pool queue depth in production.

## What Is Backpressure?

Backpressure is the mechanism by which a slow consumer tells a fast producer to slow down.

Without backpressure, if a producer sends faster than a consumer can process, work accumulates in a buffer. If the buffer is unbounded, memory grows without limit until the process runs out of RAM and the OS kills it. This is one of the most common causes of mysterious production outages - everything looks fine until suddenly the server dies.

```
No backpressure (dangerous):
  
  Producer → [====================] → Consumer
  (fast)       unbounded queue          (slow)
  
  Over time:
  Producer → [==============================================] → Consumer
  (fast)       RAM fills up...                                   (slow)
  
  Eventually: OOM kill
```

With a bounded channel, the producer blocks when the buffer is full. Instead of running out of memory, the system slows down gracefully.

```
With backpressure (safe):
  
  Producer → [====] → Consumer
  (fast)     max=10    (slow)
  
  When full:
  Producer waits ← [====] → Consumer (processing)
  
  Result: memory stays bounded, latency increases predictably
```

## Bounded Channels in Practice

You've been using `mpsc::channel` since Section 4. The difference between bounded and unbounded is one argument:

```rust
use tokio::sync::mpsc;

// Unbounded - dangerous, never use in production for request handling
let (tx, rx) = mpsc::unbounded_channel::<WorkItem>();

// Bounded - safe, provides backpressure
let (tx, mut rx) = mpsc::channel::<WorkItem>(100); // max 100 items buffered

// Sending to a full channel blocks the sender
// This is the backpressure propagating upstream
tokio::spawn(async move {
    for item in items {
        tx.send(item).await.unwrap(); // blocks if buffer is full
    }
});

// Consumer processes at its own pace
tokio::spawn(async move {
    while let Some(item) = rx.recv().await {
        slow_process(item).await; // Takes its time
        // Producer will slow down naturally
    }
});
```

In `queuemaster`, each WebSocket client's send buffer is bounded. If a client's network is slow and their outbound buffer fills up, `try_send` fails. You can drop the message (acceptable for real-time updates - the client will get the next one) or close the connection (appropriate if the client is too far behind to be useful).

```rust
// Broadcasting with backpressure handling
for (client_id, sender) in &room.clients {
    match sender.try_send(event.clone()) {
        Ok(_) => {}
        Err(tokio::sync::mpsc::error::TrySendError::Full(_)) => {
            // Client's buffer is full - they're too slow
            // For music queue: drop this event, they'll get the next state update
            tracing::warn!("Client {} is lagging, dropping event", client_id);
        }
        Err(tokio::sync::mpsc::error::TrySendError::Closed(_)) => {
            // Channel is closed - client disconnected
            // Mark for cleanup (remove from room on next tick)
            disconnected.push(*client_id);
        }
    }
}
```

## Semaphore: Limiting Concurrency

A `Semaphore` lets you limit how many concurrent operations happen at once. It's like a pool of permits - each operation acquires a permit, does its work, and releases the permit.

Use this when you want to limit concurrent database queries, concurrent file operations, or concurrent external API calls.

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

// Allow at most 10 concurrent calls to the external music metadata API
let api_semaphore = Arc::new(Semaphore::new(10));

async fn fetch_song_metadata(song_id: i64, semaphore: Arc<Semaphore>) -> SongMetadata {
    // Acquire a permit - blocks if 10 others are already running
    let _permit = semaphore.acquire().await.unwrap();
    // _permit is dropped at end of scope, releasing the slot

    // Now make the external API call
    external_api::get_song(song_id).await.unwrap()
}

// Called from multiple places concurrently - semaphore keeps it bounded
let semaphore = Arc::new(Semaphore::new(10));
let futures: Vec<_> = song_ids.iter().map(|&id| {
    let sem = semaphore.clone();
    async move { fetch_song_metadata(id, sem).await }
}).collect();

// This starts all futures, but only 10 run at a time
let results = futures::future::join_all(futures).await;
```

## Rate Limiting in Async Code

Tower provides a `RateLimitLayer` for per-service rate limiting. But sometimes you need finer control, like per-user limits in WebSocket handlers.

For per-user rate limiting in WebSocket message handlers, the simplest approach uses an `Instant` and a counter:

```rust
use std::time::{Duration, Instant};

struct RateLimiter {
    max_per_second: u32,
    window_start: Instant,
    count: u32,
}

impl RateLimiter {
    fn new(max_per_second: u32) -> Self {
        RateLimiter {
            max_per_second,
            window_start: Instant::now(),
            count: 0,
        }
    }

    fn allow(&mut self) -> bool {
        let now = Instant::now();
        if now.duration_since(self.window_start) >= Duration::from_secs(1) {
            // New window
            self.window_start = now;
            self.count = 0;
        }
        if self.count < self.max_per_second {
            self.count += 1;
            true
        } else {
            false
        }
    }
}

// In the WebSocket receive loop:
let mut limiter = RateLimiter::new(10); // 10 actions per second

while let Some(Ok(msg)) = receiver.next().await {
    if !limiter.allow() {
        // Send a rate limit error to the client
        let _ = sender.send(Message::Text(
            r#"{"error":"rate_limit","message":"Too many actions"}"#.into()
        )).await;
        continue;
    }
    // Process the message
}
```

For Redis-backed rate limiting (shared across multiple server instances), use the INCR pattern from doc 01.

## Monitoring Pool Health

In production, you want to know when your pool is under pressure.

```rust
// Log pool metrics periodically
tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(30));
    loop {
        interval.tick().await;
        let status = redis_pool.status();
        tracing::info!(
            available = status.available,
            size = status.size,
            waiting = status.waiting,
            "Redis pool metrics"
        );
        if status.waiting > 0 {
            tracing::warn!(
                waiting = status.waiting,
                "Redis pool pressure - consider increasing max_size"
            );
        }
    }
});
```

`status.waiting` is the key metric. If it's consistently above zero, your pool is a bottleneck. Either increase `max_size` or reduce how long your handlers hold connections.

## What Happens Without Backpressure

Here is a realistic failure scenario. You have a WebSocket broadcast that sends queue updates to 500 clients. Each send is `O(1)` in time but creates a future. If those futures pile up faster than they resolve:

1. Tokio's task queue grows without bound
2. Memory usage climbs steadily
3. GC (well, allocator) pressure causes latency spikes
4. Eventually the process runs out of memory
5. OOM kill - all 500 clients disconnected simultaneously

The fix is to use bounded channels for client send buffers and `try_send` instead of `send`. Slow clients get dropped, but fast clients keep working. The service stays up.

## Common Mistakes

**Not setting a wait timeout on the pool.** If all pool connections are held and you don't set a timeout, `pool.get()` waits forever. One slow database query that holds a connection open can cause an entire service to hang. Always set a `wait` timeout.

**Using `try_send` everywhere because `send` might block.** `try_send` silently drops messages if the buffer is full. For critical messages (like a disconnect notification), use `send().await` so the sender waits rather than losing data. For real-time streaming updates where losing one frame is acceptable, `try_send` is appropriate.

**Creating connections outside the pool for "one-off" operations.** Every time you call `redis::Client::get_async_connection()` directly, you create and tear down a TCP connection. Under load this is hundreds of milliseconds per request. Always go through the pool.

**Pool size too large.** Redis has a default max of 10,000 clients, but each connection consumes resources on the Redis server. A pool of 1,000 connections per service instance will put Redis under significant pressure. Size your pool based on actual concurrency, not as a safety net.
