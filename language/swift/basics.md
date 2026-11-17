# Swift Language Fundamentals

## Rules

- Follow the Swift API Design Guidelines: clarity at the point of use, `UpperCamelCase` types and protocols, `lowerCamelCase` everything else (including constants), argument labels that read as grammatical phrases.
- Prefer `struct` and value semantics; use `class` only for identity, shared mutable state or inheritance, and mark classes `final` by default.
- Use `let` over `var`, and optionals instead of sentinel values. Avoid force unwraps (`!`) and `try!`; the only common exception is an `@IBOutlet` that is guaranteed to be set after load.
- Exit early with `guard`; keep the happy path at the lowest indentation.
- Use the narrowest access control (`private` by default, `public` only for API) and keep stored state private.
- Organize code with extensions (one per protocol conformance) and abstract with protocols.
- Do not shadow standard library types (`Result`, `Optional`); use `Result<Success, Failure>` from the standard library.
- Run SwiftLint and SwiftFormat, and enable Swift 6 strict concurrency for new code.

## Naming

```swift
struct UserProfile { }          // types: UpperCamelCase
protocol Drawable { }
func calculateTotal(items: [Item]) -> Double { }   // functions, vars: lowerCamelCase
let maxRetries = 3              // constants are lowerCamelCase, not MAX_RETRIES
enum NetworkError { case timeout }                 // enum: singular type, lowerCamelCase cases
```

## Types and inference

```swift
let name = "Alice"                       // infer obvious types
let timeout: TimeInterval = 30           // annotate when it adds meaning
typealias UserID = String                // name a type to document intent
typealias CompletionHandler = (Result<Data, Error>) -> Void
```

Always declare return types explicitly; avoid redundant annotations like `let message: String = "Hello"`.

## Value vs reference types

```swift
struct User {                   // copied on assignment; safe to share
  let id: String
  var name: String
}

final class NetworkManager {    // identity and cleanup need a class
  private let session: URLSession
  init(configuration: URLSessionConfiguration) { session = URLSession(configuration: configuration) }
  deinit { session.invalidateAndCancel() }
}
```

## Properties

```swift
struct Circle {
  var radius: Double
  var diameter: Double { radius * 2 }       // derive, do not store
  var area: Double { .pi * radius * radius }
}
```

Use `willSet` / `didSet` only for real side effects; prefer computed properties for derived values.

## Extensions and access control

```swift
public struct User {
  public let id: String
  public var name: String
  private var cache: [String: Any] = [:]    // keep internals private
  public init(id: String, name: String) { self.id = id; self.name = name }
}

extension User: Codable { }
extension User: Equatable {
  static func == (lhs: User, rhs: User) -> Bool { lhs.id == rhs.id }
}
extension User {
  var displayName: String { name.isEmpty ? "Unknown User" : name }
}
```

## Guard and optionals

```swift
func process(_ user: User?) {
  guard let user else { return }                  // shorthand optional binding (Swift 5.7+)
  guard user.isValid, user.age >= 18 else { return }
  print("Processing \(user.name)")
}

let name = optionalName ?? "Unknown"              // default
let email = users.first?.email                    // chaining
```

## Standard library

```swift
let names: [String] = ["Alice", "Bob"]            // Swift collections, not NSArray / NSDictionary
let doubled = [1, 2, 3].map { $0 * 2 }
let valid = ["1", "foo", "3"].compactMap { Int($0) }   // filter and transform at once
```

Use trailing closures for a single closure argument; keep parentheses when it makes a chain clearer.

## Enums

```swift
enum HTTPMethod: String { case get = "GET", post = "POST" }

func message(for error: NetworkError) -> String {
  switch error {                                  // exhaustive: no default, so new cases fail to compile
  case .noConnection: "No internet connection"
  case .timeout: "Request timed out"
  case .unauthorized: "Unauthorized"
  }
}
```

## Documentation

```swift
/// Fetches a user.
///
/// - Parameter id: The user's unique identifier.
/// - Returns: The user.
/// - Throws: `NetworkError` if the request fails.
func fetchUser(id: String) async throws -> User { }
```

Document every public declaration with a summary line, parameters, return value and thrown errors.

Sources: Swift API Design Guidelines (swift.org), Google Swift Style Guide.
