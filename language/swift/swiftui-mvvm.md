# SwiftUI and MVVM Architecture

## MVVM Pattern in SwiftUI

SwiftUI works naturally with the MVVM (Model-View-ViewModel) pattern:

```swift
// Model - Data and business logic
struct User: Identifiable, Codable {
  let id: String
  var name: String
  var email: String
  var isActive: Bool
}

// ViewModel - Presentation logic
@MainActor
class UserViewModel: ObservableObject {
  @Published var users: [User] = []
  @Published var isLoading = false
  @Published var errorMessage: String?

  private let repository: UserRepository

  init(repository: UserRepository = UserRepository()) {
    self.repository = repository
  }

  func loadUsers() async {
    isLoading = true
    errorMessage = nil

    do {
      users = try await repository.fetchUsers()
    } catch {
      errorMessage = "Failed to load users: \(error.localizedDescription)"
    }

    isLoading = false
  }

  func deleteUser(_ user: User) async {
    do {
      try await repository.delete(user)
      users.removeAll { $0.id == user.id }
    } catch {
      errorMessage = "Failed to delete user: \(error.localizedDescription)"
    }
  }
}

// View - UI and user interaction
struct UserListView: View {
  @StateObject private var viewModel = UserViewModel()

  var body: some View {
    NavigationView {
      Group {
        if viewModel.isLoading {
          ProgressView("Loading users...")
        } else if let error = viewModel.errorMessage {
          ErrorView(message: error) {
            Task { await viewModel.loadUsers() }
          }
        } else {
          userList
        }
      }
      .navigationTitle("Users")
      .task {
        await viewModel.loadUsers()
      }
    }
  }

  private var userList: some View {
    List {
      ForEach(viewModel.users) { user in
        UserRow(user: user)
      }
      .onDelete { indexSet in
        Task {
          for index in indexSet {
            await viewModel.deleteUser(viewModel.users[index])
          }
        }
      }
    }
  }
}
```

## State Management

### @State for View-Local State

```swift
// ✅ Use @State for simple view-specific state
struct CounterView: View {
  @State private var count = 0

  var body: some View {
    VStack {
      Text("Count: \(count)")
      Button("Increment") {
        count += 1
      }
    }
  }
}

// ❌ Don't use @State for complex or shared state
struct BadView: View {
  @State private var users: [User] = [] // Should be in ViewModel
  @State private var repository = UserRepository() // Should be injected
}
```

### @StateObject for ViewModel Ownership

```swift
// ✅ Use @StateObject when view creates and owns the ViewModel
struct UserListView: View {
  @StateObject private var viewModel = UserViewModel()

  var body: some View {
    List(viewModel.users) { user in
      Text(user.name)
    }
    .task {
      await viewModel.loadUsers()
    }
  }
}

// ✅ StateObject with dependency injection
struct UserListView: View {
  @StateObject private var viewModel: UserViewModel

  init(repository: UserRepository) {
    _viewModel = StateObject(wrappedValue: UserViewModel(repository: repository))
  }
}
```

### @ObservedObject for Passed ViewModels

```swift
// ✅ Use @ObservedObject when ViewModel is passed from parent
struct UserDetailView: View {
  @ObservedObject var viewModel: UserDetailViewModel

  var body: some View {
    VStack {
      Text(viewModel.user.name)
      TextField("Name", text: $viewModel.editableName)
    }
  }
}

// Parent view creates and passes ViewModel
struct ParentView: View {
  @StateObject private var detailViewModel = UserDetailViewModel()

  var body: some View {
    UserDetailView(viewModel: detailViewModel)
  }
}
```

### @EnvironmentObject for App-Wide State

```swift
// ✅ Use @EnvironmentObject for app-wide dependencies
class AppState: ObservableObject {
  @Published var currentUser: User?
  @Published var isAuthenticated = false

  func login(user: User) {
    currentUser = user
    isAuthenticated = true
  }

  func logout() {
    currentUser = nil
    isAuthenticated = false
  }
}

// Inject at app level
@main
struct MyApp: App {
  @StateObject private var appState = AppState()

  var body: some Scene {
    WindowGroup {
      ContentView()
        .environmentObject(appState)
    }
  }
}

// Access in any view
struct ProfileView: View {
  @EnvironmentObject var appState: AppState

  var body: some View {
    if let user = appState.currentUser {
      Text("Welcome, \(user.name)")
      Button("Logout") {
        appState.logout()
      }
    }
  }
}
```

### @Binding for Two-Way Data Flow

```swift
// ✅ Use @Binding for child views that need to modify parent state
struct ToggleRow: View {
  let title: String
  @Binding var isOn: Bool

  var body: some View {
    Toggle(title, isOn: $isOn)
  }
}

// Parent provides the binding
struct SettingsView: View {
  @State private var notificationsEnabled = false

  var body: some View {
    ToggleRow(title: "Notifications", isOn: $notificationsEnabled)
  }
}
```

## View Composition

### Extract Subviews

```swift
// ❌ Monolithic view
struct ProductView: View {
  let product: Product

  var body: some View {
    VStack {
      HStack {
        AsyncImage(url: product.imageURL) { image in
          image.resizable().aspectRatio(contentMode: .fit)
        } placeholder: {
          ProgressView()
        }
        .frame(width: 60, height: 60)

        VStack(alignment: .leading) {
          Text(product.name).font(.headline)
          Text(product.description).font(.subheadline).foregroundColor(.gray)
        }
      }

      HStack {
        Text("$\(product.price, specifier: "%.2f")")
        Spacer()
        if product.inStock {
          Text("In Stock").foregroundColor(.green)
        } else {
          Text("Out of Stock").foregroundColor(.red)
        }
      }
    }
  }
}

// ✅ Composed from smaller views
struct ProductView: View {
  let product: Product

  var body: some View {
    VStack(alignment: .leading, spacing: 12) {
      ProductHeader(product: product)
      ProductFooter(product: product)
    }
  }
}

struct ProductHeader: View {
  let product: Product

  var body: some View {
    HStack {
      ProductImage(url: product.imageURL)
      ProductInfo(name: product.name, description: product.description)
    }
  }
}

struct ProductFooter: View {
  let product: Product

  var body: some View {
    HStack {
      PriceLabel(price: product.price)
      Spacer()
      StockStatus(inStock: product.inStock)
    }
  }
}
```

### Use ViewBuilder

```swift
// ✅ ViewBuilder for conditional content
struct ContentView: View {
  let isLoggedIn: Bool

  var body: some View {
    VStack {
      headerContent
      mainContent
    }
  }

  @ViewBuilder
  private var headerContent: some View {
    if isLoggedIn {
      ProfileHeader()
    } else {
      LoginPrompt()
    }
  }

  @ViewBuilder
  private var mainContent: some View {
    Text("Main Content")
  }
}
```

## Navigation Patterns

### NavigationStack (iOS 16+)

```swift
// ✅ Type-safe navigation with NavigationStack
struct NavigationExample: View {
  @State private var path = NavigationPath()

  var body: some View {
    NavigationStack(path: $path) {
      List(users) { user in
        NavigationLink(value: user) {
          UserRow(user: user)
        }
      }
      .navigationDestination(for: User.self) { user in
        UserDetailView(user: user)
      }
      .navigationTitle("Users")
    }
  }
}

// ✅ Programmatic navigation
struct ProgrammaticNav: View {
  @State private var path = NavigationPath()

  var body: some View {
    NavigationStack(path: $path) {
      Button("Go to Detail") {
        path.append(DetailRoute.userDetail(id: "123"))
      }
      .navigationDestination(for: DetailRoute.self) { route in
        destinationView(for: route)
      }
    }
  }

  @ViewBuilder
  func destinationView(for route: DetailRoute) -> some View {
    switch route {
    case .userDetail(let id):
      UserDetailView(userId: id)
    case .settings:
      SettingsView()
    }
  }
}

enum DetailRoute: Hashable {
  case userDetail(id: String)
  case settings
}
```

### Sheet Presentation

```swift
// ✅ Present sheet with binding
struct SheetExample: View {
  @State private var showingSheet = false
  @State private var selectedUser: User?

  var body: some View {
    List(users) { user in
      Button(user.name) {
        selectedUser = user
        showingSheet = true
      }
    }
    .sheet(isPresented: $showingSheet) {
      if let user = selectedUser {
        UserDetailView(user: user)
      }
    }
  }
}

// ✅ Use item-based sheet (cleaner)
struct ItemSheetExample: View {
  @State private var selectedUser: User?

  var body: some View {
    List(users) { user in
      Button(user.name) {
        selectedUser = user
      }
    }
    .sheet(item: $selectedUser) { user in
      UserDetailView(user: user)
    }
  }
}
```

## Data Flow Patterns

### Unidirectional Data Flow

```swift
// ✅ Data flows down, events flow up
struct ParentView: View {
  @StateObject private var viewModel = ParentViewModel()

  var body: some View {
    VStack {
      // Data flows down
      ChildView(
        data: viewModel.data,
        // Events flow up
        onAction: { action in
          viewModel.handle(action)
        }
      )
    }
  }
}

struct ChildView: View {
  let data: String
  let onAction: (Action) -> Void

  var body: some View {
    Button(data) {
      onAction(.buttonTapped) // Event flows up
    }
  }
}

enum Action {
  case buttonTapped
  case itemSelected(id: String)
}
```

### Combine Integration

```swift
// ✅ Use Combine for reactive updates
@MainActor
class SearchViewModel: ObservableObject {
  @Published var searchText = ""
  @Published var results: [SearchResult] = []
  @Published var isSearching = false

  private var cancellables = Set<AnyCancellable>()
  private let searchService: SearchService

  init(searchService: SearchService = SearchService()) {
    self.searchService = searchService

    $searchText
      .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
      .removeDuplicates()
      .sink { [weak self] query in
        Task {
          await self?.search(query: query)
        }
      }
      .store(in: &cancellables)
  }

  private func search(query: String) async {
    guard !query.isEmpty else {
      results = []
      return
    }

    isSearching = true
    defer { isSearching = false }

    do {
      results = try await searchService.search(query: query)
    } catch {
      results = []
    }
  }
}
```

## Custom View Modifiers

```swift
// ✅ Reusable styling with ViewModifier
struct CardStyle: ViewModifier {
  func body(content: Content) -> some View {
    content
      .padding()
      .background(Color.white)
      .cornerRadius(12)
      .shadow(radius: 4)
  }
}

extension View {
  func cardStyle() -> some View {
    modifier(CardStyle())
  }
}

// Usage
Text("Hello")
  .cardStyle()

// ✅ Parameterized modifiers
struct PrimaryButtonStyle: ViewModifier {
  let isEnabled: Bool

  func body(content: Content) -> some View {
    content
      .font(.headline)
      .foregroundColor(.white)
      .padding()
      .background(isEnabled ? Color.blue : Color.gray)
      .cornerRadius(8)
  }
}

extension View {
  func primaryButton(enabled: Bool = true) -> some View {
    modifier(PrimaryButtonStyle(isEnabled: enabled))
  }
}
```

## Performance Optimization

### Avoid Expensive Computations in body

```swift
// ❌ Expensive work in body (called on every update)
struct BadView: View {
  let items: [Item]

  var body: some View {
    let processed = processExpensiveData(items) // Recomputed every render!
    List(processed) { item in
      Text(item.name)
    }
  }
}

// ✅ Compute in ViewModel or use computed property
@MainActor
class GoodViewModel: ObservableObject {
  @Published var items: [Item] = []

  var processedItems: [ProcessedItem] {
    processExpensiveData(items)
  }
}

// ✅ Or memoize the result
struct GoodView: View {
  let items: [Item]

  private var processedItems: [ProcessedItem] {
    // Only recomputed when items change
    items.map { ProcessedItem($0) }
  }

  var body: some View {
    List(processedItems) { item in
      Text(item.name)
    }
  }
}
```

### Use Equatable for Performance

```swift
// ✅ Conform to Equatable to avoid unnecessary updates
struct UserRow: View, Equatable {
  let user: User

  var body: some View {
    HStack {
      AsyncImage(url: user.avatarURL)
      Text(user.name)
    }
  }

  static func == (lhs: UserRow, rhs: UserRow) -> Bool {
    lhs.user.id == rhs.user.id
  }
}

// Use .equatable() to enable optimization
List(users) { user in
  UserRow(user: user)
    .equatable()
}
```

## Testing ViewModels

```swift
// ✅ Testable ViewModel with dependency injection
@MainActor
class UserViewModel: ObservableObject {
  @Published var users: [User] = []
  private let repository: UserRepositoryProtocol

  init(repository: UserRepositoryProtocol) {
    self.repository = repository
  }

  func loadUsers() async {
    users = try? await repository.fetchUsers()
  }
}

// Test with mock repository
@MainActor
class UserViewModelTests: XCTestCase {
  func testLoadUsers() async {
    let mockRepo = MockUserRepository()
    mockRepo.usersToReturn = [
      User(id: "1", name: "Alice"),
      User(id: "2", name: "Bob")
    ]

    let viewModel = UserViewModel(repository: mockRepo)
    await viewModel.loadUsers()

    XCTAssertEqual(viewModel.users.count, 2)
    XCTAssertEqual(viewModel.users.first?.name, "Alice")
  }
}
```

## Best Practices

### Keep Views Simple

```swift
// ✅ View only contains UI logic
struct ProductView: View {
  @ObservedObject var viewModel: ProductViewModel

  var body: some View {
    VStack {
      Text(viewModel.productName)
      Text(viewModel.formattedPrice)
      Button("Add to Cart") {
        Task { await viewModel.addToCart() }
      }
    }
  }
}

// ViewModel contains all business logic
@MainActor
class ProductViewModel: ObservableObject {
  @Published var product: Product

  var productName: String { product.name }
  var formattedPrice: String { "$\(product.price, specifier: "%.2f")" }

  func addToCart() async {
    // Business logic here
  }
}
```

### Use Preview Provider

```swift
// ✅ Provide previews for development
struct UserListView_Previews: PreviewProvider {
  static var previews: some View {
    Group {
      // Default state
      UserListView()
        .previewDisplayName("Default")

      // Loading state
      UserListView(viewModel: .loading)
        .previewDisplayName("Loading")

      // Error state
      UserListView(viewModel: .error)
        .previewDisplayName("Error")
    }
  }
}

// Helper to create preview ViewModels
extension UserViewModel {
  static var loading: UserViewModel {
    let vm = UserViewModel()
    vm.isLoading = true
    return vm
  }

  static var error: UserViewModel {
    let vm = UserViewModel()
    vm.errorMessage = "Failed to load users"
    return vm
  }
}
```

### Environment for Dependencies

```swift
// ✅ Use Environment for injecting dependencies
private struct RepositoryKey: EnvironmentKey {
  static let defaultValue: UserRepository = UserRepository()
}

extension EnvironmentValues {
  var userRepository: UserRepository {
    get { self[RepositoryKey.self] }
    set { self[RepositoryKey.self] = newValue }
  }
}

// Provide at app level
@main
struct MyApp: App {
  let repository = UserRepository()

  var body: some Scene {
    WindowGroup {
      ContentView()
        .environment(\.userRepository, repository)
    }
  }
}

// Access in views
struct UserListView: View {
  @Environment(\.userRepository) var repository
  @StateObject private var viewModel: UserViewModel

  init() {
    _viewModel = StateObject(wrappedValue: UserViewModel(repository: repository))
  }
}
```

---

**Sources:**
- [Apple SwiftUI Documentation](https://developer.apple.com/documentation/swiftui/)
- [SwiftUI Data Flow](https://developer.apple.com/documentation/swiftui/managing-model-data-in-your-app)
- [WWDC22: The SwiftUI cookbook for navigation](https://developer.apple.com/videos/play/wwdc2022/10054/)
- [Swift by Sundell: SwiftUI Architecture](https://www.swiftbysundell.com/basics/swiftui/)
