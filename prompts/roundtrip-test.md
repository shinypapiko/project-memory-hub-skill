# Fresh-chat round-trip test

## Purpose

Prove that a fresh session can derive project identity from the local workspace binding, route through the runtime hub to the correct project, read the expected authoritative state, and perform a narrowly scoped write-back without the user restating the repository path or project ID.

## Challenge requirements

Use a unique challenge ID and nonce, and record the expected pre-test project-record blob SHA.

The fresh session must:

1. read applicable local workspace instructions;
2. derive the `project_id` from the binding;
3. read the runtime routing entry, project registry, and correct project record;
4. read the project-specific challenge;
5. write only the permitted response artifact;
6. preserve local workspace files/instructions;
7. report the challenge ID, nonce, project/workspace identity, binding source, project-record SHA, and read confirmations.

## Final reconciliation

A separate reviewer reads the response and its commit diff. Mark the project `bound` only when identifiers and SHAs match and the response commit changed only the allowed runtime artifact.
