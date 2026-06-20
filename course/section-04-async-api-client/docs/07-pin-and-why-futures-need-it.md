# Doc 07 - Pin and Why Futures Need It

🔴 This is the most confusing thing in async Rust. Understanding it unlocks everything.

You've seen `Pin<Box<dyn Future<Output = T>>>` in function signatures and `Box::pin(future)` in code. You've probably also seen the compiler error "future cannot be moved after being pinned." This doc explains what `Pin` is, why the async runtime requires it, and the three practical patterns you'll use.

---

## The Root Cause: Self-Referential State Machines

When you write an `async fn`, the compiler transforms it into a state machine - an enum that stores all the variables that need to persist across `.await` points:

```rust
async fn fetch_two(url1: &str, url2: &str) -> (String, String) {
    let response1 = reqwest::get(url1).await.unwrap().text().await.unwrap();
    // At this .await, the runtime saves our state and runs other tasks
    let response2 = reqwest::get(url2).await.unwrap().text().await.unwrap();
    (response1, response2)
}
```

The compiler turns this into something like:

```rust
// What the compiler generates (simplified):
enum FetchTwoStateMachine<'a> {
    State0 { url1: &'a str, url2: &'a str },              // Before first await
    State1 { url1: &'a str, url2: &'a str, req1: Request }, // Waiting for req1
    State2 { url2: &'a str, response1: String },           // Waiting for req2
    Complete,
}
```

So far, so good. But consider a more subtle case - what if you hold a reference to a local variable across an await:

```rust
async fn process() {
    let data = vec![1, 2, 3, 4, 5];
    let first = &data[0];  // reference into `data`
    
    do_something_async().await;  // Must save state here
    
    println!("First was: {first}");  // Use reference after await
}
```

The state machine must store both `data` (the Vec) AND `first` (a reference that points INTO `data`). If the state machine is ever **moved in memory**, `data` moves to a new address, but `first` still points to the old address - a dangling pointer.

This is a **self-referential struct**: a struct that contains a reference to its own field.

---

## Why Moving Breaks Self-Referential Structs

```
Before move:
  State machine at address 0x1000:
    data: Vec<i32> → heap at 0x5000
    first: *const i32 = 0x5000  ← points into `data`
  
  Everything is consistent.

After move to address 0x2000:
  State machine now at 0x2000:
    data: Vec<i32> → heap at 0x5000  ← Vec's heap allocation didn't move (that's fine)
    first: *const i32 = 0x5000  ← still fine, Vec heap didn't move
    
  Actually fine here because `data` is a Vec (heap-allocated separately).
  
But if `first` referenced a FIELD of `data` stored inline in the state machine:
  data: [1, 2, 3] stored at offset 0 of state machine  → was at 0x1000
  first: *const i32 = 0x1000  ← now dangling! Data moved to 0x2000.
```

This happens whenever you hold a reference across an await to a local variable that's stored inline in the state machine struct. The compiler generates these structs, so it's inherent to async.

---

## `Pin<P>`: A Wrapper That Prevents Moving

`Pin<P>` is a wrapper around a pointer `P` that **prevents moving the value it points to**:

```rust
use std::pin::Pin;

let mut my_future = async { 42 };

// Pin on the heap - the future now has a stable address forever
let pinned: Pin<Box<dyn Future<Output = i32>>> = Box::pin(my_future);

// The future's address is pinned - it will never move.
// The Box itself can be moved (that just copies the pointer).
```

The key insight: `Pin` doesn't pin the _pointer_ - it pins the _value behind the pointer_. A `Pin<Box<T>>` can be moved (you're just moving the 8-byte pointer), but the heap allocation that `T` lives in never moves.

---

## The `Future::poll` Signature Requires `Pin`

This is why Pin exists in the first place:

```rust
pub trait Future {
    type Output;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

`poll` takes `Pin<&mut Self>` instead of `&mut Self`. This is the guarantee: when the runtime calls `poll`, the future is pinned - its address is stable. Internal self-references are valid because nothing will move the state machine.

Before the runtime can call `poll` on your future, it must pin it. That's why `tokio::spawn` and other task spawners require their futures to be `Pin`ned (or pinnable - they handle the pinning internally).

---

## The Three Practical Patterns

In day-to-day async programming, Pin appears in three situations:

### 1. `Box::pin()` - Heap allocation with a stable address

```rust
use std::future::Future;

// When you need to store a future in a collection or return it from a function:
let futures: Vec<Pin<Box<dyn Future<Output = String>>>> = vec![
    Box::pin(fetch("https://api.github.com/repos/rust-lang/rust")),
    Box::pin(fetch("https://api.github.com/repos/tokio-rs/tokio")),
    Box::pin(fetch("https://api.github.com/repos/serde-rs/serde")),
];

// Now you can await them concurrently:
let results = futures::future::join_all(futures).await;
```

`Box::pin(future)` heap-allocates the future and wraps it in `Pin<Box<T>>`. The future's address is now stable for its entire lifetime.

### 2. `tokio::pin!()` or `std::pin::pin!()` - Stack pinning

```rust
use tokio::time::{timeout, Duration};

async fn with_timeout() {
    let my_future = expensive_computation();
    
    // Pin it on the stack (avoids heap allocation):
    tokio::pin!(my_future);
    // my_future is now Pin<&mut impl Future>
    
    match timeout(Duration::from_secs(5), &mut my_future).await {
        Ok(result) => println!("Got: {result}"),
        Err(_) => println!("Timed out"),
    }
    
    // Can still use my_future after timeout if needed!
    // (Because &mut Pin<...> lets you re-poll after timeout)
}
```

`tokio::pin!` is a macro that pins the future to the current stack frame. It rebinds the variable to `Pin<&mut F>`. Stack pinning avoids heap allocation but the future's lifetime is tied to the current stack frame.

### 3. `Pin<Box<dyn Future>>` - Type-erased futures

```rust
// When you need to store futures of different types in the same collection:
fn make_tasks() -> Vec<Pin<Box<dyn Future<Output = i32> + Send>>> {
    vec![
        Box::pin(async { 1 }),
        Box::pin(async { 2 + 2 }),
        Box::pin(async { heavy_computation().await }),
    ]
}

// This works because dyn Future erases the specific state machine type,
// and Box::pin provides the stable address Pin requires.
```

This is the pattern for building batches of concurrent requests in the GitHub analyzer project.

---

## `Unpin`: The Escape Hatch

Most types in Rust are `Unpin` - they can be moved freely even when "pinned." For them, `Pin<&mut T>` is identical to `&mut T`:

```rust
// These are all Unpin - most types are:
let n: i32 = 42;
let s: String = "hello".into();
let v: Vec<u8> = vec![1, 2, 3];

// For Unpin types, Pin is just a formality:
let pinned: Pin<&mut i32> = Pin::new(&mut n);
// You can get the &mut back:
let mutable: &mut i32 = Pin::into_inner(pinned);  // Only works if i32: Unpin
```

**Which types are `!Unpin` (cannot be moved when pinned)?**
- Futures generated by `async fn` and `async {}` blocks - because they contain self-referential state
- Types that manually implement `!Unpin` to express some invariant

**Which types are `Unpin` (the vast majority)?**
- All primitives: `i32`, `f64`, `bool`, etc.
- `String`, `Vec<T>`, `HashMap<K,V>`, `Box<T>`
- References: `&T`, `&mut T`
- Any type where all fields are `Unpin`

For types you write yourself that don't contain async state, you're `Unpin` automatically. You only need to think about `Unpin` when writing custom `Future` implementations.

---

## Connecting to the GitHub Analyzer Project

The GitHub analyzer fetches data from multiple repositories concurrently. Here's where Pin shows up in practice:

```rust
use std::future::Future;
use std::pin::Pin;

type BoxFuture<T> = Pin<Box<dyn Future<Output = T> + Send>>;

pub struct RepoAnalyzer {
    client: reqwest::Client,
}

impl RepoAnalyzer {
    // Returns a type-erased future - callers don't see the concrete Future type
    pub fn fetch_repo(&self, repo: &str) -> BoxFuture<Result<RepoInfo, Error>> {
        let client = self.client.clone();
        let repo = repo.to_string();
        
        Box::pin(async move {
            let url = format!("https://api.github.com/repos/{repo}");
            let info: RepoInfo = client.get(&url)
                .send().await?
                .json().await?;
            Ok(info)
        })
    }

    // Fetch multiple repos concurrently using a Vec of pinned futures
    pub async fn analyze_all(&self, repos: &[&str]) -> Vec<Result<RepoInfo, Error>> {
        let futures: Vec<BoxFuture<Result<RepoInfo, Error>>> = repos.iter()
            .map(|repo| self.fetch_repo(repo))
            .collect();

        futures::future::join_all(futures).await
    }
}
```

The `BoxFuture<T>` type alias (`Pin<Box<dyn Future<Output = T> + Send>>`) appears constantly in async code that stores or returns futures. Naming it avoids writing the full type signature repeatedly.

---

## How It Breaks

**Calling `Pin::new()` on a `!Unpin` type.**
`Pin::new()` only works if the type is `Unpin`. Async blocks are `!Unpin`. Trying to use `Pin::new()` on an async block gives: "the trait `Unpin` is not implemented for `{async block}...`". Use `Box::pin()` or `tokio::pin!()` instead.

**Storing a future in a field and then moving the containing struct.**
If you store a `Pin<Box<dyn Future>>` in a struct and then move the struct, the `Box`'s *pointer* moves, but the future stays put on the heap - this is fine. But if you have an inline future (not boxed) and you move the struct, you've moved the future - undefined behavior if it's self-referential. The compiler prevents this with `!Unpin`, but the error message can be confusing.

**Trying to `mem::swap` two pinned values.**
```rust
let mut a = async { 1 };
let mut b = async { 2 };
std::mem::swap(&mut a, &mut b);  // This moves both - fine before pinning
// But after Box::pin or tokio::pin!, you can't swap - Pin prevents it
```
This is intentional. Two self-referential futures can't be swapped without corrupting their internal pointers.

**Using `Box::pin` when `tokio::pin!` would avoid the allocation.**
`Box::pin` allocates. If the future is only used locally and there's no need to store it in a `Vec` or return it, use `tokio::pin!` (stack pinning, zero allocation):
```rust
// Unnecessary allocation:
let fut = Box::pin(expensive_computation());
let result = fut.await;

// Stack pinned - no allocation:
let fut = expensive_computation();
tokio::pin!(fut);
let result = fut.await;
```
