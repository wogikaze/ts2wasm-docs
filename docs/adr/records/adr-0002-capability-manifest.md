# ADR-0002: Capability Manifest for Host and WASI Powers

## Status

Accepted

## Context

Generated wasm can require stdout, stdin, env, random, clock, filesystem, or Node-compatible host APIs. These powers must not be invisible side effects of compilation.

## Decision

Emit and validate `CapabilityManifest` JSON when requested. Host imports and enabled WASI capabilities require auditable reasons. Standalone manifests cannot require node host imports.

## Consequences

- Host-dependent runtime helpers must declare imports and reasons.
- `--host-deny` can enforce standalone-only builds.
- Manifest schema and runtime ABI version changes need docs and tests.

## Anchors

- `crates/shared/src/capability.rs`
- `crates/runtime-catalog/src/link_plan.rs`
- `docs/03-api-and-host-capability.md`
- `docs/09-security-and-capability-model.md`
