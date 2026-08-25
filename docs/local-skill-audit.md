# Existing local `project-memory` skill audit

## Purpose

Before extending this repository, audit any already-deployed local `project-memory` skill implementation and determine whether this repository should become its canonical source, a Hub adapter, or a separate component.

The audit is evidence-gathering only. Do not modify the local skill, workspace memory, runtime Hub, or this source repository as part of the audit run itself.

## Sources to compare

1. User-level installed `project-memory` skill.
2. Any development/test copy of that same local skill.
3. This GitHub source repository, `project-memory-hub-skill`.
4. The runtime `project-memory-hub` protocol only where needed to understand integration boundaries.
5. One representative local `.ai/` tree for schema/behavior verification; project-specific content must not be copied into this public source repository.

## Phase A — inventory and identity

For each local skill copy:

- enumerate every source/config/template/reference file, excluding caches;
- record relative path, byte size, and SHA-256;
- compare the installed and development copies byte-for-byte;
- identify generated/cache files separately;
- identify version markers, if any;
- record how Codex discovers or invokes the skill, including `agents/openai.yaml` or equivalent declarations.

For the GitHub source repository:

- record current `VERSION`;
- enumerate the implementation/documentation/template/test structure;
- identify which files are normative behavior and which are descriptive only.

## Phase B — behavior contract

Read the local `SKILL.md` and references and build an explicit operation map. At minimum identify:

- LOAD / startup behavior;
- RETRIEVE / selective history loading;
- CHECKPOINT / write-back behavior;
- compaction behavior;
- experiment logging;
- decision logging and ID allocation;
- handoff behavior;
- validation/status commands;
- failure/refusal behavior;
- any automatic or implicit Codex invocation behavior.

For every operation, state:

- trigger;
- files read;
- files written;
- provenance rules;
- expected output;
- whether it can mutate state;
- whether an equivalent exists in `project-memory-hub-skill` 0.1.x.

## Phase C — local memory schema

Inspect one representative `.ai/` tree and determine the actual schema and authority of:

- `PROJECT.md`;
- `CURRENT.md`;
- `INDEX.md`;
- `TASKS.md`;
- `decisions/`;
- `experiments/`;
- `handoffs/`;
- any additional metadata files.

Do not copy project-specific technical content into the source-repository audit. Report only reusable structure, fields, lifecycle, and read/write relationships.

Determine which files are normally loaded at startup, loaded lazily, appended, rewritten, or archived.

## Phase D — scripts and transport

Inspect `memory_tool.py`, `transport_tool.py`, and any other executable helper.

For each command/function, record:

- purpose;
- arguments;
- files or repositories touched;
- whether it is read-only or mutating;
- error handling;
- assumptions about paths, Git, GitHub, Python, or workspace structure;
- tests or validation coverage.

Do not execute a command unless static inspection establishes that the exact invocation is read-only. `--help`, source inspection, and non-mutating status/validate operations may be used when proven safe.

For transport, determine whether it is currently configured and whether it provides push, pull, merge, export/import, or only an exchange primitive. Distinguish implemented capability from enabled runtime configuration.

## Phase E — compare with `project-memory-hub-skill`

Produce a capability matrix with four outcomes:

- local-only;
- Hub-skill-only;
- overlapping but semantically different;
- equivalent.

At minimum compare:

- activation/discovery;
- memory schema;
- startup retrieval;
- selective retrieval;
- write policy;
- compaction;
- experiments;
- decisions;
- handoffs;
- project isolation;
- cross-workspace routing;
- GitHub transport/shared memory;
- session logs;
- stale-SHA concurrency handling;
- migration/versioning;
- tests and validation tooling.

Explicitly identify duplicated implementations and places where combining them naively would create two competing sources of truth.

## Phase F — source-of-truth recommendation

Do not merge or rewrite anything yet. End the audit with evidence-backed options, for example:

1. make the existing local `project-memory` implementation the canonical core and add a Hub adapter;
2. migrate the local implementation into this repository and install releases from here;
3. keep two packages with a sharply defined core/transport boundary;
4. another structure supported by the audit.

For each option state migration risk, compatibility impact, and what would happen to existing `.ai/` trees and bound Hub projects.

## Required audit report

The report should contain:

1. executive summary;
2. exact inventories and hashes;
3. installed-vs-development-copy comparison;
4. local behavior map;
5. `.ai/` reusable schema map;
6. script/tool map;
7. transport status;
8. capability matrix against GitHub `project-memory-hub-skill`;
9. conflicts/duplication/gaps;
10. source-of-truth options;
11. recommended next step;
12. explicit list of anything not verified.

## Privacy boundary

This repository may be public. Do not publish private workspace contents, experiment results, personal paths beyond what is strictly needed for implementation diagnosis, credentials, repository tokens, or user-specific project state. A detailed audit containing private data should remain local or be stored only in an explicitly private location; only sanitized reusable conclusions belong here.
