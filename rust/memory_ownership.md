# Memory & Ownership Patterns from Rust's Standard Library

Skills and patterns extracted from studying `Vec`, `String`, `Box`, `Rc`, `Arc`, `Cell`, `RefCell`, `UnsafeCell`, and `Pin` in Rust's standard library.

---

## Skill 11: The Stack Handle + Heap Data Layout

**Pattern:** Heap-owning types use a fixed-size stack handle (pointer + metadata) that owns dynamically-sized heap data.

```rust
// Vec<T> is always 24 bytes on the stack, regardless of heap size
// Stack handle:          Heap:
// +------+-----+-----+    +---+---+---+---+---+---+---+---+
// | ptr  | len | cap | -> | 0 | 1 | 2 | 3 | . | . | . | . |
// +------+-----+-----+    +---+---+---+---+---+---+---+---+
//                           ^--- len=4 used --^ ^- cap=8 -^

// String is a newtype around Vec<u8> — same layout, plus UTF-8 invariant
pub struct String {
    vec: Vec<u8>,
}
```

**Why it matters:** Moving a `Vec` or `String` is always a cheap 24-byte memcpy, no matter how large the heap data is.

**Skill to practice:** When designing owned heap types, use a fixed-size handle struct. Keep metadata (length, capacity, flags) in the handle to avoid pointer chasing.

---

## Skill 12: Layered Architecture (Separation of Concerns)

**Pattern:** Each layer in the type hierarchy adds exactly one concern.

```
String      — adds UTF-8 validity invariant
  └─ Vec<u8>    — adds element lifetimes (Drop for elements)
       └─ RawVec<u8>  — adds allocation management (grow, shrink)
            └─ Allocator    — raw alloc/dealloc interface
```

String doesn't manage memory. Vec doesn't validate encoding. Each layer does one thing.

**Similarly for smart pointers:**
```
Arc<Mutex<T>>
  Arc    — shared ownership via atomic refcount
  Mutex  — mutual exclusion for mutation
  T      — the actual data
```

**Skill to practice:** When building types that combine ownership, validation, or synchronization, layer them. Don't put allocation logic in your validation type.

---

## Skill 13: The Newtype Invariant Pattern

**Pattern:** Wrap an existing type in a newtype to enforce an additional invariant that the inner type doesn't guarantee.

```rust
// String wraps Vec<u8>, adding the UTF-8 invariant
pub struct String { vec: Vec<u8> }

// Safe: String -> Vec<u8> (dropping an invariant is always safe)
pub fn into_bytes(self) -> Vec<u8> { self.vec }

// Fallible: Vec<u8> -> String (must validate the invariant)
pub fn from_utf8(vec: Vec<u8>) -> Result<String, FromUtf8Error> {
    match str::from_utf8(&vec) {
        Ok(..) => Ok(String { vec }),
        Err(e) => Err(FromUtf8Error { bytes: vec, error: e }),
    }
}

// Unsafe escape hatch: skip validation (caller guarantees invariant)
pub unsafe fn from_utf8_unchecked(bytes: Vec<u8>) -> String {
    String { vec: bytes }
}
```

The three-tier API:
1. **Safe fallible** (`from_utf8`) — validates, returns `Result`
2. **Unsafe unchecked** (`from_utf8_unchecked`) — trusts the caller
3. **Safe infallible unwrap** (`into_bytes`) — dropping an invariant is always safe

**Skill to practice:** When you have a type that's "like X but with constraint Y", use a newtype. Provide safe checked constructors and unsafe unchecked ones.

---

## Skill 14: Amortized Growth Strategy

**Pattern:** Double capacity on reallocation to achieve amortized O(1) insertion.

```rust
// Vec::push fast path — no allocation if capacity available
pub fn push(&mut self, value: T) {
    if self.len == self.buf.capacity() {
        self.buf.reserve_for_push(self.len); // slow path
    }
    unsafe {
        let end = self.as_mut_ptr().add(self.len);
        ptr::write(end, value);
        self.len += 1;
    }
}

// Growth policy: max(current * 2, required)
// Minimum allocation: 8 elements for non-ZSTs
let cap = cmp::max(self.cap * 2, required);
let cap = cmp::max(cap, 8);
```

**The reserve API gives users control:**
- `with_capacity(n)` — pre-allocate when you know the size (avoids all reallocs)
- `reserve(n)` — ensure room for n more, may over-allocate
- `reserve_exact(n)` — ensure room for exactly n more, no over-allocation
- `shrink_to_fit()` — release excess capacity after building

**Skill to practice:** If you know the final size, use `with_capacity`. For types you design, provide both eager (`reserve`) and exact (`reserve_exact`) allocation control.

---

## Skill 15: RAII and the Drop Hierarchy

**Pattern:** Resource cleanup happens automatically through `Drop`, forming a deterministic hierarchy.

```rust
// Vec::drop — two-phase cleanup
impl<T, A: Allocator> Drop for Vec<T, A> {
    fn drop(&mut self) {
        unsafe {
            // Phase 1: Drop each element (call their destructors)
            ptr::drop_in_place(ptr::slice_from_raw_parts_mut(
                self.as_mut_ptr(), self.len
            ));
        }
        // Phase 2: RawVec::drop runs, deallocating the heap buffer
    }
}
```

**A complex drop cascade:**
```
Vec<Arc<Mutex<Connection>>>::drop()
  → for each element: Arc::drop()
    → decrement strong count
    → if strong == 0:
      → Mutex::drop() (release OS resources)
        → Connection::drop() (close socket)
      → if weak == 0: deallocate
```

**Key rules:**
- Custom `Drop` impl runs first
- Then fields drop in declaration order
- `String::drop` is zero-cost: `u8` is `Copy`, so element drop is a no-op — just a single `dealloc`

**Skill to practice:** Lean on RAII for cleanup. Never write manual `free`/`close` calls when Drop can handle it. Design types so dropping them does the right thing.

---

## Skill 16: Deref Coercion for Owned/Borrowed Duality

**Pattern:** Implement `Deref` to make owned types transparently usable as their borrowed counterparts.

```rust
impl Deref for String {
    type Target = str;
    fn deref(&self) -> &str {
        unsafe { str::from_utf8_unchecked(&self.vec) }
    }
}

impl<T> Deref for Vec<T> {
    type Target = [T];
    fn deref(&self) -> &[T] {
        unsafe { slice::from_raw_parts(self.as_ptr(), self.len) }
    }
}
```

**Why this is powerful:**
- `Vec<T>` inherits all ~100 methods of `[T]` (sort, binary_search, windows, chunks...)
- Functions taking `&str` work with both `&str` and `&String`
- Functions taking `&[T]` work with both `&[T]` and `&Vec<T>`

```rust
fn process(data: &[i32]) { /* ... */ }
let v: Vec<i32> = vec![1, 2, 3];
process(&v); // auto-deref: &Vec<i32> → &[i32]
```

**Skill to practice:** Write functions that accept the *borrowed* form (`&str`, `&[T]`, `&Path`). This maximizes compatibility with both owned and borrowed callers.

---

## Skill 17: Unsafe Encapsulation — Safe Shell, Unsafe Core

**Pattern:** Confine unsafe operations behind a safe public API with documented invariants.

**The three tiers of safety in the stdlib:**

```rust
// Tier 1: Fully safe public API (bounds-checked)
pub fn swap_remove(&mut self, index: usize) -> T {
    if index >= self.len() {
        panic!("index out of bounds");
    }
    unsafe {
        let value = ptr::read(self.as_ptr().add(index));
        ptr::copy(base_ptr.add(len - 1), base_ptr.add(index), 1);
        self.set_len(len - 1);
        value
    }
}

// Tier 2: Public unsafe API (caller guarantees invariants)
/// # Safety
/// - `new_len` must be ≤ `capacity()`
/// - Elements at `old_len..new_len` must be initialized
pub unsafe fn set_len(&mut self, new_len: usize) {
    debug_assert!(new_len <= self.capacity());
    self.len = new_len;
}

// Tier 3: Internal unsafe (implementation detail)
unsafe { ptr::write(end, value) }
```

**SAFETY comment convention:**
```rust
// Every unsafe block must have a SAFETY comment explaining WHY it's sound:
unsafe {
    // SAFETY: We just verified index < len, so the pointer arithmetic
    // stays within the allocated region. The value at this index is
    // initialized because len tracks initialized elements.
    let value = ptr::read(self.as_ptr().add(index));
}
```

**Micro-optimization pattern:** Cold/unlikely paths are marked to keep hot paths small:
```rust
#[cold]
#[inline(never)]
fn assert_failed(index: usize, len: usize) -> ! {
    panic!("index {index} out of bounds for length {len}");
}
```

**Skill to practice:** When you must use `unsafe`, wrap it in a safe function. Document the invariants. Use `debug_assert!` to catch violations in tests.

---

## Skill 18: Smart Pointer Design — Single vs Shared Ownership

**Pattern:** Choose your smart pointer based on the ownership model needed.

| Property | `Box<T>` | `Rc<T>` | `Arc<T>` |
|---|---|---|---|
| Ownership | Single | Shared | Shared |
| Thread-safe | Yes (if T is) | No (`!Send`, `!Sync`) | Yes (if T is) |
| Ref counting | None | `Cell<usize>` | `AtomicUsize` |
| `DerefMut` | Yes | No | No |
| Clone | Deep clone of T | Increment count | Atomic increment |
| Mutation | Direct `&mut` | Needs `RefCell<T>` | Needs `Mutex<T>` |
| Overhead | 0 (just a pointer) | 2 `usize` counts | 2 `AtomicUsize` counts |

**Key design insights from the source:**

**Box gives `DerefMut` because single ownership = exclusive access:**
```rust
impl<T: ?Sized> DerefMut for Box<T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.0.as_ptr() }
    }
}
```

**Rc/Arc do NOT give `DerefMut` — multiple owners can't all have `&mut T`:**
```rust
// Rc only implements Deref, NOT DerefMut
impl<T: ?Sized> Deref for Rc<T> {
    type Target = T;
    fn deref(&self) -> &T { ... }
}
// DerefMut is deliberately absent
```

**The exception — `Rc::get_mut` when you're the only owner:**
```rust
pub fn get_mut(this: &mut Rc<T>) -> Option<&mut T> {
    if Rc::is_unique(this) {
        unsafe { Some(Rc::get_mut_unchecked(this)) }
    } else {
        None
    }
}
```

**Skill to practice:** Default to `Box`. Use `Rc` only when you truly need shared ownership in single-threaded code. Use `Arc` only when sharing across threads. Never reach for `Arc` when `Rc` suffices.

---

## Skill 19: Atomic Ordering in Arc (Thread-Safe Reference Counting)

**Pattern:** Use the minimum sufficient memory ordering for each operation.

```rust
// Clone: Relaxed is enough — just recording "one more owner exists"
fn clone(&self) -> Self {
    self.inner().strong.fetch_add(1, Relaxed);
    // ...
}

// Drop: Release on decrement, Acquire fence before deallocation
fn drop(&mut self) {
    if self.inner().strong.fetch_sub(1, Release) != 1 {
        return; // Not the last owner
    }
    // CRITICAL: This fence synchronizes with all the Release stores
    // from other threads' drops. Ensures we see all writes to the
    // data before we deallocate it.
    atomic::fence(Acquire);
    unsafe { self.drop_slow(); }
}
```

**The synchronization chain:**
```
Thread A writes to data
  → Thread A: fetch_sub(1, Release)     // "publishes" A's writes
    → Thread B: fence(Acquire)           // "observes" all prior writes
      → Thread B deallocates safely
```

**Why `Rc` uses `Cell<usize>` instead:** No atomics needed for single-threaded code. The `!Send + !Sync` bounds on `Rc` enforce this at compile time.

**Skill to practice:** When designing thread-safe types, choose the weakest ordering that's correct. `Relaxed` for non-synchronizing updates, `Release`/`Acquire` pairs for publish/observe patterns.

---

## Skill 20: `UnsafeCell` — The Foundation of All Interior Mutability

**Pattern:** `UnsafeCell<T>` is the *only* legal way to get `&mut` from `&` in Rust. Everything else builds on it.

```rust
#[lang = "unsafe_cell"]
#[repr(transparent)]
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}

impl<T: ?Sized> UnsafeCell<T> {
    // The ONLY place in Rust where &T → *mut T is sound
    pub const fn get(&self) -> *mut T {
        self as *const UnsafeCell<T> as *const T as *mut T
    }
}
```

**Everything builds on it:**
```
              Your Code
             /    |     \
        Cell   RefCell   Mutex / RwLock
         |       |            |
         +---+---+      std::sync primitives
             |                |
        UnsafeCell        UnsafeCell
             |                |
    Compiler lang item: disables
    aliasing assumptions for &T
```

**Skill to practice:** Never attempt interior mutability without `UnsafeCell`. If you're writing `unsafe` code that mutates through `&`, it *must* go through `UnsafeCell` or it's instant undefined behavior.

---

## Skill 21: Cell vs RefCell — Choose the Lightest Tool

**Pattern:** `Cell` is zero-cost but never gives out references. `RefCell` gives out references but has runtime cost and panic risk.

### Cell — No references escape, zero overhead

```rust
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}

impl<T: Copy> Cell<T> {
    pub fn get(&self) -> T {          // Returns a COPY, not a reference
        unsafe { *self.value.get() }
    }
}

impl<T> Cell<T> {
    pub fn set(&self, val: T) {       // Takes a value, not a reference
        let old = self.replace(val);
        drop(old);
    }
    pub fn replace(&self, val: T) -> T {
        unsafe { mem::replace(&mut *self.value.get(), val) }
    }
}
```

**Why Cell is safe:** It *never* hands out `&T` or `&mut T`. You copy values in and out. No aliasing possible because there are no aliases.

### RefCell — Runtime borrow checking

```rust
pub struct RefCell<T: ?Sized> {
    borrow: Cell<BorrowFlag>,    // isize: 0=unused, >0=N readers, <0=writer
    value: UnsafeCell<T>,
}

// borrow() → Ref<T> guard (increments reader count, decrements on drop)
// borrow_mut() → RefMut<T> guard (sets writer flag, clears on drop)
// Panics if rules violated at runtime
```

### Decision guide

```rust
// GOOD: Cell for simple Copy values — zero overhead, no panic risk
struct Counter { count: Cell<u32> }
impl Counter {
    fn increment(&self) { self.count.set(self.count.get() + 1); }
}

// BAD: RefCell where Cell would suffice — unnecessary overhead
struct Counter { count: RefCell<u32> }  // Don't do this
impl Counter {
    fn increment(&self) { *self.count.borrow_mut() += 1; }
}

// GOOD: RefCell when you genuinely need &T or &mut T
struct Cache { data: RefCell<HashMap<String, String>> }
impl Cache {
    fn get(&self, key: &str) -> Option<Ref<'_, String>> {
        // Need to return a reference into the map
        Ref::filter_map(self.data.borrow(), |m| m.get(key)).ok()
    }
}
```

**Skill to practice:** Always try `Cell` first. Only reach for `RefCell` when you need references to the inner data.

---

## Skill 22: The Interior Mutability Decision Matrix

**Pattern:** Match the interior mutability tool to your threading and access needs.

```
Single-threaded, Copy type, no refs needed  →  Cell<T>
Single-threaded, need &T or &mut T          →  RefCell<T>
Multi-threaded, exclusive access             →  Mutex<T>
Multi-threaded, many readers few writers     →  RwLock<T>
Multi-threaded, Copy type, lock-free         →  AtomicT (AtomicBool, AtomicUsize, etc.)
Write-once, read-many                        →  OnceCell<T> / OnceLock<T>
```

**Common combinations:**
```rust
Rc<RefCell<T>>       // Shared ownership + mutation, single-threaded
Arc<Mutex<T>>        // Shared ownership + mutation, multi-threaded
Arc<RwLock<T>>       // Shared ownership + read-heavy mutation, multi-threaded
```

**Why the type system enforces correctness:**
```rust
Arc<RefCell<T>>  // COMPILE ERROR: RefCell is !Sync, Arc requires T: Sync
Rc<Mutex<T>>     // Compiles but pointless: Rc is !Send, can't share across threads
```

**Skill to practice:** Let the compiler guide you. If `Arc<RefCell<T>>` doesn't compile, that's the type system telling you `RefCell` isn't thread-safe. Use `Mutex` instead.

---

## Skill 23: Pin for Self-Referential Types

**Pattern:** `Pin<P>` prevents a value from being moved, enabling self-referential types (critical for async/await).

```rust
// The problem: self-referential struct breaks if moved
struct SelfRef {
    data: String,
    ptr_to_data: *const String, // Points to self.data — dangles if moved!
}

// The solution: Pin guarantees the value stays at its memory address
pub struct Pin<Ptr> { pointer: Ptr }

// Safe construction ONLY for Unpin types (pinning is a no-op)
impl<Ptr: Deref<Target: Unpin>> Pin<Ptr> {
    pub const fn new(pointer: Ptr) -> Pin<Ptr> { ... }
}

// For !Unpin types: unsafe, caller must guarantee no moves
impl<Ptr> Pin<Ptr> {
    pub const unsafe fn new_unchecked(pointer: Ptr) -> Pin<Ptr> { ... }
}
```

**Where this matters — the Future trait:**
```rust
pub trait Future {
    type Output;
    // poll takes Pin<&mut Self> — the future won't be moved between polls
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

**The Unpin auto-trait:**
- Almost everything is `Unpin` (pinning has no effect, can freely move)
- `async fn` futures are `!Unpin` (they may be self-referential)
- Opt out with `PhantomPinned`

**Skill to practice:** You rarely need `Pin` directly unless writing async runtimes, custom futures, or intrusive data structures. Understand it conceptually so you can read async code fluently.

---

## Skill 24: Weak References to Break Cycles

**Pattern:** Use `Weak<T>` for non-owning references that don't prevent deallocation.

```rust
// The cycle problem:
//   Parent → Child (Rc)
//   Child → Parent (Rc)  ← Both strong counts stay > 0 forever. Memory leak!

// The solution:
struct Node {
    parent: RefCell<Weak<Node>>,       // Weak — doesn't keep parent alive
    children: RefCell<Vec<Rc<Node>>>,  // Strong — parent owns children
}
```

**The upgrade pattern:**
```rust
// Weak::upgrade() returns Option<Rc<T>>
// Returns Some if the value is still alive, None if it was dropped
if let Some(parent) = node.parent.borrow().upgrade() {
    // parent is an Rc<Node> — value is alive
} else {
    // parent was already dropped
}
```

**Allocation lifecycle with Weak:**
```
strong > 0, weak > 0  →  Value alive, allocation alive
strong = 0, weak > 0  →  Value DROPPED, allocation still alive (Weak reads counts)
strong = 0, weak = 0  →  Allocation FREED
```

**Skill to practice:** In any graph/tree structure with back-references, use `Weak` for the "upward" or "backward" edges to prevent cycles.

---

## Skill 25: PhantomData for Drop Checker Correctness

**Pattern:** Use `PhantomData<T>` to tell the compiler about logical ownership that raw pointers don't convey.

```rust
pub struct Rc<T: ?Sized> {
    ptr: NonNull<RcBox<T>>,
    phantom: PhantomData<RcBox<T>>,  // "I logically own RcBox<T>"
    alloc: A,
}
```

Without `PhantomData`, the drop checker wouldn't know that `Rc<T>` accesses `T` during `Drop` (through the raw pointer). This could allow dangling references.

**Also useful for niche optimization:**
```rust
// NonNull + PhantomData enable Option optimization:
assert_eq!(
    size_of::<Option<Rc<i32>>>(),
    size_of::<Rc<i32>>()
);
// None uses the null pointer niche — zero-cost Option!
```

**Skill to practice:** When storing raw pointers to owned data, add `PhantomData<T>` to express ownership to the compiler.

---

## Quick Reference: When to Apply Each Skill

| Situation | Skill |
|-----------|-------|
| Designing a heap type | #11 Stack handle + heap data |
| Combining ownership + validation | #12 Layered architecture |
| "Like X but with constraint Y" | #13 Newtype invariant |
| Growing collections | #14 Amortized growth |
| Resource cleanup | #15 RAII / Drop hierarchy |
| Owned type wrapping borrowed type | #16 Deref coercion |
| Using `unsafe` in a library | #17 Unsafe encapsulation |
| Choosing a smart pointer | #18 Box vs Rc vs Arc |
| Thread-safe reference counting | #19 Atomic ordering |
| Interior mutability foundation | #20 UnsafeCell |
| Simple mutation through `&` | #21 Cell vs RefCell |
| Choosing mutation strategy | #22 Interior mutability matrix |
| Async / self-referential types | #23 Pin |
| Graph structures with cycles | #24 Weak references |
| Raw pointers + ownership | #25 PhantomData |

---

## Change Log

- **2026-02-14**: Initial version — derived from analysis of Vec, String, Box, Rc, Arc, Cell, RefCell, UnsafeCell, and Pin in Rust's standard library.
