# Project Memory Skill — History, Goals, Plan, and Status

> Status: active planning document
>
> This file is the persistent project-management record for the **Project Memory Skill / Hub integration effort**. It records history, goals, architectural direction, completed work, open questions, tasks, milestones, and acceptance criteria.
>
> It is **not** the normative behavior contract. Normative behavior remains in `skill/SKILL.md` and the relevant policy/schema documents. When this plan and a released specification differ, the released specification wins until an explicit migration/release updates it.

## 1. Project goal

Build one maintainable Project Memory system for ChatGPT/Codex-style agents that:

- preserves useful project state across fresh chats and context compaction;
- keeps high-frequency working memory local to the workspace;
- supports selective retrieval rather than loading all history;
- preserves experiment, decision, task, handoff, session, and provenance history;
- supports deterministic multi-project routing and strict project isolation;
- provides a low-frequency shared/durable Hub for cross-session, cross-agent, and cross-device coordination;
- prevents stale sessions from silently overwriting newer shared state;
- has one canonical source repository, explicit releases, migrations, executable validation, and regression tests;
- minimizes normal startup memory payload and avoids redundant local/remote injection.

## 2. Current architectural direction

The project is converging on a **single Skill with two memory domains**, not two competing Skills.

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

### 2.1 Authority model

Authority is divided by scope rather than by declaring either local or remote storage globally superior.

| Information class | Intended authority |
| --- | --- |
| Source code, datasets, generated outputs, raw logs, direct experiment artifacts | Workspace evidence itself |
| Active objective, current route, latest local result, immediate blockers, near-term actions | local `.ai/CURRENT.md` |
| Detailed project tasks, experiments, decisions, handoffs, local sessions, local research | local `.ai/` records and indexes |
| Shared project identity and last stable cross-agent checkpoint | Hub `projects/<project-id>/PROJECT.md` |
| Cross-project/project registry and workspace routing | Hub `projects/INDEX.md` |
| Important shared session trace | Hub `projects/<project-id>/sessions/` |
| Reusable cross-project findings / consequential cross-project decisions | Hub shared `research/` / `decisions/` when explicitly promoted |

The key rule is:

> **Evidence and scope determine authority; neither local nor remote storage wins merely because of location.**

### 2.2 Frequency model

- **Local memory is high-frequency.** Codex may update local working-memory records after ordinary substantive tasks and experiments.
- **Hub memory is low-frequency.** The Hub should receive stable milestones, shared blockers, adopted technical routes, handoff-worthy state, and other information that another agent/device genuinely needs.
- Detailed `EXP-*`, local decision mechanics, transient debugging, and every task result must not be copied automatically into the Hub.

## 3. What exists today

### 3.1 Existing local `project-memory` implementation

A deployed local implementation identified during the read-only audit is materially more mature than the initial Hub Skill reference implementation.

Audited reusable capabilities include:

- implicit Codex discovery metadata;
- LOAD / RETRIEVE / CHECKPOINT workflows;
- local `.ai/` schema with `PROJECT.md`, `CURRENT.md`, `INDEX.md`, `TASKS.md`;
- indexed decisions and experiments;
- handoff inbox/outbox/processed lifecycle;
- optional sessions and research;
- retrieval, write, handoff, transport, schema, and architecture policies;
- `memory_tool.py` validation/status/ID-allocation tooling;
- `transport_tool.py` mailbox/receipt/snapshot transport with defensive Git/hash checks;
- executable smoke and transport E2E tests.

The installed and development/test copies inspected by the audit were byte-for-byte identical at audit time. The audit found local implementation version `1.1.0` through repository/tool version markers.

### 3.2 `project-memory-hub-skill`

Current public source repository release:

- Skill version: `0.1.0`
- Supported Hub schema: `1`
- Status: early reference implementation

Implemented protocol concepts include:

- deterministic workspace → `project_id` routing;
- strict project isolation;
- remote project registry;
- bound-workspace lifecycle and fresh-chat round-trip validation;
- Hub `PROJECT.md` and optional Hub session logs;
- blob-SHA pre-write refresh;
- optimistic stale-session reconciliation;
- runtime schema/version/migration model;
- synthetic protocol tests and a defined live-concurrency test procedure.

### 3.3 Runtime Hub

The current Runtime Hub uses `hub_schema_version: 1` and already supports:

- `START_HERE.md` routing/protocol entry;
- `projects/INDEX.md` as the sole remote project registry;
- isolated per-project `PROJECT.md` records;
- optional durable `sessions/`;
- pre-write refresh and stale-SHA handling;
- shared research/decision areas;
- fresh-chat round-trip validation for bound workspaces.

### 3.4 Existing transport

The local `transport_tool.py` is **not** the same thing as a Hub synchronization adapter.

Its current semantics are mailbox/handoff/receipt/non-canonical snapshot exchange. In particular, its CURRENT snapshot is explicitly non-canonical. It should not be silently repurposed into authoritative Hub synchronization.

## 4. Important history

### Stage H0 — local project-memory system created

A local file-based Project Memory system was built around `.ai/`, Markdown, Git-aware helper scripts, retrieval/write policies, experiments, decisions, and handoffs. It became the practical high-frequency memory used by Codex.

### Stage H1 — multi-project Runtime Hub introduced

A separate GitHub Runtime Hub was introduced to solve problems the local single-project memory did not target directly:

- deterministic workspace/project identity;
- cross-workspace project isolation;
- shared durable state across ChatGPT/Codex/devices;
- fresh-session recovery through a remote record;
- remote optimistic-concurrency protection.

### Stage H2 — Hub Skill source repository created

`project-memory-hub-skill` was created so Hub protocol, templates, tests, migration rules, and release history could be maintained independently from runtime project data. Version `0.1.0` was established as an early reference implementation supporting Hub schema 1.

### Stage H3 — session and remote concurrency protocol added

The Hub protocol added:

- session logs;
- `base_project_sha` semantics;
- mandatory remote pre-write refresh;
- stale-session detection;
- reconcile-before-write rules.

This remains an important design direction.

### Stage H4 — real use exposed local/remote duplication

Real Codex usage showed that local `.ai/` is updated more frequently and at finer granularity than the Hub. Loading local `CURRENT/INDEX` and then unconditionally loading remote `PROJECT.md` can duplicate state and waste context.

This produced the Hot/Durable direction:

- local `.ai/` = working/hot memory + detailed evidence;
- Hub = shared durable checkpoint + routing/coordination;
- remote full reads should eventually become lazy/conditional rather than unconditional.

### Stage H5 — strict read-only audit performed

A read-only audit compared the deployed local implementation, its development copy, `project-memory-hub-skill`, the Hub protocol, and a representative `.ai/` schema.

Key findings:

1. The local implementation already contains many capabilities that must not be reimplemented independently in the Hub Skill.
2. The Hub has important unique capabilities: multi-project routing, remote shared checkpointing, remote stale-SHA concurrency, schema/migration rules.
3. If both systems independently declare overlapping `PROJECT/CURRENT` state canonical, a dual-source-of-truth problem results.
4. The correct next step is **unification/design**, not immediate `0.2.0` feature implementation.

## 5. Decisions already considered stable enough for planning

These are planning-level decisions. They become normative only when incorporated into released specifications.

### D-P1 — Do not build a second local-memory implementation

Reuse and productize the existing local `project-memory` core instead of recreating LOAD/RETRIEVE/CHECKPOINT, experiment, decision, handoff, indexing, and local validation behavior in parallel.

### D-P2 — Keep the Hub, but narrow its role

The Hub remains valuable for:

- project registry and routing;
- shared stable checkpoint;
- cross-agent/device coordination;
- important shared session trace;
- remote optimistic concurrency;
- migration/version compatibility.

It should not mirror every local experiment or every change to hot working state.

### D-P3 — Pre-write refresh remains mandatory

The earlier concurrency principle remains valid and should be generalized:

> **Before writing any shared mutable state, re-read/re-check the latest state and reconcile changes rather than blindly overwriting.**

For remote Hub files this uses blob SHAs when available. Local multi-session concurrency requires an equivalent local-safe-write design and must be specified/tested before being claimed solved.

### D-P4 — One canonical Skill source is required

Long-term, installed Skill copies must come from one source-controlled release stream. Maintaining multiple manually synchronized behavior implementations is not acceptable.

### D-P5 — Existing transport and future Hub adapter are different layers

Mailbox/handoff transport may remain a component, but Hub synchronization must have separate semantics and must not inherit the non-canonical snapshot model by accident.

## 6. Open design questions

These must be resolved before implementation/migration begins.

- Which repository becomes the final canonical Skill source, and should the current repository eventually be renamed?
- What exact subset of the local `1.1.0` implementation is migrated into source control first?
- What is the precise boundary between local `.ai/PROJECT.md` and Hub `PROJECT.md`?
- Should local `.ai/PROJECT.md` remain stable identity/constraints while Hub `PROJECT.md` becomes a compact shared checkpoint, or should one be renamed/restructured during a future migration?
- What metadata records the last reconciled Hub state (`hub_project_sha`, sync time, schema, remote path)?
- What is the correct local concurrency primitive when several Codex sessions write the same `.ai/CURRENT.md` or index?
- How are conflicts represented when local and remote both changed since the last reconciliation?
- Which local records are never allowed to leave the workspace?
- What is the exact milestone/publish trigger for a Hub update?
- How should web ChatGPT, which may not have local filesystem access, recover enough context without making local state remote-canonical?
- What are the performance budgets for startup, retrieval, Hub probes, and Hub compaction?

## 7. Target operating model

### 7.1 Normal bound local session — target fast path

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

This is a **target**, not current released `0.1.0` behavior.

### 7.2 Remote/recovery path

Use the full Hub routing/recovery path when local memory is unavailable or cannot be trusted, including cases such as:

- new device;
- missing `.ai/`;
- unbound workspace;
- web-only agent without local filesystem access;
- schema mismatch;
- remote/shared state changed;
- explicit request for current Hub state;
- suspected conflict.

### 7.3 Write path

```text
substantive task
    ↓
local checkpoint as appropriate
    ↓
Does this change shared durable state?
   ├─ no  → remain local
   └─ yes → pre-write remote refresh
             ↓
          unchanged?
          ├─ yes → publish compact shared checkpoint
          └─ no  → reconcile local + remote + evidence, then publish
```

## 8. Performance targets

These are engineering targets, not release guarantees yet.

### P1 — normal startup memory payload

Target approximately **1k–2k tokens of memory payload** for a healthy bound local workspace under the normal fast path.

This budget applies to memory injected by the Skill, not to system prompts, tool wrappers, user input, or model-internal context.

### P2 — selective retrieval

Detailed history must be loaded only when relevant:

- read index first;
- load one or a few matching DEC/EXP/session/research records;
- fall back to broader evidence only if uncertainty remains.

### P3 — remote project compactness

Hub `PROJECT.md` should remain a compact shared checkpoint. When detailed history causes growth, move detail to session/research/decision records and compact the checkpoint without losing provenance.

A concrete soft token/byte threshold still needs to be defined and tested.

## 9. Work breakdown

Status values:

- `DONE` — completed and evidence exists;
- `ACTIVE` — current design work;
- `PLANNED` — accepted direction, not started;
- `BLOCKED` — requires a prior decision/test;
- `DEFERRED` — intentionally postponed;
- `REJECTED` — should not be implemented in the proposed form.

### Milestone M0 — establish evidence baseline

| ID | Task | Status |
| --- | --- | --- |
| M0.1 | Create Runtime Hub with deterministic project routing/isolation | DONE |
| M0.2 | Validate fresh-chat round trip on real bound workspaces | DONE |
| M0.3 | Add Hub session log schema and remote stale-SHA protocol | DONE |
| M0.4 | Create source-controlled `project-memory-hub-skill` reference repo | DONE |
| M0.5 | Establish Skill `0.1.0` / Hub schema 1 compatibility model | DONE |
| M0.6 | Perform strict read-only audit of existing local `project-memory` implementation | DONE |
| M0.7 | Identify duplicate-source-of-truth risk | DONE |

### Milestone M1 — unified architecture specification

| ID | Task | Status |
| --- | --- | --- |
| M1.1 | Define final authority model by information scope | ACTIVE |
| M1.2 | Define local-memory vs Hub-checkpoint boundary field-by-field | PLANNED |
| M1.3 | Define local multi-session pre-write refresh/conflict semantics | PLANNED |
| M1.4 | Define remote pre-write refresh/reconciliation semantics after unification | PLANNED |
| M1.5 | Define Hub adapter vs existing transport boundary | PLANNED |
| M1.6 | Define privacy / never-publish classes | PLANNED |
| M1.7 | Define normal fast path vs recovery path | PLANNED |
| M1.8 | Define memory payload and compaction budgets | PLANNED |
| M1.9 | Approve a Unified Architecture Proposal before code migration | BLOCKED on M1.1–M1.8 |

### Milestone M2 — canonical source and version model

| ID | Task | Status |
| --- | --- | --- |
| M2.1 | Decide canonical source repository name/location | PLANNED |
| M2.2 | Import reusable local v1.1 implementation without user/project data | BLOCKED on M1.9 |
| M2.3 | Preserve/port implicit Codex discovery metadata | BLOCKED on M2.2 |
| M2.4 | Port local executable tests and validation tools | BLOCKED on M2.2 |
| M2.5 | Define version mapping from local 1.1.0 and Hub Skill 0.1.0 to unified releases | PLANNED |
| M2.6 | Define installation/update procedure so installed/dev copies derive from releases | PLANNED |

### Milestone M3 — Hub adapter and lazy synchronization

| ID | Task | Status |
| --- | --- | --- |
| M3.1 | Define sync metadata schema (`hub_project_sha` or equivalent) | PLANNED |
| M3.2 | Define “last reconciled SHA” semantics precisely | PLANNED |
| M3.3 | Implement lightweight remote-change probe | BLOCKED on M2 |
| M3.4 | Implement unchanged-remote fast path | BLOCKED on M3.3 |
| M3.5 | Implement changed-remote fetch + reconcile path | BLOCKED on M3.1–M3.3 |
| M3.6 | Implement milestone/shared-checkpoint publication rules | BLOCKED on M1.2/M1.6 |
| M3.7 | Keep detailed local EXP/DEC/task records local by default | PLANNED |
| M3.8 | Define web/new-device hydration behavior | PLANNED |

### Milestone M4 — concurrency hardening

| ID | Task | Status |
| --- | --- | --- |
| M4.1 | Remote Hub blob-SHA stale-write detection | DONE at protocol level |
| M4.2 | Execute live two-session remote concurrency test | PLANNED |
| M4.3 | Design safe concurrent writes to local `.ai/CURRENT.md` / indexes | PLANNED |
| M4.4 | Add executable local multi-session conflict tests | BLOCKED on M4.3 |
| M4.5 | Add local-vs-remote divergence synthetic tests | PLANNED |
| M4.6 | Add three-session stress scenario | PLANNED |
| M4.7 | Verify no lost update under compatible concurrent changes | BLOCKED on M4.2/M4.4/M4.5 |

### Milestone M5 — migration and compatibility

| ID | Task | Status |
| --- | --- | --- |
| M5.1 | Define migration for existing local `.ai/` trees | BLOCKED on M1/M2 |
| M5.2 | Define migration for already bound Hub projects | BLOCKED on M1/M2 |
| M5.3 | Ensure migration does not copy project-private evidence into public source | PLANNED |
| M5.4 | Preserve pre-existing `AGENTS.md` instructions | DONE as existing invariant; retest required |
| M5.5 | Define rollback and pre-upgrade checkpoint procedure | PLANNED |
| M5.6 | Run migration on synthetic fixtures before any real project | PLANNED |

### Milestone M6 — validation and release

| ID | Task | Status |
| --- | --- | --- |
| M6.1 | Executable smoke tests for unified install | PLANNED |
| M6.2 | Executable retrieval/write/experiment/decision tests | PLANNED |
| M6.3 | Executable Hub adapter tests | PLANNED |
| M6.4 | Live remote concurrency test | PLANNED |
| M6.5 | Token/startup payload benchmark | PLANNED |
| M6.6 | Fresh-chat round trip after behavior-affecting release | PLANNED |
| M6.7 | Release only after migration/rollback documentation is complete | BLOCKED on M5/M6 |

## 10. Explicitly rejected or paused implementation directions

The following should **not** proceed unless this plan is intentionally revised:

- `REJECTED` — building another independent local `.ai` schema inside the Hub Skill;
- `REJECTED` — automatically copying every local `EXP-*` into Hub `PROJECT.md`;
- `REJECTED` — treating existing transport CURRENT snapshots as authoritative Hub state;
- `REJECTED` — blindly allowing Hub `PROJECT.md` to overwrite newer local active-work evidence;
- `REJECTED` — blindly allowing local active state to overwrite newer shared Hub state;
- `PAUSED` — releasing a nominal Hub Skill `0.2.0` before the unified architecture/source-of-truth design is approved;
- `PAUSED` — migrating current real `.ai/` trees into a new schema before synthetic migration tests exist.

## 11. Acceptance criteria for the unified architecture

The architecture is ready for implementation only when all of the following are explicit:

1. one canonical source repository for Skill code/policies/templates/tests;
2. one unambiguous authority owner for every information class;
3. no file pair can both silently claim authority over the same semantic state;
4. local and remote pre-write refresh rules are specified;
5. local/remote divergence has a deterministic reconciliation procedure;
6. detailed experiments and transient work are not automatically promoted to shared state;
7. privacy boundaries and never-publish records are defined;
8. normal local startup can avoid a full remote project read when safely unchanged;
9. recovery/new-device/web paths remain possible without local state;
10. executable tests cover routing, isolation, retrieval, checkpointing, migration, and concurrency;
11. a rollback path exists for upgrades;
12. token/memory payload is measured rather than assumed.

## 12. Release gate

Do **not** call the next implementation release complete merely because Markdown protocols are updated.

A behavior-affecting unified release requires:

- approved architecture;
- source migration/import completed;
- executable tests passing;
- synthetic migration passing;
- live remote-concurrency validation;
- representative local multi-session validation;
- fresh-chat round-trip after upgrade;
- documented rollback;
- version and compatibility metadata updated.

## 13. Maintenance rules for this document

Update this file when any of the following occurs:

- architecture direction changes;
- a planning decision becomes normative;
- a milestone/task changes status;
- a new audit materially changes understanding;
- a migration/release is completed;
- an assumption is invalidated;
- a new blocker is discovered.

Do not use this document as a transcript. Preserve only history and planning information that affects implementation, maintenance, or future decisions.

When a task becomes `DONE`, record enough evidence (commit, test result, released document, or concise verification note) that a future maintainer can determine why it is considered complete.
