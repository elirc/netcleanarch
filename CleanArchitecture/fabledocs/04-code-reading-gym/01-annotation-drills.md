# Annotation Drills

Method: open the anchored code. For each excerpt write six annotations — **Inputs** (incl. implicit ones like DI and ambient state), **Outputs** (incl. exceptions), **Dependencies**, **Invariants** (what must always hold), **Side effects**, **Failure modes** (concrete input/state → concrete wrong outcome). Then grade against the rubric.

Grading scale for every drill — **Basic:** correct inputs/outputs. **Solid:** + dependencies and at least one true invariant. **Strong:** + a failure mode that names *specific state*, not "it could error."

---

## Drill 1: [TodoItem.Done setter](../../src/Domain/Entities/TodoItem.cs#L13-L26)

Rubric highlights: implicit input is the *current* `_done` value — the setter is stateful. Invariant: `TodoItemCompletedEvent` fires exactly on false→true transitions (true→true is silent). Side effect: appends to `_domainEvents`, which is *observable later* by the interceptor — a deferred side effect. Failure mode: set `Done=true` on an entity that is never saved → event never published, but also: event object holds a reference to the whole entity ([TodoItemCompletedEvent](../../src/Domain/Events/TodoItemCompletedEvent.cs#L5-L10)), so handlers see live, possibly not-yet-persisted state. Strong answers mention that seeding items with `Done = true` ([GetTodosTests L32-L34](../../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L26-L40)) fires events during test setup — surprise coupling.

## Drill 2: [ValidationBehaviour.Handle](../../src/Application/Common/Behaviours/ValidationBehaviour.cs#L15-L33)

Rubric: inputs include the *injected collection* of validators — possibly empty, possibly several. Invariant: `next()` is reached only if zero failures; all validators run even after one fails (aggregate errors, better UX than fail-fast). Side effect: none directly — but validators themselves may query the DB (BeUniqueTitle), so "validation" can do I/O. Failure modes: a validator that *throws* (rather than returning failures) escapes as a 500; two validators for the same type both run — duplicate rules produce duplicate messages. Strong: `Task.WhenAll` runs DB-touching validators concurrently on the same scoped DbContext if a request ever had two async validators — DbContext is not thread-safe; latent hazard worth an "investigate" flag.

## Drill 3: [AuditableEntityInterceptor.UpdateEntities](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L36-L54)

Rubric: implicit inputs: ChangeTracker state, `IUser.Id` (may be **null** — anonymous saves stamp null CreatedBy), `TimeProvider`. Invariant: `Created*` set only on Added; `LastModified*` on Added *and* Modified — so a fresh row has Created == LastModified. Failure mode: unchanged entity with a modified *owned* entity (Colour) still bumps LastModified — that's the `HasChangedOwnedEntities` clause ([L57-L64](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L57-L64)); without it, colour-only edits would be invisible in audit data. Strong: notice `utcNow` is captured *per entity* inside the loop — a large batch save gets microsecond-different timestamps; harmless but a real "did you actually read it" detail.

## Drill 4: [CreateTodoListCommandValidator.BeUniqueTitle](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26)

Rubric: dependency: `IApplicationDbContext` injected *into a validator* — validators are DI services here. Invariant it *claims*: titles unique across all lists. Invariant it *delivers*: titles unique at check time. Failure modes: (1) the TOCTOU race under concurrency; (2) it's global-unique, so once user isolation lands, users will collide with titles they can't even see — cross-feature interaction; (3) compare the update variant ([UpdateTodoListCommandValidator L21-L26](../../src/Application/TodoLists/Commands/UpdateTodoList/UpdateTodoListCommandValidator.cs#L21-L26)) which must exclude self (`Id != model.Id`) — forgetting that clause is the classic "can't save without renaming" bug.

## Drill 5: [WebApplicationExtensions.MapEndpoints](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28)

Rubric: inputs: an `Assembly` — the reflection scan boundary. Invariant: every non-abstract exported `IEndpointGroup` gets exactly one route group named by class. Dependencies: the *convention itself* (class name = route = OpenAPI tag). Failure modes: a group class made `internal` silently disappears from the API (`GetExportedTypes` sees public only) — deployment-time surprise from a visibility keyword; `RoutePrefix` read via static property reflection means a *typo'd* property name would be silently ignored (that's why it uses `nameof`). Strong: `Invoke(null, [group])` — a throwing `Map` becomes `TargetInvocationException` at startup: fail-fast, good.

## Drill 6: [TestApp.RunAsUserAsync](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L48-L80)

Rubric: side effects: creates *real* Identity rows and mutates *static* fields — global state. Invariant: after success, `_userId` matches a user that exists in the current (post-reset) DB. Failure modes: (1) parallel tests corrupt each other's identity — the statics are the reason NUnit runs this suite serially; (2) role creation isn't guarded like the seeder's — calling twice with the same role in one test would throw on duplicate; (3) `ResetState` ([L82-L91](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L82-L91)) clears statics *and* DB — a test calling `RunAsUserAsync` before `ResetState` order-couples with the base class SetUp.

## Drill 7: [Todo.jsx commitEdit + cancelEdit](../../src/Web/ClientApp-React/src/components/Todo.jsx#L176-L197)

Rubric: inputs: `editValue` state, `editingItem` (compared by *reference*), `originalTitle` and `editCancelledRef` — two refs, deliberately non-reactive. Invariants: blur always resolves the edit exactly once (the ref guards double-fire when Escape triggers blur). Side effects: optimistic state write, then PUT — note the *empty title deletes the item* ([L186-L190](../../src/Web/ClientApp-React/src/components/Todo.jsx#L185-L190)): a surprising product decision hiding in an edit handler. Failure modes: PUT fails → UI shows the new title, server has the old (no rollback); `i === editingItem` reference equality breaks if lists were ever re-fetched mid-edit. Strong: explain *why* refs instead of state for the cancel flag (state updates are async/batched; the blur handler needs a synchronous read).

## Drill 8: [FunctionalTestSetup.OneTimeSetUp](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L14-L46)

Rubric: dependencies: TestAppHost project reference, Docker *not* required for SQLite variant, 60s cancellation budget for the whole boot. Outputs: static `ScopeFactory` + `DbResetter` for the whole suite. Invariants: DB is healthy before any test runs (`WaitForResourceHealthyAsync`). Failure modes: slow first-time container pull blows the 60s token on other DB variants → entire suite errors in setup, reported as "all tests failed"; misleading signal worth recognizing (setup failure ≠ code failure). Strong: `ASPIRE_ALLOW_UNSECURED_TRANSPORT` — tests speak http to skip cert ceremony; fine in tests, a red flag anywhere near production config.
