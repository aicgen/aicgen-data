# Service Boundaries

## Defining Service Boundaries

Each service should own a specific business capability:

```
✅ Good Boundaries:
- User Service: Authentication, profiles, preferences
- Order Service: Order processing, fulfillment
- Payment Service: Payment processing, billing
- Notification Service: Emails, SMS, push notifications

❌ Bad Boundaries:
- Data Access Service (technical, not business)
- Utility Service (too generic)
- God Service (does everything)
```

## Bounded Contexts

Use Domain-Driven Design to identify boundaries:

- Each service represents a bounded context
- Services are organized around business domains
- Clear ownership of data and logic
- Services should be independently deployable

## Ownership Rules

**Each service:**
- Owns its own database (no shared databases)
- Owns its domain logic
- Exposes well-defined APIs
- Can be developed by autonomous teams

## Communication Rules

**Avoid:**
- Direct database access between services
- Chatty communication (N+1 service calls)
- Tight coupling through shared libraries

**Prefer:**
- API-based communication
- Event-driven for data synchronization
- Async messaging where possible

## Data Ownership

```text
// ✅ Good - Service owns its data
Class OrderService:
  Method CreateOrder(data):
    # Order service owns order data
    Order = OrderRepository.Save(data)

    # Publish event for other services
    EventBus.Publish("order.created", {
      orderId: Order.id,
      userId: Order.userId,
      total: Order.total
    })

    Return Order

// ❌ Bad - Direct access to another service's database
Class OrderService:
  Method CreateOrder(data):
    # Don't do this!
    User = UserDatabase.FindOne({ id: data.userId })
```

## Sizing Guidelines

Keep services:
- Small enough to be maintained by a small team (2-3 developers)
- Large enough to provide business value
- Focused on a single bounded context
- Independently deployable and scalable
