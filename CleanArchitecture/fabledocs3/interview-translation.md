# Interview Translation

For each ADR: a **90-second spoken answer** (how you'd actually say it out loud — roughly 160–190 words, conversational, leading with the decision, always naming a tradeoff and a failure mode) and the **follow-up question** an interviewer is most likely to ask next. The golden rule throughout: answer with a concrete example, a tradeoff, and a failure mode — never a definition.

---

## ADR-001 — Clean Architecture layering

**90-second answer.** "The codebase is four projects — Domain, Application, Infrastructure, Web — with dependencies pointing strictly inward. The move that makes it work is that inner layers *define* the interfaces they need and outer layers implement them. My favorite example is `IUser`: it's two properties, defined in Application, implemented in Web from `HttpContext`. So the business layer knows *who's acting* without knowing HTTP exists — and that same seam is what lets tests swap in a fake user and keep everything else real. The payoff is that swapping the database provider touches one file in Infrastructure; nothing inner knows which database runs. The cost is honesty: the 'Domain depends on nothing' ideal isn't quite true — it references EF's abstractions and a MediatR contracts package, and there's even a persistence attribute leaking into a base entity. And the dependency rule is invisible when you break it — the compiler stays green; you find out weeks later when a unit test suddenly drags in the whole stack."

**Likely follow-up.** "You said Domain references EF Core abstractions — doesn't that violate the dependency rule?" (Answer: a *reference to an assembly* isn't a *dependency on an implementation*; the direction of the arrow is what matters, and it still points inward.)

---

## ADR-002 — CQRS-lite via MediatR

**90-second answer.** "Every use case is a command or a query — a small message object with exactly one handler — dispatched through MediatR instead of living in a big service class. The real reason isn't fashion; it's that reads and writes want opposite things. A read uses `AsNoTracking` and projects straight to a DTO in one SQL query; a write loads a tracked entity and mutates it. Modeling them as separate objects lets each do what it needs, and it gives me a uniform shape that a cross-cutting pipeline can wrap. The tradeoff is class proliferation — a trivial operation still needs a command, a handler, a validator, an endpoint, and a test; there's literally a scaffolding template because the boilerplate is real. And the indirection hurts navigation: 'go to definition' on `Send` tells you nothing, you navigate by the request type. For a small CRUD service with no cross-cutting needs I'd probably skip it; here the pipeline earns it."

**Likely follow-up.** "When would you *not* use MediatR?" (Answer: a handful of endpoints with no shared cross-cutting concerns — the ceremony buys nothing; a plain service is clearer.)

---

## ADR-003 — Behaviour pipeline for cross-cutting concerns

**90-second answer.** "Logging, authorization, validation, and performance timing are all implemented once, as MediatR pipeline behaviours that wrap every request in a deliberate order. Order is a *correctness* property, not a preference — authorization runs before validation, so an anonymous caller never gets validation feedback that leaks which resources exist. The big win is that authorization is an *application* concern, not an HTTP filter, so it still fires when a test calls the handler directly with no web server — we actually have a test that proves an anonymous user is denied with zero HTTP. The cost is hidden coupling: nothing in a handler's file shows the five wrappers around it, and reordering them silently changes behavior. And the sharpest wart lives here — authorization only checks 'are you authenticated and in a role,' never 'is this resource yours.' There's no ownership column anywhere, so any logged-in user can read anyone's data by guessing IDs. That's fine for a single-user template and a critical IDOR bug in any multi-user fork."

**Likely follow-up.** "How would you add per-user data isolation without editing every handler?" (Answer: an EF global query filter keyed on the current user so reads are scoped structurally, plus a write-side ownership check, returning 404 not 403 for foreign resources.)

---

## ADR-004 — EF Core in Application, no repository

**90-second answer.** "We put EF Core directly in the Application layer behind an interface the layer owns — `IApplicationDbContext`, which exposes `DbSet`s — and deliberately skipped the repository pattern. The reasoning is that repositories don't actually remove the ORM coupling, they hide it: you still end up with methods like `GetActiveOrderedWithItems`, which is an EF query wearing a trench coat, plus a layer to maintain. Dependency inversion still holds because the interface is defined inward. The honest cost is testing — you can't mock a repository, so handlers need a real database to test, which drove our whole testing strategy. And an ORM swap would be an Application-layer rewrite, not an Infrastructure swap — I accept that because greenfield projects basically never replace their ORM. The one place I'd reverse this is a true DDD model: there, repositories are a domain concept expressed in aggregate terms, and that's a different decision for different reasons — not 'hiding EF.'"

**Likely follow-up.** "Doesn't skipping repositories make your Application layer un-unit-testable?" (Answer: yes for the data path — so we test it with a *real* database instead of pretending; that's ADR-008.)

---

## ADR-005 — Minimal APIs with reflection registration

**90-second answer.** "The HTTP layer is Minimal APIs, one class per resource implementing a shared interface, and a startup routine reflects over the assembly and registers each one as a route group automatically. So adding an endpoint group is zero registration edits — the convention finds it, names the route, and tags it in OpenAPI. Handlers stay thin: they bind, send a MediatR message, and translate the result to a typed HTTP result. The benefit is consistency by construction and no drift between a registration list and the handlers. The cost is that the convention is load-bearing and *invisible* — nothing in the class says 'I'm reachable at this route,' and there's a real gotcha where marking a group `internal` instead of `public` silently drops it from the API because the reflection only sees exported types. I chose it over controllers because the endpoints here have no business logic to justify controller weight."

**Likely follow-up.** "What happens when two endpoint classes have the same name?" (Answer: both map to `/api/{SameName}` and collide at registration — the convention has no disambiguation.)

---

## ADR-006 — Contract-first client generation

**90-second answer.** "Both frontends generate their typed API clients from the server's OpenAPI document with NSwag, regenerated on every build, so a contract change becomes a client *build* failure instead of a runtime surprise. The subtle decision underneath is that the OpenAPI operationId is derived from the endpoint handler's *method name* — so `GetTodoLists` in C# becomes `getTodoLists()` in TypeScript. That keeps the names in lockstep automatically. The tradeoff that bites is that a method rename now looks local but is a breaking API change: it renames the generated client method and breaks every call site in both SPAs and any external consumer. I actually saw this cause an incident — a 'harmless' backend rename turned the frontend build red because the two ran on separate pipelines. If I were doing a public-facing API, I'd pin explicit operationIds instead and accept the drift risk, because 'your rename broke three teams' is worse than 'a string went stale and a test caught it.'"

**Likely follow-up.** "How would you make a safe rename, then?" (Answer: keep the operationId stable with an explicit `.WithName("GetTodoLists")` so the C# identifier can change without touching the contract.)

---

## ADR-007 — Centralized error contract

**90-second answer.** "There's one error contract for the whole API. Handlers throw typed exceptions — not found, unauthorized, validation — and a single exception handler maps each to an RFC ProblemDetails response with the right status. Validation is special: it runs as a pipeline stage that collects *every* failure and returns them together, so a bad form comes back with all three errors at once instead of one per round trip. What I like is that it's not dogmatic — where a failure is *expected* rather than exceptional, like a weak password on user creation, the code returns a `Result` object instead of throwing. Exceptions for the exceptional, results for the expected. The wart is the fall-through: unmapped exceptions become 500, which is right for a null-reference but *wrong* for a domain invariant we forgot to map — a bad colour code currently returns 500 instead of 400. That's reachable via the API even though the UI can't trigger it, and it's a good reminder that the API is the contract, not the UI."

**Likely follow-up.** "Why is throwing for not-found better than returning a result there?" (Answer: it isn't universally — it's better *here* because not-found is exceptional relative to the handler's happy path and a central mapper keeps handlers clean; expected failures still use `Result`.)

---

## ADR-008 — Real-database functional testing

**90-second answer.** "The main test suite runs against a *real* database, not a mock, entering at the MediatR sender so it drives real commands through the real pipeline, validators, and interceptors. We reset data between tests with Respawn rather than rebuilding the schema, and we mock exactly one thing — the current user. The reason is scar tissue: mocked and in-memory databases lie. We had a query that passed against the in-memory double and failed against the real provider because it couldn't translate the LINQ — the test gave us confidence we hadn't earned. The cost is real: the suite can't run in parallel because the current user is stored in static fields and Respawn resets shared state, so a well-meaning engineer who enables parallelism to 'fix flakiness' would actually *cause* cross-talk. And there's a coverage seam — tests enter below the endpoints, so endpoint logic like the id-consistency check isn't covered by this suite at all."

**Likely follow-up.** "How would you make that suite run in parallel?" (Answer: first remove the static user state — thread a scoped user context — then isolate data per test, e.g. schema- or database-per-worker; parallelism before that is unsafe.)

---

## ADR-009 — In-process, pre-commit domain events

**90-second answer.** "Entities raise domain events into an in-memory list — the item's `Done` setter adds a 'completed' event only on the false-to-true transition, so no call site can forget it — and a SaveChanges interceptor publishes them through MediatR. The important detail is *timing*: they're dispatched *before* the commit. The upside is that a handler writing to the same DbContext joins the same transaction and commits atomically. The downside, which is the whole distributed-systems conversation, is that a handler doing something *external* — sending an email — happens before we know the transaction will commit, so a failed save means a ghost email, and a throwing handler aborts the save. Today every handler just logs, so it's latent. The moment one sends email, the right answer is a transactional outbox: persist the event in the same transaction, deliver it from a background worker with retries, make consumers idempotent. Right now these 'events' are really synchronous method calls wearing event clothing — at-most-once, in-process, lost on crash."

**Likely follow-up.** "Walk me through the outbox — why does it fix this?" (Answer: the event is written in the same transaction as the change, so capture is atomic; a separate dispatcher delivers at-least-once with retries, which is why consumers must be idempotent.)

---

## ADR-010 — Schema by convention, no migrations

**90-second answer.** "In development the app drops and recreates the database from the EF model on every run, then reseeds — there are no migrations at all, and the pending-model-changes warning is actually suppressed. For a template that's the right call: clone and run, instant seeded database, zero migration ceremony to learn, and one model targets SQLite, Postgres, and SQL Server behind build switches. But I'm very clear-eyed that this is the decision you *reverse first* in any real fork, because as shipped this codebase literally cannot evolve a production database — `EnsureCreated` and migrations are mutually exclusive. The failure modes are that every dev run wipes local data, which surprises newcomers, and that suppressing the drift warning turns off the exact alarm that would tell you the model diverged. Job zero on a production fork is: generate an initial migration, switch startup to migrate instead of recreate, and un-suppress that warning so drift is visible again."

**Likely follow-up.** "How do you change a schema in production with zero downtime, then?" (Answer: expand–contract — add a nullable column, deploy code that reads either shape, backfill, tighten, then drop the old column; never rename in one step, and rollback is usually roll-forward.)

---

## Using this in a loop

If you get one system-design or deep-dive slot, the three richest answers to lead with are **ADR-003** (authorization gap → global query filter is a complete security story), **ADR-009** (pre-commit events → outbox is the canonical reliability story), and **ADR-010** (no migrations → expand–contract is the canonical schema-evolution story). Each one lets you demonstrate the junior→senior gradient in a single breath: name the decision, name what it costs, name the failure mode, and name the fix.
