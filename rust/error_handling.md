# Error Handling Patterns from Rust's Standard Library & Ecosystem

Skills and patterns for idiomatic Rust error handling. Continues from Skills 1-40.

---

## Skill 41: The Error Trait — Display + Debug + source()

**Pattern:** Every error type implements `std::error::Error`, which requires `Display` (user-facing) and `Debug` (developer-facing), and optionally chains causes via `source()`.

```rust
pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

**Display vs Debug:**
- `Display` → concise, user-facing: `"connection failed"`
- `Debug` → structural, developer-facing: `ConnectionError { host: "db.local", port: 5432 }`

**Walking the source chain:**
```rust
fn print_error_chain(err: &dyn std::error::Error) {
    eprintln!("Error: {err}");
    let mut current = err.source();
    while let Some(cause) = current {
        eprintln!("  Caused by: {cause}");
        current = cause.source();
    }
}
```

**Downcasting** recovers concrete types from `dyn Error`:
```rust
if let Some(io_err) = err.downcast_ref::<std::io::Error>() {
    match io_err.kind() {
        ErrorKind::NotFound => { /* handle */ }
        _ => { /* generic */ }
    }
}
```

**Skill to practice:** Always implement both `Display` and `Debug`. Make `Display` actionable ("file not found: config.toml"), not generic ("an error occurred"). Chain causes with `source()` so error reporters can show the full story.

---

## Skill 42: Enum-Based Error Types

**Pattern:** Use an enum with one variant per failure mode. This is the standard pattern for library errors.

```rust
#[derive(Debug)]
pub enum DatabaseError {
    ConnectionFailed(std::io::Error),
    QuerySyntax { query: String, position: usize },
    Timeout { duration: std::time::Duration },
    NotFound,
}

impl fmt::Display for DatabaseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::ConnectionFailed(e) => write!(f, "connection failed: {e}"),
            Self::QuerySyntax { query, position } =>
                write!(f, "syntax error at position {position}: {query}"),
            Self::Timeout { duration } =>
                write!(f, "timed out after {duration:?}"),
            Self::NotFound => write!(f, "record not found"),
        }
    }
}

impl std::error::Error for DatabaseError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            Self::ConnectionFailed(e) => Some(e),
            _ => None,
        }
    }
}
```

**Enum vs struct errors:**
- **Enum** → multiple failure modes, callers match and recover differently
- **Struct** → single category with varying details, opaque to callers

```rust
// Opaque struct error — forwards-compatible with #[non_exhaustive]
#[derive(Debug)]
#[non_exhaustive]
pub struct ValidationError {
    pub field: String,
    pub message: String,
}
```

**Skill to practice:** Use enums when callers need to react differently to different failures. Use structs when the error is informational and callers just propagate it.

---

## Skill 43: Error Conversion with `From` and `?`

**Pattern:** Implementing `From<E>` for your error type enables the `?` operator to auto-convert.

```rust
// ? desugars to:
match some_operation() {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),  // ← From conversion here
}
```

**Implementing conversions:**
```rust
impl From<std::io::Error> for DatabaseError {
    fn from(err: std::io::Error) -> Self {
        DatabaseError::ConnectionFailed(err)
    }
}

impl From<serde_json::Error> for DatabaseError {
    fn from(err: serde_json::Error) -> Self {
        DatabaseError::QuerySyntax {
            query: String::new(),
            position: err.column(),
        }
    }
}

// Now ? works for both:
fn load_config() -> Result<Config, DatabaseError> {
    let data = std::fs::read_to_string("config.json")?;  // io::Error → DatabaseError
    let config: Config = serde_json::from_str(&data)?;    // serde::Error → DatabaseError
    Ok(config)
}
```

**Important:** `?` applies exactly one `From` conversion. It does not chain `A → B → C`. If you need `A → C`, implement `From<A> for C` directly.

**Skill to practice:** Implement `From<ExternalError>` for each external error type your functions encounter. This makes `?` just work.

---

## Skill 44: The `thiserror` Pattern — Derive Away Boilerplate

**Pattern:** `thiserror` generates `Display`, `Error`, and `From` implementations from attributes.

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ServiceError {
    #[error("database query failed")]
    Database(#[from] DatabaseError),       // generates From impl + source()

    #[error("request to {url} failed")]
    Http {
        url: String,
        #[source]                          // marks as source() but no From
        inner: reqwest::Error,
    },

    #[error("invalid input: {0}")]
    Validation(String),                    // no source, simple message

    #[error("rate limited, retry after {retry_after}s")]
    RateLimited { retry_after: u64 },
}
```

**What the attributes do:**
- `#[error("...")]` → generates `Display` (supports `{0}`, `{field_name}`)
- `#[from]` → generates `From<T>` impl AND marks as `source()`
- `#[source]` → marks as `source()` without generating `From`

**When to use:** In library code and anywhere you want typed, matchable errors. The output is identical to hand-written code — the macro is fully transparent.

**Skill to practice:** Use `thiserror` for any error type with more than two variants. The boilerplate savings are significant and the generated code is predictable.

---

## Skill 45: The `anyhow` Pattern — Type-Erased Errors for Applications

**Pattern:** `anyhow::Error` is a type-erased error container ideal for application code where you report errors rather than match on them.

```rust
use anyhow::{Context, Result, bail, ensure};

fn process_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config: {path}"))?;

    let config: Config = toml::from_str(&content)
        .context("failed to parse TOML")?;

    ensure!(config.port > 0, "port must be positive, got {}", config.port);

    if config.workers > 1024 {
        bail!("unreasonable worker count: {}", config.workers);
    }

    Ok(config)
}
```

**Key tools:**
- `context()` / `with_context()` — wrap errors with additional message
- `bail!("msg")` — shorthand for `return Err(anyhow!("msg"))`
- `ensure!(cond, "msg")` — like `assert!` but returns `Err` instead of panicking
- `err.downcast_ref::<T>()` — recover concrete type when needed

**The library/application boundary:**
```
Application layer   →  anyhow::Result    (flexible, context-rich reporting)
    |
    | calls via ?
    v
Library layer       →  Result<T, MyError> (typed, matchable via thiserror)
```

| Situation | Choice |
|-----------|--------|
| Library crate, public API | thiserror (typed errors) |
| Binary / application code | anyhow |
| CLI tool with rich diagnostics | miette |

**Skill to practice:** Use `anyhow` in `main()` and application-level functions. Add `.context()` at every `?` call site to build a readable error chain.

---

## Skill 46: The Error Boundary Pattern

**Pattern:** At the boundary between subsystems, convert internal errors to a public error type to avoid leaking dependencies.

```rust
// Internal module uses anyhow freely
mod internal {
    pub(crate) fn compute() -> anyhow::Result<i32> { /* ... */ Ok(42) }
}

// Public API exposes a typed, stable error
#[derive(Debug, thiserror::Error)]
#[error("computation failed")]
pub struct ComputeError {
    #[source]
    inner: anyhow::Error,
}

pub fn public_compute() -> Result<i32, ComputeError> {
    internal::compute().map_err(|e| ComputeError { inner: e })
}
```

**The newtype error pattern — hiding dependencies:**
```rust
#[derive(Debug)]
pub struct DbError {
    kind: DbErrorKind,
    source: Box<dyn std::error::Error + Send + Sync>,
}

#[derive(Debug, Clone, Copy)]
pub enum DbErrorKind {
    Connection,
    Query,
    NotFound,
    Constraint,
}

// Internal: convert from the real dependency
impl From<sqlx::Error> for DbError {
    fn from(err: sqlx::Error) -> Self {
        let kind = match &err {
            sqlx::Error::RowNotFound => DbErrorKind::NotFound,
            sqlx::Error::Database(e) if e.is_unique_violation() => DbErrorKind::Constraint,
            _ => DbErrorKind::Query,
        };
        DbError { kind, source: Box::new(err) }
    }
}
```

**Opaque vs transparent:**
- Opaque (`source()` returns `None`) → full freedom to change internals
- Transparent (`source()` returns `Some`) → consumers can inspect the chain

**Skill to practice:** At every crate boundary, define a public error type. Convert internal errors via `From` so `?` works cleanly. This decouples your API from your dependency versions.

---

## Skill 47: Result Combinator Fluency

**Pattern:** Choose the right style — `?`, `match`, or combinators — based on what you're doing.

### `?` — The default (propagation)
```rust
fn process() -> Result<Output, MyError> {
    let data = read_input()?;
    let parsed = parse(data)?;
    Ok(compute(parsed)?)
}
```

### `match` — When you handle specific variants
```rust
match db.get_user(id) {
    Ok(user) => Ok(render(user)),
    Err(DbError::NotFound) => Ok(render_404()),
    Err(e) => Err(e.into()),
}
```

### `map_err` — When `From` can't capture context
```rust
let file = std::fs::File::open(path)
    .map_err(|e| AppError::Config {
        path: path.to_owned(),  // extra context From can't provide
        source: e,
    })?;
```

### `or_else` — Fallback chain
```rust
fn load_config() -> Result<Config, ConfigError> {
    load_from_file("config.toml")
        .or_else(|_| load_from_file("/etc/app/config.toml"))
        .or_else(|_| Ok(Config::default()))
}
```

### Option/Result bridging
```rust
let port: u16 = std::env::var("PORT")
    .ok()                                    // Result → Option
    .and_then(|s| s.parse().ok())            // parse, discard errors
    .unwrap_or(8080);                        // fallback
```

**Skill to practice:** Default to `?`. Use `match` when recovering from specific errors. Use `map_err` when adding context. Avoid deep combinator chains on `Result` — they're harder to read than `?` with early returns.

---

## Skill 48: Panic vs Result — The Decision Framework

**Pattern:** Panics are for programmer bugs. Results are for expected failures.

| Situation | Mechanism |
|-----------|-----------|
| Broken invariant / logic error | `panic!`, `unreachable!` |
| Can't happen but compiler doesn't know | `.expect("reason this is safe")` |
| Environment/input failure | `Result<T, E>` |
| Setup that must succeed | `.expect("critical: reason")` in main |
| Test code | `.unwrap()` is fine |

**`expect` over `unwrap`:**
```rust
// BAD: panics with "called `Option::unwrap()` on a `None` value"
let config = load_config().unwrap();

// GOOD: panics with "config must exist: called `Result::unwrap()` on an `Err`..."
let config = load_config().expect("config file must exist at startup");
```

**`catch_unwind` — converting panics to Results at boundaries:**
```rust
use std::panic;

fn run_plugin(f: impl FnOnce() -> i32 + panic::UnwindSafe) -> Result<i32, String> {
    panic::catch_unwind(f).map_err(|payload| {
        payload.downcast_ref::<&str>()
            .map(|s| s.to_string())
            .unwrap_or_else(|| "unknown panic".into())
    })
}
```

**`panic=abort` vs `panic=unwind`:**
| | `panic=unwind` (default) | `panic=abort` |
|---|---|---|
| Behavior | Unwinds stack, runs destructors | Immediately aborts |
| Binary size | Larger (unwind tables) | Smaller |
| `catch_unwind` | Works | No effect |
| Use case | Applications needing cleanup | Embedded, size-critical |

**Skill to practice:** Ask: "Can the caller reasonably recover?" Yes → `Result`. No → `panic`. In library code, almost always `Result`. Reserve panics for actual bugs.

---

## Skill 49: Error Reporting for End Users

**Pattern:** Display the error chain clearly, with context flowing from general to specific.

### The main() reporting pattern
```rust
fn main() {
    if let Err(err) = run() {
        eprintln!("Error: {err}");
        let mut source = err.source();
        while let Some(cause) = source {
            eprintln!("  Caused by: {cause}");
            source = cause.source();
        }
        std::process::exit(1);
    }
}

fn run() -> anyhow::Result<()> {
    // application logic
    Ok(())
}
```

**anyhow auto-formats the chain:**
```rust
fn main() -> anyhow::Result<()> {
    run()
    // On error, anyhow prints:
    // Error: top-level message
    //
    // Caused by:
    //     0: middle context
    //     1: root cause
}
```

**`miette` for rich CLI diagnostics:**
```rust
use miette::{Diagnostic, SourceSpan};

#[derive(Debug, thiserror::Error, Diagnostic)]
#[error("invalid configuration")]
#[diagnostic(code(config::invalid), help("expected a positive integer"))]
struct ConfigError {
    #[source_code]
    src: String,
    #[label("this value")]
    span: SourceSpan,
}
// Produces:  × invalid configuration
//            ╭─[config.toml:3:1]
//          3 │ port = -1
//            ·        ^^ this value
//            ╰────
//            help: expected a positive integer
```

**Skill to practice:** Never show raw `Debug` output to users. Always format with `Display`. Add `.context()` at every `?` so the chain tells a story.

---

## Skill 50: The Complete Error Strategy

**Decision tree:**

```
Is this a library crate?
├── Yes → thiserror with enum/struct errors
│   ├── Hide dependencies with newtype pattern (#46)
│   ├── Use #[non_exhaustive] for forward compat
│   └── Implement From<ExternalError> for ? support
└── No (binary/app) → anyhow::Result
    ├── Add .context() at every ? site
    ├── Match on specific library errors where recovery is possible
    └── Report full chain in main()

Should this panic or return Result?
├── Caller can recover → Result
├── It's a bug → panic / unreachable!
└── Startup requirement → .expect("reason")
```

**The three-layer architecture:**
```
main()                    → anyhow, reports full chain
  └── application logic   → anyhow, adds context with .context()
        └── library calls → thiserror, typed errors, ? converts via From
```

**Skill to practice:** Apply this layering consistently across your projects. Libraries export typed errors. Applications wrap them with context. The error chain tells the full story from root cause to user-facing message.

---

## Quick Reference: When to Apply Each Skill

| Situation | Skill |
|-----------|-------|
| Implementing a new error type | #41 Error trait, #42 Enum errors |
| Making `?` work with external errors | #43 From conversion |
| Reducing error boilerplate | #44 thiserror |
| Application-level error handling | #45 anyhow |
| Public API error design | #46 Error boundary |
| Choosing `?` vs `match` vs combinators | #47 Result combinator fluency |
| Deciding panic vs Result | #48 Panic decision framework |
| Showing errors to users | #49 Error reporting |
| Overall error architecture | #50 Complete error strategy |

---

## Change Log

- **2026-02-15**: Initial version — derived from analysis of std::error, Result, From/?, thiserror, anyhow, and ecosystem conventions.
