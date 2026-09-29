# ADR-001: Four-Layer Clean Architecture with Dependency Inversion

**Status:** Accepted (implicit; it is the organizing premise of the whole solution).

## Context

An enterprise-style application needs to keep business rules insulated from the things that churn fastest — web frameworks, ORMs, database providers, transport. If domain logic depends directly on those, every framework upgrade or infrastructure swap ripples inward, and the code that should be the most stable becomes the most fragile.

## Decision

Organize the solution into four projects with a strict inward dependency rule: `Domain` at the center, `Application` around it, `Infrastructure` and `Web` on the outside. Source-code dependencies point inward only. Inner layers **define the interfaces they need**; outer layers **implement them**. The single place everything is wired together is the composition root, [src/Web/Program.cs L7-L12](../src/Web/Program.cs#L7-L12) (`AddApplicationServices` / `AddInfrastructureServices` / `AddWebServices`).

The dependency inversion is visible in three interfaces the Application layer owns and Infrastructure/Web implement:
- [IUser](../src/Application/Common/Interfaces/IUser.cs) (two properties) ← [CurrentUser](../src/Web/Services/CurrentUser.cs), which reads `HttpContext`.
- [IApplicationDbContext](../src/Application/Common/Interfaces/IApplicationDbContext.cs) ← [ApplicationDbContext](../src/Infrastructure/Data/ApplicationDbContext.cs).
- [IIdentityService](../src/Application/Common/Interfaces/IIdentityService.cs) ← [IdentityService](../src/Infrastructure/Identity/IdentityService.cs).

## Alternatives Considered

- **A single project / conventional N-tier (UI → BLL → DAL) with dependencies pointing "down" toward the database.** Simpler to start, but couples business logic to persistence and makes the domain the *least* stable layer. Rejected for an enterprise template whose whole selling point is insulation.
- **A shared "Core" library with no enforced direction.** Nothing stops inner code from referencing outer code; the boundary erodes silently. Rejected — the discipline has to be enforced by project references, not good intentions.

## Consequences

**Positive**
- Framework/provider changes stay in the outer ring. The database provider is chosen entirely in [Infrastructure/DependencyInjection.cs L23-L34](../src/Infrastructure/DependencyInjection.cs#L23-L34); nothing in Domain or Application knows which one runs.
- The interfaces double as test seams — functional tests swap only `IUser` and get everything else real ([WebApiFactory L17-L28](../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28)).
- New engineers can place any file by asking "which layer owns this responsibility?"

**Negative**
- Ceremony and indirection: three `DependencyInjection.cs` files, interfaces for things with one implementation, more projects to navigate.
- The "Domain references nothing" ideal is only *approximately* true — see [ADR-004](adr-004-efcore-in-application-no-repository.md) for the EF reference and [ADR-009](adr-009-in-process-precommit-domain-events.md) for `MediatR.Contracts`; even `BaseEntity` carries a `[NotMapped]` persistence attribute ([BaseEntity.cs L13-L14](../src/Domain/Common/BaseEntity.cs#L13-L14)).
- The rule is invisible when broken — a bad inner→outer reference still compiles; you discover it later when something that should be easy (a unit test) drags in the whole stack.

## Evidence

[Program.cs L7-L12](../src/Web/Program.cs#L7-L12), the three interface pairs above, and the project references in each `.csproj`. Grep confirms the direction holds: `rg "CleanArchitecture.Infrastructure" src/Application src/Domain` returns nothing.
