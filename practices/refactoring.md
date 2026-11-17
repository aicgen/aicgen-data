# Refactoring Patterns

## Rules

- Refactor only with passing tests; add characterization tests first if there are none.
- Change behavior or structure, never both in one commit; make small steps and run the tests after each.
- Target real smells: long methods, duplication, primitive obsession, feature envy; leave code that works and is rarely touched.
- Keep refactoring commits separate from feature work so they are easy to review and revert.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Common Code Smells

### Long Method
Split into smaller, focused functions.

```typescript
// Before
function processOrder(order: Order) {
  // 100 lines of code...
}

// After
function processOrder(order: Order) {
  validateOrder(order);
  calculateTotals(order);
  applyDiscounts(order);
  saveOrder(order);
}
```

### Duplicate Code
Extract common logic.

```typescript
// Before
function getAdminUsers() {
  return users.filter(u => u.role === 'admin' && u.active);
}
function getModeratorUsers() {
  return users.filter(u => u.role === 'moderator' && u.active);
}

// After
function getActiveUsersByRole(role: string) {
  return users.filter(u => u.role === role && u.active);
}
```

### Primitive Obsession
Use value objects.

```typescript
// Before
function sendEmail(email: string) { /* ... */ }

// After
class Email {
  constructor(private value: string) {
    if (!this.isValid(value)) throw new Error('Invalid email');
  }
}
function sendEmail(email: Email) { /* ... */ }
```

### Feature Envy
Move method to class it uses most.

```typescript
// Before - Order is accessing customer too much
class Order {
  getDiscount() {
    return this.customer.isPremium() ?
      this.customer.premiumDiscount :
      this.customer.regularDiscount;
  }
}

// After
class Customer {
  getDiscount(): number {
    return this.isPremium() ? this.premiumDiscount : this.regularDiscount;
  }
}
```

## Safe Refactoring Steps

1. Ensure tests pass before refactoring
2. Make one small change at a time
3. Run tests after each change
4. Commit frequently
5. Refactor in separate commits from feature work

## Best Practices

- Refactor when adding features, not separately
- Keep refactoring commits separate
- Use IDE refactoring tools when available
- Write tests before refactoring if missing
