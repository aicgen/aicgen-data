# Microservices Communication

## Synchronous vs Asynchronous

```text
# ⚠️ Synchronous creates coupling and multiplicative downtime
# If Service A calls B calls C, any failure breaks the chain

# ✅ Prefer asynchronous messaging for most inter-service communication
# Limit synchronous calls to one per user request

# Async Message Format Example
Event: "order.created"
Data: {
  orderId: "ord_123",
  userId: "user_456",
  items: [...]
}

# Subscribers process independently
Service Inventory -> ReserveItems(items)
Service Notification -> SendEmail(user)
```

## API-Based Communication

```text
# ✅ Well-defined REST APIs between services
GET /users/{userId}

# ✅ Use circuit breaker for resilience
Function getUserSafe(userId):
  Try:
    return UserClient.getUser(userId)
  Catch Error:
    return getCachedUser(userId) # Fallback
```

## Event-Driven Integration

```text
# ✅ Publish events for state changes
Function CreateOrder(data):
  order = Repository.Save(data)
  
  # Failures here don't block the user
  EventBus.Publish("order.created", {
    orderId: order.id,
    userId: order.userId,
    timestamp: Now()
  })

  return order

# ✅ Consumers handle events independently (Decoupled)
Service Notification:
  On("order.created"): SendConfirmation(event)

Service Inventory:
  On("order.created"): ReserveInventory(event)
```

## Tolerant Reader Pattern

```text
# ✅ Don't fail on unknown fields - enables independent evolution
Structure UserResponse:
  id: string
  name: string
  ...ignore other fields...

# ✅ Use sensible defaults for missing optional fields
Function ParseUser(data):
  return User {
    id: data.id,
    name: data.name,
    role: data.role OR 'user',   # Default
    avatar: data.avatar OR null
  }
```

## Anti-Patterns

```text
# ❌ Chatty communication (N+1 service calls)
For each orderId in orderIds:
  GetOrder(orderId)  # N network calls!

# ✅ Batch requests
GetOrders(orderIds)  # 1 network call

# ❌ Tight coupling via shared databases
# Service A directly queries Service B's tables

# ✅ API-based communication
UserClient.GetUsers(userIds)
```
