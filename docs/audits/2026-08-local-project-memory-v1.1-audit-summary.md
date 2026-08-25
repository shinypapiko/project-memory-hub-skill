# Sanitized audit summary — existing local `project-memory` implementation

- audit_date: `2026-08-26`
- audit_type: strict read-only architecture/capability audit
- privacy: sanitized; no project-specific technical state is included
- purpose: establish an evidence baseline before unifying the local Project Memory implementation with the Runtime Hub protocol

## Scope

The audit compared:

1. the user-level installed `project-memory` Skill;
2. a development/test copy of that Skill;
3. the public `project-memory-hub-skill` repository at Hub Skill `0.1.0`;
4. the Runtime Hub protocol only as needed to understand integration boundaries;
5. one representative local `.ai/` tree at the reusable schema/behavior level only.

No local files, Git state, Runtime Hub data, AGENTS instructions, or Skill sources were modified. Mutating transport commands and write-capable tests were not executed.

## Identity and version provenance

- The two audited local Skill copies contained the same 16 non-cache source/config/reference/template files and were byte-for-byte identical at audit time.
- The installed Skill directories themselves do **not** contain an independent `VERSION` file.
- The audited development distribution contains a root `VERSION` marker of `1.1.0`.
- The local transport tool reports `TOOL_VERSION = "1.1.0"`.
- Therefore the precise statement is: **the audited development distribution and transport tool identify themselves as `1.1.0`; the installed Skill directories do not yet expose an independent Skill version marker.**
- Local Codex discovery includes `agents/openai.yaml` with implicit invocation enabled.

## Local implementation capabilities observed

The local implementation has reusable behavior and executable tooling for:

- LOAD / RETRIEVE / CHECKPOINT workflows;
- local `.ai/PROJECT.md`, `CURRENT.md`, `INDEX.md`, and `TASKS.md`;
- indexed decision and experiment records;
- handoff inbox/outbox/processed lifecycle;
- optional session/research records;
- retrieval, write, schema, architecture, handoff, and transport policies;
- validation/status/ID-allocation via `memory_tool.py`;
- mailbox/handoff/receipt/non-canonical snapshot transport via `transport_tool.py`;
- executable smoke and transport E2E tests in the development distribution.

The audit directly ran only operations that static inspection established as read-only (`validate` and `status`). Validation passed for the inspected local memory tree.

## Runtime transport status

Transport capability is implemented locally, but it was **not enabled** in the inspected project because the active transport configuration/state files were absent.

The existing transport is a Git-backed mailbox/exchange mechanism, not a canonical project-memory database. Its CURRENT snapshot is explicitly non-canonical. It must not be silently reinterpreted as a Hub synchronization adapter.

## Hub Skill / Runtime Hub maturity

For Hub Skill `0.1.0` and Runtime Hub schema `1`:

- deterministic project routing, isolation, fresh-chat round-trip behavior, session-log semantics, pre-write refresh, stale-SHA reconciliation, and migration rules are **specified** in Markdown contracts/templates;
- the published static protocol regression is **STATIC_VALIDATED**;
- live two-agent concurrency remains **pending**;
- reusable runtime-Hub executable tooling is not yet present in `tools/`.

Use two independent maturity dimensions rather than a single ladder:

- `implementation_status`: `PROTOCOL_ONLY` or `EXECUTABLE`;
- `validation_status`: `UNVALIDATED`, `STATIC_VALIDATED`, or `LIVE_VALIDATED`.

For the current remote stale-SHA/reconciliation behavior, the accurate classification is:

- `implementation_status: PROTOCOL_ONLY`
- `validation_status: STATIC_VALIDATED`
- `live_validation: PENDING`

## Capability comparison — high-level

### Primarily local-only today

- Codex implicit discovery metadata;
- detailed local `.ai/` schema;
- executable validation/status tooling;
- experiments and ID allocation;
- full local decision lifecycle;
- handoff lifecycle and transport;
- executable local smoke/E2E tests.

### Primarily Hub-only today

- multi-project registry/routing;
- workspace → stable `project_id` binding across projects;
- remote per-project shared record;
- remote blob-SHA optimistic-concurrency protocol;
- Hub schema compatibility/migration model.

### Overlapping but semantically different

- startup recovery;
- selective retrieval;
- durable write policy;
- decisions;
- sessions;
- project isolation;
- GitHub/shared-memory semantics;
- test strategy.

## Core architectural risk

The largest risk is not duplicate filenames by themselves; it is two systems claiming authority over the same semantic state.

Examples requiring explicit unification rules:

- local `.ai/PROJECT.md` / `.ai/CURRENT.md` versus Hub `projects/<project-id>/PROJECT.md`;
- local per-project `INDEX.md` versus Hub cross-project `projects/INDEX.md`;
- non-canonical transport snapshots versus a remote shared project checkpoint;
- local DEC/EXP/session semantics versus Hub decisions/research/session semantics;
- duplicate startup paths that load both local and remote state without a precedence/reconciliation contract.

## Audit conclusion

Do **not** build a second local-memory core inside the Hub Skill.

The evidence supports a unification design in which:

- the existing local implementation supplies the mature local memory core;
- Hub-specific routing, shared checkpointing, remote concurrency, and migration capabilities are integrated through a separately defined Hub adapter/protocol;
- authority is assigned by information class and scope;
- local and remote mutable state use explicit pre-write refresh/reconciliation;
- detailed local evidence is not automatically mirrored to the Hub;
- the final unified Skill is released from one canonical source stream.

No migration or source import was performed by this audit.

## Unverified / pending

- live two-session Runtime Hub concurrency behavior;
- local multi-session concurrent writes to `.ai/CURRENT.md` / indexes;
- final canonical repository for the unified Skill;
- final field-level mapping between local `.ai/PROJECT.md` and Hub `PROJECT.md`;
- unified installation/update/version metadata;
- synthetic migration and rollback behavior.
