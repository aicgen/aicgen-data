# Dart Fundamentals

## Null Safety

Dart has sound null safety - non-nullable by default:

```dart
// Non-nullable types
String name = 'Alice';           // Cannot be null
int age = 30;                    // Cannot be null

// Nullable types - add ?
String? middleName;              // Can be null
int? optionalAge;                // Can be null

// ❌ Compile error
String lastName = null;          // Error: Can't assign null to non-nullable

// ✅ Correct
String? lastName = null;         // OK: Explicitly nullable
```

## Type Annotations

Always use explicit types for clarity:

```dart
// ✅ Explicit types
String getUserName(int userId) {
  final User user = fetchUser(userId);
  return user.name;
}

// Variables
final String message = 'Hello';
const int maxRetries = 3;
List<String> names = ['Alice', 'Bob'];

// ❌ Avoid dynamic
dynamic data = fetchData();      // No type safety

// ✅ Use specific types
Map<String, dynamic> data = fetchData();
```

## Null-Aware Operators

```dart
// ?. - Null-aware access
String? name = user?.name;       // null if user is null

// ?? - Null coalescing
String displayName = user?.name ?? 'Guest';

// ??= - Null-aware assignment
name ??= 'Unknown';              // Assign only if null

// ! - Null assertion (use sparingly)
String name = user!.name;        // Assert user is not null
```

## Collections

```dart
// Lists
final List<String> fruits = ['apple', 'banana'];
fruits.add('orange');

// Sets (unique values)
final Set<int> uniqueIds = {1, 2, 3};

// Maps
final Map<String, int> scores = {
  'Alice': 100,
  'Bob': 95,
};

// ✅ Use collection if for conditional elements
final List<String> items = [
  'required',
  if (showOptional) 'optional',
  if (user != null) user.name,
];

// ✅ Use spread operator
final List<int> combined = [...list1, ...list2];
```

## Functions

```dart
// Named parameters (recommended for multiple params)
void createUser({
  required String email,
  required String name,
  int? age,
}) {
  // email and name are required
  // age is optional
}

createUser(email: 'test@example.com', name: 'Alice');

// Positional parameters
int add(int a, int b) => a + b;

// Optional positional
String greet(String name, [String? title]) {
  return title != null ? '$title $name' : name;
}

// Arrow syntax for single expressions
bool isAdult(int age) => age >= 18;
```

## Classes

```dart
// ✅ Use final for immutable fields
class User {
  final String id;
  final String email;
  String name;                   // Mutable

  User({
    required this.id,
    required this.email,
    required this.name,
  });

  // Named constructors
  User.guest() : id = '', email = '', name = 'Guest';

  // Factory constructors
  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'] as String,
      email: json['email'] as String,
      name: json['name'] as String,
    );
  }
}

// ✅ Immutable classes with const
class Point {
  final double x;
  final double y;

  const Point(this.x, this.y);
}

const origin = Point(0, 0);      // Compile-time constant
```

## Enums

```dart
// Simple enum
enum Status {
  pending,
  processing,
  completed,
  failed,
}

// Enhanced enums (Dart 2.17+)
enum UserRole {
  admin('Administrator', level: 3),
  editor('Editor', level: 2),
  viewer('Viewer', level: 1);

  const UserRole(this.displayName, {required this.level});

  final String displayName;
  final int level;

  bool canEdit() => level >= 2;
}

// Usage
final role = UserRole.admin;
print(role.displayName);         // 'Administrator'
print(role.canEdit());           // true
```

## Extension Methods

```dart
// Add methods to existing types
extension StringExtensions on String {
  bool get isEmail => contains('@');

  String capitalize() {
    if (isEmpty) return this;
    return '${this[0].toUpperCase()}${substring(1)}';
  }
}

// Usage
print('test@example.com'.isEmail);    // true
print('hello'.capitalize());          // 'Hello'
```

## Cascade Notation

```dart
// ✅ Use cascades for fluent method chaining
final user = User(id: '1', email: 'test@example.com', name: 'Alice')
  ..setRole(UserRole.admin)
  ..setActive(true)
  ..save();

// ✅ Building objects
final button = Button()
  ..text = 'Submit'
  ..onPressed = handleSubmit
  ..enabled = true;
```

## Late Variables

```dart
// late - initialize later, but before use
class UserService {
  late final Database db;

  Future<void> init() async {
    db = await Database.connect();
  }

  Future<User> getUser(String id) async {
    return db.query('SELECT * FROM users WHERE id = ?', [id]);
  }
}

// ⚠️ Runtime error if accessed before initialization
late String config;
print(config);                   // Error!

// ✅ Initialize before use
config = loadConfig();
print(config);                   // OK
```

## Naming Conventions

```dart
// ✓ Classes/Enums/Typedefs: PascalCase
class UserAccount { }
enum OrderStatus { }
typedef IntCallback = void Function(int);

// ✓ Variables/Functions/Parameters: camelCase
String userName = 'Alice';
void processOrder() { }

// ✓ Constants: lowerCamelCase
const maxRetries = 3;
const apiBaseUrl = 'https://api.example.com';

// ✓ Private members: prefix with _
class User {
  String _password;              // Private field
  void _hashPassword() { }       // Private method
}

// ✓ Files: snake_case
// user_service.dart
// order_repository.dart
```

## Best Practices

- Prefer `final` over `var` for immutability
- Use `const` for compile-time constants
- Leverage null safety - avoid `!` assertion
- Use named parameters for functions with multiple parameters
- Make classes immutable when possible
- Use extension methods to add functionality to existing types
- Follow the official Dart style guide
