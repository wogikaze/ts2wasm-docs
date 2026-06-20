# IR Layer Completion

## Goal

Move the compiler toward a validated HIR/MIR path while preserving the current legacy LoweredProgram build route until parity is proven.

## Completion criteria

A native IR path is complete when:

- frontend syntax is lowered to semantic HIR without losing diagnostics,
- HIR validation catches invalid semantic states,
- MIR validation catches backend-shape violations,
- MIR emission covers the same fixture semantics as legacy emission for the selected slice,
- compat fallback is no longer needed for that slice,
- dumps/snapshots expose the path for debugging.

## Current modes

| Mode | CLI flag | Behavior |
|---|---|---|
| Legacy | default | Builtin-resolved AST -> LoweredProgram -> WAT |
| Strict HIR/MIR | `--experimental-hir-mir` | HIR/MIR must succeed; otherwise build errors |
| Compat fallback | `--experimental-hir-mir-compat-fallback` | Try HIR/MIR; fallback to legacy on unsupported MIR |

## Work slices

- Expression and literal parity.
- Control-flow/completion record parity.
- Function/class/runtime-call parity.
- Module/export parity.
- Runtime ABI and capability parity.

Each slice needs focused fixtures, dumps, validation tests, and backend parity checks.
