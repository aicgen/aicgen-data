# DDD Strategic Patterns

## Rules

- Build a ubiquitous language with domain experts and use the same terms in code, tests and docs.
- Split the domain into bounded contexts, each with its own model; the same word may mean different things in different contexts.
- Make relationships between contexts explicit with a context map (shared kernel, customer/supplier, conformist, anticorruption layer, open host service).
- Protect your model with an anticorruption layer when integrating with legacy or external models.
- Invest most in the core domain; buy or simplify supporting and generic subdomains.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Ubiquitous Language

Use the same terminology in code, documentation, and conversations.

```typescript
// Domain experts say "place an order"
class Order {
  place(): void { /* not submit(), not create() */ }
}

// Domain experts say "items are added to cart"
class ShoppingCart {
  addItem(product: Product): void { /* not insert(), not push() */ }
}
```

## Bounded Contexts

Explicit boundaries where a model applies consistently.

```
┌─────────────────┐    ┌─────────────────┐
│    Sales        │    │   Warehouse     │
│    Context      │    │    Context      │
├─────────────────┤    ├─────────────────┤
│ Order           │    │ Order           │
│ - customerId    │    │ - shipmentId    │
│ - items[]       │    │ - pickingList   │
│ - total         │    │ - status        │
└─────────────────┘    └─────────────────┘
   Same term, different model
```

## Context Mapping Patterns

### Shared Kernel
Two contexts share a subset of the model.

### Customer/Supplier
Upstream context provides what downstream needs.

### Conformist
Downstream adopts upstream's model entirely.

### Anti-Corruption Layer
Translate between contexts to protect domain model.

```typescript
class InventoryAntiCorruptionLayer {
  constructor(private legacyInventorySystem: LegacyInventory) {}

  checkAvailability(productId: ProductId): Promise<boolean> {
    // Translate from legacy format to domain model
    const legacyResult = await this.legacyInventorySystem.getStock(
      productId.toString()
    );
    return legacyResult.qty > 0;
  }
}
```

## Module Organization

```
src/
├── sales/                    # Sales bounded context
│   ├── domain/
│   │   ├── order.ts
│   │   └── customer.ts
│   ├── application/
│   │   └── place-order.ts
│   └── infrastructure/
│       └── order-repository.ts
├── warehouse/                # Warehouse bounded context
│   ├── domain/
│   │   └── shipment.ts
│   └── ...
└── shared/                   # Shared kernel
    └── money.ts
```

## Best Practices

- Define context boundaries based on team structure and business capabilities
- Use ubiquitous language within each context
- Communicate between contexts via events or explicit APIs
- Protect domain model with anti-corruption layers when integrating legacy systems
