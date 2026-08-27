---
document_role: architecture-final-approval-review
status: approved
milestone: M1.7
reviewed: 2026-08-27
proposal: docs/proposals/NORMAL_FAST_CHANGED_REMOTE_RECOVERY_PATHS.md
review_baseline_commit: df290fc9489790a1d20a209e3bf16c4e0766e90a
reviewed_proposal_git_blob_lf_sha256: ED5BC5ECD399960A2469EB9D24AF53264E06F01AFA0911928C2897E82294C085
review_provenance: independent final-review result relayed by user
---

# M1.7 final approval — Normal fast / changed-remote / recovery paths

## Result

The final review approved M1.7 against the frozen revised proposal baseline `df290fc9489790a1d20a209e3bf16c4e0766e90a` and Git-blob LF SHA-256 `ED5BC5ECD399960A2469EB9D24AF53264E06F01AFA0911928C2897E82294C085`.

- FP1–FP14: **APPROVE**
- B1 — route-selection context TOCTOU: **CLOSED**
- blocking defects: **NONE**
- required changes before ADR promotion: **NONE**
- M1.7: **APPROVE**

The approved architecture covers the three-path selector only: normal fast path, changed-remote path, and recovery path. It includes exact trusted probe semantics, bound `route_selection_context`, pre-adoption authoritative-context revalidation, discard-and-restart on drift, exact changed-remote fetch plus ADR-0003 reconciliation, base-state and absence routing, unknown-commit/finalization recovery, degraded local-only continuation, no transport fallback, and ADR-0005 privacy-before-write ordering.

## Governance consequence

M1.7 may proceed through governance closure without further semantic changes: promote ADR-0006, mark the proposal `approved-proposal / accepted-unreleased`, and mark roadmap M1.7 `DONE` with proposal/review/ADR evidence.

## Maturity boundary

Architecture approval does not imply executable behavior or validation. ADR-0006 must retain:

```text
runtime_effective: false
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
```

M1.7 `DONE` does not mean probe, adapter, recovery runtime, synchronization runtime, migration, static validation, or live validation has been implemented or completed.

M1.8 remains the next bounded architecture milestone.