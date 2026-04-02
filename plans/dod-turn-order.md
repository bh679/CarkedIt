# Plan: Day of the Dead Birthday Turn Order

## Context

Currently, turn order in CarkedItOnline is determined by randomly shuffling players when the game starts (`GameRoom.ts`). The birthday data (`birthMonth`, `birthDay`) is already captured and stored in the `Player` schema but is unused in game logic.

This change replaces the random shuffle with a deterministic "Day of the Dead" ordering rule:
- Players are kept in **join order** (the order they connected to the room)
- The sequence **rotates** to start at the player whose birthday is **closest** to Day of the Dead (November 1)
- Turn order is **recalculated on every join** so late joiners are incorporated correctly

## Repos Affected

- **carkedit-api** only — this is pure server-side game logic
- No frontend changes required

## Worktree Setup

```bash
cd /Users/brennanhatton/Projects/carkedit-api
git worktree add ../worktrees/carkedit-dod-turn-order -b dev/dod-turn-order
cd ../worktrees/carkedit-dod-turn-order
npm install
```

---

## Implementation

### 1. New utility: `src/utils/turnOrder.ts`

Create a pure utility function (no side effects, easily testable):

```typescript
import { MapSchema } from "@colyseus/schema";
import { Player } from "../schema/Player";

// Day of the Dead: November 1
const DOD_MONTH = 11;
const DOD_DAY = 1;

function dayOfYear(month: number, day: number): number {
  const daysInMonth = [0, 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];
  let d = 0;
  for (let m = 1; m < month; m++) d += daysInMonth[m];
  return d + day;
}

const DOD_DAY_OF_YEAR = dayOfYear(DOD_MONTH, DOD_DAY); // 305

function dodDistance(month: number, day: number): number {
  if (!month || !day) return Infinity; // Unknown birthday = goes last
  const bday = dayOfYear(month, day);
  const diff = Math.abs(bday - DOD_DAY_OF_YEAR);
  return Math.min(diff, 365 - diff);
}

// Returns player session IDs in join order, rotated to start at DOD-closest birthday
export function computeDodTurnOrder(players: MapSchema<Player>): string[] {
  const entries = Array.from(players.entries()); // preserves insertion (join) order

  let closestIndex = 0;
  let closestDistance = Infinity;

  entries.forEach(([, player], index) => {
    const dist = dodDistance(player.birthMonth, player.birthDay);
    if (dist < closestDistance) {
      closestDistance = dist;
      closestIndex = index;
    }
  });

  const ids = entries.map(([id]) => id);
  return [...ids.slice(closestIndex), ...ids.slice(0, closestIndex)];
}
```

### 2. Update `src/rooms/GameRoom.ts`

**In `onJoin`** — after adding the player, recompute and store turn order:

```typescript
import { computeDodTurnOrder } from "../utils/turnOrder";

onJoin(client: Client, options: any) {
  // ... existing player setup code (unchanged) ...
  this.state.players.set(client.sessionId, player);

  // Recalculate DOD turn order on every join
  const newOrder = computeDodTurnOrder(this.state.players);
  this.state.turnOrder.clear();  // or splice(0)
  newOrder.forEach((id) => this.state.turnOrder.push(id));

  console.log(`[GameRoom] ${player.name} joined (${client.sessionId})`);
}
```

**In `handleStartGame`** — remove the random shuffle and use the pre-computed `turnOrder`:

```typescript
// BEFORE (remove):
const playerKeys = Array.from(this.state.players.keys());
const shuffledOrder = shuffle(playerKeys);
shuffledOrder.forEach((id) => this.state.turnOrder.push(id));

// AFTER (replace with):
// turnOrder is already set and maintained via onJoin; recompute once at start
// to ensure final state is correct (handles edge case of joins during start)
const finalOrder = computeDodTurnOrder(this.state.players);
this.state.turnOrder.clear();
finalOrder.forEach((id) => this.state.turnOrder.push(id));
```

If `shuffle` is no longer used anywhere else, remove its import.

---

## Critical Files

| File | Change |
|---|---|
| `src/utils/turnOrder.ts` | **New** — pure `computeDodTurnOrder` function |
| `src/rooms/GameRoom.ts` | **Modify** — call `computeDodTurnOrder` in `onJoin` and `handleStartGame` |

---

## Tests

Add `src/utils/turnOrder.test.ts` covering:

1. **Basic rotation** — 4 players, player 3 is closest to DOD → order is [3,4,1,2]
2. **New joiner takes DOD spot** — 5th player joins with closest DOD birthday → [5,1,2,3,4]
3. **Unknown birthday** — player with `birthMonth=0` or `birthDay=0` goes last
4. **Tie** — two players equally close to DOD → first joiner wins (stable sort)
5. **All unknown** — all players lack birthdays → join order preserved, starts at index 0

---

## Verification

```bash
# Unit tests
cd /Users/brennanhatton/Projects/carkedit-api
npx jest src/utils/turnOrder.test.ts

# Integration: start server and join with multiple players via lobby
npm run dev  # port 2567
# Open carkedit-online and join 4 players with different birthdays
# Observe lobby turn order reflects DOD rotation
```

---

## No Deployment Impact

- No new env vars, dependencies, DB changes, or infrastructure changes
- Pure in-memory game logic change
- No wiki deployment doc update required
