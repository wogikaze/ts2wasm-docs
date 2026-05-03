# Frontend Parser: Grammar Family Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement remaining TypeScript parser syntax by grammar family (top-down), eliminating the remaining ~625 child issues depending on meta-issue #5000.

**Architecture:** Extend the recursive-descent parser in `crates/frontend/src/parser/` with new parse rules and TypeScript erasure paths. Each task targets a specific grammar family from `docs/language-reference/frontend-parser-wave.md` Source Order, with spec refs, valid/invalid snippets, and reference paths.

**Tech Stack:** Rust, recursive-descent parser in `crates/frontend/`, validated via `cargo nextest run -p ts2wasm-frontend` and `mise run reference-coverage -- tsc --path-filter <path> --detail`.

---

## Source Order (from frontend-parser-wave.md)

| Order | Family | Primary source | Est. remaining |
|-------|--------|---------------|----------------|
| 13 | TypeScript erased syntax | TypeScript parser | ~300 issues |
| 14 | TypeScript value syntax | TypeScript parser | ~200 issues |
| 9 | Functions, arrows, generators, async | ECMA-262 + TS | ~60 issues |
| 10 | Classes and private elements | ECMA-262 + TS | ~65 issues |

## Current state per 200 tsc tests

| Label | Count | Map to order |
|-------|-------|-------------|
| build_pass | 75 | — |
| unknown-unsupported | 25 | Orders 13/14 (generic catch-all) |
| type-alias | 19 | Order 13 |
| import-export | 17 | Order 14 |
| module-system-amd | 9 | Order 14 |
| duplicate-function | 7 | Order 9 |
| arguments-object | 7 | Order 9 (JS runtime) |
| ambient-declaration | 7 | Order 13 |
| declaration-emit | 6 | Order 14 |
| class-accessor | 6 | Order 10 |
| scope-analysis | 4 | Order 9 |
| class | 2 | Order 10 |
| type-assertion | 1 | Order 13 |

---

### Task 1: Type alias erasure in expression positions (Order 13)

**Spec ref:** TypeScript parser `parseTypeAliasDeclaration`
**Files:**
- Modify: `crates/frontend/src/parser/statements_ts.rs` (type alias erasure)
- Modify: `crates/frontend/src/parser/expressions_main.rs` (expression-level type alias usage)
- Test: `crates/frontend/src/parser/tests.rs`

**Problem:** The parser can erase `type Foo = ...` at statement level, but fails when type aliases appear in expression context (e.g., `let x: Foo = bar` where `Foo` is a type alias and produces `UnsupportedSyntax`).

**What to implement:**
In `classify_feature` / `feature-labels.sh`, `type-alias` label matches diagnostic messages containing "type-alias" or when the expression parser encounters an identifier that is a known type alias. The fix: ensure type alias references in expression positions are erased/skipped properly.

- [ ] **Step 1: Identify failing patterns**
    Run: `mise run reference-triage -- tsc reference/typescript/tests/cases/compiler/aliasUsageInVarAssignment.ts`

- [ ] **Step 2: Add failing unit tests** for type alias usage in variable assignments, function expressions, array literals, object literals

- [ ] **Step 3: Implement type alias reference erasure in expression parser**

- [ ] **Step 4: Run tests**
    Run: `cargo nextest run -p ts2wasm-frontend`

- [ ] **Step 5: Validate with tsc reference**
    Run: `mise run reference-coverage -- tsc --path-filter "aliasUsage" --detail`

- [ ] **Step 6: Commit**

### Task 2: Ambient declaration erasure — remaining forms (Order 13)

**Spec ref:** TypeScript parser `parseDeclaration` → ambient declarations
**Files:**
- Modify: `crates/frontend/src/parser/statements_ts.rs` (`consume_erasable_typescript_declaration`)
- Test: `crates/frontend/src/parser/tests.rs`

**Problem:** `declare class`, `declare function`, `declare const` are already handled. But some ambient declaration forms like `declare global { }`, `declare abstract class`, and ambient declarations inside class bodies may not be handled.

**What to implement:**
- Add handler for `declare global { ... }` — skip balanced brace block, return Ok(true)
- Add handler for `declare abstract class` — same as `declare class` but skip `abstract`
- Add handler for `declare enum` with modifiers

- [ ] **Step 1: Identify failing patterns**
    Run: `mise run reference-triage -- tsc reference/typescript/tests/cases/compiler/ambientDeclarations.ts`

- [ ] **Step 2: Add unit tests** for `declare global`, `declare abstract class`, `declare enum`

- [ ] **Step 3: Implement** missing ambient declaration handlers

- [ ] **Step 4-6: Run tests, validate, commit**

### Task 3: Type assertion erasure — `as` and angle-bracket (Order 13)

**Spec ref:** TypeScript parser `parseTypeAssertion`, `parseAsExpression`
**Files:**
- Modify: `crates/frontend/src/parser/expressions_main.rs`
- Test: `crates/frontend/src/parser/tests.rs`

**Problem:** `as` assertions and angle-bracket type assertions in certain positions may not be fully erased.

**What to implement:**
Ensure all `as Type` and `<Type>expr` forms are handled in expression parser. The `satisfies` keyword already has erasure. Extend the same pattern to angle-bracket assertions.

- [ ] **Step 1-6: TDD cycle for type assertion erasure**

### Task 4: Ambient module/namespace — remaining edge cases (Order 14)

**Spec ref:** TypeScript parser `parseModuleDeclaration`
**Files:**
- Modify: `crates/frontend/src/parser/statements_ts.rs`
- Test: `crates/frontend/src/parser/tests.rs`

**Problem:** Most `declare module`/`declare namespace` are now erased, but some edge cases remain:
- `export module` at statement level (not inside declare)
- Module merging with class/function declarations
- Nested modules

- [ ] **Step 1-6: TDD cycle for remaining module/namespace edge cases**

### Task 5: Getter/setter accessor syntax in classes (Order 10)

**Spec ref:** ECMA-262 Class Definitions + TypeScript accessor syntax
**Files:**
- Modify: `crates/frontend/src/parser/statements_general.rs` (class body parsing)
- Test: `crates/frontend/src/parser/tests.rs`

**Problem:** `get`/`set` accessors in class bodies with TypeScript type annotations on the setter parameter may fail.

`class-accessor` label matches files like:
- `accessorWithoutBody1.ts`: `var v = { get foo() }` (object literal getter without body)
- `accessorDeclarationOrder.ts`: has `#name` private fields and get/set with type annotations

**What to implement:**
- Object literal getter/setter without body: handle `{ get foo() }` as valid syntax
- Class get/set with parameter type annotations: ensure type annotation skip works correctly
- Private field `#name` accessor integration

- [ ] **Step 1-6: TDD cycle for getter/setter accessor syntax**

### Task 6: Function overload edge cases (Order 9)

**Spec ref:** ECMA-262 Functions + TypeScript overload signatures
**Files:**
- Modify: `crates/frontend/src/parser/statements_general.rs` (function parsing)
- Test: `crates/frontend/src/parser/tests.rs`

**Problem:** `duplicate-function` classifier matches test cases where TypeScript has function overloads that the parser treats as duplicate definitions.

**What to implement:**
- Ensure all overload signature patterns (with `;` termination, with type parameters, with differing parameter lists) are handled
- Ensure that when multiple functions with the same name appear, only the last (implementation) body is kept

- [ ] **Step 1-6: TDD cycle for function overload patterns**

### Task 7: `arguments` object handling (Order 9, JS runtime)

**Spec ref:** ECMA-262 Function Definitions — `arguments` object
**Files:**
- Modify: `crates/ir/src/lowered/resolver_expr.rs` or runtime code
- Test: relevant runtime tests

**Problem:** `arguments` keyword used in various positions (property name, iterator, etc.) may be unsupported.

**Note:** This is a **runtime** issue (meta-5004), NOT a parser issue. Many of these should have been reclassified by Phase A.5. If they remain, they should be moved.

- [ ] **Step 1: Verify diagnostic code**
    Run: `mise run reference-triage -- tsc reference/typescript/tests/cases/compiler/arguments.ts`
    If `DiagCode::UnsupportedRuntimeSubset`, reclassify to meta-5004.

### Task 8: Declaration emit edge cases (Order 14)

**Spec ref:** TypeScript declaration emit
**Files:**
- Classify: each case needs individual triage

**Problem:** `declaration-emit` label matches files related to TypeScript's `--declaration` output. These are NOT parser issues — they're about `.d.ts` generation which ts2wasm doesn't implement.

**Note:** These should be reclassified or closed as out of scope.

- [ ] **Step 1: Verify each is truly out of scope → reclassify to `frontend/syntax` depends_on issue-346**

## Validation

```bash
# After each task:
cargo fmt --all --check
cargo nextest run -p ts2wasm-frontend

# After completing a Source Order family:
mise run reference-coverage -- tsc --limit 200 --detail

# Full gate:
mise run gate-fast
```

## Acceptance Criteria

- [ ] All grammar families from Orders 9-14 have explicit parser support or erasure
- [ ] Parser tests pass (130+ tests)
- [ ] build_pass per 200 tsc tests increases from 75 toward 125+
- [ ] Remaining child issues depending on #5000 trend toward zero
