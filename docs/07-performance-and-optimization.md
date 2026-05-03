# Performance and optimization

この文書は performance goal、optimization levels、optimization strategy、benchmark state を扱う。

## Performance Goal

本プロジェクトは、TypeScript を WebAssembly へ変換するだけでなく、生成コードの実行性能を主要な価値として扱う。目標は、TypeScript / JavaScript の既存コードを可能な限り保ちながら、WASM backend によって Node.js / Bun と同等以上の性能を達成することである。

特に、型情報が静的に利用できるコード、数値演算が多いコード、配列アクセスが多いコード、標準入出力中心の短命 CLI、host API 呼び出しが少ないコードでは、Node.js / Bun より高速な実行を目指す。これは専用 subset を導入するためではなく、通常の TypeScript コードに対して最適化レベルを上げることで達成する。

本プロジェクトでは、用途別 profile ではなく、一般的な compiler optimization level を採用する。optimization level と semantic safety mode の対応は `docs/11-shared-definitions.md` を正とする。

```bash
ts2wasm build main.ts -O0 -o main.wasm
ts2wasm build main.ts -O1 -o main.wasm
ts2wasm build main.ts -O2 -o main.wasm
ts2wasm build main.ts -O3 -o main.wasm
ts2wasm dump --optimize -O2 main.ts
ts2wasm dump --optimize --unparse -O2 main.ts
```

`-O2` では、TypeScript の型情報、制御フロー、到達可能性、escape analysis、定数畳み込み、不要 runtime call の削除、primitive fast path、packed array の選択などを行う。`-O3` では、さらに関数インライン化、monomorphization、shape specialization、loop optimization、bounds check elimination、runtime helper の特殊化を行う。ただし `-O3` でも observable JavaScript semantics は壊さない。

性能目標は「全 JavaScript プログラムで常に Node.js / Bun に勝つ」ことではない。動的 property access、`any`、prototype mutation、Proxy、`eval`、高度な reflection を多用するコードでは、互換性を保つために runtime cost が発生する。一方で、TypeScript の型が十分に効くコードでは、JIT に依存せず、事前コンパイルされた WASM と軽量 runtime により、安定して高速な実行を狙う。

| 領域                  | 性能目標                                                       |
| ------------------- | ---------------------------------------------------------- |
| 数値演算                | `number` / `i32` / `f64` fast path により Node.js / Bun と同等以上 |
| 配列処理                | packed representation と bounds check 最適化により高速化             |
| 文字列処理               | runtime 実装を最適化し、不要な host call を避ける                         |
| 短命 CLI              | 起動時間込みで Node.js より軽い実行を目指す                                 |
| 標準入出力               | WASI stdio を直接使い、Node.js 依存を避ける                            |
| object-heavy code   | shape specialization により改善。ただし動的性が高い場合は互換性優先               |
| dynamic JS features | 正確性優先。性能目標は限定的                                             |

## Optimization Strategy

本プロジェクトでは、用途別 profile ではなく、通常の最適化レベルによって性能を制御する。特定用途向けの subset を導入すると、TypeScript to WASM transpiler という主目的が曖昧になるためである。

最適化は、TypeScript の通常コードに対して適用される。ユーザーは特別な競技用 API や専用 profile を使う必要はない。`-O2` または `-O3` を指定することで、compiler が型情報と実行形状を解析し、WASM に適した表現へ変換する。

主な最適化は次の通り。

| 最適化                      | 内容                                                                 |
| ------------------------ | ------------------------------------------------------------------ |
| primitive fast path      | `number`、`boolean`、`string` などの型が明確な処理を runtime call なしで実行         |
| numeric specialization   | 整数的に扱える演算を `i32` / `i64` / `f64` に落とす                              |
| packed array             | 要素型が安定した配列を linear memory 上の連続領域に配置                                |
| shape specialization     | object の property layout が安定している場合に lookup を高速化                    |
| function inlining        | 小さい関数や hot path の runtime helper を inline 化                        |
| monomorphization         | generic 関数を利用型ごとに特殊化                                               |
| escape analysis          | heap allocation を stack-like allocation または scalar replacement に変換 |
| bounds check elimination | 安全に証明できる配列境界チェックを削除                                                |
| dead code elimination    | 未使用 builtin / runtime helper を link しない                            |
| host call reduction      | Node.js / WASI import 呼び出しをまとめる、または削る                              |

> 本プロジェクトの性能戦略は、用途別 subset ではなく、TypeScript の型情報と最適化レベルに基づく汎用的な高速化である。結果として、数値計算、配列処理、短命 CLI、標準入出力中心のプログラムでは、競技用途にも耐える性能を目指す。

## Optimizer diagnostics

`ts2wasm dump --optimize -O0..-O3 <input.ts>` は、typed HIR に optimizer pipeline を適用した後の構造化 IR を表示する。`-O0` は pass を適用しない基準出力であり、`-O1` 以上は安全性を満たす pass だけを適用する。dump は `applied_passes` を含め、どの optimizer pass が実際に IR を変更したかを監査できるようにする。

`ts2wasm dump --optimize --unparse -O2 <input.ts>` は、optimized HIR を pseudo source として表示する。これは source text の復元ではなく、`local$N` / `fn$N` ID と semantic operation を保持した optimizer 後の診断出力である。

## Performance State

性能は、最初から測る。初期実装が遅くても、測定項目を固定しておけば改善できる。比較対象は Node、Bun、ts2wasm とする。測定条件、記録項目、regression gate は `docs/11-shared-definitions.md` の benchmark policy に従う。

| 指標                          | 意味                                 |
| --------------------------- | ---------------------------------- |
| compile time                | `.ts` から `.wasm` 生成までの時間           |
| startup time                | 実行開始から main 到達まで                   |
| execution time              | 実処理時間                              |
| memory usage                | heap / linear memory / host memory |
| wasm size                   | 生成 `.wasm` サイズ                     |
| host calls                  | Node.js / WASI import 呼び出し回数       |
| iwasm compatibility         | iwasm で実行できるか                      |
| correctness under benchmark | 高速化で意味論が壊れていないか                    |

ベンチマークは、数値計算、文字列処理、配列処理、オブジェクト操作、JSON、ファイル I/O、CLI 入出力に分ける。Node や Bun より遅いこと自体は問題ではない。問題なのは、どの層で遅いか分からない状態である。比較表には runner version、input size、cold/warm 区分を紐づける。

性能比較では、少なくとも次のような表を継続的に出す。

| benchmark     | Node | Bun | ts2wasm/wasi | ts2wasm/node-host | 備考              |
| ------------- | ---: | --: | -----------: | ----------------: | --------------- |
| fib           |   計測 |  計測 |           計測 |                計測 | 関数呼び出し          |
| array-sum     |   計測 |  計測 |           計測 |                計測 | typed fast path |
| string-concat |   計測 |  計測 |           計測 |                計測 | runtime string  |
| json-parse    |   計測 |  計測 |           計測 |                計測 | builtin         |
| fs-read       |   計測 |  計測 |           計測 |                計測 | WASI / Node 差分  |
| cli-stdio     |   計測 |  計測 |           計測 |                計測 | iwasm 重視        |

## 追加設計: cold and warm benchmark split

| Metric | Why |
|---|---|
| cold startup | WASM/iwasm が勝ちやすい可能性を見る |
| warm execution | Node/Bun JIT と比較する |
| compile time | ts2wasm 自体の実用性を見る |
| runtime init time | JS value runtime 初期化コストを見る |
| host call count | WASI/Node bridge の重さを見る |
| allocation count | JS 値 runtime の重さを見る |
| wasm size | iwasm/組み込み用途で重要 |
| peak linear memory | runtime 設計の健全性を見る |

## Expected performance regions

| Region | Expectation |
|---|---|
| short-lived CLI | 勝ちやすい |
| stdin → compute → stdout | 勝ちやすい |
| typed numeric loop | 勝ちやすい |
| packed array | 勝ちやすい |
| low host-call code | 勝ちやすい |
| object-heavy code | 互換性優先で限定的 |
| prototype mutation | 厳しい |
| Proxy / eval / reflection | 初期は性能目標外 |
| Promise / async / event loop | runtime/host 設計後に評価 |
