---
document_role: architecture-decision-record
adr_id: ADR-0007
status: accepted
accepted: 2026-08-27
milestone: M1.8
runtime_effective: false
release_required_for_runtime_effect: true
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
proposal: docs/proposals/MEMORY_PAYLOAD_COMPACTION_BUDGETS.md
final_approval: docs/reviews/2026-08-27-memory-payload-compaction-budgets-final-approval.md
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
  - docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md
  - docs/decisions/ADR-0004-hub-adapter-and-transport-boundary.md
  - docs/decisions/ADR-0005-privacy-never-publish-policy.md
  - docs/decisions/ADR-0006-normal-fast-changed-remote-recovery-paths.md
---

# ADR-0007 — Memory payload / compaction budgets

## Status

**Accepted architecture decision.**

This ADR records the approved M1.8 measurable resource envelopes for startup, probe, shared checkpoint payload, changed fetch, compaction, and recovery in the future unified Project Memory architecture.

It is normative for later unified-architecture design and implementation, but it does **not** change current released Skill, Runtime Hub, local Project Memory, adapter, transport, migration, or validation behavior.

Current maturity remains:

```text
runtime_effective: false
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
```

## Decision

MB1–MB14 from `docs/proposals/MEMORY_PAYLOAD_COMPACTION_BUDGETS.md` are accepted in full.

### 1. Measurable budget dimensions

The bounded surfaces use measurable architecture-level dimensions rather than implementation-specific allocator/API behavior:

- model-visible tokens;
- logical Project Memory bytes retained for the operation;
- logical local/remote object reads;
- local content bytes;
- remote metadata bytes;
- remote content bytes;
- canonical checkpoint payload bytes.

Implementation-specific RSS, HTTP calls, Git commands, syscalls, retry mechanics, or transport buffers are not substituted for these logical budget dimensions.

### 2. Cumulative accounting scope

Each bounded top-level startup/sync/recovery operation has one conceptual `budget_accounting_scope` with monotonic counters.

Pagination, staging, compaction tiers, ADR-0006 route restart/re-selection, exact re-fetch, ADR-0003 stale recomputation, explicit-resolution recomputation, and recovery streaming do **not** reset cumulative accounting inside that operation.

Re-reading/re-fetching an object is charged again. If accounting becomes unknown or a hard cumulative limit is exceeded, the bounded operation cannot report in-budget success.

### 3. Accepted default envelopes

| Surface | Model-visible tokens soft / hard | Logical memory soft / hard | Key cumulative I/O / payload limits |
| --- | ---: | ---: | --- |
| Normal startup | 1,500 / 2,000 | 128 / 256 KiB | local reads 12 / 24; local bytes 256 / 512 KiB; remote metadata reads 1 / 2; metadata bytes 4 / 16 KiB; remote checkpoint body reads/bytes 0 / 0 |
| Probe | 0 / 0 | 16 / 32 KiB | remote metadata reads 1 / 2; metadata bytes 4 / 16 KiB; local context reads 2 / 4; local bytes 16 / 32 KiB; checkpoint body reads/bytes 0 / 0 |
| Shared checkpoint payload | 2,000 / 4,000 when model-visible | 64 / 128 KiB | local reads 24 / 48; local bytes 512 KiB / 1 MiB; canonical serialized checkpoint 16 / 32 KiB |
| Changed fetch | 4,000 / 8,000 | 256 / 512 KiB | remote checkpoint reads 2 / 4; remote bytes 64 / 128 KiB; remote metadata reads 4 / 8; metadata bytes 32 / 64 KiB; local reads 24 / 48; local bytes 512 KiB / 1 MiB; each checkpoint body hard 32 KiB |
| Compaction working set | destination-governed | 512 KiB / 1 MiB | local reads 32 / 64; local bytes 1 / 2 MiB; remote reads 4 / 8; remote bytes 128 / 256 KiB; remote metadata reads 8 / 16; metadata bytes 64 / 128 KiB |
| Recovery presentation | 6,000 / 12,000 | 1 / 4 MiB | local reads 32 / 64; local bytes 2 / 8 MiB; remote reads 8 / 16; remote bytes 256 / 512 KiB; metadata reads 16 / 32; metadata bytes 64 / 128 KiB; concurrent retained objects 8 / 16 |

These are architecture defaults, not claims that a current runtime meets them. Later measured evidence may justify an explicit superseding decision; silent runtime drift is not allowed.

### 4. Soft thresholds and hard limits

A soft exceedance triggers deterministic selection/minimization/compaction/recovery behavior and remeasurement.

A hard exceedance is typed non-success unless an accepted alternative produces a valid in-budget representation without resetting the same top-level accounting scope.

No hard-limit path may silently truncate required material and still claim success.

### 5. Protected semantic fidelity floor

Compaction/resource pressure must preserve 100% semantic fidelity for applicable protected classes:

```text
AUTHORITY
DECISION
ACTIVE_TASK
CONFLICT
PRIVACY
UNRESOLVED_STATE
RECONCILIATION_IDENTITY
PROVENANCE_REQUIRED_FOR_MEANING
```

The guarantee is semantic rather than verbatim. Wording may shrink, but current authority, accepted decision state, active task/next action, unresolved conflict/state, privacy prohibition/classification, observed-vs-reconciled meaning, and required provenance may not be weakened, omitted, or converted into success.

Failure to prove the protected fidelity floor is a typed compaction failure.

### 6. Probe remains body-free

The ADR-0006 lightweight exact probe remains zero model-visible tokens and zero checkpoint-body reads/bytes. Probe metadata/local-context work is bounded cumulatively.

Unknown, oversized, non-exact, over-budget, or unaccountable probe results cannot authorize the unchanged fast path.

### 7. Changed fetch remains exact ADR-0003 reconciliation input

A changed remote still requires exact `R` plus ADR-0003 B/R/L reconciliation. An oversized/incomplete checkpoint body cannot be partially treated as exact `R`.

Exact re-fetch/recompute remains inside the same cumulative changed-fetch scope; each new exact observation/fetch consumes budget rather than opening a fresh allowance.

### 8. Deterministic compaction tiers

Compaction proceeds, when applicable, through:

```text
C0 structural deduplication
C1 reference substitution
C2 bounded semantic summary
C3 explicit split presentation
C4 hard stop
```

C3 is explicit continuation/reference presentation, not hidden truncation. C4 is typed non-success.

### 9. Class A versus Class B representation changes

A **Class A** semantics-preserving presentation/serialization rewrite may retain existing semantic reconciliation state only when semantic equivalence and all inherited context identities remain valid.

A **Class B** semantic minimization/derivative changes shared semantic state or meaning-bearing content. It must not reuse old candidate conclusions. The required sequence is:

```text
old C invalid
old reconciliation/resolution/CAS/privacy contexts invalid
-> form new L / derivative
-> capture current B + exact R
-> rerun ADR-0003 B/R/L reconciliation
   or current exact explicit resolution
-> produce new C
-> rerun schema + cross-field + required provenance checks
-> obtain fresh ADR-0005 privacy ALLOW before any remote write
-> only then enter write-capable CAS
```

Replacing only candidate identity/SHA/token/context values does not make old conclusions reusable.

### 10. Recovery is larger but finite

Recovery may stage evidence and use a larger envelope, but pagination/streaming cannot evade cumulative hard limits. Staying under a concurrent-resident-object limit while sequentially reading unlimited evidence is prohibited.

If budget limits prevent enough evidence from resolving uncertainty, recovery remains unresolved and returns a typed over-budget evidence requirement. It must not claim remote-current, synchronized, reconciled, conflict-resolved, or privacy-approved status.

### 11. Budget outcomes do not change semantic authority

Budget states do not by themselves:

- advance `B/reconciled_sha`;
- authorize publication/CAS;
- resolve conflicts;
- repair binding/base lineage;
- weaken privacy;
- turn recovery into synchronization success;
- authorize transport fallback.

ADR-0001 through ADR-0006 continue to govern those semantics.

### 12. Deferred implementation

This ADR does not choose or implement tokenizer/model SDK, allocator/RSS instrumentation, probe/fetch API, Git/GitHub API, sync-metadata physical encoding, lock/Windows/filesystem primitives, retry values, adapter code, publication trigger, privacy detector, migration, real-data corpus, UI, or executable/static/live validation.

## Evidence and review

- proposal: `docs/proposals/MEMORY_PAYLOAD_COMPACTION_BUDGETS.md`
- final approval: `docs/reviews/2026-08-27-memory-payload-compaction-budgets-final-approval.md`
- initial baseline: `8bd7ac87cc903aae9fafcdce03ded1fbccc719a7`
- revised final-review baseline: `009b379f3f978223c1e28ad4906872ed9f4a6c48`
- revised proposal Git-blob LF SHA-256: `B8713A83AC6A980A827014517DAD7CD4A6858D2F190F0C1BC3B57B3E9CBFCCE1`

Final review approved MB1–MB14, closed B1 and B2, approved M1.8, and required no further semantic changes.
