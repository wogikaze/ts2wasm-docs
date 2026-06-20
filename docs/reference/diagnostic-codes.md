# Diagnostic Codes Reference

Every diagnostic emitted by the compiler carries a `DiagCode` variant that
classifies the kind of error or unsupported construct. This document is the
canonical listing; see `crates/diagnostic/src/lib.rs` for the enum definition.

| Code | Phase | Severity | Description |
|------|-------|----------|-------------|
| `UnresolvedName` | resolve | error | A name referenced in source was not declared in any enclosing scope. |
| `UnresolvedFunction` | resolve | error | A function called in source was not declared in the program. |
| `DuplicateFunction` | resolve | error | Two functions share the same name in the same program. |
| `DuplicateLocal` | resolve | error | Two local bindings share the same name in the same lexical scope. |
| `DuplicateParameter` | resolve | error | Two parameters share the same name in the same function parameter list. |
| `DuplicateLabel` | resolve | error | Two labels share the same name in the same label scope. |
| `UnresolvedLabel` | resolve | error | A label referenced in break/continue was not declared in any enclosing scope. |
| `ClassSideEffect` | resolve | error | A name binding is a class declaration and cannot be assigned to. |
| `NumberOutOfRange` | lower | error | A number literal is outside small-int tagged range. |
| `ArityMismatch` | lower | error | A function call passes the wrong number of arguments. |
| `InvalidTopLevelReturn` | lower | error | `return` is used in top-level script scope. |
| `InvariantViolation` | any | internal | A lowered IR node violates a structural invariant — this is a compiler bug. |
| `SyntaxError` | parse | error | Source code contains invalid ECMAScript syntax. |
| `UnsupportedSyntax` | parse | error | Source uses syntax that is not supported in the current milestone. |
| `UnsupportedBuiltin` | lower | error | Source references a builtin API outside the current supported subset. |
| `UnsupportedDate` | lower | error | Source uses Date behavior outside the current deterministic Date subset. |
| `UnsupportedRegExp` | lower | error | Source uses RegExp behavior outside the current supported RegExp subset. |
| `UnsupportedModule` | lower | error | Source uses module loading/resolution behavior outside the current subset. |
| `UnsupportedEval` | lower | error | Source uses eval behavior outside the current direct-eval subset. |
| `UnsupportedTypeScriptSyntax` | parse | error | Source uses TypeScript syntax that is not yet parsed or erased. |
| `UnsupportedRuntimeSubset` | lower | error | Source reaches a runtime/lowering subset boundary rather than parser syntax. |
| `BackendIo` | emit | internal | I/O or command execution failure at the backend boundary. |
| `TypeScriptTypeCheck` | resolve | error | TypeScript compiler API reported a type-checking diagnostic. |
| `UnsupportedTarget` | emit | error | Target execution environment is not supported by the current backend. |
