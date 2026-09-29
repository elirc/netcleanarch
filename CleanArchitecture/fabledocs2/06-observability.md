# 6. Observability

## What it does and why it exists

The question observability answers is: *when this breaks at 3 a.m., how will you know, and how fast can you find why?* The foundation is solid; the details are where I'll show you the warts.

The telemetry backbone lives in [src/ServiceDefaults/Extensions.cs](../src/ServiceDefaults/Extensions.cs) — a project every service references. It wires **OpenTelemetry** for logs, metrics, and traces ([L44-L71](../src/ServiceDefaults/Extensions.cs#L44-L71)): ASP.NET Core + HttpClient + runtime instrumentation for metrics, request + HTTP-client tracing, all exported via OTLP when `OTEL_EXPORTER_OTLP_ENDPOINT` is set ([L73-L80](../src/ServiceDefaults/Extensions.cs#L73-L80)) — which the Aspire dashboard sets automatically in dev. So locally you get a dashboard with structured logs, distributed traces, and metrics per resource, for free.

On top of that, three cross-cutting behaviours give per-request visibility without any handler writing logging code:
- [LoggingBehaviour](../src/Application/Common/Behaviours/LoggingBehaviour.cs) logs every request on entry.
- [UnhandledExceptionBehaviour](../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs) logs any exception with the request, then rethrows ([L21-L27](../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L21-L27)).
- [PerformanceBehaviour](../src/Application/Common/Behaviours/PerformanceBehaviour.cs) warns on any request slower than 500ms ([L37-L50](../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L37-L50)).

Combined with per-request traces, you can follow one user action across the whole stack in the dashboard. That's the payoff of the pipeline pattern showing up again in a new place.

## The hardest decisions

### Decision 1: We log whole request payloads — and that's a security incident with a fuse

This is the one that makes me nervous. The logging behaviours serialize the *entire command object* into the log: `{@Request}` in [LoggingBehaviour L32-L33](../src/Application/Common/Behaviours/LoggingBehaviour.cs#L32-L33) and again in [UnhandledExceptionBehaviour L25](../src/Application/Common/Behaviours/UnhandledExceptionBehaviour.cs#L25). Structured logging with destructuring — genuinely great for debugging, because when something fails you can see exactly what was sent.

The alternative was to log only metadata (request type name, user id, a correlation id) and never the payload. I chose full payloads because for a *template* with commands that carry nothing sensitive (a todo title, a colour), the debuggability is pure upside and the risk is zero.

**The fuse:** the risk is zero *today*, because no command in this repo carries a secret. The day someone adds a `ChangePasswordCommand`, or a `CreateUserCommand` with the plaintext password, or a command with PII, that value goes straight into the log stream — and log streams get shipped to third-party aggregators, indexed, retained, and read by people who should never see a password. This is the classic way credentials end up in logs: not a dramatic mistake, just a sensible-for-now default that nobody revisited when the data changed. If you fork this and start adding commands, the redaction policy — a destructuring allow-list, or a `[LogMasked]`-style convention — needs to land *before* the first sensitive command, not after the incident. I'd honestly consider this the highest-priority piece of tech debt in the observability story.

### Decision 2: Observability that adds load exactly when you're already struggling

Look closely at [PerformanceBehaviour L43-L46](../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L43-L46) and [LoggingBehaviour L27-L30](../src/Application/Common/Behaviours/LoggingBehaviour.cs#L27-L30): to put a *username* (not just an id) in the log line, they call `IIdentityService.GetUserNameAsync`, which is **a database query.** LoggingBehaviour does it on *every* authenticated request. PerformanceBehaviour does it again on every slow one.

The intent was good — a human reading logs wants "alice@example.com," not a GUID. But the cost is a DB round trip per request purely for a log label, and the PerformanceBehaviour version is worse: it fires precisely when a request is *already slow*, adding a query to a request that's struggling. Observability is supposed to be cheap; this is observability that taxes the system hardest at its worst moment.

**What I'd do differently:** log the user *id* (which we already have, for free, from `IUser`) and resolve the name at query time in the dashboard, or enrich asynchronously. The name is *derivable* — it never needed to be on the hot path. This is a small, safe, measurable performance win and a great early PR: removing a query from every authenticated request. I left it because for a low-traffic template it's invisible, but it's the wrong pattern to teach, and I'd rather you learn to *see* "we're doing I/O to produce a log label" as a smell.

### Decision 3: Health checks are Development-only, and there's no database check

Health endpoints exist — `/health` (all checks) and `/alive` (liveness only) — but they're gated to Development ([ServiceDefaults L101-L118](../src/ServiceDefaults/Extensions.cs#L101-L118)). The code even carries a comment warning that exposing them in production has security implications. That gating is a *deliberate* safety default: health endpoints can leak internal topology, so opting them out of production by default is the conservative choice.

The wart on top of it: even the checks we *have* are thin — `/health` only registers a liveness self-check ([L92-L96](../src/ServiceDefaults/Extensions.cs#L92-L96)). There is **no database health check**, so a dead database wouldn't fail readiness. The `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore` package is already referenced ([Directory.Packages.props L23](../Directory.Packages.props#L23)); adding `AddDbContextCheck<ApplicationDbContext>()` is a one-liner we just never did.

**The tradeoff:** a production fork needs readiness/liveness for its load balancer, which means explicitly (and securely) enabling these endpoints *and* adding real checks — the DB check at minimum. As shipped, a load balancer has nothing meaningful to probe in production. That's correct for a template (don't expose internals by default) and a gap for a real deployment (you have no readiness signal). Know which one you're building.

## Evidence to open

- [src/ServiceDefaults/Extensions.cs L44-L71](../src/ServiceDefaults/Extensions.cs#L44-L71) — the OpenTelemetry setup. This part is genuinely good.
- [src/Application/Common/Behaviours/LoggingBehaviour.cs L27-L33](../src/Application/Common/Behaviours/LoggingBehaviour.cs#L27-L33) — the per-request payload log *and* the username DB query.
- [src/Application/Common/Behaviours/PerformanceBehaviour.cs L37-L50](../src/Application/Common/Behaviours/PerformanceBehaviour.cs#L37-L50) — the 500ms watchdog (note: the threshold is a hardcoded literal, which should be configuration the day anyone tunes it).
- [src/ServiceDefaults/Extensions.cs L92-L118](../src/ServiceDefaults/Extensions.cs#L92-L118) — health checks, dev-gated, DB-check-less.

## The incident this half-caused

A fork of this template extended a command with a field that, in their domain, contained a customer's personal data. It flowed through the exact `{@Request}` logging you see in [LoggingBehaviour L32-L33](../src/Application/Common/Behaviours/LoggingBehaviour.cs#L32-L33), into their centralized log platform, where it sat searchable and retained. Nobody noticed for months — the logs *worked*, which is the problem; nothing was broken, the data was just quietly everywhere it shouldn't be. It surfaced during a compliance review, and then it was a scramble: purge historical logs, rotate anything exposed, add redaction, and explain to the compliance team how PII ended up in a log aggregator. No outage, no alert — just a slow, expensive cleanup and an awkward meeting.

I call it "half-caused" because the template didn't put the PII there; the fork did. But the template's *default* — log the whole payload, no redaction policy, no guardrail — made it the path of least resistance. That's the honest weight of a default: it's the decision everyone downstream makes without deciding. The lesson for you: **when you add a field to a command, ask "would I be comfortable seeing this value in our log platform?" If not, the logging default is working against you, and redaction has to come first.** And the meta-lesson — the most dangerous observability failures aren't "we couldn't see the problem." They're "we saw, and stored, something we never should have."

## Three questions before you move on

1. A request comes in, runs its handler, and takes 700ms. Walk me through every DB query that happens *just for logging and telemetry* on that one request. Then tell me which of those queries is the most wasteful and why.
2. Someone adds a `ResetPasswordCommand` carrying a new plaintext password. With zero other changes, where does that password end up, via which two files? What's the fix, and why does it have to land before the command, not after?
3. In production, our `/health` endpoint is disabled and, even in dev, doesn't check the database. Your load balancer needs to stop routing to an instance whose database connection has died. What two changes do you make, and what's the security consideration you have to respect while making the first one?

---

*That's the tour. You've now seen the six subsystems and, more importantly, the way I think about them — every good decision has a bill attached, and a senior engineer is someone who can tell you the price without being asked. When you hit something in this codebase that seems wrong, it might be a wart I know about (I've flagged a lot of them), or it might be one I don't. Either way: open the file, trace it, and come tell me. That's the job.*
