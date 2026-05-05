# Parallel Parent/Child Development

This workflow runs child agents in isolated git worktrees while a parent agent supervises assignment, review, merge, and Discord reporting.

It intentionally does not use `.agents/state`, `current_task.json`, `project_state.json`, or `dev-loop`.

## Layout

- Worktrees live in sibling directories: `../ts2wasm-<prefix>-<id>-<timestamp>/`
- Parent/child prompts live under `.agents/prompts/`
- Local assignment files live under ignored `reports/agents/<agent_id>/assignment.md`
- Discord run reports live under ignored `reports/runs/<run_id>/`
- Reference corpus is symlinked with `mise run link-reference`

## Parent Loop

The parent follows:

```text
SYNC -> QUEUE_SCAN -> SPLIT_OR_SELECT -> WORKTREE_ASSIGN
-> CHILD_SUPERVISE -> MERGE_REVIEW -> REPORT -> QUEUE_REFILL
```

The tracked source of truth remains `issues/`, `docs/`, and git history. Parent queue notes and child assignments are local report artifacts, not tracked state.

## Batch Worktree Creation

Create one worktree per issue:

```bash
mise run spawn-worktrees -- \
  --base master \
  issues/open/225-*.md \
  issues/open/255-*.md
```

The command outputs a JSON manifest and creates local assignment files under `reports/agents/`.

Each worktree gets:

- isolated branch from `--base`
- reference corpus symlink when available
- shared cargo `target/` through `.cargo/config.toml`
- no `.agents/state` files

## Child Launch

For each manifest entry, start a child with:

- prompt: `.agents/prompts/autonomous-child-worker.md`
- assignment: `reports/agents/<agent_id>/assignment.md`
- worktree path from the manifest

The child reports back with a `PARENT_EVENT:` line.

## Status Collection

Collect all worktree status:

```bash
mise run worktree-status -- --format json
```

Useful variants:

```bash
mise run worktree-status -- --dirty-only
mise run worktree-status -- --ahead-only --base origin/master
```

## Merge Review

The parent reviews each child branch before merge:

1. Inspect diff scope and commits.
2. Confirm validation evidence.
3. Run relevant narrow validation.
4. Run `mise run check`.
5. Merge or cherry-pick only after review passes.
6. Run `mise run update-issue-index` and `mise run check issues` when issues changed.

## Discord Reporting

Discord reporting is required after each parent cycle and issue-close wave:

```bash
mise run discord-report -- reports/runs/<run_id>/cycle_report.md --run-id <run_id>
```

If sending fails or no webhook is configured, save the markdown/payload under `reports/runs/<run_id>/` and report that it is deferred.

## Prerequisites

- `mise run spawn-worktrees`
- `mise run worktree-status`
- `mise run discord-report`
- `mise run check`
