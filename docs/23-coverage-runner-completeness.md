# Coverage Runner Completeness

Reference coverage must classify outcomes accurately, preserve evidence, and avoid conflating build success with semantic success.

## Runner Paths

Coverage can flow through three execution paths:

| Path | Contract |
|---|---|
| Server batch mode | Preferred for large suites; reports per-case build/runtime/oracle outcomes. |
| Legacy subprocess mode | Fallback path; must keep outcome semantics compatible with server mode. |
| Server failure fallback | Infrastructure failure must be visible per case or as a clearly blocked shard. |

The same source case must not receive a stronger semantic classification solely because it used a different runner path.

## Required Properties

- Each run records suite, denominator, executed count, outcome counts, and evidence command.
- Unsupported diagnostics are grouped by diagnostic code and feature reason.
- Negative compile tests are separated from positive semantic passes.
- Batch server timeouts, panics, and transport failures are reported without hiding selected cases.
- JSONL schema remains compatible with `docs/17-jsonl-test-record-schema.md`.
- Dashboard/report generation keeps enough metadata to trace results back to the runner command.

## Build Pass Versus Semantic Pass

`build_pass` means wasm was produced. It is not semantic evidence.

A case can become `semantic_pass` only when:

1. the wasm build succeeded,
2. the oracle policy requires or allows semantic execution,
3. the wasm runtime completed successfully,
4. stdout/observable result matched the oracle, and
5. the record has `semantic_checked=true`.

Negative compile tests are successful only when the expected rejection is verified. Use `verified_negative_compile`; do not count it as positive semantic parity.

## Completeness Checks

When changing the runner or matrix generation, check:

- server mode and legacy mode classify the same sample consistently,
- build-only counters remain visible,
- verified negative compile cases are not mixed into positive pass counts,
- `selection_hash` changes when the selected case list changes,
- path-filtered or sampled runs are labeled as such in evidence,
- generated matrix updates include the generator command.

## Commands

```bash
python scripts/manager.py reference-coverage test262 --jsonl
python scripts/manager.py reference-coverage tsc --no-semantic
python scripts/manager.py reference-coverage tsgo --no-semantic
python scripts/manager.py test-differential-reporter < run.jsonl
python scripts/manager.py update-coverage-matrix
python scripts/check/test-records-schema.py --self-test
```

## Completeness Warning

A sampled, limited, or path-filtered run is useful for triage but must not be described as full-suite status. Keep the exact evidence command visible.
