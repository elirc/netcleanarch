# System Design From This Repo

The move: reverse-engineer this codebase into a whiteboard exercise. You already know a working design *and its tradeoffs* — that's a huge advantage over reasoning from nothing. At each step, "what this repo actually chose (with an anchor)" plus "a stronger/simpler alternative."

## The prompt: "Design a todo-list application from scratch."

### Step 1 — Requirements (2 min; always start here)
Functional: users create lists and items; edit, complete, delete; organize by colour/priority. Non-functional: multi-user with data isolation, auth, reasonable latency, evolvable schema.
This repo chose: the functional set, minus isolation (single logical user) — a scope decision, not an oversight. **Junior** dives into tables immediately; **mid** enumerates requirements first; **senior** asks clarifying questions ("how many users? shared lists? real-time? mobile?") and states assumptions before drawing.

### Step 2 — API sketch
Resource endpoints: `GET/POST/PUT/DELETE /api/TodoLists`, same for items, plus an aggregate read.
This repo chose: a single [GetTodos](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs) returning lists+items+lookup data in one call ([TodoLists.cs](../../src/Web/Endpoints/TodoLists.cs)). Stronger/simpler alternative: separate endpoints per resource (more RESTful, more round trips) vs the aggregate VM (fewer round trips, coupled to this UI). **Mid** picks one and justifies; **senior** names the coupling cost of the aggregate and when you'd split it (multiple clients with different needs).

### Step 3 — Data model
Entities: `TodoList (Id, Title, Colour, audit)`, `TodoItem (Id, ListId, Title, Note, Priority, Done, audit)`, one-to-many with cascade ([data model](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
This repo chose: Colour as an owned value object ([TodoListConfiguration](../../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs#L15-L16)), int PKs, no indexes beyond keys, no concurrency token. Stronger alternative for scale: add `OwnerId` + index, a unique index on title, a rowversion for concurrency. **Senior** calls out the missing ownership column as *the* design gap and the arbitrary item ordering.

### Step 4 — Architecture / layering
This repo chose: Clean Architecture — Domain/Application/Infrastructure/Web with CQRS-lite + MediatR pipeline ([system map](../01-codebase-cartography/01-system-map.md)). Simpler alternative: a two-layer CRUD app (controllers → DbContext) — less ceremony, fine for this size; the layering pays off when cross-cutting concerns and testability matter. **Mid** can describe the layers; **senior** can argue *whether they're worth it here* (the honest answer: as a template teaching the pattern, yes; as a 4-endpoint todo app in isolation, it's more structure than the domain needs — and saying that is the senior signal).

### Step 5 — Scaling concerns
This repo chose: not to address them (single instance, unbounded queries, in-process events). At scale you'd add: pagination ([M3](../06-contribution-practice/02-mid-level-feature-tickets.md)), caching of static lookup data, read replicas, an outbox for reliable events ([S1](../06-contribution-practice/03-senior-build-projects.md)), a message broker, horizontal scaling (stateless API + shared DB, cookie auth needs a shared data-protection key across instances — a real gotcha). **Senior** volunteers the data-protection-key detail and the unbounded-query problem before being pushed.

### Step 6 — Tradeoffs, stated out loud
The repo's own ADRs *are* the tradeoff analysis: no repository over EF ([ADR-001](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md)), Aspire over docker-compose ([ADR-002](../../docs/decisions/ADR-002-Aspire-For-Orchestration-And-Testing.md)), a Domain dependency for convenience ([ADR-003](../../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md)). Citing "I'd document this as an ADR with the Harder section filled in" is a strong closing move.

---

## Variation 1: "Now add multi-tenancy / per-user isolation."
The single most likely follow-up (it's the repo's biggest gap). Walk it: add `OwnerId`/`TenantId` → **EF global query filter** so reads scope automatically → write-side ownership check → **404 not 403** for foreign resources (don't confirm existence) → cross-user denial tests. Hard parts to volunteer: injecting the current user into a once-built model ([kata 2](../06-contribution-practice/04-refactor-and-design-katas.md)), that query filters don't guard by-key `FindAsync` writes, and migration/backfill of existing rows. This is [ticket M1](../06-contribution-practice/02-mid-level-feature-tickets.md) — do that ticket's design note and you can whiteboard this cold.

## Variation 2: "Now support real-time collaboration."
SignalR hub broadcasting changes, triggered from domain events; client reconciles its optimistic cache against server pushes (fixes the [cache-drift](../02-stack-and-language-mastery/02-framework-mental-models.md) in Todo.jsx); **the hub must honor the same ownership authorization as the API** (a user can't subscribe to another's list). Scaling: per-list groups not global broadcast; a backplane (Redis) for multi-instance. This is [S3](../06-contribution-practice/03-senior-build-projects.md).

## Variation 3: "Now handle 10x traffic."
Measure first (where's the bottleneck?). Then: pagination for unbounded reads, cache the static lookup data (colours/priorities never change), the per-request username-lookup removal ([performance](../05-quality-engineering/04-performance-thinking.md)), read replicas for the read-heavy `GetTodos`, horizontal API scaling (stateless — mind the shared data-protection key for cookies), CDN for the SPA. Say "I'd load-test to find the real limit before optimizing" — measuring-first is a senior reflex.

## Variation 4: "Now send an email when a task is completed."
The reliability story. Naive: the existing [domain-event handler](../../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs) sends the email — but events dispatch *before commit* in-process ([reliability](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)), so a failed save sends a ghost email and a crash loses it (dual-write). Upgrade: transactional outbox — persist the event in the trigger's transaction, a background dispatcher delivers with retries/backoff, consumers are idempotent (at-least-once). This is [S1](../06-contribution-practice/03-senior-build-projects.md); it's also the most common senior system-design prompt in disguise.

## Variation 5: "Now make it work offline / add a mobile client."
Bearer tokens instead of cookies (the template already supports this variant, [Infrastructure/DI L46-L56](../../src/Infrastructure/DependencyInjection.cs#L46-L56)) — contrast the security tradeoffs ([03-architecture/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)); offline sync needs a client-side queue of mutations + conflict resolution (the concurrency-token gap becomes central); the generated-client approach ([contract chain](../02-stack-and-language-mastery/03-type-system-and-contracts.md)) extends to a mobile client from the same OpenAPI doc.

---

## What junior / mid / senior answers sound like (the meta-rubric)

- **Junior:** jumps to tables and endpoints; describes *a* solution; struggles when asked to change it.
- **Mid:** requirements first; justifies each choice; can compare two options; handles one follow-up variation.
- **Senior:** asks clarifying questions; states assumptions; volunteers the gaps and failure modes *before* being asked; changes the design gracefully under new constraints; closes with "here's what I'd document and what I'd measure."

## Drill

Do the full 6-step walkthrough out loud in 20 minutes, then have a friend (or yourself) throw two random variations. Record yourself once — you'll hear where you stated a choice without its tradeoff. The anchors mean you can always ground a claim; the skill is *reaching for the tradeoff every time*, which is what [08-interview-prep/README's golden rule](README.md) drills.
