# Synthetic fixtures

These scenarios define test data without reproducing any private runtime project.

## Fixture A — correct routing

- workspace name: `Alpha Lab`
- workspace root: `C:\Synthetic\Alpha`
- project_id: `alpha-lab`
- expected route: `projects/alpha-lab/PROJECT.md`

## Fixture B — wrong workspace

Bootstrap target is `alpha-lab`, but the observed workspace is:

- workspace name: `Beta Lab`
- workspace root: `C:\Synthetic\Beta`
- project_id: `beta-lab`

Expected result: refusal; no binding or runtime write.

## Fixture C — concurrent sessions

- session A reads project SHA `P0`
- session B reads project SHA `P0`
- session A writes a validated finding and produces `P1`
- session B attempts a write based on `P0`

Expected result: session B detects `P1 != P0`, refreshes, preserves session A's finding, merges its own compatible change, and produces `P2` with no lost update.

## Fixture D — provenance

A session contains one directly verified filesystem fact and one prior-agent-reported claim that was not reverified.

Expected result: only the first is placed under validated conclusions; the second remains explicitly unverified/historical.
