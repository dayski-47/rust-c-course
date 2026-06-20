# Doc 06 — Docker and Deployment 🟡

You have a working API. Now let's talk about getting it to run anywhere — your teammate's machine, a staging server, a cloud provider. Docker is the standard answer. It packages your binary and everything it needs into a single image that runs identically everywhere.

Rust has a specific trick that makes Docker images very small: you can cross-compile to a statically linked binary, which means the runtime image needs almost nothing.

## Multi-Stage Builds

Compiling Rust requires the full Rust toolchain — rustc, cargo, build dependencies. That's hundreds of megabytes. The final binary doesn't need any of that. A multi-stage Dockerfile uses one stage to compile and a second minimal stage to run:

```dockerfile
# Stage 1: Build
FROM rust:1.78-slim AS builder

WORKDIR /app

# Copy manifest files first (cache layer trick — explained below)
COPY Cargo.toml Cargo.lock ./

# Copy source
COPY src ./src
COPY migrations ./migrations

# Build in release mode
RUN cargo build --release

# Stage 2: Run
FROM debian:bookworm-slim AS runtime

WORKDIR /app

# Install runtime dependencies (CA certificates for HTTPS, SQLite)
RUN apt-get update && apt-get install -y \
    ca-certificates \
    libsqlite3-0 \
    && rm -rf /var/lib/apt/lists/*

# Copy only the compiled binary from the build stage
COPY --from=builder /app/target/release/melody-api .

# Copy migrations (needed at runtime for sqlx::migrate!)
COPY --from=builder /app/migrations ./migrations

# The port your API listens on
EXPOSE 3000

CMD ["./melody-api"]
```

The final image contains only `debian:bookworm-slim` + the binary + SQLite library. That's around 100-150 MB instead of 1.5+ GB for the full builder stage.

## The Dependency Caching Layer Trick

Cargo compiles all your dependencies before your code. Dependencies rarely change, but your source code changes constantly. If you just `COPY . .` and then `cargo build`, Docker recompiles all dependencies every time you change one line of your code.

The trick: copy only `Cargo.toml` and `Cargo.lock` first, then build a fake `main.rs` to force dependency compilation, then copy your real source:

```dockerfile
FROM rust:1.78-slim AS builder

WORKDIR /app

COPY Cargo.toml Cargo.lock ./

# Build dependencies only (this layer gets cached)
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm src/main.rs

# Now copy real source — only recompiles your crate, not dependencies
COPY src ./src
COPY migrations ./migrations

# Touch main.rs so cargo knows it changed
RUN touch src/main.rs
RUN cargo build --release
```

With this in place, rebuilding after changing your handler code takes 30 seconds instead of 5 minutes.

## .dockerignore

Without a `.dockerignore`, Docker copies your entire project including `target/` (gigabytes of compiled output) into the build context. Always create this file:

```
target/
.git/
.github/
*.md
.env
.env.local
music.db
```

This makes `docker build` much faster because Docker doesn't need to copy the `target/` directory.

## Environment Variables

Your API needs configuration that changes between environments. Hard-coding `music.db` is fine for development, not for production. Use environment variables:

```rust
// src/config.rs
use std::env;

pub struct Config {
    pub database_url: String,
    pub port: u16,
    pub host: String,
}

impl Config {
    pub fn from_env() -> Self {
        Config {
            database_url: env::var("DATABASE_URL")
                .unwrap_or_else(|_| "sqlite:music.db".into()),
            port: env::var("PORT")
                .ok()
                .and_then(|p| p.parse().ok())
                .unwrap_or(3000),
            host: env::var("HOST")
                .unwrap_or_else(|_| "0.0.0.0".into()),
        }
    }
}
```

In `main.rs`:

```rust
use dotenvy::dotenv;

#[tokio::main]
async fn main() {
    dotenv().ok(); // Load .env file if present (development only)

    let config = Config::from_env();
    let pool = SqlitePool::connect(&config.database_url).await.unwrap();
    // ...
    let listener = TcpListener::bind(
        format!("{}:{}", config.host, config.port)
    ).await.unwrap();
}
```

`dotenvy::dotenv().ok()` — the `.ok()` ignores the error if `.env` doesn't exist. In production containers you won't have a `.env` file; you'll set variables directly. In development, the `.env` file saves you from exporting them manually.

## Docker Compose for Development

Running just `docker run` works for the API alone. For development you usually want the API to restart automatically when you change code. Docker Compose orchestrates this:

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: sqlite:/data/music.db
      RUST_LOG: info
    volumes:
      - db_data:/data   # Persistent SQLite database file
    restart: unless-stopped

volumes:
  db_data:
```

```bash
# Build and start
docker compose up --build

# Start in background
docker compose up -d --build

# View logs
docker compose logs -f

# Stop
docker compose down
```

The `volumes:` section mounts a named volume at `/data` inside the container. Your SQLite database file lives there and persists between container restarts. Without this, you lose all data every time the container restarts.

## The .env File for Development

Keep your local development config in `.env`. Don't commit it to git:

```bash
# .env
DATABASE_URL=sqlite:music.db
PORT=3000
HOST=0.0.0.0
RUST_LOG=debug
```

Add `.env` to `.gitignore`. For other developers on the project, provide `.env.example` with the same keys but no real values:

```bash
# .env.example
DATABASE_URL=sqlite:music.db
PORT=3000
HOST=0.0.0.0
RUST_LOG=info
```

## Graceful Shutdown 🔴

When Docker stops a container, it sends SIGTERM to your process. If you don't handle it, Docker waits 10 seconds and then sends SIGKILL — potentially killing requests mid-flight. Handle SIGTERM properly:

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    // ... setup pool, state, router ...

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();

    println!("Listening on port 3000");

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();

    println!("Server shut down cleanly");
}

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }

    println!("Shutdown signal received, finishing in-flight requests...");
}
```

`axum::serve().with_graceful_shutdown(signal)` tells Axum to stop accepting new connections when the signal fires, but finish processing all in-flight requests before exiting. Kubernetes and Docker both respect this — they send SIGTERM and wait for the process to exit cleanly.

## Build and Run Commands

```bash
# Build the Docker image
docker build -t melody-api .

# Run the container
docker run -p 3000:3000 \
  -e DATABASE_URL=sqlite:/data/music.db \
  -e RUST_LOG=info \
  -v melody_db:/data \
  melody-api

# Check the health endpoint
curl http://localhost:3000/health

# Build for production (will be smaller with release optimizations already set)
docker build -t melody-api:latest .
```

## Health Check in Docker

Docker can automatically restart unhealthy containers. Add a health check to your Dockerfile:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

This tells Docker to check `GET /health` every 30 seconds. If it fails 3 times in a row, the container is marked unhealthy. Combined with `restart: unless-stopped` in docker-compose, Docker will restart it automatically.

## Common Mistakes

**Forgetting `COPY migrations ./migrations`**: The `sqlx::migrate!` macro embeds migration file paths at compile time, but it still reads the files at runtime. If the `migrations/` directory isn't in the container, startup crashes. Copy it in the Dockerfile.

**Using `127.0.0.1` as the bind address**: Inside a Docker container, `127.0.0.1` only accepts connections from within the container. Docker routes external traffic to `0.0.0.0`. If you bind to `127.0.0.1`, `docker run -p 3000:3000` will not work — no connections will reach your API.

**Not setting `RUST_LOG` in production**: Without it, `TraceLayer` produces no log output. Set at minimum `RUST_LOG=info` in your container environment.

**SQLite file in the container filesystem without a volume**: Every container restart wipes all data. Mount a volume for the database file, or use an external database.

**Building in debug mode in Docker**: The default `cargo build` produces a large, slow debug binary. Always use `cargo build --release` in your Dockerfile. Debug builds are 10-20x slower and 3-5x larger.

**Not using `.dockerignore`**: Forgetting this causes Docker to copy the entire `target/` directory into the build context, which can be several gigabytes. The copy step alone can take minutes.
