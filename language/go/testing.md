# Go Testing

## Test File Structure

```
myproject/
├── user.go
├── user_test.go      # Tests alongside source
├── user_internal_test.go  # Internal tests (same package)
└── testdata/         # Test fixtures
    └── fixtures.json
```

## Test Functions

```go
// Basic test
func TestCreateUser(t *testing.T) {
    user := CreateUser("test@example.com")

    if user.Email != "test@example.com" {
        t.Errorf("expected email %q, got %q", "test@example.com", user.Email)
    }
}

// Table-driven tests (preferred)
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        {"valid email", "test@example.com", false},
        {"missing @", "invalid", true},
        {"empty", "", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateEmail(tt.email)
            if (err != nil) != tt.wantErr {
                t.Errorf("ValidateEmail(%q) error = %v, wantErr %v",
                    tt.email, err, tt.wantErr)
            }
        })
    }
}
```

## Subtests and Parallel

```go
func TestUserService(t *testing.T) {
    t.Run("Create", func(t *testing.T) {
        t.Parallel() // Run in parallel
        // test create
    })

    t.Run("Update", func(t *testing.T) {
        t.Parallel()
        // test update
    })
}
```

## Test Helpers

```go
// Helper function (call t.Helper() first)
func createTestUser(t *testing.T, email string) *User {
    t.Helper()
    user, err := NewUser(email)
    if err != nil {
        t.Fatalf("failed to create test user: %v", err)
    }
    return user
}

// Cleanup
func TestWithTempFile(t *testing.T) {
    f, err := os.CreateTemp("", "test")
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() { os.Remove(f.Name()) })

    // use f...
}
```

## Mocking with Interfaces

```go
// Define interface for dependency
type UserRepository interface {
    Save(user *User) error
    FindByID(id string) (*User, error)
}

// Mock implementation
type mockRepository struct {
    users map[string]*User
    saveErr error
}

func (m *mockRepository) Save(user *User) error {
    if m.saveErr != nil {
        return m.saveErr
    }
    m.users[user.ID] = user
    return nil
}

func (m *mockRepository) FindByID(id string) (*User, error) {
    if user, ok := m.users[id]; ok {
        return user, nil
    }
    return nil, ErrNotFound
}

// Use in tests
func TestUserService_Create(t *testing.T) {
    repo := &mockRepository{users: make(map[string]*User)}
    service := NewUserService(repo)

    user, err := service.Create("test@example.com")

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if _, exists := repo.users[user.ID]; !exists {
        t.Error("user not saved to repository")
    }
}
```

## Benchmarks

```go
func BenchmarkProcessUsers(b *testing.B) {
    users := generateTestUsers(1000)
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        ProcessUsers(users)
    }
}

// Run: go test -bench=. -benchmem
```

## Test Coverage

```bash
# Generate coverage
go test -coverprofile=coverage.out ./...

# View coverage report
go tool cover -html=coverage.out

# Check coverage percentage
go test -cover ./...
```

## Integration Tests

```go
//go:build integration

package mypackage_test

import (
    "testing"
    "database/sql"
)

func TestDatabaseIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    db, err := sql.Open("postgres", os.Getenv("TEST_DATABASE_URL"))
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() { db.Close() })

    // integration tests...
}

// Run: go test -tags=integration ./...
```
