# Doc 04 - Functional vs. Imperative: When to Chain

🟡 Think about it - you just learned closures and iterators. Now: when do you actually use them?

Rust gives you a genuine choice: you can write C-style for-loops or you can chain functional combinators. Both compile to the same machine code. The question is which communicates intent more clearly for a given situation. This doc builds that judgment - using the system monitor project as a concrete lens.

---

## The Core Principle

**Functional style** (iterator chains, combinators) wins when you are *transforming data through a pipeline*. Each step takes input, produces output, no mutation.

**Imperative style** (for loops, mutable variables) wins when you are *managing state transitions with side effects* - especially when you're building multiple outputs simultaneously or when branches do fundamentally different things.

Most real code has both, and the skill is recognizing which pattern you're in.

---

## Iterator Chains: When They Win

### Data pipelines - filtering and transforming collections

This is the strongest use case. Compare processing CPU samples:

```rust
// Imperative: 8 lines, 2 mutable bindings, two nested checks
let mut high_cpu_ids = Vec::new();
for sample in &samples {
    if sample.cpu_percent > 80.0 {
        if let Some(name) = &sample.process_name {
            high_cpu_ids.push(name.clone());
        }
    }
}

// Functional: 5 lines, 0 mutable bindings, one coherent pipeline
let high_cpu_ids: Vec<String> = samples.iter()
    .filter(|s| s.cpu_percent > 80.0)
    .filter_map(|s| s.process_name.clone())
    .collect();
```

The functional version is better here because:
- Each step is independently readable: filter hot processes, then extract their names
- No mutation - data flows in one direction from left to right
- You can add/remove/reorder stages without restructuring the whole function
- LLVM inlines these adapter calls to the same assembly as the loop

### Aggregation - computing one value from a collection

```rust
// Imperative: mutable accumulators, manual loop
let mut total_cpu = 0.0f64;
let mut count = 0usize;
for s in &samples {
    total_cpu += s.cpu_percent;
    count += 1;
}
let avg_cpu = total_cpu / count as f64;

// Functional: standard library vocabulary
let avg_cpu = samples.iter()
    .map(|s| s.cpu_percent)
    .sum::<f64>() / samples.len() as f64;
```

The standard library has a word for almost every common aggregation. Learn them:

```rust
let max_cpu = samples.iter().map(|s| s.cpu_percent).fold(0.0f64, f64::max);
let min_mem = samples.iter().map(|s| s.mem_bytes).min().unwrap_or(0);
let total   = samples.iter().map(|s| s.cpu_percent).sum::<f64>();
let count   = samples.iter().filter(|s| s.cpu_percent > 50.0).count();
let found   = samples.iter().any(|s| s.cpu_percent > 95.0);
let all_ok  = samples.iter().all(|s| s.mem_bytes < MAX_MEM);
let first   = samples.iter().find(|s| s.process_name.as_deref() == Some("nginx"));
```

Every loop that computes one of these values should be replaced by the corresponding method. `any`, `all`, `find`, `count`, `sum`, `min`, `max` - these have a single, clear name that says what you're doing. A for-loop with a boolean accumulator requires reading the whole body to understand it.

---

## Option and Result Combinators

`Option<T>` and `Result<T, E>` have their own combinator families. Learning them eliminates most of the `if let Some(x) = ...` boilerplate in your code.

### The Option combinator family

```rust
// Each of these replaces a match expression:

// opt.map(f) - transform the inside value if present
let display_name: Option<String> = sample.process_name
    .map(|name| name.to_uppercase());

// opt.unwrap_or(default) - use a fallback if None
let name: String = sample.process_name
    .unwrap_or_else(|| String::from("[kernel]"));

// opt.filter(pred) - keep it only if it passes a test
let hot_name: Option<&str> = sample.process_name.as_deref()
    .filter(|_| sample.cpu_percent > 80.0);

// opt.and_then(f) - chain operations that might also return None
let port: Option<u16> = config.get("port")
    .and_then(|v| v.parse::<u16>().ok());

// bool.then_some(v) - condition to Option
let alert: Option<&str> = (sample.cpu_percent > 95.0).then_some("CRITICAL");
```

**Rule**: if your `if let Some(x)` produces a value (not a side effect), there's a combinator for it. The combinator version is shorter and names the intent.

**When `if let` is better**: when the `Some` branch contains multiple statements, or when the `None` branch does something fundamentally different (not just "use a default"):

```rust
// if let is clearer here - these branches are different code paths
if let Some(conn) = pool.try_acquire() {
    let result = conn.query(sql)?;
    conn.release();
    Ok(result)
} else {
    log::warn!("Pool exhausted, queuing request");
    self.request_queue.push(sql.to_string());
    Err(Error::PoolExhausted)
}
```

### The ? operator: functional error propagation with readable syntax

`?` is the best synthesis of functional and imperative in Rust:

```rust
// This and_then chain...
fn load_config(path: &str) -> Result<Config, Error> {
    read_file(path)
        .and_then(|text| text.parse::<toml::Value>())
        .and_then(|toml| Config::from_toml(toml))
}

// ...is equivalent to this (usually preferred because you get named intermediates):
fn load_config(path: &str) -> Result<Config, Error> {
    let text = read_file(path)?;
    let toml = text.parse::<toml::Value>()?;
    Config::from_toml(toml)
}
```

The `?` version lets you add logging (`log::debug!("{toml:?}")`), extra validation, or context between steps. Use `.and_then()` chains when you're building an `Option` or `Result` without a function to return from; use `?` when you're in a function that returns `Result`.

---

## For Loops: When They Win

### Building multiple outputs in one pass

This is the case where functional style genuinely loses:

```rust
// Multiple outputs - the loop is clearly better
let mut warnings: Vec<String> = Vec::new();
let mut errors: Vec<String> = Vec::new();
let mut stats = SampleStats::default();

for sample in &samples {
    stats.total_cpu += sample.cpu_percent;
    stats.total_mem += sample.mem_bytes;

    match classify(sample) {
        Level::Warning => warnings.push(format!("{}: high CPU", sample.pid)),
        Level::Error   => {
            errors.push(format!("{}: critical", sample.pid));
            if sample.cpu_percent > 99.0 {
                trigger_alert(sample); // side effect
            }
        }
        Level::Normal  => {}
    }
}

// Functional version would require three separate filter passes,
// or a fold with three mutable accumulators inside a tuple - worse in every way.
```

Use the loop when:
- You're building multiple output collections in one pass
- The branches contain side effects (I/O, logging, alerting)
- The logic inside each branch is different, not just a transform

### State machines

```rust
// A parser reading tokens - the loop IS the algorithm
let mut state = ParseState::Start;
for token in tokens {
    state = match (state, token) {
        (ParseState::Start, Token::Header(h)) => ParseState::InSection(h),
        (ParseState::InSection(h), Token::Key(k)) => ParseState::GotKey(h, k),
        (ParseState::GotKey(h, k), Token::Value(v)) => {
            config.insert(h, k, v);
            ParseState::InSection(h)
        }
        (_, Token::Eof) => break,
        (s, t) => return Err(ParseError::unexpected(s, t)),
    };
}
```

No functional equivalent is better. The loop with state is the natural model here.

---

## Performance: They Are Equal

A critical point for engineers coming from C: **Rust iterator chains compile to the same machine code as hand-written loops.**

LLVM inlines the closure calls, eliminates the adapter structs, and produces identical assembly. This is not aspirational - it is the design principle, and it is measured.

```rust
// These two compile to IDENTICAL assembly in release mode:

// Functional
let sum: i64 = (0..1_000_000)
    .filter(|n| n % 2 == 0)
    .map(|n| n * n)
    .sum();

// Imperative
let mut sum: i64 = 0;
for n in 0..1_000_000 {
    if n % 2 == 0 {
        sum += n * n;
    }
}
```

**The one real difference**: `.collect()` allocates. If you write `.map().collect().iter().map().collect()` with intermediate vectors, you pay for those allocations. The fix: chain adapters directly all the way to the final `.collect()`. Don't collect intermediate results unless you actually need them as a reusable collection.

---

## The Standard Library Vocabulary

One of the biggest mistakes C developers make in Rust: writing for-loops for operations that have a named method.

| Don't write this | Write this instead |
|------------------|--------------------|
| Loop with a bool flag to find if any element passes | `.any(pred)` |
| Loop with a bool flag to check if all pass | `.all(pred)` |
| Loop tracking the first matching element | `.find(pred)` |
| Loop counting matches | `.filter(pred).count()` |
| Loop accumulating a sum | `.map(f).sum()` |
| Loop finding the maximum | `.map(f).max_by(cmp)` or `.fold(init, f64::max)` |
| Loop pushing matching elements to a new Vec | `.filter(pred).collect()` |
| Loop mapping + filtering in one | `.filter_map(f)` |
| Loop flattening a Vec<Vec<T>> | `.flatten().collect()` or `.flat_map(f)` |
| Loop building a HashMap | `.map(|x| (key(x), val(x))).collect::<HashMap<_,_>>()` |

Knowing these names is how you read other engineers' Rust code. They're also how the compiler infers your intent and generates the best code.

---

## Applied to the System Monitor

Here's a real function from the system monitor project showing both styles working together appropriately:

```rust
pub fn generate_report(samples: &[Sample]) -> MonitorReport {
    // Functional: computing statistics is a pure data pipeline
    let cpu_avg = samples.iter().map(|s| s.cpu_percent).sum::<f64>()
        / samples.len() as f64;
    let cpu_max = samples.iter().map(|s| s.cpu_percent)
        .fold(0.0f64, f64::max);
    let mem_max = samples.iter().map(|s| s.mem_bytes).max().unwrap_or(0);

    // Functional: top-5 CPU consumers is a clean pipeline
    let mut top_cpu: Vec<&Sample> = samples.iter()
        .filter(|s| s.process_name.is_some())
        .collect();
    top_cpu.sort_by(|a, b| b.cpu_percent.partial_cmp(&a.cpu_percent).unwrap());
    top_cpu.truncate(5);

    // Imperative: building three separate output collections
    let mut alerts = Vec::new();
    let mut warnings = Vec::new();
    let mut infos = Vec::new();

    for sample in samples {
        if sample.cpu_percent > 90.0 {
            alerts.push(Alert::HighCpu(sample.pid, sample.cpu_percent));
        }
        if sample.mem_bytes > MEM_WARN_THRESHOLD {
            warnings.push(Warning::HighMem(sample.pid, sample.mem_bytes));
        }
        infos.push(Info::from(sample));
    }

    MonitorReport { cpu_avg, cpu_max, mem_max, top_cpu, alerts, warnings, infos }
}
```

The statistics (pure transformations with one output) use the functional style. The classification (multiple output collections, side-effectful branches) uses a loop. This is the correct split.

---

## How It Breaks

**`.collect()` on an intermediate result when you don't need it.**
```rust
// Allocates a Vec just to immediately iterate it again - wasteful
let filtered: Vec<&Sample> = samples.iter().filter(pred).collect();
let result: Vec<_> = filtered.iter().map(transform).collect();

// Correct: chain adapters directly
let result: Vec<_> = samples.iter().filter(pred).map(transform).collect();
```

**`fold()` for something that has a named method.**
```rust
// Opaque: what does this compute?
let total = samples.iter().fold(0.0, |acc, s| acc + s.cpu_percent);

// Clear: the intent is in the name
let total: f64 = samples.iter().map(|s| s.cpu_percent).sum();
```

**Five-level chain that nobody can read.**
When a chain exceeds ~4 adapters, give intermediate results names:
```rust
// Hard: too much to hold in your head at once
let result = data.iter().filter(|x| x.active).flat_map(|x| x.events.iter())
    .filter(|e| e.severity > 2).map(|e| e.id).collect::<HashSet<_>>();

// Better: named intermediate
let active_events = data.iter()
    .filter(|x| x.active)
    .flat_map(|x| x.events.iter())
    .filter(|e| e.severity > 2);

let high_severity_ids: HashSet<u64> = active_events.map(|e| e.id).collect();
```

**Using `.for_each()` when a loop is clearer.**
`for_each` is a method that runs a closure for its side effects. It almost never improves readability over a regular loop, and it hides the side effects:
```rust
// Opaque - what does this do? What does it produce?
samples.iter().for_each(|s| process_and_log(s));

// Clearer - it's obviously a loop with side effects
for s in &samples {
    process_and_log(s);
}
```

---

## Decision Flowchart

```
What are you computing?
│
├── One value from a collection (sum, max, count, any, all, find)?
│     → Standard library aggregation method
│
├── A new collection from an existing one (map, filter, transform)?
│     → Iterator chain → .collect()
│
├── Multiple output collections or side effects in branches?
│     → For loop
│
├── A single Option/Result with a short transform?
│     → Combinator (.map, .unwrap_or, .and_then)
│
└── A state machine or parser?
      → For/while loop with explicit state
```

The fastest path to good Rust style: stop writing `if let Some(x) = opt { Some(transform(x)) } else { None }` and write `opt.map(transform)`. Then stop writing `for x in xs { if pred(x) { result.push(x); } }` and write `xs.iter().filter(pred).collect()`. Those two habits cover 80% of the cases.
