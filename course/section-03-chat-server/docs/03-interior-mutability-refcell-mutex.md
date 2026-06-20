# Doc 03 — Interior Mutability: RefCell and Mutex

Here is a problem you'll run into almost immediately when building anything real in Rust. You have a data structure — let's say a cache, or a list of connections — that you've wrapped in an `Arc` so multiple parts of your program can share it. But `Arc<T>` only lets you get a shared `&T` reference to the inner value, and shared references are immutable in Rust.

You need to mutate. But you can't. The borrow rules say you'd need a `&mut T`, but you only have `&T`. What now?

This is the **interior mutability** problem, and it has two solutions depending on whether you're in a single-threaded or multi-threaded context.

---

## The Core Problem

Rust normally enforces borrow rules at compile time: either one mutable reference, or any number of immutable references. These rules exist to prevent data races and ensure memory safety.

But sometimes your code structure makes it impossible for the compiler to prove correctness at compile time — even when you know your logic is sound. Interior mutability moves the borrow checking to **runtime**, and if you violate the rules at runtime, the program panics rather than producing undefined behavior.

Think of it this way: instead of the compiler checking "is there a mutable borrow right now?", the data structure itself checks at runtime. If the check fails, you get a clean panic rather than memory corruption. That's a much better failure mode than C.

---

## Cell\<T\>: Simple Interior Mutability for Copy Types 🟢

`Cell<T>` is the simplest interior mutability tool. It works by copying values in and out — you `get()` a copy of the value and `set()` a new value. No references involved, no borrow checking needed, zero runtime overhead.

The catch: `T` must be a `Copy` type (integers, bools, floats, etc.).

```rust
use std::cell::Cell;

struct Config {
    max_connections: u32,     // regular field
    active_count: Cell<u32>,  // interior mutable field
}

impl Config {
    fn new(max: u32) -> Self {
        Config { max_connections: max, active_count: Cell::new(0) }
    }

    // Note: &self not &mut self — this is the whole point
    fn connect(&self) {
        let current = self.active_count.get();
        self.active_count.set(current + 1);
    }

    fn count(&self) -> u32 {
        self.active_count.get()
    }
}

fn main() {
    let config = Config::new(100);
    config.connect();
    config.connect();
    println!("Active: {}", config.count());  // 2
}
```

`Cell` is useful when you want most of a struct to be read-only but one field needs to track state (a counter, a flag, a cache hit count). You don't need `mut` on the variable holding the struct to use `Cell`.

---

## RefCell\<T\>: Runtime Borrow Checking 🟡

`RefCell<T>` is the more powerful version. It works with any type (not just `Copy`) by lending references — but it enforces the borrow rules at **runtime**.

- `borrow()` returns a `Ref<T>` — an immutable borrow. Multiple can coexist.
- `borrow_mut()` returns a `RefMut<T>` — a mutable borrow. This one is exclusive.

If you call `borrow_mut()` while a `Ref` or `RefMut` is still alive, the program **panics**.

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);

    {
        let v = data.borrow();           // immutable borrow
        println!("length: {}", v.len()); // works fine
        // v drops here, borrow ends
    }

    {
        let mut v = data.borrow_mut();   // mutable borrow
        v.push(4);
        println!("after push: {:?}", *v);
        // v drops here, borrow ends
    }

    // This would panic at runtime:
    // let _a = data.borrow();
    // let _b = data.borrow_mut();  // PANIC: already borrowed as immutable
}
```

The `Ref<T>` and `RefMut<T>` returned by `borrow()` and `borrow_mut()` work just like regular references. `RefMut<T>` implements `DerefMut` so you can call methods on the inner value just like you would with `&mut T`.

### When to Use RefCell

Use `RefCell` when:
- You're in a single-threaded context
- You need to mutate data through a shared reference (`&T`)
- You're confident your logic guarantees the borrow rules won't be violated at runtime
- The borrow checker can't prove it statically (graph traversal, callback patterns, caches)

The classic pattern is `Rc<RefCell<T>>` — shared ownership plus interior mutability, all in one thread.

---

## Mutex\<T\>: Thread-Safe Interior Mutability 🟡

For multi-threaded code, `RefCell` doesn't work — it's not `Sync` (can't be shared between threads). The thread-safe equivalent is `Mutex<T>`.

`Mutex` works like a lock. When a thread wants to access the inner data, it calls `.lock()`. If another thread already holds the lock, `.lock()` blocks until that thread releases it. When you get the lock, you get a `MutexGuard<T>` that derefs to the inner data. When the guard drops, the lock is released.

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(0_i32);

    {
        let mut guard = m.lock().unwrap();
        *guard = 42;
        println!("set to {guard}");
        // guard drops here, lock released
    }

    println!("value is now: {}", m.lock().unwrap());
}
```

This is RAII, exactly like `std::lock_guard<std::mutex>` in C++. The lock is released when the guard goes out of scope. You cannot forget to unlock — it's automatic.

### The MutexGuard

`MutexGuard<T>` implements `Deref` and `DerefMut`, so you access the inner value directly through it. You don't need to unwrap it or call any extra method:

```rust
let mut guard = mutex.lock().unwrap();
*guard += 1;           // dereference to modify
println!("{}", *guard); // dereference to read
// guard.method()        // method calls work via Deref coercion
```

---

## Mutex Poisoning 🔴

In C, if a thread dies while holding a mutex, the mutex is just... stuck. Other threads block forever.

In Rust, `Mutex` detects this. If a thread panics while holding the lock, the mutex becomes **poisoned**. Subsequent calls to `.lock()` return `Err(PoisonError)` instead of `Ok(guard)`. This is why we call `.unwrap()` on the result of `.lock()` — if the mutex is poisoned, `.unwrap()` panics, which is usually the right behavior.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));
    let data2 = Arc::clone(&data);

    let handle = thread::spawn(move || {
        let mut guard = data2.lock().unwrap();
        guard.push(4);
        panic!("thread panicked while holding lock");  // lock is now poisoned
    });

    let _ = handle.join();  // thread panicked, but main keeps going

    match data.lock() {
        Ok(guard) => println!("Data: {:?}", *guard),
        Err(poisoned) => {
            // The data is still there, the push(4) succeeded before the panic
            let guard = poisoned.into_inner();
            println!("Recovered (was poisoned): {:?}", *guard);  // [1, 2, 3, 4]
        }
    }
}
```

For the chat server, you'll mostly use `.unwrap()` on locks. If a thread panics while holding the client list mutex, we want the whole server to crash loudly rather than silently continue in a corrupted state.

---

## RwLock\<T\>: Many Readers OR One Writer 🟡

`RwLock<T>` is a refinement of `Mutex`. It has two lock modes:
- `read()` — multiple threads can hold read locks simultaneously
- `write()` — only one thread can hold a write lock, and no readers can hold locks

This is the classic readers-writer lock, equivalent to `std::shared_mutex` in C++.

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let config = Arc::new(RwLock::new(String::from("v1.0")));
    let mut handles = vec![];

    // 5 readers — all run concurrently
    for i in 0..5 {
        let config = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            let val = config.read().unwrap();
            println!("Reader {i}: {}", *val);
        }));
    }

    // 1 writer — waits for all readers, then gets exclusive access
    let config = Arc::clone(&config);
    handles.push(thread::spawn(move || {
        let mut val = config.write().unwrap();
        *val = String::from("v2.0");
    }));

    for h in handles { h.join().unwrap(); }
}
```

Use `RwLock` when you have a read-heavy workload — lots of threads reading configuration or a shared cache, with infrequent writes. Use `Mutex` when writes are as common as reads or when the critical section is very short (the overhead of tracking read/write state isn't worth it).

---

## The Full Pattern: Arc\<Mutex\<T\>\>

In the chat server, you'll share the list of clients across all threads. Here it is in full:

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::sync::mpsc::Sender;

// This type gets cloned into every thread that handles a client
type ClientMap = Arc<Mutex<HashMap<String, Sender<String>>>>;

fn main() {
    let clients: ClientMap = Arc::new(Mutex::new(HashMap::new()));

    // When a client connects, add them:
    {
        let mut map = clients.lock().unwrap();
        // map is a MutexGuard<HashMap<...>>
        // we can use it like a regular &mut HashMap
        // map.insert("alice".to_string(), some_sender);
    }  // lock released here

    // When broadcasting a message:
    {
        let map = clients.lock().unwrap();
        for (username, sender) in map.iter() {
            let _ = sender.send(format!("broadcast: hello {username}"));
        }
    }  // lock released here
}
```

The key insight: you want to hold the lock for as short a time as possible. Lock, do the minimum amount of work, drop the guard. This reduces contention and prevents deadlocks.

---

## Deadlock: The Danger Zone 🔴

Deadlock happens when thread A holds lock 1 and waits for lock 2, while thread B holds lock 2 and waits for lock 1. Both wait forever.

Rust does not prevent deadlocks. The type system prevents data races, but deadlock is a logic error.

Rules to avoid deadlock in the chat server:

1. **Always acquire locks in the same order.** If you ever need two locks at once, every thread must lock them in the same sequence.
2. **Never hold a lock across an I/O call.** Don't hold the client map lock while writing to a socket. Lock, copy what you need, release the lock, then do the I/O.
3. **Keep critical sections short.** The longer you hold a lock, the more contention you create.

```rust
// BAD: holding the lock while doing I/O
{
    let map = clients.lock().unwrap();
    for (_, sender) in map.iter() {
        expensive_network_call();  // lock held during I/O — bad
    }
}

// BETTER: collect what you need, release the lock, then do I/O
let senders: Vec<Sender<String>> = {
    let map = clients.lock().unwrap();
    map.values().cloned().collect()
};  // lock released here
for sender in senders {
    sender.send(msg.clone()).ok();  // no lock held during send
}
```

---

## Quick Reference

| Type | Thread-safe | Failure mode | Use when |
|---|---|---|---|
| `Cell<T>` | No | Never fails | Single thread, `Copy` types |
| `RefCell<T>` | No | Runtime panic | Single thread, any type |
| `Mutex<T>` | Yes | Blocks (+ poison) | Multi-thread, general state |
| `RwLock<T>` | Yes | Blocks (+ poison) | Multi-thread, read-heavy |

---

## Common Mistakes

**Calling `borrow_mut()` while a `borrow()` is still alive.** `RefCell` panics at runtime for this. The fix is to ensure the immutable borrow is dropped before you take a mutable one. Use scopes `{}` to force the drop.

**Holding a `MutexGuard` in a variable that lives too long.** If you write `let guard = mutex.lock().unwrap();` and then do a bunch of work, the lock is held for all of it. Be intentional about when guards drop.

**Calling `.lock().unwrap()` twice in the same thread without dropping the first guard.** This will deadlock on most mutex implementations because a single thread tries to acquire a lock it already holds.

**Using `RefCell` in multi-threaded code.** It doesn't implement `Sync`. The compiler will catch this, but the error message can be confusing if you don't know what to look for.

---

## How It Breaks

**`RefCell` runtime panic.** `borrow()` and `borrow_mut()` enforce the borrow rules at runtime rather than compile time. If you call `borrow_mut()` while any `Ref` or `RefMut` is still alive — even if you're sure the logic is correct — the program panics. This is worse than a compile-time error because it crashes in production, not during development. The fix is to ensure all borrows are dropped before taking a new conflicting borrow, using explicit scopes `{}` to force early drops.

**`std::sync::Mutex` poisoning.** If a thread panics while holding a `Mutex` lock, the mutex becomes poisoned. Every subsequent call to `.lock()` on that mutex returns `Err(PoisonError)`. If your code uses `.unwrap()` everywhere (reasonable during development), a poisoned mutex causes every thread that accesses it to panic. For the chat server, this is usually the right behavior — a corrupted client list should crash loudly rather than silently continue. For library code or long-running servers, handle `PoisonError` explicitly with `lock().unwrap_or_else(|e| e.into_inner())`.

**Holding a `MutexGuard` across `.await` in async code.** The guard from `std::sync::Mutex` is not `Send` — it cannot be moved between threads. In a multi-threaded async runtime (like Tokio's default), a task can be moved to a different thread at any `.await` point. If your task holds a `std::sync::MutexGuard` when it yields, the compiler will refuse to compile with a confusing error about `Send`. The fix: either drop the guard before any `.await` (restructure so the lock is held in a `{}` block that ends before the `.await`), or switch to `tokio::sync::Mutex` which is designed to be held across await points.

**`RwLock` writer starvation.** On some platforms and implementations, if reader threads continuously hold the read lock, writers can be blocked indefinitely — every time the last reader releases, a new reader grabs it before the writer can get in. Whether this happens depends on the platform's fairness policy. If you have a write-heavy workload and see that writes are stalling, `Mutex` (which doesn't distinguish readers from writers) avoids this entirely.
