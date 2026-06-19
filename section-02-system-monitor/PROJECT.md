# Section 02 Project: sysmon — CLI System Resource Monitor

## What You're Building

A command-line tool called `sysmon` that monitors the current machine's resources and displays them in a clean, readable format. Think of it as a simplified version of `top` or `htop` — but one you built yourself, in Rust.

The tool has two modes:

1. **Snapshot mode** (default): collects current stats once and prints them, then exits.
2. **Watch mode** (`--watch`): clears the terminal and reprints updated stats every N seconds until you press Ctrl+C.

Sample output (snapshot mode):

```
=== System Monitor ===
Timestamp: 2024-01-15 14:32:07

  CPU Usage:     42.7%
  Memory:        6.2 GB / 16.0 GB  (38.7%)

  Disk Usage:
    /              45%  [################..........]
    /home          72%  [##########################..........]
    /tmp            3%  [#..................................]

Press Ctrl+C to exit (or run without --watch for single snapshot)
```

---

## Before You Write Code

Stop. Don't open your editor yet. Answer these questions on paper or in a text file:

1. **What is the data model?**
   - What information does the program need to hold? What types represent CPU data? Memory data? Disk data?
   - Should these be separate structs or one combined struct?
   - What fields does each struct need?

2. **What modules do I need?**
   - Which parts of the program are conceptually separate?
   - What does each module expose (public interface) vs. hide (implementation details)?
   - Where does the `Collector` trait live? Where does the `Metrics` struct live?

3. **What is the flow of the program?**
   - Draw it as a sequence of steps. What happens first? What happens last?
   - In watch mode, what changes compared to snapshot mode?
   - Where does argument parsing happen? Where does output happen?

4. **What can go wrong?**
   - What if sysinfo fails to read a sensor?
   - What if the output file can't be created?
   - How does the program handle these failures — panic, or return an error?

If you can answer these clearly before writing a line of code, the implementation will go much faster. If you can't answer them, you'll discover the confusion through compile errors.

---

## Architecture

Here's the intended architecture. You don't have to follow it exactly, but it's designed so each concept from this section's docs maps to a concrete part of the project.

```
sysmon/
├── Cargo.toml
└── src/
    ├── main.rs       <- argument parsing, main loop, coordinates everything
    ├── cpu.rs        <- CpuCollector, reads CPU usage via sysinfo
    ├── memory.rs     <- MemoryCollector, reads RAM stats via sysinfo
    ├── disk.rs       <- DiskCollector, reads per-mount-point disk usage
    └── display.rs    <- Display impl for Metrics, ASCII bar rendering
```

### The Collector Trait

Define this trait (probably in `main.rs` or a `types.rs` file you create):

```rust
pub trait Collector {
    fn name(&self) -> &str;
    fn collect(&self) -> f64;  // Returns a percentage (0.0 - 100.0)
}
```

Each module (`cpu.rs`, `memory.rs`, `disk.rs`) has a struct that implements this trait.

### The Metrics Struct

```rust
pub struct Metrics {
    pub cpu_percent: f64,
    pub memory_used_mb: u64,
    pub memory_total_mb: u64,
    pub disks: HashMap<String, DiskInfo>,   // mount point -> disk info
    pub timestamp: String,
}

pub struct DiskInfo {
    pub total_gb: u64,
    pub used_gb: u64,
    pub percent: u64,
}
```

This struct holds one complete snapshot of the system state. The program fills it, then passes it to the display and (optionally) JSON output functions.

### Main Loop (watch mode)

```
loop:
    collect all metrics -> Metrics struct
    clear terminal
    display metrics
    sleep N seconds
    (repeat)
```

---

## Requirements

### Command-line Interface

```
sysmon [OPTIONS]

Options:
  --watch              Continuously monitor (refresh until Ctrl+C)
  --interval <N>       Refresh interval in seconds [default: 2]
  --output <FILE>      Write JSON output to file
  -h, --help           Print help
  -V, --version        Print version
```

Examples:
```bash
sysmon                          # Single snapshot, printed to terminal
sysmon --watch                  # Refresh every 2 seconds
sysmon --watch --interval 5     # Refresh every 5 seconds
sysmon --output metrics.json    # Single snapshot written as JSON
sysmon --watch --output log.json  # Watch mode, write JSON each refresh
```

### Metrics to Display

- **CPU**: Overall usage percentage (0-100%). Show all cores if you feel ambitious.
- **Memory**: Used and total in MB (or GB if large enough). Show as percentage.
- **Disk**: For each mounted filesystem, show total, used, and a visual bar.

### Dependencies (add these to Cargo.toml)

```toml
[dependencies]
sysinfo = "0.31"
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
chrono = "0.4"
```

- **sysinfo**: Cross-platform system information (CPU, memory, disks, processes). Works on Linux, macOS, Windows.
- **clap**: Argument parsing. The "derive" feature lets you define your CLI with a struct.
- **serde** + **serde_json**: Serialization. The "derive" feature auto-generates JSON serialization for your structs.
- **chrono**: Timestamp formatting for the snapshot header and JSON output.

> **Version note:** The `sysinfo` API changed significantly between major versions. Use `0.31` — the method names and struct layout in `0.30` differ from what's shown in examples below.

---

## Engineering Approach: Data-Driven Design

Before you open your editor, define the data model. This is not optional — it is the first engineering task.

The question to answer: what does a `SystemSnapshot` look like? Not in code — in words. What information does it hold? What are the types?

Here is the data model for this project, described in prose. Before you write any Rust, you should be able to draw this yourself:

**CpuMetrics** holds everything about the CPU in a single moment: usage as a percentage (floating point, 0.0 to 100.0), core count (whole number), and current frequency in MHz (whole number). Usage is `f64` because it comes from the OS as a float and you display it with a decimal. Core count is `u32` because negative cores don't exist. Frequency is `u64` because it's always non-negative and can be large.

**MemoryMetrics** holds RAM and swap state: total memory in MB, used memory in MB, available memory in MB, and swap used in MB. Everything in MB — not bytes (too large), not GB (not precise enough for small systems). All `u64` because memory sizes are always non-negative.

**DiskMetrics** holds the state of one filesystem: the mount path as a `String`, total space in GB, used space in GB, and usage as a whole-number percentage. One `DiskMetrics` per mounted filesystem.

**SystemSnapshot** is the container for one complete reading: a timestamp as a `String` (formatted before storage), one `CpuMetrics`, one `MemoryMetrics`, and a `Vec<DiskMetrics>` (one entry per disk).

Once this model is defined, the functions become obvious:
- Collecting CPU data → fill a `CpuMetrics` struct
- Displaying disk usage → iterate over `Vec<DiskMetrics>`
- Writing JSON → serialize `SystemSnapshot`
- Watching over time → collect a new `SystemSnapshot` every N seconds

If you find yourself struggling to write a function, go back and check the data model. Hard functions are often a sign that the data is modeled wrong.

---

## What You Bring From Section 1

Section 2 is not starting over. Every concept from Section 1 reappears here, applied in a new context.

**Error handling with `Result` and `?`** — In Section 1, you used it for network connections. In Section 2, you'll use it when reading sysinfo data (which can fail if a sensor is unavailable) and when writing to files for `--output`. The pattern is identical: return `Result`, propagate with `?`, handle at the call site.

**Structs and impl blocks** — In Section 1, you may have defined a simple struct to hold scan results. In Section 2, you define a complete data model with multiple related structs (`CpuMetrics`, `MemoryMetrics`, `DiskMetrics`, `SystemSnapshot`) and impl blocks for each. The mechanics are the same; the scale is larger.

**Threads** — Section 1 used `thread::spawn` for parallelism: scan many ports simultaneously. Section 2 uses `thread::sleep` in the `--watch` loop: pause between refreshes. Same module, different purpose. Notice how the concept of "threads" covers both concurrency and simple timing.

**The `?` operator** — You'll use it extensively when writing JSON to files (`serde_json::to_string_pretty(&metrics)?` and `std::fs::write(path, json)?`). If either fails, the error propagates up without a match arm. Same operator, same behavior, new context.

**Command-line argument parsing** — Section 1 used `std::env::args()` directly. Section 2 uses `clap`. Compare the two approaches after you've implemented both. Notice how much boilerplate `clap` eliminates: help text, defaults, type parsing, and validation are all handled automatically. This is a common progression in Rust projects — start with the standard library, add a crate when the standard library becomes limiting.

---

## How This Project Works in Rust

Understanding why Rust's features matter for this specific project, not just in general.

**The sysinfo crate gives you OS data in its type system; your job is to translate it into yours.** `sysinfo` exposes a `System` object with methods like `.global_cpu_info().cpu_usage()` (returns `f32`) and `.total_memory()` (returns `u64` bytes). Your `CpuMetrics` and `MemoryMetrics` structs use different types and units. The work of creating a `SystemSnapshot` is mostly translation: convert `f32` to `f64`, divide bytes by `1024 * 1024` to get MB, format a timestamp. This pattern — take external data, convert it to your types — appears everywhere in real Rust programs.

**The `Display` trait is how you make `println!("{}", snapshot)` produce a formatted table.** Without it, `println!("{}", snapshot)` won't compile. You must implement `fmt::Display` for `SystemSnapshot`. The compiler enforces this. Once you've implemented it, the display logic is encapsulated in one place — not scattered across `main.rs`. This is the trait system doing what it's supposed to do: defining contracts that the type system enforces.

**The `Collector` trait you define is your first real taste of abstraction via traits.** You can write a function `fn collect_all(collectors: &[Box<dyn Collector>])` that works on CPU, memory, and disk collectors without knowing which one is which. If you add a network collector later, you don't change that function — you add a new type that implements the trait. This is the open/closed principle in Rust form: open for extension, closed for modification.

**`HashMap` for disk data gives O(1) lookup by mount point; `Vec` would require a linear scan.** If you use `Vec<DiskMetrics>` and want to look up the disk at `/home`, you have to iterate through all disks until you find it. With `HashMap<String, DiskMetrics>`, you call `.get("/home")` and get the answer immediately regardless of how many disks exist. For display (which iterates all disks anyway), Vec and HashMap perform the same. For any lookup-by-key operation, HashMap wins clearly.

---

## What the Output Should Look Like

### Default mode (one snapshot):
```
System Monitor — 2024-01-15 14:23:01
══════════════════════════════════════

CPU Usage
  Core 0:   23.4%  [████████░░░░░░░░░░░░░░░░]
  Core 1:    8.1%  [███░░░░░░░░░░░░░░░░░░░░░]

Memory
  Used:   4.2 GB / 16.0 GB  (26%)

Disks
  /          180.3 GB / 500.0 GB  (36%)
  /boot/efi    0.2 GB /   0.5 GB  (40%)
```

### --watch mode:
Screen clears every N seconds and the above updates in place.

### --output file.json:
```json
{
  "timestamp": "2024-01-15T14:23:01Z",
  "cpu_usage_percent": [23.4, 8.1],
  "memory_used_bytes": 4509715456,
  "memory_total_bytes": 17179869184
}
```

---

## Milestones

Work through these in order. Each milestone builds on the last. Don't skip ahead.

### Milestone 1 — Print Raw Stats to Console

Get something printing. Don't worry about structs, traits, or modules yet. Just get the numbers out.

```rust
use sysinfo::System;

fn main() {
    let mut system = System::new_all();
    system.refresh_all();

    let cpu = system.global_cpu_info().cpu_usage();
    let mem_used = system.used_memory() / 1024 / 1024;
    let mem_total = system.total_memory() / 1024 / 1024;

    println!("CPU: {:.1}%", cpu);
    println!("Memory: {} / {} MB", mem_used, mem_total);
}
```

Verify it runs and shows sensible numbers. If it panics or shows zeros, check the `sysinfo` docs — you might need to call a specific `refresh_*` method before reading certain values.

**Stuck? sysinfo API shape (v0.31):**

```rust
use sysinfo::System;

let mut sys = System::new_all();
sys.refresh_all();

// CPU: usage per core
for cpu in sys.cpus() {
    println!("CPU: {:.1}%", cpu.cpu_usage());
}

// Memory: values are in bytes
let total = sys.total_memory();
let used = sys.used_memory();
println!("Memory: {}/{} bytes", used, total);
```

`sys.refresh_all()` is the call that actually polls the OS. Call it once before reading any values. If you call it again, the values update. If you're seeing zero CPU usage, try calling `refresh_all()` twice with a brief `thread::sleep` between them — CPU usage is calculated as a delta between two samples.

### Milestone 2 — Define a Metrics Struct

Create the `Metrics` struct and `DiskInfo` struct. Populate them from sysinfo. Print with `{:#?}` (pretty Debug):

```rust
#[derive(Debug)]
struct Metrics { /* ... */ }

#[derive(Debug)]
struct DiskInfo { /* ... */ }

fn collect_metrics(system: &System) -> Metrics {
    // Fill and return the struct
}

fn main() {
    let mut system = System::new_all();
    system.refresh_all();
    let m = collect_metrics(&system);
    println!("{:#?}", m);
}
```

At this point you should see all the data you care about, just not formatted nicely.

**Stuck? Disk API shape (v0.31):**

```rust
use sysinfo::Disks;

let disks = Disks::new_with_refreshed_list();
for disk in &disks {
    println!(
        "{}: {}/{}",
        disk.mount_point().display(),
        disk.available_space(),
        disk.total_space()
    );
}
```

Note: `available_space()` is free space, not used space. Used space = total - available.

### Milestone 3 — Implement Display for Metrics

Implement `std::fmt::Display` so you can `println!("{}", metrics)` and get the human-readable format with the visual bars. Read **Doc 05 — Traits** before this milestone: it shows exactly how `impl fmt::Display for YourStruct` works, including the `write!(f, ...)` syntax.

For the ASCII bar: you need a function that takes a percentage (0–100) and a width, and returns a `String` with `filled` `#` characters and `empty` `.` characters inside brackets. The math is: filled = `(percent * width) / 100`, empty = `width - filled`. The `String::repeat()` method will be useful. Figure out the implementation — it's a handful of lines and the logic is straightforward from the description.

Then use it inside your `fmt` method for each disk entry.

Expected output when working:
```
CPU: 42.7%  |  Memory: 6144 / 16384 MB
  /       [################..........] 62%
  /home   [#########################.] 95%
```

### Milestone 4 — Split Into Modules

Move the collection logic into separate files:
- `cpu.rs` — reads CPU usage
- `memory.rs` — reads memory usage
- `disk.rs` — reads disk usage

Define the `Collector` trait. Have each module's struct implement it.

In `main.rs`:

```rust
mod cpu;
mod memory;
mod disk;

let collectors: Vec<Box<dyn Collector>> = vec![
    Box::new(cpu::CpuCollector),
    Box::new(memory::MemoryCollector::new(&system)),
];
```

Milestone complete when `cargo build` succeeds with the split structure.

### Milestone 5 — Add CLI Argument Parsing with clap

Create a `struct Args` with clap's derive macro:

```rust
use clap::Parser;

#[derive(Parser, Debug)]
#[command(version, about = "CLI system resource monitor")]
struct Args {
    #[arg(long, help = "Continuously monitor until Ctrl+C")]
    watch: bool,

    #[arg(long, default_value = "2", help = "Refresh interval in seconds")]
    interval: u64,

    #[arg(long, help = "Write JSON output to this file")]
    output: Option<String>,
}

fn main() {
    let args = Args::parse();
    println!("{:?}", args);
}
```

Run `cargo run -- --help` and verify the help text looks right. Run `cargo run -- --watch --interval 5` and verify the args are parsed correctly.

### Milestone 6 — Add Watch Mode

When `--watch` is set, the program should loop: refresh metrics, clear the terminal, print, sleep. When not set, collect once and exit. Use `thread::sleep(Duration::from_secs(args.interval))` for the delay.

The ANSI escape to clear the terminal is `\x1B[2J\x1B[1;1H` (clear screen + cursor to top-left). Print this before each refresh print — not after — so the screen isn't blank during the sleep.

Ctrl+C kills the process cleanly without any extra code. That's the expected behavior.

**Stuck? Key shapes:**

```rust
use std::{thread, time::Duration};

// Clear terminal — call this before printing each refresh
fn clear_terminal() {
    print!("\x1B[2J\x1B[1;1H");
    // flush stdout so the clear takes effect immediately
    use std::io::Write;
    std::io::stdout().flush().ok();
}

// The structure of the main branch:
if args.watch {
    loop {
        // refresh → clear → print → sleep
    }
} else {
    // single snapshot
}
```

Figure out the loop body yourself — you have all the pieces: `system.refresh_all()`, `collect_metrics()`, `clear_terminal()`, `println!()`, `thread::sleep()`.

### Milestone 7 — Add JSON Output

Add `#[derive(Serialize)]` to your structs and implement JSON output:

```rust
use serde::Serialize;

#[derive(Debug, Serialize)]
pub struct Metrics {
    pub cpu_percent: f64,
    pub memory_used_mb: u64,
    // ...
}
```

Then write to file when `--output` is provided. Your `run()` function returns `Result`, so use `?` to propagate errors — never `.expect()` in production code paths:

```rust
if let Some(path) = &args.output {
    let json = serde_json::to_string_pretty(&metrics)?;
    std::fs::write(path, json)?;
}
```

If serialization fails or the file can't be created, `?` propagates the error to the caller, which prints it and exits with code 1. That's the correct behavior — not a panic.

Verify with `cargo run -- --output metrics.json && cat metrics.json`.

**Stuck? serde_json shape:**

Your structs need to derive `Serialize`:
```rust
use serde::Serialize;

#[derive(Serialize)]
pub struct SystemSnapshot {
    pub timestamp: String,
    pub cpu_usage_percent: Vec<f32>,
    pub memory_used_bytes: u64,
    pub memory_total_bytes: u64,
}
```

Writing to a file:
```rust
use std::fs::File;
use std::io::BufWriter;

let file = File::create(&output_path)?;
let writer = BufWriter::new(file);
serde_json::to_writer_pretty(writer, &snapshot)?;
```

If you already have `#[derive(Debug)]` on a struct, just add `Serialize` alongside it: `#[derive(Debug, Serialize)]`. Each field's type also needs to implement `Serialize` — all the primitive types (`String`, `u64`, `f32`, `Vec<T>`) already do.

---

## Hints (Not Solutions)

These are nudges, not code. Try to figure it out first.

**sysinfo API:**
- `System::new_all()` creates a system object. Call `system.refresh_all()` before reading any values — otherwise you get zeros.
- `system.global_cpu_info().cpu_usage()` returns a `f32`. You'll need `as f64`.
- `system.total_memory()` and `system.used_memory()` return bytes (u64).
- `system.disks()` returns an iterator over `Disk` objects. Each `Disk` has `.mount_point()`, `.total_space()`, `.available_space()`.
- For CPU usage to be non-zero, you may need to call `refresh_cpu()` twice with a small sleep between them, or call `refresh_all()` — check the sysinfo docs for your version.

**For the Collector trait:**
- Think about what every collector has in common. They all take a snapshot and return a number. What's the minimum interface for that?
- Note that `collect()` returning `f64` works well for percentages. For memory in MB, you might want a different method or return type. Design your trait to fit the project.

**For Display impl:**
- `write!(f, "...")` works like `print!`. Use `\n` for newlines inside the formatter.
- `writeln!(f, "...")` adds a newline automatically.
- You can call other formatting functions from inside `fmt()`.

**For watch mode:**
- The ANSI escape sequence `\x1B[2J\x1B[1;1H` clears the screen and moves the cursor to position (1,1). It works in most terminals on Linux/macOS. On Windows, it may require enabling ANSI support.
- `thread::sleep(Duration::from_secs(n))` blocks the current thread for n seconds.

**For disk data:**
- A `HashMap<String, DiskInfo>` works well. The key is the mount point as a String.
- `entry().or_insert()` is useful if you need to update disk stats over multiple refreshes.

**For JSON:**
- `serde_json::to_string_pretty(&metrics)` returns a `Result<String, serde_json::Error>`.
- To serialize a `HashMap`, the key type must implement `Serialize`. `String` does.
- `#[serde(rename_all = "snake_case")]` on a struct makes field names in JSON use snake_case.

---

## Stretch Goals

Once all 7 milestones are working, try extending the project:

**Process listing:** Show the top 10 processes by CPU usage. `system.processes()` returns a HashMap of running processes. Each process has `.name()`, `.cpu_usage()`, `.memory()`. Sort them, take the top 10, format them in a table.

**Color output:** Add the `colored` crate. Use red for values above 90%, yellow for above 70%, green for normal. The `colored` crate lets you write `"text".red()` or `format!("{:.1}", pct).yellow()`.

**Network stats:** `system.networks()` gives you per-interface network stats. Show bytes received/sent since the last refresh.

**Historical logging:** In watch mode, append each snapshot as a JSON line to a log file (`--log-file history.jsonl`). Use newline-delimited JSON format — one JSON object per line. A tool like `jq` can then query the history.

**Graceful shutdown:** The `ctrlc` crate lets you register a handler for Ctrl+C. On shutdown, print "Goodbye" or flush a final metrics snapshot before exiting.

---

## Engineering Note

**Iteration order in HashMap:** When you iterate over a `HashMap<String, DiskInfo>`, the order is not guaranteed. If you want disk entries to appear in a consistent order (alphabetical by mount point), collect into a `Vec` and sort before printing:

```rust
let mut disks: Vec<(&String, &DiskInfo)> = self.disks.iter().collect();
disks.sort_by_key(|(mount, _)| mount.as_str());
for (mount, info) in disks {
    // print in order
}
```

**Percentage calculation with integer division:** `6144 / 16384` in integer arithmetic is `0`. Always cast to `f64` before dividing: `(used as f64 / total as f64) * 100.0`.

**sysinfo memory is in bytes:** The raw values from sysinfo are in bytes. Divide by `1024 * 1024` for MB. Divide by `1024 * 1024 * 1024` for GB. Or define constants for these conversions.

**Don't use unwrap() in production paths.** Use `?` in functions that return `Result`, or provide a meaningful default. Reserve `unwrap()` for cases where you can prove the value is always `Some`/`Ok`, or in the early milestones where you're prototyping.
