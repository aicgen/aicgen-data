# Rust Error Handling

## Principles

- Recoverable failures are `Result<T, E>`. Absence is `Option<T>`. Bugs and broken invariants are panics.
- Libraries return precise, typed errors callers can match on. Applications use one catch-all error type at the top and add context.
- Handle an error once, at the layer that can decide what to do (retry, fallback, report). Do not log and return the same error at every level.

## Library Errors: Typed Enums

Define an enum per module or crate. Implement `Display` (the message, lowercase, no trailing period) and `Error` (with `source()` for the cause).

```rust
use std::fmt;

#[derive(Debug)]
pub enum ConfigError {
    Missing(&'static str),
    Invalid { key: &'static str, value: String },
    Parse(std::num::ParseIntError),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::Missing(key) => write!(f, "missing config key `{key}`"),
            Self::Invalid { key, value } => write!(f, "invalid value `{value}` for `{key}`"),
            Self::Parse(_) => write!(f, "could not parse number"),
        }
    }
}

impl std::error::Error for ConfigError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            Self::Parse(err) => Some(err),
            _ => None,
        }
    }
}

impl From<std::num::ParseIntError> for ConfigError {
    fn from(err: std::num::ParseIntError) -> Self {
        Self::Parse(err)
    }
}

pub fn port(raw: Option<&str>) -> Result<u16, ConfigError> {
    let raw = raw.ok_or(ConfigError::Missing("PORT"))?;
    let port: u16 = raw.parse()?; // ParseIntError -> ConfigError via From
    if port == 0 {
        return Err(ConfigError::Invalid { key: "PORT", value: raw.to_owned() });
    }
    Ok(port)
}

fn main() {
    assert_eq!(port(Some("8080")).unwrap(), 8080);
    assert!(matches!(port(None), Err(ConfigError::Missing("PORT"))));
    assert!(matches!(port(Some("0")), Err(ConfigError::Invalid { .. })));
    assert!(matches!(port(Some("x")), Err(ConfigError::Parse(_))));
}
```

With `thiserror` (current major version is 2.x) the same type is much shorter. Use it in libraries.

```rust,ignore
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("missing config key `{0}`")]
    Missing(&'static str),
    #[error("invalid value `{value}` for `{key}`")]
    Invalid { key: &'static str, value: String },
    #[error("could not parse number")]
    Parse(#[from] std::num::ParseIntError),
}
```

- Use `#[from]` for transparent wrapping and `#[source]` when you want the cause chain without a `From` impl.
- Do not expose a dependency's error type in your public API unless that dependency is part of the contract. Wrap it.
- Mark public error enums `#[non_exhaustive]` so adding a variant is not a breaking change.
- Make error types `Send + Sync + 'static` so they work across threads and with `anyhow`.

## Application Errors: `anyhow`

Binaries rarely match on error kinds. They need context and a readable report.

```rust,ignore
use anyhow::{bail, Context, Result};

fn load(path: &std::path::Path) -> Result<Settings> {
    let text = std::fs::read_to_string(path)
        .with_context(|| format!("reading {}", path.display()))?;
    let settings: Settings = toml::from_str(&text).context("parsing settings")?;
    if settings.workers == 0 {
        bail!("workers must be at least 1");
    }
    Ok(settings)
}

fn main() -> Result<()> {
    let settings = load("app.toml".as_ref())?;
    run(settings)
}
```

- `anyhow::Error` prints the whole chain with `{:#}` (single line) or `{:?}` (multi-line report). Print it once at the boundary.
- Use `.context()` to say what you were doing, not what went wrong; the underlying error already says that.
- Do not return `anyhow::Error` from a library. Callers cannot match on it.

## The `?` Operator And Conversions

```rust
use std::error::Error;

// ✅ Quick prototype or small tool: Box<dyn Error>
fn read_number(text: &str) -> Result<i32, Box<dyn Error + Send + Sync>> {
    let n: i32 = text.trim().parse()?;
    if n < 0 {
        return Err("negative numbers are not allowed".into());
    }
    Ok(n)
}

// ✅ Convert Option to Result with context
fn first_arg(args: &[String]) -> Result<&str, &'static str> {
    args.first().map(String::as_str).ok_or("expected one argument")
}

fn main() {
    assert_eq!(read_number(" 5 ").unwrap(), 5);
    assert!(read_number("-1").is_err());
    assert!(first_arg(&[]).is_err());
}
```

## Panics

Panic only when continuing would be wrong and the caller cannot reasonably recover: a violated internal invariant, an impossible state, or a failed test assertion.

| Situation | Use |
|-----------|-----|
| Bad user input, I/O, network, parsing | `Result` |
| Missing optional data | `Option` |
| Internal invariant that cannot be false | `expect("invariant: ...")` with the reason |
| Unfinished code | `todo!()` (never merge to main) |
| Provably unreachable branch | `unreachable!("reason")` |
| Test code | `unwrap()` / `expect()` are fine |

```rust
use std::collections::HashMap;

// ✅ expect documents why this cannot fail
fn lookup(table: &HashMap<&'static str, u32>, key: &'static str) -> u32 {
    *table.get(key).expect("invariant: every key is registered at startup")
}

// ✅ Prefer non-panicking APIs
fn middle(values: &[i32]) -> Option<i32> {
    values.get(values.len() / 2).copied() // not values[values.len() / 2]
}

fn main() {
    let table = HashMap::from([("a", 1)]);
    assert_eq!(lookup(&table, "a"), 1);
    assert_eq!(middle(&[]), None);
}
```

- Enforce `clippy::unwrap_used`, `clippy::expect_used` and `clippy::panic` as warnings in non-test code (see `tooling.md`).
- Indexing (`v[i]`), integer overflow in release builds, `RefCell::borrow_mut` and division by zero can panic. Use `get`, checked arithmetic and `try_borrow_mut` when the input is not trusted.
- In servers, a panic in one task must not take down the process silently. Log it with context and return a 500; never use panics for control flow.

## Error Messages

- Say what failed and with which values: `failed to open "app.toml": permission denied`.
- Lowercase, no trailing punctuation, no leading "Error:" (callers and `anyhow` add framing).
- Never include secrets, tokens or full request bodies in error text.
- Map internal errors to a stable external representation at the API boundary (HTTP status, error code) instead of leaking `Debug` output.

## Quick Checklist

- [ ] Library errors are typed enums implementing `std::error::Error`
- [ ] Binaries use `anyhow` (or `eyre`) with `.context()` at I/O boundaries
- [ ] No `unwrap()`/`expect()` on runtime input outside tests
- [ ] Each error is handled or reported once, not logged at every layer
- [ ] Errors carry no secrets and map to stable codes at the API edge
