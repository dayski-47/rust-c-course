# Why Rust for C Developers

🟢 Easy reading — no code exercises here, just context-setting.

---

## Engineering Discipline: Requirements-First Thinking

Before you write a single line of code for any project in this course, answer these four questions on paper (or a text file):

1. **What are the inputs?** What enters the program — user-provided values, environment data, network data?
2. **What are the outputs?** What does the user see — on success, on partial success, on failure?
3. **What are the failure cases?** What can go wrong? Network unreachable, invalid host, port out of range, OS-level permission errors?
4. **What does "correct" look like?** How do you know it worked? What would a correct run look like from the outside?

This is requirements-first thinking. Software engineers who skip this step write code twice — once to figure out what they're building, and again to fix it. The compounding effect is real: a bug that takes 2 minutes to catch in requirements takes 2 hours to catch in code and 2 days to catch in production.

The hex viewer has very clear answers to all four questions. Write them down before you start. The PROJECT.md file has worked examples of those answers — read it alongside this doc.

---

You've been writing C. You know `malloc`, `free`, `socket`, `connect`. You've debugged segfaults at 2am. You know what undefined behavior means — not just the textbook definition, but the lived experience of your compiler doing something completely insane because you triggered it.

So why Rust?

This doc gives you the honest answer. Not the marketing pitch.

---

## The Real Problem with C

C isn't broken. It's honest. It does exactly what you tell it to, even when what you're telling it is "write 40 bytes into a 10-byte buffer" or "dereference this pointer I freed three lines ago."

That honesty is both C's greatest strength and its most persistent source of CVEs.

The numbers are well-documented: over 70% of security vulnerabilities in major codebases (Chrome, Windows, Linux, Android) come from memory safety bugs. Not logic errors. Not missing features. Memory bugs. Buffer overflows, use-after-free, dangling pointers, uninitialized reads.

Here's the catalog you probably know from painful experience:

**Buffer overflow** — C arrays don't know their own size. Once you pass a pointer to a function, the size information is gone.
```c
char buf[10];
strcpy(buf, "this string is too long");  // Works. Corrupts memory. May crash later.
```

**Use-after-free** — you freed it, then used it. Behavior is undefined, meaning the compiler is allowed to do anything at all.
```c
char *p = malloc(20);
free(p);
p[0] = 'a';  // Undefined behavior. May work. May crash. May silently corrupt.
```

**Dangling pointer** — returning the address of a stack variable. Classic.
```c
int *get_value() {
    int x = 42;
    return &x;  // x is gone when this function returns
}
```

**Uninitialized variable** — C lets you declare a variable and immediately read garbage from it.
```c
int x;
if (x > 0) { ... }  // x is whatever was on the stack. Undefined behavior.
```

**NULL dereference** — nothing checks for you.
```c
int *p = NULL;
*p = 5;  // SEGV. Compiler didn't warn you.
```

None of these are exotic bugs. They're the everyday bugs that have been plaguing C codebases for 50 years.

---

## What Rust Actually Does Differently

Rust doesn't just add guardrails on top of the same model. It changes the model.

The core insight: **make it impossible to express these bugs at compile time**, rather than catching them at runtime or relying on programmer discipline.

Here's what that looks like for each C problem above:

| C Problem | Rust's Approach |
|-----------|-----------------|
| Buffer overflow | Arrays and slices carry their length. Out-of-bounds access panics (defined behavior) instead of corrupting memory. |
| Use-after-free | The ownership system makes this a compile error. When memory is freed (at end of scope), you can't have any references to it. |
| Dangling pointer | The borrow checker tracks lifetimes. Returning a reference to a local variable is rejected at compile time. |
| Uninitialized variable | All variables must be initialized before use. The compiler enforces this. |
| NULL dereference | There are no null pointers in safe Rust. Nullable values use `Option<T>`, which forces you to handle the "missing" case explicitly. |
| Data races | Sharing mutable data across threads is a compile error unless you use the right synchronization primitives. |

These aren't runtime checks bolted on (well, bounds checking is at runtime, but it panics with defined behavior rather than silently corrupting). The structural bugs — dangling references, use-after-free, use-after-move, data races — are **compile-time errors**.

If your Rust code compiles, it doesn't have those bugs.

---

## What Rust Gives You That C Doesn't

**Memory safety without a garbage collector.** You get C's performance (zero-cost abstractions, no GC pauses, predictable memory layout) plus the safety properties that normally require a GC or extreme programmer discipline.

**A package manager that actually works.** You know how painful it is to depend on a C library. Find the source, hope it's packaged for your distro, deal with header paths, linker flags, version conflicts. In Rust, you add one line to a config file and `cargo` handles everything. We'll cover this properly in the next doc.

**Fearless concurrency.** In C, sharing data between threads means you manually ensure all your mutex disciplines are correct. The compiler doesn't help. In Rust, if you try to share mutable data across threads without proper synchronization, it won't compile. The data race is caught before you ever run the program.

**Errors as values.** No `errno`. No checking a return code and forgetting. Rust functions that can fail return `Result<T, E>`. If you don't handle the error, you get a compiler warning at minimum (and a panic if you call `.unwrap()` on a failed result). The error handling is visible in the type signature.

**Explicit mutability.** In C, everything is mutable unless you add `const`. In Rust it's the opposite: everything is immutable unless you add `mut`. This forces you to think about what actually needs to change.

---

## What Rust Costs

Let's be honest about the tradeoffs.

**The borrow checker will fight you at first.** When you're learning Rust, you'll spend time arguing with the compiler about ownership. Code that "obviously should work" won't compile, and the error messages — while increasingly good — will sometimes feel cryptic. This is temporary. Most developers say it clicks within a few weeks, and then the borrow checker feels like a helpful teammate rather than an obstacle.

**The learning curve is steeper than C's.** C is a small language. You can hold the whole thing in your head. Rust has more concepts: ownership, borrowing, lifetimes, traits, generics. You don't need to master all of them to be productive, but there's more to learn upfront.

**Compile times are slower than C.** Rust does a lot of work at compile time. Debug builds are fast enough for iteration, but they're not as fast as `cc -O0`.

**The `unsafe` escape hatch exists, but use it carefully.** When you genuinely need to do something the borrow checker won't allow (calling C FFI, writing a custom allocator, implementing low-level data structures), you can wrap code in `unsafe` blocks. This tells the compiler "I know what I'm doing, don't check this." But it puts the burden back on you. The goal is to keep unsafe blocks small, well-commented, and behind safe interfaces.

---

## The Pitch for This Course

We're building a hex dump viewer — a tool that reads any binary file and displays its bytes in hexadecimal alongside the ASCII representation, just like `xxd -C`. You already know what raw bytes look like at the C level — `fread`, `uint8_t`, byte offsets, endianness. That knowledge transfers directly.

What you'll learn is how Rust expresses those same concepts, and what the language gives you for free when you do it in Rust instead of C.

By the end of this section you'll have a working tool, and you'll have touched every major Rust concept that matters for systems programming: types, ownership, error handling, file I/O, slices, and iterators.

---

## Common Mistakes C Developers Make Early On

**Trying to think in pointers.** In Rust you don't manage raw pointers (in safe code). References (`&T`, `&mut T`) are the main tool. They look like pointers but the compiler guarantees they're always valid.

**Fighting immutability.** Coming from C, you'll want to mutate things freely. Rust makes you declare intent up front with `let mut`. This is a feature, not a restriction.

**Expecting implicit conversions.** In C, adding an `int` and a `long` just works. Rust never does implicit numeric conversions. You use `as` for explicit casts. This prevents a whole class of subtle truncation bugs.

**Ignoring compiler errors.** Rust compiler errors are often long and detailed. Read them. They frequently tell you exactly what to do.

**Calling `unwrap()` everywhere.** It's fine when learning. It's a panic waiting to happen in real code. We'll cover proper error handling in doc 06.
