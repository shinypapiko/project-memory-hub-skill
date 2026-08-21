# Concurrency

Multiple sessions may work on the same project concurrently. The default strategy is optimistic concurrency.

## Session baseline

A substantive session records the `PROJECT.md` blob SHA it read as `base_project_sha` when the runtime/tool exposes one.

## Pre-write check

Before changing a shared authoritative project file:

1. re-read the latest remote content;
2. obtain the current blob SHA;
3. compare it with the session's last-read SHA;
4. if unchanged, merge and write;
5. if changed, mark the session stale, inspect intervening state, reconcile, then write.

## Lost-update rule

Never force an older project snapshot over newer state merely because the write API rejects the old SHA.

## Reconciliation

Preserve both valid lines of work where they are compatible. If they conflict semantically, retain provenance and make the conflict explicit rather than silently choosing one result.

## Session logs

Session logs reduce contention because detailed work can be written to session-specific paths while only durable current-state changes are merged into `PROJECT.md`.

## Locks

Schema 1 does not require a lock service. A future schema may introduce stronger coordination if optimistic concurrency proves insufficient.
