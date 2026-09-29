# Observability & Operations

Observability answers one question: *when this breaks in production, how will I know, and how fast can I find why?* This repo has a strong telemetry foundation and some deliberate gaps.

## What's wired

| Signal | Mechanism | Anchor |
|---|---|---|
| Structured logs | `ILogger` with message templates + destructuring (`{@Request}`, `{@UserId}`) | [LoggingBehaviour](../../src/Application/Common/Behaviours/LoggingBehaviour.cs#L32-L33), [PerformanceBehaviour](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L48-L49), [UnhandledExceptionBehaviour](../../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L25) |
| Metrics | OpenTelemetry: ASP.NET Core, HttpClient, runtime instrumentation | [ServiceDefaults L52-L58](../../src/ServiceDefaults/Extensions.cs#L52-L58) |
| Distributed tracing | OTel tracing (ASP.NET Core + HttpClient sources) | [ServiceDefaults L59-L66](../../src/ServiceDefaults/Extensions.cs#L59-L66) |
| Export | OTLP when `OTEL_EXPORTER_OTLP_ENDPOINT` set (Aspire dashboard sets it in dev) | [ServiceDefaults L73-L80](../../src/ServiceDefaults/Extensions.cs#L73-L80) |
| Health checks | `/health` (all checks) + `/alive` (liveness tag), **Development-gated** | [ServiceDefaults L92-L118](../../src/ServiceDefaults/Extensions.cs#L92-L118) |
| Perf watchdog | requests >500ms log a warning with user context | [PerformanceBehaviour L37-L50](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L37-L50) |

Every MediatR request auto-logs on entry ([LoggingBehaviour as a pre-processor](../../src/Application/Common/Behaviours/LoggingBehaviour.cs#L7)), and every unhandled exception logs with the request payload before rethrow. Combined with per-request traces, you can follow one user action across the stack in the Aspire dashboard — that's the payoff of the behaviour pattern for observability.

## "How would I know this broke?" — per flow

| Flow | Signal you'd watch | Blind spot |
|---|---|---|
| [Load Todo page (1)](../01-codebase-cartography/05-key-flows.md) | trace span for `GetTodosQuery`; error rate on `/api/TodoLists` | client-side `.catch(console.error)` failures never reach the server — a broken page can be *invisible* to backend telemetry |
| [Create list (2)](../01-codebase-cartography/05-key-flows.md) | 400 rate (validation), 500 rate (colour gap) | which validation rule fired isn't a metric — you'd grep logs |
| [Login (3)](../01-codebase-cartography/05-key-flows.md) | auth failure rate, `manage/info` probe volume | no metric distinguishes "wrong password" from "expired session" |
| [Complete item / events (4)](../01-codebase-cartography/05-key-flows.md) | the info log from [LogTodoItemCompleted](../../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs#L17) | **no signal if an event handler silently misbehaves** — events are fire-and-forget in-process |
| [Startup/DB (5)](../01-codebase-cartography/05-key-flows.md) | startup logs, health checks | health checks are **dev-only** — production has no `/health` unless you enable it (the code even warns about this, [L103-L104](../../src/ServiceDefaults/Extensions.cs#L101-L104)) |

## Gaps and the operations story

- **No business metrics.** You can see request rates and latencies, not "todo items completed per hour." Adding a custom `Meter`/counter in the event handler is a clean, high-signal ticket.
- **No health check for the database.** `/health` only checks liveness (`self`, [L96](../../src/ServiceDefaults/Extensions.cs#L92-L96)); a dead DB wouldn't fail readiness. `AddDbContextCheck<ApplicationDbContext>()` (package already referenced, [Directory.Packages.props L23](../../Directory.Packages.props#L23)) closes it.
- **Health checks gated to Development** — intentional (they can leak internals) but means production readiness/liveness for a load balancer needs explicit, secured enablement. Know the tradeoff the comment describes.
- **PII/secret log risk** from `{@Request}` (see [security checklist](05-security-checklist.md)) — an observability feature that becomes a liability as commands grow.
- **Deploy/rollback:** the repo has no CD or migration story ([architecture critique P2](../03-architecture-and-patterns/06-architecture-critique.md)). Rollback of *code* is redeploy-previous; rollback of *schema* doesn't exist because migrations don't. Operationally, that means today's app can't be safely evolved in production — the thing to build first.

## The three questions to ask of any system (transferable)

1. **Detection:** what fires an alert when this fails? (Here: error-rate and latency from OTel; nothing for silent client failures or event-handler misbehavior.)
2. **Diagnosis:** given an alert, can I find the cause without adding code? (Here: yes for server flows — trace + request-logged exception; no for client-only failures.)
3. **Recovery:** what's the rollback, and has it been rehearsed? (Here: code redeploy yes; schema no.)

Bring these three to any on-call or design conversation and you sound like someone who's been paged before.

## Drill

Design the observability for the domain-event flow so a silently-failing email handler (hypothetical) would page someone. Sketch: a counter `todo_events_published_total` vs `todo_events_handled_total` (divergence = dropped events), a handler-duration histogram, and an alert on handler error rate. Then note *why the current before-save, in-process design makes some of this impossible* (a handler that throws takes down the request instead of incrementing a failure counter — you'd need the outbox first, [reliability module](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)). Strong answers connect observability limits back to architecture.

## Interview angle

- *"How would you know if feature X broke in production?"* → the per-flow table; name the blind spots unprompted. [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md)
- *"Logs, metrics, traces — when each?"* → this repo has all three; give the one-line purpose of each with an anchor.
- *"What's your rollback strategy?"* → the honest code-yes/schema-no answer + what you'd build.
