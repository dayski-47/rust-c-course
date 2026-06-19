# Project: queuemaster — Real-Time Music Queue Service

## What You're Building

`queuemaster` is a microservice that manages shared music play queues. Multiple users connect via WebSocket to a shared "room." When any user adds a song, skips the current track, or reorders the queue, every other connected client sees the change in real time — no polling, no page refresh.

This plugs into `melody-api` from Section 5. Where melody-api owns song metadata and user accounts, queuemaster owns the concept of "what's playing right now in this room." They communicate over HTTP: queuemaster calls melody-api to validate song IDs and fetch song details.

**The core user experience:**
1. User logs in → gets a JWT
2. User opens the music player and joins a room → WebSocket connection established
3. User adds a song → the room's queue updates for all listeners instantly
4. The DJ skips the current song → everyone's "now playing" updates instantly
5. Another user joins mid-session → they receive the current queue state immediately

This is the architecture used by every collaborative listening app (Discord Stage, Spotify Group Session, etc.), game lobby, or collaborative document editor. The patterns you learn here apply directly.

## Architecture

```
                      ┌─────────────────────────────────────────────┐
                      │              queuemaster process             │
                      │                                             │
  Client A            │  ┌──────────────────────────────────────┐  │
  (WebSocket) ────────┼─►│         Connection Manager            │  │
                      │  │  (tracks all WS connections)         │  │
  Client B            │  │                                      │  │
  (WebSocket) ────────┼─►│  client_a_tx ─┐                     │  │
                      │  │  client_b_tx ─┤→ Room Actor (room1) │  │
  Client C            │  │  client_c_tx ─┘  (owns queue state) │  │
  (WebSocket) ────────┼─►│                       │              │  │
                      │  └──────────────────────────────────────┘  │
                      │                           │                  │
                      │              ┌────────────▼──────────────┐  │
                      │              │     Redis Operations       │  │
                      │              │  - LPUSH/LRANGE on queue  │  │
                      │              │  - HSET on now_playing     │  │
                      │              │  - ZADD on history         │  │
                      │              │  - PUBLISH queue_events    │  │
                      │              └────────────┬──────────────┘  │
                      │                           │                  │
  HTTP Clients        │  ┌──────────────────────────────────────┐  │
  (GET /queue/:id) ───┼─►│        REST API Handlers             │  │
  (POST /auth/login)  │  │  (read from Redis, write to Redis)   │  │
                      │  └──────────────────────────────────────┘  │
                      └─────────────────────────────────────────────┘
                                           │
                                           │  SUBSCRIBE queue_events:*
                                           ▼
                               ┌───────────────────────┐
                               │         Redis         │
                               │                       │
                               │  queue:room1 (List)   │
                               │  queue:room2 (List)   │
                               │  now_playing:room1    │
                               │  history:room1 (ZSet) │
                               │  ratelimit:user:min   │
                               │  refresh_token:user   │
                               └───────────────────────┘
                                           │
                                           │  melody-api integration
                                           ▼
                               ┌───────────────────────┐
                               │      melody-api        │
                               │  (Section 5 project)  │
                               │  - GET /songs/:id     │
                               │  - GET /users/:id     │
                               └───────────────────────┘
```

**How it flows:**

When Client A sends `{"action":"add","song_id":42}` over WebSocket:

1. The recv task for Client A reads the message from the socket
2. It sends an `AddSong` command to the Room Actor via an mpsc channel
3. The Room Actor calls melody-api to validate song 42 exists
4. The Room Actor pushes song 42 to the Redis List `queue:room1`
5. The Room Actor serializes a `queue_updated` event and PUBLISH it to `queue_events:room1`
6. The Room Actor also sends the event directly to all connected clients' send tasks
7. Each send task writes the event to its WebSocket connection
8. Clients B and C see the updated queue within milliseconds

The Redis PUBLISH step (step 6 alternative) enables horizontal scaling. If you run two instances of queuemaster, both subscribe to the same Redis pub/sub channels. A message published by one instance is received by the other, which then forwards it to the clients connected to that instance. Redis becomes the message bus between server processes.

## API Reference

### Authentication

**POST /auth/register**
```json
// Request body
{
  "username": "alice",
  "password": "hunter2"
}

// Response 201
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9...",
  "user_id": 42
}
```

**POST /auth/login**
```json
// Request body
{
  "username": "alice",
  "password": "hunter2"
}

// Response 200
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiJ9..."
}
```

**POST /auth/refresh**
```json
// Request body
{
  "refresh_token": "eyJhbGciOiJIUzI1NiJ9..."
}

// Response 200
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9..."
}
```

### REST Endpoints

**GET /queue/:room_id**
- Requires: `Authorization: Bearer <token>` header
- Returns the current queue state

```json
{
  "room_id": "room1",
  "now_playing": {
    "song_id": 42,
    "title": "Bohemian Rhapsody",
    "artist": "Queen",
    "duration_seconds": 354,
    "started_at": 1718800000
  },
  "queue": [
    { "song_id": 99, "title": "Stairway to Heaven", "artist": "Led Zeppelin" },
    { "song_id": 7, "title": "Hotel California", "artist": "Eagles" }
  ],
  "listener_count": 3
}
```

**GET /queue/:room_id/history**
- Returns the last 50 songs played in this room, newest first

```json
{
  "room_id": "room1",
  "history": [
    {
      "song_id": 42,
      "title": "Bohemian Rhapsody",
      "played_at": 1718800000,
      "added_by": "alice"
    }
  ]
}
```

### WebSocket Connection

**ws://localhost:8081/queue/:room_id**

Authentication: pass the JWT as a query parameter on the initial connection:
```
ws://localhost:8081/queue/room1?token=eyJhbGciOiJIUzI1NiJ9...
```

Or send it as the first message after connecting (if query params are inconvenient):
```json
{ "action": "auth", "token": "eyJhbGciOiJIUzI1NiJ9..." }
```

## WebSocket Message Protocol

All messages are JSON text frames.

### Client → Server Messages

**Add a song to the end of the queue:**
```json
{
  "action": "add",
  "song_id": 42
}
```

**Skip the currently playing song:**
```json
{
  "action": "skip"
}
```

**Reorder the queue:** (move item at index `from` to index `to`)
```json
{
  "action": "reorder",
  "from": 2,
  "to": 0
}
```

**Remove a song from the queue:**
```json
{
  "action": "remove",
  "position": 1
}
```

**Send a chat message (stretch goal):**
```json
{
  "action": "chat",
  "message": "great song choice!"
}
```

### Server → Client Messages

**Queue state changed** (sent to all room members after any add/remove/reorder):
```json
{
  "event": "queue_updated",
  "queue": [
    { "song_id": 99, "title": "Stairway to Heaven", "artist": "Led Zeppelin", "duration_seconds": 482 },
    { "song_id": 7, "title": "Hotel California", "artist": "Eagles", "duration_seconds": 391 }
  ],
  "changed_by": "alice",
  "timestamp": 1718800042
}
```

**Now playing changed** (sent when a song starts or skip occurs):
```json
{
  "event": "now_playing",
  "song": {
    "song_id": 42,
    "title": "Bohemian Rhapsody",
    "artist": "Queen",
    "duration_seconds": 354,
    "started_at": 1718800000
  },
  "skipped_by": null
}
```

**User joined the room:**
```json
{
  "event": "user_joined",
  "username": "bob",
  "listener_count": 4,
  "timestamp": 1718800100
}
```

**User left the room:**
```json
{
  "event": "user_left",
  "username": "bob",
  "listener_count": 3,
  "timestamp": 1718800200
}
```

**Initial state** (sent to a client immediately upon connecting):
```json
{
  "event": "room_state",
  "room_id": "room1",
  "now_playing": { "..." : "..." },
  "queue": [ "..." ],
  "listeners": ["alice", "charlie"],
  "listener_count": 2
}
```

**Error response** (invalid action, rate limit, permission denied):
```json
{
  "event": "error",
  "code": "rate_limited",
  "message": "Too many actions. Slow down.",
  "retry_after_ms": 1000
}
```

**Heartbeat** (sent every 30 seconds to detect dead connections):
```json
{
  "event": "heartbeat",
  "timestamp": 1718800300
}
```

## Data in Redis

All queue state lives in Redis. The service is stateless — kill it and restart it, and clients reconnect to the same queue.

| Key Pattern | Type | Contents |
|---|---|---|
| `queue:{room_id}` | List | Song IDs in queue order (LPUSH to front, RPUSH to end) |
| `now_playing:{room_id}` | Hash | Fields: song_id, title, artist, duration_seconds, started_at |
| `history:{room_id}` | Sorted Set | Member: `song:{id}`, Score: Unix timestamp played |
| `queue_events:{room_id}` | Channel | Pub/Sub channel for real-time events |
| `ratelimit:{user_id}:{minute}` | String | Increment counter, expires at minute boundary |
| `refresh_token:{user_id}` | String | Refresh token, expires after 7 days |
| `room_listeners:{room_id}` | Set | Set of currently connected usernames |

Concrete examples:
```
queue:room1          → ["42", "99", "7"]   (list, head=next to play)
now_playing:room1    → {song_id: "42", title: "Bohemian Rhapsody", started_at: "1718800000"}
history:room1        → {("song:42", 1718800000.0), ("song:13", 1718799600.0)}
ratelimit:42:28646   → "7"   (user 42 has made 7 requests in minute window 28646)
refresh_token:42     → "eyJhbGciOiJIUzI1NiJ9..."
```

## Rate Limits

| Context | Limit | Implementation |
|---|---|---|
| REST endpoints | 30 requests per minute per user | Redis INCR + EXPIRE |
| WebSocket actions | 10 actions per second per connection | In-memory token bucket |
| Auth endpoints | 5 attempts per minute per IP | Redis INCR + EXPIRE on IP |

## Project Structure

```
queuemaster/
├── Cargo.toml
├── .env
├── docker-compose.yml
├── migrations/
│   └── 001_initial.sql          # users table
├── src/
│   ├── main.rs                  # startup, routing, graceful shutdown
│   ├── state.rs                 # AppState (pools, config)
│   ├── error.rs                 # AppError type with thiserror
│   ├── auth/
│   │   ├── mod.rs
│   │   ├── handlers.rs          # register, login, refresh
│   │   ├── jwt.rs               # create_token, verify_token
│   │   └── middleware.rs        # AuthUser extractor
│   ├── queue/
│   │   ├── mod.rs
│   │   ├── actor.rs             # RoomActor, RoomHandle, RoomCommand
│   │   ├── handlers.rs          # GET /queue/:id, GET /queue/:id/history
│   │   ├── ws_handler.rs        # WebSocket upgrade handler
│   │   └── models.rs            # Song, QueueEvent, NowPlaying structs
│   ├── redis/
│   │   ├── mod.rs
│   │   ├── pool.rs              # create_redis_pool
│   │   ├── queue_ops.rs         # LPUSH, LRANGE, LREM on queue keys
│   │   └── pubsub.rs            # subscribe_to_room, publish_event
│   └── rate_limit.rs            # RateLimiter struct, check_redis_rate_limit
```

## Cargo.toml

```toml
[package]
name = "queuemaster"
version = "0.1.0"
edition = "2021"

[dependencies]
# Web framework
axum = { version = "0.7", features = ["ws"] }
tower-http = { version = "0.5", features = ["cors", "trace"] }

# Async runtime
tokio = { version = "1", features = ["full"] }
tokio-util = "0.7"
tokio-stream = "0.1"
futures = "0.3"

# Redis
deadpool-redis = "0.14"
redis = { version = "0.25", features = ["tokio-comp"] }

# Database (for user accounts — reuse melody-api's DB or run a second one)
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }

# Auth
jsonwebtoken = "9"
argon2 = "0.5"

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Error handling
thiserror = "1"
anyhow = "1"

# Observability
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Utilities
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }

[dev-dependencies]
tokio = { version = "1", features = ["full", "test-util"] }
```

## Critical Patterns for This Section

### 1. Redis Connection Pool (deadpool-redis)

```toml
[dependencies]
deadpool-redis = "0.15"
redis = "0.26"
```

```rust
use deadpool_redis::{Config, Pool, Runtime};

pub async fn create_redis_pool(url: &str) -> anyhow::Result<Pool> {
    let cfg = Config::from_url(url);
    let pool = cfg.create_pool(Some(Runtime::Tokio1))?;
    Ok(pool)
}

// In a handler, get a connection from the pool:
let mut conn = state.redis.get().await?;
let _: () = redis::cmd("SET")
    .arg("key")
    .arg("value")
    .query_async(&mut conn)
    .await?;
```

### 2. WebSocket Upgrade in Axum

```rust
use axum::extract::ws::{WebSocket, WebSocketUpgrade, Message};

pub async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
    Path(room_id): Path<String>,
) -> Response {
    ws.on_upgrade(move |socket| handle_socket(socket, state, room_id))
}

async fn handle_socket(mut socket: WebSocket, state: AppState, room_id: String) {
    while let Some(msg) = socket.recv().await {
        match msg {
            Ok(Message::Text(text)) => {
                // deserialize the message, update room state, broadcast to others
                // ...
            }
            Ok(Message::Close(_)) | Err(_) => break,
            _ => {}
        }
    }
}
```

### 3. JWT — Create and Validate

```toml
jsonwebtoken = "9"
```

```rust
use jsonwebtoken::{encode, decode, Header, Algorithm, Validation, EncodingKey, DecodingKey};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,   // subject (user id)
    pub exp: usize,    // expiry (unix timestamp)
}

// Create a token:
let claims = Claims {
    sub: user_id.to_string(),
    exp: (chrono::Utc::now() + chrono::Duration::hours(24)).timestamp() as usize,
};
let token = encode(&Header::default(), &claims, &EncodingKey::from_secret(secret.as_bytes()))?;

// Validate a token:
let token_data = decode::<Claims>(
    &token_str,
    &DecodingKey::from_secret(secret.as_bytes()),
    &Validation::new(Algorithm::HS256),
)?;
let user_id = token_data.claims.sub;
```

### 4. tokio::select! for Concurrent Event Handling

The WebSocket handler needs to do two things simultaneously: receive messages from the client AND receive broadcast messages to send to the client.

```rust
use tokio::sync::broadcast;

async fn handle_socket(
    mut socket: WebSocket,
    mut broadcast_rx: broadcast::Receiver<String>,
) {
    loop {
        tokio::select! {
            // Branch 1: message from client
            msg = socket.recv() => {
                match msg {
                    Some(Ok(Message::Text(text))) => {
                        // deserialize → handle command → broadcast update
                        // ...
                    }
                    _ => break,  // disconnect or error
                }
            }
            // Branch 2: message to broadcast to this client
            broadcast_msg = broadcast_rx.recv() => {
                match broadcast_msg {
                    Ok(msg) => {
                        socket.send(Message::Text(msg)).await.ok();
                    }
                    Err(broadcast::error::RecvError::Lagged(n)) => {
                        // we missed n messages — subscriber was too slow
                        eprintln!("subscriber lagged by {n} messages");
                    }
                    Err(_) => break,
                }
            }
        }
    }
}
```

## docker-compose.yml

```yaml
version: "3.9"

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  melody-api:
    build:
      context: ../section-05-melody-api
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/melody
      RUST_LOG: info
    depends_on:
      db:
        condition: service_healthy

  queuemaster:
    build: .
    ports:
      - "8081:8081"
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/melody
      REDIS_URL: redis://redis:6379
      MELODY_API_URL: http://melody-api:8080
      JWT_SECRET: change-me-in-production-use-32-random-bytes
      RUST_LOG: info
    depends_on:
      redis:
        condition: service_healthy
      melody-api:
        condition: service_started

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: melody
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  redis_data:
  pg_data:
```

## .env (for local development without Docker)

```
DATABASE_URL=postgres://postgres:postgres@localhost:5432/melody
REDIS_URL=redis://127.0.0.1:6379
MELODY_API_URL=http://localhost:8080
JWT_SECRET=local-dev-secret-32-bytes-long!!
RUST_LOG=queuemaster=debug,tower_http=debug
HOST=0.0.0.0
PORT=8081
```

## Engineering Approach: State Machine + Event-Driven Design

Map the queue state machine before writing code:

**States:**
- `IDLE`: no songs queued
- `PLAYING(song_id)`: currently playing, queue may have more songs
- `PAUSED(song_id)`: paused, same song still "current"

**Events and transitions:**
- `ADD_SONG` in `IDLE` → `PLAYING(first_song)`
- `ADD_SONG` in `PLAYING` → `PLAYING(same_song)`, queue grows
- `SKIP` in `PLAYING` → `PLAYING(next_song)` or `IDLE` if queue empty
- `PAUSE` in `PLAYING` → `PAUSED(current_song)`
- `RESUME` in `PAUSED` → `PLAYING(current_song)`
- `REMOVE_CURRENT` in `PLAYING` → `PLAYING(next_song)` or `IDLE`

For each transition, what gets published to clients? What gets updated in Redis?

This map is your implementation guide. Each transition becomes a function. Each published event becomes a WebSocket message type.

## What You Bring From Sections 1-5

- **S3 chat server architecture → WebSocket client management**: the two-task-per-client pattern (one reads, one writes) from the chat server is exactly what you use for WebSocket clients. The channel-based broadcast is the same design.
- **S4 Tokio tasks and channels**: `tokio::spawn`, `mpsc`, `broadcast` — all used here. The difference is the message types are now structured JSON events instead of strings.
- **S4 `select!`**: the main event loop uses `tokio::select!` to handle incoming WebSocket messages AND outgoing broadcast events simultaneously — a pattern from S4.
- **S5 JWT**: same JWT auth from S5's middleware, now used to authenticate WebSocket connections on upgrade.
- **S5 Redis basics**: you learned GET/SET/EXPIRE in S5. Now you use Redis Lists for the queue and Redis pub/sub for broadcasting. Same connection pool.
- **S5 AppError + IntoResponse**: HTTP errors for the REST endpoints use the same pattern as S5.

## How This Project Works in Rust — The Full Picture

The queue service runs two layers simultaneously:

**HTTP layer (Axum):** handles REST requests for queue state (`GET /queue/:room_id`) and authentication (`POST /auth/login`). Standard async handlers, same as S5.

**WebSocket layer (also Axum):** when a client connects to `ws://host/queue/:room_id`, Axum upgrades the HTTP connection to WebSocket. Your handler splits the socket into a read half and write half. You spawn two tasks: one reads messages from the client and sends them to the room coordinator. One receives from the room coordinator and writes to the client's socket.

The room coordinator is a Tokio task that owns the queue state for one room. It receives events via an `mpsc` channel, updates the state in Redis (so it persists), publishes the change to a Redis pub/sub channel (so other server instances also get notified), and then fans out the update to all connected clients' write channels.

Redis pub/sub is what makes this horizontally scalable: if you run two instances of this service, both subscribe to the same Redis channel. When one instance processes a queue change, it publishes to Redis, and both instances receive it and update their connected clients. The queue stays consistent across instances.

## Build Milestones

Work through these in order. Each milestone gives you something that runs.

**Milestone 1: Basic Auth**
- Set up Axum, SQLx, database migrations
- Implement POST /auth/register with argon2 password hashing
- Implement POST /auth/login returning a JWT
- Implement the `AuthUser` extractor
- Test: register a user, login, use the token on a protected route

Expected:
```bash
# Get a token:
$ curl -X POST http://localhost:8081/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"secret"}'

{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...","refresh_token":"eyJ..."}

# Use the token on a protected route:
$ curl -H "Authorization: Bearer eyJ0..." \
  http://localhost:8081/queue/room1/state

{"queue":[],"now_playing":null}
```

If you get 401, the token is invalid or expired. If you get 403, the user doesn't have permission. If you get 500 on login, check that your `JWT_SECRET` env var is set.

**Milestone 2: WebSocket Echo Server**
- Add WebSocket upgrade handler to a `/ws/echo` route
- Split the socket, echo every text message back with a prefix
- Test with `websocat ws://localhost:8081/ws/echo`
- You'll type a message and see "echo: [message]" come back

Expected:
```bash
# Install websocat if needed: cargo install websocat
$ websocat ws://localhost:8081/ws/echo
hello
echo: hello
world
echo: world
```

If the upgrade fails immediately, check that you added the `ws` feature to axum in Cargo.toml: `axum = { version = "0.7", features = ["ws"] }`.

If curl is all you have for a quick check:
```bash
curl -i -N -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  http://localhost:8081/ws/echo
```
If the upgrade succeeds, curl doesn't close immediately (it hangs waiting for frames). If it returns 400 or 500, the upgrade route has an issue.

**Milestone 3: Shared Room State in Memory**
- Create `RoomMap` as `Arc<Mutex<HashMap<String, Room>>>`
- When a client connects to `/queue/:room_id`, add them to the room
- When a client sends `{"action":"add","song_id":42}`, broadcast a `queue_updated` event to all room members
- On disconnect, remove from the room
- Test: open two browser tabs, add a song in one, see the update in both

**Milestone 4: Redis Queue Storage**
- Add `deadpool-redis` and create the connection pool
- When a song is added, push its ID to `queue:{room_id}` in Redis
- When serving GET /queue/:room_id, read from Redis instead of in-memory state
- Queue now persists across server restarts
- Test: add songs, restart the server, GET /queue/room1 still shows them

Expected — server starts with Redis connected:
```
$ docker run -d -p 6379:6379 redis:7
$ cargo run
queuemaster starting on 0.0.0.0:8081
Redis: connected to redis://127.0.0.1:6379
```

Test Redis is working before running the server: `redis-cli ping` should return `PONG`.

**Milestone 5: Redis Pub/Sub Broadcasting**
- Keep a separate Redis `Client` (not the pool) for pub/sub subscriptions
- On startup, create a subscriber task that subscribes to `queue_events:*` (pattern subscribe)
- When a client action modifies the queue, PUBLISH to `queue_events:{room_id}`
- The subscriber task receives this and forwards to room clients
- This step enables horizontal scaling: run two instances, they stay in sync via Redis
- Test: run two instances on different ports, connect to both, add a song through one, see the update in the other

**Milestone 6: Full Message Handlers**
- Implement all WebSocket actions: add, skip, reorder, remove
- Send `room_state` event to newly connected clients
- Send `user_joined`/`user_left` events on connect/disconnect
- Track `now_playing` in Redis
- Send `now_playing` event when skip occurs
- Update `history:{room_id}` sorted set when a song finishes

**Milestone 7: Rate Limiting**
- REST rate limiting: check `ratelimit:{user_id}:{minute}` in Redis before processing
- WebSocket rate limiting: in-memory `RateLimiter` struct per connection, 10 actions/second
- Send `{"event":"error","code":"rate_limited"}` when exceeded
- Test: write a script that sends 100 actions quickly, confirm 429 responses

**Milestone 8: Graceful Shutdown**
- Add `CancellationToken` that gets cancelled on Ctrl+C
- Pass the token to all long-running tasks
- Each task: finish current operation, then exit when token is cancelled
- Main function: wait up to 30 seconds for all tasks to finish, then force exit
- Test: start the server, connect a WebSocket client, send Ctrl+C, verify the client receives a close frame

## Implementation Hints

**WebSocket handler structure:**

```rust
pub async fn ws_handler(
    ws: WebSocketUpgrade,
    Path(room_id): Path<String>,
    State(state): State<AppState>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| async move {
        handle_socket(socket, room_id, state).await;
    })
}
```

**Always split the socket early** — a single `WebSocket` handle requires exclusive mutable access, so only one task could use it at a time. Splitting gives you independent `sender` and `receiver` halves that can move into separate tasks concurrently:

```rust
let (mut sender, mut receiver) = socket.split();
```

**Two tasks per client, joined with select!:**

```rust
let recv_handle = tokio::spawn(async move {
    // read incoming frames, parse ClientMessage, send to room actor
    // ...
});
let send_handle = tokio::spawn(async move {
    // receive from mpsc channel, write frames to sender
    // ...
});

tokio::select! {
    _ = recv_handle => {},
    _ = send_handle => {},
}

// cleanup here
```

**Redis pub/sub needs a dedicated connection:**

The connection that calls `SUBSCRIBE` enters a special mode and can only receive pub/sub messages. You cannot reuse it for normal Redis commands. Create it directly from the `redis::Client`, not from the `deadpool` pool:

```rust
let dedicated_conn = redis_client.get_async_pubsub().await?;
```

**CancellationToken is cleaner than abort():**

Rather than calling `handle.abort()` on shutdown, pass a `CancellationToken` clone into each task and check `cancel.cancelled()` in a `select!`. The task gets to finish its current item before exiting.

**The room actor owns all room state:**

Don't reach into `Arc<Mutex<RoomMap>>` from every handler. Build a `RoomHandle` that wraps an mpsc sender, and implement all operations as methods on it. Handlers call `room.add_song(42, "alice").await`. The actor decides what to do.

**Parsing WebSocket messages:**

Define an enum for client messages and deserialize with serde:

```rust
#[derive(serde::Deserialize)]
#[serde(tag = "action", rename_all = "snake_case")]
enum ClientMessage {
    Add { song_id: i64 },
    Skip,
    Reorder { from: usize, to: usize },
    Remove { position: usize },
}

// In the recv task:
if let Ok(msg) = serde_json::from_str::<ClientMessage>(&text) {
    match msg {
        ClientMessage::Add { song_id } => { /* ... */ }
        ClientMessage::Skip => { /* ... */ }
        // ...
    }
}
```

## Testing Your Implementation

```bash
# Start services
docker-compose up -d redis

# Start queuemaster
cargo run

# Register and login
curl -X POST http://localhost:8081/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"hunter2"}'

TOKEN=$(curl -s -X POST http://localhost:8081/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"hunter2"}' | jq -r .access_token)

# Connect to WebSocket (websocat must be installed: cargo install websocat)
websocat "ws://localhost:8081/queue/room1?token=$TOKEN"

# In another terminal, connect a second client
websocat "ws://localhost:8081/queue/room1?token=$TOKEN"

# In the first terminal, type:
{"action":"add","song_id":42}
# Both terminals should show the queue_updated event

# Check the queue via REST
curl -H "Authorization: Bearer $TOKEN" http://localhost:8081/queue/room1
```

## Stretch Goals

These are not required but push you into genuinely interesting territory:

**Vote to Skip** — The skip action only succeeds if at least 50% of connected listeners agree. Each listener can vote, and when the threshold is reached, the song skips automatically. Requires tracking votes per-song-per-room in the actor state.

**DJ Mode** — One user (the DJ) controls the queue. Others can only suggest songs. The DJ accepts or rejects suggestions. Implement as a new `role` field in the room state and permission checks in the actor.

**Song Suggestions with Auto-Queue** — Connect to an external API (Last.fm, Spotify recommendations) to suggest what to play next when the queue is empty. The actor watches queue length and triggers a suggestion fetch when it drops to zero.

**Crossfade Metadata** — Extend the `add` message to accept `{"action":"add","song_id":42,"crossfade_ms":3000}`. Store the crossfade duration with the queue entry. Clients use this to start loading the next song before the current one ends.

**Reconnection with State Recovery** — When a client disconnects and reconnects within 60 seconds, send them a `room_state` event with `{"reconnected": true}` so their UI can smoothly pick up where it left off rather than showing a "disconnected" flash.

**Presence Expiry** — Track last-seen timestamps for listeners in Redis. A background task sweeps every 60 seconds and marks inactive listeners as offline, even if their TCP connection is still alive. This handles clients that are online but not sending any messages.

## Common Problems

**`No such host: redis` or connection refused**
Redis isn't running. Start it: `docker run -d -p 6379:6379 redis:7` or `redis-server`. Verify with `redis-cli ping` — it should return `PONG`.

**WebSocket upgrade returns 400**
Missing required WebSocket headers. Use a proper WebSocket client (websocat, browser, or correctly formed curl). Also confirm `axum = { version = "0.7", features = ["ws"] }` is in Cargo.toml.

**JWT `InvalidSignature` error**
The secret used to validate doesn't match the secret used to sign. Make sure the `JWT_SECRET` environment variable is the same in both the signing path (`/auth/login`) and the validation path (the `AuthUser` extractor).

**`broadcast::error::RecvError::Lagged`**
Your subscriber is processing messages slower than they're being sent. The broadcast channel dropped old messages. Either increase the channel capacity (`broadcast::channel(1024)`) or process messages faster.

**`MutexGuard` across `.await`**
You held a Mutex lock and then called `.await`. This will either fail to compile or deadlock at runtime. Use `tokio::sync::Mutex` instead of `std::sync::Mutex` for async code, OR drop the lock before the await point by limiting the lock scope with a block:
```rust
let value = {
    // .unwrap() is correct for Mutex::lock() — the only failure is Mutex poisoning
    // (another thread panicked while holding the lock), which is a programming error.
    let guard = state.map.lock().unwrap();
    guard.get("key").cloned()
};  // lock dropped here
some_async_fn(value).await;
// ...
```

**`deadpool-redis` version mismatch with `redis` crate**
`deadpool-redis` re-exports `redis` — use the version it bundles rather than adding a separate `redis` dependency with a conflicting version. Check `cargo tree | grep redis` to see what version is pulled in.

**Pub/sub connection gets stuck after subscribing**
A connection that calls `SUBSCRIBE` enters pub/sub mode and can only receive pub/sub messages — you cannot reuse it for `GET`/`SET` commands. Keep a dedicated `redis::Client` for subscriptions, separate from the `deadpool` pool used for normal commands.
