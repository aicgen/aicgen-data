# Modern PHP

## Attributes

Use Attributes (PHP 8+) instead of PHPDoc annotations.

```php
#[Route('/api/users', methods: ['GET'])]
public function list(): Response { ... }
```

## Constructor Property Promotion

Reduce boilerplate by defining properties in the constructor.

```php
class Point {
    public function __construct(
        public float $x = 0.0,
        public float $y = 0.0,
    ) {}
}
```

## Match Expression

Use `match` instead of `switch`. It is strict, returns a value, and doesn't fall through.

```php
$message = match ($statusCode) {
    200, 300 => null,
    400 => 'not found',
    500 => 'server error',
    default => 'unknown status code',
};
```
