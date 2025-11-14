---
applyTo: "**/*Tests.cs,**/*Test.cs,**/Tests/**/*.cs"
---

# Testing Standards & Guidelines

## Purpose
Comprehensive testing guidance for writing maintainable, valuable tests that focus on behavior over implementation. These standards apply to all test code in the repository.

---

## Testing Philosophy

### Core Principles
- **Test behavior, not implementation** — Focus on observable outcomes through public APIs, not internal mechanics
- **Outside-in approach** — Start from high-level integration tests (API endpoints) with broad, behavioral assertions; use strict assertions only in focused unit tests for complex logic
- **Mock sparingly** — Mock only out-of-process dependencies (external APIs, message queues, file systems); use real in-process collaborators
- **Resist the refactoring anti-pattern** — Good tests survive refactoring; if tests break when you rename methods or restructure code without changing behavior, they're coupled to implementation details

---

## Test Stack

### Testing Libraries
- **xUnit** — Test framework for organizing and running tests
- **FluentAssertions** — Readable, expressive assertions
- **NSubstitute** — Mocking framework for external dependencies
- **WebApplicationFactory<T>** — In-memory API testing without network overhead
- **Testcontainers** — Dockerized dependencies (databases, message brokers) for true integration tests

### When to Use Each Tool
- **xUnit** — All tests (facts, theories, class/collection fixtures)
- **FluentAssertions** — All assertions (`result.Should().Be(expected)`)
- **NSubstitute** — Mock external HTTP APIs, message buses, file systems, third-party services
- **WebApplicationFactory** — API integration tests hitting real endpoints
- **Testcontainers** — When you need real database behavior (transactions, constraints, complex queries)

---

## What to Test

### High-Value Test Targets
- **Business-critical logic** — Core domain rules, calculations, workflows that directly impact business outcomes
- **API contracts** — Request/response behavior through the entire stack (endpoint → service → repository → database)
- **Integration points** — Interactions with databases, external services, message buses
- **Edge cases and error paths** — Boundary conditions, validation failures, exception scenarios, unhappy paths
- **Complex algorithms** — Non-trivial calculations, data transformations, state machines

### What NOT to Test
- **Private methods** — Test through public API instead; if a private method seems to need its own test, it might belong in its own class
- **Trivial code** — Simple getters/setters, auto-properties, DTOs without logic, constructor assignments
- **Framework code** — Don't test EF Core's query translation, ASP.NET Core's routing, or third-party library internals
- **Implementation details** — Internal state, private fields, exact method call counts (unless verifying critical side effects like audit logs)
- **Configuration wiring** — DI container registrations (trust the framework; catch issues in integration tests)

---

## Test Naming Convention

Write test names in **plain English as if describing the scenario to a non-programmer** familiar with the problem domain.

**Rules:**
- Use **underscores to separate words** for readability
- Write as **complete sentences** that describe the behavior being tested
- **No rigid formulas** like `MethodName_Scenario_ExpectedResult` — they encourage focusing on implementation
- **Avoid "should"** — Tests state facts, not wishes (use "is" instead of "should be")
- **Include articles** (a, an, the) for natural reading
- **Don't reference method names** unless testing utility code with no business meaning

```csharp
// Good: Plain English describing business behavior
[Fact]
public async Task Delivery_with_a_past_date_is_invalid()

[Fact]
public async Task Creating_an_order_with_insufficient_stock_returns_an_error()

[Fact]
public async Task Customer_cannot_cancel_an_order_after_it_has_shipped()

// Avoid: Rigid formula coupled to implementation
[Fact]
public async Task CreateOrder_InsufficientStock_ReturnsError()

[Fact]
public async Task IsDeliveryValid_PastDate_ReturnsFalse()
```

---

## Test Structure

### Arrange-Act-Assert (AAA) Pattern
Every test should follow this three-phase structure with **clear visual separation**:

```csharp
[Fact]
public async Task Order_total_includes_tax_and_shipping_costs()
{
    // Arrange
    var order = new Order(
        id: OrderId.Create(),
        customerId: CustomerId.Create("customer-1"),
        lines: [
            new OrderLine(ProductId.Create("p1"), quantity: 2, unitPrice: 10.00m),
            new OrderLine(ProductId.Create("p2"), quantity: 1, unitPrice: 15.00m)
        ],
        shippingCost: 5.00m,
        taxRate: 0.10m
    );
    
    // Act
    var total = order.CalculateTotal();
    
    // Assert
    total.Should().Be(43.50m); // (20 + 15) * 1.10 + 5
}
```

### Assertion Guidelines
- **One logical assertion per test** — A test may have multiple `Assert`/`Should()` calls if they verify the same logical concept
- **Use FluentAssertions** for readability: `result.Should().Be(expected)`, `list.Should().HaveCount(3)`
- **Add assertion messages** for complex conditions: `value.Should().BeGreaterThan(0, "because orders must have positive totals")`
- **Assert on meaningful values** — Don't just check for non-null; verify actual business outcomes

---

## Database Testing

### Option 1: In-Memory Database (Lightweight, Start Here)
Use EF Core's in-memory provider for simple scenarios. **Limitations:** doesn't enforce constraints, relationships, or complex SQL behavior.

**Best for:** Quick iteration, simple CRUD operations, early development.

```csharp
public class OrderRepositoryTests
{
    private readonly AppDbContext _context;
    
    public OrderRepositoryTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
            
        _context = new AppDbContext(options);
    }
    
    [Fact]
    public async Task Saving_an_order_persists_it_to_the_database()
    {
        // Arrange
        var repository = new OrderRepository(_context);
        var order = new Order(/* ... */);
        
        // Act
        await repository.SaveAsync(order);
        
        // Assert
        var saved = await _context.Orders.FindAsync(order.Id);
        saved.Should().NotBeNull();
    }
}
```

### Option 2: SQLite In-Memory (Better SQL Fidelity)
Use SQLite in-memory mode for closer-to-real database behavior without Docker.

**Best for:** Testing constraints, transactions, more realistic SQL without infrastructure overhead.

```csharp
public class OrderRepositoryTests : IDisposable
{
    private readonly DbConnection _connection;
    private readonly AppDbContext _context;
    
    public OrderRepositoryTests()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();
        
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(_connection)
            .Options;
            
        _context = new AppDbContext(options);
        _context.Database.EnsureCreated();
    }
    
    public void Dispose()
    {
        _context.Dispose();
        _connection.Dispose();
    }
}
```

### Option 3: Testcontainers (Production-Like Database)
Use Docker containers for true database behavior. **Requires Docker Desktop or Docker Engine running.**

**Best for:** Testing database-specific features (stored procedures, triggers, complex queries, production migrations).

```csharp
public class OrderApiTests : IClassFixture<WebApplicationFactory<Program>>, IAsyncLifetime
{
    private readonly HttpClient _client;
    private readonly PostgreSqlContainer _dbContainer;
    private readonly WebApplicationFactory<Program> _factory;
    
    public OrderApiTests(WebApplicationFactory<Program> factory)
    {
        _dbContainer = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .Build();
            
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureTestServices(services =>
            {
                services.RemoveAll<DbContextOptions<AppDbContext>>();
                services.AddDbContext<AppDbContext>(options =>
                    options.UseNpgsql(_dbContainer.GetConnectionString()));
            });
        });
        
        _client = _factory.CreateClient();
    }
    
    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();
        
        // Run migrations
        using var scope = _factory.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await context.Database.MigrateAsync();
    }
    
    public async Task DisposeAsync()
    {
        await _dbContainer.DisposeAsync();
    }
}
```

### Database Testing Strategy
1. **Start with in-memory** — Fast feedback during early development
2. **Move to SQLite** — When you need constraints, transactions, or realistic SQL
3. **Upgrade to Testcontainers** — When testing production database features or running CI/CD pipelines

---

## Unit Testing Domain Logic

### Test Pure Domain Logic
Focus unit tests on complex business rules, calculations, and algorithms.

```csharp
[Fact]
public void Order_cannot_be_cancelled_after_it_has_shipped()
{
    // Arrange
    var order = new Order(/* ... */, status: OrderStatus.Shipped);
    
    // Act
    var result = order.Cancel();
    
    // Assert
    result.IsFailure.Should().BeTrue();
    result.Error.Should().Contain("shipped");
}

[Theory]
[InlineData(0, 0)]
[InlineData(1, 10.00)]
[InlineData(5, 50.00)]
public void Order_line_total_is_quantity_times_unit_price(int quantity, decimal expected)
{
    // Arrange
    var line = new OrderLine(ProductId.Create("p1"), quantity, unitPrice: 10.00m);
    
    // Act
    var total = line.CalculateTotal();
    
    // Assert
    total.Should().Be(expected);
}
```

### Mock Only External Dependencies
Use real collaborators for in-process dependencies; mock out-of-process ones.

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task Creating_an_order_calls_payment_gateway_with_correct_amount()
    {
        // Arrange
        var paymentGateway = Substitute.For<IPaymentGateway>();
        var repository = Substitute.For<IOrderRepository>();
        var service = new OrderService(repository, paymentGateway);
        
        var request = new CreateOrderRequest(
            CustomerId: "customer-1",
            Items: [new OrderItem("product-1", Quantity: 2, UnitPrice: 10.00m)]
        );
        
        // Act
        await service.CreateAsync(request);
        
        // Assert
        await paymentGateway.Received(1).ChargeAsync(
            Arg.Is<decimal>(amount => amount == 20.00m),
            Arg.Any<string>()
        );
    }
}
```

### Handcrafted Test Doubles For Reusability
Don't shy away from handcrafted test doubles where makes sense to reduce code duplication and boost reusability.

---

## Test Data Management

Always begin with the simplest solution for test data creation. Only introduce advanced patterns—specifically the Test Data Builder or the Test Fixture—when simple in-line creation becomes ambiguous, repetitive, or complex to manage.

### Test Data Builders
Extract builders for complex object graphs to keep tests readable.

```csharp
public class OrderBuilder
{
    private OrderId _id = OrderId.Create();
    private CustomerId _customerId = CustomerId.Create("default-customer");
    private List<OrderLine> _lines = [];
    private OrderStatus _status = OrderStatus.Pending;
    
    public OrderBuilder WithId(OrderId id)
    {
        _id = id;
        return this;
    }
    
    public OrderBuilder WithCustomer(string customerId)
    {
        _customerId = CustomerId.Create(customerId);
        return this;
    }
    
    public OrderBuilder WithLine(string productId, int quantity, decimal unitPrice)
    {
        _lines.Add(new OrderLine(ProductId.Create(productId), quantity, unitPrice));
        return this;
    }
    
    public OrderBuilder WithStatus(OrderStatus status)
    {
        _status = status;
        return this;
    }
    
    public Order Build() => new(_id, _customerId, _lines, _status);
}

// Usage in tests
var order = new OrderBuilder()
    .WithCustomer("customer-123")
    .WithLine("product-1", quantity: 2, unitPrice: 10.00m)
    .WithLine("product-2", quantity: 1, unitPrice: 15.00m)
    .WithStatus(OrderStatus.Pending)
    .Build();
```

### Test Fixtures
Use custom test fixture classes that encapsulate SUT creation and dependency management. This pattern isolates tests from constructor changes and keeps test setup focused.
```csharp
// Good: Test fixture encapsulates SUT creation and dependencies
public class OrderServiceFixture
{
    public IOrderRepository Repository { get; }
    public IPaymentGateway PaymentGateway { get; }
    
    public OrderServiceFixture()
    {
        Repository = Substitute.For();
        PaymentGateway = Substitute.For();
    }
    
    public OrderService CreateSut()
    {
        return new OrderService(Repository, PaymentGateway);
    }
    
    public Order CreateOrder(
        string? customerId = null,
        OrderStatus status = OrderStatus.Pending)
    {
        return new Order(
            id: OrderId.Create(),
            customerId: CustomerId.Create(customerId ?? "customer-123"),
            lines: [],
            status: status
        );
    }
    
    public CreateOrderRequest CreateRequest(
        string? customerId = null,
        List? items = null)
    {
        return new CreateOrderRequest(
            CustomerId: customerId ?? "customer-123",
            Items: items ?? [new OrderItem("product-1", Quantity: 1)]
        );
    }
}

// Usage in tests
public class OrderServiceTests
{
    [Fact]
    public async Task Creating_an_order_saves_it_to_the_repository()
    {
        // Arrange
        var fixture = new OrderServiceFixture();
        var request = fixture.CreateRequest();
        var sut = fixture.CreateSut();
        
        // Act
        await sut.CreateAsync(request);
        
        // Assert
        await fixture.Repository.Received(1).SaveAsync(Arg.Any());
    }
    
    [Fact]
    public async Task Creating_an_order_for_specific_customer_uses_that_customer()
    {
        // Arrange
        var fixture = new OrderServiceFixture();
        var request = fixture.CreateRequest(customerId: "customer-456");
        var sut = fixture.CreateSut();
        
        // Act
        var result = await sut.CreateAsync(request);
        
        // Assert
        result.CustomerId.Should().Be("customer-456");
    }
}
```

### Why Test Fixtures
- **Encapsulation** — All SUT creation logic lives in one place
- **Flexibility** — Easy to configure specific scenarios via optional parameters
- **Constructor isolation** — Adding constructor parameters only affects the fixture's `CreateSut()` method
- **Dependency management** — Mocks and test doubles are created once and accessible to all tests
- **Sensible defaults** — Provide reasonable default values; override only what matters for each test

---

## Test Maintainability

### Keep Tests Independent
- **No shared state** — Each test must run independently in any order
- **Isolate test data** — Use unique IDs, separate database instances, or proper cleanup
- **Avoid test interdependence** — Tests should never depend on other tests running first

### Favor Clarity Over DRY
- **Some duplication is acceptable** in tests — readability trumps reusability
- **Keep setup visible** — Prefer inline Arrange sections over deeply nested helper methods
- **Extract only when beneficial** — Don't abstract for the sake of abstraction

### Organize Tests by Feature
```
/Tests
  /Features
    /Orders
      CreateOrderTests.cs
      CancelOrderTests.cs
      OrderCalculationTests.cs
    /Customers
      RegisterCustomerTests.cs
```

---

## Test Coverage Guidance

### Coverage as an Output, Not a Goal
- **High coverage ≠ good tests** — You can have 100% coverage with worthless tests
- **Focus on meaningful tests** — Test critical paths, business rules, and edge cases
- **Aim for ~80% coverage of domain logic** — Infrastructure and trivial code can have lower coverage
- **100% is not the target** — Chasing complete coverage leads to testing implementation details

### What Deserves High Coverage
- Domain models with business rules
- Application services orchestrating workflows
- Complex calculations and algorithms
- Validation logic
- Critical integration points

### What Can Have Lower Coverage
- DTOs and simple data classes
- Infrastructure code (repositories, HTTP clients)
- Framework configuration
- Auto-generated code

---

## Testing Anti-Patterns to Avoid

### Testing Implementation Details
```csharp
// Bad: Tests internal behavior
[Fact]
public async Task CreateOrder_Calls_Repository_Save_Exactly_Once()
{
    mockRepository.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);
}

// Good: Tests observable outcome
[Fact]
public async Task Creating_an_order_persists_it_to_the_database()
{
    await service.CreateAsync(request);
    
    var saved = await repository.GetByIdAsync(order.Id);
    saved.Should().NotBeNull();
}
```

### Brittle Tests That Break on Refactoring
```csharp
// Bad: Coupled to method names and internal structure
[Fact]
public void CalculateTotal_Returns_Correct_Value()
{
    var result = order.CalculateTotal();
    Assert.Equal(expectedTotal, result);
}

// Good: Describes behavior independent of implementation
[Fact]
public void Order_total_includes_line_items_tax_and_shipping()
{
    var total = order.Total; // Implementation can change
    total.Should().Be(expectedTotal);
}
```

### Testing Too Many Things at Once
```csharp
// Bad: Tests multiple behaviors in one test
[Fact]
public async Task CreateOrder_Does_Everything_Correctly()
{
    // Tests validation, persistence, notifications, payments, etc.
}

// Good: One behavior per test
[Fact]
public async Task Creating_an_order_with_invalid_data_returns_validation_errors()

[Fact]
public async Task Creating_an_order_sends_confirmation_email_to_customer()

[Fact]
public async Task Creating_an_order_charges_the_payment_method()
```

---

## Running Tests

### Test Organization
- Use **xUnit traits** to categorize tests: `[Trait("Category", "Integration")]`
- Separate slow integration tests from fast unit tests
- Run fast tests frequently; integration tests before commits

### Parallel Execution
- xUnit runs test classes in parallel by default
- Use **collection fixtures** to share expensive setup across tests in a class
- Disable parallelism for tests that share state: `[Collection("Sequential")]`

### CI/CD Considerations
- Run unit tests on every commit
- Run integration tests on pull requests
- Use Testcontainers in CI pipelines (most CI services support Docker)
- Consider test result reporting and flaky test detection

---

## Further Reading

- **"Unit Testing Principles, Practices, and Patterns"** by Vladimir Khorikov — Comprehensive guide to effective testing
- **"Code That Fits in Your Head"** by Mark Seemann — Outside-in testing and sustainable architecture
- **xUnit documentation** — https://xunit.net
- **Testcontainers for .NET** — https://dotnet.testcontainers.org