# Project Definition

## One sentence

`ts2wasm` is a Rust compiler/runtime project that translates a practical TypeScript/JavaScript subset into auditable WebAssembly, primarily for WASI Preview 1 and optionally for a Node-compatible host shim.

## Goals

- Compile useful JS/TS fixtures to valid wasm without bundling a full JavaScript engine.
- Keep host capabilities explicit through a manifest and link plan.
- Preserve JavaScript semantics where implemented; reject or diagnose unsupported semantics rather than silently changing behavior.
- Make each compiler phase inspectable through CLI dump commands and tests.
- Grow compatibility through fixtures, reference coverage, and issue-backed slices.

## Non-goals

- Full ECMAScript conformance in one step.
- Replacing QuickJS/Javy/Node as a general JS runtime.
- Running arbitrary Node programs without a capability boundary.
- Treating TypeScript types as a full type checker inside the compiler. TypeScript syntax is parsed/erased or rejected according to frontend ownership rules.
- Hiding semantic gaps behind best-effort output.

## Current product surface

- `ts2wasm build` writes wasm and optionally a capability manifest.
- `ts2wasm check` validates parse/type-reference level input.
- `ts2wasm dump` exposes tokens, AST, resolved AST, typed IR, optimized IR, lowered IR, WAT, and unparse output.
- `ts2wasm server` provides JSONL batch compilation for coverage runners.

## Design values

- **Capability first**: host imports must be justified and visible.
- **Phase ownership**: frontend owns syntax, IR owns semantic operations, runtime owns JavaScript runtime helpers, backend owns wasm emission.
- **Small verified slices**: every new feature needs a focused fixture/test and a clear unsupported fallback.
- **Code-derived docs**: docs describe Why and contracts; code remains the source for exact implementation.

## What does not exist

- No microservice architecture.
- No database schema or migration layer.
- No production web API server; `server` is a compiler batch protocol over stdin/stdout.
- No browser runtime target in the current implementation.
- No implemented `wasm32-wasi-gc` or `wasm32-component` backend yet.
