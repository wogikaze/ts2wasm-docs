# Compatibility and Semantics

## Compatibility strategy

Compatibility grows by small, evidence-backed slices. A feature is treated as implemented only when code, fixture/catalog status, and a focused gate agree.

## Current fixture status

| Fixture directory status | Count |
|---|---:|
| `pass` | 29 |
| `unknown` | 7 |

| Feature matrix status | Count |
|---|---:|
| `pass` | 29 |
| `partial` | 5 |

## Supported families at a glance

The fixture catalog currently contains pass coverage for core expressions/statements, arrays/objects, classes, async-await fixtures, module-system fixtures, builtins/I/O, type-erasure, and negative diagnostics. Unknown directories remain in the catalog and must not be described as fully supported.

Feature family coverage density:

| Feature family | Count |
|---|---:|
| `expr` | 65 |
| `stmt` | 51 |
| `control` | 29 |
| `builtin` | 24 |
| `value-types` | 23 |
| `class` | 22 |
| `fn` | 18 |
| `string` | 17 |
| `ts` | 13 |
| `module` | 11 |
| `io` | 10 |
| `obj-kernel` | 9 |
| `obj-semantics` | 9 |
| `obj` | 8 |
| `misc` | 7 |
| `array` | 7 |
| `linker` | 5 |
| `test-infra` | 3 |
| `async` | 2 |

## Semantics rules

- Prefer correct rejection over silent behavior drift.
- Unsupported source should use a specific `DiagCode` when possible: module, TypeScript syntax, eval, builtin, runtime subset, Date, RegExp, unresolved name/function, etc.
- Differential tests compare Node output to generated wasm/iwasm output for semantic fixtures.
- Negative tests must assert diagnostics, not just non-zero exit.

## TypeScript handling

TypeScript syntax is frontend-owned. Some syntax is parsed and erased, some is preserved for module/declaration shape, and some is rejected with precise diagnostics. See `docs/28-frontend-syntax-ownership.md`.

## Runtime subset handling

The compiler does not claim full JS runtime coverage. Runtime behavior must be backed by `RuntimeFn` specs and fixtures. Where runtime helpers are host-backed, capability docs and host-deny tests should cover the boundary.

## Reference suites

Reference coverage currently tracks test262, tsc, and tsgo through generated matrix artifacts. Treat those artifacts as snapshots, not as promises that every executed case has semantic parity.
