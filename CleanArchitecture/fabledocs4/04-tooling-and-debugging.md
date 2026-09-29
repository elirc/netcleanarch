# Tooling & Debugging

How to run this thing, and what to do when it breaks. Commands are marked **verified** (checked in this environment) or **inferred** (from the repo's scripts/README, not run here).

## 1. Run it

Prereqs: the .NET 10 SDK and Node.js. Docker is **not** needed — this app defaults to SQLite, an in-file database ([appsettings.json L2-L4](../../src/Web/appsettings.json#L2-L4)). Verified in this environment: .NET SDK `10.0.302`, Node `v22.16.0`.

**Full stack** (database + API + Angular frontend), orchestrated by Aspire:

```bash
dotnet run --project ./src/AppHost      # inferred (README)
```

An Aspire dashboard opens showing every service, its URL, and its logs. That dashboard is your first stop when something's wrong.

**Just the API** (faster loop when you're not touching the UI):

```bash
dotnet run --project ./src/Web          # inferred — serves http://localhost:5000, opens /scalar
```

`/scalar` is an interactive API reference — you can send requests to every endpoint from the browser. Great for testing an endpoint without the frontend. The [Web-webapi.http](../../src/Web/Web-webapi.http) file does the same from your editor.

**Remember:** in development the database is **dropped and recreated on every run** ([initialiser L44-L46](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46)), then seeded with an admin user (`administrator@localhost` / `Administrator1!`) and a sample list ([L68-L109](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L68-L109)). Your local data is disposable by design. Don't be precious with it.

## 2. Build and test

```bash
dotnet build                                          # build everything
dotnet test ./tests/Domain.UnitTests                  # one project (fast, no DB)
dotnet test ./tests/Application.FunctionalTests        # real DB, slower
dotnet test ./tests/Application.FunctionalTests --filter "Name~ShouldCreateTodoItem"   # one test
```

All **inferred** (standard `dotnet test`). Note: warnings are treated as **errors** in this repo ([Directory.Build.props L5](../../Directory.Build.props#L5)) — a build can fail on something another project would let slide. That's a feature; fix the warning.

## 3. Read a stack trace (the skill juniors skip)

When something throws, you get a stack trace — a list of method calls, *most recent first*. Read it top-down:

- **Line 1** is the exception type and message — often the whole answer (`UnsupportedColourException: Colour "#123456" is unsupported.`).
- **The next lines** are the call chain. Look for the **first line that mentions a file in `src/`** (not a framework namespace) — that's usually *your* code where the problem surfaced.
- Ignore the framework frames (`Microsoft.*`, `System.*`) at first; they're the plumbing, not the leak.

Don't skim past a stack trace in a panic. It is the single most information-dense thing you'll get. Read the message, find your file, go there.

## 4. Set a breakpoint

In VS Code or Visual Studio, click in the gutter left of a line to set a breakpoint, then run with the debugger (F5) instead of `dotnet run`. When execution hits that line it *pauses*, and you can inspect every variable's current value. This beats scattering `Console.WriteLine` — you see everything, not just what you thought to print.

Good first place to try it: put a breakpoint in [UpdateTodoItem.Handle](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L23-L34), trigger an update from `/scalar`, and watch `entity` before and after the mutation. Seeing the tracked entity change in memory *before* `SaveChangesAsync` makes the [unit-of-work idea](02-fundamentals-primers/04-what-a-transaction-is.md) click.

## 5. See the SQL EF generates

The most common data bug is "the query isn't doing what I think." EF can log the actual SQL it runs. In development, lower the EF log level (in `appsettings.Development.json` or config) so `Microsoft.EntityFrameworkCore` logs at `Information`, and the generated SQL appears in the console / Aspire dashboard. Then you can *see* that [GetTodos](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47) produces one query with a join — not the N+1 storm a naive version would. When a query is slow or wrong, look at the SQL before you look at anything else.

## 6. The Aspire dashboard

When you run the full stack, the dashboard shows **structured logs**, **traces** (one request's whole journey across services), and **metrics** — all wired via OpenTelemetry ([ServiceDefaults L44-L71](../../src/ServiceDefaults/Extensions.cs#L44-L71)). Every request is logged on entry ([LoggingBehaviour](../../src/Application/Common/Behaviours/LoggingBehaviour.cs)), and slow ones (>500ms) log a warning ([PerformanceBehaviour L37-L50](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L37-L50)). When you can't reproduce a bug locally, the trace for the failing request is where you look.

## 7. Debug the frontend

In the browser: **F12** opens DevTools. The **Network** tab shows every request the SPA made — the URL, the payload it sent, the status it got back. When the UI misbehaves, check here first: did the request even go out? What did the server actually return? (Because of the dev proxy, the browser talks to the dev server which forwards `/api` to the backend — [vite.config.ts](../../src/Web/ClientApp-React/vite.config.ts#L13-L25).) The **Console** tab shows JS errors and anything `console.error`'d — and note the app swallows some failures there ([Todo.jsx L166](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167)), so a silent bug may only show as a console line.

## A debugging checklist for when you're stuck

1. **Read the error message and stack trace** — actually read them, top-down, find your file.
2. **Reproduce it reliably** — a bug you can't trigger on demand, you can't fix.
3. **Narrow**: is it the browser, the request, the endpoint, the handler, or the database? Check the Network tab, then the logs, then a breakpoint. Halve the search space each step.
4. **Check the SQL** if it's data-related.
5. **Change one thing at a time.** Multiple simultaneous changes make it impossible to know what fixed (or broke) it.
6. **When truly stuck, ask — well.** See [how to ask for help](05-workflow-and-habits.md).

Full worked debugging scenarios are in [fabledocs/05-quality-engineering/03-systematic-debugging](../../fabledocs/05-quality-engineering/03-systematic-debugging.md); [Lab 5](01-guided-labs/lab-05-fix-a-real-bug.md) walks one end to end.
