# Doc 07 - Memory Layout, repr(C), and the Wire

🔴 This is where low-level meets Rust - the C dev's home territory.

When you build a TCP protocol, you're working with bytes on the wire. The question is: how do those bytes map to Rust structs? And how does Rust lay out structs in memory compared to C? This doc answers both questions, because getting this wrong means either corrupted messages, undefined behavior, or both.

---

## How Rust Lays Out Structs by Default

In C, struct layout is mostly predictable: fields in declaration order, with padding added for alignment. In Rust, the **default layout is not guaranteed**. The compiler is free to reorder fields to minimize padding, and it does:

```rust
struct DefaultLayout {
    a: u8,   // 1 byte
    b: u64,  // 8 bytes - needs 8-byte alignment
    c: u8,   // 1 byte
}

// In C, this would be: a(1) + padding(7) + b(8) + c(1) + padding(7) = 24 bytes
// In Rust default layout: may be reordered to b(8) + a(1) + c(1) + padding(6) = 16 bytes
// Or some other ordering - it's up to the compiler.

fn main() {
    use std::mem;
    println!("Size of DefaultLayout: {}", mem::size_of::<DefaultLayout>());
    // Might print 16, 24, or something else - unspecified
}
```

This reordering is the compiler's optimization: it tries to minimize wasted padding bytes. But it means you **cannot** send a Rust struct over the network or share it with C code without telling the compiler to use a specific layout.

---

## `#[repr(C)]`: The Predictable Layout

`#[repr(C)]` tells the compiler: "lay this out exactly as C would - fields in declaration order, standard C alignment and padding rules."

```rust
#[repr(C)]
struct CLayout {
    a: u8,   // 1 byte
    b: u64,  // 8 bytes - needs 8-byte alignment
    c: u8,   // 1 byte
}

// Now the layout is exactly what C would produce:
// a(1) + padding(7) + b(8) + c(1) + padding(7) = 24 bytes
// This is predictable, stable, and matches what C expects.
```

Use `#[repr(C)]` when:
- The struct crosses the FFI boundary (calling C or being called by C)
- The struct is serialized to/from bytes that follow a known wire format
- You're doing `unsafe` transmutation between `[u8]` and your struct

---

## Understanding `size_of`, `align_of`, and Padding

These tools let you inspect layout at compile time:

```rust
use std::mem;

#[repr(C)]
struct MessageHeader {
    msg_type: u8,   // 1 byte
    _pad: u8,       // 1 byte padding (explicit - see below)
    length: u16,    // 2 bytes, needs 2-byte alignment
    sequence: u32,  // 4 bytes, needs 4-byte alignment
}

fn main() {
    println!("Size:      {}", mem::size_of::<MessageHeader>());   // 8
    println!("Alignment: {}", mem::align_of::<MessageHeader>());  // 4
    println!("u8 align:  {}", mem::align_of::<u8>());             // 1
    println!("u16 align: {}", mem::align_of::<u16>());            // 2
    println!("u32 align: {}", mem::align_of::<u32>());            // 4
}
```

**Alignment rule**: a type with alignment N must be stored at an address divisible by N. `u16` at offset 1 (odd byte) is misaligned - undefined behavior on many platforms, slow on others.

**Padding rule**: the compiler inserts padding bytes between fields to ensure each field is properly aligned. In `#[repr(C)]`, padding follows the same rules as C:
- After each field, pad until the next field's alignment requirement is met
- At the end of the struct, pad until the struct's overall alignment requirement is met (which is the maximum alignment of any field)

### Explicit vs. Implicit Padding

```rust
// Bad: implicit padding - struct layout is surprising
#[repr(C)]
struct ImplicitPad {
    flag: u8,       // 1 byte
    // 3 bytes of hidden padding here!
    value: u32,     // 4 bytes
}
// size = 8 bytes, but only 5 bytes of actual data

// Good: explicit padding - reader knows exactly what's in the struct
#[repr(C)]
struct ExplicitPad {
    flag: u8,       // 1 byte
    _pad: [u8; 3],  // 3 bytes explicit padding - self-documenting
    value: u32,     // 4 bytes
}
// Same size, same layout, but intent is visible
```

The explicit padding approach is strongly preferred for wire protocols. When someone reads the struct definition, they should see every byte accounted for.

---

## The Chat Protocol Header

The chat server uses a fixed-size binary header for every message:

```
┌─────────┬──────────┬──────────────────────────────────────┐
│ msg_type│  _pad    │       payload_length                 │
│  (u8)   │  (u8)    │           (u16 LE)                   │
│  1 byte │  1 byte  │           2 bytes                    │
└─────────┴──────────┴──────────────────────────────────────┘
 Total: 4 bytes header, followed by payload_length bytes of UTF-8 text
```

```rust
use std::mem;

#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct ChatHeader {
    pub msg_type: u8,       // 0x01 = chat message, 0x02 = join, 0x03 = leave
    pub _pad: u8,           // Reserved, always 0 - ensures u16 is aligned
    pub payload_length: u16, // Length of the UTF-8 payload in bytes (little-endian)
}

#[repr(u8)]
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum MessageType {
    Chat  = 0x01,
    Join  = 0x02,
    Leave = 0x03,
}

impl ChatHeader {
    pub const SIZE: usize = mem::size_of::<ChatHeader>(); // 4 bytes

    pub fn new(msg_type: MessageType, payload_len: u16) -> Self {
        ChatHeader {
            msg_type: msg_type as u8,
            _pad: 0,
            payload_length: payload_len,
        }
    }

    // Write header to bytes (handles endianness explicitly)
    pub fn to_bytes(self) -> [u8; 4] {
        [
            self.msg_type,
            self._pad,
            (self.payload_length & 0xFF) as u8,         // Low byte (little-endian)
            ((self.payload_length >> 8) & 0xFF) as u8,  // High byte
        ]
    }

    // Parse header from bytes
    pub fn from_bytes(bytes: &[u8]) -> Option<Self> {
        if bytes.len() < Self::SIZE {
            return None;
        }
        Some(ChatHeader {
            msg_type: bytes[0],
            _pad: bytes[1],
            payload_length: u16::from_le_bytes([bytes[2], bytes[3]]),
        })
    }
}
```

Notice: we parse endianness **explicitly** using `u16::from_le_bytes`. Never transmute raw bytes to a `u16` - the endianness of the bytes might not match the CPU's native endianness.

---

## Endianness: The Network Protocol Tax

C developers are familiar with `htons`/`ntohs`. In Rust, you have explicit byte-order methods on all integer types:

```rust
let value: u32 = 0xDEADBEEF;

// Write as little-endian (most x86/x64 systems are LE native)
let le_bytes: [u8; 4] = value.to_le_bytes();  // [EF, BE, AD, DE]
let be_bytes: [u8; 4] = value.to_be_bytes();  // [DE, AD, BE, EF]
let ne_bytes: [u8; 4] = value.to_ne_bytes();  // Native endian - depends on CPU

// Read back
let from_le = u32::from_le_bytes([0xEF, 0xBE, 0xAD, 0xDE]);  // 0xDEADBEEF
let from_be = u32::from_be_bytes([0xDE, 0xAD, 0xBE, 0xEF]);  // 0xDEADBEEF

// Network protocols typically use big-endian ("network byte order")
// Use .to_be_bytes() when writing to the wire, .from_be_bytes() when reading
```

**Rule**: never rely on native endianness for protocol data. Always use explicit `to_le_bytes()`/`from_le_bytes()` or `to_be_bytes()`/`from_be_bytes()`. Your chat protocol uses little-endian (common for modern protocols); document this in comments.

---

## Safe Byte Parsing vs. Unsafe Transmutation

You might be tempted to do this - it looks clean and it's exactly what C code does:

```rust
// WRONG - undefined behavior
let bytes: [u8; 4] = [0x01, 0x00, 0x05, 0x00];
let header: ChatHeader = unsafe { std::mem::transmute(bytes) };
```

This is UB for several reasons:
- The bytes must be properly aligned for `ChatHeader` (4-byte alignment), but `[u8; 4]` has 1-byte alignment
- The byte order of multi-byte fields depends on the platform's endianness
- If `ChatHeader` has any padding, the padding bytes might not be valid

The safe approach is to parse field-by-field:

```rust
// CORRECT - explicit parsing, handles alignment and endianness correctly
pub fn from_bytes(bytes: &[u8]) -> Option<ChatHeader> {
    if bytes.len() < 4 { return None; }

    Some(ChatHeader {
        msg_type:       bytes[0],
        _pad:           bytes[1],
        payload_length: u16::from_le_bytes([bytes[2], bytes[3]]),
    })
}
```

For high-performance code that needs zero-copy parsing of many structs, use the `zerocopy` or `bytemuck` crates - they verify alignment and layout safety at compile time:

```rust
// With bytemuck (for types where ALL bit patterns are valid):
use bytemuck::{Pod, Zeroable};

#[derive(Pod, Zeroable, Clone, Copy, Debug)]
#[repr(C)]
pub struct SensorReading {
    pub sensor_id: u16,
    pub flags:     u8,
    pub _reserved: u8,
    pub value:     u32,
}

// Now you can cast a byte slice directly - no copies, compile-time verified:
let bytes: &[u8] = &[...];
let readings: &[SensorReading] = bytemuck::cast_slice(bytes);
```

`bytemuck::cast_slice` verifies at compile time that `SensorReading` implements `Pod` (all bit patterns valid) and `Zeroable` (zero-initialized is valid). If your type has padding that needs to be zero (for security), use `zerocopy` instead which has stricter guarantees.

---

## `#[repr(C, packed)]`: Removing Padding

Sometimes you need to match a wire format where there is no padding - bytes are packed tightly regardless of alignment:

```rust
#[repr(C, packed)]
#[derive(Debug, Clone, Copy)]
pub struct PackedHeader {
    pub version: u8,    // byte 0
    pub msg_type: u8,   // byte 1
    pub length: u16,    // bytes 2-3 (may be unaligned!)
    pub checksum: u32,  // bytes 4-7 (may be unaligned!)
}
```

**Critical warning**: in a `packed` struct, fields may not be aligned. Taking a reference to a field (`&header.length`) creates an unaligned reference - this is undefined behavior on architectures that require alignment (ARM, RISC-V, most embedded targets), and causes a bus error.

```rust
// WRONG with packed structs:
let h = PackedHeader { version: 1, msg_type: 2, length: 256, checksum: 0 };
let len_ref = &h.length;  // UB: unaligned reference to u16

// CORRECT: copy the field out first (works because the field is Copy)
let len = h.length;  // OK: copies the u16 out (handles unaligned read)
println!("{len}");
```

**Use `packed` only when the wire format demands it.** Prefer designing your protocol with natural alignment when possible, or add explicit padding bytes. The extra padding bytes are worth it to avoid UB and maintain performance on all target platforms.

---

## `size_of` as a Compile-Time Safety Check

Use `size_of` and `assert_eq!` in const context to catch layout mistakes at compile time:

```rust
use std::mem;

#[repr(C)]
pub struct ChatHeader {
    pub msg_type:       u8,
    pub _pad:           u8,
    pub payload_length: u16,
}

// This fails at compile time if the struct is ever accidentally resized:
const _: () = assert!(mem::size_of::<ChatHeader>() == 4,
    "ChatHeader must be exactly 4 bytes for wire protocol compatibility");

// Or as a const:
impl ChatHeader {
    pub const WIRE_SIZE: usize = mem::size_of::<Self>();  // 4
}
```

This is particularly valuable for protocol structs that are exchanged between different systems or versions of the software. If a developer accidentally adds a field, the const assertion catches it before it becomes a wire compatibility bug.

---

## Field Offset Calculation

Sometimes you need to know the byte offset of a field within a struct (for zero-copy parsing or for writing a specific field in a buffer):

```rust
use std::mem;

#[repr(C)]
pub struct ChatHeader {
    pub msg_type:       u8,
    pub _pad:           u8,
    pub payload_length: u16,
}

fn main() {
    // Compute field offsets manually (for a repr(C) struct, this is predictable):
    println!("msg_type offset:       0");   // Always 0
    println!("_pad offset:           1");   // After u8
    println!("payload_length offset: 2");   // After u8 + u8 = 2

    // Or use the memoffset crate for safety:
    // use memoffset::offset_of;
    // println!("payload_length offset: {}", offset_of!(ChatHeader, payload_length));
}
```

For the chat server's receive loop, knowing that `payload_length` is at bytes 2-3 of the header lets you read the 4-byte header, parse the length, then read exactly `length` more bytes for the payload - no guessing, no overreading.

---

## How It Breaks

**Forgetting `#[repr(C)]` on protocol structs.**
Without `#[repr(C)]`, the compiler can reorder fields. A struct that happens to work correctly on one compiler version or architecture may produce corrupt data on another. Always use `#[repr(C)]` for anything that touches the wire.

**Assuming padding bytes are zero.**
When you create a `#[repr(C)]` struct with `Default`, the padding bytes have undefined values - they're whatever was in memory before. If you're sending the struct as raw bytes, those padding bytes are part of the message. Receivers should not interpret padding, but for security, zero it out explicitly:

```rust
// Safe: zero-initialize before filling in fields
let mut header: ChatHeader = unsafe { std::mem::zeroed() };
header.msg_type = MessageType::Chat as u8;
header.payload_length = payload.len() as u16;
```

**Mismatched endianness between client and server.**
If the client writes `u16::to_le_bytes()` and the server reads `u16::from_be_bytes()`, the length field is corrupted. Document the byte order in the protocol definition and enforce it in both directions. Big-endian is traditional for network protocols (RFC 791 specifies network byte order as big-endian). Little-endian is common in modern binary protocols (Protocol Buffers, FlatBuffers, and many others).

**Taking references to fields of packed structs.**
As described above - unaligned references in packed structs are UB. Compiler warning `-W unaligned_references` catches most of these, but make it a rule: never take `&field` in a packed struct. Always copy fields with `let val = packed_struct.field;`.

**`size_of` returning 0 for zero-sized types.**
If you have an empty struct or a struct containing only `PhantomData`, `size_of` returns 0. Multiplying buffer sizes by 0 gives you a 0-length buffer. Check for ZSTs explicitly if your code handles arbitrary types.
