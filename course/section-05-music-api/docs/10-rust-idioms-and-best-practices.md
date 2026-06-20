# Doc 10 - Rust Idioms and Best Practices

🟢 These are the patterns that separate working Rust from idiomatic Rust.

By now you've built the music API: routing, database, validation, middleware, deployment, build scripts, coverage. This doc steps back and names the patterns you've been using - the idioms and habits that experienced Rust developers apply without thinking. Understanding them explicitly helps you recognize them in unfamiliar codebases and apply them to new problems.

---

## Newtype Pattern: Give Every Concept Its Own Type

The music API uses newtype wrappers throughout:

```rust
pub struct TrackTitle(String);
pub struct TrackDuration { milliseconds: u32 }
pub struct ArtistName(String);
pub struct PageLimit(u32);
pub struct PageOffset(u64);
```

This looks verbose. Why not just use `String` and `u32`?

Because `String` can be empty, null, or 500,000 characters long. `TrackTitle` cannot. The type carries the validation. Passing `TrackTitle` to a function is a promise: "this was validated." Passing `String` is a question: "was this validated?"

The newtype also prevents logic errors that the compiler can't catch with raw primitives:

```rust
// Easy bug with raw types - swapped arguments
fn insert_track(title: String, artist: String) { /* ... */ }
insert_track(artist_name, track_title);  // compiles, wrong order

// Impossible with newtypes - compiler catches the swap
fn insert_track(title: TrackTitle, artist: ArtistName) { /* ... */ }
insert_track(artist_name, track_title);  // compile error: types don't match
```

---

## Builder Pattern: Constructing Complex Objects Step by Step

When a struct has many optional or interdependent fields, don't take them all as constructor arguments:

```rust
// Hard to use - 7 arguments, easy to confuse order
let query = TrackQuery::new(
    Some("jazz"),
    None,
    Some(1_800_000),
    None,
    Some(50),
    10,
    "created_at",
    true,
);

// Builder - self-documenting, order doesn't matter
let query = TrackQuery::builder()
    .genre("jazz")
    .max_duration_ms(1_800_000)
    .limit(50)
    .offset(10)
    .sort_by("created_at")
    .descending()
    .build()?;
```

The builder pattern is especially useful for HTTP clients, configuration structs, and test fixtures:

```rust
pub struct TrackQueryBuilder {
    genre: Option<Genre>,
    min_duration: Option<TrackDuration>,
    max_duration: Option<TrackDuration>,
    limit: u32,
    offset: u64,
}

impl TrackQueryBuilder {
    pub fn new() -> Self {
        TrackQueryBuilder {
            genre: None,
            min_duration: None,
            max_duration: None,
            limit: 20,
            offset: 0,
        }
    }

    pub fn genre(mut self, genre: Genre) -> Self {
        self.genre = Some(genre);
        self
    }

    pub fn limit(mut self, limit: u32) -> Self {
        self.limit = limit.min(100);  // enforce max here, not in caller
        self
    }

    pub fn offset(mut self, offset: u64) -> Self {
        self.offset = offset;
        self
    }

    pub fn build(self) -> TrackQuery {
        TrackQuery {
            genre: self.genre,
            min_duration: self.min_duration,
            max_duration: self.max_duration,
            limit: PageLimit(self.limit),
            offset: PageOffset(self.offset),
        }
    }
}

impl Default for TrackQueryBuilder {
    fn default() -> Self { Self::new() }
}
```

Note the method chaining: each method takes `self` and returns `Self`. This lets you write `builder.genre(x).limit(50).build()` without intermediate variables.

---

## Impl Trait in Return Position

When you want to return something that implements a trait without naming the exact type, use `impl Trait`:

```rust
// Without impl Trait - must name the exact type (not always possible)
fn routes() -> axum::Router {
    axum::Router::new()
        .route("/tracks", get(list_tracks))
        .route("/tracks/:id", get(get_track).put(update_track))
}

// With impl Trait - return "something that is a Future"
fn fetch_tracks(db: &SqlitePool) -> impl Future<Output = Vec<Track>> + '_ {
    async move {
        sqlx::query_as!(Track, "SELECT * FROM tracks")
            .fetch_all(db)
            .await
            .unwrap_or_default()
    }
}

// In function parameters - accept "anything that implements Iterator<Item = Track>"
fn process_tracks<'a>(tracks: impl Iterator<Item = &'a Track>) -> Summary {
    let total_duration: u32 = tracks
        .map(|t| t.duration.milliseconds())
        .sum();
    Summary { total_duration }
}
```

`impl Trait` in parameters is syntactic sugar for a generic (`<T: Trait>`) - use either form. In return position, `impl Trait` hides the concrete type from callers, which is useful for returning closures and complex iterators.

---

## The `?` Operator and the Error Propagation Pattern

The `?` operator has a single job: if the result is `Err`, convert the error to the function's return error type (via `From`) and return early. Otherwise, unwrap the `Ok` value.

```rust
// Without ?:
fn create_track(db: &SqlitePool, body: CreateTrackBody) -> Result<Track, AppError> {
    let validated = match ValidatedTrack::try_from(body) {
        Ok(v) => v,
        Err(e) => return Err(AppError::from(e)),
    };
    
    let id = match sqlx::query!("INSERT INTO tracks ...").execute(db).await {
        Ok(r) => r.last_insert_rowid(),
        Err(e) => return Err(AppError::from(e)),
    };
    
    // ...
}

// With ?:
async fn create_track(db: &SqlitePool, body: CreateTrackBody) -> Result<Track, AppError> {
    let validated = ValidatedTrack::try_from(body)?;        // ValidationError → AppError
    let id = sqlx::query!("...").execute(db).await?;       // sqlx::Error → AppError
    // ...
}
```

For `?` to work across different error types, implement `From<specific_error> for AppError`. The `thiserror` crate's `#[from]` attribute does this automatically:

```rust
#[derive(thiserror::Error, Debug)]
pub enum AppError {
    #[error("validation error: {0}")]
    Validation(#[from] ValidationError),  // From<ValidationError> for AppError
    
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),        // From<sqlx::Error> for AppError
    
    #[error("not found")]
    NotFound,
}
```

This pattern (thiserror + `#[from]`) is the industry standard for applications. For libraries where you need stable API compatibility, consider keeping error types more manual.

---

## Prefer `iter()` Chains Over Mutable Loops for Data Pipelines

When transforming data, iterator chains express intent more clearly than mutable loops:

```rust
// Mutable loop - intent is buried in mutation details
fn summarize_tracks(tracks: &[Track]) -> TrackStats {
    let mut total_duration = 0u64;
    let mut count = 0usize;
    let mut genres: HashMap<String, usize> = HashMap::new();
    
    for track in tracks {
        total_duration += track.duration.milliseconds() as u64;
        count += 1;
        *genres.entry(track.genre.clone().unwrap_or_default()).or_insert(0) += 1;
    }
    
    TrackStats { total_duration, count, genre_distribution: genres }
}

// Iterator chain - transformations are named and linear
fn summarize_tracks(tracks: &[Track]) -> TrackStats {
    let total_duration: u64 = tracks.iter()
        .map(|t| t.duration.milliseconds() as u64)
        .sum();
    
    let count = tracks.len();
    
    let genre_distribution: HashMap<String, usize> = tracks.iter()
        .map(|t| t.genre.as_deref().unwrap_or("unknown"))
        .fold(HashMap::new(), |mut acc, genre| {
            *acc.entry(genre.to_string()).or_insert(0) += 1;
            acc
        });
    
    TrackStats { total_duration, count, genre_distribution }
}
```

The iterator chain version compiles to the same assembly. The advantage is readability: each `.map()`, `.filter()`, `.sum()` names exactly what it's doing.

Use a loop when:
- You're building output that grows (collecting multiple results per input item)
- You need early returns (`break` / `return`) inside the iteration
- The transformation involves side effects that need sequencing

---

## Use `Option` Combinators to Avoid Nesting

Option combinators flatten nested match/if let chains:

```rust
// Nested if-let - hard to read at depth
fn get_genre_display(track: &Track) -> String {
    if let Some(genre) = &track.genre {
        if let Some(display) = GENRE_NAMES.get(genre) {
            display.to_string()
        } else {
            genre.to_string()
        }
    } else {
        "Unknown".to_string()
    }
}

// Option combinators - linear flow
fn get_genre_display(track: &Track) -> String {
    track.genre.as_ref()
        .map(|g| GENRE_NAMES.get(g).copied().unwrap_or(g.as_str()))
        .unwrap_or("Unknown")
        .to_string()
}
```

Common combinators and when to use each:

```rust
let opt: Option<String> = Some("jazz".to_string());

// .map(f)        - transform the value if Some
opt.map(|s| s.to_uppercase())          // Some("JAZZ")

// .and_then(f)   - flatMap: apply f which returns Option (avoids Some(Some(...)))
opt.and_then(|s| lookup_genre(&s))     // None if lookup returns None

// .unwrap_or(x)  - default value if None
opt.unwrap_or_else(|| "unknown".to_string())  // "unknown" if None

// .filter(pred)  - None if predicate fails
opt.filter(|s| !s.is_empty())          // None if s is empty

// .ok_or(err)    - convert Option to Result
opt.ok_or(AppError::NotFound)?         // propagate NotFound if None

// .zip(other)    - combine two Options - None if either is None
let a: Option<i32> = Some(1);
let b: Option<i32> = Some(2);
a.zip(b)                               // Some((1, 2))
```

---

## Prefer Early Returns to Reduce Nesting

Deep nesting makes code hard to read. Early returns flatten it:

```rust
// Deep nesting - hard to follow
async fn get_track_with_lyrics(
    db: &SqlitePool,
    lyrics_api: &LyricsClient,
    id: i64,
) -> Result<TrackWithLyrics, AppError> {
    if let Some(track) = db.get_track(id).await? {
        if let Some(lyrics) = lyrics_api.get_lyrics(&track.artist, &track.title).await? {
            Ok(TrackWithLyrics { track, lyrics: Some(lyrics) })
        } else {
            Ok(TrackWithLyrics { track, lyrics: None })
        }
    } else {
        Err(AppError::NotFound)
    }
}

// Early return - flat and readable
async fn get_track_with_lyrics(
    db: &SqlitePool,
    lyrics_api: &LyricsClient,
    id: i64,
) -> Result<TrackWithLyrics, AppError> {
    let track = db.get_track(id).await?
        .ok_or(AppError::NotFound)?;  // early return if not found
    
    // After the guard, we know `track` exists
    let lyrics = lyrics_api.get_lyrics(&track.artist, &track.title).await?;
    
    Ok(TrackWithLyrics { track, lyrics })
}
```

The pattern: validate your prerequisites with `?` and `ok_or()` at the top of the function. The rest of the function can assume they succeeded.

---

## `Clone` Mindfully, Move When Possible

Cloning is safe and correct, but can be expensive. Prefer moving ownership into functions when the caller doesn't need it afterward:

```rust
// Unnecessary clone - track is moved into the response, caller never uses it again
async fn create_and_return(db: &Db, track: Track) -> Json<Track> {
    db.insert(track.clone()).await;  // clone just to keep ownership
    Json(track)                     // then move
}

// Better - move into db first, construct response from the data you need
async fn create_and_return(db: &Db, body: CreateTrackBody) -> Result<Json<TrackResponse>, AppError> {
    let track = ValidatedTrack::try_from(body)?;
    let id = db.insert(&track).await?;  // pass reference
    
    Ok(Json(TrackResponse {
        id,
        title: track.title.as_str().to_string(),  // only clone what you need
        // ...
    }))
}
```

Common patterns where cloning is appropriate:
- Sharing data across threads via `Arc` (you clone the `Arc`, not the data)
- Test fixtures that need to be reused across multiple test cases
- Implementing `Clone` explicitly for API response types that serialize to JSON

---

## `#[derive]` Over Manual Implementations

The derive macros handle the boilerplate. Manual implementations are only needed when the derived behavior is wrong:

```rust
// Derive everything that can be derived - less code, harder to get wrong
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct TrackId(pub i64);

// Only implement manually when you need custom behavior
impl Display for TrackId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "track#{}", self.0)  // custom format - derive wouldn't know this
    }
}

impl FromStr for TrackId {
    type Err = &'static str;
    
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        s.strip_prefix("track#")
            .and_then(|n| n.parse().ok())
            .map(TrackId)
            .ok_or("invalid track ID format")
    }
}
```

Always derive: `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `Serialize`, `Deserialize` (when needed), `Default` (when zero/empty is a valid default).

---

## Test the Behavior, Not the Implementation

Tests should assert what a function does, not how it does it:

```rust
// Tests implementation details - fragile, breaks on refactoring
#[test]
fn test_track_has_three_fields() {
    let track = Track { id: 1, title: "test".to_string(), duration_ms: 1000 };
    // Asserting struct field existence - not useful
    assert!(track.title.len() > 0);
}

// Tests behavior - what actually matters
#[test]
fn search_by_genre_returns_only_matching_tracks() {
    let tracks = vec![
        make_track("Yesterday", Genre::Rock),
        make_track("Blue in Green", Genre::Jazz),
        make_track("Coltrane", Genre::Jazz),
    ];
    
    let results = search_by_genre(&tracks, Genre::Jazz);
    
    assert_eq!(results.len(), 2);
    assert!(results.iter().all(|t| t.genre == Some(Genre::Jazz)));
}

#[test]
fn empty_title_returns_validation_error() {
    let err = TrackTitle::try_from("".to_string()).unwrap_err();
    assert!(matches!(err, TitleError::Empty));
}
```

A good test fails when behavior breaks, not when you rename a variable or split a function into two.

---

## The Music API: Idioms in Context

Looking back at what you've built, the idioms compound:

- **Newtype** (`TrackTitle`, `TrackDuration`) - validation lives in the type
- **Parse, don't validate** - `TryFrom` at the HTTP boundary, `&ValidatedTrack` everywhere else
- **Builder** - `TrackQueryBuilder` for search parameters
- **`?` with `From`** - error handling flows through the entire call stack
- **Iterator chains** - `tracks.iter().map().filter().sum()` for summaries
- **Early returns** - `ok_or(NotFound)?` before operating on data
- **`impl Trait`** - function signatures that accept any `Into<String>` or return any `Future`
- **`#[derive]`** - serialization, debugging, and equality without boilerplate

These patterns interact. The newtype makes validation explicit. Parse-don't-validate delivers the newtype at the right layer. The `?` operator propagates validation errors cleanly. Iterator chains process the validated data without mutation. The result is code that's harder to misuse than to use correctly.

---

## Industry Practices Checklist

Before marking the music API project complete:

- [ ] Every endpoint validates input at the boundary and uses `ValidatedTrack` (not raw `String`) throughout
- [ ] Error types are defined with `thiserror` and all use `#[from]` for `?` compatibility
- [ ] No `unwrap()` in production paths (only in tests and build scripts)
- [ ] Iterator chains for data transformation, not mutable accumulators
- [ ] `build.rs` embeds git hash for deployed version tracking
- [ ] Coverage is measured and branch coverage is above 80%
- [ ] Clippy warnings are addressed: `cargo clippy --workspace -- -D warnings`
- [ ] No unused imports or dead code: `cargo check 2>&1 | grep "unused"`
- [ ] `cargo fmt --check` passes - consistent formatting

The last three items are enforced by CI automatically (section 5's docker deployment doc covers the CI setup). Adding `cargo clippy` and `cargo fmt --check` to CI means these checks can't be forgotten on a busy day.
