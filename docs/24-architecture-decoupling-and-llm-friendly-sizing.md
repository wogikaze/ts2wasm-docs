# Architecture Decoupling and LLM-Friendly Sizing

## Why this exists

Large, entangled files make both human review and agent work brittle. The repository should expose phase boundaries and route agents to focused docs rather than relying on one giant manual.

## Design rules

- Root `AGENTS.md` is a router, not a full manual.
- `docs/INDEX.md` must lead to task-specific docs within three hops.
- Compiler layers should depend downward, not sideways through convenience imports.
- Runtime helper declarations belong in catalog specs, not hidden in emitter code.
- Generated artifacts should be labeled and reproducible.

## File sizing guidance

- Prefer modules with one stable responsibility.
- Prefer real submodule paths over pseudo-hierarchical underscores in `src/` filenames; e.g. move `mir_dump.rs` toward `mir/dump.rs` when the structure permits it. Test files are exempt.
- If a docs file grows beyond a few hundred lines, split by task/phase and update `docs/INDEX.md`.
- If a Rust module grows by mixing declarations, emission, validation, and tests, split by phase or domain.

## Agent guidance

When context is tight, keep only:

- target task,
- changed files,
- relevant docs paths,
- focused test command/result,
- next blocker.
