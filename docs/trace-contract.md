# Trace Contract

Coverage PR は実行結果だけでなく、正しい経路を通った証拠として trace を提出する。

## Trace Field Schema

各 trace event は最低限次の fields を持つ:

| Field | Required | Description |
|-------|----------|-------------|
| `trace_kind` | 必須 | SemanticIRTrace / SpecOpTrace / RuntimeCoreTrace / DeoptTrace |
| `event` | 必須 | event identifier |
| `span` / `source_location` | 必須 | source location, runtime-only は `runtime` |
| `op` | 必須 | operation name |
| `inputs` | 必須 | operation inputs |
| `outputs` | 必須 | operation outputs |
| `frame` / `frame_state` | deopt/runtime では必須 | frame state reference |
| `realm` | realm-sensitive では必須 | realm reference |
| `object_kind` | object internal method では必須 | Ordinary / Proxy / ArrayExotic / ... |
| `shape` | shape-sensitive では必須 | shape id |
| `result_status` | 必須 | normal / throw / abrupt / deopt |

## Trace Kinds

| Trace Kind | 生成元 | 内容 | 必須条件 |
|-----------|--------|------|---------|
| `SemanticIRTrace` | `semantic-ir` | CFG block 列、Reference 解決、GetValue/PutValue 呼び出し | P1 実装後 |
| `SpecOpTrace` | `spec-kernel` | SpecOp 呼び出し順序、object kind dispatch 結果 | P2 実装後 |
| `RuntimeCoreTrace` | `runtime-core` | Shape 遷移、PropertyDescriptor 操作、Env record 操作 | P0 実装後 |
| `DeoptTrace` | `opt-mir` | Guard 挿入、Guard 破壊、DeoptToBaseline 発火 | P4 実装後 |

## Trace 記法

trace は `->` で操作を連結する。

```
ToPropertyKey -> GetOwnProperty -> OrdinaryGetOwnProperty
```

Proxy 経由:
```
ToPropertyKey -> Get -> ProxyGet -> OrdinaryGet
```

Guard/deopt:
```
GuardShape { expected: S1 } -> GuardFailure(S1→S2) -> DeoptToBaseline -> Get -> OrdinaryGet
```

## Required Trace Samples

### ordinary property get

```js
let o = { x: 1 };
return o.x;
```

Expected SpecOpTrace:
```
ToPropertyKey("x") -> Get(object=o, key="x") -> OrdinaryGet -> return 1
```

Expected RuntimeCoreTrace:
```
ShapeLoad(o, S1) -> ShapeFind(o, "x") -> offset=8 -> Load(inline[8]) -> return 1
```

### ordinary property set

```js
let o = {};
o.x = 1;
```

Expected SpecOpTrace:
```
ToPropertyKey("x") -> Set(object=o, key="x", value=1) -> OrdinarySet -> DefineOwnProperty or SetExistingProperty -> return true
```

### accessor getter

```js
let o = { get x() { return 1 } };
o.x;
```

Expected SpecOpTrace:
```
ToPropertyKey("x") -> Get(object=o, key="x") -> GetOwnProperty -> accessor descriptor found -> Call(getter=get, receiver=o) -> return 1
```

### Proxy get

```js
let p = new Proxy({}, { get(t, k, r) { return 2 } });
p.x;
```

Expected SpecOpTrace:
```
ToPropertyKey("x") -> Get(object=p, key="x") -> ObjectKind::Proxy -> ProxyGet(target={}, handler={get:...}) -> Call(trap=get, args=[{}, "x", p]) -> invariant check -> return 2
```

### function call

```js
function f(x) { return x; }
f(1);
```

Expected SpecOpTrace:
```
Call(callee=f, this=undefined, args=[1]) -> OrdinaryCall -> EnterContext -> EvaluateBody -> Return(result=1) -> LeaveContext
```

### constructor call

```js
function C(v) { this.x = v; }
new C(1);
```

Expected SpecOpTrace:
```
Construct(constructor=C, args=[1], newTarget=C) -> OrdinaryConstruct -> InitializeThisBinding -> Call(body) -> Return(this)
```

### throw / catch

```js
try { throw 1; } catch(e) { e; }
```

Expected SpecOpTrace:
```
Throw(value=1) -> ExceptionCarrier set -> catch edge -> GetBindingValue(env, "e") -> return 1
```

### try / finally

```js
try { 1; } finally { 2; }
```

Expected SpecOpTrace:
```
MaterializeCompletion(Normal, 1) -> EnterFinally -> EvaluateFinally(2) -> ResumeCompletion -> return 1
```

### deopt

```js
let o = { x: 1 };
o.x;
Object.defineProperty(o, "x", { get() { return 2 } });
o.x;
```

Expected DeoptTrace (first access):
```
GuardShape(o, S1) -> ShapeLoad(o, x) -> return 1
```

Expected DeoptTrace (after defineProperty):
```
GuardShape(o, S1) -> GuardFailure(shape mutated) -> FrameState(o={...}) -> DeoptToBaseline -> Get -> GetOwnProperty -> accessor descriptor found -> Call(getter) -> return 2
```

## Coverage PR Rules

1. 変更した層に対応する trace kind を記述する
2. `expected_trace` と `actual_trace` を比較する
3. 差分がある場合は `coverage-classification.py` で分類する
4. `native_lowered.rs` / `typed.rs` 経路の trace は証明として認めない
5. coverage PR には必ず coverage-classification record が必要
