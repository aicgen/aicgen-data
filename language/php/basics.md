# PHP Basics

## Strict Types

Always enable strict typing in new files to prevent unexpected type coercion.

```php
declare(strict_types=1);
```

## Type Declarations

Always specify argument and return types.

```php
function add(int $a, int $b): int {
    return $a + $b;
}
```

## Union and Intersection Types

Use robust type definitions.

```php
function process(Input|string $data): Result&Serializable {
    // ...
}
```

## Nullable Types

Use `?Type` for nullable arguments or return values.

```php
function find(?int $id): ?User {
    return $id ? User::find($id) : null;
}
```
