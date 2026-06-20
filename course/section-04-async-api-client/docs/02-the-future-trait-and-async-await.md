# 02 - The Future Trait and async/await 🟡

In the last doc, we talked about *why* async is useful. Now we need to understand *how* it works. The good news: the entire async system in Rust is built on one trait. Once you understand that trait, everything else makes sense.

---

## The Future Trait

Everything in async Rust implements this:

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),    // The future has completed, here is the value.
    Pending,     // Not done yet. I'll notify you when there's progress.
}
```

A `Future` is anything that can be asked "are you done yet?" and answers either "yes, here's the result" (`Ready`) or "not yet" (`Pending`).

That's the whole thing. The complexity in async Rust doesn't come from the trait itself - it comes from *what the compiler does with it* and *how executors drive it*.

---

## async fn: Syntactic Sugar for State Machines

When you write this:

```rust
async fn load_user(id: u64) -> String {
    let data = fetch_from_db(id).await;
    let profile = parse_profile(data).await;
    format!("User: {}", profile.name)
}
```

The compiler does NOT write a function that runs top to bottom. Instead, it generates a **state machine** - a struct with an enum inside it that tracks where the function currently is:

```
State 0: Just started. About to call fetch_from_db().
State 1: Waiting for fetch_from_db() to complete.
         (Storing: id, and the in-progress fetch future)
State 2: Waiting for parse_profile() to complete.
         (Storing: the data string, and the in-progress parse future)
State 3: Done. Returning the formatted string.
```

Each time the executor calls `poll()` on this future, it checks what state it's in, tries to make progress, and either advances to the next state or returns `Pending` if it's still waiting.

The storage between states is what makes async functions *interesting* from an ownership perspective. The compiler has to keep all local variables alive across `.await` points, which is why async state machines can sometimes be large structs.

---

## .await: Yield Until Ready

The `.await` syntax does two things:

1. It polls the given future.
2. If the future is `Pending`, it yields control back to the executor and suspends the current task until the future is ready.

```rust
// This is NOT sequential blocking. This is "try to make progress, yield if not ready."
let response = reqwest::get("https://api.github.com/users/torvalds").await?;
```

When the executor comes back to this task (because the HTTP response arrived), it resumes execution from right after the `.await`, with `response` now bound to the completed value.

**Crucially: you cannot use `.await` outside of an async function.** The `.await` syntax only works inside `async fn` or `async` blocks.

---

## Futures Are Lazy

This is the single most important thing to understand and the most common source of confusion:

**Calling an async function does not execute it. It just creates a future.**

```rust
// This does NOTHING except allocate a Future struct on the stack:
let future = fetch_from_db(42);

// Nothing has happened. No network call. No allocation beyond the struct.
// You could drop `future` here and no work would ever occur.
drop(future); // Work silently cancelled, never started.

// To actually RUN the future, you must .await it:
let result = fetch_from_db(42).await;  // NOW it executes.

// OR pass it to tokio::spawn to run it as a background task:
tokio::spawn(fetch_from_db(42));  // Runs independently.
```

Compare this to how you might expect it to work if you're coming from JavaScript, where `async` functions start executing immediately up to the first `await`. In Rust, nothing happens until someone drives the future by calling `poll()`.

This lazy design has a nice property: creating a future is always cheap. You can build up complex combinations of futures before starting any of them.

---

## async Blocks

You can create a future inline without writing a whole function:

```rust
// An async block is just a future that captures its environment
let fetcher = async {
    let url = "https://api.github.com/repos/torvalds/linux";
    let response = reqwest::get(url).await?;
    response.json::<serde_json::Value>().await
};

// Nothing has run yet. Start it now:
let result = fetcher.await;
```

Async blocks are useful when you want to pass a future to `tokio::spawn` or `tokio::join!` without defining a whole function.

---

## Running Multiple Futures Concurrently

One of the most common operations in async code: start several futures at the same time and wait for all of them.

### tokio::join! - Wait for All

```rust
use tokio::time::{sleep, Duration};

async fn fetch_user(id: u64) -> String {
    sleep(Duration::from_millis(100)).await;
    format!("user_{}", id)
}

async fn fetch_repos(username: &str) -> Vec<String> {
    sleep(Duration::from_millis(150)).await;
    vec!["repo1".into(), "repo2".into()]
}

#[tokio::main]
async fn main() {
    // WRONG: These run sequentially. Takes 250ms total.
    let user = fetch_user(42).await;        // 100ms
    let repos = fetch_repos("alice").await; // 150ms

    // CORRECT: These run concurrently. Takes ~150ms total.
    let (user, repos) = tokio::join!(
        fetch_user(42),
        fetch_repos("alice"),
    );

    println!("{}: {:?}", user, repos);
}
```

`tokio::join!` polls all given futures concurrently and returns when *all* of them have completed. The results come back as a tuple in the same order as the arguments.

### tokio::select! - Use the First

`select!` is different: it races multiple futures against each other and returns as soon as *any one* of them completes, cancelling the rest.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        result = fetch_from_primary() => {
            println!("Primary responded: {}", result);
        }
        result = fetch_from_backup() => {
            println!("Backup responded first: {}", result);
        }
        _ = sleep(Duration::from_secs(5)) => {
            println!("Both timed out!");
        }
    }
}
```

`select!` is great for timeouts, cancellation signals, and "use whichever source responds first" patterns.

---

## The Poll Enum in Practice

You don't normally write code that manually calls `poll()`. The `.await` keyword does that for you. But understanding `Poll` helps you reason about what's happening:

```rust
// Poll::Ready means the future is done. The value is available.
Poll::Ready("hello".to_string())

// Poll::Pending means come back later. The future has registered a
// "waker" that will notify the executor when it should try again.
Poll::Pending
```

The **waker** is a callback. When a future returns `Pending`, it promises to call the waker when it's ready to make progress. The executor uses this to know when to poll again - rather than spinning in a busy loop checking every microsecond.

---

## A Brief Word on Pin

You may encounter `Pin<&mut T>` in Future-related code and compiler errors. Here's the minimum you need to know:

When an async function has local variables that hold references to other local variables, the generated state machine becomes **self-referential** - a field in the struct might contain a pointer to another field in the same struct. If you move that struct in memory, the pointer breaks.

`Pin<T>` is a wrapper that tells the compiler: "this value will not be moved after it's pinned." This ensures those self-referential pointers stay valid.

In practice:
- You will almost never implement `Future` by hand, so you rarely deal with `Pin` directly.
- When the compiler complains about a `Future` needing to be pinned, the fix is usually `Box::pin(some_future)` which allocates the future on the heap and pins it there.
- If you see `Pin<Box<dyn Future<Output = T>>>` in a trait or error message, that's just "a heap-allocated pinned future of this output type."

---

## What the Compiler Really Does

Let's look at a real example and what the compiler turns it into conceptually:

```rust
async fn greet(name: &str) -> String {
    let greeting = fetch_greeting().await;   // First await
    let suffix = fetch_suffix().await;        // Second await
    format!("{}, {}! {}", greeting, name, suffix)
}
```

The compiler generates something roughly equivalent to:

```rust
enum GreetState<'a> {
    // Before the first .await
    Start { name: &'a str },
    // Waiting for fetch_greeting()
    WaitingForGreeting {
        name: &'a str,
        fut: FetchGreetingFuture,
    },
    // Waiting for fetch_suffix(), holding the completed greeting
    WaitingForSuffix {
        name: &'a str,
        greeting: String,
        fut: FetchSuffixFuture,
    },
    // All done
    Done,
}

// The Future impl transitions between these states on each poll().
```

Each state stores everything needed to resume. This is why async Rust has different memory behavior than threaded code: the futures are explicit data structures, not implicit thread stacks.

---

## Common Mistakes

**Forgetting to .await.** This is by far the most common mistake when learning async Rust. If you call an async function and don't `.await` it, nothing happens and the compiler may not warn you loudly enough.

```rust
// WRONG: fetch_data() is called but the returned Future is immediately dropped.
// No request is ever made.
fetch_data("https://api.github.com");

// CORRECT:
fetch_data("https://api.github.com").await;
```

**Using sequential awaits when you want concurrency.** As shown above, `let a = foo().await; let b = bar().await;` is sequential - `bar` doesn't start until `foo` finishes. Use `tokio::join!` when you want both running at the same time.

**Trying to return a Future from a non-async function without boxing it.** If a non-async function needs to return a future, the return type gets complicated because async functions generate anonymous types. The usual fix is `-> impl Future<Output = T>` or `-> Pin<Box<dyn Future<Output = T>>>` (the boxed version works across trait objects).

**Holding large values across .await.** Everything a future needs to be resumed is stored in the state machine struct. If you hold a large buffer or complex data structure across a `.await`, it lives in that struct for the duration of the wait. Be thoughtful about what you're storing.

---

## How It Breaks

**A Future that never returns `Poll::Ready` hangs forever.** If your async code awaits something that never resolves - a channel that no one ever sends to, a network call to a server that never responds and has no timeout, a `Mutex` that no one ever releases - the task hangs silently. No error, no panic, no log message. The task is just stuck in `Pending` indefinitely. Use timeouts (`tokio::time::timeout`) around all operations that could hang, and ensure every channel has a path to being closed.

**Forgetting to `.await` a future.** You write `tokio::time::sleep(Duration::from_secs(1))` without `.await`. The future is created as a value but never polled. No sleep happens. No error is raised. The future is silently dropped and the function proceeds immediately. This is the single most common async mistake. The compiler sometimes warns about "unused `impl Future`" but not always, especially when the future is returned from a function that ignores the return value. Always `.await` async calls.

**Dropping a future cancels it.** If you drop a `JoinHandle` without awaiting it, the underlying task keeps running (it's detached). But if you drop a future that's not inside a spawn - for example, you're using `select!` or you manually drop it - whatever work it was doing is cancelled at the current yield point. If the future was in the middle of writing data to a database or a socket, that write is abandoned. The partial write may or may not be visible on the other end, depending on the protocol. Design for cancellation: do work that must not be interrupted in a `spawn_blocking` call, or accept that cancelled futures may leave partial state.

**`select!` cancels the losing branch.** When using `tokio::select!`, as soon as one branch completes, all the other branches are dropped. If a losing branch was in the middle of an operation - writing to a database, sending a network request, holding a lock - it is cancelled at its current `.await` point. This is a common source of data loss and corruption when developers use `select!` without understanding cancellation. If a branch must complete even if it loses the race, run it with `tokio::spawn` instead of inside `select!`.

---

## How the Executor Actually Runs Your Future

The runtime loop is simpler than it looks. Here's what Tokio does at its core:

```
1. Call poll() on the future
2. If Poll::Ready(value) → done, return value
3. If Poll::Pending → park the task until wake() is called
4. When wake() fires → go back to step 1
```

The future itself is responsible for calling `wake()` when it's ready to make progress. That's what I/O drivers do: they register with the OS (via epoll on Linux, kqueue on macOS, IOCP on Windows), and when the OS says "this socket is readable now," the driver calls `wake()` on the waker the future provided. The executor then re-polls the future.

Here's the simplest possible executor (for understanding - don't use in production):

```rust
use std::future::Future;
use std::task::{Context, Poll, RawWaker, RawWakerVTable, Waker};
use std::pin::Pin;

fn block_on<F: Future>(mut future: F) -> F::Output {
    let mut future = unsafe { Pin::new_unchecked(&mut future) };

    // Minimal no-op waker: just spin-poll until ready
    fn noop_raw_waker() -> RawWaker {
        fn no_op(_: *const ()) {}
        fn clone(_: *const ()) -> RawWaker { noop_raw_waker() }
        let vtable = &RawWakerVTable::new(clone, no_op, no_op, no_op);
        RawWaker::new(std::ptr::null(), vtable)
    }
    let waker = unsafe { Waker::from_raw(noop_raw_waker()) };
    let mut cx = Context::from_waker(&waker);

    loop {
        match future.as_mut().poll(&mut cx) {
            Poll::Ready(value) => return value,
            Poll::Pending => std::thread::yield_now(), // real executors sleep here
        }
    }
}
```

The real Tokio executor does the same thing but instead of spin-looping on `Pending`, it parks the thread in an epoll wait - zero CPU usage until the OS says something is ready.

## What `.await` Actually Does

When you write:

```rust
let data = fetch_url(url).await;
```

The compiler rewrites this roughly as:

```rust
let mut fut = fetch_url(url);
let data = loop {
    match fut.poll(&mut cx) {
        Poll::Ready(val) => break val,
        Poll::Pending => { /* suspend here, yield control back to executor */ }
    }
};
```

The `async fn` becomes an enum (a state machine) where each `.await` point is a variant. The state machine stores all the local variables that are live across the `.await`. When the future is polled again after returning `Pending`, it jumps back to the exact `.await` point where it suspended - it picks up exactly where it left off.

## Why Pin Exists (and What the Error Means)

Every `async fn` that holds a reference across an `.await` point creates a **self-referential struct** - a state machine that contains both data and a reference to that data. If that struct were moved in memory (by something like a `Vec` resize), the internal reference would become dangling.

```rust
async fn example() {
    let data = String::from("hello");
    let slice = &data;          // slice points into data on the stack
    some_io().await;            // .await: state machine stores BOTH data AND slice
    println!("{slice}");        // uses the reference after await
}
// Generated state machine has both `data` and `slice` as fields.
// slice points into the data field. If the state machine moves → dangling pointer.
```

`Pin<P>` is the type that prevents the thing behind a pointer from being moved. Once a future is pinned, it stays at that memory address until it completes.

In practice:
- `Box::pin(future)` - heap-allocates the future and pins it there
- `tokio::pin!(future)` - macro that pins a future to the stack
- `Pin::new(&mut future)` - for futures that implement `Unpin` (can be safely moved)

**When you'll see `Unpin` errors:** If you write a function that takes `impl Future` and tries to call `.await` on it, you might need `impl Future + Unpin` or to box it. The error `the trait 'Unpin' is not implemented` means you're trying to use a future that might be self-referential in a context where it could be moved. The fix is almost always `Box::pin(your_future)`.

**`Unpin` is an auto-trait:** Types that are safe to move even after pinning implement `Unpin`. All primitives, most standard library types, and most futures you write implement `Unpin` automatically. Only `async fn` bodies that hold references across `.await` points do NOT implement `Unpin` - and the compiler handles that for you.
