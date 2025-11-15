# Code Quality Standards

## Naming Conventions

**Casing:**
- `PascalCase`: Types, methods, properties, constants
- `camelCase`: Parameters, local variables, private fields
- Prefix interfaces with `I` (e.g., `IOrderService`)

**Descriptive names:**
- Use intention-revealing names
- Avoid abbreviations unless universally understood
- Be specific, not generic

```csharp
// Good
public async Task<Order> GetOrderByIdAsync(OrderId orderId)
public record CreateOrderRequest(string CustomerId, List<OrderItem> Items)

// Avoid
public async Task<Order> GetAsync(string id)  // Too generic
public record CrtOrdReq(string cid, List<OrderItem> itms)  // Abbreviations
```

## C# Language Features

**Use modern C# features:**
- Records for DTOs and immutable data
- Pattern matching
- Collection expressions
- Nullable reference types
- `is null` and `is not null` over `== null`

```csharp
// Good: Modern C#
public record OrderCreatedEvent(OrderId OrderId, DateTime CreatedAt);

var items = order.Status switch
{
    OrderStatus.Pending => "Awaiting payment",
    OrderStatus.Processing => "Being prepared",
    OrderStatus.Shipped => "On the way",
    _ => "Unknown status"
};

if (order is not null && order.CanBeCancelled())
{
    await orderService.CancelAsync(order.Id);
}

// Collection expressions
List<string> tags = ["urgent", "priority"];
```

## Method Design

**Keep methods focused:**
- Under 50 lines as guideline
- Single Responsibility Principle
- One reason to change

**Reduce nesting:**
- Maximum 3 levels of nesting
- Use early returns
- Extract to helper methods

```csharp
// Good: Early return pattern
public bool CanProcessOrder(Order order)
{
    if (order is null)
        return false;
    
    if (order.Status is not OrderStatus.Pending)
        return false;
    
    if (order.Items.Count == 0)
        return false;
    
    return order.CalculateTotal() > 0;
}

// Avoid: Deep nesting
public bool CanProcessOrder(Order order)
{
    if (order is not null)
    {
        if (order.Status is OrderStatus.Pending)
        {
            if (order.Items.Count > 0)
            {
                return order.CalculateTotal() > 0;
            }
        }
    }
    return false;
}
```

**Prefer pure functions:**
- No side effects when possible
- Easier to test and reason about
- Predictable behavior

## Async/Await Patterns

**Use async for I/O-bound operations:**
- Database queries
- HTTP calls
- File operations
- Message queue operations

```csharp
// Good
public async Task<Order> GetOrderAsync(OrderId id)
{
    return await dbContext.Orders
        .Include(o => o.Items)
        .FirstOrDefaultAsync(o => o.Id == id);
}
```

**Naming convention:**
- Append `Async` suffix to async method names

**Never block on async:**
```csharp
// Bad - can cause deadlocks
var order = orderService.GetOrderAsync(id).Result;
var order = orderService.GetOrderAsync(id).Wait();
var order = orderService.GetOrderAsync(id).GetAwaiter().GetResult();

// Good
var order = await orderService.GetOrderAsync(id);
```

## Cross-Cutting Concerns

### Logging

**Structured logging with appropriate levels:**
```csharp
logger.LogInformation(
    "Order {OrderId} created for customer {CustomerId}", 
    order.Id, 
    order.CustomerId);

logger.LogWarning(
    "Insufficient stock for product {ProductId}: requested {Requested}, available {Available}",
    productId,
    requestedQuantity,
    availableStock);

logger.LogError(
    exception,
    "Failed to process payment for order {OrderId}",
    order.Id);
```

**Do not log sensitive data:**
- Passwords, tokens, API keys
- PII without proper safeguards
- Full connection strings

### Configuration

**Options Pattern:**
```csharp
// appsettings.json
{
  "Database": {
    "ConnectionString": "...",
    "CommandTimeout": 30
  }
}

// Configuration class
public class DatabaseSettings
{
    public string ConnectionString { get; set; } = "";
    public int CommandTimeout { get; set; } = 30;
}

// Registration
builder.Services.Configure<DatabaseSettings>(
    builder.Configuration.GetSection("Database"));

// Usage
public class OrderRepository
{
    private readonly DatabaseSettings _settings;
    
    public OrderRepository(IOptions<DatabaseSettings> settings)
    {
        _settings = settings.Value;
    }
}
```

### Error Handling

**Global exception handler:**
```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionFeature?.Error;
        
        logger.LogError(exception, "Unhandled exception");
        
        var problemDetails = new ProblemDetails
        {
            Title = "An error occurred",
            Status = StatusCodes.Status500InternalServerError,
            Detail = environment.IsDevelopment() 
                ? exception?.Message 
                : "An unexpected error occurred"
        };
        
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(problemDetails);
    });
});
```

**Result pattern for expected failures:**
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
public async Task<Result<Order>> CreateOrderAsync(CreateOrderRequest request)
{
    if (!await HasSufficientStock(request.Items))
        return Result<Order>.Failure("Insufficient stock");
    
    var order = await SaveOrderAsync(request);
    return Result<Order>.Success(order);
}
```

### Health Checks

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddUrlGroup(new Uri("https://api.external.com/health"), "External API");

app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready");
```

## Patterns to Favor

- **Dependency Injection**: For all services
- **Repository Pattern**: For data access abstraction
- **Options Pattern**: For configuration
- **Result Pattern**: For expected failures
- **Mediator Pattern**: Only when CQRS provides clear value

## Patterns to Avoid

- Service Locator anti-pattern
- God classes and fat controllers
- Deep inheritance hierarchies
- Premature performance optimization
- Over-engineering with unnecessary abstractions