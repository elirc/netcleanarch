# Security Checklist

Threat by threat, mapped to real code — including the ones this template doesn't handle (a template's scope, your production job). Then a pre-merge checklist you can actually use.

## Threat map

| Threat | Status here | Evidence / where |
|---|---|---|
| **Broken object-level auth (IDOR)** | ⚠️ **absent by scope** — the headline finding | No ownership column or filter; any user reads/writes any list ([GetTodos L43-L47](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47), [UpdateTodoItem L23-L34](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L23-L34)). OWASP API #1. Fix: [ticket M1](../06-contribution-practice/02-mid-level-feature-tickets.md) |
| **Broken function-level auth** | ⚠️ inconsistent | Only [GetTodosQuery](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L9) carries `[Authorize]`; mutating commands rely solely on endpoint `RequireAuthorization()` ([TodoItems L13](../../src/Web/Endpoints/TodoItems.cs#L13)). Two entry points, uneven depth ([03-architecture/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)) |
| **Authentication** | ✅ framework-grade | Identity via `MapIdentityApi` ([Users L12](../../src/Web/Endpoints/Users.cs#L12)); cookie (`HttpOnly`) or bearer per variant ([Infrastructure/DI L46-L74](../../src/Infrastructure/DependencyInjection.cs#L46-L74)) |
| **SQL injection** | ✅ prevented | EF Core parameterizes everything; no raw SQL/string concat in the codebase (grep `FromSql` → none) |
| **XSS** | ✅ mostly | React escapes by default; no `dangerouslySetInnerHTML`, no Angular `bypassSecurityTrust*`. Colours flow into inline `style` ([Todo.jsx L246](../../src/Web/ClientApp-React/src/components/Todo.jsx#L242-L248)) but are constrained by `Colour.From` server-side — *if* the constructor bypass ([Colour L3](../../src/Domain/ValueObjects/Colour.cs#L3)) ever let arbitrary strings in, that inline style is where they'd land. Defense-in-depth argument for closing the bypass |
| **CSRF** | ⚠️ think about it | Cookie auth = CSRF-relevant. Mitigations present: JSON content-type requirement (logout's `[FromBody] object` trick, [Users L19](../../src/Web/Endpoints/Users.cs#L19)), same-origin topology. No explicit antiforgery tokens on the custom endpoints — acceptable for JSON+cookie SPAs but worth stating deliberately |
| **CORS** | ⚠️ sloppy default | `AllowAnyOrigin/Method/Header` ([Program.cs L28-L31](../../src/Web/Program.cs#L28-L31)). Browsers refuse wildcard-origin + credentials, so cookie auth isn't *broken* by it, but it's a bad default to ship — tighten to known origins |
| **Secrets management** | ✅ path exists / ⚠️ dev seed | Key Vault wired when configured ([Web/DI L39-L48](../../src/Web/DependencyInjection.cs#L39-L48)); but hardcoded dev password `Administrator1!` ([initialiser L83](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L83)) and test creds — fine for dev/test, a finding if it ever escaped that boundary |
| **Sensitive data in logs** | ⚠️ latent | `{@Request}` logs whole commands ([LoggingBehaviour L32](../../src/Application/Common/Behaviours/LoggingBehaviour.cs#L32-L33), [UnhandledExceptionBehaviour L25](../../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L25)). Today commands hold no secrets; the day one carries a password/token/PII, it's in the log stream. Add destructuring policies before that day |
| **Mass assignment / over-posting** | ⚠️ watch | Commands *are* the bind target; a command that included, say, `OwnerId` or `Role` would let clients set it. Current commands are safe; the pattern invites the mistake when server-controlled fields get added |
| **SSRF** | ✅ n/a | No outbound requests from user input (the only HttpClient path is Aspire service discovery) |
| **Open redirect** | ✅ low risk | `returnUrl` handling is client-side and same-origin-pathed ([LoginPage L17](../../src/Web/ClientApp-React/src/components/api-authorization/LoginPage.jsx#L16-L18), [ProtectedRoute L9](../../src/Web/ClientApp-React/src/components/api-authorization/ProtectedRoute.jsx#L9)); still, validate `returnUrl` is a local path if you extend it |
| **Rate limiting / lockout** | ⚠️ defaults only | No rate limiting on login or API; Identity's default lockout only. Add ASP.NET Core rate limiting for a public deployment |
| **Input size limits** | ⚠️ gap | `Note` unbounded ([UpdateTodoItemDetail](../../src/Application/TodoItems/Commands/UpdateTodoItemDetail/UpdateTodoItemDetail.cs) has no rule; no DB max length) — a cheap DoS vector and a real ticket |
| **Dependency risk** | ✅ managed | Central versions ([Directory.Packages.props](../../Directory.Packages.props)) + Renovate bumps (git history) + [CodeQL workflow](../../.github/workflows/codeql.yml); `"nswag": "latest"` is the one unpinned dep |
| **Uploads / webhooks** | ✅ n/a | none in this app |

Legend: ✅ handled · ⚠️ gap or scope-limited. Nothing here is labeled a confirmed exploit — the two headline items (IDOR, function-level auth) are template scope decisions that become real vulnerabilities only in a multi-user deployment. Say it exactly that carefully in an interview; over-claiming "this repo has a vuln" reads as junior.

## The IDOR walkthrough (the one to master)

Attack, against the current code: authenticate as any user → `GET /api/TodoLists` returns *everyone's* lists → `DELETE /api/TodoLists/{anyId}` succeeds → iterate small integer ids to enumerate and destroy all data. Nothing stops it because authorization only asks "authenticated?" never "yours?".
Structural defense (not per-handler patching): `OwnerId` column → EF **global query filter** `HasQueryFilter(l => l.OwnerId == _currentUser.Id)` so *reads* are filtered everywhere at once → write-side ownership assert in a shared helper → return **404** (not 403) for foreign ids so you don't confirm they exist → functional tests proving cross-user denial ([recipe 3](02-writing-tests-here.md)). This is [ticket M1](../06-contribution-practice/02-mid-level-feature-tickets.md) and [system-design variation 1](../08-interview-prep/04-system-design-from-this-repo.md).

## Pre-merge security checklist (use on every PR)

- [ ] Any new query/command scoped to the current user where data is user-owned? (Once M1 lands: does it respect the query filter, and does the write path check ownership?)
- [ ] New command fields: are any server-controlled (owner, role, status)? If so, they must **not** be settable from the request body.
- [ ] New exception types: mapped to a deliberate HTTP status ([ProblemDetailsExceptionHandler](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs)), never leaking internals in `Detail`?
- [ ] New logged data: any chance of secrets/PII in `{@Request}` or a new log line?
- [ ] New string inputs: length-bounded in a validator *and* the DB config?
- [ ] New endpoints: `RequireAuthorization()` (or an explicit, reviewed reason it's anonymous) **and** an `[Authorize]` on the command for defense in depth?
- [ ] New outbound calls: is the URL user-influenced (SSRF)? New redirects: local-path validated?
- [ ] New dependency: pinned centrally, and does CodeQL/Renovate cover it?

## Drill

Run the checklist against a hypothetical PR that adds `POST /api/TodoLists/{id}/share` accepting `{ userId, role }`. List every box that trips. (At minimum: function-level auth — who may share?; mass-assignment of `role`; IDOR — is `{id}` yours to share?; the new relationship needs ownership rules of its own.) Strong answers notice sharing *changes the entire ownership model* the M1 filter assumes — a feature that looks small and isn't.

## Interview angle

- *"What's the most common API vulnerability and how do you prevent it?"* → IDOR + global query filter. [08-interview-prep/04, variation 1](../08-interview-prep/04-system-design-from-this-repo.md)
- *"Cookie vs token auth — security tradeoffs?"* → the threat-map rows + [03-architecture/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).
- *"How do you keep secrets out of logs?"* → the `{@Request}` latent issue and the redaction fix.
