# Debugging & Code-Review Rounds — Timed Simulations

These rounds test how you *think out loud*, not just whether you reach the answer. Four debugging sims (from [05-quality-engineering/03](../05-quality-engineering/03-systematic-debugging.md)) and four review sims (from [04-code-reading-gym/04](../04-code-reading-gym/04-review-katas.md)), reframed with a clock, interviewer follow-ups, and rubrics.

## How these rounds are scored

Interviewers grade: **method** (do you reproduce → narrow → hypothesize → fix → cover, or thrash?), **narration** (can they follow your reasoning?), **prioritization** (do you check the likely/cheap things first?), and **composure** (do you stay systematic under a follow-up curveball). The fix matters less than the path.

---

## Debugging Sim 1 (10 min): "Added items sometimes vanish on refresh."
Setup: [debug scenario 1](../05-quality-engineering/03-systematic-debugging.md). Narrate: "First I reproduce and check whether the item is missing from the DB or just the response" — then bisect.
Interviewer follow-ups: "The POST returned 201 — now what?" (→ check the persisted `ListId`.) "It's saved with `ListId: 0` — why?" (→ `CreateTodoItemCommand.ListId` has no validator rule; `0` passes validation, [validator](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItemCommandValidator.cs).) "How do you prevent it?" (→ add a rule / FK existence check + regression test.)
Rubric — Basic: guesses at the UI. Solid: bisects DB-vs-response, finds the validation gap. Strong: adds the regression test at the right layer and notes the client sent a stale `selectedListId`.

## Debugging Sim 2 (8 min): "Custom colour returns 500."
Setup: [scenario 2](../05-quality-engineering/03-systematic-debugging.md). The cleanest to narrate — good to rehearse first.
Follow-ups: "How do you know it's not a validation 400?" (→ body shape; validation passed.) "Where's it thrown?" (→ [Colour.From](../../src/Domain/ValueObjects/Colour.cs#L5-L15) via the handler.) "Why 500 not 400?" (→ unmapped in [ProblemDetailsExceptionHandler](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L43).) "Fix?" (→ validator rule preferred; keeps the 400 error shape consistent.)
Rubric — Strong: chooses validator over exception-mapping *and explains the error-contract-consistency reason*.

## Debugging Sim 3 (12 min): "Duplicate list titles in production."
Setup: [scenario 3](../05-quality-engineering/03-systematic-debugging.md). The hard one — it's a race.
Follow-ups: "Single-request test passes — so?" (→ read the validator; check-then-act.) "What's it called?" (→ TOCTOU; define it.) "Fix?" (→ unique index + 409 handling + keep validator + dedupe existing data.) "Why keep the validator?" (→ friendly errors for the common case.)
Rubric — Strong: names TOCTOU unprompted, and remembers the *existing-data dedupe* before the index can be added.

## Debugging Sim 4 (10 min): "Whole test suite fails in setup on a teammate's machine."
Setup: [scenario 5](../05-quality-engineering/03-systematic-debugging.md).
Follow-ups: "All tests failed — panic?" (→ no; identical errors ⇒ shared fixture, read [OneTimeSetUp](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L14-L46).) "It's the 60s timeout — cause?" (→ cold container pull / Docker down / DB file lock.) "Durable fix?" (→ improve the setup's failure message so the next person narrows in one read.)
Rubric — Strong: recognizes "all failed = fixture failed," and improves *diagnosability* rather than just bumping the timeout.

---

## Review Sim 1 (10 min): "Add DueDate to items."
Setup: [kata 1](../04-code-reading-gym/04-review-katas.md). You have the diff; find the blocking issues aloud.
Follow-ups: "Anything about the default value?" (→ non-nullable `DateTime` → `0001-01-01`.) "Can users see it?" (→ no — DTO not updated; write-only feature.)
Rubric — Strong: catches both the null-default and the missing read path, frames comments as observation→consequence→suggestion ([review mindset](../07-career-and-collaboration/01-code-review-mindset.md)).

## Review Sim 2 (10 min): "Cache GetTodos."
Setup: [kata 2](../04-code-reading-gym/04-review-katas.md).
Follow-ups: "It's faster though — approve?" (→ no: global key crosses users, no invalidation.) "Was it even slow?" (→ ask for the measurement.)
Rubric — Strong: correctness-before-performance, asks for evidence, and notes the double-stale interaction with the client cache.

## Review Sim 3 (8 min): "Make logout a GET."
Setup: [kata 6](../04-code-reading-gym/04-review-katas.md). Tests whether you spot security in an innocuous-looking change.
Follow-ups: "It's simpler — what's wrong?" (→ CSRF/prefetch via GET; the empty-body param is load-bearing.)
Rubric — Strong: recognizes ugly-but-load-bearing, blocks kindly, explains *why* the weird parameter exists rather than just "don't."

## Review Sim 4 (12 min): "Reorder pipeline behaviours for speed."
Setup: [kata 7](../04-code-reading-gym/04-review-katas.md). The subtle one — order is a security property.
Follow-ups: "Validation is cheaper than auth — why not first?" (→ validation-before-auth leaks existence info to anonymous callers and burns DB queries on unauthenticated traffic.)
Rubric — Strong: identifies order as security-relevant, and corrects the false premise (auth is usually a null-check, not a DB hit).

---

## Running these well

- **Say your plan before executing:** "I'll reproduce, then bisect DB vs response" — the interviewer is grading the plan.
- **Check cheap/likely first:** logs and the Network tab before the debugger; the validator before the database.
- **State hypotheses as falsifiable:** "if it's a validation gap, sending `ListId: 0` reproduces it" — then test *that*.
- **When stuck, narrate the narrowing, not silence:** "I've ruled out the response shape, so it's server-side; next I'd check…"
- **End with coverage:** always propose the regression test at the lowest observing layer.

## Drill

Have someone read you Sim 3 and Sim 4 (the hardest) with a timer and the follow-ups, while you talk. Score against the rubric. Then swap — running the interviewer side sharpens your sense of what a strong answer sounds like faster than anything else.
