# C# / .NET / Runtime Deep-Dive — Question Cards

15 cards. Each: what it's really testing, the repo anchor to reference, and the junior→mid→senior gradient. Practice out loud, 90 seconds each.

---

## Q1: "What does `await` actually do — what happens to the thread?"
Round: language. Testing: do you understand async is about *not blocking threads*, not "running in the background."
Repo anchor: [DeleteTodoList handler](../../src/Application/TodoLists/Commands/DeleteTodoList/DeleteTodoList.cs#L16-L27) passing `cancellationToken` to EF.
Junior: "it waits for the task." Mid: "it returns the thread to the pool and resumes the rest of the method as a continuation when the awaited task completes — that's how a server handles thousands of concurrent requests on few threads." Senior: adds when it *doesn't* help (CPU-bound work — async won't free anything), and cancellation: "here every handler forwards the token to EF so a client disconnect aborts the query instead of doing dead work."
Follow-ups: "async vs multithreading?" "what's a continuation?"
Drill: trace the token from an endpoint to EF; note where the endpoint *doesn't* pass one ([02-stack/01, drill 1](../02-stack-and-language-mastery/01-language-runtime-model.md)).

## Q2: "Walk me through the ASP.NET Core request pipeline."
Round: framework. Testing: middleware ordering and the container/pipeline distinction.
Repo anchor: [Program.cs L27-L49](../../src/Web/Program.cs#L27-L49).
Junior: lists some middleware. Mid: "two things — the DI container built before startup, and the middleware pipeline requests flow through in order; order matters because middleware wraps what comes after it." Senior: gives a *consequence* — "exception handling is registered before endpoints so it can catch their exceptions; CORS is early so preflight is answered before rejection" — and names the seam where MediatR takes over.
Follow-ups: "why does `UseExceptionHandler` position matter?"

## Q3: "Show me a sync-over-async bug and when it deadlocks."
Round: language. Testing: real async hazards, not textbook.
Repo anchor: [DispatchDomainEventsInterceptor L19](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L19) `.GetAwaiter().GetResult()`.
Junior: doesn't recognize it. Mid: "blocking on async work; risky." Senior: "it blocks a thread on a task; with a synchronization context (old ASP.NET, UI apps) the continuation needs the captured context which is blocked → deadlock. Here it's on the sync `SavingChanges` path with no sync context in modern ASP.NET Core, so it survives — but I'd flag it." Naming *why it's safe here* is the senior tell.

## Q4: "Explain DI lifetimes and a bug the wrong one causes."
Round: framework. Testing: singleton/scoped/transient + the capture trap.
Repo anchor: [Infrastructure/DI L42, L76](../../src/Infrastructure/DependencyInjection.cs#L42-L76) — scoped DbContext, singleton TimeProvider.
Junior: defines the three. Mid: "scoped = per request; the DbContext is scoped so one per request." Senior: the captured-dependency bug — "a singleton depending on a scoped service captures the first request's instance; e.g. if the audit interceptor (scoped, consuming scoped `IUser`) were registered singleton, every save would stamp the wrong user." Concrete failure = senior.

## Q5: "Why is `IApplicationDbContext` registered so both the interface and concrete type resolve the *same* instance?"
Round: framework. Testing: subtle DI + unit-of-work.
Repo anchor: [Infrastructure/DI L42](../../src/Infrastructure/DependencyInjection.cs#L42) forwarding factory.
Junior: unaware. Mid: "so handlers and interceptors share one DbContext per request." Senior: "without the forwarding registration you'd get two DbContext instances for the two types — changes tracked on one wouldn't save via the other; a classic 'my updates disappear' bug. The unit of work must be one per request."

## Q6: "`IEnumerable` vs `IQueryable` — why does it matter here?"
Round: language/data. Testing: deferred execution, where code runs.
Repo anchor: [GetTodos ProjectTo L43-L47](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47) vs [interceptor's in-memory LINQ L36-L45](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L36-L45).
Junior: "both are collections." Mid: "`IQueryable` builds an expression tree translated to SQL; `IEnumerable` runs in memory — chaining on IQueryable produces one query." Senior: the failure mode — "an un-translatable expression either throws or client-evaluates, turning one query into N; and the same LINQ syntax over the ChangeTracker runs in memory — you must know which world you're in."

## Q7: "How does EF Core know what SQL to emit on `SaveChanges`?"
Round: framework/data. Testing: unit of work + change tracking.
Repo anchor: [UpdateTodoItem handler](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L23-L34) — no UPDATE statement, yet transactional.
Junior: "it saves the object." Mid: "the DbContext tracks loaded entities and diffs them; SaveChanges computes UPDATE/INSERT/DELETE for changed ones in a transaction." Senior: adds `AsNoTracking` for reads (why: skip tracking overhead, avoid accidental writes) and interceptors hooking the save seam ([audit](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs)).

## Q8: "Records for DTOs, classes for entities — why?"
Round: language. Testing: value vs reference semantics, immutability intent.
Repo anchor: [CreateTodoItemCommand record](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs#L6-L11) vs [TodoItem class](../../src/Domain/Entities/TodoItem.cs).
Junior: "records are shorter." Mid: "records give value equality and immutability — right for messages that carry data unchanged; entities are mutable because EF tracks and mutates them by identity." Senior: notes value objects ([Colour](../../src/Domain/ValueObjects/Colour.cs)) hand-roll value equality that records would give free — and why the repo hasn't converted it (the validating factory still needed).

## Q9: "`string?` vs `string`, `unknown` vs `any` — what do they buy you?"
Round: language/types. Testing: types as enforced contracts.
Repo anchor: [TodoItem `Title` string? but `List = null!`](../../src/Domain/Entities/TodoItem.cs#L7-L28); warnings-as-errors ([Directory.Build.props L5](../../Directory.Build.props#L5)).
Junior: "avoid nulls." Mid: "nullable reference types make null-ness a compile-checked contract; `null!` is me overruling it where EF guarantees population." Senior: "each `null!` and each `as`/`any` is a place the checker was overruled — I grep for them to find where the honest contracts end; the React app's `.jsx` seam is exactly such a hole ([types](../02-stack-and-language-mastery/03-type-system-and-contracts.md))."

## Q10: "How would you add created/modified auditing across 40 entities?"
Round: framework/design. Testing: cross-cutting concerns without duplication.
Repo anchor: [AuditableEntityInterceptor](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs).
Junior: "add fields and set them in each handler." Mid: "a base class + a SaveChanges interceptor that stamps anything of that base type — one enforcement point." Senior: adds the owned-entity subtlety (`HasChangedOwnedEntities`, [L57-L64](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L57-L64)) and the downside — "it's invisible to a handler's reader; someone might re-implement it."

## Q11: "What are generics genuinely good for in production code?"
Round: language. Testing: beyond `List<T>`.
Repo anchor: open behaviours — [AddOpenBehavior(typeof(ValidationBehaviour<,>))](../../src/Application/DependencyInjection.cs#L21).
Junior: "reusable collections." Mid: "the pipeline behaviours are open generics — one class handles every request type, closed by DI per request." Senior: adds constraints as contracts (`where TRequest : notnull`) and `static abstract` interface members enabling the [endpoint registration](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28).

## Q12: "Exceptions vs result types for error handling?"
Round: language/design. Testing: nuance, not dogma.
Repo anchor: the repo uses **both** — [Guard/exceptions](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L28) for domain violations, [Result](../../src/Application/Common/Models/Result.cs) for Identity operations.
Junior: picks one. Mid: "exceptions for exceptional/programming-contract violations, results for expected outcomes the caller branches on." Senior: reads the repo's actual split — "`CreateUserAsync` returns `Result` (expected failures like weak password), `UpdateTodoItem` throws `NotFoundException` (a contract violation mapped to 404). Same codebase, chosen per audience — that's the mature answer."

## Q13: "`TimeProvider` vs `DateTime.Now` — why does it matter?"
Round: language/testing. Testing: testability, UTC discipline.
Repo anchor: [AuditableEntityInterceptor uses TimeProvider L44](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L44) (good) vs [GetWeatherForecasts uses DateTime.Now L20](../../src/Application/WeatherForecasts/Queries/GetWeatherForecasts/GetWeatherForecastsQuery.cs#L20) (lazy).
Junior: "they both give the time." Mid: "`TimeProvider` is injectable so tests control the clock; `DateTime.Now` is untestable and local-time — should be UtcNow." Senior: notes the repo contains both patterns and that the tests currently assert audit times with a ±10s tolerance instead of a fake provider — a real improvement ticket.

## Q14: "What does `nameof` buy over a string literal?"
Round: language. Testing: refactor-safety awareness.
Repo anchor: [WebApplicationExtensions L21-L22](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L21-L24) reflecting on `nameof(IEndpointGroup.RoutePrefix)`.
Junior: "same thing." Mid: "`nameof` is checked at compile time and updates on rename — a raw string silently breaks." Senior: connects to blast radius — "the repo leans on names as contracts (method name = operationId); `nameof` keeps the reflection rename-safe, but the operationId coupling is *not* protected — renaming a handler still breaks clients."

## Q15: "How do you keep dependency versions consistent across a big solution?"
Round: tooling. Testing: real-world build hygiene.
Repo anchor: [Directory.Packages.props](../../Directory.Packages.props) (Central Package Management).
Junior: "update each csproj." Mid: "CPM — versions centralized, projects reference versionless; one file to update, which is what Renovate bumps here." Senior: adds the failure it prevents (version skew across projects), and the one gap (`"nswag": "latest"` unpinned in the npm apps).

---

## Meta-drill

Shuffle these 15, draw 5 at random, answer each in 90 seconds out loud, self-score junior/mid/senior against the gradient. Repeat until every card lands at mid or above. The anchors are your safety net — if you blank, "let me think of where I saw this" and reconstruct from the code is itself a strong move.
