# Java Fundamentals

## Project Structure (Maven/Gradle)

```
myproject/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/
│   │   │       ├── Application.java
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       └── model/
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       └── java/
├── pom.xml (Maven) or build.gradle (Gradle)
```

## Naming Conventions

```java
// Classes: PascalCase
public class UserService {}

// Interfaces: PascalCase, often -able suffix
public interface Comparable {}
public interface UserRepository {}

// Methods/variables: camelCase
public void createUser() {}
private String userName;

// Constants: UPPER_SNAKE_CASE
public static final int MAX_RETRIES = 3;

// Packages: lowercase
package com.example.service;
```

## Modern Java Features (17+)

```java
// Records for immutable data
public record User(String id, String email, String name) {}

// Pattern matching
if (obj instanceof String s) {
    System.out.println(s.length());
}

// Switch expressions
String result = switch (status) {
    case ACTIVE -> "Active";
    case PENDING -> "Pending";
    case DELETED -> "Deleted";
};

// Text blocks
String json = """
    {
        "name": "test",
        "value": 123
    }
    """;

// Sealed classes
public sealed interface Shape permits Circle, Rectangle {}
```

## Exception Handling

```java
// Checked exceptions
public User findUser(String id) throws UserNotFoundException {
    return repository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}

// Try-with-resources
try (var reader = new BufferedReader(new FileReader(path))) {
    return reader.readLine();
} catch (IOException e) {
    throw new DataAccessException("Failed to read file", e);
}

// Custom exceptions
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String id) {
        super("User not found: " + id);
    }
}
```

## Optional

```java
// Avoid null, use Optional
public Optional<User> findUser(String id) {
    return Optional.ofNullable(repository.find(id));
}

// Chaining
findUser(id)
    .map(User::getEmail)
    .filter(email -> email.contains("@"))
    .orElse("unknown@example.com");

// Never use Optional in fields or parameters
// Only for return types
```

## Streams

```java
// Filter, map, collect
List<String> emails = users.stream()
    .filter(u -> u.isActive())
    .map(User::getEmail)
    .collect(Collectors.toList());

// Grouping
Map<String, List<User>> byDepartment = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment));

// Reduce
int totalAge = users.stream()
    .mapToInt(User::getAge)
    .sum();
```

## Testing (JUnit 5)

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository repository;

    @InjectMocks
    private UserService service;

    @Test
    void shouldCreateUser() {
        // Arrange
        var request = new CreateUserRequest("test@example.com");
        when(repository.save(any())).thenReturn(new User("1", "test@example.com"));

        // Act
        var user = service.create(request);

        // Assert
        assertThat(user.getEmail()).isEqualTo("test@example.com");
        verify(repository).save(any());
    }

    @ParameterizedTest
    @ValueSource(strings = {"invalid", "no-at", ""})
    void shouldRejectInvalidEmails(String email) {
        assertThrows(ValidationException.class,
            () -> service.create(new CreateUserRequest(email)));
    }
}
```

## Dependency Injection (Spring)

```java
@Service
public class UserService {
    private final UserRepository repository;
    private final EmailService emailService;

    // Constructor injection (preferred)
    public UserService(UserRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
}
```
