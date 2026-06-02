---
description: Enforce TDD workflow for C#. Write xUnit tests first, then implement. Verify 80%+ coverage with coverlet / dotnet test --collect.
---

# C# TDD Command

This command enforces test-driven development methodology for C# code using idiomatic .NET testing patterns (xUnit, FluentAssertions, NSubstitute).

## What This Command Does

1. **Define Types/Interfaces**: Scaffold class and method signatures first
2. **Write Tests**: Create comprehensive xUnit test cases (RED)
3. **Run Tests**: Verify tests fail for the right reason
4. **Implement Code**: Write minimal code to pass (GREEN)
5. **Refactor**: Improve while keeping tests green
6. **Check Coverage**: Ensure 80%+ coverage

## When to Use

Use `/csharp-test` when:
- Implementing new C# classes or methods
- Adding test coverage to existing code
- Fixing bugs (write failing test first)
- Building critical business logic
- Learning TDD workflow in .NET

## TDD Cycle

```text
RED      → Write failing xUnit test
GREEN    → Implement minimal code to pass
REFACTOR → Improve code, tests stay green
REPEAT   → Next test case
```

## Test Stack

- **xUnit** — test framework (`[Fact]`, `[Theory]` + `[InlineData]`)
- **FluentAssertions** — readable assertions (`result.Should().Be(...)`)
- **NSubstitute** — mocking dependencies
- **WebApplicationFactory** — ASP.NET Core integration tests
- **coverlet** — code coverage (`--collect:"XPlat Code Coverage"`)

## Example Session

````text
User: /csharp-test I need a service to validate email addresses

Agent:
# TDD Session: Email Validator

## Step 1: Define Interface

```csharp
// Application/Validation/IEmailValidator.cs
namespace MyApp.Application.Validation;

public interface IEmailValidator
{
    bool IsValid(string email);
}

public sealed class EmailValidator : IEmailValidator
{
    public bool IsValid(string email) => throw new NotImplementedException();
}
```

## Step 2: Write Tests (RED)

```csharp
// Tests/Validation/EmailValidatorTests.cs
using FluentAssertions;
using Xunit;

public class EmailValidatorTests
{
    private readonly EmailValidator _sut = new();

    [Theory]
    [InlineData("user@example.com")]
    [InlineData("first.last@sub.domain.org")]
    public void IsValid_WithValidAddress_ReturnsTrue(string email)
        => _sut.IsValid(email).Should().BeTrue();

    [Theory]
    [InlineData("")]
    [InlineData("no-at-sign")]
    [InlineData("@example.com")]
    [InlineData("user@")]
    public void IsValid_WithInvalidAddress_ReturnsFalse(string email)
        => _sut.IsValid(email).Should().BeFalse();
}
```

```bash
$ dotnet test --no-build -q
# Fails: NotImplementedException (RED — expected)
```

## Step 3: Implement (GREEN)

```csharp
public bool IsValid(string email)
{
    if (string.IsNullOrWhiteSpace(email)) return false;
    var at = email.IndexOf('@');
    return at > 0 && at < email.Length - 1 && email.IndexOf('@', at + 1) < 0;
}
```

```bash
$ dotnet test -q
Passed!  - Failed: 0, Passed: 6, Skipped: 0
```

## Step 4: Coverage

```bash
$ dotnet test --collect:"XPlat Code Coverage"
EmailValidator: 100% line coverage
```
````

## Coverage Verification

```bash
# Run tests with coverage collection
dotnet test --collect:"XPlat Code Coverage"

# Generate a human-readable report (if reportgenerator is installed)
reportgenerator -reports:**/coverage.cobertura.xml -targetdir:coveragereport
```

## Test Naming Convention

Use `MethodName_StateUnderTest_ExpectedBehavior`:

- `IsValid_WithValidAddress_ReturnsTrue`
- `GetById_WhenNotFound_ThrowsNotFoundException`
- `Charge_WithInsufficientFunds_ReturnsDeclined`

## Anti-Patterns to Avoid

| Anti-pattern | Use instead |
|--------------|-------------|
| `Thread.Sleep` in tests | `await Task.Delay` or a time abstraction |
| `async` test returning `void` | Return `Task` so failures are observed |
| `new DbContext()` in tests | `WebApplicationFactory` or `UseInMemoryDatabase` |
| Asserting on multiple unrelated things | One logical assertion per test |
| Weak names like `TestGetUser` | `MethodName_StateUnderTest_ExpectedBehavior` |

## Related Commands

- `/csharp-build` — Fix build errors before testing
- `/csharp-review` — Review code quality after tests pass
- `tdd-workflow` skill — Language-agnostic TDD methodology

## Related

- Agent: `agents/csharp-reviewer.md`
- Skills: `skills/csharp-testing/`, `skills/dotnet-patterns/`
- Rules: `rules/csharp/testing.md`
