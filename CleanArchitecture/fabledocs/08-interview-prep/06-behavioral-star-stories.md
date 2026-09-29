# Behavioral STAR Stories

Ten worksheets. STAR = **Situation, Task, Action, Result**. Behavioral rounds test collaboration and judgment; for a learner without years of shipped features, stories sourced from *deeply studying and contributing to this repo* are legitimate and often more concrete than vague team anecdotes. Fill each worksheet from your own work on the fabledocs tickets — the scaffolding tells you what makes each story land.

Rule: every story ends with **impact** (a number, a merged PR, a prevented bug), stays under 2 minutes, and shows *you* thinking, not just doing.

---

## Story 1: "Tell me about a complex codebase you had to learn."
Source: this entire curriculum. Situation: handed an unfamiliar Clean Architecture .NET solution with two SPA frontends. Task: understand it well enough to contribute safely. Action: found the composition root, traced one read and one write path end to end ([key flows](../01-codebase-cartography/05-key-flows.md)), built a system map, read the ADRs for the *why*. Result: could explain the architecture and its tradeoffs, and identified concrete gaps (no user isolation, no migrations).
Senior-signal detail: you learned the *tradeoffs* (ADRs), not just the structure. Resume bullet: "Reverse-engineered a multi-layer .NET/React codebase into a system map and gap analysis within days."
Prompts it answers: learning ambiguity, onboarding, "walk me through a system."

## Story 2: "A time you found a bug others missed."
Source: [debug scenario 3](../05-quality-engineering/03-systematic-debugging.md) (TOCTOU on unique titles) or the [colour 500](../05-quality-engineering/03-systematic-debugging.md). Situation: uniqueness enforced by a validator with no DB constraint. Task: determine if it was actually safe. Action: reasoned about concurrency, built a parallel-request repro, confirmed the race, proposed index+409+dedupe. Result: turned an advisory check into a real invariant with a regression test.
Senior-signal: you didn't just fix it — you *named the class* (TOCTOU) and handled existing data. Bullet: "Identified and fixed a check-then-act race in a uniqueness rule; added a DB constraint and regression coverage."

## Story 3: "A technical tradeoff you weighed."
Source: [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) — e.g. exception-mapping vs a validator for the colour fix, or whether to add an outbox. Action: listed both options, their costs, chose the one keeping the error contract consistent / kept the outbox optional to preserve template simplicity. Result: a decision you can defend from both sides.
Senior-signal: you can argue the side you *didn't* pick. Bullet: "Evaluated competing approaches against contract consistency and blast radius before choosing."

## Story 4: "A time you disagreed with a teammate or senior."
Source: [review kata 4](../04-code-reading-gym/04-review-katas.md) (someone removes test isolation for speed) or [maintainer communication](../07-career-and-collaboration/03-maintainer-communication.md). Action: acknowledged the real concern (slow setup), held the line on the isolation invariant, offered a concrete alternative (seed the user inside `ResetState`), stayed warm. Result: speed win *and* isolation preserved.
Senior-signal: separated correctness from taste; disagreed kindly. Bullet: "Blocked a change that would have made tests order-dependent while proposing an alternative that met the author's goal."

## Story 5: "A mistake you made and what you learned."
Source: any ticket where the first approach was wrong — e.g. mapping the colour exception in the handler, then realizing a validator keeps the error contract consistent. Action: caught it (in self-review or via the checklist), corrected course, documented why. Result: a better fix and a reusable principle (invariant violations belong at the validation layer for contract consistency).
Senior-signal: you have a *self-review process* that catches your own mistakes ([review mindset](../07-career-and-collaboration/01-code-review-mindset.md)). Bullet: "Caught a design flaw in self-review and adjusted to keep the API error contract uniform."

## Story 6: "A time you improved code quality or process."
Source: [ticket J3](../06-contribution-practice/01-good-first-tickets.md) (added missing event-dispatch test) or [J12](../06-contribution-practice/01-good-first-tickets.md) (enabled React lint in CI). Action: found a verified coverage/CI gap, closed it, respecting template-conditional constraints. Result: a safety net where there was none.
Senior-signal: proactive, and you respected existing constraints ([tooling](../02-stack-and-language-mastery/04-tooling-and-build-system.md)). Bullet: "Closed a test-coverage gap on the domain-event dispatch path; added CI linting for the frontend."

## Story 7: "A time you had to make a decision with incomplete information."
Source: any "investigate"-labeled item — e.g. the [role-string split](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) behavior. Action: labeled it a hypothesis, verified by reading/tracing before acting, avoided claiming a bug you hadn't confirmed. Result: an accurate risk assessment, not a false alarm.
Senior-signal: intellectual honesty — you distinguish confirmed from suspected ([verification log](../09-reference/verification-log.md)). Bullet: "Triaged an ambiguous authorization behavior, verified before escalating, avoiding a false bug report."

## Story 8: "A time you owned something end to end."
Source: a mid-level ticket done fully — [M4 (move-item hardening)](../06-contribution-practice/02-mid-level-feature-tickets.md) or [M2 (unique titles)](../06-contribution-practice/02-mid-level-feature-tickets.md). Action: design note → cross-layer implementation → tests → PR with risk/rollback sections. Result: a merged, tested, documented change.
Senior-signal: you did the *design note and rollback plan*, not just code ([PR/RFC](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)). Bullet: "Delivered a cross-layer feature with a design note, tests, and a rollback plan."

## Story 9: "A time you dealt with technical debt or a legacy footgun."
Source: [ticket J15](../06-contribution-practice/01-good-first-tickets.md) (documented the dev DB reset) or the [stale Web.http](../06-contribution-practice/01-good-first-tickets.md). Action: recognized an undocumented footgun (EnsureDeleted wipes dev data every run), documented it for future contributors. Result: fewer surprised newcomers.
Senior-signal: you fix *onboarding paper cuts*, which compounds. Bullet: "Documented a data-loss footgun in local dev setup that repeatedly surprised new contributors."

## Story 10: "A time you learned something new quickly to get unblocked."
Source: any unfamiliar tech here — Aspire orchestration, EF interceptors, MediatR pipeline, NSwag codegen. Action: read the relevant code + one ADR, built a mental model, applied it. Result: contributed in that area.
Senior-signal: a *method* for learning (code + ADR + trace), not just grit. Bullet: "Ramped on Aspire orchestration and EF interceptors by tracing real flows, then contributed within the week."

---

## Filling and rehearsing

For each story, write four tight sentences (S/T/A/R), one evidence citation (PR, test, anchor), and the resume bullet. Then the rehearsal check: **under 2 minutes? concrete (names, numbers, files)? ends with impact?** Record yourself once — behavioral answers fail on rambling far more than on content.

Map to common prompt families so you're never caught flat: conflict → Story 4; ambiguity → 1, 7; mistake → 5; tradeoff → 3; ownership/leadership → 8; learning → 1, 10; quality → 6, 9; bug/rigor → 2.

## Drill

Pick your three strongest, rehearse each under 2 minutes to a timer, and have someone ask a *follow-up* ("what would you do differently?" / "how did the other person react?"). The follow-up is where prepared stories fall apart and genuine ones hold — which is exactly why doing the underlying [ticket](../06-contribution-practice/README.md) for real beats inventing the story.
