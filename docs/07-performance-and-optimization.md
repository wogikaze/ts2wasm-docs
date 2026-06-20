# Performance and Optimization

## Performance concerns

- Reference coverage can run tens of thousands of cases, so compiler server throughput matters.
- WAT generation and conversion should avoid unnecessary subprocesses; current output uses Rust `wat` parsing.
- Runtime helpers should be linked by dependency closure rather than all-included templates.
- Optimization passes must preserve diagnostics and source spans where user-facing behavior depends on them.

## Compiler server

`ts2wasm server` supports batch JSONL requests. Batch execution has:

- configurable worker cap via `TS2WASM_SERVER_MAX_WORKERS`,
- worker stack size via `TS2WASM_SERVER_WORKER_STACK_BYTES`,
- timeout classification rather than process-wide crash,
- per-item panic aggregation.

## Optimization levels

`dump --optimize -O <0..3>` exercises HIR optimization display. Optimization should remain phase-local and must not bypass validation.

## Runtime costs

Runtime ABI currently uses tagged `i32` values and linear-memory heap layouts. Heap allocation, GC roots, and runtime helper deps affect performance. Runtime changes should be measured with focused fixtures and, when relevant, `python scripts/manager.py benchmark-tracker` or dedicated cost diagnostics.

## Rules

- Do not optimize by weakening semantics silently.
- Do not introduce a runtime helper dependency without declaring it in the catalog.
- Do not make reference coverage faster by discarding unsupported/fail evidence.
