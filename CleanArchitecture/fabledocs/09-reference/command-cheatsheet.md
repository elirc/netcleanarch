# Command Cheatsheet

Each command marked **verified** (run while writing these docs) or **inferred** (read from scripts/CI/README, not executed here). See [verification-log](verification-log.md) for what was actually run. Shell is PowerShell on Windows; `pwsh` scripts are cross-platform.

## Environment facts (verified)
- .NET SDK installed: `10.0.302` (repo pins `10.0.201` + `rollForward: latestFeature`, [global.json](../../global.json)) — `dotnet --list-sdks`
- Node: `v22.16.0` — `node --version`
- 231 tracked files — `rg --files`

## Build

| Task | Command | Status |
|---|---|---|
| Restore | `dotnet restore` | inferred ([build.yml L54-L55](../../.github/workflows/build.yml#L54-L55)) |
| Build (Release) | `dotnet build --configuration Release` | inferred |
| Full build incl. clients + tests | `pwsh ./build/build.ps1` | inferred ([build.ps1](../../build/build.ps1)) |
| Build without acceptance tests | `pwsh ./build/build.ps1 -Target Basic` | inferred ([build.ps1 L31-L37](../../build/build.ps1#L31-L37)) |

## Run

| Task | Command | Status |
|---|---|---|
| Full stack (Aspire: DB + API + Angular) | `dotnet run --project ./src/AppHost` | inferred ([README L61-L65](../../README.md#L61-L65)) |
| API only | `dotnet run --project ./src/Web` → http://localhost:5000, opens `/scalar` | inferred ([launchSettings.json](../../src/Web/Properties/launchSettings.json)) |

## Test

| Task | Command | Status |
|---|---|---|
| All tests | `dotnet test` | inferred ([build.yml L65-L66](../../.github/workflows/build.yml#L65-L66)) |
| One project | `dotnet test ./tests/Domain.UnitTests` | inferred |
| One test by name | `dotnet test ./tests/Application.FunctionalTests --filter "Name~ShouldCreateTodoItem"` | inferred (standard filter) |
| By namespace/class | `dotnet test --filter "FullyQualifiedName~TodoLists"` | inferred |
| Template matrix (9 combos, heavy) | `pwsh ./build/test.ps1` | inferred ([test.ps1](../../build/test.ps1)) |
| One template combo | `pwsh ./build/test.ps1 -ClientFramework react -Database sqlite` | inferred ([test.ps1 L1-L4](../../build/test.ps1#L1-L4)) |

Test project fast/slow: Domain.UnitTests + Application.UnitTests are fast (no infra). Application.FunctionalTests boots Aspire DB (seconds). Web.AcceptanceTests boots the full stack + Playwright (minutes) — excluded by `-Target Basic`.

## Frontend (run inside `src/Web/ClientApp-React` or `src/Web/ClientApp`)

| Task | Command | Status |
|---|---|---|
| Install deps | `npm ci` | inferred ([build.ps1 L20](../../build/build.ps1#L20)) |
| Dev server (regenerates API client first) | `npm start` | inferred ([package.json scripts](../../src/Web/ClientApp-React/package.json#L27-L35)) |
| Build | `npm run build` | inferred |
| Lint (React only) | `npm run lint` | inferred ([package.json L33](../../src/Web/ClientApp-React/package.json#L33)) |
| Angular unit tests | `npm test` (Karma/Jasmine) | inferred ([ClientApp/package.json L13](../../src/Web/ClientApp/package.json#L13)) |
| Regenerate typed API client | `npm run generate-api` (NSwag; needs the .NET build's `wwwroot/openapi/v1.json` first) | inferred ([nswag.json](../../src/Web/ClientApp-React/nswag.json)) |

## Migrations — none exist

This repo uses `EnsureDeleted/EnsureCreated` in dev ([initialiser L44-L46](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46)); there is **no** migrations folder. If you add migrations ([ticket M9](../06-contribution-practice/02-mid-level-feature-tickets.md)), the commands would be (inferred, EF Design pkg is referenced [Directory.Packages.props L22](../../Directory.Packages.props#L22)):

```bash
# inferred — will not work until migrations are introduced
dotnet ef migrations add <Name> --project src/Infrastructure --startup-project src/Web
dotnet ef database update --project src/Infrastructure --startup-project src/Web
```

## Central package management

Add a NuGet package: put the version in [Directory.Packages.props](../../Directory.Packages.props), then reference it versionless in the target `.csproj` (`<PackageReference Include="X" />`). Do **not** put versions in csproj files ([tooling](../02-stack-and-language-mastery/04-tooling-and-build-system.md)).

## Docker

Not required for the SQLite (default) variant. Required for PostgreSQL/SQL Server variants (Aspire runs the DB as a container). This checkout is the SQLite variant ([appsettings.json](../../src/Web/appsettings.json#L2-L4)).

## Handy repo exploration (verified as a category — Grep/Glob tools used throughout)

| Task | Command |
|---|---|
| List all files | `rg --files` |
| Find layer violations | `rg "CleanArchitecture.Infrastructure" src/Application src/Domain` (should be empty) |
| Find type-system escape hatches | `rg "null!\|as \| any\b\|dynamic"` |
| Find all endpoint groups | `rg "IEndpointGroup" src/Web` |

## Default dev credentials (dev/seed only — [initialiser L79-L83](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L79-L83))
`administrator@localhost` / `Administrator1!` — created only in Development, recreated each run. Never a production credential.
