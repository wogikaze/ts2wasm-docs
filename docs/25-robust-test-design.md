# Robust Test Design

Robust tests make semantic drift visible without becoming brittle snapshots of unrelated implementation detail. This is a durable test contract, not a historical completion report.

## Principle

- Test observable JavaScript behavior with differential tests when possible.
- Test compiler invariants with focused unit/property tests.
- Test generated artifacts structurally when semantics are not the claim.
- Keep every unsupported/blocked result traceable to a feature label or issue.

## Test Classes

| Class | Use for | Evidence |
|---|---|---|
| `build_smoke` | compile pipeline coverage | build succeeds or expected diagnostic appears |
| `semantic_diff` | user-visible JS/TS behavior | Node/oracle output equals wasm/iwasm output |
| `parser_smoke` | syntax acceptance only | AST parse/validation result |
| `negative_diagnostic` | unsupported or invalid input | specific diagnostic code/category |
| `runtime_invariant` | ABI, link plan, host import, memory layout | structural assertions and snapshots |
| `reference_coverage` | broad suite progress | JSONL/matrix with evidence command |

Build-smoke evidence must never be described as semantic parity.

## Patterns

- Pair parser acceptance with semantic coverage or explicit unsupported coverage.
- Add a negative diagnostic when rejecting a syntax/semantic family.
- Validate runtime ABI invariants directly when tags/layout change.
- Keep host capability tests structural: manifest imports must match wasm imports.
- Use focused reference shards for regression detection before full-suite runs.
- Use canary fixtures for high-risk semantics such as BigInt, arrays, modules, async, exceptions, and object descriptors.

## Anti-Patterns

- Broad skips without a feature label and recheck condition.
- Snapshot-only tests for behavior that needs semantic oracle coverage.
- Asserting exact stderr prose when a stable diagnostic code exists.
- Updating generated coverage artifacts without the generator command.
- Counting `verified_negative_compile` as positive semantic compatibility.

## Flake And Performance Policy

Flaky tests require a quarantine reason, owner, and recheck condition. Performance smoke tests must record benchmark name, input size, target, runner version, cold/warm mode, iteration count, median, p95, peak memory, wasm size, and host call count when available.

## Docs Linkage

New test classes or output fields must update:

- `docs/06-testing-and-coverage.md`,
- `docs/11-shared-definitions.md`,
- `docs/17-jsonl-test-record-schema.md`,
- `docs/18-web-ui-reporting.md` when dashboard data changes,
- relevant feature or ABI docs.
