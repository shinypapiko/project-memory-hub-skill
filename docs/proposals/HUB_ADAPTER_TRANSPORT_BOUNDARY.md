---
document_role: architecture-proposal
status: review-required
normative: false
architecture_state: proposed-unreleased
runtime_load_policy: maintenance-only
milestone: M1.5
created: 2026-08-27
last_reviewed: 2026-08-27
review_state: clarifications-integrated-final-review-pending
initial_review_baseline: 0b7cebe64b26f91df40be23e44fa2b786e6a2655
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

This proposal does **not** reopen ADR-0001, ADR-0002, or ADR-0003. It does not implement the Hub adapter, alter released Hub Skill `0.1.0`, alter the deployed local Skill, choose GitHub/API/credential mechanisms, define publication triggers, define privacy-detection tooling, define M1.7 recovery policy, or migrate real local/Runtime Hub data.

M1.5 is specifically about:

- semantic ownership;
- authority and non-authority;
- state namespaces;
- remote resource ownership classes;
- accepted inputs/outputs;
- allowed cross-layer calls;
- cross-layer project-binding checks;
- shared Git transaction-domain constraints;
- forbidden fallback/reinterpretation;
- coexistence and capability identity;
- failure/result namespaces;
- compatibility boundaries.

The initial review at `main@0b7cebe64b26f91df40be23e44fa2b786e6a2655` approved TB1/TB4/TB5/TB6/TB7/TB9/TB11/TB14/TB15, required revision of TB2/TB3/TB8/TB10/TB12/TB13, and identified three blocking defects: remote resource ownership, shared Git transaction-domain isolation, and cross-layer project binding. This revision addresses those defects while preserving the scope above. Formal M1.5 approval remains pending final review.

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

### 3.4 Remote resource ownership classes

Logical subsystem namespaces are not sufficient by themselves. Every remote resource touched by transport, adapter, or Runtime Hub maintenance must belong to one explicit semantic resource class before a write-capable operation proceeds.

M1.5 defines the following ownership classes without fixing final physical paths:

| Remote resource class | Examples / semantic contents | Primary writer(s) | Reader(s) | Conflict behavior | Fail-closed conditions |
| --- | --- | --- | --- | --- | --- |
| **Transport mailbox resources** | transport channel/mailbox containers, message objects, mailbox indexes, receipt/ack records, transport snapshots and transport delivery metadata | Transport subsystem only, according to its mailbox lifecycle | Transport; Local Memory Core may inspect delivered handoff content/provenance; Hub adapter may at most carry opaque references when explicitly permitted | Transport-local duplicate/hash/path/non-fast-forward/delivery conflicts remain transport conflicts; they never become ADR-0003 field conflicts | resource class cannot be determined; a transport write would touch adapter/checkpoint/registry/infrastructure resources; ownership overlaps unexpectedly; target identity is ambiguous |
| **Adapter checkpoint resources** | the future shared checkpoint content that participates in ADR-0003 observation/reconciliation/CAS/finalization | Hub adapter only through ADR-0003-compatible checkpoint operations | Hub adapter; authorized shared-state readers; transport may only reference an opaque checkpoint identity and has no checkpoint-state authority | stale/CAS/semantic/schema/identity/unknown-commit outcomes use adapter reconciliation semantics | checkpoint target cannot be uniquely identified; operation would include transport resources; exact classified revision cannot be established; resource is actually registry/infrastructure state |
| **Adapter local/remote state resources** | adapter-owned sync metadata and any remote adapter-specific coordination metadata distinct from the checkpoint itself | Hub adapter / guarded local state transition as appropriate to the later design | Hub adapter; diagnostic readers may inspect typed state | adapter-state conflict is not transport delivery conflict; canonical pair changes obey ADR-0002/ADR-0003 guards | state namespace/type is ambiguous; transport metadata would be used as canonical adapter state; ownership cannot be verified |
| **Runtime Hub registry/routing resources** | cross-project registry entries, stable project identity/routing metadata and binding lifecycle state | Dedicated Runtime Hub registry/binding lifecycle, not ordinary transport operations and not ordinary checkpoint reconciliation | Hub adapter reads enough to resolve/validate its target; transport may carry a configured project reference but does not own registry truth | identity/routing mismatch fails closed and is not merged as checkpoint content | project identity is missing/ambiguous/mismatched; ordinary adapter reconciliation or transport operation would mutate registry state without a separately authorized registry operation |
| **Project-container/shared infrastructure resources** | Hub metadata, schema/bootstrap markers, container attributes, compatibility/schema metadata and other shared project-container infrastructure | Dedicated Runtime Hub/schema/maintenance lifecycle only unless a later accepted design explicitly delegates a typed operation | Adapter/transport may read only what their operation requires; neither gains general write authority | schema/bootstrap/attribute conflicts remain infrastructure/compatibility conflicts and do not become mailbox or checkpoint conflicts | unsupported/unknown schema; infrastructure ownership is unclear; an ordinary transport or checkpoint operation would mutate infrastructure; a mixed commit cannot prove exact owned-resource scope |

Rules:

1. Resource ownership is semantic, not inferred from directory proximity, filename similarity, Git object type, or a common repository.
2. A subsystem must not write outside the resource classes assigned to its current typed operation.
3. Runtime Hub registry/routing and shared infrastructure are **not** generic adapter-owned storage merely because the adapter reads them.
4. Transport has no write authority over checkpoint, adapter-state, Runtime Hub registry/routing, or shared infrastructure resources.
5. Ordinary checkpoint reconciliation has no write authority over transport mailbox resources or Runtime Hub registry/infrastructure resources.
6. If one proposed operation spans resource classes with different semantic owners and there is no separately accepted composite-operation contract, the operation fails closed rather than bundling them into one convenient commit.
7. M1.5 intentionally does not select the final physical paths for any class.

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

1. A transport field must never be read as an adapter field merely because the values have the same type, for example because both happen to be Git SHAs.
2. Adapter state must never be reconstructed from transport snapshot/receipt metadata.
3. Transport state must never be overwritten by adapter recovery as if it were adapter cache.
4. Physical co-location in one file is permitted only if the released design retains explicit typed namespaces and independent validation; separate physical files remain equally valid. M1.5 does not choose the physical layout.
5. Existing `transport.json` semantics remain transport-only. A future implementation may wrap or migrate configuration only through a later explicit compatibility design; M1.5 does not silently extend existing transport fields to mean adapter state.
6. Logical state namespaces do **not** imply isolation of a shared Git worktree/index/HEAD/branch/ref transaction domain. Repository operation isolation is a separate requirement defined in section 7.3.
7. Logical state namespaces do **not** create independent project identity authorities. Cross-layer identity is governed by section 7.4.

## 5. Capability matrix

The `TRANSPORT_*`, `ADAPTER_*`, `BINDING_*`, and similar typed result names used below are a **future unified-interface taxonomy** for keeping subsystem results distinguishable. They are not a claim that the current executable `transport_tool.py` already emits these structured result objects or exact symbolic names.

| Capability | Semantic owner | Transport semantics | Hub adapter semantics | Canonical/shared-checkpoint effect | Allowed bridge | Forbidden reinterpretation/fallback | Result/failure namespace | Maturity |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Project routing | Runtime Hub routing / canonical workspace binding | May address a configured transport peer/channel and may carry a project reference; neither is independent project authority | Selects/validates the Runtime Hub project/checkpoint target against canonical binding/registry identity | Adapter target selection affects which checkpoint is read; transport routing does not | The same stable `project_id` may be referenced by both only after identity consistency is verified | channel name, directory similarity, `hub_project_path`, or transport envelope identity cannot substitute for canonical binding | future `BINDING_*` / `ADAPTER_*` identity errors; current transport retains its existing behavior | transport current; adapter future |
| Handoff/message | Transport | Primary exchange payload/lifecycle | Not a checkpoint primitive | none directly | Validated content may later be adopted into local memory after binding checks for any automatic cross-layer bridge | Handoff body must not become `L`, `B`, or `C` directly | future transport message taxonomy; current tool behavior unchanged | current transport |
| Receive | Transport | Discovers/copies transport message into handoff/inbox lifecycle | Adapter may later fetch remote checkpoint, but that is a separate operation | no `B/reconciled_sha` advance | Received claims may be validated locally; an automatic bridge also requires canonical binding alignment | `receive` must not be called/treated as checkpoint hydration or reconcile | future `TRANSPORT_RECEIVE_*` taxonomy | current transport |
| Receipt / ack | Transport | Proves transport message-processing acknowledgement only | No checkpoint confirmation semantics | none | May be retained as provenance that a message was processed | Receipt/ack must not confirm CAS, exact revision, `RECONCILED_*`, or `reconciled_sha` | future `TRANSPORT_ACK_*` taxonomy | current transport |
| CURRENT snapshot | Transport | Optional non-canonical mirror of selected local CURRENT information | No adapter authority | none | May assist human/agent awareness only | Snapshot must never become `B`, `R`, `L`, `C`, hydration base, or recovery base | future `TRANSPORT_SNAPSHOT_*` taxonomy | current transport |
| Transport status/state | Transport | Delivery/config/channel/peer bookkeeping | Separate from adapter sync metadata | none | Diagnostic display may show both subsystems side-by-side | Transport state/SHA must not seed `observed_sha`, `reconciled_sha`, or `base_pair_token` | future `TRANSPORT_STATUS_*` taxonomy | current transport |
| Shared checkpoint | Hub adapter / Runtime Hub checkpoint class | No ownership | Fetches/reconciles future Hub shared checkpoint | authoritative for shared-checkpoint scope only after accepted ADR semantics/release | Transport may reference checkpoint IDs as opaque references only | Mailbox message or snapshot cannot substitute for checkpoint content | `ADAPTER_CHECKPOINT_*` future taxonomy | future |
| Remote probe | Hub adapter | Transport may inspect its own Git/mailbox remote for transport safety only | Observes shared-checkpoint remote identity | may update `observed_sha` only under ADR-0003 observation semantics | Semantics-neutral low-level read helpers may later be shared | transport fetch/status or repository HEAD observation cannot update adapter `observed_sha` unless performed as a typed adapter probe against the checkpoint resource | `ADAPTER_PROBE_*` future taxonomy | future |
| `observed_sha` | Hub adapter | no meaning | Latest trusted observation of checkpoint state | observation only; not reconciliation | none from transport | Transport commit/blob/message/ref SHA must never be promoted to `observed_sha` by type matching | adapter observation results | future, ADR-0003 protocol-only |
| `reconciled_sha` / `B` | Hub adapter + guarded local canonical pair | no meaning | Last fully reconciled shared base | canonical reconciliation bookkeeping | none from transport | Receipt, snapshot, message delivery, transport sync or remote SHA cannot advance `B/reconciled_sha` | ADR-0003 finalization results | future, ADR-0003 protocol-only |
| `base_pair_token` | Hub adapter/local safe-write integration | no meaning | Guards canonical local reconciled-pair advancement | protects reconciliation bookkeeping | may reuse future low-level local safe-write helper only | Transport state/version/message id cannot act as base-pair token | local/adapter stale/finalization results | future, ADR-0002/0003 protocol-only |
| Shared candidate projection | Local Memory Core + later publication gate | Transport payload is not projection input merely because it was received | Adapter accepts only a separately produced shared-schema candidate | may become `L` only after local adoption/projection/policy | Validated handoff may change local canonical memory; any automatic transport→local→adapter bridge must also pass project-binding alignment | Direct transport→`L` passthrough or identity-by-path/channel is forbidden | later projection/privacy/binding result namespaces | future; M1.6+ |
| Publication | Semantically split | `publish` means publish a transport message to mailbox/channel | Future checkpoint publication means reconcile/commit an approved shared candidate | transport publish: none; adapter publication: checkpoint mutation under ADR-0003 | Higher-level orchestration may invoke each separately only with independent preconditions/results | Adapter unavailable must not fall back to transport `publish`; transport unavailable must not fall back to checkpoint write | future `TRANSPORT_PUBLISH_*` vs `ADAPTER_PUBLISH/RECONCILE_*` taxonomy | transport current; adapter future |
| Hydration | Hub adapter / recovery policy | Receive only supplies handoff proposals | Future hydration obtains shared checkpoint/recovery input under M1.7 | never auto-advances reconciled base merely by fetch | A hydrated shared fact may later be incorporated into local memory through defined reconciliation/adoption | `receive` or CURRENT snapshot cannot serve as hydration baseline | `ADAPTER_HYDRATE_*` / base-required/invalid outcomes | future M1.7/M3 |
| Conflict | Semantically split | Transport conflict means duplicate/hash/path/non-fast-forward/delivery problem | Adapter conflict means semantic B/R/L/identity/schema/resolution conflict | adapter conflicts block checkpoint advancement | Common UI may display both with typed source | Transport conflict resolution must not resolve ADR-0003 field conflict, and vice versa | separate typed conflict classes | transport current; adapter future |
| Session log / shared trace | Local core or explicit Hub shared-record layer | A handoff may reference a session but does not own session history | Future Hub shared session/trace is distinct from checkpoint and transport receipt | no automatic checkpoint effect | Explicit promoted trace may reference transport message IDs as provenance | receipt history must not be reclassified as Hub session evidence automatically | `LOCAL_SESSION_*` / future `HUB_TRACE_*` | local current; Hub trace protocol/future integration |
| Recovery | Semantically split | Retries/rediscovers transport messages and delivery state | Recovers checkpoint/base lineage only under ADR-0003 + later M1.7/M3 | adapter recovery may eventually restore trusted reconciliation bookkeeping | Shared diagnostic shell may report both | Transport snapshot/message history must not repair missing/corrupt `B/reconciled_sha`; adapter state must not repair mailbox delivery state | future `TRANSPORT_RECOVERY_*` vs `ADAPTER_BASE/RECOVERY_*` taxonomy | transport current; adapter future |
| Low-level Git/filesystem plumbing | Utility layer, no semantic authority | May use pull/push/hash/path/dirty-worktree helpers | May later use fetch/read/CAS/revision helpers if verified suitable | none by utility call alone | Reuse is allowed only through semantics-neutral helpers with typed callers/results **and** shared Git transaction-domain isolation | A shared helper must not stage/commit/push another subsystem's resources or map one subsystem's success to the other's success | utility errors wrapped by owning subsystem | implementation deferred |
| Configuration/capability discovery | Each subsystem independently + canonical binding authority separately | Transport config/version/capabilities describe mailbox layer; transport `project_id` is only a claimed/configured reference | Adapter config/version/capabilities describe checkpoint layer and must validate target identity against canonical binding/registry | none by discovery alone | Higher-level status may aggregate both after preserving typed identities | One generic `hub_enabled=true`, transport `project_id`, `hub_project_path`, channel name, or similar path/name hint is insufficient to establish adapter identity | typed configuration/capability/binding errors | transport current; adapter future |
| Remote resource ownership | Per resource class in section 3.4 | May write only transport mailbox resource classes | May write only adapter checkpoint/state classes under accepted semantics; normal reconciliation does not own registry/infrastructure | depends on exact owned class | Read-only opaque references may cross layers where policy permits | mixed-class write without an explicit composite contract is forbidden | owner-specific typed conflict/result | transport current; adapter future |

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
  only when resource and transaction-domain isolation are satisfied
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
Transport project_id/path/channel -> override canonical workspace/Hub binding
Transport or adapter -> stage/commit/push unowned cross-subsystem resources
```

If a higher-level orchestrator invokes both subsystems in one workflow, each operation retains its own typed result and commit meaning. "Both succeeded" may be reported only as a composite result; neither subsystem's success is evidence for the other's success.

## 7. Coexistence, capability identity, Git transaction domain, and project binding

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

A transport capability/configuration record may contain a `project_id` or similar identity claim, but that field is not an independent canonical project authority. Capability discovery and project-identity authority are separate concerns.

M1.5 does not fix the physical config keys or version numbers.

### 7.3 Shared Git transaction domain

Semantic namespaces do not isolate Git operational state. A worktree, index, HEAD, checked-out branch, repository refs, staged set, pending commits, and push ancestry form a shared or mutually coupled **transaction domain** whose exact storage granularity depends on the later Git topology. Two subsystems using different semantic paths can still interfere through that transaction domain.

If transport and adapter share a clone, worktree, branch, ref, or other Git operational state, every write-capable operation must be able to prove all of the following before it may claim subsystem success:

1. **Operation isolation.** The operation has one typed subsystem purpose and one declared owned-resource set. Repository-global state from an unrelated operation cannot be silently absorbed.
2. **Owned-path/resource commit scope.** The staged/committed resource set is exactly within the current operation's declared ownership class. Unowned transport/adapter/registry/infrastructure changes are not staged or committed opportunistically.
3. **No cross-subsystem staging/commit/push.** A transport operation cannot commit or push adapter changes, and an adapter operation cannot commit or push transport changes. Pending unrelated commits must not ride along with the current subsystem's push.
4. **Head/revision revalidation.** The relevant local HEAD/ref/revision assumptions and any remote revision preconditions are revalidated at the point where the operation depends on them. A previously observed clean/expected state is not assumed to remain valid indefinitely.
5. **Dirty/unknown/ahead state classification.** Unexpected staged entries, unowned dirty paths, ambiguous HEAD/ref state, unknown commit state, unexplained local-ahead commits, or other ancestry/state ambiguity cause a typed fail-closed result. They are not normalized by broad `add`, opportunistic commit, force/reset, or implicit push.
6. **Subsystem result integrity.** Git command success alone does not imply transport delivery success or adapter reconciliation success; the owning subsystem still has to satisfy its own semantic confirmation/finalization rules.

If these properties cannot be proven for a shared transaction domain, the implementation must use an isolated worktree/clone/ref arrangement or stop the operation. M1.5 deliberately does not choose which isolation topology, lock API, GitHub API, operating-system primitive, or retry count will implement this requirement.

Separate worktrees/clones/refs reduce some coupling but do not automatically transfer semantic authority; remote resource ownership and project-binding checks still apply.

### 7.4 Cross-layer project binding

A transport configuration, envelope, handoff, channel, snapshot, or receipt may carry a `project_id` or another project reference for exchange purposes. That value is a **claim/reference**, not an independent canonical project authority.

For any **automatic** workflow that crosses:

```text
transport input
  -> local adoption
  -> adapter shared projection/reconciliation target
```

the workflow must verify identity consistency among all three of these roles:

```text
W = canonical workspace binding / resolved stable project_id
T = transport-side project identity claim for the input, when required for the bridge
A = adapter target project identity resolved/validated through the Hub binding/routing authority
```

Automatic bridging is permitted only when the workflow can establish the required identities unambiguously and prove:

```text
W == T == A
```

for the same stable project identity.

Rules:

1. Missing required identity, ambiguous identity, or any mismatch returns a typed binding conflict such as `BINDING_IDENTITY_MISSING`, `BINDING_IDENTITY_AMBIGUOUS`, or `BINDING_IDENTITY_MISMATCH`; the automatic bridge stops.
2. A human/agent may still inspect a received handoff as a proposal, but inspection does not authorize automatic adapter projection until the binding conflict is resolved through the proper authority path.
3. `hub_project_path`, channel name, mailbox directory, workspace folder name, similar directory layout, remote URL similarity, or physical co-location are not valid substitutes for stable identity equality.
4. Transport config must not overwrite `AGENTS.md`/canonical workspace binding or Runtime Hub registry identity merely to make an automatic bridge succeed.
5. Adapter target selection must not be derived solely from transport routing configuration.
6. M1.5 does not define bootstrap/rebinding UX or recovery policy; it only requires fail-closed identity verification before automatic cross-layer bridging.

## 8. Failure and fallback contract

### 8.1 Result names in this proposal are interface taxonomy, not current-tool claims

Names such as `TRANSPORT_UNAVAILABLE`, `TRANSPORT_RECEIVE_*`, `ADAPTER_*`, and `BINDING_*` describe the **future unified interface taxonomy** required to preserve subsystem identity. They do not assert that the current `transport_tool.py` already returns these exact structured symbols or objects.

Current transport remains whatever its audited executable interface currently provides. A later unified implementation may wrap/translate current transport outcomes into the typed taxonomy without changing their semantic meaning.

### 8.2 No silent fallback

Examples:

```text
adapter unavailable
  -> future ADAPTER_UNAVAILABLE-class result
  -> NOT transport publish/snapshot fallback

transport unavailable
  -> future TRANSPORT_UNAVAILABLE-class result
  -> NOT checkpoint write fallback

adapter base invalid
  -> ADR-0003 base error
  -> NOT repair from mailbox snapshot

transport receive hash mismatch
  -> transport integrity error
  -> NOT adapter semantic conflict

binding mismatch
  -> BINDING_* conflict
  -> NOT path/channel-based identity guess

shared Git transaction domain ambiguous
  -> typed operational isolation failure
  -> NOT broad stage/commit/push
```

### 8.3 Composite workflows preserve per-subsystem results

If a future workflow does both:

```text
1. publish transport handoff
2. reconcile Hub checkpoint
```

then conceptually distinct outcomes include:

```text
transport=COMMITTED, adapter=FAILED
transport=FAILED, adapter=RECONCILED
transport=COMMITTED, adapter=RECONCILED
transport=UNKNOWN, adapter=NOT_ATTEMPTED
binding=CONFLICT, adapter=NOT_ATTEMPTED
```

The system must not flatten these into one ambiguous boolean or infer one operation from the other.

### 8.4 Unknown states remain local to their subsystem/domain

Transport unknown delivery/commit state does not make adapter reconciliation unknown unless a separately attempted adapter operation is itself unknown. Adapter `REMOTE_COMMIT_STATE_UNKNOWN` does not imply a transport message was or was not delivered. A shared Git transaction-domain ambiguity may block a new operation in either subsystem, but it does not retroactively reinterpret the semantic result of a previously confirmed operation.

## 9. Authority and provenance crossing rules

### 9.1 Transport-to-local adoption

Receiving a handoff produces a **proposal/evidence input**, not canonical truth.

Before durable local adoption, the local memory workflow may require direct validation, provenance classification, or explicit user/agent acceptance according to local policy. For an automatic cross-layer workflow, the project-binding checks in section 7.4 are also mandatory before the adopted result is allowed to feed an adapter projection.

Only after local adoption does the conclusion become eligible to influence a later shared projection.

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
12. current released Skill behavior is described as already using this future adapter;
13. a transport operation writes adapter checkpoint/state, Runtime Hub registry/routing, or shared infrastructure resources;
14. an ordinary adapter checkpoint operation writes transport mailbox, Runtime Hub registry/routing, or shared infrastructure resources outside its declared owned class;
15. a mixed-resource operation proceeds when resource ownership cannot be uniquely classified or when no accepted composite-operation contract exists;
16. transport and adapter share a Git transaction domain without proving operation isolation, owned-resource commit scope, no cross-subsystem staging/commit/push, and relevant HEAD/revision revalidation;
17. unknown dirty/staged/ahead/HEAD/ref/ancestry state is repaired by broad staging, opportunistic commit, force/reset, or implicit push rather than failing closed;
18. transport-side `project_id`, `hub_project_path`, channel name, directory similarity, or other routing hints override or substitute for canonical workspace/Hub project binding;
19. an automatic transport→local→adapter bridge proceeds when canonical workspace identity, transport identity, and adapter target identity are missing, ambiguous, or inconsistent;
20. future `TRANSPORT_*` result taxonomy is described as an already-implemented structured-result contract of the current `transport_tool.py`.

## 13. Initial review defects addressed by this revision

### B1 — Remote resource ownership

Addressed by section 3.4 and the corresponding capability/invariant changes. Remote resources are now partitioned into transport mailbox resources, adapter checkpoint/state resources, Runtime Hub registry/routing resources, and project-container/shared infrastructure resources. Each class has writer/reader boundaries, conflict semantics, and fail-closed conditions. Final physical paths remain deferred.

### B2 — Shared Git transaction domain

Addressed by section 7.3. Logical namespaces no longer imply Git operational isolation. A shared clone/worktree/ref is acceptable only if operation isolation, owned-resource scope, no cross-subsystem staging/commit/push, revision revalidation, and dirty/unknown/ahead fail-closed behavior can be proven. Otherwise the later implementation must isolate the Git transaction domain or stop.

### B3 — Cross-layer project binding

Addressed by sections 7.2, 7.4, and 9.1. Transport identity is explicitly non-canonical. Any automatic transport→local→adapter bridge requires consistent stable identity across canonical workspace binding, transport claim, and adapter target; missing/ambiguous/mismatched identity blocks the bridge with a typed binding conflict. Paths/channels/directories cannot substitute for identity.

These are proposal-level closures pending final-review confirmation; they do not claim executable implementation or live validation.

## 14. Decisions requested for M1.5 final review

Final review should approve or revise these decisions:

- **TB1:** existing transport and future Hub adapter are independent semantic subsystems with different authority and lifecycle. **Initial-review disposition: APPROVE; unchanged in intent.**
- **TB2:** transport owns mailbox/handoff/receipt/non-canonical snapshot semantics and its transport resource classes only; adapter owns shared-checkpoint observation/reconciliation/finalization and adapter resource classes only; Runtime Hub registry/routing and shared infrastructure remain separately owned resource classes rather than generic adapter storage. **Revised for B1.**
- **TB3:** transport state and adapter state use independent logical namespaces, equal-looking IDs/SHAs never imply semantic equivalence, and namespace separation does not by itself establish remote-resource ownership, Git transaction isolation, or project identity. **Revised for B1/B2/B3.**
- **TB4:** transport receive produces only a handoff/proposal input; it cannot advance `B`, `observed_sha`, `reconciled_sha`, or adapter finalization state. **Initial-review disposition: APPROVE; unchanged in intent.**
- **TB5:** transport receipt/ack proves message lifecycle only and cannot confirm checkpoint CAS, exact revision, or reconciliation. **Initial-review disposition: APPROVE; unchanged in intent.**
- **TB6:** transport CURRENT snapshot remains non-canonical and cannot serve as checkpoint/hydration/recovery base. **Initial-review disposition: APPROVE; unchanged in intent.**
- **TB7:** transport `publish` and future shared-checkpoint publication are distinct operations; neither is a fallback for the other. **Initial-review disposition: APPROVE; unchanged in intent.**
- **TB8:** semantics-neutral Git/filesystem plumbing may be reused only with typed callers/results, explicit remote-resource ownership, and proven shared Git transaction-domain isolation; a shared helper must not stage/commit/push another subsystem's resources or translate one subsystem's success into another's semantic success. **Revised for B2.**
- **TB9:** adapter unavailable/failed/unknown and transport unavailable/failed/unknown remain separate typed outcomes with no silent fallback; symbolic `TRANSPORT_*` names in this proposal are future unified-interface taxonomy, not claims about current `transport_tool.py` output. **Initial-review disposition: APPROVE; clarified without changing intent.**
- **TB10:** received transport content may reach Hub only through validate/adopt into local canonical memory, then later explicit shared projection/publication; any automatic bridge must also prove canonical workspace binding, transport identity, and adapter target identity agree; direct transport-to-checkpoint passthrough is forbidden. **Revised for B3.**
- **TB11:** local session history, Hub shared trace, transport receipt history, and shared checkpoint are separate record classes; references may connect them without transferring authority. **Initial-review disposition: APPROVE; unchanged in intent.**
- **TB12:** capability/config discovery must identify `transport` and `hub_adapter` independently, while project identity is validated through the separate canonical binding/routing authority; transport config/envelope `project_id`, a generic untyped `hub enabled` state, `hub_project_path`, channel name, or directory similarity is insufficient to establish adapter target identity. **Revised for B3.**
- **TB13:** physical config/worktree co-location is permitted only if logical namespaces remain typed **and** any shared Git transaction domain can prove operation isolation, owned-resource commit scope, no cross-subsystem staging/commit/push, head/revision revalidation, and fail-closed handling of unknown/dirty/ahead state; otherwise use an isolated topology or stop. **Revised for B2.**
- **TB14:** existing transport remains current executable behavior until an explicit compatibility/release decision; M1.5 does not turn `transport_tool.py` into the adapter. **Initial-review disposition: APPROVE; unchanged in intent.**
- **TB15:** M1.5 remains architecture/protocol-only and does not define adapter code, API/credentials, publication triggers, executable privacy gates, M1.7 recovery workflow, migration, or real-data movement. **Initial-review disposition: APPROVE; unchanged in intent.**

## 15. Deferred implementation choices

Deliberately deferred:

- adapter implementation language/module layout;
- exact GitHub/remote API and credentials;
- exact adapter config/state filenames;
- exact physical remote paths for the ownership classes in section 3.4;
- whether transport and adapter use separate or shared Git clones/worktrees/refs;
- exact locking/serialization primitive for a shared Git transaction domain;
- exact common Git helper interface;
- concrete capability/version negotiation fields;
- publication trigger policy;
- M1.6 classification implementation;
- M1.7 startup/hydration/recovery workflow;
- migrations and compatibility code;
- retry/backoff values and UI wording.

## 16. Evidence

- `docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md`
- `docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md`
- `docs/decisions/ADR-0002-local-multi-session-safe-write.md`
- `docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md`
- `skill/SKILL.md` and release metadata for current `0.1.0` behavior
- M1.5 initial review at `main@0b7cebe64b26f91df40be23e44fa2b786e6a2655`

This proposal is an M1.5 boundary specification only. Approval must not be interpreted as adapter implementation, runtime effectiveness, migration completion, or validation.