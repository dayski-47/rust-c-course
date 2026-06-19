# Doc 02 — Routing, Handlers, and State 🟡

Real APIs are not just "return this fixed string." They take parameters from the URL, parse query strings, accept JSON bodies, and share a database connection across every request. This doc covers all of that — the mechanics you'll use in literally every handler you write.

## Path Parameters

When your URL is `/songs/42`, the `42` is a path parameter. Declare it in the route with `:id`, then extract it in the handler with `Path`:

```rust
use axum::{extract::Path, routing::get, Router};

async fn get_song(Path(id): Path<u64>) -> String {
    format!("Getting song with id {}", id)
}

let app = Router::new()
    .route("/songs/:id", get(get_song));
```

For multiple parameters — say `/artists/:artist_id/albums/:album_id` — extract a tuple:

```rust
async fn get_album(
    Path((artist_id, album_id)): Path<(u64, u64)>,
) -> String {
    format!("Artist {}, Album {}", artist_id, album_id)
}
```

Or extract into a struct for readability:

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct AlbumPath {
    artist_id: u64,
    album_id: u64,
}

async fn get_album(Path(params): Path<AlbumPath>) -> String {
    format!("Artist {}, Album {}", params.artist_id, params.album_id)
}
```

If the path parameter can't be parsed to the expected type (someone hits `/songs/not-a-number`), Axum returns 400 Bad Request automatically.

## Query Parameters 🟡

Query parameters come after the `?` in the URL: `/songs?limit=10&offset=0`. Use the `Query` extractor with a struct:

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct ListParams {
    limit: Option<u32>,
    offset: Option<u32>,
    artist_id: Option<u64>,
}

async fn list_songs(Query(params): Query<ListParams>) -> String {
    let limit = params.limit.unwrap_or(20);
    let offset = params.offset.unwrap_or(0);
    format!("limit={}, offset={}, artist={:?}", limit, offset, params.artist_id)
}
```

Fields marked `Option<T>` are optional in the query string. If a required field is missing, Axum returns 400. If the value can't be parsed to the type, also 400.

## Request Bodies (JSON)

POST, PUT, and PATCH requests carry a body. Extract it with `Json`:

```rust
use axum::{extract::Json, http::StatusCode};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateSongRequest {
    title: String,
    album_id: u64,
    track_number: Option<u32>,
    duration_seconds: Option<u32>,
}

#[derive(Serialize)]
struct Song {
    id: u64,
    title: String,
    album_id: u64,
}

async fn create_song(
    Json(body): Json<CreateSongRequest>,
) -> (StatusCode, Json<Song>) {
    // body.title, body.album_id are available here
    let song = Song {
        id: 1, // would come from DB insert
        title: body.title,
        album_id: body.album_id,
    };
    (StatusCode::CREATED, Json(song))
}
```

If the request body is missing, malformed JSON, or doesn't match the expected shape, Axum returns 422 Unprocessable Entity before your handler is called.

## Sharing State Across Handlers 🔴

Here's the problem: every handler is its own function, but all handlers need access to the database connection pool. How do you share that?

The answer is the `State` extractor. You attach application state to the router, and handlers extract it by type.

First, define your state:

```rust
use sqlx::SqlitePool;

#[derive(Clone)]
struct AppState {
    db: SqlitePool,
}
```

It must implement `Clone`. Connection pools are designed to be cloned cheaply — they share the underlying pool under the hood.

Attach state to the router:

```rust
let state = AppState { db: pool };

let app = Router::new()
    .route("/songs", get(list_songs).post(create_song))
    .with_state(state);
```

Extract it in handlers:

```rust
use axum::extract::State;

async fn list_songs(
    State(state): State<AppState>,
) -> Json<Vec<Song>> {
    // state.db is your SqlitePool
    let songs = sqlx::query_as!(Song, "SELECT id, title FROM songs")
        .fetch_all(&state.db)
        .await
        .unwrap();
    Json(songs)
}
```

If you have both path parameters and state, both go in the function signature. Order matters — `State` must come before `Json` (body extractors must be last, because the HTTP request body can only be read once — after `Json` consumes it, no other extractor can access it):

```rust
async fn get_song(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> impl IntoResponse {
    // state.db and id are both available
}

async fn create_song(
    State(state): State<AppState>,
    Json(body): Json<CreateSongRequest>,
) -> impl IntoResponse {
    // body must be last
}
```

## Nested Routers

As your API grows, you don't want all routes in one place. Split by resource:

```rust
mod routes {
    pub mod artists;
    pub mod albums;
    pub mod songs;
}

// In main.rs:
use axum::Router;

fn artists_router() -> Router<AppState> {
    Router::new()
        .route("/", get(routes::artists::list).post(routes::artists::create))
        .route("/:id", get(routes::artists::get_one)
            .put(routes::artists::update)
            .delete(routes::artists::delete))
}

let app = Router::new()
    .nest("/artists", artists_router())
    .nest("/albums",  albums_router())
    .nest("/songs",   songs_router())
    .with_state(state);
```

`nest("/artists", router)` means all routes in `artists_router()` get the `/artists` prefix. The route `/` inside the nested router becomes `/artists`. The route `/:id` becomes `/artists/:id`.

## Returning JSON Responses

Your handlers should return meaningful HTTP status codes, not always 200:

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Json};

// 200 OK (default when you return Json directly)
async fn get_song(Path(id): Path<u64>) -> Json<Song> {
    Json(Song { id, title: "Some song".into() })
}

// 201 Created for successful creation
async fn create_song(
    Json(body): Json<CreateSongRequest>,
) -> (StatusCode, Json<Song>) {
    (StatusCode::CREATED, Json(Song { id: 1, title: body.title }))
}

// 404 Not Found
async fn get_missing() -> StatusCode {
    StatusCode::NOT_FOUND
}

// 404 with body
async fn get_song_or_404(Path(id): Path<u64>) -> impl IntoResponse {
    if id == 999 {
        return (
            StatusCode::NOT_FOUND,
            Json(serde_json::json!({ "error": "song not found", "id": id })),
        ).into_response();
    }
    (StatusCode::OK, Json(Song { id, title: "Found it".into() })).into_response()
}
```

The common status codes you'll use:

| Constant | Code | When |
|---|---|---|
| `StatusCode::OK` | 200 | Successful GET, PUT |
| `StatusCode::CREATED` | 201 | Successful POST that created something |
| `StatusCode::NO_CONTENT` | 204 | Successful DELETE (no body) |
| `StatusCode::BAD_REQUEST` | 400 | Invalid input from client |
| `StatusCode::NOT_FOUND` | 404 | Resource doesn't exist |
| `StatusCode::UNPROCESSABLE_ENTITY` | 422 | Body parsed but semantically wrong |
| `StatusCode::INTERNAL_SERVER_ERROR` | 500 | Something broke on the server |

## Standard Error Response Structure

Don't return raw error strings — clients can't reliably parse an unstructured string to decide what to show users or how to retry. A consistent JSON shape means every error, from every endpoint, is parseable the same way. Use a consistent shape:

```rust
use serde::Serialize;

#[derive(Serialize)]
struct ErrorResponse {
    error: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    details: Option<Vec<String>>,
}

// 404 response:
let body = ErrorResponse {
    error: "Song not found".into(),
    details: None,
};
(StatusCode::NOT_FOUND, Json(body))

// 400 with validation details:
let body = ErrorResponse {
    error: "Validation failed".into(),
    details: Some(vec![
        "title: must not be empty".into(),
        "duration_seconds: must be positive".into(),
    ]),
};
(StatusCode::BAD_REQUEST, Json(body))
```

This is exactly the error shape you'll implement with `AppError` in Doc 04.

## How It Breaks

- Path parameter type mismatch: the URL has `:id` as a string but you're extracting it as `u64`. "abc" would fail to parse at runtime, returning 422 Unprocessable Entity. Handle this gracefully.
- Missing state extraction: if your handler function takes `State(pool): State<SqlitePool>` but you didn't add the pool to the router with `.with_state()`, the server compiles but panics on the first request.
- Extractor ordering matters in Axum: the `Json` extractor must be last in the function parameters (it consumes the request body). Getting this wrong is a compile error.
- Route conflicts: having both `/users/:id` and `/users/me` — Axum matches in order, so `/users/me` might be caught by `/users/:id` first. Define specific routes before wildcard routes.

## Common Mistakes

**Putting the body extractor (`Json`) before other extractors**: Axum has a rule — the `Json` body extractor must be the last argument in your handler. If you put `State` after `Json`, you get a compile error. Keep `State`, `Path`, and `Query` first; `Json` last.

**Not cloning state**: `AppState` must implement `Clone`. If you store something in it that isn't `Clone` (like a raw mutex guard), you'll hit a confusing error. Use `Arc` to wrap non-cloneable items: `Arc<Mutex<NonCloneableThing>>`.

**Returning `Json` when the value isn't `Serialize`**: Axum will fail to compile with a hard-to-read error about trait bounds. The fix is adding `#[derive(Serialize)]` to the struct.

**Using `.unwrap()` in handlers for database errors**: This panics the handler task on DB errors. The right approach is to propagate errors — which is exactly what Doc 04 covers with `AppError`. Don't use `.unwrap()` in production handlers.

**Forgetting that `nest()` strips the prefix**: If you nest a router under `/artists` and the inner router has a route `/artists/:id`, the actual path becomes `/artists/artists/:id`. Put only the suffix in the nested router (`:id`, not `/artists/:id`).
