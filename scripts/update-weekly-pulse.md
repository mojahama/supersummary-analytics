# Weekly Pulse Update Playbook

Run every Wednesday. Refreshes `dashboard/data/weekly-pulse.json` with cross-platform CL data and pushes to master so GitHub Pages redeploys.

## Inputs you need
- GSC MCP (`mcp__gsc-mcp__*`) — site `sc-domain:supersummary.com`
- GA4 MCP (`mcp__analytics-mcp__*`) — property `272381877`
- Amplitude MCP — project `284674` (Production)

## Step 1 — Pull last 8 full weeks of Amplitude data

Call `mcp__Amplitude__query_dataset` with this definition:

```json
{
  "type": "eventsSegmentation",
  "app": "284674",
  "name": "CL Weekly Pulse",
  "params": {
    "range": "Last 60 Days",
    "events": [
      {"event_type": "View Guide Page - Character List", "filters": [], "group_by": []},
      {"event_type": "View Guide Page - Character",      "filters": [], "group_by": []},
      {"event_type": "View Guide Page - Summary",        "filters": [], "group_by": []}
    ],
    "metric": "totals",
    "countGroup": "User",
    "groupBy": [],
    "interval": 1,
    "segments": [{"conditions": []}]
  }
}
```

Pass `excludeIncompleteDatapoints: true`. Aggregate daily values into ISO weeks (Mon–Sun, labeled by week-ending Sunday). Keep the last 8 complete weeks.

## Step 2 — Pull GSC totals for CL pages (last 28d)

`mcp__gsc-mcp__get_search_analytics` with `dimensions: page`, `days: 28`, `row_limit: 500`. Filter rows whose page URL contains `/character-list/`. Sum the clicks → `gscCLClicks` for the latest week's row.

For the top-10 list (`topPages.gsc28d`), sort the filtered CL pages by clicks desc and take 10 with `{page, clicks, impressions, ctr, position}`. Strip the `https://www.supersummary.com/` prefix and the trailing slash.

## Step 3 — Pull GA4 sessions for CL pages (last 7d, current week)

`mcp__analytics-mcp__run_report`:
- `property_id: 272381877`
- `date_ranges`: latest complete Mon–Sun (e.g. `[{start_date, end_date}]`)
- `dimensions: ["pagePath"]`
- `metrics: ["sessions"]` (pass as plain strings, NOT `{name: "..."}`)
- `dimension_filter`: pagePath CONTAINS `/character-list`
- Sum the `sessions` values → `ga4CLSessions`

## Step 4 — Compute KPIs

From the daily Amplitude series:
- `clViewsPerDay` = avg of last 7 complete days
- `clViewsPerDayPriorWeek` = avg of the 7 days before that
- `wowPct` = (current − prior) / prior × 100
- `vs4WeeksAgoMultiplier` = current / (avg of 7 days, 4 weeks earlier)
- `clCaRatio` = sum(CL last 7d) / sum(CA last 7d)
- `summaryViewsPerDay` = avg Summary last 7 days
- `clClicks28d` = sum from Step 2
- `clClicks28dPriorPeriod` = same query for the prior 28 days (use `mcp__gsc-mcp__compare_search_periods` with `period1_start/end` and `period2_start/end` set explicitly)

## Step 5 — Identify newest top CL page

In the GSC top-10, flag a page whose position is < 3.5 AND wasn't in last week's JSON. Set `kpis.topNewCLPage` to `{slug, clicks28d, ctr, position}`. If none qualifies, keep prior week's value.

## Step 6 — Rewrite the JSON

Read `dashboard/data/weekly-pulse.json`, then write a new version:
- `meta.lastUpdated` = today (YYYY-MM-DD)
- `meta.lastUpdatedBy` = `"scheduled-agent"`
- `kpis.*` = computed values
- `weekly` = append the new week-ending row, drop the oldest if the array is longer than 12 weeks
- `topPages.gsc28d` = full new top-10

Preserve any existing rows in `weekly[]` for older weeks — don't recompute history.

## Step 7 — Commit and push

```bash
git add dashboard/data/weekly-pulse.json
git commit -m "Weekly pulse: refresh CL metrics ($(date +%Y-%m-%d))"
git push origin master
```

GitHub Pages auto-deploys within ~1 minute. Verify at https://mojahama.github.io/supersummary-analytics/.

## Step 8 — Slack-style summary

Post a 5-bullet recap (CL views/day + WoW%, CL/CA ratio, top GSC mover, summary verdict on cannibalization, new entrant if any). Keep it tight.

## Guardrails
- Never overwrite older rows in `weekly[]` — only append.
- If any data source fails, write a partial update with the working sources and leave the missing fields `null`. Note the failure in the commit message.
- If WoW swings >25%, double-check the math before committing — likely a window-alignment bug.
- Convert any relative dates in user requests to absolute dates before querying.
