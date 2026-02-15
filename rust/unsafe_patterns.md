# Unsafe Rust Patterns — Writing Sound Unsafe Code

A deep dive into the rules, tools, and patterns for writing correct unsafe Rust. Continues from Skills 1-62.

---

## Skill 63: The Five Unsafe Superpowers (and What They Don't Disable)

**Pattern:** `unsafe` unlocks exactly five capabilities. Everything else — borrow checker, type system, lifetimes — is still enforced.

**The five superpowers:**
```rust
unsafe {
    // 1. Dereference a raw pointer
    let val = *raw_ptr;

    // 2. Call an unsafe function
    let v = Vec::from_raw_parts(ptr, len, cap);

    // 3. Access or modify a mutable static
    static mut COUNTER: u32 = 0;
    COUNTER += 1;

    // 4. Implement an unsafe trait (at item level)
    // unsafe impl Send for MyType {}

    // 5. Access fields of a union
    union U { i: i32, f: f32 }
    let u = U { i: 42 };
    let f = u.f;
}
```

**What `unsafe` does NOT disable:**
```rust
unsafe {
    let mut s = String::from("hello");
    let r1 = &s;
    let r2 = &mut s;  // ERROR: borrow checker still running

    let x: i32 = 42;
    let y: &str = x;  // ERROR: type system still running
}
```

**The mental model:** `unsafe` is a contract. You tell the compiler: "I've verified the invariants you can't check." If you're wrong, the entire program has undefined behavior — not just the unsafe block.

**Skill to practice:** Before writing `unsafe`, ask: "Which of the five superpowers do I need?" If the answer is none, you don't need `unsafe`.

---

## Skill 64: Raw Pointers — Rules and Arithmetic

**Pattern:** Raw pointers (`*const T`, `*mut T`) have no safety guarantees. Creating them is safe; dereferencing requires `unsafe`.

| Property | `&T` / `&mut T` | `*const T` / `*mut T` |
|---|---|---|
| Non-null | Guaranteed | No |
| Aligned | Guaranteed | No |
| Points to valid data | Guaranteed | No |
| Borrow-checked | Yes | No |
| Dereference | Safe | Unsafe |

**Pointer arithmetic:**
```rust
let arr = [10i32, 20, 30, 40, 50];
let base: *const i32 = arr.as_ptr();

unsafe {
    // .add(n) — offset by n elements. Both start and result must be in-bounds.
    assert_eq!(*base.add(2), 30);

    // .offset(n) — like add but isize (can go backwards)
    assert_eq!(*base.add(3).offset(-1), 30);

    // .wrapping_add(n) — no UB even if out of bounds, but dereferencing may be UB
    let oob = base.wrapping_add(100); // OK to create
    // *oob would be UB
}
```

**Key rule:** `.add()` and `.offset()` require both start and result pointers to be within the same allocation (or one-past-end). Violating this is UB even without dereferencing.

**Casting:**
```rust
let p: *const i32 = &42;
let p_mut: *mut i32 = p as *mut i32;     // const → mut
let p_u8: *const u8 = p as *const u8;    // between types
let addr: usize = p as usize;            // to integer
let p_back: *const i32 = addr as *const i32; // from integer
```

**Skill to practice:** Prefer `.add()` over manual byte arithmetic. Use `wrapping_add` when you're computing addresses without dereferencing (e.g., bounds checking).

---

## Skill 65: `NonNull<T>` — Non-Null Pointers with Niche Optimization

**Pattern:** `NonNull<T>` wraps a raw pointer with a compile-time non-null guarantee, enabling zero-cost `Option`.

```rust
use std::ptr::NonNull;
use std::mem::size_of;

// Option<NonNull<T>> is the same size as NonNull<T>
// because None uses the null niche
assert_eq!(size_of::<NonNull<i32>>(), size_of::<Option<NonNull<i32>>>());
```

**This is why `Box`, `Vec`, `Rc`, `Arc` all use `NonNull` internally:**
```rust
// Option<Box<T>> is the same size as Box<T> — zero overhead
assert_eq!(size_of::<Option<Box<i32>>>(), size_of::<Box<i32>>());
```

**`NonNull::dangling()` for empty allocations:**
```rust
// For zero-length allocations, use a dangling but non-null pointer
let ptr: NonNull<u8> = NonNull::dangling(); // aligned, non-null, not dereferenceable
```

**Covariance:** `NonNull<T>` is covariant in `T` (like `*const T`), which is correct for owning containers like `Vec`. If building a type with interior mutability, add `PhantomData<fn(T) -> T>` for invariance.

**Skill to practice:** Use `NonNull` instead of `*mut T` in data structure internals. You get null-safety, niche optimization, and clearer intent.

---

## Skill 66: `MaybeUninit<T>` — Safe Uninitialized Memory

**Pattern:** Uninitialized memory is UB in Rust, even for integers. `MaybeUninit<T>` is the safe way to handle it.

```rust
use std::mem::MaybeUninit;

// BAD: instant UB, even for i32
// let x: i32 = unsafe { std::mem::uninitialized() }; // deprecated

// GOOD: MaybeUninit tells the compiler "no valid T here yet"
let mut x = MaybeUninit::<i32>::uninit();
x.write(42);
let val: i32 = unsafe { x.assume_init() }; // SAFETY: we just wrote 42
```

**The `assume_init()` contract — caller must guarantee:**
1. Every byte has been written
2. The bit pattern is valid for `T`

**Array initialization pattern:**
```rust
let mut arr = [const { MaybeUninit::<String>::uninit() }; 5];
for (i, elem) in arr.iter_mut().enumerate() {
    elem.write(format!("element {i}"));
}
// SAFETY: all elements initialized in the loop
let arr: [String; 5] = unsafe { arr.map(|x| x.assume_init()) };
```

**`zeroed()` — when all-zeros is valid:**
```rust
let x: i32 = unsafe { MaybeUninit::zeroed().assume_init() };  // OK: 0 is valid i32
let b: bool = unsafe { MaybeUninit::zeroed().assume_init() };  // OK: 0 is false
// let r: &i32 = unsafe { MaybeUninit::zeroed().assume_init() }; // UB: null ref!
```

**Skill to practice:** Use `MaybeUninit` for any buffer or array where elements are initialized incrementally. Never use `std::mem::uninitialized()`.

---

## Skill 67: `mem::transmute` — Last Resort Bit Reinterpretation

**Pattern:** `transmute<T, U>(val)` reinterprets bits of `T` as `U`. Requires equal sizes. Almost always has a safer alternative.

```rust
// Reinterpret f32 as its IEEE 754 bits
let bits: u32 = unsafe { std::mem::transmute(1.0f32) };
assert_eq!(bits, 0x3F800000);
```

**Common pitfalls:**
```rust
// Invalid bit pattern → UB
let b: bool = unsafe { std::mem::transmute(2u8) };  // UB: bool must be 0 or 1
let c: char = unsafe { std::mem::transmute(0xD800u32) }; // UB: surrogate

// Lifetime extension → unsound
fn extend<'a>(s: &'a str) -> &'static str {
    unsafe { std::mem::transmute(s) } // Compiles but unsound!
}
```

**Safer alternatives (prefer these):**
```rust
// f32 ↔ u32: use to_bits/from_bits
let bits = 1.0f32.to_bits();
let float = f32::from_bits(bits);

// Pointer casts: use `as`
let p: *const u8 = some_ptr as *const u8;

// Integer to enum: use match or TryFrom
fn color_from_u8(v: u8) -> Option<Color> {
    match v { 0 => Some(Color::Red), 1 => Some(Color::Green), _ => None }
}

// General byte reinterpretation: use bytemuck crate
// let x: u32 = bytemuck::cast(1.0f32);
```

**Skill to practice:** Before using `transmute`, check if `to_bits`/`from_bits`, `as` casts, `TryFrom`, or `bytemuck` can solve the problem. Use `transmute` only when nothing else works.

---

## Skill 68: `mem::forget` and `ManuallyDrop` — Suppressing Drop

**Pattern:** `mem::forget` consumes a value without running its destructor. It is *safe* because leaking is not memory-unsafety.

```rust
use std::mem::ManuallyDrop;

// mem::forget is implemented as:
pub fn forget<T>(t: T) {
    let _ = ManuallyDrop::new(t); // wraps t, preventing automatic drop
}
```

**FFI ownership transfer:**
```rust
fn transfer_to_c(data: Vec<u8>) {
    let mut data = ManuallyDrop::new(data);
    let ptr = data.as_mut_ptr();
    let len = data.len();
    unsafe {
        // C library takes ownership — we must NOT drop the Vec
        c_library_takes_ownership(ptr, len);
    }
    // data's destructor does NOT run
}
```

**Leak amplification (stdlib design principle):** If someone calls `mem::forget` on a `Drain` iterator, the worst that happens is memory leaks — never UB. Any type whose safety depends on `Drop` running is unsound.

**Skill to practice:** Use `ManuallyDrop` when you need precise control over when (or if) a destructor runs. Remember: safe code must tolerate leaks.

---

## Skill 69: The SAFETY Comment Convention

**Pattern:** Every unsafe block and unsafe impl must have a SAFETY comment explaining why the invariants hold.

**On `unsafe fn` declarations — document caller's obligations:**
```rust
/// # Safety
///
/// - `src` must be valid for reads.
/// - `src` must be properly aligned.
/// - `src` must point to a properly initialized value of type `T`.
pub unsafe fn read<T>(src: *const T) -> T { ... }
```

**At `unsafe` block call sites — explain why obligations are met:**
```rust
pub fn push(&mut self, value: T) {
    if self.len == self.buf.capacity() {
        self.buf.reserve(1);
    }
    unsafe {
        // SAFETY: We just ensured len < capacity via reserve().
        // The pointer at buf + len is within the allocation and
        // properly aligned. The slot is uninitialized, so we use
        // ptr::write (not assignment, which would drop garbage).
        std::ptr::write(self.as_mut_ptr().add(self.len), value);
    }
    self.len += 1;
}
```

**On `unsafe impl` — explain why the trait contract holds:**
```rust
// SAFETY: FixedVec owns its buffer exclusively. Sending it to
// another thread is safe if T itself is safe to send.
unsafe impl<T: Send> Send for FixedVec<T> {}
```

**Enforced by:** `clippy::undocumented_unsafe_blocks` — mandatory in production codebases.

**Skill to practice:** Write the SAFETY comment *before* writing the unsafe code. If you can't explain why it's sound, don't write it.

---

## Skill 70: Aliasing Rules and Stacked Borrows

**Pattern:** `&T` = no mutation. `&mut T` = exclusive access. Raw pointers must respect these rules too, or it's UB.

**The Stacked Borrows model:** Every memory location has a conceptual "borrow stack" tracking which pointers may access it. Creating a new reference pushes onto the stack and may invalidate older entries.

```rust
fn aliasing_example() {
    let mut x: i32 = 42;
    let ref1: &mut i32 = &mut x;          // Push Unique(ref1)
    let ptr: *mut i32 = ref1 as *mut i32;  // Push SharedRW(ptr)
    let ref2: &mut i32 = &mut *ref1;       // Push Unique(ref2), invalidates ptr

    *ref2 = 10;  // OK: ref2 is on top

    // unsafe { *ptr = 20; } // UB: ptr was invalidated when ref2 was created

    *ref1 = 30;  // OK after ref2 goes out of scope
}
```

**Common aliasing UB:**
```rust
// TWO &mut to same data — instant UB even without writing
unsafe {
    let r1: &mut i32 = &mut *p1;
    let r2: &mut i32 = &mut *p2; // UB: r1 is invalidated
}

// Mutating through &T without UnsafeCell — UB
let x: i32 = 42;
let r: &i32 = &x;
unsafe { *(r as *const i32 as *mut i32) = 100; } // UB!

// CORRECT: use UnsafeCell
let cell = UnsafeCell::new(42);
unsafe { *cell.get() = 100; } // OK: UnsafeCell opts out
```

**Miri catches all of this:** Run `cargo +nightly miri test` to detect aliasing violations with detailed traces.

**Skill to practice:** Never create two `&mut` to the same data. Never mutate through `&T` without `UnsafeCell`. Run Miri on all unsafe code.

---

## Skill 71: Unsafe Traits — Send, Sync, and Custom Contracts

**Pattern:** An `unsafe trait` defines a contract in prose. `unsafe impl` asserts the implementor has verified it.

### Send and Sync

```rust
// Send: safe to move to another thread
// Sync: safe to share &T across threads

// Send + Sync: i32, String, Vec<T>, Arc<Mutex<T>>
// Send + !Sync: Cell<T>, RefCell<T> (unsynchronized interior mutability)
// !Send + !Sync: Rc<T> (non-atomic refcount)
// !Send + Sync: MutexGuard<T> (must unlock on same thread)
```

**Why they're unsafe:** Implementing them incorrectly causes data races — UB the borrow checker can't prevent.

**Negative impls and PhantomData:**
```rust
// Opt out of Send/Sync via PhantomData containing a raw pointer:
struct ThreadLocal {
    data: i32,
    _not_send: PhantomData<*const ()>,  // *const T is !Send + !Sync
}
```

### Custom unsafe traits

```rust
/// # Safety
/// Implementors must guarantee that `size()` returns the exact
/// element count. Unsafe code relies on this for unchecked indexing.
unsafe trait TrustedLen: Iterator {
    fn size(&self) -> usize;
}

/// SAFETY: MyRange yields exactly (end - start) elements.
unsafe impl TrustedLen for MyRange {
    fn size(&self) -> usize { self.end.saturating_sub(self.start) }
}
```

**Skill to practice:** Only write `unsafe impl` when you can explain *exactly* why the contract holds. Document it in a SAFETY comment.

---

## Skill 72: FFI Safety Patterns

**Pattern:** Interfacing with C requires explicit attention to layout, ownership, strings, and panic safety.

### `repr(C)` for layout compatibility
```rust
#[repr(C)]
struct Point { x: f64, y: f64 }  // C-compatible field order and padding

#[repr(i32)]
enum ErrorCode { Success = 0, NotFound = -1 }  // C-style enum
```

### Opaque types pattern
```rust
/// Opaque handle — we never construct this, only hold pointers
#[repr(C)]
pub struct DbConnection {
    _private: [u8; 0],
    _marker: PhantomData<(*const (), UnsafeCell<()>)>, // !Send, !Sync
}

extern "C" {
    fn db_open(path: *const c_char) -> *mut DbConnection;
    fn db_close(conn: *mut DbConnection);
}
```

### Safe wrapper with RAII
```rust
pub struct Database { conn: *mut DbConnection }

impl Database {
    pub fn open(path: &str) -> Result<Self, NulError> {
        let c_path = CString::new(path)?;
        let conn = unsafe { db_open(c_path.as_ptr()) };
        if conn.is_null() { panic!("db_open returned null"); }
        Ok(Database { conn })
    }
}

impl Drop for Database {
    fn drop(&mut self) {
        // SAFETY: conn was returned by db_open, not yet closed
        unsafe { db_close(self.conn); }
    }
}
```

### CString/CStr — string handling
```rust
// Rust → C: CString adds null terminator
let c_str = CString::new("hello")?;
unsafe { c_function(c_str.as_ptr()); }

// COMMON BUG: temporary CString dangling pointer
// let ptr = CString::new("hello").unwrap().as_ptr(); // DANGLING!

// C → Rust: CStr borrows existing null-terminated data
let rust_str: &str = unsafe { CStr::from_ptr(c_ptr) }.to_str()?;
```

### Panic safety at FFI boundaries
```rust
#[no_mangle]
pub extern "C" fn safe_entry(input: i32) -> i32 {
    // Unwinding across extern "C" is UB — always catch panics
    std::panic::catch_unwind(|| {
        if input < 0 { panic!("negative"); }
        input * 2
    }).unwrap_or(-1)
}
```

**Skill to practice:** Always wrap FFI in a safe Rust type with `Drop`. Use `CString`/`CStr` for strings. Catch panics at every FFI boundary.

---

## Skill 73: Variance and PhantomData in Unsafe Code

**Pattern:** Variance determines lifetime substitutability. `PhantomData` controls what variance your type has.

### The three variances
```rust
// Covariant: &'long T can be used where &'short T is expected (longer is fine)
// Invariant: no substitution allowed (Cell<T> — interior mutability breaks covariance)
// Contravariant: fn(&'short str) can be used where fn(&'long str) is expected
```

### Why `Vec<T>` is covariant but `Cell<T>` is invariant

`Cell<T>` allows mutation through `&Cell<T>`. If it were covariant, you could smuggle a short-lived reference into a long-lived slot:
```rust
// If Cell were covariant (it's NOT — this is hypothetical UB):
// let cell_long: Cell<&'static str> = Cell::new("hello");
// let cell_short: &Cell<&'short str> = &cell_long; // hypothetical covariant upcast
// cell_short.set(short_lived);  // puts 'short into cell_long
// cell_long.get();              // reads 'short as 'static → DANGLING
```

### PhantomData controls variance

```rust
use std::marker::PhantomData;

// Covariant in T (owning container, like Vec):
struct MyVec<T> {
    ptr: NonNull<T>,
    _marker: PhantomData<T>,  // "I own T" → covariant
}

// Invariant in T (interior mutability or mutable access through &self):
struct MyCell<T> {
    ptr: NonNull<T>,
    _marker: PhantomData<fn(T) -> T>,  // contravariant + covariant = invariant
}
```

### PhantomData for drop checker
```rust
struct LinkedList<T> {
    head: Option<NonNull<Node<T>>>,
    // Without this, compiler doesn't know dropping LinkedList drops T values
    _marker: PhantomData<Box<Node<T>>>,  // "I own Node<T>" for drop check
}
```

**Skill to practice:** For owning containers, use `PhantomData<T>`. For types with interior mutability, use `PhantomData<fn(T) -> T>`. Always include `PhantomData` when using raw pointers to owned data.

---

## Skill 74: Common Unsoundness Patterns

A catalog of bugs to watch for in unsafe code.

### Use-after-free via reallocation
```rust
let mut v = vec![1, 2, 3];
let ptr = v.as_ptr();  // points into v's buffer
v.push(4);             // may reallocate → ptr is dangling
// unsafe { *ptr }     // UB!
```

### Data races
```rust
// Two threads writing to UnsafeCell without synchronization → UB
// Even two threads where one writes and one reads → UB
```

### Invalid values
```rust
// bool must be 0 or 1
// &T must be non-null and aligned
// char must be a valid Unicode scalar
// enum must have a valid discriminant
// Violating ANY of these is instant UB, even if you "never look at the value"
```

### Uninitialized reads
```rust
// Reading uninitialized memory is UB even for integers
// Always use MaybeUninit
```

### Stack-use-after-return
```rust
fn bad() -> *const i32 {
    let local = 42;
    &local as *const i32  // dangling when function returns!
}
```

**Skill to practice:** Run `cargo +nightly miri test` on all unsafe code. Miri catches all of these.

---

## Skill 75: Testing Unsafe Code

**Pattern:** Use multiple layers of testing to catch UB that human reasoning misses.

### Layer 1: Miri — the primary UB detector
```bash
cargo +nightly miri test
```
Catches: OOB access, use-after-free, Stacked Borrows violations, data races, invalid values, memory leaks.

### Layer 2: Drop correctness tests
```rust
use std::sync::atomic::{AtomicUsize, Ordering};
static DROPS: AtomicUsize = AtomicUsize::new(0);

struct Canary;
impl Drop for Canary {
    fn drop(&mut self) { DROPS.fetch_add(1, Ordering::SeqCst); }
}

#[test]
fn drops_all_elements() {
    DROPS.store(0, Ordering::SeqCst);
    { let mut v = MyVec::new(); v.push(Canary); v.push(Canary); v.push(Canary); }
    assert_eq!(DROPS.load(Ordering::SeqCst), 3);
}
```

### Layer 3: Edge case tests
```rust
#[test]
fn test_zst() { let mut v: MyVec<()> = MyVec::new(); v.push(()); assert_eq!(v.len(), 1); }

#[test]
fn test_empty() { let v: MyVec<i32> = MyVec::new(); assert_eq!(v.pop(), None); }
```

### Layer 4: Fuzzing
```rust
// cargo-fuzz with arbitrary inputs:
// fuzz_target!(|ops: Vec<Operation>| {
//     let mut container = MyVec::new();
//     for op in ops { match op { Push(v) => container.push(v), Pop => { container.pop(); } } }
// });
```

### Layer 5: Property-based testing
```rust
// proptest! {
//     fn push_pop_roundtrip(values: Vec<i32>) {
//         let mut v = MyVec::new();
//         for &val in &values { v.push(val); }
//         for &val in values.iter().rev() { assert_eq!(v.pop(), Some(val)); }
//     }
// }
```

**Skill to practice:** Every unsafe data structure should have: Miri tests, drop canary tests, ZST tests, and empty-state tests at minimum. Add fuzzing for anything that takes external input.

---

## Skill 76: The Unsafe Encapsulation Checklist

A systematic process for writing sound unsafe code.

| # | Check | How to verify |
|---|-------|---------------|
| 1 | **Document invariants** | `// SAFETY:` on every `unsafe` block and `unsafe impl` |
| 2 | **Minimize scope** | Only the raw pointer op is inside `unsafe {}` |
| 3 | **Runtime checks** | `debug_assert!` for bounds, alignment, non-null, len ≤ cap |
| 4 | **Test with Miri** | `cargo +nightly miri test` passes |
| 5 | **Safe public API** | All `pub fn` are safe; unsafe is internal only |
| 6 | **Variance + PhantomData** | Correct for all type/lifetime parameters |
| 7 | **Drop correctness** | No double-free, no use-after-free, elements dropped |
| 8 | **Send/Sync** | Manually verified and `unsafe impl`'d (or opted out) |

**The soundness principle:** Safe code must *never* be able to cause UB, no matter what inputs it receives. If a safe API can trigger UB, that's a soundness bug in the library.

**Privacy is load-bearing:** `pub(crate)` fields can be relied upon in safety arguments because only code within the crate can access them. This is why Rust's module system is part of the safety story.

**Skill to practice:** Go through this checklist for every module containing `unsafe`. Write the SAFETY comment *before* the unsafe code. If you can't justify it, don't write it.

---

## Quick Reference: When to Apply Each Skill

| Situation | Skill |
|-----------|-------|
| Understanding what unsafe allows | #63 Five superpowers |
| Working with raw pointers | #64 Pointer rules and arithmetic |
| Data structure internals | #65 NonNull |
| Uninitialized memory / buffers | #66 MaybeUninit |
| Bit reinterpretation | #67 transmute (prefer alternatives) |
| Preventing Drop / FFI ownership | #68 forget + ManuallyDrop |
| Documenting unsafe code | #69 SAFETY comments |
| Avoiding aliasing UB | #70 Stacked Borrows |
| Thread safety assertions | #71 Send/Sync and unsafe traits |
| Calling C code | #72 FFI patterns |
| Lifetime correctness in unsafe types | #73 Variance + PhantomData |
| Recognizing UB patterns | #74 Common unsoundness |
| Verifying correctness | #75 Testing with Miri/fuzzing |
| Systematic unsafe review | #76 Encapsulation checklist |

---

## Change Log

- **2026-02-15**: Initial version — derived from the Rustonomicon, stdlib source (ptr, mem, marker), Stacked Borrows model, and FFI conventions.
