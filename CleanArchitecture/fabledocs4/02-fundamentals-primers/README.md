# Part 2 — Fundamentals Primers

The labs assume you understand a handful of concepts that a CRUD tutorial never really explains. These five primers teach each one **from first principles**, then show exactly where this codebase relies on it. Read a primer whenever you hit a concept you couldn't explain out loud to another junior.

| Primer | The question it answers | Read it when |
|---|---|---|
| [01 — Dependency Injection](01-dependency-injection.md) | "Why does nothing ever call `new` on the classes that matter?" | A constructor takes interfaces and you wonder who supplies them |
| [02 — Async / Await](02-async-await.md) | "What does `await` actually do, and why is everything `async`?" | Every method returns `Task` and you're not sure why |
| [03 — How Web Auth Actually Works](03-web-auth.md) | "How does the server know who I am on the next request?" | You hit cookies, `[Authorize]`, 401 vs 403 |
| [04 — What a Transaction Is](04-what-a-transaction-is.md) | "How does a write either fully happen or not at all?" | You see `SaveChangesAsync` and 'unit of work' |
| [05 — HTTP & REST Basics](05-http-and-rest.md) | "What do GET/POST/PUT/DELETE and status codes really mean?" | You read the endpoints and want the *why* behind the verbs |

## How to read these

Each primer is: **the concept from scratch** (no jargon assumed) → **where this repo uses it** (with anchors) → **the one misconception juniors have** → **check yourself** (three questions). Don't skim. If you can teach the concept to someone else afterward, it stuck. If you can't, read it again — that's not failure, that's how understanding works.

These aren't reference docs; they're the *why* underneath the reference docs in [fabledocs/02-stack-and-language-mastery](../../fabledocs/02-stack-and-language-mastery/README.md), which go deeper once these click.
