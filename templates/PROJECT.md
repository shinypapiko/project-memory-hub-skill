# <Project Name>

- project_id: `<stable-slug>`
- status: active
- integration_status: registered-pending-workspace-audit | workspace-bound-pending-roundtrip | bound | needs-review
- workspace_names: <exact workspace display name(s)>
- workspace_roots: <exact observed root path(s), or `Unverified`>
- aliases: <names/phrases the user may use>
- scope: <what belongs to this project>
- excludes: <similar projects/topics that must remain separate>

## Goal

<Primary objective and success criteria.>

## Current state

<Concise authoritative snapshot.>

## Workspace binding

- Expected project_id: `<stable-slug>`
- Expected workspace name: <exact display name>
- Workspace root: `Unverified` until directly observed
- Root instruction file: `Unverified` until directly observed
- Hub entry point: `<owner>/<runtime-hub>/START_HERE.md`
- Binding evidence: <direct audit / round-trip evidence>

## Important files and paths

- `<path>` — <purpose>

## Environment

- <tool/version/configuration>

## Validated conclusions

- <directly supported conclusion>

## Unverified or historical claims

- <claim and provenance limitation>

## Open problems / blockers

- <blocker>

## Next actions

1. <next action>

## Shared references

- Research: <exact shared links only when needed>
- Decisions: <exact shared links only when needed>

## Session references

- <exact `sessions/...` links only when useful>

## Isolation notes

- Do not import state from another project unless explicitly requested or linked through a shared reference.
- Keep project-specific paths, assumptions, experiments, and conclusions in this project only.
- Do not mark workspace identity as verified until directly observed.
- Do not mark `integration_status: bound` until a fresh-chat round-trip succeeds and is reconciled.
