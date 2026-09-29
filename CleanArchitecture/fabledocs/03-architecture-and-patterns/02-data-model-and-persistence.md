# Data Model & Persistence

## The model

```
IdentityUser tables (AspNetUsers, AspNetRoles, ...)   ← from IdentityDbContext base
        (no FK link to domain tables — audit columns hold user IDs as plain strings)

TodoLists                              TodoItems
  Id (int, PK)                           Id (int, PK)
  Title (nvarchar(200), required)        ListId (FK → TodoLists, cascade delete)
  Colour_Code (owned value object)       Title (nvarchar(200), required)
  Created/CreatedBy                      Note (unbounded string)
  LastModified/LastModifiedBy            Priority (int enum)
                                         Done (bit)
                                         Created/CreatedBy, LastModified/LastModifiedBy
```

Sources: [ApplicationDbContext](../../src/Infrastructure/Data/ApplicationDbContext.cs#L10-L23) (inherits `IdentityDbContext<ApplicationUser>`, so Identity tables share the schema), [TodoListConfiguration](../../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs) (`OwnsOne(b => b.Colour)` maps the value object to a column on the same table), [TodoItemConfiguration](../../src/Infrastructure/Data/Configurations/TodoItemConfiguration.cs), audit columns from [BaseAuditableEntity](../../src/Domain/Common/BaseAuditableEntity.cs).

Things a senior reads off this model in thirty seconds:

- **Relationship by convention**: `TodoItem.ListId` + `TodoItem.List` + `TodoList.Items` ([TodoItem.cs L5, L28](../../src/Domain/Entities/TodoItem.cs#L5-L28), [TodoList.cs L9](../../src/Domain/Entities/TodoList.cs#L9)) → EF infers required one-to-many with **cascade delete**. Nothing states it explicitly; the delete-list-deletes-items behavior is invisible unless you know the convention. Implicit invariants are the ones that surprise people.
- **No indexes beyond keys**: `Title` uniqueness is *validated* ([CreateTodoListCommandValidator L21-L26](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26)) but not *constrained* — no `HasIndex(...).IsUnique()`. Under concurrency the rule can be violated. Rule of thumb: a validator is UX; a constraint is an invariant.
- **No optimistic concurrency**: no rowversion/concurrency token anywhere. Two users editing the same item = silent last-write-wins. The React client makes this concrete: `updateCheckbox` PUTs the *whole* item ([Todo.jsx L161-L167](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167)), so checking a box can overwrite a title someone else just changed.
- **`Note` is unbounded**: no max length in [TodoItemConfiguration](../../src/Infrastructure/Data/Configurations/TodoItemConfiguration.cs) and no validator rule on `Note` in [UpdateTodoItemDetail](../../src/Application/TodoItems/Commands/UpdateTodoItemDetail/UpdateTodoItemDetail.cs) — a 10MB note is accepted. Small, real finding; ticketed.
- **Audit columns are strings, not FKs**: `CreatedBy` holds the Identity user id with no referential integrity — deleting a user orphans nothing and breaks nothing, at the cost of joins requiring app knowledge.

## Transaction boundaries and consistency expectations

The **unit of work is one `SaveChangesAsync` call**. Every handler makes exactly one ([e.g. CreateTodoItem L33](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs#L31-L35)), so each use case is atomic without any explicit `BeginTransaction`. EF wraps multi-row saves in a transaction automatically.

Two subtleties this repo sets up perfectly for discussion:

1. Domain event handlers run *inside* the save call (before the write — [DispatchDomainEventsInterceptor L25-L30](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L30)). If a handler writes to the same context, its changes join the same save = atomic. If it does anything external, that effect escapes the transaction. Full treatment in [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md).
2. The async validator's DB read ([BeUniqueTitle](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26)) happens in a *separate* earlier query — read-then-write across two operations with no lock: the TOCTOU race.

## How to safely change the schema (the part this repo skips)

This repo has **no migrations** — dev recreates the DB per run ([ApplicationDbContextInitialiser L44-L46](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46)). You must still know the production playbook cold; it's a top-five interview topic:

1. **Generate a migration** (`dotnet ef migrations add AddDueDate -p src/Infrastructure -s src/Web`) — a C#/SQL diff checked into git, replayable, reviewable. *(Command shape inferred; EF Design package is referenced in [Directory.Packages.props L22](../../Directory.Packages.props#L22) but no migrations folder exists.)*
2. **Make changes expand–contract**: add nullable column → deploy code that writes both/reads either → backfill → tighten to NOT NULL → remove old column. Never rename in one step; a rename is an add + copy + drop spread over deploys.
3. **Roll back = roll forward**: practice the down-migration, but know that in production you usually ship a fixing migration instead (data written since deploy makes true rollback lossy).
4. **The blast radius checklist** for any schema change in this codebase: entity ([Domain/Entities](../../src/Domain/Entities/TodoItem.cs)) → configuration ([Infrastructure/Data/Configurations](../../src/Infrastructure/Data/Configurations/TodoItemConfiguration.cs)) → commands/DTOs + validators (Application) → AutoMapper profile ([TodoItemDto.cs L19-L26](../../src/Application/TodoLists/Queries/GetTodos/TodoItemDto.cs#L19-L26)) → regenerate TS clients → UI. Six stops; miss one and either the compiler (good) or a runtime mapping test ([MappingTests](../../tests/Application.UnitTests/Common/Mappings/MappingTests.cs#L30-L34)) catches you.

## Seeds and fixtures

Dev seed: admin role/user + one sample list ([TrySeedAsync](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L68-L109)), guarded by "if missing" checks so it's re-runnable — *idempotent seeding*. Test data is different: Respawn wipes tables before each test ([TestBase](../../tests/Application.FunctionalTests/Infrastructure/TestBase.cs) → [DatabaseResetter](../../tests/Application.FunctionalTests/Infrastructure/DatabaseResetter.cs#L40-L45)), and each test builds exactly the rows it asserts about ([GetTodosTests L26-L40](../../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L22-L44)). Seed-for-humans and fixtures-for-tests kept separate — a distinction sloppy codebases blur.

## Drill

Design (on paper) the change to make list titles unique **for real**: (a) the EF configuration line, (b) what happens to the existing validator — keep it? why?, (c) how the handler learns about a constraint violation and what status the API should return, (d) the test you'd add. Self-grade — Basic: `HasIndex(t => t.Title).IsUnique()`. Solid: keep the validator for friendly 400s, catch `DbUpdateException` for the race, map to 409. Strong: you mention the *migration* needed on a live DB and what to do about pre-existing duplicates (dedupe script before the constraint lands).

## Interview angle

- *"How do you evolve a schema without downtime?"* → expand–contract narrative above; contrast with this repo's EnsureCreated. → [08-interview-prep/03, Q8](../08-interview-prep/03-api-and-data-modeling-questions.md)
- *"Where are your transaction boundaries?"* → one save per use case; events inside the save. → [08-interview-prep/03, Q9](../08-interview-prep/03-api-and-data-modeling-questions.md)
- *"Validator vs constraint?"* → the uniqueness story. → [08-interview-prep/03, Q5](../08-interview-prep/03-api-and-data-modeling-questions.md)
