# Project: ironkv - A Persistent Key-Value Storage Engine

> **Section 8 Capstone Project**  
> This is the final project of the course. You are building a real storage engine
> from scratch - not a toy, not a tutorial exercise. A stripped-down but structurally
> honest version of what RocksDB, LevelDB, and LSDB are built on.

---

## What You're Building

A persistent key-value storage engine called `ironkv`.

- Stores arbitrary key-value pairs: both keys and values are raw byte arrays
- Persists data to disk in a custom binary format you design
- Uses memory-mapped files for fast reads with no syscall overhead in the hot path
- Uses a write-ahead log (WAL) for crash safety
- Has an in-memory LRU cache for hot keys
- Optionally compresses large values using zstd (an FFI exercise)
- Exposes both a Rust library API and a CLI

This is the section that says: you are a systems programmer now.

---

## Architecture

```
+--------------------------------------------------------------+
|                        IronKV Engine                         |
+-------------------------------------+------------------------+
|         In-Memory Layer             |      Disk Layer        |
|                                     |                        |
|  +----------------------------+     |  +------------------+  |
|  |   LRU Cache                |     |  |  Data File       |  |
|  |  (hot keys: Vec<u8>)       | --> |  |  (.ikv, mmap'd)  |  |
|  +----------------------------+     |  +------------------+  |
|          cache miss                 |                        |
|  +----------------------------+     |  +------------------+  |
|  |  Index                     |     |  |  WAL File        |  |
|  |  BTreeMap<Vec<u8>,         | --> |  |  (.wal, append)  |  |
|  |           (offset, len)>   |     |  +------------------+  |
|  +----------------------------+     |                        |
|                                     |                        |
|  +----------------------------+     |                        |
|  |  Write Buffer              |     |                        |
|  |  (pending writes)          | --> |    flush on close /    |
|  +----------------------------+     |    explicit flush()    |
+-------------------------------------+------------------------+

Read path:  LRU cache -> index -> mmap slice of data file
Write path: WAL -> in-memory index + cache update -> periodic flush to data file
```

---

## The Public API

```rust
use ironkv::{IronKV, WriteBatch};

// Open or create a database at the given path
let db = IronKV::open("mydata.ikv")?;

// Write a single key-value pair
db.set(b"user:1:name", b"Alice")?;
db.set(b"user:1:email", b"alice@example.com")?;

// Read a value - returns None if the key does not exist
let name: Option<Vec<u8>> = db.get(b"user:1:name")?;

// Delete a key (writes a tombstone, not a physical delete)
db.delete(b"user:1:name")?;

// Range scan: iterate all keys in [start, end)
for (key, value) in db.scan(b"user:1:", b"user:2:")? {
    println!("{}: {}",
        String::from_utf8_lossy(&key),
        String::from_utf8_lossy(&value));
}

// Batch write: multiple operations atomically
let mut batch = WriteBatch::new();
batch.set(b"a", b"1");
batch.set(b"b", b"2");
batch.delete(b"c");
db.write_batch(batch)?;

// Force all pending writes to disk
db.flush()?;

// Close: flushes pending writes, drops the mmap
db.close()?;
```

---

## The CLI

```bash
# Point lookups
ironkv get mydata.ikv "user:1:name"
ironkv set mydata.ikv "user:1:name" "Alice"
ironkv delete mydata.ikv "user:1:name"

# Iteration
ironkv list mydata.ikv --prefix "user:1:"
ironkv scan mydata.ikv --start "user:1:" --end "user:2:"

# Administration
ironkv stats mydata.ikv          # file size, key count, cache hit rate
ironkv compact mydata.ikv        # rewrite file, removing tombstones and stale versions
ironkv verify mydata.ikv         # check all CRC32 checksums - report corruption
ironkv bench mydata.ikv          # run the built-in benchmark
```

---

## File Format: Exact Binary Layout

### Data File (.ikv)

Every file starts with a 64-byte header:

```
Offset   Size   Type        Field
------   ----   ----        -----
0        4      [u8; 4]     magic = b"IKVF"
4        4      u32 LE      version = 1
8        8      u64 LE      entry_count (live, non-tombstone entries)
16       8      u64 LE      index_offset (byte offset where index section begins)
24       8      u64 LE      created_at (Unix timestamp, seconds)
32       4      u32 LE      checksum (CRC32 of bytes 0..32)
36       28     [u8; 28]    _padding (reserved, must be zero)
```

After the header comes the data section. Records are appended sequentially.
The file is never modified in place - records are only appended:

```
For each record:
  Offset   Size       Field
  ------   ----       -----
  +0       4          key_len: u32 LE
  +4       4          value_len: u32 LE   (u32::MAX = tombstone / deleted)
  +8       4          crc32: u32 LE       (CRC32 of key_bytes ++ value_bytes)
  +12      key_len    key bytes
  +12+k    value_len  value bytes (may be zstd-compressed for large values)
```

After the last record comes the index section (at offset `header.index_offset`):

```
For each key (sorted lexicographically):
  Offset   Size       Field
  ------   ----       -----
  +0       4          key_len: u32 LE
  +4       key_len    key bytes
  +4+k     8          data_offset: u64 LE  (byte offset of value in data section)
  +12+k    4          value_len: u32 LE
```

### WAL File (.wal)

The WAL is append-only. Every write goes here first, before the data file.

```
For each WAL entry:
  Offset   Size       Field
  ------   ----       -----
  +0       1          op: u8   (0x01 = Set, 0x02 = Delete)
  +1       4          key_len: u32 LE
  +5       4          value_len: u32 LE   (0 for Delete ops)
  +9       key_len    key bytes
  +9+k     value_len  value bytes
  end      4          crc32: u32 LE  (CRC32 of op ++ key ++ value)
```

On startup, if the WAL is non-empty, replay it into the in-memory state and data file.
After a successful flush to the data file, truncate the WAL to zero length.

---

## Cargo.toml

```toml
[package]
name = "ironkv"
version = "0.1.0"
edition = "2021"

[dependencies]
bytemuck = { version = "1", features = ["derive"] }
memmap2 = "0.9"
crc32fast = "1"
zstd = "0.13"
thiserror = "1"
clap = { version = "4", features = ["derive"] }
lru = "0.12"
tracing = "0.1"
tracing-subscriber = "0.3"

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
tempfile = "3"

[[bench]]
name = "storage_bench"
harness = false

[[bin]]
name = "ironkv"
path = "src/main.rs"
```

---

## Critical Patterns for This Section

### 1. Binary File Layout Constants

Define your file format as constants before writing any I/O code. Everything else flows from these:

```rust
pub const MAGIC: &[u8; 4] = b"IKVF";
pub const VERSION: u8 = 1;
pub const HEADER_SIZE: usize = 64;

// Record layout: [key_len: u32][value_len: u32][flags: u8][crc32: u32][key][value]
pub const RECORD_HEADER_SIZE: usize = 4 + 4 + 1 + 4;  // 13 bytes

#[repr(C)]
#[derive(Debug, Clone, Copy, bytemuck::Pod, bytemuck::Zeroable)]
pub struct FileHeader {
    pub magic: [u8; 4],     // b"IKVF"
    pub version: u8,
    pub _padding: [u8; 3],
    pub record_count: u64,
    pub created_at: u64,    // unix timestamp
    pub _reserved: [u8; 44],
}
```

`#[repr(C)]` ensures the struct lays out exactly as you specify, with no padding surprises. `bytemuck::Pod` + `bytemuck::Zeroable` let you safely cast bytes to this struct.

### 2. Reading a Struct from Bytes (bytemuck)

```toml
bytemuck = { version = "1", features = ["derive"] }
```

```rust
use bytemuck;

// Read the header from the first 64 bytes of the file:
let mut header_bytes = [0u8; HEADER_SIZE];
file.read_exact(&mut header_bytes)?;

let header: &FileHeader = bytemuck::from_bytes(&header_bytes);

// Verify it's our format:
if &header.magic != b"IKVF" {
    return Err(StorageError::InvalidFormat("bad magic bytes".into()));
}
```

`bytemuck::from_bytes` is safe because `FileHeader` implements `Pod` (Plain Old Data) - no pointers, no padding bytes, no undefined states. The compiler verifies this through the derive.

### 3. Memory-Mapped File with memmap2

```toml
memmap2 = "0.9"
```

```rust
use memmap2::Mmap;
use std::fs::File;

let file = File::open("storage.ikv")?;
let mmap = unsafe { Mmap::map(&file)? };
// mmap is now a &[u8] you can index directly:
let header_bytes = &mmap[0..HEADER_SIZE];
```

Why unsafe? Because the OS can change the file while you have it mapped - if the file is truncated, accessing mapped bytes beyond the new end is undefined behavior. As long as you never truncate a mapped file and only use immutable Mmap (not MmapMut), the safety invariants hold. Document this in a SAFETY comment.

### 4. CRC32 for Record Integrity

```toml
crc32fast = "1"
```

```rust
use crc32fast::Hasher;

pub fn compute_crc32(data: &[u8]) -> u32 {
    let mut hasher = Hasher::new();
    hasher.update(data);
    hasher.finalize()
}

// When writing a record:
let crc = compute_crc32(&key) ^ compute_crc32(&value);

// When reading a record:
let expected_crc = compute_crc32(&key) ^ compute_crc32(&value);
if stored_crc != expected_crc {
    return Err(StorageError::Corruption {
        offset,
        expected: expected_crc,
        found: stored_crc
    });
}
```

### 5. The WAL Pattern (Write-Ahead Log)

Before modifying the data file, write to the WAL. On crash, replay the WAL:

```rust
// Writing a record:
// 1. Write to WAL first (append only):
wal.write_entry(WalEntry::Set { key: key.clone(), value: value.clone() })?;
wal.sync()?;

// 2. Write to data file:
let offset = data_file.append_record(&key, &value)?;

// 3. Update in-memory index:
index.insert(key, (offset, record_len));

// 4. Clear the WAL entry (or truncate WAL):
wal.commit()?;

// On startup, replay any uncommitted WAL entries:
pub fn recover(wal: &mut Wal, data_file: &mut DataFile) -> anyhow::Result<()> {
    for entry in wal.uncommitted_entries()? {
        data_file.append_record(&entry.key, &entry.value)?;
    }
    wal.clear()?;
    Ok(())
}
```

### 6. FFI: Calling zstd from C

```toml
[build-dependencies]
cc = "1"

[dependencies]
libc = "1"
```

`build.rs`:
```rust
fn main() {
    // zstd provides a C API we can call directly
    // On most systems, link to the system zstd:
    println!("cargo:rustc-link-lib=zstd");
}
```

`src/compression.rs`:
```rust
extern "C" {
    fn ZSTD_compress(
        dst: *mut libc::c_void,
        dst_capacity: libc::size_t,
        src: *const libc::c_void,
        src_size: libc::size_t,
        compression_level: libc::c_int,
    ) -> libc::size_t;

    fn ZSTD_isError(code: libc::size_t) -> libc::c_uint;
    fn ZSTD_compressBound(src_size: libc::size_t) -> libc::size_t;
}

pub fn compress(data: &[u8]) -> Result<Vec<u8>, StorageError> {
    unsafe {
        let bound = ZSTD_compressBound(data.len());
        let mut output = vec![0u8; bound];
        let compressed_size = ZSTD_compress(
            output.as_mut_ptr() as *mut libc::c_void,
            output.len(),
            data.as_ptr() as *const libc::c_void,
            data.len(),
            3,  // compression level
        );
        if ZSTD_isError(compressed_size) != 0 {
            return Err(StorageError::CompressionFailed);
        }
        output.truncate(compressed_size);
        Ok(output)
    }
}
```

---

## Milestone Expected Output

**Milestone: basic get/set works:**

```rust
// In a test or main:
let mut store = IronKV::open("test.ikv")?;
store.set(b"hello", b"world")?;
let value = store.get(b"hello")?;
assert_eq!(value, Some(b"world".to_vec()));
println!("OK: get/set works");
```

```
$ cargo test test_basic_get_set
test test_basic_get_set ... ok
```

**Milestone: persistence across restarts:**

```rust
// First run:
{
    let mut store = IronKV::open("test.ikv")?;
    store.set(b"key", b"value")?;
}  // store drops, file closed

// Second run - data should still be there:
{
    let store = IronKV::open("test.ikv")?;
    assert_eq!(store.get(b"key")?, Some(b"value".to_vec()));
    println!("Persistence works!");
}
```

**Milestone: verify catches corruption:**

```bash
$ ironkv verify test.ikv
Verifying test.ikv...
  Records scanned: 1,247
  Corrupted records: 0
  File is clean.

# After manually corrupting a byte with a hex editor:
$ ironkv verify test.ikv
Verifying test.ikv...
  Records scanned: 847
  ERROR: CRC mismatch at offset 0x1A4F2
    Expected: 0x8B3E91AC
    Found:    0x8B3E91AD
  File has corruption. Do not use until repaired.
```

---

## Testing with Miri (Correctness Verification)

Before shipping any unsafe code, run it under Miri:

```bash
# Install Miri:
rustup component add miri

# Run your tests under Miri:
cargo miri test

# Miri will catch:
# - reading uninitialized memory
# - use-after-free
# - invalid pointer arithmetic
# - alignment violations
```

Miri is slow (10-1000x slower than normal tests). Run it only on the test that exercises your unsafe code, not the whole suite:

```bash
cargo miri test test_mmap_read
cargo miri test test_bytemuck_cast
```

If Miri reports an error, it's a real bug. Fix it before proceeding.

---

## The Unsafe Parts

Be explicit about what requires `unsafe` and why.

**Memory-mapping the data file.**

```rust
// SAFETY: We are the sole process reading and writing this file. The file will
// not be truncated while this mapping is live (we hold the exclusive lock).
let mmap = unsafe { Mmap::map(&self.file)? };
```

**Casting mmap bytes to FileHeader via bytemuck.**

```rust
// bytemuck::from_bytes requires:
//   1. The slice length equals size_of::<FileHeader>() (checked by bytemuck)
//   2. The pointer is aligned for FileHeader (guaranteed by mmap page alignment)
//   3. All bit patterns are valid for FileHeader (guaranteed by #[derive(Pod)])
let header: &FileHeader = bytemuck::from_bytes(&mmap[..64]);
```

**Index rebuild: scanning through raw bytes.**  
This is safe Rust (slice indexing), but requires careful bounds checking because
sizes come from file data that could be malformed or corrupt. Always validate
against `mmap.len()` before slicing.

**The zstd FFI wrapper.**  
Calling `ZSTD_compress` and `ZSTD_decompress` via `extern "C"`. The safety
invariant: provide correctly-sized output buffers, check return values for errors
before interpreting the output as valid compressed/decompressed data.

---

## Engineering Approach: Correctness Contract and Safety Invariants

Before writing `unsafe` code, write your safety contract in a text file. ironkv's contract:

**Correctness guarantees (what the API promises):**
1. If `set(key, value)` returns `Ok(())`, then `get(key)` returns `Ok(Some(value))` in the same process.
2. If the process crashes after `set()` returns `Ok(())`, `get(key)` returns `Ok(Some(value))` on the next `open()` (WAL ensures this).
3. If `get(key)` returns `Ok(None)`, no previous successful `set(key, _)` call has been made (or it was deleted).
4. `verify()` detects any corruption (CRC32 mismatch) introduced after the file was written.

**Safety invariants for unsafe code:**
1. A pointer derived from `Mmap::as_ptr()` is valid for reads only, only for the lifetime of the `Mmap`, and only within the mapped region `[0, file_size)`.
2. A struct cast via `bytemuck::from_bytes` is only done after verifying the byte slice has exactly `size_of::<Struct>()` bytes and the alignment is satisfied.
3. The file is only extended (never shrunk) while it is mapped. The mmap is closed before any truncation.

Write these before writing code. When you write `unsafe { }`, add a `// SAFETY:` comment referencing the relevant invariant. This is how production Rust `unsafe` code is written.

---

## What You Bring From Sections 1-7

Section 8 is the synthesis of everything:

- **S1 error handling:** `StorageError` is your error type, same `thiserror` pattern you've used since S3. The `?` operator chains storage errors throughout.
- **S2 structs with impl blocks:** the `IronKV` struct has an `impl` block with all the storage operations. `BTreeMap<Vec<u8>, (u64, u32)>` for the index uses the same `BTreeMap` you learned in S2.
- **S3 Arc<Mutex<T>>:** concurrent access to the index uses `Arc<RwLock<BTreeMap<...>>>` - readers don't block each other, writers block readers. Same principle as S3's `Arc<Mutex<>>`, upgraded to `RwLock` for read-heavy workloads.
- **S4 async:** ironkv is mostly synchronous (I/O at this level is often synchronous for simplicity), but the HTTP API layer (if you add it) is async Axum from S5.
- **S5 AppError pattern:** `StorageError` implements `IntoResponse` if you add the HTTP API.
- **S7 trait patterns:** the storage engine could be behind a trait (`trait Storage { fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>, StorageError>; }`) so it can be swapped with an in-memory implementation for tests. Same repository pattern from S7.
- **S7 newtype pattern:** `Key(Vec<u8>)` and `Value(Vec<u8>)` newtypes prevent mixing them up. `CRC32(u32)` as a newtype means you can't accidentally compare a checksum with a length.

---

## How This Project Works in Rust - The Full Picture

The storage engine is three layers.

**Layer 1 - The file (disk):** your data lives in a binary file. On disk, it's a header followed by records in append order. Deleted records are marked with a tombstone. Updates are new records (the old one becomes stale). The file only grows - records are never modified in place. This is an append-only log, the simplest durable storage design.

**Layer 2 - The index (memory):** when you open the database, you scan the entire data file and build a `BTreeMap` in memory mapping each key to its latest record's file offset and length. This is fast at runtime (key lookup is O(log n)) but takes time on startup for large files. The `BTreeMap` also enables range scans: find all keys between `"user:1:"` and `"user:2:"`.

**Layer 3 - The WAL (write-ahead log):** before writing a record to the data file, you first write it to the WAL. On crash during the data file write, the WAL lets you replay the incomplete write on next startup. After successfully writing to the data file and updating the index, you truncate the WAL. This is the "write-ahead" in WAL: the log is written before the data, always.

The `unsafe` code lives at the boundary between these layers: reading the file header and record headers from the mmap as typed structs (`bytemuck`), and the raw pointer operations in the ring buffer cache.

Everything else is safe Rust: `BTreeMap` for the index, file operations through `std::fs`, CRC32 computation through the `crc32fast` crate (pure safe Rust), zstd compression through the `zstd` crate (wraps the C library safely).

---

## Milestones

Work through these in order. Each milestone is a vertical slice - the database
works end-to-end after each one, just with fewer features.

### Milestone 1: In-Memory Only
Implement `IronKV` backed by a `HashMap<Vec<u8>, Vec<u8>>`. Get the API shape right.
Write unit tests. Get the range scan working with a `BTreeMap` instead.

No files. No unsafe. Just the interface.

### Milestone 2: Write to File, Read from File
Write key-value pairs as records appended to a file. On open, do a full sequential
scan to rebuild the in-memory state. `get()` reads from the in-memory map.

This is O(n) on open but correct. Ship it.

### Milestone 3: In-Memory Index (BTreeMap)
Replace the rebuild-then-query pattern with a persistent `BTreeMap<Vec<u8>, (u64, u32)>`
mapping key to (file_offset, value_len). `get()` looks up the offset and reads from
the file directly. Now `open()` rebuilds the index, but `get()` is O(log n).

### Milestone 4: File Header
Define `FileHeader` with `#[repr(C)]` and `#[derive(Pod, Zeroable)]`. Write it
at the start of the file. Validate magic bytes and version on open.

```rust
// Hint: use bytemuck to write and read the header
let header_bytes = bytemuck::bytes_of(&header);
file.write_all(header_bytes)?;
```

### Milestone 5: CRC32 Checksums
Add CRC32 to every record on write. Verify CRC32 on every record read. Return
`Err(DataCorruption)` when the check fails.

Add the `ironkv verify` CLI command: scan all records, report how many are valid
and how many are corrupt.

### Milestone 6: Write-Ahead Log (WAL)
Every `set()` and `delete()` writes a WAL entry first. `flush()` copies WAL
contents to the data file and truncates the WAL.

On `open()`, replay the WAL if it is non-empty. This handles the case where
the process crashed after writing to the WAL but before flushing to the data file.

Test: write 100 keys, kill the process mid-write (use `std::process::exit(0)`
after some writes), reopen, verify all keys that reached the WAL are present.

### Milestone 7: Memory-Mapped Reads
Switch the read path from `file.read_at()` to `Mmap::map()`. `get()` now reads
from the memory map. The index maps keys to offsets in the mmap.

Add the required `SAFETY:` comments for every `unsafe` block in this milestone.

After adding, re-run tests. Run `cargo +nightly miri test -- parse_header` and
any test that exercises the mmap parsing path.

### Milestone 8: zstd Compression
Values larger than 512 bytes are compressed with zstd before writing and
decompressed on read. Store a compression flag in the record header
(use bit 31 of `value_len`, or add a `flags: u32` field to the record header).

Use the `zstd` crate (which provides safe Rust bindings on top of the C library):

```rust
// Compress
let compressed = zstd::encode_all(value, 3)?;  // level 3 is a good default

// Decompress
let decompressed = zstd::decode_all(&compressed[..])?;
```

Benchmark: compare write throughput and file size with and without compression
for 1 MB values. The `ironkv bench` command should report both configurations.

### Milestone 9: Compaction
`compact()` rewrites the database file, keeping only the latest version of each
key and discarding tombstones.

Algorithm:
1. Open the current data file for reading (mmap)
2. Open a new temp file for writing
3. Iterate the current index (sorted BTreeMap); for each live key, copy its
   record to the temp file
4. Write the new file header and index section
5. Atomically rename the temp file over the original

This reduces file size after many updates/deletes.

### Milestone 10: LRU Cache
Add `lru::LruCache<Vec<u8>, Vec<u8>>` to the engine. `get()` checks the cache
first; on a miss, reads from the mmap and populates the cache. `set()` updates
both the cache and the index.

The `ironkv stats` command reports cache hit rate.

### Milestone 11: Criterion Benchmarks
Create `benches/storage_bench.rs` with benchmarks for:
- Sequential write throughput (keys/sec, MB/sec) for 10k, 100k entries
- Random read latency (P50, P95, P99) for 100k entries
- Range scan throughput (keys/sec) across all entries
- Cache: hit vs miss latency comparison
- Compression: write throughput with and without zstd, file size ratio

```bash
cargo bench
open target/criterion/report/index.html
```

### Milestone 12: Miri Verification
Run the test suite under Miri. Fix any UB it finds.

```bash
MIRIFLAGS="-Zmiri-disable-isolation" cargo +nightly miri test -- parse_header rebuild_index
```

Every `unsafe` block must have a `// SAFETY:` comment before Milestone 12 is
considered complete.

### Milestone 13: Fuzzing the Parser
Add cargo-fuzz targets for:
- `parse_header`: feed arbitrary bytes to `FileHeader::from_bytes`
- `parse_record`: feed arbitrary bytes to `read_record`
- `open_wal`: feed arbitrary bytes to the WAL replay logic

```bash
cargo install cargo-fuzz
cargo fuzz add parse_header
cargo +nightly fuzz run parse_header -- -max_total_time=300
```

Fix every panic the fuzzer finds. The goal is that no input - however malformed -
causes a panic. All bad inputs return `Err(...)`.

---

## Hints (Not Solutions)

**Bytemuck for the file header:**  
Use `#[derive(Pod, Zeroable)]` with `#[repr(C)]` on `FileHeader`. Then
`bytemuck::from_bytes::<FileHeader>(&mmap[0..64])` gives you a typed reference
into the mmap with zero copying. Use `bytemuck::bytes_of(&header)` to write it.

**Index rebuild from the mmap:**  
Keep a `cursor: usize` starting at 64 (after the header). Read key_len, value_len,
crc32 as little-endian u32. If value_len == u32::MAX, it's a tombstone - remove
from BTreeMap. Otherwise, insert (key, (cursor + 12 + key_len as u64, value_len)).
Advance cursor by 12 + key_len + value_len.

**Range scans via BTreeMap:**  
`BTreeMap::range(start_key..end_key)` gives you an iterator over all keys in that
range. This is how `scan()` is implemented - iterate the range in the BTreeMap,
fetch each value from the mmap using the stored offset.

**WAL replay:**  
On `open()`, if the WAL file exists and is non-empty, read it entry by entry.
For each Set entry, call the internal write-to-data-file path. For each Delete
entry, write a tombstone. Then truncate the WAL. This is safe to do even if the
WAL is truncated mid-entry - detect truncated entries by a CRC mismatch and stop.

**File growth invalidates mmap:**  
After flushing (writing new records to the data file), the mmap must be refreshed. Why: `Mmap::map()` snapshots the file's current size into the mapping - bytes written after that point are outside the mapped region. Drop the old `Mmap`, then call `Mmap::map()` again. The new mmap covers the full updated file.

**LRU cache sizing:**  
Express cache size as a number of entries, not bytes, for simplicity. 10,000 entries
is a reasonable default for a development config. The entry limit is set via
`IronKV::open_with_options()`.

---

## Testing Requirements

**Unit tests (in `src/`):**

```rust
#[test]
fn set_and_get_roundtrip() {
    let dir = tempfile::TempDir::new().unwrap();
    let db = IronKV::open(dir.path().join("test.ikv")).unwrap();
    db.set(b"hello", b"world").unwrap();
    assert_eq!(db.get(b"hello").unwrap(), Some(b"world".to_vec()));
}

#[test]
fn delete_removes_key() {
    let dir = tempfile::TempDir::new().unwrap();
    let db = IronKV::open(dir.path().join("test.ikv")).unwrap();
    db.set(b"key", b"val").unwrap();
    db.delete(b"key").unwrap();
    assert_eq!(db.get(b"key").unwrap(), None);
}

#[test]
fn persists_across_reopen() {
    let dir = tempfile::TempDir::new().unwrap();
    let path = dir.path().join("test.ikv");
    {
        let db = IronKV::open(&path).unwrap();
        db.set(b"persistent_key", b"persistent_value").unwrap();
        db.close().unwrap();
    }
    // Reopen and verify
    let db2 = IronKV::open(&path).unwrap();
    assert_eq!(db2.get(b"persistent_key").unwrap(),
               Some(b"persistent_value".to_vec()));
}

#[test]
fn wal_recovery_after_crash() {
    let dir = tempfile::TempDir::new().unwrap();
    let path = dir.path().join("test.ikv");
    {
        let db = IronKV::open(&path).unwrap();
        db.set(b"key_a", b"value_a").unwrap();  // Goes to WAL
        // "Crash" - no flush, no close
        // Drop db without calling close()
    }
    // The WAL should have been written; reopen replays it
    let db2 = IronKV::open(&path).unwrap();
    assert_eq!(db2.get(b"key_a").unwrap(), Some(b"value_a".to_vec()));
}

#[test]
fn crc32_corruption_detected() {
    // Write a valid file, flip a byte in the data section, try to read it
    // Expect: Err(DataCorruption { ... })
}

#[test]
fn range_scan_returns_sorted_results() {
    let dir = tempfile::TempDir::new().unwrap();
    let db = IronKV::open(dir.path().join("test.ikv")).unwrap();
    db.set(b"c", b"3").unwrap();
    db.set(b"a", b"1").unwrap();
    db.set(b"b", b"2").unwrap();

    let results: Vec<_> = db.scan(b"", b"\xFF").unwrap().collect();
    assert_eq!(results[0].0, b"a");
    assert_eq!(results[1].0, b"b");
    assert_eq!(results[2].0, b"c");
}
```

**Property tests (property_test! via quickcheck or proptest):**

```rust
// For every sequence of set/delete/get operations, the final state is consistent.
// Run with proptest: generate random operation sequences, apply to IronKV,
// verify that the result matches a reference HashMap implementation.
```

**Fuzz targets:**

```rust
// fuzz/fuzz_targets/parse_header.rs
fuzz_target!(|data: &[u8]| {
    let _ = ironkv::FileHeader::from_bytes(data);
    // Must not panic. May return None or Err.
});
```

---

## Safety Invariants

Document these in `src/lib.rs` at the top of the crate:

- **Append-only writes.** The data file is never modified in place. Records are
  appended. Old versions of a key become stale after an update and are removed
  during compaction. This makes crash recovery simple: the file is always a valid
  append-only log.

- **WAL before data.** Every write goes to the WAL before the data file. The WAL
  is fsync'd before acknowledging a write. This guarantees that no acknowledged
  write is lost even if the process crashes immediately after.

- **Index under exclusive lock.** The BTreeMap index is modified only while holding
  an exclusive write lock (Mutex or RwLock write guard). Reads hold a shared lock.
  This is the basis for concurrent access safety.

- **Mmap lifetime.** The raw bytes obtained from the mmap are valid only while the
  `Mmap` value is alive. The `Mmap` is kept in the engine struct alongside the
  index. If the mmap is refreshed (after file growth), all references into the old
  mmap are invalidated - Rust's borrow checker enforces this if the mmap is accessed
  via references tied to its lifetime.

- **CRC32 per record.** No record is considered valid unless its CRC32 matches the
  stored checksum. This is checked on every read. Corruption returns `Err`, never
  silently returns garbage data.

---

## Stretch Goals

These are real production features, not busy work. Each is a substantial challenge.

**Transactions with ACID guarantees.**  
Add a `Transaction` type that buffers writes and reads consistently from a snapshot.
Commits are atomic via the WAL.

**Concurrent reads with RwLock.**  
Replace `Mutex<EngineInner>` with `RwLock<EngineInner>`. Multiple threads can read
simultaneously. Only writes take an exclusive lock. Run ThreadSanitizer to verify.

**Bloom filters.**  
Before a `get()` checks the index, check a Bloom filter. If the filter says "definitely
not present," return `None` immediately. This eliminates a BTreeMap lookup for missing
keys - critical for write-heavy workloads that frequently query non-existent keys.

**Custom SIMD CRC32.**  
Use `std::arch::x86_64::_mm_crc32_u64` to compute CRC32 directly with hardware
intrinsics. Requires `unsafe` and the `target_feature(+sse4.2)` annotation.
Benchmark against `crc32fast` (which likely does the same thing via runtime detection).

**`no_std` compatibility for the core engine.**  
Strip the core library of all `std` dependencies, using only `core` and `alloc`.
This enables ironkv to run on embedded targets with an allocator but no OS.
File I/O becomes an injected trait (`trait Storage: Read + Write + Seek`).

**TCP server.**  
Wrap the engine in a TCP server with a simple binary protocol - similar in spirit
to the chat server from Section 3. Clients send `SET key value` and `GET key` commands.
The server serializes responses in the ironkv binary format.

---

## What You've Learned: Course Completion

You made it. Look back at where you started - a C developer who knew pointers but
not ownership, who knew structs but not traits, who knew "write and pray" but not
the compile-time safety net.

Here is what you can now do:

**Safe, idiomatic Rust for systems programming.**  
Ownership, borrowing, lifetimes. The borrow checker is not an obstacle -
it is an always-on C++ undefined behavior scanner that runs at compile time.
You now write code where the contract is in the types, not in the comments.

**Production REST APIs.**  
Async Rust with Tokio. Axum or Actix for routing. Proper error handling with
`thiserror`/`anyhow`. Docker packaging. Health checks. You built this in Sections
4-5.

**Concurrent systems.**  
`Arc<Mutex<T>>`, `RwLock`, channels, `Rayon` for data parallelism. You understand
the Send/Sync model at a deep level. Data races are a compile error, not a latent bug.

**Unsafe Rust, FFI, and memory-mapped I/O.**  
You know the five unsafe superpowers. You know when to use them and when not to.
You can call any C library. You can map files directly into memory and parse
binary formats with zero copies.

**Production-grade systems.**  
Not toy code. Not tutorial exercises. Code that has CRC32 checksums, a write-ahead
log, correctness tools (Miri, ASan, fuzzing), and criterion benchmarks. Code you
could check into a real company's codebase and defend in a code review.

**What comes next:**

- **Async internals:** The Rust async book. Implement your own executor. Understand
  `Future`, `Poll`, and `Waker` at the implementation level.
- **Embedded Rust:** The Embedded Rust Book. Write firmware for a microcontroller.
  `no_std`, `embedded-hal`, RTIC for interrupt handling.
- **WebAssembly:** `wasm-pack`, `wasm-bindgen`. Compile Rust to WASM and run it
  in a browser. Rust's WASM toolchain is the best in the ecosystem.
- **Contributing to open source:** Find a crate you use, read the issues list,
  fix something. The Rust open source community is welcoming to contributors
  who read the code carefully before submitting.

You have the foundation. What you build with it is up to you.

```text
$ ironkv stats production.ikv
  Keys:        4,291,847
  File size:   2.3 GB
  Cache hits:  98.7%
  p99 latency: 0.4 ms

ironkv 0.1.0
```

Ship it.
