# Step 3: Cloudflare API

## Context
Cloudflare provides CDN/security for carkedit.com. Their GraphQL Analytics API gives usage metrics (requests, bandwidth). Note: this is usage data, not official billing — costs shown as "estimated" based on plan tier.

## What This Step Delivers
- Backend endpoint to fetch Cloudflare usage data on-demand
- Cloudflare usage/cost data in the financial dashboard
- Usage metrics: HTTP requests, bandwidth, cache hit ratio

## Implementation

### Backend

**New env vars** (`.env.example`):
```
CLOUDFLARE_API_TOKEN=
CLOUDFLARE_ACCOUNT_ID=
CLOUDFLARE_ZONE_ID=          # for carkedit.com zone
CLOUDFLARE_PLAN_MONTHLY_USD= # e.g., 0 for free, 20 for Pro, 200 for Business
```

**New route** (`carkedit-api/src/index.ts`):

`POST /api/carkedit/costs/fetch/cloudflare` (requireAdmin)
- Calls Cloudflare GraphQL Analytics API at `https://api.cloudflare.com/client/v4/graphql`
- Queries: `httpRequests1dGroups` for request counts and bandwidth per day/month
- Aggregates into monthly totals
- Cost = `CLOUDFLARE_PLAN_MONTHLY_USD` (flat monthly fee, not usage-based for most plans)
- Response: `{ months: [{ month, plan_cost_usd, requests, bandwidth_gb, cache_hit_pct }], fetched_at }`

If `cost_entries` table exists, inserts with `source='api'`, `service='cloudflare'`.

### Frontend

**`financial-dashboard.js` changes:**
- Add "Cloudflare" section to dashboard grid
- "Fetch Latest" button
- Show monthly plan cost + usage metrics (requests, bandwidth, cache ratio)
- Mark as "Plan cost" not "usage cost" since most Cloudflare plans are flat-rate
- Integrate into main summary cards

### Prerequisites
- Cloudflare API token with `Zone:Analytics:Read` permission
- Account ID and Zone ID for carkedit.com

### Files Modified
| File | Change |
|------|--------|
| `carkedit-api/src/index.ts` | Add fetch route (~50 lines) |
| `carkedit-api/.env.example` | Add Cloudflare credentials |
| `carkedit-online/js/financial-dashboard.js` | Cloudflare section |
| `carkedit-online/css/financial-dashboard.css` | Cloudflare badge style |

### Verification
1. Set Cloudflare credentials in `.env`
2. `curl -X POST /api/carkedit/costs/fetch/cloudflare` — verify usage data returned
3. Dashboard shows Cloudflare plan cost and usage metrics
