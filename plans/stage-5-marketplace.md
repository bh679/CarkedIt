# Stage 5: Marketplace / Browse Page

## Overview

A public page to browse, search, and preview published expansion packs. Includes basic analytics (usage count, save count) and "save to collection" for logged-in users.

**Dependencies:** Stage 1 (pack data), Stage 3 (auth for saving), Stage 4 (usage tracking from games)
**Parallel with:** UI scaffolding can begin before Stage 4 completes

---

## User Flow

```
Main Menu → [Browse Packs] → Marketplace
  → Search/filter packs
  → Click pack → Pack Detail
    → View cards
    → [Save to Collection] (requires auth)
    → [Use in Game] → Navigate to lobby with pack pre-selected
  → Sort by: Newest, Most Used, Most Saved
```

---

## Database Schema

### Table: `pack_saves`

```sql
CREATE TABLE IF NOT EXISTS pack_saves (
  user_id TEXT NOT NULL REFERENCES users(id),
  pack_id TEXT NOT NULL REFERENCES expansion_packs(id) ON DELETE CASCADE,
  saved_at TEXT NOT NULL DEFAULT (datetime('now')),
  PRIMARY KEY (user_id, pack_id)
);
CREATE INDEX IF NOT EXISTS idx_pack_saves_pack ON pack_saves(pack_id);
```

### Table: `pack_usage`

```sql
CREATE TABLE IF NOT EXISTS pack_usage (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  pack_id TEXT NOT NULL REFERENCES expansion_packs(id),
  game_id TEXT NOT NULL REFERENCES games(id),
  used_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_pack_usage_pack ON pack_usage(pack_id);
```

---

## API Endpoints

### Browse Packs

#### `GET /api/carkedit/packs/browse`

Returns paginated public published packs with aggregated stats.

**Query params:**
- `q` — search term (matches title, description, creator name)
- `sort` — `newest` (default) | `most_used` | `most_saved`
- `limit` — default 20, max 100
- `offset` — default 0

**Response (200):**
```json
{
  "packs": [
    {
      "id": "pack_uuid",
      "title": "Dark Humor Pack",
      "description": "Cards for those with...",
      "creator_name": "Player One",
      "creator_avatar": "https://...",
      "card_count": 12,
      "die_count": 4,
      "living_count": 5,
      "bye_count": 3,
      "usage_count": 47,
      "save_count": 15,
      "created_at": "2026-04-01T...",
      "is_saved": false
    }
  ],
  "total": 150,
  "limit": 20,
  "offset": 0
}
```

**SQL (core query):**
```sql
SELECT
  ep.*,
  u.display_name AS creator_name,
  u.avatar_url AS creator_avatar,
  COUNT(DISTINCT ec.id) AS card_count,
  SUM(CASE WHEN ec.deck_type = 'die' THEN 1 ELSE 0 END) AS die_count,
  SUM(CASE WHEN ec.deck_type = 'living' THEN 1 ELSE 0 END) AS living_count,
  SUM(CASE WHEN ec.deck_type = 'bye' THEN 1 ELSE 0 END) AS bye_count,
  COUNT(DISTINCT pu.id) AS usage_count,
  COUNT(DISTINCT ps.user_id) AS save_count
FROM expansion_packs ep
JOIN users u ON ep.creator_id = u.id
LEFT JOIN expansion_cards ec ON ec.pack_id = ep.id
LEFT JOIN pack_usage pu ON pu.pack_id = ep.id
LEFT JOIN pack_saves ps ON ps.pack_id = ep.id
WHERE ep.visibility = 'public' AND ep.status = 'published'
  AND (? IS NULL OR ep.title LIKE '%' || ? || '%' OR ep.description LIKE '%' || ? || '%' OR u.display_name LIKE '%' || ? || '%')
GROUP BY ep.id
ORDER BY
  CASE WHEN ? = 'most_used' THEN COUNT(DISTINCT pu.id) END DESC,
  CASE WHEN ? = 'most_saved' THEN COUNT(DISTINCT ps.user_id) END DESC,
  ep.created_at DESC
LIMIT ? OFFSET ?;
```

### Pack Stats

#### `GET /api/carkedit/packs/:id/stats`

**Response (200):**
```json
{
  "usage_count": 47,
  "save_count": 15,
  "games_this_week": 8,
  "unique_players": 32
}
```

### Save/Unsave Pack

#### `POST /api/carkedit/packs/:id/save`

Requires auth. Saves pack to user's collection.

**Response (201):** `{ "saved": true }`

#### `DELETE /api/carkedit/packs/:id/save`

Requires auth. Removes pack from collection.

**Response (200):** `{ "saved": false }`

### User's Saved Packs

#### `GET /api/carkedit/packs/saved`

Requires auth. Returns user's saved packs (same shape as browse response).

---

## Usage Tracking

### Modified: `src/rooms/GameRoom.ts`

Record pack usage when a game starts with expansion packs:

```typescript
// In startGame(), after successful deck merge
if (selectedPackIds.length > 0) {
  const gameId = this.state.gameId; // or generate if not yet assigned
  for (const packId of selectedPackIds) {
    recordPackUsage(packId, gameId);
  }
}
```

### In `src/db/packs.ts`:

```typescript
export function recordPackUsage(packId: string, gameId: string): void {
  db.prepare('INSERT INTO pack_usage (pack_id, game_id) VALUES (?, ?)').run(packId, gameId);
}
```

---

## Client: Marketplace Screen

### Screen Layout

```
┌─────────────────────────────────┐
│       ← Back to Menu            │
│                                 │
│     🏪 Expansion Packs          │
│                                 │
│  [Search packs...____________]  │
│                                 │
│  Sort: [Newest] [Popular] [Saved]│
│                                 │
│  ┌───────────────────────────┐  │
│  │ 🃏 Dark Humor Pack        │  │
│  │ by Player One             │  │
│  │ 12 cards · 47 games · ♥15 │  │
│  │ [View] [♥ Save] [Use]    │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │ 🃏 Aussie Deaths          │  │
│  │ by Mate                   │  │
│  │ 8 cards · 23 games · ♥8   │  │
│  │ [View] [♥ Save] [Use]    │  │
│  └───────────────────────────┘  │
│                                 │
│  [Load More]                    │
└─────────────────────────────────┘
```

### Pack Detail View

```
┌─────────────────────────────────┐
│       ← Back to Browse          │
│                                 │
│  🃏 Dark Humor Pack             │
│  by Player One                  │
│  47 games played · 15 saves     │
│                                 │
│  "Cards for those with a        │
│  twisted sense of humor"        │
│                                 │
│  ── Die Cards (4) ──────────── │
│  • Eaten by a drop bear         │
│  • Death by vegemite            │
│  • Swooped to death by magpie   │
│  • Killed by a huntsman spider  │
│                                 │
│  ── Live Cards (5) ──────────── │
│  • Try vegemite toast            │
│  • ...                           │
│                                 │
│  ── Bye Cards (3) ──────────── │
│  • ...                           │
│                                 │
│  [♥ Save to Collection]         │
│  [Use in Next Game]             │
└─────────────────────────────────┘
```

---

## File Changes

### Modified Files

| File | Changes |
|------|---------|
| `carkedit-api/src/db/database.ts` | Add `pack_saves` and `pack_usage` CREATE TABLE statements |
| `carkedit-api/src/db/packs.ts` | Add browse query, stats query, save/unsave, recordPackUsage |
| `carkedit-api/src/index.ts` | Register browse, stats, save/unsave endpoints |
| `carkedit-api/src/rooms/GameRoom.ts` | Record pack usage in startGame() |
| `carkedit-online/js/router.js` | Add 'marketplace' screen, pack detail handlers |
| `carkedit-online/js/screens/menu.js` | Add "Browse Packs" button |
| `carkedit-online/js/state.js` | Add marketplace state fields |

### New Files

| File | Purpose |
|------|---------|
| `carkedit-online/js/screens/marketplace.js` | Marketplace render function (browse + detail views) |
| `carkedit-online/js/components/pack-card.js` | Pack preview card component (reused in marketplace and lobby) |
| `carkedit-online/css/marketplace.css` | Marketplace styles |

### State Additions

```javascript
// Marketplace state
marketplaceView: 'browse',     // 'browse' | 'detail'
marketplacePacks: [],           // Pack[] with stats
marketplaceTotal: 0,            // Total results for pagination
marketplaceQuery: '',           // Search input
marketplaceSort: 'newest',     // 'newest' | 'most_used' | 'most_saved'
marketplaceOffset: 0,           // Pagination offset
marketplaceDetailPack: null,   // Pack detail being viewed
savedPackIds: [],               // string[] — user's saved pack IDs
marketplaceLoading: false,
```

---

## Testing

### curl Test Script

```bash
PORT=4500

# Browse public packs
curl -s "http://localhost:$PORT/api/carkedit/packs/browse" | jq .

# Search
curl -s "http://localhost:$PORT/api/carkedit/packs/browse?q=humor" | jq .

# Sort by most used
curl -s "http://localhost:$PORT/api/carkedit/packs/browse?sort=most_used" | jq .

# Pack stats
curl -s "http://localhost:$PORT/api/carkedit/packs/$PACK_ID/stats" | jq .

# Save pack (requires auth)
curl -s -X POST "http://localhost:$PORT/api/carkedit/packs/$PACK_ID/save" \
  -H "Authorization: Bearer $TOKEN" | jq .

# Unsave pack
curl -s -X DELETE "http://localhost:$PORT/api/carkedit/packs/$PACK_ID/save" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

### Manual Testing Checklist

- [ ] Marketplace accessible from main menu
- [ ] Browse shows public published packs with stats
- [ ] Search filters by title, description, and creator name
- [ ] Sort toggles work (newest, most used, most saved)
- [ ] Pack detail shows all cards grouped by deck
- [ ] "Save to Collection" works for logged-in users
- [ ] "Save to Collection" prompts sign-in for anonymous users
- [ ] "Use in Next Game" navigates to lobby with pack pre-selected
- [ ] Load More pagination works
- [ ] Usage count increments after playing a game with a pack
- [ ] Save count increments after saving a pack
- [ ] Empty state shown when no packs match search

---

## Risks

| Risk | Mitigation |
|------|------------|
| Browse query slow with many packs | Add composite index on (visibility, status, created_at); paginate |
| N+1 queries for stats | Use single aggregate query with JOINs |
| Search quality poor with LIKE | Acceptable for MVP; can add FTS5 later |
| Pack detail loads all cards (large packs) | Paginate cards if pack has 100+; unlikely for early packs |
