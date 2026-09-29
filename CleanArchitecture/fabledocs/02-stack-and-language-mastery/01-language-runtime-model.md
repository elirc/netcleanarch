# C# / .NET Runtime Model — Through This Repo's Code

## 1. async/await: what actually happens

Mental model: `await` does not block a thread. It splits the method at that point, returns the thread to the pool, and registers the rest of the method as a continuation to run when the awaited task finishes. A web server survives thousands of concurrent requests on a few dozen threads because *waiting is free*.

Where this repo uses it well:
- Every handler is `async Task` and passes `CancellationToken` down to EF Core, e.g. [DeleteTodoList.cs L16-L27](../../src/Application/TodoLists/Commands/DeleteTodoList/DeleteTodoList.cs#L16-L27). If the client disconnects, ASP.NET Core cancels the token and the DB query aborts instead of finishing work nobody wants.
- [ValidationBehaviour.cs L19-L21](../../src/Application/Common/Behaviours/ValidationBehaviour.cs#L19-L21) runs all validators concurrently with `Task.WhenAll` — parallel awaiting when tasks are independent, instead of `await`-ing serially in a loop.

Sharp edge in this repo — **sync-over-async**: [DispatchDomainEventsInterceptor.cs L19](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L19) does `DispatchDomainEvents(...).GetAwaiter().GetResult()` in the synchronous `SavingChanges` override. That blocks a thread until async work completes. Here it's survivable (no synchronization context in modern ASP.NET Core, and the sync save path is rarely hit), but the same line in a UI app or legacy ASP.NET deadlocks. Recognize the pattern: `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` on a hot path = smell.

Second sharp edge — **fake async**: [GetWeatherForecastsQuery handler](../../src/Application/WeatherForecasts/Queries/GetWeatherForecasts/GetWeatherForecastsQuery.cs#L12-L14) is `async` with zero awaits (the `#pragma` admits it). Harmless demo code; in review, ask whether the signature is aspirational or accidental.

Pitfall checklist: □ token passed all the way down □ no `.Result`/`.Wait()` □ `Task.WhenAll` for independent work □ no `async void` (none in this repo — event handlers here are MediatR's `Task`-returning kind).

## 2. Dependency injection lifetimes

Three lifetimes: **singleton** (one instance forever), **scoped** (one per request), **transient** (new every resolve). Getting these wrong is a classic production bug (captured DbContext, stale user).

Real examples, all in [Infrastructure/DependencyInjection.cs](../../src/Infrastructure/DependencyInjection.cs):
- `AddSingleton(TimeProvider.System)` ([L76](../../src/Infrastructure/DependencyInjection.cs#L76)) — stateless clock, safe as singleton, mockable in tests.
- `AddScoped<IApplicationDbContext>` forwarding to the same scoped `ApplicationDbContext` ([L42](../../src/Infrastructure/DependencyInjection.cs#L42)) — **one** DbContext per request even when resolved under two different types. Interview-worthy detail: register the forward with a factory lambda, or you'd get two contexts and phantom "my changes aren't saving" bugs.
- `AddScoped<IUser, CurrentUser>` ([Web/DependencyInjection.cs L15](../../src/Web/DependencyInjection.cs#L15)) — must be scoped: it reads the *current request's* claims via `IHttpContextAccessor` ([CurrentUser.cs](../../src/Web/Services/CurrentUser.cs#L16-L17)).
- `AddTransient<IIdentityService, IdentityService>` ([L77](../../src/Infrastructure/DependencyInjection.cs#L77)) — cheap, stateless, no reason to cache.

The lifetime rule that bites: a singleton may not depend on a scoped service (the container throws, or worse, silently captures). Predict-and-verify: could `AuditableEntityInterceptor` (scoped, [L20](../../src/Infrastructure/DependencyInjection.cs#L20)) be a singleton? No — it consumes scoped `IUser`.

## 3. LINQ and `IQueryable`: deferred execution

`IQueryable` builds an *expression tree*; nothing touches the database until a terminal call (`ToListAsync`, `AnyAsync`, `FindAsync`). That's why [GetTodos.cs L43-L47](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47) can chain `AsNoTracking → ProjectTo → OrderBy → ToListAsync` and produce **one** SQL statement, and why the `BeUniqueTitle` check ([CreateTodoListCommandValidator.cs L21-L26](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26)) becomes `SELECT EXISTS(...)` rather than loading rows.

Contrast inside the repo: [DispatchDomainEventsInterceptor.cs L36-L45](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L36-L45) works on the *ChangeTracker* (in-memory, LINQ-to-Objects) — same syntax, completely different execution. Being able to say which world a LINQ query runs in is a mid-level discriminator; it's also where "works in tests, dies in prod" bugs live (client evaluation of un-translatable expressions).

## 4. Records, value semantics, and why commands are records

Commands/queries are `record` types, e.g. [CreateTodoItemCommand](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs#L6-L11) with `{ get; init; }` properties, or positional [DeleteTodoItemCommand(int Id)](../../src/Application/TodoItems/Commands/DeleteTodoItem/DeleteTodoItem.cs#L5). Records give value equality and immutability-by-default — right for *messages*, whose job is to carry data across a boundary unchanged. Entities ([TodoItem.cs](../../src/Domain/Entities/TodoItem.cs)) are mutable classes because EF Core tracks them by identity and mutates them.

The repo also hand-rolls value semantics where records didn't fit: [ValueObject.cs](../../src/Domain/Common/ValueObject.cs#L23-L44) implements `Equals`/`GetHashCode` over `GetEqualityComponents()`. Ask yourself why `Colour` isn't just `record Colour(string Code)` — answer: the base class predates records, records would give this for free, but the *factory validation* (`Colour.From`) would still be needed. Good refactor kata (see [06-contribution-practice/04-refactor-and-design-katas.md](../06-contribution-practice/04-refactor-and-design-katas.md)).

## 5. Nullable reference types as a contract language

`<Nullable>enable</Nullable>` + warnings-as-errors ([Directory.Build.props](../../Directory.Build.props#L5-L12)) makes `string?` vs `string` a compile-checked promise. Read [TodoItem.cs](../../src/Domain/Entities/TodoItem.cs#L5-L28): `Title` is `string?` (DB requires it — [TodoItemConfiguration](../../src/Infrastructure/Data/Configurations/TodoItemConfiguration.cs#L11-L13) says `IsRequired` — but the *object* can exist momentarily without it), while `List` is `TodoList` with `= null!;` — the null-forgiveness operator saying "EF will populate this; trust me." Every `null!` is a place the type system was overruled by a framework reality. Count them in a file during review; each is a small risk acknowledgment.

`Guard.Against.NotFound(request.Id, entity)` ([UpdateTodoItem.cs L28](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L28)) doubles as a *nullability narrowing* point: after it, the compiler knows `entity` is non-null.

## 6. Modern C# features worth naming in interviews

- **`static abstract` interface members**: [IEndpointGroup](../../src/Web/Infrastructure/IEndpointGroup.cs#L16-L18) requires a *static* `Map` on implementors — impossible before C# 11; enables the reflection registration in [WebApplicationExtensions](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L18-L25).
- **Primary constructors**: [Colour(string code)](../../src/Domain/ValueObjects/Colour.cs#L3), [WebApiFactory(string connectionString)](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L10).
- **Collection expressions**: `[..roles]`, `Items = []` ([TestApp.cs L73](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L73), [TodosVm.cs L7-L11](../../src/Application/TodoLists/Queries/GetTodos/TodosVm.cs#L7-L11)).
- **Pattern-matching switch expressions**: the exception→status mapping in [ProblemDetailsExceptionHandler.cs L17-L44](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44) — a tuple-returning switch with type patterns.
- **`TimeProvider`** (.NET 8+): injectable clock, [AuditableEntityInterceptor L44](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L44). Contrast with `DateTime.Now` in [GetWeatherForecastsQuery L20](../../src/Application/WeatherForecasts/Queries/GetWeatherForecasts/GetWeatherForecastsQuery.cs#L20) — the repo contains both the good and the lazy pattern; know which is which and why (testability, UTC-vs-local).

## Drills

1. **Cancellation trace:** follow `cancellationToken` from [TodoItems.CreateTodoItem](../../src/Web/Endpoints/TodoItems.cs#L23-L28)… wait — it isn't there. Minimal APIs *can* inject one; this endpoint doesn't take it, so where does the token MediatR receives come from? Investigate (answer: `sender.Send(command)` without a token passes `default` — cancellation is effectively disabled at the endpoint seam; the handler tokens are real but never fire from client disconnects). Rate: Solid if you found the gap; Strong if you can propose the one-parameter fix and say what it buys.
2. **Lifetime audit:** for each registration in [Web/DependencyInjection.cs](../../src/Web/DependencyInjection.cs#L11-L37), say why its lifetime is correct. Check yourself against §2.
3. **Query or objects?** For each LINQ expression in [TestApp.cs](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs) and [GetTodos.cs](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs), label it SQL-translated or in-memory. One is deliberately tricky: `Enum.GetValues(...).Cast<PriorityLevel>()` ([GetTodos.cs L27-L30](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L27-L30)) never touches the DB.

## Interview angle

1. *"What does `await` free up, and what happens to the rest of the method?"* → [Q1 in 08-interview-prep/01](../08-interview-prep/01-csharp-dotnet-deep-dive.md)
2. *"Explain DI lifetimes and a bug wrong lifetimes cause."* → [Q4](../08-interview-prep/01-csharp-dotnet-deep-dive.md)
3. *"What's the difference between `IEnumerable` and `IQueryable`?"* → [Q6](../08-interview-prep/01-csharp-dotnet-deep-dive.md)
4. *"Why records for DTOs but classes for entities?"* → [Q8](../08-interview-prep/01-csharp-dotnet-deep-dive.md)
5. *"Show me a sync-over-async bug."* → point at the interceptor line above; explain when it deadlocks. → [Q3](../08-interview-prep/01-csharp-dotnet-deep-dive.md)
