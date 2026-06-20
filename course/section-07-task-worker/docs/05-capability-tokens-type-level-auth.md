# Doc 05 — Capability Tokens and Type-Level Authorization

🟡 Runtime permission checks let bugs in. Compile-time capability tokens make them impossible.

In every production task-worker system there are operations that only certain callers should be allowed to perform. Cancelling someone else's job. Reprioritizing the high-priority queue. Purging all failed jobs. Accessing job payloads (which may contain PII).

In C — and in most Rust code written by people coming from C — these restrictions are runtime checks:

```c
int cancel_job(job_system_t *sys, job_id_t id, user_t *caller) {
    if (!caller->is_admin) {          // runtime check
        return -EPERM;
    }
    if (!job_exists(sys, id)) {       // runtime check
        return -ENOENT;
    }
    // ... do the cancellation ...
}
```

Every function that does something sensitive must repeat the check. Forget one — privilege escalation. The checks scatter throughout the codebase and are impossible to audit completely.

This doc shows how to replace runtime permission checks with compile-time **capability tokens**: zero-sized types that prove the caller has the required authority, enforced by the compiler at zero runtime cost.

---

## Zero-Sized Types as Proof

A capability token is a struct with no fields (or only `pub(crate)` private fields that prevent external construction). It exists only in the type system:

```rust
/// Proof that the holder has worker-management authority.
/// Zero bytes at runtime — exists only for the compiler.
pub struct WorkerAdminToken {
    _private: (),  // prevents construction outside this module
}

/// Proof that the holder has been authenticated for this specific job owner.
pub struct OwnerToken {
    pub job_owner_id: Uuid,  // the owner this token was issued for
    _private: (),
}
```

`WorkerAdminToken { _private: () }` compiles fine inside the module. From outside, the `_private` field is inaccessible — the compiler prevents forging the token.

---

## Issuing Tokens at Authentication

The only way to obtain a capability token is through a function that performs the actual check. That function runs once, at the authentication boundary:

```rust
use uuid::Uuid;

pub struct AuthService {
    db: sqlx::SqlitePool,
}

impl AuthService {
    /// Authenticate as a worker admin.
    /// Returns a token only on success. The token can then be used
    /// to call any admin-gated function without further checks.
    pub async fn authenticate_admin(
        &self,
        credentials: &AdminCredentials,
    ) -> Result<WorkerAdminToken, AuthError> {
        let record = sqlx::query!(
            "SELECT password_hash FROM admins WHERE username = ?",
            credentials.username
        )
        .fetch_optional(&self.db)
        .await?
        .ok_or(AuthError::InvalidCredentials)?;

        if !argon2::verify_encoded(&record.password_hash, credentials.password.as_bytes())? {
            return Err(AuthError::InvalidCredentials);
        }

        Ok(WorkerAdminToken { _private: () })
    }

    /// Issue an owner token for a specific user.
    /// The token carries the owner's ID so functions can check
    /// "are you the owner of this specific job?"
    pub async fn issue_owner_token(&self, user_id: Uuid) -> OwnerToken {
        // Precondition: caller has already authenticated the user
        OwnerToken { job_owner_id: user_id, _private: () }
    }
}
```

---

## Gating Functions on Tokens

Functions that require authority declare it in their signature. The compiler enforces the proof obligation:

```rust
pub struct JobQueue {
    redis: redis::Client,
    db: sqlx::SqlitePool,
}

impl JobQueue {
    /// Cancel any job. Requires admin authority.
    /// No runtime check — the AdminToken IS the check.
    pub async fn cancel_job(
        &self,
        job_id: Uuid,
        _admin: &WorkerAdminToken,  // proof of authority
    ) -> Result<(), JobError> {
        sqlx::query!("UPDATE jobs SET status = 'cancelled' WHERE id = ?", job_id)
            .execute(&self.db)
            .await?;
        Ok(())
    }

    /// Cancel your own job. Requires owner authority.
    pub async fn cancel_own_job(
        &self,
        job_id: Uuid,
        owner: &OwnerToken,  // proof of ownership
    ) -> Result<(), JobError> {
        // The owner ID is in the token — no separate "who is calling?" lookup
        let affected = sqlx::query!(
            "UPDATE jobs SET status = 'cancelled' WHERE id = ? AND owner_id = ?",
            job_id,
            owner.job_owner_id
        )
        .execute(&self.db)
        .await?
        .rows_affected();

        if affected == 0 {
            return Err(JobError::NotFound);
        }
        Ok(())
    }

    /// Purge all failed jobs from all queues.
    /// Administrative — requires admin token. No owner token accepted.
    pub async fn purge_failed(&self, _admin: &WorkerAdminToken) -> Result<u64, JobError> {
        let result = sqlx::query!("DELETE FROM jobs WHERE status = 'failed'")
            .execute(&self.db)
            .await?;
        Ok(result.rows_affected())
    }

    /// Reprioritize a job to high-priority queue.
    pub async fn reprioritize(
        &self,
        job_id: Uuid,
        _admin: &WorkerAdminToken,
    ) -> Result<(), JobError> {
        sqlx::query!(
            "UPDATE jobs SET queue = 'high' WHERE id = ?",
            job_id
        )
        .execute(&self.db)
        .await?;
        Ok(())
    }
}
```

The compiler rejects calls that lack the token:

```rust
// This won't compile:
async fn unauthorized_cancel(queue: &JobQueue, id: Uuid) {
    queue.cancel_job(id, ???).await;  // where does the token come from?
    // error[E0061]: this function takes 2 arguments but 1 argument was supplied
}

// This won't compile either — owner token for wrong owner is a logic error at compile time:
// owner.job_owner_id is a runtime field, so this check stays runtime.
// But the type system guarantees an owner token WAS issued — no check forgotten.
```

---

## Token Hierarchies: Composing Proofs

Some operations require multiple tokens — you need to prove both "I am an admin" AND "I own the infra account":

```rust
/// Proof of both admin authority AND infrastructure-level access.
/// Requires both tokens to construct.
pub struct InfraAdminToken {
    _private: (),
}

impl InfraAdminToken {
    /// You can only get an InfraAdminToken if you have both an AdminToken
    /// AND have passed infra-level 2FA.
    pub fn from_admin_with_mfa(
        _admin: &WorkerAdminToken,  // proof of admin authority
        mfa_valid: bool,             // runtime check (unavoidable)
    ) -> Option<Self> {
        if mfa_valid {
            Some(InfraAdminToken { _private: () })
        } else {
            None
        }
    }
}

impl JobQueue {
    /// Delete the entire job queue — destructive.
    /// Requires the strongest authority token.
    pub async fn nuke_all_queues(&self, _infra: &InfraAdminToken) -> Result<(), JobError> {
        sqlx::query!("DELETE FROM jobs").execute(&self.db).await?;
        Ok(())
    }
}
```

The hierarchy is encoded in the `from_admin_with_mfa` constructor: you cannot get `InfraAdminToken` without `WorkerAdminToken`. The construction function may still have a runtime check (MFA validation), but it runs *once* and produces a durable proof. Every subsequent call that requires infra-level authority gets the proof for free.

---

## Revocable Tokens

Standard tokens are valid for the lifetime of the reference. For revocable authority (sessions that expire, tokens that can be invalidated), pair the type-level token with a runtime validity check at issuance:

```rust
use std::time::{Duration, Instant};

pub struct SessionToken {
    pub user_id: Uuid,
    pub expires_at: Instant,
    _private: (),
}

impl SessionToken {
    pub(crate) fn issue(user_id: Uuid, ttl: Duration) -> Self {
        SessionToken {
            user_id,
            expires_at: Instant::now() + ttl,
            _private: (),
        }
    }

    /// Check that the token is still valid.
    /// Must be called at the start of each request.
    pub fn is_valid(&self) -> bool {
        Instant::now() < self.expires_at
    }
}

// HTTP middleware verifies and passes the token:
async fn require_session(
    headers: &HeaderMap,
    auth: &AuthService,
) -> Result<SessionToken, ApiError> {
    let token = extract_bearer_token(headers)?;
    let session = auth.validate_session_token(&token).await?;
    // Returns only if the session is valid — subsequent handlers get the proof
    Ok(session)
}

// Handlers receive SessionToken — they know it was valid at request start:
async fn get_my_jobs(
    session: SessionToken,
    queue: &JobQueue,
) -> Result<Vec<Job>, ApiError> {
    queue.list_jobs_for_owner(session.user_id).await
}
```

This is not zero-cost — `validate_session_token` does a database or cache lookup. But it runs once per request, in the middleware. Every handler downstream gets the `SessionToken` proof for free: no repeated validation, no "did someone forget to check auth on this handler?"

---

## Capability Tokens vs. Middleware Guards

A common question: isn't this what HTTP middleware is for?

Middleware is a *runtime* mechanism — it runs at request time and can enforce auth for a group of routes. Capability tokens are a *compile-time* mechanism — the compiler rejects code that calls privileged functions without a token.

They complement each other:

- Middleware: correct at runtime, catches requests that should have been rejected
- Tokens: correct at compile time, catches internal functions called without going through auth

The worst case for middleware-only auth: an internal function bypasses the middleware chain (called directly from a background task, a cron job, an admin script). With capability tokens, that internal function still requires the token — there's no way to bypass it from within the same codebase.

---

## C Comparison

In C:

```c
// C — runtime check, easily forgotten
int cancel_job(job_system_t *sys, job_id_t id, user_context_t *ctx) {
    if (!ctx || !ctx->authenticated || ctx->role < ROLE_ADMIN) {
        return -EPERM;  // Forgot to add this check? Bug.
    }
    // ...
}
```

In Rust with capability tokens:

```rust
// Rust — compile-time check, impossible to forget
pub async fn cancel_job(
    &self,
    job_id: Uuid,
    _admin: &WorkerAdminToken,  // Required by the type system
) -> Result<(), JobError> {
    // No runtime check needed — if we got here, the token exists
}
```

The C version requires trust that every caller remembered to pass a valid `ctx` and that the function remembered to check it. The Rust version requires nothing from the caller except producing the token — which they can only do by going through the authentication function.

---

## taskforge Capability Model

In taskforge, implement these token types:

| Token | Authority | How Obtained |
|-------|-----------|--------------|
| `SessionToken` | Authenticated user | Login endpoint |
| `OwnerToken` | Access to own jobs | Session validated + owner check |
| `WorkerAdminToken` | All jobs, queue management | Admin credential verification |
| `WorkerProcessToken` | Mark jobs as running/complete | Worker process startup |

The `WorkerProcessToken` is particularly useful: it's issued once at worker startup and carried into every job-processing function. Without it, the executor loop cannot mark jobs as `Running` or `Completed`. This prevents a rogue caller from manually flipping job statuses to corrupt the queue.

---

## Exercises

**Exercise 1 — Implement WorkerAdminToken**

Implement the sealed `WorkerAdminToken` with `_private: ()`. Add a constructor that
validates an admin credential. Implement two functions:
- `list_all_jobs(token: &WorkerAdminToken, repo: &dyn JobRepository) -> Vec<Job>` — requires admin
- `clear_dead_letter_queue(token: &WorkerAdminToken, repo: &dyn JobRepository)` — requires admin

Try to construct a `WorkerAdminToken { _private: () }` from outside the crate —
it should fail with a private-field error. Use `trybuild` to capture the expected error.

**Exercise 2 — Token Hierarchy**

Implement an `InfraAdminToken` that can issue `WorkerAdminToken` instances, and a
`WorkerAdminToken` that can issue `WorkerProcessToken` instances. Each issuance requires
the parent token plus an additional credential check.

```rust
let infra = InfraAdminToken::new(infra_credential)?;
let admin = infra.issue_worker_admin(admin_credential)?;
let process = admin.issue_worker_process()?;
// process can now mark jobs running/complete
```

Write a test that verifies a `WorkerProcessToken` cannot directly issue an
`InfraAdminToken` (no `InfraAdminToken::from_process_token` method exists).

**Exercise 3 — Revocable Session Token**

Implement a `SessionToken { issued_at: Instant, expires_after: Duration }` that
wraps a `WorkerAdminToken`. Add a method `is_valid(&self) -> bool` that checks
if `issued_at.elapsed() < expires_after`. Modify the admin functions to accept
`SessionToken` and call `is_valid()` before proceeding:

```rust
fn list_all_jobs(token: &SessionToken, repo: &dyn JobRepository) -> Result<Vec<Job>, AuthError>
```

Write a test that creates a session with 1ms expiry, sleeps 2ms, and verifies
that `list_all_jobs` returns `Err(AuthError::SessionExpired)`.
