# Stage 2: Card Designer UI

## Overview

A new "Card Designer" screen accessible from the main menu. Users create expansion packs, add text cards to each deck type, and preview cards using existing card styling. Works without auth initially (private packs with anonymous user).

**Dependencies:** Stage 1 (API endpoints for pack/card CRUD)
**Parallel with:** UI layout can scaffold with mock data while Stage 1 API is built

---

## User Flow

```
Main Menu → [Card Designer] → Pack List
  → [Create New Pack] → Pack Editor
    → Edit title/description
    → [Add Card] → Card Form (pick deck type, enter text, preview)
    → Card List (edit/delete individual cards)
    → [Publish Pack] (requires auth later, disabled for now if anonymous)
  → [Edit Pack] → Pack Editor (same as above)
  → [Delete Pack] → Confirm dialog
```

---

## Screen: Card Designer (`card-designer`)

### Pack List View

Shows all packs belonging to the current user (anonymous or authenticated).

```
┌─────────────────────────────────┐
│       ← Back to Menu            │
│                                 │
│     🃏 Card Designer            │
│                                 │
│  ┌───────────────────────────┐  │
│  │ Dark Humor Pack           │  │
│  │ 12 cards · Draft          │  │
│  │ [Edit]  [Delete]          │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │ Aussie Deaths Pack        │  │
│  │ 8 cards · Published       │  │
│  │ [Edit]  [Delete]          │  │
│  └───────────────────────────┘  │
│                                 │
│  [+ Create New Pack]            │
│                                 │
└─────────────────────────────────┘
```

### Pack Editor View

Edit a single pack — metadata + card list.

```
┌─────────────────────────────────┐
│       ← Back to Packs           │
│                                 │
│  Pack Title: [Dark Humor Pack ] │
│  Description: [Cards for...   ] │
│                                 │
│  ── Die Cards (4) ──────────── │
│  │ "Eaten by a drop bear"  [✎][✗]│
│  │ "Death by vegemite"     [✎][✗]│
│  │ "Swooped to death"      [✎][✗]│
│  │ "Killed by a magpie"    [✎][✗]│
│                                 │
│  ── Live Cards (5) ──────────── │
│  │ "Try vegemite toast"    [✎][✗]│
│  │ ...                          │
│                                 │
│  ── Bye Cards (3) ──────────── │
│  │ ...                          │
│                                 │
│  [+ Add Card]                   │
│                                 │
│  [Save Draft]  [Publish]        │
└─────────────────────────────────┘
```

### Add/Edit Card Modal

```
┌─────────────────────────────────┐
│       Add New Card              │
│                                 │
│  Deck: [Die ▼] [Live] [Bye]    │
│                                 │
│  Card Text:                     │
│  ┌───────────────────────────┐  │
│  │ Eaten by a drop bear      │  │
│  └───────────────────────────┘  │
│                                 │
│  Preview:                       │
│  ┌─────────────┐               │
│  │   💀 DIE    │               │
│  │             │               │
│  │ Eaten by a  │               │
│  │ drop bear   │               │
│  │             │               │
│  └─────────────┘               │
│                                 │
│  [Cancel]  [Save Card]          │
└─────────────────────────────────┘
```

Card preview reuses the existing `renderCard()` component from `js/components/card.js`.

---

## State Additions

Add to `js/state.js` state object:

```javascript
// Card Designer state
designerView: 'list',          // 'list' | 'editor' | 'add-card'
myPacks: [],                   // ExpansionPack[] from API
currentPack: null,             // PackWithCards being edited
currentPackCards: [],           // Cards in current pack
editingCard: null,             // Card being edited (null = new card)
cardFormDeckType: 'die',       // Selected deck type in card form
cardFormText: '',              // Card text input value
designerLoading: false,        // Loading state for API calls
designerError: null,           // Error message
localUserId: null,             // Anonymous user ID (persisted in localStorage)
```

---

## File Changes

### Modified Files

| File | Changes |
|------|---------|
| `carkedit-online/js/router.js` | Add `'card-designer'` to SCREENS map (~line 41). Import `renderCardDesigner`. Add `window.game` handlers for designer actions (~line 402+) |
| `carkedit-online/js/screens/menu.js` | Add "Card Designer" button after existing buttons |
| `carkedit-online/js/state.js` | Add designer state fields listed above |
| `carkedit-online/index.html` | Add `<link>` for `card-designer.css` |

### New Files

| File | Purpose |
|------|---------|
| `carkedit-online/js/screens/card-designer.js` | Main render function — switches between list/editor/add-card views based on `state.designerView` |
| `carkedit-online/js/managers/pack-manager.js` | API client for pack/card CRUD: `fetchMyPacks()`, `createPack()`, `updatePack()`, `deletePack()`, `addCards()`, `updateCard()`, `deleteCard()` |
| `carkedit-online/css/card-designer.css` | Styles for pack list, pack editor, card form, card preview |

---

## Component Details

### `renderCardDesigner(state)` in `card-designer.js`

```javascript
export function renderCardDesigner(state) {
  switch (state.designerView) {
    case 'list':    return renderPackList(state);
    case 'editor':  return renderPackEditor(state);
    case 'add-card': return renderCardForm(state);
    default:        return renderPackList(state);
  }
}
```

### `pack-manager.js` API Client

```javascript
const API_BASE = `${window.location.origin}/api/carkedit`;

export async function fetchMyPacks(userId) {
  const res = await fetch(`${API_BASE}/packs?creator_id=${userId}`);
  return res.json();
}

export async function createPack(creatorId, title, description) { ... }
export async function getPack(packId) { ... }
export async function updatePack(packId, updates) { ... }
export async function deletePack(packId) { ... }
export async function addCards(packId, cards) { ... }
export async function updateCard(packId, cardId, updates) { ... }
export async function deleteCard(packId, cardId) { ... }
```

### Anonymous User Handling

On first visit to card designer, check `localStorage.getItem('carkedit-user-id')`. If null:
1. Call `POST /api/carkedit/users` with `display_name: "Anonymous Creator"`
2. Store returned `id` in localStorage

This creates a user record without auth. Stage 3 will link this to Firebase when user logs in.

---

## Router Handlers

Add to `window.game` in `router.js`:

```javascript
// Card Designer navigation
showCardDesigner: () => showScreen('card-designer', { designerView: 'list' }),
editPack: (packId) => { /* fetch pack, show editor */ },
newPack: () => { /* create pack via API, show editor */ },
deletePack: (packId) => { /* confirm + delete */ },

// Card form
showAddCard: () => showScreen('card-designer', { designerView: 'add-card', editingCard: null }),
editCard: (card) => showScreen('card-designer', { designerView: 'add-card', editingCard: card }),
saveCard: () => { /* POST or PUT card, return to editor */ },
deleteCard: (cardId) => { /* DELETE card, refresh editor */ },
setCardDeckType: (type) => setState({ cardFormDeckType: type }),
setCardText: (text) => setState({ cardFormText: text }),

// Pack actions
savePack: (updates) => { /* PUT pack metadata */ },
publishPack: (packId) => { /* PUT status: published, visibility: public */ },
```

---

## Menu Button

Add to `renderMenu()` in `menu.js`:

```html
<button class="btn btn--secondary" onclick="window.game.showCardDesigner()">
  Card Designer
</button>
```

Place after "Start Game" button, before version info.

---

## CSS Structure

```css
/* Pack list */
.designer__pack-list { ... }
.designer__pack-item { display: flex; justify-content: space-between; ... }
.designer__pack-meta { font-size: 0.85em; color: var(--text-muted); }

/* Pack editor */
.designer__editor { ... }
.designer__input { width: 100%; padding: 0.5em; ... }
.designer__section-header { font-weight: bold; border-bottom: 1px solid; ... }
.designer__card-row { display: flex; align-items: center; gap: 0.5em; ... }

/* Card form */
.designer__card-form { ... }
.designer__deck-picker { display: flex; gap: 0.5em; ... }
.designer__deck-btn--active { background: var(--accent); ... }
.designer__preview { margin: 1em auto; max-width: 200px; }

/* Reuse existing .card styles from card.css for preview */
```

---

## Testing

### Playwright E2E Script

```javascript
test('Card Designer - create pack and add card', async ({ page }) => {
  await page.goto('http://localhost:PORT');

  // Navigate to card designer
  await page.click('text=Card Designer');
  await expect(page.locator('.screen--card-designer')).toBeVisible();

  // Create a new pack
  await page.click('text=Create New Pack');
  await page.fill('[data-field="pack-title"]', 'Test Pack');
  await page.fill('[data-field="pack-description"]', 'A test pack');

  // Add a card
  await page.click('text=Add Card');
  await page.click('[data-deck="die"]');
  await page.fill('[data-field="card-text"]', 'Death by testing');
  await page.click('text=Save Card');

  // Verify card appears in list
  await expect(page.locator('text=Death by testing')).toBeVisible();

  // Verify preview renders
  await expect(page.locator('.designer__preview .card')).toBeVisible();
});
```

### Manual Testing Checklist

- [ ] Card Designer accessible from main menu
- [ ] Can create a new pack with title/description
- [ ] Can add cards of each deck type (die/living/bye)
- [ ] Card preview shows correct deck color and text
- [ ] Can edit existing card text and deck type
- [ ] Can delete individual cards
- [ ] Can delete entire pack with confirmation
- [ ] Pack list shows card count and status
- [ ] Anonymous user ID persisted in localStorage
- [ ] Back navigation works (editor → list → menu)
- [ ] Loading states shown during API calls
- [ ] Error states shown for failed API calls

---

## Risks

| Risk | Mitigation |
|------|------------|
| Card preview not matching in-game rendering | Reuse the exact `renderCard()` component from `card.js` |
| Lost work on accidental navigation | Auto-save pack metadata on input blur; confirm before delete |
| Anonymous user ID lost if localStorage cleared | Acceptable for MVP; auth in Stage 3 provides persistent identity |
| Long card text overflow | Add CSS text overflow handling + character limit (200 chars) |
