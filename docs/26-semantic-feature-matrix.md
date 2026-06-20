# Semantic Feature Matrix

This document summarizes `fixtures/feature-matrix.yaml` and `fixtures/catalog.yaml` for human/agent navigation. The YAML files remain the source of truth.

## Fixture directory status

| Status | Count |
|---|---:|
| `pass` | 29 |
| `unknown` | 7 |

## Feature matrix status

| Status | Count |
|---|---:|
| `pass` | 29 |
| `partial` | 5 |

## Feature family density

| Family | Count |
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

## Fixture catalog summary

| Directory | Category | Status | Expected | Fixture count |
|---|---|---|---|---:|
| `arrays-objects` | `semantic` | `pass` | array and object property operations | 13 |
| `arrow-functions` | `semantic` | `pass` | arrow function syntax and semantics | 1 |
| `async-await` | `semantic` | `pass` | async/await compilation | 10 |
| `atcoder` | `semantic` | `unknown` | algorithmic stress tests from AtCoder problems | 1 |
| `basics-equality` | `semantic` | `pass` | equality operator semantics | 1 |
| `basics-hello` | `semantic` | `pass` | basic stdout/exit/stdin smoke tests | 4 |
| `basics-oom` | `semantic` | `unknown` | out-of-memory stress test | 1 |
| `basics-syntax` | `semantic` | `pass` | basic syntax acceptance tests | 2 |
| `basics-typeof` | `semantic` | `pass` | typeof operator semantics | 1 |
| `basics-types` | `type-erasure` | `pass` | TypeScript type annotation erasure verification | 13 |
| `basics-utf8` | `semantic` | `pass` | UTF-8 string handling | 1 |
| `builtins-and-io` | `semantic` | `pass` | built-in objects, methods, and I/O operations | 392 |
| `classes` | `semantic` | `pass` | class declaration and expression semantics | 21 |
| `classes-and-inheritance` | `semantic` | `pass` | class inheritance, super, and extends | 19 |
| `control-flow-and-exceptions` | `semantic` | `pass` | control flow and exception handling | 24 |
| `core-expressions` | `semantic` | `pass` | core expression forms (literals, operators, calls) | 70 |
| `core-semantics` | `semantic` | `pass` | core language semantics (equality, coercion, prototypes) | 573 |
| `core-statements` | `semantic` | `pass` | core statement forms (loops, conditionals) | 12 |
| `hir-support` | `semantic` | `pass` | HIR supported slice and UnsupportedSyntax boundary fixtures | 6 |
| `html-comments` | `parser` | `pass` | HTML comment parsing acceptance | 3 |
| `linker` | `build-smoke` | `unknown` | linker/static analysis tests | 5 |
| `module-system` | `semantic` | `pass` | ES module import/export semantics | 58 |
| `modules-and-typed-optimizations` | `semantic` | `unknown` | module and typed optimization passes | 7 |
| `negative` | `negative` | `pass` | compiler diagnostic tests for invalid inputs | 14 |
| `node-apis` | `semantic` | `unknown` | Node.js core API exercises | 12 |
| `object-semantics-kernel` | `semantic` | `pass` | Object property descriptor and attribute semantics | 15 |
| `parser` | `parser` | `unknown` | parser acceptance tests | 1 |
| `parser-errors` | `negative` | `pass` | parser error diagnostics | 1 |
| `primitives-control-flow` | `semantic` | `pass` | primitives and control flow basics | 8 |
| `qa-techniques` | `semantic` | `pass` | QA technique examples | 5 |
| `rest-parameters` | `semantic` | `pass` | rest parameter syntax and semantics | 1 |
| `spread-args` | `semantic` | `pass` | spread argument syntax | 4 |
| `stmt` | `semantic` | `pass` | statement forms with Node comparison | 36 |
| `test-infrastructure` | `test-infrastructure` | `pass` | test framework self-verification | 3 |
| `this-binding` | `semantic` | `pass` | this binding semantics | 1 |
| `typescript-directives` | `parser` | `unknown` | TypeScript directive parsing | 6 |

## Feature group summary

| Fixture group | Status | Feature families |
|---|---|---|
| `arrays-objects` | `pass` | `array`, `expr`, `obj`, `value-types` |
| `arrow-functions` | `pass` | `expr`, `fn` |
| `async-await` | `partial` | `async` |
| `atcoder` | `pass` | `control`, `fn`, `io`, `value-types` |
| `basics-equality` | `pass` | `expr` |
| `basics-hello` | `pass` | `io`, `stmt`, `value-types` |
| `basics-oom` | `pass` | `misc` |
| `basics-syntax` | `partial` | `misc` |
| `basics-typeof` | `pass` | `expr` |
| `basics-types` | `pass` | `ts` |
| `basics-utf8` | `pass` | `io`, `value-types` |
| `builtins-and-io` | `pass` | `builtin`, `string` |
| `classes` | `pass` | `class` |
| `classes-and-inheritance` | `pass` | `class`, `expr` |
| `control-flow-and-exceptions` | `pass` | `control`, `stmt` |
| `core-expressions` | `pass` | `expr`, `value-types` |
| `core-semantics` | `pass` | `array`, `builtin`, `class`, `control`, `expr`, `fn`, `misc`, `obj`, `string`, `value-types` |
| `core-statements` | `pass` | `control`, `fn`, `stmt` |
| `functions` | `pass` | `control`, `expr`, `stmt` |
| `hir-support` | `partial` | `expr`, `stmt`, `value-types` |
| `html-comments` | `pass` | `misc` |
| `linker` | `pass` | `linker` |
| `module-system` | `partial` | `module` |
| `modules-and-typed-optimizations` | `pass` | `misc`, `module` |
| `node-apis` | `pass` | `io` |
| `object-semantics-kernel` | `pass` | `builtin`, `obj-kernel`, `obj-semantics` |
| `parser-errors` | `pass` | `ts` |
| `primitives-control-flow` | `pass` | `control`, `fn`, `stmt`, `value-types` |
| `rest-parameters` | `pass` | `fn` |
| `spread-args` | `pass` | `array`, `expr`, `fn` |
| `stmt` | `pass` | `stmt` |
| `test-infrastructure` | `pass` | `test-infra` |
| `this-binding` | `pass` | `fn` |
| `typescript-directives` | `partial` | `ts` |

## Update rule

When adding or changing fixture behavior, update the YAML catalog first, then refresh this document or generated reports.
