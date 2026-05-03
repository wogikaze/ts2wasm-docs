# Test Status Taxonomy Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade test result classification from 4 vague categories (pass/fail/skip/error) to 6 granular categories (pass/mismatch/runtime_error/fail/unsupported/blocked) across all data layers and the web UI.

**Architecture:** Change flows from bottom up: coverage runner → test262 runner → web-ui data generator → web UI. Each layer adds or maps new status values while keeping old fields for backward compatibility.

**Tech Stack:** Python scripts (data pipeline), TypeScript/React (web UI)

---

## Task 1: reference-coverage.py — Add mismatch and runtime_error counters

**Files:**
- Modify: `scripts/run/reference-coverage.py`

- [ ] **Step 1: Initialize new counters after line 712**

Add `mismatch_count` and `runtime_error_count` initialization next to existing counters:

Currently around line 711-712:

```python
    build_pass_count = 0
    semantic_pass_count = 0
```

After:

```python
    build_pass_count = 0
    semantic_pass_count = 0
    mismatch_count = 0
    runtime_error_count = 0
```

- [ ] **Step 2: Expand semantic check logic (lines ~744-765)**

Replace the existing semantic check block to distinguish all cases:

```python
            if result.returncode == 0:
                build_pass_count += 1
                
                if semantic_enabled:
                    node_out = tmp_dir / "node.out"
                    wasm_out = tmp_dir / "wasm.out"
                    
                    node_result = subprocess.run(
                        ["timeout", "8s", "node", str(node_input)],
                        capture_output=True,
                        cwd=REPO_ROOT
                    )
                    wasm_result = subprocess.run(
                        ["timeout", "8s", "iwasm", str(out_wasm)],
                        capture_output=True,
                        cwd=REPO_ROOT
                    )
                    
                    if node_result.returncode != 0:
                        blocked_count += 1
                    elif wasm_result.returncode != 0:
                        runtime_error_count += 1
                    elif node_result.stdout == wasm_result.stdout:
                        semantic_pass_count += 1
                    else:
                        mismatch_count += 1
```

- [ ] **Step 3: Add new fields to the summary dict (after line 825)**

```python
        "mismatch": mismatch_count,
        "runtime_error": runtime_error_count,
```

- [ ] **Step 4: Add new fields to the key=value output (after line 855)**

```python
        print(f"mismatch={mismatch_count}")
        print(f"runtime_error={runtime_error_count}")
```

- [ ] **Step 5: Add new fields to the limit==0 summary block (around line 646)**

Add to the dry-run summary:

```python
            "mismatch": 0,
            "runtime_error": 0,
```

- [ ] **Step 6: Verify**

```bash
python3 -c "import ast; ast.parse(open('scripts/run/reference-coverage.py').read()); print('syntax OK')"
python3 scripts/run/reference-coverage.py test262 --limit 0 --no-web-ui 2>&1 | grep -E 'mismatch|runtime_error'
# Expected: mismatch=0, runtime_error=0
```

---

## Task 2: test262.py — Add mismatch and runtime_error statuses

**Files:**
- Modify: `scripts/run/reference-coverage.py` (was `scripts/run/test262.py`, now merged)

- [ ] **Step 1: Change process_one_test() — output mismatch from "fail" to "mismatch"**

In `process_one_test()` (around line 615-617), change:

```python
        elif node_ok:
            record = create_test_record("test262", str(test_file), "wasm-iwasm", "fail", expected, result_actual, "output mismatch", source_code=source_code, stderr=stderr_full)
            return record, "fail"
```

To:

```python
        elif node_ok:
            record = create_test_record("test262", str(test_file), "wasm-iwasm", "mismatch", expected, result_actual, "output mismatch", source_code=source_code, stderr=stderr_full)
            return record, "mismatch"
```

- [ ] **Step 2: Change compile_and_run_test() — runtime crash from "fail" to "runtime_error"**

In `compile_and_run_test()` (around line 557), change:

```python
    result_status = "pass" if metadata.expects_negative else "fail"
```

To:

```python
    result_status = "pass" if metadata.expects_negative else "runtime_error"
```

- [ ] **Step 3: Verify**

```bash
python3 -c "import ast; ast.parse(open('scripts/run/reference-coverage.py').read()); print('syntax OK')"
# Expected: syntax OK
```

---

## Task 3: web-ui-data.py — Update STATUS_MAP and summary calculation

**Files:**
- Modify: `scripts/gen/web-ui-data.py`

- [ ] **Step 1: Update STATUS_MAP to handle new status values**

Add new entries to `STATUS_MAP` (after the existing entries, before the closing `}`):

```python
    "mismatch": "mismatch",
    "runtime_error": "runtime_error",
```

- [ ] **Step 2: Change the summary structure in build_test_results()**

In `build_test_results()`, change the summary calculation to use the new 6 categories. Replace the `count_summary()` call and summary construction:

```python
def build_test_results(artifacts, jsonl_paths, generated_at, row_limit=1000):
    if jsonl_paths:
        all_tests = load_jsonl_test_records(jsonl_paths, 1)
        record_mode = "jsonl"
    else:
        all_tests = aggregate_test_records(artifacts)
        record_mode = "aggregate"

    shown_tests = all_tests[:row_limit]
    
    # Count by status
    passed = sum(1 for t in all_tests if t["status"] == "pass")
    mismatch = sum(1 for t in all_tests if t["status"] == "mismatch")
    runtime_error = sum(1 for t in all_tests if t["status"] == "runtime_error")
    build_error = sum(1 for t in all_tests if t["status"] == "fail")
    unsupported = sum(1 for t in all_tests if t["status"] == "unsupported")
    blocked = sum(1 for t in all_tests if t["status"] == "blocked")
    
    return {
        "tests": shown_tests,
        "summary": {
            "passed": passed,
            "mismatch": mismatch,
            "runtime_error": runtime_error,
            "build_error": build_error,
            "unsupported": unsupported,
            "blocked": blocked,
        },
        "metadata": {
            ...
```

Note: Keep the existing metadata structure unchanged.

- [ ] **Step 3: Verify**

```bash
python3 -c "import ast; ast.parse(open('scripts/gen/web-ui-data.py').read()); print('syntax OK')"
python3 scripts/gen/web-ui-data.py --coverage-dir artifacts/coverage/results --out-dir /tmp/web-ui-test 2>&1
cat /tmp/web-ui-test/test-results.json | python3 -m json.tool | head -30
# Expected: summary with passed/mismatch/runtime_error/build_error/unsupported/blocked
```

---

## Task 4: Web UI types.ts and useData.ts — Update TypeScript types and hooks

**Files:**
- Modify: `web-ui/src/types.ts`
- Modify: `web-ui/src/hooks/useData.ts`

- [ ] **Step 1: Update types.ts — add new status values to TestResult**

Change the `status` union type and add a summary interface:

```typescript
export type TestStatus = 'pass' | 'mismatch' | 'runtime_error' | 'fail' | 'unsupported' | 'blocked';

export interface TestResult {
  id: string;
  name: string;
  status: TestStatus;
  suite: string;
  duration?: number;
  error?: string;
  case?: string;
  target?: string;
  reason?: string;
  expected?: string;
  actual?: string;
  stderr?: string;
  source_code?: string;
  error_line?: number;
}
```

- [ ] **Step 2: Add TestSummary interface to types.ts**

```typescript
export interface TestSummary {
  passed: number;
  mismatch: number;
  runtime_error: number;
  build_error: number;
  unsupported: number;
  blocked: number;
}
```

- [ ] **Step 3: Update useData.ts — import and use TestSummary**

Change the summary state initialization:

```typescript
import type { TestResult, CoverageData, HistoricalData, TestResultsMetadata, TestSummary } from '../types'
// ...
const [summary, setSummary] = useState<TestSummary>({ passed: 0, mismatch: 0, runtime_error: 0, build_error: 0, unsupported: 0, blocked: 0 })
```

And update the data assignment in the `fetch` handler:

```typescript
setSummary(data.summary || { passed: 0, mismatch: 0, runtime_error: 0, build_error: 0, unsupported: 0, blocked: 0 })
```

- [ ] **Step 4: Verify TypeScript compilation**

```bash
cd web-ui && npx tsc --noEmit 2>&1 | head -20
# Expected: no type errors
```

---

## Task 5: App.tsx — Update summary cards, filter, and row rendering

**Files:**
- Modify: `web-ui/src/App.tsx`

- [ ] **Step 1: Update type imports and StatusFilter**

```typescript
import type { CoverageData, HistoricalData, TestResult, TestSummary } from './types'
// ...
type StatusFilter = 'all' | TestResult['status']
```

- [ ] **Step 2: Update getStatusIcon() — add cases for new statuses**

```typescript
  const getStatusIcon = (status: string) => {
    switch (status) {
      case 'pass': return <CheckCircle className="w-5 h-5 text-green-500" />
      case 'mismatch': return <XCircle className="w-5 h-5 text-orange-500" />
      case 'runtime_error': return <AlertTriangle className="w-5 h-5 text-red-500" />
      case 'fail': return <XCircle className="w-5 h-5 text-red-500" />
      case 'unsupported': return <SkipForward className="w-5 h-5 text-yellow-500" />
      case 'blocked': return <AlertCircle className="w-5 h-5 text-gray-500" />
      default: return <AlertCircle className="w-5 h-5 text-gray-500" />
    }
  }
```

- [ ] **Step 3: Update getStatusColor() — add cases for new statuses**

```typescript
  const getStatusColor = (status: string) => {
    switch (status) {
      case 'pass': return 'bg-green-500/10 text-green-500 border-green-500/20'
      case 'mismatch': return 'bg-orange-500/10 text-orange-500 border-orange-500/20'
      case 'runtime_error': return 'bg-red-500/10 text-red-500 border-red-500/20'
      case 'fail': return 'bg-red-700/10 text-red-400 border-red-700/20'
      case 'unsupported': return 'bg-yellow-500/10 text-yellow-500 border-yellow-500/20'
      case 'blocked': return 'bg-gray-500/10 text-gray-500 border-gray-500/20'
      default: return 'bg-gray-500/10 text-gray-500 border-gray-500/20'
    }
  }
```

- [ ] **Step 4: Update summary cards (6 cards replacing the 4-card grid)**

Change the summary cards section (lines 498-529):

```tsx
            {/* Summary Cards */}
            <div className="grid grid-cols-2 xl:grid-cols-6 gap-3 mb-5">
              <div className="bg-gray-800 rounded-lg p-4 border border-gray-700">
                <div className="flex items-center justify-between gap-3">
                  <span className="text-gray-400">Total</span>
                  <span className="text-2xl font-bold">{
                    summary.passed + summary.mismatch + summary.runtime_error + summary.build_error + summary.unsupported + summary.blocked
                  }</span>
                </div>
              </div>
              <div className="bg-gray-800 rounded-lg p-4 border border-gray-700">
                <div className="flex items-center justify-between gap-3">
                  <span className="text-gray-400">Passed</span>
                  <span className="text-2xl font-bold text-green-500">{summary.passed}</span>
                </div>
              </div>
              <div className="bg-gray-800 rounded-lg p-4 border border-gray-700">
                <div className="flex items-center justify-between gap-3">
                  <span className="text-gray-400">Mismatch</span>
                  <span className="text-2xl font-bold text-orange-500">{summary.mismatch}</span>
                </div>
              </div>
              <div className="bg-gray-800 rounded-lg p-4 border border-gray-700">
                <div className="flex items-center justify-between gap-3">
                  <span className="text-gray-400">Runtime</span>
                  <span className="text-2xl font-bold text-red-500">{summary.runtime_error}</span>
                </div>
              </div>
              <div className="bg-gray-800 rounded-lg p-4 border border-gray-700">
                <div className="flex items-center justify-between gap-3">
                  <span className="text-gray-400">Build Error</span>
                  <span className="text-2xl font-bold text-red-400">{summary.build_error}</span>
                </div>
              </div>
              <div className="bg-gray-800 rounded-lg p-4 border border-gray-700">
                <div className="flex items-center justify-between gap-3">
                  <span className="text-gray-400">Unsupported</span>
                  <span className="text-2xl font-bold text-yellow-500">{summary.unsupported}</span>
                </div>
              </div>
            </div>
```

Note: `blocked` is not shown as a separate card (it's typically 0 and would clutter the display). If needed, it can be shown in the metadata line.

- [ ] **Step 5: Update the filter dropdown to include new statuses**

```tsx
                  <select
                    value={statusFilter}
                    onChange={(e) => setStatusFilter(e.target.value as StatusFilter)}
                    className="min-w-44 px-4 py-2 bg-gray-800 border border-gray-700 rounded-lg focus:outline-none focus:ring-2 focus:ring-purple-500"
                  >
                    <option value="all">All Status</option>
                    <option value="pass">Pass</option>
                    <option value="mismatch">Mismatch</option>
                    <option value="runtime_error">Runtime Error</option>
                    <option value="fail">Build Error</option>
                    <option value="unsupported">Unsupported</option>
                    <option value="blocked">Blocked</option>
                  </select>
```

- [ ] **Step 6: Verify TypeScript compilation**

```bash
cd web-ui && npx tsc --noEmit 2>&1
# Expected: no errors
```

---

## Task 6: Full verification

- [ ] **Step 1: Run Python syntax checks on all modified scripts**

```bash
python3 -c "import ast; [ast.parse(open(f).read()) for f in ['scripts/run/reference-coverage.py', 'scripts/gen/web-ui-data.py']]; print('All syntax OK')"
```

- [ ] **Step 2: Run TypeScript check**

```bash
cd web-ui && npx tsc --noEmit 2>&1
```

- [ ] **Step 3: Run the existing project checks**

```bash
mise run check scripts
mise run check
```

- [ ] **Step 4: Run a small test262 dry-run to verify the pipeline**

```bash
# Generate coverage data with new statuses
python3 scripts/run/reference-coverage.py test262 --jsonl --sample 0 2>&1 | grep -E 'generated|Error'

# Check the generated web UI data has new summary fields
python3 -c "
import json
with open('web-ui/public/data/test-results.json') as f:
    d = json.load(f)
print('Summary keys:', list(d['summary'].keys()))
print('Sample test statuses:', list(set(t['status'] for t in d['tests'])))
"
```

- [ ] **Step 5: Run mise run gate-fast if feasible**

```bash
mise run gate-fast 2>&1 | tail -20
```
