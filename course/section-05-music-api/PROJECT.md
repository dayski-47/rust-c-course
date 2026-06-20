# Section 05 Project - melody-api

## What You're Building

A production-quality REST API for a music library. Songs, artists, albums, full CRUD operations, proper error handling, middleware, and Docker packaging. This is the first project in the course that looks like something you'd actually deploy at work.

The four previous projects got you to know Rust: you understand ownership, async, error handling, networking, and JSON serialization. This project is where you apply all of that to build a real backend service - one with the structure, patterns, and practices you'd see in a professional codebase.

By the end, you'll have:
- A working REST API with 15+ endpoints
- A SQLite database with compile-time checked SQL queries
- Centralized error handling that turns Rust errors into HTTP responses
- Logging, CORS, and timeout middleware
- A Dockerfile that produces a small, deployable image
- Integration tests you can run with `cargo test`

The next two sections will build on this foundation directly.

## The Full API Spec

Every endpoint the finished API should expose:

```
Health:
  GET  /health                       Check if the server is running

Artists:
  GET  /artists                      List all artists (supports ?limit=&offset=)
  POST /artists                      Create an artist
  GET  /artists/:id                  Get an artist plus their albums
  PUT  /artists/:id                  Update an artist
  DELETE /artists/:id                Delete an artist (fails if they have albums)

Albums:
  GET  /albums                       List albums (supports ?artist_id= ?limit= ?offset=)
  POST /albums                       Create an album (requires valid artist_id)
  GET  /albums/:id                   Get an album plus its songs
  PUT  /albums/:id                   Update an album
  DELETE /albums/:id                 Delete an album (fails if it has songs)

Songs:
  GET  /songs                        List songs (supports ?album_id= ?artist_id= ?limit= ?offset=)
  POST /songs                        Add a song (requires valid album_id)
  GET  /songs/:id                    Get one song
  PUT  /songs/:id                    Update a song
  DELETE /songs/:id                  Delete a song

Search:
  GET  /search?q=query               Search songs, albums, artists by name/title
```

All list endpoints should support pagination with `?limit=20&offset=0` (default limit 20, max 100).

All successful creates return 201 Created. All successful deletes return 204 No Content. List and get operations return 200 OK. Not found returns 404. Validation failures return 422 with a list of errors.

## Database Schema

Create this in `migrations/20240101000001_initial_schema.sql`:

```sql
CREATE TABLE artists (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    bio TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE albums (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    artist_id INTEGER NOT NULL REFERENCES artists(id),
    title TEXT NOT NULL,
    release_year INTEGER,
    genre TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE songs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    album_id INTEGER NOT NULL REFERENCES albums(id),
    title TEXT NOT NULL,
    duration_seconds INTEGER,
    track_number INTEGER,
    file_path TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## Project Structure

Aim for this layout by the end:

```
melody-api/
├── Cargo.toml
├── Dockerfile
├── docker-compose.yml
├── .env                    ← local dev config (not in git)
├── .env.example            ← template for other devs (in git)
├── .dockerignore
├── migrations/
│   └── 20240101000001_initial_schema.sql
└── src/
    ├── main.rs             ← startup, router assembly, graceful shutdown
    ├── config.rs           ← load env vars into a Config struct
    ├── error.rs            ← AppError enum + IntoResponse impl
    ├── db.rs               ← create_pool(), run_migrations()
    ├── routes/
    │   ├── mod.rs          ← assemble the sub-routers
    │   ├── health.rs
    │   ├── artists.rs      ← list, create, get_one, update, delete
    │   ├── albums.rs
    │   ├── songs.rs
    │   └── search.rs
    └── models/
        ├── mod.rs
        ├── artist.rs       ← Artist, CreateArtistRequest, UpdateArtistRequest
        ├── album.rs        ← Album, CreateAlbumRequest, UpdateAlbumRequest
        └── song.rs         ← Song, CreateSongRequest, UpdateSongRequest
```

Each `models/` file should define:
- The database row struct (derives `Serialize`, `FromRow`)
- The create request struct (derives `Deserialize`) with a `.validate()` method
- The update request struct (all fields `Option<T>`)
- A response struct if the shape returned to clients differs from the DB row

## Cargo.toml Dependencies

```toml
[package]
name = "melody-api"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["sqlite", "runtime-tokio", "migrate"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace", "compression-full", "timeout"] }
thiserror = "1"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
dotenvy = "0.15"
```

## Critical Axum Patterns

### 1. Application State

Axum shares state via `State` extractor. Your state is the database pool:

```rust
use axum::extract::State;
use sqlx::SqlitePool;

#[derive(Clone)]
pub struct AppState {
    pub db: SqlitePool,
}

// In your handler:
pub async fn list_artists(
    State(state): State<AppState>,
) -> Result<Json<Vec<Artist>>, AppError> {
    // run your SQL query here, return Ok(Json(results))
    // ...
}
```

Wire it up in your router:
```rust
let state = AppState { db: pool };
let app = Router::new()
    .route("/artists", get(list_artists).post(create_artist))
    .with_state(state);
```

### 2. Extracting Path Params and Request Bodies

```rust
use axum::extract::{Path, Json};

// GET /artists/:id
pub async fn get_artist(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Artist>, AppError> {
    // ...
}

// POST /artists
pub async fn create_artist(
    State(state): State<AppState>,
    Json(payload): Json<CreateArtistRequest>,
) -> Result<(StatusCode, Json<Artist>), AppError> {
    // ...
}
```

Note: `Json` body extractor must be the LAST parameter in the function signature. Axum reads extractors left to right, and reading the HTTP request body consumes it - you can't read it twice. Any extractor that needs the body (`Json`, `Form`, `Bytes`) must come after all other extractors (`State`, `Path`, `Query`, `Extension`) that read from headers or URL segments.

### 3. The AppError Pattern

This is the most important pattern in this section. Every handler returns `Result<T, AppError>`. `AppError` implements `IntoResponse` so Axum can convert it to an HTTP response:

```rust
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde_json::json;

pub enum AppError {
    NotFound(String),
    BadRequest(String),
    Internal(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound(msg)  => (StatusCode::NOT_FOUND, msg),
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg),
            AppError::Internal(msg)  => (StatusCode::INTERNAL_SERVER_ERROR, msg),
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}

// Convert sqlx errors automatically.
// The From trait is how ? knows how to convert sqlx::Error → AppError.
// If you haven't seen From<T> before, see Section 2 → docs/05-traits-the-rust-interface.md
impl From<sqlx::Error> for AppError {
    fn from(e: sqlx::Error) -> Self {
        match e {
            sqlx::Error::RowNotFound => AppError::NotFound("not found".into()),
            e => AppError::Internal(e.to_string()),
        }
    }
}
```

### 4. SQLx Setup - DATABASE_URL Required

SQLx's `sqlx::query!` macro checks SQL at compile time. This requires a connected database during compilation:

```bash
# Install the CLI:
cargo install sqlx-cli --no-default-features --features sqlite

# Create the database and run migrations:
export DATABASE_URL="sqlite:./melody.db"
sqlx database create
sqlx migrate run

# Now compile (the macro connects to verify your SQL):
cargo build
```

If you don't want to run the macro version yet, use `sqlx::query()` (no `!`) during development - it works without DATABASE_URL but doesn't check SQL at compile time.

### 5. Running a Query and Mapping Results

```rust
// With compile-time checking (requires DATABASE_URL and migrated DB):
let artists = sqlx::query_as!(
    Artist,
    "SELECT id, name, bio, created_at FROM artists ORDER BY name LIMIT ? OFFSET ?",
    limit,
    offset
)
.fetch_all(&state.db)
.await?;  // ? converts sqlx::Error to AppError via our From impl
```

For the result to map correctly, your struct field names must EXACTLY match the SQL column names. If a column can be NULL in the schema, the struct field must be `Option<T>`.

## Engineering Approach: Domain-Driven Design

Before writing handlers, define the domain:

**Entities and their invariants:**
- Artist: must have a name (non-empty string). Bio is optional.
- Album: must have a title and must belong to an existing Artist. `artist_id` must reference a real artist row.
- Song: must have a title and must belong to an existing Album. Duration must be positive if present. Track number must be positive if present.

**Valid operations and their constraints:**
- Creating a Song requires an `album_id` that exists. If the `album_id` doesn't exist, return 404 (not 500).
- Deleting an Artist that has Albums should either fail (409 Conflict) or cascade. Choose one and document it.
- Updating a Song's `album_id` to a non-existent album should fail with 404.

These constraints become the WHERE clauses in your SQL and the validation logic in your handlers. If you don't think about them before coding, you'll discover them the hard way when data gets corrupted.

## What You Bring From Sections 1-4

- **S1/S3 error handling → AppError**: you've been returning `Result` and using `?` since section 1. The `AppError` type is just a named, structured version of the same pattern. Now errors also carry HTTP status codes.
- **S4 async**: every Axum handler is an async function. Every database call uses `.await`. You're using the same `tokio::main` runtime you learned in S4.
- **S4 serde**: `GithubRepo` used `#[derive(Deserialize)]` in S4. Now you use `#[derive(Serialize, Deserialize)]` on `CreateArtistRequest`, `ArtistResponse`, etc. Same tool.
- **S2 structs + data modeling**: the domain modeling you practiced in S2 (defining `SystemSnapshot`) is exactly what you're doing with `Artist`, `Album`, `Song`.
- **S3 Arc for shared state**: `AppState` uses `Arc<SqlitePool>` - same `Arc` you used in S3 for the client list, now shared across async handlers instead of threads.

## How This Project Works in Rust - The Full Picture

Axum works by matching incoming HTTP requests to handler functions. When a request arrives for `GET /artists/42`, Axum extracts `42` from the URL path (`Path(id): Path<i64>`), extracts the database pool from shared state (`State(pool): State<SqlitePool>`), runs your async handler function, and serializes the return value to JSON.

Your handler queries SQLite through SQLx, which returns typed Rust structs. You convert those into response structs (possibly different from the database structs - separation of concerns) and return them as `Json(response)`. Axum calls serde to serialize the struct to JSON and writes the HTTP response.

Errors flow through `AppError`: if the database returns "not found", your handler returns `Err(AppError::NotFound)`. Axum's `IntoResponse` implementation for `AppError` converts this to a 404 response with a JSON error body. One error type, consistent responses everywhere.

The pool is the key to async database access. SQLx manages a pool of SQLite connections so multiple requests can run concurrently. Each handler borrows one connection from the pool for the duration of the request, then returns it.

## Milestones

Work through these in order. Each milestone is a working, testable state - don't move on until the current one works.

**Milestone 1 - Working server**
Create the Cargo project with `cargo new melody-api`. Add dependencies. Write a `main.rs` with a single `GET /health` endpoint that returns `{"status": "ok"}`. Run it and curl the endpoint.

Expected:
```
$ cargo run
Starting melody-api on 0.0.0.0:3000
```

Test: `curl http://localhost:3000/health` should return `{"status":"ok"}`.

**Milestone 2 - Database connection**
Set `DATABASE_URL=sqlite:music.db` in your `.env`. Write `src/db.rs` with a `create_pool()` function using `SqlitePool::connect`. Write `src/config.rs` to load the URL from the environment. Create the migration file and wire `sqlx::migrate!()` into startup. Verify the database file is created and the tables exist (`sqlite3 music.db .tables`).

Expected:
```
$ cargo run
Starting melody-api on 0.0.0.0:3000
Database: ./music.db
Migrations: applied successfully
```

Test: `sqlite3 music.db .tables` should print `albums  artists  songs`.

**Milestone 3 - Artists CRUD**
Define `Artist`, `CreateArtistRequest`, and `UpdateArtistRequest` in `src/models/artist.rs`. Implement all five artist handlers (`list`, `create`, `get_one`, `update`, `delete`). Add them to the router. Test all five with curl. At this stage it's okay to use `.unwrap()` - you'll fix that in Milestone 6.

Expected - POST /artists:
```bash
$ curl -X POST http://localhost:3000/artists \
  -H "Content-Type: application/json" \
  -d '{"name":"Miles Davis","bio":"Jazz legend"}'

# Response (201 Created):
{
  "id": 1,
  "name": "Miles Davis",
  "bio": "Jazz legend",
  "created_at": "2024-01-15T14:23:01"
}
```

Expected - GET /artists:
```bash
$ curl http://localhost:3000/artists

[
  {"id":1,"name":"Miles Davis","bio":"Jazz legend","created_at":"..."},
  {"id":2,"name":"John Coltrane","bio":null,"created_at":"..."}
]
```

Empty list `[]` if no artists exist yet - that's correct, not an error.

**Milestone 4 - Albums CRUD**
Same pattern as artists but albums belong to artists. The `CREATE` endpoint should reject an `artist_id` that doesn't exist (check foreign key errors from SQLx). The `GET /albums/:id` endpoint should return the album plus all its songs in the response body.

**Milestone 5 - Songs CRUD**
Same pattern. Songs belong to albums. `GET /songs` should support filtering by `?album_id=` and `?artist_id=`. The `GET /artists/:id` endpoint should also return the artist's albums in the response.

**Milestone 6 - Error handling**
Write `src/error.rs` with the `AppError` enum and its `IntoResponse` implementation. Change every `.unwrap()` in your handlers to `?` (which converts `sqlx::Error` to `AppError::Database` via `#[from]`). Change every `None` response to `Err(AppError::NotFound)`. Change every handler return type to `Result<..., AppError>`. Add validation to all create/update request structs and return `AppError::Validation` with all field errors.

Expected - 404 response:
```bash
$ curl -i http://localhost:3000/artists/999

HTTP/1.1 404 Not Found
Content-Type: application/json

{"error":"Artist not found"}
```

If you're getting 500 instead of 404, check your `From<sqlx::Error>` implementation - `sqlx::Error::RowNotFound` should map to `AppError::NotFound`.

**Milestone 7 - Search**
Implement `GET /search?q=query`. It should query artists, albums, and songs using SQL `LIKE '%' || ? || '%'` and return all matches in a single JSON response. Shape the response as `{ "artists": [...], "albums": [...], "songs": [...] }`.

**Milestone 8 - Middleware**
Add `TraceLayer`, `CorsLayer`, `CompressionLayer`, and `TimeoutLayer` to your router using `ServiceBuilder`. Test that request logging appears in the console. Test CORS by checking that the response includes the right headers.

**Milestone 9 - Docker**
Write the `Dockerfile` (multi-stage build). Write `.dockerignore`. Write `docker-compose.yml` with a named volume for the database. Run `docker compose up --build` and verify the API works from inside the container. Check the health endpoint: `curl http://localhost:3000/health`.

**Milestone 10 - Integration tests**
Write integration tests in `src/routes/artists.rs` (or a separate `tests/` directory) using `SqlitePool::connect("sqlite::memory:")` and Axum's `TestClient` or `axum::test`. Each test should create a fresh in-memory database, run migrations, create the router with test state, and make HTTP requests against it. Cover at minimum: create, get (found), get (not found → 404), update, delete, and list with pagination.

## The AppError Pattern

Here's the shape to implement in `src/error.rs`. Read Doc 04 for the full implementation with `IntoResponse`.

`AppError` is an enum with variants for each failure category:

```
AppError::NotFound                       → 404
AppError::BadRequest(String)             → 400 with message
AppError::Validation(Vec<String>)        → 422 with list of errors
AppError::Database(sqlx::Error)          → 500 (log internally, generic message to client)
AppError::Internal                       → 500
```

`impl IntoResponse for AppError` maps each variant to a status code and a JSON body with at least an `"error"` field. For `Validation`, include a `"details"` array.

Once you implement this, any handler returning `Result<T, AppError>` automatically gets correct HTTP responses for free. No per-handler match arms needed.

Define a type alias in `error.rs`:

```rust
pub type ApiResult<T> = Result<T, AppError>;
```

Use `ApiResult<Json<Song>>` in handler signatures for brevity.

## Hints

Do not skip ahead to read these until you're stuck. The struggle is the learning.

**Getting the database pool into handlers**: Use `State(pool): State<SqlitePool>` as the first argument. Wire `pool` into `AppState` and call `.with_state(state)` on the router.

**Single-row lookups**: Use `fetch_optional(&pool)` not `fetch_one`. It returns `Option<Row>`. Convert `None` to `AppError::NotFound` with `.ok_or(AppError::NotFound)?`.

**Getting the newly created row's ID**: `sqlx::query!(...).execute(&pool).await?.last_insert_rowid()` returns the `ROWID` of the inserted row. Use it to fetch the full row for the 201 response.

**SQLx compile-time verification**: You must have `DATABASE_URL` set and the database file with schema present at compile time. If you get an error about "couldn't connect to database", run `export DATABASE_URL=sqlite:music.db` and create the database first.

**Search query**: Use `WHERE title LIKE '%' || ? || '%'` with the query parameter bound to `?`. This is SQLite's way of doing substring search.

**Joining artists and their albums**: You can either run two queries (get artist, then get albums for that artist) or use a JOIN and reshape the results. Two queries is simpler to start with.

**Foreign key enforcement in SQLite**: SQLite does not enforce foreign keys by default. Run `PRAGMA foreign_keys = ON` after connecting, or use `sqlx::sqlite::SqliteConnectOptions` to enable it. Without this, inserting a song with an invalid `album_id` will silently succeed.

Enable foreign keys via connect options:
```rust
use sqlx::sqlite::{SqliteConnectOptions, SqlitePoolOptions};
use std::str::FromStr;

let options = SqliteConnectOptions::from_str(&database_url)?
    .foreign_keys(true);

let pool = SqlitePoolOptions::new()
    .connect_with(options)
    .await?;
```

**Pagination**: Query parameters `?limit=20&offset=0`. Your SQL: `LIMIT ? OFFSET ?`. Cap the limit at 100 to prevent clients from requesting everything at once.

**DotenVy order of operations**: Call `dotenvy::dotenv().ok()` as the very first line of `main()`, before anything reads environment variables.

## Testing the API with curl

Once Milestone 3 is done, test with these commands:

```bash
# Health check
curl http://localhost:3000/health

# Create an artist
curl -X POST http://localhost:3000/artists \
  -H "Content-Type: application/json" \
  -d '{"name": "The Beatles", "bio": "A rock band from Liverpool"}'

# List artists
curl http://localhost:3000/artists

# Get one artist (replace 1 with the actual id)
curl http://localhost:3000/artists/1

# Update an artist
curl -X PUT http://localhost:3000/artists/1 \
  -H "Content-Type: application/json" \
  -d '{"bio": "The most successful band of the 20th century"}'

# Delete an artist
curl -X DELETE http://localhost:3000/artists/1

# Create an album (must use an existing artist_id)
curl -X POST http://localhost:3000/albums \
  -H "Content-Type: application/json" \
  -d '{"artist_id": 1, "title": "Abbey Road", "release_year": 1969, "genre": "Rock"}'

# Create a song
curl -X POST http://localhost:3000/songs \
  -H "Content-Type: application/json" \
  -d '{"album_id": 1, "title": "Come Together", "track_number": 1, "duration_seconds": 259}'

# Search
curl "http://localhost:3000/search?q=beatles"

# Test 404
curl http://localhost:3000/songs/99999

# Test validation (missing required field)
curl -X POST http://localhost:3000/artists \
  -H "Content-Type: application/json" \
  -d '{"name": ""}'
```

For integration tests using Axum's test utilities, look at the `axum::test` module and `tower::ServiceExt`. You create the app in a test, call it with a mock request, and assert on the response status and body - all without starting a real TCP server.

## Stretch Goals

These are genuinely challenging additions. Attempt them after all 10 milestones are complete.

**Pagination with Link headers**: Instead of returning a `total_count` field, return a `Link` header per RFC 5988 with `next`, `prev`, `first`, and `last` URLs. This is what the GitHub API does.

**Sorting**: Add `?sort=title&order=asc` to list endpoints. Build the ORDER BY clause dynamically, but whitelist the allowed sort fields to prevent SQL injection from user-controlled column names.

**Bulk create**: `POST /songs/batch` that accepts an array of songs and creates them all in a single transaction. Return partial success info if some fail validation.

**Export endpoints**: `GET /albums/:id/export/m3u` returns an M3U playlist file. `GET /songs?format=csv` returns a CSV file. Both require correct `Content-Type` and `Content-Disposition` headers.

**Full-text search with SQLite FTS5**: SQLite has a built-in full-text search engine. Create a virtual FTS5 table and keep it in sync with songs/albums/artists. Switch the `/search` endpoint to use FTS5 for much better search quality.

**Rate limiting**: Use `tower::limit::RateLimitLayer` to limit each IP to N requests per second. This requires adding a per-IP request count, which means per-request state - a meaningful architectural challenge.

## Common Problems

**`thread 'main' panicked at 'no DATABASE_URL'`**
The `sqlx::query!` macro needs DATABASE_URL at compile time. Either set `export DATABASE_URL="sqlite:./melody.db"` or switch to `sqlx::query()` (no `!`) for now.

**`Cannot borrow immutable field 'state.db'`**
Make `AppState` derive `Clone`. The `State` extractor clones the state for each request.

**`expected fn pointer, found fn item` when registering routes**
This happens when your handler function has a signature mismatch. Make sure all extractors are in the right order (State first, Path second, Json last).

**`Json(payload)` body not extracting**
The client must send `Content-Type: application/json`. In curl: `-H "Content-Type: application/json"`.

**Migration not running**
Call `sqlx::migrate!().run(&pool).await?` in your startup code, before creating the router.

**Getting 500 instead of 404 on missing rows**
`sqlx::query_as!` with `.fetch_one()` panics on `RowNotFound`. Use `.fetch_optional()` instead and map `None` to `AppError::NotFound`:
```rust
let artist = sqlx::query_as!(Artist, "SELECT ... WHERE id = ?", id)
    .fetch_optional(&state.db)
    .await?
    .ok_or_else(|| AppError::NotFound("Artist not found".into()))?;
```

**Foreign key violations silently succeed**
SQLite doesn't enforce foreign keys by default. Enable them via `SqliteConnectOptions` (see the hint in the Hints section) or inserts with invalid `artist_id` / `album_id` will silently succeed and corrupt your data.
