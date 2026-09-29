# Tooling & Build System

Complements [01-codebase-cartography/04-runtime-and-tooling-map.md](../01-codebase-cartography/04-runtime-and-tooling-map.md) (the *what*); this file is the *why* and the transferable mechanics.

## MSBuild's inheritance trick: `Directory.Build.props` / `Directory.Packages.props`

Any `.csproj` automatically imports `Directory.Build.props` found in ancestor directories. This repo uses it to set, **once**, for all 13 projects: `net10.0`, nullable, implicit usings, warnings-as-errors, and the `artifacts/` output layout ([Directory.Build.props](../../Directory.Build.props)). [Directory.Packages.props](../../Directory.Packages.props) does the same for package *versions* (Central Package Management): projects reference `<PackageReference Include="MediatR" />` with no version, versions live centrally, and Renovate bumps one file — which is exactly what the recent git history is (`chore(deps)` commits).

Transferable: monorepo version skew ("project A on Moq 4.18, project B on 4.20") is a real failure class; CPM is .NET's answer, the same way a root `package.json` + workspaces is npm's. Interview line: *"one version of truth per dependency, enforced by the build, updated by a bot."*

## GlobalUsings: read these first or imports will confuse you

Each project has a `GlobalUsings.cs` that makes namespaces ambient — e.g. Application's ([GlobalUsings.cs](../../src/Application/GlobalUsings.cs)) pulls in `Ardalis.GuardClauses`, `AutoMapper`, `MediatR`, `FluentValidation`, `Microsoft.EntityFrameworkCore` everywhere. This is why [CreateTodoItem.cs](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs) uses `IRequest`, `Guard`, and `AbstractValidator` with almost no `using` lines. When a symbol appears from nowhere in any .NET repo, check GlobalUsings before assuming magic.

## The template preprocessor: this repo compiles in nine flavors

`#if (UseApiOnly)` / `#if (UsePostgreSQL)` in C# ([Infrastructure/DependencyInjection.cs L26-L74](../../src/Infrastructure/DependencyInjection.cs#L26-L74)), `<!--#if ... -->` in XML ([Directory.Packages.props L35-L51](../../Directory.Packages.props#L35-L51)), `//#if` in the use-case template ([CleanArchitectureUseCase.cs](../../templates/ca-use-case/FeatureName/Commands/CleanArchitectureUseCase/CleanArchitectureUseCase.cs)). These are `dotnet new` template conditions; when a user runs `dotnet new ca-sln -cf react -db postgresql`, the engine keeps/removes blocks. [test.ps1](../../build/test.ps1) exists to build and test all 3×3 combinations — the template's own integration test. When reading, always ask "which variant am I looking at?" — this checkout behaves as the *SQLite + non-API-only* variant since the `#if` symbols are undefined under normal compilation.

## Two client apps, one Web project

The Web `.csproj` hosts both SPA folders, but each SPA is an independent npm project with its own lockfile and scripts. The dev-time contract between them:

1. Aspire starts the API, then the SPA with env vars (`PORT`, `services__webapi__*`) — [AppHost/Program.cs L36-L46](../../src/AppHost/Program.cs#L36-L46).
2. `npm start` → `prestart` regenerates the typed client → dev server with proxy ([React scripts](../../src/Web/ClientApp-React/package.json#L27-L35), [Angular scripts](../../src/Web/ClientApp/package.json#L4-L15) — note Angular needs `run-script-os` for the `%PORT%` vs `$PORT` Windows/Unix split).
3. Production: `npm run build` output lands where `UseFileServer`/`MapFallbackToFile` ([Program.cs L33-L48](../../src/Web/Program.cs#L33-L48)) can serve it.

Sharp edge: NSwag reads `../wwwroot/openapi/v1.json` ([nswag.json L7](../../src/Web/ClientApp-React/nswag.json#L4-L11)) — a *file* the .NET build emits (via `Microsoft.Extensions.ApiDescription.Server`), not the live endpoint. So the generation order is: build .NET → OpenAPI file appears → npm can generate. If the client types look stale, rebuild the API first. Debugging codegen-order problems is a rite of passage in every generated-client shop.

## CI as the source of truth for "what must pass"

[build.yml](../../.github/workflows/build.yml): restore → `dotnet build -c Release` → install Playwright chromium → `dotnet test -c Release`. Notice `dotnet build` also builds the SPAs (the Web project triggers them), and Playwright browsers are installed from the *built artifacts* path ([L60-L63](../../.github/workflows/build.yml#L60-L63)). What CI **doesn't** run: `npm run lint`, Angular Karma tests. If you add either, you've made a real contribution (ticketed in [06-contribution-practice/01](../06-contribution-practice/01-good-first-tickets.md)).

Reading CI files is the fastest way to learn any repo's definition of done. Habit: open `.github/workflows/*` in your first ten minutes; whatever CI runs *is* the contract, whatever it skips is on you.

## Drills

1. Predict, then verify with [build.ps1](../../build/build.ps1#L31-L41): what's the difference between `build.ps1` and `build.ps1 -Target Basic`, and why would you use Basic locally? (Skips Playwright-driven AcceptanceTests — the slow, browser-dependent layer.)
2. Find the one package version *not* centrally managed. (Trick question — check both client `package.json` files: `"nswag": "latest"` is an unpinned npm dependency, [React](../../src/Web/ClientApp-React/package.json#L22) and [Angular](../../src/Web/ClientApp/package.json#L46). `latest` in a lockfile-managed app is mostly tamed, but it's still the loosest pin in the repo.)
3. You need to add `FluentAssertions` to one test project. Write the two edits CPM requires (version in Directory.Packages.props, versionless reference in the csproj).

## Interview angle

1. *"How do you keep dependency versions consistent across a large solution?"* → CPM above.
2. *"Walk me through your CI pipeline and what it catches."* → build.yml narration, plus what it misses.
3. *"How does your frontend get built and served in production vs dev?"* → the two-client section; [08-interview-prep/02, Q10](../08-interview-prep/02-frontend-framework-questions.md).
