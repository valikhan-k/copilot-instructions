# Architecture Guidelines

## Hexagonal Architecture

**Core Principle**: Isolate business logic from external concerns (databases, APIs, UI, infrastructure).

### Layers

**Domain** (Core)
- Pure business entities, rules, and logic
- No external dependencies
- Use plain C# records and classes
- Apply business rules through methods directly on models

**Application** (Use Cases)
- Orchestrates domain logic
- Defines interfaces for external dependencies (e.g., `IOrderRepository`)
- Provides application services that API layer calls
- Depends on Domain only

**Infrastructure** (External Concerns)
- Implements Application interfaces
- Handles databases, HTTP clients, file systems, message queues
- Depends on Application and Domain

**API/Presentation** (Entry Points)
- HTTP endpoints, request/response handling
- Depends on Application for use cases
- References Infrastructure only for DI registration

### Dependency Rules

```
Domain → (nothing)
Application → Domain
Infrastructure → Application + Domain
API → Application + Infrastructure (DI only)
```

## Folder Organization

Follow hexagonal architecture principles with pragmatic evolution.
Start simple, add structure only when complexity demands it.

Examples (lean on existing codebase namings and patterns):

### Small projects & microservices

Single project with layer-based folders:
```
/Api
  CreateOrder.cs           # Endpoint + DTOs
  RegisterCustomer.cs
/Application
  OrderService.cs          # Business logic
  IOrderRepository.cs      # Port (interface)
/Domain
  Order.cs                 # Core domain model
  Customer.cs
/Infrastructure
  OrderRepository.cs       # Adapter (implementation)
  PaymentGatewayAdapter.cs
```

**When to evolve:** Multiple developers, >15 files per folder, or cross-cutting concerns blur.

### Medium projects

Add feature folders within layers:
```
/Api
  /Orders
    CreateOrder.cs
    CreateOrderDto.cs
  /Customers
    RegisterCustomer.cs
    RegisterCustomerDto.cs
/Application
  /Orders
    OrderService.cs
    IOrderRepository.cs
  /Customers
    CustomerService.cs
    ICustomerRepository.cs
/Domain
  /Orders
    Order.cs
  /Customers
    Customer.cs
/Infrastructure
  /Persistence
    OrderRepository.cs
    CustomerRepository.cs
  /ExternalServices
    PaymentGatewayAdapter.cs
```

**When to evolve:** Teams working on different features, or preparing for extraction.

### Large projects & modular monoliths

Vertical slices per bounded context (project or top-level folder):
```
/Orders                     # Self-contained module
  /Api
    CreateOrder.cs
  /Application
    OrderService.cs
    IOrderRepository.cs
  /Domain
    Order.cs
  /Infrastructure
    OrderRepository.cs
    
/Customers
  /Api
    RegisterCustomer.cs
  /Application
    CustomerService.cs
  /Domain
    Customer.cs
  /Infrastructure
    CustomerRepository.cs
```

**When to use:** Multiple teams, clear bounded contexts, or microservice migration path.

---

**Key principle:** Defer structure until pain points emerge. Wrong abstraction is worse than duplication.

## API Design

**One endpoint per file** - each endpoint in its own .cs file within feature folder

```csharp
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

// Program.cs
CreateOrder.Map(app);
GetOrder.Map(app);
```

**Keep endpoints thin** - delegate to services for business logic

**Co-locate DTOs** with their endpoint in the same feature folder

## Domain Modeling

**No heavyweight DDD patterns** - Skip Entity, ValueObject, DomainEvent, Aggregate base classes unless explicitly needed.

```csharp
// Good: Simple, focused domain model
public record Order(
    OrderId Id, 
    CustomerId CustomerId, 
    List<OrderLine> Lines, 
    OrderStatus Status)
{
    public decimal CalculateTotal() => 
        Lines.Sum(l => l.Quantity * l.UnitPrice);
    
    public bool CanBeCancelled() => 
        Status is OrderStatus.Pending;
}

// Avoid: Unnecessary ceremony
public class Order : Entity, IAggregateRoot { }
```

## Evolutionary Approach

**Start small, single project**
- Use folder boundaries initially
- Keep everything in one project until complexity demands separation

**Extract when complexity emerges**
- Move to separate projects only when boundaries are clear
- Benefits must justify the overhead

**Grow organically**
- Let actual usage patterns dictate structure
- Don't pre-optimize for theoretical future needs

## Abstraction Principles

- Avoid premature abstraction
- Duplication is better than wrong abstraction
- Extract only when patterns emerge across 3+ locations
- Keep code "wet" (Write Everything Twice) before drying
- Prefer composition over inheritance
- Challenge every base class: does it provide value?