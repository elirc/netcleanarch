# Part 1 — Guided Labs

Five labs, in order. Each builds on the reflexes from the last. Do them on a branch; break things freely.

| Lab | You'll build | New skill | Time |
|---|---|---|---|
| [Lab 1](lab-01-first-change.md) | Add a new colour to the palette, end to end | The full change→build→test loop; where a "simple" change actually touches | 30–45 min |
| [Lab 2](lab-02-add-a-field.md) | Add a `DueDate` field from the database to the API | A vertical slice through every layer; the blast-radius habit | 1.5–2 hr |
| [Lab 3](lab-03-validation-and-errors.md) | Add a validation rule and watch it become an HTTP 400 | How input validation and the error contract actually connect | 45–60 min |
| [Lab 4](lab-04-your-first-tests.md) | Write a unit test and a functional test that pass | Testing with this repo's real helpers; test-first thinking | 1 hr |
| [Lab 5](lab-05-fix-a-real-bug.md) | Find and fix a genuine bug (a wrong HTTP status) | Systematic debugging: reproduce → narrow → fix → cover | 1–1.5 hr |

## Ground rules for every lab

- **Branch first.** `git switch -c lab/01-add-colour` (see [workflow](../05-workflow-and-habits.md)). Never work on `main`.
- **Snippets marked `// Illustrative` are examples to adapt, not paste blindly.** Real anchors point at real files with line numbers — open those and read them.
- **Verify every step.** Each lab tells you exactly how to know it worked. Don't move on from a step you haven't confirmed.
- **When it breaks, that's the lab working.** Read the error, form a guess, test the guess. [04-tooling-and-debugging](../04-tooling-and-debugging.md) is your friend.
- **If a concept is fuzzy, pause and read its [primer](../02-fundamentals-primers/README.md).** You'll go faster overall.

## After the labs — capstone ideas

When you've done all five, prove it by shipping one of these on your own (they're sized for a junior with the labs behind them):

1. Add a `Priority` filter to the todo list read path (query + endpoint + test).
2. Bound the `Note` field length in a validator and the database config, with a test.
3. Add a "mark all items in a list as done" command, handler, endpoint, and functional test.
4. Add rollback + an error message to the React checkbox when a save fails ([the swallowed-error site](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167)).
5. Write the missing test proving a completed item raises its domain event.

Each of these maps to a real ticket in [fabledocs/06-contribution-practice](../../fabledocs/06-contribution-practice/01-good-first-tickets.md) if you want the full spec.
