# Writing Tests Here — Six Recipes

Each recipe: when to use it, the real file to crib from, the skeleton, and the command to run it. Commands are **inferred** (standard `dotnet test`; not executed while writing these docs). Skeletons are labeled fake code — adapt, don't paste blindly.

Run one project: `dotnet test ./tests/Application.FunctionalTests`
Run one test: `dotnet test ./tests/Application.FunctionalTests --filter "Name~ShouldCreateTodoItem"`

## Recipe 1: Happy-path command test (functional)

Crib from: [CreateTodoItemTests.ShouldCreateTodoItem](../../tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs#L18-L45)
Shape: arrange **through commands**, act via `TestApp.SendAsync`, assert via `TestApp.FindAsync` on the real row — including audit fields.

```csharp
// Illustrative fake code: not from this repo — skeleton for a new command test
public class ArchiveTodoListTests : TestBase
{
    [Test]
    public async Task ShouldArchiveList()
    {
        var userId = await TestApp.RunAsDefaultUserAsync();
        var listId = await TestApp.SendAsync(new CreateTodoListCommand { Title = "L" });

        await TestApp.SendAsync(new ArchiveTodoListCommand(listId));

        var list = await TestApp.FindAsync<TodoList>(listId);
        list!.Archived.ShouldBeTrue();
        list.LastModifiedBy.ShouldBe(userId);   // interceptors are part of the behavior — assert them
    }
}
```

Notes: inherit [TestBase](../../tests/Application.FunctionalTests/Infrastructure/TestBase.cs) (gets the per-test reset); create a user first or `CreatedBy` will be null.

## Recipe 2: Validation-failure test

Crib from: [CreateTodoItemTests.ShouldRequireMinimumFields](../../tests/Application.FunctionalTests/TodoItems/Commands/CreateTodoItemTests.cs#L10-L16) and the richer uniqueness case in [CreateTodoListTests](../../tests/Application.FunctionalTests/TodoLists/Commands/CreateTodoListTests.cs).
Shape: send an invalid command, `Should.ThrowAsync<ValidationException>`. Upgrade beyond the repo's own tests by asserting *which* rule fired:

```csharp
// Illustrative fake code: not from this repo
var ex = await Should.ThrowAsync<ValidationException>(() => TestApp.SendAsync(command));
ex.Errors.ShouldContainKey("Title");
ex.Errors["Title"].ShouldContain(e => e.Contains("must be unique"));
```

Why assert the key: a test that only checks the exception type passes even when the *wrong* rule fires (e.g. NotEmpty instead of unique) — you'd be green for the wrong reason.

## Recipe 3: Not-found and permission-failure tests

Crib from: [UpdateTodoItemTests.ShouldRequireValidTodoItemId](../../tests/Application.FunctionalTests/TodoItems/Commands/UpdateTodoItemTests.cs#L10-L15) (`NotFoundException`) and [GetTodosTests.ShouldDenyAnonymousUser](../../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L50-L58) (`UnauthorizedAccessException` — just *don't* call `RunAs*`).
The one this suite is missing — **cross-user denial** — is what you'll write the day ownership lands:

```csharp
// Illustrative fake code: not from this repo — the isolation test this repo can't have yet
[Test]
public async Task ShouldNotRevealOtherUsersList()
{
    await TestApp.RunAsUserAsync("alice@local", "Alice1234!", []);
    var listId = await TestApp.SendAsync(new CreateTodoListCommand { Title = "Alice's" });

    await TestApp.RunAsUserAsync("bob@local", "Bobby1234!", []);   // switch acting user
    await Should.ThrowAsync<NotFoundException>(                    // 404, not 403: don't confirm existence
        () => TestApp.SendAsync(new DeleteTodoListCommand(listId)));
}
```

Note `RunAsUserAsync` just overwrites the statics — switching identities mid-test is that cheap ([TestApp L48-L80](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L48-L80)).

## Recipe 4: Role-based authorization test

Crib from: [TestApp.RunAsAdministratorAsync](../../tests/Application.FunctionalTests/Infrastructure/TestApp.cs#L43-L46). Pattern: default user → expect `ForbiddenAccessException`; admin → expect success. Prerequisite: a command actually decorated `[Authorize(Roles = Roles.Administrator)]` — none currently is (only [GetTodosQuery](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L9) has plain `[Authorize]`), so writing one is part of any role-gated feature ticket.

## Recipe 5: Domain event side-effect test (fills a real gap)

No existing test asserts the event→handler chain (verified in [testing strategy](01-testing-strategy.md)). Two options:
*Domain unit* (fast, proves raising): construct a `TodoItem`, set `Done = true`, assert `DomainEvents` contains a `TodoItemCompletedEvent`; set `true` again, assert count still 1 (the transition invariant, [TodoItem L17-L24](../../src/Domain/Entities/TodoItem.cs#L17-L24)).
*Functional* (proves dispatch): register a test `INotificationHandler<TodoItemCompletedEvent>` recording invocations via `WebApiFactory.ConfigureTestServices`, drive `UpdateTodoItemCommand{Done=true}`, assert one invocation. The second is the valuable one — it covers [the interceptor](../../src/Infrastructure/Data/Interceptors/DispatchDomainEventsInterceptor.cs), which is where the risk lives.

## Recipe 6: Domain unit test for an invariant

Crib from: [ColourTests](../../tests/Domain.UnitTests/ValueObjects/ColourTests.cs) — construction, conversion, equality, and the rejection case (`Should.Throw<UnsupportedColourException>`, [L45-L48](../../tests/Domain.UnitTests/ValueObjects/ColourTests.cs#L44-L48)). Template for any new value object or entity rule: one test per legal behavior, one per *illegal* behavior, one for equality semantics if it's a value type. No infrastructure, no mocks, milliseconds.

## Acceptance-layer note

You'll rarely add these, but the recipe: a `.feature` file in Gherkin ([Login.feature](../../tests/Web.AcceptanceTests/Features/Login.feature)), step definitions binding phrases to Playwright page objects ([LoginStepDefinitions](../../tests/Web.AcceptanceTests/StepDefinitions/LoginStepDefinitions.cs)), a page object per screen ([LoginPage](../../tests/Web.AcceptanceTests/Pages/LoginPage.cs)). Page objects keep selectors in one place — when `#email` changes, one file updates. There is **no Todo.feature** — writing one is [mid-level ticket M6](../06-contribution-practice/02-mid-level-feature-tickets.md).

## Drill

Write Recipe 5's functional variant for real (it compiles against existing infrastructure — the only new piece is the recording handler). Run it. If you've done it: you have made a genuinely useful contribution candidate; see [06-contribution-practice/01, ticket J3](../06-contribution-practice/01-good-first-tickets.md).
