# Snake Roguelike — Combat-Feel Playground

**Date:** 2026-05-14
**Status:** Draft, pending user review
**Scope:** Pre-MVP combat-feel prototype, not the full game

---

## Why this exists

The v0.1 design doc frames the project's load-bearing question as: *does building a 10-segment chain feel like building a deck, or just collecting upgrades?* That question is downstream of a more primitive one: **does fighting enemies with this snake feel good at all?**

The existing prototype (`index.html`) covers most of design-doc Day 1 — snake on grid, 4 segment types (Flame / Magnet / Shield / Echo), ECHO recursive re-fire, basic chase enemy, pickups — but enemy contact is currently a non-event (snake and enemy push each other; nothing dies, nothing degrades). Until that interaction has weight, the combo system has nothing to chew on.

This spec deliberately cuts MVP scope (rooms, doors, procgen, boss, run loop) and focuses on making a single-room combat sandbox that lets us rapidly answer the combat-feel question with different starting builds.

## Goal

Build a combat-feel playground that lets a player:

1. Pick a starting ability segment.
2. Fight enemies in a single torus room and feel the consequences (damage, healing, death).
3. Add more ability segments on demand to construct any chain they want to test.
4. Die and restart instantly.

**Done when:** the question *"does combat feel good with this build?"* is answerable in five minutes of play, for any segment combination the player wants to try.

## Explicitly deferred (not in this scope)

- Bounded rooms with walls / wall damage
- Room transitions, multi-room state, cooldown carry-over
- Doors as segment-choice mechanic
- Procedural generation
- Boss enemy / boss room / win condition
- Run loop (run → die → restart with stats)
- Meta-progression / segment unlocks
- Solid self-collision (snake passes through itself — revisit when Coil segment is added)
- New enemy types beyond the existing chase mob
- Tail-loss-only damage (deferred in favor of the unified "decay" rule below)

These come back on the table once combat feels right.

## Locked decisions

| Area | Decision |
|---|---|
| Playfield | Existing 25×18 torus, unchanged |
| Starting snake | head + 3 BODY + 1 chosen ability |
| Head | Inherent weak forward shot, ~30-tick cooldown (Flame segment is 18 ticks, so Flame is a clear upgrade) |
| Damage rule | On contact, remove the tail-most BODY if any exist; otherwise remove the tail-most ability segment. Snake dies when only the head remains. |
| Wall behavior | N/A (torus, no walls) |
| i-frames | ~6 ticks invulnerability after any hit; head color flashes during i-frames |
| Self-collision | Snake passes through its own body (no change from prototype) |
| Ability acquisition | Starting choice card on reset; on-demand choice card via `C` during play |
| Choice card content | All 4 abilities shown each time (not "3 of 4 randomized" — that's a real-doors concern, not a playground concern) |
| Healing | BODY pickups continue to spawn in the room as the prototype already does, but the pool is **BODY-only**. Ability-type pickups are removed. |
| Enemies | Existing chase mob, unchanged behavior. Spawn count is tunable live. |
| End state | Death triggers a "DEAD — press R" overlay. No win condition. |

## Detailed design

### 1. Playfield

Unchanged from the prototype. 25 columns × 18 rows × 32px tiles. Torus wrap on all edges. The existing render, grid lines, and unwrap math (`if (Math.abs(dx) > COLS / 2) dx -= Math.sign(dx) * COLS`) stay as-is.

### 2. Snake model

`game.snake` array is unchanged in shape. Each segment has `{x, y, type, cd}`.

**Starting composition** (on every `reset()` and after a starting-choice pick):

```
[head]  [BODY]  [BODY]  [BODY]  [chosen ability]
```

The chosen ability sits at the tail. The starting state is "BODY-armored snake with one ability."

**Chain construction.** Both choice cards append the chosen segment **at the tail**. There is no insertion-in-the-middle UI. To test a chain like `[Flame][Echo][Echo]`, the player picks Flame at start (snake becomes `[head, BODY, BODY, BODY, Flame]`), then presses `C` and picks Echo twice (snake becomes `[head, BODY, BODY, BODY, Flame, Echo, Echo]`). This means the chain order is dictated by add-order. Most useful combinations are reachable; some specific orderings (e.g., ability adjacent to head) are not without an insert mechanic, which we'll add only if the playtest reveals a real need.

**Head** gains an inherent forward-shot ability. Conceptually this is the same projectile a Flame segment fires, but on a longer cooldown (~30 ticks vs 18). Implementation: in the existing `step()` ability-cooldown phase, treat the head as a segment with a custom cooldown that fires the Flame-style projectile in `headDir`. Head does not appear in the snake's "ability segments" for ECHO-chain purposes (it's an inherent ability, not a chain-able one).

### 3. Damage model

Replaces the prototype's `step()` phase 9 (snake-enemy contact pushes apart). New rule:

```
on snake-enemy contact:
  if game.iframes > 0: skip
  if any BODY exists in snake:
    remove the tail-most BODY segment
  else if any ability segment exists:
    remove the tail-most ability segment
  set game.iframes = 6
  particles + brief head-color flash
  if game.snake.length === 1: set game.dead = true
```

Notes:
- "Tail-most BODY" means: iterate from tail toward head, find the first BODY, remove it. The BODY ordering otherwise doesn't matter — players have no way to position BODY anyway since the system auto-appends.
- "Tail-most ability segment" once BODY is exhausted means your back-of-chain abilities are the first to go. The original design doc's "watch your build erode in reverse" beat survives, just gated behind the BODY buffer.
- The push-apart behavior in current phase 9 is **removed**. Enemies pass through the snake (and vice versa). This is fine because contact = damage now, not a static collision.
- ECHO segments are valid removal targets when ability-eating begins.

### 4. Segment acquisition

**Starting choice card.** When `reset()` runs, `game.choosingStart = true` is set and the main loop skips `step()` while this flag is on. An overlay renders on top of the canvas showing four labeled options:

```
PICK YOUR STARTING SEGMENT
  [1] FLAME  · fires forward
  [2] MAGNET · pulls pickups
  [3] SHIELD · damaging shockwave
  [4] ECHO   · repeats the segment in front
```

Pressing `1` / `2` / `3` / `4` appends the chosen ability after the 3 BODY, clears `choosingStart`, and the game starts.

**On-demand choice card.** Pressing `C` mid-game sets `game.choosing = true`. The tick loop pauses (same gate as `choosingStart`). The same overlay renders. `1–4` appends the chosen ability **at the tail** and resumes play. No limit on use.

**Pickups.** The existing `spawnPickup()` function selects from `['FLAME', 'MAGNET', 'SHIELD', 'ECHO']`. Change to always spawn `BODY`. Visual: rendered same as existing pickups but with the BODY gray color and a heart or `+` glyph instead of an ability letter. Picked-up BODY appends a BODY segment to the tail.

### 5. Combat content

Chase enemies behave exactly as the current prototype:
- Move 3 ticks at a time toward the head.
- Take 1 HP damage from projectiles and damaging shockwaves.
- `hitFlash` visual on hit.
- Die at 0 HP; particles spawn; respawn scheduled via `setTimeout` 800–2200ms later.

Starting enemy count is 6 (current default). `;` and `'` keys reduce / increase the target enemy population by 1 (clamped 0–20). The game uses `target enemy count` as a live tuning knob — `step()` spawns enemies until the live count reaches the target.

### 6. Dev / iteration tooling

| Key | Action |
|---|---|
| WASD / Arrows | Steer head (existing) |
| `R` | Reset → reopens starting choice card |
| `C` | Open on-demand choice card mid-game |
| `1`–`4` | When no card open: force-add Flame/Magnet/Shield/Echo (existing power-user shortcut). When a card is open: select that option. |
| `X` | Drop tail segment (existing) |
| `[` / `]` | Slower / faster tick rate (existing) |
| `;` / `'` | Decrease / increase target enemy count |

**HUD update:** the existing side panel shows total Length. Split this into two readouts:
- `BODY: 3` — the remaining health buffer
- `ABILITY: 2` — the count of non-BODY segments

This makes "how much damage can I take?" a glanceable fact during combat-feel iteration.

### 7. End state

When `game.snake.length === 1` (head only), set `game.dead = true`. The tick loop stops stepping; the renderer keeps drawing the last frame and overlays:

```
DEAD
press R to restart
```

Only `R` is honored. `R` resets the game and re-opens the starting choice card.

## Build sequence

Ordered so each step is independently testable and lands working behavior on the screen:

1. **Damage model overhaul.** Replace `step()` phase 9 (push-apart) with the new decay rule + i-frames. Add the `game.dead` flag and a stub overlay. Test: walk into enemies, watch BODY segments disappear, then abilities, then head-alone = DEAD overlay.
2. **Head inherent shot.** Treat the head as having a cooldown-based forward-shot. Test: empty starting snake (no abilities) can still kill enemies, but slowly.
3. **Pickup behavior change.** `spawnPickup()` always spawns BODY; pickup collision appends BODY; render BODY pickups with the gray color + heart glyph. Test: collect pickups in fights and watch BODY count refill.
4. **Starting choice card.** Pause-on-reset overlay; key handler appends chosen ability. Test: reload, see card, pick Flame, snake starts as head + 3 BODY + Flame.
5. **On-demand choice card (`C`).** Same overlay shown mid-game with pause. Test: press `C`, pick Echo three times, watch a [Flame][Echo][Echo][Echo] chain fire 4 projectiles per Flame trigger.
6. **HUD updates + dev keybinds.** Split BODY/ability counters in the side panel. Wire `;` / `'` for enemy-count tuning.
7. **Polish death overlay.** Style the overlay properly (semi-transparent backdrop, centered text). Gate input — only `R` honored while dead.

Total surface area: edits to `index.html` only. No new files. Estimated 200–300 lines added/changed.

## Open tuning knobs (decide via playtest, not now)

- Head shot cooldown (start at 30 ticks; may want slower so Flame feels more valuable)
- Head shot projectile speed/lifetime (default to current Flame values, then differentiate)
- i-frame duration (start at 6 ticks; raise if stack-chewing happens, lower if combat feels too forgiving)
- BODY pickup spawn rate (start at current prototype rate; lower if healing feels free)
- Enemy spawn rate after a kill (start at current 800–2200ms; lower to crank pressure)
- Starting enemy count default (6 → may want 4 for easier feel-test sessions)

These are knobs, not design questions. Iterate by tweaking constants and replaying.

## Definition of "done" for this scope

- Player can pick a starting segment and immediately fight.
- Player can press `C` to add arbitrary segments at any time.
- Player can die — the failure state is real and feels bad in the right way.
- Player can restart in under 2 seconds.
- All 4 segment types behave correctly under the new damage model (ECHO chains still re-fire; Magnet still pulls; Shield shockwave still hits; Flame still shoots).
- The combat-feel question is answerable: "yes this is fun" or "no, here's what's wrong."

## Next steps after this prototype

Once combat feels right:

1. Add bounded walls + wall damage (replaces torus). This becomes the first real "room."
2. Add room transitions with cooldown carry-over.
3. Add doors with segment-choice (replaces the on-demand `C` card with kill-gated doors).
4. Procgen for a 5-room linear floor.
5. Beefy chase boss in room 5.
6. Win/death overlays with run stats.

That's the MVP scope from the earlier brainstorming session — re-enter that flow after this prototype answers the combat-feel question.
