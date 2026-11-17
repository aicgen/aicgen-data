# Rust Fundamentals

## Toolchain And Edition

- New code uses `edition = "2024"`. Set `rust-version` (MSRV) in `Cargo.toml` and pin the toolchain for the repo.
- Libraries state an MSRV and test it in CI. Applications track the latest stable.

```toml
# Cargo.toml
[package]
name = "myapp"
version = "0.1.0"
edition = "2024"
rust-version = "1.88"   # let chains need 1.88 and edition 2024
```

```toml
# rust-toolchain.toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy"]
```

## Project Layout

Keep `main.rs` thin and put logic in a library crate so it can be tested and reused.

```
myapp/
├── Cargo.toml
├── src/
│   ├── main.rs        # parse args, build config, call lib::run
│   ├── lib.rs         # public API and module tree
│   ├── config.rs
│   └── user/
│       ├── mod.rs     # or user.rs next to a user/ directory
│       └── repo.rs
├── tests/             # integration tests (public API only)
├── benches/
└── examples/
```

- Prefer `user.rs` + `user/` (or `mod.rs`) consistently; do not mix styles in one crate.
- Split into a Cargo workspace when crates have different dependencies, release cadence or compile-time cost.
- Keep the public surface small: default to private, then `pub(crate)`, then `pub`.

## Ownership And Borrowing

Accept the most general borrowed type; take ownership only when you store the value.

```rust
use std::path::Path;

// ✅ &str accepts &String, &str and string literals
fn greet(name: &str) -> String {
    format!("Hello, {name}!")
}

// ✅ &[T] accepts &Vec<T>, arrays and slices
fn total(values: &[u64]) -> u64 {
    values.iter().sum()
}

// ✅ &Path accepts &PathBuf, &str and &OsStr via AsRef
fn extension(path: impl AsRef<Path>) -> Option<String> {
    path.as_ref().extension()?.to_str().map(str::to_owned)
}

// ✅ Take ownership when the function stores the value
struct User {
    name: String,
}

impl User {
    fn new(name: impl Into<String>) -> Self {
        Self { name: name.into() }
    }
}

// ❌ Avoid: forces callers to own a String or Vec
// fn greet(name: &String) -> String
// fn total(values: &Vec<u64>) -> u64

fn main() {
    println!("{}", greet("Ada"));
    println!("{}", total(&[1, 2, 3]));
    println!("{:?}", extension("notes.md"));
    println!("{}", User::new("Ada").name);
}
```

- Do not `.clone()` to silence the borrow checker. First restructure: shorten borrows, split structs, return owned values, or use indices.
- Use `Cow<'_, str>` when a function usually returns its input unchanged but sometimes needs to allocate.
- Use `Arc` for shared ownership across threads and `Rc` only in single-threaded code. Prefer passing references over wrapping everything in `Arc<Mutex<_>>`.

## Iterators

Prefer iterator chains for transformations and plain loops when control flow is complex. Do not collect into a `Vec` just to iterate again.

```rust
#[derive(Debug)]
struct Order {
    id: u32,
    total_cents: u64,
    paid: bool,
}

fn unpaid_total(orders: &[Order]) -> u64 {
    orders
        .iter()
        .filter(|order| !order.paid)
        .map(|order| order.total_cents)
        .sum()
}

// Collect Result items without an intermediate Vec of Results
fn parse_all(inputs: &[&str]) -> Result<Vec<u32>, std::num::ParseIntError> {
    inputs.iter().map(|s| s.parse()).collect()
}

fn main() {
    let orders = [
        Order { id: 1, total_cents: 500, paid: true },
        Order { id: 2, total_cents: 700, paid: false },
    ];
    assert_eq!(unpaid_total(&orders), 700);
    assert_eq!(parse_all(&["1", "2"]), Ok(vec![1, 2]));
    assert!(parse_all(&["1", "x"]).is_err());
}
```

## Pattern Matching

```rust
#[derive(Debug)]
enum Shape {
    Circle { radius: f64 },
    Rect { w: f64, h: f64 },
}

// ✅ Exhaustive match: adding a variant becomes a compile error here
fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
        Shape::Rect { w, h } => w * h,
    }
}

// ✅ let-else keeps the happy path unindented
fn first_word(text: &str) -> &str {
    let Some(word) = text.split_whitespace().next() else {
        return "";
    };
    word
}

// ✅ Let chains (stable since Rust 1.88, edition 2024 only)
fn describe(input: Option<&str>) -> String {
    if let Some(text) = input
        && let Ok(number) = text.trim().parse::<i64>()
        && number > 0
    {
        format!("positive: {number}")
    } else {
        "not a positive number".to_owned()
    }
}

fn main() {
    assert!(area(&Shape::Rect { w: 2.0, h: 3.0 }) > 5.9);
    assert_eq!(first_word("hello world"), "hello");
    assert_eq!(describe(Some(" 42 ")), "positive: 42");
    assert!(matches!(Shape::Circle { radius: 1.0 }, Shape::Circle { .. }));
}
```

- Avoid `_ =>` wildcards on your own enums; they hide new variants. Use them for foreign or `#[non_exhaustive]` enums.
- Use `matches!` for boolean checks and `if let` / `let else` for a single pattern.

## Naming (RFC 430 and API Guidelines)

| Item | Convention | Example |
|------|-----------|---------|
| Crates, modules, functions, variables | `snake_case` | `parse_config` |
| Types, traits, enum variants | `UpperCamelCase` | `UserRepository` |
| Constants and statics | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES` |
| Lifetimes | short lowercase | `'a`, `'de` |
| Conversions | `as_` (cheap borrow), `to_` (expensive), `into_` (consumes) | `as_str`, `to_vec`, `into_inner` |
| Getters | field name, no `get_` prefix | `user.name()` |
| Constructors | `new`, `with_*`, `from_*` | `Config::with_timeout(d)` |

## Data Types

```rust
use std::time::Duration;

// ✅ Derive what you need; Debug on every public type
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct UserId(pub u64);

// ✅ Config-style structs: Default + struct update syntax
#[derive(Debug, Clone)]
pub struct Settings {
    pub retries: u32,
    pub timeout: Duration,
}

impl Default for Settings {
    fn default() -> Self {
        Self { retries: 3, timeout: Duration::from_secs(30) }
    }
}

// ✅ #[non_exhaustive] lets a library add variants and fields later
#[derive(Debug)]
#[non_exhaustive]
pub enum Status {
    Active,
    Suspended,
}

// ✅ Checked conversions instead of lossy `as` casts
fn to_u8(value: i64) -> Option<u8> {
    u8::try_from(value).ok()
}

fn main() {
    let settings = Settings { retries: 5, ..Settings::default() };
    assert_eq!(settings.timeout, Duration::from_secs(30));
    assert_eq!(to_u8(300), None);
    assert_eq!(UserId(1), UserId(1));
    let _ = Status::Active;
}
```

- Mark functions whose result must not be ignored with `#[must_use]` (builders, guards, `Result`-like values).
- Use `u64`/`usize`/`i64` deliberately. `as` truncates silently; prefer `try_from`/`try_into` unless wrapping is the intent.
- Money is integer minor units or a decimal type, never `f64`.

## Statics And Constants

```rust
use std::sync::{LazyLock, OnceLock};

const MAX_BODY_BYTES: usize = 1024 * 1024;

// ✅ Lazily initialised global (stable since Rust 1.80)
static DEFAULT_NAMES: LazyLock<Vec<&'static str>> = LazyLock::new(|| vec!["ada", "grace"]);

// ✅ Set-once value initialised at runtime (e.g. from config)
static BASE_URL: OnceLock<String> = OnceLock::new();

fn main() {
    assert_eq!(MAX_BODY_BYTES, 1_048_576);
    assert_eq!(DEFAULT_NAMES.len(), 2);
    BASE_URL.set("https://example.com".to_owned()).unwrap();
    assert!(BASE_URL.get().is_some());
}
```

- `static mut` is rejected for references in edition 2024. Use atomics, `Mutex`, `OnceLock` or pass state explicitly.
- Prefer dependency injection over global mutable state.

## Lints And Warnings

- Fix warnings; do not blanket-`#![allow(warnings)]`.
- Use `#[expect(lint)]` instead of `#[allow(lint)]` for deliberate exceptions. It warns once the exception is no longer needed.

```rust
#[expect(clippy::too_many_arguments, reason = "mirrors the wire protocol")]
fn send(a: u8, b: u8, c: u8, d: u8, e: u8, f: u8, g: u8, h: u8) -> u32 {
    [a, b, c, d, e, f, g, h].iter().map(|&x| u32::from(x)).sum()
}

fn main() {
    assert_eq!(send(1, 1, 1, 1, 1, 1, 1, 1), 8);
}
```

## Quick Checklist

- [ ] `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test` all pass
- [ ] No `unwrap()`/`expect()` on fallible runtime input outside tests
- [ ] Function parameters borrow (`&str`, `&[T]`, `&Path`) unless they store the value
- [ ] Public types derive `Debug`; public enums that may grow are `#[non_exhaustive]`
- [ ] No `clone()` added only to satisfy the borrow checker
