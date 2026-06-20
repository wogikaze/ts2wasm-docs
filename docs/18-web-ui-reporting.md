# Web UI Reporting

## Components

| Path | Role |
|---|---|
| `web-ui/` | React/Vite dashboard app |
| `site/` | VitePress docs/site wrapper |
| `site/docs/dashboard/index.md` | static landing page for published dashboard |
| `scripts/coverage-dashboard-data` path | generates dashboard JSON data |
| `scripts/build-dashboard-site.py` | builds dashboard bundle |
| `scripts/sync-docs-site.py` | syncs docs/dashboard output to external docs repo when configured |

## Dashboard data

The React app reads:

- `data/test-results.json`
- `data/coverage.json`
- `data/history.json`

In dev or with `?live=1`, it polls with cache-busting query params. `?liveIntervalMs=<n>` controls polling with a minimum interval.

## Build commands

```bash
python scripts/manager.py coverage-dashboard-data
python scripts/build-dashboard-site.py
cd web-ui && npm run build:dashboard
cd site && npm run build
```

## Type contracts

Dashboard TypeScript contracts live in `web-ui/src/types.ts`. Schema changes in test/coverage JSON must update those types and this document.

## Do not duplicate source of truth

Dashboard pages visualize generated data. They are not canonical architecture docs. Link back to `docs/INDEX.md` for design decisions.
