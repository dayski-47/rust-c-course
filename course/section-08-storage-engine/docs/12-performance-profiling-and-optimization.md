# 05 — Performance Profiling and Optimization

> **Difficulty:** 🟡 think about it  
> **You'll learn:** Why naive timing is broken, statistical benchmarking with Criterion,
> flamegraph-guided hot-spot analysis, the specific bottlenecks storage engines hit,
> and how to systematically close the gap between "runs" and "runs fast."

---

## The Rule Engineers Break Constantly

"Measure before you optimize."

You've heard it before. Experienced engineers break it anyway. They have a hunch
about where the slow part is, skip measuring, optimize the wrong thing, and end up
with code that is more complex and no faster.

For a storage engine, the performance model has specific known bottlenecks. You will
still encounter them in your own measurements. The order matters:

1. Make it correct (Milestone 1–6)
2. Make it measurable (Milestone 11 — Criterion benchmarks)
3. Run a profiler and look at what it says
4. Optimize the thing the profiler says is slow, not the thing you assume is slow

---

## Step 0: Build in Release Mode

Before benchmarking anything, the most important step:

```bash
cargo build --release
cargo run --release
cargo bench  # always uses release
```

Debug builds have bounds checks, no inlining, no LLVM optimizations, no loop
unrolling. The difference between debug and release for a storage engine is routinely
10-50x. If you benchmark debug builds, your numbers are meaningless.

The Criterion benchmark harness always builds in release mode. But your flamegraph
workflow requires you to explicitly build with `--release`.

---

## Criterion: Statistical Benchmarking

Naive timing with `Instant::now()` is broken for three reasons:
- The compiler may optimize the computation away if the result is not used
- A single sample has no statistical significance — thermal throttling, background
  processes, and OS scheduling all add noise
- You cannot detect regressions without a baseline

Criterion solves all three:

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "storage_bench"
harness = false
```

```rust
// benches/storage_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, Throughput, BenchmarkId};
use tempfile::TempDir;
use ironkv::IronKV;

fn bench_writes(c: &mut Criterion) {
    let mut group = c.benchmark_group("writes");

    for count in [1_000u64, 10_000, 100_000] {
        group.throughput(Throughput::Elements(count));
        group.bench_with_input(
            BenchmarkId::new("sequential_write", count),
            &count,
            |b, &count| {
                b.iter_batched(
                    || TempDir::new().unwrap(),
                    |dir| {
                        let db = IronKV::open(dir.path().join("bench.ikv")).unwrap();
                        for i in 0..count {
                            let key = format!("key:{:08}", i);
                            let value = format!("value:{:016}", i);
                            db.set(black_box(key.as_bytes()), black_box(value.as_bytes())).unwrap();
                        }
                        db.flush().unwrap();
                        // Return dir so it lives long enough
                        dir
                    },
                    criterion::BatchSize::SmallInput,
                );
            },
        );
    }
    group.finish();
}

fn bench_reads(c: &mut Criterion) {
    // Pre-populate the database once, then benchmark reads
    let dir = TempDir::new().unwrap();
    let path = dir.path().join("bench.ikv");
    let db = IronKV::open(&path).unwrap();
    let count = 100_000u64;
    for i in 0..count {
        let key = format!("key:{:08}", i);
        let value = vec![42u8; 128];
        db.set(key.as_bytes(), &value).unwrap();
    }
    db.flush().unwrap();

    let mut group = c.benchmark_group("reads");
    group.throughput(Throughput::Elements(1));

    // Random read: worst case for cache performance
    group.bench_function("random_get", |b| {
        let mut rng_counter = 0u64;
        b.iter(|| {
            rng_counter = rng_counter.wrapping_mul(6364136223846793005).wrapping_add(1442695040888963407);
            let idx = rng_counter % count;
            let key = format!("key:{:08}", idx);
            black_box(db.get(black_box(key.as_bytes())).unwrap())
        })
    });

    // Sequential scan
    group.bench_function("full_scan", |b| {
        b.iter(|| {
            let mut n = 0usize;
            for (k, v) in db.scan(b"", b"\xFF").unwrap() {
                black_box((k, v));
                n += 1;
            }
            black_box(n)
        })
    });

    group.finish();
}

criterion_group!(benches, bench_writes, bench_reads);
criterion_main!(benches);
```

Run and look at the numbers:

```bash
cargo bench
# After the run, open target/criterion/report/index.html for HTML charts

# Or just check the terminal output:
# writes/sequential_write/100000
#   time:   [1.2345 s  1.2456 s  1.2578 s]
#   thrpt:  [79.494 Kelem/s  80.283 Kelem/s  81.005 Kelem/s]
```

`black_box()` prevents the compiler from eliminating your benchmark computation.
Always wrap inputs and outputs in `black_box`.

---

## Flamegraphs: Finding the Hot Function

Criterion tells you the total time. Flamegraphs tell you which function is consuming it.

```bash
# Install
cargo install flamegraph

# Build with debug symbols in release mode (add to Cargo.toml temporarily)
# [profile.release]
# debug = true

# Profile 100k writes
cargo flamegraph --root --bin ironkv -- bench --count 100000

# Or profile the bench binary directly:
cargo flamegraph -- --bench storage_bench
```

Reading the flamegraph:
- **Width = time spent in that function** — wider is slower
- **Height = call stack depth** — tall stacks just mean deep call chains, not slow
- **Look for wide bars near the top** — those are leaf functions doing actual work
- If `crc32fast::hash` is 30% wide, your checksum computation is the bottleneck
- If `syscall` or `write` is wide, you are making too many small writes

The flamegraph will tell you something surprising. Act on what it says.

---

## `perf stat`: Hardware Performance Counters

For cache and branch analysis on Linux:

```bash
# Basic stats: cycles, instructions, cache misses, branch mispredictions
perf stat cargo run --release --bin ironkv -- bench --count 100000

# Sample output:
#  Performance counter stats for 'target/release/ironkv bench --count 100000':
#
#    1,234,567,890  cycles
#    2,345,678,901  instructions         # 1.90 insn per cycle
#       12,345,678  cache-misses         # 1.23% of all cache refs
#           45,678  branch-misses        # 0.45% of all branches
#
#     2.456789012 seconds time elapsed
```

Cache miss rate above 5-10% for a hot path is a red flag. For random reads from
a 100k entry database that doesn't fit in L3 cache, you will see high cache miss
rates — that's expected. The question is whether your in-memory LRU cache layer
(Milestone 10) brings the hot-key miss rate down.

---

## Known Storage Engine Bottlenecks

These are the places where storage engines consistently lose time. Knowing them
in advance helps you interpret your flamegraph:

**Small write syscalls.** Writing each record individually (12 header bytes, then
key bytes, then value bytes as three separate `write()` calls) generates 3x the
syscalls of buffered writing. Fix: use `BufWriter<File>` or assemble the full
record into a `Vec<u8>` and write it in one call.

```rust
// SLOW: three syscalls per record
file.write_all(&key_len.to_le_bytes())?;
file.write_all(&val_len.to_le_bytes())?;
file.write_all(key)?;

// FAST: one syscall, one copy
let mut buf = Vec::with_capacity(12 + key.len() + value.len());
buf.extend_from_slice(&key_len.to_le_bytes());
buf.extend_from_slice(&val_len.to_le_bytes());
buf.extend_from_slice(&crc.to_le_bytes());
buf.extend_from_slice(key);
buf.extend_from_slice(value);
file.write_all(&buf)?;
```

**Heap allocation per get().** If every `get()` allocates a `Vec<u8>` for the
returned value, you are allocating and freeing memory on every read. The LRU
cache layer should return `Bytes` (reference-counted slices) so callers that only
need to read the value do not trigger allocation.

**Unnecessary clone.** Rebuilding the index on open requires scanning all records.
If you clone every key into a `Vec<u8>` for the `BTreeMap`, that is N allocations.
These are unavoidable but can be batch-allocated using a pool if they dominate startup.

**Compression on every small value.** zstd compression of a 10-byte value costs
more CPU than the compression saves in I/O. Only compress values above a threshold
(512 bytes is reasonable). The benchmark `bench zstd vs no-zstd` will confirm this.

**Lock contention.** If you wrap the entire engine in `Mutex<IronKV>`, concurrent
readers block each other. Use `RwLock<Index>` to allow concurrent readers with
exclusive write access.

---

## The LRU Cache Layer

The `lru` crate gives you a ready-made LRU cache:

```toml
lru = "0.12"
```

```rust
use lru::LruCache;
use std::num::NonZeroUsize;

struct Engine {
    index: BTreeMap<Vec<u8>, (u64, u32)>,
    data: Mmap,
    cache: LruCache<Vec<u8>, Vec<u8>>,
}

impl Engine {
    fn new(capacity: usize) -> Self {
        Engine {
            index: BTreeMap::new(),
            data: /* ... */ todo!(),
            cache: LruCache::new(NonZeroUsize::new(capacity).unwrap()),
        }
    }

    fn get(&mut self, key: &[u8]) -> Option<Vec<u8>> {
        // Cache hit — O(1), no disk access
        if let Some(value) = self.cache.get(key) {
            return Some(value.clone());
        }

        // Cache miss — look up index, read from mmap
        let (offset, value_len) = *self.index.get(key)?;
        let offset = offset as usize;
        let value = self.data[offset..offset + value_len as usize].to_vec();

        // Populate cache
        self.cache.put(key.to_vec(), value.clone());
        Some(value)
    }
}
```

Measure the cache hit rate. Your benchmark should show:
- Without cache: read latency proportional to file I/O
- With cache for a hot dataset: nearly all reads are O(1) memory accesses

---

## Cargo Profile Settings for Maximum Speed

When benchmarking, ensure your release profile is fully optimized:

```toml
[profile.release]
opt-level = 3          # Full optimization (default, but state it explicitly)
lto = true             # Link-Time Optimization — removes dead code across crate boundaries
codegen-units = 1      # Single codegen unit — slower build, better optimization
strip = false          # Keep symbols for flamegraph (set true for final distribution)
# debug = true         # Uncomment when profiling — re-comment for benchmark numbers
```

LTO (`lto = true`) is particularly valuable for a library that calls into `crc32fast`,
`zstd`, `bytemuck`, and `memmap2`. The compiler can inline across crate boundaries
and eliminate the overhead of those abstractions. Typical improvement: 10-30%.

---

## Hardware CRC32: The Free Win

Modern x86 processors have a hardware `CRC32` instruction (SSE4.2). `crc32fast`
uses it automatically via runtime detection. But you need to tell the compiler
the CPU supports it for compile-time dispatch:

```bash
# Set target CPU to your machine when benchmarking locally
RUSTFLAGS="-C target-cpu=native" cargo bench
```

For distribution, you cannot assume SSE4.2. `crc32fast` handles runtime dispatch
automatically — no action required on your part. But for local benchmark numbers,
`target-cpu=native` removes the dispatch overhead and gives you the true hardware ceiling.

---

## Benchmark Comparison: What to Measure

After Milestone 11, you should have answers to all of these:

| Benchmark | Expected delta | Why it matters |
|-----------|---------------|----------------|
| No-cache reads vs cache reads (hot keys) | 10-100x faster | Cache hit avoids mmap access |
| No-compression vs compression (large values) | 0.5-0.8x write speed, 0.7-0.9x size | CPU vs I/O tradeoff |
| BufWriter vs unbuffered writes | 5-10x faster | Syscall reduction |
| Single write vs batch write (WriteBatch) | 2-5x faster | WAL sync amortization |
| Index rebuild on open: 100k vs 1M keys | Linear | Verify O(n) not O(n log n) |

---

## Common Mistakes

**Benchmarking with `Instant::now()` and a single run.** The numbers will have
20-50% variance due to OS scheduling, thermal throttling, and instruction cache
state. Criterion runs hundreds of iterations and gives you a confidence interval.

**Not using `black_box()` on benchmark inputs and outputs.** The compiler is smart.
If it can prove the result is unused, it eliminates the entire computation. Your
benchmark measures 0 nanoseconds and you trust the number. Always `black_box`.

**Optimizing without running the profiler first.** You see a BTreeMap lookup and
switch to a HashMap. The BTreeMap lookup was 2% of runtime. The thing taking 60%
of the time was a CRC32 computed before checking the cache. Profile first.

**Including database setup in the benchmark measurement.** Opening the database,
creating the tempdir, and writing 100k keys to populate it should happen in the
`iter_batched` setup function, not inside the timed function body. Criterion's
`iter_batched` exists precisely for this.

**Comparing debug and release numbers.** A 10x improvement that is actually just
`--release` vs default is not a real improvement. All comparisons must use the same
build profile.

---

## Throughput Benchmarks and Parameterization

The benchmarks above measure latency (time per operation). For a storage engine you also care about throughput (bytes per second). Criterion supports this:

```rust
use criterion::{Criterion, Throughput, BenchmarkId};

pub fn write_throughput(c: &mut Criterion) {
    let mut group = c.benchmark_group("write_throughput");
    
    // Test different payload sizes
    for size in [64, 256, 1024, 4096, 65536usize] {
        let payload = vec![0u8; size];
        
        // Tell Criterion how many bytes each iteration processes
        group.throughput(Throughput::Bytes(size as u64));
        
        group.bench_with_input(
            BenchmarkId::from_parameter(size),
            &payload,
            |b, payload| {
                b.iter_batched(
                    || open_test_db(),                    // setup: not measured
                    |db| db.put(b"key", payload),        // measured
                    BatchSize::SmallInput,
                )
            }
        );
    }
    group.finish();
}
```

Output shows MB/s at each size, making it easy to see where performance degrades (usually when data exceeds L1/L2 cache).

## Reading Flamegraphs

Flamegraphs show where CPU time is spent. The y-axis is the call stack (bottom = main, top = leaf). The x-axis is time proportion (wider = more time spent there). The color is meaningless.

```bash
# Install
cargo install flamegraph

# Profile (Linux: needs perf, macOS: uses DTrace)
cargo flamegraph --bench storage_bench -- --bench

# Opens flamegraph.svg in browser
```

**How to read it:**
- Look for the widest blocks near the TOP — those are the hot leaf functions
- "Self time" = time in a function's own code (not in functions it calls) — the actual bottleneck
- "Total time" = time in a function including all callees — tells you the call chain cost
- If you see a wide `memcpy` or `memmove` at the top, you're copying too much data
- If you see wide allocator calls (`__rust_alloc`, `jemalloc_alloc`), you're allocating too much

For the storage engine, common findings:
- WAL writes are slow → they're fsync-bound, not CPU-bound (check with `strace`)
- Compaction is slow → usually a memory copy issue (switch to `mmap` reads)
- Index lookup is slow → measure your `BTreeMap` vs `HashMap` choice

## Release Profile: Binary Size and LTO

When you're shipping the storage engine binary (not just benchmarking):

```toml
# Cargo.toml
[profile.release]
opt-level = 3          # Maximum optimization (default for release)
lto = "thin"           # Link-Time Optimization across crate boundaries
codegen-units = 1      # One LLVM codegen unit for maximum optimization
strip = "symbols"      # Remove debug symbols from the binary
panic = "abort"        # Smaller binary: no stack unwinding on panic

# Check binary size:
# cargo build --release && ls -lh target/release/ironkv
# cargo install cargo-bloat && cargo bloat --release
```

`lto = "thin"` is the sweet spot: 5-15% binary size reduction with reasonable compile times. `lto = true` (fat LTO) gives more optimization but takes 3-5× longer to link.

After enabling these, run your Criterion benchmarks again — LTO can change performance significantly (usually better, occasionally worse if it inlines something that was previously cache-friendly).
