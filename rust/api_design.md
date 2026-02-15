# API Design Patterns from Rust's Standard Library & Ecosystem

Skills and patterns for designing idiomatic, ergonomic, and forwards-compatible Rust APIs. Continues from Skills 1-50.

---

## Skill 51: The Builder Pattern

**Pattern:** Separate complex construction from configuration. Use when a type has many optional parameters.

**Borrowing builder** (`&mut self → &mut Self`) — reusable:
```rust
let output = Command::new("grep")
    .arg("-r")
    .arg("pattern")
    .arg("src/")
    .current_dir("/home/user")
    .output()?;

// Builder can be reused:
let mut cmd = Command::new("echo");
cmd.arg("hello");
let _ = cmd.output(); // first use
let _ = cmd.output(); // still valid
```

**Consuming builder** (`self → Self`) — used once, can move fields without cloning:
```rust
let handler = thread::Builder::new()
    .name("worker-1".into())
    .stack_size(4 * 1024 * 1024)
    .spawn(|| { /* ... */ })?;  // consumes the builder
```

**When to use builders vs alternatives:**
- **Builder** → many optional fields, validation needed, construction has side effects
- **`Default` + struct update** → few public fields, no invariants
- **`new()` constructor** → few required parameters, no options

```rust
// Default + struct update is simpler when all fields are public
let config = Config {
    retries: 3,
    timeout_ms: 5000,
    ..Config::default()
};
```

**Skill to practice:** Use borrowing builders when the builder may be reused. Use consuming builders when construction is a one-shot operation. Provide `Default` for simple config structs.

---

## Skill 52: The Typestate Pattern

**Pattern:** Encode state machines in the type system so invalid transitions are compile-time errors. Zero runtime cost via zero-sized marker types.

```rust
use std::marker::PhantomData;

// Zero-sized state markers
pub struct NoUrl;
pub struct HasUrl;

pub struct RequestBuilder<UrlState> {
    url: Option<String>,
    headers: Vec<(String, String)>,
    _state: PhantomData<UrlState>,
}

impl RequestBuilder<NoUrl> {
    pub fn new() -> Self {
        RequestBuilder { url: None, headers: vec![], _state: PhantomData }
    }

    // Setting URL transitions NoUrl → HasUrl
    pub fn url(self, url: &str) -> RequestBuilder<HasUrl> {
        RequestBuilder {
            url: Some(url.to_owned()),
            headers: self.headers,
            _state: PhantomData,
        }
    }
}

// Headers work in any state
impl<S> RequestBuilder<S> {
    pub fn header(mut self, key: &str, val: &str) -> Self {
        self.headers.push((key.to_owned(), val.to_owned()));
        self
    }
}

// send() is ONLY available when URL is set — compile error otherwise
impl RequestBuilder<HasUrl> {
    pub fn send(self) -> Result<Response, Error> { /* ... */ }
}
```

```rust
// Compiles:
RequestBuilder::new().url("https://example.com").send()?;

// Does NOT compile: "method `send` not found for RequestBuilder<NoUrl>"
// RequestBuilder::new().send();
```

**When to use:** Protocol-level correctness, connection lifecycles, multi-phase initialization. For simple builders, a runtime check is clearer.

**Skill to practice:** Reserve typestate for cases where misuse is a security or correctness hazard. For everything else, a `Result` from `build()` is simpler.

---

## Skill 53: Extension Traits

**Pattern:** Add methods to foreign types without violating the orphan rule. Convention: name them `*Ext`.

```rust
use std::io::Read;

pub trait ReadExt: Read {
    fn read_string(&mut self) -> std::io::Result<String> {
        let mut buf = String::new();
        self.read_to_string(&mut buf)?;
        Ok(buf)
    }
}

// Blanket impl: every Read type gets ReadExt for free
impl<T: Read> ReadExt for T {}
```

**Ecosystem examples:**
- `futures::StreamExt` adds `map`, `filter`, `collect` to `Stream`
- `itertools::Itertools` adds `chunks`, `tuple_windows` to `Iterator`
- `tokio::io::AsyncReadExt` adds `read_exact`, `read_to_end` to `AsyncRead`

**Extension trait vs newtype wrapper:**
| Approach | Use when |
|----------|----------|
| Extension trait | Adding methods, no new data needed |
| Newtype | Need to impl a foreign trait on a foreign type, or add state |

**Skill to practice:** Use extension traits to add convenience methods to traits you don't own. Always use blanket implementations so the methods are available on all implementors.

---

## Skill 54: Sealed Traits

**Pattern:** Prevent external crates from implementing your trait, preserving your freedom to evolve it.

```rust
mod private {
    pub trait Sealed {}
}

/// External crates can USE this trait but cannot IMPLEMENT it.
pub trait MyTrait: private::Sealed {
    fn method(&self) -> u32;
}

pub struct MyType;
impl private::Sealed for MyType {}
impl MyTrait for MyType {
    fn method(&self) -> u32 { 42 }
}
```

External crates can call `my_type.method()` but cannot write `impl MyTrait for TheirType` because `private::Sealed` is unreachable.

**Real-world usage:** `std::slice::SliceIndex` is sealed — you can use existing index types but can't define new ones. This lets the stdlib add new index types in minor releases.

**When to seal:**
- Adding new required methods should not be a breaking change
- Arbitrary implementations would be unsound
- You want to return `impl MyTrait` from a known set of types

**Skill to practice:** Seal traits that are part of your internal abstraction layer. Leave traits unsealed when user implementations are the whole point (like `Iterator`).

---

## Skill 55: The `Cow` Pattern

**Pattern:** `Cow<'a, B>` (Clone-on-Write) avoids allocations when data might not need modification.

```rust
use std::borrow::Cow;

fn normalize_path(path: &str) -> Cow<'_, str> {
    if path.contains("//") {
        Cow::Owned(path.replace("//", "/"))  // allocates only when needed
    } else {
        Cow::Borrowed(path)                   // zero-cost borrow
    }
}

let clean = normalize_path("/usr/local/bin");   // Cow::Borrowed — no alloc
let fixed = normalize_path("/usr//local//bin");  // Cow::Owned — one alloc
// Both deref to &str transparently
```

**In data structures:**
```rust
pub struct LogEntry<'a> {
    pub level: &'static str,
    pub message: Cow<'a, str>,  // borrowed when possible, owned when needed
}
```

**As function parameters — guidelines:**
- `&str` → only need to read
- `impl Into<String>` → need to store an owned copy
- `impl AsRef<str>` → maximum flexibility, no ownership
- `Cow<str>` → caller benefits from expressing "I might already own this"

**Use `Cow` primarily as a return type** when a function sometimes allocates and sometimes doesn't.

**Skill to practice:** Return `Cow` from functions that conditionally modify data. For parameters, prefer `&str` or `impl AsRef<str>`.

---

## Skill 56: Fallible Constructors

**Pattern:** Separate infallible and fallible construction with clear naming conventions.

| Pattern | Signature | When |
|---------|-----------|------|
| `new` | `fn new() -> Self` | Cannot fail |
| `new` returning Result | `fn new() -> Result<Self, E>` | Can fail, primary constructor |
| `TryFrom<T>` | `fn try_from(T) -> Result<Self, E>` | Conversion that validates |
| `From<T>` | `fn from(T) -> Self` | Infallible conversion |

**Maintaining type invariants:**
```rust
pub struct Email(String);

impl Email {
    pub fn new(addr: impl Into<String>) -> Result<Self, InvalidEmail> {
        let addr = addr.into();
        if addr.contains('@') && addr.len() > 3 {
            Ok(Email(addr))
        } else {
            Err(InvalidEmail(addr))
        }
    }

    pub fn as_str(&self) -> &str { &self.0 }
}

// TryFrom gives .try_into() for free
impl TryFrom<String> for Email {
    type Error = InvalidEmail;
    fn try_from(value: String) -> Result<Self, Self::Error> {
        Email::new(value)
    }
}
```

**Stdlib examples:**
- `NonZeroU32::new(0)` → `None` (invariant: nonzero)
- `String::from_utf8(bytes)` → `Result` (invariant: valid UTF-8)
- `u8::try_from(256u16)` → `Err` (invariant: fits in 8 bits)

**Skill to practice:** If your type has an invariant, validate it in the constructor and return `Result`. Never expose public fields that could break the invariant.

---

## Skill 57: Method Naming Conventions

**Pattern:** Follow the stdlib's strict naming conventions so users can predict your API.

### The `as_` / `to_` / `into_` triptych

| Prefix | Cost | Receiver | Example |
|--------|------|----------|---------|
| `as_` | Free (pointer cast) | `&self` | `str::as_bytes() → &[u8]` |
| `to_` | Expensive (allocates) | `&self` | `str::to_uppercase() → String` |
| `into_` | Cheap (moves data) | `self` | `String::into_bytes() → Vec<u8>` |

### Other conventions

```rust
// _mut suffix: mutable variant
v.first()       // → Option<&T>
v.first_mut()   // → Option<&mut T>

// is_ / has_: boolean predicates
v.is_empty()    // → bool
result.is_ok()  // → bool

// iter / iter_mut / into_iter: the iterator triptych
for x in v.iter() {}       // yields &T
for x in v.iter_mut() {}   // yields &mut T
for x in v.into_iter() {}  // yields T, consumes v

// len / is_empty: always pair these (clippy enforces it)
pub fn len(&self) -> usize { ... }
pub fn is_empty(&self) -> bool { self.len() == 0 }
```

**Skill to practice:** Follow these conventions exactly. Users will try `as_*` before `to_*` before `into_*` by instinct. Meeting that expectation makes your API discoverable.

---

## Skill 58: Generic Parameter Design

**Pattern:** Choose the right level of generics for your use case.

### `impl Trait` vs named generics vs `dyn Trait`

```rust
// impl Trait: simple, one generic param, callers don't name it
fn process(reader: impl Read) { ... }

// Named generic: same type used multiple times, or turbofish needed
fn copy<R: Read, W: Write>(r: &mut R, w: &mut W) { ... }

// dyn Trait: heterogeneous collections, runtime polymorphism
fn log_all(writers: &mut [&mut dyn Write]) { ... }
```

| | Generics / `impl Trait` | `dyn Trait` |
|-|-------------------------|-------------|
| Dispatch | Static (inlined) | Dynamic (vtable) |
| Binary size | Larger (monomorphized) | Smaller |
| Performance | Faster | Slight indirection |
| Flexibility | Compile-time types | Runtime types |

### `where` clauses for readability
```rust
// Hard to parse:
fn process<T: Display + Clone + Send + 'static>(item: T) -> Result<T, Error> { ... }

// Clearer:
fn process<T>(item: T) -> Result<T, Error>
where
    T: Display + Clone + Send + 'static,
{ ... }
```

### `impl Trait` in return position — hide concrete types
```rust
fn even_numbers(limit: u32) -> impl Iterator<Item = u32> {
    (0..limit).filter(|n| n % 2 == 0)
}
// Callers see Iterator<Item = u32>, not the concrete Filter<Range<...>> type
```

### Don't over-constrain structs — put bounds on methods instead
```rust
// BAD: forces T: Clone even when not needed
pub struct Cache<T: Clone> { entries: Vec<T> }

// GOOD: struct is unconstrained
pub struct Cache<T> { entries: Vec<T> }

impl<T: Clone> Cache<T> {
    pub fn get_copy(&self, i: usize) -> Option<T> { self.entries.get(i).cloned() }
}

impl<T> Cache<T> {
    pub fn len(&self) -> usize { self.entries.len() }  // no Clone needed
    pub fn push(&mut self, item: T) { self.entries.push(item) }
}
```

**Skill to practice:** Default to `impl Trait` for simple cases. Use named generics when the type appears multiple times. Use `dyn Trait` for heterogeneous collections. Put trait bounds on methods, not structs.

---

## Skill 59: `#[non_exhaustive]` for Forward Compatibility

**Pattern:** Prevent downstream code from relying on the exact set of variants or fields.

### On enums
```rust
#[non_exhaustive]
pub enum DatabaseError {
    ConnectionFailed,
    QueryTimeout,
    AuthenticationFailed,
}

// Downstream MUST include a wildcard:
match err {
    DatabaseError::ConnectionFailed => retry(),
    DatabaseError::QueryTimeout => backoff(),
    DatabaseError::AuthenticationFailed => abort(),
    _ => log_unknown(err),  // REQUIRED — won't compile without this
}
```

### On structs — prevents external construction
```rust
#[non_exhaustive]
pub struct ConnectionConfig {
    pub host: String,
    pub port: u16,
    pub timeout_secs: u64,
}

// External crates CANNOT construct via struct literal.
// Must use the provided constructor. You can add fields in minor versions.
impl ConnectionConfig {
    pub fn new(host: impl Into<String>, port: u16) -> Self {
        ConnectionConfig { host: host.into(), port, timeout_secs: 30 }
    }
}
```

### On enum variants
```rust
pub enum Event {
    #[non_exhaustive]
    Click { x: i32, y: i32 },  // can add fields later
}

// Downstream must use `..`:
match event {
    Event::Click { x, y, .. } => { /* .. is required */ }
}
```

**Stdlib usage:** `std::io::ErrorKind` is non-exhaustive — that's why matches always need `_ =>`.

**Skill to practice:** Use `#[non_exhaustive]` on any public enum or struct where you might add variants/fields in future versions.

---

## Skill 60: Standard Derives and the Orphan Rule

### What to derive

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct UserId(u64);

#[derive(Debug, Clone, PartialEq)]
pub struct Measurement {
    pub value: f64,   // f64 is not Eq/Hash (NaN), so we can't derive them
    pub unit: String,
}
```

**Order of consideration:**
1. `Debug` — nearly always
2. `Clone` — unless type manages unique resources
3. `PartialEq` / `Eq` — if equality is meaningful
4. `Hash` — if `Eq` is implemented (needed for HashMap keys)
5. `PartialOrd` / `Ord` — if natural ordering exists
6. `Default` — if "empty" or "zero" value makes sense
7. `Serialize` / `Deserialize` — for data interchange

### The orphan rule and newtype workaround

```rust
// ERROR: both Display and Vec are foreign
// impl Display for Vec<i32> { ... }

// Workaround: newtype
pub struct DisplayVec(pub Vec<i32>);

impl std::fmt::Display for DisplayVec {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:?}", self.0)
    }
}

// Make it transparent with Deref:
impl std::ops::Deref for DisplayVec {
    type Target = Vec<i32>;
    fn deref(&self) -> &Vec<i32> { &self.0 }
}
```

**Skill to practice:** Derive all applicable common traits on public types. Use newtypes to work around the orphan rule when you need to implement foreign traits on foreign types.

---

## Skill 61: Documentation as API Contract

**Pattern:** Follow the stdlib's documentation sections: Examples, Errors, Panics.

```rust
/// Divides two numbers, returning the quotient.
///
/// # Examples
///
/// ```
/// let result = my_crate::divide(10.0, 3.0).unwrap();
/// assert!((result - 3.333).abs() < 0.01);
/// ```
///
/// # Errors
///
/// Returns [`DivisionError::DivideByZero`] if `divisor` is zero.
///
/// # Panics
///
/// This function does not panic.
pub fn divide(dividend: f64, divisor: f64) -> Result<f64, DivisionError> { ... }
```

**Hidden lines** — compiled but not shown in docs:
```rust
/// ```
/// # use std::error::Error;
/// # fn main() -> Result<(), Box<dyn Error>> {
/// let config = Config::from_file("settings.toml")?;
/// assert_eq!(config.port, 8080);
/// # Ok(())
/// # }
/// ```
```

**Linking and aliases:**
```rust
/// Converts this value to a [`String`].
/// See also [`std::fmt::Display`].
#[doc(alias = "to_text")]  // discoverable via search
pub fn to_string_custom(&self) -> String { ... }
```

**Module-level docs** use `//!`:
```rust
//! # My Crate
//!
//! Utilities for network protocol handling.
//!
//! ## Quick Start
//! ```
//! use my_crate::Client;
//! let client = Client::new("https://example.com");
//! ```
```

**Skill to practice:** Write `# Examples` on every public item. Add `# Errors` for fallible functions and `# Panics` for functions that can panic. Doc examples are compiled and tested — they're your first line of regression tests.

---

## Skill 62: The Prelude Pattern

**Pattern:** Re-export commonly used items in a `prelude` module for ergonomic imports.

```rust
// src/prelude.rs
pub use crate::client::Client;
pub use crate::config::Config;
pub use crate::error::{Error, Result};
pub use crate::traits::{Encode, Decode};
pub use crate::ext::ResponseExt;
```

```rust
// Consumer code:
use my_crate::prelude::*;

fn main() -> Result<()> {
    let client = Client::new(Config::default());
    let data = client.get("/api")?.decode()?;
    Ok(())
}
```

**What to include:**
- Primary types users always need
- Traits whose methods are called frequently (especially extension traits)
- A crate-level `Result` alias

**What to exclude:**
- Builder types and intermediate types
- Rarely-used or advanced types
- Items that could collide with common names

**Ecosystem examples:**
- `diesel::prelude` — query DSL traits
- `bevy::prelude` — core ECS types
- `tokio::prelude` (older versions) — AsyncRead, AsyncWrite, Future

**Skill to practice:** Create a prelude when your crate has 5+ commonly imported items. Keep it focused — a prelude that imports everything defeats the purpose.

---

## Quick Reference: When to Apply Each Skill

| Situation | Skill |
|-----------|-------|
| Complex type construction | #51 Builder pattern |
| Compile-time state enforcement | #52 Typestate |
| Adding methods to foreign types | #53 Extension traits |
| Preventing external impl | #54 Sealed traits |
| Conditional allocation | #55 Cow pattern |
| Constructor with invariants | #56 Fallible constructors |
| Naming methods | #57 Naming conventions |
| Choosing generics strategy | #58 Generic parameter design |
| Future-proofing enums/structs | #59 #[non_exhaustive] |
| Standard trait implementation | #60 Derives + orphan rule |
| Writing API documentation | #61 Documentation sections |
| Ergonomic crate imports | #62 Prelude pattern |

---

## Change Log

- **2026-02-15**: Initial version — derived from Rust API Guidelines, stdlib patterns, and ecosystem conventions.
