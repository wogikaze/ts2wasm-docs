# API and Host Capability Model

## Purpose

The capability model ensures generated wasm does not acquire host powers invisibly. Every required host import or WASI capability must appear in a manifest with an auditable reason.

## Manifest contract

The shared manifest type is `CapabilityManifest` in `crates/shared/src/capability.rs`.

Key fields:

| Field | Meaning |
|---|---|
| `schema_version` | manifest schema version; current value is `1` |
| `target` | compatibility target string |
| `target_id` | canonical target id |
| `target_aliases` | accepted aliases for the target |
| `runtime_abi_name` | `ts2wasm-runtime-abi` |
| `runtime_abi_version` | runtime ABI version; current value is `2` |
| `standalone` | true when no node host shim is required |
| `wasi` | stdout/stdin/env/random/clock/filesystem capability flags |
| `node_host` | host shim requirement and import names |
| `capability_reasons` | reason list keyed by capability/import |

## Validation rules

- `schema_version` must match the current schema.
- `standalone=true` cannot coexist with `node_host.required=true`.
- `node_host.required=true` requires at least one import.
- Every enabled WASI capability needs a `capability_reasons` entry.
- Every Node host import must start with `host.` and have a reason.
- Capability reasons are canonicalized and deduplicated before serialization.

## Runtime link plan

Runtime functions declare dependencies, host imports, capabilities, runtime strings, and result shape in `crates/runtime-catalog/src/runtime_fn.rs`. `RuntimeLinkPlan` collects the transitive closure and validates consistency. The backend should consume a validated plan, not ad-hoc runtime lists.

## Host import policy

| Import family | Capability expectation |
|---|---|
| WASI stdout/stdin/stderr | corresponding `wasi.*` reason |
| Clock/date live values | `wasi.clock.realtime` or host date shim reason |
| Node process/fs/path/crypto | `node_host.required=true`, `host.*` import, explicit reason |
| test262 hooks | host shim import with test-only reason |

## Adding a host capability

> **IMPORTANT**: `RuntimeFn` variant addition is FROZEN (P5 constraint).
> New host capabilities MUST use `SpecOp` (see `crates/spec-kernel/`).
> Legacy `RuntimeFn` entries are for backward compatibility only.

1. Define the new capability as a `SpecOp` variant in `crates/spec-kernel/`.
2. Implement the runtime function in `crates/backend-wasm/src/runtime/spec/`.
3. Update link-plan tests.
4. Ensure manifest JSON includes the reason.
5. Add a fixture that proves `--host-deny` rejects the host-dependent path when appropriate.
6. Update this doc if a new capability family appears.
