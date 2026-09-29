# Refactor & Design Katas

Senior katas: no ticket hand-holding, just a target and self-grading criteria. Each takes 1–4 hours of *thinking* (some also coding). The goal is judgment, not a passing test.

---

## Kata 1: Close the Colour invariant leak
Target: make `new Colour("junk")` impossible while keeping the static factories and equality working.
The tension: private constructor breaks the static properties (`Colour.Red => new("#E05C4D")`, [L17-L29](../../src/Domain/ValueObjects/Colour.cs#L17-L29)) and the tests that use `new Colour(...)` ([ColourTests L51-L58](../../tests/Domain.UnitTests/ValueObjects/ColourTests.cs#L51-L58)). Resolve it: private ctor is fine for internal statics; update the tests to use `From`; decide what to do with the empty→`#000000` coercion ([L31](../../src/Domain/ValueObjects/Colour.cs#L31)) that manufactures an *unsupported* colour.
Self-grade — Basic: private ctor compiles. Solid: statics still work, tests updated to the factory. Strong: you also convert `Colour` to a `record` and get equality for free, *and* you notice the coercion should throw (or be removed) rather than silently create an invalid value.

## Kata 2: Design the ownership query filter (M1's hard core)
Target: on paper, make every read automatically scoped to the current user via EF `HasQueryFilter`, without threading `userId` through every handler.
The hard part: `OnModelCreating` runs once at model build, but the current user changes per request. Research and choose: capture a *provider* (`Func<string?>`/scoped `IUserContext`) in the filter expression; understand model caching implications; handle the seeder/tests where there is no user.
Self-grade — Basic: you know `HasQueryFilter` exists. Solid: you solve the per-request-user-in-a-once-built-model problem. Strong: you address filter *composition* (what happens when soft-delete adds a second filter), the write-side gap (query filters don't guard writes/`FindAsync` by key — you still need an explicit check), and how tests opt out.

## Kata 3: Extract a reducer from Todo.jsx
Target: the 423-line [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx) holds ~13 pieces of `useState`/refs and hand-patches its cache in six places. Design (don't necessarily build) a `useReducer` or state-machine refactor.
Self-grade — Basic: you list the state and group it (lists cache / dialog state / edit state). Solid: you define actions (`ITEM_TOGGLED`, `LIST_CREATED`, …) and centralize the cache patches. Strong: you argue *whether to bother* — is the complexity from missing server-cache management (TanStack Query would delete half this code) rather than missing local structure? A senior sometimes concludes "replace the pattern, don't reorganize it," and can say why.

## Kata 4: Split the God-component vs adopt a server-cache library
Target: a written decision (ADR-style) — for a growing version of this app, do you (a) split `Todo.jsx` into `ListsPanel`/`ItemsPanel`/dialogs, (b) adopt TanStack Query, or (c) both?
Self-grade — Strong answers weigh: (a) fixes render scope and readability but keeps manual cache surgery; (b) fixes cache drift/refetch/optimism-rollback but is a dependency and a paradigm; the interaction between them. Decide, justify, name what you'd measure to know you were right.

## Kata 5: Design the outbox schema and dispatcher (S1's core, standalone)
Target: table columns, dispatcher loop, failure handling, idempotency mechanism — one page, no code.
Self-grade — Basic: table + a polling loop. Solid: same-transaction capture, backoff, dead-letter, at-least-once + idempotent consumers. Strong: you address serialization of `BaseEvent` polymorphism, ordering, and the poll-latency vs load tradeoff, and you can state the failure matrix the outbox *fixes* vs the ones it doesn't (a bug in a consumer still needs its own retry semantics).

## Kata 6: Remove the per-request username lookup
Target: [LoggingBehaviour](../../src/Application/Common/Behaviours/LoggingBehaviour.cs#L27-L30) and [PerformanceBehaviour](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L43-L46) each do a DB query to turn a user id into a username for logs. Design the fix.
Self-grade — Basic: "log the id instead." Solid: log the id everywhere, resolve names at query/dashboard time; note it removes a query from every authenticated request. Strong: you consider whether the username is ever *needed* live (probably not — it's derivable), and you measure the before/after query count on a trace to prove the win rather than asserting it.

## Kata 7: Write the migration RFC (schema evolution)
Target: an RFC proposing the move from `EnsureCreated` to migrations (M9), aimed at a maintainer who *deliberately* avoided migrations for simplicity.
Self-grade — Strong answers acknowledge the counter-argument (template simplicity is a feature), propose it as opt-in or fork-guidance rather than a default, and include the expand–contract playbook, the CI pending-changes check, and multi-provider handling. Persuading someone who had a *reason* for the status quo is the senior communication skill ([07-career/03](../07-career-and-collaboration/03-maintainer-communication.md)).

## Kata 8: Reduce a hypothetical N+1
Target: this repo has none server-side, so *introduce* the N+1 from [fake-code contrast 4](../04-code-reading-gym/03-fake-code-contrasts.md) in a scratch branch, prove it with EF SQL logging (count the queries), then fix it with projection and re-measure.
Self-grade — Basic: you see more queries in the log. Solid: you fix it with `ProjectTo`/`Select` and the count drops to one. Strong: you can explain *why* the projection collapses to one query (expression tree translated as a JOIN) and when it *wouldn't* (un-translatable mapping → client eval → silent N+1 returns).

## Kata 9: Review the flawed PR (integration exercise)
Target: take [review kata 4](../04-code-reading-gym/04-review-katas.md) ("share the test user") and write the *complete* review — the blocking findings, the kind framing, and the follow-up reply when the author says "but it's so much faster."
Self-grade — Strong answers hold the line on the isolation invariant while offering the author a real alternative (seed the user inside `ResetState`), and stay warm under pushback. Correctness + kindness, simultaneously, is the whole game ([07-career/01](../07-career-and-collaboration/01-code-review-mindset.md)).

---

## Meta

These katas are deliberately open. The self-grade "Strong" bar is consistent across all nine: you can (1) make the change, (2) name what it costs, (3) name what it *doesn't* fix, and (4) decide whether it's worth doing *at all* in this specific repo. That fourth item — the judgment to say "correct fix, wrong project" — is what separates a senior from a very good mid-level, and it's exactly what a mid-level interview loop is probing for at its ceiling.
