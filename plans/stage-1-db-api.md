# Stage 1: Database Schema + Expansion Pack CRUD API

## Overview

Add SQLite tables for users, expansion packs, and expansion cards. Expose REST endpoints for full CRUD. No UI — API-only, testable with curl.

**Dependencies:** None (first implementation stage)
**Parallel with:** Stage 2 UI can scaffold with mock data alongside this

---

## Database Schema

### Table: `users`

```sql
CREATE TABLE IF NOT EXISTS users (
  id TEXT PRIMARY KEY,
  firebase_uid TEXT UNIQUE,
  display_name TEXT NOT NULL,
  email TEXT,
  avatar_url TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_users_firebase_uid ON users(firebase_uid);
```

### Table: `expansion_packs`

```sql
CREATE TABLE IF NOT EXISTS expansion_packs (
  id TEXT PRIMARY KEY,
  creator_id TEXT NOT NULL REFERENCES users(id),
  title TEXT NOT NULL,
  description TEXT DEFAULT '',
  visibility TEXT NOT NULL DEFAULT 'private' CHECK(visibility IN ('private', 'public')),
  status TEXT NOT NULL DEFAULT 'draft' CHECK(status IN ('draft', 'published')),
  version INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_packs_creator ON expansion_packs(creator_id);
CREATE INDEX IF NOT EXISTS idx_packs_visibility_status ON expansion_packs(visibility, status);
```

### Table: `expansion_cards`

```sql
CREATE TABLE IF NOT EXISTS expansion_cards (
  id TEXT PRIMARY KEY,
  pack_id TEXT NOT NULL REFERENCES expansion_packs(id) ON DELETE CASCADE,
  deck_type TEXT NOT NULL CHECK(deck_type IN ('die', 'live', 'bye')),
  text TEXT NOT NULL,
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_cards_pack ON expansion_cards(pack_id);
```

---

## TypeScript Interfaces

Add to `carkedit-api/src/db/types.ts`:

```typescript
export interface User {
  id: string;
  firebase_uid: string | null;
  display_name: string;
  email: string | null;
  avatar_url: string | null;
  created_at: string;
  updated_at: string;
}

export interface ExpansionPack {
  id: string;
  creator_id: string;
  title: string;
  description: string;
  visibility: 'private' | 'public';
  status: 'draft' | 'published';
  version: number;
  created_at: string;
  updated_at: string;
}

export interface ExpansionCard {
  id: string;
  pack_id: string;
  deck_type: 'die' | 'live' | 'bye';
  text: string;
  sort_order: number;
  created_at: string;
  updated_at: string;
}

export interface PackWithCards extends ExpansionPack {
  cards: ExpansionCard[];
}
```

---

## API Endpoints

### Users

#### `POST /api/carkedit/users`

Create or upsert a user. Used by client after Firebase auth or to create anonymous user.

**Request:**
```json
{
  "display_name": "Player One",
  "firebase_uid": "abc123",  // optional, null for anonymous
  "email": "player@example.com",  // optional
  "avatar_url": "https://..."  // optional
}
```

**Response (201):**
```json
{
  "id": "usr_uuid",
  "display_name": "Player One",
  "firebase_uid": "abc123",
  "email": "player@example.com",
  "avatar_url": null,
  "created_at": "2026-04-05T00:00:00.000Z",
  "updated_at": "2026-04-05T00:00:00.000Z"
}
```

**Upsert logic:** If `firebase_uid` is provided and exists, update `display_name`, `email`, `avatar_url`, `updated_at`. Otherwise insert new row.

#### `GET /api/carkedit/users/:id`

**Response (200):** User object (same shape as POST response)
**Response (404):** `{ "error": "User not found" }`

---

### Expansion Packs

#### `POST /api/carkedit/packs`

**Request:**
```json
{
  "creator_id": "usr_uuid",
  "title": "Dark Humor Pack",
  "description": "Cards for those with a twisted sense of humor"
}
```

**Response (201):**
```json
{
  "id": "pack_uuid",
  "creator_id": "usr_uuid",
  "title": "Dark Humor Pack",
  "description": "Cards for those with a twisted sense of humor",
  "visibility": "private",
  "status": "draft",
  "version": 1,
  "created_at": "...",
  "updated_at": "..."
}
```

#### `GET /api/carkedit/packs`

**Query params:**
- `creator_id` — filter by creator (required for "my packs")
- `visibility` — `private` | `public`
- `status` — `draft` | `published`
- `limit` — default 50
- `offset` — default 0

**Response (200):**
```json
{
  "packs": [ { ...pack, card_count: 12 } ],
  "total": 25
}
```

#### `GET /api/carkedit/packs/:id`

Returns pack with all cards.

**Response (200):**
```json
{
  ...pack,
  "cards": [
    { "id": "card_uuid", "deck_type": "die", "text": "...", "sort_order": 0, ... }
  ]
}
```

#### `PUT /api/carkedit/packs/:id`

**Request (partial update):**
```json
{
  "title": "Updated Title",
  "description": "Updated description",
  "visibility": "public",
  "status": "published"
}
```

**Validation:** Cannot set `visibility: "public"` or `status: "published"` unless pack has at least 1 card.

**Response (200):** Updated pack object

#### `DELETE /api/carkedit/packs/:id`

**Response (204):** No content (cascades to delete all cards)

---

### Expansion Cards

#### `POST /api/carkedit/packs/:id/cards`

Supports single or batch card creation.

**Request:**
```json
{
  "cards": [
    { "deck_type": "die", "text": "You died from excessive laughter" },
    { "deck_type": "live", "text": "Learn to juggle chainsaws" }
  ]
}
```

**Response (201):**
```json
{
  "cards": [
    { "id": "card_uuid1", "pack_id": "pack_uuid", "deck_type": "die", "text": "...", "sort_order": 0, ... },
    { "id": "card_uuid2", ... }
  ]
}
```

**sort_order:** Auto-assigned based on current max sort_order + 1 for each card.

#### `PUT /api/carkedit/packs/:id/cards/:cardId`

**Request:**
```json
{
  "text": "Updated card text",
  "deck_type": "bye",
  "sort_order": 3
}
```

**Response (200):** Updated card object

#### `DELETE /api/carkedit/packs/:id/cards/:cardId`

**Response (204):** No content

---

## File Changes

### Modified Files

| File | Changes |
|------|---------|
| `carkedit-api/src/db/database.ts` | Add 3 CREATE TABLE + 3 CREATE INDEX statements in `initDatabase()` after existing tables (line ~114) |
| `carkedit-api/src/db/types.ts` | Add User, ExpansionPack, ExpansionCard, PackWithCards interfaces |
| `carkedit-api/src/index.ts` | Register 10 new route handlers after existing routes (line ~228) |

### New Files

| File | Purpose |
|------|---------|
| `carkedit-api/src/db/users.ts` | User DB operations: createUser, upsertUserByFirebaseUid, getUserById |
| `carkedit-api/src/db/packs.ts` | Pack/card DB operations: CRUD for packs and cards, list queries with filters |

---

## ID Generation

Use `crypto.randomUUID()` with prefixes for readability:
- Users: `usr_<uuid>`
- Packs: `pack_<uuid>`
- Cards: `card_<uuid>`

This matches the existing pattern in `database.ts` where `issue_reports` uses TEXT primary keys.

---

## Testing

### curl Test Script

```bash
PORT=4500  # or current session port

# Create anonymous user
curl -s -X POST http://localhost:$PORT/api/carkedit/users \
  -H "Content-Type: application/json" \
  -d '{"display_name": "Test User"}' | jq .

# Create pack
USER_ID="<from above>"
curl -s -X POST http://localhost:$PORT/api/carkedit/packs \
  -H "Content-Type: application/json" \
  -d "{\"creator_id\": \"$USER_ID\", \"title\": \"Test Pack\"}" | jq .

# Add cards
PACK_ID="<from above>"
curl -s -X POST http://localhost:$PORT/api/carkedit/packs/$PACK_ID/cards \
  -H "Content-Type: application/json" \
  -d '{"cards": [{"deck_type": "die", "text": "Test death"}, {"deck_type": "live", "text": "Test life"}]}' | jq .

# Get pack with cards
curl -s http://localhost:$PORT/api/carkedit/packs/$PACK_ID | jq .

# List my packs
curl -s "http://localhost:$PORT/api/carkedit/packs?creator_id=$USER_ID" | jq .

# Update pack
curl -s -X PUT http://localhost:$PORT/api/carkedit/packs/$PACK_ID \
  -H "Content-Type: application/json" \
  -d '{"title": "Updated Pack", "visibility": "public", "status": "published"}' | jq .

# Delete card
CARD_ID="<from above>"
curl -s -X DELETE http://localhost:$PORT/api/carkedit/packs/$PACK_ID/cards/$CARD_ID

# Delete pack (cascade)
curl -s -X DELETE http://localhost:$PORT/api/carkedit/packs/$PACK_ID

# Verify cards deleted
curl -s http://localhost:$PORT/api/carkedit/packs/$PACK_ID | jq .  # should 404
```

### Verification Checklist

- [ ] All 3 tables created on server start
- [ ] User CRUD works
- [ ] Pack CRUD works with proper validation
- [ ] Card batch create works
- [ ] FK cascade: deleting pack deletes all its cards
- [ ] Cannot publish pack with 0 cards
- [ ] List packs supports filtering by creator_id, visibility, status
- [ ] Proper 404s for missing resources

---

## Risks

| Risk | Mitigation |
|------|------------|
| UUID collisions | Extremely unlikely with crypto.randomUUID(); add UNIQUE constraints |
| Large batch card inserts | Use transaction for batch creates |
| No auth yet | Stage 1 is API-only; auth middleware added in Stage 3 |
