# Execution Model and Targets

## Execution model

A program is compiled to a wasm module with a small ts2wasm runtime. JavaScript values are represented as tagged `i32` values. Heap values live in wasm linear memory. Runtime helpers are linked according to a `RuntimeLinkPlan`, so a program includes only required runtime functions, globals, strings, host imports, and capabilities.

## Implemented targets

| CLI alias | Canonical manifest target | Status | Notes |
|---|---|---|---|
| `wasm32-wasi`, `wasm32-wasi-p1` | `wasm32-wasi-p1` | implemented | standalone WASI Preview 1 path |
| `wasm32-wasi+node-host`, `wasm32-wasi-p1+node-host`, `wasm32-wasi-p1+node-shim` | `wasm32-wasi-p1+node-shim` | implemented | allows explicit Node-compatible host shims |

## Reserved targets

| Target | Status | Rule |
|---|---|---|
| `wasm32-wasi-gc` | known but unimplemented | CLI rejects build with `UnsupportedTarget` |
| `wasm32-component` | known but unimplemented | CLI rejects build with `UnsupportedTarget` |

Do not document reserved targets as usable until `ExecutionTarget::is_implemented()` returns true and backend emission/gates exist.

## Build output

`ts2wasm build` emits a wasm binary. The compiler asks the backend for wasm bytes, writes those bytes directly, and attaches ABI metadata derived from the selected `ExecutionTarget`. Build success is determined by the backend binary emitter; unsupported binary shapes fail the build instead of succeeding through WAT conversion. WAT remains a dump/debug format and is not the build artifact.

When `--emit-manifest` is passed, the compiler emits a canonical capability manifest from the validated runtime link plan.

## CLI contract

```bash
ts2wasm build input.ts -o output.wasm   [--emit-manifest manifest.json]   [--host-deny]   [--explain-unsupported]   [--experimental-hir-mir | --experimental-hir-mir-compat-fallback]   [--target wasm32-wasi]

ts2wasm check input.ts

ts2wasm dump [--tokens|--ast|--resolved|--tir|--optimize|-O 0..3|--lowered|--wat|--unparse] input.ts

ts2wasm server
```

`--experimental-hir-mir` and `--experimental-hir-mir-compat-fallback` are mutually exclusive.

## Server protocol

`ts2wasm server` reads one JSON request per line and writes one JSON response per line. It supports single-file and batch requests. Batch mode is used by reference coverage runners and has bounded worker count, timeout handling, and panic aggregation.

## Host-deny behavior

`--host-deny` rejects programs that need Node host imports. Use it for standalone/WASI-only validation.
