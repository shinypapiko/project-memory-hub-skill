---
document_role: architecture-decision-record
adr_id: ADR-0008
status: accepted
accepted: 2026-08-27
milestone: M2.1
runtime_effective: false
release_required_for_runtime_effect: true
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
depends_on:
  - docs/reviews/2026-08-27-unified-architecture-final-approval.md
---

# ADR-0008 — Canonical source and version authority

## Status

**Accepted bounded M2.1 source/version decision.**

This decision closes only canonical source/version authority. It does not implement the unified runtime, adapter, migration, publication trigger, UI, real-data handling, or validation.

## Decision

### 1. Canonical source repository and path

The canonical source repository for the future unified Project Memory Skill is:

```text
shinypapiko/project-memory-hub-skill
```

Within that repository:

```text
repository root = release/governance/version/test source root
skill/          = installable Skill source subtree
```

The existing public repository therefore remains canonical for its released Hub Protocol `0.1.x` history and becomes the canonical source location for the future unified Skill line. Later reusable local-core imports must land in this repository through explicit reviewed commits; M2.1 does not perform that import.

Public-source privacy remains governed by ADR-0005. Private project/runtime data is never promoted merely because this repository is canonical.

### 2. Canonical version identifier

The repository-root file:

```text
VERSION
```

is the canonical source-tree version identifier.

Current released truth remains:

```text
VERSION = 0.1.0
```

M2.1 selects the next unified development/release line as:

```text
first unified development identifier: 0.2.0-dev.0
first unified release identifier:     0.2.0
```

The root `VERSION` must not be changed to `0.2.0-dev.0` merely by this governance decision. It changes only when the first actual unified implementation/source promotion begins. Until then, `0.1.0` remains the truthful released source version.

The audited local development distribution's `1.1.0` marker remains provenance for that separate predecessor lineage; it is not silently adopted as the canonical unified SemVer line. Detailed historical lineage mapping remains later work.

### 3. Authority of development, released, and installed copies

Authority is one-directional:

```text
canonical repository source
        -> immutable tagged release/prerelease
        -> installed copies
```

Roles:

- **Canonical development source:** the exact commit on the canonical repository branch being developed. Untagged local checkouts are working copies, not independent authorities.
- **Canonical released source:** the exact canonical repository commit referenced by an immutable version tag/release and whose root `VERSION` matches that release identifier.
- **Installed copy:** a derivative runtime copy. It never becomes source authority merely because it is newer locally or has user modifications.
- **Legacy audited local development/installed copies:** predecessor evidence/reusable implementation inputs only. They may be imported by explicit reviewed promotion, but they do not override the canonical repository automatically.

If a development or installed copy differs from its recorded canonical source identity, the difference is drift, not a new source of truth.

### 4. Commit / tag / version mapping

For a canonical release or prerelease identifier `V`:

```text
root VERSION = V
Git tag       = vV
Git tag vV    -> exactly one canonical source commit
that commit   -> exactly the source tree represented by V
```

Examples for the selected unified line:

```text
VERSION 0.2.0-dev.0 <-> tag v0.2.0-dev.0 <-> one exact source commit
VERSION 0.2.0       <-> tag v0.2.0       <-> one exact source commit
```

A commit SHA is the exact development identity. A human version string without a matching canonical commit/tag does not prove source equivalence.

M2.1 does not create these tags or bump `VERSION`; it fixes the mapping rule only.

### 5. Drift detection

A development or installed copy is considered exactly aligned only when its declared source identity can be mapped to the canonical repository and its governed source bytes are consistent with that identity.

Conceptually the minimum evidence is:

```text
canonical repository identity
source commit SHA
canonical version identifier
relevant governed source-byte identity
```

The following are drift/unknown rather than implicit success:

- missing or ambiguous source commit identity;
- version marker that does not match its claimed canonical commit/tag;
- governed source bytes differing from the claimed source commit;
- local edits to an installed copy;
- a legacy installed copy with no independent version/source marker.

Exact implementation of manifests, hashes, installer metadata, or commands is deferred. M2.6 may add installed-copy metadata, but it must implement this authority rule rather than invent a second version authority.

### 6. Promotion direction

Permitted promotion direction is explicit:

```text
legacy/local reusable implementation
  -> reviewed privacy-safe import into canonical repository

canonical repository development commit
  -> reviewed release/prerelease tag
  -> installed copies
```

Automatic reverse promotion is forbidden:

```text
installed copy -> canonical source        NO
runtime project data -> public source     NO
transport/runtime snapshot -> source      NO
unreviewed local working tree -> release  NO
```

A local predecessor implementation may contribute reusable source only through an explicit reviewed import that preserves provenance and ADR-0005 public-source privacy requirements.

## Consequences

- There is one canonical future source repository rather than manually synchronized peer sources.
- Root `VERSION` plus exact source commit/tag become the release/version authority.
- Installed and local development copies are derivatives whose drift can be detected rather than silently promoted.
- Existing Hub `0.1.0` and audited local `1.1.0` provenance remain truthful without pretending they are one continuous historical SemVer sequence.
- The next implementation work can target one canonical source tree.

## Explicitly deferred

M2.1 does not decide or implement:

- reusable local-core import contents;
- adapter implementation;
- migration or real data;
- publication trigger;
- full runtime behavior;
- concrete installed-version metadata format;
- installer/update mechanism;
- UI;
- executable/static/live validation;
- detailed predecessor-version lineage mapping beyond preserving the `0.1.0` and audited `1.1.0` provenance facts.

## Maturity

```text
runtime_effective: false
implementation_status: PROTOCOL_ONLY
validation_status: UNVALIDATED
```

The next step may implement a minimum vertical slice in the canonical repository without reopening M1 architecture semantics.