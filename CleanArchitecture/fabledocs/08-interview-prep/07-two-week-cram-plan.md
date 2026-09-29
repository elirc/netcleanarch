# Two-Week Cram Plan

For a mid-level fullstack interview ~14 days out. Assumes ~2 focused hours/day plus one longer weekend session. Adjust to your gaps — if you're strong on React and weak on .NET, shift weight. Self-assessment checkpoints at Day 7 and Day 14.

Prereq: do the [fast track](../00-fast-track.md) first if you haven't — everything below assumes you can run and trace the app.

## Week 1 — Build the foundation and the evidence base

**Day 1 — Orientation.** Read [01-codebase-cartography/01-system-map](../01-codebase-cartography/01-system-map.md) + [05-key-flows](../01-codebase-cartography/05-key-flows.md) (flows 1–3). Do the system-map drill from memory. Goal: you can draw the architecture and trace a request.

**Day 2 — .NET fundamentals.** [02-stack/01-language-runtime-model](../02-stack-and-language-mastery/01-language-runtime-model.md). Answer [interview cards 01/Q1–Q8](01-csharp-dotnet-deep-dive.md) out loud. Goal: async, DI lifetimes, IQueryable, records land at mid level.

**Day 3 — Frameworks + the rest of .NET cards.** [02-stack/02-framework-mental-models](../02-stack-and-language-mastery/02-framework-mental-models.md). Finish [01/Q9–Q15](01-csharp-dotnet-deep-dive.md). Goal: pipeline, MediatR, EF change tracking articulate.

**Day 4 — Frontend.** [02-stack/02 (React section)](../02-stack-and-language-mastery/02-framework-mental-models.md) + all [02-frontend cards](02-frontend-framework-questions.md). Open [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx) and answer while pointing at code. Goal: render model, effects, stale closures, optimistic UI.

**Day 5 — API & data.** [03-architecture/02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md) + [03-validation-auth](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) + all [03-api cards](03-api-and-data-modeling-questions.md). Goal: validation layers, auth vs authz, migrations story, idempotency.

**Day 6 — Patterns + first drills.** [03-architecture/05-pattern-catalog](../03-architecture-and-patterns/05-pattern-catalog.md) (read all 14; deep-focus 5). Do 3 [annotation drills](../04-code-reading-gym/01-annotation-drills.md). Goal: recognize patterns and their failure modes.

**Day 7 — Checkpoint + review round.** Two timed [debugging sims](05-debugging-and-code-review-rounds.md) (Sims 2 and 3). Then self-assess (below). Goal: find your weakest area for week 2 weighting.

### Day 7 self-assessment
- [ ] Draw the architecture + trace a write path with real class names, unprompted.
- [ ] Answer 5 random .NET cards at mid level (tradeoff + failure mode, not definition).
- [ ] Explain optimistic UI and its missing rollback in this repo.
- [ ] Name three concrete gaps in the codebase and their fixes.
- [ ] Run one debugging sim narrating reproduce→narrow→fix.
Score each 0–2. Anything ≤1 gets extra time in week 2.

## Week 2 — Depth, system design, and rehearsal

**Day 8 — System design core.** [08-interview/04-system-design](04-system-design-from-this-repo.md) — the full 6-step walkthrough out loud, 20 min, recorded. Goal: requirements-first, tradeoff-per-step.

**Day 9 — System design variations.** The 5 variations (multi-tenancy, real-time, 10x, email/outbox, mobile). Do the [M1 design note](../06-contribution-practice/02-mid-level-feature-tickets.md) — it *is* the multi-tenancy answer. Goal: adapt a design under new constraints gracefully.

**Day 10 — Do a real ticket.** Implement [J2 (colour→400)](../06-contribution-practice/01-good-first-tickets.md) or [J3 (event test)](../06-contribution-practice/01-good-first-tickets.md) end to end, with tests and a [PR description](../07-career-and-collaboration/02-writing-prs-and-rfcs.md). Goal: a concrete, honest STAR story from real work.

**Day 11 — Behavioral.** [08-interview/06-STAR-stories](06-behavioral-star-stories.md). Fill and rehearse your 4 strongest (must include: learning a codebase, a bug you found, a tradeoff, a disagreement). Under 2 min each, ending in impact. Goal: no rambling, every story concrete.

**Day 12 — Review + reading gym.** Two [review sims](05-debugging-and-code-review-rounds.md) (Sims 1 and 4) + 2 [review katas](../04-code-reading-gym/04-review-katas.md). Practice observation→consequence→suggestion phrasing aloud. Goal: calibrated severity, kind + specific comments.

**Day 13 — Weak-area blitz + trace tables.** Return to your Day-7 ≤1 items. Do 2 [trace tables](../04-code-reading-gym/02-trace-tables.md) cold. Skim the [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) — it's a system-design cheat sheet.

**Day 14 — Full rehearsal + checkpoint.** Simulate a loop: 30 min mixed cards (draw 8 at random across files), 20 min system design (a fresh variation), 15 min debugging sim, 15 min behavioral with follow-ups. Then final self-assessment.

### Day 14 self-assessment (interview-ready bar)
- [ ] Any card from files 01–03, answered at mid+ with a repo anchor and a failure mode.
- [ ] Full system-design walkthrough + one live variation without losing the thread.
- [ ] A debugging sim narrated cleanly, ending with the regression test.
- [ ] A review sim: two blocking findings, phrased kindly and specifically.
- [ ] Four STAR stories under 2 minutes, each ending in impact, surviving one follow-up.
- [ ] One real ticket done, so at least one story is unquestionably yours.
Six green = interview-ready. Reds = your morning-of review list.

## The night before

Don't cram new material. Re-skim: the [system map](../01-codebase-cartography/01-system-map.md), your 4 STAR stories, and the [golden rule](README.md) — *example + tradeoff + failure mode, never a definition*. Sleep.

## If you have less than two weeks

Compress to the priority spine: Day 1 (orient) → Day 5 (API/data) → Day 8–9 (system design) → Day 11 (behavioral) → Day 14 (rehearsal). Skip the reading-gym depth; keep the system-design walkthrough and four STAR stories — those carry the most weight in a mid-level loop.
