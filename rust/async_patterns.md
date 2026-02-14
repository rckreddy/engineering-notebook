# Async Patterns from Rust's Standard Library & Ecosystem

Skills and patterns extracted from studying `core::future`, `core::task`, `core::async_iter`, and the patterns mirrored in the `tokio` and `futures` ecosystems. Continues from Skills 1-25.

---

## Skill 26: The Future Trait — One Method, Compiler Does the Rest

**Pattern:** Like `Iterator::next()`, `Future` requires only one method — `poll`. The compiler generates state machines from `async/await` that implement it.

```rust
#[must_use = "futures do nothing unless you `.await` or poll them"]
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

**Why `Pin<&mut Self>`:** Async state machines can be self-referential (a local variable borrowed across `.await`). `Pin` guarantees the future won't move after first poll, making self-references sound.

**Why `Context` instead of just `&Waker`:** Forward compatibility. `Context` started with just a `Waker` but has since gained `local_waker` and `ext` fields — no trait signature change needed.

**Skill to practice:** Understand that `async fn` is syntactic sugar. Every `.await` becomes a state transition in a compiler-generated enum. This mental model explains everything else in async Rust.

---

## Skill 27: Async/Await Desugars to State Machine Enums

**Pattern:** The compiler transforms each `async fn` into an enum with one variant per `.await` point, implementing `Future` as a state machine.

```rust
// Source:
async fn fetch_data(url: String) -> Result<Vec<u8>, Error> {
    let response = http_get(&url).await?;
    let body = response.read_body().await?;
    Ok(body)
}

// Compiler generates (conceptually):
enum FetchDataFuture {
    Start { url: String },
    WaitingHttpGet { url: String, fut: HttpGetFuture },
    WaitingReadBody { fut: ReadBodyFuture },
    Done,
}

impl Future for FetchDataFuture {
    type Output = Result<Vec<u8>, Error>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        loop {
            match self.state {
                Start { .. } => { /* create sub-future, transition */ }
                WaitingHttpGet { fut, .. } => {
                    match fut.poll(cx) {
                        Poll::Ready(Ok(response)) => { /* transition */ }
                        Poll::Ready(Err(e)) => return Poll::Ready(Err(e)),
                        Poll::Pending => return Poll::Pending,
                    }
                }
                // ... etc
            }
        }
    }
}
```

**Key consequences:**
- **Zero heap allocation** — the future is a stack-allocated enum (unless explicitly `Box::pin`'d or spawned)
- **Size = largest variant** — variables held across `.await` become enum fields; variables not crossing `.await` are stack-local during that poll
- **`?` in async** = transition to `Done` + `return Poll::Ready(Err(e))`

**Contrast with other runtimes:**

| Runtime | Mechanism | Cost |
|---------|-----------|------|
| Rust async | Compiler-generated enum | Zero heap alloc, size known at compile time |
| Go goroutines | Growable stacks | 2-8 KB initial stack per goroutine |
| JS Promises | Heap-allocated closures | GC-managed heap objects |

**Skill to practice:** When debugging async, think in terms of the state machine. "Where is this future suspended?" maps to "which enum variant is it in?"

---

## Skill 28: The Waker Contract — How Futures Get Re-polled

**Pattern:** When a future returns `Poll::Pending`, it *must* arrange for `waker.wake()` to be called when progress is possible. The Waker is the executor's callback mechanism.

```rust
// Waker is built from a raw vtable (lives in core, no alloc needed)
pub struct RawWaker {
    data: *const (),
    vtable: &'static RawWakerVTable,
}

pub struct RawWakerVTable {
    clone:       unsafe fn(*const ()) -> RawWaker,
    wake:        unsafe fn(*const ()),       // consumes the waker
    wake_by_ref: unsafe fn(*const ()),       // borrows the waker
    drop:        unsafe fn(*const ()),
}
```

**Why raw function pointers, not trait objects:** The `Future` trait lives in `core` (no allocator). Trait objects need `Arc`/`Box` which require `alloc`. The vtable approach works in `#![no_std]` with zero allocations.

**The ergonomic `Wake` trait (in `alloc`):**
```rust
pub trait Wake {
    fn wake(self: Arc<Self>);
    fn wake_by_ref(self: &Arc<Self>) {
        self.clone().wake() // default: clone then consume
    }
}
```

**wake vs wake_by_ref optimization:**

| Method | Ownership | Use when |
|--------|-----------|----------|
| `wake(self)` | Consumes the waker | You don't need the waker anymore (saves a clone) |
| `wake_by_ref(&self)` | Borrows | You need to keep the waker for future use |

**Skill to practice:** When writing manual `Future` impls, always store `cx.waker().clone()` before returning `Pending`. Failure to wake = task hangs forever.

---

## Skill 29: `poll_fn` — Quick Custom Futures Without a Struct

**Pattern:** `poll_fn` creates a `Future` from a closure, bridging poll-based APIs to async.

```rust
pub fn poll_fn<T, F>(f: F) -> PollFn<F>
where
    F: FnMut(&mut Context<'_>) -> Poll<T>,
{
    PollFn { f }
}

// Usage: a future that yields once then completes
async fn yield_now() {
    let mut yielded = false;
    std::future::poll_fn(|cx| {
        if !yielded {
            yielded = true;
            cx.waker().wake_by_ref();
            Poll::Pending
        } else {
            Poll::Ready(())
        }
    }).await
}
```

**The `ready!` macro — propagate `Pending` in manual poll impls:**
```rust
// Instead of:
match sub_future.poll(cx) {
    Poll::Ready(val) => val,
    Poll::Pending => return Poll::Pending,
}

// Write:
let val = ready!(sub_future.poll(cx));
```

**Skill to practice:** Use `poll_fn` for quick one-off futures that need `Context` access. Use `ready!` to reduce boilerplate in manual `poll` implementations.

---

## Skill 30: `join!` — Concurrent Polling (All Must Complete)

**Pattern:** `join!` polls multiple futures within a single task, completing when *all* are done. No heap allocation, no spawning.

```rust
// All three requests are in-flight simultaneously
let (user, posts, comments) = tokio::join!(
    fetch_user(id),
    fetch_posts(id),
    fetch_comments(id),
);
```

**How it works internally:** Expands to a struct holding `MaybeDone<F>` for each future. Each `poll` call round-robins through all sub-futures. Returns `Ready` only when every sub-future is `Done`.

**`join!` is NOT parallelism:**
```rust
// Concurrent (interleaved on one thread/task):
tokio::join!(future_a, future_b);

// Parallel (actually on different threads):
let a = tokio::spawn(future_a);
let b = tokio::spawn(future_b);
(a.await?, b.await?)
```

**`join!` can borrow local data — `spawn` cannot:**
```rust
async fn structured(data: &[u8]) {
    // Works! No 'static, no Send needed
    let (hash, compressed) = tokio::join!(
        compute_hash(data),
        compress(data),
    );
}
```

**Skill to practice:** Default to `join!` for I/O-bound concurrency. It's simpler, borrows-friendly, and zero-alloc. Use `spawn` only when you need true parallelism or task independence.

---

## Skill 31: `select!` — Racing Futures (First Wins, Rest Cancelled)

**Pattern:** `select!` polls multiple futures, completes when the *first* one resolves. The others are **dropped** (cancelled).

```rust
async fn fetch_with_timeout() -> Result<Data, Error> {
    tokio::select! {
        result = fetch_data() => result,
        _ = tokio::time::sleep(Duration::from_secs(5)) => {
            Err(Error::Timeout) // fetch_data is DROPPED here
        }
    }
}
```

**The cancel safety problem:**
```rust
// DANGEROUS: read_exact buffers partial data internally
loop {
    tokio::select! {
        result = stream.read_exact(&mut buf) => { process(&buf); }
        _ = sleep(Duration::from_secs(30)) => {
            // If 500 of 1024 bytes were read, those 500 bytes are LOST
        }
    }
}

// FIX: Persist the future across iterations
let read_fut = stream.read_exact(&mut buf);
tokio::pin!(read_fut);
loop {
    tokio::select! {
        result = &mut read_fut => { process(&buf); break; }
        _ = cancel.recv() => { continue; } // read_fut survives
    }
}
```

**Cancel-safe operations:** `recv()`, `accept()`, `sleep()` (stateless or message stays in channel)
**Cancel-unsafe operations:** `read_exact()`, `read_line()` (partial data buffered internally)

| | `join!` | `select!` |
|---|---|---|
| Completes when | All done | First done |
| Other futures | Kept alive | Dropped |
| Cancel safety | Not a concern | Critical |
| Use case | Fan-out | Timeout / race |

**Skill to practice:** Prefer `join!` by default. Use `select!` for timeouts and races. Always verify cancel safety of operations inside `select!` branches.

---

## Skill 32: The Executor/Reactor Split

**Pattern:** The stdlib provides the trait (`Future`), the compiler provides the transform (`async/await` → state machine), and the *ecosystem* provides the runtime. This separation is deliberate.

```
┌─────────────────────────────────────────────────┐
│  Your async code (async fn, .await)             │
├─────────────────────────────────────────────────┤
│  std::future::Future + Poll + Waker             │  ← stdlib (trait only)
├─────────────────────────────────────────────────┤
│  Executor (polls tasks, work-stealing)          │  ← runtime (tokio, smol, embassy)
│  Reactor (epoll/kqueue/IOCP, I/O events)        │
│  Timer driver                                   │
├─────────────────────────────────────────────────┤
│  Compiler: async fn → state machine enum        │  ← zero-cost abstraction
│  No heap alloc unless boxed/spawned             │
└─────────────────────────────────────────────────┘
```

**The flow:**
1. Executor polls a future
2. Future tries I/O → not ready
3. Future registers `Waker` with the reactor
4. Returns `Poll::Pending`
5. Reactor detects readiness (epoll)
6. Reactor calls `waker.wake()`
7. Executor re-polls the future

**Why this matters:** The same `async/await` syntax works on a multi-threaded server (tokio), an embedded microcontroller (embassy), or WASM (wasm-bindgen-futures) — each with a tuned runtime, sharing the same zero-cost compilation.

**Skill to practice:** Understand that `Future` is runtime-agnostic. Your async business logic shouldn't depend on a specific runtime unless it uses runtime-specific APIs (timers, I/O, spawning).

---

## Skill 33: The `Send` Bound on Spawned Futures

**Pattern:** `tokio::spawn` requires `F: Future + Send + 'static` because work-stealing schedulers migrate tasks between threads.

```rust
pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
```

**What makes a future `!Send`:** Holding a `!Send` type across an `.await` point.

```rust
async fn fails() {
    let data = Rc::new(42);          // Rc is !Send
    tokio::time::sleep(d).await;     // data held across .await
    println!("{data}");              // ERROR: future is !Send
}

// Fix 1: Drop before .await
async fn fixed() {
    { let data = Rc::new(42); println!("{data}"); }
    tokio::time::sleep(d).await;
}

// Fix 2: Use Send equivalent
async fn fixed2() {
    let data = Arc::new(42);         // Arc IS Send
    tokio::time::sleep(d).await;
    println!("{data}");
}

// Fix 3: spawn_local (single-threaded, no Send required)
tokio::task::spawn_local(async {
    let data = Rc::new(42);
    tokio::time::sleep(d).await;
    println!("{data}");
});
```

**Skill to practice:** The compiler tells you exactly what's `!Send` across which `.await`. Scope `!Send` types tightly, or use their `Send` equivalents (`Arc` for `Rc`, `tokio::sync::Mutex` for `std::sync::Mutex` if held across `.await`).

---

## Skill 34: Cooperative Scheduling and Blocking

**Pattern:** Every `.await` is a yield point. If you never `.await`, you never yield — starving all other tasks on that thread.

```rust
// BAD: Blocks the executor thread
async fn blocking() {
    std::thread::sleep(Duration::from_secs(10)); // NO .await — blocks!
    std::fs::read_to_string("big.csv").unwrap(); // Sync I/O — blocks!
    (0..1_000_000_000).sum::<u64>();              // CPU-bound — blocks!
}

// GOOD: Use async equivalents or spawn_blocking
async fn non_blocking() {
    tokio::time::sleep(Duration::from_secs(10)).await;

    let data = tokio::task::spawn_blocking(|| {
        std::fs::read_to_string("big.csv").unwrap()
    }).await.unwrap();

    let sum = tokio::task::spawn_blocking(|| {
        (0..1_000_000_000).sum::<u64>()
    }).await.unwrap();
}
```

**Tokio's budget system:** Even for I/O that's always ready, tokio forces a yield after ~128 operations per poll to prevent task starvation.

**Skill to practice:** Never use `std::thread::sleep`, `std::fs::*`, or CPU-heavy loops inside async. Use `tokio::time::sleep`, `tokio::fs::*`, and `spawn_blocking` respectively.

---

## Skill 35: `AsyncIterator` (Stream) — Async Version of Iterator

**Pattern:** Mirrors `Iterator` with `poll_next` instead of `next`. Same "skinny required, fat provided" design.

```rust
pub trait AsyncIterator {
    type Item;
    fn poll_next(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Option<Self::Item>>;

    fn size_hint(&self) -> (usize, Option<usize>) { (0, None) }
}
```

**The parallel:**
```
Iterator::next(&mut self)             → Option<Item>        // blocking
AsyncIterator::poll_next(Pin, cx)     → Poll<Option<Item>>  // non-blocking
```

**Practical stream processing (via `futures::StreamExt`):**
```rust
use futures::StreamExt;

let results: Vec<Response> = request_stream
    .map(|req| fetch(req))       // Stream of Futures
    .buffer_unordered(10)         // Run up to 10 concurrently
    .collect()                    // Gather into Vec
    .await;
```

**Skill to practice:** Think of streams as "async iterators." The same combinator intuitions (map, filter, fold, collect) apply. Use `StreamExt` from `futures` for the full API.

---

## Skill 36: `IntoFuture` — The Builder Pattern for Async

**Pattern:** Like `IntoIterator` enables `for x in expr`, `IntoFuture` enables `.await` on types that aren't futures themselves.

```rust
pub trait IntoFuture {
    type Output;
    type IntoFuture: Future<Output = Self::Output>;
    fn into_future(self) -> Self::IntoFuture;
}

// Blanket impl: every Future is already IntoFuture
impl<F: Future> IntoFuture for F { ... }
```

**The killer use case — builder pattern:**
```rust
impl IntoFuture for RequestBuilder {
    type Output = Result<Response, Error>;
    type IntoFuture = Pin<Box<dyn Future<Output = Self::Output> + Send>>;

    fn into_future(self) -> Self::IntoFuture {
        Box::pin(async move { execute(self).await })
    }
}

// No explicit .send() needed:
let response = client
    .get("https://example.com")
    .header("Authorization", "Bearer token")
    .await?;  // IntoFuture kicks in here
```

**Skill to practice:** Implement `IntoFuture` on builder types to make `.await` trigger execution. Users get a clean, chainable API without a terminal `.send()` or `.execute()`.

---

## Skill 37: Structured Concurrency and Task Management

**Pattern:** Prefer structured concurrency (`join!`) over unstructured spawning (`spawn`). Use `JoinSet` for dynamic task groups.

**The spectrum:**
```rust
// Most structured → least structured:
let a = work_a().await;                              // Sequential
let (a, b) = tokio::join!(work_a(), work_b());       // Concurrent, scoped
let mut set = JoinSet::new(); set.spawn(work());      // Spawned, grouped
tokio::spawn(fire_and_forget());                      // Unstructured
```

**`JoinSet` — structured spawning with automatic cleanup:**
```rust
use tokio::task::JoinSet;

async fn fetch_all(urls: Vec<String>) -> Vec<String> {
    let mut set = JoinSet::new();
    for url in urls {
        set.spawn(async move { reqwest::get(&url).await.unwrap().text().await.unwrap() });
    }

    let mut results = Vec::new();
    while let Some(res) = set.join_next().await {
        results.push(res.unwrap());
    }
    results
}
// When JoinSet is dropped, all remaining tasks are aborted (RAII!)
```

**Error handling with spawned tasks — the double-Result:**
```rust
match tokio::spawn(fallible_work()).await {
    Ok(Ok(data))    => { /* success */ }
    Ok(Err(app_err)) => { /* application error */ }
    Err(join_err)    => { /* task panicked or was cancelled */ }
}
```

**Skill to practice:** `join!` > `JoinSet` > `spawn`. Each step down trades structure for flexibility. Only reach for `spawn` when you genuinely need task independence or `'static` decoupling.

---

## Skill 38: Async Anti-Patterns

### 1. Holding locks across `.await`
```rust
// BAD: Lock held during I/O — other tasks blocked for seconds
let mut guard = state.lock().await;
guard.data = fetch_remote().await;

// GOOD: Minimize lock scope
let data = fetch_remote().await;
state.lock().await.data = data;
```

### 2. Blocking in async context
```rust
// BAD                              // GOOD
std::thread::sleep(d);              tokio::time::sleep(d).await;
std::fs::read_to_string(p);        tokio::fs::read_to_string(p).await;
heavy_computation();                spawn_blocking(|| heavy_computation()).await;
```

### 3. Unnecessary boxing
```rust
// Only box when needed:
// ✓ Trait objects: fn() -> Pin<Box<dyn Future + Send>>
// ✓ Recursive async: Box::pin(recursive_call())
// ✗ Regular composition: just .await, the compiler inlines it
```

### 4. Recreating futures in `select!` loops
```rust
// BAD: New sleep every iteration
loop {
    tokio::select! {
        msg = rx.recv() => { /* ... */ }
        _ = tokio::time::sleep(Duration::from_secs(30)) => { cleanup(); }
    }
}

// GOOD: Create once, reset on fire
let sleep = tokio::time::sleep(Duration::from_secs(30));
tokio::pin!(sleep);
loop {
    tokio::select! {
        msg = rx.recv() => { /* ... */ }
        _ = &mut sleep => {
            cleanup();
            sleep.as_mut().reset(Instant::now() + Duration::from_secs(30));
        }
    }
}
```

**Skill to practice:** Use `clippy::await_holding_lock`. Use `tokio-console` to spot blocking tasks. Review `select!` loops for cancel safety and future recreation.

---

## Skill 39: Pinning Patterns in Practice

### Stack pinning with `pin!`
```rust
use std::pin::pin;

let fut = some_async_fn();
let mut fut = pin!(fut);  // Pin<&mut impl Future>, pinned to the stack
// `fut` can never be moved again — the macro shadows the binding
```

### Safe field projection with `pin-project`
```rust
use pin_project::pin_project;

#[pin_project]
struct Timeout<F> {
    #[pin]
    future: F,           // Structurally pinned — accessed as Pin<&mut F>
    deadline: Instant,   // NOT pinned — accessed as &mut Instant
}

impl<F: Future> Future for Timeout<F> {
    type Output = Option<F::Output>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project(); // Safe projection!
        // this.future: Pin<&mut F>
        // this.deadline: &mut Instant

        if Instant::now() >= *this.deadline {
            return Poll::Ready(None);
        }
        this.future.poll(cx).map(Some)
    }
}
```

**When pinning matters:**

| Type | Needs pinning? | Why |
|------|---------------|-----|
| `i32`, `String`, `Vec<T>` | No (`Unpin`) | No self-references |
| `async fn` return type | Yes (`!Unpin`) | Self-referential across `.await` |
| `Box<dyn Future>` | No | `Box` is `Unpin` regardless |

**Skill to practice:** Use `pin!` for local futures. Use `pin-project` for custom `Future` structs. Raw unsafe pin projection is almost never needed in application code.

---

## Skill 40: Async Trait Patterns

**Before Rust 1.75:** `async fn` in traits was impossible. The workaround:
```rust
// Manual desugaring (what async-trait crate does)
trait Service {
    fn call(&self, req: Request)
        -> Pin<Box<dyn Future<Output = Response> + Send + '_>>;
}
```

**After Rust 1.75:** Native support, but with nuances:
```rust
// Works natively:
trait Service {
    async fn call(&self, req: Request) -> Response;
}

// But the returned future is NOT automatically Send.
// For spawning compatibility, explicitly bound it:
trait SendService: Send + Sync {
    fn call(&self, req: Request) -> impl Future<Output = Response> + Send + '_;
}
```

**Dynamic dispatch still needs boxing:**
```rust
trait DynService {
    fn call(&self, req: Request)
        -> Pin<Box<dyn Future<Output = Response> + Send + '_>>;
}
// Can use: &dyn DynService, Box<dyn DynService>
```

**Skill to practice:** Use native `async fn` in traits when possible. Add `+ Send` bounds explicitly when the trait will be used with `tokio::spawn`. Use `Pin<Box<dyn Future>>` when you need dynamic dispatch.

---

## Quick Reference: When to Apply Each Skill

| Situation | Skill |
|-----------|-------|
| Understanding async fundamentals | #26 Future trait |
| Debugging async execution | #27 State machine desugaring |
| Writing manual Future impls | #28 Waker contract, #29 poll_fn |
| Running multiple async ops | #30 join! |
| Timeout / race patterns | #31 select! |
| Understanding runtime architecture | #32 Executor/reactor split |
| Fixing `Send` errors | #33 Send bound |
| Avoiding blocking | #34 Cooperative scheduling |
| Processing async data streams | #35 AsyncIterator/Stream |
| Builder pattern + async | #36 IntoFuture |
| Task management | #37 Structured concurrency |
| Code review for async | #38 Anti-patterns |
| Custom Future types | #39 Pinning patterns |
| Traits with async methods | #40 Async trait patterns |

---

## Change Log

- **2026-02-14**: Initial version — derived from analysis of Future, Poll, Waker, Context, async/await desugaring, streams, and runtime architecture patterns.
