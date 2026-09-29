# ADR-007: Typed Exceptions → One ProblemDetails Mapper; `Result` for Expected Failures

**Status:** Accepted (implicit).

## Context

An API needs one predictable error shape so clients don't guess what a failure looks like. Handlers shouldn't each build HTTP responses, and error-signaling shouldn't be threaded as return values through every layer when most failures are genuinely exceptional relative to the happy path.

## Decision

Handlers and domain code **throw typed exceptions**; a single [ProblemDetailsExceptionHandler L17-L44](../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44) maps each type to an RFC-compliant status and body: [ValidationException](../src/Application/Common/Exceptions/ValidationException.cs) → 400 (with a `PropertyName → messages[]` dictionary), `NotFoundException` → 404 (thrown by `Guard.Against.NotFound`, e.g. [UpdateTodoItem L28](../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L28)), `UnauthorizedAccessException` → 401, `ForbiddenAccessException` → 403; anything unmapped falls through to a default 500 ([L43](../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L43)). Validation runs as a pipeline behaviour that aggregates *all* failures and throws once ([ValidationBehaviour L15-L33](../src/Application/Common/Behaviours/ValidationBehaviour.cs#L15-L33)).

Crucially, this is *not* dogmatic: where a failure is **expected** rather than exceptional, the code returns a [Result](../src/Application/Common/Models/Result.cs) object instead — see [IIdentityService.CreateUserAsync](../src/Application/Common/Interfaces/IIdentityService.cs#L13), because "password too weak" is a normal branch, not an exception.

## Alternatives Considered

- **Per-endpoint error handling.** Every endpoint builds its own responses; guaranteed drift in error shape. Rejected — one contract, one mapper.
- **`Result<T>` everywhere (railway-oriented).** Explicit and allocation-friendly, but threads `if (result.IsFailure) return ...` through every layer and turns not-found/not-authorized into ceremony. Rejected as the *default*, but deliberately kept for expected-failure cases — the codebase uses both, chosen by whether the failure is exceptional or expected.

## Consequences

**Positive**
- One error shape end to end; the client parses `errors.Title[0]` from the 400 body directly ([Todo.jsx L77-L83](../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83)).
- Aggregated validation returns every input error in one response — one round trip.
- The contract and its documentation are co-designed: the OpenAPI transformer advertises exactly the statuses the pipeline can produce ([ApiExceptionOperationTransformer](../src/Web/Infrastructure/ApiExceptionOperationTransformer.cs)).

**Negative**
- **Unmapped exceptions become 500.** That's correct for a `NullReferenceException` but wrong for a *domain invariant* violation: [UnsupportedColourException](../src/Domain/Exceptions/UnsupportedColourException.cs) is not in the switch, so a bad colour via the API returns 500 instead of 400. Reachable only outside the UI (which restricts colours to valid swatches), which is why it survived — but the API is the contract, not the UI. A real, live gap.
- Exceptions-as-control-flow has a cost on hot paths and can be misused for expected outcomes (mitigated here by the `Result` duality, but only by discipline).
- The dual model (exceptions *and* `Result`) works only as long as the same *kind* of failure is handled the same way; inconsistency here would be confusing.

## Evidence

[ProblemDetailsExceptionHandler L17-L44](../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44) (the entire policy in one switch, including the fall-through), [ValidationBehaviour L15-L33](../src/Application/Common/Behaviours/ValidationBehaviour.cs#L15-L33), [ValidationException L13-L19](../src/Application/Common/Exceptions/ValidationException.cs#L13-L19), and the expected-failure path [Result](../src/Application/Common/Models/Result.cs) + [IIdentityService](../src/Application/Common/Interfaces/IIdentityService.cs#L13).
