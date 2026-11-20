# Ember Crypt — Event-Driven Game (EDP project)

A playable turn-based dungeon crawl whose **entire engine is a local event queue**. There is no
central game loop: pressing Attack publishes an event, handlers resolve it, and those handlers
publish the next events. You can watch the whole thing happen in the live queue panel next to the
arena.

- **Play** (`/`) — the arena, plus a live log of every event delivered
- **Event catalog** (`/events`) — all 16 events with producer, payload and subscribers
- **Architecture** (`/architecture`) — producers → queue → handlers, and the delivery guarantees

Written documentation in the repository:

| File | What it contains |
| --- | --- |
| [`events.md`](./events.md) | Every event, its producer, payload and subscribers |
| [`architecture.md`](./architecture.md) | The architecture with a diagram of producers, queue and handlers |
| `README.md` | This file — how to run the game |

---

## Quick start

Requirements: **Node.js 20+** and npm.

```bash
# 1. install dependencies
npm install

# 2. start the dev server
npm run dev
```

Open the URL printed in the terminal (by default **http://localhost:5173**) and press
**Enter the crypt**.

### Controls

| Action | Click | Key |
| --- | --- | --- |
| Attack | Attack button | `1` |
| Block (reduces the next hit to 35%) | Block button | `2` |
| Heal (uses a potion, +28 HP) | Heal button | `3` |
| Start / restart a run | Overlay button | `Enter` |

Waves get harder forever; the run ends when your health hits zero. Score comes from kills and loot,
and every third wave cleared unlocks a badge.

### Production build

```bash
npm run build
npm run preview
```

---

## What actually happens when you attack

```
PlayerActed ──> ResolveAction ──> EnemyDamaged ──> CloseTurn ──> TurnEnded ──> EnemyTurn ──> PlayerDamaged
```

If the enemy's health reaches zero the chain branches instead:

```
PlayerActed ──> ResolveAction ──> EnemyDefeated ─┬─> DropLoot ──> LootDropped ──> ApplyLoot ──> LootCollected
                                                 ├─> KeepScore
                                                 └─> ClearWave ──> WaveCleared ──> NextWave ──> EnemySpawned
```

Every box is a handler subscribed to a topic. Adding a rule means adding a subscriber, never editing
the input code.

---

## Project structure

```
.
├── README.md                     # this file
├── events.md                     # all events in the game
├── architecture.md               # architecture, producers, queue, handlers
└── src/
    ├── lib/
    │   ├── eventBus.js           # the local queue: publish, retry, dead-letter
    │   ├── game.js               # state, enemy table, every handler, the game factory
    │   ├── catalog.js            # event catalog used by the /events page
    │   └── useGame.js            # React binding between the queue and the UI
    ├── components/
    │   ├── SiteHeader.jsx
    │   ├── game/
    │   │   ├── Arena.jsx         # the battlefield
    │   │   ├── EnemySigil.jsx    # procedural enemy mark
    │   │   ├── HealthBar.jsx
    │   │   ├── ActionBar.jsx     # attack / block / heal
    │   │   └── RunOverlay.jsx    # start and game-over screens
    │   └── edp/
    │       ├── EventLog.jsx      # live delivery log
    │       └── Pipeline.jsx      # producers → queue → handlers diagram
    └── routes/
        ├── index.jsx             # the game
        ├── events.jsx            # event catalog page
        └── architecture.jsx      # architecture page
```

---

## Adding a new event

1. Document it in `events.md` and add it to `src/lib/catalog.js` so it shows on `/events`.
2. Publish it from a producer or another handler:
   `ctx.publish("MyEvent", { ... }, "My Producer")`.
3. Subscribe a handler in `src/lib/game.js`:

```js
{
  id: "my-handler",
  label: "MyHandler",
  topic: "MyEvent",              // or topics: ["A", "B"], or "*" for every event
  run: (state) => async (event, ctx) => {
    // change game state, then optionally publish the next event
    ctx.publish("NextEvent", { value: 1 }, "My Producer");
  },
}
```

Handlers that throw are retried automatically, so only throw for technical failures. Game outcomes
(no potions, full health) should be published as their own events — see `ActionRejected`.

---

## Tech

React 19 + TanStack Start, Tailwind CSS, lucide icons. The queue itself is plain JavaScript with no
dependencies — see `src/lib/eventBus.js`.
