---
document_role: architecture-review-record
normative: false
runtime_load_policy: maintenance-only
reviewed_document: docs/proposals/REMOTE_REFRESH_RECONCILIATION_STATE_MACHINE.md
review_date: 2026-08-27
review_state: revisions-required
milestone: M1.4
---

# Review record — Remote refresh/reconciliation state machine, M1.4

## Outcome

- RR1: APPROVE
- RR2: APPROVE
- RR3: APPROVE
- RR4: REVISE
- RR5: REVISE
- RR6: APPROVE
- RR7: REVISE
- RR8: APPROVE
- RR9: APPROVE
- RR10: APPROVE
- RR11: APPROVE
- RR12: REVISE
- RR13: REVISE
- RR14: APPROVE
- M1.4: NOT APPROVED

## Blocking defects

1. **B1 — Explicit same-field conflict resolution is incomplete.** An ordinary replacement `L2` cannot terminate a divergent conflict with local-wins or a third value because the normal B/R/L matrix still classifies both sides as changed and different. M1.4 needs an explicit resolution operation bound to the exact conflict context and protected by remote CAS.
2. **B2 — Canonical local base-pair drift is not represented.** Another local session may advance the canonical `B/reconciled_sha` while an operation is in flight. Candidate use and local ADVANCE must detect that drift, invalidate the candidate, and require a fresh local projection against the new base.
3. **B3 — Post-CAS finalization is under-specified.** Deterministic semantics are required when the remote head advances after our commit, exact committed-revision confirmation fails, persisting the canonical local base pair fails or is ambiguous, or the process crashes after remote commit but before local advancement.
4. **B4 — Base-loss and remote-ABSENT states are incomplete.** The protocol must distinguish absent base metadata, missing/mismatched base content, trusted ABSENT baseline, and a previously existing reconciled remote checkpoint becoming absent.

## Required clarifications before approval

1. Add an explicit conflict-resolution operation that may choose remote, local, or a schema-valid third value, but is bound to exact B/R/L conflict context and may commit only through CAS against the exact remote revision it resolved.
2. Introduce a canonical local base-pair identity/version token. Candidate use, no-write acceptance, and post-commit ADVANCE must re-check it. Any drift invalidates the old candidate/resolution context and requires a new operation with local intent re-projected against the new canonical base.
3. Define post-CAS confirmation/finalization states, including: exact committed revision confirmed while remote head is already newer; confirmation unavailable/unknown; local base-pair persistence definite failure; local persistence ambiguous; local base-pair stale; and crash recovery when remote is ahead of the persistent reconciled base.
4. Make `B/reconciled_sha` advancement a guarded local state transition rather than an in-memory assignment. A remote CAS success alone must never imply persistent local reconciliation success.
5. Expand base safety to reject missing or mismatched B content, distinguish trusted initialized `ABSENT` from unknown lineage, and treat disappearance of a previously reconciled remote checkpoint as an explicit conflict/state transition rather than silently as first creation or ordinary omission.

## Scope boundary

These revisions remain semantic/protocol-level M1.4 work. They do not select concrete APIs, adapters, retry counts, publication triggers, migration behavior, privacy tooling, or startup fast-path policy.

## Approval condition

After the revised proposal closes B1–B4 and updates RR4/RR5/RR7/RR12/RR13 consistently, perform a final RR1–RR14 review. M1.4 may be approved only if no new blocking defect remains.