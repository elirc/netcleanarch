# 03 — Architecture & Patterns

The module where "how do I make it work" becomes "is this the right pattern." Everything here is anchored to real code and paired with the interview version of the same idea.

| File | Covers |
|---|---|
| [01-boundaries-and-layers.md](01-boundaries-and-layers.md) | What each layer owns, must not own, and where the boundaries hold or leak |
| [02-data-model-and-persistence.md](02-data-model-and-persistence.md) | Entities, schema config, transactions, and how to change a schema safely |
| [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) | The full validation stack and the authn/authz map — including what's missing |
| [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) | Domain events, interceptors, idempotency, retries, outbox thinking |
| [05-pattern-catalog.md](05-pattern-catalog.md) | 14 pattern cards: recognize, apply, and know the failure modes |
| [06-architecture-critique.md](06-architecture-critique.md) | Strongest choices, real risks, and "what I'd change owning this for 3 months" — doubles as system-design interview prep |

Suggested order: 01 → 05 → 03 → 02 → 04 → 06. The pattern catalog early gives you names for what the other files analyze.
