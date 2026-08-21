# Project Memory Hub Skill

## Purpose

Maintain durable, isolated project memory across multiple agent sessions and workspaces without treating chat transcripts as the authoritative project state.

## Supported runtime

- Skill version: `0.1.0`
- Supported `hub_schema_version`: `1`

Before acting on a runtime hub, read its schema marker and refuse incompatible migrations unless an explicit migration path exists.

## Operating contract

### 1. Identify the runtime hub and project

Use the runtime hub's routing entry point. Prefer project identity in this order:

1. explicit `project_id` from workspace instructions or the user;
2. exact directly observed `workspace_root`;
3. exact `workspace_name`;
4. canonical name / aliases / scope only when unambiguous.

Route to exactly one project by default. If identity remains ambiguous, ask rather than blend project states.

### 2. Start a substantive session

Read the project's authoritative `PROJECT.md` and record its blob SHA as `base_project_sha` when available. Do not repeatedly re-read the hub for ordinary follow-up messages.

Create a durable session log only when the work produces traceability value. Session logs are evidence/history, not the authoritative current project state.

### 3. Preserve isolation

Read and write only the selected project plus the minimum shared references required by the task. Do not import facts from similarly named projects, neighboring workspaces, prior chats, or shared tools unless explicitly linked or requested.

### 4. Separate provenance

Keep directly validated conclusions separate from hypotheses, inferred state, historical claims, and user/agent-reported claims that were not independently reverified.

### 5. Pre-write refresh

Before modifying `PROJECT.md` or another shared authoritative project file:

1. re-read the latest remote file;
2. obtain its current blob SHA when available;
3. compare with the SHA last read by the session;
4. if unchanged, merge the new durable state normally;
5. if changed, treat the session as stale, read intervening changes, reconcile, then write;
6. never overwrite newer state with an older snapshot merely to make a write succeed.

This is optimistic concurrency control; no lock is required.

### 6. Classify write-back

After substantive progress, decide whether the result belongs in:

- a session log only;
- `PROJECT.md` because authoritative current state changed;
- shared `research/` because the result is reusable across projects;
- shared `decisions/` because a consequential decision needs durable applicability metadata.

Do not write trivial chatter or transient debugging noise into durable project state.

### 7. Workspace binding

A workspace-root instruction file such as `AGENTS.md` may bind the workspace to a stable `project_id` and runtime-hub entry point. Preserve unrelated pre-existing instructions. Never silently replace an existing instruction file.

If a bootstrap instruction names a different workspace than the currently observed workspace, stop with a wrong-workspace refusal and make no binding changes.

### 8. Integration lifecycle

Recommended states:

- `registered-pending-workspace-audit`
- `workspace-bound-pending-roundtrip`
- `bound`
- `needs-review`

Do not mark a project `bound` until a fresh-chat round-trip proves workspace binding → correct hub routing → correct project read → scoped hub write-back.

### 9. Runtime upgrades

Read the current runtime schema marker, preserve the pre-upgrade commit, apply every required migration in order, validate isolation/concurrency behavior, and repeat a fresh-chat round-trip after behavior-affecting upgrades.

### 10. Source/runtime boundary

Never copy private project records, datasets, paths, experimental results, or user-specific project state into this skill source repository. Tests must use synthetic fixtures.
