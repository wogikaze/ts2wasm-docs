# Runtime ABI

## ABI Identity

- ABI name: `ts2wasm-runtime-abi`
- Current runtime ABI version: `2`
- ABI metadata is attached to generated wasm from the selected `ExecutionTarget`.
- Source of truth for constants: `crates/runtime-abi/src/*`.

## Tagged Values

The current wasm runtime represents JavaScript values as tagged `i32` bits. This is distinct from logical `AbiType::JsVal`; do not treat them as interchangeable without an explicit conversion boundary.

Low 3 bits hold the tag. Upper bits hold either a small-int payload or an aligned heap pointer.

| Tag | Constant | Meaning |
|---:|---|---|
| `0` | `ValueTag::UNDEFINED` | `undefined` |
| `1` | `ValueTag::NULL` | `null` |
| `2` | `ValueTag::FALSE` | `false` |
| `3` | `ValueTag::TRUE` | `true` |
| `4` | `ValueTag::NUMBER` | small integer payload or reserved numeric sentinel |
| `5` | `ValueTag::ARRAY` | heap array pointer |
| `6` | `ValueTag::STRING` | heap string pointer |
| `7` | `ValueTag::OBJECT` | heap object pointer |

Number payloads use `NUMBER_SHIFT = 3`. Reserved number payloads encode NaN, infinities, negative zero, builtin identity tokens, native error constructors, and direct-local user function identity tokens.

Use these wrappers instead of raw `i32` where possible:

- `TaggedValue`: complete tagged JS value.
- `HeapPtr`: aligned heap payload pointer.
- `RawLocalValue`: untyped local value only at migration boundaries.
- `ValueTag`: tag and sentinel constants.

## Linear Memory Regions

`Layout` owns all offsets and sizes. Backend/runtime code must not duplicate these values as magic numbers.

| Region | Contract |
|---|---|
| `DATA_START` | static string data/table region |
| `SCRATCH_OFFSET` / `SCRATCH_SIZE` | temporary string/output scratch |
| `STDIN_*` and `FILE_*` | WASI fd staging buffers and iovec slots |
| `HEAP_START` | dynamic heap start |
| `GC_HEADER_SIZE` | bytes reserved immediately before each GC-managed payload |
| `MODULE_CACHE_*` | fixed module cache table |
| `SYMBOL_*` / `WELL_KNOWN_SYMBOL_*` | symbol registry/cache slots |

Memory starts with bounded pages and may grow up to `MEMORY_MAX_PAGES`. Allocation must trap explicitly on OOM rather than silently overlapping static regions.

## Heap Object Layouts

All heap payloads are 8-byte aligned. GC-managed allocations have a header at `payload_ptr - GC_HEADER_SIZE`.

### Strings

String payloads begin with a 4-byte byte length followed by UTF-8 bytes. String values are tagged as `heap_ptr | ValueTag::STRING`.

### Arrays

Array payloads use:

| Offset | Field |
|---:|---|
| `0` | logical length |
| `ARRAY_CAPACITY_OFFSET` | backing capacity |
| `ARRAY_PRESENCE_WORD_COUNT_OFFSET` | sparse presence word count |
| `ARRAY_ELEMENTS_OFFSET_OFFSET` | element payload offset from payload start |
| `ARRAY_PRESENCE_WORDS_OFFSET` | first presence word |

Elements are tagged `i32` values. Sparse holes are represented by presence bits, not by writing `undefined` into absent slots.

### Objects

Object payloads use:

| Offset | Field |
|---:|---|
| `0` | property count or sentinel |
| `OBJECT_FLAGS_OFFSET` | frozen/sealed/enumerability/writable/configurable/accessor flags |
| `OBJECT_PROTOTYPE_OFFSET` | raw prototype heap pointer |
| `OBJECT_ENTRIES_OFFSET` | property entries |

Each property entry is `(key_raw_value, value)` and uses `OBJECT_ENTRY_SIZE`. Property attribute masks currently cover bounded small-object slices; broader descriptor storage must extend the ABI explicitly.

Builtin constructor globals are heap objects with `name`, `length`, and `prototype` data properties. They are registered during module start and referenced by `$builtin_ctor_*` globals. Error-family constructors may additionally use reserved NUMBER-tagged sentinel payloads (`ERROR_PAYLOAD`, `AGGREGATE_ERROR_PAYLOAD`) for backward-compatible reflective operations.

### BigInt And Heap Number

BigInt payload stores sign, limb count, first-limb low/high words, and a cached decimal spelling. Current dynamic BigInt support is a signed-i64/small multi-limb slice, not a complete arbitrary-precision engine.

Heap-number-like payloads use sentinel object-count values and cached decimal strings. Do not invent new sentinels outside `Layout`.

## GC Header

The GC header stores flags/type, aligned body size, sweep/free-list linkage, and reserved generation/finalizer metadata. Heap kind bits use `GC_KIND_*` constants.

Scanner expectations:

- String payloads have no tagged-value children.
- Array payloads scan present element slots.
- Object payloads scan property keys/values and prototype pointers.
- BigInt payloads scan no tagged children.
- Call-frame roots are managed as a LIFO stack by function entry/exit.

## Runtime Exception Substrate

Runtime helpers that model JS errors must use cataloged runtime functions and tagged values. Error constructor identity tokens use reserved numeric payloads. Compatible JS exception behavior must be covered by completion/exception tests before being described as semantic parity.

## Host Import ABI

WASI and Node/test host imports are selected by runtime catalog specs and validated by runtime link plans. Backend code consumes a validated plan; it must not append ad-hoc imports.

Host import changes require:

- runtime catalog import/capability entries,
- capability manifest reasons,
- link-plan and manifest equality tests,
- `--host-deny` coverage when user code can trigger the path.

## Runtime Catalog Connection

Runtime functions declare stack signatures and dependencies in `runtime-catalog`. If a function's parameter/result shape changes, update:

- `runtime-abi` signature or stack-effect definitions,
- `runtime-catalog` spec/signature,
- backend emitter calls,
- ABI/link-plan snapshot tests,
- this document.

## Versioning Rule

Bump the runtime ABI version when tags, layout offsets, wire value encoding, memory metadata, runtime signature shape, or ABI custom-section schema change in a backward-incompatible way. Update snapshots in the same change.
