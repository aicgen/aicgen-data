# C# Testing (xUnit)

## Test Structure

```
MyProject/
├── src/
│   └── MyProject/
│       └── Services/UserService.cs
└── tests/
    └── MyProject.Tests/
        └── Services/UserServiceTests.cs
```

## Basic Tests

```csharp
using Xunit;

public class UserServiceTests
{
    private readonly UserService _sut; // System Under Test

    public UserServiceTests()
    {
        _sut = new UserService();
    }

    [Fact]
    public void Create_WithValidEmail_ReturnsUser()
    {
        var user = _sut.Create("test@example.com");

        Assert.NotNull(user);
        Assert.Equal("test@example.com", user.Email);
    }

    [Fact]
    public void Create_WithInvalidEmail_ThrowsValidationException()
    {
        Assert.Throws<ValidationException>(() => _sut.Create("invalid"));
    }
}
```

## Assertions

```csharp
// Equality
Assert.Equal(expected, actual);
Assert.NotEqual(unexpected, actual);

// Boolean
Assert.True(condition);
Assert.False(condition);

// Null
Assert.Null(value);
Assert.NotNull(value);

// Collections
Assert.Empty(collection);
Assert.NotEmpty(collection);
Assert.Contains(item, collection);
Assert.DoesNotContain(item, collection);
Assert.All(collection, item => Assert.NotNull(item.Name));

// Type
Assert.IsType<User>(result);
Assert.IsAssignableFrom<IEntity>(result);

// Exceptions
var ex = Assert.Throws<NotFoundException>(() => service.Find("invalid"));
Assert.Equal("User not found", ex.Message);
```

## Theory Tests (Parameterized)

```csharp
[Theory]
[InlineData("test@example.com", true)]
[InlineData("invalid", false)]
[InlineData("", false)]
public void IsValidEmail_WithVariousInputs_ReturnsExpected(string email, bool expected)
{
    var result = _validator.IsValid(email);
    Assert.Equal(expected, result);
}

[Theory]
[MemberData(nameof(GetTestUsers))]
public void ProcessUser_WithVariousUsers_ReturnsExpected(User user, string expectedStatus)
{
    var result = _processor.Process(user);
    Assert.Equal(expectedStatus, result);
}

public static IEnumerable<object[]> GetTestUsers()
{
    yield return new object[] { new User("active@example.com") { IsActive = true }, "processed" };
    yield return new object[] { new User("inactive@example.com") { IsActive = false }, "skipped" };
}
```

## Test Classes and Collections

```csharp
// Shared context across tests in a class
public class UserServiceTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public UserServiceTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public void Test1() { /* uses _fixture */ }
}

// Shared context across multiple test classes
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database")]
public class UserServiceTests { }

[Collection("Database")]
public class OrderServiceTests { }
```

## Mocking with Moq

```csharp
using Moq;

public class UserServiceTests
{
    private readonly Mock<IUserRepository> _repositoryMock;
    private readonly Mock<ILogger<UserService>> _loggerMock;
    private readonly UserService _sut;

    public UserServiceTests()
    {
        _repositoryMock = new Mock<IUserRepository>();
        _loggerMock = new Mock<ILogger<UserService>>();
        _sut = new UserService(_repositoryMock.Object, _loggerMock.Object);
    }

    [Fact]
    public async Task GetUser_WhenExists_ReturnsUser()
    {
        // Arrange
        var user = new User("1", "test@example.com");
        _repositoryMock.Setup(r => r.FindByIdAsync("1"))
            .ReturnsAsync(user);

        // Act
        var result = await _sut.GetUserAsync("1");

        // Assert
        Assert.Equal("test@example.com", result.Email);
        _repositoryMock.Verify(r => r.FindByIdAsync("1"), Times.Once);
    }

    [Fact]
    public async Task GetUser_WhenNotFound_ThrowsNotFoundException()
    {
        _repositoryMock.Setup(r => r.FindByIdAsync(It.IsAny<string>()))
            .ReturnsAsync((User?)null);

        await Assert.ThrowsAsync<NotFoundException>(
            () => _sut.GetUserAsync("invalid"));
    }
}
```

## Async Testing

```csharp
[Fact]
public async Task CreateAsync_WithValidData_ReturnsUser()
{
    var user = await _sut.CreateAsync("test@example.com");

    Assert.NotNull(user);
    Assert.Equal("test@example.com", user.Email);
}

[Fact]
public async Task CreateAsync_WithTimeout_CompletesInTime()
{
    var task = _sut.CreateAsync("test@example.com");
    var completed = await Task.WhenAny(task, Task.Delay(5000)) == task;

    Assert.True(completed, "Operation timed out");
}
```

## Test Output

```csharp
public class UserServiceTests
{
    private readonly ITestOutputHelper _output;

    public UserServiceTests(ITestOutputHelper output)
    {
        _output = output;
    }

    [Fact]
    public void Test_WithLogging()
    {
        _output.WriteLine("Starting test...");
        // test logic
        _output.WriteLine($"Result: {result}");
    }
}
```

## Integration Testing with WebApplicationFactory

```csharp
public class UserApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UserApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace real services with test doubles
                services.AddScoped<IUserRepository, InMemoryUserRepository>();
            });
        }).CreateClient();
    }

    [Fact]
    public async Task GetUser_ReturnsUser()
    {
        var response = await _client.GetAsync("/api/users/1");

        response.EnsureSuccessStatusCode();
        var user = await response.Content.ReadFromJsonAsync<User>();
        Assert.Equal("test@example.com", user?.Email);
    }
}
```
