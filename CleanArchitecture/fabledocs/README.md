# fabledocs — A Training Lab Built From This Codebase

This is a learning curriculum generated from the code in this repository. It is written for one specific learner: **a junior fullstack engineer** (comfortable with CRUD web dev in .NET/C# and React/TypeScript or similar) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page teaches two things at once:

1. **This codebase** — where things live, how its real flows work, with exact file-and-line anchors.
2. **The transferable skill** — why the pattern exists, what the alternatives are, how it fails, and how to talk about it in an interview.

## What this repository is

This is [jasontaylordev/CleanArchitecture](../README.md) — the source of the `Clean.Architecture.Solution.Template` NuGet template (`dotnet new ca-sln`). It is a **.NET 10 solution template** demonstrating Clean Architecture with:

- **Domain** layer: entities, value objects, domain events — almost no external dependencies ([ADR-003](../docs/decisions/ADR-003-MediatR-Contracts-In-Domain.md)).
- **Application** layer: CQRS use cases via MediatR, FluentValidation, AutoMapper, and a pipeline of cross-cutting behaviours (logging, exception handling, authorization, validation, performance).
- **Infrastructure** layer: EF Core 10 (`ApplicationDbContext`), ASP.NET Core Identity, save-changes interceptors for auditing and domain-event dispatch.
- **Web** layer: Minimal API endpoint groups, OpenAPI + Scalar, ProblemDetails error mapping, and two SPA clients (Angular 21 and React 19) whose API clients are **generated** from the OpenAPI spec by NSwag.
- **Aspire** orchestration: `AppHost` runs the full stack; `TestAppHost` runs just the database for functional tests ([ADR-002](../docs/decisions/ADR-002-Aspire-For-Orchestration-And-Testing.md)).
- Four test layers: domain unit tests, application unit tests, functional tests against a real database (Respawn resets), and Playwright/Reqnroll acceptance tests.

The sample domain is deliberately small — todo lists and todo items — so the architecture is the star. That makes it an ideal gym: every enterprise pattern is present, but no page of business logic gets in your way.

**One caveat you must internalize early:** this repo is a *template*. You will see `#if (UseApiOnly)` / `#if (UsePostgreSQL)` conditional blocks in C# and `<!--#if ... -->` in XML — these are template preprocessor directives, not normal code (e.g. [Program.cs](../src/Web/Program.cs#L40-L49), [DependencyInjection.cs](../src/Infrastructure/DependencyInjection.cs#L26-L32)). And it intentionally omits some production concerns (per-user data ownership, migrations, rate limiting) — those gaps are *teaching opportunities*, flagged throughout, not bugs.

## How to use this curriculum

| Time budget | Path |
|---|---|
| **One weekend** | [00-fast-track.md](00-fast-track.md) only. Run it, trace two flows, make one safe change, do one teach-back. |
| **Two weeks** | Fast track → [01-codebase-cartography](01-codebase-cartography/README.md) → [03-architecture-and-patterns/05-pattern-catalog.md](03-architecture-and-patterns/05-pattern-catalog.md) → 3 drills from [04-code-reading-gym](04-code-reading-gym/README.md) → 2 tickets from [06-contribution-practice/01-good-first-tickets.md](06-contribution-practice/01-good-first-tickets.md). |
| **Eight weeks** | All modules in order, one per ~4 days, doing every drill and at least 5 tickets + 1 mid-level ticket. |
| **Ongoing contribution** | Live in [06-contribution-practice](06-contribution-practice/README.md) and [07-career-and-collaboration](07-career-and-collaboration/README.md); return to modules as reference. |
| **Interview in two weeks** | Go directly to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md). It schedules everything else you need. |

### Recommended entry points by profile

- **Brand-new junior:** 00 → 01 (all) → 02 → 04 drills before touching 03.
- **Junior who knows .NET + React already:** 00 → 01/05-key-flows → 03 (all) → 04 → 06.
- **Mid-level, new to this repo:** 01/01-system-map + 01/05-key-flows → 03/06-architecture-critique → 06/02-mid-level-feature-tickets.
- **Senior doing architecture review:** 03/06-architecture-critique → 09-reference/risk-register.md → the three ADRs in [docs/decisions](../docs/decisions/README.md).
- **Interview candidate:** [08-interview-prep](08-interview-prep/README.md), starting with the cram plan.

## Conventions used in every file

- **File anchors** — claims about this repo link to exact files and lines, e.g. [TodoItem.cs](../src/Domain/Entities/TodoItem.cs#L14-L26). Line numbers were verified against the working tree at the time of writing (see [verification-log](09-reference/verification-log.md)); if the repo has moved on, trust the path and search for the symbol.
- **Fake code is labeled** — any snippet that is *not* from this repo starts with `// Illustrative fake code: not from this repo`. Everything else is real.
- **Drills** — most pages end with a task and a self-grading rubric (Basic / Solid / Strong). Do the drill before reading the rubric.
- **Verification notes** — commands are marked **verified** (actually run while writing these docs) or **inferred** (read from scripts/CI but not executed here).
- **Interview angle** — sections flagged with this heading tell you how the material surfaces in interviews and what a mid-level answer sounds like.
- **Senior vocabulary** — words like *invariant*, *boundary, contract, idempotency, isolation, blast radius* are used deliberately and defined in context the first time each module uses them. They are interview vocabulary; collect them.

## The mindset ladder

- A **junior** asks: *"How do I make it work?"*
- A **mid-level** asks: *"Is this the right pattern? What does it cost?"*
- A **senior** asks: *"What does this commit us to, who pays that cost, and how do we reduce the risk?"*

Interviews for mid-level roles test exactly the second and third questions. Nobody passes a mid-level loop by reciting definitions; you pass by pointing at a concrete system — this one — and explaining a decision, its tradeoff, and its failure mode. This curriculum exists to give you that concrete system.

## Module index

| Module | What you get |
|---|---|
| [00-fast-track.md](00-fast-track.md) | Weekend on-ramp: run, trace, change, test, teach back |
| [01-codebase-cartography](01-codebase-cartography/README.md) | System map, reading order, glossary, tooling map, 6 key flows |
| [02-stack-and-language-mastery](02-stack-and-language-mastery/README.md) | C#/.NET runtime model, ASP.NET Core + React mental models, contracts, build system |
| [03-architecture-and-patterns](03-architecture-and-patterns/README.md) | Layers, persistence, validation/auth, side effects, 14 pattern cards, critique |
| [04-code-reading-gym](04-code-reading-gym/README.md) | Annotation drills, trace tables, fake-code contrasts, review katas |
| [05-quality-engineering](05-quality-engineering/README.md) | Testing strategy, test recipes, debugging method, performance, security, observability |
| [06-contribution-practice](06-contribution-practice/README.md) | 15 junior tickets, 10 mid-level tickets, 5 senior projects, design katas |
| [07-career-and-collaboration](07-career-and-collaboration/README.md) | Review mindset, PRs/RFCs, maintainer communication |
| [08-interview-prep](08-interview-prep/README.md) | 45+ question cards anchored to this repo, system design walkthrough, STAR stories, cram plan |
| [09-reference](09-reference/command-cheatsheet.md) | Commands, risk register, rubrics, verification log |
