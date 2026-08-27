---
document_role: architecture-decision-record
adr_id: ADR-0006
status: accepted
accepted: 2026-08-27
milestone: M1.7
runtime_effective: false
release_required_for_runtime_effect: true
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
proposal: docs/proposals/NORMAL_FAST_CHANGED_REMOTE_RECOVERY_PATHS.md
final_approval: docs/reviews/2026-08-27-normal-fast-changed-remote-recovery-final-approval.md
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
  - docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md
  - docs/decisions/ADR-0004-hub-adapter-and-transport-boundary.md
  - docs/decisions/ADR-0005-privacy-never-publish-policy.md
---

# ADR-0006 — Normal fast / changed-remote / recovery paths

## Status

**Accepted architecture decision.**

This ADR records the approved M1.7 route-selection semantics for startup and synchronization-state handling in the future unified Project Memory architecture.

It is normative for later unified-architecture design and implementation, but it does **not** change current released Skill behavior, Runtime Hub runtime behavior, local Project Memory behavior, adapter behavior, or transport behavior. A later implementation, validation, compatibility/migration work, and release are required before these semantics become runtime-effective.

Current maturity remains:

```text
runtime_effective: false
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
```

## Context

ADR-0003 defines authoritative B/R/L reconciliation, observation versus reconciliation, base states, CAS, confirmation, and guarded local finalization. ADR-0004 keeps transport separate from Hub-adapter authority. ADR-0005 requires current privacy approval before write-capable publication.

M1.7 composes those accepted semantics into exactly three startup/synchronization routes: normal fast path, changed-remote path, and recovery path. Initial review found one blocking defect, B1 route-selection context TOCTOU. The clarification bound route inputs and exact probe evidence into a `route_selection_context` and required pre-adoption revalidation. Final review approved FP1–FP14, closed B1, and found no remaining blocking defects.

## Decision

FP1–FP14 from `docs/proposals/NORMAL_FAST_CHANGED_REMOTE_RECOVERY_PATHS.md` are accepted in full.

### 1. Exactly three operating paths

The architecture has exactly:

```text
NORMAL_FAST_PATH
CHANGED_REMOTE_PATH
RECOVERY_PATH
```

Restart/re-selection is control flow around these paths, not a fourth path and not synchronization success.

### 2. Route selection uses trusted typed inputs

The selector distinguishes canonical binding, exact adapter target, trusted base state, schema/capability context, unresolved operation-recovery state, and exact probe evidence. Unknown, unsupported, missing, ambiguous, mismatched, or invalid inputs do not default to success.

Trusted base states remain those defined by ADR-0003:

```text
BASE_READY_PRESENT
BASE_READY_ABSENT
BASE_UNINITIALIZED
BASE_INVALID
```

`BASE_UNINITIALIZED` and `BASE_INVALID` are not weak forms of a trusted base.

### 3. Route-selection evidence is bound against TOCTOU

Each tentative route decision binds at least:

```text
route_selection_context {
    canonical_binding_identity
    exact_adapter_target_identity
    BaseState
    base_pair_token
    reconciled_sha
    schema_context_identity
    capability_context_identity
    OperationRecoveryState_identity_or_version
    exact_probe_target_identity
    exact_probe_result_identity_or_state
}
```

Before a tentative route is adopted, authoritative local inputs are revalidated against this captured context.

If binding, target, base, base-pair, schema/capability, or operation-recovery context has drifted, both the old route and its probe evidence become stale. They are discarded and route selection restarts from current inputs.

Old probe evidence after such drift must not be used to skip a full fetch, enter remote action, advance markers, or claim unchanged/current/reconciled status.

### 4. Fast path requires an exact trusted unchanged probe

Full checkpoint fetch may be skipped only when all fast-path prerequisites are current and trusted, including valid binding, trusted base, supported schema/capability, clear unresolved-operation state, an exact probe against the exact checkpoint target, and successful pre-adoption route-context revalidation.

For a present base, the exact remote checkpoint identity must match `reconciled_sha`. For a trusted absent base, exact observation must confirm the checkpoint remains absent.

Probe failure, timeout, unavailable, unknown, invalid, repository-global HEAD, transport SHA, cached timestamp, path state, or an old probe captured under stale local context never means unchanged.

The fast path may skip the full remote checkpoint fetch for that check. It does not create a new reconciliation event or advance `B/reconciled_sha`.

### 5. Changed remote requires exact fetch plus ADR-0003 reconciliation

When a trusted current route context shows a different present remote checkpoint, the path is:

```text
exact changed observation
  -> exact checkpoint fetch
  -> R + R_sha
  -> current local shared projection L relative to captured B
  -> ADR-0003 reconciliation
```

Probe output is observation evidence, not merge content. Full fetch is not reconciliation. `observed_sha` is not `reconciled_sha`.

If an outbound candidate requires remote mutation, current ADR-0005 privacy `ALLOW` for the exact candidate/bundle/destination is required before ADR-0003 write-capable CAS.

### 6. Base states and remote absence route deterministically

- `BASE_READY_PRESENT` + exact same present remote may use fast path.
- `BASE_READY_ABSENT` + exact remote absent may use fast path.
- `BASE_READY_ABSENT` + exact remote present uses changed-remote exact fetch/reconciliation.
- `BASE_READY_PRESENT` + exact remote absent enters recovery as `REMOTE_CHECKPOINT_ABSENT_CONFLICT` unless a separately accepted deletion/tombstone policy authorizes the transition.
- `BASE_UNINITIALIZED`, whether remote present or absent, remains base-required and never auto-adopts the newest/current remote state.
- `BASE_INVALID` enters recovery and is not repaired from transport snapshots/history, guessed SHAs, or current remote state.

### 7. Recovery is fail-closed and not synchronization success

Recovery handles missing/ambiguous/mismatched binding, base-required/invalid states, probe uncertainty, unsupported/unknown schema or capability, remote checkpoint disappearance conflicts, ambiguous remote commit/confirmation, local finalization failed/unknown, and route-context drift that cannot immediately re-establish a trusted selector context.

Recovery may gather authorized read-only evidence. Such evidence does not itself establish reconciliation.

When trustworthy evidence changes, old route context/probe evidence is discarded and selection restarts against current inputs. If trusted context cannot be restored, the system remains in recovery.

### 8. Unknown commit/confirmation and local finalization remain ADR-0003 recovery states

Ambiguous remote commit or confirmation requires exact re-observation; there is no blind retry, force overwrite, rollback, or marker advancement.

Remote acceptance/write does not make the local instance reconciled if guarded local finalization failed or is unknown. The persistent old canonical local base remains authoritative until ADR-0003-compatible recovery/finalization succeeds.

### 9. Degraded local-only continuation is permitted but bounded

Recovery may permit explicit local-only continuation when local memory/evidence is trustworthy. It may continue local reasoning/work and local-safe writes where otherwise permitted.

It must not:

- perform remote checkpoint writes or CAS;
- advance `B`, `reconciled_sha`, or `base_pair_token`;
- claim remote state is current, unchanged, synchronized, or reconciled;
- treat transport/cached state as remote truth;
- reuse stale route/probe evidence;
- silently publish accumulated work when trust/connectivity returns without fresh route selection and privacy evaluation.

This is a mode inside recovery, not a fourth path.

### 10. Transport remains outside adapter fallback authority

ADR-0004 continues to govern transport. Mailbox messages, receipts, snapshots, transport SHAs, or histories cannot substitute for checkpoint probe, trusted base, recovery lineage, route-context repair, or Hub adapter write paths.

### 11. Marker and result claims remain disciplined

Trusted probe/fetch may update observation state only under ADR-0003 observation semantics.

Only ADR-0003 guarded local finalization may advance canonical `B/reconciled_sha`.

Route results describe routing decisions, not synchronization completion.

### 12. Implementation details remain deferred

This ADR does not choose or implement M1.8 budgets, probe/fetch API, Git/GitHub API, sync-metadata physical schema, locks/Windows/filesystem primitives, retry values, adapter module layout, credentials implementation, publication triggers, detector implementation, migration, UI, real-data operations, or static/live validation.

## Consequences

The architecture can safely skip unchanged remote full reads only under exact trusted evidence bound to current local authority context. Remote changes, uncertain state, missing lineage, and recovery conditions are routed without inventing synchronization success or weakening ADR-0003/0004/0005 boundaries.

The cost is that a future implementation must maintain enough route-context identity to detect local TOCTOU drift and must expose recovery/degraded results instead of optimistic fallback.

## Rejected alternatives

- Treat probe unavailable/unknown as unchanged.
- Use repository-global HEAD, transport SHA, cached time/path state, or stale probe evidence as checkpoint identity.
- Reuse an old unchanged/changed probe after binding/base/schema/capability/recovery-state drift.
- Auto-adopt newest remote under `BASE_UNINITIALIZED` or repair `BASE_INVALID` from transport/current remote.
- Treat recovery, re-selection, remote CAS success, or remote write as reconciliation success.
- Use transport as adapter fallback.
- Let degraded local-only continuation mutate remote checkpoint state or reconciliation markers.

## Evidence and review

- `docs/proposals/NORMAL_FAST_CHANGED_REMOTE_RECOVERY_PATHS.md`
- `docs/reviews/2026-08-27-normal-fast-changed-remote-recovery-final-approval.md`
- initial baseline: `a83b76b8aa3d5d2392f46c3ed60529a19070c31e`
- revised final-review baseline: `df290fc9489790a1d20a209e3bf16c4e0766e90a`
- revised proposal Git-blob LF SHA-256: `ED5BC5ECD399960A2469EB9D24AF53264E06F01AFA0911928C2897E82294C085`

Final review approved FP1–FP14, closed B1, approved M1.7, and required no further semantic changes.