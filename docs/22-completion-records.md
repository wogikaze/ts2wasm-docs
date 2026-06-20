# Completion Records

Completion records model ECMAScript statement completion: `[[Type]]`, `[[Value]]`, and `[[Target]]`. They are required for correct loops, labels, `switch`, `try/catch/finally`, functions, generators, async paths, and eval completion values.

## Model

`crates/ir/src/semantic.rs` defines:

| Item | Contract |
|---|---|
| `CompletionStatus::Normal` | statement completed normally |
| `CompletionStatus::Return` | function return completion |
| `CompletionStatus::Throw` | thrown value completion |
| `CompletionStatus::Break` | abrupt break with target |
| `CompletionStatus::Continue` | abrupt continue with target |
| `JSVAL_EMPTY` | empty completion value sentinel, distinct from `undefined` |
| `CompletionRecord` | status, value, target tuple |

Status numeric values are part of the internal contract and are covered by unit tests. Do not renumber them without updating tests and all lowering/emission consumers.

## Ownership

HIR owns semantic completion shape. Lowering may choose an efficient representation, but it must preserve status, value, and target until a construct consumes the completion.

Do not model completion-sensitive behavior as a plain expression value. This is especially important for `finally`, labeled break/continue, eval, and generator completion.

## UpdateEmpty

`UpdateEmpty(completion, defaultValue)` replaces `JSVAL_EMPTY` with `defaultValue` while preserving status and target. It does not replace `undefined`.

Use it where ECMAScript says an empty normal or abrupt completion inherits a previous/default value.

## Lowering Rules

- Every statement lowering logically produces or propagates a completion.
- Function boundaries consume `Return` and convert it to the function result.
- `Throw` propagates until caught or reaches the runtime exception boundary.
- `Break` and `Continue` carry explicit target labels.
- `finally` executes after try/catch body and may override the prior completion.
- Direct wasm `return`/`br` is allowed only after proving it cannot skip required `finally` or target handling.

## Labels

Labels must be represented as explicit target ids or equivalent resolved labels. A label-consuming statement may convert a matching `Break` to `Normal`; non-matching abrupt completions propagate.

Invalid top-level `return`, unknown labels, and invalid `continue` targets are diagnostics, not backend traps.

## Eval Completion

Eval expansion uses completion plans so declaration side effects and the last non-empty completion value are preserved exactly once. Completion/declaration plans must remain internally consistent before lowering; inconsistent plans are compiler diagnostics.

## Async And Generators

Async/generator lowering may suspend and resume execution, but it must preserve completion status and value across suspension points. A future representation may specialize this, but must keep the observable completion contract.

## Tests

- `crates/ir/src/semantic.rs` completion unit tests.
- `fixtures/control-flow-and-exceptions`.
- `tests/fixtures/completion_record`.
- `crates/cli/tests/control_flow.rs`.
- Eval completion tests in `crates/compiler/src/tests.rs`.
