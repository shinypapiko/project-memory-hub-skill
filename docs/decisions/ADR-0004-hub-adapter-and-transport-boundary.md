---
document_role: architecture-decision-record
adr_id: ADR-0004
status: accepted
accepted: 2026-08-27
milestone: M1.5
runtime_effective: false
release_required_for_runtime_effect: true
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
proposal: docs/proposals/HUB_ADAPTER_TRANSPORT_BOUNDARY.md
final_approval: docs/reviews/2026-08-27-hub-adapter-transport-boundary-final-approval.md
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
  - docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md
---

# ADR-0004 — Hub adapter and existing transport boundary

## Status

**Accepted architecture decision.**

This ADR records the approved M1.5 semantic boundary between the existing executable Git transport/mailbox subsystem and the future Hub adapter in the unified Project Memory architecture.

It is normative for subsequent unified-architecture design and implementation work, but it does **not** change current released Hub Skill `0.1.0`, current deployed local Project Memory behavior, Runtime Hub runtime behavior, or existing `transport_tool.py` semantics. A later implementation, compatibility/migration plan, validation suite, and release are required before these semantics become runtime-effective.

Current maturity remains:

```text
runtime_effective: false
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
```

## Context

The audited local Project Memory distribution already contains an executable Git-backed transport layer for mailbox/handoff exchange, receipt/ack lifecycle, transport bookkeeping, and an optional non-canonical CURRENT snapshot. Separately, ADR-0001 and ADR-0003 require a future Hub adapter for minimized shared-checkpoint observation, reconciliation, CAS, confirmation, and guarded local finalization.

Because both layers may touch Git, remote resources, project identity, and cross-agent information, implementation could accidentally merge their authority if the boundary were left implicit. The M1.5 initial review identified three blocking defects: incomplete remote resource ownership, incomplete shared Git transaction-domain isolation, and incomplete cross-layer project-binding rules. The revised proposal closed all three; final review approved TB1–TB15 with no blocking defects.

## Decision

TB1–TB15 from `docs/proposals/HUB_ADAPTER_TRANSPORT_BOUNDARY.md` are accepted in full.

### 1. Transport and Hub adapter are separate semantic subsystems

The existing transport owns mailbox/exchange lifecycle semantics such as handoffs/messages, receipt/ack, transport delivery/bookkeeping, and optional non-canonical CURRENT snapshots.

The future Hub adapter owns shared-checkpoint observation/reconciliation/finalization semantics such as checkpoint probe/fetch, adapter sync metadata, `observed_sha`, canonical `B/reconciled_sha/base_pair_token` integration, ADR-0003 reconciliation/CAS/confirmation/finalization, and structured checkpoint conflicts.

Code or utility reuse never transfers authority from one subsystem to the other.

The following remain non-equivalent:

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

### 2. Logical state namespaces are independent

Transport state and adapter state require separate typed logical namespaces. Equal-looking values such as project IDs, paths, or Git SHAs do not imply semantic equivalence.

Existing `transport.json` remains transport-only unless a later explicit compatibility design changes its representation. Transport state must not reconstruct adapter reconciliation state, and adapter recovery must not overwrite transport lifecycle state.

Namespace separation alone does not prove remote-resource ownership, Git transaction isolation, or project identity.

### 3. Remote resources have explicit semantic ownership classes

Before a write-capable operation proceeds, each touched remote resource must belong unambiguously to an ownership class. The accepted classes include:

- transport mailbox/channel/message/index/receipt/snapshot/delivery resources;
- adapter checkpoint resources;
- adapter-specific state/coordination resources;
- Runtime Hub registry/routing resources;
- project-container/shared infrastructure such as Hub metadata, schema/bootstrap markers, compatibility metadata, and attributes.

Transport writes only transport-owned resource classes. Ordinary checkpoint reconciliation writes only adapter checkpoint/state classes under its accepted semantics. Runtime Hub registry/routing and shared infrastructure are separately governed and are not generic adapter storage merely because the adapter reads them.

If ownership cannot be classified, overlaps unexpectedly, or a proposed operation spans independently owned classes without an accepted composite-operation contract, the operation fails closed.

M1.5 does not choose final physical paths for these classes.

### 4. Shared Git operational state is a transaction domain, not isolated by semantic paths

A Git worktree, index, HEAD, checked-out branch, refs, staged set, pending commits, ancestry, and push state may couple transport and adapter operations even when their semantic files are separate.

If transport and adapter share a clone/worktree/branch/ref or otherwise share Git operational state, a write-capable operation must prove:

- one typed subsystem purpose and declared owned-resource set;
- exact owned-resource staging/commit scope;
- no cross-subsystem staging, commit, or push;
- no unrelated pending commit riding along in a push;
- relevant HEAD/ref/revision assumptions revalidated at the point of use;
- unexpected dirty/staged/ahead/unknown/ancestry state fails closed;
- Git command success alone is not treated as transport delivery or adapter reconciliation success.

If those properties cannot be proven, the later implementation must isolate the transaction domain, for example through a separate worktree/clone/ref topology, or stop the operation. This ADR does not select the concrete isolation mechanism, lock/API primitive, operating-system function, or retry value.

### 5. Cross-layer project identity has one canonical authority path

A `project_id` or similar value appearing in transport configuration, an envelope, channel metadata, handoff, snapshot, or receipt is a claim/reference, not an independent canonical project authority.

For any automatic workflow crossing:

```text
transport input
  -> local adoption
  -> adapter projection/reconciliation target
```

the workflow must establish consistent stable identity across:

```text
W = canonical workspace binding / resolved stable project_id
T = transport-side project identity claim required for the bridge
A = adapter target identity validated through canonical Hub binding/routing
```

Automatic bridging requires the relevant identities to be present, unambiguous, and equal for the same project. Missing, ambiguous, or mismatched identity produces a typed binding conflict and stops the bridge.

`hub_project_path`, channel name, mailbox directory, workspace directory name, remote URL similarity, or physical co-location cannot substitute for stable project identity. Transport configuration must not rewrite canonical workspace binding or Runtime Hub registry identity merely to make a bridge succeed, and adapter target selection must not be derived solely from transport routing configuration.

### 6. Transport-to-Hub flow must cross local canonical adoption and later projection

Receiving a handoff yields a proposal/evidence input, not canonical local truth and not a Hub shared candidate.

The allowed conceptual path is:

```text
transport message
  -> validate / inspect
  -> local canonical adoption if warranted
  -> later minimized shared projection
  -> Hub adapter reconciliation/publication path
```

Direct transport message/receipt/snapshot → `B`, `R`, `L`, `C`, hydration base, `observed_sha`, `reconciled_sha`, or adapter finalization state is forbidden.

Automatic cross-layer use must also pass the canonical binding checks above.

### 7. Receipt, snapshot, publication, hydration, conflict, trace, and recovery remain typed by subsystem

- Receipt/ack proves only transport message lifecycle state.
- CURRENT snapshot remains non-canonical.
- Transport `publish` and future checkpoint publication are distinct operations and are never fallbacks for one another.
- Transport receive is not Hub hydration.
- Transport conflicts and ADR-0003 reconciliation conflicts are different classes.
- Local sessions, Hub shared trace, transport receipt history, and shared checkpoints are separate record classes; references do not transfer authority.
- Transport recovery cannot repair missing/corrupt adapter `B/reconciled_sha`, and adapter recovery cannot repair mailbox delivery state.

### 8. Result/capability identity is typed, without rewriting current executable behavior

Future unified interfaces must preserve subsystem identity in capability discovery, success/failure/unknown outcomes, and composite workflows.

Names such as `TRANSPORT_*`, `ADAPTER_*`, and `BINDING_*` in the proposal are taxonomy for the future unified interface. They are **not** a claim that the current executable `transport_tool.py` already emits those exact structured result types or symbolic names.

A future wrapper may translate current transport outcomes into typed unified results without changing their semantic meaning.

### 9. No silent fallback or authority repair across layers

Adapter unavailable/failed/unknown must not fall back to transport publish or snapshot. Transport unavailable/failed/unknown must not fall back to checkpoint mutation. Missing/corrupt adapter lineage must not be repaired from mailbox snapshot/history. Identity conflicts must not be repaired by path/channel guesses.

If a higher-level workflow invokes both subsystems, it preserves each subsystem's independent result. One subsystem's success is not evidence of the other's success.

### 10. Compatibility remains explicit

The existing transport remains current executable behavior until a later explicit release/compatibility decision retains, deprecates, migrates, or removes it.

M1.5 does not rename or upgrade `transport_tool.py` into the Hub adapter. Existing transport state must never be silently reinterpreted as adapter reconciliation state.

## Consequences

### Positive

- Existing mailbox/handoff functionality can coexist with a future Hub adapter without becoming a second reconciliation authority.
- A non-canonical transport snapshot cannot silently become a shared-checkpoint base.
- Shared Git plumbing can be reused later only when operation/resource scope remains provably isolated.
- Project identity cannot fork into separate transport and adapter authorities.
- Runtime Hub registry/infrastructure stays distinct from ordinary checkpoint mutation.
- Future interfaces can report composite outcomes without flattening transport and adapter success/failure together.

### Costs / constraints

- A future implementation must track typed resource ownership and project identity explicitly.
- Sharing one Git worktree/ref between transport and adapter requires provable transaction isolation or an isolated topology.
- Higher-level automatic bridges require binding verification before data can cross from transport input toward adapter projection.
- Existing transport cannot be reused as an adapter merely by renaming commands or treating existing SHAs/snapshots/receipts as equivalent metadata.

## Rejected alternatives

- Treat the existing Git transport as the Hub adapter because it already supports push/pull/sync.
- Treat transport CURRENT snapshot as shared-checkpoint `B` or hydration/recovery base.
- Treat receipt/ack as CAS or reconciliation confirmation.
- Treat transport remote SHA as `observed_sha` or `reconciled_sha` by type matching.
- Let logical namespace separation stand in for Git transaction isolation.
- Broad-stage or opportunistically commit/push unrelated subsystem changes from a shared worktree.
- Infer project identity from channel/path/directory/remote similarity.
- Let transport configuration become an independent canonical project authority.
- Let adapter failure silently fall back to transport publication or snapshots.
- Let ordinary checkpoint reconciliation modify Runtime Hub registry/routing or infrastructure state.

## Explicitly deferred

This ADR does not decide or implement:

- M1.6 privacy/classification implementation;
- M1.7 startup/recovery policy;
- exact publication triggers;
- migration;
- concrete Git/GitHub APIs or credential mechanisms;
- lock or Windows primitives;
- retry counts;
- final physical resource paths;
- exact clone/worktree/ref topology;
- adapter implementation code;
- real project/runtime data operations.

## Evidence and review

- `docs/audits/2026-08-local-project-memory-v1.1-audit-summary.md`
- `docs/proposals/HUB_ADAPTER_TRANSPORT_BOUNDARY.md`
- `docs/reviews/2026-08-27-hub-adapter-transport-boundary-final-approval.md`
- `docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md`
- `docs/decisions/ADR-0002-local-multi-session-safe-write.md`
- `docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md`

The final review baseline was commit `7ce8b596b225a84e7247315210adc17587394ffa`, proposal SHA-256 `8B32662148AE134610DB946526375F556BB1E4A82D36C372051AC05F5E3E602F`. It approved TB1–TB15, closed B1–B3, found no blocking defects, and approved M1.5 for ADR promotion.
