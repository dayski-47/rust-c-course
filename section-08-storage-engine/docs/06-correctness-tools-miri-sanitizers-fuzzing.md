# 06 — Correctness Tools: Miri, Sanitizers, and Fuzzing

> **Difficulty:** 🔴 challenging  
> **You'll learn:** Why UB in unsafe code is catastrophic (not just wrong — arbitrary),
> Miri as a Rust-specific UB interpreter, AddressSanitizer and ThreadSanitizer for
> runtime checking, cargo-fuzz for automated crash discovery, and the verification
> stack you run before shipping unsafe code.

---

## Why Correctness Tools Are Non-Negotiable for Unsafe Code

Safe Rust gives you strong guarantees by construction. When you write `unsafe`, you
take those guarantees onto yourself — and the consequences of getting it wrong are
not what you might expect.

In C, a bug causes wrong output or a crash. In Rust with UB, the compiler's optimizer
is allowed to assume UB never happens. That assumption lets it make optimizations
that, in the presence of actual UB, produce code that bears no resemblance to what
you wrote.

Examples of real UB consequences (not hypothetical):
- A null pointer dereference is optimized away because "that can never happen"
  and the code continues executing with a garbage value
- Use-after-free reads return values from unrelated allocations that happen to
  be at the same address
- A data race causes a loop condition to become permanently stale, creating an
  infinite loop
- The compiler reorders writes around a memory fence it does not believe is necessary,
  causing crash-safety violations

The tools in this chapter are not optional extras. They are the safety net that
makes it reasonable to ship unsafe code. Run them. Fix what they find.

---

## Miri: The Rust UB Interpreter

Miri is Rust's mid-level interpreter. Instead of compiling your program to machine
code, Miri interprets the MIR (Mid-level Intermediate Representation) and checks
every operation for undefined behavior at the semantic level.

Install and run:

```bash
rustup +nightly component add miri
cargo +nightly miri test
```

What Miri catches that regular tests miss:

| Bug class | What Miri does |
|-----------|---------------|
| Use-after-free | Tracks every allocation's lifetime; errors on access after drop |
| Invalid pointer arithmetic | Rejects `.add()` that crosses allocation boundaries |
| Unaligned access | Checks alignment at every dereference |
| Stacked Borrows violations | Enforces Rust's aliasing model; catches aliased `&mut` |
| Uninitialized reads | Tracks whether each byte has been written before read |
| Data races | With `cfg(miri)`, finds races between spawned threads |
| Invalid enum values | Catches `transmute::<u8, bool>(2)` at runtime |

A concrete example — this code compiles and usually runs without incident:

```rust
#[test]
fn dangling_pointer_bug() {
    let mut v = vec![1u8, 2, 3];
    let ptr: *const u8 = v.as_ptr();
    v.push(4);  // May reallocate; ptr may now point to freed memory
    // Miri: "pointer to alloc42 was dereferenced after this allocation got freed"
    let _val = unsafe { *ptr };  // UB: ptr may be dangling
}
```

Running this test normally: passes (the allocator probably didn't reuse that memory).  
Running under Miri: immediate, precise error at the exact line.

---

## Running Miri on ironkv

Miri is slow — 10-100x slower than native execution. You do not run it on the
entire test suite. You run it on the tests that exercise `unsafe` code.

```bash
# Run all tests (fine for small test suites)
cargo +nightly miri test

# Run only tests in a specific module
cargo +nightly miri test -- parse_header

# Enable full backtraces for easier debugging
MIRIFLAGS="-Zmiri-backtrace=full" cargo +nightly miri test

# Disable isolation to allow file I/O in tests (needed for ironkv)
MIRIFLAGS="-Zmiri-disable-isolation" cargo +nightly miri test

# Strict provenance checking (stricter than default)
MIRIFLAGS="-Zmiri-strict-provenance" cargo +nightly miri test
```

For ironkv, the unsafe code lives in:
- The mmap access path (bytemuck casts from raw mmap bytes to `&FileHeader`)
- The index rebuild walk (pointer arithmetic through the data section)
- Any direct `ptr.add()` operations in record parsing

Tag slow integration tests to skip under Miri:

```rust
#[test]
#[cfg_attr(miri, ignore)]  // This test is too slow for Miri
fn test_100k_roundtrip() {
    // ... writes 100k keys
}

#[test]
fn test_header_parse() {
    // Fast unit test — runs under Miri
    let header = FileHeader {
        magic: *b"IKVF",
        version: 1,
        ..FileHeader::zeroed()
    };
    let bytes = header.to_bytes();
    let parsed = FileHeader::from_bytes(&bytes).unwrap();
    assert_eq!(parsed.magic, *b"IKVF");
}
```

---

## AddressSanitizer: Runtime Memory Checking

ASan instruments your compiled binary to track every memory allocation and access.
It is 2-5x slower than native (much faster than Miri) and catches a different class
of bugs: out-of-bounds reads/writes and use-after-free in code that crosses FFI.

Miri cannot look inside C library calls. ASan can — it watches the actual machine
instructions.

```bash
# Requires nightly and rust-src component
rustup component add rust-src --toolchain nightly

# Run tests with ASan
RUSTFLAGS="-Zsanitizer=address" \
    cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu

# Run a specific test
RUSTFLAGS="-Zsanitizer=address" \
    cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu -- test_mmap_read
```

A stack-buffer overflow that ASan catches:

```rust
#[test]
fn oob_record_read() {
    let data = vec![0u8; 8];  // Too short for a full record header
    let ptr = data.as_ptr();
    // ASan: "heap-buffer-overflow: READ at offset 12, size 4"
    let crc = unsafe { *(ptr.add(8) as *const u32) };  // reads 4 bytes at offset 8
    // data only has 8 bytes; reading at offset 8 is out of bounds
}
```

Run ASan against your zstd FFI wrappers. If zstd writes past the end of the buffer
you gave it, ASan will catch that.

---

## ThreadSanitizer: Data Race Detection

TSan instruments your binary to detect data races — when two threads access the
same memory concurrently with at least one write and no synchronization.

```bash
RUSTFLAGS="-Zsanitizer=thread" \
    cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu
```

For ironkv's stretch goal of concurrent reads with `RwLock<Index>`, TSan verifies
that your locking is correct. Run it before shipping:

```rust
#[test]
fn concurrent_reads() {
    let db = Arc::new(IronKV::open_in_memory().unwrap());
    db.set(b"key", b"value").unwrap();

    let handles: Vec<_> = (0..8).map(|_| {
        let db = db.clone();
        std::thread::spawn(move || {
            for _ in 0..1000 {
                let _ = db.get(b"key");
            }
        })
    }).collect();

    for h in handles { h.join().unwrap(); }
}
// TSan will catch it if db.get() has an unprotected shared state access
```

---

## Fuzzing with cargo-fuzz

Fuzzing generates random inputs automatically and runs your code against them,
searching for crashes (panics, out-of-bounds accesses, stack overflows). It is
particularly effective for parsers — exactly what your file format parser is.

```bash
# Install
cargo install cargo-fuzz

# Initialize fuzzing in your crate
cargo fuzz init

# Add a fuzz target for the file header parser
cargo fuzz add parse_header
```

This creates `fuzz/fuzz_targets/parse_header.rs`:

```rust
// fuzz/fuzz_targets/parse_header.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
use ironkv::FileHeader;

fuzz_target!(|data: &[u8]| {
    // Give arbitrary bytes to the parser and ensure it never panics.
    // It can return Err, but it must not crash or hit UB.
    let _ = FileHeader::from_bytes(data);
});
```

Add a fuzz target for the full record parser too:

```rust
// fuzz/fuzz_targets/parse_record.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
use ironkv::read_record;

fuzz_target!(|data: &[u8]| {
    // The fuzzer will generate: truncated headers, impossible lengths,
    // wrong CRC values, malformed key lengths. All must be handled gracefully.
    let _ = read_record(data, 0);
});
```

Run the fuzzer:

```bash
# Run until you interrupt it or it finds a crash
cargo +nightly fuzz run parse_header

# Run for 5 minutes then stop
cargo +nightly fuzz run parse_header -- -max_total_time=300

# After a crash, minimize the reproducer
cargo +nightly fuzz tmin parse_header artifacts/parse_header/crash-abc123...
```

The fuzzer runs millions of inputs per second. It will find inputs your unit tests
never considered:
- `value_len` that is larger than the remaining file
- A key_len of zero
- CRC bytes that happen to match garbage data
- Truncated records mid-key
- A single byte (the header magic check)

Fix every crash the fuzzer finds. They represent real corrupted file scenarios.

---

## cargo audit: Known Vulnerabilities

```bash
cargo install cargo-audit
cargo audit
```

This checks your dependencies against the RustSec advisory database for known
security vulnerabilities. Run it before shipping, and set it up in CI on a schedule
(dependencies that were safe today can have advisories published tomorrow).

---

## The Complete Verification Stack

Run these in order, from fastest to most thorough:

```text
Layer           Tool                    When to run
-----           ----                    -----------
1. Unit tests   cargo test              Every commit
2. Miri         cargo +nightly miri test  Every PR (on unsafe modules only)
3. ASan         RUSTFLAGS=-Zsanitizer=address  Every PR (on FFI code)
4. TSan         RUSTFLAGS=-Zsanitizer=thread   Every PR (if concurrent)
5. Fuzzing      cargo fuzz run          Weekly schedule; always on parser changes
6. Audit        cargo audit             Weekly schedule; on dependency changes
```

For ironkv specifically, the minimal CI matrix is:

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo test --all

  miri:
    runs-on: ubuntu-latest
    steps:
      - uses: dtolnay/rust-toolchain@nightly
        with: { components: miri }
      - run: MIRIFLAGS="-Zmiri-disable-isolation" cargo miri test -- parse_header parse_record rebuild_index

  asan:
    runs-on: ubuntu-latest
    steps:
      - uses: dtolnay/rust-toolchain@nightly
      - run: rustup component add rust-src --toolchain nightly
      - run: |
          RUSTFLAGS="-Zsanitizer=address" \
          cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu
```

---

## Interpreting Miri Errors

When Miri finds a problem, it tells you exactly what went wrong:

```text
error: Undefined Behavior: attempting a read access using <untagged> at alloc17,
       but no exposed provenance is established for this pointer
   --> src/parser.rs:42:14
    |
42  |     let val = unsafe { *ptr.add(n) };
    |                        ^^^^^^^^^^^
    | help: this pointer was created here
   --> src/parser.rs:39:19
    |
39  |     let ptr = mmap.as_ptr();
    |
```

The key phrase is "but no exposed provenance is established." This means you created
a pointer from one allocation and tried to use it to access a different allocation (or
past the end of the original). Fix: ensure the pointer and the accessed range are both
within the same allocation.

Another common Miri error:

```text
error: Undefined Behavior: DATA RACE detected between (1) Read on thread `main` and
       (2) Write on thread `thread2` at alloc7+0x0
```

Fix: add a `Mutex` or `RwLock` around the shared state, or use an `Atomic` type.

---

## What Fuzzing Typically Finds in Storage Engines

Based on what fuzzers have found in real storage engines:

**Length field exceeds file size.** `key_len = 0xFFFFFFFF` causes the parser
to try to read 4 GB of data. Fix: bounds-check before allocating or slicing.

**key_len + value_len overflows u32.** Two `u32` fields that are each under the
limit can sum to more than `usize::MAX` on 32-bit platforms. Fix: cast to `u64`
before adding.

**Zero-length keys.** Some code paths assume `key.len() > 0`. A zero-length key
sent through the fuzzer can trigger a panic in code like `key.split_last().unwrap()`.

**CRC matches garbage.** A fuzzer will eventually generate bytes where the CRC32
of the key+value happens to match the stored CRC field, even though the record is
garbage. This is not a bug — it means your CRC check passed — but then subsequent
logic fails because the key or value has garbage content. Handle downstream
validation as a separate layer from checksum verification.

---

## How It Breaks (The Whole Point of This Doc)

### What Each Tool Catches That the Others Miss

**Miri catches:** reading uninitialized memory, dangling pointer dereference, incorrect pointer arithmetic, violation of aliasing rules — all before you run the program on real hardware.

**Miri misses:** races (it's single-threaded by default), performance issues, platform-specific bugs.

**AddressSanitizer catches:** heap buffer overflows, stack buffer overflows, use-after-free — at runtime with real data.

**ASan misses:** logical errors (reading valid but wrong memory), races in carefully lock-protected code.

**ThreadSanitizer catches:** data races — two threads accessing the same memory without synchronization.

**TSan misses:** memory errors that don't involve races.

**Fuzzing catches:** crashes and panics from unexpected inputs — invaluable for parsers.

**Fuzzing misses:** logical errors that don't crash (returning wrong data without panicking).

Run all four on your storage engine. They complement each other.

---

## Common Mistakes

**Only testing the happy path.** Unit tests are written by humans who know the expected
input. The fuzzer doesn't know what "valid" means and finds the edge cases you forgot.

**Ignoring Miri results as "false positives."** Miri has a very low false positive
rate. If Miri reports UB, there is almost certainly real UB. Read the error carefully
before dismissing it.

**Running Miri on the whole test suite.** It is 10-100x slower. Tag slow tests with
`#[cfg_attr(miri, ignore)]` and run Miri only on the tests that exercise `unsafe` code.
Otherwise CI takes forever and the team disables it.

**Not fuzzing after adding new record fields.** Every time the binary format changes,
run the fuzzer again on the parser. A field added to the record header changes the
length expectations and can introduce new truncation bugs.

**Shipping without running any sanitizers.** A common compromise is "we test thoroughly"
with regular unit tests. Unit tests cannot find unaligned accesses, aliasing violations,
or race conditions in code that happens to work correctly in tests but fails under
different timing or allocator behavior.

---

## Safety Invariants to Maintain

- **Miri clean:** Every `unsafe` block in ironkv must pass `cargo +nightly miri test`
  (with isolation disabled for file I/O tests). Before marking any milestone complete,
  run the relevant tests under Miri.

- **Fuzz target coverage:** The file header parser and record parser must have fuzz
  targets. These run in CI weekly. Any crash found must be fixed before the next release.

- **ASan clean on FFI paths:** The zstd FFI wrapper and any other C library calls
  must pass ASan testing. C code cannot be checked by Miri; ASan is the substitute.

- **Panic-free from bad input:** Every parser (`from_bytes`, `read_record`, the WAL
  reader) must return `Result` or `Option` rather than panic on malformed input.
  The fuzzer will verify this. A parser that panics on corrupted file data is a
  denial-of-service vulnerability.
