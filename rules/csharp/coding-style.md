---
paths:
  - "**/*.cs"
  - "**/*.csx"
---
# C# Coding Style

> This file extends [common/coding-style.md](../common/coding-style.md) with C#-specific content.

## Standards

- Follow current .NET conventions and enable nullable reference types
- Prefer explicit access modifiers on public and internal APIs
- Keep files aligned with the primary type they define
- Prefer expression-bodied members where appropriate

## Types and Models

- Prefer `record` or `record struct` for immutable value-like models
- Use `class` for entities or types with identity and lifecycle
- Use `interface` for service boundaries and abstractions
- Avoid `dynamic` in application code; prefer generics or explicit models

```csharp
public sealed record UserDto(Guid Id, string Email);

public interface IUserRepository
{
    Task<UserDto?> FindByIdAsync(Guid id, CancellationToken cancellationToken);
}
```

## Immutability

- Prefer `init` setters, constructor parameters, and immutable collections for shared state
- Do not mutate input models in-place when producing updated state

```csharp
public sealed record UserProfile(string Name, string Email);

public static UserProfile Rename(UserProfile profile, string name) =>
    profile with { Name = name };

// Immutable collection usage
using System.Collections.Immutable;
var list = ImmutableList.Create(1, 2, 3);
var newList = list.Add(4);  // Returns new instance
```

## Naming Conventions

- **PascalCase**: Classes, methods, properties, public fields
- **camelCase**: Local variables, parameters
- **_camelCase**: Private fields (with underscore prefix)
- **IPascalCase**: Interfaces (prefix with 'I')

```csharp
public interface IUserService
{
    Task<User> GetUserAsync(string userId);
}

public class UserService : IUserService
{
    private readonly ILogger<UserService> _logger;

    public async Task<User> GetUserAsync(string userId)
    {
        var cachedUser = await GetFromCacheAsync(userId);
        return cachedUser;
    }
}
```

## Async and Error Handling

- Prefer `async`/`await` over blocking calls like `.Result` or `.Wait()`
- Suffix async methods with `Async`; pass `CancellationToken` through public async APIs
- Use `ConfigureAwait(false)` in library code; avoid `async void` except for event handlers
- Throw specific exceptions and log with structured properties

```csharp
public async Task<Order> LoadOrderAsync(
    Guid orderId,
    CancellationToken cancellationToken)
{
    try
    {
        return await repository.FindAsync(orderId, cancellationToken)
            ?? throw new InvalidOperationException($"Order {orderId} was not found.");
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to load order {OrderId}", orderId);
        throw;
    }
}

// WRONG: Catching Exception or using empty catch
try { DoSomething(); }
catch (Exception) { }  // Never do this
```

## Null Safety

Enable nullable reference types in your project:

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

```csharp
// Nullable reference type
public string? MiddleName { get; set; }

// Non-nullable with null-forgiving operator (use sparingly)
public string FirstName { get; set; } = null!;

// Null checking with pattern matching
if (user is { Email: not null })
{
    SendEmail(user.Email);
}
```

## Formatting

- Use `dotnet format` for formatting and analyzer fixes
- Keep `using` directives organized and remove unused imports
- Prefer expression-bodied members only when they stay readable

```bash
# Format code
dotnet format

# Run analyzers
dotnet build /p:EnforceCodeStyleInBuild=true
```

## LINQ Best Practices

Prefer method syntax and avoid unnecessary materialization:

```csharp
// CORRECT: Method syntax
var adults = users
    .Where(u => u.Age >= 18)
    .OrderBy(u => u.Name)
    .Select(u => u.Name);

// Avoid unnecessary ToList() calls
var count = users.Count(u => u.IsActive);  // CORRECT
var count = users.Where(u => u.IsActive).ToList().Count();  // WRONG
```

## Dependency Injection

- Depend on interfaces at service boundaries
- Keep constructors focused; if a service needs too many dependencies, split responsibilities
- Register lifetimes intentionally: singleton for stateless/shared services, scoped for request data, transient for lightweight pure workers

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderRepository repository,
        ILogger<OrderService> logger)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }
}
```

## Reference

See related skills for comprehensive C# patterns and best practices when available.
