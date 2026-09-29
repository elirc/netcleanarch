# Writing PRs and RFCs

## PR description template (use for every fabledocs ticket)

```markdown
## What
One or two sentences: the change, in the reader's terms.

## Why
The problem this solves. Link the issue/ticket. If it fixes a gap, name it.

## How
The approach, and the key decision(s). If you rejected an alternative, say which and why —
this is where you show judgment.

## Testing
What you added/ran. "Functional test asserting X" beats "tested locally".
Commands: `dotnet test ./tests/Application.FunctionalTests --filter "Name~..."`

## Risk & blast radius
What could this break? Who's affected? Is it additive/reversible?
Call out contract changes (endpoint renames, DTO shape) explicitly — they regenerate clients.

## Follow-ups
What you deliberately left out of scope (with a link if you filed it).
```

Why each section earns its place *here*: the **Risk & blast radius** section matters more than usual because this repo generates clients from the API — a change that looks internal can break both SPAs ([contract chain](../02-stack-and-language-mastery/03-type-system-and-contracts.md)). The **How** section's "rejected alternative" line is where you convert a code change into evidence of seniority — and into a future STAR story.

Worked example (Ticket 2, colour → 400):

```markdown
## What
Return 400 (not 500) when a todo list is created with an unsupported colour code.

## Why
`Colour.From` throws `UnsupportedColourException` for codes outside the supported set;
it wasn't mapped in `ProblemDetailsExceptionHandler`, so the API returned 500 for what is
a client input error. (Trace: fabledocs 04/02, Trace 4.)

## How
Added a `Colour` validation rule to `CreateTodoListCommandValidator` rather than mapping the
exception. Reason: it joins the aggregated 400 error shape the client already parses, keeping
one error contract, and catches the bad value before it reaches the domain. Considered mapping
the exception in the handler — rejected because it would only cover this one throw site and
produce a different (single-error) body shape.

## Testing
Added `CreateTodoListTests.ShouldRejectUnsupportedColour` asserting `ValidationException`
with an error under the `Colour` key. `dotnet test ./tests/Application.FunctionalTests`.

## Risk & blast radius
Additive validation rule; no contract-breaking change (still 400, richer body). No client
regen needed. Reversible by removing the rule.

## Follow-ups
The UI can only pick valid swatches, so this only affects direct API callers — expected.
```

## Commit messages

Follow the repo's own convention (its history is [Conventional Commits](../../README.md): `chore(deps): ...`, so use `feat:`, `fix:`, `test:`, `docs:`). Subject ≤ ~72 chars, imperative mood ("add", not "added"). Body explains *why*, not *what* (the diff shows what). One logical change per commit — the [TS-migration kata](../06-contribution-practice/04-refactor-and-design-katas.md) lesson: never mix mechanical and behavioral changes in one commit.

## When to write an RFC instead of just a PR

Write an RFC (a design doc, reviewed before code) when the change: touches many layers or files; changes a public contract; has a large or irreversible blast radius; or has multiple viable approaches worth debating. In fabledocs terms: junior tickets → straight PR; **M1, M9, all S-projects → RFC first** ([mid-level](../06-contribution-practice/02-mid-level-feature-tickets.md), [senior](../06-contribution-practice/03-senior-build-projects.md)). The repo already models this: its [ADRs](../../docs/decisions/README.md) are lightweight RFCs for decisions already made.

## RFC template (tailored to this repo)

```markdown
# RFC: <title>

## Status
Draft | Under review | Accepted | Rejected

## Context
The problem and the constraints. What's true today (link the code). Why now.

## Decision
The proposed approach, concretely enough to implement.

## Rationale
Why this over the alternatives. List the alternatives and their rejection reasons.
(Model: ADR-001 rejecting the repository pattern with specific arguments.)

## Consequences
### Easier
### Harder
(Copy the ADRs' honesty — always fill "Harder".)

## Migration & rollback
Layer-by-layer touch list. How to ship incrementally. How to revert.

## Test strategy
Which layer(s), what proves success.

## Open questions
```

Study the three real ADRs before writing one: [ADR-001](../../docs/decisions/ADR-001-Use-EFCore-In-Application-Layer.md) (the gold standard — rejects an alternative with force, includes "when this decision does not apply"), [ADR-002](../../docs/decisions/ADR-002-Aspire-For-Orchestration-And-Testing.md) (lists rejected alternatives explicitly), [ADR-003](../../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md) (honest about a small compromise). Their shared move: every one has a "Harder:" section. A design doc with no downsides listed is a sales pitch, and reviewers know it.

## Drill

Write the RFC for **S1 (outbox)** using this template. Then the harder test: put it away for a day, reread it as a skeptical maintainer, and add the three objections you'd raise against your own proposal (and answers). If you can't find three, you haven't stress-tested it. This RFC doubles as your [system-design interview](../08-interview-prep/04-system-design-from-this-repo.md) script for "design reliable event delivery."

## Interview angle

- *"Walk me through a design doc / technical proposal you wrote."* → bring the S1 RFC; the "Harder" and "rejected alternatives" sections are what signal seniority. [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md)
- *"How do you communicate a risky change?"* → the Risk & blast radius section + incremental rollout.
