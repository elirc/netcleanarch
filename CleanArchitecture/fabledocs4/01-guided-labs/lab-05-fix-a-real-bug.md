# Lab 5 — Fix a Real Bug

**Goal:** find, understand, and fix a genuine bug in this codebase — an endpoint that returns the wrong HTTP status — using a repeatable debugging method. Then add a test that stops it coming back. Debugging is the skill that separates "can write code" from "can be trusted with production."

**Time:** 1–1.5 hours. **Prereqs:** Labs 1–4.

## The bug

Creating a todo list with an unsupported colour code returns **HTTP 500** (a server error) when it should return **400** (a client error — *you* sent bad input). A 500 says "the server is broken"; a 400 says "your request was bad." Returning the wrong one is a real defect: it pages the on-call team for something that's actually a client mistake, and it hides the real problem from the caller.

This is a real gap in the code, not a contrived one. Let's fix it properly.

## The method (memorize this)

1. **Reproduce** — make it fail on demand.
2. **Narrow** — cut the search space in half at each step. Is it the client, the endpoint, the pipeline, the handler, the database?
3. **Hypothesize** — one falsifiable sentence.
4. **Test cheaply** — a log line or a quick test beats redeploying.
5. **Fix the root cause** — not the symptom.
6. **Add regression coverage** — the test that would have caught it.

## Step 0 — Branch

```bash
git switch -c lab/05-colour-status
```

## Step 1 — Reproduce

Write a quick functional test that sends a bad colour and observe what happens:

```csharp
// Illustrative — a temporary reproduction test in the TodoLists command tests
[Test]
public async Task Repro_BadColour()
{
    await TestApp.RunAsDefaultUserAsync();
    await TestApp.SendAsync(new CreateTodoListCommand { Title = "L", Colour = "#123456" });
}
```

Run it. It throws `UnsupportedColourException` — not a `ValidationException`. That's your first real clue: **the failure isn't going through validation.** (Over HTTP this unmapped exception becomes a 500; at the command layer you see the raw exception.)

## Step 2 — Narrow

Ask: *is the bug in validation or after it?* Check the validator for the create command — [CreateTodoListCommandValidator.cs](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs). Read its rules. It validates `Title` (not empty, max length, unique). **It has no rule about `Colour` at all.** So a bad colour passes validation, reaches the handler, and the handler calls `Colour.From(...)` — [CreateTodoList.cs L27-L28](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs#L25-L29) — which throws.

Now: *why does that throw become a 500 and not a 400?* Open the exception handler, [ProblemDetailsExceptionHandler.cs L17-L44](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44). It maps `ValidationException`, `NotFoundException`, `UnauthorizedAccessException`, `ForbiddenAccessException` — and **everything else falls through** to `_ => (-1, null)` at [L43](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L43), which returns "not handled" → default 500. `UnsupportedColourException` is a domain-invariant violation that nobody mapped.

You've narrowed it to a precise root cause: **a bad colour isn't validated at the contract layer, so it surfaces as an unmapped domain exception → 500.**

## Step 3 — Hypothesize and choose a fix

There are two candidate fixes; a junior picks one, a senior can argue between them:

- **(A) Add a colour rule to the validator** so a bad colour is a normal validation failure → clean, aggregated 400.
- **(B) Map `UnsupportedColourException` → 400** in the exception handler.

Choose **(A).** Reasoning: it keeps the error contract *consistent* — the bad colour joins the same 400-with-field-errors shape everything else uses, so the client parses it the same way. Option B would introduce a *second* 400 path with a different body shape. (Option B also has merit — it catches the exception no matter where it's thrown, including future call sites — so the truly complete answer is "validator now, and consider mapping the exception too as defense in depth." For this lab, do A.)

## Step 4 — Fix the root cause

Add a colour rule to the validator. You want "if a colour was supplied, it must be one of the supported ones." The supported set lives in [Colour.SupportedColours](../../src/Domain/ValueObjects/Colour.cs#L48-L60):

```csharp
// Illustrative — in CreateTodoListCommandValidator's constructor
RuleFor(v => v.Colour)
    .Must(BeASupportedColour)
    .When(v => v.Colour is not null)
    .WithMessage("'{PropertyValue}' is not a supported colour.");

// Illustrative — a helper method on the validator
private static bool BeASupportedColour(string? code) =>
    Colour.SupportedColours.Any(c => c.Code == code);
```

Read what you wrote: `.When(...)` means the rule only runs if a colour was provided (a null colour is allowed and defaults to grey in the handler). `.Must(...)` is a custom predicate. This is the same declarative style from [Lab 3](lab-03-validation-and-errors.md).

## Step 5 — Verify the fix

Change your repro test into a real regression test — now you expect a *validation* failure, not the raw exception:

```csharp
// Illustrative — the repro, promoted to a real test
[Test]
public async Task ShouldRejectUnsupportedColour()
{
    await TestApp.RunAsDefaultUserAsync();
    var command = new CreateTodoListCommand { Title = "L", Colour = "#123456" };

    var ex = await Should.ThrowAsync<ValidationException>(() => TestApp.SendAsync(command));
    ex.Errors.ShouldContainKey("Colour");
}
```

```bash
dotnet test ./tests/Application.FunctionalTests --filter "Name~Colour"
```

Green means: a bad colour is now a `ValidationException`, which the HTTP layer maps to a clean 400 — bug fixed at the root, and a test now guards it forever.

## Step 6 — Confirm over HTTP (optional but satisfying)

Run the app, and via `/scalar` or [Web-webapi.http](../../src/Web/Web-webapi.http) POST a list with `"colour": "#123456"`. You now get a **400** with a `Colour` error, not a 500. You changed a server-fault into a proper client error.

## Step 7 — Commit

```bash
git add -A
git commit -m "fix: return 400 for unsupported colour instead of 500"
```

## What you learned

- The debugging **method** works: reproduce, narrow by halving, hypothesize, fix the *root cause*, add a regression test. You didn't guess — you cornered it.
- The root cause was a *missing* rule, not a broken line. Absence is a bug class; validators need review too.
- You could *argue between two fixes* and justify your choice by the error-contract-consistency principle. That reasoning is what makes a fix senior-quality, not just correct.

## Common mistakes

- "Fixing" it by wrapping the handler in a try/catch that swallows the exception — that hides the problem instead of solving it, and you lose the error entirely. Never catch without handling meaningfully, logging, or rethrowing.
- Mapping the exception to 400 *and* forgetting the validator — works, but leaves a second error-body shape. Consistency matters.
- Skipping the regression test. A fix with no test is a fix that comes back.

## You're through the labs

Pick a [capstone](README.md) and ship it on your own. Then you're ready to read the deeper material in [fabledocs](../../fabledocs/README.md) with real context, because you've actually *done* the things it talks about.
