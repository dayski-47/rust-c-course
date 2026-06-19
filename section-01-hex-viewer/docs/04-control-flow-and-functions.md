# Control Flow and Functions

🟢 Easy — most of this feels familiar, with one key twist: expressions vs statements.

If you know C control flow, you know 80% of this already. The big difference is that Rust control flow constructs are **expressions** — they produce values, not just side effects. Once that clicks, a lot of Rust idioms make sense.

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

`loop` can also return a value — you put the expression after `break`:

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
    let mut port = 1;
    while port <= 1024 {
        // scan port
        port += 1;
    }
}
```

Identical to C's `while`. Use it when you have a condition but not a known number of iterations.

---

## `for` over ranges

This is the idiomatic way to iterate in Rust. You won't write `for (int i = 0; i < n; i++)` — you write:

```rust
fn main() {
    // 1..=1024 means 1 through 1024 inclusive
    for port in 1u16..=1024 {
        println!("scanning port {port}");
    }

    // 1..1024 is exclusive on the right: 1 through 1023
    for i in 0..10 {
        println!("{i}");  // prints 0 through 9
    }
}
```

`1..=1024` is an inclusive range. `1..1024` is exclusive on the right (0 to 1023 for the second example). For port scanning, you almost always want `start..=end`.

You can also iterate over collections directly:

```rust
fn main() {
    let ports = vec![22u16, 80, 443, 8080];
    for port in &ports {
        println!("checking port {port}");
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
- **The last expression in a function body is the return value** — no `return` keyword needed if there's no semicolon

The implicit return is idiomatic Rust. The expression without a semicolon at the end is what the function returns. You can use explicit `return` for early exits:

```rust
fn check_port(port: u16) -> bool {
    if port == 0 {
        return false;  // early exit
    }
    // ...
    true  // implicit return
}
```

Functions that don't return anything have return type `()` (pronounced "unit"), which is the empty tuple. You usually omit it:

```rust
fn print_port(port: u16) {  // implicitly returns ()
    println!("port: {port}");
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

This is how functions work under the hood — a function body is just a block, and the last expression is what it returns.

---

## Structs

You'll need structs to represent a scan result or a configuration. They work like C structs, with some differences:

```rust
// Define the struct
struct ScanResult {
    port: u16,
    is_open: bool,
}

// Create an instance
fn main() {
    let result = ScanResult {
        port: 443,
        is_open: true,
    };

    // Access fields with dot notation
    println!("Port {} is {}", result.port, if result.is_open { "open" } else { "closed" });
}
```

Structs are immutable by default, same as variables. To mutate fields, the instance must be `let mut`:

```rust
fn main() {
    let mut result = ScanResult { port: 80, is_open: false };
    result.is_open = true;  // OK because result is mut
}
```

You can add methods to structs with `impl`:

```rust
struct ScanResult {
    port: u16,
    is_open: bool,
}

impl ScanResult {
    // Associated function (like a static method / constructor in C++)
    fn new(port: u16) -> ScanResult {
        ScanResult { port, is_open: false }
    }

    // Method (takes &self — immutable reference to the instance)
    fn display(&self) {
        if self.is_open {
            println!("Port {} is OPEN", self.port);
        }
    }
}

fn main() {
    let result = ScanResult::new(443);
    result.display();
}
```

`&self` is like `const ScanResult *self` in C — a read-only pointer to the struct. `&mut self` would be a mutable pointer. No implicit `this` keyword in Rust; you always write it explicitly.

---

## `match` — Pattern Matching

`match` is like `switch` in C but more powerful. You'll use it constantly with `Result` and `Option` (covered in doc 06). Quick preview:

```rust
fn main() {
    let port: u16 = 443;

    match port {
        22 => println!("SSH"),
        80 => println!("HTTP"),
        443 => println!("HTTPS"),
        1024..=65535 => println!("high port"),
        _ => println!("other"),  // _ is the default case (like default: in C)
    }
}
```

Rust's `match` is exhaustive — you must cover all cases or the compiler complains. No falling through between cases. No forgetting a `break`.

---

## Common Mistakes C Developers Make

**Putting a semicolon on the last expression in a function.** `a + b;` returns `()`, not the value of `a + b`. The semicolon "discards" the expression's value. No semicolon on the return expression.

**Writing `for (int i = 0; i < n; i++)`.** That syntax doesn't exist in Rust. Use `for i in 0..n`.

**Forgetting that `if` needs curly braces.** `if x > 0 do_thing();` doesn't compile. Always `if x > 0 { do_thing(); }`.

**Forgetting `&` when iterating over a Vec.** `for x in ports` moves ownership of each element out of the Vec (which works for `Copy` types like `u16`, but fails for types that don't implement `Copy`). `for x in &ports` borrows instead, and is usually what you want.

**Writing `switch` instead of `match`.** There's no `switch` in Rust. `match` is the equivalent, and it's much more powerful.
