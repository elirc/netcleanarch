# Fast Track — One Weekend With This Codebase

Goal: by Sunday night you can run the app, trace two end-to-end flows out loud, have made one safe change with a passing test, and can explain the architecture to another engineer for five minutes without notes.

## 1. Install and run (Saturday morning)

Prerequisites (from [README.md](../README.md#L17-L21)): .NET 10 SDK, Node.js LTS. Docker is **not** required — this repo defaults to SQLite; Docker only matters for the PostgreSQL/SQL Server template variants.

```bash
# From the repo root — starts Aspire, which starts the database, the API, and the Angular dev server
dotnet run --project ./src/AppHost
```

Status: **inferred** from [README.md](../README.md#L61-L68) and [AppHost/Program.cs](../src/AppHost/Program.cs#L25-L46). The environment used to write these docs had .NET SDK 10.0.302 and Node v22.16.0 installed (**verified**), but the app itself was not launched.

What happens when you run it — worth predicting *before* you look:

1. [AppHost/Program.cs](../src/AppHost/Program.cs#L20-L23) registers a SQLite resource, then the Web project ([L25-L34](../src/AppHost/Program.cs#L25-L34)), then the Angular `ClientApp` as a JavaScript app that waits for the API ([L36-L46](../src/AppHost/Program.cs#L36-L46)).
2. The Web app's startup deletes and recreates the database in development — see [ApplicationDbContextInitialiser.cs](../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46) (`EnsureDeletedAsync` then `EnsureCreatedAsync`). Yes, every dev run **wipes the database**. That's a deliberate template choice; production apps use migrations instead. Remember this — it's a favorite interview topic.
3. Seeding creates role `Administrator`, user `administrator@localhost` / `Administrator1!`, and one sample list ([ApplicationDbContextInitialiser.cs](../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L68-L109)).
4. The Angular client's `prestart` script runs NSwag to **generate** `src/app/web-api-client.ts` from the API's OpenAPI document ([ClientApp/package.json](../src/Web/ClientApp/package.json#L4-L14)). The React client does the same ([ClientApp-React/package.json](../src/Web/ClientApp-React/package.json#L27-L35)). The generated client is not committed — if imports of `web-api-client` look broken in your editor, that's why.

Also useful: `/scalar` on the API host is an interactive API reference ([Program.cs](../src/Web/Program.cs#L35-L36)), and [Web-webapi.http](../src/Web/Web-webapi.http) can drive the API from your editor.

## 2. First 10 files to open, in this order (Saturday afternoon)

| # | File | Why |
|---|---|---|
| 1 | [src/Web/Program.cs](../src/Web/Program.cs) | The whole app in 51 lines: service registration, middleware order, endpoint mapping. |
| 2 | [src/Web/Endpoints/TodoLists.cs](../src/Web/Endpoints/TodoLists.cs) | How HTTP becomes a MediatR command/query. Note how thin it is. |
| 3 | [src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs](../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs) | One complete use case: command + handler in one file. |
| 4 | [src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs](../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs) | Validation as a separate, DB-aware class. |
| 5 | [src/Application/DependencyInjection.cs](../src/Application/DependencyInjection.cs#L16-L23) | The pipeline: every request passes through logging → exception → authorization → validation → performance. |
| 6 | [src/Application/Common/Behaviours/AuthorizationBehaviour.cs](../src/Application/Common/Behaviours/AuthorizationBehaviour.cs) | Where `[Authorize]` on a request class is enforced. |
| 7 | [src/Domain/Entities/TodoItem.cs](../src/Domain/Entities/TodoItem.cs) | A domain event raised from a property setter — the repo's one piece of "rich domain" behavior. |
| 8 | [src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs](../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs) | How those events get published — and *when* (before save; think about why that matters). |
| 9 | [src/Infrastructure/DependencyInjection.cs](../src/Infrastructure/DependencyInjection.cs) | DbContext, interceptors, Identity/auth wiring — where abstraction meets implementation. |
| 10 | [src/Web/ClientApp-React/src/components/Todo.jsx](../src/Web/ClientApp-React/src/components/Todo.jsx) | The entire React feature UI: generated clients, optimistic updates, dialog state. |

## 3. Trace two flows aloud (Saturday evening)

Do these with the files open. Speak each step — this is interview rehearsal, not reading.

**Flow A — Read:** browser hits `/todo` → [ProtectedRoute](../src/Web/ClientApp-React/src/components/api-authorization/ProtectedRoute.jsx#L4-L11) checks auth → [Todo.jsx effect](../src/Web/ClientApp-React/src/components/Todo.jsx#L33-L42) calls `listsClient.getTodoLists()` → Vite proxy forwards `/api` to the API ([vite.config.ts](../src/Web/ClientApp-React/vite.config.ts#L4-L25)) → [TodoLists.GetTodoLists](../src/Web/Endpoints/TodoLists.cs#L23-L28) sends `GetTodosQuery` → pipeline behaviours run → [GetTodosQueryHandler](../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L23-L49) projects entities to DTOs *inside the SQL query* via `ProjectTo` → JSON back up.

**Flow B — Write with a side effect:** user checks a todo item → [updateCheckbox](../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167) updates React state *first* (optimistic), then PUTs → [UpdateTodoItem endpoint](../src/Web/Endpoints/TodoItems.cs#L32-L40) checks route id vs body id → [UpdateTodoItemCommandHandler](../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L23-L34) sets `entity.Done = true` → the [Done setter](../src/Domain/Entities/TodoItem.cs#L14-L26) raises `TodoItemCompletedEvent` only on the false→true transition → `SaveChangesAsync` triggers [DispatchDomainEventsInterceptor](../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L30) which publishes to [LogTodoItemCompleted](../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs#L15-L20), and [AuditableEntityInterceptor](../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L36-L54) stamps `LastModifiedBy`/`LastModified`.

Pause-and-predict: what happens if the PUT in Flow B fails? Look at the `catch` on [line 166](../src/Web/ClientApp-React/src/components/Todo.jsx#L166). The checkbox stays checked while the server says otherwise. Hold that thought — it becomes a ticket in [06-contribution-practice](06-contribution-practice/01-good-first-tickets.md).

## 4. One small safe change + one test (Sunday)

Add a new colour to the palette:

1. Add `public static Colour Pink => new("#EC407A");` beside the others in [Colour.cs](../src/Domain/ValueObjects/Colour.cs#L17-L29), and add it to `SupportedColours` ([L48-L60](../src/Domain/ValueObjects/Colour.cs#L48-L60)).
2. Add it to the `Colours` list the API returns in [GetTodos.cs](../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L32-L41).
3. Add a unit test in [ColourTests.cs](../tests/Domain.UnitTests/ValueObjects/ColourTests.cs) asserting `Colour.From("#EC407A")` succeeds.
4. Run the fast tests (**inferred** commands):

```bash
dotnet test ./tests/Domain.UnitTests
dotnet test ./tests/Application.UnitTests
```

Why this change is safe: it follows an existing pattern exactly, touches no contracts, and the UI colour picker renders whatever the API returns ([Todo.jsx](../src/Web/ClientApp-React/src/components/Todo.jsx#L330-L336)) — zero client changes needed. Being able to *say why a change is low-risk* ("small blast radius: additive, follows existing pattern, covered by a test") is a mid-level skill.

## 5. Teach-back (Sunday night)

Explain aloud, as if an interviewer asked *"walk me through a codebase you know well"*, in under 5 minutes:

- The four layers and the direction dependencies point (inward: Web → Application → Domain; Infrastructure implements Application's interfaces).
- How a request travels (endpoint → MediatR pipeline → handler → EF Core → interceptors).
- One deliberate tradeoff you can defend: no repository layer over EF Core, and why ([ADR-001](../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md)).
- One thing you'd flag if this were production code (e.g., all authenticated users see all todo lists — there's no per-user ownership filter anywhere in [GetTodos.cs](../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47)).

Self-grade: **Basic** — you name the layers. **Solid** — you trace a request end to end naming real classes. **Strong** — you also state one tradeoff, one failure mode, and one improvement, unprompted.

## What the fast path skips

Aspire internals and service discovery, the Angular client, the acceptance-test stack (Playwright + Reqnroll), OpenAPI transformers, identity endpoint details, the template conditional system, and all the drills. The other modules cover them.
