# Ownership: The Concept That Changes Everything

🔴 Challenging — but foundational. Everything else in Rust depends on this. Take your time.

If you learn one thing from this course, it's ownership. Everything else — error handling, threads, even the borrow checker errors you'll fight — traces back to ownership. Get this right and the rest falls into place.

---

## Stack vs Heap (You Already Know This)

As a C developer, you understand this:

```c
int x = 42;          // stack: fast, automatically freed when function returns
char *s = malloc(20); // heap: slow(er), you must call free()
free(s);
```

Stack allocation is fast and automatic. Heap allocation is flexible but requires manual management. In C, forgetting `free` leaks memory. Calling `free` twice corrupts the allocator. Using memory after `free` is undefined behavior.

Rust has the same stack/heap split. The difference is what it does about it.

---

## The Three Rules of Ownership

Every value in Rust follows these rules:

```
1. Every value has exactly one owner.
2. When the owner goes out of scope, the value is dropped (freed).
3. Ownership can be transferred, but there's always exactly one owner at any time.
```

That's it. These three rules, enforced by the compiler at compile time, eliminate use-after-free, double-free, and memory leaks — all at once.

---

## Rule 1 in Action: Moves

In C, assignment copies bytes. In Rust, assignment of heap-owning values **transfers ownership**:

```rust
fn main() {
    let s1 = String::from("hello");  // s1 owns the heap allocation
    let s2 = s1;                     // ownership MOVED to s2. s1 is gone.
    
    // println!("{s1}");  // COMPILE ERROR: value used after move
    println!("{s2}");     // fine
}
// s2 goes out of scope here. The String is freed. Exactly once.
```

In C, `s2 = s1` would give you two pointers to the same memory. Then `free(s1); free(s2);` would be a double-free. Rust prevents this by making the original variable invalid after the move.

The heap data doesn't move — only the ownership does. The bytes stay where they are; the compiler just stops you from accessing them through the old name.

```
Before: let s2 = s1;
    Stack          Heap
    s1 -> [ ptr ]---> [ h e l l o ]

After: let s2 = s1;
    Stack          Heap
    s1 -> [MOVED]
    s2 -> [ ptr ]---> [ h e l l o ]
```

---

## Rule 2 in Action: Drop

When a variable goes out of scope, Rust automatically calls `drop` on it. For a `String`, `drop` frees the heap memory. You never call `free` in Rust.

```rust
fn main() {
    {
        let s = String::from("inner");
        // s is live here
    }  // s goes out of scope -> drop is called -> heap freed

    // s doesn't exist here
}  // end of main
```

This is RAII — the same concept as C++ destructors, but without the Rule of Five complexity. You don't write `drop` implementations unless you have a custom resource to release (file handles, sockets, etc.).

For the hex viewer, `File` implements `Drop` to close the file descriptor when it goes out of scope. You never call `fclose()` like you would in C. The OS handle is released when the `File` variable is dropped — at the end of the block, or when the owning struct is dropped.

---

## Primitive Types Don't Move — They Copy

For small, stack-only types (`i32`, `u64`, `bool`, `char`, `u16`, etc.), assignment makes a copy rather than a move. This is the `Copy` trait.

```rust
fn main() {
    let x: i32 = 42;
    let y = x;          // x is COPIED, not moved
    println!("{x}");    // x is still valid
    println!("{y}");
}
```

Why? Because copying a 4-byte integer is cheap. Moving it would add complexity for no benefit. `String` doesn't implement `Copy` because copying it would require a heap allocation — that's expensive and should be explicit.

---

## Ownership and Functions

Passing a value to a function moves (or copies) it:

```rust
fn consume(s: String) {
    println!("{s}");
}   // s dropped here; heap freed

fn main() {
    let s = String::from("hello");
    consume(s);          // s is moved into consume()
    // println!("{s}"); // COMPILE ERROR: s was moved
}
```

This means if you pass a `String` to a function, you lose access to it. You have three options:
1. Return it back from the function (awkward)
2. Clone it (explicit allocation)
3. Pass a reference instead (best)

---

## Borrowing: References Without Ownership Transfer

Instead of moving, you can **lend** a value with a reference. The borrower can use it but can't own it:

```rust
fn print_string(s: &String) {  // takes a reference, not ownership
    println!("{s}");
}   // s (the reference) goes out of scope, but the String is NOT dropped

fn main() {
    let s = String::from("hello");
    print_string(&s);   // lend s with &
    println!("{s}");    // s is still valid!
}
```

`&String` is an immutable reference — you can read through it but not modify the value. This is like `const char *` in C, but the compiler guarantees the pointed-to value is still alive.

---

## Mutable References

To let a function modify a value, pass a mutable reference with `&mut`:

```rust
fn append(s: &mut String) {
    s.push_str(" world");
}

fn main() {
    let mut s = String::from("hello");
    append(&mut s);
    println!("{s}");  // "hello world"
}
```

---

## The Borrow Checker Rules

Here's where the compiler enforces safety. At any given moment, for any given variable:

```
You can have EITHER:
  - Any number of immutable references (&T)
OR:
  - Exactly one mutable reference (&mut T)

Not both. Never more than one mutable.
```

In C terms: you can have as many const readers as you want, OR one mutable writer, but never a reader and writer simultaneously.

This rule eliminates data races (in single-threaded code, it prevents iterator invalidation and aliased mutations; in multi-threaded code, it prevents data races at the compiler level).

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    let r1 = &v;      // immutable borrow
    let r2 = &v;      // another immutable borrow — fine
    println!("{r1:?} {r2:?}");
    // r1 and r2 go out of scope here

    let r3 = &mut v;  // mutable borrow — fine because r1/r2 are gone
    r3.push(4);
}
```

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    let r1 = &v;      // immutable borrow
    let r3 = &mut v;  // COMPILE ERROR: cannot borrow `v` as mutable because it is also borrowed as immutable
}
```

---

## Clone When You Need Two Copies

If you genuinely need two independent copies of a heap value, call `.clone()`:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();   // explicit deep copy — new heap allocation
    
    println!("{s1}");  // both valid
    println!("{s2}");
}
```

Clone is explicit because it allocates memory. If you find yourself cloning a lot, step back and ask whether borrowing would work instead. Unnecessary cloning is a code smell in Rust.

---

## Reading Borrow Checker Errors

You'll fight the borrow checker. Here's how to read the errors:

**"use of moved value"** — you used a variable after moving it somewhere. Solution: use a reference instead of moving, or clone before moving.

**"cannot borrow as mutable because it is also borrowed as immutable"** — you have both a `&T` and a `&mut T` alive at the same time. Solution: let the immutable borrows go out of scope before taking a mutable one.

**"cannot borrow as mutable more than once"** — you have two `&mut T` borrows. You can only have one at a time. Solution: don't overlap them.

**"does not live long enough"** — you're returning a reference to something that goes out of scope before the caller can use it. Solution: return owned data instead of a reference, or restructure so the data lives long enough.

---

## Drop in the Hex Viewer Context

For our hex viewer, this means:

```rust
use std::fs::File;
use std::io::{self, BufReader, Read};

fn read_chunk(path: &str) -> io::Result<Vec<u8>> {
    let file = File::open(path)?;  // File owns the OS file descriptor
    let mut reader = BufReader::new(file);  // file is MOVED into BufReader
    // file no longer accessible by name here — BufReader owns it now
    let mut buf = vec![0u8; 16];
    let n = reader.read(&mut buf)?;
    buf.truncate(n);
    Ok(buf)
}  // reader goes out of scope -> Drop is called -> file descriptor closed
```

No `fclose()`. The `File` (inside `BufReader`) closes when `BufReader` goes out of scope. In C, forgetting `fclose` is a resource leak. In Rust, the compiler guarantees it runs — through the same `Drop` trait that closes sockets, releases mutexes, and frees memory. One mechanism for all resources.

---

## How It Breaks

The borrow checker is correct more often than you are. But there are genuinely confusing cases where it produces errors that aren't immediately obvious.

**Fighting the borrow checker on struct fields.**
You might expect this to work:
```rust
struct Scanner {
    host: String,
    results: Vec<u16>,
}

fn process(scanner: &mut Scanner) {
    let h = &scanner.host;           // borrow host
    scanner.results.push(80);        // COMPILE ERROR: borrow of `scanner` occurs here
    println!("{}", h);
}
```
The compiler sees `&mut scanner` (to push) and `&scanner.host` simultaneously — a mutable and immutable borrow of the same thing. Even though `host` and `results` are different fields, the compiler reasons about the whole struct, not individual fields, in some situations. The fix: drop the first borrow before taking the second, or restructure so you don't hold both at once.

**Non-lexical lifetimes help, but edge cases remain.**
Rust's borrow checker was improved significantly with "non-lexical lifetimes" — borrows now end at the last point they're used, not the end of the block. This resolves many confusing errors from older Rust. But edge cases with loops and conditionals can still produce errors that look wrong. If the compiler says a borrow lasts longer than you expect, check whether it's used inside a loop or match that keeps it alive.

**Rc cycles — two values pointing to each other, neither freed.**
`Rc<T>` (reference-counted pointer) frees a value when the count reaches zero. If two `Rc` values hold references to each other, neither count ever reaches zero. The values leak for the lifetime of the program. The compiler doesn't catch this. Use `Weak<T>` for back-references in graph or tree structures where cycles are possible.

**When clone is the right answer, not a workaround.**
"Avoid clone" is common advice that gets misapplied. Clone is the right answer when:
- You genuinely need two independent copies of data with independent lifetimes
- You're passing data to a thread that must own it (`move` closure requires ownership)
- You're building a new value from a borrowed one and returning it from a function

Clone is a workaround when:
- You're cloning to avoid restructuring code that should use references
- You're cloning inside a hot loop because fixing the borrow checker error feels hard
- You have ownership issues that the design could solve

The smell: if you clone and then immediately use the clone in the same place the original was, you probably need a reference, not a clone.

---

## Common Mistakes C Developers Make

**Trying to use a value after moving it.** This is the most common early borrow checker fight. The solution is usually to pass a reference (`&`) instead.

**Not understanding why `String` moves but `i32` copies.** Small `Copy` types (`i32`, `u16`, `bool`) are copied on assignment. Types with heap data (`String`, `Vec`) are moved. Once you internalize which types implement `Copy`, this becomes intuitive.

**Fighting the borrow checker instead of listening to it.** The borrow checker is often right. If you can't satisfy it, it usually means there's a design problem. Step back and think about ownership — who should own this data?

**Returning references to local variables.** You cannot return `&` to something that lives on the current stack frame. It'll be gone when the function returns. Return owned data instead.

**Cloning everything to avoid borrow issues.** Cloning works but hides the real issue. If you're cloning to get around the borrow checker, try restructuring to use references. Clones cost allocations.
