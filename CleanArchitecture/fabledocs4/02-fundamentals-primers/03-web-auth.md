# Primer 3 — How Web Auth Actually Works

## The concept from scratch

HTTP is **stateless** — each request arrives with no memory of the last one. So how does the server know that the request asking for your todo lists is *you*, when you logged in five minutes and twelve requests ago?

The answer is a **credential the browser attaches to every request.** Two common shapes:

- **A cookie.** When you log in, the server sends back a `Set-Cookie` header. The browser stores it and *automatically* attaches it to every subsequent request to that site. The server reads the cookie, looks up who it belongs to, and knows it's you. This app uses cookies (for its SPA variants).
- **A token (bearer token).** The server returns a token string; the client stores it and manually attaches it as an `Authorization: Bearer ...` header on each request. This app supports this too, for its API-only variant.

Two words you must not confuse:

- **Authentication (authn)** — *who are you?* Proving identity (login → credential).
- **Authorization (authz)** — *what are you allowed to do?* Checking permissions.

They map to two HTTP statuses: **401 Unauthorized** actually means "unauthenticated — I don't know who you are," and **403 Forbidden** means "I know who you are, and you're not allowed."

## Where this repo uses it

**Login and the cookie.** The identity endpoints come from one line — `MapIdentityApi<ApplicationUser>()` in [Users.cs L12](../../src/Web/Endpoints/Users.cs#L12) — which provides register/login/etc. The React client logs in with cookies enabled ([AuthContext.jsx L19-L21](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx#L19-L21)) and detects whether it still has a live session by pinging a "who am I" endpoint on startup ([L12-L17](../../src/Web/ClientApp-React/src/components/api-authorization/AuthContext.jsx#L12-L17)).

**Reading identity per request.** On the server, [CurrentUser L16-L17](../../src/Web/Services/CurrentUser.cs#L16-L17) pulls your id and roles out of the request's authentication data (`HttpContext.User`). This is the concrete implementation of the `IUser` interface from the [DI primer](01-dependency-injection.md) — and it's *scoped* because it's about the *current* request's user.

**Authorization in two places.** First, endpoints require a logged-in user: `RequireAuthorization()` in [TodoItems.cs L13](../../src/Web/Endpoints/TodoItems.cs#L13) — no valid credential, 401, before any handler runs. Second, the [AuthorizationBehaviour](../../src/Application/Common/Behaviours/AuthorizationBehaviour.cs) checks `[Authorize]` attributes on request objects and throws `UnauthorizedAccessException` (→401) or `ForbiddenAccessException` (→403). Two layers of defense: one at the HTTP edge, one in the application core that works even without HTTP (which is why a functional test can prove [an anonymous user is denied](../../tests/Application.FunctionalTests/TodoLists/Queries/GetTodosTests.cs#L50-L58) with no web server involved).

**The client-side guard is not security.** [ProtectedRoute.jsx](../../src/Web/ClientApp-React/src/components/api-authorization/ProtectedRoute.jsx) redirects logged-out users to the login page. That's *user experience* — it stops the UI showing a broken page. It is **not** a security boundary; anyone can call the API directly, bypassing the React app entirely. The real boundary is the server-side checks above. Confusing "the button is hidden" with "the action is protected" is a serious junior mistake.

## The big gap worth knowing

Authorization here answers "are you logged in?" and "are you in this role?" — but **never "is this list *yours*?"** There's no per-user ownership check anywhere. Any logged-in user can read or delete any other user's data by guessing IDs. That's a known, deliberate simplification for this template (it's single-user by design), and it's the first thing you'd fix in a real multi-user app. The vulnerability class is called **IDOR** (Insecure Direct Object Reference). Filing that away now will make you look sharp later.

## The misconception juniors have

*"If the login page and the protected routes work, the app is secure."* No — front-end guards are convenience. Security lives on the server, on every request, independent of what the UI chose to show. Always ask: "if someone skipped my UI and called the API directly with curl, what stops them?"

## Check yourself

1. After you log in, what exactly makes the *next* request identify you as the same person? (Name the mechanism.)
2. What's the difference between 401 and 403, and where in this codebase is each one produced?
3. `ProtectedRoute` hides the todo page from logged-out users. Why is that *not* enough to protect todo data, and where is the actual protection?
