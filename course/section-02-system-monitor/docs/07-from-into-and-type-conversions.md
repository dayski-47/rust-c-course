# Doc 07 — From, Into, and Type Conversions

🟡 Think about it — Rust has no implicit conversions. That sounds restrictive until you see how it eliminates entire categories of bugs.

In C, you can assign an `int` to a `long`, a `float` to a `double`, a `char*` to `const char*` — all silently. The compiler promotes and converts without asking. In Rust, every conversion is explicit. That sounds like more work, but it means: narrowing bugs, sign confusion, precision loss — all become compile errors. This doc explains how Rust's conversion traits replace implicit casts, and why that's better.

---

## Why No Implicit Conversion

Consider this C code:

```c
int get_count(void) { return -1; }  // -1 means "error"

size_t len = get_count();   // len is SIZE_MAX — silent bug
if (len > 1000) {           // This is true when you meant "error"!
    allocate_huge_buffer(len);  // crash
}
```

This is `int` being silently converted to `size_t`, turning -1 into 18446744073709551615. Rust prevents this entirely:

```rust
fn get_count() -> i32 { -1 }

let len: usize = get_count();  // COMPILE ERROR: mismatched types
                               // expected `usize`, found `i32`
```

The compiler forces you to make the conversion explicit — and at that point, you'd use `TryInto` which tells you if the conversion fails:

```rust
use std::convert::TryInto;

match get_count().try_into() {
    Ok(len) => allocate(len),
    Err(_)  => handle_error(),
}
```

---

## The Conversion Trait Family

| Trait | When it succeeds | Example |
|-------|------------------|---------|
| `From<T>` / `Into<T>` | Always — infallible | `u8` → `u32`, `&str` → `String` |
| `TryFrom<T>` / `TryInto<T>` | Maybe — can fail | `i64` → `u8`, `String` → custom type |
| `AsRef<T>` | Always — borrows as another reference type | `String` → `&str`, `Vec<u8>` → `&[u8]` |
| `Deref<Target=T>` | Always — automatic coercion | `Box<T>` → `&T`, `String` → `&str` |
| `Display` / `FromStr` | Print / Parse | `42.to_string()`, `"42".parse::<i32>()` |

---

## `From<T>` and `Into<T>`: Infallible Conversions

Implement `From<T>` when converting from `T` to your type can never fail.

```rust
struct CpuPercent(f64);  // A newtype wrapping f64

impl From<f64> for CpuPercent {
    fn from(val: f64) -> Self {
        CpuPercent(val)
    }
}

fn main() {
    // Use From explicitly:
    let cpu = CpuPercent::from(72.5);

    // Use Into — works automatically when From is implemented:
    let cpu: CpuPercent = 72.5.into();
}
```

**The key rule**: if you implement `From<A> for B`, Rust automatically provides `Into<B> for A`. You never need to implement both. Always implement `From`, let `Into` come for free.

### Standard library `From` implementations you use constantly

```rust
// &str → String
let s: String = String::from("hello");
let s: String = "hello".into();

// integer widening (always safe, never loses data)
let n: u64 = u64::from(42u32);
let n: i64 = i64::from(42i32);

// bool → integer
let n: u8 = u8::from(true);  // 1
let n: i32 = i32::from(false); // 0

// Vec<u8> → various
let bytes: Vec<u8> = Vec::from("hello".as_bytes());

// Option → conversion
let opt: Option<i32> = Some(42);
```

### Implementing From for domain types

This pattern appears constantly in the system monitor — converting raw OS data into typed representations:

```rust
#[derive(Debug)]
pub struct MemoryBytes(u64);

#[derive(Debug)]
pub struct MemoryMegabytes(f64);

impl From<MemoryBytes> for MemoryMegabytes {
    fn from(bytes: MemoryBytes) -> Self {
        MemoryMegabytes(bytes.0 as f64 / 1_048_576.0)
    }
}

#[derive(Debug)]
pub struct RawSample {
    pub cpu_raw: u64,    // from /proc/stat: raw tick counts
    pub total_ticks: u64,
    pub mem_kb: u64,     // from /proc/meminfo: kilobytes
}

#[derive(Debug)]
pub struct Sample {
    pub cpu_percent: f64,
    pub mem_bytes: MemoryBytes,
}

impl From<RawSample> for Sample {
    fn from(raw: RawSample) -> Self {
        Sample {
            cpu_percent: (raw.cpu_raw as f64 / raw.total_ticks as f64) * 100.0,
            mem_bytes: MemoryBytes(raw.mem_kb * 1024),
        }
    }
}

fn main() {
    let raw = RawSample { cpu_raw: 750, total_ticks: 1000, mem_kb: 8_192_000 };
    let sample: Sample = raw.into();   // From<RawSample> for Sample kicks in
    let mem_mb: MemoryMegabytes = sample.mem_bytes.into();  // MemoryBytes → MB
    println!("CPU: {:.1}%, Mem: {:.0}MB", sample.cpu_percent, mem_mb.0);
}
```

---

## `TryFrom<T>` and `TryInto<T>`: Fallible Conversions

Use `TryFrom`/`TryInto` when the conversion might fail — typically when the value is out of range or invalid.

```rust
use std::convert::TryFrom;

// Built-in: narrowing integer conversions
let big: i64 = 300;
let small = u8::try_from(big);  // Err — 300 doesn't fit in u8

let ok: i64 = 200;
let small = u8::try_from(ok);   // Ok(200)

// The ? operator works naturally:
fn narrow(n: i64) -> Result<u8, std::num::TryFromIntError> {
    u8::try_from(n)  // returns Err automatically if out of range
}
```

### Implementing TryFrom for validated types

This is where `TryFrom` really shines — converting raw input into types that enforce invariants:

```rust
use std::convert::TryFrom;

#[derive(Debug, Clone)]
pub struct ValidatedSampleRate {
    hz: u32,  // only 1–1000 Hz is valid
}

#[derive(Debug)]
pub struct InvalidSampleRate(u32);

impl std::fmt::Display for InvalidSampleRate {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "sample rate {} Hz is out of range (1–1000)", self.0)
    }
}

impl TryFrom<u32> for ValidatedSampleRate {
    type Error = InvalidSampleRate;

    fn try_from(hz: u32) -> Result<Self, Self::Error> {
        if hz >= 1 && hz <= 1000 {
            Ok(ValidatedSampleRate { hz })
        } else {
            Err(InvalidSampleRate(hz))
        }
    }
}

fn main() {
    match ValidatedSampleRate::try_from(100) {
        Ok(rate) => println!("Sampling at {} Hz", rate.hz),
        Err(e)   => eprintln!("Error: {e}"),
    }

    // With ? in a function that returns Result:
    // let rate = ValidatedSampleRate::try_from(user_input)?;
}
```

---

## `AsRef<T>` and `Deref`: Borrowing Across Types

`AsRef<T>` lets a type be borrowed as a reference to `T`. This is how you write functions that accept both `String` and `&str` without overloading:

```rust
// Without AsRef: you'd need two functions or clone everything
fn log_event(name: &str) {
    println!("[EVENT] {name}");
}

fn main() {
    let owned: String = "disk_read".into();
    log_event(&owned);    // Works — &String coerces to &str via Deref
    log_event("cpu_high"); // Works — &str is &str
}

// With AsRef: even more flexible
fn log_event<S: AsRef<str>>(name: S) {
    println!("[EVENT] {}", name.as_ref());
}
// Now accepts: &str, String, Box<str>, Cow<str>, and any type implementing AsRef<str>
```

`AsRef` is commonly used for paths:

```rust
use std::path::Path;

fn read_config<P: AsRef<Path>>(path: P) -> std::io::Result<String> {
    std::fs::read_to_string(path.as_ref())
}

fn main() {
    read_config("/etc/monitor.conf")?;       // &str
    read_config(String::from("/etc/app"))?;  // String
    read_config(std::path::PathBuf::from("/etc/x"))?;  // PathBuf
}
```

**Deref coercions** happen automatically when you dereference a smart pointer or call a method that needs a reference. The most common ones:
- `Box<T>` → `&T`
- `Vec<T>` → `&[T]` (that's why `&my_vec` works where `&[T]` is expected)
- `String` → `&str` (that's why `&my_string` works where `&str` is expected)

---

## The `as` Keyword: When to Use It

`as` is Rust's explicit cast operator. It always succeeds but may truncate, wrap, or lose precision:

```rust
let big: u32 = 1000;
let small = big as u8;   // 232 — wraps around silently (1000 % 256)
let f: f64 = 3.99;
let n = f as i64;        // 3 — truncates, no rounding

let signed: i32 = -1;
let unsigned = signed as u32;  // 4294967295 — same bit pattern
```

**When `as` is appropriate:**
- Converting between float and integer types where you've verified the range
- Pointer casting in `unsafe` code
- Index arithmetic: `index as usize`

**When to use `From`/`TryFrom` instead of `as`:**
- Widening integers: always use `From` — it's infallible and documents intent
- Narrowing integers: always use `TryFrom` — it tells you if data is lost
- Any struct-to-struct conversion: implement `From`

The issue with `as`: it silently wraps on overflow, silently truncates floats, and silently reinterprets signed/unsigned. `TryFrom` gives you an `Err` instead of silent data corruption.

---

## Converting Error Types: The `?` + `From` Pattern

This is where `From` has its biggest impact on practical code. The `?` operator uses `From` to convert error types automatically:

```rust
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum MonitorError {
    Io(io::Error),
    Parse(ParseIntError),
    InvalidRange { value: i64, min: i64, max: i64 },
}

// Implementing From for each error type lets ? do the conversion:
impl From<io::Error> for MonitorError {
    fn from(e: io::Error) -> Self { MonitorError::Io(e) }
}

impl From<ParseIntError> for MonitorError {
    fn from(e: ParseIntError) -> Self { MonitorError::Parse(e) }
}

// Now ? works across different error types in one function:
fn load_sample_rate(path: &str) -> Result<u32, MonitorError> {
    let text = std::fs::read_to_string(path)?;  // io::Error → MonitorError via From
    let rate: i64 = text.trim().parse()?;        // ParseIntError → MonitorError via From

    if rate < 1 || rate > 1000 {
        return Err(MonitorError::InvalidRange { value: rate, min: 1, max: 1000 });
    }

    Ok(rate as u32)
}
```

Without `From`, you'd need `map_err(MonitorError::Io)?` on every I/O operation and `map_err(MonitorError::Parse)?` on every parse. `From` eliminates the repetition. You'll use this pattern in every section from here forward — errors are the thing that makes multi-layer Rust code readable.

---

## `Display` and `FromStr`: Human-Readable Conversion

`Display` is how a type converts to a human-readable string. `FromStr` is how it converts from one:

```rust
use std::fmt;
use std::str::FromStr;

#[derive(Debug)]
pub struct CpuPercent(pub f64);

// Display: used by println!("{}", val), format!("{}", val), .to_string()
impl fmt::Display for CpuPercent {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:.1}%", self.0)
    }
}

// FromStr: used by "72.5".parse::<CpuPercent>()
#[derive(Debug)]
pub struct ParseCpuError(String);

impl fmt::Display for ParseCpuError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "invalid CPU percent: {}", self.0)
    }
}

impl FromStr for CpuPercent {
    type Err = ParseCpuError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let s = s.trim_end_matches('%');
        s.parse::<f64>()
            .map(CpuPercent)
            .map_err(|_| ParseCpuError(s.to_string()))
    }
}

fn main() {
    let cpu = CpuPercent(72.5);
    println!("{cpu}");  // "72.5%"

    let parsed: CpuPercent = "85.3%".parse().expect("valid");
    println!("{parsed}");  // "85.3%"
}
```

---

## Conversions in the System Monitor

In your system monitor, you'll work with several layers of data representation:

```
/proc/stat (raw text)
    ↓ parse with FromStr
RawStat (raw tick counts, kilobytes from the kernel)
    ↓ From<RawStat> for Sample
Sample (cpu_percent: f64, mem_bytes: u64)
    ↓ From<Sample> for DisplaySample
DisplaySample (cpu: "72.5%", mem: "7.8GB", formatted for output)
    ↓ Display
Output to terminal / report file
```

Each arrow is a `From` or `Display` implementation. When something goes wrong in parsing, `TryFrom` or `FromStr` captures the error at exactly the right layer. This layered conversion architecture means your processing logic deals with clean typed data, not raw strings.

---

## How It Breaks

**Implementing `Into` directly instead of `From`.**
Rust's blanket impl gives you `Into` for free when you implement `From`. If you implement `Into` directly, you won't get `From`, and code that calls `From::from(...)` won't work. Always implement `From`, never implement `Into`.

**Using `as` for narrowing without thinking.**
`42i64 as u8` compiles. It also produces 42. `300i64 as u8` produces 44. `1000i64 as u8` produces 232. There's no warning. If you're narrowing an integer that might be out of range, use `TryFrom` — it produces an `Err` instead of silent corruption.

**Forgetting to implement `From` for errors when using `?`.**
If your function returns `Result<_, MyError>` and you call `?` on something returning `Result<_, io::Error>`, the compiler needs `From<io::Error> for MyError`. Without it, you get a type mismatch error at the `?`. The fix: add `impl From<io::Error> for MyError`. This is so common that the `thiserror` crate (which you'll use from section 3 onward) generates these `From` impls automatically from `#[from]` annotations.

**Confusion between `AsRef` and `Borrow`.**
Both let you view a type as a reference to another type. The difference is subtle: `AsRef` is for ergonomic conversions, `Borrow` is for types that should behave identically for hashing and comparison purposes. For function parameters that accept "anything string-like", use `AsRef<str>`. For `HashMap` lookups, use `Borrow<str>` (already implemented for `String`).

---

## Quick Reference

```
Infallible conversion (guaranteed to succeed):
  From<T>       → implement this, get Into<T> for free
  Into<T>       → use this in function signatures for flexibility

Fallible conversion (might return Err):
  TryFrom<T>    → implement this, get TryInto<T> for free
  TryInto<T>    → use this at call sites

Borrowing as another reference type:
  AsRef<T>      → implement to let your type be passed where &T is expected
  Deref<Target> → implement for smart pointers; enables automatic coercions

Human-readable:
  Display       → enables println!("{}", val), .to_string()
  FromStr       → enables "text".parse::<MyType>()

The as keyword:
  → Use for pointer casts and float↔integer in unsafe contexts
  → Prefer From/TryFrom for all other numeric conversions
```
