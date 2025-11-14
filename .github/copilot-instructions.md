# Project Guidelines & Standards

## Technology Stack & Foundation

### Platform & Language
- **.NET 8/9+** using latest stable SDK
- **C# 12+** with latest language features enabled
- **Minimal APIs** as the default HTTP interface (not MVC Controllers)
- **Nullable reference types** enabled throughout
- Use `is null` and `is not null` over `== null` and `!= null`

### API Design
- **One endpoint per file** — each endpoint in its own `.cs` file within a feature folder
- **Co-locate DTOs** with their endpoint in the same feature folder
- **Keep endpoints thin** — delegate to services for business logic
- Use extension methods to organize endpoint registration by feature
```csharp
/* Feature folder structure

/Features
  /Orders
    CreateOrder.cs
    CreateOrderRequest.cs
    GetOrder.cs
*/

// CreateOrder.cs
public static class CreateOrder
{
    public static void Map(WebApplication app)
    {
        app.MapPost("/api/orders", async (
            CreateOrderRequest request,
            OrderService orderService) =>
        {
            var order = await orderService.CreateAsync(request);
            return Results.Created($"/api/orders/{order.Id}", order);
        });
    }
}

// CreateOrderRequest.cs
public record CreateOrderRequest(string CustomerId, List<OrderItem> Items);

// Program.cs
CreateOrder.Map(app);
GetOrder.Map(app);
```

---

## Architectural Principles

### Hexagonal Architecture
The core idea: **isolate business logic from external concerns** (databases, APIs, UI, infrastructure).

**Layers:**
- **Domain** — Core business entities, rules, and logic (no external dependencies)
- **Application** — Use cases and application services that orchestrate domain logic (depends on Domain only)
- **Infrastructure** — External concerns: databases, HTTP clients, file systems, message queues (depends on Application to implement contracts)
- **API/Presentation** — HTTP endpoints, request/response handling (depends on Application for use cases)

**How it works:**
- Application layer defines **interfaces** for external dependencies it needs (e.g., `IOrderRepository`, `IPaymentGateway`)
- Infrastructure layer provides **concrete implementations** of those interfaces
- Application layer also provides **application services** that the API layer calls to satisfy requests
- Both the contracts (interfaces) and implementations (services) for driving interactions live in Application
- Dependency injection wires everything together at startup

**For deeper understanding of Hexagonal Architecture, see Alistair Cockburn's original article.**

### Critical Boundaries

**Dependency rules (what can reference what):**
- **Domain** → references nothing (pure business logic)
- **Application** → references Domain only
- **Infrastructure** → references Application (to implement its interfaces) and Domain
- **API/Presentation** → references Application (to call use cases) and Infrastructure (for DI registration only)


### Evolutionary Architecture
- **Start small, single project** — Use folder boundaries initially (`/Features`, `/Domain`, `/Infrastructure`)
- **Extract when complexity emerges** — Move to separate projects only when boundaries become clear and benefits justify the overhead
- **Grow organically** — Let actual usage patterns dictate structure, not theoretical purity
- **Pragmatic over dogmatic** — Favor solutions that work over textbook perfection

### Domain Modeling
- **No heavyweight DDD patterns** — Skip Entity, ValueObject, DomainEvent, Aggregate base classes unless explicitly needed
- Use **plain C# records and classes** for domain models
- Apply business rules through methods and validation logic directly on models
- Focus on behavior that matters, not ceremonial patterns

```csharp
// Good: Simple, focused domain model
public record Order(OrderId Id, CustomerId CustomerId, List<OrderLine> Lines, OrderStatus Status)
{
    public decimal CalculateTotal() => Lines.Sum(l => l.Quantity * l.UnitPrice);
    public bool CanBeCancelled() => Status is OrderStatus.Pending;
}

// Avoid: Unnecessary base classes and ceremony
public class Order : Entity, IAggregateRoot { /* ... */ }
```

### Folder Organization
- **Feature/domain-oriented** top-level structure
- Group by business capability, not technical layer

```
/Features
  /Orders
    CreateOrder.cs
    GetOrder.cs
    CancelOrder.cs
    OrderService.cs
  /Customers
    RegisterCustomer.cs
    CustomerService.cs
/Domain
  Order.cs
  Customer.cs
/Infrastructure
  /Persistence
    OrderRepository.cs
  /ExternalServices
    PaymentGatewayAdapter.cs
```

---

## Security Standards (Non-Negotiable)

### Secrets & Credentials
- **Never hardcode secrets, API keys, passwords, or connection strings** in code
- Use `IConfiguration` with User Secrets (dev), Environment Variables (prod), or Azure Key Vault
- Validate all secrets are loaded at startup; fail fast if missing critical configuration

### Input Validation
- **Validate all external input** (HTTP requests, message payloads, file uploads)
- Use FluentValidation or DataAnnotations for DTOs
- Sanitize input to prevent XSS, SQL injection, command injection
- Enforce size limits on uploads and payloads

### Data Access
- **Use parameterized queries** — Never concatenate user input into SQL
- Prefer ORMs (EF Core) or micro-ORMs (Dapper) with proper parameter binding
- Apply principle of least privilege for database credentials

### Authentication & Authorization
- Implement authentication at the API gateway/middleware level
- Use standard mechanisms: JWT, OAuth 2.0, OpenID Connect
- Apply authorization checks before executing business logic
- Never trust client-provided role/permission claims without validation

```csharp
// Good: Parameterized query
var orders = await dbContext.Orders
    .Where(o => o.CustomerId == customerId)
    .ToListAsync();

// Bad: SQL injection risk
var query = $"SELECT * FROM Orders WHERE CustomerId = '{customerId}'";
```

---

## Cross-Cutting Concerns

### Logging
- **Structured logging** with Serilog or Microsoft.Extensions.Logging
- Log at appropriate levels: Trace, Debug, Information, Warning, Error, Critical
- Include correlation IDs for request tracing
- **Do not log sensitive data** (passwords, tokens, PII)
- Log exceptions with full stack traces

```csharp
logger.LogInformation("Order {OrderId} created for customer {CustomerId}", order.Id, order.CustomerId);
```

### Configuration
- Read configuration via **IConfiguration** injected through DI
- Use Options Pattern (`IOptions<T>`, `IOptionsSnapshot<T>`) for strongly-typed configuration
- Validate configuration at startup using `IValidateOptions<T>`

```csharp
builder.Services.Configure<DatabaseSettings>(builder.Configuration.GetSection("Database"));
builder.Services.AddSingleton<IValidateOptions<DatabaseSettings>, DatabaseSettingsValidator>();
```

### Health Checks
- Implement health check endpoints (`/health`, `/health/ready`)
- Check critical dependencies: database connectivity, external API availability
- Return appropriate HTTP status codes (200, 503)
- Include health checks in deployment pipelines

### Error Handling
- Use global exception handling middleware
- Return RFC 7807 Problem Details for API errors
- Differentiate between client errors (4xx) and server errors (5xx)
- Never expose internal error details (stack traces) to clients in production

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionFeature?.Error;
        
        logger.LogError(exception, "Unhandled exception occurred");
        
        context.Response.StatusCode = 500;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Title = "An error occurred",
            Status = 500
        });
    });
});
```

---

## Code Quality Standards

### Naming & Style
- **PascalCase** for types, methods, properties, constants
- **camelCase** for parameters, local variables, private fields
- Prefix interfaces with `I` (e.g., `IOrderService`)
- Use descriptive, intention-revealing names
- Avoid abbreviations unless universally understood

### Method Design
- **Keep methods focused** (under 50 lines)
- Single Responsibility Principle — one reason to change
- Prefer pure functions where possible (no side effects)
- Avoid deep nesting (max 3 levels); extract to helper methods

### Async/Await
- Use `async/await` for I/O-bound operations
- Append `Async` suffix to async method names
- **Never block on async code** — avoid `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`
- Use `ConfigureAwait(false)` in library code (not needed in ASP.NET Core)

### Testing Standards

- **Test behavior, not implementation** — Focus on observable outcomes through public APIs
- **Outside-in approach** — Start testing from entry points (endpoints, handlers) and work inward
- **Favor integration tests** — Test through real collaborators when practical (use TestContainers for databases)
- **Mock only external dependencies** — Mock out-of-process dependencies (APIs, message queues); use real in-process collaborators
- Write tests for **business-critical logic, complex algorithms, and integration points**
- Use **Arrange-Act-Assert (AAA)** pattern with clear visual separation
- **One logical assertion per test** — Multiple Assert calls are fine if testing the same concept
- **Test naming** — Write in plain English as if describing to a non-programmer familiar with the domain; separate words with underscores
- Keep tests maintainable and readable — favor clarity over DRY principles
```csharp
// Good: Tests observable behavior with plain English name
[Fact]
public async Task Creating_an_order_with_insufficient_stock_returns_an_error()
{
    // Arrange
    await SeedProduct(productId: "p1", stockQuantity: 5);
    var request = new CreateOrderRequest("customer-1", [new OrderItem("p1", Quantity: 10)]);
    
    // Act
    var response = await Client.PostAsJsonAsync("/api/orders", request);
    
    // Assert
    response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    var problem = await response.Content.ReadFromJsonAsync();
    problem.Detail.Should().Contain("insufficient stock");
}

// Avoid: Tests implementation details
[Fact]
public void CreateOrder_CallsRepositorySaveExactlyOnce()
{
    mockRepository.Verify(r => r.SaveAsync(It.IsAny()), Times.Once);
}
```

Consult [Testing Instructions](./instructions/testing.instructions.md) for comprehensive testing guidelines.

### Abstraction & Duplication
- **Avoid premature abstraction** — duplication is better than wrong abstraction
- Extract abstractions only when patterns emerge across 3+ locations
- Keep code "wet" (Write Everything Twice) before drying it out
- Prefer composition over inheritance
- Challenge every base class: does it truly provide value?

---

## Code Review Persona

As a code reviewer, adopt this approach:

### Review Priorities (in order)
1. **Security vulnerabilities** — Hardcoded secrets, injection risks, authentication flaws
2. **Architectural violations** — Layer boundary breaches, wrong dependencies
3. **Performance issues** — N+1 queries, inefficient algorithms, resource leaks
4. **Correctness** — Logic errors, edge cases, error handling gaps
5. **Maintainability** — Clarity, naming, complexity, testability

### Review Style
- **Be specific and actionable** — Point to exact lines and suggest concrete improvements
- **Explain the "why"** — Don't just state rules; explain reasoning and consequences
- **Acknowledge good patterns** — Recognize well-designed code, clever solutions, clear logic
- **Ask clarifying questions** — When intent is unclear, ask rather than assume
- **Stay pragmatic** — Balance idealism with project context and deadlines

### Feedback Format
```markdown
🔴 **CRITICAL**: [Security/Architecture/Performance issue]
Explanation of the problem and why it matters.
Suggested fix with code example.

🟡 **SUGGESTION**: [Improvement opportunity]
Explanation of alternative approach.
Trade-offs to consider.

✅ **WELL DONE**: [Positive feedback]
What was done well and why it's a good pattern.
```

---

## Technologies & Patterns

### Preferred Libraries
- **Web Framework**: ASP.NET Core Minimal APIs
- **ORM**: Entity Framework Core or Dapper
- **Validation**: FluentValidation
- **Logging**: Serilog with structured logging
- **Testing**: xUnit, NSubstitute/Moq, FluentAssertions, Testcontainers
- **Mapping**: Mapster or manual mapping (avoid AutoMapper complexity)

### Patterns to Favor
- Result pattern over exceptions for expected failures
- Dependency Injection for all services
- Repository pattern for data access abstraction
- Options pattern for configuration
- Mediator pattern (MediatR) only when command/query segregation provides clear value

### Patterns to Avoid
- Service Locator anti-pattern
- God classes and fat controllers
- Anemic domain models (if using domain-driven approach)
- Premature performance optimization
- Deep inheritance hierarchies

---

## Development Workflow

### Code Generation
- Generate code that matches existing patterns in the codebase
- Follow the folder structure and naming conventions already established
- Use latest C# features (records, pattern matching, collection expressions)
- Include XML documentation for public APIs

### Incremental Changes
- Make small, focused commits
- Test each change before moving to the next
- Refactor in separate commits from feature work
- Keep PRs reviewable (under 400 lines when possible)

### Documentation
- Document **why**, not **what** — code should be self-documenting for the "what"
- Add XML doc comments for public APIs
- Update README.md when adding new features or changing architecture
- Keep architecture decision records (ADRs) for significant choices

---

## Final Reminders

- **Empirical cleanliness over theoretical purity** — Does it actually improve the codebase?
- **Pragmatism over dogma** — Rules serve the team, not the other way around
- **Start simple, evolve deliberately** — Add complexity only when justified by actual needs
- **Question abstractions** — Every layer, every base class must earn its place
- **Focus on readability** — Code is read 10x more than it's written

When in doubt, prefer the simpler solution that solves the current problem over the more "architecturally correct" solution that anticipates future problems that may never materialize.

