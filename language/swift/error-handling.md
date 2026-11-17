# Swift Error Handling

## Rules

- Model failures as specific error enums that conform to `Error` (add `LocalizedError` for user-facing text). Avoid catch-all cases such as `.failed` or `.unknown`.
- Use `throws` / `async throws` by default. Use `Result` only to store an outcome, cross a callback boundary, or collect several outcomes.
- Handle an error where you know what to do about it; otherwise let it propagate. Never leave a `catch` block empty.
- Avoid `try!` and force-unwrap outside tests or provably infallible code (such as a bundled resource). Use `try?` only when the failure reason does not matter.
- Programmer mistakes (violated preconditions) use `precondition` / `assert`, not thrown errors.
- Rethrow `CancellationError` and call `Task.checkCancellation()` in long loops. Cancellation is not a failure to report to the user.
- Never show raw `error` text to users and never log secrets or personal data in error context.

## Define errors

```swift
enum NetworkError: Error {
  case connectionFailed
  case timeout
  case unauthorized
  case serverError(code: Int)
}

enum ValidationError: Error {
  case emptyField(name: String)
  case tooShort(name: String, minimum: Int)
}

// User-facing text belongs in LocalizedError.errorDescription.
// Overriding localizedDescription directly does not work on a plain Error.
enum AuthError: LocalizedError {
  case invalidCredentials
  case sessionExpired

  var errorDescription: String? {
    switch self {
    case .invalidCredentials: "The email or password is incorrect."
    case .sessionExpired: "Your session has expired. Please sign in again."
    }
  }
}
```

Swift 6 supports typed throws (`func load() throws(NetworkError)`). Use it only for closed, stable error sets inside a module; keep plain `throws` on public API that may grow.

## Throw and catch

```swift
func fetchUser(id: String) async throws -> User {
  guard !id.isEmpty else { throw ValidationError.emptyField(name: "id") }
  let (data, response) = try await URLSession.shared.data(from: url(for: id))
  guard let http = response as? HTTPURLResponse else { throw NetworkError.connectionFailed }
  guard (200..<300).contains(http.statusCode) else { throw NetworkError.serverError(code: http.statusCode) }
  return try JSONDecoder().decode(User.self, from: data)
}

func loadProfile() async {
  do {
    user = try await fetchUser(id: userId)
  } catch NetworkError.unauthorized {
    showLoginScreen()                       // we know what to do here
  } catch {
    errorMessage = error.localizedDescription
  }
}

func transform<T, U>(_ items: [T], using f: (T) throws -> U) rethrows -> [U] {
  try items.map(f)                          // rethrows: only throws if `f` does
}
```

## `try`, `try?` and `try!`

| Form | Use for |
|------|---------|
| `try` | The normal case; propagate or catch |
| `try?` | Failure is expected and the reason is irrelevant (cache read, optional config) |
| `try!` | Never in production paths; only where failure is a bug (use `fatalError` with a message instead) |

## Result

```swift
// Store an outcome in state.
@Published var userResult: Result<User, Error>?

// Convert between styles.
let result = Result { try parse(data) }     // throwing -> Result
let user = try result.get()                 // Result -> throwing

// Transform without unwrapping.
let name = result.map(\.name)
let message = result.mapError { UserFacingError($0) }
```

`Result` has no default-value accessor; switch on it, or use `try? result.get()`.

## Recovery

```swift
// Retry transient failures with exponential backoff.
func retry<T>(maxAttempts: Int = 3, _ operation: () async throws -> T) async throws -> T {
  var attempt = 0
  while true {
    do {
      return try await operation()
    } catch is CancellationError {
      throw CancellationError()
    } catch where attempt < maxAttempts - 1 {
      attempt += 1
      try await Task.sleep(for: .seconds(pow(2.0, Double(attempt))))
    }
  }
}

// Fall back through sources, most authoritative first.
func loadConfiguration() -> Configuration {
  (try? fetchRemoteConfig()) ?? (try? loadCachedConfig()) ?? .default
}
```

Retry only idempotent operations and only for transient errors (timeouts, 5xx). Never retry validation or authorization failures.

## Concurrency

```swift
// The first thrown error cancels the sibling tasks.
func fetchDashboard() async throws -> Dashboard {
  try await withThrowingTaskGroup(of: DashboardData.self) { group in
    group.addTask { try await fetchUserData() }
    group.addTask { try await fetchPosts() }
    var parts: [DashboardData] = []
    for try await part in group { parts.append(part) }
    return Dashboard(parts: parts)
  }
}

// Need every outcome? Return a Result from each child instead of throwing.
group.addTask {
  do { return .success(try await fetch(from: url)) }
  catch { return .failure(error) }
}

// Check for cancellation in long loops and clean up.
for item in items {
  try Task.checkCancellation()
  await process(item)
}
```

## Add context and log

```swift
struct ContextualError: Error {
  let underlying: Error
  let context: String
}

func chargeOrder() throws {
  do { try chargePayment() }
  catch { throw ContextualError(underlying: error, context: "Payment failed for order") }
}
```

Log once, at the boundary that handles the error (use `Logger` from `os`), with the context but not the user's data. Do not log and rethrow at every layer.

## Programmer errors

```swift
func divide(_ a: Int, by b: Int) -> Int {
  precondition(b != 0, "Division by zero")   // checked in release builds
  return a / b
}

assert(!items.isEmpty)                        // debug builds only
```

## User-facing messages

Map errors to friendly text in one place (for example an extension on `Error` or a view model) and keep technical detail in logs. Document thrown errors in the `- Throws:` section of doc comments on public API.

Sources: The Swift Programming Language (Error Handling), Apple's Swift documentation, SE-0413 (typed throws).
