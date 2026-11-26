---
title: <project> — Task Breakdown
type: TASK-BREAKDOWN
status: baseline
owner: <name/email>
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# <project> — Task Breakdown

Groups the project PRD's requirements into tasks and sequences them.
Coarse-grained — detail lives in each task's Level-2 TECH. The **Status** column
is the authoritative project state.

## Tasks

| Task  | Title            | REQs             | Depends on | Phase | Status  |
| ----- | ---------------- | ---------------- | ---------- | ----- | ------- |
| T-001 | <title>          | REQ-001, REQ-002 | —          | P0    | pending |
| T-002 | <title>          | REQ-003          | T-001      | P1    | pending |

Status ∈ `pending | in-progress | in-review | done | blocked`.

## Ordering
```
<dependency sketch / critical path>
```

## REQ coverage
Every REQ is assigned to exactly one task. New REQ → extend a task or add one
here in the same change.

## Change log
- YYYY-MM-DD — Baseline breakdown.
