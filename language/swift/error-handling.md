# Swift Error Handling

## Error Types

Define custom error types using enums conforming to Error:

```swift
// ✅ Simple error enum
enum NetworkError: Error {
  case connectionFailed
  case timeout
  case invalidResponse
  case unauthorized
  case serverError(code: Int)
}

// ✅ Error with associated values
enum ValidationError: Error {
  case emptyField(fieldName: String)
  case invalidFormat(fieldName: String, reason: String)
  case tooShort(fieldName: String, minimumLength: Int)
  case tooLong(fieldName: String, maximumLength: Int)
}

// ✅ Error with localized descriptions
enum AuthError: Error {
  case invalidCredentials
  case accountLocked
  case sessionExpired

  var localizedDescription: String {
    switch self {
    case .invalidCredentials:
      return "The email or password you entered is incorrect."
    case .accountLocked:
      return "Your account has been temporarily locked. Please try again later."
    case .sessionExpired:
      return "Your session has expired. Please log in again."
    }
  }
}

// ❌ Avoid overly generic errors
enum GenericError: Error {
  case error
  case failed
}
```

## Throwing Functions

Use `throws` to indicate a function can throw an error:

```swift
// ✅ Throwing function
func fetchUser(id: String) throws -> User {
  guard !id.isEmpty else {
    throw ValidationError.emptyField(fieldName: "id")
  }

  guard let user = database.findUser(id: id) else {
    throw DatabaseError.notFound
  }

  return user
}

// ✅ Async throwing function
func fetchUserAsync(id: String) async throws -> User {
  let response = try await URLSession.shared.data(from: url)
  return try JSONDecoder().decode(User.self, from: response.0)
}

// ✅ Rethrows - propagates errors from closure
func transform<T, U>(_ items: [T], using: (T) throws -> U) rethrows -> [U] {
  var results: [U] = []
  for item in items {
    let transformed = try using(item)
    results.append(transformed)
  }
  return results
}
```

## Handling Errors with do-catch

```swift
// ✅ Basic error handling
func loadUser() {
  do {
    let user = try fetchUser(id: "123")
    print("User: \(user.name)")
  } catch {
    print("Error: \(error)")
  }
}

// ✅ Catch specific errors
func handleSpecificErrors() {
  do {
    let user = try fetchUser(id: "123")
    print(user.name)
  } catch NetworkError.connectionFailed {
    print("No internet connection")
  } catch NetworkError.timeout {
    print("Request timed out")
  } catch NetworkError.serverError(let code) {
    print("Server error: \(code)")
  } catch {
    print("Unknown error: \(error)")
  }
}

// ✅ Async error handling
func loadUserAsync() async {
  do {
    let user = try await fetchUserAsync(id: "123")
    print("User: \(user.name)")
  } catch {
    print("Failed to load user: \(error)")
  }
}

// ❌ Don't catch and ignore errors silently
func badErrorHandling() {
  do {
    try riskyOperation()
  } catch {
    // Silent failure - bad practice!
  }
}
```

## Try Variants

Swift provides three variants of `try`:

```swift
// ✅ try - Must be in do-catch or throwing function
func normalTry() throws {
  let user = try fetchUser(id: "123")
  process(user)
}

// ✅ try? - Converts error to optional (nil on error)
func optionalTry() {
  if let user = try? fetchUser(id: "123") {
    print("User: \(user.name)")
  } else {
    print("Failed to fetch user")
  }
}

// ⚠️ try! - Force try (crashes on error) - Use sparingly
func forceTry() {
  let config = try! loadConfig() // Only if error is truly impossible
  print(config.apiKey)
}

// ✅ Good use of try! - Error is truly impossible
func loadBundledFile() -> String {
  guard let url = Bundle.main.url(forResource: "data", withExtension: "json"),
        let data = try? Data(contentsOf: url),
        let string = String(data: data, encoding: .utf8) else {
    fatalError("Bundled file must exist")
  }
  return string
}

// ❌ Bad use of try! - Error is possible
func badForceTry() {
  let user = try! fetchUser(id: "123") // Crashes if fetch fails!
}
```

## Result Type

Use Result for explicit success/failure without exceptions:

```swift
// ✅ Return Result instead of throwing
func fetchUser(id: String) -> Result<User, Error> {
  guard !id.isEmpty else {
    return .failure(ValidationError.emptyField(fieldName: "id"))
  }

  do {
    let user = try database.findUser(id: id)
    return .success(user)
  } catch {
    return .failure(error)
  }
}

// ✅ Handle Result with switch
func handleResult() {
  let result = fetchUser(id: "123")

  switch result {
  case .success(let user):
    print("User: \(user.name)")
  case .failure(let error):
    print("Error: \(error)")
  }
}

// ✅ Use Result methods
func useResultMethods() {
  let result = fetchUser(id: "123")

  // Get value or provide default
  let user = result.get(default: User.guest)

  // Map success value
  let userName = result.map { $0.name }

  // Map error
  let friendly = result.mapError { error in
    "Failed to load user: \(error.localizedDescription)"
  }
}

// ✅ Convert between Result and throws
func convertToThrowing() throws -> User {
  let result = fetchUser(id: "123")
  return try result.get() // Throws if failure
}

func convertFromThrowing() -> Result<User, Error> {
  Result { try fetchUserThrowing(id: "123") }
}
```

## When to Use Throws vs Result

### Use `throws` when:

```swift
// ✅ Synchronous operations where errors are exceptional
func parseJSON<T: Decodable>(_ data: Data) throws -> T {
  try JSONDecoder().decode(T.self, from: data)
}

// ✅ Async operations
func fetchData() async throws -> Data {
  try await URLSession.shared.data(from: url).0
}

// ✅ Errors should propagate automatically
func processOrder() throws {
  try validateOrder()
  try chargePayment()
  try shipOrder()
}
```

### Use `Result` when:

```swift
// ✅ Storing error state
class ViewModel: ObservableObject {
  @Published var userResult: Result<User, Error>?

  func loadUser() async {
    userResult = await fetchUserResult(id: "123")
  }
}

// ✅ Callback-based APIs
func fetchUser(completion: @escaping (Result<User, Error>) -> Void) {
  // Async operation
  DispatchQueue.global().async {
    let result = self.performFetch()
    DispatchQueue.main.async {
      completion(result)
    }
  }
}

// ✅ When you want to delay error handling
func loadUsers() -> [Result<User, Error>] {
  userIDs.map { id in
    Result { try fetchUser(id: id) }
  }
}
```

## Error Recovery Patterns

### Retry with Exponential Backoff

```swift
// ✅ Retry failing operations
func fetchWithRetry<T>(
  maxAttempts: Int = 3,
  operation: @escaping () async throws -> T
) async throws -> T {
  var lastError: Error?

  for attempt in 0..<maxAttempts {
    do {
      return try await operation()
    } catch {
      lastError = error

      if attempt < maxAttempts - 1 {
        let delay = UInt64(pow(2.0, Double(attempt)) * 1_000_000_000)
        try await Task.sleep(nanoseconds: delay)
      }
    }
  }

  throw lastError!
}

// Usage
let user = try await fetchWithRetry {
  try await fetchUser(id: "123")
}
```

### Fallback Values

```swift
// ✅ Provide fallback on error
func loadUserWithFallback(id: String) -> User {
  do {
    return try fetchUser(id: id)
  } catch {
    return User.guest
  }
}

// ✅ Try multiple sources
func loadConfiguration() -> Configuration {
  // Try remote config first
  if let config = try? fetchRemoteConfig() {
    return config
  }

  // Fall back to cached config
  if let config = try? loadCachedConfig() {
    return config
  }

  // Last resort: default config
  return Configuration.default
}
```

### Partial Success

```swift
// ✅ Return partial results with errors
func fetchAllUsers(ids: [String]) async -> (users: [User], errors: [Error]) {
  var users: [User] = []
  var errors: [Error] = []

  await withTaskGroup(of: Result<User, Error>.self) { group in
    for id in ids {
      group.addTask {
        do {
          return .success(try await fetchUser(id: id))
        } catch {
          return .failure(error)
        }
      }
    }

    for await result in group {
      switch result {
      case .success(let user):
        users.append(user)
      case .failure(let error):
        errors.append(error)
      }
    }
  }

  return (users, errors)
}
```

## Custom Error Context

Add context to errors:

```swift
// ✅ Wrap errors with context
struct ContextualError: Error {
  let underlyingError: Error
  let context: String
  let metadata: [String: Any]

  init(_ error: Error, context: String, metadata: [String: Any] = [:]) {
    self.underlyingError = error
    self.context = context
    self.metadata = metadata
  }
}

func fetchUserWithContext(id: String) throws -> User {
  do {
    return try fetchUser(id: id)
  } catch {
    throw ContextualError(
      error,
      context: "Failed to fetch user",
      metadata: ["userId": id, "timestamp": Date()]
    )
  }
}

// ✅ Add context without wrapping
extension Error {
  func addingContext(_ context: String) -> Error {
    ContextualError(self, context: context)
  }
}

func processOrder() throws {
  do {
    try chargePayment()
  } catch {
    throw error.addingContext("Payment processing failed")
  }
}
```

## Error Logging

```swift
// ✅ Structured error logging
func logError(_ error: Error, context: String, metadata: [String: Any] = [:]) {
  var logData: [String: Any] = [
    "error": String(describing: error),
    "context": context,
    "timestamp": Date().ISO8601Format()
  ]

  logData.merge(metadata) { (_, new) in new }

  // Log to your logging system
  print("ERROR: \(logData)")
}

// Usage
func handleError() {
  do {
    try riskyOperation()
  } catch {
    logError(error, context: "Risk operation failed", metadata: [
      "userId": currentUserId,
      "operation": "riskyOperation"
    ])
  }
}

// ✅ Error boundary pattern
func performWithErrorBoundary<T>(
  operation: () throws -> T,
  onError: (Error) -> Void
) -> T? {
  do {
    return try operation()
  } catch {
    logError(error, context: "Operation failed")
    onError(error)
    return nil
  }
}
```

## Preconditions and Assertions

For programmer errors, use preconditions instead of throwing:

```swift
// ✅ Precondition for programmer errors
func divide(_ a: Int, by b: Int) -> Int {
  precondition(b != 0, "Division by zero")
  return a / b
}

// ✅ Assert in debug builds only
func process(_ items: [Item]) {
  assert(!items.isEmpty, "Items should not be empty")
  // Process items
}

// ✅ Fatal error for unrecoverable errors
func loadRequiredConfiguration() -> Configuration {
  guard let config = try? loadConfiguration() else {
    fatalError("Failed to load required configuration")
  }
  return config
}

// ❌ Don't use throwing for programmer errors
func badDivide(_ a: Int, by b: Int) throws -> Int {
  guard b != 0 else {
    throw MathError.divisionByZero // Should be precondition
  }
  return a / b
}
```

## Error Handling Best Practices

### Be Specific with Error Types

```swift
// ❌ Generic error
enum AppError: Error {
  case error
  case somethingWentWrong
}

// ✅ Specific errors
enum UserServiceError: Error {
  case userNotFound(id: String)
  case invalidEmail(String)
  case duplicateEmail(String)
  case insufficientPermissions(required: Permission, actual: Permission)
}
```

### Handle Errors at the Right Level

```swift
// ✅ Handle errors where you have context
func loadUserProfile() async {
  do {
    let user = try await fetchUser(id: userId)
    self.user = user
  } catch NetworkError.unauthorized {
    // Handle at this level - we know what to do
    showLoginScreen()
  } catch {
    // Let other errors propagate or show generic error
    showErrorAlert(error.localizedDescription)
  }
}

// ❌ Don't catch errors you can't handle
func badErrorHandling() {
  do {
    try fetchUser(id: "123")
  } catch {
    print("Error!") // What now? No useful handling
  }
}
```

### Provide User-Friendly Messages

```swift
// ✅ Convert technical errors to user messages
extension Error {
  var userFriendlyMessage: String {
    switch self {
    case NetworkError.connectionFailed:
      return "Please check your internet connection and try again."
    case NetworkError.timeout:
      return "The request took too long. Please try again."
    case NetworkError.unauthorized:
      return "Your session has expired. Please log in again."
    case let ValidationError.invalidFormat(field, _):
      return "The \(field) format is invalid. Please check and try again."
    default:
      return "Something went wrong. Please try again later."
    }
  }
}

// Usage
do {
  try performOperation()
} catch {
  showAlert(message: error.userFriendlyMessage)
}
```

### Document Error Conditions

```swift
// ✅ Document what errors can be thrown
/// Fetches a user from the API.
///
/// - Parameter id: The unique identifier for the user
/// - Returns: The user object
/// - Throws:
///   - `NetworkError.connectionFailed` if no internet connection
///   - `NetworkError.unauthorized` if authentication fails
///   - `NetworkError.notFound` if user doesn't exist
///   - `DecodingError` if response parsing fails
func fetchUser(id: String) async throws -> User {
  // Implementation
}
```

## Async Error Handling Patterns

### Structured Concurrency Errors

```swift
// ✅ First error cancels sibling tasks
func fetchDashboard() async throws -> Dashboard {
  try await withThrowingTaskGroup(of: DashboardData.self) { group in
    group.addTask { try await fetchUserData() }
    group.addTask { try await fetchPosts() }
    group.addTask { try await fetchStats() }

    var results: [DashboardData] = []
    for try await result in group {
      results.append(result)
    }

    return Dashboard(data: results)
  }
  // If any task throws, others are automatically cancelled
}

// ✅ Collect all results, including errors
func fetchAllData() async -> [Result<Data, Error>] {
  await withTaskGroup(of: Result<Data, Error>.self) { group in
    for url in urls {
      group.addTask {
        do {
          return .success(try await fetch(from: url))
        } catch {
          return .failure(error)
        }
      }
    }

    var results: [Result<Data, Error>] = []
    for await result in group {
      results.append(result)
    }
    return results
  }
}
```

### Cancellation as Error

```swift
// ✅ Handle task cancellation
func performLongOperation() async throws -> Result {
  for i in 0..<1000 {
    // Check for cancellation
    try Task.checkCancellation()

    await processItem(i)
  }

  return result
}

// ✅ Graceful cancellation handling
func downloadWithCancellation() async throws -> Data {
  do {
    return try await performDownload()
  } catch is CancellationError {
    // Clean up partial download
    cleanupPartialData()
    throw DownloadError.cancelled
  }
}
```

---

**Sources:**
- [Swift.org: Error Handling](https://docs.swift.org/swift-book/LanguageGuide/ErrorHandling.html)
- [Apple Documentation: Error Handling](https://developer.apple.com/documentation/swift/error)
- [Swift by Sundell: Error Handling](https://www.swiftbysundell.com/basics/error-handling/)
- [WWDC: Modern Swift Error Handling](https://developer.apple.com/videos/play/wwdc2020/10672/)
