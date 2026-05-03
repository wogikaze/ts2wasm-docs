# Coverage Matrix

Last updated: 2026-05-04

この文書は reference 配下のテスト資産を分母にして coverage を可視化する dashboard である。
workstream の進行度ではなく、外部参照スイートに対してどこまで実行・分類できているかを 1 行ずつ管理する。

## 運用ルール

- coverage 行は reference suite 単位で更新する（test262、TypeScript tests、typescript-go など）。
- 分母は原則 `reference/<suite>` 配下のファイル数またはテストケース数で固定し、算出コマンドを `evidence` に残す。
- 分子は `build_pass` と `semantic_pass` を別列で記録する。`unsupported` / `blocked` / `skip-with-reason` は内訳として別管理し、いずれの coverage% の分子にも含めない。
- 同一 suite に ramp と deterministic selected subset の両方の証拠がある場合は、canonical suite 行を ramp 証拠として残し、選択条件を明示した追加 evidence 行を同じ generated table に追記する。selected subset 行は `evidence` に `--paths-file` / `--path-filter` を含め、canonical ramp 行を置換しない。
- gate 判定に影響する変更時は、影響する suite 行の `executed` と status 内訳を同時に更新する。
- `executed` は段階的に増やす。1 回の更新あたり `test262:+50`、`tsc:+30`、`tsgo:+20` を基本ステップとする。
- `unsupported (DiagCode breakdown)` と `unsupported (feature breakdown)` 列で優先実装対象を可視化する（例: `UnsupportedSyntax:120`, `class:50,import-export:30`）。

## Reference Coverage Dashboard

基準日 (2026-04-25) に集計した分母:

- test262: 53,445 files (`reference/test262/test/**/*.js`)
- TypeScript compiler cases: 6,419 files (`reference/typescript/tests/cases/compiler/**/*.ts`)
- typescript-go testdata: 165 files (`reference/typescript-go/testdata/tests/**`)

注記:

- PR では coverage gate (`mise run update-coverage-matrix -- --check`) を実行し、matrix 未更新を失敗扱いにする。
- PR では base branch 比較 gate も実行し、`executed` 減少と `fail` 増加を失敗扱いにする。
- 定期実行では ramp ステップで executed を拡大し、`artifacts/coverage/reference-coverage-matrix.md` 更新 PR を自動作成する。
- この dashboard の `build_coverage%` は `(build_pass / denominator) * 100`、`semantic_coverage%` は `(semantic_pass / denominator) * 100` で計算する。
- 実測 table は generated artifact に分離し、`artifacts/coverage/reference-coverage-matrix.md` を正とする。

## Metric Definitions

Executed:

- 実際に `mise run reference-coverage` で走らせた件数。
- full corpus 件数ではなく、現在の ramp limit までの実行数を表す。

Build-pass:

- `ts2wasm build` が成功した件数（wasm ファイルが生成された）。
- build-pass は「コンパイルが通った」ことを示すのみで、実行結果の正しさを保証しない。
- 対外的な coverage claim で build-pass を「conformance」と表現することは禁止する。

Semantic-pass:

- `ts2wasm build` が成功し、かつ `iwasm` 実行結果が Node.js reference と一致した件数。
- test262 conformance を主張するには semantic-pass の分子/分母を使う。
- coverage table には `build_pass` と `semantic_pass` を別列として管理する。
- 現在の test262 semantic-pass は `artifacts/coverage/reference-coverage-matrix.md` の `semantic_pass` 列を参照する。

Coverage%:

- `build_coverage% = build_pass / denominator * 100`。
- `semantic_coverage% = semantic_pass / denominator * 100`。
- 外部向け conformance 報告には `semantic_coverage%` を使う。
- `unsupported` / `blocked` / `skip-with-reason` は coverage の分子には含めない。

Unsupported:

- 診断として終了したが、既知未対応として扱う件数。
- 現在は `InvariantViolation` と `BackendIo` 以外の compiler diagnostics を unsupported として集計する。
- `unsupported (feature breakdown)` は diagnostic message と reference path から導出した安定 label を集計する。TestRecord の `tracking` も `feature:<label>` を使う。

Fail:

- panic / compiler bug / unexpected internal failure / invalid wasm 相当として扱う件数。
- 現在は `InvariantViolation` を fail として集計する。

Gate:

- fail count must not increase
- executed count must not decrease
- `artifacts/coverage/reference-coverage-matrix.md` must match `mise run update-coverage-matrix -- --check` output
- build_pass と semantic_pass が別列として記録されていること

Generated table:

- `artifacts/coverage/reference-coverage-matrix.md` を参照する。

### テストクラス運用（build vs semantic）

`parser_smoke`, `build_smoke`, `semantic_diff` の三段階で fixture を区別する。

- `build_smoke`: `ts2wasm build` が成功するかを確認。`build_pass` にのみ反映。
- `semantic_diff`: Node.js 参照と `iwasm` 実行の一致を確認。`semantic_pass` にのみ反映。
- `parser_smoke`: parser / 基本検証に限定する（現時点では未導入）。

現時点では `m6_builtin_methods.rs`、`m7_control_flow.rs`、`m8_oop_classes.rs`、`m9_modules.rs`、`m10_node_apis.rs` を build_smoke として運用し、  
class/module/node-api の semantic claim は `m2_node_diff.rs` の differential テストで明示しない限り扱わない。

## 計測スクリプト

計測スクリプトの使い方、運用順序、ゲート実行手順は `AGENTS.md` の Build/Test Commands を正とする。

## Test262 Coverage

This section tracks test262 coverage using the Stream G test infrastructure.

### Test262 Runner Workflow

実行コマンドと計測運用手順は `AGENTS.md` を参照する。

### Test Status Classification (align with `build_pass` / `semantic_pass`)

- **Build-pass**: `ts2wasm build` が成功し wasm が生成された（`build_pass` 列に集計）。
- **Semantic-pass**: build-pass に加え、`iwasm` の stdout が Node.js reference と一致した（`semantic_pass` 列に集計。Node/iwasm が無い環境では `semantic_enabled=0` となり semantic-pass は増えない）。
- **Fail**: コンパイラ内部不整合・`InvariantViolation` など、ビルド失敗のうち unsupported/blocked に分類されないもの。
- **Unsupported**: 未対応機能など、compiler diagnostic として終了したもの（`UnsupportedSyntax`, `UnresolvedName` など）。
- **Blocked**: `BackendIo`、タイムアウト、実行時 I/O 失敗など。

### Current Test262 Sample Results

The coverage matrix above shows test262 execution counts. For detailed test results, see the Stream G artifacts:

- `test262-results.jsonl`: Machine-readable test records (JSONL format)
- `test262-report.html`: Human-readable HTML report with category breakdown
- `test262-report.md`: Markdown version of the report

## Internal Smoke (Reference 分母の外)

| suite | files | result | note |
|---|---:|---|---|
| project fixtures (`fixtures/basics-hello,primitives-control-flow,core-semantics,arrays-objects`) | 19 | pass | これは reference coverage の分子には含めない |

## Gate 連携ルール

- Gate F に影響する変更では、まず `artifacts/coverage/reference-coverage-matrix.md` の該当 suite 行を更新する。
- `current-state.md` は実装詳細、`docs/15-coverage-matrix.md` は coverage のポリシーと判定基準に専念する。
- Gate F 判定では test262 行の `executed > 0` と status 内訳が必須。

## 更新チェックリスト

- reference suite の分母が変わっていないか確認したか
- 変更対象 suite の `executed` と status 内訳を `artifacts/coverage/reference-coverage-matrix.md` に反映したか
- `build_coverage%` / `semantic_coverage%` の再計算を反映したか
- `unsupported (DiagCode breakdown)` と `unsupported (feature breakdown)` が最新の実行結果を反映しているか
- 必要なら `current-state.md` と整合したか
