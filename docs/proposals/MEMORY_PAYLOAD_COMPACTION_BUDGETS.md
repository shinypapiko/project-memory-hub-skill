---
document_role: architecture-proposal
status: review-required
normative: false
architecture_state: proposed-unreleased
runtime_load_policy: maintenance-only
milestone: M1.8
created: 2026-08-27
last_reviewed: 2026-08-27
review_state: initial-review-pending
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
  - docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md
  - docs/decisions/ADR-0004-hub-adapter-and-transport-boundary.md
  - docs/decisions/ADR-0005-privacy-never-publish-policy.md
  - docs/decisions/ADR-0006-normal-fast-changed-remote-recovery-paths.md
---

# Memory payload / compaction budgets — M1.8

## 0. Purpose and strict scope

This proposal defines measurable resource budgets for the future unified Project Memory architecture and is deliberately limited to six bounded surfaces:

1. startup;
2. probe;
3. checkpoint payload;
4. changed fetch;
5. compaction;
6. recovery.

For those surfaces it defines:

- token budgets;
- logical-memory budgets;
- logical I/O budgets;
- payload soft thresholds and hard limits;
- deterministic over-budget behavior;
- a compaction fidelity floor.

It does not reopen ADR-0001 through ADR-0006. It does not define or implement APIs, Git/GitHub behavior, locks, Windows/filesystem primitives, adapter code, publication triggers, privacy-detector code, migration, real-data operations, or UI.

No current released Skill, Runtime Hub behavior, local Project Memory runtime, adapter, transport, or real data is modified by this proposal.

## 1. Measurement model

Budgets are evaluated against logical architecture surfaces rather than implementation-specific allocator or HTTP details.

### 1.1 Token count

`model_visible_tokens` means tokens that will actually be injected into the active model context for the operation.

The exact runtime tokenizer is recorded with the measurement. A token budget is not estimated from characters when an actual model injection is about to occur.

If an exact tokenizer count is unavailable before final injection, byte limits still apply; the operation must obtain an exact token count before crossing a hard model-visible token limit.

### 1.2 Logical memory

`logical_memory_bytes` means the total canonical UTF-8 byte size of simultaneously retained Project Memory payload bodies, normalized semantic forms, and budget-relevant manifests required by the operation.

It explicitly excludes interpreter/runtime allocator overhead, executable code pages, unrelated model state, and transport buffers not semantically part of the operation.

This makes the architecture budget measurable across implementations without selecting a language or allocator.

### 1.3 Logical I/O

Logical I/O counts semantic objects and bytes, not HTTP requests, Git commands, syscalls, or retry mechanics.

The measurable dimensions are:

```text
logical_remote_metadata_reads
logical_remote_content_reads
logical_local_content_reads
remote_content_bytes
local_content_bytes
```

A later implementation may require multiple lower-level calls to satisfy one logical read; M1.8 does not choose those calls.

### 1.4 Soft threshold versus hard limit

- **soft threshold**: crossing it requires a deterministic budget-reduction action before continuing on the preferred path;
- **hard limit**: crossing it forbids claiming successful completion of that bounded operation unless an accepted alternative route produces a new in-budget representation.

Soft-threshold exceedance is not itself data loss permission.

Hard-limit exceedance is never resolved by silent truncation.

## 2. Protected semantic fidelity floor

Compaction or budget pressure must never silently discard or weaken semantic state required for authority, correctness, privacy, or recovery.

The following protected semantic classes have a **100% retention fidelity floor** whenever they are applicable to the bounded object being compacted or presented:

```text
AUTHORITY
  canonical project/workspace identity
  authority/ownership role needed to interpret the state

DECISION
  accepted/current decision outcome
  supersession/dependency links required to know which decision is current

ACTIVE_TASK
  current goal
  active task identity/status
  next required action
  blocking dependency that changes what may be done next

CONFLICT
  unresolved conflict identity
  conflicting alternatives/context required for safe resolution

PRIVACY
  classification
  explicit prohibition / never-publish state
  destination restriction
  current privacy non-allow reason where relevant

UNRESOLVED_STATE
  BASE_UNINITIALIZED / BASE_INVALID
  remote commit or confirmation unknown
  local finalization failed/unknown
  route-selection recovery condition
  stale/conflict/derivative-required or equivalent non-success state

RECONCILIATION_IDENTITY
  B/reconciled identity required to interpret checkpoint state
  base-pair / observed-vs-reconciled distinction where applicable

PROVENANCE_REQUIRED_FOR_MEANING
  source/reference identity needed to verify or correctly interpret a protected semantic item
```

This fidelity floor protects semantics, not verbatim prose. Eligible verbose explanation may be summarized only if the resulting representation still preserves every protected semantic atom and the provenance needed to interpret it.

A compaction result that cannot prove this floor is not an acceptable compacted representation.

## 3. Budget table

All values below are architecture defaults for M1.8. They are measurable targets and limits for the future implementation, not claims about current runtime performance.

| Surface | Model-visible tokens soft / hard | Logical memory soft / hard | Logical I/O / payload limits | Required over-budget behavior |
| --- | ---: | ---: | --- | --- |
| **Normal startup** | 1,500 / 2,000 | 128 KiB / 256 KiB | unchanged fast path: remote checkpoint content reads hard = 0; exact probe metadata payload soft 4 KiB / hard 16 KiB | compact/select local memory before injection; if still above hard, return startup budget non-success or bounded degraded/recovery presentation; never truncate protected semantics |
| **Probe** | 0 / 0 | 16 KiB / 32 KiB | one logical exact checkpoint-target observation per route-selection attempt; remote checkpoint body bytes hard = 0; probe metadata soft 4 KiB / hard 16 KiB | oversized/non-exact/unknown probe cannot authorize fast path; route according to ADR-0006 recovery semantics |
| **Shared checkpoint payload** | 2,000 / 4,000 when model-visible | 64 KiB / 128 KiB | canonical serialized checkpoint UTF-8 soft 16 KiB / hard 32 KiB | compact/minimize to a new candidate; if hard limit remains exceeded, checkpoint publication/use is blocked for that candidate; no silent field removal |
| **Changed fetch** | 4,000 / 8,000 | 256 KiB / 512 KiB | one exact current checkpoint body as `R` per reconciliation classification step; fetched checkpoint body hard = 32 KiB; no bulk history fetch by default | if exact checkpoint exceeds hard payload limit, do not partially reconcile it; enter typed over-budget/recovery outcome |
| **Compaction working set** | output governed by destination surface | 512 KiB / 1 MiB | compaction input may use indexes/manifests plus exact referenced records needed for protected semantics; no unbounded history sweep | apply deterministic compaction tiers; remeasure; if hard destination limit still exceeded, return non-success/derivative-required rather than truncate |
| **Recovery presentation** | 6,000 / 12,000 | 1 MiB / 4 MiB | index/manifest first; concurrently retained remote/local content objects soft 8 / hard 16; each shared-checkpoint body still obeys 32 KiB hard limit | stage recovery, summarize eligible resolved history, and preserve unresolved/protected semantics; if still over hard, remain in recovery and request/select narrower evidence rather than claim synchronized/current |

## 4. Normal startup budget

ADR-0006 permits normal fast path only after an exact trusted unchanged probe bound to current route-selection context.

M1.8 adds the resource condition:

```text
normal_startup_success
  requires
    model_visible_tokens <= 2,000
    logical_memory_bytes <= 256 KiB
    remote_checkpoint_content_reads == 0
```

The preferred operating target is at or below 1,500 model-visible tokens and 128 KiB logical memory.

Crossing the soft threshold triggers deterministic selection/compaction before model injection.

The normal startup payload should prioritize, in order:

```text
1. authority/binding facts required for the current project
2. current project goal / active task / next action
3. unresolved blockers/conflicts/privacy/recovery state
4. only the minimal stable project background needed now
5. links/index references to non-loaded history
```

Resolved historical detail is not entitled to displace protected active semantics merely because it is newer or verbose.

If the protected semantics alone exceed the hard startup limit, the operation must not silently truncate them and claim normal startup success. It returns a typed over-budget/degraded/recovery presentation result and may require a narrower operator/model-visible view while keeping the protected semantic state intact outside that view.

## 5. Probe budget

A probe is a lightweight observation surface, not a checkpoint-content fetch.

For one route-selection attempt:

```text
model_visible_tokens = 0
remote_checkpoint_content_reads = 0
probe_metadata_bytes <= 16 KiB
logical_memory_bytes <= 32 KiB
```

An implementation may internally perform lower-level work, but if the semantic result requires loading the full checkpoint body, it is no longer counted as the M1.8 lightweight probe surface.

Probe budget failure, unknown result, oversized result, or inability to establish exact target identity cannot be converted to `UNCHANGED`. ADR-0006 recovery routing applies.

A route-selection restart creates a new attempt. Old probe evidence is not kept alive merely to avoid re-probing.

## 6. Shared checkpoint payload budget

The future shared checkpoint is a minimized shared state, not a mirror of local `.ai/`.

Its canonical serialized UTF-8 payload has:

```text
soft threshold = 16 KiB
hard limit     = 32 KiB
```

When model-visible, it additionally has:

```text
soft threshold = 2,000 tokens
hard limit     = 4,000 tokens
```

If the candidate crosses a soft threshold, it must be minimized/compacted into a **new candidate identity**, then revalidated for schema, provenance, privacy, and any other context whose identity changed.

If the candidate remains above either applicable hard limit:

```text
CHECKPOINT_PAYLOAD_OVER_LIMIT
```

or equivalent typed non-success is required.

The system must not silently drop fields, attachments, provenance, P3/UNCLASSIFIED evidence, conflicts, decisions, active-task state, or unresolved markers to force the same candidate under limit.

## 7. Changed-fetch budget

The changed-remote route defined by ADR-0006 still requires exact fetch plus ADR-0003 reconciliation.

M1.8 requires the exact fetched shared checkpoint `R` to obey the 32 KiB hard shared-checkpoint body limit. An oversized exact checkpoint is not partially treated as `R`.

For one reconciliation classification step:

```text
exact current checkpoint body reads = 1
logical_memory soft = 256 KiB
logical_memory hard = 512 KiB
model-visible soft = 4,000 tokens
model-visible hard = 8,000 tokens
```

The logical-memory allowance is intentionally larger than one checkpoint because B/R/L/C semantic forms and validation/provenance manifests may coexist.

If a remote revision changes after fetch, ADR-0003 controls stale/recompute behavior. M1.8 does not add a retry policy.

If reconciliation requires evidence beyond the bounded current checkpoint, retrieval is index/reference driven rather than an implicit bulk-history fetch.

## 8. Deterministic compaction tiers

Compaction is applied only to semantics eligible for compaction. Protected semantic classes in Section 2 retain the 100% fidelity floor.

When a soft threshold is crossed, the deterministic order is:

### Tier C0 — structural deduplication

Remove duplicate rendering, repeated headings, redundant prose, repeated already-bound metadata, and reconstructible formatting.

No semantic item is removed.

### Tier C1 — reference substitution

Replace eligible already-resolved detailed history with stable references/index entries where the referenced source remains available and the current operation does not require its full content.

Protected current decision/task/conflict/privacy/unresolved semantics remain materialized.

### Tier C2 — bounded semantic summary

Summarize eligible resolved/background material while preserving:

- exact current outcome;
- scope/applicability;
- supersession status where relevant;
- required provenance link;
- uncertainty that changes interpretation.

A summary is a new representation and must not masquerade as verbatim evidence.

### Tier C3 — split presentation, not silent truncation

If the destination presentation still exceeds its hard model-visible limit while the underlying semantic object is valid, split model-visible presentation into:

```text
protected current core
+ explicit continuation/reference set
```

The first part must clearly state that additional evidence exists and is not currently loaded.

C3 may reduce what is injected into the model, but it does not mutate or silently delete the underlying source/candidate.

### Tier C4 — hard stop

If the operation cannot meet the destination hard limit without violating the fidelity floor or required schema/provenance/privacy semantics, return a typed over-budget/non-success result.

The system remains in the appropriate normal/recovery/non-publication state. It does not claim success.

## 9. Compaction fidelity and verification

Every compacted representation must be evaluated against a `fidelity_manifest` conceptually containing the protected semantic atoms expected from the source context.

The exact manifest encoding is deferred, but the semantic check is mandatory:

```text
protected_semantics_retained == 100%
required_provenance_retained == 100%
explicit_prohibitions_retained == 100%
unresolved_states_retained == 100%
```

Compaction may reduce wording, duplication, and eligible resolved-history detail. It may not:

- convert conflict into resolution;
- convert unknown into success;
- convert P3/UNCLASSIFIED into a weaker privacy state;
- remove an active blocker;
- remove the current accepted decision while leaving an obsolete one;
- remove authority/binding context needed to interpret the memory;
- remove required provenance solely to fit a budget;
- convert `observed` into `reconciled`;
- convert recovery/degraded state into normal synchronized state.

A fidelity check failure is a compaction failure, not a warning-only success.

## 10. Recovery budget

Recovery is allowed a larger bounded envelope than normal startup because it may need exact diagnostic evidence.

Preferred presentation:

```text
model_visible_tokens <= 6,000
logical_memory_bytes <= 1 MiB
concurrently retained content objects <= 8
```

Hard limits:

```text
model_visible_tokens <= 12,000
logical_memory_bytes <= 4 MiB
concurrently retained content objects <= 16
shared checkpoint body <= 32 KiB each
```

Recovery retrieval is staged:

```text
1. current binding/base/recovery-state manifest
2. exact current remote observation/checkpoint if authorized and required
3. indexes/references
4. only the specific additional records needed to resolve the current uncertainty
```

Recovery does not obtain an unlimited budget merely because the normal path failed.

If hard limits are reached before the uncertainty is resolved, recovery remains unresolved and returns a typed over-budget evidence requirement. It may request/select a narrower next evidence set. It must not claim remote-current, synchronized, reconciled, conflict-resolved, or privacy-approved status merely because some evidence was omitted for budget reasons.

## 11. Deterministic over-budget outcomes

The future unified interface must distinguish budget pressure from ordinary success.

Conceptual outcomes include:

```text
BUDGET_WITHIN_LIMIT
BUDGET_SOFT_EXCEEDED_COMPACTION_REQUIRED
BUDGET_HARD_EXCEEDED
BUDGET_PROTECTED_SEMANTICS_TOO_LARGE
BUDGET_FIDELITY_FAILURE
BUDGET_CHECKPOINT_PAYLOAD_OVER_LIMIT
BUDGET_CHANGED_FETCH_OVER_LIMIT
BUDGET_RECOVERY_EVIDENCE_OVER_LIMIT
BUDGET_PRESENTATION_SPLIT_REQUIRED
```

Exact enum names are deferred.

Rules:

1. soft exceedance triggers the relevant deterministic reduction tier;
2. after each representation-changing reduction, the result is remeasured;
3. any new candidate/derivative identity created by minimization/compaction must pass all context checks required by the inherited ADRs;
4. hard exceedance after allowed reduction returns a typed non-success;
5. no budget outcome advances reconciliation markers, authorizes publication, repairs base state, or resolves conflict by itself;
6. no over-budget condition authorizes transport fallback;
7. no hard-limit handling may silently truncate and still report the bounded operation as successful.

## 12. Relationship to accepted ADRs

M1.8 only constrains resource envelopes.

- ADR-0001 authority still decides what information is authoritative and what may be projected.
- ADR-0002 still governs safe local writes; budget pressure does not authorize unsafe in-place mutation.
- ADR-0003 still governs B/R/L/C, observed versus reconciled state, CAS, confirmation, and finalization.
- ADR-0004 still forbids transport from becoming an adapter/budget fallback.
- ADR-0005 still governs privacy; compaction cannot weaken classification or required provenance, and any new outbound candidate requires current privacy evaluation.
- ADR-0006 still governs fast/changed/recovery route selection; budget failure cannot be reinterpreted as an unchanged probe or synchronization success.

## 13. Required invariants

M1.8 is invalid if any of the following can occur silently:

1. an over-budget model-visible payload is truncated and reported as successful;
2. a hard checkpoint payload limit is bypassed by dropping arbitrary fields from the same candidate;
3. protected authority/decision/active-task/conflict/privacy/unresolved semantics are lost during compaction;
4. required provenance is removed solely to meet a budget;
5. an obsolete decision survives while the current accepted decision is compacted away;
6. an unresolved state is compacted into a success claim;
7. P3/UNCLASSIFIED or an explicit prohibition is weakened by summarization;
8. observed/recovery state is summarized as reconciled/current;
9. a probe loads checkpoint body content yet is still counted as the lightweight unchanged probe surface;
10. probe over-budget/unknown is treated as unchanged;
11. changed fetch partially loads an oversized checkpoint and proceeds as if exact `R` was obtained;
12. recovery receives an implicit unlimited history sweep;
13. compaction changes candidate identity but stale privacy/provenance/context verdicts are reused;
14. hard-limit failure advances `B/reconciled_sha`, authorizes CAS/publication, repairs binding/base state, or claims synchronization success;
15. transport is used as a budget-overflow fallback;
16. implementation-specific RSS/HTTP-call variability is confused with the logical budgets defined here.

## 14. Decisions requested for M1.8 review — MB1–MB14

Initial review should output `APPROVE` or `REVISE` for each decision.

- **MB1 — Measurement model.** Token, logical-memory, and logical-I/O units are measurable without selecting implementation APIs or allocators.
- **MB2 — Soft/hard semantics.** Soft exceedance triggers deterministic reduction; hard exceedance is non-success unless a new valid in-budget representation is produced.
- **MB3 — Protected fidelity floor.** Authority, current decisions, active tasks, conflicts, privacy, unresolved state, reconciliation identity, and meaning-critical provenance retain 100% semantic fidelity.
- **MB4 — Normal startup budget.** Preferred 1,500-token / 128-KiB envelope and hard 2,000-token / 256-KiB envelope preserve the roadmap's approximately 1k–2k-token target without silent truncation.
- **MB5 — Probe budget.** Probe is zero model-visible tokens, zero checkpoint-body reads, bounded metadata, and cannot authorize fast path when unknown/oversized/non-exact.
- **MB6 — Checkpoint payload budget.** Shared checkpoint canonical payload soft 16 KiB / hard 32 KiB, with model-visible soft 2,000 / hard 4,000 tokens when applicable.
- **MB7 — Changed-fetch budget.** Exact `R` obeys checkpoint hard limit; one current exact checkpoint body is loaded per reconciliation classification step, with bounded B/R/L/C working set.
- **MB8 — Compaction tiers.** C0 deduplication, C1 reference substitution, C2 semantic summary, C3 split presentation, C4 hard stop are deterministic and do not silently mutate source truth.
- **MB9 — Fidelity verification.** Compaction failure to preserve protected semantics/provenance is a typed non-success, not warning-only success.
- **MB10 — Recovery budget.** Recovery has a larger but finite staged evidence envelope and remains unresolved rather than claiming success when hard limits prevent sufficient evidence.
- **MB11 — Deterministic over-budget outcomes.** Budget states remain typed and cannot advance markers, resolve conflicts, authorize publication, or repair lineage by themselves.
- **MB12 — Candidate/context invalidation after compaction.** A representation-changing minimization/compaction that creates a new candidate requires fresh inherited context checks, including privacy where outbound publication is involved.
- **MB13 — No silent truncation or fallback.** No hard-limit path silently truncates protected/required material, treats unknown as success, or falls back to transport.
- **MB14 — Protocol-only scope.** M1.8 fixes measurable architecture budgets only; implementation/API/locks/adapter/publication trigger/detector/migration/real data/UI remain deferred.

## 15. Deferred implementation and next milestone

M1.8 deliberately does not choose:

- tokenizer library or model SDK;
- memory allocator or RSS instrumentation;
- probe/fetch API;
- Git/GitHub API;
- sync-metadata physical encoding;
- lock/Windows/filesystem primitives;
- retry/backoff values;
- adapter implementation;
- publication trigger;
- privacy detector/classifier implementation;
- migration;
- real-data test corpus;
- UI/operator workflow.

Executable/static/live validation of these budgets is also deferred. Later validation may revise engineering numbers through an explicit superseding decision if measurements show the defaults are unrealistic; silent runtime drift of the limits is not allowed.

After M1.8 governance closure, the next architecture step is M1.9 unified-architecture consistency/approval. M1.9 should not reopen closed milestone semantics merely to add implementation detail.

## 16. Maturity boundary

This proposal is architecture only:

```text
runtime_effective: false
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
```

M1.8 approval would not mean runtime budgeting, compaction code, probe logic, adapter behavior, migration, or validation has been implemented.
