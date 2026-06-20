# Doc 03 - Async Trait Objects and Dynamic Dispatch

🟡 nexus has a `MessageStore` - something that persists messages for replay. In
tests it's an in-memory `HashMap`. In production it's ironkv from Section 8. In the
future it might be PostgreSQL. The interface is the same; the implementation varies.
This doc covers how to write traits with async methods and use them with dynamic
dispatch.

---

## The Core Problem: Async Methods in Traits

In synchronous Rust, a trait with a method is straightforward:

```rust
trait MessageStore {
    fn store(&self, topic: &str, seq: u64, payload: &[u8]) -> Result<(), StoreError>;
    fn load(&self, topic: &str, since_seq: u64) -> Result<Vec<(u64, Vec<u8>)>, StoreError>;
}
```

Adding `async` to these methods worked as of Rust 1.75 (stable since December 2023):

```rust
trait MessageStore {
    async fn store(&self, topic: &str, seq: u64, payload: &[u8]) -> Result<(), StoreError>;
    async fn load(&self, topic: &str, since_seq: u64) -> Result<Vec<(u64, Vec<u8>)>, StoreError>;
}
```

This compiles. Static dispatch with generics works:

```rust
async fn replay_messages<S: MessageStore>(
    store: &S,
    topic: &str,
    since: u64,
    mut sink: impl futures::Sink<Frame>,
) -> Result<(), NexusError> {
    let messages = store.load(topic, since).await?;
    for (seq, payload) in messages {
        sink.send(Frame::deliver(seq, payload.into())).await
            .map_err(|_| NexusError::ClientDisconnected)?;
    }
    Ok(())
}
```

**The problem appears with `dyn` dispatch:**

```rust
// ❌ Doesn't compile
async fn replay_messages_dyn(
    store: &dyn MessageStore,  // Error: the trait MessageStore is not dyn-compatible
    ...
```

The error says "not dyn-compatible" (formerly called "not object-safe"). The reason:
`async fn` desugars to returning `impl Future<Output = T>`. Each implementation
returns a different concrete future type - `InMemoryStore::store` returns one type,
`IronkvStore::store` returns another. `dyn MessageStore` requires a single known
vtable, but there's no single return type.

---

## Why dyn Matters

With generics only (`<S: MessageStore>`), the caller's type must be known at compile
time. The handler struct can't store a `Box<dyn MessageStore>`:

```rust
// ❌ Generic struct - works but requires the type at compile time everywhere
struct NexusEngine<S: MessageStore> {
    store: S,
}
// Every type that contains NexusEngine must also carry <S> in its type signature.
// A `HashMap<u64, Box<dyn ConnectionHandler>>` won't work because each handler
// might have a different <S>.

// ✅ Dynamic dispatch - store is type-erased
struct NexusEngine {
    store: Arc<dyn MessageStore + Send + Sync>,  // Problem: async methods aren't dyn-compatible
}
```

For nexus, the engine must be `Send + Sync + 'static` to be shared across async
tasks. The store must be type-erased so users can provide different implementations
without changing the engine's type signature.

---

## How Vtables Work and Why Async Breaks Them

`dyn Trait` works by building a vtable - a struct of function pointers, one per
trait method. For a synchronous method like `fn store(&self, ...) -> Result<(), E>`,
the vtable entry is a function pointer: `fn(*const (), ...) -> Result<(), E>`. Every
implementation has the same return type, so every implementation's function pointer
fits the same slot.

`async fn` changes the return type. It's syntactic sugar for:

```rust
// This async fn:
async fn store(&self, topic: &str, seq: u64, payload: &[u8]) -> Result<(), StoreError>;

// Desugars to an RPITIT (return-position impl Trait in trait):
fn store(&self, topic: &str, seq: u64, payload: &[u8])
    -> impl Future<Output = Result<(), StoreError>>;
```

Each implementing type returns a *different concrete future type*. `InMemoryStore`
returns `InMemoryStoreFuture<'_>`; `IronkvStore` returns `IronkvStoreFuture<'_>`.
These types are different sizes and different function pointer shapes. A single vtable
slot cannot hold a pointer to functions with different return types. This is why
`dyn MessageStore` fails: there is no single function pointer type that works for
all implementations.

The fix is to make the return type *the same for all implementations*:
`Pin<Box<dyn Future<Output = T> + Send>>`. This is a type-erased heap allocation -
every implementation can return it, and it fits in a single vtable slot.

## Solution 1: Box<dyn Future> Manually

The workaround before Rust 1.75 and the solution for `dyn` today: return a boxed future:

```rust
// nexus-core/src/store.rs

use std::future::Future;
use std::pin::Pin;

// Alias for readability
type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;

// The dyn-compatible version: methods return BoxFuture instead of async fn
pub trait MessageStore: Send + Sync {
    fn store<'a>(
        &'a self,
        topic: &'a str,
        seq: u64,
        payload: &'a [u8],
    ) -> BoxFuture<'a, Result<(), StoreError>>;

    fn load<'a>(
        &'a self,
        topic: &'a str,
        since_seq: u64,
    ) -> BoxFuture<'a, Result<Vec<(u64, Vec<u8>)>, StoreError>>;
}
```

Implementing for the in-memory store:

```rust
pub struct InMemoryStore {
    inner: Arc<RwLock<HashMap<String, Vec<(u64, Bytes)>>>>,
}

impl MessageStore for InMemoryStore {
    fn store<'a>(
        &'a self,
        topic: &'a str,
        seq: u64,
        payload: &'a [u8],
    ) -> BoxFuture<'a, Result<(), StoreError>> {
        Box::pin(async move {
            let mut map = self.inner.write().await;
            map.entry(topic.to_string())
                .or_default()
                .push((seq, Bytes::copy_from_slice(payload)));
            Ok(())
        })
    }

    fn load<'a>(
        &'a self,
        topic: &'a str,
        since_seq: u64,
    ) -> BoxFuture<'a, Result<Vec<(u64, Vec<u8>)>, StoreError>> {
        Box::pin(async move {
            let map = self.inner.read().await;
            let messages = map.get(topic)
                .map(|msgs| {
                    msgs.iter()
                        .filter(|(seq, _)| *seq > since_seq)
                        .map(|(seq, payload)| (*seq, payload.to_vec()))
                        .collect()
                })
                .unwrap_or_default();
            Ok(messages)
        })
    }
}
```

Now `dyn MessageStore` works:

```rust
struct NexusEngine {
    store: Arc<dyn MessageStore>,  // ✅ dyn-compatible
}

// In tests:
let engine = NexusEngine {
    store: Arc::new(InMemoryStore::new()),
};

// In production:
let engine = NexusEngine {
    store: Arc::new(IronkvStore::open("messages.ikv")?),
};
```

---

## Solution 2: `trait_variant` for Send Bounds

When you need `async fn` syntax with an auto-generated `Send`-bounded variant,
the `trait-variant` crate (official, from the Rust team) does the boilerplate:

```toml
# nexus-core/Cargo.toml
[dependencies]
trait-variant = "0.1"
```

```rust
use trait_variant::make;

// The make attribute generates two traits:
// - MessageStore (the base trait, async fn syntax)
// - SendMessageStore (auto-generated variant with + Send bounds on futures)
#[make(SendMessageStore: Send)]
pub trait MessageStore {
    async fn store(&self, topic: &str, seq: u64, payload: &[u8]) -> Result<(), StoreError>;
    async fn load(&self, topic: &str, since_seq: u64) -> Result<Vec<(u64, Vec<u8>)>, StoreError>;
}
```

This generates something equivalent to:

```rust
pub trait SendMessageStore: Send + Sync {
    fn store<'a>(&'a self, topic: &'a str, seq: u64, payload: &'a [u8])
        -> impl Future<Output = Result<(), StoreError>> + Send + 'a;
    fn load<'a>(&'a self, topic: &'a str, since_seq: u64)
        -> impl Future<Output = Result<Vec<(u64, Vec<u8>)>, StoreError>> + Send + 'a;
}
```

Use `SendMessageStore` in generic bounds when you need `Send`:

```rust
// Static dispatch - requires Send
async fn spawn_store_worker<S: SendMessageStore + 'static>(store: Arc<S>) {
    tokio::spawn(async move {
        // This task runs on any thread - needs Send
        store.store("metrics", 1, b"data").await.ok();
    });
}
```

For `dyn` dispatch with `Send`, still use `BoxFuture`. `trait_variant` helps with
the `Send` requirement at the trait level, not with dyn compatibility.

---

## Solution 3: `async-trait` Crate (Legacy)

Before Rust 1.75, the `async-trait` crate was the standard solution. You may
encounter it in older codebases:

```toml
[dependencies]
async-trait = "0.1"
```

```rust
use async_trait::async_trait;

#[async_trait]
pub trait MessageStore: Send + Sync {
    async fn store(&self, topic: &str, seq: u64, payload: &[u8]) -> Result<(), StoreError>;
    async fn load(&self, topic: &str, since_seq: u64) -> Result<Vec<(u64, Vec<u8>)>, StoreError>;
}
```

`#[async_trait]` is a proc-macro that rewrites the trait to use `BoxFuture` internally -
the same as Solution 1, but automatically. It makes `dyn MessageStore` work.

**Should you use `async-trait` in new code?** Generally no - prefer the manual
`BoxFuture` approach for dyn dispatch, or RPITIT (`async fn` in traits) for generic
dispatch. `async-trait` adds a heap allocation per call, which is identical to the
manual `Box::pin(async move { ... })`. But since the macro hides this cost, it's
less obvious.

**Decision tree:**

```
Do you need dyn dispatch? ─── No ──→ Use `async fn` in trait (Rust 1.75+)
         │                           Works with generics and static dispatch
         Yes
         │
Do you need Send? ─── No ──→ BoxFuture without + Send
         │                    Works with dyn, but only single-threaded runtimes
         Yes
         │
Use BoxFuture + Send:
    fn method<'a>(&'a self) -> Pin<Box<dyn Future<Output = T> + Send + 'a>>
```

---

## nexus: Putting It Together

The complete `MessageStore` trait for nexus, designed for `dyn` dispatch:

```rust
// nexus-core/src/store.rs
use std::future::Future;
use std::pin::Pin;
use bytes::Bytes;

pub type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;

#[derive(Debug, thiserror::Error)]
pub enum StoreError {
    #[error("topic not found: {0}")]
    TopicNotFound(String),
    #[error("sequence overflow")]
    SequenceOverflow,
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
}

pub trait MessageStore: Send + Sync + 'static {
    /// Store a message and return the assigned sequence number.
    fn store<'a>(
        &'a self,
        topic: &'a str,
        seq: u64,
        payload: Bytes,
    ) -> BoxFuture<'a, Result<(), StoreError>>;

    /// Return messages with seq > since_seq, in order.
    fn replay<'a>(
        &'a self,
        topic: &'a str,
        since_seq: u64,
    ) -> BoxFuture<'a, Result<Vec<(u64, Bytes)>, StoreError>>;

    /// Return the highest sequence number for a topic, or 0 if empty.
    fn latest_seq<'a>(
        &'a self,
        topic: &'a str,
    ) -> BoxFuture<'a, Result<u64, StoreError>>;
}

// The engine holds a type-erased store:
pub struct NexusEngine {
    store: Arc<dyn MessageStore>,
    // ...
}

impl NexusEngine {
    pub fn new(store: impl MessageStore + 'static) -> Self {
        Self {
            store: Arc::new(store),
        }
    }
}
```

Usage in tests:

```rust
#[tokio::test]
async fn test_message_replay() {
    let engine = NexusEngine::new(InMemoryStore::new());
    
    engine.store.store("alerts", 1, Bytes::from("msg1")).await.unwrap();
    engine.store.store("alerts", 2, Bytes::from("msg2")).await.unwrap();
    
    let messages = engine.store.replay("alerts", 0).await.unwrap();
    assert_eq!(messages.len(), 2);
    assert_eq!(messages[0].1, "msg1");
}
```

Swapping in a real store is a one-line change:

```rust
let engine = NexusEngine::new(IronkvStore::open("messages.ikv").await?);
```

The same test suite runs against both stores. This is the value of dynamic dispatch:
testability and flexibility at the cost of one heap allocation per async method call.

---

## Object Safety: The Full Rules

A trait is not dyn-compatible (not "object-safe") if any method:

1. Has a generic type parameter (`fn method<T>(&self, value: T)`)
2. Returns `impl Trait` (including `async fn`, which desugars to `impl Future`)
3. Takes or returns `Self` by value in a non-obvious way
4. Has `where Self: Sized` bounds

**Fixing common violations:**

```rust
// ❌ Generic method - not dyn-compatible
trait Encoder {
    fn encode<W: Write>(&self, writer: &mut W) -> io::Result<()>;
}

// ✅ Use trait objects or type-erasure
trait Encoder {
    fn encode(&self, writer: &mut dyn Write) -> io::Result<()>;
}

// ❌ Returns impl Trait (async fn)
trait Processor {
    async fn process(&self, data: &[u8]) -> Vec<u8>;
}

// ✅ BoxFuture
trait Processor {
    fn process<'a>(&'a self, data: &'a [u8]) -> BoxFuture<'a, Vec<u8>>;
}
```

---

## Async Closures (Rust 1.85+)

Async closures are a newer feature useful for middleware and callbacks:

```rust
// An async closure that processes a frame before it's dispatched
type Middleware = Box<dyn Fn(Frame) -> Pin<Box<dyn Future<Output = Frame> + Send>> + Send + Sync>;

// Since Rust 1.85: async closures capture their environment correctly
async fn with_logging_middleware<F, Fut>(handler: F, frame: Frame) -> Frame
where
    F: async Fn(Frame) -> Frame,
{
    let input_cmd = frame.command;
    let result = handler(frame).await;
    tracing::debug!(?input_cmd, "Frame processed by middleware");
    result
}
```

Async closures (`async Fn`) solve a subtle problem with earlier approaches where
a closure that creates a future couldn't borrow from its environment for the
duration of the future. With `async Fn`, the borrow is scoped to each `.await`.

For nexus, middleware is most naturally expressed as a chain of functions that
transform or inspect frames. Async closures make this composable.

---

## Exercises

**Exercise 1 - InMemoryStore**

Implement `MessageStore` for `InMemoryStore`:
- `store()`: insert `(seq, payload)` into `HashMap<String, Vec<(u64, Bytes)>>`
- `replay()`: return entries with `seq > since_seq`, in ascending order
- `latest_seq()`: return the maximum seq for a topic, or 0 if empty

Wrap the inner `HashMap` in `Arc<tokio::sync::RwLock<...>>` so the store is `Send + Sync`.
Write a test that stores 3 messages and replays from seq 1, verifying only messages
with seq > 1 are returned.

**Exercise 2 - NullStore**

Implement a `NullStore` that implements `MessageStore` by discarding all messages:
- `store()` returns `Ok(())`
- `replay()` returns `Ok(vec![])`
- `latest_seq()` returns `Ok(0)`

Use `NullStore` in a unit test of the publish path to verify publish logic without
needing a real store. This tests the principle: swapping implementations is a one-line
change because of `dyn MessageStore`.

**Exercise 3 - dyn Compatibility Verification**

Try adding a generic method to `MessageStore`:

```rust
// Add this to the trait and observe the compiler error
fn store_many<I: IntoIterator<Item = (u64, Bytes)>>(&self, topic: &str, items: I) -> BoxFuture<'_, Result<(), StoreError>>;
```

Read the compiler error about dyn compatibility. Then fix it by changing the signature
to accept `Vec<(u64, Bytes)>` instead of `I: IntoIterator`. Explain in a comment why
generic methods break dyn compatibility.

---

## Checklist

- [ ] Traits that need `dyn` dispatch use `BoxFuture` return types, not `async fn`
- [ ] Traits that only need generic/static dispatch use `async fn` (Rust 1.75+)
- [ ] All `dyn Trait` usages have `Send + Sync + 'static` bounds in threaded contexts
- [ ] `Box::pin(async move { ... })` for each `BoxFuture` implementation
- [ ] `trait_variant::make` only when you need both variants (with and without Send)
- [ ] `async-trait` only in legacy code or when maintaining API compatibility
