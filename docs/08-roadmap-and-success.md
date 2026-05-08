# Roadmap and success conditions

この文書は実装ロードマップと成功条件を扱う。workstream と gate の canonical 定義は `docs/11-shared-definitions.md` に置き、この文書では実装順序と成功条件の読み方を説明する。

## 主要目標: test262 semantic_pass 90%

本プロジェクトの primary success metric は **test262 semantic_pass 90%**（約 48,100 / 53,445 ファイル）である。これは Gate H に相当する。90% に至る段階目標として Gate G（50%）を設定する。

semantic_pass は `ts2wasm build` が成功し、かつ `iwasm` 実行結果が Node.js reference と一致した件数であり、外部向け conformance 報告にはこの数値を使う（`docs/15-coverage-matrix.md` 参照）。

## 実装ロードマップ

本プロジェクトは単一の直列ロードマップだけに従うのではなく、複数 workstream を並行して進める。優先順位は `docs/11-shared-definitions.md` の gate で判定する。

- W0 Runtime substrate と W1 Standalone WASI execution を先行し、generated wasm が iwasm で観測可能な振る舞いを出せる状態を維持する。
- W2 Syntax coverage で parser gap を埋め、全 ECMAScript/TypeScript 構文を通す。
- W3 Name resolution で global builtin / scope chain / function 解決を完了させる。
- W4 Builtin runtime で String / Array / Date / Math / RegExp 等の runtime 実装を拡大する。
- W5 Runtime semantics で object model、prototype、closure、class、exception、module の意味論を仕様に合わせる。
- W6 test262 coverage ramp で実行数を段階的に 53,445 まで拡大し、分類の推移を `artifacts/coverage/reference-coverage-matrix.md` で管理する。
- W7 Host capability boundary は standalone と host-required の境界を manifest で監査可能にする。
- W8 Optimization は correctness-preserving を前提とし、test262 90% 達成後が主対象。

## 成功条件

このプロジェクトの成功条件は、単に `.wasm` を出せることではない。TypeScript / JavaScript として意味のあるコードが、WASM 側で実行され、Node.js への依存が明示的に分離され、テストと性能の状態が継続的に測定されることが成功条件である。

最初の明確な成功ラインは、`docs/11-shared-definitions.md` の gate A-H に従う。個別文書では gate を再定義せず、必要に応じて `Gate A` のように参照する。

**Primary success metric: test262 semantic_pass 90%（Gate H）**。これは full corpus 53,445 ファイルに対して semantic_pass が約 48,100 件に達したことを意味する。

成功判定は「どの gate を満たしたか」で行い、「どの番号まで進んだか」では評価しない。workstream は並行して進み得るが、次の gate を通すために必要な証拠（tests、manifest、coverage、benchmarks）がそろっていることを必須にする。

## First slice

```text
single file TS/JS
→ JS semantic IR
→ runtime ABI call
→ WASI wasm
→ iwasm execution
→ Node differential test
→ capability manifest
```
