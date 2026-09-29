# Fake-Code Contrasts

Nine bad-vs-better pairs. **All snippets below are illustrative fake code — none are from this repo** — but every pair maps to a real pattern here, anchored after each contrast. Read the bad version first and articulate the smell *before* reading on.

---

## Contrast 1: Business logic in the endpoint

```csharp
// Illustrative fake code: not from this repo (BAD)
groupBuilder.MapPost("", async (IApplicationDbContext db, CreateTodoListDto dto) =>
{
    if (string.IsNullOrEmpty(dto.Title) || dto.Title.Length > 200)
        return Results.BadRequest("bad title");
    if (await db.TodoLists.AnyAsync(l => l.Title == dto.Title))
        return Results.BadRequest("duplicate");
    var list = new TodoList { Title = dto.Title };
    db.TodoLists.Add(list);
    await db.SaveChangesAsync(default);
    return Results.Created($"/api/TodoLists/{list.Id}", list.Id);
});
```

Smells: validation, uniqueness, persistence, and HTTP all in one lambda; untestable without HTTP; error shape ad-hoc strings; second endpoint will copy-paste all of it.

Better shape: thin endpoint → command → pipeline handles validation/errors uniformly — exactly [TodoLists.CreateTodoList](../../src/Web/Endpoints/TodoLists.cs#L30-L37) + [validator](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs). The endpoint's *only* jobs: bind, send, translate to HTTP.

## Contrast 2: Coupling the UI to the DB shape

```jsx
// Illustrative fake code: not from this repo (BAD)
const res = await fetch('/api/TodoLists/raw'); // returns EF entities serialized
const lists = await res.json();
// UI now renders lists[0].items[0].list.items[0]... (cycles!), shows lastModifiedBy GUIDs,
// and breaks the day a DB column is renamed.
```

Smells: entities as wire format → serialization cycles (TodoItem.List ↔ TodoList.Items), overexposure (audit columns, future soft-delete flags), schema renames become UI breakages.

Better shape: purpose-built DTOs projected in the query — [TodoListDto/TodoItemDto](../../src/Application/TodoLists/Queries/GetTodos/TodoListDto.cs) via [ProjectTo](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47). The DTO is a *contract*; the entity is an implementation detail.

## Contrast 3: The missing permission filter (IDOR)

```csharp
// Illustrative fake code: not from this repo (BAD — and note how normal it looks)
public async Task<TodoListDto> Handle(GetTodoListQuery request, CancellationToken ct)
{
    var list = await _context.TodoLists.FindAsync([request.Id], ct);
    Guard.Against.NotFound(request.Id, list);
    return _mapper.Map<TodoListDto>(list);
}
```

```csharp
// Illustrative fake code: not from this repo (BETTER)
var list = await _context.TodoLists
    .Where(l => l.Id == request.Id && l.OwnerId == _user.Id)   // ownership in the query
    .SingleOrDefaultAsync(ct);
Guard.Against.NotFound(request.Id, list);                      // absent == forbidden == 404
```

The bad version is *this repo's actual idiom* ([UpdateTodoItem L25-L28](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L25-L28)) — correct for a single-user template, an IDOR the moment users matter. Note the better version returns 404 (not 403) for foreign resources: don't confirm existence. Structural upgrade: EF global query filter so no handler can forget.

## Contrast 4: N+1 queries

```csharp
// Illustrative fake code: not from this repo (BAD)
var lists = await _context.TodoLists.ToListAsync(ct);          // query 1
var dtos = new List<TodoListDto>();
foreach (var list in lists)
{
    var items = await _context.TodoItems                       // query 2..N+1
        .Where(i => i.ListId == list.Id).ToListAsync(ct);
    dtos.Add(new TodoListDto { Title = list.Title, Items = Map(items) });
}
```

Ten lists = eleven round trips; grows with data, invisible at dev scale. Better: one translated query with the child collection projected inside — the real [GetTodos handler](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47). Detection habit: any `await` inside a `foreach` over query results is a suspect ([05-quality/04-performance-thinking](../05-quality-engineering/04-performance-thinking.md)).

## Contrast 5: Optimistic update without a safety net

```jsx
// Illustrative fake code: not from this repo (BAD — simplified version of a real pattern here)
const toggle = async (item, done) => {
  setItems(prev => prev.map(i => i.id === item.id ? { ...i, done } : i));
  await client.updateTodoItem(item.id, { ...item, done });  // unhandled rejection
};
```

```jsx
// Illustrative fake code: not from this repo (BETTER)
const toggle = async (item, done) => {
  const snapshot = items;
  setItems(prev => prev.map(i => i.id === item.id ? { ...i, done } : i));
  try { await client.updateTodoItem(item.id, { ...item, done }); }
  catch { setItems(snapshot); notifyError('Couldn\'t save — try again.'); }
};
```

The real [updateCheckbox](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167) is between the two: it catches, but only `console.error`s — UI and server silently diverge. Optimistic UI is a *transaction with the user*; rollback is its compensation step.

## Contrast 6: Swallowed exceptions

```csharp
// Illustrative fake code: not from this repo (BAD)
try { await _mediator.Publish(domainEvent); }
catch { /* don't let events break saves */ }
```

Tempting — it "fixes" the coupling described in [Flow 4](../01-codebase-cartography/05-key-flows.md). But now events fail *silently*: no log, no retry, no metric; you've traded a visible failure for an invisible one. The repo rightly lets it throw and logs at the boundary ([UnhandledExceptionBehaviour](../../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L15-L28)). If you need decoupling, you need the outbox — not a `catch {}`. Rule: never catch without (a) handling meaningfully, (b) logging with context, or (c) rethrowing.

## Contrast 7: Stale-closure state updates

```jsx
// Illustrative fake code: not from this repo (BAD)
const addItem = async (title) => {
  const id = await client.createTodoItem({ title, listId: selectedListId });
  setLists(lists.map(l =>            // `lists` captured at render time!
    l.id === selectedListId ? { ...l, items: [...l.items, { id, title }] } : l));
};
```

If anything else updated `lists` between render and the awaited response (another create resolving, a checkbox toggle), this write clobbers it — the closure holds the stale array. Better: functional updates, `setLists(ls => ...)`, which the real code uses consistently (e.g. [commitNewItem L218-L221](../../src/Web/ClientApp-React/src/components/Todo.jsx#L211-L221)). Spot-check habit: any `setX(f(x))` where `x` came from the closure across an `await` is a bug waiting for concurrency.

## Contrast 8: Any-typed seams around a typed client

```typescript
// Illustrative fake code: not from this repo (BAD — but compare the Angular client)
let editor: any = {};
client.updateTodoList(id, editor as UpdateTodoListCommand).subscribe(...);
```

The cast asserts, the compiler believes, the server receives whatever the dialog happened to bind. Better: build the command explicitly — `new UpdateTodoListCommand({ id, title, colour })` — so missing/renamed fields fail the *build*. The real Angular client does the bad version ([todo.component.ts L29, L119](../../src/Web/ClientApp/src/app/todo/todo.component.ts#L29)); the generated classes it ignores would give full checking. Codegen only pays when you refuse `any` at the seams.

## Contrast 9: Casual public-contract change

```csharp
// Illustrative fake code: not from this repo (BAD — a one-line "rename for clarity" PR)
-    public static async Task<Ok<TodosVm>> GetTodoLists(ISender sender)
+    public static async Task<Ok<TodosVm>> FetchAllTodoLists(ISender sender)
```

Looks like a refactor; is a breaking change: the method name is the OpenAPI operationId ([EndpointRouteBuilderExtensions L20-L26](../../src/Web/Infrastructure/EndpointRouteBuilderExtensions.cs#L20-L26)), so both generated clients' method names change (`getTodoLists()` → `fetchAllTodoLists()`), breaking every call site — e.g. [Todo.jsx L34](../../src/Web/ClientApp-React/src/components/Todo.jsx#L33-L42) — *and* any external OpenAPI consumer. Better: don't rename; if you must, keep the operationId stable via explicit `.WithName("GetTodoLists")`. The habit: before renaming anything public-ish, ask "what generates from this? what parses this? what logs mention this?"

---

## Drill

Write contrast #10 yourself: "the handler that does two `SaveChangesAsync` calls" — construct the bad version, name the failure (partial commit between the calls breaks the one-use-case-one-transaction invariant from [03-architecture/02](../03-architecture-and-patterns/02-data-model-and-persistence.md)), write the better shape. If you can *generate* contrasts, you've internalized the pattern, not memorized the example.
