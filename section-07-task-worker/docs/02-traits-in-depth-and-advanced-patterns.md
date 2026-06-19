# Doc 02 — Traits in Depth and Advanced Patterns

You know how to define a trait and implement it. That is the easy part.
The patterns in this document are what separate Rust code that merely compiles
from Rust code that is genuinely well-designed. They are also the patterns
you will see throughout taskforge's architecture.

---

## Object Safety: Why Not Every Trait Becomes `dyn Trait`

A trait is **object safe** if it can be used as `dyn Trait` — stored behind a
pointer whose concrete type is unknown at compile time. Not every trait qualifies.

The rules for object safety:

- Methods must not return `Self` (the vtable cannot know the concrete size)
- Methods must not have generic type parameters (the vtable would need infinite entries)
- The trait must not require `Self: Sized`
- Static methods (no `self` parameter) are excluded from the vtable

```rust
// Object safe — can be Box<dyn Drawable>
trait Drawable {
    fn draw(&self);
    fn bounding_box(&self) -> (f64, f64);
}

// NOT object safe — returns Self
trait Clone {
    fn clone(&self) -> Self; // Can't be in a vtable
}

// NOT object safe — generic method
trait Converter {
    fn convert<T>(&self) -> T; // Vtable can't hold infinite monomorphizations
}
```

**Why this matters for taskforge**: The `JobRepository` trait will be used as
`dyn JobRepository`. This means its methods must not have generic type parameters
and must not return `Self`. The `async fn` in traits complicates this further —
you will need the `async_trait` crate or careful design to make async trait methods
work with `dyn`.

---

## When `dyn Trait` Is the Right Tool 🟡

Static dispatch (generics) is faster. Dynamic dispatch (`dyn Trait`) is more
flexible. Neither is universally better.

Use `dyn Trait` when:

- You need a heterogeneous collection: `Vec<Box<dyn JobHandler>>`
- The concrete type is determined at runtime (plugin systems, registries)
- Reducing binary size matters more than maximum throughput on that code path
- You are on a cold path where one indirection per call is negligible

In taskforge, the worker's job handler registry is a perfect use of `dyn Trait`:

```rust
// Workers need to handle any job type, and types are determined at runtime
// from the job's type name stored in Redis
struct JobRegistry {
    handlers: HashMap<String, Box<dyn JobHandler>>,
}
```

The I/O to Redis dwarfs any cost from the vtable call. `dyn Trait` is fine here.

For the hot path inside a job's `execute()` method — prefer static dispatch.

---

## Blanket Implementations: The Standard Library Trick 🟡

A blanket implementation implements a trait for all types satisfying some bound.
The standard library is built on these.

```rust
// From the standard library: any Display type gets ToString for free
impl<T: fmt::Display> ToString for T {
    fn to_string(&self) -> String {
        format!("{self}")
    }
}
```

This is why every type that implements `Display` automatically has `.to_string()`.
You do not implement `ToString` directly — you implement `Display` and you get it.

You can write your own blanket implementations:

```rust
// Any type that implements Job automatically gets some helper methods
trait JobExt: Job {
    fn queue_name(&self) -> &'static str {
        Self::QUEUE
    }
    
    fn serialize_payload(&self) -> Result<serde_json::Value, serde_json::Error>
    where
        Self: Serialize,
    {
        serde_json::to_value(self)
    }
}

// Blanket impl — every Job gets these for free
impl<T: Job> JobExt for T {}
```

---

## Coherence: The Orphan Rule 🔴

Rust has a rule: you can only implement a trait for a type if **you own the trait
or you own the type** (or both). You cannot implement a trait you did not define
for a type you did not define.

```rust
// You cannot do this in your crate:
impl Display for Vec<String> {
    // Error: neither Display nor Vec belongs to you
}
```

**Why does this rule exist?** If two crates could both implement `Display` for
`Vec<String>`, and you depended on both, the compiler would not know which
implementation to use. The orphan rule prevents this conflict entirely.

**The workaround: newtype pattern.** If you need to implement a foreign trait
for a foreign type, wrap the type:

```rust
struct JobIdList(Vec<Uuid>);

impl Display for JobIdList {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let ids: Vec<_> = self.0.iter().map(|id| id.to_string()).collect();
        write!(f, "[{}]", ids.join(", "))
    }
}
```

Now you own `JobIdList`, so you can implement `Display` for it.

---

## Sealed Traits: Controlled Extensibility 🔴

Sometimes you want external code to **use** a trait but not **implement** it.
Maybe you are designing a versioned API and cannot allow third parties to add
implementations — adding a method to the trait later would break their code.

The sealed trait pattern uses Rust's visibility rules to enforce this:

```rust
// The seal lives in a private module
mod private {
    pub trait Sealed {}
}

// The public trait requires the private seal
pub trait JobState: private::Sealed {
    fn name(&self) -> &'static str;
}

// Only types IN THIS CRATE can implement Sealed, and thus JobState
pub struct Pending;
pub struct Running;
pub struct Completed;
pub struct Failed;

impl private::Sealed for Pending {}
impl private::Sealed for Running {}
impl private::Sealed for Completed {}
impl private::Sealed for Failed {}

impl JobState for Pending { fn name(&self) -> &'static str { "pending" } }
impl JobState for Running { fn name(&self) -> &'static str { "running" } }
// etc.
```

External users can write `fn process<S: JobState>(job: Job<S>)` and call
`S::name()` — they can USE the trait. But they cannot implement `Sealed`,
so they cannot add new states. This is intentional.

---

## Extension Traits: Adding Methods to Types You Don't Own

The orphan rule prevents implementing foreign traits on foreign types. But it
does not prevent you from defining a new trait and giving it a blanket implementation
for foreign types.

This is how the entire ecosystem adds methods to standard types:

```rust
pub trait JobIdExt {
    fn short_form(&self) -> String;
}

// Uuid is a foreign type, but JobIdExt belongs to us
impl JobIdExt for Uuid {
    fn short_form(&self) -> String {
        // First 8 characters of the UUID
        self.to_string()[..8].to_string()
    }
}

// Now any Uuid has .short_form() — once you import JobIdExt
```

The ecosystem naming convention: trait is named `FooExt`. Importers see:
`use taskforge::JobIdExt;` and suddenly their `Uuid` values have new methods.

This is how `futures::StreamExt`, `tokio::AsyncReadExt`, and `tower::ServiceExt`
all work.

---

## From/Into: The Conversion Pair 🟢

`From` and `Into` are implemented together — you implement `From<A> for B` and
automatically get `Into<B> for A` for free (via a blanket impl in the standard library).

The convention: **implement `From`, never implement `Into` directly.**

```rust
// Implement From
impl From<Uuid> for JobId {
    fn from(id: Uuid) -> Self {
        JobId(id)
    }
}

// You get Into for free
let uuid = Uuid::new_v4();
let job_id: JobId = uuid.into(); // Works automatically
```

For fallible conversions (where parsing can fail), use `TryFrom` / `TryInto`:

```rust
impl TryFrom<&str> for Email {
    type Error = EmailError;
    
    fn try_from(s: &str) -> Result<Self, Self::Error> {
        if s.contains('@') && s.contains('.') {
            Ok(Email(s.to_string()))
        } else {
            Err(EmailError::InvalidFormat)
        }
    }
}
```

---

## How Tower Is Built on Traits 🔴

In Section 6 you used Tower's `Service` trait for middleware. It is worth
understanding why it works the way it does, because taskforge uses a similar
pattern for job handlers.

Tower's core abstraction:

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```

Every middleware implements `Service<Request>`. A logging middleware wraps an
inner `Service` and implements `Service` itself. A rate-limiting middleware wraps
that. The result is a stack of services, each transparently composable.

The key insight: because `Service` has associated types for `Response`, `Error`,
and `Future`, a single `Service` implementation can be generic over these without
anyone needing to write `Box<dyn Future>`. The types thread through the stack
at compile time.

Taskforge's `JobHandler` trait follows the same pattern:

```rust
#[async_trait]
pub trait JobHandler: Send + Sync {
    async fn handle(
        &self,
        payload: serde_json::Value,
    ) -> Result<serde_json::Value, JobError>;
}
```

It is simpler than Tower (no backpressure), but the idea is the same: everything
goes through a single trait, and you can stack, compose, and substitute
implementations freely.

---

## How It Breaks

**Blanket implementations conflicting.** You implement `Display` for all `T: MyTrait`, but the standard library implements `ToString` for all `T: Display`. Now there are two implementations of some trait for the same type, and the compiler complains.

**Object safety violation at usage.** You have a perfectly valid trait, but when you write `Box<dyn MyTrait>` the compiler says "MyTrait is not object safe." This happens if the trait has methods that return `Self`, take generic parameters, or have `where Self: Sized`.

**Orphan rule limiting design.** You want to implement `From<ExternalError>` for your error type, but you don't own `ExternalError` and you don't own `From`. Use a newtype wrapper around `ExternalError`.

**Sealed trait leaking.** Your sealed trait relies on a private module. If someone finds the module path and imports it, the seal is broken. It's a convention, not an enforced guarantee.

---

## Common Mistakes

**1. Trying to put `dyn Trait` in a collection when the trait is not object safe.**
The error "the trait cannot be made into an object" means you have a generic method
or a `Self`-returning method. Fix it by removing the generic or adding `where Self: Sized`
to exclude that method from the vtable.

**2. Implementing both the trait and the blanket impl for a type.** If you write
a blanket impl `impl<T: Display> MyTrait for T {}` and then try to write
`impl MyTrait for String {}`, you get a coherence conflict. The blanket already
covers String. Pick one.

**3. Forgetting that sealed traits still require `use` for the `Sealed` bound
to be visible.** If you put the seal in `mod private` (private module), internal
code must still bring it into scope for the where clause to work. Within the
same crate this is usually fine.

**4. Writing extension traits that conflict with inherent methods.** If `Uuid`
already has a method `.to_simple()` and you add it via an extension trait, Rust
will prefer the inherent method. Your extension method becomes invisible without
an explicit fully qualified call. Name extension methods to avoid conflicts.

**5. Reaching for `dyn Trait` when you should use an enum.** If you have three
concrete job types and they are fixed, an enum is faster than `dyn Trait` (no
vtable, stored inline, no heap allocation). `dyn Trait` is for when the set of
types is genuinely open.
