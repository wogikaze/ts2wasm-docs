# Current State

Last audited: 2026-06-15.

This page records what is true now. Final-state design contracts live in the other `docs/` files. Do not turn this page into a roadmap or issue backlog.

## Current Gate

Use the manager entrypoints first; `mise run <task>` aliases are acceptable when available.

```bash
python scripts/manager.py check
python scripts/manager.py gate-fast
python scripts/manager.py nextest
python scripts/manager.py coverage-report --format markdown

# Rust-native test262 reference coverage runner
cargo nextest run -p ts2wasm-cli --test reference_coverage --no-fail-fast
# single category
cargo nextest run -p ts2wasm-cli --test reference_coverage -- t262_builtins_json
```

Generated coverage artifacts must name their generator command. Do not hand-edit generated tables to make a gate look fresh.

## Repository Shape

- Rust workspace crates: 15 (`backend-core, backend-wasm, cli, compiler, diagnostic, frontend, ir, iwasm-runner, resolve, runtime-abi, runtime-catalog, semantics, shared, source, syntax`).
- Runtime catalog variants detected from source: approximately 490 `RuntimeFn` entries, including pseudo-intrinsics.
- Fixture catalog directories: 36.
- Issues present: 466 Markdown files under `issues/`.

## Issue Inventory Snapshot

This is an inventory snapshot, not the issue source of truth.

| Status | Count |
|---|---:|
| `done` | 402 |
| `open` | 62 |
| `doing` | 2 |

| Priority | Count |
|---|---:|
| `P2` | 212 |
| `P1` | 189 |
| `P0` | 43 |
| `P3` | 22 |

## Fixture Snapshot

| Fixture directory status | Count |
|---|---:|
| `pass` | 29 |
| `unknown` | 7 |

| Feature matrix status | Count |
|---|---:|
| `pass` | 29 |
| `partial` | 5 |

Build success is not semantic evidence. A feature is semantically supported only when a fixture, differential run, or reference record proves it.

## Implemented Execution Targets

- `wasm32-wasi-p1` through aliases `wasm32-wasi`, `wasm32-wasi-p1`.
- `wasm32-wasi-p1+node-shim` through aliases `wasm32-wasi+node-host`, `wasm32-wasi-p1+node-host`, `wasm32-wasi-p1+node-shim`.

Known but unimplemented targets: `wasm32-wasi-gc`, `wasm32-component`.

## Current Implementation Notes

- The default compiler path still lowers through legacy `LoweredProgram`, but build writes backend-provided wasm bytes and no longer uses a `wat2wasm` CLI fallback or build-facing WAT conversion fallback.
- `backend-wasm` has an explicit native `LoweredProgram -> WasmModule -> wasm bytes` subset API for simple numeric/string console output, locals, arithmetic, `if`, `while`, direct user-function calls, static module export reads, and ABI custom-section emission. The default compiler `LoweredProgram` build path asks backend-wasm for ABI-bearing bytes; the public `emit_wasm_binary` entrypoint reports unsupported native shapes instead of falling back to WAT conversion, and native coverage still does not cover the full runtime helper catalog.
- `--experimental-hir-mir` and `--experimental-hir-mir-compat-fallback` are migration modes and must not be described as default parity.
- Static named ES module import/export has a narrow local differential slice; broader module semantics are not complete.
- Class, Node API, and broader module fixtures include build-smoke coverage that must not be reported as semantic parity.
- TypeScript syntax support is a mix of erase, preserve, lower, and reject; parser acceptance alone does not imply runtime support.
- Reference coverage artifacts may be sampled; always inspect the evidence command before quoting numbers.

## test262 Reference Coverage (2026-06-15)

Run with `cargo nextest run -p ts2wasm-cli --test reference_coverage --no-fail-fast` (in-process Rust compilation, persistent Node.js oracle for semantic comparison).

| Category | Pass | Mismatch | Unsupported | Total | Semantic % |
|---|---|---|---|---|---|
| **built-ins/** | | | | | |
| ArrayBuffer | 43 | 60 | 109 | 212 | 20% |
| BigInt | 1 | 53 | 23 | 77 | 1% |
| Boolean | 9 | 31 | 11 | 51 | 18% |
| DataView | 122 | 179 | 260 | 561 | 22% |
| Error | 20 | 22 | 16 | 58 | 34% |
| Function | 104 | 207 | 198 | 509 | 20% |
| JSON | 0 | 91 | 73 | 165 | 0% |
| Map | 16 | 83 | 105 | 204 | 8% |
| Math | 30 | 262 | 35 | 327 | 9% |
| Number | 42 | 266 | 32 | 340 | 12% |
| Promise | 54 | 327 | 296 | 677 | 8% |
| Proxy | 95 | 185 | 31 | 311 | 31% |
| Reflect | 16 | 127 | 10 | 153 | 10% |
| Set | 41 | 164 | 178 | 383 | 11% |
| String | 423 | 619 | 181 | 1223 | 35% |
| Symbol | 13 | 76 | 9 | 98 | 13% |
| WeakMap | 5 | 48 | 88 | 141 | 4% |
| WeakRef | 5 | 16 | 8 | 29 | 17% |
| WeakSet | 2 | 39 | 44 | 85 | 2% |
| Atomics | 15 | 104 | 263 | 382 | 4% |
| AggregateError | 8 | 17 | 0 | 25 | 32% |
| AsyncFunction | 4 | 13 | 1 | 18 | 22% |
| AsyncIteratorPrototype | 1 | 8 | 4 | 13 | 8% |
| GeneratorFunction | 4 | 15 | 4 | 23 | 17% |
| **annexB/** | | | | | |
| built-ins | 31 | 175 | 35 | 241 | 13% |
| language | 109 | 334 | 389 | 845 | 13% |
| **language/** | | | | | |
| arguments-object | 42 | 130 | 76 | 263 | 16% |
| destructuring | 11 | 2 | 5 | 19 | 58% |
| import | 9 | 18 | 155 | 182 | 5% |
| literals | 130 | 65 | 339 | 534 | 24% |
| rest-parameters | 3 | 1 | 7 | 11 | 27% |
| syntax categories | ~200 | 0 | 0 | ~200 | 100% |
| **language/expressions/** | | | | | |
| addition | 30 | 11 | 7 | 48 | 63% |
| arrow-function | 14 | 120 | 196 | 343 | 4% |
| assignment | 73 | 19 | 393 | 485 | 15% |
| async-function | 2 | 54 | 37 | 93 | 2% |
| async-generator | 4 | 491 | 128 | 623 | 1% |
| compound-assignment | 150 | 39 | 265 | 454 | 33% |
| dynamic-import | 15 | 69 | 913 | 997 | 2% |
| generators | 16 | 170 | 68 | 290 | 6% |
| other (38 categories) | ~550 | ~200 | ~550 | ~1300 | ~42% |

**Overall numbers**: ~2,600 semantic pass / ~5,200 mismatch / ~5,800 unsupported / ~13,600 total (c. 19% semantic pass rate across tested categories).

**Timed out** (>240s, needs further splitting): `language/expressions/class`, `language/expressions/function`, `language/expressions/object`, `language/expressions/yield`.

**Methodology**: Each test262 file is compiled in-process via the Rust compiler API. If compilation succeeds, the wasm binary is run through iwasm and the source is run through a persistent Node.js oracle (single `node` process with `vm.createContext` per file). Outputs are compared: match = semantic pass, differ = semantic mismatch. Compilation failures are classified by diagnostic code (unsupported = expected, InvariantViolation = compiler bug, other = fail).

## Next Useful Docs

- Compiler pipeline: `docs/04-compiler-architecture-and-runtime.md`
- IR boundaries: `docs/13-ir-contracts.md`
- Runtime ABI: `docs/14-runtime-abi.md`
- Test status schema: `docs/11-shared-definitions.md`
- Coverage matrix: `docs/15-coverage-matrix.md`
- Fixture matrix: `docs/26-semantic-feature-matrix.md`
