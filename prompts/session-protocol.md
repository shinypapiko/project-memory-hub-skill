# Session protocol

Use this protocol for substantive work inside an already bound project.

## Start

- resolve exactly one project;
- read the authoritative project record;
- record `base_project_sha` when available;
- create a session log only when durable traceability is useful.

## During work

- use chat context for transient reasoning;
- keep direct evidence separate from unverified/inferred claims;
- avoid unnecessary hub reads on ordinary follow-up messages;
- keep detailed work in the session log rather than bloating `PROJECT.md`.

## Before shared write

- re-read latest authoritative state;
- compare current SHA with session baseline/last-read SHA;
- if changed, reconcile before writing;
- never overwrite a newer record with an old snapshot.

## End

- write/update session log if warranted;
- update `PROJECT.md` only when authoritative current state changed;
- record new project SHA when available;
- keep project boundaries intact.
