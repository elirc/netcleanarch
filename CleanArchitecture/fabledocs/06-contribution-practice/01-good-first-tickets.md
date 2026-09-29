# Good First Tickets

Fifteen junior tickets, spread across domain, application, web, infra, tests, and both clients. Each is small, follows an existing pattern, and is testable. Difficulty: **Easy** (locate + small change) to **Medium** (one layer, needs a test). Full ticket template used for the first few; later ones are compressed but carry the same fields.

---

## Ticket 1: Bound the `Note` field length
Difficulty: Easy — ~30 min. Skills: validation, DB config.
Story: As a maintainer, I want todo item notes length-limited so that oversized input can't bloat rows or become a cheap DoS.
Why good: closes a real gap ([security checklist](../05-quality-engineering/05-security-checklist.md)); tiny blast radius; follows the exact Title pattern.
Acceptance: [ ] validator rejects Note > e.g. 4000 chars with a 400 [ ] DB column bounded [ ] test.
Read first: [UpdateTodoItemDetail.cs](../../src/Application/TodoItems/Commands/UpdateTodoItemDetail/UpdateTodoItemDetail.cs), [TodoItemConfiguration.cs](../../src/Infrastructure/Data/Configurations/TodoItemConfiguration.cs), [CreateTodoItemCommandValidator.cs](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItemCommandValidator.cs) (pattern to copy).
Files likely touched: a new/existing `UpdateTodoItemDetailCommandValidator`, `TodoItemConfiguration`, a functional test.
Plan: 1. add `RuleFor(v => v.Note).MaximumLength(4000)` in a validator for the detail command. 2. add `builder.Property(t => t.Note).HasMaxLength(4000)`. 3. test the reject.
What could go wrong: DB max length without the migration story means only new-schema dev DBs enforce it — note that in the PR.
Suggested checks: functional test sends 4001-char note, expects `ValidationException`.
Review questions: is 4000 the right ceiling? should Create also limit Note? (It doesn't accept one — mention.)
Interview story potential: "found and closed an unbounded-input gap; chose the limit by matching existing Title conventions."

## Ticket 2: Map `UnsupportedColourException` to 400
Difficulty: Easy — ~30 min. Skills: error contract, exception mapping.
Story: As an API consumer, I want a bad colour to return 400 not 500 so that client errors are distinguishable from server faults.
Why good: fixes the [trace-4](../04-code-reading-gym/02-trace-tables.md) gap; follows the existing switch pattern exactly.
Read first: [ProblemDetailsExceptionHandler.cs](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44), [UnsupportedColourException.cs](../../src/Domain/Exceptions/UnsupportedColourException.cs).
Plan: add a `UnsupportedColourException` arm returning 400 with a clear Title. Decide vs the alternative (a colour validator) and justify — best PR does *validator* so it joins the aggregated 400 error shape, and mentions why. 
What could make a maintainer reject: mapping a *domain* exception in Web couples layers further; argue it's acceptable (Web already knows Domain) or prefer the validator route. Show you considered both.
Interview story potential: "made domain-invariant violations return the right HTTP status; weighed exception-mapping vs validation and chose the one that kept the error contract consistent."

## Ticket 3: Add a test proving `TodoItemCompletedEvent` is dispatched
Difficulty: Medium — ~1–2 hr. Skills: testing, DI test overrides, events.
Story: As a maintainer, I want coverage that completing an item publishes the domain event so that the interceptor can't silently break.
Why good: fills a verified coverage gap ([testing strategy](../05-quality-engineering/01-testing-strategy.md)); no product change; pure safety.
Read first: [DispatchDomainEventsInterceptor.cs](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs), [WebApiFactory.cs](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28) (how to override services), [recipe 5](../05-quality-engineering/02-writing-tests-here.md).
Plan: register a recording `INotificationHandler<TodoItemCompletedEvent>` in test services; send `UpdateTodoItemCommand{Done=true}`; assert one invocation; a second identical send asserts *zero* new invocations (transition invariant).
Interview story potential: "closed a coverage gap on the event-dispatch path and proved the false→true transition invariant, not just that events fire."

## Ticket 4: Render the update-list validation error in the React UI
Difficulty: Medium — ~1 hr. Skills: React error handling, contract parsing.
Story: As a user, I want to see why a list rename failed (e.g. duplicate title) instead of a silent no-op.
Why good: fixes the [trace-1](../04-code-reading-gym/02-trace-tables.md) inconsistency — create parses errors, update swallows them.
Read first: [Todo.jsx commitNewList L77-L83](../../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83) (the pattern) vs [updateListOptions L96-L102](../../src/Web/ClientApp-React/src/components/Todo.jsx#L96-L102) (the gap).
Plan: mirror the create dialog's error parsing into the options dialog; add an error state + inline message.
What could go wrong: coupling to the ProblemDetails shape and PascalCase keys — same coupling create already has; acceptable, but note it.
Interview story potential: "made error UX consistent across dialogs by reusing the existing contract-parsing pattern."

## Ticket 5: Refresh the stale `Web.http` file
Difficulty: Easy — ~45 min. Skills: reading the real API surface.
Story: As a developer, I want the `.http` scratch file to match real endpoints so that manual API testing works.
Why good: [verified stale](../09-reference/verification-log.md) — references `/Identity/Account/Login`, `GET /api/TodoLists/1`, paginated items, `UpdateItemDetails?Id=` that don't exist.
Read first: [Web.http](../../src/Web/Web.http) vs the four endpoint groups; [Web-webapi.http](../../src/Web/Web-webapi.http).
Plan: rewrite each request against the real routes/verbs; note there's no GET-by-id endpoint (so remove those or add the endpoint — scope decision).
Interview story potential: "documentation-drift cleanup; used it to learn the real API surface end to end."

## Ticket 6: Add `[Authorize]` to all mutating Todo commands
Difficulty: Medium — ~1 hr. Skills: authorization, defense in depth, tests.
Story: As a security-conscious maintainer, I want write commands to enforce auth at the application layer, not only the endpoint, so that non-HTTP entry points stay protected.
Why good: closes the [uneven-depth](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) gap; follows [GetTodosQuery](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L9).
Read first: [AuthorizationBehaviour.cs](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs), all TodoItems/TodoLists commands.
Plan: add `[Authorize]` to each create/update/delete command; add functional tests asserting `UnauthorizedAccessException` without a user.
What could make a maintainer reject: if they consider endpoint-level auth sufficient and the attribute redundant — bring the "works without HTTP / future transports" argument.
Interview story potential: "added defense-in-depth authorization and proved it with tests that bypass HTTP entirely."

## Ticket 7: Fix the `Created` location header path
Difficulty: Easy — ~20 min. Skills: HTTP semantics, routing.
Story: As an API consumer, I want the 201 `Location` header to point at a real URL.
Why good: [TodoLists L36](../../src/Web/Endpoints/TodoLists.cs#L36)/[TodoItems L27](../../src/Web/Endpoints/TodoItems.cs#L27) emit `/{nameof(...)}/{id}` = `/TodoLists/{id}`, but the route is `/api/TodoLists` and there's no GET-by-id.
Plan: either correct the path to `/api/TodoLists/{id}` *and* add the GET-by-id endpoint it implies, or return the id without a misleading Location. The honest fix reveals a missing endpoint — good scope conversation.
Interview story potential: "a one-line header bug surfaced a missing REST endpoint; scoped it deliberately rather than shipping a half-fix."

## Ticket 8: Add rollback to the React optimistic checkbox
Difficulty: Medium — ~1 hr. Skills: optimistic UI, error handling.
Story: As a user, if toggling a checkbox fails, I want it to revert and tell me, not silently lie.
Why good: fixes [scenario 4](../05-quality-engineering/03-systematic-debugging.md); the disciplined optimistic-update pattern.
Read first: [updateCheckbox L161-L167](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167); [fake-code contrast 5](../04-code-reading-gym/03-fake-code-contrasts.md).
Plan: snapshot before mutate; on catch, restore snapshot + surface a message.
Interview story potential: "hardened an optimistic UI with proper rollback — turned a silent data-divergence bug into a visible, recoverable error."

## Ticket 9: Add an `AddDbContextCheck` health check
Difficulty: Easy — ~30 min. Skills: health checks, operations.
Story: As an operator, I want `/health` to fail when the database is unreachable.
Why good: package already referenced ([Directory.Packages.props L23](../../Directory.Packages.props#L23)); tiny; real ops value ([observability](../05-quality-engineering/06-observability-and-operations.md)).
Read first: [ServiceDefaults AddDefaultHealthChecks L92-L99](../../src/ServiceDefaults/Extensions.cs#L92-L99).
Plan: `.AddDbContextCheck<ApplicationDbContext>()` where readiness checks live (mind the Development gating and the ServiceDefaults/Web layering).
Interview story potential: "distinguished liveness from readiness by adding a DB health check."

## Ticket 10: Order todo items deterministically
Difficulty: Easy — ~30 min. Skills: queries, invariants.
Story: As a user, I want items to appear in a stable order.
Why good: [GetTodos](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47) orders lists by Title but leaves *items* in DB-arbitrary order.
Plan: add an `OrderBy` on items inside the projection (by Id, or add/order by a position field — scope choice); test the order.
What could go wrong: ordering by Id ≠ user-intended order; if drag-reordering is ever wanted, this needs a position column — note the future.
Interview story potential: "made list rendering deterministic; flagged that true user-ordering needs a position field."

## Ticket 11: Tighten `TodoItemDto` nullability
Difficulty: Easy — ~30 min. Skills: contracts, nullable types.
Story: As a client developer, I want DTO types to reflect reality (Title is always present).
Why good: [contract looser than truth](../02-stack-and-language-mastery/03-type-system-and-contracts.md); reduces client null-checking.
Read first: [TodoItemDto.cs](../../src/Application/TodoLists/Queries/GetTodos/TodoItemDto.cs), [TodoItemConfiguration.cs](../../src/Infrastructure/Data/Configurations/TodoItemConfiguration.cs).
Plan: make `Title` non-nullable in the DTO; regenerate clients; verify [MappingTests](../../tests/Application.UnitTests/Common/Mappings/MappingTests.cs) still pass.
Interview story potential: "aligned DTO nullability with database reality to remove downstream null-handling."

## Ticket 12: Enable the React lint step in CI
Difficulty: Medium — ~1 hr. Skills: CI, tooling.
Story: As a maintainer, I want the React app linted in CI so style/hook errors are caught.
Why good: CI runs .NET tests but [not `npm run lint`](../02-stack-and-language-mastery/04-tooling-and-build-system.md); the script already exists.
Read first: [build.yml](../../.github/workflows/build.yml), [ClientApp-React/package.json L33](../../src/Web/ClientApp-React/package.json#L33).
Plan: add a lint step in the workflow (guarded by the `#if (!UseApiOnly)` convention the file uses).
What could make a maintainer reject: template conditional correctness — the step must not run for API-only generated solutions.
Interview story potential: "extended CI to cover a linting gap; respected the template's conditional-generation constraints."

## Ticket 13: Seed data idempotency guard for roles
Difficulty: Easy — ~30 min. Skills: reading idempotency, defensive code.
Story: Verify (and test) that re-running the seeder never throws on existing roles/users.
Why good: [TrySeedAsync](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L68-L88) is guarded — write the test that proves it and document the idempotency intent.
Plan: an integration-style test calling `TrySeedAsync` twice; assert single admin user/role. (Uses the real DB infra.)
Interview story potential: "proved seed idempotency with a test — turned an implicit invariant into a checked one."

## Ticket 14: Add `Colour` to `SupportedColours` correctly (guided)
Difficulty: Easy — ~20 min. Skills: value objects, the fast-track change.
Story: Add a new supported colour end to end (domain + API lookup + test).
Why good: the [fast-track](../00-fast-track.md) change, ticketed properly — smallest safe full-stack-ish change.
Plan: add the static + `SupportedColours` entry ([Colour.cs](../../src/Domain/ValueObjects/Colour.cs#L17-L60)); add to the `Colours` list in [GetTodos](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L32-L41); unit test `From`. No client change (picker is data-driven).
Interview story potential: "traced a new value through every layer that mentions it and identified which layers *didn't* need to change, and why."

## Ticket 15: Document the "no migrations" dev reset in CONTRIBUTING/README
Difficulty: Easy — ~45 min. Skills: technical writing, operations awareness.
Story: As a new contributor, I want to know the dev DB is recreated each run so I'm not surprised by lost data.
Why good: [surprising behavior](../01-codebase-cartography/05-key-flows.md) ([initialiser L44-L46](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46)) is undocumented; pure docs, zero risk.
Plan: a short "Local database" note explaining EnsureDeleted/EnsureCreated, why (dev velocity), and how production would differ (migrations).
Interview story potential: "documented a footgun for new contributors — the kind of paper cut that erodes onboarding."

---

## Drill

Pick **Ticket 2** or **Ticket 3** and actually do it in a branch. Then write the PR description using the [PR template](../07-career-and-collaboration/02-writing-prs-and-rfcs.md): what, why, how tested, risks, follow-ups. Self-grade — Basic: it compiles and tests pass. Solid: the PR description answers all five maintainer questions. Strong: you anticipated the reviewer's objection and addressed it in the description before they raised it.
