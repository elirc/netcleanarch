# Domain & Architecture Glossary

Product nouns first, then architecture vocabulary as this repo uses it. Every term is anchored. Terms marked 🎤 are interview vocabulary — expect to be asked to define them *with an example*.

## Product / domain nouns

| Term | Meaning here | Code |
|---|---|---|
| **TodoList** | A named, coloured collection of items. The aggregate-ish root of the sample domain. | [TodoList.cs](../../src/Domain/Entities/TodoList.cs) |
| **TodoItem** | A task inside a list: title, note, priority, done flag. | [TodoItem.cs](../../src/Domain/Entities/TodoItem.cs) |
| **Colour** | A value object constrained to seven supported hex codes. | [Colour.cs](../../src/Domain/ValueObjects/Colour.cs) |
| **PriorityLevel** | Enum None/Low/Medium/High; crosses the API as an `int` in [TodoItemDto](../../src/Application/TodoLists/Queries/GetTodos/TodoItemDto.cs#L15) but as the enum in [UpdateTodoItemDetailCommand](../../src/Application/TodoItems/Commands/UpdateTodoItemDetail/UpdateTodoItemDetail.cs#L12). Near-synonym trap: same concept, two wire representations. | [PriorityLevel.cs](../../src/Domain/Enums/PriorityLevel.cs) |
| **TodosVm** | The "view model" returned by `GetTodos`: lists + the lookup data (colours, priorities) the UI needs to render pickers. One round trip instead of three. | [TodosVm.cs](../../src/Application/TodoLists/Queries/GetTodos/TodosVm.cs) |
| **LookupDto** | Generic id+title pair for dropdowns. | [LookupDto.cs](../../src/Application/Common/Models/LookupDto.cs) |
| **ApplicationUser** | ASP.NET Core Identity user; deliberately empty subclass so you can add profile fields later. | [ApplicationUser.cs](../../src/Infrastructure/Identity/ApplicationUser.cs) |
| **Administrator** | The only role constant. | [Roles.cs](../../src/Domain/Constants/Roles.cs) |

## Architecture vocabulary, as practiced here

| Term | Definition (junior-friendly) | Where you can point at it |
|---|---|---|
| 🎤 **Boundary** | A line between parts of the system where knowledge is deliberately limited, crossed only through a contract. | Application/Infrastructure boundary = [IApplicationDbContext](../../src/Application/Common/Interfaces/IApplicationDbContext.cs); Web/Application boundary = MediatR `ISender`. |
| 🎤 **Contract** | The agreed shape of an interaction: request/response types, error semantics, invariants the caller may rely on. | Command records are the HTTP contract (e.g. [UpdateTodoItemCommand](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L5-L12)); the OpenAPI doc is the FE/BE contract, enforced by codegen. |
| 🎤 **Invariant** | A condition that must *always* hold, not merely be checked once. | "A list's colour is one of seven supported codes" — enforced by [Colour.From](../../src/Domain/ValueObjects/Colour.cs#L5-L15) (with a leak via the public constructor). "Completed event fires only on false→true" — [Done setter](../../src/Domain/Entities/TodoItem.cs#L14-L26). |
| 🎤 **Composition root** | The single place where all abstractions are bound to implementations. | [Web/Program.cs](../../src/Web/Program.cs#L7-L12) calling the three `Add*Services` extensions. |
| **CQRS** | Commands (writes) and Queries (reads) modeled as separate objects with separate handlers. Here it's "CQRS-lite": same database, no separate read store. | Folder shape under [src/Application/TodoLists](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs). |
| **Pipeline behaviour** | Middleware for MediatR requests; the request passes through each in registration order. | [DependencyInjection.cs](../../src/Application/DependencyInjection.cs#L16-L23) |
| **Domain event** | A fact that happened in the domain, published for other parts to react to without coupling. | [TodoItemCompletedEvent](../../src/Domain/Events/TodoItemCompletedEvent.cs) → [LogTodoItemCompleted](../../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs) |
| **Interceptor** | An EF Core hook that runs around `SaveChanges` — this repo's mechanism for auditing and event dispatch. | [AuditableEntityInterceptor](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs), [DispatchDomainEventsInterceptor](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs) |
| 🎤 **Authentication vs Authorization** | Who you are vs what you may do. 401 vs 403. | [AuthorizationBehaviour](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L26-L58) throws `UnauthorizedAccessException` (→401) vs `ForbiddenAccessException` (→403), mapped in [ProblemDetailsExceptionHandler](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L31-L42). |
| 🎤 **Isolation (tenant/user)** | Guaranteeing one user's data is invisible to another. **Absent here** — a deliberate template simplification and this curriculum's favorite exercise. | No ownership filter in [GetTodos.cs](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47) |
| 🎤 **Idempotency** | Doing an operation twice has the same effect as once. PUT here is idempotent by design; POST create is not. | [TodoItems.cs](../../src/Web/Endpoints/TodoItems.cs#L15-L18) verb choices |
| 🎤 **Consistency** | Whether related state changes land together. Domain events dispatched *in the same `SaveChanges` call* keep handler DB writes in the same transaction scope — but a handler's *non-DB* side effect can happen even if the save then fails. | [DispatchDomainEventsInterceptor.cs](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L30) |
| 🎤 **Observability** | Can you tell what the system did from the outside? Logs/metrics/traces via OpenTelemetry. | [ServiceDefaults/Extensions.cs](../../src/ServiceDefaults/Extensions.cs#L44-L71) |
| 🎤 **Migration** | A versioned, replayable schema change. This repo has none — it uses `EnsureCreated` in dev. Know the difference cold. | [ApplicationDbContextInitialiser.cs](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46) |
| 🎤 **Blast radius** | Everything that can break when you change X. Endpoint method names have surprising blast radius here (they become client method names via codegen). | [EndpointRouteBuilderExtensions.cs](../../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs#L5-L11) |
| **Value object** | A type defined by its value, not identity; equal if components equal. | [ValueObject.cs](../../src/Domain/Common/ValueObject.cs#L23-L44) |
| **Guard clause** | Fail-fast precondition check. `Guard.Against.NotFound(id, entity)` throws the `NotFoundException` that becomes a 404. | [DeleteTodoItem.cs](../../src/Application/TodoItems/Commands/DeleteTodoItem/DeleteTodoItem.cs#L21) |
| **Seed data** | Baseline rows created at startup in dev: admin user + sample list. | [ApplicationDbContextInitialiser.cs](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L68-L109) |
| **Optimistic UI** | Update the screen before the server confirms; reconcile on failure (this repo skips the reconcile). | [Todo.jsx updateCheckbox](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167) |

## Confusables

- **`AuthorizeAttribute`** exists twice: ASP.NET Core's (used implicitly by `RequireAuthorization()` on endpoint groups) and this repo's own [Application.Common.Security.AuthorizeAttribute](../../src/Application/Common/Security/AuthorizeAttribute.cs) placed on *request records*. Different layers, both enforced.
- **`ValidationException`** also exists twice: FluentValidation's and [the Application's own](../../src/Application/Common/Exceptions/ValidationException.cs) — note the alias import in [ValidationBehaviour.cs L1](../../src/Application/Common/Behaviours/ValidationBehaviour.cs#L1).
- **`DependencyInjection.cs`** appears three times (Application, Infrastructure, Web) and all sit in the `Microsoft.Extensions.DependencyInjection` namespace so callers don't need extra usings — a common .NET idiom worth recognizing.
- **`ClientApp` vs `ClientApp-React`**: Angular is the template default; the React app is the alternative. Both exist in *this* repo because it's the template source; a generated solution gets only one.
- **Functional vs Integration tests**: `Application.FunctionalTests` is the real, rich suite; [Infrastructure.IntegrationTests](../../tests/Infrastructure.IntegrationTests/GlobalUsings.cs) is an empty placeholder project (one `GlobalUsings.cs`). Don't go looking for substance there.

## Drill

Pick five 🎤 terms and, for each, say one sentence of definition + one sentence pointing at this repo. **Solid** = anchors are correct. **Strong** = for *invariant*, *idempotency*, and *isolation* you can also name where this repo falls short and how you'd fix it.
