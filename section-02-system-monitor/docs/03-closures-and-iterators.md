# Doc 03: Closures and Iterators

C has function pointers. They're useful but they can't capture local variables without passing a context struct around manually. Rust closures can capture their environment directly. Combined with iterators, they let you write data processing pipelines that are cleaner than any C loop and — critically — just as fast.

---

## Closures — Anonymous Functions That Capture Their Environment

The syntax uses `||` instead of `()`:

```rust
fn add_one(x: u32) -> u32 { x + 1 }       // Regular function

let add_one_closure = |x: u32| x + 1;       // Equivalent closure, inline

// Types can be inferred from context
let double = |x| x * 2;

println!("{}", add_one(5));            // 6
println!("{}", add_one_closure(5));    // 6
println!("{}", double(5i32));          // 10
```

The real power: closures capture variables from the surrounding scope. No context pointer needed.

```rust
fn make_adder(amount: u32) -> impl Fn(u32) -> u32 {
    // `amount` is captured by the closure
    move |x| x + amount
}

let add_ten = make_adder(10);
let add_hundred = make_adder(100);

println!("{}", add_ten(5));     // 15
println!("{}", add_hundred(5)); // 105
```

In C, you'd achieve this with:

```c
typedef struct { uint32_t amount; } AdderCtx;
uint32_t adder(AdderCtx *ctx, uint32_t x) { return ctx->amount + x; }
```

Rust closures handle that bookkeeping automatically.

---

## How Closures Capture: By Reference vs By Value

By default, Rust closures borrow what they capture:

```rust
let threshold = 80.0f64;

// `threshold` is borrowed — the closure holds &f64
let is_high = |pct: f64| pct > threshold;

println!("{}", is_high(75.0));  // false
println!("{}", is_high(85.0));  // true
println!("threshold still here: {}", threshold); // OK — it was only borrowed
```

When you need the closure to own its captured data (e.g., when passing it to a thread), use `move`:

```rust
let label = String::from("CPU");

// `label` is MOVED into the closure — the closure owns it
let printer = move || println!("Metric: {}", label);

// label is gone from this scope
// println!("{}", label); // Would not compile

printer(); // "Metric: CPU"
```

`move` closures are required when the closure outlives the scope where the captured data was defined — threads are the most common case.

---

## 🟡 The Three Closure Traits

Rust has three traits that closures implement, depending on how they use their captured data:

| Trait     | Can call | Captured data |
|-----------|----------|---------------|
| `FnOnce`  | Once     | Consumes captured values (moves them out) |
| `FnMut`   | Many times, with mutation | Mutates captured values |
| `Fn`      | Many times, read-only | Only reads captured values |

The compiler figures out which trait to give your closure automatically. You mostly encounter these when writing function signatures that accept closures as parameters:

```rust
// Accepts any closure that can be called repeatedly (most common)
fn apply_twice(f: impl Fn(f64) -> f64, x: f64) -> f64 {
    f(f(x))
}

// Accepts a closure that can mutate captured state
fn run_with_counter(mut f: impl FnMut() -> ()) {
    f();
    f();
    f();
}

let mut count = 0;
run_with_counter(|| count += 1);
println!("Called 3 times: {}", count); // 3
```

For the system monitor, you'll mostly use `Fn` — closures that filter, transform, and display data without mutating anything.

---

## Iterators — Processing Collections Without Loops

In C, you'd process an array of CPU readings like this:

```c
double sum = 0.0;
int count = 0;
for (int i = 0; i < n; i++) {
    if (readings[i] > 0.0) {
        sum += readings[i];
        count++;
    }
}
double avg = (count > 0) ? sum / count : 0.0;
```

In Rust, using iterators:

```rust
let readings = vec![0.0, 45.2, 0.0, 62.1, 71.4, 0.0, 55.8];

let avg = {
    let valid: Vec<f64> = readings.iter()
        .copied()                        // &f64 -> f64
        .filter(|&r| r > 0.0)           // skip zeros
        .collect();

    if valid.is_empty() {
        0.0
    } else {
        valid.iter().sum::<f64>() / valid.len() as f64
    }
};

println!("Average (non-zero): {:.1}", avg); // 58.9
```

Both versions do the same work. The iterator version is not slower — the compiler optimizes it to the same machine code as the loop.

---

## iter() vs into_iter() vs iter_mut()

This trips people up. Three different methods, three different semantics:

```rust
let v = vec![1, 2, 3, 4, 5];

// iter() — borrows each element as &T (most common)
// v is still usable after
for x in v.iter() {
    println!("{}", x);  // x is &i32
}
println!("v still here: {:?}", v);

// into_iter() — moves the collection, yields owned T
// v is consumed and no longer usable
for x in v.into_iter() {
    println!("{}", x);  // x is i32
}
// println!("{:?}", v); // Would not compile — v was moved

let mut v2 = vec![1, 2, 3];
// iter_mut() — borrows each element as &mut T, allows modification
for x in v2.iter_mut() {
    *x *= 2;  // double each element in place
}
println!("{:?}", v2); // [2, 4, 6]
```

Rule of thumb: use `iter()` unless you specifically need ownership (`into_iter()`) or in-place mutation (`iter_mut()`).

---

## The Core Adapters

These are the ones you'll use every day:

### map — transform each element

```rust
let mb_values = vec![1024u64, 2048, 4096, 8192];

let gb_values: Vec<f64> = mb_values.iter()
    .map(|&mb| mb as f64 / 1024.0)
    .collect();

println!("{:?}", gb_values); // [1.0, 2.0, 4.0, 8.0]
```

### filter — keep only matching elements

```rust
let processes = vec![
    ("sshd", 0.1),
    ("firefox", 45.2),
    ("rustc", 82.7),
    ("systemd", 0.0),
    ("cargo", 91.3),
];

let heavy: Vec<_> = processes.iter()
    .filter(|(_, cpu)| *cpu > 40.0)
    .collect();

for (name, cpu) in &heavy {
    println!("{}: {:.1}%", name, cpu);
}
// firefox: 45.2%
// rustc: 82.7%
// cargo: 91.3%
```

### collect — materialize the results

`collect()` is lazy — the whole chain doesn't run until you call it. It needs a type hint:

```rust
// Type annotation on the variable
let names: Vec<&str> = processes.iter()
    .map(|(name, _)| *name)
    .collect();

// Or turbofish syntax on collect
let names = processes.iter()
    .map(|(name, _)| *name)
    .collect::<Vec<_>>();
```

### any / all / find

```rust
let cpus = vec![42.1, 67.3, 88.9, 55.0];

let has_critical = cpus.iter().any(|&c| c > 85.0);      // true
let all_healthy  = cpus.iter().all(|&c| c < 90.0);      // true
let first_high   = cpus.iter().find(|&&c| c > 60.0);    // Some(67.3)

println!("Critical: {}", has_critical);
println!("All healthy: {}", all_healthy);
println!("First high: {:?}", first_high);
```

### fold — reduce to a single value

The general accumulation tool:

```rust
let disk_sizes = vec![("root", 256u64), ("/home", 512), ("/tmp", 32)];

let total_gb = disk_sizes.iter()
    .fold(0u64, |acc, (_, size)| acc + size);

println!("Total: {} GB", total_gb); // 800
```

---

## enumerate and zip

### enumerate — index + value together

Replaces `for (int i = 0; ...)`:

```rust
let mount_points = vec!["/", "/home", "/var", "/tmp"];

for (i, mount) in mount_points.iter().enumerate() {
    println!("[{}] {}", i, mount);
}
// [0] /
// [1] /home
// [2] /var
// [3] /tmp
```

### zip — pair elements from two collections

```rust
let names = vec!["cpu_usage", "mem_used", "disk_io"];
let values = vec![42.1, 7168.0, 125.3];

let pairs: Vec<String> = names.iter()
    .zip(values.iter())
    .map(|(name, val)| format!("{}: {:.1}", name, val))
    .collect();

for line in &pairs {
    println!("{}", line);
}
// cpu_usage: 42.1
// mem_used: 7168.0
// disk_io: 125.3
```

`zip` stops at the shorter of the two iterators — no off-by-one errors possible.

---

## 🔴 Why Iterators Are Zero-Cost

You might wonder: all these `.map()` and `.filter()` calls — don't they add overhead? Each adapter is a new struct that wraps the previous one. Doesn't that mean extra function calls, extra indirection?

No. Rust's iterators are zero-cost abstractions. The compiler inlines every closure and adapter call. The resulting machine code is identical to a hand-written `for` loop with manual index tracking. You can verify this by looking at compiler output — there's no extra allocation, no vtable, no indirection.

The key is that all the types are known at compile time. The compiler generates a specialized version of every iterator chain for the exact types involved. This is called monomorphization, and you'll learn more about it in the traits document.

---

## A Real Pipeline: Processing System Metrics

Compare C-style to iterator style:

```rust
// Given: a list of (process_name, cpu_percent, memory_mb)
let processes = vec![
    ("bash",    0.1,   8u64),
    ("firefox", 45.2, 512),
    ("rustc",   82.7, 256),
    ("systemd",  0.0,  16),
    ("cargo",   91.3, 384),
];

// C-style:
let mut high_mem_names = Vec::new();
for (name, cpu, mem) in &processes {
    if *cpu > 10.0 && *mem > 200 {
        high_mem_names.push(format!("{} ({} MB)", name, mem));
    }
}

// Iterator style (same result):
let high_mem_names: Vec<String> = processes.iter()
    .filter(|(_, cpu, mem)| *cpu > 10.0 && *mem > 200)
    .map(|(name, _, mem)| format!("{} ({} MB)", name, mem))
    .collect();

println!("{:?}", high_mem_names);
// ["firefox (512 MB)", "rustc (256 MB)", "cargo (384 MB)"]
```

The iterator version reads as a pipeline: take the data, keep only the interesting rows, format each row, collect the results. Once you're comfortable with this style, the C-style loop feels like it's burying the intent inside imperative details.

---

## Implementing Iterator Yourself

You've been using iterators. Now implement one. The `Iterator` trait requires exactly two things:

```rust
struct MetricWindow {
    values: Vec<f64>,
    pos: usize,
}

impl MetricWindow {
    fn new(values: Vec<f64>) -> Self {
        MetricWindow { values, pos: 0 }
    }
}

impl Iterator for MetricWindow {
    type Item = f64;  // what each call to next() produces

    fn next(&mut self) -> Option<Self::Item> {
        if self.pos < self.values.len() {
            let val = self.values[self.pos];
            self.pos += 1;
            Some(val)
        } else {
            None  // signals the iterator is exhausted
        }
    }
}
```

That's it — `type Item` and `fn next`. The moment you implement those two things, **every iterator adapter in the standard library is available for free**: `.map()`, `.filter()`, `.sum()`, `.max_by()`, `.enumerate()`, `.zip()`, `.take()`, `.skip()` — all of them work with no extra code.

```rust
let window = MetricWindow::new(vec![42.3, 80.1, 55.0, 91.7, 38.2]);

// All standard adapters work automatically
let critical: Vec<f64> = window
    .filter(|&v| v > 80.0)
    .collect();

println!("{:?}", critical);  // [80.1, 91.7]
```

This is how Rust builds a large API from a small core. You implement two methods; the trait provides 80+ adapters. The same pattern appears throughout the standard library — `Read`, `Write`, `Display` — implement one or two required methods, get rich behavior.

---

## How It Breaks

**Iterator adapters are lazy — nothing runs until you consume.**
This is the most common iterator mistake:
```rust
let readings = vec![40.0, 50.0, 60.0];

readings.iter()
    .filter(|&&r| r > 45.0)
    .map(|&r| r * 2.0);
// Nothing happened. No computation ran. No error. No warning.
```
The chain builds a description of the transformation, but executes nothing. You need a consuming call at the end: `.collect()`, `.for_each()`, `.sum()`, `.count()`, `.any()`, `.all()`, etc. Without one, the pipeline is created and immediately dropped.

**Calling `.collect()` without a type annotation.**
The compiler needs to know what type to collect into:
```rust
let names = processes.iter()
    .map(|(name, _)| name)
    .collect();  // COMPILE ERROR: type must be known at this point
```
Fix with a type annotation or turbofish:
```rust
let names: Vec<&str> = processes.iter()
    .map(|(name, _)| name)
    .collect();

// or
let names = processes.iter()
    .map(|(name, _)| name)
    .collect::<Vec<_>>();  // Vec<_> lets Rust infer the element type
```

**Infinite iterators without `.take(n)`.**
Some iterators produce values indefinitely:
```rust
let endless = std::iter::repeat(0.0f64);          // never ends
let cycle = vec![1, 2, 3].into_iter().cycle();    // repeats forever
```
Calling `.collect()` on either of these will run forever (or until memory is exhausted). Always chain `.take(n)` to cap infinite iterators:
```rust
let ten_zeros: Vec<f64> = std::iter::repeat(0.0).take(10).collect();
```

**Collecting into the wrong type.**
`.collect()` can produce different types (Vec, HashSet, String, HashMap, etc.) from the same iterator. If you write `collect::<Vec<String>>()` but your iterator yields `&str`, you'll get a type mismatch. If you want to collect string slices into owned strings, add `.map(|s| s.to_string())` before collect, or use `.collect::<Vec<&str>>()`:
```rust
let mounts = vec!["/", "/home", "/tmp"];

// This fails — &str is not String
let owned: Vec<String> = mounts.iter().collect();  // ERROR

// This works
let owned: Vec<String> = mounts.iter().map(|s| s.to_string()).collect();

// This also works — keep them as &str
let borrowed: Vec<&&str> = mounts.iter().collect();
```

---

## Common Mistakes

**1. Forgetting to call collect().** `.map()` and `.filter()` are lazy — they don't do anything by themselves. You need a consuming operation (`.collect()`, `.for_each()`, `.sum()`, etc.) at the end.

**2. Missing a type hint for collect().** The compiler can't always infer what collection type you want. Add a type annotation on the variable or use turbofish: `.collect::<Vec<_>>()`.

**3. Using iter() when you need into_iter().** If your closure needs to own the value (e.g., to move a String into a new struct), `iter()` gives you a reference which you can't move. Switch to `into_iter()`.

**4. Trying to modify the original collection with iter().** `iter()` gives you `&T`, which is immutable. Use `iter_mut()` for in-place changes.

**5. Confusion about move closures in multi-threaded code.** If you pass a closure to a new thread, it must be `move` so it owns its captured data. The compiler will tell you exactly when this is required.
