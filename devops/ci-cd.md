# CI/CD Practices

## Rules

- Run CI on every push and pull request: install from the lockfile, lint, type-check, test, build. Fail the build on any of them.
- Order jobs fast to slow, cache dependencies, and use a matrix for versions and platforms.
- Set `permissions: contents: read` by default and grant more per job; pin third-party actions by commit SHA and use OIDC instead of long-lived cloud keys.
- Deploy from CI only after checks pass, to a staging environment first, with a documented rollback.
- Use a progressive strategy (blue-green, canary or rolling) with health checks and automatic rollback; hide unfinished work behind feature flags.
- Keep secrets in the platform secret store and automate database migrations safely (backward-compatible, run before the new code).

## Continuous Integration

Run on every commit:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4   # pin to a full commit SHA in production pipelines
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test
      - run: npm run build
```

### Rust

```yaml
jobs:
  rust:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --all --check
      - run: cargo clippy --workspace --all-targets --all-features --locked -- -D warnings
      - run: cargo test --workspace --all-features --locked
```

Set `permissions: contents: read` at the top of workflows and grant more only to the jobs that need it. See the security supply-chain guideline for pinning and OIDC.

## Continuous Deployment

Deploy automatically after CI passes:

```yaml
deploy:
  needs: build
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - run: npm ci
    - run: npm run build
    - run: npm run deploy
      env:
        DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

## Deployment Strategies

### Blue-Green Deployment
Run two identical environments, switch traffic instantly.

### Canary Releases
Route small percentage of traffic to new version first.

### Rolling Updates
Gradually replace instances with new version.

## Best Practices

- Run fast tests first, slow tests later
- Cache dependencies between runs
- Use matrix builds for multiple versions/platforms
- Keep secrets in secure storage
- Automate database migrations
- Include rollback procedures
- Monitor deployments with health checks
- Use feature flags for safer releases
