---
document_role: architecture-review-record
normative: false
runtime_load_policy: maintenance-only
reviewed_document: docs/proposals/LOCAL_MULTI_SESSION_SAFE_WRITE.md
review_date: 2026-08-26
review_state: revisions-required
milestone: M1.3
---

# Review record — Local multi-session safe-write, M1.3

## Outcome

- L1: APPROVE
- L2: REVISE
- L3: REVISE
- L4: APPROVE
- L5: REVISE
- L6: APPROVE
- L7: REVISE
- L8: REVISE
- L9: REVISE
- L10: APPROVE
- L11: APPROVE
- L12: APPROVE
- M1.3: not yet approved

## Blocking defects

1. **B1 — Cooperative-writer safety boundary and canonical resource identity are unspecified.**
2. **B2 — Retry/multi-lock acquisition has no bounded liveness and release-on-failure contract.**
3. **B3 — Windows replace failure and new-file crash state are not deterministically classified.**
4. **B4 — INDEX repairability is stated too broadly.**
5. **B5 — Multi-resource partial completion has no structured result/recovery contract.**

## Required clarifications before approval

1. No-lost-update guarantees apply only to participating/cooperative writers that use the same safe-write helper. External direct writes remain detectable by content identity where possible but are outside the lock-coordination guarantee.
2. Every guarded resource must resolve to one canonical lock identity. Windows path/case aliases, 8.3 aliases, hard links, reparse points/junctions, or other ambiguous aliases must be resolved safely or cause fail-closed behavior.
3. Automatic stale/contention retries must be bounded. Exhaustion must return a structured contention result rather than retry indefinitely.
4. Multi-lock acquisition must sort by canonical lock identity. If any acquisition fails, release every lock already acquired, then back off/retry outside all locks.
5. New DEC/EXP records must be fully prepared and validated before an atomic no-replace installation. A crash/failure must not leave an unmarked partial authoritative record.
6. If atomic replace/install returns an error or ambiguous platform result, re-read the target and classify the actual state. If the helper cannot prove committed-versus-not-committed, return `COMMIT_STATE_UNKNOWN`; never degrade to truncating/in-place overwrite.
7. Distinguish reconstructible index entries from curated routing/content. Only reconstructible entries may be treated as derivable/repairable by source scan.
8. Multi-resource operations require `PARTIAL_COMMIT` (or equivalent), per-resource results, and a no-blind-rollback rule. Recovery should complete/repair forward from observed state unless an explicit reversible operation is proven safe.

## Scope boundary

These revisions remain entirely inside M1.3. They do not define Hub remote reconciliation, publication triggers, migration, or implementation code.

## Approval condition

After the proposal consistently integrates the eight clarifications above, perform a final L1–L12 review. M1.3 may be approved only when the blocking defects are closed and the revised decision set is accepted.