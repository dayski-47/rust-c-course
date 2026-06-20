# Doc 05 - Zero-Copy Serialization Patterns

🟡 Every storage engine is fundamentally a serialization problem: bytes on disk become structured data in memory, and structured data becomes bytes again. The difference between a fast storage engine and a slow one is often how much copying happens during that transformation.

ironkv stores key-value pairs where both keys and values are raw byte slices. The wire format - the on-disk binary layout - needs to be designed for two things: correctness and speed. This doc covers the tools that make parsing fast: zero-copy deserialization with serde, binary representations with `repr(C)`, and the `bytes::Bytes` type for shared ownership of byte buffers.

---

## What Zero-Copy Means

In normal deserialization, parsing a string involves:
1. Read bytes from the source buffer
2. Check that they're valid UTF-8
3. Allocate a `String` on the heap
4. Copy the bytes into the new allocation

For a storage engine reading thousands of records per second from mmap'd memory, step 3 and 4 are the bottleneck. Zero-copy deserialization skips them: instead of copying bytes into a new `String`, it returns a `&str` that *points into the original buffer*. No allocation. No copy.

---

## serde Fundamentals

`serde` separates your data model (your structs) from the format (JSON, bincode, your custom binary format). The same struct can serialize to and deserialize from any format that serde supports:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct EntryHeader {
    pub key_len: u32,
    pub value_len: u32,
    pub checksum: u32,
    pub flags: u8,
}
```

This single derive works with JSON (for debugging), `bincode` (for fast binary encoding), and custom binary formats.

### Common Serde Attributes for Storage Formats

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(deny_unknown_fields)]  // Reject extra data - strict mode for protocols
pub struct WalEntry {
    pub sequence: u64,

    #[serde(with = "bytes_as_base64")]  // Custom encoding for byte arrays in JSON
    pub key: Vec<u8>,

    #[serde(skip_serializing_if = "Option::is_none")]  // Omit if None
    pub value: Option<Vec<u8>>,

    #[serde(default = "default_op")]  // Use default if missing from old format
    pub op: EntryOp,
}

fn default_op() -> EntryOp { EntryOp::Put }

#[derive(Debug, Serialize, Deserialize, Clone, Copy)]
pub enum EntryOp {
    Put,
    Delete,
    Merge,
}
```

### Enum Representations for On-Disk Format

Different enum representations produce different binary shapes:

```rust
use serde::{Serialize, Deserialize};

// Internally tagged - best for human-readable debug formats
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum IndexEvent {
    Insert { key_hash: u64, offset: u64, len: u32 },
    Delete { key_hash: u64 },
    Compact { segment_id: u32 },
}
// JSON: {"type": "Insert", "key_hash": 12345, "offset": 4096, "len": 64}

// Untagged - for formats where shape alone disambiguates
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
pub enum KeyOrHash {
    Raw(Vec<u8>),
    Hash(u64),
}
```

For binary formats, use `bincode` for the actual wire format and reserve JSON for debug output and API responses.

---

## Zero-Copy Deserialization with `&'de str` and `&'de [u8]`

The key: use borrowed types (`&'de str`, `&'de [u8]`) instead of owned types (`String`, `Vec<u8>`) in your deserialization structs.

```rust
use serde::Deserialize;

// Owned - allocates on every parse
#[derive(Deserialize)]
struct OwnedRecord {
    key: String,    // allocates a new String
    value: Vec<u8>, // allocates a new Vec
}

// Zero-copy - borrows from the input buffer
#[derive(Deserialize)]
struct BorrowedRecord<'de> {
    key: &'de str,    // borrows from the input - no allocation
    value: &'de [u8], // borrows from the input - no allocation
}
```

The `'de` lifetime says "this struct can only live as long as the input buffer it was parsed from." When parsing from mmap'd memory - which is exactly what ironkv does - the input buffer lives for the lifetime of the mmap region, so borrowed records can live for as long as you need them.

```rust
// Parsing WAL entries from mmap'd memory
fn parse_wal_entries<'mmap>(mmap: &'mmap [u8]) -> Vec<BorrowedRecord<'mmap>> {
    // Each record borrows from mmap - no copies, no allocations
    // Records are valid as long as mmap is valid
    let mut records = Vec::new();
    let mut pos = 0;

    while pos < mmap.len() {
        let header = read_header(&mmap[pos..]);
        let key_end = pos + HEADER_SIZE + header.key_len as usize;
        let val_end = key_end + header.value_len as usize;

        records.push(BorrowedRecord {
            key: std::str::from_utf8(&mmap[pos + HEADER_SIZE..key_end]).unwrap(),
            value: &mmap[key_end..val_end],
        });

        pos = val_end;
    }

    records
}
```

---

## Binary Data with `repr(C)`

For structs that will be written directly to disk and read back with zero parsing, use `#[repr(C)]` to get a stable, predictable memory layout:

```rust
/// The 16-byte header that precedes every entry in the data file.
/// repr(C) guarantees field order and padding match C's layout rules.
/// This can be read directly from disk without deserialization.
#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct EntryHeader {
    pub key_len: u32,    // bytes 0-3
    pub value_len: u32,  // bytes 4-7
    pub checksum: u32,   // bytes 8-11
    pub flags: u8,       // byte 12
    pub _padding: [u8; 3], // bytes 13-15 - explicit, not compiler-chosen
}

const HEADER_SIZE: usize = std::mem::size_of::<EntryHeader>();  // = 16

impl EntryHeader {
    /// Read an EntryHeader from a byte slice.
    /// Safety: slice must be at least HEADER_SIZE bytes and properly aligned.
    pub fn from_bytes(bytes: &[u8]) -> Option<&Self> {
        if bytes.len() < HEADER_SIZE {
            return None;
        }
        // Safety: EntryHeader is repr(C) with known size and alignment.
        // The bytes slice is valid and at least HEADER_SIZE bytes.
        unsafe {
            let ptr = bytes.as_ptr() as *const EntryHeader;
            Some(&*ptr)
        }
    }

    pub fn to_bytes(&self) -> &[u8] {
        // Safety: EntryHeader is repr(C), all bit patterns valid.
        unsafe {
            std::slice::from_raw_parts(
                self as *const EntryHeader as *const u8,
                HEADER_SIZE,
            )
        }
    }
}
```

**Why explicit `_padding`?** Without it, the compiler might pad the struct differently on different platforms or in different compiler versions. Explicit padding makes the layout stable, portable, and self-documenting.

**Why `repr(C)` and not `repr(packed)`?** `repr(packed)` removes all padding and alignment, which breaks safe access to fields (unaligned reads are undefined behavior on most architectures). `repr(C)` respects alignment while making the layout deterministic.

---

## The `bytes::Bytes` Type for Shared Ownership

When multiple parts of ironkv need to hold onto the same data - the LRU cache, the in-flight write buffer, the response being assembled - you don't want to copy the bytes for each holder. The `bytes::Bytes` type provides shared ownership of a byte buffer:

```toml
[dependencies]
bytes = "1"
```

```rust
use bytes::Bytes;

/// A value stored in ironkv.
/// Bytes is Arc<Vec<u8>> with range slicing - shared ownership, zero-copy slicing.
#[derive(Clone, Debug)]
pub struct StoredValue {
    raw: Bytes,
}

impl StoredValue {
    pub fn new(data: impl Into<Bytes>) -> Self {
        StoredValue { raw: data.into() }
    }

    /// Zero-copy slice - returns a Bytes that shares ownership of the same allocation.
    pub fn slice(&self, range: impl std::ops::RangeBounds<usize>) -> Bytes {
        self.raw.slice(range)
    }

    pub fn len(&self) -> usize { self.raw.len() }
    pub fn as_ref(&self) -> &[u8] { &self.raw }
}

// In the LRU cache:
struct LruCache {
    entries: std::collections::HashMap<Vec<u8>, StoredValue>,
}

impl LruCache {
    pub fn get(&self, key: &[u8]) -> Option<Bytes> {
        // Returns a Bytes that shares ownership with the cache entry.
        // No copy - just an Arc reference count increment.
        self.entries.get(key).map(|v| v.raw.clone())
    }

    pub fn insert(&mut self, key: Vec<u8>, value: Bytes) {
        self.entries.insert(key, StoredValue { raw: value });
    }
}
```

`Bytes::clone()` is O(1) - it increments a reference count, not the data. Multiple components can hold `Bytes` references to the same underlying allocation. When the last holder drops it, the allocation is freed.

---

## bincode: Fast Binary Encoding

For ironkv's WAL and index formats, JSON is too verbose. Use `bincode` for compact, fast binary serialization:

```toml
[dependencies]
bincode = "1"
serde = { version = "1", features = ["derive"] }
```

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct WalEntry {
    pub sequence: u64,
    pub key: Vec<u8>,
    pub value: Option<Vec<u8>>,
    pub op: EntryOp,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum EntryOp { Put, Delete }

// Write a WAL entry:
fn write_wal_entry(entry: &WalEntry, writer: &mut impl std::io::Write) -> std::io::Result<()> {
    let encoded = bincode::serialize(entry).map_err(|e| {
        std::io::Error::new(std::io::ErrorKind::InvalidData, e)
    })?;
    
    // Write length prefix so we can read back variable-length entries
    let len = encoded.len() as u32;
    writer.write_all(&len.to_le_bytes())?;
    writer.write_all(&encoded)?;
    Ok(())
}

// Read a WAL entry:
fn read_wal_entry(reader: &mut impl std::io::Read) -> std::io::Result<WalEntry> {
    let mut len_buf = [0u8; 4];
    reader.read_exact(&mut len_buf)?;
    let len = u32::from_le_bytes(len_buf) as usize;
    
    let mut buf = vec![0u8; len];
    reader.read_exact(&mut buf)?;
    
    bincode::deserialize(&buf).map_err(|e| {
        std::io::Error::new(std::io::ErrorKind::InvalidData, e)
    })
}
```

**bincode vs custom binary format:**
- bincode is simple to add and handles arbitrary serde-compatible types
- Custom formats give you full control over layout and can be zero-copy
- For the WAL (write path), bincode is fine - writes are sequential and rare
- For the data file (read path), custom `repr(C)` headers with mmap are faster

---

## ironkv On-Disk Format Design

With these tools in hand, ironkv's on-disk format looks like:

```
Data File (.ikv):
┌─────────────────────────────────────────────┐
│  File Header (32 bytes, repr(C))            │
│  - magic: [u8; 4] = b"IRNV"               │
│  - version: u32                             │
│  - created_at: u64                          │
│  - entry_count: u64                         │
│  - padding: [u8; 8]                         │
├─────────────────────────────────────────────┤
│  Entry 0:                                   │
│  ┌─ EntryHeader (16 bytes, repr(C)) ──────┐ │
│  │  key_len: u32                          │ │
│  │  value_len: u32                        │ │
│  │  checksum: u32                         │ │
│  │  flags: u8, _padding: [u8; 3]         │ │
│  └────────────────────────────────────────┘ │
│  key bytes (key_len bytes)                  │
│  value bytes (value_len bytes)              │
├─────────────────────────────────────────────┤
│  Entry 1:                                   │
│  ...                                        │
└─────────────────────────────────────────────┘

WAL File (.wal):
┌─────────────────────────────────────────────┐
│  length: u32 (LE)                           │
│  bincode-serialized WalEntry                │
├─────────────────────────────────────────────┤
│  length: u32 (LE)                           │
│  bincode-serialized WalEntry                │
│  ...                                        │
└─────────────────────────────────────────────┘
```

The data file is read via mmap with `repr(C)` headers - O(1) field access, no parsing. The WAL is written sequentially with bincode - simple, correct, easy to recover. The two formats match the two access patterns: random reads (data file) and sequential writes (WAL).

---

## Checksum Validation

Every on-disk entry includes a checksum. Verify it on every read:

```rust
fn crc32(data: &[u8]) -> u32 {
    // Use the crc32fast crate for hardware-accelerated checksums
    crc32fast::hash(data)
}

pub fn read_entry<'mmap>(mmap: &'mmap [u8], offset: usize) -> Option<(&'mmap [u8], &'mmap [u8])> {
    let header = EntryHeader::from_bytes(&mmap[offset..])?;
    
    let key_start = offset + HEADER_SIZE;
    let val_start = key_start + header.key_len as usize;
    let val_end = val_start + header.value_len as usize;
    
    if val_end > mmap.len() { return None; }
    
    let key = &mmap[key_start..val_start];
    let value = &mmap[val_start..val_end];
    
    // Verify checksum - both key and value
    let expected = crc32(&[key, value].concat());
    if expected != header.checksum {
        return None;  // Corrupt entry
    }
    
    Some((key, value))
}
```

The checksum is the last line of defense against on-disk corruption. It costs a few microseconds per read but catches the class of bugs that otherwise manifest as wrong data silently returned to the caller.

---

## Exercises

**Exercise 1 - Zero-Copy Deserialization**

Implement `BorrowedRecord<'de>` with `#[derive(Deserialize)]` using `&'de str` and
`&'de [u8]` fields. Write a test that:
1. Creates a JSON string `{"key": "hello", "value": [72,101,108,108,111]}`
2. Deserializes it into a `BorrowedRecord<'_>` using `serde_json::from_str`
3. Asserts that `record.key == "hello"` **without** calling `.to_owned()` - the key
   should borrow directly from the input string
4. Verifies with `std::ptr::eq` that the key field's pointer is within the original string

This confirms zero-copy: no allocation happened for the key.

**Exercise 2 - repr(C) Size Assertion**

Define this struct and add a compile-time size assertion:

```rust
#[repr(C)]
struct WalEntry {
    op: u8,
    _pad1: [u8; 3],
    key_len: u32,
    value_len: u32,
    crc32: u32,
}
```

Add `const _: () = assert!(std::mem::size_of::<WalEntry>() == 16, "WalEntry must be 16 bytes");`.
Then add a `timestamp: u64` field and observe the compile error that tells you the
size changed. Fix the struct by removing the field, and verify the assertion passes.

**Exercise 3 - Bytes Roundtrip**

Implement a `WalEntryWriter` that serializes ironkv WAL entries using `bincode`:
1. Write `WalEntry` header with `bincode::serialize`
2. Write `key_len` bytes of key data
3. Write `value_len` bytes of value data
4. Write `crc32` checksum of key ++ value

Write a corresponding `WalEntryReader` using `bincode::deserialize_from`. Write a
roundtrip test that writes 100 entries and reads them back, verifying each entry's
key, value, and CRC32.
