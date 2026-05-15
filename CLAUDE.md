# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

RogueSnake is a single-file browser prototype — a snake-roguelike "combo feel test" (v0.1). The entire game lives in `index.html` as inline CSS + a single `<script>` block of vanilla JS rendering to a `<canvas>`. There is no build step, no package manager, no test suite, no framework.

## Running

Open `index.html` directly in a browser, or serve the folder with any static server:

```
python3 -m http.server 8000
# then visit http://localhost:8000/
```

There is no lint/build/test command — changes are validated by reloading the page.

## Architecture

The whole game is structured around three concepts that span the script. Touching one usually means understanding the other two.

### 1. Tick-based simulation with `requestAnimationFrame` render

`loop(now)` (bottom of script) runs every frame: it calls `step()` only when `now - game.lastStep >= game.tickMs` (default 150ms), then `render()` every frame. So **simulation is tick-rate-decoupled from render**. The `[` and `]` keys mutate `game.tickMs` to slow/speed the sim.

`step()` is a fixed 11-phase sequence (numbered inline in comments): direction → snake movement → pickup collection → ability cooldowns/triggers → projectiles → pulses → pickup physics → enemy AI → snake/enemy contact → death cleanup → particles/flashes. Order matters; e.g. abilities fire *after* the snake has moved so projectiles spawn from the new positions.

### 2. Positional segment abilities (the "combo feel" this prototype is testing)

The snake is `game.snake[]`, head at index 0, tail at last index. Each segment has a `type` from `SEG` (`BODY`, `FLAME`, `MAGNET`, `SHIELD`, `ECHO`) and a cooldown `cd`. Abilities trigger off a segment's position in the chain, and direction is **derived from the segment in front of it** (lower index = closer to head) — see `triggerAbility(i, ...)`. This means rearranging the same segments produces different fire arcs.

**ECHO is the keystone mechanic**: at the end of `triggerAbility()`, if the segment immediately behind index `i` is `ECHO`, the same ability re-fires from that segment. This is recursive, so `[Flame][Echo][Echo]` = 3 flames. `ECHO` itself has no standalone effect — it only amplifies what's in front of it. When adding new ability types, the ECHO chain at the bottom of `triggerAbility` will pick them up automatically as long as they slot into the `if (abilityType === ...)` ladder above it.

Cooldowns live on the segment (`seg.cd`), not on the ability type, so identical segments fire independently. `BODY` and `ECHO` are skipped in the cooldown loop in `step()` phase 4.

### 3. Toroidal world

The grid is `COLS × ROWS` (25×18) and wraps. Anywhere distance is computed — enemy AI, snake/enemy push, ability direction — you'll see the pattern:

```js
if (Math.abs(dx) > COLS / 2) dx -= Math.sign(dx) * COLS;
if (Math.abs(dy) > ROWS / 2) dy -= Math.sign(dy) * ROWS;
```

This unwraps to the shortest path across the torus seam. Any new code that does spatial math on entities **must** apply the same unwrap, or it will misbehave near the edges. Movement updates use `(... + COLS) % COLS` to wrap positions back into range.

## State shape

A single mutable `game` object holds everything, rebuilt by `reset()` (also bound to `R`). Entity arrays: `snake`, `enemies`, `projectiles`, `pickups`, `particles`, `pulses`, `flashes`. Positions are in **tile units** (floats), converted to pixels in `render()` via `(value + 0.5) * TILE`.

## Controls (also reflected in the side panel HTML)

WASD/arrows steer · `1`–`4` append Flame/Magnet/Shield/Echo to tail · `X` drop tail · `R` reset · `[`/`]` slow/fast.
