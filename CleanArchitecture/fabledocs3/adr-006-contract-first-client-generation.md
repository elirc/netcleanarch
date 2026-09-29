# ADR-006: Contract-First Typed Clients; Method Name = OpenAPI operationId

**Status:** Accepted (implicit).

## Context

Two SPA frontends (Angular and React) consume this API. Hand-written fetch wrappers drift from the server contract — a renamed field or endpoint breaks the client silently at runtime. The team wanted contract changes to surface as *build* failures on the client, not runtime surprises.

## Decision

Treat the server's **OpenAPI document as the source of truth** and generate typed clients from it with NSwag, regenerating on every SPA build. The React config ([nswag.json](../src/Web/ClientApp-React/nswag.json)) reads `../wwwroot/openapi/v1.json` and emits `src/web-api-client.ts`, wired as a `prestart`/`prebuild` hook ([package.json L27-L35](../src/Web/ClientApp-React/package.json#L27-L35)); the Angular client mirrors it. The generated file is **not** committed.

The OpenAPI `operationId` is derived from the endpoint **handler method name** — [EndpointRouteBuilderExtensions L20-L26](../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs#L20-L26) calls `.WithName(handler.Method.Name)`. So `GetTodoLists` (C#) becomes `getTodoLists()` (TypeScript). The doc is also kept honest about errors: every operation documents a 400 because the pipeline can throw one anywhere ([ApiExceptionOperationTransformer L15-L29](../src/Web/Infrastructure/ApiExceptionOperationTransformer.cs#L15-L29)).

## Alternatives Considered

- **Hand-written client SDKs.** Full control, guaranteed drift. Rejected — the entire point is eliminating drift.
- **Explicit operationId string literals** decoupled from method names. Rejected because the string drifts when the method is renamed and nobody updates it; deriving from the name keeps them in lockstep. (This trades one drift risk for a worse one — see Negative.)
- **Shared-types approaches (tRPC-style) with no codegen.** Not available across a .NET-to-TypeScript boundary; codegen is the idiomatic bridge.

## Consequences

**Positive**
- A server contract change becomes a client build failure — rename a DTO property and the regenerated client won't compile against old call sites.
- Server annotations (`[EndpointSummary]`, etc.) flow into docs and client comments.
- One generation pipeline serves both frontends from the same document.

**Negative**
- **A method rename is a breaking API change that looks local.** Renaming `GetTodoLists` changes the operationId, changes both clients' generated method names, and breaks every call site (e.g. [Todo.jsx L33-L42](../src/Web/ClientApp-React/src/components/Todo.jsx#L33-L42)) plus any external OpenAPI consumer. The blast radius is invisible from the method being renamed.
- **Generation ordering:** NSwag reads a *file* the .NET build emits, not the live endpoint — so the API must be built before the client can regenerate. Stale-client bugs come from getting this order wrong.
- Editing the generated file is pointless (overwritten on regen), and the type safety it provides stops at the network boundary — the React app's own components are untyped `.jsx`, so drift can still slip through inside the UI. `"nswag": "latest"` is also an unpinned dependency.

## Evidence

[nswag.json](../src/Web/ClientApp-React/nswag.json), the `prestart` hook in [package.json L27-L35](../src/Web/ClientApp-React/package.json#L27-L35), the operationId derivation at [EndpointRouteBuilderExtensions L20-L26](../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs#L20-L26), consumption at [Todo.jsx L3-L6](../src/Web/ClientApp-React/src/components/Todo.jsx#L3-L6), and the error-documentation transformer [ApiExceptionOperationTransformer](../src/Web/Infrastructure/ApiExceptionOperationTransformer.cs).
