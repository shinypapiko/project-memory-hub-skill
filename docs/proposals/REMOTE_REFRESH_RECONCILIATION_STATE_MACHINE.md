---
document_role: architecture-proposal
status: approved-proposal
normative: false
architecture_state: accepted-unreleased
runtime_load_policy: maintenance-only
milestone: M1.4
created: 2026-08-27
last_reviewed: 2026-08-27
review_state: final-approved-promoted-to-adr
review_record: docs/reviews/2026-08-27-remote-refresh-reconciliation-review.md
final_review_record: docs/reviews/2026-08-27-remote-refresh-reconciliation-final-approval.md
approved_by: docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
---

# Remote refresh and reconciliation state machine — M1.4

## 0. Purpose and scope

This proposal defines deterministic remote refresh/reconciliation semantics for the future unified Project Memory architecture.

It refines the B/R/L reconciliation principle accepted by `ADR-0001` into an explicit state machine suitable for later executable implementation. It remains **proposal-only** and does not alter current Hub Skill `0.1.0`, Runtime Hub schema-1 behavior, or the deployed local Skill.

M1.4 is limited to:

- the meaning of `B`, `R`, `L`, and candidate `C`;
- canonical local base-pair identity and drift;
- `observed_sha` versus `reconciled_sha`;
- field-level three-way classification inside the approved shared-checkpoint schema;
- an explicit same-field conflict-resolution operation;
- compare-and-set (CAS) / stale retry semantics;
- bounded remote contention;
- deterministic conflict outcomes;
- post-CAS confirmation/finalization;
- unknown/ambiguous remote or local commit handling;
- base-loss and `ABSENT` states;
- exact advancement rules for remote and local reconciliation markers.

Out of scope:

- Hub adapter code or CLI/API design;
- exact GitHub/API primitives;
- publication triggers or milestone-selection policy;
- privacy-classification implementation;
- migration;
- local filesystem safe-write implementation details;
- startup/fast-path policy beyond what is required to define base trust;
- token budgets.

## 1. Core state definitions

### 1.1 Canonical local base pair `P`

The future unified system has one canonical local representation of the last fully reconciled shared base:

```text
P = (B, reconciled_sha)
```

`B` and `reconciled_sha` are inseparable. The canonical local representation also exposes an opaque concurrency identity:

```text
base_pair_token = identity/version of the persisted canonical P
```

The exact storage/API for `P` and `base_pair_token` is deferred. Semantically, any operation that reads `P` captures all three:

```text
B0
reconciled_sha0
base_pair_token0
```

Another local session may advance canonical `P` while a remote reconciliation operation is in flight. That is **local base drift** and must be detected explicitly.

### 1.2 `B` — reconciled base checkpoint

`B` is the semantic shared checkpoint in canonical local pair `P`.

Its remote identity is `reconciled_sha`.

Invariant:

```text
B <-> reconciled_sha
```

A SHA must never be labeled reconciled if the corresponding semantic checkpoint is missing, mismatched, or was never fully reconciled.

`B` is not merely the last remote file observed.

### 1.3 `R` — current loaded remote checkpoint

`R` is the remote shared-checkpoint content actually fetched and normalized for a reconciliation attempt.

It has an exact remote revision identity:

```text
R_sha
```

For a reconciliation attempt to use `R`, the content and `R_sha` must identify the same remote revision.

A metadata-only observation may discover a different/newer `observed_sha` without loading its body. Once that happens, an older loaded `R` is not assumed to represent current head; it may still be a valid historical revision for an already-bound operation, but a fresh reconciliation/CAS attempt must load the revision it intends to classify/write against.

### 1.4 `L` — immutable local shared-checkpoint proposal for one operation

`L` is the local side's **normalized semantic proposal over the shared-checkpoint schema** for one reconciliation operation.

It is not raw `.ai/PROJECT.md`, `.ai/CURRENT.md`, `TASKS`, DEC/EXP, sessions, or other local files. Projection into `L` occurs before M1.4 under `ADR-0001` and later publication/privacy policy.

For one operation, `L` is immutable and normalized relative to the captured `B0`.

If canonical local `P` changes, old `L` must **not** simply be reinterpreted against the new `B`. The old operation terminates and the caller must re-project current local shared intent against the new canonical base to produce a new `L'` for a new reconciliation operation.

If re-projection cannot be performed safely, return a state such as:

```text
LOCAL_REPROJECTION_REQUIRED
```

Sparse source omission is not an implicit destructive delete.

### 1.5 `C` — candidate reconciled checkpoint

`C` is produced by automatic three-way reconciliation or by an explicit conflict-resolution operation:

```text
C = reconcile(B, R, L)
```

or, for explicitly resolved conflicts:

```text
C = reconcile_with_resolution(B, R, L, X)
```

where `X` is defined in §5.

Every candidate is bound to an exact dependency identity:

```text
candidate_context = (
  base_pair_token,
  reconciled_sha,
  R_sha,
  L_identity,
  resolution_context_if_any
)
```

If any dependency changes, the candidate is invalid. An invalidated candidate is never replayed merely by substituting a newer SHA/token.

### 1.6 `observed_sha`

`observed_sha` is the most recent remote checkpoint revision identity successfully observed through a trusted read/probe.

It means only:

> the remote endpoint was observed at this revision/state.

It does **not** mean the revision is loaded, compatible, incorporated, current as a write base, or reconciled.

Therefore:

```text
observed_sha may differ from reconciled_sha
```

For protocol semantics, `observed_sha` may also take the sentinel `ABSENT` when trusted observation proves that the remote checkpoint target does not currently exist. That observation does not by itself prove first-time initialization.

### 1.7 `reconciled_sha`

`reconciled_sha` is the remote revision corresponding to canonical local `B`.

It advances only through guarded canonical-local finalization in §11.

A newly observed remote SHA must never automatically become `reconciled_sha`.

A special sentinel `ABSENT` is permitted only for a **trusted initialized absent baseline**, never merely because the remote currently appears absent.

## 2. Required invariants

The design is invalid if any of the following can happen silently:

1. `observed_sha` is copied into `reconciled_sha` merely because it is newer;
2. `reconciled_sha` advances without canonical `B` advancing to the exact corresponding semantic checkpoint;
3. candidate use ignores a changed canonical local `base_pair_token`;
4. old `L` is reinterpreted against a different canonical `B` after local base drift;
5. a candidate computed from `R1` is written against `R2` after CAS reports stale;
6. a stale CAS is handled by only replacing the expected SHA while replaying old candidate bytes;
7. a divergent same-field conflict is silently resolved by default local-wins/remote-wins;
8. an explicit conflict resolution is replayed after its bound B/R/L conflict context changed;
9. arbitrary Markdown line merging substitutes for semantic field reconciliation;
10. field-by-field output bypasses whole-checkpoint validation;
11. remote CAS success alone is treated as persistent local reconciliation success;
12. a local base-pair persistence failure/unknown result is reported as reconciled;
13. an exact remote commit that cannot be confirmed is treated as confirmed;
14. an already-newer remote head forces `reconciled_sha` to skip directly to that head without reconciliation;
15. unknown remote commit state is treated as definite success/failure without re-observation;
16. retry can continue without a bounded contention budget;
17. a conflict/unknown/base-loss state advances `reconciled_sha`;
18. identity/schema incompatibility is merged as normal content;
19. missing/mismatched `B` content is accepted because a `reconciled_sha` string exists;
20. current remote `ABSENT` is assumed to mean first creation when lineage is unknown;
21. disappearance of a previously reconciled remote checkpoint is silently treated as ordinary omission or fresh initialization;
22. full local memory files enter the B/R/L merge domain.

## 3. Reconciliation domain and normalization

Only fields defined by the accepted shared-checkpoint schema participate.

Each logical field is compared semantically rather than by Markdown line position. A logical field may be scalar, structured, or a collection; if it lacks a deterministic merge policy, it is treated atomically.

M1.4 does not define generic list/set union. Collection fields require schema-specific semantic equality/merge rules; otherwise concurrent divergent edits conflict.

The normalizer must distinguish explicit semantic absence/deletion states where the eventual schema supports them. Mere source omission is not silently reinterpreted as destructive delete.

Unknown fields or incompatible schema revisions produce a fail-closed schema outcome.

Checkpoint-level `ABSENT` is distinct from a field being absent inside an existing checkpoint.

## 4. Per-field automatic three-way classification matrix

For one normalized logical field `f`:

```text
b = state of f in B
r = state of f in R
l = state of f in L

remote_changed = (r != b)
local_changed  = (l != b)
```

Semantic equality, not textual equality, is used.

| Remote changed? | Local changed? | `r == l` | Classification | Automatic candidate field |
| --- | --- | --- | --- | --- |
| no | no | yes | `UNCHANGED` | `b` |
| yes | no | — | `REMOTE_ONLY` | `r` |
| no | yes | — | `LOCAL_ONLY` | `l` |
| yes | yes | yes | `CONVERGENT_BOTH` | `r` / `l` |
| yes | yes | no | `FIELD_CONFLICT` | none; automatic reconciliation stops |

Additional hard-stop classifications precede the table:

- `IDENTITY_CONFLICT` — project/workspace/binding identity mismatch;
- `SCHEMA_INCOMPATIBLE` — checkpoint schema cannot be deterministically interpreted;
- `UNMERGEABLE_FIELD_POLICY` — field requires unavailable/ambiguous merge semantics;
- `REMOTE_CHECKPOINT_ABSENT_CONFLICT` — a previously present reconciled checkpoint has disappeared and no accepted explicit deletion/tombstone semantics authorize that transition.

### 4.1 Cross-field validation

After automatic or explicit field choices assemble a candidate `C`, the whole checkpoint must pass schema and cross-field invariant validation.

Failure returns:

```text
CANDIDATE_INVALID
```

and no remote write/final reconciliation advancement occurs.

## 5. Explicit conflict-resolution operation `X`

Automatic three-way reconciliation never chooses local-wins or a third value for `FIELD_CONFLICT`.

However, a conflict must be resolvable without forcing the user/agent into an impossible ordinary `L2` loop. M1.4 therefore defines a separate **explicit resolution operation**.

### 5.1 Bound conflict context

When automatic reconciliation returns one or more field conflicts, it also produces a conflict-context identity conceptually containing:

```text
conflict_context = (
  base_pair_token,
  reconciled_sha,
  R_sha,
  L_identity,
  conflict_field_paths,
  identities/hashes of conflicting B/R/L field states
)
```

A resolution plan `X` is valid only for that exact context.

### 5.2 Allowed explicit directives

For each conflicting field, `X` may explicitly choose one schema-valid action:

- `TAKE_REMOTE` — use the bound remote value/state;
- `TAKE_LOCAL` — use the bound local value/state;
- `SET_EXPLICIT(value_or_state)` — use a separately specified schema-valid third value/state, including an explicit deletion/tombstone only if the shared schema/policy permits it.

No directive is inferred from silence.

### 5.3 Resolution application

A resolution operation:

1. re-checks that canonical local `base_pair_token/reconciled_sha` still match the bound context;
2. re-checks that the remote revision being resolved is still exactly `R_sha`;
3. re-validates identity/schema compatibility;
4. applies ordinary matrix results to non-conflicting fields;
5. applies `X` only to the exact bound conflict fields;
6. validates the complete candidate;
7. if candidate differs from current `R`, uses CAS against the exact bound `R_sha`.

If canonical local base or remote revision changed, the old resolution context is invalid:

```text
CONFLICT_RESOLUTION_STALE
```

The old `X` is not replayed automatically. Reconciliation restarts from a fresh canonical base, a newly projected `L'`, latest remote `R'`, and a newly computed conflict set. A new explicit resolution may then re-confirm equivalent choices.

This explicit operation is still M1.4 reconciliation semantics; it does not define who/what triggers publication or choose a concrete API.

## 6. Candidate creation and pre-use local-base gate

After normalizing `B`, `R`, and immutable `L`:

1. classify identity/schema/checkpoint presence;
2. classify every shared field using §4;
3. either stop with structured conflict or, if an explicit valid `X` exists, apply §5;
4. assemble and validate `C`;
5. bind `C` to the exact candidate context;
6. **before accepting `C == R` or attempting remote CAS, re-read the canonical local base pair identity**.

If canonical local pair no longer matches captured:

```text
(base_pair_token0, reconciled_sha0)
```

return:

```text
LOCAL_BASE_STALE
```

Discard `C` and any bound resolution plan. Start a new reconciliation operation only after current local shared intent is re-projected against the new canonical `B`.

This pre-use check does not attempt to hold a local lock across a remote network operation. A second post-CAS/finalization check is therefore also required.

## 7. Abstract remote CAS contract

M1.4 assumes an abstract remote compare-and-set operation:

```text
CAS(expected_remote_sha = R_sha, candidate = C)
```

Possible abstract outcomes:

- `CAS_COMMITTED(commit_revision)` — remote reports that the candidate was committed from expected base;
- `CAS_STALE(current_revision?)` — remote changed since `R`;
- `CAS_AMBIGUOUS` — request outcome does not prove committed vs not committed;
- `CAS_FAILED` — definite non-stale failure with no successful commit known.

`CAS_COMMITTED` is **not yet** persistent local reconciliation success. It enters exact-revision confirmation and local finalization (§10–§11).

M1.4 does not select the concrete remote API.

## 8. CAS / stale retry loop

Conceptually:

```text
capture canonical P0 = (B0, reconciled_sha0, base_pair_token0)
project immutable L0 relative to B0
attempt_budget = bounded

LOOP:
    fetch latest usable remote R + R_sha
    observed_sha = R_sha

    C = reconcile(B0, R, L0) or explicit-resolution equivalent

    if conflict:
        return structured conflict

    recheck canonical local base pair
    if local base drifted:
        discard C / X / L0 operation
        return LOCAL_BASE_STALE (fresh projection required)

    if C == R:
        guarded-local-finalize expected base_pair_token0 -> (R, R_sha)
        return according to §11

    result = CAS(expected=R_sha, candidate=C)

    if COMMITTED:
        confirm exact committed revision contains C
        finalize canonical local base pair under expected token
        return according to §10–§11

    if STALE:
        discard C completely
        consume retry budget
        fetch Rnew
        recompute from same still-valid B0 + Rnew + L0
        continue

    if AMBIGUOUS:
        enter UNKNOWN_COMMIT recovery (§9)

    if definite failure:
        return REMOTE_WRITE_FAILED
```

### 8.1 Mandatory stale behavior

Forbidden:

```text
CAS(R1_sha, C1) -> STALE(R2_sha)
CAS(R2_sha, C1)
```

Required:

```text
CAS(R1_sha, C1) -> STALE
fetch R2
C2 = reconcile(B0, R2, L0)
validate C2
recheck canonical local base
CAS(R2_sha, C2) only if still required
```

If canonical local base changes during any stale-retry sequence, the operation stops; it does **not** continue with old `L0` against new `B`.

## 9. Unknown / ambiguous remote commit handling

On `CAS_AMBIGUOUS`:

1. mark operation transiently `COMMIT_UNCERTAIN`;
2. do not advance canonical local `B/reconciled_sha`;
3. re-observe/fetch remote state;
4. update `observed_sha` only from actual observation;
5. compare current remote state against the bound candidate and then re-run B/R/L classification as necessary.

Possible outcomes:

### A. Current remote revision exactly contains the validated candidate

If the current observed revision is proven to contain `C`, it becomes a confirmed candidate revision and proceeds to **local finalization** (§11). Do not skip the canonical local base-drift check.

### B. Current remote differs but is merge-compatible

Discard old candidate and recompute from the still-valid captured base and immutable `L`, subject to bounded contention and local-base checks.

### C. Current remote conflicts

Return structured conflict. Do not advance `reconciled_sha`.

### D. Remote cannot be re-observed sufficiently to classify commit state

Return:

```text
REMOTE_COMMIT_STATE_UNKNOWN
```

No reconciliation marker advances.

### 9.1 No blind rollback

Never roll remote state back to the pre-attempt snapshot merely because commit state is uncertain. Recovery uses fresh observation.

## 10. Post-CAS exact-revision confirmation

A reported remote commit has two independent questions:

1. **Was candidate `C` actually installed at a specific remote revision `S`?**
2. **What is the remote head now?**

These must not be conflated.

### 10.1 Exact candidate revision confirmed

A commit revision `S` is confirmable only when trusted remote evidence proves that exact revision `S` contains the accepted candidate `C` (or a semantically exact accepted serialization of it).

The implementation may satisfy this using a sufficiently strong CAS response or an exact-revision read; M1.4 does not choose how.

If confirmation succeeds, `S` becomes eligible for canonical local finalization.

### 10.2 Remote head already advanced beyond `S`

Another writer may advance remote head to `T` before or during confirmation.

If exact `S` is still confirmed as candidate `C`:

```text
confirmed_candidate_sha = S
observed_sha = T   # if T was actually observed
```

It is valid for canonical local reconciliation to advance only to `S` while `observed_sha` is already `T`:

```text
reconciled_sha = S
observed_sha = T
```

This means: candidate `S` was reconciled, but newer remote work remains unreconciled and must be processed next. The protocol must not skip directly to `T`.

Return may distinguish:

```text
RECONCILED_COMMITTED_REMOTE_ADVANCED
```

from the simpler `RECONCILED_COMMITTED`.

### 10.3 Exact revision cannot be confirmed

If CAS reported success but the system cannot prove that exact revision contains `C`, return/enter:

```text
REMOTE_COMMIT_CONFIRMATION_UNKNOWN
```

Do not advance canonical local `B/reconciled_sha`.

Fresh observation may later show a state equal/compatible with `C`; that is handled by §9/new reconciliation, not by assuming the unconfirmed revision succeeded.

## 11. Guarded canonical-local finalization

Remote semantic reconciliation becomes durable for this local Project Memory instance only when canonical local pair `P=(B,reconciled_sha)` is successfully advanced.

This is a guarded local state transition, not an in-memory assignment.

Conceptually:

```text
LOCAL_ADVANCE(
  expected_base_pair_token = base_pair_token0,
  new_B = accepted_remote_semantic_state,
  new_reconciled_sha = accepted_remote_revision
)
```

The exact helper/storage uses the accepted local concurrency guarantees from `ADR-0002`; M1.4 does not select implementation APIs.

### 11.1 No-write case (`C == R`)

A no-write reconciliation is successful only if:

1. `R/R_sha` passed all reconciliation validation;
2. canonical local base pair still equals captured `P0`;
3. guarded local advance persists `(R, R_sha)` as the new canonical pair.

Only then return:

```text
RECONCILED_NO_WRITE
```

If local base drifted before advance:

```text
LOCAL_BASE_STALE
```

Old candidate/`L` operation terminates and new projection is required.

If local persistence definitely fails:

```text
LOCAL_RECONCILIATION_FINALIZE_FAILED
```

If local persistence result is ambiguous, re-read the canonical local pair:

- if it equals the intended new pair, classify finalization as committed;
- if it remains exactly the old expected pair, classify finalization as not committed/failed;
- if it equals another newer pair, classify `LOCAL_BASE_STALE`;
- if it cannot be read/verified, return `LOCAL_RECONCILIATION_STATE_UNKNOWN`.

No ambiguous local result may be reported as `RECONCILED_NO_WRITE` without that reclassification.

### 11.2 Remote committed and confirmed case

After exact remote revision `S` is confirmed to contain `C`, canonical local finalization still uses expected `base_pair_token0`.

Outcomes:

- guarded local advance commits `(C,S)` → reconciliation succeeds;
- canonical local pair already changed → `REMOTE_COMMITTED_LOCAL_BASE_STALE`;
- definite local persistence failure → `REMOTE_COMMITTED_LOCAL_FINALIZE_FAILED`;
- ambiguous local persistence → re-read/classify as in §11.1; if still unknowable, `LOCAL_RECONCILIATION_STATE_UNKNOWN`.

A newer local base must never be overwritten merely to record a remote commit that happened from an older operation base.

### 11.3 Recovery after remote commit but local finalization failure/crash

A process may crash or fail after remote commit but before canonical local `P` advances.

Safety rule:

```text
persistent reconciled_sha remains old
remote may now be ahead
```

On the next operation, that remote state is treated as **unreconciled remote input** relative to the still-persistent canonical base. It is not auto-adopted merely because it may have been written by this process.

A durable operation journal may later optimize recovery, but M1.4 does not require/select one.

This conservative rule makes a crash recoverable without falsely claiming local reconciliation completion.

## 12. Bounded contention and liveness

Remote CAS/re-observation retries must have an explicit, testable attempt/time budget in the later implementation.

M1.4 does not fix numeric values.

On exhaustion return:

```text
REMOTE_CONTENTION_EXHAUSTED
```

Requirements:

- no infinite retry loop;
- no last-writer-wins fallback;
- no force overwrite;
- preserve latest actually observed `observed_sha` where available;
- do not advance `reconciled_sha` unless guarded local finalization already completed.

A local base-drift termination is not silently converted into another retry against old `L`; it requires a fresh projection/new operation.

## 13. SHA/state marker advancement rules

### 13.1 `observed_sha`

`observed_sha` may change only from trusted remote observation/report semantics sufficient to identify an actual remote state.

Allowed examples:

- trusted probe/read reports current SHA `S` → `observed_sha := S`;
- trusted observation proves checkpoint currently absent → `observed_sha := ABSENT`;
- a CAS response may supply a revision identity only if its semantics make that identity a trusted observation; otherwise re-observe first.

Not allowed:

- predicted candidate hash;
- desired SHA;
- locally constructed content identity with no remote observation.

`observed_sha` carries no reconciliation guarantee.

### 13.2 `reconciled_sha` and `B`

They advance only as one guarded canonical local pair transition after a remote state has been fully accepted/confirmed.

Remote CAS success, remote observation, candidate construction, or in-memory assignment alone never advances persistent reconciliation.

### 13.3 Newer remote head after successful reconciliation

If candidate revision `S` was confirmed and canonical local pair persisted at `S`, but newer head `T` is observed:

```text
B = semantic state at S
reconciled_sha = S
observed_sha = T
```

This is valid and explicitly represents unreconciled newer remote work.

### 13.4 States that must not advance canonical `reconciled_sha`

No advancement on:

- metadata-only newer observation;
- remote load before classification;
- any unresolved conflict;
- `CONFLICT_RESOLUTION_STALE`;
- `CANDIDATE_INVALID`;
- `CAS_STALE`;
- `REMOTE_CONTENTION_EXHAUSTED`;
- `REMOTE_WRITE_FAILED`;
- unresolved `COMMIT_UNCERTAIN`;
- `REMOTE_COMMIT_STATE_UNKNOWN`;
- `REMOTE_COMMIT_CONFIRMATION_UNKNOWN`;
- `LOCAL_BASE_STALE`;
- `REMOTE_COMMITTED_LOCAL_BASE_STALE`;
- `LOCAL_RECONCILIATION_FINALIZE_FAILED`;
- `REMOTE_COMMITTED_LOCAL_FINALIZE_FAILED`;
- `LOCAL_RECONCILIATION_STATE_UNKNOWN`;
- `RECONCILIATION_BASE_REQUIRED`;
- `RECONCILIATION_BASE_INVALID`;
- `REMOTE_CHECKPOINT_ABSENT_CONFLICT`.

## 14. Base trust, base loss, and `ABSENT`

Three-way reconciliation requires a trustworthy canonical base pair.

M1.4 distinguishes these states.

### 14.1 `BASE_READY_PRESENT`

Valid when:

- canonical `B` content is available/normalizable;
- `reconciled_sha` identifies the remote revision that produced that exact semantic base;
- their linkage passes integrity checks;
- canonical base-pair token is readable.

Normal M1.4 reconciliation may proceed.

### 14.2 `BASE_READY_ABSENT`

Valid only when canonical local lineage explicitly records a trusted initialized absent baseline:

```text
B = ABSENT
reconciled_sha = ABSENT
```

This is not inferred solely from current remote absence.

A later remote checkpoint creation is then a real change from trusted absent base and may enter deterministic reconciliation/CAS semantics.

### 14.3 `BASE_UNINITIALIZED` / lineage unknown

Examples:

- `reconciled_sha` missing and no trusted initialized absent baseline exists;
- local base metadata was never established;
- local recovery cannot determine whether a checkpoint previously existed.

Whether current remote is present **or absent**, M1.4 does not invent a base.

Return:

```text
RECONCILIATION_BASE_REQUIRED
```

Establishing first binding/recovery lineage belongs to later M1.7/M3 policy.

### 14.4 `BASE_INVALID`

Examples:

- `reconciled_sha` exists but corresponding `B` content is missing;
- `B` content exists but its stored/proven linkage to `reconciled_sha` does not match;
- canonical pair is structurally corrupted or cannot be trusted.

Return:

```text
RECONCILIATION_BASE_INVALID
```

Do not substitute latest remote as base.

### 14.5 Previously present remote checkpoint becomes `ABSENT`

If canonical base is present/non-ABSENT but trusted remote observation now reports checkpoint `ABSENT`, this is **not** first creation and not ordinary field omission.

Unless an accepted explicit checkpoint-deletion/tombstone policy proves that disappearance is a valid shared semantic transition, return:

```text
REMOTE_CHECKPOINT_ABSENT_CONFLICT
```

Do not auto-recreate from local state, auto-adopt `ABSENT`, or advance `reconciled_sha` to `ABSENT`.

Deletion/purge authorization policy itself remains outside M1.4.

## 15. Conflict/result contract

Minimum conflict/invalid outcomes:

- `FIELD_CONFLICT`
- `IDENTITY_CONFLICT`
- `SCHEMA_INCOMPATIBLE`
- `UNMERGEABLE_FIELD_POLICY`
- `CANDIDATE_INVALID`
- `CONFLICT_RESOLUTION_STALE`
- `RECONCILIATION_BASE_REQUIRED`
- `RECONCILIATION_BASE_INVALID`
- `REMOTE_CHECKPOINT_ABSENT_CONFLICT`

A field conflict should identify at least:

- canonical semantic field path/key;
- base identity (`reconciled_sha` + base-pair context);
- remote identity (`R_sha`);
- `L` identity;
- safe hashes/summaries/identities of B/R/L field states;
- provenance references where permitted;
- conflict-context identity usable by explicit resolution.

Sensitive field contents need not be dumped into logs/results.

## 16. Revised state machine

Conceptually:

```text
S0 READ_CANONICAL_BASE
   -> valid P0=(B0,reconciled_sha0,base_pair_token0)
   -> else BASE_REQUIRED / BASE_INVALID
        |
        v
S1 PROJECT_LOCAL
   immutable L0 relative to B0
        |
        v
S2 LOAD_REMOTE
   R + R_sha ; observed_sha may update
        |
        v
S3 CLASSIFY
   field matrix / identity / schema / absence
    |                 |
 conflict         compatible
    |                 v
    |             S4 CANDIDATE_READY
    |                 |
    |          pre-use local-base check
    |             |             |
    |          drift          unchanged
    |             |             |
    |      STOP_REPROJECT       +----------------+
    |                                              |
    v                                              v
S3R EXPLICIT_RESOLUTION(X)                   C == R ?
  X bound to exact conflict                  /      \
  context; stale context stops             yes      no
    |                                       |        |
    +--------------> validate C             v        v
                                       S5 LOCAL    S6 REMOTE_CAS
                                       FINALIZE    /   |    \
                                          |    commit stale ambiguous
                                          |       |     |       |
                                          |       v     |       v
                                          |   S7 CONFIRM |   S8 REOBSERVE
                                          |       |     |       |
                                          |       |     +-> discard C/recompute
                                          |       |
                                          |       +-> exact S confirmed?
                                          |             |       |
                                          |            yes      no
                                          |             |       |
                                          |             v       v
                                          +--------> S9 LOCAL  CONFIRM_UNKNOWN
                                                     FINALIZE
                                                        |
                                   +--------------------+--------------------+
                                   |                    |                    |
                                committed            stale               failed/unknown
                                   |                    |                    |
                                   v                    v                    v
                              RECONCILED       REMOTE_COMMITTED_      explicit failure/
                                                LOCAL_BASE_STALE       unknown state
```

After any successful local finalization to revision `S`, a separately observed newer remote head `T` is represented as:

```text
reconciled_sha = S
observed_sha = T
```

and is reconciled in a later operation.

## 17. Structured operation outcomes

Success/accepted:

- `RECONCILED_NO_WRITE`
- `RECONCILED_COMMITTED`
- `RECONCILED_COMMITTED_REMOTE_ADVANCED`
- `RECONCILED_AFTER_UNKNOWN_COMMIT`

Remote retry/uncertain/failure:

- `REMOTE_STALE_RETRY`
- `REMOTE_CONTENTION_EXHAUSTED`
- `REMOTE_WRITE_FAILED`
- `REMOTE_COMMIT_STATE_UNKNOWN`
- `REMOTE_COMMIT_CONFIRMATION_UNKNOWN`

Local-base/finalization:

- `LOCAL_BASE_STALE`
- `LOCAL_REPROJECTION_REQUIRED`
- `LOCAL_RECONCILIATION_FINALIZE_FAILED`
- `REMOTE_COMMITTED_LOCAL_BASE_STALE`
- `REMOTE_COMMITTED_LOCAL_FINALIZE_FAILED`
- `LOCAL_RECONCILIATION_STATE_UNKNOWN`

Conflict/base invalid:

- `FIELD_CONFLICT`
- `IDENTITY_CONFLICT`
- `SCHEMA_INCOMPATIBLE`
- `UNMERGEABLE_FIELD_POLICY`
- `CANDIDATE_INVALID`
- `CONFLICT_RESOLUTION_STALE`
- `RECONCILIATION_BASE_REQUIRED`
- `RECONCILIATION_BASE_INVALID`
- `REMOTE_CHECKPOINT_ABSENT_CONFLICT`

A later executable result may add detail but must not collapse materially distinct safety states into an undifferentiated boolean.

## 18. Relationship to M1.3 local concurrency

`ADR-0002` and M1.4 protect different races but meet at canonical local base finalization.

M1.3 protects local mutable state from same-workspace lost update. M1.4 requires the canonical `B/reconciled_sha` pair to be advanced using a guarded local transition consistent with that safety model.

M1.4 must not hold a local lock throughout remote network reasoning/requests. It captures a base-pair token, checks before candidate use, and checks/compares again during finalization. If drift occurs, the newer canonical local pair wins and the old remote operation is not allowed to overwrite it.

Remote CAS still protects remote checkpoint revision races; local safe-write still protects local canonical metadata races. Neither substitutes for the other.

## 19. Revised review decisions requested for M1.4

Final review approved all of the following decisions:

- **RR1:** `B` is exactly the semantic checkpoint in canonical local pair `P=(B,reconciled_sha)`; a SHA without trustworthy matching B content is not a valid base.
- **RR2:** `observed_sha` means remote observation only, may differ from `reconciled_sha`, and `ABSENT` observation does not imply reconciliation or first initialization.
- **RR3:** `L` is immutable and normalized relative to one captured `B`; local base drift requires a new operation and fresh projection `L'`, not reinterpretation of old `L`.
- **RR4:** every candidate is bound to `(base_pair_token, reconciled_sha, R_sha, L identity, resolution context if any)`; canonical local base drift invalidates the candidate before use/finalization and requires re-projection against the new base.
- **RR5:** automatic field reconciliation uses the deterministic three-way matrix; divergent same-field conflicts stop automatically but may be resolved by an explicit context-bound operation choosing remote, local, or a schema-valid third value, with exact-context revalidation and CAS.
- **RR6:** candidate assembly must pass whole-checkpoint schema/cross-field validation after automatic and explicit field choices.
- **RR7:** `C == R` permits no remote write, but reconciliation succeeds only after guarded canonical-local `(B,reconciled_sha)` finalization; base drift, local persistence failure, or unknown local state cannot be reported as success.
- **RR8:** any remote write uses CAS against the exact `R_sha` used to compute/resolve the candidate.
- **RR9:** `CAS_STALE` always discards the old candidate and forces remote refresh plus recomputation; it never replays old candidate bytes against a new SHA.
- **RR10:** remote stale/contention retries are bounded and exhaustion returns `REMOTE_CONTENTION_EXHAUSTED` without force overwrite.
- **RR11:** ambiguous remote commits are re-observed; old candidates are not blindly retried, and unresolved ambiguity/confirmation failure cannot advance reconciliation.
- **RR12:** remote commit and local reconciliation are separate stages: exact candidate revision must be confirmed, then canonical local pair must advance under its expected token; newer remote head may leave `observed_sha > reconciled_sha`, and local finalization failure/crash leaves the old persistent base for later reconciliation.
- **RR13:** base trust explicitly distinguishes valid present base, trusted initialized `ABSENT`, uninitialized/unknown lineage, invalid/mismatched B content, and disappearance of a previously present remote checkpoint; none are silently converted into a convenient base.
- **RR14:** M1.4 remains semantic/protocol-only and does not select Hub adapter code, APIs, publication triggers, retry numbers, migration, privacy implementation, or startup fast-path policy.

## 20. Deferred implementation choices

Deferred deliberately:

- exact remote API and CAS primitive;
- exact canonical local base-pair storage/schema/token mechanism;
- exact sync metadata file/path/schema;
- whether candidate/local projection uses SHA, UUID, or another identity;
- numeric retry/backoff/time budgets;
- serialization/canonicalization format;
- exact schema version negotiation;
- durable in-flight operation journal, if any;
- privacy detection/redaction tooling;
- publication trigger policy;
- web/new-device bootstrap UI/workflow;
- migration and release behavior.

## 21. Evidence

- `docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md`
- `docs/decisions/ADR-0002-local-multi-session-safe-write.md`
- `docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md`
- `docs/concurrency.md`
- `docs/reviews/2026-08-27-remote-refresh-reconciliation-review.md`
- `docs/reviews/2026-08-27-remote-refresh-reconciliation-final-approval.md`
- `tests/results/v0.1.0-static-regression.md`

M1.4 final review approved RR1–RR14, closed B1–B4, identified no new blocking defects, and promoted the accepted semantics to `ADR-0003`. This approval must not be interpreted as executable or live validation.