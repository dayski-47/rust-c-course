# Doc 06 — Writing Your Own Future

🟡 Most async Rust developers never write a custom Future. After this doc, you'll understand exactly what the runtime is doing every time you `.await` something.

You've been using `async fn` and `.await` since section 4. Under the hood, the compiler turns every `async fn` into a struct that implements `Future`. This doc lifts the hood, shows you what that struct looks like, and teaches you to write custom futures from scratch. The music queue project specifically needs a `ThrottledFuture` for rate limiting — understanding how to build it from first principles is the goal.

---

## The `Future` Trait Revisited

At its core, a `Future` is a state machine with one method:

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

The contract:
- Return `Poll::Pending` if the result isn't ready yet. Before returning `Pending`, register a waker via `cx.waker().clone()` so the executor knows to re-poll when the situation changes.
- Return `Poll::Ready(value)` when the result is available.
- Once you return `Poll::Ready`, you must never be polled again.

The executor (Tokio, in the music queue project) calls `poll` on your task's root future. That future might poll sub-futures. Each sub-future either returns `Pending` (not ready yet) or `Ready(value)` (done). When everything is `Pending`, Tokio parks the thread until a waker fires.

---

## Building a `TimerFuture` From Scratch

The simplest non-trivial future: resolves after a delay.

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::Duration;

pub struct TimerFuture {
    shared: Arc<Mutex<TimerState>>,
}

struct TimerState {
    elapsed: bool,
    waker: Option<Waker>,
}

impl TimerFuture {
    pub fn new(duration: Duration) -> Self {
        let shared = Arc::new(Mutex::new(TimerState {
            elapsed: false,
            waker: None,
        }));

        // Spawn a background thread that fires the waker after `duration`
        let shared_clone = Arc::clone(&shared);
        thread::spawn(move || {
            thread::sleep(duration);
            let mut state = shared_clone.lock().unwrap();
            state.elapsed = true;
            // Notify the executor: "come poll me again, I might be ready"
            if let Some(waker) = state.waker.take() {
                waker.wake();
            }
        });

        TimerFuture { shared }
    }
}

impl Future for TimerFuture {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        let mut state = self.shared.lock().unwrap();

        if state.elapsed {
            Poll::Ready(())
        } else {
            // CRITICAL: update the waker every time poll is called.
            // The executor might have given us a new waker since the last poll.
            // If we store a stale waker, we'll never get polled again.
            state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

**The waker is the key.** When you return `Poll::Pending`, you're making a promise: "I will call `waker.wake()` when the situation changes." If you don't keep that promise, your future sits in `Pending` forever — the executor never polls it again.

```rust
// Using TimerFuture in async code
async fn delayed_play(queue: &MusicQueue) {
    println!("Queuing song, will play in 2 seconds...");
    TimerFuture::new(Duration::from_secs(2)).await;
    queue.play_next().await;
}
```

> **Note**: This implementation spawns one OS thread per timer. In production, use `tokio::time::sleep` — it's backed by a shared timer wheel and costs essentially zero. The hand-rolled `TimerFuture` is for understanding, not for production queues.

---

## How the Executor Interacts With Poll

Here's what Tokio's scheduler does internally when you `.await` a future:

```
1. Call poll(future, waker)
   ├─ Future returns Poll::Ready(val)  → extract val, continue execution
   └─ Future returns Poll::Pending     → put task in "parked" list, stop polling
   
2. Later: waker.wake() is called (from a background thread, I/O callback, etc.)
   → Tokio moves the task from "parked" back to the "ready" queue
   
3. Worker thread picks up the task, calls poll again
   → Repeat from step 1
```

The waker is the bridge between the asynchronous world (network I/O, timers, channels) and Tokio's task scheduler.

---

## Building a `Join` Combinator

The `join!` macro is itself a future that polls two sub-futures concurrently and returns both results. Here's how to build one:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

/// A future that completes when BOTH A and B complete.
/// Polls A and B on each poll — they run concurrently on the same thread.
pub struct JoinTwo<A, B>
where
    A: Future,
    B: Future,
{
    a: MaybeDone<A>,
    b: MaybeDone<B>,
}

enum MaybeDone<F: Future> {
    Pending(F),           // still running
    Done(F::Output),      // finished, output waiting
    Taken,                // output already extracted
}

impl<A, B> JoinTwo<A, B>
where
    A: Future + Unpin,
    B: Future + Unpin,
{
    pub fn new(a: A, b: B) -> Self {
        JoinTwo {
            a: MaybeDone::Pending(a),
            b: MaybeDone::Pending(b),
        }
    }
}

impl<A, B> Future for JoinTwo<A, B>
where
    A: Future + Unpin,
    B: Future + Unpin,
{
    type Output = (A::Output, B::Output);

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.get_mut();

        // Poll A if it's still pending
        if let MaybeDone::Pending(ref mut fut) = this.a {
            if let Poll::Ready(val) = Pin::new(fut).poll(cx) {
                this.a = MaybeDone::Done(val);
            }
        }

        // Poll B if it's still pending
        if let MaybeDone::Pending(ref mut fut) = this.b {
            if let Poll::Ready(val) = Pin::new(fut).poll(cx) {
                this.b = MaybeDone::Done(val);
            }
        }

        // Return Ready only when BOTH are done
        if matches!((&this.a, &this.b), (MaybeDone::Done(_), MaybeDone::Done(_))) {
            let a = match std::mem::replace(&mut this.a, MaybeDone::Taken) {
                MaybeDone::Done(v) => v,
                _ => unreachable!(),
            };
            let b = match std::mem::replace(&mut this.b, MaybeDone::Taken) {
                MaybeDone::Done(v) => v,
                _ => unreachable!(),
            };
            Poll::Ready((a, b))
        } else {
            Poll::Pending
        }
    }
}
```

Notice what "concurrent" means here: A and B are both polled in the **same `poll()` call**. No threads are involved. When A is pending, the executor parks the task. When A's waker fires, the task is re-polled, and now B might advance further. It's cooperative multitasking on a single thread.

```rust
// Using JoinTwo in the music queue
async fn load_queue_data(db: &Db) -> (Vec<Track>, UserPreferences) {
    JoinTwo::new(
        db.get_queue_tracks(),
        db.get_user_preferences(),
    ).await
    // Both queries run concurrently on the same async task
}
```

---

## Building a Rate-Limiting Future for the Music Queue

The music queue needs to respect external API rate limits when fetching track metadata. Here's a custom future that limits throughput:

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::collections::VecDeque;
use std::time::{Duration, Instant};

/// Shared state for the rate limiter — allows the background timer
/// to wake parked futures when a new slot becomes available.
pub struct RateLimiterState {
    tokens: u32,              // available request slots
    max_tokens: u32,          // maximum slots (e.g., 60 per minute)
    refill_per_period: u32,   // how many tokens to add each period
    last_refill: Instant,
    refill_period: Duration,
    waiting: VecDeque<Waker>, // wakers of futures waiting for a token
}

impl RateLimiterState {
    fn try_acquire(&mut self) -> bool {
        self.refill_if_needed();
        if self.tokens > 0 {
            self.tokens -= 1;
            true
        } else {
            false
        }
    }

    fn refill_if_needed(&mut self) {
        let now = Instant::now();
        if now.duration_since(self.last_refill) >= self.refill_period {
            self.tokens = (self.tokens + self.refill_per_period).min(self.max_tokens);
            self.last_refill = now;
            // Wake all waiting futures — there might be tokens available now
            while let Some(waker) = self.waiting.pop_front() {
                waker.wake();
            }
        }
    }
}

pub struct RateLimiter {
    state: Arc<Mutex<RateLimiterState>>,
}

impl RateLimiter {
    pub fn new(max_per_minute: u32) -> Self {
        RateLimiter {
            state: Arc::new(Mutex::new(RateLimiterState {
                tokens: max_per_minute,
                max_tokens: max_per_minute,
                refill_per_period: max_per_minute,
                last_refill: Instant::now(),
                refill_period: Duration::from_secs(60),
                waiting: VecDeque::new(),
            })),
        }
    }

    /// Returns a future that resolves when a rate limit token is available
    pub fn acquire(&self) -> AcquireFuture {
        AcquireFuture {
            state: Arc::clone(&self.state),
        }
    }
}

pub struct AcquireFuture {
    state: Arc<Mutex<RateLimiterState>>,
}

impl Future for AcquireFuture {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        let mut state = self.state.lock().unwrap();

        if state.try_acquire() {
            Poll::Ready(())
        } else {
            // No token available — register our waker so we'll be notified
            // when tokens refill
            state.waiting.push_back(cx.waker().clone());
            Poll::Pending
        }
    }
}

// Using the rate limiter in the queue's metadata fetcher:
pub async fn fetch_track_metadata(
    limiter: &RateLimiter,
    spotify_client: &SpotifyClient,
    track_id: &str,
) -> Result<TrackMetadata, Error> {
    limiter.acquire().await;  // Wait for a rate limit slot
    spotify_client.get_track(track_id).await
}
```

---

## When to Write a Custom Future vs. Use Combinators

Most async code never needs a custom `Future` implementation:

| Use case | Recommended approach |
|----------|---------------------|
| Timer / delay | `tokio::time::sleep` |
| Concurrent execution | `tokio::join!` / `futures::join_all` |
| Race to completion | `tokio::select!` |
| Retry with backoff | `tokio_retry` crate |
| Rate limiting | `tokio::time::interval` + channel, or use a crate |
| Custom I/O source | Implement `Future` using `std::task::Waker` |
| Novel concurrency primitive | Implement `Future` from scratch |

The only times to implement `Future` by hand:
1. Wrapping a callback-based async system (event emitters, OS APIs that use completion callbacks)
2. Creating a novel concurrency primitive that existing combinators can't express
3. Educational purposes (this doc)

For the music queue, the hand-rolled `TimerFuture` is replaced by `tokio::time::sleep` in production. The rate limiter uses a real implementation from the `governor` crate (a token bucket implementation that handles refill correctly under concurrent access).

---

## How It Breaks

**Returning `Poll::Pending` without registering a waker.**
This is the most common custom Future bug. If you return `Poll::Pending` and don't call `cx.waker().clone()` and store it somewhere, the executor will never poll you again. The future silently hangs forever.

```rust
// BUG: returns Pending without registering a waker
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
    if self.ready {
        Poll::Ready(())
    } else {
        Poll::Pending  // executor will NEVER poll us again — future hangs
    }
}
```

**Not updating the waker on re-poll.**
```rust
// BUG: stores the waker only once, ignores updates
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
    let mut state = self.shared.lock().unwrap();
    if state.elapsed {
        Poll::Ready(())
    } else {
        if state.waker.is_none() {  // WRONG: only sets it once
            state.waker = Some(cx.waker().clone());
        }
        Poll::Pending
    }
}
```
The executor can give you a different `Waker` on each call to `poll`. Always replace the stored waker unconditionally.

**Holding a `Mutex` lock across a waker call.**
```rust
// DEADLOCK: calling waker.wake() while holding the mutex
let mut state = shared.lock().unwrap();
state.elapsed = true;
if let Some(waker) = &state.waker {
    waker.wake_by_ref();  // This might immediately re-poll us, which tries to lock the mutex again
}
// Drop the lock BEFORE waking to prevent deadlock
```
The fix: take the waker out of the lock, drop the lock, then call `wake()`:
```rust
let waker = {
    let mut state = shared.lock().unwrap();
    state.elapsed = true;
    state.waker.take()  // take ownership, releasing it from the locked state
};
// Lock is released here (state drops)
if let Some(waker) = waker {
    waker.wake();  // safe — no lock held
}
```

**Polling a future after it returned `Poll::Ready`.**
The `Future` contract says: once you return `Poll::Ready`, the future must not be polled again. Violating this is undefined behavior for some futures (they may panic or return wrong values). The executor handles this correctly; custom combinators must too.
