# Doc 09 - Cross-Compilation and Conditional Builds

🟡 A storage engine that only runs on the machine where it was compiled is limited. Cross-compilation lets you build on a fast developer workstation and deploy to a resource-constrained server, an embedded ARM board, or a customer's on-premise Linux distribution.

This doc covers the Rust cross-compilation toolchain, conditional compilation with `cfg`, and how to make ironkv build correctly on multiple platforms without maintaining separate codebases.

---

## How Rust Targets Work

A Rust target is a triple: `architecture-vendor-os-environment`. Common targets:

| Target | Description |
|--------|-------------|
| `x86_64-unknown-linux-gnu` | 64-bit Linux with glibc (default) |
| `x86_64-unknown-linux-musl` | 64-bit Linux with musl (statically linked) |
| `aarch64-unknown-linux-gnu` | ARM64 Linux (Raspberry Pi 4, Apple M1) |
| `x86_64-pc-windows-msvc` | 64-bit Windows with MSVC |
| `wasm32-unknown-unknown` | WebAssembly (no OS) |
| `thumbv7m-none-eabi` | ARM Cortex-M (embedded, no OS) |

When you run `cargo build`, it builds for the host target. `--target` builds for a different one.

---

## Setting Up Cross-Compilation

Two approaches: native cross-compilation (for well-supported targets) and the `cross` tool (for everything else).

### Native cross-compilation to musl

A musl build produces a fully statically linked binary - no glibc dependency. This makes the binary portable across all Linux distributions.

```bash
# Add the musl target
rustup target add x86_64-unknown-linux-musl

# On Ubuntu, install the musl toolchain
sudo apt install musl-tools

# Build
cargo build --release --target x86_64-unknown-linux-musl

# Verify the binary is fully static
ldd target/x86_64-unknown-linux-musl/release/ironkv
# → "not a dynamic executable" (statically linked)

# Binary size: typically smaller than glibc equivalent
ls -lh target/x86_64-unknown-linux-musl/release/ironkv
```

For ironkv, a musl build is the recommended distribution format: the binary runs on any Linux host regardless of glibc version. This is especially useful if you're shipping ironkv to customer machines with diverse Linux environments.

### `cross`: Docker-based cross-compilation

For targets that need a different C toolchain (e.g., `aarch64-unknown-linux-gnu` on a Linux host), use `cross`:

```bash
cargo install cross

# Build for ARM64 Linux - downloads a Docker image with the right toolchain
cross build --release --target aarch64-unknown-linux-gnu

# Test on the target architecture (runs in QEMU via Docker)
cross test --target aarch64-unknown-linux-gnu
```

`cross` requires Docker. It's slower than native compilation (Docker overhead) but eliminates the need to install target-specific compilers manually.

---

## Conditional Compilation with `cfg`

`cfg` lets you write platform-specific code that only compiles on the appropriate target:

```rust
// Platform-specific imports
#[cfg(target_os = "linux")]
use std::os::unix::fs::OpenOptionsExt;

#[cfg(target_os = "windows")]
use std::os::windows::fs::OpenOptionsExt;

// Platform-specific implementation
pub fn set_file_hint(file: &std::fs::File) -> std::io::Result<()> {
    #[cfg(target_os = "linux")]
    {
        // On Linux, use POSIX_FADV_SEQUENTIAL for WAL writes
        use nix::fcntl::{posix_fadvise, PosixFadviseAdvice};
        posix_fadvise(
            file,
            0, 0,
            PosixFadviseAdvice::POSIX_FADV_SEQUENTIAL,
        ).ok();
    }

    #[cfg(not(target_os = "linux"))]
    {
        // No-op on other platforms
        let _ = file;
    }

    Ok(())
}
```

### Compile-time cfg constants

```rust
// Pointer size - relevant for index offsets
const POINTER_BITS: u32 = usize::BITS;

// Architecture check for SIMD optimization
#[cfg(target_arch = "x86_64")]
pub fn accelerated_checksum(data: &[u8]) -> u32 {
    // Use SSE4.2 CRC32 instruction if available
    #[cfg(target_feature = "sse4.2")]
    {
        // SSE4.2 hardware CRC
        return sse42_crc32(data);
    }

    // Fallback: software CRC
    software_crc32(data)
}

#[cfg(not(target_arch = "x86_64"))]
pub fn accelerated_checksum(data: &[u8]) -> u32 {
    software_crc32(data)
}
```

### `cfg_if!` for complex conditionals

The `cfg-if` crate provides cleaner multi-platform branching:

```toml
[dependencies]
cfg-if = "1"
```

```rust
use cfg_if::cfg_if;

cfg_if! {
    if #[cfg(target_os = "linux")] {
        pub use linux::MmapFile;
    } else if #[cfg(target_os = "macos")] {
        pub use macos::MmapFile;
    } else if #[cfg(target_os = "windows")] {
        pub use windows::MmapFile;
    } else {
        compile_error!("ironkv does not support this operating system");
    }
}
```

`compile_error!` is a macro that fails compilation with a custom message - better than a cryptic "type not found" error when a platform isn't supported.

---

## Feature Flags for Optional Capabilities

Cargo feature flags are the idiomatic way to provide optional capabilities:

```toml
# ironkv-core/Cargo.toml
[features]
default = []

# Optional compression using zstd
compression = ["dep:zstd"]

# Optional encryption (AES-GCM)
encryption = ["dep:aes-gcm", "dep:rand"]

# CLI utilities
cli = ["dep:clap", "dep:human-panic"]

[dependencies]
zstd = { version = "0.13", optional = true }
aes-gcm = { version = "0.10", optional = true }
rand = { version = "0.8", optional = true }
clap = { version = "4", optional = true, features = ["derive"] }
human-panic = { version = "2", optional = true }
```

Feature-gated code:

```rust
// Only available when the "compression" feature is enabled
#[cfg(feature = "compression")]
pub mod compress {
    use zstd;

    pub fn compress(data: &[u8], level: i32) -> Vec<u8> {
        zstd::encode_all(data, level).expect("compression failed")
    }

    pub fn decompress(data: &[u8]) -> Vec<u8> {
        zstd::decode_all(data).expect("decompression failed")
    }
}

// In the storage engine - compress values if the feature is enabled
pub fn put(&mut self, key: &[u8], value: &[u8]) -> Result<(), Error> {
    #[cfg(feature = "compression")]
    let value_to_store = compress::compress(value, 3);
    
    #[cfg(not(feature = "compression"))]
    let value_to_store = value.to_vec();

    self.write_entry(key, &value_to_store)
}
```

---

## The `build.rs` for Platform Detection

When conditional code depends on things `cfg` can't detect directly (e.g., whether a library exists), use `build.rs`:

```rust
// build.rs
fn main() {
    // Emit platform constants
    if cfg!(target_os = "linux") {
        println!("cargo::rustc-cfg=has_fallocate");   // Linux fallocate() available
        println!("cargo::rustc-cfg=has_direct_io");   // O_DIRECT flag available
    }
    
    // Check for optional system libraries
    if pkg_config::probe_library("zstd").is_ok() {
        println!("cargo::rustc-cfg=system_zstd");     // Use system zstd
    }
    
    println!("cargo::rerun-if-changed=build.rs");
}
```

In Rust code:

```rust
#[cfg(has_fallocate)]
fn preallocate_file(file: &std::fs::File, size: u64) {
    use std::os::unix::io::AsRawFd;
    unsafe {
        libc::fallocate(file.as_raw_fd(), 0, 0, size as i64);
    }
}

#[cfg(not(has_fallocate))]
fn preallocate_file(file: &std::fs::File, size: u64) {
    // Fallback: seek to end and write a zero byte
    file.set_len(size).ok();
}
```

---

## Cross-Compilation in CI

Test all target platforms in CI - a build that works on Linux but silently fails on ARM is a production incident waiting to happen:

```yaml
# .github/workflows/ci.yml
cross:
  name: Cross-compile (${{ matrix.target }})
  strategy:
    matrix:
      include:
        - target: x86_64-unknown-linux-musl
          os: ubuntu-latest
        - target: aarch64-unknown-linux-gnu
          os: ubuntu-latest
          use_cross: true
        - target: x86_64-pc-windows-msvc
          os: windows-latest
  runs-on: ${{ matrix.os }}
  steps:
    - uses: actions/checkout@v4
    - uses: dtolnay/rust-toolchain@stable
      with:
        targets: ${{ matrix.target }}

    - name: Install cross (if needed)
      if: matrix.use_cross
      run: cargo install cross

    - name: Build
      run: |
        if [ "${{ matrix.use_cross }}" = "true" ]; then
          cross build --target ${{ matrix.target }}
        else
          cargo build --target ${{ matrix.target }}
        fi
```

---

## The `target_os`, `target_arch`, and `target_env` Keys

The full list of `cfg` keys available for conditional compilation:

```rust
// Operating system
#[cfg(target_os = "linux")]
#[cfg(target_os = "macos")]
#[cfg(target_os = "windows")]

// Architecture
#[cfg(target_arch = "x86_64")]
#[cfg(target_arch = "aarch64")]
#[cfg(target_arch = "wasm32")]

// C runtime (environment)
#[cfg(target_env = "musl")]   // musl libc
#[cfg(target_env = "gnu")]    // glibc
#[cfg(target_env = "msvc")]   // Windows MSVC runtime

// Pointer width
#[cfg(target_pointer_width = "64")]
#[cfg(target_pointer_width = "32")]

// Endianness
#[cfg(target_endian = "little")]
#[cfg(target_endian = "big")]

// Feature detection (at compile time)
#[cfg(target_feature = "avx2")]
#[cfg(target_feature = "sse4.2")]
#[cfg(target_feature = "neon")]  // ARM SIMD
```

For ironkv, the relevant platform distinctions are:
- `target_os = "linux"` vs others (for Linux-specific mmap flags, `fallocate`, `O_DIRECT`)
- `target_env = "musl"` (for static distribution builds)
- `target_arch = "x86_64"` (for potential SSE4.2 CRC acceleration)

---

## Platform-Specific mmap for ironkv

The most platform-specific part of ironkv is its memory-mapped file access. On Linux, `mmap` has flags that don't exist on macOS or Windows:

```rust
#[cfg(target_os = "linux")]
pub fn open_data_file(path: &std::path::Path) -> std::io::Result<MmapFile> {
    use std::os::unix::fs::OpenOptionsExt;
    
    let file = std::fs::OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .custom_flags(libc::O_DIRECT)  // O_DIRECT: bypass page cache for the WAL
        .open(path)?;
    
    MmapFile::new(file)
}

#[cfg(target_os = "macos")]
pub fn open_data_file(path: &std::path::Path) -> std::io::Result<MmapFile> {
    let file = std::fs::OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .open(path)?;
    
    // F_NOCACHE is the macOS equivalent of O_DIRECT
    unsafe {
        libc::fcntl(
            std::os::unix::io::AsRawFd::as_raw_fd(&file),
            libc::F_NOCACHE,
            1,
        );
    }
    
    MmapFile::new(file)
}

#[cfg(target_os = "windows")]
pub fn open_data_file(path: &std::path::Path) -> std::io::Result<MmapFile> {
    // Windows doesn't have a direct equivalent - use FILE_FLAG_NO_BUFFERING
    // This requires aligned I/O sizes (sector size multiples)
    use std::os::windows::fs::OpenOptionsExt;
    
    let file = std::fs::OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .custom_flags(winapi::um::winbase::FILE_FLAG_NO_BUFFERING)
        .open(path)?;
    
    MmapFile::new(file)
}
```

The platform abstraction is a single function `open_data_file` with three implementations. Callers see one function. The `cfg` conditions pick the right one at compile time.
