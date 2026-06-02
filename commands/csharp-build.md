---
description: Fix C# / .NET build errors, CS compiler errors, and NuGet dependency issues incrementally. Invokes the csharp-build-resolver agent for minimal, surgical fixes.
---

# C# Build and Fix

This command invokes the **csharp-build-resolver** agent to incrementally fix C#/.NET build errors with minimal changes.

## What This Command Does

1. **Run Diagnostics**: Execute `dotnet build`, `dotnet restore`, `dotnet format --verify-no-changes`
2. **Parse Errors**: Group by file and CS error code, sort by severity
3. **Fix Incrementally**: One error at a time
4. **Verify Each Fix**: Re-run `dotnet build` after each change
5. **Report Summary**: Show what was fixed and what remains

## When to Use

Use `/csharp-build` when:
- `dotnet build` fails with CS-prefixed errors
- NuGet restore fails or reports version conflicts
- Nullable reference type warnings are treated as errors
- Source generator or Roslyn analyzer errors appear
- After pulling changes that break the build

## Diagnostic Commands Run

```bash
# Primary build check
dotnet build 2>&1

# Restore and dependency checks
dotnet restore 2>&1
dotnet list package --vulnerable
dotnet list package --include-transitive 2>&1 | grep -i "conflict\|downgrade"

# Formatting
dotnet format --verify-no-changes

# SDK / target framework
dotnet --list-sdks
cat global.json
```

## Example Session

```text
User: /csharp-build

Agent:
# C# Build Resolution

## Initial Diagnostics

```bash
$ dotnet build
src/Api/Controllers/OrdersController.cs(23,17): error CS0246: type or namespace 'OrderDto' could not be found
src/Infrastructure/Data/AppDbContext.cs(41,16): error CS8603: possible null reference return
```

Errors found: 2

## Fix 1: Missing Type/Namespace

File: src/Api/Controllers/OrdersController.cs:23
Error: CS0246 — type or namespace 'OrderDto' could not be found
Fix: Added `using MyApp.Application.DTOs;`

```bash
$ dotnet build
# 1 error remaining
```

## Fix 2: Nullable Reference Return

File: src/Infrastructure/Data/AppDbContext.cs:41
Error: CS8603 — possible null reference return
Fix: Added null guard
return user ?? throw new InvalidOperationException($"User {id} not found.");

```bash
$ dotnet build
# Build succeeded
```

## Final Verification

```bash
$ dotnet test --no-build -q
Passed!  - Failed: 0, Passed: 128, Skipped: 0
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 2 |
| Files modified | 2 |
| Remaining issues | 0 |

Build Status: PASS: SUCCESS
```

## Common Errors Fixed

| Error | Typical Fix |
|-------|-------------|
| `CS0246: type or namespace not found` | Add `using` or `<PackageReference>` |
| `CS0103: name does not exist` | Fix typo, scope, or declaration |
| `CS1061: no definition for 'Y'` | Add `using` for extension method or fix type |
| `CS0029: cannot convert type` | Explicit cast or fix type at source |
| `CS0161: not all code paths return` | Add return/throw in missing branch |
| `CS8600/CS8602/CS8603` | Null check, `!` with justification, or fix flow |
| `NU1605: package downgrade` | Add explicit `<PackageReference>` at required version |
| `NU1101: unable to find package` | Add NuGet source or fix package name |
| `NETSDK1045: SDK does not support` | Update `global.json` or install correct SDK |

## Fix Strategy

1. **Build errors first** — code must compile
2. **Restore/dependency conflicts second** — resolve NuGet versions
3. **Nullable and analyzer warnings third** — fix flow, don't suppress
4. **One fix at a time** — verify each change
5. **Minimal changes** — don't refactor, just fix

## Stop Conditions

The agent will stop and report if:
- Same error persists after 3 attempts
- Fix introduces more errors than it resolves
- Requires architectural changes beyond build resolution
- Missing packages from private feeds that need credentials or a decision

## Related Commands

- `/csharp-test` — Run tests after build succeeds
- `/csharp-review` — Review code quality
- `verification-loop` skill — Full verification loop

## Related

- Agent: `agents/csharp-build-resolver.md`
- Skill: `skills/dotnet-patterns/`
- Rules: `rules/csharp/`
