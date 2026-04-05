# Stage 4: Lobby Pack Selection + Game Integration

## Overview

The host selects expansion packs in the game lobby. When the game starts, selected pack cards are merged with base-game decks. This is the critical integration point where user-created content enters the game flow.

**Dependencies:** Stage 1 (pack data in DB), Stage 2 (way to create packs)
**Parallel with:** Server deck-merging logic and client lobby UI can be built independently

---

## Architecture

### Current Flow (GameRoom.startGame)

```
startGame() → hardcoded arrays (DIE_CARDS, LIVING_CARDS, BYE_CARDS)
  → createDeck(cards, deckType) → Card schema objects
  → shuffle() → assign to state.dieDeck/livingDeck/byeDeck
```

### New Flow

```
startGame() → read selectedPackIds from game state
  → query DB: getCardsByPackIds(selectedPackIds)
  → merge expansion cards with base-game arrays
  → createDeck(mergedCards, deckType)
  → shuffle() → assign to state decks
```

### Card ID Strategy

- Base-game cards: integer IDs (1-48 for die, 1-68 for living/bye)
- Expansion cards: `card_<uuid>` string IDs
- The Card schema already uses `@type("string") id` — no schema change needed
- `card_plays` and `card_draws` tables use `TEXT` for `card_id` — compatible

---

## Server Changes

### New File: `src/utils/deck.ts`

```typescript
import { ExpansionCard } from '../db/types.js';
import { CardData } from '../data/cards.js';

/**
 * Convert expansion card DB rows to CardData format for createDeck().
 */
export function expansionCardsToCardData(cards: ExpansionCard[]): {
  die: CardData[];
  living: CardData[];
  bye: CardData[];
} {
  const result = { die: [], living: [], bye: [] };
  for (const card of cards) {
    result[card.deck_type].push({
      id: card.id,        // string ID like "card_uuid"
      text: card.card_text,
    });
  }
  return result;
}

/**
 * Merge base-game cards with expansion cards.
 */
export function mergeDecks(
  baseDie: CardData[], baseLiving: CardData[], baseBye: CardData[],
  expansion: { die: CardData[]; living: CardData[]; bye: CardData[] }
): { die: CardData[]; living: CardData[]; bye: CardData[] } {
  return {
    die: [...baseDie, ...expansion.die],
    living: [...baseLiving, ...expansion.living],
    bye: [...baseBye, ...expansion.bye],
  };
}
```

### Modified: `src/db/packs.ts`

Add batch query for game start:

```typescript
/**
 * Get all published cards from the given pack IDs.
 * Only returns cards from packs that are published (or private + owned by the requester).
 */
export function getCardsByPackIds(packIds: string[]): ExpansionCard[] {
  if (packIds.length === 0) return [];

  const placeholders = packIds.map(() => '?').join(',');
  return db.prepare(`
    SELECT ec.* FROM expansion_cards ec
    JOIN expansion_packs ep ON ec.pack_id = ep.id
    WHERE ep.id IN (${placeholders})
    AND (ep.status = 'published' OR ep.visibility = 'private')
  `).all(...packIds);
}
```

### Modified: `src/schema/GameState.ts`

Add selected pack IDs to synced game state:

```typescript
// After existing settings fields (~line 60)
@type(["string"])
selectedPackIds = new ArraySchema<string>();
```

### Modified: `src/rooms/GameRoom.ts`

#### Settings Message Handler

Add handler for pack selection (alongside existing `update_settings` handler):

```typescript
this.onMessage("select_packs", (client, data: { packIds: string[] }) => {
  if (client.sessionId !== this.state.hostId) return;
  if (this.state.phase !== "lobby") return;

  this.state.selectedPackIds.clear();
  for (const id of data.packIds) {
    this.state.selectedPackIds.push(id);
  }
});
```

#### startGame() Modification

```typescript
// In startGame(), before deck creation (~line 533)
// Existing:
// const shuffledDieDeck = shuffle(createDeck(DIE_CARDS, "die"));

// New:
import { getCardsByPackIds } from '../db/packs.js';
import { expansionCardsToCardData, mergeDecks } from '../utils/deck.js';

// Get expansion cards if packs selected
const selectedPackIds = Array.from(this.state.selectedPackIds);
const expansionCards = getCardsByPackIds(selectedPackIds);
const expansionDecks = expansionCardsToCardData(expansionCards);
const merged = mergeDecks(DIE_CARDS, LIVING_CARDS, BYE_CARDS, expansionDecks);

const shuffledDieDeck = shuffle(createDeck(merged.die, "die"));
const shuffledLivingDeck = shuffle(createDeck(merged.living, "living"));
const shuffledByeDeck = shuffle(createDeck(merged.bye, "bye"));
```

### CardData Interface Update

The `CardData` interface in `src/data/cards.ts` currently uses `id: number`. Expansion cards use string IDs. Update:

```typescript
// Before
interface CardData { id: number; text: string; special?: string; }

// After
interface CardData { id: number | string; text: string; special?: string; }
```

The `createDeck()` function in GameRoom already converts to string via the Card schema's `@type("string") id`, so `id.toString()` handles both.

---

## Client Changes

### New File: `js/components/pack-selector.js`

```javascript
/**
 * Renders pack selection UI for the lobby advanced settings panel.
 * Shows available packs with toggle switches.
 */
export function renderPackSelector(state) {
  const packs = state.availablePacks || [];
  const selected = state.selectedPackIds || [];

  if (packs.length === 0) {
    return `
      <div class="pack-selector">
        <div class="pack-selector__empty">
          No expansion packs available.
          <button class="btn btn--small" onclick="window.game.showCardDesigner()">
            Create One
          </button>
        </div>
      </div>
    `;
  }

  return `
    <div class="pack-selector">
      ${packs.map(pack => `
        <div class="pack-selector__item ${selected.includes(pack.id) ? 'pack-selector__item--selected' : ''}">
          <div class="pack-selector__info">
            <span class="pack-selector__title">${pack.title}</span>
            <span class="pack-selector__meta">${pack.card_count} cards</span>
          </div>
          <button class="btn btn--small ${selected.includes(pack.id) ? 'btn--active' : ''}"
            onclick="window.game.togglePack('${pack.id}')">
            ${selected.includes(pack.id) ? 'On' : 'Off'}
          </button>
        </div>
      `).join('')}
    </div>
  `;
}
```

### Modified: `js/screens/lobby.js`

Add expansion packs section in `renderAdvancedPanel()`:

```javascript
// After existing settings sections, before closing tags
// ~line 200 in renderAdvancedPanel()

// Expansion Packs section
${renderPackSelector(state)}
```

Add section header:

```html
<div class="settings-section">
  <h3 class="settings-section__title">Expansion Packs</h3>
  ${renderPackSelector(state)}
</div>
```

### Modified: `js/screens/online-lobby.js`

Mirror the pack selector for online mode. The host sees pack toggles; other players see a read-only list of selected packs.

### Modified: `js/state.js`

```javascript
// Add to state
availablePacks: [],        // Pack[] — user's packs + saved public packs
selectedPackIds: [],       // string[] — pack IDs selected for this game
```

### Modified: `js/router.js`

Add handlers to `window.game`:

```javascript
// Pack selection
togglePack: (packId) => {
  const current = getState().selectedPackIds;
  const updated = current.includes(packId)
    ? current.filter(id => id !== packId)
    : [...current, packId];
  setState({ selectedPackIds: updated });

  // If online, sync to server
  if (getState().gameMode === 'online') {
    client.send("select_packs", { packIds: updated });
  }
},

// Load available packs when entering lobby
loadAvailablePacks: async () => {
  const userId = localStorage.getItem('carkedit-user-id');
  if (!userId) return;
  const res = await fetch(`/api/carkedit/packs?creator_id=${userId}&status=published`);
  const data = await res.json();
  setState({ availablePacks: data.packs });
},
```

### Modified: `js/network/client.js`

Sync `selectedPackIds` from server state to local state:

```javascript
// In onStateChange or state listener
// When state.selectedPackIds changes, update local state
state.selectedPackIds.onChange(() => {
  setState({ selectedPackIds: Array.from(state.selectedPackIds) });
});
```

---

## Local Mode Support

For local (non-online) games, pack selection is stored in `state.selectedPackIds`. When starting a local game:

```javascript
// In game-manager.js or wherever local game starts
// Fetch expansion cards directly from API
async function getLocalGameDecks() {
  const packIds = getState().selectedPackIds;
  if (packIds.length === 0) return null; // Use default decks

  const res = await fetch(`/api/carkedit/packs/cards?packIds=${packIds.join(',')}`);
  const data = await res.json();
  return data; // { die: [...], living: [...], bye: [...] }
}
```

New API endpoint for local mode:

```
GET /api/carkedit/packs/cards?packIds=pack1,pack2,pack3
Response: { die: CardData[], living: CardData[], bye: CardData[] }
```

---

## Deck Info Display

Show the total card count per deck type in the lobby when packs are selected:

```
Base Game: 48 Die · 68 Live · 68 Bye
+ Dark Humor Pack: +5 Die · +3 Live · +4 Bye
────────────────────────────────
Total: 53 Die · 71 Live · 72 Bye
```

---

## File Changes Summary

### Modified Files

| File | Changes |
|------|---------|
| `carkedit-api/src/rooms/GameRoom.ts` | Modify `startGame()` to merge expansion cards; add `select_packs` message handler |
| `carkedit-api/src/schema/GameState.ts` | Add `selectedPackIds: ArraySchema<string>` |
| `carkedit-api/src/data/cards.ts` | Update `CardData.id` to `number \| string` |
| `carkedit-api/src/db/packs.ts` | Add `getCardsByPackIds()` batch query |
| `carkedit-api/src/index.ts` | Add `GET /api/carkedit/packs/cards` endpoint for local mode |
| `carkedit-online/js/screens/lobby.js` | Add pack selector section in `renderAdvancedPanel()` |
| `carkedit-online/js/screens/online-lobby.js` | Add pack selector for online mode |
| `carkedit-online/js/state.js` | Add `availablePacks`, `selectedPackIds` |
| `carkedit-online/js/router.js` | Add `togglePack()`, `loadAvailablePacks()` handlers |
| `carkedit-online/js/network/client.js` | Sync `selectedPackIds` from Colyseus state |

### New Files

| File | Purpose |
|------|---------|
| `carkedit-api/src/utils/deck.ts` | Expansion card conversion and deck merging utilities |
| `carkedit-online/js/components/pack-selector.js` | Pack toggle list component for lobby |
| `carkedit-online/css/pack-selector.css` | Pack selector styles |

---

## Testing

### Automated Tests

```bash
PORT=4500

# 1. Create a user and pack with cards
USER_ID=$(curl -s -X POST http://localhost:$PORT/api/carkedit/users \
  -H "Content-Type: application/json" \
  -d '{"display_name": "Test"}' | jq -r '.id')

PACK_ID=$(curl -s -X POST http://localhost:$PORT/api/carkedit/packs \
  -H "Content-Type: application/json" \
  -d "{\"creator_id\": \"$USER_ID\", \"title\": \"Test Pack\"}" | jq -r '.id')

curl -s -X POST http://localhost:$PORT/api/carkedit/packs/$PACK_ID/cards \
  -H "Content-Type: application/json" \
  -d '{"cards": [
    {"deck_type": "die", "card_text": "Expansion death 1"},
    {"deck_type": "die", "card_text": "Expansion death 2"},
    {"deck_type": "living", "card_text": "Expansion life 1"},
    {"deck_type": "bye", "card_text": "Expansion bye 1"}
  ]}'

# Publish the pack
curl -s -X PUT http://localhost:$PORT/api/carkedit/packs/$PACK_ID \
  -H "Content-Type: application/json" \
  -d '{"status": "published"}'

# 2. Verify cards endpoint
curl -s "http://localhost:$PORT/api/carkedit/packs/cards?packIds=$PACK_ID" | jq .
# Should return { die: [2 cards], living: [1 card], bye: [1 card] }

# 3. Start an online game with expansion pack selected
# (Use Colyseus client or Playwright to test full flow)
```

### Manual Testing Checklist

- [ ] Pack selector appears in lobby advanced settings
- [ ] Toggling packs updates the selected list
- [ ] Deck info display shows correct card counts
- [ ] Online mode: pack selection syncs to all players via Colyseus
- [ ] Game starts with merged decks (base + expansion cards)
- [ ] Expansion cards appear during gameplay with correct deck colors
- [ ] `card_plays` records expansion card IDs correctly
- [ ] `card_draws` records expansion card draws correctly
- [ ] Empty packs (0 cards in a deck type) don't break the game
- [ ] Very large packs (100+ cards) work without performance issues
- [ ] Local mode: pack selection works without Colyseus
- [ ] Selecting no packs = normal base-game behavior (regression test)

---

## Edge Cases

| Case | Handling |
|------|----------|
| Pack deleted between selection and game start | `getCardsByPackIds` returns whatever exists; missing pack = no cards from it |
| Pack has 0 cards for a deck type | Merged deck just has base cards; no error |
| Pack has only die cards, no live/bye | Only die deck gets extra cards; other decks unchanged |
| 500+ expansion cards selected | SQLite handles this fine; createDeck() just iterates a longer array |
| Pack selected but not published | Query filters to `status = 'published' OR visibility = 'private'` (creator's own packs) |

---

## Risks

| Risk | Mitigation |
|------|------------|
| Expansion cards unbalancing game | Show total deck sizes in lobby so host can judge |
| Synchronous DB query in startGame() | SQLite is fast for small queries; async not needed for <1000 cards |
| Pack selection not syncing in online mode | Colyseus ArraySchema handles sync; test with 2+ clients |
| CardData.id type change breaking existing code | Only the interface changes; runtime already handles both via toString() |
