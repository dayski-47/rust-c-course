# Doc 02: Enums and Pattern Matching

This is probably the biggest conceptual shift from C to Rust. In C, `enum` is just a fancy integer. In Rust, an `enum` can carry completely different data in each variant. Combined with `match`, this gives you something more powerful than any `switch` statement you've written.

---

## Rust Enums Are Not C Enums

In C:

```c
typedef enum { NORTH, SOUTH, EAST, WEST } Direction;
// Direction is just an int underneath
```

In Rust, a simple enum looks similar:

```rust
enum Direction {
    North,
    South,
    East,
    West,
}
```

But here's where it diverges. Rust enums can carry data, and each variant can carry *different* data:

```rust
enum CollectorResult {
    Cpu(f64),             // a single float
    Memory { used: u64, total: u64 },  // a struct-style variant
    Error(String),        // a string error message
    Unavailable,          // no data at all
}
```

This is an algebraic data type. The size of `CollectorResult` in memory is the size of its largest variant, plus a tag byte to identify which variant is active. The compiler tracks that tag for you - you can never read the wrong variant's data. In C you'd do this with a tagged union, manually, with all the bugs that implies.

---

## match - The Most Important Control Flow in Rust

`match` is exhaustive. The compiler checks that you've handled every possible variant. If you add a new enum variant later and forget to handle it in a `match`, the build fails. This is enormously valuable for correctness.

```rust
fn describe_result(result: CollectorResult) {
    match result {
        CollectorResult::Cpu(pct) => {
            println!("CPU usage: {:.1}%", pct);
        }
        CollectorResult::Memory { used, total } => {
            println!("Memory: {} / {} MB", used, total);
        }
        CollectorResult::Error(msg) => {
            println!("Error collecting data: {}", msg);
        }
        CollectorResult::Unavailable => {
            println!("Metrics unavailable");
        }
    }
}
```

Notice that `match` binds the inner data to variable names (`pct`, `used`, `total`, `msg`). You don't need separate accessor functions. The data is right there in the arm.

### match yields a value

Like almost everything in Rust, `match` is an expression. It returns a value:

```rust
let description = match direction {
    Direction::North => "heading north",
    Direction::South => "heading south",
    Direction::East  => "heading east",
    Direction::West  => "heading west",
};
// description: &str
```

All arms must return the same type. The compiler enforces this.

### The _ catch-all

When you don't care about some variants:

```rust
match error_code {
    0 => println!("OK"),
    1 => println!("Permission denied"),
    2 => println!("Not found"),
    _ => println!("Unknown error: {}", error_code),
}
```

`_` matches everything and discards the value. Use it when you have a small number of interesting cases and want to handle the rest in bulk.

### Match guards - conditional arms

You can add an `if` condition to a match arm:

```rust
fn classify_cpu(pct: f64) -> &'static str {
    match pct {
        p if p < 0.0   => "invalid",
        p if p < 25.0  => "idle",
        p if p < 75.0  => "normal",
        p if p < 90.0  => "busy",
        _              => "critical",
    }
}
```

The arm only fires if the pattern matches AND the guard is true.

### Ranges in match

```rust
match error_code {
    0       => println!("Success"),
    1..=9   => println!("Soft error ({})", error_code),
    10..=99 => println!("Hard error ({})", error_code),
    _       => println!("Catastrophic"),
}
```

---

## if let - The Single-Variant Shorthand

Sometimes you only care about one variant. Writing a full `match` with `_` for everything else feels heavy. `if let` is the shortcut:

```rust
// Full match
match result.get("/proc/stat") {
    Some(data) => process(data),
    None => (), // do nothing
}

// Shorthand with if let
if let Some(data) = result.get("/proc/stat") {
    process(data);
}
```

Both do the same thing. Use `if let` when there's only one variant you care about.

### if let with else

```rust
if let CollectorResult::Cpu(pct) = collector_result {
    println!("CPU: {:.1}%", pct);
} else {
    println!("Not a CPU result");
}
```

---

## while let - Keep Going Until It Doesn't Match

Useful for draining iterators or channels:

```rust
let mut readings = vec![42.1, 43.7, 41.9];

while let Some(r) = readings.pop() {
    println!("Reading: {:.1}", r);
}
// Stops when pop() returns None (vec is empty)
```

---

## Option<T> and Result<T, E> Are Enums

You used these in Section 1. Now you can see what they really are:

```rust
// In the standard library:
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

They're just enums. When you write `match` on them, you're doing exactly what you learned above:

```rust
fn get_cpu_usage(system: &System) -> Option<f64> {
    let cpu = system.global_cpu_info();
    if cpu.cpu_usage() >= 0.0 {
        Some(cpu.cpu_usage() as f64)
    } else {
        None
    }
}

fn main() {
    // ... setup system ...
    match get_cpu_usage(&system) {
        Some(pct) => println!("CPU: {:.1}%", pct),
        None      => println!("Could not read CPU"),
    }
}
```

And for Result:

```rust
fn parse_threshold(s: &str) -> Result<f64, String> {
    s.parse::<f64>()
        .map_err(|e| format!("Invalid threshold '{}': {}", s, e))
}

match parse_threshold("75.5") {
    Ok(t)    => println!("Threshold: {:.1}", t),
    Err(msg) => println!("Error: {}", msg),
}
```

---

## 🟡 Nested Patterns

Patterns can be nested for deeper destructuring:

```rust
#[derive(Debug)]
struct Disk {
    mount: String,
    percent_full: u8,
}

enum Alert {
    DiskFull(Disk),
    CpuSpike { peak: f64, duration_secs: u32 },
    None,
}

fn handle_alert(alert: Alert) {
    match alert {
        Alert::DiskFull(Disk { mount, percent_full }) if percent_full > 95 => {
            println!("CRITICAL: {} is {}% full!", mount, percent_full);
        }
        Alert::DiskFull(Disk { mount, .. }) => {
            println!("Warning: {} is getting full", mount);
        }
        Alert::CpuSpike { peak, duration_secs } if duration_secs > 30 => {
            println!("Sustained spike: {:.1}% for {}s", peak, duration_secs);
        }
        Alert::CpuSpike { peak, .. } => {
            println!("Brief spike: {:.1}%", peak);
        }
        Alert::None => {}
    }
}
```

The `..` in a pattern means "ignore the rest of the fields". You don't need to name every field if you don't use it.

---

## Defining Your Own Enum for the Project

For the system monitor, an enum is a natural way to represent different types of metrics:

```rust
#[derive(Debug)]
enum MetricValue {
    Percentage(f64),
    Megabytes(u64),
    Count(u32),
    Text(String),
}

#[derive(Debug)]
struct Metric {
    name: String,
    value: MetricValue,
}

fn format_metric(m: &Metric) -> String {
    match &m.value {
        MetricValue::Percentage(p) => format!("{}: {:.1}%", m.name, p),
        MetricValue::Megabytes(mb) => format!("{}: {} MB", m.name, mb),
        MetricValue::Count(n)      => format!("{}: {}", m.name, n),
        MetricValue::Text(s)       => format!("{}: {}", m.name, s),
    }
}
```

---

## How It Breaks

**Adding a new enum variant breaks every exhaustive match.**
This is a feature, but it surprises people the first time. If you add a new variant to an enum:
```rust
enum MetricValue {
    Percentage(f64),
    Megabytes(u64),
    Count(u32),
    Text(String),
    RawBytes(u64),  // new variant added here
}
```
Every `match` in your codebase that pattern-matches `MetricValue` without a `_` catch-all will now fail to compile. The compiler tells you exactly which files and lines need updating. This is intentional - it forces you to handle the new case everywhere. The alternative (silently ignoring new variants) causes bugs. If you're a library author adding a variant to a public enum, use `#[non_exhaustive]` to warn users that new variants may appear.

**`#[non_exhaustive]` on external library enums.**
When you depend on a crate that marks an enum `#[non_exhaustive]`, you must include a `_` arm in your match even if you handle every current variant:
```rust
match some_external_error_kind {
    ExternalError::NotFound => { ... }
    ExternalError::Permission => { ... }
    _ => { /* handle future variants */ }
}
```
Without the `_`, the code won't compile, and the error message explains why. This is the crate author saying "we may add variants in minor versions; handle the unknown case."

**Using `if let` when `match` would catch a missed case.**
`if let Some(x) = value { ... }` silently ignores `None`. This is correct when absence is genuinely fine to ignore. It becomes a bug when you meant to handle the `None` case differently but forgot:
```rust
// Silently ignores failure to read CPU
if let Some(cpu) = get_cpu_reading() {
    display(cpu);
}

// Forces you to handle the failure case
match get_cpu_reading() {
    Some(cpu) => display(cpu),
    None => display_error("CPU data unavailable"),
}
```
Prefer `match` when both outcomes matter. Use `if let` only when you've consciously decided the other case is a no-op.

---

## Common Mistakes

**1. Not handling all variants.** If you don't cover every variant in a `match`, the compiler refuses to compile. This is a feature, not a bug. Add the missing arm.

**2. Forgetting that match arms need to return the same type.** If one arm returns a `String` and another returns `()`, the compiler complains. Use `()` as a no-op return or restructure so all arms agree.

**3. Moving out of an enum when you only want to borrow.** If you `match result` and your arm does `Some(data) => use(data)`, you've moved `data` out. To avoid this, match on a reference: `match &result { Some(data) => use(data), ... }`.

**4. Using if let when you actually need to handle both cases.** `if let Some(x) = opt { ... }` silently ignores `None`. If you need different behavior for both cases, use a full `match` or add an `else` branch.

**5. Confusing `_` with `..`.** `_` ignores one value. `..` ignores the remaining fields in a struct pattern. They're different and serve different purposes.
