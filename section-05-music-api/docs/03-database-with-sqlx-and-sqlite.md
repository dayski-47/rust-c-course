# Doc 03 — Database with SQLx and SQLite 🟡

Every API needs persistent storage. You could write SQL queries as raw strings and hope they work — that's what most languages do. SQLx does something better: it checks your SQL queries at **compile time**. If you typo a column name, reference a table that doesn't exist, or pass the wrong type, the compiler tells you. Not a runtime crash, not a test failure — the compiler.

This is a big deal. Let's see how it works.

## Adding SQLx to Cargo.toml

```toml
[dependencies]
sqlx = { version = "0.7", features = ["sqlite", "runtime-tokio", "migrate"] }
tokio = { version = "1", features = ["full"] }
```

The `sqlite` feature enables SQLite support. `runtime-tokio` makes SQLx work with your Tokio async runtime. `migrate` enables the migration system you'll use to manage your schema.

## Setting Up a Connection Pool

You never hold a single database connection. You hold a **pool** — a collection of connections that handlers borrow when they need to run a query and return when they're done. This is how you serve many concurrent requests without opening a new database connection for each one.

```rust
use sqlx::SqlitePool;

async fn create_pool(database_url: &str) -> SqlitePool {
    SqlitePool::connect(database_url)
        .await
        .expect("Failed to connect to database")
}

// In your main function:
let pool = create_pool("sqlite:music.db").await;
// Pass pool into your AppState
let state = AppState { db: pool };
```

The database URL `sqlite:music.db` creates (or opens) a file called `music.db` in the current directory. For an in-memory database (great for tests), use `sqlite::memory:`.

SQLx needs the `DATABASE_URL` environment variable set at **compile time** so it can verify your queries against the real schema. Set it before building:

```bash
export DATABASE_URL=sqlite:music.db
cargo build
```

Or put it in a `.env` file and use the `dotenvy` crate to load it. You'll see this in the project setup.

## Migrations

Before you can query the database, the tables need to exist. SQLx has a migration system for this. Migrations are SQL files in a `migrations/` directory, numbered in order:

```
migrations/
├── 20240101000001_initial_schema.sql
└── 20240101000002_add_genres.sql
```

A migration file is just SQL:

```sql
-- migrations/20240101000001_initial_schema.sql
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

Run migrations in your main function before starting the server:

```rust
sqlx::migrate!("./migrations")
    .run(&pool)
    .await
    .expect("Failed to run migrations");
```

The `migrate!` macro embeds the migration files into your binary at compile time and runs any that haven't been applied yet. Safe to call on every startup — already-applied migrations are skipped.

## The query! Macro — Compile-Time SQL Checking 🔴

Here's the magic. The `sqlx::query!` macro sends your SQL to the database at **compile time** and verifies:
- The table exists
- The columns exist
- The parameter types match
- The result columns have the right types

```rust
async fn list_songs(pool: &SqlitePool) -> Vec<SongRow> {
    sqlx::query!("SELECT id, title, duration_seconds FROM songs")
        .fetch_all(pool)
        .await
        .unwrap()
        .into_iter()
        .map(|row| SongRow {
            id: row.id,
            title: row.title,
            duration_seconds: row.duration_seconds,
        })
        .collect()
}
```

The return type of `query!` is an anonymous struct with typed fields matching your SELECT columns. Try selecting a column that doesn't exist — it won't compile.

Parameters use `?` placeholders:

```rust
async fn get_song_by_id(pool: &SqlitePool, id: i64) -> Option<SongRow> {
    sqlx::query!("SELECT id, title, duration_seconds FROM songs WHERE id = ?", id)
        .fetch_optional(pool)
        .await
        .unwrap()
}
```

Use `?` for each parameter, in order. SQLx checks that the types match at compile time.

## query_as! — Mapping to Your Own Structs

The `query!` macro returns an anonymous type. More often you want to map directly to your own struct:

```rust
use sqlx::FromRow;

#[derive(Debug, Serialize, FromRow)]
struct Song {
    id: i64,
    title: String,
    duration_seconds: Option<i64>,
    track_number: Option<i64>,
    album_id: i64,
}

async fn list_songs(pool: &SqlitePool) -> Vec<Song> {
    sqlx::query_as!(Song,
        "SELECT id, title, duration_seconds, track_number, album_id FROM songs ORDER BY track_number"
    )
    .fetch_all(pool)
    .await
    .unwrap()
}

async fn get_song(pool: &SqlitePool, id: i64) -> Option<Song> {
    sqlx::query_as!(Song,
        "SELECT id, title, duration_seconds, track_number, album_id FROM songs WHERE id = ?",
        id
    )
    .fetch_optional(pool)
    .await
    .unwrap()
}
```

The field names in your struct must match the column names in your SELECT. If they don't match, use aliases in SQL: `SELECT id, duration_seconds AS duration FROM songs`.

## CRUD Operations

Let's build out all four operations for songs:

```rust
// CREATE
async fn create_song(pool: &SqlitePool, album_id: i64, title: &str, track_number: Option<i64>) -> i64 {
    let result = sqlx::query!(
        "INSERT INTO songs (album_id, title, track_number) VALUES (?, ?, ?)",
        album_id, title, track_number
    )
    .execute(pool)
    .await
    .unwrap();

    result.last_insert_rowid()
}

// READ (one)
async fn get_song(pool: &SqlitePool, id: i64) -> Option<Song> {
    sqlx::query_as!(Song,
        "SELECT id, title, duration_seconds, track_number, album_id FROM songs WHERE id = ?",
        id
    )
    .fetch_optional(pool)
    .await
    .unwrap()
}

// READ (many, with filter)
async fn list_songs_for_album(pool: &SqlitePool, album_id: i64) -> Vec<Song> {
    sqlx::query_as!(Song,
        "SELECT id, title, duration_seconds, track_number, album_id
         FROM songs WHERE album_id = ? ORDER BY track_number",
        album_id
    )
    .fetch_all(pool)
    .await
    .unwrap()
}

// UPDATE
async fn update_song(pool: &SqlitePool, id: i64, title: &str) -> bool {
    let result = sqlx::query!(
        "UPDATE songs SET title = ? WHERE id = ?",
        title, id
    )
    .execute(pool)
    .await
    .unwrap();

    result.rows_affected() > 0
}

// DELETE
async fn delete_song(pool: &SqlitePool, id: i64) -> bool {
    let result = sqlx::query!("DELETE FROM songs WHERE id = ?", id)
        .execute(pool)
        .await
        .unwrap();

    result.rows_affected() > 0
}
```

Note the return values:
- `execute()` returns `SqliteQueryResult` with `rows_affected()` and `last_insert_rowid()`
- `fetch_all()` returns `Vec<Row>`
- `fetch_optional()` returns `Option<Row>` — use this for single-item lookups, not `fetch_one()`, which panics if nothing is found
- `fetch_one()` returns `Row` or an error if zero or multiple rows

## Transactions 🔴

Some operations need to be atomic. If you're creating an album and then linking songs to it, you don't want a partial state where the album exists but the songs don't.

```rust
async fn create_album_with_songs(
    pool: &SqlitePool,
    artist_id: i64,
    album_title: &str,
    song_titles: &[String],
) -> Result<i64, sqlx::Error> {
    let mut tx = pool.begin().await?;

    // Insert the album
    let album_result = sqlx::query!(
        "INSERT INTO albums (artist_id, title) VALUES (?, ?)",
        artist_id, album_title
    )
    .execute(&mut *tx)
    .await?;

    let album_id = album_result.last_insert_rowid();

    // Insert each song
    for (i, title) in song_titles.iter().enumerate() {
        sqlx::query!(
            "INSERT INTO songs (album_id, title, track_number) VALUES (?, ?, ?)",
            album_id, title, (i + 1) as i64
        )
        .execute(&mut *tx)
        .await?;
    }

    tx.commit().await?;
    Ok(album_id)
}
```

If any step returns an error and you don't call `tx.commit()`, the transaction rolls back automatically when `tx` is dropped. The `?` operator propagates errors up, so if the album insert fails, the song inserts never run and everything is left clean.

Note the `&mut *tx` — transactions implement the same executor trait as pools, but you pass a mutable reference to them.

## Connection Pool Configuration

For production you may want to tune the pool:

```rust
use sqlx::sqlite::SqlitePoolOptions;

let pool = SqlitePoolOptions::new()
    .max_connections(10)
    .min_connections(2)
    .acquire_timeout(std::time::Duration::from_secs(3))
    .connect("sqlite:music.db")
    .await?;
```

`max_connections(10)` limits how many connections can be open simultaneously. `acquire_timeout` means if all connections are busy, wait up to 3 seconds before returning an error instead of waiting forever.

## Testing with an In-Memory Database

For tests, use `sqlite::memory:`. Each test gets its own clean database:

```rust
#[cfg(test)]
mod tests {
    use sqlx::SqlitePool;

    async fn setup_test_db() -> SqlitePool {
        let pool = SqlitePool::connect("sqlite::memory:").await.unwrap();
        sqlx::migrate!("./migrations").run(&pool).await.unwrap();
        pool
    }

    #[tokio::test]
    async fn test_create_and_get_song() {
        let pool = setup_test_db().await;

        let id = create_song(&pool, 1, "Test Song", None).await;
        let song = get_song(&pool, id).await.unwrap();

        assert_eq!(song.title, "Test Song");
    }
}
```

In-memory databases are fast and isolated. No need to clean up between tests — each test creates a fresh pool and a fresh schema.

## How It Breaks

- Connection pool exhaustion: if all pool connections are held (by slow queries or connection leaks), new requests block waiting. Set pool `max_connections` and set timeouts.
- SQLx compile-time checking requires `DATABASE_URL` at compile time. If you don't set it, the build fails. Use `sqlx::query()` (runtime-checked) during development, switch to `sqlx::query!()` (compile-time checked) when the schema is stable.
- Migrations not running on startup: your app connects but the tables don't exist. Always call `sqlx::migrate!().run(&pool)` on startup.
- N+1 query problem: fetching 100 albums, then fetching the artist for each album separately = 101 queries. Use JOINs or fetch all artists in one query.
- Transaction not committed: if you forget to call `tx.commit()`, the transaction rolls back automatically. No error, no data change. This is a very common silent bug.

## Common Mistakes

**Not setting `DATABASE_URL` before building**: The `query!` macro contacts your database at compile time. If `DATABASE_URL` isn't set, you get a build error. Set it in `.env` and load with `dotenvy`, or export it in your shell before running `cargo build`.

**Using `fetch_one()` for nullable lookups**: `fetch_one()` returns an error if no row is found. For "get by id" endpoints, use `fetch_optional()` instead, which returns `Option<Row>`. Then you can return 404 when `None`.

**Forgetting `?` in queries with parameters**: `sqlx::query!("SELECT ... WHERE id = ?", id)` — the `?` is the placeholder. Mixing this up with named placeholders (like `$1` in Postgres) or forgetting it entirely causes a compile or runtime error.

**Using `rows_affected()` to check existence**: When you update a row, check `rows_affected() > 0`. If it returns 0, the row didn't exist — return 404. Don't skip this check.

**Running migrations every startup without `sqlx::migrate!`**: Calling the migration function manually (instead of using the macro) means the files aren't embedded in the binary. The macro is the right way — it includes and checksums the migration files at compile time.
