# 06 - Rust Idioms and Best Practices

> **Project:** System Monitor - polling CPU, memory, and disk metrics at intervals and displaying them in the terminal.

## In This Section

You are building a system monitor that queries OS metrics via the `sysinfo` crate, stores results in structs, and formats them into a live-updating terminal display. Three idioms from this document apply directly:

- **Builder pattern** - `MonitorConfig` will grow optional fields quickly (poll interval, display mode, metric filters). Use a builder instead of a positional constructor.
- **Type aliases** - iterator chains over `sysinfo` data produce complex intermediate types. A `type CpuSnapshot = Vec<(String, f32)>` is clearer than the raw type in every function signature.
- **Owned types in structs, references in functions** - your metric structs own `String` names and `Vec` histories; your formatting functions borrow slices of that data.

---

## 1. Naming Conventions

The Rust community follows these naming conventions. Your code looks wrong to any Rust developer if you don't:

| What | Convention | Example |
|------|-----------|---------|
| Variables, functions, methods | `snake_case` | `poll_metrics`, `cpu_usage` |
| Types, structs, enums, traits | `PascalCase` | `CpuMetric`, `DiskUsage`, `MetricError` |
| Constants, statics | `UPPER_SNAKE_CASE` | `DEFAULT_POLL_SECS`, `MAX_HISTORY_LEN` |
| Modules | `snake_case` | `mod metrics`, `mod display` |
| Enum variants | `PascalCase` | `Ok`, `Err`, `Some`, `None`, `PermissionDenied` |

Do not use `camelCase` for variables (that is JavaScript). Do not use `ALL_CAPS` for types.

---

## 2. Prefer Owned Types in Structs, References in Function Signatures

A C developer's instinct: "use a pointer to avoid copying." In Rust, this leads to lifetime headaches.

**Rule: Structs should own their data.** Use `String`, not `&str`. Use `Vec<T>`, not `&[T]`.

**Rule: Functions should borrow.** Function parameters should be `&str` (not `String`), `&[T]` (not `Vec<T>`), `&T` (not `T`) unless you need to store or transfer ownership.

Why this works: `String` can be passed as `&str` automatically (deref coercion). `Vec<T>` can be passed as `&[T]` automatically. The reverse is not true - you cannot pass a `&str` where a `String` is needed without cloning.

In the system monitor, a `CpuMetric` struct owns its name and history buffer, but the rendering function borrows a slice:

```rust
struct CpuMetric {
    name: String,         // owned - this struct is the source of truth
    history: Vec<f32>,    // owned - we accumulate data here
}

// Good: borrow a slice of the history, return owned display string
fn render_sparkline(history: &[f32]) -> String {
    // ...
}

// Bad: takes Vec - forces the caller to clone or give up the buffer
fn render_sparkline(history: Vec<f32>) -> String {
    // ...
}
```

---

## 3. The Builder Pattern for Complex Construction

When a struct has many optional fields, avoid a constructor with 7 parameters. Use a builder:

```rust
// Bad: hard to read at call site, fragile to change
let cfg = MonitorConfig::new(1, true, false, Some(10), None, true);

// Good: self-documenting, easy to extend without breaking call sites
let cfg = MonitorConfig::builder()
    .poll_interval_secs(1)
    .show_cpu(true)
    .show_disk(false)
    .history_depth(10)
    .build()?;
```

`MonitorConfig` is a natural fit here because "which metrics to display" and "how often to poll" are independent knobs that users will configure differently. You will use this same pattern when constructing `reqwest` HTTP clients, Tokio runtimes, and database pools in later sections.

---

## 4. Error Handling Hierarchy

**Do not use:** `unwrap()` or `expect()` in production paths - anything that can fail due to OS permissions, missing hardware, or bad configuration.

**Early development:** `Box<dyn std::error::Error>` as the error type. Gets you started without designing an error type first.

**Applications (tools, services):** `anyhow::Error`. Context-aware, stackable. The `anyhow::Context` trait lets you attach messages.

**Libraries:** `thiserror` with a typed enum. Callers need to match on variants; `anyhow` would hide them.

The pattern:

```rust
fn run() -> anyhow::Result<()> {
    let cfg = MonitorConfig::builder()
        .poll_interval_secs(1)
        .build()
        .context("failed to build monitor config")?;

    loop {
        let snapshot = poll_metrics(&cfg)
            .context("failed to read system metrics")?;
        render(&snapshot);
        std::thread::sleep(cfg.poll_interval());
    }
}

fn main() {
    if let Err(e) = run() {
        eprintln!("error: {e:#}");
        std::process::exit(1);
    }
}
```

---

## 5. Don't Match on Bool - Use the Right Abstraction

```rust
// Bad: checking is_some() then calling unwrap()
if metric.cpu_name.is_some() {
    let name = metric.cpu_name.unwrap();
    println!("{name}");
}

// Good: if let destructures in one step
if let Some(name) = &metric.cpu_name {
    println!("{name}");
}

// Bad: manually re-wrapping an error
if result.is_err() {
    return Err(result.unwrap_err());
}

// Good: ? propagates automatically
let snapshot = poll_metrics(&cfg)?;
```

---

## 6. Closures as First-Class Values

In C, function pointers do what closures do, but closures also capture their environment. You will use closures constantly when transforming metric data with iterators:

```rust
let usage_above_threshold: Vec<&CpuMetric> = metrics
    .iter()
    .filter(|m| m.usage > threshold)  // closure captures `threshold`
    .collect();
```

Know the three closure traits:

- `Fn` - can be called multiple times, borrows captured variables
- `FnMut` - can be called multiple times, mutably borrows captured variables
- `FnOnce` - can only be called once, takes ownership of captured variables

When you pass a closure to `thread::spawn` or `tokio::spawn` (in later sections), it must be `FnOnce + Send + 'static`. That `'static` requirement is what forces you to use `Arc::clone` instead of passing a raw reference - the spawned thread may outlive the stack frame where the reference lives.

---

## 7. The Type Alias Habit

System monitoring code tends to accumulate complex types fast. If you see a type written out more than once, alias it:

```rust
// Without alias - appears in every function signature and struct field
fn render(metrics: &HashMap<String, Vec<f32>>) { ... }
fn aggregate(metrics: &HashMap<String, Vec<f32>>) -> HashMap<String, f32> { ... }

// With alias - reads as intent
type MetricHistory = HashMap<String, Vec<f32>>;

fn render(metrics: &MetricHistory) { ... }
fn aggregate(metrics: &MetricHistory) -> HashMap<String, f32> { ... }
```

You will see this pattern used for complex `Result` types as well:

```rust
type Result<T> = std::result::Result<T, AppError>;
```

---

## 8. Zero-Cost Abstractions: When to Use Them

Rust's iterators, closures, and generics compile down to the same assembly as hand-written loops. Prefer them over manual index loops not just for readability, but because the compiler can optimize them better.

```rust
// Avoid: manual index loop with runtime bounds checks on every iteration
let mut total = 0.0f32;
for i in 0..cpu_usages.len() {
    total += cpu_usages[i];
}
let avg = total / cpu_usages.len() as f32;

// Prefer: iterator - same performance, clearer intent
let avg = cpu_usages.iter().sum::<f32>() / cpu_usages.len() as f32;
```

Use generics over `Box<dyn Trait>` when the concrete metric type is known at compile time. Use `Box<dyn Trait>` when you need a heterogeneous list of metric collectors chosen at runtime - for example, a plugin system where disk, network, and CPU collectors share a common `Collector` trait.

---

## 9. Testing: Write Tests in the Same File

Rust's `#[cfg(test)]` module lets you put unit tests directly in the same file as the code they test. Do not create a separate test file for unit tests.

For the system monitor, pure transformation functions - not the OS polling itself - are what you unit test. Isolate the logic:

```rust
pub fn average(samples: &[f32]) -> Option<f32> {
    if samples.is_empty() {
        return None;
    }
    Some(samples.iter().sum::<f32>() / samples.len() as f32)
}

pub fn format_bytes(bytes: u64) -> String {
    const GB: u64 = 1 << 30;
    const MB: u64 = 1 << 20;
    if bytes >= GB {
        format!("{:.1} GB", bytes as f64 / GB as f64)
    } else {
        format!("{:.1} MB", bytes as f64 / MB as f64)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn average_empty_is_none() {
        assert_eq!(average(&[]), None);
    }

    #[test]
    fn average_single_value() {
        assert_eq!(average(&[42.0]), Some(42.0));
    }

    #[test]
    fn format_bytes_gigabytes() {
        assert_eq!(format_bytes(2 * (1 << 30)), "2.0 GB");
    }
}
```

Run with `cargo test`. No extra framework or configuration needed.

---

## 10. Clippy: Your Automated Code Reviewer

`cargo clippy` is Rust's linter. Run it constantly. It catches:

- `if x == true` should be `if x`
- `vec.len() == 0` should be `vec.is_empty()`
- `for i in 0..vec.len() { vec[i] }` should use `.iter()` or `.iter().enumerate()`
- Unnecessary `.clone()` calls
- `match x { true => ..., false => ... }` should be `if x { ... } else { ... }`
- Hundreds more

Add this to `Cargo.toml` to treat Clippy warnings as build errors:

```toml
[lints.clippy]
all = "warn"
```

Run `cargo clippy -- -D warnings` in CI to enforce this automatically.
