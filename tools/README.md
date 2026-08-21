# Tools

Future automation belongs here only when it is reusable across runtime hubs.

Potential utilities:

- runtime schema/version validator
- template drift checker
- project registry validator
- stale-write/concurrency test harness
- migration dry-run checker
- round-trip challenge generator/verifier

Tools must operate on explicit runtime-hub targets and must not embed private project state in this repository.

Until utilities are implemented, the Markdown protocol and synthetic tests are the source of truth.
