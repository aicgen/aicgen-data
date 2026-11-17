# SwiftUI and MVVM Architecture

## Rules

- Views render state and forward user events. Business logic, formatting and I/O live in a view model (or model).
- One owner per piece of state; pass it down as a value, a `Binding`, or an observable object. Events flow up through closures or view-model methods.
- Mark view models `@MainActor`. Inject dependencies through `init` (protocols), never create services inside views.
- Use `NavigationStack` with value-based navigation. `NavigationView` is deprecated.
- Keep `body` cheap and free of side effects: no sorting, filtering, networking or object creation. Start async work with `.task`.
- Extract subviews once a `body` exceeds roughly 40 lines or repeats a block.

## Choose the state tool

| Need | iOS 17+ (Observation) | iOS 16 and earlier |
|------|----------------------|--------------------|
| View-local value | `@State` | `@State` |
| View owns a view model | `@State private var vm = VM()` | `@StateObject` |
| View receives a view model | plain property (`@Bindable` for `$` bindings) | `@ObservedObject` |
| App-wide dependency | `@Environment` | `@EnvironmentObject` |
| Child edits parent's value | `@Binding` | `@Binding` |

Prefer `@Observable` (Observation framework) for new code on iOS 17+; use `ObservableObject` only when supporting older systems.

## MVVM structure

```swift
struct User: Identifiable, Codable {
  let id: String
  var name: String
}

@MainActor
final class UserListViewModel: ObservableObject {
  @Published private(set) var users: [User] = []
  @Published private(set) var isLoading = false
  @Published private(set) var errorMessage: String?

  private let repository: UserRepositoryProtocol

  init(repository: UserRepositoryProtocol) {
    self.repository = repository
  }

  func load() async {
    isLoading = true
    errorMessage = nil
    defer { isLoading = false }
    do {
      users = try await repository.fetchUsers()
    } catch {
      errorMessage = "Failed to load users: \(error.localizedDescription)"
    }
  }
}

struct UserListView: View {
  @StateObject private var viewModel: UserListViewModel

  init(repository: UserRepositoryProtocol) {
    _viewModel = StateObject(wrappedValue: UserListViewModel(repository: repository))
  }

  var body: some View {
    NavigationStack {
      Group {
        if viewModel.isLoading {
          ProgressView()
        } else if let message = viewModel.errorMessage {
          ErrorView(message: message) { Task { await viewModel.load() } }
        } else {
          List(viewModel.users) { UserRow(user: $0) }
        }
      }
      .navigationTitle("Users")
      .task { await viewModel.load() }
    }
  }
}
```

- `@StateObject` / `@State` creates and owns the view model once; `@ObservedObject` only observes one owned elsewhere. Using `@ObservedObject` for an object created inside the view recreates it on every update.
- Expose state as `private(set)` so only the view model mutates it.

## Data flow

```swift
// Data flows down as values; events flow up as closures.
struct ChildView: View {
  let title: String
  let onSelect: () -> Void

  var body: some View {
    Button(title, action: onSelect)
  }
}

// A child that edits the parent's value takes a Binding.
struct ToggleRow: View {
  let title: String
  @Binding var isOn: Bool

  var body: some View { Toggle(title, isOn: $isOn) }
}
```

Use `@Environment` / `@EnvironmentObject` only for genuinely app-wide values (current user, theme, services), not to avoid passing a parameter one level down.

## Navigation and sheets

```swift
enum Route: Hashable {
  case userDetail(id: String)
  case settings
}

struct RootView: View {
  @State private var path = NavigationPath()

  var body: some View {
    NavigationStack(path: $path) {
      Button("Open") { path.append(Route.userDetail(id: "123")) }
        .navigationDestination(for: Route.self) { route in
          switch route {
          case .userDetail(let id): UserDetailView(userId: id)
          case .settings: SettingsView()
          }
        }
    }
  }
}

// Item-based sheet: no separate Bool to keep in sync.
.sheet(item: $selectedUser) { user in
  UserDetailView(user: user)
}
```

## Reactive input

```swift
// Debounce search input in the view model with Combine.
$searchText
  .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
  .removeDuplicates()
  .sink { [weak self] query in Task { await self?.search(query) } }
  .store(in: &cancellables)
```

Always capture `self` weakly in stored Combine pipelines.

## Performance

- Keep expensive derivation out of `body`: compute it in the view model when its inputs change.
- Break large views into subviews so SwiftUI re-evaluates only what changed.
- Give list rows stable identity (`Identifiable` with a real `id`); never use `\.self` or array indices for mutable data.
- Conform small row views to `Equatable` and apply `.equatable()` only after profiling shows redundant updates.

## Testing view models

```swift
@MainActor
final class UserListViewModelTests: XCTestCase {
  func testLoadPublishesUsers() async {
    let repo = MockUserRepository(users: [User(id: "1", name: "Alice")])
    let viewModel = UserListViewModel(repository: repo)

    await viewModel.load()

    XCTAssertEqual(viewModel.users.map(\.name), ["Alice"])
    XCTAssertNil(viewModel.errorMessage)
  }
}
```

Test view models directly, with fake repositories injected through `init`. Add `#Preview` blocks for the loading, error and empty states.

## Reusable styling

```swift
struct CardStyle: ViewModifier {
  func body(content: Content) -> some View {
    content.padding().background(.background).clipShape(RoundedRectangle(cornerRadius: 12)).shadow(radius: 4)
  }
}

extension View {
  func cardStyle() -> some View { modifier(CardStyle()) }
}
```

Sources: Apple's SwiftUI, "Managing model data in your app" and Observation documentation.
