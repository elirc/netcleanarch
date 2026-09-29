# Learning Rubrics

Observable behaviors, not vague traits. Use these to place yourself honestly and to know what "the next level" concretely looks like. The **Interview-ready** column is the mid-level-loop bar for each skill.

## How to use
Rate yourself on each row: can you *demonstrate* the behavior right now, with this repo, unprompted? Not "do I know about it" — "can I do it." Anything below Mid is your study list; the [cram plan](../08-interview-prep/07-two-week-cram-plan.md) is built around closing these.

## Skill 1: Navigating an unfamiliar codebase
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Finds a file when told its name | Finds the composition root, one read path, one write path unaided | Builds a system map + ownership map + public/private surface from scratch | Can trace a request end to end naming real classes, and place any new feature in the right layer ([cartography](../01-codebase-cartography/README.md)) |

## Skill 2: Reading code for correctness
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Reads what a method does | Identifies inputs, outputs, side effects, dependencies | Names invariants and failure modes with *specific* triggering state | Given an excerpt, states one failure mode as "input X in state Y → wrong outcome Z" ([annotation drills](../04-code-reading-gym/01-annotation-drills.md)) |

## Skill 3: Language & runtime (.NET/C#)
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Uses async/DI/LINQ | Explains what await frees, DI lifetimes, IQueryable deferral | Names the failure each concept causes when misused | Answers a language card with definition + repo anchor + tradeoff + failure mode ([cards 01](../08-interview-prep/01-csharp-dotnet-deep-dive.md)) |

## Skill 4: Frontend (React)
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Writes components with state | Explains render vs commit, effects, stale closures | Diagnoses re-render scope, cache drift, reference-equality traps from real code | Points at Todo.jsx and explains optimistic UI + its missing rollback ([cards 02](../08-interview-prep/02-frontend-framework-questions.md)) |

## Skill 5: API & data modeling
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Writes CRUD endpoints | Chooses verbs/status codes, places validation, knows authn vs authz | Reasons about transactions, idempotency, migrations, concurrency | Explains this repo's validation layers, the migration gap, and the IDOR gap with fixes ([cards 03](../08-interview-prep/03-api-and-data-modeling-questions.md)) |

## Skill 6: Architecture judgment
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Recognizes "it's layered" | Names each layer's responsibility and dependency direction | Spots boundary leaks; argues a tradeoff from both sides | Cites an ADR's decision + its "Harder" consequences, and can argue the rejected side ([architecture](../03-architecture-and-patterns/README.md)) |

## Skill 7: Testing
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Writes a happy-path test | Tests validation/permission/error paths at the right layer | Designs a test strategy; knows each layer's blind spot; prevents flake | Explains real-DB-vs-mock choice and picks the lowest layer that observes a given fear ([testing](../05-quality-engineering/01-testing-strategy.md)) |

## Skill 8: Debugging
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Adds print statements randomly | Follows reproduce→narrow→hypothesize→fix | Fixes root cause + adds regression + improves diagnosability | Narrates a debugging sim cleanly, checking cheap/likely first, ending with a test ([debugging](../05-quality-engineering/03-systematic-debugging.md)) |

## Skill 9: Security awareness
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Knows "validate input" | Names IDOR, XSS, CSRF, injection and where they'd occur | Maps threats to specific code; designs structural (not per-handler) defenses | Walks the IDOR attack + the global-query-filter defense on this repo ([security](../05-quality-engineering/05-security-checklist.md)) |

## Skill 10: Performance
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Adds caching reflexively | Measures first; recognizes N+1, serial async, missing indexes | Decomposes latency by evidence; knows when *not* to optimize | Answers "find a slow endpoint" with trace→SQL→code and a real example ([performance](../05-quality-engineering/04-performance-thinking.md)) |

## Skill 11: Contribution & communication
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Opens a PR | Writes what/why/tested/risk; scopes changes small | Writes RFCs with honest tradeoffs; reviews at correctness level; disagrees kindly | Has a real merged-quality ticket + a design note, and can tell the STAR story ([contribution](../06-contribution-practice/README.md), [career](../07-career-and-collaboration/README.md)) |

## Skill 12: System design
| Junior | Mid | Senior | Interview-ready |
|---|---|---|---|
| Jumps to tables/endpoints | Requirements first; justifies each choice; handles one variation | Asks clarifying questions; volunteers gaps/failure modes; adapts under new constraints | Runs the 6-step walkthrough + a live variation, closing with "what I'd document and measure" ([system design](../08-interview-prep/04-system-design-from-this-repo.md)) |

## Self-assessment checklist (interview-ready overall)
- [ ] I can draw this system and trace a write path with real class names, unprompted.
- [ ] I can answer any card in [08-interview-prep](../08-interview-prep/README.md) files 01–03 at mid+ with an anchor and a failure mode.
- [ ] I can run the [system-design walkthrough](../08-interview-prep/04-system-design-from-this-repo.md) + one variation without losing the thread.
- [ ] I can narrate a debugging sim ending in a regression test.
- [ ] I can review a PR with two blocking findings, phrased kindly and specifically.
- [ ] I have four STAR stories under 2 minutes, each ending in impact.
- [ ] I have done at least one [ticket](../06-contribution-practice/01-good-first-tickets.md) for real.

Seven checks = ready for a mid-level fullstack loop. The gaps *are* your prep plan.

## The one-line ladder to keep in mind
Junior: *"How do I make it work?"* → Mid: *"Is this the right pattern, and what does it cost?"* → Senior: *"What does this commit us to, who pays, and how do we reduce the risk?"* Mid-level interviews are testing your reach into that second and third question — every rubric row above is a place to practice reaching.
