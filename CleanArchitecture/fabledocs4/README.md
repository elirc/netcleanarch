# fabledocs4 — The Hands-On Workbook

The other three doc sets in this repo are things you *read*. This one is things you *do*.

- **fabledocs** — the full curriculum (maps, patterns, drills, interview prep).
- **fabledocs2** — the candid principal-engineer onboarding tour.
- **fabledocs3** — reverse-engineered architecture decision records.
- **fabledocs4 (this one)** — guided labs you build with your own hands, plus first-principles primers on the fundamentals this codebase quietly assumes you already know.

If you're a junior engineer and the other folders felt like drinking from a firehose, **start here.** You learn a codebase by changing it, breaking it, watching a test go red and then green — not by reading about it. This workbook is built around that.

## What's inside

| Part | What it is | When to use it |
|---|---|---|
| [01-guided-labs/](01-guided-labs/README.md) | Five step-by-step labs. You edit real files, run real commands, verify real results. | Do these first, in order. This is the core. |
| [02-fundamentals-primers/](02-fundamentals-primers/README.md) | Five "from scratch" explainers: dependency injection, async/await, web auth, transactions, HTTP/REST. Each teaches the concept, then shows exactly where this repo uses it. | Read a primer when a lab or a file uses a concept you can't fully explain to someone else. |
| [03-syntax-decoder.md](03-syntax-decoder.md) | Every unfamiliar C# and TypeScript symbol in this codebase, decoded — `init`, `record`, `null!`, `=>`, `??`, `[]`, `static abstract`, and friends. | Keep it open while you read code. Look things up as you hit them. |
| [04-tooling-and-debugging.md](04-tooling-and-debugging.md) | How to actually run it, set a breakpoint, read a stack trace, see the SQL EF generates, and use the Aspire dashboard. | Before Lab 1, and whenever something breaks and you don't know why. |
| [05-workflow-and-habits.md](05-workflow-and-habits.md) | Branching, commits, PRs, breaking a task down, estimating, and — the most important junior skill — how to ask for help well. | Read once early; revisit before your first real PR. |

## The one rule

**Do the labs on a branch, and never be afraid to break things.** Nothing here can hurt the real project — you're working on your own copy, and in development this app *recreates its database from scratch on every run* ([evidence](../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46)), so there's no precious state to lose. Breaking something and fixing it is the entire point. The engineer who's afraid to break things stays junior; the one who breaks things on purpose, in a safe place, to see what happens, becomes senior.

## How to start

1. Read [04-tooling-and-debugging.md](04-tooling-and-debugging.md) §1 and get the app running.
2. Do [Lab 1](01-guided-labs/lab-01-first-change.md). It takes 30–45 minutes and ends with a passing test you wrote.
3. Keep [03-syntax-decoder.md](03-syntax-decoder.md) open in a second tab for anything that looks like line noise.
4. When a lab uses a word you couldn't define out loud — *dependency injection, transaction, cookie auth* — stop and read the matching [primer](02-fundamentals-primers/README.md).

Progress feels slow for the first week and then suddenly isn't. That's normal. Keep going.
