# Documentation Index

このディレクトリは ts2wasm の current source of truth である。`issues/` は作業台帳、`plans/` と `reports/` は履歴であり、設計判断の現在値はここから辿る。

## 読み方

1. `README.md` で全体像を掴む。
2. この `docs/INDEX.md` でタスクに対応する文書を選ぶ。
3. 個別文書からコード path へ進む。

3 ホップ以内で目的地に着けない場合は、`docs/00-docs-list.md` を更新する。

## Task routing

| Task | Primary docs | Code/data anchor |
|---|---|---|
| 仕様・目標の確認 | `docs/01-project-definition.md` | `Cargo.toml`, `crates/*/Cargo.toml` |
| CLI/target/output | `docs/02-execution-model-and-targets.md` | `crates/cli/src/main.rs`, `crates/shared/src/abi.rs` |
| Capability/host import | `docs/03-api-and-host-capability.md` | `crates/shared/src/capability.rs`, `crates/runtime-catalog/src/link_plan.rs` |
| 設計判断の背景 | `docs/adr/INDEX.md` | `docs/adr/records/` |
| Architecture audit | `docs/29-architecture-audit-procedure.md` | `arch-rules.toml`, `architecture-exceptions.toml`, `scripts/check/*` |
| Compiler pipeline | `docs/04-compiler-architecture-and-runtime.md` | `crates/compiler/src/pipeline.rs` |
| Spec-kernel correctness backend | `docs/04-compiler-architecture-and-runtime.md` (Spec-Kernel section) | `crates/spec-kernel/`, `crates/backend-correctness/`, `crates/builtin-kernel/` |
| Semantics/support status | `docs/05-compatibility-and-semantics.md`, `docs/26-semantic-feature-matrix.md`, `docs/language-reference/INDEX.md` | `fixtures/catalog.yaml`, `fixtures/feature-matrix.yaml` |
| Tests/gates | `docs/06-testing-and-coverage.md`, `docs/15-coverage-matrix.md` | `scripts/manager.py`, `crates/*/tests` |
| Runtime ABI | `docs/14-runtime-abi.md` | `crates/runtime-abi/src/*` |
| IR/HIR/MIR | `docs/13-ir-contracts.md`, `docs/27-ir-layer-completion.md` | `crates/ir/src/*` |
| Frontend ownership | `docs/28-frontend-syntax-ownership.md` | `crates/syntax/src/ast.rs`, `crates/frontend/src/parser.rs` |
| Dashboard/site | `docs/18-web-ui-reporting.md` | `web-ui/`, `site/` |
| Roadmap/current work | `docs/current-state.md`, `docs/roadmap.md` | `issues/`, `fixtures/catalog.yaml` |

## Document families

| File | Role |
|---|---|
| `00-docs-list.md` | 全 Markdown の責務と更新ルール |
| `01-project-definition.md` | Why / goals / non-goals |
| `02-execution-model-and-targets.md` | CLI、target、output、server |
| `03-api-and-host-capability.md` | manifest、host import、security boundary |
| `04-compiler-architecture-and-runtime.md` | end-to-end pipeline と crate boundary |
| `05-compatibility-and-semantics.md` | supported subset と未対応分類 |
| `06-testing-and-coverage.md` | unit/snapshot/differential/reference gates |
| `07-performance-and-optimization.md` | server/batch/optimization/perf policy |
| `08-roadmap-and-success.md` | success criteria と roadmap lane |
| `09-security-and-capability-model.md` | capability-first security model |
| `10-related-projects.md` | TypeScript/QuickJS/Javy 等との差分 |
| `11-shared-definitions.md` | shared terms and schemas |
| `12-coding-standard.md` | human rules not enforceable by formatter/lint |
| `13-ir-contracts.md` | AST/HIR/MIR/Lowered boundaries |
| `14-runtime-abi.md` | tagged values and memory layout |
| `15-coverage-matrix.md` | reference coverage table interpretation |
| `16-commit-and-push-policy.md` | local work and verification policy |
| `17-jsonl-test-record-schema.md` | reference JSONL record schema |
| `17-windows-development.md` | Windows/WSL guidance |
| `18-web-ui-reporting.md` | dashboard data and build pipeline |
| `19-parallel-development.md` | parent/child worktree loop |
| `20-async-await-design.md` | Promise/task async design |
| `21-object-semantics-kernel.md` | object/prototype/property descriptor kernel |
| `22-completion-records.md` | JS completion semantics |
| `23-coverage-runner-completeness.md` | reference runner correctness |
| `24-architecture-decoupling-and-llm-friendly-sizing.md` | file-size and layering constraints |
| `25-robust-test-design.md` | test patterns resilient to drift |
| `26-semantic-feature-matrix.md` | fixture-derived support matrix |
| `27-coverage-expansion-epics.md` | coverage wave grouping |
| `27-ir-layer-completion.md` | HIR/MIR completion contract |
| `28-frontend-syntax-ownership.md` | syntax ownership table |
| `29-architecture-audit-procedure.md` | architecture audit and best-practice alignment procedure |

## Subdirectory maps

| Directory | Index | Role |
|---|---|---|
| `docs/adr/` | `docs/adr/INDEX.md` | accepted/rejected architecture decisions and their background |
| `docs/language-reference/` | `docs/language-reference/INDEX.md` | JS/TS/WASM/WASI feature support maps and parser wave planning |
| `docs/templates/` | `docs/templates/feature-slice.md` | templates for issue-backed feature slices |

## Freshness rule

When code and docs disagree, trust code for What and docs for Why. Then update docs in the same change. Generated artifacts must name their generator command.
