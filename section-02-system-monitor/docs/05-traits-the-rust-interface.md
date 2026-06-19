# Doc 05: Traits — The Rust Interface

Rust has no inheritance. There is no base class, no virtual dispatch hierarchy, no `extends` keyword. Instead, Rust uses traits to define shared behavior. A trait is a contract: "any type that implements this trait can do these things."

If you're thinking "that sounds like a C++ pure virtual interface" — yes, that's the closest analogy, but with some important differences that make it safer and more flexible.

---

## Defining and Implementing a Trait

A trait is a set of method signatures. Types implement the trait by providing the method bodies:

```rust
trait Collector {
    fn name(&self) -> &str;
    fn collect(&self) -> f64;
}
```

Now any type that implements `Collector` has to provide both methods:

```rust
struct CpuCollector {
    core_count: u32,
}

struct MemoryCollector {
    total_mb: u64,
}

impl Collector for CpuCollector {
    fn name(&self) -> &str {
        "cpu"
    }

    fn collect(&self) -> f64 {
        // Real code would read from sysinfo here
        42.7
    }
}

impl Collector for MemoryCollector {
    fn name(&self) -> &str {
        "memory"
    }

    fn collect(&self) -> f64 {
        // Returns percentage used
        62.5
    }
}
```

Now you can call `collect()` on either type without knowing which one it is.

---

## Default Method Implementations

A trait can provide a default implementation. Types can override it or use the default:

```rust
trait Collector {
    fn name(&self) -> &str;
    fn collect(&self) -> f64;

    // Default implementation — types can override this
    fn is_healthy(&self) -> bool {
        self.collect() < 90.0
    }

    fn report(&self) -> String {
        format!("{}: {:.1} ({})",
            self.name(),
            self.collect(),
            if self.is_healthy() { "OK" } else { "WARN" }
        )
    }
}
```

Any type that implements `name()` and `collect()` automatically gets `is_healthy()` and `report()` for free. They can override those defaults if they need different behavior.

---

## Traits as Parameters — Static Dispatch

The `impl Trait` syntax lets you write a function that accepts any type implementing a trait:

```rust
fn print_metric(collector: &impl Collector) {
    println!("{}", collector.report());
}

let cpu = CpuCollector { core_count: 8 };
let mem = MemoryCollector { total_mb: 16384 };

print_metric(&cpu);  // Works
print_metric(&mem);  // Works
```

This is **static dispatch**. The compiler generates a separate version of `print_metric` for each concrete type (`CpuCollector`, `MemoryCollector`). At runtime there's no overhead — the call is direct, not through a pointer.

The equivalent using trait bounds with generics looks slightly different but behaves the same:

```rust
fn print_metric<T: Collector>(collector: &T) {
    println!("{}", collector.report());
}

// For multiple trait requirements:
fn print_and_log<T>(collector: &T)
where
    T: Collector + std::fmt::Debug,
{
    println!("{:?}", collector);
    println!("{}", collector.report());
}
```

The `where` clause is just a different way to write the bounds — use it when the inline version gets too long.

---

## impl Trait in Return Position

You've seen `impl Trait` as a parameter. It also appears in return position:

```rust
fn active_collectors(collectors: &[Box<dyn Collector>]) -> impl Iterator<Item = f64> + '_ {
    collectors.iter()
        .map(|c| c.collect())
        .filter(|&v| v > 0.0)
}
```

The caller gets an iterator — they can chain `.sum()`, `.max_by()`, etc. — but they never need to know the concrete iterator type (`Filter<Map<...>>`), which the compiler generates internally. This is zero cost: no heap allocation, the chain is inlined.

The same pattern works for returning closures:

```rust
fn threshold_checker(limit: f64) -> impl Fn(f64) -> bool {
    move |value| value > limit
}

let is_critical = threshold_checker(90.0);
println!("{}", is_critical(92.5));  // true
println!("{}", is_critical(42.0));  // false
```

**The critical rule:** all return paths must return the *same* concrete type. The compiler resolves `impl Trait` to one specific type — it's not a runtime union.

```rust
// COMPILE ERROR: if/else returns different concrete types
fn make_collector(cpu: bool) -> impl Collector {
    if cpu { CpuCollector }        // concrete type: CpuCollector
    else { MemCollector { ... } }  // concrete type: MemCollector — mismatch!
}
```

When you genuinely need to return different types based on runtime data, use `Box<dyn Trait>`:

```rust
fn make_collector(cpu: bool) -> Box<dyn Collector> {
    if cpu { Box::new(CpuCollector) }
    else { Box::new(MemCollector { total_mb: 16384 }) }
}
```

`Box<dyn Trait>` heap-allocates and uses runtime dispatch; `impl Trait` is stack-allocated and statically dispatched. Prefer `impl Trait` unless you need the type flexibility.

---

## The Display Trait — Making Your Struct Printable

You've used `{:?}` (Debug) to print structs. To use `{}` — the human-readable format — you implement `std::fmt::Display`:

```rust
use std::fmt;

#[derive(Debug)]
struct Metrics {
    cpu_percent: f64,
    memory_used_mb: u64,
    memory_total_mb: u64,
}

impl fmt::Display for Metrics {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f,
            "CPU: {:.1}%  |  Memory: {} / {} MB  ({:.0}%)",
            self.cpu_percent,
            self.memory_used_mb,
            self.memory_total_mb,
            (self.memory_used_mb as f64 / self.memory_total_mb as f64) * 100.0
        )
    }
}

fn main() {
    let m = Metrics {
        cpu_percent: 42.7,
        memory_used_mb: 6144,
        memory_total_mb: 16384,
    };

    println!("{}", m);   // Uses Display
    println!("{:?}", m); // Uses Debug
}
```

Output:
```
CPU: 42.7%  |  Memory: 6144 / 16384 MB  (37%)
Metrics { cpu_percent: 42.7, memory_used_mb: 6144, memory_total_mb: 16384 }
```

Implementing `Display` also gives you `.to_string()` for free — it calls your `fmt` method.

**When to use Debug vs Display:**
- `Debug` (`{:?}`) — for developers, shows internal structure. Derive it automatically.
- `Display` (`{}`) — for end users, you control the format. Implement it manually.

---

## From and Into — Type Conversions

The `From` and `Into` traits are how Rust handles explicit type conversion. You implement `From<T>` for your type and `Into<YourType>` is automatically available on `T`:

```rust
#[derive(Debug)]
struct Megabytes(u64);

impl From<u64> for Megabytes {
    fn from(bytes: u64) -> Megabytes {
        Megabytes(bytes / (1024 * 1024))
    }
}

fn main() {
    let total_bytes: u64 = 17_179_869_184; // 16 GB in bytes

    // Using From explicitly
    let total_mb = Megabytes::from(total_bytes);

    // Using Into (derived automatically when From is implemented)
    let total_mb: Megabytes = total_bytes.into();

    println!("{} MB", total_mb.0); // 16384 MB
}
```

This replaces C's implicit casts with something explicit and type-safe. You can't accidentally convert bytes to megabytes without going through the `From` impl.

---

## Common Standard Library Traits

These traits appear everywhere in Rust. You'll encounter them constantly:

| Trait | What it enables | How to get it |
|-------|----------------|---------------|
| `Debug` | `println!("{:?}", x)` | `#[derive(Debug)]` |
| `Display` | `println!("{}", x)` | `impl fmt::Display for T` |
| `Clone` | `x.clone()` | `#[derive(Clone)]` |
| `Copy` | Implicit copy instead of move | `#[derive(Copy, Clone)]` |
| `PartialEq` | `x == y` | `#[derive(PartialEq)]` |
| `PartialOrd` | `x < y`, `x > y` | `#[derive(PartialOrd)]` |
| `Default` | `T::default()`, zero/empty values | `#[derive(Default)]` |
| `From<T>` | `MyType::from(t)` and `t.into()` | `impl From<T> for MyType` |

`Copy` is special: a type that is `Copy` gets implicitly copied when assigned or passed, instead of moved. Primitive types (`i32`, `f64`, `bool`, etc.) are all `Copy`. Your own types are `Copy` only if you derive it and all their fields are also `Copy`. If a type owns heap data (like `String` or `Vec`), it can't be `Copy`.

---

## 🔴 Dynamic Dispatch — Box<dyn Trait>

Static dispatch requires knowing the concrete type at compile time. When you want to store multiple different types in the same collection, you need dynamic dispatch:

```rust
fn main() {
    let collectors: Vec<Box<dyn Collector>> = vec![
        Box::new(CpuCollector { core_count: 8 }),
        Box::new(MemoryCollector { total_mb: 16384 }),
        // Could add DiskCollector, NetworkCollector, etc.
    ];

    for collector in &collectors {
        println!("{}", collector.report());
    }
}
```

`Box<dyn Collector>` is a "fat pointer" — it holds a pointer to the data and a pointer to a vtable (a table of function pointers). Each call goes through the vtable at runtime. This has a small overhead compared to static dispatch.

**When to use which:**

| Approach | Syntax | Dispatch | Use when |
|----------|--------|----------|----------|
| Static dispatch | `&impl Collector` or `<T: Collector>` | Compile-time, zero overhead | Most cases, especially hot paths |
| Dynamic dispatch | `&dyn Collector` or `Box<dyn Collector>` | Runtime vtable | Heterogeneous collections, plugin systems |

For the system monitor, a `Vec<Box<dyn Collector>>` is a natural way to hold all your collectors and iterate over them:

```rust
fn collect_all(collectors: &[Box<dyn Collector>]) -> Vec<(String, f64)> {
    collectors.iter()
        .map(|c| (c.name().to_string(), c.collect()))
        .collect()
}
```

---

## Implementing the Collector Trait for the Project

Here's a concrete preview of what your project will look like:

```rust
pub trait Collector {
    fn name(&self) -> &str;
    fn collect(&self) -> f64;

    fn report_line(&self) -> String {
        let value = self.collect();
        let status = if value > 90.0 { "[WARN]" } else { "[OK]  " };
        format!("{} {}: {:.1}", status, self.name(), value)
    }
}

pub struct CpuCollector;
pub struct MemCollector { pub total_mb: u64 }

impl Collector for CpuCollector {
    fn name(&self) -> &str { "CPU Usage %" }
    fn collect(&self) -> f64 {
        // Real: system.global_cpu_info().cpu_usage() as f64
        42.7
    }
}

impl Collector for MemCollector {
    fn name(&self) -> &str { "Memory Usage %" }
    fn collect(&self) -> f64 {
        // Real: (used / total) * 100
        62.5
    }
}

fn main() {
    let collectors: Vec<Box<dyn Collector>> = vec![
        Box::new(CpuCollector),
        Box::new(MemCollector { total_mb: 16384 }),
    ];

    println!("=== System Status ===");
    for c in &collectors {
        println!("{}", c.report_line());
    }
}
```

Output:
```
=== System Status ===
[OK]   CPU Usage %: 42.7
[OK]   Memory Usage %: 62.5
```

---

## How It Breaks

**Object safety violations — `dyn Trait` doesn't work with all traits.**
Not every trait can be used as `dyn Trait`. A trait is "object safe" only if all its methods can be called without knowing the concrete type at compile time. Two things break object safety:
- Methods that return `Self` — the compiler can't know the size of `Self` through a vtable
- Generic methods — each monomorphization would need a separate vtable entry

```rust
trait Bad {
    fn clone_me(&self) -> Self;  // returns Self — not object safe
    fn process<T>(&self, t: T);  // generic — not object safe
}

// Vec<Box<dyn Bad>> won't compile
```
If you need `dyn Trait`, design your trait without `Self` returns and without generic methods. Use associated types or specific concrete types instead.

**The orphan rule — you can't implement a foreign trait for a foreign type.**
You can implement your trait for someone else's type, or implement someone else's trait for your type. But you can't implement someone else's trait for someone else's type:
```rust
// Your trait, external type: OK
impl Collector for sysinfo::System { ... }

// External trait, your type: OK
impl std::fmt::Display for SystemSnapshot { ... }

// External trait, external type: COMPILE ERROR (orphan rule)
impl std::fmt::Display for sysinfo::System { ... }
```
The fix: the newtype pattern. Wrap the foreign type in your own struct:
```rust
struct MySysinfo(sysinfo::System);
impl std::fmt::Display for MySysinfo { ... }  // now it compiles
```

**Conflicting blanket implementations.**
A blanket implementation applies a trait to all types that meet some condition. Two blanket impls that overlap produce a "conflicting implementations" error. This most often happens when you try to implement a standard library trait for "any type that implements X" and the standard library already has a similar blanket impl. The error is confusing but the solution is usually to implement the trait for your concrete types instead of writing a blanket impl.

**Forgotten trait bounds in generic functions.**
When you write a generic function, you must declare every trait the type needs to satisfy:
```rust
fn print_metric<T>(collector: &T) {
    println!("{}", collector.report());  // COMPILE ERROR: no method named `report` found for type `T`
}

// Fix: add the bound
fn print_metric<T: Collector>(collector: &T) {
    println!("{}", collector.report());  // now OK
}
```
The compiler error tells you which method is missing and suggests adding the bound. This isn't a complex issue — just easy to forget when writing a quick generic function.

---

## Common Mistakes

**1. Implementing a trait method with the wrong signature.** If the trait says `fn collect(&self) -> f64` and you write `fn collect(&self) -> i32`, the compiler gives an error. The signature must match exactly.

**2. Trying to use dyn Trait without Box.** You can't store a `dyn Trait` directly in a Vec because its size isn't known at compile time. Wrap it in `Box<dyn Collector>`.

**3. Implementing Display without importing fmt.** You need `use std::fmt;` at the top of the file, and the method signature is `fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result`.

**4. Forgetting that derive macros only work if all fields support the trait.** If your struct has a `Vec<Something>`, you can derive `Debug` only if `Something` also implements `Debug`. If it doesn't, you need to implement `Debug` manually.

**5. Using Box<dyn Trait> everywhere out of habit.** Static dispatch is usually better. Only reach for `Box<dyn Trait>` when you genuinely need to store different types in the same collection, or when you're building plugin-style extensibility.
