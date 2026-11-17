# PHP OOP

## Rules

- Depend on interfaces and inject dependencies through the constructor; avoid static state and service locators.
- Use `readonly` properties and value objects for immutability, and `final` classes by default.
- Prefer composition and small interfaces; use traits sparingly and never for shared state.
- Use backed enums instead of string or integer constants.

## Classes and properties

Use typed properties and visibility modifiers.

```php
class User {
    public function __construct(
        private string $name,
        public readonly int $id
    ) {}
}
```

## Interfaces

Define contracts using interfaces.

```php
interface Logger {
    public function log(string $message): void;
}
```

## Traits

Use traits for horizontal code reuse, but use sparingly.

```php
trait Loggable {
    private function log(string $msg) { ... }
}
```

## Enums

Use native Enums (PHP 8.1+) instead of class constants.

```php
enum Status: string {
    case Draft = 'draft';
    case Published = 'published';
}
```
