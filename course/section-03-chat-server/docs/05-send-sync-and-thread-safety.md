# Doc 05 - Send, Sync, and Thread Safety

🔴 This is where C developers hit a wall - and then realize it's actually a net.

If you're building the chat server and you tried to share an `Rc<RefCell<Vec<TcpStream>>>` across threads, you've already seen the error. The compiler stopped you from writing a data race. This doc explains exactly what happened, why, and how the type system provides thread-safety guarantees that C can't.

---

## The Problem in C

In C, thread safety is advisory. The compiler enforces nothing. The programmer writes documentation and hopes:

```c
/* THREAD SAFETY: sensor_buf must be protected by sensor_lock */
pthread_mutex_t sensor_lock;
int sensor_buf[64];
int buf_index = 0;

void sensor_irq_handler(void) {
    /* WRONG: forgot to lock - race condition */
    sensor_buf[buf_index++] = read_sensor();
}

void process_sensors(void) {
    pthread_mutex_lock(&sensor_lock);
    for (int i = 0; i < buf_index; i++) {
        process(sensor_buf[i]);
    }
    buf_index = 0;
    pthread_mutex_unlock(&sensor_lock);
}
```

One function locked, one didn't. The compiler sees nothing wrong. The race happens in production, under load, intermittently.

In Rust, the equivalent mistake is a **compile error**. Not a warning. Not a sanitizer finding. A compile error.

---

## What `Send` and `Sync` Prove

Rust has two marker traits that encode thread-safety proofs at the type level:

| Trait | What it means |
|-------|--------------|
| `Send` | This value can be **moved** to another thread safely |
| `Sync` | A **shared reference** to this value can exist on multiple threads simultaneously |

They're called **auto-traits** because the compiler derives them automatically by inspecting every field:

- A struct is `Send` if all its fields are `Send`
- A struct is `Sync` if all its fields are `Sync`
- If any field is `!Send` or `!Sync`, the whole struct becomes `!Send` or `!Sync`

This is structural. You don't annotate it. Adding a single `Rc<T>` field to a struct makes the whole struct `!Send`. There's no way to forget.

---

## Why `Rc<T>` is `!Send`

`Rc<T>` is a reference-counted pointer. The reference count is a plain `usize` - **not atomic**. If two threads both try to clone or drop an `Rc` simultaneously, the non-atomic increment/decrement creates a data race.

The compiler knows this because `Rc`'s source code explicitly marks it:

```rust
// From the standard library (conceptually):
impl<T> !Send for Rc<T> {}
impl<T> !Sync for Rc<T> {}
```

So when you try to move an `Rc` into a thread:

```rust
use std::rc::Rc;
use std::cell::RefCell;

let clients = Rc::new(RefCell::new(vec![]));

std::thread::spawn(move || {  // COMPILE ERROR
    // error[E0277]: `Rc<RefCell<Vec<...>>>` cannot be sent between threads safely
    println!("{}", clients.borrow().len());
});
```

The compiler traces the error: "`Rc<RefCell<Vec<TcpStream>>>` doesn't implement `Send`, because `Rc` doesn't implement `Send`." It tells you exactly what's wrong.

---

## The Solution: `Arc<T>`

`Arc<T>` is the thread-safe alternative. It uses **atomic** reference counting, which is correct under concurrent access:

```rust
use std::sync::Arc;

let clients = Arc::new(vec![]);         // Arc is Send
let clients_clone = Arc::clone(&clients);

std::thread::spawn(move || {            // Works - Arc is Send
    println!("{}", clients_clone.len());
});
```

But `Arc<Vec<TcpStream>>` alone is not enough for mutable shared state - multiple threads can't all call `.push()` at the same time. You need `Arc<Mutex<T>>`:

```rust
use std::sync::{Arc, Mutex};
use std::net::TcpStream;

// This is the pattern for shared mutable state across threads:
let clients: Arc<Mutex<Vec<TcpStream>>> = Arc::new(Mutex::new(Vec::new()));

let clients_for_thread = Arc::clone(&clients);
std::thread::spawn(move || {
    let mut guard = clients_for_thread.lock().unwrap();  // MUST lock before access
    // guard: MutexGuard<Vec<TcpStream>>
    // The guard gives you a &mut Vec<TcpStream>
    guard.push(new_client_connection());
    // Guard drops here, mutex unlocks automatically
});
```

The key insight: `Mutex<T>` makes it **structurally impossible** to access the inner `T` without holding the lock. You call `.lock()`, you get a `MutexGuard`. The `MutexGuard` implements `DerefMut` to `&mut T`. When the guard drops (at the end of the scope), the lock releases. There's no way to forget to unlock - the lock releases automatically when the guard goes out of scope.

---

## The Full Type Table

| Type | Send? | Sync? | Why |
|------|:-----:|:-----:|-----|
| `i32`, `u64`, `bool`, `f64` | ✅ | ✅ | Primitives - no pointers, no shared state |
| `String`, `Vec<T>` (if T: Send) | ✅ | ✅ | Owned, heap-allocated, no interior mutability |
| `Arc<T>` (if T: Send + Sync) | ✅ | ✅ | Atomic reference counting |
| `Mutex<T>` (if T: Send) | ✅ | ✅ | Serializes all access |
| `RwLock<T>` (if T: Send + Sync) | ✅ | ✅ | Multiple readers OR one writer |
| `Cell<T>` | ✅ | ❌ | Can mutate through shared ref, but not atomic |
| `RefCell<T>` | ✅ | ❌ | Runtime borrow checking, not atomic |
| `Rc<T>` | ❌ | ❌ | Non-atomic reference count |
| `*const T`, `*mut T` | ❌ | ❌ | Raw pointers - no safety guarantees |

The ❌ entries are compile-time restrictions, not runtime checks.

---

## Why `Mutex<T>` Makes `T: !Sync` Safe to Share

`RefCell<T>` is `!Sync` - you can't share a `&RefCell<T>` across threads. But `Mutex<RefCell<T>>` IS `Sync`. Why?

Because `Mutex` adds the missing synchronization. The compiler knows:
- `Mutex<T>: Sync` if `T: Send`
- To get access to the inner `T` through a `Mutex`, you must call `.lock()`
- `.lock()` blocks until no other thread holds the lock
- Therefore, only one thread can access the inner `T` at any time

The compiler verifies this structurally. You can't bypass the lock because the only way to get a reference to the inner `T` is through the `MutexGuard` returned by `.lock()`.

---

## `thread::spawn` Enforces This

The signature of `thread::spawn` is:

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T + Send + 'static,
    T: Send + 'static,
```

`F: Send + 'static` means: the closure and everything it captures must be `Send`, and must not contain any references with shorter lifetimes than `'static`. This is why:

```rust
let data = vec![1, 2, 3];
let reference = &data;           // reference borrows data

thread::spawn(move || {          // COMPILE ERROR
    println!("{}", reference[0]);
    // error: `reference` cannot be shared between threads safely
    // because `&Vec<i32>` is not `Send`
    // (more precisely: the thread might outlive 'data')
});
```

The `'static` bound catches the lifetime issue: `data` lives on the main thread's stack and might be dropped before the spawned thread finishes. The compiler refuses.

The fix: move ownership into the thread instead of borrowing:

```rust
let data = vec![1, 2, 3];

thread::spawn(move || {          // Works - data is moved, not borrowed
    println!("{}", data[0]);
});
// Note: data is gone from this scope - it was moved
```

---

## The Chat Server Pattern

For the chat server, you need to share a list of connected clients across threads. Each accepted connection spawns a handler thread, and every thread needs to broadcast to all clients. The correct pattern:

```rust
use std::sync::{Arc, Mutex};
use std::net::{TcpListener, TcpStream};
use std::io::Write;
use std::thread;

type ClientList = Arc<Mutex<Vec<TcpStream>>>;

fn broadcast(clients: &ClientList, message: &[u8]) {
    let mut guard = clients.lock().unwrap();
    // Retain only clients where write succeeds (drop disconnected ones)
    guard.retain_mut(|client| client.write_all(message).is_ok());
}

fn handle_client(stream: TcpStream, clients: ClientList) {
    use std::io::BufRead;
    let peer = stream.peer_addr().unwrap();
    let reader = std::io::BufReader::new(stream.try_clone().unwrap());

    for line in reader.lines() {
        match line {
            Ok(msg) => {
                let outgoing = format!("{peer}: {msg}\n");
                broadcast(&clients, outgoing.as_bytes());
            }
            Err(_) => break,
        }
    }

    // Client disconnected - remove from list
    let mut guard = clients.lock().unwrap();
    guard.retain(|c| c.peer_addr().map(|a| a != peer).unwrap_or(false));
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6789")?;
    let clients: ClientList = Arc::new(Mutex::new(Vec::new()));

    for stream in listener.incoming() {
        let stream = stream?;
        let clients = Arc::clone(&clients);

        // Register new client
        clients.lock().unwrap().push(stream.try_clone()?);

        thread::spawn(move || handle_client(stream, clients));
    }

    Ok(())
}
```

Notice: `Arc::clone` for the shared list, `Mutex::lock` before any access, and the lock guard drops automatically at the end of each scope. The compiler enforces all of this.

---

## Mutex Poisoning

When a thread panics while holding a Mutex, the mutex becomes "poisoned." Subsequent calls to `.lock()` return `Err(PoisonError)` to alert other threads that the data might be in an inconsistent state.

```rust
let lock_result = clients.lock();

match lock_result {
    Ok(guard) => {
        // Normal case: got the lock
        guard.push(new_stream);
    }
    Err(poison) => {
        // A thread panicked while holding this lock
        // The data might be corrupted
        // You can recover the guard with .into_inner():
        let guard = poison.into_inner();
        // Or just panic yourself:
        // panic!("Mutex poisoned: {:?}", poison);
    }
}

// In practice, most production code uses .unwrap() or .expect():
// If a thread panics while holding a lock, your server has bigger problems
let mut guard = clients.lock().unwrap();
```

For most servers, `.unwrap()` on mutex locks is fine. The alternative is complex error handling for a scenario (panic inside a lock) that indicates a programming error anyway.

---

## Manual `Send` and `Sync` Impls

Sometimes you have a type that's safe to send across threads but the compiler can't prove it (because it contains a raw pointer or `Rc`). You can tell the compiler manually:

```rust
// A type wrapping a C library handle (just an integer internally)
struct LibraryContext {
    handle: *mut std::ffi::c_void,  // raw pointer makes this !Send by default
}

// SAFETY: The C library documentation guarantees this handle is thread-safe
// when accessed through our mutex. We've verified there are no thread-local
// side effects in the C library.
unsafe impl Send for LibraryContext {}
unsafe impl Sync for LibraryContext {}
```

The `unsafe` keyword here means: **you** are making a promise the compiler can't verify. You must uphold it. If you lie, you get undefined behavior - the same as C, but explicitly opted into rather than silently fallen into.

This is rare. The standard approach is to use `Arc<Mutex<T>>` and let the compiler handle it.

---

## The "Future is not Send" Error

When you move to async programming (section 4), you'll hit this error frequently:

```
error: future cannot be sent between threads safely
  |
  = help: the trait `Send` is not implemented for `Rc<i32>`
note: future is not `Send` as this value is used across an await
```

This means: your async function holds an `Rc<T>` across an `.await` point. When a future is sent to another thread (e.g., by `tokio::spawn`), everything it holds must be `Send`. `Rc` isn't. The fix: replace `Rc` with `Arc`, or restructure so the `Rc` is dropped before the `.await`.

Understanding `Send` now makes that error instantly recognizable when you hit it.

---

## How It Breaks

**`unwrap()` on a poisoned mutex causing a second panic.**
If a mutex is poisoned and you `.unwrap()` the `lock()` result, you get a second panic. In a multi-threaded server, this can cascade - one panic poisons the mutex, every subsequent access panics. Consider handling the `Err` case with `.unwrap_or_else(|e| e.into_inner())` if you want to continue.

**Holding a `MutexGuard` across an `await` point.**
`MutexGuard` is `!Send`. If you hold a guard and then `.await` something, the future becomes `!Send` and can't be spawned on a multi-threaded runtime. Solution: structure your code so the guard is dropped before the `.await`:

```rust
// Wrong - guard held across await
async fn bad(data: Arc<Mutex<Vec<u8>>>) {
    let guard = data.lock().unwrap();
    some_async_operation().await;  // guard is still alive here - !Send
    println!("{}", guard.len());
}

// Correct - clone or extract what you need, drop the guard before await
async fn good(data: Arc<Mutex<Vec<u8>>>) {
    let len = {
        let guard = data.lock().unwrap();
        guard.len()  // guard drops at end of this block
    };
    some_async_operation().await;  // no guard alive
    println!("{}", len);
}
```

**Deadlock from double-locking the same mutex.**
Rust's `Mutex` is not reentrant. If a thread calls `.lock()` while already holding the lock, it deadlocks (blocks waiting for itself forever):

```rust
let data = Mutex::new(vec![]);
let guard1 = data.lock().unwrap();
let guard2 = data.lock().unwrap();  // DEADLOCK - waiting for guard1 to release
```

Structure your code to hold locks for the shortest possible time and never call a function that might lock the same mutex while you're holding it.
