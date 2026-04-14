# Step 6: GitHub Actions Billing API

## Context
CI/CD deploys run on GitHub Actions. Free tier covers 2000 Linux minutes/month but macOS runners cost 10x. The REST API gives exact usage data.

## What This Step Delivers
- Backend endpoint to fetch GitHub Actions usage on-demand
- Usage/cost displayed in the dashboard
- Minutes used vs free tier visualization

## Implementation

### Backend

Uses existing `GITHUB_TOKEN` env var (needs repo access).

**New route** (`carkedit-api/src/index.ts`):

`POST /api/carkedit/costs/fetch/github` (requireAdmin)
- Calls `GET https://api.github.com/user/billing/actions` (or `/orgs/{org}/billing/actions`)
- Response includes: `total_minutes_used`, `total_paid_minutes_used`, breakdown by OS
- Minutes are multiplied: Linux 1x, Windows 2x, macOS 10x
- Calculate cost: paid_minutes * $0.008/min (Linux rate)
- Inserts into `cost_entries` with `source='api'`, `service='github_actions'`
- Response: `{ total_minutes, paid_minutes, free_minutes_remaining, cost_usd, by_os: {...}, fetched_at }`

### Frontend

**`financial-dashboard.js` changes:**
- "GitHub Actions" section in dashboard
- "Fetch Latest" button
- Show: minutes used / 2000 free, paid minutes, cost
- Progress bar for free tier usage (green → yellow → red)
- Breakdown by OS (Linux, Windows, macOS)
- Integrate into main summary cards

### Files Modified
| File | Change |
|------|--------|
| `carkedit-api/src/index.ts` | Add fetch route (~30 lines) |
| `carkedit-online/js/financial-dashboard.js` | GitHub Actions section |
| `carkedit-online/css/financial-dashboard.css` | Progress bar style |

### Verification
1. `curl -X POST /api/carkedit/costs/fetch/github` — verify usage data returned
2. Dashboard shows minutes used and any paid costs
3. Progress bar shows free tier consumption
