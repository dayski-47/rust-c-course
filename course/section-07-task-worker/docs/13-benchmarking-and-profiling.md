# Doc 13 - Benchmarking and Profiling

🟡 A benchmark you can't trust is worse than no benchmark. This doc teaches you to produce numbers that mean something - and to find the code that actually matters to optimize.

Performance work has two phases: *measuring* (figuring out what's slow) and *optimizing* (making it faster). Beginners skip the measuring phase and optimize what they guessed was slow. This is almost always wrong. The only way to know where time goes is to measure it.

The taskforge worker processes thousands of jobs per day. The hot paths are: Redis command execution, job deserialization, and the scheduling loop. This doc builds a benchmark suite for those paths and shows how to profile when the benchmarks reveal a regression.

---

## Why `std::time::Instant` Isn't Enough

The obvious approach to benchmarking:

```rust
let start = std::time::Instant::now();
let result = parse_job_payload(&payload);
let elapsed = start.elapsed();
println!("Took {:?}", elapsed);
```

Problems:

1. **Dead code elimination** - the compiler may prove `result` is unused and skip the computation entirely
2. **Single sample** - one timing tells you nothing about variance or statistical significance
3. **No warm-up** - the first call hits cold CPU caches and branch predictor; not representative
4. **No regression tracking** - you can't compare this run to last week's run

Criterion solves all four.

---

## Criterion.rs: Statistical Benchmarking

Criterion runs your function hundreds to thousands of times, applies statistical analysis, and reports confidence intervals. It also saves historical results and reports regressions.

### Setup

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "job_processing"
harness = false  # Use Criterion's harness
```

### A complete benchmark for taskforge

```rust
// benches/job_processing.rs
use criterion::{black_box, criterion_group, criterion_main, BenchmarkId, Criterion, Throughput};
use taskforge_core::{Job, JobSpec, Priority};

// ── Benchmark 1: Job payload serialization ───────────────────────────────────

fn bench_serialize_job(c: &mut Criterion) {
    let spec = JobSpec {
        job_type: "email_notification".to_string(),
        payload: serde_json::json!({
            "to": "user@example.com",
            "subject": "Your job is done",
            "template": "job_complete",
            "vars": { "job_id": "550e8400", "result": "success" }
        })
        .to_string()
        .into_bytes(),
        priority: Priority::Normal,
        max_retries: 3,
        timeout_secs: 300,
    };

    c.bench_function("serialize_job_spec", |b| {
        b.iter(|| {
            // black_box prevents the compiler from optimizing away the serialization
            serde_json::to_vec(black_box(&spec)).unwrap()
        })
    });
}

// ── Benchmark 2: Job deserialization (the hot path) ──────────────────────────

fn bench_deserialize_job(c: &mut Criterion) {
    let raw = serde_json::json!({
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "job_type": "email_notification",
        "status": "Pending",
        "priority": "Normal",
        "retry_count": 0,
        "created_at": 1700000000_i64
    })
    .to_string();

    c.bench_function("deserialize_job", |b| {
        b.iter(|| {
            serde_json::from_str::<Job>(black_box(&raw)).unwrap()
        })
    });
}

// ── Benchmark 3: Queue routing across priority levels ────────────────────────

fn bench_queue_routing(c: &mut Criterion) {
    let priorities = vec![
        (Priority::Low, "low"),
        (Priority::Normal, "normal"),
        (Priority::High, "high"),
    ];

    let mut group = c.benchmark_group("queue_routing");

    for (priority, label) in &priorities {
        group.bench_with_input(
            BenchmarkId::new("route", label),
            priority,
            |b, priority| {
                b.iter(|| taskforge_core::queue_for_priority(black_box(*priority)))
            },
        );
    }

    group.finish();
}

// ── Benchmark 4: Throughput - how many jobs can we route per second? ─────────

fn bench_routing_throughput(c: &mut Criterion) {
    let jobs: Vec<Job> = (0..1000)
        .map(|i| Job {
            id: uuid::Uuid::new_v4().into(),
            job_type: "resize_image".to_string(),
            status: taskforge_core::JobStatus::Pending,
            priority: if i % 3 == 0 { Priority::High }
                      else if i % 3 == 1 { Priority::Normal }
                      else { Priority::Low },
            retry_count: 0,
            created_at: 1_700_000_000,
        })
        .collect();

    let mut group = c.benchmark_group("routing_throughput");
    group.throughput(Throughput::Elements(jobs.len() as u64));

    group.bench_function("route_1000_jobs", |b| {
        b.iter(|| {
            jobs.iter()
                .map(|job| taskforge_core::queue_for_priority(black_box(job.priority)))
                .collect::<Vec<_>>()
        })
    });

    group.finish();
}

criterion_group!(
    benches,
    bench_serialize_job,
    bench_deserialize_job,
    bench_queue_routing,
    bench_routing_throughput,
);
criterion_main!(benches);
```

### Running and reading results

```bash
# Run all benchmarks
cargo bench

# Run a specific benchmark
cargo bench -- deserialize_job

# Output:
# deserialize_job    time:   [1.2345 µs 1.2456 µs 1.2578 µs]
#                             ▲           ▲           ▲
#                          lower 95%   median    upper 95%
#
# routing_throughput/route_1000_jobs
#                    time:   [38.123 µs 38.456 µs 38.812 µs]
#                    thrpt:  [25.740 Melem/s 26.012 Melem/s 26.229 Melem/s]
#                    change: [-1.23% -0.56% +0.12%] (p = 0.09 > 0.05)
#                    No change in performance detected.
```

**Reading the output:**
- Three numbers = 95% confidence interval: (lower, median, upper)
- A 10%+ change with `p < 0.05` is a statistically significant regression or improvement
- The `change` line compares against the previous run's saved baseline

### `black_box()`: why it matters

```rust
// Without black_box - compiler may skip the computation
b.iter(|| serde_json::to_vec(&spec))

// With black_box - compiler must treat spec as "could have changed" and recompute
b.iter(|| serde_json::to_vec(black_box(&spec)))
```

`black_box` is an identity function that the compiler cannot see through. It prevents dead code elimination (the compiler can't prove the result is unused) and prevents constant folding (the compiler can't evaluate `black_box(x)` at compile time). Every benchmark should `black_box` its inputs.

---

## Continuous Benchmarking in CI

Add a benchmark job that runs on pull requests and posts a comment with the comparison:

```yaml
# .github/workflows/bench.yml
name: Benchmarks

on:
  pull_request:
    branches: [main]

jobs:
  bench:
    name: Benchmark comparison
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history for baseline comparison

      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2

      - name: Install cargo-criterion
        run: cargo install cargo-criterion

      - name: Run benchmarks and save results
        run: cargo criterion --message-format json > bench_results.json

      - name: Comment PR with benchmark results
        uses: benchmark-action/github-action-benchmark@v1
        with:
          tool: criterion
          output-file-path: bench_results.json
          github-token: ${{ secrets.GITHUB_TOKEN }}
          comment-on-alert: true
          alert-threshold: '110%'  # Alert if 10% slower than baseline
          fail-on-alert: false     # Don't fail CI - performance is informational
```

The benchmark CI job is informational, not a hard gate. A 10% slowdown on a CPU that varies ±5% run-to-run is noise, not a regression. Use the data to catch *large* regressions (>15–20%) and to track trends over time.

---

## Profiling: Finding the Hot Spot

Benchmarks tell you *that* something is slow. Profiling tells you *why*.

### Flamegraphs with `cargo-flamegraph`

```bash
cargo install flamegraph

# Profile the benchmark binary
cargo flamegraph --bench job_processing -- --bench deserialize_job
```

A flamegraph shows the call stack at every sample point. Wide boxes at the bottom are functions that consumed more CPU time. If `serde_json::from_str` takes 80% of the deserialize_job benchmark time, that's your target.

On Linux, flamegraph uses `perf`. On macOS, it uses DTrace. On Windows, use `cargo-instruments` with Xcode Instruments.

### `perf` for Linux systems

```bash
# Build with debug symbols (needed for perf to resolve function names)
RUSTFLAGS="-C debuginfo=2" cargo build --release --bench job_processing

# Profile the benchmark
perf record --call-graph dwarf ./target/release/deps/job_processing-*
perf report
```

### `cargo-show-asm`: see what the compiler generated

```bash
cargo install cargo-show-asm

# Show assembly for a specific function
cargo asm --release taskforge_core::queue_for_priority
```

This is most useful for verifying zero-cost abstractions. If `queue_for_priority` should be a single array lookup but the assembly shows a function call chain, something went wrong with inlining.

---

## What to Benchmark in taskforge

The jobs worth benchmarking are the hot paths: code that runs once per job processed.

| Benchmark | Why |
|-----------|-----|
| Job payload deserialization | Runs once per job; JSON parsing can be slow at high throughput |
| Redis command round-trip | Network I/O dominates, but parsing response matters too |
| Priority queue routing | Runs on every job enqueue; should be nearly zero-cost |
| Job status serialization to Redis hash | Runs on every status transition |
| Worker scheduling loop overhead | Overhead per iteration should be < 1µs |

Start with the highest-throughput operations. At 1000 jobs/second, a 1µs overhead per job = 1ms/second overhead, which is fine. At 100,000 jobs/second, the same overhead consumes 10% of capacity.

---

## Performance Anti-Patterns to Catch with Benchmarks

### Unnecessary clones in hot paths

```rust
// ❌ Clones the entire job payload on every dequeue check
fn should_process(job: &Job) -> bool {
    let payload = job.payload.clone();  // unnecessary
    payload.len() < MAX_PAYLOAD_SIZE
}

// ✅ Works with the reference directly
fn should_process(job: &Job) -> bool {
    job.payload.len() < MAX_PAYLOAD_SIZE
}
```

A benchmark catching this: `bench_should_process` showing 2x slower than expected → flamegraph showing `Vec::clone` is the hot function → fix is obvious.

### Re-serializing to compare

```rust
// ❌ Serializes job to JSON just to compare type strings
fn is_email_job(job: &Job) -> bool {
    let json = serde_json::to_string(job).unwrap();  // unnecessary
    json.contains("email_notification")
}

// ✅ Compare the field directly
fn is_email_job(job: &Job) -> bool {
    job.job_type == "email_notification"
}
```

### HashMap allocation in the hot path

```rust
// ❌ Allocates a HashMap on every job to look up one value
fn route_job(job: &Job) -> &'static str {
    let map: HashMap<&str, &str> = [
        ("email", "queue:email"),
        ("image", "queue:image"),
    ].into_iter().collect();  // allocates every call
    map.get(job.job_type.as_str()).copied().unwrap_or("queue:default")
}

// ✅ Use a static lookup or match
fn route_job(job: &Job) -> &'static str {
    match job.job_type.as_str() {
        "email" => "queue:email",
        "image" => "queue:image",
        _ => "queue:default",
    }
}
```

---

## Divan: A Lighter Alternative

`divan` is a newer benchmark framework with a simpler API. Consider it for projects where Criterion's statistical overhead and HTML reports are more than needed:

```toml
[dev-dependencies]
divan = "0.1"

[[bench]]
name = "light_bench"
harness = false
```

```rust
// benches/light_bench.rs
use divan::Bencher;

#[divan::bench]
fn route_job_priority(bencher: Bencher) {
    bencher.bench(|| {
        taskforge_core::queue_for_priority(divan::black_box(Priority::High))
    });
}

fn main() {
    divan::main();
}
```

Divan is faster to compile, simpler to use, and produces clean output. Criterion is better for long-running statistical comparison and HTML reports. Use Criterion for anything you'll track over time; Divan for quick development-time benchmarks.

---

## The Benchmark / Profiling Workflow

When a user reports "taskforge feels slow at scale":

1. **Write a benchmark** that exercises the reported slow path
2. **Run `cargo bench`** to establish baseline numbers
3. **Compare against expectations** - is 38µs for 1000 jobs reasonable? (Yes, ~38ns each)
4. **Run flamegraph** if numbers are worse than expected
5. **Identify the hot function** in the flamegraph
6. **Optimize that function** (not the ones around it)
7. **Re-run the benchmark** to verify improvement
8. **Add the benchmark to CI** so the regression doesn't come back

Step 1 is the hardest. "Feels slow" is not measurable. "Deserialization latency > 10µs at P99" is. The benchmark is how you turn a report into a number, and a number into a verifiable fix.
