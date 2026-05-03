# Test Status Taxonomy Upgrade

## Problem

Current test result statuses collapse meaningful distinctions:
- `fail` covers output mismatch, runtime crash, and compiler bugs as one category
- There is no explicit `runtime_error` or `mismatch` counter in coverage artifacts
- The web UI only shows pass/fail/skip/error, losing diagnostic granularity

## Design

### 1. JSONL Test Record Statuses (6 values)

| Status | Meaning | Previously |
|---|---|---|
| `pass` | Build succeeded + iwasm output matches Node reference | `pass` |
| `mismatch` | Build succeeded + both ran successfully + output differs | `fail` (buried) |
| `runtime_error` | Build succeeded + iwasm crashed/non-zero exit | `fail` (buried) |
| `fail` | InvariantViolation / Test262AssertFailure / ExpectedNegativeFailure | `fail` |
| `unsupported` | Feature not yet implemented (UnsupportedSyntax etc.) | `unsupported` |
| `blocked` | External blocker (timeout, BackendIo, Node failure) | `blocked` |

### 2. Coverage Summary Counters

Existing fields kept for backward compatibility:

| Field | Status |
|---|---|
| `build_pass` | kept (compat) |
| `semantic_pass` | kept (compat, also used for "passed" display) |
| `mismatch` | **new**: build succeeded + both ran + output differs from Node |
| `runtime_error` | **new**: build succeeded + iwasm non-zero exit |
| `fail` | kept: InvariantViolation only |
| `unsupported` | kept |
| `blocked` | kept |
| `skip_with_reason` | kept |

### 3. web UI Display (6 categories)

Summary cards in Test Results tab:

```
✅ Passed: N | ❌ Mismatch: N | 💥 Runtime: N | 🔧 Build Error: N | ⏳ Unsupported: N | 🚫 Blocked: N
```

- Filter dropdown and row coloring updated to match new categories
- Coverage tab updated to show `mismatch` and `runtime_error` in suite breakdowns

## Implementation Plan

### Phase 1: reference-coverage.py

- Add `mismatch_count` and `runtime_error_count` counters
- Expand semantic check logic to distinguish:
  - node fails → blocked (node reference unavailable)
  - wasm crashes → runtime_error
  - both succeed + output matches → semantic_pass
  - both succeed + output differs → mismatch
- Add new fields to summary dict and output (JSON + key=value)
- Keep old fields (build_pass, semantic_pass) for compat

### Phase 2: test262.py

- In `process_one_test()`:
  - output mismatch → `mismatch` (was `fail` with "output mismatch")
- In `compile_and_run_test()`:
  - iwasm non-zero exit → `runtime_error` (was `fail` with "RuntimeError:N")
  - Test262AssertFailure → keep as `fail`
  - ExpectedNegativeFailure → keep as `fail`
- Keep `fail` for InvariantViolation and assertion failures

### Phase 3: web-ui-data.py

- Update `STATUS_MAP`: add `mismatch` → `mismatch`, `runtime_error` → `runtime_error`
- Update `count_summary()` to calculate 6 categories
- Update `build_test_results()` metadata
- Coverage data passes through `mismatch` and `runtime_error` from artifacts

### Phase 4: web UI (TypeScript)

- Update `types.ts`: add `mismatch`, `runtime_error`, `build_error` to status types
- Update `useData.ts` summary interface to match new 6-field summary
- Update `App.tsx`:
  - 6 summary cards with distinct colors/icons
  - Filter dropdown includes new categories
  - Row color/icon per new status

## Backward Compatibility

- Old fields in coverage JSONs (`build_pass`, `semantic_pass`) are preserved
- Old JSONL records with `fail`/`unsupported`/`blocked` statuses are still valid
- Web UI treats unknown statuses as `error` (graceful fallback)
- `mismatch` and `runtime_error` counters default to 0 in old data

## Non-Goals

- Rust `TestStatus` enum changes (canonical enum stays at pass/fail/unsupported/blocked/skip-with-reason)
- CI/CD deployment changes (in scope of separate issue)
- History tab data migration (old history entries keep old categories)
