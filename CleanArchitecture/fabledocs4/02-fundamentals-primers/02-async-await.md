# Primer 2 — Async / Await

## The concept from scratch

Some work is slow because your code is *waiting on something else* — a database query, a network call, a disk read. During that wait, the CPU has nothing to do for *this* request. The question is: what should the thread do while it waits?

A **thread** is a worker. A web server has a limited pool of them. If a thread just *blocks* — sits and waits for the database — it's a worker standing frozen, unable to serve anyone else. With enough blocked threads, your server stops responding even though the CPU is idle. That's the problem.

**`async`/`await` solves it by making waiting free.** When you `await` a slow operation, the method *pauses* and **returns its thread to the pool** so it can serve other requests. When the slow operation finishes, the runtime schedules the rest of your method to continue on an available thread. You wrote it top-to-bottom like normal code; under the hood, `await` is a "pause here and give the thread back" marker.

Key mental correction: `await` does **not** mean "run this in the background" or "make it faster." The operation takes exactly as long. What changes is that your *thread* isn't held hostage during the wait.

```csharp
// Illustrative
var item = await _context.TodoItems.FindAsync(id);   // pauses here, frees the thread,
entity.Title = request.Title;                        // resumes here when the DB responds
```

A method that uses `await` must be marked `async` and (almost always) returns a `Task` (a "this will finish later, possibly with a value" handle). That's why nearly every method in this codebase returns `Task` — the async-ness ripples all the way up.

## Where this repo uses it

Every handler is `async Task`. Look at [UpdateTodoItem.Handle L23-L34](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L23-L34): it `await`s `FindAsync` and `SaveChangesAsync`. Each `await` is a point where, if the database is slow, the thread goes back to serving other users instead of freezing.

Notice the `CancellationToken cancellationToken` parameter threaded through and passed to EF Core. That's the "stop early" signal: if the client disconnects, the token is cancelled and the database query can abort instead of finishing work nobody wants. Passing the token down is good hygiene you'll see throughout.

One place the repo runs things *in parallel* on purpose: [ValidationBehaviour L19-L21](../../src/Application/Common/Behaviours/ValidationBehaviour.cs#L19-L21) uses `Task.WhenAll` to run all validators at once rather than one-after-another, because they're independent. `await`-ing tasks one at a time is serial; `Task.WhenAll` is concurrent — a useful distinction.

## The two traps

1. **Sync-over-async.** Calling `.Result` or `.Wait()` or `.GetAwaiter().GetResult()` on an async operation *blocks* the thread — throwing away the entire benefit, and in some app types causing a deadlock. There's one in this repo at [DispatchDomainEventsInterceptor L19](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L19); it's survivable there (modern ASP.NET Core), but the pattern is a smell. If you ever feel tempted to write `.Result` to "get the value out," stop — make the method `async` and `await` instead.
2. **Fake async.** A method marked `async` with no `await` inside runs synchronously and warns you. [GetWeatherForecasts](../../src/Application/WeatherForecasts/Queries/GetWeatherForecasts/GetWeatherForecastsQuery.cs#L12-L14) does this deliberately (demo code) with a `#pragma` to silence the warning. Not a pattern to copy.

## The misconception juniors have

*"`async` makes my code faster / run in parallel."* No. A single awaited call is exactly as slow. `async` improves **throughput under load** (more requests served with the same threads), not the latency of one operation. Parallelism (`Task.WhenAll`) is a *separate* thing you opt into for independent work. Conflating "async" with "parallel" leads to bugs.

## Check yourself

1. When you `await _context.SaveChangesAsync(...)`, what happens to the thread during the wait, and why does that matter for a busy server?
2. Why is `.Result` on an async call considered a smell? What benefit does it throw away?
3. `ValidationBehaviour` uses `Task.WhenAll`; a handler `await`s its DB calls one by one. What's the difference, and when is each correct?
