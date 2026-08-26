---
document_role: architecture-proposal
status: review-required
normative: false
architecture_state: proposed-unreleased
runtime_load_policy: maintenance-only
milestone: M1.4
created: 2026-08-27
last_reviewed: 2026-08-27
review_state: initial-review-pending
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
---

# Remote refresh and reconciliation state machine — M1.4

## 0. Purpose and scope

This proposal defines deterministic remote refresh/reconciliation semantics for the future unified Project Memory architecture.

It refines the B/R/L reconciliation principle accepted by `ADR-0001` into an explicit state machine suitable for later executable implementation. It remains **proposal-only** and does not alter current Hub Skill `0.1.0`, Runtime Hub schema-1 behavior, or the deployed local Skill.

M1.4 is limited to:

- the meaning of `B`, `R`, `L`, and the merged candidate `C`;
- `observed_sha` versus `reconciled_sha`;
- field-level three-way classification inside the approved shared-checkpoint schema;
- compare-and-set (CAS) / stale retry semantics;
- bounded remote contention;
- deterministic conflict outcomes;
- unknown/ambiguous commit handling;
- exact advancement rules for remote state markers.

Out of scope:

- Hub adapter code or CLI/API design;
- exact GitHub/API primitives;
- publication triggers or milestone-selection policy;
- privacy-classification implementation;
- migration;
- local filesystem safe-write implementation;
- startup/fast-path policy beyond what is required to define remote state;
- token budgets.

## 1. Core state definitions

### 1.1 `B` — reconciled base checkpoint

`B` is the **last fully reconciled shared-checkpoint semantic state** known to this local Project Memory instance.

Its persistent identity is `reconciled_sha`.

Invariant:

```text
B <-> reconciled_sha
```

`B` and `reconciled_sha` are one logical state pair. They must advance together. A SHA must never be labeled reconciled if the corresponding checkpoint content/semantic state is unavailable or has not completed reconciliation.

`B` is not merely "the last remote file we happened to read".

### 1.2 `R` — current loaded remote checkpoint

`R` is the latest remote shared-checkpoint content actually fetched and normalized for the current reconciliation attempt.

It has an exact remote identity:

```text
R_sha = SHA/revision identity of R
```

For a reconciliation attempt to use `R`, the loaded content and its SHA must correspond to the same remote revision.

A metadata-only probe may discover a newer SHA without yet loading its content. In that case `observed_sha` may move, but the previous `R` is invalid for reconciliation until the newly observed revision is fetched.

### 1.3 `L` — immutable local shared-checkpoint proposal for one operation

`L` is the local side's **normalized semantic proposal over the shared-checkpoint schema** for one reconciliation operation.

It is not raw `.ai/PROJECT.md`, `.ai/CURRENT.md`, `TASKS`, DEC/EXP, sessions, or other local files. Projection into `L` happens before M1.4 and remains governed by `ADR-0001` and later publication/privacy milestones.

For M1.4, `L` is treated as an immutable operation input. If the caller decides that local shared intent changed materially during the operation, it must start a new reconciliation operation with a new `L` rather than mutating `L` underneath an in-flight state machine.

`L` must be normalized relative to `B` so each participating logical field has a deterministic semantic state. Sparse source omission is not treated as an implicit destructive delete.

### 1.4 `C` — candidate reconciled checkpoint

`C` is the candidate shared checkpoint produced by deterministic three-way reconciliation of:

```text
C = reconcile(B, R, L)
```

`C` is ephemeral until either:

1. it is proven semantically equal to current `R` and that remote state is accepted as reconciled; or
2. it is committed with CAS against the exact `R_sha` from which it was computed and the resulting remote state is confirmed.

Every candidate therefore has an implicit dependency tuple:

```text
candidate_base = reconciled_sha
candidate_remote = R_sha
candidate_local = identity of immutable L snapshot
```

If any dependency changes, the old candidate is invalid and must not be replayed.

### 1.5 `observed_sha`

`observed_sha` is the most recent remote checkpoint SHA/revision identity successfully observed through a trusted remote read/probe.

It means only:

> "the remote endpoint was observed at this revision."

It does **not** mean:

- the checkpoint body has been loaded;
- the revision is compatible with local state;
- conflicts have been resolved;
- local state has incorporated it;
- the revision is safe to use as a write base;
- the revision is reconciled.

Therefore:

```text
observed_sha may be newer than reconciled_sha
```

This is normal and must not be treated as an error.

### 1.6 `reconciled_sha`

`reconciled_sha` is the exact remote revision corresponding to `B`, the last remote shared checkpoint that has completed semantic reconciliation.

It may advance only under the rules in §10.

A newly observed SHA must **never** automatically become `reconciled_sha`.

## 2. Required invariants

The design is invalid if any of the following can happen silently:

1. `observed_sha` is copied into `reconciled_sha` merely because it is newer;
2. `reconciled_sha` advances without `B` advancing to the exact corresponding semantic checkpoint;
3. a candidate computed from `R1` is written against `R2` after CAS reports stale;
4. a stale CAS is handled by only replacing the expected SHA while replaying old candidate bytes;
5. a same-field divergent change is silently resolved by local-wins or remote-wins default;
6. arbitrary Markdown line merging substitutes for semantic field reconciliation;
7. field-by-field merge output bypasses whole-checkpoint schema/invariant validation;
8. unknown commit state is treated as definite success/failure without re-observation;
9. retry can continue without a bounded contention budget;
10. a conflict/unknown state advances `reconciled_sha`;
11. identity/schema incompatibility is merged as normal content;
12. full local memory files enter the B/R/L merge domain.

## 3. Reconciliation domain and normalization

Only fields defined by the accepted shared-checkpoint schema participate.

Each logical field is compared semantically rather than by Markdown line position. A logical field may be a scalar, structured object, or collection, but if it lacks a deterministic merge policy it is treated atomically as one field.

M1.4 does not define generic list/set union. Collection fields require schema-specific semantic equality/merge rules; otherwise concurrent divergent edits conflict.

The normalizer must distinguish semantic values required by the schema, including explicit absence/deletion markers where the eventual schema supports them. Mere omission in a source projection must not be silently reinterpreted as a destructive delete.

Unknown fields or incompatible schema revisions produce a fail-closed schema outcome rather than being casually preserved/dropped.

## 4. Per-field three-way classification matrix

For one normalized logical field `f`:

```text
b = value/state of f in B
r = value/state of f in R
l = value/state of f in L

remote_changed = (r != b)
local_changed  = (l != b)
```

Semantic equality, not textual equality, is used.

| Remote changed? | Local changed? | `r == l` | Classification | Candidate field |
| --- | --- | --- | --- | --- |
| no | no | yes | `UNCHANGED` | `b` |
| yes | no | — | `REMOTE_ONLY` | `r` |
| no | yes | — | `LOCAL_ONLY` | `l` |
| yes | yes | yes | `CONVERGENT_BOTH` | `r` / `l` |
| yes | yes | no | `FIELD_CONFLICT` | none; stop/fail closed |

This matrix naturally covers creation, clearing/absence, and replacement so long as those are represented as normalized semantic states.

Additional hard-stop classifications precede the table:

- `IDENTITY_CONFLICT` — project/workspace/binding identity mismatch;
- `SCHEMA_INCOMPATIBLE` — checkpoint schema cannot be deterministically interpreted;
- `UNMERGEABLE_FIELD_POLICY` — a field requires a merge policy that is unavailable/ambiguous.

### 4.1 Cross-field validation

A set of individually compatible field choices may still produce an invalid checkpoint.

After all fields classify without conflict, the assembled `C` must pass whole-checkpoint validation, including schema and cross-field invariants.

Failure returns:

```text
CANDIDATE_INVALID
```

and no remote write occurs.

## 5. Deterministic reconciliation outcomes before write

After `B`, `R`, and immutable `L` are normalized:

1. classify identity/schema compatibility;
2. classify every shared field using §4;
3. if any field conflicts, return a structured conflict result;
4. assemble and validate `C`;
5. compare `C` semantically with `R`.

If:

```text
C == R
```

no remote write is required. The state machine may proceed to **accept-current-remote** (§10.2) and advance `B/reconciled_sha` only after the reconciliation acceptance conditions are satisfied.

If:

```text
C != R
```

publication of the reconciled checkpoint requires CAS against the exact `R_sha` used to compute `C`.

## 6. Abstract CAS contract

M1.4 assumes an abstract remote operation with compare-and-set semantics:

```text
CAS(expected_remote_sha = R_sha, candidate = C)
```

Possible abstract outcomes:

- `CAS_COMMITTED(new_sha)` — remote accepted `C` from the expected base;
- `CAS_STALE(current_sha?)` — remote changed since `R`;
- `CAS_AMBIGUOUS` — request returned an error/timeout and commit state cannot be inferred;
- `CAS_FAILED` — definite non-stale failure, no successful commit known.

M1.4 does not select the concrete API. A later implementation must use a primitive or verified protocol that satisfies these semantics.

## 7. CAS / stale retry loop

### 7.1 Main loop

Conceptually:

```text
start with fixed B + reconciled_sha
start with immutable L
attempt_budget = bounded

LOOP:
    observe/fetch latest remote R + R_sha
    observed_sha = R_sha

    C = reconcile(B, R, L)

    if conflict:
        return CONFLICT

    if C semantically equals R:
        accept current R as reconciled
        advance B + reconciled_sha together
        return RECONCILED_NO_WRITE

    result = CAS(expected=R_sha, candidate=C)

    if COMMITTED:
        confirm resulting remote state
        advance observed_sha as observed
        advance B + reconciled_sha only after confirmation
        return RECONCILED_COMMITTED

    if STALE:
        discard C completely
        consume retry budget
        refresh latest remote
        recompute from B + new R + same immutable L
        continue

    if AMBIGUOUS:
        enter UNKNOWN_COMMIT recovery (§9)

    if definite failure:
        return REMOTE_WRITE_FAILED
```

### 7.2 Mandatory stale behavior

A stale CAS invalidates the candidate.

Forbidden behavior:

```text
CAS(old_sha, C1) -> STALE(new_sha)
CAS(new_sha, C1) -> retry old candidate
```

Required behavior:

```text
CAS(old_sha, C1) -> STALE
fetch R2
C2 = reconcile(B, R2, L)
validate C2
CAS(R2_sha, C2) only if still required
```

The stale remote revision may contain compatible edits, convergent edits, or new conflicts; only recomputation can determine which.

`B` remains the last reconciled base throughout these stale retries. `reconciled_sha` does not move merely because several newer remote revisions were observed.

## 8. Bounded contention and liveness

Remote CAS retries must have an explicit, testable attempt/time budget in the later implementation.

M1.4 does not fix the numeric value.

The retry budget must count repeated remote stale/CAS-contention cycles. The implementation may also count selected transient unknown/error recovery attempts, but it must document that behavior.

On exhaustion return:

```text
REMOTE_CONTENTION_EXHAUSTED
```

Requirements:

- no infinite retry loop;
- no silent last-writer-wins fallback;
- no forced overwrite;
- preserve the last `observed_sha` actually observed;
- do not advance `reconciled_sha` unless a valid reconciliation already completed before exhaustion.

A later fresh operation may start again from the still-valid `B/reconciled_sha` and newly observed remote state.

## 9. Unknown / ambiguous commit handling

A timeout, transport failure, or remote error may leave it unknown whether the CAS committed.

The old candidate must not be blindly retried.

### 9.1 Recovery sequence

On `CAS_AMBIGUOUS`:

1. mark the operation transiently as `COMMIT_UNCERTAIN`;
2. do **not** advance `reconciled_sha`;
3. re-observe/fetch the current remote checkpoint;
4. update `observed_sha` only according to the normal observation rule;
5. compare/reconcile current remote state against `B` and immutable `L` again.

Possible outcomes:

#### A. Current remote exactly matches the validated candidate state

If the fetched current checkpoint is confirmed equivalent to the candidate and all invariants/provenance requirements for accepting that state hold, the system may return:

```text
RECONCILED_AFTER_UNKNOWN_COMMIT
```

It need not prove which writer produced the state; it must prove that the current remote state is the accepted reconciled result.

`B/reconciled_sha` may then advance to the actually observed current revision.

#### B. Current remote differs but is merge-compatible

Discard the old candidate and recompute a new candidate from:

```text
B + current R + same immutable L
```

Retry only within the bounded contention budget.

#### C. Current remote now conflicts

Return structured conflict. Do not advance `reconciled_sha`.

#### D. Current remote cannot be re-observed

Return:

```text
REMOTE_COMMIT_STATE_UNKNOWN
```

Do not claim success/failure. Do not advance `reconciled_sha` or fabricate a newer `observed_sha` that was not actually observed.

### 9.2 No blind rollback

M1.4 never rolls the remote checkpoint back to the pre-attempt snapshot merely because a commit became ambiguous. Another writer may have legitimately advanced it.

Recovery is always based on freshly observed remote state.

## 10. SHA/state marker advancement rules

### 10.1 `observed_sha` advancement

`observed_sha` may advance when a trusted remote probe/read successfully reports the current remote revision identity.

Rules:

- a SHA learned from an actual remote observation may become `observed_sha` even before full content is loaded;
- if content for that newer SHA has not been fetched, the old `R` becomes unusable for reconciliation;
- locally predicted/desired SHA values never become `observed_sha`;
- a locally constructed candidate hash is not a remote observation;
- CAS success may provide a new revision identity, but if the platform semantics do not make that identity sufficiently authoritative, the implementation must re-observe before treating it as `observed_sha`.

`observed_sha` carries no reconciliation guarantee.

### 10.2 `reconciled_sha` advancement — no-write case

If deterministic reconciliation produces `C == R`, `reconciled_sha` may advance to `R_sha` only when:

1. `R` was fetched/normalized at `R_sha`;
2. identity/schema checks pass;
3. all field classifications are compatible;
4. whole-checkpoint validation passes;
5. the reconciliation result is accepted as the new shared base by the caller/state machine;
6. `B` is advanced atomically/logically to exactly that same semantic checkpoint.

Then:

```text
B := R
reconciled_sha := R_sha
```

### 10.3 `reconciled_sha` advancement — committed case

After CAS reports success, `reconciled_sha` may advance only when the resulting remote revision is confirmed to contain the accepted candidate state.

Then:

```text
B := confirmed remote candidate state
reconciled_sha := confirmed remote sha
```

If confirmation is ambiguous, use §9; do not advance early.

### 10.4 `reconciled_sha` advancement — unknown-commit recovered case

After ambiguous commit recovery, `reconciled_sha` may advance only to the revision that is actually re-observed and deterministically accepted as reconciled under §9.

Never advance it to the SHA that the original request merely intended to create.

### 10.5 States that must not advance `reconciled_sha`

No advancement on:

- metadata-only discovery of a newer SHA;
- remote fetch before semantic classification;
- `FIELD_CONFLICT`;
- `IDENTITY_CONFLICT`;
- `SCHEMA_INCOMPATIBLE`;
- `UNMERGEABLE_FIELD_POLICY`;
- `CANDIDATE_INVALID`;
- `CAS_STALE`;
- `REMOTE_CONTENTION_EXHAUSTED`;
- `CAS_FAILED`;
- unresolved `COMMIT_UNCERTAIN`;
- `REMOTE_COMMIT_STATE_UNKNOWN`.

## 11. Base-loss and bootstrap safety

Three-way reconciliation requires a trustworthy base.

If `reconciled_sha` is absent/unknown while an existing remote checkpoint already exists, M1.4 must not invent `B` by treating the newest remote state as automatically reconciled.

Return a state such as:

```text
RECONCILIATION_BASE_REQUIRED
```

and hand control to the later recovery/hydration/bootstrap policy (M1.7/M3).

An explicitly initialized `ABSENT` base may be supported for first creation if the later schema/adapter can prove that no prior remote checkpoint exists and uses a create-if-absent/CAS-equivalent primitive.

## 12. Conflict result contract

Conflict results must be structured enough for an agent/user to reason about them without guessing.

Minimum outcome classes:

- `FIELD_CONFLICT`
- `IDENTITY_CONFLICT`
- `SCHEMA_INCOMPATIBLE`
- `UNMERGEABLE_FIELD_POLICY`
- `CANDIDATE_INVALID`
- `RECONCILIATION_BASE_REQUIRED`

A field conflict should identify at least:

- canonical semantic field path/key;
- classification (`FIELD_CONFLICT`);
- base identity (`reconciled_sha`);
- remote identity (`R_sha`);
- safe summaries or hashes/identities of B/R/L field states;
- provenance references where permitted.

M1.4 does not require dumping sensitive field contents into logs/results.

Conflict resolution is external to the automatic merge. A later explicit resolution produces a new local proposal `L2` and starts a new reconciliation operation.

## 13. State machine

Conceptual states:

```text
S0 BASE_READY
   B + reconciled_sha known
        |
        v
S1 REMOTE_OBSERVED
   observed_sha known
        |
        v
S2 REMOTE_LOADED
   R at R_sha == current loaded revision
        |
        v
S3 CLASSIFIED
   field matrix + schema/identity checks
    |                |
 conflict          compatible
    |                v
    |            S4 CANDIDATE_READY
    |             C from B/R/L
    |                |
    |        +-------+--------+
    |        |                |
    |      C==R             C!=R
    |        |                |
    |        v                v
    |   S5 ACCEPT_R      S6 CAS_ATTEMPT
    |        |          /      |       \
    |        |      success   stale   ambiguous
    |        |         |        |        |
    |        |         v        |        v
    |        |     S7 CONFIRM    |   S8 REOBSERVE_UNKNOWN
    |        |         |        |        |
    |        +-----> ADVANCE     |        +--> reclassify from B/Rnew/L
    |          B + reconciled    |
    |                            +--> discard C, fetch Rnew, recompute
    v
STOP_CONFLICT
```

`observed_sha` may move during S1/S2/retry/re-observation. `reconciled_sha` moves only at a valid ADVANCE transition.

## 14. Structured operation outcomes

Recommended architecture-level result vocabulary:

Success/accepted:

- `RECONCILED_NO_WRITE`
- `RECONCILED_COMMITTED`
- `RECONCILED_AFTER_UNKNOWN_COMMIT`

Retry/termination:

- `REMOTE_STALE_RETRY`
- `REMOTE_CONTENTION_EXHAUSTED`
- `REMOTE_WRITE_FAILED`
- `REMOTE_COMMIT_STATE_UNKNOWN`

Conflict/invalid:

- `FIELD_CONFLICT`
- `IDENTITY_CONFLICT`
- `SCHEMA_INCOMPATIBLE`
- `UNMERGEABLE_FIELD_POLICY`
- `CANDIDATE_INVALID`
- `RECONCILIATION_BASE_REQUIRED`

A later executable result may include more detail, but it must not collapse materially distinct states above into an undifferentiated boolean success/failure.

## 15. Relationship to local M1.3 concurrency

`ADR-0002` and M1.4 protect different races.

Local M1.3:

```text
same workspace filesystem
exact local content hash
short local commit lock
atomic local replace/install
```

Remote M1.4:

```text
shared checkpoint
B/R/L semantic reconciliation
remote revision identity
CAS/stale retry
```

Neither substitutes for the other. Local safe-write behavior does not make a stale remote candidate safe, and remote CAS does not protect local `.ai/` files from same-workspace lost updates.

## 16. Review decisions requested for M1.4

Final approval should explicitly approve or revise these decisions:

- **RR1:** `B` is exactly the last fully reconciled shared checkpoint and is inseparable from `reconciled_sha`.
- **RR2:** `observed_sha` means remote observation only and may advance independently of `reconciled_sha`.
- **RR3:** `L` is an immutable normalized shared-schema proposal for one reconciliation operation, not a raw local-memory file tree.
- **RR4:** every candidate `C` is bound to `(reconciled_sha, R_sha, L identity)` and becomes invalid if any dependency changes.
- **RR5:** field reconciliation uses the deterministic three-way matrix in §4; concurrent divergent same-field edits fail closed.
- **RR6:** candidate assembly must pass whole-checkpoint validation after per-field classification.
- **RR7:** `C == R` permits a no-write reconciliation path, but `reconciled_sha` advances only after full semantic acceptance.
- **RR8:** any remote write uses CAS against the exact `R_sha` used to compute the candidate.
- **RR9:** `CAS_STALE` always discards the old candidate and forces remote refresh plus recomputation from `B + Rnew + same L`.
- **RR10:** remote stale/contention retries are bounded and exhaustion returns `REMOTE_CONTENTION_EXHAUSTED` without force overwrite.
- **RR11:** ambiguous commits are re-observed; old candidates are not blindly retried, and unresolved ambiguity returns `REMOTE_COMMIT_STATE_UNKNOWN`.
- **RR12:** `reconciled_sha` and `B` advance only together on a confirmed/accepted reconciled remote state; merely observing a SHA never advances reconciliation.
- **RR13:** missing trustworthy base against an existing remote checkpoint returns `RECONCILIATION_BASE_REQUIRED`; newest remote is not auto-adopted as base.
- **RR14:** M1.4 remains semantic/protocol-only and does not select Hub adapter code, APIs, publication triggers, migration, or privacy implementation.

## 17. Deferred implementation choices

Deferred deliberately:

- exact remote API and CAS primitive;
- exact sync metadata file/path/schema;
- whether candidate/local projection gets a SHA, UUID, or other operation identity;
- numeric retry/backoff/time budgets;
- serialization/canonicalization format;
- exact schema version upgrade negotiation;
- privacy detection/redaction tooling;
- publication trigger policy;
- web/new-device bootstrap UI/workflow;
- migration and release behavior.

## 18. Evidence

- `docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md`
- `docs/decisions/ADR-0002-local-multi-session-safe-write.md`
- `docs/concurrency.md`
- `tests/results/v0.1.0-static-regression.md`

This proposal should be reviewed strictly as M1.4 remote reconciliation semantics. Approval must not be interpreted as executable or live validation.