# DevOps Practices

## Infrastructure as Code

```text
# Define infrastructure as code (e.g., Terraform, Pulumi)

Resource S3Bucket "app-bucket":
  Access: Private
  Versioning: Enabled

Resource LambdaFunction "api":
  Runtime: Node.js / Python / Go
  Handler: index.handler
  Code: ./dist
```

## Containerization

```dockerfile
# Build Stage
FROM base-image AS builder
WORKDIR /app
COPY dependency-files ./
RUN install-dependencies
COPY source-code .
RUN build-application

# Runtime Stage
FROM runtime-image
WORKDIR /app
COPY --from=builder /app/dist ./dist
EXPOSE 8080
CMD ["run", "application"]
```

## Observability

### Logging
```json
{
  "level": "info",
  "message": "Order created",
  "orderId": "ord_123",
  "customerId": "cust_456",
  "total": 99.99,
  "timestamp": "2023-10-27T10:00:00Z"
}
```

### Metrics
```text
Metrics.Increment("orders.created")
Metrics.Histogram("order.value", total)
Metrics.Timing("order.processing_time", duration)
```

### Health Checks
```text
GET /health
Response 200 OK:
{
  "status": "healthy",
  "version": "1.2.3",
  "uptime": 3600
}
```

## Best Practices

- Version control all infrastructure
- Use immutable infrastructure
- Implement proper secret management
- Set up alerting for critical metrics
- Use structured logging (JSON)
- Include correlation IDs for tracing
- Document runbooks for incidents
