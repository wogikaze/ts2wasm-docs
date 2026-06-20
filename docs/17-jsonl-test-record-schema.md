# JSONL Test Record Schema

Differential tests, reference coverage, and dashboard imports use JSONL: one JSON object per line. The schema has two layers:

1. Canonical `TestRecord` for project fixtures and differential tests.
2. Coverage runner schema v2 for reference-suite results from `scripts/run/reference-coverage.py`.

## Layer 1: Canonical TestRecord

| Field | Type | Required | Description |
|---|---|---|---|
| `suite` | string | always | Test suite path, e.g. `fixtures/arrays-objects`. |
| `case` | string | always | Test case filename or stable suite path. |
| `target` | string | always | Runtime target, e.g. `wasm32-wasi` or `wasm-iwasm`. |
| `status` | string | always | `pass`, `fail`, `unsupported`, `blocked`, or `skip-with-reason`. |
| `expected` | string or null | on mismatch | Oracle stdout or expected diagnostic. |
| `actual` | string or null | on mismatch | Wasm/iwasm stdout or actual diagnostic. |
| `reason` | string or null | on unsupported/blocked/skip | Human-readable classification. |
| `tracking` | string or null | on unsupported/blocked/skip | `issue-NNN` or `feature:label`. |

Validation rules:

- `suite`, `case`, and `target` must be non-empty.
- `unsupported`, `blocked`, and `skip-with-reason` require `reason` and `tracking`.
- `pass` must not hide stderr/runtime failures.
- `fail` must include enough expected/actual detail to reproduce the mismatch.

## Layer 2: Coverage Runner Schema V2

Reference coverage records extend `TestRecord` with `schema_version: 2` and runner-specific fields.

| Field | Type | Description |
|---|---|---|
| `schema_version` | integer | Always `2` for coverage-runner records. |
| `outcome` | string | Canonical outcome from `CoverageOutcome`. |
| `build_pass` | boolean | Whether the compiler produced a valid wasm module. |
| `semantic_checked` | boolean | Whether an oracle comparison was attempted. |
| `phase` | string or null | `metadata`, `prepare`, `parse`, `compile`, `link`, `runtime`, `oracle`, or `triage`. |
| `diagnostic_code` | string or null | Compiler diagnostic code, e.g. `UnsupportedSyntax`. |
| `feature_label` | string or null | Feature label from coverage classification. |
| `oracle_policy` | string or null | `auto`, `always`, or `never`. |
| `selection_hash` | string or null | SHA-256 of the canonical sorted case path list. |
| `abi_version` | string or null | Runtime ABI version string. |
| `target_id` | string or null | Target identifier, e.g. `wasm-iwasm`. |
| `ts_boundary` | string or null | `ts-parse`, `ts-erase`, `ts-declaration-only`, `ts-emit`, `ts-runtime`, or `unknown`. |
| `executable_source` | boolean or null | Whether the source contains executable code. |
| `declaration_only` | boolean or null | Whether the source is declaration-only. |
| `duration_ms` | integer or null | Wall-clock duration. |

Outcome values:

| Outcome | Meaning |
|---|---|
| `build_pass` | Compiler produced wasm; no semantic check was attempted. |
| `semantic_pass` | Wasm output matched the oracle. |
| `semantic_mismatch` | Wasm output differed from the oracle. |
| `runtime_error` | Wasm ran but exited non-zero or trapped. |
| `unsupported` | Compiler produced an unsupported diagnostic. |
| `blocked` | Runner infrastructure failure. |
| `internal_failure` | Compiler panic or invariant failure. |
| `verified_negative_compile` | Negative compile test was correctly rejected. |
| `unverified_negative_compile` | Negative compile test rejected without verified error type. |
| `oracle_skipped` | Oracle was unavailable or intentionally skipped. |
| `skip_with_reason` | Case skipped with an explicit reason. |

## Examples

```jsonl
{"suite":"fixtures/arrays-objects","case":"array.ts","target":"wasm32-wasi","status":"pass","expected":null,"actual":null,"reason":null,"tracking":null}
{"suite":"fixtures/test-infrastructure","case":"unsupported-fixture.ts","target":"wasm32-wasi","status":"unsupported","expected":null,"actual":null,"reason":"Unsupported syntax: async","tracking":"feature:async"}
{"build_pass":true,"case":"test/language/foo.js","expected":"ok","actual":"ok","outcome":"semantic_pass","schema_version":2,"semantic_checked":true,"status":"pass","suite":"test262","target":"wasm-iwasm"}
{"build_pass":true,"case":"test/language/baz.js","outcome":"runtime_error","schema_version":2,"semantic_checked":false,"status":"runtime_error","suite":"test262","target":"wasm-iwasm","reason":"iwasm trap"}
```

## Source Of Truth

- `scripts/lib/coverage_outcome.py`: `CoverageOutcome`, `CoveragePhase`, and record constructors.
- `scripts/lib/coverage_labels.py`: feature labels and diagnostic classification.
- `scripts/check/test-records-schema.py`: validates canonical records and schema v2 records.
- `crates/shared/src/test_status.rs`: Rust `TestRecord`, `TestStatus`, `TrackingId`.
- `crates/cli/tests/differential_jsonl.rs`: fixture JSONL enumeration and differential runner checks.

## Validation

```bash
python scripts/manager.py check-test-records-schema < records.jsonl
python scripts/check/test-records-schema.py --self-test
```

Schema changes must update dashboard types in `web-ui/src/types.ts`, reporting docs in `docs/18-web-ui-reporting.md`, and any coverage fixtures that assert JSON keys.
