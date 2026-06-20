# Testing and Coverage

## Test layers

| Layer | Purpose | Examples |
|---|---|---|
| Unit tests | crate-local invariants and small behavior | `crates/runtime-abi/tests`, `crates/runtime-catalog/tests` |
| Snapshot tests | AST/HIR/MIR/lowered/manifest/link-plan stability | `crates/frontend/tests/parser_snapshot.rs`, `crates/ir/tests/*snapshot*` |
| Differential tests | Node.js vs generated wasm output | `crates/cli/tests/node_diff.rs`, catalog fixtures |
| Negative tests | expected diagnostics for invalid or unsupported input | `fixtures/negative`, parser/diagnostic tests |
| Reference coverage | larger suite execution and classification | `python scripts/manager.py reference-coverage ...` |
| Dashboard data | visualization of coverage/result history | `python scripts/manager.py coverage-dashboard-data` |

## Default commands

```bash
python scripts/manager.py check
python scripts/manager.py gate-fast
python scripts/manager.py nextest
python scripts/manager.py reference-coverage test262 --jsonl
python scripts/manager.py update-coverage-matrix -- --check
```

`gate` runs fmt, architecture checks, coverage matrix check, and nextest. Use focused tests first, then broader gates.

## Fixture catalog

`fixtures/catalog.yaml` is the fixture directory source of truth. Each directory records category, status, expected behavior, fixture names, and host import policy. Do not infer support from file existence alone.

## Feature matrix

`fixtures/feature-matrix.yaml` groups feature tags by fixture directory. `docs/26-semantic-feature-matrix.md` summarizes it for humans and agents.

## Reference coverage matrix

`artifacts/coverage/reference-coverage-matrix.md` is generated. It contains per-suite denominator, executed count, build/semantic coverage, unsupported reason tables, and evidence commands. Update with the manager script rather than manual edits.

## Source-Level Profiling

test262 runner profiling can collect Rust source-level timing records from the compiler server. The profiler is compiled out by default; build a profiling binary explicitly:

```bash
cargo build -p ts2wasm-cli --release --features source-profiler
TS2WASM_BINARY=target/release/ts2wasm mise run reference-coverage -- test262 --semantic fast --sample 2000 --source-profile --no-dashboard-data
```

The runner writes aggregate phase timing and source-level records to `artifacts/coverage/results/test262-profile.json`. Source records include `name`, `file`, `line`, `module_path`, `count`, `total_ms`, `avg_ms`, and `max_ms`.

## Evidence standard

A feature/change is done when the final report names:

- changed files,
- focused test command and result,
- broader gate command and result or why it was skipped,
- remaining unsupported/fail cases if any.
