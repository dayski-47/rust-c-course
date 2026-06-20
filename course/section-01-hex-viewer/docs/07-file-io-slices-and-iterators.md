# File I/O, Slices, and Iterators

🟡 Think about it - these are the hands-on tools that make hexview real.

Everything in the previous docs - ownership, error handling, types - was building a mental model. This doc is about the concrete tools you'll actually use when you write hexview. By the end, you'll know how to open a binary file, read it in 16-byte chunks, format each byte as hex, and output it correctly.

---

## Reading Binary Files

### Why File I/O looks the way it does

In C, reading a file looks like this:

```c
FILE *f = fopen("data.bin", "rb");
if (!f) { /* handle error */ }
unsigned char buf[16];
size_t n = fread(buf, 1, 16, f);
fclose(f);
```

You pass a pointer, get back a count. Errors come from `errno` or NULL returns. You must remember to call `fclose`.

Rust's version is structurally similar, but three things change:
1. Errors are `Result` - you can't ignore them
2. The file closes automatically (Drop)
3. Buffering is explicit - you opt in with `BufReader`

### Opening a File

`std::fs::File::open(path)` returns `Result<File, io::Error>`. If the file doesn't exist, you get `Err`. If it exists, you get `Ok(File)`. The `File` value owns the OS file descriptor.

You almost never use `File` directly for reading. You wrap it:

```rust
use std::fs::File;
use std::io::BufReader;

let file = File::open("data.bin")?;
let reader = BufReader::new(file);  // file is moved into BufReader
```

`BufReader::new(file)` takes ownership of the `File`. The variable `file` is no longer accessible after this line - ownership transferred.

### Why BufReader?

Raw `File::read()` calls the OS kernel for each read. On most systems, that's a context switch - slow. `BufReader` reads a large chunk from the OS into an internal buffer, then hands bytes out of that buffer for your smaller reads. Same data, fewer syscalls.

In C, `setvbuf()` does this. In Rust, you compose the buffering in explicitly. It's the same idea - you're just more in control of when it happens.

### Reading Bytes

The `Read` trait (from `std::io`) defines how to read bytes. `BufReader<File>` implements `Read`. The core method:

```rust
reader.read(&mut buf)  // returns Result<usize, io::Error>
```

The `usize` is the number of bytes actually placed into `buf`. This is not necessarily `buf.len()`. At the end of a file, it could be 7 or 3 or 0. **Always check the returned count.** Reading 0 bytes means EOF - you're done.

`read_exact(&mut buf)` is the variant that fills the buffer completely or errors. Use it when you know exactly how many bytes you expect (like reading a fixed file header). It returns `UnexpectedEof` error if the file is shorter than `buf.len()`.

### How It Breaks

**`read()` returning fewer bytes than the buffer size is not an error.** At the end of a 37-byte file, reading into a 16-byte buffer gives you `Ok(5)` on the last read. If you don't check the count and assume 16 bytes, you'll process old/garbage data in the tail of the buffer.

**Not flushing writes.** If you write to a `BufWriter` (the write equivalent) and don't flush, the last chunk of data may not reach disk before the file is dropped. For reads, this doesn't apply.

**Seeking past the end of file.** `seek(SeekFrom::Start(9999))` on a 100-byte file does not error - but the next `read()` returns `Ok(0)` immediately. You don't get an error for seeking out of range; you get EOF on the next read.

---

## Byte Arrays and Slices

### The types for raw binary data

A byte in Rust is `u8` - `unsigned char` in C. It holds values 0–255.

A fixed array of bytes: `[u8; 16]` - exactly like `unsigned char buf[16]` in C. It lives on the stack. It's a `Copy` type (assignment copies the bytes, no move needed). The size is baked into the type - `[u8; 16]` and `[u8; 32]` are different types.

```rust
let mut buf = [0u8; 16];  // 16 zero bytes, on the stack
let n = reader.read(&mut buf)?;
// buf[0..n] contains the bytes just read
// buf[n..16] contains zeros (or whatever was there before from the last read)
```

### Slices: Fat Pointers

A slice `&[u8]` is a pointer-plus-length pair. In C, when you pass a buffer to a function, you pass both the pointer and the length separately: `void process(unsigned char *buf, size_t len)`. In Rust, the slice bundles them: `fn process(buf: &[u8])` - `buf.len()` is right there.

You get a slice from an array with range syntax:

```rust
let data: &[u8] = &buf[0..n];  // first n bytes of buf
```

This doesn't copy. `data` is a reference into `buf`. It's valid as long as `buf` is alive. If you try to use `data` after `buf` goes out of scope, the compiler rejects it.

**Why does this matter for hexview?** After each `read()`, you have a 16-byte buffer but only `n` valid bytes. You construct a slice `&buf[0..n]` and work with that. If you accidentally work with `&buf` (the full 16 bytes), you're processing garbage on the last line of output.

### How It Breaks

**Index out of bounds panics, not undefined behavior.** `buf[20]` on a 16-byte array panics at runtime with a clear message. In C, `buf[20]` is undefined behavior that silently corrupts memory. Rust panics are better - they're loud. But use `buf.get(i)` (which returns `Option<&u8>`) when you're not sure the index is valid.

**Working with the wrong slice.** This is the subtle one:

```rust
let n = reader.read(&mut buf)?;
process_line(&buf);        // WRONG: passes all 16 bytes, even if only 5 were read
process_line(&buf[0..n]); // RIGHT: passes only the bytes that were read
```

No compiler error on the first line. But your output will have garbage bytes at the end of the last line.

---

## Iterators: Processing Data Without Index Arithmetic

### Why iterators exist

In C, processing a byte array in 16-byte chunks looks like:

```c
for (size_t i = 0; i < len; i += 16) {
    size_t chunk_size = (i + 16 <= len) ? 16 : len - i;
    process(&buf[i], chunk_size);
}
```

The arithmetic `(i + 16 <= len) ? 16 : len - i` handles the last chunk being shorter. It's correct but tedious and error-prone.

Rust's iterators express the same thing without the manual arithmetic:

```rust
for chunk in bytes.chunks(16) {
    process(chunk);  // chunk is &[u8], automatically the right size
}
```

`chunks(16)` handles the last-chunk-shorter-than-16 case for you. The last chunk is whatever's left.

### `.chunks(n)` - Split a Slice Into Fixed-Size Pieces

```rust
let bytes: &[u8] = &[0x7f, 0x45, 0x4c, 0x46, /* ... */ ];
for chunk in bytes.chunks(16) {
    // chunk is &[u8] - exactly 16 bytes, except the last which may be shorter
    println!("{} bytes in this chunk", chunk.len());
}
```

`chunks(16)` produces an iterator. Calling `.chunks(16)` alone doesn't process anything - it creates a description of an iteration. The processing happens when you loop.

### `.enumerate()` - Getting the Index Too

For a hex viewer, you need the byte offset of each line. `.enumerate()` wraps an iterator so each item becomes `(index, item)`:

```rust
for (i, chunk) in bytes.chunks(16).enumerate() {
    let offset: u64 = (i * 16) as u64;
    // offset is the byte position of the first byte in this chunk
    println!("{:08x}: ({}  bytes)", offset, chunk.len());
}
```

The index `i` counts chunks (0, 1, 2, ...). Multiplying by 16 gives the byte offset.

### Iterator Adapters Are Lazy

Calling `.chunks(16)` returns an iterator object. No data is processed yet. Calling `.chunks(16).enumerate()` returns another iterator object. Still nothing processed. The processing happens when you loop with `for`, or call `.collect()`, or call `.for_each()`.

This matters when you're chaining multiple operations: `bytes.chunks(16).enumerate().filter(...).map(...)` builds a pipeline description. The bytes are touched once, when the final `for` loop drives it. There's no intermediate allocation.

### How It Breaks

**Doing nothing by accident.** This compiles and runs silently, producing no output:

```rust
bytes.chunks(16).enumerate();  // creates an iterator, immediately drops it
```

You must consume the iterator - with `for`, `.for_each()`, `.collect()`, or similar.

**Assuming all chunks are exactly 16 bytes.** The last chunk of a file is usually shorter. `chunk.len()` may be 1 through 16. If your formatting code assumes 16 bytes (to align the ASCII column), you need to explicitly pad the short last line.

**Collecting when iteration is enough.** `.collect::<Vec<_>>()` allocates a Vec holding all the chunks at once. For a hex viewer, you don't need all chunks in memory simultaneously - just process them one at a time with `for`.

---

## Formatting: Producing Hex Output

### The output format

Every line of hexview looks like:

```
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
```

Three parts: offset (8 hex digits), hex bytes (grouped in pairs), ASCII (16 characters, `.` for non-printable).

### Formatting numbers as hex

Rust's `format!` macro uses the same printf-style format specifiers you know from C:

| C | Rust |
|---|------|
| `printf("%08lx", offset)` | `format!("{:08x}", offset)` |
| `printf("%02x", byte)` | `format!("{:02x}", byte)` |
| `printf("%c", ch)` | `format!("{}", ch as char)` |

- `:08x` - lowercase hex, minimum 8 characters, zero-padded
- `:02x` - lowercase hex, minimum 2 characters, zero-padded
- `:X` - uppercase hex (use lowercase for hex dumps by convention)

### Building a line

You can build the output for one line with string formatting. The approach that reads cleanly: build the hex portion and ASCII portion separately as `String` values, then combine them.

For the hex portion, join each byte's two-digit hex representation with a space between each pair of bytes.

For the ASCII portion, for each byte check if it's printable (`byte >= 0x20 && byte < 0x7f`). If yes, the character itself. If no, `.`.

For the last line (shorter than 16 bytes), the hex portion needs right-padding with spaces so the ASCII column stays aligned. Each missing byte takes 3 characters of space (two hex digits + one space).

### `write!` for building strings

When building a string incrementally, `write!` (from `std::fmt::Write`) works like `fprintf` into a string buffer:

```rust
use std::fmt::Write;

let mut hex_part = String::new();
for &byte in chunk {
    write!(hex_part, "{:02x} ", byte).unwrap();
}
```

This is different from `std::io::Write` (for writing to files or stdout). The `write!` macro works with both, but which trait you need depends on what you're writing to. For building `String`s in memory, `use std::fmt::Write`.

### How It Breaks

**`write!` not available without the right `use`.** `write!(some_string, ...)` fails with "method not found" unless you have `use std::fmt::Write` in scope. The error message points at the `write!` call but the fix is the use statement.

**Not padding the last line.** If the last line has only 5 bytes, the hex portion is shorter. The ASCII column shifts left. You need to add `(16 - n) * 3` spaces of padding after the hex bytes to keep alignment. Forgetting this makes the last line look wrong.

**Mixing `std::fmt::Write` and `std::io::Write`.** Both traits have a `write_fmt` method that `write!` uses internally. If you bring both into scope with `use std::fmt::Write; use std::io::Write;`, the compiler complains about ambiguity. Prefer bringing only the one you need at a given point.

---

## How It All Comes Together in hexview

The hexview program doesn't do anything magic. It's these four pieces in sequence:

Open the file with `File::open`, which returns a `Result` - your first `?` of the program. Wrap it in `BufReader` for efficient reads. If `--offset` was provided, seek to that position.

Enter a read loop. Each iteration calls `reader.read(&mut buf)`. The `usize` returned tells you how many bytes landed in `buf`. If it's 0, you're at EOF - break. If `--length` was provided, stop when you've processed that many total bytes.

For each chunk of up to 16 bytes, format one line: the offset (bytes processed so far, formatted as 8-digit hex), the hex bytes (each as two hex digits, space-separated in pairs), and the ASCII column (printable bytes as characters, others as `.`). Print the line.

The file closes when `BufReader` goes out of scope - `Drop` handles it. No `fclose`. No resource leak.

Every failure - file not found, seek error, read error, argument parse error - comes back as a `Result` that you propagate with `?` up to `main`. At `main`, you handle the final error: print a clear message to stderr and exit with code 1.

That's the whole program. The complexity is in getting the formatting exactly right and handling edge cases (offset beyond file, empty file, last line shorter than 16 bytes). The Rust parts - ownership, Result, slices, iterators - are the scaffolding that holds it together safely.
