# Session lifecycle

## Start

At the start of a fresh substantive session:

1. resolve exactly one project;
2. read its authoritative `PROJECT.md`;
3. record the project blob SHA as `base_project_sha` when available;
4. load only linked shared notes required for the task.

Do not re-read the hub on every ordinary follow-up message.

## Refresh triggers

Refresh authoritative project state when:

- the user explicitly requests current/latest hub state;
- the session resumes after a meaningful gap and staleness is plausible;
- project identity changes;
- another session may have changed relevant state;
- a shared authoritative write is about to occur;
- a state conflict is detected or suspected.

## Session log

Create a durable session log only when the work has traceability value. Record task scope, evidence, actions, validated/unverified findings, proposed project-state changes, concurrency checks, blockers, and next actions.

## End

If authoritative state changed, perform a pre-write refresh and reconcile before writing `PROJECT.md`. Record the resulting project SHA in the session log when available.
