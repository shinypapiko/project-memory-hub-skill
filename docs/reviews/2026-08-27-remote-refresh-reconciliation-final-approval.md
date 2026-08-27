---
document_role: architecture-final-approval-record
normative: false
runtime_load_policy: maintenance-only
reviewed_document: docs/proposals/REMOTE_REFRESH_RECONCILIATION_STATE_MACHINE.md
review_date: 2026-08-27
review_state: final-approved
milestone: M1.4
initial_review: docs/reviews/2026-08-27-remote-refresh-reconciliation-review.md
---

# Final approval — Remote refresh/reconciliation state machine, M1.4

## Outcome

- RR1: APPROVE
- RR2: APPROVE
- RR3: APPROVE
- RR4: APPROVE
- RR5: APPROVE
- RR6: APPROVE
- RR7: APPROVE
- RR8: APPROVE
- RR9: APPROVE
- RR10: APPROVE
- RR11: APPROVE
- RR12: APPROVE
- RR13: APPROVE
- RR14: APPROVE
- B1: CLOSED
- B2: CLOSED
- B3: CLOSED
- B4: CLOSED
- M1.4: APPROVED
- New blocking defects: none

## Accepted closure

The revised proposal closes the initial review defects without reopening M1.1–M1.3.

Accepted semantics include:

- context-bound explicit conflict resolution `X` supporting `TAKE_REMOTE`, `TAKE_LOCAL`, and schema-valid explicit third values while rejecting stale replay;
- canonical local `base_pair_token` drift detection before candidate use and again during guarded local finalization, with mandatory fresh projection after drift;
- exact post-CAS candidate-revision confirmation distinct from current remote-head observation;
- deterministic handling of a newer remote head, local finalization failure/ambiguity, and crash after remote commit but before local base advancement;
- strict separation of `observed_sha` from `reconciled_sha`;
- distinct present, trusted initialized-ABSENT, uninitialized, invalid-base, and previously-present-to-ABSENT states.

## Maturity boundary

This approval is architecture/protocol approval only.

It does **not** imply an executable implementation or live validation. The promoted ADR must retain:

```text
runtime_effective: false
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
```

Concrete APIs, adapter code, retry constants, publication triggers, migration, privacy implementation, and startup fast-path behavior remain outside M1.4.

## Promotion decision

M1.4 is eligible for ADR promotion with no further semantic changes.