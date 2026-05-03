# Static ES Module Export Forms Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove `issue-055` unsupported diagnostics for entry-module export forms, enabling `export const`, `export default`, `export { x, y }`, and default/namespace imports to produce WASM.

**Architecture:** The compiler's `lower_static_named_import_bindings_for_build` rewrites `ImportNamed` → `Let` statements before the built-in resolver catches them with issue-055. We extend this rewrite to also handle entry-module export declarations (ExportDecl, ExportDefault, ExportNamed), then populate the entry module's exports (id=0) in `populate_static_module_exports_for_build`. Non-entry module exports are handled by extending `collect_literal_named_exports`.

**Tech Stack:** Rust, ts2wasm compiler (`crates/compiler/src/lib.rs`), lowered IR (`crates/ir/src/lowered/`), built-in resolver (`crates/ir/src/builtin_resolver.rs`)

---

## Task 1: Entry-module `export const` rewrite (ExportDecl with Let)

**Files:**
- Modify: `crates/compiler/src/lib.rs:204-271` (lower_static_named_import_bindings_for_build)
- Modify: `crates/compiler/src/lib.rs:187-203` (StaticModuleBindingLowering struct)
- Modify: `crates/compiler/src/lib.rs:135-184` (populate_static_module_exports_for_build)
- Test: `crates/cli/tests/m9_modules.rs`
- Fixtures: `fixtures/module-system/`

- [ ] **Step 1: Extend StaticModuleBindingLowering with module_exports field**

Add `module_exports: Vec<ModuleExport>` alongside `named_imports`. `ModuleExport` records `(name: String, is_default: bool)` pairs from the entry module.

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
struct ModuleExport {
    name: String,
    is_default: bool,
}

// In StaticModuleBindingLowering:
struct StaticModuleBindingLowering {
    rewritten_program: Vec<Stmt>,
    named_imports: Vec<StaticNamedImportBinding>,
    module_exports: Vec<ModuleExport>,
}
```

- [ ] **Step 2: Handle ExportDecl in lower_static_named_import_bindings_for_build**

Add a match arm for `Stmt::ExportDecl` before the `other` catch-all. Extract the declaration (Let/Function/ClassDecl), push it as a regular statement, and record the export name.

```rust
Stmt::ExportDecl {
    declaration,
    specifier,
    ..
} => {
    let name = specifier.exported.clone();
    rewritten.push(*declaration.clone());
    module_exports.push(ModuleExport {
        name: name.clone(),
        is_default: false,
    });
    if lowers_to_top_level_statement(&declaration) {
        lowered_statement_index += 1;
    }
}
```

Note: For `export const x = <expr>`, the declaration is `Stmt::Let { name: "x", expr, .. }`. For `export function f() {}`, it's `Stmt::Function { name: "f", .. }`. For `export class C {}`, it's `Stmt::ClassDecl { name: "C", .. }`.

Return `module_exports` in the `StaticModuleBindingLowering` result.

- [ ] **Step 3: Populate entry module exports in populate_static_module_exports_for_build**

Currently this function skips module_id=0. Change it to also process the entry module when it has exports. For the entry module, use the recorded `module_exports` list instead of calling `collect_literal_named_exports`.

```rust
fn populate_static_module_exports_for_build(
    mut lowered: lowered::LoweredProgram,
    module_graph: &ModuleGraph,
    module_exports: &[ModuleExport],
) -> Result<lowered::LoweredProgram, Diagnostic> {
    // Handle entry module exports
    if !module_exports.is_empty() {
        let entry = module_graph.entry();
        let statements = module_exports
            .iter()
            .map(|export| {
                // Find the corresponding lowered statement via the name
                let lowered_stmt = lowered.top_level_statements
                    .iter()
                    .find(|stmt| matches!(stmt, lowered::LoweredStmt::Let(name, _) if name == &export.name))
                    .ok_or_else(|| Diagnostic { ... })?;
                // Return a reference to the expression
                match lowered_stmt {
                    lowered::LoweredStmt::Let(_, expr) => Ok(lowered::LoweredStmt::Export {
                        name: export.name.clone(),
                        expr: expr.clone(),
                    }),
                    _ => Err(...),
                }
            })
            .collect::<Result<Vec<_>, Diagnostic>>()?;

        lowered.modules.push(lowered::ModuleInfo {
            id: 0,
            specifier: "<entry>".to_owned(),
            statements,
            locals_count: 0,
        });
    }

    // Existing logic for non-entry modules (unchanged)
    for step in module_graph.dependency_first_initialization_steps() {
        if step.module_id() == 0 {
            continue;
        }
        // ... existing code ...
    }

    Ok(lowered)
}
```

- [ ] **Step 4: Update build_file_with_host_deny call sites**

Update `build_file_with_host_deny` (line 45) and `build_file_with_options` (line 37) to pass `module_exports` through the pipeline:

```rust
let lowered = populate_static_module_exports_for_build(lowered, &module_graph, &static_module_binding.module_exports)?;
```

- [ ] **Step 5: Add fixture for entry-module export const**

Create `fixtures/module-system/static-export-const-entry.ts`:

```typescript
export const x = 1;
console.log(x);
```

Expected: builds to WASM, prints `1` when run.

- [ ] **Step 6: Add build smoke test for entry-module export const**

In `crates/cli/tests/m9_modules.rs`:

```rust
#[test]
fn static_export_const_entry_build_smoke() {
    assert_fixture_build_smoke("module-system/static-export-const-entry.ts");
}
```

- [ ] **Step 7: Run tests to verify**

```bash
cargo fmt --all --check
cargo nextest run -p ts2wasm-compiler
cargo nextest run -p ts2wasm-cli module
```

Expected: All existing tests pass plus new test passes.

- [ ] **Step 8: Update the `static_declaration_export_reports_issue_055` test**

This test currently expects `static-declaration-export-unsupported.ts` to fail with issue-055. If the fixture is `export const x = 1` (ExportDecl with Let), change it to expect build smoke. If it uses a non-const declaration (e.g., `export function`), keep the test but narrow the assertion.

Check the fixture content:

```bash
cat fixtures/module-system/static-declaration-export-unsupported.ts
```

If it is `export const x = 1`, delete or update the test.

## Task 2: `export default <literal>` in entry module

**Files:**
- Modify: `crates/compiler/src/lib.rs`
- Modify: `crates/cli/tests/m9_modules.rs`
- Create: `fixtures/module-system/`

- [ ] **Step 1: Handle ExportDefault in lower_static_named_import_bindings_for_build**

Add match arm for `Stmt::ExportDefault`:

```rust
Stmt::ExportDefault {
    expr,
    span,
    ..
} => {
    // export default <expr> → const __ts2wasm_default = <expr>
    rewritten.push(Stmt::Let {
        name: "__ts2wasm_default".to_owned(),
        expr: *expr.clone(),
        span: *span,
    });
    module_exports.push(ModuleExport {
        name: "default".to_owned(),
        is_default: true,
    });
    lowered_statement_index += 1;
}
```

- [ ] **Step 2: Add fixture for export default literal**

Create `fixtures/module-system/static-export-default-entry.ts`:

```typescript
export default 42;
console.log(42);
```

Expected: builds to WASM.

- [ ] **Step 3: Add build smoke test**

```rust
#[test]
fn static_export_default_entry_build_smoke() {
    assert_fixture_build_smoke("module-system/static-export-default-entry.ts");
}
```

- [ ] **Step 4: Update the `static_default_export_reports_issue_055` test**

Check fixture content. If it's a plain `export default <literal>`, update to expect build smoke. Otherwise narrow the assertion.

- [ ] **Step 5: Run tests**

```bash
cargo fmt --all --check
cargo nextest run -p ts2wasm-compiler
cargo nextest run -p ts2wasm-cli module
```

## Task 3: `export { x, y }` named export list in entry module

**Files:**
- Modify: `crates/compiler/src/lib.rs`
- Modify: `crates/cli/tests/m9_modules.rs`
- Create: `fixtures/module-system/`

- [ ] **Step 1: Handle ExportNamed in lower_static_named_import_bindings_for_build**

Add match arm for `Stmt::ExportNamed`:

```rust
Stmt::ExportNamed {
    specifiers,
    ..
} => {
    // export { x, y as z } — no new statement, just record exports
    for specifier in specifiers {
        module_exports.push(ModuleExport {
            name: specifier.exported.clone(),
            is_default: false,
        });
    }
    // No statement added — exports reference existing local bindings
}
```

- [ ] **Step 2: Handle ExportNamed in export population**

In `populate_static_module_exports_for_build`, when building entry module exports, look up the export name as a local binding in `lowered.top_level_statements`. For `export { x }`, find `Let("x", expr)` and create `Export { name: "x", expr }`.

This requires matching export names to lowered local bindings. For `export { x, y as z }`, `x` maps to `Let("x", expr)` and `z` maps to `Let("y", expr)` — actually `export { x as z }` means "export binding `x` under name `z`". So we need `specifier.local` (original) and `specifier.exported` (exported name):

```rust
// specifier has: local (local binding name), exported (exported name), ...
```

Check the `Specifier` struct in the frontend AST for field names.

- [ ] **Step 3: Add fixture for named export list**

Create `fixtures/module-system/static-export-named-list-entry.ts`:

```typescript
const a = 1;
const b = 2;
export { a, b as c };
console.log(a, b);
```

Expected: builds to WASM.

- [ ] **Step 4: Add build smoke test**

```rust
#[test]
fn static_export_named_list_entry_build_smoke() {
    assert_fixture_build_smoke("module-system/static-export-named-list-entry.ts");
}
```

- [ ] **Step 5: Update `static_named_export_reports_issue_055` test**

Check fixture `static-named-export-unsupported.ts`. If it's `export { value }` (ExportNamed), update to build smoke.

- [ ] **Step 6: Run tests**

```bash
cargo fmt --all --check
cargo nextest run -p ts2wasm-compiler
cargo nextest run -p ts2wasm-cli module
```

## Task 4: `import x from "./mod"` default import

**Files:**
- Modify: `crates/compiler/src/lib.rs`
- Modify: `crates/compiler/src/module_graph.rs` (if needed)
- Modify: `crates/cli/tests/m9_modules.rs`
- Create: `fixtures/module-system/`

- [ ] **Step 1: Check if module_graph already handles ImportDefault**

Run a quick check: does `collect_static_module_specifiers` in `module_graph.rs` already include `Stmt::ImportDefault` and `Stmt::ImportDefaultNamed`? If yes (line 212-213 shows it does), module graph resolution already works.

- [ ] **Step 2: Handle ImportDefault in lower_static_named_import_bindings_for_build**

Add match arm:

```rust
Stmt::ImportDefault {
    binding,
    source,
    ..
} => {
    let dependency = module_graph.entry().dependencies()
        .iter()
        .find(|d| d.specifier() == source.value)
        .ok_or_else(|| Diagnostic {
            code: DiagCode::InvariantViolation,
            message: format!("module graph has no dependency for default import `{}`", source.value),
            span: Some(source.span),
        })?;
    // Look up "default" export from the module
    let exports = collect_literal_named_exports(dependency.resolved_path())?;
    let expr = exports.get("default").cloned()
        .ok_or_else(|| Diagnostic {
            code: DiagCode::UnsupportedSyntax,
            message: format!("issue-5005: module `{}` does not have a default export", source.value),
            span: Some(source.span),
        })?;
    rewritten.push(Stmt::Let {
        name: binding.clone(),
        expr,
        span: *span,
    });
    named_imports.push(StaticNamedImportBinding {
        source_specifier: source.value.clone(),
        source_module_id: dependency.resolved_module_id(),
        source_path: dependency.resolved_path().to_path_buf(),
        imported_name: "default".to_owned(),
        local_name: binding.clone(),
        lowered_statement_index,
        initializer: expr,
    });
    lowered_statement_index += 1;
}
```

- [ ] **Step 3: Add fixture for default import**

Create `fixtures/module-system/static-default-import-entry.ts` and source module:

```typescript
// static-default-import-source.ts
export default 42;
```

```typescript
// static-default-import-entry.ts
import value from "./static-default-import-source";
console.log(value);
```

Expected: builds to WASM, prints `42`.

- [ ] **Step 4: Add build smoke test**

```rust
#[test]
fn static_default_import_entry_build_smoke() {
    assert_fixture_build_smoke("module-system/static-default-import-entry.ts");
}
```

- [ ] **Step 5: Run tests**

```bash
cargo fmt --all --check
cargo nextest run -p ts2wasm-compiler
cargo nextest run -p ts2wasm-cli module
```

## Task 5: Remove or narrow issue-055 catch-all in builtin_resolver

**Files:**
- Modify: `crates/ir/src/builtin_resolver.rs:893-908`

- [ ] **Step 1: Audit which forms are now handled by the compiler rewrite**

Check each form:
- ImportNamed → handled by rewrite (already before issue-5005)
- ExportDecl → handled by Task 1
- ExportDefault → handled by Task 2
- ExportNamed → handled by Task 3
- ImportDefault → handled by Task 4
- Side-effect import → NOT yet handled (keep issue-055)
- ImportDefaultNamed → NOT yet handled (keep or partial)
- ImportNamespace → NOT yet handled (keep)
- ImportDefaultNamespace → NOT yet handled (keep)
- ExportNamedFrom → NOT yet handled (keep)
- ExportAllFrom → NOT yet handled (keep)
- ExportNamespaceFrom → NOT yet handled (keep)

Remove the handled forms from the match, or replace with a more targeted check.

- [ ] **Step 2: Update the catch-all**

```rust
// Replace the broad match with targeted unsupported diagnostics
Stmt::ImportSideEffect { span, .. } => Err(unsupported_module_decl(*span, "side-effect import")),
Stmt::ImportNamespace { span, .. }
| Stmt::ImportDefaultNamespace { span, .. } => Err(unsupported_module_decl(*span, "namespace import")),
Stmt::ExportNamedFrom { span, .. } => Err(unsupported_module_decl(*span, "named re-export from module")),
Stmt::ExportAllFrom { span, .. } => Err(unsupported_module_decl(*span, "star re-export")),
Stmt::ExportNamespaceFrom { span, .. } => Err(unsupported_module_decl(*span, "namespace re-export")),
// ImportDefaultNamed: only the default part is handled, named part still needs work
Stmt::ImportDefaultNamed { span, .. } => Err(unsupported_module_decl(*span, "combined default+named import")),
```

- [ ] **Step 3: Run all tests**

```bash
cargo fmt --all --check
cargo nextest run
```

All existing tests must pass, with issue-055 diagnostic tests for now-handled forms either removed or updated.

## Task 6: Node/iwasm differential test for entry export forms

**Files:**
- Modify: `crates/cli/tests/m2_node_diff.rs`

- [ ] **Step 1: Add differential test for `export const x = 1` entry module**

Add a test case that:
1. Builds the fixture with ts2wasm
2. Runs it under iwasm
3. Compares output to Node.js

Use the existing pattern from `crates/cli/tests/m2_node_diff.rs`.

```rust
#[test]
fn static_export_const_iwasm_matches_node() {
    let entry = compile_fixture("module-system/static-export-const-entry.ts");
    run_iwasm_and_compare_with_node(&entry, "module-system/static-export-const-entry.ts");
}
```

- [ ] **Step 2: Add differential tests for default export and named export list**

Follow the same pattern for `static-export-default-entry.ts` and `static-export-named-list-entry.ts`.

- [ ] **Step 3: Run differential tests**

```bash
cargo nextest run -p ts2wasm-cli static_export_const_iwasm_matches_node
```
