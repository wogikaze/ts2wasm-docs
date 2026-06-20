# Related Projects

This project sits between source-to-source TypeScript tooling, embedded JS engines, and wasm component tooling.

| Project family | Difference from ts2wasm |
|---|---|
| TypeScript / typescript-go | Type checking and transpilation are references/oracles; ts2wasm emits wasm with a runtime subset. |
| AssemblyScript | AssemblyScript is a TS-like language with different runtime assumptions; ts2wasm targets existing JS/TS semantics where possible. |
| QuickJS / QuickJS-ng | Full JS engine approach; ts2wasm avoids bundling a full engine and instead compiles/lower supported semantics. |
| Javy | JS-to-WASM using a JS engine runtime; ts2wasm focuses on explicit runtime helper/link plan and capability manifest. |
| wasm-bindgen | Rust/wasm interop tooling; ts2wasm compiles JS/TS source. |
| wasm-tools / wasmtime / WAMR | Execution/validation/reference tooling used around wasm output, not compiler replacement. |
| JCO/component model tools | Future target inspiration; current component target is reserved but unimplemented. |

Use related projects for comparison and validation, not as hidden dependencies in the compiler pipeline.
