---
paths:
  - "**/*.cs"
  - "**/*.csx"
  - "**/*.csproj"
---
# C# Testing

> This file extends [common/testing.md](../common/testing.md) with C#-specific content.

## Test Framework

- Prefer **xUnit** for unit and integration tests
- Use **FluentAssertions** for readable assertions
- Use **Moq** or **NSubstitute** for mocking dependencies
- Use **Testcontainers** when integration tests need real infrastructure

## Test Naming Convention

Use descriptive names: `MethodName_StateUnderTest_ExpectedBehavior`

```csharp
public sealed class OrderServiceTests
{
    [Fact]
    public async Task FindByIdAsync_ReturnsOrder_WhenOrderExists()
    {
        // Arrange
        // Act
        // Assert
    }

    [Fact]
    public void Withdraw_InsufficientFunds_ThrowsException() { }
}
```

## xUnit Features

```csharp
[Theory]
[InlineData(1, 2, 3)]
[InlineData(-1, -1, -2)]
[InlineData(0, 0, 0)]
public void Add_VariousInputs_ReturnsCorrectSum(int a, int b, int expected)
{
    var result = new Calculator().Add(a, b);
    Assert.Equal(expected, result);
}
```

## Test Organization

- Mirror `src/` structure under `tests/`
- Separate unit, integration, and end-to-end coverage clearly
- Name tests by behavior, not implementation details

```
MyApp/
├── src/
│   └── MyApp.Core/
└── tests/
    ├── MyApp.Core.UnitTests/
    ├── MyApp.Core.IntegrationTests/
    └── MyApp.E2E.Tests/
```

Use traits to categorize and filter tests:

```csharp
[Fact, Trait("Category", "Unit")]
public void UnitTest() { }

[Fact, Trait("Category", "Integration")]
public void IntegrationTest() { }

// dotnet test --filter "Category=Unit"
```

## Mocking

```csharp
[Fact]
public async Task ProcessOrder_ValidOrder_CallsRepository()
{
    var mockRepository = new Mock<IOrderRepository>();
    mockRepository
        .Setup(r => r.SaveAsync(It.IsAny<Order>()))
        .ReturnsAsync(true);

    var service = new OrderService(mockRepository.Object);
    await service.ProcessOrderAsync(new Order { Id = "1", Amount = 100 });

    mockRepository.Verify(
        r => r.SaveAsync(It.Is<Order>(o => o.Id == "1")),
        Times.Once);
}
```

## Test Fixtures

```csharp
public class DatabaseFixture : IDisposable
{
    public DbContext Context { get; } = CreateInMemoryContext();
    public void Dispose() => Context.Dispose();
}

public class UserRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;
    public UserRepositoryTests(DatabaseFixture fixture) => _fixture = fixture;

    [Fact]
    public async Task SaveUser_ValidUser_SavesToDatabase()
    {
        var repository = new UserRepository(_fixture.Context);
        // ... test implementation
    }
}
```

## ASP.NET Core Integration Tests

- Use `WebApplicationFactory<TEntryPoint>` for API integration coverage
- Test auth, validation, and serialization through HTTP, not by bypassing middleware

```csharp
public class ApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public ApiTests(WebApplicationFactory<Program> factory) => _client = factory.CreateClient();

    [Fact]
    public async Task GetUsers_ReturnsSuccessStatusCode()
    {
        var response = await _client.GetAsync("/api/users");
        response.EnsureSuccessStatusCode();
    }
}
```

## Coverage

- Target 80%+ line coverage
- Focus coverage on domain logic, validation, auth, and failure paths
- Run `dotnet test` in CI with coverage collection enabled where available

```bash
dotnet test --collect:"XPlat Code Coverage"

dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coveragereport" -reporttypes:Html
```

## Reference

See related C# testing skills for comprehensive patterns and best practices when available.
