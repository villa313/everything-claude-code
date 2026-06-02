---
paths:
  - "**/*.cs"
  - "**/*.csx"
  - "**/*.csproj"
  - "**/appsettings*.json"
---
# C# Security

> This file extends [common/security.md](../common/security.md) with C#-specific content.

## Secret Management

- Never hardcode API keys, tokens, or connection strings in source code
- Use environment variables, user secrets for local development, and a secret manager in production
- Keep `appsettings.*.json` free of real credentials

```bash
# Initialize user secrets (development)
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Database" "Server=localhost;..."
```

```csharp
// BAD
const string ApiKey = "sk-live-123";

// GOOD
var apiKey = builder.Configuration["OpenAI:ApiKey"]
    ?? throw new InvalidOperationException("OpenAI:ApiKey is not configured.");

// Production: Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{keyVaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```

## Input Validation

- Validate DTOs at the application boundary
- Use data annotations, FluentValidation, or explicit guard clauses
- Reject invalid model state before running business logic

```csharp
public class UserRegistrationDto
{
    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Required]
    [StringLength(100, MinimumLength = 8)]
    public string Password { get; set; } = string.Empty;
}

[HttpPost]
public async Task<IActionResult> Register([FromBody] UserRegistrationDto dto)
{
    if (!ModelState.IsValid) return BadRequest(ModelState);
    // Process registration
}
```

## SQL Injection Prevention

- Always use parameterized queries with ADO.NET, Dapper, or EF Core
- Never concatenate user input into SQL strings
- Validate sort fields and filter operators before using dynamic query composition

```csharp
// CORRECT: Dapper parameterized query
const string sql = "SELECT * FROM Orders WHERE CustomerId = @customerId";
await connection.QueryAsync<Order>(sql, new { customerId });

// CORRECT: LINQ (automatically parameterized)
var user = await _context.Users.FirstOrDefaultAsync(u => u.Email == email);

// WRONG: String concatenation (SQL injection risk!)
var query = $"SELECT * FROM Users WHERE Email = '{email}'";  // NEVER DO THIS
```

## Authentication and Authorization

- Prefer framework auth handlers instead of custom token parsing
- Enforce authorization policies at endpoint or handler boundaries
- Never log raw tokens, passwords, or PII

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"] ?? ""))
        };
    });

[Authorize(Roles = "Admin")]
[HttpDelete("{id}")]
public async Task<IActionResult> Delete(string id) { /* ... */ }
```

## Password Hashing

Use Identity's password hasher or a proven library — never roll your own:

```csharp
public string HashPassword(User user, string password) =>
    _passwordHasher.HashPassword(user, password);

public bool VerifyPassword(User user, string password) =>
    _passwordHasher.VerifyHashedPassword(user, user.PasswordHash, password)
        == PasswordVerificationResult.Success;
```

## Error Handling

- Return safe client-facing messages
- Log detailed exceptions with structured context server-side
- Do not expose stack traces, SQL text, or filesystem paths in API responses

## HTTPS Enforcement

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
    app.UseHttpsRedirection();
}
```

## Security Scanning

```bash
# Check for vulnerable dependencies
dotnet list package --vulnerable

dotnet tool install --global security-scan
dotnet security-scan --solution MySolution.sln
```

## References

See skill: `security-review` for broader application security review checklists.
