# Syntax Decoder

Modern C# and TypeScript are dense with symbols a tutorial never explains. This is a lookup table for the ones that show up in *this* codebase, each with a real anchor so you can see it in context. Keep it open while you read code.

## C# — keywords and modifiers

| Symbol / keyword | What it means | Seen here |
|---|---|---|
| `record` | A reference type with value-equality and concise syntax; ideal for immutable data/messages | [CreateTodoItemCommand](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItem.cs#L6-L11) |
| `{ get; init; }` | A property you can set *only* during object construction, then it's read-only | [UpdateTodoItemCommand](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L5-L12) |
| `{ get; set; }` | A normal read/write property | [TodoItem](../../src/Domain/Entities/TodoItem.cs#L5) |
| positional record `record X(int Id)` | Record with a constructor parameter that becomes a property | [DeleteTodoItemCommand](../../src/Application/TodoItems/Commands/DeleteTodoItem/DeleteTodoItem.cs#L5) |
| primary constructor `class C(string x)` | Constructor parameters declared on the class line itself | [Colour(string code)](../../src/Domain/ValueObjects/Colour.cs#L3) |
| `?` after a type (`string?`) | Nullable — this value is *allowed* to be null; the compiler enforces null-checks | [TodoItem.Title](../../src/Domain/Entities/TodoItem.cs#L7) |
| `null!` | "Trust me, this isn't null" — overrides the nullable checker (a place the dev overruled the compiler; each one is a small risk) | [TodoItem.List](../../src/Domain/Entities/TodoItem.cs#L28) |
| `??` | Null-coalescing: `a ?? b` = "a, unless it's null, then b" | [CreateTodoList handler](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs#L28) (`request.Colour ?? Colour.Grey`) |
| `?.` | Null-conditional: `a?.B` = "a.B, but null if a is null" (no crash) | [CurrentUser](../../src/Web/Services/CurrentUser.cs#L16) |
| `=>` (expression body) | Shorthand for a method/property that's a single expression | [CurrentUser.Id](../../src/Web/Services/CurrentUser.cs#L16) |
| `=> ` (lambda) | An inline anonymous function: `x => x.Id` | everywhere in LINQ, e.g. [GetTodos](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L46) |
| `async` / `await` / `Task` | Asynchronous code — see [the async primer](02-fundamentals-primers/02-async-await.md) | every handler |
| `static abstract` (interface) | An interface member implemented as a *static* on the type, not an instance (C# 11+) | [IEndpointGroup](../../src/Web/Infrastructure/IEndpointGroup.cs#L16-L18) |
| `[]` (collection expression) | Concise collection literal: `Items = []` = empty list; `[..roles]` spreads | [TodosVm](../../src/Application/TodoLists/Queries/GetTodos/TodosVm.cs#L7-L11), [TestApp](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L73) |
| `nameof(X)` | The *name* of X as a string, but refactor-safe (renames update it) | [WebApplicationExtensions](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L21-L22) |
| `[Attribute]` | Metadata attached to a type/member, read by frameworks or reflection | `[Authorize]` on [GetTodosQuery](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L9) |
| `<T>` generics | Type parameter — code that works for many types | `IRequest<int>`, the pipeline behaviours |
| `where T : notnull` | A constraint on a generic type parameter | [ValidationBehaviour](../../src/Application/Common/Behaviours/ValidationBehaviour.cs#L5-L6) |
| switch expression `x switch { ... }` | Concise multi-branch returning a value, often with type patterns | [ProblemDetailsExceptionHandler](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44) |
| `global using` | An import that applies to the whole project (declared once) | [Application/GlobalUsings.cs](../../src/Application/GlobalUsings.cs) |
| `namespace X;` (file-scoped) | Namespace for the whole file without braces | top of nearly every `.cs` file |

## C# — library idioms specific to this repo

| Idiom | What it does | Seen here |
|---|---|---|
| `IRequest` / `IRequest<T>` | A MediatR message; `<T>` is the return type | commands/queries |
| `ISender.Send(command)` | Dispatch a message to its handler | [endpoints](../../src/Web/Endpoints/TodoItems.cs#L25) |
| `Guard.Against.NotFound(id, entity)` | Throw `NotFoundException` if null — a fail-fast one-liner | [UpdateTodoItem](../../src/Application/TodoItems/Commands/UpdateTodoItem/UpdateTodoItem.cs#L28) |
| `RuleFor(x => x.Prop)...` | FluentValidation rule declaration | [validators](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItemCommandValidator.cs) |
| `.AsNoTracking()` | EF: read-only query, skip change tracking (faster reads) | [GetTodos](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L44) |
| `.ProjectTo<Dto>(...)` | AutoMapper: shape the query straight into a DTO, in SQL | [GetTodos](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L45) |
| `.ToListAsync(ct)` | EF: execute the query and materialize results (this is when SQL runs) | [GetTodos](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L47) |
| `TypedResults.Ok(x)` / `.Created(...)` / `.NoContent()` | Strongly-typed HTTP results | [endpoints](../../src/Web/Endpoints/TodoLists.cs) |

## TypeScript / React (the frontend)

| Symbol | What it means | Seen here |
|---|---|---|
| `useState(initial)` | React hook: a piece of component state + its setter | [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx#L9) |
| `useEffect(fn, [deps])` | React hook: run `fn` after render; re-run when `deps` change | [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx#L33-L47) |
| `useRef(v)` | A mutable value that persists across renders *without* causing re-renders | [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx#L24-L31) |
| `setX(prev => ...)` | Functional state update — computes new state from the previous (avoids stale-closure bugs) | [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx#L163) |
| `?.` and `??` | Same as C# — optional chaining and nullish-coalescing (TS borrowed them) | [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx#L39) |
| `{ ...obj, x: 1 }` (spread) | Copy `obj` into a new object, overriding `x` — immutable update | [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx#L162) |
| `async` / `await` | Same idea as C#; used for `fetch`-based API calls | [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx#L69-L76) |
| `x as T` | Type assertion: "treat x as T" — *asserts*, doesn't verify (a TS escape hatch) | [Angular todo.component.ts](../../src/Web/ClientApp/src/app/todo/todo.component.ts#L119) |
| `signal(x)` / `computed(...)` (Angular) | Angular's reactive state primitives (its answer to `useState`) | [todo.component.ts](../../src/Web/ClientApp/src/app/todo/todo.component.ts#L20-L33) |

## How to use this

Don't memorize it — *look things up as you hit them.* After a week the common ones (`record`, `init`, `=>`, `??`, `async`) will be automatic. The rarer ones (`static abstract`, `null!`) you'll look up two or three times and then they'll stick. That's the normal path; nobody memorizes syntax from a table, they absorb it from repeated real encounters. For the deeper "why these exist," see [fabledocs/02-stack-and-language-mastery/03-type-system-and-contracts](../../fabledocs/02-stack-and-language-mastery/03-type-system-and-contracts.md).
