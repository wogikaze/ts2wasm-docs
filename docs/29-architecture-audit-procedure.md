# Architecture Audit & Best-Practice Alignment Procedure

目的は、test262 coverage pressure が旧 `backend-wasm` / `LoweredProgram` / `RuntimeFn` 経路へ戻らず、必ず `classification -> semantic-ir / spec-kernel / runtime-core / backend-correctness / opt-mir` の正しい owning layer へ流れる状態を維持すること。

この手順書は、P0a Runtime Core 以降の開発で毎回使う監査・実装・レビュー手順を定義する。coverage rate を直接上げることを主目的にしない。coverage 改善は、分類済み failure を正しい層の未実装として処理した結果としてのみ認める。

## 1. 基本原則

ts2wasm は TS subset compiler ではなく、すべての JS/TS を高性能に wasm で動かす compiler + runtime system である。したがって、構造は AOT wasm、Baseline VM、Spec Kernel、Runtime Core、Opt MIR、Guard、FrameState、Deopt、slow correctness path を前提にする。

設計上の source of truth は legacy `LoweredProgram` ではない。新しい正しさの経路は `semantic-ir -> spec-kernel -> runtime-core -> backend-correctness` であり、高速化は `semantic-ir -> opt-mir -> backend-wasm` の guard/deopt 付き fast path として扱う。

`backend-wasm` は最終的に thin emitter へ寄せる。JS semantics、descriptor validation、Proxy trap ordering、iterator close、completion propagation、binding resolution を `backend-wasm` に置いてはいけない。

`RuntimeFn` は legacy catalog として扱う。新しい仕様操作は `SpecOp` に追加する。既存 `RuntimeFn` の migration は P5 の責務であり、coverage 改善 PR の逃げ道にしてはいけない。

## 2. 毎回の監査入口

監査は必ず clean tree で行う。commit hash を記録し、未コミット変更がある場合は監査結果を GO 判定に使わない。

実行する基本コマンドは次の通り。

```text
git status --short
cargo metadata --format-version=1
cargo check --workspace
cargo test --workspace --no-run
python scripts/gate/fast-gate.py --skip-nextest
python scripts/check/gate-negative-tests.py
```

cargo がない環境では cargo-dependent gate は未検証として扱う。未検証を pass に読み替えない。`fast-gate.py` が cargo を要求して fail する場合、その環境では P0/P1/P2 の GO 判定はできない。

## 3. Target DAG と exception の扱い

`arch-rules.toml` は target architecture を表す。現状の legacy 依存を正当化するための allowed DAG にしてはいけない。

legacy dependency は `architecture-exceptions.toml` にだけ置く。例外は debt であり、設計ではない。各 exception は必ず `id`, `reason`, `owner`, `expires`, `migration_issue` を持つ。file exception では `allowed_change` も必須にする。

禁止する運用は次の通り。

| 禁止事項 | 理由 |
| --- | --- |
| legacy edge を `arch-rules.toml` の `allowed_deps` に入れる | debt が正規設計になる |
| 期限なし exception | migration pressure が消える |
| owner なし exception | 責任境界が消える |
| migration_issue なし exception | 完了条件が消える |
| broad glob exception | freeze が無効化される |
| exception id なしで frozen file touch を許可 | gate が blanket allow になる |

監査時は、`arch-rules.toml` と `architecture-exceptions.toml` を同時に見る。`backend-wasm -> ir`, `backend-wasm -> resolve`, `backend-wasm -> semantics`, `backend-wasm -> runtime-catalog`, `compiler -> ir`, `cli -> backend-wasm` のような legacy edge がある場合、allowed edge ではなく期限付き exception になっていることを確認する。

## 4. Legacy freeze の運用

frozen file は原則として触らない。

最低限の frozen file は次の通り。

```text
crates/backend-wasm/src/native_lowered.rs
crates/backend-wasm/src/runtime/core/typed.rs
crates/backend-wasm/src/native_runtime_embed.rs
crates/runtime-catalog/src/runtime_fn.rs
crates/ir/src/lowered/*
```

`legacy-freeze.py` は touched-file deny でなければならない。LOC 増加禁止だけでは不十分。同じ行数内で仕様判断を追加できるため。

許可される変更は、期限付き exception id を明示した `delete`, `move`, `bugfix` のみ。`feature`, `coverage`, `helper-addition` は不可。

監査で確認する negative case はこれ。

```text
frozen file を1文字変更 -> fail
同じ行数内で変更 -> fail
例外 id なし -> fail
不正 exception id -> fail
期限切れ exception -> fail
allowed_change=feature -> fail
valid exception id + allowed_change=bugfix -> pass
```

## 5. RuntimeFn / SpecOp boundary

新しい JS semantics は `RuntimeFn` ではなく `SpecOp` に入れる。

`RuntimeFn` gate は variant count だけを見てはいけない。enum body hash、variant rename、variant reorder、variant deletion、generated catalog count との一致を検査する。count が同じでも enum body が変わった場合は fail にする。

`RuntimeFn` deprecation gate は二段階にする。P0/P1/P2 では新規増加を reject する。P5 migration-complete mode では deprecated RuntimeFn が active emission order に残ることも reject する。

coverage PR では、deprecated `RuntimeFn` 経由で新しい pass を得ることを coverage improvement として認めない。これは `coverage-classification.py` または coverage PR checker 側で検出する。

## 6. SpecOp completeness

`SpecOp` を追加したら、次のすべてが揃っていなければならない。

| 必須項目 | 目的 |
| --- | --- |
| `SpecOp` enum variant | 仕様操作の識別 |
| `param_count` / `result_count` metadata | ABI と trace の安定化 |
| dispatch arm | unimplemented でも明示 |
| backend builder mapping | wasm module emission の可視化 |
| trace rendering | coverage evidence として使う |
| self-test negative case | false green 防止 |

`SpecOp` に関する wildcard match は原則禁止。`_ =>` で新 variant を吸収できるなら gate ではない。未実装なら wildcard ではなく、該当 variant の明示 arm から `Unimplemented` を返す。

`backend-wasm/spec_emit` や `backend-correctness` で `_ => None` により unsupported `SpecOp` を silent drop してはいけない。未対応なら明示 error にする。placeholder lowering は P0a/P1 の scaffold として存在してよいが、correctness evidence として使ってはいけない。

## 7. Coverage classification

test262 coverage PR は必ず classification record から始める。pass rate が上がっただけでは merge しない。

strict mode では、最低限これを reject する。

```text
failure_kind = Unclassified
status = unclassified
expected_trace = ""
actual_trace = ""
evidence = ""
linked_issue = ""
unknown owning_layer
unknown failure_kind
OptimizationGap を coverage blocker として扱う record
skip / expected-fail 更新だけの coverage improvement
```

classification record の required fields は次を固定する。

```text
suite
case
failure_kind
owning_layer
first_missing_capability
required_change
expected_trace
actual_trace
status
evidence
linked_issue
```

failure kind は最低限これを持つ。

```text
ParseGap
ResolveGap
SemanticIRGap
SpecOpGap
RuntimeCoreGap
CorrectnessBackendGap
LegacyBackendLeak
OptimizationGap
HarnessGap
OracleGap
Unclassified
```

`LegacyBackendLeak` は fix target ではなく design violation として扱う。`OptimizationGap` は coverage blocker ではなく performance issue として別管理する。

## 8. Trace contract

coverage evidence として使える trace contract を固定する。

必須 trace kind は次の4つ。

```text
SemanticIRTrace
SpecOpTrace
RuntimeCoreTrace
DeoptTrace
```

minimum sample は次を含める。

```text
ordinary property get: ToPropertyKey -> Get -> OrdinaryGet
ordinary property set: ToPropertyKey -> Set -> OrdinarySet
accessor getter: Get -> GetOwnProperty -> Call(getter)
Proxy get: Get -> ProxyGet -> Call(trap) -> invariant check
function call: Call -> OrdinaryCall -> EnterContext -> Return
constructor call: Construct -> OrdinaryConstruct -> InitializeThisBinding
throw/catch: Throw -> exception carrier -> catch edge
try/finally: abrupt completion materialize -> finally -> resume
deopt: GuardShape fail -> DeoptToBaseline -> FrameState restore -> SpecOp
```

trace は docs にあるだけでは不十分。coverage classification の `expected_trace` / `actual_trace` がこの contract を参照できるようにする。

## 9. Fast gate / manager への統合

checker は存在するだけでは gate ではない。標準開発経路で実行されなければ制約にならない。

`fast-gate.py` は最低限これを実行する。

```text
architecture-rules.py
crate-dag.py
check-arch-dag.py --reject-increase
legacy-freeze.py
runtimefn-invariants.py or equivalent
check-runtimefn-deprecation.py --reject-increase
specop-dispatch.py
coverage-classification.py --self-test
coverage-classification.py --strict fixtures/gate/coverage-classification-valid.json
coverage-classification.py --strict fixtures/gate/coverage-classification-bad.json must fail
trace-contract.py --self-test
architecture-exceptions.py
docs-routing.py
compiler-source-truth.py
gate-negative-tests.py
```

`gate-negative-tests.py` は必須。positive test だけでは false green を検出できない。negative tests は、少なくとも以下を含む。

```text
bad exception id
frozen file touch
fake RuntimeFn variant
fake SpecOp variant without dispatch
SpecOp wildcard absorption
missing SpecOp metadata
missing SpecOp builder
Unclassified classification
missing evidence
missing linked_issue
empty trace
legacy dependency added without exception
docs old-routing phrase
```

## 10. Compiler default / source of truth

compiler default path を監査する。`BuildPipelineOptions::default()` が新経路を source of truth にしているか確認する。

許容される状態は次のどちらか。

1. default が `spec_kernel_mode = Strict` で、fallback なしに new path を使う。
2. legacy default を残すが、coverage PR / new path gate / dashboard では legacy pass を coverage gain として扱わない。

望ましいのは1。2を選ぶ場合、legacy pass count と new path pass count を dashboard で分ける。legacy-only pass は新設計 coverage とはみなさない。

`compiler` は orchestration に留める。builtin semantics、property descriptor logic、Proxy trap order、iterator close、completion propagation を `compiler` に置いてはいけない。

## 11. P0a Runtime Core の範囲

P0a で許可するもの。

```text
Value representation skeleton
TaggedValue / ValueKind
heap object header
ObjectRef
ShapeId
PropertyKey
PropertyDescriptor
DescriptorAttributes
ordinary object allocation primitive
shape transition skeleton
environment record skeleton
RealmId / intrinsics table skeleton
ExecutionContext skeleton
ExceptionCarrier
FrameState schema
JobQueue skeleton
Baseline VM container shell
RuntimeCoreTrace hook
```

P0a で禁止するもの。

```text
test262 coverage improvement を主目的にする
SpecOp を大量実装して pass rate を上げる
builtin method helper を大量追加する
native_lowered.rs を触る
typed.rs を触る
native_runtime_embed.rs を触る
RuntimeFn を追加する
backend-wasm に JS semantics を追加する
opt-mir fast path を実装する
deopt 本体を実装する
baseline VM instruction set を広げる
```

P0a の成功条件は coverage rate ではない。P1/P2/P3/P4 が載る runtime substrate を安全に定義できたことを成功条件にする。

## 12. PR workflow

すべての PR は次を埋める。

```text
Change type
Owning layer
Affected crates
test262 classification id
expected trace
actual trace
evidence
linked issue
touched frozen files
architecture exception id
RuntimeFn changed?
SpecOp changed?
dispatch / metadata / builder / trace added?
deopt / FrameState impact?
legacy path touched?
new path source of truth?
commands run
```

coverage PR では classification id、expected trace、actual trace、evidence、linked issue を必須にする。

SpecOp PR では dispatch、metadata、builder、trace を必須にする。

runtime-core PR では FrameState / Value / heap / shape / descriptor への影響を明記する。

legacy frozen file を触る PR は、例外 id がなければ review に入らない。

## 13. 定期監査

毎週または major PR ごとに、以下を監査する。

```text
dependency graph の重力井戸
backend-wasm の新規変更
ir の新規変更
RuntimeFn enum diff
SpecOp variant / dispatch / builder / trace の一致
coverage classification の Unclassified 件数
legacy path pass count vs new path pass count
architecture exception の期限切れ
file size の肥大化
fast-gate に含まれない checker
```

特に `backend-wasm`, `ir`, `runtime-catalog`, `compiler` の4つは毎回見る。これらが新機能の最短経路になっているなら、設計はまた崩れ始めている。

## 14. Human scorecard

人間用 scorecard は 100 点満点に固定する。hard blocker が1つでもあれば、点数に関係なく NO-GO。

| 評価軸 | 重み |
| --- | --: |
| Target DAG / exception separation | 15 |
| Legacy freeze enforcement | 10 |
| RuntimeFn / SpecOp boundary | 10 |
| SpecOp completeness gate | 10 |
| Coverage classification strictness | 10 |
| Trace contract and evidence | 8 |
| Fast gate / manager integration | 10 |
| Compiler source-of-truth separation | 10 |
| P0a scope discipline | 7 |
| Developer usability / PR routing | 5 |
| Exception hygiene | 5 |
| Total | 100 |

GO 条件は、score 85 以上、hard blocker 0、negative tests pass、cargo workspace health pass のすべてを満たすこと。

## 15. Hard blocker 定義

次のいずれかがある場合は P0a / P1 / P2 の GO を出さない。

```text
legacy dependency が allowed_deps に入っていて exception 管理されていない
frozen file touch が default gate で pass する
RuntimeFn 追加が pass する
SpecOp variant 追加時に dispatch / metadata / builder / trace 欠落が pass する
coverage-classification --strict で Unclassified / empty trace / missing evidence / missing linked_issue が pass する
backend-correctness または spec_emit が unsupported SpecOp を silent drop する
fast-gate が P0 Entry Gate checker を実行していない
compiler default / dashboard が legacy-only pass を new path coverage と混ぜる
architecture exception が期限なし、owner なし、migration_issue なし
```

## 16. 最終判定基準

設計の中心が正しく切り替わっている状態とは、次の状態を指す。

```text
test262 failure がまず classification record になる
classification が owning layer を指定する
実装は owning layer にしか入れられない
legacy backend は frozen
RuntimeFn は増やせない
SpecOp は completeness gate を通る必要がある
coverage PR は trace evidence を持つ
legacy-only pass は new path coverage として扱われない
```

この状態になっていれば、coverage pressure は architecture を壊さず、semantic floor / spec-kernel / runtime-core / correctness backend を育てる方向に働く。
