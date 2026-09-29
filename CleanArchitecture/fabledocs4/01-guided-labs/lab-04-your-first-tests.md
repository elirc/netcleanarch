# Lab 4 — Your First Tests, Properly

**Goal:** understand the difference between the test *types* in this repo, then write one unit test and one functional test that pass — and understand what each one can and can't prove. Testing is where junior engineers earn trust fastest, because good tests are how the team knows your code works without re-reading it.

**Time:** 1 hour. **Prereqs:** Labs 1–3 (you've already written a couple of tests; now you'll understand them).

## The four test types, and the one rule

This repo has four test projects. You mostly care about two.

| Type | Project | Runs against | Use it for |
|---|---|---|---|
| **Unit** | `Domain.UnitTests`, `Application.UnitTests` | plain objects, no database | pure logic and rules (like `Colour`) |
| **Functional** | `Application.FunctionalTests` | a **real database** | whole use cases: validation + pipeline + real SQL |
| Acceptance | `Web.AcceptanceTests` | a real browser (Playwright) | end-to-end flows; you'll rarely write these early |
| (Integration placeholder) | `Infrastructure.IntegrationTests` | — | effectively empty; ignore it |

**The one rule for choosing:** *test at the lowest level that can actually observe the failure you're worried about.* Worried "does `Colour.From` reject junk?" → unit test (no database can fail that). Worried "does my query actually work against the database?" → functional test (a unit test literally cannot answer that). Getting this wrong is the most common junior testing mistake — writing a slow functional test for pure logic, or a fast unit test for something only the database can prove.

## Step 0 — Branch

```bash
git switch -c lab/04-tests
```

## Step 1 — A unit test (fast, no database)

Open [tests/Domain.UnitTests/ValueObjects/ColourTests.cs](../../tests/Domain.UnitTests/ValueObjects/ColourTests.cs) and read all of it — it's short and it's a great model. Notice the structure of each test: **Arrange** (set up), **Act** (do the thing), **Assert** (check the result), using Shouldly's `.ShouldBe(...)`.

Write one that proves an unsupported colour is rejected:

```csharp
// Illustrative — add to ColourTests.cs
[Test]
public void ShouldThrowGivenUnsupportedCode()
{
    Should.Throw<UnsupportedColourException>(() => Colour.From("#000001"));
}
```

Run it:

```bash
dotnet test ./tests/Domain.UnitTests
```

Milliseconds. No database, no web server. That speed is *why* pure logic belongs at this layer — you can run hundreds of these constantly.

## Step 2 — A functional test (real database, real pipeline)

Now something a unit test *can't* prove: that creating an item actually persists, with audit fields stamped by the real infrastructure. Open [tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs](../../tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs) and read `ShouldCreateTodoItem` ([L18-L45](../../tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs#L18-L45)). Three things to notice:

- `TestApp.RunAsDefaultUserAsync()` — creates a real user and makes the test "act as" them, so audit fields get populated.
- `TestApp.SendAsync(command)` — sends the command through the **real** MediatR pipeline (validation, authorization, the handler, EF Core) — everything except the HTTP layer.
- The assertions check `CreatedBy`, `Created`, etc. — proving the audit interceptor ran. A mock would never catch a bug there; this uses the real thing.

Write your own that proves a *deletion* actually removes the row:

```csharp
// Illustrative — a new functional test
[Test]
public async Task ShouldDeleteTodoItem()
{
    await TestApp.RunAsDefaultUserAsync();
    var listId = await TestApp.SendAsync(new CreateTodoListCommand { Title = "L" });
    var itemId = await TestApp.SendAsync(new CreateTodoItemCommand { ListId = listId, Title = "T" });

    await TestApp.SendAsync(new DeleteTodoItemCommand(itemId));

    var item = await TestApp.FindAsync<TodoItem>(itemId);
    item.ShouldBeNull();   // it's really gone from the database
}
```

Run just yours:

```bash
dotnet test ./tests/Application.FunctionalTests --filter "Name~ShouldDeleteTodoItem"
```

## Step 3 — Understand what makes the functional test safe to run

Two mechanisms, worth knowing so nothing surprises you:

- **Real database, booted once.** [FunctionalTestSetup](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L14-L46) starts a database via Aspire before any test. (No Docker needed for the SQLite default.)
- **Data reset between tests.** Before each test, the data is wiped ([TestBase](../../tests/Application.FunctionalTests/Infrastructure/TestBase.cs) → Respawn). That's why your test could assume an empty database and build exactly the rows it needed.

One consequence to file away: these tests **can't run in parallel** — the "current user" is stored in shared static state ([TestApp L13-L14](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L13-L14)). If you ever see "flaky" functional tests, do *not* reach for parallelism to fix it — that would *cause* the cross-talk. (This is a genuine trap; senior engineers know it, and now so do you.)

## Step 4 — Test-first, once, to feel it

Try this on the next tiny thing you build: write the test *before* the code. Watch it fail (red) for the right reason — the feature doesn't exist yet. Then write the code until it passes (green). That red-then-green loop is called test-driven development; you don't have to do it always, but doing it a few times teaches you to think about *what correct means* before you think about *how to build it*. That reordering is worth a lot.

## Step 5 — Commit

```bash
git add -A
git commit -m "test: add unit and functional coverage for colour rejection and item deletion"
```

## What you learned

- **Unit vs functional** is a decision, not a preference: lowest layer that can observe the failure you fear.
- Functional tests here use a real database *on purpose* — mocks lied to earlier versions of this code, so the team chose real infrastructure ([ADR-004/008 in fabledocs3](../../fabledocs3/adr-008-real-database-functional-testing.md)).
- Assert the *specific* thing (`ShouldBeNull`, `ShouldContainKey`), not just "no exception."
- The suite is serial by design; parallelism would break it.

## Common mistakes

- Functional-testing pure logic (slow and pointless) or unit-testing a real query (impossible — mocks can't prove SQL translation).
- Asserting too weakly (`ShouldNotBeNull` when you meant to check a specific value).
- Forgetting `RunAsDefaultUserAsync`, then being confused when an authorization or audit assertion fails.

## Next

[Lab 5 — Fix a real bug](lab-05-fix-a-real-bug.md). Time to debug.
