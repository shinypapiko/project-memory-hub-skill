---
document_role: architecture-proposal
status: review-required
normative: false
architecture_state: proposed-unreleased
runtime_load_policy: maintenance-only
milestone: M1.5
created: 2026-08-27
last_reviewed: 2026-08-27
review_state: initial-review-pending
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
  - docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md
---

# Hub adapter vs existing transport boundary — M1.5

## 0. Purpose and scope

This proposal defines the semantic boundary between:

1. the **existing executable Git transport** in the audited local Project Memory distribution; and
2. the **future Hub adapter** required by the accepted unified architecture.

The goal is to allow both layers to coexist, and to permit reuse of semantics-neutral Git/filesystem plumbing where useful, without allowing either layer to impersonate the other.

This proposal does **not** reopen ADR-0001, ADR-0002, or ADR-0003. It does not implement the Hub adapter, alter released Hub Skill `0.1.0`, alter the deployed local Skill, choose GitHub/API/credential mechanisms, define publication triggers, define privacy-detection tooling, or migrate real local/Runtime Hub data.

M1.5 is specifically about:

- semantic ownership;
- authority and non-authority;
- state namespaces;
- accepted inputs/outputs;
- allowed cross-layer calls;
- forbidden fallback/reinterpretation;
- coexistence and capability identity;
- failure/result namespaces;
- compatibility boundaries.

## 1. Baseline facts this proposal preserves

### 1.1 Existing transport

The audited local transport is already executable and serves a Git-backed **mailbox/exchange** role. Its capabilities include handoff/message exchange, receipt/ack lifecycle, transport-local state, and an optional CURRENT snapshot that is explicitly **non-canonical**.

Its transport snapshot is not a shared-checkpoint database. `receive`, `publish`, `publish-snapshot`, `ack`, and `sync` are transport operations, not ADR-0003 reconciliation operations.

### 1.2 Future Hub adapter

The future Hub adapter is the semantic layer that will eventually connect the local memory core to Runtime Hub shared coordination state. Under accepted ADRs it must respect:

- Hub project routing and project identity;
- the future shared checkpoint role of Hub `projects/<project-id>/PROJECT.md`;
- minimized local-to-shared projection;
- ADR-0003 `B/R/L/C`, `observed_sha`, `reconciled_sha`, `base_pair_token`, CAS, conflict-resolution, exact revision confirmation, and guarded local finalization semantics;
- later M1.6 publication/privacy gates;
- later M1.7 recovery/bootstrap policy.

No such adapter implementation is claimed to exist yet.

## 2. Two semantic layers, not two names for the same layer

The architecture requires two distinct subsystem identities:

```text
TRANSPORT
  purpose: exchange / mailbox / handoff lifecycle

HUB_ADAPTER
  purpose: shared-checkpoint observation / reconciliation / finalization
```

They may share lower-level utility code later, but **code reuse does not transfer semantic authority**.

The following equivalences are forbidden:

```text
mailbox message        != shared-checkpoint candidate
mailbox receive        != checkpoint hydration/reconciliation
transport publish      != shared-checkpoint publication
receipt / ack          != checkpoint reconciliation confirmation
CURRENT snapshot       != B
transport remote SHA   != observed_sha
transport state        != base_pair_token/reconciled state
transport sync success != RECONCILED_* success
```

## 3. Semantic ownership model

### 3.1 Transport-owned semantics

Transport owns only exchange lifecycle semantics such as:

- outbound handoff/message construction;
- remote channel/mailbox placement;
- receive/discovery of transport messages;
- receipt/acknowledgement of transport message processing;
- transport-local delivery/state bookkeeping;
- optional non-canonical CURRENT mirror/snapshot;
- transport-specific duplicate/hash/path/dirty-worktree/non-fast-forward safety results.

A transport record may carry claims, conclusions, references, or suggested next actions, but receiving it does not make those claims canonical local memory or canonical shared Hub state.

### 3.2 Hub-adapter-owned semantics

The future Hub adapter owns only shared coordination/checkpoint semantics such as:

- deterministic Runtime Hub project target selection after project identity is already resolved according to released/accepted routing policy;
- shared-checkpoint probe/fetch;
- adapter sync metadata;
- `observed_sha`;
- canonical local `B/reconciled_sha/base_pair_token` integration;
- construction/use of ADR-0003 reconciliation operations;
- remote CAS against the exact classified revision;
- exact committed-revision confirmation;
- guarded local reconciliation finalization;
- structured field conflicts/resolution contexts;
- structured unknown/stale/finalization outcomes;
- later checkpoint hydration/bootstrap only under M1.7 rules;
- later publication only from a separately produced, policy-approved shared candidate.

### 3.3 Local Memory Core remains a separate authority domain

Neither transport nor Hub adapter becomes the general owner of local `.ai/PROJECT.md`, `.ai/CURRENT.md`, TASKS, DEC/EXP, indexes, or local sessions/research.

A mailbox handoff may influence local canonical memory only through the ordinary local validation/adoption/checkpoint path. A later Hub shared candidate may then be derived from the resulting local canonical state under accepted projection/publication policy.

The bridge is therefore:

```text
transport message
    -> validate / inspect
    -> local canonical adoption if warranted
    -> later minimized shared projection
    -> Hub adapter reconciliation/publication path
```

The forbidden shortcut is:

```text
transport message/receipt/snapshot
    -> directly become Hub B/L/C/reconciled state
```

## 4. State namespace separation

The two subsystems require logically independent state namespaces even if a future implementation stores them in one physical configuration envelope.

Minimum logical separation:

```text
transport.*
  channel/message ids
  inbox/outbox/processed state
  receipt/ack state
  transport peer/channel metadata
  transport snapshot metadata
  transport delivery/retry state

hub_adapter.*
  bound project/checkpoint identity
  observed remote checkpoint identity
  canonical reconciled base identity
  reconciled_sha
  base_pair_token
  adapter schema/capability identity
  reconciliation/finalization state
```

Rules:

1. A transport field must never be read as an adapter field merely because the values have the same type (for example both happen to be Git SHAs).
2. Adapter state must never be reconstructed from transport snapshot/receipt metadata.
3. Transport state must never be overwritten by adapter recovery as if it were adapter cache.
4. Physical co-location in one file is permitted only if the released design retains explicit typed namespaces and independent validation; separate physical files remain equally valid. M1.5 does not choose the physical layout.
5. Existing `transport.json` semantics remain transport-only. A future implementation may wrap or migrate configuration only through a later explicit compatibility design; M1.5 does not silently extend existing transport fields to mean adapter state.

## 5. Capability matrix

| Capability | Semantic owner | Transport semantics | Hub adapter semantics | Canonical/shared-checkpoint effect | Allowed bridge | Forbidden reinterpretation/fallback | Result/failure namespace | Maturity |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Project routing | Hub routing / binding policy | May address a configured transport peer/channel; not project-state authority | Selects/validates the Runtime Hub project/checkpoint target under project identity rules | Adapter target selection may affect which checkpoint is read; transport routing does not | Stable `project_id` may be referenced by both when explicitly configured | A mailbox channel/name must not become Hub project identity by coincidence | `TRANSPORT_*` routing/config errors vs `ADAPTER_*` identity/routing errors | transport current; adapter future |
| Handoff/message | Transport | Primary exchange payload/lifecycle | Not a checkpoint primitive | none directly | Validated content may later be adopted into local memory | Handoff body must not become `L`, `B`, or `C` directly | transport message validation/delivery results | current transport |
| Receive | Transport | Discovers/copies transport message into handoff/inbox lifecycle | Adapter may later fetch remote checkpoint, but that is a separate operation | no `B/reconciled_sha` advance | Received claims may be validated locally | `receive` must not be called/treated as checkpoint hydration or reconcile | `TRANSPORT_RECEIVE_*` | current transport |
| Receipt / ack | Transport | Proves transport message-processing acknowledgement only | No checkpoint confirmation semantics | none | May be retained as provenance that a message was processed | Receipt/ack must not confirm CAS, exact revision, `RECONCILED_*`, or `reconciled_sha` | `TRANSPORT_ACK_*` | current transport |
| CURRENT snapshot | Transport | Optional non-canonical mirror of selected local CURRENT information | No adapter authority | none | May assist human/agent awareness only | Snapshot must never become `B`, `R`, `L`, `C`, hydration base, or recovery base | `TRANSPORT_SNAPSHOT_*` | current transport |
| Transport status/state | Transport | Delivery/config/channel/peer bookkeeping | Separate from adapter sync metadata | none | Diagnostic display may show both subsystems side-by-side | Transport state/SHA must not seed `observed_sha`, `reconciled_sha`, or `base_pair_token` | `TRANSPORT_STATUS_*` | current transport |
| Shared checkpoint | Hub adapter / Runtime Hub | No ownership | Fetches/reconciles future Hub shared checkpoint | authoritative for shared-checkpoint scope only after accepted ADR semantics/release | Transport may reference checkpoint IDs as opaque references only | Mailbox message or snapshot cannot substitute for checkpoint content | `ADAPTER_CHECKPOINT_*` | future |
| Remote probe | Hub adapter | Transport may inspect its own Git/mailbox remote for transport safety only | Observes shared-checkpoint remote identity | may update `observed_sha` only under ADR-0003 observation semantics | Low-level Git read helper may later be shared | A transport `git fetch/status` result cannot update adapter `observed_sha` | `ADAPTER_PROBE_*` | future |
| `observed_sha` | Hub adapter | no meaning | Latest trusted observation of checkpoint state | observation only; not reconciliation | none from transport | Transport commit/blob/message SHA must never be promoted to `observed_sha` by type matching | adapter observation results | future, ADR-0003 protocol-only |
| `reconciled_sha` / `B` | Hub adapter + guarded local canonical pair | no meaning | Last fully reconciled shared base | canonical reconciliation bookkeeping | none from transport | Receipt, snapshot, message delivery, transport sync or remote SHA cannot advance `B/reconciled_sha` | ADR-0003 finalization results | future, ADR-0003 protocol-only |
| `base_pair_token` | Hub adapter/local safe-write integration | no meaning | Guards canonical local reconciled-pair advancement | protects reconciliation bookkeeping | may reuse future low-level local safe-write helper only | Transport state/version/message id cannot act as base-pair token | local/adapter stale/finalization results | future, ADR-0002/0003 protocol-only |
| Shared candidate projection | Local Memory Core + later publication gate | Transport payload is not projection input by virtue of being received | Adapter accepts only a separately produced shared-schema candidate | may become `L` only after projection/policy | Validated handoff may change local canonical memory, which later influences projection | Direct transport->`L` passthrough is forbidden | later projection/privacy result namespace | future; M1.6+ |
| Publication | Semantically split | `publish` means publish a transport message to mailbox/channel | Future checkpoint publication means reconcile/commit an approved shared candidate | transport publish: none; adapter publication: checkpoint mutation under ADR-0003 | none except explicit higher-level orchestration may invoke each separately | Adapter unavailable must not fall back to transport `publish`; transport unavailable must not fall back to checkpoint write | `TRANSPORT_PUBLISH_*` vs `ADAPTER_PUBLISH/RECONCILE_*` | transport current; adapter future |
| Hydration | Hub adapter / recovery policy | Receive only supplies handoff proposals | Future hydration obtains shared checkpoint/recovery input under M1.7 | never auto-advances reconciled base merely by fetch | A hydrated shared fact may later be incorporated into local memory through defined reconciliation/adoption | `receive` or CURRENT snapshot cannot serve as hydration baseline | `ADAPTER_HYDRATE_*` / base-required/invalid outcomes | future M1.7/M3 |
| Conflict | Semantically split | Transport conflict means duplicate/hash/path/non-fast-forward/delivery problem | Adapter conflict means semantic B/R/L/identity/schema/resolution conflict | adapter conflicts block checkpoint advancement | Common UI may display both with typed source | Transport conflict resolution must not resolve ADR-0003 field conflict, and vice versa | separate typed conflict classes | transport current; adapter future |
| Session log / shared trace | Local core or explicit Hub shared-record layer | A handoff may reference a session but does not own session history | Future Hub shared session/trace is distinct from checkpoint and transport receipt | no automatic checkpoint effect | Explicit promoted trace may reference transport message IDs as provenance | receipt history must not be reclassified as Hub session evidence automatically | `LOCAL_SESSION_*` / future `HUB_TRACE_*` | local current; Hub trace protocol/future integration |
| Recovery | Semantically split | Retries/rediscovers transport messages and delivery state | Recovers checkpoint/base lineage only under ADR-0003 + later M1.7/M3 | adapter recovery may eventually restore trusted reconciliation bookkeeping | Shared diagnostic shell may report both | Transport snapshot/message history must not repair missing/corrupt `B/reconciled_sha`; adapter state must not repair mailbox delivery state | `TRANSPORT_RECOVERY_*` vs `ADAPTER_BASE/RECOVERY_*` | transport current; adapter future |
| Low-level Git/filesystem plumbing | Utility layer, no semantic authority | May use pull/push/hash/path/dirty-worktree helpers | May later use fetch/read/CAS/revision helpers if verified suitable | none by utility call alone | Reuse is allowed only through semantics-neutral helpers with typed callers/results | A shared helper must not silently map transport success to adapter success or vice versa | utility errors wrapped by owning subsystem | implementation deferred |
| Configuration/capability discovery | Each subsystem independently | Transport config/version/capabilities describe mailbox layer | Adapter config/version/capabilities describe checkpoint layer | none by discovery alone | Higher-level status may aggregate both | One generic `hub_enabled=true` must not imply both are configured/compatible | typed configuration/capability errors | transport current; adapter future |

## 6. Call and data-flow rules

### 6.1 Allowed calls

The architecture permits these directions when later implementation exists:

```text
Local Memory Core -> Transport
  send/receive handoff lifecycle

Transport -> Local Memory Core
  deliver a proposal/handoff for validation and possible adoption

Local Memory Core -> Hub Adapter
  provide separately derived shared-schema candidate / request observation-reconciliation

Hub Adapter -> Local Memory Core
  return structured remote shared state/conflict/finalization result
  update only adapter-owned canonical reconciliation metadata through guarded local semantics

Transport + Hub Adapter -> shared utility layer
  use semantics-neutral Git/path/hash/serialization primitives
```

### 6.2 Forbidden direct calls or semantic shortcuts

```text
Transport -> set observed_sha
Transport -> set reconciled_sha/B/base_pair_token
Transport -> mark RECONCILED_*
Transport snapshot -> adapter hydration base
Transport receipt -> adapter commit confirmation
Transport publish -> adapter publication fallback
Hub Adapter -> mutate mailbox receipt/delivery state as recovery
Hub Adapter -> reinterpret handoff body as shared candidate without local adoption/projection
```

If a higher-level orchestrator invokes both subsystems in one workflow, each operation retains its own typed result and commit meaning. "Both succeeded" may be reported only as a composite result; neither subsystem's success is evidence for the other's success.

## 7. Coexistence and capability identity

### 7.1 Both may be installed/enabled independently

Valid future combinations include:

```text
transport OFF, adapter OFF
transport ON,  adapter OFF
transport OFF, adapter ON
transport ON,  adapter ON
```

No combination implies silent fallback.

### 7.2 Capability negotiation must be typed

A future implementation must expose subsystem identity/version/capability in a way that cannot confuse the two layers. Conceptually:

```text
subsystem: transport
capability: mailbox_exchange

subsystem: hub_adapter
capability: shared_checkpoint_reconciliation
```

M1.5 does not fix the physical config keys or version numbers.

### 7.3 Shared Git clone/worktree is not automatically forbidden, but semantics stay isolated

A later implementation may decide to use the same underlying Git repository/worktree for multiple remote operations only if it can preserve:

- disjoint logical state namespaces;
- typed operation purpose;
- independent preconditions;
- independent result/failure classification;
- no cross-layer dirty-state or commit inference;
- no implicit state adoption from one layer to the other.

If those properties cannot be proven, the implementation must use separate worktrees/state stores or fail closed. M1.5 does not choose the concrete topology.

## 8. Failure and fallback contract

### 8.1 No silent fallback

Examples:

```text
adapter unavailable
  -> ADAPTER_UNAVAILABLE
  -> NOT transport publish/snapshot fallback

transport unavailable
  -> TRANSPORT_UNAVAILABLE
  -> NOT checkpoint write fallback

adapter base invalid
  -> ADR-0003 base error
  -> NOT repair from mailbox snapshot

transport receive hash mismatch
  -> transport integrity error
  -> NOT adapter semantic conflict
```

### 8.2 Composite workflows preserve per-subsystem results

If a future workflow does both:

```text
1. publish transport handoff
2. reconcile Hub checkpoint
```

then possible results include:

```text
transport=COMMITTED, adapter=FAILED
transport=FAILED, adapter=RECONCILED
transport=COMMITTED, adapter=RECONCILED
transport=UNKNOWN, adapter=NOT_ATTEMPTED
```

The system must not flatten these into one ambiguous boolean or infer one operation from the other.

### 8.3 Unknown states remain local to their subsystem

Transport unknown delivery/commit state does not make adapter reconciliation unknown unless a separately attempted adapter operation is itself unknown. Adapter `REMOTE_COMMIT_STATE_UNKNOWN` does not imply a transport message was or was not delivered.

## 9. Authority and provenance crossing rules

### 9.1 Transport-to-local adoption

Receiving a handoff produces a **proposal/evidence input**, not canonical truth.

Before durable local adoption, the local memory workflow may require direct validation, provenance classification, or explicit user/agent acceptance according to local policy. Only after adoption does the conclusion become eligible to influence a later shared projection.

### 9.2 Transport references may be provenance, not authority

A shared candidate/checkpoint may later include a permitted provenance reference to a transport message/receipt where policy allows. Such a reference proves only the referenced transport event and does not elevate the transport record to checkpoint authority.

### 9.3 Adapter output does not rewrite transport history

Successful reconciliation may be linked from a handoff/session record as later evidence, but it does not retroactively change mailbox delivery/ack history.

## 10. Privacy boundary for M1.5

M1.5 does not define the executable P0/P1/P2/P3/UNCLASSIFIED gate; that is M1.6.

It does fix these boundary rules:

- transport receipt/snapshot existence does not waive later publication classification;
- data being remotely present in a mailbox does not make it eligible for shared checkpoint publication;
- transport-to-local adoption and local-to-shared publication are separate policy decisions;
- adapter publication must later apply M1.6 policy to the candidate actually being published;
- no public Skill source fixture may contain real project/handoff/checkpoint/private Runtime Hub content.

## 11. Compatibility lifecycle

The existing transport remains a legacy/current executable subsystem until a later explicit release says otherwise. M1.5 does not rename or upgrade `transport_tool.py` into the Hub adapter.

A future unified release must preserve one of these explicit compatibility outcomes for existing transport users:

- transport retained as a separately typed supported subsystem;
- transport deprecated with an explicit migration/deprecation path;
- transport removed only through an explicit breaking compatibility decision.

Under no outcome may existing transport state be silently reinterpreted as adapter reconciliation state.

## 12. Required invariants

The design is invalid if any of the following occurs silently:

1. a transport CURRENT snapshot becomes `B` or hydration base;
2. a transport receipt/ack becomes exact-revision or reconciliation confirmation;
3. a transport remote SHA becomes `observed_sha`/`reconciled_sha` by type matching;
4. mailbox receive advances canonical local reconciliation state;
5. transport publish is used as adapter publication fallback;
6. adapter checkpoint write is used as transport delivery fallback;
7. transport and adapter write the same untyped state keys;
8. a generic success/unknown/error code hides which subsystem produced it;
9. missing/corrupt adapter base is repaired from transport snapshot/history;
10. a received handoff bypasses local adoption and becomes shared candidate directly;
11. shared low-level Git helpers acquire semantic authority from one caller and leak it into another;
12. current released Skill behavior is described as already using this future adapter.

## 13. Decisions requested for M1.5 review

Initial review should approve or revise these decisions:

- **TB1:** existing transport and future Hub adapter are independent semantic subsystems with different authority and lifecycle.
- **TB2:** transport owns mailbox/handoff/receipt/non-canonical snapshot semantics only; adapter owns shared-checkpoint observation/reconciliation/finalization semantics only.
- **TB3:** transport state and adapter state use independent logical namespaces; equal-looking IDs/SHAs never imply semantic equivalence.
- **TB4:** transport receive produces only a handoff/proposal input; it cannot advance `B`, `observed_sha`, `reconciled_sha`, or adapter finalization state.
- **TB5:** transport receipt/ack proves message lifecycle only and cannot confirm checkpoint CAS, exact revision, or reconciliation.
- **TB6:** transport CURRENT snapshot remains non-canonical and cannot serve as checkpoint/hydration/recovery base.
- **TB7:** transport `publish` and future shared-checkpoint publication are distinct operations; neither is a fallback for the other.
- **TB8:** shared Git/filesystem plumbing may be reused only as semantics-neutral utilities with typed callers/results; reuse does not merge authority.
- **TB9:** adapter unavailable/failed/unknown and transport unavailable/failed/unknown remain separate typed outcomes with no silent fallback.
- **TB10:** received transport content may reach Hub only through validate/adopt into local canonical memory, then later explicit shared projection/publication; direct transport-to-checkpoint passthrough is forbidden.
- **TB11:** local session history, Hub shared trace, transport receipt history, and shared checkpoint are separate record classes; references may connect them without transferring authority.
- **TB12:** capability/config discovery must identify `transport` and `hub_adapter` independently; a generic untyped "hub enabled" state is insufficient.
- **TB13:** physical config/worktree co-location is an implementation choice only if logical namespaces, operation purpose, preconditions, and result classes remain provably isolated; otherwise fail closed/separate them.
- **TB14:** existing transport remains current executable behavior until an explicit compatibility/release decision; M1.5 does not turn `transport_tool.py` into the adapter.
- **TB15:** M1.5 remains architecture/protocol-only and does not define adapter code, API/credentials, publication triggers, executable privacy gates, migration, or real-data movement.

## 14. Deferred implementation choices

Deliberately deferred:

- adapter implementation language/module layout;
- exact GitHub/remote API and credentials;
- exact adapter config/state filenames;
- whether transport and adapter use separate or shared Git clones/worktrees;
- exact common Git helper interface;
- concrete capability/version negotiation fields;
- publication trigger policy;
- M1.6 classification implementation;
- M1.7 startup/hydration/recovery workflow;
- migrations and compatibility code;
- retry/backoff values and UI wording.

## 15. Evidence

- `docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md`
- `docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md`
- `docs/decisions/ADR-0002-local-multi-session-safe-write.md`
- `docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md`
- `skill/SKILL.md` and release metadata for current `0.1.0` behavior

This proposal is an M1.5 boundary specification only. Approval must not be interpreted as adapter implementation, runtime effectiveness, migration completion, or validation.