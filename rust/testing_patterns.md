# Testing Patterns in Rust

Patterns for writing comprehensive, maintainable tests — from the built-in framework to property-based testing, snapshot testing, mocking, async testing, benchmarking, and fuzzing. Continues from Skills 1-108.

---

## Skill 109: The Built-in Test Framework

**Pattern**: Any function annotated with `#[test]` becomes a test. Tests pass if they don't panic.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_adds() {
        assert_eq!(2 + 2, 4);
    }

    #[test]
    fn it_detects_empty() {
        let v: Vec<i32> = vec![];
        assert!(v.is_empty(), "expected empty vec, got len {}", v.len());
    }

    // Tests can return Result — Err means failure
    #[test]
    fn result_returning_test() -> Result<(), String> {
        let val: i32 = "42".parse().map_err(|e| format!("{e}"))?;
        if val != 42 {
            return Err(format!("expected 42, got {val}"));
        }
        Ok(())
    }
}
```

**Assert macros**:
| Macro | Purpose |
|-------|---------|
| `assert!(expr)` | Panics if false |
| `assert_eq!(left, right)` | Panics if not equal, prints both (needs `PartialEq` + `Debug`) |
| `assert_ne!(left, right)` | Panics if equal |
| All accept trailing format: | `assert!(x > 0, "x was {x}")` |

**Key cargo test flags**:
```bash
cargo test                          # run all tests
cargo test --lib                    # only unit tests (in src/)
cargo test --test integration_foo   # only a specific integration test
cargo test --doc                    # only doc tests
cargo test my_func                  # filter by name substring
cargo test -- --nocapture           # show println! output
cargo test -- --test-threads=1      # run serially
cargo test -- --ignored             # run only #[ignore] tests
cargo test -- --include-ignored     # run everything
```

**Common mistake**: Forgetting the `--` separator. Flags before `--` go to Cargo; flags after go to the test binary. `cargo test --nocapture` is wrong; `cargo test -- --nocapture` is correct.

**Skill to practice**: Write tests for a function using all three assert macros. Try `cargo test -- --nocapture` to see output from passing tests.

---

## Skill 110: Test Organization — Unit vs Integration vs Doc Tests

**Pattern**: Rust has three test locations with different visibility and purpose.

**Unit tests** — inside the source file, can access private items:
```rust
// src/parser.rs
pub fn parse(input: &str) -> Result<Ast, ParseError> {
    let tokens = tokenize(input)?; // private function
    build_ast(tokens)
}

fn tokenize(input: &str) -> Result<Vec<Token>, ParseError> { /* ... */ }

#[cfg(test)]
mod tests {
    use super::*; // can access tokenize!

    #[test]
    fn tokenize_empty() {
        assert_eq!(tokenize("").unwrap(), vec![]);
    }

    #[test]
    fn parse_hello() {
        assert!(parse("hello").is_ok());
    }
}
```

**Integration tests** — in `tests/`, only public API:
```
my_crate/
  src/lib.rs
  tests/
    smoke.rs              # each .rs file is a separate test binary
    api_tests.rs
    common/
      mod.rs              # shared helpers (NOT a test file itself)
```

```rust
// tests/smoke.rs
use my_crate::parse;

#[test]
fn parse_smoke() {
    assert!(parse("hello world").is_ok());
}

// tests/api_tests.rs
mod common;  // imports tests/common/mod.rs
#[test]
fn test_with_fixture() {
    let input = common::sample_input();
    assert!(parse(&input).is_ok());
}
```

**Important**: Put shared helpers in `tests/common/mod.rs` (not `tests/common.rs`, which would become its own test binary).

**Binary crates** (only `main.rs`, no `lib.rs`) cannot have integration tests. The standard pattern: put logic in `lib.rs`, have `main.rs` be a thin wrapper.

**Skill to practice**: Organize a crate with unit tests for internal logic, integration tests for the public API, and shared test helpers in `tests/common/mod.rs`.

---

## Skill 111: Test Helpers, Fixtures & Cleanup

**Pattern**: Rust has no built-in `beforeEach`/`afterEach`. Use functions for setup and `Drop` for cleanup.

**Setup function pattern**:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn make_parser(input: &str) -> Parser {
        let mut p = Parser::new();
        p.set_strict_mode(true);
        p.load(input);
        p
    }

    #[test]
    fn parses_empty() {
        let p = make_parser("");
        assert!(p.result().is_none());
    }
}
```

**Drop for cleanup (RAII fixtures)**:
```rust
struct TestFixture {
    db: TestDb,
    temp_dir: tempfile::TempDir,
}

impl TestFixture {
    fn new() -> Self {
        Self {
            db: TestDb::new_in_memory(),
            temp_dir: tempfile::tempdir().unwrap(),
        }
    }
}
// Cleanup happens automatically via Drop, even on panic

#[test]
fn test_with_fixture() {
    let f = TestFixture::new();
    f.db.insert("key", "value").unwrap();
    assert_eq!(f.db.get("key").unwrap(), "value");
} // db and temp_dir cleaned up here
```

**The `tempfile` crate** — essential dev-dependency:
```rust
#[test]
fn test_file_write() {
    let dir = tempfile::tempdir().unwrap();
    let path = dir.path().join("test.txt");
    std::fs::write(&path, "hello").unwrap();
    assert_eq!(std::fs::read_to_string(&path).unwrap(), "hello");
} // dir cleaned up on Drop
```

**`#[cfg(test)]` visibility trick** — expose internals only for tests:
```rust
impl Connection {
    #[cfg(test)]
    pub(crate) fn state(&self) -> &State { &self.state }
}
```

**Test utilities crate** for workspaces:
```toml
# crates/server/Cargo.toml
[dev-dependencies]
test-utils = { path = "../test-utils" }
```

**Skill to practice**: Create a fixture struct with Drop cleanup for a test that creates temp files and a database connection.

---

## Skill 112: Testing Error Paths

**Pattern**: Test error cases as rigorously as success cases.

**`#[should_panic]` — test that code panics**:
```rust
#[test]
#[should_panic(expected = "division by zero")]
fn divide_by_zero_panics() {
    divide(10, 0); // panic message must CONTAIN "division by zero"
}
```

**Testing Result errors**:
```rust
#[test]
fn parse_rejects_invalid() {
    let err = parse_config("{{bad").unwrap_err();
    assert!(matches!(err, ConfigError::Syntax { line, .. } if line == 1));
}

#[test]
fn error_message_is_helpful() {
    let err = validate_email("not-an-email").unwrap_err();
    let msg = err.to_string();
    assert!(msg.contains("invalid email"), "got: {msg}");
}
```

**Testing error source chains**:
```rust
use std::error::Error;

#[test]
fn io_error_is_source() {
    let err = open_config("/nonexistent").unwrap_err();
    let source = err.source().expect("should have a source");
    assert!(source.downcast_ref::<std::io::Error>().is_some());
}
```

**When to use what**:
| Tool | Use for |
|------|---------|
| `#[should_panic]` | Functions that panic (invariant violations) |
| `unwrap_err()` + `matches!` | Functions returning `Result` — more precise |
| `.to_string()` assertions | Verifying user-facing error messages |
| `.source()` chain | Verifying error wrapping |

**Skill to practice**: Write tests for every error variant of a custom error enum, verifying both the variant and the Display message.

---

## Skill 113: #[ignore] and Conditional Tests

**Pattern**: Mark slow or environment-dependent tests with `#[ignore]`.

```rust
#[test]
#[ignore = "requires network access"]
fn test_live_api() {
    let resp = reqwest::blocking::get("https://api.example.com/health").unwrap();
    assert!(resp.status().is_success());
}
```

```bash
cargo test                       # skips ignored tests
cargo test -- --ignored          # runs ONLY ignored tests
cargo test -- --include-ignored  # runs everything
```

**Runtime skip** (when `#[cfg]` isn't enough):
```rust
#[test]
fn test_needs_env_var() {
    let Ok(key) = std::env::var("API_KEY") else {
        eprintln!("skipping: API_KEY not set");
        return; // test passes silently
    };
    // ... test using key
}
```

**Skill to practice**: Tag integration tests that hit external services with `#[ignore]` and run them separately in CI.

---

## Skill 114: Doc Tests — Executable Documentation

**Pattern**: Code blocks in `///` comments are compiled and run by `cargo test --doc`.

```rust
/// Adds two numbers.
///
/// # Examples
///
/// ```
/// use my_crate::add;
/// assert_eq!(add(2, 3), 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }
```

**Hiding boilerplate with `#`** (compiled but hidden in rendered docs):
```rust
/// ```
/// # use std::io::Write;
/// # let dir = tempfile::tempdir().unwrap();
/// # let path = dir.path().join("test.txt");
/// # std::fs::write(&path, "first\nsecond").unwrap();
/// use my_crate::first_line;
/// let line = first_line(&path).unwrap();
/// assert_eq!(line, "first");
/// ```
```

**Using `?` in doc tests** — add a hidden `Ok` return:
```rust
/// ```
/// let n: i32 = "42".parse()?;
/// assert_eq!(n, 42);
/// # Ok::<(), Box<dyn std::error::Error>>(())
/// ```
```

**Special annotations**:
| Annotation | Effect |
|-----------|--------|
| ` ```no_run ` | Compiles but doesn't execute |
| ` ```compile_fail ` | Proves the API prevents misuse at compile time |
| ` ```should_panic ` | Demonstrates a panic |
| ` ```ignore ` | Not compiled (examples rot — avoid) |

**Skill to practice**: Add doc tests to your public API. Use `compile_fail` to demonstrate that invalid usage is rejected by the compiler.

---

## Skill 115: Property-Based Testing with proptest

**Pattern**: Describe properties that must hold for all inputs. The framework generates random inputs and shrinks failures to minimal cases.

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn encode_decode_roundtrip(input in "\\PC*") {
        let encoded = encode(&input);
        let decoded = decode(&encoded).unwrap();
        prop_assert_eq!(decoded, input);
    }

    #[test]
    fn sort_preserves_length(ref v in prop::collection::vec(any::<i32>(), 0..100)) {
        let mut sorted = v.clone();
        sorted.sort();
        prop_assert_eq!(sorted.len(), v.len());
    }

    #[test]
    fn sort_is_idempotent(ref v in prop::collection::vec(any::<i32>(), 0..100)) {
        let mut v1 = v.clone();
        v1.sort();
        let mut v2 = v1.clone();
        v2.sort();
        prop_assert_eq!(v1, v2);
    }
}
```

**Strategies** generate test values:
```rust
any::<i32>()                                   // any i32
0..100i32                                      // range
prop::collection::vec(any::<u8>(), 0..50)      // vec of 0-50 u8s
"[a-z]{1,10}"                                  // regex-based string
prop::option::of(any::<i32>())                 // Option<i32>
```

**Custom strategies with `prop_compose!`**:
```rust
prop_compose! {
    fn valid_email()(
        user in "[a-z]{1,10}",
        domain in "[a-z]{1,5}",
        tld in prop::sample::select(vec!["com", "org", "net"]),
    ) -> String {
        format!("{user}@{domain}.{tld}")
    }
}

proptest! {
    #[test]
    fn email_parses(email in valid_email()) {
        prop_assert!(Email::parse(&email).is_ok());
    }
}
```

**Shrinking**: When a test fails with `vec![103, -42, 0, 88]`, proptest automatically finds the minimal failing input like `vec![1, 0]`. Commit the `proptest-regressions/` files to version control.

**Strong property ideas**: roundtrip (encode/decode), invariant preservation, idempotency, commutativity, comparison with reference implementation.

**Skill to practice**: Write a roundtrip property test for a serialize/deserialize pair.

---

## Skill 116: Snapshot Testing with insta

**Pattern**: Capture actual output as a "snapshot" and review changes. Ideal for complex structured output.

```rust
use insta::{assert_snapshot, assert_debug_snapshot, assert_json_snapshot};

#[test]
fn error_message_format() {
    let err = validate("bad input").unwrap_err();
    assert_snapshot!(err.to_string());
}

#[test]
fn user_serialization() {
    let user = User { name: "Alice".into(), age: 30 };
    assert_debug_snapshot!(user);       // Debug format
    assert_json_snapshot!(user);        // JSON (requires Serialize)
}
```

**Workflow**:
```bash
cargo test                    # snapshots written as .snap.new
cargo insta review            # interactive TUI to accept/reject
cargo insta test --check      # CI: fail if pending snapshots
```

**Inline snapshots** (stored in the source file):
```rust
#[test]
fn test_greeting() {
    assert_snapshot!(greet("World"), @"Hello, World!");
}
```

**Redactions** (mask unstable values):
```rust
#[test]
fn test_api_response() {
    let response = get_user_response();
    assert_json_snapshot!(response, {
        ".id" => "[user_id]",
        ".created_at" => "[timestamp]",
        ".**.token" => "[redacted]",
    });
}
```

Commit `.snap` files to version control. Add `.snap.new` to `.gitignore`.

**Skill to practice**: Add snapshot tests for a complex struct's Debug and JSON representations. Use redactions for timestamps and IDs.

---

## Skill 117: Mocking with mockall

**Pattern**: `mockall` generates mock structs from traits. Use for external dependencies (I/O, network, DB).

```rust
use mockall::{automock, predicate::*};

#[automock]
trait UserRepository {
    fn find_by_id(&self, id: u64) -> Option<User>;
    fn save(&self, user: &User) -> Result<(), DbError>;
}

#[test]
fn test_user_service() {
    let mut mock = MockUserRepository::new();

    mock.expect_find_by_id()
        .with(eq(42))
        .times(1)
        .returning(|_| Some(User { id: 42, name: "Alice".into() }));

    let service = UserService::new(mock);
    let user = service.get_user(42).unwrap();
    assert_eq!(user.name, "Alice");
}
```

**Key expectation methods**:
| Method | Purpose |
|--------|---------|
| `.returning(closure)` | Set return value |
| `.times(n)` | Expect exactly n calls |
| `.with(predicate)` | Match arguments |
| `.withf(\|args\| bool)` | Match with closure |
| `.in_sequence(&mut seq)` | Enforce call ordering |
| `.never()` | Expect zero calls |

**Hand-written test doubles** (often simpler):
```rust
struct FakeClock { fixed: Instant }
impl Clock for FakeClock {
    fn now(&self) -> Instant { self.fixed }
}

struct SpyNotifier { sent: Arc<Mutex<Vec<String>>> }
impl Notifier for SpyNotifier {
    fn send(&self, msg: &str) {
        self.sent.lock().unwrap().push(msg.to_string());
    }
}
```

**When to mock vs not**:
| Use mocks | Avoid mocks |
|-----------|-------------|
| External I/O (network, DB, filesystem) | Pure logic / data transformations |
| Need to verify interactions (was X called?) | Only care about output |
| Simulate error conditions | Mock setup longer than test logic |

**Anti-pattern**: Over-mocking. If your test has more mock setup than assertions, you're testing the mocking framework.

**Skill to practice**: Write a service test using mockall, then rewrite it with a hand-written fake. Compare readability.

---

## Skill 118: Async Testing

**Pattern**: Use `#[tokio::test]` or `#[async_std::test]` to test async functions.

```rust
#[tokio::test]
async fn test_fetch_user() {
    let user = fetch_user(42).await.unwrap();
    assert_eq!(user.name, "Alice");
}

// Multi-threaded runtime (for spawn, parallelism)
#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_concurrent_ops() { /* ... */ }
```

**Time manipulation** — make tests deterministic and instant:
```rust
#[tokio::test(start_paused = true)]
async fn test_timeout_expires() {
    use tokio::time::{timeout, sleep, Duration};

    let result = timeout(Duration::from_secs(30), async {
        sleep(Duration::from_secs(60)).await;
        "done"
    }).await;

    assert!(result.is_err()); // timeout fires because 60 > 30
    // Wall-clock time: ~0ms. Logical time: 30s.
}
```

**Testing streams**:
```rust
use tokio_stream::StreamExt;

#[tokio::test]
async fn test_stream_processing() {
    let input = tokio_stream::iter(vec![1, 2, 3, 4, 5]);
    let results: Vec<i32> = input
        .filter(|x| *x % 2 == 0)
        .map(|x| x * 10)
        .collect().await;
    assert_eq!(results, vec![20, 40]);
}
```

**Testing cancellation**:
```rust
#[tokio::test(start_paused = true)]
async fn test_cancellation_cleans_up() {
    let cleaned_up = Arc::new(AtomicBool::new(false));
    let flag = cleaned_up.clone();

    let handle = tokio::spawn(async move {
        let _guard = CleanupGuard { flag };
        tokio::time::sleep(Duration::from_secs(3600)).await;
    });

    tokio::time::sleep(Duration::from_millis(1)).await;
    handle.abort();
    assert!(handle.await.unwrap_err().is_cancelled());
    assert!(cleaned_up.load(Ordering::SeqCst));
}
```

**Common mistakes**:
- Using `#[tokio::test]` without `features = ["macros", "rt", "test-util"]`
- Forgetting that `tokio::test` defaults to `current_thread` — use `multi_thread` if you need `spawn` parallelism

**Skill to practice**: Write an async test with `start_paused = true` that tests a retry-with-backoff function completes instantly.

---

## Skill 119: Integration & E2E Testing

**Pattern**: Test the full system with real or mock external dependencies.

**Testing CLI tools** with `assert_cmd`:
```rust
use assert_cmd::Command;
use predicates::prelude::*;

#[test]
fn cli_help() {
    Command::cargo_bin("my-tool").unwrap()
        .arg("--help")
        .assert()
        .success()
        .stdout(predicate::str::contains("Usage:"));
}

#[test]
fn cli_invalid_input() {
    Command::cargo_bin("my-tool").unwrap()
        .arg("--input").arg("nonexistent.json")
        .assert()
        .failure()
        .stderr(predicate::str::contains("not found"))
        .code(1);
}
```

**Testing HTTP APIs** with `wiremock`:
```rust
use wiremock::{MockServer, Mock, ResponseTemplate};
use wiremock::matchers::{method, path, header};

#[tokio::test]
async fn test_api_client() {
    let server = MockServer::start().await;

    Mock::given(method("GET"))
        .and(path("/users/42"))
        .and(header("Authorization", "Bearer test-token"))
        .respond_with(ResponseTemplate::new(200)
            .set_body_json(serde_json::json!({"id": 42, "name": "Alice"})))
        .expect(1)
        .mount(&server).await;

    let client = ApiClient::new(&server.uri(), "test-token");
    let user = client.get_user(42).await.unwrap();
    assert_eq!(user.name, "Alice");
}
```

**Database testing** with `testcontainers`:
```rust
use testcontainers::{clients::Cli, images::postgres::Postgres};

#[tokio::test]
async fn test_user_repository() {
    let docker = Cli::default();
    let postgres = docker.run(Postgres::default());
    let port = postgres.get_host_port_ipv4(5432);
    let url = format!("postgres://postgres:postgres@localhost:{port}/postgres");

    let pool = sqlx::PgPool::connect(&url).await.unwrap();
    sqlx::migrate!("./migrations").run(&pool).await.unwrap();

    let repo = UserRepository::new(pool);
    repo.create(&User { id: 0, name: "Alice".into() }).await.unwrap();
    assert!(repo.find_by_name("Alice").await.unwrap().is_some());
} // container cleaned up on drop
```

**Skill to practice**: Write a wiremock test that simulates a server error (500) and verifies your client handles it correctly.

---

## Skill 120: Parametrized Tests with rstest and test-case

**Pattern**: Run the same test logic with multiple inputs without copy-pasting.

**rstest** — fixtures + parametrization:
```rust
use rstest::{rstest, fixture};

#[fixture]
fn db() -> TestDb { TestDb::new_in_memory() }

// Fixture injection by name
#[rstest]
fn test_insert(db: TestDb) {
    db.insert("key", "value").unwrap();
    assert_eq!(db.get("key").unwrap(), "value");
}

// Parametrized cases
#[rstest]
#[case("hello", 5)]
#[case("", 0)]
#[case("rust", 4)]
fn test_length(#[case] input: &str, #[case] expected: usize) {
    assert_eq!(input.len(), expected);
}

// Test matrix (cartesian product)
#[rstest]
fn test_permissions(
    #[values("read", "write", "delete")] op: &str,
    #[values(Role::Admin, Role::User, Role::Guest)] role: Role,
) {
    // Tests all 9 combinations
    let result = authorize(op, role);
    match (op, role) {
        ("delete", Role::Guest) => assert!(result.is_err()),
        _ => assert!(result.is_ok()),
    }
}
```

**test-case** — lighter, attribute-driven:
```rust
use test_case::test_case;

#[test_case(-2, -4 ; "negative doubles")]
#[test_case(0, 0   ; "zero stays zero")]
#[test_case(3, 6   ; "positive doubles")]
fn test_double(input: i32, expected: i32) {
    assert_eq!(double(input), expected);
}

#[test_case("valid@email.com" => matches Ok(_))]
#[test_case("no-at-sign"      => matches Err(_))]
fn test_parse_email(input: &str) -> Result<Email, ParseError> {
    Email::parse(input)
}
```

**Skill to practice**: Convert a set of copy-pasted tests into parametrized tests using `rstest` or `test-case`.

---

## Skill 121: Benchmarking with criterion and divan

**Pattern**: Measure performance with statistical rigor. Detect regressions across commits.

**criterion** — feature-rich, HTML reports:
```rust
// benches/my_benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId, Throughput};

fn bench_sorting(c: &mut Criterion) {
    let mut group = c.benchmark_group("sorting");
    for size in [100, 1_000, 10_000] {
        group.throughput(Throughput::Elements(size as u64));
        group.bench_with_input(BenchmarkId::from_parameter(size), &size, |b, &size| {
            b.iter(|| {
                let mut v: Vec<i32> = (0..size).rev().collect();
                v.sort();
                black_box(v)
            })
        });
    }
    group.finish();
}

criterion_group!(benches, bench_sorting);
criterion_main!(benches);
```

**divan** — simpler, attribute-driven:
```rust
// benches/my_bench.rs
fn main() { divan::main(); }

#[divan::bench(args = [100, 1000, 10_000])]
fn bench_sort(n: usize) {
    let mut v: Vec<i32> = divan::black_box((0..n as i32).rev().collect());
    v.sort();
}
```

| Tool | Best for |
|------|----------|
| `criterion` | Detailed stats, HTML reports, CI regression detection |
| `divan` | Quick benchmarks, less boilerplate |

**Common mistakes**:
- Not using `black_box()` — compiler may optimize away the code
- Running in debug mode — always use `cargo bench` (release mode)

**Skill to practice**: Benchmark two implementations of the same function with criterion. Compare them in the HTML report.

---

## Skill 122: Fuzzing

**Pattern**: Feed millions of random inputs to find crashes. Essential for parsers, deserializers, and unsafe code.

```rust
// fuzz/fuzz_targets/parse_input.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(s) = std::str::from_utf8(data) {
        let _ = my_crate::parse(s); // should never panic
    }
});
```

**Structured fuzzing** with `arbitrary` (much better than raw bytes):
```rust
use arbitrary::Arbitrary;

#[derive(Debug, Arbitrary)]
struct FuzzInput {
    name: String,
    age: u8,
    tags: Vec<String>,
}

fuzz_target!(|input: FuzzInput| {
    let _ = process(&input.name, input.age, &input.tags);
});
```

```bash
cargo install cargo-fuzz
cargo fuzz init
cargo fuzz add parse_input
cargo fuzz run parse_input           # run until interrupted
cargo fuzz tmin parse_input crash-*  # minimize a crash
```

**When to fuzz**:
| Fuzz | Don't bother |
|------|-------------|
| Parsing untrusted input | Pure business logic with typed inputs |
| Serde/deserialization | Simple CRUD |
| Any unsafe code | Function has 2 possible inputs |
| Crypto/security code | Already covered by property tests |

**Commit your corpus** (`fuzz/corpus/`) to version control — it represents discovered edge cases.

**Skill to practice**: Set up cargo-fuzz for a parser, run it for 5 minutes, and turn any crashes into regression tests.

---

## Skill 123: Essential Dev-Dependencies

**Pattern**: The Rust testing ecosystem has a standard toolkit. Here's what to reach for.

```toml
[dev-dependencies]
# Core
tempfile = "3"              # temp dirs/files with cleanup
pretty_assertions = "1"     # colorful diffs for assert_eq!

# Parametrization
rstest = "0.18"             # fixtures + parametrized tests
test-case = "3"             # lightweight parametrization

# Snapshots
insta = { version = "1", features = ["yaml", "json"] }

# Property testing
proptest = "1"

# Mocking
mockall = "0.12"

# CLI testing
assert_cmd = "2"
predicates = "3"

# HTTP mocking
wiremock = "0.6"

# Benchmarking
criterion = { version = "0.5", features = ["html_reports"] }

# Fuzzing (separate section in Cargo.toml)
# [workspace.dependencies]
# cargo-fuzz via CLI, not Cargo.toml
```

**`pretty_assertions`** — drop-in replacement for better diffs:
```rust
use pretty_assertions::assert_eq;

#[test]
fn large_struct_comparison() {
    let expected = Config { /* many fields */ };
    let actual = load_config("test.toml").unwrap();
    assert_eq!(expected, actual); // shows colored line-by-line diff
}
```

**Skill to practice**: Add `pretty_assertions` and `tempfile` to your next project's dev-dependencies.

---

## Skill 124: Testing Checklist

**What to test where**:

| What | Tool | Where |
|------|------|-------|
| Private implementation details | `#[cfg(test)] mod tests` | Same file |
| Public API contracts | Integration tests | `tests/` |
| API usage examples | Doc tests (`///`) | On each pub item |
| Error paths and edge cases | `unwrap_err()`, `matches!` | Unit tests |
| Complex invariants | proptest | Unit or integration |
| Output format stability | insta snapshots | Unit tests |
| CLI behavior | assert_cmd | Integration tests |
| HTTP client correctness | wiremock | Integration tests |
| Performance regressions | criterion / divan | `benches/` |
| Parser robustness | cargo-fuzz | `fuzz/` |
| Unsafe code soundness | Miri (`cargo +nightly miri test`) | Same tests |

**Common mistakes**:
1. Testing implementation instead of behavior (brittle tests)
2. No tests for error paths ("happy path only")
3. `unwrap()` in test setup without context — use `.expect("creating temp file")`
4. Tests that depend on execution order — each test must be independent
5. Tests that depend on wall-clock time — use `tokio::time::pause()`
6. Ignoring `cargo test --doc` — doc examples rot

**Skill to practice**: Audit a project against this checklist. Add the missing test categories.

---

## Quick Reference

| Skill | Pattern | One-Liner |
|-------|---------|-----------|
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

---

## Change Log

| Date | Change |
|------|--------|
| 2026-02-16 | Initial version — Skills 109-124 from stdlib/ecosystem analysis |
