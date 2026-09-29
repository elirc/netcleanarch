# 1. Architecture & Layering

## What it does and why it exists

Four projects, one dependency rule. `Domain` at the center, `Application` around it, then `Infrastructure` and `Web` on the outside. Source-code dependencies point *inward*: `Web` knows everything, `Application` knows `Domain`, `Domain` knows (almost) nothing. `Infrastructure` implements interfaces that `Application` defines.

The whole point is **isolating the stuff that changes slowly (business rules) from the stuff that changes fast (frameworks, databases, transport)**. When Microsoft renames an ASP.NET API or we swap SQLite for Postgres, I want that blast radius contained to the outer ring. The composition root — the one place all of this gets wired together — is [src/Web/Program.cs L7-L12](../src/Web/Program.cs#L7-L12):

```
builder.AddApplicationServices();
builder.AddInfrastructureServices();
builder.AddWebServices();
```

Each of those is an extension method on a different layer's `DependencyInjection.cs`. `Web` is allowed to know about all of them; that's its job. Nobody else is.

The dependency direction is enforced by project references and one discipline: **inner layers define interfaces, outer layers implement them.** The cleanest example in the codebase is [IUser](../src/Application/Common/Interfaces/IUser.cs) — two properties, `Id` and `Roles` — defined in `Application`, implemented by [CurrentUser](../src/Web/Services/CurrentUser.cs) in `Web`, which pulls them out of `HttpContext`. `Application` needs to know *who is acting* without knowing HTTP exists. That's dependency inversion earning its keep in about twenty lines.

## The hardest decisions

### Decision 1: EF Core lives in the Application layer (no repository pattern)

This is the one people argue with me about, so I wrote an ADR for it: [docs/decisions/ADR-001](../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md). Read it — it's the best example in the repo of how to argue a tradeoff.

The Application layer defines [IApplicationDbContext](../src/Application/Common/Interfaces/IApplicationDbContext.cs), which exposes `DbSet<TodoList>` and `DbSet<TodoItem>` directly. Handlers query EF Core through it — LINQ, `ProjectTo`, the works. There is **no repository layer**.

The rejected alternative was the textbook one: define `ITodoListRepository` in `Application`, implement it in `Infrastructure`, keep EF out of the inner ring entirely. I rejected it because in practice repositories don't *remove* the coupling to EF — they *hide* it. You still end up with methods like `GetActiveListsOrderedByTitleWithItems`, which is an EF query wearing a trench coat. You've added a layer of indirection and a maintenance burden, and you've gained the *illusion* of portability you'll never cash in.

**The tradeoff that still bothers me:** this makes the Application layer harder to unit-test in isolation — you can't just mock a repository, you need a real database (which is why our testing story is what it is; see [section 4](04-testing-strategy.md)). And if we ever *did* need to swap ORMs, it'd be an Application-layer rewrite, not an Infrastructure swap. I made peace with that because greenfield projects essentially never replace their ORM. But if you're building a genuine DDD system with rich aggregates and invariants that must load as a unit, the repository pattern is the right call — for *different* reasons than "hiding EF." The ADR says this explicitly. Don't cargo-cult my decision into a project where it doesn't fit.

### Decision 2: `Domain` has exactly one NuGet dependency, and I'm okay with that

The ideal is "the Domain layer references nothing." We violated it on purpose: `Domain` references `MediatR.Contracts` so that [BaseEvent](../src/Domain/Common/BaseEvent.cs) can implement `INotification` directly. The alternative was to define our own `IDomainEvent` marker in `Domain` and write an adapter in `Application` to convert it to MediatR's `INotification` before publishing.

I rejected the adapter because it exists *solely* to preserve a purity ideal, and it papers over a seam between two interfaces with identical semantics. `MediatR.Contracts` is interface-definitions-only, no transitive dependencies, stable across major versions. The coupling is real but minimal. Full reasoning in [ADR-003](../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md).

**The wart:** there's a *second*, quieter leak I didn't write an ADR for. [BaseEntity](../src/Domain/Common/BaseEntity.cs#L13-L14) carries a `[NotMapped]` attribute — that's a *persistence* concern (`System.ComponentModel.DataAnnotations.Schema`) sitting in the Domain layer. It's cosmetic, the alternative (configuring `Ignore` in Infrastructure) is purer, and I never got around to it. It's the kind of thing that's beneath the threshold of "worth a PR" forever, which is exactly how small leaks become permanent. I'm pointing at it so you learn to *see* them.

### Decision 3: The Users endpoint reaches straight into Infrastructure

Look at [src/Web/Endpoints/Users.cs L1-L19](../src/Web/Endpoints/Users.cs#L1-L19). It imports `CleanArchitecture.Infrastructure.Identity` and takes a `SignInManager<ApplicationUser>` directly. There is no Application-layer use case between the HTTP endpoint and the Identity framework. Every *other* endpoint goes endpoint → MediatR command → handler. This one doesn't.

I did that because wrapping `MapIdentityApi` and the sign-in flow in our CQRS ceremony would have been pure overhead — a command and handler that do nothing but forward to `SignInManager`. The alternative (a `LogoutCommand`, an `IIdentityService.SignOut`) buys architectural consistency and nothing else.

**The tradeoff:** it's an inconsistency you now have to *remember*. A new engineer reads the todo endpoints, learns "endpoints are thin, logic is in handlers," then hits `Users.cs` and gets confused — or worse, copies its shortcut into a place that *should* have a use case. Consistency has value precisely because it lets you stop thinking; every documented exception spends some of that value. I decided the ceremony was worse than the exception. On a bigger team I might have decided the opposite.

## Evidence to open

- [src/Web/Program.cs L7-L12](../src/Web/Program.cs#L7-L12) — the composition root.
- [src/Application/Common/Interfaces/IUser.cs](../src/Application/Common/Interfaces/IUser.cs) + [src/Web/Services/CurrentUser.cs](../src/Web/Services/CurrentUser.cs) — dependency inversion in miniature.
- [src/Application/Common/Interfaces/IApplicationDbContext.cs](../src/Application/Common/Interfaces/IApplicationDbContext.cs) — the anti-repository decision, made visible.
- [docs/decisions/](../docs/decisions/README.md) — three ADRs. Every one has a "Consequences → Harder" section. A design doc with no downsides listed is a sales pitch; these aren't.
- [src/Web/Endpoints/Users.cs L1-L19](../src/Web/Endpoints/Users.cs#L1-L19) — the documented boundary exception.

## The incident this prevented

Early on, before I locked down the dependency direction with project references, someone (fine — me) added a reference from `Application` to `Infrastructure` to "just quickly" use a concrete Identity type in a handler. It compiled. It worked. Two weeks later a colleague tried to write a unit test for that handler and discovered it now transitively dragged in the entire Identity + EF Core + ASP.NET stack, because the dependency graph had quietly inverted. The test project wouldn't even construct the handler without a database.

The fix was to define [IIdentityService](../src/Application/Common/Interfaces/IIdentityService.cs) in `Application` and push the concrete [IdentityService](../src/Infrastructure/Identity/IdentityService.cs) out to `Infrastructure`. That's why the interface exists in the shape it does. The lesson I want you to take: **the dependency rule doesn't announce itself when you break it. The compiler stays green. You find out weeks later when something that should be easy — like a unit test — is suddenly impossible.** The project references are load-bearing guardrails, not bureaucracy. If you ever find yourself adding a reference from an inner project to an outer one, stop and come find me.

## Three questions before you move on

1. `Application` references `Microsoft.EntityFrameworkCore` and yet we claim the dependency rule holds. Explain why a *reference to an assembly* is not the same thing as a *dependency on an implementation*, and why that distinction is the entire justification for ADR-001.
2. If I asked you to add "send a welcome email when a user registers," which layer owns the `IEmailSender` interface, which layer owns the implementation, and why must it *not* go on `IApplicationDbContext`?
3. Find one more boundary leak in the Domain or Application layer that I didn't mention in this section. (Hint: grep the Domain project for framework namespaces, and think about what `Guard.Against` drags in.)
