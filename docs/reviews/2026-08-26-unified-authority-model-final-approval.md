---
document_role: architecture-final-approval-record
normative: false
runtime_load_policy: maintenance-only
reviewed_document: docs/proposals/UNIFIED_AUTHORITY_MODEL.md
review_date: 2026-08-26
review_state: final-approved
milestones:
  - M1.1
  - M1.2
---

# Final approval — Unified Authority Model, M1.1 / M1.2

## Outcome

- R1: APPROVE
- R2: APPROVE
- R3: APPROVE
- R4: APPROVE
- R5: APPROVE
- R6: APPROVE
- R7: APPROVE
- R8: APPROVE
- R9: APPROVE
- R10: APPROVE
- R11: APPROVE
- R12: APPROVE
- M1.1: APPROVE
- M1.2: APPROVE
- Blocking defects: none

No further substantive architecture changes are required before ADR promotion.

## Approval scope

This approval covers only the authority model and field-level local/Hub boundary defined by M1.1 and M1.2. It does not approve or define:

- local multi-session safe-write mechanics (M1.3);
- executable remote B/R/L reconciliation details (M1.4);
- exact Hub publication triggers;
- Hub adapter implementation details;
- privacy-classification detection implementation;
- startup/token budgets;
- migration or release behavior.

Those remain later milestones.

## Approved decision set

The final review accepts R1–R12 exactly as enumerated in `docs/proposals/UNIFIED_AUTHORITY_MODEL.md`, including:

- semantic-scope-based authority rather than globally local-first or Hub-first authority;
- `.ai/PROJECT.md` as stable detailed local background and `.ai/CURRENT.md` as active local work;
- future Hub `PROJECT.md` as the last reconciled shared checkpoint, not a mirror of the local memory tree;
- distinct local retrieval indexes and Hub project-routing registry;
- `AGENTS.md` as a small invocation/binding layer only;
- local-by-default detailed EXP/DEC/task/session/research/handoff records with selective derived promotion;
- remote hydration that does not overwrite local PROJECT/CURRENT wholesale;
- classification/privacy/provenance/minimization/pre-write-refresh gates for promotion;
- fail-closed identity conflicts and scoped B/R/L semantic reconciliation;
- policy inheritance that cannot weaken prohibitions, provenance floors, privacy restrictions, or retention floors;
- explicit deletion semantics: supersession, compaction, and omission are not deletion;
- deferral of exact sync metadata, local concurrent-write mechanics, publication triggers, privacy-detection implementation, and performance budgets to later milestones.

## Evidence chain

- `docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md`
- `docs/proposals/UNIFIED_AUTHORITY_MODEL.md`
- `docs/reviews/2026-08-26-unified-authority-model-review.md`
- this final approval record
- `tests/results/v0.1.0-static-regression.md`

## Promotion instruction

Create a dedicated ADR/decision record for the accepted M1.1/M1.2 architecture. The proposal remains non-normative and must not be converted into released runtime behavior by this approval alone.

Current `skill/SKILL.md`, `docs/architecture.md`, Runtime Hub schema-1 behavior, and the deployed local Skill remain unchanged until an explicit later release/migration changes them.
