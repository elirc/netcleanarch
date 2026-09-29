# ADR-004: EF Core Exposed via `IApplicationDbContext`; No Repository Layer

**Status:** Accepted — and the authors documented this one explicitly in [docs/decisions/ADR-001](../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md). Reverse-engineered here independently and confirmed.

## Context

The Application layer must read and write data. Clean Architecture restricts inner layers from depending on outer ones, and ORMs conventionally live in Infrastructure. The question: should the Application layer access EF Core through an interface it owns, or should a repository layer sit between them to "hide" the ORM?

## Decision

Define [IApplicationDbContext](../src/Application/Common/Interfaces/IApplicationDbContext.cs) in the Application layer, exposing `DbSet<TodoList>`, `DbSet<TodoItem>`, and `SaveChangesAsync`. Handlers use it directly with full LINQ. [ApplicationDbContext](../src/Infrastructure/Data/ApplicationDbContext.cs) in Infrastructure implements it. **No repository layer.** The concrete context is registered once and forwarded so both the interface and the class resolve the *same* scoped instance ([Infrastructure/DependencyInjection.cs L42](../src/Infrastructure/DependencyInjection.cs#L42)).

Dependency inversion is still satisfied: the interface is defined *inward*, Infrastructure depends on Application. The Application layer takes a compile-time reference on EF Core's abstractions (`DbSet`, `IQueryable`) but knows nothing of the concrete context or provider. *A reference to an assembly is not a dependency on an implementation* — that distinction is the whole argument.

## Alternatives Considered

- **Repository interfaces in Application, implemented in Infrastructure.** The textbook Clean Architecture move. Rejected because repositories don't *remove* the EF coupling, they *hide* it: you still express intent in ORM terms through methods like `GetActiveOrderedByNameWithItems`, adding indirection and a maintenance burden for the *illusion* of portability. The authors' ADR-001 makes exactly this case.
- **Generic `IRepository<T>`.** Even leakier — either too generic to express real queries, or a dumping ground of one-off methods. Rejected.

## Consequences

**Positive**
- Handlers use the full expressiveness of EF Core — projection, includes, `AsNoTracking` — without workarounds ([GetTodos L43-L47](../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47)).
- One less layer to design and keep in sync as handler needs evolve.
- The data-access path is honest and short: handler → `IApplicationDbContext` → `ApplicationDbContext`.

**Negative**
- **Handlers can't be unit-tested with a mocked repository** — they need a real database, which is *why* the testing strategy is what it is ([ADR-008](adr-008-real-database-functional-testing.md)). The two decisions reinforce each other.
- Swapping ORMs would be an Application-layer rewrite, not an Infrastructure swap. Accepted as YAGNI — greenfield projects essentially never replace their ORM.
- Query logic can duplicate across handlers with no natural home for reuse; once ownership filtering is needed ([ADR-003](adr-003-behaviour-pipeline.md)), "lists visible to user X" would appear in many handlers, arguing for an EF global query filter rather than copy-paste.
- Does **not** apply to a true DDD model — there, repositories are a *domain* concept expressed in aggregate terms, and `IApplicationDbContext` should not be used directly. The authors' ADR-001 states this caveat explicitly.

## Evidence

[IApplicationDbContext](../src/Application/Common/Interfaces/IApplicationDbContext.cs) (three members), the same-instance forwarding at [Infrastructure/DependencyInjection.cs L42](../src/Infrastructure/DependencyInjection.cs#L42), direct LINQ in every handler (e.g. [DeleteTodoList L16-L27](../src/Application/TodoLists/Commands/DeleteTodoList/DeleteTodoList.cs#L16-L27)), and the authors' own [ADR-001](../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md).
