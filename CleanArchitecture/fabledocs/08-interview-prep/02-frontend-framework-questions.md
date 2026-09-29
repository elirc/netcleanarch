# Frontend Framework Questions — Cards

12 cards, React-primary with Angular contrasts (this repo has both, [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx) and [todo.component.ts](../../src/Web/ClientApp/src/app/todo/todo.component.ts) — being able to compare the *same feature* in both is a standout).

---

## Q1: "Explain React's render vs commit."
Testing: the render model, not just "components re-render."
Anchor: [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx) — state changes trigger re-render.
Junior: "state change re-renders." Mid: "render is React calling the component function to compute the next tree; commit is applying the diff to the DOM. Render is pure and can run multiple times; effects run after commit." Senior: connects to a bug — "the whole Todo feature is one component, so a keystroke in the edit input re-renders every list and item; fine at 20 items, a Profiler finding at 2,000 ([performance](../05-quality-engineering/04-performance-thinking.md))."

## Q2: "What are effects for, and what's a common misuse?"
Anchor: [Todo.jsx mount fetch L33-L42](../../src/Web/ClientApp-React/src/components/Todo.jsx#L33-L42), [selection reset L44-L47](../../src/Web/ClientApp-React/src/components/Todo.jsx#L44-L47).
Junior: "run code after render." Mid: "synchronize with external systems — fetch on mount, reset editor when `selectedListId` changes via the dependency array." Senior: the misuse — "effects to derive state you could compute during render; the repo correctly derives `selectedList` inline ([L49](../../src/Web/ClientApp-React/src/components/Todo.jsx#L49)) instead of storing it in an effect."

## Q3: "What's a stale closure? Show one you'd avoid."
Anchor: functional updates throughout Todo.jsx, e.g. [commitNewItem L218-L221](../../src/Web/ClientApp-React/src/components/Todo.jsx#L211-L221).
Junior: unfamiliar. Mid: "a callback captures state from an old render; if you `setLists(lists.map(...))` with a captured `lists` across an await, a concurrent update gets clobbered — use `setLists(ls => ...)`." Senior: points to the repo *consistently* using the functional form and to [review kata 8](../04-code-reading-gym/04-review-katas.md) where a migration nearly reintroduced the bug.

## Q4: "React state vs context vs a server-cache library — when each?"
Anchor: [useState in Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx#L9-L22), [AuthContext](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx), [ThemeContext](../../src/Web/ClientApp-React/src/components/ThemeContext.jsx).
Junior: "context for global state." Mid: "component state for local UI, context for low-churn app-wide values (auth, theme — exactly the two contexts here), a server-cache lib for server data." Senior: "the repo stores server data in `useState` and hand-patches the cache in six places — it works but drifts; TanStack Query exists to replace that hand-rolled cache with fetch+invalidate. Context would be the *wrong* tool for the high-churn lists data (re-renders everything)."

## Q5: "How does this app do optimistic updates, and what's missing?"
Anchor: [updateCheckbox L161-L167](../../src/Web/ClientApp-React/src/components/Todo.jsx#L161-L167).
Junior: "updates UI immediately." Mid: "state updates before the server confirms for responsiveness; on failure it should reconcile." Senior: the gap — "here the catch only `console.error`s, so a failed save leaves the UI showing the wrong state; the disciplined pattern is snapshot → apply → restore-on-failure + notify ([contrast 5](../04-code-reading-gym/03-fake-code-contrasts.md), [ticket J8](../06-contribution-practice/01-good-first-tickets.md))."

## Q6: "When would you use a ref instead of state?"
Anchor: [editCancelledRef / dialog refs, Todo.jsx L24-L31](../../src/Web/ClientApp-React/src/components/Todo.jsx#L24-L31).
Junior: "to access the DOM." Mid: "for values that shouldn't trigger re-render or that need synchronous reads — dialog element handles, and the cancel flag that coordinates blur-vs-Escape." Senior: explains *why a ref not state* for the cancel flag — "state updates are async/batched; the blur handler needs to read 'was this cancelled' synchronously, so a ref is correct, not a hack."

## Q7: "When do you memoize (`memo`/`useMemo`/`useCallback`)?"
Anchor: **absence** of memoization in [Todo.jsx](../../src/Web/ClientApp-React/src/components/Todo.jsx) — and why that's fine.
Junior: "always, for performance." Mid: "only when a measured render cost or referential-identity problem justifies it — premature memo adds complexity and its own bugs." Senior: "this component has none and correctly so at its data size; the answer to 'should we memo?' is 'what did the Profiler say?' If item lists grew huge, I'd split the component *before* sprinkling memo."

## Q8: "Reference equality vs value equality in React state — a trap you've seen?"
Anchor: [`editingItem === item`, Todo.jsx L266](../../src/Web/ClientApp-React/src/components/Todo.jsx#L266).
Junior: unaware. Mid: "comparing objects by identity works only while the same instances stay in state." Senior: "this edit UI relies on `editingItem === item` reference equality; it works today but would silently break the moment lists are re-fetched (new object identities) — comparing by `id` makes it robust. Fragile invariants like this are what I look for in review."

## Q9: "How does auth state work in this SPA?"
Anchor: [AuthContext L12-L28](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx#L12-L28), [ProtectedRoute](../../src/Web/ClientApp-React/src/components/api-authorization/ProtectedRoute.jsx).
Junior: "a login page." Mid: "cookie-based; the app probes `manage/info` on mount to detect a live session, and `ProtectedRoute` redirects unauthenticated users." Senior: "crucially `ProtectedRoute` is UX, not security — the API enforces auth server-side; a determined user hits the API directly. The client guard just improves the experience."

## Q10: "How does the frontend get its API types, and get built/served?"
Anchor: [nswag.json](../../src/Web/ClientApp-React/nswag.json) + [prestart hook](../../src/Web/ClientApp-React/package.json#L27-L35); [Program.cs fallback L47-L48](../../src/Web/Program.cs#L47-L48).
Junior: "writes fetch calls." Mid: "NSwag generates a typed client from the OpenAPI doc on `prestart`; dev runs a Vite dev server proxying `/api` to the backend, production serves the built SPA from wwwroot with a fallback to index.html for client routing." Senior: adds the generation-order gotcha (needs the .NET build first) and that the `.jsx` seam means type safety stops at the network boundary.

## Q11: "React vs Angular — you've worked in both here; contrast the state model."
Anchor: [Todo.jsx useState](../../src/Web/ClientApp-React/src/components/Todo.jsx#L9-L22) vs [todo.component.ts signals L20-L33](../../src/Web/ClientApp/src/app/todo/todo.component.ts#L20-L33).
Junior: "different frameworks." Mid: "React uses hooks/`useState` and manual immutable updates; Angular 21 here uses signals (`signal`, `computed`) and RxJS for HTTP — `selectedList` is a `computed` in Angular vs derived-inline in React." Senior: "both solve the same problems — the Angular version leans on `effect()` for the selection-reset that React does with a dependency array; comparing the two implementations of one feature is the fastest way to see what's framework-idiom vs fundamental."

## Q12: "How do you handle a server validation error in the UI?"
Anchor: [commitNewList error parsing L77-L83](../../src/Web/ClientApp-React/src/components/Todo.jsx#L77-L83) vs the swallowed [updateListOptions L96-L102](../../src/Web/ClientApp-React/src/components/Todo.jsx#L96-L102).
Junior: "show an alert." Mid: "parse the 400 ProblemDetails body, pull the field errors, show them inline — the create dialog does exactly this with `errors.Title[0]`." Senior: notes the inconsistency — "the *update* dialog swallows the same error (`console.error` only); consistent error UX means reusing the parse logic ([ticket J4](../06-contribution-practice/01-good-first-tickets.md)), and it's coupled to the PascalCase ProblemDetails shape."

---

## Meta-drill

For any three cards, give the answer while *pointing at the actual code* (open the file). The interview version won't have the file open — but rehearsing with it builds the muscle memory to say "in the app I studied, the component did X on line-such" with confidence. Then do Q11 as a whiteboard: sketch the same feature's state model in React and Angular side by side.
