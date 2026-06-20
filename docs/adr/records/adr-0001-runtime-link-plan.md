# ADR-0001: Runtime Link Plan as Runtime Source of Truth

## Status

Accepted

## Context

Runtime helpers have dependencies, host imports, capabilities, globals, strings, and signatures. If these are hidden inside WAT templates or backend emit functions, generated wasm becomes hard to audit.

## Decision

Declare runtime helper metadata in `runtime-catalog` and collect required helpers through a validated `RuntimeLinkPlan`. Backend emission consumes the validated plan rather than ad-hoc dependency lists.

## Consequences

- Runtime additions need catalog metadata and link-plan tests.
- Capability manifest emission can be derived from the same plan.
- The catalog is large, so docs and tests must route contributors carefully.

## Anchors

- `crates/runtime-catalog/src/runtime_fn.rs`
- `crates/runtime-catalog/src/link_plan.rs`
- `docs/03-api-and-host-capability.md`
- `docs/14-runtime-abi.md`
