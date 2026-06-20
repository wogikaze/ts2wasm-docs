# test262 Coverage Roadmap

Created: 2026-06-17.
Source data: `docs/current-state.md` (2026-06-15 audit).

This document defines the attack order for increasing test262 semantic pass rate. The key insight: the current 19% plateau is not stagnation — it marks the transition from "TypeScript syntax to Wasm" to "JavaScript runtime implementation". The remaining failures are not independent small bugs; they are large semantic blocks with deep dependencies.

## Current State Summary

| Metric | Value |
|--------|-------|
| Total tested | ~13,600 |
| Semantic pass | ~2,600 (19%) |
| Mismatch | ~5,200 (code runs but wrong output) |
| Unsupported | ~5,800 (compilation fails) |

## Strategy: Categorical, Not Overall

Stop using overall pass rate as the primary metric. Instead, track by category. The overall number will stay flat until foundational infrastructure lands, then jump in stages.

## Phase 1: Keystone Infrastructure (I-20260615-8WMZG9 + I-20260615-WFE84B)

**Impact: +100-130 tests across 10+ categories**
**Effort: 1-2 weeks**
**ROI: Highest — two issues unblock 10%+ of all tests**

These two issues are the single biggest blockers. The test262 harness `verifyProperty()` and `isConstructor()` helpers are used by EVERY builtin constructor test.

### 1a. Object.getOwnPropertyDescriptor (I-20260615-8WMZG9)

The test262 harness `verifyProperty()` depends on this. Without it, every builtin constructor property descriptor test fails.

**Required RuntimeFn:**
- `Object.getOwnPropertyDescriptor(obj, prop)` → undefined | {value, writable, enumerable, configurable}
- `Object.defineProperty(obj, prop, desc)` → define/modify property
- `Object.prototype.propertyIsEnumerable(prop)`
- `Object.prototype.hasOwnProperty(prop)`

**Categories unblocked:**
- Error: 12 tests
- AggregateError: 8 tests
- Boolean: ~12 tests
- Symbol: many tests
- Number: property descriptor tests
- Reflect: property descriptor tests
- All builtin constructors with `.name`/`.length`/`.prototype` descriptor checks

**Dependency chain:**
```
verifyProperty(obj, name, expected)
  → Object.getOwnPropertyDescriptor(obj, name)  ← CRITICAL
  → Object.defineProperty (for restore step)
  → delete obj[name] (for isConfigurable check)
  → for...in loop (for isEnumerable check)
  → Array.isArray
  → Object.prototype.hasOwnProperty.call
  → Object.prototype.propertyIsEnumerable.call
```

### 1b. Reflect.construct / IsConstructor (I-20260615-WFE84B)

The test262 harness `isConstructor()` helper depends on `Reflect.construct`. Without it, every `isConstructor(Error)`, `isConstructor(Boolean)`, etc. evaluates incorrectly.

**Required implementation:**
- `Reflect.construct(target, argumentsList, newTarget)` — the core function
- Each builtin constructor's `[[Construct]]` internal method
- Each builtin function's (non-constructor) internal method check

**Categories unblocked:**
- Error: 4 tests
- AggregateError: 4 tests
- Boolean: is-a-constructor.js
- Symbol: is-a-constructor.js
- All builtin constructors with isConstructor checks (~20+ categories)

### 1c. Symbol.toStringTag (I-20260615-7VP3TF)

Requires `Object.defineProperty` support. Blocks 3 Error tests.

---

## Phase 2: High-Mismatch WAT Fixes (Quick Wins)

**Impact: +400-500 tests**
**Effort: 2-3 weeks**
**ROI: High — code already runs, just produces wrong output**

These categories have high mismatch counts (code compiles and runs but produces wrong results). They are fixable with WAT-level adjustments, not architectural changes.

### 2a. String remaining bugs (I-20260519-XR7WBX)

- **Status:** doing, 4 specific bugs remaining
- **Mismatch:** 619, Pass: 423 (35%)
- **Remaining:**
  - 2 normalize tests (stringNormalize host import missing)
  - 1 RegExp.match test (outputs "1024")
  - 1 String.fromCharCode test (multi-arg handling)
- **Effort:** Low (4 specific bugs identified)

### 2b. JSON host bridge (I-20260518-5WTEA5)

- **Status:** open, 91 mismatches
- **Pass:** 0 (0%)
- **Root cause:** Host stubs return undefined instead of real parse/stringify results
- **Fix:** Implement actual `host.json.parse` and `host.json.stringify` in the host bridge
- **Effort:** Medium (need real host bridge, not stubs)

### 2c. Number formatting edge cases

- **Mismatch:** 266, Pass: 42 (12%)
- **Root cause:** `toExponential`, `toFixed`, `toPrecision` precision/edge case bugs
- **Effort:** Medium (parity testing against V8/Node output)

### 2d. Math precision/edge cases

- **Mismatch:** 262, Pass: 30 (9%)
- **Root cause:** NaN, -0, Infinity, precision edge cases in Math functions
- **Effort:** Medium (pure WAT fixes, but many edge cases)

### 2e. Function remaining (I-20260518-N39XFG)

- **Mismatch:** 207, Pass: 104 (20%)
- **Already fixed:** call/apply on runtime function value locals
- **Remaining:** bind, toString on non-ident receiver, hasInstance
- **Effort:** Medium (bind requires WAT closure creation)

---

## Phase 3: Promise + Async Foundation

**Impact: +200-300 tests (Promise) + 561 tests (async/await)**
**Effort: 3-4 weeks**
**ROI: High — Promise is the biggest single category by test count**

### 3a. Promise infrastructure (I-20260518-8KG9D3)

- **Mismatch:** 327, Pass: 54 (8%)
- **Already fixed:** Constructor executor calling
- **Remaining:**
  1. Create proper callable resolve/reject closures (capturing the promise base)
  2. Add element-level Promise.resolve wrapping in Promise.all/race/any/allSettled
  3. Install Symbol.species getter on Promise constructor
- **Effort:** Medium (WAT-level closure creation)

### 3b. Microtask queue integration

- **Required for:** async/await state machine
- **Effort:** High (runtime architecture change)

### 3c. async/await/generator (I-20260514-F3MYS5)

- **Unsupported:** 561 cases (async functions, await, generators, yield)
- **Requires:** Promise infrastructure + microtask queue
- **Sub-tasks:**
  - Async function state machine lowering
  - await expression suspension
  - Generator function protocol (next/return/throw)
  - yield* delegation (101 cases)
  - for-await-of
  - Async generator functions
- **Effort:** Very High (backend/runtime architecture)

---

## Phase 4: Collection + Proxy Polish

**Impact: +200-300 tests**
**Effort: 2-3 weeks**

### 4a. Proxy handler traps (I-20260519-VYQT8Y)

- **Mismatch:** 185, Pass: 95 (31%)
- **Status:** Traps mostly work but have edge case bugs
- **Effort:** Medium

### 4b. Reflect method fixes

- **Mismatch:** 127, Pass: 16 (10%)
- **Note:** Reflect maps to Object/Proxy operations — fixing Proxy fixes Reflect too
- **Effort:** Medium

### 4c. Map/Set remaining

- **Map:** 83 mismatches (8%)
- **Set:** 164 mismatches (11%)
- **Root cause:** Collection methods are WAT-level, many already done
- **Effort:** Medium

### 4d. WeakMap/WeakSet (I-20260519-NCHGK7)

- **Combined:** 55 tests
- **Effort:** Low-Medium

---

## Phase 5: Language-Level Features

**Impact: +300-500 tests**
**Effort: 4-6 weeks**
**Note:** These require IR-level and runtime architecture changes, not just WAT fixes

### 5a. Generator state machine improvements

- **Total:** 290, Mismatch: 170
- **Effort:** High (backend/runtime)

### 5b. yield* delegation

- **Cases:** 101
- **Status:** IR done, runtime needed
- **Effort:** Medium

### 5c. Arrow function `this` binding

- **Total:** 343, Mismatch: 120
- **Effort:** High (scope/closure)

### 5d. Dynamic import

- **Total:** 997, Mismatch: 69
- **Effort:** High (module system)

---

## Expected Progress

| Phase | Tests Unlocked (est.) | Cumulative Pass Rate |
|-------|----------------------|---------------------|
| Current | ~2,600 / 13,600 | 19% |
| Phase 1 (Foundation) | +100-130 | ~20-21% |
| Phase 2 (WAT Fixes) | +400-500 | ~23-25% |
| Phase 3 (Promise) | +200-300 | ~25-27% |
| Phase 4 (Collection/Proxy) | +200-300 | ~27-29% |
| Phase 5 (Language) | +300-500 | ~29-33% |
| **Total potential** | **+1,200-1,730** | **~28-32%** |

## Category Priorities (by mismatch count, easiest first)

| Category | Mismatch | Pass | Total | Pass % | Priority |
|----------|----------|------|-------|--------|----------|
| String | 619 | 423 | 1223 | 35% | 2a (quick win) |
| Promise | 327 | 54 | 677 | 8% | 3a |
| Number | 266 | 42 | 340 | 12% | 2c |
| Math | 262 | 30 | 327 | 9% | 2d |
| Function | 207 | 104 | 509 | 20% | 2e |
| DataView | 179 | 122 | 561 | 22% | 2f |
| Proxy | 185 | 95 | 311 | 31% | 4a |
| Set | 164 | 41 | 383 | 11% | 4c |
| Reflect | 127 | 16 | 153 | 10% | 4b |
| JSON | 91 | 0 | 165 | 0% | 2b |

## Recommended Focus Order

1. **Phase 1** first — keystone infrastructure unblocks the most tests per effort
2. **Phase 2a** (String) alongside Phase 1 — only 4 bugs, easy wins
3. **Phase 2b** (JSON) — host bridge is self-contained
4. **Phase 3** after Phase 1 — Promise depends on Object infrastructure
5. **Phase 4** after Phase 3 — Proxy/Reflect depend on Object infrastructure
6. **Phase 5** last — highest effort, needs everything above first

## Measurement

Track progress per category, not overall:

```bash
# Per-category pass rate
cargo nextest run -p ts2wasm-cli --test reference_coverage --no-fail-fast

# Compare before/after a phase
diff <(jq -r '...' before.jsonl) <(jq -r '...' after.jsonl)
```

The overall number will stay flat during Phase 1 (infrastructure work) then jump in Phase 2-3 as the infrastructure unlocks categories.
