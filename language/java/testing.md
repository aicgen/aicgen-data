# Java Testing (JUnit 5)

## Test Structure

```
src/
├── main/java/com/example/
│   └── UserService.java
└── test/java/com/example/
    └── UserServiceTest.java
```

## Basic Tests

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class UserServiceTest {

    private UserService service;

    @BeforeEach
    void setUp() {
        service = new UserService();
    }

    @Test
    void createUser_WithValidEmail_ReturnsUser() {
        User user = service.create("test@example.com");

        assertEquals("test@example.com", user.getEmail());
        assertNotNull(user.getId());
    }

    @Test
    void createUser_WithInvalidEmail_ThrowsException() {
        assertThrows(ValidationException.class, () -> {
            service.create("invalid");
        });
    }
}
```

## Assertions

```java
// Equality
assertEquals(expected, actual);
assertNotEquals(unexpected, actual);

// Boolean
assertTrue(condition);
assertFalse(condition);

// Null checks
assertNull(value);
assertNotNull(value);

// Collections
assertIterableEquals(expected, actual);
assertArrayEquals(expectedArray, actualArray);

// Multiple assertions (all run even if some fail)
assertAll(
    () -> assertEquals("John", user.getName()),
    () -> assertEquals("john@example.com", user.getEmail()),
    () -> assertTrue(user.isActive())
);
```

## Parameterized Tests

```java
@ParameterizedTest
@ValueSource(strings = {"", " ", "invalid", "no-at-sign"})
void validateEmail_WithInvalidInput_ReturnsFalse(String email) {
    assertFalse(validator.isValid(email));
}

@ParameterizedTest
@CsvSource({
    "test@example.com, true",
    "invalid, false",
    "'', false"
})
void validateEmail_WithVariousInputs(String email, boolean expected) {
    assertEquals(expected, validator.isValid(email));
}

@ParameterizedTest
@MethodSource("provideUsersForTest")
void processUser_WithVariousUsers(User user, String expectedStatus) {
    assertEquals(expectedStatus, processor.process(user));
}

static Stream<Arguments> provideUsersForTest() {
    return Stream.of(
        Arguments.of(new User("active@example.com", true), "processed"),
        Arguments.of(new User("inactive@example.com", false), "skipped")
    );
}
```

## Nested Tests

```java
@DisplayName("UserService")
class UserServiceTest {

    @Nested
    @DisplayName("create")
    class Create {

        @Test
        @DisplayName("with valid email creates user")
        void withValidEmail() {
            // test
        }

        @Test
        @DisplayName("with duplicate email throws exception")
        void withDuplicateEmail() {
            // test
        }
    }

    @Nested
    @DisplayName("delete")
    class Delete {
        // delete tests
    }
}
```

## Mocking with Mockito

```java
import org.mockito.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository repository;

    @InjectMocks
    private UserService service;

    @Test
    void create_SavesUserToRepository() {
        User user = new User("test@example.com");
        when(repository.save(any(User.class))).thenReturn(user);

        User result = service.create("test@example.com");

        verify(repository).save(any(User.class));
        assertEquals("test@example.com", result.getEmail());
    }

    @Test
    void findById_WhenNotFound_ThrowsException() {
        when(repository.findById("123")).thenReturn(Optional.empty());

        assertThrows(NotFoundException.class, () -> {
            service.findById("123");
        });
    }
}
```

## Lifecycle Hooks

```java
@BeforeAll
static void initAll() {
    // Run once before all tests
}

@BeforeEach
void init() {
    // Run before each test
}

@AfterEach
void tearDown() {
    // Run after each test
}

@AfterAll
static void tearDownAll() {
    // Run once after all tests
}
```

## Async Testing

```java
@Test
void asyncOperation_CompletesWithinTimeout() {
    assertTimeout(Duration.ofSeconds(2), () -> {
        service.longRunningOperation();
    });
}

@Test
void asyncOperation_WithCompletableFuture() throws Exception {
    CompletableFuture<User> future = service.createAsync("test@example.com");

    User user = future.get(5, TimeUnit.SECONDS);
    assertEquals("test@example.com", user.getEmail());
}
```

## Test Tags and Filtering

```java
@Tag("integration")
@Test
void databaseIntegrationTest() {
    // integration test
}

@Tag("slow")
@Test
void slowPerformanceTest() {
    // slow test
}

// Run: mvn test -Dgroups="integration"
// Exclude: mvn test -DexcludedGroups="slow"
```

## Spring Boot Testing

```java
@SpringBootTest
class UserServiceIntegrationTest {

    @Autowired
    private UserService service;

    @MockBean
    private EmailService emailService;

    @Test
    void create_SendsWelcomeEmail() {
        service.create("test@example.com");

        verify(emailService).sendWelcome(any(User.class));
    }
}

@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService service;

    @Test
    void getUser_ReturnsUser() throws Exception {
        when(service.findById("1")).thenReturn(new User("1", "test@example.com"));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.email").value("test@example.com"));
    }
}
```
