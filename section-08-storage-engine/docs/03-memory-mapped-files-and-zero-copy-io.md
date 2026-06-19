# 03 — Memory-Mapped Files and Zero-Copy I/O

> **Difficulty:** 🟡 think about it  
> **You'll learn:** How `mmap` works at the OS level, the `memmap2` crate, zero-copy
> parsing of binary files, the `bytes` crate for reference-counted slices, and the
> specific dangers that make mmap `unsafe` in Rust.

---

## What Memory Mapping Actually Is

Traditional file I/O: you call `read()`, the kernel copies data from the page cache
into your userspace buffer. Two copies if the file isn't cached. One syscall minimum
per read.

Memory-mapped I/O is different. The OS maps a region of the file into your process's
virtual address space. When you access `mmap[i]`, if the page isn't in RAM yet, a
page fault fires, the OS loads that page from disk into the page cache, and execution
resumes — transparently. No explicit read call. The data is just *there*, at a memory
address.

```text
Without mmap:
  File on disk → [kernel page cache] → syscall → [user buffer] → your code
  Two copies. One syscall per read batch.

With mmap:
  File on disk → [kernel page cache] → virtual address mapping → your code
  Zero copies. No syscall in the hot read path.
  Page faults only when pages are first accessed.
```

For a storage engine doing random reads across a large file, mmap is extremely useful:
seeking to a record is just pointer arithmetic. No lseek + read pair per lookup.

---

## The `memmap2` Crate

```toml
memmap2 = "0.9"
```

Basic usage — mapping a file read-only:

```rust
use memmap2::Mmap;
use std::fs::File;

fn read_file_with_mmap(path: &str) -> std::io::Result<()> {
    let file = File::open(path)?;

    // SAFETY: We assume no other process truncates or replaces this file
    //         while the mapping is live. If the file shrinks, accessing
    //         pages past the new end will produce SIGBUS.
    let mmap = unsafe { Mmap::map(&file)? };

    // mmap is now a &[u8]-like view of the entire file
    println!("File size: {} bytes", mmap.len());
    println!("First 4 bytes: {:?}", &mmap[..4]);

    // Access any byte by index — no syscall
    let byte_at_1000 = mmap[1000];

    // Get a slice from the middle of the file — zero copy
    let chunk = &mmap[512..1024];

    Ok(())
}
// mmap is dropped here — the OS unmaps the region
```

Writing with `MmapMut`:

```rust
use memmap2::MmapMut;
use std::fs::OpenOptions;

fn write_with_mmap(path: &str) -> std::io::Result<()> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .open(path)?;

    // Pre-allocate the file to the desired size before mapping.
    // mmap cannot grow the file; mapping a 0-byte file panics.
    file.set_len(4096)?;

    // SAFETY: Same preconditions as read-only mmap plus: we are the sole
    //         writer. No other process writes to this file concurrently.
    let mut mmap = unsafe { MmapMut::map_mut(&file)? };

    // Write directly into the mapped region
    let header_bytes = b"IKVF";
    mmap[..4].copy_from_slice(header_bytes);
    mmap[4..8].copy_from_slice(&1u32.to_le_bytes()); // version = 1

    // Flush modified pages to disk (required for durability)
    mmap.flush()?;       // flushes entire mapping
    // or: mmap.flush_range(0, 64)?;  // flush just the header

    Ok(())
}
```

---

## Why `Mmap::map` is `unsafe`

You might wonder why creating an mmap requires an `unsafe` block when it looks like
a straightforward OS operation. Two reasons:

**1. External file modification.** If another process writes to the file while you have it
mapped, you will see changed data mid-read. In most cases this is benign, but for a
storage engine parsing binary structures, a partial write by another process can make
your parsed struct internally inconsistent.

**2. File truncation causes SIGBUS.** If the file is truncated while mapped, and you
access a page that no longer has backing disk space, the OS sends `SIGBUS` to your
process. This terminates the process by default and cannot be caught as a Rust error.
No `Result`, no panic — just death.

The `unsafe` is there to force you to acknowledge these conditions and document what
you are guaranteeing about the file's lifetime and stability.

In ironkv, your invariant is: "only this process opens and writes to the data file."
That is sufficient to make `Mmap::map` sound.

---

## Zero-Copy Parsing: Structs from Raw Bytes

Once you have a `Mmap`, you have `&[u8]` pointing directly into the file. The next
step is interpreting those bytes as typed structures. Here is where performance lives.

**The unsafe way (transmute) — do not do this:**

```rust
// EXTREMELY DANGEROUS. Do not do this.
let header: FileHeader = unsafe {
    std::mem::transmute_copy(&mmap[0])
};
// If FileHeader's size doesn't match, or alignment is wrong, this is UB.
// If the file data is malicious or corrupt, this reads garbage as a Header.
```

**The safe way — bytemuck:**

```toml
bytemuck = { version = "1", features = ["derive"] }
```

```rust
use bytemuck::{Pod, Zeroable};

/// File header, exactly 64 bytes.
/// Pod: all bit patterns are valid (no padding bytes hidden, no invalid enum states)
/// Zeroable: a zero-initialized instance is valid
/// repr(C): byte layout matches what C and disk files expect
#[derive(Debug, Clone, Copy, Pod, Zeroable)]
#[repr(C)]
pub struct FileHeader {
    pub magic: [u8; 4],        // "IKVF"
    pub version: u32,
    pub entry_count: u64,
    pub index_offset: u64,
    pub created_at: u64,
    pub checksum: u32,
    pub _padding: [u8; 28],    // fill to 64 bytes
}

const HEADER_SIZE: usize = std::mem::size_of::<FileHeader>();  // must be 64

fn parse_header(mmap: &Mmap) -> Option<&FileHeader> {
    if mmap.len() < HEADER_SIZE {
        return None;
    }
    // bytemuck::from_bytes panics if the slice length doesn't match size_of::<FileHeader>
    // and if alignment is wrong. For mmap, alignment depends on the OS mapping address.
    // Use try_from_bytes if you want a Result instead of a panic.
    let header: &FileHeader = bytemuck::from_bytes(&mmap[..HEADER_SIZE]);
    // No copy. header is a typed reference into the memory-mapped file.
    Some(header)
}
```

Notice that `&mmap[..HEADER_SIZE]` and the resulting `&FileHeader` point into the
same memory. No data was copied. Reading `header.entry_count` reads directly from
the mapped file page.

---

## Iterating Through Records

After parsing the header, you need to walk the data section reading records. The
pattern is a cursor — track your current offset and advance it:

```rust
fn rebuild_index(mmap: &Mmap, header: &FileHeader) -> BTreeMap<Vec<u8>, (u64, u32)> {
    let mut index = BTreeMap::new();
    let mut offset = HEADER_SIZE;

    while offset + 12 <= mmap.len() {  // 12 = key_len + value_len + crc32
        let key_len   = u32::from_le_bytes(mmap[offset..offset+4].try_into().unwrap()) as usize;
        let value_len = u32::from_le_bytes(mmap[offset+4..offset+8].try_into().unwrap());
        let crc       = u32::from_le_bytes(mmap[offset+8..offset+12].try_into().unwrap());

        offset += 12;

        if offset + key_len > mmap.len() { break; }
        let key = mmap[offset..offset + key_len].to_vec();
        offset += key_len;

        if value_len == u32::MAX {
            // Tombstone: deleted key — remove from index
            index.remove(&key);
        } else {
            // Live record: insert into index
            // Store (data_offset, value_len) — data_offset points to the value bytes
            index.insert(key, (offset as u64, value_len));
        }

        offset += value_len as usize;
    }

    index
}
```

This is zero-copy: we read bytes from the mmap, parse lengths and keys from the
raw bytes, and store offsets. The values remain on disk. A `get()` call later
reads directly from the mmap using the stored offset.

---

## The `bytes` Crate: Reference-Counted Slices

When multiple parts of your engine need to share the same buffer without copying,
use `bytes::Bytes`:

```toml
bytes = "1"
```

```rust
use bytes::Bytes;

// Bytes is reference-counted, like Arc<[u8]>.
// Clone is O(1) — increments a refcount, does not copy data.
let data = Bytes::from(vec![1u8, 2, 3, 4, 5, 6, 7, 8]);

let header = data.slice(0..4);   // Bytes pointing into the same buffer
let payload = data.slice(4..8);  // Another view, same allocation

// Both header and payload are valid; data can be dropped
drop(data);
// header and payload still valid — refcount was 3, now 2
println!("{:?}", header);   // [1, 2, 3, 4]
println!("{:?}", payload);  // [5, 6, 7, 8]
```

In a storage engine, `Bytes` is useful when returning values from the cache:
you can return a `Bytes` that shares the cache's internal buffer without copying
the value out. The cache entry stays alive until the last `Bytes` clone is dropped.

---

## Advising the OS: Sequential vs Random Access

The OS has heuristics for prefetching file pages. You can improve performance by
telling it your access pattern:

```rust
#[cfg(target_os = "linux")]
fn advise_sequential(mmap: &Mmap) {
    use memmap2::Advice;
    // Tell the OS: we'll read this mapping front to back
    // OS will prefetch pages aggressively
    mmap.advise(Advice::Sequential).ok();
}

#[cfg(target_os = "linux")]
fn advise_random(mmap: &Mmap) {
    use memmap2::Advice;
    // Tell the OS: we'll jump around this mapping randomly
    // OS will not bother with readahead prefetch
    mmap.advise(Advice::Random).ok();
}
```

For a storage engine doing sequential compaction: use `Advice::Sequential`.
For random point lookups: use `Advice::Random` to avoid wasting I/O on prefetch.

---

## Alignment Trap

One subtle issue when casting mmap bytes to structs: the mmap base address may
not satisfy the alignment requirement of your struct.

```rust
// If FileHeader requires 8-byte alignment (for u64 fields) and the mmap
// base address is 0x7f3400001004 (not 8-byte aligned), bytemuck will panic.

// Solution 1: bytemuck::try_from_bytes returns an error instead of panicking
let header = bytemuck::try_from_bytes::<FileHeader>(&mmap[..HEADER_SIZE])
    .expect("mmap address not aligned for FileHeader");

// Solution 2: Copy the bytes into an aligned buffer first (small one-time cost)
let mut header_buf = FileHeader::zeroed();
bytemuck::bytes_of_mut(&mut header_buf).copy_from_slice(&mmap[..HEADER_SIZE]);
// Now header_buf is on the stack with correct alignment
```

In practice, OS-managed mmaps on Linux/macOS are always at least page-aligned
(4096 bytes), which satisfies alignment requirements for any standard primitive type.
But be aware of this for the rare platforms where it might matter.

---

## When the File Grows

A memory-mapped region has a fixed size determined when you create the mapping.
If you append to the file, the new bytes are **not** visible through the existing mmap.

```rust
struct DataFile {
    file: File,
    mmap: Option<Mmap>,  // None when remapping is in progress
}

impl DataFile {
    fn remap(&mut self) -> std::io::Result<()> {
        // Drop the old mmap first, then create a new one at the new size
        self.mmap = None;
        // SAFETY: we are the sole writer; file is stable while we hold it
        let new_mmap = unsafe { Mmap::map(&self.file)? };
        self.mmap = Some(new_mmap);
        Ok(())
    }
}
```

The ironkv design avoids this by using mmap for **reads** and normal file writes
for **writes** (appending to the WAL and data file). After a flush, the read mmap
is refreshed to cover the new file size.

---

## How It Breaks

**SIGBUS when the file shrinks.** If you memory-map a 1 GB file and then the file is truncated to 500 MB, accessing bytes in the second 500 MB causes `SIGBUS` (not a recoverable error in Rust without signal handling). Never shrink a mapped file.

**Mmap not automatically syncing to disk.** Writes to `MmapMut` don't immediately hit disk. Use `mmap.flush()` for synchronous sync or `mmap.flush_async()` for async. On crash without a flush, recent writes are lost.

**File grows but mmap doesn't.** If you extend the file with `ftruncate` after mmapping, the mmap still has the original size. You need to unmap and remap to see the new bytes. This is why append-only storage engines often use a write-ahead log separately from the mmap.

**Virtual address space exhaustion.** On 32-bit systems, you can only address 4 GB. On 64-bit systems this is rarely a problem, but on memory-constrained systems (IoT, embedded), mmapping multiple large files can exhaust virtual memory even if you have swap.

---

## Common Mistakes

**Mapping a zero-byte file.** `Mmap::map` on an empty file returns an error on
some platforms and panics on others. Always check `file.metadata()?.len() > 0`
before mapping.

**Keeping a raw pointer from `mmap.as_ptr()` after dropping the Mmap.** The mapping
is unmapped when the `Mmap` is dropped. Any pointer into it becomes dangling.
Keep the `Mmap` alive for as long as you need to access the data.

**Not flushing `MmapMut` before drop.** Changes in a mutable mapping may not reach
disk unless you call `.flush()`. An OS crash without flush can lose your writes.
Always flush before acknowledging a write as durable.

**Accessing past the mapped length.** The `&mmap[offset..offset+size]` slice index
will panic if out of bounds. Always check bounds before slicing, especially when
parsing variable-length records where sizes come from untrusted file data.

---

## Safety Invariants to Maintain

- **File stability:** The file must not shrink while the mapping is live. Truncating a
  mapped file and then accessing pages past the new end delivers `SIGBUS`, which kills
  the process.

- **No double mmap on write:** Never have an `MmapMut` and modify the underlying file
  through the `File` handle simultaneously unless you know what you are doing. The
  changes may or may not be visible through the mmap depending on OS caching.

- **Alignment:** When casting mmap slices to typed structs via bytemuck or zerocopy,
  the slice must start at an address satisfying the type's alignment. On standard
  desktop/server hardware with OS-managed mmaps, this is usually guaranteed, but
  verify if you port to unusual platforms.

- **Mmap lifetime outlives all references:** Any `&[u8]` or `&T` derived from the
  mmap has an implicit lifetime tied to the `Mmap` object. If the `Mmap` is dropped,
  those references become dangling. Design your data structures to hold the `Mmap`
  alongside the references that borrow from it (using self-referential patterns via
  `Pin`, or by keeping both in the same struct with careful ownership).
