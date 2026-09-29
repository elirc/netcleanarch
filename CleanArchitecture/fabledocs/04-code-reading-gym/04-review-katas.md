# Review Katas

Eight fake PRs against this repo. For each: read the intent and diff summary, write your review (aim for 5 minutes each), then compare with the expected findings. Findings are graded **Blocking** (must fix before merge), **Important** (fix now or file immediately), **Optional** (style/nice-to-have). All diffs are **illustrative fake code — not from this repo**.

Review voice rule, used in every example comment: *observation → consequence → suggestion*, and questions where you're not sure. Never "this is wrong"; always "this does X, which causes Y — consider Z."

---

## Kata 1: "Add DueDate to todo items"

Author intent: users want deadlines.
Fake diff summary: adds `public DateTime DueDate { get; set; }` to `TodoItem`; adds it to `UpdateTodoItemCommand` and the handler; nothing else.
Files this resembles: [TodoItem.cs](../../src/Domain/Entities/TodoItem.cs), [UpdateTodoItem.cs](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs), [TodoItemDto.cs](../../src/Application/TodoLists/Queries/GetTodos/TodoItemDto.cs)
Expected findings — **Blocking:** non-nullable `DateTime` on an existing entity means every existing row/new item gets `0001-01-01` — should be `DateTime?` (or `DateTimeOffset?` to match the audit fields' convention, [BaseAuditableEntity](../../src/Domain/Common/BaseAuditableEntity.cs)); DTO not updated so the UI can never *see* the due date — feature is write-only. **Important:** no validator rule (past dates allowed? by design? say so); `CreateTodoItemCommand` untouched — can't set a due date at creation; no test. **Optional:** consider `DateOnly` if time-of-day is meaningless.
Good review comment example:
> `DueDate` as non-nullable `DateTime` will default to `0001-01-01` for every existing item, which the UI will render as a real date. Making it `DateTimeOffset?` matches the audit-field convention in `BaseAuditableEntity` and makes "no deadline" representable. Also — I don't see it in `TodoItemDto`; is read support coming in a follow-up?

## Kata 2: "Cache GetTodos for performance"

Author intent: page load feels slow; adds `IMemoryCache` in `GetTodosQueryHandler` with a 5-minute TTL, key `"todos"`.
Files this resembles: [GetTodos.cs](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs)
Expected findings — **Blocking:** single global key means one user's (future: one tenant's) data serves everyone — correctness before performance; no invalidation: every create/update/delete serves stale data up to 5 min, and the UI *also* maintains its own optimistic cache — two stale layers compounding. **Important:** was slowness measured? (Ask for evidence — see [performance module](../05-quality-engineering/04-performance-thinking.md); this query is one SQL statement.) Cache belongs behind the interface seam or at HTTP layer with proper variance if ever added. **Optional:** if lookup data (colours/priorities) is the slow part, cache *that* — it's static.
Good review comment example:
> Before adding a cache: do we have a trace showing this query is the bottleneck? It's a single projected SQL query. If we do cache, keying by `"todos"` alone will cross user boundaries once ownership lands (M1), and there's no invalidation on the write paths — a user who creates a list won't see it for 5 minutes.

## Kata 3: "Return the entity from CreateTodoList"

Author intent: saves the client a round trip by returning the whole created `TodoList` instead of the id.
Fake diff: endpoint returns `TypedResults.Created(..., entity)` where `entity` is the tracked `TodoList`.
Files this resembles: [TodoLists.cs L32-L37](../../src/Web/Endpoints/TodoLists.cs#L30-L37), [CreateTodoList.cs](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs)
Expected findings — **Blocking:** serializing the *entity* leaks audit fields and invites cycles (`Items` ↔ `List`) — return a DTO if returning anything (contrast 2). **Important:** this changes the OpenAPI response schema → regenerated clients change return type → both SPAs' `createTodoList` call sites ([Todo.jsx L72](../../src/Web/ClientApp-React/src/components/Todo.jsx#L69-L76)) now receive an object where they expect an int — coordinate the client updates in the same PR. **Optional:** the motivation is real; a `TodoListDto` return is a fine design, just do it contract-first.

## Kata 4: "Fix flaky functional tests by sharing the user"

Author intent: `RunAsDefaultUserAsync` in every test is slow; moves user creation to `OneTimeSetUp` and removes per-test `ResetState`.
Files this resembles: [TestBase.cs](../../tests/Application.FunctionalTests/Infrastructure/TestBase.cs), [FunctionalTestSetup.cs](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs)
Expected findings — **Blocking:** removing per-test reset makes tests order-dependent — the uniqueness validator will fail on the second run of any create test ("New List" already exists, see [CreateTodoItemTests L22-L25](../../tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs#L22-L31)); shared user + Respawn is also inconsistent: reset wipes Identity tables, so the "shared" user is deleted after the first test anyway. **Important:** if setup cost is the real complaint, measure it and consider seeding the user *inside* `ResetState`. **Optional:** none.
Senior lesson: test isolation is an invariant; performance PRs that trade it away need a much higher bar of evidence.

## Kata 5: "Add logging to the Colour value object"

Author intent: debugging colour issues; injects `ILogger<Colour>` into `Colour` and logs in `From`.
Files this resembles: [Colour.cs](../../src/Domain/ValueObjects/Colour.cs)
Expected findings — **Blocking:** value objects are constructed by EF materialization and object initializers — they can't take constructor-injected services, and Domain referencing `Microsoft.Extensions.Logging` breaks the dependency budget ([ADR-003](../../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md) allowed *contracts only*). **Important:** the observable event ("unsupported colour rejected") already surfaces as `UnsupportedColourException` and is logged with the request by [UnhandledExceptionBehaviour](../../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L21-L27) — the information exists; point the author at it. **Optional:** if rejection *frequency* matters, a metric at the boundary (behaviour), not the domain.

## Kata 6: "Harden logout"

Author intent: "the `[FromBody] object empty` thing is ugly"; changes [Users.cs Logout](../../src/Web/Endpoints/Users.cs#L19-L28) to a parameterless GET.
Expected findings — **Blocking:** GET logout is a CSRF/prefetch hazard — any `<img src="/api/Users/logout">` or link-prefetcher logs users out; state changes belong on POST. The weird empty-body parameter is *load-bearing*: it forces a JSON content-type request, which browsers won't send cross-site without CORS success. **Important:** if the author finds it unclear, the fix is a comment, not a semantic change. **Optional:** rename `empty` to something self-documenting.
Senior lesson: before deleting something ugly, archaeology: *why* is it there? Ugly-but-load-bearing is a category.

## Kata 7: "Speed up the pipeline by reordering behaviours"

Author intent: "validation is cheap, auth queries the DB — run validation first to fail faster"; swaps lines in [Application/DependencyInjection.cs L20-L21](../../src/Application/DependencyInjection.cs#L16-L23).
Expected findings — **Blocking:** validation-before-auth leaks information to unauthenticated callers — the uniqueness validator would confirm which titles exist to anonymous requests (via `ISender`-style entry) and burns DB queries on unauthenticated traffic; order is a security property, not a perf knob. **Important:** the premise is also wrong for attributed requests: auth for most requests is a null-check on `IUser.Id` ([AuthorizationBehaviour L29](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L26-L32)), no DB. **Optional:** an ADR-style comment in the DI file documenting the order rationale would prevent the next such PR — suggest the author write it.

## Kata 8: "Migrate Todo.jsx to TypeScript"

Author intent: renames to `Todo.tsx`, adds types, and in passing "cleans up" by replacing all functional `setLists(ls => ...)` updates with direct `setLists(newLists)` computed at the top of each handler.
Expected findings — **Blocking:** the "cleanup" reintroduces stale-closure bugs under concurrent updates (contrast 7) — the functional-update form is deliberate; request it be reverted while keeping the type work. **Important:** typing `editingItem` will surface the reference-equality comparison ([L266](../../src/Web/ClientApp-React/src/components/Todo.jsx#L266)) — good moment to switch to id comparison, but that's a behavior change: separate commit, called out. **Optional:** enable `strict` in tsconfig for the file; add the missing `web-api-client` types note for reviewers who haven't generated it.
Senior lesson: mechanical migrations and behavior changes must not share a diff — reviewers can verify one or the other, not both at once.

---

## Meta-drill

Score yourself across all eight: +2 per blocking finding you caught, +1 per important, −1 per real blocking finding you missed. 20+ = you're reviewing at a solid mid level. Then the harder rep: for katas 4 and 6, write the *full* review comment thread — including the reply you'd send when the author pushes back. Kindness under disagreement is the actual skill; see [07-career/01](../07-career-and-collaboration/01-code-review-mindset.md).
