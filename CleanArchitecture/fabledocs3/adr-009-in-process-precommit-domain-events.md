# ADR-009: Domain Events Dispatched In-Process, Pre-Commit, via a SaveChanges Interceptor

**Status:** Accepted (implicit). The related decision to reference `MediatR.Contracts` from Domain *is* documented — [ADR-003](../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md) — but the *dispatch mechanism and timing* below are not.

## Context

The domain needs to signal that something happened ("a todo item was completed") so other code can react without the trigger knowing who's listening. Those events must reach handlers reliably relative to the database write that caused them.

## Decision

Entities raise events into an in-memory collection — the [TodoItem.Done setter L14-L26](../src/Domain/Entities/TodoItem.cs#L14-L26) adds a `TodoItemCompletedEvent` *only* on the false→true transition, an invariant encoded so no call site can forget it. Events are `INotification`s ([BaseEvent](../src/Domain/Common/BaseEvent.cs), enabled by the `MediatR.Contracts` reference of ADR-003). A **SaveChanges interceptor publishes them** — [DispatchDomainEventsInterceptor L25-L30](../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L30) — during `SavingChangesAsync`, i.e. **before** the rows are committed. Handlers run in-process, synchronously, via `_mediator.Publish`.

This is one of two cross-cutting concerns hung on the save seam via interceptors (the other is audit stamping, [AuditableEntityInterceptor](../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs)), both registered at [Infrastructure/DependencyInjection.cs L20-L21](../src/Infrastructure/DependencyInjection.cs#L20-L21).

## Alternatives Considered

- **After-commit dispatch (`SavedChangesAsync`).** Database truth first, then events. Inverts the failure mode: save succeeds, handler fails, event lost. A legitimate choice with a different risk profile; the authors chose before-commit for atomicity with same-context handler writes.
- **Transactional outbox** (persist events in the same transaction, deliver from a background worker with retries, idempotent consumers). The industry-standard answer for *reliable external* side effects. Not implemented — appropriate given every current handler only logs, but it's the upgrade path the moment a handler does I/O.
- **A separate `IDomainEvent` marker + adapter** instead of referencing MediatR from Domain. Rejected in the authors' [ADR-003](../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md) as an adapter that papers over identical semantics.

## Consequences

**Positive**
- A handler that writes to the *same* `DbContext` joins the same `SaveChanges` and commits atomically with the trigger — no dual-write for the common in-DB case.
- The transition invariant lives in the setter, so events fire correctly from every path including seeds and tests.

**Negative**
- **Pre-commit dispatch means external side effects escape the transaction.** If a handler sends an email and the save then fails, you've sent a ghost notification; if a handler throws, the save aborts and the user's action fails (today the only handler just logs — [LogTodoItemCompleted](../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs) — so this is latent).
- **"Events" are synchronous method calls in disguise:** `Publish` awaits all handlers inline, so their latency and failures land on the user's request. There is no delivery guarantee beyond at-most-once, in-process, lost on crash.
- **Sync-over-async landmine:** [line 19](../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L19) blocks on async work via `.GetAwaiter().GetResult()` on the synchronous save path — harmless under modern ASP.NET Core's lack of a synchronization context, a deadlock elsewhere.

## Evidence

[TodoItem.Done setter L14-L26](../src/Domain/Entities/TodoItem.cs#L14-L26), [DispatchDomainEventsInterceptor L25-L49](../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs#L25-L49) (note events are collected, cleared to prevent re-entrancy, then published), interceptor registration at [Infrastructure/DependencyInjection.cs L20-L21](../src/Infrastructure/DependencyInjection.cs#L20-L21), and the authors' [ADR-003](../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md).
