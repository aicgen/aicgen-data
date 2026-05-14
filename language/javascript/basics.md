# JavaScript Fundamentals

## Project Structure

```
myproject/
├── src/
│   ├── index.js          # Entry point
│   ├── config/
│   ├── controllers/
│   ├── services/
│   ├── models/
│   └── utils/
├── tests/
├── package.json
└── .eslintrc.js
```

## Modern JavaScript (ES6+)

```javascript
// const/let instead of var
const immutable = "cannot change";
let mutable = "can change";

// Arrow functions
const add = (a, b) => a + b;
const users = data.map(d => new User(d));

// Destructuring
const { name, email } = user;
const [first, ...rest] = items;

// Spread operator
const merged = { ...defaults, ...options };
const combined = [...array1, ...array2];

// Template literals
const message = `Hello, ${name}!`;

// Optional chaining
const avatar = user?.profile?.avatar ?? defaultAvatar;

// Nullish coalescing
const value = input ?? defaultValue;
```

## Async/Await

```javascript
// Async functions
async function fetchUser(id) {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new Error(`Failed to fetch user: ${response.status}`);
  }
  return response.json();
}

// Error handling
async function processUser(id) {
  try {
    const user = await fetchUser(id);
    return await processData(user);
  } catch (error) {
    console.error("Processing failed:", error);
    throw error;
  }
}

// Parallel execution
const [users, posts] = await Promise.all([
  fetchUsers(),
  fetchPosts()
]);

// Promise.allSettled for partial failures
const results = await Promise.allSettled([
  fetchUser(1),
  fetchUser(2),
  fetchUser(3)
]);
```

## Classes

```javascript
class UserService {
  #repository; // Private field

  constructor(repository) {
    this.#repository = repository;
  }

  async create(data) {
    const user = new User(data);
    await this.#validate(user);
    return this.#repository.save(user);
  }

  async #validate(user) {
    if (!user.email) {
      throw new ValidationError("Email required");
    }
  }

  static fromConfig(config) {
    return new UserService(new Repository(config));
  }
}
```

## Error Handling

```javascript
// Custom errors
class NotFoundError extends Error {
  constructor(id) {
    super(`Not found: ${id}`);
    this.name = "NotFoundError";
    this.id = id;
  }
}

class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
  }
}

// Error handling
async function handleRequest(req) {
  try {
    return await processRequest(req);
  } catch (error) {
    if (error instanceof NotFoundError) {
      return { status: 404, message: error.message };
    }
    if (error instanceof ValidationError) {
      return { status: 400, message: error.message, field: error.field };
    }
    throw error; // Re-throw unexpected errors
  }
}
```

## Modules

```javascript
// Named exports
export function createUser(data) { }
export const MAX_USERS = 100;

// Default export
export default class UserService { }

// Imports
import UserService from "./user-service.js";
import { createUser, MAX_USERS } from "./utils.js";
import * as utils from "./utils.js";
```

## Array Methods

```javascript
// Map, filter, reduce
const emails = users.map(u => u.email);
const active = users.filter(u => u.isActive);
const total = orders.reduce((sum, o) => sum + o.total, 0);

// Find
const user = users.find(u => u.id === targetId);
const index = users.findIndex(u => u.id === targetId);

// Some/every
const hasAdmin = users.some(u => u.role === "admin");
const allActive = users.every(u => u.isActive);

// Chaining
const result = users
  .filter(u => u.isActive)
  .map(u => u.email)
  .sort();
```

## Testing (Jest)

```javascript
describe("UserService", () => {
  let repository;
  let service;

  beforeEach(() => {
    repository = { save: jest.fn(), find: jest.fn() };
    service = new UserService(repository);
  });

  describe("create", () => {
    it("creates user with valid data", async () => {
      repository.save.mockResolvedValue({ id: "1", email: "test@example.com" });

      const user = await service.create({ email: "test@example.com" });

      expect(user.email).toBe("test@example.com");
      expect(repository.save).toHaveBeenCalled();
    });

    it("throws on invalid email", async () => {
      await expect(service.create({ email: "" }))
        .rejects
        .toThrow(ValidationError);
    });
  });
});
```
