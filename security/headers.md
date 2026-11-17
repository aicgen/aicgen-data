# Security Headers

## Rules

- Set security headers globally (Helmet or equivalent) and verify them with an automated check.
- Use a Content-Security-Policy that denies by default (`default-src 'self'`); avoid `unsafe-inline` and `unsafe-eval`, and use nonces or hashes.
- Send HSTS, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY` (or CSP `frame-ancestors`), `Referrer-Policy` and a minimal `Permissions-Policy`. Do not rely on `X-XSS-Protection`.
- Configure CORS with an explicit origin allowlist; never combine a wildcard origin with credentials.
- Mark cookies `Secure`, `HttpOnly` and `SameSite`, and redirect all HTTP to HTTPS.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Essential Headers with Helmet

```typescript
import helmet from 'helmet';

// ✅ Apply security headers with sensible defaults
app.use(helmet());

// ✅ Custom configuration
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
}));
```

## Manual Header Configuration

```typescript
app.use((req, res, next) => {
  // Prevent MIME sniffing
  res.setHeader('X-Content-Type-Options', 'nosniff');

  // Prevent clickjacking
  res.setHeader('X-Frame-Options', 'DENY');

  // Legacy XSS auditor is deprecated and can introduce bugs: disable it, rely on CSP
  res.setHeader('X-XSS-Protection', '0');

  // Force HTTPS
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');

  // Referrer policy
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');

  // Permissions policy
  res.setHeader('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');

  next();
});
```

## Content Security Policy (CSP)

```typescript
// ✅ Strict CSP for maximum protection
res.setHeader('Content-Security-Policy', [
  "default-src 'self'",
  "script-src 'self'",
  "style-src 'self' 'unsafe-inline'",
  "img-src 'self' data: https:",
  "font-src 'self'",
  "connect-src 'self' https://api.example.com",
  "frame-ancestors 'none'",
  "form-action 'self'"
].join('; '));

// For APIs that don't serve HTML
res.setHeader('Content-Security-Policy', "default-src 'none'");
```

## CORS Configuration

```typescript
import cors from 'cors';

// ✅ Configure CORS properly
app.use(cors({
  origin: ['https://example.com', 'https://app.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400 // Cache preflight for 24 hours
}));

// ❌ Never use in production
app.use(cors({ origin: '*' })); // Allows any origin
```

## HTTPS Enforcement

```typescript
// ✅ Redirect HTTP to HTTPS
app.use((req, res, next) => {
  if (!req.secure && req.get('x-forwarded-proto') !== 'https') {
    return res.redirect(301, `https://${req.hostname}${req.url}`);
  }
  next();
});

// ✅ HSTS header (included in helmet)
res.setHeader(
  'Strict-Transport-Security',
  'max-age=31536000; includeSubDomains; preload'
);
```

## Cookie Security

```typescript
// ✅ Secure cookie settings
app.use(session({
  cookie: {
    secure: true,        // Only send over HTTPS
    httpOnly: true,      // Not accessible via JavaScript
    sameSite: 'strict',  // CSRF protection
    maxAge: 30 * 60 * 1000
  }
}));

// ✅ Set secure cookies manually
res.cookie('token', value, {
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'strict',
  maxAge: 3600000
});
```

## Header Checklist

```
✅ X-Content-Type-Options: nosniff
✅ X-Frame-Options: DENY
✅ X-XSS-Protection: 0 (or omit; rely on CSP)
✅ Strict-Transport-Security: max-age=31536000
✅ Content-Security-Policy: (appropriate policy)
✅ Referrer-Policy: strict-origin-when-cross-origin
✅ Permissions-Policy: restrict unused features
✅ Secure, HttpOnly, SameSite cookies
```
