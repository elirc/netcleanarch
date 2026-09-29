# ADR-008: Functional Tests Run Against a Real Database at the `ISender` Seam

**Status:** Accepted (implicit). Depends on the authors' explicit [ADR-002 (Aspire)](../docs/decisions/ADR-002-Aspire-For-Orchestration-And-Testing.md), which provides the orchestration this strategy relies on.

## Context

Because handlers use EF Core directly with no repository ([ADR-004](adr-004-efcore-in-application-no-repository.md)), they can't be unit-tested against a mocked data layer. And mocked/in-memory databases lie: they accept queries the real provider can't translate, ignore constraints, and fake transaction semantics. The team needed tests that exercise the *real* query translation, validators, pipeline, and interceptors.

## Decision

Run the primary suite ([Application.FunctionalTests](../tests/Application.FunctionalTests/FunctionalTestSetup.cs)) against a **real database**, orchestrated by Aspire's `TestAppHost`, entering the system at `ISender` — [TestApp.SendAsync](../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L16-L23) — so tests drive real commands through the real pipeline. Between tests, **Respawn resets data, not schema** ([DatabaseResetter L40-L45](../tests/Application.FunctionalTests/Infrastructure/DatabaseResetter.cs#L40-L45)). The only mocked seam is `IUser` ([WebApiFactory L17-L28](../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28)) — swappable precisely because the Application layer depends on the interface, not `HttpContext`. Test setup builds state *through commands* ([UpdateTodoItemTests L20-L38](../tests/Application.FunctionalTests/TodoItems/Commands/UpdateTodoItemTests.cs#L20-L38)) so tests can't create impossible states. This sits inside a four-layer strategy (Domain unit, Application unit, functional, Playwright acceptance).

## Alternatives Considered

- **Mock `IApplicationDbContext` / EF in-memory provider.** Fast, hermetic, and *wrong* — green tests over broken SQL. Explicitly rejected; a real production incident (an untranslatable LINQ query that passed against the in-memory double) motivated the switch.
- **Container-per-test isolation.** Cleaner isolation, far slower. Rejected in favor of Respawn's fast data reset.
- **Transaction-rollback-per-test.** Breaks the moment code under test commits its own transaction (which every handler does). Rejected.

## Consequences

**Positive**
- Tests catch provider-specific translation failures, constraint violations, and interceptor behavior — the bugs mocks hide.
- The audit and event interceptors are exercised for real; `CreatedBy` assertions ([CreateTodoItemTests L36-L45](../tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs#L36-L45)) pass because the mocked `IUser` feeds the real interceptor.
- `AssertConfigurationIsValid()` ([MappingTests L30-L34](../tests/Application.UnitTests/Common/Mappings/MappingTests.cs#L30-L34)) turns AutoMapper drift into a build failure.

**Negative**
- **No parallelism.** The "current user" lives in *static* fields ([TestApp L13-L14](../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L13-L14)) and Respawn resets shared state, so the suite must run serially. A newcomer who sees "flaky" tests and enables parallelism would *cause* the cross-talk they imagined.
- **Infrastructure dependency.** No database, no functional tests; container DB variants require Docker for `dotnet test`, and a cold container pull can blow the 60s setup budget ([FunctionalTestSetup L17](../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L17)).
- **Coverage seam:** functional tests enter *below* the endpoints, so endpoint code (id-consistency guard, status mapping, binding) is untested here — and the acceptance suite has no `Todo.feature`. The todo endpoints' HTTP behavior is covered by neither suite.

## Evidence

[FunctionalTestSetup L14-L46](../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L14-L46), [WebApiFactory L17-L28](../tests/Application.FunctionalTests/Infrastructure/WebApiFactory.cs#L17-L28), [DatabaseResetter L24-L45](../tests/Application.FunctionalTests/Infrastructure/DatabaseResetter.cs#L24-L45), the statics at [TestApp L13-L14](../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L13-L14), and the testing rationale in the authors' [ADR-001](../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md).
