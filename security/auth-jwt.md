# Authentication & JWT Security

## Rules

- Hash passwords with a slow, salted algorithm: Argon2id preferred, or bcrypt with cost 12 or higher. Never use plain SHA or MD5, and never log or return passwords.
- Keep access tokens short-lived (about 15 minutes) and rotate refresh tokens; store refresh tokens server-side or as httpOnly, Secure, SameSite cookies, never in localStorage.
- Verify JWTs with a pinned algorithm allowlist, and check signature, `exp`, `iss` and `aud`; reject `alg: none`. Keep signing keys in a secret manager.
- Rate-limit and lock out login, password reset and token endpoints; return the same message for unknown user and wrong password.
- Authorize on the server for every request (role and ownership checks); never trust client-supplied roles or IDs.
- Regenerate the session ID on login, set cookie flags (`httpOnly`, `secure`, `sameSite`), and invalidate sessions on logout and password change.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## Password Storage

```typescript
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12; // Work factor

// ✅ Hash password with bcrypt
async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}

// ✅ Validate password strength
function validatePassword(password: string): void {
  if (password.length < 12) {
    throw new Error('Password must be at least 12 characters');
  }
  if (password.length > 160) {
    throw new Error('Password too long'); // Prevent DoS via bcrypt
  }
}
```

## JWT Best Practices

```typescript
import jwt from 'jsonwebtoken';

const JWT_SECRET = process.env.JWT_SECRET!;
const ACCESS_TOKEN_EXPIRY = '15m';
const REFRESH_TOKEN_EXPIRY = '7d';

// ✅ Generate tokens
function generateTokens(userId: string) {
  const accessToken = jwt.sign(
    { sub: userId, type: 'access' },
    JWT_SECRET,
    { expiresIn: ACCESS_TOKEN_EXPIRY }
  );

  const refreshToken = jwt.sign(
    { sub: userId, type: 'refresh' },
    JWT_SECRET,
    { expiresIn: REFRESH_TOKEN_EXPIRY }
  );

  return { accessToken, refreshToken };
}

// ✅ Verify and decode token
function verifyToken(token: string) {
  try {
    return jwt.verify(token, JWT_SECRET);
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw new UnauthorizedError('Token expired');
    }
    throw new UnauthorizedError('Invalid token');
  }
}
```

## Login Protection

```typescript
import rateLimit from 'express-rate-limit';

// ✅ Rate limit login attempts
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts
  message: 'Too many login attempts, please try again later',
});

app.post('/login', loginLimiter, async (req, res) => {
  const { email, password } = req.body;

  const user = await userService.findByEmail(email);

  // ✅ Generic error message (don't reveal if user exists)
  if (!user || !await verifyPassword(password, user.passwordHash)) {
    return res.status(401).json({ error: 'Invalid email or password' });
  }

  const tokens = generateTokens(user.id);

  // Regenerate session to prevent fixation
  req.session.regenerate(() => {
    res.json({ ...tokens });
  });
});
```

## Session Security

```typescript
app.use(session({
  secret: process.env.SESSION_SECRET!,
  name: 'sessionId', // Don't use default 'connect.sid'

  cookie: {
    secure: true,        // HTTPS only
    httpOnly: true,      // Prevent XSS access
    sameSite: 'strict',  // CSRF protection
    maxAge: 30 * 60 * 1000, // 30 minutes
  },

  resave: false,
  saveUninitialized: false,
  store: new RedisStore({ client: redisClient })
}));

// ✅ Session regeneration after login
app.post('/login', async (req, res, next) => {
  // ... authenticate user ...

  req.session.regenerate((err) => {
    req.session.userId = user.id;
    res.json({ success: true });
  });
});
```

## Authorization Middleware

```typescript
// ✅ Require authentication
const requireAuth = async (req: Request, res: Response, next: NextFunction) => {
  const token = req.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'Authentication required' });
  }

  try {
    const payload = verifyToken(token);
    req.user = await userService.findById(payload.sub);
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// ✅ Require specific role
const requireRole = (...roles: string[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
};
```
