# Concurrency Patterns in Rust

Patterns for writing correct, performant concurrent Rust — from std primitives to lock-free data structures, Rayon data parallelism, and real-world composition patterns. Continues from Skills 1-92.

---

## Skill 93: Thread Spawning & JoinHandle

**Pattern**: `thread::spawn` takes a `Send + 'static` closure, returns a `JoinHandle<T>` for the return value.

```rust
use std::thread;

// Capture return value
let handle: thread::JoinHandle<i32> = thread::spawn(|| {
    expensive_computation()
});
let result = handle.join().expect("thread panicked");

// Builder for configuration
thread::Builder::new()
    .name("worker-0".into())
    .stack_size(4 * 1024 * 1024)
    .spawn(move || { /* ... */ })
    .expect("failed to spawn thread");
```

**Key details**:
- `join()` returns `Result<T, Box<dyn Any + Send>>` — the `Err` contains the panic payload
- Dropping a `JoinHandle` without joining **detaches** the thread (it keeps running, but you lose the result)
- `Builder::spawn` returns `io::Result<JoinHandle<T>>` (can fail if OS refuses), while `thread::spawn` panics on failure

**Skill to practice**: Spawn 4 threads that each compute a partial sum of a slice, then join and combine results.

---

## Skill 94: Scoped Threads — Borrowing Without `'static`

**Pattern**: `thread::scope` (Rust 1.63+) lets spawned threads borrow from the caller's stack frame. All threads are joined before the scope exits.

```rust
use std::thread;

let mut data = vec![1, 2, 3, 4];
let mut results = vec![0u64; 4];

thread::scope(|s| {
    for (i, val) in data.iter().enumerate() {
        let slot = &mut results[i]; // borrow from stack!
        s.spawn(move || {
            *slot = (*val as u64) * (*val as u64);
        });
    }
}); // all threads joined here
// data and results are usable again
assert_eq!(results, vec![1, 4, 9, 16]);
```

**When to use scoped vs regular threads**:
| Use `thread::scope` | Use `thread::spawn` |
|---------------------|---------------------|
| Threads are short-lived | Threads outlive the caller |
| Need to borrow local data | Owning/moving data is fine |
| Fork-join parallelism | Long-running background services |

**Panic behavior**: If any scoped thread panics, the scope itself panics after joining all remaining threads.

**Skill to practice**: Parallelize a data processing function using `thread::scope`, borrowing input and output slices without cloning.

---

## Skill 95: Mutex & RwLock — RAII Locking

**Pattern**: `Mutex<T>` gives exclusive access via an RAII guard. `RwLock<T>` allows multiple readers OR one writer.

```rust
use std::sync::{Arc, Mutex, RwLock};
use std::thread;

// Mutex: exclusive access
let counter = Arc::new(Mutex::new(0u64));
let handles: Vec<_> = (0..10).map(|_| {
    let counter = Arc::clone(&counter);
    thread::spawn(move || {
        let mut guard = counter.lock().unwrap();
        *guard += 1;
        // guard dropped here → lock released
    })
}).collect();
for h in handles { h.join().unwrap(); }

// RwLock: many readers, one writer
let config = Arc::new(RwLock::new(AppConfig::default()));
let cfg = config.read().unwrap();   // shared read access
drop(cfg);
let mut cfg = config.write().unwrap(); // exclusive write access
cfg.port = 8080;
```

**Poisoning**: If a thread panics while holding the lock, the Mutex becomes "poisoned". Subsequent `lock()` calls return `Err(PoisonError)`. Recover with `into_inner()` or use `parking_lot::Mutex` (no poisoning).

**Anti-patterns**:
- Holding a `MutexGuard` across `.await` — the guard is not `Send`
- Recursive locking — Rust's Mutex is not reentrant, will deadlock
- Holding locks during I/O — blocks all other threads

**Early unlock**:
```rust
let result = {
    let guard = data.lock().unwrap();
    guard.iter().sum::<i32>()
    // guard dropped at end of block
};
// or: drop(guard) explicitly
```

**When to prefer RwLock**: Read-heavy workloads with infrequent writes and non-trivial critical sections.

**Skill to practice**: Build a shared config store with `Arc<RwLock<Config>>` where reader threads vastly outnumber writer threads.

---

## Skill 96: Condvar & Barrier — Thread Coordination

**Pattern**: `Condvar` lets a thread sleep until a condition becomes true. `Barrier` synchronizes N threads at a rendezvous point.

```rust
use std::sync::{Arc, Mutex, Condvar, Barrier};

// Condvar: wait for a condition
let pair = Arc::new((Mutex::new(false), Condvar::new()));

// Notifier
let (lock, cvar) = &*pair;
*lock.lock().unwrap() = true;
cvar.notify_one();

// Waiter — ALWAYS wait in a loop (spurious wakeups!)
let mut ready = lock.lock().unwrap();
while !*ready {
    ready = cvar.wait(ready).unwrap();
}
// Or use the convenience method:
let _guard = cvar.wait_while(lock.lock().unwrap(), |ready| !*ready).unwrap();

// Barrier: all threads wait for each other
let barrier = Arc::new(Barrier::new(4));
// In each of 4 threads:
barrier.wait(); // blocks until all 4 arrive
```

**Condvar internals**: `wait()` atomically releases the Mutex and sleeps. On wakeup, it re-acquires the Mutex. A Condvar must always be used with the **same** Mutex.

**Barrier details**: Reusable across "generations". One thread per generation gets `is_leader() == true` (useful for a single merge step). No timeout support — if a thread panics before reaching the barrier, remaining threads block forever.

**Skill to practice**: Implement a bounded blocking queue using `Mutex` + `Condvar` (block on push when full, block on pop when empty).

---

## Skill 97: Channels — mpsc and crossbeam

**Pattern**: Message passing decouples producers from consumers. Choose bounded vs unbounded carefully.

```rust
use std::sync::mpsc;

// Unbounded (async) — sends never block, but can OOM
let (tx, rx) = mpsc::channel();

// Bounded (sync) — sends block when full = backpressure
let (tx, rx) = mpsc::sync_channel(64);

// Rendezvous (bound=0) — direct hand-off, both sides synchronize
let (tx, rx) = mpsc::sync_channel::<String>(0);

// Multiple producers
for i in 0..5 {
    let tx = tx.clone();
    thread::spawn(move || { tx.send(i).unwrap(); });
}
drop(tx); // CRITICAL: drop original sender so channel closes!

// Iterate until all senders are dropped
for msg in rx {
    process(msg);
}
```

**crossbeam channels** — faster, MPMC, with `select!`:
```rust
use crossbeam::channel::{self, select, tick, after};
use std::time::Duration;

let (tx, rx) = channel::bounded::<Task>(256); // MPMC

let heartbeat = tick(Duration::from_secs(60));
let deadline = after(Duration::from_secs(300));

loop {
    select! {
        recv(rx) -> msg => { if let Ok(task) = msg { handle(task); } }
        recv(heartbeat) -> _ => { run_maintenance(); }
        recv(deadline) -> _ => { break; }
    }
}
```

**The #1 channel bug**: Forgetting to `drop(tx)` (the original sender) after cloning to producers. The channel never closes, and consumers hang on `recv()` forever.

**Decision**:
| Need | Use |
|------|-----|
| Simple MPSC | `std::sync::mpsc` |
| MPMC, select, performance | `crossbeam::channel` |
| Backpressure | Bounded channel (sync_channel or bounded) |

**Skill to practice**: Build a log aggregator with multiple producer threads sending log lines through a bounded channel to a single writer thread.

---

## Skill 98: Atomics & Memory Ordering

**Pattern**: Lock-free atomic operations on primitive types. The building blocks for all higher-level primitives.

```rust
use std::sync::atomic::{AtomicU64, AtomicBool, Ordering, fence};

static COUNTER: AtomicU64 = AtomicU64::new(0);

// Simple operations
COUNTER.fetch_add(1, Ordering::Relaxed);
let val = COUNTER.load(Ordering::Relaxed);
COUNTER.store(42, Ordering::Relaxed);

// Compare-and-swap loop (for complex updates)
let mut current = COUNTER.load(Ordering::Relaxed);
loop {
    let new = if current < 1000 { current + 1 } else { break; };
    match COUNTER.compare_exchange_weak(
        current, new,
        Ordering::AcqRel,  // success
        Ordering::Acquire,  // failure
    ) {
        Ok(_) => break,
        Err(actual) => current = actual, // retry
    }
}
```

**Memory ordering cheat sheet**:

| Ordering | Use when |
|----------|----------|
| `Relaxed` | Counter/flag only, no other data depends on it |
| `Acquire` (loads) | Reading a flag that guards other data |
| `Release` (stores) | Publishing data another thread will read |
| `AcqRel` (RMW) | CAS that both reads and publishes |
| `SeqCst` | Multiple independent atomics must appear globally ordered |
| **Not sure?** | **Use `SeqCst` — correct but slower** |

**The Acquire/Release handshake**:
```rust
static DATA_READY: AtomicBool = AtomicBool::new(false);
static mut DATA: u64 = 0;

// Producer: write data, then Release-store the flag
unsafe { DATA = 42; }
DATA_READY.store(true, Ordering::Release);

// Consumer: Acquire-load the flag, then read data
while !DATA_READY.load(Ordering::Acquire) {}
let val = unsafe { DATA }; // guaranteed to see 42
```

**`compare_exchange` vs `compare_exchange_weak`**: Weak may fail spuriously on ARM (LL/SC) but is cheaper in loops. Use weak in loops, strong for single-shot operations.

**Skill to practice**: Implement a lock-free one-shot channel using `AtomicBool` + `UnsafeCell` with correct Acquire/Release ordering.

---

## Skill 99: Send & Sync — Compile-Time Thread Safety

**Pattern**: Two auto-traits enforce thread safety at compile time. `Send` = can be moved across threads. `Sync` = can be shared (`&T`) across threads.

**The key relationship**: `T: Sync` iff `&T: Send`.

| Type | Send? | Sync? | Why |
|------|-------|-------|-----|
| `i32`, `String`, `Vec<T>` | Yes | Yes | No interior mutability |
| `Rc<T>` | **No** | **No** | Non-atomic reference count |
| `Arc<T>` (T: Send+Sync) | Yes | Yes | Atomic reference count |
| `Cell<T>`, `RefCell<T>` | Yes | **No** | Interior mutability without sync |
| `Mutex<T>` (T: Send) | Yes | Yes | Synchronization makes sharing safe |
| `MutexGuard<'_, T>` | **No** | Yes | POSIX requires unlock on same thread |
| `*mut T` | **No** | **No** | Raw pointers carry no guarantees |

**How it works**: `thread::spawn` requires `F: Send + 'static`. If you capture a `Rc` in the closure, the compiler rejects it:
```rust
let r = Rc::new(42);
thread::spawn(move || println!("{}", r)); // ERROR: Rc is not Send
// Fix: use Arc
let r = Arc::new(42);
thread::spawn(move || println!("{}", r)); // OK
```

**Manual implementations** (rare, requires `unsafe`):
```rust
struct SharedBuffer { ptr: *mut u8, len: usize }
// SAFETY: buffer is heap-allocated, not aliased, access externally synchronized
unsafe impl Send for SharedBuffer {}
unsafe impl Sync for SharedBuffer {}
```

**Skill to practice**: Try to send various types across threads and predict which ones the compiler will reject. Understand why for each case.

---

## Skill 100: Once, OnceLock, LazyLock — Initialize Exactly Once

**Pattern**: Three levels of abstraction for one-time initialization.

```rust
use std::sync::{OnceLock, LazyLock};

// LazyLock: simplest, Deref-based access
static REGEX: LazyLock<regex::Regex> = LazyLock::new(|| {
    regex::Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap()
});
fn is_date(s: &str) -> bool { REGEX.is_match(s) }

// OnceLock: when init value comes from elsewhere (config, CLI args)
static DB_POOL: OnceLock<DbPool> = OnceLock::new();
fn init_pool(url: &str) { DB_POOL.set(DbPool::connect(url)).ok(); }
fn pool() -> &'static DbPool {
    DB_POOL.get_or_init(|| DbPool::connect("postgres://localhost/db"))
}
```

**Choosing between them**:
| | `Once` | `OnceLock` | `LazyLock` |
|-|--------|-----------|-----------|
| Stores value | No | Yes | Yes |
| Requires unsafe | Yes | No | No |
| Init source | Closure | `set()` or `get_or_init()` | Fixed closure |
| Access | Manual | `.get()` | Deref (`*val`) |

**Rule of thumb**: `LazyLock` for global constants. `OnceLock` when init depends on runtime config. `Once` only for very low-level scenarios.

**Skill to practice**: Replace a `lazy_static!` usage with `LazyLock`, or use `OnceLock` for a database pool that gets its URL from CLI args.

---

## Skill 101: Lock-Free Patterns & crossbeam

**Pattern**: Use CAS loops for lock-free updates. Use crossbeam's epoch-based GC for safe memory reclamation.

```rust
use crossbeam::queue::{ArrayQueue, SegQueue};

// ArrayQueue: bounded, fixed capacity, cache-friendly
let q: ArrayQueue<WorkItem> = ArrayQueue::new(1024);
q.push(item).ok();     // Returns Err(item) if full
let item = q.pop();     // Returns Option<T>

// SegQueue: unbounded, allocates segments on demand
let q: SegQueue<WorkItem> = SegQueue::new();
q.push(item);           // never fails (may allocate)
let item = q.pop();     // Returns Option<T>
```

| Property | `ArrayQueue` | `SegQueue` |
|----------|-------------|-----------|
| Bounded | Yes | No |
| Allocation | Zero after `new()` | Segments on demand |
| Backpressure | Natural (push fails when full) | None (potential OOM) |
| Cache | Contiguous, cache-friendly | Segments scattered |

**crossbeam Deque — work stealing** (what Rayon is built on):
```rust
use crossbeam::deque::{Worker, Injector, Stealer};

fn find_task(local: &Worker<Task>, global: &Injector<Task>, stealers: &[Stealer<Task>]) -> Option<Task> {
    // 1. Pop from own deque (fast, no contention)
    local.pop().or_else(|| {
        // 2. Steal batch from global queue
        std::iter::repeat_with(|| global.steal_batch_and_pop(local))
            .find(|s| !s.is_retry())
            .and_then(|s| s.success())
    }).or_else(|| {
        // 3. Steal from a sibling
        stealers.iter()
            .map(|s| s.steal())
            .find(|s| !s.is_retry())
            .and_then(|s| s.success())
    })
}
```

**When to use lock-free vs Mutex**:
| Favor lock-free | Favor Mutex |
|----------------|-------------|
| Very high contention, many readers | Low to moderate contention |
| Tiny critical section (pointer swap) | Complex state update |
| Real-time / priority inversion concerns | Normal applications |

**Default advice**: Use `Mutex` until profiling proves it's the bottleneck.

**Skill to practice**: Replace an `Arc<Mutex<VecDeque>>` work queue with `crossbeam::ArrayQueue` and benchmark the difference.

---

## Skill 102: Rayon — Data Parallelism

**Pattern**: Rayon turns sequential iterators into parallel ones with work-stealing. Change `.iter()` to `.par_iter()`.

```rust
use rayon::prelude::*;

// Parallel map + collect
let results: Vec<f64> = data.par_iter()
    .filter(|&&x| x > 0.0)
    .map(|&x| expensive_computation(x))
    .collect();

// Parallel sort
let mut data = vec![5, 3, 1, 4, 2];
data.par_sort_unstable();

// Fork-join: two tasks potentially in parallel
fn parallel_quicksort<T: Ord + Send>(slice: &mut [T]) {
    if slice.len() <= 32 { slice.sort(); return; }
    let pivot = partition(slice);
    let (left, right) = slice.split_at_mut(pivot);
    rayon::join(
        || parallel_quicksort(left),
        || parallel_quicksort(right),
    );
}

// fold + reduce for per-thread accumulators
let histogram: HashMap<char, usize> = text.par_chars()
    .fold(|| HashMap::new(), |mut map, ch| {
        *map.entry(ch).or_insert(0) += 1; map
    })
    .reduce(|| HashMap::new(), |mut a, b| {
        for (k, v) in b { *a.entry(k).or_insert(0) += v; } a
    });
```

**When par_iter helps vs hurts**:
| Helps | Hurts |
|-------|-------|
| Per-element work > 1μs | Trivial per-element work |
| Dataset > 1000 elements | Small collections |
| Uniform work per element | Heavy I/O or allocation |

**Critical rule**: Never block inside a Rayon task (no `Mutex::lock()`, no I/O waits, no sleep). A blocked worker starves the entire pool.

**Custom pool** (isolate from global):
```rust
let pool = rayon::ThreadPoolBuilder::new()
    .num_threads(4)
    .build().unwrap();
pool.install(|| {
    data.par_iter().map(process).collect()
});
```

| Construct | Use when |
|-----------|----------|
| `par_iter()` | Process a collection in parallel |
| `join(a, b)` | Exactly 2 parallel tasks, can borrow |
| `scope(\|s\| s.spawn(...))` | Dynamic number of tasks, borrowing |
| `spawn(closure)` | Fire-and-forget (`'static`) |

**Skill to practice**: Take a sequential data pipeline and convert it to use `par_iter()`. Benchmark with `criterion` to find the crossover point where parallelism pays off.

---

## Skill 103: Shared State — Arc<Mutex<T>> vs Channels vs DashMap

**Pattern**: Choose the right shared state mechanism based on access patterns.

**Arc<Mutex<T>>** — the workhorse:
```rust
let state = Arc::new(Mutex::new(HashMap::new()));
// Clone Arc, lock briefly, release quickly
let snapshot = state.lock().unwrap().clone(); // clone out
let result = expensive_computation(&snapshot); // compute outside lock
state.lock().unwrap().insert(key, result);     // lock briefly to write
```

**DashMap** — concurrent HashMap with per-shard locking:
```rust
use dashmap::DashMap;
let map: DashMap<String, Vec<u64>> = DashMap::new();
map.insert("key".into(), vec![1, 2, 3]);
map.entry("key".into()).and_modify(|v| v.push(4)).or_insert_with(|| vec![4]);
```

**ArcSwap** — RCU for read-heavy, write-rare data:
```rust
use arc_swap::ArcSwap;
let config: ArcSwap<Config> = ArcSwap::from_pointee(Config::default());
let current = config.load();         // near-zero cost read
let mut new = (**current).clone();   // clone + modify + swap
new.setting = 42;
config.store(Arc::new(new));
```

**Decision matrix**:
| Scenario | Use |
|----------|-----|
| Simple shared value, moderate contention | `Arc<Mutex<T>>` |
| Read-heavy HashMap, high contention | `DashMap` |
| Config read on every request, rare writes | `ArcSwap` |
| Complex state transitions, many services | Channels (actor model) |

| Factor | `Arc<Mutex<T>>` | Channels |
|--------|-----------------|----------|
| Coupling | Tight (all threads know data shape) | Loose (only message types) |
| Debugging | Potential deadlocks | Easier to trace |
| Latency | Lower (no message copy) | Higher (message passing) |
| Complexity at scale | Grows quickly | Grows linearly |

**Skill to practice**: Build the same cache with `Arc<Mutex<HashMap>>`, `DashMap`, and an actor (channel + dedicated thread). Benchmark under read-heavy and write-heavy loads.

---

## Skill 104: Concurrency Anti-Patterns

**1. Deadlock via lock ordering**:
```rust
// DEADLOCK: Thread A locks X then Y, Thread B locks Y then X
// FIX: always lock in consistent order (by ID, address, etc.)
fn transfer(from: &Mutex<u64>, from_id: usize, to: &Mutex<u64>, to_id: usize) {
    let (first, second) = if from_id < to_id { (from, to) } else { (to, from) };
    let mut a = first.lock().unwrap();
    let mut b = second.lock().unwrap();
    // ...
}
```

**2. Unbounded channels as memory leaks**:
```rust
// BAD: unbounded, fast producer → OOM
let (tx, rx) = mpsc::channel();
// GOOD: bounded with backpressure
let (tx, rx) = mpsc::sync_channel(1024);
```

**3. Holding locks across `.await`**:
```rust
// BAD: lock held while task is suspended
async fn bad(state: &Mutex<State>) {
    let mut guard = state.lock().unwrap();
    let data = fetch().await; // lock held across await!
    guard.data = data;
}
// GOOD: release before await
async fn good(state: &Mutex<State>) {
    let old = state.lock().unwrap().data.clone();
    let data = fetch().await;
    state.lock().unwrap().data = data;
}
// Or use tokio::sync::Mutex (designed to be held across await)
```

**4. Busy-waiting**:
```rust
// BAD: burns CPU
while !flag.load(Ordering::Acquire) {}
// GOOD: use Condvar, park/unpark, or crossbeam Parker
```

**5. DashMap deadlock**: Never hold multiple DashMap refs simultaneously (two keys might be in the same shard).

**Skill to practice**: Audit an existing concurrent codebase for these anti-patterns. Fix any you find.

---

## Skill 105: Producer-Consumer Pattern

**Pattern**: Producers generate work items, consumers process them, channels provide decoupling and backpressure.

```rust
use crossbeam::channel;
use std::thread;

fn producer_consumer(n_producers: usize, n_consumers: usize) {
    let (tx, rx) = channel::bounded::<WorkItem>(256);

    thread::scope(|s| {
        for p in 0..n_producers {
            let tx = tx.clone();
            s.spawn(move || {
                for i in 0..1000 {
                    tx.send(WorkItem { id: p * 1000 + i }).ok();
                }
            });
        }
        drop(tx); // CRITICAL: close channel when producers done

        for _ in 0..n_consumers {
            let rx = rx.clone(); // crossbeam channels are MPMC
            s.spawn(move || {
                while let Ok(item) = rx.recv() { process(item); }
            });
        }
    });
}
```

**Scaling**: Add more producers or consumers independently. Bounded channel naturally throttles fast producers.

**Skill to practice**: Build a concurrent image processing pipeline — producers read files, consumers apply filters, a collector writes results.

---

## Skill 106: Pipeline Pattern — Staged Processing

**Pattern**: Each stage has its own thread(s) and input/output channel. Different stages can have different parallelism.

```rust
use crossbeam::channel;
use std::thread;

fn pipeline(input: &[RawData]) -> Vec<Processed> {
    let (tx1, rx1) = channel::bounded::<&RawData>(64);
    let (tx2, rx2) = channel::bounded::<Validated>(64);
    let (tx3, rx3) = channel::bounded::<Processed>(64);

    thread::scope(|s| {
        // Stage 1: Parse (1 thread, fast)
        s.spawn(|| {
            for item in input { tx1.send(item).unwrap(); }
        });

        // Stage 2: Validate (4 threads, CPU-bound)
        for _ in 0..4 {
            let rx = rx1.clone();
            let tx = tx2.clone();
            s.spawn(move || {
                while let Ok(raw) = rx.recv() {
                    if let Some(valid) = validate(raw) {
                        tx.send(valid).unwrap();
                    }
                }
            });
        }
        drop(rx1); drop(tx2);

        // Stage 3: Enrich (2 threads, I/O-bound)
        for _ in 0..2 {
            let rx = rx2.clone();
            let tx = tx3.clone();
            s.spawn(move || {
                while let Ok(item) = rx.recv() {
                    tx.send(enrich(item)).unwrap();
                }
            });
        }
        drop(rx2); drop(tx3);

        // Collect results
        rx3.iter().collect()
    })
}
```

**Key insight**: Bounded channels between stages absorb bursts. Each stage's parallelism is tuned independently based on its bottleneck.

**Ordering**: Pipelines do NOT preserve input order. If order matters, include an index and sort at the end.

**Skill to practice**: Build a 3-stage pipeline with different parallelism per stage. Measure throughput and find which stage is the bottleneck.

---

## Skill 107: Actor Model Using Channels

**Pattern**: Each actor owns its state, communicates only via messages. No shared mutable state, no locks, no deadlocks.

```rust
use crossbeam::channel::{self, Sender, Receiver};

enum CacheMsg {
    Get { key: String, reply: Sender<Option<String>> },
    Set { key: String, value: String },
    Shutdown,
}

struct CacheActor {
    rx: Receiver<CacheMsg>,
    store: HashMap<String, String>,
}

impl CacheActor {
    fn run(mut self) {
        while let Ok(msg) = self.rx.recv() {
            match msg {
                CacheMsg::Get { key, reply } => {
                    let _ = reply.send(self.store.get(&key).cloned());
                }
                CacheMsg::Set { key, value } => {
                    self.store.insert(key, value);
                }
                CacheMsg::Shutdown => break,
            }
        }
    }
}

// Handle provides a typed API hiding the channel
#[derive(Clone)]
struct CacheHandle { tx: Sender<CacheMsg> }

impl CacheHandle {
    fn spawn() -> Self {
        let (tx, rx) = channel::bounded(256);
        std::thread::spawn(|| CacheActor { rx, store: HashMap::new() }.run());
        Self { tx }
    }

    fn get(&self, key: &str) -> Option<String> {
        let (reply_tx, reply_rx) = channel::bounded(1);
        self.tx.send(CacheMsg::Get { key: key.into(), reply: reply_tx }).ok()?;
        reply_rx.recv().ok().flatten()
    }

    fn set(&self, key: &str, value: &str) {
        let _ = self.tx.send(CacheMsg::Set { key: key.into(), value: value.into() });
    }
}
```

**Advantages**: No locks, no deadlocks, easy to test (send messages, check replies), easy to distribute across processes later.

**Disadvantages**: Request-reply has double channel latency, one-shot reply channels allocate per request.

**Skill to practice**: Implement a key-value store actor with Get, Set, Delete, and List operations. Add a second actor that periodically snapshots state to disk.

---

## Skill 108: Graceful Shutdown

**Pattern**: Use an atomic flag + Condvar (or channel close) to signal all threads to wind down cleanly.

```rust
use std::sync::{Arc, atomic::{AtomicBool, Ordering}, Mutex, Condvar};
use std::time::Duration;

struct Worker {
    shutdown: AtomicBool,
    notify: (Mutex<()>, Condvar),
}

impl Worker {
    fn run(self: &Arc<Self>) -> std::thread::JoinHandle<()> {
        let w = Arc::clone(self);
        std::thread::spawn(move || {
            loop {
                if w.shutdown.load(Ordering::Acquire) {
                    println!("shutting down gracefully");
                    return;
                }
                do_work();
                // Sleep interruptibly using Condvar timeout
                let (lock, cvar) = &w.notify;
                let guard = lock.lock().unwrap();
                let _ = cvar.wait_timeout(guard, Duration::from_secs(1));
            }
        })
    }

    fn stop(&self) {
        self.shutdown.store(true, Ordering::Release);
        self.notify.1.notify_all(); // wake sleeping threads
    }
}
```

**Channel-based shutdown** (simpler): Just drop all senders — receivers get `Err` and exit their loops naturally.

**Skill to practice**: Add graceful shutdown to a multi-threaded server so that in-flight requests complete but no new ones are accepted.

---

## Quick Reference

| Skill | Pattern | One-Liner |
|-------|---------|-----------|
| 93 | Thread Spawning | `spawn` for `'static`, `join()` for results |
| 94 | Scoped Threads | `thread::scope` — borrow from stack, auto-join |
| 95 | Mutex & RwLock | RAII guards, poisoning, hold briefly |
| 96 | Condvar & Barrier | Wait for conditions, synchronize N threads |
| 97 | Channels | Bounded for backpressure, drop sender to close |
| 98 | Atomics | Relaxed for counters, Acquire/Release for data |
| 99 | Send & Sync | Compile-time thread safety via auto-traits |
| 100 | OnceLock/LazyLock | LazyLock for globals, OnceLock for runtime config |
| 101 | Lock-Free & crossbeam | ArrayQueue, SegQueue, work-stealing deque |
| 102 | Rayon | `par_iter()`, `join()`, never block in pool |
| 103 | Shared State | Arc<Mutex>, DashMap, ArcSwap — pick by pattern |
| 104 | Anti-Patterns | Lock ordering, unbounded channels, lock across await |
| 105 | Producer-Consumer | Bounded channel + scope, drop sender to close |
| 106 | Pipeline | Staged processing, per-stage parallelism |
| 107 | Actor Model | Thread + channel + message enum, typed handle |
| 108 | Graceful Shutdown | Atomic flag + Condvar, or drop senders |

---

## Change Log

| Date | Change |
|------|--------|
| 2026-02-16 | Initial version — Skills 93-108 from stdlib/ecosystem analysis |
