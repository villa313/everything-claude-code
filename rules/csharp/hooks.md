---
paths:
  - "**/*.cs"
  - "**/*.csx"
  - "**/*.csproj"
  - "**/*.sln"
  - "**/Directory.Build.props"
  - "**/Directory.Build.targets"
---
# C# Hooks

> This file extends [common/hooks.md](../common/hooks.md) with C#-specific content.

## PostToolUse Hooks

Configure in `~/.claude/settings.json`:

- **dotnet format**: Auto-format edited C# files and apply analyzer fixes
- **dotnet build**: Verify the solution or project still compiles after edits
- **dotnet test --no-build**: Re-run the nearest relevant test project after behavior changes
- **Analyzer warnings**: Run Roslyn analyzers after edits
- **Console.WriteLine warning**: Warn about `Console.WriteLine` in production code

## Stop Hooks

- Run a final `dotnet build` before ending a session with broad C# changes
- Warn on modified `appsettings*.json` files so secrets do not get committed
- **Coverage check**: Verify test coverage meets 80% threshold before session ends
- **Console output audit**: Check all modified files for `Console.WriteLine` or `Debug.WriteLine` statements
- **Nullable reference check**: Ensure nullable reference types are properly annotated

## Example Hook Configuration

```json
{
  "hooks": {
    "postToolUse": [
      {
        "matcher": {
          "toolName": "Edit",
          "filePattern": "**/*.cs"
        },
        "command": "dotnet format {{file}} --verify-no-changes"
      },
      {
        "matcher": {
          "toolName": "Edit",
          "filePattern": "**/*.cs"
        },
        "command": "dotnet build --no-restore /p:TreatWarningsAsErrors=true"
      }
    ]
  }
}
```
