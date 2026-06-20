# Doc 05 - Middleware and Layers 🟡

Middleware is code that runs before or after your handlers - across every request - without duplicating logic in every handler. Logging every incoming request, adding CORS headers, killing requests that take too long - all of this belongs in middleware.

Axum uses Tower for middleware. You've seen Tower mentioned earlier in the course. Here's the practical side.

## What Middleware Is

Think of it like a stack of wrappers around your handlers:

```
Request → [Timeout] → [Logger] → [CORS] → [Handler] → [CORS] → [Logger] → [Timeout] → Response
```

Each middleware layer can:
- Inspect or modify the request before passing it down
- Inspect or modify the response coming back up
- Short-circuit and return a response directly (authentication failure, timeout)

In Axum, you add middleware with `.layer()` on the router.

## tower-http: Pre-Built Middleware

The `tower-http` crate ships middleware you'll use on nearly every project. Add the features you need to `Cargo.toml`:

```toml
tower-http = { version = "0.5", features = ["cors", "trace", "compression-full", "timeout"] }
```

## Logging with TraceLayer

Structured request/response logging is the first middleware you should add. `TraceLayer` logs every request and its outcome:

```rust
use tower_http::trace::TraceLayer;
use tracing_subscriber::EnvFilter;

// In main, set up the tracing subscriber first
tracing_subscriber::fmt()
    .with_env_filter(EnvFilter::from_default_env())
    .init();

// Add the layer to your router
let app = Router::new()
    .route("/songs", get(list_songs))
    .layer(TraceLayer::new_for_http());
```

Now every request logs something like:
```
INFO  request{method=GET path=/songs}: started processing request
INFO  request{method=GET path=/songs}: finished processing request status=200 latency=2ms
```

Control the verbosity with the `RUST_LOG` environment variable:
```bash
RUST_LOG=info cargo run         # Info level - requests and important events
RUST_LOG=debug cargo run        # Debug - much more verbose
RUST_LOG=tower_http=debug cargo run  # Debug only for HTTP layer
```

## CORS: Cross-Origin Resource Sharing

If a browser frontend running on `localhost:5173` calls your API at `localhost:3000`, browsers block the request by default (different ports = different origin). CORS headers tell the browser it's allowed:

```rust
use tower_http::cors::{CorsLayer, Any};
use axum::http::Method;

let cors = CorsLayer::new()
    .allow_origin(Any)         // Allow any origin (development)
    .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
    .allow_headers(Any);

let app = Router::new()
    .route("/songs", get(list_songs))
    .layer(cors);
```

For production, restrict `allow_origin` to your actual frontend domain:

```rust
use axum::http::HeaderValue;

let cors = CorsLayer::new()
    .allow_origin("https://myapp.com".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
    .allow_headers(Any);
```

`allow_origin(Any)` is fine for development and public APIs, but you usually want to lock it down in production.

## Timeout: Kill Slow Requests

Requests that hang indefinitely tie up a thread and eventually exhaust your connection pool. Add a timeout to kill requests that take too long:

```rust
use tower_http::timeout::TimeoutLayer;
use std::time::Duration;

let app = Router::new()
    .route("/songs", get(list_songs))
    .layer(TimeoutLayer::new(Duration::from_secs(10)));
```

Any request that doesn't complete within 10 seconds gets terminated with a 408 Request Timeout response. Handlers don't need to know this exists - they just get cancelled if they run too long.

## Compression

Response bodies can be large (hundreds of songs in one request). Compression shrinks them before sending:

```rust
use tower_http::compression::CompressionLayer;

let app = Router::new()
    .route("/songs", get(list_songs))
    .layer(CompressionLayer::new());
```

`CompressionLayer` automatically picks the best algorithm (gzip, brotli, zstd) based on what the client's `Accept-Encoding` header says it supports. Clients that don't request compression get uncompressed responses.

## Adding Multiple Layers

Add layers to the router in a chain. The order matters - layers are applied from bottom to top as the request flows in, and top to bottom as the response flows out:

```rust
use tower::ServiceBuilder;

let app = Router::new()
    .route("/songs",    get(list_songs).post(create_song))
    .route("/songs/:id", get(get_song).put(update_song).delete(delete_song))
    .route("/artists",  get(list_artists).post(create_artist))
    .with_state(state)
    .layer(
        ServiceBuilder::new()
            .layer(TraceLayer::new_for_http())        // Outermost: logs everything
            .layer(TimeoutLayer::new(Duration::from_secs(30)))
            .layer(CompressionLayer::new())
            .layer(cors)
    );
```

`ServiceBuilder` lets you chain layers cleanly. The first `.layer()` in the builder is the outermost middleware - it sees the request first and the response last.

A practical ordering guideline:

```
1. TraceLayer        - must be outermost to log everything including timeout errors
2. TimeoutLayer      - kill slow requests before they hit downstream layers
3. CompressionLayer  - compress the outgoing response
4. CorsLayer         - add CORS headers
5. Your handlers
```

## Writing a Simple Custom Middleware Function

For straightforward cross-cutting concerns that don't need the full Tower `Service` trait, Axum supports middleware as async functions:

```rust
use axum::{
    extract::Request,
    middleware::{self, Next},
    response::Response,
    http::StatusCode,
};

// A middleware that adds a custom header to every response
async fn add_version_header(req: Request, next: Next) -> Response {
    let mut response = next.run(req).await;
    response.headers_mut().insert(
        "X-Api-Version",
        "1.0".parse().unwrap(),
    );
    response
}

// Add it to a specific route group
let app = Router::new()
    .route("/songs", get(list_songs))
    .route_layer(middleware::from_fn(add_version_header));
```

The `next.run(req)` call passes the request to the next middleware or handler. You get the response back and can modify it before returning it.

## Authentication Middleware 🔴

A real-world use case: check the `Authorization` header and reject unauthorized requests. Here's the shape without the full JWT verification logic:

```rust
use axum::{
    extract::{Request, State},
    middleware::Next,
    response::Response,
    http::{StatusCode, header},
};

async fn require_auth(
    State(state): State<AppState>,
    req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    // Get the Authorization header
    let auth_header = req
        .headers()
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "));

    let token = match auth_header {
        Some(t) => t,
        None => return Err(StatusCode::UNAUTHORIZED),
    };

    // Verify the token (your actual verification logic here)
    if !is_valid_token(token, &state) {
        return Err(StatusCode::UNAUTHORIZED);
    }

    Ok(next.run(req).await)
}

// Apply only to specific routes that need auth
let protected = Router::new()
    .route("/admin/songs", get(admin_list_songs))
    .route_layer(middleware::from_fn_with_state(state.clone(), require_auth));

let public = Router::new()
    .route("/songs", get(list_songs))
    .route("/health", get(health));

let app = Router::new()
    .merge(public)
    .merge(protected)
    .with_state(state);
```

`route_layer` applies middleware only to routes in that router. `layer` applies to the entire router including nested ones. Use `route_layer` when you want to protect specific routes without wrapping everything.

## Health Check Endpoint

Every service deployed with Docker or Kubernetes needs a health check. It should be fast and require no authentication:

```rust
use axum::http::StatusCode;
use serde::Serialize;

#[derive(Serialize)]
struct HealthResponse {
    status: &'static str,
    version: &'static str,
}

async fn health() -> (StatusCode, Json<HealthResponse>) {
    (StatusCode::OK, Json(HealthResponse {
        status: "ok",
        version: env!("CARGO_PKG_VERSION"),
    }))
}

// In your router, this is always public, never behind auth
let app = Router::new()
    .route("/health", get(health))
    // ... other routes
```

## Common Mistakes

**Adding `layer()` after `with_state()`**: Layer ordering can be surprising. Add `with_state()` before `.layer()` to avoid state not being visible inside middleware. When in doubt, use `ServiceBuilder` to keep the order explicit.

**Using `layer()` when you meant `route_layer()`**: `layer()` on a Router wraps everything including the 404 handler. `route_layer()` wraps only the routes defined in that specific Router. For auth middleware, you almost always want `route_layer()` so unauthenticated 404s still work correctly.

**CORS rejecting preflight requests**: Browsers send an OPTIONS preflight before POST/PUT/DELETE. If you forget to add `Method::OPTIONS` to `allow_methods`, preflight fails and the actual request is never sent. `CorsLayer` handles this automatically, but only if you haven't over-restricted it.

**Timeout set too short for large uploads**: File uploads or bulk operations take longer than simple queries. If you set a global 5-second timeout, large uploads will fail. Consider setting different timeouts per route group.

**Logging sensitive data in TraceLayer**: By default, `TraceLayer` doesn't log request bodies (which could contain passwords). Don't add body logging without thinking about what might be in those bodies.
