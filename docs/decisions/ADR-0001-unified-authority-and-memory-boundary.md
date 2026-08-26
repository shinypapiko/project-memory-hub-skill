---
document_role: architecture-decision-record
adr_id: ADR-0001
status: accepted
accepted: 2026-08-26
milestones:
  - M1.1
  - M1.2
runtime_effective: false
release_required_for_runtime_effect: true
proposal: docs/proposals/UNIFIED_AUTHORITY_MODEL.md
initial_review: docs/reviews/2026-08-26-unified-authority-model-review.md
final_approval: docs/reviews/2026-08-26-unified-authority-model-final-approval.md
---

# ADR-0001 — Unified authority model and local/Hub memory boundary

## Status

**Accepted architecture decision.**

This ADR records the approved target architecture for M1.1 and M1.2. It is normative for subsequent unified-architecture design work, but it does **not** by itself change current released Hub Skill `0.1.0`, Runtime Hub schema-1 behavior, or the currently deployed local Project Memory Skill. Runtime behavior changes require later compatibility, migration, implementation, test, and release work.

## Context

A strict read-only audit found two materially different systems:

1. a mature local Project Memory implementation centered on workspace `.ai/` files, selective retrieval, experiments, decisions, handoffs, and executable helper tooling;
2. a Hub Protocol `0.1.x` / Runtime Hub schema-1 design centered on deterministic multi-project routing, remote project state, shared sessions, and stale-SHA optimistic-concurrency rules.

Using both systems without a defined semantic boundary creates a dual-source-of-truth risk. The design therefore requires authority to be assigned by information scope rather than by declaring either local or remote storage globally superior.

## Decision

R1–R12 from `docs/proposals/UNIFIED_AUTHORITY_MODEL.md` are accepted in full.

### 1. Evidence precedes memory

Direct/reproducible workspace evidence—source files, datasets, generated outputs, raw logs, and experiment artifacts—remains the factual backstop. Memory records summarize and organize evidence; they do not outrank contradictory direct evidence.

### 2. Local memory owns detailed and active project state

- `.ai/PROJECT.md` owns stable detailed local project background, constraints, terminology, repository/workspace context, and durable local assumptions.
- `.ai/CURRENT.md` owns active local work: current objective, route, latest local result, blockers, and near-term actions.
- `.ai/TASKS.md`, local indexes, DEC/EXP records, local sessions, research, and handoffs remain local by default.
- Detailed local records are not wholesale mirrored into the Hub.

### 3. The Hub owns shared coordination state

- Hub `projects/INDEX.md` remains the remote multi-project identity/routing authority.
- Under the future unified architecture, Hub `projects/<project-id>/PROJECT.md` becomes the **last reconciled shared checkpoint**, not a mirror of `.ai/PROJECT.md + .ai/CURRENT.md`.
- Hub sessions/research/decisions contain only explicitly shared material with appropriate scope and provenance.

### 4. `AGENTS.md` is binding, not memory

`AGENTS.md` remains a small workspace invocation/binding layer containing items such as stable `project_id`, Hub entry metadata, and local-memory location. It must not accumulate active objectives, experiment results, task lists, shared checkpoint prose, or project history.

### 5. Promotion is explicit, minimized, and derived

Local detailed state may produce a compact shared representation only through explicit promotion. Promotion is not file synchronization and does not transfer ownership of the local source record.

The shared candidate must be minimized to fields in the shared-checkpoint schema and preserve permitted provenance.

### 6. Privacy/classification fails closed

- `UNCLASSIFIED` is local-only and non-publishable until classification completes.
- P3 `PROHIBITED_SYNC` material hard-blocks the publication transaction.
- The same candidate payload must not be made publishable by silently redacting P3 fragments and continuing.
- A safe shared result must be a new independently derived/sanitized object, classified on its own contents and carrying only permitted provenance.

### 7. Policy inheritance cannot weaken safety

For ordinary semantic configuration, more specific rules may refine broader rules, but:

- explicit prohibitions cannot be weakened;
- the most restrictive applicable privacy rule wins;
- provenance requirements may be strengthened but not silently removed;
- evidence/history retention floors may not be weakened by child fields or compact derivatives;
- authority is never reassigned implicitly to a cache, mirror, summary, or derivative.

### 8. Retention and deletion are explicit

Supersession, compaction, omission, or deleting a derivative are not equivalent to deleting the owning source record.

Absence of a field from a later checkpoint is not an implicit delete request. Destructive deletion/purge requires an explicit operation governed by the owning record's policy. Evidence/history defaults to preserve, archive, or supersede unless a later explicit privacy/purge policy requires stronger behavior.

### 9. Remote-to-local hydration does not overwrite local memory wholesale

A newer Hub checkpoint is reconciliation input. It must not directly replace `.ai/PROJECT.md`, `.ai/CURRENT.md`, or the local evidence/history tree. Compatible shared changes may be incorporated into local work while preserving local provenance and source history.

### 10. Shared reconciliation is scoped B/R/L semantic reconciliation

Before Hub reconciliation, local state is projected into a minimized shared candidate.

```text
B = last fully reconciled Hub checkpoint
R = latest refreshed remote Hub checkpoint
L = local proposed shared-checkpoint candidate
```

Only semantic fields belonging to the shared-checkpoint schema participate in the B/R/L merge. Full local PROJECT/CURRENT/TASKS files, indexes, DEC/EXP source records, raw evidence, local sessions/research, and handoff/transport state remain outside the Hub merge domain.

Compatible changes from R and L are preserved. Unresolved conflicts on the same shared semantic field fail closed rather than being silently overwritten.

### 11. Identity conflicts fail closed

Workspace/project binding mismatches must not be automatically merged or rewritten. They require explicit review.

### 12. Detailed implementation choices remain later milestones

This ADR intentionally does not fix:

- the exact Hub sync-metadata file/schema;
- local multi-session safe-write mechanics;
- exact Hub publication triggers;
- executable reconciliation implementation details;
- privacy-detection implementation;
- startup, retrieval, probe, and compaction budgets.

Those belong to M1.3+ / M3+ and related validation milestones.

## Consequences

### Positive

- Local and Hub memory no longer need to compete for the same semantic authority.
- The mature local memory core can be reused rather than reimplemented.
- Remote Hub state can remain compact and useful for cross-agent/device recovery.
- Future lazy remote reads become architecturally possible because local active state and shared checkpoint state have distinct roles.
- Reconciliation can operate on a small semantic schema instead of attempting dangerous full-tree synchronization.

### Costs / constraints

- A Hub adapter must implement explicit projection, classification, provenance, and reconciliation semantics.
- A later migration is required because current Hub `0.1.0` still treats remote `PROJECT.md` as authoritative current project state.
- Web/new-device agents can recover only the last shared checkpoint, not unsynchronized local work.
- Local multi-session concurrency remains unsolved by this ADR and must be addressed separately.

## Rejected alternatives

- Make the Hub globally authoritative for all current/local project state.
- Make local `.ai/` globally authoritative and reduce the Hub to an unstructured mirror.
- Bidirectionally synchronize entire local memory files with Hub files.
- Copy every EXP/DEC/task/session into the Hub automatically.
- Treat existing transport CURRENT snapshots as authoritative Hub checkpoint state.
- Allow omission/compaction to imply deletion.
- Allow P3 or unclassified data to enter Project Memory promotion.

## Evidence and review

- `docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md`
- `docs/proposals/UNIFIED_AUTHORITY_MODEL.md`
- `docs/reviews/2026-08-26-unified-authority-model-review.md`
- `docs/reviews/2026-08-26-unified-authority-model-final-approval.md`
- `tests/results/v0.1.0-static-regression.md`

The final approval accepted R1–R12, approved M1.1 and M1.2, and identified no blocking defects.
