# PHP Basics

## Rules

- Start every file with `declare(strict_types=1);` and target a supported PHP version (8.2 or newer).
- Declare parameter, return and property types; avoid `mixed`, and use union, nullable and `never` types where they fit.
- Follow PSR-12 and PSR-4 autoloading, and run PHP-CS-Fixer or Pint and PHPStan.
- Use exceptions for errors, never suppress with `@`, and never trust input: use prepared statements and escape output.

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
