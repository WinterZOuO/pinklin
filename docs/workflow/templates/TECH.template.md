---
id: T-NNN
title: <task title>
type: TECH
status: draft            # draft → approved → implemented → superseded
owner: <name/email>
reqs: [REQ-0NN]          # requirements from PRD-project this task implements
task: docs/tasks/T-NNN-slug/
created: YYYY-MM-DD
updated: YYYY-MM-DD
superseded-by:
---

# TECH — <task title>

> **Gate:** do not begin implementation until `status: approved`. The `reqs`
> above must exist and be accepted in [`PRD-project.md`](../../project/PRD-project.md).
> After approval the cycle is TEST-SPEC → RED → GREEN → MUTATION-CHECK → REVIEW.

## Approach
The chosen design for this task's REQs, in prose. What and why this way.

## Components & files
Packages/files touched, new interfaces, how they fit the existing architecture.

## Data model / API changes
Schema, protobuf/DTO, or endpoint changes. Include migration notes.

## Alternatives considered
- <option> — why rejected.

## Risks & mitigations
- <risk> → <mitigation>

## Verification approach
How this will be proven — points at the TEST-SPEC and the mutation-check target.

## Rollout
Sequencing, flags, backward-compat, and any ops steps.

## Decision log
- YYYY-MM-DD — <decision> (commit `<hash>`)
