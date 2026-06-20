# Doc 01: Data Structures - Vec, HashMap, and Structs

---

## Engineering Methodology: Data-Driven Design

Data-driven design means you define your data model before writing any functions. The shape of your data determines the shape of your code. Functions become obvious once you know what they operate on. If a function is hard to write, the data model is usually wrong.

For the system monitor: before you write a single line of Rust, answer these questions on paper:

- What is a "metric reading"? What fields does it have? What are the types of those fields?
- Is CPU usage a percentage (`f32`, `f64`) or raw ticks (`u64`)? Both exist in the OS - which does your program need?
- Is memory stored in bytes, kilobytes, or megabytes? Make a consistent choice and document it.
- What's the relationship between a CPU reading and a memory reading? Are they the same struct? Siblings in a parent struct? Completely separate?
- What's the structure for disk data? There are multiple disks. Is that a `Vec<DiskMetrics>` or a `HashMap<String, DiskMetrics>` keyed by mount point?

Draw the structs on paper first. If your structs feel right, the functions write themselves. Here's a starting point for your sketch:

```
CpuMetrics   → usage_percent: f64, core_count: u32, frequency_mhz: u64
MemoryMetrics → total_mb: u64, used_mb: u64, available_mb: u64, swap_used_mb: u64
DiskMetrics  → path: String, total_gb: u64, used_gb: u64, percent_full: u64
SystemSnapshot → timestamp + CpuMetrics + MemoryMetrics + Vec<DiskMetrics>
```

This is your data model. Once it exists, the question "how do I display disk usage?" has an obvious answer: iterate over `Vec<DiskMetrics>` and format each entry. The question "how do I look up a disk by mount point?" has an obvious answer: you can't with a `Vec` - change the structure to a `HashMap` if that lookup matters.

This habit - define data first, then functions - is one of the highest-leverage engineering practices you can build. Apply it to every feature you add for the rest of this course.

---

Section 1 gave you enough to build a hex viewer. You used basic types, slices, iterators, and Result/Option. Now you need richer data structures to hold CPU readings, memory stats, and disk info. This document covers the three workhorses you'll reach for constantly: `Vec<T>`, `HashMap<K, V>`, and structs.

---

## Vec<T> - The Dynamic Array

In C, you'd write something like:

```c
int *readings = malloc(n * sizeof(int));
// ... fill it ...
realloc(readings, new_size * sizeof(int));
free(readings);
```

Rust's `Vec<T>` does all of that for you, automatically, without leaks:

```rust
fn main() {
    let mut readings: Vec<f64> = Vec::new();

    readings.push(45.2);  // Add to the end
    readings.push(46.8);
    readings.push(47.1);

    println!("Count: {}", readings.len());         // 3
    println!("Capacity: {}", readings.capacity()); // 4 or more - Vec reserves extra

    // Safe indexing - returns an Option
    if let Some(first) = readings.get(0) {
        println!("First reading: {}", first);
    }

    // Direct indexing - panics if out of bounds (like C, but controlled)
    println!("Last: {}", readings[readings.len() - 1]);

    // Remove from the end and get the value back
    if let Some(last) = readings.pop() {
        println!("Popped: {}", last);
    }
}
```

**len() vs capacity():** `len()` is how many elements you've pushed. `capacity()` is how much space is allocated. When you push past capacity, Vec doubles its allocation. This is the same growth strategy as C++'s `std::vector`. You almost never need to think about capacity, but knowing it exists helps you understand why Vec is efficient.

**The vec! macro** is shorthand for initialization:

```rust
let temps = vec![72.1, 68.4, 75.3]; // already has 3 elements
let zeroes = vec![0.0f64; 10];       // 10 elements, all 0.0
```

**Iterating** - prefer `&v` to avoid consuming the Vec:

```rust
let cpu_samples = vec![40.0, 41.5, 42.1, 38.9];

for sample in &cpu_samples {
    println!("CPU: {:.1}%", sample);
}

// cpu_samples is still usable here
println!("Total samples: {}", cpu_samples.len());
```

---

## HashMap<K, V> - The Hash Table

`HashMap` is like C's hash table, but built into the standard library with a safe interface. You need to explicitly import it:

```rust
use std::collections::HashMap;

fn main() {
    let mut disk_usage: HashMap<String, u64> = HashMap::new();

    // Insert key-value pairs
    disk_usage.insert("/".to_string(), 85);
    disk_usage.insert("/home".to_string(), 42);
    disk_usage.insert("/tmp".to_string(), 3);

    // Look up a key - returns Option<&V>
    match disk_usage.get("/home") {
        Some(pct) => println!("/home is {}% full", pct),
        None => println!("/home not found"),
    }

    // Check if a key exists
    if disk_usage.contains_key("/tmp") {
        println!("/tmp is being tracked");
    }

    // Iterate over all pairs (order is not guaranteed)
    for (mount, pct) in &disk_usage {
        println!("{}: {}%", mount, pct);
    }
}
```

### 🟡 The entry API - idiomatic and important

One of the most common patterns in Rust: update a value if it exists, or insert a default if it doesn't. The entry API handles both cases without a double lookup:

```rust
use std::collections::HashMap;

fn main() {
    let mut hit_counts: HashMap<String, u32> = HashMap::new();

    let events = vec!["cpu", "mem", "cpu", "disk", "cpu", "mem"];

    for event in events {
        // If the key exists, return a mutable ref to the value.
        // If not, insert 0 first, then return a mutable ref.
        let count = hit_counts.entry(event.to_string()).or_insert(0);
        *count += 1;
    }

    println!("{:?}", hit_counts);
    // {"cpu": 3, "mem": 2, "disk": 1} (order may vary)
}
```

This pattern replaces the C idiom of `if (find) update; else insert;`. Learn it - you'll use it constantly.

---

## Structs - Grouping Related Data

You already saw basic structs in Section 1. Let's go deeper.

```rust
#[derive(Debug, Clone, PartialEq)]
struct CpuMetrics {
    core_count: u32,
    usage_percent: f64,
    frequency_mhz: u64,
}
```

### impl blocks - methods and associated functions

An `impl` block is where you put functions that belong to your struct. Two kinds:

1. **Methods** take `&self` or `&mut self` - they operate on an instance
2. **Associated functions** take no `self` - they're called on the type itself, like constructors

```rust
#[derive(Debug)]
struct CpuMetrics {
    core_count: u32,
    usage_percent: f64,
    frequency_mhz: u64,
}

impl CpuMetrics {
    // Associated function - called as CpuMetrics::new(...)
    // Like a constructor. No self parameter.
    pub fn new(core_count: u32) -> Self {
        CpuMetrics {
            core_count,
            usage_percent: 0.0,
            frequency_mhz: 0,
        }
    }

    // Method - called on an instance as metrics.update(...)
    // &mut self means we can modify the struct
    pub fn update(&mut self, usage: f64, freq: u64) {
        self.usage_percent = usage;
        self.frequency_mhz = freq;
    }

    // Method - &self means we only read, not modify
    pub fn is_overloaded(&self) -> bool {
        self.usage_percent > 90.0
    }

    // Method that returns a formatted string
    pub fn summary(&self) -> String {
        format!(
            "{} cores @ {} MHz, {:.1}% used",
            self.core_count, self.frequency_mhz, self.usage_percent
        )
    }
}

fn main() {
    let mut cpu = CpuMetrics::new(8);
    cpu.update(73.5, 3400);
    println!("{}", cpu.summary());
    println!("Overloaded: {}", cpu.is_overloaded());
}
```

In C, you'd pass a pointer to the struct explicitly: `void update(CpuMetrics *m, double usage)`. In Rust, `&mut self` handles that automatically.

### Public vs private fields

By default, struct fields are private to the module they're defined in. You control visibility with `pub`:

```rust
pub struct MemoryMetrics {
    pub total_mb: u64,        // visible everywhere
    pub used_mb: u64,         // visible everywhere
    cached_mb: u64,           // private - only accessible within this module
}
```

If a field is private, code outside the module can't read or write it directly. They have to go through your methods. This is how you enforce invariants.

### Tuple structs - brief but useful

When you want to wrap a primitive to give it a distinct type:

```rust
struct Megabytes(u64);
struct Milliseconds(u64);

fn sleep_for(ms: Milliseconds) { /* ... */ }

let timeout = Milliseconds(500);
sleep_for(timeout);
// sleep_for(Megabytes(500));  // Won't compile - types don't match
```

Access the inner value with `.0`: `timeout.0` gives you the `u64`. Tuple structs are great for preventing you from passing memory amounts where time is expected.

### Derive macros - free implementations

The `#[derive(...)]` attribute auto-generates trait implementations. You'll use these constantly:

```rust
#[derive(Debug, Clone, PartialEq)]
struct DiskInfo {
    mount_point: String,
    total_gb: u64,
    used_gb: u64,
}
```

- **`Debug`** - lets you print with `{:?}` and `{:#?}`. Add this to every struct you create. Debugging without it is painful.
- **`Clone`** - lets you make an explicit copy: `let copy = original.clone()`. Without this you'd have to implement it by hand.
- **`PartialEq`** - lets you compare with `==` and `!=`. Without it `if a == b` doesn't compile for your struct.

You don't need to know how they work internally yet. Just know when to use them: `Debug` on almost everything, `Clone` when you need to copy values, `PartialEq` when you need to compare values.

---

## Putting It Together

Here's a small preview of what your system monitor will look like once you add methods:

```rust
use std::collections::HashMap;

#[derive(Debug)]
struct SystemSnapshot {
    cpu_usage: f64,
    memory_used_mb: u64,
    memory_total_mb: u64,
    disk_usage: HashMap<String, u64>, // mount -> percent full
}

impl SystemSnapshot {
    pub fn new() -> Self {
        SystemSnapshot {
            cpu_usage: 0.0,
            memory_used_mb: 0,
            memory_total_mb: 0,
            disk_usage: HashMap::new(),
        }
    }

    pub fn memory_percent(&self) -> f64 {
        if self.memory_total_mb == 0 {
            return 0.0;
        }
        (self.memory_used_mb as f64 / self.memory_total_mb as f64) * 100.0
    }

    pub fn add_disk(&mut self, mount: &str, percent: u64) {
        self.disk_usage.insert(mount.to_string(), percent);
    }
}

fn main() {
    let mut snapshot = SystemSnapshot::new();
    snapshot.cpu_usage = 42.7;
    snapshot.memory_used_mb = 6144;
    snapshot.memory_total_mb = 16384;
    snapshot.add_disk("/", 55);
    snapshot.add_disk("/home", 72);

    println!("{:.1}% memory used", snapshot.memory_percent());
    println!("{:#?}", snapshot);
}
```

---

## How It Breaks

**Vec: index out of bounds panics at runtime.**
`v[i]` panics if `i >= v.len()`. In C this is undefined behavior; in Rust it's a controlled panic with a message and line number - better, but still a crash. In production code that indexes into a Vec based on external data, use `.get(i)` which returns `Option<&T>`:
```rust
// unsafe in the sense of "will crash on bad index"
let first = readings[0];

// production-safe
let first = readings.get(0).copied().unwrap_or(0.0);
```

**Vec growing and invalidating references.**
You can't hold a reference to an element of a `Vec` while also pushing to it. The push might reallocate the underlying buffer, making all existing references dangle. The borrow checker prevents this:
```rust
let mut v = vec![1.0, 2.0];
let first = &v[0];   // borrow of element
v.push(3.0);         // COMPILE ERROR: cannot borrow `v` as mutable because it is also borrowed as immutable
println!("{}", first);
```
If you need to keep a reference after modification, index into the Vec again after the modification.

**HashMap: iteration order is not guaranteed.**
`HashMap` does not preserve insertion order. If you insert "/", "/home", "/tmp" in that order and then iterate, you might see them in any order. Tests that check for a specific ordering of HashMap output will flake. If you need consistent order (alphabetical mount points, for example), collect into a `Vec` and sort:
```rust
let mut mounts: Vec<_> = disk_map.iter().collect();
mounts.sort_by_key(|(path, _)| path.as_str());
```

**Modifying a HashMap while iterating.**
You can't call `.insert()` or `.remove()` while you have an iterator over the same map - the borrow checker prevents it. Collect the keys you want to modify first, then loop over them to update:
```rust
let keys_to_update: Vec<String> = map.keys().cloned().collect();
for key in keys_to_update {
    map.insert(key, new_value);
}
```

---

## Common Mistakes

**1. Iterating and consuming.** If you use `for x in my_vec` you move the Vec and can't use it again. Use `for x in &my_vec` to borrow it.

**2. Indexing with the wrong type.** Vec indices must be `usize`. If you have a counter as `i32`, you'll get a compile error. Use `i as usize` to convert.

**3. Forgetting `use std::collections::HashMap`.** Vec is in the prelude (automatically imported). HashMap is not. You need the `use` statement.

**4. Mutating through a non-mutable method.** If you write a method that needs to change the struct, it must take `&mut self` and the binding must be `let mut`.

**5. Using `entry().or_insert()` and forgetting to dereference.** The value returned from `or_insert` is `&mut V`. You need `*count += 1`, not `count += 1`.
