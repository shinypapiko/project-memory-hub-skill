---
document_role: architecture-decision-record
adr_id: ADR-0005
status: accepted
accepted: 2026-08-27
milestone: M1.6
runtime_effective: false
release_required_for_runtime_effect: true
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
proposal: docs/proposals/PRIVACY_NEVER_PUBLISH_POLICY.md
final_approval: docs/reviews/2026-08-27-privacy-never-publish-policy-final-approval.md
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
  - docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md
  - docs/decisions/ADR-0004-hub-adapter-and-transport-boundary.md
---

# ADR-0005 — Privacy / never-publish policy

## Status

**Accepted architecture decision.**

This ADR records the approved M1.6 deterministic privacy, classification, and publication-gate semantics for the future unified Project Memory architecture.

It is normative for later unified-architecture design and implementation, but it does **not** change current released Skill behavior, Runtime Hub runtime behavior, local Project Memory behavior, or transport behavior. A later implementation and release are required before these semantics become runtime-effective.

Current maturity remains:

```text
runtime_effective: false
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
```

## Context

ADR-0001 established fail-closed privacy principles: `UNCLASSIFIED` remains local-only until classified, P3 `PROHIBITED_SYNC` hard-blocks publication, explicit prohibitions cannot be weakened, the most restrictive applicable rule wins, and a blocked candidate cannot be made publishable by silently deleting prohibited fragments from the same candidate.

M1.6 makes those principles deterministic across destination, inheritance, provenance, candidate identity, bundle preflight, and the boundary with ADR-0003 write-capable reconciliation.

The final review approved PV1–PV16 with no blocking defects and no required changes before ADR promotion.

## Decision

PV1–PV16 from `docs/proposals/PRIVACY_NEVER_PUBLISH_POLICY.md` are accepted in full.

### 1. Five privacy states

The architecture uses exactly these semantic states:

- P0 `STRUCTURAL`
- P1 `RUNTIME_SHAREABLE`
- P2 `LOCAL_BY_DEFAULT`
- P3 `PROHIBITED_SYNC`
- `UNCLASSIFIED`

`UNCLASSIFIED` is fail-closed for publication; it is not an implicit low-risk class.

### 2. Destination-aware permission

Privacy permission is evaluated for an exact destination. At minimum the architecture distinguishes local use, an exact authorized private Runtime Hub destination, and the public Skill source.

P0 may be eligible for public source only when independently public-source-safe. P1 may be eligible only for the exact authorized private runtime destination/scope and is not public-source permission. Permission for one destination never transfers automatically to another.

### 3. P2 requires explicit promotion and reclassification

P2 is local-by-default and is not directly remotely publishable. Remote use requires explicit promotion, minimization or semantic projection, creation of a new candidate/derivative, and independent destination-specific classification.

Reclassification is not a relabel-only operation.

### 4. P3 and UNCLASSIFIED hard-block remote publication

Any applicable P3 or included `UNCLASSIFIED` component hard-blocks the exact remote publication candidate/bundle. Parent defaults, child overrides, destination defaults, prior publication, transport state, or remote presence cannot weaken that block.

Explicit never-publish rules are monotonic and cannot be weakened.

### 5. Classification inheritance is restrictive and fail-closed

Privacy evaluation covers records, fields, attachments, provenance, candidates, and publication bundles. More specific rules may strengthen restrictions but cannot weaken an applicable prohibition or more restrictive parent/context rule.

If applicable classification or policy rules cannot be deterministically reconciled, the result is a conflict/non-allow outcome rather than a permissive interpretation.

### 6. Metadata and provenance are privacy-bearing

Privacy evaluation includes all outbound or semantically required provenance and metadata, including paths, filenames, hashes, identifiers, attachments, logs, and diagnostic/error details when present in the publication object or required provenance.

Required prohibited or unclassified provenance blocks publication. Required provenance must not be silently discarded merely to obtain an allow verdict.

### 7. Blocked candidates cannot be silently redacted and resumed

If an exact candidate contains P3 or `UNCLASSIFIED` material, that candidate remains blocked. The system must not silently strip the blocked fragment and continue publication under the same candidate identity or verdict.

A safe sanitized/minimized result must be a new independent derivative with a new identity and a fresh provenance, classification, destination, and privacy evaluation context.

### 8. Privacy verdicts bind exact context

A reusable privacy decision must bind the exact candidate identity plus the applicable classification/policy context, destination, permitted provenance, binding/scope, and publication-bundle identity.

Candidate, classification, policy, destination, provenance, binding/scope, or bundle drift invalidates the old verdict. A prior `ALLOW` cannot be replayed after such drift.

### 9. Whole-bundle preflight occurs before remote publication effects

The complete intended publication bundle must receive current privacy `ALLOW` before the first remote publication side effect. Any blocked, stale, conflicting, unresolved, or derivative-required member blocks the attempt before publication writes begin.

M1.6 defines this ordering requirement, not the concrete transaction implementation.

### 10. Typed privacy outcomes remain distinct

The unified architecture must distinguish at least allow, block, derivative-required, stale, and conflict classes of outcome, including destination/provenance/classification/binding causes where relevant.

Non-allow outcomes must not be flattened into success.

### 11. Privacy blocking is not deletion

A publication block does not delete the source. Omission, redaction, minimization, retention, purge, Git-history deletion, and secure deletion are distinct semantic operations.

M1.6 does not define purge or secure-deletion implementation.

### 12. Existing remote or transport presence grants no publication permission

Transport mailbox state, receipts, snapshots, prior remote copies, prior publication elsewhere, or the mere fact that a remote is private/authenticated do not grant new Runtime Hub or public-source publication permission.

Each publication attempt is evaluated against its exact current privacy context and destination.

### 13. Privacy gates ADR-0003 write-capable reconciliation

The exact current candidate/bundle must have a current privacy `ALLOW` before entering the ADR-0003 write-capable reconciliation/CAS path.

If reconciliation recomputation, conflict resolution, stale-CAS handling, or any other step changes the bound candidate/privacy context before the write, privacy must be re-evaluated. Privacy `ALLOW` does not itself mean CAS, reconciliation, or finalization succeeded.

## Consequences

The architecture now has a deterministic privacy boundary for future shared publication without requiring detector/runtime implementation in M1.6. Later implementation must preserve destination isolation, restrictive inheritance, exact-context verdict validity, full-bundle preflight, and privacy-before-write ordering.

M1.6 `DONE` records architecture acceptance only. It does not establish executable privacy classification, runtime publication gating, migration, or validation.

## Deferred

This ADR does not decide M1.7 startup/recovery behavior, publication triggers, detector/classifier implementation, adapter/runtime implementation, concrete storage/schema encoding, concrete remote API choices, lock primitives, retry values, migration, real-data operations, purge execution, Git-history deletion procedures, or secure-deletion implementation.

## Evidence and review

- `docs/proposals/PRIVACY_NEVER_PUBLISH_POLICY.md`
- `docs/reviews/2026-08-27-privacy-never-publish-policy-final-approval.md`
- final review frozen baseline: `1ef9f4b69d699e79cf02314787acb10d82d4c303`
- reviewed proposal Git-blob LF SHA-256: `91916BE03E68AEE652265CC33761D0FB375C2A2E1D3ED26DAE8DC85760C4CF69`

The final review approved PV1–PV16, found no blocking defects, and required no semantic changes before promotion.