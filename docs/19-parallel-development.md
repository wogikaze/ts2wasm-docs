# Parallel Development Workflow

Workflow for running independent tasks in isolated git worktrees.

## Layout

- Worktrees live in sibling directories: `../ts2wasm-<prefix>-<id>-<timestamp>/`
- Batch scripts under `scripts/dev/`
- State setup: `_worktrees/setup-worktree.py`
- Reference corpus symlinked (not copied) via `mise run link-reference`

## Batch worktree creation

Create up to 12 worktrees per wave for 12-core hardware:

```bash
./scripts/dev/spawn-worktrees.sh \
  --base master \
  issues/open/225-*.md \
  issues/open/255-*.md \
  issues/open/274-*.md
```

Output: JSON manifest with worktree paths, branch names, issue IDs, base refs.

Each worktree gets:
- Isolated git branch from `--base`
- Reference corpus symlinks
- Shared cargo `target/` directory (via `.cargo/config.toml` → parent's `target/`)
- Dev-loop state files (`.agents/state/project_state.json`, `current_task.json`)

## File affinity groups

Issues in different groups can safely run in parallel. Groups defined in
`setup-worktree.py` and `autonomous-parent-orchestrator.md`:

| Group | Scope |
|-------|-------|
| frontend/parser | `crates/frontend/src/parser/` |
| frontend/semantics | `crates/frontend/src/`, `crates/ir/src/` |
| ir/lowering | `crates/ir/src/resolved.rs`, `crates/ir/src/lowered.rs` |
| runtime/semantics | IR + backend expr_emit + fixtures |
| runtime/builtins | `crates/runtime-abi/src/` + fixtures |
| backend/wasm | `crates/backend-wasm/src/` |
| cli/orchestration | `crates/cli/src/`, `crates/compiler/src/` |
| test/fixtures | `fixtures/`, `crates/cli/tests/` |
| meta/issues | `issues/`, `docs/` |

Assign at most one issue per group per wave.

## Supervision

Batch status collection for the 15-minute supervision cycle:

```bash
# All worktrees, text table
./scripts/dev/worktree-batch-status.sh

# Dirty-only, JSON output (for parent agent)
./scripts/dev/worktree-batch-status.sh --format json --dirty-only

# Ahead-of-base only
./scripts/dev/worktree-batch-status.sh --ahead-only --base origin/master
```

Collects worktree status in parallel using background processes.

## Integration

When merging multiple parallel branches:

1. Merge/cherry-pick branches one at a time, starting with lowest-conflict groups.
2. For `issues/index.md` conflicts: accept ours and regenerate:

   ```bash
   git checkout --ours -- issues/index.md
   mise run update-issue-index
   git add issues/index.md
   ```

3. After each merge, run:

   ```bash
   mise run update-issue-index -- --check
   mise run check issues
   ```

4. Run `mise run gate` after all branches are merged.

## Shared cargo target

All worktrees point `.cargo/config.toml` → parent repo's `target/` directory.
This:
- Saves ~1.8 GB per worktree (×12 = ~21.6 GB saved)
- Reuses compiled crate cache across worktrees
- Is created automatically by `setup-worktree.py` and `spawn-worktrees.sh`

## Prerequisites

- Worktree helpers: `scripts/dev/git-worktree.sh`
- Batch spawn: `scripts/dev/spawn-worktrees.sh`
- Batch status: `scripts/dev/worktree-batch-status.sh`
- `.config/nextest.toml` — 8 parallel test workers

## CI hooks

- Pre-commit: fmt + clippy + issue health + markdownlint
- Pre-push: fast gate + architecture rules + diff smoke + webhook
