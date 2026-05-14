# Swift Concurrency

## Async/Await Fundamentals

Swift's modern concurrency model uses async/await for asynchronous operations:

```swift
// ✅ Async function definition
func fetchUser(id: String) async throws -> User {
  let url = URL(string: "https://api.example.com/users/\(id)")!
  let (data, _) = try await URLSession.shared.data(from: url)
  return try JSONDecoder().decode(User.self, from: data)
}

// ✅ Calling async functions
func loadUserData() async {
  do {
    let user = try await fetchUser(id: "123")
    print("User: \(user.name)")
  } catch {
    print("Error: \(error)")
  }
}

// ❌ Cannot call async function without await
func badExample() {
  let user = fetchUser(id: "123") // Compile error
}
```

## Actors for Thread Safety

Actors ensure data race safety by serializing access to mutable state:

```swift
// ✅ Actor protects mutable state
actor Counter {
  private var value = 0

  func increment() {
    value += 1
  }

  func getValue() -> Int {
    return value
  }
}

// Usage
let counter = Counter()
await counter.increment()
let value = await counter.getValue()

// ✅ Actor with async methods
actor ImageCache {
  private var cache: [String: UIImage] = [:]

  func image(for key: String) async -> UIImage? {
    if let cached = cache[key] {
      return cached
    }

    let image = await downloadImage(for: key)
    cache[key] = image
    return image
  }

  private func downloadImage(for key: String) async -> UIImage? {
    // Download implementation
    return nil
  }
}

// ❌ Don't use class with locks when actor is safer
class UnsafeCache {
  private var cache: [String: UIImage] = [:]
  private let lock = NSLock()

  func image(for key: String) -> UIImage? {
    lock.lock()
    defer { lock.unlock() }
    return cache[key]
  }
}
```

## @MainActor for UI Updates

Use @MainActor to ensure code runs on the main thread:

```swift
// ✅ Mark entire class as @MainActor
@MainActor
class UserViewModel: ObservableObject {
  @Published var user: User?
  @Published var isLoading = false

  func loadUser(id: String) async {
    isLoading = true
    defer { isLoading = false }

    do {
      user = try await fetchUser(id: id)
    } catch {
      print("Error: \(error)")
    }
  }
}

// ✅ Mark specific methods as @MainActor
class DataService {
  func fetchData() async -> Data {
    // Background work
    return Data()
  }

  @MainActor
  func updateUI(with data: Data) {
    // UI updates guaranteed on main thread
  }
}

// ✅ Mark properties as @MainActor
class ViewModel {
  @MainActor var title: String = ""

  func loadData() async {
    let data = await fetchData()
    await updateTitle(data.title)
  }

  @MainActor
  private func updateTitle(_ newTitle: String) {
    title = newTitle
  }
}

// ❌ Avoid manual dispatch to main queue
func badUpdate() async {
  let data = await fetchData()
  DispatchQueue.main.async {
    // Update UI
  }
}
```

## Structured Concurrency with Task

Use Task for structured concurrent operations:

```swift
// ✅ Simple Task creation
Task {
  let user = try await fetchUser(id: "123")
  print(user)
}

// ✅ Task with priority
Task(priority: .high) {
  let urgentData = try await fetchUrgentData()
  process(urgentData)
}

// ✅ Detached task (unstructured)
Task.detached {
  // Runs independently of current context
  await performBackgroundWork()
}

// ✅ Task cancellation
let task = Task {
  try await longRunningOperation()
}

// Later, cancel the task
task.cancel()

// ✅ Check for cancellation
func processItems(_ items: [Item]) async throws {
  for item in items {
    try Task.checkCancellation() // Throw if cancelled
    await process(item)
  }
}

// ✅ Handle cancellation gracefully
func fetchWithCancellation() async throws -> Data {
  guard !Task.isCancelled else {
    throw CancellationError()
  }

  return try await URLSession.shared.data(from: url).0
}
```

## TaskGroup for Parallel Operations

Use TaskGroup to run multiple async operations in parallel:

```swift
// ✅ Parallel data fetching
func fetchAllUsers(ids: [String]) async throws -> [User] {
  try await withThrowingTaskGroup(of: User.self) { group in
    for id in ids {
      group.addTask {
        try await fetchUser(id: id)
      }
    }

    var users: [User] = []
    for try await user in group {
      users.append(user)
    }
    return users
  }
}

// ✅ Process results as they complete
func processImages(_ urls: [URL]) async {
  await withTaskGroup(of: UIImage?.self) { group in
    for url in urls {
      group.addTask {
        await downloadImage(from: url)
      }
    }

    for await image in group {
      if let image = image {
        await display(image)
      }
    }
  }
}

// ✅ Limit concurrency with semaphore pattern
func processWithLimit(_ items: [Item], limit: Int) async {
  await withTaskGroup(of: Void.self) { group in
    var iterator = items.makeIterator()

    // Start initial batch
    for _ in 0..<min(limit, items.count) {
      if let item = iterator.next() {
        group.addTask {
          await process(item)
        }
      }
    }

    // As tasks complete, start new ones
    for await _ in group {
      if let item = iterator.next() {
        group.addTask {
          await process(item)
        }
      }
    }
  }
}

// ❌ Don't use unstructured concurrency for related tasks
func badParallelFetch(ids: [String]) async -> [User] {
  var users: [User] = []
  for id in ids {
    Task { // Unstructured - loses parent context
      if let user = try? await fetchUser(id: id) {
        users.append(user) // Data race!
      }
    }
  }
  return users
}
```

## AsyncSequence for Streaming Data

Use AsyncSequence for processing streams of data:

```swift
// ✅ Consuming AsyncSequence
func processNotifications() async {
  for await notification in notificationStream {
    handle(notification)
  }
}

// ✅ Creating custom AsyncSequence
struct Countdown: AsyncSequence {
  typealias Element = Int
  let start: Int

  struct AsyncIterator: AsyncIteratorProtocol {
    var current: Int

    mutating func next() async -> Int? {
      guard current >= 0 else { return nil }

      let value = current
      current -= 1
      try? await Task.sleep(nanoseconds: 1_000_000_000)
      return value
    }
  }

  func makeAsyncIterator() -> AsyncIterator {
    AsyncIterator(current: start)
  }
}

// Usage
for await number in Countdown(start: 5) {
  print(number)
}

// ✅ Transform AsyncSequence
func processStream() async {
  let stream = URLSession.shared.bytes(from: url)

  for try await byte in stream {
    process(byte)
  }
}

// ✅ Async sequence with map/filter
extension AsyncSequence {
  func mapAsync<T>(_ transform: @escaping (Element) async throws -> T) -> AsyncThrowingStream<T, Error> {
    AsyncThrowingStream { continuation in
      Task {
        do {
          for try await element in self {
            let transformed = try await transform(element)
            continuation.yield(transformed)
          }
          continuation.finish()
        } catch {
          continuation.finish(throwing: error)
        }
      }
    }
  }
}
```

## Continuations for Bridging Callback APIs

Use continuations to wrap callback-based APIs:

```swift
// ✅ Wrap completion handler with continuation
func fetchData() async throws -> Data {
  try await withCheckedThrowingContinuation { continuation in
    legacyFetchData { result in
      switch result {
      case .success(let data):
        continuation.resume(returning: data)
      case .failure(let error):
        continuation.resume(throwing: error)
      }
    }
  }
}

// ✅ Wrap delegate callback
func requestPermission() async -> Bool {
  await withCheckedContinuation { continuation in
    permissionManager.request { granted in
      continuation.resume(returning: granted)
    }
  }
}

// ⚠️ Use unsafe continuation only when necessary
func unsafeExample() async -> String {
  await withUnsafeContinuation { continuation in
    // Must call resume exactly once!
    continuation.resume(returning: "value")
  }
}

// ❌ Never resume continuation multiple times
func badContinuation() async {
  await withCheckedContinuation { continuation in
    continuation.resume(returning: 1)
    continuation.resume(returning: 2) // Runtime error!
  }
}
```

## Sendable Protocol

Use Sendable to ensure types can be safely passed across concurrency boundaries:

```swift
// ✅ Value types are implicitly Sendable
struct User: Sendable {
  let id: String
  let name: String
}

// ✅ Immutable reference types can be Sendable
final class Configuration: Sendable {
  let apiKey: String
  let baseURL: URL

  init(apiKey: String, baseURL: URL) {
    self.apiKey = apiKey
    self.baseURL = baseURL
  }
}

// ✅ Actor types are implicitly Sendable
actor DatabaseManager: Sendable {
  func save(_ item: Item) async throws {
    // Implementation
  }
}

// ❌ Mutable classes should not be Sendable
class MutableState: Sendable { // Warning!
  var count = 0 // Unsafe across concurrency boundaries
}

// ✅ Use actor instead
actor SafeState {
  var count = 0

  func increment() {
    count += 1
  }
}
```

## Best Practices

### Avoid Data Races

```swift
// ❌ Data race with shared mutable state
class Counter {
  var value = 0

  func increment() {
    value += 1 // Unsafe if called from multiple tasks
  }
}

// ✅ Use actor for shared mutable state
actor Counter {
  var value = 0

  func increment() {
    value += 1 // Safe - actor serializes access
  }
}
```

### Prefer Structured Concurrency

```swift
// ❌ Unstructured tasks lose parent context
func badExample() {
  Task {
    await work1()
  }
  Task {
    await work2()
  }
  // Returns immediately, tasks run independently
}

// ✅ Structured concurrency with async let
func goodExample() async {
  async let result1 = work1()
  async let result2 = work2()

  let (r1, r2) = await (result1, result2)
  // Both complete before function returns
}
```

### Use async let for Independent Operations

```swift
// ✅ Parallel execution with async let
func loadDashboard() async throws -> Dashboard {
  async let user = fetchUser()
  async let posts = fetchPosts()
  async let stats = fetchStats()

  return try await Dashboard(
    user: user,
    posts: posts,
    stats: stats
  )
}

// ❌ Sequential execution (slower)
func slowDashboard() async throws -> Dashboard {
  let user = try await fetchUser()
  let posts = try await fetchPosts()
  let stats = try await fetchStats()

  return Dashboard(user: user, posts: posts, stats: stats)
}
```

### Handle Task Cancellation

```swift
// ✅ Respect cancellation
func processItems(_ items: [Item]) async throws {
  for item in items {
    try Task.checkCancellation()
    await process(item)
  }
}

// ✅ Clean up on cancellation
func downloadFile() async throws -> URL {
  let task = Task {
    try await actualDownload()
  }

  defer {
    if Task.isCancelled {
      cleanupPartialDownload()
    }
  }

  return try await task.value
}
```

### Avoid Blocking the Main Thread

```swift
// ❌ Blocking UI thread
@MainActor
func loadData() {
  let data = fetchDataSynchronously() // Blocks UI!
  updateUI(data)
}

// ✅ Use async/await
@MainActor
func loadData() async {
  let data = await fetchDataAsynchronously()
  updateUI(data) // Already on main actor
}
```

## Common Patterns

### Retry with Backoff

```swift
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

### Timeout

```swift
func withTimeout<T>(
  seconds: TimeInterval,
  operation: @escaping () async throws -> T
) async throws -> T {
  try await withThrowingTaskGroup(of: T.self) { group in
    group.addTask {
      try await operation()
    }

    group.addTask {
      try await Task.sleep(nanoseconds: UInt64(seconds * 1_000_000_000))
      throw TimeoutError()
    }

    let result = try await group.next()!
    group.cancelAll()
    return result
  }
}

// Usage
let user = try await withTimeout(seconds: 5) {
  try await fetchUser(id: "123")
}
```

### Debounce

```swift
actor Debouncer<T: Sendable> {
  private var task: Task<T, Error>?
  private let delay: Duration
  private let operation: @Sendable () async throws -> T

  init(delay: Duration, operation: @escaping @Sendable () async throws -> T) {
    self.delay = delay
    self.operation = operation
  }

  func call() async throws -> T {
    task?.cancel()

    let newTask = Task {
      try await Task.sleep(for: delay)
      return try await operation()
    }

    task = newTask
    return try await newTask.value
  }
}

// Usage
let searchDebouncer = Debouncer(delay: .milliseconds(300)) {
  await searchAPI(query: currentQuery)
}

// Only last call within 300ms executes
try await searchDebouncer.call()
```

---

**Sources:**
- [Swift Evolution: Concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [Swift.org: Concurrency](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)
- [WWDC21: Meet async/await in Swift](https://developer.apple.com/videos/play/wwdc2021/10132/)
- [WWDC21: Protect mutable state with Swift actors](https://developer.apple.com/videos/play/wwdc2021/10133/)
