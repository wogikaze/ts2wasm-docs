# Commit and Push Policy

This repository is normally worked on with git, although distributed archives may not include `.git`.

## Before work

```bash
git status --short
```

Do not mix user changes with agent changes. If no git metadata exists, produce a patch or updated archive and state that no commit could be created.

## Commit granularity

- One commit = one coherent intent.
- Separate docs-only, tests-only, refactor-only, and behavior-changing work when possible.
- Include validation commands in commit body when useful.

## Required checks

For docs-only changes:

```bash
git diff --check
python scripts/manager.py check
```

For code/runtime changes:

```bash
python scripts/manager.py gate-fast
python scripts/manager.py nextest
```

For capability/runtime/ABI changes, add the matching focused tests.

## Push

Do not bypass hooks. If a gate fails due to known baseline, report the exact command, failure summary, and why it is baseline rather than new regression.
