# New Passing Tests Discord Notification

## Problem

After running test262 (or other suites), there is no way to see which specific tests newly started passing. The web UI shows aggregate counts, and coverage summaries show totals, but neither identifies individual tests that regressed or improved. The user wants automatic Discord notification when tests newly pass.

## Design

### 1. Baseline Storage

Each suite (test262, tsc, tsgo) stores a per-test status snapshot after each run:

**File:** `artifacts/coverage/baselines/{suite}-statuses.json`

```json
{
  "test262::reference/test262/test/language/arguments-object/10.5-1gs.js": "pass",
  "test262::reference/test262/test/built-ins/Array/from/mapfn.js": "unsupported",
  "_meta": {
    "updated_at": "2026-04-30T12:00:00Z",
    "total": 53445,
    "pass_count": 4667
  }
}
```

Key format: `{suite}::{case_path}` (case_path is the absolute path from the JSONL record).

### 2. Comparison Logic

After a JSONL-producing run completes:
1. Load previous baseline (if exists)
2. For each JSONL record:
   - If `(suite, case)` exists in baseline with status != "pass" AND new status == "pass" → newly passing
   - Update baseline dict with new status
3. If newly passing tests found → send Discord notification
4. Save updated baseline

### 3. Discord Notification

Sent via existing `DISCORD_WEBHOOK_URL` from `.env`.

Format:

```json
{
  "embeds": [{
    "title": "New Passing Tests",
    "color": 5814783,
    "fields": [
      {"name": "Suite", "value": "test262 | 前回 4,667 pass → 今回 4,723 pass (+56)", "inline": false},
      {"name": "Newly passing tests (56)", "value": "annexB/RegExp/hello.js\nbuilt-ins/Array/from/mapfn.js\n... and 52 more", "inline": false}
    ],
    "timestamp": "..."
  }]
}
```

- Max 20 test paths displayed, rest truncated with "... and N more"
- Test paths relative to `reference/test262/test/`
- Field value must stay under 2000 chars (Discord limit)
- If zero new passes: no notification sent

### 4. Integration Point

In `reference-coverage.py`, after JSONL writing completes:

```
main()
  → test loop (existing)
  → write JSONL (existing)  
  → refresh_web_ui_data (existing)
  → notify_new_passes(jsonl_path)  ← NEW
      ・load baseline
      ・compare JSONL records
      ・if new passes: send Discord
      ・save baseline
```

- Only fires when `--jsonl` is active (test262 only, since tsc/tsgo don't produce JSONL)
- If `DISCORD_WEBHOOK_URL` is not set: skip silently
- If Discord send fails: log warning, don't fail the run

### 5. File: `scripts/report/new-passes-notify.py`

A standalone function module (not CLI script):

```python
def notify_new_passes(jsonl_path: Path, baseline_dir: Path) -> None:
    """Compare JSONL against baseline and send Discord notification if new passes found."""
```

Called from reference-coverage.py after JSONL write completes.

### 6. Error Handling

| Scenario | Behavior |
|---|---|
| No baseline file exists (first run) | Save baseline, no notification |
| JSONL file empty or missing | Skip silently |
| `DISCORD_WEBHOOK_URL` not set | Skip silently |
| Discord API unreachable | Log warning to stderr |
| Baseline file corrupted | Delete and recreate (fresh start) |
| New passes = 0 | Skip notification (save baseline only) |

## Non-Goals

- Historical trend tracking (only single-step comparison: previous → current)
- Per-test failure tracking (only newly passing, not newly failing)
- Webhook URL configuration UI (remains in `.env`)
- Notification for non-JSONL runs (tsc/tsgo summary-only runs)
