# Runtime & Tooling Map

## Toolchain

| Tool | Version pin | Where declared |
|---|---|---|
| .NET SDK | 10.0.201, `rollForward: latestFeature` | [global.json](../../global.json) |
| Target framework | `net10.0`, nullable + implicit usings on, **warnings are errors** | [Directory.Build.props](../../Directory.Build.props#L3-L13) |
| NuGet versions | Central Package Management — every version lives in one file | [Directory.Packages.props](../../Directory.Packages.props) |
| Node | 24.x in CI ([build.yml](../../.github/workflows/build.yml#L41-L49)); any recent LTS locally | — |
| npm | Two independent client apps, each with its own lockfile | [ClientApp/package.json](../../src/Web/ClientApp/package.json), [ClientApp-React/package.json](../../src/Web/ClientApp-React/package.json) |
| Build output | Everything under `artifacts/` (the modern .NET `ArtifactsPath` layout) | [Directory.Build.props L10](../../Directory.Build.props#L10) |

`TreatWarningsAsErrors` is worth pausing on: it converts every compiler nudge into a broken build. That is a *cultural* decision — it keeps the template pristine, and it means your PRs fail CI for warnings that other repos tolerate.

## Commands (each marked verified/inferred)

| Task | Command | Status |
|---|---|---|
| Run full stack (DB + API + Angular SPA) | `dotnet run --project ./src/AppHost` | inferred ([README](../../README.md#L61-L65)) |
| Run API only | `dotnet run --project ./src/Web` | inferred ([launchSettings.json](../../src/Web/Properties/launchSettings.json) → http://localhost:5000, opens `/scalar`) |
| Build everything incl. clients + all tests | `pwsh ./build/build.ps1` | inferred ([build.ps1](../../build/build.ps1)) |
| Fast tests only (skips Playwright acceptance tests) | `pwsh ./build/build.ps1 -Target Basic` | inferred ([build.ps1 L31-L37](../../build/build.ps1#L31-L37)) |
| One test project | `dotnet test ./tests/Domain.UnitTests` | inferred |
| One test | `dotnet test ./tests/Application.FunctionalTests --filter "Name~ShouldCreateTodoItem"` | inferred (standard `dotnet test` filter syntax) |
| React dev server | `npm start` in `src/Web/ClientApp-React` (runs NSwag first via `prestart`) | inferred ([package.json](../../src/Web/ClientApp-React/package.json#L27-L35)) |
| Regenerate API clients | `npm run generate-api` in either client | inferred |
| Lint (React only) | `npm run lint` in ClientApp-React | inferred |
| Angular unit tests | `npm test` in `src/Web/ClientApp` (Karma/Jasmine) | inferred |
| Template matrix test (heavy: 9 combos) | `pwsh ./build/test.ps1` | inferred ([test.ps1](../../build/test.ps1)) |

There is **no** repo-wide lint/format command for C#; conventions come from `.editorconfig` + warnings-as-errors. There are **no** EF migrations to run — dev uses `EnsureCreated` (see [key flow 5](05-key-flows.md)).

## Runtime boundaries

Three processes in development, orchestrated by Aspire:

1. **The API** (`Web`) — ASP.NET Core, Kestrel, serves `/api/*`, `/openapi/v1.json`, `/scalar`, health checks `/health` & `/alive` (dev-only: [ServiceDefaults/Extensions.cs L101-L118](../../src/ServiceDefaults/Extensions.cs#L101-L118)), and — in production builds — the compiled SPA from `wwwroot` via `UseFileServer` + `MapFallbackToFile("index.html")` ([Program.cs](../../src/Web/Program.cs#L33-L48)).
2. **The SPA dev server** — Vite (React) or `ng serve` (Angular). In dev the browser talks to *this* server, which proxies API paths to the backend. The proxy target arrives via environment variables Aspire injects, named `services__webapi__https__0` ([vite.config.ts L4-L6](../../src/Web/ClientApp-React/vite.config.ts#L4-L6), [proxy.conf.js L3-L5](../../src/Web/ClientApp/proxy.conf.js#L3-L5)). This is Aspire *service discovery* — no hardcoded ports.
3. **The database** — SQLite file in this variant ([appsettings.json](../../src/Web/appsettings.json#L2-L4)); a container for PostgreSQL/SQL Server variants.

Transferable: **same-origin in production, proxied in development** is the standard SPA topology. It's why cookie auth works here without CORS pain — the browser always thinks it's talking to one origin. When you see a repo with a dev proxy config, immediately ask "so auth is cookies, and CSRF is handled how?" (see [05-quality-engineering/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md)).

## Environment variables (high level, no secrets)

| Variable | Purpose | Anchor |
|---|---|---|
| `ASPNETCORE_ENVIRONMENT` | Development gates DB reset/seed, health endpoints, Scalar behavior | [AppHost/Extensions.cs](../../src/AppHost/Extensions.cs#L6-L10) forwards it to child resources |
| `services__webapi__{https,http}__0` | Aspire-injected API URL for the SPA proxies | [vite.config.ts](../../src/Web/ClientApp-React/vite.config.ts#L4-L6) |
| `PORT` | Port Aspire assigns the SPA dev server | [AppHost/Program.cs L43](../../src/AppHost/Program.cs#L43) |
| `ConnectionStrings__CleanArchitectureDb` | DB connection; Aspire supplies it at run time, [appsettings.json](../../src/Web/appsettings.json#L2-L4) has the standalone fallback | [Infrastructure/DependencyInjection.cs L17-L18](../../src/Infrastructure/DependencyInjection.cs#L17-L18) |
| `AZURE_KEY_VAULT_ENDPOINT` | If set, config also loads from Key Vault | [Web/DependencyInjection.cs L39-L48](../../src/Web/DependencyInjection.cs#L39-L48) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | If set, telemetry exports via OTLP (Aspire dashboard sets it) | [ServiceDefaults/Extensions.cs L73-L80](../../src/ServiceDefaults/Extensions.cs#L73-L80) |

## CI

[build.yml](../../.github/workflows/build.yml): restore → build (Release) → install Playwright chromium → `dotnet test`. Note what CI does *not* run: the React app's lint, and the Angular Karma tests — `dotnet test` only runs the .NET test projects; the SPA builds happen because the Web project wraps them. [codeql.yml] adds static security analysis; [test-templates.yml] runs the 9-combination template matrix via [test.ps1](../../build/test.ps1).

## Drill

Without looking: which process serves `index.html` in development, and which in production? Then verify against [Program.cs L33-L48](../../src/Web/Program.cs#L33-L48) and [vite.config.ts](../../src/Web/ClientApp-React/vite.config.ts). **Strong** answer also explains what `MapFallbackToFile` is for (client-side routing deep links).
