# ts2wasm Roadmap

大きな未実装領域を W0-W8 で整理する。

このファイルは **epic 一覧**であり、個別の実装作業は `TRACKING.yaml` で管理する。

## Roadmap model

W0-W8 は厳密な直列フェーズではない。

- **W0-W5**: 主な実装レーン
- **W6-W7**: 横断的な検証レーン（W2-W5 の各実装と同時に進む）
- **W8**: 後回しの最適化領域（semantic coverage が安定した後）

```txt
TRACKING.yaml   = 今週の作業台帳（最大 20 件、active 1 件）
docs/roadmap.md = 大きな未実装領域の分類
artifacts/coverage/ = 診断データ（自動生成、TRACKING.yaml には書かない）
```

## Rules

- Roadmap item は epic であり、そのまま `TRACKING.yaml` にコピーしない。
- `TRACKING.yaml` には、Roadmap から手で切り出した小さな作業だけを入れる（1 PR / 1 agent session の粒度）。
- coverage / reference-coverage / dashboard の結果から `TRACKING.yaml` を自動生成しない。
- `TRACKING.yaml` の item は acceptance command と done evidence を必須にする。
- `TRACKING.yaml` の active item は最大 1 件。

## Gate overview

| Gate | Meaning | Main waves |
|------|---------|------------|
| Gate A | Stable runtime substrate | W0 |
| Gate B | Standalone WASI execution | W1 |
| Gate C | Parser does not block common fixtures | W2 |
| Gate D | Known names and builtin dispatch are explicit | W3 |
| Gate E | Core runtime builtins are implemented or precisely rejected | W4 |
| Gate F | JS/TS runtime semantics become coherent | W5 |
| Gate G | test262 ramp is measurable and regression-safe | W6 |
| Gate H | Host capability boundary is auditable | W7 |
| Post-Gate H | Optimization and large backend replacement | W8 |

---

## W0: Runtime substrate / ABI baseline（Gate A precondition）

Goal: generated WASM should have a stable runtime substrate before adding large JS semantics.

This wave is about **minimum correctness and maintainability**, not full optimization.
W8 handles the post-90% full replacement.

- [X] Minimal binary WASM emitter path for simple programs — id 112 done
- [ ] Runtime core raw WAT reduction where it blocks maintainability
- [X] ABI contract document: logical values vs wire representation — id 115 done
- [X] ABI mismatch tests for `i64` logical value vs `i32` wire handle/value representation — id 116 done
- [X] Runtime value representation smoke tests under iwasm/WAMR — id 117 done

Non-goals:
- No full WAT replacement — that is W8.
- No optimization passes — that is W8.
- No post-90% binary emitter rewrite — that is W8.

---

## W1: Standalone WASI baseline

Goal: generated WASM should run as a standalone WASI program where the supported host capabilities are explicit.

W1 is the **initial baseline gate**. Ongoing host capability auditing belongs to W7 (cross-cutting validation track).

- [ ] WASI args support: `args_get` / `args_sizes_get`
- [ ] WASI env support: `environ_get` / `environ_sizes_get`
- [X] WASI `proc_exit` clean exit — routed through HostImport system during issue 128
- [ ] WASI clock resolution: `clock_res_get`
- [ ] Standalone iwasm/WAMR smoke test coverage
- [ ] Capability manifest entries for every new WASI import

Non-goals:
- No implicit Node.js compatibility layer.
- No broad `process` emulation.
- No hidden host import expansion.

---

## W2: Syntax acceptance and precise rejection

Goal: parser should accept or precisely reject common JS/TS syntax without generic parse failures.

W2 = "parse or precise reject". Runtime semantics belong to W4/W5.

- [X] RegExp literal flags: `g`, `i`, `m` — id 109 done
- [X] RegExp literal flags: `s`, `u`, `y` — id 110 done (d flag rejected with precise diagnostic)
- [X] SequenceExpression — already implemented, id 118
- [X] Optional chaining — already implemented (OptionalMember/OptionalCall/OptionalIndex in AST)
- [X] Nullish coalescing — already implemented (BinaryOp::NullishCoalesce)
- [X] `yield` / generator syntax parsing — already compiles
- [X] `async` / `await` syntax parsing — already compiles
- [X] `with` / `debugger` — precise diagnostic added, id 125
- [X] Annex B block-level function hoisting syntax handling — id 126
- [X] Cover initializers — id 124
- [X] Labelled function declarations — id 126
- [ ] JSX parsing
- [ ] Decorator parsing
- [ ] TypeScript parameter property parsing
- [ ] Parser-specific regression tests

Non-goals:
- Parser support does not imply runtime semantics.
- `async`, `yield`, decorators, JSX may initially lower to explicit unsupported diagnostics.
- Do not combine syntax parsing with runtime implementation unless the feature is tiny.

---

## W3: Name/call resolution and builtin dispatch

Goal: unresolved names/functions should become either known supported operations or precise unsupported diagnostics.

This wave should reduce `UnresolvedName` / `UnresolvedFunction` noise without pretending unsupported runtime semantics exist.

- [X] Register core ECMAScript global builtin names:
  [x] `Symbol`, `Proxy`, `Reflect`, `Promise` (id 101 done)
  [x] `ArrayBuffer`, `DataView` (id 102 done)
  [x] `WeakMap`, `WeakSet`, `Atomics`, `Intl`, `globalThis`, `AggregateError`, `URIError`, `EvalError` (done)
  [x] `Map`, `Set`, `Error`, `TypeError`, `RangeError`, `ReferenceError`, `SyntaxError` (already registered)
- [X] Register TypedArray constructor names:
  [x] `Int8Array` through `BigUint64Array` (11 types) — id 102 done
- [X] Register well-known symbols:
  [x] `Symbol.iterator`, `toStringTag`, `hasInstance`, `toPrimitive`, `for`, `keyFor` — id 103 done
- [X] Builtin method dispatch table — already complete (program_builtins.rs)
- [X] String / Array / Object / Number / Function.prototype method dispatch routing — already complete
- [ ] Nested namespace/module resolution: `A.B.C`
- [ ] Type-only imports
- [ ] Triple-slash directives
- [ ] Module augmentation

Non-goals:
- Name registration does not imply runtime implementation.
- Unsupported builtins should route to precise diagnostics.
- Avoid "complete all methods" as one work item.

---

## W4: Builtin API semantics

Goal: implement selected builtins only after names and dispatch paths are explicit.

See `docs/language-reference/` for the detailed feature coverage tables.

- [ ] Promise minimal substrate and constructor
- [X] `Promise.prototype.then` / `catch` / `finally` — precise unsupported diagnostic added (id 104 done)
- [ ] `Promise.resolve` / `reject` / `all` / `race` / `allSettled` / `any` / `withResolvers`
- [X] Proxy constructor — [x] precise unsupported diagnostic (id 106 done)
- [ ] Proxy handler trap slices + `Proxy.revocable`
- [X] Reflect API — [x] precise unsupported diagnostic (id 106 done)
- [ ] TypedArray constructors by family + basic read/write
- [ ] ArrayBuffer / SharedArrayBuffer / DataView
- [ ] WeakMap / WeakSet
- [ ] Symbol constructor + well-known symbol runtime behavior
- [ ] Atomics / Intl
- [ ] `String.prototype.replace` / `replaceAll` / `matchAll`
- [ ] `Array.prototype.sort` / `reduceRight`
- [X] `Array.prototype.reduce` — [x] already works via array-like routing (id 105 done)
- [ ] Upgrade selected existing builtins from build_smoke to semantic_diff

Non-goals:
- Do not implement whole builtin families as single work items.
- Do not add async/await lowering here — that is W5 (after Promise substrate exists).
- Do not implement Proxy/Reflect fully before object model invariants are clear.

---

## W5: Language runtime semantics

Goal: runtime behavior should match the supported JS/TS subset, not only parse or build.

W4 = API surface / builtin method behavior.
W5 = language execution model / control flow / object model invariants.

- [ ] Iterator protocol: Array/String/Map/Set iterators + `Iterator.prototype`
- [ ] Well-known symbol runtime wiring: `Symbol.iterator`, `hasInstance`, `toPrimitive`, `toStringTag`
- [ ] Proper completion records for `if`/`switch`/`try`/`for`/`while`/`break`/`continue`/`return`/`throw`
- [ ] Generator functions: `function*`, `yield`, `yield*`, `Generator.prototype`
- [ ] Promise-backed async/await lowering
- [ ] Async generators + `for-await-of`
- [ ] Object model completeness: property descriptors, `seal`/`freeze`, `[[GetPrototypeOf]]`/`[[SetPrototypeOf]]`
- [ ] ES module live binding updates + module namespace objects
- [ ] Dynamic import + circular dependency evaluation
- [ ] Mutable capture environments for escaping closures
- [ ] `this` binding: global this, strict-mode receiver, method receiver

Non-goals:
- Do not use W5 as a dumping ground for parser gaps.
- Do not start generators or async before completion records are reliable.
- Do not implement object model invariants partially without diagnostics.

---

## W6: Coverage and regression infrastructure

Goal: coverage growth should be measurable, reproducible, and regression-safe.

This is a **cross-cutting validation track**, not a sequential phase.
Work here runs in parallel with W2-W5 implementation: each feature change should be accompanied by coverage measurement, regression detection, and delta reporting.

- [X] Ramp 500 → 2,000 with stable parallel execution and caching — already works (--jobs N flag)
- [ ] Ramp 2,000 → 10,000 / 10,000 → 30,000 / 30,000 → 53,445
- [X] Regression detection: fail on build_pass / semantic_pass decrease — [x] `--record-baseline` / `--compare-baseline` flags (id 107 done)
- [X] Delta reporting: feature-level and diagnostic-class pass/fail deltas — id 128
- [ ] Coverage dashboard: trend graph, feature-level burn-down, diagnostic burn-down
- [ ] Gate progress visualization

Non-goals:
- Do not auto-create `TRACKING.yaml` entries from coverage gaps.
- Do not treat coverage summaries as issue lists.
- Do not block normal build on dashboard generation.

---

## W7: Host capability boundary

Goal: every host interaction should be explicit, auditable, and deny-testable.

This is a **cross-cutting validation track** that protects the core project constraint:
generated WASM should not silently depend on Node.js or hidden host capabilities.

- [X] Full host import audit with `--emit-manifest` — id 129
- [ ] Manifest golden tests for all supported fixtures
- [ ] Host-deny test matrix expansion
- [ ] Standalone assurance for new features (Promise, Proxy, Reflect, TypedArray, WASI args/env)
- [X] Capability review checklist in coding standard — [x] already documented (id 113 done)
- [ ] CI gate for unexpected host imports

Non-goals:
- No broad host shim.
- No hidden Node.js delegation.
- No "temporary" imports without manifest entries.

---

## W8: Optimization and backend replacement（deferred）

Goal: after semantic coverage is high enough, improve speed, size, maintainability, and backend quality.

**Entry condition**: semantic_diff coverage is stable, host capability boundary is audited,
test262 ramp is regression-safe, major runtime semantics are no longer moving daily.

- [ ] Benchmark suite + performance regression tracker
- [ ] Typed fast path / packed array / devirtualization
- [ ] Property inline cache / dead code elimination / constant folding / inlining
- [ ] Full binary WASM emitter replacing WAT text dependency
- [ ] Replace giant WAT templates with typed writers
- [ ] ABI bridge cleanup after logical/wire contract is stable

Non-goals:
- Do not start broad optimization before semantic correctness.
- Do not replace WAT wholesale while runtime semantics are still unstable.
- Do not treat W8 as a prerequisite for early test262 growth.
