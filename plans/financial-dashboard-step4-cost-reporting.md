# Step 4: Cost Reporting Endpoint (staging/local → production)

## Context
Image generation on staging (`brennan.games`) and local dev uses real FLUX/Leonardo API credits but costs are invisible to production's dashboard. This step creates a reporting mechanism so all environments POST their costs to production.

## What This Step Delivers
- `cost_entries` table for unified cost storage (also used by Steps 2, 3, 5, 6)
- Reporting endpoint on production that staging/local POST to after each generation
- Fire-and-forget hook in the image gen flow
- Environment breakdown in the dashboard (production vs staging vs local)

## Implementation

### Backend

**New table** (`carkedit-api/src/db/database.ts` — add to `initDatabase()`):
```sql
CREATE TABLE IF NOT EXISTS cost_entries (
  id TEXT PRIMARY KEY,
  service TEXT NOT NULL,
  category TEXT NOT NULL,
  description TEXT NOT NULL,
  amount_usd REAL NOT NULL,
  period_start TEXT NOT NULL,
  period_end TEXT NOT NULL,
  environment TEXT NOT NULL DEFAULT 'production',
  source TEXT NOT NULL DEFAULT 'manual',
  source_ref TEXT,
  entered_by TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_cost_entries_service ON cost_entries(service);
CREATE INDEX IF NOT EXISTS idx_cost_entries_period ON cost_entries(period_start, period_end);
```

**New module** (`carkedit-api/src/db/cost-entries.ts`):
- `createCostEntry(entry)` — insert + return
- `listCostEntries(opts)` — filter by service, date range, source
- `getCostSummaryByService(opts)` — aggregate by service + month

**New env vars:**
```
DEPLOY_ENV=production|staging|local
COST_REPORT_KEY=<shared-secret>
COST_REPORT_URL=https://api.carkedit.com/api/carkedit/costs/report
```

**New route** (`POST /api/carkedit/costs/report`):
- Auth: `X-Cost-Report-Key` header (not Firebase — server-to-server)
- Body: `{ environment, provider, cost_usd, tokens_used, description, timestamp }`
- Inserts into `cost_entries` with `source='remote_report'`, `service='image_gen'`
- Returns `{ ok: true }`

**Hook in image gen flow** (`index.ts` after `createGenerationLog()` ~lines 1219, 1357):
- If `DEPLOY_ENV !== 'production'` and `COST_REPORT_URL` is set:
  - Fire-and-forget `fetch(COST_REPORT_URL, { method: 'POST', headers: { 'X-Cost-Report-Key': key }, body: ... })`
  - `.catch(() => {})` — never fail the generation

**Update `/api/carkedit/costs/summary`:**
- Merge `cost_entries` (remote reports) with local `generation_log` data
- Add `by_environment` breakdown to response

### Frontend

**`financial-dashboard.js` changes:**
- Add environment breakdown under summary cards (e.g., "Production: $2.50, Staging: $0.80, Local: $0.41")
- Color-code environments in monthly chart

### Files Modified
| File | Change |
|------|--------|
| `carkedit-api/src/db/database.ts` | Add `cost_entries` table |
| `carkedit-api/src/db/cost-entries.ts` | **NEW** — CRUD + aggregation |
| `carkedit-api/src/index.ts` | Report endpoint + gen hook + summary update |
| `carkedit-api/.env.example` | Add `DEPLOY_ENV`, `COST_REPORT_KEY`, `COST_REPORT_URL` |
| `carkedit-online/js/financial-dashboard.js` | Environment breakdown display |

### Verification
1. Start server, verify `cost_entries` table created
2. `curl -X POST /costs/report -H "X-Cost-Report-Key: ..."` — verify entry created
3. Run image gen on a non-production server — verify report sent
4. Dashboard shows environment breakdown
