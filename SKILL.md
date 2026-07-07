---
name: project-conventions
description: Architecture rules, naming conventions, and CQRS handler patterns for this .NET solution. Load this whenever generating, reviewing, or refactoring C# code in this repository so output matches the existing codebase instead of generic patterns.
---

# Project Conventions

This file defines how code is written in this codebase. It is not a general
C#/.NET style guide — it encodes the specific decisions this project has
already made, so an AI agent (or a new engineer) produces code that looks
like it was written by the same team, not a tutorial.

Apply these rules whenever you generate, edit, or review code in this
solution. If a request conflicts with a rule below, follow the rule and
say so, rather than silently defaulting to a generic pattern.

## 1. Architecture

- **Layering**: `Domain` → `Application` → `Infrastructure` → `Api`.
  Dependencies point inward only. `Domain` has no references to any other
  project. `Application` may reference `Domain` but never `Infrastructure`
  or `Api`.
- **No business logic in controllers/endpoints.** Endpoints only: bind the
  request, dispatch a command/query, map the result to an HTTP response.
- **No business logic in EF Core entities beyond invariants.** Entities may
  validate their own state (e.g. guard clauses in constructors/methods) but
  must not call out to services, repositories, or infrastructure.
- **Cross-cutting concerns** (logging, validation, transactions, caching)
  live in pipeline behaviors, not scattered inside handlers.

## 2. CQRS Handler Pattern

Every write operation is a **Command**, every read is a **Query**. Both
follow the same shape:

```csharp
// Commands/Orders/CreateOrder/CreateOrderCommand.cs
public sealed record CreateOrderCommand(Guid CustomerId, List<OrderLineDto> Lines)
    : IRequest<Result<OrderId>>;

// Commands/Orders/CreateOrder/CreateOrderHandler.cs
public sealed class CreateOrderHandler(
    IOrderRepository orders,
    IUnitOfWork unitOfWork) : IRequestHandler<CreateOrderCommand, Result<OrderId>>
{
    public async Task<Result<OrderId>> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId, request.Lines);
        if (order.IsFailure)
            return Result<OrderId>.Failure(order.Error);

        orders.Add(order.Value);
        await unitOfWork.SaveChangesAsync(ct);

        return Result<OrderId>.Success(order.Value.Id);
    }
}

// Commands/Orders/CreateOrder/CreateOrderValidator.cs
public sealed class CreateOrderValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Lines).NotEmpty();
    }
}
```

Rules:

- One folder per use case: `Commands/{Aggregate}/{UseCase}/` or
  `Queries/{Aggregate}/{UseCase}/`, containing the command/query, handler,
  and validator together. Do not group by technical type (no
  `Handlers/`, `Validators/` top-level folders).
- Handlers are `sealed` and take dependencies via primary constructor.
- Queries never touch `IUnitOfWork` or call `SaveChangesAsync`.
- Commands return `Result<T>` (or `Result` for no payload) — see §3. They
  do not throw for expected failures (validation, not-found, business
  rule violations). Exceptions are reserved for truly unexpected failures.

## 3. Result Pattern — No Exceptions for Control Flow

All `Application` layer operations that can fail in an expected way return
`Result` / `Result<T>`, never throw.

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T Value { get; }
    public Error Error { get; }
}
```

- `Api` layer maps `Result` failures to HTTP status codes in one place
  (a shared extension method / filter), not per-endpoint `if` chains.
- Reserve exceptions for programmer errors and truly exceptional
  infrastructure failures (DB unreachable, etc.), not for "customer not
  found" or "invalid input."

## 4. Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Commands | `{Verb}{Noun}Command` | `CreateOrderCommand` |
| Queries | `Get{Noun}{By/Filter}Query` | `GetOrderByIdQuery` |
| Handlers | `{CommandOrQueryName}Handler` | `CreateOrderHandler` |
| Validators | `{CommandOrQueryName}Validator` | `CreateOrderValidator` |
| DTOs (outbound) | `{Noun}Dto` / `{Noun}Response` | `OrderResponse` |
| DTOs (inbound) | `{Noun}Request` | `CreateOrderRequest` |
| Interfaces | `I{Noun}` — no `Async` suffix on the interface name itself | `IOrderRepository` |
| Async methods | Always suffixed `Async` | `SaveChangesAsync` |
| Private fields | `_camelCase` | `_orderRepository` |
| EF Core entities | Singular noun, no `Entity` suffix | `Order`, not `OrderEntity` |

## 5. EF Core Rules

- **No N+1 queries.** Use `.Include()`/`.ThenInclude()` or projection
  (`.Select()` into a DTO) for anything that will access related data.
  Prefer projection over `Include` for read-only queries — it avoids
  pulling entire entity graphs for data you only display.
- **Queries are `AsNoTracking()` by default.** Only track entities on the
  write path where you intend to call `SaveChangesAsync`.
- **No repository methods that return `IQueryable<T>`** across a layer
  boundary — evaluate the query and return concrete data before leaving
  `Infrastructure`.
- Migrations are named `{Timestamp}_{PastTenseDescription}`, e.g.
  `20260107_AddedOrderStatusColumn`.

## 6. Testing

- Unit tests target `Application` handlers with mocked/in-memory
  repositories — no real database.
- Integration tests use a real database (Testcontainers or LocalDB), never
  a mocked `DbContext`. Mocking EF Core behavior has previously hidden real
  migration/tracking bugs — don't reintroduce that gap.
- Test naming: `MethodOrHandlerName_Scenario_ExpectedOutcome`, e.g.
  `CreateOrderHandler_WhenCustomerMissing_ReturnsFailure`.

## 7. What Not to Do

- Do not introduce a new abstraction (generic repository, base handler,
  service locator) unless at least three concrete cases already need it.
- Do not add `try/catch` around code that can't actually throw in a way
  the caller can meaningfully recover from.
- Do not use `AutoMapper`/reflection-based mapping for simple DTOs — hand
  write the mapping or use a static `ToDto()` extension method. Keeps
  mapping logic debuggable and avoids hidden reflection cost.
- Do not put configuration values, connection strings, or secrets in code
  — `IOptions<T>` bound from configuration only.

---

*Copy this file into your project root as `SKILL.md` (or your agent's
equivalent skill file). Edit §1–§7 to match your actual decisions — the
value isn't this exact content, it's having your team's real conventions
written down so your AI agent stops guessing.*
