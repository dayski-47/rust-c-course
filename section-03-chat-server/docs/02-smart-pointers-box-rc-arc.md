# Doc 02 — Smart Pointers: Box, Rc, Arc

Here's a question you've probably already hit: what do you do when you want to share data between two parts of your program, but the ownership model says only one place can own something at a time?

You're about to meet three answers to that question: `Box<T>`, `Rc<T>`, and `Arc<T>`. By the end of this chapter you'll know which one to reach for and — most importantly — you'll understand the `Arc<Mutex<T>>` pattern that makes the chat server work.

---

## Box\<T\>: Heap Allocation 🟢

In C, `malloc` puts data on the heap and you call `free` when you're done. In C++, `std::unique_ptr<T>` wraps that pattern so the destructor handles the free. In Rust, `Box<T>` is the equivalent: it allocates a single value on the heap and frees it automatically when the `Box` goes out of scope.

```rust
fn main() {
    let b = Box::new(42_i32);   // 42 lives on the heap
    println!("{}", *b);          // deref to get the i32
    println!("{}", b);           // Deref coercion works here too
    // b drops here, heap memory freed automatically
}
```

```
Stack                  Heap
┌────────────┐        ┌────────────┐
│  b: Box    │──────▶ │     42     │
│  (pointer) │        └────────────┘
└────────────┘
```

When do you actually need `Box`?

**Recursive types.** A linked list node that contains itself can't be a fixed size on the stack — the type would be infinitely large. Box solves this because a pointer is always the same size regardless of what it points to.

```rust
enum List {
    Cons(i32, Box<List>),  // Box makes this a fixed-size pointer
    Nil,
}

fn main() {
    let list = List::Cons(1,
        Box::new(List::Cons(2,
            Box::new(List::Cons(3,
                Box::new(List::Nil))))));
}
```

**Large values.** If you have a 10 MB array and need to pass it between functions without copying it every time, heap-allocate it.

**Trait objects.** When you want to store different types implementing the same trait in a collection: `Vec<Box<dyn Draw>>`. This is covered more in the traits section.

---

## Rc\<T\>: Reference Counting (Single-Threaded) 🟡

Here's the ownership problem Box doesn't solve. You have an `Employee` record that you want to store in two different collections — say, a department list and a global registry. With normal ownership, once you push the employee into the first collection, it's moved. You can't also push it into the second.

`Rc<T>` (Reference Counted) solves this with shared ownership. Multiple `Rc` handles can point to the same data. The data is dropped when the last `Rc` is dropped — when the reference count hits zero.

```rust
use std::rc::Rc;

#[derive(Debug)]
struct Employee {
    id: u64,
    name: String,
}

fn main() {
    let emp = Rc::new(Employee { id: 42, name: "Alice".to_string() });

    let mut department = vec![];
    let mut global = vec![];

    department.push(Rc::clone(&emp));  // bumps reference count to 2
    global.push(Rc::clone(&emp));      // bumps reference count to 3

    println!("Reference count: {}", Rc::strong_count(&emp));  // 3
    println!("From department: {:?}", department[0]);
    println!("From global: {:?}", global[0]);

    // When department and global go out of scope, count drops to 1.
    // When emp goes out of scope, count hits 0 and Employee is freed.
}
```

`Rc::clone(&emp)` does NOT copy the `Employee`. It just increments a counter and gives you another pointer to the same heap data. This is cheap — it's just an integer increment.

Critical limitation: **Rc cannot be sent to another thread.** The reference count is not atomic — if two threads incremented it simultaneously, the count would corrupt. The compiler enforces this: `Rc<T>` does not implement the `Send` trait, so trying to move it into `thread::spawn` is a compile error. For threads, you need `Arc`.

---

## Arc\<T\>: Atomic Reference Counting (Multi-Threaded) 🟡

`Arc<T>` (Atomically Reference Counted) is exactly like `Rc<T>` except it uses atomic operations for the reference count. Atomic operations are thread-safe — they can't be interrupted mid-increment by another thread. This makes `Arc` slightly more expensive than `Rc` (an atomic increment costs more than a plain increment), but it's the right tool whenever threads are involved.

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3, 4, 5]);
    let mut handles = vec![];

    for i in 0..3 {
        let data = Arc::clone(&data);  // cheap: just atomic increment
        handles.push(thread::spawn(move || {
            println!("Thread {i} sees: {:?}", data);
        }));
    }

    for h in handles {
        h.join().unwrap();
    }
}
```

All three threads share the same `Vec` without copying it. The `Arc::clone` is just "bump the reference count and give me another handle to the same allocation."

But wait — `Arc<T>` alone only lets you *read* the shared data. You can't mutate it through an `Arc` because that would allow multiple threads to write simultaneously, which is a data race. To also mutate, you pair `Arc` with `Mutex`.

---

## The Arc\<Mutex\<T\>\> Pattern 🔴

This is the bread and butter of shared mutable state in multi-threaded Rust. You'll use it constantly in the chat server.

`Arc` provides shared ownership across threads. `Mutex` provides exclusive mutable access — only one thread can hold the lock at a time.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // Arc wraps Mutex wraps the actual data
    let counter = Arc::new(Mutex::new(0_u64));
    let mut handles = vec![];

    for _ in 0..5 {
        let counter = Arc::clone(&counter);  // clone before moving into thread
        handles.push(thread::spawn(move || {
            let mut guard = counter.lock().unwrap();  // blocks until lock is free
            *guard += 1;
            // guard drops here, lock is released automatically
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    println!("Final: {}", counter.lock().unwrap());  // 5
}
```

Think of `Mutex` like a ticket system. The data is inside the booth. Only one person can enter at a time. When you call `.lock()`, you wait your turn. When you get the `MutexGuard` back, you're inside the booth — exclusive access to the data. When the guard drops (goes out of scope), you leave the booth and the next thread can enter. This is RAII — exactly like `std::lock_guard` in C++.

In the chat server, this pattern will hold the list of connected clients:

```
Arc<Mutex<HashMap<String, Sender<String>>>>
 ^      ^       ^
 |      |       └── the actual data: username → message channel
 |      └── only one thread can access the map at a time
 └── shared across all threads that handle clients
```

---

## The Deref Trait

One thing that makes `Box`, `Rc`, and `Arc` feel seamless is the `Deref` trait. When you do `*arc` you're calling `Deref::deref()` which gives you a reference to the inner value. And because of **Deref coercion**, Rust automatically inserts deref calls when you call methods on the inner type:

```rust
let arc = Arc::new(String::from("hello"));
println!("{}", arc.len());  // arc auto-derefs to String, then calls .len()
// equivalent to: (*arc).len()
```

You rarely need to write `*` manually. Rust inserts it for you.

---

## Weak\<T\>: Breaking Reference Cycles 🔴

`Rc` and `Arc` count references. That means if two values hold `Rc` references to each other, neither will ever reach a count of zero — they'll leak forever. This is a **reference cycle**.

```rust
// This would leak if parent and child both held strong Rc refs to each other
// A -> B and B -> A: count never reaches 0, memory leaks
```

`Weak<T>` is a non-owning reference. It does NOT increment the strong count. To use a `Weak` reference you have to try to upgrade it to an `Rc` — which returns `Option<Rc<T>>` since the data might already have been freed.

```rust
use std::rc::{Rc, Weak};

struct Node {
    value: i32,
    parent: Option<Weak<Node>>,  // doesn't count toward Rc strong count
}

let parent = Rc::new(Node { value: 1, parent: None });
let child = Rc::new(Node {
    value: 2,
    parent: Some(Rc::downgrade(&parent)),  // Weak reference, not strong
});

// To use the parent from child:
if let Some(p) = child.parent.as_ref().unwrap().upgrade() {
    println!("Parent: {}", p.value);
}
// If parent were already dropped, upgrade() returns None
```

Use `Weak` for back-pointers in trees and graphs, or for caches where it's okay if the cached value is already gone.

---

## Decision Table

| You need... | Use |
|---|---|
| Heap allocation, single owner | `Box<T>` |
| Shared ownership, one thread | `Rc<T>` |
| Shared ownership, multiple threads | `Arc<T>` |
| Shared mutable state, one thread | `Rc<RefCell<T>>` (next chapter) |
| Shared mutable state, multiple threads | `Arc<Mutex<T>>` |
| Back-pointer that shouldn't prevent drop | `Weak<T>` |

---

## Common Mistakes

**Using `Rc` across threads.** The compiler will stop you with a `Send` trait error. If you get "cannot be sent between threads safely" and you're using `Rc`, switch to `Arc`.

**Calling `.clone()` on an `Arc` and thinking you're copying the data.** You're not. `Arc::clone(&x)` increments a counter and gives you another handle to the same allocation. This is intentional and cheap. If you actually want a copy of the data, call `(*arc).clone()` or `arc.as_ref().clone()` to clone the inner value.

**Holding a `MutexGuard` across an `await` point.** This isn't relevant until async, but good to know: holding a lock while waiting for I/O is a common deadlock source. Drop the guard before any blocking call.

**Creating `Rc` cycles without `Weak`.** If you build a tree where children hold strong `Rc` references to their parents and parents hold strong `Rc` references to their children, nothing will ever be freed. Use `Weak` for "up" references (child to parent) and `Rc` for "down" references (parent to child).

---

## How It Breaks

**`Rc<T>` used across a thread boundary.** The compiler catches this with a hard error: `Rc<T>` does not implement `Send`. The fix is always `Arc<T>`. If you see "cannot be sent between threads safely" and you're using `Rc`, that's your answer.

**Arc\<Mutex\<T\>\> deadlock.** Thread A holds the lock on mutex 1 and tries to acquire mutex 2. Thread B holds the lock on mutex 2 and tries to acquire mutex 1. Both wait forever. Rust cannot detect this at compile time — it's a logic error. The prevention rule: always acquire multiple locks in the same order across all code paths.

**Reference cycles with `Rc`.** A points to B with a strong `Rc`, and B points back to A with a strong `Rc`. The reference count for each never reaches zero. The memory leaks for the lifetime of the program. The heap grows. No crash — just quiet resource exhaustion. Use `Weak` to break the cycle for back-pointers.

**`Arc::clone()` is cheap, but cloning the data inside the Arc is not.** `Arc::clone(&x)` is just an atomic counter increment — it does not copy the contained data. If you write `(*arc).clone()` or call `.to_owned()` on the contents on every access, you've turned a cheap pointer operation into a full data copy. Only clone the `Arc`, not the thing inside it, unless you specifically need an independent copy of the data.

**`Rc`/`Arc` doesn't make the `T` inside thread-safe.** `Arc<Vec<u8>>` gives you a shared pointer to a Vec, but you cannot call `.push()` on it from multiple threads — you don't have `&mut Vec<u8>`, and getting one from a shared reference is impossible without interior mutability. The `Arc` only makes the *pointer* (reference count) thread-safe. The data still needs `Mutex` or `RwLock` to mutate safely from multiple threads.
