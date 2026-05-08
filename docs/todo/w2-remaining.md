# W2 (Syntax coverage / parser) — remaining work

Last updated: 2026-05-08

## Current status

- **12 parser files** in `crates/frontend/src/parser/`
- **UnsupportedSyntax = 204** at test262 limit 500 (the #1 blocker for test262 pass rate)
- **376 open child issues** under meta-issue 5000 (TypeScript Compiler Parser Syntax Coverage)
- **64 done children** out of 440 total under meta-issue 5000
- Parser handles: most ECMAScript expression/statement syntax, class declarations, TypeScript-specific constructs, destructuring patterns
- **Frontend crate** includes lexer, parser, AST definitions, diagnostics, span tracking

### Known parser gaps at test262 limit 500

Based on feature breakdown (204 UnsupportedSyntax at limit 500), the major syntactic blockers are:

| Syntax feature | Approx count | Notes |
|---|---|---|
| RegExp literal (with flags) | ~45 | regexp-literal feature; basic regexp compiles, many patterns not handled |
| eval-related syntax | ~38 | UnsupportedEval category; static string eval works, dynamic eval blocked |
| Function syntax variants | ~6 | Some function forms not parsed |
| Duplicate function definitions | ~2 | DuplicateFunction reported |
| Various parser syntax gaps | rest | Distributed across less frequent constructs |

## Remaining work

### Parser gap closure (highest impact on test262)

- [ ] **RegExp literal flags**: Implement all RegExp flag combinations (`g`, `i`, `m`, `s`, `u`, `y`, `d`). Issue 332.
- [ ] **Remaining expression forms**: SequenceExpression, yield/generator, async/await, optional chaining, nullish coalescing.
- [ ] **Remaining statement forms**: `with` statement, `debugger`, Annex B block-level function hoisting in non-strict mode.
- [ ] **Cover initializers**: `for (var x = y in obj)` forms.
- [ ] **Labelled function declarations**: Annex B.3.2.
- [ ] **Hashbang/HTML comments**: Basic support exists; need broader test262 coverage.

### TypeScript parser gaps (lower priority for test262)

- [ ] **JSX**: ~8 tsgo cases blocked by JSX parsing.
- [ ] **Decorators**: ~4 tsgo cases blocked.
- [ ] **Parameter properties**: ~2 tsgo cases blocked.

### Architecture

- [ ] **Parser test coverage**: Add parser-specific tests for edge cases discovered through test262 ramp.
- [ ] **Diagnostic quality**: Ensure all unsupported syntax produces source-spanned diagnostics with issue IDs.

## Key open issues

- `issues/open/059-implement-parser-syntax-extensions.md` — meta for parser syntax extension work
- `issues/open/332-*` — RegExp flag support (if exists)
- Meta-issue 5000 with 376 open child issues

## Reference

- Parser files: `crates/frontend/src/parser/statements*.rs`, `expressions*.rs`, `binding_patterns.rs`, `helpers.rs`, `tokens.rs`
- Gate B: `test262 limit 500` で `UnsupportedSyntax` ゼロ
- Feature priority: P1 for all standard ECMAScript syntax
