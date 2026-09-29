# Pattern Catalog

Fourteen cards. The goal is **recognition** — seeing the shape in any repo and knowing its failure modes — not name-dropping. Each card: problem → shape → two real anchors → why it works → failure modes → when to use/avoid → interview angle → drill.

---

## Pattern 1: CQRS-lite (command/query objects)

Problem it solves: use cases tangled into fat services; reads and writes with conflicting needs (projection vs tracking) forced through one shape.
General shape: each operation is a message object (`XCommand`/`XQuery`) with exactly one handler.
Real example: [CreateTodoItemCommand + handler](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs)
Second example: [GetTodosQuery + handler](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs)
Why this implementation works: message = contract = wire format; handlers stay ~20 lines; reads use `AsNoTracking`+projection while writes use tracking — different needs, cleanly separated.
Failure modes: class explosion for trivial CRUD; anemic handlers that are just DbContext pass-throughs; teams misreading "CQRS" as requiring separate read/write databases (this is the *lite* form — same DB).
Use it when: cross-cutting pipeline concerns exist; use cases are the unit of testing. Avoid when: a 5-endpoint microservice where the ceremony outweighs the wins.
Interview angle: "What did CQRS buy you and what's the difference from full CQRS/event sourcing?"
Drill: count the files needed to add one new command here (command+handler file, validator, endpoint method, test — 4). Is that cost justified in this repo? Argue both ways.

## Pattern 2: Pipeline behaviours (chain of responsibility / middleware)

Problem it solves: logging, auth, validation, timing copy-pasted into every handler.
General shape: open-generic wrappers around handlers, executed in registration order.
Real example: registration order — [Application/DependencyInjection.cs L16-L23](../../src/Application/DependencyInjection.cs#L16-L23)
Second example: [ValidationBehaviour](../../src/Application/Common/Behaviours/ValidationBehaviour.cs) (representative of all five)
Why this implementation works: order encodes policy — exceptions logged (UnhandledException) around auth, auth before validation (don't leak validation info to anonymous callers), performance timing innermost so it measures only the handler.
Failure modes: order changes silently changing semantics (swap auth/validation and anonymous users learn which titles exist); behaviours doing I/O per request (see PerformanceBehaviour's username lookup); hidden coupling — nothing in a handler's file reveals the pipeline around it.
Use it when: ≥2 cross-cutting concerns across many handlers. Avoid when: only one handler needs the concern — just write it there.
Interview angle: "How do you handle cross-cutting concerns without base classes?" and "why does registration order matter?"
Drill: write the order you'd choose from scratch, then justify each adjacency against the real order.

## Pattern 3: Dependency inversion via inward-owned interfaces

Problem it solves: business logic welded to frameworks (HTTP, Identity, ORM concrete types).
General shape: inner layer defines the interface it *wishes* existed; outer layer implements.
Real example: [IUser](../../src/Application/Common/Interfaces/IUser.cs) ← [CurrentUser](../../src/Web/Services/CurrentUser.cs)
Second example: [IIdentityService](../../src/Application/Common/Interfaces/IIdentityService.cs) ← [IdentityService](../../src/Infrastructure/Identity/IdentityService.cs)
Why this implementation works: interfaces are *minimal* (IUser = 2 properties) — shaped by what the consumer needs, not what the provider offers. Tests exploit the seam directly ([WebApiFactory L17-L28](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28)).
Failure modes: header interfaces (1:1 mirror of the implementation, inversion theater); interfaces owned by the wrong side; abstraction so leaky the "swap" it promises is fiction (see Card 4's honest answer to that).
Use it when: the dependency is a *policy/mechanism* boundary or a test seam. Avoid when: you'd abstract something that has exactly one conceivable implementation and no test value.
Interview angle: "Explain dependency inversion with a concrete example" — IUser in 30 seconds beats any UML.
Drill: design `IEmailSender` for this codebase: minimal surface, owner layer, registration site.

## Pattern 4: Interface-exposed DbContext (the anti-repository decision)

Problem it solves: repositories that add indirection while still leaking ORM semantics.
General shape: Application defines `IApplicationDbContext` exposing `DbSet<T>`; handlers query EF directly.
Real example: [IApplicationDbContext](../../src/Application/Common/Interfaces/IApplicationDbContext.cs)
Second example: any handler, e.g. [DeleteTodoList L16-L27](../../src/Application/TodoLists/Commands/DeleteTodoList/DeleteTodoList.cs#L16-L27)
Why this implementation works: full LINQ/projection power in handlers; one less layer to keep in sync; honesty about the EF dependency. The tradeoff is *documented* in [ADR-001](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md) — including when it's wrong (real DDD aggregates want repositories as a domain concept).
Failure modes: query logic duplicated across handlers with no home for reuse; handlers becoming untestable without a real DB (this repo accepts that and tests against a real DB); ORM swap becomes an Application-wide rewrite (accepted as YAGNI).
Use it when: CRUD-heavy, EF-committed, real-DB testing. Avoid when: rich aggregates with invariants that must load as a unit.
Interview angle: "Do you use the repository pattern?" — the strongest possible answer is this ADR's argument, either side.
Drill: find a query worth centralizing if ownership filtering lands everywhere (hint: "lists visible to user X" would appear in 6+ handlers).

## Pattern 5: Value Object

Problem it solves: primitive obsession — `string colour` accepting `"banana"`.
General shape: immutable type, equality by components, validating factory.
Real example: [Colour](../../src/Domain/ValueObjects/Colour.cs) over [ValueObject](../../src/Domain/Common/ValueObject.cs)
Second example: no second value object exists — itself a finding; `Title` (validated to 200 chars in three places) is a candidate.
Why this implementation works: `Colour.From` centralizes the invariant; owned-entity mapping ([TodoListConfiguration L15-L16](../../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs#L15-L16)) persists it without an FK table; equality-by-value makes `SupportedColours.Contains(colour)` work ([L9](../../src/Domain/ValueObjects/Colour.cs#L9)).
Failure modes: **the public constructor bypasses the factory** — `new Colour("junk")` is legal ([L3](../../src/Domain/ValueObjects/Colour.cs#L3)), and the repo's own tests do it ([ColourTests L51-L58](../../tests/Domain.UnitTests/ValueObjects/ColourTests.cs#L51-L58)); the odd empty→`#000000` coercion ([L31](../../src/Domain/ValueObjects/Colour.cs#L31)) silently manufactures an *unsupported* colour. An invariant with a side door isn't an invariant.
Use it when: a primitive has rules or units. Avoid when: truly free-form data.
Interview angle: "What's a value object vs an entity?" — identity vs value equality, plus this bypass story as your failure-mode flourish.
Drill: make the constructor private on paper; list everything that breaks (the tests at L51-L58, the static factories — fixable how?).

## Pattern 6: Domain events

Problem it solves: "when X happens, also do Y" without the X-code knowing Y exists.
General shape: entity records facts; infrastructure publishes them; handlers subscribe.
Real example: raised — [TodoItem.Done setter](../../src/Domain/Entities/TodoItem.cs#L14-L26); carried — [DispatchDomainEventsInterceptor](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs); handled — [LogTodoItemCompleted](../../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs)
Second example: the machinery is generic — [BaseEntity.AddDomainEvent](../../src/Domain/Common/BaseEntity.cs#L16-L19) works for any future event.
Why this implementation works: the transition check (false→true only) lives *in the setter* — impossible to forget at any call site, including seeds and tests.
Failure modes: dispatch-before-save consequences (full analysis in [04-side-effects](04-side-effects-async-and-reliability.md)); "events" that are really synchronous calls with extra steps; event handlers mutating unrelated aggregates until no one can trace causality.
Use it when: genuine fan-out or decoupling need. Avoid when: one caller, one effect — call the method.
Interview angle: "How do domain events reach their handlers, and what are the delivery guarantees?" (Here: in-process, at-most-once, pre-commit. Know all three words.)
Drill: add (on paper) a `TodoListDeletedEvent`: where raised, what handler, what goes wrong if the handler is slow?

## Pattern 7: EF SaveChanges interceptors

Problem it solves: per-entity bookkeeping (audit fields, event dispatch) duplicated in every handler.
General shape: hook the unit-of-work seam; apply policy to the ChangeTracker.
Real example: [AuditableEntityInterceptor](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs) (base-class-driven: anything `BaseAuditableEntity` gets stamped)
Second example: [DispatchDomainEventsInterceptor](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs); both registered [Infrastructure/DependencyInjection.cs L20-L25](../../src/Infrastructure/DependencyInjection.cs#L20-L25)
Why this implementation works: one enforcement point; impossible to "forget" auditing on a new entity type; interceptors resolve scoped services (`IUser`, `TimeProvider`) cleanly.
Failure modes: invisible behavior — a reader of the handler sees no audit code and may re-implement it; ordering between multiple interceptors is implicit; heavy logic at the save seam slows *every* write.
Use it when: policy applies to a whole class of entities. Avoid when: logic is specific to one use case — that belongs in the handler.
Interview angle: "How would you add created/modified auditing across 40 entities?" — this exact pattern.
Drill: [AuditableEntityInterceptor.HasChangedOwnedEntities](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs#L57-L64) — explain in one sentence why it exists (changing only `Colour` — an owned entity — must still bump the list's `LastModified`).

## Pattern 8: Guard clauses

Problem it solves: arrow-shaped null/validity checks; inconsistent not-found handling.
General shape: fail fast at method entry with intention-revealing one-liners.
Real example: `Guard.Against.NotFound(request.Id, entity)` — [UpdateTodoItem L28](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L28)
Second example: connection-string guard — [Infrastructure/DependencyInjection.cs L18](../../src/Infrastructure/DependencyInjection.cs#L18); anonymous-handler guard — [EndpointRouteBuilderExtensions L22](../../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs#L22)
Why this implementation works: the thrown `NotFoundException` participates in the global error contract (→404 via [ProblemDetailsExceptionHandler L24-L30](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L24-L30)) — a guard library and an exception mapper *co-designed*.
Failure modes: guards as flow control for expected business outcomes (a "not found" that's actually a normal empty state); exception cost on hot paths; swallowing the guard's specific exception type upstream.
Use it when: preconditions whose violation is exceptional. Avoid when: the caller legitimately branches on the outcome — return a result type instead.
Interview angle: "Exceptions vs result types for error handling?" — this repo is the exceptions-camp specimen; know the result-object counterargument ([Result](../../src/Application/Common/Models/Result.cs) exists here for Identity operations — the repo uses *both*, chosen per audience!).
Drill: articulate why `CreateUserAsync` returns `Result` ([IIdentityService L13](../../src/Application/Common/Interfaces/IIdentityService.cs#L13)) while `UpdateTodoItem` throws. (User-facing expected failures vs programming-contract violations.)

## Pattern 9: Exception-to-ProblemDetails mapping

Problem it solves: every endpoint hand-rolling error responses; error shape drift.
General shape: throw typed exceptions anywhere; one `IExceptionHandler` maps type→status→RFC 9457/9110 body.
Real example: [ProblemDetailsExceptionHandler](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44)
Second example: the contract side — [ApiExceptionOperationTransformer](../../src/Web/Infrastructure/ApiExceptionOperationTransformer.cs) documents those statuses in OpenAPI.
Why this implementation works: one switch expression is the entire error policy; clients can rely on one shape ([Todo.jsx parses it, L77-L83](../../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83)).
Failure modes: unmapped exceptions default to 500 (the `UnsupportedColourException` gap); over-mapping (exposing internals in `Detail`); handler ordering issues if multiple `IExceptionHandler`s exist.
Use it when: always, honestly. This should be your default in any ASP.NET Core API.
Interview angle: "How do you standardize API errors?" — ProblemDetails + this mapping is the canonical answer.
Drill: add (on paper) the `UnsupportedColourException → 400` case; what Title/Type would you give it, and should Domain exception types be referenced by name in Web? (They must be — Web references Domain transitively; is that OK? Yes: outer may know inner.)

## Pattern 10: Convention-based endpoint registration

Problem it solves: forgetting to register routes; inconsistent route naming.
General shape: reflection finds every `IEndpointGroup`; class name → route prefix + OpenAPI tag; method name → operationId.
Real example: [WebApplicationExtensions.MapEndpoints](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28)
Second example: consumption — [Program.cs L45](../../src/Web/Program.cs#L45); contract — [IEndpointGroup](../../src/Web/Infrastructure/IEndpointGroup.cs)
Why this implementation works: adding a class adds routes — zero registration drift; naming conventions become *load-bearing* and therefore consistent.
Failure modes: renames become breaking API changes (method name = operationId = generated client method); discoverability — nothing in the class says "I am reachable at /api/X" except convention knowledge; reflection cost at startup (negligible here).
Use it when: many uniform endpoint groups. Avoid when: routes need per-endpoint nuance that fights the convention.
Interview angle: "Convention vs configuration — where's the line?" Use the rename-blast-radius as your cost example.
Drill: what happens if two endpoint classes share a name in different namespaces? Read [L14-L25](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L14-L25) and predict (both map to `/api/{SameName}` — collision at route registration).

## Pattern 11: DB-level projection (`ProjectTo`)

Problem it solves: loading full entity graphs to return three fields; N+1 lazy-load storms.
General shape: expression-based mapping composed *into* the `IQueryable` so SQL selects only DTO columns.
Real example: [GetTodos L43-L47](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47) with profiles in [TodoListDto](../../src/Application/TodoLists/Queries/GetTodos/TodoListDto.cs#L20-L27)/[TodoItemDto](../../src/Application/TodoLists/Queries/GetTodos/TodoItemDto.cs#L19-L26)
Second example: mapping validity is *tested* — [MappingTests.ShouldHaveValidConfiguration](../../tests/Application.UnitTests/Common/Mappings/MappingTests.cs#L30-L34), which catches DTO/entity drift at test time instead of runtime.
Why this implementation works: nested `Items` collection is included in the same translated query — no N+1; `AsNoTracking` compounds the saving.
Failure modes: un-translatable mapping expressions silently failing or client-evaluating; AutoMapper's magic hiding *which* columns a query touches (grep the profile, always); profile drift when entities change (mitigated by the test above).
Use it when: read paths returning DTOs. Avoid when: the mapping needs behavior/conditionals — write the `Select` by hand; explicit beats magic at equal length.
Interview angle: "How do you avoid over-fetching and N+1 in EF?" → this card is the answer.
Drill: hand-write the LINQ `Select` equivalent of the TodoListDto projection. Notice what AutoMapper was saving you — and what it was hiding.

## Pattern 12: Generated typed API client

Problem it solves: hand-written fetch wrappers drifting from the server contract.
General shape: OpenAPI document → codegen (NSwag) → typed client classes; regenerate on every build.
Real example: [nswag.json](../../src/Web/ClientApp-React/nswag.json) + `prestart` hook ([package.json L27-L35](../../src/Web/ClientApp-React/package.json#L27-L35))
Second example: consumption — [Todo.jsx L3-L6](../../src/Web/ClientApp-React/src/components/Todo.jsx#L3-L6); Angular twin uses the same generator with DI-injected clients ([todo.component.ts L2-L6, L35-L38](../../src/Web/ClientApp/src/app/todo/todo.component.ts#L2-L6)).
Why this implementation works: contract drift becomes a *build failure* on the typed side; server annotations ([EndpointSummary etc.](../../src/Web/Endpoints/TodoItems.cs#L21-L22)) flow into docs and clients.
Failure modes: generated-file edits lost on regen; stale generation order (needs the .NET build first); the untyped `.jsx` seam means drift still slips through in this repo's React app; `"nswag": "latest"` unpinned.
Use it when: FE and BE evolve together. Avoid when: public API for third parties — then the *spec* is the product and you version it deliberately.
Interview angle: "How do you keep client and server in sync?" → [02-stack/03](../02-stack-and-language-mastery/03-type-system-and-contracts.md) has the full chain.
Drill: find the three `prestart`/`prebuild` hooks across both clients and CI that keep generation fresh; what dev mistake breaks each?

## Pattern 13: Real-database functional testing with reset

Problem it solves: mocked-DB tests that pass while the real query fails (translation, constraints, transactions).
General shape: boot real infra once per run (Aspire), reset *data* not *schema* between tests (Respawn), drive tests through the real composition.
Real example: [FunctionalTestSetup](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs) + [DatabaseResetter](../../tests/Application.FunctionalTests/Infrastructure/DatabaseResetter.cs)
Second example: the seam swap that keeps it hermetic — [WebApiFactory](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs) replacing only `IUser`.
Why this implementation works: per-test cost is one fast delete-based reset, not a container boot; tests exercise validators, pipeline, interceptors, and SQL translation together.
Failure modes: static state forbidding parallelism ([TestApp L13-L14](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L13-L14)); suite-boot flakiness under the 60s timeout; blind spot at the HTTP layer (endpoints untested here — acceptance tests own that).
Use it when: the DB is where your risk lives (usually true). Avoid when: pure logic — unit test it (this repo does: Domain.UnitTests).
Interview angle: "Unit vs integration testing for data access?" → cite [ADR-001's testing rationale](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md).
Drill: explain why Respawn reset beats (a) dropping/recreating the DB per test, (b) transactions-rolled-back-per-test. (Speed; and rollback isolation breaks when code under test commits.)

## Pattern 14: Orchestration as code (Aspire)

Problem it solves: "works on my machine" — README steps, hand-managed containers, hardcoded ports.
General shape: a C# program declares resources (DB, API, SPA), dependencies (`WaitFor`), and wiring (connection strings, env vars); the same graph serves dev, tests, and deployment manifests.
Real example: [AppHost/Program.cs](../../src/AppHost/Program.cs)
Second example: scoped-down test variant — [TestAppHost](../../tests/TestAppHost/Program.cs) (DB only); consumed programmatically by tests ([FunctionalTestSetup L20-L41](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L20-L41)).
Why this implementation works: services reference each other by name ([Shared/Services.cs](../../src/Shared/Services.cs)); SPA proxies get URLs from injected env vars instead of hardcoding; `dotnet test` is self-contained ([ADR-002](../../docs/decisions/ADR-002-Aspire-For-Orchestration-And-Testing.md) lists the rejected alternatives — read it).
Failure modes: coupling to a Microsoft-specific orchestrator; local resource weight; two AppHosts to keep consistent.
Use it when: multi-resource .NET solutions. Avoid when: single self-contained service — `dotnet run` was already enough.
Interview angle: "How does your team handle local environments and integration-test infra?" — compare Aspire vs docker-compose vs Testcontainers, which is literally ADR-002's rationale section.
Drill: trace how the connection string travels from AppHost to `AddDbContext` (resource reference → env var `ConnectionStrings__CleanArchitectureDb` → `builder.Configuration.GetConnectionString` at [Infrastructure/DependencyInjection.cs L17](../../src/Infrastructure/DependencyInjection.cs#L17)).

---

## Meta-drill

Pick any unfamiliar repo (tomorrow's interview take-home, say). Give yourself 45 minutes to fill in a blank card for three patterns you find. The card template *is* the reading method: problem → shape → anchors → failure modes. If you can't fill "failure modes," you've recognized the name, not the pattern.
