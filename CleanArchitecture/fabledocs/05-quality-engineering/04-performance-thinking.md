# Performance Thinking

Rule zero: **measure before touching anything.** This repo hands you the measurement tools (OTel traces per request, [PerformanceBehaviour's](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L37-L50) >500ms warnings, EF SQL logging, browser devtools). A performance claim without a number is a mood.

## The performance domains in this app, and where to look

| Domain | Where it lives here | Current state (verified by reading) | How to find problems |
|---|---|---|---|
| DB queries | Application handlers | Healthy: single projected query for the main read ([GetTodos L43-L47](../../src/Application/TodoLists/Queries/GetTodos/GetTodos.cs#L43-L47)), `AsNoTracking`, by-key `FindAsync` writes | EF logs: count queries per request; any `await` in a loop |
| Query volume growth | same | **Unbounded**: GetTodos returns *all* lists with *all* items — no paging anywhere. Fine at 10 lists; a product problem at 10,000 items | trace duration vs row count; the fix is paging/virtualization, a contract change — see M-ticket |
| Pipeline overhead | behaviours | Per request: log write incl. username DB lookup ([LoggingBehaviour L27-L30](../../src/Application/Common/Behaviours/LoggingBehaviour.cs#L21-L34)) — a DB hit on **every** authenticated request; PerformanceBehaviour adds another on slow ones ([L43-L46](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L43-L46)) | trace waterfall: identical small query on every span |
| Server allocations | reflection at startup only ([MapEndpoints](../../src/Web/Infrastructure/WebApplicationExtensions.cs#L12-L28)) | fine — one-time cost | dotnet-counters if ever suspicious |
| Network | SPA ↔ API | One aggregate VM avoids chatty loads (good); full-item PUTs for a checkbox are chunky but trivial | devtools: payload sizes, request counts per interaction |
| React rendering | [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx) | Whole feature is one component: every keystroke in the edit input re-renders *all* lists and items. Invisible at 20 items; measurable at 2,000 | React DevTools Profiler: highlight re-renders while typing |
| Bundle | Vite build | Small deps (pico, lucide, router). No code-splitting; login page loads the todo code | `vite build` output sizes; `rollup-plugin-visualizer` if needed |
| Startup | dev DB drop/create/seed ([initialiser](../../src/Infrastructure/Data/ApplicationDbContextInitialiser.cs#L40-L46)) | seconds each run; dev-only tax | — |

The single most interview-useful observation in this table: the **per-request username lookup**. It's a textbook "observability tax" — every request pays a DB round trip so a log line can show a username instead of an id. The fix (log the id; enrich at query time) costs nothing and removes a query from *every* trace. Finding one of these in any codebase you join makes an outstanding first performance PR because it's measurable, safe, and small.

## The classic hunts, with this repo as the range

- **N+1:** none in server code today (the projection prevents it). Know *why*: `ProjectTo` composes the child collection into one translated query. The N+1 version is written out for contrast in [fake-code contrast 4](../04-code-reading-gym/03-fake-code-contrasts.md).
- **Serial async:** [ValidationBehaviour L19-L21](../../src/Application/Common/Behaviours/ValidationBehaviour.cs#L19-L21) already parallelizes validators. Hunt pattern: consecutive independent `await`s → `Task.WhenAll` candidates.
- **Missing indexes:** the uniqueness validator's `AnyAsync(l => l.Title == title)` scans on Title (no index — [TodoListConfiguration](../../src/Infrastructure/Data/Configurations/TodoListConfiguration.cs)). At scale, every create pays a table scan. The unique index from the correctness fix (debug scenario 3) is *also* the performance fix — a lovely two-birds story.
- **Expensive renders:** memoization (`React.memo`, `useMemo`) is absent and *correctly* absent — at this data size it would be complexity without measured need. The senior answer to "should we memo?" is "what did the Profiler say?"
- **Unbounded queries:** GetTodos, as noted. Also `Note` (unbounded text) rides along in every list fetch — a field only the details dialog needs. Projection trimming is the surgical fix.

## Latency budget thinking (transferable)

A request here spends: proxy hop (dev only) → auth middleware → 5 behaviours (2 with possible DB hits) → handler query → serialization. When someone says "the API is slow," decompose against that chain *in order of evidence*: trace first (which span is fat?), then EF log (how many queries? how long?), then code. The repo's >500ms warning threshold ([PerformanceBehaviour L37](../../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L37)) is a *starting* SLO — know that hardcoded numbers like that should graduate to configuration once anyone tunes them.

## Drill

Estimate, then verify by reasoning (no profiler needed): a user with 50 lists × 100 items loads `/todo`. (a) How many SQL queries does GetTodos run? (b) How many rows cross the wire? (c) What does the browser then re-render when they type one character into a new-item input? Answers: (a) 1–2 (projected query; +1 per-request username log lookup) — knowing it's *not* 51 is the point; (b) all 5,000 items incl. notes; (c) the full list panel — all 5,000 rendered rows re-evaluate. Now write the one-paragraph perf plan: paging or per-list fetch, note-field trimming, input isolation (component split) — *in evidence order, cheapest measurement first*.

## Interview angle

- *"How do you find a slow endpoint?"* → trace → SQL count → code, with the username-lookup as your told-from-experience example. [08-interview-prep/03, Q11](../08-interview-prep/03-api-and-data-modeling-questions.md)
- *"When do you memoize React components?"* → Profiler-first answer above. [08-interview-prep/02, Q7](../08-interview-prep/02-frontend-framework-questions.md)
- *"Tell me about a query optimization"* → projection + the index-for-uniqueness twofer.
