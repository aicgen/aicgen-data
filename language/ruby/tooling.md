# Ruby Tooling And Modern Ruby

## Baseline

- Pin Ruby with `.ruby-version` (and `ruby` in the `Gemfile`), commit `Gemfile.lock`, and install with `bundle install --frozen` (or `BUNDLE_FROZEN=true`) in CI.
- Add `# frozen_string_literal: true` to every file (RuboCop enforces it).
- Use keyword arguments for anything with more than two parameters, and `Struct`/`Data.define` for simple value objects.

```ruby
# frozen_string_literal: true

Money = Data.define(:amount_cents, :currency) do
  def +(other)
    raise ArgumentError, "currency mismatch" unless currency == other.currency

    with(amount_cents: amount_cents + other.amount_cents)
  end
end

def describe(value)
  case value
  in Integer => n if n.positive? then "positive #{n}"
  in { name: String => name, age: Integer => age } then "#{name} (#{age})"
  in [first, *] then "list starting with #{first}"
  else "unknown"
  end
end

puts describe({ name: "Ada", age: 36 })
puts Money.new(100, "USD") + Money.new(50, "USD")
```

## The Required Gate

```bash
bundle exec rubocop
bundle exec rspec          # or: bin/rails test
bundle exec bundler-audit check --update
bundle exec brakeman -q    # Rails applications
```

| Concern | Tool |
|---------|------|
| Style and lint | RuboCop (with `rubocop-rspec`, `rubocop-rails`, `rubocop-performance`) |
| Tests | RSpec or Minitest; `factory_bot`, `webmock`/`vcr` for HTTP |
| Types (optional) | Sorbet or RBS + Steep for libraries and critical modules |
| Security | `bundler-audit`, Brakeman, strong parameters |

## Practices

- Prefer small objects with explicit collaborators over fat models; keep controllers thin and put business rules in plain Ruby objects (services/commands) that are easy to test.
- Raise specific exception classes inheriting from `StandardError`; `rescue` the narrowest class; never `rescue Exception`; re-raise with context using `raise NewError, "msg", cause: e` implicitly via `rescue`.
- Use `Comparable`, `Enumerable` and `each_with_object`/`group_by`/`tally` instead of manual loops. Use safe navigation (`&.`) sparingly; fix the nil source instead.
- Avoid monkey patching; use refinements or wrappers. Avoid `eval`, `send` with user input, and `Marshal.load`/`YAML.load` on untrusted data (use `YAML.safe_load`).
- Rails: strong parameters, parameterised queries (`where(email: email)`, never string interpolation), `has_secure_password`, `config.force_ssl`, N+1 detection with `bullet`, background work in jobs with idempotent handlers.
- Log with tagged, structured output; never log params containing secrets (`filter_parameters`).

## Testing

- One behaviour per example; use `let`/`subject` lazily; prefer real objects and fakes over stubbing everything; use `travel_to` for time.
- Keep the suite deterministic and parallelisable; fix flaky tests instead of retrying.

## Quick Checklist

- [ ] Ruby version and `Gemfile.lock` committed; CI installs frozen
- [ ] RuboCop, tests, `bundler-audit` (and Brakeman for Rails) run in CI
- [ ] No `eval`/unsafe deserialisation; strong parameters and parameterised queries
