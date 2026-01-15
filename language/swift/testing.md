# Swift Testing

## Swift Testing Framework (Swift 6+)

The new Swift Testing framework (introduced at WWDC24) provides a modern, expressive API for testing:

```swift
import Testing

// ✅ Simple test with @Test macro
@Test func additionWorks() {
  #expect(2 + 2 == 4)
}

// ✅ Test with description
@Test("Addition produces correct results")
func testAddition() {
  let result = Calculator().add(2, 3)
  #expect(result == 5)
}

// ✅ Async test
@Test func fetchUserReturnsValidData() async throws {
  let user = try await fetchUser(id: "123")
  #expect(user.id == "123")
  #expect(!user.name.isEmpty)
}

// ✅ Test with throws
@Test func divisionByZeroThrows() throws {
  #expect(throws: DivisionError.self) {
    try Calculator().divide(10, by: 0)
  }
}
```

## Test Suites

Organize tests into logical groups:

```swift
import Testing

// ✅ Test suite with @Suite macro
@Suite("Calculator Tests")
struct CalculatorTests {
  let calculator = Calculator()

  @Test("Addition")
  func testAddition() {
    #expect(calculator.add(2, 3) == 5)
  }

  @Test("Subtraction")
  func testSubtraction() {
    #expect(calculator.subtract(5, 3) == 2)
  }
}

// ✅ Nested suites for organization
@Suite("User Management")
struct UserTests {
  @Suite("Authentication")
  struct AuthTests {
    @Test func loginSucceeds() async throws {
      let result = try await login(email: "test@example.com", password: "password")
      #expect(result.isSuccess)
    }

    @Test func loginFailsWithInvalidCredentials() async throws {
      await #expect(throws: AuthError.self) {
        try await login(email: "test@example.com", password: "wrong")
      }
    }
  }

  @Suite("Profile Management")
  struct ProfileTests {
    @Test func updateProfileSucceeds() async throws {
      let user = try await updateProfile(name: "New Name")
      #expect(user.name == "New Name")
    }
  }
}
```

## Parameterized Tests

Test multiple scenarios efficiently:

```swift
import Testing

// ✅ Test with multiple arguments
@Test(arguments: [
  (2, 3, 5),
  (0, 0, 0),
  (-1, 1, 0),
  (100, 200, 300)
])
func additionWorks(a: Int, b: Int, expected: Int) {
  let calculator = Calculator()
  #expect(calculator.add(a, b) == expected)
}

// ✅ Test with array of values
@Test(arguments: ["alice@example.com", "bob@test.com", "charlie@mail.com"])
func emailValidationWorks(email: String) {
  #expect(EmailValidator.isValid(email))
}

// ✅ Test combinations
@Test(arguments: [true, false], [1, 2, 3])
func featureWorksInAllCombinations(flag: Bool, value: Int) {
  let result = processFeature(enabled: flag, value: value)
  #expect(result != nil)
}

// ✅ Test with custom types
struct TestCase {
  let input: String
  let expected: Bool
}

@Test(arguments: [
  TestCase(input: "hello", expected: true),
  TestCase(input: "", expected: false),
  TestCase(input: "a", expected: true)
])
func validation(testCase: TestCase) {
  #expect(validate(testCase.input) == testCase.expected)
}
```

## Expectations and Assertions

### Basic Expectations

```swift
import Testing

// ✅ Boolean expectations
#expect(value == 5)
#expect(value > 0)
#expect(name.isEmpty)
#expect(!isLoading)

// ✅ Optional expectations
#expect(optionalValue != nil)
#expect(users.first?.name == "Alice")

// ✅ Collection expectations
#expect(array.count == 3)
#expect(array.contains(item))
#expect(dictionary["key"] == "value")
```

### Error Expectations

```swift
// ✅ Expect specific error type
#expect(throws: NetworkError.self) {
  try performNetworkRequest()
}

// ✅ Expect any error
#expect(throws: Error.self) {
  try riskyOperation()
}

// ✅ Expect no error (default behavior)
#expect {
  try safeOperation()
}

// ✅ Async error expectations
await #expect(throws: APIError.unauthorized) {
  try await fetchProtectedResource()
}
```

### Custom Error Messages

```swift
// ✅ Add context to failures
#expect(user.age >= 18, "User must be at least 18 years old")
#expect(items.count > 0, "Shopping cart cannot be empty")
```

## Test Lifecycle

### Setup and Teardown

```swift
import Testing

@Suite("Database Tests")
struct DatabaseTests {
  let database: Database

  // ✅ Initialize resources before each test
  init() async throws {
    database = try await Database.connect()
  }

  // ✅ Clean up after tests
  deinit {
    database.disconnect()
  }

  @Test func insertRecordWorks() async throws {
    try await database.insert(record)
    let fetched = try await database.fetch(id: record.id)
    #expect(fetched == record)
  }
}
```

## XCTest (Traditional Framework)

For backwards compatibility and existing codebases:

```swift
import XCTest

// ✅ XCTest class
class UserServiceTests: XCTestCase {
  var service: UserService!

  // Setup before each test
  override func setUp() {
    super.setUp()
    service = UserService()
  }

  // Teardown after each test
  override func tearDown() {
    service = nil
    super.tearDown()
  }

  // ✅ Test method (must start with "test")
  func testFetchUserSucceeds() async throws {
    let user = try await service.fetchUser(id: "123")
    XCTAssertEqual(user.id, "123")
    XCTAssertFalse(user.name.isEmpty)
  }

  // ✅ Test expected error
  func testFetchUserThrowsForInvalidID() async {
    await XCTAssertThrowsError(try await service.fetchUser(id: "invalid")) { error in
      XCTAssertTrue(error is ServiceError)
    }
  }

  // ✅ Async test
  func testAsyncOperation() async throws {
    let result = try await service.performAsyncOperation()
    XCTAssertTrue(result.success)
  }
}
```

## Common XCTest Assertions

```swift
// Equality
XCTAssertEqual(actual, expected)
XCTAssertNotEqual(actual, unexpected)

// Boolean
XCTAssertTrue(condition)
XCTAssertFalse(condition)

// Nil checks
XCTAssertNil(optionalValue)
XCTAssertNotNil(optionalValue)

// Errors
XCTAssertThrowsError(try dangerousOperation())
XCTAssertNoThrow(try safeOperation())

// Numeric comparisons
XCTAssertGreaterThan(value, 0)
XCTAssertLessThan(value, 100)

// Custom messages
XCTAssertEqual(user.age, 18, "User age should be 18")
```

## Testing Async Code

### Swift Testing Framework

```swift
import Testing

// ✅ Simple async test
@Test func fetchDataCompletes() async throws {
  let data = try await fetchData()
  #expect(!data.isEmpty)
}

// ✅ Test with timeout (future feature)
@Test(.timeLimit(.seconds(5)))
func slowOperationCompletes() async throws {
  try await performSlowOperation()
}

// ✅ Test concurrent operations
@Test func concurrentFetchesSucceed() async throws {
  async let user1 = fetchUser(id: "1")
  async let user2 = fetchUser(id: "2")
  async let user3 = fetchUser(id: "3")

  let users = try await [user1, user2, user3]
  #expect(users.count == 3)
}
```

### XCTest with Async

```swift
import XCTest

class AsyncTests: XCTestCase {
  // ✅ Async test method
  func testAsyncOperation() async throws {
    let result = try await fetchData()
    XCTAssertFalse(result.isEmpty)
  }

  // ✅ Test with expectation (older pattern)
  func testCallbackCompletion() {
    let expectation = expectation(description: "Callback called")

    fetchDataWithCallback { result in
      XCTAssertNotNil(result)
      expectation.fulfill()
    }

    waitForExpectations(timeout: 5)
  }
}
```

## Mocking and Test Doubles

### Protocol-Based Mocking

```swift
// ✅ Define protocol for dependency
protocol UserRepository {
  func fetchUser(id: String) async throws -> User
  func saveUser(_ user: User) async throws
}

// Real implementation
class ProductionUserRepository: UserRepository {
  func fetchUser(id: String) async throws -> User {
    // Real API call
  }

  func saveUser(_ user: User) async throws {
    // Real save
  }
}

// ✅ Mock for testing
class MockUserRepository: UserRepository {
  var fetchUserCalled = false
  var userToReturn: User?
  var errorToThrow: Error?

  func fetchUser(id: String) async throws -> User {
    fetchUserCalled = true

    if let error = errorToThrow {
      throw error
    }

    guard let user = userToReturn else {
      throw MockError.noUserSet
    }

    return user
  }

  func saveUser(_ user: User) async throws {
    // Store for verification
  }
}

// Test usage
@Test func serviceUsesMockRepository() async throws {
  let mockRepo = MockUserRepository()
  mockRepo.userToReturn = User(id: "123", name: "Alice")

  let service = UserService(repository: mockRepo)
  let user = try await service.getUser(id: "123")

  #expect(mockRepo.fetchUserCalled)
  #expect(user.name == "Alice")
}
```

### Spy Pattern

```swift
// ✅ Spy records method calls
class SpyUserRepository: UserRepository {
  var fetchUserCallCount = 0
  var fetchedUserIDs: [String] = []
  var savedUsers: [User] = []

  func fetchUser(id: String) async throws -> User {
    fetchUserCallCount += 1
    fetchedUserIDs.append(id)
    return User(id: id, name: "Test User")
  }

  func saveUser(_ user: User) async throws {
    savedUsers.append(user)
  }
}

@Test func serviceCallsRepositoryCorrectly() async throws {
  let spy = SpyUserRepository()
  let service = UserService(repository: spy)

  try await service.getUser(id: "123")
  try await service.getUser(id: "456")

  #expect(spy.fetchUserCallCount == 2)
  #expect(spy.fetchedUserIDs == ["123", "456"])
}
```

## Testing Best Practices

### Write Focused Tests

```swift
// ❌ Testing too much in one test
@Test func userManagement() async throws {
  let user = try await createUser(name: "Alice")
  let updated = try await updateUser(user, name: "Bob")
  try await deleteUser(updated.id)
  let fetched = try? await fetchUser(id: updated.id)
  #expect(fetched == nil)
}

// ✅ One test per behavior
@Test func createUserSucceeds() async throws {
  let user = try await createUser(name: "Alice")
  #expect(user.name == "Alice")
}

@Test func updateUserChangesName() async throws {
  let user = try await createUser(name: "Alice")
  let updated = try await updateUser(user, name: "Bob")
  #expect(updated.name == "Bob")
}

@Test func deleteUserRemovesFromDatabase() async throws {
  let user = try await createUser(name: "Alice")
  try await deleteUser(user.id)
  await #expect(throws: NotFoundError.self) {
    try await fetchUser(id: user.id)
  }
}
```

### Test Edge Cases

```swift
@Suite("String Validation Tests")
struct ValidationTests {
  @Test func validEmailPasses() {
    #expect(validate("test@example.com"))
  }

  @Test func emptyStringFails() {
    #expect(!validate(""))
  }

  @Test func stringWithoutAtSymbolFails() {
    #expect(!validate("notanemail"))
  }

  @Test func veryLongEmailFails() {
    let longEmail = String(repeating: "a", count: 300) + "@example.com"
    #expect(!validate(longEmail))
  }

  @Test func emailWithSpecialCharactersWorks() {
    #expect(validate("user+tag@example.co.uk"))
  }
}
```

### Use Descriptive Names

```swift
// ❌ Unclear test names
@Test func test1() { }
@Test func testUser() { }

// ✅ Descriptive names that explain what is tested
@Test("User registration succeeds with valid email")
func userRegistrationWithValidEmail() { }

@Test("Login fails when password is incorrect")
func loginWithIncorrectPassword() { }

@Test("Shopping cart total includes tax")
func cartTotalCalculation() { }
```

### Avoid Test Interdependence

```swift
// ❌ Tests depend on execution order
class BadTests: XCTestCase {
  static var userId: String?

  func test1_createUser() {
    BadTests.userId = createUser().id
  }

  func test2_updateUser() {
    updateUser(id: BadTests.userId!) // Fails if test1 doesn't run first
  }
}

// ✅ Each test is independent
@Suite("User Tests")
struct GoodTests {
  @Test func userCreation() async throws {
    let user = try await createUser(name: "Alice")
    #expect(user.name == "Alice")
  }

  @Test func userUpdate() async throws {
    // Create user within this test
    let user = try await createUser(name: "Alice")
    let updated = try await updateUser(user, name: "Bob")
    #expect(updated.name == "Bob")
  }
}
```

## Testing View Models

```swift
import Testing

@Suite("User View Model Tests")
@MainActor
struct UserViewModelTests {
  @Test func loadingUsersUpdatesState() async {
    let mockRepo = MockUserRepository()
    mockRepo.usersToReturn = [
      User(id: "1", name: "Alice"),
      User(id: "2", name: "Bob")
    ]

    let viewModel = UserViewModel(repository: mockRepo)
    await viewModel.loadUsers()

    #expect(viewModel.users.count == 2)
    #expect(!viewModel.isLoading)
    #expect(viewModel.errorMessage == nil)
  }

  @Test func loadingUsersHandlesErrors() async {
    let mockRepo = MockUserRepository()
    mockRepo.errorToThrow = NetworkError.connectionFailed

    let viewModel = UserViewModel(repository: mockRepo)
    await viewModel.loadUsers()

    #expect(viewModel.users.isEmpty)
    #expect(!viewModel.isLoading)
    #expect(viewModel.errorMessage != nil)
  }

  @Test func deletingUserRemovesFromList() async {
    let mockRepo = MockUserRepository()
    let user1 = User(id: "1", name: "Alice")
    let user2 = User(id: "2", name: "Bob")
    mockRepo.usersToReturn = [user1, user2]

    let viewModel = UserViewModel(repository: mockRepo)
    await viewModel.loadUsers()
    await viewModel.deleteUser(user1)

    #expect(viewModel.users.count == 1)
    #expect(viewModel.users.first?.id == "2")
  }
}
```

## Performance Testing

### Swift Testing

```swift
import Testing

// ✅ Measure performance
@Test func sortingLargeArray() {
  let array = (0..<10000).shuffled()
  measure {
    _ = array.sorted()
  }
}
```

### XCTest Performance

```swift
import XCTest

class PerformanceTests: XCTestCase {
  func testSortingPerformance() {
    let array = (0..<10000).shuffled()

    measure {
      _ = array.sorted()
    }
  }

  func testDatabaseQueryPerformance() async {
    await measureAsync {
      try? await database.fetchAllRecords()
    }
  }
}

extension XCTestCase {
  func measureAsync(_ block: @escaping () async -> Void) async {
    measure {
      let expectation = expectation(description: "Async operation")
      Task {
        await block()
        expectation.fulfill()
      }
      wait(for: [expectation], timeout: 10)
    }
  }
}
```

## Test Organization

### File Structure

```
Tests/
├── UnitTests/
│   ├── Models/
│   │   ├── UserTests.swift
│   │   └── ProductTests.swift
│   ├── Services/
│   │   ├── UserServiceTests.swift
│   │   └── AuthServiceTests.swift
│   └── ViewModels/
│       ├── UserViewModelTests.swift
│       └── ProductViewModelTests.swift
├── IntegrationTests/
│   ├── APITests.swift
│   └── DatabaseTests.swift
└── TestHelpers/
    ├── Mocks.swift
    └── Fixtures.swift
```

### Shared Test Utilities

```swift
// ✅ Create reusable test helpers
enum TestFixtures {
  static func makeUser(id: String = "123", name: String = "Test User") -> User {
    User(id: id, name: name, email: "\(name)@test.com")
  }

  static func makeUsers(count: Int) -> [User] {
    (0..<count).map { i in
      makeUser(id: "\(i)", name: "User \(i)")
    }
  }
}

// Usage in tests
@Test func processMultipleUsers() {
  let users = TestFixtures.makeUsers(count: 5)
  #expect(users.count == 5)
}
```

---

**Sources:**
- [Swift Testing (WWDC24)](https://developer.apple.com/videos/play/wwdc2024/10179/)
- [Apple XCTest Documentation](https://developer.apple.com/documentation/xctest)
- [Testing Swift](https://www.swiftbysundell.com/basics/testing/)
- [Swift.org: Testing](https://www.swift.org/documentation/testing/)
