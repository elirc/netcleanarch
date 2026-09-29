# Boundaries and Layers

A **boundary** is a line you may only cross through a contract. A **layer** is a set of code sharing one responsibility and one dependency direction. Clean Architecture's single rule: source-code dependencies point *inward*, toward policy, away from mechanism.

## What each layer owns — and must not own

### Domain ([src/Domain](../../src/Domain/Domain.csproj))
Owns: entities ([TodoItem](../../src/Domain/Entities/TodoItem.cs), [TodoList](../../src/Domain/Entities/TodoList.cs)), value objects ([Colour](../../src/Domain/ValueObjects/Colour.cs)), domain events ([TodoItemCompletedEvent](../../src/Domain/Events/TodoItemCompletedEvent.cs)), enums, domain exceptions, base types ([BaseEntity](../../src/Domain/Common/BaseEntity.cs), [BaseAuditableEntity](../../src/Domain/Common/BaseAuditableEntity.cs)).
Must not own: persistence knowledge, HTTP anything, use-case orchestration.
Dependency budget: **one** NuGet package (`MediatR.Contracts`) — a measured, documented violation of "zero dependencies" ([ADR-003](../../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md)). The `[NotMapped]` attribute on [BaseEntity.DomainEvents](../../src/Domain/Common/BaseEntity.cs#L13-L14) is a second, quieter leak: a persistence annotation living in the domain. Small, pragmatic, worth noticing — the alternative (configuring `Ignore` in Infrastructure) is purer and one refactor kata away.

### Application ([src/Application](../../src/Application/Application.csproj))
Owns: use cases (commands/queries/handlers), validation rules, authorization *policy* (which requests need what), DTO shapes, the pipeline behaviours, and — critically — **the interfaces outer layers must implement**: [IApplicationDbContext](../../src/Application/Common/Interfaces/IApplicationDbContext.cs), [IUser](../../src/Application/Common/Interfaces/IUser.cs), [IIdentityService](../../src/Application/Common/Interfaces/IIdentityService.cs).
Must not own: how identity is stored, which database runs, HTTP status codes.
Deliberate coupling: EF Core's `DbSet`/`IQueryable` appear in the interface — the repo's most argued-about decision, defended in [ADR-001](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md). Read it twice; it's a model of *how to argue a tradeoff* (dependency direction vs framework reference; "repositories hide, not remove, the coupling").

### Infrastructure ([src/Infrastructure](../../src/Infrastructure/Infrastructure.csproj))
Owns: [ApplicationDbContext](../../src/Infrastructure/Data/ApplicationDbContext.cs) (implements Application's interface), EF configurations, interceptors, Identity ([IdentityService](../../src/Infrastructure/Identity/IdentityService.cs), [ApplicationUser](../../src/Infrastructure/Identity/ApplicationUser.cs)), database provider selection.
Must not own: business rules. Test: could you describe any Infrastructure file's job without mentioning a business concept? [AuditableEntityInterceptor](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs) passes — "stamp who/when on anything auditable" mentions no todo-anything.

### Web ([src/Web](../../src/Web/Web.csproj))
Owns: HTTP translation (endpoints, status codes, ProblemDetails), OpenAPI, the composition root, [CurrentUser](../../src/Web/Services/CurrentUser.cs) (claims → `IUser`), static file serving, SPA hosting.
Must not own: business decisions. The endpoints prove it — the fattest endpoint logic in the repo is an id-consistency check ([TodoItems.cs L34-L35](../../src/Web/Endpoints/TodoItems.cs#L34-L35)).

## Boundaries done well — three specimens

1. **`IUser`**: Application needs "who is acting" but must not know about HTTP. [IUser](../../src/Application/Common/Interfaces/IUser.cs) (two properties) is defined inward; [CurrentUser](../../src/Web/Services/CurrentUser.cs) implements it from `HttpContext` outward. Payoff: tests swap it with a mock trivially ([WebApiFactory L17-L28](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28)), and a future message-queue consumer could implement it from message metadata. This is dependency inversion earning its keep, in ~20 lines.
2. **`IIdentityService`**: Application asks "is user X in role Y / authorized for policy Z" ([AuthorizationBehaviour L67](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L67)) without knowing ASP.NET Identity exists. All `UserManager`/`ClaimsPrincipal` machinery stays in [IdentityService](../../src/Infrastructure/Identity/IdentityService.cs).
3. **Domain events as the Domain→Application channel**: the Domain announces ([TodoItem raises the event](../../src/Domain/Entities/TodoItem.cs#L17-L24)); Application reacts ([LogTodoItemCompleted](../../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs)); Infrastructure carries ([DispatchDomainEventsInterceptor](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs)). No layer references the wrong direction.

## Boundary leaks and tensions — learn to spot them

| Leak | Evidence | Severity |
|---|---|---|
| Users endpoint reaches into Infrastructure | [Users.cs L1-L19](../../src/Web/Endpoints/Users.cs#L1-L19) imports `CleanArchitecture.Infrastructure.Identity` and takes `SignInManager<ApplicationUser>` directly — no Application use case in between | Accepted tradeoff: wrapping `MapIdentityApi` in CQRS would be ceremony. But *know* it's an exception, not the pattern |
| Web references Infrastructure for startup | [Program.cs L1](../../src/Web/Program.cs#L1) imports `Infrastructure.Data` for `InitialiseDatabaseAsync` | Fine — composition root is *allowed* to know everything; that's its job |
| Persistence attribute in Domain | `[NotMapped]` ([BaseEntity L13](../../src/Domain/Common/BaseEntity.cs#L13)) | Cosmetic |
| EF Core types in Application contracts | `DbSet<>` in [IApplicationDbContext](../../src/Application/Common/Interfaces/IApplicationDbContext.cs) | Deliberate, ADR'd. Say "documented tradeoff," not "violation" |
| Wire format = command records | Endpoints bind HTTP JSON straight to Application commands — the HTTP contract and the use-case contract are the same type. Cheap and consistent; the cost appears the day one HTTP field shouldn't reach the handler (e.g. adding a server-set `OwnerId`) | Watch-item |

The last row deserves an interview-ready sentence: *"binding transport DTOs directly to application commands trades an anti-corruption seam for less mapping code — fine until the wire shape and the use-case shape need to diverge."*

## How to find layer violations in any repo (transferable procedure)

1. Read the `.csproj` (or package.json) references first — the dependency graph can't lie.
2. Grep inner layers for outer namespaces (here: `rg "CleanArchitecture.Infrastructure" src/Application src/Domain` returns nothing — verify it stays that way in review).
3. Look for framework types in interface signatures — each one is a coupling decision; ask if it's ADR-grade deliberate or accidental.
4. Check where `HttpContext` appears. Here: only Web ([CurrentUser](../../src/Web/Services/CurrentUser.cs)). If it shows up in Application, the boundary is gone.

## Drill

A teammate proposes adding `Task SendEmailAsync(string to, string subject, string body)` to `IApplicationDbContext` "since it's already injected everywhere." Write your review comment: which layer should own the interface, which the implementation, what the interface should be named, and why the DbContext is the wrong host. Self-grade — Basic: "make an `IEmailService`." Solid: interface in `Application/Common/Interfaces`, implementation in Infrastructure, registered in [Infrastructure/DependencyInjection.cs](../../src/Infrastructure/DependencyInjection.cs). Strong: you also raise the reliability question (email inside a request? see [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md)) and single-responsibility for the DbContext.
