---
id: T-NNN
title: <task title>
type: REVIEW
status: pending          # pending → changes-requested ⇄ ... → approved
reviewer: <name/email>
tech: docs/tasks/T-NNN-slug/TECH.md
reqs: [REQ-0NN]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# REVIEW — <task title>

## Scope reviewed
Commit range / diff under review (e.g. `feat(T-NNN)` commits abc123..def456).

## Checklist
- [ ] Implements the approved TECH and covers every listed REQ (no design drift)
- [ ] TEST-SPEC cases are all encoded as tests; suite passes (evidence below)
- [ ] Mutation-check run; surviving mutants triaged/killed (score below)
- [ ] Security & leak considerations addressed (per the project's stated posture)
- [ ] Docs updated

## Evidence
```
# tests (RED history → GREEN)
<paste the test-runner output>

# mutation-check
<paste the mutation tool's summary: killed / survived / score>
```

## Findings
- **F1** (blocker | major | minor) — <description and location>

## Verdict
`pending | changes-requested | approved` — rationale.

> On `approved`: set the linked TECH `status: implemented`, then mark the task
> `done` in [`TASK-BREAKDOWN.md`](../../project/TASK-BREAKDOWN.md) (the resync).
