# Runtime ABI

このドキュメントは ts2wasm の runtime ABI を定める。
`RawValue` の tagged representation (`crates/runtime-abi/src/value.rs`)、
heap レイアウト (`crates/runtime-abi/src/layout.rs`)、
`RuntimeFn` カタログ、host import ABI を定義する。

> **ABI contract**: The logical ABI (`jsval` = `i64`, defined in `docs/04`) is the semantic value type at import/export boundaries. The wire representation (`RawValue` = `i32` tagged encoding, defined below) is the actual wasm module encoding. These two representations must be kept distinct; any conversion between them requires an explicit bridge in the backend layer. See `docs/04-compiler-architecture-and-runtime.md` §Runtime ABI for the policy.

## Tagged i32 value representation（small-int plus integer heap-number subset）

### RawValue (i32 tagged encoding)

現行パイプラインでは JavaScript の値を wasm モジュール内で `i32` の tagged encoding として表現する。
これは意図的な **integer-only subset** であり、JS 全体の `number` セマンティクスではない。

> **論理 ABI との関係**: `docs/04-compiler-architecture-and-runtime.md` の論理 ABI では `jsval` を `i64` として定義し、`crates/shared/src/abi.rs` の `AbiType::JsVal` もそれに従う。`i32` RawValue は wasm 本体の wire 表現であり、論理 `i64` との変換は明示的な bridge のみで行う。backend は wire と論理表現を暗黙に混在させてはならない。

```text
i32 tagged value (RawValue):

  undefined: 0b000 = 0
  null:      0b001 = 1
  false:     0b010 = 2
  true:      0b011 = 3
  number:    (n << 3) | 0b100   — n は i32 の範囲内の整数のみ
  array:     ptr | 0b101        — ptr はヒープ上の Array object のアドレス (8-byte aligned)
  string:    ptr | 0b110        — ptr はヒープ上の String object のアドレス (8-byte aligned)
  object:    ptr | 0b111        — ptr はヒープ上の Object のアドレス (8-byte aligned)

TAG_MASK:  0b111
HEAP_MASK: !0b111 (= -8 in two's complement)
```

### Typed wrapper types

`crates/runtime-abi/src/value.rs` は生の `i32` をラップする 3 つの typed wrapper を提供する。
backend コードはこれらの型を使って、tagged value / heap pointer / raw local value を型レベルで区別する。

- `TaggedValue(i32)`: 完全な JS 値。下位 3 bit が tag。`encode_number(n)`, `decode_number()`, `heap_ptr()`, `from_heap_ptr(ptr, tag)` を提供する。
- `HeapPtr(u32)`: heap 上の 8-byte aligned ポインタ。`new(offset)` / `new_unchecked(offset)` / `offset(amount)` / `tag(tag)` を提供する。
- `LocalRawValue(i32)`: wasm local variable slot に格納された未解釈の `i32` 値。`as_tagged()` で `TaggedValue` へ変換する。

backend が生の `i32` を tagged value として直接構築することは禁止。
かならず `TaggedValue` / `HeapPtr` / `ValueTag` の API を通す。

定数は `crates/runtime-abi/src/value.rs` の `ValueTag` で定義する。
backend が直接数値を埋め込むことは禁止。

Tagged `number` は small-int payload range
`ValueTag::NUMBER_PAYLOAD_MIN..=ValueTag::NUMBER_PAYLOAD_MAX` の整数だけを
直接表す。issue 300 の progress slice では、この範囲外でも `i32` に収まる
整数値を **heap number** として扱う narrow path を追加している。heap number
は `object` tag (`ptr | 0b111`) を使い、ordinary object payload の count slot
に `HEAP_NUMBER_SENTINEL == -1` を置くことで object / BigInt / closure と区別する。

```text
Heap number payload:

  +0  i32 sentinel      ; HEAP_NUMBER_SENTINEL (-1)
  +4  i32 prototype     ; 0 in the current primitive-number slice
  +8  i32 decimal_len   ; cached decimal spelling length
  +12 u8 decimal[...]   ; signed base-10 spelling, no fractional form
```

`$number_from_i32` は small-int payload range 内なら tagged number を返し、
範囲外の `i32` 整数なら heap number を割り当てる。`$number_to_i32` は tagged
number または heap number の cached decimal spelling から `i32` へ戻す。現時点の
保証範囲は ABC451 D で必要な integer-only arithmetic / comparison / `String(n)` /
unary-plus round trip であり、fractional values、`NaN`、`Infinity`、`-0`、
IEEE-754 丸めは未実装のまま維持する。

## Memory Layout

```text
bounded linear memory (current default: initial 2 pages, max 185 pages):

  [0 .. 8)                          — 予約
  [8 .. 16)                         — fd_write iovec (IOVEC_PTR=8, IOVEC_LEN=12)
  [16 .. 28)                        — stdin fd_read iovec + nread slot
                                       (STDIN_IOVEC_OFFSET=16, STDIN_IOVEC_PTR=16,
                                        STDIN_IOVEC_LEN=20, STDIN_NREAD_OFFSET=24)
  [28 .. DATA_START)                — 予約
  [DATA_START .. SCRATCH_OFFSET)    — static data segment (interned strings)
  [SCRATCH_OFFSET .. SCRATCH_OFFSET+SCRATCH_SIZE) — scratch buffer ($value_to_string_into 用)
  [STDIN_BUFFER_OFFSET .. STDIN_BUFFER_OFFSET+STDIN_BUFFER_SIZE) — stdin read staging buffer
  [HEAP_START .. )                  — heap ($heap global, bump allocator)
```

定数は `runtime/layout.rs` の `Layout` で定義する。

| 定数 | 値 | 用途 |
|---|---|---|
| `MEMORY_MIN_PAGES` | 2 | wasm memory の初期 page 数 |
| `MEMORY_MAX_PAGES` | 185 | wasm memory.grow の現在の上限 (11.5625 MiB)。ABC451 depth-8 live-set reducer が `292743` を出力する最小確認値で、OOM trap 境界は維持する |
| `DATA_START` | 256 | interned string data の開始オフセット |
| `ALIGN` | 8 | data segment / heap の alignment |
| `SCRATCH_OFFSET` | 1500 | 一時バッファの開始オフセット |
| `SCRATCH_SIZE` | 256 | 一時バッファのサイズ |
| `HEAP_START` | 2048 | heap bump allocator の開始アドレス |
| `IOVEC_PTR` | 8 | fd_write iovec の ptr フィールドオフセット |
| `IOVEC_LEN` | 12 | fd_write iovec の len フィールドオフセット |
| `STDIN_IOVEC_OFFSET` | 16 | stdin fd_read iovec 構造体のベースオフセット |
| `STDIN_IOVEC_PTR` | 16 | stdin fd_read iovec の buf ptr フィールドオフセット |
| `STDIN_IOVEC_LEN` | 20 | stdin fd_read iovec の buf_len フィールドオフセット |
| `STDIN_NREAD_OFFSET` | 24 | fd_read が書き込む nread 値のオフセット |
| `STDIN_BUFFER_OFFSET` | 1792 | stdin read staging buffer の開始オフセット |
| `STDIN_BUFFER_SIZE` | 256 | stdin read staging buffer のサイズ |
| `STDIN_READ_LIMIT` | 65536 | 1 回の readFileSync(0) で読める最大バイト数 (64 KiB) |

### Heap String Object

interned string / runtime string はヒープ上に以下の形式で配置する。

```text
[offset + 0 .. +4)   : i32 length (バイト数)
[offset + 4 .. +4+N) : UTF-8 bytes

RawValue = ptr | 0b110  (ptr は 8-byte aligned)
```

ptr を取り出すには: `ptr = raw_value & HEAP_MASK`
length を読むには: `len = i32.load(ptr)`
文字列バイトは: `ptr + 4` から `len` バイト

### Heap Array Object

Array payload は dense/sparse 共通の logical contract として、length、
per-index presence、RawValue element storage を持つ。Hole は ordinary
`undefined` value ではない。Presence bit が 0 の index は absent property
であり、element storage の値を JavaScript value として観測してはならない。

Sparse-capable array payload は以下の形式を使う。

```text
offset + 0  .. +4      : i32 length
offset + 4  .. +8      : i32 capacity
offset + 8  .. +12     : i32 presence_word_count
offset + 12 .. +16     : i32 elements_offset_from_payload_start
offset + 16 .. +16+W*4 : u32 presence_words[W]
offset + elements_offset_from_payload_start
                       : RawValue elements[capacity]

RawValue = ptr | 0b101  (ptr は 8-byte aligned)
```

`presence_word_count` は `ceil(capacity / 32)` で、bit `index % 32` が 1
のとき index は present property である。Dense array はすべての
`0 <= index < length` の presence bit を 1 にする。Sparse array hole は
presence bit 0 とし、該当 element storage には `undefined` を格納する。
GC は stale heap reference を保持しないよう、presence bit 0 の slot を mark
対象にしないか、slot が canonical `undefined` であることを前提に全 slot scan
してもよい。

Parser/frontend は array literal slots を expression list ではなく
slot list として表す。Allowed representation は
`ArrayLiteralElement::Present(expr)`, `ArrayLiteralElement::Hole(span)`,
`ArrayLiteralElement::Spread(expr)` 相当である。Elision は comma structure
から作る。`[1,]` は length 1 の dense array、`[1,,]` は index 1 が hole の
length 2 array、`[,]` は index 0 が hole の length 1 array として扱う。

Resolved/lowered IR は present と absent を失わない slot representation を持つ。
Allowed implementation path は `LoweredArraySlot::{Present(LoweredExpr), Hole}`
を使う `ArrayNew` equivalent、または dense `ArrayNew` と sparse
`ArrayNewSparse` の分割である。Either path must keep hole information until
backend array allocation writes the presence bitmap. Existing dense-only lowering
may remain as an optimization only when it proves every slot is present.

Array observability follows ECMAScript property presence:

- `arr.length` returns the logical length and counts holes.
- `arr[index]` returns `undefined` for holes and out-of-range indexes, but this does
  not make the index present.
- Supported numeric `index in arr` checks return true only when
  `0 <= index < length` and the presence bit is 1.
- `Array.prototype.map` checks presence before invoking the callback. It skips holes,
  preserves holes in the result, and stores callback results as present values even
  when the callback returns `undefined`.
- Array literal spread and call spread consume the array iterator/Get semantics.
  A source hole is read as `undefined`; the destination array element or argument is
  present `undefined`, not a preserved hole.

GC header の `body_size_bytes` から `ARRAY_HEADER_SIZE` を引き、`ARRAY_ELEM_SHIFT`
で割ることで、現 allocation が保持できる element capacity を求められる。
issue 300 の progress slice では、unused statement 形式の local-array
`arr.push(value);` だけを `ArrayPushGrow` lowering/backend path に落とし、
capacity が残っていれば in-place で length と element を更新し、足りなければ
payload を倍増させた新 array に copy して local を差し替える。expression value
として `push` の戻り値を観測する broader path や array-like object receiver は、
既存の `Array.prototype.push` runtime boundary に従う。

### Heap Object Layout (current)

Ordinary JS object の heap payload 形式。下位 3bit が `0b111` (`object` tag) の
RawValue が指す先。

```text
offset + 0  .. +4  : i32 property_count
offset + 4  .. +8  : i32 flags
offset + 8  .. +12 : i32 prototype_ptr   (raw heap pointer, object tag なし)
offset + 12 ..     : (key:value) entries × property_count

RawValue = ptr | OBJECT_TAG (ptr は 8-byte aligned)
```

`flags` フィールド:

| Bit | 定数 | 意味 |
|---|---|---|
| 0 | `OBJECT_FLAG_FROZEN` (1) | Object.freeze() — 全 property が non-writable, non-configurable |
| 1 | `OBJECT_FLAG_SEALED` (2) | Object.seal() — 全 property が non-configurable |
| 2+ | `OBJECT_NON_ENUM_SHIFT` | Per-property non-enumerable mask: bit `(2+i)` が 1 のとき property i は非列挙 |

各 entry は `OBJECT_ENTRY_SIZE` (8 bytes) 固定で、`(key_raw_value, value_raw_value)`
のペア。`key` は interned string の tagged `string` value、`value` は任意の
RawValue。

定数は `crates/runtime-abi/src/layout.rs` の `Layout` で定義:
- `OBJECT_HEADER_SIZE` = 12 (property_count + flags + prototype_ptr)
- `OBJECT_FLAGS_OFFSET` = 4
- `OBJECT_PROTOTYPE_OFFSET` = 8
- `OBJECT_ENTRIES_OFFSET` = 12
- `OBJECT_ENTRY_SIZE` = 8
- `OBJECT_ENTRY_SHIFT` = 3
- `OBJECT_VALUE_OFFSET` = 4

Heap number (`HEAP_NUMBER_SENTINEL = -1` が property_count に格納されている object)
は `object` tag を使うが、上記の property entry 形式ではなく固定 payload 形式を持つ。
ordinary object と区別するには `i32.load(payload_ptr) == HEAP_NUMBER_SENTINEL` を確認する。

### Private class private metadata

Class instances with lowered private fields use the GC header reserved word as packed
private metadata:

```text
bits 0..15   : private slot count
bits 16..31  : per-class private brand token
```

Mask/shift constants (defined in `crates/backend-wasm/src/expr_emit.rs`):
- `PRIVATE_FIELD_COUNT_MASK` = `0xffff` (lower 16 bits for slot count)
- `PRIVATE_FIELD_BRAND_SHIFT` = `16` (brand token shift)
- `PRIVATE_FIELD_SLOT_SIZE` = `4` (bytes per private slot)
- `CLASS_INSTANCE_PUBLIC_SLOT_CAPACITY` = `16` (public property slots reserved before private field payload)

`PrivateFieldGet` / `PrivateFieldSet` lowered runtime calls carry both the brand token
and the slot index. Same-class private field reads/writes, including non-`this`
receivers inside the declaring class, lower to these calls. The backend checks that the
value is an object, the packed brand matches, and the masked slot count is greater than
the requested slot before reading or writing private slot payload after the public
property capacity. A mismatch now routes through the cataloged
`private_brand_type_error` runtime helper. Without an active supported handler it
writes a `TypeError` message and aborts instead of returning `undefined` or silently
skipping a write; with an active handler it raises a catchable TypeError-like object
and lets the current `TryCatch` statement-boundary propagation bind it.

`PrivateBrandCheck(receiver, brand)` is available as a backend runtime-call primitive
for private method/accessor receiver checks. It validates the object tag and packed
brand, returns the checked receiver on success, and uses the same
`private_brand_type_error` path on mismatch. Same-class non-`this` instance private
method calls and getter reads lower their receiver through this helper. The backend
user-call emission recognizes that checked receiver position and short-circuits the
callee call when the helper raises `exception_pending`, so the private method/getter
body is not executed on mismatch in the supported statement-boundary `try/catch`
shape. Setter assignment, extracted private method/accessor values, and broader
expression-nested propagation remain future issue 351 / issue 255 work.

This is current progress toward ECMAScript private brands. Compatible catchable
`TypeError` propagation now covers lowered private field brand mismatches that occur
inside the supported `try/catch` statement shape, plus same-class non-`this` instance
private method calls and getter reads emitted as expression statements in that shape.
Other private accessor/method receiver forms remain issue 351 / issue 255 work until
expression-level exception propagation is generalized.

### BigInt value representation (accepted design)

BigInt は **heap object representation** を採用する。現行 `RawValue` の下位 3bit tag はすでに immediate / array / string / object で使い切っているため、BigInt 専用 immediate tag は追加しない。BigInt `RawValue` は `object` tag (`ptr | 0b111`) を使い、GC heap header の object kind で BigInt payload として判別する。

選択理由:

- BigInt は任意精度整数であり、small immediate だけでは ECMA-262 の値域を表せない
- `RawValue` の tag 空間を拡張すると既存 array/string/object wire encoding と backend helper 全体に波及する
- GC heap object に統一すると literal / arithmetic / boxed builtin boundary / future Wasm GC backend の差し替えを同じ論理 ABI で扱える

BigInt object payload は canonical little-endian limb sequence とする。Issue 259 implements the literal runtime slice for values that fit the current first-limb backend constructor path and stores a cached decimal representation after the canonical prefix for observable literal printing and `String(<literal>)`. Issue 260 keeps the signed-i64-backed dynamic unary-minus slice. Issues 382/397 add decimal-cache backed runtime `+` / `-` coverage for known BigInt operands outside signed i64 and supported if/else branch-assigned BigInt locals. Issue 383 adds decimal-cache backed runtime `*` coverage for known BigInt operands beyond signed i64, including branch/loop/switch/try assigned locals that remain BigInt. Issue 384 adds decimal-cache backed runtime `/` and `%` for known BigInt operands beyond signed i64; nested control-flow assignments for div/rem still conservatively invalidate tracked safe state and are split to issue 398. Remaining out-of-slice multi-limb runtime arithmetic is tracked by issue 369 before broad compatibility can be claimed.

```text
BigInt payload:

  +0: i32 sign        ; -1 negative, 0 zero, 1 positive
  +4: i32 limb_count  ; canonical zero は sign=0, limb_count=0
  +8: u64 limbs[limb_count] little-endian magnitude limbs
  +16: i32 decimal_len            ; issue 259 literal cache
  +20: u8 decimal[decimal_len]    ; no "n" suffix
```

Canonicalization rules:

- zero は必ず `sign=0, limb_count=0` に正規化する
- non-zero は `sign` を `-1` または `1` にし、最上位 limb の zero padding を持たない
- `-0n` は ECMA-262 と同じく `0n` と同一値に正規化する
- 文字列化は `n` suffix を付けない decimal string を返す

`RawValue` 判定は次の順で行う。

1. 下位 tag が immediate / array / string / object のどれかを判定する
2. object tag の場合だけ heap header kind を読む
3. heap kind が `bigint` のとき BigInt payload として扱う

BigInt object は GC mark 対象だが、payload は primitive limb array なので子参照を持たない。interned BigInt は当面作らず、literal lowering は runtime constructor で heap allocation する。

### BigInt runtime ABI boundary

BigInt を扱う runtime helper は論理 `jsval` を入出力に使う。現行 wasm wire では `RawValue i32` を返し、将来の `jsval i64` bridge では同じ論理 helper 名を維持する。

| Logical helper | Signature | First implementation owner | Notes |
|---|---|---|---|
| `make_bigint_literal` | `(sign, limb_count, limb_low, limb_high, ptr decimal, len) -> jsval` | issue 259 | Lowered literal decimal cache and canonical prefix を heap BigInt object に配置する |
| `bigint_to_string` | `(jsval) -> jsval` | issue 259/262 | issue 259 は literal-only `String(<BigInt literal>)` 用。broader builtin/coercion boundary は issue 262 |
| `bigint_to_boolean` | `(jsval) -> bool` | issue 259 | `0n` は false、それ以外は true |
| `bigint_strict_equal` | `(jsval, jsval) -> bool` | issue 261 | BigInt 同士は mathematical value 比較。Number とは常に false |
| `bigint_abstract_equal` | `(jsval, jsval) -> bool` | issue 261 | Number/String/Boolean との ECMA-262 coercion 境界を実装する |
| `bigint_compare` | `(op, jsval, jsval) -> jsval` | issue 261 | `< <= > >=`。成功時 bool `jsval`、例外時 pending exception |
| `bigint_add` / `sub` | `(jsval, jsval) -> jsval` | issue 260; future issue 369/370 | Known BigInt operands/results proven signed-i64-safe still use the issue-260 first-limb reconstruction path. Known BigInt operands outside signed i64 now use cached-decimal add/sub, including the supported if/else branch-assigned BigInt local shape validated by issue 397. Number 混在 is split to issue 370 for TypeError parity; broader unknown dynamic values remain issue 369 |
| `bigint_mul` / `div` / `rem` | `(jsval, jsval) -> jsval` | issue 260/263/383/384; future issue 369/370 | `mul` now uses cached decimal schoolbook multiplication for known BigInt operands outside signed i64, preserving sign and canonical zero for the validated local/literal and branch/loop/switch/try-assigned BigInt local slices. `div` / `rem` now use cached decimal long division for known BigInt operands outside signed i64 and preserve truncation, quotient sign, remainder sign, and canonical zero. Control-flow-assigned BigInt locals now preserve div/rem operand type through conservative if/else joins when all branches remain BigInt; Division/remainder by zero emits `RangeError: Division by zero` through the runtime exception substrate when uncaught and raises a catchable RangeError-like object with message parity when a supported `try/catch` is active. |
| `bigint_unary_minus` | `(jsval) -> jsval` | issue 260; future issue 369 | Progress slice implemented through signed-i64-backed first-limb reconstruction; full multi-limb fallback is issue 369. `-0n` は `0n` |
| `bigint_pow` | `(jsval, jsval) -> jsval` | issue 376; future issue 369/370 | Progress slice implemented for known BigInt operands/results proven signed-i64-safe through first-limb reconstruction. Negative exponent RangeError parity remains issue 370; full multi-limb fallback remains issue 369; out-of-slice dynamic exponentiation remains issue 376. |
| `bigint_bitwise_not` / `and` / `or` / `xor` | `(jsval[, jsval]) -> jsval` | issue 387; future issue 369/370 | Progress slice implemented for known BigInt operands/results proven signed-i64-safe through first-limb reconstruction. Static BigInt literal `~` folds with arbitrary decimal magnitude using `~x == -x - 1`, and static binary BigInt literal `&` / `|` / `^` fold through arbitrary-width two's-complement bits, so results outside the signed-i64 helper boundary can still lower as canonical BigInt literals. Issue 387 also folds local-known BigInt bitwise expressions outside signed i64 when operands remain statically tracked BigInt values, including constant-condition branch assignments where only one branch can execute and non-constant if/else assignments where both branch exits prove the same BigInt literal. Mixed Number/BigInt and unproved dynamic values stay diagnosed; operators never lower through ordinary number bitwise helpers. |
| `bigint_shl` / `shr` | `(jsval, jsval) -> jsval` | issue 378; future issue 369/370 | Static literal BigInt `<<` / `>>` fold to canonical BigInt literals for the validated bounded shift-count slice. Runtime helpers are not implemented for dynamic operands. BigInt `>>>` is an ECMAScript TypeError path, not a coercion to Number, and remains source-diagnosed in this slice. |

IR は BigInt literal と BigInt operations を phase-specific に扱う。

- Parser/frontend: BigInt syntax classification only。invalid literal syntax は issue 244 の diagnostics を維持する
- Resolver/BuiltinResolver: BigInt literal node を runtime-capable expression として残し、未実装 operation は該当 implementation issue ID を含む source diagnostic にする
- Lowering: literal は `BigIntLiteral { raw, radix, negative }` 相当の semantic/lowered node へ落とし、backend が runtime constructor を選ぶ。Known BigInt unary minus, `+`, `-`, `*`, signed-i64-safe `**`, and signed-i64-safe BigInt-specific `~` / `&` / `|` / `^` currently lower to runtime helpers when pre-lowering proof keeps them inside the signed-i64 helper slice. Known BigInt `+`, `-`, `/`, and `%` lower to cached-decimal runtime helpers for values outside signed i64 when operands remain tracked as BigInt; `+` / `-` also preserve the supported if/else branch-assigned operand shape. Literal BigInt `**` is folded before lowering only for non-negative exponent literals in `0..=64`; static literal BigInt `<<` / `>>` fold before lowering in the bounded shift-count slice, including negative shift counts by reversing direction. Dynamic BigInt exponentiation outside the signed-i64-safe proof boundary remains issue 376. Dynamic/out-of-slice BigInt shifts and BigInt `>>>` remain issue 378 diagnostics. BigInt bitwise and shift helpers do not share ordinary number bitwise/shift lowering. BigInt operation mixed Number/BigInt TypeError parity is implemented for the current runtime helper slice; current statically visible unsupported number-model mixes remain issue-linked diagnostics
- Backend/runtime link plan: BigInt helper は `RuntimeFn` catalog で deps/imports/capabilities/runtime strings を持つ。BigInt だけでは host import を要求しない

### Runtime exception diagnostic substrate

Runtime-generated JavaScript exceptions currently have a staged ABI. The implemented
substrate has two layers:

- Uncaught runtime exceptions emit the JavaScript error class/name and message through
  the same WASI `$write` path used by runtime diagnostics, then abort execution. This
  preserves the existing observable failure mode for top-level uncaught helper errors.
- When a supported `try/catch` is active, selected runtime helpers allocate an Error-like
  heap object with the matching builtin prototype and `message` property, store it in
  `$exception_pending`, return a dummy `undefined`, and let `TryCatch` bind and clear the
  pending object before executing the catch body.

This is not full ECMAScript completion-record propagation. The catchable slice is limited
to statement-boundary propagation through the current `TryCatch` emitter; nested
expression unwinding, `finally` interactions beyond the existing emitter shape, and
broader runtime helper adoption remain future exception-substrate work.

Issue 396 implemented these helpers behind one backend diagnostic/catchable-error emitter:

- `bigint_mixed_arithmetic_type_error(jsval, jsval) -> jsval` evaluates both operands
  before entering the helper. Without an active handler it writes `TypeError: Cannot mix BigInt and other types, use explicit conversions`
  and aborts; with an active handler it raises a catchable TypeError-like object whose
  `message` property contains the Node-compatible message text without the `TypeError:`
  diagnostic prefix.
- `bigint_division_by_zero_range_error() -> jsval` writes `RangeError: Division by zero`
  and aborts when uncaught; with an active handler it raises a catchable RangeError-like
  object whose `message` property is `Division by zero`. `bigint_div` / `bigint_rem`
  depend on this helper instead of inlining their diagnostic path, so RangeError uses
  the same cataloged runtime-string and `$write` dependency substrate as TypeError.
- `private_brand_type_error() -> jsval` writes `TypeError: Cannot read private member from an object whose class did not declare it`
  and aborts for uncaught private brand mismatches; with an active handler it
  raises a catchable TypeError-like object whose `message` property contains the same
  generic private-member message without the `TypeError:` diagnostic prefix. This shares
  the same runtime string, `$write`, Error-like allocation, and TypeError prototype
  substrate as the BigInt TypeError helper.
- Each helper declares its dependencies and runtime strings through the `RuntimeFn`
  catalog, so capabilities, exception globals, and string interning remain link-plan
  driven. The catchable helper path allocates only the minimal Error-like object shape
  needed by the current `e.message` differential fixtures.
- This is intentionally not yet full ECMAScript `throw` / `try` / `catch` propagation;
  broader completion-record unwinding remains future exception-substrate work.

Unsupported boundary:

- literal runtime values: implemented by issue 259 for decimal/binary/octal/hex literal construction, `console.log`, `typeof`, literal `String(...)`, and truthiness
- Full multi-limb BigInt arithmetic beyond the issue-260 signed-i64-backed progress slice, excluding the issue-382/397 known/supported-branch BigInt `+` / `-` slice, the issue-383 known/control-flow-BigInt `*` slice, and issue-384 known-BigInt `/` and `%` slice: `unsupported-bigint-arithmetic` / issue 369
- BigInt arithmetic exception parity: mixed Number/BigInt TypeError and division/remainder-by-zero RangeError now have uncaught diagnostic and supported `try/catch` Error-like object coverage through issues 380 and 381; broader completion-record unwinding remains future exception-substrate work
- Dynamic BigInt exponentiation outside the signed-i64-safe helper proof boundary: `unsupported-bigint-exponentiation` / issue 376
- BigInt dynamic bitwise NOT and binary AND/OR/XOR beyond the signed-i64-safe helper slice: `unsupported-bigint-bitwise` / issue 387
- Dynamic/out-of-slice BigInt shift operators and unsigned-right-shift TypeError policy: `unsupported-bigint-shift` / issue 378
- BigInt equality, relational comparison, and coercion until issue 261: `unsupported-bigint-comparison` / issue 261
- BigInt builtin functions and string conversion until issue 262: `unsupported-bigint-builtin` / issue 262

Broad BigInt implementation must not be hidden inside parser, backend emitter, or generic number helpers. Each slice must update docs/current-state/tests with Node differential evidence for the exact operation class it enables.

## Planned Heap GC strategy (017a)

This section records the planned GC model for the current runtime. The implementation is not in this issue.

### Chosen strategy

**Stop-the-world mark-and-sweep** is selected as the baseline strategy.

Rationale:

- 長寿命の `closure`/`class`/`module` オブジェクトが増えるケースを安全側に扱える
- `arena` は明示的な生存区間が必要で、現行の型付けと実行モデルでは閉じ込めが困難
- `string`/`array`/`object` が同一ヒープを使う現状では、最初に `mark+list-based sweep` を導入するのが既存 runtime の変更面積を最小化できる

### Planned heap object header

GC enabled allocation uses a **separate runtime header** before each heap block.
ユーザから観測される `RawValue` ポインタは header より `GC_HEADER_SIZE` 分だけ進んだ本体先頭を指す。

```text
obj_ptr + -16: i32 flags_and_type    ; mark bit + type bits
obj_ptr + -12: i32 body_size_bytes   ; this object's payload size in bytes (aligned)
obj_ptr + -8 : i32 sweep_next        ; freelist / sweep list linkage
obj_ptr + -4 : i32 gen_or_reserved   ; optional next-generation field (future)
obj_ptr      : payload
```

Flag layout:

- bit0: `mark` (1 = live in current mark cycle)
- bit1: `finalizable` (reserved)
- bits2-4: heap kind (`001` string, `010` array, `011` object)
- bits5-31: reserved

### Heap payload layout (planned)

The existing logical payload shape is kept; header is additive.

- `string`: `[len:i32, bytes... ]`
- `array`:  `[len:i32, elem0, elem1, ...]` (`i32` raw values)
- `object`: `[property_count:i32, prototype_ptr:i32, (key:value)×N]`

`object` は既存仕様に合わせて `prototype_ptr` を保持し、`[[Prototype]]` 走査は将来の markフェーズと連携させる。

### Closure heap object ABI

Escaping ordinary functions and arrows are represented as GC-managed heap
objects. The low-bit `RawValue` tag remains `object` (`ptr | 0b111`); no new
low-bit closure tag is introduced in this ABI slice. The closure subtype is
identified by the first payload word, so existing value-tag checks can keep
treating closure values as heap object values while runtime helpers can route
them away from ordinary property-entry scanning.

```text
RawValue closure value:

  closure_value = closure_payload_ptr | OBJECT_TAG

GC header:

  flags/type kind = GC_KIND_OBJECT

Closure payload:

  +0  i32 object_subtype     ; CLOSURE_SENTINEL (-2), distinct from property_count (>=0)

CLOSURE_SENTINEL は `crates/backend-wasm/src/expr_emit.rs` で定義 (値: -2)。
ordinary object の `property_count` は常に 0 以上なので、負の値と比較して判別する。
  +4  i32 code_id            ; lowered function identity, stable within module
  +8  i32 capture_count      ; number of RawValue capture slots
  +12 i32 env_flags          ; reserved, must be 0 in the immutable-env slice
  +16 i32 capture0           ; RawValue
  +20 i32 capture1           ; RawValue
  ...
```

The closure payload deliberately does not reuse the ordinary object
`property_count/prototype_ptr/(key:value)` layout. Generic object-property
helpers must detect `CLOSURE_SENTINEL` before interpreting the payload as
property entries. Until function metadata/prototype work extends this contract,
closure objects expose no own JS properties through the generic object table.

`code_id` is the lowered `FuncId` ordinal for a compiler-generated wasm
function. The function body ABI is unchanged from the current non-escaping
closure path: user arguments are followed by hidden immutable capture
parameters in the lowered function parameter order. A heap-closure dispatch
helper loads `code_id`, selects the generated target for the requested arity,
loads captures from the payload in order, and calls the target with
`user_args..., captures...`.

Mutable captured bindings are not represented by this immutable closure payload.
Until a separate environment-cell object contract exists, mutable capture forms
must remain an issue-linked diagnostic rather than being lowered to stale
captured values.

GC marking for closure objects is part of the closure ABI:

```text
mark_object_payload(payload):
  if i32.load(payload + 0) == CLOSURE_SENTINEL:
    count = i32.load(payload + 8)
    for i in 0..count:
      mark_value(i32.load(payload + 16 + i * 4))
    return
  scan ordinary object prototype and property entries
```

Closure allocation must keep all evaluated capture values rooted before calling
`$alloc_heap`; the newly-created closure value is then mirrored into the caller
root slot like any other heap value. This preserves captured heap objects across
allocation pressure and after the declaring function's activation has returned.

### GC trigger points

`$alloc_heap` は以下を満たすと GC または memory growth を試行する:

- `alloc_bytes_since_last_gc + requested_block_size >= GC_THRESHOLD` かつ
  bump allocation result が現在の `memory.size` の末尾 12 pages 以内に入る場合
- 現在の `memory.size` が `MEMORY_MAX_PAGES` に達しており、bump allocation
  result が現在の memory に収まらない場合は、free-list scan の前に last-chance
  GC を実行する
- bump allocation result が `MEMORY_MAX_PAGES * WASM_PAGE_SIZE` を超える場合は、
  free-list scan と明示 OOM trap の前に last-chance GC を実行する
- 現在の memory に収まらない場合は bounded `memory.grow` を試みる

Pseudo flow:

1. markフェーズ: ルートとして `globals` / `runtime stacks` / `module cache` を走査
2. sweepフェーズ: 生存フラグがないブロックを free-list へ回収
3. GC 後は bump allocation cursor を再計算する。sweep が heap 末尾まで続く unmarked range を見つけて `$heap` を range 先頭へ戻した場合、同じ allocation がその top-of-heap garbage を即時再利用できる
4. free-list に十分な block があれば再利用する。block が要求 payload より十分大きい場合は、要求分と remainder free block に split する。sweep は free-list 内の最大 body size も記録し、要求 payload がそれを超える場合は `$alloc_heap` の線形 free-list scan を省略する。sweep が heap 末尾まで続く unmarked range を見つけた場合は、その range を free-list に入れず `$heap` を range 先頭へ戻す
5. 必要なら `memory.grow`。要求 pages が `MEMORY_MAX_PAGES - memory.size`
   を超える場合は、失敗する `memory.grow` を発行せず明示的に trap する

### Safety and compatibility notes

- この設計は最初の実装では stop-the-world 全停止 GC とし、同時実行は対象外
- mark ビットは各 GC サイクルで反転 bit を使って O(1) リセットする方式を採用（全ヒープ走査の clear を回避）
- 文字列 primitive の一時 `scratch` は現在どおり GC 対象外

## RuntimeFn Catalog

runtime 関数は `RuntimeFn` カタログとして管理する（catalog 化が完了すれば linker が単一導線になる）。
現状は巨大な WAT template として `runtime_builder.rs` に存在するが、以下の形へ移行する。

```rust
pub struct RuntimeFn {
    pub name: RuntimeSymbol,
    pub deps: &'static [RuntimeSymbol],
    pub imports: &'static [HostImport],
    pub capabilities: &'static [Capability],
    pub emit: fn(&mut ModuleBuilder),
}
```

### Tree-shaking 方針

`console.log` を使っていない program には `$log`, `$write`, `fd_write` を含めない。
`+` 演算子を使っていない program には `$add`, `$concat` を含めない。
これは `RuntimeFn.deps` と `RuntimeFn.capabilities` を静的に解析することで実現する。

## Host Import ABI

host import は capability manifest から生成する。
backend が直接 import 文字列を持つことは禁止（`RuntimeLinkPlan` 由来に限定する）。

### Direct eval execution strategy

Static-string direct `eval(...)` does not add a runtime helper or host import.
The frontend/resolver/lowering slice parses the eval source at compile time and
expands the supported eval-code statements into caller-scope `Lowered IR`.
The backend therefore emits ordinary wasm for the expanded statements, and the
capability manifest remains standalone unless the expanded code itself uses a
capability such as `console.log` / WASI stdout.

Dynamic eval strings cannot be executed by pure wasm without a JavaScript
interpreter or host eval capability. A future dynamic-eval slice must use an
auditable Node host shim import, mark `standalone: false`, list the exact
`host.eval.*` import in `node_host.imports`, and record the matching
`capability_reasons` entry before backend emission.

### API 分類

| 分類 | 説明 | 例 |
|---|---|---|
| Wasm-native | runtime 内で完結する | 算術演算、string concat、===  |
| WASI-backed | fd_read / fd_write / clock_time_get などで実装 | console.log, Date.now |
| Host-backed | Node.js host が必要 | process.argv, fs.readFileSync |
| Unsupported | compile error。必要なら fallback plan を表示 | crypto.createHash |

この分類は `docs/03-api-and-host-capability.md` と `docs/11-shared-definitions.md` に正式定義する。
backend が API 分類を決めることは禁止。必ず capability manifest / semantic pass を通す。

## Bump Allocator

`$heap` global は `HEAP_START` 初期値を持つ。
string alloc 時は以下の手順で行う。

```text
1. align_to($heap, ALIGN) → base
2. i32.store(base, len)
3. $copy(src, base + 4, len)
4. $heap = align_to(base + 4 + len, ALIGN)
5. return base | STRING_TAG
```

### OOM Handling

`$alloc_heap` は `memory.size` を使用して利用可能なメモリをチェックする。
割り当てが現在のメモリサイズを超える場合、bounded `memory.grow` を試みる。
小さい growth は `MEMORY_MAX_PAGES` に収まる範囲で最低 16 pages ずつ要求し、
GC の直後に小刻みな `memory.grow` と sweep を繰り返す状態を避ける。
現在の `memory.size` がすでに `MEMORY_MAX_PAGES` の場合、または bump allocation
result が `MEMORY_MAX_PAGES * WASM_PAGE_SIZE` を超える場合は、`memory.grow` が
失敗して trap する前に last-chance GC を実行し、直後の free-list scan で
回収済み block を再利用できるようにする。
Sweep が heap 末尾まで続く unmarked range を回収する場合は、free-list 登録ではなく
`$heap` の tail trim として処理し、以後の bump allocation がその末尾領域を直接再利用
できるようにする。
`MEMORY_MAX_PAGES` まで増やせなければ `unreachable` で trap する。
このとき allocator はまず必要な page 数と残り page 数を比較し、要求が残り
`MEMORY_MAX_PAGES - memory.size` を超える場合は `memory.grow` 前に明示 trap する。
これにより、大きな割り当てによる未定義動作やメモリ破損を防ぐ。
現在の上限は 185 pages で、ABC451 D の depth-8 large live-set reducer はこの cap で
Node と同じ `292743` を出力する。184 pages では同じ reducer が `$alloc_heap` で trap
するため、現在の default はこの reducer を通す最小確認値として扱う。意図的な
large allocation fixture は引き続き `unreachable` で trap し、OOM 境界を保持する。

## GC Strategy

### Choice: Mark-and-Sweep GC

初期実装ではシンプルな mark-and-sweep GC を採用する。

**理由:**
- 現在の bump allocator からの移行が容易
- Arena allocator は allocation pattern の大幅な変更が必要
- 短命プログラム (CLI tools) では GC 頻度が低く、パフォーマンス影響が限定的
- 将来的に generational GC への移行が可能

### Heap Object Header Design

すべての GC-managed heap allocation は payload の直前に以下の header を持つ:

```text
payload_ptr - 16 .. -12 : i32 flags_and_type    ; mark bit + heap kind
payload_ptr - 12 .. -8  : i32 body_size_bytes   ; aligned payload size
payload_ptr - 8  .. -4  : i32 sweep_next        ; free-list linkage
payload_ptr - 4  ..  0  : i32 reserved          ; ordinary object private-slot count, 0 otherwise
payload_ptr             : type-specific payload
```

`flags_and_type` encoding:
- bit 0: mark bit (`GC_MARK_FLAG` = 0x1)
- bit 1: reserved finalizer bit (`GC_FINALIZABLE_FLAG` = 0x2)
- bits 2-4: heap kind (shift `GC_KIND_SHIFT` = 2, mask `GC_KIND_MASK` = 0x1c)
- bits 5-31: reserved

GC kind values:

| 定数 | 値 | 種別 |
|---|---|---|
| `GC_KIND_UNKNOWN` | 0 | 未指定 (current `$alloc_heap(size)` ABI) |
| `GC_KIND_STRING` | 4 | String |
| `GC_KIND_ARRAY` | 8 | Array |
| `GC_KIND_OBJECT` | 12 | Object / Closure / Heap Number |
| `GC_KIND_BIGINT` | 16 | BigInt |

定数は `crates/runtime-abi/src/layout.rs` の `Layout` で定義:
- `GC_HEADER_SIZE`: 16
- `GC_BODY_SIZE_OFFSET`: 4
- `GC_SWEEP_NEXT_OFFSET`: 8
- `GC_RESERVED_OFFSET`: 12

`RuntimeConst::ABI_VERSION` (現在値: 1) は layout/tag/offset 定数が変更された
ときにインクリメントする。`layout.rs` の `abi_layout_golden_snapshot` テストが
全定数を文字列スナップショットとして記録しており、定数を変更するたびに
スナップショット期待値を更新し、同時に `ABI_VERSION` をバンプしなければ
コンパイルが通らない構造になっている。

### GC Trigger Points

GC は以下のタイミングで実行:

1. **Allocation threshold**: `alloc_bytes_since_last_gc + requested_block_size >= GC_THRESHOLD` かつ
   bump allocation result が現在の memory 末尾 12 pages 以内に入る場合
   - `GC_THRESHOLD` は初期値として 64KB
   - `GC_HEADROOM_PAGES` は 12 pages。ABC451 depth-8 live-set reducer を
     維持しながら depth-9 1,000,000-allocation telemetry の sweep pressure を下げる
   - threshold は GC 後に動的に調整可能

2. **Max-cap last chance**: `memory.size == MEMORY_MAX_PAGES` かつ bump allocation
   result が現在の memory に収まらない場合、または bump allocation result が
   `MEMORY_MAX_PAGES * WASM_PAGE_SIZE` を超える場合。これは free-list scan と
   OOM trap の前に行い、回収可能 block を再利用できるか確認する。

3. **Explicit collection**: 将来的に `gc()` API を追加可能

### Mark Phase

Mark phase は以下の root set から開始:

1. Global variables (将来の実装)
2. Top-level local root table and active function call-frame roots
3. Runtime strings (interned strings は GC 対象外)

Function call-frame roots are stored in a fixed root-frame stack allocated once
from `_start` as part of the GC root table allocation. Function entry registers a
frame containing the previous-frame pointer, slot count, and mirrored local
slots; every function return unregisters that frame before returning the saved
result. This avoids allocating during function prologue, so heap parameters are
not exposed to collection before registration. Backend-owned temporary locals
are cleared from the mirrored root slots at statement boundaries; user locals
stay rooted until reassignment or frame pop so block-scope root narrowing does
not introduce extra free-list fragmentation in recursive array workloads.

Mark algorithm:

```
mark(root):
  if root is heap object:
    if not marked:
      set mark bit
      for each reference in object:
        mark(reference)
```

### Sweep Phase

Sweep phase は heap を走査し、unmarked objects を回収:

```
sweep():
  ptr = HEAP_START
  while ptr < $heap:
    size = i32.load(ptr)
    type_tag = i32.load(ptr + 4)
    if not marked:
      free(ptr, size)
    else:
      clear mark bit
    ptr += size
```

Committed runtime also keeps `$gc_free_list_max_body_size` as a conservative
upper bound for free-list reuse. Sweep resets it and raises it while adding
free blocks; allocation uses it only to skip scans when the requested aligned
payload is larger than every block discovered by the last sweep. Sweep also
tracks `$gc_free_list_second_max_body_size` so allocation can tighten the
summary after consuming or splitting a block whose original body size matched
the maximum. If later free-list reuse makes either value stale, it may
overestimate and allow an unnecessary scan, but it must not underestimate and
skip a reusable block.
When a coalesced unmarked range reaches the current `$heap` end, sweep lowers
`$heap` to the start of that range and exits instead of linking the tail into
the free list. This keeps the high-water bump pointer from staying pinned by
dead top-of-heap objects and avoids scanning a tail block that future bump
allocation can reuse directly. Allocation recomputes its bump cursor after the
collection hook, so a tail-trimmed `$heap` can satisfy the same allocation
attempt before `memory.grow` or the explicit OOM trap.

### Implementation Notes

- 初期実装では stack locals の追跡は簡略化 (GC 時に stack frame を走査)
- Interned strings は GC 対象外 (static data segment)
- 将来的に write barrier を追加して generational GC へ移行可能

## Module Cache Layout

Module キャッシュは `crates/runtime-abi/src/layout.rs` で定義された固定サイズのエントリ配列。
動的 `import()` の解決結果をキャッシュする。

```text
MODULE_CACHE_MAX = 64               ; 最大同時キャッシュ数
MODULE_CACHE_ENTRY_SIZE = 8         ; 1 エントリ: i32 loaded_flag + i32 value

Entry layout:
  +0  i32 loaded_flag  ; 1 = loaded, 0 = empty
  +4  i32 value        ; RawValue (exported module namespace object)
```

## ABI Versioning

ABI のレイアウト / tag / offset 定数は `RuntimeConst::ABI_VERSION` (現在値: 1) で
管理する。定数を変更するたびにバージョンをインクリメントする。

機械的検証として `crates/runtime-abi/src/layout.rs` の `abi_layout_golden_snapshot`
テストが全定数を文字列スナップショットとして記録しており、定数を変更するたびに
スナップショット期待値を更新し、同時に `ABI_VERSION` をバンプしなければ
コンパイルが通らない構造になっている。

### Backward compatibility

`ABI_VERSION` が v1 の間は backward-compat archive は不要。v1 以降にバンプした
場合は、古い wasm モジュールとの互換性検証のために `compat/vN-snapshot.txt`
形式の参照ファイルを作成する。
