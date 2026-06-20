# Doc 06 — Typed Command Interfaces

🟡 Associated types on a trait create a compile-time contract between a command and its response. This eliminates an entire class of parsing bugs — the kind that survive testing and explode in production at 3 AM.

The taskforge system talks to Redis using a command/response protocol. Every command produces a specific response type: `LLEN` returns a count, `BRPOP` returns a list entry, `HGETALL` returns a hash. In a naively typed system, they all return `Vec<u8>` and the caller parses whatever bytes arrive. The compiler has no idea whether the caller parsed the right thing.

Typed command interfaces bind each command to its response type — at the trait level, not just in a comment.

---

## The Untyped Problem

Before: every Redis interaction looks the same to the compiler.

```rust
// Naively typed — everything is Vec<u8>
struct Redis {
    conn: redis::Connection,
}

impl Redis {
    fn command(&mut self, args: &[&str]) -> Result<Vec<u8>, redis::RedisError> {
        // ... send args, receive bytes ...
        Ok(vec![])  // stub
    }
}

fn run_queue_operations(redis: &mut Redis) -> Result<(), Box<dyn std::error::Error>> {
    // LLEN returns a single integer — but we get Vec<u8>
    let raw = redis.command(&["LLEN", "queue:high"])?;
    let count = String::from_utf8(raw)?.parse::<u64>()?;  // 🤞 is this right?

    // BRPOP returns a two-element list [key, value] — but we parse it wrong:
    let raw = redis.command(&["BRPOP", "queue:high", "1"])?;
    let job_id = String::from_utf8(raw)?;  // 🐛 raw is [key_bytes, value_bytes] — not a string

    // HGETALL returns alternating key/value pairs
    let raw = redis.command(&["HGETALL", &format!("job:{job_id}")])?;
    let status = String::from_utf8(raw)?;  // 🐛 this is the whole hash, not just status

    // HGET returns a single optional value
    let raw = redis.command(&["HGET", &format!("job:{job_id}"), "status"])?;
    let count2 = raw[0] as u64;  // 🐛 interpreting status bytes as a number

    Ok(())
}
```

Four bugs, all compile and "work" until they hit production data. The type system offered no resistance.

---

## The Typed Command Pattern

The fix: an associated type on a trait that binds each command to exactly one response type.

### Step 1 — Domain types for Redis values

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct QueueDepth(pub u64);

#[derive(Debug, Clone, PartialEq)]
pub struct JobId(pub String);

#[derive(Debug, Clone)]
pub struct JobRecord {
    pub id: String,
    pub status: String,
    pub job_type: String,
    pub payload: Vec<u8>,
    pub retry_count: u32,
    pub created_at: i64,
}

#[derive(Debug, Clone, PartialEq)]
pub struct JobStatus(pub String);
```

### Step 2 — The command trait

```rust
use std::io;

/// A typed Redis command. `Response` is the associated type — it binds
/// each command implementation to exactly one return type.
pub trait RedisCmd {
    type Response;

    /// The command and its arguments (e.g., ["LLEN", "queue:high"])
    fn args(&self) -> Vec<String>;

    /// Parse the raw Redis response bytes into `Self::Response`.
    /// Parsing logic lives here — never scattered across call sites.
    fn parse(&self, raw: redis::Value) -> io::Result<Self::Response>;
}
```

### Step 3 — One struct per command

```rust
use redis::Value;

/// LLEN queue_name → QueueDepth
pub struct LLen {
    pub queue: String,
}

impl RedisCmd for LLen {
    type Response = QueueDepth;

    fn args(&self) -> Vec<String> {
        vec!["LLEN".into(), self.queue.clone()]
    }

    fn parse(&self, raw: Value) -> io::Result<QueueDepth> {
        match raw {
            Value::Int(n) => Ok(QueueDepth(n as u64)),
            other => Err(io::Error::new(
                io::ErrorKind::InvalidData,
                format!("LLEN: expected integer, got {:?}", other),
            )),
        }
    }
}

/// BRPOP queue_name timeout → Option<JobId>
/// None when the timeout expires with no item.
pub struct BrPop {
    pub queue: String,
    pub timeout_secs: u64,
}

impl RedisCmd for BrPop {
    type Response = Option<JobId>;

    fn args(&self) -> Vec<String> {
        vec!["BRPOP".into(), self.queue.clone(), self.timeout_secs.to_string()]
    }

    fn parse(&self, raw: Value) -> io::Result<Option<JobId>> {
        match raw {
            Value::Nil => Ok(None),  // timeout — no item
            Value::Bulk(items) if items.len() == 2 => {
                // BRPOP returns [queue_key, value]
                match &items[1] {
                    Value::Data(bytes) => {
                        let id = String::from_utf8(bytes.clone())
                            .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
                        Ok(Some(JobId(id)))
                    }
                    other => Err(io::Error::new(
                        io::ErrorKind::InvalidData,
                        format!("BRPOP value: expected bytes, got {:?}", other),
                    )),
                }
            }
            other => Err(io::Error::new(
                io::ErrorKind::InvalidData,
                format!("BRPOP: unexpected response {:?}", other),
            )),
        }
    }
}

/// HGETALL job:{id} → JobRecord
pub struct HGetAll {
    pub job_id: String,
}

impl RedisCmd for HGetAll {
    type Response = Option<JobRecord>;  // None if the key doesn't exist

    fn args(&self) -> Vec<String> {
        vec!["HGETALL".into(), format!("job:{}", self.job_id)]
    }

    fn parse(&self, raw: Value) -> io::Result<Option<JobRecord>> {
        let pairs = match raw {
            Value::Bulk(items) if items.is_empty() => return Ok(None),
            Value::Bulk(items) => items,
            other => {
                return Err(io::Error::new(
                    io::ErrorKind::InvalidData,
                    format!("HGETALL: expected bulk, got {:?}", other),
                ))
            }
        };

        // Pairs alternate: key, value, key, value, ...
        let mut map: std::collections::HashMap<String, String> = std::collections::HashMap::new();
        for chunk in pairs.chunks(2) {
            if let (Value::Data(k), Value::Data(v)) = (&chunk[0], &chunk[1]) {
                let key = String::from_utf8_lossy(k).into_owned();
                let val = String::from_utf8_lossy(v).into_owned();
                map.insert(key, val);
            }
        }

        let record = JobRecord {
            id: self.job_id.clone(),
            status: map.remove("status").unwrap_or_default(),
            job_type: map.remove("job_type").unwrap_or_default(),
            payload: map.remove("payload")
                .map(|s| s.into_bytes())
                .unwrap_or_default(),
            retry_count: map.remove("retry_count")
                .and_then(|s| s.parse().ok())
                .unwrap_or(0),
            created_at: map.remove("created_at")
                .and_then(|s| s.parse().ok())
                .unwrap_or(0),
        };

        Ok(Some(record))
    }
}

/// HGET job:{id} field → Option<JobStatus>
pub struct HGet {
    pub job_id: String,
    pub field: &'static str,
}

impl RedisCmd for HGet {
    type Response = Option<JobStatus>;

    fn args(&self) -> Vec<String> {
        vec!["HGET".into(), format!("job:{}", self.job_id), self.field.into()]
    }

    fn parse(&self, raw: Value) -> io::Result<Option<JobStatus>> {
        match raw {
            Value::Nil => Ok(None),
            Value::Data(bytes) => {
                let s = String::from_utf8(bytes)
                    .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
                Ok(Some(JobStatus(s)))
            }
            other => Err(io::Error::new(
                io::ErrorKind::InvalidData,
                format!("HGET: expected data or nil, got {:?}", other),
            )),
        }
    }
}
```

### Step 4 — A generic executor

```rust
pub struct TypedRedis {
    client: redis::Client,
}

impl TypedRedis {
    pub fn new(url: &str) -> Result<Self, redis::RedisError> {
        Ok(TypedRedis { client: redis::Client::open(url)? })
    }

    /// Execute any typed command. The return type is inferred from the command.
    pub fn execute<C: RedisCmd>(&self, cmd: &C) -> io::Result<C::Response> {
        let mut conn = self.client
            .get_connection()
            .map_err(|e| io::Error::new(io::ErrorKind::ConnectionRefused, e))?;

        let args = cmd.args();
        let mut redis_cmd = redis::cmd(args[0].as_str());
        for arg in &args[1..] {
            redis_cmd.arg(arg.as_str());
        }

        let raw: redis::Value = redis_cmd
            .query(&mut conn)
            .map_err(|e| io::Error::new(io::ErrorKind::Other, e))?;

        cmd.parse(raw)
    }
}
```

### Step 5 — The original bugs become compile errors

```rust
fn run_queue_operations_typed(redis: &TypedRedis) -> io::Result<()> {
    let count: QueueDepth = redis.execute(&LLen { queue: "queue:high".into() })?;
    let item: Option<JobId> = redis.execute(&BrPop { queue: "queue:high".into(), timeout_secs: 1 })?;
    let record: Option<JobRecord> = redis.execute(&HGetAll { job_id: "abc123".into() })?;
    let status: Option<JobStatus> = redis.execute(&HGet { job_id: "abc123".into(), field: "status" })?;

    // Bug #1 (wrong parse) — IMPOSSIBLE: parsing lives in each struct's parse()
    // Bug #2 (BRPOP returned as string) — IMPOSSIBLE: BrPop::parse handles the 2-element bulk
    // Bug #3 (HGETALL treated as one string) — IMPOSSIBLE: HGetAll::parse handles key/value pairs
    // Bug #4 (comparing count and status) — COMPILE ERROR:
    // if count == status { }  ← QueueDepth vs Option<JobStatus>: mismatched types

    if let Some(job_id) = item {
        println!("Processing job: {}", job_id.0);
    }

    Ok(())
}
```

Adding a new command type is one `struct` + one `impl`:

```rust
/// LPUSH queue_name value → QueueDepth (the new length)
pub struct LPush {
    pub queue: String,
    pub value: String,
}

impl RedisCmd for LPush {
    type Response = QueueDepth;

    fn args(&self) -> Vec<String> {
        vec!["LPUSH".into(), self.queue.clone(), self.value.clone()]
    }

    fn parse(&self, raw: Value) -> io::Result<QueueDepth> {
        match raw {
            Value::Int(n) => Ok(QueueDepth(n as u64)),
            other => Err(io::Error::new(
                io::ErrorKind::InvalidData,
                format!("LPUSH: expected integer, got {:?}", other),
            )),
        }
    }
}
```

Every caller that uses `redis.execute(&LPush { ... })` automatically gets `QueueDepth` back — no parsing code anywhere else.

---

## Extending to Async

The same pattern extends to async connections:

```rust
pub trait AsyncRedisCmd {
    type Response;
    fn args(&self) -> Vec<String>;
    fn parse(&self, raw: redis::Value) -> io::Result<Self::Response>;
}

pub struct AsyncTypedRedis {
    client: redis::Client,
}

impl AsyncTypedRedis {
    pub async fn execute<C: AsyncRedisCmd>(&self, cmd: &C) -> io::Result<C::Response> {
        let mut conn = self.client
            .get_async_connection()
            .await
            .map_err(|e| io::Error::new(io::ErrorKind::ConnectionRefused, e))?;

        let args = cmd.args();
        let mut redis_cmd = redis::cmd(args[0].as_str());
        for arg in &args[1..] {
            redis_cmd.arg(arg.as_str());
        }

        let raw: redis::Value = redis_cmd
            .query_async(&mut conn)
            .await
            .map_err(|e| io::Error::new(io::ErrorKind::Other, e))?;

        cmd.parse(raw)
    }
}
```

The trait definition is identical — the executor changes, but every `RedisCmd` implementation works unchanged.

---

## Why This Beats the Alternative

The alternative is what most code does: a giant `enum RedisResponse { Int(i64), Bulk(Vec<Vec<u8>>), Nil }` that the caller has to match on. This has the same problem as the original — the caller is responsible for knowing which variant to expect and which fields to read. The command and its parsing are separated.

The typed command pattern keeps them together:

```
Command struct → knows its args
Command struct → knows how to parse the response
Executor → generic over any command type
Caller → gets the right type, enforced by the compiler
```

The mental model for the caller collapses from "I need to know which variant of the response enum this command returns" to "I execute a command and get back exactly what the command says it returns."

---

## taskforge Command Inventory

In taskforge, implement these Redis commands as typed command structs:

| Command | Response Type | Use Case |
|---------|--------------|----------|
| `LLen { queue }` | `QueueDepth` | Monitor queue backlogs |
| `BrPop { queue, timeout }` | `Option<JobId>` | Worker fetch loop |
| `LPush { queue, value }` | `QueueDepth` | Enqueue a job |
| `HGetAll { job_id }` | `Option<JobRecord>` | Load full job state |
| `HGet { job_id, field }` | `Option<String>` | Check single field |
| `HSet { job_id, field, value }` | `()` | Update job field |
| `HMSet { job_id, fields }` | `()` | Update multiple fields at once |
| `Del { key }` | `u32` | Remove a job record |

Each struct is 10–30 lines. The executor is written once. New command types add zero complexity to existing code.
