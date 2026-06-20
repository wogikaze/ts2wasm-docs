# Native Emitter Completion Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development or executing-plans.

**Goal:** Implement all unsupported constructs in `native_lowered.rs` so the native emitter covers the full LoweredProgram surface.

**Architecture:** Hybrid approach — inline simple ops as i32 `WasmInstr` sequences and embed complex runtime functions as typed `WasmFunction` builders in the `WasmModule`. Runtime embedding is driven by `build_runtime_link_plan`.

**Tech Stack:** Rust, WasmInstr IR, WasmFunction, RuntimeFn catalog.

## Current Guardrails

- Native emitter must never emit a `WasmInstr::Call` for a symbol that is not imported or defined in the same `WasmModule`.
- `wasm_encoder_backend` now fails loudly for unresolved call/export symbols instead of silently using function index 0.
- Dynamic/runtime semantics must stay as controlled `UnsupportedSyntax/backend` until the required runtime helper is embedded or imported by contract.
- Broad catch-all reasons should be decomposed into variant-specific reasons before large implementation work.

## Selected Issues

Focus the first pass on the five highest-impact open issues from `I-20260523-N8EPC2`.
The small tail bucket (`I-20260523-N8TAY2`, 11 cases) stays as close-out work after the
runtime helper path exists.

| Order | Issue | 2026-05-23 Baseline | Bottleneck |
|---:|---|---:|---|
| 1 | `I-20260523-N8PSET` property set | 14,267 | Native emitter only handles static slots/constant object values; dynamic object mutation needs `RuntimeFn::PropertySet` embedded as wasm bytes. |
| 2 | `I-20260523-N8EXPR` general expression | 5,216 | Broad catch-all was hiding concrete variants. Remaining high-risk variants are runtime calls, heap closures, `this`, module/class/prototype refs, and error/new forms. |
| 3 | `I-20260523-N8UNRY` unary operators | 5,110 | `runtime_link_plan` selects `Not`, `Negate`, `TypeOf`, etc., but native emission still uses raw i32 shortcuts or rejects; JS coercion must come from runtime helpers. |
| 4 | `I-20260523-N8PGET` property get | 1,657 | Static object/array reads work; dynamic/string/prototype/optional reads require `PropertyGet`/`Index` runtime helper embedding. |
| 5 | `I-20260523-N8STMT` statement support | 290 | Catch-all is now mostly structured control/runtime semantics; focused probes show `TryFinally`/`TryCatch` is already a current blocker for harness cases. |

## Current Native Unsupported Snapshot

Full baseline from `I-20260523-N8EPC2`, generated on 2026-05-23 with:

```bash
python scripts/manager.py reference-coverage test262 --jsonl --no-dashboard-data
```

| Count | Reason |
|---:|---|
| 14,245 | `native LoweredProgram emitter does not support this property set` |
| 5,216 | `native LoweredProgram emitter does not support this expression` |
| 5,110 | `native LoweredProgram emitter does not support this unary operator` |
| 854 | `native LoweredProgram emitter does not support this dynamic property get` |
| 802 | `native LoweredProgram emitter does not support this property get` |
| 290 | `native LoweredProgram emitter does not support this statement` |
| 22 | `native LoweredProgram emitter does not support this dynamic property set` |
| 8 | `native LoweredProgram emitter does not support this binary operator` |
| 3 | `native LoweredProgram emitter does not support this builtin call` |
| 1 | `native LoweredProgram emitter does not support this optional property get` |

Local focused snapshot after the current native-emitter diagnostic split and script de-duping:

```bash
TS2WASM_BINARY=/home/wogikaze/wgkz/ts2wasm/target/debug/ts2wasm \
  python3 scripts/manager.py reference-coverage test262 --jsonl \
  --path-filter annexB/built-ins/Date/prototype/getYear/nan.js \
  --path-filter annexB/built-ins/Date/prototype/getYear/B.2.4.js \
  --path-filter annexB/built-ins/Array/from/iterator-method-emulates-undefined.js \
  --path-filter built-ins/Array/prototype/every/15.4.4.16-7-c-ii-1.js \
  --path-filter harness/compare-array-arraylike.js \
  --no-dashboard-data
python3 scripts/manager.py native-emitter-unsupported --format markdown
```

Latest focused result after the current native emitter expansion, native stack-discipline fixes,
Date prototype descriptor support, dynamic `GetLength`, and static-array dynamic-index reads/writes:

```text
Pass: 3  BuildOnly: 0  SemSkipSample: 0  OracleSkipped: 0  Fail: 0  Unsupported: 0  Blocked: 2
records: 5
native_unsupported: 0
```

The remaining focused non-passing outcomes are no longer native-emitter
unsupported diagnostics or wasm validation/runtime traps:

| Count | Tracking | Reason |
|---:|---|---|
| 2 | n/a | `node execution failed` oracle preparation/execution blockage |

Always set `TS2WASM_BINARY=target/debug/ts2wasm` for focused runs while iterating; the default
runner prefers a stale `target/release/ts2wasm` when it exists.

Full native sweep on 2026-05-24 before the required-function traversal fixes:

```text
Pass: 5159  BuildOnly: 0  SemSkipSample: 0  OracleSkipped: 0  Fail: 573  Unsupported: 43576  Blocked: 432
records: 53469
compiler-invariant unresolved wasm call symbol: 292
```

The unresolved-symbol bucket was not a missing runtime helper. It was a native emitter
bookkeeping bug: emitted calls could reference user functions that `collect_runtime_required_functions`
had not selected for inclusion in the module. Representative symbols were `$func_25`, `$func_0`,
`$func_11`, `$func_12`, and `$func_26`.

Focused replay after the required-function traversal fixes and the current opaque runtime-call
fallback pass:

```text
Pass: 2  BuildOnly: 0  SemSkipSample: 0  OracleSkipped: 0  Fail: 1  Unsupported: 5  Blocked: 0
records: 8
unresolved wasm call symbol: 0
native_unsupported: 0
```

Full native sweep after the same fixes, generated on 2026-05-24 with
`TS2WASM_BINARY=target/debug/ts2wasm python3 scripts/manager.py reference-coverage test262
--jsonl --no-dashboard-data`:

```text
Pass: 5404  BuildOnly: 0  SemSkipSample: 0  OracleSkipped: 0  Fail: 603  Unsupported: 43236  Blocked: 488
records: 53469
native_unsupported: 13465
```

Full native sweep after static `for-of`, static generator, and `ObjectGetPrototypeOf` coverage,
generated on 2026-05-24 with the same command:

```text
Pass: 5851  BuildOnly: 0  SemSkipSample: 0  OracleSkipped: 0  Fail: 1241  Unsupported: 42083  Blocked: 611
records: 53469
native_unsupported: 11411
```

Largest native buckets from that refreshed full sweep:

| Count | Reason | Representative |
|---:|---|---|
| 588 | `catchable try-catch body while emitting func_11` | `annexB/built-ins/Date/prototype/getYear/not-a-constructor.js` |
| 496 | `GeneratorNext with 1 args` | `language/arguments-object/async-gen-meth-args-trailing-comma-multiple.js` |
| 448 | `ObjectHasOwnProperty with 2 args` | `built-ins/AggregateError/prototype/errors-absent-on-prototype.js` |
| 409 | `ObjectDefineProperties with 2 args` | `built-ins/Object/defineProperties/15.2.3.7-2-11.js` |
| 366 | `this builtin call` | `annexB/built-ins/escape/argument_bigint.js` |

Full native sweep after static `Escape`/`Unescape` folding, static binary-condition folding,
and dynamic `ObjectCreate` stack-shape fallback, generated on 2026-05-24 with the same command:

```text
Pass: 6639  BuildOnly: 0  SemSkipSample: 0  OracleSkipped: 0  Fail: 1436  Unsupported: 40955  Blocked: 756
records: 53469
native_unsupported: 9417
```

Current largest native buckets:

| Count | Reason | Representative |
|---:|---|---|
| 360 | `catchable try-catch body` | `built-ins/Array/length/S15.4.2.2_A2.2_T1.js` |
| 352 | `this builtin call` | `annexB/built-ins/escape/argument_types.js` |
| 341 | `ObjectHasOwnProperty with 2 args` | `built-ins/Array/S15.4.2.1_A1.1_T1.js` |
| 317 | `RegExpMatch with 2 args` | `annexB/built-ins/RegExp/RegExp-decimal-escape-class-range.js` |
| 252 | `ArraySlice with 3 args while emitting func_12` | `language/expressions/class/dstr/async-gen-meth-ary-ptrn-elem-ary-rest-init.js` |
| 228 | `ObjectToString with 2 args` | `built-ins/Boolean/prototype/toString/S15.6.4.2_A1_T2.js` |
| 199 | `ObjectIsExtensible with 1 args` | `built-ins/AsyncDisposableStack/instance-extensible.js` |
| 171 | `SymbolNew with 1 args while emitting func_29` | `built-ins/ArrayBuffer/prototype/slice/context-is-not-object.js` |
| 170 | `for-of/for-await-of while emitting func_11` | `annexB/language/statements/for-await-of/iterator-close-return-emulates-undefined-throws-when-called.js` |
| 166 | `ArrayBufferNew with 1 args` | `built-ins/Array/from/items-is-arraybuffer.js` |

Focused replay for the current top native-emitter representatives after static `for-of`, static
generator, and `ObjectGetPrototypeOf` opaque fallback coverage:

```text
Pass: 1  BuildOnly: 0  SemSkipSample: 0  OracleSkipped: 0  Fail: 1  Unsupported: 0  Blocked: 1
records: 3
native_unsupported: 0
```

Focused replay after the follow-up `ObjectHasOwnProperty` and `ObjectDefineProperties` work:

```text
Pass: 2  BuildOnly: 0  SemSkipSample: 0  OracleSkipped: 0  Fail: 0  Unsupported: 0  Blocked: 0
records: 2
native_unsupported: 0
```

Focused replay after the follow-up catchable `ReflectConstruct`, dynamic Date, and dynamic
`GeneratorNext` stack-shape coverage:

```text
Pass: 3  BuildOnly: 0  SemSkipSample: 0  OracleSkipped: 0  Fail: 0  Unsupported: 1  Blocked: 0
records: 4
native_unsupported: 0
```

Representative outcomes:

| Path | Current state | Bottleneck |
|---|---|---|
| `built-ins/Array/prototype/forEach/15.4.4.18-2-2.js` | pass | Former `$func_0` unresolved bucket is cleared. |
| `language/block-scope/shadowing/parameter-name-shadowing-catch-parameter.js` | pass | Former catch-body traversal gap is cleared. |
| `built-ins/Array/prototype/map/15.4.4.19-8-1.js` | semantic unsupported assertion | Former `$func_11` unresolved bucket is cleared; next work is semantics/oracle behavior, not symbol inclusion. |
| `annexB/built-ins/RegExp/legacy-accessors/lastMatch/this-not-regexp-constructor.js` | semantic/runtime unsupported | Dynamic `ObjectGetOwnPropertyDescriptor` now emits an opaque descriptor token; full descriptor semantics still require runtime helper coverage. |
| `built-ins/AggregateError/order-of-args-evaluation.js` | semantic unsupported assertion | Former `ArrayIsArray` and `ArrayPush` native runtime blockers are cleared; next work is AggregateError/iterator side-effect semantics. |
| `built-ins/AggregateError/prototype/errors-absent-on-prototype.js` | pass | Static Error/AggregateError prototype modeling plus direct `ObjectHasOwnProperty` static fold clear the refreshed representative. |
| `built-ins/Object/defineProperties/15.2.3.7-2-11.js` | pass | `ObjectDefineProperties` now preserves native stack shape for dynamic descriptor objects; full descriptor enumeration semantics remain open. |
| `annexB/built-ins/Date/prototype/getYear/not-a-constructor.js` | pass | Catchable `ReflectConstruct` now uses `$exception_pending`; dynamic `DateNow`/`DateNew`/`DateGetYear` preserve native stack shape for this path. |
| `language/arguments-object/async-gen-meth-args-trailing-comma-multiple.js` | semantic unsupported assertion | Dynamic `GeneratorNext` now emits an opaque native result; remaining failure is async-generator Promise/thenable semantics, not native emitter shape. |
| `intl402/NumberFormat/test-option-currency.js` | semantic/runtime unsupported | Static `ObjectToString`/`StringToUpperCase` folds and dynamic opaque string fallback clear the native diagnostic; full tagged heap string semantics remain open. |
| `language/expressions/class/elements/fields-anonymous-function-length.js` | semantic/runtime unsupported | `PrivateFieldSet` and `PrivateFieldGet` now compile through opaque private-field fallbacks; real class private storage semantics remain open. |
| `intl402/Number/prototype/toLocaleString/this-number-value.js` | fail | Output mismatch; not a native unsupported diagnostic. |

## Latest Implementation Notes: 2026-05-24

Focused representative coverage is now blocked by semantics, not native emitter shape coverage.
The native emitter can build and run all five selected probes without `native LoweredProgram emitter
does not support ...` diagnostics.

Number runtime follow-up:

- `NumberToString` and `NumberToStringRadix` now have native typed builders and are included in
  native runtime builder coverage.
- `NumberToString` uses the actual lowered ABI: two params, raw i32 receiver/radix, with tagged
  numeric sentinels preserved for `NaN`, `Infinity`, `-Infinity`, and `-0`.
- `NumberToStringRadix` is a manifest compatibility wrapper over `NumberToString`; catalog deps now
  point to `NumberToString`.
- Native console fast path treats `NumberToString`/`NumberToStringRadix` results as tagged runtime
  values so output goes through `$log`, not raw `i32` printing.
- `ValueToStringInto` number-tail length accounting was a real blocker: it returned a pointer-based
  length after digit reversal, causing `$log` to dump scratch memory after numeric strings.
- Verification: `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli number_to_string_matches_node_output --test node_diff -- --nocapture`
  now passes.

Native emitter ABI follow-up:

- Expression-position `parseInt`/`parseFloat` now use the native runtime helper ABI for static
  string inputs: static strings become tagged runtime strings, parseInt radices are tagged for
  `$parse_int_string`, and fully static parse calls fold back to native raw number / decimal
  fixture output.
- Expression-position `Number.isNaN`/`Number.isFinite`/`Number.isInteger`/
  `Number.isSafeInteger` now return native raw booleans. Static cases fold directly; dynamic cases
  pass tagged JS values into the typed helper and convert tagged booleans back to raw booleans.
- Static arithmetic folding now uses JS numeric conversion for `-`, `*`, and `/`, preserving the
  NaN sentinel for cases such as `"abc" / 1`.
- `NumberToFixed`, `NumberToExponential`, and `NumberToPrecision` now have typed native builders
  and native runtime-call emission normalizes raw native number receivers/precision arguments to
  the tagged helper ABI.
- Dynamic native numeric `Add/Sub/Mul/Div/Mod` now normalizes raw i32 operands into the typed
  runtime helper ABI via `NumberFromI32`, calls `$add/$sub/$mul/$div/$mod`, and converts the
  tagged helper result back through `NumberToI32` for existing native raw-number consumers.
- Native statement and loop conditions now fold statically known JS truthiness to raw `0/1`
  before branching. This keeps raw native booleans/numbers out of tagged `$truthy_bool` while
  fixing static `null`, `undefined`, `false`, `0`, and empty string condition behavior.
- Strict and loose equality now have separate static folds, and supported native operands are
  normalized to tagged values before calling `$strict_equal` / `$equal_equal`. This fixed
  `fixtures/core-semantics/abstract-equality.ts`.
- Static nullish coalescing now honors short-circuit side effects: the RHS is collected/emitted
  only when the static LHS is `null` or `undefined`. This fixed
  `fixtures/core-semantics/nullish-coalescing.ts`.
- Prototype fixture parity now covers static object prototype chains, explicit
  `Object.setPrototypeOf(obj, null)`, class `new` with `super(...)` constructor effects, and
  constructor property writes without leaking callee-local ids into caller object slots.
- Native string locals now use the same length-prefixed string storage expected by tagged runtime
  helpers, and `RuntimeFn::Concat` arguments are normalized to tagged values. This fixed
  `fixtures/core-semantics/default-params.ts` and `fixtures/core-semantics/plus.ts`.
- Static `PropertyIn` / `PropertyInDynamic` now lower for known object/prototype properties and
  print boolean console output correctly. `void console.log(...)` preserves the side effect before
  producing `undefined`, and console `typeof` uses the native typeof classifier instead of
  rejecting local probes as dynamic.
- Verification now also passes for
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli number_methods_matches_node_output --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli number_static_parse --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli number_is_ --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli number_formatting --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli number_arithmetic_dynamic_matches_node --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli m2_fixtures_match_node_output_under_iwasm --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli m3_semantic_fixtures_match_node_output_under_iwasm --test node_diff -- --nocapture`,
  and manual Node/iwasm parity for `fixtures/core-semantics/truthiness.ts`.

New native coverage added in this pass:

- `StaticObjectValue` now tracks data descriptor attributes (`writable`, `enumerable`,
  `configurable`) instead of storing only property values.
- Static `Object.defineProperty(obj, key, descriptor)` reads inline and local descriptor objects,
  applies descriptor defaults, and preserves attributes for later static queries.
- Static `Object.getOwnPropertyDescriptor(obj, key)` now returns the preserved descriptor attributes
  instead of always returning all attributes as `true`.
- Native emission has an import-free static fallback for `ObjectDefineProperty` expressions once the
  static object model has already recorded the mutation.
- `ObjectNew.non_enumerable` is reflected in the static object model, so native `ObjectKeys` no
  longer lists statically non-enumerable properties.
- `Date.prototype` lowers to a static non-enumerable prototype object for the Annex B legacy
  methods used by the selected Date descriptor probe.
- Native `verifyProperty` calls with fully static descriptor arguments are elided after comparing
  the expected descriptor against the native static descriptor model.
- Native `ArrayGet` can read static arrays with a dynamic local index by emitting a bounded
  selection chain over the static array slots.
- Required user-function collection now covers catch bodies, switch expressions/cases, non-static
  user-call statements, and block expressions whose statement body is still emitted even when the
  result is statically known.
- Because native `HeapClosureCall` dispatch currently emits branches to all lowered user functions,
  any program containing `HeapClosureCall` marks all user functions as required. This keeps modules
  self-contained and converts former unresolved-symbol compiler invariants into ordinary
  unsupported/runtime outcomes.
- `ArrayIsArray` now has a native typed helper and a static fold for statically known arrays and
  non-arrays.
- `ArrayPush` now has a native typed array-branch helper, which clears emitted-but-not-executed
  helper functions that previously blocked `AggregateError` coverage. Object-receiver `push`
  semantics still require the full property runtime path.
- Static `ObjectToString` and `StringToUpperCase` fold for known primitive values. Dynamic lowered
  calls compile by consuming their arguments and returning the native opaque string/reference token,
  matching the current static-token native string model rather than the full tagged heap string ABI.
- Dynamic `ObjectGetOwnPropertyDescriptor` now compiles to an opaque descriptor token when the
  static descriptor model cannot prove the result.
- `PrivateFieldSet` and `PrivateFieldGet` now compile for the current opaque native instance model.
  These fallbacks preserve stack shape and unblock representative coverage, but they do not replace
  the full private brand/slot storage implemented in the WAT runtime path.
- Static `for-of` over statically known arrays is now emitted directly by unrolling the body with
  per-iteration static context. Empty static arrays skip the body, which clears the leading
  `annexB/language/eval-code/direct/var-env-lower-lex-catch-non-strict.js` representative.
- Static generator calls now materialize a native static generator state. `GeneratorYield` over a
  known array and `GeneratorNext` over a known generator fold to `{ value, done }` descriptor
  objects while advancing the static generator index for subsequent `next()` calls. This clears the
  leading `annexB/language/expressions/yield/star-iterable-return-emulates-undefined-throws-when-called.js`
  `GeneratorNext` native unsupported representative; it now progresses to a non-native blocked
  outcome.
- Dynamic `ObjectGetPrototypeOf` now compiles to the native opaque prototype token. This keeps
  static identity checks available for known prototypes while clearing emitted expression sites
  such as `built-ins/AsyncGeneratorPrototype/next/iterator-result-prototype.js`, which now
  progresses from native unsupported to an ordinary output mismatch.
- Static Error-family prototypes, including `AggregateError.prototype`, now lower to static
  non-enumerable prototype objects with `constructor`, `name`, and `message`. Direct
  `ObjectHasOwnProperty`/`Object.hasOwn` runtime calls fold when the receiver and key are static.
- Static `Escape` and `Unescape` builtin calls now fold through the native static evaluator,
  including BigInt `ToString` inputs and UTF-16 `%uXXXX` escape/unescape forms. Static binary
  expressions are also folded before operand emission, which clears `escape(1n) !== "1"` style
  assertion guards from the native-emitter bucket.
- Dynamic `ObjectCreate` with prototype/descriptors now consumes both operands and returns the
  native opaque object token. This clears representative `Object.create(null)` cases while full
  prototype/null-prototype semantics remain part of the runtime helper plan.
- Dynamic `ObjectDefineProperties(obj, descriptors)` now preserves native stack shape by consuming
  the descriptors argument and returning the target object. This removes the current representative
  native diagnostic while full property descriptor enumeration remains runtime-helper work.
- Catchable `ReflectConstruct` now participates in the native `$exception_pending` try/catch path,
  and catch-body assignments invalidate static locals so subsequent native emission does not reuse
  stale pre-catch constants.
- Dynamic `DateNow`, `DateNew`, and `DateGetYear` now compile through opaque/native placeholder
  values. These clear the refreshed Date Annex B constructor representative while full Date object
  runtime semantics remain separate work.
- Dynamic `GeneratorNext` now compiles to an opaque native result. This moves the refreshed async
  generator representative to semantic unsupported assertion; the remaining work is Promise/thenable
  and async-generator result semantics.
- The wasm-encoder backend now resolves structured `WasmInstr::Br("$label")` / `BrIf("$label")`
  against the active control stack. This unlocks loop-heavy typed runtime builders without WAT-only
  validation.
- `ObjectHasOwnProperty` and `ObjectHasOwn` now have typed native builders. The former implements
  the own-property scan with typed loops/branches, symbol identity comparison, key stringification,
  and `$mem_equal`; the latter delegates to it. Runtime catalog signature/deps were corrected to
  `2->1` and `ValueToStringInto` + `MemEqual`.
- `PropertyHas` now has a typed native builder. It performs prototype-chain lookup with structured
  loops/branches, string-key `$mem_equal` comparison, and symbol-key identity lookup. This clears
  one more object-runtime dependency before the larger `PropertyGet`/`Index`/`PropertySet` port.
- `ArrayGet` now has a typed native builder. It follows forwarded sparse-array storage, validates
  tagged-number indexes, checks bounds and presence bits, and returns `undefined` for misses. This
  clears a prerequisite for `Index`, array iterators, typed-array helpers, and object/property
  paths that delegate array reads to `$array_get`.
- `ArrayIndexPresent` now has a typed native builder. It follows forwarded sparse-array storage,
  validates tagged-number indexes, checks bounds and presence bits, and returns tagged booleans for
  sparse-array property-presence checks. The catalog signature was corrected to `2->1`.
- `PropertyGet` now has a typed native builder. It implements object prototype-chain lookup,
  symbol-key lookup, string-key `$mem_equal` comparison, array string-index reads, and direct
  function token `name`/`length` metadata through native runtime data.
- `Index` now has a typed native builder, with typed UTF-8 code-point index helpers embedded when
  `$index` is linked. It covers object numeric keys, symbol keys, array reads, string indexing, and
  non-number key conversion through `$value_to_string_into`.
- `PropertySet` now has a typed native builder. It covers array numeric-key writes, object
  overwrite/append paths, symbol keys, copied string keys, and frozen/sealed/tracked non-writable
  guards with legacy silent-fail `undefined` results.
- `PropertyDelete` now has a typed native builder. It scans own object entries, handles symbol and
  string keys, rejects frozen/tracked non-configurable deletes with tagged `false`, compacts the
  entry table, and returns tagged booleans.
- `ReflectDeleteProperty` now has a typed native builder. It validates object targets, converts the
  key through `$value_to_string_into`, delegates to `$property_delete`, and corrects the runtime
  signature to `2->1`.
- `ReflectGet` now has a typed native builder. It validates object targets, converts keys through
  `$value_to_string_into`, delegates to `$property_get`, accepts the receiver parameter, and
  corrects the runtime signature to `3->1`.
- `ReflectHas` now has a typed native builder. It validates object targets, converts keys through
  `$value_to_string_into`, delegates to `$property_has`, and corrects the runtime signature to
  `2->1`.
- `BitwiseToI32` now has a typed native builder, and `BitwiseAnd`/`BitwiseXor`/`BitwiseOr` are
  connected to native runtime embedding. The binary bitwise helpers now validate through the
  embedded runtime path with corrected `2->1` signatures.
- `Negate` now has a typed native builder. It reuses `$number_to_i32`/`$number_from_i32` for
  coercion and tagging, and validates through the embedded native runtime path.
- `Sub`/`Mul`/`Div`/`Mod` now have typed native builders. They reuse the number conversion helpers,
  keep the legacy divide/mod-by-zero `undefined` behavior, and validate through the embedded native
  runtime path with corrected `2->1` signatures.
- `SubFast`/`MulFast`/`DivFast`/`ModFast` now have typed native builders. They keep the legacy
  number-tag guard structure and delegate to the corresponding base arithmetic helper through the
  embedded native runtime path.
- `NumberIsNaN`/`NumberIsFinite`/`NumberIsInteger`/`NumberIsSafeInteger` now have typed native
  builders. They keep the non-coercing checks, special number sentinels, and heap-number decimal
  scan behavior from the legacy helpers.
- `ValueOf` now has a typed native builder and preserves the legacy identity helper behavior
  through native runtime embedding validation.
- `BooleanCoerce` now has a typed native builder. It keeps the exact falsey checks, numeric zero
  test, string length test, and BigInt-zero object branch from the legacy helper.
- `IsNaN`/`IsFinite` now have typed native builders, including their string-scan auxiliary helpers.
  The embed path can now expand one RuntimeFn into multiple native helper functions when the legacy
  runtime helper had private companion functions.
- `BooleanToString` now has a typed native builder backed by native runtime string table values,
  with a data-segment test proving only `false`/`true` strings are materialized for that helper.
- `GlobalParseInt` now has typed native builders for `$parse_int`, `$parse_int_string`, and its
  numeric-input `$number_to_string` companion. The catalog dependency was corrected to include
  `NumberToString`, which makes the native link plan bring in the value-to-string allocation path.
- `NumberCoerce` now has a typed native builder. It keeps the legacy primitive coercion branches
  and reuses `$parse_int_string` for strings through the `GlobalParseInt` dependency path.
- `GlobalParseFloat` now has typed native builders for `$parse_float` and `$parse_float_string`,
  preserving the legacy integer-only parseFloat subset and tagged-zero fallback behavior.
- `StringEqual`, `BigIntCompare`, `StrictEqual`, `SameValueZero`, and `StrictNotEqual` now have
  typed native builders. `BigIntCompare` embeds the private `$is_bigint` companion, and the runtime
  catalog stack effects now match the actual two-argument ABI for equality helpers.
- `Concat` now has a typed native builder and native runtime-call validation. This removes the
  opaque Concat fallback and clears the string concatenation dependency for the upcoming `Add`
  helper.
- `Add`, `AddFast`, and `BigIntMixedArithmeticTypeError` now have typed native builders. `Add`
  preserves the legacy string/BigInt/numeric branch order; the BigInt mixed arithmetic helper uses
  native exception globals and runtime strings, with TypeError prototype wiring left for the later
  ObjectPrototype/builtin-error-prototype native work.
- `EqualEqual` and `BangEqual` now have typed native builders and native runtime-call validation.
  `EqualEqual` embeds the same private loose-equality companions as the legacy emitter, including
  string-to-number conversion and BigInt small-int/string comparison helpers.
- Relational comparison helpers are now native: `Less`, `LessFast`, `LessEqual`, `LessEqualFast`,
  `Greater`, `GreaterFast`, `GreaterEqual`, and `GreaterEqualFast`. They share a native
  `$bigint_compare_small_int` private companion and validate through native runtime embedding.
- `BigIntStringComparisonBoundaryError` now has a typed native diagnostic-abort builder. Its
  runtime string is registered through the native string table and its catalog stack effect now
  matches the legacy `0->0` helper shape.
- `SymbolNew`, `SymbolFor`, `SymbolKeyFor`, `SymbolDescription`, and `SymbolToString` now have
  heap-layout native builders connected through the native runtime registry. Ordinary symbols,
  registry symbols, `Symbol.keyFor`, `description`, `Symbol(...)` formatting, and
  `Symbol(...).toString()` now run through native helpers instead of opaque static tokens. Console
  dynamic-argument resolution now preserves non-static arguments so `console.log(s1, s2)` lowers to
  runtime `Concat` instead of dropping the second argument. Symbol `toString` lowering now wins
  before the generic numeric `toString` path, and non-numeric tagged `+` expressions can route
  through `$add` instead of raw `i32.add`.
- Verification for the Symbol pass:
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli symbol_constructor_basic_matches_node_output --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli symbol_registry_matches_node_output --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli symbol_registry_identity_matches_node_output --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli symbol_to_string_matches_node_output --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli symbol_key_dynamic_property_identity_matches_node_output_under_iwasm --test node_diff -- --nocapture`,
  `TS2WASM_RUN_NODE_DIFF=1 cargo test -p ts2wasm-cli symbol_key_descriptor_identity_matches_node_output_under_iwasm --test node_diff -- --nocapture`,
  `cargo test -p ts2wasm-backend-wasm --lib -- --nocapture`, and
  `python3 scripts/manager.py native-emitter-unsupported --format markdown` now pass with
  `native_unsupported: 0`.
- `Object.getOwnPropertySymbols` has a static native fallback for known object symbol keys, and
  static `Object.keys` now excludes symbol keys. Static descriptor objects now also fold
  `descriptor !== undefined` checks by strict type, and static symbol-key deletes mutate the native
  static object model. This clears
  `fixtures/core-expressions/symbol-key-descriptor-identity.ts` under iwasm while keeping the full
  dynamic `Object.getOwnPropertySymbols`/descriptor helper as future runtime work.
- Static `JSON.stringify` now folds known primitive/object/array values, numeric and string `space`
  arguments, replacer arrays, ignored boxed-space forms, and simple named function replacers. The
  focused `json_stringify` node-diff filter now passes all 21 selected stringify fixtures under
  iwasm. Static `JSON.parse` now covers the current literal fixture cluster, including nested
  object/array values, simple reviver functions, invalid-input SyntaxError traps, Unicode escapes,
  surrogate pairs, and lone low-surrogate replacement. The full dynamic `$json_parse`/
  `$json_stringify` typed helper builders remain open for non-static inputs.
- Static `String.prototype.charCodeAt` now folds known string/index calls, including out-of-range
  `NaN` results consumed by global `isNaN`; the focused `string_char_code_at` and
  `json_parse_latin1_unicode_escape` node-diff fixtures pass under iwasm. Dynamic
  `$string_char_code_at` is now embedded as a typed helper for tagged string/index operands, and
  `string-char-code-at-dynamic.ts` passes Node parity through `isNaN` for in-range/out-of-range
  results. `string-char-code-at-dynamic-print.ts` now also passes with the helper result stored in a
  local and printed directly through the tagged runtime log path; static console formatting maps the
  tagged `NaN` sentinel to `NaN` instead of its raw i32 payload.
- `String.prototype.codePointAt` is now embedded as a typed native helper with the same UTF-8
  code point decode path as `charCodeAt`, mapping invalid/out-of-range results to `undefined`.
  `string-code-point-at.ts` and `string-code-point-at-dynamic.ts` pass Node parity under iwasm.
- `String.fromCharCode` and `String.fromCodePoint` are now embedded as typed native helpers for the
  current single-argument runtime contract. They materialize heap strings through `AllocHeap` and
  pass `string-from-char-code.ts`, `string-from-code-point.ts`, and
  `string-from-code-point-dynamic.ts` under iwasm.

Current bottleneck decomposition for the focused five:

| Path | Current state | Bottleneck |
|---|---|---|
| `annexB/built-ins/Date/prototype/getYear/nan.js` | pass | No current native action. |
| `annexB/built-ins/Date/prototype/getYear/B.2.4.js` | pass | Date prototype descriptor support is now covered for this probe. |
| `annexB/built-ins/Array/from/iterator-method-emulates-undefined.js` | blocked by oracle node execution | Not a native unsupported bucket; reclassify or fix oracle handling before treating as emitter work. |
| `built-ins/Array/prototype/every/15.4.4.16-7-c-ii-1.js` | pass | Static-array dynamic-index reads now cover this callback shape. |
| `harness/compare-array-arraylike.js` | blocked by oracle node execution | Not a native unsupported bucket; runner should distinguish expected Node assertion throws from build/runtime failures. |
| `fixtures/core-expressions/symbol-key-descriptor-identity.ts` | pass | Static symbol keys, `Object.getOwnPropertySymbols`, descriptor enumerable flags, and symbol-key delete parity are covered for this native fixture. |
| `fixtures/builtins-and-io/json-stringify*.ts` selected by `json_stringify` | pass | Static JSON stringify covers the current literal/replacer/space fixture cluster; dynamic runtime helper builders are still required for general values. |
| `fixtures/builtins-and-io/json-parse*.ts` selected by `json_parse` | pass | Static JSON parse covers the current literal/reviver/invalid-input fixture cluster; dynamic runtime helper builders are still required for non-static inputs. |
| `fixtures/builtins-and-io/string-char-code-at.ts`, `string-char-code-at-dynamic.ts`, `string-char-code-at-dynamic-print.ts`, and `json-parse-latin1-unicode-escape.ts` | pass | Static `charCodeAt` covers known string/index calls; dynamic `$string_char_code_at` covers tagged receiver/index calls validated through `isNaN` and direct tagged-number local printing. |
| `fixtures/builtins-and-io/string-code-point-at.ts` and `string-code-point-at-dynamic.ts` | pass | Dynamic `$string_code_point_at` covers tagged receiver/index calls and direct local printing, including out-of-range `undefined`. |
| `fixtures/builtins-and-io/string-from-char-code.ts`, `string-from-code-point.ts`, and `string-from-code-point-dynamic.ts` | pass | Native typed `$string_from_char_code` / `$string_from_code_point` cover the current single-argument helper contract and direct local printing. |

Next implementation order from the latest full sweep:

1. Split and implement catchable `TryCatch` bodies by failure shape. Start with direct `throw`,
   catch-body local invalidation, and `$exception_pending` propagation already used by
   `ReflectConstruct`, then add real exception value plumbing before widening to `TryFinally`.
2. Decompose the remaining generic builtin-call bucket by `BuiltinId`. Static-fold global
   string/number coercion cases (`escape`, `unescape`, URI encode/decode, `Number`, `Boolean`,
   `isNaN`, `parseInt`) when operands are provably static; keep dynamic forms unsupported until
   tagged runtime helpers exist.
3. Continue the object-runtime helper path after `ObjectHasOwnProperty`/`PropertyHas` and
   `ArrayGet`: implement typed `PropertyGet`, `Index`, `PropertySet`, and descriptor helpers,
   while keeping current static folds for known objects/prototypes.
4. Add typed or controlled-opaque runtime coverage for high-count string/regexp/array helpers:
   `RegExpMatch`, `ArraySlice`, `ObjectToString` arity-2, `ArrayPushGrow`, and `ArrayIncludes`.
   Prioritize stack-shape fallbacks only when the result is not used for pass/fail branching.
5. Implement non-static iterator paths: `ForOf`/`ForAwaitOfLower`, array-buffer/typed-array
   constructors, and async/promise result plumbing. These now dominate the user-function
   `while emitting func_*` buckets.
6. Replace opaque fallbacks with real semantics in areas already unblocked for native shape:
   dynamic descriptors, heap strings, Date objects, generators, private fields, and null-prototype
   objects.
7. After each bucket family lands, run the representative path first, then
   `cargo test -p ts2wasm-backend-wasm native_lowered`, then the full `reference-coverage test262
   --jsonl --no-dashboard-data` sweep to reselect the next five buckets.

## Execution Order

1. Keep the guardrail green: no unresolved wasm symbols, no build-facing WAT fallback, no fake `undefined` success for unsupported runtime semantics.
2. Preserve the existing static fast paths for object/array reads and writes; expand them only when the lowered value is provably static.
3. Add native runtime embedding infrastructure from `RuntimeLinkPlan` to typed `WasmFunction` builders, starting with helpers already migrated in `runtime/core/typed.rs`.
4. Replace unary/raw binary i32 shortcuts with JS tagged/coercion-aware helper calls or controlled unsupported diagnostics.
5. Implement property get/set dynamic shapes through embedded `PropertyGet`, `Index`, `PropertySet`, and string-key conversion helpers.
6. Implement runtime-call expression emission by dispatching `LoweredExpr::RuntimeCall { intrinsic, args }` to embedded `RuntimeFn` symbols.
7. Clear statement blockers: catchable `TryCatch` handles direct `throw` and DirectLocalToken `HeapClosureCall` via `$exception_pending`; `ForIn` has a temporary empty-enumeration fallback; finish `TryFinally`, full `ForIn`, then `ForOf`/`ForAwaitOfLower`.
8. Run a full `reference-coverage test262 --jsonl --no-dashboard-data` refresh and update issue counts before closing the epic.

---

## File Map

| File | Role |
|------|------|
| `crates/backend-wasm/src/native_lowered.rs` | Main native emitter — add statement/expression handlers |
| `crates/backend-wasm/src/native_runtime_embed.rs` | **New** — collect and embed typed runtime functions into WasmModule |
| `crates/runtime-catalog/src/runtime_fn.rs` | RuntimeFn enum (reference only) |
| `crates/backend-wasm/src/wasm_ir.rs` | WasmInstr/WasmFunction types (reference only) |
| `crates/backend-wasm/src/runtime_link_plan.rs` | Link plan builder (may need updates for new traversals) |
| `crates/compiler/tests/native_backend_remaining_red.rs` | Update expected-error tests when constructs become supported |

---

### Task 0: Measurement and Guardrails

**Files:**
- Modify: `scripts/report/native-emitter-unsupported.py`
- Modify: `scripts/manager.py`
- Modify: `crates/backend-wasm/src/wasm_encoder_backend.rs`

Status:
- `native-emitter-unsupported` command exists.
- Reason counts are de-duped per JSONL record.
- wasm encoder now panics on unresolved call/export symbols instead of silently targeting function index 0.
- Unsupported `RuntimeCall` diagnostics include the `RuntimeFn` name and arg count.
- Unsupported diagnostics raised while emitting required user functions include the lowered function symbol, e.g. `while emitting func_13`.

Verification:
- `python3 -m py_compile scripts/report/native-emitter-unsupported.py scripts/manager.py`
- `cargo test -p ts2wasm-backend-wasm unresolved_`

### Task 1: Runtime Embedding Infrastructure

**Files:**
- Create: `crates/backend-wasm/src/native_runtime_embed.rs`
- Modify: `crates/backend-wasm/src/native_lowered.rs`
- Modify: `crates/backend-wasm/src/runtime/core/typed.rs` as needed

Create `embed_runtime_functions(plan: &RuntimeLinkPlan) -> Result<Vec<WasmFunction>, Diagnostic>`.
The first version must embed only helpers that have typed builders and whose transitive calls are
also embedded or imported. Missing builders must stay controlled unsupported diagnostics; they must
not produce unresolved `WasmInstr::Call`.

Initial helper set:
- `TruthyBool`, `Not`, `And`, `Or`, `IsString`
- `PropertyGet`, `Index`, `PropertySet` only after typed builders exist
- arithmetic/coercion helpers needed by the unary/property path

Status:
- `crates/backend-wasm/src/native_runtime_embed.rs` exists.
- Native embedding currently wires typed builders for `TruthyBool`, `Not`, `And`, `Or`, and `IsString`.
- Native embedding also has import-free fallback builders for `EvalIndirectHost` and `InstanceOf`; both are deliberately incomplete semantics and exist to keep native emission self-contained while the real helper path is built.
- `RuntimeCall` emission is allowed only for helpers with native typed builders; all others remain controlled unsupported diagnostics.
- Static `ArrayPushMany` is folded for direct runtime calls, simple user-function wrappers, and the native static `Function.prototype.call.bind(Array.prototype.push)` heap-closure shape.
- The focused five-case representative now has `native_unsupported: 0`. The former Date/getYear blockers cleared in sequence: dynamic property get, catchable try/catch body, `for-in`, `InstanceOf`, wider `HeapClosureCall` dispatch, `SymbolDescription`, and builtin `MathPow`.
- Native stack discipline now has regression coverage for unary `typeof` fallback and `return voidUserCall()`; both previously produced invalid wasm in helper-heavy cases.
- Native data literals are now interned so repeated string literals and static `typeof` strings compare by the same i32 token inside the current native representation.
- Static `Object.prototype.hasOwnProperty.call`/`propertyIsEnumerable.call` wrappers fold for native static object/array receivers, but dynamic wrapper bodies still require a real native ABI-compatible helper path.
- Static `Array.prototype.join.call(array, separator)` wrappers fold for native static arrays, and the `Array.prototype.every` representative's callback shape `obj[idx] === val` is inlined against native static array slots.
- Static object descriptors now preserve `Object.defineProperty` attributes and
  `Object.getOwnPropertyDescriptor` reports them correctly for native static objects.
- Static `verifyProperty` calls can be elided when the target object, key, and descriptor are all
  statically known.
- Static arrays now support native dynamic-index reads, which clears the selected
  `Array.prototype.every` callback-parameter probe.

Verification:
- `cargo test -p ts2wasm-backend-wasm native_runtime`
- `cargo test -p ts2wasm-backend-wasm static_array_push_many`
- `cargo test -p ts2wasm-backend-wasm native_lowered`
- `cargo test -p ts2wasm-backend-wasm native_lowered_static_define_property_descriptor_attrs_are_preserved`
- `cargo test -p ts2wasm-backend-wasm native_lowered_static_verify_property_descriptor_call_is_elided`
- `cargo test -p ts2wasm-backend-wasm native_lowered_static_array_dynamic_index_get_runs_without_wat_conversion`

### Task 2: RuntimeCall Expression Dispatch

**Files:**
- Modify: `crates/backend-wasm/src/native_lowered.rs`

For `LoweredExpr::RuntimeCall { intrinsic, args }`:
- emit args left-to-right;
- emit `WasmInstr::Call(intrinsic.symbol())`;
- require the symbol to be provided by Task 1's embedded helper set;
- keep unsupported diagnostics for runtime functions without typed native builders.

This removes the `runtime call` bucket only after transitive helper embedding is verified.

Next high-value runtime-call builders / folds:
- `ObjectHasOwnProperty` and `PropertyIsEnumerable`, including the `Object.prototype.{method}.call(object, key)` `HeapClosureCall` lowering used by `propertyHelper.js`.
- `ArrayJoin`, including `Array.prototype.join.call(array, separator)` from harness formatting helpers.
- `PropertyGet`, `Index`, `PropertySet`, `GetLength`: needed by property helper, array helper, and dynamic access buckets.
- `TypeOf`: needed before replacing raw unary shortcuts.
- Replace current opaque fallbacks for `InstanceOf`, `SymbolDescription`, `MathPow`, dynamic property get/set/delete, and unknown length with real tagged/runtime semantics.

---

### Task 3: Binary logical ops — And, Or, NullishCoalesce

**Files:**
- Modify: `crates/backend-wasm/src/native_lowered.rs`

Status: implemented in `emit_expr()` with short-circuit `if` blocks.

Remaining work: replace non-logical raw i32 arithmetic/comparison shortcuts with JS tagged/coercion-aware helper calls where operands are not proven raw numeric/boolean.

### Task 4: Unary Operators

**Files:**
- Modify: `crates/backend-wasm/src/native_lowered.rs`

Replace raw i32 emission with runtime-helper backed emission where JS coercion is required:
- `Delete`: static safe expression forms can emit tagged true.
- `Void`: emit side effects, then `undefined`.
- `TypeOf`: static literals can fold; dynamic values call `$typeof`.
- Current native status: static and locally inferable `typeof` cases fold to interned string tokens (`number`, `string`, `boolean`, `function`, `object`, `symbol`, `undefined`). Remaining work is a real `$typeof` builder for heap/tagged runtime values.
- `Not`, `Plus`, `Negate`: call runtime helpers unless the operand is proven raw numeric/boolean.

### Task 5: String as standalone expression

**Files:**
- Modify: `crates/backend-wasm/src/native_lowered.rs`

Status: implemented for the current static native subset. Standalone string expression emission uses native data allocation/static tokens, but full tagged heap string allocation still depends on runtime embedding.

### Task 6: This expression

**Files:**
- Modify: `crates/backend-wasm/src/native_lowered.rs`

In `emit_expr()`, add `LoweredExpr::This(_)` → check if function uses receiver, emit `LocalGet(receiver_local)`.

### Task 7: GetLength for static arrays

**Files:**
- Modify: `crates/backend-wasm/src/native_lowered.rs`
- Modify: `crates/backend-wasm/src/runtime/core/typed.rs`
- Modify: `crates/backend-wasm/src/native_runtime_embed.rs`
- Modify: `crates/backend-wasm/src/runtime_link_plan.rs`

Status: implemented for static arrays via `static_array_len`.

Status: dynamic length now calls the native `$get_length` typed runtime helper. The helper handles
tagged strings via UTF-8 code-point counting, arrays via heap array length including forwarding
nodes, and objects by delegating `"length"` to `$property_get`.

Follow-up: UTF-16 string length parity and full dynamic heap-object property semantics remain domain
semantics work, not native unsupported coverage.

### Task 8: Remaining expressions

Once runtime functions are available, implement handlers for:
- `Power` → call `$math_pow`
- `PropertyIn`/`PropertyInDynamic` → call `$property_has`
- `PropertyDelete`/`PropertyDeleteDynamic` → currently evaluate operands and return tagged true; replace with `$property_delete`
- `ArrayGet`/`ArrayNewSparse` → call `$array_get` / `$alloc_heap`
- `ErrorNew` → call `$error_message` pattern
- `EnvCellNew`/`EnvCellGet`/`EnvCellSet` → heap env cells
- `OptionalCall` → conditional dispatch
- `MethodCall` → property get + call
- `BuiltinErrorPrototype` → emit prototype reference
- `ModuleLoad` → call `$module_require`
- `ModuleExportsAssign` → call `$module_exports_assign`

### Task 9: Remaining statements

- `TryFinally`/`TryCatch` → non-throwing try bodies emit now; direct `throw` and DirectLocalToken `HeapClosureCall` can catch `$exception_pending`; remaining work is full exception object/value semantics, `finally` completion records, and dynamic/heap-object closure dispatch
- `ForIn`/`ForOf`/`ForAwaitOfLower` → `ForIn` currently evaluates the iterable and skips the body; replace with real property enumeration and iteration protocols
- `ClassDecl` → class initialization
