# Systematic Debugging

The method, then five scenarios set in this codebase. The method is the transferable asset; the scenarios build the reflexes.

## The method

1. **Reproduce** — a failing case you can run on demand. No repro, no debugging; you're guessing.
2. **Narrow** — binary-search the stack: is the bad value in the browser, the wire, the endpoint, the pipeline, the handler, or the database? Each probe should *halve* the search space.
3. **Hypothesize** — one falsifiable sentence: "the validator never runs because X."
4. **Test cheaply** — prefer a log line, a breakpoint, an http-file request, or a one-off test over redeploying the world.
5. **Fix the root cause** — not the symptom (if the DTO shows nulls, the fix is rarely in the DTO).
6. **Add regression coverage** — the test that would have caught it, at the lowest layer that can observe it.

Tools of the house, by layer: browser devtools Network tab (what actually crossed the wire — with the Vite proxy, check the *dev server* request and remember it forwards per [vite.config.ts](../../src/Web/ClientApp-React/vite.config.ts#L13-L25)); [Web-webapi.http](../../src/Web/Web-webapi.http) to hit the API without the UI; the Aspire dashboard (structured logs + traces per resource — every request logs via [LoggingBehaviour](../../src/Application/Common/Behaviours/LoggingBehaviour.cs)); EF SQL logging (dev exposes `Microsoft.EntityFrameworkCore` at Information via appsettings if you lower the filter); the debugger on a functional test (fastest server-side repro: no browser, real pipeline); Scalar at `/scalar` for schema truth.

---

## Scenario 1: "I created a list but the item I add to it vanishes on refresh"

Reproduction: create list → immediately add item → F5 → item gone (sometimes).
First question: is the item missing from the **database** or only from the **response**?
Narrowing path:
1. Check the POST `/api/TodoItems` response in devtools — did it return 201 with an id? If no → server path. If yes → …
2. …check the DB (or GET `/api/TodoLists` in the http file): is the item there with the right `ListId`?
3. If the item's `listId` was `0` or stale: inspect what the client sent — [commitNewItem](../../src/Web/ClientApp-React/src/components/Todo.jsx#L211-L222) captures `listId = selectedListId` *before* the await; if the user switched lists mid-flight, state and payload diverge.
Useful probes: Network tab request payload; `TestApp.SendAsync(new CreateTodoItemCommand { ListId = 0, ... })` in a scratch test.
Likely root causes: `CreateTodoItemCommand.ListId` has **no validator rule** ([validator only checks Title](../../src/Application/TodoItems/Commands/CreateTodoItem/CreateTodoItemCommandValidator.cs)) — `ListId = 0` sails through validation and fails (or worse, half-succeeds) at the FK level depending on provider.
Regression test to add: functional — creating an item with `ListId = 0`/nonexistent throws a mapped, deliberate error (after you add the rule or an existence check).
Senior lesson: absence of a validation rule *is* a bug class; validators need review as much as handlers.
Interview version: narrate as "client sent a legal-shaped but semantically invalid FK; the contract had a hole; I closed it at the contract layer and added the regression test at the functional layer."

## Scenario 2: 500 instead of 400 when creating a list via the API with a custom colour

Reproduction: `POST /api/TodoLists {"title":"x","colour":"#123456"}` via [Web-webapi.http](../../src/Web/Web-webapi.http) → 500.
First question: which layer threw — validation (should be 400) or later?
Narrowing path: 1. Response body: generic 500, not ValidationProblemDetails → validation *passed*. 2. Logs (UnhandledExceptionBehaviour logs the request + exception): `UnsupportedColourException`. 3. Find the throw: [Colour.From](../../src/Domain/ValueObjects/Colour.cs#L5-L15) via [CreateTodoList handler L28](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoList.cs#L25-L29). 4. Why unmapped: not in [ProblemDetailsExceptionHandler's switch](../../src/Web/Infrastructure/ProblemDetailsExceptionHandler.cs#L17-L44).
Likely root causes: contract gap — domain invariant violations have no HTTP mapping.
Fix root cause: add a colour rule to the validator (and/or map the exception → 400). Regression: functional test asserting `ValidationException` for bad colours.
Senior lesson: every exception type that can escape a handler needs a deliberate HTTP identity; audit the set periodically (grep `: Exception` in Domain/Application).

## Scenario 3: Duplicate list titles exist in production despite the uniqueness rule

Reproduction: hard — it's a race. Two concurrent POSTs with the same title (write a quick parallel test: two `TestApp.SendAsync` via `Task.WhenAll`).
First question: is the *rule* broken or *bypassed*?
Narrowing path: 1. Single-request test passes → rule works serially. 2. Read [BeUniqueTitle](../../src/Application/TodoLists/Commands/CreateTodoList/CreateTodoListCommandValidator.cs#L21-L26): check-then-act, two operations. 3. Confirm no unique index in [TodoListConfiguration](../../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs). Hypothesis confirmed: TOCTOU.
Fix root cause: unique index (the invariant) + catch `DbUpdateException` → 409/400 (the UX). Keep the validator (nice messages for the 99% case).
Regression: the parallel test; also a migration-time dedupe plan for existing data.
Senior lesson: only the database can enforce cross-row invariants; validators are advisory. Say "TOCTOU" in the interview and define it in the same breath.

## Scenario 4: Checkbox shows done, server says not done

Reproduction: throttle network to offline in devtools → click a checkbox → it stays checked; refresh → unchecked.
First question: UI state bug or failed request?
Narrowing path: 1. Network tab: PUT failed. 2. Read [updateCheckbox](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167): state updated *before* the request, `catch` only logs. 3. Classify: designed optimism, missing rollback — not a mystery, a decision nobody finished.
Useful probes: none needed beyond reading — the lesson is that some "bugs" are incomplete designs; the fix is a product-visible decision (rollback + toast? retry queue?), not a patch.
Regression: hard at unit level in an untested `.jsx` codebase — argue for either a component test (after TSX migration) or an acceptance-test step with network interception (Playwright can fail requests).
Senior lesson: error paths are features; "happy path implemented, sad path logged" is the most common front-end debt.

## Scenario 5: The whole functional test suite fails on a colleague's machine with a timeout

Reproduction: `dotnet test tests/Application.FunctionalTests` → every test errors identically in setup.
First question: one shared cause or many? (Identical error text in *all* tests → shared fixture; read [FunctionalTestSetup](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L14-L46) first.)
Narrowing path: 1. Is the failure inside `OneTimeSetUp`? (Stack trace says so.) 2. Which await hit the [60s token](../../tests/Application.FunctionalTests/FunctionalTestSetup.cs#L17) — build, start, or `WaitForResourceHealthyAsync`? Add a temporary log between them. 3. For container-DB variants: is Docker running / image pulled? For SQLite: is the file locked by a zombie process?
Likely root causes: first-run container pull exceeding 60s; Docker not started; leftover process holding the DB.
Fix root cause: pre-pull in CI/docs, or raise the timeout with a comment; kill zombies.
Regression: none — instead, improve the *failure message* (wrap setup steps with context) so the next person narrows in one read.
Senior lesson: "all tests failed" almost never means all tests failed; it means the fixture failed. Fixing the *diagnosability* is the durable contribution.

---

## Drill

Pick scenario 3. Set a 25-minute timer and actually build the parallel repro test in this repo. Whether or not you finish: write the three-sentence incident summary (symptom → root cause → fix) as you'd post it in a team channel. Rubric — Basic: correct cause named. Solid: repro test compiles. Strong: your summary includes the data-cleanup consideration for existing duplicates.

Interview note: debugging rounds grade your *narration*, not just the fix. Practice saying steps 1–6 out loud with scenario 2 — it's the most narratable of the five. Timed versions of these scenarios live in [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).
