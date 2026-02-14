# Skills from Rust's Standard Library

These are the core coding skills and patterns demonstrated by how the Rust stdlib authors write code. Organized from foundational to advanced.

---

## Skill 1: The "Skinny Required, Fat Provided" Trait Design

**Pattern:** Require implementors to write the absolute minimum, then provide dozens of methods for free via default implementations.

**Example:** `Iterator` requires only `next()` — one method — and provides **75+ methods** (map, filter, fold, collect, sum, any, all, find, zip, chain, etc.) as defaults.

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>; // Only this is required

    // Everything else is free:
    fn map<B, F>(self, f: F) -> Map<Self, F> { ... }
    fn filter<P>(self, p: P) -> Filter<Self, P> { ... }
    fn fold<B, F>(self, init: B, f: F) -> B { ... }
    // ... 70+ more
}
```

**Skill to practice:** When designing a trait, identify the *one* primitive operation and build everything else on top of it.

---

## Skill 2: The `try_fold` Backbone (Internal Iteration)

**Pattern:** Implement one powerful internal method (`try_fold`), then define many consumers in terms of it. Override that one method for a whole family of optimizations.

```rust
// These ALL delegate to try_fold internally:
fn count(self)          -> usize        { self.fold(0, |c, _| c + 1) }
fn last(self)           -> Option<T>    { self.fold(None, |_, x| Some(x)) }
fn all(&mut self, f: F) -> bool         { self.try_fold(...) }
fn any(&mut self, f: F) -> bool         { self.try_fold(...) }
fn find(&mut self, p: P)-> Option<T>    { self.try_fold(...) }
fn position(...)        -> Option<usize>{ self.try_fold(...) }
```

**Skill to practice:** Find the "universal combinator" in your abstraction — one method that others can be expressed through. Override it once, optimize 15+ methods at once.

---

## Skill 3: Ownership-Aware API Design (`as_ref` / `as_mut` / consuming)

**Pattern:** For every operation, provide variants for each ownership mode so users never clone unnecessarily.

```rust
// Borrowing — inspect without consuming
fn as_ref(&self) -> Option<&T>
fn as_mut(&mut self) -> Option<&mut T>
fn as_deref(&self) -> Option<&T::Target>

// Consuming — take ownership
fn map(self, f: F) -> Option<U>
fn unwrap(self) -> T

// Mutating in place — extract without consuming self
fn take(&mut self) -> Option<T>        // leaves None behind
fn replace(&mut self, val: T) -> Option<T>
```

**Skill to practice:** When writing a method, ask: "Does the caller need to give up ownership?" If not, take `&self` or `&mut self`. Provide the full spectrum when it makes sense.

---

## Skill 4: Blanket Implementations for "Implement One, Get Many Free"

**Pattern:** Use blanket impls so users implement one trait and get related traits automatically.

```rust
// User implements From:
impl From<Meters> for Millimeters {
    fn from(m: Meters) -> Self { Millimeters(m.0 * 1000.0) }
}

// Blanket impl gives them Into for free:
impl<T, U> Into<U> for T where U: From<T> {
    fn into(self) -> U { U::from(self) }
}

// AND TryFrom for free (with Error = Infallible):
impl<T, U> TryFrom<U> for T where T: From<U> { ... }

// AND TryInto for free:
impl<T, U> TryInto<U> for T where U: TryFrom<T> { ... }
```

One `From` impl produces **four** usable conversion paths.

**Skill to practice:** Design trait hierarchies where blanket impls cascade benefits. Identify the "source" trait users should implement and derive the rest.

---

## Skill 5: Combinator Consistency (The Option/Result API Mirror)

**Pattern:** When two types are structurally similar, give them near-identical combinator APIs so users learn one and know both.

| Operation | `Option<T>` | `Result<T, E>` |
|-----------|------------|----------------|
| Transform | `map(f)` | `map(f)` + `map_err(f)` |
| Flatmap | `and_then(f)` | `and_then(f)` |
| Default | `unwrap_or(v)` | `unwrap_or(v)` |
| Alternative | `or_else(f)` | `or_else(f)` |
| Inspect | `inspect(f)` | `inspect(f)` + `inspect_err(f)` |
| Bridge | `ok_or(e)` | `ok()` |
| Transpose | `Option<Result<T,E>>` ↔ `Result<Option<T>,E>` |

**Skill to practice:** When you have related types, design symmetric APIs. Users who learn one can immediately use the other.

---

## Skill 6: Lazy Evaluation via Adapter Structs

**Pattern:** Adapter methods return a new struct that wraps the original, deferring computation until consumed.

```rust
// map() doesn't iterate — it returns a Map struct
fn map<B, F>(self, f: F) -> Map<Self, F> {
    Map::new(self, f)
}

// Map implements Iterator, calling f lazily in next()
impl<B, I: Iterator, F: FnMut(I::Item) -> B> Iterator for Map<I, F> {
    type Item = B;
    fn next(&mut self) -> Option<B> {
        self.iter.next().map(&mut self.f)
    }
}
```

Chaining `.map().filter().take(5)` creates nested structs with zero heap allocation. Work only happens when a consumer (`.collect()`, `.sum()`, `for` loop) drives the chain.

**Skill to practice:** Return descriptive structs instead of eagerly computing results. Let the consumer decide when and how much to evaluate.

---

## Skill 7: `#[must_use]` as a Correctness Guard

**Pattern:** Mark types and methods where ignoring the return value is almost certainly a bug.

```rust
#[must_use = "this `Result` may be an `Err` variant, which should be handled"]
pub enum Result<T, E> { Ok(T), Err(E) }

#[must_use = "if you intended to assert that this is ok, consider `.unwrap()` instead"]
pub fn is_ok(&self) -> bool { ... }
```

This turns silent bugs into compile-time warnings.

**Skill to practice:** Add `#[must_use]` to any type or function where discarding the return value indicates a logic error.

---

## Skill 8: `#[inline]` for Zero-Cost Abstractions

**Pattern:** Mark small, frequently-called methods as `#[inline]` so they compile away to nothing.

Every combinator on `Option` and `Result` is `#[inline]`. This ensures `some_val.map(|x| x + 1)` compiles to the same machine code as `match some_val { Some(x) => Some(x + 1), None => None }`.

**Skill to practice:** Use `#[inline]` on small methods in library code, especially in cross-crate boundaries where the compiler can't inline by default.

---

## Skill 9: `AsRef` for Flexible Function Signatures

**Pattern:** Accept `impl AsRef<T>` instead of a concrete type, so callers pass whatever they have.

```rust
fn open<P: AsRef<Path>>(path: P) -> File {
    let path: &Path = path.as_ref();
    // ...
}

// All of these work:
open("foo.txt");           // &str
open(String::from("f"));   // String
open(PathBuf::from("f"));  // PathBuf
```

**Skill to practice:** Use `AsRef`/`Into` bounds on function parameters to accept multiple types cheaply without generics explosion.

---

## Skill 10: Documentation as Specification

**Pattern:** Every public method follows a strict template:

1. One-line summary
2. Both variants shown (`Some`/`None`, `Ok`/`Err`)
3. `assert_eq!` for verifiable behavior (doc tests are compiled and run)
4. `should_panic` for panic paths
5. Real-world-ish examples, not abstract placeholders

```rust
/// Returns `true` if the result is [`Ok`].
///
/// # Examples
///
/// ```
/// let x: Result<i32, &str> = Ok(-3);
/// assert_eq!(x.is_ok(), true);
///
/// let x: Result<i32, &str> = Err("Some error message");
/// assert_eq!(x.is_ok(), false);
/// ```
```

**Skill to practice:** Write doc examples that double as tests. Always show both the happy path and the failure path.

---

## Quick Reference: When to Apply Each Skill

| Situation | Skill |
|-----------|-------|
| Designing a trait | #1 Skinny required, fat provided |
| Optimizing an abstraction | #2 try_fold backbone |
| Writing methods on a type | #3 Ownership-aware variants |
| Building a trait ecosystem | #4 Blanket implementations |
| Creating related types | #5 Combinator consistency |
| Building pipelines/chains | #6 Lazy adapter structs |
| Returning important values | #7 `#[must_use]` |
| Writing library methods | #8 `#[inline]` |
| Writing function parameters | #9 `AsRef`/`Into` bounds |
| Documenting public APIs | #10 Doc-as-spec |

---

## Change Log

- **2026-02-14**: Initial version — derived from analysis of `Option`, `Result`, `Iterator`, and `From`/`Into`/`AsRef` in Rust's standard library.
