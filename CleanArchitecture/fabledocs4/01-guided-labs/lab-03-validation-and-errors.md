# Lab 3 — Validation and the Error Contract

**Goal:** add a validation rule, then follow it all the way to the HTTP 400 the client sees. You'll learn how input validation and the API's error shape are wired together — a connection most juniors never see clearly.

**Time:** 45–60 minutes. **Prereqs:** Labs 1–2.

## The concept in one paragraph

**Validation** here isn't scattered `if` checks in handlers. Each command can have a **validator** class (using a library called FluentValidation). A single **pipeline behaviour** runs every validator before the handler, collects *all* the failures, and throws one `ValidationException`. A single **exception handler** turns that into an HTTP `400` with a structured body. So: validator → behaviour → exception → 400. One rule, one consistent error shape, every time. Let's watch it happen.

## Step 0 — Branch

```bash
git switch -c lab/03-validation
```

## Step 1 — Read the existing machinery (don't write yet)

Open these three files in order and read them — this is the whole pipeline:

1. [CreateTodoItemCommandValidator.cs](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItemCommandValidator.cs) — a validator. Notice `RuleFor(v => v.Title).MaximumLength(200).NotEmpty()`. That's the whole rule set for creating an item. It's declarative — you *describe* the rules, the library enforces them.
2. [ValidationBehaviour.cs L15-L33](../../src/Application/Common/Behaviours/ValidationBehaviour.cs#L15-L33) — the behaviour that runs *all* validators for a request, gathers every failure, and throws a `ValidationException` if there are any. Note it runs them and aggregates — so a client gets *all* errors at once, not one at a time.
3. [ProblemDetailsExceptionHandler.cs L17-L23](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L23) — the `ValidationException` arm that maps to a `400` with the field-errors dictionary.

**Pause and predict:** where does the validator get *registered* so the behaviour can find it? (Answer: [Application/DependencyInjection.cs](../../src/Application/DependencyInjection.cs#L14) calls `AddValidatorsFromAssembly`, which scans and registers every validator automatically. You never wire one up by hand.)

## Step 2 — Add a rule

Give the `Note` field on the detail command a maximum length. First check: is there a validator for `UpdateTodoItemDetailCommand`? Look in its folder ([src/Application/TodoItems/Commands/UpdateTodoItemDetail/](../../src/Application/TodoItems/Commands/UpdateTodoItemDetail/UpdateTodoItemDetail.cs)). If there isn't one, create it, following the shape of the create-item validator:

```csharp
// Illustrative — new file UpdateTodoItemDetailCommandValidator.cs in the same folder
namespace CleanArchitecture.Application.TodoItems.Commands.UpdateTodoItemDetail;

public class UpdateTodoItemDetailCommandValidator : AbstractValidator<UpdateTodoItemDetailCommand>
{
    public UpdateTodoItemDetailCommandValidator()
    {
        RuleFor(v => v.Note).MaximumLength(4000);
    }
}
```

That's it. Because of the assembly scan from step 1, this validator is now live — no registration needed. This is **why closing an unbounded-input gap here is a genuine good-first contribution**: a `Note` with no length limit is a small denial-of-service and bloat vector.

## Step 3 — Watch it become a 400 (a test)

Write a functional test that sends an over-long note and expects a `ValidationException`:

```csharp
// Illustrative — a functional test
[Test]
public async Task ShouldRejectOverlongNote()
{
    await TestApp.RunAsDefaultUserAsync();
    var listId = await TestApp.SendAsync(new CreateTodoListCommand { Title = "L" });
    var itemId = await TestApp.SendAsync(new CreateTodoItemCommand { ListId = listId, Title = "T" });

    var command = new UpdateTodoItemDetailCommand
    {
        Id = itemId, ListId = listId, Note = new string('x', 4001)
    };

    var ex = await Should.ThrowAsync<ValidationException>(() => TestApp.SendAsync(command));
    ex.Errors.ShouldContainKey("Note");   // assert the RIGHT rule fired, not just "an" error
}
```

Notice the last line. Asserting `ThrowAsync<ValidationException>` alone would pass even if the *wrong* rule failed. Checking `Errors` contains the `Note` key proves *your* rule is what rejected it. Weak tests pass for the wrong reasons; this one can't.

```bash
dotnet test ./tests/Application.FunctionalTests --filter "Name~ShouldRejectOverlongNote"
```

## Step 4 — See the actual HTTP response

The test enters at the command layer, below HTTP. To see the *actual 400 body* a browser would get, run the app ([tooling §1](../04-tooling-and-debugging.md)) and hit the endpoint with an over-long note using the Scalar API reference at `/scalar`, or the [Web-webapi.http](../../src/Web/Web-webapi.http) file. You'll get a JSON body shaped like:

```json
{ "errors": { "Note": ["The length of 'Note' must be 4000 characters or fewer. ..."] } }
```

That exact shape is what the React client parses when it shows an inline error ([Todo.jsx L77-L83](../../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83) does this for the create dialog). The whole chain — validator → behaviour → exception → ProblemDetails → client parse — just executed in front of you.

## Step 5 — The senior observation

You just added *one* rule and it produced a consistent, documented 400 with zero endpoint code. Now notice a **gap** the same machinery reveals: some errors *don't* go through validation. A domain-invariant violation like an unsupported colour throws an exception that **isn't** mapped, so it returns a 500 instead of a 400 ([ProblemDetailsExceptionHandler L43](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L43) is the unmapped fall-through). That's a real bug — and it's exactly what you'll fix in [Lab 5](lab-05-fix-a-real-bug.md). Seeing the *contrast* between "validation → clean 400" and "unmapped exception → ugly 500" is the point of this lab.

## Step 6 — Commit

```bash
git add -A
git commit -m "feat: bound Note length with a validation rule"
```

## What you learned

- Validation is **declarative and centralized**: describe rules in a validator, and a behaviour + exception handler turn a broken rule into a consistent HTTP 400. No endpoint code.
- Validators are auto-discovered — you never register one.
- A good test asserts *which* rule fired, not just that *something* failed.
- The error contract has a seam: validation errors are clean 400s; unmapped exceptions leak as 500s.

## Common mistakes

- Writing the validator but expecting it to run without the `AddValidatorsFromAssembly` scan — it's automatic here, but on other projects you might have to register it, so know that the scan is doing the wiring.
- A test that only checks the exception type and not the error key — green for the wrong reason.

## Next

[Lab 4 — Your first tests, properly](lab-04-your-first-tests.md).
