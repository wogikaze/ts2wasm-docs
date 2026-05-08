# W6 (test262 coverage ramp) — remaining work

Last updated: 2026-05-08

## Current status

- **Current ramp**: limit=500, executed=500/53,445
- **build_pass=61** (0.11%), **semantic_pass=31** (0.06%)
- **unsupported=439**, **fail=0**, **blocked=1**
- **JSONL pipeline**: Working — generates `test262-results.jsonl`, HTML report, and Markdown report
- **Coverage matrix**: Generated in `artifacts/coverage/reference-coverage-matrix.md` via `mise run update-coverage-matrix`
- **Gate D**: Currently requires executed >= 100 (passes); new target will be >= 2,000

### Ramp trajectory

| Phase | Limit | Est. build_pass | Est. semantic_pass | Expected blockers |
|---|---|---|---|---|
| Current | 500 | 61 | 31 | UnsupportedSyntax 204, UnresolvedName 120, UnresolvedFunction 70 |
| Next | 2,000 | ~250 | ~120 | Same pattern, new feature categories appear |
| Medium | 10,000 | ~500-800 | ~300 | RegExp, Date, more String methods needed |
| Large | 30,000 | ~1,500-5,000 | ~1,000 | Promise, Proxy, full builtin coverage |
| Full | 53,445 | ~48,100 (90%) | ~48,100 (90%) | All semantic gaps resolved |

### triage infrastructure

- **`mise run reference-coverage -- test262 --limit N`**: Runs N test262 files, collects build_pass/semantic_pass/unsupported/blocked stats
- **JSONL records**: Each test record includes suite, case, status, reason, tracking, feature labels
- **Unsupported diagnostics**: Classified into diagnostic codes (UnsupportedSyntax, UnresolvedName, etc.) and feature labels (eval, regexp-literal, name-resolution, etc.)
- **`mise run update-coverage-matrix`**: Regenerates coverage matrix artifact and dashboard data
- **Gen-issues-from-coverage**: `mise run gen-issues-from-coverage -- --suite test262`
- **Differential JSONL**: `crates/cli/tests/differential_jsonl.rs` for differential test runner

## Remaining work

### Coverage ramp execution

- [ ] **Ramp from 500 → 2,000**: After W2/W3/W4 reduce unsupported count.
- [ ] **Ramp from 2,000 → 10,000**: After W4 builtin coverage expands.
- [ ] **Ramp from 10,000 → 30,000**: After Promise/Proxy/iterator protocol implementation.
- [ ] **Ramp from 30,000 → 53,445**: After full semantic coverage.
- [ ] **Run with `--detail`**: Generate feature breakdown at each ramp milestone.

### Triage automation

- [ ] **Auto-generate issues from new unsupported features**: After each ramp milestone, run `gen-issues-from-coverage` to create granular tracking issues for new unsupported categories.
- [ ] **Automatic regression detection**: Compare against previous ramp run; fail if build_pass or semantic_pass decreased.
- [ ] **Delta reporting**: For each ramp step, report which features changed from unsupported → pass.

### Dashboard improvements

- [ ] **Coverage dashboard trend graph**: Track semantic_pass% over time (across ramp milestones).
- [ ] **Feature-level burn-down chart**: Track unsupported count per feature category.
- [ ] **Workstream attribution**: Tag each coverage gain to the workstream (W2/W3/W4/W5) that enabled it.
- [ ] **Gate progress visualization**: Show current position relative to Gates D→E→G→H.

### Performance of ramp execution

- [ ] **Parallel test execution**: Currently runs each test262 file sequentially. Add parallel job support for faster ramp.
- [ ] **Caching build results**: Avoid re-building unchanged test262 files between runs.
- [ ] **Timeout handling**: Some test262 files may hang due to infinite loops or unhandled async.

### Dashboard data

- [ ] **Web UI data**: Ensure `mise run coverage-dashboard-data` outputs current data correctly.
- [ ] **Data freshness**: Automatic update on each coverage run.

## Key open issues

- 1 open issue in coverage area (from issue index)
- Meta-issue D: coverage matrix check

## Reference

- Coverage matrix: `artifacts/coverage/reference-coverage-matrix.md`
- Policy: `docs/15-coverage-matrix.md`
- Ramp command: `mise run reference-coverage -- test262 --limit N`
- Matrix update: `mise run update-coverage-matrix`
- Dashboard: `mise run coverage-dashboard-data`
- Issue generation: `mise run gen-issues-from-coverage -- --suite test262`
- Gate D: executed >= 2,000
- Gate G: semantic_pass >= 50%
- Gate H: semantic_pass >= 90%
