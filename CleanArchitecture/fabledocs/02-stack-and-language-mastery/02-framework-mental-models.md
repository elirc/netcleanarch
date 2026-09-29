# Framework Mental Models — ASP.NET Core, MediatR, EF Core, React

## ASP.NET Core: two pipelines, not one

**Mental model:** an ASP.NET Core app is (1) a *service container* built before startup and (2) a *middleware pipeline* every request flows through, in the order written. Confusing the two is the classic junior error.

In [Program.cs](../../src/Web/Program.cs): lines [4-12](../../src/Web/Program.cs#L4-L12) build the container (order between `Add*` calls barely matters); lines [27-49](../../src/Web/Program.cs#L27-L49) build the pipeline (order is everything). Why must `UseExceptionHandler` ([L38](../../src/Web/Program.cs#L38)) be registered before `MapEndpoints` executes handlers? Because middleware wraps what comes *after* it — the exception handler can only catch exceptions from things nested inside it. Why is `UseCors` ([L28](../../src/Web/Program.cs#L28)) early? Preflight `OPTIONS` requests must be answered before anything else rejects them.

**Minimal APIs** here are class-per-resource endpoint groups, not controllers: [IEndpointGroup](../../src/Web/Infrastructure/IEndpointGroup.cs) + reflection registration ([WebApplicationExtensions](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28)). Handler parameters are model-bound by type: `ISender sender` from DI, `int id` from the route, `CreateTodoItemCommand command` from the JSON body ([TodoItems.cs L23](../../src/Web/Endpoints/TodoItems.cs#L23)). `TypedResults` + union return types like `Results<NoContent, BadRequest>` ([L32](../../src/Web/Endpoints/TodoItems.cs#L32)) make the response contract *compile-checked and OpenAPI-visible* — compare with returning bare `IResult`, where the docs and the code can drift.

Failure mode to know: model binding failures (malformed JSON, wrong types) produce a 400 *before* your code runs; `SuppressModelStateInvalidFilter` ([Web/DependencyInjection.cs L22-L23](../../src/Web/DependencyInjection.cs#L22-L23)) turns off MVC's automatic DataAnnotations 400 so FluentValidation is the single validation authority.

## MediatR: an in-process message bus

**Mental model:** `sender.Send(command)` is a function call with a routing table — one request type → one handler — plus an onion of behaviors around it. It buys: uniform cross-cutting concerns ([the five behaviours](../../src/Application/DependencyInjection.cs#L16-L23)), handlers testable without HTTP ([TestApp.SendAsync](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L16-L23)), and thin endpoints. It costs: indirection (F12 on `Send` tells you nothing — navigate by *type*, search the command class name), and the temptation to treat it as magic.

`Publish` (notifications) is the other half: zero-to-many handlers, used for domain events ([LogTodoItemCompleted](../../src/Application/TodoItems/EventHandlers/LogTodoItemCompleted.cs)). Send = "do this" (exactly one owner); Publish = "this happened" (anyone may care). That sentence is interview gold.

When *not* to use MediatR: single-handler CRUD apps with no cross-cutting pipeline needs — the ceremony buys nothing. Being able to argue both sides beats cargo-culting it.

## EF Core: a unit of work with a change tracker

**Mental model:** the `DbContext` is a **unit of work**. You load entities (tracked snapshots), mutate them in memory, and `SaveChangesAsync` diffs and flushes everything in one transaction. That's why handlers like [UpdateTodoItem](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L23-L34) never write UPDATE statements — the tracker computes them.

Consequences visible in this repo:
- `FindAsync` ([L25-L26](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L25-L26)) checks the tracker *before* querying — cheap for by-key loads, but returns tracked entities only.
- `AsNoTracking` ([GetTodos.cs L44](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L44)) opts a read-only query out of tracking — less memory, no accidental writes. Reads untracked, writes tracked: a house rule worth stealing.
- Interceptors hook the save seam once for auditing and events, instead of every handler remembering ([AuditableEntityInterceptor](../../src/Infrastructure/Data/Interceptors/AuditableEntityInterceptor.cs), [DispatchDomainEventsInterceptor](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs)).
- Schema by convention + fluent overrides: [TodoListConfiguration](../../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs) sets max length, required, and `OwnsOne(b => b.Colour)` — the value object becomes columns on the TodoLists table (owned entity), not a separate table with a FK.
- Deleting a list deletes its items via conventional cascade (relationship discovered from [TodoList.Items](../../src/Domain/Entities/TodoList.cs#L9) / [TodoItem.List](../../src/Domain/Entities/TodoItem.cs#L28)); [DeleteTodoListTests](../../tests/Application.FunctionalTests/TodoLists/Commands/DeleteTodoListTests.cs) relies on it.

## React (as this repo writes it): state is the source of truth, effects synchronize with the outside

**Mental model:** a component is a function of state → UI. Re-render happens when state changes; *effects* run after render to synchronize with external systems (network, DOM APIs, localStorage).

[Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx) is a compact specimen:
- **Server state in client state:** the whole dataset lives in `useState` ([L9-L22](../../src/Web/ClientApp-React/src/components/Todo.jsx#L9-L22)), loaded once by a mount effect ([L33-L42](../../src/Web/ClientApp-React/src/components/Todo.jsx#L33-L42)). Every mutation then hand-patches this cache with immutable updates (`setLists(ls => ls.map(...))`, e.g. [L139-L147](../../src/Web/ClientApp-React/src/components/Todo.jsx#L139-L147)). This works, and teaches immutability well — but recognize the pattern's cost: each new mutation needs bespoke cache surgery, and drift accumulates. Libraries like TanStack Query exist precisely to replace this hand-rolled cache (fetch + invalidate instead of patch).
- **Functional updates:** `setLists(ls => ...)` uses the callback form because the next state depends on the previous — avoiding stale-closure bugs. Compare `useEffect`'s dependency array `[selectedListId]` ([L44-L47](../../src/Web/ClientApp-React/src/components/Todo.jsx#L44-L47)) resetting the add-item editor when the selection changes.
- **Refs for non-render state:** `editCancelledRef` ([L26](../../src/Web/ClientApp-React/src/components/Todo.jsx#L26)) coordinates the blur-vs-Escape race without triggering re-renders; dialog refs drive the native `<dialog>` API ([L28-L31](../../src/Web/ClientApp-React/src/components/Todo.jsx#L28-L31)).
- **Derived state, not stored state:** `selectedList` is computed on render ([L49](../../src/Web/ClientApp-React/src/components/Todo.jsx#L49)), never stored — the "don't store what you can derive" rule.
- **Reference-equality trap:** `editingItem === item` ([L266](../../src/Web/ClientApp-React/src/components/Todo.jsx#L266)) compares object identity. It works because the same object instances stay in state — but would silently break the edit UI the moment anything re-fetches lists (new object identities). Fragile invariant; compare by `id` to make it robust.
- **Context for app-wide state:** [AuthContext](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx) and [ThemeContext](../../src/Web/ClientApp-React/src/components/ThemeContext.jsx) — small, stable values; correct use of context (not high-churn data).

The Angular twin ([todo.component.ts](../../src/Web/ClientApp/src/app/todo/todo.component.ts#L20-L33)) solves identical problems with **signals** (`lists = signal(...)`, `selectedList = computed(...)`) and RxJS observables for HTTP. Reading both implementations of one feature is the fastest framework-transfer exercise this repo offers.

## Drills

1. Reorder experiment (thought-only): what breaks if `app.UseExceptionHandler` moved to the last line of [Program.cs](../../src/Web/Program.cs)? (Exceptions from endpoints would escape it — wrapping order.)
2. In [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx), find every place the client cache is patched after a mutation (there are six). For each: what does the server know that the patch ignores? (E.g. `commitNewItem` [L217-L221](../../src/Web/ClientApp-React/src/components/Todo.jsx#L211-L221) invents `priority: priorityLevels[0]?.id` — but the server defaulted `Done=false, Priority=None`; if `None` isn't first in the lookup, cache ≠ truth.)
3. Explain to a rubber duck why [UpdateTodoItem handler](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs) contains no `UPDATE` statement and no explicit transaction, yet is transactional.

## Interview angle

1. *"Walk me through your framework's request pipeline."* → [08-interview-prep/01, Q2](../08-interview-prep/01-csharp-dotnet-deep-dive.md)
2. *"What is MediatR buying you? When would you drop it?"* → [08-interview-prep/03, Q10](../08-interview-prep/03-api-and-data-modeling-questions.md)
3. *"How does EF Core know what SQL to run on save?"* → [08-interview-prep/03, Q7](../08-interview-prep/03-api-and-data-modeling-questions.md)
4. *"When do you reach for React context vs component state vs a server-cache library?"* → [08-interview-prep/02, Q4](../08-interview-prep/02-frontend-framework-questions.md)
5. *"What's a stale closure? Show one you avoided."* → the functional-update sites above; [08-interview-prep/02, Q3](../08-interview-prep/02-frontend-framework-questions.md)
