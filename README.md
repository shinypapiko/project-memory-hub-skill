# Project Memory Hub Skill

Reusable source repository for the protocol, templates, tests, migrations, and maintenance guidance behind a persistent multi-project memory hub for ChatGPT/Codex-style agents.

## Boundary

This repository is the **skill/source layer**, not a runtime project-memory database.

- Skill source: protocol, templates, tests, migrations, release history.
- Runtime hub: user/project state such as `PROJECT.md`, session logs, research, and decisions.
- Workspace: local project files plus a small binding such as `AGENTS.md`.

Project-specific technical state must never be copied into this repository.

## Current release

- Skill version: `0.1.0`
- Supported runtime schema: `hub_schema_version: 1`
- Status: early reference implementation; backward-compatible refinement is expected before `1.0.0`.

## Core guarantees

1. Deterministic workspace → `project_id` → runtime-hub routing.
2. Strict project isolation by default.
3. Fresh-session recovery of authoritative project state.
4. Optional durable per-session work logs without turning `PROJECT.md` into a transcript.
5. Pre-write refresh and optimistic-concurrency protection using project blob SHAs when available.
6. Provenance separation between validated facts and unverified/historical claims.
7. Explicit migrations for incompatible runtime-schema changes.
8. Wrong-workspace refusal rather than silently binding the wrong project.

## Repository layout

```text
skill/        skill behavior contract
docs/         architecture and protocol documentation
templates/    reusable runtime/workspace templates
prompts/      bootstrap, round-trip, and session workflows
migrations/   schema migration records
tests/        synthetic protocol test scenarios
tools/        future validation/migration utilities
```

See `skill/SKILL.md` for the operational contract and `docs/compatibility.md` before changing runtime schema behavior.
