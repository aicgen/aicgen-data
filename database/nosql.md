# NoSQL Database Patterns

## Document Databases (MongoDB, CouchDB)

### Schema Design

```typescript
// Embed vs Reference

// ✅ Embed: Data accessed together, 1:few relationship
interface Order {
  _id: ObjectId;
  customerId: ObjectId;
  items: Array<{           // Embedded
    productId: ObjectId;
    name: string;
    quantity: number;
    price: number;
  }>;
  shippingAddress: {       // Embedded
    street: string;
    city: string;
    country: string;
  };
}

// ✅ Reference: Data accessed separately, 1:many relationship
interface Post {
  _id: ObjectId;
  title: string;
  content: string;
  authorId: ObjectId;      // Reference to User
}

interface Comment {
  _id: ObjectId;
  postId: ObjectId;        // Reference to Post
  authorId: ObjectId;      // Reference to User
  content: string;
}
```

### Indexing

```typescript
// Single field index
db.users.createIndex({ email: 1 });

// Compound index (order matters!)
db.orders.createIndex({ customerId: 1, createdAt: -1 });

// Partial index
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { status: "active" } }
);

// Text search index
db.products.createIndex({ name: "text", description: "text" });
```

### Query Patterns

```typescript
// Avoid N+1 queries - use $lookup for joins
const ordersWithCustomers = await db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: "$customer" }
]).toArray();

// Use projections to limit returned fields
const userNames = await db.users.find(
  { status: "active" },
  { projection: { name: 1, email: 1 } }
).toArray();
```

## Key-Value Stores (Redis, DynamoDB)

### Redis Patterns

```typescript
// Caching with TTL
await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));

// Rate limiting
async function isRateLimited(userId: string): Promise<boolean> {
  const key = `rate:${userId}`;
  const count = await redis.incr(key);
  if (count === 1) {
    await redis.expire(key, 60); // 60 second window
  }
  return count > 100; // 100 requests per minute
}

// Distributed locks
async function acquireLock(key: string, ttl: number): Promise<boolean> {
  const result = await redis.set(`lock:${key}`, "1", "NX", "EX", ttl);
  return result === "OK";
}

// Pub/Sub
await redis.publish("events", JSON.stringify({ type: "user.created", data: user }));
```

### DynamoDB Patterns

```typescript
// Single table design - access patterns first
interface Item {
  PK: string;           // Partition key: USER#<userId>
  SK: string;           // Sort key: ORDER#<orderId>
  GSI1PK?: string;      // Global Secondary Index
  GSI1SK?: string;
  // ... attributes
}

// Write with condition
await dynamodb.put({
  TableName: "orders",
  Item: order,
  ConditionExpression: "attribute_not_exists(PK)"  // Prevent duplicates
});

// Query with begins_with
const userOrders = await dynamodb.query({
  TableName: "orders",
  KeyConditionExpression: "PK = :pk AND begins_with(SK, :sk)",
  ExpressionAttributeValues: {
    ":pk": `USER#${userId}`,
    ":sk": "ORDER#"
  }
});
```

## Wide Column Stores (Cassandra)

### Data Modeling

```sql
-- Design for queries, not relationships
-- One table per query pattern

CREATE TABLE orders_by_customer (
  customer_id UUID,
  order_date TIMESTAMP,
  order_id UUID,
  total DECIMAL,
  PRIMARY KEY ((customer_id), order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC);

CREATE TABLE orders_by_date (
  order_date DATE,
  order_id UUID,
  customer_id UUID,
  total DECIMAL,
  PRIMARY KEY ((order_date), order_id)
);
```

## Best Practices

### Denormalization
- Duplicate data for read performance
- Design for query patterns, not normalization
- Accept eventual consistency

### Data Modeling
- Start with access patterns
- One table per query (or single-table design for DynamoDB)
- Embed related data that's always accessed together
- Reference data that's accessed independently

### Performance
- Use indexes strategically
- Batch operations when possible
- Use TTL for temporary data
- Consider sharding/partitioning for scale

### Consistency
- Understand your consistency model (eventual vs strong)
- Use transactions when available and necessary
- Design for idempotency

```typescript
// Idempotent writes with version/timestamp
async function updateUser(user: User): Promise<void> {
  await db.users.updateOne(
    { _id: user._id, version: user.version },
    { $set: { ...user, version: user.version + 1 } }
  );
}
```

### Error Handling

```typescript
// Retry with exponential backoff
async function withRetry<T>(
  operation: () => Promise<T>,
  maxRetries = 3
): Promise<T> {
  let lastError: Error;

  for (let i = 0; i < maxRetries; i++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error as Error;
      await sleep(Math.pow(2, i) * 100);
    }
  }

  throw lastError!;
}
```
