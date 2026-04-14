# Step 2: AWS Cost Explorer API

## Context
AWS hosts carkedit.com infrastructure. The AWS Cost Explorer API provides exact billing data filterable by service, tag, or account — no guesswork.

## What This Step Delivers
- Backend endpoint to fetch AWS costs on-demand
- AWS cost data displayed in the financial dashboard alongside image gen costs
- Persisted to `cost_entries` table (requires Step 4's table, or creates it here)

## Implementation

### Backend

**New dependency:** `@aws-sdk/client-cost-explorer`

**New env vars** (`.env.example`):
```
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_COST_REGION=us-east-1
```

**New route** (`carkedit-api/src/index.ts`):

`POST /api/carkedit/costs/fetch/aws` (requireAdmin)
- Uses `CostExplorerClient` + `GetCostAndUsageCommand`
- Fetches last N months of cost data (param: `months`, default 6)
- Filters for carkedit.com resources (by tag or account — needs user to confirm tagging strategy)
- Groups by SERVICE dimension for per-service breakdown (EC2, S3, Route53, etc.)
- Response: `{ months: [{ month, total_usd, services: { EC2: x, S3: y, ... } }], fetched_at }`

If `cost_entries` table exists (from Step 4), also inserts/upserts entries with `source='api'`, `service='aws'`.

### Frontend

**`financial-dashboard.js` changes:**
- Add "AWS" section to the dashboard grid
- "Fetch Latest" button that calls `POST /costs/fetch/aws`
- Show per-AWS-service breakdown (EC2, S3, Route53, etc.)
- Show last-fetched timestamp
- Integrate AWS totals into the main summary cards

### Prerequisites
- AWS IAM user/role with `ce:GetCostAndUsage` permission
- If filtering by tag: AWS resources must be tagged (e.g., `Project: carkedit`)
- If filtering by account: use linked account ID

### Files Modified
| File | Change |
|------|--------|
| `carkedit-api/package.json` | Add `@aws-sdk/client-cost-explorer` |
| `carkedit-api/src/index.ts` | Add fetch route (~40 lines) |
| `carkedit-api/.env.example` | Add AWS credentials placeholders |
| `carkedit-online/js/financial-dashboard.js` | AWS section + fetch button |
| `carkedit-online/css/financial-dashboard.css` | Service-specific styles |

### Verification
1. Set AWS credentials in `.env`
2. `curl -X POST /api/carkedit/costs/fetch/aws` — verify AWS cost data returned
3. Dashboard shows AWS costs alongside image gen costs
