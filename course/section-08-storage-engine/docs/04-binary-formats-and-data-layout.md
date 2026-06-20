# 04 - Binary Formats and Data Layout

> **Difficulty:** 🟡 think about it  
> **You'll learn:** Why custom binary formats beat JSON for storage engines, how to
> design a file format with `#[repr(C)]` headers and variable-length records,
> byte order handling, CRC32 checksums, and varint encoding.

---

## Why a Custom Binary Format

JSON is fine for APIs and config files. For a storage engine, it is the wrong tool.

| Concern | JSON | Custom Binary |
|---------|------|---------------|
| Parse overhead | High - tokenize every char | Near-zero - seek to offset, cast bytes |
| Disk space | Large - field names stored repeatedly | Compact - only data |
| Random access | Impossible without index | Direct offset arithmetic |
| Type fidelity | Lossy - all numbers as floats | Exact - u32 is always 4 bytes |
| Corruption detection | None | Built-in CRC32 per record |

A custom binary format lets you memory-map the file and access any record in O(1)
via its offset in the index. No parsing step in the read hot path. That is how
RocksDB and every serious storage engine works.

---

## `#[repr(C)]` Structs as File Headers

The file header is a fixed-size block at the start of the file. It tells you
everything you need to open and validate the file. Define it with `#[repr(C)]`
so the layout is deterministic:

```rust
use bytemuck::{Pod, Zeroable};

/// File header for the ironkv data file (.ikv).
/// Exactly 64 bytes, laid out as C would lay it out.
///
/// Byte layout:
///   [0..4]   magic          "IKVF"
///   [4..8]   version        u32 little-endian
///   [8..16]  entry_count    u64 little-endian
///   [16..24] index_offset   u64 little-endian (offset where index section starts)
///   [24..32] created_at     u64 unix timestamp seconds
///   [32..36] checksum       u32 CRC32 of bytes [0..32]
///   [36..64] _padding       reserved, must be zero
#[derive(Debug, Clone, Copy, Pod, Zeroable)]
#[repr(C)]
pub struct FileHeader {
    pub magic: [u8; 4],
    pub version: u32,
    pub entry_count: u64,
    pub index_offset: u64,
    pub created_at: u64,
    pub checksum: u32,
    pub _padding: [u8; 28],
}

const _: () = assert!(std::mem::size_of::<FileHeader>() == 64,
    "FileHeader must be exactly 64 bytes");

impl FileHeader {
    pub const MAGIC: [u8; 4] = *b"IKVF";
    pub const VERSION: u32 = 1;

    pub fn is_valid(&self) -> bool {
        self.magic == Self::MAGIC && self.version == Self::VERSION
    }

    /// Serialize the header to exactly 64 bytes.
    pub fn to_bytes(&self) -> [u8; 64] {
        // SAFETY: FileHeader is Pod, so any bit pattern is valid.
        //         The size is exactly 64 bytes, guaranteed by the const assert above.
        *bytemuck::bytes_of(self).try_into().unwrap()
    }

    /// Deserialize from raw bytes. Returns None if the slice is too short.
    pub fn from_bytes(data: &[u8]) -> Option<&FileHeader> {
        if data.len() < 64 { return None; }
        bytemuck::try_from_bytes(&data[..64]).ok()
    }
}
```

One important habit: verify the size with a compile-time `assert!`. If you add
a field and the struct grows beyond 64 bytes, the compiler catches it immediately -
not when you open a file and the offsets are all wrong.

---

## Byte Order: Little-Endian and You

x86/x86-64 is little-endian: the least significant byte is stored first. Most
network protocols use big-endian. Your file format should pick one and be explicit.

For a local-first storage engine, little-endian is the natural choice - no conversion
on the CPU that writes and reads the file. But make it explicit in code so anyone
reading your implementation knows it is a deliberate choice:

```rust
// Always use explicit byte-order methods. Never cast directly.

fn write_u32_le(value: u32) -> [u8; 4] {
    value.to_le_bytes()
}

fn read_u32_le(bytes: &[u8; 4]) -> u32 {
    u32::from_le_bytes(*bytes)
}

// In practice, when reading from a mmap slice:
let entry_count = u64::from_le_bytes(
    mmap[8..16].try_into().expect("slice is exactly 8 bytes")
);
```

The `to_le_bytes()` / `from_le_bytes()` family exists for all numeric types:
`u16`, `u32`, `u64`, `i32`, `f64`, etc. Use them always. Never write:

```rust
// WRONG: assumes host byte order, breaks on big-endian systems
let value = unsafe { *(bytes.as_ptr() as *const u32) };
```

---

## The Record Format

Each key-value pair in the data file is a record with a fixed-size header followed
by variable-length data:

```text
Record Layout:
  Offset  Size   Field
  ------  ----   -----
  0       4      key_len    (u32 LE) - length of the key in bytes
  4       4      value_len  (u32 LE) - length of the value in bytes; u32::MAX = tombstone
  8       4      crc32      (u32 LE) - CRC32 of (key_bytes ++ value_bytes)
  12      key_len  key      - raw key bytes
  12+key_len  value_len  value - raw value bytes (possibly compressed)

Total record size = 12 + key_len + value_len (or 12 + key_len for tombstone)
```

The tombstone convention - `value_len = u32::MAX` - marks a deleted key without
physically removing it from the file. This makes writes append-only, which is
fast and crash-safe. Deleted space is reclaimed during compaction.

---

## CRC32 Checksums

Every record has a CRC32. On read, you recompute the CRC and compare. If they
differ, the file is corrupt. This is the minimum viable corruption detection for
a storage engine.

```toml
crc32fast = "1"
```

```rust
use crc32fast::Hasher;

fn compute_crc32(key: &[u8], value: &[u8]) -> u32 {
    let mut hasher = Hasher::new();
    hasher.update(key);
    hasher.update(value);
    hasher.finalize()
}

fn write_record(writer: &mut impl std::io::Write, key: &[u8], value: &[u8])
    -> std::io::Result<u64>
{
    let crc = compute_crc32(key, value);

    // Fixed-size record header
    let key_len_bytes = (key.len() as u32).to_le_bytes();
    let val_len_bytes = (value.len() as u32).to_le_bytes();
    let crc_bytes = crc.to_le_bytes();

    writer.write_all(&key_len_bytes)?;
    writer.write_all(&val_len_bytes)?;
    writer.write_all(&crc_bytes)?;
    writer.write_all(key)?;
    writer.write_all(value)?;

    Ok(12 + key.len() as u64 + value.len() as u64)
}

#[derive(Debug, thiserror::Error)]
pub enum ReadError {
    #[error("CRC32 mismatch: expected {expected:#010x}, got {got:#010x}")]
    Checksum { expected: u32, got: u32 },
    #[error("truncated record at offset {offset}")]
    Truncated { offset: u64 },
}

fn read_record(data: &[u8], offset: usize)
    -> Result<(Vec<u8>, Vec<u8>), ReadError>
{
    if offset + 12 > data.len() {
        return Err(ReadError::Truncated { offset: offset as u64 });
    }
    let key_len = u32::from_le_bytes(data[offset..offset+4].try_into().unwrap()) as usize;
    let val_len = u32::from_le_bytes(data[offset+4..offset+8].try_into().unwrap()) as usize;
    let stored_crc = u32::from_le_bytes(data[offset+8..offset+12].try_into().unwrap());

    let key_start = offset + 12;
    let val_start = key_start + key_len;
    let val_end   = val_start + val_len;

    if val_end > data.len() {
        return Err(ReadError::Truncated { offset: offset as u64 });
    }

    let key = &data[key_start..val_start];
    let val = &data[val_start..val_end];

    let actual_crc = compute_crc32(key, val);
    if stored_crc != actual_crc {
        return Err(ReadError::Checksum { expected: stored_crc, got: actual_crc });
    }

    Ok((key.to_vec(), val.to_vec()))
}
```

`crc32fast` uses SIMD instructions on x86 (PCLMULQDQ / SSE4.2) automatically.
Hardware CRC32 on modern Intel/AMD is extremely fast - typically 10+ GB/s.

---

## Parse-Don't-Validate

A principle worth internalizing: at the boundary where raw bytes enter your system,
parse them into a typed structure immediately. Never pass raw bytes around and validate
them later.

```rust
// BAD: Pass raw bytes around and check magic "later"
fn open_file_bad(data: &[u8]) -> &[u8] {
    data  // caller must remember to check magic
}

// GOOD: Parse at the boundary, return a typed value or an error
#[derive(Debug, thiserror::Error)]
pub enum OpenError {
    #[error("not an ironkv file (magic mismatch)")]
    WrongMagic,
    #[error("unsupported version {0}")]
    UnsupportedVersion(u32),
    #[error("header checksum failed")]
    CorruptHeader,
    #[error("file too short")]
    TooShort,
}

fn open_file(data: &[u8]) -> Result<&FileHeader, OpenError> {
    let header = FileHeader::from_bytes(data).ok_or(OpenError::TooShort)?;
    if header.magic != FileHeader::MAGIC {
        return Err(OpenError::WrongMagic);
    }
    if header.version != FileHeader::VERSION {
        return Err(OpenError::UnsupportedVersion(header.version));
    }
    // Verify header checksum (CRC32 of first 32 bytes)
    let actual = {
        let mut h = Hasher::new();
        h.update(&data[..32]);
        h.finalize()
    };
    if actual != header.checksum {
        return Err(OpenError::CorruptHeader);
    }
    Ok(header)
}
```

After `open_file` returns `Ok(&FileHeader)`, every caller knows they have a valid,
verified header. The magic check, version check, and checksum are never repeated.

---

## `#[repr(packed)]`: Compact but Careful

`#[repr(packed)]` eliminates padding between fields, making the struct as small as
possible. This is useful for dense record headers.

```rust
#[repr(C, packed)]
struct RecordHeader {
    key_len: u32,
    value_len: u32,
    crc32: u32,
}
// sizeof = 12 - no padding, exactly right
```

The danger: taking a reference to a field of a packed struct can produce an
unaligned pointer. Reading an unaligned `u32` is UB on architectures that require
alignment (ARM without unaligned access support). On x86 it works but may be slower.

```rust
// WRONG with repr(packed):
let h: RecordHeader = ...;
let p = &h.value_len;  // May be unaligned! UB on strict-alignment architectures.

// CORRECT with repr(packed):
let value_len = h.value_len;  // Copy into a local - now aligned
```

For ironkv, `RecordHeader` has three `u32` fields (12 bytes) and the record
immediately follows with key bytes. As long as the file starts on a page boundary
(guaranteed by mmap) and records are naturally aligned from the start, you are fine.

---

## Variable-Length Integer (VarInt) Encoding

For keys and values where lengths are typically small, encoding the length as a
fixed `u32` wastes 3 bytes on most records. Varint encoding uses 1 byte for values
0-127, 2 bytes for 128-16383, etc.:

```rust
/// Encode a u64 as a varint (LEB128). Returns number of bytes written.
fn encode_varint(mut value: u64, buf: &mut Vec<u8>) -> usize {
    let mut count = 0;
    loop {
        let byte = (value & 0x7F) as u8;
        value >>= 7;
        if value == 0 {
            buf.push(byte);  // High bit clear = last byte
            count += 1;
            break;
        } else {
            buf.push(byte | 0x80);  // High bit set = more bytes follow
            count += 1;
        }
    }
    count
}

/// Decode a varint from a byte slice. Returns (value, bytes_consumed).
fn decode_varint(buf: &[u8]) -> Option<(u64, usize)> {
    let mut value: u64 = 0;
    let mut shift = 0u32;
    for (i, &byte) in buf.iter().enumerate() {
        value |= ((byte & 0x7F) as u64) << shift;
        if byte & 0x80 == 0 {
            return Some((value, i + 1));
        }
        shift += 7;
        if shift >= 64 { return None; }  // Overflow
    }
    None  // Truncated
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn varint_roundtrip() {
        for &n in &[0u64, 1, 127, 128, 16383, 16384, u64::MAX] {
            let mut buf = Vec::new();
            encode_varint(n, &mut buf);
            let (decoded, _) = decode_varint(&buf).unwrap();
            assert_eq!(decoded, n, "roundtrip failed for {n}");
        }
    }
}
```

For ironkv's milestone goals, fixed `u32` for key_len and value_len is fine.
Varint is a stretch goal that improves space efficiency for many short keys.

---

## Full File Format Summary

```text
ironkv Data File (.ikv)
=======================

Offset   Size    Description
------   ----    -----------
0        64      File Header (FileHeader struct, #[repr(C)])
64       var     Data Section: sequential records
                   Each record:
                     0   4    key_len: u32 LE
                     4   4    value_len: u32 LE  (u32::MAX = tombstone)
                     8   4    crc32: u32 LE  (of key ++ value)
                     12  key_len   key bytes
                     12+key_len  value_len  value bytes (optionally zstd compressed)
var      var     Index Section (at offset stored in header.index_offset)
                   For each unique key (sorted lexicographically):
                     0   4    key_len: u32 LE
                     4   key_len  key bytes
                     4+key_len  8  data_offset: u64 LE  (offset of value in data section)
                     12+key_len  4  value_len: u32 LE

ironkv WAL File (.wal)
======================

Append-only log. Replayed on startup to recover uncommitted writes.

Each WAL Entry:
  0   1   op: u8   (0x01 = Set, 0x02 = Delete)
  1   4   key_len: u32 LE
  5   4   value_len: u32 LE  (0 for Delete op)
  9   key_len  key bytes
  9+key_len  value_len  value bytes
  9+key_len+value_len  4  crc32: u32 LE  (of op ++ key ++ value)
```

---

## How It Breaks

**Not handling endianness.** Write on x86 (little-endian), read on ARM big-endian (or network-order code). The `u32` value `1` becomes `0x01000000` instead of `0x00000001`. Always use explicit byte order methods.

**Off-by-one in record parsing.** Your record parser reads `key_len` bytes and then `value_len` bytes. If `key_len` is wrong (corrupted), you read the wrong bytes and all subsequent records are misaligned. CRC32 catches this.

**Not validating magic bytes.** You open a non-storage-engine file, try to parse it as one, and interpret garbage bytes as a valid record count. Your program does nonsensical things. Always check the magic bytes and version first.

**Integer overflow in offsets.** A `u32` offset maxes out at 4 GB. If your database file grows past 4 GB, `u32` offsets overflow silently. Use `u64` for file offsets.

---

## Common Mistakes

**Forgetting to explicitly set padding bytes.** If your `#[repr(C)]` struct has
implicit compiler-inserted padding, `bytemuck::bytes_of()` will expose those padding
bytes (which have undefined content). Use `Zeroable` derive and call `.zeroed()`
to ensure they're always zero, or use explicit `_padding: [u8; N]` fields.

**Assuming host byte order.** If you write a `u32` to a file on a little-endian
machine and read it on a big-endian machine, you get garbage. Always use
`to_le_bytes()` / `from_le_bytes()`. Pick one byte order and document it.

**Writing the index before the data.** The `index_offset` field in the header
cannot be known until after all data records are written. Write data first,
note the current file position, then write the index, then update the header.
Or update the header at the very end (a second write to offset 0).

**Accepting any `value_len` from untrusted input.** If you read a record header
where `value_len = 0xFFFFFFFE` (just below tombstone), you will try to allocate
a 4 GB buffer. Always validate lengths against the remaining file size before
allocating or slicing.
