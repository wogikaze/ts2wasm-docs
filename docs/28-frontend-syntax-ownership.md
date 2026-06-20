# Frontend Syntax Ownership Contract

## Ownership categories

| Category | Meaning | Diagnostic expectation |
|---|---|---|
| Parse-and-erase | TypeScript-only syntax accepted then removed before runtime lowering. | silent when erased correctly; precise TS diagnostic when missing |
| Parse-and-preserve | Syntax kept for module/declaration shape but not runtime behavior. | module/TS diagnostics when unsupported |
| Parse-and-lower | Frontend rewrites syntax into executable JS semantics. | TS/runtime diagnostics when lowering cannot represent behavior |
| Reject | Parser recognizes enough to report a precise error. | `UnsupportedTypeScriptSyntax`, `UnsupportedModule`, or `SyntaxError` |
| Runtime-bearing | Syntax reaches IR/runtime/backend as executable semantics. | runtime subset/builtin diagnostics when unsupported |

## AST coverage anchor

`crates/syntax/src/ast.rs` currently represents import/export forms, let/var/assignment, if/loops/switch/try/throw, functions/generators/async flags, classes/static blocks/private elements, enums, labels, break/continue, arrays/objects/spread, optional chaining/calls/indexing, BigInt, decimal numbers, `new.target`, `import.meta`, `this`, and sequence expressions.

AST support is not the same as semantic/runtime support. Always check fixtures and lowerer/runtime coverage.

## TypeScript-only syntax

Type annotations, interfaces, type aliases, generic syntax, ambient declarations, `as`, `satisfies`, const assertions, type-only imports/exports, overload signatures, and related forms are frontend-owned. The frontend should erase, preserve for module shape, or reject precisely.

## Module syntax

Import/export syntax is frontend plus compiler/module-graph owned. Runtime module behavior is handled by lower/static-import stages and module runtime helpers.

## Rule for adding syntax

1. Add token/parser AST support.
2. Decide one ownership category above.
3. Add parser snapshot and either erasure/lowering/rejection tests.
4. Update fixture catalog and this doc if category changes.
