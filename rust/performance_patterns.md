# Performance Patterns in Rust

Patterns for writing fast, allocation-aware Rust code — from profiling and benchmarking to zero-copy, SIMD, cache-friendly data structures, and compile-time computation. Continues from Skills 1-186.

---

## Skill 187: Profiling Fundamentals

**Pattern**: Always measure before optimizing. The profiling workflow is: measure, identify hotspot, optimize, verify.

**cargo bench with criterion** (cross-reference Skill 121):
```rust
// benches/hot_path.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_process(c: &mut Criterion) {
    c.bench_function("process_data", |b| {
        let data = generate_test_data(10_000);
        b.iter(|| process(black_box(&data)))
    });
}

criterion_group!(benches, bench_process);
criterion_main!(benches);
```

**perf on Linux** — sampling profiler for CPU-bound code:
```bash
# Record a profile (run your binary under perf)
perf record --call-graph dwarf ./target/release/my_app

# View the report interactively
perf report

# Quick stats: cache misses, branch mispredictions, IPC
perf stat ./target/release/my_app
```

**Flamegraphs** — visualize where time is spent:
```bash
cargo install flamegraph

# Generate a flamegraph SVG (needs perf on Linux)
cargo flamegraph --bin my_app -- --input large_file.txt

# For benchmarks
cargo flamegraph --bench my_benchmark -- --bench
```

The SVG is interactive — wider bars mean more time. Look for wide bars that shouldn't be there.

**cargo instruments on macOS**:
```bash
cargo install cargo-instruments
cargo instruments --bin my_app -t Allocations
cargo instruments --bin my_app -t "Time Profiler"
```

**Heap profiling with DHAT**:
```rust
// In your binary, temporarily:
use dhat;

#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    let _profiler = dhat::Profiler::new_heap();
    // ... your code ...
    // On drop, prints allocation summary
}
```

```bash
cargo run --release  # produces dhat-heap.json
# Open in https://nnethercote.github.io/dh_view/dh_view.html
```

**The profiling workflow**:
1. **Measure** — get a baseline with criterion or `time`
2. **Identify** — use flamegraph/perf to find hotspots
3. **Optimize** — change the hotspot (algorithm, allocation, data layout)
4. **Verify** — re-run benchmark to confirm improvement

**Common mistake**: Profiling debug builds. Always profile with `--release`. Debug builds have no optimizations and give misleading results.

**Skill to practice**: Profile a real binary with `cargo flamegraph`. Find the hottest function. Optimize it and verify the improvement with criterion.

---

## Skill 188: Allocation-Aware Programming

**Pattern**: Understand where allocations happen and avoid unnecessary ones. Stack is fast, heap is slow.

**Stack vs heap — when allocations happen**:
```rust
let x = 42;                    // stack — free
let s = "hello";               // stack pointer to static data — free
let v = vec![1, 2, 3];         // heap allocation
let s = String::from("hello"); // heap allocation
let b = Box::new(42);          // heap allocation
```

**`String` vs `&str`, `Vec<T>` vs `&[T]`** — borrow instead of owning:
```rust
// BAD: forces caller to allocate
fn process(data: String) { /* ... */ }

// GOOD: accepts borrowed data — no allocation required
fn process(data: &str) { /* ... */ }

// BAD: takes ownership of a vec
fn sum(nums: Vec<i32>) -> i32 { nums.iter().sum() }

// GOOD: borrows a slice
fn sum(nums: &[i32]) -> i32 { nums.iter().sum() }
```

**`Cow<'a, str>` for conditional allocation**:
```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.contains('\t') {
        // Only allocate when we need to modify
        Cow::Owned(input.replace('\t', "    "))
    } else {
        // No allocation — just borrow
        Cow::Borrowed(input)
    }
}

let result = normalize("no tabs here"); // zero allocations
let result = normalize("has\ttabs");     // one allocation
```

**SmallVec for small-buffer optimization**:
```rust
use smallvec::SmallVec;

// Stores up to 4 elements inline (on the stack)
// Only heap-allocates if more than 4 elements
let mut v: SmallVec<[i32; 4]> = SmallVec::new();
v.push(1); // no heap allocation
v.push(2); // no heap allocation
v.push(3); // no heap allocation
v.push(4); // no heap allocation
v.push(5); // NOW it heap-allocates
```

**Tracking allocations with `#[global_allocator]`**:
```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering};

struct CountingAllocator;

static ALLOC_COUNT: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for CountingAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        ALLOC_COUNT.fetch_add(1, Ordering::Relaxed);
        unsafe { System.alloc(layout) }
    }
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        unsafe { System.dealloc(ptr, layout) }
    }
}

#[global_allocator]
static A: CountingAllocator = CountingAllocator;

#[test]
fn test_no_allocs_in_hot_path() {
    let before = ALLOC_COUNT.load(Ordering::Relaxed);
    hot_path(&data);
    let after = ALLOC_COUNT.load(Ordering::Relaxed);
    assert_eq!(before, after, "hot path should not allocate");
}
```

**Pre-allocate with capacity**:
```rust
// BAD: multiple reallocations as vec grows
let mut results = Vec::new();
for item in input {
    results.push(process(item));
}

// GOOD: single allocation up front
let mut results = Vec::with_capacity(input.len());
for item in input {
    results.push(process(item));
}

// ALSO GOOD: collect knows the size from the iterator
let results: Vec<_> = input.iter().map(process).collect();
```

**Skill to practice**: Replace `String` parameters with `&str` in a codebase. Use `Cow<str>` where you conditionally modify strings. Measure the allocation difference with DHAT.

---

## Skill 189: Zero-Copy Patterns

**Pattern**: Process data in place without copying it. The fastest copy is no copy.

**Borrowing instead of cloning**:
```rust
// BAD: clones the string just to look at it
fn first_word(s: &str) -> String {
    s.split_whitespace().next().unwrap_or("").to_string()
}

// GOOD: returns a reference into the original string
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}
```

**`bytes::Bytes` for reference-counted byte buffers**:
```rust
use bytes::Bytes;

let data = Bytes::from(vec![1, 2, 3, 4, 5]);

// slice() creates a new Bytes sharing the same underlying memory
let header = data.slice(0..2);  // no copy
let body = data.slice(2..);     // no copy

// Both `header` and `body` are reference-counted views into `data`
```

**Zero-copy parsing with nom**:
```rust
use nom::{bytes::complete::tag, character::complete::alpha1, IResult};

// Parses from &str without allocating — returns slices into the input
fn parse_greeting(input: &str) -> IResult<&str, &str> {
    let (input, _) = tag("hello ")(input)?;
    let (input, name) = alpha1(input)?;
    Ok((input, name))
}

let (_, name) = parse_greeting("hello world").unwrap();
assert_eq!(name, "world"); // `name` is a slice of the original input
```

**Memory-mapped file I/O with memmap2**:
```rust
use memmap2::Mmap;
use std::fs::File;

fn count_lines(path: &std::path::Path) -> std::io::Result<usize> {
    let file = File::open(path)?;
    let mmap = unsafe { Mmap::map(&file)? };
    // `mmap` is a &[u8] backed by the OS page cache — no read() into a buffer
    Ok(mmap.iter().filter(|&&b| b == b'\n').count())
}
```

**Serde zero-copy deserialization** (cross-reference Skill 146):
```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct LogEntry<'a> {
    #[serde(borrow)]
    message: &'a str,    // borrows from the input JSON string
    #[serde(borrow)]
    source: &'a str,
    level: u8,
}

let json = r#"{"message": "hello", "source": "app", "level": 1}"#;
let entry: LogEntry = serde_json::from_str(json).unwrap();
// entry.message points into `json` — no String allocation
```

**When zero-copy is worth it vs when it overcomplicates**:
| Use zero-copy | Avoid zero-copy |
|--------------|----------------|
| Parsing large files (GB+) | Small config files |
| High-throughput network processing | One-time startup parsing |
| Hot loops processing many records | Code where lifetimes would infect the whole API |
| Read-only access to data | When you need to mutate the data anyway |

**Skill to practice**: Rewrite a parser that allocates Strings to return `&str` references instead. Benchmark both versions.

---

## Skill 190: Iterator & Collect Optimization

**Pattern**: Iterator chains are zero-cost abstractions. They compile to the same code as hand-written loops — often better, because the compiler sees the whole pipeline.

**Iterator chains compile to loops**:
```rust
// These produce identical assembly:
let sum: i32 = (0..1000).filter(|x| x % 2 == 0).map(|x| x * x).sum();

let mut sum = 0i32;
for x in 0..1000 {
    if x % 2 == 0 {
        sum += x * x;
    }
}
```

**`collect()` with size hints — pre-allocation**:
```rust
// Vec::collect() calls size_hint() and pre-allocates
// For ranges and slices, size_hint() is exact
let v: Vec<i32> = (0..1000).collect(); // one allocation of 1000 elements

// For filter, size_hint() gives (0, Some(1000)) — upper bound only
// Vec allocates for the upper bound, which may waste memory
let evens: Vec<i32> = (0..1000).filter(|x| x % 2 == 0).collect();
```

**`extend()` vs repeated `push()`**:
```rust
let mut results = Vec::new();

// BAD: no size hint, may reallocate multiple times
for item in source {
    results.push(transform(item));
}

// GOOD: extend uses size_hint for pre-allocation
results.extend(source.iter().map(transform));
```

**`chain()` vs manual concatenation**:
```rust
// BAD: allocates a new vec, copies both
let mut combined = first.clone();
combined.extend(second.iter());

// GOOD: lazy concatenation, no intermediate allocation
let total: i32 = first.iter().chain(second.iter()).sum();

// If you need a Vec, chain + collect is clean
let combined: Vec<_> = first.iter().chain(second.iter()).collect();
```

**Avoiding intermediate allocations with `filter_map`**:
```rust
// OK but two passes through the adapter chain
let results: Vec<Output> = inputs.iter()
    .filter(|x| x.is_valid())
    .map(|x| x.process())
    .collect();

// Slightly better: filter_map fuses the check and transform
let results: Vec<Output> = inputs.iter()
    .filter_map(|x| if x.is_valid() { Some(x.process()) } else { None })
    .collect();

// Best for Option-returning transforms:
let results: Vec<Output> = inputs.iter()
    .filter_map(|x| x.try_process().ok())
    .collect();
```

**`flat_map` vs `map().flatten()`**:
```rust
// Equivalent — flat_map is just syntactic sugar
let words: Vec<&str> = lines.iter().flat_map(|l| l.split_whitespace()).collect();
let words: Vec<&str> = lines.iter().map(|l| l.split_whitespace()).flatten().collect();
```

**Skill to practice**: Rewrite a loop that pushes to a Vec into an iterator chain with `collect()`. Verify with `cargo bench` that performance is equivalent or better.

---

## Skill 191: Cache-Friendly Data Structures

**Pattern**: Modern CPUs are fast; memory is slow. Cache-friendly data layout can give 10x speedups without algorithmic changes.

**Struct of Arrays (SoA) vs Array of Structs (AoS)**:
```rust
// AoS — each entity is contiguous (good if you access ALL fields per entity)
struct Particle { x: f32, y: f32, z: f32, mass: f32 }
let particles: Vec<Particle> = vec![/* ... */];

// SoA — each field is contiguous (good if you access ONE field at a time)
struct Particles {
    x: Vec<f32>,
    y: Vec<f32>,
    z: Vec<f32>,
    mass: Vec<f32>,
}

// SoA is faster for "update all positions" because x, y, z are
// contiguous in memory — one cache line holds many x values
fn update_positions(p: &mut Particles, dt: f32) {
    for i in 0..p.x.len() {
        p.x[i] += p.vx[i] * dt;
        p.y[i] += p.vy[i] * dt;
    }
}
```

**Data-oriented design — keep hot data together**:
```rust
// BAD: Entity has hot data (position) mixed with cold data (name, description)
struct Entity {
    name: String,           // cold — accessed rarely
    description: String,    // cold
    x: f32,                 // hot — accessed every frame
    y: f32,                 // hot
    health: f32,            // hot
    biography: String,      // cold
}

// GOOD: separate hot and cold data
struct HotData { x: f32, y: f32, health: f32 }
struct ColdData { name: String, description: String, biography: String }

let hot: Vec<HotData> = vec![/* ... */];   // tight, cache-friendly
let cold: Vec<ColdData> = vec![/* ... */]; // accessed only when needed
```

**`Vec<T>` is cache-friendly, `LinkedList<T>` is not**:
```rust
// Vec<T>: contiguous memory, prefetcher-friendly
// Iterating 1M elements: ~1ms

// LinkedList<T>: each node separately allocated, pointer-chasing
// Iterating 1M elements: ~10ms (cache miss per node)

// Almost NEVER use LinkedList. Use VecDeque for double-ended needs.
```

**`#[repr(C)]` and `#[repr(packed)]` for layout control**:
```rust
// Default Rust layout: compiler reorders fields to minimize padding
struct A { a: u8, b: u64, c: u8 } // Rust may reorder to [b, a, c, padding]

// repr(C): fields in declaration order (for FFI or when you need control)
#[repr(C)]
struct B { a: u8, b: u64, c: u8 } // 24 bytes (with padding)

// repr(packed): no padding (may cause unaligned access — slower on some archs)
#[repr(packed)]
struct C { a: u8, b: u64, c: u8 } // 10 bytes exactly
```

**Measuring cache misses with `perf stat`**:
```bash
perf stat -e cache-references,cache-misses,L1-dcache-load-misses \
    ./target/release/my_app

# Look at the cache-miss ratio:
# < 5% — good
# > 20% — data layout problem
```

**Skill to practice**: Convert an AoS layout to SoA for a hot loop. Benchmark both with criterion and check cache misses with `perf stat`.

---

## Skill 192: String Performance

**Pattern**: Strings are the most common allocation in many programs. Choose the right string type and avoid allocating in hot paths.

**Choosing the right string type**:
| Type | Heap alloc? | Use when |
|------|:-----------:|----------|
| `&str` | No | Borrowing existing string data |
| `String` | Yes | Owning mutable string data |
| `Cow<str>` | Sometimes | Might borrow, might own |
| `Arc<str>` | Once, shared | Multiple owners, immutable |
| `&'static str` | No | Compile-time constant strings |

**`format!` allocates — avoid in hot loops**:
```rust
// BAD: allocates a new String every iteration
for item in items {
    log(&format!("processing: {}", item.id));
}

// GOOD: write! to a pre-allocated buffer
let mut buf = String::with_capacity(256);
for item in items {
    buf.clear();
    use std::fmt::Write;
    write!(&mut buf, "processing: {}", item.id).unwrap();
    log(&buf);
}
```

**`itoa` / `ryu` crates for fast integer/float formatting**:
```rust
// itoa: 2-3x faster than format!("{}", n) for integers
let mut buf = itoa::Buffer::new();
let s: &str = buf.format(12345);

// ryu: 2-4x faster than format!("{}", f) for floats
let mut buf = ryu::Buffer::new();
let s: &str = buf.format(3.14159);
```

**String interning for repeated strings**:
```rust
use std::collections::HashSet;

struct Interner {
    pool: HashSet<Box<str>>,
}

impl Interner {
    fn new() -> Self { Self { pool: HashSet::new() } }

    fn intern(&mut self, s: &str) -> &str {
        if !self.pool.contains(s) {
            self.pool.insert(s.into());
        }
        // SAFETY: the reference is valid as long as the Interner lives
        // In practice, use a crate like `string-interner` or `lasso`
        self.pool.get(s).unwrap()
    }
}
```

For production use, reach for the `lasso` or `string-interner` crate.

**`compact_str` / `smol_str` for small string optimization**:
```rust
use compact_str::CompactString;

// CompactString stores strings up to 24 bytes inline (no heap allocation)
let s = CompactString::from("hello"); // inline — no heap
let s = CompactString::from("this is a longer string that exceeds 24 bytes"); // heap

// Same API as String: push_str, +, format, etc.
// Drop-in replacement in many cases
```

**Skill to practice**: Profile a string-heavy application with DHAT. Replace `format!` calls in hot paths with `write!` to a buffer. Measure the improvement.

---

## Skill 193: SIMD & Vectorization

**Pattern**: Process multiple data elements in a single CPU instruction. The compiler auto-vectorizes many loops, but you can help it or use explicit SIMD.

**Auto-vectorization — when the compiler does it for you**:
```rust
// The compiler will auto-vectorize this into SIMD instructions
fn sum(data: &[f32]) -> f32 {
    data.iter().sum()
}

// Helps auto-vectorization:
// - Simple loops over slices
// - No early returns or branches in the loop body
// - Known alignment and length
// - Operations that map to SIMD instructions (add, mul, min, max)
```

**Checking vectorization**:
```bash
# View assembly for a function
cargo install cargo-show-asm
cargo asm my_crate::sum

# Look for SIMD instructions: vaddps, vmulps, vfmadd (AVX)
# or addps, mulps (SSE)
```

**`std::simd` (nightly) — portable SIMD**:
```rust
#![feature(portable_simd)]
use std::simd::prelude::*;

fn dot_product_simd(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len());

    let (a_chunks, a_remainder) = a.as_chunks::<8>();
    let (b_chunks, b_remainder) = b.as_chunks::<8>();

    let mut sum = f32x8::splat(0.0);
    for (a_chunk, b_chunk) in a_chunks.iter().zip(b_chunks) {
        let a_vec = f32x8::from_slice(a_chunk);
        let b_vec = f32x8::from_slice(b_chunk);
        sum += a_vec * b_vec;
    }

    let mut result = sum.reduce_sum();
    for (a, b) in a_remainder.iter().zip(b_remainder) {
        result += a * b;
    }
    result
}
```

**Stable SIMD with the `wide` crate**:
```rust
use wide::f32x8;

fn sum_wide(data: &[f32]) -> f32 {
    let chunks = data.chunks_exact(8);
    let remainder = chunks.remainder();

    let mut acc = f32x8::ZERO;
    for chunk in chunks {
        let v = f32x8::from(chunk);
        acc += v;
    }

    let mut total: f32 = acc.as_array_ref().iter().sum();
    total += remainder.iter().sum::<f32>();
    total
}
```

**Common SIMD-friendly patterns**:
```rust
// Process arrays in chunks — avoids branch per element
fn threshold(data: &mut [f32], limit: f32) {
    // Process in chunks of 8 (SIMD-friendly)
    for chunk in data.chunks_exact_mut(8) {
        for val in chunk {
            *val = val.min(limit);
        }
    }
    // Handle remainder
    for val in data.chunks_exact_mut(8).into_remainder() {
        *val = val.min(limit);
    }
}
```

**When SIMD helps**:
| Good candidates | Poor candidates |
|----------------|----------------|
| Bulk array math (dot products, sums) | Branchy code with data-dependent paths |
| String searching / byte scanning | Linked list traversal |
| Image processing, audio processing | Small arrays (< 32 elements) |
| Hashing, checksums | Code dominated by memory latency |

**Skill to practice**: Write a sum function, check if the compiler auto-vectorizes it with `cargo asm`. If not, rewrite with explicit SIMD using the `wide` crate and benchmark both.

---

## Skill 194: Concurrency for Performance

**Pattern**: Use parallelism for CPU-bound work, async for I/O-bound work. Measure to confirm speedup — parallelism has overhead.

**Rayon `par_iter` for data parallelism** (cross-reference Skill 102):
```rust
use rayon::prelude::*;

// Sequential
let results: Vec<_> = data.iter().map(|x| expensive_compute(x)).collect();

// Parallel — one-line change
let results: Vec<_> = data.par_iter().map(|x| expensive_compute(x)).collect();

// Parallel sort
let mut data = vec![/* ... */];
data.par_sort_unstable();
```

**Amdahl's law — when parallelism helps vs hurts**:
```
Speedup = 1 / (S + P/N)
  S = serial fraction
  P = parallel fraction (S + P = 1)
  N = number of cores

If 50% of your code is serial, max speedup with ∞ cores is 2x.
If 10% is serial, max speedup is 10x.
```

```rust
// BAD: overhead exceeds savings for small work
let sum: i32 = small_vec.par_iter().sum(); // rayon overhead > compute

// GOOD: parallelize only when work is substantial
if data.len() > 10_000 {
    data.par_iter().map(|x| expensive(x)).collect()
} else {
    data.iter().map(|x| expensive(x)).collect()
}
```

**Lock contention as a bottleneck**:
```rust
use std::sync::{Arc, Mutex};

// BAD: all threads contend on a single lock
let counter = Arc::new(Mutex::new(0u64));
handles.iter().for_each(|_| {
    let c = counter.clone();
    // Every thread locks, increments, unlocks — serialized!
});

// GOOD: sharded counters — each thread has its own, merge at the end
use std::sync::atomic::{AtomicU64, Ordering};
let counter = AtomicU64::new(0);
// Or use thread-local accumulators:
let local_sum: u64 = data.par_iter().map(|x| process(x)).sum();
```

**`crossbeam` for low-overhead concurrent data structures**:
```rust
use crossbeam::queue::ArrayQueue;

// Lock-free bounded queue
let queue = Arc::new(ArrayQueue::new(1024));

// Producer
let q = queue.clone();
std::thread::spawn(move || {
    for i in 0..1000 {
        while q.push(i).is_err() { std::thread::yield_now(); }
    }
});

// Consumer
while let Some(item) = queue.pop() {
    process(item);
}
```

**Async for I/O-bound, threads/rayon for CPU-bound**:
| Workload | Tool | Why |
|----------|------|-----|
| HTTP requests | tokio/async | Thousands of concurrent I/O waits |
| Image processing | rayon | CPU-bound, data-parallel |
| File parsing (many files) | rayon | CPU-bound per file |
| Database queries | tokio/async | Waiting on network I/O |
| Mixed | tokio::spawn_blocking | Bridge async and CPU-bound |

**Skill to practice**: Take a sequential data processing pipeline, parallelize with rayon, and measure the actual speedup. Compare with the theoretical Amdahl's law prediction.

---

## Skill 195: Compile-Time Computation

**Pattern**: Move work from runtime to compile time. Zero runtime cost for anything the compiler can evaluate.

**`const fn` for compile-time evaluation**:
```rust
const fn fibonacci(n: u32) -> u64 {
    let mut a = 0u64;
    let mut b = 1u64;
    let mut i = 0;
    while i < n {
        let temp = b;
        b = a + b;
        a = temp;
        i += 1;
    }
    a
}

// Evaluated at compile time — zero runtime cost
const FIB_20: u64 = fibonacci(20);

// Also works in const contexts
const LOOKUP: [u64; 10] = {
    let mut table = [0u64; 10];
    let mut i = 0;
    while i < 10 {
        table[i] = fibonacci(i as u32);
        i += 1;
    }
    table
};
```

**Lookup tables computed at compile time**:
```rust
const CRC32_TABLE: [u32; 256] = {
    let mut table = [0u32; 256];
    let mut i = 0u32;
    while i < 256 {
        let mut crc = i;
        let mut j = 0;
        while j < 8 {
            if crc & 1 != 0 {
                crc = (crc >> 1) ^ 0xEDB88320;
            } else {
                crc >>= 1;
            }
            j += 1;
        }
        table[i as usize] = crc;
        i += 1;
    }
    table
};

fn crc32(data: &[u8]) -> u32 {
    let mut crc = !0u32;
    for &byte in data {
        let index = ((crc ^ byte as u32) & 0xFF) as usize;
        crc = (crc >> 8) ^ CRC32_TABLE[index];
    }
    !crc
}
```

**`include_bytes!` / `include_str!` for embedding data**:
```rust
// Embed file contents directly into the binary at compile time
const SCHEMA: &str = include_str!("../schema.sql");
const LOGO: &[u8] = include_bytes!("../assets/logo.png");
const DEFAULT_CONFIG: &str = include_str!("../defaults.toml");
```

**`const` generics for compile-time parameters**:
```rust
fn fixed_sum<const N: usize>(arr: &[f64; N]) -> f64 {
    let mut sum = 0.0;
    let mut i = 0;
    while i < N {
        sum += arr[i];
        i += 1;
    }
    sum
}

// Compiler generates specialized code for each N — may unroll the loop
let a = fixed_sum(&[1.0, 2.0, 3.0]);       // N=3
let b = fixed_sum(&[1.0, 2.0, 3.0, 4.0]);  // N=4
```

**`build.rs` for code generation**:
```rust
// build.rs — runs at compile time, generates Rust code
fn main() {
    let out_dir = std::env::var("OUT_DIR").unwrap();
    let dest = std::path::Path::new(&out_dir).join("generated.rs");

    let mut code = String::new();
    code.push_str("pub const PRIMES: &[u32] = &[");
    for p in primes_up_to(1000) {
        code.push_str(&format!("{p}, "));
    }
    code.push_str("];\n");

    std::fs::write(dest, code).unwrap();
    println!("cargo:rerun-if-changed=build.rs");
}

// In src/lib.rs:
include!(concat!(env!("OUT_DIR"), "/generated.rs"));
```

**Tradeoff: compile time vs runtime**:
| Compile-time | Runtime |
|-------------|---------|
| `const fn` — limited to const-safe operations | Full Rust language |
| Increases compile time | Zero compile overhead |
| Result baked into binary | Computed on each run |
| Great for lookup tables, constants | Needed for dynamic data |

**Skill to practice**: Convert a runtime-computed lookup table into a `const` block. Verify with `cargo asm` that the table appears as static data.

---

## Skill 196: Binary Size Optimization

**Pattern**: Smaller binaries mean faster downloads, less memory, and faster startup. Rust binaries can be large by default — here's how to shrink them.

**`cargo bloat` to find what's taking space**:
```bash
cargo install cargo-bloat

# Show largest functions
cargo bloat --release -n 20

# Show largest crates
cargo bloat --release --crates

# Show with regex filter
cargo bloat --release --filter "serde"
```

**Release profile for size optimization**:
```toml
# Cargo.toml
[profile.release]
opt-level = "z"      # optimize for size (vs "3" for speed)
lto = true           # link-time optimization — removes unused code across crates
strip = true         # remove debug symbols from binary
panic = "abort"      # remove unwinding machinery (~10% size reduction)
codegen-units = 1    # slower compile, better optimization
```

**Size comparison** (typical CLI tool):
| Configuration | Size |
|:-------------|-----:|
| Debug | ~50 MB |
| Release (defaults) | ~8 MB |
| Release + LTO | ~5 MB |
| Release + LTO + strip | ~3 MB |
| Release + LTO + strip + opt-level=z + panic=abort | ~2 MB |

**Feature flags to exclude unused dependencies**:
```toml
[dependencies]
# BAD: pulls in everything
reqwest = "0.12"

# GOOD: only what you need
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls"] }

# Disable default features of serde_json if you don't need arbitrary precision
serde_json = { version = "1", default-features = false, features = ["std"] }
```

**`#[no_std]` for minimal binaries**:
```rust
// For extreme size constraints (embedded, WASM)
#![no_std]
#![no_main]

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

// Can produce binaries under 10 KB
```

**Checking what's in the binary**:
```bash
# List symbols sorted by size
nm -S --size-sort target/release/my_app | tail -20

# View sections
size target/release/my_app
```

**Skill to practice**: Take a Rust binary, measure its size, then apply each optimization from the profile settings. Record the size after each change.

---

## Skill 197: Benchmarking Best Practices

**Pattern**: Benchmarks must be statistically rigorous, reproducible, and resistant to common pitfalls. Bad benchmarks lead to bad decisions.

**Criterion setup and interpretation** (cross-reference Skill 121, deeper here):
```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion, BatchSize};

fn bench_hashmap_insert(c: &mut Criterion) {
    c.bench_function("hashmap_insert_1000", |b| {
        b.iter_batched(
            // Setup: create a fresh HashMap each iteration
            || std::collections::HashMap::with_capacity(1000),
            // Routine: the code being measured
            |mut map| {
                for i in 0..1000 {
                    map.insert(i, i);
                }
                black_box(map)
            },
            BatchSize::SmallInput,
        )
    });
}

criterion_group!(benches, bench_hashmap_insert);
criterion_main!(benches);
```

**`black_box` to prevent dead code elimination**:
```rust
// BAD: compiler may optimize this away entirely
b.iter(|| {
    let v: Vec<i32> = (0..1000).collect();
    v.len() // compiler knows this is 1000 — removes the Vec
});

// GOOD: black_box prevents optimization
b.iter(|| {
    let v: Vec<i32> = (0..1000).collect();
    black_box(&v);
});
```

**Benchmarking pitfalls**:
```rust
// PITFALL 1: Cold caches
// First iteration is always slower. Criterion handles this with warmup,
// but if your benchmark has setup that fills caches, be aware.

// PITFALL 2: Branch predictor warmup
// Sorted data benchmarks faster than random data due to branch prediction.
// Be consistent in your input generation.

// PITFALL 3: Measuring allocation, not computation
b.iter_batched(
    || generate_input(),    // setup (not measured)
    |input| process(input), // only this is measured
    BatchSize::SmallInput,
);
```

**Comparing before/after with criterion's regression detection**:
```bash
# Run benchmarks and save baseline
cargo bench -- --save-baseline before

# Make your changes...

# Run benchmarks and compare
cargo bench -- --baseline before

# Criterion reports:
# "Performance has improved by 15.2% (±2.1%)"
# or "Performance has regressed by 3.4% (±1.8%)"
```

**Micro-benchmarks vs macro-benchmarks**:
| Micro | Macro |
|-------|-------|
| Single function in isolation | End-to-end workflow |
| criterion/divan | `time`, custom harness |
| Good for comparing implementations | Good for real-world performance |
| Can be misleading (no cache pressure) | Captures system-level effects |
| Use for: "is A faster than B?" | Use for: "is this fast enough?" |

**Continuous benchmarking in CI**:
```yaml
# .github/workflows/bench.yml
- name: Run benchmarks
  run: cargo bench -- --output-format bencher | tee output.txt
- name: Store benchmark result
  uses: benchmark-action/github-action-benchmark@v1
  with:
    tool: 'cargo'
    output-file-path: output.txt
    alert-threshold: '120%'  # alert if 20% slower
```

**Skill to practice**: Set up criterion benchmarks with `iter_batched` and `--save-baseline`. Make a change and use regression detection to measure the impact.

---

## Skill 198: Performance Anti-Patterns & Decision Matrix

**Pattern**: Know the common performance traps and have a systematic approach to optimization.

**Premature optimization vs actual bottlenecks**:
```rust
// DON'T optimize this:
fn config_load() -> Config {
    serde_json::from_str(&std::fs::read_to_string("config.json").unwrap()).unwrap()
}
// It runs once at startup. Optimizing it saves microseconds nobody notices.

// DO optimize this:
fn process_request(req: &Request) -> Response {
    let data = format!("{}", req.body); // allocation in hot path!
    // ... called 10,000 times per second
}
```

**`.clone()` everywhere — when it's fine vs when it kills performance**:
```rust
// FINE: clone small/cheap types
let id = user.id;            // Copy
let flag = config.verbose;   // Copy
let name = user.name.clone(); // OK in cold path

// BAD: clone in hot loops
for item in &items {
    let owned = item.data.clone(); // allocates every iteration!
    process(&owned);
}

// FIX: borrow instead
for item in &items {
    process(&item.data); // zero allocations
}
```

**`HashMap` with bad hash functions**:
```rust
use std::collections::HashMap;

// Default HashMap uses SipHash — safe against DoS but not the fastest
let mut map = HashMap::new();

// For non-adversarial data, use a faster hasher
use rustc_hash::FxHashMap; // or ahash::AHashMap
let mut map: FxHashMap<u64, String> = FxHashMap::default();
// 2-5x faster for integer keys

// NEVER use FxHashMap for user-controlled keys (vulnerable to HashDoS)
```

**Unbounded allocations in hot loops**:
```rust
// BAD: allocates a new String every iteration
for line in reader.lines() {
    let line = line?;
    let upper = line.to_uppercase(); // new allocation
    output.write_all(upper.as_bytes())?;
}

// GOOD: reuse the buffer
let mut buf = String::new();
for line in reader.lines() {
    let line = line?;
    buf.clear();
    for c in line.chars() {
        buf.extend(c.to_uppercase());
    }
    output.write_all(buf.as_bytes())?;
}
```

**`Arc<Mutex<T>>` contention — alternatives**:
| Pattern | When to use |
|---------|------------|
| `Arc<Mutex<T>>` | Low contention, simple |
| `Arc<RwLock<T>>` | Many readers, few writers |
| `DashMap` | Concurrent HashMap (sharded internally) |
| `ArcSwap` | Read-heavy, rare updates |
| Channels | Message passing instead of shared state |
| `AtomicU64` etc. | Simple counters/flags |
| Thread-local + merge | Accumulation across threads |

**The decision matrix — when to optimize**:

```
1. Is it actually slow?
   NO  → Stop. Don't optimize.
   YES → Continue.

2. Have you profiled?
   NO  → Profile first. cargo flamegraph / perf / DHAT.
   YES → Continue.

3. Where is the bottleneck?
   ALGORITHM  → Fix the algorithm (O(n²) → O(n log n))
   ALLOCATION → Reduce/reuse allocations (Skills 188-189)
   CACHE      → Improve data layout (Skill 191)
   CPU        → SIMD or parallelism (Skills 193-194)
   I/O        → Async, batching, buffering

4. After optimizing:
   → Re-run benchmarks to verify improvement
   → If no improvement, revert the change
   → Document why the optimization exists
```

**"Make it work, make it right, make it fast" — in that order**:
1. **Work**: Get correct behavior with tests
2. **Right**: Clean code, good abstractions, no tech debt
3. **Fast**: Profile, identify bottleneck, optimize, verify

Optimized code is harder to read and maintain. Only optimize what the profiler tells you to.

**Skill to practice**: Audit a project for these anti-patterns. Profile it, find the actual bottleneck, fix it, and verify with benchmarks.

---

## Quick Reference

| Skill | Pattern | One-Liner |
|-------|---------|-----------|
| 187 | Profiling Fundamentals | perf, flamegraph, DHAT — measure before optimizing |
| 188 | Allocation-Aware Programming | Stack vs heap, Cow, SmallVec, pre-allocate with capacity |
| 189 | Zero-Copy Patterns | Borrow don't clone, bytes::Bytes, memmap2, serde zero-copy |
| 190 | Iterator & Collect Optimization | Zero-cost chains, size hints, extend vs push, filter_map |
| 191 | Cache-Friendly Data Structures | SoA vs AoS, hot/cold splitting, Vec over LinkedList |
| 192 | String Performance | &str over String, write! over format!, itoa/ryu, compact_str |
| 193 | SIMD & Vectorization | Auto-vectorization, cargo asm, std::simd, wide crate |
| 194 | Concurrency for Performance | Rayon par_iter, Amdahl's law, sharding, crossbeam |
| 195 | Compile-Time Computation | const fn, lookup tables, include_bytes!, build.rs codegen |
| 196 | Binary Size Optimization | cargo bloat, LTO, strip, panic=abort, feature flags |
| 197 | Benchmarking Best Practices | iter_batched, black_box, baselines, CI benchmarks |
| 198 | Anti-Patterns & Decision Matrix | Profile first, clone audit, hasher choice, optimize cycle |

---

## Change Log

| Date | Change |
|------|--------|
| 2026-02-20 | Initial version — Skills 187-198 from performance patterns analysis |
