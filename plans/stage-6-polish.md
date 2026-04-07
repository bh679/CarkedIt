# Stage 6: Polish, Analytics & Future-Proofing

## Overview

Final stage: creator analytics dashboard, card designer UX improvements, community moderation scaffolding, and AI image preparation. This stage turns the MVP into a polished feature set.

**Dependencies:** All previous stages (1-5)

---

## Feature 1: Creator Analytics Dashboard

### New Screen: `creator-dashboard`

Accessible from the card designer when viewing a published pack.

```
┌─────────────────────────────────┐
│     📊 Pack Analytics           │
│     Dark Humor Pack             │
│                                 │
│  ┌──────┐ ┌──────┐ ┌──────┐   │
│  │  47  │ │  15  │ │  32  │   │
│  │games │ │saves │ │players│   │
│  └──────┘ └──────┘ └──────┘   │
│                                 │
│  This Week: 8 games (+3)        │
│                                 │
│  ── Most Popular Cards ──────── │
│  1. "Eaten by a drop bear" - 23 wins │
│  2. "Death by vegemite" - 18 wins    │
│  3. "Swooped to death" - 12 wins     │
│                                 │
│  ── Deck Breakdown ──────────── │
│  Die: 4 cards (played 89 times) │
│  Live: 5 cards (played 67 times)│
│  Bye: 3 cards (played 45 times) │
└─────────────────────────────────┘
```

### API Endpoint

#### `GET /api/carkedit/packs/:id/analytics`

Requires auth (creator only).

**Response:**
```json
{
  "total_games": 47,
  "total_saves": 15,
  "unique_players": 32,
  "games_this_week": 8,
  "games_last_week": 5,
  "top_cards": [
    { "card_id": "card_uuid", "card_text": "...", "deck_type": "die", "play_count": 89, "win_count": 23, "win_rate": 0.258 }
  ],
  "deck_stats": {
    "die": { "card_count": 4, "total_plays": 89 },
    "living": { "card_count": 5, "total_plays": 67 },
    "bye": { "card_count": 3, "total_plays": 45 }
  }
}
```

### SQL Queries

```sql
-- Top cards by win rate (joins card_plays with expansion_cards)
SELECT
  ec.id, ec.card_text, ec.deck_type,
  COUNT(cp.id) AS play_count,
  SUM(CASE WHEN cp.is_winner = 1 THEN 1 ELSE 0 END) AS win_count,
  ROUND(CAST(SUM(CASE WHEN cp.is_winner = 1 THEN 1 ELSE 0 END) AS REAL) / COUNT(cp.id), 3) AS win_rate
FROM expansion_cards ec
LEFT JOIN card_plays cp ON cp.card_id = ec.id
WHERE ec.pack_id = ?
GROUP BY ec.id
ORDER BY win_count DESC
LIMIT 10;

-- Unique players (count distinct player_name across games that used this pack)
SELECT COUNT(DISTINCT gp.player_name) AS unique_players
FROM game_players gp
JOIN pack_usage pu ON pu.game_id = gp.game_id
WHERE pu.pack_id = ?;
```

---

## Feature 2: Card Designer UX Improvements

### Drag-and-Drop Reordering

Add drag handles to card rows in the pack editor. Use HTML5 Drag and Drop API (no library needed for vanilla JS).

```javascript
// In card-designer.js
function renderCardRow(card, index) {
  return `
    <div class="designer__card-row" draggable="true"
      data-card-id="${card.id}" data-sort="${card.sort_order}"
      ondragstart="window.game.cardDragStart(event, '${card.id}')"
      ondragover="window.game.cardDragOver(event)"
      ondrop="window.game.cardDrop(event, '${card.id}')">
      <span class="designer__drag-handle">⠿</span>
      <span class="designer__card-text">${card.card_text}</span>
      <button onclick="window.game.editCard(${JSON.stringify(card)})">✎</button>
      <button onclick="window.game.deleteCard('${card.id}')">✗</button>
    </div>
  `;
}
```

### Bulk Import

Add a "Bulk Import" button that opens a textarea for pasting multiple cards (one per line):

```
┌─────────────────────────────────┐
│  Bulk Import Cards              │
│                                 │
│  Deck: [Die ▼]                  │
│                                 │
│  Paste one card per line:       │
│  ┌───────────────────────────┐  │
│  │ Eaten by a drop bear      │  │
│  │ Death by vegemite         │  │
│  │ Swooped to death          │  │
│  │ Killed by a huntsman      │  │
│  └───────────────────────────┘  │
│  4 cards detected               │
│                                 │
│  [Cancel]  [Import All]         │
└─────────────────────────────────┘
```

### Duplicate Detection

When adding a card, check for similar text in the same pack:

```javascript
function checkDuplicates(newText, existingCards) {
  const normalized = newText.toLowerCase().trim();
  return existingCards.filter(c =>
    c.card_text.toLowerCase().trim() === normalized
  );
}
```

Show warning: "Similar card already exists: 'Eaten by a drop bear'. Add anyway?"

---

## Feature 3: Community Moderation Scaffolding

### Table: `content_flags`

```sql
CREATE TABLE IF NOT EXISTS content_flags (
  id TEXT PRIMARY KEY,
  reporter_id TEXT NOT NULL REFERENCES users(id),
  pack_id TEXT NOT NULL REFERENCES expansion_packs(id),
  card_id TEXT,
  reason TEXT NOT NULL CHECK(reason IN ('offensive', 'spam', 'copyright', 'other')),
  description TEXT,
  status TEXT NOT NULL DEFAULT 'pending' CHECK(status IN ('pending', 'reviewed', 'actioned', 'dismissed')),
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  reviewed_at TEXT,
  reviewed_by TEXT
);
CREATE INDEX IF NOT EXISTS idx_flags_pack ON content_flags(pack_id);
CREATE INDEX IF NOT EXISTS idx_flags_status ON content_flags(status);
```

### API Endpoints

```
POST /api/carkedit/packs/:id/flag
Body: { "reason": "offensive", "description": "Contains slurs", "card_id": "card_uuid" }
Requires auth.

GET /api/carkedit/admin/flags?status=pending&limit=50
Requires admin role (future).
```

### Auto-Hide Rule

If a pack receives 3+ flags from unique users with status `pending`, automatically set `visibility: 'hidden'` (new visibility value) until admin reviews.

```sql
-- Check flag count trigger (run after each new flag)
SELECT COUNT(DISTINCT reporter_id) AS flag_count
FROM content_flags
WHERE pack_id = ? AND status = 'pending';
-- If flag_count >= 3:
UPDATE expansion_packs SET visibility = 'hidden' WHERE id = ?;
```

### UI: Flag Button

Add to pack detail in marketplace:

```html
<button class="btn btn--small btn--muted" onclick="window.game.flagPack('${pack.id}')">
  Report
</button>
```

Flag modal:
```
┌─────────────────────────────────┐
│  Report Pack                    │
│                                 │
│  Reason:                        │
│  ○ Offensive content            │
│  ○ Spam                         │
│  ○ Copyright violation          │
│  ○ Other                        │
│                                 │
│  Details (optional):            │
│  [________________________]     │
│                                 │
│  [Cancel]  [Submit Report]      │
└─────────────────────────────────┘
```

---

## Feature 4: AI Image Preparation

### Schema Change

```sql
ALTER TABLE expansion_cards ADD COLUMN image_url TEXT;
ALTER TABLE expansion_cards ADD COLUMN image_status TEXT DEFAULT 'none'
  CHECK(image_status IN ('none', 'generating', 'ready', 'failed'));
```

### Stub API Endpoint

```
POST /api/carkedit/cards/:cardId/generate-image
Response: { "status": "queued", "estimated_seconds": 30 }

GET /api/carkedit/cards/:cardId/image
Response: { "status": "ready", "image_url": "https://..." }
        | { "status": "generating" }
        | { "status": "none" }
```

Implementation deferred — the endpoint returns `{ "status": "not_available", "message": "AI image generation coming soon" }` for now.

### Card Schema Update

```typescript
// In Card.ts
@type("string") imageUrl = "";  // Empty = no image
```

### Card Rendering Update

When `imageUrl` is set, show image on the card face. The existing `renderCard()` component needs a conditional:

```javascript
// In card.js
const imageHtml = card.imageUrl
  ? `<img class="card__image" src="${card.imageUrl}" alt="" />`
  : '';
```

---

## Feature 5: Pack Versioning

When editing a published pack, create a version snapshot so in-progress games aren't affected.

### Schema Change

```sql
CREATE TABLE IF NOT EXISTS pack_versions (
  id TEXT PRIMARY KEY,
  pack_id TEXT NOT NULL REFERENCES expansion_packs(id),
  version INTEGER NOT NULL,
  cards_json TEXT NOT NULL,  -- JSON snapshot of all cards at this version
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  UNIQUE(pack_id, version)
);
```

### Versioning Logic

1. Publishing a pack creates version 1 snapshot
2. Any edit to a published pack auto-increments version and creates new snapshot
3. `pack_usage` records the version used: `ALTER TABLE pack_usage ADD COLUMN pack_version INTEGER`
4. `getCardsByPackIds()` can optionally take a version parameter for replay/history

---

## File Changes Summary

### Modified Files

| File | Changes |
|------|---------|
| `carkedit-api/src/db/database.ts` | Add content_flags, pack_versions tables; ALTER TABLE for image columns |
| `carkedit-api/src/db/packs.ts` | Add analytics queries, flag operations, versioning logic |
| `carkedit-api/src/index.ts` | Register analytics, flag, image-stub endpoints |
| `carkedit-online/js/screens/card-designer.js` | Add drag-drop, bulk import, duplicate detection, analytics link |
| `carkedit-online/js/screens/marketplace.js` | Add flag button and modal |
| `carkedit-online/js/components/card.js` | Conditional image rendering |

### New Files

| File | Purpose |
|------|---------|
| `carkedit-online/js/screens/creator-dashboard.js` | Creator analytics screen |
| `carkedit-online/css/creator-dashboard.css` | Analytics dashboard styles |

---

## Testing

### Checklist

- [ ] Creator analytics shows correct game count, save count, unique players
- [ ] Top cards ranked by win count
- [ ] Drag-and-drop reorders cards and persists sort_order
- [ ] Bulk import parses one card per line, creates cards in batch
- [ ] Duplicate detection warns on exact text matches
- [ ] Flag modal submits report with reason
- [ ] Auto-hide triggers at 3 unique flags
- [ ] Image stub endpoint returns "not_available" message
- [ ] Pack versioning creates snapshot on publish
- [ ] Version increments on edit of published pack

---

## Risks

| Risk | Mitigation |
|------|------------|
| Analytics queries slow on large datasets | Add proper indexes; cache stats with 5-min TTL |
| Drag-drop browser compatibility | HTML5 DnD works in all modern browsers; test mobile touch |
| False-positive flag abuse | Require auth to flag; rate-limit to 1 flag per user per pack |
| AI image costs (future) | Rate-limit generation; cache generated images; defer to Stage 7+ |
| Pack versioning complexity | Keep it simple: JSON snapshot of cards, no diff-based versioning |
