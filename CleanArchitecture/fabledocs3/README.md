# Reverse-Engineered Architecture Decision Records

This folder reconstructs the **ten most significant architecture decisions** the authors of this codebase made — most of them *implicitly*, i.e. embodied in code without a written ADR. Each is written in standard ADR format (Context, Decision, Alternatives Considered, Consequences, Status) and grounded in specific files and classes you can open.

## Method and honesty notes

- These are *reverse-engineered*. I inferred the decision and its rationale from the code; where the authors actually documented a decision themselves, I say so and mark the status accordingly rather than pretending I discovered it.
- The repo already contains three real ADRs in [docs/decisions/](../docs/decisions/README.md): ADR-001 (EF Core in the Application layer), ADR-002 (Aspire for orchestration/testing), ADR-003 (MediatR.Contracts in Domain). Two of my ten overlap those; I reverse-engineer them independently from code and then note the authors confirmed them. The other eight are genuinely undocumented.
- Anchors use `path#Lstart-Lend`. Line numbers were checked against the working tree (branch `main`, a clean checkout); if dependency-bump commits have since shifted lines, trust the path and symbol.
- This checkout compiles as the **SQLite + full-stack** template variant. `#if` blocks for other variants are inert here.

## The ten ADRs

| # | Decision | Category |
|---|---|---|
| [ADR-001](adr-001-clean-architecture-layering.md) | Four-layer Clean Architecture with dependency inversion via inward-owned interfaces | Layering / project structure |
| [ADR-002](adr-002-cqrs-lite-via-mediatr.md) | CQRS-lite: every use case is a MediatR command or query with one handler | Project structure |
| [ADR-003](adr-003-behaviour-pipeline.md) | Cross-cutting concerns as an ordered MediatR behaviour pipeline | App design / error handling |
| [ADR-004](adr-004-efcore-in-application-no-repository.md) | EF Core exposed via `IApplicationDbContext`; no repository layer | Data access |
| [ADR-005](adr-005-minimal-apis-reflection-registration.md) | Minimal APIs with convention/reflection endpoint registration | API design |
| [ADR-006](adr-006-contract-first-client-generation.md) | Contract-first typed clients; endpoint method name = OpenAPI operationId | API contract design |
| [ADR-007](adr-007-centralized-error-contract.md) | Typed exceptions → one ProblemDetails mapper; `Result` for expected failures | Error handling |
| [ADR-008](adr-008-real-database-functional-testing.md) | Functional tests run against a real database at the `ISender` seam | Testing approach |
| [ADR-009](adr-009-in-process-precommit-domain-events.md) | Domain events dispatched in-process, pre-commit, via a SaveChanges interceptor | Distributed systems |
| [ADR-010](adr-010-schema-by-convention-no-migrations.md) | Schema by convention with `EnsureCreated`; no migrations; provider-agnostic via switches | Data access / operations |

## What I deliberately left out of the top ten

Also-significant decisions that didn't make the cut, and where they surface: **two-layer attribute-driven authorization with no resource-level ownership** (treated as a load-bearing consequence inside [ADR-003](adr-003-behaviour-pipeline.md), because authorization here *is* a pipeline behaviour, and it's the single most interview-relevant gap in the codebase); **value objects + owned-entity mapping** (`Colour`); **Central Package Management** and **nullable + warnings-as-errors** as quality gates; **Aspire orchestration** ([ADR-002 in the real docs](../docs/decisions/ADR-002-Aspire-For-Orchestration-And-Testing.md), referenced from my ADR-008). I kept the ten that carry the most architectural weight and hit every category the analysis prioritized: layering, data access, API contract, error handling, testing, and distributed-systems concerns.

## Companion

[interview-translation.md](interview-translation.md) — for each of the ten, a 90-second spoken answer and the follow-up question an interviewer is most likely to ask next.
