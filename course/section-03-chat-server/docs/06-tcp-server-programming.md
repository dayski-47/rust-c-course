# Doc 05 - TCP Server Programming

Here's where the rubber meets the road. We're going to write a real TCP server - the kind that listens on a port, accepts connections, reads data, and writes responses. If you've done this in C with `socket()`, `bind()`, `listen()`, `accept()`, and `read()`/`write()`, this will feel familiar. Rust's API mirrors the same concepts; it just wraps them in types that prevent you from using a closed socket or forgetting to close one.

---

## The Basics: Bind, Listen, Accept 🟢

In C:
```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
bind(fd, ...);
listen(fd, backlog);
int client_fd = accept(fd, ...);
```

In Rust:
```rust
use std::net::TcpListener;

fn main() {
    // bind() + listen() in one call
    let listener = TcpListener::bind("0.0.0.0:8080").unwrap();
    println!("Listening on port 8080");

    // accept() loop - blocks until a client connects
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        println!("Client connected: {:?}", stream.peer_addr());
        // handle the connection here
    }
}
```

`TcpListener::bind` takes any address string Rust can parse - `"0.0.0.0:8080"` (all interfaces), `"127.0.0.1:8080"` (localhost only), or just `":8080"`. It returns `Result<TcpListener>` because the port might be in use or you might not have permission.

`listener.incoming()` is an iterator that yields `Result<TcpStream>` for each new connection. Each `TcpStream` represents a connected client - equivalent to the file descriptor returned by `accept()` in C.

---

## TcpStream: Reading and Writing 🟢

`TcpStream` implements both `Read` and `Write`. You can call `read()` and `write()` directly, but raw byte reads are painful. You almost always want buffered I/O.

```rust
use std::net::{TcpListener, TcpStream};
use std::io::{BufRead, BufReader, Write};

fn handle_client(stream: TcpStream) {
    let peer = stream.peer_addr().unwrap();
    let mut reader = BufReader::new(stream.try_clone().unwrap());
    let mut writer = stream;  // original stream is used for writing

    // Write a greeting
    writer.write_all(b"Welcome! Type your message:\n").unwrap();

    // Read lines until the client disconnects
    let mut line = String::new();
    loop {
        line.clear();
        match reader.read_line(&mut line) {
            Ok(0) => {
                // 0 bytes read means EOF - client disconnected
                println!("{peer} disconnected");
                break;
            }
            Ok(_) => {
                print!("{peer} says: {line}");
                // Echo the line back
                writer.write_all(line.as_bytes()).unwrap();
            }
            Err(e) => {
                eprintln!("Error reading from {peer}: {e}");
                break;
            }
        }
    }
}

fn main() {
    let listener = TcpListener::bind("0.0.0.0:8080").unwrap();
    for stream in listener.incoming() {
        handle_client(stream.unwrap());
        // Note: this handles one client at a time - we'll fix this next
    }
}
```

`BufReader` wraps the stream and adds buffering. Without it, `read_line` would make a syscall for every byte. With it, the system reads in chunks and delivers them line by line. `BufWriter` does the same for writes - buffering output until you flush or the buffer fills.

---

## try_clone: Splitting Read and Write 🟡

Notice `stream.try_clone()` in the example above. `TcpStream` doesn't implement `Clone` because you can't just bit-copy a socket - it's a kernel resource. But `try_clone()` creates a new `TcpStream` handle that refers to the same underlying socket, similar to `dup()` in C.

This is essential for a pattern we need in the chat server: one thread reads from the socket, another thread writes to it. You can't give the same `TcpStream` to two threads at once (ownership rules), but you can clone it and give one half to each.

```rust
let stream = listener.accept()?.0;
let read_stream = stream.try_clone().expect("failed to clone stream");
let write_stream = stream;  // original goes to the write thread

thread::spawn(move || {
    // reading thread
    let reader = BufReader::new(read_stream);
    for line in reader.lines() { /* ... */ }
});

thread::spawn(move || {
    // writing thread
    // receives messages from a channel and writes them here
});
```

---

## Handling Each Client in Its Own Thread 🟡

The single-threaded version blocks on each client before moving to the next. The real server needs to handle many clients simultaneously:

```rust
use std::net::{TcpListener, TcpStream};
use std::thread;
use std::io::{BufRead, BufReader, Write};

fn handle_client(mut stream: TcpStream) {
    let peer = stream.peer_addr().unwrap();
    let reader = BufReader::new(stream.try_clone().unwrap());

    stream.write_all(b"Connected.\n").unwrap();

    for line in reader.lines() {
        match line {
            Ok(msg) => {
                println!("{peer}: {msg}");
                stream.write_all(format!("Echo: {msg}\n").as_bytes()).unwrap();
            }
            Err(e) => {
                eprintln!("Error from {peer}: {e}");
                break;
            }
        }
    }
    println!("{peer} disconnected");
}

fn main() {
    let listener = TcpListener::bind("0.0.0.0:8080").unwrap();

    for stream in listener.incoming() {
        match stream {
            Ok(stream) => {
                thread::spawn(move || handle_client(stream));
            }
            Err(e) => eprintln!("Accept error: {e}"),
        }
    }
}
```

Now every client gets its own OS thread. The main thread does nothing but call `accept()` in a loop. Each handler thread is independent - a slow or misbehaving client doesn't block others.

---

## Broadcasting to All Connected Clients 🔴

This is the core of the chat server. When client A sends a message, every other client must receive it. We need a shared data structure that every handler thread can reach to get a list of write channels for all clients.

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::sync::mpsc::{self, Sender};
use std::net::{TcpListener, TcpStream};
use std::io::{BufRead, BufReader, Write};
use std::thread;

type Clients = Arc<Mutex<HashMap<String, Sender<String>>>>;

fn handle_client(
    stream: TcpStream,
    username: String,
    clients: Clients,
) {
    let peer = stream.peer_addr().unwrap();
    let mut write_stream = stream.try_clone().unwrap();
    let read_stream = BufReader::new(stream);

    // Create a channel for this client's outgoing messages
    let (tx, rx) = mpsc::channel::<String>();

    // Register this client in the shared map
    clients.lock().unwrap().insert(username.clone(), tx);

    // Spawn a write thread: listens on rx, writes to socket
    thread::spawn(move || {
        for msg in rx {
            if write_stream.write_all(format!("{msg}\n").as_bytes()).is_err() {
                break;
            }
        }
    });

    // Announce arrival
    broadcast_all(&clients, format!("{username} joined"), &username);

    // Read loop
    for line in read_stream.lines() {
        match line {
            Ok(msg) if !msg.is_empty() => {
                broadcast_all(&clients, format!("{username}: {msg}"), &username);
            }
            Ok(_) => {}
            Err(_) => break,
        }
    }

    // Client disconnected - remove from map
    clients.lock().unwrap().remove(&username);
    broadcast_all(&clients, format!("{username} left"), &username);
    println!("{peer} ({username}) disconnected");
}

fn broadcast_all(clients: &Clients, msg: String, skip_user: &str) {
    // Collect senders without holding the lock during sends
    let senders: Vec<Sender<String>> = {
        let map = clients.lock().unwrap();
        map.iter()
            .filter(|(name, _)| name.as_str() != skip_user)
            .map(|(_, tx)| tx.clone())
            .collect()
    };  // lock released here

    for tx in senders {
        tx.send(msg.clone()).ok();  // ignore if recipient disconnected
    }
}
```

The `broadcast_all` function demonstrates the important pattern from chapter 3: lock, collect what you need, release the lock, then do the work (sending messages) without holding the lock. This prevents the mutex from being held across slow channel sends.

---

## Detecting Client Disconnection 🟡

When a client disconnects:
- On a clean disconnect: `read_line` or `lines()` returns `Ok(0)` (zero bytes = EOF)
- On a network error: `read_line` returns `Err(io::Error)`
- The `for line in reader.lines()` iterator ends in either case

When the read loop ends, clean up:
1. Remove the client's `Sender` from the client map
2. The write thread will notice its channel closed (all senders dropped) and terminate
3. Announce the departure to remaining clients

This is why we `.remove()` the sender from the map when the read loop ends - that drops the `Sender`, closing the write thread's `rx`, ending the `for msg in rx` loop, and letting the write thread finish naturally.

---

## Socket Options 🟢

Two useful options for a TCP server:

```rust
use std::net::TcpListener;
use std::time::Duration;

let listener = TcpListener::bind("0.0.0.0:8080").unwrap();

// For client streams:
let stream = listener.accept().unwrap().0;

// Disable Nagle's algorithm - send data immediately, don't buffer
// Good for interactive protocols like chat; bad for bulk transfers
stream.set_nodelay(true).unwrap();

// Timeout for read operations - prevents a hung client from blocking forever
stream.set_read_timeout(Some(Duration::from_secs(300))).unwrap();

// Timeout for write operations
stream.set_write_timeout(Some(Duration::from_secs(30))).unwrap();
```

For the chat server, `set_nodelay(true)` makes messages feel responsive (no 200ms Nagle delay before small packets are sent). Read timeouts let you detect clients that connected and then went silent (disconnected without sending FIN).

---

## Graceful Shutdown

The basic shutdown strategy: set an atomic flag when Ctrl+C is pressed, then have threads check the flag periodically.

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

let running = Arc::new(AtomicBool::new(true));
let r = Arc::clone(&running);

ctrlc::set_handler(move || {
    r.store(false, Ordering::SeqCst);
}).unwrap();

// In the listener loop:
// TcpListener::set_nonblocking(true) + loop { if !running.load(...) { break; } }
```

A simpler approach for this project: let the main thread accept connections normally, and when the server process is killed, the OS cleans up all sockets. The more advanced approach (graceful drain) is a stretch goal.

---

## Testing with netcat

While building, test your server with `nc`:

```bash
# Terminal 1: run your server
cargo run

# Terminal 2: first client
nc localhost 8080

# Terminal 3: second client
nc localhost 8080
```

Type in either terminal and watch the message appear in the other. This is the simplest possible integration test - and it feels great when it works.

---

## Common Mistakes

**Handling clients sequentially instead of spawning threads.** If your `for stream in listener.incoming()` loop calls `handle_client(stream)` directly without `thread::spawn`, each client blocks all others. Always spawn.

**Forgetting to handle EOF.** When `read_line` returns `Ok(0)` or the `lines()` iterator ends, the client has disconnected. Not handling this means your thread runs forever on a closed connection, and the client stays in the client map forever.

**Holding the client map lock while writing to sockets.** This is the deadlock setup from chapter 3. Lock, collect senders, release the lock, then send. Never hold the map lock while doing any I/O.

**Not cloning the stream before passing it to BufReader.** `BufReader::new(stream)` moves the stream into the reader. You can't write to it anymore. Call `try_clone()` first, pass the clone to the reader, keep the original for writing (or vice versa).

**Not dropping the Sender when a client leaves.** If you don't remove the disconnected client's `Sender` from the map, you'll keep trying to send messages to a dead channel. The `send()` will return `Err` but you'll be wasting work, and the map will grow forever.

---

## How It Breaks

**`TIME_WAIT` and "Address already in use".** After a TCP connection closes, the OS keeps that connection's local port in `TIME_WAIT` state for approximately 2 × MSL (typically 60–120 seconds). During this time the OS refuses to reuse the port. If you stop and restart your server quickly during development, `TcpListener::bind()` fails with "Address already in use." The fix is to set `SO_REUSEADDR` on the socket before binding. In Rust, you can't set this through `TcpListener::bind()` directly - use the `socket2` crate, or on Linux rely on the fact that `TcpListener` sets `SO_REUSEADDR` by default (it does on most Rust versions).

**`TcpListener` backlog.** When clients connect faster than your `accept()` loop can call `accept()`, the OS queues them in the TCP backlog. The backlog has a system-defined limit (often 128 on Linux). If connections arrive faster than you accept them and the queue is full, the OS drops new connection attempts silently. For a chat server this is unlikely, but for a high-traffic server, you'd want to ensure your `accept()` loop is as tight as possible and your per-connection handling is offloaded to threads immediately.

**Read returning 0 bytes is EOF, not an error.** When `read()` returns `Ok(0)`, the client has closed the connection - it sent a TCP FIN. If you treat this as "try again later" instead of "connection is done," you'll spin in a tight loop calling `read()` repeatedly, each returning 0 immediately, consuming 100% CPU for that thread forever. The `for line in reader.lines()` iterator handles this correctly - the iterator ends on EOF. But if you write your own `read()` loop, always check `Ok(0)` and break.

**Writing to a disconnected client.** When a client disconnects without your write thread knowing, the next call to `write_all()` on that client's socket returns `Err` with a "Broken pipe" error (or "Connection reset by peer"). If you don't handle this and remove the client from your client map, subsequent broadcasts will hit the same error for that client every time. The map grows stale. The fix: when a write fails, treat it the same as an explicit disconnect - remove the client's sender from the map and let the write thread terminate.
