---
paths:
  - "**/*.cs"
  - "**/*.csx"
---
# C# Patterns

> This file extends [common/patterns.md](../common/patterns.md) with C#-specific content.

## API Response Pattern

```csharp
public sealed record ApiResponse<T>(
    bool Success,
    T? Data = default,
    string? Error = null,
    object? Meta = null);
```

## Repository Pattern

```csharp
public interface IRepository<T>
{
    Task<IReadOnlyList<T>> FindAllAsync(CancellationToken cancellationToken);
    Task<T?> FindByIdAsync(Guid id, CancellationToken cancellationToken);
    Task<T> CreateAsync(T entity, CancellationToken cancellationToken);
    Task<T> UpdateAsync(T entity, CancellationToken cancellationToken);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken);
}

public class UserRepository : IRepository<User>
{
    private readonly DbContext _context;

    public UserRepository(DbContext context) => _context = context;

    public async Task<User?> FindByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        return await _context.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id, cancellationToken);
    }
}
```

## Options Pattern

Use strongly typed options for config instead of reading raw strings throughout the codebase.

```csharp
public sealed class PaymentsOptions
{
    public const string SectionName = "Payments";
    public required string BaseUrl { get; init; }
    public required string ApiKeySecretName { get; init; }
}

// Registration
builder.Services.Configure<PaymentsOptions>(
    builder.Configuration.GetSection(PaymentsOptions.SectionName));

// Usage
public class PaymentsService
{
    private readonly PaymentsOptions _options;
    public PaymentsService(IOptions<PaymentsOptions> options) => _options = options.Value;
}
```

## Dependency Injection

- Depend on interfaces at service boundaries
- Keep constructors focused; if a service needs too many dependencies, split responsibilities
- Register lifetimes intentionally: singleton for stateless/shared services, scoped for request data, transient for lightweight pure workers

```csharp
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddSingleton<IConfiguration>(configuration);
builder.Services.AddTransient<IEmailSender, EmailSender>();
```

## Result Pattern

Handle errors without exceptions:

```csharp
public record Result<T>
{
    public bool IsSuccess { get; init; }
    public T? Value { get; init; }
    public string? Error { get; init; }

    public static Result<T> Success(T value) =>
        new() { IsSuccess = true, Value = value };

    public static Result<T> Failure(string error) =>
        new() { IsSuccess = false, Error = error };
}

// Usage
public async Task<Result<User>> GetUserAsync(Guid id)
{
    var user = await _repository.FindByIdAsync(id, default);
    return user is not null
        ? Result<User>.Success(user)
        : Result<User>.Failure("User not found");
}
```

## Mediator Pattern (MediatR)

Decouple request/response logic:

```csharp
// Request
public record GetUserQuery(string UserId) : IRequest<UserDto>;

// Handler
public class GetUserQueryHandler : IRequestHandler<GetUserQuery, UserDto>
{
    private readonly IUserRepository _repository;

    public GetUserQueryHandler(IUserRepository repository) => _repository = repository;

    public async Task<UserDto> Handle(GetUserQuery request, CancellationToken cancellationToken)
    {
        var user = await _repository.FindByIdAsync(Guid.Parse(request.UserId), cancellationToken);
        return user.ToDto();
    }
}

// Usage in controller
[HttpGet("{id}")]
public async Task<ActionResult<UserDto>> GetUser(string id)
{
    var result = await _mediator.Send(new GetUserQuery(id));
    return Ok(result);
}
```

## Pattern Matching

Use modern C# pattern matching:

```csharp
// Type pattern
public string GetShape(object shape) => shape switch
{
    Circle c => $"Circle with radius {c.Radius}",
    Rectangle r => $"Rectangle {r.Width}x{r.Height}",
    _ => "Unknown shape"
};

// Property pattern
public decimal CalculateDiscount(Customer customer) => customer switch
{
    { IsPremium: true, YearsActive: > 5 } => 0.20m,
    { IsPremium: true } => 0.15m,
    { YearsActive: > 3 } => 0.10m,
    _ => 0m
};
```

## Specification Pattern

Encapsulate business rules:

```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
    Expression<Func<T, bool>> ToExpression();
}

public class ActiveUserSpecification : ISpecification<User>
{
    public bool IsSatisfiedBy(User user) => user.IsActive && !user.IsDeleted;

    public Expression<Func<User, bool>> ToExpression() =>
        user => user.IsActive && !user.IsDeleted;
}
```

## Extension Methods

Create reusable utility methods:

```csharp
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string? value) =>
        string.IsNullOrEmpty(value);
}
```

## Reference

See related C# pattern skills for comprehensive design patterns when available.
