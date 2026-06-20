# Project: hexview - Binary File Hex Dump Viewer

**Project name:** `hexview`  
**Difficulty:** 🟢→🟡 - Milestones 1-3 are straightforward. Milestones 5-6 require real thinking.

---

## Engineering Approach: Requirements-First

Before writing a line of code, answer these four questions. Write them down.

**What are the inputs?**
- A file path (required, positional argument)
- `--offset` / `-o`: byte offset to start reading from (optional, default 0)
- `--length` / `-n`: maximum bytes to display (optional, default: read until EOF)

**What are the outputs?**
- A formatted hex dump to stdout: one line per 16 bytes, in this exact format:
  ```
  00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
  ```
- On error: a human-readable message to stderr, exit code 1
- On success: exit code 0

**What are the failure cases?**
- File does not exist → stderr message, exit 1
- File exists but is not readable (permission denied) → stderr message, exit 1
- `--offset` or `--length` is not a valid non-negative integer → stderr message, exit 1
- `--offset` is beyond the end of the file → print nothing, exit 0 (you asked for 0 bytes and got 0)
- File is empty → print nothing, exit 0

**What does "correct" look like?**
Your output should match `xxd` byte-for-byte for any file. That's your test. Run `xxd some_file` and `hexview some_file` and diff them. If they match, you're done.

---

## What You're Building

`hexview` is the Rust equivalent of `xxd`, `hexdump -C`, or the hex view you've stared at in a debugger. It reads a binary file and prints each byte as two hex digits, 16 per line, with the raw ASCII on the right (non-printable bytes show as `.`).

This is a real tool. The output format:

```
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0300 3e00 0100 0000 4010 4000 0000 0000  ..>.....@.@.....
00000020: 3800 0000 0000 0000 b80c 0200 0000 0000  8...............
```

- **Column 1:** byte offset in the file, printed as 8 lowercase hex digits, followed by a colon
- **Column 2:** the 16 bytes in hex, printed as two-digit pairs, grouped as pairs-of-two with a space between groups
- **Column 3:** the same 16 bytes as ASCII - printable characters shown directly, all others shown as `.`

The last line may have fewer than 16 bytes. The hex column must be padded with spaces so the ASCII column aligns on every line.

---

## How This Project Uses Rust - Connection to the Docs

This is a first-principles explanation of how each doc topic shows up in the actual program. No code - just the concepts.

**Doc 01 (Why Rust):** In C, if your loop assumes 16 bytes per chunk but the last chunk has 5, you process 11 bytes of garbage. Rust's slice system prevents this - `&buf[0..n]` is guaranteed to be exactly `n` bytes of valid data. The compiler won't let you access beyond it.

**Doc 03 (Types):** Byte values are `u8` (exactly 0–255, never negative). File offsets are `u64` (files can exceed 4GB, so not `u32`). Array indices and sizes are `usize` - Rust's machine-word-sized integer for all lengths and indices. Mixing `u64` and `usize` requires explicit casts (`as`), which is intentional - the compiler forces you to think about the conversion.

**Doc 04 (Control flow):** The read loop uses `loop { ... break }` rather than a `while` because the exit condition (EOF) comes from inside the loop body, not before it. Printability checking is a range check on `u8`. Formatting the line uses iteration over the chunk bytes.

**Doc 05 (Ownership):** `File::open()` gives you an owned `File` value. You move it into `BufReader::new()` - after that, the variable `file` is gone. The `BufReader` owns everything for the rest of the function. When `BufReader` goes out of scope, `Drop` closes the file descriptor. No `fclose`. This is Rust's RAII in action.

**Doc 06 (Error handling):** Every I/O call returns `Result`. You use `?` to propagate errors from inner functions to `main`. In `main`, you handle the final `Err` by printing to `eprintln!` and calling `std::process::exit(1)`. This exact pattern - `run() -> Result<(), E>` called from `main` - is what you'll use in every section.

**Doc 07 (File I/O, slices, iterators):** The core of the project. `BufReader` for efficient reads. `read(&mut buf)` returning the byte count. `&buf[0..n]` for slicing to valid data. `chunks(16).enumerate()` for the line loop. `format!("{:02x}", byte)` for hex digits.

---

## Rust Patterns Applied Here

These patterns appear in later sections too. Learn them here first.

**`run()` → `main()` separation.** Never put logic directly in `main` when it can fail. Write `fn run() -> Result<(), AppError>` that contains the real work. In `main`, call `run()` and convert `Err` to a stderr message + exit code. This keeps `main` clean and makes error handling centralized.

**Newtype for semantics.** A `u64` for a byte offset and a `u64` for a byte length are the same type but mean different things. As you build larger programs, consider wrapper types like `struct ByteOffset(u64)` and `struct ByteCount(u64)` - the compiler will stop you from passing one where the other is expected. You don't have to do this in Milestone 1, but recognize where it would help.

**Early return on error, not nested ifs.** Use `?` to exit a function early on failure rather than nesting your success path inside an `if let Ok(...)` chain. The success path should read left-to-right without nesting.

**`eprintln!` for errors, `println!` for output.** Errors go to stderr. Program output goes to stdout. This allows `hexview file.bin > out.txt` to capture only the hex dump, not error messages.

---

## Scope: What You Will NOT Build

- No hex editing or writing - read-only
- No TUI or interactive mode - plain stdout
- No color output
- No reading from stdin - file path only
- No multiple files
- No reverse hex dump (hex back to binary)

If you find yourself thinking about any of these, you're scope-creeping. Ship the thing that works.

---

## Milestones

Work through these in order. Each milestone produces a working program before the next one adds complexity.

### Milestone 1: Open and Count

`cargo new hexview`. Write a program that takes a single command-line argument (a file path), opens the file, reads all bytes into a `Vec<u8>`, and prints the total byte count.

```
$ hexview /etc/hostname
13 bytes
```

You will fight your first error type issue when `?` won't work in `main`. Fix it: change `main` to `fn main() -> Result<(), Box<dyn std::error::Error>>`. You'll understand why later - for now, just do it.

**Stuck? Key shapes:**

Getting args and opening the file:
```rust
use std::env;
use std::fs::File;
use std::io::{self, Read};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args: Vec<String> = env::args().collect();
    if args.len() < 2 {
        eprintln!("Usage: hexview <file>");
        std::process::exit(1);
    }
    let path = &args[1];
    // open the file, read bytes, call your format function, print
    // ...
    Ok(())
}
```

Reading all bytes:
```rust
let mut bytes = Vec::new();
let mut file = File::open(path)?;
file.read_to_end(&mut bytes)?;
```

Expected output when it works:
```
$ hexview /etc/hostname
13 bytes
```

### Milestone 2: First Line of Output

Print the first 16 bytes as a correctly formatted hex dump line. Don't worry about the last-line padding yet.

```
$ hexview /etc/hostname
00000000: 6465 7363 746f 702d 6c6e 780a            desktop-lnx.
```

Focus on getting the format exactly right: `{offset:08x}: {hex pairs}  {ascii}`. Two spaces between the hex and ASCII columns.

**The tricky part - hex grouping:**

The output groups bytes in pairs: `7f45 4c46` not `7f 45 4c 46`. That's 2 bytes per group, space between groups. Here's the grouping logic shape:

```rust
fn format_hex(chunk: &[u8]) -> String {
    chunk
        .chunks(2)              // group pairs
        .map(|pair| {
            pair.iter()
                .map(|b| format!("{:02x}", b))
                .collect::<String>()
        })
        .collect::<Vec<_>>()
        .join(" ")              // space between groups
}
```

The ASCII column: printable bytes show as the character, everything else as `.`. A byte is printable if it's in range `0x20..=0x7e`.

Expected output for the first line of `/etc/hostname` (value: "desktop-lnx\n"):
```
00000000: 6465 7363 746f 702d 6c6e 780a            desktop-lnx.
```
Note: that trailing gap before `desktop-lnx.` is the padding for a short line. Wait - for MILESTONE 2 only the first line, and if it happens to be a short file, you'll see the padding issue early. That's fine for now - fix it properly in milestone 3.

### Milestone 3: Full Dump

Loop and dump the entire file, 16 bytes per line. Handle the last line being shorter than 16 bytes - the hex portion needs right-padding of `(16 - chunk.len()) * 3` spaces so the ASCII column stays aligned.

Verify against `xxd`:
```bash
xxd some_file > expected.txt
hexview some_file > actual.txt
diff expected.txt actual.txt
```

**The padding formula:**

When the last chunk has fewer than 16 bytes, the hex column is shorter. You need to right-pad it so the ASCII column always starts at the same position.

Each missing byte would have occupied `3` characters in the hex column (`XX` + ` `). So pad with `(16 - chunk.len()) * 3` spaces.

```rust
// after building hex_part for a chunk:
let padding = " ".repeat((16 - chunk.len()) * 3);
// your output line: "{:08x}: {}{}  {}", offset, hex_part, padding, ascii_part
```

This is the kind of off-by-one detail that makes you stare at your output next to xxd wondering why the last line is off. Get this right in milestone 3 before adding features in 4-6.

### Milestone 4: Offset and Length Flags

Add `-o <N>` and `-n <N>` flags. Parse them manually from `std::env::args()`. Use `std::io::Seek` (`SeekFrom::Start(offset)`) to skip to the start offset. Track bytes processed and stop when you hit the length limit.

The offset column always shows the actual file offset - not "0" even if you started mid-file.

### Milestone 5: Proper CLI with clap

Add `clap = { version = "4", features = ["derive"] }` and use the derive macro. clap automatically handles `--help`, type validation (`--offset abc` gives a proper error), and argument naming. You delete your manual parsing loop and replace it with a struct.

**Clap derive shape:**

```rust
use clap::Parser;

#[derive(Parser, Debug)]
#[command(name = "hexview", about = "Binary file hex dump viewer")]
struct Args {
    /// File to display
    file: String,

    /// Start at byte offset
    #[arg(short = 'o', long, default_value_t = 0)]
    offset: u64,

    /// Maximum bytes to display
    #[arg(short = 'n', long)]
    length: Option<u64>,
}

fn main() {
    let args = Args::parse();
    // args.file, args.offset, args.length are ready to use
}
```

Add to `Cargo.toml`: `clap = { version = "4", features = ["derive"] }`

### Milestone 6: Correct Error Handling

Replace `Box<dyn Error>` with a typed error enum using `thiserror`. Create a `run() -> Result<(), HexviewError>` function. In `main`, call `run()` and match on the error variant to produce a specific user-facing message:

- `HexviewError::FileNotFound(path)` → `"error: cannot open '{path}': No such file or directory"`
- `HexviewError::PermissionDenied(path)` → `"error: cannot open '{path}': Permission denied"`
- `HexviewError::ReadError(e)` → `"error: read failed: {e}"`

Test all failure cases explicitly. Run `hexview nonexistent.txt` and verify you get exit code 1. Run `hexview /etc/shadow` (on Linux) and verify permission denied. Run `hexview /dev/null` and verify you get no output and exit code 0.

**thiserror shape:**

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum HexviewError {
    #[error("cannot open '{path}': {source}")]
    Open { path: String, source: std::io::Error },
    
    #[error("read error: {0}")]
    Read(#[from] std::io::Error),
}

fn run(args: &Args) -> Result<(), HexviewError> {
    let file = File::open(&args.file).map_err(|e| HexviewError::Open {
        path: args.file.clone(),
        source: e,
    })?;
    // ...
    Ok(())
}

fn main() {
    let args = Args::parse();
    if let Err(e) = run(&args) {
        eprintln!("error: {e}");
        std::process::exit(1);
    }
}
```

---

## Testing

The gold standard: byte-identical output to `xxd`.

```bash
xxd /bin/ls > expected.txt
hexview /bin/ls > actual.txt
diff expected.txt actual.txt
```

Test edge cases explicitly:
- Empty file: no output, exit 0
- Exactly 16 bytes: one complete line, no padding
- Exactly 17 bytes: one complete line + one 1-byte line (the padding matters here)
- Large binary (`/bin/ls` or any ELF binary): tests multi-line output and alignment
