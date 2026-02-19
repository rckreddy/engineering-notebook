# Rust Skills Index

A condensed reference of all Rust skills extracted from studying the standard library and ecosystem. Each skill has a number, pattern name, and one-liner — use the source document for full details and code examples.

**Usage**: Reference this file from any project's `CLAUDE.md` so the agent applies these patterns when writing code.

Example CLAUDE.md snippet for your projects:
```markdown
## Coding Standards
Follow the Rust skills documented in ~/code/engineering-notebook/rust/skills.md
When writing Rust code, apply the relevant patterns from that reference.
```

---

## Core Types & Traits (Skills 1-10)
**Source**: [stdlib_skills.md](stdlib_skills.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 1 | Skinny Required, Fat Provided | Traits: few required methods, many provided defaults built on them |
| 2 | try_fold Backbone | Build complex iterator methods on one powerful primitive |
| 3 | Ownership-Aware API Triads | Offer `as_ref` / `to_owned` / `into_inner` variants |
| 4 | Blanket Impl Cascades | Implement From → get Into, TryFrom, TryInto for free |
| 5 | Combinator Consistency | Option/Result share `map`, `and_then`, `unwrap_or_else` shape |
| 6 | Lazy Adapter Structs | Iterator adapters are structs — zero work until consumed |
| 7 | #[must_use] Guardrails | Force callers to handle return values |
| 8 | Strategic #[inline] | Inline small functions and trait methods at crate boundaries |
| 9 | AsRef/AsMut for Generics | Accept `impl AsRef<str>` instead of `&str` for flexibility |
| 10 | Documentation as Specification | Panics, Errors, Safety, Examples sections |

## Memory & Ownership (Skills 11-25)
**Source**: [memory_ownership.md](memory_ownership.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 11 | Stack Handle + Heap Data | Vec/String/Box: small stack struct owns heap allocation |
| 12 | Layered Architecture | Public API → safe core → unsafe foundation |
| 13 | Newtype Invariants | Wrap primitives to enforce constraints via type system |
| 14 | Amortized Growth | Double capacity on realloc — O(1) amortized push |
| 15 | RAII / Drop | Destructor runs automatically — for cleanup, locks, resources |
| 16 | Deref Coercion | `Vec<T>` → `&[T]`, `String` → `&str` transparently |
| 17 | Unsafe Encapsulation | Safe shell wrapping unsafe core — users never see unsafe |
| 18 | Smart Pointer Selection | Box (single owner) → Rc (shared) → Arc (threaded) |
| 19 | Atomic Ordering | Relaxed/Acquire/Release/SeqCst — match to your needs |
| 20 | UnsafeCell | The primitive behind all interior mutability |
| 21 | Cell vs RefCell | Cell for Copy types, RefCell for borrows with runtime check |
| 22 | Interior Mutability Matrix | Cell → RefCell → Mutex → RwLock by context |
| 23 | Pin for Self-Referential | Pin prevents moves, enabling self-referential structs |
| 24 | Weak References | Break reference cycles in Rc/Arc graphs |
| 25 | PhantomData | Encode ownership/variance without runtime cost |

## Async Patterns (Skills 26-40)
**Source**: [async_patterns.md](async_patterns.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 26 | Future Trait | `poll(cx) → Poll<T>` — lazy, does nothing until polled |
| 27 | State Machine Desugaring | async fn compiles to an enum of states |
| 28 | Waker Contract | Wake signals the executor to re-poll a future |
| 29 | poll_fn | Create futures from closures for quick adapters |
| 30 | join! | Run futures concurrently, all must complete |
| 31 | select! | Race futures, first one wins (cancel safety matters) |
| 32 | Executor/Reactor Split | Executor polls futures, reactor watches I/O |
| 33 | Send Bounds | Futures crossing threads must be Send |
| 34 | Cooperative Scheduling | Yield periodically to avoid starving other tasks |
| 35 | AsyncIterator/Stream | Async version of Iterator — `poll_next` |
| 36 | IntoFuture | Enable `my_builder.await` syntax |
| 37 | Structured Concurrency | Scope-based spawning ensures cleanup |
| 38 | Async Anti-Patterns | No blocking in async, no holding locks across await |
| 39 | Pinning in Practice | Box::pin for heap, pin! macro for stack |
| 40 | Async Traits | Use `async fn` in traits (Rust 1.75+) |

## Error Handling (Skills 41-50)
**Source**: [error_handling.md](error_handling.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 41 | Error Trait | `Display` + `Debug` + optional `source()` chain |
| 42 | Enum Errors | One variant per failure mode |
| 43 | From/? Conversion | Implement From to enable `?` operator |
| 44 | thiserror | Derive Error for library error types |
| 45 | anyhow | Catch-all for application error handling |
| 46 | Error Boundaries | thiserror in libraries, anyhow in applications |
| 47 | Result Combinator Fluency | `map_err`, `context`, `with_context` |
| 48 | Panic vs Result | Panic for bugs, Result for expected failures |
| 49 | Error Reporting | Chain display for users, Debug for developers |
| 50 | Complete Error Strategy | Type hierarchy + boundaries + reporting |

## API Design (Skills 51-62)
**Source**: [api_design.md](api_design.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 51 | Builder Pattern | Construct complex types step by step |
| 52 | Typestate Pattern | Encode state machines in the type system |
| 53 | Extension Traits | Add methods to foreign types |
| 54 | Sealed Traits | Public trait, private implementation |
| 55 | Cow (Clone-on-Write) | Borrow when possible, clone only when needed |
| 56 | Fallible Constructors | `new()` → infallible, `try_new()` → Result |
| 57 | Naming Conventions | `as_` (cheap ref), `to_` (expensive), `into_` (consuming) |
| 58 | Generic Parameter Design | Accept broad, return narrow |
| 59 | #[non_exhaustive] | Forward-compatible enums and structs |
| 60 | Derive Strategy | Derive common traits, respect orphan rule |
| 61 | Documentation Sections | Examples, Panics, Errors, Safety |
| 62 | Prelude Pattern | Re-export common items in a prelude module |

## Unsafe Rust (Skills 63-76)
**Source**: [unsafe_patterns.md](unsafe_patterns.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 63 | Five Superpowers | Deref raw ptrs, call unsafe fns, access mutables, impl unsafe traits, union fields |
| 64 | Raw Pointers | `*const T` / `*mut T` — no aliasing guarantees |
| 65 | NonNull | Non-zero pointer with covariance |
| 66 | MaybeUninit | Defer initialization without UB |
| 67 | Transmute | Reinterpret bits — last resort |
| 68 | mem::forget / ManuallyDrop | Skip destructors when needed |
| 69 | SAFETY Comments | Document invariants on every unsafe block |
| 70 | Stacked Borrows | The aliasing model unsafe code must follow |
| 71 | Send/Sync Contracts | Unsafe trait impls require proof of thread safety |
| 72 | FFI Patterns | repr(C), CString, catch_unwind, opaque types |
| 73 | Variance & PhantomData | Control subtyping behavior of generic types |
| 74 | Common Unsoundness | Iterator invalidation, use-after-free, aliasing violations |
| 75 | Miri Testing | Detect UB at runtime with `cargo +nightly miri test` |
| 76 | Encapsulation Checklist | Safe API wrapping unsafe internals — verification steps |

## Embedded & no_std (Skills 77-92)
**Source**: [embedded_patterns.md](embedded_patterns.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 77 | no_std Fundamentals | `core` always, `alloc` optional, define panic handler |
| 78 | embedded-hal Traits | Write drivers against traits, not concrete HALs |
| 79 | Typestate GPIO | Encode pin modes in types, transitions consume old state |
| 80 | Heapless Collections | `Vec<T, N>` — fixed capacity, no allocator |
| 81 | Memory Layout | Linker script defines FLASH/RAM, sections map to regions |
| 82 | Interrupt Safety | Atomics → critical_section → heapless queues → Embassy |
| 83 | Singleton Peripherals | `Peripherals::take()` → HAL → type-safe pins |
| 84 | defmt Logging | Compile-time format strings, minimal runtime cost |
| 85 | Embassy Fundamentals | Async tasks = zero-alloc state machines on bare metal |
| 86 | Embassy HAL | Peripheral ops are async, DMA-backed, CPU sleeps |
| 87 | Signals & Channels | `Signal` (latest value), `Channel` (buffered queue) |
| 88 | Power Management | Every `await` is a sleep opportunity; use Ticker |
| 89 | ESP32 Patterns | `esp-hal` + `esp-hal-embassy` for no_std async |
| 90 | RP2040 Patterns | `embassy-rp` with PIO, USB, dual-core |
| 91 | Testing Embedded | Separate logic from hardware, use embedded-hal-mock |
| 92 | Embedded Anti-Patterns | No blocking in async, no `static mut`, clear IRQ flags |

## Concurrency (Skills 93-108)
**Source**: [concurrency_patterns.md](concurrency_patterns.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 93 | Thread Spawning | `spawn` for `'static`, `join()` for results |
| 94 | Scoped Threads | `thread::scope` — borrow from stack, auto-join |
| 95 | Mutex & RwLock | RAII guards, poisoning, hold briefly |
| 96 | Condvar & Barrier | Wait for conditions, synchronize N threads |
| 97 | Channels | Bounded for backpressure, drop sender to close |
| 98 | Atomics & Ordering | Relaxed for counters, Acquire/Release for data |
| 99 | Send & Sync | Compile-time thread safety via auto-traits |
| 100 | OnceLock / LazyLock | LazyLock for globals, OnceLock for runtime config |
| 101 | Lock-Free & crossbeam | ArrayQueue, SegQueue, work-stealing deque |
| 102 | Rayon | `par_iter()`, `join()`, never block in pool |
| 103 | Shared State | Arc<Mutex>, DashMap, ArcSwap — pick by pattern |
| 104 | Anti-Patterns | Lock ordering, unbounded channels, lock across await |
| 105 | Producer-Consumer | Bounded channel + scope, drop sender to close |
| 106 | Pipeline | Staged processing, per-stage parallelism |
| 107 | Actor Model | Thread + channel + message enum, typed handle |
| 108 | Graceful Shutdown | Atomic flag + Condvar, or drop senders |

## Testing (Skills 109-124)
**Source**: [testing_patterns.md](testing_patterns.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 109 | Built-in Framework | `#[test]`, assert macros, cargo test flags |
| 110 | Test Organization | Unit (same file), integration (`tests/`), doc tests |
| 111 | Helpers & Fixtures | Setup functions, Drop cleanup, tempfile crate |
| 112 | Error Path Testing | `unwrap_err()`, `should_panic(expected)`, `matches!` |
| 113 | Ignore & Conditional | `#[ignore]`, runtime skipping, `#[cfg(target_os)]` |
| 114 | Doc Tests | `///` examples, `#` hidden lines, `compile_fail` |
| 115 | proptest | Strategies, `prop_compose!`, shrinking, roundtrip properties |
| 116 | insta Snapshots | `assert_snapshot!`, `cargo insta review`, redactions |
| 117 | Mocking (mockall) | `#[automock]`, expectations, hand-written fakes |
| 118 | Async Testing | `#[tokio::test]`, `start_paused = true`, stream testing |
| 119 | Integration/E2E | assert_cmd, wiremock, testcontainers |
| 120 | Parametrized Tests | rstest fixtures + `#[case]`, test-case crate |
| 121 | Benchmarking | criterion (thorough), divan (quick), `black_box()` |
| 122 | Fuzzing | cargo-fuzz, arbitrary, commit corpus |
| 123 | Dev-Dependencies | tempfile, pretty_assertions, insta, proptest |
| 124 | Testing Checklist | What to test, which tool, where to put it |

## Macros (Skills 125-140)
**Source**: [macros_patterns.md](macros_patterns.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 125 | macro_rules! Fundamentals | Basic syntax, fragment specifiers, multiple arms |
| 126 | Repetition Patterns | `*`/`+`/`?` repetitions, trailing comma trick, nested repetitions |
| 127 | Advanced macro_rules! | Recursive macros, TT munchers, push-down accumulation, `@` rules |
| 128 | Hygiene & Debugging | Macro hygiene, `$crate`, `cargo expand`, `trace_macros!` |
| 129 | Proc Macros Overview | Three types, crate setup, TokenStream in/out, syn/quote ecosystem |
| 130 | Derive Macros | `#[derive(MyTrait)]`, DeriveInput parsing, handling generics |
| 131 | Attribute Macros | `#[my_attr]` on items, parsing args, wrapping functions |
| 132 | Function-like Proc Macros | `my_macro!(...)` for DSLs, compile-time validation |
| 133 | syn Parsing | DeriveInput, custom Parse, visit/fold, error spans |
| 134 | quote Generation | `#var` interpolation, `#(#iter)*` repetition, format_ident! |
| 135 | Error Handling | `syn::Error`, collecting errors, trybuild, proc-macro-error |
| 136 | Ecosystem Patterns | cfg_if!, bitflags!, LazyLock, json!, pin_project! |
| 137 | Best Practices | When to avoid macros, extract logic, don't hide control flow |
| 138 | Testing Macros | Unit tests, trybuild compile-fail, macrotest snapshots |
| 139 | Complete Derive Walkthrough | Workspace setup, trait + derive, structs + enums + generics |
| 140 | Decision Matrix | When to use each macro type vs generics/traits/const fn |

## Serde Patterns (Skills 141-150)
**Source**: [serde_patterns.md](serde_patterns.md)

| # | Skill | One-Liner |
|---|-------|-----------|
| 141 | Serde Fundamentals | Derive `Serialize`/`Deserialize`, format-agnostic, data model |
| 142 | Field Attributes | `rename_all`, `default`, `skip_serializing_if`, `flatten`, `alias` |
| 143 | Enum Representations | External, internal (`tag`), adjacent (`tag`+`content`), untagged |
| 144 | Custom Serialization | `serialize_with`, `with` modules, `serde_with` crate |
| 145 | Custom Implementations | Manual `Serialize`/`Deserialize`, Visitor, `from`/`try_from`/`into` |
| 146 | Zero-Copy Deserialization | `&'a str`, `Cow`, `#[serde(borrow)]`, from_str vs from_reader |
| 147 | Complex Types | `transparent`, non-string keys, `bound`, remote derive, `Option<Option<T>>` |
| 148 | serde_json::Value | `json!` macro, indexing, `flatten` + HashMap, typed ↔ Value |
| 149 | Streaming & Large Data | `StreamDeserializer`, `to_writer`, NDJSON, from_reader vs from_str |
| 150 | Best Practices | `deny_unknown_fields`, DTO pattern, versioning, roundtrip tests |

---

## TODO: Remaining Topics

| Priority | Area | Est. Skills | Status |
|:--------:|------|:-----------:|--------|
| 1 | Macros (macro_rules!, proc macros, syn/quote) | 16 | Done |
| 2 | Serde patterns (custom ser/de, tagged enums, zero-copy) | 8-10 | Done |
| 3 | Networking & HTTP (tokio I/O, axum, middleware) | 12-15 | Pending |
| 4 | CLI patterns (clap, config, tracing/logging) | 8-10 | Pending |
| 5 | Type system advanced (GATs, HRTBs, const generics) | 10-12 | Pending |
| 6 | Performance (profiling, SIMD, zero-copy, alloc-free) | 10-12 | Pending |
| 7 | Workspace & build (cargo workspaces, features, build scripts) | 6-8 | Pending |
| 8 | FFI & interop (C, Python/PyO3, WASM) | 10-12 | Pending |
| 9 | Database patterns (sqlx, diesel, migrations) | 8-10 | Pending |
| 10 | Observability (tracing, structured logging, metrics) | 6-8 | Pending |
| 11 | Data structures (arenas, ECS, custom collections) | 8-10 | Pending |
| 12 | Design patterns (GoF in Rust, functional patterns) | 10-12 | Pending |

---

## Change Log

| Date | Change |
|------|--------|
| 2026-02-18 | Added Serde Patterns section — Skills 141-150 |
| 2026-02-17 | Added Macros section — Skills 125-140 |
| 2026-02-16 | Initial version — Skills 1-124 index + TODO roadmap |
