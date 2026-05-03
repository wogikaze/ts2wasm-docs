# Project definition

この文書は、プロジェクトの立ち位置、目標、非目標、禁止事項をまとめる。

## Terminology

このプロジェクトは便宜上 `TS transpiler to WASM` と呼ぶが、実態は単純な transpiler ではない。TypeScript / JavaScript frontend、JS semantics を扱う IR、WASM 側 runtime、WASI/Node host binding を持つ AOT compiler/runtime として扱う。

入力言語の基準は TypeScript と ECMAScript である。AssemblyScript 固有の型、構文、標準ライブラリ、実行意味論は採用しない。最適化のために内部 IR 上で `i32`、`f64`、packed array などの表現を使うことはあるが、それをユーザーが書く TypeScript 構文として要求してはいけない。

## Compatibility levels

| Level | Meaning |
|---|---|
| L0 | `.wasm` を生成できるが、意味論互換は限定的 |
| L1 | 基本式・制御構文・関数・配列・文字列が Node と一致する |
| L2 | object / property / closure / exception の主要ケースが一致する |
| L3 | test262 の指定 shard を分類付きで安定実行できる |
| L4 | 選定した既存 TypeScript package の一部を変換・実行できる |
| L5 | Node/Bun との差分が明示された production candidate |

## 概要

本プロジェクトは、TypeScript を WebAssembly にトランスパイルする処理系を実装する。目的は、TypeScript / JavaScript の既存資産を可能な限り活用しながら、Node.js や Bun の上でしか動かない実行モデルから離れ、WASI や iwasm などの WebAssembly 実行環境で動作可能な `.wasm` を生成することである。

このプロジェクトは「TypeScript を別言語に寄せる」ものではない。TypeScript の構文、JavaScript の実行意味論、既存テスト資産、既存エコシステムを尊重しつつ、出力先を WebAssembly にする。したがって、単に TypeScript 風の新言語を作るのではなく、TypeScript / JavaScript として書かれたコードを、できる限りそのまま WebAssembly 実行系に持ち込むことを目標にする。

ただし、すべての機能を Node.js API に逃がすことは禁止する。Node.js は補助的な host としてのみ使う。標準入出力、ファイルシステム、基本的なメモリ管理、数値演算、配列、オブジェクト、文字列、関数呼び出し、制御構文などは、可能な限り生成された WASM とランタイムライブラリ上で実行する。

Node.js API が必要になる場合は、WASM 側から明示的な import として呼び出す。つまり、Node.js は「実行本体」ではなく「不足 API の host provider」である。Node.js に処理を丸投げする設計、JavaScript ソースをそのまま Node に渡す設計、WASM の中に JS 実行のふりをした薄い wrapper だけを置く設計は、このプロジェクトの目的に反する。

## 目標

このプロジェクトの中心目標は、TypeScript を `.wasm` に変換し、WASI 環境または最小限の Node.js host 環境で実行できるようにすることである。

最終的には、Node.js API に依存しないコードは iwasm 上で直接実行できるようにする。たとえば、標準入力を読み、計算し、標準出力に書く CLI プログラムは、Node.js を介さず `.wasm` 単体で動作することを目指す。一方で、高度な OS 情報、Node 固有の path 解決、非同期 I/O、ネットワーク、タイマーなど、WASI だけでは十分に扱えない API は、Node.js host を明示的に併用する。
`process.argv` や `process.env` の基本的な読み取りは WASI args / environ に対応付けるが、Node.js と完全に同じ `process` object の挙動までは standalone WASI の保証対象にしない。

このプロジェクトでは、TypeScript の型情報を単なるコメントとして捨てるのではなく、最適化・診断・変換戦略に利用する。TypeScript の型は実行時には消えるが、コンパイル時には豊富な情報を与える。これを使って、数値演算、配列アクセス、オブジェクト形状、関数呼び出し、クラス、ジェネリクスの単相化、不要な runtime check の削減などに活かす。

## 非目標

このプロジェクトは、TypeScript 互換の別言語を作るものではない。構文を都合よく変えたり、JavaScript の面倒な意味論を削ったり、既存 TypeScript コードを書き直さないと動かない前提にしたりしない。

また、初期段階で対応できない機能が出たとしても、それを仕様から削除したことにはしない。未対応機能は `unsupported`、`planned`、`blocked-by-runtime`、`requires-host-api` のように状態を分けて管理する。これは機能を減らすためではなく、実装順序と失敗理由を明確にするためである。

さらに、性能を捨てる設計は採用しない。初期実装が遅くてもよいが、測定しない、比較しない、改善できない構造にすることは避ける。Node.js や Bun に常に勝てるとは限らないが、数値計算、短命 CLI、標準入出力中心のプログラムでは、最終的に Node.js / Bun より速い実行を目指す。特に、型が静的に確定するコード、配列と数値演算が中心のコード、GC 負荷が小さいコード、host API 呼び出しが少ないコードでは、WASM backend が明確に勝つことを性能目標にする。

## 禁止事項

Node.js 互換 API を一部提供することと、Node.js runtime を WASM 上に載せることは別である。
本プロジェクトが許容するのは前者だけであり、後者は明確に非目標である。

このプロジェクトは、WASM 上に Node.js ランタイムや JavaScript engine を丸ごと載せるものではない。
生成物の中心は、TypeScript/JavaScript の実行意味論を compiler が lowering した generated wasm である。
WASM 側 runtime は JS 値表現と標準的な意味論を支えるための最小 runtime に限定する。
Node.js は、WASI で表現できない外部 API を提供する host provider としてのみ使う。

すべてを Node.js で処理することは禁止する。Node.js は host API provider としてのみ使う。生成された `.wasm` が実質的に何もせず、Node.js 側に JavaScript ソースを渡して実行する構造は認めない。

JavaScript を文字列として保持し、実行時に `eval`、`Function`、Node.js VM、外部 JS engine に渡すだけの実装も認めない。それは TypeScript to WASM transpiler ではなく、WASM 起動 wrapper である。

テストを通すために仕様を緩めることも禁止する。失敗するテストは失敗として記録し、未対応なら未対応として記録する。`skip` を増やして coverage が良くなったように見せることは禁止する。

性能を諦める提案も採用しない。互換性のために初期実装が遅くなることは許容するが、性能測定をしない、改善余地を潰す、すべて runtime call に逃がす、型情報を使わない、という設計は避ける。

最終的な到達対象から機能を安易に外す提案は採用しない。
初期対応の順序を決めること、未対応機能を `unsupported` として管理すること、実装段階を分けることは許容する。初期対応の順序を決めることと、機能を捨てることは別である。`async/await`、class、module、exception、builtin、Node API 互換などは段階的に扱うが、プロジェクトの目標から外さない。

## まとめ

このプロジェクトは、TypeScript の資産を WebAssembly 実行環境へ持ち込むための transpiler である。AssemblyScript のように TypeScript 風の別言語へ寄せるのではなく、QuickJS / Javy のように JS engine を同梱して JavaScript を解釈実行する方向にも寄せすぎない。

Node.js は使うが、逃げ道にはしない。WASI で動くものは WASI で動かし、Node.js が必要なものは host API として明示する。テストは緩めず、失敗理由を分類する。性能は捨てず、最初から測る。機能は削らず、段階的に到達する。

この方針なら、「TypeScript の巨大な資産を活用したい」「しかし Node.js に閉じ込められたくない」「iwasm で動く WASM を生成したい」という原案の価値を保ったまま、研究プロジェクトではなく実装可能な処理系プロジェクトとして進められる。
