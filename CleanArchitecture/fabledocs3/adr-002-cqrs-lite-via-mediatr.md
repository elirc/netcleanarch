# ADR-002: CQRS-lite — Every Use Case Is a MediatR Command or Query

**Status:** Accepted (implicit).

## Context

The Application layer needs a unit of organization for business operations. The two failure modes to avoid are (a) fat "service" classes that accrete unrelated methods and become merge-conflict magnets, and (b) reads and writes forced through one shape when they have opposite needs (writes want change-tracking and transactions; reads want projection and no tracking).

## Decision

Model each use case as a discrete message object implementing MediatR's `IRequest`, with exactly one handler. Commands mutate; queries read. They live in feature folders organized by operation — e.g. [src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs](../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs) holds the `CreateTodoItemCommand` record *and* its handler in one file. Endpoints dispatch through `ISender.Send(...)` and never call handlers directly. Registration is assembly-scanned in [Application/DependencyInjection.cs L16-L23](../src/Application/DependencyInjection.cs#L16-L23).

The read/write asymmetry is real, not cosmetic: [GetTodos](../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47) uses `AsNoTracking().ProjectTo<TodoListDto>()`, while [UpdateTodoItem](../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L23-L34) loads a tracked entity and mutates it. Same layer, deliberately different mechanics.

## Alternatives Considered

- **Application service classes** (`TodoService.CreateItem(...)`, `.UpdateItem(...)`). Familiar, fewer types, but methods pile up, dependencies become the union of every method's needs, and there's no natural seam for cross-cutting concerns. Rejected — the seam is the whole point (see [ADR-003](adr-003-behaviour-pipeline.md)).
- **Full CQRS with separate read/write data stores and event sourcing.** Massive overkill for this domain; the "lite" version keeps one database and one model. Rejected as premature.

## Consequences

**Positive**
- Each operation is small, independently testable ([TestApp.SendAsync](../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L16-L23) enters here), and adds no coupling to its neighbors.
- The message object *is* the contract — it's what the endpoint binds HTTP JSON into and what the handler consumes.
- A uniform `IRequest` shape is what makes the behaviour pipeline in [ADR-003](adr-003-behaviour-pipeline.md) possible at all — one wrapper can decorate every use case.

**Negative**
- Class proliferation: a trivial operation still needs a command, a handler, usually a validator, an endpoint method, and a test. There's even a scaffolding template for it ([templates/ca-use-case](../templates/ca-use-case/FeatureName/Commands/CleanArchitectureUseCase/CleanArchitectureUseCase.cs)) precisely because the boilerplate is real.
- Indirection: pressing "go to definition" on `sender.Send(command)` tells you nothing; you navigate by request *type*, not by call graph. New engineers find this disorienting.
- Some handlers are thin DbContext pass-throughs where the ceremony buys little.

## Evidence

Folder shape under [src/Application/TodoLists](../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs); command-as-record contracts like [CreateTodoItemCommand](../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs#L6-L11) (`IRequest<int>`) vs [DeleteTodoItemCommand](../src/Application/TodoItems/Commands/DeleteTodoItem/DeleteTodoItem.cs#L5) (`IRequest`); the return-type-in-the-contract distinction mirrored by the endpoint's HTTP status (201-with-body vs 204).
