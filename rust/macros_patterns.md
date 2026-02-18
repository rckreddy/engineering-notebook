# Macro Patterns in Rust

Patterns for writing declarative and procedural macros — from `macro_rules!` fundamentals through `syn`/`quote` proc macros, error handling, testing, and decision frameworks. Continues from Skills 1-124.

---

## Skill 125: macro_rules! Fundamentals

**Pattern**: `macro_rules!` defines pattern-matching macros that expand at compile time. Each arm matches a syntax pattern and produces replacement code.

```rust
// Basic syntax: match patterns, produce expansions
macro_rules! say_hello {
    () => {
        println!("Hello, world!");
    };
}

say_hello!(); // expands to println!("Hello, world!");
```

**Fragment specifiers** — what each matcher captures:

| Specifier | Matches | Example |
|-----------|---------|---------|
| `$e:expr` | Any expression | `2 + 2`, `foo.bar()` |
| `$t:ty` | A type | `i32`, `Vec<String>` |
| `$i:ident` | An identifier | `foo`, `my_var` |
| `$tt:tt` | A single token tree | any token or `(...)` / `[...]` / `{...}` group |
| `$p:pat` | A pattern | `Some(x)`, `_`, `1..=5` |
| `$path:path` | A type path | `std::collections::HashMap` |
| `$s:stmt` | A statement | `let x = 5` |
| `$b:block` | A block | `{ do_thing(); 42 }` |
| `$item:item` | An item | `fn foo() {}`, `struct Bar;` |
| `$lit:literal` | A literal | `42`, `"hello"`, `true` |

**Multiple arms** — pattern matching on syntax:
```rust
macro_rules! calculate {
    // arm 1: addition
    (add $a:expr, $b:expr) => { $a + $b };
    // arm 2: multiplication
    (mul $a:expr, $b:expr) => { $a * $b };
}

assert_eq!(calculate!(add 2, 3), 5);
assert_eq!(calculate!(mul 4, 5), 20);
```

**A `vec!`-style macro**:
```rust
macro_rules! my_vec {
    () => { Vec::new() };
    ($($elem:expr),+ $(,)?) => {{
        let mut v = Vec::new();
        $(v.push($elem);)+
        v
    }};
}

let v = my_vec![1, 2, 3];
assert_eq!(v, vec![1, 2, 3]);
```

**Common mistake**: Using `$e:expr` when you need `$t:tt`. Expressions are greedily parsed, so `$a:expr, $b:expr` in `foo!(1 + 2, 3)` works, but `$a:expr $b:expr` (no separator) fails because the parser can't tell where the first expression ends.

**Skill to practice**: Write a `macro_rules!` macro with multiple arms that constructs different types based on the keyword used (e.g., `make!(vec 1, 2, 3)` vs `make!(set 1, 2, 3)`).

---

## Skill 126: Repetition Patterns in macro_rules!

**Pattern**: Repetitions let macros accept variable-length input. The syntax is `$(pattern)separator*` (or `+` or `?`).

```rust
// Zero or more: $(...),*
macro_rules! print_all {
    ($($x:expr),*) => {
        $(println!("{}", $x);)*
    };
}
print_all!("a", "b", "c");

// One or more: $(...),+
macro_rules! sum {
    ($($x:expr),+) => {
        0 $(+ $x)+
    };
}
assert_eq!(sum!(1, 2, 3), 6);

// Zero or one: $(...)?
macro_rules! with_optional_label {
    ($name:ident $(: $label:expr)?) => {
        let $name = concat!(stringify!($name) $(, " — ", $label)?);
    };
}
```

**The trailing comma trick** — `$(,)?` accepts an optional trailing comma:
```rust
macro_rules! list {
    ($($x:expr),* $(,)?) => {
        vec![$($x),*]
    };
}
// Both work:
let a = list![1, 2, 3];
let b = list![1, 2, 3,]; // trailing comma OK
```

**Nested repetitions**:
```rust
macro_rules! matrix {
    ( $( [ $($val:expr),* ] ),* $(,)? ) => {{
        vec![ $( vec![$($val),*] ),* ]
    }};
}
let m = matrix![
    [1, 2, 3],
    [4, 5, 6],
];
assert_eq!(m[1][2], 6);
```

**HashMap literal macro**:
```rust
macro_rules! hashmap {
    ($($key:expr => $val:expr),* $(,)?) => {{
        let mut map = ::std::collections::HashMap::new();
        $(map.insert($key, $val);)*
        map
    }};
}

let scores = hashmap! {
    "Alice" => 100,
    "Bob" => 85,
    "Carol" => 92,
};
```

**Important**: The separator in repetitions (`,`, `;`, etc.) must be a single token. You cannot use multi-token separators like `=>` — but `=>` is actually a single token, so it works. Multi-character operators that are single tokens: `=>`, `->`, `::`, `..`, `..=`.

**Skill to practice**: Write a `hashmap!` macro and a `hashset!` macro that both accept trailing commas.

---

## Skill 127: macro_rules! Advanced Techniques

**Pattern**: Recursive macros, TT munchers, and push-down accumulation enable complex transformations within `macro_rules!`.

**Recursive macros** — counting elements:
```rust
macro_rules! count {
    () => { 0usize };
    ($head:tt $($tail:tt)*) => { 1usize + count!($($tail)*) };
}
assert_eq!(count!(a b c d), 4);
```

**TT muncher** — process token trees one at a time:
```rust
macro_rules! stringify_each {
    // Base case: no tokens left
    () => {};
    // Recursive case: process one token, recurse on rest
    ($head:tt $($tail:tt)*) => {
        println!("{}", stringify!($head));
        stringify_each!($($tail)*);
    };
}
stringify_each!(hello world 42);
```

**Push-down accumulation** — build output incrementally using internal rules:
```rust
macro_rules! reverse {
    // Public entry point
    ($($all:tt)*) => {
        reverse!(@acc [] $($all)*)
    };
    // Base case: accumulator has everything
    (@acc [$($acc:tt)*]) => {
        ($($acc)*)
    };
    // Recursive: move one token from input to front of accumulator
    (@acc [$($acc:tt)*] $head:tt $($tail:tt)*) => {
        reverse!(@acc [$head $($acc)*] $($tail)*)
    };
}
```

**Internal rules with `@` prefix** — convention for private macro arms:
```rust
macro_rules! my_macro {
    // Public API
    ($($input:tt)*) => {
        my_macro!(@parse [] $($input)*)
    };
    // Internal: these arms are implementation details
    (@parse [$($parsed:tt)*] done) => { /* ... */ };
    (@parse [$($parsed:tt)*] $next:tt $($rest:tt)*) => {
        my_macro!(@parse [$($parsed)* $next] $($rest)*)
    };
}
```

**Macro scoping and export**:
```rust
// Available within the crate only (defined at any point in the module tree)
macro_rules! crate_only {
    () => {};
}

// Available to downstream crates — placed at the crate root
#[macro_export]
macro_rules! public_macro {
    () => {};
}
// #[macro_export] always exports at the crate root, regardless of where defined.
// Users: use my_crate::public_macro;
```

**Common mistake**: `macro_rules!` macros defined in a module are visible only *after* the definition in textual order (not in the module tree order). Use `#[macro_use]` on the module or define macros early in `lib.rs`.

**Skill to practice**: Write a recursive TT muncher that transforms a list of `key: value` pairs into struct field definitions.

---

## Skill 128: Hygiene & Debugging macro_rules!

**Pattern**: Rust macros are *partially hygienic* — identifiers created inside the macro don't clash with identifiers outside.

**Hygiene in action**:
```rust
macro_rules! make_x {
    () => {
        let x = 42; // this "x" is in the macro's scope
    };
}

fn main() {
    let x = 10;
    make_x!();
    println!("{x}"); // prints 10 — macro's x is different!
}
```

**When hygiene breaks** — the caller must pass identifiers:
```rust
macro_rules! declare_var {
    ($name:ident, $val:expr) => {
        let $name = $val; // $name comes from the caller, so it's in caller's scope
    };
}

fn main() {
    declare_var!(y, 42);
    println!("{y}"); // works — "y" was passed by the caller
}
```

**`$crate` for paths in exported macros** — always resolves to the defining crate:
```rust
#[macro_export]
macro_rules! create_thing {
    ($val:expr) => {
        $crate::Thing::new($val)  // always points to this crate's Thing
    };
}
// Without $crate, users would need `use my_crate::Thing` in scope
```

**`cargo expand`** — see what macros generate:
```bash
cargo install cargo-expand
cargo expand                  # expand entire crate
cargo expand module::name     # expand a specific module
cargo expand --test test_name # expand test code
```

**Nightly debugging tools**:
```rust
#![feature(trace_macros, log_syntax)]

trace_macros!(true);
my_macro!(a, b, c);  // prints each expansion step to stderr
trace_macros!(false);

macro_rules! debug_me {
    ($($t:tt)*) => {
        log_syntax!($($t)*);  // prints tokens at compile time
    };
}
```

**Common error messages and fixes**:

| Error | Likely cause |
|-------|-------------|
| `no rules expected the token ...` | Input doesn't match any arm — check your patterns |
| `unexpected end of macro invocation` | Macro expects more tokens — missing argument |
| `local ambiguity` | Two arms match equally well — reorder or add distinguishing tokens |
| `recursion limit reached` | Infinite recursion or deep recursion — add `#![recursion_limit = "256"]` |
| `variable 'x' is still repeating` | Used `$x` outside repetition `$()*` when `$x` was captured inside one |

**Skill to practice**: Use `cargo expand` on a crate that uses `serde_json::json!` or `vec!` to see the expanded code. Intentionally break a macro and study the error messages.

---

## Skill 129: Procedural Macros Overview

**Pattern**: Proc macros are Rust functions that transform token streams at compile time. They live in a dedicated crate and have full Rust power.

**Three types**:

| Type | Syntax | Use for |
|------|--------|---------|
| **Derive macro** | `#[derive(MyTrait)]` | Auto-implement traits for structs/enums |
| **Attribute macro** | `#[my_attr]` on an item | Transform or wrap functions, structs, etc. |
| **Function-like** | `my_macro!(...)` | DSLs, compile-time checked strings, etc. |

**Crate setup** — proc macros must live in their own crate:
```toml
# my_macros/Cargo.toml
[package]
name = "my_macros"

[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

**The fundamental contract** — `TokenStream` in, `TokenStream` out:
```rust
use proc_macro::TokenStream;

// Derive macro
#[proc_macro_derive(MyTrait)]
pub fn my_trait_derive(input: TokenStream) -> TokenStream {
    // input = the struct/enum definition
    // output = new items (impl blocks) to add
    todo!()
}

// Attribute macro
#[proc_macro_attribute]
pub fn my_attribute(attr: TokenStream, item: TokenStream) -> TokenStream {
    // attr = arguments inside the attribute
    // item = the annotated item
    // output = replacement for the annotated item
    todo!()
}

// Function-like macro
#[proc_macro]
pub fn my_fn_macro(input: TokenStream) -> TokenStream {
    // input = everything inside the parentheses
    // output = replacement code
    todo!()
}
```

**When to use proc macros vs macro_rules!**:

| Use `macro_rules!` | Use proc macros |
|--------------------|--------------------|
| Simple substitution/repetition | Need to parse Rust syntax (structs, fields) |
| No external crate needed | Derive trait implementations |
| Pattern matching on tokens is enough | Need string processing, external files, complex logic |
| Performance matters (zero overhead) | Need good error spans/messages |

**The ecosystem**:
- **`syn`** — parses `TokenStream` into a Rust AST you can inspect
- **`quote`** — generates `TokenStream` from Rust-like template syntax
- **`proc-macro2`** — wrapper for better testing and `Span` handling

**Skill to practice**: Create a minimal proc-macro crate with `proc-macro = true` in Cargo.toml and write a trivial derive macro that compiles.

---

## Skill 130: Derive Macros

**Pattern**: Derive macros generate trait implementations automatically from struct/enum definitions.

**Complete example — deriving a `Describe` trait**:
```rust
// In the main crate:
pub trait Describe {
    fn describe(&self) -> String;
}
```

```rust
// In the proc-macro crate:
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput, Data, Fields};

#[proc_macro_derive(Describe)]
pub fn describe_derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;

    let description = match &input.data {
        Data::Struct(data) => {
            let field_count = match &data.fields {
                Fields::Named(f) => f.named.len(),
                Fields::Unnamed(f) => f.unnamed.len(),
                Fields::Unit => 0,
            };
            let field_names: Vec<String> = match &data.fields {
                Fields::Named(f) => f.named.iter()
                    .map(|f| f.ident.as_ref().unwrap().to_string())
                    .collect(),
                _ => vec![],
            };
            quote! {
                impl Describe for #name {
                    fn describe(&self) -> String {
                        format!(
                            "Struct {} with {} fields: {:?}",
                            stringify!(#name),
                            #field_count,
                            vec![#(#field_names),*],
                        )
                    }
                }
            }
        }
        Data::Enum(data) => {
            let variant_names: Vec<String> = data.variants.iter()
                .map(|v| v.ident.to_string())
                .collect();
            let variant_count = variant_names.len();
            quote! {
                impl Describe for #name {
                    fn describe(&self) -> String {
                        format!(
                            "Enum {} with {} variants: {:?}",
                            stringify!(#name),
                            #variant_count,
                            vec![#(#variant_names),*],
                        )
                    }
                }
            }
        }
        Data::Union(_) => {
            quote! { compile_error!("Describe does not support unions"); }
        }
    };

    description.into()
}
```

**Usage**:
```rust
use my_macros::Describe;

#[derive(Describe)]
struct User {
    name: String,
    age: u32,
    email: String,
}

let u = User { name: "Alice".into(), age: 30, email: "a@b.com".into() };
println!("{}", u.describe());
// "Struct User with 3 fields: ["name", "age", "email"]"
```

**Handling generics**:
```rust
#[proc_macro_derive(Describe)]
pub fn describe_derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();

    let expanded = quote! {
        impl #impl_generics Describe for #name #ty_generics #where_clause {
            fn describe(&self) -> String {
                format!("Instance of {}", stringify!(#name))
            }
        }
    };
    expanded.into()
}
```

**Helper attributes** — let the derive macro read extra annotations:
```rust
#[proc_macro_derive(Describe, attributes(describe))]
pub fn describe_derive(input: TokenStream) -> TokenStream {
    // Users can now write:
    // #[derive(Describe)]
    // struct Foo {
    //     #[describe(skip)]
    //     internal_field: u32,
    // }
    todo!()
}
```

**Skill to practice**: Write a derive macro for a `Builder` pattern — generating a `UserBuilder` from a `User` struct with setter methods and a `build()` method.

---

## Skill 131: Attribute Macros

**Pattern**: Attribute macros receive the annotated item and can transform, wrap, or replace it entirely.

**Example: `#[log_calls]` that wraps a function with tracing**:
```rust
// Proc-macro crate:
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemFn};

#[proc_macro_attribute]
pub fn log_calls(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let input_fn = parse_macro_input!(item as ItemFn);
    let fn_name = &input_fn.sig.ident;
    let fn_name_str = fn_name.to_string();
    let vis = &input_fn.vis;
    let sig = &input_fn.sig;
    let body = &input_fn.block;

    let expanded = quote! {
        #vis #sig {
            println!("[ENTER] {}", #fn_name_str);
            let __result = (|| #body)();
            println!("[EXIT]  {}", #fn_name_str);
            __result
        }
    };
    expanded.into()
}
```

**Usage**:
```rust
#[log_calls]
fn process_data(input: &str) -> usize {
    input.len()
}
// Calling process_data("hello") prints:
// [ENTER] process_data
// [EXIT]  process_data
```

**Attribute with arguments**:
```rust
// #[route("GET", "/users")]
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
    let input_fn = parse_macro_input!(item as ItemFn);
    let fn_name = &input_fn.sig.ident;

    // Parse attr manually or with a custom struct
    let attr_str = attr.to_string();
    // attr_str would be: "GET", "/users"

    let expanded = quote! {
        #input_fn

        // Register the route at startup
        inventory::submit! {
            Route {
                method: #attr_str,
                handler: #fn_name,
            }
        }
    };
    expanded.into()
}
```

**Parsing structured attribute arguments with syn**:
```rust
use syn::parse::{Parse, ParseStream};
use syn::{LitStr, Token};

struct RouteArgs {
    method: LitStr,
    path: LitStr,
}

impl Parse for RouteArgs {
    fn parse(input: ParseStream) -> syn::Result<Self> {
        let method: LitStr = input.parse()?;
        input.parse::<Token![,]>()?;
        let path: LitStr = input.parse()?;
        Ok(RouteArgs { method, path })
    }
}

#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
    let args = parse_macro_input!(attr as RouteArgs);
    let input_fn = parse_macro_input!(item as ItemFn);
    // Use args.method.value(), args.path.value()
    // ...
    todo!()
}
```

**Key difference from derive**: Attribute macros *replace* the annotated item. If you want to keep the original, you must include it in your output (as shown above with `#input_fn`).

**Skill to practice**: Write a `#[timed]` attribute macro that measures and prints the execution time of any function it annotates.

---

## Skill 132: Function-like Procedural Macros

**Pattern**: Function-like proc macros look like `my_macro!(...)` but have the full power of proc macros — arbitrary parsing, external resources, compile-time validation.

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, LitStr};

/// Compile-time validated SQL: sql!("SELECT * FROM users WHERE id = ?")
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
    let query = parse_macro_input!(input as LitStr);
    let query_str = query.value();

    // Validate SQL syntax at compile time
    if !query_str.to_uppercase().starts_with("SELECT")
        && !query_str.to_uppercase().starts_with("INSERT")
        && !query_str.to_uppercase().starts_with("UPDATE")
        && !query_str.to_uppercase().starts_with("DELETE")
    {
        return syn::Error::new_spanned(&query, "SQL must start with SELECT, INSERT, UPDATE, or DELETE")
            .to_compile_error()
            .into();
    }

    let expanded = quote! {
        Query::new(#query_str)
    };
    expanded.into()
}
```

**When to use function-like proc macros**:

| Use case | Example |
|----------|---------|
| DSLs (domain-specific languages) | `html! { <div class="foo">...</div> }` |
| Compile-time validation | `regex!("^[a-z]+$")` — checked at compile time |
| Embedding external resources | `include_graphql!("query.graphql")` |
| Code generation from data | `config! { setting: type = default; }` |

**Comparison with `macro_rules!` function-like macros**:

| Feature | `macro_rules!` | Proc macro |
|---------|----------------|------------|
| Crate setup | None — inline | Separate `proc-macro = true` crate |
| Parsing power | Pattern matching only | Full Rust code, syn, custom parsers |
| Error quality | Often cryptic | Precise spans with `syn::Error` |
| Dependencies | None | Can use any crate |
| Compile time | Near-zero | Compiles the proc-macro crate first |

**Example: compile-time regex validation**:
```rust
#[proc_macro]
pub fn checked_regex(input: TokenStream) -> TokenStream {
    let lit = parse_macro_input!(input as LitStr);
    let pattern = lit.value();

    // Validate the regex at compile time
    if let Err(e) = regex::Regex::new(&pattern) {
        return syn::Error::new_spanned(
            &lit,
            format!("invalid regex: {e}"),
        )
        .to_compile_error()
        .into();
    }

    quote! {
        ::regex::Regex::new(#pattern).unwrap()
    }
    .into()
}
```

**Skill to practice**: Write a function-like proc macro that takes a string literal and validates it as a URL at compile time (check for scheme, host, etc.).

---

## Skill 133: syn — Parsing Rust Syntax

**Pattern**: `syn` is the standard crate for parsing `TokenStream` into a structured AST. It turns raw tokens into types you can pattern-match on.

**`parse_macro_input!`** — the entry point:
```rust
use syn::{parse_macro_input, DeriveInput, ItemFn, ItemStruct, Expr};

// In a derive macro:
let input = parse_macro_input!(input as DeriveInput);

// In an attribute macro on a function:
let func = parse_macro_input!(item as ItemFn);

// For expressions:
let expr = parse_macro_input!(input as Expr);
```

**Key syn types**:

| Type | Represents |
|------|-----------|
| `DeriveInput` | A struct, enum, or union (for derive macros) |
| `ItemFn` | A function definition |
| `ItemStruct` | A struct definition |
| `ItemEnum` | An enum definition |
| `Expr` | Any expression |
| `Type` | A type annotation |
| `Fields` | Named, unnamed (tuple), or unit fields |
| `Generics` | Generic parameters and where clauses |
| `Attribute` | An `#[attr]` annotation |

**Custom parsing** — implement `Parse` for your own syntax:
```rust
use syn::parse::{Parse, ParseStream};
use syn::{Ident, Token, Expr, Result};

// Syntax: key_value!(name = "Alice", age = 30)
struct KeyValue {
    key: Ident,
    value: Expr,
}

impl Parse for KeyValue {
    fn parse(input: ParseStream) -> Result<Self> {
        let key: Ident = input.parse()?;
        input.parse::<Token![=]>()?;
        let value: Expr = input.parse()?;
        Ok(KeyValue { key, value })
    }
}

// Parse multiple key-value pairs
struct KeyValueList {
    pairs: syn::punctuated::Punctuated<KeyValue, Token![,]>,
}

impl Parse for KeyValueList {
    fn parse(input: ParseStream) -> Result<Self> {
        Ok(KeyValueList {
            pairs: input.parse_terminated(KeyValue::parse, Token![,])?,
        })
    }
}
```

**Walking the AST with `syn::visit`**:
```rust
use syn::visit::{self, Visit};

struct FnCounter { count: usize }

impl<'ast> Visit<'ast> for FnCounter {
    fn visit_item_fn(&mut self, node: &'ast syn::ItemFn) {
        self.count += 1;
        visit::visit_item_fn(self, node); // recurse into nested items
    }
}
```

**Transforming with `syn::fold`** — similar to visit but produces a modified AST:
```rust
use syn::fold::{self, Fold};

struct AddPub;

impl Fold for AddPub {
    fn fold_visibility(&mut self, _vis: syn::Visibility) -> syn::Visibility {
        syn::parse_quote!(pub) // make everything pub
    }
}
```

**Error reporting** — point to the right source location:
```rust
fn validate_struct(input: &DeriveInput) -> syn::Result<()> {
    match &input.data {
        syn::Data::Struct(_) => Ok(()),
        _ => Err(syn::Error::new_spanned(
            &input.ident,
            "Describe can only be derived for structs",
        )),
    }
}
```

**Skill to practice**: Implement a custom `Parse` type that parses `name: Type => default_value` syntax for a config DSL.

---

## Skill 134: quote — Generating Code

**Pattern**: `quote!` lets you write Rust-like syntax that becomes a `TokenStream`. It is the complement to `syn` — syn parses, quote generates.

**Basic usage**:
```rust
use quote::quote;

let name = format_ident!("MyStruct");
let expanded = quote! {
    struct #name {
        value: i32,
    }
};
// Produces: struct MyStruct { value: i32, }
```

**Interpolation with `#var`**:
```rust
let struct_name = &input.ident;
let field_name = format_ident!("count");
let field_type = quote!(usize);

let expanded = quote! {
    impl #struct_name {
        pub fn #field_name(&self) -> #field_type {
            self.items.len()
        }
    }
};
```

**Repetition with `#(#iter)*`**:
```rust
let field_names: Vec<&Ident> = fields.iter()
    .map(|f| f.ident.as_ref().unwrap())
    .collect();
let field_types: Vec<&Type> = fields.iter()
    .map(|f| &f.ty)
    .collect();

let expanded = quote! {
    impl #name {
        pub fn field_names() -> &'static [&'static str] {
            &[#(stringify!(#field_names)),*]
        }

        pub fn new(#(#field_names: #field_types),*) -> Self {
            Self { #(#field_names),* }
        }
    }
};
```

**`quote_spanned!`** — attach error spans for better diagnostics:
```rust
use quote::quote_spanned;
use syn::spanned::Spanned;

for field in &fields {
    let ty = &field.ty;
    let span = field.span();
    // If this bound fails, the error points to the field, not the derive
    let check = quote_spanned! { span =>
        const _: fn() = || {
            fn assert_clone<T: Clone>() {}
            assert_clone::<#ty>();
        };
    };
}
```

**`format_ident!`** — create identifiers dynamically:
```rust
use quote::format_ident;

let name = format_ident!("get_{}", field_name);       // get_name
let builder = format_ident!("{}Builder", struct_name); // UserBuilder
let test_fn = format_ident!("test_{}", method_name);   // test_process
```

**Nested quoting** — quoting inside quoting:
```rust
let methods: Vec<proc_macro2::TokenStream> = fields.iter().map(|f| {
    let name = &f.ident;
    let ty = &f.ty;
    quote! {
        pub fn #name(&self) -> &#ty {
            &self.#name
        }
    }
}).collect();

let expanded = quote! {
    impl #struct_name {
        #(#methods)*
    }
};
```

**Skill to practice**: Generate an `impl` block with getter methods for every field in a struct using `quote!` and repetitions.

---

## Skill 135: Error Handling in Proc Macros

**Pattern**: Good proc macros give precise, helpful compile errors that point to the correct source location. Bad ones emit "proc macro panicked" with no context.

**`syn::Error::new_spanned`** — the primary tool:
```rust
use syn::Error;

fn validate(input: &DeriveInput) -> syn::Result<TokenStream2> {
    if let Data::Union(_) = &input.data {
        return Err(Error::new_spanned(
            &input.ident,
            "MyTrait cannot be derived for unions",
        ));
    }

    for field in get_fields(input) {
        if field.ident.is_none() {
            return Err(Error::new_spanned(
                field,
                "MyTrait requires named fields (not tuple structs)",
            ));
        }
    }

    Ok(generate_impl(input))
}

#[proc_macro_derive(MyTrait)]
pub fn derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    match validate(&input) {
        Ok(tokens) => tokens.into(),
        Err(err) => err.to_compile_error().into(),
    }
}
```

**Collecting multiple errors** — don't stop at the first one:
```rust
fn validate_all_fields(fields: &Fields) -> syn::Result<()> {
    let mut errors: Option<syn::Error> = None;

    for field in fields.iter() {
        if let Err(e) = validate_field(field) {
            match &mut errors {
                Some(existing) => existing.combine(e), // append error
                None => errors = Some(e),
            }
        }
    }

    match errors {
        Some(e) => Err(e),
        None => Ok(()),
    }
}
```

**Testing with `trybuild`** — compile-fail tests:
```rust
// tests/compile_tests.rs
#[test]
fn compile_fail_tests() {
    let t = trybuild::TestCases::new();
    t.pass("tests/ui/pass_*.rs");        // should compile
    t.compile_fail("tests/ui/fail_*.rs"); // should fail with expected errors
}
```

```rust
// tests/ui/fail_union.rs
use my_macros::MyTrait;

#[derive(MyTrait)]
union Bad {       // should fail: "MyTrait cannot be derived for unions"
    x: i32,
    y: f32,
}

fn main() {}
```

```text
// tests/ui/fail_union.stderr (expected error output)
error: MyTrait cannot be derived for unions
 --> tests/ui/fail_union.rs:4:7
  |
4 | union Bad {
  |       ^^^
```

**`proc-macro-error` crate** — convenient diagnostics:
```rust
use proc_macro_error::{proc_macro_error, abort, emit_error};

#[proc_macro_derive(MyTrait)]
#[proc_macro_error]
pub fn derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    // Fatal error — stops immediately
    if something_very_wrong(&input) {
        abort!(input.ident, "completely unsupported type");
    }

    // Non-fatal — collects and continues
    for field in get_fields(&input) {
        if is_problematic(field) {
            emit_error!(field, "this field will be skipped: {}", reason);
        }
    }

    // All emitted errors are reported at the end
    generate_impl(&input).into()
}
```

**Never `panic!()` in a proc macro** — it produces an unhelpful "proc macro panicked" message. Always use `syn::Error` or `proc_macro_error` for user-facing diagnostics.

**Skill to practice**: Write a derive macro that validates its input (e.g., "must be a struct with named fields") and uses `trybuild` to test both the success and failure cases.

---

## Skill 136: Common Macro Patterns from the Ecosystem

**Pattern**: Study widely-used macros to learn idiomatic patterns.

**`cfg_if!`** — conditional compilation blocks:
```rust
// From the cfg-if crate
cfg_if::cfg_if! {
    if #[cfg(unix)] {
        fn platform_specific() { /* unix impl */ }
    } else if #[cfg(windows)] {
        fn platform_specific() { /* windows impl */ }
    } else {
        fn platform_specific() { /* fallback */ }
    }
}
```

**`bitflags!`** — flag enums with bitwise operations:
```rust
use bitflags::bitflags;

bitflags! {
    #[derive(Debug, Clone, Copy, PartialEq, Eq)]
    struct Permissions: u32 {
        const READ    = 0b0001;
        const WRITE   = 0b0010;
        const EXECUTE = 0b0100;
        const ALL     = Self::READ.bits() | Self::WRITE.bits() | Self::EXECUTE.bits();
    }
}

let perms = Permissions::READ | Permissions::WRITE;
assert!(perms.contains(Permissions::READ));
assert!(!perms.contains(Permissions::EXECUTE));
```

**`LazyLock` / `thread_local!`**:
```rust
use std::sync::LazyLock;
use std::collections::HashMap;

// Initialized on first access, thread-safe
static CONFIG: LazyLock<HashMap<String, String>> = LazyLock::new(|| {
    let mut m = HashMap::new();
    m.insert("key".into(), "value".into());
    m
});

// Thread-local storage
thread_local! {
    static BUFFER: RefCell<Vec<u8>> = RefCell::new(Vec::with_capacity(1024));
}
BUFFER.with(|buf| {
    buf.borrow_mut().push(42);
});
```

**`serde_json::json!`** — building data literals:
```rust
use serde_json::json;

let payload = json!({
    "name": "Alice",
    "age": 30,
    "scores": [100, 95, 88],
    "active": true,
    "address": null,
});
// Type: serde_json::Value
```

**Builder derive pattern** (from `derive_builder` crate):
```rust
use derive_builder::Builder;

#[derive(Builder)]
#[builder(setter(into))]
struct Server {
    host: String,
    port: u16,
    #[builder(default = "4")]
    max_connections: u32,
}

let server = ServerBuilder::default()
    .host("localhost")
    .port(8080)
    .build()
    .unwrap();
```

**`pin_project!`** — safe pin projections:
```rust
use pin_project::pin_project;

#[pin_project]
struct MyFuture<F> {
    #[pin]
    inner: F,       // structurally pinned
    started: bool,  // not pinned
}
// Generates safe projection methods:
// let projected = self.project();
// projected.inner: Pin<&mut F>
// projected.started: &mut bool
```

**Skill to practice**: Use `bitflags!` to model file permissions, then use `serde_json::json!` to serialize them into a JSON API response.

---

## Skill 137: Macro Best Practices & Anti-Patterns

**Pattern**: Macros are powerful but can make code harder to read, debug, and maintain. Use them judiciously.

**When to use macros vs alternatives**:

| Need | Best tool | Why |
|------|-----------|-----|
| Reduce boilerplate for trait impls | `derive` macro | Standard pattern, well-understood |
| Variadic arguments | `macro_rules!` | Rust functions can't be variadic |
| Compile-time validation | Proc macro | Full Rust logic at compile time |
| Conditional method impls | Generics + trait bounds | More discoverable, better errors |
| Default method behavior | Trait default methods | No macro needed |
| Constant computation | `const fn` | Evaluated at compile time, no macro needed |

**Best practices**:

1. **Keep macros small** — extract logic into functions:
```rust
// BAD: all logic in the macro
macro_rules! parse_and_validate {
    ($input:expr) => {{
        let parsed = $input.parse::<i32>().unwrap();
        if parsed < 0 { panic!("negative"); }
        if parsed > 100 { panic!("too large"); }
        parsed
    }};
}

// GOOD: macro calls a function
fn validate_range(input: &str) -> i32 {
    let parsed: i32 = input.parse().expect("not a number");
    assert!(parsed >= 0 && parsed <= 100, "out of range: {parsed}");
    parsed
}
macro_rules! parse_and_validate {
    ($input:expr) => { validate_range($input) };
}
```

2. **Don't hide control flow**:
```rust
// BAD: surprising return hidden in a macro
macro_rules! try_or_return {
    ($e:expr) => {
        match $e {
            Ok(v) => v,
            Err(_) => return,  // hidden return!
        }
    };
}

// GOOD: use the ? operator or be explicit
let value = some_operation()?;
```

3. **Ensure macros work in all positions**:
```rust
// Wrap in braces to work as an expression
macro_rules! max {
    ($a:expr, $b:expr) => {{
        // double braces = block expression
        let a = $a;
        let b = $b;
        if a > b { a } else { b }
    }};
}
// Works as: let x = max!(1, 2);
// Works as: println!("{}", max!(1, 2));
```

4. **Evaluate arguments once** — avoid double evaluation:
```rust
// BAD: $x evaluated twice (side effects happen twice)
macro_rules! double_bad {
    ($x:expr) => { $x + $x };
}
double_bad!(expensive_function()); // called twice!

// GOOD: bind to a local variable first
macro_rules! double_good {
    ($x:expr) => {{
        let __x = $x;
        __x + __x
    }};
}
```

5. **Document macros with examples**:
```rust
/// Creates a `HashMap` from key-value pairs.
///
/// # Examples
///
/// ```
/// let map = my_crate::hashmap! {
///     "name" => "Alice",
///     "role" => "admin",
/// };
/// assert_eq!(map["name"], "Alice");
/// ```
#[macro_export]
macro_rules! hashmap { /* ... */ }
```

**Skill to practice**: Audit a crate's macros against these rules. Refactor any that hide control flow or evaluate arguments multiple times.

---

## Skill 138: Testing Macros

**Pattern**: Macros need tests too — both that they expand correctly and that errors are helpful.

**Testing `macro_rules!` — regular `#[test]` functions**:
```rust
macro_rules! hashmap {
    ($($k:expr => $v:expr),* $(,)?) => {{
        let mut m = std::collections::HashMap::new();
        $(m.insert($k, $v);)*
        m
    }};
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn empty_hashmap() {
        let m: std::collections::HashMap<String, i32> = hashmap! {};
        assert!(m.is_empty());
    }

    #[test]
    fn hashmap_with_entries() {
        let m = hashmap! { "a" => 1, "b" => 2 };
        assert_eq!(m["a"], 1);
        assert_eq!(m.len(), 2);
    }

    #[test]
    fn trailing_comma() {
        let m = hashmap! { "a" => 1, };
        assert_eq!(m.len(), 1);
    }
}
```

**Testing proc macros with `trybuild`**:
```toml
# Cargo.toml
[dev-dependencies]
trybuild = "1"
```

```rust
// tests/derive_tests.rs
#[test]
fn derive_tests() {
    let t = trybuild::TestCases::new();
    // Files that should compile successfully
    t.pass("tests/ui/pass/*.rs");
    // Files that should fail with specific error messages
    t.compile_fail("tests/ui/fail/*.rs");
}
```

```rust
// tests/ui/pass/basic_struct.rs
use my_macros::MyDerive;

#[derive(MyDerive)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p = Point { x: 1.0, y: 2.0 };
    // Use the derived functionality
    println!("{}", p.describe());
}
```

**Snapshot testing with `macrotest`**:
```toml
[dev-dependencies]
macrotest = "1"
```

```rust
#[test]
fn expand_macros() {
    macrotest::expand("tests/expand/*.rs");
    // First run: creates .expanded.rs files
    // Subsequent runs: compares against saved expansion
}
```

**Snapshot testing with `insta` and `cargo expand`**:
```rust
#[test]
fn test_macro_expansion() {
    // Capture the expansion as a string, snapshot it
    let expanded = quote! {
        // simulate calling your macro's internal logic
    };
    insta::assert_snapshot!(expanded.to_string());
}
```

**Testing error messages** — the user experience matters:
```rust
// tests/ui/fail/no_unions.rs
use my_macros::MyDerive;

#[derive(MyDerive)]
union Bad {
    x: i32,
    y: f32,
}

fn main() {}
```

```text
// tests/ui/fail/no_unions.stderr
error: MyDerive cannot be derived for unions
 --> tests/ui/fail/no_unions.rs:4:7
  |
4 | union Bad {
  |       ^^^
```

**`cargo expand` as a development tool**:
```bash
# See what your macro generates
cargo expand --lib            # expand all macros in lib.rs
cargo expand --test my_test   # expand macros in a test file
cargo expand my_module        # expand a specific module
```

**Skill to practice**: Set up `trybuild` tests for a derive macro — at least two passing cases and two compile-fail cases with verified error messages.

---

## Skill 139: Writing a Complete Derive Macro (Walkthrough)

**Pattern**: End-to-end walkthrough of a derive macro for a `Describe` trait, from workspace setup to handling generics.

**Step 1: Workspace structure**:
```
my_project/
  Cargo.toml          # workspace root
  my_describe/
    Cargo.toml         # main crate (re-exports the trait + derive)
    src/lib.rs
  my_describe_derive/
    Cargo.toml         # proc-macro crate
    src/lib.rs
```

```toml
# Cargo.toml (workspace root)
[workspace]
members = ["my_describe", "my_describe_derive"]
```

```toml
# my_describe_derive/Cargo.toml
[package]
name = "my_describe_derive"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

```toml
# my_describe/Cargo.toml
[package]
name = "my_describe"
version = "0.1.0"
edition = "2021"

[dependencies]
my_describe_derive = { path = "../my_describe_derive" }
```

**Step 2: Define the trait**:
```rust
// my_describe/src/lib.rs
pub use my_describe_derive::Describe;

pub trait Describe {
    fn describe(&self) -> String;
}
```

**Step 3: Implement the derive macro**:
```rust
// my_describe_derive/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput, Data, Fields, Ident};

#[proc_macro_derive(Describe)]
pub fn describe_derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    match impl_describe(&input) {
        Ok(tokens) => tokens.into(),
        Err(err) => err.to_compile_error().into(),
    }
}

fn impl_describe(input: &DeriveInput) -> syn::Result<proc_macro2::TokenStream> {
    let name = &input.ident;
    let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();

    let body = match &input.data {
        Data::Struct(data) => describe_struct(name, &data.fields)?,
        Data::Enum(data) => describe_enum(name, data)?,
        Data::Union(_) => {
            return Err(syn::Error::new_spanned(name, "Describe does not support unions"));
        }
    };

    Ok(quote! {
        impl #impl_generics my_describe::Describe for #name #ty_generics #where_clause {
            fn describe(&self) -> String {
                #body
            }
        }
    })
}

fn describe_struct(name: &Ident, fields: &Fields) -> syn::Result<proc_macro2::TokenStream> {
    let name_str = name.to_string();
    match fields {
        Fields::Named(f) => {
            let field_strs: Vec<String> = f.named.iter()
                .map(|f| f.ident.as_ref().unwrap().to_string())
                .collect();
            let count = field_strs.len();
            Ok(quote! {
                format!("{} {{ {} }}", #name_str,
                    [#(#field_strs),*].iter()
                        .map(|s| s.to_string())
                        .collect::<Vec<_>>()
                        .join(", "))
            })
        }
        Fields::Unnamed(f) => {
            let count = f.unnamed.len();
            Ok(quote! {
                format!("{}({} fields)", #name_str, #count)
            })
        }
        Fields::Unit => {
            Ok(quote! { format!("{}", #name_str) })
        }
    }
}

fn describe_enum(name: &Ident, data: &syn::DataEnum) -> syn::Result<proc_macro2::TokenStream> {
    let name_str = name.to_string();
    let arms = data.variants.iter().map(|v| {
        let variant = &v.ident;
        let variant_str = variant.to_string();
        match &v.fields {
            Fields::Named(_) => quote! {
                Self::#variant { .. } => format!("{}::{} {{ ... }}", #name_str, #variant_str)
            },
            Fields::Unnamed(_) => quote! {
                Self::#variant(..) => format!("{}::{}(..)", #name_str, #variant_str)
            },
            Fields::Unit => quote! {
                Self::#variant => format!("{}::{}", #name_str, #variant_str)
            },
        }
    });

    Ok(quote! {
        match self {
            #(#arms,)*
        }
    })
}
```

**Step 4: Use it**:
```rust
use my_describe::Describe;

#[derive(Describe)]
struct Point {
    x: f64,
    y: f64,
}

#[derive(Describe)]
enum Shape {
    Circle { radius: f64 },
    Rectangle(f64, f64),
    Point,
}

fn main() {
    let p = Point { x: 1.0, y: 2.0 };
    println!("{}", p.describe()); // "Point { x, y }"

    let s = Shape::Circle { radius: 5.0 };
    println!("{}", s.describe()); // "Shape::Circle { ... }"
}
```

**Step 5: Handle generics** — the `split_for_impl()` method does the heavy lifting:
```rust
// This works for:
#[derive(Describe)]
struct Wrapper<T> {
    inner: T,
}
// Generates: impl<T> Describe for Wrapper<T> { ... }

// For bounded generics:
#[derive(Describe)]
struct Pair<T: Clone> {
    a: T,
    b: T,
}
// Generates: impl<T: Clone> Describe for Pair<T> where T: Clone { ... }
```

**Skill to practice**: Follow this walkthrough end-to-end. Add a `#[describe(skip)]` helper attribute to exclude fields from the description.

---

## Skill 140: Macro Decision Matrix

**Pattern**: Choose the right macro type (or avoid macros entirely) based on your requirements.

**Decision flowchart**:

```
Do you need code generation at all?
├── No: Can generics/traits/const fn solve it?
│   └── Yes → Use generics, trait defaults, or const fn. No macro needed.
│
├── Yes: What kind of code do you need to generate?
│   ├── Simple repetition/substitution → macro_rules!
│   ├── Trait impl from struct/enum shape → derive macro
│   ├── Transform/wrap an item → attribute macro
│   └── Custom syntax / DSL → function-like proc macro
```

**Detailed matrix**:

| Requirement | macro_rules! | Derive | Attribute | Fn-like proc | No macro |
|-------------|:---:|:---:|:---:|:---:|:---:|
| Variadic arguments | x | | | x | |
| Trait impl from type shape | | x | | | |
| Wrap/modify functions | | | x | | |
| DSL / custom syntax | | | | x | |
| Compile-time validation | | x | x | x | |
| No extra crate dependency | x | | | | x |
| Best error messages | | x | x | x | |
| Fastest compile time | x | | | | x |
| Access to field names/types | | x | x | x | |
| Works in expression position | x | | | x | |

**When NOT to use macros**:

| Instead of | Use |
|-----------|-----|
| Macro that wraps N types with same impl | Blanket impl or generic function |
| Macro for default values | `Default` trait or builder pattern |
| Macro for constant computation | `const fn` |
| Macro for conditional compilation | `#[cfg(...)]` attributes |
| Macro that just calls a function | Just call the function |
| Macro for "convenience" | A well-designed API |

**Performance: compile-time cost**:
```
No macro        → 0ms
macro_rules!    → ~0ms (token substitution only)
Simple derive   → 50-200ms (must compile proc-macro crate + syn + quote)
Complex derive  → 200ms-2s (depends on syn features used)
```

The first proc macro in a dependency tree pays the cost of compiling `syn` (~5-10s). Subsequent proc macros share the already-compiled `syn`. Use `syn = { version = "2", features = ["derive"] }` instead of `features = ["full"]` to reduce compile times when you only need derive support.

**Real-world decision examples**:

| Scenario | Choice | Why |
|----------|--------|-----|
| Custom `Debug` output | `impl fmt::Debug` manually | One type, no repetition |
| `Debug` for 20 types | `#[derive(Debug)]` | Built-in derive |
| HashMap literal syntax | `macro_rules!` | Simple repetition |
| Auto-generate builder | Derive proc macro | Needs field introspection |
| `#[test]` alternative with setup | Attribute proc macro | Wraps function body |
| SQL validation | Function-like proc macro | Custom parsing needed |
| Add logging to all methods | Attribute proc macro | Transforms functions |
| sum of N numbers | `macro_rules!` or just a function | `fn sum(vals: &[i32])` is simpler |

**Skill to practice**: For your current project, list every macro you use. For each, ask: "Could this be a function, a generic, or a trait instead?" Replace any macro that doesn't need to be a macro.

---

## Quick Reference

| Skill | Pattern | One-Liner |
|-------|---------|-----------|
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

---

## Change Log

| Date | Change |
|------|--------|
| 2026-02-17 | Initial version — Skills 125-140 from macro patterns analysis |
