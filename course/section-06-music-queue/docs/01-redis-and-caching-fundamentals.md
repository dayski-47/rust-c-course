# 01 - Redis and Caching Fundamentals 🟡

## Engineering Methodology: Event-Driven Architecture + State Machine Design

The music queue is a state machine. At any moment, it's in one of these states:
- Empty queue, nothing playing
- Queue has items, playing the first one
- Queue has items, paused
- Queue is being modified (song added, removed, reordered)

Before coding, draw the state machine:
- What states exist?
- What events trigger transitions? (user adds song → Empty becomes Non-empty; song finishes → advance to next; skip → remove current, play next)
- What is broadcast when state changes? (every connected client needs to know about every state change)

Event-driven architecture means the system is driven by events (user adds song, song finishes, user joins room) rather than polling (checking every second if anything changed). Redis pub/sub and WebSocket messages are the implementation of event-driven thinking.

When you design this way, the code becomes: receive event → validate → update state → publish event to subscribers. The loop is clear. The failure modes are clear (what if state update succeeds but publish fails?).

---

Redis is an in-memory data store. It holds everything in RAM, which makes it roughly 100x faster than hitting a PostgreSQL database for a simple key lookup. You use it when you have data that is read frequently but changes infrequently, or when you need coordination between multiple processes that can't share memory directly.

In `queuemaster` you will use Redis for three things: persisting the music queue as a Redis List, broadcasting queue changes with pub/sub, and rate limiting WebSocket actions with a counter + expiry pattern.

## Running Redis Locally

```bash
docker run -d -p 6379:6379 --name redis redis:alpine
```

That's it. Redis starts on port 6379 with no authentication. For production you'd set `requirepass` in the config, but for development this works fine.

## Connecting with deadpool-redis

`deadpool-redis` gives you a connection pool. Creating a Redis connection is cheap compared to Postgres, but you still want pooling because you may have hundreds of WebSocket tasks all touching Redis simultaneously.

```toml
# Cargo.toml
deadpool-redis = "0.14"
redis = { version = "0.25", features = ["tokio-comp"] }
```

```rust
use deadpool_redis::{Config, Pool, Runtime};

pub fn create_redis_pool(url: &str) -> Pool {
    let cfg = Config::from_url(url);
    cfg.create_pool(Some(Runtime::Tokio1))
        .expect("failed to create Redis pool")
}

// In main:
let redis_pool = create_redis_pool("redis://127.0.0.1:6379");

// Getting a connection:
let mut conn = redis_pool.get().await?;
// conn is returned to the pool when it drops at end of scope
```

The pool manages a set of persistent connections. When your handler calls `pool.get().await`, it either borrows an idle connection or waits for one to become available. When the guard drops, the connection goes back. You never open and close connections per request.

## Basic Operations

```rust
use redis::AsyncCommands;

async fn basic_ops(pool: &Pool) -> redis::RedisResult<()> {
    let mut conn = pool.get().await.unwrap();

    // SET a string value
    conn.set("greeting", "hello world").await?;

    // GET it back (returns Option<String>)
    let val: Option<String> = conn.get("greeting").await?;
    println!("{:?}", val); // Some("hello world")

    // SET with expiry: key disappears after 60 seconds
    conn.set_ex("session:abc123", "user_data", 60).await?;

    // Check if a key exists
    let exists: bool = conn.exists("greeting").await?;

    // Delete a key
    conn.del("greeting").await?;

    // Increment a counter (creates it at 0 if missing)
    let count: i64 = conn.incr("page_views", 1).await?;

    Ok(())
}
```

The type annotation on `get` is required. Redis returns raw bytes; the `redis` crate will try to deserialize them into whatever type you specify. If the key doesn't exist you get `None`, not an error.

## Data Structures

Redis is not just a key-value store for strings. It has several specialized structures that you will use in this project.

**Lists** - ordered sequences with O(1) push/pop at either end. The music queue is a List.

```rust
// RPUSH appends to the right (tail). LPUSH prepends to the left (head).
conn.rpush("queue:room1", 42u64).await?;  // add song id 42
conn.rpush("queue:room1", 99u64).await?;  // add song id 99

// LRANGE fetches a slice. 0, -1 means "everything".
let queue: Vec<u64> = conn.lrange("queue:room1", 0, -1).await?;
// [42, 99]

// LPOP removes and returns from the left (dequeue from front)
let next: Option<u64> = conn.lpop("queue:room1", None).await?;
// Some(42)

// LLEN returns the current length
let length: usize = conn.llen("queue:room1").await?;
```

**Hashes** - a map stored under one key. Good for structured objects like "current song."

```rust
// HSET stores field-value pairs
conn.hset_multiple("now_playing:room1", &[
    ("song_id", "42"),
    ("title", "Bohemian Rhapsody"),
    ("artist", "Queen"),
    ("started_at", "1718800000"),
]).await?;

// HGET fetches one field
let title: Option<String> = conn.hget("now_playing:room1", "title").await?;

// HGETALL fetches all fields as pairs
let song: std::collections::HashMap<String, String> =
    conn.hgetall("now_playing:room1").await?;
```

**Sorted Sets** - like a set, but each member has a numeric score. Perfect for play history sorted by timestamp.

```rust
// ZADD with score (unix timestamp) and member (song id as string)
conn.zadd("history:room1", "song:42", 1718800000f64).await?;
conn.zadd("history:room1", "song:99", 1718800060f64).await?;

// ZREVRANGE with scores, newest first, limited to 50
let history: Vec<(String, f64)> =
    conn.zrevrange_withscores("history:room1", 0, 49).await?;
```

## The Cache-Aside Pattern

This is the most common caching pattern. Check Redis first. If the data is there (cache hit), return it. If not (cache miss), hit the database, store the result in Redis with a TTL, and return it.

```rust
use deadpool_redis::Pool as RedisPool;
use sqlx::PgPool;

async fn get_song(
    song_id: i64,
    redis: &RedisPool,
    db: &PgPool,
) -> anyhow::Result<Song> {
    let cache_key = format!("song:{}", song_id);
    let mut conn = redis.get().await?;

    // Try cache first
    let cached: Option<String> = conn.get(&cache_key).await?;
    if let Some(json) = cached {
        return Ok(serde_json::from_str(&json)?);
    }

    // Cache miss: query the database
    let song = sqlx::query_as!(Song, "SELECT * FROM songs WHERE id = $1", song_id)
        .fetch_one(db)
        .await?;

    // Store in cache with 5-minute TTL
    let json = serde_json::to_string(&song)?;
    conn.set_ex(&cache_key, json, 300).await?;

    Ok(song)
}
```

The TTL is important. Without it, stale data lives in Redis forever. The right TTL depends on how often the data changes. Song metadata rarely changes, so 5 minutes is reasonable. User session data might change frequently, so use a shorter TTL or invalidate it explicitly on write.

## Pub/Sub

Redis pub/sub is how `queuemaster` broadcasts queue changes to all server instances. When a client adds a song, the server publishes a message to a channel. Every server instance (and every tab of `tokio-console`) that is subscribed to that channel receives the message.

Publishing is simple. Subscribing requires a dedicated connection that you put into a loop - you cannot use a pooled connection for subscriptions because the connection changes mode.

```rust
use redis::aio::PubSub;
use redis::Client;

// Publishing (from any pooled connection)
async fn publish_queue_event(
    conn: &mut deadpool_redis::Connection,
    room_id: &str,
    event_json: &str,
) -> redis::RedisResult<()> {
    let channel = format!("queue_events:{}", room_id);
    conn.publish(channel, event_json).await
}

// Subscribing (dedicated connection, not from the pool)
async fn subscribe_to_room(client: &Client, room_id: &str) {
    let mut pubsub: PubSub = client.get_async_pubsub().await.unwrap();
    let channel = format!("queue_events:{}", room_id);
    pubsub.subscribe(&channel).await.unwrap();

    let mut stream = pubsub.into_on_message();
    while let Some(msg) = stream.next().await {
        let payload: String = msg.get_payload().unwrap();
        println!("Received: {}", payload);
        // Forward to WebSocket clients in this room
    }
}
```

## Rate Limiting with INCR + EXPIRE

This pattern uses an incrementing counter that expires automatically. The first request in a new window creates the key with an expiry. Subsequent requests increment it. If the count exceeds the limit, reject the request.

```rust
async fn check_rate_limit(
    conn: &mut deadpool_redis::Connection,
    user_id: i64,
    limit: i64,
) -> redis::RedisResult<bool> {
    let now = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_secs();
    let window = now / 60; // one-minute windows
    let key = format!("ratelimit:{}:{}", user_id, window);

    // Atomic increment
    let count: i64 = conn.incr(&key, 1).await?;

    // Set expiry on first request in this window
    if count == 1 {
        conn.expire(&key, 60).await?;
    }

    Ok(count <= limit)
}
```

## Cache Invalidation

When you update data in the database, you must also remove or update the cached copy. The simplest approach is to delete the cache key. The next request will repopulate it.

```rust
async fn update_song_metadata(
    song_id: i64,
    new_title: &str,
    db: &PgPool,
    redis: &RedisPool,
) -> anyhow::Result<()> {
    // Update the database
    sqlx::query!("UPDATE songs SET title = $1 WHERE id = $2", new_title, song_id)
        .execute(db)
        .await?;

    // Invalidate the cache
    let mut conn = redis.get().await?;
    conn.del(format!("song:{}", song_id)).await?;

    Ok(())
}
```

For compound data - like a user's profile that shows up under multiple keys - you need to be systematic about what to invalidate. One approach is to store all related keys in a Redis Set and delete them all at once. Another is to use a version number in the key and increment it on writes, making old keys instantly stale.

## How It Breaks

- Cache invalidation: you cache data in Redis, then the source data changes in the database, but Redis still has the old value. Your reads return stale data. Solution: always set a TTL, and explicitly delete cache keys when the source data changes.
- Redis as single point of failure: if Redis goes down, your whole queue service goes down if you haven't coded a fallback.
- Redis connection pool exhaustion: same as database pool exhaustion. Set timeouts and max pool size.
- Pub/sub and dedicated connections: Redis pub/sub requires a connection to be in SUBSCRIBE mode - it can't be used for other commands. Mixing pub/sub with regular commands on the same connection fails.

## Common Mistakes

**Using a pooled connection for pub/sub.** A connection that has subscribed to channels enters a different protocol mode and cannot be used for regular commands. Keep a separate `Client` instance for creating subscription connections.

**No TTL on cached data.** Without expiry, deleted database rows and renamed songs will show stale data forever. Always set a TTL. When in doubt, shorter is safer.

**Serializing large objects.** Redis excels at small, frequently-accessed data. Caching a paginated list of 10,000 songs as one JSON blob is counterproductive - you'll evict more useful data from Redis memory. Cache individual entities, not collections.

**Forgetting that INCR is atomic but INCR + EXPIRE is not.** Between those two calls, another process could set the key and leave a counter without an expiry. The pattern above handles this by only calling EXPIRE when `count == 1`, which means only the first increment in a window triggers the expiry - and if two processes race on the very first increment, both will call EXPIRE with the same value, which is harmless.
