# Secrets Management

## Rules

- Never commit secrets or hardcode them; load them from environment variables or a secret manager at startup and fail fast when one is missing.
- Keep `.env*` in `.gitignore` and commit only a `.env.example` with placeholder values.
- Generate secrets with a cryptographically secure source (at least 32 random bytes); never reuse them across environments.
- Store CI/CD secrets in the platform's secret store, never in workflow files, and scope them to the jobs that need them.
- Keep secrets out of logs, errors, URLs and client bundles; redact on output.
- Rotate on a schedule and immediately after exposure; treat any leaked secret as compromised and revoke it.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Environment Variables

```typescript
// ❌ NEVER hardcode secrets
const config = {
  dbPassword: 'super_secret_password',
  apiKey: 'sk-1234567890abcdef'
};

// ✅ Use environment variables
import dotenv from 'dotenv';
dotenv.config();

const config = {
  dbPassword: process.env.DB_PASSWORD,
  apiKey: process.env.API_KEY,
  sessionSecret: process.env.SESSION_SECRET
};
```

## Validate Required Secrets

```typescript
// ✅ Fail fast if secrets missing
const requiredEnvVars = [
  'DB_PASSWORD',
  'API_KEY',
  'SESSION_SECRET',
  'JWT_SECRET'
];

requiredEnvVars.forEach(varName => {
  if (!process.env[varName]) {
    throw new Error(`Missing required environment variable: ${varName}`);
  }
});

// ✅ Type-safe config
interface Config {
  dbPassword: string;
  apiKey: string;
  sessionSecret: string;
}

function loadConfig(): Config {
  const dbPassword = process.env.DB_PASSWORD;
  if (!dbPassword) throw new Error('DB_PASSWORD required');

  // ... validate all required vars

  return { dbPassword, apiKey, sessionSecret };
}
```

## Generate Strong Secrets

```bash
# Generate cryptographically secure secrets
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"

# Or using OpenSSL
openssl rand -base64 32

# Or using head
head -c32 /dev/urandom | base64
```

## .gitignore Configuration

```bash
# .gitignore - NEVER commit secrets
.env
.env.local
.env.*.local
*.key
*.pem
secrets/
credentials.json
```

## Environment Example File

```bash
# .env.example - commit this to show required variables
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp
DB_USER=
DB_PASSWORD=

API_KEY=
SESSION_SECRET=
JWT_SECRET=

# Copy to .env and fill in actual values
```

## Secrets in CI/CD

```yaml
# GitHub Actions
- name: Deploy
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    API_KEY: ${{ secrets.API_KEY }}
  run: ./deploy.sh

# ❌ Never echo secrets in logs
- name: Configure
  run: |
    echo "Configuring application..."
    # echo "DB_PASSWORD=$DB_PASSWORD"  # NEVER do this!
```

## Secrets Rotation

```typescript
// ✅ Support for rotating secrets
class SecretManager {
  async getSecret(name: string): Promise<string> {
    // Check for new secret first (during rotation)
    const newSecret = process.env[`${name}_NEW`];
    if (newSecret) {
      return newSecret;
    }

    const secret = process.env[name];
    if (!secret) {
      throw new Error(`Secret ${name} not found`);
    }
    return secret;
  }
}

// ✅ Accept multiple JWT signing keys during rotation
function verifyToken(token: string) {
  const keys = [process.env.JWT_SECRET, process.env.JWT_SECRET_OLD].filter(Boolean);

  for (const key of keys) {
    try {
      return jwt.verify(token, key);
    } catch {}
  }
  throw new Error('Invalid token');
}
```

## Rust: Keeping Secrets Out Of Logs

```rust,ignore
use secrecy::{ExposeSecret, SecretString};

#[derive(Debug, serde::Deserialize)]
pub struct Settings {
    pub database_url: SecretString, // Debug prints "[REDACTED]"
}

let pool = sqlx::PgPool::connect(settings.database_url.expose_secret()).await?;
```

- Load secrets from the environment or a secret manager at startup; never commit them or embed with `include_str!`.
- Do not put secrets in error messages, `tracing` fields or `#[derive(Debug)]` output of types that hold them.
