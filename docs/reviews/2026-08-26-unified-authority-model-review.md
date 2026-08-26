---
document_role: architecture-review-record
normative: false
runtime_load_policy: maintenance-only
reviewed_document: docs/proposals/UNIFIED_AUTHORITY_MODEL.md
review_date: 2026-08-26
review_state: clarifications-integrated-final-approval-pending
---

# Review record — Unified Authority Model, M1.1 / M1.2

## Review outcome

- Architecture direction: **acceptable**
- M1.1: proposal review may proceed
- M1.2: proposal review may proceed
- Approval / DONE: **not yet**
- Blocking defects: **none identified**

The review required six clarifications before approval:

1. roadmap status/evidence;
2. field-policy inheritance;
3. retention/deletion semantics;
4. P3 hard-block semantics;
5. default behavior for unclassified information;
6. exact reconciliation scope for the shared checkpoint.

## Clarifications adopted

### 1. Roadmap status and evidence

M1.1 and M1.2 remain `ACTIVE` until a final approval review is recorded and the approved architecture is promoted to an ADR/decision record. This review record and the revised proposal are evidence of design progress, not evidence that the milestones are already DONE.

### 2. Field-policy inheritance

Policy inheritance is dimension-specific rather than one global precedence rule.

For ordinary semantic ownership/configuration, the most specific valid policy refines the broader policy:

```text
semantic-field policy
  > record-type policy
  > file/container policy
  > system fallback
```

However:

- an explicit prohibition cannot be weakened by a narrower rule;
- privacy uses the most restrictive applicable classification;
- provenance requirements may be strengthened but not silently removed;
- evidence/history retention floors may not be shortened by omission or implicit inheritance.

### 3. Retention and deletion

The architecture must distinguish semantic replacement from deletion:

- superseding a decision is not deletion;
- omitting detail from a compact checkpoint is not deletion;
- deleting a derived Hub summary does not delete the local source evidence;
- a remote deletion does not imply local evidence deletion;
- absence of a field in a later checkpoint is not an implicit delete instruction.

Destructive deletion/purge requires an explicit operation and a policy applicable to the owning record. Evidence/history records default to preserve-or-supersede rather than silent deletion.

### 4. P3 hard block

If a candidate publication contains P3 material, the publication transaction fails. The system must not make the same payload publishable by silently redacting the P3 fragment and continuing.

A safe shared result may instead be created as a **new, independently derived sanitized record** that contains no P3 material, is reclassified on its own contents, preserves permitted provenance, and then enters the normal promotion path.

### 5. Unclassified default

`UNCLASSIFIED` is a provisional state, not a publishable privacy class:

```text
classification: UNCLASSIFIED
effective_storage_policy: local-only
publish_allowed: false
```

An item must be classified before Project Memory promotion. Unknown classification therefore fails closed.

### 6. Shared-checkpoint reconciliation scope

Three-way Hub reconciliation is limited to the semantic schema of the shared checkpoint. It does not merge the full local `.ai/` tree.

The local side first creates a minimized shared candidate/projection. Reconciliation then compares:

```text
B = last fully reconciled Hub checkpoint
R = latest remote Hub checkpoint
L = local proposed shared-checkpoint candidate
```

Only fields that belong to the shared-checkpoint schema participate in the B/R/L merge. Local `CURRENT`, `TASKS`, indexes, EXP/DEC source records, raw evidence, local sessions, and handoff state remain outside that merge except insofar as they provide evidence for producing L.

## Approval condition

No new blocking defect was identified. M1.1/M1.2 may proceed to final approval review once the proposal contains the six clarifications above consistently.

A successful final review should approve the authority/boundary decisions and promote the normative result to a dedicated ADR/decision record. It should not expand M1.1/M1.2 to absorb local multi-session write mechanics, exact Hub publication triggers, adapter implementation details, or performance budgets that belong to later milestones.
