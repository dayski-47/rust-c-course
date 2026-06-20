# 02 - WebSockets and Real-Time Communication 🟡

HTTP is request-response: the client asks, the server answers, and the connection closes. That model breaks down when you need the server to push data to the client - like when another user adds a song to the shared queue. You don't want the client to poll every second asking "anything new?" You want the server to say "hey, the queue changed" the moment it happens.

WebSockets solve this. After an initial HTTP handshake, the connection upgrades to a persistent, bidirectional channel. Either side can send a message at any time. The connection stays open until one side closes it or the network drops.

## The Upgrade Handshake

A WebSocket connection starts as a plain HTTP GET request with special headers:

```
GET /queue/room1 HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

The server responds with `101 Switching Protocols`, and from that point on the TCP connection speaks the WebSocket frame protocol, not HTTP.

Axum handles the entire handshake for you through the `WebSocketUpgrade` extractor. You write a handler that accepts the upgrade, and Axum does the rest.

## WebSocket Handler in Axum

```toml
# Cargo.toml
axum = { version = "0.7", features = ["ws"] }
```

```rust
use axum::{
    extract::{Path, State, WebSocketUpgrade},
    response::IntoResponse,
};
use axum::extract::ws::{Message, WebSocket};

// The handler receives an upgrade object, not the socket directly.
// The socket only exists after you call on_upgrade().
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

The `on_upgrade` call gives you a future that the HTTP layer resolves. Inside that closure you get the actual `WebSocket` object. The handler returns a `101` response immediately; the rest happens inside the async task that `on_upgrade` spawns.

## Splitting the Socket

A WebSocket has two halves: a sink for sending and a stream for receiving. Splitting them lets you move each into its own task, which is the pattern you want for a production server.

```rust
use futures::{SinkExt, StreamExt};
use axum::extract::ws::{Message, WebSocket};

async fn handle_socket(socket: WebSocket, room_id: String, state: AppState) {
    let (mut sender, mut receiver) = socket.split();

    // Task 1: Read from WebSocket, process actions
    let recv_task = tokio::spawn(async move {
        while let Some(result) = receiver.next().await {
            match result {
                Ok(Message::Text(text)) => {
                    // Parse and process the client's message
                    println!("Received: {}", text);
                }
                Ok(Message::Close(_)) => {
                    println!("Client closed connection");
                    break;
                }
                Ok(Message::Ping(data)) => {
                    // Axum handles Pong automatically in recent versions
                    // but you can send manually if needed
                }
                Err(e) => {
                    eprintln!("WebSocket error: {}", e);
                    break;
                }
                _ => {} // Binary, Pong - ignore for now
            }
        }
    });

    // Task 2: Receive from broadcast channel, write to WebSocket
    let (broadcast_tx, mut broadcast_rx) = tokio::sync::mpsc::channel::<String>(32);
    // (register broadcast_tx with room state here)

    let send_task = tokio::spawn(async move {
        while let Some(msg) = broadcast_rx.recv().await {
            if sender.send(Message::Text(msg)).await.is_err() {
                break; // Client disconnected
            }
        }
    });

    // Wait for either task to finish (client disconnect or send error)
    tokio::select! {
        _ = recv_task => {},
        _ = send_task => {},
    }

    // When we fall through here, clean up: remove this client from the room
    println!("Client {} disconnected", "...");
}
```

The `select!` at the end is important. If the client closes the connection, `recv_task` ends. If the server's broadcast channel closes, `send_task` ends. Either way you clean up and exit.

## Message Types

WebSocket frames have a type field. The relevant ones:

| Type | Use |
|------|-----|
| `Text` | JSON messages - this is what you'll use for application data |
| `Binary` | Raw bytes - useful for audio data or binary protocols |
| `Ping` | Keep-alive probe from either side |
| `Pong` | Automatic reply to Ping |
| `Close` | Graceful close with optional status code and reason |

For `queuemaster`, all application messages are JSON text frames.

## Broadcasting to Multiple Clients

The coordinator pattern: a shared map holds one `mpsc::Sender` per connected client. When an event happens, you clone the event and send it to every sender in the map.

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{Mutex, mpsc};
use uuid::Uuid;

// A room holds all connected clients
#[derive(Default)]
pub struct Room {
    // client_id -> channel sender
    pub clients: HashMap<Uuid, mpsc::Sender<String>>,
}

pub type RoomMap = Arc<Mutex<HashMap<String, Room>>>;

// Called when a new client connects
pub async fn register_client(
    rooms: &RoomMap,
    room_id: &str,
    client_id: Uuid,
    sender: mpsc::Sender<String>,
) {
    let mut map = rooms.lock().await;
    let room = map.entry(room_id.to_string()).or_default();
    room.clients.insert(client_id, sender);
}

// Called when broadcasting an event
pub async fn broadcast_to_room(rooms: &RoomMap, room_id: &str, message: String) {
    let map = rooms.lock().await;
    if let Some(room) = map.get(room_id) {
        for (client_id, sender) in &room.clients {
            // try_send won't block - if the buffer is full, we skip that client
            if sender.try_send(message.clone()).is_err() {
                // Client is slow or disconnected - handle in cleanup
                eprintln!("Client {} is not keeping up", client_id);
            }
        }
    }
}

// Called when a client disconnects
pub async fn unregister_client(rooms: &RoomMap, room_id: &str, client_id: Uuid) {
    let mut map = rooms.lock().await;
    if let Some(room) = map.get_mut(room_id) {
        room.clients.remove(&client_id);
    }
}
```

The key insight: the `Mutex` protects the map, but you only hold it long enough to copy the senders. The actual sending happens outside the lock, one per client. If you held the mutex while sending, you'd block all other clients from joining or leaving while you waited on slow network writes.

## Complete Echo Server Example

Before you build the real thing, implement this. It proves your WebSocket stack works end to end.

```rust
use axum::{Router, extract::WebSocketUpgrade, response::IntoResponse};
use axum::extract::ws::{Message, WebSocket};
use futures::{SinkExt, StreamExt};

async fn echo_handler(ws: WebSocketUpgrade) -> impl IntoResponse {
    ws.on_upgrade(|socket| echo_socket(socket))
}

async fn echo_socket(socket: WebSocket) {
    let (mut sender, mut receiver) = socket.split();

    while let Some(Ok(msg)) = receiver.next().await {
        match msg {
            Message::Text(text) => {
                // Echo back with a prefix
                let reply = format!("echo: {}", text);
                if sender.send(Message::Text(reply)).await.is_err() {
                    break;
                }
            }
            Message::Close(_) => break,
            _ => {}
        }
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/echo", axum::routing::get(echo_handler));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

Test it with `websocat`:
```bash
websocat ws://localhost:8080/echo
hello
# echo: hello
```

## Heartbeat / Ping-Pong

TCP connections can silently die. A client might go offline without sending a close frame - their phone went to sleep, the VPN dropped, the router rebooted. If you don't detect this, you'll keep a dead entry in your room map forever, and broadcasts to that client will pile up.

The solution is a periodic ping. If the client doesn't pong back within N seconds, consider them gone.

```rust
use tokio::time::{interval, Duration, timeout};

async fn handle_socket_with_heartbeat(socket: WebSocket, room_id: String) {
    let (mut sender, mut receiver) = socket.split();
    let mut heartbeat = interval(Duration::from_secs(30));

    loop {
        tokio::select! {
            msg = receiver.next() => {
                match msg {
                    Some(Ok(Message::Text(text))) => {
                        // process message
                    }
                    Some(Ok(Message::Pong(_))) => {
                        // Client is alive - reset any dead-client timer
                    }
                    None | Some(Err(_)) | Some(Ok(Message::Close(_))) => break,
                    _ => {}
                }
            }
            _ = heartbeat.tick() => {
                // Send a ping every 30 seconds
                if sender.send(Message::Ping(vec![])).await.is_err() {
                    break; // Can't reach client
                }
            }
        }
    }
    // Client disconnected - unregister here
}
```

## Scaling: One Task Per Client

Every connected WebSocket client needs two tasks: one reading from the socket, one writing to it. With 1,000 connected clients, you have 2,000 tasks. Tokio tasks are lightweight (a few KB each), so this is fine. You are not spawning OS threads.

```
  ┌─────────────────────────────────────────────────────┐
  │                     Tokio Runtime                    │
  │                                                     │
  │  Client A ──(recv task)──→ [Room Coordinator]       │
  │  Client A ←─(send task)──    ↕ mpsc channel         │
  │                                                     │
  │  Client B ──(recv task)──→ [Room Coordinator]       │
  │  Client B ←─(send task)──    ↕ mpsc channel         │
  │                                                     │
  │  Client C ──(recv task)──→ [Room Coordinator]       │
  │  Client C ←─(send task)──    ↕ mpsc channel         │
  └─────────────────────────────────────────────────────┘
```

The room coordinator is the single task that knows about all clients in a room. Client tasks communicate with it over channels, not by directly touching shared state. This avoids lock contention and makes the flow easy to reason about.

## How It Breaks

- Client disconnect without close frame: browsers don't always send a proper close frame (e.g., if the user closes the tab while mobile data drops). The server side doesn't know the client is gone until a send fails. You need heartbeat pings to detect this.
- Sending to a client that disconnected: `write_all` on the socket returns an error. If you don't remove disconnected clients from your broadcast list, you'll keep failing on every broadcast.
- Message framing: WebSocket has built-in message framing (unlike TCP). A "message" arrives complete. But very large messages might be fragmented at the TCP level and reassembled by the WebSocket library - don't assume small messages.
- Close message on upgrade fail: if your upgrade handler panics, the HTTP connection is dropped without a proper WebSocket close. The client gets a connection reset.

## Common Mistakes

**Not handling the close frame.** When a client sends `Message::Close`, your loop must break. If you keep calling `receiver.next()` after close, you'll get errors. Match on it explicitly and exit cleanly.

**Holding the room lock while sending.** If you take a `Mutex` lock and then `send().await` inside it, you hold the lock for the duration of every network write. One slow client blocks all other room operations. Clone the senders first, then drop the lock, then send.

**Using a single task for both reading and writing.** If you try to receive and send in the same task without splitting the socket, you cannot do both concurrently. You'll receive a message, process it, then send a reply - but while you're waiting to receive the next message, you can't send heartbeats or broadcast messages. Split the socket.

**Forgetting to unregister disconnected clients.** When the recv or send task exits, call `unregister_client`. If you don't, the room map grows forever, and broadcasts keep trying to send to dead channels.
