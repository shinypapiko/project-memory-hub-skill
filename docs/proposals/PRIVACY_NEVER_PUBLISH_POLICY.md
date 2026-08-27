---
document_role: architecture-proposal
status: approved-proposal
normative: false
architecture_state: accepted-unreleased
runtime_load_policy: maintenance-only
milestone: M1.6
created: 2026-08-27
last_reviewed: 2026-08-27
review_state: final-approved-promoted-to-adr
initial_review_baseline: 1ef9f4b69d699e79cf02314787acb10d82d4c303
final_review_baseline: 1ef9f4b69d699e79cf02314787acb10d82d4c303
reviewed_proposal_git_blob_lf_sha256: 91916BE03E68AEE652265CC33761D0FB375C2A2E1D3ED26DAE8DC85760C4CF69
final_approval: docs/reviews/2026-08-27-privacy-never-publish-policy-final-approval.md
approved_by: docs/decisions/ADR-0005-privacy-never-publish-policy.md
depends_on:
  - docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md
  - docs/decisions/ADR-0002-local-multi-session-safe-write.md
  - docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md
  - docs/decisions/ADR-0004-hub-adapter-and-transport-boundary.md
---

# Privacy / never-publish classes — M1.6

## 0. Purpose and strict scope

This proposal defines deterministic privacy, classification, and publication-gate semantics for the future unified Project Memory architecture.

It refines the already-accepted privacy boundary in ADR-0001 without reopening ADR-0001 through ADR-0004. In particular, ADR-0001 already establishes that `UNCLASSIFIED` is local-only until classification completes, P3 `PROHIBITED_SYNC` hard-blocks publication, explicit prohibitions cannot be weakened, the most restrictive applicable privacy rule wins, and a blocked candidate cannot be made publishable by silently deleting prohibited fragments from that same candidate.

M1.6 turns those accepted principles into a deterministic decision model for candidate construction and remote publication eligibility.

This proposal is intentionally limited to:

- privacy classes and their semantics;
- destination-aware allow/deny behavior;
- classification inheritance across records, fields, attachments, provenance, candidates, and publication bundles;
- promotion/minimization/reclassification rules;
- exact-context privacy verdicts and invalidation on drift;
- whole-bundle preflight before any remote side effect;
- typed allow/block/stale/conflict/derivative-required outcomes;
- the boundary between privacy blocking and retention/deletion semantics;
- the ordering constraint between privacy approval and write-capable reconciliation/CAS.

This proposal does **not** define or implement:

- M1.7 startup, hydration, recovery, or degraded-mode behavior;
- publication triggers or milestone-selection policy;
- Hub adapter implementation;
- privacy detector/scanner/classifier code;
- Git/GitHub API selection;
- credentials handling implementation;
- migration;
- real project/runtime data operations;
- secure deletion implementation;
- purge execution mechanisms;
- concrete hashing algorithm, serialization format, storage path, API object shape, or retry count.

No current released Skill, Runtime Hub, local Project Memory implementation, transport tool, or real data is modified by this proposal.

## 1. Core model

Privacy is evaluated against an **exact candidate** and an **exact destination context**. A class is not a universal statement that an object is always publishable or always unpublishable to every destination.

The deterministic gate evaluates:

```text
PrivacyDecision = evaluate(
    exact_candidate_identity,
    effective_classification_manifest,
    applicable_policy_context,
    exact_destination,
    permitted_provenance_manifest,
    publication_bundle_identity
)
```

A prior decision is reusable only while every bound input remains unchanged and valid.

The model has five classification states:

```text
P0 STRUCTURAL
P1 RUNTIME_SHAREABLE
P2 LOCAL_BY_DEFAULT
P3 PROHIBITED_SYNC
UNCLASSIFIED
```

`UNCLASSIFIED` is a state, not an implicit low-risk class.

## 2. Privacy classes

### 2.1 P0 — STRUCTURAL

`P0 STRUCTURAL` is material intentionally classified as non-project-sensitive structural information that may be eligible for broad publication when the destination policy permits it.

Typical semantic roles can include reusable schema structure, protocol names, generic field definitions, version/compatibility structure, public documentation structure, and synthetic fixtures. Merely being metadata, a path-like string, a hash, a filename, a field name, or a small object does **not** automatically make content P0.

P0 rules:

1. P0 may be eligible for an authorized private Runtime Hub.
2. P0 may be eligible for the public Skill source only when the object and all included context/provenance are independently classified as public-source-safe under the applicable policy.
3. A P0 parent cannot weaken a more restrictive child field, attachment, provenance item, or contextual prohibition.
4. Real project content does not become P0 merely because it has been summarized or moved into a structural-looking field.
5. Public-source eligibility is a destination decision, not an intrinsic consequence of the P0 label alone.

### 2.2 P1 — RUNTIME_SHAREABLE

`P1 RUNTIME_SHAREABLE` is material explicitly classified as eligible for a specifically authorized private Runtime Hub or equivalent authorized private runtime destination within its project/scope boundary.

P1 rules:

1. P1 is not public-source permission.
2. P1 may be published only to an exact authorized private destination that matches the applicable project/binding/scope policy.
3. P1 content remains subject to more restrictive nested classifications and destination restrictions.
4. P1 does not authorize reuse in a different project, different private Hub, public source repository, transport channel, or other destination merely because those destinations are remote or authenticated.
5. A previously allowed P1 decision is invalid if destination identity, authorization scope, binding, policy, candidate contents, or permitted provenance changes.

### 2.3 P2 — LOCAL_BY_DEFAULT

`P2 LOCAL_BY_DEFAULT` is material whose authoritative/default lifecycle remains local and which is **not directly publishable** as P2.

P2 rules:

1. P2 may remain in local canonical memory/evidence according to local ownership and retention policy.
2. P2 requires an **explicit promotion operation** before any remote publication can be considered.
3. Promotion must produce a minimized candidate or derivative rather than treating the local source object itself as remotely shareable.
4. The promoted/minimized object must be classified independently for its actual contents and destination. For an authorized private Runtime Hub it must become an independently valid P1 or P0 object; for a public Skill source it must become an independently valid public-source-safe P0 object.
5. Reclassification is not a relabel-only operation. The new object must have its own exact identity, provenance decision, and privacy evaluation.
6. If the proposed remote payload still contains P2 material, the gate returns a derivative/promotion-required outcome rather than `ALLOW`.

### 2.4 P3 — PROHIBITED_SYNC

`P3 PROHIBITED_SYNC` is material explicitly prohibited from Project Memory remote publication/synchronization.

P3 rules:

1. P3 is a hard block for the publication transaction/bundle in which it appears.
2. P3 cannot be weakened by a field override, record override, destination default, broader parent classification, user-interface convenience, summary operation, transport receipt, existing remote copy, or successful prior publication of some related object.
3. P3 may remain in its owning local source subject to local policy; publication blocking is not source deletion.
4. A blocked candidate cannot be made publishable by silently deleting or redacting the P3 fragment and continuing under the same candidate/privacy decision.
5. If a safe result can be derived, it must be a **new independently derived object** with a new identity and a complete new classification/privacy evaluation.
6. P3 provenance, attachment metadata, path/name data, logs, hashes, or error details block the exact candidate/bundle if they are included in or required by that publication object.

### 2.5 UNCLASSIFIED

`UNCLASSIFIED` means classification has not been completed or cannot be trusted for the exact object/context.

Rules:

1. `UNCLASSIFIED` is local-only for publication purposes.
2. Any included `UNCLASSIFIED` component hard-blocks the publication candidate/bundle.
3. No default destination policy may reinterpret `UNCLASSIFIED` as P0, P1, or P2.
4. Classification uncertainty, missing classification lineage, incompatible policy versions, or inability to determine the effective class produces an `UNCLASSIFIED` or classification-conflict block rather than an optimistic allow.
5. Completing classification creates a new valid classification context; it does not retroactively convert an earlier blocked verdict into an allow verdict.

## 3. Destination model and allow/deny matrix

M1.6 distinguishes at least these destination classes:

```text
LOCAL
  local workspace / local canonical memory / local evidence domain

AUTHORIZED_PRIVATE_RUNTIME_HUB
  exact authorized private Runtime Hub project/scope destination

PUBLIC_SKILL_SOURCE
  public reusable Skill source/specification/test/documentation repository
```

A transport mailbox remains a separate M1.5 subsystem/destination. Its existence, receipt history, or remote copy does not grant permission for Runtime Hub or public-source publication. M1.6 does not redefine current transport runtime behavior.

### 3.1 Destination-aware matrix

| Class | Local storage/use | Authorized private Runtime Hub | Public Skill source |
| --- | --- | --- | --- |
| **P0 STRUCTURAL** | ALLOW subject to local policy | ALLOW if destination/binding/policy context is valid | ALLOW only if independently public-source-safe and no more restrictive component applies |
| **P1 RUNTIME_SHAREABLE** | ALLOW | ALLOW only for the exact authorized private destination/scope | BLOCK |
| **P2 LOCAL_BY_DEFAULT** | ALLOW | `DERIVATIVE_REQUIRED`: explicit promotion + minimization + independent reclassification to valid P1/P0 | `DERIVATIVE_REQUIRED`: explicit promotion + minimization + independent reclassification to public-source-safe P0 |
| **P3 PROHIBITED_SYNC** | local retention/use governed separately | HARD BLOCK | HARD BLOCK |
| **UNCLASSIFIED** | may remain local pending classification | HARD BLOCK | HARD BLOCK |

Rules:

1. The matrix defines publication eligibility, not retention or deletion behavior.
2. Authorization of a destination is necessary but never sufficient: content classification, binding, provenance, policy, and bundle state must also pass.
3. Public Skill source is a stricter destination than an authorized private Runtime Hub. Real private/runtime project material must not enter public source merely because it is valid P1 for a private Hub.
4. A private repository or authenticated remote is not automatically an `AUTHORIZED_PRIVATE_RUNTIME_HUB`; the exact destination must be classified/recognized by the applicable policy.
5. A candidate allowed for one destination has no carry-over permission to another destination.

## 4. Classification units and inheritance

Privacy evaluation applies to all semantically included publication material, not only the visible main Markdown body.

The classification model covers:

- record;
- field;
- attachment;
- provenance item/reference;
- candidate;
- publication bundle;
- path/name metadata;
- content hashes and identifiers;
- logs and diagnostic/error details included in an outbound object.

### 4.1 Record and field inheritance

A record may define a default classification, but individual fields may have more restrictive classifications or explicit prohibitions.

For each included field:

```text
effective_field_policy = combine(
    system/policy defaults,
    record policy,
    field policy,
    destination policy,
    contextual prohibitions
)
```

The combination rule is monotonic toward restriction:

- a child/specific rule may strengthen restriction;
- an explicit prohibition cannot be weakened;
- a less restrictive child rule cannot override a more restrictive applicable parent/context rule;
- if two applicable rules cannot be deterministically reconciled, evaluation returns a classification conflict rather than choosing the permissive interpretation.

### 4.2 Attachments

Attachments are independently classified objects. An attachment does not inherit publication permission merely because its parent record is allowed.

If an attachment is included in the candidate/bundle, its classification participates in the effective decision. A blocked attachment blocks the bundle unless a **new candidate/bundle** is intentionally derived without that attachment and then independently evaluated. The existing blocked candidate is not mutated in place and resumed.

### 4.3 Provenance

Provenance is privacy-bearing data.

The evaluation includes all provenance material that would be emitted or is semantically required by the candidate, including where applicable:

- source record identifiers;
- path fragments;
- filenames;
- content hashes;
- commit/revision identifiers;
- experiment/session/decision identifiers;
- transport-message references;
- remote URLs or repository identifiers;
- timestamps or contextual labels;
- logs/error excerpts or diagnostic metadata.

A candidate may carry only **permitted provenance** for its exact destination. If required provenance is prohibited or unclassified, publication blocks. The system must not silently remove required provenance to obtain an allow verdict.

If provenance can be safely transformed, the transformed provenance and candidate constitute a new derivative context that must be independently classified.

### 4.4 Candidate classification

A publication candidate is a distinct object from its local source records.

Its effective privacy state is computed from:

- candidate-level classification;
- all included fields;
- all included attachments;
- all included/required provenance;
- applicable source/record prohibitions;
- destination-specific policy;
- classification/policy lineage.

A candidate-level P0/P1 label cannot override a P2/P3/UNCLASSIFIED nested component or an explicit source-level never-publish restriction that is applicable to the included material.

### 4.5 Bundle classification

A publication bundle is the exact set of remote objects that one logical publication operation intends to cause as remote side effects.

Examples may conceptually include a checkpoint candidate plus separately emitted permitted provenance/trace records. M1.6 does not decide which object types a future publication trigger will bundle.

The bundle decision is fail-closed:

```text
bundle_allow = every included remote-effect object is individually allowed
               AND every cross-object invariant is valid
               AND no required provenance/component is blocked
```

One blocked or unresolved member blocks the entire bundle before the first remote side effect.

## 5. Most-restrictive-applicable-rule semantics

Privacy rules are destination-aware and need not form one global numeric ordering. Determinism is achieved by evaluating all applicable rules for the exact destination and selecting the most restrictive resulting action.

For a given candidate/destination:

```text
HARD_BLOCK > DERIVATIVE_REQUIRED > ALLOW
```

`STALE` and `CONFLICT` are non-allow terminal outcomes for the current evaluation and require re-evaluation or resolution.

Rules:

1. If any applicable rule yields P3/never-publish block, the result is blocked.
2. If any included component is `UNCLASSIFIED`, the result is blocked.
3. If no hard block exists but included material is still P2 for the remote destination, the result is `DERIVATIVE_REQUIRED`.
4. `ALLOW` exists only when every included component and required provenance is allowed for the exact destination.
5. Explicit never-publish rules are monotonic and cannot be weakened by more specific fields, lower layers, later summaries, or destination defaults.
6. Privacy ambiguity fails closed; the evaluator does not select a permissive interpretation merely to complete publication.

## 6. Promotion, minimization, and safe derivatives

### 6.1 P2 promotion

P2 content may influence a remote shared object only through an explicit promotion workflow conceptually shaped as:

```text
P2 local source
    -> explicit promotion intent
    -> minimization / semantic projection
    -> new candidate object
    -> independent classification for destination
    -> privacy gate
```

The new candidate does not inherit remote permission from the local source. It receives permission only from its own classification and applicable policy.

### 6.2 No silent redact-and-continue

For an already materialized candidate `C`:

```text
C contains blocked P3 or UNCLASSIFIED material
    -> C is BLOCKED
```

The publication operation must not do this:

```text
C
  -> silently delete blocked fragment
  -> keep same candidate/privacy context
  -> publish remainder
```

That behavior is prohibited even if the remaining text would appear harmless.

### 6.3 Safe derivative

A safe derivative `D` must be treated as a new object:

```text
D.identity != C.identity
```

It must have:

- independently derived content;
- independently evaluated provenance;
- independently established classification;
- an exact destination;
- a new privacy decision context;
- no assumption that the source candidate's allow/block status transfers.

The existence of a safe derivative does not change or delete the blocked source/candidate.

## 7. Exact privacy decision context

Every reusable privacy result must bind to an immutable evaluation context sufficient to prove exactly what was approved or blocked.

Conceptually:

```text
PrivacyContext {
    candidate_identity
    candidate_schema/semantic_identity
    classification_manifest_identity
    policy_context_identity
    destination_identity_and_class
    permitted_provenance_manifest_identity
    publication_bundle_identity
    relevant_binding/scope_identity
}
```

M1.6 does not select the storage format or hashing algorithm. The requirement is semantic exactness: the system must be able to determine whether the object/context being sent is the same object/context that was evaluated.

### 7.1 Candidate identity

`candidate_identity` identifies the exact outbound candidate contents and semantically relevant metadata. A content or metadata change that affects the outbound object produces a different identity or otherwise invalidates the verdict.

### 7.2 Policy/classification context identity

The verdict is bound to the policy and classification inputs used to produce it. Changes to:

- privacy policy revision;
- classification manifest;
- field/record rule inheritance;
- explicit never-publish markers;
- destination policy;
- applicable binding/scope rules;

invalidate the old verdict.

### 7.3 Destination identity

A verdict for one destination cannot be replayed against another destination, even when both are private repositories or share similar paths/names.

Destination identity must be sufficient to distinguish the exact authorized private Runtime Hub/project/scope or public Skill source target under later implementation.

### 7.4 Permitted provenance identity

The exact set of provenance that may accompany the candidate is part of the verdict. Adding, changing, or substituting provenance invalidates the old result and requires re-evaluation.

## 8. Drift and verdict invalidation

An old privacy result becomes invalid when any bound input drifts.

At minimum, the following invalidate an earlier allow/block decision for reuse:

- candidate content/metadata drift;
- candidate schema/semantic identity drift;
- classification drift;
- policy revision/context drift;
- destination identity/class/authorization drift;
- project/binding/scope drift;
- permitted provenance drift;
- publication bundle membership/order/semantic-role drift where relevant.

A stale verdict returns a typed stale outcome and must not be treated as an allow.

```text
old ALLOW + any bound drift
    -> PRIVACY_STALE
    -> recompute classification/evaluation
```

This rule also applies after reconciliation recomputes a candidate. A new B/R/L-derived candidate cannot inherit the privacy verdict of the old candidate merely because the local intent is similar.

## 9. Publication-bundle preflight

### 9.1 Entire bundle before first side effect

Before the first remote side effect of a logical publication operation, the complete intended publication bundle must be known well enough to run privacy preflight over all outbound objects and required provenance.

Required ordering:

```text
construct exact bundle
    -> classify/evaluate every member + provenance
    -> validate destination/binding context
    -> produce bundle privacy verdict
    -> only if ALLOW: enter write-capable remote path
```

If any member is blocked, stale, conflicting, derivative-required, or unclassified, **no member of that publication bundle may be remotely written by that attempt**.

This is a privacy preflight requirement, not a claim that later multi-resource remote writes are transactionally atomic. M1.6 does not redefine ADR-0003 remote commit semantics.

### 9.2 First remote side effect

Read-only probe/fetch/validation needed to understand destination or remote state is not itself a publication side effect. Creating/updating remote publication objects, checkpoint content, shared trace/provenance objects, or equivalent remote state is a side effect and must occur only after bundle privacy `ALLOW`.

## 10. Relationship to ADR-0003 reconciliation/CAS

Privacy approval gates entry into the **write-capable** reconciliation/CAS path.

The ordering is:

```text
local source / explicit promotion
    -> exact minimized candidate
    -> privacy classification + bundle preflight
    -> PRIVACY_ALLOW for exact candidate/bundle/destination
    -> write-capable reconciliation/CAS/finalization path
```

Rules:

1. Ordinary remote observation/read-only refresh may occur before privacy approval when needed for state awareness.
2. No candidate may enter a write-capable CAS/publication attempt unless the exact candidate/bundle has a current `PRIVACY_ALLOW` verdict for the exact destination.
3. If ADR-0003 stale-CAS handling forces refresh and recomputation of `C`, the new candidate identity invalidates the old privacy verdict. The recomputed candidate must pass privacy evaluation again before a new write attempt.
4. An ADR-0003 conflict resolution that changes candidate contents similarly requires a new privacy evaluation.
5. Remote CAS success does not retroactively validate privacy. Privacy approval must precede the side effect; a missing/stale privacy verdict is not repairable after publication by simply recording a later allow.
6. Privacy approval does not replace ADR-0003 concurrency, exact-revision confirmation, or guarded local finalization. Both layers must independently pass their own contracts.

## 11. Transport mailbox and existing remote presence

M1.5 already separates transport mailbox state from Hub adapter reconciliation state. M1.6 adds the corresponding permission rule:

```text
already remote somewhere != authorized for this publication destination
```

Therefore:

1. A transport mailbox message, receipt, snapshot, channel record, or prior transport publication does not grant permission to publish the same or derived material to Runtime Hub or public Skill source.
2. Existing presence of data in a Runtime Hub, repository history, cache, transport remote, or other remote system does not by itself classify the data as P0/P1 or waive P2/P3/UNCLASSIFIED rules.
3. A prior valid publication to destination A does not authorize destination B.
4. A prior privacy violation or accidental remote presence must not be used as justification to continue publication.
5. M1.6 does not define transport's own outbound runtime implementation or retrofit current `transport_tool.py` behavior.

## 12. Typed privacy outcomes

The future unified interface must preserve typed privacy outcomes rather than flattening them into a generic boolean.

Conceptual outcome taxonomy:

```text
PRIVACY_ALLOW

PRIVACY_BLOCK_P3
PRIVACY_BLOCK_UNCLASSIFIED
PRIVACY_BLOCK_DESTINATION
PRIVACY_BLOCK_EXPLICIT_PROHIBITION
PRIVACY_BLOCK_PROVENANCE
PRIVACY_BUNDLE_BLOCKED

PRIVACY_DERIVATIVE_REQUIRED

PRIVACY_STALE

PRIVACY_CLASSIFICATION_CONFLICT
PRIVACY_POLICY_CONFLICT
PRIVACY_DESTINATION_CONFLICT
PRIVACY_BINDING_CONFLICT
```

The names are architectural taxonomy, not a claim that current runtime code already implements these symbols.

Each result should be sufficient for a later caller to know at minimum:

- whether remote side effects are permitted;
- which exact candidate/bundle/destination context was evaluated;
- whether the result is blocked, stale, conflicting, or derivative-required;
- whether a new candidate must be constructed rather than retrying the old one;
- whether required provenance contributed to the block.

A caller must not translate `STALE`, `CONFLICT`, `DERIVATIVE_REQUIRED`, or an unknown result into `ALLOW`.

## 13. Publication block and source retention

A privacy publication block is not a deletion request.

The following are distinct operations/concepts:

```text
omission
redaction
minimization
safe-derivative construction
retention
archive/supersession
purge
Git-history deletion/rewrite
secure deletion
```

Rules:

1. `PRIVACY_BLOCK_*` prevents the current remote publication attempt; it does not delete the local source record.
2. Omitting a field from a later derivative does not delete it from its owning source.
3. Redaction/minimization creates or contributes to a derivative; it is not equivalent to purge.
4. Deleting a derivative does not imply deletion of its source, and deleting a source does not automatically prove all derivatives/caches/history are gone.
5. Git-history deletion/rewrite is distinct from deleting the current working-tree object.
6. Secure deletion is a separate implementation/problem domain and is not defined by M1.6.
7. If a later privacy policy requires purge, that must be an explicit, separately governed operation with its own safety/verification semantics. M1.6 does not implement it.

## 14. Deterministic evaluation procedure

For one proposed publication attempt, the semantic evaluator performs this sequence:

```text
1. Identify exact destination and binding/scope context.
2. Materialize the exact candidate and intended publication bundle.
3. Resolve classification for every record/field/attachment/provenance component.
4. Apply inheritance and explicit prohibitions using most-restrictive-applicable semantics.
5. Detect P3, UNCLASSIFIED, policy/classification conflicts, or P2 remote content.
6. Evaluate destination-aware matrix for every included component.
7. Bind the verdict to exact candidate, policy/classification, destination,
   provenance, binding/scope, and bundle identities.
8. If every member permits publication, return PRIVACY_ALLOW.
9. Otherwise return the most specific fail-closed typed outcome.
10. Before the first remote side effect, verify the verdict context is still current.
11. Any drift -> PRIVACY_STALE -> recompute; do not reuse the old allow.
```

The evaluator must not mutate a blocked candidate in-place to make it pass. A changed payload is a new candidate and starts again at step 1/2 as appropriate.

## 15. Required invariants

The M1.6 architecture is invalid if any of the following can occur silently:

1. P3 or `UNCLASSIFIED` material reaches a remote publication destination.
2. A public Skill source accepts P1 merely because it is valid for a private Runtime Hub.
3. A P2 source object is published directly without explicit promotion, minimization, and independent reclassification.
4. A P0/P1 parent weakens a more restrictive field, attachment, provenance item, or explicit prohibition.
5. A blocked candidate is edited/redacted in place and published using the old privacy verdict/context.
6. A safe derivative inherits publication permission without independent classification.
7. A path, filename, hash, identifier, provenance reference, log, or error detail bypasses privacy evaluation because it is treated as "metadata".
8. A privacy allow is reused after candidate, policy/classification, destination, provenance, binding/scope, or bundle drift.
9. One member of a publication bundle is remotely written before whole-bundle privacy preflight completes.
10. A blocked member is silently dropped from the exact bundle and the remainder is published under the old bundle verdict.
11. A transport receipt/snapshot/message or prior remote presence is treated as publication permission.
12. A prior publication to one destination authorizes a different destination.
13. Write-capable reconciliation/CAS begins without a current privacy allow for the exact candidate/bundle/destination.
14. ADR-0003 stale recomputation reuses the privacy verdict of the discarded candidate.
15. Privacy block is treated as authority to delete/purge the local source.
16. Omission/redaction/retention/purge/Git-history deletion/secure deletion are collapsed into one implicit operation.
17. Privacy/classification ambiguity is resolved by selecting the least restrictive interpretation.
18. Explicit never-publish policy is weakened by a more specific child rule or later derivative metadata.

## 16. Decisions requested for M1.6 initial review — PV1–PV16

Initial review should output `APPROVE` or `REVISE` for every decision below.

- **PV1 — Five-state classification model.** M1.6 uses P0 `STRUCTURAL`, P1 `RUNTIME_SHAREABLE`, P2 `LOCAL_BY_DEFAULT`, P3 `PROHIBITED_SYNC`, and `UNCLASSIFIED`, with `UNCLASSIFIED` treated as fail-closed rather than low risk.
- **PV2 — Destination-aware permission.** Privacy permission is evaluated against an exact destination; authorized private Runtime Hub and public Skill source have different allow matrices, and permission never transfers automatically between destinations.
- **PV3 — P0/P1 public-private boundary.** P0 may be eligible for public source only when independently public-source-safe; P1 is limited to an exact authorized private runtime destination and is blocked from public Skill source.
- **PV4 — P2 promotion contract.** P2 is not directly remotely publishable; it requires explicit promotion, minimization/semantic projection, creation of a new candidate, and independent destination-specific reclassification.
- **PV5 — P3 and UNCLASSIFIED hard block.** P3 and `UNCLASSIFIED` hard-block the candidate/bundle for remote publication, and no default/inheritance rule can reinterpret them permissively.
- **PV6 — Monotonic inheritance.** Record/field/attachment/provenance/candidate/bundle classification uses most-restrictive-applicable semantics; explicit never-publish rules cannot be weakened and unresolved policy/classification ambiguity fails closed.
- **PV7 — Metadata/provenance are privacy-bearing.** Paths, filenames, hashes, identifiers, attachments, provenance, logs, and error/diagnostic details included in or required by publication participate in privacy evaluation.
- **PV8 — No redact-and-continue.** A blocked exact candidate cannot be silently stripped of P3/UNCLASSIFIED material and continued under the same identity/verdict; the blocked candidate remains blocked.
- **PV9 — Safe derivative independence.** Any sanitized/minimized safe derivative is a new independent object with new identity, provenance decision, classification, destination context, and privacy evaluation.
- **PV10 — Exact-context privacy verdict.** A privacy result binds exact candidate identity, classification/policy context, destination, permitted provenance, binding/scope, and publication-bundle identity.
- **PV11 — Drift invalidates verdict.** Candidate, classification/policy, destination, provenance, binding/scope, or bundle drift makes an earlier result stale; old `ALLOW` cannot be replayed.
- **PV12 — Whole-bundle preflight.** The complete intended publication bundle must receive privacy `ALLOW` before the first remote publication side effect; any blocked/stale/conflicting/derivative-required member blocks the attempt before writes begin.
- **PV13 — Typed fail-closed outcomes.** The unified interface distinguishes allow, P3/unclassified/destination/provenance blocks, derivative-required, stale, and classification/policy/destination/binding conflicts; non-allow outcomes cannot be flattened into success.
- **PV14 — Privacy does not imply deletion.** A publication block does not delete source data, and omission, redaction, minimization, retention, purge, Git-history deletion, and secure deletion remain distinct semantics.
- **PV15 — Remote presence grants no permission.** Transport mailbox state, receipts, snapshots, prior remote copies, or prior publication elsewhere do not create Runtime Hub/public-source publication permission.
- **PV16 — Privacy gates write-capable reconciliation.** The exact candidate/bundle must have a current privacy `ALLOW` before entering the ADR-0003 write-capable reconciliation/CAS path; stale CAS recomputation or conflict resolution that changes the candidate requires privacy re-evaluation.

## 17. Deferred implementation and later milestones

The following remain intentionally deferred and must not be inferred from M1.6 approval:

- detector/classifier implementation and confidence model;
- where classification manifests or privacy verdicts are physically stored;
- exact content-hash/identity algorithm and canonical serialization;
- public-source review tooling;
- Runtime Hub authorization/credential implementation;
- exact adapter APIs and Git/GitHub mechanisms;
- publication trigger policy;
- M1.7 startup/hydration/recovery semantics;
- migration of existing local or Runtime Hub records;
- transport runtime retrofits;
- purge executor;
- Git-history rewrite tooling;
- secure deletion guarantees/implementation;
- concrete UI wording, retry counts, or operator workflow;
- real-data tests or real project migration.

## 18. Evidence and architecture boundary

This proposal depends on and preserves the accepted boundaries in:

- `docs/decisions/ADR-0001-unified-authority-and-memory-boundary.md`;
- `docs/decisions/ADR-0002-local-multi-session-safe-write.md`;
- `docs/decisions/ADR-0003-remote-refresh-and-reconciliation-state-machine.md`;
- `docs/decisions/ADR-0004-hub-adapter-and-transport-boundary.md`.

In particular:

- ADR-0001 remains authoritative for explicit promotion, P3/UNCLASSIFIED fail-closed behavior, monotonic prohibition inheritance, and source retention boundaries;
- ADR-0003 remains authoritative for B/R/L/C concurrency/reconciliation after privacy allows a candidate to enter the write-capable path;
- ADR-0004 remains authoritative for separation between transport mailbox semantics and Hub adapter/shared-checkpoint semantics.

This document is an approved proposal. It remains `normative:false` and `accepted-unreleased`; the accepted decision is recorded normatively in ADR-0005. M1.6 approval creates no runtime-effective behavior, performs no migration, validates no implementation, and authorizes no real-data publication.