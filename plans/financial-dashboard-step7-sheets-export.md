# Step 7: Google Sheets Export

## Context
Export all cost data to a shareable Google Sheet for accounting, reporting, and sharing with non-admin stakeholders.

## What This Step Delivers
- "Export to Google Sheets" button on the dashboard
- Creates a formatted Google Spreadsheet with all cost data
- Returns a shareable link — opens in new tab

## Implementation

### Backend

**New dependency:** `googleapis`

**Reuses existing Firebase service account** (`carkedit-5cc8e`) — same GCP project. No new credentials needed.

**One-time setup:** Enable Google Sheets API at `console.cloud.google.com/apis/api/sheets.googleapis.com` for project `carkedit-5cc8e`.

**New route** (`carkedit-api/src/index.ts`):

`POST /api/carkedit/costs/export/sheets` (requireAdmin)
- Authenticates via Firebase service account JSON (already loaded at startup)
- Creates a new Google Spreadsheet: "CarkedIt Costs — YYYY-MM-DD"
- **Sheet 1 "Summary":**
  - Row 1: Headers (Service, All-Time Cost, This Month, Last Month, Count)
  - Rows: per-service breakdown (Image Gen, AWS, SiteGround, Cloudflare, etc.)
  - Final row: Grand Total
- **Sheet 2 "Monthly":**
  - Columns: Month, then one column per service
  - Rows: each month's cost per service
  - Final column: Monthly Total
- **Sheet 3 "Image Gen Detail":**
  - All generation_log entries: Date, Provider, Cost (actual or estimated), Pack, Card Text
- Shares the spreadsheet with the requesting user's email (from `req.localUser.email`) as editor
- Response: `{ spreadsheet_url, spreadsheet_id, title }`

### Frontend

**`financial-dashboard.js` changes:**
- "Export to Sheets" button in dashboard header (next to Refresh)
- Style: secondary button with spreadsheet icon
- On click:
  - Button shows loading state ("Exporting...")
  - POST to `/api/carkedit/costs/export/sheets`
  - On success: `window.open(spreadsheet_url, '_blank')`
  - On error: show error message inline

### Files Modified
| File | Change |
|------|--------|
| `carkedit-api/package.json` | Add `googleapis` |
| `carkedit-api/src/index.ts` | Add export route (~80 lines) |
| `carkedit-online/js/financial-dashboard.js` | Export button + handler |
| `carkedit-online/css/financial-dashboard.css` | Button loading state |

### Prerequisites
- Enable Google Sheets API in GCP Console (one-time)
- Firebase service account email (`firebase-adminsdk-fbsvc@carkedit-5cc8e.iam.gserviceaccount.com`) needs Sheets API access (enabled by default with API enabled)

### Verification
1. Click "Export to Sheets" button
2. Verify new Google Sheet opens in new tab
3. Check Sheet 1 (Summary), Sheet 2 (Monthly), Sheet 3 (Detail) have correct data
4. Verify requesting user has editor access
