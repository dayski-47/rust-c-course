# 01 — Unsafe Rust: The Escape Hatch

> **Difficulty:** 🟡 think about it / 🔴 challenging for the ring buffer  
> **You'll learn:** The five specific capabilities `unsafe` unlocks, raw pointer mechanics,
> the contract you sign when you write `unsafe`, and how to wrap danger inside a safe API.

---

## Engineering Methodology: Correctness Before Performance + Safety Invariant Design

Systems engineers face a constant temptation: reach for `unsafe` to make things faster. Resist this.

The discipline is: correctness first, performance second, `unsafe` only when necessary.

Correctness means: all written data can be read back. The system doesn't corrupt data on crash. The API cannot be misused to produce undefined behavior.

You prove correctness before optimizing. Once you have a safe, correct implementation, you measure whether it's fast enough. Usually it is. When it isn't, you profile to find the bottleneck, and you reach for `unsafe` only at that specific bottleneck — wrapped in a safe API so the caller never touches the danger.

For the storage engine, the invariants you must define before writing any unsafe code:
- The file is always in a valid state that can be parsed. (Never leave it half-written.)
- Every record's checksum matches its content. (Corrupted records are detectable.)
- The in-memory index always matches the on-disk data. (No phantom records in the index.)
- A mmap pointer is valid as long as the `Mmap` object is alive. (Never store raw pointers that outlive the `Mmap`.)

Document each safety invariant in a comment where the `unsafe` code lives. The comment should state what you're promising the compiler, so the next person who reads the code knows what they'd be breaking if they changed it.

---

## First: What `unsafe` Does NOT Do

Before anything else, clear this up. Unsafe Rust does **not** turn off the borrow checker.
It does **not** disable the type system. Lifetimes, ownership, `Result`, pattern exhaustion —
all of that still applies. The borrow checker is watching.

What `unsafe` does is unlock **five specific capabilities** that the compiler cannot prove
are correct on its own. It trusts you to prove them instead. That's the deal.

---

## The Five Unsafe Superpowers

```rust
// Each of the five superpowers, shown inline:

// 1. Dereference raw pointers
let x: i32 = 42;
let ptr: *const i32 = &x;
// SAFETY: ptr was created from a valid reference, still in scope
let val = unsafe { *ptr };

// 2. Call unsafe functions (including extern "C" functions)
unsafe fn dangerous() { /* ... */ }
// SAFETY: dangerous() has no preconditions; this call is sound
unsafe { dangerous() };

// 3. Access or modify mutable static variables
static mut COUNTER: u64 = 0;
// SAFETY: single-threaded context, no concurrent access
unsafe { COUNTER += 1; }

// 4. Implement unsafe traits
// unsafe impl Send for MyType {}   // You declare this type is safe to send across threads

// 5. Access fields of a union
union IntOrFloat { i: i32, f: f32 }
let u = IntOrFloat { i: 0x3F800000 };
// SAFETY: we know we wrote an i32 that represents 1.0f32 in IEEE 754
let f = unsafe { u.f };
```

Nothing else changes. You still can't violate borrow rules in an `unsafe` block.

---

## Raw Pointers: `*const T` and `*mut T`

Coming from C, raw pointers look familiar. But they work differently from C's
perspective of "everything is just a number."

**Creating** raw pointers is safe. **Dereferencing** them is not.

```rust
fn create_pointers() {
    let mut value: i32 = 10;

    // Create from a reference — safe
    let const_ptr: *const i32 = &value;
    let mut_ptr: *mut i32 = &mut value;

    // Create from a heap allocation — you now own the raw pointer
    let boxed: Box<i32> = Box::new(42);
    let heap_ptr: *mut i32 = Box::into_raw(boxed);
    // boxed is gone. The heap allocation is YOURS now.
    // If you never call Box::from_raw(heap_ptr), you leak memory.

    // Dereference — requires unsafe
    // SAFETY: const_ptr came from a valid &i32, which is still in scope
    let v = unsafe { *const_ptr };
    assert_eq!(v, 10);

    // Write through a mutable pointer
    // SAFETY: mut_ptr came from a valid &mut i32, no other references active
    unsafe { *mut_ptr = 20; }
    assert_eq!(value, 20);

    // Clean up the heap pointer — construct Box back, let it drop
    // SAFETY: heap_ptr was produced by Box::into_raw above; called exactly once
    let _recovered = unsafe { Box::from_raw(heap_ptr) };
    // _recovered drops here → memory freed
}
```

Notice the pattern with `Box::into_raw` / `Box::from_raw`. This is the standard
idiom for passing heap-allocated Rust values across an FFI boundary. You leak the
Box, pass the raw pointer to C, and reconstruct the Box to free it when C calls back.

**Pointer arithmetic** works with `.add(n)` and `.offset(n)`:

```rust
fn pointer_arithmetic() {
    let data = [10u8, 20, 30, 40, 50];
    let ptr = data.as_ptr();

    // SAFETY: data has 5 elements; we access indices 0..4 only
    unsafe {
        assert_eq!(*ptr.add(0), 10);
        assert_eq!(*ptr.add(2), 30);
        assert_eq!(*ptr.add(4), 50);
    }

    // What you MUST NOT do:
    // *ptr.add(5)  — one past the end of the array is UB
    // *ptr.add(100) — obviously UB
    // *ptr.sub(1)   — before the start is UB
}
```

The C equivalent `ptr[5]` is UB there too, but at least in C there's no way to
express "this pointer is valid for exactly these offsets." In Rust you can, and you
should — in your `SAFETY:` comments.

---

## The Unsafe Contract: `// SAFETY:` Comments

When you write `unsafe`, you are making a promise the compiler cannot verify.
You must **document** that promise. Every `unsafe` block in idiomatic Rust has a
`// SAFETY:` comment explaining why the operation is valid.

```rust
// SAFETY: ptr is derived from `data.as_ptr()`, the slice has `n` elements,
//         and we only access indices 0..n. The slice is still live at this point.
let val = unsafe { *ptr.add(index) };
```

This is not optional ceremony. It is how unsafe code gets reviewed, how it gets
audited, and how you catch your own mistakes six months later. The `SAFETY:` format
is recognized by tooling and by convention across the Rust ecosystem.

What belongs in a `SAFETY:` comment:
- What invariant makes this operation valid
- Where that invariant comes from (caller, struct internal state, etc.)
- Any aliasing or lifetime considerations

---

## `unsafe fn` vs `unsafe {}` Block

These are different things. Do not confuse them.

```rust
// unsafe fn: the FUNCTION ITSELF is unsafe to call.
// The caller must uphold some precondition.
// Inside the body, you don't need an inner unsafe block for the fn's own contract.
unsafe fn read_at(ptr: *const u8, offset: usize) -> u8 {
    // SAFETY: caller guarantees ptr+offset is valid (that's the fn's precondition)
    *ptr.add(offset)
}

// A normal fn with an unsafe BLOCK inside:
// The function is safe to call. The block isolates the dangerous operation.
fn first_byte(data: &[u8]) -> Option<u8> {
    if data.is_empty() {
        return None;
    }
    // SAFETY: data is non-empty; data.as_ptr() + 0 is valid
    let byte = unsafe { *data.as_ptr() };
    Some(byte)
}

fn caller() {
    // first_byte is safe — no unsafe block required here
    let b = first_byte(&[1, 2, 3]);

    // read_at is unsafe — must use unsafe block
    let arr = [10u8, 20, 30];
    // SAFETY: arr has 3 elements; offset=1 is in bounds
    let v = unsafe { read_at(arr.as_ptr(), 1) };
}
```

Use `unsafe fn` when **callers** must uphold preconditions to make the call valid.
Use `unsafe {}` when **you** are doing something unsafe internally but can guarantee
correctness before exposing a safe API.

---

## The Safe Abstraction Principle

The most important pattern in unsafe Rust: **wrap unsafe code in a safe API**.
Users of your type should never need to write `unsafe`. All the danger is hidden
inside.

`Vec<T>` is the canonical example. Internally it uses `unsafe` pointer arithmetic
for every push, pop, and grow. Its public API is entirely safe. You cannot cause
UB by using `Vec` correctly.

Here is a simple example — a ring buffer:

```rust
use std::mem::MaybeUninit;

/// A fixed-capacity ring buffer. All public methods are safe.
/// Internals use MaybeUninit to avoid zero-initializing unwritten slots.
pub struct RingBuffer<T, const N: usize> {
    data: [MaybeUninit<T>; N],
    head: usize,  // next write position
    tail: usize,  // next read position
    len: usize,
}

impl<T, const N: usize> RingBuffer<T, N> {
    pub fn new() -> Self {
        RingBuffer {
            data: [const { MaybeUninit::uninit() }; N],
            head: 0,
            tail: 0,
            len: 0,
        }
    }

    pub fn push(&mut self, value: T) {
        if self.len == N {
            // Drop the element we're about to overwrite
            // SAFETY: slot at head has been initialized — we're full
            unsafe { self.data[self.head].assume_init_drop(); }
            self.tail = (self.tail + 1) % N;
        } else {
            self.len += 1;
        }
        self.data[self.head] = MaybeUninit::new(value);
        self.head = (self.head + 1) % N;
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            return None;
        }
        // SAFETY: data[tail] was initialized by push(); len > 0 guarantees this
        let value = unsafe { self.data[self.tail].assume_init_read() };
        self.tail = (self.tail + 1) % N;
        self.len -= 1;
        Some(value)
    }

    pub fn len(&self) -> usize { self.len }
    pub fn is_empty(&self) -> bool { self.len == 0 }
}

impl<T, const N: usize> Drop for RingBuffer<T, N> {
    fn drop(&mut self) {
        // SAFETY: slots in [tail..tail+len) (wrapping) are initialized
        for i in 0..self.len {
            unsafe { self.data[(self.tail + i) % N].assume_init_drop(); }
        }
    }
}
```

Three rules of sound unsafe code:
1. **Document invariants** — every `SAFETY:` comment explains why the operation is valid
2. **Encapsulate** — the unsafe is inside a safe API; users can't trigger UB
3. **Minimize** — only the smallest possible block is `unsafe`

---

## Mutable Statics: Global State and Why It's Unsafe

```rust
static mut INSTANCE_COUNT: u32 = 0;

fn increment() {
    // SAFETY: single-threaded; no concurrent access possible in this program
    unsafe { INSTANCE_COUNT += 1; }
}
```

Mutable statics are unsafe because two threads could access them simultaneously —
that is a data race, and data races are UB. The compiler cannot verify that
you have taken appropriate precautions. That is why every access requires `unsafe`.

The idiomatic solution is `AtomicU32` or wrapping with `Mutex`. But when you
absolutely need a global writable variable (common in embedded or FFI initialization),
mutable statics are how you do it. Just document why concurrent access is not possible,
or use the right atomic type.

---

## When to Actually Use `unsafe`

After six sections of idiomatic Rust, here is when `unsafe` is legitimately
justified:

| Situation | Why unsafe is needed |
|-----------|---------------------|
| Calling C libraries (FFI) | C functions cannot give Rust safety guarantees |
| Memory-mapped I/O and files | The OS can change the mapping under you |
| Custom allocators | You are managing memory yourself |
| Zero-copy parsing of binary data | Casting raw bytes to typed structs |
| SIMD intrinsics | Platform-specific instructions with specific alignment requirements |
| Building data structures like `Vec`, `HashMap`, `LinkedList` | Pointer arithmetic that the type system cannot verify |
| `no_std` environments | No heap allocator, must manage memory explicitly |

Everything else in a normal application can and should be written in safe Rust.
If you find yourself reaching for `unsafe` to make the borrow checker happy,
stop. Restructure the ownership instead.

---

## How It Breaks

**Misaligned access.** Reading a `u32` from a byte slice that's not 4-byte aligned is UB on some architectures. `bytemuck` handles this correctly. Manual `transmute` does not.

**Use-after-free with Box::from_raw.** You convert a raw pointer back to a `Box` and then it drops. If you have any other raw pointers to the same memory, they're now dangling.

**Aliasing violation.** Rust's aliasing rules say you can't have a mutable reference and any other reference to the same memory simultaneously. `unsafe` code that creates `&T` and `&mut T` from the same pointer violates this even if no concurrent access happens.

**UB that "works" in debug but breaks in release.** The optimizer is allowed to assume UB doesn't happen. If your UB involves reading uninitialized memory, the optimizer might constant-fold the read to zero in debug mode but do something entirely different in release mode. This is the scary part of UB.

---

## Common Mistakes

**Forgetting `Box::from_raw` after `Box::into_raw`.** You leak the allocation.
Use RAII or a guard type to ensure cleanup.

**Creating `*mut T` from an `&T`.** A shared reference says "I promise nothing
else mutates this." Casting it to `*mut T` and then mutating through it violates
that promise. Use `UnsafeCell` if you need interior mutability.

**Holding a raw pointer across an operation that could move the data.** Pushing
to a `Vec` can reallocate and move all elements. Any raw pointer into the Vec
is dangling after a push. Miri will catch this.

**Writing `unsafe { ... }` with no `SAFETY:` comment.** Your future self and your
code reviewers cannot tell whether this is correct. Write the comment even if it
feels obvious.

**Using `std::mem::transmute` when bytemuck or zerocopy would work.** `transmute`
is the most dangerous tool in the box. If the types differ in size, it is immediate
UB. `bytemuck::cast` checks size and alignment at compile time.

---

## Safety Invariants to Maintain

When writing unsafe code in this section's storage engine, you must uphold:

- **Pointer validity:** A raw pointer you dereference must point to an allocated,
  initialized value of the correct type and alignment.
- **Exclusive access:** When writing through `*mut T`, no shared `&T` references
  must exist simultaneously.
- **Lifetime:** The allocation must still be live when you dereference the pointer.
  Do not hold pointers across operations that could drop or reallocate the backing store.
- **Correct size:** When casting raw bytes to a type (`transmute`, `bytemuck`),
  the byte count must equal `std::mem::size_of::<T>()`.
- **Alignment:** Reading a `u32` at an odd byte offset is UB on most architectures.
  Ensure your byte-to-struct casts meet alignment requirements, or use
  `read_unaligned()` when you cannot guarantee alignment.
