# Architecture

The system is intentionally split into five layers:

```text
skill source repository
        ↓ protocol / templates / migrations
runtime memory hub
        ↓ project routing and durable shared state
project record
        ↓ authoritative current state
session log
        ↓ durable work evidence/history
chat context
        ↓ transient working context
```

## Responsibilities

### Skill source repository

Defines reusable behavior. It contains no project-specific runtime data.

### Runtime memory hub

Stores the routing table, schema marker, global conventions, isolated project folders, shared research/decisions, and project session logs.

### `PROJECT.md`

Authoritative snapshot of the project's current state. It should be concise enough to load at session start.

### Session logs

Append-oriented evidence and work history. They preserve useful detail without turning `PROJECT.md` into a transcript.

### Chat

Transient context. A fresh chat is not expected to inherit another chat's full transcript; it recovers durable state through the workspace binding and runtime hub.

## Design principles

- deterministic identity over semantic guessing
- isolation by default
- evidence-backed state
- explicit uncertainty
- pre-write synchronization
- optimistic concurrency rather than coarse locks
- migrations for incompatible schema changes
