# Doc 07 — Build Scripts and Compile-Time Checks

🟢 This is a superpower most Rust developers underuse — moving work from runtime to compile time.

The music API has a problem every deployed service has: how do you know which version is running on a given server? How do you embed the git commit hash, build timestamp, or target architecture into the binary — without making the code brittle?

The answer is `build.rs`. It's also how you compile C libraries, run code generation, and probe the system for libraries you depend on. This doc covers all five patterns, including how to connect them to the music API.

---

## How `build.rs` Fits Into the Build Pipeline

Cargo compiles and runs `build.rs` **before** compiling your crate:

```
1. Resolve dependencies
2. Download crates
3. Compile build.rs  ← runs on the HOST machine
4. Execute build.rs  ← its stdout becomes Cargo instructions
5. Compile your crate  (using the instructions from step 4)
6. Link
```

Key facts that matter for real usage:
- `build.rs` runs on the **host** machine, not the target. During cross-compilation (section 8), the build script runs on your laptop even if you're targeting ARM embedded.
- It communicates by printing special instructions to stdout — `cargo::rustc-env=KEY=VALUE`, `cargo::rerun-if-changed=PATH`, etc.
- Without any `rerun-if-changed` instruction, it re-runs on **every change to any file** in your crate. That's slow. Always emit at least one.

```rust
// build.rs — the minimal structure
fn main() {
    // This is the most important line in any build script.
    // Without it, Cargo re-runs build.rs whenever ANY file in the package changes.
    println!("cargo::rerun-if-changed=build.rs");
    
    // Everything else goes here.
}
```

---

## The Cargo Instruction Protocol

Your build script talks to Cargo through stdout:

| Instruction | Purpose |
|-------------|---------|
| `cargo::rerun-if-changed=PATH` | Only re-run when PATH changes (file or directory) |
| `cargo::rerun-if-env-changed=VAR` | Only re-run when env variable VAR changes |
| `cargo::rustc-env=KEY=VALUE` | Set env var available via `env!("KEY")` in your crate |
| `cargo::rustc-cfg=KEY` | Enable `#[cfg(KEY)]` conditional compilation |
| `cargo::rustc-cfg=KEY="VALUE"` | Enable `#[cfg(KEY = "VALUE")]` |
| `cargo::rustc-link-lib=NAME` | Link against a native library |
| `cargo::rustc-link-search=PATH` | Add a directory to the linker search path |
| `cargo::warning=MESSAGE` | Print a build warning |

---

## Pattern 1: Embed Build Metadata (Version Stamps)

The most common use case: baking build identity into the binary for traceability:

```rust
// build.rs
use std::process::Command;

fn main() {
    // Only re-run when the git state changes
    println!("cargo::rerun-if-changed=.git/HEAD");
    println!("cargo::rerun-if-changed=.git/refs");
    println!("cargo::rerun-if-changed=build.rs");

    // Git commit hash
    let git_hash = Command::new("git")
        .args(["rev-parse", "--short=10", "HEAD"])
        .output()
        .map(|o| String::from_utf8_lossy(&o.stdout).trim().to_string())
        .unwrap_or_else(|_| "unknown".to_string());
    println!("cargo::rustc-env=APP_GIT_HASH={git_hash}");

    // Build profile: "debug" or "release"
    let profile = std::env::var("PROFILE").unwrap_or_else(|_| "unknown".to_string());
    println!("cargo::rustc-env=APP_BUILD_PROFILE={profile}");

    // Target triple (what architecture you're compiling for)
    let target = std::env::var("TARGET").unwrap_or_else(|_| "unknown".to_string());
    println!("cargo::rustc-env=APP_BUILD_TARGET={target}");

    // Timestamp — use SOURCE_DATE_EPOCH if set for reproducible builds
    let timestamp = std::env::var("SOURCE_DATE_EPOCH").unwrap_or_else(|_| {
        std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .map(|d| d.as_secs().to_string())
            .unwrap_or_else(|_| "0".to_string())
    });
    println!("cargo::rustc-env=APP_BUILD_EPOCH={timestamp}");
}
```

Consuming the metadata in the music API:

```rust
// src/version.rs

pub struct BuildInfo {
    pub version: &'static str,
    pub git_hash: &'static str,
    pub profile: &'static str,
    pub target: &'static str,
    pub build_epoch: &'static str,
}

pub const BUILD_INFO: BuildInfo = BuildInfo {
    version:     env!("CARGO_PKG_VERSION"),
    git_hash:    env!("APP_GIT_HASH"),
    profile:     env!("APP_BUILD_PROFILE"),
    target:      env!("APP_BUILD_TARGET"),
    build_epoch: env!("APP_BUILD_EPOCH"),
};

impl std::fmt::Display for BuildInfo {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{} v{} (git:{} target:{} profile:{})",
            env!("CARGO_PKG_NAME"),
            self.version,
            self.git_hash,
            self.target,
            self.profile,
        )
    }
}
```

Now add a health check endpoint to the music API:

```rust
// src/routes/health.rs
use axum::Json;
use serde::Serialize;
use crate::version::BUILD_INFO;

#[derive(Serialize)]
pub struct HealthResponse {
    pub status: &'static str,
    pub version: &'static str,
    pub git_hash: &'static str,
    pub build_target: &'static str,
}

pub async fn health_check() -> Json<HealthResponse> {
    Json(HealthResponse {
        status: "ok",
        version: BUILD_INFO.version,
        git_hash: BUILD_INFO.git_hash,
        build_target: BUILD_INFO.target,
    })
}
```

**Why this matters**: When a deployment has a bug, you can hit `GET /health` and immediately know which commit is running. No guessing, no SSH-ing into instances and running `--version`. The answer is in the response.

---

## Pattern 2: Compiling C Code with `cc`

When your service wraps a C library — common when integrating vendor SDKs, hardware drivers, or performance-critical code — the `cc` crate handles compilation inside `build.rs`:

```toml
# Cargo.toml
[build-dependencies]
cc = "1.0"
```

```rust
// build.rs — compiling a C helper for the music API's audio fingerprinting
fn main() {
    println!("cargo::rerun-if-changed=csrc/");

    cc::Build::new()
        .file("csrc/chromaprint_wrapper.c")
        .include("csrc/include")
        .flag("-Wall")
        .flag("-O2")
        .compile("chromaprint_wrapper");
    // This automatically emits:
    // cargo::rustc-link-lib=static=chromaprint_wrapper
    // cargo::rustc-link-search=native=[output_dir]
}
```

```rust
// src/fingerprint.rs — FFI bindings to the C code
extern "C" {
    fn chromaprint_get_fingerprint(
        audio_data: *const f32,
        num_samples: usize,
        sample_rate: u32,
        out_fingerprint: *mut u8,
        out_len: *mut usize,
    ) -> i32;
}

pub fn compute_fingerprint(samples: &[f32], sample_rate: u32) -> Option<Vec<u8>> {
    let mut output = vec![0u8; 4096];
    let mut output_len: usize = output.len();

    let rc = unsafe {
        chromaprint_get_fingerprint(
            samples.as_ptr(),
            samples.len(),
            sample_rate,
            output.as_mut_ptr(),
            &mut output_len,
        )
    };

    if rc != 0 {
        return None;
    }
    output.truncate(output_len);
    Some(output)
}
```

The `cc` crate respects the cross-compilation toolchain — if `$CC` is set to a cross-compiler, it uses that. Raw `Command::new("gcc")` doesn't. Always use `cc` for C compilation in build scripts.

---

## Pattern 3: Code Generation from Schema Files

Build scripts run code generation — turning `.proto`, `.capnp`, or `.fbs` schema files into Rust source at compile time. For the music API, this could be a Protocol Buffers schema for a gRPC API:

```toml
# Cargo.toml
[build-dependencies]
prost-build = "0.13"
tonic-build = "0.12"
```

```rust
// build.rs
fn main() {
    println!("cargo::rerun-if-changed=proto/");

    tonic_build::compile_protos("proto/music_service.proto")
        .expect("Failed to compile protobuf definitions");
}
```

```rust
// src/lib.rs — include the generated types
pub mod music_service {
    include!(concat!(env!("OUT_DIR"), "/music.music_service.rs"));
}
```

`OUT_DIR` is a Cargo-provided path where build scripts should write generated files. Each crate gets its own `OUT_DIR` under `target/`. Never write to `src/` — Cargo doesn't expect source to change during a build.

---

## Pattern 4: Probing System Libraries with `pkg-config`

For system libraries that provide `.pc` files (OpenSSL, libpq, ALSA), `pkg-config` probes the system and emits the right link instructions:

```toml
[build-dependencies]
pkg-config = "0.3"
```

```rust
// build.rs — probing for optional system libraries
fn main() {
    println!("cargo::rerun-if-changed=build.rs");

    // Probe for ALSA (Linux audio) — needed for live audio input in the music API
    if pkg_config::probe_library("alsa").is_ok() {
        println!("cargo::rustc-cfg=has_alsa");
    }

    // Probe for libpq (PostgreSQL) — optional alternative to SQLite
    if pkg_config::probe_library("libpq").is_ok() {
        println!("cargo::rustc-cfg=has_postgresql");
    }
}
```

```rust
// src/audio_input.rs — conditional compilation based on detected system libraries
#[cfg(has_alsa)]
pub mod live_input {
    extern "C" {
        fn alsa_open_capture(device: *const std::ffi::c_char) -> *mut std::ffi::c_void;
        // ...
    }
    
    pub fn start_recording(device: &str) -> Result<RecordingHandle, AudioError> {
        // ALSA code here
        todo!()
    }
}

#[cfg(not(has_alsa))]
pub mod live_input {
    pub fn start_recording(_device: &str) -> Result<RecordingHandle, AudioError> {
        Err(AudioError::NotSupported("ALSA not available on this platform"))
    }
}
```

---

## Pattern 5: Feature Detection and Compile-Time Cfg Flags

Build scripts can probe the compilation environment and set `#[cfg]` flags for conditional code paths:

```rust
// build.rs — target-aware build configuration
fn main() {
    println!("cargo::rerun-if-changed=build.rs");

    let target = std::env::var("TARGET").unwrap_or_default();
    let target_os = std::env::var("CARGO_CFG_TARGET_OS").unwrap_or_default();

    // Detect target architecture for optimized code paths
    if target.starts_with("x86_64") {
        println!("cargo::rustc-cfg=has_x86_64_simd");
    }
    if target.starts_with("aarch64") {
        println!("cargo::rustc-cfg=has_arm_neon");
    }

    // Platform-specific behavior
    if target_os == "linux" {
        println!("cargo::rustc-cfg=platform_linux");
    } else if target_os == "macos" {
        println!("cargo::rustc-cfg=platform_macos");
    } else if target_os == "windows" {
        println!("cargo::rustc-cfg=platform_windows");
    }
}
```

```rust
// src/audio_device.rs — different implementations per platform
#[cfg(platform_linux)]
pub fn default_audio_device() -> &'static str {
    "hw:0,0"  // ALSA default
}

#[cfg(platform_macos)]
pub fn default_audio_device() -> &'static str {
    "default"  // CoreAudio default
}

#[cfg(platform_windows)]
pub fn default_audio_device() -> &'static str {
    "default"  // WASAPI default
}
```

---

## Compile-Time Constants with `env!`

Free Cargo-provided variables you always have, no `build.rs` needed:

```rust
// Available everywhere, no build.rs needed:
let version = env!("CARGO_PKG_VERSION");      // "0.1.0"
let name    = env!("CARGO_PKG_NAME");         // "music-api"
let authors = env!("CARGO_PKG_AUTHORS");      // from Cargo.toml
let dir     = env!("CARGO_MANIFEST_DIR");     // absolute path to Cargo.toml

// With build.rs:
let hash    = env!("APP_GIT_HASH");           // "a3f2c1d4b5"
let profile = env!("APP_BUILD_PROFILE");      // "release"
```

The `env!` macro resolves these at compile time — the resulting binary contains the string literal, not a runtime lookup.

---

## Reproducible Builds: The Tension

Embedding timestamps and git hashes makes binaries traceable. It also makes them non-reproducible — the same source code produces a different binary every time you build it, because the timestamp changes.

This matters for security-conscious deployments that want to verify binary integrity.

The resolution is `SOURCE_DATE_EPOCH`:

```rust
// build.rs — respects SOURCE_DATE_EPOCH for reproducibility
let timestamp = std::env::var("SOURCE_DATE_EPOCH")
    .unwrap_or_else(|_| {
        std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .map(|d| d.as_secs().to_string())
            .unwrap_or_else(|_| "0".to_string())
    });
println!("cargo::rustc-env=APP_BUILD_EPOCH={timestamp}");
```

```bash
# Development builds: live timestamp
cargo build

# CI release builds: reproducible timestamp from last commit
SOURCE_DATE_EPOCH=$(git log -1 --format=%ct) cargo build --release --locked
# Same commit + same locked deps + SOURCE_DATE_EPOCH = bit-identical binary
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| No `rerun-if-changed` | Build script runs on every change to any file — slow rebuilds | Always emit at least `cargo::rerun-if-changed=build.rs` |
| Network calls in build.rs | Offline builds fail; non-reproducible | Vendor files or separate fetch step |
| Writing to `src/` | Cargo doesn't expect source to change during builds | Write to `OUT_DIR` and use `include!()` |
| Using `Command::new("gcc")` for C compilation | Breaks cross-compilation | Use the `cc` crate |
| Build-time hardware detection for optional peripherals | Binary baked to build machine's hardware | Detect at runtime instead |
| `.unwrap()` without context | Opaque "build script failed" error | Use `.expect("descriptive message")` |

---

## How It Breaks

**Forgetting that `env!` is checked at compile time.**
If `APP_GIT_HASH` isn't set (because `build.rs` didn't run or failed silently), `env!("APP_GIT_HASH")` is a **compile error**, not a runtime panic. This is actually good — it forces you to fix the build script rather than silently shipping a binary with no git hash. But it's surprising the first time.

**Build script runs on every `cargo check`.**
`cargo check` also runs build scripts (it needs the cfg flags to type-check conditional code). A slow build script (one that calls git, runs pkg-config multiple times, etc.) makes `cargo check` slow. Keep build scripts fast and always use `rerun-if-changed`.

**`OUT_DIR` not in the path.**
`include!(concat!(env!("OUT_DIR"), "/generated.rs"))` only works if the file is actually written to `OUT_DIR` before the include. If your code generation step fails silently, you'll get a cryptic "file not found" error during compilation. Always check the output of your code generation commands with `.expect("codegen failed")`.

**The `cc` crate doesn't find the compiler.**
On Windows or unusual Linux configurations, `cc` might not find a C compiler. The error is "No C compiler found" and comes from `cc`, not from Cargo. Fix: install `build-essential` on Linux, or the Visual C++ Build Tools on Windows.
