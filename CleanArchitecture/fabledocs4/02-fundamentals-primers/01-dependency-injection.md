# Primer 1 — Dependency Injection

## The concept from scratch

A class usually needs other objects to do its job. A handler needs a database context. The naive way is to create them yourself:

```csharp
// Illustrative — the naive way, NOT how this repo works
public class CreateTodoItemCommandHandler
{
    private readonly ApplicationDbContext _context = new ApplicationDbContext(...); // who supplies the connection string? ugh
}
```

This is bad for reasons that compound: the handler is now welded to one *concrete* database class, you can't swap it for a test double, and every class that news-up its own dependencies duplicates setup logic everywhere.

**Dependency injection (DI) inverts this.** Instead of a class *creating* what it needs, it *declares* what it needs (usually as constructor parameters) and something else *supplies* them. That "something else" is the **DI container** — a registry that knows how to build each type. You ask the container for a handler; it sees the handler needs an `IApplicationDbContext`, builds one, and hands it over. The class never says `new`.

Two payoffs:
1. **Swappability.** The handler asks for the *interface* `IApplicationDbContext`. In production the container gives it the real EF Core context; in tests, a version pointed at a test database. Same handler, different guts.
2. **One place to configure.** Connection strings, lifetimes, and wiring live in one registration file, not scattered across constructors.

## Where this repo uses it

Look at any handler — [CreateTodoItemCommandHandler](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs#L13-L21):

```csharp
public CreateTodoItemCommandHandler(IApplicationDbContext context)
{
    _context = context;
}
```

It *asks for* `IApplicationDbContext` and never creates one. Who supplies it? The registration in [Infrastructure/DependencyInjection.cs L42](../../src/Infrastructure/DependencyInjection.cs#L42):

```csharp
builder.Services.AddScoped<IApplicationDbContext>(provider => provider.GetRequiredService<ApplicationDbContext>());
```

That line says "when someone needs `IApplicationDbContext`, give them the real `ApplicationDbContext`." Change that one line and every handler in the app gets a different implementation, no handler edited.

The whole app is assembled this way at startup — the three `Add...Services()` calls in [Program.cs L7-L12](../../src/Web/Program.cs#L7-L12) register everything. And the *reason* the [IUser](../../src/Application/Common/Interfaces/IUser.cs) test swap in [Lab 4](../01-guided-labs/lab-04-your-first-tests.md) works ([WebApiFactory L17-L28](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28)) is DI: the test replaces the `IUser` registration with a fake, and every class that depends on `IUser` transparently gets the fake instead. No production code changes.

## Lifetimes (the part that bites)

The container also decides *how long* an object lives. Three options you'll see:

- **Singleton** — one instance for the whole app. Used for stateless things like the clock: `AddSingleton(TimeProvider.System)` ([Infrastructure/DI L76](../../src/Infrastructure/DependencyInjection.cs#L76)).
- **Scoped** — one instance per web request. The database context is scoped ([L42](../../src/Infrastructure/DependencyInjection.cs#L42)) so all the work in one request shares one context (and one transaction). `IUser` is scoped ([Web/DI L15](../../src/Web/DependencyInjection.cs#L15)) because it reads the *current request's* logged-in user.
- **Transient** — a fresh instance every time it's asked for.

The classic bug: a **singleton that depends on a scoped service** accidentally captures the first request's instance and reuses it forever — so, e.g., every request sees the *first* user's identity. Getting lifetimes right is a real skill; when something "remembers" state it shouldn't, suspect a lifetime mismatch.

## The misconception juniors have

*"DI is a framework thing / magic."* It isn't magic and it isn't really about frameworks. The core idea is one sentence: **a class should ask for what it needs, not build it.** The container is just an automated way to satisfy those asks. You could do DI by hand (pass everything into constructors yourself); the container just saves you from wiring a thousand constructors manually.

## Check yourself

1. `CreateTodoItemCommandHandler` never calls `new ApplicationDbContext(...)`. Who actually creates the context it uses, and where is that decided?
2. Why is `IUser` registered as *scoped* and not *singleton*? What would break if it were a singleton?
3. In a test, how does swapping the `IUser` registration for a fake ([WebApiFactory](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28)) change the behaviour of a handler that was never edited?
