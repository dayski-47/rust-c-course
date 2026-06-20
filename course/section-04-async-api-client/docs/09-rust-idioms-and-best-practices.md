# 07 — Rust Idioms and Best Practices

> **Project:** Async API Client — querying HTTP APIs with `reqwest`, deserializing JSON with `serde`, and orchestrating concurrent requests with Tokio tasks.

## In This Section

You are building an async API client. You will hit three Rust idioms immediately that do not appear in synchronous code:

- **`async fn` in traits** — required when you want a trait with async methods (a `ApiClient` trait, for example). This is now stable in Rust 1.75+. For older toolchains the `async-trait` crate was needed; you will still encounter it in the ecosystem.
- **`#[tokio::test]`** — the async equivalent of `#[test]`. Without it, your test function is `async fn` but nothing runs the future.
- **`#[instrument]`** — a single attribute from the `tracing` crate that wraps every `async fn` with a span. Structured logs are non-negotiable in production async code because standard `println!` logs lose their context across `.await` points.

---

## 1. Naming Conventions

The Rust community follows these naming conventions. Your code looks wrong to any Rust developer if you don't:

| What | Convention | Example |
|------|-----------|---------|
| Variables, functions, methods | `snake_case` | `fetch_user`, `parse_response` |
| Types, structs, enums, traits | `PascalCase` | `ApiClient`, `UserResponse`, `ClientError` |
| Constants, statics | `UPPER_SNAKE_CASE` | `BASE_URL`, `DEFAULT_TIMEOUT_SECS` |
| Modules | `snake_case` | `mod client`, `mod models` |
| Enum variants | `PascalCase` | `Ok`, `Err`, `Some`, `None`, `RateLimited` |

Do not use `camelCase` for variables (that is JavaScript). Do not use `ALL_CAPS` for types.

---

## 2. Prefer Owned Types in Structs, References in Function Signatures

A C developer's instinct: "use a pointer to avoid copying." In Rust, this leads to lifetime headaches.

**Rule: Structs should own their data.** Use `String`, not `&str`. Use `Vec<T>`, not `&[T]`.

**Rule: Functions should borrow.** Function parameters should be `&str` (not `String`), `&[T]` (not `Vec<T>`), `&T` (not `T`) unless you need to store or transfer ownership.

Why this works: `String` can be passed as `&str` automatically (deref coercion). `Vec<T>` can be passed as `&[T]` automatically. The reverse is not true.

In the API client, response models own their fields, and query functions borrow parameters:

```rust
#[derive(Debug, serde::Deserialize)]
struct UserResponse {
    id: u64,
    name: String,      // owned — the struct is the source of truth
    email: String,     // owned
}

// Good: borrow the query string, return owned result
async fn search_users(client: &reqwest::Client, query: &str) -> anyhow::Result<Vec<UserResponse>> {
    let resp = client
        .get("https://api.example.com/users")
        .query(&[("q", query)])
        .send()
        .await?
        .json::<Vec<UserResponse>>()
        .await?;
    Ok(resp)
}
```

---

## 3. The Builder Pattern for Complex Construction

`reqwest::Client` is itself built with a builder. You will use this pattern constantly in async code:

```rust
// Bad: positional constructor — what does `true` mean here?
let client = ApiClient::new("https://api.example.com", "my-key", true, 30, None);

// Good: self-documenting
let client = ApiClient::builder()
    .base_url("https://api.example.com")
    .api_key("my-key")
    .timeout_secs(30)
    .retry_on_rate_limit(true)
    .build()?;
```

Under the hood, your `ApiClient::builder()` will call `reqwest::Client::builder()` — you are composing builders. This is idiomatic for any struct that wraps an HTTP client or database pool.

---

## 4. Error Handling Hierarchy

**Do not use:** `unwrap()` or `expect()` in production paths — network calls, JSON parsing, and environment variable reads all fail in ways outside your control.

**Early development:** `Box<dyn std::error::Error>` as the error type. Gets you started without designing an error type first.

**Applications (tools, services):** `anyhow::Error`. Context-aware, stackable. The `anyhow::Context` trait lets you attach messages.

**Libraries:** `thiserror` with a typed enum. Callers need to match on variants; `anyhow` would hide them. For an API client library, define variants for `RateLimited`, `NotFound`, `AuthError`, `NetworkError`, etc. so callers can handle each case differently.

The pattern:

```rust
#[derive(Debug, thiserror::Error)]
enum ClientError {
    #[error("rate limited: retry after {retry_after_secs}s")]
    RateLimited { retry_after_secs: u64 },
    #[error("not found: {0}")]
    NotFound(String),
    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),
}

async fn run() -> anyhow::Result<()> {
    let client = build_client().context("failed to build API client")?;
    let users = client.fetch_users()
        .await
        .context("failed to fetch users")?;
    println!("fetched {} users", users.len());
    Ok(())
}

#[tokio::main]
async fn main() {
    if let Err(e) = run().await {
        eprintln!("error: {e:#}");
        std::process::exit(1);
    }
}
```

---

## 5. Don't Match on Bool — Use the Right Abstraction

```rust
// Bad: checking is_some() then calling unwrap()
if response.next_page.is_some() {
    let url = response.next_page.unwrap();
    fetch(url).await?;
}

// Good: if let destructures in one step
if let Some(url) = response.next_page {
    fetch(url).await?;
}

// Bad: manually re-wrapping an error
if result.is_err() {
    return Err(result.unwrap_err());
}

// Good: ? propagates automatically
let body = response.json::<UserResponse>().await?;
```

---

## 6. Closures as First-Class Values — and `async` Closures

In synchronous Rust, you use closures with iterators and higher-order functions. In async code, the equivalent is async closures or `async` blocks:

```rust
// Spawning multiple concurrent requests with tokio::spawn
let handles: Vec<_> = user_ids
    .into_iter()
    .map(|id| {
        let client = client.clone();  // Arc clone — cheap
        tokio::spawn(async move {
            client.fetch_user(id).await
        })
    })
    .collect();

// Join all concurrently
let results: Vec<_> = futures::future::join_all(handles).await;
```

The `move` keyword is required because the `async` block must own `client` and `id` — it may be polled on a different thread than the one that created it (`Send + 'static`). Clone your `Arc`-wrapped values before the `move`, exactly as you would with `thread::spawn`.

Know the three closure traits:

- `Fn` — can be called multiple times, borrows captured variables
- `FnMut` — can be called multiple times, mutably borrows captured variables
- `FnOnce` — can only be called once, takes ownership of captured variables

---

## 7. `async fn` in Traits

If you want to define an `ApiClient` trait with async methods, the syntax is now stable in Rust 1.75+:

```rust
trait ApiClient {
    async fn fetch_user(&self, id: u64) -> Result<UserResponse, ClientError>;
    async fn create_user(&self, payload: &CreateUser) -> Result<UserResponse, ClientError>;
}
```

Before Rust 1.75, this required the `async-trait` crate:

```rust
#[async_trait::async_trait]
trait ApiClient {
    async fn fetch_user(&self, id: u64) -> Result<UserResponse, ClientError>;
}
```

You will encounter `#[async_trait]` in older codebases and many popular crates. It works by desugaring `async fn` into `fn(...) -> Pin<Box<dyn Future<Output = ...> + Send>>`. The stable syntax (Rust 1.75+) does the same thing but without the macro.

---

## 8. The Type Alias Habit

Async code produces verbose types quickly:

```rust
// Without alias
fn spawn_fetch(id: u64) -> tokio::task::JoinHandle<Result<UserResponse, ClientError>> { ... }

// With alias
type FetchHandle = tokio::task::JoinHandle<Result<UserResponse, ClientError>>;

fn spawn_fetch(id: u64) -> FetchHandle { ... }
```

The most common alias you will define is a local `Result`:

```rust
type Result<T> = std::result::Result<T, ClientError>;
```

This lets you write `Result<UserResponse>` everywhere instead of `std::result::Result<UserResponse, ClientError>`.

---

## 9. Testing Async Code with `#[tokio::test]`

The standard `#[test]` attribute does not work for `async fn`. Use `#[tokio::test]` instead:

```rust
// Bad: this compiles but never actually awaits the future
#[test]
async fn test_fetch_user() {
    let result = client.fetch_user(1).await;
    assert!(result.is_ok());
}

// Good: tokio::test sets up a runtime and drives the future to completion
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn fetch_returns_user_for_valid_id() {
        let client = MockApiClient::new();
        client.expect_fetch_user().returning(|_| Ok(UserResponse {
            id: 1,
            name: "Alice".to_owned(),
            email: "alice@example.com".to_owned(),
        }));

        let result = client.fetch_user(1).await;
        assert!(result.is_ok());
        assert_eq!(result.unwrap().name, "Alice");
    }

    #[tokio::test]
    async fn fetch_returns_not_found_for_missing_id() {
        let client = MockApiClient::new();
        client.expect_fetch_user()
            .returning(|id| Err(ClientError::NotFound(format!("user {id} not found"))));

        let result = client.fetch_user(999).await;
        assert!(matches!(result, Err(ClientError::NotFound(_))));
    }
}
```

Run with `cargo test`. The `#[tokio::test]` macro spins up a Tokio runtime for each test function, runs the future to completion, and tears down the runtime — exactly like `#[tokio::main]` does for `main`.

---

## 10. `#[instrument]` for Production Async Code

In async code, `println!` and `eprintln!` lose their meaning because multiple concurrent tasks interleave their output. The `tracing` crate is the standard solution. The `#[instrument]` attribute wraps an `async fn` with a tracing span automatically:

```rust
use tracing::instrument;

#[instrument(skip(client), fields(user_id = %id))]
async fn fetch_user(client: &reqwest::Client, id: u64) -> anyhow::Result<UserResponse> {
    tracing::debug!("sending request");
    let resp = client
        .get(format!("https://api.example.com/users/{id}"))
        .send()
        .await
        .context("request failed")?;
    tracing::debug!(status = %resp.status(), "got response");
    resp.json().await.context("failed to deserialize")
}
```

`skip(client)` prevents the `reqwest::Client` (which does not implement `Debug`) from being logged. `fields(user_id = %id)` attaches the ID to every log line emitted inside the span.

Initialize tracing in `main`:

```rust
tracing_subscriber::fmt::init();
```

Every `tracing::debug!`, `tracing::info!`, and `tracing::error!` inside any `#[instrument]`-annotated function will automatically include the span's fields — including which task, which request ID, and which user ID. This is what separates debuggable async services from ones that are impossible to reason about in production.

---

## 11. Clippy: Your Automated Code Reviewer

`cargo clippy` is Rust's linter. Run it constantly. It catches:

- `if x == true` should be `if x`
- `vec.len() == 0` should be `vec.is_empty()`
- `for i in 0..vec.len() { vec[i] }` should use `.iter()` or `.iter().enumerate()`
- Unnecessary `.clone()` calls
- Redundant `.await` patterns
- Hundreds more

Add this to `Cargo.toml` to treat Clippy warnings as build errors:

```toml
[lints.clippy]
all = "warn"
```

Run `cargo clippy -- -D warnings` in CI to enforce this automatically.
