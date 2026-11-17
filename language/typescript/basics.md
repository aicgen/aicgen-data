# TypeScript Fundamentals

## Rules

- Enable `strict: true` (plus `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` for new projects); never turn strictness off to silence an error.
- Never use `any`; use `unknown` and narrow with type guards, or a generic.
- Annotate function parameters and public return types; let inference handle locals.
- Validate external data (HTTP, JSON, env) at runtime with a schema library; types alone do not check it.
- Naming: `PascalCase` for types and classes (no `I` prefix), `camelCase` for values, `UPPER_SNAKE_CASE` for constants, `kebab-case` files.
- Avoid non-null assertions (`!`) and `as` casts; fix the types instead.

## Strict Mode (Required)

Always use strict mode in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true
  }
}
```

## Type Annotations

Use explicit types for clarity:

```typescript
// Function signatures
function calculateTotal(items: CartItem[], taxRate: number): number {
  const subtotal = items.reduce((sum, item) => sum + item.price, 0);
  return subtotal * (1 + taxRate);
}

// Variable declarations
const userName: string = "Alice";
const age: number = 30;
const isActive: boolean = true;
```

## Avoid `any`

Never use `any` - use `unknown` with type guards:

```typescript
// ❌ Bad
function processData(data: any) {
  return data.value;
}

// ✅ Good
function processData(data: unknown): string {
  if (typeof data === 'object' && data !== null && 'value' in data) {
    return String(data.value);
  }
  throw new Error('Invalid data structure');
}
```

## Type Guards

Implement custom type guards:

```typescript
interface User {
  id: string;
  email: string;
}

function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'email' in value &&
    typeof value.id === 'string' &&
    typeof value.email === 'string'
  );
}

// Usage
if (isUser(data)) {
  console.log(data.email); // Type: User
}
```

## Naming Conventions

- Classes/Interfaces: `PascalCase`
- Functions/Variables: `camelCase`
- Constants: `UPPER_SNAKE_CASE`
- Files: `kebab-case.ts`
- No `I` prefix for interfaces
