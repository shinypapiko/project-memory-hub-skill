# START HERE

This is the mandatory routing entry point for this memory hub.

## Startup protocol

1. Read this file first.
2. Read `MEMORY.md` for stable cross-project conventions only.
3. Read `projects/INDEX.md`; it is the sole project registry.
4. Identify exactly one project using this priority order:
   - explicit `project_id` supplied by the user or workspace instructions;
   - exact `workspace_root` match when the current root is known;
   - exact `workspace_name` match;
   - canonical name, aliases, and scope only when the match is unambiguous.
5. Read only `projects/<project-id>/PROJECT.md` for authoritative project state.
6. Record the blob SHA of that project record as `base_project_sha` when available.
7. Read linked shared notes only when needed.
8. If project identity is not unique, ask rather than blending state.

## Session lifecycle

- `PROJECT.md` is authoritative current state.
- `projects/<project-id>/sessions/` stores durable work logs when useful.
- Session logs do not replace `PROJECT.md`.
- Do not create durable logs for trivial chatter or work with no durable result.

## Read triggers

Refresh `PROJECT.md` when the user requests latest state, a meaningful gap makes staleness plausible, project identity changes, another session may have changed relevant state, a shared write is about to occur, or a conflict is suspected.

Do not re-read the hub on every ordinary follow-up message.

## Pre-write synchronization

Before modifying `PROJECT.md` or another shared authoritative file:

1. re-read the latest remote file and obtain its blob SHA when available;
2. compare it with the session's last-read SHA;
3. if unchanged, merge and write normally;
4. if changed, treat the session as stale, inspect intervening state, reconcile, then write;
5. never overwrite newer state with an older snapshot merely to make a write succeed;
6. after a successful write, record the new project SHA.

## Isolation rules

- Project facts stay under their own `projects/<project-id>/` boundary.
- `MEMORY.md` contains only genuinely cross-project durable conventions/findings.
- Shared research and decisions must have explicit scope/applicability.
- Never merge project state because names, tools, datasets, paths, or topics look similar.

## Write-back protocol

After substantive progress, decide whether the result belongs in a session log, `PROJECT.md`, shared research, a shared decision, or no durable write at all. Keep validated findings separate from hypotheses and unverified/historical claims.

## Runtime limitation

Repository instructions do not execute themselves. A chat/agent/tool must first access the hub; these rules govern behavior after access.
