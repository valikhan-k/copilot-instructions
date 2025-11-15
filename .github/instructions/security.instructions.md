# Security Standards (Non-Negotiable)

## Secrets & Credentials

**Never hardcode secrets** - API keys, passwords, connection strings

```csharp
// Bad
var connectionString = "Server=prod.db.com;User=admin;Password=secret123";

// Good
var connectionString = configuration.GetConnectionString("DefaultConnection");
```

**Configuration sources:**
- Development: User Secrets
- Production: Environment Variables or Azure Key Vault
- Validate at startup; fail fast if critical config missing

```csharp
var apiKey = configuration["ExternalApi:ApiKey"] 
    ?? throw new InvalidOperationException("API key not configured");
```

## Input Validation

**Validate all external input** - HTTP requests, message payloads, file uploads

```csharp
public record CreateOrderRequest(string CustomerId, List<OrderItem> Items);

public class CreateOrderValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty();
        RuleForEach(x => x.Items).SetValidator(new OrderItemValidator());
    }
}
```

**Sanitization:**
- Prevent XSS, SQL injection, command injection
- Enforce size limits on uploads and payloads
- Validate file types and content

## Data Access

**Use parameterized queries** - Never concatenate user input into SQL

```csharp
// Good: Parameterized with EF Core
var orders = await dbContext.Orders
    .Where(o => o.CustomerId == customerId)
    .ToListAsync();

// Good: Parameterized with Dapper
var orders = await connection.QueryAsync<Order>(
    "SELECT * FROM Orders WHERE CustomerId = @CustomerId",
    new { CustomerId = customerId });

// Bad: SQL injection risk
var query = $"SELECT * FROM Orders WHERE CustomerId = '{customerId}'";
var orders = await connection.QueryAsync<Order>(query);
```

**Database credentials:**
- Apply principle of least privilege
- Use separate credentials for different environments
- Rotate credentials regularly

## Authentication & Authorization

**Authentication at middleware level**
- Use standard mechanisms: JWT, OAuth 2.0, OpenID Connect
- Validate tokens properly

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { /* ... */ });

app.UseAuthentication();
app.UseAuthorization();
```

**Authorization before business logic**
- Apply authorization checks early
- Never trust client-provided role/permission claims
- Validate claims server-side

```csharp
app.MapPost("/api/orders", async (
    CreateOrderRequest request,
    ClaimsPrincipal user,
    OrderService orderService) =>
{
    var customerId = user.FindFirst("customer_id")?.Value;
    if (customerId is null)
        return Results.Unauthorized();
    
    var order = await orderService.CreateAsync(request, customerId);
    return Results.Created($"/api/orders/{order.Id}", order);
})
.RequireAuthorization();
```

## Sensitive Data Handling

**Do not log sensitive data:**
- Passwords, tokens, API keys
- Personal Identifiable Information (PII)
- Credit card numbers, SSNs
- Full connection strings

```csharp
// Bad
logger.LogInformation("User logged in with password {Password}", password);

// Good
logger.LogInformation("User {UserId} logged in successfully", userId);
```

**Secure data in transit and at rest:**
- Use HTTPS/TLS for all communication
- Encrypt sensitive data in database
- Use secure cookie flags (HttpOnly, Secure, SameSite)

## Common Vulnerabilities to Prevent

- **SQL Injection**: Always use parameterized queries
- **XSS**: Sanitize and encode user input
- **CSRF**: Use anti-forgery tokens for state-changing operations
- **Path Traversal**: Validate and sanitize file paths
- **Mass Assignment**: Use DTOs, never bind directly to domain models
- **Insecure Deserialization**: Validate input before deserializing

## Security Checklist

Before merging code, verify:
- [ ] No hardcoded secrets or credentials
- [ ] All external input validated
- [ ] Parameterized queries used
- [ ] Authentication/authorization implemented
- [ ] Sensitive data not logged
- [ ] HTTPS enforced
- [ ] Security headers configured
- [ ] Error messages don't leak internal details