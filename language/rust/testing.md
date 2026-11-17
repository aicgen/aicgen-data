# Rust Testing

## Test Types And Where They Live

| Kind | Location | Sees | Run by |
|------|----------|------|--------|
| Unit tests | `#[cfg(test)] mod tests` in the same file | private items | `cargo test` |
| Integration tests | `tests/*.rs` | public API only (each file is its own crate) | `cargo test` |
| Doc tests | code examples inside `///` doc comments | public API, as a user would call it | `cargo test --doc` |
| Benchmarks | `benches/` | public API | `cargo bench` / criterion / divan |

## Unit Tests

```rust
pub fn parse_port(raw: &str) -> Result<u16, String> {
    let port: u16 = raw.trim().parse().map_err(|e| format!("invalid port `{raw}`: {e}"))?;
    if port == 0 {
        return Err("port must be non-zero".to_owned());
    }
    Ok(port)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_port() {
        assert_eq!(parse_port(" 8080 "), Ok(8080));
    }

    #[test]
    fn rejects_zero() {
        assert_eq!(parse_port("0"), Err("port must be non-zero".to_owned()));
    }

    #[test]
    fn rejects_garbage_with_input_in_message() {
        let err = parse_port("abc").unwrap_err();
        assert!(err.contains("abc"), "message should name the input: {err}");
    }

    // Tests can return Result so `?` replaces unwrap
    #[test]
    fn trims_whitespace() -> Result<(), String> {
        let port = parse_port("\t443\n")?;
        assert_eq!(port, 443);
        Ok(())
    }

    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn indexing_past_the_end_panics() {
        let values: Vec<u32> = Vec::new();
        let _ = values[1];
    }
}
```

- Name tests after behaviour (`rejects_zero`), not the function (`test_parse_port_2`).
- Prefer `assert_eq!(actual, expected)`; both sides need `Debug + PartialEq`. Add a message when the failure would be ambiguous.
- Keep tests independent and parallel-safe. `cargo test` runs tests in parallel threads; share no mutable global state or fixed ports/files.
- Assert on one behaviour per test. Group related tests in nested modules.

## Table-Driven Tests

```rust
fn classify(n: i32) -> &'static str {
    match n {
        i32::MIN..=-1 => "negative",
        0 => "zero",
        _ => "positive",
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn classifies_numbers() {
        let cases = [(-5, "negative"), (0, "zero"), (7, "positive")];
        for (input, expected) in cases {
            assert_eq!(classify(input), expected, "input = {input}");
        }
    }
}
```

For many cases with separate failure reporting, use the `rstest` crate's `#[case]` attributes.

## Test Doubles: Fakes Over Mocks

Define the dependency as a trait, inject it, and write a small hand-rolled fake. Reach for `mockall` only when you need to assert exact call patterns.

```rust
use std::cell::RefCell;
use std::collections::HashMap;

pub trait UserRepo {
    fn find(&self, id: u64) -> Option<String>;
    fn save(&self, id: u64, name: &str);
}

pub struct Service<R: UserRepo> {
    repo: R,
}

impl<R: UserRepo> Service<R> {
    pub fn new(repo: R) -> Self {
        Self { repo }
    }

    pub fn rename(&self, id: u64, name: &str) -> Result<(), String> {
        if self.repo.find(id).is_none() {
            return Err(format!("user {id} not found"));
        }
        self.repo.save(id, name);
        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[derive(Default)]
    struct FakeRepo {
        users: RefCell<HashMap<u64, String>>,
    }

    impl UserRepo for FakeRepo {
        fn find(&self, id: u64) -> Option<String> {
            self.users.borrow().get(&id).cloned()
        }
        fn save(&self, id: u64, name: &str) {
            self.users.borrow_mut().insert(id, name.to_owned());
        }
    }

    #[test]
    fn renames_existing_user() {
        let repo = FakeRepo::default();
        repo.save(1, "ada");
        let service = Service::new(repo);

        service.rename(1, "grace").unwrap();

        assert_eq!(service.repo.find(1).as_deref(), Some("grace"));
    }

    #[test]
    fn rejects_unknown_user() {
        let service = Service::new(FakeRepo::default());
        assert_eq!(service.rename(9, "x"), Err("user 9 not found".to_owned()));
    }
}
```

- Fake time, randomness and the network behind traits or parameters. Do not sleep in tests.
- For async code use `#[tokio::test(start_paused = true)]` to control time instead of real sleeps.

## Integration Tests

```
myapp/
├── src/lib.rs
└── tests/
    ├── common/
    │   └── mod.rs      # shared helpers (not compiled as a test crate)
    ├── api.rs
    └── cli.rs
```

```rust,ignore
// tests/api.rs
mod common;

#[test]
fn creates_and_reads_a_user() {
    let app = common::spawn_app();           // bind to port 0, return the address
    let created = app.post("/users", r#"{"name":"ada"}"#);
    assert_eq!(created.status(), 201);
    assert_eq!(app.get(created.location()).status(), 200);
}
```

- Integration tests exercise only the public API. If you need private access, it is a unit test.
- Binary-only crates cannot be imported by `tests/`. Keep logic in `lib.rs` and a thin `main.rs`.
- Bind servers to port 0 and use temporary directories (`tempfile` crate) so tests never collide.
- Test CLIs end to end with `assert_cmd` + `predicates`.
- Use real dependencies (Postgres via Testcontainers or a temporary SQLite file) for repository code; fakes cannot prove your SQL works.

## Doc Tests

Examples in `///` comments are compiled and run. They keep documentation honest.

```rust,ignore
/// Splits `"key=value"` into its parts.
///
/// # Examples
///
/// ```
/// use mycrate::split_pair;
///
/// assert_eq!(split_pair("a=1"), Some(("a", "1")));
/// assert_eq!(split_pair("nope"), None);
/// ```
///
/// # Errors
/// This function does not fail; it returns `None` when there is no `=`.
pub fn split_pair(s: &str) -> Option<(&str, &str)> {
    s.split_once('=')
}
```

- `cargo nextest` does not run doc tests. Run `cargo test --doc` as a separate CI step.
- Hide boilerplate lines with a leading `# `. Use `no_run` for code that should compile but not execute.

## Property-Based Tests

Use `proptest` (or `quickcheck`) for parsers, serializers, and anything with an inverse.

```rust,ignore
use proptest::prelude::*;

proptest! {
    #[test]
    fn roundtrips_through_json(user in any::<User>()) {
        let json = serde_json::to_string(&user).unwrap();
        let back: User = serde_json::from_str(&json).unwrap();
        prop_assert_eq!(user, back);
    }

    #[test]
    fn never_panics_on_arbitrary_input(input in ".*") {
        let _ = parse_port(&input);
    }
}
```

Commit the `proptest-regressions/` directory so found failures stay covered.

## Snapshot Tests

Use `insta` for large structured output (rendered text, JSON, CLI help) where hand-written expected values are noisy.

```rust,ignore
#[test]
fn renders_invoice() {
    let text = render(&sample_invoice());
    insta::assert_snapshot!(text);
}
```

Review snapshot diffs in code review (`cargo insta review`). Do not accept snapshots blindly.

## Async Tests

```rust,ignore
#[tokio::test]
async fn fetches_user() {
    let service = Service::new(FakeRepo::default());
    assert!(service.find(1).await.is_none());
}

// Virtual time: sleeps complete instantly and deterministically
#[tokio::test(start_paused = true)]
async fn retries_after_backoff() {
    let result = retry_with_backoff(flaky_call).await;
    assert!(result.is_ok());
}
```

## Running Tests

```bash
cargo test                           # unit + integration + doc tests
cargo test parse_port                # filter by name
cargo test -- --nocapture            # show println! output
cargo test --doc                     # doc tests only
cargo test --all-features            # include optional features
cargo nextest run                    # faster runner, per-test isolation, retries (no doc tests)
cargo llvm-cov --lcov --output-path lcov.info   # coverage
cargo +nightly miri test             # detect undefined behaviour in unsafe code
```

## What To Cover

- Behaviour and edge cases: empty input, boundaries, invalid input, error paths, concurrency limits.
- Every bug fix gets a regression test that fails before the fix.
- Do not chase coverage percentages on generated code or trivial getters. Test public behaviour, not private structure.
- Flaky tests are bugs. Quarantine with `#[ignore = "reason, issue link"]` only temporarily.

## Quick Checklist

- [ ] Unit tests beside the code, integration tests in `tests/`, examples as doc tests
- [ ] Tests are named by behaviour and run in parallel safely
- [ ] Dependencies are behind traits; fakes preferred over heavy mocking
- [ ] No real sleeps, fixed ports or shared files
- [ ] CI runs `cargo test --doc` in addition to nextest
- [ ] Parsers and codecs have property tests
