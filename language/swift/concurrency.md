# Swift Concurrency

## Rules

- Prefer structured concurrency (`async let`, `TaskGroup`) over unstructured `Task { }` and `Task.detached`. Child tasks inherit cancellation, priority and actor context; detached tasks do not.
- Protect shared mutable state with an `actor` or `@MainActor` isolation, not locks or dispatch queues.
- Mark UI-facing types (`ObservableObject` view models, anything that touches views) `@MainActor`. Do not call `DispatchQueue.main.async` from async code.
- Types that cross concurrency boundaries must be `Sendable`: value types of Sendable members, immutable `final` classes, or actors. Never mark a class with mutable state `Sendable` (except `@unchecked Sendable` with a documented lock).
- Do not block: no `Thread.sleep`, semaphores or `DispatchGroup.wait()` in async code. Use `Task.sleep(for:)`.
- Support cancellation: call `try Task.checkCancellation()` in loops and clean up in `defer`.
- Build with Swift 6 language mode (strict concurrency) for new code and fix data-race diagnostics rather than silencing them.
- Never resume a continuation more than once or forget to resume it.

## async / await

```swift
func fetchUser(id: String) async throws -> User {
  let url = URL(string: "https://api.example.com/users/\(id)")!
  let (data, _) = try await URLSession.shared.data(from: url)
  return try JSONDecoder().decode(User.self, from: data)
}

// Independent work runs in parallel with async let.
func loadDashboard() async throws -> Dashboard {
  async let user = fetchUser(id: "me")
  async let posts = fetchPosts()
  async let stats = fetchStats()
  return try await Dashboard(user: user, posts: posts, stats: stats)
}
```

`await` marks a suspension point: state may change across it, so do not assume invariants hold afterwards.

## Actors and @MainActor

```swift
actor ImageCache {
  private var images: [String: Data] = [:]

  func image(for key: String) async -> Data? {
    if let cached = images[key] { return cached }
    let data = await download(key)
    images[key] = data          // re-check: another call may have filled it while suspended
    return data
  }

  private func download(_ key: String) async -> Data? { nil }
}

@MainActor
final class UserViewModel: ObservableObject {
  @Published private(set) var user: User?

  func load(id: String) async {
    user = try? await fetchUser(id: id)   // already on the main actor
  }
}
```

- Access an actor's state with `await` from outside; keep actor methods short so the actor is not a bottleneck.
- Actor reentrancy: any `await` inside an actor method lets other calls run. Re-check state after it.
- Use `nonisolated` for members that touch only immutable `let` state.
- Swift 6.2 adds opt-in default main-actor isolation and `@concurrent` for work that must run off the actor; adopt them for app targets that are mostly UI code.

## Tasks and cancellation

```swift
let task = Task { try await longRunningOperation() }   // inherits actor and priority
task.cancel()                                           // cooperative; the work must check

func processItems(_ items: [Item]) async throws {
  for item in items {
    try Task.checkCancellation()
    await process(item)
  }
}
```

Store a `Task` you may need to cancel (for example a search-as-you-type task) and cancel the previous one before starting the next. SwiftUI's `.task { }` cancels automatically when the view disappears.

## TaskGroup

```swift
func fetchAllUsers(ids: [String]) async throws -> [User] {
  try await withThrowingTaskGroup(of: User.self) { group in
    for id in ids {
      group.addTask { try await fetchUser(id: id) }
    }
    var users: [User] = []
    for try await user in group { users.append(user) }   // completion order, not input order
    return users
  }
}

// Cap concurrency: seed `limit` tasks, add one as each finishes.
func process(_ items: [Item], limit: Int) async {
  await withTaskGroup(of: Void.self) { group in
    var iterator = items.makeIterator()
    for _ in 0..<limit {
      guard let item = iterator.next() else { break }
      group.addTask { await process(item) }
    }
    for await _ in group {
      if let item = iterator.next() {
        group.addTask { await process(item) }
      }
    }
  }
}
```

A thrown error cancels the remaining children. Return `Result` from each child if you need every outcome. Do not mutate captured variables from child tasks; collect results in the group's loop.

## AsyncSequence and streams

```swift
for await notification in NotificationCenter.default.notifications(named: .didUpdate) {
  handle(notification)
}

// Bridge a callback or delegate producer into an AsyncStream.
let updates = AsyncStream<Location> { continuation in
  manager.onUpdate = { continuation.yield($0) }
  continuation.onTermination = { _ in manager.stop() }
}
```

Prefer `AsyncStream` over hand-written `AsyncIteratorProtocol` types. Always set `onTermination` so the producer stops when the consumer is cancelled.

## Continuations (bridging callbacks)

```swift
func fetchData() async throws -> Data {
  try await withCheckedThrowingContinuation { continuation in
    legacyFetch { result in
      continuation.resume(with: result)   // exactly once, on every path
    }
  }
}
```

Use `withChecked...` continuations (they diagnose misuse); switch to `withUnsafe...` only after profiling. Prefer the async overload if the API already offers one.

## Sendable

```swift
struct Config: Sendable {            // value type with Sendable members
  let baseURL: URL
}

final class Settings: Sendable {     // immutable final class
  let apiKey: String
  init(apiKey: String) { self.apiKey = apiKey }
}

actor Counter {                      // actors are implicitly Sendable
  private(set) var value = 0
  func increment() { value += 1 }
}
```

Closures passed to `Task`, `addTask` and `@Sendable` parameters may only capture Sendable values.

## Patterns

```swift
// Timeout: race the work against a sleeping task.
func withTimeout<T: Sendable>(_ timeout: Duration, _ operation: @escaping @Sendable () async throws -> T) async throws -> T {
  try await withThrowingTaskGroup(of: T.self) { group in
    group.addTask { try await operation() }
    group.addTask {
      try await Task.sleep(for: timeout)
      throw TimeoutError()
    }
    defer { group.cancelAll() }
    return try await group.next()!
  }
}
```

For retry with backoff, see the Swift error-handling guideline. For debouncing, cancel the previous `Task` and sleep before running the new one.

Sources: The Swift Programming Language (Concurrency), SE-0296, SE-0306, SE-0302, WWDC21 sessions 10132 and 10133.
