# IR Contracts

This document maps compiler representation boundaries. Current implementation details live in code; accepted migration policy lives in `docs/27-ir-layer-completion.md`.

## Layer Contract

```text
Source text
  -> AST
  -> BuiltinResolved AST
  -> HIR
  -> MIR
  -> LoweredProgram compatibility bridge / backend emission
```

The default compiler path still emits through validated `LoweredProgram`. HIR/MIR is the migration path and must be strengthened without breaking the default path.

## AST

AST represents parsed syntax. It may contain TypeScript-only constructs, import/export forms, class elements, optional chaining, sequence expressions, decimal/bigint literals, and parser-only syntax families.

AST invariants:

- Every user-facing node keeps source/span information where the parser can provide it.
- AST does not encode runtime helper choice, host imports, WAT snippets, or memory layout.
- TypeScript-only syntax is classified as erase, preserve, lower, or reject by `docs/28-frontend-syntax-ownership.md`.
- `validate_ast` rejects syntax shapes that later phases cannot safely interpret.

## Builtin-Resolved AST

Resolution classifies names, bindings, builtin identities, and direct eval source handling.

Invariants:

- Unresolved globals are allowed only through the documented builtin/host/test allowlist.
- Unsupported builtins produce specific diagnostics where possible.
- Resolution must not choose backend runtime helpers directly; it records semantic identity.
- Eval completion/declaration plans must be internally consistent before lowering.

## HIR

HIR owns JavaScript semantic operations before backend representation choices.

HIR owns:

- completion records and control-flow completion shape,
- semantic local/function ids,
- builtin semantic operations,
- relational/logical operations,
- frontend-independent unsupported diagnostics for unimplemented semantics.

HIR must not contain backend strings such as WAT snippets, `host.*` import names, or `RuntimeFn` symbol names. `validate_hir` rejects backend/capability leakage and impossible semantic states.

## MIR

MIR is the backend-facing native path. It can mention runtime calls and backend-visible value operations, but it must be validated before emission.

MIR invariants:

- local/function/module ids must reference existing declarations,
- runtime calls must match catalog signatures,
- object/array/private-field operations must preserve runtime ABI expectations,
- host/capability behavior still flows through runtime catalog and link plan,
- `Validated<MirProgram>` is preferred for APIs that emit native MIR.

## LoweredProgram

Legacy `LoweredProgram` remains the default backend input. It is also the compatibility bridge for HIR-to-MIR work.

Lowered invariants:

- `Validated<LoweredProgram>` is required before backend WAT emission.
- Residual unsupported constructs (`this`, method calls, invalid locals, impossible module ids) are invariant errors before WAT generation.
- Runtime helpers appear as cataloged `RuntimeFn`, not raw WAT snippets.
- Direct memory/layout behavior uses `runtime-abi` constants.

## Dumps And Snapshots

- `dump --ast` exposes parser shape.
- `dump --resolved` exposes name/builtin resolution.
- `dump --tir` exposes HIR.
- `dump --optimize` exposes optimized HIR slices.
- `dump --lowered` exposes `LoweredProgram`.
- MIR/HIR snapshot tests must cover every supported variant.

## Adding An IR Feature

1. Define the owner layer here before coding.
2. Add the IR variant or semantic operation in the owner crate.
3. Update validation in the same change.
4. Update dump/snapshot output.
5. Add focused positive and negative tests.
6. Update runtime catalog/ABI docs if the feature reaches backend/runtime.
