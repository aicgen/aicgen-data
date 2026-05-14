# Microservices Data Management

## Database Per Service

```
Each service owns its database:

✅ Order Service → order_db (PostgreSQL)
✅ User Service → user_db (PostgreSQL)
✅ Catalog Service → catalog_db (MongoDB)
✅ Search Service → search_index (Elasticsearch)

❌ Never share databases between services
❌ Never query another service's tables directly
```

## Polyglot Persistence

```text
# Each service uses the best database for its needs

# User Service -> Relational (ACID, relationships)
Repository UserRepository:
  Method Create(user):
    SQL "INSERT INTO users..."

# Catalog Service -> Document (Flexible schema)
Repository ProductRepository:
  Method Create(product):
    Collection("products").Insert(product)

# Analytics Service -> Time-Series (High write volume)
Repository MetricsRepository:
  Method Record(metric):
    InfluxDB.Write(metric)
```

## Eventual Consistency

```text
# ✅ Embrace eventual consistency for cross-service data

1. Order Service: Save Order -> Publish "order.created"
2. Inventory Service: Listen "order.created" -> Reserve Inventory

# Data may be temporarily inconsistent - that's OK

# ✅ Use compensating actions for failures
Function ProcessOrder(order):
  Try:
    InventoryService.Reserve(order.items)
    PaymentService.Charge(order.total)
  Catch Error:
    # Compensate: undo previous actions
    InventoryService.Release(order.items)
    Throw Error
```

## Data Synchronization Patterns

```text
# Pattern: Event Sourcing / CQRS
Service OrderQuery:
  On("product.updated"):
    # Update local read-optimized copy
    Cache.Set(event.productId, { name: event.name, price: event.price })

# Pattern: Saga for distributed transactions
Saga CreateOrder:
  Step 1:
    Action: Inventory.Reserve()
    Compensate: Inventory.Release()
  Step 2:
    Action: Payment.Charge()
    Compensate: Payment.Refund()
```

## Data Ownership

```text
# ✅ Each service is the source of truth for its data
Service User:
  Function UpdateEmail(userId, email):
    Database.Update(userId, email)
    EventBus.Publish("user.email.changed", { userId, email })

# Other services maintain their own copies (projections)
Service Order:
  On("user.email.changed"):
    # Update local cache, never query User DB directly
    LocalUserCache.Update(event.userId, event.email)
```
