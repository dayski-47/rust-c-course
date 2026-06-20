# Doc 01 — Generics: The Full Picture

---

## Engineering Methodology: Type-Driven Design + Invariant Thinking

Type-driven design means using the type system to make invalid states impossible to represent in code. Not just to represent data, but to enforce correctness at compile time.

The core question: "If this type compiles, is it necessarily correct?" For most code, the answer is "not really — you can pass the wrong ID, call methods in the wrong order, put a job in the wrong state." Type-driven design works to make the answer "yes."

Invariant thinking means asking: "What must ALWAYS be true? What must NEVER happen?"

For the task worker:
- MUST always be true: a Running job always has a worker assigned to it. A Completed job always has a result. A Failed job always has an error message and a retry count.
- MUST NEVER happen: a job transitions from Pending directly to Completed (skipping Running). A job has retry count > max retries but isn't in the Dead state.

When you encode these invariants in types — using type-state for job lifecycle, newtypes for IDs — the compiler enforces them. You don't write runtime checks. You don't write unit tests for "can this job skip states." The code that could cause it literally doesn't compile.

---

You have used generics. You have written `fn foo<T: Display>(val: T)` and moved on.
But there is a lot happening under the hood that changes how you design systems —
especially once your job queue needs a `Worker<J: Job>` that is fast, correct, and
does not bloat your binary. This document explains what generics actually are in Rust,
when they hurt you, and several patterns you will use constantly in taskforge.

---

## What Monomorphization Actually Means

When the Rust compiler sees a generic function, it does not produce generic machine code.
It produces **one copy of the function per concrete type you use it with**. This is called
monomorphization.

```rust
fn max_of<T: PartialOrd>(a: T, b: T) -> T {
    if a >= b { a } else { b }
}

fn main() {
    max_of(3_i32, 5_i32);       // generates max_of_i32
    max_of(2.0_f64, 7.0_f64);   // generates max_of_f64
}
```

Conceptually, the compiler produces:

```rust
fn max_of_i32(a: i32, b: i32) -> i32 { if a >= b { a } else { b } }
fn max_of_f64(a: f64, b: f64) -> f64 { if a >= b { a } else { b } }
```

These are separate functions. The optimizer can inline them, vectorize them, or
specialize them independently. There is zero runtime overhead compared to writing
the concrete version yourself.

This is fundamentally different from Java generics (type erased at runtime) or
C# generics (JIT specialization). In Rust, the specialization happens at
compile time, and you pay nothing at runtime.

**Comparison with C++ templates**: Rust generics work like templates but with one
critical difference. In C++, template errors appear at instantiation — often inside
library code, giving you walls of incomprehensible error text. In Rust, the bound
`T: PartialOrd` is checked when you write the function. If T does not satisfy the
bound, you get a clear error pointing at your code, not at the standard library
internals.

---

## 🟡 The Cost You Do Not Think About: Binary Size

Every monomorphized instantiation is a separate function in the binary. If you use
a generic `serialize<T: Serialize>(val: &T)` function with 50 different types, you
get 50 copies of that function body in the final binary. On small embedded systems
this is a real problem. On a server it is usually fine but worth knowing.

The mitigation is called the "outline" pattern: extract the non-generic core into
a non-generic function and keep the generic wrapper thin.

```rust
// The generic wrapper is small — just converts T to Value
pub fn serialize<T: Serialize>(val: &T) -> Result<Vec<u8>, serde_json::Error> {
    let json_value = serde_json::to_value(val)?;
    // Delegate to non-generic function that exists once in the binary
    serialize_value(json_value)
}

// This exists exactly once, regardless of how many T types you use
fn serialize_value(val: serde_json::Value) -> Result<Vec<u8>, serde_json::Error> {
    serde_json::to_vec(&val)
}
```

The rule of thumb: use generics (monomorphized, static dispatch) for hot paths where
inlining matters. Use `dyn Trait` (dynamic dispatch, vtable) for cold paths where
the flexibility is worth the one-pointer-indirection cost. Logging, configuration,
error handling — all fine as `dyn Trait`. Job processing loops — probably worth
keeping generic.

---

## Where Clauses for Complex Bounds

When your bounds get complicated, `where` clauses keep signatures readable.

```rust
// Hard to read inline
fn process<T: Serialize + Send + 'static, E: std::error::Error + Send>(
    item: T,
) -> Result<(), E> { ... }

// Same thing, readable
fn process<T, E>(item: T) -> Result<(), E>
where
    T: Serialize + Send + 'static,
    E: std::error::Error + Send,
{ ... }
```

You can also add bounds on associated types:

```rust
fn process_job<J>(job: J) -> Result<J::Output, J::Error>
where
    J: Job,
    J::Output: Serialize,
    J::Error: std::error::Error,
{ ... }
```

This is how you will bound the `Job` trait in taskforge. The trait itself specifies
that `Output` must be `Serialize`, and the worker function adds whatever extra
constraints it needs.

---

## 🟡 Associated Types vs Generic Type Parameters

This is one of the most important design decisions in trait-heavy Rust code, and it
trips up almost everyone coming from other languages.

```rust
// Associated type approach
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// Generic type parameter approach  
trait Convert<T> {
    fn convert(&self) -> T;
}
```

The rule is: **use an associated type when there is exactly one natural choice of
type per implementor. Use a generic parameter when a type could meaningfully implement
the trait for many different type arguments.**

`Iterator` uses an associated type because `Vec<i32>`'s iterator produces `i32`s.
There is no ambiguity. Asking "what does this iterator produce?" has one answer.

`Convert<T>` uses a generic parameter because `i32` can be converted to `f64`,
to `String`, to `bool`, and to other things. A single type can implement `Convert`
multiple times. Asking "what can this convert to?" has many answers.

For taskforge's `Job` trait, `Output` and `Error` will be associated types.
A `SendEmailJob` produces exactly one kind of output and exactly one kind of error.
There is no ambiguity. The compiler can always infer what `J::Output` means without
you telling it.

---

## Generic Structs: Carrying the Job Type

```rust
pub struct Worker<J: Job> {
    id: WorkerId,
    repository: Arc<dyn JobRepository>,
    _job_type: PhantomData<J>,
}
```

The struct carries the job type `J` as a type parameter. This means the compiler
will generate a separate `Worker<SendEmailJob>`, `Worker<ResizeImageJob>`, etc.
Each one is fully optimized for its specific job type.

---

## Const Generics 🟢

Since Rust 1.51, you can parameterize types over constant values, not just over types.

```rust
struct RingBuffer<T, const N: usize> {
    data: [T; N],
    head: usize,
    tail: usize,
}

impl<T: Default + Copy, const N: usize> RingBuffer<T, N> {
    fn new() -> Self {
        RingBuffer {
            data: [T::default(); N],
            head: 0,
            tail: 0,
        }
    }
}

// Different sizes are different types at compile time
let small: RingBuffer<u8, 64> = RingBuffer::new();
let large: RingBuffer<u8, 4096> = RingBuffer::new();
```

In taskforge, you might use const generics for a fixed-size batch of jobs or
for priority levels.

---

## PhantomData: Carrying a Type Without Storing It 🟡

`PhantomData<T>` is a zero-sized type. It takes no space at runtime. Its sole
purpose is to tell the compiler "this struct logically owns or relates to T,
even though it does not store a T."

```rust
use std::marker::PhantomData;

// This struct carries the job type in the type system but not in memory
pub struct TypedQueue<J> {
    redis_key: String,
    _job: PhantomData<J>,
}

impl<J: Job> TypedQueue<J> {
    pub fn new(key: &str) -> Self {
        TypedQueue {
            redis_key: key.to_string(),
            _job: PhantomData,
        }
    }
}
```

Without `PhantomData`, the compiler would say "parameter J is never used" and
refuse to compile. With `PhantomData`, the parameter exists at the type level,
letting you write impl blocks restricted to specific `J` types.

You will see this pattern constantly in type-state code (see Doc 03).

---

## Turbofish: Telling the Compiler What You Mean 🟢

Sometimes Rust cannot infer the type you want and you need to be explicit.
The turbofish syntax `::<Type>` does this:

```rust
// Without turbofish: ambiguous
let id = "550e8400-e29b-41d4-a716-446655440000".parse()?;
// The compiler does not know what type to parse into

// With turbofish: clear
let id = "550e8400-e29b-41d4-a716-446655440000".parse::<Uuid>()?;

// Or annotate the binding
let id: Uuid = "550e8400-e29b-41d4-a716-446655440000".parse()?;

// With generic functions
fn create<T: Default>() -> T { T::default() }
let val = create::<String>();
```

In taskforge you will see turbofish when deserializing job payloads from JSON:

```rust
let job = serde_json::from_value::<SendEmailJob>(payload)?;
```

---

## How It Breaks

**Monomorphization explosion.** If you have a generic function used with 50 different types, you get 50 compiled copies. Binary size grows. Compile time grows. Fix: use `Box<dyn Trait>` for cases where monomorphization cost outweighs the performance benefit.

**Turbofish ambiguity.** `parse::<i32>()` is fine, but complex nested turbofish like `collect::<HashMap<String, Vec<i32>>>()` is hard to read and easy to get wrong.

**Generic bounds that are too tight.** `fn process<T: Clone + Debug + PartialEq + Hash>(x: T)` — you've restricted the callers to types that implement all four. Often you only actually need one of them.

**Const generics limitations.** You can use const generics for sizes, but you can't compute them (no arithmetic) in current stable Rust.

---

## Common Mistakes

**1. Adding bounds to the struct when the bounds belong on the impl.**

```rust
// Wrong — forces EVERY use of the struct to satisfy all bounds
struct Cache<K: Eq + Hash + Clone, V: Clone> { ... }

// Right — struct is simple, bounds live where they're needed
struct Cache<K, V> { ... }

impl<K: Eq + Hash + Clone, V: Clone> Cache<K, V> { ... }
```

**2. Confusing associated types with generic parameters.** If you write
`trait Processor<Output>` when there is clearly one output type per implementor,
the compiler cannot tell which `Processor<_>` you mean at call sites without extra
help. Use `type Output;` instead.

**3. Forgetting PhantomData variance.** `PhantomData<T>` is covariant over T.
If you need invariance (common for mutable state), use `PhantomData<fn(T) -> T>`.
This is an advanced concern — just be aware it exists.

**4. Benchmarking generic code and seeing surprising performance.** Remember that
each instantiation is compiled separately, which means LLVM can make different
optimization decisions for each one. Your benchmark for `Worker<SendEmailJob>` says
nothing about `Worker<ResizeImageJob>`.

---

## Where-Clause vs Inline Bounds

Both syntaxes do the same thing — use whichever is more readable:

```rust
// Inline bounds — fine for 1-2 constraints
fn process<J: Job + Send + 'static>(job: J) { ... }

// Where-clause — better when constraints get complex
fn process<J>(job: J)
where
    J: Job + Send + 'static,
    J::Output: Debug + Send,
{
    ...
}
```

The rule: inline when it fits on one line. Where-clause when you have multiple type parameters each with multiple bounds.

## Higher-Ranked Trait Bounds (HRTBs)

You'll encounter HRTBs when a generic function needs to work with a closure that borrows something for *any* lifetime:

```rust
// This doesn't compile:
fn apply<F: Fn(&str) -> usize>(f: F, s: &str) -> usize {
    f(s)
}

// This works — HRTB: "F must implement Fn for any lifetime 'a"
fn apply<F>(f: F, s: &str) -> usize
where
    F: for<'a> Fn(&'a str) -> usize,
{
    f(s)
}
```

The `for<'a>` syntax means "for all lifetimes `'a`." In practice you rarely write this manually — the compiler infers it for closures. You'll see it in compiler errors when a closure doesn't satisfy a bound on a trait object. When that happens, the fix is usually to make the closure own its data instead of borrowing.

## Sealed Traits: Preventing External Implementors

A sealed trait is one you design specifically so that crate users can call its methods but cannot implement it themselves. The pattern uses a private supersupertrait:

```rust
// In your crate:
mod private {
    pub trait Sealed {}  // pub so it compiles, but the mod is private
}

pub trait JobKind: private::Sealed {
    fn kind_name(&self) -> &str;
}

// Only types in THIS crate can implement JobKind
// because only this crate can implement private::Sealed
pub struct EmailJob;
pub struct ResizeJob;

impl private::Sealed for EmailJob {}
impl private::Sealed for ResizeJob {}

impl JobKind for EmailJob {
    fn kind_name(&self) -> &str { "email" }
}
impl JobKind for ResizeJob {
    fn kind_name(&self) -> &str { "resize" }
}
```

Users can call `.kind_name()` and use `JobKind` as a bound, but they cannot add their own implementor. Use sealed traits when you need the flexibility of a trait interface internally but you want to maintain strict control over what the set of types is.

## Blanket Impls and the Orphan Rule

A blanket impl applies a trait to every type that satisfies some bound:

```rust
// The standard library does this:
impl<T: Display> ToString for T {
    fn to_string(&self) -> String { format!("{self}") }
}
// Now ANY type that implements Display automatically gets to_string()
```

The orphan rule prevents conflicts: you can only implement a trait for a type if either the trait OR the type is defined in your crate. You cannot implement `Display` for `Vec<T>` in your crate — both `Display` and `Vec` are foreign. The newtype pattern solves this:

```rust
// You own this type — you can implement any trait for it
struct JobList(Vec<Job>);

impl Display for JobList {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        // custom formatting
    }
}
```
