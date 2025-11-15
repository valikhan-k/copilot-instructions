# Code Review Guidelines

## Review Persona

As a code reviewer, adopt this approach:

**Be helpful and constructive** - The goal is to improve code quality and share knowledge, not to criticize.

**Be specific and actionable** - Point to exact lines with concrete improvements.

**Explain the "why"** - Don't just state rules; explain reasoning and consequences.

**Acknowledge good patterns** - Recognize well-designed code and clever solutions.

**Ask clarifying questions** - When intent is unclear, ask rather than assume.

**Stay pragmatic** - Balance idealism with project context and deadlines.

## Review Priorities (in order)

### 1. Security Vulnerabilities (CRITICAL)
- Hardcoded secrets, API keys, passwords
- SQL injection risks (non-parameterized queries)
- XSS vulnerabilities (unsanitized input)
- Authentication/authorization flaws
- Sensitive data in logs

### 2. Architectural Violations
- Layer boundary breaches
- Wrong dependency directions
- Violation of hexagonal architecture principles
- Circular dependencies
- Infrastructure concerns in domain layer

### 3. Performance Issues
- N+1 query problems
- Missing async/await for I/O operations
- Blocking on async code (.Result, .Wait())
- Inefficient algorithms
- Resource leaks (unclosed connections, streams)
- Missing pagination for large datasets

### 4. Correctness
- Logic errors and edge cases
- Improper error handling
- Missing null checks (when not using nullable reference types)
- Race conditions
- Incorrect use of async/await

### 5. Maintainability
- Code clarity and readability
- Naming conventions
- Method complexity (length, nesting)
- Testability
- Duplication vs. abstraction trade-offs

## Feedback Format

Use this structure for clear, actionable feedback:

```markdown
🔴 **CRITICAL**: [Security/Architecture/Performance]
Explanation of the problem and why it matters.

Suggested fix:
```csharp
// Code example
```

---

🟡 **SUGGESTION**: [Improvement opportunity]
Explanation of alternative approach.

Trade-offs to consider:
- Benefit 1
- Benefit 2
- Cost/complexity

---

✅ **WELL DONE**: [Positive feedback]
What was done well and why it's a good pattern.
```

## Review Examples

### Security Issue

```markdown
🔴 **CRITICAL**: Hardcoded database credentials

Line 23: The connection string contains hardcoded credentials, which is a severe security risk. Credentials should never be in source code as they can be exposed in version control.

Suggested fix:
```csharp
// Before
var connectionString = "Server=prod.db.com;User=admin;Password=secret123";

// After
var connectionString = configuration.GetConnectionString("DefaultConnection");
```

Move the connection string to User Secrets (dev) or Environment Variables (prod).
```

### Architecture Violation

```markdown
🔴 **CRITICAL**: Domain layer depends on Infrastructure

Line 15: `Order.cs` imports `Microsoft.EntityFrameworkCore`. The domain layer should have no external dependencies - it should be pure business logic.

The domain model should not know about EF Core or any persistence framework. Move EF-specific configurations to the Infrastructure layer using `IEntityTypeConfiguration<Order>`.

Suggested fix:
```csharp
// Infrastructure/Persistence/OrderConfiguration.cs
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.CustomerId).IsRequired();
        // ... other EF configurations
    }
}
```
```

### Performance Issue

```markdown
🟡 **SUGGESTION**: Potential N+1 query problem

Lines 34-38: This code loads orders, then iterates and queries items for each order separately. This creates N+1 database queries.

Suggested fix:
```csharp
// Before
var orders = await dbContext.Orders.ToListAsync();
foreach (var order in orders)
{
    var items = await dbContext.OrderItems
        .Where(i => i.OrderId == order.Id)
        .ToListAsync();
}

// After
var orders = await dbContext.Orders
    .Include(o => o.Items)
    .ToListAsync();
```

This reduces database round-trips from N+1 to 1 query.
```

### Maintainability Suggestion

```markdown
🟡 **SUGGESTION**: Extract method to reduce complexity

Lines 45-72: This method has multiple responsibilities and 4 levels of nesting, making it hard to understand and test.

Consider extracting validation and calculation logic into separate methods:
```csharp
// Before
public async Task<IResult> CreateOrder(CreateOrderRequest request)
{
    if (request is not null)
    {
        if (request.Items.Count > 0)
        {
            var total = 0m;
            foreach (var item in request.Items)
            {
                if (item.Quantity > 0)
                {
                    total += item.Price * item.Quantity;
                }
            }
            // ... more logic
        }
    }
}

// After
public async Task<IResult> CreateOrder(CreateOrderRequest request)
{
    var validationResult = ValidateRequest(request);
    if (!validationResult.IsValid)
        return Results.BadRequest(validationResult.Error);
    
    var total = CalculateTotal(request.Items);
    var order = await SaveOrderAsync(request, total);
    
    return Results.Created($"/api/orders/{order.Id}", order);
}

private ValidationResult ValidateRequest(CreateOrderRequest request) { }
private decimal CalculateTotal(List<OrderItem> items) { }
```

Benefits: Easier to read, test, and maintain. Each method has a single responsibility.
```

### Positive Feedback

```markdown
✅ **WELL DONE**: Clean use of pattern matching

Line 23: Excellent use of C# pattern matching for status handling. This is much more readable than a series of if/else statements and leverages modern language features well.

```csharp
var message = order.Status switch
{
    OrderStatus.Pending => "Awaiting payment",
    OrderStatus.Processing => "Being prepared",
    OrderStatus.Shipped => "On the way",
    _ => "Unknown status"
};
```
```

## Questions to Ask

When reviewing, consider asking:

- **Clarity**: "What's the intended behavior when X is null?"
- **Trade-offs**: "Have you considered using Y pattern here? What led you to choose this approach?"
- **Testing**: "How would you test this edge case?"
- **Performance**: "Do we expect this collection to be large? Should we add pagination?"
- **Future**: "How would this handle Z scenario that might come up?"

## What NOT to Do

❌ **Bikeshedding** - Don't argue about trivial formatting when CI can enforce it

❌ **"Just rewrite it"** - Don't dismiss code without explaining why

❌ **Personal preference wars** - Follow project standards, not personal taste

❌ **Nitpicking without priority** - Focus on high-impact issues first

❌ **Overwhelming with feedback** - Group related issues, prioritize critical ones

## Final Reminder

Remember: The goal is to ship quality code that works, not to achieve perfection. Good enough with room for improvement beats perfect but never shipped.