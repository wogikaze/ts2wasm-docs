# Execution model and targets

この文書は、generated wasm、WASM runtime、host shim の三層モデル、実行ターゲット、出力形式、CLI を扱う。

## 基本方針

このプロジェクトの実行モデルは、次の三層で考える。

| 層               | 役割                                             | Node.js 依存         |
| --------------- | ---------------------------------------------- | ------------------ |
| generated wasm  | TypeScript から生成される実行本体                         | なし、または明示 import のみ |
| ts2wasm runtime | JS 値、文字列、配列、オブジェクト、例外、GC 補助などを提供する WASM 側ランタイム | なし                 |
| host shim       | WASI では足りない API を提供する薄い層                       | 必要な場合のみ            |

重要なのは、host shim を薄く保つことである。たとえば `console.log` を呼ぶために Node.js を使うのは許容できるが、関数本体の実行やオブジェクト操作を Node.js に戻すのは許容しない。`fs.readFileSync` を Node host 経由で提供する場合でも、読み込んだデータの処理は WASM 側で行う。

## 実行ターゲット

第一ターゲットは WASI 対応の core wasm とする。iwasm での実行を重視するため、初期段階では Wasm GC や Component Model に強く依存しすぎない。線形メモリ上に JS 値表現を実装し、WASI の `fd_read`、`fd_write`、ファイル API、環境変数、引数などを扱う。

WAMR (wasm-micro-runtime) は W3C Wasm MVP 完全準拠で、WASI 対応、multi-thread、socket API、JIT/AOT をサポートしている。バイナリサイズも小さく（fast interpreter ~58.9K, AOT ~29.4K）、組み込み環境に適している。

第二ターゲットとして、Node.js host 付き WASM を用意する。このターゲットでは、WASI だけでは表現しにくい API を Node.js import として補う。`process` 全体の完全互換、一部の `fs`、`path` の Node 固有差分、`Buffer` の Node 固有 API、タイマー、非同期処理などはこの層で段階的に扱う。`process.argv` と `process.env` の基本読み取りは、まず WASI args / environ に対応付ける。

第三ターゲットとして、将来的に Wasm GC / Component Model / WIT を利用したより型付きの host interface を検討する。wasm-tools は既に Component Model を実装しており（Stage 4+ではないがデフォルト有効）、jco は JavaScript/TypeScript から Components をビルドし、ES modules に transpile できる。ただし、これは初期の iwasm 実行可能性を壊さない範囲で進める。iwasm で動く `.wasm` と、より高機能な runtime 向け `.wasm` は、同じ compiler pipeline から別 backend として出す。

WASM 提案の詳細な対応状況は `docs/language-reference/wasm-features.md` を参照。

## Target matrix

| Target | Runtime dependency | Intended use | Hard gate |
|---|---|---|---|
| `wasm32-wasi` | WASI imports only | iwasm / wasmtime / wasmer | iwasm execution |
| `wasm32-wasi+node-host` | WASI + generated Node host shim | Node-specific API fallback | manifest lists host imports |
| `wasm32-wasi-gc` | future Wasm GC backend | typed object/runtime optimization | not initial gate |
| `wasm32-component` | future Component Model/WIT backend | typed host interface | not initial gate |

## 出力形式

出力形式は少なくとも三種類を用意する。

| 出力                | 内容                               | 用途                        |
| ----------------- | -------------------------------- | ------------------------- |
| `.wasm`           | standalone または WASI 向け core wasm | iwasm / wasmtime / wasmer |
| `.wasm + host.js` | Node.js host shim 付き WASM        | Node API 併用               |
| `.wat`            | デバッグ用テキスト出力                      | compiler 開発・差分確認          |

`.wasm` 単体で動くものは、iwasm 実行を必須ゲートにする。Node.js host が必要なものは、host import の一覧を manifest として出す。これにより、「このプログラムはなぜ iwasm 単体で動かないのか」を説明できる。

出力には `capabilities.json` のようなメタデータを付けるとよい。たとえば、使用している API、必要な WASI capability、Node host import、メモリ初期値、export 関数、entrypoint などを記録する。

## CLI 設計

CLI 名は仮に `ts2wasm` とする。

```bash
ts2wasm build main.ts -o main.wasm
ts2wasm run main.ts
ts2wasm check main.ts
ts2wasm test tests/**/*.ts
ts2wasm emit-wat main.ts -o main.wat
ts2wasm explain main.ts
ts2wasm dump main.ts
ts2wasm dump --ast --unparse main.ts
ts2wasm dump --lowered main.ts
```

`build` は `.wasm` を生成する。`run` はビルド後に適切な runtime で実行する。Node host が不要なら iwasm または wasmtime で実行し、Node host が必要なら生成された host shim を使う。`check` は parser / semantic / unsupported feature を確認する。`explain` は、そのソースが standalone で動くのか、Node host が必要なのか、どの API が原因なのかを表示する。

`dump` は compiler 開発用に pipeline の中間表現を表示する。フェーズ指定なしでは利用可能なフェーズを順に表示し、個別指定では `--tokens`、`--ast`、`--resolved`、`--lowered`、`--wat` を選べる。`--ast --unparse` は AST を正規化された疑似ソースとして出力する。

たとえば、`process.env.FOO` の読み取りだけを使っているコードに対して `explain` を実行すると、WASI environ で実行できることを示す。

```text
target: wasm32-wasi
standalone: yes
required WASI capabilities:
  - wasi.env
required host APIs: []
reason:
  process.env read is initialized from WASI environ
suggestion:
  pass environment through the selected WASI runner
```

Node.js host fallback が必要なのは、WASI environ から作った runtime facade では表現できない挙動を使う場合に限る。

```text
target: wasm32-wasi+node-host
standalone: no
required WASI capabilities:
  - wasi.env
required host APIs:
  - host.process.env.native_mutation
reason:
  observed process.env operation requires Node-compatible host state
suggestion:
  avoid relying on OS environment mutation, or compile with --host node
```
