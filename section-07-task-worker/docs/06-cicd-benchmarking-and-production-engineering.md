# Doc 06 — CI/CD, Benchmarking, and Production Engineering

Code that works on your machine is a start. Code that is measured, monitored,
automatically verified, and gracefully shut down is production. This document
covers the engineering practices that close that gap for taskforge.

---

## The GitHub Actions Pipeline 🟡

A Rust service with Redis dependencies needs a specific CI setup. Here is
a pipeline that covers linting, testing with a real Redis instance, security
scanning, and building release artifacts.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always

jobs:
  # Stage 1: Fast checks — fail early, fail cheap
  check:
    name: Format + Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      - uses: Swatinem/rust-cache@v2
        with:
          # Only save cache on main — PRs read but don't write
          save-if: ${{ github.ref == 'refs/heads/main' }}
      - name: Check formatting
        run: cargo fmt --all -- --check
      - name: Clippy (deny warnings)
        run: cargo clippy --workspace --all-targets -- -D warnings

  # Stage 2: Tests with real Redis
  test:
    name: Test
    needs: check
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - name: Run unit tests
        run: cargo test --workspace --lib
      - name: Run integration tests
        run: cargo test --workspace --test '*'
        env:
          REDIS_URL: redis://127.0.0.1:6379/1
      - name: Run doc tests
        run: cargo test --workspace --doc

  # Stage 3: Security scan
  audit:
    name: Security Audit
    needs: check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - name: Install cargo-audit
        run: cargo install cargo-audit --locked
      - name: Audit dependencies
        run: cargo audit

  # Stage 4: Release build and Docker push (main branch only)
  release:
    name: Build and Push
    needs: [test, audit]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - name: Build release binary
        run: cargo build --release --workspace
      - name: Build Docker image
        run: docker build -t taskforge:${{ github.sha }} .
      - name: Push to registry
        run: |
          echo "${{ secrets.REGISTRY_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}/taskforge:${{ github.sha }}
```

Key decisions in this pipeline:

**Stage 1 runs first.** Format check and clippy take about 90 seconds. If they
fail, nothing else runs. This saves the 5-10 minutes the test stage would consume.

**Redis runs as a service container.** GitHub Actions service containers start
before the job and are network-accessible. The health check ensures Redis is ready
before the test step starts. Your integration tests read `REDIS_URL` from the
environment.

**Cache is only saved on main.** The `save-if` condition prevents PRs from
writing their own cache entries that pollute the shared cache. PRs read from
the cache written by main, which is usually good enough.

---

## Caching Cargo Correctly 🟢

Cargo's build cache is the single biggest time saver in Rust CI. Without it,
every run compiles all dependencies from scratch — 5-10 minutes for a complex
project. With a warm cache: 30-90 seconds.

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    save-if: ${{ github.ref == 'refs/heads/main' }}
    # Caches ~/.cargo/registry, ~/.cargo/git, and target/ by default
```

What gets cached:
- `~/.cargo/registry`: the crate source downloads (avoid re-downloading)
- `~/.cargo/git`: git-sourced crates
- `target/`: compiled artifacts (the big win)

The cache key is based on `Cargo.lock`. When dependencies change, the cache
is automatically invalidated and rebuilt.

---

## Criterion Benchmarks for Job Throughput 🔴

Benchmarks with `Instant::now()` are not reliable — the compiler can optimize
away computations, CPU frequency scaling affects results, and you have no
variance data. Criterion solves all of this.

```toml
# taskforge-core/Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "job_queue"
harness = false
```

```rust
// taskforge-core/benches/job_queue.rs
use criterion::{
    black_box, criterion_group, criterion_main,
    BenchmarkId, Criterion, Throughput,
};
use taskforge_core::{storage::InMemoryJobRepository, service::JobService};
use std::sync::Arc;

fn setup_service() -> JobService {
    let repo = Arc::new(InMemoryJobRepository::new());
    JobService::new(repo)
}

// Benchmark: how many jobs per second can we push to the in-memory queue?
fn bench_push_throughput(c: &mut Criterion) {
    let rt = tokio::runtime::Runtime::new().unwrap();
    
    let mut group = c.benchmark_group("push_throughput");
    
    for num_jobs in [100, 1_000, 10_000] {
        group.throughput(Throughput::Elements(num_jobs));
        
        group.bench_with_input(
            BenchmarkId::from_parameter(num_jobs),
            &num_jobs,
            |b, &n| {
                b.to_async(&rt).iter(|| async {
                    let service = setup_service();
                    for _ in 0..n {
                        let payload = serde_json::json!({ "to": "user@example.com" });
                        service
                            .submit_job("send_email", black_box(payload), "normal")
                            .await
                            .unwrap();
                    }
                });
            },
        );
    }
    group.finish();
}

// Benchmark: latency for a full push-then-pop roundtrip
fn bench_roundtrip_latency(c: &mut Criterion) {
    let rt = tokio::runtime::Runtime::new().unwrap();
    
    c.bench_function("push_pop_roundtrip", |b| {
        b.to_async(&rt).iter(|| async {
            let service = setup_service();
            let payload = serde_json::json!({ "x": 42 });
            
            let id = service
                .submit_job("test", black_box(payload), "normal")
                .await
                .unwrap();
            
            service.get_status(black_box(id)).await.unwrap()
        });
    });
}

criterion_group!(benches, bench_push_throughput, bench_roundtrip_latency);
criterion_main!(benches);
```

Run with:
```bash
cargo bench
# HTML report at target/criterion/report/index.html

# Compare against a saved baseline
cargo bench -- --save-baseline before_optimization
# ... make changes ...
cargo bench -- --baseline before_optimization
```

Criterion reports P50/P95/P99 latencies and detects regressions against baselines
with statistical confidence. If a PR regresses throughput by more than noise,
Criterion will tell you.

---

## Three Questions Before Optimizing 🟡

Before you touch a line of code to make it faster, answer these:

1. **Have you measured it?** Not guessed — actually measured with a profiler or
   benchmark. Most intuitions about where code is slow are wrong.

2. **Is this actually a bottleneck?** If the slowest part of your system is
   the Redis round-trip (500µs), optimizing the JSON serialization (2µs) saves
   0.4%. Not worth your time.

3. **What does the flamegraph say?** Install `cargo-flamegraph` and generate one:
   ```bash
   cargo install flamegraph
   cargo flamegraph --bin taskforge-worker -- --queue normal
   ```
   Look for the widest stacks at the top of the graph. That is where the time goes.

---

## Structured Logging with tracing 🟡

Never use `println!` for production logging — it has no log level, no timestamps, no context, no way to filter by component, and in async code the output from concurrent tasks interleaves unreadably. `tracing` provides structured, filterable, leveled logging that works correctly with async code.

```rust
use tracing::{info, warn, error, instrument};

#[instrument(skip(repository))]
pub async fn process_job(
    job: Job<Running>,
    repository: &dyn JobRepository,
) -> Result<Job<Completed>, WorkerError> {
    info!(
        job.id = %job.id,
        job.queue = %job.queue,
        "Starting job execution"
    );
    
    // ... execute job ...
    
    info!(
        job.id = %job.id,
        duration_ms = elapsed.as_millis(),
        "Job completed successfully"
    );
    
    Ok(completed_job)
}
```

In `main()`, configure the tracing subscriber:

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, fmt};

// Development: human-readable output
tracing_subscriber::registry()
    .with(fmt::layer())
    .with(tracing_subscriber::EnvFilter::from_default_env())
    .init();

// Production: JSON output for log aggregation systems
tracing_subscriber::registry()
    .with(fmt::layer().json())
    .with(tracing_subscriber::EnvFilter::from_default_env())
    .init();
```

Control verbosity at runtime:
```bash
RUST_LOG=info taskforge-worker
RUST_LOG=taskforge_core=debug,taskforge_worker=info taskforge-worker
```

---

## Metrics with Prometheus 🔴

Prometheus metrics give you visibility into what the system is doing right now.
You need three types of metrics for a job queue:

- **Gauge**: current queue depth (can go up or down)
- **Counter**: total jobs processed (only goes up)
- **Histogram**: job processing time (distribution of values)

```toml
[dependencies]
prometheus = "0.13"
axum-prometheus = "0.6"  # Integrates Prometheus with Axum
```

```rust
use prometheus::{Counter, Gauge, Histogram, HistogramOpts, Opts, Registry};
use std::sync::Arc;

pub struct Metrics {
    pub queue_depth: Gauge,
    pub jobs_processed: Counter,
    pub job_duration_seconds: Histogram,
    pub jobs_failed: Counter,
}

impl Metrics {
    pub fn new(registry: &Registry) -> Self {
        let queue_depth = Gauge::with_opts(
            Opts::new("taskforge_queue_depth", "Current number of pending jobs")
        ).unwrap();
        
        let jobs_processed = Counter::with_opts(
            Opts::new("taskforge_jobs_processed_total", "Total jobs processed")
        ).unwrap();
        
        let job_duration_seconds = Histogram::with_opts(
            HistogramOpts::new(
                "taskforge_job_duration_seconds",
                "Job processing duration in seconds"
            )
            .buckets(vec![0.001, 0.01, 0.1, 0.5, 1.0, 5.0, 30.0])
        ).unwrap();
        
        let jobs_failed = Counter::with_opts(
            Opts::new("taskforge_jobs_failed_total", "Total jobs that failed")
        ).unwrap();
        
        registry.register(Box::new(queue_depth.clone())).unwrap();
        registry.register(Box::new(jobs_processed.clone())).unwrap();
        registry.register(Box::new(job_duration_seconds.clone())).unwrap();
        registry.register(Box::new(jobs_failed.clone())).unwrap();
        
        Metrics { queue_depth, jobs_processed, job_duration_seconds, jobs_failed }
    }
}
```

Expose at `/metrics`:

```rust
pub async fn metrics_handler() -> String {
    use prometheus::Encoder;
    let encoder = prometheus::TextEncoder::new();
    let mut buffer = Vec::new();
    encoder.encode(&prometheus::gather(), &mut buffer).unwrap();
    String::from_utf8(buffer).unwrap()
}
```

---

## Health and Readiness Endpoints 🟢

Kubernetes (and load balancers) use two endpoints:

- `/health`: is the binary alive? (always 200 OK unless process is broken)
- `/ready`: is the service ready to accept traffic? (checks Redis connection, etc.)

```rust
pub async fn health() -> impl IntoResponse {
    StatusCode::OK
}

pub async fn ready(
    State(state): State<AppState>,
) -> impl IntoResponse {
    // Check Redis
    match state.repository.ping().await {
        Ok(_) => StatusCode::OK,
        Err(_) => StatusCode::SERVICE_UNAVAILABLE,
    }
}
```

The distinction matters: during startup, the process is alive (`/health` returns 200)
but Redis might not be connected yet (`/ready` returns 503). A load balancer should
only route traffic once `/ready` returns 200.

---

## Graceful Shutdown 🔴

Never SIGKILL a worker that is processing a job — the job is in `Running` state in Redis, but no worker is alive to update it to `Completed` or `Failed`. It stays stuck in `Running` forever, orphaned. You need graceful shutdown: catch the signal, finish the current job, then exit cleanly so the status is updated correctly.

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    // Set up shutdown channel
    let (shutdown_tx, mut shutdown_rx) = tokio::sync::oneshot::channel::<()>();
    
    // Handle SIGTERM and SIGINT
    tokio::spawn(async move {
        let mut sigterm = signal::unix::signal(signal::unix::SignalKind::terminate())
            .unwrap();
        let mut sigint = signal::unix::signal(signal::unix::SignalKind::interrupt())
            .unwrap();
        
        tokio::select! {
            _ = sigterm.recv() => info!("Received SIGTERM"),
            _ = sigint.recv() => info!("Received SIGINT"),
        }
        
        let _ = shutdown_tx.send(());
    });
    
    // Worker loop
    loop {
        tokio::select! {
            _ = &mut shutdown_rx => {
                info!("Shutdown signal received, finishing current job...");
                break;
            }
            result = process_next_job(&service, &repository) => {
                if let Err(e) = result {
                    error!("Job processing error: {}", e);
                }
            }
        }
    }
    
    info!("Worker shut down cleanly");
}
```

The `tokio::select!` races the shutdown signal against the job processing.
When shutdown arrives, the current `process_next_job` completes (or the worker
waits for it), and then the loop breaks. The job is not dropped mid-execution.

In Kubernetes, configure a `terminationGracePeriodSeconds` longer than your
longest expected job execution time. The default is 30 seconds — probably not
enough for a report generation job.

---

## Common Mistakes

**1. Treating `cargo bench` output as absolute numbers.** Numbers vary between
machines, CPU frequency states, and load. Benchmark relative changes (is this
PR faster or slower than the baseline?) not absolute throughput numbers.

**2. Not running `cargo clippy` in CI.** Clippy catches real bugs, not just style.
`clippy::unwrap_used`, `clippy::expect_used`, and others flag dangerous patterns.
Add `-- -D warnings` to make warnings fail the build.

**3. Using `println!` in production code.** It is not structured, not filterable,
and floods logs when the rate is high. Replace with `tracing::info!` from day one.

**4. Not measuring before optimizing.** Job queue code is I/O-bound. Optimizing
the serialization saves microseconds when Redis costs milliseconds. Measure first.

**5. Ignoring graceful shutdown.** If your CI/CD pipeline kills workers with SIGKILL,
you will have jobs stuck in `Running` state with no worker to complete them.
Implement SIGTERM handling before shipping.

**6. Caching `target/` without `save-if` conditions.** Without `save-if`, every
PR writes its own cache. On a busy repo with many concurrent PRs, the cache
fills up and GitHub evicts old entries. The main branch cache — the one every PR
wants to read — gets evicted. Use `save-if: ${{ github.ref == 'refs/heads/main' }}`.
