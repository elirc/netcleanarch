# Side Effects, Async Work & Reliability

A **side effect** is anything a use case changes beyond its return value: DB writes, logs, events, emails, cache entries, external API calls. Reliability engineering is mostly the discipline of asking, for each side effect: *what if it runs twice? what if it half-runs? how would I know?*

## Inventory: every side effect in this codebase

| Side effect | Trigger | File | Reliability property |
|---|---|---|---|
| DB writes | each handler's single `SaveChangesAsync` | e.g. [CreateTodoItem L31-L35](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs#L31-L35) | atomic per use case (EF transaction) |
| Audit stamping | save interceptor | [AuditableEntityInterceptor L36-L54](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L36-L54) | same transaction as the write — can't drift |
| Domain event handlers | save interceptor, **before** the write | [DispatchDomainEventsInterceptor L25-L49](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L49) | in-process, synchronous, unrecoverable on crash |
| Structured logs | pipeline pre-processor + warning path | [LoggingBehaviour](../../src/Application/Common/Behaviours/LoggingBehaviour.cs), [PerformanceBehaviour L37-L50](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L37-L50) | fire-and-forget by nature |
| Telemetry | OpenTelemetry auto-instrumentation | [ServiceDefaults L44-L71](../../src/ServiceDefaults/Extensions.cs#L44-L71) | export best-effort |
| DB drop/create + seed | dev startup | [ApplicationDbContextInitialiser](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L40-L109) | idempotent seed (guarded inserts) |
| Identity user/role writes | seeding, tests, register endpoint | [TrySeedAsync L68-L88](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L68-L88) | guarded, idempotent |

What's **absent** — and therefore what you must be able to design when asked: queues, background workers, scheduled jobs, webhooks, outgoing emails, caches, external API calls, retries with backoff (the only retry machinery is the default HTTP resilience handler for *outgoing* HttpClient calls, [ServiceDefaults L26-L33](../../src/ServiceDefaults/Extensions.cs#L26-L33) — and the app currently makes no outgoing calls).

## The one deep mechanism: domain events, dissected

Read [DispatchDomainEventsInterceptor.DispatchDomainEvents](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L32-L49) line by line:

1. Collect tracked entities holding events → 2. copy events to a list → 3. **clear events first** (prevents re-entrancy loops if a handler triggers another save) → 4. `await _mediator.Publish` each, serially.

Now the load-bearing design fact: this runs from `SavingChangesAsync` — **before** rows hit the database.

| Property | Consequence |
|---|---|
| Handler writes to the same DbContext | joins the same save → **atomic with the trigger**. This is the strongest argument *for* before-save dispatch |
| Handler does external I/O (email, HTTP) | effect happens even if the save subsequently **fails** → ghost side effects |
| Handler throws | save aborts → trigger change lost → user gets 500. Coupling: your logging handler can break todo-checking |
| Process crashes between publish and commit | external effects done, DB says nothing happened |
| Latency | all handlers run inside the request; slow handler = slow API |

Alternative designs you must be able to sketch on a whiteboard:

- **After-save dispatch** (`SavedChangesAsync`): DB truth first, then effects. Failure mode inverts: save succeeds, handler fails → lost event. Pick your poison — *dual-write problem* either way.
- **Transactional outbox** (the industry answer): inside the same transaction, write the event to an `OutboxMessages` table. A background dispatcher reads the table, delivers (email/queue/webhook), marks done, retries with backoff on failure. Consumers get *at-least-once* delivery, so handlers must be **idempotent** (processing the same message twice is harmless — dedupe by message id). This is [senior project S1](../06-contribution-practice/03-senior-build-projects.md).

Idempotency elsewhere in the repo, to anchor the word: PUT handlers are naturally idempotent (setting `Done=true` twice = once — even the event respects it via the transition check, [TodoItem L17-L24](../../src/Domain/Entities/TodoItem.cs#L17-L24)); POST create is not (retrying a timed-out create can duplicate); the seed is (guarded inserts). "Which of your endpoints can be safely retried?" is a beautiful interview question — now you can answer it per-verb with reasons.

## Failure visibility

For each flow, "how would I know it broke?" — the observability module ([05-quality-engineering/06](../05-quality-engineering/06-observability-and-operations.md)) goes deeper, but the reliability-relevant facts: unhandled exceptions are logged with full request payload ([UnhandledExceptionBehaviour L21-L27](../../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L21-L27)); slow requests (>500ms) warn with user context ([PerformanceBehaviour L37-L50](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L37-L50)); domain event handling logs at info ([LogTodoItemCompleted L17](../../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs#L15-L20)). Two quiet risks in those behaviours themselves: `{@Request}` serializes **whole commands into logs** — the day a command carries a password or PII, it's in the log stream (the seed of a real security ticket); and PerformanceBehaviour rewards slow requests with an *extra DB query* for the username ([L43-L46](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L43-L46)) — observability adding load precisely when the system is struggling. Small, but exactly the kind of second-order thinking seniors are paid for.

## Client-side effects deserve the same questions

The React app's optimistic update ([updateCheckbox, Todo.jsx L161-L167](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167)) is a side effect (server write) with a UI assumption (it succeeded). On failure: `console.error`, UI keeps the lie, no rollback, no retry. The disciplined pattern — snapshot → apply → on failure restore + toast — is [junior ticket J8](../06-contribution-practice/01-good-first-tickets.md). Same discipline, different tier of the stack.

## Drills

1. **Ghost-effect trace:** suppose `LogTodoItemCompleted` sent an email instead of logging. Walk the failure matrix: save fails after publish / handler throws / process dies mid-request. For each: email sent? item saved? user told what? Write the 3×3 grid; check against the table above.
2. **Design the outbox schema** on paper: table columns, the dispatcher loop, how "processed" is marked, what happens when delivery fails 5 times. Compare with the sketch in [senior project S1](../06-contribution-practice/03-senior-build-projects.md).
3. **Idempotency audit:** for each endpoint in [TodoItems.cs](../../src/Web/Endpoints/TodoItems.cs) and [TodoLists.cs](../../src/Web/Endpoints/TodoLists.cs), mark retry-safe or not, and why. (DELETE is idempotent in effect but returns 204 even for already-deleted? No — check: `Guard.Against.NotFound` makes the *second* delete a 404. Effect-idempotent, response-non-idempotent. That nuance is a strong-answer marker.)

## Interview angle

- *"How do you reliably do X after Y commits?"* → outbox narrative, grounded in the interceptor's before-save tradeoff. [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md)
- *"What is idempotency and where does it matter in your app?"* → per-verb audit above. [08-interview-prep/03, Q6](../08-interview-prep/03-api-and-data-modeling-questions.md)
- *"Tell me about a subtle bug pattern with events."* → ghost effects / dual-write, with this repo's line numbers as evidence.
