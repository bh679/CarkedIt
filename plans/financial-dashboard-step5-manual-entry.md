# Step 5: Manual Cost Entry Form

## Context
SiteGround hosting, domains, and other fixed costs have no public API. Admins need a way to manually enter these recurring costs so the dashboard shows the full financial picture.

## What This Step Delivers
- CRUD API for manual cost entries (uses `cost_entries` table from Step 4)
- In-dashboard form for adding/editing/deleting manual costs
- Manual costs integrated into the summary view

## Implementation

### Backend

**New routes** (`carkedit-api/src/index.ts`, all `requireAdmin()`):

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/carkedit/costs/entries` | List entries. Params: `service`, `from`, `to`, `limit`, `offset` |
| POST | `/api/carkedit/costs/entries` | Create entry. Body: `{ service, category, description, amount_usd, period_start, period_end }` |
| PUT | `/api/carkedit/costs/entries/:id` | Update entry |
| DELETE | `/api/carkedit/costs/entries/:id` | Delete (only `source='manual'`) |

**Update `/api/carkedit/costs/summary`:**
- Include manual entries grouped by service + month alongside image gen and API-fetched costs
- Add `by_service` array with all services (image_gen, aws, siteground, cloudflare, etc.)

### Frontend

**`financial-dashboard.js` changes:**

New "Manual Costs" card in dashboard grid:
- Table of existing manual entries (description, amount, period, edit/delete buttons)
- "Add Cost" button opens inline form:
  - Service dropdown: SiteGround, AWS, Cloudflare, Domain, Other
  - Category: auto-populated from service, editable
  - Description: text input (e.g., "SiteGround GoGeek Monthly")
  - Amount USD: number input (step 0.01)
  - Period Start / End: date inputs
- Edit uses same form pre-populated
- Delete shows confirm dialog

**Summary updates:**
- Summary cards now show total across ALL services (image gen + manual + API)
- New "Service Breakdown" card replacing "Provider Breakdown" — shows all services
- Monthly chart includes all service types

### Files Modified
| File | Change |
|------|--------|
| `carkedit-api/src/index.ts` | 4 CRUD routes (~60 lines) |
| `carkedit-api/src/db/cost-entries.ts` | Add CRUD functions |
| `carkedit-online/js/financial-dashboard.js` | Entry list + form + summary merge |
| `carkedit-online/css/financial-dashboard.css` | Form styles, entry list styles |

### Verification
1. Create a manual entry via the form — verify it appears in the list
2. Edit the entry — verify changes persist
3. Delete the entry — verify removed
4. Summary cards show manual + image gen totals combined
