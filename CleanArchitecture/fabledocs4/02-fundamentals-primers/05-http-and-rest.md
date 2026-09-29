# Primer 5 — HTTP & REST Basics

## The concept from scratch

Every interaction between the browser and the server is an **HTTP request** and an **HTTP response**. A request has:

- A **method (verb)**: `GET`, `POST`, `PUT`, `PATCH`, `DELETE` — what you want to *do*.
- A **path**: `/api/TodoLists/5` — what you want to do it *to*.
- Optional **headers**: metadata (content type, the auth cookie from the [auth primer](03-web-auth.md)).
- An optional **body**: data you're sending (usually JSON), for `POST`/`PUT`/`PATCH`.

The response has a **status code** (a three-digit number) and usually a body. Status codes come in families:

- **2xx success** — 200 OK, 201 Created, 204 No Content.
- **4xx client error** — *you* did something wrong: 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found.
- **5xx server error** — the *server* broke: 500 Internal Server Error.

That 4xx-vs-5xx line matters enormously in practice: a 4xx says "fix your request," a 5xx pages the on-call engineer. Returning the wrong family (like the 500-that-should-be-400 bug in [Lab 5](../01-guided-labs/lab-05-fix-a-real-bug.md)) is a real defect.

**REST** is a style for designing these APIs around *resources* (nouns) that you act on with the standard verbs. `GET /api/TodoLists` reads lists; `POST /api/TodoLists` creates one; `DELETE /api/TodoLists/5` removes list 5. The verb carries the intent, so the path can stay a clean noun.

## Where this repo uses it

Open [src/Web/Endpoints/TodoLists.cs](../../src/Web/Endpoints/TodoLists.cs) and read the verb-to-action mapping:

- `MapGet(GetTodoLists)` → `GET /api/TodoLists` → returns 200 with the lists.
- `MapPost(CreateTodoList)` → `POST /api/TodoLists` → returns **201 Created** with the new id.
- `MapPut(UpdateTodoList, "{id}")` → `PUT /api/TodoLists/5` → returns **204 No Content**.
- `MapDelete(DeleteTodoList, "{id}")` → `DELETE /api/TodoLists/5` → 204.

The return types encode the status: [TodoItems.cs L32](../../src/Web/Endpoints/TodoItems.cs#L32) declares `Results<NoContent, BadRequest>`, meaning "this returns 204 or 400" — compile-checked and visible in the API docs.

**Two idempotency ideas you should be able to explain:**

- **PUT is idempotent** — sending the same update twice leaves the same result as sending it once. Setting `Done = true` twice is still just done.
- **POST create is not idempotent** — sending "create a list" twice makes *two* lists. This is why retrying a timed-out POST is dangerous (it can duplicate), and it connects directly to the race condition in the [transactions primer](04-what-a-transaction-is.md).

Being able to say "which of my endpoints are safe to retry?" is a genuinely senior-sounding answer, and now you can reason about it per-verb.

## Where this repo is imperfect (learn to notice)

- `POST /api/TodoLists` returns a `Created` result with a `Location` header pointing at `/TodoLists/{id}` ([TodoLists.cs L36](../../src/Web/Endpoints/TodoLists.cs#L36)) — but the real route is `/api/TodoLists`, and there's **no GET-by-id endpoint** to point at. So the "here's where to find the thing you created" convention is broken. Small, but it's the kind of imperfection a careful engineer spots.
- The id-consistency guard ([TodoItems.cs L34-L35](../../src/Web/Endpoints/TodoItems.cs#L34-L35)) returns a bare 400 with *no body*, while validation failures return a rich 400 *with* a body. Same status, two different shapes — an inconsistency worth noticing.

## The misconception juniors have

*"POST is for creating, GET is for reading, and that's all I need."* The verbs also carry *guarantees* — idempotency, safety (GET must not change data), the right status family. An API that returns 200 for everything, or uses POST to delete, "works" but lies to every client and every cache and every monitoring tool that relies on those conventions. The conventions are the contract.

## Check yourself

1. What's the practical difference, for the on-call engineer, between a 400 and a 500? Which one did the Lab 5 bug wrongly return?
2. Why is `PUT` idempotent but `POST` (create) not, and why does that make retrying a POST riskier?
3. Find one place in the endpoints where the HTTP conventions are imperfect, and explain what a "correct" version would do.
