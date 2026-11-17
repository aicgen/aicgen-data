# Injection Prevention

## Rules

- Never build SQL, shell commands, queries or HTML by concatenating user input; use parameterized queries, argument arrays and context-aware escaping.
- Validate input at the boundary with an allowlist (type, length, format); validation supplements, and never replaces, parameterization.
- Do not pass user-controlled objects into NoSQL queries; coerce to expected primitive types first.
- Avoid shell execution; if unavoidable, call the program directly with an argument array and no shell.
- Resolve file paths against a fixed base directory and reject any result that escapes it.
- Use least-privilege database accounts so an injection has limited impact.
- Examples below are TypeScript; apply the same principles idiomatically in the project language.

## SQL Injection Prevention

```typescript
// ❌ DANGEROUS: String concatenation
const getUserByEmail = async (email: string) => {
  const query = `SELECT * FROM users WHERE email = '${email}'`;
  // Input: ' OR '1'='1
  // Result: SELECT * FROM users WHERE email = '' OR '1'='1'
  return db.query(query);
};

// ✅ SAFE: Parameterized queries
const getUserByEmail = async (email: string) => {
  return db.query('SELECT * FROM users WHERE email = ?', [email]);
};

// ✅ SAFE: Using ORM
const getUserByEmail = async (email: string) => {
  return userRepository.findOne({ where: { email } });
};

// ✅ SAFE: Query builder
const getUsers = async (minAge: number) => {
  return db
    .select('*')
    .from('users')
    .where('age', '>', minAge); // Automatically parameterized
};
```

## NoSQL Injection Prevention

```typescript
// ❌ DANGEROUS: Accepting objects from user input
app.post('/login', (req, res) => {
  const { username, password } = req.body;
  // If password = {$gt: ""}, it bypasses password check!
  db.users.findOne({ username, password });
});

// ✅ SAFE: Validate input types
app.post('/login', (req, res) => {
  const { username, password } = req.body;

  if (typeof username !== 'string' || typeof password !== 'string') {
    throw new Error('Invalid input types');
  }

  db.users.findOne({ username, password });
});
```

## Command Injection Prevention

```typescript
// ❌ DANGEROUS: Shell command with user input
const convertImage = async (filename: string) => {
  exec(`convert ${filename} output.jpg`);
  // Input: "file.png; rm -rf /"
};

// ✅ SAFE: Use arrays, avoid shell
import { execFile } from 'child_process';

const convertImage = async (filename: string) => {
  execFile('convert', [filename, 'output.jpg']);
};

// ✅ SAFE: Validate input against whitelist
const allowedFilename = /^[a-zA-Z0-9_-]+\.(png|jpg|gif)$/;
if (!allowedFilename.test(filename)) {
  throw new Error('Invalid filename');
}
```

## Path Traversal Prevention

```typescript
// ❌ DANGEROUS: Direct path usage
app.get('/files/:filename', (req, res) => {
  res.sendFile(`/uploads/${req.params.filename}`);
  // Input: ../../etc/passwd
});

// ✅ SAFE: Validate and normalize path
import path from 'path';

app.get('/files/:filename', (req, res) => {
  const safeName = path.basename(req.params.filename);
  const filePath = path.join('/uploads', safeName);
  const normalizedPath = path.normalize(filePath);

  if (!normalizedPath.startsWith('/uploads/')) {
    return res.status(400).json({ error: 'Invalid filename' });
  }

  res.sendFile(normalizedPath);
});
```

## Input Validation

```typescript
// ✅ Whitelist validation
import { z } from 'zod';

const userSchema = z.object({
  email: z.string().email(),
  password: z.string().min(12).max(160),
  age: z.number().int().min(0).max(150),
  role: z.enum(['user', 'admin'])
});

const validateUser = (data: unknown) => {
  return userSchema.parse(data);
};
```

## Rust: Injection Prevention

```rust,ignore
// ❌ DANGEROUS: formatting user input into SQL
let sql = format!("SELECT * FROM users WHERE email = '{email}'");

// ✅ SAFE: bound parameters (sqlx)
let user = sqlx::query_as!(User, "SELECT id, email FROM users WHERE email = $1", email)
    .fetch_optional(&pool)
    .await?;

// ❌ DANGEROUS: shell with interpolated input
std::process::Command::new("sh").arg("-c").arg(format!("convert {filename} out.jpg"));

// ✅ SAFE: no shell; each argument is passed separately
std::process::Command::new("convert").arg("--").arg(&filename).arg("out.jpg").status()?;

// ✅ SAFE: path traversal check before touching the filesystem
let full = base_dir.join(&requested).canonicalize()?;
if !full.starts_with(&base_dir) {
    return Err(ApiError::Forbidden);
}
```
