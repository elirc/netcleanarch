# Workflow & Habits

The technical skills get you started; the habits get you trusted. This is the stuff nobody writes down and everybody is quietly judged on.

## Git, the parts you'll use daily

You don't need Git mastery. You need a small, reliable loop.

```bash
git switch -c feat/add-due-date     # start a branch off main for your change
# ... edit, build, test ...
git add -A
git commit -m "feat: add optional DueDate to todo items"
git push -u origin feat/add-due-date
# ... open a pull request ...
```

Rules that matter:

- **Never work on `main`.** Always branch. `main` is shared; your experiments aren't.
- **One logical change per branch/PR.** "Add DueDate" is one PR. "Add DueDate and also reformat three files and rename a method" is three PRs crammed into one, and a reviewer can't trust any of it. Keep the diff about one thing.
- **Commit messages: what and why, imperative mood.** This repo uses [Conventional Commits](../../README.md) — `feat:`, `fix:`, `test:`, `docs:`, `chore:`. The subject says *what* ("add DueDate validation"); the body, if any, says *why*. The diff already shows *how*.
- **Never mix a mechanical change with a behavioural one in the same commit.** Reformatting + a logic change in one commit means the reviewer can't see the logic change in the noise. Separate them.

## Writing a pull request people can review

A good PR description answers five questions. Copy this:

```markdown
## What
One sentence: the change.

## Why
The problem it solves. Link the issue/ticket.

## How
The approach, and any decision you made (e.g. "chose a validator over mapping the
exception, to keep the error contract consistent"). This is where you show judgment.

## Testing
What you added and ran. "Functional test asserting the Colour error key" beats "tested locally".

## Risk
What could break, who's affected, is it reversible. Call out any contract change
(renamed endpoint, changed DTO) — those regenerate the clients.
```

The **How** and **Risk** sections are where a junior PR starts looking like a senior one. Anyone can write code; explaining *why this approach* and *what it could break* is the part that builds trust. (For this repo specifically: renaming an endpoint method is a breaking API change because it changes the generated client — always flag it. See [fabledocs3 ADR-006](../../fabledocs3/adr-006-contract-first-client-generation.md).)

## Before you open the PR — self-review

Read your own diff, line by line, as if someone else wrote it. You'll catch half your mistakes here — a leftover `Console.WriteLine`, a commented-out block, a test you meant to un-skip. A quick checklist for *this* codebase:

- Did I follow the existing pattern, or invent a new one? (Match the neighbors.)
- New string input → bounded in a validator *and* the DB config?
- New exception type → mapped to an HTTP status, or it's a silent 500?
- New endpoint → does it need `RequireAuthorization()`?
- Does the build pass with **warnings-as-errors** ([Directory.Build.props L5](../../Directory.Build.props#L5))?
- Did I add a test at the [right layer](02-fundamentals-primers/README.md)?

Running this on yourself is the fastest way to earn a reviewer's trust — they learn your PRs are already clean.

## Breaking a task down

Handed "add due dates to todo items," a junior opens `TodoItem.cs` and starts typing. A stronger move: write the **touch list** first (you did this in [Lab 2](01-guided-labs/lab-02-add-a-field.md)) — every layer the change hits, in order — then work the list. Benefits: you spot the hard part early, you can estimate honestly, and you don't discover the DTO you forgot after you thought you were done. A task you can't break into steps is a task you don't understand yet; breaking it down *is* the understanding.

## Estimating (and why yours will be wrong at first)

You'll be asked "how long will this take?" Your early estimates will be too low, universally — because juniors estimate the *happy path* (write the code) and forget the rest (understanding the existing code, edge cases, tests, review, the thing that breaks). A rough correction: estimate the coding, then *add the same amount again* for everything around it. And estimate in ranges ("1–2 days"), not false precision ("6 hours"). Nobody is angry about a range; everybody remembers a blown "definitely by 3pm."

## How to ask for help — the highest-leverage skill

You *will* get stuck. Getting stuck is fine; how you get unstuck is what people notice. The difference between a question that annoys people and one that earns respect is whether you did the narrowing first.

**Bad:** "The tests don't work, can you help?"

**Good:** "The functional tests fail in setup with a 60-second timeout on my machine. I checked Docker's running and the SQLite file isn't locked; the failure is in `WaitForResourceHealthyAsync` ([FunctionalTestSetup L38](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L38)). Is the 60s budget expected to cover a cold first-run, or should something be pre-warmed?"

The good version proves you tried, shows exactly where you're stuck, and asks *one specific answerable question*. It respects the other person's time — which is the whole currency here. The structure:

1. **Goal** — what you're trying to do.
2. **What you tried** — the hypotheses you ruled out.
3. **The specific blocker** — the exact line/error, not "it's broken."
4. **One precise question.**

And a timing rule: try on your own for a bit (say 30–60 minutes of real narrowing), but **don't burn a whole day stuck out of pride.** Asking well after an honest attempt is a strength, not a weakness. Nobody respects the junior who silently thrashed for eight hours; everybody respects the one who narrowed it down and asked a sharp question.

## The habits that compound

- **Read before you write.** Understand the pattern you're extending; match it.
- **Verify every step.** Don't move on from something you haven't confirmed works.
- **Leave it a little better.** Fix the typo, the stale comment, the missing test you noticed — small, in-scope. (But no drive-by refactors in an unrelated PR.)
- **Write things down.** Your questions, your findings, the thing that confused you. Your notes today are the onboarding doc you write for the next junior — and teaching is how you find out what you actually understand.

None of these are about talent. They're about care, and care is a choice you make every day. That's what turns a junior into someone the team relies on.

## Where to go next

You've done the [labs](01-guided-labs/README.md), read the [primers](02-fundamentals-primers/README.md), and you have the [decoder](03-syntax-decoder.md) and [tooling](04-tooling-and-debugging.md) in your back pocket. Now the deeper material in [fabledocs](../fabledocs/README.md) will land, because you've *done* the things it analyzes — pick a [capstone](01-guided-labs/README.md), ship it, and start reading the pattern catalog and architecture critique with real context.
