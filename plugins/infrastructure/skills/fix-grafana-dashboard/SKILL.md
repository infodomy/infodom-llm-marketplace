---
name: fix-grafana-dashboard
description: Modify a Grafana dashboard based on provided instructions — create or update panels, fix Prometheus queries, and reload provisioned dashboards via Playwright MCP and the Grafana REST API.
---

# fix-grafana-dashboard

Modify a Grafana dashboard based on the provided instructions.

**Instructions:** $ARGUMENTS

## How to proceed

1. **Identify the target dashboard** from the instructions. Dashboard JSON files are in `sre-tools/grafana/dashboards/`. Read the relevant file first.

2. **Use Playwright MCP** to access the live Grafana at https://grafana.baiji-wall.ts.net (credentials in `sre-tools/grafana/.env`).
   - Create a copy of the target dashboard first (`Save as copy`) — do all experimental changes there.
   - The copy lets you preview changes and confirm they work before touching the original.

3. **Make the requested changes** on the copy dashboard:
   - For new panels: prefer using the Grafana REST API (`curl POST /api/dashboards/db`) over Playwright UI interactions for reliability.
   - For Prometheus panels: use `increase(metric[5m])` for counters (cumulative count over window), `irate()` or `rate()` for per-second rates.
   - Use `$__rate_interval` carefully — if it returns no data on a new panel, fall back to a fixed window like `[5m]`.
   - Verify the panel shows correct data in the copy before proceeding.

4. **Ask for user confirmation** — show the user what changed and wait for approval before modifying the original.

5. **Apply to the original** once confirmed:
   - Update the JSON file in `sre-tools/grafana/dashboards/` directly (provisioned dashboards cannot be saved via the Grafana API).
   - Append new panels at the bottom: set `gridPos.y` to `max(y + h)` across all existing panels.
   - Use the next available panel `id` (max existing id + 1).
   - Preserve the exact panel JSON confirmed on the copy dashboard.

6. **Reload Grafana** if needed: reload provisioning via the Grafana API (`POST https://grafana.baiji-wall.ts.net/api/admin/provisioning/dashboards/reload`).

## Key conventions in this project

- Prometheus datasource uid: `prometheus-datasource`
- Namespace variable: `$namespace` (filter all queries with `namespace=~"$namespace"`)
- Standard timeseries panel unit for counts: `"unit": "short"`
- Standard timeseries panel unit for rates: `"unit": "reqps"`
- Plugin version: `"pluginVersion": "12.3.0"`
