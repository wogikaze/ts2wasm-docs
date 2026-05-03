# Testing and coverage

この文書は、テスト分類、coverage state、oracle、差分実行、skip policy を扱う。

## テスト方針

テストは緩くしない。ただし、最初から test262 や TypeScript 全体を全 pass できる前提にも置かない。重要なのは、失敗を曖昧にしないことである。

すべてのテストは `docs/11-shared-definitions.md` の test status schema に従って分類する。単なる skip は禁止する。未対応機能による除外には issue ID または tracking label を必ず付ける。これにより、coverage が増えているのか、ただ skip が増えているのかを区別できる。

## Coverage State

Coverage は、単なる数字ではなく、どの意味論領域をどれだけ通過しているかを示す。test262、TypeScript compiler tests、tsc parser/checker、TypeScript-Go は、それぞれ役割が違う。

| テスト資産              | 目的                         | このプロジェクトでの使い方                      |
| ------------------ | -------------------------- | ---------------------------------- |
| test262            | JavaScript 仕様互換            | JS runtime semantics の検証           |
| TypeScript tests   | TS 構文・型・emit の互換           | parser / checker / transform の差分検出 |
| tsc parser/checker | TypeScript 公式挙動の oracle    | 自前 frontend の比較対象                  |
| TypeScript-Go      | 実装構造・高速化方針の参考              | parser/checker 設計の比較対象             |
| 独自 fixtures        | WASM / WASI / host API の検証 | このプロジェクト固有の回帰テスト                   |

test262 は巨大なので、最初から全件 pass を要求するのではなく、対象領域ごとに shard を切る。たとえば、`language/expressions`、`built-ins/Array`、`built-ins/String`、`built-ins/Number`、`built-ins/Promise` のように分ける。各 shard には canonical test status schema に従った現在状態を記録する。

TypeScript tests は、構文と型の互換性を見るために使う。`tsc` と同じ診断を最初から完全再現する必要はないが、parser の AST 差分、checker の symbol 差分、emit 前 IR の差分は追跡する。

Coverage dashboard は、少なくとも suite、shard、semantic area、target、status を軸に集計する。運用上の実測 dashboard は `artifacts/coverage/reference-coverage-matrix.md` を正とする。`build_pass` / `semantic_pass` と `unsupported` / `blocked` は別列・別グラフで追い、未対応の増加を coverage 改善として扱わない（`docs/15-coverage-matrix.md` の Metric Definitions に従う）。

### TypeScript coverage label ownership

`tsc` / `tsgo` labels are triage labels, not implementation boundaries. Each TypeScript-specific unsupported record must map to the parse / erase / emit categories in `docs/05-compatibility-and-semantics.md` before it becomes implementation-ready.

| Coverage label | Coverage status guidance | Required tracking |
|---|---|---|
| `type-alias`, `type-annotation`, `type-assertion`, `type-system` | `unsupported` with `UnsupportedTypeScriptSyntax` until the parser/erasure slice accepts and erases the form | TypeScript erasure issue |
| `ambient-declaration`, `declaration-emit` | `unsupported` with `UnsupportedTypeScriptSyntax` for declaration-only syntax or `UnsupportedModule` for module-shaped declaration effects | Ambient/declaration emit child issue |
| `import-export`, `module-resolution`, `module-system-amd` | `unsupported` with `UnsupportedModule` when the blocker is graph shape, module kind, or resolver policy | Module graph/emit issue |
| `class-accessor`, `decorator`, `jsx`, `enum` | `unsupported` with `UnsupportedTypeScriptSyntax` until the TS transform is defined; after transform, remaining JS behavior uses the normal runtime label | Transform issue plus runtime issue only if needed |
| `typescript-directive` | `unsupported` or `skip-with-reason` only when the directive is an environment/compiler-mode constraint; otherwise classify by the syntax it affects | Directive/harness issue |
| `parser-syntax`, `unknown-unsupported` | Temporary triage labels only; representative cases must be split into a concrete label above | Triage issue with evidence |
| `arguments-object`, `runtime-subset`, `class`, `object-literal` | Not TypeScript-only after transform; use `UnsupportedRuntimeSubset` or existing JS feature diagnostics | JS semantics issue |

Reference coverage must not count a TypeScript-only parse success as `build_pass` unless the erased or transformed executable program reaches the normal build pipeline. Declaration-only success is a parser/transform result, not semantic parity.

### テスト分類運用

- `parser_smoke`: 構文受理や parser/解決の最小確認。意味論適合は保証しない。
- `build_smoke`: `ts2wasm build` が成功し wasm が生成されることを確認。  
  ここで成功した件数は `build_pass` のみを増やす。
- `semantic_diff`: Node.js と `iwasm` 実行結果を比較。  
  ここで一致した場合のみ `semantic_pass` に計上する。

### 今回の再分類の運用ルール

- `m6_builtin_methods.rs`, `m7_control_flow.rs`, `m8_oop_classes.rs`, `m9_modules.rs`, `m10_node_apis.rs` の fixture は build smoke として扱う。
- これらの class/module/node-api については、意味論互換は `m2_node_diff.rs` の differential テストで明示していない限り主張しない。
- class/module/node-api semantic gap は、`current-state.md` と issue 追跡で「未達領域」として管理する。

## 追加設計: required test classes

| Test | 内容 |
|---|---|
| differential execution | 同じ JS/TS を Node/Bun/ts2wasm で実行し stdout/stderr/exit code を比較 |
| semantic golden test | `NaN`, `-0`, `undefined`, `null`, `==`, `===`, truthiness などを固定 |
| runtime ABI test | compiler と runtime helper の ABI 破壊を検出 |
| capability manifest test | 使った API と manifest の一致を確認 |
| host-deny test | Node host なしで動くべきものが Node import を要求しないか確認 |
| unsupported diagnostic test | 未対応機能が panic ではなく理由付き診断になるか確認 |
| fuzzing | parser/lowering/runtime helper のクラッシュ検出 |
| performance regression test | `docs/11-shared-definitions.md` の benchmark policy に従って特定 benchmark の悪化を検出 |

## Initial test262 shard candidates

| Priority | Shard |
|---:|---|
| 1 | `language/expressions` |
| 1 | `language/statements` |
| 1 | `built-ins/Number` |
| 1 | `built-ins/Boolean` |
| 2 | `built-ins/String` subset |
| 2 | `built-ins/Array` subset |
| 3 | `built-ins/Object` minimal subset |

## Deferred test areas

| Area | Reason |
|---|---|
| Promise | event loop / job queue が必要 |
| async/await | Promise と host scheduling が必要 |
| Proxy | object semantics と最適化を破壊しやすい |
| RegExp full compatibility | runtime 実装コストが大きい |
| Intl | data/runtime dependency が大きい |
| WeakMap / WeakSet | GC semantics が必要 |
| Atomics / SharedArrayBuffer | threading model が必要 |
| eval / Function constructor | AOT compiler と相性が悪い |
