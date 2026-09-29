# Lab 1 — Your First Change: Add a Colour

**Goal:** add a new colour (`Pink`) to the app's palette, end to end, and prove it works with a test you write. By the end you'll have made a real change and seen exactly which files a "simple" feature touches.

**Time:** 30–45 minutes. **Prereqs:** the app builds ([tooling §1](../04-tooling-and-debugging.md)).

## Why this change is a good first one

It's *additive* (you're not changing existing behaviour), it follows a pattern that already exists (six colours are already defined), and it has a small **blast radius** — the set of things a change can break. Recognizing low-risk changes is a real skill; this one is low-risk because the UI renders whatever colours the API sends, so you won't have to touch the frontend at all. (You'll confirm that claim yourself in step 5.)

## Step 0 — Branch

```bash
git switch -c lab/01-add-colour
```

## Step 1 — Understand the current shape

Open [src/Domain/ValueObjects/Colour.cs](../../src/Domain/ValueObjects/Colour.cs). This is a **value object** — a small type defined by its value, with rules baked in. Read three things:

- The static colours, [L17-L29](../../src/Domain/ValueObjects/Colour.cs#L17-L29): `Red`, `Orange`, `Green`, `Teal`, `Blue`, `Purple`, `Grey`, each a hex code.
- The `SupportedColours` list, [L48-L60](../../src/Domain/ValueObjects/Colour.cs#L48-L60) — the allow-list `Colour.From` checks against.
- `Colour.From`, [L5-L15](../../src/Domain/ValueObjects/Colour.cs#L5-L15) — the factory that *rejects* anything not in that list by throwing `UnsupportedColourException`.

**Pause and predict:** if you add a static `Pink` but forget to add it to `SupportedColours`, what happens when someone calls `Colour.From(Pink)`? (Answer at the end.)

## Step 2 — Add the colour

Add a `Pink` static beside the others in [Colour.cs](../../src/Domain/ValueObjects/Colour.cs#L17-L29):

```csharp
// Illustrative — add alongside the existing statics (~line 29)
public static Colour Pink => new("#EC407A");
```

Then add it to `SupportedColours` ([L48-L60](../../src/Domain/ValueObjects/Colour.cs#L48-L60)):

```csharp
// Illustrative — add inside the SupportedColours getter, next to the other yields
yield return Pink;
```

## Step 3 — Expose it through the API

The read endpoint returns the list of colours the UI shows in its picker. Open [src/Application/TodoLists/Queries/GetTodos/GetTodos.cs](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L32-L41) — there's a hand-built `Colours` list. Add Pink:

```csharp
// Illustrative — add to the Colours collection (~line 40)
new ColourDto { Code = Colour.Pink, Name = nameof(Colour.Pink) },
```

**Notice** what just happened: the domain *rule* (which colours are valid) and the API *payload* (which colours the UI shows) are maintained in two different places. That's a small design smell — a senior would ask "why isn't the API list derived from `SupportedColours`?" Hold that thought; it's a real observation, not a mistake you made.

## Step 4 — Write a test

Open [tests/Domain.UnitTests/ValueObjects/ColourTests.cs](../../tests/Domain.UnitTests/ValueObjects/ColourTests.cs). Copy the shape of an existing test and add one that proves `Colour.From("#EC407A")` returns your colour:

```csharp
// Illustrative — add a test method following the existing ones
[Test]
public void ShouldReturnPinkGivenSupportedCode()
{
    var colour = Colour.From("#EC407A");
    colour.ShouldBe(Colour.Pink);
}
```

`ShouldBe` comes from Shouldly (an assertion library); it works here because value objects compare by value, not reference — that's what [ValueObject.cs](../../src/Domain/Common/ValueObject.cs#L23-L44) does. If that sentence was fuzzy, that's fine for now.

## Step 5 — Build, test, and verify your blast-radius claim

```bash
dotnet build
dotnet test ./tests/Domain.UnitTests
```

Both should pass. Now verify the claim from the top — that you didn't need to touch the frontend. Open [src/Web/ClientApp-React/src/components/Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx#L330-L336): the colour picker `.map`s over whatever colours the API returned. It never hard-codes a colour. **That's why your change stopped at the API.** You reasoned about blast radius and the code confirmed you.

If you want to see it live: run the app ([tooling §1](../04-tooling-and-debugging.md)), open the todo UI, create a list, and Pink is in the picker.

## Step 6 — Commit

```bash
git add -A
git commit -m "feat: add Pink to the supported colour palette"
```

## What you learned

- A "simple" feature touched **four files across three layers** (Domain, Application, tests) — and you can *explain why each one was necessary* and why the frontend wasn't.
- Value objects enforce rules in one place (`Colour.From`).
- You reasoned about blast radius *before* changing code and then confirmed it against the code. That habit — predict, then verify — is the whole game.

## Common mistakes

- Forgetting `SupportedColours` (that's the pause-and-predict answer: `Colour.From("#EC407A")` would **throw `UnsupportedColourException`**, because the factory checks the allow-list and Pink wouldn't be on it — your test would fail with that exception, which is the system correctly telling you the rule and the value disagree).
- A malformed hex code with the wrong number of digits — the app doesn't validate hex *format*, only membership in the list, so a typo like `#EC40` would "work" as a distinct colour. Notice what the code does and doesn't guarantee.

## Next

[Lab 2 — Add a field end to end](lab-02-add-a-field.md). This one's bigger; it's the vertical slice.
