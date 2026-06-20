# Parallel Development

## When to use parent/child worktrees

Use child worktrees when work can be split by independent file scopes, especially compiler/runtime/test/docs slices that would otherwise collide or exceed one focused iteration.

## Commands

```bash
python scripts/manager.py spawn-worktrees -- --base master --prefix <prefix> <issue-file>...
python scripts/manager.py worktree-status
python scripts/manager.py git-worktree
```

## Parent responsibilities

- Define child scopes and done criteria.
- Avoid assigning the same file to multiple children.
- Merge child commits in a deliberate order.
- Run final focused and broader gates.
- Record any unresolved blocker with the next command/file.

## Child responsibilities

- Do not stop at investigation unless explicitly assigned.
- Produce a coherent change and focused validation.
- Report changed files, validation result, blockers, and commit/patch.

## Docs-specific note

For broad documentation work, split by canonical docs, agent/skill docs, and archive headers. Do not rewrite `issues/` unless requested.
