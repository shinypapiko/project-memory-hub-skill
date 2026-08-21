# Routing and isolation

## Routing priority

1. explicit `project_id`
2. exact directly observed `workspace_root`
3. exact `workspace_name`
4. canonical name / aliases / scope only when unambiguous

Exactly one project is selected by default.

## Wrong-workspace refusal

If an instruction targets project A but the observed workspace is project B, stop before changing workspace bindings or runtime state. Report the mismatch and require the agent to run the bootstrap from the intended workspace.

## Isolation boundary

Project-specific state lives under `projects/<project-id>/`. Similar names, common software, shared datasets, methods, paths, or research themes are not sufficient reason to merge project state.

Cross-project work must be explicit. Shared reusable findings should be linked through shared notes rather than copied blindly between projects.

## Runtime identity evidence

A workspace root is authoritative only after direct observation. Unknown paths remain `Unverified`. A local instruction file is not considered verified until its existence and effective scope have been directly inspected.
