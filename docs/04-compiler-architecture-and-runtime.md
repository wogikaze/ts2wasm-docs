# Compiler Architecture and Runtime

## Pipeline

```text
read_source_file
  -> split_file_name_sections (optional multi-section module fixtures)
  -> validate_type_reference_directives
  -> Lexer::tokenize
  -> Parser::parse_program
  -> validate_ast
  -> module_graph::build_module_graph
  -> static_imports::lower_static_named_import_bindings_for_build
  -> stages::name_resolve::resolve_names
  -> stages::builtin_resolve::resolve_builtins
  -> eval_expand::expand_static_eval_fragments
  -> semantic_validate::validate_semantics
  -> legacy LoweredProgram or experimental HIR/MIR
  -> lowered_validate / runtime_gate
  -> runtime link plan / capability manifest
  -> backend::emit_wasm_binary or backend::emit_mir_wasm_binary
  -> write_wasm_bytes_with_abi
```

The canonical implementation anchor is `crates/compiler/src/pipeline.rs`.

`test262` include/frontmatter handling is a reference-suite adapter, not a language pipeline responsibility. Official/reference coverage runners should provide already prepared source when possible. The CLI/dump path still has a compatibility preprocessor for raw test262 files, but docs and architecture diagrams must not describe it as a normal ts2wasm compile phase.

## Crate boundaries

| Layer | Crates | Owns |
|---|---|---|
| Source/diagnostics | `source`, `diagnostic` | spans, source-originating diagnostics, phase tagging |
| Syntax/frontend | `syntax`, `frontend` | tokens, AST, parser, TypeScript erasure/oracle |
| Resolution/semantics | `resolve`, `semantics` | binding/name resolution, builtin domain identity, host API classification |
| IR | `ir` | builtin-resolved AST, HIR, MIR, legacy LoweredProgram, optimizer |
| Runtime/ABI | `runtime-abi`, `runtime-catalog` | value layout, runtime functions, link plans, host imports, capabilities |
| Backend | `backend-core`, `backend-wasm` | wasm binary emission, WAT debug emission, runtime templates, MIR emission subset |
| Orchestration | `compiler`, `cli` | end-to-end build/check/dump/server API |

## Legacy and HIR/MIR paths

- Default build path lowers builtin-resolved statements to legacy `LoweredProgram` and requires native wasm bytes from the backend.
- Strict HIR/MIR path uses `lower_to_hir`, validates HIR, lowers to native MIR, validates MIR, then calls MIR emission.
- Compat fallback tries HIR/MIR first and falls back to legacy when unsupported, but the final build artifact still comes from wasm bytes rather than an external WAT conversion command.
- WAT remains available through dump/debug APIs and HIR/MIR comparison diagnostics; it is not the default build success path.
- Docs and tests should identify which path a behavior uses.

## Spec-Kernel correctness path

A third pipeline path exists for semantics-preserving compilation via `SpecAlgoIR`:

```
resolved AST → semantic-ir lowering → CorrectnessLowering → SpecOp list → spec_emit → wasm binary
```

Key components:
- **`crates/semantic-ir/`**: Semantic IR types (SemExpr, SemStmt, SemBlock) bridging frontend and correctness backend.
- **`crates/spec-kernel/`**: SpecOp enum (~35 ECMAScript internal method symbols), SpecAlgoIR type system (completion model, 40+ algorithm steps), and ordinary algorithm builders (Get, Set, Call, Construct, ToPropertyKey, etc.).
- **`crates/builtin-kernel/`**: Observable ECMAScript builtin algorithms (Array.push, String.replace, Promise.then, etc.) expressed as SpecAlgoPrograms with real SpecOp call sequences (62 modules, 123 tests).
- **`crates/backend-correctness/`**: Mechanical SpecAlgoIR→WasmInstr compiler (`algo_compile.rs`) and module builder (`spec_emit.rs`). No hand-written JS semantics — purely mechanical translation.
- **`crates/runtime-store-wasm/`**: PropertyStore wasm primitives (9 functions: own_property_lookup, own_property_insert, etc.). Storage primitives only — no prototype walk, accessor dispatch, or descriptor validation.
- **`crates/runtime-wasm/`**: Non-observable runtime primitives (math, BigInt, heap, string, typedarray, atomics). ~80 small wasm functions with fixed symbols.

Enable via `--spec-kernel` CLI flag or `TS2WASM_SPEC_KERNEL=1` environment variable for coverage runs.

Design principles:
- **Single source of truth**: ECMAScript semantics exist only in `spec-kernel` SpecAlgoIR. No crate outside `spec-kernel` implements JS semantics.
- **Mechanical compilation**: `backend-correctness` is a pure translator. Every WasmInstr traces back to a SpecAlgoStep.
- **PropertyStore primitives**: Storage layer is strictly primitives (lookup, insert, update, delete, keys). All semantic operations (prototype walk, accessor dispatch, descriptor validation) live in SpecAlgoIR.
- **Builtin observability**: Observable builtins (Array methods, String methods, Promise methods) belong in `builtin-kernel`, NOT in `runtime-wasm`.

## Runtime shape

The runtime is assembled from cataloged `RuntimeFn` specs. The catalog currently defines hundreds of helpers spanning core arithmetic/comparison, arrays, objects, collections, typed arrays, Date, RegExp, strings, JSON, modules, Promise/async, Symbol, console, host eval, and Node/test262 host hooks.

## Negative constraints

- Backend must not add new direct dependencies on frontend AST when IR can express the behavior.
- Runtime helper dependencies must not be hidden in WAT templates; declare them in the runtime catalog.
- Host import addition must flow through `RuntimeLinkPlan` and capability manifest.
- ABI constants must live in `runtime-abi`, not as scattered numeric literals.
