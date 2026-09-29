# Architecture Critique

An honest assessment, written the way a staff engineer would after two weeks in the codebase. Confirmed observations are anchored; hypotheses are labeled. This file doubles as system-design interview material — [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md) turns it into a whiteboard exercise.

Frame it fairly: this is a **template**, optimized to teach and to scaffold, not to run a business. Several "gaps" are scope decisions. Critiquing it still builds exactly the judgment interviews test — as long as you distinguish *wrong* from *out of scope*.

## Strongest design choices (steal these)

1. **Tradeoffs are written down.** Three real ADRs with honest "Harder:" sections ([ADR-001](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md), [ADR-002](../../docs/decisions/ADR-002-Aspire-For-Orchestration-And-Testing.md), [ADR-003](../../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md)). Most production codebases can't produce one. The meta-lesson outranks any pattern: *decisions have documented costs*.
2. **Minimal, consumer-shaped interfaces.** [IUser](../../src/Application/Common/Interfaces/IUser.cs) is two properties. The test suite's ability to swap it in one place ([WebApiFactory L17-L28](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28)) is the proof the boundary is real.
3. **One error contract, co-designed with its documentation.** Typed exceptions → [ProblemDetails mapping](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs) → [OpenAPI transformer](../../src/Web/Infrastructure/ApiExceptionOperationTransformer.cs) advertising exactly those statuses → clients that parse the shape ([Todo.jsx L77-L83](../../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83)).
4. **The test pyramid has a real strategy.** Pure logic in Domain.UnitTests; pipeline+SQL truth in FunctionalTests against a real DB with Respawn; browser truth in Playwright acceptance tests. Each layer covers another's blind spot ([05-quality-engineering/01](../05-quality-engineering/01-testing-strategy.md)).
5. **Policy enforced at seams, not by convention.** Auditing and event dispatch live at the SaveChanges seam ([interceptors](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs)); cross-cutting request concerns at the pipeline seam. New code gets the policy for free — the architecture makes the right thing automatic.
6. **Contract-first client generation** rather than hand-written fetch code ([pattern 12](05-pattern-catalog.md)).

## Risks and gaps, prioritized

### P1 — No resource-level authorization (confirmed, by scope)
Any authenticated user reads/writes/deletes anyone's lists ([GetTodos L43-L47](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47); no ownership check in any handler). For a template: intentional. For any deployment: the first blocker.
Migration path: add `OwnerId` to `TodoList` → EF **global query filter** (`HasQueryFilter`) so reads are filtered structurally, not per-handler → write-side check in a shared helper or behaviour → cross-user denial tests. Test strategy: functional tests asserting user B gets `NotFoundException` for user A's list id (404 over 403 to avoid existence leaks — say this in interviews).

### P2 — No migration story (confirmed, by scope)
`EnsureDeleted/EnsureCreated` in dev ([initialiser L44-L46](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46)); zero migrations checked in; the pending-model-changes warning is *suppressed* ([Infrastructure/DependencyInjection.cs L33](../../src/Infrastructure/DependencyInjection.cs#L33)) — the alarm that would tell you the model drifted is off.
Migration path: `dotnet ef migrations add Initial`, switch initialiser to `MigrateAsync()` in dev, un-suppress the warning, document expand–contract in CONTRIBUTING. Test strategy: the functional suite already runs against real schema; add a CI step asserting no pending model changes.

### P3 — Domain events dispatched pre-commit, in-process (confirmed design tension)
Full analysis in [04-side-effects](04-side-effects-async-and-reliability.md). Fine while handlers only log; becomes a correctness bug the day one sends email. The design *invites* that day — the machinery looks general-purpose.
Migration path: outbox ([senior project S1](../06-contribution-practice/03-senior-build-projects.md)). Cheap interim: move dispatch to `SavedChangesAsync` and accept lost-event risk for non-critical handlers — and write the ADR for whichever you pick.

### P4 — Advisory uniqueness, no optimistic concurrency (confirmed)
Unique titles by validator query only ([validator L21-L26](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26)); no unique index; no rowversion anywhere; full-object PUTs from the client mean last-write-wins clobbering ([updateCheckbox](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167)).
Migration path: unique index + 409 handling; add a concurrency token to `BaseAuditableEntity` (or reuse `LastModified`) and honor `If-Match`/`ETag` if staleness matters. Assess honestly: for a todo app, low stakes; the *pattern literacy* is the point.

### P5 — Observability half-wired (confirmed)
OTel is set up beautifully ([ServiceDefaults](../../src/ServiceDefaults/Extensions.cs#L44-L71)), but: request logging serializes whole command payloads (`{@Request}` in [LoggingBehaviour L32-L33](../../src/Application/Common/Behaviours/LoggingBehaviour.cs#L32-L33) and [UnhandledExceptionBehaviour L25](../../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L25)) — a PII/credentials time bomb as commands grow; the slow-request path does an extra DB read ([PerformanceBehaviour L43-L46](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L43-L46)); no custom metrics or health checks beyond liveness ([ServiceDefaults L92-L99](../../src/ServiceDefaults/Extensions.cs#L92-L99)).
Migration path: redaction policy (destructure allow-lists), drop the username lookup (log the id; join later), add a DbContext health check.

### P6 — Cosmetic/rot items (confirmed, small)
Stale [Web.http](../../src/Web/Web.http) referencing removed routes; `Created` location header `/{nameof}` mismatch ([TodoLists L36](../../src/Web/Endpoints/TodoLists.cs#L36)) with no GET-by-id to point at; wide-open CORS default ([Program.cs L28-L31](../../src/Web/Program.cs#L28-L31)); OpenAPI/Scalar unmapped from environment gates ([Program.cs L35-L36](../../src/Web/Program.cs#L35-L36)); empty [Infrastructure.IntegrationTests](../../tests/Infrastructure.IntegrationTests/GlobalUsings.cs) project; `Note` unbounded ([03-validation](03-validation-auth-and-permissions.md)); role-string trim question ([AuthorizationBehaviour L41-L45](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L41-L45), *investigate*). Each is a ready-made first PR — see [06-contribution-practice/01](../06-contribution-practice/01-good-first-tickets.md).

## Hypotheses (unconfirmed — would verify before acting)

- The React app's dialog-heavy state (13 `useState`/refs in one 423-line component) will resist the next feature; a reducer or state chart would pay off. *Hypothesis: based on reading, not on trying to extend it.*
- Aspire suite boot under the [60s test timeout](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L17) may flake on slow CI runners. *Hypothesis: no CI failure data examined.*
- `MapIdentityApi`'s default lockout/rate-limiting posture may be inadequate for a public deployment. *Hypothesis: framework defaults not audited here.*

## "What I'd change if I owned this for three months"

In order: (1) ship the ownership model (P1) with global query filters and denial tests — it converts the template into a deployable multi-user app and is the best learning-per-line change available; (2) migrations + un-suppress the warning (P2); (3) unique index + 409 (P4, small); (4) log redaction (P5, small); (5) the outbox as a *documented optional pattern* (P3) — with an ADR, because the template's simplicity is a feature and every addition fights it. I would *not* add repositories, MassTransit, or microservices — the ADRs already argue why, and restraint is the senior move.

## Drill

Defend the opposite: write ten sentences arguing the template is right to ship without ownership, migrations, or an outbox. (Audience is developers *learning the architecture*; every addition dilutes the signal; scope creep is the death of templates.) If you can argue both directions with specifics, you're interviewing at mid+ level. Then decide which side you'd take *for a startup's production fork* and write the one-paragraph ADR.
