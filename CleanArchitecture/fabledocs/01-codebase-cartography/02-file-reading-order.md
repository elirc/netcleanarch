# File Reading Order

Thirty files, ordered so each one makes the next legible. Per file: why it matters, what to look for, what to ignore. Tracks: files 1–12 are the **junior path**; add 13–22 for **mid**; add 23–30 for **senior**.

## Junior path — "how does a request work here?"

| # | File | Why / look for | Ignore |
|---|---|---|---|
| 1 | [README.md](../../README.md) | Repo identity: it's a *template*. Note the `dotnet new ca-sln` options — they explain every `#if` you'll meet. | Badge noise |
| 2 | [src/Web/Program.cs](../../src/Web/Program.cs) | Composition root. Middleware **order**: HSTS → HTTPS redirect → CORS → static files → OpenAPI → exception handler → endpoints. Predict: why is `UseExceptionHandler` ([L38](../../src/Web/Program.cs#L38)) before `MapEndpoints`? | `#if (UseApiOnly)` blocks |
| 3 | [src/Web/Infrastructure/IEndpointGroup.cs](../../src/Web/Infrastructure/IEndpointGroup.cs) | The endpoint convention contract: `static abstract Map`, default route prefix `/api/{ClassName}`. | — |
| 4 | [src/Web/Infrastructure/WebApplicationExtensions.cs](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28) | Reflection discovery of endpoint groups. This is why adding a class = adding routes with zero registration. | — |
| 5 | [src/Web/Endpoints/TodoLists.cs](../../src/Web/Endpoints/TodoLists.cs) | The thin-endpoint pattern: bind → `sender.Send` → typed result. Note `RequireAuthorization()` at group level ([L13](../../src/Web/Endpoints/TodoLists.cs#L13)) and the id-mismatch guard ([L43](../../src/Web/Endpoints/TodoLists.cs#L43)). | — |
| 6 | [src/Application/DependencyInjection.cs](../../src/Application/DependencyInjection.cs#L16-L23) | The pipeline registration **order** — it is execution order. Memorize it. | — |
| 7 | [src/Application/TodoLists/Queries/GetTodos/GetTodos.cs](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs) | A query: `[Authorize]` on the record ([L9](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L9)), `AsNoTracking`, `ProjectTo`. | — |
| 8 | [src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs) + [its validator](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs) | A command with an *async DB-backed* validation rule (`BeUniqueTitle`). Ask: what happens if two requests race? |
| 9 | [src/Domain/Entities/TodoList.cs](../../src/Domain/Entities/TodoList.cs) + [TodoItem.cs](../../src/Domain/Entities/TodoItem.cs) | Anemic-ish entities, except the `Done` setter ([L14-L26](../../src/Domain/Entities/TodoItem.cs#L14-L26)) which raises a domain event on transition. | — |
| 10 | [src/Application/Common/Interfaces/IApplicationDbContext.cs](../../src/Application/Common/Interfaces/IApplicationDbContext.cs) | The Application-owned persistence contract. Then read [ADR-001](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md) — the single best tradeoff writeup in the repo. | — |
| 11 | [src/Web/ClientApp-React/src/components/Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx) | Whole feature UI. Look for: generated clients ([L3-L6](../../src/Web/ClientApp-React/src/components/Todo.jsx#L3-L6)), optimistic update in `updateCheckbox` ([L161-L167](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167)), server error parsing in `commitNewList` ([L77-L83](../../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83)). | Dialog markup details |
| 12 | [src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx) | Client auth state: probes `manage/info` to detect a live cookie session. | — |

## Mid path — "where do correctness and safety come from?"

| # | File | Why / look for |
|---|---|---|
| 13 | [ValidationBehaviour.cs](../../src/Application/Common/Behaviours/ValidationBehaviour.cs) | Collects *all* validator failures, throws one `ValidationException`. |
| 14 | [AuthorizationBehaviour.cs](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs) | Authn vs authz split: 401 when no user ([L29-L32](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L29-L32)), 403 on role/policy failure ([L55-L58](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L55-L58)). |
| 15 | [ProblemDetailsExceptionHandler.cs](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs) | Exception → HTTP status mapping. Ask: where does `NotFoundException` get thrown? (Answer: `Guard.Against.NotFound`, an Ardalis.GuardClauses call, e.g. [UpdateTodoItem.cs L28](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L28).) |
| 16 | [AuditableEntityInterceptor.cs](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs) | Cross-cutting persistence concern done once at the DbContext seam, not in every handler. |
| 17 | [DispatchDomainEventsInterceptor.cs](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs) | Events published **before** `SaveChanges` completes ([L25-L30](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L30)). Sit with the implications. |
| 18 | [src/Infrastructure/DependencyInjection.cs](../../src/Infrastructure/DependencyInjection.cs) | Provider selection via template `#if`, cookie vs bearer auth modes ([L46-L74](../../src/Infrastructure/DependencyInjection.cs#L46-L74)), `TimeProvider.System` ([L76](../../src/Infrastructure/DependencyInjection.cs#L76)). |
| 19 | [ApplicationDbContextInitialiser.cs](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs) | `EnsureDeleted/EnsureCreated` + seeding. Know why this can't ship to production. |
| 20 | [tests/Application.FunctionalTests/FunctionalTestSetup.cs](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs) | Aspire spins up a real DB for tests; `WebApiFactory` swaps `IUser` with a mock ([WebApiFactory.cs](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28)). |
| 21 | [tests/Application.FunctionalTests/Infrastructure/TestApp.cs](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs) | The test façade: `SendAsync`, `RunAsUserAsync`, `ResetState` (Respawn). Note the static `_userId` ([L13-L14](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L13-L14)) — what does that forbid? (Parallel test runs.) |
| 22 | [src/Web/ClientApp-React/vite.config.ts](../../src/Web/ClientApp-React/vite.config.ts) | Dev proxy wired from Aspire-injected env vars ([L4-L6](../../src/Web/ClientApp-React/vite.config.ts#L4-L6)) — how the SPA finds the API without hardcoded URLs. |

## Senior path — "what does this design commit us to?"

| # | File | Why / look for |
|---|---|---|
| 23 | [docs/decisions/ADR-001](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md), [ADR-002](../../docs/decisions/ADR-002-Aspire-For-Orchestration-And-Testing.md), [ADR-003](../../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md) | Model ADRs: context → decision → rationale → consequences, including the "harder" column. |
| 24 | [src/Domain/ValueObjects/Colour.cs](../../src/Domain/ValueObjects/Colour.cs) | Value object with a factory (`From`) that validates — but a public constructor ([L3](../../src/Domain/ValueObjects/Colour.cs#L3)) that doesn't. Invariant leak; see [03-architecture-and-patterns/05-pattern-catalog.md](../03-architecture-and-patterns/05-pattern-catalog.md). |
| 25 | [src/Domain/Common/ValueObject.cs](../../src/Domain/Common/ValueObject.cs) | Structural equality by components — the mechanic under `Colour ==`. |
| 26 | [src/ServiceDefaults/Extensions.cs](../../src/ServiceDefaults/Extensions.cs) | OTel logging/metrics/tracing, health endpoints gated to Development ([L101-L118](../../src/ServiceDefaults/Extensions.cs#L101-L118)), default HTTP resilience. |
| 27 | [src/AppHost/Program.cs](../../src/AppHost/Program.cs) | Resource graph, `WaitFor`, external endpoints — orchestration as code. |
| 28 | [src/Web/Infrastructure/ApiExceptionOperationTransformer.cs](../../src/Web/Infrastructure/ApiExceptionOperationTransformer.cs) | The OpenAPI doc kept honest: 400 added everywhere *because the pipeline can throw it everywhere*. Contract and implementation co-designed. |
| 29 | [tests/Web.AcceptanceTests/AspireSetup.cs](../../tests/Web.AcceptanceTests/AspireSetup.cs) + [Features/Login.feature](../../tests/Web.AcceptanceTests/Features/Login.feature) | Full-stack Gherkin/Playwright tests booting the real AppHost. Cost vs confidence tradeoff. |
| 30 | [templates/ca-use-case/.../CleanArchitectureUseCase.cs](../../templates/ca-use-case/FeatureName/Commands/CleanArchitectureUseCase/CleanArchitectureUseCase.cs) | The maintainers' own definition of "a correctly shaped use case" — the pattern they expect contributions to follow. |

Skip-list (legacy/generated/not for study): `web-api-client.ts` (generated), `package-lock.json`, [README-template.md](../../README-template.md) (shipped into generated solutions), `wwwroot/favicon.ico`, karma/polyfill scaffolding in the Angular app.

## Drill

After the junior path, close everything and redraw the system map from memory, then diff against [01-system-map.md](01-system-map.md). **Basic:** layers right. **Solid:** interfaces (`IApplicationDbContext`, `IUser`) on the correct side of the boundary. **Strong:** you can also mark the three places a request can be rejected before reaching a handler (endpoint auth, AuthorizationBehaviour, ValidationBehaviour) without looking.
