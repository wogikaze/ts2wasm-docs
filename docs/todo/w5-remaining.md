# W5 (Runtime semantics) — remaining work

Last updated: 2026-05-08

## Current status

Runtime semantics covers the non-builtin JS execution semantics: object model, prototype chain, `this` binding, closures, classes, exceptions, modules, control flow, destructuring.

- **330 open child issues** under meta-issue 5001 (Semantic Analysis Coverage)
- **46 test functions** in `m2_node_diff.rs` (semantic diff tests)
- **15 build_smoke** in `m7_control_flow.rs`
- **9 build_smoke** in `m8_oop_classes.rs`
- **43 build_smoke** in `m9_modules.rs`

### What works (semantic_diff parity with Node)

- **Control flow**: if/else, while, for, for-in, for-of, do-while, switch/case, switch fallthrough, break, continue, labeled statements
- **Exceptions**: throw, try/catch/finally, nested try/catch, rethrow, exception escaping top level → trap
- **Primitive semantics**: truthiness, ===/!==, abstract equality, + operator, typeof, delete, in operator, template literals, logical assignment
- **Arrow functions**: local binding calls, expression bodies, block bodies, captured locals, lexical `this`, closures preserved across GC pressure
- **Non-arrow functions**: direct calls, object-receiver method calls, immutable outer-local capture, basic arguments.length/indexed reads
- **Class (instance)**: constructor, methods, private fields, private methods, private getters/setters, static methods, static blocks, `new` expressions, `extends`, `super()` calls, derived class private fields
- **Class (static)**: static private fields, static private methods, static private accessors, source-order initialization
- **Destructuring**: array declarations, object declarations, function parameters, arrow params, array elisions, rest bindings, nested destructuring, default initializers
- **Module**: Static named ES module import/export, alias import, repeated source imports, missing export diagnostics
- **Property access**: Static member access, computed member access, length/index on arrays, string indexed access
- **Property mutation**: Property set/delete, dynamic property assignment
- **Spread**: Array literal spread, call spread, object literal spread (static, dynamic, mutated objects)
- **Direct eval**: Static-string `eval(expr)` for expression statements with caller-local scope access
- **instanceof**: Ordinary class constructors, inherited prototype chains, Object.setPrototypeOf
- **BigInt/BigInt comparison**: ===, !==, ==, !=, <, >, <=, >= via mathematical value (sign + decimal)
- **Rest parameters**: zero, one, multiple extra arguments
- **Binary WASM emitter**: Partial — `binary_mvp.rs` handles hello-world level programs

### What doesn't work (gaps at test262 limit 500)

Based on feature breakdown:
- **eval** = 148 cases: Most test262 eval tests use dynamic eval strings, not static-string eval (only static-string eval supported)
- **function resolution** = 70 cases: Partially overlaps with W3 (UnresolvedFunction); semantic handling of non-builtin function calls
- **regexp-literal** = 45 cases: Many test262 RegExp tests use complex patterns/flags not yet parsed

### Broader semantic gaps

| Area | Impact | Description |
|---|---|---|
| **Prototype chain completeness** | Critical | Object.create/prototype chain for builtin interop; `Symbol.hasInstance`, `Symbol.toPrimitive`, `Symbol.toStringTag`, `Symbol.iterator` not wired |
| **Iterator protocol** | Critical | `[Symbol.iterator]()` → `next()` — required by for-of, spread, destructuring, Array.from, Promise.all, Map/Set, etc. |
| **Complete completion records** | High | Proper [[Type]]/[[Value]]/[[Target]] for all statement types; return/throw from generators; break/continue to non-existent labels |
| **Generators / async / async generators** | High | `function*`, `yield`, `yield*`, `async function`, `await`, `for-await-of` |
| **Object model** | High | Property attributes (writable, enumerable, configurable); Object.defineProperty runtime semantics; property descriptor invariants; [[GetPrototypeOf]], [[SetPrototypeOf]] |
| **Module live bindings** | Medium | ES module live binding updates; re-export mutation visibility |
| **Closure escape** | Medium | Returned/escaping closures with mutable capture; heap closure object for ordinary functions (basic heap closure exists for immutable) |
| **This binding** | Medium | Global `this` at top level (currently unsupported); strict-mode `this` for function calls without receiver |
| **Super** | Medium | super.property in method calls; super() in derived constructors (basic exists) |

## Remaining work

### P0: Iterator protocol

- [ ] Implement `[Symbol.iterator]` for Array, String, Map, Set
- [ ] Implement `Iterator.prototype.next()`, `return()`, `throw()`
- [ ] Wire iterator-based semantics for: for-of, spread, destructuring, Array.from
- [ ] Implement `Generator.prototype` for `function*` (if implementing generators)

### P0: Prototype chain for builtin interop

- [ ] Wire `Symbol.hasInstance` for instanceof
- [ ] Wire `Symbol.toPrimitive` for object-to-primitive coercion (affects Date, String wrapper, Number wrapper)
- [ ] Wire `Symbol.toStringTag` for Object.prototype.toString
- [ ] Wire `Symbol.iterator` for iteration

### P0: Complete completion records

- [ ] Proper completion type propagation for all statement types (if, switch, try, for, while, etc.)
- [ ] `return` outside function → SyntaxError or runtime
- [ ] Break/continue with labels across nested constructs (basic exists)
- [ ] Completion records for statement boundary throw propagation

### P1: Generator functions

- [ ] `function* gen() { yield ... }` parsing (basic exists, issue-353)
- [ ] Generator runtime: next(), return(), throw()
- [ ] async/await on top of Promise
- [ ] async generators + for-await-of

### P1: Object model completeness

- [ ] Property descriptor creation/modification with full attribute support
- [ ] [[GetPrototypeOf]] / [[SetPrototypeOf]] runtime
- [ ] Object.seal, Object.preventExtensions, Object.isSealed, Object.isFrozen, Object.isExtensible
- [ ] Accessor properties vs data properties distinction in runtime

### P2: Module live bindings

- [ ] Live binding mutation propagation across module boundaries
- [ ] Module namespace objects (`import * as ns from "..."`)
- [ ] Dynamic import (`import("...")`)
- [ ] Circular module dependency evaluation order

### P2: Closure completeness

- [ ] Mutable capture environments for escaping closures
- [ ] Multiple escaping closures sharing captured variables

## Key open issues

- Meta-issue 5001: Semantic Analysis Coverage (330 open children)
- `issues/open/416-implement-async.md`
- `issues/open/417-implement-async-iteration.md`
- `issues/open/418-implement-break-continue.md`
- `issues/open/420-implement-call-expression.md`
- `issues/open/421-implement-class.md`
- `issues/open/274-implement-spread-operator.md`
- `issues/open/021-implement-full-wasm-backend.md`

## Reference

- Semantic diff tests: `crates/cli/tests/m2_node_diff.rs` (46 test functions)
- Control flow tests: `crates/cli/tests/m7_control_flow.rs` (15 functions, build_smoke)
- Class tests: `crates/cli/tests/m8_oop_classes.rs` (9 functions, build_smoke)
- Module tests: `crates/cli/tests/m9_modules.rs` (43 functions, build_smoke)
- IR: `crates/ir/src/lowered/resolver.rs`, `resolver_expr.rs`
- Gates G/H: test262 semantic_pass >= 50% / 90%
