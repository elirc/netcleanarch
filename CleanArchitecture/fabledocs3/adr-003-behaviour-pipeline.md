# ADR-003: Cross-Cutting Concerns as an Ordered MediatR Behaviour Pipeline

**Status:** Accepted (implicit).

## Context

Logging, input validation, authorization, performance timing, and unhandled-exception logging apply to nearly every use case. Duplicating them in each handler is error-prone (someone forgets validation on a new command) and noisy. They need to live in exactly one place and apply uniformly.

## Decision

Wrap every MediatR request in a pipeline of `IPipelineBehavior` implementations, registered — and therefore executed — in a deliberate order. From [Application/DependencyInjection.cs L16-L23](../src/Application/DependencyInjection.cs#L16-L23):

1. `LoggingBehaviour` (a request pre-processor) — [logs every request on entry](../src/Application/Common/Behaviours/LoggingBehaviour.cs).
2. `UnhandledExceptionBehaviour` — [logs any exception with the request, then rethrows](../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L15-L28).
3. `AuthorizationBehaviour` — [enforces `[Authorize]` on request records](../src/Application/Common/Behaviours/AuthorizationBehaviour.cs).
4. `ValidationBehaviour` — [runs all FluentValidation validators, aggregates failures, throws once](../src/Application/Common/Behaviours/ValidationBehaviour.cs#L15-L33).
5. `PerformanceBehaviour` — [warns on requests over 500ms](../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L37-L50).

The order is load-bearing: exception logging wraps authorization; authorization runs before validation so anonymous callers don't get validation feedback that could leak which resources exist; performance timing is innermost so it measures only the handler.

## Alternatives Considered

- **ASP.NET Core MVC filters / middleware.** Would tie cross-cutting logic to the HTTP transport — but these concerns are *application* concerns that must also apply to non-HTTP entry points (tests via `ISender`, a future queue consumer). Rejected: the [functional test that denies an anonymous user](../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L50-L58) proves authorization works with zero HTTP involved — only possible because it's a *behaviour*, not a filter.
- **Base handler classes / manual decorators.** Rigid inheritance or hand-wired decorators per handler. Rejected — open generics register once and apply to everything.

## Consequences

**Positive**
- One place per concern; new use cases inherit all five for free.
- Application-layer authorization survives transport changes and is directly testable.
- Aggregated validation returns *every* input error at once — one round trip, not one-error-at-a-time.

**Negative**
- **Hidden coupling.** Nothing in a handler's file reveals the five wrappers around it; you must know the pipeline exists. Reorder it and semantics change silently.
- **I/O in the pipeline.** `LoggingBehaviour` and `PerformanceBehaviour` each do a DB query to turn a user id into a username ([LoggingBehaviour L27-L30](../src/Application/Common/Behaviours/LoggingBehaviour.cs#L27-L30)), so *every authenticated request* pays a round trip for a log label.
- **Authorization is coarse — and this is the biggest architectural gap in the codebase.** The behaviour answers "authenticated?" and "in role/policy?" ([AuthorizationBehaviour L24-L74](../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L24-L74)) but there is **no resource-level ownership check** anywhere; queries aren't filtered by owner ([GetTodos L43-L47](../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47)) and no entity carries an owner column. Any authenticated user can read or mutate any other user's data by guessing integer IDs (classic IDOR). Deliberate for a single-user template; a critical vulnerability in any multi-user fork. There's also a latent sharp edge: the role string is split on `,` with no `Trim()` ([L41-L45](../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L41-L45)), so `"Admin, User"` would never match — *investigate*, not confirmed (no live caller passes such a string).

## Evidence

[Application/DependencyInjection.cs L16-L23](../src/Application/DependencyInjection.cs#L16-L23) for the registration/execution order; the five files under [src/Application/Common/Behaviours/](../src/Application/Common/Behaviours/AuthorizationBehaviour.cs); the ownership gap is provable by *absence* — no `OwnerId`, no per-user `Where` clause in any handler.
