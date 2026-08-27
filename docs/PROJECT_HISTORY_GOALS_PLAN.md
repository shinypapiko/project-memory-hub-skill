---
document_role: non-normative-project-governance-index
canonical_for:
  - planning-status
  - milestone-rollup
  - open-architecture-questions
  - project-history-rollup
not_canonical_for:
  - released-behavior
  - release-history
  - test-verdicts
  - runtime-project-state
architecture_state: proposed-unreleased
runtime_load_policy: maintenance-only
last_reviewed: 2026-08-27
evidence_baseline:
  - docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md
  - tests/results/v0.1.0-static-regression.md
  - docs/proposals/UNIFIED_AUTHORITY_MODEL.md
  - docs/reviews/2026-08-26-unified-authority-model-review.md
  - docs/reviews/2026-08-26-unified-authority-model-final-approval.md
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/proposals/LOCAL_MULTI_SESSION_SAFE_WRITE.md
  - docs/reviews/2026-08-26-local-multi-session-safe-write-review.md
  - docs/reviews/2026-08-27-local-multi-session-safe-write-final-approval.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
  - docs/proposals/REMOTE_REFRESH_RECONCILIATION_STATE_MACHINE.md
  - docs/reviews/2026-08-27-remote-refresh-reconciliation-review.md
  - docs/reviews/2026-08-27-remote-refresh-reconciliation-final-approval.md
  - docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md
  - docs/proposals/HUB_ADAPTER_TRANSPORT_BOUNDARY.md
  - docs/reviews/2026-08-27-hub-adapter-transport-boundary-final-approval.md
  - docs/decisions/ADR-0004-hub-adapter-and-transport-boundary.md
  - docs/proposals/PRIVACY_NEVER_PUBLISH_POLICY.md
  - docs/reviews/2026-08-27-privacy-never-publish-policy-final-approval.md
  - docs/decisions/ADR-0005-privacy-never-publish-policy.md
---

# Project Memory Skill — History, Goals, Plan, and Status

This file is the persistent **governance index and roadmap** for the Project Memory Skill / Hub integration effort.

It is not a runtime memory file and must not be loaded during normal project startup. It is intended for Skill maintenance, architecture work, milestone review, and recovery of the development history.

If this document conflicts with a released specification, release metadata, a test verdict, an accepted ADR, or runtime project state, the specialized source of truth wins.

## 1. Source-of-truth map

| Content | Canonical location |
| --- | --- |
| Planning status, milestone rollup, open design questions | this document |
| Released Skill behavior | `skill/SKILL.md` plus normative policy/schema docs |
| Version and release history | `VERSION`, `CHANGELOG.md`, Git tags/releases |
| Test verdicts | `tests/results/` and executable test outputs |
| Approved architecture decisions | dedicated ADR/decision records; this file keeps only rollup/linkage |
| Runtime project state | workspace `.ai/` and/or Runtime Hub according to the released architecture |
| Private project evidence | private workspace / Runtime Hub; never copied here merely for planning |

Execution-task tracking must not be duplicated indefinitely between this file and GitHub Issues. Until a future decision changes it, **this file is the primary milestone/task rollup**; Issues may be used for implementation work but must link back to milestone IDs.

## 2. Project goal

Build one maintainable Project Memory system for ChatGPT/Codex-style agents that:

- preserves useful project state across fresh chats and context compaction;
- keeps high-frequency working memory local to the workspace;
- supports selective retrieval rather than loading all history;
- preserves experiment, decision, task, handoff, session, and provenance history;
- supports deterministic multi-project routing and strict project isolation;
- provides a low-frequency shared/durable Hub for cross-session, cross-agent, and cross-device coordination;
- prevents stale sessions from silently overwriting newer mutable state;
- has one canonical release stream, explicit versioning/migrations, executable validation, and regression tests;
- minimizes normal startup memory payload and avoids redundant local/remote injection.

## 3. Architecture states — do not conflate them

Three different architecture states currently coexist. They are intentionally separated here.

### 3.1 Current released Hub Protocol — `project-memory-hub-skill 0.1.0`

Current released Hub behavior still defines the remote Hub project record as the authoritative project state for the Hub protocol. A substantive Hub session reads the selected remote `PROJECT.md`, records its blob SHA when available, and must refresh/reconcile before a shared write.

Current canonical source scope:

- repository: `shinypapiko/project-memory-hub-skill`
- canonical **for**: Hub Protocol `0.1.x` reusable specification/templates/tests
- supported Runtime Hub schema: `1`
- not yet established as the final canonical repository for the future unified Project Memory Skill

Maturity of the remote stale-SHA/reconciliation behavior:

- `implementation_status: PROTOCOL_ONLY`
- `validation_status: STATIC_VALIDATED`
- `live_validation: PENDING`

The protocol text and synthetic/static regression exist, but reusable Runtime Hub executable concurrency tooling is not yet implemented and the defined live two-agent concurrency test has not yet passed.

### 3.2 Current deployed local `project-memory` implementation

A strict read-only audit found a materially more mature local implementation with:

- Codex implicit discovery metadata;
- LOAD / RETRIEVE / CHECKPOINT workflows;
- `.ai/PROJECT.md`, `CURRENT.md`, `INDEX.md`, `TASKS.md`;
- indexed decisions and experiments;
- handoff inbox/outbox/processed lifecycle;
- optional sessions and research;
- retrieval/write/schema/handoff/transport policies;
- executable validation/status/ID tooling;
- Git-backed mailbox/handoff/receipt/non-canonical snapshot transport;
- executable smoke and transport E2E tests in the development distribution.

Version wording must remain precise:

> The audited development distribution and transport tool identify themselves as `1.1.0`; the installed Skill directories do not yet contain an independent Skill version marker.

The installed and development/test Skill copies inspected by the audit were byte-for-byte identical at audit time.

Evidence: `docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md`.

### 3.3 Target unified architecture — PARTIALLY ACCEPTED / UNRELEASED

The project is converging on **one Skill with two memory domains**, not two competing Skills:

```text
                         Canonical Skill Source
                                  |
                                  v
                         project-memory Skill
                       /                       \
                      /                         \
             Local Memory Core              Hub Adapter
                    |                            |
                    v                            v
              workspace .ai/              Runtime Hub
          hot + detailed evidence       shared durable checkpoint
                    |                            |
                    +--------- reconcile --------+
```

M1.1/M1.2 authority and field-boundary decisions are accepted in `ADR-0001`. M1.3 local multi-session safe-write semantics are accepted in `ADR-0002`. M1.4 remote refresh/reconciliation semantics are accepted in `ADR-0003`. M1.5 Hub adapter vs existing transport boundary semantics are accepted in `ADR-0004`. M1.6 deterministic privacy/classification/publication-gate semantics are accepted in `ADR-0005`. The wider unified architecture remains unreleased and incomplete: M1.7–M1.8, M1.9 final architecture approval, source/version decisions, implementation, migration, and validation still remain.

The accepted ADRs do not override current released `0.1.0` Hub behavior or the current deployed local behavior until an explicit later release/migration is approved.

## 4. Accepted authority/boundary architecture — ADR-0001

Authority is divided by information class and scope rather than by declaring local or remote storage globally superior.

| Information class / record | Accepted owner / role | Publication direction |
| --- | --- | --- |
| Source code, datasets, generated outputs, raw logs, direct experiment artifacts | workspace evidence itself | never replaced by memory summaries |
| `AGENTS.md` binding metadata | workspace binding for Skill/project identity | may reference stable `project_id` / Hub entry; not a project-state transcript |
| local `.ai/PROJECT.md` | stable local project background, constraints, terminology, repository map | selected minimized shared fields only |
| local `.ai/CURRENT.md` | authoritative active-work state for the local workspace | promoted selectively, never mirrored automatically |
| local `.ai/TASKS.md`, DEC/EXP/handoffs/local sessions/research | detailed local lifecycle/evidence | local by default; promote derived conclusions only when policy permits |
| Hub `projects/<project-id>/PROJECT.md` | future last reconciled shared checkpoint for other agents/devices | shared-checkpoint projection/reconciliation only |
| Hub `projects/INDEX.md` | cross-project identity/routing registry | Hub-only routing role; not equivalent to local `.ai/INDEX.md` |
| Hub per-project `sessions/` | important shared session trace | only when cross-session traceability justifies publication |
| Hub shared `research/` / `decisions/` | explicitly promoted reusable/cross-project material | promotion requires scope/applicability metadata |

Accepted principle:

> **Evidence and scope determine authority; neither local nor remote storage wins merely because of location.**

The field-level architecture additionally fixes classification/privacy fail-closed behavior, field-policy inheritance, retention/deletion semantics, and semantic B/R/L shared-checkpoint reconciliation scope.

Evidence:

- `docs/proposals/UNIFIED_AUTHORITY_MODEL.md`
- `docs/reviews/2026-08-26-unified-authority-model-review.md`
- `docs/reviews/2026-08-26-unified-authority-model-final-approval.md`
- `docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md`

## 5. Frequency and retrieval model — proposed

- Local memory is high-frequency.
- Hub memory is low-frequency.
- Detailed `EXP-*`, local decision mechanics, transient debugging, and every task result must not be copied automatically into the Hub.
- Normal healthy local startup should eventually use local LOAD plus a lightweight Hub change check, not unconditional remote full-text injection.
- Full Hub routing/recovery remains necessary for web-only agents, new devices, missing/untrusted local memory, schema mismatch, remote changes, or explicit recovery/current-state requests.

Target normal path:

```text
AGENTS / Skill discovery
        ↓
local LOAD
        ↓
.ai/PROJECT + CURRENT + INDEX
        ↓
Hub sync metadata / lightweight remote check
        ↓
remote unchanged?
   ├─ yes → continue without full Hub PROJECT injection
   └─ no  → fetch changed shared state → reconcile → continue
```

This is proposed, not current released behavior.

## 6. Maturity vocabulary

Do not use one ladder that confuses implementation with validation. Track them independently.

### Implementation status

- `PROTOCOL_ONLY` — behavior exists as specification/templates/policy only.
- `EXECUTABLE` — reusable implementation code exists for the behavior.

### Validation status

- `UNVALIDATED` — no recorded validation result.
- `STATIC_VALIDATED` — specification/static/synthetic consistency validated.
- `LIVE_VALIDATED` — representative live behavior executed successfully.

A capability may be `EXECUTABLE` but not `LIVE_VALIDATED`, or `PROTOCOL_ONLY` yet have a manually executed live validation. Record both dimensions where maturity matters. Validation evidence should be capability-scoped, version/commit-bound, and append-oriented rather than overwriting older validation records.

## 7. Important history

- **H0 — local Project Memory created.** `.ai/`, Markdown, retrieval/write policies, experiments, decisions, handoffs, and helper scripts became the practical local memory used by Codex.
- **H1 — Runtime Hub introduced.** Added deterministic workspace/project identity, multi-project isolation, remote shared state, and remote coordination concepts.
- **H2 — Hub Skill source repo created.** `project-memory-hub-skill 0.1.0` became the canonical reusable source for Hub Protocol `0.1.x` / schema-1 compatibility.
- **H3 — session / remote concurrency protocol added.** Session logs, `base_project_sha`, pre-write refresh, stale detection, and reconcile-before-write were specified.
- **H4 — real usage exposed duplication.** Local `.ai` proved faster-moving and more detailed than remote Hub state; unconditional dual loading created redundant context.
- **H5 — strict read-only audit completed.** Confirmed the local implementation already contains many features that must not be reimplemented independently; identified dual-authority risk and the need for unification before `0.2.0` implementation.
- **H6 — governance review.** The project-management document itself was corrected so proposed architecture, released behavior, maturity, evidence, and runtime state cannot silently become a second specification/source of truth.
- **H7 — M1 authority/boundary review.** Architecture direction was accepted with no blocking defect; six required clarifications were identified and integrated: roadmap evidence, field-policy inheritance, retention/deletion, P3 hard block, unclassified fail-closed default, and shared-checkpoint reconciliation scope.
- **H8 — M1.1/M1.2 final approval and ADR promotion.** R1–R12 were approved, M1.1/M1.2 were approved with no blocking defects, and the accepted architecture was promoted to `ADR-0001`. No current runtime/released behavior was changed.
- **H9 — M1.3 local concurrency design started.** A proposal was drafted for optimistic per-file content hashes plus a short commit-time filesystem lock, stale-write rejection/rebase, atomic replace, concurrent DEC/EXP ID protection, and conservative no-lost-update behavior without requiring Git.
- **H10 — M1.3 first review.** L1/L4/L6/L10/L11/L12 were approved; L2/L3/L5/L7/L8/L9 required revision. Five blocking defects were identified around cooperative-writer scope/canonical resource identity, bounded liveness and multi-lock cleanup, Windows/atomic-install failure classification, INDEX repairability, and partial multi-resource completion. The eight required clarifications were integrated into the proposal.
- **H11 — M1.3 final approval and ADR promotion.** Revised L1–L12 were all approved, B1–B5 were all closed, no new blocking defect was identified, and the accepted local safe-write architecture was promoted to `ADR-0002`. The future unified capability remains `PROTOCOL_ONLY` / `UNVALIDATED`; no current runtime/released behavior was changed.
- **H12 — M1.4 remote reconciliation design started.** The first proposal separated `observed_sha` from `reconciled_sha`, defined B/R/L/C, field-level three-way classification, CAS stale recomputation, bounded contention, conflict outcomes, unknown-commit recovery, and marker-advancement rules.
- **H13 — M1.4 initial review and clarification revision.** RR1/RR2/RR3/RR6/RR8/RR9/RR10/RR11/RR14 were approved; RR4/RR5/RR7/RR12/RR13 required revision. Four blocking defects were identified around explicit conflict resolution, canonical local base-pair drift, post-CAS/local-finalization states, and base-loss/ABSENT semantics. The proposal added a context-bound resolution operation, canonical base-pair token checks and re-projection, exact commit confirmation plus guarded local finalization, and explicit present/absent/uninitialized/invalid base states.
- **H14 — M1.4 final approval and ADR promotion.** RR1–RR14 were all approved, B1–B4 were closed, no new blocking defect was identified, and accepted remote refresh/reconciliation semantics were promoted to `ADR-0003`. The capability remains `PROTOCOL_ONLY` / `UNVALIDATED` with `runtime_effective:false`; current runtime/released behavior remains unchanged.
- **H15 — M1.5 boundary review and ADR promotion.** The initial review approved TB1/TB4/TB5/TB6/TB7/TB9/TB11/TB14/TB15 and required revision of TB2/TB3/TB8/TB10/TB12/TB13, identifying B1 remote resource ownership, B2 shared Git transaction domain, and B3 cross-layer project binding. The revised proposal closed B1–B3. Final review approved TB1–TB15 with no blocking defects and the accepted transport/Hub-adapter boundary was promoted to `ADR-0004`. The future adapter capability remains `PROTOCOL_ONLY` / `UNVALIDATED` with `runtime_effective:false`; no current runtime/released behavior was changed.
- **H16 — M1.6 privacy policy final approval and ADR promotion.** Initial and final review both approved PV1–PV16 with no blocking defects or required semantic clarifications. The deterministic destination-aware privacy/classification/publication-gate semantics were promoted to `ADR-0005`. The capability remains `PROTOCOL_ONLY` / `UNVALIDATED` with `runtime_effective:false`; M1.6 `DONE` does not imply runtime implementation, migration, or validation.

## 8. Planning decisions

These remain planning-level unless covered by an accepted ADR. Where a planning item overlaps `ADR-0001`, `ADR-0002`, `ADR-0003`, `ADR-0004`, or `ADR-0005`, the ADR is authoritative for the accepted architecture scope.

- **D-P1 — Do not build a second local-memory implementation.** Reuse/productize the existing local core.
- **D-P2 — Keep the Hub, but narrow its proposed unified role.** Registry/routing, shared checkpoint, cross-agent/device coordination, shared trace, remote concurrency, compatibility/migration.
- **D-P3 — Pre-write refresh remains mandatory.** Before writing shared mutable state, re-read/re-check latest state and reconcile rather than blindly overwrite. Remote Hub uses blob SHA where available; local M1.3 additionally requires a commit-time concurrency primitive to close the TOCTOU window.
- **D-P4 — One canonical Skill release stream is required.** Manually synchronized installed/dev behavior copies are not acceptable long term.
- **D-P5 — Existing transport and future Hub adapter are separate semantic layers.** Accepted by `ADR-0004`; mailbox/handoff/receipt/non-canonical snapshot semantics cannot become shared-checkpoint reconciliation state, and shared Git plumbing/resource/project identity must remain typed and fail closed across subsystem boundaries.
- **D-P6 — Shared reconciliation is semantic, not full-tree synchronization.** Accepted by `ADR-0001`; local state is projected into a minimized shared-checkpoint candidate before B/R/L reconciliation.
- **D-P7 — Publication fails closed on privacy uncertainty.** Accepted and made deterministic by `ADR-0005`; destination-aware evaluation, monotonic classification inheritance, P3/UNCLASSIFIED hard blocks, safe-derivative independence, exact-context verdict invalidation, bundle preflight, and privacy-before-CAS ordering are required.
- **D-P8 — Local no-lost-update is a cooperative-writer contract.** Accepted by `ADR-0002`; the target uses canonical resource identity, exact base hashes, short commit locks, atomic replace/no-replace install, bounded retry, and structured partial/unknown commit results; direct writers that bypass the helper are outside the mutual-exclusion guarantee.
- **D-P9 — Remote observation and reconciliation are distinct.** Accepted by `ADR-0003`; observing a newer remote SHA never by itself advances the reconciled base, and stale CAS invalidates the old candidate and forces fresh fetch plus B/R/L recomputation.
- **D-P10 — Remote reconciliation finalizes locally under a canonical base-pair guard.** Accepted by `ADR-0003`; remote CAS success and persistent local reconciliation are separate stages, canonical local base drift invalidates old candidates/explicit resolution context, and a newer remote head may legitimately remain ahead of `reconciled_sha`.

## 9. Open architecture questions

Resolve before migration/implementation:

- Which repository becomes the canonical source for the **future unified** Skill?
- What reusable subset of the audited local distribution is imported first?
- What metadata/storage representation will implement the accepted canonical `B/reconciled_sha` pair, `base_pair_token`, `observed_sha`, and exact base-content integrity checks?
- Which concrete platform primitives satisfy `ADR-0002` on supported Windows/NTFS environments, and how will they be executable-tested?
- What later executable mechanism will implement `ADR-0003` B/R/L/C + explicit-resolution + CAS/confirmation/finalization semantics?
- What exact milestone/shared-state trigger causes a Hub publication?
- What privacy-classification/detection mechanism implements accepted `ADR-0005` semantics safely?
- How does web/new-device recovery establish or recover trusted base lineage without making a merely observed remote state automatically reconciled?
- What startup/retrieval/Hub-probe/compaction budgets are realistic?
- Do privacy-driven purge requirements need a dedicated secure-deletion model beyond ordinary preserve/archive/supersede retention?

## 10. Performance targets — engineering targets, not guarantees

- Normal healthy bound-local startup: approximately **1k–2k tokens of Skill memory payload**.
- Selective retrieval: index first, then only one/few relevant DEC/EXP/session/research records, broader evidence only if uncertainty remains.
- Hub `PROJECT.md`: compact shared checkpoint; detailed history belongs in linked records. Concrete soft byte/token threshold remains to be measured.

## 11. Work breakdown

Status values are only `DONE`, `ACTIVE`, `PLANNED`, `BLOCKED`, `DEFERRED`, `REJECTED`. Do not encode maturity/dependency explanations inside Status.

### M0 — evidence baseline

| ID | Deliverable | Status | Depends on | Acceptance | Evidence | Last verified |
| --- | --- | --- | --- | --- | --- | --- |
| M0.1 | Runtime Hub routing/isolation baseline exists | DONE | — | deterministic single-project routing + isolation specified in schema-1 runtime | Runtime Hub `START_HERE.md`, `projects/INDEX.md`, `HUB_SCHEMA.md` | 2026-08-26 |
| M0.2 | Fresh-chat round trip demonstrated on real bound workspaces | DONE | M0.1 | fresh chat derives binding, reads correct project, performs scoped write-back | private Runtime Hub round-trip challenge/response artifacts | 2026-08-26 |
| M0.3 | Remote session/stale-SHA protocol specified | DONE | M0.1 | pre-write refresh, stale detection, reconcile-before-write documented | `docs/concurrency.md`, `prompts/session-protocol.md`, `tests/results/v0.1.0-static-regression.md` | 2026-08-26 |
| M0.4 | Hub Skill reference source repository created | DONE | — | source-controlled Hub protocol/templates/tests/versioning exist | `VERSION`, `CHANGELOG.md`, `skill/SKILL.md` | 2026-08-26 |
| M0.5 | Hub Skill `0.1.0` / Hub schema 1 compatibility defined | DONE | M0.4 | release/schema compatibility is explicit | `VERSION`, `docs/compatibility.md`, Runtime Hub `HUB_SCHEMA.md` | 2026-08-26 |
| M0.6 | Existing local implementation audited read-only | DONE | — | reusable behavior, schema, tooling, transport, version provenance, gaps recorded without mutation | `docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md` | 2026-08-26 |
| M0.7 | Dual-source-of-truth risk identified | DONE | M0.6 | conflicting authority surfaces and non-equivalent indexes/snapshots documented | audit summary + governance review | 2026-08-26 |

### M1 — unified architecture specification

| ID | Deliverable | Status | Depends on | Acceptance | Evidence | Last verified |
| --- | --- | --- | --- | --- | --- | --- |
| M1.1 | Authority model by information scope | DONE | M0 | every information class has one explicit owner/role; final review accepts R1–R12 | `docs/proposals/UNIFIED_AUTHORITY_MODEL.md`; initial review; final approval; `ADR-0001` | 2026-08-26 |
| M1.2 | Field-level local/Hub mapping | DONE | M1.1 | owner, writer, readers, publish direction, conflict rule, provenance, privacy/classification, retention/deletion and reconciliation scope defined; final review accepts R1–R12 | `docs/proposals/UNIFIED_AUTHORITY_MODEL.md`; initial review; final approval; `ADR-0001` | 2026-08-26 |
| M1.3 | Local multi-session safe-write semantics | DONE | M1.2 | no-lost-update cooperative-writer contract, canonical resource identity, bounded retry, atomic replace/no-replace, ID allocation, index repair boundaries, multi-lock cleanup and partial/unknown commit recovery specified; final review accepts revised L1–L12 | `docs/proposals/LOCAL_MULTI_SESSION_SAFE_WRITE.md`; initial review; final approval; `ADR-0002` | 2026-08-27 |
| M1.4 | Unified remote refresh/reconcile semantics | DONE | M1.2 | deterministic B/R/L/C state machine defines observed vs reconciled SHA, canonical local base drift, explicit conflict resolution, CAS stale recomputation, bounded contention, exact remote confirmation, guarded local finalization, conflict/unknown-commit outcomes, base-loss/ABSENT handling, and exact marker-advancement rules; final review accepts revised RR1–RR14 | `docs/proposals/REMOTE_REFRESH_RECONCILIATION_STATE_MACHINE.md`; initial review; final approval; `ADR-0003` | 2026-08-27 |
| M1.5 | Hub adapter vs transport boundary | DONE | M0.6 | transport/adapter semantic ownership, remote resource classes, shared Git transaction-domain isolation, cross-layer canonical binding checks, typed result/fallback boundaries and compatibility lifecycle defined; final review accepts TB1–TB15 and closes B1–B3 | `docs/proposals/HUB_ADAPTER_TRANSPORT_BOUNDARY.md`; final approval; `ADR-0004` | 2026-08-27 |
| M1.6 | Privacy / never-publish classes | DONE | M1.2 | deterministic destination-aware P0/P1/P2/P3/UNCLASSIFIED policy, restrictive inheritance, provenance evaluation, exact-context verdict invalidation, whole-bundle preflight, safe-derivative contract, retention/purge boundary and privacy-before-CAS ordering defined; final review accepts PV1–PV16 with no blocking defects | `docs/proposals/PRIVACY_NEVER_PUBLISH_POLICY.md`; final approval; `ADR-0005` | 2026-08-27 |
| M1.7 | Normal fast path vs recovery path | PLANNED | M1.2 | conditions and fallbacks explicit | — | — |
| M1.8 | Memory payload / compaction budgets | PLANNED | M1.7 | measurable budgets and thresholds defined | — | — |
| M1.9 | Unified Architecture Proposal approved | BLOCKED | M1.1–M1.8 | approved ADR/spec before source migration | M1.1–M1.6 accepted via `ADR-0001` / `ADR-0002` / `ADR-0003` / `ADR-0004` / `ADR-0005`; M1.7–M1.8 pending | — |

### M2 — canonical source and version model

| ID | Deliverable | Status | Depends on | Acceptance | Evidence | Last verified |
| --- | --- | --- | --- | --- | --- | --- |
| M2.1 | Decide future unified canonical source repo | PLANNED | M1.9 | repo/name/ownership/release scope explicit | current repo remains canonical only for Hub Protocol `0.1.x` | — |
| M2.2 | Import reusable local implementation without project/user data | BLOCKED | M1.9, M2.1 | reusable core appears in canonical source with privacy review | — | — |
| M2.3 | Preserve Codex discovery metadata | BLOCKED | M2.2 | installed release remains discoverable/implicit where intended | — | — |
| M2.4 | Port executable tests/validation | BLOCKED | M2.2 | local smoke/E2E/validation capabilities run from canonical source | — | — |
| M2.5 | Define unified version lineage | PLANNED | M1.9 | local distribution `1.1.0` provenance and Hub `0.1.0` lineage mapped without pretending direct semver continuity | — | — |
| M2.6 | Add independent installed-Skill version metadata and release-based install/update | PLANNED | M2.1–M2.5 | installed Skill version can be verified without relying on outer repo/tool marker | — | — |

### M3 — Hub adapter and lazy synchronization

| ID | Deliverable | Status | Depends on | Acceptance | Evidence | Last verified |
| --- | --- | --- | --- | --- | --- | --- |
| M3.1 | Sync metadata schema | PLANNED | M1.2 | last-reconciled remote identity/SHA/schema/path semantics explicit | — | — |
| M3.2 | Last-reconciled SHA semantics | PLANNED | M3.1 | distinguishes merely observed remote SHA from fully reconciled state | — | — |
| M3.3 | Lightweight remote-change probe | BLOCKED | M2, M3.1 | detects changed shared checkpoint without full payload injection | — | — |
| M3.4 | Unchanged-remote fast path | BLOCKED | M3.3 | full Hub project read skipped safely when unchanged | — | — |
| M3.5 | Changed-remote fetch/reconcile path | BLOCKED | M3.1–M3.3, M1.4 | preserves compatible local + remote shared fields, surfaces conflicts, never merges full local tree | — | — |
| M3.6 | Milestone/shared-checkpoint publication | BLOCKED | M1.2, M1.6 | explicit promotion trigger and compact classified payload | — | — |
| M3.7 | Local detailed evidence stays local by default | PLANNED | M1.6 | EXP/DEC/task details are never bulk mirrored implicitly | — | — |
| M3.8 | Web/new-device hydration | PLANNED | M1.7 | recovery works without local filesystem access | — | — |

### M4 — concurrency hardening

| ID | Deliverable | Status | Depends on | Acceptance | Evidence | Last verified |
| --- | --- | --- | --- | --- | --- | --- |
| M4.1a | Remote stale-SHA protocol specified/static-validated | DONE | M0.3 | protocol consistency passes static regression | `tests/results/v0.1.0-static-regression.md` | 2026-08-26 |
| M4.1b | Remote stale-SHA behavior live-validated | PLANNED | M4.1a | defined two-session live test passes | `tests/LIVE_CONCURRENCY_TEST.md` (procedure only) | — |
| M4.2 | Local safe concurrent-write design | PLANNED | M1.3 | accepted M1.3 contract is translated into executable design including canonical identity, atomic install, bounded retry and partial recovery | — | — |
| M4.3 | Executable local multi-session conflict tests | BLOCKED | M4.2 | controlled concurrent fixtures pass | — | — |
| M4.4 | Local-vs-remote divergence tests | PLANNED | M1.4, M3 | compatible and conflicting shared projections behave as specified | — | — |
| M4.5 | Three-session stress scenario | PLANNED | M4.3, M4.4 | A/B/C concurrent compatible updates produce no lost update | — | — |

### M5 — migration and compatibility

| ID | Deliverable | Status | Depends on | Acceptance | Evidence | Last verified |
| --- | --- | --- | --- | --- | --- | --- |
| M5.1 | Existing `.ai/` migration plan | BLOCKED | M1, M2 | no existing local evidence is silently lost/reclassified | — | — |
| M5.2 | Existing bound-Hub project migration plan | BLOCKED | M1, M2 | routing/round-trip compatibility preserved | — | — |
| M5.3 | Migration privacy guarantee | PLANNED | M1.6 | no private project evidence copied into public Skill source | — | — |
| M5.4a | Existing local installer preserves unrelated `AGENTS.md` instructions | DONE | — | existing behavior evidenced by audit/test coverage | sanitized audit summary; existing local smoke-test coverage reported | 2026-08-26 |
| M5.4b | Unified migration preserves unrelated `AGENTS.md` instructions | PLANNED | M5.1, M5.2 | synthetic migration proves preservation after unification | — | — |
| M5.5 | Rollback / pre-upgrade checkpoint | PLANNED | M5.1, M5.2 | rollback target and procedure explicit | — | — |
| M5.6 | Synthetic migration before real project | PLANNED | M5.1–M5.5 | synthetic fixtures pass before any real migration | — | — |

### M6 — validation and release

| ID | Deliverable | Status | Depends on | Acceptance | Evidence | Last verified |
| --- | --- | --- | --- | --- | --- | --- |
| M6.1 | Unified install smoke tests | PLANNED | M2, M5 | install/update/idempotence/discovery pass | — | — |
| M6.2 | Retrieval/write/EXP/DEC executable tests | PLANNED | M2 | local core behavior preserved | — | — |
| M6.3 | Hub adapter executable tests | PLANNED | M3 | probe/reconcile/publish/error paths pass | — | — |
| M6.4 | Live remote concurrency | PLANNED | M4 | two-session live validation passes | — | — |
| M6.5 | Token/startup benchmark | PLANNED | M3 | measured normal payload meets/adjusts target | — | — |
| M6.6 | Fresh-chat round trip after behavior-affecting release | PLANNED | M5, M6 | upgraded bound workspace passes scoped round trip | — | — |
| M6.7 | Release gate | BLOCKED | M5, M6 | architecture, migration, executable tests, rollback, compatibility metadata complete | — | — |

## 12. Explicitly rejected or paused directions

- `REJECTED` — build another independent local `.ai` schema inside the Hub Skill.
- `REJECTED` — automatically copy every local `EXP-*` into Hub `PROJECT.md`.
- `REJECTED` — treat existing transport CURRENT snapshots as authoritative Hub state.
- `REJECTED` — blindly allow remote Hub state to overwrite newer local active-work evidence.
- `REJECTED` — blindly allow local active state to overwrite newer shared Hub state.
- `REJECTED` — use full-tree/local-file bidirectional synchronization as Hub checkpoint reconciliation.
- `REJECTED` — allow P3 or UNCLASSIFIED material to enter Project Memory promotion.
- `REJECTED` — treat pre-write refresh alone as sufficient local concurrency protection without a commit-time primitive.
- `REJECTED` — claim local no-lost-update mutual exclusion against non-cooperative direct writers.
- `REJECTED` — fall back from failed atomic replace/install to truncating or in-place overwrite.
- `REJECTED` — treat all index content as safely reconstructible.
- `REJECTED` — silently roll back partially committed multi-resource operations using stale snapshots.
- `REJECTED` — treat a newly observed remote SHA as reconciled without semantic acceptance.
- `REJECTED` — retry a stale remote CAS by replaying a candidate computed from an older remote revision.
- `REJECTED` — treat remote CAS success as equivalent to successfully persisting the canonical local `B/reconciled_sha` pair.
- `REJECTED` — replay an explicit conflict resolution after its bound B/R/L context or canonical local base has changed.
- `REJECTED` — infer trusted `reconciled_sha=ABSENT` solely because the remote checkpoint is currently absent.
- `REJECTED` — reinterpret transport mailbox/receipt/snapshot state as Hub adapter reconciliation state.
- `REJECTED` — assume separate semantic namespaces isolate shared Git index/HEAD/ref/staging/commit/push state.
- `REJECTED` — infer canonical project identity from transport config, path/channel similarity, or physical co-location.
- `PAUSED` — nominal `0.2.0` implementation before the unified architecture/source model is approved.
- `PAUSED` — migrate real `.ai/` trees before synthetic migration/rollback tests exist.

## 13. Unified architecture acceptance criteria

Implementation may begin only when:

1. the future unified canonical source repository/release stream is explicit;
2. each information class has one unambiguous authority role;
3. field-level local/Hub mapping prevents silent dual authority;
4. local and remote pre-write refresh semantics are specified;
5. local/remote divergence has deterministic reconciliation behavior;
6. detailed experiments/transient work are not automatically promoted;
7. privacy / never-publish classes are explicit;
8. normal local startup can safely skip unchanged remote full reads;
9. web/new-device/recovery paths remain possible;
10. executable tests cover core local behavior, routing/isolation, adapter behavior, migration, and concurrency;
11. rollback exists;
12. memory/token payload is measured.

M1.1 and M1.2 satisfy the authority and field-boundary portions of these criteria through `ADR-0001`; M1.3 satisfies the local cooperative-writer safe-write architecture portion through `ADR-0002`; M1.4 satisfies the deterministic remote refresh/reconciliation architecture portion through `ADR-0003`; M1.5 satisfies the transport/Hub-adapter semantic, resource, Git transaction-domain, binding, and fallback boundary through `ADR-0004`; M1.6 satisfies deterministic privacy/classification/publication-gate architecture through `ADR-0005`. The overall architecture gate remains open until M1.7–M1.8 and the later source/implementation/validation requirements are complete.

## 14. Release gate

A behavior-affecting unified release is not complete merely because Markdown protocols changed. It requires:

- approved architecture/ADR;
- canonical source migration/import;
- executable tests passing;
- synthetic migration passing;
- live remote concurrency validation;
- representative local multi-session validation;
- fresh-chat round trip after upgrade;
- documented rollback;
- version / compatibility metadata updated.

## 15. Maintenance rules

Update this governance index when architecture direction, planning decisions, milestone status, audits, migrations/releases, assumptions, or blockers materially change.

Do not use it as a transcript or a runtime LOAD input.

When a planning decision becomes approved architecture, move the normative decision into a dedicated ADR/decision record and keep only a concise rollup/link here.

When a task becomes `DONE`, populate enough evidence and a verification date that a future maintainer can determine why it is complete.