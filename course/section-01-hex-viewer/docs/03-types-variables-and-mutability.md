# Types, Variables, and Mutability

🟢 Easy - but pay attention to strings. They're weirder than you expect.

Rust's type system is familiar in structure - integers, floats, booleans, characters - but different in some important ways. No implicit conversions. Immutable by default. And strings are genuinely more complicated than in C, for good reasons.

---

## Primitive Types

Here's the full map:

| Category | Types | C Equivalent |
|----------|-------|--------------|
| Signed integers | `i8`, `i16`, `i32`, `i64`, `i128`, `isize` | `int8_t`, `int16_t`, `int32_t`, `int64_t`, `intptr_t` |
| Unsigned integers | `u8`, `u16`, `u32`, `u64`, `u128`, `usize` | `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`, `size_t` |
| Floating point | `f32`, `f64` | `float`, `double` |
| Boolean | `bool` | `_Bool` / `bool` from `<stdbool.h>` |
| Character | `char` | **Not the same** - see below |

`isize` and `usize` are pointer-sized, so 64 bits on a 64-bit system. Use `usize` for indices and sizes, same as you'd use `size_t` in C.

`char` in Rust is a **Unicode scalar value** (4 bytes), not a single byte like in C. If you're storing a byte, use `u8`. If you're storing a Unicode character, use `char`.

You can use underscores as visual separators in numbers - `1_000_000` is the same as `1000000`. Handy for port numbers and timeouts.

---

## Declaring Variables with `let`

In C: `int x = 42;`

In Rust:

```rust
fn main() {
    let x: i32 = 42;   // explicit type
    let y = 42;         // type inferred as i32 by default
    println!("{x} {y}");
}
```

The `let` keyword declares a variable. The type annotation (`: i32`) is optional when Rust can infer it from context.

---

## Immutable by Default

This is probably the biggest shock coming from C.

**In C, everything is mutable unless you write `const`.**  
**In Rust, everything is immutable unless you write `mut`.**

```rust
fn main() {
    let x = 42;
    x = 43;  // COMPILE ERROR: cannot assign twice to immutable variable `x`
}
```

To make a variable mutable:

```rust
fn main() {
    let mut x = 42;
    x = 43;  // OK
    println!("{x}");
}
```

This feels annoying at first. It becomes useful quickly. It forces you to be explicit about what changes, which makes code easier to reason about and helps the borrow checker understand your intent.

---

## Type Inference

Rust's type inference is strong. You often don't need annotations:

```rust
fn main() {
    let port = 8080;          // inferred as i32
    let timeout_ms = 200u64;  // explicit suffix: u64
    let open = true;          // inferred as bool
    let ratio = 0.95;         // inferred as f64
}
```

When the compiler can't figure out the type (usually with generic containers), it'll tell you, and you add the annotation then.

---

## No Implicit Conversions - Use `as`

In C, you can mix types freely:
```c
int x = 42;
long y = x;   // implicit widening conversion, fine
float f = x;  // fine
```

In Rust, **no implicit numeric conversions exist**. You must cast explicitly with `as`:

```rust
fn main() {
    let x: i32 = 42;
    let y: i64 = x as i64;   // explicit widening
    let z: u8 = x as u8;     // explicit narrowing - truncates if needed
    let f: f64 = x as f64;   // explicit int-to-float

    println!("{x} {y} {z} {f}");
}
```

This prevents an entire class of subtle bugs where values silently truncate or sign-extend in unexpected ways. If you write `x as u8` and `x` is 300, you get 44 - but you asked for it explicitly.

---

## Strings: The Part That Trips Everyone Up

In C, a string is a `char *` pointing at a null-terminated sequence of bytes. Simple. Dangerous, but simple.

Rust has two string types, and understanding the difference is essential:

### `&str` - a string slice (borrowed, view into existing data)

- A reference to a sequence of UTF-8 bytes you don't own
- Has a pointer and a length baked in (no null terminator needed)
- Cannot be resized
- Like a `const char *` with a length attached

```rust
let s: &str = "hello, world";  // string literal - lives in the binary
```

### `String` - an owned, heap-allocated string

- You own this memory
- Can be grown, shrunk, modified
- Like a `char *` you got from `malloc` and are responsible for `free`ing - except Rust frees it automatically when it goes out of scope

```rust
let s: String = String::from("hello, world");
let s2 = "hello".to_string();   // same thing, different syntax
```

### When to use which

```
&str  →  reading/viewing string data (passing to functions, string literals)
String →  owning/building/modifying string data
```

You'll see function parameters that take `&str` rather than `String`. That's intentional - a function that only reads a string shouldn't require ownership of it. You can pass a `&String` where a `&str` is expected (Rust auto-converts).

```rust
fn greet(name: &str) {
    println!("Hello, {name}!");
}

fn main() {
    let owned = String::from("Alice");
    greet(&owned);   // &String coerces to &str automatically
    greet("Bob");    // string literal is &str already
}
```

For the hex viewer, you'll mostly deal with `&str` for the file path argument and `String` when you need to build formatted output lines dynamically.

---

## Tuples

Group a fixed number of values of different types:

```rust
fn main() {
    let pair: (u16, bool) = (443, true);   // port number and whether it's open
    let port = pair.0;
    let is_open = pair.1;
    println!("Port {port} open: {is_open}");
}
```

You can also destructure:

```rust
let (port, is_open) = (443, true);
```

Tuples are useful for returning multiple values from a function - something you'd do with out-parameters in C.

---

## Arrays

Fixed-size, stack-allocated, all same type. Syntax: `[type; length]`.

```rust
fn main() {
    let ports: [u16; 4] = [22, 80, 443, 8080];
    println!("First port: {}", ports[0]);
    println!("Array length: {}", ports.len());

    // Out-of-bounds at compile time is an error
    // Out-of-bounds at runtime is a panic (not UB like in C)
}
```

For variable-length collections, use `Vec<T>` (the growable equivalent of a dynamically-allocated C array). In the hex viewer, you'll use `Vec<u8>` to hold the raw bytes read from the file.

---

## Shadowing

Rust lets you re-declare a variable with the same name. This is called shadowing:

```rust
fn main() {
    let port = "8080";            // &str
    let port = port.parse::<u16>().unwrap();  // shadow with u16
    println!("Port: {port}");
}
```

Shadowing is different from mutation - you're creating a new variable, not changing the old one. The new variable can even have a different type, which is useful when transforming values through a pipeline. This is idiomatic Rust; don't confuse it with reassignment.

---

## How It Breaks

Understanding how a feature fails is as important as understanding how it works. These are the type-related failure modes that catch developers off guard.

**Integer overflow - u16 wraps in release mode.**
In debug mode, Rust panics on integer overflow:
```rust
let x: u16 = u16::MAX; // 65535
let y = x + 1;         // debug: panic. release: wraps to 0.
```
In release mode (`cargo build --release`), overflow wraps silently. For port numbers and counts in the scanner, this is unlikely to matter - but it's critical in any arithmetic where overflow is plausible. Use `saturating_add` to clamp at the max value, or `checked_add` to get `None` on overflow rather than wrapping:
```rust
let clamped = x.saturating_add(1);   // 65535, stays at max
let checked = x.checked_add(1);      // None
```

**Implicit conversion bugs - Rust has none, which is the point.**
In C, `int port = some_u16_value` silently works. In Rust, mixing types is a compile error. This feels annoying until you realize it prevents a whole class of bugs where a 32-bit value silently truncates into a 16-bit slot. The `as` cast is explicit truncation - use `u16::try_from(n)` when you want to know if the value fits:
```rust
let n: u32 = 70000;
let p = u16::try_from(n); // Err(TryFromIntError) - 70000 doesn't fit in u16
```

**Shadowing confusion - shadowing is not mutation.**
When you shadow a variable, you create a new binding. The old one isn't changed:
```rust
let x = 42u16;
let x = x as u32; // new x of type u32; old x is still u16 (just inaccessible)
```
This matters if you pass a reference to the original before shadowing - the reference still points to the original type and value. Shadowing is safe, but it can hide logic errors when you expect the original to update.

**`&str` pointing to dropped data.**
A `&str` is a borrowed view into existing bytes. If those bytes are owned by a `String` that gets dropped, the `&str` becomes invalid. Rust prevents this at compile time - the borrow checker won't let you hold a `&str` that outlives its source. But when you're fighting "does not live long enough" errors with string slices, this is usually why.

---

## Common Mistakes C Developers Make

**Trying `x = 43` without `let mut`.** You'll see this error constantly at first. Just add `mut` and move on.

**Treating `char` like a byte.** In C, `char` is a byte. In Rust, `char` is 4 bytes (a Unicode scalar value). If you need a raw byte, use `u8`.

**Mixing `String` and `&str` without understanding why.** When a function asks for `&str` and you have a `String`, pass `&my_string`. When you have a string literal, it's already `&str`.

**Assuming integer conversion is implicit.** You will try to pass a `u16` where a `usize` is expected, and the compiler will refuse. Use `as usize` explicitly.

**Using the wrong integer type for indexing.** Array and Vec indices must be `usize`. If you compute a port number as `u16`, you'll need `as usize` when using it as an index.
