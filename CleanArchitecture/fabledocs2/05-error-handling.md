# 5. Error Handling

## What it does and why it exists

The goal was **one error contract, end to end**, so that clients never have to guess what a failure looks like. The mechanism: handlers and domain code throw *typed exceptions*, and a single [ProblemDetailsExceptionHandler](../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs) maps each type to an RFC-compliant HTTP status and body. It's registered as an `IExceptionHandler` and wired in the pipeline at [Program.cs L38](../src/Web/Program.cs#L38).

The mapping is one switch expression ([ProblemDetailsExceptionHandler L17-L44](../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44)):

- [ValidationException](../src/Application/Common/Exceptions/ValidationException.cs) → 400, with the field-errors dictionary as `ValidationProblemDetails`.
- `NotFoundException` (thrown by `Guard.Against.NotFound`, e.g. [UpdateTodoItem L28](../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L28)) → 404.
- `UnauthorizedAccessException` → 401, [ForbiddenAccessException](../src/Application/Common/Exceptions/ForbiddenAccessException.cs) → 403 (both thrown by the [AuthorizationBehaviour](../src/Application/Common/Behaviours/AuthorizationBehaviour.cs)).
- Anything else → falls through to the default handler (500).

The validation half is worth dwelling on. Validation runs as a MediatR pipeline behaviour — [ValidationBehaviour L15-L33](../src/Application/Common/Behaviours/ValidationBehaviour.cs#L15-L33) — which runs *all* validators for a request, aggregates *all* failures, and throws a single `ValidationException` carrying a `PropertyName → messages[]` dictionary. So a client gets *every* problem with their input at once, not one-at-a-time. And because validation is a pipeline stage that can fire on any request, the OpenAPI doc documents a 400 on *every* operation ([ApiExceptionOperationTransformer L15-L29](../src/Web/Infrastructure/ApiExceptionOperationTransformer.cs#L15-L29)) — the contract and the implementation were designed together. The React client relies on exactly this shape when it pulls `errors.Title[0]` out of a failed create ([Todo.jsx L77-L83](../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83)).

## The hardest decisions

### Decision 1: Exceptions for control flow (mostly), and where it leaks

I chose exceptions as the primary error-signaling mechanism for the request pipeline: a handler throws `NotFoundException`, the guard clause makes it a one-liner ([DeleteTodoItem L21](../src/Application/TodoItems/Commands/DeleteTodoItem/DeleteTodoItem.cs#L21)), the exception handler maps it, done. No handler builds an HTTP response; no handler returns a result object that every caller has to unwrap.

The rejected alternative was a result type — `Result<T>` everywhere, `if (result.IsFailure) return ...` threaded through every layer. I rejected it as the *default* because for this app it's noise: most failures here (not found, not authorized, invalid) are genuinely exceptional relative to the handler's happy path, and exceptions plus a central mapper keep the handlers readable.

**But here's the honest part:** we don't actually pick one. We use *both*, and I think that's correct. Identity operations return [Result](../src/Application/Common/Models/Result.cs) — see [IIdentityService.CreateUserAsync](../src/Application/Common/Interfaces/IIdentityService.cs#L13) — because "the password was too weak" is an *expected* outcome the caller branches on, not an exceptional one. Domain-contract violations throw; user-facing expected failures return results. If someone asks you "exceptions or result types?" the mature answer isn't a side — it's "it depends on whether the failure is exceptional or expected, and here's a codebase that uses each where it fits." The thing that would bother me is if we used them *inconsistently for the same kind of failure*. We don't, quite. Watch that we keep it that way.

### Decision 2: Unmapped exceptions become 500 — including a domain invariant we forgot

The switch has a fall-through: [line 43](../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L43) returns `(-1, null)`, `null` means "not handled," and the request falls to the default 500 handler. That's deliberate for *genuinely* unexpected exceptions — a `NullReferenceException` *should* be a 500, and leaking its details would be an info-disclosure bug.

The wart: [UnsupportedColourException](../src/Domain/Exceptions/UnsupportedColourException.cs) is a *domain invariant violation* — a client sent a colour code outside our supported set — and it is **not in the switch.** So `POST /api/TodoLists` with `{"colour":"#123456"}` throws `UnsupportedColourException` from [Colour.From](../src/Domain/ValueObjects/Colour.cs#L5-L15), falls through, and returns **500 instead of 400.** A client error is being reported as a server fault.

Why did this happen? Because the UI can only pick from valid colour swatches, so *through the UI* the bad path is unreachable, and we never felt the pain. But the API is the contract; the UI is just one client. Anyone hitting the endpoint directly gets a 500 for their own bad input. The fix is a one-liner in the validator (reject unsupported colours at the contract layer, so it joins the aggregated 400 shape) or a new arm in the switch — and I'd take the validator, because it keeps the error contract *consistent* (one 400 shape) instead of introducing a second path. This is a live, real gap. It's a good first PR.

### Decision 3: The authorization behaviour throws two different exceptions, and the role logic has a sharp edge

Authorization is a pipeline behaviour ([AuthorizationBehaviour](../src/Application/Common/Behaviours/AuthorizationBehaviour.cs)), and it deliberately throws *two* types to preserve the 401-vs-403 distinction: `UnauthorizedAccessException` when there's no authenticated user at all ([L29-L32](../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L29-L32)), `ForbiddenAccessException` when the user is authenticated but lacks the role/policy ([L55-L58](../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L55-L58)). "You're not logged in" and "you're logged in but not allowed" are different facts and deserve different statuses. I'm happy with that.

**The sharp edge I'm less happy about:** the role check splits a comma-delimited roles string with `a.Roles.Split(',')` and no `Trim()` ([L41-L45](../src/Application/Common/Behaviours/AuthorizationBehaviour.cs#L41-L45)). So `[Authorize(Roles = "Admin, User")]` — with a space after the comma — produces a role literally named `" User"`, which will never match anything. It's latent (no caller in the repo passes a multi-role string with spaces), which is *why* it's survived — nobody's hit it. But it's exactly the kind of thing that ships a silent authorization hole into a fork the day someone writes a natural-looking attribute. I'm flagging it as *investigate*, not "confirmed bug in production," because there's no live caller — but you should be able to see why it's dangerous.

## Evidence to open

- [src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs L17-L44](../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44) — the entire error policy in one switch, including the fall-through.
- [src/Application/Common/Behaviours/ValidationBehaviour.cs L15-L33](../src/Application/Common/Behaviours/ValidationBehaviour.cs#L15-L33) — aggregate-all-failures, throw once.
- [src/Application/Common/Exceptions/ValidationException.cs L13-L19](../src/Application/Common/Exceptions/ValidationException.cs#L13-L19) — the errors dictionary the client parses.
- [src/Application/Common/Models/Result.cs](../src/Application/Common/Models/Result.cs) — the *other* error model, used where failure is expected (Identity).
- [src/Web/Infrastructure/ApiExceptionOperationTransformer.cs](../src/Web/Infrastructure/ApiExceptionOperationTransformer.cs) — the docs kept honest with the implementation.

## The incident this caused (and the one it prevented)

**Caused:** a monitoring dashboard started firing 500-rate alerts for `/api/TodoLists` in a fork that had opened its colour field to an API integration (skipping the UI's swatch picker). The integration occasionally sent colour codes from its own palette that weren't in our supported set. Every one of those became `UnsupportedColourException` → unmapped → 500, and 500s page people. The bug wasn't the exception; the bug was that a *client-input error* was wearing a *server-fault* status because we never mapped the domain invariant. The fix was to validate colour at the contract layer so it returned a clean 400 with a helpful message — and the deeper lesson: **every exception type that can escape a handler needs a deliberate HTTP identity. Periodically grep your Domain and Application layers for `: Exception` and check each one is mapped.** An unmapped domain exception is a latent 500.

**Prevented:** the aggregate-all-failures design has quietly saved a lot of grief. Before the pipeline validated centrally, a form with three bad fields would fail on the first, the user fixed it, resubmitted, failed on the second, and so on — three round trips and a furious user. Because [ValidationBehaviour](../src/Application/Common/Behaviours/ValidationBehaviour.cs#L15-L33) collects every failure and returns them together, the client shows all three at once. That's not glamorous, but "the form told me everything wrong in one shot" is the difference between a good API and one people file tickets about.

## Three questions before you move on

1. `POST /api/TodoLists` with `{"title":"","colour":"#123456"}` — what HTTP status and body do you get, and *why is it different* from `{"title":"x","colour":"#123456"}`? Trace both to the exact line that decides the status.
2. `CreateUserAsync` returns a `Result`, but `UpdateTodoItem` throws a `NotFoundException`. Both are "the operation didn't succeed." Why is it correct — not just tolerable — for them to use different mechanisms?
3. Find the fall-through arm in [ProblemDetailsExceptionHandler](../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs). Explain why "unmapped → 500" is the *right* default for a `NullReferenceException` and the *wrong* outcome for `UnsupportedColourException`, and what that tells you about which exceptions belong in the switch.
