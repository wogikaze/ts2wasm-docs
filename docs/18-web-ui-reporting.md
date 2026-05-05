# Web UI Reporting

The ts2wasm web UI is a static reporting interface for test results, reference
coverage, and historical run comparison. It is built from generated JSON data
under `site/docs/coverage/web-ui/public/data/` and can be served by the Vite development server or
by any static file host after `npm run build`.

## Data Contract

The UI reads these files from `site/docs/coverage/web-ui/public/data/`:

| File | Purpose |
|---|---|
| `test-results.json` | Test rows, status, duration, and error text for the Test Results tab. |
| `coverage.json` | Coverage totals, priority breakdowns, and suite-level coverage metrics. |
| `history.json` | Historical run totals, compile/runtime measurements, and run timestamps. |
| `metadata.json` | Generator metadata for the data snapshot. |

Generate the data before running or deploying the UI:

```bash
mise run coverage-dashboard-data
```

Reference and test262 runs can refresh the same files as part of a local run:

```bash
mise run reference-coverage -- test262 --limit 50 --dashboard-data
mise run test262 -- --sample 50 --dashboard-data
```

## Local Use

Start the documentation site and build the embedded dashboard bundle, then open `/dashboard`:

```bash
mise run coverage-dashboard-data
mise run build-dashboard-site
cd site
npm run dev
```

The development server serves the report at `http://localhost:5173/dashboard/` and
refreshes under `/dashboard/`.

## Local Live Mode

Static report mode reads the generated JSON files once. Vite development mode
enables local live mode automatically. Static builds can opt into the same
polling behavior with `live=1`:

```text
http://localhost:5173/?live=1
```

When live mode is active, the dashboard polls `test-results.json`,
`coverage.json`, and `history.json`, updates the views without a page reload,
and displays the live connection state plus the last successful test-results
refresh time. The default poll interval is two seconds. Local debugging can
override it with `liveIntervalMs`:

```text
http://localhost:5173/?live=1&liveIntervalMs=500
```

Live mode is intended for local development loops where a test runner or helper
command refreshes `site/docs/coverage/web-ui/public/data/test-results.json` during the run. Static
deployment continues to use the generated JSON snapshot without requiring a live
server.

## Static Deployment

Build the static bundle after generating fresh data:

```bash
mise run coverage-dashboard-data
cd site
npm run build
```

This generates static dashboard assets under `site/docs/public/dashboard/` and serves
them from `/dashboard/`.

## Views

The Test Results tab provides status filtering and search over individual test
records. The Coverage tab renders implementation status, priority mix, and
suite-level coverage charts. The History tab compares runs over time, including
pass/fail/skip deltas, regression flags, and compile/runtime trend charts.

## Export Controls

The header export controls operate on the active tab:

| Control | Output |
|---|---|
| `JSON` | Downloads the active tab payload as formatted JSON. |
| `CSV` | Downloads tabular rows for test results, coverage, or history. |
| `PDF` | Opens the browser print flow so the active view can be saved as PDF. |

JSON and CSV downloads are local browser downloads. The PDF path uses the
browser's print/save-as-PDF support and does not require a server-side renderer.

## Theme Preference

The header theme control switches between dark and light themes. The selected
theme is stored in browser `localStorage` under `ts2wasm-web-ui-theme` and is
restored on reload. If no stored preference exists, the UI follows the browser
`prefers-color-scheme` setting.

## Operational Checks

Use these checks when changing the UI, generator, or documentation:

```bash
cd site && npm run build
cargo fmt --all --check
mise run update-issue-index -- --check
mise run check issues
```
