# Serverless Architecture

## Rules

- Keep functions small, single-purpose and stateless; store state in managed services.
- Initialize clients and connections outside the handler so warm invocations reuse them.
- Make handlers idempotent: triggers can deliver the same event more than once.
- Minimize package size and dependencies to reduce cold starts.
- Set explicit timeouts, memory sizes and least-privilege IAM roles per function.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Key Principles

- **Stateless functions**: Each invocation is independent
- **Event-driven**: Functions triggered by events
- **Auto-scaling**: Platform handles scaling
- **Pay-per-use**: Billed by execution

## Function Design

```typescript
// Handler pattern
export async function handler(
  event: APIGatewayEvent,
  context: Context
): Promise<APIGatewayProxyResult> {
  try {
    const body = JSON.parse(event.body || '{}');
    const result = await processOrder(body);

    return {
      statusCode: 200,
      body: JSON.stringify(result)
    };
  } catch (error) {
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal error' })
    };
  }
}
```

## Cold Start Optimization

```typescript
// Initialize outside handler (reused across invocations)
const dbPool = createPool(process.env.DATABASE_URL);

export async function handler(event: Event): Promise<Response> {
  // Use cached connection
  const result = await dbPool.query('SELECT * FROM orders');
  return { statusCode: 200, body: JSON.stringify(result) };
}
```

## State Management

```typescript
// Use external state stores
class OrderService {
  constructor(
    private dynamodb: DynamoDB,
    private redis: Redis
  ) {}

  async getOrder(id: string): Promise<Order> {
    // Check cache first
    const cached = await this.redis.get(`order:${id}`);
    if (cached) return JSON.parse(cached);

    // Fall back to database
    const result = await this.dynamodb.get({ Key: { id } });
    await this.redis.set(`order:${id}`, JSON.stringify(result));
    return result;
  }
}
```

## Best Practices

- Keep functions small and focused
- Use environment variables for configuration
- Minimize dependencies to reduce cold start time
- Handle timeouts gracefully
- Use async/await for all I/O operations
- Implement idempotency for event handlers
- Log structured data for observability
- Set appropriate memory and timeout limits
