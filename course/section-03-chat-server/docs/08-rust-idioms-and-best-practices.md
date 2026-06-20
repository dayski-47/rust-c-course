# 07 — Rust Idioms and Best Practices

> **Project:** Chat Server — a multi-threaded TCP server where each connected client gets its own thread, and messages are broadcast to all other clients.

## In This Section

You are building a TCP chat server. Every client connection spawns a thread. Threads share the list of connected clients behind an `Arc<Mutex<...>>`. Two idioms from this document will hit you as soon as you try to share state across threads:

- **`Send + 'static` in closures** — when you pass a closure to `thread::spawn`, the compiler requires it to be `Send + 'static`. This is the most common confusing moment for students. The fix is always the same: clone your `Arc` before moving it into the closure.
- **RAII lock guards and explicit `drop`** — `Mutex::lock()` returns a guard that releases the lock when it is dropped. If you hold the guard across a blocking operation (like writing to a socket), you deadlock every other thread. Use `drop(guard)` to release the lock early and explicitly.

---

## 1. Naming Conventions

The Rust community follows these naming conventions. Your code looks wrong to any Rust developer if you don't:

| What | Convention | Example |
|------|-----------|---------|
| Variables, functions, methods | `snake_case` | `handle_client`, `broadcast_message` |
| Types, structs, enums, traits | `PascalCase` | `ClientId`, `ChatRoom`, `ServerError` |
| Constants, statics | `UPPER_SNAKE_CASE` | `MAX_CLIENTS`, `BUFFER_SIZE` |
| Modules | `snake_case` | `mod server`, `mod client` |
| Enum variants | `PascalCase` | `Ok`, `Err`, `Some`, `None`, `Disconnected` |

Do not use `camelCase` for variables (that is JavaScript). Do not use `ALL_CAPS` for types.

---

## 2. Prefer Owned Types in Structs, References in Function Signatures

A C developer's instinct: "use a pointer to avoid copying." In Rust, this leads to lifetime headaches.

**Rule: Structs should own their data.** Use `String`, not `&str`. Use `Vec<T>`, not `&[T]`.

**Rule: Functions should borrow.** Function parameters should be `&str` (not `String`), `&[T]` (not `Vec<T>`), `&T` (not `T`) unless you need to store or transfer ownership.

Why this works: `String` can be passed as `&str` automatically (deref coercion). `Vec<T>` can be passed as `&[T]` automatically. The reverse is not true — you cannot pass a `&str` where a `String` is needed without cloning.

In the chat server, each connected `Client` owns its username and sender handle, but the broadcast function borrows a slice of the client list:

```rust
struct Client {
    id: u64,
    username: String,                    // owned
    sender: mpsc::Sender<String>,        // owned
}

// Good: borrow a slice, iterate without taking ownership
fn broadcast(clients: &[Client], from_id: u64, message: &str) {
    for client in clients {
        if client.id != from_id {
            let _ = client.sender.send(message.to_owned());
        }
    }
}
```

---

## 3. The Builder Pattern for Complex Construction

When a struct has many optional fields, avoid a constructor with 7 parameters. Use a builder:

```rust
// Bad: fragile positional constructor
let server = ChatServer::new("0.0.0.0", 8080, 100, true, 30, None);

// Good: self-documenting, easy to add fields later
let server = ChatServer::builder()
    .bind_addr("0.0.0.0")
    .port(8080)
    .max_clients(100)
    .idle_timeout_secs(30)
    .build()?;
```

You will use this pattern with `reqwest` clients and Tokio runtimes in the async section.

---

## 4. Error Handling Hierarchy

**Do not use:** `unwrap()` or `expect()` in production paths — anything that can fail due to network errors, client disconnects, or OS limits.

**Early development:** `Box<dyn std::error::Error>` as the error type. Gets you started without designing an error type first.

**Applications (tools, services):** `anyhow::Error`. Context-aware, stackable. The `anyhow::Context` trait lets you attach messages.

**Libraries:** `thiserror` with a typed enum. Callers need to match on variants; `anyhow` would hide them.

The pattern:

```rust
fn run() -> anyhow::Result<()> {
    let listener = TcpListener::bind("0.0.0.0:8080")
        .context("failed to bind TCP listener on port 8080")?;

    for stream in listener.incoming() {
        let stream = stream.context("failed to accept connection")?;
        // ...
    }
    Ok(())
}

fn main() {
    if let Err(e) = run() {
        eprintln!("error: {e:#}");
        std::process::exit(1);
    }
}
```

---

## 5. Don't Match on Bool — Use the Right Abstraction

```rust
// Bad: checking is_some() then calling unwrap()
if client.username.is_some() {
    let name = client.username.unwrap();
    broadcast(&clients, client.id, &name);
}

// Good: if let destructures in one step
if let Some(ref name) = client.username {
    broadcast(&clients, client.id, name);
}

// Bad: manually re-wrapping an error
if result.is_err() {
    return Err(result.unwrap_err());
}

// Good: ? propagates automatically
let n = stream.read(&mut buf)?;
```

---

## 6. Closures as First-Class Values — and the `Send + 'static` Requirement

In C, function pointers do what closures do, but closures also capture their environment. When you pass a closure to `thread::spawn`, the compiler requires it to be `FnOnce + Send + 'static`.

This is the most common confusing moment for students. Here is what the error means and how to fix it:

```rust
// WRONG: you cannot move a reference into a thread — the reference might
// become invalid when the stack frame that owns the data is popped.
let clients = Arc::new(Mutex::new(Vec::<Client>::new()));

thread::spawn(|| {
    handle_client(&clients, stream);  // error: `clients` does not live long enough
});

// CORRECT: clone the Arc before spawning. The clone is owned by the closure,
// and Arc guarantees the data lives as long as any clone exists.
let clients_clone = Arc::clone(&clients);
thread::spawn(move || {
    handle_client(&clients_clone, stream);
});
```

The `'static` requirement does not mean the data must live forever. It means the closure cannot hold any *borrowed* references — only owned values (including `Arc`, which is owned). Cloning an `Arc` is cheap: it increments an atomic reference count, it does not copy the data.

Know the three closure traits:

- `Fn` — can be called multiple times, borrows captured variables
- `FnMut` — can be called multiple times, mutably borrows captured variables
- `FnOnce` — can only be called once, takes ownership of captured variables

---

## 7. RAII Lock Guards and Explicit `drop`

`Mutex::lock()` returns a `MutexGuard`. The lock is released when the guard is dropped. If you hold the guard across a blocking call — writing to a socket, sleeping, waiting on a channel — every other thread that tries to acquire the lock will block for that entire duration.

Release the lock early with `drop(guard)`:

```rust
// BAD: holding the lock while writing to a socket
// Every other thread blocks for the duration of the send
{
    let guard = clients.lock().unwrap();
    for client in guard.iter() {
        client.sender.send(message.clone()).unwrap();  // may block
    }
    // guard drops here — but we held it for the entire loop
}

// GOOD: collect what you need, release the lock, then do the blocking work
let senders: Vec<mpsc::Sender<String>> = {
    let guard = clients.lock().unwrap();
    guard.iter().map(|c| c.sender.clone()).collect()
    // guard drops here — lock is released before any network I/O
};

for sender in senders {
    let _ = sender.send(message.clone());
}
```

You can also call `drop(guard)` explicitly at any point inside a block — you do not have to restructure into an inner scope. The explicit `drop` is idiomatic when the intent to release early is not obvious from the structure alone.

---

## 8. The Type Alias Habit

Shared state in concurrent code tends to grow complex types quickly:

```rust
// Without alias — repeated in every function that touches shared state
type SharedClients = Arc<Mutex<Vec<Client>>>;

fn handle_client(clients: Arc<Mutex<Vec<Client>>>, stream: TcpStream) { ... }
fn remove_client(clients: Arc<Mutex<Vec<Client>>>, id: u64) { ... }

// With alias — readable and easy to change in one place
type SharedClients = Arc<Mutex<Vec<Client>>>;

fn handle_client(clients: SharedClients, stream: TcpStream) { ... }
fn remove_client(clients: SharedClients, id: u64) { ... }
```

You will also see this for error types: `type Result<T> = std::result::Result<T, ServerError>`.

---

## 9. Testing: Write Tests in the Same File

Rust's `#[cfg(test)]` module lets you put unit tests directly in the same file as the code they test. Do not create a separate test file for unit tests.

For the chat server, the pure logic — message parsing, username validation, formatting — is what you unit test. The networking integration is covered by integration tests or manual testing:

```rust
pub fn parse_command(input: &str) -> Option<(&str, &str)> {
    let input = input.trim();
    if let Some(rest) = input.strip_prefix('/') {
        let (cmd, args) = rest.split_once(' ').unwrap_or((rest, ""));
        Some((cmd, args))
    } else {
        None
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_join_command() {
        assert_eq!(parse_command("/join alice"), Some(("join", "alice")));
    }

    #[test]
    fn non_command_returns_none() {
        assert_eq!(parse_command("hello world"), None);
    }

    #[test]
    fn command_with_no_args() {
        assert_eq!(parse_command("/quit"), Some(("quit", "")));
    }
}
```

Run with `cargo test`. No extra framework or configuration needed.

---

## 10. Clippy: Your Automated Code Reviewer

`cargo clippy` is Rust's linter. Run it constantly. It catches:

- `if x == true` should be `if x`
- `vec.len() == 0` should be `vec.is_empty()`
- `for i in 0..vec.len() { vec[i] }` should use `.iter()` or `.iter().enumerate()`
- Unnecessary `.clone()` calls
- `match x { true => ..., false => ... }` should be `if x { ... } else { ... }`
- Hundreds more

Add this to `Cargo.toml` to treat Clippy warnings as build errors:

```toml
[lints.clippy]
all = "warn"
```

Run `cargo clippy -- -D warnings` in CI to enforce this automatically.
