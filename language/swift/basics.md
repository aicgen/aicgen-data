# Swift Language Fundamentals

## Naming Conventions

Follow Swift's standard naming conventions for clarity and consistency:

```swift
// ✅ Types and protocols: UpperCamelCase
class UserManager { }
struct UserProfile { }
protocol Drawable { }
enum NetworkError { }

// ✅ Functions, variables, constants: lowerCamelCase
func calculateTotal(items: [Item]) -> Double { }
let userName = "Alice"
var isLoggedIn = false
private let maxRetries = 3

// ✅ Enums: singular, UpperCamelCase
enum Result {
  case success
  case failure(Error)
}

// ❌ Bad: Incorrect casing
class user_manager { } // Wrong
func Calculate_Total() { } // Wrong
let MAX_RETRIES = 3 // Use camelCase for constants
```

## Type Safety and Type Inference

Leverage Swift's powerful type system while maintaining code clarity:

```swift
// ✅ Type inference for obvious types
let name = "Alice" // Clearly a String
let count = 5 // Clearly an Int
let items = [1, 2, 3] // Clearly [Int]

// ✅ Explicit types when beneficial
let apiKey: String? = configuration.apiKey
let timeout: TimeInterval = 30.0
let callback: (Result<Data, Error>) -> Void

// ✅ Always specify return types for functions
func fetchUser(id: String) -> User? {
  // Implementation
}

// ❌ Avoid unnecessary type annotations
let message: String = "Hello" // Type is obvious
```

## Value Types vs Reference Types

Prefer structs (value types) over classes (reference types) when possible:

```swift
// ✅ Use structs for simple data models
struct User {
  let id: String
  var name: String
  var email: String
}

// ✅ Use classes when you need:
// - Inheritance
// - Reference semantics
// - Deinitializers
class NetworkManager {
  private var session: URLSession

  init(configuration: URLSessionConfiguration) {
    self.session = URLSession(configuration: configuration)
  }

  deinit {
    session.invalidateAndCancel()
  }
}

// ✅ Structs are copied, classes are referenced
var user1 = User(id: "1", name: "Alice", email: "alice@example.com")
var user2 = user1
user2.name = "Bob"
// user1.name is still "Alice" (different instance)
```

## Properties and Computed Properties

Use appropriate property types based on your needs:

```swift
struct Rectangle {
  // ✅ Stored properties
  var width: Double
  var height: Double

  // ✅ Computed property for derived values
  var area: Double {
    return width * height
  }

  // ✅ Computed property with setter
  var perimeter: Double {
    get {
      return 2 * (width + height)
    }
    set {
      // Distribute new perimeter equally
      let side = newValue / 4
      width = side
      height = side
    }
  }

  // ✅ Property observers
  var description: String = "" {
    willSet {
      print("About to set description to: \(newValue)")
    }
    didSet {
      print("Description changed from: \(oldValue)")
    }
  }
}

// ❌ Avoid storing values that can be computed
struct Circle {
  var radius: Double
  var diameter: Double // ❌ Should be computed
  var area: Double // ❌ Should be computed
}

// ✅ Better approach
struct Circle {
  var radius: Double

  var diameter: Double {
    return radius * 2
  }

  var area: Double {
    return .pi * radius * radius
  }
}
```

## Access Control

Use appropriate access levels to maintain encapsulation:

```swift
// ✅ Public: Available to all modules
public class APIClient {
  // ✅ Internal (default): Available within module
  func fetchData() -> Data? {
    return nil
  }

  // ✅ Private: Only visible within this type
  private var apiKey: String = ""

  // ✅ Fileprivate: Visible within this file
  fileprivate func log(_ message: String) {
    print(message)
  }
}

// ✅ Keep implementation details private
public struct User {
  public let id: String
  public var name: String

  // ❌ Don't expose internal state unnecessarily
  // public var internalCache: [String: Any] = [:]

  // ✅ Keep internal state private
  private var cache: [String: Any] = [:]

  public init(id: String, name: String) {
    self.id = id
    self.name = name
  }
}
```

## Extensions for Organization

Use extensions to organize code and add functionality:

```swift
// ✅ Protocol conformance in extensions
struct User {
  let id: String
  var name: String
}

extension User: Codable { }

extension User: Equatable {
  static func == (lhs: User, rhs: User) -> Bool {
    return lhs.id == rhs.id
  }
}

// ✅ Computed properties in extensions
extension User {
  var displayName: String {
    return name.isEmpty ? "Unknown User" : name
  }
}

// ✅ Methods in extensions
extension User {
  func isValid() -> Bool {
    return !id.isEmpty && !name.isEmpty
  }
}

// ✅ Extend types you don't own
extension String {
  var isValidEmail: Bool {
    return self.contains("@") && self.contains(".")
  }
}
```

## Guard Statements for Early Returns

Use `guard` for early returns and validation:

```swift
// ✅ Use guard for preconditions
func processUser(_ user: User?) {
  guard let user = user else {
    print("No user provided")
    return
  }

  guard user.isValid() else {
    print("Invalid user")
    return
  }

  guard user.age >= 18 else {
    print("User must be 18 or older")
    return
  }

  // Happy path at the lowest indentation level
  print("Processing user: \(user.name)")
}

// ❌ Avoid deep nesting with if-let
func processUser(_ user: User?) {
  if let user = user {
    if user.isValid() {
      if user.age >= 18 {
        print("Processing user: \(user.name)")
      }
    }
  }
}
```

## Modern Swift Types

Use Swift's modern collection types and standard library:

```swift
// ✅ Use Swift types, not Objective-C types
let names: [String] = ["Alice", "Bob"] // Not NSArray
let scores: [String: Int] = ["Alice": 95] // Not NSDictionary
let uniqueIds: Set<String> = ["id1", "id2"] // Not NSSet

// ✅ Use Swift's String, not NSString
let message: String = "Hello, World!"
let uppercased = message.uppercased() // Not message.uppercaseString

// ✅ Leverage collection methods
let numbers = [1, 2, 3, 4, 5]
let doubled = numbers.map { $0 * 2 }
let evens = numbers.filter { $0 % 2 == 0 }
let sum = numbers.reduce(0, +)

// ✅ Use compactMap to filter and transform
let strings = ["1", "2", "foo", "3"]
let validNumbers = strings.compactMap { Int($0) }
// Result: [1, 2, 3]
```

## Immutability with let

Prefer `let` over `var` for immutable values:

```swift
// ✅ Use let whenever possible
let maxRetries = 3
let timeout: TimeInterval = 30.0
let apiEndpoint = "https://api.example.com"

// ✅ Only use var when value will change
var currentAttempt = 0
var isLoading = false

// ✅ Immutable collections are safer
let users: [User] = fetchUsers()
// users.append(newUser) // Compile error - safer!

// ❌ Avoid var for values that don't change
var serverURL = "https://api.example.com" // Should be let
```

## Trailing Closures

Use trailing closure syntax for better readability:

```swift
// ✅ Trailing closure for single closure parameter
UIView.animate(withDuration: 0.3) {
  view.alpha = 0
}

// ✅ Multiple trailing closures (Swift 5.3+)
UIView.animate(withDuration: 0.3) {
  view.alpha = 0
} completion: { _ in
  view.removeFromSuperview()
}

// ❌ Avoid when it reduces clarity
array.map({ $0 * 2 }) // OK - simple and clear
```

## Avoid Force Unwrapping

Never use force unwrapping (`!`) unless absolutely necessary:

```swift
// ❌ Force unwrapping can crash
let user = optionalUser!
let name = user.name!

// ✅ Optional binding
if let user = optionalUser {
  print(user.name)
}

// ✅ Guard for early exit
guard let user = optionalUser else {
  return
}
print(user.name)

// ✅ Nil coalescing for defaults
let name = optionalName ?? "Unknown"

// ✅ Optional chaining
let firstUserEmail = users.first?.email

// ⚠️ Force unwrap only when guaranteed non-nil
// (e.g., outlets in view controllers after viewDidLoad)
@IBOutlet private weak var titleLabel: UILabel!
```

## Type Aliases for Clarity

Use type aliases to make complex types more readable:

```swift
// ✅ Clarify complex types
typealias CompletionHandler = (Result<Data, Error>) -> Void
typealias UserID = String
typealias Coordinate = (latitude: Double, longitude: Double)

// Usage
func fetchUser(id: UserID, completion: CompletionHandler) {
  // Implementation
}

// ✅ Document units and intent
typealias Seconds = Double
typealias Meters = Double

let timeout: Seconds = 30.0
let distance: Meters = 100.0
```

## Enums for Type Safety

Use enums to represent a fixed set of related values:

```swift
// ✅ Simple enum
enum NetworkError: Error {
  case noConnection
  case timeout
  case invalidResponse
  case unauthorized
}

// ✅ Enum with associated values
enum Result<Success, Failure: Error> {
  case success(Success)
  case failure(Failure)
}

// ✅ Enum with raw values
enum HTTPMethod: String {
  case get = "GET"
  case post = "POST"
  case put = "PUT"
  case delete = "DELETE"
}

// ✅ Exhaustive switch statements
func handle(error: NetworkError) {
  switch error {
  case .noConnection:
    print("No internet connection")
  case .timeout:
    print("Request timed out")
  case .invalidResponse:
    print("Invalid response from server")
  case .unauthorized:
    print("Unauthorized access")
  }
  // No default needed - all cases covered
}
```

## Documentation Comments

Write clear documentation using Swift's markdown format:

```swift
/// Fetches user data from the API.
///
/// - Parameters:
///   - id: The unique identifier for the user
///   - completion: Closure called when the request completes
/// - Returns: A cancellable task
/// - Throws: `NetworkError` if the request fails
func fetchUser(
  id: String,
  completion: @escaping (Result<User, NetworkError>) -> Void
) -> URLSessionTask {
  // Implementation
}

/// Represents a user in the system.
///
/// Use this type to store user information retrieved from the API.
///
/// Example:
/// ```swift
/// let user = User(id: "123", name: "Alice")
/// print(user.displayName)
/// ```
struct User {
  /// The unique identifier for this user
  let id: String

  /// The user's display name
  var name: String
}
```

---

**Sources:**
- [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [Google Swift Style Guide](https://google.github.io/swift/)
- [Kodeco Swift Style Guide](https://github.com/kodecocodes/swift-style-guide)
