# Swift Testing

## Rules

- Use the Swift Testing framework (`import Testing`, Swift 6 / Xcode 16+) for new unit tests. Keep XCTest for UI tests, performance tests (`measure`) and existing suites; both run side by side in one target.
- One behavior per test, with a name that states it. Tests must not depend on order or shared mutable state: each test builds its own inputs.
- Inject dependencies through `init` using protocols, and replace them with fakes. Do not hit the network or real disk in unit tests.
- Use `#expect` for checks that should keep running after a failure, and `#require` when the rest of the test cannot continue (unwrapping, preconditions).
- Cover empty, boundary and error inputs, not only the happy path.
- Mark view-model tests `@MainActor`. Never use `sleep` to wait; `await` the work.

## Swift Testing basics

```swift
import Testing

@Test func additionWorks() {
  #expect(Calculator().add(2, 3) == 5)
}

@Test("Division by zero throws")
func divisionByZero() {
  #expect(throws: DivisionError.self) {
    try Calculator().divide(10, by: 0)
  }
}

@Test func fetchUserReturnsRequestedID() async throws {
  let user = try #require(try await service.fetchUser(id: "123"))
  #expect(user.id == "123")
  #expect(!user.name.isEmpty, "User name must be populated")
}
```

- `#expect(throws: SomeError.self) { ... }` asserts a specific error type; pass an error value to match that exact error. Await it for async bodies: `await #expect(throws: AuthError.self) { try await login() }`.
- Put a message as the second `#expect` argument when the failure would otherwise be unclear.

## Suites, setup and traits

```swift
@Suite("Authentication")
struct AuthTests {
  let service: AuthService          // fresh instance for every test

  init() async throws {             // runs before each test
    service = AuthService(store: InMemoryStore())
  }

  @Test(.tags(.network), .timeLimit(.minutes(1)))
  func loginSucceeds() async throws {
    let session = try await service.login(email: "a@example.com", password: "pw")
    #expect(session.isActive)
  }

  @Test(.disabled("Flaky until the clock is injectable"))
  func tokenExpires() { }
}

extension Tag {
  @Tag static var network: Self
}
```

Each test gets a new suite instance, so use `init` (and `deinit` for classes) instead of shared setup state. Use `.tags` to select groups from the command line or Xcode test plans.

## Parameterized tests

```swift
@Test(arguments: ["alice@example.com", "bob@test.co.uk", "user+tag@mail.com"])
func validEmailsPass(email: String) {
  #expect(EmailValidator.isValid(email))
}

struct Case: Sendable {
  let input: String
  let expected: Bool
}

@Test(arguments: [
  Case(input: "hello", expected: true),
  Case(input: "", expected: false),
])
func validation(_ testCase: Case) {
  #expect(validate(testCase.input) == testCase.expected)
}
```

- Passing two collections runs every combination. Use `zip(a, b)` to pair values one to one.
- Each argument runs as its own test case, so one failing input does not hide the others.

## Fakes and spies

```swift
protocol UserRepository: Sendable {
  func fetchUser(id: String) async throws -> User
}

final class FakeUserRepository: UserRepository, @unchecked Sendable {
  var users: [String: User] = [:]
  private(set) var fetchedIDs: [String] = []

  func fetchUser(id: String) async throws -> User {
    fetchedIDs.append(id)
    guard let user = users[id] else { throw RepositoryError.notFound }
    return user
  }
}

@Test func serviceReturnsRepositoryUser() async throws {
  let repo = FakeUserRepository()
  repo.users["123"] = User(id: "123", name: "Alice")

  let user = try await UserService(repository: repo).getUser(id: "123")

  #expect(user.name == "Alice")
  #expect(repo.fetchedIDs == ["123"])
}
```

Prefer a hand-written fake that returns canned data and records calls over a heavy mocking setup. Assert on results first, and on interactions only when the call itself is the behavior.

## Testing view models

```swift
@Suite @MainActor
struct UserListViewModelTests {
  @Test func loadPublishesUsers() async {
    let repo = FakeUserRepository()
    repo.listResult = .success([User(id: "1", name: "Alice")])
    let viewModel = UserListViewModel(repository: repo)

    await viewModel.load()

    #expect(viewModel.users.map(\.name) == ["Alice"])
    #expect(!viewModel.isLoading)
    #expect(viewModel.errorMessage == nil)
  }

  @Test func loadFailureSetsErrorMessage() async {
    let repo = FakeUserRepository()
    repo.listResult = .failure(NetworkError.connectionFailed)
    let viewModel = UserListViewModel(repository: repo)

    await viewModel.load()

    #expect(viewModel.users.isEmpty)
    #expect(viewModel.errorMessage != nil)
  }
}
```

## XCTest (existing suites, UI and performance)

```swift
import XCTest

final class UserServiceTests: XCTestCase {
  var service: UserService!

  override func setUp() {
    super.setUp()
    service = UserService(repository: FakeUserRepository())
  }

  override func tearDown() {
    service = nil
    super.tearDown()
  }

  func testFetchUserSucceeds() async throws {
    let user = try await service.fetchUser(id: "123")
    XCTAssertEqual(user.id, "123")
  }

  func testSortingPerformance() {
    let values = (0..<10_000).shuffled()
    measure { _ = values.sorted() }
  }
}
```

- Test methods start with `test`. Prefer `async throws` test methods over expectations; use `expectation(description:)` and `wait(for:timeout:)` only for callback APIs.
- Common assertions: `XCTAssertEqual`, `XCTAssertTrue`, `XCTAssertNil`, `XCTAssertThrowsError`, `XCTAssertNoThrow`. Add a message argument for non-obvious checks.
- For async code that must throw, use `do { try await op(); XCTFail("expected error") } catch { ... }`.
- Swift Testing has no `measure`; keep performance tests in XCTest.

## Layout

```
Tests/
├── UnitTests/       Models/ Services/ ViewModels/
├── IntegrationTests/
└── TestHelpers/     Fakes.swift, Fixtures.swift
```

Build test data with small factory helpers with default arguments (`makeUser(name: "Test")`) so each test states only what it cares about.

Sources: Apple documentation for Swift Testing and XCTest; WWDC24 session 10179.
