# Key Flows

Six end-to-end traces. Later modules reuse these by number — Flow 2 feeds the validation drills, Flow 4 the reliability module, Flow 6 the testing module. Read with the files open; answer each pause-and-predict before scrolling on.

---

## Flow 1: Load the Todo page (UI → API → DB read)

Why this flow matters: it's the repo's canonical read path and shows the whole stack — client codegen, proxying, group authorization, the MediatR pipeline, and DB-side projection.

Open these files first:
- [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx#L33-L42) — the fetch effect
- [TodoLists.cs](../../src/Web/Endpoints/TodoLists.cs#L13-L28) — endpoint
- [GetTodos.cs](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs) — query + handler

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
|---|---|---|---|---|---|
| 1 | React | [ProtectedRoute.jsx L4-L11](../../src/Web/ClientApp-React/src/components/api-authorization/ProtectedRoute.jsx#L4-L11) | Redirect to `/login` unless `AuthContext` confirmed a session | — | Client-side only; not a security boundary |
| 2 | React | [Todo.jsx L33-L42](../../src/Web/ClientApp-React/src/components/Todo.jsx#L33-L42) | `listsClient.getTodoLists()` on mount (generated NSwag client, `fetch`, cookies included) | — | `.catch(console.error)` — user sees eternal "Loading…" on failure |
| 3 | Vite | [vite.config.ts L13-L25](../../src/Web/ClientApp-React/vite.config.ts#L13-L25) | `/api` proxied to the backend | HTTP GET `/api/TodoLists` | — |
| 4 | Web | [TodoLists.cs L13-L15](../../src/Web/Endpoints/TodoLists.cs#L13-L15) | Group `RequireAuthorization()` → cookie principal required; then `GetTodoLists` sends `GetTodosQuery` | `GetTodosQuery` (empty record) | — |
| 5 | Application | [DependencyInjection.cs L16-L23](../../src/Application/DependencyInjection.cs#L16-L23) | Pipeline: Logging → UnhandledException → Authorization → Validation → Performance | — | Order is load-bearing |
| 6 | Application | [AuthorizationBehaviour.cs L24-L32](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L24-L32) | `[Authorize]` on the query record ([GetTodos.cs L9](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L9)) → requires `IUser.Id` | — | Second, defense-in-depth check |
| 7 | Application | [GetTodos.cs L43-L47](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47) | `TodoLists.AsNoTracking().ProjectTo<TodoListDto>().OrderBy(Title).ToListAsync` — mapping happens **in SQL**, items included via the AutoMapper profile | `TodosVm { PriorityLevels, Colours, Lists }` | **No user filter — every user gets every list** |
| 8 | React | [Todo.jsx L34-L41](../../src/Web/ClientApp-React/src/components/Todo.jsx#L34-L41) | State set: lists, priorities, colours; first list auto-selected | JS objects from generated DTO classes | — |

Validation and authorization: no validator exists for `GetTodosQuery` (nothing to validate); authorization happens twice — endpoint group ([L13](../../src/Web/Endpoints/TodoLists.cs#L13)) and behaviour ([L9](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L9)).

Persistence and side effects: read-only; `AsNoTracking` skips change tracking (cheaper reads).

Tests that cover it: [GetTodosTests.cs](../../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs) — including `ShouldDenyAnonymousUser` ([L50-L58](../../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L50-L58)) which proves the behaviour-level check works without any HTTP involved.

What juniors usually miss: `ProjectTo` ≠ "map after fetching." It rewrites the SQL so only DTO columns are selected — the difference between one lean query and loading whole entity graphs.

What seniors notice: the missing ownership filter (isolation gap); lookup data (colours/priorities) bundled into the VM to save round trips — a *contract* decision favoring this UI; `OrderBy` applied to lists but **not** to items within a list, so item order is database-arbitrary.

Interview angle: "Walk me through what happens when a page loads data in your app" — this flow *is* the answer. Mid-level answers name the boundary crossings; senior answers add the isolation gap unprompted.

Drill: predict what a logged-in user sees if another user created a list called "Private". Verify by reading [GetTodos.cs L43-L47](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47). Self-grade — Basic: they see it. Solid: you name the fix (filter by `CreatedBy`, which [AuditableEntityInterceptor](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L45-L49) already populates). Strong: you note the fix must apply to *every* command handler too, and propose an EF global query filter to make it structural.

---

## Flow 2: Create a Todo List (write + async validation + error round trip)

Why this flow matters: the repo's richest validation story — including a DB-backed uniqueness rule and a client that actually parses the 400 response.

Open these files first:
- [Todo.jsx L69-L84](../../src/Web/ClientApp-React/src/components/Todo.jsx#L69-L84) — `commitNewList`
- [CreateTodoListCommandValidator.cs](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs)
- [ProblemDetailsExceptionHandler.cs](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L23)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
|---|---|---|---|---|---|
| 1 | React | [Todo.jsx L69-L76](../../src/Web/ClientApp-React/src/components/Todo.jsx#L69-L76) | POST `{ title, colour }`; on success, list added to state locally (not re-fetched) | `CreateTodoListCommand` JSON | Local echo can drift from server truth |
| 2 | Web | [TodoLists.cs L32-L37](../../src/Web/Endpoints/TodoLists.cs#L32-L37) | Bind body → `sender.Send` → `201 Created` with id | `Created<int>` | Location header `/TodoLists/{id}` doesn't match real route `/api/TodoLists` |
| 3 | Application | [ValidationBehaviour.cs L15-L33](../../src/Application/Common/Behaviours/ValidationBehaviour.cs#L15-L33) | All validators run; failures aggregated; throw `ValidationException` | `IDictionary<string, string[]>` | — |
| 4 | Application | [CreateTodoListCommandValidator.cs L13-L25](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L13-L25) | NotEmpty, MaxLength(200), `MustAsync(BeUniqueTitle)` → queries the DB | — | **TOCTOU race**: two simultaneous creates with the same title both pass; no unique DB index backs the rule |
| 5 | Application | [CreateTodoList.cs L23-L36](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs#L23-L36) | `Colour.From(request.Colour ?? Grey)` — domain invariant enforced; entity added; save | `TodoList` | `Colour.From` throws `UnsupportedColourException` → unmapped → 500 |
| 6 | Infrastructure | [AuditableEntityInterceptor.cs L40-L52](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L40-L52) | `CreatedBy`/`Created` stamped from `IUser` + `TimeProvider` | — | — |
| 7 | Web (error path) | [ProblemDetailsExceptionHandler.cs L19-L23](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L19-L23) | `ValidationException` → 400 `ValidationProblemDetails` with the errors dictionary | RFC 9110 problem JSON | — |
| 8 | React (error path) | [Todo.jsx L77-L83](../../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83) | Parses `errors.Title[0]` out of the response and shows it inline | — | Coupled to ProblemDetails shape and PascalCase key |

Validation and authorization: three layers — client `maxLength={200}` on the input ([Todo.jsx L327](../../src/Web/ClientApp-React/src/components/Todo.jsx#L323-L327)), FluentValidation (authoritative), and the domain's `Colour.From` (invariant of last resort). Authorization: group-level only; `CreateTodoListCommand` has **no** `[Authorize]` attribute — pause-and-predict: what does that mean for the functional tests, which bypass HTTP? (Answer: [CreateTodoItemTests.ShouldRequireMinimumFields](../../tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs#L10-L16) runs unauthenticated and still reaches validation.)

Tests that cover it: [CreateTodoListTests.cs](../../tests/Application.FunctionalTests/TodoLists/Commands/CreateTodoListTests.cs) — includes a uniqueness test. No test covers the *race*.

What juniors usually miss: validators can be async and hit the DB — validation isn't only shape-checking.

What seniors notice: the uniqueness rule without a unique index is advisory, not an invariant; the real invariant would be a DB constraint plus handling of the violation exception. Also the create Location header/`nameof` mismatch — a cosmetic contract bug ([TodoLists.cs L36](../../src/Web/Endpoints/TodoLists.cs#L36)).

Interview angle: "Where should validation live?" Mid answer: every layer validates for a different audience (UX / contract / invariant), with the server as the authority. Strong answer uses this flow's three layers plus the race as the punchline.

Drill: write down what HTTP status and body you get for (a) empty title, (b) duplicate title, (c) colour `"#123456"`. Then verify: (a) and (b) → 400 with errors dict; (c) → 500, because `UnsupportedColourException` isn't in the handler's switch. Strong: propose the one-line-ish fix (map it to 400, or validate colour in the validator).

---

## Flow 3: Login and the authorization chain (auth/security boundary)

Why this flow matters: authentication is framework-provided (`MapIdentityApi`), so the learning is in *where the boundaries sit* and how the client cooperates.

Open these files first:
- [Users.cs](../../src/Web/Endpoints/Users.cs) — identity endpoints + logout
- [Infrastructure/DependencyInjection.cs L58-L74](../../src/Infrastructure/DependencyInjection.cs#L58-L74) — cookie auth wiring
- [AuthContext.jsx](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx) — client session state

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
|---|---|---|---|---|---|
| 1 | React | [LoginPage.jsx L13-L22](../../src/Web/ClientApp-React/src/components/api-authorization/LoginPage.jsx#L13-L22) | Submit → `login(email, password)`; on failure just `aria-invalid` styling | — | — |
| 2 | React | [AuthContext.jsx L19-L21](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx#L19-L21) | `client.login(true, undefined, LoginRequest)` — the `true` is `?useCookies=true` | POST `/api/Users/login` | Bearer mode exists for API-only variant |
| 3 | Web | [Users.cs L12](../../src/Web/Endpoints/Users.cs#L12) | `MapIdentityApi<ApplicationUser>()` — register/login/refresh/2fa/manage endpoints, framework-generated | Set-Cookie `.AspNetCore.Identity.Application` | No rate limiting / lockout tuning here |
| 4 | Browser | — | Cookie stored; sent automatically on subsequent same-origin (proxied) requests | — | CSRF surface — see security module |
| 5 | React | [AuthContext.jsx L12-L17](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx#L12-L17) | On app mount: `infoGET()` (`/api/Users/manage/info`) probes whether the cookie is alive | boolean auth state | Extra round trip on every load |
| 6 | Web | [CurrentUser.cs L16-L17](../../src/Web/Services/CurrentUser.cs#L16-L17) | Per-request: claims → `IUser.Id` / `IUser.Roles` | claims principal | — |
| 7 | Application | [AuthorizationBehaviour.cs L24-L74](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L24-L74) | `[Authorize]` request records checked: authenticated? roles? policies (via [IdentityService.AuthorizeAsync](../../src/Infrastructure/Identity/IdentityService.cs#L52-L66))? | — | Role string split on `,` without trimming ([L41-L45](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L41-L45)) — `"Admin, User"` with a space silently never matches. Investigate-level flag |
| 8 | React | [Users.cs L19-L28](../../src/Web/Endpoints/Users.cs#L19-L28) + [AuthContext.jsx L26-L28](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx#L26-L28) | Logout: POST with an empty `{}` body (the `[FromBody] object empty` quirk prevents CSRF-ish GET logout) | — | — |

Validation and authorization: password rules etc. live inside Identity defaults. The important structure: **endpoint-level** `RequireAuthorization()` (returns 401 before any handler runs) plus **application-level** `AuthorizeAttribute` (works even when requests arrive without HTTP — tests, future message consumers). The Angular client adds a third piece: an interceptor that redirects to `/login` on any 401 ([authorize.interceptor.ts L13-L26](../../src/Web/ClientApp/src/api-authorization/authorize.interceptor.ts#L13-L26)).

Tests that cover it: [Login.feature](../../tests/Web.AcceptanceTests/Features/Login.feature) driven by [LoginStepDefinitions.cs](../../tests/Web.AcceptanceTests/StepDefinitions/LoginStepDefinitions.cs) (Playwright, real browser); `ShouldDenyAnonymousUser` in [GetTodosTests.cs](../../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L50-L58).

What juniors usually miss: `ProtectedRoute` is UX, not security — anyone can call the API directly; the server-side checks are the boundary.

What seniors notice: defense in depth is real here but *authorization is coarse* — it answers "are you someone?" never "is this yours?" (no resource-level checks anywhere). Object-level authorization failures (IDOR) are the #1 API vulnerability class; this repo is a clean specimen to discuss.

Interview angle: "Explain authn vs authz in your app" and "how would you prevent users seeing each other's data?" — this flow plus Flow 1's gap is a complete, evidence-based answer.

Drill: list every check a request to `PUT /api/TodoItems/5` passes before the handler mutates data. Solid: endpoint auth, id-match guard ([TodoItems.cs L34-L35](../../src/Web/Endpoints/TodoItems.cs#L34-L35)), validator ([UpdateTodoItemCommandValidator](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItemCommandValidator.cs)), `Guard.Against.NotFound`. Strong: you notice none of them check *ownership*.

---

## Flow 4: Completing a todo item — domain events and side effects

Why this flow matters: the repo's only asynchronous-ish side-effect machinery. Everything you'll ever say in an interview about events, consistency, and outboxes can be grounded here.

Open these files first:
- [TodoItem.cs L13-L26](../../src/Domain/Entities/TodoItem.cs#L13-L26) — event raised in the setter
- [DispatchDomainEventsInterceptor.cs](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs)
- [LogTodoItemCompleted.cs](../../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
|---|---|---|---|---|---|
| 1 | Application | [UpdateTodoItem.cs L30-L31](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L30-L31) | Handler sets `entity.Done = request.Done` | — | — |
| 2 | Domain | [TodoItem.cs L17-L24](../../src/Domain/Entities/TodoItem.cs#L17-L24) | Setter raises `TodoItemCompletedEvent` **only on false→true** — an invariant encoded in the model, impossible to forget at call sites | event in `BaseEntity._domainEvents` | — |
| 3 | Infrastructure | [DispatchDomainEventsInterceptor.cs L25-L30](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L30) | On `SavingChangesAsync` — i.e. **before** the DB write — events are collected, cleared, and published via MediatR | `INotification` | Handlers run before commit: if the save then fails, side effects already happened. If a handler throws, the save never happens |
| 4 | Application | [LogTodoItemCompleted.cs L15-L20](../../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs#L15-L20) | Handler logs. (In real apps: send email, enqueue job, update projection) | — | In-process, synchronous, lost on crash |

Persistence and side effects: publishing before save means a handler that *writes to the same DbContext* gets its changes committed atomically with the original change — that's the upside. The downside is the non-DB side effect problem above. The industry answer is the **transactional outbox** (persist events in the same transaction, deliver from a background worker) — designed as a senior project in [06-contribution-practice/03-senior-build-projects.md](../06-contribution-practice/03-senior-build-projects.md).

Also note [line 19](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L19): the synchronous `SavingChanges` path calls `.GetAwaiter().GetResult()` — sync-over-async, a deadlock-classic worth recognizing (harmless here, dangerous in old ASP.NET with a synchronization context).

Tests that cover it: none directly assert the event fires. Evidence: no test references `TodoItemCompletedEvent` outside Domain — a genuine coverage gap and a ready-made junior ticket.

What juniors usually miss: the event comes from a *property setter*, so every code path that sets `Done` gets the behavior for free — including seed data and tests.

What seniors notice: `_mediator.Publish` awaits all handlers inline — "domain events" here are synchronous method calls wearing event clothing; latency and failure of handlers land on the user's request.

Interview angle: "How would you send an email when X happens, reliably?" — narrate this flow, then upgrade it: outbox table → background dispatcher → idempotent consumer. That progression *is* the mid-to-senior gradient.

Drill: predict what happens if `LogTodoItemCompleted` threw. Trace it: exception propagates through `Publish` → interceptor → `SaveChangesAsync` → `UnhandledExceptionBehaviour` logs ([L15-L28](../../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L15-L28)) → 500. The item is **not** saved. Basic: "error." Solid: exact path. Strong: you can argue whether that coupling is desirable (consistency!) or not (fragility!), and when each answer is right.

---

## Flow 5: Application startup and database lifecycle

Why this flow matters: startup is where orchestration, configuration, and schema strategy meet — and where this template diverges most sharply from production practice.

Open these files first:
- [AppHost/Program.cs](../../src/AppHost/Program.cs)
- [Web/Program.cs](../../src/Web/Program.cs)
- [ApplicationDbContextInitialiser.cs](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs)

Trace:

| Step | Owner | File | What happens | Risk |
|---|---|---|---|---|
| 1 | AppHost | [Program.cs L20-L23](../../src/AppHost/Program.cs#L20-L23) | SQLite resource registered (containers for other DB variants) | — |
| 2 | AppHost | [Program.cs L25-L34](../../src/AppHost/Program.cs#L25-L34) | Web project referenced with `.WithReference(databaseServer).WaitFor(databaseServer)` — connection string injected, ordering guaranteed | — |
| 3 | Web | [Program.cs L7-L12](../../src/Web/Program.cs#L7-L12) | Service registration: ServiceDefaults → KeyVault (if configured) → Application → Infrastructure → Web | Order within builder is mostly free; the *pipeline* order below is not |
| 4 | Web | [Program.cs L17-L20](../../src/Web/Program.cs#L17-L20) | **Development only:** `InitialiseDatabaseAsync()` | — |
| 5 | Infrastructure | [ApplicationDbContextInitialiser.cs L44-L46](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46) | `EnsureDeletedAsync` → `EnsureCreatedAsync`: drop and recreate schema from the current model. No migration history | All dev data lost each run; schema drift invisible; **cannot evolve a production DB this way** |
| 6 | Infrastructure | [ApplicationDbContextInitialiser.cs L68-L109](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L68-L109) | Seed: `Administrator` role, `administrator@localhost` user, one sample list | Hardcoded dev password — fine for a template, a finding anywhere else |
| 7 | Web | [Program.cs L27-L49](../../src/Web/Program.cs#L27-L49) | Pipeline: HTTPS redirect → CORS (**allow any origin/method/header**, [L28-L31](../../src/Web/Program.cs#L28-L31)) → static files → OpenAPI/Scalar → exception handler → health endpoints → `MapEndpoints` → SPA fallback | Wide-open CORS is a red flag to interrogate: harmless-ish for cookie auth (browsers refuse `AllowAnyOrigin` + credentials) but sloppy as a default |

Tests that cover it: every functional test implicitly (the stack boots via [TestAppHost](../../tests/TestAppHost/Program.cs)); acceptance tests boot the full [AppHost](../../tests/Web.AcceptanceTests/AspireSetup.cs#L19-L52).

What juniors usually miss: `MapEndpoints(typeof(Program).Assembly)` ([L45](../../src/Web/Program.cs#L45)) is why new endpoint classes "just work" — reflection over `IEndpointGroup` implementors.

What seniors notice: the missing migration story is the single biggest gap between this template and a deployable system; they'd also ask why `MapOpenApi`/`MapScalarApiReference` aren't gated to Development.

Interview angle: "How do you handle schema changes / zero-downtime deploys?" Answer with the contrast: this repo's `EnsureCreated` vs `dotnet ef migrations` with expand–contract (add column → deploy code reading both → backfill → remove old). See [03-architecture-and-patterns/02-data-model-and-persistence.md](../03-architecture-and-patterns/02-data-model-and-persistence.md).

Drill: you add `public DateTime? DueDate { get; set; }` to `TodoItem`. What must change for (a) this dev setup, (b) a production system with data? Solid: (a) nothing — next run recreates the schema. Strong: (b) a migration, nullable-or-defaulted column, plus the DTO/command/validator/client-regen chain — and you can name that chain from [02-file-reading-order](02-file-reading-order.md).

---

## Flow 6: How a functional test gets its world

Why this flow matters: the test infrastructure *is* the architecture demo — the interfaces the layers define are exactly the seams the tests swap.

Open these files first:
- [FunctionalTestSetup.cs](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs)
- [WebApiFactory.cs](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs)
- [TestApp.cs](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs)

Trace:

| Step | Owner | File | What happens | Risk |
|---|---|---|---|---|
| 1 | NUnit | [FunctionalTestSetup.cs L14-L46](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L14-L46) | Once per run: boot [TestAppHost](../../tests/TestAppHost/Program.cs) (DB only), wait healthy, grab connection string | 60s timeout on slow machines |
| 2 | Test infra | [WebApiFactory.cs L12-L28](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L12-L28) | In-memory `WebApplicationFactory<Program>` pointed at that DB; `IUser` replaced with a Moq stub reading `TestApp` statics | Swapping `IUser` is possible *because* Application depends on the interface, not `HttpContext` |
| 3 | Test infra | [DatabaseResetter.cs L24-L45](../../tests/Application.FunctionalTests/Infrastructure/DatabaseResetter.cs#L24-L45) | Respawn snapshots the schema; [TestBase.SetUp](../../tests/Application.FunctionalTests/Infrastructure/TestBase.cs#L5-L10) resets data before every test | Reset ≠ reseed: tests start from *empty* tables, no admin user, no sample list |
| 4 | Test | [TestApp.RunAsDefaultUserAsync](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L38-L46) | Creates a real Identity user, stores id/roles in statics for the `IUser` mock | Statics forbid parallel tests ([L13-L14](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L13-L14)) |
| 5 | Test | [TestApp.SendAsync](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L16-L23) | Sends the command through the **real** MediatR pipeline — behaviours, validators, interceptors, real DB — but no HTTP | Endpoint code (id-match guards, status codes) is *not* covered here; that's the acceptance tests' job |

What juniors usually miss: these tests don't make HTTP calls. They enter at `ISender`, which is why `ShouldDenyAnonymousUser` proves the *behaviour* rejects, not the endpoint.

What seniors notice: the boundary each test layer exercises is chosen deliberately — unit (pure logic), functional (pipeline+DB), acceptance (browser→everything). Each layer's blind spots are another layer's coverage. That's a testing *strategy*, not a pile of tests.

Interview angle: "How do you test code that touches the database?" — describe Respawn + real DB and *why* mocks were rejected ([ADR-001, testing section](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md)).

Drill: write (on paper) the functional test for "creating a list with a 201-char title fails." Compare with [CreateTodoItemTests.ShouldRequireMinimumFields](../../tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs#L10-L16) for shape. Strong: your test asserts on the `Errors` dictionary key, not just the exception type.
