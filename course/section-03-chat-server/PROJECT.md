# Section 3 Project: rustchat - Multi-Client TCP Chat Server

This is the hardest project in the course. Not because the concepts are abstract - you've just read about all of them. It's hard because you have to use them all at once, under concurrency, where the borrow checker is at its most demanding. Real Rust engineers find this section challenging. That's normal. Push through.

When it works, it's genuinely cool: multiple people connecting with `nc`, messages flying between them in real time. You built a networked server from scratch with a language that made you think hard about memory and ownership at every step.

---

## What You're Building

`rustchat` is a real-time, terminal-based multi-client chat server. Multiple people connect to it simultaneously using `nc` (netcat) from their terminals. When one person types a message and hits Enter, every other connected person sees it immediately.

Think of it as IRC without the protocol complexity. No HTTP, no WebSockets, no JSON. Raw TCP, line-based messages, and concurrent threads.

Features:
- Multiple clients connected simultaneously (target: 50+)
- Each client picks a username when they connect
- Messages from any client are broadcast to all others
- Join/leave announcements are sent to everyone in the room
- Server logs all events to stdout with timestamps
- Clean handling of clients that disconnect abruptly

Test it after each milestone:
```bash
# Terminal 1: start the server
cargo run -- --port 8080

# Terminal 2: first client
nc localhost 8080

# Terminal 3: second client
nc localhost 8080
```

---

## Architecture

The server uses a hybrid design: channels for message delivery, shared state for the client registry.

```
                    ┌─────────────────────────────────────┐
                    │           Main Thread                │
                    │   TcpListener::bind("0.0.0.0:8080") │
                    │   loop { listener.accept() }         │
                    └──────────────┬──────────────────────┘
                                   │ for each new connection
                                   ▼
              ┌────────────────────────────────────────────┐
              │         spawn ClientHandler thread          │
              │                                            │
              │  ┌─────────────────┐   ┌────────────────┐ │
              │  │  Read Thread    │   │  Write Thread  │ │
              │  │                 │   │                │ │
              │  │ BufReader reads │   │ for msg in rx  │ │
              │  │ from TcpStream  │   │   write to     │ │
              │  │                 │   │   TcpStream    │ │
              │  │ sends to        │   │                │ │
              │  │ broadcast_tx ───┼──▶│ ◀── msg_rx     │ │
              │  └─────────────────┘   └────────────────┘ │
              └────────────┬───────────────────────────────┘
                           │ broadcast_tx (mpsc::Sender<ChatMessage>)
                           ▼
              ┌────────────────────────────────────────────┐
              │           Broadcaster Thread               │
              │                                            │
              │  for msg in broadcast_rx {                 │
              │      let map = clients.lock();             │
              │      for (name, tx) in map.iter() {        │
              │          if name != msg.from {             │
              │              tx.send(msg.text.clone());    │
              │          }                                  │
              │      }                                     │
              │  }                                         │
              └──────────────────────────────────────────┬─┘
                                                         │
              ┌──────────────────────────────────────────▼─┐
              │      Arc<Mutex<HashMap<String, Sender>>>    │
              │                                            │
              │  "alice" ──▶ Sender<String>  ──▶ Alice's  │
              │  "bob"   ──▶ Sender<String>  ──▶ Bob's    │
              │  "carol" ──▶ Sender<String>  ──▶ Carol's  │
              │              write thread receives         │
              └────────────────────────────────────────────┘
```

**Key relationships:**

- There is ONE broadcaster thread, with ONE mpsc Receiver.
- Every client read thread has a clone of the broadcaster's mpsc Sender.
- Each client has its OWN personal mpsc channel (tx/rx pair).
- The broadcaster holds all clients' personal Senders in the HashMap.
- The client map (`Arc<Mutex<HashMap<...>>>`) is cloned into every thread that needs it.

---

## Data Flow for One Message

```
Alice types "hello"
        │
        ▼
Alice's read thread reads "hello" from TcpStream
        │
        ▼
read thread sends ChatMessage { from: "alice", text: "hello" }
        to broadcast_tx (the broadcaster's Sender)
        │
        ▼
Broadcaster thread receives the message from broadcast_rx
        │
        ├──▶ locks client map
        ├──▶ skips "alice" (the sender)
        ├──▶ sends "alice: hello" to Bob's personal Sender
        ├──▶ sends "alice: hello" to Carol's personal Sender
        └──▶ releases client map lock
                │
                ├──▶ Bob's write thread: write to Bob's TcpStream
                └──▶ Carol's write thread: write to Carol's TcpStream
```

---

## Project Setup

```bash
cd course/section-03-chat-server
cargo new rustchat --bin
cd rustchat
```

`Cargo.toml`:
```toml
[package]
name = "rustchat"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "rustchat"
path = "src/main.rs"

[dependencies]
thiserror = "1"
```

Suggested file layout:
```
src/
├── main.rs       # argument parsing, TcpListener loop, spawn broadcaster
├── server.rs     # Server struct, client registration, broadcast logic
├── client.rs     # handle_client, read loop, write loop
└── error.rs      # ChatError type
```

Start with everything in `main.rs`. Extract into modules as the file gets big.

---

## Critical Rust Patterns for This Project

These four patterns are the skeleton of the server. You cannot avoid them. Understand the shape before you start coding.

### 1. Splitting TcpStream for Concurrent Read + Write

You can't give the same TcpStream to two threads (one to read, one to write) without cloning it. The `try_clone()` method creates a second handle to the same underlying socket:

```rust
let stream = /* TcpStream from listener.accept() */;
let write_stream = stream.try_clone()?;
// stream: give to the read thread
// write_stream: give to the write thread
```

Both handles point to the same OS socket. Reading from `stream` and writing to `write_stream` works correctly - TCP sockets are full-duplex.

### 2. Sharing the Client Map Across Threads

The client map must be shared across: the main thread (add new clients), client read threads (remove disconnected clients), and the broadcaster thread (iterate senders).

The pattern is always:

```rust
use std::sync::{Arc, Mutex};
use std::collections::HashMap;
use std::sync::mpsc::Sender;

type ClientMap = Arc<Mutex<HashMap<String, Sender<String>>>>;

// Create once in main:
let clients: ClientMap = Arc::new(Mutex::new(HashMap::new()));

// Clone before moving into each thread - cheap, just increments a counter:
let clients_for_thread = Arc::clone(&clients);
thread::spawn(move || {
    // use clients_for_thread here
});
```

### 3. The mpsc Channel Setup

The broadcaster needs ONE receiver. Every client read thread gets a CLONE of the sender:

```rust
use std::sync::mpsc;

// Create the broadcast channel once:
let (broadcast_tx, broadcast_rx) = mpsc::channel::<ChatMessage>();

// For each new client, clone the sender:
let client_broadcast_tx = broadcast_tx.clone();
thread::spawn(move || {
    // client read thread uses client_broadcast_tx
});

// The broadcaster thread takes the receiver:
thread::spawn(move || {
    for msg in broadcast_rx {
        // fan out to all clients
    }
});
```

### 4. Locking the Map Safely (Don't Hold Lock During I/O)

Lock briefly, get what you need, unlock, then do I/O:

```rust
// WRONG: holding the lock while writing to sockets
let map = clients.lock().unwrap();
for (_, tx) in map.iter() {
    tx.send(msg.clone()).ok();  // send through channel (fast, ok)
}
// map dropped here, lock released

// The actual socket write happens in each client's write thread
// AFTER the lock is released - that's the whole point of the channel design
```

---

## Message Types

Define the internal message type early - it'll clarify your thinking:

```rust
#[derive(Debug, Clone)]
pub struct ChatMessage {
    pub from: String,          // username
    pub text: String,          // message text
    pub kind: MessageKind,
}

#[derive(Debug, Clone)]
pub enum MessageKind {
    Chat,      // regular message
    Join,      // "[username] joined"
    Leave,     // "[username] left"
    System,    // server announcements
}
```

---

## Engineering Approach: Concurrent System Design Before Code

Before writing any concurrent code, answer three questions:

1. **What shared state exists?** For the chat server: the list of connected clients.
2. **Who reads it and who writes it?** Readers: every thread that broadcasts a message to all clients. Writers: the thread that adds a new client on connect, and the thread that removes a client on disconnect.
3. **What messages flow between components?** Client to server: a text message. Server to client: a broadcast message. System events: join and leave announcements.

The answers tell you exactly which concurrency primitives to use:
- Shared mutable state accessed by many threads → `Arc<Mutex<T>>`
- One-way message passing → `mpsc` channel (sender cloned per client, one receiver for the broadcaster)

The design for this server: each client gets two threads - one that reads from the socket, one that writes to it. The reader threads send messages to a single broadcaster channel. The broadcaster fans out each message to every client's dedicated write channel.

Draw this on paper before writing code. It should look like boxes for threads, arrows showing data flow, and labels on each arrow indicating what data crosses it. If you can't draw it clearly, you don't understand it clearly enough to implement it.

---

## What You Bring From Sections 1 and 2

This is the first project where you're building something genuinely concurrent. But none of the tools are new:

- **Threads (new in S3):** `thread::spawn` is introduced here for the first time. Each connected client gets its own thread. The clone-before-move pattern (`Arc::clone` before `move` into the closure) is a rule you'll apply in every concurrent project going forward.
- **S1 Result and error handling:** `TcpStream` operations return `Result`. The chat server error handling follows the exact same `?` propagation pattern you used for `File::open` in the hex viewer - just applied to socket I/O instead of file I/O.
- **Arc<Mutex> (new in S3):** The chat server uses `Arc<Mutex<HashMap<String, Sender<String>>>>` for the client list. This is the canonical pattern for shared mutable state across threads. You will use it in every concurrent project from here.
- **S2 structs:** `ChatMessage` is a struct. `Client` is a struct. You're doing the same data modeling you practiced in Section 2, now for a networking domain.
- **S2 enums:** `MessageKind` is an enum. The same exhaustive pattern matching with `match` applies here.
- **S2 `HashMap`:** The client registry is a `HashMap<username, Sender>`. Same HashMap you've used before - different key and value types.

The difficulty is not any individual concept. The difficulty is combining all of them under concurrency, where the borrow checker enforces rules you could previously ignore.

---

## How This Project Works in Rust - The Full Picture

The chat server operates through three Rust concepts working together simultaneously:

**TCP sockets.** `TcpListener` binds to a port and produces `TcpStream` values as clients connect. Each `TcpStream` is a full-duplex connection - you can read from it and write to it independently. `try_clone()` creates a second handle to the same underlying socket, which lets you give one handle to a read thread and one to a write thread without violating ownership.

**Shared state.** Multiple threads need access to the client registry - to add clients when they connect, remove them when they disconnect, and get write channels when broadcasting. `Arc` makes the `HashMap` accessible from any number of threads without copying it. `Mutex` ensures only one thread can modify the map at a time, preventing data races.

**Channels.** The broadcaster does not write to each client's socket directly. Instead, each client has a dedicated `mpsc` channel. Broadcasting means: lock the client map briefly, clone all the `Sender` handles, release the lock, then send a message to each `Sender`. The actual socket write happens in each client's write thread, after the lock is released. This pattern keeps lock-hold time short and avoids doing I/O under a lock.

The lock-briefly-then-act pattern is the key insight of this project. Locking while doing I/O is the primary source of both deadlocks and poor performance in concurrent networked servers. Lock to get what you need, release immediately, then do the work.

---

## Milestones

### Milestone 1: Single-Client Echo Server 🟢

Accept one connection. Read whatever the client sends. Echo it back. Don't handle multiple clients yet.

Goal: prove you understand `TcpListener`, `TcpStream`, `BufReader`, and the basic read/write loop.

```bash
nc localhost 8080
# type anything, see it echoed back
```

Check: does the server handle client disconnect without panicking?

**Expected output when working:**
```
$ cargo run -- --port 8080
rustchat listening on 0.0.0.0:8080
[14:23:01] New connection from 127.0.0.1:54231
[14:23:05] New connection from 127.0.0.1:54232
```
Test with: `nc localhost 8080` in another terminal. You should see the connection log.

---

### Milestone 2: Multi-Client (Threads) 🟢

Spawn a thread for each incoming connection. Each thread handles its own client independently.

Goal: prove `thread::spawn(move || handle_client(stream))` works and that clients don't block each other.

Check: connect two netcat clients. Each can communicate independently.

---

### Milestone 3: Shared Client Registry 🟡

Add `Arc<Mutex<HashMap<String, ...>>>` to track connected clients.

For now, just track usernames (no broadcasting yet). When a client connects, add them. When they disconnect, remove them.

Print the client count from the server every time someone connects or disconnects.

Goal: prove the `Arc<Mutex<...>>` pattern compiles and works under concurrent access.

Hint: The borrow checker will fight you. Remember: clone the `Arc` before the `move ||` closure.

Check: the server prints "2 clients connected" when two nc sessions are active.

---

### Milestone 4: Broadcast Messages 🔴

When any client sends a message, send it to all OTHER connected clients.

This is where the design gets real. Each client needs a way to receive messages from other clients. An mpsc channel per client is the right tool: the client's write thread listens on `rx`, and the broadcaster sends to `tx`.

You'll need to store the `Sender<String>` in the HashMap instead of (or alongside) the username.

Hint: `HashMap<String, Sender<String>>` where the key is the username.

Check: connect two nc clients. Message from one appears in the other's terminal.

**Expected output when working:**
```
# Alice's terminal:                    # Bob's terminal:
Enter username: alice                  Enter username: bob
Welcome, alice!                        Welcome, bob!
hello everyone                         [alice] hello everyone
```
This is the milestone where the project becomes real. When Bob sees Alice's message appear, it's working.

---

### Milestone 5: Username Negotiation 🟡

When a client first connects, before entering the main chat loop, ask them for a username.

The first line they send is their username. Validate it:
- Not empty
- No spaces
- 1-20 characters
- Not already taken

If validation fails, send an error message and close the connection.

Hint: read one line first (before the main message loop), validate it, register in the map, then enter the loop.

Check: try to connect with a username that's already taken. The server should reject it.

**Expected output when working:**
```
# Server terminal:
[14:23:01] alice connected from 127.0.0.1:54231

# Client terminal (nc localhost 8080):
Enter username: alice
Welcome, alice! 2 users online.
```

---

### Milestone 6: Join and Leave Announcements 🟢

When a user joins, broadcast `"[alice] joined"` to everyone already connected.
When a user leaves, broadcast `"[alice] left"` to everyone still connected.

This should trigger on both clean disconnects (EOF) and network errors.

Check: two clients connected. Third connects and both see "[carol] joined". Carol disconnects and both see "[carol] left".

**Expected output when working:**
```
# Bob's terminal when Carol joins:
*** carol has joined the room

# Carol's terminal when Bob leaves:
*** bob has left the room
```

---

### Milestone 7: Broadcaster Thread Architecture 🔴

Refactor: instead of each client handler directly iterating the client map and sending messages, introduce a dedicated broadcaster thread.

Client read threads send `ChatMessage` structs to a shared `mpsc::Sender<ChatMessage>`. The broadcaster thread owns the `Receiver<ChatMessage>` and fans messages out to all clients.

This is a cleaner separation of concerns and easier to extend.

Why bother? Because the current design (locking the map in every handler thread while sending) can cause contention under load. The broadcaster bottlenecks the sending, but the handlers no longer fight over the map lock.

```
client read threads ──▶ broadcast_tx (clones)
                    ──▶ broadcast_tx (clones)
                    ──▶ broadcast_tx (clones)
                              │
                              ▼
                    Broadcaster thread (broadcast_rx)
                              │
                    locks map once per message
                    sends to each client's Sender
```

Hint: `broadcast_tx` is cloned (not moved) into each client thread. The original sender and its receiver are created in `main`, and the receiver is moved into the broadcaster thread.

---

### Milestone 8: Graceful Shutdown 🟡

Handle Ctrl+C. When the server receives SIGINT:
1. Stop accepting new connections
2. Send a shutdown message to all connected clients
3. Close the listener

Simple approach: use an `Arc<AtomicBool>` as a running flag. Set it to `false` in a Ctrl+C handler. Check it in the accept loop.

For the netcat-based test, you can also just kill the process - netcat clients will disconnect cleanly when the server closes.

---

## Hints (Not Solutions)

**The type you're building toward:**

```rust
type ClientMap = Arc<Mutex<HashMap<String, Sender<String>>>>;
```

This is what gets cloned into every thread. The `String` key is the username. The `Sender<String>` is how you get messages to that client's write thread.

**Cloning before moving - the required pattern:**
```rust
let clients_for_thread = Arc::clone(&clients);
thread::spawn(move || {
    handle_client(stream, clients_for_thread);
});
// clients is still usable here
```

**Splitting read and write** (inside a function returning `Result<(), io::Error>`):
```rust
// try_clone() returns Result - propagate the error with ? instead of panicking
let write_stream = stream.try_clone()?;
let read_stream = stream;  // moved into BufReader
// ...
```

**Registering a client:**
```rust
let (client_tx, client_rx) = mpsc::channel::<String>();
{
    // .unwrap() on Mutex::lock() is correct here - the only way it returns Err
    // is if the Mutex is "poisoned" (another thread panicked while holding it).
    // That's a programming error, so panicking is the right response.
    let mut map = clients.lock().unwrap();
    map.insert(username.clone(), client_tx);
}  // lock released here - important
// ...
```

**The write thread pattern:**
```rust
thread::spawn(move || {
    for msg in client_rx {  // blocks until a message arrives or channel closes
        if write_stream.write_all(format!("{msg}\n").as_bytes()).is_err() {
            break;  // client disconnected on the write side
        }
    }
});
```

**When a client disconnects (read side returns):**
```rust
// Remove from map so no one tries to send to them
// .unwrap() is intentional - see "Registering a client" note above
clients.lock().unwrap().remove(&username);
// The Sender is dropped from the map - this closes the write thread's Receiver
// The write thread's for loop ends, thread terminates
// ...
```

**Broadcasting without holding the lock:**
```rust
// Collect senders first (brief lock), then send (no lock)
let senders: Vec<Sender<String>> = {
    let map = clients.lock().unwrap();  // .unwrap() intentional - see above
    map.iter()
        .filter(|(name, _)| *name != skip)
        .map(|(_, tx)| tx.clone())
        .collect()
};  // lock released here
for tx in senders {
    tx.send(msg.clone()).ok();  // .ok() because receiver might have disconnected
}
```

**Detecting EOF vs error on reads:**
```rust
match reader.read_line(&mut line) {
    Ok(0) => break,          // EOF - clean disconnect
    Ok(_) => { /* process */ }
    Err(e) => {
        eprintln!("Read error: {e}");
        break;
    }
}
```

---

## Common Compile Errors in This Project

**"cannot move out of `stream` because it is borrowed"**
You tried to move `stream` into a thread after already borrowing it. Call `try_clone()` first and move the clone into the thread.

**"the trait `Send` is not implemented for `MutexGuard`"**
You held a `MutexGuard` across a `.await` boundary (or tried to send it to another thread). Drop the guard before crossing the thread/async boundary: add an explicit scope `{ let guard = ...; /* use */ }` or call `drop(guard)` before the boundary.

**"closure may outlive the current function"**
The closure passed to `thread::spawn` references data that might not live long enough. Fix: use `move` keyword on the closure, and `Arc::clone` anything you need inside. The `move` keyword transfers ownership of the captured variables into the closure.

**"`broadcast_rx` moved into closure"**
You tried to clone `broadcast_rx`. You can't - `Receiver` is not `Clone`. Only `Sender` is `Clone`. This is intentional: the broadcaster owns the one `Receiver`, and clients each own a cloned `Sender`.

---

## Stretch Goals

**Private messages.** Detect messages starting with `/pm username ` and route them only to that user. Look up their `Sender` in the client map and send directly.

**Rooms.** Add `/join #roomname` and `/leave #roomname` commands. The client map becomes a `HashMap<String, HashMap<String, Sender<String>>>` - room name → (username → sender). Only broadcast within the current room.

**Message history.** Store the last 50 messages in an `Arc<Mutex<VecDeque<String>>>`. When a new client connects, send them the history before entering the main loop.

**Simple client binary.** Write a second binary in the same project that connects to the server, reads lines from stdin (in one thread) and prints incoming messages (in another thread). This gives a better experience than raw netcat.

**Timestamps.** Include a timestamp in each broadcast message. Use `std::time::SystemTime` or the `chrono` crate.

**Client timeout.** Disconnect clients that haven't sent anything in 5 minutes. Use `TcpStream::set_read_timeout()`.

---

## Testing Your Server

**Basic connectivity:**
```bash
cargo run -- --port 8080
# in another terminal:
nc localhost 8080
```

**Multi-client test (manual):**
```bash
# Three terminals:
nc localhost 8080  # Alice
nc localhost 8080  # Bob
nc localhost 8080  # Carol
```
Type in one, verify the message appears in the others (not in the sender's terminal).

**Stress test - 50 simultaneous connections:**
```bash
# On Linux/macOS, open many connections at once
for i in $(seq 1 50); do
    echo "test message from $i" | nc -q 1 localhost 8080 &
done
wait
```

**Abrupt disconnect test:**
```bash
nc localhost 8080
# Connect, type a username, then hit Ctrl+C (SIGINT)
# Server should print "[username] left" without panicking
```

**Reconnect test:**
```bash
# Connect with username "alice"
# Disconnect
# Connect again with username "alice"
# Should work - alice's name is no longer in the registry
```

---

## Server Logging Format

Use this format consistently so the logs are readable:

```
[2024-01-15 10:23:45] Server started on port 8080
[2024-01-15 10:23:47] alice connected from 127.0.0.1:54321
[2024-01-15 10:23:50] bob connected from 127.0.0.1:54322
[2024-01-15 10:23:52] alice: hello everyone
[2024-01-15 10:23:55] bob: hey alice!
[2024-01-15 10:24:01] alice disconnected
[2024-01-15 10:24:01] broadcast: [alice] left (2 clients remaining → 1)
```

For timestamps without external dependencies:
```rust
fn timestamp() -> String {
    let now = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap();
    // Simple seconds-since-epoch, or format with chrono crate for human-readable
    format!("[t={}s]", now.as_secs())
}
```

---

## What This Project Teaches

When you finish this project you'll have practiced:

- `Arc<Mutex<T>>` under real concurrent load - not a toy example
- mpsc channels for message delivery between threads
- `thread::spawn` with captured state
- `TcpListener` / `TcpStream` for real network I/O
- `BufReader` / `BufWriter` for efficient socket I/O
- `try_clone()` to split a socket into read and write halves
- Custom error types with `thiserror`
- The "lock briefly, collect, release, then act" pattern
- Detecting and cleaning up after client disconnects

More importantly: you'll understand *why* Rust's ownership rules are the way they are. Sharing mutable state across threads is genuinely hard. Rust forces you to be explicit about it, catches mistakes at compile time, and gives you tools (`Arc`, `Mutex`, channels) that make the correct patterns clear.

The next time you read C code that shares a socket descriptor between threads with a global array protected by a mutex, you'll see the exact same pattern - but now you'll understand the guarantees and the failure modes.
