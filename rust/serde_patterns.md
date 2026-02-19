# Serde Patterns in Rust

Patterns for serialization and deserialization with serde — from derive basics through custom implementations, zero-copy deserialization, dynamic JSON, streaming, and best practices. Continues from Skills 1-140.

---

## Skill 141: Serde Fundamentals — Serialize & Deserialize

**Pattern**: Serde is format-agnostic. You derive `Serialize` and `Deserialize` on your types once, then use any format (JSON, YAML, TOML, MessagePack, etc.) without changing the type.

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Config {
    name: String,
    port: u16,
    debug: bool,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = Config {
        name: "my-app".to_string(),
        port: 8080,
        debug: true,
    };

    // Serialize to JSON string
    let json = serde_json::to_string(&config)?;
    println!("{json}");
    // {"name":"my-app","port":8080,"debug":true}

    // Serialize to pretty JSON
    let pretty = serde_json::to_string_pretty(&config)?;

    // Deserialize from JSON string
    let parsed: Config = serde_json::from_str(&json)?;
    assert_eq!(parsed.port, 8080);

    // Serialize directly to a writer (file, socket, etc.)
    let mut buf = Vec::new();
    serde_json::to_writer(&mut buf, &config)?;

    // Deserialize from a reader
    let from_reader: Config = serde_json::from_reader(buf.as_slice())?;

    Ok(())
}
```

**The serde data model**: Serde defines an intermediate data model (29 types like bool, u32, string, map, seq, etc.). `Serialize` converts your type into data model tokens. `Deserialize` converts tokens back. Format crates (serde_json, serde_yaml) handle the data model to/from bytes. This is why the same derive works with every format.

**Same type, multiple formats**:
```rust
// Cargo.toml: serde_json, serde_yaml, toml
let config = Config { name: "app".into(), port: 3000, debug: false };

let json = serde_json::to_string(&config)?;   // JSON
let yaml = serde_yaml::to_string(&config)?;    // YAML
let toml = toml::to_string(&config)?;          // TOML
```

**When to use `serde_json::Value` vs typed deserialization**:
- Use typed structs (like `Config` above) when you know the schema. You get compile-time safety and better performance.
- Use `serde_json::Value` when the schema is unknown or highly dynamic (see Skill 148).

**Common mistake**: Forgetting to enable the `derive` feature. In Cargo.toml:
```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

**Skill to practice**: Define a struct with several field types (String, numbers, Vec, Option). Serialize it to JSON, YAML, and TOML. Deserialize it back and verify roundtrip equality.

---

## Skill 142: Field Attributes — Renaming, Defaults, Skipping

**Pattern**: Serde field attributes control how individual fields are serialized and deserialized without writing custom code.

**Renaming**:
```rust
use serde::{Serialize, Deserialize};

// Rename all fields in one shot
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct ApiResponse {
    user_name: String,       // serializes as "userName"
    created_at: String,      // serializes as "createdAt"
    is_active: bool,         // serializes as "isActive"
}

// Rename a single field
#[derive(Serialize, Deserialize)]
struct Record {
    #[serde(rename = "type")]   // "type" is a Rust keyword
    kind: String,
}
```

**Common `rename_all` values**: `"camelCase"`, `"snake_case"`, `"PascalCase"`, `"SCREAMING_SNAKE_CASE"`, `"kebab-case"`.

**Defaults** — provide values for missing fields:
```rust
#[derive(Serialize, Deserialize)]
struct ServerConfig {
    host: String,

    #[serde(default)]              // uses u16::default() → 0
    port: u16,

    #[serde(default = "default_timeout")]
    timeout_secs: u64,

    #[serde(default)]              // uses Vec::default() → []
    tags: Vec<String>,
}

fn default_timeout() -> u64 { 30 }

// This JSON is valid even with missing fields:
// {"host": "localhost"}
// port → 0, timeout_secs → 30, tags → []
```

**Skipping fields**:
```rust
#[derive(Serialize, Deserialize)]
struct Session {
    user_id: u64,

    #[serde(skip)]                             // skip both ser & de
    internal_cache: Vec<u8>,

    #[serde(skip_serializing)]                 // present in input, absent in output
    password_hash: String,

    #[serde(skip_serializing_if = "Option::is_none")]  // omit null fields
    nickname: Option<String>,
}
```

**Alias for migration** — accept old and new field names:
```rust
#[derive(Deserialize)]
struct User {
    #[serde(alias = "username")]   // accepts both "name" and "username"
    name: String,
}
```

**Flatten** — embed one struct inside another:
```rust
#[derive(Serialize, Deserialize)]
struct Pagination {
    page: u32,
    per_page: u32,
}

#[derive(Serialize, Deserialize)]
struct UserQuery {
    search: String,

    #[serde(flatten)]
    pagination: Pagination,   // fields merged: {"search":"...", "page":1, "per_page":20}
}
```

**Skill to practice**: Design an API response struct that uses `rename_all = "camelCase"`, `skip_serializing_if` for optional fields, and `flatten` for a nested metadata struct.

---

## Skill 143: Enum Representations — Tagged, Untagged, Adjacent

**Pattern**: Serde offers four enum representations. Choosing the right one is critical for API design.

**Default: externally tagged** — `{"variant": { ...fields... }}`:
```rust
#[derive(Serialize, Deserialize)]
enum Message {
    Text { body: String },
    Image { url: String, width: u32 },
    Ping,
}

// Text { body: "hi" } → {"Text": {"body": "hi"}}
// Ping                 → "Ping"
```

**Internally tagged** — `{"type": "variant", ...fields...}`:
```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Event {
    UserCreated { id: u64, name: String },
    UserDeleted { id: u64 },
}

// UserCreated → {"type": "UserCreated", "id": 1, "name": "Alice"}
```

Best for: APIs where consumers switch on a `type` field. Cannot contain tuple variants or non-string/map content.

**Adjacently tagged** — `{"type": "variant", "data": { ...fields... }}`:
```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type", content = "data")]
enum Shape {
    Circle { radius: f64 },
    Rect { width: f64, height: f64 },
}

// Circle → {"type": "Circle", "data": {"radius": 5.0}}
```

Best for: when you want a type tag but variants hold diverse types (tuples, primitives, etc.).

**Untagged** — no tag, tries each variant in order:
```rust
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
enum Value {
    Integer(i64),
    Float(f64),
    Text(String),
}

// 42    → Integer(42)
// 3.14  → Float(3.14)
// "hi"  → Text("hi")
```

Best for: flexible inputs (e.g., a field that accepts a string or a number). Caution: error messages are poor ("data did not match any variant"), and order matters since serde tries variants top to bottom.

**Handling unknown variants with `#[serde(other)]`**:
```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "status")]
enum JobStatus {
    Pending,
    Running,
    Complete,
    #[serde(other)]
    Unknown,          // catches any unrecognized value
}
// {"status": "SomeFutureStatus"} → Unknown
```

`#[serde(other)]` only works with internally tagged or adjacently tagged enums, and the catch-all variant must be a unit variant.

**Decision guide**:
| Representation | Best for | Limitation |
|----------------|----------|------------|
| External (default) | Rust-to-Rust, simple | Verbose JSON |
| Internal (`tag`) | REST APIs, readable JSON | No tuple variants |
| Adjacent (`tag` + `content`) | Mixed variant types | Slightly verbose |
| Untagged | Flexible input parsing | Bad error messages |

**Skill to practice**: Model an API event system with internally tagged enums. Add an `Unknown` variant with `#[serde(other)]` for forward compatibility.

---

## Skill 144: Custom Serialization — `serialize_with` and `deserialize_with`

**Pattern**: Override serialization for individual fields without implementing the full trait.

**`serialize_with` / `deserialize_with`** — per-field custom functions:
```rust
use serde::{Serialize, Deserialize, Serializer, Deserializer};
use std::time::Duration;

#[derive(Serialize, Deserialize)]
struct Task {
    name: String,

    #[serde(serialize_with = "serialize_duration_secs")]
    #[serde(deserialize_with = "deserialize_duration_secs")]
    timeout: Duration,
}

fn serialize_duration_secs<S>(duration: &Duration, serializer: S) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    serializer.serialize_u64(duration.as_secs())
}

fn deserialize_duration_secs<'de, D>(deserializer: D) -> Result<Duration, D::Error>
where
    D: Deserializer<'de>,
{
    let secs = u64::deserialize(deserializer)?;
    Ok(Duration::from_secs(secs))
}

// {"name": "build", "timeout": 30} ↔ Task { name: "build", timeout: Duration::from_secs(30) }
```

**The `with` module pattern** — combine ser + de in one module:
```rust
mod duration_secs {
    use serde::{Serializer, Deserializer, Deserialize};
    use std::time::Duration;

    pub fn serialize<S>(duration: &Duration, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_u64(duration.as_secs())
    }

    pub fn deserialize<'de, D>(deserializer: D) -> Result<Duration, D::Error>
    where
        D: Deserializer<'de>,
    {
        let secs = u64::deserialize(deserializer)?;
        Ok(Duration::from_secs(secs))
    }
}

#[derive(Serialize, Deserialize)]
struct Task {
    name: String,

    #[serde(with = "duration_secs")]   // uses duration_secs::serialize and ::deserialize
    timeout: Duration,
}
```

**DateTime as ISO 8601 string** (using the `chrono` crate):
```rust
mod iso8601 {
    use chrono::{DateTime, Utc};
    use serde::{self, Serializer, Deserializer, Deserialize};

    pub fn serialize<S>(date: &DateTime<Utc>, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let s = date.to_rfc3339();
        serializer.serialize_str(&s)
    }

    pub fn deserialize<'de, D>(deserializer: D) -> Result<DateTime<Utc>, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        DateTime::parse_from_rfc3339(&s)
            .map(|dt| dt.with_timezone(&Utc))
            .map_err(serde::de::Error::custom)
    }
}
```

**The `serde_with` crate** — avoids writing boilerplate for common transformations:
```rust
use serde_with::{serde_as, DurationSeconds, DisplayFromStr};

#[serde_as]
#[derive(Serialize, Deserialize)]
struct Task {
    name: String,

    #[serde_as(as = "DurationSeconds<u64>")]
    timeout: Duration,

    #[serde_as(as = "DisplayFromStr")]
    port: u16,   // serialized as string "8080", deserialized from "8080"
}
```

**Skill to practice**: Write a `with` module that serializes a `Vec<u8>` as a hex string and deserializes it back. Test the roundtrip.

---

## Skill 145: Custom Serializer/Deserializer Implementations

**Pattern**: When derive and attributes are not enough, implement `Serialize` or `Deserialize` manually.

**Manual Serialize**:
```rust
use serde::{Serialize, Serializer};
use serde::ser::SerializeStruct;

struct Color {
    r: u8,
    g: u8,
    b: u8,
}

impl Serialize for Color {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        // Serialize as "#rrggbb" string
        let hex = format!("#{:02x}{:02x}{:02x}", self.r, self.g, self.b);
        serializer.serialize_str(&hex)
    }
}
// Color { r: 255, g: 128, b: 0 } → "#ff8000"
```

**Manual Deserialize with a Visitor**:
```rust
use serde::{Deserialize, Deserializer};
use serde::de::{self, Visitor};
use std::fmt;

impl<'de> Deserialize<'de> for Color {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_str(ColorVisitor)
    }
}

struct ColorVisitor;

impl<'de> Visitor<'de> for ColorVisitor {
    type Value = Color;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("a color string like #ff8000")
    }

    fn visit_str<E>(self, value: &str) -> Result<Color, E>
    where
        E: de::Error,
    {
        let value = value.strip_prefix('#').ok_or_else(|| {
            de::Error::custom("color must start with #")
        })?;
        if value.len() != 6 {
            return Err(de::Error::custom("color must be 6 hex digits"));
        }
        let r = u8::from_str_radix(&value[0..2], 16).map_err(de::Error::custom)?;
        let g = u8::from_str_radix(&value[2..4], 16).map_err(de::Error::custom)?;
        let b = u8::from_str_radix(&value[4..6], 16).map_err(de::Error::custom)?;
        Ok(Color { r, g, b })
    }
}
```

**Shortcut: `#[serde(from)]` and `#[serde(into)]`** — convert via an intermediate type:
```rust
#[derive(Deserialize)]
struct HexString(String);

impl From<HexString> for Color {
    fn from(s: HexString) -> Self {
        // parse "#rrggbb" ...
        let v = &s.0[1..];
        Color {
            r: u8::from_str_radix(&v[0..2], 16).unwrap(),
            g: u8::from_str_radix(&v[2..4], 16).unwrap(),
            b: u8::from_str_radix(&v[4..6], 16).unwrap(),
        }
    }
}

#[derive(Deserialize)]
#[serde(from = "HexString")]
struct Color { r: u8, g: u8, b: u8 }
```

**`#[serde(try_from)]`** for fallible conversions (recommended over `from` when parsing can fail):
```rust
#[derive(Deserialize)]
#[serde(try_from = "String")]
struct Port(u16);

impl TryFrom<String> for Port {
    type Error = String;
    fn try_from(s: String) -> Result<Self, Self::Error> {
        let n: u16 = s.parse().map_err(|e| format!("invalid port: {e}"))?;
        if n == 0 { return Err("port must be nonzero".into()); }
        Ok(Port(n))
    }
}
```

**When to use which approach**:
| Approach | Use when |
|----------|----------|
| Derive + attributes | Covers 90% of cases |
| `serialize_with` / `with` | One field needs special treatment |
| `#[serde(from/try_from/into)]` | Validate or transform the whole struct |
| Manual `Serialize`/`Deserialize` | Full control over wire format |

**Skill to practice**: Implement a type that serializes as a string (like the Color example) with both `Serialize` and `Deserialize` using a Visitor.

---

## Skill 146: Zero-Copy Deserialization

**Pattern**: Borrow string data directly from the input buffer instead of allocating new Strings. This avoids heap allocations for string-heavy data.

```rust
use serde::Deserialize;

// Owned version — allocates a new String for every field
#[derive(Deserialize)]
struct OwnedRecord {
    name: String,
    value: String,
}

// Zero-copy version — borrows from the input &str
#[derive(Deserialize)]
struct BorrowedRecord<'a> {
    name: &'a str,
    value: &'a str,
}
```

**How it works**: When you deserialize from `&str` or `&[u8]`, serde_json can hand out `&str` references pointing into the original buffer. No allocation, no copy.

```rust
let input = r#"{"name": "Alice", "value": "hello"}"#;

// Zero-copy: name and value point into `input`
let record: BorrowedRecord = serde_json::from_str(input)?;
assert_eq!(record.name, "Alice");

// from_slice also supports zero-copy (for &[u8] input)
let bytes = input.as_bytes();
let record: BorrowedRecord = serde_json::from_slice(bytes)?;
```

**`#[serde(borrow)]` and `Cow`** — borrow when possible, own when necessary:
```rust
use std::borrow::Cow;

#[derive(Deserialize)]
struct FlexRecord<'a> {
    #[serde(borrow)]
    name: Cow<'a, str>,    // borrows if possible, allocates if escape sequences present

    #[serde(borrow)]
    tags: Vec<&'a str>,    // each element borrows from input
}
```

**When zero-copy works and when it does not**:
| Scenario | Zero-copy? |
|----------|------------|
| JSON strings with no escapes | Yes — direct borrow |
| JSON strings with `\"` or `\n` escapes | No — serde must unescape into a new buffer |
| `from_str` | Yes (if no escapes) |
| `from_reader` (file, network) | No — no persistent buffer to borrow from |
| Binary formats (bincode, etc.) | Usually yes |

**Performance tradeoffs**:
- Zero-copy is fastest when you process and discard data quickly (e.g., filtering log lines).
- If you need to store the deserialized data longer than the input buffer lives, use `Cow` or owned types.
- For small strings, the allocation overhead is negligible. Profile before optimizing.

```rust
// Pattern: parse many records from a large buffer efficiently
fn process_records(json_data: &str) -> Result<Vec<&str>, serde_json::Error> {
    #[derive(Deserialize)]
    struct Record<'a> {
        name: &'a str,
        status: &'a str,
    }

    let records: Vec<Record> = serde_json::from_str(json_data)?;
    Ok(records
        .into_iter()
        .filter(|r| r.status == "active")
        .map(|r| r.name)
        .collect())
}
```

**Skill to practice**: Benchmark owned vs zero-copy deserialization of a large JSON array of objects. Measure allocation counts with `dhat` or `jemalloc` stats.

---

## Skill 147: Serde with Complex Types

**Pattern**: Handle tricky type situations — non-string map keys, newtypes, generics, remote types, and nested Options.

**Non-string map keys**:
```rust
use std::collections::HashMap;

// JSON only supports string keys. A HashMap<u32, String> won't serialize directly to JSON.
// Solution 1: serde_json serializes non-string keys as strings automatically (since serde_json 1.0.45+)
#[derive(Serialize, Deserialize)]
struct Scores {
    #[serde(flatten)]
    map: HashMap<u32, String>,  // JSON: {"1": "Alice", "2": "Bob"}
}

// Solution 2: serialize as a list of pairs for complex key types
use serde_with::serde_as;

#[serde_as]
#[derive(Serialize, Deserialize)]
struct Grid {
    #[serde_as(as = "Vec<(_, _)>")]
    cells: HashMap<(i32, i32), String>,
    // JSON: [[[0, 0], "origin"], [[1, 2], "point"]]
}
```

**`#[serde(transparent)]`** — newtype serializes as its inner type:
```rust
#[derive(Serialize, Deserialize)]
#[serde(transparent)]
struct UserId(u64);

// Serializes as just 42, not {"0": 42}
// let id = UserId(42); → 42

#[derive(Serialize, Deserialize)]
#[serde(transparent)]
struct Email(String);

// "alice@example.com", not {"0": "alice@example.com"}
```

**Custom trait bounds with `#[serde(bound)]`**:
```rust
use serde::{Serialize, Deserialize};

// Serde's generated bounds might be too strict.
// Override them when your type has special requirements.
#[derive(Serialize, Deserialize)]
#[serde(bound(
    serialize = "T: Serialize + Clone",
    deserialize = "T: Deserialize<'de> + Default",
))]
struct Wrapper<T> {
    value: T,
    #[serde(skip, default)]
    _cache: Option<T>,
}
```

**Remote derive** — derive serde for types you don't own:
```rust
// Suppose `external_crate::Point` exists without Serialize.
// Define a local "mirror" type:
mod point_serde {
    use serde::{Serialize, Deserialize};

    #[derive(Serialize, Deserialize)]
    #[serde(remote = "external_crate::Point")]
    pub struct PointDef {
        pub x: f64,
        pub y: f64,
    }
}

#[derive(Serialize, Deserialize)]
struct MyStruct {
    #[serde(with = "point_serde::PointDef")]
    location: external_crate::Point,
}
```

**Distinguishing null from absent** with `Option<Option<T>>`:
```rust
#[derive(Serialize, Deserialize)]
struct Update {
    // None           → field absent from JSON (don't update)
    // Some(None)     → field is null (set to null)
    // Some(Some(v))  → field has value (update to v)
    #[serde(
        default,                                    // absent → None
        skip_serializing_if = "Option::is_none",    // don't emit if None
    )]
    nickname: Option<Option<String>>,
}

// {}                       → nickname: None
// {"nickname": null}       → nickname: Some(None)
// {"nickname": "Alice"}    → nickname: Some(Some("Alice"))
```

**Skill to practice**: Create a newtype with `#[serde(transparent)]` and a PATCH-style update struct using `Option<Option<T>>`. Test that absent, null, and present values all deserialize correctly.

---

## Skill 148: serde_json::Value — Dynamic JSON

**Pattern**: Use `serde_json::Value` when you need to work with JSON whose structure is unknown at compile time.

**The Value enum**:
```rust
use serde_json::Value;

// Value has six variants:
// Value::Null
// Value::Bool(bool)
// Value::Number(Number)
// Value::String(String)
// Value::Array(Vec<Value>)
// Value::Object(Map<String, Value>)
```

**The `json!` macro** — build values inline:
```rust
use serde_json::json;

let user = json!({
    "name": "Alice",
    "age": 30,
    "tags": ["admin", "active"],
    "address": null
});
```

**Indexing**:
```rust
let name = &user["name"];             // &Value::String("Alice")
let first_tag = &user["tags"][0];     // &Value::String("admin")
let missing = &user["nonexistent"];   // &Value::Null (no panic!)

// Use .as_str(), .as_u64(), etc. to extract:
if let Some(name) = user["name"].as_str() {
    println!("Name: {name}");
}
```

**Mixing typed and dynamic** — partially deserialize with `flatten`:
```rust
use std::collections::HashMap;

#[derive(Serialize, Deserialize)]
struct KnownFields {
    id: u64,
    name: String,

    #[serde(flatten)]
    extra: HashMap<String, Value>,   // catches everything else
}

let json = r#"{"id": 1, "name": "Alice", "role": "admin", "score": 95}"#;
let parsed: KnownFields = serde_json::from_str(json)?;
assert_eq!(parsed.extra["role"], json!("admin"));
assert_eq!(parsed.extra["score"], json!(95));
```

**Converting between Value and typed**:
```rust
#[derive(Serialize, Deserialize)]
struct User { name: String, age: u32 }

// Typed → Value
let user = User { name: "Alice".into(), age: 30 };
let value: Value = serde_json::to_value(&user)?;

// Value → Typed
let user2: User = serde_json::from_value(value)?;
```

**Modifying Value dynamically**:
```rust
let mut data = json!({"name": "Alice", "scores": [90, 85]});

// Modify a field
data["name"] = json!("Bob");

// Add a field
data["active"] = json!(true);

// Push to an array
data["scores"].as_array_mut().unwrap().push(json!(95));
// {"name":"Bob","scores":[90,85,95],"active":true}
```

**When to use Value**:
| Use `Value` | Use typed structs |
|-------------|-------------------|
| Schema unknown at compile time | Schema is known and stable |
| Proxying/forwarding JSON | Domain logic that operates on fields |
| Quick scripting / exploration | Production code with validation |
| Catch-all for extra fields | Performance-critical paths |

**Skill to practice**: Write a function that accepts arbitrary JSON, adds a `"processed_at"` timestamp field, and forwards it. Use `Value` for the outer structure and typed deserialization for a known inner field.

---

## Skill 149: Streaming & Large Data

**Pattern**: Process large JSON data without loading everything into memory at once.

**`serde_json::StreamDeserializer`** — parse newline-delimited or concatenated JSON:
```rust
use serde::Deserialize;
use serde_json::Deserializer;
use std::io::BufReader;
use std::fs::File;

#[derive(Deserialize)]
struct LogEntry {
    timestamp: String,
    level: String,
    message: String,
}

fn process_log_file(path: &str) -> Result<usize, Box<dyn std::error::Error>> {
    let file = File::open(path)?;
    let reader = BufReader::new(file);

    let stream = Deserializer::from_reader(reader).into_iter::<LogEntry>();
    let mut error_count = 0;

    for entry in stream {
        let entry = entry?;
        if entry.level == "ERROR" {
            error_count += 1;
            eprintln!("[{}] {}", entry.timestamp, entry.message);
        }
    }
    Ok(error_count)
}
```

This works with newline-delimited JSON (NDJSON), where each line is a complete JSON object:
```json
{"timestamp":"2026-01-01T00:00:00Z","level":"INFO","message":"started"}
{"timestamp":"2026-01-01T00:00:01Z","level":"ERROR","message":"disk full"}
```

**`to_writer` for streaming output**:
```rust
use std::io::BufWriter;
use std::fs::File;

fn write_records(path: &str, records: &[Record]) -> std::io::Result<()> {
    let file = File::create(path)?;
    let mut writer = BufWriter::new(file);

    for record in records {
        serde_json::to_writer(&mut writer, record)?;
        // Write newline separator for NDJSON
        std::io::Write::write_all(&mut writer, b"\n")?;
    }
    Ok(())
}
```

**`from_reader` vs `from_str`** — buffering tradeoffs:
```rust
// from_reader: reads incrementally, lower peak memory
// BUT: slightly slower due to many small reads
let reader = BufReader::new(File::open("data.json")?);
let data: Vec<Record> = serde_json::from_reader(reader)?;

// from_str: requires entire file in memory
// BUT: faster due to single contiguous buffer (and enables zero-copy)
let contents = std::fs::read_to_string("data.json")?;
let data: Vec<Record> = serde_json::from_str(&contents)?;
```

| Method | Peak memory | Speed | Zero-copy? |
|--------|-------------|-------|------------|
| `from_reader` | Low (buffered) | Slower | No |
| `from_str` | File size + parsed structs | Faster | Yes |
| `StreamDeserializer` | One record at a time | Depends | No |

**Processing a large JSON array without loading all elements**:
```rust
// If the file is a JSON array [item1, item2, ...], you can't directly stream.
// Options:
// 1. Change format to NDJSON (one object per line)
// 2. Use a SAX-style parser like `json-event-parser` or `simd-json`
// 3. Read the whole file with from_str and iterate the Vec
```

**Skill to practice**: Write a program that reads an NDJSON log file with `StreamDeserializer`, filters entries by level, and writes matching entries to a new NDJSON file using `to_writer`. Verify it works with a 100MB+ file.

---

## Skill 150: Serde Best Practices & Anti-Patterns

**Pattern**: Apply these guidelines to keep serialization code maintainable, safe, and forward-compatible.

**Use `deny_unknown_fields` for strict parsing** (config files, CLI input):
```rust
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
struct AppConfig {
    host: String,
    port: u16,
}

// {"host": "localhost", "port": 8080, "typo_field": true}
// → Error: unknown field `typo_field`
```

This catches typos in config files. Do NOT use it for API responses from external services (they may add fields).

**Separate API types from domain types** (DTO pattern):
```rust
// API layer — matches the wire format
#[derive(Deserialize)]
#[serde(rename_all = "camelCase")]
struct CreateUserRequest {
    user_name: String,
    email: String,
}

// Domain layer — your internal model
struct User {
    id: UserId,
    name: String,
    email: Email,     // validated newtype
    created_at: DateTime<Utc>,
}

// Convert at the boundary
impl TryFrom<CreateUserRequest> for User {
    type Error = ValidationError;
    fn try_from(req: CreateUserRequest) -> Result<Self, Self::Error> {
        Ok(User {
            id: UserId::generate(),
            name: req.user_name,
            email: Email::parse(&req.email)?,
            created_at: Utc::now(),
        })
    }
}
```

**Use `#[non_exhaustive]` with serde for forward-compatible APIs**:
```rust
#[derive(Serialize, Deserialize)]
#[non_exhaustive]
pub struct ApiResponse {
    pub data: Vec<Item>,
    pub total: usize,
    // Future: pub cursor: Option<String>,
    // Adding fields won't break downstream Deserialize
}
```

**Version your serialization format**:
```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "version")]
enum Config {
    #[serde(rename = "1")]
    V1(ConfigV1),
    #[serde(rename = "2")]
    V2(ConfigV2),
}

// Migrate at load time
fn load_config(json: &str) -> Result<ConfigV2, Error> {
    match serde_json::from_str::<Config>(json)? {
        Config::V1(v1) => Ok(v1.migrate()),
        Config::V2(v2) => Ok(v2),
    }
}
```

**Test serialization roundtrips**:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn config_roundtrip() {
        let original = Config { host: "localhost".into(), port: 8080 };
        let json = serde_json::to_string(&original).unwrap();
        let parsed: Config = serde_json::from_str(&json).unwrap();
        assert_eq!(original, parsed);
    }

    // With proptest for exhaustive coverage:
    use proptest::prelude::*;
    proptest! {
        #[test]
        fn roundtrip_never_loses_data(port in 1..=65535u16) {
            let config = Config { host: "h".into(), port };
            let json = serde_json::to_string(&config).unwrap();
            let parsed: Config = serde_json::from_str(&json).unwrap();
            prop_assert_eq!(config.port, parsed.port);
        }
    }
}
```

**Common mistakes and anti-patterns**:

| Mistake | Fix |
|---------|-----|
| Forgetting `#[serde(default)]` on optional config fields | Add `default` — config files shouldn't require every field |
| Not distinguishing null from absent | Use `Option<Option<T>>` or `#[serde(deserialize_with)]` |
| Wrong enum representation for external APIs | Default (external) is rarely right for REST APIs — use `#[serde(tag)]` |
| Deriving `Serialize` on internal types | Separate DTOs from domain types to avoid coupling |
| No roundtrip tests | Test `serialize → deserialize → assert_eq` for every serde type |
| Using `#[serde(untagged)]` as first choice | Untagged has terrible error messages — prefer tagged |
| Ignoring `deny_unknown_fields` for configs | Typos silently ignored without it |

**Skill to practice**: Audit a project's serde types against this checklist. Add `deny_unknown_fields` to config structs, separate at least one API type from its domain type, and add roundtrip tests.

---

## Quick Reference

| Skill | Pattern | One-Liner |
|-------|---------|-----------|
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

## Change Log

| Date | Change |
|------|--------|
| 2026-02-18 | Initial version — Skills 141-150 from serde/serde_json analysis |
