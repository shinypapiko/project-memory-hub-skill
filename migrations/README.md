# Migrations

Record runtime-hub schema migrations here. A migration is required when a hub schema change cannot be adopted safely by template replacement alone.

## Naming

Use one file per transition, for example:

```text
001-schema-1-to-2.md
002-schema-2-to-3.md
```

## Required migration contents

Each migration must state:

- source `hub_schema_version`
- target `hub_schema_version`
- compatibility assumptions
- exact files/fields affected
- backup / rollback point requirements
- project-isolation safeguards
- transformation procedure
- validation procedure
- whether a fresh-chat round-trip is mandatory
- rollback procedure

## Current state

No migrations are required yet. Skill `0.1.0` supports runtime schema `1`.
