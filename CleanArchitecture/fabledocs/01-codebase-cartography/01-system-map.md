# System Map

## Shape

A single .NET solution ([CleanArchitecture.slnx](../../CleanArchitecture.slnx)) — not a monorepo of independent services — with seven `src` projects, six `tests` projects, and two embedded SPA frontends that live *inside* the Web project folder but run as separate Node processes in development.

```
                       ┌────────────────────────────┐
                       │  AppHost (Aspire)          │  orchestrates everything below
                       └──────┬──────────────┬──────┘
                              │              │
              ┌───────────────▼───┐   ┌──────▼──────────────────┐
   Node dev   │ ClientApp (ng 21) │   │ Web (ASP.NET Core 10)   │◄── ServiceDefaults
   servers    │ ClientApp-React   │──►│  Minimal API endpoints  │    (OTel, health,
   (proxy /api│  (React 19+Vite)  │   │  OpenAPI + Scalar       │     resilience)
    to Web)   └───────────────────┘   └──────┬──────────────────┘
                                             │ ISender.Send(command/query)
                                      ┌──────▼──────────────────┐
                                      │ Application             │
                                      │  CQRS handlers          │
                                      │  Pipeline behaviours    │
                                      │  IApplicationDbContext  │──┐ interfaces defined here,
                                      │  IUser, IIdentityService│  │ implemented in outer layers
                                      └──────┬──────────────────┘  │
                                             │ uses entities       │
                                      ┌──────▼──────────┐   ┌──────▼──────────────┐
                                      │ Domain          │   │ Infrastructure      │
                                      │  Entities,      │◄──│  ApplicationDbContext│
                                      │  ValueObjects,  │   │  Identity, EF Core, │
                                      │  Events         │   │  Interceptors       │
                                      └─────────────────┘   └─────────┬───────────┘
                                                                      │
                                                              SQLite / PostgreSQL / SQL Server
```

Dependency rule: arrows of *compile-time dependency* point inward. `Domain` depends on almost nothing (one deliberate exception: `MediatR.Contracts`, [ADR-003](../../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md)). `Application` depends on Domain + EF Core abstractions ([ADR-001](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md)). `Infrastructure` depends on Application (it *implements* [IApplicationDbContext](../../src/Application/Common/Interfaces/IApplicationDbContext.cs) and [IIdentityService](../../src/Application/Common/Interfaces/IIdentityService.cs)). `Web` depends on everything and is the **composition root** — the one place all pieces are wired together ([Program.cs](../../src/Web/Program.cs#L7-L12)).

## Ownership map

| Concern | Lives in | Key files |
|---|---|---|
| HTTP surface / routing | `src/Web/Endpoints` | [TodoLists.cs](../../src/Web/Endpoints/TodoLists.cs), [TodoItems.cs](../../src/Web/Endpoints/TodoItems.cs), [Users.cs](../../src/Web/Endpoints/Users.cs), [WeatherForecasts.cs](../../src/Web/Endpoints/WeatherForecasts.cs) |
| Endpoint conventions (reflection discovery, operationId naming) | `src/Web/Infrastructure` | [IEndpointGroup.cs](../../src/Web/Infrastructure/IEndpointGroup.cs), [WebApplicationExtensions.cs](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28), [EndpointRouteBuilderExtensions.cs](../../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs) |
| Error contract (ProblemDetails) | `src/Web/Infrastructure` | [ProblemDetailsExceptionHandler.cs](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44) |
| OpenAPI shaping | `src/Web/Infrastructure` | [ApiExceptionOperationTransformer.cs](../../src/Web/Infrastructure/ApiExceptionOperationTransformer.cs), [IdentityApiOperationTransformer.cs](../../src/Web/Infrastructure/IdentityApiOperationTransformer.cs) |
| Use cases (business operations) | `src/Application/{Feature}/{Commands\|Queries}` | e.g. [CreateTodoList.cs](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs) |
| Cross-cutting request pipeline | `src/Application/Common/Behaviours` | five behaviours, registered in [DependencyInjection.cs](../../src/Application/DependencyInjection.cs#L16-L23) |
| Domain model + invariants | `src/Domain` | [TodoItem.cs](../../src/Domain/Entities/TodoItem.cs), [Colour.cs](../../src/Domain/ValueObjects/Colour.cs) |
| Persistence + schema config | `src/Infrastructure/Data` | [ApplicationDbContext.cs](../../src/Infrastructure/Data/ApplicationDbContext.cs), [Configurations/](../../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs), [Interceptors/](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs) |
| Identity / users | `src/Infrastructure/Identity` + `src/Web/Endpoints/Users.cs` | [IdentityService.cs](../../src/Infrastructure/Identity/IdentityService.cs) |
| Current-user access | `src/Web/Services` | [CurrentUser.cs](../../src/Web/Services/CurrentUser.cs) implements Application's [IUser](../../src/Application/Common/Interfaces/IUser.cs) |
| Orchestration | `src/AppHost`, `tests/TestAppHost` | [AppHost/Program.cs](../../src/AppHost/Program.cs), [TestAppHost/Program.cs](../../tests/TestAppHost/Program.cs) |
| Observability / resilience defaults | `src/ServiceDefaults` | [Extensions.cs](../../src/ServiceDefaults/Extensions.cs) |
| Shared service names | `src/Shared` | [Services.cs](../../src/Shared/Services.cs) |
| React UI | `src/Web/ClientApp-React/src` | [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx), [AuthContext.jsx](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx) |
| Angular UI | `src/Web/ClientApp/src` | [todo.component.ts](../../src/Web/ClientApp/src/app/todo/todo.component.ts), [auth.service.ts](../../src/Web/ClientApp/src/api-authorization/auth.service.ts) |
| Tests | `tests/*` | see [05-quality-engineering/01-testing-strategy.md](../05-quality-engineering/01-testing-strategy.md) |
| Template scaffolding for new use cases | `templates/ca-use-case` | [Commands template](../../templates/ca-use-case/FeatureName/Commands/CleanArchitectureUseCase/CleanArchitectureUseCase.cs) |

## Public interfaces vs private internals

**Public contracts** (changing these has a wide *blast radius* — the set of things a change can break):

- The HTTP API: routes derived from class names via [MapEndpoints](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28) (`/api/TodoLists`, `/api/TodoItems`, `/api/Users`, `/api/WeatherForecasts`) plus the framework-generated Identity endpoints ([Users.cs](../../src/Web/Endpoints/Users.cs#L12)).
- The OpenAPI document — because **both frontends generate their API clients from it** (NSwag config: [React](../../src/Web/ClientApp-React/nswag.json#L4-L11), [Angular](../../src/Web/ClientApp/nswag.json)). Rename a C# endpoint method and the generated TypeScript method name changes with it ([EndpointRouteBuilderExtensions.cs](../../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs#L20-L26) uses the method name as operationId).
- DTO/command shapes in `src/Application` — they *are* the wire format; Minimal APIs bind request JSON straight to command records (e.g. [CreateTodoItemCommand](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs#L6-L11)).
- The database schema (implicitly, via EF model + `EnsureCreated`).
- For template consumers: the `dotnet new ca-sln` parameter surface and the `#if` symbols.

**Private internals** (safe to refactor freely): handler bodies, behaviours' implementation details, interceptors, React component internals, test helpers.

Senior noticing: the *generated* files — `web-api-client.ts` in both clients — are build artifacts, not source. Never teach yourself patterns from them, and never edit them (they're regenerated by `npm run generate-api` on every start/build).

## Drill

Without opening the answer: in which project would you put (a) a new `DueDate` property on todo items, (b) an email-sending service, (c) a rule that a list may hold at most 100 items, (d) a new `/api/Tags` endpoint group, (e) the interface that (b) is consumed through?

Self-grade — **Basic:** (a) Domain. **Solid:** (b) Infrastructure, (d) Web/Endpoints, (e) Application/Common/Interfaces. **Strong:** (c) sparks a real discussion — enforced in the Domain entity to be an invariant, but *validated* in an Application validator for a friendly 400; you can articulate why both places exist and what each protects against.
