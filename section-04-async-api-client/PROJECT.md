# Section 4 Project: ghanalyze — GitHub Repository Analyzer

## What You're Building

`ghanalyze` is a command-line tool that takes a GitHub username (or organization name) and produces a detailed analysis of their public repositories. It fetches repository metadata, language breakdowns, and activity information — all concurrently using async Tokio — and formats the results into a readable report.

This project brings together everything from the section docs: async tasks, tokio::spawn for concurrent fetching, reqwest for HTTP, serde for JSON parsing, rate limit awareness, and proper async error handling.

---

## What the Output Looks Like

```
$ ghanalyze torvalds

GitHub Analysis: torvalds
=========================================
Fetching repo list...
Analyzing 8 repos concurrently...
[1/8] torvalds/linux
[2/8] torvalds/subsurface
[3/8] torvalds/test-tlb
...

=========================================
RESULTS
=========================================

Total repos: 8 | Stars: 185,432 | Forks: 47,231

Top Repos by Stars:
  linux        ⭐ 178,432  🍴 47,123  Lang: C        Updated: 2 days ago
  subsurface   ⭐  3,124   🍴    823  Lang: C++      Updated: 1 week ago
  uemacs       ⭐  1,987   🍴    234  Lang: C        Updated: 3 months ago

Languages (across all repos):
  C           67.3%  ########################################
  C++         18.4%  ###########
  Perl        12.1%  #######
  Python       2.2%  #

Activity:
  Most active repo: linux
  Archived repos:   0
  Repos updated in last 30 days: 2
```

---

## Architecture

The program follows a pipeline:

```
1. Parse CLI arguments
   |
   v
2. Create reqwest::Client (once, shared with Arc)
   |
   v
3. Fetch list of repos for the username
   (GET /users/{username}/repos?per_page=100, paginated)
   |
   v
4. For each repo (concurrently, bounded by semaphore):
   |--- GET /repos/{owner}/{repo}/languages
   |    (returns language byte counts)
   |--- Check rate limit headers on each response
   |
   v
5. Collect results as tasks complete (JoinSet)
   |
   v
6. Compute summary statistics
   |
   v
7. Display formatted report
   |
   v (optional)
8. Save full JSON output to file
```

The key architectural decisions:
- **One `reqwest::Client`** wrapped in `Arc<Client>`, shared across all tasks.
- **`JoinSet`** to spawn and collect repo analysis tasks.
- **`tokio::sync::Semaphore`** to limit concurrent requests (respect rate limits).
- **Individual task failures** don't crash the whole analysis — if one repo's language fetch fails, the others continue.

---

## Requirements

### CLI Interface

```
ghanalyze <username> [OPTIONS]

ARGS:
    <username>    GitHub username or organization name

OPTIONS:
    --limit <N>           Maximum number of repos to analyze [default: 30]
    --output <file>       Save full JSON report to this file
    --token <TOKEN>       GitHub personal access token
                          (or set GITHUB_TOKEN environment variable)
    --concurrency <N>     Maximum concurrent API requests [default: 10]

EXAMPLES:
    ghanalyze torvalds
    ghanalyze rust-lang --limit 50 --token ghp_xxxxx
    ghanalyze microsoft --output report.json --limit 100
```

Use the `clap` crate for argument parsing:

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

### Concurrency

Use `tokio::spawn` for each repo's detail fetch, bounded by a `tokio::sync::Semaphore` with the concurrency limit. All repo fetches should run at the same time (up to the limit), not one after another.

### Rate Limit Handling

Check `X-RateLimit-Remaining` on every API response. If remaining drops below 10, compute seconds until `X-RateLimit-Reset` and call `tokio::time::sleep` to wait it out before continuing.

### Error Handling

A single failing repo fetch should not crash the program. Report the failure (with the repo name and error), skip that repo, and continue with the rest.

### Progress Output

Show progress to stderr as repos are analyzed: which repo is being fetched, how many done out of total. This goes to stderr so it doesn't interfere with piped output to `--output`.

---

## GitHub API Endpoints

All endpoints require these headers:
```
User-Agent: ghanalyze/1.0
Accept: application/vnd.github.v3+json
Authorization: Bearer <token>   (optional, but increases rate limit)
```

### List User Repositories

```
GET https://api.github.com/users/{username}/repos?per_page=100&page=1&sort=updated
```

Returns an array of repo objects. The `per_page` maximum is 100. If a user has more than 100 repos, you need to paginate using the `?page=2`, `?page=3`, etc. parameters. The response headers include `Link` with next/prev/last page URLs when there are more pages.

For the basic milestone, fetching only page 1 (up to 100 repos) is fine.

### Repository Languages

```
GET https://api.github.com/repos/{owner}/{repo}/languages
```

Returns a JSON object mapping language names to byte counts:
```json
{
  "Rust": 2345678,
  "Shell": 12345,
  "Makefile": 3456
}
```

### Rate Limit Status

```
GET https://api.github.com/rate_limit
```

Returns current rate limit status without consuming any quota. Useful to check before starting a large analysis.

### Check if API is accessible

```
GET https://api.github.com
```

Returns 200 with a list of API endpoint URLs. Use this to verify your token works.

---

## Data Structures

These are the serde structs you'll need. Put them in `src/models.rs`:

### GitHub API Response Shapes

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

/// Matches the GitHub API /users/{username}/repos response items.
#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct GitHubRepo {
    pub id: u64,
    pub name: String,
    pub full_name: String,
    pub description: Option<String>,
    pub stargazers_count: u64,
    pub forks_count: u64,
    pub open_issues_count: u64,
    pub language: Option<String>,     // Primary language detected by GitHub
    pub updated_at: String,           // ISO 8601 timestamp: "2024-01-15T08:30:00Z"
    pub created_at: String,
    pub html_url: String,
    pub clone_url: String,
    pub archived: bool,
    pub fork: bool,                   // True if this is a fork of another repo
    pub size: u64,                    // Repository size in kilobytes
}

/// Language byte counts from /repos/{owner}/{repo}/languages.
/// The key is the language name, value is bytes of code.
/// Example: {"Rust": 2345678, "Shell": 12345}
pub type LanguageMap = HashMap<String, u64>;

/// The analysis result for a single repository (combining repo metadata + languages).
#[derive(Debug, Serialize, Clone)]
pub struct RepoAnalysis {
    pub repo: GitHubRepo,
    pub languages: LanguageMap,
}

/// The complete analysis report for a user/org.
#[derive(Debug, Serialize)]
pub struct AnalysisReport {
    pub username: String,
    pub generated_at: String,
    pub total_repos_analyzed: usize,
    pub total_stars: u64,
    pub total_forks: u64,
    pub repos: Vec<RepoAnalysis>,
    pub language_totals: LanguageMap,   // Aggregated across all repos
}
```

### Reference: What the GitHub API Actually Returns

Here is the JSON shape for a single repo (simplified to show the fields you need):

```json
{
  "id": 2325298,
  "name": "linux",
  "full_name": "torvalds/linux",
  "description": "Linux kernel source tree",
  "stargazers_count": 178432,
  "forks_count": 47123,
  "open_issues_count": 512,
  "language": "C",
  "updated_at": "2024-01-15T08:30:00Z",
  "created_at": "2011-09-04T22:48:12Z",
  "html_url": "https://github.com/torvalds/linux",
  "clone_url": "https://github.com/torvalds/linux.git",
  "archived": false,
  "fork": false,
  "size": 4021456
}
```

The array returned by `/users/{username}/repos` contains many more fields than this. Serde will silently ignore any fields not in your struct.

---

## Suggested Project Structure

```
ghanalyze/
├── Cargo.toml
└── src/
    ├── main.rs        — CLI argument parsing, orchestration, output formatting
    ├── models.rs      — Serde structs (GitHubRepo, RepoAnalysis, etc.)
    ├── github.rs      — All GitHub API calls (fetch_repos, fetch_languages, etc.)
    ├── analysis.rs    — Compute statistics from collected data
    └── display.rs     — Format and print the report
```

---

## Critical Async Patterns for This Project

### 1. The tokio::main Setup

Every async program needs a runtime. The `#[tokio::main]` macro creates one:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // your async code here
    // you can use .await anywhere in this function
    Ok(())
}
```

Add to Cargo.toml:
```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
clap = { version = "4", features = ["derive"] }
```

### 2. Making HTTP Requests with reqwest

Create ONE client per program (not one per request — that's expensive). Share it with `Arc`:

```rust
use reqwest::Client;

let client = Client::builder()
    .user_agent("ghanalyze/0.1")
    .build()?;

// GET request that deserializes JSON directly into your struct:
let repos: Vec<GithubRepo> = client
    .get(format!("https://api.github.com/users/{}/repos", username))
    .query(&[("per_page", "100")])
    .send()
    .await?
    .json()
    .await?;
```

### 3. Deserializing GitHub API Responses

Your struct field names must match the JSON keys (or use `#[serde(rename)]`):

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct GithubRepo {
    pub name: String,
    pub description: Option<String>,  // Option because it can be null
    pub stargazers_count: u32,
    pub language: Option<String>,
    pub fork: bool,
    pub size: u64,
}
```

If a field in your struct doesn't exist in the JSON, deserialization fails. If a JSON field isn't in your struct, it's silently ignored. You don't have to map every JSON field — only the ones you need.

### 4. Concurrent Requests with JoinSet

Fetching language data for 50 repos sequentially would take minutes. Do it concurrently:

```rust
use tokio::task::JoinSet;
use std::sync::Arc;

let client = Arc::new(client);  // wrap in Arc to share across tasks
let mut set = JoinSet::new();

for repo in &repos {
    let client = Arc::clone(&client);
    let repo_name = repo.name.clone();  // clone so the task owns it
    let owner = username.to_string();
    
    set.spawn(async move {
        // fetch language data for this repo
        // use client, owner, repo_name here
        // return the result
    });
}

// Collect results as tasks complete:
while let Some(result) = set.join_next().await {
    match result {
        Ok(data) => {
            // ...  ← store data, update stats, etc.
        }
        Err(e) => eprintln!("task failed: {e}"),
    }
}
```

### 5. Rate Limit Awareness

The GitHub API allows 60 unauthenticated requests per hour. Your analyzer makes 1 request for the repo list + 1 per repo for languages. With 50 repos, that's 51 requests — close to the limit.

Check the response headers:
```rust
let response = client.get(...).send().await?;
if let Some(remaining) = response.headers().get("x-ratelimit-remaining") {
    let remaining: u32 = remaining.to_str()?.parse()?;
    if remaining < 5 {
        eprintln!("Warning: only {} API requests remaining", remaining);
    }
}
```

For a GitHub token (increases limit to 5000/hour):
```rust
let client = Client::builder()
    .user_agent("ghanalyze/0.1")
    .default_headers({
        let mut headers = reqwest::header::HeaderMap::new();
        if let Ok(token) = std::env::var("GITHUB_TOKEN") {
            headers.insert(
                reqwest::header::AUTHORIZATION,
                format!("Bearer {}", token).parse()?,
            );
        }
        headers
    })
    .build()?;
```

---

## Engineering Approach: Failure-Mode Analysis

Before building `ghanalyze`, list every external interaction and its failure modes. This is not optional — it's the design step that determines whether your error handling will be complete.

**GitHub API calls:**

- `GET /users/{username}/repos` — User doesn't exist (404), rate limited (429 or 403 with rate limit headers), network connection failure, valid user but 0 repos (empty JSON array — this is success, not a failure, but your code must handle it), malformed JSON in the response body.
- `GET /repos/{owner}/{repo}/languages` — Repo exists but has no detected language data (empty JSON object `{}` — valid response, not an error), request timeout on a slow repo, 404 if the repo was deleted between your list fetch and your language fetch.

**Rate limit handling — the details matter:**

- The `X-RateLimit-Remaining` header tells you how many requests you can still make in the current window. Check it on every response, not just at startup.
- The `X-RateLimit-Reset` header is a Unix timestamp (seconds since epoch) indicating when the limit resets. If `Remaining` is 0, compute `Reset - now` to get the sleep duration.
- Checking only at the start of the program is wrong: your remaining count decreases with every request, and you have no idea how many you'll make until you know how many repos the user has.

**Pagination — the silent truncation failure:**

- GitHub returns at most 100 repos per page. If a user has more than 100 repos and you don't handle pagination, you silently analyze only the first 100. No error, no warning — just incomplete data. The `Link` response header tells you if there is a next page: `<https://...?page=2>; rel="next"`. Failing to handle pagination is a category 4 failure (silent data loss).

Write the failure-mode list for your specific implementation before writing the code. Then handle each failure mode explicitly — not with a generic `?` that loses the failure type, but with specific error variants that let you respond differently to each.

---

## What You Bring From Sections 1–3

Nothing in this project is entirely new. The tools are new; the patterns are familiar:

- **S1 error handling:** Every `Result` chain in `ghanalyze` uses the same `?` propagation pattern you learned in Section 1. The difference is that the errors come from HTTP responses instead of TCP connections. The error type structure is the same.
- **S3 threads → S4 tasks:** In Section 3 you used `thread::spawn` for each chat client. Here you use `tokio::spawn` instead — same ownership rules (`'static`, `move` into the closure, `Arc::clone` for shared data), but tasks are cooperatively scheduled instead of preemptively. One Tokio thread can drive thousands of tasks; one OS thread can only run one thing.
- **S2 `HashMap`:** The language breakdown is a `HashMap<String, u64>` — language name to byte count. The same data structure you used in Section 2, applied to a different domain.
- **S2 structs + serde:** `GithubRepo`, `LanguageMap`, `AnalysisReport` are all structs. Same as Section 2 structs, but now they derive `Deserialize` so serde fills them from JSON automatically. You don't write any parsing code.
- **S3 `Arc`:** `Arc<reqwest::Client>` shared across spawned tasks is the same pattern as `Arc<Mutex<client_list>>` in Section 3 — shared ownership across concurrent units of work. The difference is tasks instead of threads, and no `Mutex` because `reqwest::Client` is already designed to be shared.

The difficulty here is not the Rust concepts. It's understanding what the GitHub API does, handling all its failure modes, and managing concurrency cleanly at the task level rather than the thread level.

---

## How This Project Works in Rust — The Full Picture

The data flow in `ghanalyze`:

The program starts by creating one `reqwest::Client`. This client maintains a connection pool and handles TLS negotiation — creating it is expensive, so you create it once. It goes into an `Arc<reqwest::Client>` so it can be shared across all spawned tasks.

To fetch all repos for a user, the program makes a paginated GET request to the GitHub API. The response comes back as raw bytes. `serde_json` deserializes those bytes directly into `Vec<GithubRepo>` using the `Deserialize` implementation that `#[derive(Deserialize)]` generated. No manual JSON parsing — serde does it from the struct definition.

For each repo in the list, the program spawns a Tokio task with `tokio::spawn`. Each task receives an `Arc` clone of the client (cheap — just an atomic counter increment) and an owned `repo` struct (moved into the task). A `Semaphore` limits how many tasks run simultaneously, preventing the program from opening hundreds of connections at once and hitting rate limits immediately.

A `JoinSet` tracks all the spawned tasks. As each task completes, `join_next()` returns its result. The results are collected and processed to compute statistics.

The key insight about async futures: none of the async code runs until you `.await` it. Calling `reqwest::get(url)` does not make a network request — it creates a `Future` struct that describes a network request. The request happens when the runtime polls that future. This is why futures can be cancelled cleanly: dropping an un-polled future is simply dropping a struct. No partial request was made. This also means you can compose complex combinations of operations (join, select, timeout) before starting any of them, with full control over when and whether they execute.

---

## Milestones

Work through these in order. Each milestone builds on the previous one.

### Milestone 1: Fetch and Print Repos (Blocking)

Get the project compiling and talking to the GitHub API. Use `reqwest::blocking::get()` (the synchronous version) first — no async needed yet. Just fetch and print the repo list as raw JSON.

Goals:
- Project compiles with reqwest and serde dependencies
- Reads username from command line arguments (can use `std::env::args()` for now)
- Makes a GET request to `/users/{username}/repos`
- Prints the raw JSON response
- Handles basic errors (network failure, bad username)

You are not writing async code yet. Get the basics working first.

### Milestone 2: Deserialize into Structs

Replace the raw JSON printing with proper serde deserialization into `Vec<GitHubRepo>`.

Goals:
- `GitHubRepo` struct defined with `#[derive(Deserialize)]`
- `.json::<Vec<GitHubRepo>>()` used on the response
- Print each repo's name, star count, and language in a simple table
- Handle the case where language is `None`

**Stuck on parsing the GitHub response?**

Check what the actual JSON looks like first:
```bash
curl -s "https://api.github.com/users/torvalds/repos" | head -100
```
Your struct fields must match the JSON keys exactly. Start with just `name` and `stargazers_count` — add more fields as you need them.

### Milestone 3: Make It Async

Convert from `reqwest::blocking` to async reqwest. Add Tokio.

Goals:
- `#[tokio::main]` on `main()`
- `reqwest::Client` created once (not per-request)
- All HTTP calls use `.await`
- Program still works identically to Milestone 2

This milestone is intentionally small — you're just changing the plumbing, not the logic.

### Milestone 4: Concurrent Language Fetching

For each repo, fetch its language breakdown concurrently using `tokio::spawn`.

Goals:
- For each repo in the list, spawn a task that calls `/repos/{owner}/{repo}/languages`
- Use `JoinSet` to collect results as tasks complete
- Limit concurrency to 10 simultaneous requests using `tokio::sync::Semaphore`
- Combine each `GitHubRepo` with its `LanguageMap` into `RepoAnalysis`
- Print progress to stderr: "[3/30] fetching torvalds/linux..."

At this point, all language fetches should run concurrently. Test with `--limit 20` and check that the fetching is visibly faster than sequential.

**Stuck on the `'static` / Arc requirement for spawned tasks?**

`tokio::spawn` requires that everything the task captures is `'static` — meaning it must be owned, not borrowed. You can't capture a `&str` or a reference into a local variable.

The fix: clone what you need before spawning. You saw `Arc::clone` used for the client map in Section 3 (chat server) — this is the same pattern: cheap reference-count increment, full shared ownership.

```rust
let repo_name = repo.name.clone();  // owned String, not &str
let client = Arc::clone(&client);   // cheap — increments a counter, not a deep clone
tokio::spawn(async move {           // move captures repo_name and client by ownership
    // ...
});
```

### Milestone 5: Rate Limit Handling

Read rate limit headers and back off when needed.

Goals:
- After each API response, read `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers
- If remaining < 10, print a warning and sleep until the reset time
- Test with a low concurrency (`--concurrency 1`) and many repos to trigger rate limiting

Hint: You'll need to pass the rate limit state somewhere accessible. An `Arc<Mutex<RateLimitState>>` shared across tasks works, or you can check headers per-response and just sleep inline.

### Milestone 6: Add Full CLI Parsing with clap

Replace manual `std::env::args()` parsing with `clap`.

Goals:
- `--limit <N>` argument (default 30)
- `--concurrency <N>` argument (default 10)
- `--token <TOKEN>` argument, or fall back to `GITHUB_TOKEN` env var
- `--output <file>` argument (just parse it for now, implement in Milestone 7)
- Helpful `--help` output generated automatically by clap

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

### Milestone 7: Compute and Display Statistics

Calculate the summary statistics and produce the formatted report.

Goals:
- Total star count and fork count across all repos
- Top 5 repos by star count
- Language totals across all repos (aggregate byte counts, compute percentages)
- Most recently updated repo
- Count of archived repos
- Nice formatted output matching the example at the top of this document

The bar chart for languages can be simple: 40 `#` characters for 100%, scale proportionally.

### Milestone 8: JSON Output

Save the full `AnalysisReport` to a JSON file when `--output` is specified.

Goals:
- Serialize `AnalysisReport` to JSON using `serde_json::to_writer_pretty()`
- Write to the specified file path
- Print "Report saved to report.json" to stderr after writing
- The JSON should be pretty-printed (human readable)

---

## Key Hints (Not Solutions)

**The reqwest Client.** Create it once in `main()` and wrap it in `Arc<reqwest::Client>`. Pass `Arc::clone(&client)` into each spawned task. Don't create a new `Client` per request — you'll lose connection pooling.

**Concurrent fetching pattern.** The pattern for fetching N things concurrently:

```rust
let semaphore = Arc::new(Semaphore::new(concurrency));
let mut set = JoinSet::new();

for repo in repos {
    let client = Arc::clone(&client);
    let sem = Arc::clone(&semaphore);
    let owner = username.clone();

    set.spawn(async move {
        let _permit = sem.acquire().await.unwrap();
        fetch_languages(&client, &owner, &repo.name).await
    });
}

while let Some(result) = set.join_next().await {
    // process each result as it arrives
}
```

**Rate limit headers.** Read them from the response before consuming the body. The headers are on `response.headers()`, which you can access before calling `.json()` or `.text()`.

**Handling one failing task.** When using `JoinSet`, each `join_next()` returns `Result<TaskOutput, JoinError>`. The inner `TaskOutput` itself might be `Result<RepoAnalysis, GhanalyzeError>`. You need to unwrap two layers. Pattern match the outer JoinError (task panic) separately from the inner GhanalyzeError (API failure):

```rust
while let Some(join_result) = set.join_next().await {
    match join_result {
        Err(join_err) => eprintln!("Task panicked: {}", join_err),
        Ok(Err(api_err)) => eprintln!("API error: {}", api_err),
        Ok(Ok(analysis)) => results.push(analysis),
    }
}
```

**Pagination.** If you want to fetch more than 100 repos, you need to handle pagination. The GitHub API includes a `Link` header on paginated responses:

```
Link: <https://api.github.com/...?page=2>; rel="next",
      <https://api.github.com/...?page=5>; rel="last"
```

For the basic project, fetching only the first 100 repos and capping with `--limit` is fine. Pagination is a stretch goal.

**Timestamps.** The GitHub API returns ISO 8601 timestamps like `"2024-01-15T08:30:00Z"`. To display "2 days ago" you need to parse the timestamp and compare it to now. The `chrono` crate handles this well:

```toml
chrono = { version = "0.4", features = ["serde"] }
```

For the initial milestones, displaying the raw timestamp string is fine.

---

## Cargo.toml Template

```toml
[package]
name = "ghanalyze"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "ghanalyze"
path = "src/main.rs"

[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
clap = { version = "4", features = ["derive"] }
thiserror = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Optional but recommended:
# chrono = { version = "0.4", features = ["serde"] }  # For timestamp parsing
```

---

## Stretch Goals

Once the main milestones are complete, these extend the project in interesting directions:

**Pagination support.** Fetch all pages of repos for users with more than 100 repos. Parse the `Link` response header to find the next page URL.

**Contributor analysis.** For each repo, fetch the top contributors from `/repos/{owner}/{repo}/contributors` and show who contributes most across the user's repos.

**Compare two users.** `ghanalyze compare alice bob` — fetch both users' repos, compute stats for each, and produce a side-by-side comparison table.

**Historical activity trend.** Use the commit activity endpoint `/repos/{owner}/{repo}/stats/commit_activity` to determine if a repo's activity has increased or decreased over the past year.

**Disk cache.** Save API responses to a `.ghanalyze-cache/` directory (JSON files named by repo). On subsequent runs, use cached data if it's less than an hour old. This avoids re-fetching during development. Use `serde_json::from_reader()` to read cached files.

**Progress bar.** Replace the simple "[N/M]" progress messages with a real terminal progress bar using the `indicatif` crate.

**Filter by language.** `ghanalyze torvalds --lang C` — only show repos where C is the primary language.

**Org support.** Detect whether the username is a GitHub user or organization and use the appropriate endpoint (`/users/{name}/repos` vs `/orgs/{name}/repos`).

---

## Common Gotchas in This Project

**Not setting User-Agent.** GitHub returns 403 Forbidden without it. Every request needs `User-Agent: ghanalyze/1.0`.

**Using reqwest::get() instead of Client.get().** If you use `reqwest::get()` inside your task loop, you create a new client per request. Create one `Client` and reuse it.

**Cloning the token unnecessarily.** If you pass the GitHub token to every API function, you'll be cloning a String on every call. Instead, build the auth header into the `Client` itself using `Client::builder().default_headers(...)`, or store the token in your `GitHubClient` struct.

**Forgetting that JoinSet::join_next() returns None when empty.** The `while let Some(result) = set.join_next().await` loop exits when all tasks have completed. If you add tasks after starting the loop, they won't be picked up. Add all tasks first, then enter the collection loop.

**Printing progress to stdout instead of stderr.** If you ever want to pipe the JSON output to a file or another program, progress messages mixed in with the JSON will break it. Print progress with `eprintln!` (stderr), data with `println!` (stdout).
