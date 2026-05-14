# Go Fundamentals

## Project Structure

```
myproject/
├── cmd/
│   └── myapp/
│       └── main.go       # Application entry point
├── internal/             # Private application code
│   ├── handler/
│   ├── service/
│   └── repository/
├── pkg/                  # Public library code
├── go.mod
└── go.sum
```

## Naming Conventions

```go
// Package names: lowercase, single word
package user

// Exported (public): PascalCase
func CreateUser() {}
type UserService struct {}

// Unexported (private): camelCase
func validateEmail() {}
type userCache struct {}

// Interfaces: -er suffix when possible
type Reader interface {}
type UserRepository interface {}

// Constants: PascalCase for exported
const MaxRetries = 3
const defaultTimeout = 30
```

## Error Handling

```go
// Always check errors
result, err := doSomething()
if err != nil {
    return fmt.Errorf("failed to do something: %w", err)
}

// Custom errors
var ErrNotFound = errors.New("not found")
var ErrInvalidInput = errors.New("invalid input")

// Error wrapping for context
if err != nil {
    return fmt.Errorf("creating user %s: %w", name, err)
}

// Errors.Is and errors.As for checking
if errors.Is(err, ErrNotFound) {
    // Handle not found
}
```

## Concurrency

```go
// Goroutines for concurrent work
go func() {
    // Concurrent task
}()

// Channels for communication
ch := make(chan int)
ch <- value    // Send
value := <-ch  // Receive

// Context for cancellation
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// sync.WaitGroup for coordination
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    // Work
}()
wg.Wait()
```

## Testing

```go
// Test file: user_test.go
func TestCreateUser(t *testing.T) {
    // Arrange
    svc := NewUserService()

    // Act
    user, err := svc.Create("test@example.com")

    // Assert
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Email != "test@example.com" {
        t.Errorf("got %s, want test@example.com", user.Email)
    }
}

// Table-driven tests
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        valid bool
    }{
        {"valid", "test@example.com", true},
        {"no at", "invalid", false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := ValidateEmail(tt.email)
            if got != tt.valid {
                t.Errorf("ValidateEmail(%q) = %v, want %v", tt.email, got, tt.valid)
            }
        })
    }
}
```

## Common Patterns

```go
// Functional options
type Option func(*Config)

func WithTimeout(d time.Duration) Option {
    return func(c *Config) { c.Timeout = d }
}

func NewClient(opts ...Option) *Client {
    cfg := defaultConfig()
    for _, opt := range opts {
        opt(&cfg)
    }
    return &Client{config: cfg}
}

// Interface-based design
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
}
```
