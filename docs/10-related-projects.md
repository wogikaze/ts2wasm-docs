# Related projects

この文書は related projects を競合ではなく比較対象として整理する。

## Projects

関連プロジェクトは、競合というより比較対象として扱う。特に QuickJS と AssemblyScript は重要だが、どちらもこのプロジェクトと完全には一致しない。

| Project        | 概要                        | 強み                           | このプロジェクトとの差分                                  |
| -------------- | ------------------------- | ---------------------------- | --------------------------------------------- |
| QuickJS        | 小型 JavaScript engine      | JS 互換性が高い                    | JS を WASM にトランスパイルするのではなく、JS engine を動かす方向    |
| AssemblyScript | TypeScript 風構文から WASM を生成 | WASM 向けに設計されている              | TypeScript / JavaScript 完全互換ではなく、サポート範囲が別言語寄り |
| Emscripten     | C/C++ から WASM             | runtime / libc / JS glue が成熟 | TypeScript 入力ではない                             |
| wasm-bindgen   | Rust と JS の橋渡し            | JS interop が強い               | TypeScript を WASM にする compiler ではない           |
| Javy           | JS を WASM で実行             | WASI 上で JS を動かせる             | transpiler というより JS runtime 同梱に近い             |
| tsc            | TypeScript 公式 compiler    | 仕様挙動の基準                      | 出力は JS であり WASM ではない                          |
| TypeScript-Go  | TypeScript 実装の再構成         | parser/checker 実装の参考         | WASM backend を目的としているわけではない                   |

このプロジェクトの立ち位置は、AssemblyScript より TypeScript 互換に寄せ、QuickJS より compiler / transpiler に寄せる位置にある。つまり、「TypeScript 風の WASM 言語」でも「WASM 上の JS interpreter」でもなく、「TypeScript / JavaScript の実行意味論を保ったまま WASM に落とす compiler」を目指す。

## Comparison axes

| Axis | Question |
|---|---|
| Input language | TypeScript/JavaScript そのものか、TS-like subset か |
| Execution model | compiler output か、JS engine/runtime 同梱か |
| WASM independence | generated wasm がどれだけ自律的に動くか |
| Node dependency | Node API に逃がす量 |
| Compatibility | JS/TS 意味論の再現度 |
| Performance strategy | AOT optimization か runtime/JIT/interpreter か |
| Host interface | WASI / Node host / Component Model / WIT |
