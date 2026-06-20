# Windows Development

## Recommended path

Use WSL2 for full compatibility with Unix-like tooling, WAMR/iwasm, shell hooks, and reference corpora.

## Native Windows path

Many project commands use the Python manager and can run without bash:

```powershell
python scripts/manager.py check
python scripts/manager.py gate-fast
python scripts/manager.py nextest
python scripts/manager.py coverage-report --format markdown
```

`mise` is optional. If installed, `mise run <task>` forwards to the same Python manager for most tasks.

## Known limitations

- Some shell scripts and hooks remain bash-oriented.
- Reference corpora and symlink-heavy workflows are easier in WSL2.
- WAMR/iwasm availability depends on local installation.
- Path handling should be tested when adding scripts.

## Rule for new scripts

Prefer Python manager entry points for cross-platform workflows. If a bash script is necessary, document the Windows/WSL behavior.
