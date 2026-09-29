# ADR-005: Minimal APIs with Convention/Reflection Endpoint Registration

**Status:** Accepted (implicit).

## Context

The HTTP surface needs a structure that stays thin (transport translation only, no business logic), consistent (uniform routes and OpenAPI tags), and low-friction to extend. The default risk is controllers accreting logic, or a manual registration list that drifts as endpoints are added.

## Decision

Use ASP.NET Core **Minimal APIs**, grouped one class per resource, each implementing [IEndpointGroup](../src/Web/Infrastructure/IEndpointGroup.cs) (which uses a C# 11 `static abstract Map`). At startup, [WebApplicationExtensions.MapEndpoints L12-L28](../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28) reflects over the Web assembly, finds every non-abstract `IEndpointGroup`, and registers each as a route group `/api/{ClassName}` with a matching OpenAPI tag — invoked from [Program.cs L45](../src/Web/Program.cs#L45). Handlers bind by parameter type (`ISender` from DI, `int id` from route, command from body) and return typed unions like `Results<NoContent, BadRequest>` ([TodoItems.cs L32](../src/Web/Endpoints/TodoItems.cs#L32)).

## Alternatives Considered

- **MVC controllers.** More ceremony, attribute routing, and a standing invitation to put logic in actions. Rejected — the endpoints here are deliberately anemic; controllers add weight without benefit.
- **Explicit `app.MapPost(...)` registration in `Program.cs`.** Direct, but grows into an unmaintainable wall and drifts from the handlers. Rejected in favor of convention: adding a class *is* adding routes.

## Consequences

**Positive**
- Adding an endpoint group requires zero registration edits — reflection finds it. Route and tag naming are consistent by construction.
- Typed `Results<...>` unions make the response contract compile-checked *and* visible to OpenAPI, so docs and code can't silently diverge on status codes.
- Endpoints stay thin: the fattest logic in the repo is a route/body id-consistency guard ([TodoItems.cs L34-L35](../src/Web/Endpoints/TodoItems.cs#L34-L35)).

**Negative**
- **Convention becomes load-bearing and invisible.** Nothing in an endpoint class says "I'm reachable at `/api/TodoItems`" except knowledge of the convention. Two groups with the same class name in different namespaces would collide on the same route prefix.
- **Visibility gotcha:** `GetExportedTypes()` sees public types only — mark an endpoint group `internal` and it silently vanishes from the API with no error.
- Method names leak into the public contract via operationId — a separate, significant decision covered in [ADR-006](adr-006-contract-first-client-generation.md).
- Reflection at startup (negligible here) and a small "magic" cost for newcomers who can't grep a registration list.

## Evidence

[IEndpointGroup](../src/Web/Infrastructure/IEndpointGroup.cs), [WebApplicationExtensions L12-L28](../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28), the endpoint groups themselves ([TodoLists.cs](../src/Web/Endpoints/TodoLists.cs), [TodoItems.cs](../src/Web/Endpoints/TodoItems.cs), [Users.cs](../src/Web/Endpoints/Users.cs)), and the id-consistency guard at [TodoItems.cs L34-L35](../src/Web/Endpoints/TodoItems.cs#L34-L35).
