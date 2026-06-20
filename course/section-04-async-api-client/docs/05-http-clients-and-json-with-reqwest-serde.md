# 05 - HTTP Clients and JSON with reqwest and serde 🟡

This is where the rubber meets the road for `ghanalyze`. We need to make HTTP requests to the GitHub API and parse the JSON responses into Rust structs. Two crates handle this: `reqwest` for HTTP and `serde` for serialization/deserialization.

---

## reqwest: The Standard Async HTTP Client

`reqwest` is the most widely used HTTP client in the Rust ecosystem. It's built on top of Tokio and handles connection pooling, TLS, redirects, cookies, and everything else you'd expect from a modern HTTP client.

Add it to your `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

The `json` feature on reqwest enables the `.json::<T>()` method for automatic deserialization. The `derive` feature on serde enables the `#[derive(Deserialize, Serialize)]` macros.

---

## Making a Simple GET Request

```rust
#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    // reqwest::get() creates a one-off client and makes a GET request.
    // Fine for simple cases, but not for production use (explained below).
    let response = reqwest::get("https://api.github.com/users/torvalds").await?;

    println!("Status: {}", response.status());

    // Get the response body as a String:
    let body = response.text().await?;
    println!("Body: {}", &body[..200]);  // First 200 chars

    Ok(())
}
```

---

## The Client: Create Once, Reuse Many Times

`reqwest::get()` is convenient but creates a new HTTP client for each call. This is wasteful - HTTP clients maintain connection pools. Creating a new client per request means no connection reuse, which is slow.

**Create one `reqwest::Client` and reuse it for all requests:**

```rust
use reqwest::Client;

struct GitHubClient {
    client: Client,
    token: Option<String>,
}

impl GitHubClient {
    fn new(token: Option<String>) -> Self {
        let client = Client::builder()
            // GitHub requires a User-Agent header on all API requests.
            .user_agent("ghanalyze/1.0")
            .build()
            .expect("Failed to build HTTP client");

        GitHubClient { client, token }
    }
}
```

`reqwest::Client` is cheap to clone - it's backed by an `Arc` internally. This means you can freely pass `Client::clone()` across tasks without paying to copy connection pool state.

---

## Setting Headers

The GitHub API requires two headers:
- `User-Agent`: identifies your application (required, GitHub will reject requests without it)
- `Accept`: specifies which API version format you want

```rust
use reqwest::{Client, header};

async fn fetch_user_repos(
    client: &Client,
    username: &str,
    token: Option<&str>,
) -> Result<String, reqwest::Error> {
    let url = format!(
        "https://api.github.com/users/{}/repos?per_page=100",
        username
    );

    let mut request = client
        .get(&url)
        .header(header::USER_AGENT, "ghanalyze/1.0")
        .header(header::ACCEPT, "application/vnd.github.v3+json");

    // Add auth token if provided (increases rate limit from 60 to 5000/hour).
    if let Some(token) = token {
        request = request.bearer_auth(token);
    }

    let response = request.send().await?;
    response.text().await
}
```

---

## serde: Serialization and Deserialization

`serde` is a framework for converting Rust data structures to and from formats like JSON, TOML, YAML, and many others. It's ubiquitous - nearly every Rust crate that deals with structured data uses serde.

The core concept: derive `Deserialize` on any struct you want to deserialize from JSON, and `Serialize` on any struct you want to serialize to JSON.

```rust
use serde::{Deserialize, Serialize};

// This struct can now be deserialized from JSON and serialized to JSON.
#[derive(Debug, Deserialize, Serialize)]
struct GitHubRepo {
    id: u64,
    name: String,
    full_name: String,
    description: Option<String>,  // Option<T> handles null JSON values
    stargazers_count: u64,
    forks_count: u64,
    language: Option<String>,
    updated_at: String,
    html_url: String,
    archived: bool,
}
```

The struct fields must match the JSON field names by default. If the JSON has a field `stargazers_count`, your struct needs a field named `stargazers_count`.

---

## Mapping JSON to Rust Structs

Here's what the GitHub API returns for a single repository:

```json
{
  "id": 2325298,
  "name": "linux",
  "full_name": "torvalds/linux",
  "description": "Linux kernel source tree",
  "stargazers_count": 178432,
  "forks_count": 47123,
  "language": "C",
  "updated_at": "2024-01-15T08:30:00Z",
  "html_url": "https://github.com/torvalds/linux",
  "archived": false,
  "private": false
}
```

And the matching Rust struct:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct GitHubRepo {
    pub id: u64,
    pub name: String,
    pub full_name: String,
    pub description: Option<String>,    // null in JSON → None in Rust
    pub stargazers_count: u64,
    pub forks_count: u64,
    pub language: Option<String>,       // Some repos have no primary language
    pub updated_at: String,
    pub html_url: String,
    pub archived: bool,
    // private: bool - you can include or omit fields you don't need.
    // serde ignores fields in the JSON that aren't in your struct by default.
}
```

If you omit a field from the struct, serde ignores it in the JSON. You don't have to map every field - only the ones you need.

---

## Optional Fields and Attributes

### Handling null: Option<T>

Any JSON field that might be `null` should be `Option<T>`:

```rust
pub description: Option<String>,   // JSON: "description": null or "description": "text"
pub language: Option<String>,      // JSON: "language": null or "language": "Rust"
```

### Renaming fields: #[serde(rename)]

When the JSON field name doesn't match what you'd want in Rust:

```rust
#[derive(Deserialize)]
struct ApiResponse {
    #[serde(rename = "full_name")]  // JSON: "full_name", Rust field: repo_full_name
    repo_full_name: String,
}
```

### Skip serializing None values: #[serde(skip_serializing_if)]

When writing JSON output, you may not want to include null fields:

```rust
#[derive(Serialize)]
struct RepoSummary {
    name: String,

    #[serde(skip_serializing_if = "Option::is_none")]
    description: Option<String>,   // Omitted entirely from JSON output if None
}
```

---

## Deserializing HTTP Responses

reqwest can deserialize directly to your struct using `.json::<T>()`:

```rust
async fn fetch_repos(
    client: &reqwest::Client,
    username: &str,
) -> Result<Vec<GitHubRepo>, reqwest::Error> {
    let url = format!(
        "https://api.github.com/users/{}/repos?per_page=100",
        username
    );

    client
        .get(&url)
        .header("User-Agent", "ghanalyze/1.0")
        .header("Accept", "application/vnd.github.v3+json")
        .send()
        .await?
        .json::<Vec<GitHubRepo>>()  // Deserialize response body as a Vec of repos
        .await
}
```

If the JSON doesn't match your struct (wrong field types, unexpected structure), `.json()` returns an error. You'll want to check the actual API response first with `.text()` when debugging.

---

## Language Data: Using HashMap

The `/languages` endpoint returns a JSON object with language names as keys and byte counts as values:

```json
{
  "C": 15234567,
  "Makefile": 89023,
  "Perl": 234567,
  "Python": 12345
}
```

This maps naturally to `HashMap<String, u64>`:

```rust
use std::collections::HashMap;

async fn fetch_languages(
    client: &reqwest::Client,
    owner: &str,
    repo: &str,
) -> Result<HashMap<String, u64>, reqwest::Error> {
    let url = format!(
        "https://api.github.com/repos/{}/{}/languages",
        owner, repo
    );

    client
        .get(&url)
        .header("User-Agent", "ghanalyze/1.0")
        .header("Accept", "application/vnd.github.v3+json")
        .send()
        .await?
        .json::<HashMap<String, u64>>()
        .await
}
```

---

## Rate Limiting: Reading Response Headers

GitHub's API includes rate limit information in every response header:

```
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4987
X-RateLimit-Reset: 1704067200    (Unix timestamp when the limit resets)
```

Read these headers from the response before consuming the body:

```rust
use reqwest::Response;

async fn check_rate_limit(response: &Response) -> (u32, u32, u64) {
    let limit = response
        .headers()
        .get("X-RateLimit-Limit")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.parse().ok())
        .unwrap_or(60);

    let remaining = response
        .headers()
        .get("X-RateLimit-Remaining")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.parse().ok())
        .unwrap_or(0);

    let reset = response
        .headers()
        .get("X-RateLimit-Reset")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.parse().ok())
        .unwrap_or(0);

    (limit, remaining, reset)
}

async fn handle_rate_limit_if_needed(response: &Response) {
    let (_, remaining, reset_timestamp) = check_rate_limit(response);

    if remaining < 10 {
        let now = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_secs();

        if reset_timestamp > now {
            let wait_secs = reset_timestamp - now + 1;  // +1 second buffer
            eprintln!(
                "Rate limit nearly exhausted ({} remaining). Waiting {}s...",
                remaining, wait_secs
            );
            tokio::time::sleep(std::time::Duration::from_secs(wait_secs)).await;
        }
    }
}
```

---

## Handling reqwest Errors

`reqwest::Error` covers network failures, DNS errors, TLS errors, timeouts, and JSON deserialization failures. In `ghanalyze`, you'll want to propagate these through your `Result` chain:

```rust
use reqwest;

#[derive(Debug, thiserror::Error)]
pub enum AnalysisError {
    #[error("HTTP request failed: {0}")]
    Request(#[from] reqwest::Error),

    #[error("GitHub API returned error {status}: {message}")]
    ApiError { status: u16, message: String },

    #[error("Rate limit exceeded")]
    RateLimited,
}

async fn fetch_repo_safe(
    client: &reqwest::Client,
    url: &str,
) -> Result<GitHubRepo, AnalysisError> {
    let response = client
        .get(url)
        .header("User-Agent", "ghanalyze/1.0")
        .send()
        .await?;  // reqwest::Error converted to AnalysisError via #[from]

    if response.status() == reqwest::StatusCode::FORBIDDEN {
        return Err(AnalysisError::RateLimited);
    }

    if !response.status().is_success() {
        return Err(AnalysisError::ApiError {
            status: response.status().as_u16(),
            message: response.text().await.unwrap_or_default(),
        });
    }

    Ok(response.json::<GitHubRepo>().await?)
}
```

---

## Common Mistakes

**Creating a new `reqwest::Client` for every request.** Each client has its own connection pool. If you make one per request, you lose connection reuse and TLS handshake caching. Create one client, reuse it everywhere.

**Not setting User-Agent on GitHub API requests.** GitHub rejects requests without a User-Agent with a 403. This is easy to forget and produces a confusing error.

**Forgetting to check the HTTP status code.** `.json()` will fail if the response body isn't JSON, but it won't fail just because the status is 404. Always check `response.status().is_success()` before deserializing, or use `response.error_for_status()?.json()`.

**Trying to match every JSON field.** You don't need to map the entire GitHub API response. Only put the fields you actually need in your struct. Serde ignores the rest.

**Not handling Option for nullable fields.** If a JSON field can be `null` and your struct uses `String` instead of `Option<String>`, deserialization will fail on any response where that field is null. Use `Option<T>` liberally when you're not sure.

**Reading the body twice.** The HTTP response body is a stream - you can only read it once. If you call `.text().await` to debug, you've consumed the body and can't call `.json().await` afterward. Store the text, then parse it.
