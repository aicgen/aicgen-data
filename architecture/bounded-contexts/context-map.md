# Bounded Contexts

## Rules

- Give each bounded context one team, one model and one clearly owned data store; do not share tables across contexts.
- Integrate through published contracts (APIs or events), never by reaching into another context's internals.
- Choose the relationship pattern deliberately: partnership, shared kernel (small), customer/supplier, conformist, or anticorruption layer.
- Use an anticorruption layer to translate foreign models at the boundary.
- Keep a context map document up to date.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Core Principle

Divide the domain into logical boundaries where a specific model applies. Each context has its own ubiquitous language, data model, and rules.

## Context Mapping Patterns

Relates different bounded contexts to each other.

### Partnership

Two contexts succeed or fail together. Teams work closely to align interfaces.

### Shared Kernel

Sharing a subset of the domain model (code/database) between contexts. High coupling, use sparingly (e.g., for generic Auth/IDs).

### Customer/Supplier

Downstream context (Customer) depends on Upstream context (Supplier). Upstream must negotiate changes with Downstream.

### Conformist

Downstream blindly conforms to Upstream's model without negotiation.

### Anticorruption Layer (ACL)

Downstream isolates itself from Upstream's model by translating it into its own internal model.

## Implementation Structure

```typescript
// Context: Sales
namespace Sales {
  export class Order { ... } // Sales-specific Order model
}

// Context: Shipping
namespace Shipping {
  export class Shipment { ... }
  // Shipping might interact with Sales via ACL or events
}
```

## Best Practices

- **Explicit Boundaries**: Code for one context should not bleed into another.
- **Ubiquitous Language**: Use the same terminology in code as functionality.
- **Decoupled Deployment**: Ideally, contexts can be deployed independently (microservices or modular monoliths).
