# Live synthetic concurrency test

## Purpose

Validate actual agent behavior for optimistic concurrency without using private runtime project data.

## Synthetic state

Use a disposable synthetic runtime fixture with:

- project_id: `alpha-lab`
- authoritative file: `projects/alpha-lab/PROJECT.md`
- initial blob SHA observed by both sessions: call it `P0`
- session A durable change: add validated finding `A1`
- session B durable change: add validated finding `B1`

## Procedure

1. Start fresh session A and fresh session B against the same synthetic project.
2. Require both to read the same initial `PROJECT.md` and record `base_project_sha = P0`.
3. Let session A write `A1` to the authoritative project state, producing `P1`.
4. Without refreshing B manually, instruct session B to persist `B1`.
5. Session B must re-read the current project state before writing.
6. Session B must detect that the current SHA is `P1`, not its baseline `P0`.
7. Session B must preserve `A1`, merge `B1`, and write a new authoritative state `P2`.
8. Verify the final project state contains both `A1` and `B1` and no unrelated changes.

## Acceptance criteria

PASS only if all are true:

- both sessions originally record the same `P0`;
- A produces a newer authoritative state `P1`;
- B explicitly detects stale state before its shared write;
- B does not overwrite `P1` with its older snapshot;
- final state `P2` contains both compatible findings;
- session logs, if created, remain scoped to their own session paths;
- no other synthetic project is read or modified.

## Failure conditions

Any of the following is a failure:

- B writes without refreshing;
- A's change disappears from the final state;
- B forces an update using old content merely to satisfy a SHA conflict;
- project state is blended with another fixture;
- the agent claims reconciliation without evidence of reading the intervening state.

## Evidence to retain

Record only synthetic evidence in this repository:

- initial/final project blob SHAs;
- session A and B logs or responses;
- commit SHAs for A and B writes;
- exact final synthetic `PROJECT.md` content;
- PASS/FAIL verdict and any protocol defect discovered.
