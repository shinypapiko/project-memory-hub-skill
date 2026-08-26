---
document_role: architecture-proposal
status: review-required
normative: false
architecture_state: proposed-unreleased
runtime_load_policy: maintenance-only
milestone: M1.3
created: 2026-08-26
last_reviewed: 2026-08-26
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
---

# Local multi-session safe-write proposal

## 0. Purpose and scope

This proposal defines M1.3: how several Codex/agent sessions may safely write the same workspace-local `.ai/` memory without silently losing another session's update.

It addresses **same-workspace local filesystem concurrency**. It does not define remote Hub B/R/L reconciliation, cross-device synchronization, Hub publication triggers, or the final Hub adapter implementation.

This is not released behavior. The currently deployed local Project Memory implementation remains unchanged until a later implementation/release step.

## 1. Problem statement

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

B can still overwrite A even though both performed a pre-write refresh. This is a classic time-of-check/time-of-use race.

The local design therefore needs both:

1. **optimistic stale-state detection** using exact content identities; and
2. a **very short commit lock** protecting the final compare-and-write window.

The lock must not be held for the duration of an agent task or while the model performs semantic reconciliation.

## 2. Design goals

The design must:

- prevent blind overwrite of a newer local memory file;
- work even when the workspace is not a Git worktree;
- keep ordinary agent work lock-free;
- keep the critical lock window short;
- force stale sessions to re-read and rebase their intended semantic change;
- use atomic file replacement for committed single-file writes;
- prevent duplicate DEC/EXP IDs under concurrent allocation;
- define safe behavior for derived indexes;
- fail closed on uncertain lock ownership or unsupported filesystem guarantees;
- preserve unrelated workspace instructions and evidence/history;
- provide enough before/after identity for validation and debugging.

The design does **not** need to provide a distributed lock across devices. Remote/cross-device concurrency belongs to the Hub layer.

## 3. Core model: optimistic work, serialized commit

Each mutable local resource uses an exact byte-content identity, normally SHA-256.

A session that reads a mutable file records an ephemeral base token:

```text
base_token = SHA256(exact bytes read)
```

For a file that does not yet exist:

```text
base_token = ABSENT
```

The base token belongs to the session's working context/tool state. It is not itself project knowledge and does not need to be written into `.ai/` after every read.

The session performs normal reasoning and drafting without holding a filesystem lock.

At checkpoint/write time, it submits:

```text
resource
expected_base_token
intended semantic change / replacement candidate
```

The commit path then performs a short compare-and-write critical section.

## 4. Commit protocol

### 4.1 Single-resource commit

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
acquire short per-resource commit lock
  ↓
re-read exact current bytes INSIDE lock
  ↓
current_hash == expected_base_hash ?
  ├─ yes → validate candidate → atomic replace → return new_hash
  └─ no  → release lock → return STALE_LOCAL with latest_hash
```

A stale session must never overwrite merely because its candidate is internally valid.

### 4.2 Reconciliation after stale detection

Semantic reconciliation occurs **outside the lock**:

1. read the latest resource;
2. compare the session's original intent against the intervening change;
3. preserve compatible new state;
4. resolve or surface semantic conflict;
5. produce a rebased intended change;
6. set the latest content hash as the new expected base;
7. retry the commit protocol.

If another session writes again before the retry commits, the retry may become stale again. The correct behavior is to repeat or surface contention—not to bypass the guard.

### 4.3 Why the model must not reason while locked

Holding a filesystem lock while an LLM reads context, reasons, calls tools, or asks the user would create long and unpredictable lock times and increase deadlock/stale-lock risk.

Therefore the lock protects only:

```text
fresh read → expected-token comparison → local validation → atomic write
```

not semantic reconciliation.

## 5. Filesystem lock primitive

The target implementation should use a filesystem operation with atomic exclusive creation semantics, such as an atomic lock-directory creation or equivalent proven primitive.

Conceptual path:

```text
.ai/.locks/<canonical-resource-key>.lock/
```

The lock exists only for the commit critical section.

Lock metadata should include at least:

- random ownership token;
- hostname/device identity when available;
- process ID when available;
- creation timestamp;
- tool/version identifier.

A release operation must verify the ownership token before removing the lock.

### 5.1 Stale locks

A lock must not be stolen merely because it is old.

Automatic recovery is allowed only when the implementation can establish sufficiently strong evidence that the owning process is no longer active and the filesystem semantics are understood. Otherwise the operation fails closed and reports a stale/ambiguous lock for explicit recovery.

Later implementation may define platform-specific safe recovery rules. M1.3 does not require aggressive lock stealing.

### 5.2 Filesystem support

The implementation must not claim safe concurrency on a filesystem unless the required exclusive-create and atomic-replace semantics are supported and tested.

Local NTFS-style semantics are a target environment. Unknown/network/synchronized filesystems may require validation or fail-closed behavior.

## 6. Atomic write primitive

A successful single-file commit should:

1. create/write a temporary file in the **same directory** as the target;
2. flush/close it as required by the platform;
3. validate its complete contents before replacement;
4. replace the target using an atomic rename/replace primitive supported by the filesystem;
5. compute/return the new exact content hash.

Do not update a shared Markdown file by truncating it in place and then writing new contents.

For creation of a previously absent authoritative record, use exclusive creation or another primitive that cannot silently replace an existing file created by another session.

## 7. Resource classes and merge policy

Not every `.ai/` file should have the same automatic merge policy.

### 7.1 Hot mutable state: `CURRENT.md`, `PROJECT.md`, `TASKS.md`

These are semantic state files.

Default stale behavior:

- reject the stale write;
- force fresh read/rebase;
- do not perform an unconstrained text/line merge automatically.

A later executable implementation may support proven field-level merges for stable structured fields, but absence of such a merger must never weaken no-lost-update protection.

### 7.2 Local indexes

`INDEX.md` and subordinate indexes are routing/derived structures.

Where an index entry is derived from an already-created DEC/EXP/session/research record, the source record is primary and the index is repairable.

Concurrent index updates still use the commit protocol. A stale index write is rebased by re-reading the newest index and applying only the intended add/remove operation.

A full stale index snapshot must not overwrite newly added entries.

### 7.3 DEC/EXP record creation

Sequential ID allocation such as `DEC-xxxx` or `EXP-xxxx` has a separate race:

```text
A scans → next ID 8
B scans → next ID 8
```

The future implementation must allocate/create these IDs under a short namespace lock or equivalent atomic reservation.

Conceptually:

```text
lock decision-id namespace
  ↓
rescan latest IDs
  ↓
choose next ID
  ↓
exclusive-create record
  ↓
release namespace lock
```

The corresponding index is updated through its own guarded write.

If the source record is created but index update later fails or the process crashes, validation should detect the unindexed source record and allow deterministic repair. The source record must not be deleted merely to make the index look consistent.

### 7.4 Session/handoff/research records

Where possible, new append-oriented records should use collision-resistant IDs/names that avoid a global sequential counter. Existing schema compatibility still governs actual naming until migration.

Updates to shared local indexes or lifecycle state remain guarded.

## 8. Multi-resource operations

This proposal guarantees **no blind lost update per guarded resource**. It does not claim that an arbitrary set of Markdown files can be committed as one filesystem-atomic transaction.

For an operation touching several resources:

- identify all resources that truly require guarded mutation;
- acquire any simultaneously required locks in a deterministic canonical-path order to avoid deadlock;
- keep the multi-lock critical section minimal;
- prefer writing primary evidence/source records before derived indexes;
- make repairable derived state detectable by validation;
- never report a multi-file operation as fully atomic unless an actual transaction/journal mechanism is later implemented and tested.

If model reasoning is required after detecting that any resource is stale, release all commit locks before reconciliation.

## 9. Local three-state terminology

To keep local and remote concepts distinct, local stale-write reasoning uses:

```text
B_local = exact content read by this session / its expected hash
R_local = latest local file content at commit attempt
I_local = session's intended semantic change
```

This is **not** the Hub shared-checkpoint B/R/L merge from ADR-0001.

For M1.3 the conservative rule is:

```text
if hash(R_local) != hash(B_local):
    reject commit as STALE_LOCAL
    reconcile intent outside lock
    retry against new base
```

The local helper does not need to automatically understand arbitrary Markdown semantics to prevent lost updates.

## 10. Conflict/result contract

The executable write helper should eventually return a structured result rather than only free-form text.

Minimum outcomes:

- `COMMITTED`
- `STALE_LOCAL`
- `LOCK_BUSY`
- `LOCK_AMBIGUOUS`
- `VALIDATION_FAILED`
- `UNSUPPORTED_FILESYSTEM`
- `CREATE_COLLISION`
- `IO_ERROR`

A successful result should include:

- resource path/key;
- previous hash;
- new hash;
- operation type;
- optional transaction/attempt ID.

A stale result should include the current observed hash but must not expose unrelated file contents unless requested by the caller.

## 11. Git relationship

Git may provide provenance and history where a workspace is a Git worktree, but Git must **not** be the required local concurrency primitive.

Reasons:

- some real target workspaces are not Git worktrees;
- local `.ai/` may be excluded from Git;
- several sessions may modify files before a commit exists;
- Git HEAD does not by itself provide a compare-and-swap guarantee for an individual local file.

The safe-write layer therefore uses filesystem content identity + short commit locking. Git integration remains optional provenance/backup behavior.

## 12. Startup/read implications

A session should track base hashes only for mutable resources it actually reads and may later write.

Normal LOAD does not need to hash every historical file recursively.

Typical startup may record hashes for:

- `.ai/PROJECT.md`;
- `.ai/CURRENT.md`;
- `.ai/INDEX.md`;
- other files loaded for the active task.

Selective retrieval adds base hashes for additional mutable records only when they are read for possible mutation.

## 13. Invariants

The local safe-write implementation is unacceptable if any of these can occur silently:

1. two sessions both pass a pre-write check and one overwrites the other's newer update;
2. a stale whole-file snapshot replaces a newer `CURRENT`, `TASKS`, `PROJECT`, or index;
3. two sessions allocate the same DEC/EXP ID and one replaces the other's record;
4. the implementation holds a commit lock while waiting for LLM reasoning/user interaction;
5. lock recovery deletes a lock with uncertain ownership merely because it is old;
6. a source DEC/EXP record is deleted because a derived index update failed;
7. the system claims multi-file atomicity that it does not actually provide;
8. safe operation requires the workspace to be a Git repository;
9. unsupported filesystem semantics are treated as safe without validation.

## 14. Proposed acceptance decisions

M1.3 review should explicitly approve or revise:

- **L1:** mandatory expected-base content hash for guarded local writes;
- **L2:** short per-resource commit lock around fresh-read/compare/atomic-write only;
- **L3:** semantic reconciliation occurs outside locks and stale writes must retry against a new base;
- **L4:** no automatic unconstrained line/text merge for stale hot-state Markdown;
- **L5:** atomic same-directory replace for committed existing files and exclusive creation for new source records;
- **L6:** DEC/EXP sequential ID allocation requires atomic namespace reservation/locking;
- **L7:** indexes are guarded but repairable derivatives; source records are not deleted to repair an index;
- **L8:** deterministic lock ordering for operations that truly need multiple simultaneous locks;
- **L9:** no claim of arbitrary multi-file atomic transactions in the initial design;
- **L10:** filesystem hash/lock semantics, not Git HEAD, are the required local concurrency primitive;
- **L11:** ambiguous/stale lock ownership fails closed unless a safe platform-specific recovery rule proves reclaimability;
- **L12:** unsupported/unknown filesystem atomicity fails closed rather than silently degrading safety.

## 15. Deferred implementation choices

This proposal intentionally does not yet fix:

- exact Python API/CLI command names;
- lock filename encoding and hash algorithm implementation constants beyond a SHA-256 default;
- platform-specific stale-lock recovery thresholds;
- whether a future transaction journal is worth adding for stronger multi-file crash consistency;
- richer automatic field-level semantic merging;
- final task-entry identifiers/schema changes;
- Hub remote reconciliation implementation.

Those are implementation/schema/test decisions after M1.3 approval.
