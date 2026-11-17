# PHP Tooling And Modern PHP

## Baseline

- Target a supported PHP version (declare it in `composer.json` `require.php`) and start every file with `declare(strict_types=1);`.
- Commit `composer.lock`; install with `composer install --no-interaction --prefer-dist` in CI and run `composer validate --strict` and `composer audit`.
- Use PSR-4 autoloading, PSR-12 style and PSR-3/7/11/15 interfaces so libraries stay interchangeable.

```json
{
  "require": { "php": "^8.3" },
  "require-dev": {
    "phpstan/phpstan": "^2.0",
    "phpunit/phpunit": "^11.0",
    "laravel/pint": "^1.0"
  },
  "autoload": { "psr-4": { "App\\": "src/" } },
  "config": { "sort-packages": true, "allow-plugins": {} },
  "scripts": {
    "lint": "pint --test",
    "stan": "phpstan analyse --level=max src tests",
    "test": "phpunit"
  }
}
```

## The Required Gate

```bash
composer validate --strict
composer audit
composer lint && composer stan && composer test
```

| Concern | Tool |
|---------|------|
| Style | Laravel Pint or PHP-CS-Fixer (PSR-12 plus rules) |
| Static analysis | PHPStan (level max for new code; baseline legacy) or Psalm |
| Tests | PHPUnit or Pest; Mockery sparingly; real database via containers |
| Security | `composer audit`, framework security advisories |

## Modern PHP

```php
<?php

declare(strict_types=1);

enum Status: string
{
    case Active = 'active';
    case Suspended = 'suspended';

    public function label(): string
    {
        return match ($this) {
            self::Active => 'Active',
            self::Suspended => 'Suspended',
        };
    }
}

final readonly class User
{
    public function __construct(
        public int $id,
        public string $email,
        public Status $status = Status::Active,
    ) {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException("invalid email: {$email}");
        }
    }
}

interface UserRepository
{
    public function find(int $id): ?User;
}

function describe(?User $user): string
{
    return $user?->status->label() ?? 'unknown';
}

echo describe(new User(1, 'ada@example.com')), PHP_EOL;
```

- Use **enums**, **readonly** classes/properties, constructor promotion, `match`, union types and named arguments. Type every parameter, return and property.
- Prefer composition and small interfaces; inject dependencies through the constructor (no service locators, no static state).
- Throw specific exceptions; never use the `@` operator; convert warnings to exceptions with a global error handler.
- Use prepared statements (PDO with bound parameters) for every query; escape output for its context (`htmlspecialchars`); hash passwords with `password_hash`/`password_verify`; generate tokens with `random_bytes`.
- Never `unserialize()` untrusted data; validate uploads by content, not extension; disable `display_errors` in production.

## Laravel / Symfony Notes

- Validate with Form Requests (Laravel) or the Validator (Symfony); authorise with policies/voters; use Eloquent/Doctrine parameter binding; keep controllers thin.
- Queue slow work, make jobs idempotent, and configure via environment and cached config (never read `env()` outside config files in Laravel).

## Quick Checklist

- [ ] `strict_types` everywhere; PHPStan at max level for new code
- [ ] `composer.lock` committed; `composer audit` runs in CI
- [ ] Prepared statements, output escaping and `password_hash` used
