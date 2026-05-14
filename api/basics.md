# API Basics

## HTTP Methods

Use the right method for each operation:

| Method | Purpose | Example |
|--------|---------|---------|
| GET | Read data | Get list of users |
| POST | Create new resource | Create a new user |
| PUT | Replace entire resource | Update all user fields |
| PATCH | Update part of resource | Update user's email only |
| DELETE | Remove resource | Delete a user |

## GET - Reading Data

```pseudocode
// Get all items
route GET "/api/users":
    users = getAllUsers()
    return JSON(users)

// Get single item by ID
route GET "/api/users/:id":
    user = getUserById(params.id)
    if user is null:
        return status 404, JSON({ error: "User not found" })
    return JSON(user)
```

## POST - Creating Data

```pseudocode
route POST "/api/users":
    name = request.body.name
    email = request.body.email

    // Validate input
    if name is empty or email is empty:
        return status 400, JSON({ error: "Name and email required" })

    newUser = createUser({ name, email })

    // Return 201 Created with new resource
    return status 201, JSON(newUser)
```

## PUT - Replacing Data

```pseudocode
route PUT "/api/users/:id":
    id = params.id
    name = request.body.name
    email = request.body.email

    user = getUserById(id)
    if user is null:
        return status 404, JSON({ error: "User not found" })

    updated = replaceUser(id, { name, email })
    return JSON(updated)
```

## PATCH - Updating Data

```pseudocode
route PATCH "/api/users/:id":
    id = params.id
    updates = request.body  // Only fields to update

    user = getUserById(id)
    if user is null:
        return status 404, JSON({ error: "User not found" })

    updated = updateUser(id, updates)
    return JSON(updated)
```

## DELETE - Removing Data

```pseudocode
route DELETE "/api/users/:id":
    id = params.id

    user = getUserById(id)
    if user is null:
        return status 404, JSON({ error: "User not found" })

    deleteUser(id)

    // 204 No Content - successful deletion
    return status 204
```

## HTTP Status Codes

### Success Codes (2xx)

```pseudocode
// 200 OK - Request succeeded
return status 200, JSON(data)

// 201 Created - New resource created
return status 201, JSON(newResource)

// 204 No Content - Success with no response body
return status 204
```

### Client Error Codes (4xx)

```pseudocode
// 400 Bad Request - Invalid input
return status 400, JSON({ error: "Invalid email format" })

// 401 Unauthorized - Not authenticated
return status 401, JSON({ error: "Login required" })

// 403 Forbidden - Authenticated but not allowed
return status 403, JSON({ error: "Admin access required" })

// 404 Not Found - Resource doesn't exist
return status 404, JSON({ error: "User not found" })

// 409 Conflict - Resource already exists
return status 409, JSON({ error: "Email already registered" })
```

### Server Error Codes (5xx)

```pseudocode
// 500 Internal Server Error - Unexpected error
return status 500, JSON({ error: "Internal server error" })

// 503 Service Unavailable - Temporary issue
return status 503, JSON({ error: "Database unavailable" })
```

## URL Structure

Use clear, hierarchical URLs:

```
✅ Good
GET  /api/users           # List all users
GET  /api/users/123       # Get user 123
POST /api/users           # Create user
GET  /api/users/123/posts # Get posts by user 123

❌ Bad
GET  /api/getUsers
POST /api/createUser
GET  /api/user?id=123
```

## Request and Response Format

### JSON Request Body

```
// Client sends
POST /api/users
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

### JSON Response

```
// Server responds
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

## Query Parameters

Use query parameters for filtering, sorting, and pagination:

```pseudocode
// Filter by status
// GET /api/orders?status=pending
route GET "/api/orders":
    status = query.status
    orders = getOrders({ status })
    return JSON(orders)

// Sort by field
// GET /api/users?sort=name

// Pagination
// GET /api/users?page=2&limit=20
```

## Error Responses

Always return consistent error format:

```pseudocode
// ✅ Good: Structured error
return status 400, JSON({
    error: {
        code: "VALIDATION_ERROR",
        message: "Invalid input",
        details: {
            email: "Email format is invalid"
        }
    }
})

// ❌ Bad: Inconsistent
return status 400, "Bad request"
return status 400, JSON({ msg: "Error" })
```

## Best Practices

1. **Use correct HTTP methods** - GET for reading, POST for creating, etc.
2. **Use appropriate status codes** - 200 for success, 404 for not found, etc.
3. **Return JSON** - Standard format for APIs
4. **Validate input** - Check data before processing
5. **Handle errors** - Return clear error messages

```pseudocode
// Complete example
route POST "/api/products":
    name = request.body.name
    price = request.body.price

    // Validate
    if name is empty or price is empty:
        return status 400, JSON({
            error: "Name and price are required"
        })

    if price < 0:
        return status 400, JSON({
            error: "Price cannot be negative"
        })

    // Check for duplicates
    if productExists(name):
        return status 409, JSON({
            error: "Product already exists"
        })

    // Create
    product = createProduct({ name, price })

    // Return success
    return status 201, JSON(product)
```

## Common Mistakes

```pseudocode
// ❌ Wrong method for operation
route GET "/api/users/delete/:id"  // Should be DELETE

// ❌ Wrong status code
route POST "/api/users":
    user = createUser(body)
    return status 200, JSON(user)  // Should be 201

// ❌ Not handling missing resources
route GET "/api/users/:id":
    user = getUserById(params.id)
    return JSON(user)  // What if user is null?

// ✅ Correct
route DELETE "/api/users/:id"

route POST "/api/users":
    user = createUser(body)
    return status 201, JSON(user)

route GET "/api/users/:id":
    user = getUserById(params.id)
    if user is null:
        return status 404, JSON({ error: "User not found" })
    return JSON(user)
```
