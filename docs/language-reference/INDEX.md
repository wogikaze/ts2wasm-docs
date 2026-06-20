# Language Reference Index

These files map external language/platform features to the current ts2wasm support model. They are navigation and triage docs; executable work lives in `issues/`.

| File | Use |
|---|---|
| `javascript-features.md` | ECMAScript syntax, runtime semantics, builtins, and feature tracking. |
| `typescript-features.md` | TypeScript syntax/type-system boundaries and erasure policy. |
| `typescript-boundary.yaml` | Machine-readable TypeScript boundary categories. |
| `frontend-parser-wave.md` | Parser/frontend work splitting rules and acceptance criteria. |
| `wasm-features.md` | WebAssembly feature/proposal support policy. |
| `wasi-features.md` | WASI Preview feature/capability mapping. |

## Rules

- Do not infer semantic support from parser acceptance.
- Keep broad feature status here; put implementation slices and acceptance commands in `issues/`.
- When code changes support status, update `fixtures/catalog.yaml`, `fixtures/feature-matrix.yaml`, or the relevant generated report first.
- If a table becomes too large to guide work, split by feature family and update this index.
