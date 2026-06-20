# Docs List

`docs/INDEX.md` is the primary map. This file records ownership, freshness expectations, and how non-canonical Markdown should be treated.

## Canonical docs

| Scope | Files | Update when |
|---|---|---|
| Project identity | `README.md`, `docs/01-project-definition.md` | goals, non-goals, repository shape, or target positioning change |
| Execution/API | `docs/02-execution-model-and-targets.md`, `docs/03-api-and-host-capability.md` | CLI flags, targets, manifest schema, host imports, server protocol change |
| Architecture | `docs/04-compiler-architecture-and-runtime.md`, `docs/13-ir-contracts.md`, `docs/14-runtime-abi.md` | phase boundary, IR type, ABI layout, runtime link plan change |
| Semantics | `docs/05-compatibility-and-semantics.md`, `docs/20-async-await-design.md`, `docs/21-object-semantics-kernel.md`, `docs/22-completion-records.md`, `docs/28-frontend-syntax-ownership.md` | JS/TS support behavior changes |
| Tests/coverage | `docs/06-testing-and-coverage.md`, `docs/15-coverage-matrix.md`, `docs/23-coverage-runner-completeness.md`, `docs/25-robust-test-design.md`, `docs/26-semantic-feature-matrix.md` | fixture catalog, reference runner, JSONL schema, or gate changes |
| Process | `AGENTS.md`, `CLAUDE.md`, `docs/16-commit-and-push-policy.md`, `docs/19-parallel-development.md` | agent workflow, commit policy, worktree model changes |
| UI/reporting | `docs/18-web-ui-reporting.md`, `web-ui/README.md`, `site/README.md` | dashboard data schema/build path changes |

## Generated or archived Markdown

| Location | Treatment |
|---|---|
| `issues/` | Excluded from broad docs rewrites unless explicitly requested. Source of truth for work items. |
| `artifacts/coverage/reference-coverage-matrix.md` | Generated coverage artifact. Refresh with `python scripts/manager.py update-coverage-matrix`. |
| `artifacts/issue-inventory-report.md` | Snapshot report. Regenerate or update from `issues/` inventory. |
| `plans/`, `.agents/plans/` | Historical plans. Keep current routing header; do not treat as canonical. |
| `reports/`, `reports/runs/`, `.recursive/run/` | Historical run reports. Preserve evidence; add current routing header if touched. |
| `.agents/skills/` | Agent procedure docs. Keep concise and route to canonical docs. |
| `site/docs/**` | Published site output or static site content. Do not duplicate design truth there. |
| `docs/adr/` | Decision records. Keep immutable after acceptance except status/supersession updates. |
| `docs/language-reference/` | Feature maps. Keep implementation status concise and route executable work to `issues/`. |

## Rules for adding docs

- Add a short entry to `docs/INDEX.md` or this file.
- Prefer a small focused doc over expanding root `AGENTS.md`.
- Use path references instead of inline-expanding large docs.
- State negative constraints explicitly: what must not exist or must not be added.
- If a doc contains mainly completion history, issue roll-up, or run narrative, delete it or move the surviving contract into a canonical doc.
