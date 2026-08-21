# Protocol tests

Tests use synthetic fixtures only. Never copy private runtime project data into this repository.

## Required matrix

1. **Correct routing** — workspace binding resolves the intended project.
2. **Wrong-workspace refusal** — bootstrap for project A run from project B makes no binding/runtime changes.
3. **Project isolation** — similarly named projects do not leak state into each other.
4. **Fresh-chat round trip** — new session reads binding → hub routing → correct project → scoped response write.
5. **Instruction preservation** — pre-existing unrelated `AGENTS.md` content survives binding insertion.
6. **Provenance** — validated evidence stays separate from unverified/historical claims.
7. **Pre-write refresh** — shared project write re-reads latest authoritative state.
8. **Stale SHA detection** — changed project blob SHA blocks blind overwrite.
9. **Two-session reconciliation** — concurrent sessions merge without lost updates.
10. **Schema migration** — each migration preserves routing and isolation invariants.

## Current regression assets

- `fixtures/synthetic-scenarios.md` — reusable synthetic cases.
- `results/v0.1.0-static-regression.md` — current static consistency verdict for skill `0.1.0` / schema `1`.
- `LIVE_CONCURRENCY_TEST.md` — required live two-session stale-SHA/reconciliation procedure.

## Acceptance rule

Behavior-affecting skill changes should add or update a synthetic test scenario before release. Runtime schema migrations must define their own validation cases.

A static PASS validates protocol consistency only. Concurrency is not considered live-validated until the live synthetic procedure passes with retained evidence.