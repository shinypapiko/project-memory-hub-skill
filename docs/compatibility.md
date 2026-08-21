# Compatibility and upgrades

## Version domains

Two versions are intentionally separate:

- **skill version** — semantic version of this repository, e.g. `0.1.0`;
- **hub schema version** — runtime structural/behavioral contract, e.g. `hub_schema_version: 1`.

A skill release declares which runtime schema versions it supports.

## Current compatibility

Skill `0.1.x` supports `hub_schema_version: 1`.

## Skill semantic versioning

- PATCH: wording, tests, validation, or maintenance change that requires no runtime migration.
- MINOR: backward-compatible protocol/template capability.
- MAJOR: incompatible skill behavior/API expectation; this may or may not coincide with a runtime schema change.

## Runtime schema changes

Increment `hub_schema_version` when runtime structure or behavior changes in a way compatibility checks or migrations need to understand.

Before upgrading a bound runtime hub:

1. read its schema marker;
2. preserve the pre-upgrade Git commit;
3. read and apply each required migration in order;
4. preserve project boundaries and provenance;
5. validate routing, wrong-workspace refusal, isolation, session behavior, and concurrency;
6. run a fresh-chat round-trip after behavior-affecting changes.

Never infer that an incompatible runtime can be upgraded safely without a documented migration.
