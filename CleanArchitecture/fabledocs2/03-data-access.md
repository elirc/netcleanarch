# 3. Data Access

## What it does and why it exists

Persistence is EF Core 10, accessed through [IApplicationDbContext](../src/Application/Common/Interfaces/IApplicationDbContext.cs) (defined in `Application`) and implemented by [ApplicationDbContext](../src/Infrastructure/Data/ApplicationDbContext.cs) (in `Infrastructure`, inheriting `IdentityDbContext` so the Identity tables share the schema). Handlers hold the interface, never the concrete context. The provider — SQLite by default, Postgres or SQL Server via template switches — is chosen in [Infrastructure/DependencyInjection.cs L23-L34](../src/Infrastructure/DependencyInjection.cs#L23-L34).

The design decision behind *all* of it is the one from [section 1](01-architecture.md): no repository layer. Handlers write LINQ directly. A read looks like [GetTodos L43-L47](../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47) — `AsNoTracking().ProjectTo<TodoListDto>().OrderBy(...).ToListAsync()` — one translated SQL query that selects only the DTO columns and pulls the child items in the same statement. A write looks like [UpdateTodoItem L23-L34](../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L23-L34) — load by key, mutate the tracked entity, `SaveChangesAsync`. No UPDATE statement is written by hand; the change tracker computes it. **The `DbContext` is a unit of work**: one `SaveChangesAsync` per handler, and EF wraps it in a transaction automatically. That's why each use case is atomic without anybody writing `BeginTransaction`.

Two things hang off the save seam via interceptors, so no handler has to remember them:
- [AuditableEntityInterceptor](../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs) stamps `Created/CreatedBy/LastModified/LastModifiedBy` on anything deriving from `BaseAuditableEntity`.
- [DispatchDomainEventsInterceptor](../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs) publishes domain events.

## The hardest decisions

### Decision 1: Validators do database work, and that turned uniqueness into a lie

This is the sharpest footgun in the data layer and I want you to understand it cold. Our validators are FluentValidation classes, dependency-injected, and they can hold `IApplicationDbContext`. So [CreateTodoListCommandValidator L21-L26](../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26) enforces unique titles by *querying the database*:

```
return !await _context.TodoLists.AnyAsync(l => l.Title == title, cancellationToken);
```

It reads clean. It's also a **check-then-act race** — TOCTOU (time-of-check to time-of-use). Two requests with the same title arrive together, both run the `AnyAsync` check, both see "no duplicate," both proceed, both insert. Now you have two lists with the same "unique" title, and there's **no unique index** in [TodoListConfiguration](../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs) to stop it. The validator is *advisory*, not enforcing. It gives you a friendly error 99% of the time and silently fails under concurrency.

The rejected alternative was a database unique index with no validator — let the DB be the authority, catch the constraint violation, map it to an error. I didn't do *only* that because a raw constraint violation gives a lousy user experience (a 500 or an opaque DB error instead of "that title's taken"). But I also didn't add the index *alongside* the validator, and that's the actual mistake. **The correct design is both: the index is the invariant, the validator is the UX.** What we shipped is the UX with no invariant behind it.

**Why it still bothers me:** it's a beautiful illustration of a principle — *a validator is advisory, only the database can enforce a cross-row invariant* — and we're on the wrong side of it in our own flagship example. If you take one thing from this section: when you see a "uniqueness" or "no overlap" or "at most N" rule enforced only in application code, assume it's racy until you find the constraint that backs it.

### Decision 2: Dev recreates the database every run; there are no migrations

Open [ApplicationDbContextInitialiser L44-L46](../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46):

```
await _context.Database.EnsureDeletedAsync();
await _context.Database.EnsureCreatedAsync();
```

In development, **every run drops and recreates the schema** from the current model, then reseeds ([L68-L109](../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L68-L109)). There is no migrations folder. There's no migration history table. And — this is the part that makes me wince — we *suppress the pending-model-changes warning* in [Infrastructure/DependencyInjection.cs L33](../src/Infrastructure/DependencyInjection.cs#L33), so EF won't even tell you the model has drifted from what a migration would produce.

For a *template*, this is the right call, and that's the only reason it's here: it makes "clone and run" instant, with zero migration ceremony to learn, and every developer gets a clean, seeded database on demand. The rejected alternative — ship an initial migration and use `MigrateAsync` — adds real friction to the "getting started" experience the template exists to optimize.

**The tradeoff, stated bluntly:** this codebase, as-is, *cannot evolve a production database.* `EnsureCreated` and migrations are mutually exclusive; you can't add migrations later without an annoying reconciliation. The single biggest gap between this template and a deployable system is the migration story, and if you fork this to build something real, **introducing migrations is job zero** — generate the initial migration, switch the initialiser to `MigrateAsync`, and un-suppress that warning so drift becomes visible again. I left a loud comment-shaped hole here on purpose; don't ship it to prod as-is.

### Decision 3: Domain events dispatch *before* the commit

This one is subtle and I go back and forth on it. The [DispatchDomainEventsInterceptor L25-L30](../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L30) publishes events from `SavingChangesAsync` — that is, *before* the rows actually hit the database.

I chose before-save because of one real benefit: if an event handler writes to the *same* `DbContext`, its changes join the same `SaveChanges` and commit atomically with the trigger. No dual-write, no "the event handler's change committed but the original didn't."

The rejected alternative was after-save (`SavedChangesAsync`): DB truth first, then fire events. That inverts the failure mode — the save succeeds, then a handler fails, and you've lost the event.

**The tradeoff that keeps me up:** before-save means a handler's *external* side effect (send an email, call an API) happens *before we know the transaction will commit*. If the save then fails, you've sent a ghost email for a thing that didn't happen. And if a handler *throws*, the save aborts — so today, your logging handler can prevent a todo item from being saved. There's also a sync-over-async landmine on [line 19](../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L19) (`.GetAwaiter().GetResult()` on the synchronous save path) that's harmless in modern ASP.NET Core but would deadlock under a synchronization context. Right now every handler only logs ([LogTodoItemCompleted](../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs)), so none of this hurts. The day someone adds an email-sending handler, the correct answer is a **transactional outbox** — persist the event in the same transaction, deliver it from a background worker with retries, make consumers idempotent. Until then, the machinery *looks* general-purpose but has a sharp edge, and I'd rather you know that than discover it.

## Evidence to open

- [src/Application/Common/Interfaces/IApplicationDbContext.cs](../src/Application/Common/Interfaces/IApplicationDbContext.cs) — three members. That's the whole persistence contract.
- [src/Application/TodoLists/Queries/GetTodos/GetTodos.cs L43-L47](../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47) — projection and `AsNoTracking`; note items come back in DB-arbitrary order (there's an `OrderBy` on lists but not on items).
- [src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs L21-L26](../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26) — the racy uniqueness check.
- [src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs](../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs) — cross-cutting persistence at the save seam. Read `HasChangedOwnedEntities` ([L57-L64](../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L57-L64)) and figure out why it exists.
- [src/Infrastructure/Data/ApplicationDbContextInitialiser.cs L44-L46](../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46) — the drop-and-recreate.

## The incident this caused

A duplicate-detection feature in a downstream app built on this pattern relied on exactly the validator-only uniqueness approach. Under normal load it was flawless — every manual test passed, because a human can't submit the same form twice within the race window. Then a client added a retry-on-timeout to their integration: a request would time out at the load balancer *after* the server had actually received it, the client would retry, and now two identical creates were in flight microseconds apart. Both passed the `AnyAsync` check. Both inserted. The "unique" table grew duplicates, a nightly reconciliation job started failing on the duplicate keys it assumed couldn't exist, and someone got paged at 3 a.m. for a data-integrity alarm.

Root cause: **a uniqueness invariant enforced only by a check-then-act query, with no database constraint behind it** — precisely the shape in `CreateTodoListCommandValidator`. The fix was a unique index (the invariant), keeping the validator for friendly errors (the UX), catching `DbUpdateException` and returning a 409, and — the ugly part — a one-off script to dedupe the rows that had already snuck in *before* the index could be created. That last step is the tax you pay for finding the bug in production instead of in design: you don't just add the constraint, you first have to clean up the data that violated the constraint you didn't have. Design the invariant in from the start.

## Three questions before you move on

1. `UpdateTodoItem` contains no `UPDATE` statement and no explicit `BeginTransaction`, yet it's atomic. Explain the mechanism — what makes it a unit of work, and what actually computes the SQL.
2. Two requests create a list titled "Groceries" at the same instant. Trace both through [CreateTodoListCommandValidator](../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs) and the handler, and tell me the end state of the database. Then tell me the two-part fix.
3. I add `public DateTime? DueDate` to `TodoItem`. In *this* dev setup, what do I have to do to see it in the database? In a production system with real data, what's the full list of steps — and why is the answer different?
