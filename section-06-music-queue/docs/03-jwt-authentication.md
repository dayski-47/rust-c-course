# 03 — JWT Authentication 🟡

HTTP is stateless, which creates a problem: after a user logs in, how does the server know who they are on the next request? The classic answer is sessions — the server stores a session record in a database and gives the client a cookie with a session ID. Every request, the server looks up the session.

JWT (JSON Web Token) takes a different approach. Instead of storing state on the server, you encode the user's identity into a token and sign it. The client includes the token in every request. The server verifies the signature and trusts the contents without any database lookup. This is called stateless authentication.

For `queuemaster` specifically, JWTs solve a WebSocket problem: you cannot easily set headers on a WebSocket connection from the browser. You pass the JWT as a query parameter or in the first message after connecting.

## What a JWT Looks Like

A JWT is three base64-encoded JSON objects joined by dots:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI0MiIsInVzZXJuYW1lIjoiYWxpY2UiLCJleHAiOjE3MTg4MDQwMDB9.abc123sig
```

Part 1 (header): `{"alg":"HS256"}` — the signing algorithm.
Part 2 (payload): `{"sub":"42","username":"alice","exp":1718804000}` — your claims.
Part 3 (signature): HMAC-SHA256 of `header.payload` using your secret key.

The signature is the key. Anyone can decode the header and payload — they are just base64, not encrypted. But without the secret key, no one can produce a valid signature. If a client tries to tamper with the payload (changing their user ID from 42 to 1), the signature check fails.

## The jsonwebtoken Crate

```toml
jsonwebtoken = "9"
serde = { version = "1", features = ["derive"] }
```

```rust
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims {
    pub sub: String,       // "subject" — conventionally the user ID as a string
    pub username: String,
    pub role: String,      // "user", "dj", "admin"
    pub exp: u64,          // expiry as Unix timestamp (required by JWT spec)
    pub iat: u64,          // issued-at timestamp
}

fn current_timestamp() -> u64 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs()
}

pub fn create_access_token(
    user_id: i64,
    username: &str,
    role: &str,
    secret: &str,
) -> jsonwebtoken::errors::Result<String> {
    let now = current_timestamp();
    let claims = Claims {
        sub: user_id.to_string(),
        username: username.to_string(),
        role: role.to_string(),
        exp: now + 15 * 60,  // 15 minutes
        iat: now,
    };
    encode(
        &Header::default(),          // HS256 by default
        &claims,
        &EncodingKey::from_secret(secret.as_bytes()),
    )
}

pub fn verify_token(token: &str, secret: &str) -> jsonwebtoken::errors::Result<Claims> {
    let token_data = decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret.as_bytes()),
        &Validation::default(),  // validates exp automatically
    )?;
    Ok(token_data.claims)
}
```

`Validation::default()` requires the `exp` claim and rejects expired tokens. It also validates the algorithm matches. You get a typed error back telling you exactly why verification failed: `ExpiredSignature`, `InvalidSignature`, `InvalidToken`, etc.

## Access Tokens and Refresh Tokens

Access tokens expire quickly — 15 minutes is common. This limits the damage if a token is stolen: after 15 minutes it's useless. But you don't want users to log in every 15 minutes.

Refresh tokens solve this. They live longer (days or weeks) and are stored in a secure HTTP-only cookie. When the access token expires, the client calls `/auth/refresh` with the refresh token. The server verifies it, issues a new access token, and rotates the refresh token.

You store refresh tokens in Redis so you can invalidate them (log out from all devices, revoke a specific session, etc.).

```rust
pub fn create_refresh_token(
    user_id: i64,
    secret: &str,
) -> jsonwebtoken::errors::Result<String> {
    let now = current_timestamp();
    let claims = Claims {
        sub: user_id.to_string(),
        username: String::new(),  // not needed in refresh token
        role: String::new(),
        exp: now + 7 * 24 * 3600, // 7 days
        iat: now,
    };
    encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(secret.as_bytes()),
    )
}

// Store refresh token in Redis; delete it on logout
async fn save_refresh_token(
    conn: &mut deadpool_redis::Connection,
    user_id: i64,
    token: &str,
) -> redis::RedisResult<()> {
    use redis::AsyncCommands;
    let key = format!("refresh_token:{}", user_id);
    // 7 days TTL matches token expiry
    conn.set_ex(key, token, 7 * 24 * 3600).await
}
```

## Axum Middleware: JWT Extractor

You want `Claims` to be available in any handler that requires authentication, without repeating the verification logic in every handler. In Axum, you do this by implementing `FromRequestParts`.

```rust
use axum::{
    async_trait,
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

pub struct AuthUser(pub Claims);

#[async_trait]
impl<S: Send + Sync> FromRequestParts<S> for AuthUser {
    type Rejection = (StatusCode, String);

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        // Extract the Authorization header
        let auth_header = parts
            .headers
            .get("Authorization")
            .and_then(|v| v.to_str().ok())
            .ok_or((StatusCode::UNAUTHORIZED, "Missing Authorization header".into()))?;

        // Expect "Bearer <token>"
        let token = auth_header
            .strip_prefix("Bearer ")
            .ok_or((StatusCode::UNAUTHORIZED, "Invalid Authorization format".into()))?;

        // Pull the JWT secret from app state
        // (you'll need to extract it from `state` — requires a concrete State type)
        let secret = "your-secret-here"; // replace with state.jwt_secret

        let claims = verify_token(token, secret)
            .map_err(|e| (StatusCode::UNAUTHORIZED, format!("Invalid token: {}", e)))?;

        Ok(AuthUser(claims))
    }
}

// Now any handler can request the authenticated user:
async fn protected_handler(AuthUser(claims): AuthUser) -> String {
    format!("Hello, {}!", claims.username)
}
```

## Password Hashing

Never store plaintext passwords. Use `argon2`, which is the current recommendation for password hashing. It's intentionally slow and memory-hard, making brute-force attacks impractical.

```toml
argon2 = "0.5"
```

```rust
use argon2::{
    password_hash::{rand_core::OsRng, PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};

pub fn hash_password(password: &str) -> anyhow::Result<String> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    let hash = argon2
        .hash_password(password.as_bytes(), &salt)
        .map_err(|e| anyhow::anyhow!("Hash failed: {}", e))?;
    Ok(hash.to_string()) // PHC format: $argon2id$v=19$m=...
}

pub fn verify_password(password: &str, hash: &str) -> anyhow::Result<bool> {
    let parsed_hash = PasswordHash::new(hash)
        .map_err(|e| anyhow::anyhow!("Invalid hash: {}", e))?;
    Ok(Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok())
}
```

## Registration and Login Flow

```rust
use axum::{extract::State, Json};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
pub struct RegisterRequest {
    pub username: String,
    pub password: String,
}

#[derive(Serialize)]
pub struct AuthResponse {
    pub access_token: String,
}

pub async fn register(
    State(state): State<AppState>,
    Json(body): Json<RegisterRequest>,
) -> Result<Json<AuthResponse>, (StatusCode, String)> {
    // Check if username already exists
    let existing = sqlx::query!("SELECT id FROM users WHERE username = $1", body.username)
        .fetch_optional(&state.db)
        .await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    if existing.is_some() {
        return Err((StatusCode::CONFLICT, "Username already taken".into()));
    }

    // Hash password and store
    let hash = hash_password(&body.password)
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    let user = sqlx::query!(
        "INSERT INTO users (username, password_hash) VALUES ($1, $2) RETURNING id",
        body.username,
        hash,
    )
    .fetch_one(&state.db)
    .await
    .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    // Issue access token
    let token = create_access_token(user.id, &body.username, "user", &state.jwt_secret)
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    Ok(Json(AuthResponse { access_token: token }))
}

#[derive(Deserialize)]
pub struct LoginRequest {
    pub username: String,
    pub password: String,
}

pub async fn login(
    State(state): State<AppState>,
    Json(body): Json<LoginRequest>,
) -> Result<Json<AuthResponse>, (StatusCode, String)> {
    let user = sqlx::query!(
        "SELECT id, password_hash, role FROM users WHERE username = $1",
        body.username
    )
    .fetch_optional(&state.db)
    .await
    .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?
    .ok_or((StatusCode::UNAUTHORIZED, "Invalid credentials".into()))?;

    let valid = verify_password(&body.password, &user.password_hash)
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    if !valid {
        return Err((StatusCode::UNAUTHORIZED, "Invalid credentials".into()));
    }

    let token = create_access_token(user.id, &body.username, &user.role, &state.jwt_secret)
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    Ok(Json(AuthResponse { access_token: token }))
}
```

Note that login returns the same error message whether the username doesn't exist or the password is wrong. This is intentional — you don't want to tell attackers which usernames are valid.

## Security Considerations

- **HTTPS only.** JWTs are signed, not encrypted. Anyone who intercepts the token can read the claims. Always run behind TLS in production.
- **Short access token expiry.** 15 minutes is a good default. An hour is too long. Stolen tokens live until they expire.
- **Don't put sensitive data in the payload.** The payload is readable by anyone. Don't include passwords, SSNs, or payment info.
- **Rotate refresh tokens on use.** When a client refreshes, issue a new refresh token and invalidate the old one. If you see the old token used again, someone is replaying it — log them out everywhere.
- **Store JWT secret in environment variables.** Never hardcode it. Generate at least 32 random bytes: `openssl rand -base64 32`.

## How It Breaks

- Using a weak or exposed secret key: any JWT signed with a known key can be forged. Use a long random key, store it as an environment variable, never commit it.
- Not checking expiry: if you decode a JWT but don't check the `exp` claim, expired tokens work forever.
- JWT is not encrypted — it's only signed. The payload is base64-encoded and readable by anyone. Never put sensitive data (passwords, financial data) in JWT claims.
- Refresh token reuse: if you issue a new refresh token but don't invalidate the old one, a stolen refresh token can be used forever.
- Clock skew: JWT expiry is a Unix timestamp. If the server clock is wrong, tokens expire at the wrong time.

## Common Mistakes

**Using a weak or short secret.** The security of HMAC-SHA256 tokens depends entirely on the secret being unguessable. A short secret (less than 32 bytes) or a predictable string ("secret", "jwt_secret") makes the token worthless.

**Not checking the algorithm in the token header.** Some early JWT libraries had a vulnerability where an attacker could change the algorithm to "none" and bypass signature verification. The `jsonwebtoken` crate's `Validation::default()` requires HS256, which prevents this. Don't set `insecure_disable_signature_validation: true` in production.

**Storing the access token in localStorage.** Tokens in localStorage are accessible to JavaScript and vulnerable to XSS attacks. For web clients, store in a secure, HTTP-only cookie. For mobile and server-to-server API clients, localStorage or secure storage equivalents are fine.

**Forgetting to revoke refresh tokens on logout.** If you only have the client delete the token locally, stolen tokens remain valid until expiry. Store refresh tokens in Redis and delete the record on logout.
