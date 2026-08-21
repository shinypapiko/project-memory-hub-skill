# Project Index

Use this file as the sole registry for runtime projects.

Routing order:
1. explicit project_id
2. exact observed workspace_root
3. exact workspace_name
4. canonical name or aliases only when unambiguous

Project entry template:

```md
### <project name>
- project_id: `<stable-slug>`
- path: `projects/<stable-slug>/PROJECT.md`
- status: active
- integration_status: registered-pending-workspace-audit
- workspace_names: <exact display name>
- workspace_roots: `Unverified`
- aliases: <aliases>
- scope: <boundary>
- excludes: <nearby projects that must remain separate>
```

Ask for clarification instead of blending projects when routing is ambiguous.
