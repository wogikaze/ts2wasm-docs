# Roadmap and success conditions

この文書は実装ロードマップと成功条件を扱う。workstream と gate の canonical 定義は `docs/11-shared-definitions.md` に置き、この文書では実装順序と成功条件の読み方を説明する。

## 実装ロードマップ

本プロジェクトは単一の直列ロードマップだけに従うのではなく、複数 workstream を並行して進める。優先順位は `docs/11-shared-definitions.md` の gate で判定する。

- W0 Runtime substrate と W1 Standalone WASI execution を先行し、generated wasm が iwasm で観測可能な振る舞いを出せる状態を維持する。
- W2 JS semantic core と W3 Data model は differential test を前提に拡張し、Node との差分を status schema で分類する。
- W4 Control/module/class は W2/W3 の整合を崩さない範囲で段階導入する。
- W5 Differential testing and coverage は常時運用し、test262/TypeScript corpus の executed と分類の推移を `artifacts/coverage/reference-coverage-matrix.md` で管理する。
- W6 Host capability boundary は standalone と host-required の境界を manifest で監査可能にする。
- W7 Optimization は correctness-preserving を前提とし、fast path/generic path の differential test と benchmark gate を同時に育てる。

## 成功条件

このプロジェクトの成功条件は、単に `.wasm` を出せることではない。TypeScript / JavaScript として意味のあるコードが、WASM 側で実行され、Node.js への依存が明示的に分離され、テストと性能の状態が継続的に測定されることが成功条件である。

最初の明確な成功ラインは、`docs/11-shared-definitions.md` の gate A-G に従う。個別文書では gate を再定義せず、必要に応じて `Gate A` のように参照する。

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
