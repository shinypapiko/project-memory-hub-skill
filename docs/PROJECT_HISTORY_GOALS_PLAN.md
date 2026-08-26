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
last_reviewed: 2026-08-26
evidence_baseline:
  - docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md
  - tests/results/v0.1.0-static-regression.md
  - docs/proposals/UNIFIED_AUTHORITY_MODEL.md
  - docs/reviews/2026-08-26-unified-authority-model-review.md
---

# Project Memory Skill — History, Goals, Plan, and Status

This file is the persistent **governance index and roadmap** for the Project Memory Skill / Hub integration effort.

It is not a runtime memory file and must not be loaded during normal project startup. It is intended for Skill maintenance, architecture work, milestone review, and recovery of the development history.

If this document conflicts with a released specification, release metadata, a test verdict, or runtime project state, the specialized source of truth wins.

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

### 3.3 Target unified architecture — PROPOSED / UNRELEASED

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

This model is a planning target only. It does not override current released `0.1.0` Hub behavior or the current deployed local behavior until an explicit unified release/migration is approved.

## 4. Proposed authority model — detailed proposal under final approval review

Authority is intended to be divided by information class and scope rather than by declaring local or remote storage globally superior.

| Information class / record | Proposed owner / role | Publication direction |
| --- | --- | --- |
| Source code, datasets, generated outputs, raw logs, direct experiment artifacts | workspace evidence itself | never replaced by memory summaries |
| `AGENTS.md` binding metadata | workspace binding for Skill/project identity | may reference stable `project_id` / Hub entry; not a project-state transcript |
| local `.ai/PROJECT.md` | stable local project background, constraints, terminology, repository map | selected minimized shared fields only |
| local `.ai/CURRENT.md` | authoritative active-work state for the local workspace | promoted selectively, never mirrored automatically |
| local `.ai/TASKS.md`, DEC/EXP/handoffs/local sessions/research | detailed local lifecycle/evidence | local by default; promote derived conclusions only when policy permits |
| Hub `projects/<project-id>/PROJECT.md` | proposed last reconciled shared checkpoint for other agents/devices | shared-checkpoint projection/reconciliation only |
| Hub `projects/INDEX.md` | cross-project identity/routing registry | Hub-only routing role; not equivalent to local `.ai/INDEX.md` |
| Hub per-project `sessions/` | important shared session trace | only when cross-session traceability justifies publication |
| Hub shared `research/` / `decisions/` | explicitly promoted reusable/cross-project material | promotion requires scope/applicability metadata |

Planning principle:

> **Evidence and scope determine authority; neither local nor remote storage wins merely because of location.**

The detailed M1.1/M1.2 proposal now defines owner/writer/reader, publication direction, conflict behavior, provenance expectations, privacy/classification, retention/deletion semantics, field-policy inheritance, and the exact semantic scope of Hub checkpoint reconciliation. It remains non-normative until final approval and ADR promotion.

Evidence:

- `docs/proposals/UNIFIED_AUTHORITY_MODEL.md`
- `docs/reviews/2026-08-26-unified-authority-model-review.md`

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
- **H7 — M1 authority/boundary review.** Architecture direction was accepted with no blocking defect; six required clarifications were identified and integrated: roadmap evidence, field-policy inheritance, retention/deletion, P3 hard block, unclassified fail-closed default, and shared-checkpoint reconciliation scope. Final approval remains pending.

## 8. Planning decisions

These are planning-level decisions only. When approved as architecture, promote them to dedicated ADR/decision files and leave only a rollup here.

- **D-P1 — Do not build a second local-memory implementation.** Reuse/productize the existing local core.
- **D-P2 — Keep the Hub, but narrow its proposed unified role.** Registry/routing, shared checkpoint, cross-agent/device coordination, shared trace, remote concurrency, compatibility/migration.
- **D-P3 — Pre-write refresh remains mandatory.** Before writing shared mutable state, re-read/re-check latest state and reconcile rather than blindly overwrite. Remote Hub uses blob SHA where available; local multi-session semantics still need executable design/tests.
- **D-P4 — One canonical Skill release stream is required.** Manually synchronized installed/dev behavior copies are not acceptable long term.
- **D-P5 — Existing transport and future Hub adapter are separate semantic layers.** Mailbox/handoff/receipt/non-canonical snapshot behavior must not silently become authoritative Hub synchronization.
- **D-P6 — Shared reconciliation is semantic, not full-tree synchronization.** Local state is projected into a minimized shared-checkpoint candidate before B/R/L reconciliation; local source files remain outside the Hub merge domain.
- **D-P7 — Publication fails closed on privacy uncertainty.** P3 and UNCLASSIFIED material cannot enter Project Memory promotion; a safe derivative must be a new independently classified object.

## 9. Open architecture questions

Resolve before migration/implementation:

- Which repository becomes the canonical source for the **future unified** Skill?
- What reusable subset of the audited local distribution is imported first?
- What metadata records the last fully reconciled Hub state (`hub_project_sha` or equivalent)?
- What local concurrency primitive prevents lost updates when several Codex sessions write `.ai/CURRENT.md` / indexes?
- What exact executable three-way reconciliation mechanism implements the approved semantic B/R/L rules?
- What exact milestone/shared-state trigger causes a Hub publication?
- What privacy-classification/detection mechanism implements P0/P1/P2/P3/UNCLASSIFIED safely?
- How does web/new-device recovery hydrate context without making local active state remote-canonical?
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
| M1.1 | Authority model by information scope | ACTIVE | M0 | every information class has one explicit owner/role; final review accepts R1–R12 | `docs/proposals/UNIFIED_AUTHORITY_MODEL.md`; `docs/reviews/2026-08-26-unified-authority-model-review.md` | 2026-08-26 |
| M1.2 | Field-level local/Hub mapping | ACTIVE | M1.1 | owner, writer, readers, publish direction, conflict rule, provenance, privacy/classification, retention/deletion and reconciliation scope defined; final review accepts R1–R12 | `docs/proposals/UNIFIED_AUTHORITY_MODEL.md`; `docs/reviews/2026-08-26-unified-authority-model-review.md` | 2026-08-26 |
| M1.3 | Local multi-session safe-write semantics | PLANNED | M1.2 | lost-update prevention specified for shared local mutable files | — | — |
| M1.4 | Unified remote refresh/reconcile semantics | PLANNED | M1.2 | executable-oriented B/R/L three-way behavior deterministic within shared-checkpoint schema | — | — |
| M1.5 | Hub adapter vs transport boundary | PLANNED | M0.6 | mailbox/snapshot semantics cannot be confused with Hub checkpoint sync | — | — |
| M1.6 | Privacy / never-publish classes | PLANNED | M1.2 | classification/detection/purge behavior implements P0/P1/P2/P3/UNCLASSIFIED rules safely | — | — |
| M1.7 | Normal fast path vs recovery path | PLANNED | M1.2 | conditions and fallbacks explicit | — | — |
| M1.8 | Memory payload / compaction budgets | PLANNED | M1.7 | measurable budgets and thresholds defined | — | — |
| M1.9 | Unified Architecture Proposal approved | BLOCKED | M1.1–M1.8 | approved ADR/spec before source migration | — | — |

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
| M4.2 | Local safe concurrent-write design | PLANNED | M1.3 | CURRENT/index shared writes have deterministic no-lost-update semantics | — | — |
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
