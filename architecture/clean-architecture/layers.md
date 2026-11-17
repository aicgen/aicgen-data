# Clean Architecture

## Rules

- Dependencies point inward only: entities, then use cases, then interface adapters, then frameworks and drivers. Inner code never imports outer code.
- Keep entities and use cases free of framework, database and HTTP types.
- Define interfaces (ports, repositories) in the inner layer and implement them in the outer layer.
- Cross boundaries with simple data structures (DTOs), never with entities or framework objects.
- Controllers translate input into a use-case call and format the output; they contain no business rules.
- Test use cases with in-memory fakes, without frameworks or a database.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Core Principle

Dependencies point inward. Inner layers know nothing about outer layers.

```
┌─────────────────────────────────────────────┐
│            Frameworks & Drivers             │
│  ┌─────────────────────────────────────┐    │
│  │       Interface Adapters            │    │
│  │  ┌─────────────────────────────┐    │    │
│  │  │     Application Business    │    │    │
│  │  │  ┌─────────────────────┐    │    │    │
│  │  │  │  Enterprise Business│    │    │    │
│  │  │  │     (Entities)      │    │    │    │
│  │  │  └─────────────────────┘    │    │    │
│  │  │       (Use Cases)           │    │    │
│  │  └─────────────────────────────┘    │    │
│  │    (Controllers, Gateways)          │    │
│  └─────────────────────────────────────┘    │
│      (Web, DB, External APIs)               │
└─────────────────────────────────────────────┘
```

## The Dependency Rule

Source code dependencies only point inward.

## Layer Structure

### Entities (Enterprise Business Rules)

```typescript
class Order {
  constructor(
    public readonly id: string,
    private items: OrderItem[],
    private status: OrderStatus
  ) {}

  calculateTotal(): Money {
    return this.items.reduce(
      (sum, item) => sum.add(item.subtotal()),
      Money.zero()
    );
  }

  canBeCancelled(): boolean {
    return this.status === OrderStatus.Pending;
  }
}
```

### Use Cases (Application Business Rules)

```typescript
class CreateOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private productRepository: ProductRepository
  ) {}

  async execute(request: CreateOrderRequest): Promise<CreateOrderResponse> {
    const products = await this.productRepository.findByIds(request.productIds);
    const order = new Order(generateId(), this.createItems(products));
    await this.orderRepository.save(order);
    return { orderId: order.id };
  }
}
```

### Interface Adapters

```typescript
// Controller (adapts HTTP to use case)
class OrderController {
  constructor(private createOrder: CreateOrderUseCase) {}

  async create(req: Request, res: Response) {
    const result = await this.createOrder.execute(req.body);
    res.json(result);
  }
}

// Repository Implementation (adapts use case to database)
class PostgreSQLOrderRepository implements OrderRepository {
  async save(order: Order): Promise<void> {
    await this.db.query('INSERT INTO orders...');
  }
}
```

### Frameworks & Drivers

```typescript
// Express setup
const app = express();
app.post('/orders', (req, res) => orderController.create(req, res));

// Database connection
const db = new Pool({ connectionString: process.env.DATABASE_URL });
```

## Best Practices

- Keep entities pure with no framework dependencies
- Use cases orchestrate domain logic
- Interfaces defined in inner layers, implemented in outer layers
- Cross boundaries with simple data structures
- Test use cases independently of frameworks
