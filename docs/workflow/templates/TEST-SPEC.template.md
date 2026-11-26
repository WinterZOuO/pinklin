---
id: T-NNN
title: <task title>
type: TEST-SPEC
status: draft            # draft → approved
owner: <name/email>
tech: docs/tasks/T-NNN-slug/TECH.md
reqs: [REQ-0NN]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# TEST-SPEC — <task title>

The observable behaviours this task must exhibit, written **before** the code.
These become the RED tests. Each case should trace to a REQ.

## Behaviours (cases)
- **C1** (REQ-0NN) — *Given* <state>, *when* <action>, *then* <observable result>.
- **C2** (REQ-0NN) — Given …, when …, then …

## Edge cases & failure modes
- <boundary / error / concurrency / restart case and expected handling>

## Mutation-check focus
Logic where a surviving mutant would be dangerous (must be killed by a test):
- <e.g. the anti-double-count arithmetic; the quota comparison boundary>

## Out of scope for this task's tests
- <explicitly not covered here, and why>

## Test placement
- Unit: `<path/to/pkg>` — <what>
- Integration: `<harness>` — <what>
