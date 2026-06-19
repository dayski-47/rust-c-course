# Doc 01 — Ownership Deep Dive: Lifetimes

## Engineering Methodology: Design for Ownership Before Writing Code

When you design a concurrent system, the first question is never "how do I synchronize this?" — it's "who owns this data?"

In the chat server you're about to build, ask before writing: Who owns the list of connected clients? Who owns each client's TcpStream? Who owns the message being broadcast?

If you can answer "exactly one entity owns this, and it's clear which one," your code will be simpler. If ownership is unclear before you start, you'll end up with Arc<Mutex<Arc<Mutex<...>>>> nesting that signals a design problem.

For the chat server: the server owns the client list. Each client connection owns its socket. The broadcaster owns nothing — it receives messages and forwards them. This ownership map should be on paper before you touch the keyboard.

---

You already know the basic borrow rules from Section 1. Now we go deeper.

In Section 1 you learned: one mutable reference OR any number of immutable references, never both at the same time. The borrow checker enforced those rules and you probably fought it a few times. Now you're going to understand *why* it works the way it does, and you're going to learn about **lifetimes** — the mechanism underneath the borrow rules.

This is where many C developers feel genuinely frustrated. Stick with it. Once it clicks, you'll realize Rust is just forcing you to be explicit about something you were always tracking mentally in C, but the compiler wasn't checking.

---

## Why Lifetimes Exist

In C, every pointer is just a number — a memory address. The compiler trusts you to know whether the data at that address is still alive. You've probably seen what happens when you get it wrong:

```c
char* get_name(void) {
    char buf[32] = "Alice";
    return buf;  // BUG: buf is on the stack, dies when function returns
}
```

Rust's lifetime system is the compiler's way of tracking exactly this: **how long is the data behind this reference still alive?** A lifetime is not a duration in time — it's a region of code during which a reference is valid.

The golden rule: **lifetimes don't change how long data lives. They describe the relationship between references.** Adding a lifetime annotation to a function doesn't make memory last longer. It just tells the compiler which reference the output is connected to.

---

## When the Compiler Infers Lifetimes

Most of the time you don't write any lifetime annotations at all. The compiler applies three rules automatically (called **lifetime elision**). If those rules produce an unambiguous answer, no annotation is needed.

```rust
fn first_word(s: &str) -> &str {
    match s.find(' ') {
        Some(pos) => &s[..pos],
        None => s,
    }
}
```

This compiles without any `'a`. Here's why the compiler can figure it out:

**Rule 1:** Each input reference gets its own lifetime. One input `&str`, so one lifetime `'a`.

**Rule 2:** If there is exactly one input lifetime, all output references get that same lifetime.

After applying both rules the compiler sees `fn first_word<'a>(s: &'a str) -> &'a str`. It knows the returned reference borrows from `s`. Done — no annotation needed from you.

```
Rule 1: each input ref gets a lifetime
        fn f(&str, &str)  →  fn f<'a,'b>(&'a str, &'b str)

Rule 2: one input lifetime → outputs inherit it
        fn f(&str) → &str  →  fn f<'a>(&'a str) -> &'a str

Rule 3: &self or &mut self → outputs inherit that lifetime
        fn get(&self, &str) -> &str  →  output borrows from self
```

When elision fails — when the compiler cannot determine which input the output borrows from — it stops and asks you to annotate.

---

## When You Must Annotate 🔴

Two input references, no `&self`. The compiler can't guess which one the output comes from.

```rust
// Does NOT compile — the compiler doesn't know if the return borrows from a or b
// fn longest(a: &str, b: &str) -> &str { ... }

// With annotation — we're telling the compiler:
// "the output lives at most as long as the shorter of a and b"
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() >= b.len() { a } else { b }
}

fn main() {
    let s1 = String::from("long string");
    let result;
    {
        let s2 = String::from("xy");
        result = longest(s1.as_str(), s2.as_str());
        println!("{result}");  // OK — result used before s2 dies
    }
    // println!("{result}");  // ERROR — s2 is gone, result might point to it
}
```

Important: the `'a` annotation does NOT mean "both references must have the same lifetime." It means "the output lifetime is constrained by whichever input has the shorter scope." The compiler uses this to catch uses of `result` after one of the inputs has been dropped.

---

## Lifetimes in Structs 🟡

If a struct holds a reference, you must tell the compiler how long that reference needs to stay alive. Otherwise the struct could outlive the data it points to.

```rust
// This struct borrows a slice of text — it does NOT own the string
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");

    let first_sentence;
    {
        let i = novel.find('.').unwrap_or(novel.len());
        first_sentence = &novel[..i];
    }

    // This works because novel lives long enough
    let excerpt = ImportantExcerpt { part: first_sentence };
    println!("Excerpt: {}", excerpt.part);
}
```

The `'a` on `ImportantExcerpt<'a>` says: "an instance of this struct cannot outlive the `'a` reference it holds." If you tried to let `excerpt` live past `novel`, the compiler would stop you.

Impl blocks on structs with lifetimes look like this:

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce(&self, announcement: &str) -> &str {
        println!("Attention: {announcement}");
        self.part  // Rule 3 applies: output borrows from &self
    }
}
```

---

## The 'static Lifetime

`'static` means the reference is valid for the **entire program**. This is equivalent to a C string literal baked into the binary's read-only data section.

```rust
let s: &'static str = "hello world";
// Same as: static const char* s = "hello world"; in C
// The string lives in the binary, never gets freed
```

You'll also see `'static` in trait bounds, especially with threads:

```rust
// thread::spawn requires the closure to be 'static
// That means: you can't borrow local variables into a thread
// You must either move them in or use Arc to share them

let data = vec![1, 2, 3];
// thread::spawn(|| println!("{data:?}"));  // ERROR: borrows local data

// Fix: move ownership into the thread
thread::spawn(move || println!("{data:?}"));
```

This is why `Arc` exists — when you need to share data with a thread but don't want to move it, you put it in an `Arc` (Chapter 2).

---

## ASCII Scope Diagram

Here's what the borrow checker actually sees when it validates lifetimes:

```
fn main() {
    let s1 = String::from("hello");    // 'a begins
    |
    |   let result;
    |   {
    |       let s2 = String::from("world"); // 'b begins
    |       |
    |       result = longest(s1.as_str(), s2.as_str());
    |       println!("{result}");   // OK — both 'a and 'b alive here
    |       |
    |   }   // 'b ends — s2 dropped here
    |
    |   // println!("{result}");  // ERROR — result might borrow from 'b (s2)
    |                             // but 'b is over
    |
}   // 'a ends — s1 dropped here
```

The compiler draws a similar picture internally. The lifetime annotation `'a` on `longest` is what lets it draw the connection between the output and the inputs, so it knows which scope boundary matters.

---

## Common Borrow Checker Errors and What They Mean

**"borrowed value does not live long enough"**
You're returning a reference to something that will be dropped before the caller can use it. Usually: you created a value inside a function and tried to return a reference to it.

```rust
fn bad() -> &str {
    let s = String::from("oops");
    &s  // s is dropped when bad() returns — this is the dangling pointer problem
}
```

Fix: return an owned `String`, or take a reference to something the caller owns.

**"cannot borrow as mutable because it is also borrowed as immutable"**
You have an immutable borrow active and tried to take a mutable borrow. Common in loops.

**"lifetime may not live long enough"**
A struct or return value requires a longer lifetime than you've guaranteed.

---

## Common Mistakes

**Trying to return a reference to a local variable.** This is the #1 lifetime mistake. In C this compiles silently and produces undefined behavior. In Rust it's a compile error. Return an owned value (`String` instead of `&str`, `Vec` instead of `&[T]`) when you need the function to produce new data.

**Adding lifetime annotations to everything hoping they'll fix errors.** Annotations don't extend lifetimes. They describe relationships. If the underlying lifetime structure is broken, annotations won't save you — you need to restructure ownership.

**Confusing `'static` with "long enough."** `'static` means "valid for the entire program." It's a specific, strong requirement — not a way to make a reference live longer.

**Fighting the struct lifetime annotation.** If your struct holds a `&str` field and you forget `<'a>`, the compiler error can seem cryptic. Remember: any reference in a struct requires you to declare a lifetime parameter on the struct.

---

## How It Breaks

Real patterns that cause lifetime pain in practice:

**Lifetime annotations that are too restrictive.** You annotated `'a` where you didn't need to, now the function can't be used in the way you intended. Adding `'a` to a function signature constrains the caller, not just the implementation. If you over-annotate, callers may find they cannot construct the required lifetimes and the function becomes unusable even when your logic is sound.

**Returning references from methods when you should return owned values.** This is the #1 beginner mistake. The instinct is to avoid allocation. The right fix is to return an owned `String` instead of `&str`, a `Vec<T>` instead of `&[T]`. References make sense when you're lending data that already exists. They don't make sense when you're creating new data inside a function.

**Self-referential structs.** A struct that holds a reference to one of its own fields. Lifetimes cannot express this relationship — the borrow checker cannot prove it is safe because moving the struct would invalidate the internal pointer. The solutions are: redesign to avoid self-reference, use `Pin` to prevent the struct from moving, or use owned data with indices instead of references.

**The borrow checker saying "doesn't live long enough" in async code.** The real problem is that async state machines are structs, and those structs must own all the data they carry across `.await` points. References don't work because the executor might run your task on a different thread at a different time. The fix is always the same: move owned data into the async block, or use `Arc` to share ownership. You cannot borrow your way through an async boundary.
