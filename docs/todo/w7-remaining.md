# W7 (Host capability boundary) — remaining work

Last updated: 2026-05-08

## Current status

### What works

- **Capability manifest**: `--emit-manifest` outputs JSON manifest with schema_version=1.
- **Capability schema**: Defined in `docs/11-shared-definitions.md` and `crates/shared/src/capability.rs`.
- **Host-deny tests**: `crates/cli/tests/m11_host_deny.rs` verifies:
  - Standalone console.log allowed without Node host
  - WASI filesystem read/write allowed
  - Node host imports (fs, crypto, path, process) rejected under host-deny
  - Math.random declares `wasi.random` without Node host
  - Date.now() declares `wasi.clock.realtime` without Node host
  - Deterministic Date epoch test omits `wasi.clock.realtime`
  - Static eval declares no Node host eval capability
  - Standalone fixtures pass host-deny
- **Host import tracking**: `RuntimeLinkPlan` tracks which runtime functions need host imports vs WASI-only.
- **Standalone purity**: `standalone: true` in manifest when no Node host imports needed.
- **Schema migration policy**: Documented with versioning and backward compatibility rules.

### What doesn't work

- **Node host import growth**: Each new Node-builtin-like runtime function must be audited for host dependency.
- **Capability reasons**: Some imports may lack proper reason documentation.

## Remaining work

### Capability audit

- [ ] **Full host import audit**: Run `--emit-manifest` on all fixture files; verify each manifest accurately reflects the actual wasm imports.
- [ ] **Audit new RuntimeFn additions**: Each new runtime function in `runtime_fn.rs` must be classified as standalone (WASI-only) or host-required.
- [ ] **Schema version bump procedure**: Tooled migration path when adding new manifest fields.

### Host-deny test expansion

- [ ] **Expand host-deny test matrix**: Cover all categories of runtime functions.
- [ ] **Negative test for each host import**: Verify each Node host import is correctly rejected under `--host-deny`.
- [ ] **WASI-only runtime function audit**: Create a test that ensures all WASI-only functions don't accidentally require Node host.
- [ ] **Manifest fixture golden tests**: Add golden file tests comparing emitted manifest against expected JSON.

### Standalone boundary

- [ ] **Standalone assurance for new features**: Each new feature (Promise, Proxy, etc.) needs standalone/WASI capability analysis.
- [ ] **WASI-only output guarantee**: Document the full set of runtime functions that work standalone.

### Policy

- [ ] **Capability review checklist**: Add to `docs/12-coding-standard.md` or gate checklist: "Does this change introduce a new host import? If so, update manifest."
- [ ] **CI gate**: Add manifest-imports check (`mise run check-manifest-imports`) to pre-push or CI gate.

## Key open issues

- Gate C: manifest output and import consistency
- Gate F: standalone target runs without Node host imports

## Reference

- Test file: `crates/cli/tests/m11_host_deny.rs` (12 test functions)
- Capability schema: `docs/11-shared-definitions.md` §Capability manifest schema
- Runtime link plan: `crates/backend-wasm/src/runtime_link_plan.rs`
- Shared crate: `crates/shared/src/capability.rs`
