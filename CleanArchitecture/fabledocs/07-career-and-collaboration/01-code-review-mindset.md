# Code Review Mindset

## The five layers (review in this order)

1. **Does it work?** — does it do what the PR claims, at all?
2. **Is it correct?** — edge cases, error paths, concurrency, the failure modes.
3. **Will it stay correct?** — tests, invariants encoded so the *next* change can't break it silently.
4. **Does it fit?** — follows this repo's patterns, layers, conventions.
5. **Is it kind to the next maintainer?** — readable, documented where non-obvious, no traps.

Most junior reviews stop at layer 1 and nitpick style (layer 5) while missing layers 2–4. The value is in the middle.

## A repo-specific review checklist

Layer-fit questions unique to this codebase — run them on any PR here:

- **Thin endpoints?** Business logic belongs in a handler, not the endpoint ([TodoLists pattern](../../src/Web/Endpoints/TodoLists.cs)). An endpoint doing more than bind/send/translate is a smell.
- **Validation in the right place?** Shape/contract rules → FluentValidation; invariants → Domain ([validation stack](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)). New string input → bounded in *both* validator and DB config.
- **New exception types mapped?** Anything that can escape a handler needs an arm in [ProblemDetailsExceptionHandler](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs), or it's a 500.
- **Layer direction intact?** No `Infrastructure`/`HttpContext` references creeping into Application or Domain ([boundaries](../03-architecture-and-patterns/01-boundaries-and-layers.md)).
- **Public-contract blast radius?** Renamed endpoint method / changed DTO → regenerated clients change → SPA call sites break ([contract chain](../02-stack-and-language-mastery/03-type-system-and-contracts.md)). Are the client changes in the same PR?
- **Ownership/auth (post-M1)?** New query scoped to the user? New command field server-controlled? ([security checklist](../05-quality-engineering/05-security-checklist.md)).
- **Tested at the right layer?** ([testing strategy](../05-quality-engineering/01-testing-strategy.md)) — lowest layer that observes the fear.
- **Warnings-as-errors respected?** ([Directory.Build.props L5](../../Directory.Build.props#L5)) — the build fails on warnings; don't `#pragma` them away without a reason (the one existing suppression, [GetWeatherForecastsQuery L12](../../src/Application/WeatherForecasts/Queries/GetWeatherForecasts/GetWeatherForecastsQuery.cs#L12), is demo code).
- **Optimistic UI has a rollback?** New client mutation → snapshot + revert on failure ([contrast 5](../04-code-reading-gym/03-fake-code-contrasts.md)).

## Comment craft: observation → consequence → suggestion

Bad: "This is wrong." / "Don't do this." / "Use a DTO."
Good, three examples in this repo's context:

> The `DueDate` field is a non-nullable `DateTime`, so existing items and any created without it default to `0001-01-01`, which the UI will render as a real date. `DateTimeOffset?` matches the audit-field convention and makes "no deadline" representable — want me to point at `BaseAuditableEntity` for the type?

> Returning the tracked `TodoList` entity here will serialize the audit fields and can cycle through `Items` → `List`. The repo's convention is a projected DTO (see `GetTodos`) — returning a `TodoListDto` keeps the wire contract stable. Happy to pair on the projection if useful.

> Small thing (non-blocking): this catch only `console.error`s, so a failed save leaves the checkbox showing the wrong state. The `commitNewList` handler snapshots and restores on failure — mirroring that here would make it recover. Fine to file as a follow-up if it's out of scope.

Note the moves: name the *specific* consequence (not "this could break"), point at the *existing pattern* to follow, offer help, and mark severity honestly (blocking vs non-blocking vs follow-up).

## Severity discipline

Tag every finding: **Blocking** (correctness/security/contract break — merge stops), **Important** (should fix, but could be a fast follow-up), **Optional/nit** (preference — say "nit:" and mean it). A review that marks everything blocking is as useless as one that blocks nothing; calibrated severity is what makes your approval *mean* something. The [review katas](../04-code-reading-gym/04-review-katas.md) drill exactly this calibration.

## When to approve

Approve when layers 1–4 pass and layer 5 has only nits. You are not the author's compiler or their taste; you're the guardian of correctness, fit, and future-maintainability. "I'd have done it differently" is not a blocking reason; "this breaks under concurrency" is.

## Drill

Run this checklist against your own most recent change (or a fabledocs ticket you implemented). Count how many items you'd have caught *before* opening the PR. The gap is your pre-PR self-review checklist — write it down; running it on yourself is the fastest way to earn reviewer trust.

## Interview angle

- *"Tell me about a code review you gave / received."* → a STAR story from [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md); use the observation→consequence→suggestion structure as evidence of *how* you review.
- *"How do you decide what blocks a merge?"* → the severity discipline above.
