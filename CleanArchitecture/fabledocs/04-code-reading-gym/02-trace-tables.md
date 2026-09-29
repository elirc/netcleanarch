# Trace Tables

Fill each table yourself first (columns: Step, File, Value/shape at this point, Owner, Transformation, Risk). Completed reference tables follow each prompt — fold the page, no peeking.

---

## Trace 1 (UI→API): user renames a list to a duplicate title

Prompt: user opens List Options, types a title that another list already has, clicks Update. Trace until the UI shows… what, exactly?

| Step | File | Value shape | Owner | Transformation | Risk |
|---|---|---|---|---|---|
| 1 | [Todo.jsx L96-L102](../../src/Web/ClientApp-React/src/components/Todo.jsx#L96-L102) | `listOptionsEditor = {id, title, colour}` | React | state → `updateTodoList(id, editor)` call | editor object doubles as command DTO |
| 2 | generated `web-api-client.ts` | JSON body + `id` in URL | NSwag client | JS object → `PUT /api/TodoLists/{id}` | — |
| 3 | [TodoLists.cs L41-L48](../../src/Web/Endpoints/TodoLists.cs#L41-L48) | `UpdateTodoListCommand{Id,Title,Colour}` | Web | JSON → record; id-match guard | mismatch → bare 400, *no ProblemDetails body* — inconsistent error shape |
| 4 | [UpdateTodoListCommandValidator L13-L26](../../src/Application/TodoLists/Commands/UpdateTodoList/UpdateTodoListCommandValidator.cs#L13-L26) | failures: `[{Title: "…must be unique"}]` | Application | async DB query on other rows | race window |
| 5 | [ValidationException](../../src/Application/Common/Exceptions/ValidationException.cs#L13-L19) → [handler L19-L23](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L19-L23) | 400 + `{errors: {Title: [...]}}` | Web | exception → ValidationProblemDetails | — |
| 6 | [Todo.jsx L96-L101](../../src/Web/ClientApp-React/src/components/Todo.jsx#L96-L101) | `catch (e) { console.error(e) }` | React | **nothing** | dialog stays open, no message — unlike the create dialog which parses errors ([L77-L83](../../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83)). UX inconsistency = junior ticket J7 |

The point of this trace: the same validation error gets *rendered* on create and *swallowed* on update. Contract consistency means nothing if consumers differ.

## Trace 2 (persistence): what one `SaveChangesAsync` actually does

Prompt: `UpdateTodoItemCommand{Id=3, Title="x", Done=true}` reaches its handler; item 3 exists, currently not done. Trace the save call only.

| Step | File | What happens | Risk |
|---|---|---|---|
| 1 | [UpdateTodoItem L25-L33](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L25-L33) | `FindAsync` (tracker → DB), mutate Title + Done; setter queues event | — |
| 2 | [DispatchDomainEventsInterceptor L25-L30](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L30) | interceptors fire on `SavingChangesAsync`: events collected, cleared, published; [LogTodoItemCompleted](../../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs) runs **inside** the save | handler exception aborts the save |
| 3 | [AuditableEntityInterceptor L29-L34](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L29-L34) | LastModified/By stamped (entity is Modified) | anonymous ISender callers stamp null |
| 4 | EF Core | tracker diff → single `UPDATE TodoItems SET Title=@p, Done=@p2, LastModified=..., LastModifiedBy=... WHERE Id=3` in an implicit transaction | no concurrency token → blind overwrite |
| 5 | back in handler | `Task` completes; endpoint returns 204 | — |

Both interceptors run because both are registered as `ISaveChangesInterceptor` ([DI L20-L25](../../src/Infrastructure/DependencyInjection.cs#L20-L25)); their relative order follows registration order — audit stamps happen after event handlers ran. Did you catch that? Event handlers observe *pre-audit* timestamps.

## Trace 3 (auth): anonymous curl against a protected endpoint

Prompt: `curl -X POST https://localhost:5001/api/TodoLists -d '{"title":"hack"}'` with no cookie. Where exactly does it die?

| Step | File | What happens | Risk |
|---|---|---|---|
| 1 | Kestrel/middleware | no auth cookie → anonymous principal | — |
| 2 | routing → group | [TodoLists.cs L13](../../src/Web/Endpoints/TodoLists.cs#L13) `RequireAuthorization()` → authorization middleware challenges | dies **here**: 401, cookie scheme |
| 3 | *(never reached)* | handler, pipeline, validator | — |

Now the twin trace that separates mid from junior: same command sent via `ISender` in a functional test with no `RunAsUserAsync`. Steps: pipeline runs → [AuthorizationBehaviour L24-L26](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L24-L26) finds **no `[Authorize]` attribute on `CreateTodoListCommand`** → proceeds → validation → handler executes, `CreatedBy = null`. Two entry points, two different auth outcomes. Write one sentence on why that's acceptable-or-not before reading [03-architecture/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).

## Trace 4 (error): unsupported colour end to end

Prompt: POST create-list with `{"title":"ok","colour":"#123456"}`.

| Step | File | Value shape | Risk |
|---|---|---|---|
| 1 | validator | Title passes; **no colour rule exists** | gap |
| 2 | [CreateTodoList handler L27-L28](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs#L25-L29) | `Colour.From("#123456")` | — |
| 3 | [Colour.From L5-L15](../../src/Domain/ValueObjects/Colour.cs#L5-L15) | throws `UnsupportedColourException("Colour \"#123456\" is unsupported.")` | — |
| 4 | [UnhandledExceptionBehaviour L15-L28](../../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L15-L28) | logs error **with full request payload**, rethrows | payload logging |
| 5 | [ProblemDetailsExceptionHandler L43](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L43) | `_ => (-1, null)` → returns false → default 500 | domain invariant surfaces as server fault |
| 6 | [Todo.jsx L77-L83](../../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83) | JSON.parse of a non-validation body fails → generic "Failed to create list." | graceful by accident |

The UI *can't* trigger this (swatches only) — the API can. Contract question: is a 500 acceptable for un-triggerable-via-UI input? (No: the API is the contract, the UI is one client.)

## Trace 5 (test lifecycle): one functional test, from process start to assertion

Prompt: `dotnet test` runs only `CreateTodoItemTests.ShouldCreateTodoItem`. Order every piece of infrastructure that executes.

| Step | File | What happens |
|---|---|---|
| 1 | [FunctionalTestSetup L14-L46](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L14-L46) | once: TestAppHost boots DB resource → healthy → connection string captured |
| 2 | [WebApiFactory L12-L28](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L12-L28) | once: in-memory server built against that DB; `Program.cs` runs — **including dev DB init/seed** (environment defaults to Development) |
| 3 | [DatabaseResetter.CreateAsync L24-L38](../../tests/Application.FunctionalTests/Infrastructure/DatabaseResetter.cs#L24-L38) | once: Respawn learns the schema |
| 4 | [TestBase.SetUp L5-L10](../../tests/Application.FunctionalTests/Infrastructure/TestBase.cs#L5-L10) | per test: wipe data, clear user statics |
| 5 | [test body L19-L35](../../tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs#L19-L45) | `RunAsDefaultUserAsync` (real Identity rows) → create list → create item — each via real pipeline |
| 6 | [TestApp.FindAsync L93-L101](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L93-L101) | fresh scope + context → read back → Shouldly asserts incl. audit fields |

Self-check question: why does the assertion `item.CreatedBy.ShouldBe(userId)` pass, given the HTTP layer is bypassed? Because the mocked `IUser` feeds [AuditableEntityInterceptor](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L47) the static test user id — the audit path is *fully* exercised minus claims parsing. Knowing precisely which seam a test enters at, and what that leaves untested, is the entire skill of reading test suites.
