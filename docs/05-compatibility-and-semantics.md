# Compatibility and semantics

この文書は TypeScript 構文、TypeScript 型、JavaScript 実行意味論、module/npm ecosystem の扱いをまとめる。

## TypeScript 構文対応

構文対応は、単純な文法から始めるが、対応予定を削らない。未対応構文は明示的な診断を出し、テスト上も `expected unsupported` として管理する。

構文の基準は TypeScript compiler が受理する TypeScript と、実行時の ECMAScript semantics である。AssemblyScript 固有の `i32`、`i64`、`f32`、`usize`、`changetype`、明示メモリ操作 API、AssemblyScript 標準ライブラリは入力言語として扱わない。内部最適化で primitive 表現を使う場合も、ユーザー可視の構文・型は TypeScript に限定する。

構文ごとの対応方針と実装状況の詳細は `docs/language-reference/typescript-features.md` および `docs/language-reference/javascript-features.md` を参照。

初期段階で重要なのは、「簡単な構文しか対応しない」ことではなく、「構文ごとの未対応理由を潰せる形で管理する」ことである。たとえば `async/await` が未対応なら、parser が読めないのか、IR が表現できないのか、runtime に Promise がないのか、host event loop がないのかを分ける。

## TypeScript parse / erase / emit boundary

TypeScript-only syntax is owned by the frontend before runtime lowering. The compiler classifies every TypeScript-specific construct into one of four categories and uses that category to choose the diagnostic and follow-up issue owner.

| Category | Construct family | Compiler contract | Diagnostic owner |
|---|---|---|---|
| 1. Parse and erase | `type` aliases, `interface`, type annotations, type-only imports/exports, `as` assertions, angle-bracket assertions outside JSX, `satisfies`, generic type parameters, overload signatures with no implementation body | Parser accepts the syntax, preserves spans for diagnostics, and the erasure pass removes type-only nodes before JS runtime lowering. No runtime IR or backend representation is created. | `UnsupportedTypeScriptSyntax` until the parser/eraser slice exists |
| 2. Parse and preserve emit/module shape | `import` / `export` forms, `export =`, `import = require`, `namespace` / internal `module`, external module declarations, declaration emit forms that affect `.d.ts` or module shape | Parser keeps enough structure for module graph construction, declaration-shape validation, or future JS/module emit decisions. Runtime lowering only receives executable JS statements and resolved module effects. | `UnsupportedModule` when module graph or emit shape is the blocker; `UnsupportedTypeScriptSyntax` when the parser cannot represent the TS form |
| 3. Parse and lower executable JS semantics | class accessors with bodies, parameter properties that emit constructor assignments, enums and const enums, decorators when selected semantics require runtime calls or metadata, JSX when configured to emit function calls | Frontend must make the TypeScript transform explicit before lowering: either rewrite to executable JS-equivalent AST/HIR or reject with a source diagnostic. Runtime issues own only the resulting JavaScript behavior after the transform boundary. | `UnsupportedTypeScriptSyntax` for missing TS transform; `UnsupportedRuntimeSubset` for a transformed JS runtime feature that is unsupported |
| 4. Reject as TypeScript-only unsupported | ambient declarations in executable input when they would introduce no runtime binding, unsupported declaration-only files, unsupported `declare global` / `declare module` shapes, TS directives with no compiler-mode equivalent | Parser may recognize the form for a precise diagnostic, but the compiler must not silently create runtime bindings or placeholder objects. | `UnsupportedTypeScriptSyntax`, except module-graph-only ambient module blockers use `UnsupportedModule` |

The erasure boundary is before name resolution for runtime bindings. Type-only names must not enter runtime scope unless a construct explicitly emits JavaScript. A source form that is erased must still be checked enough to avoid changing runtime meaning, for example by preserving import/export kind and by rejecting ambiguous JSX/type-assertion syntax in the wrong parse goal.

### TypeScript reference label mapping

Coverage labels from `tsc` and `tsgo` map to the boundary categories above:

| Label | Boundary category | Owner issue guidance |
|---|---|---|
| `type-alias` | 1. Parse and erase | Implement through a narrow type-alias erasure child such as issue 345, not as runtime type support |
| `type-annotation` | 1. Parse and erase | Parser/eraser slice; no runtime issue unless the remaining executable expression fails |
| `type-assertion` | 1. Parse and erase, with JSX parse-goal guard | Parser/eraser slice; JSX ambiguity belongs to JSX/parser selection |
| `type-system` | 1. Parse and erase unless it requires checker-only diagnostics | Checker/type-erasure follow-up; not runtime semantics |
| `ambient-declaration` | 4. Reject or erase precisely | Dedicated ambient declaration erasure/rejection child, with module-shaped ambient declarations split to module ownership |
| `declaration-emit` | 2. Preserve emit/module shape or 4. reject declaration-only input | Declaration emit child such as issue 346; no wasm backend work unless executable JS is produced |
| `import-export` | 2. Preserve emit/module shape | Module graph, resolver, or emit-shape issue; runtime only after executable module body exists |
| `module-resolution` | 2. Preserve emit/module shape | Module resolver ownership |
| `module-system-amd` | 2. Preserve emit/module shape | Module-system transform/policy ownership, normally `UnsupportedModule` |
| `class-accessor` | 3. Parse and lower executable JS semantics, or 1/4 for declaration-only accessors | TS transform/frontend ownership first; runtime class semantics only after JS-equivalent accessor form exists |
| `decorator` | 3. Parse and lower executable JS semantics | TS transform/policy ownership; runtime decorator calls are separate only after transform selection |
| `jsx` | 3. Parse and lower executable JS semantics | JSX transform policy and factory/module ownership |
| `typescript-directive` | 4. Reject or model compiler-mode directive | Reference coverage/harness ownership unless the directive changes parse or module behavior |
| `parser-syntax` | Split by representative case | Must be re-triaged into one of the categories above before implementation-ready work |
| `unknown-unsupported` | Split by representative case | Must not become a runtime issue until the TypeScript boundary category is known |
| `arguments-object`, `runtime-subset`, `class`, `object-literal` | JavaScript runtime semantics after TS transform | Defer to existing JS runtime/frontend issues, not TypeScript-only syntax buckets |

## TypeScript 型対応

TypeScript の型は実行時に消えるが、コンパイル時には重要である。型情報を利用することで、WASM 生成の品質を上げられる。たとえば、`number[]` と分かる配列は generic object array より効率よく扱える可能性がある。`string` と分かる値への `+` は文字列結合に落とせる。`boolean` と分かる値の branch は truthiness 判定を単純化できる。

ただし、TypeScript の型は sound ではない。`any`、型アサーション、構造的部分型、union、intersection、generic、conditional type などがあるため、型だけを信じて runtime check を完全に消すと壊れる。optimization level と semantic safety mode の対応は `docs/11-shared-definitions.md` を正とする。`unsafe-fast` を標準動作にしてはいけない。これは性能を捨てないためではなく、性能改善と互換性維持を分離するためである。

TypeScript 型は、最適化のヒントであって別言語への opt-in ではない。`number` を内部的に `i32` fast path へ落とす場合は、範囲解析、overflow、`NaN`、`-0`、`Infinity`、property escape、function call boundary を検査する。検査できない場合は JS `number` semantics に戻す。

| 入力 | TypeScript としての扱い | 方針 |
|---|---|---|
| `let x: number = 1` | 標準 TypeScript | fast path 候補 |
| AssemblyScript 固有型名 | TypeScript 上は通常の型参照 | 組み込み primitive 型として特別扱いしない |
| `arr: number[]` | 標準 TypeScript | packed number array 候補 |
| `arr: Int32Array` | 標準 JS typed array | typed array runtime として扱う |
| AssemblyScript intrinsic 風 API | TypeScript 上は通常の識別子参照 | intrinsic として扱わない |

## JavaScript 意味論

このプロジェクトの難所は TypeScript 構文ではなく JavaScript 意味論である。特に、`this`、prototype、property lookup、dynamic object shape、truthiness、`==`、`===`、`NaN`、`-0`、exception、closure、`eval`、`with`、getter / setter、Proxy などは WASM への直接変換を難しくする。

対応方針として、まず通常の TypeScript コードでよく使われる範囲を正確に実装する。`eval` や `with` のような最適化を破壊する機能は、最初から最重要扱いにはしない。ただし、仕様から削除はしない。明示的に `unsupported-dynamic-code` として扱う。

WAMR は multi-thread (wasi-threads)、socket API (Berkeley/Posix Socket) をサポートしており、WASI 経由でネットワーク機能や並列処理も実行可能である。wasm-tools は既に Wasm GC、reference-types、function-references、multi-memory、multi-value、SIMD、tail-call、threads などの提案を実装しており（多くは Stage 4+）、これらを活用することでより効率的な実装が可能になる。

| 機能              | 方針                                        |
| --------------- | ----------------------------------------- |
| `this`          | call site ごとに receiver を明示して IR に落とす      |
| prototype       | class 対応後に object model に統合               |
| property lookup | string key lookup から開始し、shape cache を後で追加 |
| `==`            | runtime helper で primitive coercion を実装し、object ToPrimitive は object model 安定後に追加 |
| `===`           | primitive fast path を用意                   |
| `NaN` / `-0`    | number semantics のテスト対象にする                |
| exception       | runtime stack と wasm exception の両案を検討     |
| `eval`          | static-string direct eval は eval-code を caller scope 内の Lowered IR に展開して wasm として実行。dynamic / indirect eval は診断または将来の audited host shim 対象 |
| `with`          | 初期非対応、診断必須                                |
| Proxy           | 初期非対応、object model 安定後に検討                 |

Compatibility evidence distinguishes syntax/build support from semantic parity. A feature that parses, lowers, or builds with placeholder behavior is recorded as `部分実装` in `docs/language-reference/javascript-features.md` and must have an open issue with Node differential acceptance criteria before it can count toward semantic gates. Current placeholder or partial semantic trackers include issues 207-214 for `instanceof`, switch fall-through, labeled control flow, arrow closures, `this`, rest parameters, template interpolation, and string methods.

`Math.random` is capability-gated for standalone WASI output: use of `Math.random()` imports `wasi_snapshot_preview1.random_get`, sets `wasi.random: true` in the capability manifest, and remains valid under host-deny because it does not require a Node host import. The current tagged-int runtime can only expose an integer-backed random payload; full ECMAScript fractional double parity is part of the broader number representation model, not a silent deterministic placeholder.

Abstract equality (`==` / `!=`) supports the current primitive runtime value set: `undefined`, `null`, booleans, tagged integer numbers, and strings that coerce to tagged integers. Full object `ToPrimitive`, floating point, `NaN`, and `-0` behavior remain tied to the broader object and number-model work.

BigInt は heap object representation として設計済みだが、runtime 値と操作は段階実装である。BigInt literal runtime values は issue 259、arithmetic は issue 260、equality/comparison/coercion の最初の境界は issue 261、builtin/string conversion は issue 262 が所有する。BigInt と Number の arithmetic は暗黙変換せず TypeError path にする。issue-261 では BigInt 同士の mathematical value strict equality、abstract equality、relational comparison と、Number など非 BigInt との `===` / `!==` false/true 境界を実装済みである。Literal BigInt/String `==` / `!=` は supported StringToBigInt subset で fold し、invalid string は Node と同じ false/true 境界にする。Literal BigInt/Boolean `==` / `!=` は `false -> 0n`、`true -> 1n` の境界で fold する。Literal BigInt/Number `==` / `!=` は representable tagged-int number literals と unary-negative integer literals（`-0` を含む）だけを fold する。Literal BigInt/Number `<` / `<=` / `>` / `>=` も同じ静的 integer subset だけを fold する。Literal BigInt/nullish `==` / `!=` は Node と同じ false/true 境界に fold する。Runtime BigInt/String、BigInt/Boolean、BigInt/nullish mixed coercion は current object-carried primitive boundary で実装済みだが、BigInt/String runtime comparison は signed-i32 StringToBigInt value に制限する。Literal-derived local and object-property out-of-range dynamic strings は issue-282 diagnostic とする。Unknown non-source-backed dynamic strings that parse outside that boundary trap at runtime instead of returning a normal boolean; full compatible out-of-range comparison remains tied to broader BigInt representation work。Object `ToPrimitive` は direct object-literal/local の引数なし arrow `valueOf` / `toString` が supported primitive literal を返す場合だけ primitive として扱う。返り値 subset は BigInt literal、boolean、supported tagged-int number、nullish equality、supported StringToBigInt string で、relational では nullish を除く current runtime-compatible primitive subset に制限する。Direct object-literal/local の引数なし arrow `toString` が invalid または signed-i32 StringToBigInt comparison boundary 外の string literal を返す場合は issue-373 source diagnostic とする。non-arrow/function body/prototype/Proxy/side-effectful coercion は issue 374、unknown out-of-range dynamic strings は issue 375 の follow-up として扱う。Broader number model limits (`NaN`, `Infinity`, signed unary `NaN` / `Infinity`, fractional number tokens) は issue 281 が所有する。現時点では statically visible BigInt/Number comparison の `NaN` / `Infinity` / signed unary special globals / `Number.NaN` / `Number.POSITIVE_INFINITY` / `Number.NEGATIVE_INFINITY` / other unshadowed `Number.*` numeric constants / fractional number token sequences は source-spanned issue-281 diagnostics にする。

Object literal/local mixed BigInt comparisons support the narrow direct no-argument arrow `valueOf` / `toString` primitive-return slice for BigInt literals, booleans, supported tagged-int numbers, nullish equality, and supported StringToBigInt strings. Direct no-argument arrow `toString` invalid/out-of-range string returns are source-diagnosed by issue 373; other `ToPrimitive` shapes remain source-diagnosed or unsupported by precise follow-ups: broader object-model-dependent coercion in issue 374 and non-source-backed unknown out-of-range runtime strings in issue 375.

Broader object `ToPrimitive` for mixed BigInt comparison is split by the object-model surface that must be made observable. Direct object-literal or direct local objects with own data properties are the only source-backed shapes that may be folded before the general object model is complete. No-argument own arrow properties and own methods whose body is a single primitive `return` are supported because their receiver, call ordering, and return primitive are statically visible. Prototype lookup, inherited `valueOf` / `toString`, getters, Proxy traps, receiver-sensitive method bodies, mutation/side-effectful coercion, and exception-producing coercion must remain diagnosed or deferred until property lookup, call receiver binding, and completion propagation have explicit runtime contracts. `valueOf` is attempted before `toString` for ordinary comparison coercion; if both return objects, the compatible result is a TypeError rather than silent boolean fallback.

## Array / object semantics（実装済み範囲）

現行 lowering がカバーする array と object の semantics 要件を記録する。

| 機能 | 実装状態 | 備考 |
|---|---|---|
| array literal `[e0, e1, ...]` | 実装済み | heap block `[i32 len, elem₀, ...]` tagged `ptr\|5` |
| numeric array index `arr[n]` | 実装済み | tag check あり; 範囲外は `undefined` |
| `arr.length` | 実装済み | tag check あり; 非 array/string は `undefined` |
| `str.length` | 実装済み (basic) | UTF-8 byte storage を使う。完全な UTF-16 code unit parity は追跡中 |
| object literal `{k: v}` | 実装済み | heap block `[i32 count, (key_raw, value)×n]` tagged `ptr\|7` |
| data property read `obj.key` | 実装済み | reverse scan; last duplicate key wins (JS 仕様) |
| dynamic property key | 実装済み (basic) | string key による `obj[key]` / assignment をサポート |
| prototype / method call | 部分実装 (basic) | `[[Prototype]]` slot と method receiver の basic path をサポート。`instanceof` full traversal and `this` receiver parity are tracked by issues 207 and 211 |
| non-ASCII string literal | 実装済み (basic) | UTF-8 byte storage。decode/encode runtime helper は追跡中 |
| object literal key (string literal) | 未実装 | `{key: v}` の key は identifier only; `{"x": v}` は parse error |
| `obj["key"]` computed property | 実装済み (basic) | object property lookup path を使う |
| heap OOM check | 実装済み | `$alloc_heap` は memory.size を検査し、超過時に trap する |

Sparse array holes are absent indexed properties, not values equal to
`undefined`. Array elisions increase `length` while leaving the corresponding
presence bit unset. Numeric `index in array` observes presence, `array[index]`
reads holes as `undefined`, `Array.prototype.map` skips holes and preserves them
in the result, and array/call spread reads holes through iterator/Get semantics as
present `undefined` destination values. The runtime representation contract is
defined in `docs/14-runtime-abi.md`.

`$property_get` の reverse scan により、`{a:1, a:2}.a === 2` が成立する (JS 仕様準拠)。

## Date timezone formatting policy (issue 5244)

`Date.prototype.toString()` and related formatting methods (`toDateString`, `toTimeString`,
`toISOString`) require access to the local timezone offset to produce human-readable output.

### Constraints

1. **Standalone WASI mode**: No host timezone API is available. WASI preview 1 provides
   `clock_time_get` (realtime/monotonic) but no timezone database or offset query.
2. **Node host shim**: When `host.date.*` imports are available, we can use Node's
   `Intl.DateTimeFormat` or `Date.prototype.getTimezoneOffset` host shim.

### Policy

| Mode | Timezone behavior | Implementation |
|---|---|---|
| Standalone WASI | UTC-only: all formatting methods return UTC-based strings | Runtime builtin with hardcoded UTC offset calculation from epoch milliseconds. No timezone name suffix (e.g., `"GMT+0000"`) |
| Node host shim | Full local timezone from host | `host.date.getTimezoneOffset(epoch_ms)` returns minutes offset. Date formatting uses host offset to produce local-time strings |
| WASI + limited host | Explicit host import per call | Each formatting method that needs timezone declares `host.date.getTimezoneOffset` in capability manifest |

### Implementation order

1. **Phase 1**: Implement UTC-only `Date.prototype.toString()` in WAT runtime using
   calendar math (year/month/day/hour/minute/second from epoch ms). Output format:
   `"Tue, 01 Jan 2026 00:00:00 GMT"`. This is standalone-capable.
2. **Phase 2**: Add `host.date.getTimezoneOffset` shim (WAT→host). Use offset to compute
   local time and append timezone offset string. This requires Node host.
3. **Phase 3 (deferred)**: Full `Intl.DateTimeFormat`-style formatting with
   locale-aware month/day names.

The current `Date.prototype.toString()` is blocked on Phase 1 implementation.
Issue 050 tracks the overall Date implementation. Issue 5244 tracks this policy.
Annex B legacy methods (`getYear`, `setYear`, `toGMTString`) remain issue-241 diagnostics.

> **残タスク**: RuntimeLinkPlan と WatEmitter の分離、AST に一貫した `Span`、BuiltinResolver pass の整理、
> capability manifest の本番出力などは、`current-state.md` と `docs/11-shared-definitions.md` の gate / issue で追跡する。

## Standard idiom semantics

docs/03 の WASI-compatible idiom は、host shim 不要という意味で standalone 候補である。ただし、そこに含まれる JS 意味論は runtime 側で別途実装されている必要がある。

| Idiom | 必要な JS semantics | 未実装時の扱い |
|---|---|---|
| `input.trim()` | `String.prototype.trim` | `unsupported-string-trim` |
| `input.split(/\s+/)` | RegExp literal、RegExp split | `unsupported-regexp-split` |
| `.map(Number)` | Array iteration、callback call、`Number` conversion | `unsupported-array-map` または `unsupported-number-constructor` |
| `console.log(sum)` | string conversion、WASI stdout | host shim 不要 |

この分類により、「Node.js host が不要」と「JS runtime semantics が実装済み」を混同しない。前者は capability の問題であり、後者は互換性・runtime 実装の問題である。

## 追加設計: module and npm ecosystem

既存 TypeScript/JavaScript 資産を活用するには、構文だけでなく module 解決と package ecosystem の扱いが必要である。初期から npm 全体を扱う必要はないが、段階を明確にする。

| Phase | Scope |
|---|---|
| Phase 1 | single file only |
| Phase 2 | relative `import` / `export` |
| Phase 3 | builtin module lowering: `fs`, `path`, `process`, `buffer` |
| Phase 4 | `package.json` based resolution |
| Phase 5 | selected npm package compatibility |

扱う必要がある論点:

| Topic | Policy |
|---|---|
| ESM / CommonJS | 段階的に両対応。最初は静的に解けるものを優先 |
| `require()` | literal require を compile-time builtin resolution から開始 |
| dynamic require | 初期は unsupported-dynamic-module |
| `package.json exports` | npm package 対応段階で導入 |
| native addon | 初期非対応。host capability として扱う |
| optional dependency | package compatibility 段階で明示管理 |
| side-effect import | module graph に side-effect bit を持たせる |
| tree shaking | host shim trimming とは別に module DCE として扱う |
