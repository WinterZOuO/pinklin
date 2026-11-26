# The File-as-Database Pattern

> The development workflow does not need a workflow engine. The file system is
> the database, git is the audit log, and the structured documents you already
> write **are** the state.

This is the *substrate*: how state is stored and moved. The *structure* built on
top of it — which documents exist and how they're organized into project and
task levels — is the [Two-Level Task Architecture](two-level-task-architecture.md).
Together they replace Jira / Temporal / a bespoke state machine for coordinating
work between humans and AI agents.

## Core insight

Most workflow tools exist to answer four questions:

1. **What is the state of this work?** (scoped / designing / building / review / done)
2. **Why is it in that state?** (the intent, the decisions, the trade-offs)
3. **Who changed it, when?** (the audit trail)
4. **What happens next?** (the gates and transitions)

The artifacts you already produce — a project **PRD** with enumerated
requirements, per-task **TECH** / **TEST-SPEC** / **REVIEW** documents — answer
#1 and #2 far better than a ticket field ever could. If those documents live in
the repo with an explicit `status`, then:

- The **document set + their statuses** *is* the state → answers #1.
- The **document bodies** are the intent and decisions → answer #2.
- **`git log`** over a document path is a complete, timestamped, attributed,
  immutable audit trail — for free → answers #3.
- A small set of **transition rules** are the gates → answer #4.

No separate database can drift out of sync with the code, because the state
lives *beside* the code and moves through the *same* review and merge process.

## State lives in frontmatter

Every document carries YAML frontmatter. `status` is the state field; the rest is
metadata that makes the git trail queryable.

```yaml
---
id: T-004
title: node-agent ↔ control-plane desired-state stream
type: TECH          # PRD-project | TASK-BREAKDOWN | TECH | TEST-SPEC | REVIEW
status: draft       # state machine depends on type
reqs: [REQ-010, REQ-011]
owner: alice@example.com
created: 2026-07-07
updated: 2026-07-07
---
```

Encode state in a **field**, not in the folder location, so that:

- moving between states is a small diff, not a file rename (clean `git blame`),
- one folder holds the entire history of one unit of work in one place,
- `git log -- docs/tasks/T-004-*` reads as the story of that work.

The invariant is simply: **if a document exists, its `status` is authoritative.**

## Git is the audit trail

Two conventions turn `git log` into a queryable event log.

**One state change per commit.** Don't bundle "approve the design" with "start
coding". Each meaningful transition is its own commit so the log has clean,
atomic events.

**Conventional commit subjects, scoped by id:**

```
<type>(<id>): <summary>
type ∈ { prd, tasks, tech, test, feat, fix, mut, review, docs, chore }
```

This makes the trail directly greppable:

```bash
# The full decision history of one task:
git log --oneline -- docs/tasks/T-004-node-agent-control-plane-stream/

# When and why did a design change?
git log -p -- docs/tasks/T-004-*/TECH.md

# Everything that reached review last week:
git log --since=1.week --grep '^review('
```

You get provenance (`git blame`), timestamps, authorship, and the *reasoning*
(the diff to the document body) without maintaining any of it by hand.

## How AI agents use this

An agent working in this repo treats the documents as shared memory:

1. **Read before acting.** Load the relevant project + task documents. Their
   `status` fields tell you what you're allowed to do next; their bodies tell you
   the agreed intent. Do not re-derive intent from code.
2. **Respect the gates.** Do not write feature code while the task's `TECH` is
   `draft`. If the design is unclear, update the `TECH` and request approval
   instead of guessing in code.
3. **Record decisions where they belong.** A design choice goes in the `TECH`
   body (and its Decision Log), not buried in a commit message or a chat reply.
   The commit points *at* the change; the document *is* the change.
4. **One transition per commit,** using the conventional subject above.
5. **Leave evidence in `REVIEW`.** Paste actual test output and the mutation
   score so an `approved` verdict is auditable, not asserted.

## What this deliberately is *not*

- Not a replacement for CI, tests, or code review of the *code* — it organizes
  the *decisions* around them.
- Not a rigid process for trivial work. Skip documents that add no signal; the
  rule is "if a document exists, its status is truth," not "every change needs a
  full document set."
- Not a second copy of state that can drift. There is exactly one copy, in the
  repo, moving through the same merge process as the code.

## Where the documents live

The concrete layout (project `PRD-project.md` + `TASK-BREAKDOWN.md`, and per-task
`docs/tasks/T-NNN-slug/` folders) is defined by the
[Two-Level Task Architecture](two-level-task-architecture.md). Templates for
every document type are in [`../assets/templates/`](../assets/templates/).
