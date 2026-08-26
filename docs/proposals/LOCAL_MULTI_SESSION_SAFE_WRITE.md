---
document_role: architecture-proposal
status: approved-proposal
normative: false
architecture_state: accepted-unreleased
runtime_load_policy: maintenance-only
milestone: M1.3
created: 2026-08-26
last_reviewed: 2026-08-27
review_state: final-approved-promoted-to-adr
review_record: docs/reviews/2026-08-26-local-multi-session-safe-write-review.md
final_review_record: docs/reviews/2026-08-27-local-multi-session-safe-write-final-approval.md
approved_by: docs/decisions/ADR-0002-local-multi-session-safe-write.md
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
---

# Local multi-session safe-write proposal

## 0. Purpose and scope

This proposal defines M1.3: how several Codex/agent sessions may safely write the same workspace-local `.ai/` memory without silently losing another participating session's update.

It addresses **same-workspace local filesystem concurrency**. It does not define remote Hub B/R/L reconciliation, cross-device synchronization, Hub publication triggers, or the final Hub adapter implementation.

This is not released behavior. The currently deployed local Project Memory implementation remains unchanged until a later implementation/release step.

Review state:

- M1.3 final review completed on 2026-08-27;
- L1–L12 approved;
- B1–B5 closed;
- no new blocking defects were identified;
- the accepted semantics were promoted to `ADR-0002`;
- architecture approval does not imply `EXECUTABLE` or `LIVE_VALIDATED` status.

## 1. Safety boundary

### 1.1 Cooperative writers

The no-lost-update guarantee applies to **cooperative writers**: processes/agents that mutate guarded `.ai/` resources through the same released safe-write helper/protocol and therefore obey the same lock identity, compare-and-write, and atomic-install rules.

An external editor, script, sync client, or older Skill that writes directly to the target file without the helper is outside the lock-coordination guarantee.

The helper should still detect many such external writes because the exact target bytes/hash will differ from the expected base, but it must not claim mutual exclusion against a writer that ignores the lock protocol.

The implementation/documentation must therefore distinguish:

```text
cooperative writer safety
= lock coordination + stale-state detection + atomic commit

non-cooperative external mutation
= best-effort stale-state detection only; no mutual-exclusion guarantee
```

### 1.2 Problem statement

A mandatory pre-write re-read is necessary but not sufficient by itself.

Without a commit-time concurrency primitive, two sessions can both do this:

```text
A reads latest X
B reads latest X
A decides it is safe to write
B decides it is safe to write
A writes X+A
B writes X+B
```

B can still overwrite A even though both performed a pre-write refresh. This is a time-of-check/time-of-use race.

The local design therefore needs:

1. **optimistic stale-state detection** using exact content identities;
2. a **canonical resource identity** shared by all cooperative writers;
3. a **very short commit lock** protecting the final compare-and-write window; and
4. atomic replace/install primitives whose failure state can be classified conservatively.

The lock must not be held for the duration of an agent task or while the model performs semantic reconciliation.

## 2. Design goals

The design must:

- prevent blind overwrite of a newer local memory file by cooperative writers;
- work even when the workspace is not a Git worktree;
- keep ordinary agent work lock-free;
- keep the critical lock window short;
- force stale sessions to re-read and rebase their intended semantic change;
- use atomic file replacement for committed existing-file writes;
- use atomic no-replace installation for new authoritative records;
- prevent duplicate DEC/EXP IDs under concurrent allocation;
- distinguish reconstructible indexes from curated routing content;
- make retry/contention behavior bounded;
- define deterministic release-on-lock-acquisition-failure behavior;
- expose partial/ambiguous commit states structurally rather than hiding them;
- fail closed on uncertain lock ownership, ambiguous resource identity, or unsupported filesystem guarantees;
- preserve unrelated workspace instructions and evidence/history;
- provide enough before/after identity for validation and debugging.

The design does **not** provide a distributed lock across devices. Remote/cross-device concurrency belongs to the Hub layer.

## 3. Canonical resource identity

### 3.1 Requirement

All cooperative writers targeting the same semantic filesystem resource must derive the same **canonical resource identity** and therefore the same lock key.

A plain user-supplied path string is not sufficient because two different strings may identify the same object.

Conceptually:

```text
resource_path
  ↓ normalize/resolve under verified workspace root
canonical_resource_identity
  ↓ stable encoding/hash
canonical_lock_key
```

The exact implementation is deferred, but approval of M1.3 requires these semantics:

- relative paths are resolved against one verified workspace/local-memory root;
- `.` / `..` and ordinary normalization cannot produce two lock identities for the same resolved path;
- Windows case-insensitive aliases must not create separate lock keys;
- a known 8.3/long-name alias pair must not be treated as safely distinct;
- hard-link ambiguity, reparse points, junctions, symlinks, or mount aliases must either be resolved to one proven identity or fail closed;
- resource resolution must not escape the allowed workspace/local-memory boundary;
- if the helper cannot prove that two path forms are safely identical or safely distinct, it must not claim concurrency safety.

This proposal does not require a specific OS file-ID API, only a unique proven identity for each guarded target in supported environments.

### 3.2 Lock namespace

Conceptually:

```text
.ai/.locks/<canonical-lock-key>.lock/
```

Lock ordering, ownership, result reporting, and debugging all use the canonical identity/key rather than a raw caller path.

## 4. Core model: optimistic work, serialized commit

Each mutable local resource uses an exact byte-content identity, normally SHA-256.

A session that reads a mutable file records an ephemeral base token:

```text
base_token = SHA256(exact bytes read)
```

For a file that does not yet exist:

```text
base_token = ABSENT
```

The base token belongs to the session's working context/tool state. It is not project knowledge and does not need to be persisted after every read.

The session performs normal reasoning and drafting without holding a filesystem lock.

At checkpoint/write time, it submits:

```text
canonical resource identity
expected_base_token
intended semantic change / replacement candidate
```

The commit path then performs a short compare-and-write critical section.

## 5. Commit protocol

### 5.1 Single-resource existing-file commit

For a mutable file such as `.ai/CURRENT.md`:

```text
session read
  ↓
record expected_base_hash
  ↓
work/reason without lock
  ↓
prepare intended change
  ↓
resolve canonical resource identity
  ↓
acquire short per-resource commit lock
  ↓
re-read exact current bytes INSIDE lock
  ↓
current_hash == expected_base_hash ?
  ├─ yes → validate candidate → atomic replace → verify/classify result → return
  └─ no  → release lock → return STALE_LOCAL with latest_hash
```

A stale session must never overwrite merely because its candidate is internally valid.

### 5.2 Semantic reconciliation after stale detection

Semantic reconciliation occurs **outside all commit locks**:

1. read the latest resource;
2. compare the original intent against the intervening change;
3. preserve compatible new state;
4. resolve or surface semantic conflict;
5. produce a rebased intended change;
6. set the latest content hash as the new expected base;
7. retry the commit protocol if policy permits.

No arbitrary Markdown line merge is implied.

### 5.3 Bounded retry and liveness

Automatic retry must be bounded by an implementation-defined attempt/time budget that is explicit and testable.

If repeated stale writes or lock contention exhaust that budget, return a structured result such as:

```text
CONTENTION_EXHAUSTED
```

The helper must not spin indefinitely or hold a lock while waiting for semantic reasoning, user input, or backoff.

Backoff/retry occurs outside all locks.

### 5.4 Why the model must not reason while locked

Holding a filesystem lock while an LLM reads context, reasons, calls tools, or asks the user would create long and unpredictable lock times and increase deadlock/stale-lock risk.

Therefore the lock protects only a short critical section such as:

```text
fresh read
→ expected-token comparison
→ local candidate validation
→ atomic replace/install
→ result verification/classification
```

## 6. Filesystem lock primitive

The target implementation should use an operation with atomic exclusive creation semantics, such as atomic lock-directory creation or an equivalent proven primitive.

Lock metadata should include at least:

- canonical resource identity/key;
- random ownership token;
- hostname/device identity when available;
- process ID when available;
- creation timestamp;
- tool/version identifier.

A release operation must verify the ownership token before removing the lock.

### 6.1 Stale locks

A lock must not be stolen merely because it is old.

Automatic recovery is allowed only when the implementation can establish sufficiently strong evidence that the owning process is no longer active and the filesystem semantics are understood. Otherwise the operation fails closed and reports `LOCK_AMBIGUOUS` or equivalent for explicit recovery.

### 6.2 Filesystem support

The implementation must not claim safe concurrency on a filesystem unless the required exclusive-create, canonical-identity, and atomic-replace/install semantics are supported and tested.

Local NTFS-style semantics are a target environment. Unknown/network/synchronized filesystems may require validation or fail-closed behavior.

## 7. Atomic write/install primitives

### 7.1 Existing target: atomic replace

A successful single-file replacement should:

1. create/write a temporary file in the **same directory** as the target;
2. flush/close it as required by the platform;
3. validate its complete contents before replacement;
4. replace the target using a proven atomic rename/replace primitive;
5. re-observe/classify the target state if the platform/API reports failure or ambiguity;
6. compute/return the new exact content hash only when commit state is known.

Do not update a shared Markdown file by truncating it in place and then writing new contents.

If the atomic replace primitive fails, the helper must **not** fall back to a non-atomic in-place overwrite.

### 7.2 Ambiguous replace result

After a replace/install error, the helper must re-read/re-stat the actual target and compare against known before/candidate identities where possible.

Possible classifications include:

- definitely not committed;
- definitely committed with expected candidate hash;
- target changed to some other known/unknown state;
- commit state cannot be proven.

If the helper cannot determine whether the new candidate became authoritative, return:

```text
COMMIT_STATE_UNKNOWN
```

or an equivalent hard-error state. The caller must re-read/recover from observed state rather than blindly retrying the same overwrite.

### 7.3 New authoritative record: atomic no-replace install

For a new DEC/EXP/source record:

1. prepare the complete contents in a temporary same-directory file or other private staging object;
2. fully flush/close and validate the complete record before it becomes authoritative;
3. install it with an atomic **no-replace** primitive or equivalent that fails if the final target already exists;
4. verify/classify the final target state on ambiguous failure;
5. never expose an intentionally partial record at the authoritative target name.

`CREATE_COLLISION` is distinct from I/O failure and from `COMMIT_STATE_UNKNOWN`.

A crash may leave a private temporary/staging artifact, but that artifact must be distinguishable from a valid authoritative record and safely recoverable/cleanable.

## 8. Resource classes and merge/repair policy

### 8.1 Hot mutable state: `CURRENT.md`, `PROJECT.md`, `TASKS.md`

These are semantic state files.

Default stale behavior:

- reject the stale write;
- force fresh read/rebase;
- do not perform an unconstrained text/line merge automatically.

A future executable implementation may support proven field-level merges for stable structured fields, but absence of such a merger must never weaken no-lost-update protection.

### 8.2 Indexes: reconstructible vs curated content

`INDEX.md` and subordinate indexes must not be treated as uniformly disposable/repairable.

Each index portion/entry should be classified conceptually as one of:

1. **RECONSTRUCTIBLE** — deterministically derivable from authoritative source records, e.g. an entry that can be recreated by scanning existing DEC/EXP records;
2. **CURATED** — human/agent-maintained routing, grouping, annotations, ordering, scope notes, aliases, or other information not safely reconstructible from source-file existence alone.

Rules:

- both classes still use guarded writes;
- stale full-index snapshots never overwrite newer entries;
- reconstructible entries may be repaired/rebuilt from authoritative source records;
- curated content is semantic state and must be preserved/rebased like other mutable state;
- a repair operation must not discard curated content merely because some entries are reconstructible;
- the schema/implementation should make reconstruction boundaries explicit enough for deterministic validation/repair.

### 8.3 DEC/EXP sequential record creation

Sequential ID allocation such as `DEC-xxxx` or `EXP-xxxx` has a separate race:

```text
A scans → next ID 8
B scans → next ID 8
```

The future implementation must allocate/reserve these IDs under a short namespace lock or equivalent atomic reservation.

Conceptually:

```text
acquire namespace lock
  ↓
rescan latest authoritative IDs
  ↓
choose next ID
  ↓
prepare + validate complete record
  ↓
atomic no-replace install final record
  ↓
release namespace lock
```

The corresponding index is updated through its own guarded write.

If the source record is committed but the index update later fails or the process crashes, validation should detect the missing reconstructible index entry and allow deterministic repair. The source record must not be deleted merely to make the index appear consistent.

### 8.4 Session/handoff/research records

Where possible, new append-oriented records should use collision-resistant IDs/names that avoid a global sequential counter. Existing schema compatibility still governs actual naming until migration.

Updates to shared local indexes or lifecycle state remain guarded.

## 9. Multi-resource operations

This proposal guarantees **no blind lost update per guarded resource among cooperative writers**. It does not claim that an arbitrary set of Markdown files can be committed as one filesystem-atomic transaction.

### 9.1 Lock acquisition

For an operation that truly requires simultaneous locks:

1. resolve every target to its canonical resource identity;
2. reject duplicate/ambiguous aliases;
3. sort the unique canonical lock keys in one deterministic total order;
4. acquire locks only in that order;
5. if acquisition of any later lock fails, release **all** locks already acquired by this attempt;
6. back off/retry outside all locks, subject to the bounded contention budget.

No semantic reasoning or user interaction occurs while a partial lock set is held.

### 9.2 Commit ordering

Prefer committing primary evidence/source records before repairable derived/index state.

This ordering does not create full transactional atomicity; it makes crash recovery more deterministic because a committed primary source can be discovered and a reconstructible derivative repaired later.

### 9.3 Partial completion contract

A multi-resource operation must report per-resource results.

If at least one resource commits and at least one required later resource does not, the overall operation returns:

```text
PARTIAL_COMMIT
```

with, at minimum:

- operation/attempt ID;
- ordered resource list;
- per-resource canonical identity;
- per-resource result;
- before hash/state where known;
- after hash/state where known;
- which resources definitely committed;
- which resources definitely did not commit;
- which resources, if any, are `COMMIT_STATE_UNKNOWN`.

### 9.4 No blind rollback

The helper must not automatically roll back already committed resources using stale saved snapshots. Another session may have legitimately changed them after commit.

Recovery is normally **forward repair from freshly observed state**:

- re-read committed resources;
- validate the invariant that should span them;
- complete missing repairable/derived updates if safe;
- surface semantic conflicts for non-repairable state.

A rollback is allowed only if a future explicitly designed reversible transaction mechanism proves that rollback is safe against intervening writes.

## 10. Local stale-state terminology

To keep local and remote concepts distinct, local stale-write reasoning uses:

```text
B_local = exact content read by this session / expected hash
R_local = latest local file content at commit attempt
I_local = session's intended semantic change
```

This is **not** the Hub shared-checkpoint B/R/L merge from ADR-0001.

For M1.3 the conservative rule is:

```text
if hash(R_local) != hash(B_local):
    reject commit as STALE_LOCAL
    reconcile intent outside lock
    retry against new base within bounded retry policy
```

The local helper does not need to understand arbitrary Markdown semantics to prevent blind lost updates.

## 11. Structured result contract

The executable write helper should return structured outcomes rather than only free-form text.

Minimum single-resource outcomes:

- `COMMITTED`
- `STALE_LOCAL`
- `LOCK_BUSY`
- `LOCK_AMBIGUOUS`
- `CONTENTION_EXHAUSTED`
- `VALIDATION_FAILED`
- `UNSUPPORTED_FILESYSTEM`
- `RESOURCE_IDENTITY_AMBIGUOUS`
- `CREATE_COLLISION`
- `COMMIT_STATE_UNKNOWN`
- `IO_ERROR`

Multi-resource operations additionally use:

- `PARTIAL_COMMIT`

A successful result should include:

- canonical resource identity/key;
- display path;
- previous hash/state;
- new hash/state;
- operation type;
- attempt/operation ID.

A stale result should include the current observed hash/state but must not expose unrelated file contents unless requested by the caller.

`PARTIAL_COMMIT` must include the per-resource recovery information defined in §9.3.

## 12. Git relationship

Git may provide provenance and history where a workspace is a Git worktree, but Git must **not** be the required local concurrency primitive.

Reasons:

- some target workspaces are not Git worktrees;
- local `.ai/` may be excluded from Git;
- several sessions may modify files before a Git commit exists;
- Git HEAD does not provide compare-and-swap for an individual local file.

The safe-write layer therefore uses filesystem content identity + canonical resource locking + atomic replace/install. Git integration remains optional provenance/backup behavior.

## 13. Startup/read implications

A session should track base hashes only for mutable resources it actually reads and may later write.

Normal LOAD does not need to hash every historical file recursively.

Typical startup may record hashes for:

- `.ai/PROJECT.md`;
- `.ai/CURRENT.md`;
- `.ai/INDEX.md`;
- other files loaded for the active task.

Selective retrieval adds base hashes for additional mutable records only when they are read for possible mutation.

## 14. Invariants

The local safe-write design is unacceptable if any of these can occur silently:

1. two cooperative sessions both pass a pre-write check and one overwrites the other's newer update;
2. two path aliases to the same guarded target can obtain different locks without detection;
3. a stale whole-file snapshot replaces a newer `CURRENT`, `TASKS`, `PROJECT`, or curated index content;
4. two sessions allocate the same DEC/EXP ID and one replaces the other's record;
5. the implementation holds a commit lock while waiting for LLM reasoning/user interaction/backoff;
6. multi-lock acquisition failure can strand a partial lock set;
7. automatic retry can loop indefinitely without surfacing contention exhaustion;
8. lock recovery deletes a lock with uncertain ownership merely because it is old;
9. a new authoritative DEC/EXP target can be left as an unmarked partial file;
10. an atomic replace/install error silently falls back to a non-atomic in-place overwrite;
11. an ambiguous commit result is reported as definitely committed/not committed without verification;
12. a source DEC/EXP record is deleted because a derived index update failed;
13. curated index/routing content is discarded under the assumption that all indexes are reconstructible;
14. the system claims multi-file atomicity that it does not provide;
15. a partial multi-resource operation is reported as ordinary success/failure without per-resource state;
16. automatic rollback overwrites intervening legitimate writes;
17. safe operation requires the workspace to be a Git repository;
18. unsupported/ambiguous filesystem or resource-identity semantics are treated as safe without validation.

## 15. Revised acceptance decisions

Final M1.3 review explicitly accepted:

- **L1:** every guarded local write uses an expected-base exact-byte content hash/token;
- **L2:** cooperative writers derive one canonical resource identity/lock key, and a short per-resource commit lock protects only fresh-read/compare/validate/atomic-write classification; ambiguous path aliases fail closed;
- **L3:** semantic reconciliation and backoff occur outside all locks; automatic stale/contention retries are bounded and exhaustion returns `CONTENTION_EXHAUSTED`;
- **L4:** stale hot-state Markdown is never resolved by unconstrained automatic line/text merge;
- **L5:** existing files use same-directory atomic replace; new authoritative source records use fully validated atomic no-replace installation; ambiguous replace/install failure is re-observed and may return `COMMIT_STATE_UNKNOWN`, with no non-atomic fallback;
- **L6:** DEC/EXP sequential ID allocation requires atomic namespace reservation/locking plus re-scan and collision-safe final installation;
- **L7:** index policy distinguishes `RECONSTRUCTIBLE` entries from `CURATED` semantic routing/content; only reconstructible material may be rebuilt from source records;
- **L8:** simultaneous locks are ordered by canonical lock key; any acquisition failure releases the complete partial lock set before bounded retry outside locks;
- **L9:** the initial design makes no arbitrary multi-file atomicity claim; partial completion returns `PARTIAL_COMMIT` with per-resource state and uses no blind rollback;
- **L10:** filesystem content identity/locking/atomicity, not Git HEAD, is the required local concurrency primitive;
- **L11:** ambiguous/stale lock ownership fails closed unless a safe platform-specific recovery rule proves reclaimability;
- **L12:** unsupported or ambiguous filesystem/resource-identity/atomicity semantics fail closed rather than silently degrading safety.

## 16. Deferred implementation choices

This proposal intentionally does not yet fix:

- exact Python API/CLI command names;
- the exact canonical-path/file-ID API chosen for each supported platform;
- lock filename encoding and implementation constants beyond a SHA-256 content-identity default;
- numeric retry/backoff/time budgets;
- platform-specific stale-lock recovery thresholds;
- exact Windows replace/no-replace APIs, which must satisfy the accepted semantics and executable tests;
- whether a future transaction journal is worth adding for stronger multi-file crash consistency;
- richer automatic field-level semantic merging;
- final task-entry identifiers/schema changes;
- Hub remote reconciliation implementation.

These are implementation/schema/test decisions after M1.3 architecture approval.

## 17. Evidence

- `docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md`
- `docs/reviews/2026-08-26-local-multi-session-safe-write-review.md`
- `docs/reviews/2026-08-27-local-multi-session-safe-write-final-approval.md`
- `docs/decisions/ADR-0002-local-multi-session-safe-write.md`

The accepted architecture is recorded in `ADR-0002`. This proposal remains non-normative and does not establish executable or live-validated behavior.
