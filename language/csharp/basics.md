# C# Fundamentals

## Project Structure

```
MyProject/
├── src/
│   ├── MyProject.Api/
│   │   ├── Controllers/
│   │   ├── Program.cs
│   │   └── MyProject.Api.csproj
│   ├── MyProject.Core/
│   │   ├── Entities/
│   │   ├── Interfaces/
│   │   └── Services/
│   └── MyProject.Infrastructure/
│       ├── Data/
│       └── Services/
├── tests/
│   └── MyProject.Tests/
└── MyProject.sln
```

## Naming Conventions

```csharp
// Classes, interfaces, methods: PascalCase
public class UserService { }
public interface IUserRepository { }
public void CreateUser() { }

// Interface prefix: I
public interface IRepository { }

// Private fields: _camelCase
private readonly IUserRepository _repository;

// Parameters, local variables: camelCase
public User GetUser(string userId) { }

// Constants: PascalCase
public const int MaxRetries = 3;
```

## Modern C# Features (10+)

```csharp
// Records for immutable data
public record User(string Id, string Email, string Name);

// Pattern matching
var result = obj switch
{
    User u when u.IsActive => "Active user",
    User u => "Inactive user",
    null => "No user",
    _ => "Unknown"
};

// Null handling
string? nullableString = GetValue();
string nonNull = nullableString ?? "default";
int length = nullableString?.Length ?? 0;

// Target-typed new
List<User> users = new();
Dictionary<string, int> counts = new();

// File-scoped namespaces
namespace MyProject.Services;

public class UserService { }
```

## Async/Await

```csharp
// Async methods return Task
public async Task<User> GetUserAsync(string id)
{
    var user = await _repository.FindByIdAsync(id);
    return user ?? throw new NotFoundException(id);
}

// Async enumerable
public async IAsyncEnumerable<User> GetUsersAsync()
{
    await foreach (var user in _repository.StreamAsync())
    {
        yield return user;
    }
}

// Cancellation
public async Task ProcessAsync(CancellationToken ct)
{
    ct.ThrowIfCancellationRequested();
    await _service.DoWorkAsync(ct);
}
```

## LINQ

```csharp
// Query syntax
var activeUsers = from u in users
                  where u.IsActive
                  orderby u.Name
                  select u;

// Method syntax (preferred)
var emails = users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .Select(u => u.Email)
    .ToList();

// Grouping
var byDepartment = users
    .GroupBy(u => u.Department)
    .ToDictionary(g => g.Key, g => g.ToList());
```

## Exception Handling

```csharp
// Custom exceptions
public class NotFoundException : Exception
{
    public NotFoundException(string id)
        : base($"Entity not found: {id}") { }
}

// Try-catch-finally
try
{
    await ProcessAsync();
}
catch (NotFoundException ex) when (ex.Message.Contains("user"))
{
    _logger.LogWarning(ex, "User not found");
    throw;
}
catch (Exception ex)
{
    _logger.LogError(ex, "Unexpected error");
    throw;
}
```

## Dependency Injection

```csharp
// Service registration
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IUserService, UserService>();

// Constructor injection
public class UserService : IUserService
{
    private readonly IUserRepository _repository;
    private readonly ILogger<UserService> _logger;

    public UserService(IUserRepository repository, ILogger<UserService> logger)
    {
        _repository = repository;
        _logger = logger;
    }
}
```

## Testing (xUnit)

```csharp
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _repositoryMock;
    private readonly UserService _sut;

    public UserServiceTests()
    {
        _repositoryMock = new Mock<IUserRepository>();
        _sut = new UserService(_repositoryMock.Object);
    }

    [Fact]
    public async Task GetUser_WhenExists_ReturnsUser()
    {
        // Arrange
        var user = new User("1", "test@example.com", "Test");
        _repositoryMock.Setup(r => r.FindByIdAsync("1"))
            .ReturnsAsync(user);

        // Act
        var result = await _sut.GetUserAsync("1");

        // Assert
        Assert.Equal("test@example.com", result.Email);
    }

    [Theory]
    [InlineData("")]
    [InlineData(null)]
    public async Task GetUser_WhenInvalidId_ThrowsArgumentException(string id)
    {
        await Assert.ThrowsAsync<ArgumentException>(
            () => _sut.GetUserAsync(id));
    }
}
```
