# Workstream todo files

各 workstream の残作業を記録する。正本は `docs/11-shared-definitions.md` の Workstreams 定義。

## Summary

| File | Workstream | Current status | Next gate target |
|---|---|---|---|
| `w0-remaining.md` | W0: Runtime substrate | Gate A precondition — dirty files, build gate, architecture debt | Gate A |
| `w1-remaining.md` | W1: Standalone WASI | Foundation — console.log/stdin/WASI imports work | Gate A |
| `w2-remaining.md` | W2: Syntax coverage | UnsupportedSyntax = 204/500 — #1 blocker | Gate B (UnsupportedSyntax=0) |
| `w3-remaining.md` | W3: Name resolution | UnresolvedName=120 + UnresolvedFunction=70 — #2/#3 blockers | Gate E (build_pass >= 5,000) |
| `w4-remaining.md` | W4: Builtin runtime | 18 runtime files, ~83 open runtime issues; Promise/Proxy missing | Gate G (semantic_pass >= 50%) |
| `w5-remaining.md` | W5: Runtime semantics | 330 open semantic issues; iterator protocol, prototype chain, completion records | Gate G/H |
| `w6-remaining.md` | W6: test262 coverage ramp | Current limit=500/53,445; build_pass=61/31 semantic | Gate D (executed >= 2,000) |
| `w7-remaining.md` | W7: Host capability | Manifest works, 12 host-deny tests, audit coverage | Gate C/F |
| `w8-remaining.md` | W8: Optimization | Deferred to post-90% test262; no active work | Gate G (benchmark) |

## 90% test262 trajectory

```
Gate A (現状) ──► Gate D (executed >= 2,000) ──► Gate E (build_pass >= 5,000)
    │                                                   │
    └── W0/W1 完了 ──► W2/W3 で構文+名前解決             │
                                                        ▼
                                              Gate G (semantic_pass >= 50%)
                                                    │
                                              W4/W5/W6 並行拡大
                                                    │
                                              Gate H (semantic_pass >= 90%)
                                                    │
                                              プロジェクト主要目標達成
```
