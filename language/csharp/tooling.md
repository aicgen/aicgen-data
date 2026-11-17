# C# And .NET Tooling

## Project Baseline

- Pin the SDK with `global.json`, target a supported **LTS** framework (`net8.0` or newer LTS), and enable analyzers and nullable reference types for every project.
- Centralise settings in `Directory.Build.props` and package versions in `Directory.Packages.props` (Central Package Management) so projects cannot drift.

```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>latest</LangVersion>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
  </PropertyGroup>
</Project>
```

```xml
<!-- Directory.Packages.props -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="xunit" Version="2.9.2" />
  </ItemGroup>
</Project>
```

## The Required Gate

```bash
dotnet restore --locked-mode
dotnet format --verify-no-changes
dotnet build -c Release --no-restore -warnaserror
dotnet test -c Release --no-build
dotnet list package --vulnerable --include-transitive
```

## Modern C# Idioms

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0m, currency);
}

public interface IUserRepository
{
    Task<User?> FindAsync(int id, CancellationToken cancellationToken);
}

public sealed class UserService(IUserRepository users, TimeProvider clock)   // primary constructor
{
    public async Task<string> DescribeAsync(int id, CancellationToken cancellationToken)
    {
        var user = await users.FindAsync(id, cancellationToken)
            ?? throw new KeyNotFoundException($"user {id} not found");

        return user switch
        {
            { IsActive: false } => "inactive",
            { CreatedAt: var created } when created > clock.GetUtcNow().AddDays(-7) => "new",
            _ => "active",
        };
    }
}

public sealed record User(int Id, bool IsActive, DateTimeOffset CreatedAt);
```

- Use **records** for immutable data, **pattern matching** and `switch` expressions for branching, `required` members for mandatory properties, and `init` setters.
- Treat nullable warnings as errors; do not silence them with `!` without a comment explaining the invariant.
- Accept a `CancellationToken` in every async method that does I/O and pass it through. Use `async`/`await` end to end; never `.Result` or `.Wait()`.
- Inject `TimeProvider` and `ILogger<T>`; use `IHttpClientFactory` rather than `new HttpClient()`.
- Use `IAsyncEnumerable<T>` for streams and `await using` for async disposables.
- Use structured logging (`logger.LogInformation("Created {UserId}", id)`), never string interpolation in log templates, and never log secrets.

## ASP.NET Core

- Minimal APIs or controllers with `[ApiController]`; validate input at the edge; return `ProblemDetails` for errors.
- Bind configuration to validated options (`AddOptions<T>().BindConfiguration(...).ValidateDataAnnotations().ValidateOnStart()`).
- Authentication and authorisation through policies; never trust client-supplied roles or user ids. Keep secrets in user-secrets locally and a secret store in production.
- Use Entity Framework Core with parameterised queries, `AsNoTracking()` for reads, and migrations reviewed in code review.

## Testing

- xUnit (or NUnit/MSTest) with `FluentAssertions`/`Shouldly` optional; `WebApplicationFactory<T>` for API integration tests; Testcontainers for real databases.
- Prefer fakes over heavy mocking frameworks; use `FakeTimeProvider` instead of sleeping.

## Quick Checklist

- [ ] `global.json`, nullable, analyzers and warnings-as-errors configured centrally
- [ ] `dotnet format`, build, tests and vulnerable-package check run in CI
- [ ] CancellationToken passed through all async I/O
- [ ] No `.Result`/`.Wait()`, no secrets in logs or source
