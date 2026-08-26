---
document_role: architecture-proposal
status: review-required
normative: false
architecture_state: proposed-unreleased
runtime_load_policy: maintenance-only
milestones:
  - M1.1
  - M1.2
created: 2026-08-26
last_reviewed: 2026-08-26
review_state: clarifications-integrated-final-approval-pending
review_record: docs/reviews/2026-08-26-unified-authority-model-review.md
evidence_baseline:
  - docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md
  - tests/results/v0.1.0-static-regression.md
---

# Unified Project Memory — Authority Model and Field-Level Boundary Proposal

## 0. Purpose and status

This document proposes the authority model for the future unified Project Memory Skill. It covers M1.1 (authority by information scope) and M1.2 (local-memory versus Hub-checkpoint boundary field-by-field).

It is **not released behavior**. Current `project-memory-hub-skill 0.1.0`, the currently deployed local `project-memory` implementation, and Runtime Hub schema 1 continue to behave according to their existing contracts until an explicit release/migration changes them.

This proposal must not be loaded during ordinary project startup.

Review state after the first architecture review:

- architecture direction: acceptable;
- blocking defects: none identified;
- M1.1 / M1.2: active, not DONE;
- six required clarifications have been integrated here;
- final approval / ADR promotion remains pending.

Evidence: `docs/reviews/2026-08-26-unified-authority-model-review.md`.

## 1. Terms

### 1.1 Authority

`Authority` means the designated primary mutable representation for a semantic information class. It does **not** mean a Markdown record outranks direct evidence.

When a memory record conflicts with directly observed source code, files, generated outputs, logs, or reproducible experiment evidence, the direct evidence takes precedence and the memory record must be corrected with provenance.

### 1.2 Local memory

The workspace `.ai/` tree managed by the mature local Project Memory core. It holds high-frequency working state and detailed project evidence/history.

### 1.3 Shared checkpoint

A compact, low-frequency representation in the Runtime Hub containing state that another authorized agent/device should be able to recover without receiving the full local memory tree.

The target architecture narrows the future semantic role of Hub `projects/<project-id>/PROJECT.md` from generic "current state" to **shared checkpoint**. The filename may remain `PROJECT.md`; the semantic role is what changes after a future approved migration.

### 1.4 Promotion

Creating a shared summary/reference from local evidence. Promotion is not file mirroring. A promoted record remains derived from its local evidence and must carry sufficient provenance to explain its origin.

### 1.5 Reconciliation

Combining a local proposed shared update with intervening remote shared changes without blindly overwriting either side. Reconciliation is evidence-aware and may fail closed when a semantic conflict cannot be safely resolved.

### 1.6 Shared projection

A minimized candidate containing only semantic fields that belong to the shared-checkpoint schema. It is derived from local state/evidence before Hub reconciliation. It is not a copy of the full local `.ai/` tree.

## 2. Proposed authority principles

1. **Evidence precedes memory.** Raw/reproducible workspace evidence is the factual backstop.
2. **Authority is scoped by semantic class.** Local and remote stores may both be authoritative, but not for the same semantic state in the same scope.
3. **Local memory owns active work.** High-frequency current work, tasks, detailed experiments, project decisions, retrieval indexes, and local handoff state remain local by default.
4. **The Hub owns shared coordination state.** The Hub registry owns remote multi-project identity/routing; the Hub project record owns the last reconciled shared checkpoint.
5. **No full local↔remote mirror.** `CURRENT.md`, `TASKS.md`, DEC/EXP trees, and indexes are not copied wholesale into Hub `PROJECT.md`.
6. **Promotion is explicit and lossy by design.** The Hub receives the stable conclusion/checkpoint needed by other agents, not every intermediate detail.
7. **Writes to shared mutable state require refresh/reconcile.** Remote writes use the latest remote state/blob SHA when available. Local multi-session safe-write mechanics remain a separate M1.3/M4 design task.
8. **Identity conflicts fail closed.** A mismatch in project identity or workspace binding must not be auto-merged.
9. **Privacy constrains promotion.** Unclassified or prohibited information cannot be published merely because it is useful context.
10. **Derived copies never silently become owners.** A cached, hydrated, sanitized, or summarized copy does not inherit authority from its source unless a released policy explicitly assigns it.
11. **Absence is not deletion.** Omitting a field from a compact checkpoint or derivative must not silently delete the owning source state.

## 3. Proposed high-level ownership map

| ID | Information class | Primary authority | Main writers | Typical readers | Default publish direction | Conflict behavior | Provenance requirement | Privacy default | Retention default |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A01 | Raw source code, datasets, generated outputs, raw logs, direct artifacts | Workspace evidence itself | project tools / user / agent | local agents; explicit reviewers | none automatically | evidence overrides memory summaries | direct path/hash/tool/run context as applicable | UNCLASSIFIED until known; often P2/P3 | governed by project/evidence policy; never deleted by memory reconciliation |
| A02 | Shared stable `project_id` | Hub `projects/INDEX.md` once Hub-bound | Hub registration workflow | AGENTS binding, Hub routing, agents | Hub → workspace binding reference | mismatch = stop; no automatic merge | registration/audit/round-trip evidence | P1 | persistent until explicit migration/de-registration |
| A03 | Workspace-local Hub binding (`project_id`, hub location, local memory location) | root `AGENTS.md` binding block for that workspace | bootstrap/migration workflow | Codex/Skill startup | Hub identity → local binding; local observed root may be reported back | mismatch with Hub registry = stop and review | direct workspace audit / binding evidence | P1/P2 | persistent binding; unrelated instructions preserved |
| A04 | Stable local project background: detailed goals, constraints, terminology, repository map, durable local environment assumptions | `.ai/PROJECT.md` | local Project Memory core | local startup / selective retrieval | selected summaries may be promoted | do not overwrite from Hub wholesale; reconcile only promoted semantic subset | local evidence/DEC/EXP/research references where claims require support | P2 | current stable local record may be rewritten; supporting history remains in evidence/history records |
| A05 | Active objective, current route, latest local result, immediate blocker, next local actions | `.ai/CURRENT.md` | active local sessions | local startup | selected milestone/blocker/conclusion may be promoted as summary | local safe-write rules; Hub checkpoint never blindly overwrites | reference supporting evidence for durable conclusions | P2 | rewritable hot state; superseded detail may move to session/EXP/DEC history |
| A06 | Detailed task lifecycle / definition of done | `.ai/TASKS.md` | local sessions | local retrieval | milestone-level subset only | local merge semantics; no Hub task mirror | acceptance/result reference when task closes | P2 | retain lifecycle per local policy; omission from Hub has no deletion meaning |
| A07 | Local retrieval routing/index | `.ai/INDEX.md` and subordinate indexes | local Project Memory core | local retrieval | none | local safe-write rules | links to indexed records | P2 | derived/rebuildable index; source records remain authoritative |
| A08 | Detailed experiment record and reproducibility evidence | `.ai/experiments/EXP-*.md` + referenced raw evidence | local sessions/tools | local selective retrieval | stable conclusion may be promoted; full EXP not automatic | evidence-preserving; never replaced by a Hub summary | experiment inputs/method/result/artifact references | UNCLASSIFIED until known; normally P2, potentially P3 | preserve/archive/supersede; destructive purge only by explicit owning policy |
| A09 | Project-local decision lifecycle (including supersession) | `.ai/decisions/DEC-*.md` | local sessions | local selective retrieval | consequential/shared decision may be promoted to a distinct Hub decision or checkpoint summary | local DEC remains source record; shared derivative links back | decision rationale + related evidence + supersession links | P2 | preserve; use superseded/supersedes rather than silent deletion |
| A10 | Local session history | `.ai/sessions/` when used | local sessions | local selective retrieval | important cross-agent trace may be summarized into Hub session | derivative summary must not replace local source history | session/time/task references | P2 | append/archive by default; explicit purge only |
| A11 | Local handoff inbox/outbox/processed state and mailbox receipts | local handoff/transport subsystem | transport + local agents | local/peer agents | transport channel only; not Hub PROJECT mirroring | transport-specific hash/state rules | message IDs/hashes/receipts | UNCLASSIFIED until known; P2/P3 common | transport lifecycle policy controls archive/processed state |
| A12 | Local research notes | `.ai/research/` when used | local sessions | local selective retrieval | explicitly reusable finding may be promoted to Hub `research/` | promoted record is a scoped derivative with provenance | source links / evidence status | P2 | preserve/archive; shared derivative deletion does not delete source |
| A13 | Last reconciled cross-agent shared project checkpoint | Hub `projects/<project-id>/PROJECT.md` under future unified semantics | Hub adapter / authorized remote agent | web ChatGPT, other devices, local Hub adapter | local shared projection → Hub; Hub → local as shared input/reference | remote pre-write refresh + scoped three-way reconcile; unresolved semantic conflict fails closed | source record IDs/shared record links/validation status sufficient for review | P1 only; P3 and UNCLASSIFIED blocked | latest checkpoint rewritable/versioned; field omission is not source deletion |
| A14 | Cross-project registry/routing metadata | Hub `projects/INDEX.md` | Hub registration/migration workflows | Hub routing | Hub → agents/workspaces | identity conflict fails closed | binding/audit status | P1 | persistent; changes require explicit registration/migration workflow |
| A15 | Important shared session trace | Hub `projects/<project-id>/sessions/` | authorized agent/session | cross-agent audit/recovery | local/session → Hub only when durable cross-agent value exists | append-oriented; does not replace checkpoint | base/final checkpoint refs when applicable | P1/P2 only after classification/promotion | append/archive; no implicit deletion from checkpoint changes |
| A16 | Reusable cross-project research | Hub `research/` | explicit promotion workflow | linked projects | local → Hub only when reusable and permitted | scope/applicability must be explicit | source/provenance + applicability | P1 | preserve/archive/supersede explicitly |
| A17 | Consequential shared/cross-project decision | Hub `decisions/` | explicit promotion/decision workflow | linked projects | local/shared proposal → Hub after approval | applicability and supersession explicit; do not duplicate local DEC semantics blindly | rationale/source/applicability/supersession | P1 | preserve; supersede instead of silent delete |
| A18 | Hub synchronization metadata (last reconciled SHA, remote path/schema, timestamps) | local Hub-adapter metadata store (exact schema deferred to M3.1) | Hub adapter | Skill/adapter only | metadata only | stale base triggers reconciliation; never treated as project fact | remote identity/SHA/schema/time | P0/P1 | replaceable operational metadata; not project-history evidence |

## 4. Field-policy inheritance

Policy inheritance is **dimension-specific**. There is no single precedence rule that may weaken safety properties.

For ordinary semantic ownership/configuration, the most specific valid rule refines the broader rule:

```text
semantic-field policy
  > record-type policy
  > file/container policy
  > system fallback
```

The following constraints override that general refinement model:

1. **Explicit prohibition wins.** A narrower rule cannot relax a broader explicit `deny`, `never-publish`, identity constraint, or destructive-operation guard.
2. **Privacy uses the most restrictive applicable classification.** Any applicable P3 classification makes the candidate non-publishable. `UNCLASSIFIED` also blocks publication until classification completes.
3. **Provenance requirements are monotonic.** A more specific rule may add provenance requirements but cannot silently remove provenance required by a broader policy.
4. **Evidence/history retention floors are monotonic.** A child field or compact derivative cannot shorten an owning record's required retention merely by omission or a weaker default.
5. **Authority cannot be reassigned implicitly.** A derivative, cache, mirror, or promoted summary cannot become owner solely because it is more specific or newer.

If inherited rules remain ambiguous or contradictory after applying these constraints, the operation must fail closed and require an explicit policy decision.

## 5. Privacy / publication classes

These are proposal-level classes for M1.2/M1.6 integration. M1.6 will define the final policy and detection rules.

| Class | Meaning | May enter public Skill source? | May enter private Runtime Hub? | Default behavior |
| --- | --- | --- | --- | --- |
| P0 `STRUCTURAL` | schema/version/hash/timestamps/synthetic metadata with no private project content | yes when generic/synthetic | yes | freely reusable |
| P1 `RUNTIME_SHAREABLE` | project-specific state intentionally needed for authorized cross-agent/device coordination | no, except sanitized synthetic examples | yes | may be promoted to Hub |
| P2 `LOCAL_BY_DEFAULT` | detailed project state, local paths, task/experiment/decision detail not needed remotely | no | only after explicit classification/promotion/minimization | remain local |
| P3 `PROHIBITED_SYNC` | credentials, tokens, secrets, sensitive personal/private material, or records explicitly marked never-publish | never | never via Project Memory promotion | hard-block publication |

### 5.1 Unclassified default

`UNCLASSIFIED` is a provisional state, not a publishable privacy class:

```text
classification: UNCLASSIFIED
effective_storage_policy: local-only
publish_allowed: false
```

Unknown or missing classification therefore fails closed for Project Memory promotion. Classification must complete before a candidate may become P0/P1/P2/P3 for publication-policy purposes.

### 5.2 P3 hard-block rule

`Private Runtime Hub` does not make P3 publishable. P3 is a semantic prohibition, not merely a repository-visibility rule.

If **any P3 material is present in a candidate publication payload**, that publication transaction fails. The system must not silently redact a fragment from the same payload and continue as though the P3 source had been downgraded.

A shareable result may instead be produced through an explicit derivation boundary:

```text
source record containing P3
        ↓ explicit derivation/sanitization
new independent derivative containing no P3
        ↓ classify derivative on its own contents
P1 (or other allowed class)
        ↓ normal promotion path
```

The derivative must not expose prohibited provenance. It may carry only provenance that is itself permitted to leave the workspace.

## 6. Retention / deletion semantics

Retention is separate from authority and synchronization. The target architecture must never infer destructive deletion from ordinary reconciliation or compaction.

### 6.1 Core rules

- `supersede` is not `delete`;
- omission from a compact checkpoint is not deletion;
- removal of a derived Hub summary does not delete its local source evidence;
- a remote deletion does not imply deletion of local evidence/history;
- a local record deletion does not automatically delete a previously published shared derivative;
- absence of a field in a later checkpoint is not an implicit delete instruction;
- compaction changes representation size, not evidence ownership.

### 6.2 Explicit deletion

A destructive deletion/purge requires an **explicit operation** governed by the policy of the owning record. The operation must identify what semantic object is being deleted and whether any independent derivatives remain.

Evidence/history classes such as DEC, EXP, session, research, and important shared decisions default to **preserve, archive, or supersede** rather than silent deletion.

Privacy-driven purge requirements, if needed, must be designed explicitly in M1.6/M5 and may impose stronger deletion behavior than ordinary retention rules. This proposal does not define a general secure-erasure mechanism.

## 7. Field-level target boundary

### 7.1 Root `AGENTS.md` — target responsibilities

`AGENTS.md` should remain a small workspace binding/invocation layer, not a project-state database.

| Field / content | Target role | Owner | Hub relationship |
| --- | --- | --- | --- |
| Project Memory Skill invocation/discovery instruction | bind workspace to Skill behavior | workspace instructions | no project-state mirroring |
| `project_id` | local binding reference to shared stable ID | Hub registry owns shared ID; AGENTS owns the binding statement in this workspace | must match Hub registry when bound |
| Hub repository/entry point | connection metadata | AGENTS binding | points to Hub; not project state |
| Local memory root (normally `.ai/`) | local storage binding | AGENTS / Skill defaults | local only |
| Existing unrelated workspace instructions | preserved verbatim | workspace/user | never replaced by Project Memory bootstrap |

Not allowed in the target AGENTS binding block: active objectives, experiment results, task lists, shared checkpoint prose, or large history.

### 7.2 Local `.ai/PROJECT.md` — target responsibilities

This file owns **stable local project background**, not high-frequency current work and not the shared Hub checkpoint.

| Semantic field | Local role | Promotion rule |
| --- | --- | --- |
| local project name / local identity context | stable human context | compact alias/name may be shared through Hub registry/checkpoint |
| goals / success criteria | detailed local formulation | Hub receives only stable shared goal needed for recovery |
| scope / exclusions | detailed local boundary | shared subset may be promoted when needed for routing/coordination |
| constraints / invariants | durable local constraints | promote only constraints another agent must obey |
| terminology | local durable glossary | normally local; promote only if required for shared interpretation |
| repository/workspace map | local detailed map | do not mirror wholesale; publish only essential shared references |
| environment assumptions | durable local environment context | publish only when another agent/device needs it and privacy allows |
| stable technical background | local durable context | promote distilled validated conclusions, not full history |
| pointers to local DEC/EXP/research | local retrieval links | Hub may receive permitted record IDs/provenance summaries, not automatic file copies |

`.ai/PROJECT.md` must not become a cache of the entire Hub `PROJECT.md`.

### 7.3 Local `.ai/CURRENT.md` — target responsibilities

This file owns the **active local work state**.

| Semantic field | Local authority | Hub promotion |
| --- | --- | --- |
| active objective | yes | only if it becomes the shared current milestone |
| active route/approach | yes | only when adopted/stable enough that another agent must know |
| latest local result | yes | promote validated conclusion, not every intermediate result |
| immediate blocker | yes | promote when blocker affects shared project coordination |
| next local actions | yes | normally local; Hub receives next shared milestone, not micro-task list |
| unresolved local questions | yes | normally local; promote only shared blocker/question |
| last local verification context | yes | may support provenance; not mirrored wholesale |

A newer Hub checkpoint must not directly overwrite these fields. It is an input to reconciliation; the local active plan changes only after the agent incorporates the shared update and preserves relevant local evidence.

### 7.4 Local `TASKS`, `INDEX`, DEC, EXP, sessions, research, handoffs

| Local record | Authority | Default Hub behavior |
| --- | --- | --- |
| `TASKS.md` | detailed local task lifecycle | no mirror; publish milestone/blocker summaries only |
| `INDEX.md` / subordinate indexes | local retrieval routing | never mirror into Hub registry |
| `DEC-*` | project-local decision lifecycle | promote only consequential/shared decision as distinct derived Hub record or checkpoint summary |
| `EXP-*` | detailed experiment/reproducibility record | never automatic full promotion; publish stable conclusion + permitted provenance pointer when needed |
| local sessions | local detailed work trace | publish only cross-agent durable trace as a Hub session summary |
| local research | project-local research | promote only reusable/shared finding |
| handoff inbox/outbox/processed | mailbox workflow | keep separate from Hub checkpoint; existing transport semantics remain distinct |

### 7.5 Hub `projects/INDEX.md` — target responsibilities

This remains the **remote multi-project registry**.

Target contents include only routing/identity metadata needed to select one project safely, such as:

- stable `project_id`;
- canonical project display name / aliases;
- integration status;
- verified workspace bindings needed for routing;
- project record path;
- concise scope/exclusion boundary.

It must not absorb local retrieval indexes, experiment indexes, task indexes, or detailed project state.

### 7.6 Hub `projects/<project-id>/PROJECT.md` — target responsibilities

Under the future unified architecture, this file becomes the **last reconciled shared checkpoint**, not a mirror of `.ai/PROJECT.md + CURRENT.md`.

| Shared field | Meaning | Source/promotion rule |
| --- | --- | --- |
| `project_id` | stable shared identity | Hub registry; must agree with binding |
| shared status / integration status | cross-agent lifecycle state | Hub workflows |
| shared goal | compact goal required for remote recovery | promoted from stable local goal; minimized |
| shared scope / exclusions | enough boundary to prevent project blending | registry/local scope; minimized |
| `shared_checkpoint` | compact statement of last reconciled stable shared project state | explicit promotion/reconciliation only |
| adopted shared route | technical/operational route another agent should rely on | promote only after sufficiently stable/validated |
| validated shared conclusions | conclusions safe and useful to other agents | derived from local evidence with permitted provenance |
| shared blockers | blockers affecting coordination/milestone | promoted from local current/tasks when warranted |
| next shared milestone | next cross-agent meaningful target | derived from local plan, not micro tasks |
| provenance references | enough permitted source identity to understand/review the checkpoint | record IDs, relative references, validation status; no P3 material |
| shared session/research/decision links | explicit links to durable shared records | Hub-native references only |
| uncertainty / historical notes | prevent stale/unverified claims being mistaken for current facts | explicit provenance/status |

Fields deliberately excluded from automatic Hub checkpoint mirroring:

- full `.ai/CURRENT.md`;
- full `.ai/TASKS.md`;
- local retrieval index contents;
- every DEC/EXP record;
- full raw logs/artifacts;
- local handoff mailbox state;
- detailed local environment/path inventory unless specifically required, classified, and permitted.

The future template should avoid an ambiguous generic heading such as `Current state`; `Shared checkpoint` is the preferred semantic label.

## 8. Promotion rules

A local fact may be promoted to the Hub only when all applicable conditions hold:

1. it has durable value beyond the active local session;
2. another authorized agent/device plausibly needs it for recovery, coordination, or a shared milestone;
3. it is sufficiently validated or explicitly marked uncertain/historical;
4. it is classified and publication is allowed by the applicable privacy rules;
5. promotion minimizes detail while preserving meaning and permitted provenance;
6. the remote target has been refreshed before mutation;
7. any intervening remote change has been reconciled within the shared-checkpoint scope.

Promotion creates a **derived shared representation**. It does not transfer ownership of the detailed local source record.

P3 and UNCLASSIFIED material fail the promotion gate before the remote write path.

## 9. Hydration / remote-to-local rules

When a local workspace observes a newer Hub checkpoint:

1. do not replace `.ai/PROJECT.md` or `.ai/CURRENT.md` wholesale;
2. identify which shared semantic fields changed since the last reconciled Hub state;
3. compare those changes with local active state and evidence;
4. incorporate compatible shared changes into local work as needed;
5. preserve local DEC/EXP/session evidence even when a local conclusion becomes obsolete;
6. record unresolved semantic conflicts explicitly and stop any conflicting shared write;
7. update sync metadata only after reconciliation is complete.

A hydrated/cache copy of Hub state must be clearly non-authoritative. The exact adapter metadata/cache format is deferred to M3.1.

## 10. Shared-checkpoint reconciliation scope

Three-way Hub reconciliation is deliberately narrower than local memory reconciliation. It operates only on semantic fields defined by the **shared-checkpoint schema**.

Before reconciliation, local detailed state is projected into a minimized shared candidate `L`.

The three inputs are:

```text
B = last fully reconciled Hub checkpoint (base)
R = latest remote Hub checkpoint after refresh
L = local proposed shared-checkpoint candidate/projection
```

The reconciliation algorithm compares **B/R/L field-by-field only for shared-checkpoint fields** and produces either a reconciled new checkpoint or an explicit unresolved conflict.

The following local sources are outside the Hub merge domain:

- full `.ai/PROJECT.md`;
- full `.ai/CURRENT.md`;
- full `.ai/TASKS.md`;
- local `INDEX.md` / subordinate indexes;
- DEC/EXP source records;
- raw evidence/artifacts;
- local session history;
- local research source records;
- handoff/transport state.

They may supply evidence used to construct `L`, but the Hub adapter must not treat them as files to merge with `R`.

A semantic field removed from `L` is not automatically a deletion request. Explicit clearing/deletion must be represented as an intentional operation allowed by the field/retention policy.

## 11. Conflict classes and proposed behavior

### C1 — project identity/binding conflict

Examples: AGENTS `project_id` differs from Hub registry, or the observed workspace is bound to another project.

**Behavior:** fail closed. Do not merge project state and do not rewrite binding automatically.

### C2 — raw evidence contradicts memory

**Behavior:** direct/reproducible evidence wins. Correct the owning memory record with provenance; do not preserve a false current claim merely for consistency.

### C3 — local active state differs from unchanged Hub checkpoint

This is not automatically a conflict because the scopes differ.

**Behavior:** keep local active state. Promote only when it reaches a shared publication trigger.

### C4 — Hub checkpoint changed, no local pending shared update

**Behavior:** hydrate/reconcile the changed shared fields into local work as needed; preserve local evidence/history.

### C5 — local pending shared update, Hub unchanged from last reconciled SHA

**Behavior:** refresh remote, then publish the minimized/classified shared projection and record the new reconciled remote identity.

### C6 — local pending shared update and Hub both changed since the same base

**Behavior:** perform three-way semantic reconciliation over the shared-checkpoint schema using B/R/L as defined in §10. Preserve compatible changes from both sides. If the same semantic field conflicts and evidence/precedence does not resolve it safely, record an explicit conflict and do not publish an overwrite.

### C7 — privacy/classification conflict

A candidate shared update contains P3 material or any UNCLASSIFIED material.

**Behavior:** hard-block the publication transaction. Do not silently weaken a classification or continue by redacting the same candidate payload. If a safe derivative is needed, create a new independently sanitized derivative, classify it on its own contents, and run it through the normal promotion gate.

### C8 — ambiguous deletion/clearing

A local or remote candidate omits a previously present shared field or appears to remove source history without an explicit delete operation.

**Behavior:** treat omission as non-deletion. Preserve the existing semantic value/source history until an explicit clearing/deletion action satisfies the owning policy.

## 12. Directionality constraints

The target model is intentionally **not** generic bidirectional file synchronization.

Allowed patterns:

- local detailed evidence → compact classified Hub summary/reference;
- Hub shared checkpoint → reconciliation input for local work;
- Hub registry identity → workspace binding reference;
- local observed binding evidence → explicit Hub registry update through registration/audit workflow;
- local reusable research/decision → explicitly promoted shared record.

Disallowed default patterns:

- Hub PROJECT → overwrite local PROJECT/CURRENT;
- local PROJECT/CURRENT → overwrite Hub PROJECT wholesale;
- local INDEX → Hub project registry;
- every EXP/DEC/session → automatic Hub copy;
- transport CURRENT snapshot → authoritative Hub checkpoint;
- omission from a checkpoint → implicit deletion of local or remote evidence;
- P3/UNCLASSIFIED candidate → Hub publication.

## 13. Startup implications (boundary only; detailed fast-path design remains M1.7/M3)

If this proposal is approved, a healthy bound local startup should conceptually prefer:

```text
Skill / AGENTS binding
      ↓
local PROJECT + CURRENT + INDEX
      ↓
lightweight Hub change check
      ↓
unchanged → continue locally
changed   → fetch changed shared checkpoint → reconcile
```

This proposal does not yet define the exact sync metadata file, probe API, token budget, or cache representation. Those remain M1.7/M1.8/M3 tasks.

A web-only/new-device agent without local `.ai/` may recover from the Hub shared checkpoint, but must describe that state as the **last shared checkpoint**, not claim knowledge of the latest unsynchronized local active work.

## 14. Proposed invariants

The unified implementation should be rejected if it violates any of these invariants:

1. one semantic information class has no clear owner;
2. two mutable records can independently claim authority over the same semantic scope without a reconciliation contract;
3. a Hub update can erase newer local evidence/history;
4. a local update can erase an intervening remote shared change without detection;
5. a derived/cached copy silently becomes authoritative;
6. project identity mismatch can be auto-merged;
7. P3 or UNCLASSIFIED data can enter Project Memory promotion;
8. detailed EXP/DEC/task/index data is mirrored to Hub by default;
9. normal local startup requires loading this architecture/governance document;
10. a remote-only agent represents the Hub checkpoint as guaranteed latest local work;
11. field-policy inheritance can silently relax an explicit safety prohibition;
12. checkpoint omission can silently delete an owning source record;
13. Hub three-way reconciliation operates over full local files instead of the shared-checkpoint schema.

## 15. M1.1 / M1.2 review decisions required

Approval of this proposal requires explicit agreement or revision on these points:

- **R1:** Authority is semantic-scope-based, not globally local-first or Hub-first.
- **R2:** `.ai/PROJECT.md` owns stable detailed local background; `.ai/CURRENT.md` owns active local work.
- **R3:** Hub `PROJECT.md` becomes the future last reconciled **shared checkpoint**, not a full local memory mirror.
- **R4:** Hub `projects/INDEX.md` remains the shared multi-project routing authority; local `INDEX.md` remains unrelated local retrieval routing.
- **R5:** `AGENTS.md` remains a small invocation/binding layer and does not store project state.
- **R6:** EXP/DEC/tasks/local sessions/research/handoffs remain local by default and are promoted selectively as derived summaries/records.
- **R7:** Remote-to-local hydration never overwrites local PROJECT/CURRENT wholesale.
- **R8:** Promotion requires classification, privacy, provenance, minimization, and pre-write remote refresh; P3/UNCLASSIFIED candidates hard-fail publication.
- **R9:** Identity conflicts fail closed; local+remote shared divergence uses scoped B/R/L three-way semantic reconciliation.
- **R10:** Policy inheritance is field-specific but cannot relax explicit prohibitions, provenance floors, privacy restrictions, or evidence/history retention floors.
- **R11:** Supersession/compaction/omission are not deletion; destructive deletion requires an explicit owning-policy operation.
- **R12:** Exact sync metadata, local concurrent-write mechanics, publication triggers, privacy detection implementation, and performance budgets remain separate later milestones rather than being prematurely fixed here.

If a final review finds no new blocking defect and accepts R1–R12, M1.1/M1.2 may be approved and promoted to a dedicated ADR/decision record. M1.3+ concerns must not be pulled into these milestones merely to delay approval.

## 16. Evidence and relationship to current released behavior

Evidence baseline:

- `docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md`
- `tests/results/v0.1.0-static-regression.md`
- `docs/reviews/2026-08-26-unified-authority-model-review.md`
- current Hub `skill/SKILL.md`, `docs/architecture.md`, `templates/PROJECT.md`, and `templates/AGENTS_BINDING.md`

Current released Hub `0.1.0` still describes remote `PROJECT.md` as authoritative current project state and instructs bound workspaces to read it. This proposal intentionally does **not** modify those files yet. If approved, changing that behavior will require an explicit architecture decision, compatibility review, migration plan, template/spec updates, and post-change validation.
