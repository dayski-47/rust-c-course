# Doc 08 — Your First Macro

🟡 Think about it — you've used macros since line 1 (`println!`). Now you'll understand what they are and write your own.

Every Rust program you've written so far has used macros: `println!`, `vec!`, `assert_eq!`, `format!`. The `!` suffix is not decoration — it's how Rust distinguishes macros from function calls. This doc explains what macros are, when to reach for them over functions, and walks you through writing the `hex_line!` macro that the hexview project uses to format its output.

---

## Why Macros Exist

Functions and generics handle most code reuse in Rust. Macros fill the gaps where the type system can't reach:

| Need | Function? | Macro? | Reason |
|------|-----------|--------|--------|
| Accept a variable number of arguments | ❌ Rust has no variadic functions | ✅ | Macros operate on syntax tokens, not typed values |
| Generate repetitive `impl` blocks | ❌ Not without generics | ✅ | Macros produce code at compile time |
| Log with file name and line number | ❌ A function doesn't know its call site | ✅ `dbg!`, `panic!` | Macros expand at the call site |
| Embed a file at compile time | ❌ | ✅ `include_bytes!` | Runs before compilation, not at runtime |

**The rule:** if a function can do it, use a function — it has better error messages, IDE support, and is easier to read. Reach for macros only when functions can't solve the problem.

---

## C Preprocessor vs Rust Macros

If you're coming from C, your instinct is that macros are textual substitution:

```c
// C preprocessor — dumb text replacement
#define MAX(a, b) ((a) > (b) ? (a) : (b))

int x = 5;
int y = MAX(x++, 3);  // UB: x is incremented twice because (a) expands twice!
```

C macros have a fundamental flaw: they operate on raw text before the compiler even sees it. The compiler gets the expanded text, not the macro. This causes:
- Double-evaluation bugs (like `x++` above)
- Name collisions — a macro variable leaks into the caller's scope
- Debugging is impossible — error messages point at the expanded text

Rust macros are completely different. They operate on the **syntax tree** — the compiler parses your code first, then macros transform it. This means:
- No textual substitution — `$x:expr` matches a full, well-formed expression
- Variables created inside a macro don't collide with the caller's variables (**hygiene**)
- Error messages point at the original macro call, not the expansion

```rust
// Rust macro — operates on syntax, not text
macro_rules! max {
    ($a:expr, $b:expr) => {
        if $a > $b { $a } else { $b }
    };
}

let x = 5;
let y = max!(x, 3);  // Safe — $a is evaluated once
```

---

## Declarative Macros with `macro_rules!`

The most common kind. They use **pattern matching on syntax** — similar to how `match` matches on values.

```rust
macro_rules! say_hello {
    () => {                     // Pattern: no arguments
        println!("Hello!");     // Expansion: what to produce
    };
}

fn main() {
    say_hello!();  // Expands to: println!("Hello!");
}
```

### Fragment Specifiers

When a pattern contains arguments, you specify what kind of syntax each argument can be:

| Specifier | Matches | Example |
|-----------|---------|---------|
| `$x:expr` | Any expression | `42`, `a + b`, `foo()`, `vec![1,2,3]` |
| `$x:ty` | A type | `i32`, `Vec<u8>`, `&str` |
| `$x:ident` | An identifier | `my_var`, `SomeStruct` |
| `$x:pat` | A pattern | `Some(x)`, `(a, b)`, `_` |
| `$x:literal` | A literal value | `42`, `"hello"`, `true` |
| `$x:tt` | Any single token tree | Anything — the wildcard |

```rust
macro_rules! greet {
    ()              => { println!("Hello, world!"); };
    ($name:expr)    => { println!("Hello, {}!", $name); };
    ($n:expr times) => { for _ in 0..$n { println!("Hello!"); } };
}

fn main() {
    greet!();                // "Hello, world!"
    greet!("Rust");          // "Hello, Rust!"
    greet!(3 times);         // "Hello!" three times
}
```

Notice the third arm: `$n:expr times` matches an expression followed by the literal identifier `times`. Macros can create mini-DSLs that look nothing like normal Rust — this is powerful but easy to abuse. Keep macro syntax close to normal Rust.

### Repetition

This is what makes macros worth learning. Macros can match and repeat patterns — something no function can do:

```rust
macro_rules! print_all {
    // $( ... ),*  means: match zero or more of this, separated by commas
    ( $( $item:expr ),* ) => {
        // $( ... )*  means: repeat the body once per matched item
        $( println!("{}", $item); )*
    };
}

fn main() {
    print_all!(1, "hello", 3.14, true);
    // Expands to:
    // println!("{}", 1);
    // println!("{}", "hello");
    // println!("{}", 3.14);
    // println!("{}", true);
}
```

| Repetition operator | Meaning |
|--------------------|---------|
| `$( ... )*` | Zero or more |
| `$( ... )+` | One or more (at least one required) |
| `$( ... )?` | Zero or one (optional) |
| `$( ... ),*` | Zero or more, comma-separated |
| `$( ... ),+` | One or more, comma-separated |
| `$(,)?` at the end | Allow optional trailing comma |

> This is exactly how `vec![]` is implemented. When you write `vec![1, 2, 3]`, the macro matches `$($x:expr),+`, then expands each `$x` into a push call.

---

## Hygiene: Why You Don't Need to Worry About Name Collisions

In C, this is a real problem:

```c
#define INIT_VAR(type) type temp = 0  // 'temp' might shadow the caller's 'temp'
```

In Rust, every identifier created inside a macro lives in the macro's own scope — it cannot shadow the caller's variables:

```rust
macro_rules! make_temp {
    () => {
        let temp = 99;  // This 'temp' is invisible outside the macro
    };
}

fn main() {
    let temp = 42;
    make_temp!();
    println!("{temp}");  // Prints 42, not 99
}
```

The macro's `temp` and the caller's `temp` are different variables, even though they share a name. You never need to prefix macro-internal variables with `__` or similar guards — hygiene handles it automatically.

---

## Standard Library Macros You Use Every Day

These are all macros, not functions. Understanding that changes how you read them:

| Macro | What it actually does |
|-------|----------------------|
| `println!("{}", x)` | Computes format at compile time, writes to stdout at runtime |
| `format!("{}", x)` | Same but allocates and returns a `String` |
| `eprintln!("{}", x)` | Same as `println!` but to stderr |
| `vec![1, 2, 3]` | Creates a `Vec` — expands to repeated `.push()` calls |
| `assert!(cond)` | Panics if `cond` is false — includes file/line in message |
| `assert_eq!(a, b)` | Panics with both values shown: `left: 5, right: 3` |
| `dbg!(expr)` | Prints `[file:line] expr = value` to stderr, returns the value |
| `include_bytes!("file")` | Embeds file as `&[u8]` at compile time |
| `todo!()` | `panic!("not yet implemented")` — marks incomplete code |
| `unreachable!()` | `panic!("unreachable")` — marks impossible branches |

### `dbg!` — insert-anywhere debugging

`dbg!` is unusually useful because it **returns the value it wraps**. You can insert it anywhere without changing program behavior:

```rust
fn format_hex_byte(byte: u8) -> String {
    format!("{:02x}", dbg!(byte))  // Prints the byte value while processing
}

fn main() {
    let bytes = [0xDE, 0xAD, 0xBE, 0xEF];
    for b in bytes {
        format_hex_byte(b);
    }
}
// Output:
// [src/main.rs:2] byte = 222
// [src/main.rs:2] byte = 173
// ... etc
```

`dbg!` prints to stderr, so it doesn't mix with your program's stdout output. Always remove `dbg!` calls before committing code.

---

## Format String Reference

`println!`, `format!`, `write!`, and `eprintln!` all use the same format string syntax. For hexview, you'll use this constantly:

```rust
let byte: u8 = 0xAB;
let offset: usize = 256;
let ascii_char = '·';

// Hex formatting — critical for hexview
println!("{byte:02x}");        // "ab"  — lowercase hex, zero-padded to 2 digits
println!("{byte:02X}");        // "AB"  — uppercase hex
println!("{byte:#04x}");       // "0xab" — with 0x prefix, padded to 4 total

// Integer formatting
println!("{offset:>8}");       // "     256" — right-aligned, width 8
println!("{offset:0>8}");      // "00000256" — zero-padded, width 8
println!("{offset:#010x}");    // "0x00000100" — hex with prefix, padded

// Multiple values
println!("{offset:08x}: {byte:02x}  {ascii_char}");
// "00000100: ab  ·"

// Debug vs Display
println!("{:?}", vec![1u8, 2, 3]);     // "[1, 2, 3]"
println!("{:#?}", vec![1u8, 2, 3]);    // Pretty-printed with newlines
```

> **For C developers:** This is a type-safe `printf`. The compiler checks that `{:02x}` is applied to an integer type. No `%d` vs `%u` bugs, no format string vulnerabilities.

---

## Writing `hex_line!` for the Hexview Project

The hexview project outputs lines like:

```
00000000: 7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  .ELF............
```

The format has three regions: the offset, the hex bytes (in two groups of 8), and the ASCII representation. Formatting each line consistently across the entire codebase is exactly what a macro is good at.

Here's how you might approach building `hex_line!`:

```rust
/// Format a 16-byte line for hexdump output.
///
/// Usage: hex_line!(offset, bytes_slice)
/// Output: "00000010: 61 62 63 ...  abc..."
macro_rules! hex_line {
    ($offset:expr, $bytes:expr) => {{
        let bytes: &[u8] = $bytes;

        // Build the hex section: "61 62 63 64  65 66 67 68"
        // Note: a space between the two groups of 8
        let hex_part: String = bytes
            .iter()
            .enumerate()
            .map(|(i, b)| {
                if i == 8 {
                    format!(" {:02x}", b)  // Extra space before second group
                } else {
                    format!("{:02x}", b)
                }
            })
            .collect::<Vec<_>>()
            .join(" ");

        // Pad hex to full width if fewer than 16 bytes (last line of file)
        let max_hex_width = 16 * 3 + 1;  // 16 bytes × "xx " + 1 space for group gap
        let hex_padded = format!("{hex_part:<width$}", width = max_hex_width);

        // Build the ASCII section: printable chars or '.' for non-printable
        let ascii_part: String = bytes
            .iter()
            .map(|&b| if (0x20..0x7e).contains(&b) { b as char } else { '.' })
            .collect();

        // Assemble the full line
        format!("{:08x}: {}  {}", $offset, hex_padded, ascii_part)
    }};
}

fn main() {
    let data = b"Hello, World!\x00\x01\x02";
    let line = hex_line!(0usize, data.as_ref());
    println!("{line}");
    // 00000000: 48 65 6c 6c 6f 2c 20 57  6f 72 6c 64 21 00 01 02  Hello, World!...
}
```

The double braces `{{ ... }}` in the expansion create a new block scope — this lets the macro define local variables (`bytes`, `hex_part`, etc.) without polluting the caller's scope.

### Why a macro instead of a function here?

Actually, you could write this as a function:

```rust
fn hex_line(offset: usize, bytes: &[u8]) -> String {
    // same body...
}
```

And for this specific case, a function is perfectly fine. The macro form becomes valuable when:
1. You want call-site information (`file!()`, `line!()`) in the output
2. You're generating different types based on arguments at compile time
3. You want the ergonomics of a specific call syntax

For the hexview project, start with a function. Graduate to a macro if you find a genuine need for compile-time behavior.

---

## Derive Macros: Code Generation for Free

You've already used derive macros in every section:

```rust
#[derive(Debug, Clone, PartialEq)]
struct HexViewConfig {
    offset: u64,
    length: Option<usize>,
    group_size: usize,
    uppercase: bool,
}
```

`#[derive(Debug)]` is a **procedural macro** that reads your struct's field names and types at compile time and generates a `Debug` implementation automatically. Without it, you'd write this by hand — every time you add a field:

```rust
// What #[derive(Debug)] generates (simplified):
impl std::fmt::Debug for HexViewConfig {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("HexViewConfig")
            .field("offset", &self.offset)
            .field("length", &self.length)
            .field("group_size", &self.group_size)
            .field("uppercase", &self.uppercase)
            .finish()
    }
}
```

### What to always derive

These are almost free in terms of runtime cost and you'll regret not having them:

| Derive | Enables | Derive it when |
|--------|---------|----------------|
| `Debug` | `{:?}` printing | Always — no reason not to |
| `Clone` | `.clone()` | Your type contains no file handles/sockets |
| `PartialEq` | `==` and `!=` | You need to compare values in tests |
| `Default` | `Type::default()` | Your type has a sensible "zero" state |

Don't derive `Copy` unless your type is small and stack-only (think: a single integer or small array). `Copy` means "assignment makes a copy" — fine for `u64`, wrong for `Vec<u8>`.

---

## Procedural Macros: The Full Picture (Conceptual)

`macro_rules!` is powerful but limited — it can only match syntax patterns. **Procedural macros** are full Rust programs that run at compile time and can do arbitrary computation on your code's syntax tree.

You won't write one in this section, but you'll use them constantly:

| Proc macro you use | What it does |
|--------------------|-------------|
| `#[derive(Serialize, Deserialize)]` from `serde` | Generates JSON encode/decode for your types |
| `#[derive(Error)]` from `thiserror` | Generates `Display` + `From` for error enums |
| `#[tokio::main]` | Rewrites `async fn main()` to set up the Tokio runtime |
| `#[instrument]` from `tracing` | Wraps your function to emit tracing spans |

In section 8 (Storage Engine), you'll write your own `#[derive(KvSerialize)]` proc macro. By then you'll know why — manual serialization code is repetitive and error-prone.

---

## How It Breaks

**Recursive macro expansion hitting limits.**
`macro_rules!` macros can call themselves recursively, but the compiler limits recursion depth to prevent infinite loops:
```rust
// This will hit "recursion limit reached" with a large input
macro_rules! count_recursive {
    () => { 0 };
    ($head:tt $($tail:tt)*) => { 1 + count_recursive!($($tail)*) };
}
```
The limit is 128 by default (or 32 in older Rust). For large inputs, use iteration or const generics instead.

**Ambiguous `$x:expr` matching causing parse errors.**
`expr` is greedy — it will consume the longest valid expression. If your macro expects `something, more` but the expression before the comma is itself complex (like a closure), the parser can get confused. The fix is usually to use `tt` for more flexibility, or to restructure your macro's syntax.

**Forgetting the `$(,)?` trailing comma.**
Users expect to write macros with trailing commas: `my_macro!(a, b, c,)`. Without `$(,)?` at the end of your pattern, that trailing comma causes a parse error. Always add it.

**Using `dbg!` in production code.**
`dbg!` prints to stderr with `[file:line]` labels. In production, this appears in logs or console output unexpectedly. Treat `dbg!` like a breakpoint — insert it to debug, remove it before committing.

**Writing macros when functions would work.**
The most common mistake: writing a `macro_rules!` macro for something a generic function handles perfectly. Macros have worse error messages and no IDE auto-complete inside the body. If a function can do it, write a function.

---

## Common Mistakes C Developers Make

**Using macros for constants.** In C, `#define MAX_SIZE 1024` is idiomatic. In Rust, use `const MAX_SIZE: usize = 1024;`. It's type-safe, scoped, and shows up in docs. Macros for constants are an antipattern.

**Expecting textual substitution.** `$x:expr` in a macro pattern matches a full expression — you can't use it to paste together identifiers. For identifier manipulation, you need `proc-macro2` / procedural macros. `$name:ident` matches an existing identifier; you can't construct new identifiers in `macro_rules!`.

**Missing semicolons in macro arms.** Each arm of `macro_rules!` ends with `;` (the pattern) and the expansion goes inside `{ ... }`. Forgetting the semicolon between arms is a common syntax error.

---

## Connection to Hexview

In the hexview project, you'll use macros for two things:

1. **`hex_line!` or a helper function** to produce the formatted hex output lines. Whether you write this as a macro or function depends on your design — either works.

2. **Standard library macros everywhere**: `format!` to build strings, `assert_eq!` in tests to verify your formatting is correct, `include_bytes!` if you want to embed test binary data directly into your test file.

Try building the line formatter as a function first. Once it works, consider: is there anything the macro version gives you that the function doesn't? That question is the right one to ask before introducing any macro.

```rust
// A clean function version — often all you need
fn format_hex_line(offset: usize, bytes: &[u8], uppercase: bool) -> String {
    let fmt_byte = |b: u8| -> String {
        if uppercase {
            format!("{b:02X}")
        } else {
            format!("{b:02x}")
        }
    };

    let hex: String = bytes
        .iter()
        .enumerate()
        .map(|(i, &b)| {
            if i == 8 { format!(" {}", fmt_byte(b)) }
            else { fmt_byte(b) }
        })
        .collect::<Vec<_>>()
        .join(" ");

    let ascii: String = bytes
        .iter()
        .map(|&b| if (0x20..0x7e).contains(&b) { b as char } else { '.' })
        .collect();

    let padded = format!("{hex:<49}");  // 49 chars: 16*3 + 1 for group gap
    format!("{offset:08x}: {padded}  {ascii}")
}
```

Move to a macro only if you later need compile-time evaluation, call-site information, or syntax that a function signature can't express.
