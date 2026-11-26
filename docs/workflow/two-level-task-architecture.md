# The Two-Level Task Architecture

> Plan the **project** once. Execute each **task** as an independent
> build → test → review cycle that periodically resyncs with the project level.

This is the task-organization model layered on top of the
[File-as-Database Pattern](file-as-database.md). The File-as-Database pattern
says *state lives in versioned documents*; this document says *how to structure
those documents into two levels* so a large project doesn't get re-planned from
scratch on every change, and so each task can move independently.

## Why two levels

A single flat backlog forces a bad trade-off: either you design every task up
front (a giant plan that's stale before you finish it), or you design nothing
and let scope drift task by task. Two levels separate the two rhythms:

- **Level 1 changes rarely.** The global scope — what the product is and the
  complete set of requirements — is written *once* and touched only when scope
  genuinely shifts.
- **Level 2 changes constantly.** Each task carries its own detailed design,
  tests, and review, and runs its full cycle without waiting on the others.

The coupling between them is deliberately thin: a task points at the
requirement ids it satisfies, and reports one thing back up — "done".

## Level 1 — Project (one-time, rarely modified)

Lives in `docs/project/`.

### `PRD-project.md`
The global scope and the **enumerated requirements** `REQ-001 … REQ-N`. Each REQ
is a single atomic capability with a stable id that never gets reused. This file
is the durable "why/what" for the whole product; individual tasks do **not** get
their own PRD — their "why" is the REQs they implement.

### `TASK-BREAKDOWN.md`
Groups REQs into **tasks** (`T-001 …`), and for each task records: the REQs it
covers, its dependencies, a rough ordering/phase, and its **status**. This is
*coarse-grained* — it scopes and sequences work; it does **not** contain
per-task implementation detail (that's Level 2). The status column here is the
single source of truth for "where is the project".

> Level 1 is a map, not a blueprint. If you find yourself writing
> implementation detail in `TASK-BREAKDOWN.md`, it belongs in a Level-2 `TECH`.

## Level 2 — Per-task cycle (repeated for every task)

Each task is a folder `docs/tasks/T-NNN-slug/` whose documents are its state
carriers. For task `T-NNN` the cycle is:

```
TECH  ──▶  TEST-SPEC  ──▶  RED  ──▶  GREEN  ──▶  MUTATION-CHECK  ──▶  REVIEW  ──▶  merge
 (how)      (what to        (fail)   (pass)      (are the tests    (correct?)    (mark done in
            prove)                               real?)                          TASK-BREAKDOWN)
```

1. **`TECH.md`** — the implementation plan for *this task's* REQs; gated on
   `status: approved` before code.
2. **`TEST-SPEC.md`** — the observable behaviours this task must exhibit, written
   as concrete cases *before* implementation. It is the contract the tests encode.
3. **RED** — write the tests from the spec; they fail (nothing implemented yet).
   Commit them red so the trail shows the tests predate the code.
4. **GREEN** — implement until the tests pass, and nothing more.
5. **MUTATION-CHECK** — run mutation testing to confirm the tests actually catch
   regressions. Surviving mutants mean weak assertions → strengthen tests, not
   code. Record the surviving-mutant score in the REVIEW evidence. Pick your
   ecosystem's tool — Go: gremlins / go-mutesting; JS/TS: Stryker; Python:
   mutmut / cosmic-ray; Rust: cargo-mutants; Java: PIT.
6. **`REVIEW.md`** — correctness review with pasted evidence (test output +
   mutation score). On `approved`, set `TECH: implemented`.
7. **merge**, then **mark the task done in `TASK-BREAKDOWN.md`** — the resync.

### Why the mutation check
GREEN proves the code passes the tests. It does *not* prove the tests would
notice if the code broke. Mutation testing perturbs the code and checks that some
test fails; a surviving mutant is a blind spot in the suite. This is what keeps
"tests pass" from degrading into "tests exist".

## Resynchronizing Level 2 → Level 1

A task is not an island forever. Resync happens at two moments:

- **On completion:** flip the task's status in `TASK-BREAKDOWN.md` to `done`.
  This is the only routine write-up to Level 1.
- **On discovery:** if a task reveals a missing or wrong requirement, stop and
  amend Level 1 — add a `REQ`, or `supersede` one — in its own commit, *before*
  continuing. Never let a task silently redefine project scope inside its TECH.

Because both levels are files under git, a resync is just a commit; the history
shows exactly when scope changed and which task caused it.

## Statuses

| Document              | Status field                                        |
| --------------------- | --------------------------------------------------- |
| `PRD-project.md`      | `baseline` (rarely: `amended`)                      |
| `TASK-BREAKDOWN.md`   | per-row: `pending → in-progress → in-review → done` (or `blocked`) |
| `TECH.md`             | `draft → approved → implemented → superseded`       |
| `TEST-SPEC.md`        | `draft → approved` (then RED/GREEN/MUTATION tracked in REVIEW) |
| `REVIEW.md`           | `pending → changes-requested ⇄ … → approved`        |

The task's row status in `TASK-BREAKDOWN.md` is authoritative for the project;
the per-document statuses are authoritative within the task.

## Git conventions

One transition per commit; subject scoped by task id. Cycle-aware types:

```
prd(project):  amend/baseline the project PRD or REQs
tasks:         edit TASK-BREAKDOWN (grouping / ordering / status)
tech(T-NNN):   design
test(T-NNN):   TEST-SPEC, or RED tests
feat|fix(T-NNN): GREEN implementation
mut(T-NNN):    mutation-check hardening
review(T-NNN): review verdict
```

The trail reads as the cycle:

```
tech(T-004): approve mTLS + reconcile design
test(T-004): spec desired-set + usage behaviours
test(T-004): red — stream/reconcile/usage tests failing
feat(T-004): green — node-agent dials control plane, reconciles
mut(T-004): kill survivors in usage-baseline logic
review(T-004): approve; tests green, 0 survivors, mark TECH implemented
tasks: T-004 done
```

## Directory layout

```
docs/
├─ project/
│  ├─ PRD-project.md            # Level 1: global scope + REQ-001..N (one-time)
│  └─ TASK-BREAKDOWN.md         # Level 1: REQ→task grouping, deps, order, status
├─ tasks/
│  └─ T-NNN-slug/               # Level 2: one folder per task
│     ├─ TECH.md                # (== TECH-T-NNN.md; the folder carries {task})
│     ├─ TEST-SPEC.md
│     └─ REVIEW.md
└─ workflow/
   ├─ file-as-database.md       # the substrate (files + git = state)
   ├─ two-level-task-architecture.md   # this document (organization)
   └─ templates/
```

> Naming: conceptually the files are `TECH-{task}.md`, `TEST-SPEC-{task}.md`,
> `REVIEW-{task}.md`. Here `{task}` is carried by the enclosing `T-NNN-slug/`
> folder, so the files inside are simply `TECH.md` / `TEST-SPEC.md` /
> `REVIEW.md`. Same identity, less repetition.

## When to use it

- **Use it** for a project big enough to need sequencing and parallel tasks —
  i.e. anything with more than a handful of requirements.
- **Skip the ceremony** for a one-off change: a single `T-NNN` with a TECH +
  TEST-SPEC + REVIEW and a one-line REQ is fine; you don't need a full
  `PRD-project.md` to fix a bug.
