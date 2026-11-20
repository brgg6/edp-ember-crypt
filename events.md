# Events

Every message the Ember Crypt game puts on the event queue.

A **producer** publishes an event. The queue creates one **delivery** for each subscribed
**handler**. Handlers change game state and may publish further events, which is how a turn
progresses.

---

## Event summary

| Event | Producer | Subscribers | Meaning |
| --- | --- | --- | --- |
| `GameStarted` | Player Input | `OpenCrypt` | The run begins; the first enemy is spawned. |
| `PlayerActed` | Player Input | `ResolveAction` | The player chose attack, block or heal. |
| `EnemyDamaged` | Combat System | `CloseTurn` | The hit landed but the enemy survived. |
| `EnemyDefeated` | Combat System | `DropLoot`, `KeepScore`, `ClearWave` | Enemy health reached zero. |
| `PlayerBlocked` | Combat System | `CloseTurn` | The player is guarding this turn. |
| `PlayerHealed` | Combat System | `CloseTurn` | A potion was consumed. |
| `ActionRejected` | Combat System | `CloseTurn` | The action was not possible (no potions, full health). |
| `TurnEnded` | Turn Manager | `EnemyTurn` | The player's turn is over. |
| `PlayerDamaged` | Enemy AI | — (UI reads state) | The enemy struck back. |
| `GameOver` | Enemy AI | `EndRun`, `LeaderboardSync` | Player health reached zero. |
| `LootDropped` | Loot System | `ApplyLoot` | A defeated enemy dropped an item. |
| `LootCollected` | Loot System | `KeepScore` | The item was applied and scored. |
| `WaveCleared` | Combat System | `NextWave`, `KeepScore` | The wave is complete. |
| `EnemySpawned` | Spawn System | — (UI reads state) | A new enemy entered the arena. |
| `AchievementUnlocked` | Score Keeper | `AwardBadge` | Every third wave cleared grants a badge. |
| `DeliveryFailed` | Event Queue | `Telemetry` | System event: a delivery exhausted its retries. |

---

## Payloads

### `GameStarted`

```json
{}
```

### `PlayerActed`

```json
{ "action": "attack" }
```

`action` is `"attack"`, `"block"` or `"heal"`.

### `EnemyDamaged`

```json
{ "damage": 14, "hpLeft": 12 }
```

### `EnemyDefeated`

```json
{ "enemy": "Ash Wisp", "wave": 1 }
```

### `PlayerBlocked`

```json
{}
```

### `PlayerHealed`

```json
{ "amount": 28 }
```

### `ActionRejected`

```json
{ "reason": "no potions left" }
```

### `TurnEnded`

```json
{ "turn": 7 }
```

### `PlayerDamaged`

```json
{ "damage": 11, "hpLeft": 63 }
```

### `GameOver`

```json
{ "score": 240, "wave": 4 }
```

### `LootDropped`

```json
{ "item": "potion" }
```

`item` is `"potion"`, `"elixir"` or `"coin"`.

### `LootCollected`

```json
{ "item": "potion" }
```

### `WaveCleared`

```json
{ "wave": 3 }
```

### `EnemySpawned`

```json
{ "name": "Cinder Wraith", "hp": 51 }
```

### `AchievementUnlocked`

```json
{ "name": "Wave 3 cleared" }
```

### `DeliveryFailed`

```json
{ "topic": "GameOver", "handler": "LeaderboardSync", "attempts": 3 }
```

---

## Flow diagrams

### One attack

```
PlayerActed ─> ResolveAction ─> EnemyDamaged ─> CloseTurn ─> TurnEnded ─> EnemyTurn ─> PlayerDamaged
```

### Killing an enemy

```
EnemyDefeated ─┬─> DropLoot ─> LootDropped ─> ApplyLoot ─> LootCollected ─> KeepScore
               ├─> KeepScore            (+10 × wave)
               └─> ClearWave ─> WaveCleared ─> NextWave ─> EnemySpawned
                                              └─> KeepScore (badge every 3rd wave)
```

### Failed delivery (retry → dead letter)

```
GameOver ─> LeaderboardSync (throws "offline")
             ├─ attempt 1  failed → retry
             ├─ attempt 2  failed → retry
             └─ attempt 3  failed → dead-letter
                                      └─> DeliveryFailed ─> Telemetry
```

---

## Event semantics

- **Topic** is the routing key. A handler subscribes to one topic, a list of topics, or `"*"`.
- **Idempotency.** Delivery is at-least-once, so a handler may run twice for the same event.
  Handlers guard with game state — for example `NextWave` only advances once per `WaveCleared`.
- **Business outcome vs technical failure.** `ActionRejected` and `GameOver` are normal events.
  Throwing from a handler means a technical failure and triggers a retry.
- **Ordering.** Deliveries are processed one at a time, in the order they were created, so a turn
  always resolves in the same sequence.
