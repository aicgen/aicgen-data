# Rust Types And Traits

## Make Invalid States Unrepresentable

### Newtypes

Wrap primitives so the compiler distinguishes them and validation happens once.

```rust
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(u64);

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Email(String);

#[derive(Debug, PartialEq, Eq)]
pub struct InvalidEmail;

impl Email {
    // ✅ Parse, don't validate: an Email value is always valid
    pub fn parse(raw: &str) -> Result<Self, InvalidEmail> {
        let raw = raw.trim();
        match raw.split_once('@') {
            Some((local, domain)) if !local.is_empty() && domain.contains('.') => {
                Ok(Self(raw.to_owned()))
            }
            _ => Err(InvalidEmail),
        }
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl fmt::Display for Email {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str(&self.0)
    }
}

impl From<u64> for UserId {
    fn from(value: u64) -> Self {
        Self(value)
    }
}

// ❌ fn send(user: u64, order: u64)  // easy to swap
// ✅ fn send(user: UserId, order: OrderId)

fn main() {
    assert!(Email::parse("ada@example.com").is_ok());
    assert_eq!(Email::parse("nope"), Err(InvalidEmail));
    assert_eq!(UserId::from(7), UserId(7));
}
```

### Enums For State

Model alternatives as an enum with data per variant instead of a struct full of `Option` fields and flags.

```rust
#[derive(Debug)]
enum Payment {
    Pending,
    Authorized { auth_code: String },
    Captured { auth_code: String, amount_cents: u64 },
    Failed { reason: String },
}

impl Payment {
    fn captured_amount(&self) -> Option<u64> {
        match self {
            Self::Captured { amount_cents, .. } => Some(*amount_cents),
            _ => None,
        }
    }
}

fn main() {
    let p = Payment::Captured { auth_code: "A1".into(), amount_cents: 500 };
    assert_eq!(p.captured_amount(), Some(500));
    assert_eq!(Payment::Pending.captured_amount(), None);
    let _ = (Payment::Authorized { auth_code: String::new() }, Payment::Failed { reason: String::new() });
}
```

### Typestate

Encode protocol steps in the type so out-of-order calls do not compile.

```rust
use std::marker::PhantomData;

struct Draft;
struct Published;

struct Post<State> {
    title: String,
    _state: PhantomData<State>,
}

impl Post<Draft> {
    fn new(title: &str) -> Self {
        Self { title: title.to_owned(), _state: PhantomData }
    }

    fn publish(self) -> Post<Published> {
        Post { title: self.title, _state: PhantomData }
    }
}

impl Post<Published> {
    fn url(&self) -> String {
        format!("/posts/{}", self.title.to_lowercase().replace(' ', "-"))
    }
}

fn main() {
    let post = Post::new("Hello World").publish();
    assert_eq!(post.url(), "/posts/hello-world");
    // Post::new("x").url();   // does not compile: Draft has no url()
}
```

## Builders

Use a builder when a type has many optional settings. Consume `self` so the chain reads naturally and mark the type `#[must_use]`.

```rust
use std::time::Duration;

#[derive(Debug)]
struct Client {
    base_url: String,
    timeout: Duration,
    retries: u32,
}

#[derive(Debug)]
#[must_use]
struct ClientBuilder {
    base_url: String,
    timeout: Duration,
    retries: u32,
}

impl ClientBuilder {
    fn new(base_url: impl Into<String>) -> Self {
        Self { base_url: base_url.into(), timeout: Duration::from_secs(30), retries: 3 }
    }

    fn timeout(mut self, timeout: Duration) -> Self {
        self.timeout = timeout;
        self
    }

    fn retries(mut self, retries: u32) -> Self {
        self.retries = retries;
        self
    }

    fn build(self) -> Client {
        Client { base_url: self.base_url, timeout: self.timeout, retries: self.retries }
    }
}

fn main() {
    let client = ClientBuilder::new("https://api.example.com").retries(5).build();
    assert_eq!(client.retries, 5);
    assert_eq!(client.timeout, Duration::from_secs(30));
    assert!(client.base_url.starts_with("https"));
}
```

## Generics, `impl Trait` And `dyn Trait`

| Need | Use |
|------|-----|
| Fast, monomorphised code, caller picks the type | Generic `fn f<T: Trait>(x: T)` or `impl Trait` argument |
| Return one concrete type you do not want to name | `-> impl Trait` |
| Heterogeneous collection or runtime selection | `Box<dyn Trait>` / `&dyn Trait` / `Arc<dyn Trait + Send + Sync>` |
| Plugin boundary, reduce compile time and binary size | `dyn Trait` |

```rust
use std::fmt::Display;

trait Notifier {
    fn notify(&self, message: &str) -> String;
}

struct Email;
struct Sms;

impl Notifier for Email {
    fn notify(&self, message: &str) -> String {
        format!("email: {message}")
    }
}

impl Notifier for Sms {
    fn notify(&self, message: &str) -> String {
        format!("sms: {message}")
    }
}

// Static dispatch
fn alert_static(notifier: &impl Notifier) -> String {
    notifier.notify("disk full")
}

// Dynamic dispatch over different types
fn alert_all(notifiers: &[Box<dyn Notifier>]) -> Vec<String> {
    notifiers.iter().map(|n| n.notify("disk full")).collect()
}

// Return an unnameable iterator without boxing
fn evens(limit: u32) -> impl Iterator<Item = u32> {
    (0..limit).filter(|n| n % 2 == 0)
}

fn show_all<T: Display>(items: &[T]) -> String {
    items.iter().map(ToString::to_string).collect::<Vec<_>>().join(",")
}

fn main() {
    assert_eq!(alert_static(&Email), "email: disk full");
    assert_eq!(alert_all(&[Box::new(Email), Box::new(Sms)]).len(), 2);
    assert_eq!(evens(7).collect::<Vec<_>>(), vec![0, 2, 4, 6]);
    assert_eq!(show_all(&[1, 2, 3]), "1,2,3");
}
```

- Start concrete. Introduce a trait when you have two real implementations or need a seam for tests.
- Keep trait bounds minimal and put long bounds in a `where` clause.
- A trait is object-safe (usable as `dyn`) only if methods avoid generics and `Self` by value; split off a generic extension trait if needed.
- Do not box or `dyn` inside hot loops unless profiling says it is fine.

## Common Conversion Traits

- `From<T>` / `Into<T>` for infallible conversions; implement `From`, get `Into` for free.
- `TryFrom<T>` / `TryInto<T>` when the conversion can fail.
- `AsRef<T>` / `Borrow<T>` for cheap borrowed views in function arguments.
- `FromStr` for parsing from text (`"42".parse::<UserId>()`).
- `Default` for sensible zero values; derive it when all fields implement it.
- `Display` for user-facing text, `Debug` (derived) for developers. Do not parse `Debug` output.

## `async fn` In Traits (Stable Since Rust 1.75)

```rust
// Compiles without any async runtime. Futures from `async fn` in a public
// trait are not guaranteed `Send`; see the note below.
trait UserStore {
    async fn find(&self, id: u64) -> Option<String>;
}

struct InMemory;

impl UserStore for InMemory {
    async fn find(&self, id: u64) -> Option<String> {
        (id == 1).then(|| "ada".to_owned())
    }
}

async fn display_name<S: UserStore>(store: &S, id: u64) -> String {
    store.find(id).await.unwrap_or_else(|| "unknown".to_owned())
}
```

- Generic code that is spawned on a multi-threaded runtime needs `Send` futures. For a public trait, either write `fn find(&self, id: u64) -> impl Future<Output = Option<String>> + Send;` or use the `trait_variant` crate to generate a `Send` variant.
- `dyn` with `async fn` still needs boxing (`async-trait` crate or manual `Pin<Box<dyn Future>>`).

## Lifetimes And Return-Position `impl Trait` (Edition 2024)

In edition 2024, `-> impl Trait` captures **all** in-scope generic parameters and lifetimes by default. Use `use<..>` to opt out precisely.

```rust
// Edition 2024: the returned iterator borrows `text` automatically.
fn words(text: &str) -> impl Iterator<Item = &str> {
    text.split_whitespace()
}

// Capture only `T`, not the lifetime of `data`: the result does not borrow it
fn repeat_first<'a, T: Clone>(data: &'a [T]) -> impl Iterator<Item = T> + use<T> {
    std::iter::repeat_n(data[0].clone(), 2)
}

fn main() {
    assert_eq!(words("a b c").count(), 3);
    let items = vec![1, 2];
    let repeated = repeat_first(&items);
    drop(items); // allowed: `repeated` does not capture the borrow
    assert_eq!(repeated.collect::<Vec<_>>(), vec![1, 1]);
}
```

- Elide lifetimes when you can. Name them only when the compiler needs it or for readability in structs.
- A struct that holds a reference needs a lifetime parameter; if that gets infectious, store owned data (`String`, `Arc<str>`) instead.

## Sealed And Extension Traits

- Seal a trait you do not want downstream crates to implement by making it depend on a private supertrait.
- Prefer inherent methods over extension traits unless you are adding methods to a foreign type.

```rust
mod sealed {
    pub trait Sealed {}
}

pub trait Backend: sealed::Sealed {
    fn name(&self) -> &'static str;
}

pub struct Memory;
impl sealed::Sealed for Memory {}
impl Backend for Memory {
    fn name(&self) -> &'static str {
        "memory"
    }
}
```

## Quick Checklist

- [ ] IDs and validated strings are newtypes; invalid values cannot be constructed
- [ ] State machines use enums or typestate, not boolean flags
- [ ] Generics for speed, `dyn Trait` for runtime variety, chosen deliberately
- [ ] Public traits that return futures state whether futures are `Send`
- [ ] Public enums and structs that may grow are `#[non_exhaustive]`
