# Control Flow and Functions

🟢 Easy - most of this feels familiar, with one key twist: expressions vs statements.

If you know C control flow, you know 80% of this already. The big difference is that Rust control flow constructs are **expressions** - they produce values, not just side effects. Once that clicks, a lot of Rust idioms make sense.

---

## `if` / `else`

Looks like C, behaves like C, but can also produce a value:

```rust
fn main() {
    let x = 42;

    // Used as a statement (exactly like C)
    if x < 42 {
        println!("too small");
    } else if x == 42 {
        println!("just right");
    } else {
        println!("too large");
    }

    // Used as an expression (assigns the result)
    let description = if x == 42 { "the answer" } else { "something else" };
    println!("{description}");
}
```

The expression form requires both branches to produce the same type. No parentheses around the condition (unlike C), and curly braces are always required around the body (no one-liners without braces).

---

## `loop`

An infinite loop until a `break`:

```rust
fn main() {
    let mut x = 0;
    loop {
        if x == 5 {
            break;
        }
        x += 1;
    }
    println!("{x}");
}
```

`loop` can also return a value - you put the expression after `break`:

```rust
fn main() {
    let mut attempts = 0;
    let result = loop {
        attempts += 1;
        if attempts == 3 {
            break "connected";  // this is the value the loop evaluates to
        }
    };
    println!("{result}");  // "connected"
}
```

This is occasionally useful for retry logic when reading from flaky I/O sources.

---

## `while`

```rust
fn main() {
    let mut offset = 0usize;
    let total_bytes = 256usize;
    while offset < total_bytes {
        // process 16-byte chunk starting at offset
        offset += 16;
    }
}
```

Identical to C's `while`. Use it when you have a condition but not a known number of iterations.

---

## `for` over ranges

This is the idiomatic way to iterate in Rust. You won't write `for (int i = 0; i < n; i++)` - you write:

```rust
fn main() {
    let file_size = 256usize;
    let row_size = 16usize;

    // iterate over row starting offsets: 0, 16, 32, ...
    for row_start in (0..file_size).step_by(row_size) {
        println!("row at offset {row_start:#06x}");
    }

    // 0..16 is exclusive on the right: indices 0 through 15
    for i in 0..16 {
        println!("{i}");  // prints 0 through 15
    }
}
```

`0..n` is exclusive on the right (0 through n-1). `0..=n` is inclusive. For iterating over fixed-size row chunks in the hex output, use `step_by` on a range.

You can also iterate over collections directly:

```rust
fn main() {
    let bytes = vec![0x48u8, 0x65, 0x6C, 0x6C];  // "Hell" in ASCII
    for byte in &bytes {
        print!("{byte:02x} ");
    }
}
```

---

## Functions

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b  // no semicolon: this is the return value
}

fn main() {
    let result = add(3, 4);
    println!("{result}");
}
```

Key differences from C:

- Parameter types are written after the parameter name: `a: i32` not `int a`
- Return type comes after `->`: `-> i32` not `int add(...)`
- **The last expression in a function body is the return value** - no `return` keyword needed if there's no semicolon

The implicit return is idiomatic Rust. The expression without a semicolon at the end is what the function returns. You can use explicit `return` for early exits:

```rust
fn is_printable(byte: u8) -> bool {
    if byte < 0x20 {
        return false;  // early exit - control characters
    }
    byte < 0x7F  // implicit return - printable ASCII range
}
```

Functions that don't return anything have return type `()` (pronounced "unit"), which is the empty tuple. You usually omit it:

```rust
fn print_byte(byte: u8) {  // implicitly returns ()
    println!("{byte:02x}");
}
```

---

## Expression Blocks

Any block `{ ... }` is an expression in Rust. The last expression in the block (without semicolon) is its value:

```rust
fn main() {
    let x = {
        let a = 10;
        let b = 20;
        a + b  // no semicolon: this is the value of the block
    };
    println!("{x}");  // 30
}
```

This is how functions work under the hood - a function body is just a block, and the last expression is what it returns.

---

## Structs

You'll need structs to represent a hex row or viewer configuration. They work like C structs, with some differences:

```rust
// Define the struct
struct HexRow {
    offset: usize,
    byte_count: usize,
}

// Create an instance
fn main() {
    let row = HexRow {
        offset: 0x0010,
        byte_count: 16,
    };

    // Access fields with dot notation
    println!("Row at {:#06x}, {} bytes", row.offset, row.byte_count);
}
```

Structs are immutable by default, same as variables. To mutate fields, the instance must be `let mut`:

```rust
fn main() {
    let mut row = HexRow { offset: 0, byte_count: 0 };
    row.byte_count = 16;  // OK because row is mut
}
```

You can add methods to structs with `impl`:

```rust
struct HexRow {
    offset: usize,
    byte_count: usize,
}

impl HexRow {
    // Associated function (like a static method / constructor in C++)
    fn new(offset: usize) -> HexRow {
        HexRow { offset, byte_count: 0 }
    }

    // Method (takes &self - immutable reference to the instance)
    fn display(&self) {
        println!("{:#010x}  ({} bytes)", self.offset, self.byte_count);
    }
}

fn main() {
    let row = HexRow::new(0x0020);
    row.display();
}
```

`&self` is like `const HexRow *self` in C - a read-only pointer to the struct. `&mut self` would be a mutable pointer. No implicit `this` keyword in Rust; you always write it explicitly.

---

## `match` - Pattern Matching

`match` is like `switch` in C but more powerful. You'll use it constantly with `Result` and `Option` (covered in doc 06). Quick preview:

```rust
fn main() {
    let byte: u8 = 0x41;  // 'A'

    match byte {
        0x00 => println!("null"),
        0x09 => println!("tab"),
        0x0A => println!("newline"),
        0x20..=0x7E => println!("printable ASCII"),
        _ => println!("non-printable"),  // _ is the default case (like default: in C)
    }
}
```

Rust's `match` is exhaustive - you must cover all cases or the compiler complains. No falling through between cases. No forgetting a `break`.

---

## Common Mistakes C Developers Make

**Putting a semicolon on the last expression in a function.** `a + b;` returns `()`, not the value of `a + b`. The semicolon "discards" the expression's value. No semicolon on the return expression.

**Writing `for (int i = 0; i < n; i++)`.** That syntax doesn't exist in Rust. Use `for i in 0..n`.

**Forgetting that `if` needs curly braces.** `if x > 0 do_thing();` doesn't compile. Always `if x > 0 { do_thing(); }`.

**Forgetting `&` when iterating over a Vec.** `for x in bytes` moves ownership of each element out of the Vec (which works for `Copy` types like `u8`, but fails for types that don't implement `Copy`). `for x in &bytes` borrows instead, and is usually what you want.

**Writing `switch` instead of `match`.** There's no `switch` in Rust. `match` is the equivalent, and it's much more powerful.
