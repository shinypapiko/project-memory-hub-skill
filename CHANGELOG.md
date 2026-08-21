# Changelog

All notable changes to the reusable skill implementation are recorded here.

## 0.1.0 — 2026-08-21

Initial source-controlled release derived from a validated runtime reference implementation.

### Added

- deterministic project routing contract
- workspace binding through stable `project_id`
- strict project-isolation rules
- fresh-session recovery and round-trip validation model
- optional durable session logs
- `base_project_sha` / pre-write refresh semantics
- optimistic concurrency and stale-session reconciliation
- validated vs unverified provenance separation
- runtime schema compatibility and migration policy
- synthetic test matrix for routing, refusal, isolation, round-trip, concurrency, and migration behavior

### Compatibility

- supports `hub_schema_version: 1`

No project-specific runtime data is part of this release.
