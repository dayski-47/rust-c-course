# Doc 06 — Parse, Don't Validate: Typed Boundaries

🟡 This is a mindset shift — validation scattered everywhere vs. validation concentrated at the edge.

You've probably written code like this: a function that accepts a `String` or `&str`, checks whether it's valid, returns an error if not, and then proceeds to use it. Then later, another function does the same check. Then another. The same validation logic appears three, five, eight times across the codebase, each slightly different.

The result: bugs where one check is stricter than another, or where a new developer adds a path that skips the validation entirely. The data passes through your system with a vague "probably valid" status that no one can actually prove.

Parse, Don't Validate is the fix. **Validate exactly once at the boundary. Turn valid data into a type that carries the proof. Pass that type everywhere.** No re-checking. No "probably valid." Proven valid.

---

## The Problem: Shotgun Validation

In C, this pattern is pervasive:

```c
// C — the same validation in five different functions
int create_track(const char *title, int duration_ms, const char *artist) {
    if (title == NULL || strlen(title) == 0) return -1;
    if (duration_ms <= 0) return -1;
    if (artist == NULL) return -1;
    // ...
}

int update_track(int id, const char *title, int duration_ms) {
    if (id <= 0) return -1;
    if (title == NULL || strlen(title) == 0) return -1;  // duplicated
    if (duration_ms <= 0) return -1;                     // duplicated
    // ...
}

// Forgotten in one place:
void export_track(Track *track) {
    // FORGOT to check title is non-null — UB if track was created wrong
    printf("Track: %s", track->title);
}
```

This is shotgun validation: scattered, redundant, and missing in at least one place.

---

## The Solution: Parse at the Boundary

Turn valid data into a type that **can only exist if the data is valid**:

```rust
// Raw input — not yet validated
pub struct RawTrackRequest {
    pub title: String,
    pub duration_ms: i64,
    pub artist: String,
    pub genre: Option<String>,
}

// Validated — can only be constructed through TryFrom, which enforces invariants
#[derive(Debug, Clone)]
pub struct ValidatedTrack {
    pub title: TrackTitle,        // non-empty, <= 500 chars
    pub duration: TrackDuration,  // 1ms to 24 hours
    pub artist: ArtistName,       // non-empty, <= 200 chars
    pub genre: Option<Genre>,     // one of the known genres or None
}

// The boundary — validate exactly once here
impl TryFrom<RawTrackRequest> for ValidatedTrack {
    type Error = ValidationError;

    fn try_from(raw: RawTrackRequest) -> Result<Self, ValidationError> {
        let title = TrackTitle::try_from(raw.title)?;
        let duration = TrackDuration::try_from(raw.duration_ms)?;
        let artist = ArtistName::try_from(raw.artist)?;
        let genre = raw.genre.map(Genre::try_from).transpose()?;

        Ok(ValidatedTrack { title, duration, artist, genre })
    }
}
```

Any function that receives `&ValidatedTrack` **knows** the data is valid. No re-checking. The type is the proof.

---

## Building the Validated Types

Each field becomes its own validated newtype:

```rust
use std::convert::TryFrom;

// --- Track Title ---

#[derive(Debug, Clone)]
pub struct TrackTitle(String);

#[derive(Debug, thiserror::Error)]
pub enum TitleError {
    #[error("title cannot be empty")]
    Empty,
    #[error("title is too long: {0} chars (max 500)")]
    TooLong(usize),
    #[error("title contains invalid characters")]
    InvalidCharacters,
}

impl TryFrom<String> for TrackTitle {
    type Error = TitleError;

    fn try_from(s: String) -> Result<Self, TitleError> {
        let trimmed = s.trim();
        if trimmed.is_empty() {
            return Err(TitleError::Empty);
        }
        if trimmed.len() > 500 {
            return Err(TitleError::TooLong(trimmed.len()));
        }
        // Allow only printable Unicode (no control characters)
        if trimmed.chars().any(|c| c.is_control()) {
            return Err(TitleError::InvalidCharacters);
        }
        Ok(TrackTitle(trimmed.to_string()))
    }
}

impl TrackTitle {
    // Expose a &str view — caller cannot bypass the validation
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

// --- Track Duration ---

#[derive(Debug, Clone, Copy)]
pub struct TrackDuration {
    milliseconds: u32,  // private — callers can't set invalid values
}

#[derive(Debug, thiserror::Error)]
pub enum DurationError {
    #[error("duration must be positive, got {0}ms")]
    NonPositive(i64),
    #[error("duration is unreasonably long: {0}ms (max 24 hours)")]
    TooLong(i64),
}

const MAX_DURATION_MS: i64 = 24 * 60 * 60 * 1000;  // 24 hours

impl TryFrom<i64> for TrackDuration {
    type Error = DurationError;

    fn try_from(ms: i64) -> Result<Self, DurationError> {
        if ms <= 0 {
            return Err(DurationError::NonPositive(ms));
        }
        if ms > MAX_DURATION_MS {
            return Err(DurationError::TooLong(ms));
        }
        Ok(TrackDuration { milliseconds: ms as u32 })
    }
}

impl TrackDuration {
    pub fn milliseconds(&self) -> u32 { self.milliseconds }
    pub fn seconds(&self) -> f32 { self.milliseconds as f32 / 1000.0 }
    pub fn as_display(&self) -> String {
        let total_secs = self.milliseconds / 1000;
        format!("{}:{:02}", total_secs / 60, total_secs % 60)
    }
}

// --- Genre (Enumerated) ---

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Genre {
    Rock,
    Jazz,
    Classical,
    Electronic,
    HipHop,
    Country,
    Pop,
}

#[derive(Debug, thiserror::Error)]
#[error("unknown genre: {0}")]
pub struct UnknownGenre(String);

impl TryFrom<String> for Genre {
    type Error = UnknownGenre;

    fn try_from(s: String) -> Result<Self, UnknownGenre> {
        match s.to_lowercase().as_str() {
            "rock"       => Ok(Genre::Rock),
            "jazz"       => Ok(Genre::Jazz),
            "classical"  => Ok(Genre::Classical),
            "electronic" => Ok(Genre::Electronic),
            "hip-hop" | "hiphop" | "hip_hop" => Ok(Genre::HipHop),
            "country"    => Ok(Genre::Country),
            "pop"        => Ok(Genre::Pop),
            _            => Err(UnknownGenre(s)),
        }
    }
}
```

Notice: the fields of `TrackTitle` and `TrackDuration` are private. You can't construct them without going through the validation. The type system enforces this — there's no escape hatch that lets you create a `TrackTitle("")`.

---

## The Axum Integration: Validate at the HTTP Boundary

In the music API, the boundary is the HTTP request handler. Validation happens at the very edge — before any business logic sees the data:

```rust
use axum::{extract::State, response::Json, http::StatusCode};
use serde::{Deserialize, Serialize};

// The raw JSON shape — what the client sends
#[derive(Deserialize)]
pub struct CreateTrackBody {
    pub title: String,
    pub duration_ms: i64,
    pub artist: String,
    pub genre: Option<String>,
}

// The validated response shape
#[derive(Serialize)]
pub struct TrackResponse {
    pub id: i64,
    pub title: String,
    pub duration_display: String,
    pub artist: String,
    pub genre: Option<String>,
}

pub async fn create_track(
    State(db): State<SqlitePool>,
    Json(body): Json<CreateTrackBody>,
) -> Result<(StatusCode, Json<TrackResponse>), AppError> {
    // ─── BOUNDARY: parse and validate ───
    let raw = RawTrackRequest {
        title: body.title,
        duration_ms: body.duration_ms,
        artist: body.artist,
        genre: body.genre,
    };
    
    let track = ValidatedTrack::try_from(raw)
        .map_err(AppError::Validation)?;
    // ─── From here, track is guaranteed valid ───
    
    // Business logic works with typed data — no validation needed
    let id = db_insert_track(&db, &track).await?;
    
    Ok((StatusCode::CREATED, Json(TrackResponse {
        id,
        title: track.title.as_str().to_string(),
        duration_display: track.duration.as_display(),
        artist: track.artist.as_str().to_string(),
        genre: track.genre.map(|g| format!("{g:?}").to_lowercase()),
    })))
}

// The business logic function — never validates, just operates on proven-valid data
async fn db_insert_track(db: &SqlitePool, track: &ValidatedTrack) -> Result<i64, sqlx::Error> {
    let id = sqlx::query!(
        "INSERT INTO tracks (title, duration_ms, artist, genre) VALUES (?, ?, ?, ?)",
        track.title.as_str(),
        track.duration.milliseconds(),
        track.artist.as_str(),
        track.genre.as_ref().map(|g| format!("{g:?}").to_lowercase()),
    )
    .execute(db)
    .await?
    .last_insert_rowid();
    
    Ok(id)
}
```

The validation layer and the business logic layer are cleanly separated. A new developer looking at `db_insert_track` doesn't need to wonder "is this data validated?" — the type signature answers it.

---

## Combining Multiple Validated Inputs

Validated types compose naturally. A struct that requires multiple validated fields is itself a proof that all fields are valid:

```rust
// A search query — multiple optional validated constraints
#[derive(Debug)]
pub struct TrackSearchQuery {
    pub title_contains: Option<String>,    // non-empty when present
    pub min_duration: Option<TrackDuration>,
    pub max_duration: Option<TrackDuration>,
    pub genre: Option<Genre>,
    pub limit: PageLimit,                  // 1-100
    pub offset: PageOffset,               // >= 0
}

#[derive(Debug, Clone, Copy)]
pub struct PageLimit(u32);  // 1-100

#[derive(Debug, Clone, Copy)]
pub struct PageOffset(u64);

impl TryFrom<u32> for PageLimit {
    type Error = &'static str;
    fn try_from(n: u32) -> Result<Self, &'static str> {
        match n {
            1..=100 => Ok(PageLimit(n)),
            0 => Err("limit must be at least 1"),
            _ => Err("limit cannot exceed 100"),
        }
    }
}

// When you receive a TrackSearchQuery, every constraint is independently valid:
async fn search_tracks(db: &SqlitePool, query: &TrackSearchQuery) -> Vec<Track> {
    // No range checks needed — PageLimit guarantees 1-100
    // No null checks needed — all Options are explicitly typed
    // No bounds checks on duration — TrackDuration guarantees valid range
    build_search_query(db, query).await
}
```

---

## The `?` Operator and Validation Errors

For validation to compose cleanly, all your validation error types need to implement `From<specific_error> for AppError`. This is the same pattern from section 2 — type conversions enabling `?`:

```rust
#[derive(Debug, thiserror::Error)]
pub enum ValidationError {
    #[error("invalid title: {0}")]
    Title(#[from] TitleError),
    
    #[error("invalid duration: {0}")]
    Duration(#[from] DurationError),
    
    #[error("invalid genre: {0}")]
    Genre(#[from] UnknownGenre),
}

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("validation error: {0}")]
    Validation(#[from] ValidationError),
    
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("not found")]
    NotFound,
}

// With these From impls, validation chains with ?:
fn validate_track_creation(body: CreateTrackBody) -> Result<ValidatedTrack, ValidationError> {
    Ok(ValidatedTrack {
        title: TrackTitle::try_from(body.title)?,     // TitleError → ValidationError
        duration: TrackDuration::try_from(body.duration_ms)?, // DurationError → ValidationError
        artist: ArtistName::try_from(body.artist)?,
        genre: body.genre.map(Genre::try_from).transpose()?, // UnknownGenre → ValidationError
    })
}
```

This chain of `?` makes the validation code linear and readable. Any failure propagates up immediately with the appropriate error type.

---

## When to Use This Pattern

| Situation | Use validated type? |
|-----------|:-------------------:|
| Data from HTTP request body | ✅ Always |
| Data from user CLI arguments | ✅ Always |
| Data from external config files | ✅ Always |
| Data from an untrusted database | ✅ On the way in |
| Passing data between internal functions | ❌ Usually not |
| Results of pure computations in your code | ❌ Not needed |

The key criterion: **was this data touched by something outside your control?** User input, HTTP requests, files, environment variables — all need validated boundaries. Internal function calls within your own code can trust the types already in play.

---

## How It Breaks

**Creating the raw struct directly in tests to bypass validation.**
```rust
// This temptation kills the pattern:
let track = ValidatedTrack {
    title: TrackTitle("".to_string()),  // COMPILE ERROR — title is private
    // ...
};
```
Good news: because the inner fields are private, the compiler prevents this. Tests must go through `TryFrom`. This is intentional — if your test can bypass validation, so can a bug.

**Using `String::new()` or `Default::derive` on validated types.**
If you `#[derive(Default)]` on `TrackTitle`, the default is `TrackTitle("")` — an empty title that bypasses validation. Never derive `Default` on validated types. If you need a "blank" state, use `Option<TrackTitle>` instead.

**Accepting `&str` and re-validating inside business logic.**
```rust
// Anti-pattern: business logic does its own validation
fn create_track_in_db(db: &Db, title: &str, duration_ms: i64) {
    if title.is_empty() { panic!("empty title"); }  // validation re-duplicated
    // ...
}
```
The fix: accept `&ValidatedTrack` (or the specific `&TrackTitle`). Let the type signature guarantee validity.

**Forgetting the `transpose()` for `Option<TryFrom<...>>`.**
```rust
// This doesn't compile — Option<Result<T, E>> is not Result<Option<T>, E>
let genre = raw.genre.map(Genre::try_from)?;  // WRONG

// This works:
let genre = raw.genre.map(Genre::try_from).transpose()?;  // OK
```
`Option::transpose()` converts `Option<Result<T, E>>` to `Result<Option<T>, E>`. It's the standard way to apply a fallible operation to an optional value and propagate the error.
