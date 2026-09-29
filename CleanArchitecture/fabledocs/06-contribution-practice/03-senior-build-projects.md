# Senior Build Projects

Five projects, 2 days to 4 weeks. Each has a full senior checklist: problem, product value, design decisions, likely files, migration plan, test plan, security plan, performance plan, rollout/rollback, open questions, stretch. These are the artifacts you'd bring to a design review — and, reworked, to a system-design interview.

---

## S1: Transactional Outbox for domain events (1–2 weeks)

Problem: domain events dispatch **in-process, before commit** ([DispatchDomainEventsInterceptor](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs)) — the moment a handler does external I/O (email, webhook), you have the dual-write problem ([reliability](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).
Product value: reliable "notify on X" without losing events on crash or firing ghosts on rollback.
Design decisions: (1) `OutboxMessages` table (Id, Type, Payload, OccurredOn, ProcessedOn, Attempts, Error). (2) Interceptor *persists* events to the outbox in the **same transaction** as the trigger — atomic capture. (3) A `BackgroundService` polls unprocessed rows, deserializes, publishes via MediatR, marks processed; exponential backoff + max attempts + dead-letter. (4) Consumers must be **idempotent** (at-least-once delivery — dedupe by message id).
Likely files: new outbox entity + config; rewrite [DispatchDomainEventsInterceptor](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs) to write rows not publish; new dispatcher service registered in [Infrastructure/DI](../../src/Infrastructure/DependencyInjection.cs); serialization of `BaseEvent` subtypes.
Migration plan: additive table; keep in-process dispatch behind a flag until the dispatcher is proven; cut over per event type.
Test plan: functional test — trigger event, assert outbox row written in the same transaction (rollback the trigger → no row); dispatcher test — row processed once, retried on failure, dead-lettered after N.
Security plan: payloads may contain entity data — redaction / no secrets in events.
Performance plan: poll interval vs latency; index on `ProcessedOn IS NULL`; batch dispatch.
Rollout/rollback: flag-gated; rollback = flip flag to in-process.
Open questions: serialize whole entity (as [TodoItemCompletedEvent](../../src/Domain/Events/TodoItemCompletedEvent.cs) holds) or a snapshot DTO? (Snapshot — the entity may have changed by dispatch time.) Ordering guarantees needed?
Stretch: swap polling for a message broker; add an admin view of dead-lettered messages.
Interview story potential: "designed a transactional outbox to make event delivery reliable and idempotent" — a canonical senior system-design story.

## S2: Notifications feature end to end (2–3 weeks)

Problem: nothing notifies users of anything; a realistic feature exercising every layer.
Product value: "email me when a shared list changes / item assigned to me."
Design decisions: depends on **S1** (reliable delivery) and **M1** (ownership/sharing) — sequencing is itself a senior decision. New domain events (`ItemAssignedEvent`), an `INotificationService` (Application interface) + provider (Infrastructure), user notification preferences.
Likely files: new events in Domain; handlers in Application; email provider in Infrastructure behind an interface (the [IUser pattern](../03-architecture-and-patterns/01-boundaries-and-layers.md)); preferences on `ApplicationUser`.
Migration/test/security/perf: preferences column additive; test with a fake provider recording sends; **never** put provider credentials in code (Key Vault path exists, [Web/DI L39-L48](../../src/Web/DependencyInjection.cs#L39-L48)); rate-limit outbound email.
Rollout/rollback: dark-launch with sends disabled; enable per user.
Open questions: templating? unsubscribe/compliance? digest vs immediate?
Interview story potential: "shipped notifications on top of an outbox; sequenced it behind the reliability and ownership work rather than building on sand."

## S3: Real-time updates via SignalR (1–2 weeks)

Problem: multiple users (post-M1 sharing) see stale lists until refresh; the React app hand-maintains a cache with no server push ([framework mental models](../02-stack-and-language-mastery/02-framework-mental-models.md)).
Product value: collaborative live updates.
Design decisions: SignalR hub broadcasting list/item changes; trigger from domain events (again, S1 helps); client reconciles its optimistic cache with pushed truth (this directly attacks the cache-drift problem in [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx)); group per list/owner for scoping.
Test plan: hub integration test; client reconciliation logic.
Security plan: hub authorization must respect M1 ownership (a user can't subscribe to another's list) — auth on the hub, not just the API.
Performance plan: broadcast scope (per-list groups, not global); connection scaling (backplane for multi-instance).
Rollout/rollback: progressive; fall back to polling.
Open questions: conflict resolution when optimistic client and push disagree? (Server wins; define "server truth" moment.)
Interview story potential: "added real-time collaboration and made the hub honor the same authorization boundary as the REST API."

## S4: Multi-tenancy (organizations) (3–4 weeks)

Problem: M1 gives per-*user* isolation; many products need per-*organization* with roles inside an org.
Product value: teams sharing lists with member/admin roles.
Design decisions: `Organization` + `Membership(role)` entities; tenant resolution (from user's active org claim); query filters keyed on `TenantId` (generalize M1's filter); org-scoped authorization policies feeding [AuthorizationBehaviour's policy path](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L62-L74).
Migration plan: large — every domain table gets `TenantId`; backfill existing rows into a default org; expand–contract per table.
Test/security/perf: cross-tenant denial tests (the M1 recipe, generalized); tenant filter must be un-bypassable (audit every query path); composite indexes on `(TenantId, ...)`.
Rollout/rollback: enormous blast radius — flag per tenant, shadow-read validation.
Open questions: tenant per row vs schema vs database (the classic three-way — argue the tradeoffs)? cross-tenant admin?
Interview story potential: "led a multi-tenancy migration; chose row-level tenancy with query filters and expand–contract migrations, and validated isolation with denial tests" — a staff-level story.

## S5: Production readiness hardening (1 week)

Problem: the template is dev-optimized ([architecture critique](../03-architecture-and-patterns/06-architecture-critique.md)); make one fork deployable.
Product value: a fork you could actually ship.
Bundle of design decisions: migrations (M9) + DB health check + tightened CORS ([Program.cs L28-L31](../../src/Web/Program.cs#L28-L31)) + rate limiting (M7) + log redaction (M8) + gating OpenAPI/Scalar out of production ([Program.cs L35-L36](../../src/Web/Program.cs#L35-L36)) + secret management verification + a real seed strategy (no hardcoded admin password, [initialiser L83](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L83)).
Migration/test/security/perf: each sub-item as its own reviewable PR; a "production readiness" checklist as the acceptance criteria; load test the tightened config.
Rollout/rollback: staged per concern; each independently revertable.
Open questions: which of these does the *upstream* template want vs which are fork-only? (Most are fork-only — the template values simplicity; the deliverable is partly a *written recommendation* to the maintainer.)
Interview story potential: "took a template from dev-only to production-ready and documented which hardening belonged upstream vs in our fork."

---

## Drill

Pick **S1** and write its one-page RFC using the [RFC template](../07-career-and-collaboration/02-writing-prs-and-rfcs.md). Then defend, in three sentences each: why the outbox write must share the trigger's transaction, why consumers must be idempotent, and why you'd keep the in-process path behind a flag during rollout. Strong answers can also state what you'd measure to declare the migration successful (zero lost events under induced crash; dispatch latency within budget). This RFC *is* your answer to the interview prompt "design a reliable way to send email when something happens."
