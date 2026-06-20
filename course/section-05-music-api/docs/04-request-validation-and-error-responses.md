# Doc 04 — Request Validation and Error Responses 🟡

Users send bad data. Networks fail. Databases return errors. If your handler code is sprinkled with `.unwrap()` and raw panics, one bad request will crash a task and leave the user with a confusing 500 error — if they're lucky. The production approach is a single `AppError` type that your entire API uses to turn failures into meaningful HTTP responses.

## The Problem Without a Central Error Type

Here's what handlers look like without one:

```rust
async fn get_song(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> impl IntoResponse {
    match sqlx::query_as!(Song, "SELECT * FROM songs WHERE id = ?", id)
        .fetch_optional(&state.db)
        .await
    {
        Ok(Some(song)) => (StatusCode::OK, Json(song)).into_response(),
        Ok(None) => (StatusCode::NOT_FOUND, "not found").into_response(),
        Err(e) => {
            eprintln!("DB error: {e}");
            (StatusCode::INTERNAL_SERVER_ERROR, "database error").into_response()
        }
    }
}
```

That's a lot of noise for a single handler. Multiply it by 20 endpoints and you have a mess. The `AppError` pattern solves this by making handlers return `Result<T, AppError>`.

## Defining AppError with thiserror

You already know `thiserror` from the error handling chapter. In an Axum API, you use it to define all the ways a request can fail:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("not found")]
    NotFound,

    #[error("bad request: {0}")]
    BadRequest(String),

    #[error("validation error")]
    Validation(Vec<String>),

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("internal error")]
    Internal,
}
```

The `#[from]` on the `Database` variant means any `sqlx::Error` will automatically convert to `AppError::Database` via the `?` operator. This is the same `#[from]` you used in the error handling chapter — it generates a `From<sqlx::Error> for AppError` impl automatically.

## Implementing IntoResponse for AppError 🔴

This is the key step. Axum can turn your error type into an HTTP response if it implements `IntoResponse`. Once it does, handlers can return `Result<impl IntoResponse, AppError>` and Axum handles the rest automatically.

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Json, Response},
};
use serde_json::json;

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message, details) = match self {
            AppError::NotFound => (
                StatusCode::NOT_FOUND,
                "not found".to_string(),
                None,
            ),
            AppError::BadRequest(msg) => (
                StatusCode::BAD_REQUEST,
                msg,
                None,
            ),
            AppError::Validation(errors) => (
                StatusCode::UNPROCESSABLE_ENTITY,
                "validation failed".to_string(),
                Some(errors),
            ),
            AppError::Database(e) => {
                // Log the real error but don't expose it to the client
                tracing::error!("database error: {e}");
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "a database error occurred".to_string(),
                    None,
                )
            }
            AppError::Internal => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "an internal error occurred".to_string(),
                None,
            ),
        };

        let body = if let Some(d) = details {
            json!({ "error": message, "details": d })
        } else {
            json!({ "error": message })
        };

        (status, Json(body)).into_response()
    }
}
```

Notice the `Database` variant: you log the real error with `tracing::error!` but tell the client something generic. Leaking raw SQL errors to users is a security risk — they reveal your schema.

## Using AppError in Handlers

Now handlers look clean:

```rust
async fn get_song(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Song>, AppError> {
    let song = sqlx::query_as!(Song,
        "SELECT id, title, duration_seconds, track_number, album_id FROM songs WHERE id = ?",
        id
    )
    .fetch_optional(&state.db)
    .await?  // sqlx::Error → AppError::Database via #[from]
    .ok_or(AppError::NotFound)?;  // None → AppError::NotFound

    Ok(Json(song))
}
```

Two lines of query code instead of twelve lines of match arms. The `?` operators handle conversion automatically. This is what `#[from]` buys you.

## Validation — Parse, Don't Validate 🟡

The principle: don't take raw data, run checks on it, and then use the unchecked data. Instead, parse raw data into a type that can only exist if the data is valid. After the parse, there's nothing more to check.

Applied to a request struct:

```rust
use serde::Deserialize;

// Raw request — could contain anything
#[derive(Deserialize)]
pub struct CreateSongRequest {
    pub title: String,
    pub album_id: i64,
    pub track_number: Option<i32>,
    pub duration_seconds: Option<i32>,
}

impl CreateSongRequest {
    // Try to parse into a validated form
    pub fn validate(self) -> Result<ValidatedCreateSong, AppError> {
        let mut errors = Vec::new();

        if self.title.trim().is_empty() {
            errors.push("title: must not be empty".into());
        }
        if self.title.len() > 500 {
            errors.push("title: must be 500 characters or fewer".into());
        }
        if self.album_id <= 0 {
            errors.push("album_id: must be a positive integer".into());
        }
        if let Some(t) = self.track_number {
            if t <= 0 {
                errors.push("track_number: must be positive".into());
            }
        }
        if let Some(d) = self.duration_seconds {
            if d <= 0 {
                errors.push("duration_seconds: must be positive".into());
            }
        }

        if !errors.is_empty() {
            return Err(AppError::Validation(errors));
        }

        Ok(ValidatedCreateSong {
            title: self.title.trim().to_string(),
            album_id: self.album_id,
            track_number: self.track_number,
            duration_seconds: self.duration_seconds,
        })
    }
}

// This type can only be constructed by validate() — it is proof of validity
pub struct ValidatedCreateSong {
    pub title: String,
    pub album_id: i64,
    pub track_number: Option<i32>,
    pub duration_seconds: Option<i32>,
}
```

In your handler, you call `.validate()` at the boundary:

```rust
async fn create_song(
    State(state): State<AppState>,
    Json(body): Json<CreateSongRequest>,
) -> Result<(StatusCode, Json<Song>), AppError> {
    let validated = body.validate()?;  // Returns 422 with all errors if invalid

    let result = sqlx::query!(
        "INSERT INTO songs (album_id, title, track_number, duration_seconds)
         VALUES (?, ?, ?, ?)",
        validated.album_id,
        validated.title,
        validated.track_number,
        validated.duration_seconds,
    )
    .execute(&state.db)
    .await?;

    let song = get_song_by_id(&state.db, result.last_insert_rowid())
        .await?
        .ok_or(AppError::Internal)?;

    Ok((StatusCode::CREATED, Json(song)))
}
```

Notice: validation collects **all errors**, not just the first one. Users want to see every problem with their request in one response, not fix issues one at a time.

## A Consistent Error Response Format

Whatever you choose, be consistent. Here's a JSON format that works well:

```json
{
  "error": "validation failed",
  "details": [
    "title: must not be empty",
    "duration_seconds: must be positive"
  ]
}
```

For errors without detail:

```json
{
  "error": "not found"
}
```

For database/server errors (never expose internal details):

```json
{
  "error": "a database error occurred"
}
```

## A Type Alias for Handler Results

Add a type alias in `error.rs` to save typing:

```rust
pub type ApiResult<T> = Result<T, AppError>;
```

Then handlers read `ApiResult<Json<Song>>` instead of `Result<Json<Song>, AppError>`. Same thing, easier to scan.

## Checking Foreign Key Violations

When a client creates a song for an album that doesn't exist, the database will return a constraint error. You can catch that and return a clear 400 instead of a generic 500:

```rust
async fn create_song(
    State(state): State<AppState>,
    Json(body): Json<CreateSongRequest>,
) -> ApiResult<(StatusCode, Json<Song>)> {
    let validated = body.validate()?;

    let result = sqlx::query!(
        "INSERT INTO songs (album_id, title) VALUES (?, ?)",
        validated.album_id,
        validated.title,
    )
    .execute(&state.db)
    .await;

    match result {
        Ok(r) => { /* success path */ }
        Err(sqlx::Error::Database(db_err)) if db_err.message().contains("FOREIGN KEY") => {
            return Err(AppError::BadRequest(
                format!("album {} does not exist", validated.album_id)
            ));
        }
        Err(e) => return Err(AppError::Database(e)),
    }
    // ...
}
```

## How It Breaks

- Returning different error shapes from different endpoints: clients can't handle it reliably. Use one error response format everywhere.
- Exposing internal errors to clients: "query failed: column 'xyz' doesn't exist" leaks your schema. Map database errors to generic "internal server error" in production.
- Not validating at the boundary: accepting any string for email means garbage in your database. Validate once, early, at the HTTP layer.
- Status code misuse: 200 OK for "record not found" (should be 404), 500 for validation errors (should be 400), 200 for created records (should be 201).

## Common Mistakes

**Logging errors in IntoResponse AND in the handler**: Pick one place. Log in `IntoResponse` so the log always happens, regardless of which handler produced the error. Don't log twice.

**Exposing raw sqlx errors to clients**: A `sqlx::Error` message often contains table names, column names, and SQL snippets — all useful to an attacker. Always map database errors to a generic message in `IntoResponse`.

**Using a single generic "bad request" error for everything**: Return `AppError::Validation(vec![...])` with a list of specific field errors instead of a single vague "invalid input." Users and API clients need to know what specifically is wrong.

**Not trimming strings before validation**: `"  "` (whitespace only) will pass an `is_empty()` check. Always call `.trim()` first, then check.

**Forgetting that `?` on `Option` requires `From<NoneError>` or `.ok_or()`**: `option?` doesn't compile outside of functions that return `Option`. Use `.ok_or(AppError::NotFound)?` to convert `None` into your error type.
