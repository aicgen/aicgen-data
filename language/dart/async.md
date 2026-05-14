# Async Programming in Dart

## Futures and async/await

```dart
// ✅ Async function returns Future
Future<User> fetchUser(String id) async {
  final response = await http.get('/api/users/$id');
  return User.fromJson(response.data);
}

// ✅ Handle errors with try-catch
Future<User> fetchUserSafely(String id) async {
  try {
    final response = await http.get('/api/users/$id');
    return User.fromJson(response.data);
  } catch (e) {
    print('Failed to fetch user: $e');
    rethrow;
  }
}

// ❌ Don't forget await
Future<void> badExample() async {
  final user = fetchUser('123');   // Missing await!
  print(user);                     // Prints: Instance of 'Future<User>'
}

// ✅ Correct
Future<void> goodExample() async {
  final user = await fetchUser('123');
  print(user.name);                // Prints: Alice
}
```

## Parallel Execution

```dart
// ❌ Sequential - slow
Future<void> sequential() async {
  final user = await fetchUser('1');       // 100ms
  final posts = await fetchPosts('1');     // 150ms
  final comments = await fetchComments('1'); // 120ms
  // Total: 370ms
}

// ✅ Parallel - fast
Future<void> parallel() async {
  final results = await Future.wait([
    fetchUser('1'),
    fetchPosts('1'),
    fetchComments('1'),
  ]);
  // Total: 150ms (longest operation)

  final user = results[0] as User;
  final posts = results[1] as List<Post>;
  final comments = results[2] as List<Comment>;
}

// ✅ With better typing using records (Dart 3.0+)
Future<(User, List<Post>, List<Comment>)> fetchUserData(String id) async {
  final (user, posts, comments) = await (
    fetchUser(id),
    fetchPosts(id),
    fetchComments(id),
  ).wait;

  return (user, posts, comments);
}
```

## Future Methods

```dart
// timeout - fail if takes too long
Future<User> fetchWithTimeout(String id) async {
  return fetchUser(id).timeout(
    const Duration(seconds: 5),
    onTimeout: () => throw TimeoutException('Request timed out'),
  );
}

// catchError - handle errors without try-catch
final user = await fetchUser('123').catchError((error) {
  print('Error: $error');
  return User.guest();             // Fallback value
});

// then - chain operations (prefer async/await)
fetchUser('123')
  .then((user) => user.name)
  .then((name) => print(name));

// whenComplete - always runs (like finally)
await fetchUser('123').whenComplete(() {
  print('Request completed');
});
```

## Streams

```dart
// Create stream
Stream<int> countStream(int max) async* {
  for (int i = 1; i <= max; i++) {
    await Future.delayed(const Duration(seconds: 1));
    yield i;
  }
}

// ✅ Listen to stream
final subscription = countStream(5).listen(
  (count) {
    print('Count: $count');
  },
  onError: (error) {
    print('Error: $error');
  },
  onDone: () {
    print('Stream completed');
  },
);

// Cancel subscription
await subscription.cancel();

// ✅ Async for loop
Future<void> processStream() async {
  await for (final count in countStream(5)) {
    print('Count: $count');
  }
  print('Done');
}
```

## Stream Transformations

```dart
// Transform stream data
Stream<String> getUserNames() {
  return fetchUsersStream()
    .map((user) => user.name)
    .where((name) => name.isNotEmpty);
}

// Async map
Stream<User> enrichUsers(Stream<String> userIds) {
  return userIds.asyncMap((id) async {
    return await fetchUser(id);
  });
}

// Combine streams
final combined = StreamGroup.merge([stream1, stream2, stream3]);

// Buffer events
final buffered = stream.transform(
  StreamTransformer.fromHandlers(
    handleData: (data, sink) {
      sink.add(data);
    },
  ),
);
```

## StreamController

```dart
class ChatService {
  final _messagesController = StreamController<Message>.broadcast();

  Stream<Message> get messages => _messagesController.stream;

  void sendMessage(Message message) {
    _messagesController.add(message);
  }

  void dispose() {
    _messagesController.close();
  }
}

// Usage
final chat = ChatService();

final subscription = chat.messages.listen((message) {
  print('New message: ${message.text}');
});

chat.sendMessage(Message(text: 'Hello'));

// Clean up
await subscription.cancel();
chat.dispose();
```

## Isolates (Heavy Computation)

```dart
// Run expensive computation in isolate
Future<int> calculateInIsolate(int n) async {
  return await Isolate.run(() {
    int result = 0;
    for (int i = 0; i < n; i++) {
      result += i;
    }
    return result;
  });
}

// ✅ For CPU-intensive tasks
Future<Image> processImage(File imageFile) async {
  final bytes = await imageFile.readAsBytes();

  // Run in isolate to avoid blocking UI
  final processed = await Isolate.run(() {
    return applyFilters(bytes);
  });

  return Image.memory(processed);
}
```

## Completer

```dart
// Create Future manually
class DatabaseConnection {
  final Completer<void> _readyCompleter = Completer();

  Future<void> get ready => _readyCompleter.future;

  void connect() async {
    try {
      await _performConnection();
      _readyCompleter.complete();
    } catch (e) {
      _readyCompleter.completeError(e);
    }
  }
}

// Usage
final db = DatabaseConnection();
db.connect();
await db.ready;                  // Wait until connected
```

## Error Handling

```dart
// ✅ Use try-catch with async/await
Future<void> safeOperation() async {
  try {
    await riskyOperation();
  } on NetworkException catch (e) {
    print('Network error: ${e.message}');
  } on TimeoutException {
    print('Operation timed out');
  } catch (e, stackTrace) {
    print('Unexpected error: $e');
    print('Stack trace: $stackTrace');
    rethrow;
  }
}

// ✅ Handle stream errors
stream.listen(
  (data) => process(data),
  onError: (error) {
    if (error is NetworkException) {
      reconnect();
    }
  },
);
```

## Best Practices

- Always use `async`/`await` for readability
- Run independent operations in parallel with `Future.wait`
- Set timeouts for network operations
- Cancel subscriptions when done
- Use isolates for CPU-intensive work
- Handle errors explicitly
- Close StreamControllers when finished
- Use `async*` and `yield` for generating stream data
- Prefer `await for` over `listen` for sequential processing
