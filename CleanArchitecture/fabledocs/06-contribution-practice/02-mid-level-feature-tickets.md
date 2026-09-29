# Mid-Level Feature Tickets

Ten cross-layer tickets. Each **requires a design note before code** and includes risk and rollback sections — because touching schema/API/UI/tests together is where mid-level judgment shows. Compressed template; expand the design note before implementing.

Design-note requirement (every ticket): before writing code, write ½–1 page covering — data model change, API contract change, validation/auth impact, the layer-by-layer touch list (entity → config → command/DTO → validator → mapper → client regen → UI), test plan, and rollback. If you can't fill the touch list from memory, re-read [02-file-reading-order](../01-codebase-cartography/02-file-reading-order.md).

---

## M1: Per-user data ownership (the flagship)
Difficulty: Hard — 2–4 days. Skills: auth, EF query filters, migrations-thinking, security testing.
Story: As a user, I want to see and modify only my own lists so that the app is genuinely multi-user.
Design note must cover: add `OwnerId` to `TodoList`; **EF global query filter** `HasQueryFilter(l => l.OwnerId == currentUserId)` so reads filter structurally (how do you inject the current user into `OnModelCreating`? — a known EF challenge; research `IUserContext` + filter on a captured provider); write-side ownership assert; **404 not 403** for foreign ids.
Risk: query filters apply to *every* query including the seeder and tests — high blast radius; a wrong filter hides all data. Rollback: feature-flag the filter; the `OwnerId` column is additive (nullable first).
Test plan: [cross-user denial recipe](../05-quality-engineering/02-writing-tests-here.md) — user B gets NotFound for user A's list, across read/update/delete; existing tests must be updated to create a user first.
Why upstreamable: probably *not* — it changes the template's teaching-simple nature; ideal fork learning. Say so.
Interview story potential: "designed multi-tenancy isolation with EF global query filters and proved it with cross-user denial tests" — a top-3 mid-level story ([08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md)).

## M2: Unique list titles, enforced
Difficulty: Medium — 1 day. Skills: DB constraints, concurrency, error mapping.
Design note: unique index in [TodoListConfiguration](../../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs); keep the validator for friendly 400s; catch `DbUpdateException` on the race → 409; existing-duplicate dedupe plan.
Risk: existing dup data breaks the index creation — needs a dedupe step. Rollback: drop the index (additive change, reversible).
Test: the parallel-create race test from [debug scenario 3](../05-quality-engineering/03-systematic-debugging.md).
Interview story potential: "turned an advisory uniqueness check into a real invariant and handled the race the validator couldn't."

## M3: Pagination for todo items
Difficulty: Medium — 1–2 days. Skills: API design, query shaping, contract change.
Design note: the current [GetTodos](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs) returns everything — unbounded ([performance](../05-quality-engineering/04-performance-thinking.md)). Add page/size to a per-list items query (or keep lists eager, page items). Decide offset vs keyset pagination and justify (keyset scales, offset is simpler).
Risk: a real contract change — both clients' UI must handle pages; regenerate clients. Rollback: additive endpoint, old one stays until clients migrate.
Interview story potential: "introduced pagination, chose keyset over offset for scale, and migrated clients without breaking the old contract."

## M4: Move item between lists — verify and test
Difficulty: Medium — ~1 day. Skills: existing-feature hardening, validation.
Design note: [UpdateTodoItemDetail](../../src/Application/TodoItems/Commands/UpdateTodoItemDetail/UpdateTodoItemDetail.cs) already changes `ListId` (the React UI uses it, [Todo.jsx L135-L150](../../src/Web/ClientApp-React/src/components/Todo.jsx#L135-L150)) but nothing validates the target list exists or (post-M1) belongs to the user; no test covers the move.
Risk: low; mostly adds a validator rule + test. Rollback: trivial.
Interview story potential: "found an under-validated cross-entity operation and added the missing target-existence (and ownership) checks."

## M5: Soft delete for lists
Difficulty: Medium–Hard — 2 days. Skills: query filters, schema, cascade semantics.
Design note: `IsDeleted` flag + global query filter to hide deleted; decide item cascade (soft-delete children too?); restore endpoint?
Risk: interacts with M1's filter (composing two query filters); cascade delete behavior changes. Rollback: additive column, filter behind flag.
Interview story potential: "implemented soft delete and reasoned about how it composes with tenancy query filters."

## M6: Acceptance test for the Todo feature
Difficulty: Medium — 1–2 days. Skills: Gherkin, Playwright, page objects.
Design note: no `Todo.feature` exists ([acceptance tests](../../tests/Web.AcceptanceTests/Features/) cover Home/Counter/Weather/Login only). Add feature file + step defs + a `TodoPage` page object following [LoginPage](../../tests/Web.AcceptanceTests/Pages/LoginPage.cs).
Risk: acceptance tests are slow/flaky-prone; keep scenarios thin. Rollback: test-only, no product risk.
Interview story potential: "added end-to-end coverage for the core feature; kept scenarios focused to avoid flake."

## M7: Rate limiting on auth endpoints
Difficulty: Medium — 1 day. Skills: security, ASP.NET Core middleware.
Design note: add `AddRateLimiter` with a policy on login/register; choose algorithm (fixed vs sliding window vs token bucket) and justify; where in the [pipeline](../../src/Web/Program.cs#L27-L49).
Risk: too-tight limits lock out real users; needs tuning + config. Rollback: remove middleware.
Interview story potential: "added login rate limiting; chose sliding-window and made limits configurable rather than hardcoded."

## M8: Log redaction policy
Difficulty: Medium — 1 day. Skills: observability, security, structured logging.
Design note: `{@Request}` logs whole commands ([LoggingBehaviour](../../src/Application/Common/Behaviours/LoggingBehaviour.cs#L32-L33)); design a destructuring allow-list or `[LogMasked]`-style approach so future secret-bearing commands don't leak ([security](../05-quality-engineering/05-security-checklist.md)).
Risk: over-redaction kills debuggability; under-redaction leaks. Rollback: config-driven.
Interview story potential: "got ahead of a PII-in-logs risk before any command carried secrets."

## M9: Migrate to EF migrations
Difficulty: Hard — 2–3 days. Skills: EF migrations, startup, CI.
Design note: generate initial migration; switch dev initialiser from `EnsureCreated` to `MigrateAsync` ([initialiser L44-L46](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46)); un-suppress the pending-changes warning ([Infrastructure/DI L33](../../src/Infrastructure/DependencyInjection.cs#L33)); CI check for pending model changes; handle all three DB providers.
Risk: the biggest-blast-radius ticket here — changes how every environment gets its schema. Rollback: keep EnsureCreated behind a flag during transition.
Why upstreamable: unlikely — the template deliberately avoids migrations for teaching simplicity; strong fork project. Note the tension.
Interview story potential: "replaced recreate-on-run with real migrations and added a CI guard against schema drift."

## M10: Convert the React app to TypeScript (incrementally)
Difficulty: Hard — 3+ days. Skills: TS, incremental migration, avoiding behavior change.
Design note: rename `.jsx`→`.tsx` one component at a time; type props/state; consume the *generated* client types instead of the untyped seam ([type system](../02-stack-and-language-mastery/03-type-system-and-contracts.md)); **no behavior changes in migration commits** ([review kata 8](../04-code-reading-gym/04-review-katas.md)).
Risk: the temptation to "improve" logic mid-migration (reintroducing stale-closure bugs). Rollback: per-file, low risk if truly mechanical.
Interview story potential: "led an incremental TS migration and kept mechanical changes strictly separate from behavior changes so reviewers could trust each PR."

---

## Drill

Write the full design note for **M1** (the flagship). It's the single most valuable artifact in this whole module: it forces you to reason about query filters, current-user injection into `OnModelCreating`, the read/write asymmetry, 404-vs-403, test rewrites, and rollback. Self-grade — Basic: touch list complete. Solid: you address how the current user reaches the query filter. Strong: you plan the migration of *existing* tests (all of which create data without a user) and choose 404 over 403 with a stated reason. This note is also your [system-design answer](../08-interview-prep/04-system-design-from-this-repo.md) for "add multi-tenancy."
