# Pinklin Phase 1 — docs

**How we work** — a git-native, documents-as-state workflow. The files below
*are* the state database; `git log` is the audit trail.

| Doc | Covers |
| --- | --- |
| [workflow/file-as-database.md](workflow/file-as-database.md) | The substrate: files + git *are* the state DB; `status` frontmatter = state. |
| [workflow/two-level-task-architecture.md](workflow/two-level-task-architecture.md) | The structure: a one-time PRD + task breakdown (Level 1), then per-task TECH → TEST-SPEC → RED→GREEN→MUTATION-CHECK → REVIEW cycles (Level 2). |
| [workflow/templates/](workflow/templates/) | Templates for every document type. |

**The state itself**

| Path | Level |
| --- | --- |
| [project/PRD-project.md](project/PRD-project.md) | Level 1 — global scope + `REQ-001..N`. |
| [project/TASK-BREAKDOWN.md](project/TASK-BREAKDOWN.md) | Level 1 — REQ→task grouping, deps, ordering, **authoritative status**. |
| [tasks/](tasks/) | Level 2 — one folder per task, holding its `TECH` / `TEST-SPEC` / `REVIEW`. |

**Technical design（spec §13 Step 2，屬於任務 T-000）**

| spec 要求的文件 | 實際位置 | 狀態 |
| --- | --- | --- |
| `PRD.md` | [project/PRD-project.md](project/PRD-project.md) | amended |
| `ARCHITECTURE.md` | [ARCHITECTURE.md](ARCHITECTURE.md) | draft |
| `BUSINESS-RULES.md` | [BUSINESS-RULES.md](BUSINESS-RULES.md) | draft |
| `DATABASE-ERD.md` | [DATABASE-ERD.md](DATABASE-ERD.md) | draft |
| `API-SPECIFICATION.md` | [API-SPECIFICATION.md](API-SPECIFICATION.md) | draft |
| `SECURITY-DESIGN.md` | [SECURITY-DESIGN.md](SECURITY-DESIGN.md) | draft |
| `PAYMENT-FLOW.md` | [PAYMENT-FLOW.md](PAYMENT-FLOW.md) | draft |
| `REFERRAL-FLOW.md` | [REFERRAL-FLOW.md](REFERRAL-FLOW.md) | draft |
| `WALLET-LEDGER.md` | [WALLET-LEDGER.md](WALLET-LEDGER.md) | draft |
| `CHAIN-CUSTODY.md`（spec 未列，因 USDT 自管錢包新增） | [CHAIN-CUSTODY.md](CHAIN-CUSTODY.md) | draft |

建議閱讀順序：PRD → ARCHITECTURE → BUSINESS-RULES → 其餘依需要查閱。

New here? Read `workflow/two-level-task-architecture.md` for how work flows, then
see [`project/TASK-BREAKDOWN.md`](project/TASK-BREAKDOWN.md) for the active task list.
