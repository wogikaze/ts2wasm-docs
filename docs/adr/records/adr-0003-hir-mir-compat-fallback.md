# ADR-0003: HIR/MIR Migration with Compatibility Fallback

## Status

Accepted

## Context

The legacy `LoweredProgram` path is the default working compiler route. HIR/MIR is needed for clearer semantics and backend contracts, but forcing it too early would break supported fixtures.

## Decision

Expose strict and compatibility fallback CLI flags. Strict HIR/MIR errors when unsupported. Compat fallback tries HIR/MIR and falls back to legacy emission when necessary.

## Consequences

- Tests can compare MIR and legacy output incrementally.
- Unsupported MIR remains visible.
- Docs must state which path a feature relies on.

## Anchors

- `crates/compiler/src/pipeline.rs`
- `docs/13-ir-contracts.md`
- `docs/27-ir-layer-completion.md`
