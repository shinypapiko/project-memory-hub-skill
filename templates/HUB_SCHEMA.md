# Hub schema

- hub_schema_version: `1`
- status: active
- introduced: `<YYYY-MM-DD>`

## Scope

Schema version 1 includes:

- `START_HERE.md` as routing/protocol entry point
- `projects/INDEX.md` as sole project registry
- isolated `projects/<project-id>/PROJECT.md` authoritative state
- workspace binding through stable `project_id`
- integration lifecycle including fresh-chat round-trip validation
- optional durable `projects/<project-id>/sessions/` logs
- `base_project_sha` / pre-write refresh semantics when blob SHAs are available
- optimistic concurrency and stale-session reconciliation
- shared `research/`, `decisions/`, `prompts/`, and `archive/` areas with explicit scope boundaries

Increment this schema version only when reusable runtime structure or behavior changes in a way compatibility checks or migrations need to understand.
