# Maintainer Communication

Working with maintainers (of this repo or your team's) is a skill with its own techniques: ask well, disagree respectfully, and never outsource your thinking.

## Asking for help without outsourcing thinking

The difference between "please help" and a *good* question is that a good question does the maintainer's narrowing for them. Structure:

1. **Goal** — what you're trying to achieve (not just the error).
2. **What you tried** — the hypotheses you tested and ruled out ([the debugging method](../05-quality-engineering/03-systematic-debugging.md)).
3. **Minimal repro** — the smallest thing that shows the problem.
4. **Specific question** — a decision or fact, not "what do I do?"

Bad: "The functional tests don't work, help?"
Good: "Functional tests fail in `OneTimeSetUp` with a 60s timeout on my machine (SQL Server variant). I confirmed Docker is running and the image pulled; the timeout hits during `WaitForResourceHealthyAsync` ([FunctionalTestSetup L38](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L38)). Is the 60s budget expected to cover a cold container start, or should first-run pull happen out of band?"

The good version proves you read the code, narrowed the failure, and asks one answerable question. It respects the maintainer's time — which is the whole currency of open source.

## Reporting a bug

Template:
```markdown
**Environment:** template variant (client-framework + database), .NET SDK version, OS.
**Expected:** ...
**Actual:** ...
**Repro:** numbered steps, ideally against a fresh `dotnet new ca-sln ...`
**Evidence:** the failing request/response, log line, or test output (not a screenshot of text).
**Scope note:** "happens on sqlite+react; haven't checked other variants."
```
The variant note matters *specifically here* because this is a template with nine combinations ([test.ps1](../../build/test.ps1)) — "it's broken" without the variant is unactionable.

## Proposing a feature

Lead with the problem and the user, not your solution. Acknowledge the project's philosophy — this template *values simplicity*, and many good ideas ([migrations M9](../06-contribution-practice/02-mid-level-feature-tickets.md), an [outbox S1](../06-contribution-practice/03-senior-build-projects.md)) are deliberately out of scope. A proposal that says "I know the template stays minimal; here's why this earns its weight, and here's how to keep it opt-in" will get read. One that says "you should add X" (implying they forgot) won't.

Template:
```markdown
**Problem:** who hits it, how often, why it hurts.
**Proposal:** the change, concretely.
**Fit with project scope:** why this belongs (or how to keep it optional/behind a flag).
**Cost:** what it adds to the template's weight/complexity — be honest.
**Alternatives:** what else you considered.
```

## Respectful disagreement (in review, both directions)

When you disagree with a maintainer's feedback: assume they have context you don't (they usually do), state your reasoning, and make the call *theirs*. "I chose X because Y; if there's a convention I'm missing, happy to switch — what would you prefer?" invites resolution. Digging in ("but X is objectively better") burns goodwill over a todo app.

When a maintainer disagrees with *your* review as an author's reviewer: hold the line only on correctness/security (layers 1–2 of [review](01-code-review-mindset.md)); yield on taste. And when you're right about something blocking, be right *kindly* — the [share-the-test-user kata](../06-contribution-practice/04-refactor-and-design-katas.md) is practice for exactly this: "I hear that the setup is slow — that's real. The blocker is that removing the reset makes tests order-dependent (the second create will hit the uniqueness rule). Could we seed the user inside `ResetState` instead? Same speed win, isolation intact."

## Responding to review on your own PR

- Reply to every comment (even "done" or "good catch").
- If you disagree, say why once, then defer to the reviewer if it's not correctness.
- Don't force-push over the discussion — let reviewers see the follow-up commits, squash at the end.
- Thank people; reviewing takes time.

## The meta-principle

Every interaction either builds or spends trust. Maintainers give merge rights and attention to people who make their lives *easier* — well-scoped PRs, well-narrowed questions, disagreements that stay warm. That reputation compounds, in open source and on teams, and it's the substrate promotions are built on.

## Drill

Rewrite this real-feeling bad message into a good one: *"hey the colour thing returns 500 that's a bug can you fix it."* Include environment, expected/actual, repro, evidence, and one specific question. Compare with the bug template. Then write the feature proposal for [M9 (migrations)](../06-contribution-practice/02-mid-level-feature-tickets.md) knowing the maintainer *chose* `EnsureCreated` on purpose — the persuasion challenge is the point.

## Interview angle

- *"Tell me about a time you disagreed with a teammate/senior."* → the respectful-disagreement structure, with a concrete example (real or from the katas). Interviewers score whether you separated correctness from taste and kept it warm. [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md)
- *"How do you ask for help?"* → the narrow-first structure signals you don't outsource thinking — a real hiring signal.
