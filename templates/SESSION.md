# Session — <short topic>

- session_id: `<stable session id>`
- project_id: `<project-id>`
- started_at: `YYYY-MM-DDTHH:MM:SS±HH:MM`
- ended_at: `Unverified` | `YYYY-MM-DDTHH:MM:SS±HH:MM`
- agent/runtime: <Codex / ChatGPT / other>
- workspace_name: <exact display name if applicable>
- workspace_root: <exact observed root or `Unverified`>
- base_project_sha: `<blob sha>` | `Unavailable`
- final_project_sha: `<blob sha>` | `Not changed` | `Unavailable`
- status: active | completed | blocked | superseded

## Task / scope

<What this session was asked to do and what it intentionally excluded.>

## Evidence inspected

- `<path / file / commit / command>` — <why it mattered>

## Actions taken

- <meaningful action>

## Validated findings

- <directly supported conclusion>

## Unverified / inferred findings

- <claim and provenance limitation>

## Project-state changes proposed

- <durable change for PROJECT.md, or `None`>

## Write-back / concurrency check

- Latest project SHA re-read before shared write: `<sha>` | `Not applicable`
- Did remote state change since the session baseline? yes | no | unknown
- Reconciliation performed: <what was merged or why none was needed>
- Shared files changed: <exact runtime-hub paths>

## Blockers

- <blocker or `None`>

## Next actions

1. <next action>

## Notes

Keep detailed transient work here when useful for traceability; promote only current authoritative state to `PROJECT.md`.
