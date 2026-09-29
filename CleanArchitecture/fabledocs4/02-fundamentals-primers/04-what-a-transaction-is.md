# Primer 4 — What a Database Transaction Is

## The concept from scratch

Imagine transferring money: subtract $100 from account A, add $100 to account B. If the server crashes *between* those two steps, the money vanishes — subtracted from A, never added to B. You need both steps to happen **together or not at all.**

A **transaction** is the database's guarantee of exactly that: a group of changes that either *all* commit or *all* roll back, with nothing in between ever visible to anyone else. The classic properties are called ACID; the one you'll feel daily is **atomicity** — all-or-nothing.

In EF Core (this app's ORM), transactions are usually invisible because of a pattern called the **unit of work**. Here's the model:

1. You load entities from the database. EF *tracks* them — it remembers their original state.
2. You change them in memory (set a property, add an item, remove one). Nothing has touched the database yet.
3. You call `SaveChangesAsync()`. EF figures out *everything* that changed, writes it all in **one transaction**, and commits.

So the `DbContext` is a unit of work: a batch of pending changes. One `SaveChangesAsync` = one atomic commit. You never write `INSERT`/`UPDATE`/`DELETE` by hand, and you rarely write `BEGIN TRANSACTION` — EF wraps the whole save for you.

## Where this repo uses it

Open [UpdateTodoItem.Handle L23-L34](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L23-L34):

```csharp
var entity = await _context.TodoItems.FindAsync([request.Id], cancellationToken); // load (tracked)
Guard.Against.NotFound(request.Id, entity);
entity.Title = request.Title;   // change in memory — no SQL yet
entity.Done = request.Done;     // still no SQL
await _context.SaveChangesAsync(cancellationToken); // ONE transaction, commits everything
```

There is no `UPDATE` statement and no `BeginTransaction` in that handler, yet it is fully atomic. If the save fails, *none* of the changes persist. That's the unit of work doing its job. **Every handler in this codebase makes exactly one `SaveChangesAsync` call**, which is why each use case is atomic by construction — a deliberate design choice.

The same context being shared across a request (because it's registered *scoped* — see the [DI primer](01-dependency-injection.md)) is what makes this coherent: everything in one request participates in one unit of work.

There's a subtlety worth meeting now: domain events are dispatched *inside* the save ([DispatchDomainEventsInterceptor L25-L30](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L30)), *before* the commit completes. If an event handler writes to the same context, its writes join the same transaction — atomic. But if it does something external (send an email) and the save then fails, the email already went out. That's the "dual-write problem," and the industry answer is a *transactional outbox*. You don't need to build one now; just know the phrase and why it exists.

## Where it *doesn't* protect you

A transaction guarantees atomicity *within one `SaveChanges`*. It does **not** protect against two separate operations racing. The uniqueness check in [CreateTodoListCommandValidator L21-L26](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26) reads the database (does this title exist?) and *then*, in a separate step, the handler writes. Two requests can both pass the read before either writes — and you get two "unique" titles, because no database constraint enforces it. That's a **check-then-act race** (TOCTOU), and the fix is a real database unique index, not a bigger transaction. Transactions make a single save atomic; they don't serialize independent requests.

## The misconception juniors have

*"I need to write `BeginTransaction` to be safe."* Usually not — one `SaveChangesAsync` is already a transaction. The mistakes that actually hurt are the opposite: (a) making *two* `SaveChangesAsync` calls in one handler, splitting what should be atomic into two commits with a gap between them; and (b) assuming a transaction protects a *read-then-write across requests*, which it doesn't.

## Check yourself

1. `UpdateTodoItem` has no `UPDATE` statement and no `BeginTransaction`, yet it's atomic. Walk through *why* — what makes it a unit of work?
2. What would go wrong if a handler called `SaveChangesAsync` twice — once after adding a list, once after adding its items?
3. The unique-title check reads then writes in two steps. Why can a transaction not fix the race, and what actually does?
