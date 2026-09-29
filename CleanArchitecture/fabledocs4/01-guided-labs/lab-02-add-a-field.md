# Lab 2 — Add a Field End to End: `DueDate`

**Goal:** add an optional `DueDate` to todo items, all the way from the database to the API response. This is a **vertical slice** — a change that cuts through every layer — and it's the single most important shape of work a fullstack engineer does.

**Time:** 1.5–2 hours. **Prereqs:** Lab 1.

## The mental model: the "touch list"

Before writing any code, a mid-level engineer writes down every layer a change will touch, in order. For adding a field to an entity in *this* codebase, the touch list is:

1. **Entity** (Domain) — the field itself.
2. **EF configuration** (Infrastructure) — column constraints, if any.
3. **Command(s)** (Application) — so the field can be *set*.
4. **DTO + mapping** (Application) — so the field can be *read* back.
5. **Validator** (Application) — if the field has rules.
6. **Tests** — proving set and read work.
7. **The client** — regenerate the typed API client; touch the UI if it should show the field.

You'll do 1–6 in this lab and understand 7. Miss a step and either the compiler catches you (good) or a test does (also good) — that's the safety net this architecture buys you. Write the touch list on paper now; you'll check items off.

## Step 0 — Branch

```bash
git switch -c lab/02-due-date
```

## Step 1 — The entity

Open [src/Domain/Entities/TodoItem.cs](../../src/Domain/Entities/TodoItem.cs). Add the field. **Make it nullable** (`DateTimeOffset?`) — most items won't have a due date, and an existing item created before this field existed must be representable without one:

```csharp
// Illustrative — add near the other properties
public DateTimeOffset? DueDate { get; set; }
```

Why `DateTimeOffset` and not `DateTime`? Because the audit fields on the base class already use it ([BaseAuditableEntity.cs](../../src/Domain/Common/BaseAuditableEntity.cs)) — matching existing conventions is a real engineering value. Why nullable? Predict what a *non*-nullable `DateTime` would default existing rows to. (Answer: `0001-01-01`, a bogus date the UI would render as real — a classic bug. Nullable makes "no due date" honestly representable.)

## Step 2 — EF configuration (decide if you need it)

Open [src/Infrastructure/Data/Configurations/TodoItemConfiguration.cs](../../src/Infrastructure/Data/Configurations/TodoItemConfiguration.cs). The `Title` gets a max length and `IsRequired`. Your `DueDate` needs no special config — it's a nullable date, and EF maps it by convention. So you *consciously* touch-list this step and decide "nothing needed." (Knowing when a step is a no-op is as important as doing it.)

**Important reality:** this repo has no migrations — in development the schema is recreated from the model on every run ([initialiser L44-L46](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L44-L46)). So your new column appears automatically next run. In a *production* app with real data this step would require a migration; that difference is a whole primer's worth of nuance (see [transactions primer](../02-fundamentals-primers/04-what-a-transaction-is.md) and [fabledocs3 ADR-010](../../fabledocs3/adr-010-schema-by-convention-no-migrations.md)). For this lab, "recreated on run" means you get the column for free.

## Step 3 — Let it be set (the command)

To set a due date you extend a command. Use the detail command, which already carries item metadata — [src/Application/TodoItems/Commands/UpdateTodoItemDetail/UpdateTodoItemDetail.cs](../../src/Application/TodoItems/Commands/UpdateTodoItemDetail/UpdateTodoItemDetail.cs). Add `DueDate` to the command record *and* assign it in the handler:

```csharp
// Illustrative — in the command record
public DateTimeOffset? DueDate { get; init; }

// Illustrative — in the handler, alongside the other assignments
entity.DueDate = request.DueDate;
```

Read the handler as you edit it: it loads the entity by id, guards for not-found, mutates, saves. You're adding one line to the "mutates" section. Note the command uses `init` (settable only at construction) — if that's unfamiliar, [syntax decoder](../03-syntax-decoder.md).

## Step 4 — Let it be read (the DTO + mapping)

Open [src/Application/TodoLists/Queries/GetTodos/TodoItemDto.cs](../../src/Application/TodoLists/Queries/GetTodos/TodoItemDto.cs). This is the shape the API returns. Add:

```csharp
// Illustrative — in the DTO
public DateTimeOffset? DueDate { get; init; }
```

The DTO has a nested `Mapping` class using AutoMapper. For a same-named property, AutoMapper maps it automatically — you don't have to write mapping code. But there's a test that will *verify* your DTO didn't drift from the entity: [MappingTests.ShouldHaveValidConfiguration](../../tests/Application.UnitTests/Common/Mappings/MappingTests.cs#L30-L34). Keep it in mind for step 6.

## Step 5 — Validation (decide, again)

Does `DueDate` need a rule? Maybe "not in the past." That's a *product* decision, not a technical one — and a good instinct is to *not* invent rules the requirements didn't ask for. For this lab, skip validation and note on your touch list "no rule — dates in the past are allowed by design." If you *wanted* one, you'd add it in a validator; you'll do exactly that in [Lab 3](lab-03-validation-and-errors.md).

## Step 6 — Prove it works (tests)

Run the existing suites first to make sure you didn't break anything:

```bash
dotnet build
dotnet test ./tests/Application.UnitTests
```

The mapping test ([MappingTests](../../tests/Application.UnitTests/Common/Mappings/MappingTests.cs#L30-L34)) is your friend here: if you'd added `DueDate` to the entity but forgotten the DTO, or misnamed it, `AssertConfigurationIsValid()` fails and tells you. That's the architecture catching your mistake at test time instead of in production.

Now write a functional test that sets a due date and reads it back. Crib the shape from [tests/Application.FunctionalTests/TodoItems/Commands/UpdateTodoItemDetailTests.cs](../../tests/Application.FunctionalTests/TodoItems/Commands/UpdateTodoItemDetailTests.cs) (open it — it shows the setup-through-commands pattern):

```csharp
// Illustrative — a new functional test
[Test]
public async Task ShouldPersistDueDate()
{
    await TestApp.RunAsDefaultUserAsync();
    var listId = await TestApp.SendAsync(new CreateTodoListCommand { Title = "L" });
    var itemId = await TestApp.SendAsync(new CreateTodoItemCommand { ListId = listId, Title = "Task" });
    var due = new DateTimeOffset(2030, 1, 1, 0, 0, 0, TimeSpan.Zero);

    await TestApp.SendAsync(new UpdateTodoItemDetailCommand { Id = itemId, ListId = listId, DueDate = due });

    var item = await TestApp.FindAsync<TodoItem>(itemId);
    item!.DueDate.ShouldBe(due);
}
```

Run it:

```bash
dotnet test ./tests/Application.FunctionalTests --filter "Name~ShouldPersistDueDate"
```

This test uses a **real database** (see [Lab 4](lab-04-your-first-tests.md) and the [testing primer thread](../02-fundamentals-primers/README.md)). It goes through the real command pipeline, real validation, real EF Core. If it's green, your field genuinely round-trips through the database.

## Step 7 — Understand the client step (don't skip the understanding)

You won't fully do this without running the frontend build, but you must *understand* it: both SPAs generate their typed API client from the server's OpenAPI document ([contract chain](../../fabledocs3/adr-006-contract-first-client-generation.md)). Running `npm run generate-api` in a client folder would regenerate `web-api-client.ts` and your `DueDate` would appear as an optional field on the TypeScript `TodoItemDto`. Only *then* could a UI component display it. This is why "add a field" is a fullstack task even when the backend part feels done — the contract has to flow to the client.

## Step 8 — Commit

```bash
git add -A
git commit -m "feat: add optional DueDate to todo items"
```

## What you learned

- A vertical slice is a **disciplined sequence**, not a scramble. The touch list is how mid-level engineers avoid the "I forgot the DTO" class of bug.
- The type system and one clever test (`MappingTests`) form a safety net — miss a layer and something red tells you, immediately.
- "Add a field" is fullstack because the *contract* has to travel from C# through OpenAPI to TypeScript.

## Common mistakes

- Non-nullable date field (defaults existing rows to `0001-01-01`).
- Adding it to the entity but not the DTO — read-side silently missing; the mapping test catches it if you run it.
- Inventing a validation rule the product didn't ask for. Restraint is a skill.

## Next

[Lab 3 — Validation and the error contract](lab-03-validation-and-errors.md).
