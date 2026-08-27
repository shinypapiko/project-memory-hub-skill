---
document_role: architecture-decision-record
adr_id: ADR-0003
status: accepted
accepted: 2026-08-27
milestone: M1.4
runtime_effective: false
release_required_for_runtime_effect: true
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
proposal: docs/proposals/REMOTE_REFRESH_RECONCILIATION_STATE_MACHINE.md
initial_review: docs/reviews/2026-08-27-remote-refresh-reconciliation-review.md
final_approval: docs/reviews/2026-08-27-remote-refresh-reconciliation-final-approval.md
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
---

# ADR-0003 — Remote refresh and reconciliation state machine

## Status

**Accepted architecture decision.**

This ADR records the approved M1.4 semantics for deterministic remote shared-checkpoint refresh and reconciliation in the future unified Project Memory architecture.

It is normative for subsequent unified-architecture design and implementation work, but it does **not** change current Hub Skill `0.1.0`, Runtime Hub schema-1 behavior, or the deployed local Skill. Runtime behavior requires later implementation, tests, migration/compatibility work, and release.

## Context

`ADR-0001` established that the future Hub `projects/<project-id>/PROJECT.md` is a shared reconciled checkpoint rather than a mirror of local `.ai` memory. It also established semantic B/R/L reconciliation inside the shared-checkpoint schema.

M1.4 makes that principle deterministic under concurrent remote writes and concurrent local sessions. The initial review identified four defects: incomplete explicit same-field resolution, missing canonical local-base drift semantics, under-specified post-CAS/local-finalization states, and incomplete base-loss/ABSENT handling. The revised proposal closed B1–B4; final review approved RR1–RR14 with no new blocking defects.

## Decision

RR1–RR14 from `docs/proposals/REMOTE_REFRESH_RECONCILIATION_STATE_MACHINE.md` are accepted in full.

### 1. Canonical reconciled base is a guarded local pair

The local representation of the last fully reconciled shared checkpoint is conceptually:

```text
P = (B, reconciled_sha)
```

with an opaque local concurrency identity:

```text
base_pair_token
```

`B` and `reconciled_sha` are inseparable. A SHA without trustworthy matching `B` content is not a valid reconciled base.

Another local session may advance this pair while a remote operation is in flight. Such drift invalidates the old candidate and requires current local shared intent to be projected again against the new base.

### 2. Remote observation is not reconciliation

`observed_sha` records only a trusted observation of the remote checkpoint state. It may be newer than, older than only in stale metadata scenarios that must be corrected by fresh observation, or simply different from `reconciled_sha` during normal unreconciled remote progress.

A newly observed SHA never becomes `reconciled_sha` merely because it is current or newer.

`observed_sha = ABSENT` means only that trusted observation found no current remote checkpoint; it does not imply a trusted initialized absent base.

### 3. Reconciliation inputs are B/R/L and candidate C

For one operation:

- `B` is the captured reconciled base from canonical local pair `P`;
- `R` is one exact loaded remote checkpoint revision `R_sha`;
- `L` is an immutable normalized local shared-checkpoint proposal projected relative to that captured `B`;
- `C` is the candidate produced by automatic three-way reconciliation or an explicit resolution operation.

A candidate is bound to the exact context:

```text
(base_pair_token, reconciled_sha, R_sha, L identity, resolution context if any)
```

Changing any dependency invalidates the candidate. Old candidate bytes are never replayed by merely substituting a newer SHA or local token.

### 4. Automatic per-field reconciliation is deterministic

Each shared semantic field is compared against `B`:

- neither side changed → `UNCHANGED`;
- remote only changed → `REMOTE_ONLY`;
- local only changed → `LOCAL_ONLY`;
- both changed to the same semantic value → `CONVERGENT_BOTH`;
- both changed differently → `FIELD_CONFLICT`.

Automatic reconciliation never silently chooses local-wins or remote-wins for divergent same-field edits. Unknown schema, identity mismatch, unavailable field merge semantics, or unauthorized checkpoint disappearance fail closed.

All assembled candidates must also pass whole-checkpoint schema and cross-field validation.

### 5. Divergent field conflicts use an explicit context-bound resolution operation

A conflict produces a bound conflict context identifying the exact local base, remote revision, local proposal, conflict fields, and safe identities/hashes of the B/R/L field states.

An explicit resolution `X` may select per conflict field:

- `TAKE_REMOTE`;
- `TAKE_LOCAL`;
- `SET_EXPLICIT(value_or_state)` for a schema-valid third state.

The resolution is valid only for that exact conflict context. If the canonical local base or remote revision changes, the resolution becomes `CONFLICT_RESOLUTION_STALE` and must not be replayed automatically.

### 6. Local base drift is checked before candidate use and at finalization

Before a no-write acceptance or remote CAS, the operation re-checks that canonical local `base_pair_token/reconciled_sha` still equal the captured base.

After a remote commit, canonical local advancement is again guarded by the captured base-pair token.

Any drift terminates the old operation. Current local intent must be re-projected against the new canonical base; old `L` is not reinterpreted against a different `B`.

### 7. Remote writes use CAS against the exact classified remote revision

If `C != R`, the remote write uses compare-and-set against the exact `R_sha` from which `C` was computed/resolved.

`CAS_STALE` invalidates the old candidate. The system must fetch the newer remote checkpoint and recompute from the still-valid captured `B + Rnew + L`; it must not retry old candidate bytes against the newer SHA.

Remote stale/contention retry is bounded. Exhaustion returns a structured contention result and never falls back to force overwrite or last-writer-wins.

### 8. Ambiguous remote commits are re-observed

A timeout/error that leaves commit state uncertain does not advance reconciliation and does not authorize blind retry or rollback.

Fresh remote observation determines whether:

- the validated candidate is now exactly present and may proceed to local finalization;
- the new remote state is compatible and requires recomputation;
- the new remote state conflicts;
- the commit state remains unknowable.

Unresolved ambiguity returns a distinct unknown state and does not advance `reconciled_sha`.

### 9. Exact committed revision and current remote head are separate facts

After a remote CAS reports success, the exact committed revision `S` must be confirmable as containing candidate `C` before reconciliation may finalize locally.

The remote head may already have advanced to `T` through another writer. It is valid to represent:

```text
reconciled_sha = S
observed_sha = T
```

when `S` is the confirmed reconciled candidate and `T` is later unreconciled remote work.

The protocol must not skip directly to `T` merely because it is current head.

If exact candidate revision cannot be confirmed, return a confirmation-unknown state and do not advance the canonical local base.

### 10. Remote commit and persistent local reconciliation are separate stages

A remote CAS success alone is not reconciliation success for this local Project Memory instance.

After a no-write accepted remote state or a confirmed remote commit, canonical local pair advancement is a guarded local state transition:

```text
LOCAL_ADVANCE(expected_base_pair_token, new_B, new_reconciled_sha)
```

Only successful persistence of the new canonical pair permits a `RECONCILED_*` success result.

Definite persistence failure, ambiguous persistence, or local-base drift have distinct results and may not be collapsed into success.

If the process crashes after remote commit but before local finalization, the old persistent local base remains authoritative for reconciliation bookkeeping. On recovery, the remote commit is treated as unreconciled remote input relative to that still-persistent base; it is not auto-adopted because the same process may have written it.

### 11. Base trust and ABSENT states are explicit

The protocol distinguishes:

- `BASE_READY_PRESENT` — trustworthy present `B/reconciled_sha` pair;
- `BASE_READY_ABSENT` — explicitly initialized trusted absent lineage with `B=ABSENT` and `reconciled_sha=ABSENT`;
- `BASE_UNINITIALIZED` — no trustworthy lineage; returns `RECONCILIATION_BASE_REQUIRED` whether the current remote is present or absent;
- `BASE_INVALID` — missing/mismatched/corrupt B-to-SHA linkage; returns `RECONCILIATION_BASE_INVALID`;
- previously reconciled present checkpoint observed as absent — `REMOTE_CHECKPOINT_ABSENT_CONFLICT` unless a separate accepted deletion/tombstone policy authorizes the transition.

The current remote being absent is never enough to infer a first-creation base.

### 12. Marker advancement fails closed

`observed_sha` advances only from trusted remote observation semantics.

Canonical `B/reconciled_sha` advances only through successful guarded local finalization of a remote state that has already passed deterministic reconciliation and, where written, exact revision confirmation.

Conflict, stale CAS, unknown commit, unknown confirmation, local-base drift, local finalization failure/ambiguity, base-loss, and checkpoint disappearance states never advance `reconciled_sha`.

## Consequences

### Positive

- A newly observed remote revision cannot masquerade as a reconciled state.
- Remote stale CAS cannot become lost-update by replaying an old candidate.
- Same-field conflicts can be resolved explicitly without weakening automatic fail-closed behavior.
- Concurrent local sessions cannot silently finalize a remote operation against a base that another local session already superseded.
- Remote commit success, remote-head movement, and local reconciliation bookkeeping remain distinct and recoverable.
- Crash recovery is conservative without requiring an implementation-specific transaction journal.
- Missing or ambiguous base lineage is surfaced rather than repaired by inventing a convenient base.

### Costs / constraints

- A future Hub adapter must persist enough canonical local base information to verify B-to-SHA linkage and provide a base-pair concurrency token.
- It must support structured conflict and resolution contexts rather than a simple file overwrite workflow.
- Remote CAS must provide or be combined with sufficient exact-revision confirmation semantics.
- A successful remote write can still leave local reconciliation incomplete if local finalization fails; later recovery must handle that state explicitly.
- Web/new-device bootstrap must establish trustworthy base lineage through later M1.7/M3 policy rather than automatically adopting the newest remote checkpoint.

## Rejected alternatives

- Copy `observed_sha` into `reconciled_sha` because it is newer/current.
- Treat remote CAS success as sufficient local reconciliation completion.
- Retry a stale CAS by replacing only the expected SHA and reusing old candidate bytes.
- Let ordinary automatic merge silently choose local-wins or remote-wins for divergent same-field edits.
- Replay an explicit resolution after its bound B/R/L context changed.
- Reinterpret an old local proposal against a newly advanced canonical base.
- Skip directly to a newer remote head that appeared after the exact reconciled candidate revision.
- Treat current remote absence as proof of a first-time absent baseline.
- Substitute latest remote content for missing/corrupt local base lineage.
- Blindly roll remote state back after an ambiguous commit.

## Scope and maturity

This ADR fixes semantic/protocol architecture only.

It deliberately does not select:

- concrete Hub adapter code;
- GitHub/API primitives;
- retry/backoff numbers;
- publication triggers;
- privacy detection/implementation;
- migration;
- startup fast-path policy;
- sync-metadata file format;
- durable in-flight operation journal design.

Maturity remains:

```text
runtime_effective: false
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
```

## Evidence and review

- `docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md`
- `docs/decisions/ADR-0002-local-multi-session-safe-write.md`
- `docs/proposals/REMOTE_REFRESH_RECONCILIATION_STATE_MACHINE.md`
- `docs/reviews/2026-08-27-remote-refresh-reconciliation-review.md`
- `docs/reviews/2026-08-27-remote-refresh-reconciliation-final-approval.md`
- `docs/concurrency.md`
- `tests/results/v0.1.0-static-regression.md`

Final review approved RR1–RR14, closed B1–B4, approved M1.4, and identified no new blocking defects.