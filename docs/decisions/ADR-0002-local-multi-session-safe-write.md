---
document_role: architecture-decision-record
adr_id: ADR-0002
status: accepted
accepted: 2026-08-27
milestone: M1.3
runtime_effective: false
release_required_for_runtime_effect: true
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
proposal: docs/proposals/LOCAL_MULTI_SESSION_SAFE_WRITE.md
initial_review: docs/reviews/2026-08-26-local-multi-session-safe-write-review.md
final_approval: docs/reviews/2026-08-27-local-multi-session-safe-write-final-approval.md
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
---

# ADR-0002 — Local multi-session safe-write semantics

## Status

**Accepted architecture decision.**

This ADR records the approved M1.3 semantics for safe same-workspace writes by multiple cooperative Codex/agent sessions.

It is normative for subsequent unified-architecture design and implementation work, but it does **not** by itself change the current deployed local Project Memory Skill, current Hub Skill `0.1.0`, or Runtime Hub behavior. A later implementation/release must adopt and validate these semantics before they become runtime guarantees.

## Context

A simple "re-read immediately before write" rule reduces stale writes but does not close the time-of-check/time-of-use race. Two sessions may both refresh the same current state and then sequentially overwrite each other.

M1.3 therefore requires a local concurrency contract that remains usable even when the workspace is not a Git worktree and that does not hold locks while an LLM reasons, calls tools, or waits for user interaction.

The first M1.3 review identified five blocking defects around resource identity, bounded liveness, atomic commit-state classification, index repairability, and partial multi-resource completion. The revised proposal closed B1–B5, and final review approved L1–L12 with no new blocking defects.

## Decision

L1–L12 from `docs/proposals/LOCAL_MULTI_SESSION_SAFE_WRITE.md` are accepted in full.

### 1. Safety guarantee is cooperative-writer scoped

The no-lost-update guarantee applies to writers that use the same released safe-write helper/protocol and therefore share canonical resource identity, lock, compare-and-write, and atomic-install semantics.

Direct external writes by editors, scripts, sync clients, or older incompatible Skills are outside the mutual-exclusion guarantee. Exact-byte re-observation may detect many such mutations, but the system must not claim lock safety against non-cooperative writers.

### 2. Every guarded resource has one canonical identity

All cooperative writers targeting the same semantic filesystem object must derive one canonical resource identity and lock key.

Path aliases that cannot be proven identical or distinct safely—including relevant Windows case aliases, 8.3 aliases, hard links, junction/reparse/symlink/mount aliases—must fail closed rather than create independent locks for the same object.

Resource resolution must remain inside the verified workspace/local-memory boundary.

### 3. Optimistic work, serialized short commit

Normal work remains lock-free. A writer records an exact expected-base content token, normally SHA-256 of the bytes read.

At commit time:

1. resolve canonical resource identity;
2. acquire the short resource lock;
3. re-read exact current bytes while locked;
4. compare current identity to the expected base;
5. validate and atomically commit only when they match;
6. otherwise return `STALE_LOCAL` and release the lock.

Semantic reconciliation, model reasoning, backoff, and user interaction occur outside commit locks.

### 4. Retry is bounded

Automatic stale/contention retry must use an explicit bounded attempt/time budget. Exhaustion returns a structured result such as `CONTENTION_EXHAUSTED`; the helper must not spin indefinitely.

### 5. Existing-file commits require atomic replace semantics

An existing guarded file is prepared in a same-directory staging file, completed and validated, then installed by a proven atomic replacement primitive.

A failed atomic replace must never degrade to truncating or overwriting the target in place.

If the underlying operation reports failure or ambiguity, the helper re-observes the target. When commit state cannot be proven, it reports `COMMIT_STATE_UNKNOWN` rather than guessing.

### 6. New authoritative records require atomic no-replace installation

New DEC/EXP/source records must be fully prepared, flushed/closed, and validated before becoming authoritative. Final installation must use an atomic no-replace operation or equivalent that fails on collision.

A staging artifact left by a crash must remain distinguishable from a valid authoritative record.

### 7. Sequential DEC/EXP IDs require namespace reservation

Sequential ID allocation is guarded by a short namespace lock or equivalent atomic reservation. The allocator re-scans current authoritative IDs while protected, selects the next ID, and installs the complete record collision-safely.

An index failure after source-record creation does not justify deleting the source record.

### 8. Indexes have reconstructible and curated semantics

Indexes are not uniformly disposable.

- `RECONSTRUCTIBLE` material can be deterministically rebuilt from authoritative source records.
- `CURATED` routing/grouping/annotation/order/scope/alias content is semantic state and must be preserved/rebased like other mutable state.

Repair must never discard curated content merely because part of an index is reconstructible.

### 9. Multi-lock acquisition uses canonical total ordering

When an operation truly requires simultaneous locks, unique canonical lock keys are sorted in one deterministic total order.

If acquisition of a later lock fails, every lock already acquired by that attempt is released. Backoff/retry happens outside all locks and remains bounded.

### 10. Initial design does not claim arbitrary multi-file transactions

The guarantee is per guarded resource, not arbitrary filesystem-atomic multi-file commit.

If a multi-resource operation partly completes, the overall result is `PARTIAL_COMMIT` and includes per-resource before/after/result state, including any `COMMIT_STATE_UNKNOWN` resources.

Already committed resources must not be blindly rolled back from stale saved snapshots. Recovery defaults to fresh observation plus forward repair; semantic conflicts are surfaced explicitly.

### 11. Git is not the local concurrency primitive

Git may provide provenance/history but is not required for local no-lost-update safety. The concurrency foundation is exact content identity, canonical resource locking, and atomic replace/no-replace semantics.

### 12. Ambiguity fails closed

The system must not claim safe local concurrency when lock ownership, resource identity, or filesystem atomicity semantics are unsupported or ambiguous. Stale lock reclamation requires proof under platform-specific rules rather than age alone.

## Structured outcomes required by later implementation

The executable helper is expected to provide structured outcomes including at least:

- `COMMITTED`
- `STALE_LOCAL`
- `LOCK_BUSY`
- `LOCK_AMBIGUOUS`
- `CONTENTION_EXHAUSTED`
- `VALIDATION_FAILED`
- `UNSUPPORTED_FILESYSTEM`
- `RESOURCE_IDENTITY_AMBIGUOUS`
- `CREATE_COLLISION`
- `COMMIT_STATE_UNKNOWN`
- `IO_ERROR`
- `PARTIAL_COMMIT` for multi-resource partial completion

Exact API/CLI representation remains an implementation decision.

## Consequences

Positive consequences:

- pre-write refresh is upgraded from a best-effort convention to a design that can close cooperative-writer TOCTOU lost updates;
- locks stay short and do not serialize entire agent tasks;
- the design works independently of Git worktree status;
- Windows/alias/partial-write ambiguity is surfaced rather than hidden;
- DEC/EXP allocation and curated index state receive explicit concurrency semantics;
- partial multi-resource completion becomes observable and recoverable without unsafe rollback claims.

Costs/constraints:

- all participating writers must converge on one safe-write helper/protocol for the guarantee to hold;
- supported filesystems/platforms need canonical-identity and atomicity validation;
- multi-file crash consistency remains weaker than a true journaled transaction;
- executable tests are required before runtime guarantees can be claimed.

## Deferred implementation choices

ADR-0002 intentionally does not choose:

- Python function/class/CLI names;
- concrete Windows filesystem APIs;
- numeric retry/backoff/time budgets;
- lock-key encoding details;
- platform-specific stale-lock recovery thresholds;
- optional future transaction journaling;
- richer automatic semantic merge algorithms.

These details must satisfy this ADR and later executable tests but are not prerequisites for architecture acceptance.

## Maturity

Current status of the **future unified M1.3 capability** at ADR acceptance:

- `implementation_status: PROTOCOL_ONLY`
- `validation_status: UNVALIDATED`

This ADR is an accepted architecture contract, not evidence of an executable or live-validated implementation.

## Evidence

- `docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md`
- `docs/proposals/LOCAL_MULTI_SESSION_SAFE_WRITE.md`
- `docs/reviews/2026-08-26-local-multi-session-safe-write-review.md`
- `docs/reviews/2026-08-27-local-multi-session-safe-write-final-approval.md`
