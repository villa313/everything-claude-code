---
description: Comprehensive C# / ASP.NET Core code review for async correctness, EF Core patterns, nullable reference types, security, and idiomatic .NET. Invokes the csharp-reviewer agent.
---

# C# Code Review

This command invokes the **csharp-reviewer** agent for comprehensive C#-specific code review.

## What This Command Does

1. **Identify C# Changes**: Find modified `.cs` files via `git diff`
2. **Build & Format Check**: Run `dotnet build` and `dotnet format --verify-no-changes`
3. **Security Scan**: Check for SQL injection, path traversal, hardcoded secrets, missing `[Authorize]`
4. **Async Correctness**: Analyze `.Result`/`.Wait()` deadlocks, `async void`, cancellation propagation
5. **EF Core Review**: N+1 queries, missing `AsNoTracking`, unbounded result sets
6. **Idiomatic C# Check**: Nullable reference types, `record` usage, DI lifetimes
7. **Generate Report**: Categorize issues by severity with a summary table

## When to Use

Use `/csharp-review` when:
- After writing or modifying C# code
- Before committing C# changes
- Reviewing pull requests with C# / ASP.NET Core code
- Onboarding to a new .NET codebase
- Learning idiomatic .NET patterns

## Review Categories

### CRITICAL (Must Fix)
- SQL/command injection vulnerabilities
- Path traversal via unvalidated file paths
- Hardcoded secrets or connection strings
- Missing `[Authorize]` on protected endpoints
- Empty catch blocks / swallowed exceptions
- Blocking on async (`.Result`, `.Wait()`) — deadlock risk

### HIGH (Should Fix)
- Business logic in controllers instead of the service layer
- EF entity returned directly from a controller (use a DTO)
- N+1 queries and missing `AsNoTracking`
- `async void` outside event handlers
- Missing `CancellationToken` propagation
- Nullable warnings suppressed with `!` without justification

### MEDIUM (Consider)
- Mutable `class` where a `record` fits
- Captive dependency (singleton depending on scoped)
- Missing `sealed` on non-inherited classes
- `string.Format` instead of interpolation
- Weak test names, `Thread.Sleep` in tests

## Automated Checks Run

```bash
# Compilation and formatting
dotnet build -q
dotnet format --verify-no-changes

# Tests
dotnet test --no-build -q

# Targeted scans
grep -rn "\.Result\b\|\.Wait()" src/ --include="*.cs"
grep -rn "Html\.Raw\b" src/ --include="*.cshtml"

# Vulnerable dependencies
dotnet list package --vulnerable
```

## Example Usage

```text
User: /csharp-review

Agent:
# C# Code Review Report

## Files Reviewed
- src/Api/Controllers/OrdersController.cs (modified)
- src/Infrastructure/Repositories/OrderRepository.cs (modified)

## Build & Format Results
✓ dotnet build: succeeded
✓ dotnet format: no changes needed

## Issues Found

[CRITICAL] SQL injection via interpolated raw query
File: src/Infrastructure/Repositories/OrderRepository.cs:47
Issue: FromSqlRaw($"SELECT * FROM Orders WHERE Id = '{id}'") — user input concatenated into SQL.
Fix: Use FromSqlRaw("SELECT * FROM Orders WHERE Id = {0}", id) or LINQ .Where(o => o.Id == id).

[HIGH] EF entity returned directly from controller
File: src/Api/Controllers/OrdersController.cs:32
Issue: Action returns the Order entity — exposes internal fields, risks circular serialization.
Fix: Map to OrderDto before returning, or project to a record in the query.

## Review Summary

| Severity | Count | Status |
|----------|-------|--------|
| CRITICAL | 1     | block  |
| HIGH     | 1     | block  |
| MEDIUM   | 0     | info   |

Recommendation: FAIL — Block merge until CRITICAL and HIGH issues are fixed.
```

## Approval Criteria

| Status | Condition |
|--------|-----------|
| PASS: Approve | No CRITICAL or HIGH issues |
| WARNING: Warning | Only MEDIUM issues (merge with caution) |
| FAIL: Block | CRITICAL or HIGH issues found |

## Integration with Other Commands

- Use `/csharp-test` first to ensure tests pass
- Use `/csharp-build` if build errors occur
- Use `/csharp-review` before committing
- Use `/code-review` for non-C# specific concerns

## Related

- Agent: `agents/csharp-reviewer.md`
- Skills: `skills/dotnet-patterns/`, `skills/csharp-testing/`
- Rules: `rules/csharp/`
