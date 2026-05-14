# Rust Testing

## Test Module Structure

```rust
// In src/lib.rs or src/user.rs
pub fn create_user(email: &str) -> Result<User, Error> {
    // implementation
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_create_user() {
        let user = create_user("test@example.com").unwrap();
        assert_eq!(user.email, "test@example.com");
    }
}
```

## Assertions

```rust
#[test]
fn test_assertions() {
    // Equality
    assert_eq!(actual, expected);
    assert_ne!(value1, value2);

    // Boolean
    assert!(condition);
    assert!(!condition);

    // With custom message
    assert_eq!(result, expected, "failed for input: {}", input);
}
```

## Testing Results and Options

```rust
#[test]
fn test_result() -> Result<(), Error> {
    let user = create_user("test@example.com")?;
    assert_eq!(user.email, "test@example.com");
    Ok(())
}

#[test]
fn test_option() {
    let result = find_user("123");
    assert!(result.is_some());
    assert_eq!(result.unwrap().id, "123");
}
```

## Testing Panics

```rust
#[test]
#[should_panic]
fn test_panics() {
    divide(1, 0);
}

#[test]
#[should_panic(expected = "division by zero")]
fn test_panic_message() {
    divide(1, 0);
}
```

## Test Organization

```rust
#[cfg(test)]
mod tests {
    use super::*;

    mod create {
        use super::*;

        #[test]
        fn with_valid_email() {
            // test
        }

        #[test]
        fn with_invalid_email() {
            // test
        }
    }

    mod update {
        use super::*;

        #[test]
        fn existing_user() {
            // test
        }
    }
}
```

## Test Fixtures

```rust
struct TestContext {
    db: MockDatabase,
    service: UserService,
}

impl TestContext {
    fn new() -> Self {
        let db = MockDatabase::new();
        let service = UserService::new(db.clone());
        Self { db, service }
    }
}

#[test]
fn test_with_context() {
    let ctx = TestContext::new();
    let user = ctx.service.create("test@example.com").unwrap();
    assert!(ctx.db.contains(&user.id));
}
```

## Mocking with Traits

```rust
// Define trait
trait UserRepository {
    fn save(&self, user: &User) -> Result<(), Error>;
    fn find_by_id(&self, id: &str) -> Option<User>;
}

// Mock implementation
struct MockRepository {
    users: RefCell<HashMap<String, User>>,
}

impl UserRepository for MockRepository {
    fn save(&self, user: &User) -> Result<(), Error> {
        self.users.borrow_mut().insert(user.id.clone(), user.clone());
        Ok(())
    }

    fn find_by_id(&self, id: &str) -> Option<User> {
        self.users.borrow().get(id).cloned()
    }
}

#[test]
fn test_service_create() {
    let repo = MockRepository { users: RefCell::new(HashMap::new()) };
    let service = UserService::new(Box::new(repo));

    let user = service.create("test@example.com").unwrap();
    assert_eq!(user.email, "test@example.com");
}
```

## Integration Tests

```
tests/
├── integration_test.rs
└── common/
    └── mod.rs
```

```rust
// tests/common/mod.rs
pub fn setup() -> TestDatabase {
    TestDatabase::new()
}

// tests/integration_test.rs
mod common;

#[test]
fn test_database_integration() {
    let db = common::setup();
    // integration test...
}
```

## Async Tests

```rust
#[tokio::test]
async fn test_async_create() {
    let service = UserService::new();
    let user = service.create_async("test@example.com").await.unwrap();
    assert_eq!(user.email, "test@example.com");
}
```

## Running Tests

```bash
# Run all tests
cargo test

# Run specific test
cargo test test_create_user

# Run tests with output
cargo test -- --nocapture

# Run ignored tests
cargo test -- --ignored

# Run benchmarks (nightly)
cargo bench
```
