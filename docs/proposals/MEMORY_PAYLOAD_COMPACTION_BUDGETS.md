---
document_role: architecture-proposal
status: approved-proposal
normative: false
architecture_state: accepted-unreleased
runtime_load_policy: maintenance-only
milestone: M1.8
created: 2026-08-27
last_reviewed: 2026-08-27
review_state: final-approved-promoted-to-adr
initial_review_baseline: 8bd7ac87cc903aae9fafcdce03ded1fbccc719a7
initial_review_proposal_git_blob_lf_sha256: A1A409AC5D2D2D15367470DB401D1115F5E118DE42621463BDAE918E97BAAEA1
final_review_baseline: 009b379f3f978223c1e28ad4906872ed9f4a6c48
final_review_proposal_git_blob_lf_sha256: B8713A83AC6A980A827014517DAD7CD4A6858D2F190F0C1BC3B57B3E9CBFCCE1
final_approval: docs/reviews/2026-08-27-memory-payload-compaction-budgets-final-approval.md
approved_by: docs/decisions/ADR-0007-memory-payload-compaction-budgets.md
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
remote_metadata_bytes
remote_content_bytes
local_content_bytes
```

A logical read is charged when the bounded operation semantically obtains an object or metadata unit from its source. Re-reading or re-fetching the same logical object/revision is charged again. Pagination or chunking cannot hide bytes: all bytes consumed to obtain the logical evidence are charged cumulatively.

A later implementation may require multiple lower-level calls to satisfy one logical read; M1.8 does not choose those calls.

### 1.3.1 Cumulative `budget_accounting_scope`

Every bounded top-level startup/sync/recovery operation has one conceptual `budget_accounting_scope`. Its counters are monotonic for the lifetime of that operation.

At minimum it binds:

```text
budget_accounting_scope {
    bounded_surface_identity
    operation_identity
    cumulative_logical_remote_metadata_reads
    cumulative_logical_remote_content_reads
    cumulative_logical_local_content_reads
    cumulative_remote_metadata_bytes
    cumulative_remote_content_bytes
    cumulative_local_content_bytes
}
```

The exact storage/encoding is deferred. The architectural rules are:

1. pagination, staged retrieval, compaction tiers, route re-selection/restart, exact re-fetch, stale-CAS recomputation, conflict-resolution recomputation, or recovery staging **do not reset cumulative counters** within the same top-level operation;
2. a nested surface is charged against its own applicable limit and against the enclosing top-level scope where that evidence is consumed;
3. a route restart under ADR-0006 remains part of the same top-level scope until the operation reaches a terminal typed outcome or the caller explicitly begins a later independent operation;
4. a re-fetch of the same or newer remote object is a new logical read and its bytes are added again;
5. a compaction tier that loads more records adds those reads/bytes to the same scope rather than opening a fresh budget;
6. if cumulative accounting becomes unavailable or indeterminate, the bounded operation cannot claim in-budget success;
7. crossing a cumulative soft threshold triggers the deterministic reduction/recovery behavior for that surface; crossing a cumulative hard limit returns a typed non-success/recovery outcome and cannot be converted to success by resetting the scope.

This is an accounting boundary only; it does not select API, retry, lock, persistence, or implementation mechanisms.

### 1.4 Soft threshold versus hard limit

- **soft threshold**: crossing it requires a deterministic budget-reduction action before continuing on the preferred path;
- **hard limit**: crossing it forbids claiming successful completion of that bounded operation unless an accepted alternative route produces a new in-budget representation without resetting the same operation's cumulative scope.

Soft-threshold exceedance is not itself data loss permission.

Hard-limit exceedance is never resolved by silent truncation or counter reset.

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

All I/O values are **cumulative soft / hard limits inside one `budget_accounting_scope`**. Nested staging, pagination, restart, re-fetch, recompute, and compaction do not reset them.

| Surface | Model-visible tokens soft / hard | Logical memory soft / hard | Cumulative logical I/O / payload soft / hard | Required over-budget behavior |
| --- | ---: | ---: | --- | --- |
| **Normal startup** | 1,500 / 2,000 | 128 KiB / 256 KiB | local content reads 12 / 24; local bytes 256 / 512 KiB; remote metadata reads 1 / 2; remote metadata bytes 4 / 16 KiB; remote checkpoint content reads/bytes 0 / 0 | compact/select local memory before injection; if any hard limit remains exceeded, return startup budget non-success or bounded degraded/recovery presentation; never truncate protected semantics |
| **Probe** | 0 / 0 | 16 KiB / 32 KiB | remote metadata reads 1 / 2; remote metadata bytes 4 / 16 KiB; local context reads 2 / 4; local bytes 16 / 32 KiB; remote checkpoint content reads/bytes 0 / 0 | oversized/non-exact/unknown probe or cumulative hard exceedance cannot authorize fast path; route according to ADR-0006 recovery semantics |
| **Shared checkpoint payload** | 2,000 / 4,000 when model-visible | 64 KiB / 128 KiB | local content reads 24 / 48; local bytes 512 KiB / 1 MiB; canonical serialized checkpoint UTF-8 16 / 32 KiB; no independent remote-content entitlement beyond any enclosing changed-fetch scope | compact/minimize under Section 8; if hard limit remains exceeded, checkpoint use/publication is blocked for that candidate; no silent field removal |
| **Changed fetch** | 4,000 / 8,000 | 256 KiB / 512 KiB | remote checkpoint content reads 2 / 4; remote content bytes 64 / 128 KiB; remote metadata reads 4 / 8; remote metadata bytes 32 / 64 KiB; local content reads 24 / 48; local bytes 512 KiB / 1 MiB; each exact checkpoint body hard = 32 KiB | exact re-fetch/recompute remains in the same scope; if cumulative or per-body hard limit is exceeded, do not partially reconcile; enter typed over-budget/recovery outcome |
| **Compaction working set** | output governed by destination surface | 512 KiB / 1 MiB | local content reads 32 / 64; local bytes 1 / 2 MiB; remote content reads 4 / 8; remote bytes 128 / 256 KiB; remote metadata reads 8 / 16; remote metadata bytes 64 / 128 KiB | apply deterministic compaction tiers without resetting counters; if hard destination or cumulative I/O limit remains exceeded, return non-success/derivative-required rather than truncate |
| **Recovery presentation** | 6,000 / 12,000 | 1 MiB / 4 MiB | local content reads 32 / 64; local bytes 2 / 8 MiB; remote content reads 8 / 16; remote bytes 256 / 512 KiB; remote metadata reads 16 / 32; remote metadata bytes 64 / 128 KiB; concurrently retained content objects 8 / 16; each shared-checkpoint body hard = 32 KiB | stage recovery, but cumulative counters continue across stages/pages/restarts; on any hard exceedance remain in recovery with typed over-budget evidence requirement rather than claim synchronized/current |

## 4. Normal startup budget

ADR-0006 permits normal fast path only after an exact trusted unchanged probe bound to current route-selection context.

M1.8 adds the resource condition:

```text
normal_startup_success
  requires
    model_visible_tokens <= 2,000
    logical_memory_bytes <= 256 KiB
    cumulative_logical_local_content_reads <= 24
    cumulative_local_content_bytes <= 512 KiB
    cumulative_logical_remote_metadata_reads <= 2
    cumulative_remote_metadata_bytes <= 16 KiB
    cumulative_remote_checkpoint_content_reads == 0
    cumulative_remote_checkpoint_content_bytes == 0
```

The preferred operating target is at or below 1,500 model-visible tokens, 128 KiB logical memory, 12 local content reads / 256 KiB local bytes, and one exact probe metadata observation within 4 KiB.

Crossing a soft threshold triggers deterministic selection/compaction before model injection. Any selection/compaction reads remain charged to the same startup accounting scope.

The normal startup payload should prioritize, in order:

```text
1. authority/binding facts required for the current project
2. current project goal / active task / next action
3. unresolved blockers/conflicts/privacy/recovery state
4. only the minimal stable project background needed now
5. links/index references to non-loaded history
```

Resolved historical detail is not entitled to displace protected active semantics merely because it is newer or verbose.

If the protected semantics alone exceed the hard startup limit, or cumulative local/probe I/O crosses a startup hard limit, the operation must not silently truncate or reset accounting and claim normal startup success. It returns a typed over-budget/degraded/recovery presentation result and may require a narrower operator/model-visible view while keeping the protected semantic state intact outside that view.

## 5. Probe budget

A probe is a lightweight observation surface, not a checkpoint-content fetch.

Within the enclosing top-level route-selection operation, the probe charges cumulative counters:

```text
model_visible_tokens = 0
remote_checkpoint_content_reads = 0
remote_checkpoint_content_bytes = 0
remote_metadata_reads soft/hard = 1 / 2
remote_metadata_bytes soft/hard = 4 / 16 KiB
local_context_reads soft/hard = 2 / 4
local_content_bytes soft/hard = 16 / 32 KiB
logical_memory_bytes soft/hard = 16 / 32 KiB
```

A route restart may require a new exact probe, but that new metadata observation is charged again to the same top-level `budget_accounting_scope`; restart does not reset prior probe cost.

An implementation may internally perform lower-level work, but if the semantic result requires loading the full checkpoint body, it is no longer counted as the M1.8 lightweight probe surface.

Probe budget failure, unknown result, oversized result, accounting uncertainty, or inability to establish exact target identity cannot be converted to `UNCHANGED`. ADR-0006 recovery routing applies.

Old probe evidence is not kept alive merely to avoid re-probing after route-context drift.

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

Formation/minimization of one payload remains cumulatively bounded to 24 / 48 local content reads and 512 KiB / 1 MiB local bytes. Any remote evidence consumed by an enclosing changed-fetch/reconciliation operation remains charged to that enclosing scope as well; checkpoint formation does not create a fresh remote-I/O allowance.

If the candidate crosses a soft threshold, Section 8 determines whether the reduction is a semantics-preserving presentation/serialization rewrite or a semantic minimization/derivative.

A semantic minimization/derivative is **not** permitted to compact old `C`, substitute a new identity, and continue under old reconciliation or privacy conclusions. It must follow the ADR-0003 re-formation sequence in Section 8.

If the candidate remains above either applicable hard limit:

```text
CHECKPOINT_PAYLOAD_OVER_LIMIT
```

or equivalent typed non-success is required.

The system must not silently drop fields, attachments, provenance, P3/UNCLASSIFIED evidence, conflicts, decisions, active-task state, or unresolved markers to force the same candidate under limit.

## 7. Changed-fetch budget

The changed-remote route defined by ADR-0006 still requires exact fetch plus ADR-0003 reconciliation.

M1.8 requires each exact fetched shared checkpoint `R` to obey the 32 KiB hard shared-checkpoint body limit. An oversized exact checkpoint is not partially treated as `R`.

One top-level changed-fetch/reconciliation `budget_accounting_scope` persists across exact re-fetch, stale recomputation, and explicit-resolution recomputation. Its cumulative limits are:

```text
remote checkpoint content reads soft/hard = 2 / 4
remote content bytes soft/hard = 64 / 128 KiB
remote metadata reads soft/hard = 4 / 8
remote metadata bytes soft/hard = 32 / 64 KiB
local content reads soft/hard = 24 / 48
local content bytes soft/hard = 512 KiB / 1 MiB
logical_memory soft/hard = 256 / 512 KiB
model-visible soft/hard = 4,000 / 8,000 tokens
```

Each reconciliation classification still uses one exact current checkpoint body as `R`, but that statement is **not a budget reset boundary**. If ADR-0003 requires a newer `R`, the additional exact fetch is charged cumulatively to the same operation.

The logical-memory allowance is intentionally larger than one checkpoint because B/R/L/C semantic forms and validation/provenance manifests may coexist.

If a remote revision changes after fetch, ADR-0003 controls stale/recompute behavior. M1.8 does not add a retry policy; the cumulative M1.8 I/O envelope still applies to however many ADR-0003-authorized re-observations/recomputations occur.

If reconciliation requires evidence beyond the bounded current checkpoint, retrieval is index/reference driven and charged to the same scope rather than becoming an implicit bulk-history fetch.

Any cumulative hard exceedance produces a typed changed-fetch over-budget/recovery outcome. It cannot partially load an oversized or incomplete `R`, reset the counter, and proceed as successful reconciliation.

## 8. Deterministic compaction tiers and reconciliation boundary

Compaction is applied only to semantics eligible for compaction. Protected semantic classes in Section 2 retain the 100% fidelity floor. All record loads performed by compaction tiers remain charged to the same `budget_accounting_scope`.

Before applying a compacted result to checkpoint/reconciliation state, the architecture distinguishes two classes.

### 8.1 Class A — semantics-preserving presentation / serialization rewrite

Class A may deduplicate rendering, change presentation order, substitute equivalent serialization, or summarize only where the semantic object consumed by ADR-0003 remains exactly the same semantic state and all authority/provenance/privacy/conflict/unresolved atoms remain equivalent.

Class A does not form a new shared-intent `L` or a new semantic candidate `C` merely because bytes or presentation changed.

If any identity/context used by an inherited ADR would change, or semantic equivalence cannot be proven, the operation is **not** Class A and must be treated as Class B.

### 8.2 Class B — semantic minimization / derivative

Any reduction that changes the shared semantic state, field set, provenance-bearing content, meaning-bearing references, or candidate semantics is a new minimization/derivative.

For Class B the required sequence is:

```text
semantic minimization / derivative requested
  -> old C invalid
  -> old reconciliation context invalid
  -> old conflict-resolution context invalid
  -> old CAS/write context invalid
  -> old privacy verdict/context invalid
  -> form new L / derivative from current authoritative shared intent
  -> capture current ADR-0003 B and exact R context
  -> rerun ADR-0003 B/R/L reconciliation
     or rerun explicit resolution against a current exact conflict context
  -> produce new C
  -> rerun schema + cross-field + required provenance validation
  -> if remote write is required:
       obtain fresh ADR-0005 privacy ALLOW for exact new C/bundle/destination
  -> only then enter ADR-0003 write-capable CAS
```

The operation must not obtain a compacted `C2` by replacing the identity/SHA/token/context value of old `C` and reusing old reconciliation, resolution, CAS, confirmation, provenance, or privacy conclusions.

If current B/R/L or conflict context has drifted during this process, the inherited ADR-0003 stale/recompute rules apply. M1.8 does not define a new reconciliation algorithm.

### 8.3 Compaction tiers

When a soft threshold is crossed, the deterministic order is:

#### Tier C0 — structural deduplication

Remove duplicate rendering, repeated headings, redundant prose, repeated already-bound metadata, and reconstructible formatting.

No semantic item is removed. Where this is provably Class A, semantic reconciliation state is unchanged.

#### Tier C1 — reference substitution

Replace eligible already-resolved detailed history with stable references/index entries where the referenced source remains available and the current operation does not require its full content.

Protected current decision/task/conflict/privacy/unresolved semantics remain materialized.

If reference substitution changes shared candidate semantics or required provenance, it is Class B and must use the re-formation sequence above.

#### Tier C2 — bounded semantic summary

Summarize eligible resolved/background material while preserving:

- exact current outcome;
- scope/applicability;
- supersession status where relevant;
- required provenance link;
- uncertainty that changes interpretation.

A summary is a new representation and must not masquerade as verbatim evidence. If the summary changes shared candidate semantics, it is Class B.

#### Tier C3 — split presentation, not silent truncation

If the destination presentation still exceeds its hard model-visible limit while the underlying semantic object is valid, split model-visible presentation into:

```text
protected current core
+ explicit continuation/reference set
```

The first part must clearly state that additional evidence exists and is not currently loaded.

C3 may reduce what is injected into the model, but it does not mutate or silently delete the underlying source/candidate. Therefore a pure presentation split is Class A only when semantic equivalence is preserved.

#### Tier C4 — hard stop

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

Fidelity success does not by itself establish that a Class B derivative is reconciled. Class B must still complete the ADR-0003 re-formation and any fresh ADR-0005 privacy gate required by Section 8.

## 10. Recovery budget

Recovery is allowed a larger bounded envelope than normal startup because it may need exact diagnostic evidence, but streaming or staged retrieval cannot make that envelope unbounded.

Preferred presentation / cumulative I/O:

```text
model_visible_tokens <= 6,000
logical_memory_bytes <= 1 MiB
concurrently retained content objects <= 8
local content reads <= 32
local content bytes <= 2 MiB
remote content reads <= 8
remote content bytes <= 256 KiB
remote metadata reads <= 16
remote metadata bytes <= 64 KiB
```

Hard limits:

```text
model_visible_tokens <= 12,000
logical_memory_bytes <= 4 MiB
concurrently retained content objects <= 16
local content reads <= 64
local content bytes <= 8 MiB
remote content reads <= 16
remote content bytes <= 512 KiB
remote metadata reads <= 32
remote metadata bytes <= 128 KiB
shared checkpoint body <= 32 KiB each
```

Recovery retrieval is staged:

```text
1. current binding/base/recovery-state manifest
2. exact current remote observation/checkpoint if authorized and required
3. indexes/references
4. only the specific additional records needed to resolve the current uncertainty
```

All pages, stages, compaction passes, route re-selections, and re-fetches remain charged to the same recovery accounting scope. Concurrently retained-object limits and cumulative-read/byte limits are independent; streaming one object at a time cannot evade the cumulative hard limits.

Recovery does not obtain an unlimited budget merely because the normal path failed.

If hard limits are reached before the uncertainty is resolved, recovery remains unresolved and returns a typed over-budget evidence requirement. It may request/select a narrower next evidence set in a later explicitly initiated operation. It must not reset the current scope, omit necessary evidence, and claim remote-current, synchronized, reconciled, conflict-resolved, or privacy-approved status.

## 11. Deterministic over-budget outcomes

The future unified interface must distinguish budget pressure from ordinary success.

Conceptual outcomes include:

```text
BUDGET_WITHIN_LIMIT
BUDGET_SOFT_EXCEEDED_COMPACTION_REQUIRED
BUDGET_HARD_EXCEEDED
BUDGET_ACCOUNTING_UNKNOWN
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
2. after each representation-changing reduction, the result is remeasured while cumulative I/O counters remain monotonic;
3. a provably semantics-preserving Class A rewrite may retain the same semantic reconciliation state only if all inherited context identities remain valid;
4. a Class B semantic minimization/derivative invalidates old `C`, reconciliation/resolution/CAS/privacy contexts and must rerun the Section 8 ADR-0003 re-formation sequence;
5. hard exceedance after allowed reduction returns a typed non-success;
6. unknown/indeterminate cumulative accounting is non-success for the bounded operation;
7. no budget outcome advances reconciliation markers, authorizes publication, repairs base state, or resolves conflict by itself;
8. no over-budget condition authorizes transport fallback;
9. no hard-limit handling may silently truncate, reset counters, substitute candidate identity, or reuse stale context and still report the bounded operation as successful.

## 12. Relationship to accepted ADRs

M1.8 only constrains resource envelopes.

- ADR-0001 authority still decides what information is authoritative and what may be projected.
- ADR-0002 still governs safe local writes; budget pressure does not authorize unsafe in-place mutation.
- ADR-0003 still governs B/R/L/C, observed versus reconciled state, explicit conflict resolution, CAS, confirmation, finalization, and stale/recompute behavior. Class B compaction must re-enter ADR-0003 formation rather than reuse old candidate conclusions.
- ADR-0004 still forbids transport from becoming an adapter/budget fallback.
- ADR-0005 still governs privacy; compaction cannot weaken classification or required provenance, and any new outbound Class B candidate requires fresh current privacy evaluation.
- ADR-0006 still governs fast/changed/recovery route selection; budget failure cannot be reinterpreted as an unchanged probe or synchronization success, and route restart does not reset cumulative accounting inside the same top-level operation.

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
12. pagination, staging, compaction tiers, route restart, re-fetch, recomputation, or streaming resets cumulative logical-I/O counters;
13. recovery stays under concurrent-resident-object limits by streaming an unbounded number of objects/bytes;
14. Class B compaction changes candidate semantics but stale reconciliation/conflict-resolution/CAS/privacy/provenance conclusions are reused;
15. a new candidate identity/SHA/token/context value is substituted into an old ADR-0003 conclusion instead of forming new L and rerunning current B/R/L reconciliation or explicit resolution;
16. fidelity success is treated as reconciliation success for a semantic derivative;
17. hard-limit failure advances `B/reconciled_sha`, authorizes CAS/publication, repairs binding/base state, or claims synchronization success;
18. transport is used as a budget-overflow fallback;
19. implementation-specific RSS/HTTP-call variability is confused with the logical budgets defined here;
20. cumulative accounting becomes unknown but the operation still claims successful in-budget completion.

## 14. Decisions requested for M1.8 review — MB1–MB14

Subsequent review should output `APPROVE` or `REVISE` for each decision.

- **MB1 — Measurement model.** Token, logical-memory, and logical-I/O units are measurable without selecting implementation APIs or allocators; one cumulative `budget_accounting_scope` prevents pagination/staging/compaction/restart/re-fetch/recompute from resetting I/O accounting.
- **MB2 — Soft/hard semantics.** Soft exceedance triggers deterministic reduction; hard exceedance or accounting uncertainty is non-success unless a new valid in-budget representation is produced without resetting the same operation's cumulative scope.
- **MB3 — Protected fidelity floor.** Authority, current decisions, active tasks, conflicts, privacy, unresolved state, reconciliation identity, and meaning-critical provenance retain 100% semantic fidelity.
- **MB4 — Normal startup budget.** Preferred 1,500-token / 128-KiB envelope and hard 2,000-token / 256-KiB envelope are paired with cumulative local-read/byte and probe-metadata limits; no nested selection/compaction resets the startup scope.
- **MB5 — Probe budget.** Probe is zero model-visible tokens, zero checkpoint-body reads/bytes, bounded cumulative metadata/local-context reads and bytes, and cannot authorize fast path when unknown/oversized/non-exact/over-budget.
- **MB6 — Checkpoint payload budget.** Shared checkpoint canonical payload soft 16 KiB / hard 32 KiB, with model-visible soft 2,000 / hard 4,000 tokens when applicable, bounded formation I/O, and explicit Class A versus Class B compaction semantics.
- **MB7 — Changed-fetch budget.** Exact `R` obeys checkpoint hard limit; re-fetch/recompute stays within one cumulative changed-fetch scope with hard object-read and local/remote byte limits; each classification uses exact `R` without creating a budget reset boundary.
- **MB8 — Compaction tiers.** C0 deduplication, C1 reference substitution, C2 semantic summary, C3 split presentation, C4 hard stop are deterministic, cumulatively accounted, and do not silently mutate source truth.
- **MB9 — Fidelity verification.** Compaction failure to preserve protected semantics/provenance is a typed non-success, not warning-only success; fidelity success alone does not reconcile a Class B derivative.
- **MB10 — Recovery budget.** Recovery has a larger but finite cumulative staged evidence envelope with hard local/remote object-read and byte limits; streaming/pagination cannot evade it, and insufficient evidence remains unresolved rather than successful.
- **MB11 — Deterministic over-budget outcomes.** Budget states remain typed and cannot advance markers, resolve conflicts, authorize publication, repair lineage, or reset accounting by themselves.
- **MB12 — Candidate/context invalidation after compaction.** Class A semantics-preserving representation may retain semantic context only when equivalence/current identities are proven; Class B minimization invalidates old C/reconciliation/resolution/CAS/privacy contexts, forms new L/derivative, reruns current ADR-0003 B/R/L reconciliation or explicit resolution, produces new C, reruns schema/cross-field/provenance checks, and obtains fresh ADR-0005 privacy `ALLOW` before any write.
- **MB13 — No silent truncation or fallback.** No hard-limit path silently truncates protected/required material, resets cumulative counters, treats unknown as success, substitutes candidate identity to reuse old conclusions, or falls back to transport.
- **MB14 — Protocol-only scope.** M1.8 fixes measurable architecture budgets only; implementation/API/locks/adapter/publication trigger/detector/migration/real data/UI remain deferred.

## 15. Initial-review clarification record

The initial review was performed against:

```text
commit:
8bd7ac87cc903aae9fafcdce03ded1fbccc719a7

proposal Git blob LF SHA-256:
A1A409AC5D2D2D15367470DB401D1115F5E118DE42621463BDAE918E97BAAEA1
```

Verdict:

```text
MB1  REVISE
MB2  APPROVE
MB3  APPROVE
MB4  REVISE
MB5  REVISE
MB6  REVISE
MB7  REVISE
MB8  APPROVE
MB9  APPROVE
MB10 REVISE
MB11 APPROVE
MB12 REVISE
MB13 APPROVE
MB14 APPROVE

blocking defects:
B1 — cumulative logical-I/O budget scope / triggerable hard limits
B2 — post-compaction ADR-0003 candidate re-formation

M1.8: NOT APPROVE
```

This revision addresses **only B1 and B2**. B1 is addressed by one cumulative `budget_accounting_scope`, explicit cumulative object-read/byte soft/hard limits, and no-reset semantics across pagination, staging, compaction, route restart, re-fetch/recompute, and recovery streaming. B2 is addressed by the Class A / Class B distinction and the mandatory ADR-0003 re-formation plus fresh ADR-0005 privacy gate for semantic minimization/derivatives.

No other MB decision is reopened or expanded by this clarification.

## 16. Deferred implementation and next milestone

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

## 17. Maturity boundary

This proposal is architecture only:

```text
runtime_effective: false
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
```

M1.8 approval would not mean runtime budgeting, compaction code, probe logic, adapter behavior, migration, or validation has been implemented.