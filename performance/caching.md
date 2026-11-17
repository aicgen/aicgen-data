# Cache Implementations

Concrete in-memory, Redis and memoization code. For which pattern to use (cache-aside, write-through) and how to invalidate, see `caching-strategies.md`.

## Rules

- Every cache entry has a TTL; never cache without an expiry or an invalidation path.
- Namespace keys (`user:{id}`) and keep them deterministic.
- Treat the cache as optional: a miss or an outage must fall back to the source of truth.
- Never cache secrets or per-user data under a shared key.


## In-Memory Caching

```typescript
class Cache<T> {
  private cache = new Map<string, { value: T; expiry: number }>();

  set(key: string, value: T, ttlMs: number = 60000): void {
    this.cache.set(key, {
      value,
      expiry: Date.now() + ttlMs
    });
  }

  get(key: string): T | null {
    const item = this.cache.get(key);
    if (!item) return null;

    if (Date.now() > item.expiry) {
      this.cache.delete(key);
      return null;
    }

    return item.value;
  }
}

// Usage
const userCache = new Cache<User>();

async function getUser(id: string): Promise<User> {
  const cached = userCache.get(id);
  if (cached) return cached;

  const user = await db.findUser(id);
  userCache.set(id, user, 300000); // 5 minutes
  return user;
}
```

## Redis Caching

```typescript
import Redis from 'ioredis';

const redis = new Redis();

async function getCachedUser(id: string): Promise<User | null> {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await db.findUser(id);
  await redis.setex(`user:${id}`, 300, JSON.stringify(user)); // 5 min TTL
  return user;
}

// Cache invalidation on update
async function updateUser(id: string, data: Partial<User>) {
  const user = await db.updateUser(id, data);
  await redis.del(`user:${id}`); // Invalidate cache
  return user;
}
```

## Memoization

```typescript
const memoize = <T extends (...args: any[]) => any>(fn: T): T => {
  const cache = new Map<string, ReturnType<T>>();

  return ((...args: any[]) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);

    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
};

// Cache expensive computations
const calculateDiscount = memoize((price: number, tier: string) => {
  // Complex calculation
  return result;
});
```
