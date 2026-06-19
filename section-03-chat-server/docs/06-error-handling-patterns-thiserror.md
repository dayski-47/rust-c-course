# Doc 06 — Error Handling Patterns: thiserror and Friends

You've been using `Result<T, E>` and `.unwrap()` since Section 1. `unwrap()` is fine for learning, and it's even fine for prototyping. But a TCP server that's handling multiple clients can't just panic when a client sends bad data — that would crash the whole server.

This chapter teaches you how to define real error types, propagate them cleanly, and make decisions about what to crash on versus what to recover from.

---

## Quick Review: Result and ? 🟢

You know this already, but let's make it explicit before building on it.

```rust
fn read_number(s: &str) -> Result<i32, std::num::ParseIntError> {
    let n = s.trim().parse::<i32>()?;  // ? returns early with Err if parse fails
    Ok(n)
}

fn main() {
    match read_number("42") {
        Ok(n) => println!("Got {n}"),
        Err(e) => println!("Error: {e}"),
    }
}
```

The `?` operator desugars to:
```rust
let n = match s.trim().parse::<i32>() {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),  // converts e to the function's error type
};
```

The `From::from(e)` part is important. It means `?` will automatically convert the error type if a `From` implementation exists. This is the hook `thiserror` uses.

---

## Why Custom Error Types 🟡

In a real server you might get:
- `io::Error` from reading the socket
- `ParseIntError` from parsing a port number
- Your own error from invalid chat protocol
- A lock poison error if a thread panics

If each function returns a different error type, you can't use `?` to propagate across function boundaries — the types don't match. You need a single error type that can represent all of these.

Without a crate, you'd write this yourself:
```rust
use std::fmt;

#[derive(Debug)]
pub enum ChatError {
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
    InvalidUsername(String),
}

impl fmt::Display for ChatError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ChatError::Io(e) => write!(f, "I/O error: {e}"),
            ChatError::Parse(e) => write!(f, "Parse error: {e}"),
            ChatError::InvalidUsername(u) => write!(f, "Invalid username: {u}"),
        }
    }
}

impl std::error::Error for ChatError {}

impl From<std::io::Error> for ChatError {
    fn from(e: std::io::Error) -> Self {
        ChatError::Io(e)
    }
}

impl From<std::num::ParseIntError> for ChatError {
    fn from(e: std::num::ParseIntError) -> Self {
        ChatError::Parse(e)
    }
}
```

That's 30+ lines of boilerplate for a simple error type. `thiserror` generates all of it from a 10-line annotation.

---

## thiserror: The Library Error Crate 🟡

Add to `Cargo.toml`:
```toml
[dependencies]
thiserror = "1"
```

Now the same error type:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ChatError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),

    #[error("Invalid username: {0}")]
    InvalidUsername(String),

    #[error("User '{username}' not found")]
    UserNotFound { username: String },
}
```

That's it. `#[derive(Error, Debug)]` generates the `Display` and `std::error::Error` implementations. `#[error("...")]` is the format string for the `Display` impl. `#[from]` generates the `From` conversion.

Now `?` works anywhere you return `Result<_, ChatError>`:

```rust
fn read_port(s: &str) -> Result<u16, ChatError> {
    let port = s.trim().parse::<u16>()?;  // ParseIntError → ChatError::Parse automatically
    Ok(port)
}

fn write_to_client(stream: &mut TcpStream, msg: &str) -> Result<(), ChatError> {
    stream.write_all(msg.as_bytes())?;  // io::Error → ChatError::Io automatically
    Ok(())
}
```

---

## Interpolation in Error Messages 🟢

The `#[error("...")]` string supports field interpolation:

```rust
#[derive(Error, Debug)]
pub enum ChatError {
    // Tuple struct — access fields by index: {0}, {1}
    #[error("connection refused: {0}")]
    Refused(String),

    // Named fields — access by name
    #[error("user '{username}' not found in room '{room}'")]
    UserNotFound { username: String, room: String },

    // {0} with #[from] — show the source error
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    // #[error(transparent)] — delegate Display to the inner error completely
    // Use when you want to pass an error through without adding context
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
```

---

## anyhow: The Application Error Crate 🟡

`thiserror` is for libraries — when you want callers to be able to match on specific error variants.

`anyhow` is for application code — when you just want errors to propagate and display nicely without defining a type for every possible failure.

Add to `Cargo.toml`:
```toml
[dependencies]
anyhow = "1"
```

```rust
use anyhow::{Context, Result};

// anyhow::Result<T> is just Result<T, anyhow::Error>
// anyhow::Error can hold any error type
fn load_config(path: &str) -> Result<String> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from '{path}'"))?;

    if content.is_empty() {
        anyhow::bail!("config file is empty");  // bail! creates and returns an Err
    }

    Ok(content)
}

fn main() -> Result<()> {  // main() can return Result with anyhow
    let config = load_config("chat.toml")?;
    println!("Config loaded: {} bytes", config.len());
    Ok(())
}
```

`anyhow::Error` carries the full error chain — each `.context()` wraps the previous error. When you print an `anyhow::Error` with `{:#}` (alternate debug format), you see the full chain:

```
Error: failed to read config from 'chat.toml'

Caused by:
    No such file or directory (os error 2)
```

---

## When to Use Which

| Situation | Use |
|---|---|
| You're writing a library crate | `thiserror` — callers need to match on errors |
| You're writing a binary/application | `anyhow` — just make errors propagate cleanly |
| You need both | `thiserror` in library modules, `anyhow` in main.rs |
| Quick prototype | `Box<dyn std::error::Error>` or just `.unwrap()` |

For the chat server (a binary application), `anyhow` for `main` and the top-level server loop, `thiserror` for any module that defines protocol-specific errors that other modules need to match on.

---

## Error Propagation in a TCP Server 🟡

Here's how error handling flows in the chat server. The key insight: I/O errors on a single client connection should NOT crash the server. They should terminate that client's handler thread.

```rust
use std::net::TcpStream;
use std::io::{BufRead, BufReader, Write};
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ClientError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Client sent invalid username: '{0}'")]
    InvalidUsername(String),

    #[error("Client disconnected")]
    Disconnected,
}

fn negotiate_username(reader: &mut BufReader<TcpStream>, stream: &mut TcpStream)
    -> Result<String, ClientError>
{
    stream.write_all(b"Enter username: ")?;  // io::Error → ClientError::Io

    let mut username = String::new();
    let bytes_read = reader.read_line(&mut username)?;  // io::Error → ClientError::Io

    if bytes_read == 0 {
        return Err(ClientError::Disconnected);
    }

    let username = username.trim().to_string();
    if username.is_empty() || username.contains(' ') {
        return Err(ClientError::InvalidUsername(username));
    }

    Ok(username)
}

fn handle_client(stream: TcpStream) {
    let peer = stream.peer_addr().ok();
    match handle_client_inner(stream) {
        Ok(()) => println!("{peer:?} disconnected cleanly"),
        Err(ClientError::Disconnected) => println!("{peer:?} disconnected"),
        Err(e) => eprintln!("{peer:?} error: {e}"),
    }
}

fn handle_client_inner(mut stream: TcpStream) -> Result<(), ClientError> {
    let mut reader = BufReader::new(stream.try_clone()?);
    let username = negotiate_username(&mut reader, &mut stream)?;
    println!("{username} connected from {:?}", stream.peer_addr()?);
    // ... rest of the handler
    Ok(())
}
```

The structure: inner functions return `Result`, the outer `handle_client` function calls the inner one and logs errors. The thread doesn't panic — it just ends cleanly.

---

## Logging Errors: A Brief Word on tracing 🟢

For a chat server, you want to see what's happening: who connected, who said what, what errors occurred. The `tracing` crate is the idiomatic way to do structured logging in Rust:

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = "0.3"
```

```rust
use tracing::{info, warn, error};

fn main() {
    tracing_subscriber::fmt::init();  // log to stderr with timestamps

    info!("Server starting on port 8080");

    // In client handlers:
    info!(username = %username, "Client connected");
    warn!(username = %username, "Invalid message received");
    error!(username = %username, err = %e, "Connection error");
}
```

This outputs structured logs you can filter and search. For the project, plain `println!` with timestamps is fine. But know that `tracing` is what production Rust services use.

---

## Panic vs Return Error 🟡

The rule: use `Result` for *expected* failures. Use `panic!` for *programming errors* (invariant violations, bugs).

```rust
// EXPECTED failure — use Result
fn parse_message(input: &str) -> Result<ChatMessage, ChatError> {
    // Input might be malformed. That's expected. Return Err.
}

// PROGRAMMING ERROR — panic is correct
fn get_client_sender(clients: &ClientMap, username: &str) -> Sender<String> {
    clients.lock().unwrap()
        .get(username)
        .cloned()
        .expect("BUG: tried to get sender for non-existent client")
    // If this panics, it's a bug in our code, not a runtime condition
}
```

In the chat server, you'll use `Result` for all client-facing operations (I/O, parsing) and `.expect("descriptive message")` for internal invariant checks. The `.expect()` messages are your documentation of assumptions — when they fail, you know exactly which assumption was wrong.

---

## A Complete Error Type for the Chat Server

Here's a starting point for the project:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ChatError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Client disconnected")]
    Disconnected,

    #[error("Invalid username '{0}': usernames must be 1-20 characters, no spaces")]
    InvalidUsername(String),

    #[error("Username '{0}' is already taken")]
    UsernameTaken(String),

    #[error("Internal error: {0}")]
    Internal(String),
}

// Type alias to save typing
pub type Result<T> = std::result::Result<T, ChatError>;
```

With the `Result<T>` alias, your functions look clean:
```rust
fn handle_client(stream: TcpStream) -> Result<()> {
    // ...
    Ok(())
}
```

---

## Common Mistakes

**Using `unwrap()` in client handlers.** If `read_line().unwrap()` panics on a bad client, it panics the handler thread. Because threads in Rust are isolated, this won't crash the server — but it leaves the client registered in the client map forever, and you'll see a poisoned mutex if you're sharing state. Always use `?` in handler code.

**Defining an error type per function.** Start with one error enum per module. You can always split later. Over-engineering the error hierarchy early makes code unreadable.

**Forgetting to add `#[from]` and wondering why `?` doesn't compile.** If you get "the trait `From<io::Error>` is not implemented for `ChatError`", you either need `#[from]` on the variant or a manual `From` impl.

**Using `anyhow` in a library crate.** `anyhow::Error` is opaque — callers can't match on specific error variants. If you're building code others will call programmatically, use `thiserror` so they can handle errors discriminately.

**Printing errors with `{:?}` instead of `{}`**. The `{}` (Display) format gives the human-readable error message from `#[error("...")]`. The `{:?}` (Debug) format gives the internal representation, which is often noisy. Use `{}` for user-facing messages and `{:?}` or `{:#?}` when debugging.
