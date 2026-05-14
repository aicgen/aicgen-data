# PHP OOP

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
