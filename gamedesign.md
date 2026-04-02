# Game Design — Carked It! Online

## Design Layout
The game is designed for mobile first.
On larger screens, layout width is capped at the mobile breakpoint width to preserve the mobile layout.
The screens should be entirely within viewport, with no scrolling or zooming.

## Card Dimensions
Cards use the standard playing card aspect ratio (approximately 2.5:3.5 or 5:7).
All card rendering must maintain this ratio regardless of screen size.

## Card Rendering
Cards use a library of images to display different cards. Each card has:
- An illustration (top half)
- A title
- A description
- An optional discussion prompt

## Architecture
- Single HTML file — no page loads during the game
- All game state managed client-side in JavaScript
- Visually clean, functionally simple — no animations needed

## Game Screens
1. **Menu** — Title screen, start game
2. **Lobby** — Player setup, Funeral Director nomination
3. **Phase 1** — Die card draw and reveal
4. **Phase 2/3** — Same layout for both Live and Bye rounds (different card decks)
5. **Phase 4** — Eulogy Wildcard round and winner reveal

## Card Decks
- **Die deck** (Yellow) — 48 cards
- **Live deck** (Green) — 68 cards
- **Bye deck** (Blue) — 68 cards, includes 2 Wildcard Eulogy cards
