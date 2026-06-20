# Doc 04 — Concurrency: Threads and Channels

You've seen `thread::spawn` in passing. Now we're going to use it for real — with shared state, with channels, and with enough concurrent connections to run a chat server.

One honest warning: concurrency is where the borrow checker bites hardest. The error messages get longer. The solutions feel indirect at first. But by the end of this chapter you'll have two solid mental models for sharing work between threads: shared state (protected by `Arc<Mutex<T>>`) and message passing (channels). They're complementary, not competing. Good concurrent systems use both.

---

## Threads with Shared State 🟡

You already know `thread::spawn`. The tricky part is what happens when the thread needs data from the outside world.

**Rule:** closures passed to `thread::spawn` must be `'static + Send`. That means they can't borrow from the calling scope (because the calling scope might end before the thread does), and everything they capture must be safe to send to another thread.

The solution for shared data is `Arc`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0_u64));
    let mut handles = vec![];

    for _ in 0..5 {
        // Clone the Arc BEFORE moving it into the thread
        // This is the required pattern
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            let mut guard = counter.lock().unwrap();
            *guard += 1;
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    println!("Final: {}", counter.lock().unwrap());  // 5
}
```

Notice the clone-before-move pattern: `let counter = Arc::clone(&counter);` then `move ||`. The `move` keyword transfers ownership of the cloned `Arc` into the closure. The original `counter` stays in main. Each thread owns its own `Arc` handle, all pointing to the same underlying `Mutex<u64>`.

---

## thread::scope: Borrowing from the Environment 🟢

Sometimes you don't want shared ownership. You just want to run some threads that borrow data from the current scope, and you know they'll finish before the scope ends. For this, `thread::scope` is much simpler:

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4, 5, 6, 7, 8];

    thread::scope(|s| {
        s.spawn(|| {
            let sum: i32 = data.iter().sum();
            println!("Sum: {sum}");
        });

        s.spawn(|| {
            let max = data.iter().max().unwrap();
            println!("Max: {max}");
        });
        // Both threads run concurrently, can read data at the same time
    });
    // scope guarantees: ALL spawned threads have finished by the time
    // we reach this line. No join() needed.

    println!("Scope done, data still here: {:?}", data);
}
```

`thread::scope` is perfect for parallel computations over local data. The compiler proves all threads finish before the scope exits, so it doesn't need `'static` bounds. No `Arc`, no cloning.

In the chat server we'll mostly use regular `thread::spawn` with `Arc` since client handler threads outlive any particular scope.

---

## mpsc Channels: Message Passing 🟡

`mpsc` stands for Multi-Producer, Single-Consumer. You create a channel with one sender and one receiver. The sender can be cloned to create multiple producers. Messages travel in order and the receiver blocks until a message arrives.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();

    let tx2 = tx.clone();  // second producer

    thread::spawn(move || {
        tx.send("hello from thread 1".to_string()).unwrap();
        tx.send("another from thread 1".to_string()).unwrap();
    });

    thread::spawn(move || {
        tx2.send("hello from thread 2".to_string()).unwrap();
    });

    // rx.recv() blocks until a message arrives
    // The channel closes when ALL senders are dropped
    // Iterating over rx drains the channel until it closes
    for msg in rx {
        println!("Got: {msg}");
    }

    println!("All senders dropped, channel closed");
}
```

Key behaviors:
- `tx.send(msg)` returns `Err` only if the receiver has been dropped
- `rx.recv()` blocks until a message arrives or all senders are dropped
- `rx.try_recv()` returns immediately — `Ok(msg)` if available, `Err(TryRecvError::Empty)` if not
- `for msg in rx { }` is equivalent to looping on `rx.recv()` until the channel closes

---

## Bounded vs Unbounded Channels 🟡

`mpsc::channel()` creates an **unbounded** channel — the producer can send as fast as it wants, messages pile up in a queue in memory. If the consumer can't keep up, you run out of RAM.

`mpsc::sync_channel(n)` creates a **bounded** channel with capacity `n`. When the buffer is full, `send()` blocks until the consumer takes a message. This applies **backpressure**: the producer slows down when the consumer is slow.

```rust
// Bounded channel with capacity 10
let (tx, rx) = mpsc::sync_channel::<String>(10);

thread::spawn(move || {
    for i in 0..1000 {
        // BLOCKS if the buffer is full — natural throttling
        tx.send(format!("message {i}")).unwrap();
    }
});

for msg in rx {
    // simulate slow consumer
    println!("{msg}");
}
```

For the chat server: use unbounded channels for client write queues (since we don't want to block other threads on a slow client's send), but consider bounded channels for the broadcaster to prevent memory growth if clients disconnect faster than we clean them up.

---

## The Iterator Pattern Over a Channel 🟢

One of the cleanest concurrency patterns in Rust: iterate over a channel until all senders are gone.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<u64>();

    // Spawn workers that send results
    let mut handles = vec![];
    for id in 0..4 {
        let tx = tx.clone();
        handles.push(thread::spawn(move || {
            for i in 0..10 {
                tx.send(id * 100 + i).unwrap();
            }
            // tx drops here when thread ends
        }));
    }

    // Drop the original sender — otherwise rx.iter() never ends
    // (the original tx still counts as a sender)
    drop(tx);

    // Collect all results — finishes when all threads are done
    let results: Vec<u64> = rx.into_iter().collect();
    println!("Got {} results", results.len());  // 40

    for h in handles { h.join().unwrap(); }
}
```

The `drop(tx)` is important. If you don't drop the original sender, the channel never closes from the receiver's perspective, and `rx.into_iter()` blocks forever waiting for a message that won't come.

---

## Channels vs Shared State: When to Use Which 🟡

Both patterns work for concurrency. Here's how to decide:

| Situation | Prefer |
|---|---|
| Workers produce results → collector consumes | Channels |
| Multiple threads need to read current state | `Arc<RwLock<T>>` |
| Multiple threads update shared state | `Arc<Mutex<T>>` |
| Pipeline: A produces → B processes → C sinks | Channels |
| Config/cache that many threads read | `Arc<RwLock<T>>` |
| Broadcasting messages to many recipients | Channels (one channel per recipient) |

The chat server uses both: channels for message delivery (each client has an mpsc sender; the broadcaster sends to it), and `Arc<Mutex<T>>` for the client registry (who's connected, what their sender is).

---

## The Per-Client Channel Design

In the chat server each connected client will have a dedicated mpsc channel:

```
Client A                    Client B
read thread ─── tx ─→ ┐    read thread ─── tx ─→ ┐
                       ↓                           ↓
                  [broadcaster rx]
                       │
                       ├──→ client_A_tx.send(msg)
                       └──→ client_B_tx.send(msg)
                            │
                            ↓
                       write thread for each client
                       sends to TcpStream
```

Each client has:
1. A read thread that reads from the socket and sends to the broadcaster channel
2. A write thread that receives from its personal channel and writes to the socket

The broadcaster is a dedicated thread that receives from the main channel and fans out to every connected client's write channel. This design keeps concerns separated: reading, broadcasting, and writing are independent threads.

---

## Sending Complex Types (Must Be Send) 🟡

Anything you send through a channel must implement `Send`. Most types do. Types that don't:

- `Rc<T>` — not Send (use `Arc<T>`)
- `RefCell<T>` — not Send (use `Mutex<T>`)
- Raw pointers — not Send

If you try to send a non-Send type the compiler will catch it at the `thread::spawn` call or at the `tx.send()` call with an error mentioning the `Send` trait.

The chat server sends `String` messages, which is `Send`. If you want to send structured messages (for example, including metadata about who sent the message), define a struct and it will be automatically `Send` as long as all its fields are `Send`:

```rust
#[derive(Debug)]
struct ChatMessage {
    from: String,
    body: String,
    timestamp: std::time::SystemTime,
}

// All fields are Send, so ChatMessage is Send automatically
let (tx, rx) = mpsc::channel::<ChatMessage>();
```

---

## Thread Join: Don't Forget It

`thread::spawn` returns a `JoinHandle<T>`. If you drop the handle without joining, the thread keeps running (it's detached, not killed). That's usually not what you want. Always join handles or collect them for later:

```rust
let handle = thread::spawn(|| {
    // do work
    42  // return value
});

let result = handle.join().unwrap();  // blocks until thread finishes
println!("Thread returned: {result}");
```

If the thread panicked, `handle.join()` returns `Err`. Call `.unwrap()` to propagate the panic to the current thread, or handle it explicitly.

---

## Concurrency Primitives Decision Table

| Primitive | Use for | Cost |
|---|---|---|
| `Mutex<T>` | General exclusive mutable access | Lock/unlock per access |
| `RwLock<T>` | Read-heavy shared state | Read locks concurrent, write exclusive |
| `mpsc::channel` | Producer → consumer pipelines | Queue allocation, blocking recv |
| `mpsc::sync_channel` | Backpressure-aware pipelines | Same + blocks sender when full |
| `Arc::clone` | Sharing read-only data | Atomic counter increment |
| `thread::scope` | Parallel work borrowing local data | Join at scope end |

---

## Common Mistakes

**Forgetting `drop(tx)` before consuming `rx`.** If you still hold the original sender when you start iterating `rx`, the channel never closes. The loop blocks forever. Always drop extra senders when you no longer need them.

**Sending on a channel whose receiver has been dropped.** `tx.send()` returns `Err(SendError)`. If you're using `.unwrap()` in a worker thread and the receiver panics, your workers will start panicking too. Decide consciously whether a dead receiver should kill workers or be silently ignored (use `.ok()` to swallow).

**Cloning data into messages when you should be cloning the channel.** If you're doing `tx.clone()` to add another producer, that's correct and cheap. If you're trying to share a receiver across threads, that won't work — mpsc has one consumer. Use `Arc<Mutex<Receiver>>` if you need multiple consumers pulling from the same queue.

**Thread panic without joining.** If a thread panics and you never call `.join()`, you won't notice the panic. In the chat server, spawn threads in a way that you can detect and log panics from client handlers.

---

## How It Breaks

**`mpsc` channel buffer exhaustion with bounded channels.** When you use `mpsc::sync_channel(n)` with a fixed buffer size, `send()` blocks when the buffer is full. If your receiver is slow and your sender is fast, senders back up waiting for space. With unbounded `mpsc::channel()`, senders never block — but messages accumulate in memory. If a slow receiver can't keep up, the queue grows until you run out of memory. Choose bounded for backpressure, unbounded when memory growth is acceptable.

**Dropping the last sender while the receiver is blocking on `recv()`.** When all senders are dropped, the channel closes. A receiver blocked on `recv()` immediately gets back `Err(RecvError)`. This is the designed way to signal "work is done" — it's not an error condition, it's a clean shutdown signal. The `for msg in rx {}` iterator loop uses exactly this: it ends when all senders have been dropped.

**Forgetting to drop the original sender before iterating.** You clone the sender into N worker threads, then do `for msg in rx {}` to collect results. But you still hold the original `tx` in the calling scope. The channel never closes because the original sender still exists. The `for` loop blocks forever. The fix is `drop(tx)` before entering the receive loop — or use a scope that naturally drops `tx` before the loop.

**Thread panics are silent.** A thread spawned with `thread::spawn` that panics does not propagate the panic to the thread that spawned it. The panic is caught by the thread runtime, the thread terminates, and execution continues elsewhere. In a chat server, a client handler thread that panics simply disappears — no log message, no error, the client just disconnects silently. You only learn about it by calling `handle.join()` and checking if it returned `Err`. Always join handles, or log panics explicitly in the thread before they propagate.
