# 08 — Interview Prep

A first-class module, not an appendix. Target: **mid-level fullstack interviews** (C#/.NET + React/TypeScript). At least 60% of the question cards are anchored to real code in *this* repo, so you practice answering with concrete evidence instead of memorized trivia — which is exactly what distinguishes a mid-level answer.

| File | Round | Cards |
|---|---|---|
| [01-csharp-dotnet-deep-dive.md](01-csharp-dotnet-deep-dive.md) | Language/runtime | 15 |
| [02-frontend-framework-questions.md](02-frontend-framework-questions.md) | Frontend (React, with Angular contrasts) | 12 |
| [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) | API/data | 13 |
| [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) | System design | 1 full walkthrough + 5 variations |
| [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) | Practical | 4 debugging + 4 review sims |
| [06-behavioral-star-stories.md](06-behavioral-star-stories.md) | Behavioral | 10 STAR worksheets |
| [07-two-week-cram-plan.md](07-two-week-cram-plan.md) | Plan | day-by-day |

## How mid-level fullstack loops are structured

Typically: (1) recruiter/technical **screen** — fundamentals, a coding warm-up; (2) **technical deep-dive** — your stack, often a walk-through of something you built; (3) **practical coding** — a small feature or bug, sometimes take-home; (4) **system design** — "design X" at moderate depth; (5) **behavioral** — collaboration, conflict, judgment. This module has a file per round.

## Use this repo as your portfolio of talking points

You may not have shipped a big system — but you have *deeply studied* one, and can discuss its real tradeoffs with file-level evidence. That reads as more senior than vague war stories. When an interviewer asks a generic question, answer with "in a codebase I studied closely, they did X — here's the tradeoff and where it bites," and you've converted study into experience.

## The golden rule

**Answer with a concrete example, a tradeoff, and a failure mode — never a definition.** "async frees the thread during I/O" is a textbook answer. "async frees the thread — in a codebase I studied, every handler passes a cancellation token to EF so a client disconnect aborts the query; but they also had a sync-over-async `.GetAwaiter().GetResult()` in a save interceptor that would deadlock under a sync context" is a hire answer. Every card below models this junior → mid → senior gradient.

Start at [07-two-week-cram-plan.md](07-two-week-cram-plan.md) if your interview is soon; otherwise work the card files in order.
