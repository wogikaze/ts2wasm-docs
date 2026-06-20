# Feature Implementation Checklist

Use this checklist for any new JS/TS semantic feature.

## 1. Ownership

- [ ] Identify owner: frontend, resolve, semantics, IR, runtime, backend, CLI, or tooling.
- [ ] Update `docs/28-frontend-syntax-ownership.md` if syntax ownership changes.
- [ ] Update `docs/13-ir-contracts.md` if HIR/MIR/Lowered shape changes.

## 2. Parser and diagnostics

- [ ] Parser accepts or rejects the syntax intentionally.
- [ ] Unsupported forms produce specific `DiagCode`.
- [ ] Parser snapshots or negative tests are updated.

## 3. Resolution and semantics

- [ ] Name/binding rules are explicit.
- [ ] Builtin identity or host API classification is added if needed.
- [ ] Semantic validation rejects impossible states.

## 4. IR and runtime

- [ ] IR variants or semantic operations are validated.
- [ ] RuntimeFn spec includes deps/imports/capabilities/strings/signature.
- [ ] Runtime ABI constants/types are used instead of magic numbers.

## 5. Backend and capability

- [ ] Backend emission consumes validated IR/Lowered state.
- [ ] Capability manifest changes are tested.
- [ ] `--host-deny` behavior is covered when host imports are involved.

## 6. Tests and docs

- [ ] Focused unit/snapshot/differential/negative test exists.
- [ ] Fixture catalog and feature matrix are updated.
- [ ] Relevant docs are updated with Why and constraints.
- [ ] Final report lists validation command and result.
