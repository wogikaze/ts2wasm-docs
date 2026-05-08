# W3 (Name resolution / scope) — remaining work

Last updated: 2026-05-08

## Current status

- **UnresolvedName = 120** at test262 limit 500 (#2 blocker)
- **UnresolvedFunction = 70** at test262 limit 500 (#3 blocker)
- **354 open child issues** under meta-issue 5005 (Name Resolution Coverage)
- **19 open child issues** under meta-issue 5006 (Scope Analysis Coverage)
- **216 open child issues** under meta-issue 5002 (Type System Coverage, depends on 5005)
- **20 open child issues** under meta-issue 5007 (Module Resolution Coverage)

### What works

- Basic global builtin resolution (Math, JSON, Object, Array, String, Number, etc.)
- Local variable/function name resolution
- Simple scope chain traversal
- Basic ES module static import/export name resolution
- TypeScript ambient declaration name registration

### What doesn't work (major categories)

The UNresolvedName=120 and UnresolvedFunction=70 at limit 500 break down as:

| Issue | Count | Description |
|---|---|---|
| Global builtins not routed | ~120 | Many global builtins (Symbol, Proxy, Reflect, Intl, Atomics, SharedArrayBuffer, etc.) not registered as resolvable names |
| Function dispatch failures | ~70 | Builtin functions not recognized as callable — includes String.prototype methods not routed, Array.prototype methods not routed, etc. |
| Module resolution | ~20 | Issue 5007 children: package resolution, bare specifiers, node_modules lookup, path mapping |

## Remaining work

### Global builtin registration (attack UnresolvedName)

- [ ] **Register all ECMAScript global builtins**: Symbol, Proxy, Reflect, Map, Set, WeakMap, WeakSet, Promise, Error types (EvalError, RangeError, ReferenceError, SyntaxError, TypeError, URIError, AggregateError), ArrayBuffer, SharedArrayBuffer, DataView, Atomics, Intl, globalThis, decodeURI, encodeURI, etc.
- [ ] **Register all TypedArray constructors**: Int8Array, Uint8Array, Uint8ClampedArray, Int16Array, etc.
- [ ] **Standard well-known symbols**: Symbol.iterator, Symbol.toStringTag, Symbol.hasInstance, etc.

### Function resolution (attack UnresolvedFunction)

- [ ] **Complete builtin method dispatch table**: Route all String.prototype methods, Array.prototype methods, Object.prototype methods, Number.prototype methods, Function.prototype methods.
- [ ] **typeof/instanceof builtins**: instanceof for builtin constructors, typeof for all value types.

### TypeScript-specific name resolution

- [ ] **Nested namespace/module resolution**: `A.B.C` forms.
- [ ] **Type-only imports**: `import type { ... }`.
- [ ] **Triple-slash directives**: `/// <reference path="..." />` support.
- [ ] **Module augmentation**: `declare module "..." { ... }`.

### Scope analysis

- [ ] **Block-scoped hoisting**: `let`/`const` temporal dead zone semantics.
- [ ] **Function hoisting edge cases**: Annex B block-level function hoisting.
- [ ] **Global scope pollution**: Interactions across module boundaries.

## Key open issues

- Meta-issue 5005 (Name Resolution Coverage) — 354 open, 52 done
- Meta-issue 5006 (Scope Analysis Coverage) — 19 open, 8 done
- Meta-issue 5007 (Module Resolution Coverage) — 20 open, 0 done
- Meta-issue 5002 (Type System Coverage) — 216 open, 8 done

## Reference

- IR resolver: `crates/ir/src/lowered/resolver.rs`, `resolver_expr.rs`, `resolver_extra.rs`
- Name resolver: `crates/ir/src/name_resolver.rs`
- Builtin routing: `crates/ir/src/lowered/program_builtins.rs`
- Gate E: test262 build_pass >= 5,000 (~10%)
