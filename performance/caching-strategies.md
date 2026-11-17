# Cache Patterns And Invalidation

Which caching pattern to use and how to keep a cache consistent. For in-memory, Redis and memoization code, see `caching.md`.

## Rules

- Default to cache-aside (read through the cache, populate on miss); use write-through when reads must see fresh writes, and write-behind only when you can tolerate losing buffered writes.
- Pick an invalidation strategy up front: TTL for tolerable staleness, event-based deletion on writes for freshness, never both missing.
- Cache at the right layer (CDN, application, database) and key entries deterministically with a version or namespace.
- Protect against stampedes (request coalescing, jittered TTLs) and treat cache failure as a miss, not an outage.
- Never cache sensitive or per-user data under shared keys, and monitor hit rate and eviction.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Cache Patterns

### Cache-Aside (Lazy Loading)
```typescript
async function getUser(id: string): Promise<User> {
  const cached = await cache.get(`user:${id}`);
  if (cached) return cached;

  const user = await db.findUser(id);
  await cache.set(`user:${id}`, user, { ttl: 3600 });
  return user;
}
```

### Write-Through
```typescript
async function updateUser(user: User): Promise<void> {
  await db.saveUser(user);
  await cache.set(`user:${user.id}`, user);
}
```

### Write-Behind (Write-Back)
```typescript
async function updateUser(user: User): Promise<void> {
  await cache.set(`user:${user.id}`, user);
  await queue.add('sync-to-db', { user }); // Async persistence
}
```

## Cache Invalidation

```typescript
// Time-based expiration
await cache.set('key', value, { ttl: 3600 });

// Event-based invalidation
eventBus.on('user.updated', async (userId) => {
  await cache.delete(`user:${userId}`);
});

// Pattern-based invalidation
await cache.deleteByPattern('user:*');

// Cache warming for known hot keys
for (const product of await db.getMostViewed(100)) {
  await cache.set(`product:${product.id}`, product, { ttl: 3600 });
}
```

## HTTP Caching

```typescript
// Cache-Control headers
res.setHeader('Cache-Control', 'public, max-age=3600');
res.setHeader('ETag', etag(content));

// Conditional requests
if (req.headers['if-none-match'] === currentEtag) {
  return res.status(304).end();
}
```

## Best Practices

- Cache at appropriate layers (CDN, app, DB)
- Use consistent cache keys
- Set appropriate TTLs
- Monitor cache hit rates
- Plan for cache failures (fallback to source)
- Avoid caching sensitive data
