# W8 (Optimization) — remaining work

Last updated: 2026-05-08

## Status: deferred

W8 is explicitly marked as **post-90% test262** scope. No active optimization work should proceed until Gate H (test262 semantic_pass >= 90%) is achieved, except for **correctness-preserving obvious optimizations** that don't affect semantic pass/fail behavior.

## Current state

- **No conscious optimization pass** exists in the compiler pipeline.
- **WAT text format** is the primary emission target; binary WASM emitter (`binary_mvp.rs`) is minimal.
- **Performance baseline**: No benchmark suite or regression tracker exists.
- **Optimization levels**: CLI defines `-O0` through `-O3` with safety modes but lacks meaningful differentiation.
- **TypeScript type hints**: `check_typescript_file` can extract optimization candidates (number/string types) but these are not used by the backend.

## What could be done after 90%

### Benchmark infrastructure

- [ ] **Benchmark suite**: Compile representative JS/TS programs and measure execution speed, memory usage, and wasm size.
- [ ] **Benchmark tracker**: `mise run benchmark-tracker` should exist (described in Gate G but not implemented).
- [ ] **Regression detection**: Automated comparison against previous benchmark results.

### Optimization passes

- [ ] **Typed fast path**: Use TypeScript type annotations to skip runtime type checks.
- [ ] **Packed array**: Store arrays of known-typed elements (e.g., all integers) in compact representation.
- [ ] **Devirtualization**: Direct call dispatch instead of method lookup for known receiver types.
- [ ] **Property inline cache**: Cache property lookup results for hot paths.
- [ ] **Dead code elimination**: Remove unreachable branches and unused variable assignments.
- [ ] **Constant folding**: Extend existing compile-time constant evaluation.
- [ ] **Inlining**: Inline small functions, especially for known builtin calls.

### Binary WASM emitter

- [ ] **Full binary emitter**: Replace WAT text format dependency with direct binary WASM emission (issue 021).
- [ ] **Size optimizations**: Use WASM encoding tricks (block params, shortened locals encoding, etc.).
- [ ] **Streaming emission**: Emit WASM binary in one pass without intermediate WAT.

### WAT text format reduction

- [ ] **Replace giant WAT templates**: `runtime_core_emitter_part1.rs` (1,305 lines) and `part2.rs` (1,535 lines) use raw WAT; replace with typed writers.
- [ ] **ABI bridge**: Fix discrepancy between logical ABI (`i64` JsVal) and wire representation (`i32` tagged RawValue).

## Key open issues

- `issues/open/021-implement-full-wasm-backend.md`
- Gate G: benchmark suite fixed / regression detection

## Reference

- Optimization modes: `docs/11-shared-definitions.md` §Optimization and safety modes
- Binary emitter: `crates/backend-wasm/src/wasm_binary.rs`, `binary_mvp.rs`
- Coding standard: `docs/12-coding-standard.md` §1.2 (WAT禁止 — raw WAT in runtime is allowed with justification)
