# Advanced Type System Patterns in Rust

Patterns for leveraging Rust's type system to write safer, more expressive code — from generics and associated types to GATs, HRTBs, const generics, trait objects, closures, and lifetime patterns. Continues from Skills 1-174.

---

## Skill 175: Generics Deep Dive — Bounds, Where Clauses, Defaults

**Pattern**: Generics enable writing code that works across types while retaining full type safety. Master bounds, `where` clauses, defaults, and const generics to write flexible APIs.

**Multiple trait bounds**:
```rust
fn print_sorted<T: Clone + Ord + Debug>(items: &[T]) {
    let mut sorted = items.to_vec();
    sorted.sort();
    println!("{sorted:?}");
}
```

**`where` clauses for readability** — prefer when bounds are complex:
```rust
// Hard to read:
fn process<T: Clone + Debug + Send + Sync, U: From<T> + Display>(input: T) -> U { /* ... */ }

// Much clearer:
fn process<T, U>(input: T) -> U
where
    T: Clone + Debug + Send + Sync,
    U: From<T> + Display,
{
    let u = U::from(input.clone());
    println!("{u}");
    u
}
```

**`where` clauses can express things bounds can't**:
```rust
fn serialize_map<K, V>(map: &HashMap<K, V>) -> String
where
    K: Serialize + Eq + Hash,
    V: Serialize,
    // Bound on an associated type — can't do this inline
    HashMap<K, V>: Debug,
{
    format!("{map:?}")
}
```

**Default type parameters**:
```rust
use std::collections::HashMap;
use std::hash::{BuildHasher, RandomState};

// S defaults to RandomState — callers rarely need to specify it
struct MyMap<K, V, S = RandomState> {
    inner: HashMap<K, V, S>,
}

impl<K, V> MyMap<K, V> {
    fn new() -> Self {
        MyMap { inner: HashMap::new() }
    }
}

impl<K, V, S: BuildHasher> MyMap<K, V, S> {
    fn with_hasher(hash_builder: S) -> Self {
        MyMap { inner: HashMap::with_hasher(hash_builder) }
    }
}
```

**Const generics basics**:
```rust
fn first_n<const N: usize>(slice: &[u8]) -> [u8; N] {
    slice[..N].try_into().unwrap()
}

let arr: [u8; 4] = first_n(&[1, 2, 3, 4, 5]);
```

**Turbofish `::<>` disambiguation**:
```rust
let x = "42".parse::<i32>().unwrap();           // turbofish on method
let v = Vec::<i32>::new();                       // turbofish on type
let result = std::mem::size_of::<[u8; 32]>();    // turbofish on function
```

**When to use generics vs trait objects**:
| Use generics (`impl Trait` / `<T>`) | Use trait objects (`dyn Trait`) |
|------|------|
| Known type at compile time | Type chosen at runtime |
| Need maximum performance (monomorphization) | Heterogeneous collection |
| Few concrete types used | Many types, want smaller binary |

**Skill to practice**: Write a generic function with three trait bounds, then refactor it to use a `where` clause. Try adding a default type parameter.

---

## Skill 176: Associated Types vs Generic Parameters

**Pattern**: Associated types fix one implementation per type (one-to-one). Generic parameters allow multiple implementations (one-to-many). Choose based on how many implementations make sense.

**Associated types — one implementation per type**:
```rust
trait Iterator {
    type Item; // Each iterator has exactly one Item type

    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter { count: u32 }

impl Iterator for Counter {
    type Item = u32; // Fixed: Counter always yields u32
    fn next(&mut self) -> Option<u32> {
        self.count += 1;
        if self.count <= 5 { Some(self.count) } else { None }
    }
}
```

**Generic parameters — multiple implementations per type**:
```rust
trait From<T> {
    fn from(value: T) -> Self;
}

// String implements From multiple times:
// impl From<&str> for String
// impl From<Vec<u8>> for String
// impl From<char> for String
```

**Decision rule**: Can a type sensibly implement the trait multiple times with different types? If yes, use generics. If no, use associated types.

**Associated type defaults** (nightly, but common in ecosystem):
```rust
trait Handler {
    type Error = std::io::Error; // default
    fn handle(&self) -> Result<(), Self::Error>;
}

// Implementors can override or accept the default
impl Handler for SimpleHandler {
    // Uses default Error = std::io::Error
    fn handle(&self) -> Result<(), std::io::Error> { Ok(()) }
}
```

**Constraining associated types**:
```rust
fn sum_all<I>(iter: I) -> i64
where
    I: Iterator<Item = i64>, // constrain the associated type
{
    iter.sum()
}

// Or in a trait bound:
fn debug_iter<I: Iterator>(iter: I)
where
    I::Item: Debug, // bound on the associated type
{
    for item in iter {
        println!("{item:?}");
    }
}
```

**Both in one trait**:
```rust
trait Graph {
    type Node;           // Associated: one node type per graph
    type Edge;           // Associated: one edge type per graph

    fn neighbors(&self, node: &Self::Node) -> Vec<Self::Node>;
}

trait Convert<T> {       // Generic: graph can convert to many formats
    fn convert(&self) -> T;
}
```

**Skill to practice**: Design a trait for a cache. Should the key and value types be associated types or generic parameters? Implement it both ways and compare.

---

## Skill 177: GATs (Generic Associated Types)

**Pattern**: GATs allow associated types to have their own generic parameters (lifetimes or types). They solve the "lending" problem — returning references tied to `&self` rather than to the type itself.

**The problem**: Without GATs, you cannot write an iterator that borrows from itself:
```rust
// This doesn't work without GATs:
trait LendingIterator {
    type Item;  // Can't express "Item borrows from &self"
    fn next(&mut self) -> Option<Self::Item>;
}
```

**GATs solve this**:
```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;  // Item can borrow from self

    fn next(&mut self) -> Option<Self::Item<'_>>;
}

struct WindowsMut<'data, T> {
    data: &'data mut [T],
    pos: usize,
    size: usize,
}

impl<'data, T> LendingIterator for WindowsMut<'data, T> {
    type Item<'a> = &'a mut [T] where Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>> {
        if self.pos + self.size > self.data.len() {
            return None;
        }
        let start = self.pos;
        self.pos += 1;
        // Return a mutable slice that borrows from self
        Some(&mut self.data[start..start + self.size])
    }
}
```

**Collection trait with GATs**:
```rust
trait Collection {
    type Item;
    type Iter<'a>: Iterator<Item = &'a Self::Item> where Self: 'a;

    fn iter(&self) -> Self::Iter<'_>;
    fn push(&mut self, item: Self::Item);
}

impl<T> Collection for Vec<T> {
    type Item = T;
    type Iter<'a> = std::slice::Iter<'a, T> where T: 'a;

    fn iter(&self) -> Self::Iter<'_> {
        self.as_slice().iter()
    }

    fn push(&mut self, item: T) {
        Vec::push(self, item);
    }
}
```

**GATs with type parameters**:
```rust
trait Deserializer {
    type Output<T: DeserializeOwned>;

    fn deserialize<T: DeserializeOwned>(&self, data: &[u8]) -> Self::Output<T>;
}

struct JsonDeserializer;
impl Deserializer for JsonDeserializer {
    type Output<T: DeserializeOwned> = serde_json::Result<T>;

    fn deserialize<T: DeserializeOwned>(&self, data: &[u8]) -> Self::Output<T> {
        serde_json::from_slice(data)
    }
}
```

**The `where Self: 'a` bound**: Required on GATs with lifetime parameters to ensure the associated type doesn't outlive the implementing type. Always include it.

**Skill to practice**: Implement a `LendingIterator` that yields overlapping mutable windows of a slice, something standard `Iterator` cannot express.

---

## Skill 178: HRTBs (Higher-Ranked Trait Bounds)

**Pattern**: `for<'a>` means "for any lifetime `'a`". Use when a function must work with any borrowed input, not just one specific lifetime.

**The problem**: Closures taking references need HRTBs:
```rust
// This function needs to call `f` with references of ANY lifetime:
fn apply_to_both<F>(a: &str, b: &str, f: F) -> (String, String)
where
    F: for<'a> Fn(&'a str) -> String,  // works for ANY lifetime
{
    (f(a), f(b))
}

let result = apply_to_both("hello", "world", |s| s.to_uppercase());
```

**Why not just a regular lifetime?**
```rust
// This is WRONG — ties both calls to the same lifetime:
fn apply_to_both_bad<'a, F>(a: &'a str, b: &'a str, f: F) -> (String, String)
where
    F: Fn(&'a str) -> String,
{
    (f(a), f(b))
}
// The caller might have a and b with different lifetimes — this is too restrictive.
```

**Implicit HRTBs**: Rust often inserts `for<'a>` automatically. These are equivalent:
```rust
fn takes_closure(f: impl Fn(&str) -> &str) { /* ... */ }
// Compiler desugars to:
fn takes_closure(f: impl for<'a> Fn(&'a str) -> &'a str) { /* ... */ }
```

**Explicit HRTBs needed when inference fails**:
```rust
trait Parser {
    fn parse<'a>(&self, input: &'a str) -> &'a str;
}

// Must be explicit here:
fn run_parser<P>(parser: P, inputs: &[&str])
where
    P: for<'a> Parser, // P must work with any lifetime
{
    for input in inputs {
        println!("{}", parser.parse(input));
    }
}
```

**HRTBs with closures stored in structs**:
```rust
struct Transformer {
    // The closure must work with any input lifetime
    transform: Box<dyn for<'a> Fn(&'a str) -> &'a str>,
}

impl Transformer {
    fn new(f: impl for<'a> Fn(&'a str) -> &'a str + 'static) -> Self {
        Transformer { transform: Box::new(f) }
    }

    fn apply(&self, input: &str) -> &str {
        (self.transform)(input)
    }
}

let t = Transformer::new(|s: &str| s.trim());
assert_eq!(t.apply("  hello  "), "hello");
```

**Real-world examples**:
- `serde::Deserialize<'de>` uses lifetimes for zero-copy deserialization
- Axum extractors: `FromRequest` uses HRTBs internally
- `std::str::pattern::Pattern<'a>` — search patterns work with any string lifetime

**Skill to practice**: Write a function that takes a closure operating on references and stores results. Use `for<'a>` explicitly when the compiler cannot infer it.

---

## Skill 179: Const Generics

**Pattern**: Const generics allow types and functions to be parameterized by compile-time constant values, enabling type-safe fixed-size abstractions.

**Basic const generics**:
```rust
#[derive(Debug)]
struct Array<T, const N: usize> {
    data: [T; N],
}

impl<T: Default + Copy, const N: usize> Array<T, N> {
    fn new() -> Self {
        Array { data: [T::default(); N] }
    }

    fn len(&self) -> usize { N }
}

let arr3: Array<i32, 3> = Array::new();
let arr5: Array<f64, 5> = Array::new();
// Array<i32, 3> and Array<i32, 5> are DIFFERENT types — size mismatch is a compile error
```

**Replacing macro-generated impls**: Before const generics, std had to manually implement traits for `[T; 0]`, `[T; 1]`, ..., `[T; 32]`. Now:
```rust
impl<T: Debug, const N: usize> Debug for Array<T, N> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_list().entries(self.data.iter()).finish()
    }
}
```

**Compile-time size checking**:
```rust
struct Matrix<T, const ROWS: usize, const COLS: usize> {
    data: [[T; COLS]; ROWS],
}

impl<T, const R: usize, const C: usize, const C2: usize> Matrix<T, R, C>
where
    T: Default + Copy + std::ops::Add<Output = T> + std::ops::Mul<Output = T>,
{
    fn multiply(&self, other: &Matrix<T, C, C2>) -> Matrix<T, R, C2> {
        let mut result = Matrix {
            data: [[T::default(); C2]; R],
        };
        for i in 0..R {
            for j in 0..C2 {
                for k in 0..C {
                    result.data[i][j] = result.data[i][j] + self.data[i][k] * other.data[k][j];
                }
            }
        }
        result
    }
}

// Compile error if dimensions don't match:
let a = Matrix::<i32, 2, 3> { data: [[1; 3]; 2] };
let b = Matrix::<i32, 3, 4> { data: [[1; 4]; 3] };
let c: Matrix<i32, 2, 4> = a.multiply(&b); // OK: 2x3 * 3x4 = 2x4
// let d = a.multiply(&a); // ERROR: 2x3 * 2x3 — inner dimensions don't match
```

**Const generic defaults**:
```rust
struct Buffer<const SIZE: usize = 1024> {
    data: [u8; SIZE],
    len: usize,
}

impl<const SIZE: usize> Buffer<SIZE> {
    fn new() -> Self {
        Buffer { data: [0; SIZE], len: 0 }
    }
}

let default_buf = Buffer::<>::new();     // 1024 bytes
let small_buf = Buffer::<64>::new();     // 64 bytes
```

**Allowed const generic types** (stable): integers (`u8`-`u128`, `i8`-`i128`, `usize`, `isize`), `bool`, `char`.

**Const generic expressions** (nightly only):
```rust
// Nightly: const expressions in generic positions
// fn split<const N: usize>(arr: [u8; N]) -> ([u8; N/2], [u8; N - N/2]) { ... }
```

**Skill to practice**: Implement a fixed-size stack `Stack<T, const N: usize>` with `push`, `pop`, and compile-time capacity.

---

## Skill 180: Trait Objects & Dynamic Dispatch

**Pattern**: `dyn Trait` erases concrete types behind a vtable, enabling runtime polymorphism. Use when you need heterogeneous collections or plugin-like architectures.

**How it works** — a trait object is a fat pointer:
```rust
// &dyn Draw is two pointers:
// 1. pointer to the data
// 2. pointer to the vtable (function pointers for Draw methods)

trait Draw {
    fn draw(&self);
    fn bounds(&self) -> Rect;
}

struct Circle { radius: f64 }
struct Square { side: f64 }

impl Draw for Circle {
    fn draw(&self) { println!("Drawing circle r={}", self.radius); }
    fn bounds(&self) -> Rect { /* ... */ }
}
impl Draw for Square {
    fn draw(&self) { println!("Drawing square s={}", self.side); }
    fn bounds(&self) -> Rect { /* ... */ }
}

// Heterogeneous collection — impossible with generics alone
let shapes: Vec<Box<dyn Draw>> = vec![
    Box::new(Circle { radius: 5.0 }),
    Box::new(Square { side: 3.0 }),
];

for shape in &shapes {
    shape.draw(); // dynamic dispatch through vtable
}
```

**Owned vs borrowed trait objects**:
```rust
fn draw_all(shapes: &[&dyn Draw]) { /* borrowed, no allocation */ }
fn store_shape(shape: Box<dyn Draw>) { /* owned, heap-allocated */ }

// Arc<dyn Trait> for shared ownership across threads:
let shared: Arc<dyn Draw + Send + Sync> = Arc::new(Circle { radius: 1.0 });
```

**Object safety rules** — a trait is object-safe if:
```rust
// OBJECT SAFE:
trait Good {
    fn method(&self);                    // &self receiver
    fn method_mut(&mut self);            // &mut self receiver
    fn into_string(self: Box<Self>) -> String; // Box<Self> receiver
}

// NOT OBJECT SAFE:
trait Bad {
    fn clone(&self) -> Self;             // returns Self (sized)
    fn generic<T>(&self, t: T);          // generic method
    fn static_method() -> String;        // no self parameter
}

// Workaround: where Self: Sized excludes from vtable
trait Workaround: Clone {
    fn normal_method(&self);
    fn clone_boxed(&self) -> Box<dyn Workaround> where Self: Sized {
        // Only callable on concrete types, not through dyn
        todo!()
    }
}
```

**Performance comparison**:
| | Generics (static dispatch) | Trait objects (dynamic dispatch) |
|---|---|---|
| Dispatch | Inlined at compile time | Vtable lookup at runtime |
| Binary size | Larger (monomorphized copies) | Smaller (one copy) |
| Speed | Faster (inlining + optimizations) | Slight overhead (~1-3ns per call) |
| Flexibility | Type known at compile time | Type chosen at runtime |

**Skill to practice**: Build a plugin system using `Box<dyn Plugin>` where each plugin is loaded into a `Vec`. Compare with an enum-based approach.

---

## Skill 181: impl Trait — RPIT & APIT

**Pattern**: `impl Trait` hides concrete types behind an opaque interface. In argument position it's sugar for generics; in return position it's an opaque type the caller cannot name.

**Argument Position Impl Trait (APIT)** — sugar for generics:
```rust
// These are equivalent:
fn print_it(item: impl Display) { println!("{item}"); }
fn print_it<T: Display>(item: T) { println!("{item}"); }

// APIT is simpler when you don't need to name T elsewhere
fn log(msg: impl Display, level: impl Display) {
    println!("[{level}] {msg}");
}
```

**Return Position Impl Trait (RPIT)** — opaque return type:
```rust
fn make_counter() -> impl Iterator<Item = u32> {
    (0..).map(|n| n * 2) // Caller knows it's an Iterator, not the concrete type
}

fn make_greeting(name: &str) -> impl Display + '_ {
    format!("Hello, {name}!")
}
```

**Why RPIT matters**: Without it, you'd have to name unnameable types:
```rust
// The actual type is something like Map<RangeFrom<u32>, [closure@...]>
// You CAN'T write that. impl Trait lets you avoid naming it.
```

**RPITIT — `impl Trait` in trait return types** (stable since Rust 1.75):
```rust
trait Container {
    fn items(&self) -> impl Iterator<Item = &str>;
}

struct MyContainer {
    data: Vec<String>,
}

impl Container for MyContainer {
    fn items(&self) -> impl Iterator<Item = &str> {
        self.data.iter().map(|s| s.as_str())
    }
}
```

**RPITIT enables async in traits**:
```rust
trait Service {
    async fn call(&self, req: Request) -> Response;
    // Desugars to: fn call(&self, req: Request) -> impl Future<Output = Response>
}
```

**`impl Trait` in type aliases** (nightly — `type_alias_impl_trait`):
```rust
// Nightly only:
// type MyFuture = impl Future<Output = i32>;
// fn make_future() -> MyFuture { async { 42 } }
```

**Limitations**:
- Cannot use `impl Trait` in struct fields (use generics or `Box<dyn Trait>`)
- Cannot have different concrete types in branches:
```rust
// ERROR: both branches must return the same concrete type
fn make_iter(ascending: bool) -> impl Iterator<Item = i32> {
    if ascending {
        (0..10).into_iter()
    } else {
        // (0..10).rev() is a DIFFERENT type — compile error
        (0..10).rev() // ERROR
    }
}

// Fix: use Box<dyn Iterator>
fn make_iter(ascending: bool) -> Box<dyn Iterator<Item = i32>> {
    if ascending {
        Box::new(0..10)
    } else {
        Box::new((0..10).rev())
    }
}
```

**Skill to practice**: Refactor a function returning `Box<dyn Iterator>` to use `impl Iterator`. Identify cases where `Box<dyn>` is still necessary.

---

## Skill 182: Type-Level Programming

**Pattern**: Encode invariants and state in the type system so invalid states are unrepresentable. The compiler rejects mistakes at compile time instead of runtime.

**PhantomData for type-level markers**:
```rust
use std::marker::PhantomData;

struct Meters;
struct Seconds;

struct Quantity<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl<U> Quantity<U> {
    fn new(value: f64) -> Self {
        Quantity { value, _unit: PhantomData }
    }
}

let distance = Quantity::<Meters>::new(100.0);
let time = Quantity::<Seconds>::new(9.58);
// Can't accidentally add meters to seconds — different types!
```

**Typestate pattern — compile-time state machines**:
```rust
// States are types
struct Draft;
struct Review;
struct Published;

struct Article<State> {
    title: String,
    body: String,
    _state: PhantomData<State>,
}

impl Article<Draft> {
    fn new(title: String) -> Self {
        Article { title, body: String::new(), _state: PhantomData }
    }

    fn write(&mut self, text: &str) { self.body.push_str(text); }

    fn submit(self) -> Article<Review> {
        Article { title: self.title, body: self.body, _state: PhantomData }
    }
}

impl Article<Review> {
    fn approve(self) -> Article<Published> {
        Article { title: self.title, body: self.body, _state: PhantomData }
    }

    fn reject(self) -> Article<Draft> {
        Article { title: self.title, body: self.body, _state: PhantomData }
    }
}

impl Article<Published> {
    fn url(&self) -> String {
        format!("/articles/{}", self.title.to_lowercase().replace(' ', "-"))
    }
}

// Usage — compiler enforces valid transitions:
let article = Article::<Draft>::new("Rust Types".into());
// article.url(); // ERROR: url() only exists on Published
let article = article.submit();
let article = article.approve();
println!("{}", article.url()); // OK
```

**Type-safe builder with required fields**:
```rust
struct Yes;
struct No;

struct ConnectionBuilder<HasHost, HasPort> {
    host: Option<String>,
    port: Option<u16>,
    timeout: u64,
    _has_host: PhantomData<HasHost>,
    _has_port: PhantomData<HasPort>,
}

impl ConnectionBuilder<No, No> {
    fn new() -> Self {
        ConnectionBuilder {
            host: None, port: None, timeout: 30,
            _has_host: PhantomData, _has_port: PhantomData,
        }
    }
}

impl<P> ConnectionBuilder<No, P> {
    fn host(self, host: &str) -> ConnectionBuilder<Yes, P> {
        ConnectionBuilder {
            host: Some(host.to_string()), port: self.port,
            timeout: self.timeout,
            _has_host: PhantomData, _has_port: PhantomData,
        }
    }
}

impl<H> ConnectionBuilder<H, No> {
    fn port(self, port: u16) -> ConnectionBuilder<H, Yes> {
        ConnectionBuilder {
            host: self.host, port: Some(port),
            timeout: self.timeout,
            _has_host: PhantomData, _has_port: PhantomData,
        }
    }
}

impl<H, P> ConnectionBuilder<H, P> {
    fn timeout(mut self, secs: u64) -> Self {
        self.timeout = secs;
        self
    }
}

// build() only available when BOTH host and port are set
impl ConnectionBuilder<Yes, Yes> {
    fn build(self) -> Connection {
        Connection {
            host: self.host.unwrap(),
            port: self.port.unwrap(),
            timeout: self.timeout,
        }
    }
}

struct Connection { host: String, port: u16, timeout: u64 }

// Compile error if you forget required fields:
let conn = ConnectionBuilder::new()
    .host("localhost")
    .port(5432)
    .timeout(10)
    .build(); // OK

// ConnectionBuilder::new().build(); // ERROR: build() not available on <No, No>
// ConnectionBuilder::new().host("x").build(); // ERROR: build() not available on <Yes, No>
```

**Skill to practice**: Implement a typestate HTTP request builder where `send()` is only available after setting the URL and method.

---

## Skill 183: Newtype Pattern Deep Dive

**Pattern**: Wrap existing types in single-field structs for type safety, trait implementation, and semantic clarity — with zero runtime cost.

**Basic newtype for type safety**:
```rust
struct UserId(u64);
struct OrderId(u64);

fn get_order(user: UserId, order: OrderId) -> String {
    format!("User {} order {}", user.0, order.0)
}

let user = UserId(42);
let order = OrderId(100);
get_order(user, order);     // OK
// get_order(order, user);  // COMPILE ERROR — can't mix them up
```

**Implementing Deref — and when NOT to**:
```rust
use std::ops::Deref;

struct EmailAddress(String);

impl Deref for EmailAddress {
    type Target = str;
    fn deref(&self) -> &str { &self.0 }
}

let email = EmailAddress("user@example.com".into());
println!("{}", email.len()); // Deref gives access to str methods

// WARNING: Don't implement DerefMut — it would let callers bypass
// your validation and set invalid email addresses.
// Only use Deref when the newtype IS-A wrapper (like smart pointers).
// For newtypes with invariants, provide explicit methods instead.
```

**`#[repr(transparent)]` for FFI**:
```rust
#[repr(transparent)] // Guarantees same memory layout as inner type
struct FileDescriptor(i32);

// Safe to pass to C functions expecting i32
extern "C" {
    fn close(fd: i32) -> i32;
}

impl Drop for FileDescriptor {
    fn drop(&mut self) {
        unsafe { close(self.0); }
    }
}
```

**Orphan rule bypass** — implement foreign traits on foreign types:
```rust
// Can't do: impl Display for Vec<u8> (both foreign)
// CAN do with newtype:

struct Bytes(Vec<u8>);

impl std::fmt::Display for Bytes {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        for byte in &self.0 {
            write!(f, "{byte:02x}")?;
        }
        Ok(())
    }
}

let b = Bytes(vec![0xDE, 0xAD, 0xBE, 0xEF]);
println!("{b}"); // "deadbeef"
```

**Reducing boilerplate with `derive_more`**:
```rust
use derive_more::{Display, From, Into, Deref, DerefMut, AsRef, Constructor};

#[derive(Debug, Clone, PartialEq, Eq, Hash, Display, From, Into, Deref, Constructor)]
struct Username(String);

// derive_more generates: From<String>, Into<String>, Deref<Target=str>,
// Display, and a new() constructor — all in one line.

let name = Username::new("alice".into());
println!("{name}");          // Display
let s: String = name.into(); // Into<String>
```

**Newtype with validation**:
```rust
#[derive(Debug, Clone, PartialEq, Eq)]
struct Port(u16);

impl Port {
    pub fn new(value: u16) -> Result<Self, &'static str> {
        if value == 0 {
            Err("port cannot be zero")
        } else {
            Ok(Port(value))
        }
    }

    pub fn get(&self) -> u16 { self.0 }
}

// No pub field access — invariant (non-zero) always holds
```

**Skill to practice**: Create newtypes for `UserId`, `Email`, and `Port` with appropriate trait implementations. Use `derive_more` to reduce boilerplate.

---

## Skill 184: Closures & Fn Traits

**Pattern**: Closures in Rust are anonymous types implementing `Fn`, `FnMut`, or `FnOnce`. Understanding the trait hierarchy and capture semantics is key to using them effectively.

**The closure trait hierarchy**:
```rust
// FnOnce — can be called once (may consume captured values)
//   ↑ supertrait of
// FnMut — can be called multiple times (may mutate captures)
//   ↑ supertrait of
// Fn    — can be called multiple times (only reads captures)

// Every Fn is also FnMut. Every FnMut is also FnOnce.
```

**How closures capture**:
```rust
let name = String::from("Alice");
let greeting = String::from("Hello");

// Captures name by reference (&name) — implements Fn
let say_hi = || println!("{greeting}, {name}!");

// Captures count by mutable reference (&mut count) — implements FnMut
let mut count = 0;
let mut increment = || { count += 1; count };

// Captures name by value (moves it) — implements FnOnce
let consume = move || {
    drop(name); // name is moved into closure
};
consume();
// consume(); // ERROR: FnOnce, already called
```

**`move` keyword**:
```rust
fn spawn_greeting(name: String) -> std::thread::JoinHandle<()> {
    // Must use `move` — thread might outlive `name`
    std::thread::spawn(move || {
        println!("Hello, {name}!");
    })
}
```

**Each closure has a unique type**:
```rust
let add_one = |x: i32| x + 1;
let add_two = |x: i32| x + 2;
// typeof(add_one) != typeof(add_two) — even though both are Fn(i32) -> i32

// This means you can't put them in a Vec without trait objects:
let fns: Vec<Box<dyn Fn(i32) -> i32>> = vec![
    Box::new(add_one),
    Box::new(add_two),
];
```

**Returning closures**:
```rust
// Return with impl Fn (preferred — zero-cost, monomorphized):
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}

// Return with Box<dyn Fn> (when type varies at runtime):
fn make_op(add: bool) -> Box<dyn Fn(i32) -> i32> {
    if add {
        Box::new(|x| x + 1)
    } else {
        Box::new(|x| x * 2)
    }
}
```

**Function pointers `fn()` vs closure traits `Fn()`**:
```rust
// fn() is a concrete type — only for functions/closures that capture nothing
fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }

fn double(x: i32) -> i32 { x * 2 }
apply(double, 5);              // OK: function pointer
apply(|x| x * 2, 5);          // OK: non-capturing closure coerces to fn pointer

let factor = 3;
// apply(|x| x * factor, 5);  // ERROR: captures `factor`, not a fn pointer

// Use Fn trait instead for maximum flexibility:
fn apply_generic(f: impl Fn(i32) -> i32, x: i32) -> i32 { f(x) }
apply_generic(|x| x * factor, 5); // OK
```

**Closures as struct fields**:
```rust
struct EventHandler<F: Fn(&str)> {
    callback: F,
}

// Or with dynamic dispatch for heterogeneous handlers:
struct DynHandler {
    callback: Box<dyn Fn(&str) + Send>,
}
```

**Skill to practice**: Write a function that returns a closure capturing mutable state. Then store multiple closures in a `Vec` using trait objects.

---

## Skill 185: Advanced Lifetime Patterns

**Pattern**: Lifetimes prevent dangling references. Master the elision rules, multiple lifetime parameters, and tricky patterns to work productively with the borrow checker.

**The three lifetime elision rules** (applied by the compiler):
```rust
// Rule 1: Each reference parameter gets its own lifetime
fn foo(x: &str, y: &str) -> ...
// becomes: fn foo<'a, 'b>(x: &'a str, y: &'b str) -> ...

// Rule 2: If there's exactly one input lifetime, it's used for all outputs
fn foo(x: &str) -> &str
// becomes: fn foo<'a>(x: &'a str) -> &'a str

// Rule 3: If one parameter is &self or &mut self, its lifetime is used for outputs
impl Foo {
    fn bar(&self, x: &str) -> &str
    // becomes: fn bar<'a, 'b>(&'a self, x: &'b str) -> &'a str
}
```

**Multiple lifetime parameters and relationships**:
```rust
// 'a: 'b means 'a outlives 'b (or equivalently, 'a is at least as long as 'b)
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

// Different lifetimes when they're independent:
struct Parser<'input, 'config> {
    input: &'input str,
    config: &'config Config,
}

impl<'input, 'config> Parser<'input, 'config> {
    // Return lifetime tied to input, not config
    fn next_token(&self) -> &'input str {
        // ...
        &self.input[..1]
    }
}
```

**Struct lifetimes**:
```rust
struct Excerpt<'a> {
    text: &'a str,
}

impl<'a> Excerpt<'a> {
    fn new(text: &'a str) -> Self {
        Excerpt { text }
    }

    // The returned reference lives as long as the Excerpt
    fn first_word(&self) -> &str {
        self.text.split_whitespace().next().unwrap_or("")
    }
}

let excerpt;
{
    let novel = String::from("Call me Ishmael. Some years ago...");
    excerpt = Excerpt::new(&novel);
    // excerpt lives while novel is alive — OK
    println!("{}", excerpt.text);
} // novel dropped here
// println!("{}", excerpt.text); // ERROR: novel is gone
```

**`'static` misconceptions**:
```rust
// 'static means "CAN live for the entire program", NOT "lives forever"
// String literals are &'static str (baked into the binary)
let s: &'static str = "hello";

// Owned types satisfy T: 'static because they contain no borrowed data
fn spawn_thread<T: Send + 'static>(val: T) { /* ... */ }
spawn_thread(String::from("owned")); // OK: String is 'static
spawn_thread(42i32);                 // OK: i32 is 'static
// spawn_thread(&some_local);        // ERROR: &local is not 'static
```

**Lifetime bounds on generics**:
```rust
// T: 'a means "T doesn't contain references shorter than 'a"
fn store_in_vec<'a, T: 'a>(vec: &mut Vec<&'a T>, item: &'a T) {
    vec.push(item);
}

// Common in trait bounds:
struct Ref<'a, T: 'a> {
    data: &'a T,
}
```

**Self-referential structs — why they're hard**:
```rust
// This CANNOT work in safe Rust:
// struct SelfRef {
//     data: String,
//     slice: &str, // wants to point into data — but data can move!
// }

// Solutions:
// 1. Use indices instead of references
struct SafeVersion {
    data: String,
    start: usize,
    end: usize,
}

// 2. Use ouroboros crate (generates safe self-referential structs)
// 3. Use Pin<Box<T>> to prevent moves + unsafe
// 4. Restructure to avoid self-references (usually the best option)
```

**Skill to practice**: Write a struct holding two references with different lifetimes. Return a reference tied to just one of them from a method. Verify the borrow checker accepts your code.

---

## Skill 186: Type System Decision Matrix

**Pattern**: Choosing the right type system tool for the job. Use this decision matrix when designing APIs.

**Generics vs Trait Objects vs Enums**:

| Criterion | Generics `<T: Trait>` | Trait Objects `dyn Trait` | Enums |
|-----------|----------------------|--------------------------|-------|
| Set of types | Open (any type) | Open (any type) | Closed (fixed variants) |
| Known at | Compile time | Runtime | Compile time |
| Performance | Best (monomorphized) | Vtable overhead | Best (no indirection) |
| Heterogeneous collection | No | Yes | Yes |
| Binary size | Larger (copies per type) | Smaller | Smallest |
| Pattern matching | No | No | Yes |
| **Choose when** | Performance-critical, known types | Plugin systems, type erasure | Known, fixed set of variants |

**Associated Types vs Generic Parameters**:

| Question | Associated Type | Generic Parameter |
|----------|----------------|-------------------|
| How many impls per type? | Exactly one | Multiple |
| Example | `Iterator::Item` | `From<T>` |
| Ergonomics | Better (no turbofish) | Requires specifying `<T>` |
| **Choose when** | The trait has ONE natural output type | The type interacts with MANY others |

**`impl Trait` vs Explicit Generics**:

| Scenario | `impl Trait` | Explicit `<T>` |
|----------|-------------|----------------|
| Simple functions | `fn foo(x: impl Display)` | Overkill |
| Need to name T | Cannot | `fn foo<T: Display>(x: T, y: T)` |
| Return opaque type | `-> impl Iterator` | Cannot hide type |
| Turbofish needed | Cannot use | `foo::<i32>(x)` |
| **Choose when** | Single use, simple | Named, reused, or turbofish needed |

**Newtype vs Type Alias**:

| | Newtype `struct Uid(u64)` | Type Alias `type Uid = u64` |
|---|---|---|
| Type safety | Yes (distinct type) | No (same type) |
| Trait impls | Can add custom | Inherits all from inner |
| Runtime cost | Zero | Zero |
| **Choose when** | Prevent mixing up types | Readability shortcut only |

**Const Generics vs Macros**:

| | Const Generics `<const N: usize>` | Macros |
|---|---|---|
| Type safety | Full | Limited |
| Error messages | Clear | Cryptic |
| Limitations | Integer/bool/char only | Arbitrary |
| **Choose when** | Size/count parameterization | Complex code generation |

**When to reach for unsafe vs refactor**:

| Situation | Refactor first | Unsafe may be needed |
|-----------|---------------|---------------------|
| Self-referential struct | Use indices or separate structs | Pin + careful invariants |
| Need interior mutability | Cell/RefCell/Mutex | Only for custom sync primitives |
| FFI boundary | N/A | Always required for FFI calls |
| Performance-critical inner loop | Try safe alternatives first | Unchecked indexing after profiling |
| **Rule** | Always try safe Rust first | Document with SAFETY comments |

**Quick decision flowchart**:
```
Need polymorphism?
├── Fixed set of types → Enum
├── Open set, known at compile time → Generics
└── Open set, chosen at runtime → Trait objects (dyn)

Need type parameters on a trait?
├── One impl per type → Associated type
└── Many impls per type → Generic parameter

Need compile-time size/count?
├── Simple integer parameter → Const generics
└── Complex code generation → Macro

Need type safety for a value?
├── Just readability → Type alias
└── Prevent misuse → Newtype
```

**Skill to practice**: Take an existing project and audit the type design decisions. For each `dyn Trait`, ask if an enum or generic would be better. For each type alias, ask if a newtype would add safety.

---

## Quick Reference

| Skill | Pattern | One-Liner |
|-------|---------|-----------|
| 175 | Generics Deep Dive | Bounds, `where` clauses, defaults, turbofish, const generics basics |
| 176 | Associated Types vs Generics | One impl = associated, many impls = generic parameter |
| 177 | GATs | `type Item<'a>` — lending iterators, self-referential returns |
| 178 | HRTBs | `for<'a>` — closures and traits that work with any lifetime |
| 179 | Const Generics | `<const N: usize>` — type-safe fixed-size abstractions |
| 180 | Trait Objects | `dyn Trait` — vtable dispatch, object safety, heterogeneous collections |
| 181 | impl Trait (RPIT/APIT) | Opaque types in argument and return position, RPITIT |
| 182 | Type-Level Programming | PhantomData markers, typestate, compile-time enforcement |
| 183 | Newtype Pattern | Type safety, orphan rule bypass, `repr(transparent)`, derive_more |
| 184 | Closures & Fn Traits | Fn/FnMut/FnOnce hierarchy, capture modes, returning closures |
| 185 | Advanced Lifetimes | Elision rules, multiple lifetimes, `'static`, self-referential structs |
| 186 | Decision Matrix | When to use generics vs dyn vs enum, associated vs generic, newtype vs alias |

---

## Change Log

| Date | Change |
|------|--------|
| 2026-02-20 | Initial version — Skills 175-186 from type system analysis |
