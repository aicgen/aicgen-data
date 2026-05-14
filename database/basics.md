# Database Basics

## CRUD Operations

CRUD = Create, Read, Update, Delete

These are the four basic operations for working with data.

## CREATE - Inserting Data

### SQL

```sql
-- Insert single record
INSERT INTO users (name, email)
VALUES ('Alice', 'alice@example.com');

-- Insert multiple records
INSERT INTO users (name, email)
VALUES
  ('Bob', 'bob@example.com'),
  ('Carol', 'carol@example.com');
```

### With Code

```pseudocode
// Using SQL directly
database.execute(
    "INSERT INTO users (name, email) VALUES (?, ?)",
    ['Alice', 'alice@example.com']
)

// Using ORM/Model
user = User.create({
    name: 'Alice',
    email: 'alice@example.com'
})
```

## READ - Querying Data

### Select All

```sql
-- Get all records
SELECT * FROM users;

-- Get specific columns
SELECT id, name FROM users;
```

### Filter with WHERE

```sql
-- Find by ID
SELECT * FROM users WHERE id = 123;

-- Find by condition
SELECT * FROM users WHERE age > 18;

-- Multiple conditions
SELECT * FROM users
WHERE age > 18 AND status = 'active';
```

### With Code

```pseudocode
// Get all
users = database.query("SELECT * FROM users")

// Get by ID
user = database.query(
    "SELECT * FROM users WHERE id = ?",
    [userId]
)

// Using ORM/Model
user = User.findById(123)
activeUsers = User.findAll({
    where: { status: 'active' }
})
```

## UPDATE - Modifying Data

### SQL

```sql
-- Update single field
UPDATE users
SET email = 'newemail@example.com'
WHERE id = 123;

-- Update multiple fields
UPDATE users
SET name = 'Alice Smith',
    email = 'alice.smith@example.com'
WHERE id = 123;

-- Update with condition
UPDATE users
SET status = 'inactive'
WHERE last_login < '2023-01-01';
```

### With Code

```pseudocode
// Using SQL
database.execute(
    "UPDATE users SET email = ? WHERE id = ?",
    ['newemail@example.com', 123]
)

// Using ORM/Model
User.update(
    { email: 'newemail@example.com' },
    { where: { id: 123 } }
)
```

## DELETE - Removing Data

### SQL

```sql
-- Delete by ID
DELETE FROM users WHERE id = 123;

-- Delete with condition
DELETE FROM users WHERE status = 'banned';

-- Delete all (be careful!)
DELETE FROM users;
```

### With Code

```pseudocode
// Using SQL
database.execute("DELETE FROM users WHERE id = ?", [userId])

// Using ORM/Model
User.destroy({ where: { id: userId } })
```

## Sorting Results

```sql
-- Sort ascending (A-Z, 0-9)
SELECT * FROM users ORDER BY name ASC;

-- Sort descending (Z-A, 9-0)
SELECT * FROM users ORDER BY created_at DESC;

-- Sort by multiple columns
SELECT * FROM users
ORDER BY status ASC, name ASC;
```

## Limiting Results

```sql
-- Get first 10 records
SELECT * FROM users LIMIT 10;

-- Skip first 20, get next 10
SELECT * FROM users LIMIT 10 OFFSET 20;

-- Most recent 5 users
SELECT * FROM users
ORDER BY created_at DESC
LIMIT 5;
```

## Counting Records

```sql
-- Count all users
SELECT COUNT(*) FROM users;

-- Count by condition
SELECT COUNT(*) FROM users WHERE status = 'active';
```

## Common Patterns

### Check if Record Exists

```pseudocode
user = database.query(
    "SELECT * FROM users WHERE email = ?",
    [email]
)

if user is empty:
    print("User not found")
else:
    print("User exists")
```

### Get One or Return Default

```pseudocode
user = database.query(
    "SELECT * FROM users WHERE id = ?",
    [userId]
)

return user[0] or null
```

### Update if Exists, Create if Not

```pseudocode
existing = database.query(
    "SELECT * FROM users WHERE email = ?",
    [email]
)

if existing.length > 0:
    // Update
    database.execute(
        "UPDATE users SET name = ? WHERE email = ?",
        [name, email]
    )
else:
    // Create
    database.execute(
        "INSERT INTO users (name, email) VALUES (?, ?)",
        [name, email]
    )
```

## Always Use Parameterized Queries

```pseudocode
// ❌ DANGEROUS: SQL Injection risk
email = userInput
database.query("SELECT * FROM users WHERE email = '" + email + "'")
// If email = "'; DROP TABLE users; --"  😱

// ✅ SAFE: Parameterized query
database.query(
    "SELECT * FROM users WHERE email = ?",
    [email]
)
```

## Basic WHERE Conditions

```sql
-- Equals
WHERE status = 'active'

-- Not equals
WHERE status != 'banned'

-- Greater than / Less than
WHERE age > 18
WHERE price < 100

-- NULL checks
WHERE deleted_at IS NULL
WHERE email IS NOT NULL

-- IN list
WHERE status IN ('active', 'pending')

-- Pattern matching
WHERE email LIKE '%@gmail.com'

-- Multiple conditions
WHERE age > 18 AND status = 'active'
WHERE role = 'admin' OR role = 'moderator'
```

## Basic Data Types

```sql
-- Text
name VARCHAR(255)
description TEXT

-- Numbers
age INTEGER
price DECIMAL(10, 2)

-- Dates
created_at TIMESTAMP
birth_date DATE

-- Boolean
is_active BOOLEAN
```

## Best Practices

1. **Always use parameterized queries** - Prevents SQL injection
2. **Add WHERE to UPDATE/DELETE** - Avoid accidentally modifying all records
3. **Use LIMIT for testing** - Prevent fetching too much data
4. **Check query results** - Handle empty results gracefully

```pseudocode
// ✅ Good practices
async function getUser(id):
    result = database.query(
        "SELECT * FROM users WHERE id = ? LIMIT 1",
        [id]
    )

    if result.length == 0:
        return null  // Handle not found

    return result[0]

async function updateUserEmail(id, email):
    // Validate first
    if not email.contains('@'):
        throw Error("Invalid email")

    // Update with WHERE
    database.execute(
        "UPDATE users SET email = ? WHERE id = ?",
        [email, id]
    )
```

## Common Mistakes

```pseudocode
// ❌ String concatenation (SQL injection!)
database.query("SELECT * FROM users WHERE id = " + userId)

// ❌ No WHERE clause (updates everything!)
database.query("UPDATE users SET status = 'banned'")

// ❌ Not handling empty results
user = result[0]  // Crashes if empty
print(user.name)  // undefined error

// ✅ Correct
database.query("SELECT * FROM users WHERE id = ?", [userId])
database.query("UPDATE users SET status = ? WHERE id = ?", ['banned', userId])
user = result[0] or null
if user is not null:
    print(user.name)
```

## Testing Queries

```pseudocode
// Start with SELECT to verify
usersToDelete = database.query(
    "SELECT * FROM users WHERE status = ?",
    ['inactive']
)

print("Would delete " + usersToDelete.length + " users")

// If correct, change to DELETE
// database.execute("DELETE FROM users WHERE status = ?", ['inactive'])
```
