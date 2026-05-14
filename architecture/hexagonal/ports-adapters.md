# Hexagonal Architecture (Ports & Adapters)

## Core Principle

The application core (domain logic) is isolated from external concerns through ports (interfaces) and adapters (implementations).

## Structure

```
src/
├── domain/           # Pure business logic, no external dependencies
│   ├── models/       # Domain entities and value objects
│   ├── services/     # Domain services
│   └── ports/        # Interface definitions (driven & driving)
├── application/      # Use cases, orchestration
│   └── services/     # Application services
├── adapters/
│   ├── primary/      # Driving adapters (controllers, CLI, events)
│   │   ├── http/
│   │   ├── grpc/
│   │   └── cli/
│   └── secondary/    # Driven adapters (repositories, clients)
│       ├── persistence/
│       ├── messaging/
│       └── external-apis/
└── config/           # Dependency injection, configuration
```

## Port Types

### Driving Ports (Primary)
Interfaces that the application exposes to the outside world:

```typescript
// domain/ports/driving/user-service.port.ts
export interface UserServicePort {
  createUser(data: CreateUserDTO): Promise<User>;
  getUser(id: string): Promise<User | null>;
  updateUser(id: string, data: UpdateUserDTO): Promise<User>;
}
```

### Driven Ports (Secondary)
Interfaces that the application needs from the outside world:

```typescript
// domain/ports/driven/user-repository.port.ts
export interface UserRepositoryPort {
  save(user: User): Promise<void>;
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
}

// domain/ports/driven/email-sender.port.ts
export interface EmailSenderPort {
  send(to: string, subject: string, body: string): Promise<void>;
}
```

## Adapter Implementation

### Primary Adapter (HTTP Controller)

```typescript
// adapters/primary/http/user.controller.ts
export class UserController {
  constructor(private userService: UserServicePort) {}

  async create(req: Request, res: Response) {
    const user = await this.userService.createUser(req.body);
    res.status(201).json(user);
  }
}
```

### Secondary Adapter (Repository)

```typescript
// adapters/secondary/persistence/postgres-user.repository.ts
export class PostgresUserRepository implements UserRepositoryPort {
  constructor(private db: DatabaseConnection) {}

  async save(user: User): Promise<void> {
    await this.db.query('INSERT INTO users...', user);
  }

  async findById(id: string): Promise<User | null> {
    const row = await this.db.query('SELECT * FROM users WHERE id = $1', [id]);
    return row ? this.toDomain(row) : null;
  }
}
```

## Dependency Rule

Dependencies always point inward:
- Adapters depend on Ports
- Application depends on Domain
- Domain has no external dependencies

```
[External World] → [Adapters] → [Ports] → [Domain]
```

## Testing Benefits

```typescript
// Test with mock adapters
class InMemoryUserRepository implements UserRepositoryPort {
  private users = new Map<string, User>();

  async save(user: User) { this.users.set(user.id, user); }
  async findById(id: string) { return this.users.get(id) || null; }
}

// Domain logic tested without infrastructure
describe('UserService', () => {
  it('creates user', async () => {
    const repo = new InMemoryUserRepository();
    const service = new UserService(repo);
    const user = await service.createUser({ name: 'Test' });
    expect(user.name).toBe('Test');
  });
});
```

## When to Use

- Applications needing multiple entry points (HTTP, CLI, events)
- Systems requiring easy infrastructure swapping
- Projects prioritizing testability
- Long-lived applications expecting technology changes
