# 01 — Codebase Cartography

Before you can change a system safely you must be able to *place* any file in it: which layer owns it, what contract it participates in, and what breaks if it changes. This module builds that map.

| File | What it gives you |
|---|---|
| [01-system-map.md](01-system-map.md) | Solution shape, layer diagram, ownership map, public vs private surfaces |
| [02-file-reading-order.md](02-file-reading-order.md) | 30 files in reading order with junior/mid/senior tracks |
| [03-domain-glossary.md](03-domain-glossary.md) | Every domain and architecture noun, anchored to code |
| [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) | SDKs, package managers, build/test commands, runtime boundaries, env vars |
| [05-key-flows.md](05-key-flows.md) | Six end-to-end traces — the backbone of the whole curriculum |

Transferable skill: **cartography is a repeatable procedure**, not talent. On any new repo: read the manifest(s), find the composition root (where DI/routing is wired), find one write path and one read path, find where tests get their world from. That's exactly the order these files follow.
