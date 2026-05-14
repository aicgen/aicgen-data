# Rust Fundamentals

## Project Structure

```
myproject/
├── src/
│   ├── main.rs           # Binary entry point
│   ├── lib.rs            # Library root
│   ├── models/
│   │   └── mod.rs
│   └── services/
│       └── mod.rs
├── tests/                # Integration tests
├── Cargo.toml
└── Cargo.lock
```

## Ownership & Borrowing

```rust
// Ownership: each value has one owner
let s1 = String::from("hello");
let s2 = s1;  // s1 is moved, no longer valid

// Borrowing: references without ownership
fn print_len(s: &String) {
    println!("{}", s.len());
}

// Mutable borrowing (only one at a time)
fn append(s: &mut String) {
    s.push_str(" world");
}

// Lifetimes: ensure references are valid
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

## Error Handling

```rust
// Result for recoverable errors
fn read_file(path: &str) -> Result<String, io::Error> {
    fs::read_to_string(path)
}

// ? operator for propagation
fn process_file(path: &str) -> Result<Data, Error> {
    let content = fs::read_to_string(path)?;
    let data = parse(&content)?;
    Ok(data)
}

// Custom error types
#[derive(Debug)]
enum AppError {
    NotFound(String),
    InvalidInput(String),
    Database(sqlx::Error),
}

impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        AppError::Database(err)
    }
}
```

## Option & Pattern Matching

```rust
// Option for nullable values
fn find_user(id: u64) -> Option<User> {
    users.get(&id).cloned()
}

// Pattern matching
match find_user(42) {
    Some(user) => println!("Found: {}", user.name),
    None => println!("Not found"),
}

// if let for single patterns
if let Some(user) = find_user(42) {
    process(user);
}

// Combinators
find_user(42)
    .map(|u| u.email)
    .unwrap_or_default()
```

## Traits

```rust
// Define behavior
trait Repository {
    fn find(&self, id: u64) -> Option<Entity>;
    fn save(&mut self, entity: Entity) -> Result<(), Error>;
}

// Implement for types
impl Repository for PostgresRepo {
    fn find(&self, id: u64) -> Option<Entity> {
        // Implementation
    }
}

// Trait bounds
fn process<T: Repository + Clone>(repo: T) {
    // Can use Repository and Clone methods
}

// Default implementations
trait Greet {
    fn greet(&self) -> String {
        String::from("Hello!")
    }
}
```

## Testing

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_create_user() {
        let user = User::new("test@example.com");
        assert_eq!(user.email, "test@example.com");
    }

    #[test]
    #[should_panic(expected = "invalid email")]
    fn test_invalid_email_panics() {
        User::new("invalid");
    }

    #[test]
    fn test_find_user() -> Result<(), Error> {
        let repo = TestRepo::new();
        let user = repo.find(1)?;
        assert!(user.is_some());
        Ok(())
    }
}
```

## Async/Await

```rust
// Async functions
async fn fetch_data(url: &str) -> Result<Data, Error> {
    let response = reqwest::get(url).await?;
    let data = response.json().await?;
    Ok(data)
}

// Spawning tasks
let handle = tokio::spawn(async {
    fetch_data("https://api.example.com").await
});

// Concurrent execution
let (result1, result2) = tokio::join!(
    fetch_data("url1"),
    fetch_data("url2")
);
```
