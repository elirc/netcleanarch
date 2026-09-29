# ADR-010: Schema by Convention with `EnsureCreated`; No Migrations; Provider-Agnostic via Switches

**Status:** Accepted (implicit) — and explicitly scoped to the template. This is the decision most likely to be *reversed* in any production fork.

## Context

A template optimized for "clone and run" wants zero database ceremony: a developer should get a working, seeded schema instantly, on any of the supported providers, without installing tooling or learning a migration workflow. Separately, the template supports SQLite, PostgreSQL, and SQL Server from one codebase.

## Decision

In development, **drop and recreate the schema every run** from the current EF model — [ApplicationDbContextInitialiser L44-L46](../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46) (`EnsureDeletedAsync` then `EnsureCreatedAsync`) — then reseed ([TrySeedAsync L68-L109](../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L68-L109), guarded so seeding is idempotent). There is **no migrations folder** and no history table. The schema is defined by convention plus fluent configuration ([TodoItemConfiguration](../src/Infrastructure/Data/Configurations/TodoItemConfiguration.cs), [TodoListConfiguration](../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs) — the latter maps `Colour` as an owned entity). The provider is selected by template switch at [Infrastructure/DependencyInjection.cs L26-L34](../src/Infrastructure/DependencyInjection.cs#L26-L34), and the pending-model-changes warning is **suppressed** at [L33](../src/Infrastructure/DependencyInjection.cs#L33).

## Alternatives Considered

- **EF migrations from day one** (`dotnet ef migrations add`, `MigrateAsync` at startup). The production-correct approach — versioned, replayable, reviewable schema changes. Rejected *for the template* because it adds real getting-started friction the template exists to eliminate.
- **A hand-maintained SQL schema / DbUp scripts.** More control, more ceremony, provider-specific. Rejected as heavier than `EnsureCreated` for the template's goal.

## Consequences

**Positive**
- Instant, zero-config local database; every run yields a clean, seeded state — ideal for a template and for test isolation ([ADR-008](adr-008-real-database-functional-testing.md) resets data on top of this created schema).
- One model targets three providers; provider specifics are quarantined behind switches.
- Idempotent seeding safely re-runs.

**Negative**
- **This codebase cannot evolve a production database as shipped.** `EnsureCreated` and migrations are mutually exclusive; adopting migrations later requires reconciliation. Introducing migrations is "job zero" for any real fork.
- **Every dev run wipes all local data** — a genuine footgun that surprises newcomers (and it's undocumented in the app itself).
- **Schema drift is invisible** because the pending-changes warning is suppressed ([L33](../src/Infrastructure/DependencyInjection.cs#L33)) — the exact alarm that would tell you the model diverged from a would-be migration is turned off.
- No production migration means **no rollback story** for schema; combined with the absence of a DB health check ([ADR-008](adr-008-real-database-functional-testing.md) sibling gap), the operational readiness of a fork is on you.

## Evidence

[ApplicationDbContextInitialiser L44-L46](../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46) and [L68-L109](../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L68-L109), the suppressed warning at [Infrastructure/DependencyInjection.cs L33](../src/Infrastructure/DependencyInjection.cs#L33), provider selection at [L26-L34](../src/Infrastructure/DependencyInjection.cs#L26-L34), and the absence of any `Migrations/` directory in the solution.
