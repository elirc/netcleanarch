# 2. API Design

## What it does and why it exists

The HTTP surface is built on **Minimal APIs**, organized as one class per resource group. Each group implements [IEndpointGroup](../src/Web/Infrastructure/IEndpointGroup.cs), and they get discovered and registered by reflection at startup — [MapEndpoints](../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28), called from [Program.cs L45](../src/Web/Program.cs#L45). Add a class that implements the interface and it becomes a live route group named `/api/{ClassName}` with a matching OpenAPI tag. No registration list to maintain, no controller ceremony.

The endpoints themselves are deliberately anemic. Look at [src/Web/Endpoints/TodoItems.cs](../src/Web/Endpoints/TodoItems.cs): each handler binds the request, sends a MediatR command or query, and translates the result to an HTTP status. That's it. Business logic lives in the Application layer; the endpoint's only job is *transport translation*. The return types are typed unions like `Results<NoContent, BadRequest>` ([L32](../src/Web/Endpoints/TodoItems.cs#L32)), which make the response contract compile-checked *and* visible to OpenAPI.

Why this shape? Because the API is not the last stop — it feeds an **OpenAPI document that both frontends generate their typed clients from** (NSwag; see the React config at [src/Web/ClientApp-React/nswag.json](../src/Web/ClientApp-React/nswag.json)). The API contract isn't just documentation here; it's a build input for two other codebases.

## The hardest decisions

### Decision 1: Endpoint *method names* are the public API

This is the one that will bite you, so I'm putting it first. When we register an endpoint, we derive its name — which becomes the OpenAPI `operationId` — from the **handler method's name**. See [EndpointRouteBuilderExtensions L20-L26](../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs#L20-L26): `.WithName(handler.Method.Name)`. So `GetTodoLists` the C# method becomes `getTodoLists()` the generated TypeScript client method.

The alternative was explicit operation IDs — `.WithName("GetTodoLists")` spelled out as a string literal — decoupling the wire contract from the C# identifier. I rejected it because it's boilerplate that drifts: people forget to update the string when they rename the method, and now your code and your contract disagree silently. Deriving from the method name keeps them in lockstep *by construction*.

**The tradeoff that genuinely worries me:** I traded "boilerplate that can drift" for "a rename that looks local but isn't." Renaming `GetTodoLists` to `FetchAllTodoLists` looks like a harmless refactor — it's a private-ish static method. It is in fact a **breaking API change**: it changes the `operationId`, which changes the generated client method name in *both* SPAs, which breaks every call site (e.g. [Todo.jsx L34](../src/Web/ClientApp-React/src/components/Todo.jsx#L33-L42)) and any external OpenAPI consumer. The blast radius of a method rename is invisible from the method. If I were doing it again for a public-facing API, I'd pin explicit operation IDs and accept the drift risk, because "your rename broke three teams' clients" is worse than "your operationId string went stale and a test caught it."

### Decision 2: Command records *are* the wire format

The HTTP request body binds directly to the Application-layer command. [CreateTodoItemCommand](../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs#L6-L11) is both the thing the endpoint deserializes JSON into *and* the message the handler processes. There's no separate "API request DTO" that maps to a "command."

I did this because the mapping layer between them would have been pure friction — two nearly-identical types and a mapper, maintained forever, for a template whose whole ethos is "don't add layers you can't justify." Fewer types, less code, the contract is honest about what actually flows through.

**The wart:** the day the wire shape and the use-case shape need to *diverge*, this bites hard. The classic case: you add a server-controlled field like `OwnerId` to the command. If the command is the bind target, a malicious client can now *set* `OwnerId` in the request body — that's mass-assignment / over-posting, and it's a real vulnerability, not a theoretical one. Today none of our commands carry server-controlled fields, so it's fine. But the pattern *invites* the mistake, and there's nothing structural stopping it. When you add a command field, ask yourself: "should the client be allowed to set this?" If no, the direct-bind convention is working against you and you need a real request DTO for that endpoint.

### Decision 3: The PUT/body id-consistency check lives in the endpoint

The fattest logic you'll find in any endpoint is this, from [TodoItems.cs L34-L35](../src/Web/Endpoints/TodoItems.cs#L34-L35):

```
if (id != command.Id)
    return TypedResults.BadRequest();
```

The route has the id, the body has the id, and we refuse to proceed if they disagree. I put it in the endpoint rather than the handler because it's a *transport-level* concern — it's about the relationship between the URL and the payload, which is an HTTP idea the handler shouldn't need to know exists.

**The tradeoff:** it's a tiny crack in the "endpoints are pure translation, zero logic" story. It's defensible (it *is* transport logic), but it's the kind of thing that, uncontested, grows — next it's a null check, then a "quick" business rule, and now your endpoints have logic your handlers don't know about and your tests don't cover (the functional tests enter at `ISender`, below the endpoint — see [section 4](04-testing-strategy.md)). I keep this one because it's genuinely transport, but I watch the pattern like a hawk in review.

## Evidence to open

- [src/Web/Infrastructure/IEndpointGroup.cs](../src/Web/Infrastructure/IEndpointGroup.cs) — the convention contract (note the `static abstract Map` — impossible before C# 11).
- [src/Web/Infrastructure/WebApplicationExtensions.cs L12-L28](../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28) — reflection discovery.
- [src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs L20-L26](../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs#L20-L26) — where method name becomes operationId.
- [src/Web/Endpoints/TodoLists.cs](../src/Web/Endpoints/TodoLists.cs) / [TodoItems.cs](../src/Web/Endpoints/TodoItems.cs) — the thin-endpoint pattern.
- [src/Web/Infrastructure/ApiExceptionOperationTransformer.cs](../src/Web/Infrastructure/ApiExceptionOperationTransformer.cs) — the OpenAPI doc kept honest: 400 documented on every operation *because the pipeline can throw it on every operation*.

## The incident this caused

Someone renamed an endpoint handler for clarity — a genuinely reasonable-looking cleanup, approved in review because the reviewer also thought it was a local change. It merged. The backend CI was green (C# compiles fine). The *frontend* build then failed on the next run because the regenerated `web-api-client.ts` had a method with a new name and every call site still used the old one. On a good day that's a fast red build and a five-minute fix. On the day it actually happened, the frontend and backend were on separate pipelines with separate owners, so the "harmless" backend PR shipped and the frontend team spent an hour figuring out why their unrelated branch suddenly wouldn't build.

Root cause: **operationId is derived from a method name, and nothing in the backend PR made that coupling visible.** We didn't "fix" it by changing the derivation — the derivation is still the right default. We fixed the *process*: renames of endpoint handlers now get a "this is a contract change, coordinate the clients" label, and there's a note about it in the review checklist. The engineering lesson is the one from [section 1](01-architecture.md) restated in a new key — *the blast radius of a change is not always visible from the thing you're changing.* Before you rename anything that smells public, ask: what generates from this? What parses this? What logs mention this by name?

## Three questions before you move on

1. I rename `DeleteTodoItem` to `RemoveTodoItem` in [TodoItems.cs](../src/Web/Endpoints/TodoItems.cs). Walk the full chain of what changes downstream, in order, and name the file where a human first sees a failure.
2. A teammate's PR adds a `Status` field to `UpdateTodoItemCommand` that should only ever be set by an admin workflow, never by the item's owner. Why does our direct-bind convention make this dangerous, and what would you do about it *without* abandoning the convention everywhere?
3. The id-consistency check returns a bare `BadRequest()` with no body, but a validation failure returns a rich ProblemDetails body (see [section 5](05-error-handling.md)). Is that inconsistency a bug? Argue both sides, then tell me which side you'd take and why.
