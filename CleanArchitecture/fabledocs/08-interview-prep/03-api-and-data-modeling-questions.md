# API & Data Modeling Questions — Cards

13 cards on REST/RPC, validation, pagination, auth, schema, migrations, transactions, caching — all anchored to the repo's actual API and data layer.

---

## Q1: "Design the REST endpoints for todo lists and items."
Testing: resource modeling, verb/status choices.
Anchor: [TodoLists.cs](../../src/Web/Endpoints/TodoLists.cs), [TodoItems.cs](../../src/Web/Endpoints/TodoItems.cs).
Junior: lists CRUD verbs. Mid: maps verbs to semantics — POST→201+id, PUT→204, DELETE→204, GET→200 — and notes the id-in-URL-must-match-body guard ([TodoItems L34-L35](../../src/Web/Endpoints/TodoItems.cs#L34-L35)). Senior: critiques — "there's no GET-by-id despite the `Created` header pointing at one ([L27](../../src/Web/Endpoints/TodoItems.cs#L27)); items are nested under lists conceptually but exposed flat; I'd decide nesting (`/lists/{id}/items`) vs flat deliberately."

## Q2: "PUT vs PATCH — this repo has both. Why?"
Anchor: [UpdateTodoItem (PUT)](../../src/Web/Endpoints/TodoItems.cs#L32) vs [UpdateTodoItemDetail (PATCH)](../../src/Web/Endpoints/TodoItems.cs#L44).
Junior: "PUT replaces, PATCH updates part." Mid: "PUT is a full idempotent replace (title+done), PATCH a partial update (priority+note+list) — the split matches how the UI edits them separately." Senior: notes both are effectively idempotent here and that the PUT sends the whole object from the client ([updateCheckbox](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167)), which risks clobbering concurrent edits without a concurrency token.

## Q3: "AuthN vs AuthZ — where does each live in your app?"
Anchor: [Identity/MapIdentityApi](../../src/Web/Endpoints/Users.cs#L12), [AuthorizationBehaviour](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs), [RequireAuthorization](../../src/Web/Endpoints/TodoItems.cs#L13).
Junior: conflates them. Mid: "authn = who (cookie/Identity), authz = what (endpoint `RequireAuthorization` + `[Authorize]` behaviour); 401 vs 403." Senior: the depth critique — "authz here is coarse (authenticated? in role?) but has *no resource-level check* — any user touches any list; that's the IDOR gap ([security](../05-quality-engineering/05-security-checklist.md))."

## Q4: "Where should validation live?"
Anchor: the [four-layer stack](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) — UI hints, model binding, [FluentValidation](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs), [domain invariants](../../src/Domain/ValueObjects/Colour.cs#L5-L15).
Junior: "on the server." Mid: "every layer for a different audience — UI for speed, server as the authority, domain for invariants." Senior: "and they should agree on an error contract; here a domain violation (bad colour) escapes as 500 instead of 400 because layer 4 isn't mapped — the layers exist but aren't unified."

## Q5: "Validator vs database constraint — the uniqueness example."
Anchor: [BeUniqueTitle validator](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26); no unique index in [TodoListConfiguration](../../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs).
Junior: "the validator checks it." Mid: "the validator gives friendly errors but is check-then-act — a TOCTOU race lets two concurrent creates both pass." Senior: "only a unique index enforces the invariant; keep the validator for UX, add the index, catch `DbUpdateException`→409, and plan a dedupe for existing data ([debug scenario 3](../05-quality-engineering/03-systematic-debugging.md))."

## Q6: "What is idempotency and which of your endpoints have it?"
Anchor: [TodoItems verbs](../../src/Web/Endpoints/TodoItems.cs#L15-L18).
Junior: vague. Mid: "same operation twice = same effect once; PUT/DELETE idempotent, POST create not." Senior: per-verb with nuance — "PUT `Done=true` twice is idempotent, and even the domain event respects it via the false→true transition check ([TodoItem L17-L24](../../src/Domain/Entities/TodoItem.cs#L17-L24)); DELETE is effect-idempotent but the second call 404s because of the NotFound guard — effect-idempotent, response-non-idempotent."

## Q7: "How does EF translate your object changes to SQL, and where are transaction boundaries?"
Anchor: [any handler's single SaveChangesAsync](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs#L31-L35).
Junior: "it saves." Mid: "change tracker diffs entities; one SaveChanges = one transaction; each use case here makes exactly one save so it's atomic without explicit transactions." Senior: the subtlety — "domain events dispatch *inside* the save ([interceptor](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L30)), so handler DB writes join the transaction but external effects escape it — the dual-write problem."

## Q8: "How do you evolve a schema with zero downtime?"
Anchor: this repo *doesn't* — [EnsureCreated](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46).
Junior: "run migrations." Mid: "generate a migration, apply on deploy; make changes backward-compatible." Senior: expand–contract in full — "add nullable column → deploy code reading both → backfill → tighten → drop old; never rename in one step; rollback is usually roll-forward. This repo recreates the DB in dev and suppresses the pending-changes warning ([DI L33](../../src/Infrastructure/DependencyInjection.cs#L33)) — it has no production migration story, which is the first thing I'd add ([M9](../06-contribution-practice/02-mid-level-feature-tickets.md))."

## Q9: "How would you model transactions across multiple aggregates?"
Anchor: single-save-per-use-case pattern; [DeleteTodoList cascade](../../src/Application/TodoLists/Commands/DeleteTodoList/DeleteTodoList.cs).
Junior: "wrap in a transaction." Mid: "one SaveChanges covers multiple rows atomically; deleting a list cascades to items via the relationship convention." Senior: "for cross-aggregate consistency without a distributed transaction, you use eventual consistency — domain events + an outbox ([S1](../06-contribution-practice/03-senior-build-projects.md)) — and design idempotent consumers; two-phase commit is usually the wrong tool."

## Q10: "What does MediatR buy you, and when would you drop it?"
Anchor: [pipeline registration](../../src/Application/DependencyInjection.cs#L16-L23), thin endpoints.
Junior: "it sends commands." Mid: "an in-process bus giving uniform cross-cutting behaviours and HTTP-free-testable handlers; Send = one handler, Publish = many." Senior: "it costs indirection (F12 tells you nothing — navigate by type) and tempts over-abstraction; for a 5-endpoint service with no cross-cutting needs I'd skip it. Here the five behaviours justify it."

## Q11: "How do you find and fix a slow API endpoint?"
Anchor: [PerformanceBehaviour watchdog](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L37-L50), OTel traces, EF logging.
Junior: "add caching." Mid: "measure first — trace to find the fat span, EF log to count queries, then optimize." Senior: gives a real find — "the per-request username lookup in Logging/Performance behaviours adds a DB query to every authenticated request just to log a name instead of an id ([performance](../05-quality-engineering/04-performance-thinking.md)); measurable, safe, small — a great first perf PR. And GetTodos is unbounded — needs pagination at scale."

## Q12: "How do you keep client and server contracts in sync?"
Anchor: [the codegen chain](../02-stack-and-language-mastery/03-type-system-and-contracts.md) — C# records → OpenAPI → [NSwag](../../src/Web/ClientApp-React/nswag.json).
Junior: "keep them updated manually." Mid: "generate the typed client from the OpenAPI doc so a contract change becomes a build failure on the client side." Senior: the blast-radius twist — "endpoint method names *are* the operationIds ([EndpointRouteBuilderExtensions](../../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs#L20-L26)), so renaming a handler renames the generated client method — a 'refactor' is a breaking change ([contrast 9](../04-code-reading-gym/03-fake-code-contrasts.md))."

## Q13: "What API change looks safe but is breaking?"
Anchor: [operationId = method name](../../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs#L20-L26); DTO shape changes.
Junior: "removing a field." Mid: "renaming a field or endpoint — clients depend on the names." Senior: this repo's specific traps — "renaming a handler method (breaks generated client method names *and* external OpenAPI consumers), returning an entity instead of a DTO (leaks fields, cycles), changing a DTO property's nullability. I'd pin operationIds with explicit `.WithName` before any rename."

---

## Meta-drill

Pick Q5, Q8, Q11 (the three richest) and write a 5-sentence answer for each hitting: definition, this-repo evidence, tradeoff, failure mode, fix. These three come up in nearly every mid-level API round. Then whiteboard Q1 fresh — design the API without looking, then diff against the real endpoints and note every choice you'd make differently.
