# Error Handling in Dart

## Custom Exceptions

```dart
// ✅ Create structured exception hierarchy
class AppException implements Exception {
  final String message;
  final int? code;
  final dynamic details;

  AppException(this.message, {this.code, this.details});

  @override
  String toString() => 'AppException: $message (code: $code)';
}

class NotFoundException extends AppException {
  NotFoundException(String resource, String id)
      : super('$resource with id $id not found', code: 404);
}

class ValidationException extends AppException {
  ValidationException(String message, {Map<String, String>? errors})
      : super(message, code: 400, details: errors);
}

// Usage
throw NotFoundException('User', userId);
throw ValidationException('Invalid input', errors: {'email': 'Invalid format'});
```

## Try-Catch-Finally

```dart
// ✅ Catch specific exceptions first
Future<User> fetchUser(String id) async {
  try {
    final response = await http.get('/api/users/$id');
    return User.fromJson(response.data);
  } on NetworkException catch (e) {
    print('Network error: ${e.message}');
    throw AppException('Network request failed', details: e);
  } on FormatException catch (e) {
    print('Invalid JSON: $e');
    throw AppException('Invalid response format');
  } catch (e, stackTrace) {
    print('Unexpected error: $e');
    print('Stack trace: $stackTrace');
    rethrow;
  } finally {
    print('Request completed');
  }
}

// ✅ Use finally for cleanup
Future<void> processFile(String path) async {
  File? file;
  try {
    file = File(path);
    final content = await file.readAsString();
    await processContent(content);
  } catch (e) {
    print('Error processing file: $e');
    rethrow;
  } finally {
    // Always runs, even if exception thrown
    await file?.close();
  }
}
```

## Result Type Pattern

```dart
// ✅ Explicit success/failure without exceptions
class Result<T> {
  final T? data;
  final AppException? error;

  const Result.success(T data) : data = data, error = null;
  const Result.failure(AppException error) : data = null, error = error;

  bool get isSuccess => error == null;
  bool get isFailure => error != null;
}

// Usage
Future<Result<User>> fetchUserSafe(String id) async {
  try {
    final user = await fetchUser(id);
    return Result.success(user);
  } on AppException catch (e) {
    return Result.failure(e);
  } catch (e) {
    return Result.failure(AppException('Unknown error: $e'));
  }
}

// Consumer handles explicitly
final result = await fetchUserSafe('123');
if (result.isSuccess) {
  print('User: ${result.data!.name}');
} else {
  print('Error: ${result.error!.message}');
}
```

## Either Type Pattern

```dart
// ✅ Functional error handling
sealed class Either<L, R> {
  const Either();
}

class Left<L, R> extends Either<L, R> {
  final L value;
  const Left(this.value);
}

class Right<L, R> extends Either<L, R> {
  final R value;
  const Right(this.value);
}

// Usage
Future<Either<AppException, User>> fetchUser(String id) async {
  try {
    final user = await _fetchUser(id);
    return Right(user);
  } on AppException catch (e) {
    return Left(e);
  }
}

// Pattern matching (Dart 3.0+)
final result = await fetchUser('123');
switch (result) {
  case Left(value: final error):
    print('Error: ${error.message}');
  case Right(value: final user):
    print('User: ${user.name}');
}
```

## Validation

```dart
// ✅ Validate early, throw specific errors
class UserValidator {
  static void validate(String email, String password) {
    if (email.isEmpty) {
      throw ValidationException('Email is required');
    }

    if (!email.contains('@')) {
      throw ValidationException('Invalid email format');
    }

    if (password.length < 8) {
      throw ValidationException('Password must be at least 8 characters');
    }
  }
}

// Usage
try {
  UserValidator.validate(email, password);
  await createUser(email, password);
} on ValidationException catch (e) {
  showError(e.message);
}
```

## Assert for Development

```dart
// ✅ Use assert for development-time checks
void transfer(Account from, Account to, double amount) {
  assert(amount > 0, 'Amount must be positive');
  assert(from.balance >= amount, 'Insufficient funds');

  from.withdraw(amount);
  to.deposit(amount);
}

// Assertions only run in debug mode
// In production, they're removed
```

## Error Boundaries (Flutter)

```dart
// ✅ Catch errors at widget level
class ErrorBoundary extends StatefulWidget {
  final Widget child;

  const ErrorBoundary({required this.child});

  @override
  State<ErrorBoundary> createState() => _ErrorBoundaryState();
}

class _ErrorBoundaryState extends State<ErrorBoundary> {
  Object? error;

  @override
  Widget build(BuildContext context) {
    if (error != null) {
      return ErrorWidget(error: error!);
    }

    return ErrorWrapper(
      onError: (error, stackTrace) {
        setState(() => this.error = error);
      },
      child: widget.child,
    );
  }
}
```

## Global Error Handling

```dart
// ✅ Catch uncaught errors
void main() {
  // Synchronous errors
  FlutterError.onError = (details) {
    print('Flutter error: ${details.exception}');
    print('Stack trace: ${details.stack}');
    reportError(details.exception, details.stack);
  };

  // Asynchronous errors
  PlatformDispatcher.instance.onError = (error, stack) {
    print('Async error: $error');
    reportError(error, stack);
    return true;
  };

  runApp(MyApp());
}

void reportError(Object error, StackTrace? stackTrace) {
  // Send to error tracking service
}
```

## Retry Pattern

```dart
// ✅ Retry with exponential backoff
Future<T> retryWithBackoff<T>(
  Future<T> Function() operation, {
  int maxAttempts = 3,
  Duration initialDelay = const Duration(seconds: 1),
}) async {
  int attempt = 0;
  while (true) {
    try {
      return await operation();
    } catch (e) {
      attempt++;
      if (attempt >= maxAttempts) rethrow;

      final delay = initialDelay * (1 << attempt); // Exponential backoff
      print('Retry attempt $attempt after $delay');
      await Future.delayed(delay);
    }
  }
}

// Usage
final user = await retryWithBackoff(
  () => fetchUser('123'),
  maxAttempts: 5,
);
```

## Circuit Breaker Pattern

```dart
// ✅ Prevent cascading failures
class CircuitBreaker {
  final int failureThreshold;
  final Duration timeout;

  int _failureCount = 0;
  DateTime? _openedAt;

  CircuitBreaker({
    this.failureThreshold = 5,
    this.timeout = const Duration(minutes: 1),
  });

  Future<T> execute<T>(Future<T> Function() operation) async {
    if (_isOpen()) {
      if (_shouldReset()) {
        _reset();
      } else {
        throw AppException('Circuit breaker is open');
      }
    }

    try {
      final result = await operation();
      _onSuccess();
      return result;
    } catch (e) {
      _onFailure();
      rethrow;
    }
  }

  bool _isOpen() => _openedAt != null;

  bool _shouldReset() {
    return _openedAt != null &&
        DateTime.now().difference(_openedAt!) > timeout;
  }

  void _onSuccess() {
    _failureCount = 0;
  }

  void _onFailure() {
    _failureCount++;
    if (_failureCount >= failureThreshold) {
      _openedAt = DateTime.now();
    }
  }

  void _reset() {
    _failureCount = 0;
    _openedAt = null;
  }
}
```

## Best Practices

- Create custom exception classes for different error types
- Catch specific exceptions before general ones
- Use `rethrow` to preserve stack traces
- Always clean up resources in `finally` blocks
- Validate input early and throw meaningful errors
- Use Result/Either types for expected failures
- Implement global error handlers for uncaught errors
- Use retry with exponential backoff for transient failures
- Implement circuit breakers for external service calls
- Don't catch exceptions you can't handle
- Include context in error messages
- Log errors with stack traces
