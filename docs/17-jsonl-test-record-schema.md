# JSONL Test Record Schema

Differential test results (node-diff fixtures, test262, and other suites) are reported as JSONL (JSON Lines) — one JSON object per line.

## Output Schema

Each line is a JSON object with the following fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `suite` | string | always | Test suite path, e.g. `"fixtures/arrays-objects"` |
| `case` | string | always | Test case filename, e.g. `"array.ts"` |
| `target` | string | always | Target runtime, e.g. `"wasm32-wasi"` |
| `status` | string | always | One of `"pass"`, `"fail"`, `"unsupported"`, `"blocked"`, `"skip-with-reason"` |
| `expected` | string or null | on fail | Node.js stdout; present when status=`"fail"` with stdout mismatch |
| `actual` | string or null | on fail | iwasm stdout; present when status=`"fail"` with stdout mismatch |
| `reason` | string or null | on unsupported/blocked | Human-readable explanation |
| `tracking` | string or null | on unsupported/blocked | Tracking ID: `issue-NNN` (GitHub issue) or `feature:xxx` (feature label) |

## Status Values

| Status | Meaning |
|--------|---------|
| `"pass"` | Node and iwasm stdout match exactly |
| `"fail"` | Build failed (compiler bug), iwasm timed out, iwasm crashed, or stdout mismatch |
| `"unsupported"` | Compiler rejected the fixture with an unsupported construct diagnostic |
| `"blocked"` | I/O error, missing runtime, or command execution failure |
| `"skip-with-reason"` | Skipped test with an explicit reason |

## Tracking ID Format

Tracking IDs follow one of two formats:

- `issue-NNN` — references GitHub issue number NNN
- `feature:xxx` — references a feature label

## Validation Rules

- `pass` / `fail`: reason and tracking are optional
- `unsupported` / `blocked` / `skip-with-reason`: reason and tracking are required
- `suite`, `case`, and `target` must be non-empty

## Example Records

```jsonl
{"suite":"fixtures/arrays-objects","case":"array.ts","target":"wasm32-wasi","status":"pass","expected":null,"actual":null,"reason":null,"tracking":null}
{"suite":"fixtures/test-infrastructure","case":"unsupported-fixture.ts","target":"wasm32-wasi","status":"unsupported","expected":null,"actual":null,"reason":"Unsupported syntax: UnsupportedSyntax/async","tracking":"feature:async"}
{"suite":"fixtures/core-semantics","case":"bigint-runtime-add-sub.ts","target":"wasm32-wasi","status":"fail","expected":"3\n","actual":"5\n","reason":"stdout mismatch: node=\"3\\n\", iwasm=\"5\\n\"","tracking":"feature:stdout-mismatch"}
{"suite":"fixtures/module-system","case":"require-cache.ts","target":"wasm32-wasi","status":"blocked","expected":null,"actual":null,"reason":"I/O or command execution failure","tracking":"feature:backend-io"}
```

## Consumer Usage

### Using `cargo test` (recommended for CI)

```bash
# Run the full JSONL sweep (slow: all fixtures)
cargo nextest run -p ts2wasm-cli --test differential_jsonl --run-ignored ignored-only

# Quick format check (fast: 10-fixture sample)
cargo nextest run -p ts2wasm-cli --test differential_jsonl -- differential_jsonl_quick_check_formats

# Smoke test on test-infrastructure fixtures
cargo nextest run -p ts2wasm-cli --test differential_jsonl -- differential_jsonl_test_infrastructure_smoke
```

### Using the differential checker script

```bash
mise run check differential -- --jsonl
```

This runs all fixtures through the differential test runner and outputs JSONL records.

## Schema Source of Truth

The canonical schema is implemented in:

- `crates/shared/src/test_status.rs` — `TestRecord`, `TestStatus`, `TrackingId` types with `validate()` and `to_json_line()`
- `crates/cli/tests/differential_jsonl.rs` — JSONL enumeration and differential runner

## Related

- `docs/06-testing-and-coverage.md` — overall testing strategy
- `docs/11-shared-definitions.md` — shared test definitions
