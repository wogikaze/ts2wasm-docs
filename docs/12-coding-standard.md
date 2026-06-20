# Coding Standard

Formatter/linter-enforceable rules belong in tools. This document covers project-specific design rules that tools cannot fully enforce.

## Diagnostics

- Prefer a specific `DiagCode` over generic `UnsupportedSyntax`.
- Preserve source spans for user-facing errors.
- Attach `phase` when a pipeline stage wraps a diagnostic.
- Unsupported behavior must name the semantic gap, not only the syntax token.
- Do not turn internal invariant failures into user-facing unsupported diagnostics.

## Compiler Phase Ownership

| Phase | Owns | Must not own |
|---|---|---|
| `frontend` / `syntax` | lexing, parsing, TypeScript erasure shape, AST spans | runtime helper choice |
| `resolve` | bindings, scope, import binding rewrite | builtin runtime lowering |
| `semantics` / `ir` | builtin identity, semantic operations, HIR/MIR/Lowered invariants | WAT strings, host import policy |
| `runtime-catalog` | runtime helper specs, dependencies, signatures, imports, capabilities | syntax parsing |
| `backend-wasm` | WAT/WASM emission from validated IR and validated link plan | frontend AST interpretation |

Backend code must not add new direct reads of frontend `Stmt`/`Expr`. New semantics flow through IR.

## Runtime And ABI

- Use `ValueTag`, `TaggedValue`, `HeapPtr`, `Layout`, and `RuntimeConst` instead of raw numbers.
- RuntimeFn dependencies, imports, capabilities, globals, runtime strings, and signatures must be declared in catalog specs.
- Host imports require capability reasons and `--host-deny` coverage when the path is user-facing.
- ABI breaking changes require runtime ABI version/snapshot updates and `docs/14-runtime-abi.md` updates.
- Runtime strings/data layout belong in runtime catalog or `runtime-abi`, not scattered backend constants.

## IR Change Procedure

1. Define the owner layer in `docs/13-ir-contracts.md`.
2. Add the variant or operation in the owner crate.
3. Update validation before backend emission can observe the new shape.
4. Update dumps/snapshots so review can see the new representation.
5. Add focused semantic or negative tests.
6. Update runtime catalog/ABI docs if the feature reaches runtime.

## RuntimeFn Change Checklist

> **FROZEN**: `RuntimeFn` variant addition is prohibited (P5 constraint).
> Use `SpecOp` (`crates/spec-kernel/`) for new operations.
> Existing `RuntimeFn` entries are for backward compatibility only.

- Adding new `RuntimeFn` variants is REJECTED by CI.
- For new semantics, add a `SpecOp` variant in `crates/spec-kernel/src/spec_op.rs`.
- For the runtime implementation, add a builder in `crates/backend-wasm/src/runtime/spec/`.
- Connect via `crates/backend-wasm/src/spec_emit.rs`.
- Existing `RuntimeFn` entries should be migrated to `SpecOp` per P5 plan.

## Host Capability Checklist

- Every host or WASI power has a manifest key and auditable reason.
- `standalone=true` output has no Node host imports.
- `--host-deny` rejects programs that need Node host power.
- Test-only host hooks are labeled as test-only and do not leak into normal target claims.

## Tests

- Add a focused fixture or unit test for every semantic behavior change.
- Negative tests assert diagnostic code/category, not only message substrings.
- Snapshot updates must be paired with a reason.
- Reference coverage failures must remain visible; do not hide them with broad skips.
- Build-smoke tests must be named/described so they do not imply semantic parity.

## Review Reject Conditions

Reject a change when any of these are true:

- It bypasses gates or hooks with `--no-verify`.
- It adds host imports without manifest reasons.
- It changes ABI tags/layout without a version/snapshot/doc update.
- It sends frontend AST directly to backend emission.
- It introduces broad skips or untracked unsupported buckets.
- It edits generated coverage artifacts without naming the generator or reason.
