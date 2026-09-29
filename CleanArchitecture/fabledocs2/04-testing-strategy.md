# 4. Testing Strategy

## What it does and why it exists

Four layers, each entering the system at a different seam:

| Project | Enters at | Real infrastructure | Catches | Blind to |
|---|---|---|---|---|
| [Domain.UnitTests](../tests/Domain.UnitTests/ValueObjects/ColourTests.cs) | plain objects | none | invariant logic | everything else |
| [Application.UnitTests](../tests/Application.UnitTests/Common/Behaviours/RequestLoggerTests.cs) | one class + mocks | none | behaviour logic, mapping config | SQL translation, pipeline composition |
| [Application.FunctionalTests](../tests/Application.FunctionalTests/FunctionalTestSetup.cs) | `ISender` | **real database** (Aspire + Respawn) | validators, pipeline order, interceptors, real SQL | HTTP: routing, binding, status codes |
| [Web.AcceptanceTests](../tests/Web.AcceptanceTests/AspireSetup.cs) | real browser (Playwright + Gherkin) | full stack incl. SPA | end-to-end wiring, auth cookies | precise failure location |

The organizing principle: **each layer's blind spot is another layer's target.** `ShouldDenyAnonymousUser` in [GetTodosTests L50-L58](../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L50-L58) proves the *authorization behaviour* rejects, with no HTTP involved. [Login.feature](../tests/Web.AcceptanceTests/Features/Login.feature) proves the *cookie flow* works in a real browser. Neither alone proves both. When you decide where a test goes, the rule is: **test at the lowest layer that can observe the failure you actually fear.** Push a fear down a layer and it's faster but fictional; push it up and it's real but slow and vague.

## The hardest decisions

### Decision 1: Functional tests run against a real database, not mocks

The functional suite boots a real database via Aspire ([FunctionalTestSetup L14-L46](../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L14-L46)) and resets *data* (not schema) between tests with Respawn ([DatabaseResetter L40-L45](../tests/Application.FunctionalTests/Infrastructure/DatabaseResetter.cs#L40-L45)). Tests drive real MediatR commands through the real pipeline, real validators, real interceptors, real SQL — entering at `ISender` via [TestApp.SendAsync](../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L16-L23).

The rejected alternative was the fast, conventional one: mock `IApplicationDbContext` (or use the EF in-memory provider) so tests never touch a real database. I rejected it because **the mocks lied to us.** An in-memory or mocked context happily accepts queries the real provider can't translate, ignores constraints, and fakes transaction semantics. We had tests that were green against mocks and code that was broken against the actual database. That's the worst kind of test — one that gives you confidence you haven't earned. This is also *why* we chose not to have a repository layer ([section 1](01-architecture.md)); the two decisions reinforce each other, and Microsoft's own guidance points the same way (cited in [ADR-001](../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md)).

**The tradeoff:** functional tests are now slower (they boot infrastructure) and there is *no lightweight substitute*. If the database isn't available, the functional suite can't run at all. For a template that ships with SQLite this is cheap; for a fork on SQL Server it means Docker is a hard dependency of `dotnet test`. I decided catching real-database bugs was worth the boot cost. I still think that's right, but you should know you're paying it.

### Decision 2: The suite can't run in parallel, and I let it

Look at [TestApp L13-L14](../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L13-L14):

```
private static string? _userId;
private static List<string>? _roles;
```

The "current user" for a test is stored in **static fields**, which the mocked [IUser](../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28) reads from. `RunAsDefaultUserAsync` sets them; `ResetState` clears them between tests ([TestApp L82-L91](../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L82-L91)). Static process-global state means **two tests running at once would stomp on each other's identity.** So the suite runs serially. Respawn's per-test data reset assumes the same thing — one test touching the database at a time.

The alternative was per-test isolation that supports parallelism: a scoped user context threaded through, a database-per-test or schema-per-test strategy. It's more machinery, and for a template the simplicity of "statics + serial" won.

**The tradeoff that nags:** serial execution puts a ceiling on how big this suite can grow before it's annoyingly slow, and it's a trap for a new engineer who sees "flaky tests," assumes cross-talk, and enables parallelism to "fix" it — which would *cause* the cross-talk they imagined. If you ever reach for `[Parallelizable]` in this project, stop: the statics in `TestApp` and the shared Respawn reset make it unsafe. The right fix is to remove the statics first. I documented this here precisely because the failure mode is so unintuitive.

### Decision 3: Setup goes *through* commands, and we test one seam of the app, not all of it

Functional tests build their world by *sending commands*, not by inserting rows behind the app's back — see [UpdateTodoItemTests L20-L38](../tests/Application.FunctionalTests/TodoItems/Commands/UpdateTodoItemTests.cs#L20-L38), which creates a list and an item through `CreateTodoListCommand`/`CreateTodoItemCommand` before testing the update. That's deliberate: setup-through-commands can't create *impossible* states, because it goes through the same validation and rules as production.

But notice the seam. Functional tests enter at `ISender` — *below* the endpoints. That means **the endpoint code is not covered by functional tests at all**: the id-consistency guard from [section 2](02-api-design.md), the status-code mapping, the model binding — none of it. That's the acceptance layer's job, and the acceptance layer only covers Home/Counter/Weather/Login (there's no `Todo.feature`). So there's a real coverage seam: the todo endpoints' HTTP behavior is tested by *neither* the functional nor the acceptance suite.

**The wart:** I'm comfortable with the *strategy* (test each layer at its seam) but the *execution* has a hole, and it's exactly the kind of hole that's invisible until it isn't — someone breaks the id-consistency check, every test stays green, and a client gets a 500 where they expected a 400. If I were prioritizing, adding a `Todo.feature` acceptance test is the highest-value test work in the repo.

## Evidence to open

- [tests/Application.FunctionalTests/FunctionalTestSetup.cs](../tests/Application.FunctionalTests/FunctionalTestSetup.cs) — Aspire boots the DB once per run.
- [tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs L17-L28](../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28) — the *one* seam we mock: `IUser`. Everything else is real. This is only possible because `Application` depends on the interface, not on `HttpContext`.
- [tests/Application.FunctionalTests/Infrastructure/TestApp.cs](../tests/Application.FunctionalTests/Infrastructure/TestApp.cs) — the test façade, and the statics that forbid parallelism.
- [tests/Application.UnitTests/Common/Mappings/MappingTests.cs L30-L34](../tests/Application.UnitTests/Common/Mappings/MappingTests.cs#L30-L34) — `AssertConfigurationIsValid()`. One test that turns "we forgot to update the AutoMapper profile" from a runtime surprise into a build failure. My favorite test in the repo.
- [tests/Domain.UnitTests/ValueObjects/ColourTests.cs](../tests/Domain.UnitTests/ValueObjects/ColourTests.cs) — and note it *relies on* the constructor bypass that [section 5](05-error-handling.md) calls a bug. Our own tests exercise the loophole.

## The incident this prevented

Before the functional suite existed against a real database, we had a handler that filtered with a LINQ expression the EF provider couldn't translate. Against the in-memory test double it evaluated client-side and passed every test. In production, against the real provider, it either threw a translation exception or — worse, on an older EF version — silently pulled the whole table into memory and filtered there, turning a should-be-indexed lookup into a full scan. The first sign was a latency alarm, not a test failure, because *the tests couldn't see it.*

That's the incident that made me rip out the mocks and stand up the Aspire + Respawn approach. The functional tests now run the real query against the real provider, so an un-translatable expression fails *in CI*, loudly, at the layer that can actually observe it. The durable lesson: **a test that runs against a fake of the thing you're worried about isn't testing the thing you're worried about.** When you write a data-access test, ask what specifically you're afraid of — if the fear is "does this query actually work against our database," a mock cannot answer it, by definition.

## Three questions before you move on

1. `ShouldDenyAnonymousUser` makes no HTTP call, yet it proves authorization rejects an anonymous user. How? Name the exact seam the test enters at and the class that does the rejecting.
2. You inherit a "flaky" failure where two functional tests occasionally corrupt each other's data. A colleague suggests marking them `[Parallelizable]` to speed things up and shake out the flake. Why is that exactly backwards, and what's the real root cause you'd look for first?
3. I break the id-consistency check in [TodoItems.cs](../src/Web/Endpoints/TodoItems.cs) so `PUT /api/TodoItems/5` with a body id of 6 silently updates item 6. Which of our four test layers catches this today? (Be honest.) Where would you add the test that *should* catch it?
