# JavaScript Testing (Jest)

## Project Structure

```
myproject/
├── src/
│   └── services/
│       └── userService.js
└── tests/          # or __tests__/
    └── services/
        └── userService.test.js
```

## Basic Tests

```javascript
const { UserService } = require("../src/services/userService");

describe("UserService", () => {
  let service;

  beforeEach(() => {
    service = new UserService();
  });

  describe("create", () => {
    it("creates user with valid email", async () => {
      const user = await service.create({ email: "test@example.com" });

      expect(user.email).toBe("test@example.com");
      expect(user.id).toBeDefined();
    });

    it("throws on invalid email", async () => {
      await expect(service.create({ email: "invalid" }))
        .rejects
        .toThrow(ValidationError);
    });
  });
});
```

## Matchers

```javascript
// Equality
expect(value).toBe(exact);           // ===
expect(value).toEqual(deepEqual);    // deep equality
expect(value).toStrictEqual(obj);    // deep + type

// Truthiness
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();

// Numbers
expect(value).toBeGreaterThan(3);
expect(value).toBeLessThanOrEqual(5);
expect(value).toBeCloseTo(0.3, 5);   // floating point

// Strings
expect(string).toMatch(/pattern/);
expect(string).toContain("substring");

// Arrays
expect(array).toContain(item);
expect(array).toHaveLength(3);
expect(array).toContainEqual({ id: 1 });

// Objects
expect(object).toHaveProperty("name");
expect(object).toHaveProperty("user.email", "test@example.com");
expect(object).toMatchObject({ name: "John" });

// Exceptions
expect(() => fn()).toThrow();
expect(() => fn()).toThrow(ErrorType);
expect(() => fn()).toThrow("message");
```

## Async Testing

```javascript
// Async/await
it("fetches user", async () => {
  const user = await service.fetch("123");
  expect(user.id).toBe("123");
});

// Promises
it("fetches user", () => {
  return service.fetch("123").then(user => {
    expect(user.id).toBe("123");
  });
});

// Rejections
it("rejects on not found", async () => {
  await expect(service.fetch("invalid"))
    .rejects
    .toThrow(NotFoundError);
});
```

## Mocking

```javascript
// Mock functions
const mockFn = jest.fn();
mockFn.mockReturnValue(42);
mockFn.mockResolvedValue(user);
mockFn.mockRejectedValue(new Error("failed"));

// Mock implementations
mockFn.mockImplementation((x) => x * 2);

// Verify calls
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledWith("arg1", "arg2");
expect(mockFn).toHaveBeenCalledTimes(2);

// Mock modules
jest.mock("../src/services/emailService");
const { EmailService } = require("../src/services/emailService");
EmailService.prototype.send = jest.fn().mockResolvedValue(true);

// Spy on existing methods
const spy = jest.spyOn(console, "log");
// ... test
spy.mockRestore();
```

## Testing with Mocked Dependencies

```javascript
describe("UserService", () => {
  let repository;
  let notifier;
  let service;

  beforeEach(() => {
    repository = {
      save: jest.fn(),
      find: jest.fn()
    };
    notifier = {
      sendWelcome: jest.fn().mockResolvedValue(true)
    };
    service = new UserService(repository, notifier);
  });

  it("saves user and sends welcome", async () => {
    const userData = { email: "test@example.com" };
    repository.save.mockResolvedValue({ id: "1", ...userData });

    const user = await service.create(userData);

    expect(repository.save).toHaveBeenCalledWith(expect.objectContaining({
      email: "test@example.com"
    }));
    expect(notifier.sendWelcome).toHaveBeenCalledWith(user);
  });
});
```

## Setup and Teardown

```javascript
beforeAll(() => {
  // Run once before all tests
});

beforeEach(() => {
  // Run before each test
});

afterEach(() => {
  // Run after each test
  jest.clearAllMocks();
});

afterAll(() => {
  // Run once after all tests
});
```

## Snapshot Testing

```javascript
it("renders correctly", () => {
  const tree = renderer.create(<Button label="Click me" />).toJSON();
  expect(tree).toMatchSnapshot();
});

// Inline snapshot
it("serializes correctly", () => {
  expect(user.toJSON()).toMatchInlineSnapshot(`
    {
      "email": "test@example.com",
      "id": "1"
    }
  `);
});
```

## Test Each (Parameterized)

```javascript
describe("validateEmail", () => {
  it.each([
    ["test@example.com", true],
    ["invalid", false],
    ["", false]
  ])("validates %s as %s", (email, expected) => {
    expect(validateEmail(email)).toBe(expected);
  });

  // With named parameters
  it.each`
    email                 | expected
    ${"test@example.com"} | ${true}
    ${"invalid"}          | ${false}
  `("validates $email", ({ email, expected }) => {
    expect(validateEmail(email)).toBe(expected);
  });
});
```

## Timer Mocking

```javascript
jest.useFakeTimers();

it("calls callback after delay", () => {
  const callback = jest.fn();

  delayedCall(callback, 1000);

  expect(callback).not.toHaveBeenCalled();
  jest.advanceTimersByTime(1000);
  expect(callback).toHaveBeenCalled();
});

afterEach(() => {
  jest.useRealTimers();
});
```

## Running Tests

```bash
# Run all tests
npm test

# Watch mode
npm test -- --watch

# Coverage
npm test -- --coverage

# Run specific file
npm test -- userService.test.js

# Run matching tests
npm test -- -t "creates user"
```
