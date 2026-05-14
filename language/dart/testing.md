# Testing in Dart

## Unit Testing

```dart
import 'package:test/test.dart';

// ✅ Arrange-Act-Assert pattern
void main() {
  group('UserService', () {
    test('should create user with hashed password', () async {
      // Arrange
      final repository = MockUserRepository();
      final service = UserService(repository);
      final userData = UserData(email: 'test@example.com', password: 'secret');

      // Act
      final user = await service.createUser(userData);

      // Assert
      expect(user.email, equals('test@example.com'));
      expect(user.passwordHash, isNot(equals('secret')));
      verify(repository.save(any)).called(1);
    });

    test('should throw ValidationException for invalid email', () {
      final service = UserService(MockUserRepository());

      expect(
        () => service.createUser(UserData(email: 'invalid', password: 'pass')),
        throwsA(isA<ValidationException>()),
      );
    });
  });
}
```

## Test Organization

```dart
void main() {
  // ✅ Use group for related tests
  group('Calculator', () {
    late Calculator calculator;

    // setUp runs before each test
    setUp(() {
      calculator = Calculator();
    });

    // tearDown runs after each test
    tearDown(() {
      calculator.dispose();
    });

    test('adds two numbers', () {
      expect(calculator.add(2, 3), equals(5));
    });

    test('subtracts two numbers', () {
      expect(calculator.subtract(5, 3), equals(2));
    });

    group('division', () {
      test('divides two numbers', () {
        expect(calculator.divide(10, 2), equals(5));
      });

      test('throws on division by zero', () {
        expect(
          () => calculator.divide(10, 0),
          throwsA(isA<DivisionByZeroException>()),
        );
      });
    });
  });
}
```

## Matchers

```dart
// Equality
expect(value, equals(expected));
expect(value, isNot(equals(unexpected)));

// Types
expect(value, isA<String>());
expect(value, isNotNull);
expect(value, isNull);

// Numbers
expect(value, greaterThan(5));
expect(value, lessThan(10));
expect(value, closeTo(3.14, 0.01));

// Strings
expect(text, contains('hello'));
expect(text, startsWith('Hello'));
expect(text, endsWith('world'));
expect(text, matches(r'^\d+$'));

// Collections
expect(list, isEmpty);
expect(list, isNotEmpty);
expect(list, hasLength(3));
expect(list, contains(item));
expect(map, containsValue(value));

// Futures
expect(future, completes);
expect(future, throwsException);
expect(future, completion(equals(value)));

// Custom matchers
expect(user, isA<User>().having((u) => u.email, 'email', 'test@example.com'));
```

## Mocking with Mockito

```dart
import 'package:mockito/mockito.dart';
import 'package:mockito/annotations.dart';

// Generate mocks
@GenerateMocks([UserRepository, ApiClient])
void main() {
  group('UserService', () {
    late MockUserRepository repository;
    late UserService service;

    setUp(() {
      repository = MockUserRepository();
      service = UserService(repository);
    });

    test('should fetch user by id', () async {
      // Arrange - stub method
      final expectedUser = User(id: '1', name: 'Alice');
      when(repository.findById('1')).thenAnswer((_) async => expectedUser);

      // Act
      final user = await service.getUser('1');

      // Assert
      expect(user, equals(expectedUser));
      verify(repository.findById('1')).called(1);
    });

    test('should handle not found', () async {
      // Stub to return null
      when(repository.findById(any)).thenAnswer((_) async => null);

      expect(
        () => service.getUser('999'),
        throwsA(isA<NotFoundException>()),
      );
    });
  });
}
```

## Testing Async Code

```dart
test('should fetch data asynchronously', () async {
  final data = await fetchData();
  expect(data, isNotNull);
});

test('should complete within timeout', () async {
  await expectLater(
    slowOperation().timeout(const Duration(seconds: 2)),
    completes,
  );
});

test('should handle errors', () {
  expect(
    failingOperation(),
    throwsA(isA<NetworkException>()),
  );
});
```

## Testing Streams

```dart
test('should emit values from stream', () async {
  final stream = Stream.fromIterable([1, 2, 3]);

  expect(stream, emitsInOrder([1, 2, 3]));
});

test('should emit and complete', () {
  expect(
    countStream(3),
    emitsInOrder([
      1,
      2,
      3,
      emitsDone,
    ]),
  );
});

test('should handle stream errors', () {
  expect(
    errorStream(),
    emitsInOrder([
      1,
      emitsError(isA<Exception>()),
    ]),
  );
});
```

## Widget Testing (Flutter)

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('should display user name', (WidgetTester tester) async {
    // Arrange
    final user = User(id: '1', name: 'Alice');

    // Act
    await tester.pumpWidget(
      MaterialApp(home: UserProfile(user: user)),
    );

    // Assert
    expect(find.text('Alice'), findsOneWidget);
  });

  testWidgets('should handle button tap', (WidgetTester tester) async {
    bool tapped = false;

    await tester.pumpWidget(
      MaterialApp(
        home: Scaffold(
          body: ElevatedButton(
            onPressed: () => tapped = true,
            child: const Text('Tap me'),
          ),
        ),
      ),
    );

    // Find and tap button
    await tester.tap(find.text('Tap me'));
    await tester.pump();

    expect(tapped, isTrue);
  });
}
```

## Integration Testing

```dart
import 'package:integration_test/integration_test.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Login flow', () {
    testWidgets('should login successfully', (tester) async {
      await tester.pumpWidget(MyApp());

      // Enter credentials
      await tester.enterText(
        find.byKey(const Key('email_field')),
        'test@example.com',
      );
      await tester.enterText(
        find.byKey(const Key('password_field')),
        'password123',
      );

      // Tap login
      await tester.tap(find.byKey(const Key('login_button')));
      await tester.pumpAndSettle();

      // Verify navigation to home
      expect(find.byType(HomePage), findsOneWidget);
    });
  });
}
```

## Test Doubles

```dart
// Fake - working implementation (not for production)
class FakeUserRepository implements UserRepository {
  final _users = <String, User>{};

  @override
  Future<User?> findById(String id) async {
    return _users[id];
  }

  @override
  Future<void> save(User user) async {
    _users[user.id] = user;
  }
}

// Stub - returns canned responses
class StubApiClient implements ApiClient {
  @override
  Future<Response> get(String url) async {
    return Response(statusCode: 200, data: {'id': '1', 'name': 'Alice'});
  }
}

// Mock - pre-programmed with expectations (use Mockito)
final mock = MockUserRepository();
when(mock.findById('1')).thenAnswer((_) async => testUser);
```

## Coverage

```bash
# Run tests with coverage
dart test --coverage=coverage

# Generate HTML report
genhtml coverage/lcov.info -o coverage/html

# View coverage
open coverage/html/index.html
```

## Best Practices

- Use `group()` to organize related tests
- One test per behavior
- Use descriptive test names: "should [expected behavior] when [scenario]"
- Follow Arrange-Act-Assert pattern
- Test observable behavior, not implementation
- Use setUp/tearDown for common initialization
- Mock external dependencies
- Test edge cases and error conditions
- Keep tests fast and independent
- Use `pump()` and `pumpAndSettle()` in widget tests
- Verify interactions with mocks
- Aim for high code coverage (80%+)
- Run tests in CI/CD pipeline
