# Doc 01 - Web Frameworks and Axum Introduction 🟢

## Engineering Methodology: Domain-Driven Design (DDD)

Domain-Driven Design means you model the real world first, then build the code around that model.

Before writing a single HTTP handler, answer:
- What are the entities in this domain? (Song, Album, Artist - these are the nouns)
- What are the relationships? (An Album belongs to an Artist. A Song belongs to an Album.)
- What operations are valid? (You can't add a Song to an Album that doesn't exist. You can't delete an Artist that has Albums.)
- What are the invariants? (A Song must always have a title. An Album must always have a valid artist_id.)

The code you write is a translation of these rules into Rust. The database schema is a translation of the entity relationships. The REST endpoints are a translation of the valid operations.

If you skip domain modeling and go straight to writing handlers, you'll write handlers that violate business rules (accepting songs without albums, allowing artist deletion that leaves orphaned data). DDD prevents this by making you think before you code.

---

You've built a TCP chat server and an async API client. Now it's time to build something that actually looks like production backend code: a REST API. This section introduces Axum - the framework you'll use for the next three sections.

## What Axum Is

Axum is an async web framework built on top of two mature crates: **Tokio** (which you already know) and **hyper** (a battle-tested HTTP library). You don't call Tokio or hyper directly - Axum handles the plumbing and gives you a clean interface for defining routes and handlers.

It's currently the most popular production Rust web framework alongside Actix-web. Axum's philosophy is that middleware and routing should compose cleanly. That's not a marketing claim - it's structural: Axum is built on **Tower**, a generic middleware framework, which means any Tower-compatible middleware works with your Axum app out of the box.

If you've touched Node.js or Express, the mental model transfers well. If not, don't worry - you'll see the pattern immediately.

## Adding Axum to Cargo.toml

```toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

`tower-http` is the Tower middleware collection for HTTP-specific concerns like CORS, compression, and tracing. You'll use it heavily in Doc 05.

## A Minimal Server

Here's the smallest possible Axum server that actually runs:

```rust
use axum::{Router, routing::get};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/health", get(health_handler));

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Listening on port 3000");

    axum::serve(listener, app).await.unwrap();
}

async fn health_handler() -> &'static str {
    "OK"
}
```

Run it with `cargo run`, then in another terminal: `curl http://localhost:3000/health`. You'll get `OK`.

Three things to notice:
1. `Router::new().route(path, method(handler))` - routes are defined declaratively
2. Handlers are just async functions - no special boilerplate
3. `axum::serve` takes a `TcpListener` (from Tokio) and the router - same Tokio you know

## Handler Functions

A handler is any async function that Axum can call. It takes **extractors** as arguments and returns something that implements `IntoResponse`. Axum uses Rust's type system to figure out what the handler needs and how to satisfy those needs automatically.

```rust
use axum::{
    extract::Path,
    response::{Json, IntoResponse},
    http::StatusCode,
};
use serde_json::json;

// Simplest handler - returns a string
async fn hello() -> &'static str {
    "Hello, world!"
}

// Returns a status code + JSON body as a tuple
async fn not_found_example() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, Json(json!({ "error": "not found" })))
}

// Takes a path parameter, returns JSON
async fn get_song(Path(id): Path<u64>) -> impl IntoResponse {
    Json(json!({ "id": id, "title": "Song title here" }))
}
```

The `impl IntoResponse` return type is the key. Axum knows how to convert many types into HTTP responses automatically:

| Return type | HTTP result |
|---|---|
| `&str` or `String` | 200 with text/plain body |
| `Json<T>` | 200 with application/json body |
| `StatusCode` | Status code, empty body |
| `(StatusCode, Json<T>)` | Status code + JSON body |
| `(StatusCode, String)` | Status code + text body |

## The Router Type

The Router is where you wire paths to handlers:

```rust
use axum::routing::{get, post, put, delete, patch};

let app = Router::new()
    .route("/health",      get(health_handler))
    .route("/songs",       get(list_songs).post(create_song))
    .route("/songs/:id",   get(get_song).put(update_song).delete(delete_song))
    .route("/artists",     get(list_artists).post(create_artist));
```

Notice that `.route()` lets you chain multiple HTTP methods on the same path. `get(handler)` registers the handler for GET requests; `post(handler)` for POST; and so on. Axum provides `get`, `post`, `put`, `delete`, and `patch` from `axum::routing`.

## Building a Slightly More Complete Server

Here's an example that shows JSON responses with proper status codes - a preview of what you'll build:

```rust
use axum::{Router, routing::{get, post}, Json, http::StatusCode};
use serde::{Serialize, Deserialize};
use tokio::net::TcpListener;

#[derive(Serialize)]
struct Song {
    id: u64,
    title: String,
    artist: String,
}

#[derive(Deserialize)]
struct CreateSongRequest {
    title: String,
    artist: String,
}

async fn list_songs() -> Json<Vec<Song>> {
    // In the real project, this hits the database
    Json(vec![
        Song { id: 1, title: "Bohemian Rhapsody".into(), artist: "Queen".into() },
    ])
}

async fn create_song(
    Json(body): Json<CreateSongRequest>,
) -> (StatusCode, Json<Song>) {
    let song = Song {
        id: 42,
        title: body.title,
        artist: body.artist,
    };
    (StatusCode::CREATED, Json(song))
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/songs", get(list_songs).post(create_song));

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

Test with curl:
```bash
# List songs
curl http://localhost:3000/songs

# Create a song
curl -X POST http://localhost:3000/songs \
  -H "Content-Type: application/json" \
  -d '{"title": "Stairway to Heaven", "artist": "Led Zeppelin"}'
```

## Quick Comparison to Other Frameworks

If you've seen Express.js (Node.js):

```
Express:  app.get('/songs', (req, res) => { res.json(data) })
Axum:     .route("/songs", get(list_songs))
```

The big difference: Axum's handlers are type-checked at compile time. If your handler expects a JSON body of type `CreateSongRequest` but the client sends malformed JSON, Axum rejects the request automatically with a 422 before your handler is even called. No runtime surprises.

If you've seen Django or Flask (Python): it's similar in spirit but everything is async by default and types are enforced by the compiler rather than by runtime errors.

## How It Breaks

- Handler panicking crashes the whole server unless you add the `tower_http::catch_panic` layer. In production, always add this.
- Not handling all error cases in a handler: returning 500 for a 404 situation (record not found is not an internal server error).
- State sharing: `AppState` must implement `Clone`. If you put a non-`Clone` field in it, your server won't compile. Use `Arc<T>` for non-`Clone` state.
- CORS: if you don't configure CORS, browser frontends can't call your API. This is a very common "why doesn't my frontend work" bug.

## Common Mistakes

**Forgetting `#[tokio::main]`**: Axum requires an async runtime. If you forget the macro, you'll get a confusing error about not being in an async context. Every Axum main function needs `#[tokio::main]`.

**Returning the wrong type**: If your handler returns `String` but you meant to return JSON, the response will have `Content-Type: text/plain` instead of `application/json`. Always wrap JSON data with `Json<T>`.

**Wrapping the handler call in extra parens**: Write `get(handler)` not `get(handler())`. Axum wants the function itself, not the result of calling it.

**Missing serde derives**: If you try to return `Json<MySong>` but `MySong` doesn't derive `Serialize`, you get a compile error. Add `#[derive(Serialize, Deserialize)]` to all types that cross the HTTP boundary.

**Binding to 127.0.0.1 in Docker**: If you deploy inside a container, bind to `0.0.0.0` not `127.0.0.1`. `127.0.0.1` only accepts connections from inside the container.
