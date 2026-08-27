---
document_role: architecture-proposal
status: review-required
normative: false
architecture_state: proposed-unreleased
runtime_load_policy: maintenance-only
milestone: M1.7
created: 2026-08-27
last_reviewed: 2026-08-27
review_state: initial-review-pending
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
  - docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md
  - docs/decisions/ADR-0004-hub-adapter-and-transport-boundary.md
  - docs/decisions/ADR-0005-privacy-never-publish-policy.md
---

# Normal fast path / changed-remote path / recovery path — M1.7

## 0. Purpose and strict scope

This proposal defines deterministic route-selection semantics for startup and synchronization-state handling in the future unified Project Memory architecture.

M1.7 is deliberately limited to exactly three logical paths:

1. **normal fast path**;
2. **changed-remote path**;
3. **recovery path**.

It does not reopen ADR-0001 through ADR-0005. It composes their accepted authority, local-safe-write, remote-reconciliation, transport-boundary, and privacy-gate semantics into one bounded startup/routing model.

M1.7 defines only:

- which trusted inputs are required before route selection;
- when the unchanged fast path is permitted;
- when a changed remote requires an exact fetch plus ADR-0003 reconciliation;
- which binding/base/schema/capability/ambiguity states route to recovery;
- when local-only degraded continuation is permitted;
- what each route may and may not claim or mutate;
- how route selection restarts after recovery evidence changes.

M1.7 does **not** define or implement:

- M1.8 token, memory, payload, compaction, probe, or I/O budgets;
- publication triggers;
- Hub adapter implementation;
- privacy detector/classifier implementation;
- concrete probe API or Git/GitHub API;
- credential handling implementation;
- lock, filesystem, Windows, or transaction primitives;
- retry/backoff numbers;
- concrete sync-metadata file/schema encoding;
- implementation language/module layout;
- migration;
- real project/runtime data operations;
- full UI/operator workflow;
- executable/static/live validation implementation.

No current released Skill, Runtime Hub behavior, local Project Memory runtime, transport tool, or real data is modified by this proposal.

## 1. Preserved accepted semantics

M1.7 inherits these constraints without redefining them.

### 1.1 ADR-0003 remains authoritative for remote reconciliation

The canonical reconciled local pair remains conceptually:

```text
P = (B, reconciled_sha)
```

guarded by `base_pair_token`.

`observed_sha` is remote observation only. It is never automatically promoted to `reconciled_sha`.

Trusted base states remain:

```text
BASE_READY_PRESENT
BASE_READY_ABSENT
BASE_UNINITIALIZED
BASE_INVALID
```

A previously reconciled present checkpoint observed as absent remains a conflict unless a separate accepted deletion/tombstone policy authorizes that transition.

Remote CAS success, exact committed-revision confirmation, and guarded local finalization remain separate stages.

### 1.2 ADR-0004 remains authoritative for transport separation

Transport mailbox/message/receipt/snapshot state is not Hub adapter checkpoint state and is not a recovery base.

Transport cannot be used as an adapter fallback when probe, fetch, reconciliation, binding, schema, capability, or recovery fails.

### 1.3 ADR-0005 remains authoritative before any outbound write

Read-only probe/fetch/recovery observation may occur without publication privacy approval when otherwise authorized.

Any exact outbound candidate/bundle must have a current destination-bound privacy `ALLOW` before entering a write-capable remote CAS/publication attempt.

If candidate, policy/classification, destination, provenance, binding/scope, or bundle context drifts, the old privacy verdict is stale and cannot be reused.

## 2. Route-selection inputs

Route selection must use typed, trusted inputs. Directory names, path similarity, transport state, stale cached observations, or optimistic defaults cannot substitute for missing trust.

The minimum semantic input set is:

```text
BindingState
  VALID
  MISSING
  AMBIGUOUS
  MISMATCH

BaseState
  BASE_READY_PRESENT
  BASE_READY_ABSENT
  BASE_UNINITIALIZED
  BASE_INVALID

CapabilityState
  SUPPORTED
  UNSUPPORTED
  UNKNOWN

SchemaState
  SUPPORTED
  UNSUPPORTED
  UNKNOWN

OperationRecoveryState
  CLEAR
  REMOTE_COMMIT_UNKNOWN
  REMOTE_CONFIRMATION_UNKNOWN
  LOCAL_FINALIZATION_FAILED
  LOCAL_FINALIZATION_UNKNOWN

ProbeState
  UNCHANGED_EXACT
  CHANGED_EXACT
  REMOTE_ABSENT_EXACT
  UNAVAILABLE
  UNKNOWN
  INVALID
```

The exact implementation representation is deferred. The semantic requirement is that the route selector can distinguish these states without collapsing unknown or unsupported values into a success state.

### 2.1 Valid binding

`BindingState=VALID` means the canonical workspace binding and the adapter target identity are present, unambiguous, and consistent under ADR-0004.

A transport `project_id`, `hub_project_path`, channel name, directory similarity, remote URL similarity, or cached path is not sufficient to create `VALID`.

### 2.2 Trusted base

A trusted base means exactly `BASE_READY_PRESENT` or `BASE_READY_ABSENT` under ADR-0003.

`BASE_UNINITIALIZED` is not a weak form of trusted base.

`BASE_INVALID` is not repairable by choosing the newest remote checkpoint, by using a transport snapshot/history item, or by assuming the current remote is correct.

### 2.3 Trusted exact probe

A probe may authorize fast-path routing only when it is a trusted observation against the exact validated adapter checkpoint target and can establish the checkpoint identity/state needed to compare against the canonical reconciled base.

Conceptually:

```text
trusted_probe(target, expected reconciled identity)
    -> exact remote identity/state
```

A cached observation, repository-global HEAD, transport SHA, approximate timestamp, directory state, unavailable result, timeout, or unknown result is not an exact trusted probe.

A trusted exact probe may update `observed_sha` according to ADR-0003 observation semantics. It does not advance `B/reconciled_sha`.

## 3. Deterministic route-selection order

Route selection is evaluated in fail-closed priority order.

```text
1. Validate canonical binding.
2. Validate schema/capability support required for the intended route.
3. Validate canonical base state.
4. Check unresolved remote-commit / confirmation / local-finalization recovery state.
5. Perform or consume a trusted exact probe when eligible.
6. Select exactly one of:
     NORMAL_FAST_PATH
     CHANGED_REMOTE_PATH
     RECOVERY_PATH
```

Earlier failures are not bypassed merely because a later signal appears convenient.

For example, a probe that appears unchanged cannot authorize the fast path when binding is mismatched or the base is invalid.

## 4. Route-selection table

| Condition | Route | Required action | Forbidden interpretation / mutation |
| --- | --- | --- | --- |
| Valid binding + supported schema/capability + `BASE_READY_PRESENT` + no unresolved recovery state + trusted exact probe confirms the same remote revision as `reconciled_sha` | **NORMAL_FAST_PATH** | Record trusted observation as permitted; use trusted local `B`/local memory without full remote checkpoint fetch | Must not re-label observation as new reconciliation; must not write merely because remote is unchanged |
| Valid binding + supported schema/capability + `BASE_READY_ABSENT` + no unresolved recovery state + trusted exact probe confirms remote remains absent | **NORMAL_FAST_PATH** | Continue using trusted absent lineage; no full checkpoint fetch is required | Must not invent present checkpoint content; absence observation does not create new lineage |
| Valid binding + trusted base + trusted exact probe confirms a different present remote revision | **CHANGED_REMOTE_PATH** | Exact fetch of remote checkpoint revision, then ADR-0003 B/R/L reconciliation | Must not skip full fetch; must not copy `observed_sha` into `reconciled_sha` |
| `BASE_READY_ABSENT` + trusted exact probe confirms a present remote checkpoint | **CHANGED_REMOTE_PATH** | Exact fetch and ADR-0003 reconciliation from trusted absent base | Must not auto-adopt newest remote merely because local base was absent |
| Probe unavailable / timeout / unknown / invalid | **RECOVERY_PATH** | Do not claim unchanged; use authorized read-only recovery observation/full fetch if available, then restart route selection from trusted evidence | Must not enter fast path; must not write based on guessed remote state |
| Binding missing | **RECOVERY_PATH** | Require proper binding establishment before adapter synchronization | Must not infer target from path/channel/transport config |
| Binding ambiguous | **RECOVERY_PATH** | Resolve ambiguity through canonical binding authority | Must not choose an arbitrary plausible target |
| Binding mismatch | **RECOVERY_PATH** | Surface typed binding conflict; stop automatic synchronization | Must not rewrite binding/registry to make the route succeed |
| `BASE_UNINITIALIZED` with remote present or absent | **RECOVERY_PATH** | Return base-required state; optionally inspect remote read-only; require explicit trustworthy lineage/bootstrap resolution outside this proposal | Must not adopt newest remote or infer trusted absent base |
| `BASE_INVALID` | **RECOVERY_PATH** | Return base-invalid state; require trustworthy repair/re-establishment outside this proposal | Must not repair from transport snapshot/history, current remote, or guessed SHA |
| Trusted absent lineage + remote remains absent | **NORMAL_FAST_PATH** | Same as `BASE_READY_ABSENT` unchanged case | Must not confuse observation of absence with first-time initialization |
| Previously reconciled present checkpoint -> exact trusted remote absent | **RECOVERY_PATH** | Surface remote-checkpoint-absent conflict under ADR-0003 | Must not treat disappearance as ordinary changed-remote deletion or auto-advance markers |
| Schema unsupported/unknown | **RECOVERY_PATH** | Surface unsupported/unknown schema; permit only safe diagnostic/read-only actions defined by later implementation | Must not coerce, downgrade, or write using guessed schema |
| Capability unsupported/unknown | **RECOVERY_PATH** | Surface unsupported/unknown capability; stop automatic sync/write | Must not fall back to transport or a different untyped operation |
| Remote commit/confirmation state unknown | **RECOVERY_PATH** | Re-observe according to ADR-0003; if exact candidate is confirmed, resume guarded finalization; otherwise recompute/conflict/unknown as ADR-0003 requires | Must not blind-retry, roll back, or claim reconciliation success |
| Local finalization failed/unknown after remote acceptance/write | **RECOVERY_PATH** | Persistent old local base remains authoritative; re-observe remote and restart from ADR-0003-compatible recovery evidence | Must not claim reconciled merely because remote write succeeded |
| Recovery produces a new exact trusted remote observation with trusted base still valid | **RESELECT** | Restart this route-selection table from current trusted inputs | Recovery itself does not become successful synchronization |

`RESELECT` is not a fourth operating path; it is the transition that exits recovery and re-evaluates one of the three defined paths.

## 5. Normal fast path

### 5.1 Preconditions

The fast path is permitted only when all of these are true:

```text
BindingState == VALID
BaseState in {BASE_READY_PRESENT, BASE_READY_ABSENT}
CapabilityState == SUPPORTED
SchemaState == SUPPORTED
OperationRecoveryState == CLEAR
ProbeState proves exact remote state is unchanged
```

For a present base:

```text
probe.remote_identity == reconciled_sha
```

For a trusted absent base:

```text
probe.remote_state == ABSENT
and local base state == BASE_READY_ABSENT
```

If any precondition is unknown, stale, unsupported, mismatched, or invalid, the route is not the fast path.

### 5.2 What “unchanged” means

`UNCHANGED_EXACT` means the exact validated checkpoint target has been observed to have the same checkpoint identity/state as the canonical reconciled base.

It does **not** mean:

- no repository-global commits occurred;
- transport saw no new messages;
- a cached timestamp did not change;
- a branch name looks the same;
- a probe failed;
- the system did not check;
- remote content is “probably unchanged”.

### 5.3 Fast-path effect

The fast path may:

- use local canonical detailed memory as the active working source under ADR-0001;
- rely on trusted local `B` as the already reconciled shared checkpoint representation;
- record the exact trusted remote observation as `observed_sha`/ABSENT as ADR-0003 permits;
- skip a full remote checkpoint fetch for this startup/sync check.

The fast path does **not**:

- advance `reconciled_sha` merely because a probe ran;
- create a new reconciled base;
- claim a new remote synchronization event occurred;
- authorize any future write without rechecking the write path's current prerequisites;
- bypass privacy if an outbound candidate is later created.

## 6. Changed-remote path

### 6.1 Entry condition

The changed-remote path requires:

- valid canonical binding;
- a trusted base (`BASE_READY_PRESENT` or `BASE_READY_ABSENT`);
- supported schema/capability for reconciliation;
- no unresolved condition that requires recovery first;
- trusted evidence that the exact remote checkpoint no longer matches the reconciled base and is present.

### 6.2 Exact fetch is mandatory

A changed probe is an observation signal, not merge content.

The path must load the exact remote checkpoint revision used as ADR-0003 `R`.

```text
trusted changed observation
    -> exact checkpoint fetch
    -> R + R_sha
    -> project current local shared intent to L relative to captured B
    -> ADR-0003 reconciliation
```

If the remote advances between probe and fetch, the operation uses the exact fetched revision and ADR-0003 semantics. It does not merge against guessed contents from the earlier probe.

### 6.3 Observation remains separate from reconciliation

The changed-remote path may update `observed_sha` from trusted observation/fetch.

It may advance canonical `B/reconciled_sha` only through ADR-0003's successful guarded local finalization after deterministic reconciliation and, where a write occurred, exact remote confirmation.

Therefore:

```text
full fetch != reconciliation
observed_sha != reconciled_sha
remote current head != automatically accepted base
```

### 6.4 No-write acceptance versus write candidate

If deterministic ADR-0003 reconciliation determines that the accepted checkpoint is already exactly remote `R` and no remote mutation is needed, the path may proceed to the guarded local finalization allowed by ADR-0003. M1.7 does not add a publication privacy requirement to a read/no-write acceptance.

If reconciliation produces an outbound candidate requiring remote mutation:

```text
candidate C
    -> current ADR-0005 privacy ALLOW for exact candidate/bundle/destination
    -> ADR-0003 write-capable CAS
    -> exact confirmation
    -> guarded local finalization
```

Any candidate/privacy-context drift before the write invalidates the old privacy result.

## 7. Recovery path

Recovery is a fail-closed route for restoring sufficient trustworthy inputs to make a later route decision. It is not a synonym for “sync harder” and it is not a success state.

### 7.1 Recovery entry classes

Recovery includes at least:

- missing/ambiguous/mismatched binding;
- `BASE_UNINITIALIZED`;
- `BASE_INVALID`;
- probe unavailable/unknown/invalid;
- unsupported/unknown schema;
- unsupported/unknown adapter capability;
- previously-present checkpoint observed absent;
- remote commit/confirmation unknown;
- local finalization failed/unknown;
- any startup state in which the system cannot prove fast-path or changed-remote preconditions.

### 7.2 Recovery may perform read-only evidence gathering

Where authorization and capability permit, recovery may perform read-only operations needed to establish current facts, such as:

- exact target validation;
- exact remote probe;
- exact remote fetch;
- schema/capability inspection;
- remote re-observation after ambiguous commit;
- local canonical base-integrity verification.

These operations do not by themselves establish reconciliation.

### 7.3 Recovery exits by re-selection

When recovery obtains new trusted evidence, it does not declare synchronization success merely because the uncertainty was reduced.

Instead:

```text
recovery evidence changes state
    -> rebuild current route inputs
    -> re-run route-selection table
    -> NORMAL_FAST_PATH
       or CHANGED_REMOTE_PATH
       or remain RECOVERY_PATH
```

### 7.4 BASE_UNINITIALIZED

For `BASE_UNINITIALIZED`:

```text
current remote present -> RECONCILIATION_BASE_REQUIRED
current remote absent  -> RECONCILIATION_BASE_REQUIRED
```

M1.7 does not define the bootstrap/adoption mechanism that establishes first trusted lineage.

The newest remote checkpoint cannot be automatically promoted to `B/reconciled_sha` merely to make startup proceed.

### 7.5 BASE_INVALID

`BASE_INVALID` means trusted B-to-SHA lineage cannot be established.

Recovery must not repair it using:

- transport CURRENT snapshot;
- mailbox history;
- receipt history;
- transport Git SHA;
- newest remote checkpoint;
- guessed local file;
- directory/path similarity.

A later implementation may provide an explicit trusted repair/rebootstrap workflow, but M1.7 does not define that mechanism.

### 7.6 Unknown remote commit or confirmation

An ambiguous remote write enters ADR-0003-compatible recovery:

```text
unknown remote commit/confirmation
    -> fresh exact re-observation
```

If the exact candidate revision is confirmed and the captured local base remains valid, the operation may resume guarded local finalization under ADR-0003.

If remote state differs, the old candidate is not blindly replayed. The operation recomputes or surfaces conflict/unknown as ADR-0003 requires.

### 7.7 Local finalization failed or unknown

Remote acceptance/write does not make the local instance reconciled if canonical local finalization failed or is unknown.

The persistent old canonical local base remains the reconciliation bookkeeping authority.

Recovery re-observes remote state and proceeds from that old persistent base according to ADR-0003. It does not synthesize marker advancement from process memory or presumed prior success.

## 8. Remote absence semantics

Remote absence is route-sensitive.

### 8.1 Trusted absent lineage

If:

```text
BaseState == BASE_READY_ABSENT
trusted exact probe == remote ABSENT
```

the remote state is unchanged and the normal fast path is allowed.

### 8.2 Trusted absent lineage becomes present

If:

```text
BaseState == BASE_READY_ABSENT
trusted exact probe == PRESENT
```

the changed-remote path performs an exact fetch and reconciliation.

The new present checkpoint is not auto-adopted merely because the old base was absent.

### 8.3 Previously present becomes absent

If:

```text
BaseState == BASE_READY_PRESENT
trusted exact probe == ABSENT
```

the route is recovery with `REMOTE_CHECKPOINT_ABSENT_CONFLICT` under ADR-0003.

M1.7 does not define deletion/tombstone authorization, so disappearance cannot be normalized into an ordinary successful deletion transition.

### 8.4 Uninitialized base and absent remote

`BASE_UNINITIALIZED + remote ABSENT` remains base-required.

Current absence does not prove trusted absent lineage.

## 9. Local-only degraded continuation

A recovery condition may coexist with usable local project memory. M1.7 therefore permits a **degraded local-only continuation** when local policy and local-memory integrity allow work to continue without remote synchronization.

This is a mode inside the recovery path, not a fourth route.

A degraded local-only continuation may:

- read and use trusted local project memory/evidence;
- continue local-only reasoning/work;
- record local work through local-safe-write rules where otherwise permitted;
- report the exact reason remote synchronization is unavailable/untrusted.

It must not:

- perform remote checkpoint writes;
- enter write-capable CAS;
- advance `reconciled_sha`, `B`, or `base_pair_token`;
- claim that remote state is current, reconciled, synchronized, or unchanged;
- treat cached/transport state as a replacement remote truth;
- silently publish later work when connectivity/trust returns without fresh route selection and privacy evaluation.

If trusted read-only remote observations exist during degraded continuation, they remain observation facts only. The continuation itself still must not be labeled remote-current/reconciled unless normal ADR-0003 finalization later succeeds.

## 10. Binding, schema, and capability failures

### 10.1 Binding

Missing, ambiguous, or mismatched binding always blocks automatic adapter synchronization.

Recovery must use the canonical binding/routing authority; it cannot infer identity from transport or path similarity.

### 10.2 Schema

Unsupported or unknown checkpoint/schema semantics route to recovery.

The system must not:

- coerce unknown fields into known schema;
- discard unknown fields merely to proceed;
- downgrade schema silently;
- write a candidate whose validation semantics are unavailable.

### 10.3 Capability

Unsupported or unknown adapter capability routes to recovery.

No capability failure may invoke transport as an adapter fallback.

A later implementation may expose upgrade/compatibility guidance, but M1.7 does not define that UX or mechanism.

## 11. Marker and claim discipline

Route labels and state markers must not overstate what happened.

### 11.1 `observed_sha`

May advance only through trusted remote observation semantics.

An exact probe or exact fetch can provide such observation if its later implementation satisfies ADR-0003 trust requirements.

### 11.2 `reconciled_sha` / `B`

May advance only through ADR-0003 guarded local finalization.

The following never advance `reconciled_sha` by themselves:

- successful probe;
- full fetch;
- changed-remote detection;
- recovery fetch;
- remote CAS attempt;
- remote CAS success without exact confirmation and local finalization;
- ambiguous remote commit;
- local finalization failure/unknown;
- degraded local continuation.

### 11.3 Route result is not synchronization result

Examples:

```text
ROUTE_FAST_UNCHANGED
    means exact trusted probe showed no remote checkpoint change
    != new reconciliation transaction completed

ROUTE_CHANGED_REMOTE
    means exact remote reconciliation work is required
    != reconciliation succeeded

ROUTE_RECOVERY_REQUIRED
    means trust/state repair is required
    != synchronization failed permanently
    != synchronization succeeded
```

## 12. Typed route outcomes

The future unified interface must preserve route identity and reason without prescribing exact implementation enums.

Conceptual taxonomy:

```text
ROUTE_FAST_UNCHANGED_PRESENT
ROUTE_FAST_UNCHANGED_ABSENT

ROUTE_CHANGED_REMOTE_PRESENT

ROUTE_RECOVERY_PROBE_UNKNOWN
ROUTE_RECOVERY_BINDING_MISSING
ROUTE_RECOVERY_BINDING_AMBIGUOUS
ROUTE_RECOVERY_BINDING_MISMATCH
ROUTE_RECOVERY_BASE_REQUIRED
ROUTE_RECOVERY_BASE_INVALID
ROUTE_RECOVERY_REMOTE_ABSENT_CONFLICT
ROUTE_RECOVERY_SCHEMA_UNSUPPORTED
ROUTE_RECOVERY_CAPABILITY_UNSUPPORTED
ROUTE_RECOVERY_REMOTE_COMMIT_UNKNOWN
ROUTE_RECOVERY_CONFIRMATION_UNKNOWN
ROUTE_RECOVERY_LOCAL_FINALIZATION_FAILED
ROUTE_RECOVERY_LOCAL_FINALIZATION_UNKNOWN

LOCAL_DEGRADED_CONTINUATION
```

These names are architecture taxonomy only. Current runtime code is not claimed to implement them.

Unknown/unsupported results must not be flattened into `ROUTE_FAST_UNCHANGED`.

## 13. Required invariants

The M1.7 architecture is invalid if any of the following can occur silently:

1. full checkpoint fetch is skipped without an exact trusted unchanged probe;
2. probe unavailable/timeout/unknown is interpreted as unchanged;
3. a repository-global HEAD, transport SHA, cached timestamp, or path state substitutes for exact checkpoint probe identity;
4. full fetch is treated as reconciliation or `observed_sha` is copied into `reconciled_sha`;
5. a changed present remote bypasses exact fetch before ADR-0003 reconciliation;
6. `BASE_UNINITIALIZED` auto-adopts the newest remote checkpoint or current absence;
7. `BASE_INVALID` is repaired from transport snapshot/history or guessed remote/local state;
8. previously-present -> remote absent is treated as ordinary successful changed-remote deletion without an accepted deletion/tombstone policy;
9. unknown remote commit/confirmation causes blind retry, force overwrite, rollback, or marker advancement;
10. remote CAS success is treated as reconciliation success after local finalization failed/unknown;
11. recovery is reported as successful synchronization merely because read-only evidence was gathered;
12. degraded local-only continuation performs remote write, advances reconciled markers, or claims remote-current state;
13. transport is invoked as adapter fallback for probe/fetch/reconciliation/recovery failure;
14. a write-capable candidate enters CAS without a current ADR-0005 privacy `ALLOW`;
15. candidate/privacy-context drift after reconciliation or recovery reuses an old privacy verdict;
16. fast/changed/recovery route selection proceeds despite missing/ambiguous/mismatched canonical binding;
17. unsupported/unknown schema or capability is silently coerced into a writable state;
18. route selection bypasses an unresolved remote-commit/confirmation/local-finalization recovery state.

## 14. Decisions requested for M1.7 initial review — FP1–FP14

Initial review should output `APPROVE` or `REVISE` for every decision below.

- **FP1 — Startup trust inputs.** Route selection uses explicit binding, base, schema/capability, unresolved-operation, and exact-probe trust states; unknown inputs are not optimistic success.
- **FP2 — Route selection.** Every startup/sync decision selects only normal fast path, changed-remote path, or recovery path using the deterministic priority/table in this proposal.
- **FP3 — Fast-path preconditions.** Full checkpoint fetch may be skipped only with valid binding, trusted base, supported schema/capability, no unresolved recovery state, and an exact trusted probe proving the checkpoint is unchanged.
- **FP4 — Unchanged semantics.** “Unchanged” refers to the exact validated checkpoint identity/state relative to the canonical reconciled base, not repository HEAD, transport state, timestamps, cached assumptions, or probe failure.
- **FP5 — Changed-remote fetch route.** A trusted changed present remote requires exact checkpoint fetch and then ADR-0003 B/R/L reconciliation; probe output is not merge content.
- **FP6 — Observation vs reconciliation.** Probe/fetch may establish `observed_sha`, but only ADR-0003 guarded local finalization may advance `B/reconciled_sha`; full fetch, remote CAS, and remote head movement remain distinct.
- **FP7 — Base-state routing.** `BASE_READY_PRESENT/ABSENT`, `BASE_UNINITIALIZED`, and `BASE_INVALID` route distinctly; uninitialized/invalid bases cannot be auto-repaired from newest remote or transport state.
- **FP8 — Binding/schema/capability failures.** Missing/ambiguous/mismatched binding and unsupported/unknown schema/capability route to recovery and block automatic writes rather than using guesses, coercion, or fallback.
- **FP9 — Remote-absence handling.** Trusted absent lineage remaining absent may use fast path; trusted absent becoming present uses changed-remote fetch; previously-present becoming absent is a recovery conflict; uninitialized+absent remains base-required.
- **FP10 — Unknown commit/finalization recovery.** Ambiguous remote commit/confirmation and failed/unknown local finalization enter ADR-0003-compatible recovery, preserve the persistent canonical base, and never imply reconciliation success.
- **FP11 — Degraded local-only continuation.** Recovery may permit explicit local-only continuation when local state is trustworthy, but it cannot write remote state, advance reconciliation markers, or claim remote-current/synchronized status.
- **FP12 — No transport fallback.** Transport mailbox/snapshot/history/SHAs remain outside adapter route authority and cannot substitute for probe, base, recovery lineage, or Hub write paths.
- **FP13 — Privacy-before-write and marker advancement.** Any exact outbound write candidate requires current ADR-0005 privacy `ALLOW` before CAS; privacy drift forces re-evaluation, while reconciled markers advance only through ADR-0003 finalization.
- **FP14 — Protocol-only scope/deferred implementation.** M1.7 fixes route semantics only and defers budgets to M1.8 plus concrete API/schema encoding, locks, retry values, implementation, migration, UI, validation, and real-data operations.

## 15. Deferred implementation and next milestone

The following remain intentionally deferred:

- all measurable token/memory/I/O/payload/compaction/probe budgets to M1.8;
- concrete lightweight-probe implementation and API;
- concrete exact-fetch implementation;
- adapter module/interface design;
- sync-metadata physical schema and storage;
- lock/serialization/transaction primitives;
- retry/backoff values;
- credential implementation;
- publication trigger policy;
- detector/classifier implementation;
- migration;
- real-data operations;
- complete UI/operator workflow;
- executable/static/live validation.

M1.7 should not create additional architecture sub-proposals. Any implementation question that does not change FP1–FP14 semantics is deferred to M2/M3/M4 or later validation work.

## 16. Evidence and architecture boundary

This proposal depends on and preserves:

- `ADR-0001` — local/shared authority and minimized shared projection;
- `ADR-0002` — guarded cooperative local writes;
- `ADR-0003` — B/R/L/C reconciliation, observation/base separation, CAS, exact confirmation, finalization, base states, and ambiguous-operation recovery;
- `ADR-0004` — transport/Hub-adapter separation, resource identity, and no fallback;
- `ADR-0005` — destination-aware privacy and current privacy-before-write gate.

This document is a proposal only. It is `normative:false`, `proposed-unreleased`, and `initial-review-pending`. It creates no runtime-effective behavior, implements no adapter, performs no migration, establishes no M1.8 budget, validates no implementation, and authorizes no real-data operation.
