# Validation, Authentication & Permissions

## The validation stack — four layers, four audiences

| Layer | Audience / purpose | Real example | If it's the *only* layer… |
|---|---|---|---|
| 1. HTML/UI hints | UX: fast feedback, fewer bad requests | `maxLength={200}` on inputs ([Todo.jsx L327](../../src/Web/ClientApp-React/src/components/Todo.jsx#L323-L327)), `type="email"` ([LoginPage.jsx L34](../../src/Web/ClientApp-React/src/components/api-authorization/LoginPage.jsx#L33-L37)) | trivially bypassed with curl |
| 2. Model binding | Protocol shape: wrong JSON types never reach code | Minimal API binding of `CreateTodoItemCommand`; malformed body → 400 pre-handler | can't express business rules |
| 3. FluentValidation via pipeline | **The contract authority** — every rule, aggregated errors, async DB checks | [ValidationBehaviour](../../src/Application/Common/Behaviours/ValidationBehaviour.cs#L15-L33) auto-runs any `IValidator<TRequest>`; [CreateTodoListCommandValidator](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs) incl. `MustAsync(BeUniqueTitle)` | racy for cross-row rules (no lock) |
| 4. Domain invariants | Last line: illegal states unrepresentable | [Colour.From](../../src/Domain/ValueObjects/Colour.cs#L5-L15) throws `UnsupportedColourException`; the [Done setter's](../../src/Domain/Entities/TodoItem.cs#L14-L26) transition rule | errors surface as 500s unless mapped |

Registration magic worth demystifying: `AddValidatorsFromAssembly` ([Application/DependencyInjection.cs L14](../../src/Application/DependencyInjection.cs#L14)) scans for validators; the behaviour receives `IEnumerable<IValidator<TRequest>>` ([ValidationBehaviour L8-L13](../../src/Application/Common/Behaviours/ValidationBehaviour.cs#L8-L13)) — zero validators is fine (queries mostly have none), several compose.

Error shape: failures group into `PropertyName → messages[]` ([ValidationException L13-L19](../../src/Application/Common/Exceptions/ValidationException.cs#L13-L19)) → 400 `ValidationProblemDetails` ([ProblemDetailsExceptionHandler L19-L23](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L19-L23)) → parsed by the client ([Todo.jsx L77-L83](../../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83)). One consistent error contract, end to end — ask for this in any codebase you join.

Gap to notice: layer 4 isn't mapped. Send colour `"#BADBAD"` and `UnsupportedColourException` becomes a **500**, not a 400 — domain invariant violations and validation failures produce different contracts. Deliberate? Probably indifferent. Fixable two ways (add colour rule to validators; or map the exception) — argue which is better. (Validator: keeps 400s at the contract layer and the error shape consistent. Mapping: catches *all* paths incl. future ones. Best answer: both.)

## Authentication — who you are

Cookie-based Identity in this variant: [Infrastructure/DependencyInjection.cs L58-L74](../../src/Infrastructure/DependencyInjection.cs#L58-L74) (`AddIdentityCookies`, `AddIdentityCore<ApplicationUser>` + roles + SignInManager); bearer-token mode exists for the API-only template variant ([L46-L56](../../src/Infrastructure/DependencyInjection.cs#L46-L56)) — being able to compare the two *from the same file* is a gift:

- **Cookie**: browser-managed, `HttpOnly` (XSS can't read it), sent automatically (CSRF becomes your problem), needs same-origin or careful CORS. Right for "SPA served by the API."
- **Bearer**: client-managed, sent explicitly (CSRF mostly moot), storable badly (XSS becomes your problem), works cross-origin/mobile. Right for public APIs.

The framework endpoints come from one line — `MapIdentityApi<ApplicationUser>()` ([Users.cs L12](../../src/Web/Endpoints/Users.cs#L12)) — register/login/refresh/2FA/manage. The client logs in with `?useCookies=true` ([AuthContext.jsx L19-L21](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx#L19-L21)) and detects sessions by probing `manage/info` ([L12-L17](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx#L12-L17)).

## Authorization — what you may do

Two enforcement points, by design:

1. **Endpoint groups**: `RequireAuthorization()` on TodoLists/TodoItems/WeatherForecasts groups ([TodoItems.cs L13](../../src/Web/Endpoints/TodoItems.cs#L13)) → 401 challenge at the HTTP edge.
2. **Request attributes**: `[Authorize]` on request records, enforced by [AuthorizationBehaviour](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs) — authenticated ([L29-L32](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L29-L32)), roles ([L35-L59](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L35-L59)), policies via [IdentityService.AuthorizeAsync](../../src/Infrastructure/Identity/IdentityService.cs#L52-L66) ([L62-L74](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L62-L74)). Semantics: multiple role-attributes AND together; roles within one attribute OR ([the loop at L41-L52](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L41-L52))… *actually check that claim yourself — it's drill 2.*

Why both? The behaviour works without HTTP (functional tests prove it: [ShouldDenyAnonymousUser](../../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L50-L58)), and survives new transports. The endpoint check fails faster and speaks proper HTTP (401 challenge vs exception mapping). Defense in depth — but note the inconsistency: only `GetTodosQuery` carries `[Authorize]` ([GetTodos.cs L9](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L9)); the write commands rely solely on the endpoint check. Send `CreateTodoItemCommand` through `ISender` with no user (as a test can) and it executes. Inconsistent depth = the depth you can't rely on. Good mid-level ticket: attribute all mutating commands.

## The missing third dimension: resource ownership

Authorization here answers *authenticated?* and *in role?* — never *"is this todo list yours?"* There is no ownership column on TodoList (only audit fields), no filter in queries ([GetTodos L43-L47](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47)), no check in handlers ([UpdateTodoItem L23-L34](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L23-L34)). Any authenticated user can read, modify, or delete any other user's data by iterating small integer IDs.

**What a junior misses:** nothing looks wrong — every endpoint "has auth."
**What a senior checks, in order:** ① does the resource have an owner/tenant column? ② is every *read* filtered by it? ③ does every *write* verify it before mutating? ④ is the check centralized (EF global query filter / policy handler) or copy-pasted? ⑤ do tests prove cross-user denial — the test that *should* exist here and doesn't.

This class of bug is **IDOR** (Insecure Direct Object Reference) / broken object-level authorization — OWASP API Top 10 #1. For this template it's a scoping decision, not a bug; in your production PRs it's the first thing to grep for. Designing the fix is [mid-level ticket M1](../06-contribution-practice/02-mid-level-feature-tickets.md).

## Drills

1. **Bypass ladder:** for each validation layer, describe how to bypass it and what stops you at the next. (curl bypasses 1; correct JSON bypasses 2; a race bypasses 3's uniqueness; nothing bypasses 4 except the `new Colour(...)` constructor hole — [Colour.cs L3](../../src/Domain/ValueObjects/Colour.cs#L3).)
2. **Read the role loop carefully** ([AuthorizationBehaviour L35-L59](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L35-L59)): with `[Authorize(Roles="A")] [Authorize(Roles="B")]` on one request and a user in role B only — authorized or not? Trace `authorized` through both iterations. (Answer: yes — one flag across all attributes, so attributes OR together, unlike ASP.NET Core's AND semantics. Subtle divergence; worth an issue upstream? Draft one as practice.)
3. **Threat sketch:** write the 5-step attack a hostile authenticated user runs against `/api/TodoLists` given sequential int IDs and no ownership checks. Then the 3-line defense plan.

## Interview angle

- *"Where does validation belong?"* → four layers/four audiences table. [08-interview-prep/03, Q4](../08-interview-prep/03-api-and-data-modeling-questions.md)
- *"AuthN vs AuthZ, and where does each live in your app?"* → this file's map. [08-interview-prep/03, Q3](../08-interview-prep/03-api-and-data-modeling-questions.md)
- *"What's IDOR and how do you prevent it structurally?"* → ownership section. [08-interview-prep/04, variation 1](../08-interview-prep/04-system-design-from-this-repo.md)
