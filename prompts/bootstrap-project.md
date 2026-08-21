# Bootstrap project binding

Use this prompt from the intended workspace only.

## Objective

Audit the current workspace from direct evidence, preserve existing instructions, bind it to a stable runtime-hub `project_id`, and publish a scoped bootstrap report.

## Procedure

1. Verify the observed workspace identity/root before changes.
2. If the current workspace does not match the requested target, stop with a wrong-workspace refusal.
3. Inspect all applicable existing workspace instruction files.
4. Derive technical scope/current state only from direct workspace evidence; mark uncertainty `Unverified`.
5. Insert the memory-hub binding without deleting unrelated instructions.
6. Do not modify project code/data/results as part of bootstrap.
7. Publish only the permitted bootstrap report to the runtime hub.
8. Do not mark the integration `bound`; that requires a separate fresh-chat round-trip and reconciliation.

## Required report evidence

Include exact workspace identity/root, effective instruction file, preservation status, technical scope/current state, important paths, observable environment, unresolved items, source-control state if applicable, and change-scope confirmation.
