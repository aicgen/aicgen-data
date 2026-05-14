# Testing Basics

## Your First Unit Test

A unit test verifies that a small piece of code works correctly.

```pseudocode
// Function to test
function add(a, b):
    return a + b

// Test for the function
test "add should sum two numbers":
    result = add(2, 3)
    expect result equals 5
```

## Test Structure

Every test has three parts:

1. **Setup** - Prepare what you need
2. **Execute** - Run the code
3. **Verify** - Check the result

```pseudocode
test "should create user with name":
    // 1. Setup
    userName = "Alice"

    // 2. Execute
    user = createUser(userName)

    // 3. Verify
    expect user.name equals "Alice"
```

## Common Assertions

```pseudocode
// Equality
expect value equals 5
expect value equals { id: 1 }

// Truthiness
expect value is truthy
expect value is falsy
expect value is null
expect value is undefined

// Numbers
expect value > 3
expect value < 10

// Strings
expect text contains "hello"

// Arrays/Lists
expect array contains item
expect array length equals 3
```

## Testing Expected Errors

```pseudocode
test "should throw error for invalid input":
    expect error when:
        divide(10, 0)
    with message "Cannot divide by zero"
```

## Async Tests

```pseudocode
test "should fetch user data" async:
    user = await fetchUser(123)
    expect user.id equals 123
```

## Test Naming

Use clear, descriptive names:

```pseudocode
// ❌ Bad
test "test1"
test "it works"

// ✅ Good
test "should return user when ID exists"
test "should throw error when ID is invalid"
```

## Running Tests

```bash
# Run all tests
run-tests

# Run specific test file
run-tests user-test

# Watch mode (re-run on changes)
run-tests --watch
```

## Best Practices

1. **One test, one thing** - Test one behavior per test
2. **Independent tests** - Tests should not depend on each other
3. **Clear names** - Name should describe what is being tested
4. **Fast tests** - Tests should run quickly

```pseudocode
// ❌ Bad: Testing multiple things
test "user operations":
    expect createUser("Bob").name equals "Bob"
    expect deleteUser(1) equals true
    expect listUsers().length equals 0

// ✅ Good: One test per operation
test "should create user with given name":
    user = createUser("Bob")
    expect user.name equals "Bob"

test "should delete user by ID":
    result = deleteUser(1)
    expect result equals true

test "should return empty list when no users":
    users = listUsers()
    expect users.length equals 0
```

## When to Write Tests

- **Before fixing bugs** - Write test that fails, then fix
- **For new features** - Test expected behavior
- **For edge cases** - Empty input, null values, large numbers

## What to Test

✅ **Do test:**
- Public functions and methods
- Edge cases (empty, null, zero, negative)
- Error conditions

❌ **Don't test:**
- Private implementation details
- Third-party libraries (they're already tested)
- Getters/setters with no logic
