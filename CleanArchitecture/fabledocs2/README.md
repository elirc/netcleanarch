# Onboarding: The Honest Version

Welcome. I built most of this. I'm going to tell you the truth about it, which is not the same story you'll get from the marketing page or the conference talk.

## What this codebase actually is

It's a **.NET 10 Clean Architecture solution template** — the thing behind `dotnet new ca-sln`. The sample domain is deliberately tiny (todo lists and todo items) because the *architecture* is the product, not the features. That has a consequence you need to internalize on day one: **a lot of the "why" here is about teaching a pattern cleanly, not about running a business.** Some of the decisions I'll defend below are the right call for a template and the *wrong* call for the app you'll eventually deploy on top of it. I'll flag those every time.

The other consequence: you'll see conditional-compilation directives everywhere — `#if (UseApiOnly)`, `#if (UsePostgreSQL)` in C#, `<!--#if ... -->` in XML, `//#if` in the scaffolding templates. Those are `dotnet new` template switches, not normal code. This checkout compiles as the **SQLite + full-stack (Angular default)** variant because none of those symbols are defined. When you read a file, always ask "which variant am I looking at?" — see [src/Web/Program.cs L40-L49](../src/Web/Program.cs#L40-L49) and [src/Infrastructure/DependencyInjection.cs L26-L74](../src/Infrastructure/DependencyInjection.cs#L26-L74).

## How to read this document

Six sections, each about one subsystem. Every section follows the same shape, because I want you to build the same reflexes I use:

1. **What it does and why it exists** — the honest purpose, not the slogan.
2. **The 2–3 hardest decisions** — including the alternatives I rejected and the tradeoffs that *still* bother me.
3. **Evidence** — exact files and classes. Don't take my word for anything; open them.
4. **One incident** — a real failure mode this design caused or prevented. War stories stick better than principles.
5. **Three questions** — you should be able to answer all three before moving to the next section. If you can't, you skimmed.

| # | Section | The one-line truth |
|---|---|---|
| 1 | [Architecture & layering](01-architecture.md) | The dependency rule is real; the "Domain has zero dependencies" ideal is a comfortable lie. |
| 2 | [API design](02-api-design.md) | Method names are public API here, and almost nobody notices until it breaks. |
| 3 | [Data access](03-data-access.md) | We put EF Core in the Application layer on purpose, and I'd defend it, but the validators-as-invariants thing is a footgun. |
| 4 | [Testing strategy](04-testing-strategy.md) | We test against a real database because mocks lied to us. The suite can't run in parallel and I'm not thrilled about it. |
| 5 | [Error handling](05-error-handling.md) | One error contract, co-designed with its docs — except for the exceptions we forgot to map. |
| 6 | [Observability](06-observability.md) | We wired OpenTelemetry beautifully and then logged whole request payloads, which is a security incident waiting to happen. |

## The mindset I want you to leave with

Junior engineers ask "how do I make it work?" I need you asking the next two questions: "is this the right pattern, and what does it cost?" and eventually "what does this commit us to, who pays the cost, and how do we shrink the blast radius?" Every section below is me modeling those questions on my own code. Steal the habit, not just the facts.

Now go read [section 1](01-architecture.md).
