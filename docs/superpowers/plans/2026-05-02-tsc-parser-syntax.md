# Meta: TypeScript Compiler Parser Syntax Coverage — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Systematically eliminate all 1,172 `parser-syntax` diagnostic failures in the tsc reference suite, organized into implementation waves by syntactic domain.

**Architecture:** Extend the existing recursive-descent parser (`crates/frontend/src/parser/`) with new parse rules and TypeScript erasure paths. Each wave targets a specific category of child issues, from simple property-name parsing through complex TypeScript declarations.

**Tech Stack:** Rust, recursive-descent parser in `crates/frontend/`, validated via `mise run reference-coverage -- tsc --limit N --detail`.

---

## Current State

| Item | Value |
|------|-------|
| **Parser architecture** | Recursive descent, 11 files in `crates/frontend/src/parser/` |
| **Entry point** | `Parser::parse_program()` in `statements_core.rs:24` |
| **Diagnostic classification** | `DiagCode::UnsupportedTypeScriptSyntax` → `"parser-syntax"` via text heuristics |
| **TS erasure entry** | `consume_erasable_typescript_declaration()` in `statements_ts.rs:10` |
| **tsc test source** | `reference/typescript/tests/cases/compiler/` — 6,419 `.ts` files |
| **Child issues** | 1,172 `spike` issues, `class: blocked`, `depends_on: [5000]` |

## Category Estimates

| Category | Est. Count | Complexity | Wave |
|----------|-----------|-----------|------|
| Reserved words as property names | ~2 | Low | 1 |
| `for...in` with type annotations | ~1 | Low | 1 |
| Index signatures | ~6 | Low | 1 |
| `this`/`super` edge cases | ~8 | Low | 1 |
| `enum` declarations | ~15 | Medium | 2 |
| Function overloads | ~2 | Medium | 2 |
| Interface/inheritance | ~10 | Low-Medium | 2 |
| Ambient declarations | ~30 | Medium | 2 |
| Class syntax variants | ~50 | Medium | 3 |
| Module/namespace | ~80 | Medium | 3 |
| Expression constructs (async/gen/spread/destructure) | ~200 | Medium | 3 |
| Type erasure (generics, mapped, conditional) | ~500 | High | 4 |

## Implementation Waves

### Wave 0: Categorization

Categorize all 1,172 child issues into a priority-sorted implementation queue.

- [ ] Run a script to extract all issue titles → group by topic prefix → produce `docs/superpowers/plans/2026-05-02-tsc-parser-syntax-categories.md`
- [ ] Commit categorization index

### Wave 1: Simple Statement/Expression Fixes

#### Task 1.1: Reserved words as property names (2 issues)

**Files:**
- Modify: `crates/frontend/src/parser/expressions_main.rs` (object literal parsing)
- Test: `crates/frontend/src/parser/tests.rs`

- [ ] **Step 1: Write failing test**

```rust
#[test]
fn test_reserved_words_as_property_names() {
    let stmts = parse_program("var obj = { if: 0, break: 3, function: 4 }").unwrap();
    // Verify object has properties "if", "break", "function"
}
```

- [ ] **Step 2: Implement keyword-to-string mapping** — add `fn keyword_to_property_name(token: &Token) -> Option<&'static str>` that maps keyword tokens to their string values (if, else, break, case, catch, class, const, continue, debugger, default, delete, do, export, extends, finally, for, function, import, in, instanceof, let, new, return, super, switch, this, throw, try, typeof, var, void, while, with, yield, enum, interface, type, declare, module, namespace, async, await, of, satisfies, as, readonly, static, public, private, protected, abstract, implements).

- [ ] **Step 3: Update object literal property name parsing** — where property names are parsed in expression `primary()`, accept keywords via `keyword_to_property_name()` alongside `Token::Ident`.

- [ ] **Step 4: Run tests** — `cargo test -p ts2wasm-frontend`

- [ ] **Step 5: Validate with tsc reference**

```bash
mise run reference-coverage -- tsc --path-filter "reservedWords" --detail
```

- [ ] **Step 6: Commit**

#### Task 1.2: `for...in` with type annotations (~1 issue)

**Files:**
- Modify: `crates/frontend/src/parser/statements_general.rs`

- [ ] **Step 1-6:** Standard TDD cycle: write test → fail → implement type annotation skip in `for_statement()` → pass → validate → commit

#### Task 1.3: Index signatures (~6 issues)

**Files:**
- Modify: `crates/frontend/src/parser/statements_general.rs` (class body: skip `[key: Type]: ReturnType`)
- Modify: `crates/frontend/src/parser/tokens.rs` (add `skip_balanced_bracket_block` helper)

- [x] **Implemented:** Skip `[key: Type]: ReturnType;` in class body by detecting `[` before `expect_ident()`

#### Task 1.4: `this`/`super` edge cases (~8 issues)

**Files:**
- Modify: `crates/frontend/src/parser/expressions_main.rs`

- [ ] **Step 1-6:** Handle `this` and `super` in context-specific positions

### Wave 2: TS Declaration Erasure

#### Task 2.1: Enum erasure (~15 issues)

**Files:** `crates/frontend/src/parser/statements_ts.rs`

Change enum from error to skip: consume `enum` keyword + name + balanced brace block.

#### Task 2.2: Function overloads (~2 issues)

**Files:** `crates/frontend/src/parser/statements_general.rs`

Accept `function foo(...): Type;` where the body is `;` instead of `{...}`.

#### Task 2.3: Interface `extends`/`implements` (~10 issues)

**Files:** `crates/frontend/src/parser/statements_ts.rs`

Skip type references after `extends`/`implements` in interface declarations.

### Wave 3: Module, Class, Expression Extensions

#### Task 3.1: Ambient module/namespace (~30 issues)

**Files:** `crates/frontend/src/parser/statements_ts.rs`

Change `UnsupportedModule` to skip for ambient-only declarations.

#### Task 3.2: Class syntax variants (~50 issues)

**Files:** `crates/frontend/src/parser/statements_general.rs`, `crates/frontend/src/ast.rs`

Handle class expression, static blocks, private elements edge cases.

#### Task 3.3: Expression constructs (~200 issues)

**Files:** `crates/frontend/src/parser/expressions_main.rs`

Handle async, generator, spread, destructuring patterns.

### Wave 4: Complex Type Erasure

**Files:** `crates/frontend/src/parser/statements_ts.rs`

Handle mapped types, conditional types, template literal types, `as const`.

## Validation

```bash
# After each task:
cargo fmt --all --check
cargo nextest run

# After each wave:
mise run reference-coverage -- tsc --limit <N> --detail

# Full gate:
mise run gate-fast
```

## Acceptance Criteria

- [ ] All 1,172 child issues are closed or superseded
- [ ] `parser-syntax` count in `mise run reference-coverage -- tsc --limit 100` trends toward zero
- [ ] Downstream meta-issues (5001, 5003, 5005) show unblocked progress
