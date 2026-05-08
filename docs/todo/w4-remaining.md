# W4 (Builtin runtime) — remaining work

Last updated: 2026-05-08

## Current status

- **18 runtime_*.rs files** in `crates/backend-wasm/src/`
- **83 open runtime issues** across all runtime categories
- **3 open child issues** under meta-issue 5004 (Runtime Builtins Coverage)
- **~4,272 lines** in `runtime_fn.rs` + `runtime_fn_impl.rs` combined (the RuntimeFn catalog)
- **~30,723 lines** total in `crates/backend-wasm/src/`

### Implemented builtins

| Category | Status | Details |
|---|---|---|
| **Math** | Mostly done | floor, ceil, round, abs, max, min, pow, trunc, sign, random |
| **String** | Partial | charAt, charCodeAt, fromCharCode, indexOf, lastIndexOf, slice, substring, substr, split, replace, replaceAll, toLowerCase, toUpperCase, trim, padStart, padEnd, repeat, includes, match, search, at, localeCompare, isWellFormed, toWellFormed, HTML wrappers, indexing syntax |
| **Array** | Partial | push, pop, at, slice, concat, join, reverse, indexOf, includes, find, filter, every, some, findLast, findLastIndex, flatMap, flat, copyWithin, with, isArray, from, toReversed, toSorted, toSpliced, values, keys, entries, toString, shift, unshift, splice |
| **JSON** | Partial | parse (primitives, arrays, objects, escapes, unicode surrogates, malformed input rejection), stringify (primitives, arrays, objects, string escaping, space, replacer arrays, function replacers) |
| **Date** | Partial | new Date(epoch-ms), getTime, valueOf, now, getTimezoneOffset, toString, toISOString, local getters, Annex B getYear/setYear/toGMTString (diagnosed) |
| **RegExp** | Basic | test, exec, dot, digit, word, plus, star, question flags; compile fixture reports issue 051 |
| **Map/Set** | Basic | Set (add, has, delete, forEach, size, clear), Map (basic), SameValueZero for Set |
| **BigInt** | Extensive | Literal and runtime arithmetic (+,-,*,/,%), Bitwise (&,|,^,~), Shift (<<,>>), Comparison (===,==,<,>,<=,>=), BigInt(), asIntN, asUintN, ToString, mixed-type operation diagnostics |
| **Object** | Partial | keys, values, entries, assign, create, defineProperty, freeze, getOwnPropertyDescriptor, hasOwn, is, hasOwnProperty |
| **Error types** | Basic | Error, TypeError, RangeError with message; instanceof Error works |
| **Global functions** | Partial | isNaN, parseInt, parseFloat, isFinite, encodeURI, decodeURI, escape, unescape |
| **console** | Partial | .log via fd_write |

### Missing builtins (blocks test262 pass)

| Category | Impact | Details |
|---|---|---|
| **Promise** | Critical (~hundreds of test262 cases) | Promise constructor, .then, .catch, .finally, Promise.all, Promise.race, Promise.resolve, Promise.reject, async/await |
| **Proxy** | Critical (~hundreds) | Proxy constructor, handler traps (get, set, has, deleteProperty, apply, construct, etc.), revocable |
| **Symbol** | Medium (~dozens) | Symbol constructor, Symbol.prototype, well-known symbols, Symbol.for, Symbol.keyFor |
| **TypedArrays** | Medium (~dozens) | Int8Array through Float64Array, .buffer, .byteOffset, .byteLength, set, subarray, from, of |
| **ArrayBuffer/SharedArrayBuffer** | Medium | Constructor, slice, isView, transfer |
| **Reflect** | Low | All Reflect.* methods |
| **Intl** | Low | Intl.DateTimeFormat, Intl.NumberFormat, Intl.Collator |
| **Atomics** | Low | Atomics.* on SharedArrayBuffer |
| **WeakMap/WeakSet** | Medium | Constructor, get, set, has, delete |
| **Promise combinators** | Low | Promise.allSettled, Promise.any, Promise.withResolvers |

### String/Array methods needing semantic parity

Many String and Array methods are build_smoke only and lack semantic differential testing or full edge-case handling:

- [ ] **String.replace/replaceAll**: Full RegExp match semantics, capture groups, replacement patterns.
- [ ] **Array.sort**: Currently reports issue 299 for unsupported forms.
- [ ] **Array.flat/flatMap**: Depth handling, sparse array holes.
- [ ] **Array.reduce/reduceRight**: Not listed in build_smoke tests — check implementation status.
- [ ] **String.match/search**: Full RegExp semantics (global flag, sticky, capture groups).
- [ ] **String.split**: RegExp separator, limit parameter edge cases.
- [ ] **String.localeCompare**: Intl-dependent behavior.

## Remaining work by category

### P0: Promise (highest test262 impact)

- [ ] Promise constructor with executor function
- [ ] Promise.prototype.then / catch / finally
- [ ] Promise.resolve / reject
- [ ] Promise.all / race / allSettled / any
- [ ] async/await lowering and runtime
- [ ] Microtask queue / execution order

### P1: Proxy and Reflect

- [ ] Proxy constructor + handler traps (13 fundamental traps)
- [ ] Proxy.revocable
- [ ] Reflect.construct / apply / get / set / has / deleteProperty / etc.

### P1: TypedArrays and ArrayBuffer

- [ ] TypedArray constructors (all 11 types)
- [ ] TypedArray.prototype methods (set, subarray, slice, from, of, map, filter, etc.)
- [ ] ArrayBuffer / SharedArrayBuffer
- [ ] DataView

### P1: String method semantic parity

- [ ] String.prototype.replace with RegExp callback
- [ ] String.prototype.matchAll
- [ ] Full RegExp match in split, search, match

### P2: Remaining builtins

- [ ] WeakMap / WeakSet
- [ ] Symbol (well-known symbols critical for builtin correctness: Symbol.iterator, Symbol.toStringTag)
- [ ] Atomics
- [ ] Intl (basic DateTimeFormat/NumberFormat)

### Testing

- [ ] **Upgrade build_smoke to semantic_diff**: Many builtin methods are build_smoke only; add Node/iwasm differential tests for semantic verification.
- [ ] **test262 regression ramp**: Run test262 after each builtin addition to capture pass/fail deltas.

## Key open issues

- `issues/open/5004` — Meta: Runtime Builtins Coverage (3 open children)
- `issues/open/050-implement-date.md` — Full Date implementation
- `issues/open/052-implement-json.md` / `052d` — JSON implementation
- `issues/open/313-implement-array-builtin.md` — Array builtin coverage
- `issues/open/342-implement-object-builtin-coverage.md` — Object builtin coverage
- `issues/open/419-implement-builtin-api.md` — General builtin API coverage
- `issues/open/416-implement-async.md` — async/await implementation

## Reference

- Runtime function catalog: `crates/backend-wasm/src/runtime_fn.rs`, `runtime_fn_impl.rs`
- Builtin routing: `crates/ir/src/lowered/program_builtins.rs`
- Test file: `crates/cli/tests/m6_builtin_methods.rs` (109 test functions, mostly build_smoke)
- Gate G: test262 semantic_pass >= 50% (~26,700)
- Gate H: test262 semantic_pass >= 90% (~48,100)
