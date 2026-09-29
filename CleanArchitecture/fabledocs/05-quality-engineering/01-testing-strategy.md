# Testing Strategy

## The four layers and what each one is *for*

| Layer | Project | Enters at | Real infra | Speed | Catches | Blind to |
|---|---|---|---|---|---|---|
| Domain unit | [Domain.UnitTests](../../tests/Domain.UnitTests/ValueObjects/ColourTests.cs) | plain objects | none | ms | invariant logic (Colour rules, equality) | everything else |
| Application unit | [Application.UnitTests](../../tests/Application.UnitTests/Common/Behaviours/RequestLoggerTests.cs) | one class + mocks | none | ms | behaviour logic, mapping config validity | SQL translation, pipeline composition |
| Functional | [Application.FunctionalTests](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs) | `ISender` | real DB (Aspire+Respawn) | s | validators, pipeline order, interceptors, real SQL, constraints | HTTP: routing, binding, status codes, cookies |
| Acceptance | [Web.AcceptanceTests](../../tests/Web.AcceptanceTests/AspireSetup.cs) | real browser (Playwright + Reqnroll/Gherkin) | full AppHost incl. SPA | min | wiring end to end, auth cookies, UI regressions | precise failure location (everything's in the pot) |

The strategic idea: each layer's blind spot is another layer's target. `ShouldDenyAnonymousUser` ([functional](../../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L50-L58)) proves the *behaviour* rejects; [Login.feature](../../tests/Web.AcceptanceTests/Features/Login.feature) proves the *cookie flow* works; neither alone proves both. When you join any repo, build this table for it — the empty cells are your risk map.

**What belongs at each layer** (the decision rule): test at the *lowest* layer that can observe the failure you fear. Fear "Colour.From accepts junk" → domain unit. Fear "the unique rule doesn't query right" → functional (needs real SQL). Fear "login sets a cookie the SPA can use" → acceptance. Pushing a fear *down* a layer makes it faster but fictional; *up*, real but slow and vague.

**What not to test:** framework behavior (EF save mechanics, Identity hashing), generated code (`web-api-client.ts`), trivial mapping already covered by [MappingTests.ShouldHaveValidConfiguration](../../tests/Application.UnitTests/Common/Mappings/MappingTests.cs#L30-L34) — one config-validity test replaces dozens of per-property assertions. That test is a gem: `AssertConfigurationIsValid` fails when any DTO property lacks a mapped source, converting "we forgot to update the profile" from a runtime surprise into a test failure.

## Isolation mechanics (why these tests don't flake much)

- **Data isolation:** Respawn wipes tables before every test ([TestBase](../../tests/Application.FunctionalTests/Infrastructure/TestBase.cs) → [DatabaseResetter L40-L45](../../tests/Application.FunctionalTests/Infrastructure/DatabaseResetter.cs#L40-L45)). Reset-data-not-schema is the fast middle ground between per-test containers (slow) and shared state (flaky).
- **Identity isolation:** user statics cleared per test ([TestApp.ResetState L82-L91](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L82-L91)); the cost is **no parallelism** — statics are process-global. A deliberate simplicity/speed trade; know it exists before you blame "flaky tests" for cross-talk that's actually parallel execution someone enabled.
- **Time:** the app abstracts time (`TimeProvider`, [DI L76](../../src/Infrastructure/DependencyInjection.cs#L76)) but tests don't exploit it — they assert with tolerance instead: `Created.ShouldBe(DateTime.Now, TimeSpan.FromMilliseconds(10000))` ([CreateTodoItemTests L42](../../tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs#L36-L45)). A ±10s window is honest about clock skew but weak (would pass with a 9s-wrong clock). Injecting a fake `TimeProvider` in the factory would make it exact — a nice quality ticket.
- **Randomness:** [GetWeatherForecasts](../../src/Application/WeatherForecasts/Queries/GetWeatherForecasts/GetWeatherForecastsQuery.cs#L16) uses raw `Random` — untestable-by-design demo code; notice no test touches it.
- **Mock discipline:** exactly **one** seam is mocked in functional tests (`IUser`, [WebApiFactory L17-L28](../../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28)). Compare with the unit layer, where mocks are free ([RequestLoggerTests](../../tests/Application.UnitTests/Common/Behaviours/RequestLoggerTests.cs#L16-L22)). Rule of thumb worth quoting: *mock at boundaries you own, only the ones the test's fear doesn't concern.*

## Fixtures and builders

There's no builder library; tests construct entities inline via collection initializers ([GetTodosTests L26-L40](../../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L26-L40)) or drive creation through *commands* ([UpdateTodoItemTests L22-L31](../../tests/Application.FunctionalTests/TodoItems/Commands/UpdateTodoItemTests.cs#L20-L38)) — the latter is subtly better: setup goes through validation/business rules, so tests can't create impossible states. When setup-through-commands gets verbose, that's the signal to add a builder — not before.

## Coverage gaps worth knowing (verified by absence)

No test: asserts `TodoItemCompletedEvent` reaches its handler; exercises cross-user denial (no ownership feature to test — yet); covers the [Colour constructor bypass](../../src/Domain/ValueObjects/Colour.cs#L3) as a *problem* (tests actually rely on it, [ColourTests L51-L58](../../tests/Domain.UnitTests/ValueObjects/ColourTests.cs#L51-L58)); covers `UpdateTodoItemDetail` moving an item between lists; touches the React app at unit level (zero JS tests in ClientApp-React; Angular has token spec files, e.g. [counter.component.spec.ts](../../src/Web/ClientApp/src/app/counter/counter.component.spec.ts)); runs the empty [Infrastructure.IntegrationTests](../../tests/Infrastructure.IntegrationTests/GlobalUsings.cs) placeholder.

## Drill

For each fear, name the layer and sketch the test in two lines: (a) "deleting a list orphans its items," (b) "the 400 body shape changed and broke the React error parsing," (c) "priority levels lookup no longer matches the enum." Answers: (a) functional — create list+items via commands, delete, `CountAsync<TodoItem>` == 0 (cascade); (b) that's a *contract* test — acceptance level, or a functional test asserting the serialized ProblemDetails shape through a real HTTP call via the factory's client (the suite currently never uses HTTP — noticing that is Strong); (c) application unit — compare `Enum.GetValues` against the handler's lookup list.

## Interview angle

- *"Describe your testing strategy"* — the four-layer table with the blind-spot framing. → [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md)
- *"Mocks vs real database?"* — one-seam discipline + [ADR-001's](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md) argument.
- *"How do you keep integration tests fast and stable?"* — Respawn reset, boot-once, no-parallel tradeoff.
