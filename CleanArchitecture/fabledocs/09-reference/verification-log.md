# Verification Log

Running record of what was actually inspected and executed while writing this curriculum. Anything not listed under "Commands run" was **not executed** — commands in the docs derived from scripts/CI/README are marked *inferred*.

**Date:** 2026-07-16. **Repo state:** branch `main`, HEAD `43831e2` ("chore(deps): update all non-major dependencies (#1606)"), clean working tree.

## Commands run (verified)

| Command | Result |
|---|---|
| `rg --files` | 231 files enumerated |
| `git status --short` / `git log --oneline -5` | clean tree; recent history is Renovate dependency bumps |
| `ls .github/workflows` | build.yml, codeql.yml, release.yml, test-templates.yml |
| `ls tests` (and recursive listings) | 6 test projects incl. Application.UnitTests and Web.AcceptanceTests (present despite being missed by the first truncated file listing) |
| `dotnet --list-sdks` | 10.0.302 installed (repo pins 10.0.201 with `rollForward: latestFeature` — compatible) |
| `node --version` | v22.16.0 |
| `git check-ignore tests/Web.AcceptanceTests` | not ignored |

**Not run:** `dotnet build`, `dotnet test`, `dotnet run --project src/AppHost`, `npm ci`/`npm start`, NSwag generation, build.ps1/test.ps1. All run/test/build commands in these docs are therefore *inferred* from [README.md](../../README.md), [build/build.ps1](../../build/build.ps1), [build/test.ps1](../../build/test.ps1), [.github/workflows/build.yml](../../.github/workflows/build.yml), and the two client `package.json` files.

## Files read in full

All of: `src/Web` (Program, DependencyInjection, all 4 endpoint groups, all 7 Infrastructure helpers, CurrentUser, appsettings, launchSettings, Web.http), `src/Application` (all behaviours, all TodoItems/TodoLists/WeatherForecasts commands+queries+validators+DTOs, interfaces, exceptions, models, security attribute, DI), `src/Domain` (all entities, common bases, value objects, events, enums, constants, exceptions), `src/Infrastructure` (DI, DbContext, initialiser, both interceptors, both configurations, all Identity files), `src/AppHost`, `src/ServiceDefaults`, `src/Shared`, `tests/Application.FunctionalTests` (setup, infrastructure, representative tests), `tests/Domain.UnitTests/ValueObjects/ColourTests.cs`, `tests/Application.UnitTests` (RequestLoggerTests, MappingTests), `tests/Web.AcceptanceTests` (AspireSetup, PlaywrightSetup, Login feature/steps/page), `tests/TestAppHost`, both ADR sets in `docs/decisions`, `templates/ca-use-case` (both), root metadata (README, slnx, Directory.Build.props, Directory.Packages.props, global.json, build.yml), React client (App, AppRoutes, Todo, Weather, ThemeContext, AuthContext, LoginPage, ProtectedRoute, vite.config, nswag.json, package.json), Angular client (todo.component.ts first 120 lines, auth.service, auth.guard, authorize.interceptor, proxy.conf, package.json).

**Read partially:** `src/Web/ClientApp/src/app/todo/todo.component.ts` (lines 1–120 of a longer file). **Not read:** remaining Angular components/templates/scss, `ValidationExceptionTests.cs`, remaining acceptance-test pages/steps (Counter/Home/Weather), `karma.conf.js`, lockfiles, `release.yml`, `codeql.yml` bodies, `repack.ps1`, `CleanArchitecture.nuspec`, `renovate.json`.

## Uncertainties and how they are labeled in the docs

| Claim | Status |
|---|---|
| Role list `Split(',')` without `Trim()` in [AuthorizationBehaviour.cs L41-L45](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L41-L45) would fail to match `"Admin, User"` | Labeled *investigate* — true by inspection of the string comparison, but no in-repo caller passes multi-role strings, so it is latent, not live |
| Unique-title validator race ([CreateTodoListCommandValidator.cs L21-L26](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26)) | Confirmed by inspection (no unique index in [TodoListConfiguration.cs](../../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs)); race not demonstrated by a test |
| `Created` Location header path mismatch ([TodoLists.cs L36](../../src/Web/Endpoints/TodoLists.cs#L36) yields `/TodoLists/{id}`, real route `/api/TodoLists`; also no GET-by-id endpoint exists) | Confirmed by inspection; cosmetic |
| [Web.http](../../src/Web/Web.http) is stale (references `/Identity/Account/Login` Razor pages, `GET /api/TodoLists/1`, paginated `GET /api/TodoItems`, `PUT UpdateItemDetails?Id=1` — none exist in current endpoint groups; [Web-webapi.http](../../src/Web/Web-webapi.http) not fully audited) | Confirmed for the listed routes by comparing against the four endpoint groups |
| CORS `AllowAnyOrigin` + cookie auth interaction ([Program.cs L28-L31](../../src/Web/Program.cs#L28-L31)) | Reasoned from browser CORS spec (credentialed requests can't use wildcard origins), not tested against the running app |
| Whether `EnrichNpgsqlDbContext`/retry-on-failure affects the SQLite variant | Not applicable in SQLite branch by inspection of `#if` blocks |
| Acceptance tests only cover Home/Counter/Weather/Login features (no Todo feature file) | Confirmed by directory listing |
| App behavior at runtime (dashboard URLs, seed timing, generated client shape) | Described from code; **not observed live** |

## Anchor freshness

Every `path#Lx-Ly` anchor was written against file contents read during this session (2026-07-16, HEAD `43831e2`); files were not modified by this work (docs added under `fabledocs/` only). If dependency-bump PRs have since moved lines, prefer symbol search over line numbers.
