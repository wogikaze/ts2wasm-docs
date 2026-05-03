# Merge test262.py into reference-coverage.py

## Problem

`scripts/run/test262.py` and `scripts/run/reference-coverage.py` are two separate runners for the test262 suite with overlapping functionality, confusing naming, and duplicated code. Key gaps exist in both: reference-coverage lacks parallel execution and JSONL output; test262 lacks multi-suite support and comprehensive feature labeling.

## Design

### Architecture

```
scripts/
  run/
    reference-coverage.py  ← Merged runner (gains --jsonl, --jobs, --sample, --category)
  lib/
    test262_harness.py     ← NEW: Shared test262 harness functions
```

### New Flags in reference-coverage.py

| Flag | Effect |
|---|---|
| `--jsonl` | Enable JSONL per-test record output (test262 only) |
| `--jobs N` | Parallel execution via ThreadPoolExecutor (default: sequential) |
| `--sample N` | Run up to N files per category (test262 only) |
| `--category PATTERN` | Regex filter on extracted category (test262 only) |

Existing flags (`--limit`, `--json`, `--detail`, `--paths-file`, `--path-filter`, `--web-ui`/`--no-web-ui`) remain unchanged.

### lib/test262_harness.py Contents

Test262-specific code extracted from test262.py:

- Constants: `CORE_HARNESS_FILES`, `UNSUPPORTED_FLAGS`, `ASSERT_FAILURE_SENTINEL`, `TEST262_HOST_PRELUDE`, `WASM_HOST_PRELUDE`, `WASM_GLOBALS`, `WASM_HARNESS_SHIM`
- `Test262Metadata` class
- `parse_test262_metadata()` — full frontmatter parser
- `build_test262_source()` — constructs harness-wrapped source
- `compile_and_run_test()` — build + run + classify for a single test
- `process_one_test()` — orchestrator with Node comparison
- `get_node_reference()` — Node oracle execution
- `create_test_record()` — JSONL record factory
- `classify_completed_negative()` — negative test outcome handler

### Merge Mapping

| test262.py Function | Destination |
|---|---|
| `escape_json()` | lib/test262_harness.py |
| `extract_category()` | reference-coverage.py (generalized) |
| `stable_test_path()` | Deleted (use reference-coverage's `repo_relative()`) |
| `matches_path_filter()` | Deleted (use reference-coverage's `apply_path_filters()`) |
| `_parse_yaml_list()` | Deleted (merged into metadata parser in lib) |
| `feature_label()` (simple) | Deleted (use reference-coverage's comprehensive version) |
| `refresh_web_ui_data()` | Deleted (use reference-coverage's version) |
| `main()` | reference-coverage.py (generalized main with --jsonl mode) |

### Output Changes

When `--jsonl` is active and suite=test262:
- Each test produces a JSONL record to stdout (same as current test262.py)
- Coverage summary is still written to `artifacts/coverage/results/test262.json`
- JSONL records are also written to `artifacts/coverage/results/test262-results.jsonl` (for web UI)
- Web UI data refresh uses reference-coverage's version (scans all JSONLs)

When `--jsonl` is NOT active: behavior is identical to current reference-coverage.py.

### Backward Compatibility

- `test262` mise task will be removed; users run `reference-coverage test262 --jsonl` instead
- JSONL output format is preserved (same schema)
- Coverage summary format (test262.json) is preserved
- test262.py file is deleted

## Non-Goals

- Merging tsc/tsgo support into the JSONL parallel runner (tsc/tsgo continue working as before)
- Changing the JSONL record schema
- Changing the coverage summary format
- Changing the web UI data schema
