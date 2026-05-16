# Snake Roguelike — Combat-Feel Playground Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the existing `index.html` "combo feel test" prototype into a single-room combat playground that lets a player pick a starting ability segment, fight enemies with a real damage model (decay rule, i-frames), add more ability segments on demand via a `C`-key choice card, and die meaningfully.

**Architecture:** All changes land in `index.html`. New state lives on the existing `game` object. The existing `step()` 11-phase tick model is preserved; one phase (snake-enemy push-apart) is replaced and a new phase (i-frames decrement + dead-gate) is added. A new `HEAD` segment type is added to `SEG` so the head participates in the existing ability-cooldown phase. A "choice card" mode pauses `step()` while showing an overlay; the `keydown` handler routes `1`–`4` to either the existing force-add behavior or the open card's option selection based on `game.choosing` state.

**Tech Stack:** Vanilla JS in a single HTML file. No build, no test framework. Validation is manual visual testing in a browser.

---

## File Structure

- **Modify:** `index.html` — single file; entire prototype lives here. All edits land in the inline `<script>` block.
- **No new files.** The spec mandates the single-file structure.

## Verification approach

There is no automated test framework. Each task ends with a **"Verify behavior"** step listing specific actions and the expected observations. Run the game by opening `index.html` directly in a browser, or by serving the folder:

```
python3 -m http.server 8000
# visit http://localhost:8000/
```

Reload the page after each save. Use the keyboard reference in the side panel for steering and dev keys.

## Reference: existing key bindings (do not break)

- **WASD / Arrow keys** — steer the head
- **`1`–`4`** — force-add Flame / Magnet / Shield / Echo to the tail (preserved as a power-user shortcut; routes to card-pick when a choice card is open)
- **`X`** — drop tail segment
- **`R`** — reset game
- **`[` / `]`** — slower / faster tick rate

---

## Task 1: Damage model + i-frames + dead-state gate + stub overlay

**Files:**
- Modify: `index.html` — `reset()` (state init), `step()` (new i-frame decrement + dead gate + replace phase 9), add new `applyDamage()` helper, `render()` (stub DEAD overlay), keydown handler (R-only-when-dead)

**Goal of this task:** Replace the current "snake and enemy push each other apart" no-op contact with a real damage rule: a hit removes the tail-most BODY first, then the tail-most non-head ability segment, then sets a 6-tick invulnerability window. When only the head remains, the game enters a dead state showing a stub overlay; only `R` is honored in that state.

- [ ] **Step 1: Add `iframes` and `dead` fields to game state**

In `reset()`, add two new fields to the `game` object. Find the existing state init block and add the two fields shown:

```javascript
function reset() {
  game = {
    snake: [
      { x: 12, y: 9, type: 'BODY', cd: 0 },
      { x: 11, y: 9, type: 'BODY', cd: 0 },
      { x: 10, y: 9, type: 'BODY', cd: 0 },
    ],
    headDir: { x: 1, y: 0 },
    queuedDir: { x: 1, y: 0 },
    enemies: [],
    projectiles: [],
    pickups: [],
    particles: [],
    pulses: [],
    flashes: [],
    tick: 0,
    kills: 0,
    tickMs: 150,
    lastStep: 0,
    pendingSpawns: { enemy: [], pickup: [] },
    iframes: 0,    // NEW: invulnerability tick counter
    dead: false,   // NEW: game-over flag
  };
  for (let i = 0; i < 6; i++) spawnEnemy();
  for (let i = 0; i < 3; i++) spawnPickup();
}
```

- [ ] **Step 2: Add a dead-state gate at the top of `step()` and an iframes decrement**

Modify the very start of `step()`. Find the existing first line `game.tick++;` and insert the gate + decrement before it:

```javascript
function step() {
  if (game.dead) return;
  if (game.iframes > 0) game.iframes--;
  game.tick++;
  // ... rest of step() unchanged
}
```

- [ ] **Step 3: Add the `applyDamage()` helper above `step()`**

Place this new function in the helpers section (near `dropTail()`, around the existing `spawnFlash()`):

```javascript
function applyDamage() {
  // 1. Tail-most BODY first
  let removed = false;
  for (let i = game.snake.length - 1; i >= 1; i--) {
    if (game.snake[i].type === 'BODY') {
      const lost = game.snake.splice(i, 1)[0];
      for (let k = 0; k < 10; k++) spawnParticle(lost.x, lost.y, '#5a607a');
      removed = true;
      break;
    }
  }
  // 2. If no BODY, tail-most non-head ability segment
  if (!removed) {
    for (let i = game.snake.length - 1; i >= 1; i--) {
      const t = game.snake[i].type;
      if (t !== 'BODY' && t !== 'HEAD') {
        const lost = game.snake.splice(i, 1)[0];
        const c = SEG[lost.type] ? SEG[lost.type].color : '#ff4060';
        for (let k = 0; k < 14; k++) spawnParticle(lost.x, lost.y, c);
        break;
      }
    }
  }
  game.iframes = 6;
  if (game.snake.length === 1) game.dead = true;
}
```

- [ ] **Step 4: Replace step phase 9 (snake-enemy contact)**

Locate the existing phase-9 block in `step()` — it begins with the comment `// 9. Snake-enemy contact: push enemy away (no death in feel test)` and currently iterates `game.enemies.forEach(e => { ... push apart ... })`. Replace the entire block with this:

```javascript
  // 9. Snake-enemy contact: apply damage (decay rule)
  if (game.iframes <= 0) {
    let contacted = false;
    outer:
    for (const e of game.enemies) {
      for (const seg of game.snake) {
        let dx = e.x - seg.x, dy = e.y - seg.y;
        if (Math.abs(dx) > COLS / 2) dx -= Math.sign(dx) * COLS;
        if (Math.abs(dy) > ROWS / 2) dy -= Math.sign(dy) * ROWS;
        if (Math.hypot(dx, dy) < 0.8) {
          contacted = true;
          break outer;
        }
      }
    }
    if (contacted) applyDamage();
  }
```

- [ ] **Step 5: Add head-flash effect during i-frames in `render()`**

Find the existing snake-render block inside `render()` (begins with `// snake` and `game.snake.forEach((seg, i) => {`). Replace the head fillStyle line:

```javascript
    ctx.fillStyle = isHead ? '#f0f0ff' : def.color;
```

with:

```javascript
    const headColor = (game.iframes > 0 && Math.floor(game.tick / 2) % 2 === 0) ? '#ff4060' : '#f0f0ff';
    ctx.fillStyle = isHead ? headColor : def.color;
```

Also change the `shadowColor` line just below it from `isHead ? '#f0f0ff'` to `isHead ? headColor`:

```javascript
    ctx.shadowColor = isHead ? headColor : def.color;
```

- [ ] **Step 6: Add a stub DEAD overlay at the end of `render()`**

At the very end of `render()`, just before the final `updatePanel();` call, insert:

```javascript
  if (game.dead) {
    ctx.fillStyle = 'rgba(0,0,0,0.55)';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = '#ff4060';
    ctx.font = 'bold 48px monospace';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText('DEAD', canvas.width / 2, canvas.height / 2 - 16);
    ctx.fillStyle = '#d4d7e3';
    ctx.font = '16px monospace';
    ctx.fillText('press R to restart', canvas.width / 2, canvas.height / 2 + 24);
  }
```

- [ ] **Step 7: Gate the keydown handler so only `R` is honored when dead**

In the existing `document.addEventListener('keydown', e => { ... })` handler, immediately after the `const k = e.key.toLowerCase();` line, insert:

```javascript
  if (game.dead && k !== 'r') return;
```

- [ ] **Step 8: Verify behavior in a browser**

Note: in Task 1 the starting snake is unchanged from the prototype — 3 segments (head + 2 BODY behind it). The "head" segment is currently typed `'BODY'` but sits at index 0; `applyDamage` only scans `i >= 1`, so the head is never touched. (Task 2 will retype the head to `'HEAD'`.)

1. Open `index.html` in a browser. The game starts as before — 3 segments visible (head + 2 BODY behind it), enemies and pickups on screen, side panel Length: 3.
2. Steer into an enemy. Expected: one BODY segment vanishes with gray particles, head flashes red briefly, Length: 2. Cannot take another hit for ~6 ticks (head stays slightly flickery).
3. Press `1` twice and `4` twice to force-add Flame, Flame, Echo, Echo. Length: 6 (head + 1 remaining BODY + 2 Flame + 2 Echo).
4. Take 1 more hit. The remaining BODY vanishes (gray particles). Length: 5.
5. Take another hit. The tail-most ECHO vanishes (yellow particles). Length: 4.
6. Keep taking hits. Each removes the next tail-most ability segment. Length drops 4 → 3 → 2 → 1. At Length 1, "DEAD — press R to restart" overlay appears centered on the canvas with a dark backdrop.
7. While dead, press any movement keys: nothing happens. Press `R`: game resets, overlay clears, you can play again.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Task 1: damage model, i-frames, dead-state gate, stub overlay

- Snake-enemy contact now removes tail-most BODY, then tail-most
  non-head ability segment (decay rule from spec).
- 6-tick i-frames after each hit; head flickers red during i-frames.
- game.dead = true when only head remains; step() gates on dead.
- Stub DEAD overlay rendered when dead; keydown ignores everything
  except R while dead.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Head inherent forward shot

**Files:**
- Modify: `index.html` — `SEG` constant (add HEAD type), `reset()` (head's type), `triggerAbility()` (add HEAD case + guard ECHO chain), `render()` (head label suppression already works)

**Goal of this task:** Give the head an inherent slow forward-shot ability — same projectile as a Flame segment but on a 30-tick cooldown. This means a starting snake with no ability segments can still fight (slowly). The head must NOT trigger the ECHO chain (its shot is non-chainable per spec).

- [ ] **Step 1: Add `HEAD` to the `SEG` constant**

Find the `SEG` declaration block. Replace it with:

```javascript
const SEG = {
  HEAD:   { color: '#f0f0ff', label: '◉', cooldown: 30 },
  BODY:   { color: '#5a607a', label: '·',  cooldown: 0 },
  FLAME:  { color: '#ff6b35', label: 'F',  cooldown: 18 },
  MAGNET: { color: '#a64bff', label: 'M',  cooldown: 6 },
  SHIELD: { color: '#4bd9ff', label: 'S',  cooldown: 14 },
  ECHO:   { color: '#ffd84b', label: 'E',  cooldown: 0 },
};
```

- [ ] **Step 2: Change the head's `type` in `reset()` from `'BODY'` to `'HEAD'`**

In `reset()`, the snake array starts with three `{ type: 'BODY' }` segments. Change the first one to type `'HEAD'`:

```javascript
    snake: [
      { x: 12, y: 9, type: 'HEAD', cd: 0 },
      { x: 11, y: 9, type: 'BODY', cd: 0 },
      { x: 10, y: 9, type: 'BODY', cd: 0 },
    ],
```

- [ ] **Step 3: Add a HEAD case in `triggerAbility()` and guard the ECHO chain**

Replace the entire `triggerAbility(i, abilityType)` function with the version below. Two changes from the existing function: a new `abilityType === 'HEAD'` branch at the top of the if/else ladder, and an `abilityType !== 'HEAD'` guard added to the ECHO chain at the bottom. The FLAME / MAGNET / SHIELD bodies are unchanged from the existing file — they're reproduced here in full so the function is correct as a whole if dropped in.

```javascript
function triggerAbility(i, abilityType) {
  const seg = game.snake[i];
  spawnFlash({ x: seg.x, y: seg.y, type: abilityType });

  // direction = where segment in front of this is, vs this
  // (front = closer to head = lower index)
  let ndir = { x: 1, y: 0 };
  if (i > 0) {
    const front = game.snake[i - 1];
    let dx = front.x - seg.x, dy = front.y - seg.y;
    if (Math.abs(dx) > COLS / 2) dx -= Math.sign(dx) * COLS;
    if (Math.abs(dy) > ROWS / 2) dy -= Math.sign(dy) * ROWS;
    const len = Math.hypot(dx, dy) || 1;
    ndir = { x: dx / len, y: dy / len };
  } else {
    ndir = { ...game.headDir };
  }

  if (abilityType === 'HEAD') {
    game.projectiles.push({
      x: seg.x, y: seg.y,
      vx: ndir.x * 0.5, vy: ndir.y * 0.5,
      life: 30, color: '#f0f0ff'
    });
  } else if (abilityType === 'FLAME') {
    game.projectiles.push({
      x: seg.x, y: seg.y,
      vx: ndir.x * 0.5, vy: ndir.y * 0.5,
      life: 30, color: '#ff6b35'
    });
  } else if (abilityType === 'MAGNET') {
    game.pickups.forEach(p => {
      let dx = seg.x - p.x, dy = seg.y - p.y;
      if (Math.abs(dx) > COLS / 2) dx -= Math.sign(dx) * COLS;
      if (Math.abs(dy) > ROWS / 2) dy -= Math.sign(dy) * ROWS;
      const dist = Math.hypot(dx, dy);
      if (dist > 0.3 && dist < 6) {
        p.vx = dx / dist * 0.45;
        p.vy = dy / dist * 0.45;
      }
    });
    game.pulses.push({
      x: seg.x, y: seg.y, r: 0, maxR: 6,
      speed: 0.6, life: 10,
      color: '#a64bff', damage: false, hit: new Set()
    });
  } else if (abilityType === 'SHIELD') {
    game.pulses.push({
      x: seg.x, y: seg.y, r: 0, maxR: 3,
      speed: 0.3, life: 18,
      color: '#4bd9ff', damage: true, hit: new Set()
    });
  }

  // === ECHO CHAIN ===
  // If the segment immediately BEHIND this is ECHO, fire the same ability from there.
  // Chains: [Flame][Echo][Echo] = 3 flames. HEAD is excluded — its shot does not chain.
  if (abilityType !== 'HEAD' && i + 1 < game.snake.length && game.snake[i + 1].type === 'ECHO') {
    triggerAbility(i + 1, abilityType);
  }
}
```

- [ ] **Step 4: Verify behavior in a browser**

1. Reload the page. Note: phase 4 of `step()` already does `if (seg.type === 'BODY' || seg.type === 'ECHO') continue;` — HEAD type is not in that skip list, so it gets ticked.
2. Don't add any segments. The starting snake is head + 2 BODY (you'll see length 3 in the panel).
3. Wait ~0.5 seconds (30 ticks at 150ms). The head should fire a white projectile forward. Verify: a white dot leaves the head in the direction it's facing, travels forward, and can kill an enemy on contact (the existing projectile-vs-enemy phase doesn't care about color).
4. Turn the head; the next shot fires in the new direction.
5. Press `1` to add a Flame. Now both head AND Flame fire — head every ~30 ticks (slower), Flame every 18 ticks (faster). Flame shots should be orange, head shots white. **This is the head-vs-Flame distinction the spec mandates: Flame is a clear upgrade.**
6. Press `4` (Echo) to add an Echo right behind the Flame. Watch Flame trigger: it should fire 2 orange shots (Flame chained to Echo). Watch the head trigger: it should fire only 1 white shot (HEAD does NOT chain even if Echo were behind the head — verify by trying to construct that state with `1`/`4` and observing).
7. Verify the head cooldown ring renders: the existing render loop draws a partial arc for ability segments with `cooldown > 0`. The head now has cooldown 30, so a thin ring should sweep around the head's circle between shots.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Task 2: head's inherent forward shot

- Added HEAD type to SEG (cooldown 30, white #f0f0ff).
- Head segment in reset() is now type 'HEAD' (was 'BODY').
- triggerAbility('HEAD') spawns a Flame-style projectile in headDir.
- ECHO chain is guarded against HEAD: the head's shot does not chain
  even if an ECHO sits behind it.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: BODY-only pickups + healing visual

**Files:**
- Modify: `index.html` — `spawnPickup()` (always BODY), `render()` pickup loop (heart/plus glyph for BODY)

**Goal of this task:** Remove ability-type variety from in-room pickups. Pickups now always spawn BODY segments, functioning as healing (each pickup = +1 BODY appended to the tail). Visually distinguish BODY pickups with a `+` healing glyph so they don't read as a generic body part.

- [ ] **Step 1: Update `spawnPickup()` to always create a BODY pickup**

Replace the existing `spawnPickup()` function:

```javascript
function spawnPickup() {
  const t = emptyTile();
  game.pickups.push({ x: t.x, y: t.y, vx: 0, vy: 0, type: 'BODY', bob: Math.random() * Math.PI * 2 });
}
```

(Removes the `types` array and the random pick.)

- [ ] **Step 2: Update the pickup-render block to use a `+` glyph for BODY**

Find the pickup-render block in `render()` — it begins with `// pickups` and `game.pickups.forEach(p => {`. Replace the `fillText` glyph for BODY by changing this section:

```javascript
  game.pickups.forEach(p => {
    const px = (p.x + 0.5) * TILE;
    const py = (p.y + 0.5) * TILE + Math.sin(p.bob) * 3;
    const c = SEG[p.type].color;
    ctx.fillStyle = c;
    ctx.shadowColor = c;
    ctx.shadowBlur = 14;
    ctx.beginPath();
    ctx.arc(px, py, TILE * 0.32, 0, Math.PI * 2);
    ctx.fill();
    ctx.shadowBlur = 0;
    ctx.fillStyle = '#000';
    ctx.font = 'bold 13px monospace';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(SEG[p.type].label, px, py);
  });
```

with this:

```javascript
  game.pickups.forEach(p => {
    const px = (p.x + 0.5) * TILE;
    const py = (p.y + 0.5) * TILE + Math.sin(p.bob) * 3;
    const isBody = p.type === 'BODY';
    const c = isBody ? '#7ad9a0' : SEG[p.type].color;
    ctx.fillStyle = c;
    ctx.shadowColor = c;
    ctx.shadowBlur = 14;
    ctx.beginPath();
    ctx.arc(px, py, TILE * 0.32, 0, Math.PI * 2);
    ctx.fill();
    ctx.shadowBlur = 0;
    ctx.fillStyle = '#000';
    ctx.font = 'bold 16px monospace';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(isBody ? '+' : SEG[p.type].label, px, py);
  });
```

Two changes: BODY uses a green-ish color `#7ad9a0` (reads as "health") instead of the gray BODY color so it's visually distinct from snake segments; and the glyph is `+` for BODY.

- [ ] **Step 3: Verify behavior in a browser**

1. Reload. Look at the playfield. Pickups should now all be green circles with `+` glyphs.
2. Pick one up by steering the head into it. Expected: pickup vanishes with green-ish particles (uses the SEG color of the picked-up type via the existing `spawnParticle` loop in step phase 3 — that will be the gray BODY color since `p.type` is 'BODY'). The snake gains a BODY segment at the tail. Length in the side panel goes up by 1.
4. Force-add a Flame (`1`), then take a hit. Watch the chain: the new BODY (just added from pickup) at the tail dies first, not the Flame. Confirms BODY-armor-at-tail behavior.
5. Wait a few seconds. New pickups continue spawning. All are green `+` BODY pickups; no ability-type pickups appear anymore.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Task 3: BODY-only pickups as healing

- spawnPickup() always creates type 'BODY'; ability-type pickups removed.
- BODY pickups render with a green color (#7ad9a0) and '+' glyph to
  read as healing, distinct from gray BODY segments in the snake.
- addSegment('BODY') on pickup collision is unchanged — still appends
  to tail, which slots the new BODY into the armor zone.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Starting choice card (pause on reset)

**Files:**
- Modify: `index.html` — `reset()` (set `choosingStart` flag, skip enemy spawn until choice made), main `loop()` (skip `step()` while choosing), `render()` (call `renderChoiceCard` overlay), keydown handler (route 1–4 to card pick when choosing), add helper `pickStartingSegment(type)`

**Goal of this task:** On every `reset()`, instead of immediately starting the game, show a choice card overlay listing the 4 abilities. The simulation pauses. Pressing `1`–`4` appends the chosen ability to the snake and starts the game. This is the *deck-prep* moment for every run.

- [ ] **Step 1: Add `choosing` and `choosingStart` flags to game state**

In `reset()`, add two flags. Also, **don't spawn enemies or pickups yet** — defer that until the player picks. The new `reset()`:

```javascript
function reset() {
  game = {
    snake: [
      { x: 12, y: 9, type: 'HEAD', cd: 0 },
      { x: 11, y: 9, type: 'BODY', cd: 0 },
      { x: 10, y: 9, type: 'BODY', cd: 0 },
      { x: 9,  y: 9, type: 'BODY', cd: 0 },
    ],
    headDir: { x: 1, y: 0 },
    queuedDir: { x: 1, y: 0 },
    enemies: [],
    projectiles: [],
    pickups: [],
    particles: [],
    pulses: [],
    flashes: [],
    tick: 0,
    kills: 0,
    tickMs: 150,
    lastStep: 0,
    pendingSpawns: { enemy: [], pickup: [] },
    iframes: 0,
    dead: false,
    choosingStart: true,   // NEW: starting choice card is up
    choosing: false,       // NEW: on-demand choice card is up (Task 5)
  };
  // Enemies/pickups spawn after the starting choice is made.
}
```

Note: the starting snake is now **head + 3 BODY** (matches spec — 3 BODY of health buffer), one segment longer than the previous prototype default.

- [ ] **Step 2: Add a `pickStartingSegment(type)` helper**

Place this near the other helpers (e.g. just after `dropTail()`):

```javascript
function pickStartingSegment(type) {
  addSegment(type);
  game.choosingStart = false;
  for (let i = 0; i < 6; i++) spawnEnemy();
  for (let i = 0; i < 3; i++) spawnPickup();
}
```

This appends the chosen ability (so snake becomes head + 3 BODY + chosen ability), clears the flag, and seeds the room.

- [ ] **Step 3: Add a `renderChoiceCard(title)` helper**

Place this just above `render()`. It draws the overlay over the canvas:

```javascript
function renderChoiceCard(title) {
  ctx.fillStyle = 'rgba(0,0,0,0.65)';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = '#ffd84b';
  ctx.font = 'bold 22px monospace';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(title, canvas.width / 2, 90);

  const opts = [
    { key: '1', type: 'FLAME',  hint: 'fires forward' },
    { key: '2', type: 'MAGNET', hint: 'pulls pickups' },
    { key: '3', type: 'SHIELD', hint: 'damaging shockwave' },
    { key: '4', type: 'ECHO',   hint: 'repeats the segment in front' },
  ];
  const cx = canvas.width / 2;
  const startY = 170;
  const rowH = 70;

  opts.forEach((o, idx) => {
    const y = startY + idx * rowH;
    const def = SEG[o.type];
    // Card row background
    ctx.fillStyle = 'rgba(26,29,42,0.9)';
    ctx.fillRect(cx - 220, y - 25, 440, 50);
    // Color disc
    ctx.fillStyle = def.color;
    ctx.shadowColor = def.color;
    ctx.shadowBlur = 14;
    ctx.beginPath();
    ctx.arc(cx - 180, y, 14, 0, Math.PI * 2);
    ctx.fill();
    ctx.shadowBlur = 0;
    ctx.fillStyle = '#000';
    ctx.font = 'bold 14px monospace';
    ctx.textBaseline = 'middle';
    ctx.textAlign = 'center';
    ctx.fillText(def.label, cx - 180, y);
    // Key + name + hint
    ctx.fillStyle = '#d4d7e3';
    ctx.font = 'bold 16px monospace';
    ctx.textAlign = 'left';
    ctx.fillText(`[${o.key}] ${o.type}`, cx - 150, y);
    ctx.fillStyle = '#6a6f85';
    ctx.font = '13px monospace';
    ctx.fillText(o.hint, cx - 150, y + 18);
  });
}
```

- [ ] **Step 4: Call `renderChoiceCard` in `render()` when choosing**

Insert the choice-card render block **immediately before** the existing DEAD overlay block (added in Task 1). The intended final ordering at the bottom of `render()` is:

1. game-world rendering (snake, enemies, projectiles, particles, etc. — existing)
2. choice card overlay (new, this step)
3. DEAD overlay (from Task 1)
4. `updatePanel();` (existing)

This ordering means DEAD takes visual precedence if both flags are ever true simultaneously (shouldn't happen, but is defensively correct). Insert:

```javascript
  if (game.choosingStart) {
    renderChoiceCard('PICK YOUR STARTING SEGMENT');
  } else if (game.choosing) {
    renderChoiceCard('PICK A SEGMENT');
  }
```

If your local file currently has the DEAD overlay sitting just before `updatePanel()`, you do not need to move it — just insert the choice-card block immediately above the DEAD block.

- [ ] **Step 5: Pause `step()` while a card is up**

Modify the `step()` gate at the top (already added in Task 1):

```javascript
function step() {
  if (game.dead) return;
  if (game.choosingStart || game.choosing) return;
  if (game.iframes > 0) game.iframes--;
  game.tick++;
  // ... rest unchanged
}
```

- [ ] **Step 6: Route `1`–`4` keys to card-pick when a card is up**

In the keydown handler, find the existing `1`/`2`/`3`/`4` branches (they currently call `addSegment('FLAME')` etc.). Replace those four branches with a single check that routes based on choosing-state:

```javascript
  else if (k === '1' || k === '2' || k === '3' || k === '4') {
    const map = { '1': 'FLAME', '2': 'MAGNET', '3': 'SHIELD', '4': 'ECHO' };
    const chosen = map[k];
    if (game.choosingStart) {
      pickStartingSegment(chosen);
    } else if (game.choosing) {
      addSegment(chosen);
      game.choosing = false;
    } else {
      addSegment(chosen); // existing force-add behavior preserved
    }
  }
```

This single branch handles all three modes: starting card pick, on-demand card pick (Task 5 uses it), and the legacy force-add for power-user iteration.

- [ ] **Step 7: Verify behavior in a browser**

1. Reload the page. The canvas should be dimmed with a "PICK YOUR STARTING SEGMENT" header at the top and four labeled options (FLAME / MAGNET / SHIELD / ECHO). No enemies or pickups should be visible behind the overlay yet.
2. Press `1`. The card vanishes. The snake (head + 3 BODY + 1 FLAME at tail) starts moving. 6 enemies and 3 pickups spawn. Flame starts firing on its 18-tick cooldown.
3. Press `R`. The card returns. Pick `2` (Magnet). Snake starts with head + 3 BODY + 1 Magnet at tail. Magnet pulses pulse around the magnet segment.
4. Press `R`, pick `4` (Echo). Snake starts with head + 3 BODY + Echo. Echo alone is reactive only — head shot doesn't chain (per Task 2 guard), so verify the snake fires only head shots, no chained shots.
5. Confirm steering, pickups, enemy death all work normally after a pick is made.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Task 4: starting choice card

- reset() now pauses on a 'PICK YOUR STARTING SEGMENT' overlay
  showing all 4 abilities. Enemies and pickups are not spawned
  until the player picks.
- Starting snake is now head + 3 BODY (was head + 2 BODY), matching
  the spec's 3-HP buffer.
- pickStartingSegment(type) appends the chosen ability, clears the
  flag, and seeds the room.
- 1-4 keys are routed: starting card -> pickStartingSegment;
  otherwise force-add (existing power-user behavior).
- step() pauses while choosingStart is true.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: On-demand choice card (`C` key)

**Files:**
- Modify: `index.html` — keydown handler (handle `C`), the `1`–`4` routing block already from Task 4 supports `choosing`

**Goal of this task:** Allow the player to open a segment-choice card during play by pressing `C`. The simulation pauses while it's up. Picking appends the segment to the tail and resumes. This is the deck-construction tool.

- [ ] **Step 1: Add a `C` key handler in the keydown listener**

Find the keydown handler's else-if ladder. Add a new branch (place it near `'r'` and `'x'`):

```javascript
  else if (k === 'c') {
    if (!game.dead && !game.choosingStart) {
      game.choosing = !game.choosing; // toggle: press C again to cancel
    }
  }
```

(Toggle behavior — pressing `C` opens, pressing `C` again closes without picking. Useful for "I didn't want anything after all.")

- [ ] **Step 2: Update the in-panel controls hint to mention `C`**

In the HTML side panel (look for the `<h3>Controls</h3>` block in `index.html`), insert a new control row after the existing reset row:

```html
<div class="ctrl-row"><span class="key">C</span> open choice card mid-game</div>
```

Place it just before the `<div class="ctrl-row"><span class="key">[</span> slower ...` line. The exact location is cosmetic but the player needs to know the key exists.

- [ ] **Step 3: Verify behavior in a browser**

1. Reload. Pick a starting segment (say FLAME). Play for a few seconds.
2. Press `C`. The "PICK A SEGMENT" card appears, dimming the playfield. The snake stops moving. Projectiles in flight also pause.
3. Press `4` (Echo). The card vanishes; snake now has Flame followed by Echo at the tail. Watch the next Flame trigger fire 2 projectiles (chain).
4. Press `C` again. Card opens. Press `C` once more without picking. Card closes; game resumes without a new segment.
5. Try `C` while the starting card is up — should be a no-op (the `if (!game.dead && !game.choosingStart)` guard).
6. Try `C` while dead — should be a no-op (same guard).
7. Confirm the side panel now lists the `C` key under Controls.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Task 5: on-demand choice card via C key

- Pressing C during play toggles a 'PICK A SEGMENT' card; sim pauses
  while it's up. Pressing C again without a pick cancels.
- 1-4 keys (from Task 4's routing) call addSegment then clear the
  flag; segment lands at the tail.
- Guarded against use during the starting card or while dead.
- Side panel updated to document the C binding.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: HUD updates + dev keybinds (`;` / `'`)

**Files:**
- Modify: `index.html` — side panel HTML (add BODY / ABILITY stat rows; add hint for ; ' keys), `updatePanel()` (compute and write both counts), keydown handler (`;` / `'`), add `targetEnemyCount` to state, modify enemy-respawn behavior to target this count

**Goal of this task:** Split the side panel's "Length" stat into separate BODY and ABILITY counts so the player can see their damage buffer and build size at a glance. Add `;` / `'` keys to live-tune the enemy population, useful for stress-testing builds.

- [ ] **Step 1: Replace the side panel's Length stat row with BODY + ABILITY rows**

Find this block in the HTML:

```html
<h3>Stats</h3>
<div class="stat"><span>Length</span><span id="stat-len">3</span></div>
<div class="stat"><span>Kills</span><span id="stat-kills">0</span></div>
<div class="stat"><span>Speed</span><span id="stat-speed">150ms</span></div>
```

Replace with:

```html
<h3>Stats</h3>
<div class="stat"><span>BODY (health)</span><span id="stat-body">3</span></div>
<div class="stat"><span>Ability</span><span id="stat-ability">0</span></div>
<div class="stat"><span>Kills</span><span id="stat-kills">0</span></div>
<div class="stat"><span>Speed</span><span id="stat-speed">150ms</span></div>
<div class="stat"><span>Enemies</span><span id="stat-enemies">6</span></div>
```

- [ ] **Step 2: Replace `updatePanel()` with the version that writes both counts**

Replace the entire `updatePanel()` function with this. Two changes from the existing version: the stat-id assignments at the top (now five rows instead of three) and a `bodyCount`/`abilityCount` loop to compute them. The chain visualization block at the bottom is unchanged from the existing file, reproduced here so the function is correct as a whole if dropped in.

```javascript
function updatePanel() {
  let bodyCount = 0, abilityCount = 0;
  for (let i = 1; i < game.snake.length; i++) {
    if (game.snake[i].type === 'BODY') bodyCount++;
    else abilityCount++;
  }
  document.getElementById('stat-body').textContent = bodyCount;
  document.getElementById('stat-ability').textContent = abilityCount;
  document.getElementById('stat-kills').textContent = game.kills;
  document.getElementById('stat-speed').textContent = game.tickMs + 'ms';
  document.getElementById('stat-enemies').textContent = game.targetEnemyCount;

  const chain = document.getElementById('chain');
  chain.innerHTML = '';
  game.snake.forEach((seg, i) => {
    const el = document.createElement('div');
    el.className = 'chain-seg';
    const def = SEG[seg.type];
    if (i === 0) {
      el.style.background = '#f0f0ff';
      el.textContent = '◉';
    } else {
      el.style.background = def.color;
      el.textContent = def.label;
    }
    chain.appendChild(el);
    if (i < game.snake.length - 1) {
      const ar = document.createElement('div');
      ar.className = 'arrow';
      ar.textContent = '›';
      chain.appendChild(ar);
    }
  });
}
```

(`i` starts at 1 in the count loop to skip the head.)

- [ ] **Step 3: Add `targetEnemyCount` to game state**

In `reset()`, add the new field (start at 6, matching the current default):

```javascript
    targetEnemyCount: 6, // NEW
```

- [ ] **Step 4: Use `targetEnemyCount` in the kill-and-respawn block of `step()`**

Find phase 10 (`// 10. Clean dead enemies + respawn`). Currently it does `setTimeout(spawnEnemy, ...)` for each killed enemy. We want this to converge to the live target — kills schedule respawns only if the eventual count would be at or below the target. Replace the phase-10 block with:

```javascript
  // 10. Clean dead enemies + respawn toward targetEnemyCount
  const before = game.enemies.length;
  game.enemies = game.enemies.filter(e => e.hp > 0);
  const killed = before - game.enemies.length;
  game.kills += killed;
  // Schedule respawns to converge toward target
  const deficit = game.targetEnemyCount - game.enemies.length;
  for (let i = 0; i < deficit; i++) {
    setTimeout(spawnEnemy, 800 + rand(1400));
    if (Math.random() < 0.35) setTimeout(spawnPickup, 400 + rand(600));
  }
```

This makes the population organically chase the target after kills.

- [ ] **Step 5: Add `;` and `'` keys to tune `targetEnemyCount`**

In the keydown handler, add two new branches (place near `[` and `]`):

```javascript
  else if (k === ';') game.targetEnemyCount = Math.max(0, game.targetEnemyCount - 1);
  else if (k === "'") game.targetEnemyCount = Math.min(20, game.targetEnemyCount + 1);
```

Note the single-quote key is `"'"` (a string containing a single quote).

- [ ] **Step 6: Add a control hint for `;` and `'` to the side panel**

In the controls list inside the side panel HTML, near the `[` / `]` row, add:

```html
<div class="ctrl-row"><span class="key">;</span> fewer enemies &nbsp; <span class="key">'</span> more enemies</div>
```

- [ ] **Step 7: Verify behavior in a browser**

1. Reload. Pick FLAME. The side panel shows: BODY 3 / Ability 1 / Kills 0 / Speed 150ms / Enemies 6.
2. Press `4` once via on-demand card (`C` then `4`) — Ability goes to 2.
3. Take a hit. BODY 3 → 2. Take 2 more hits — BODY 0. Take another — Ability 2 → 1 (tail-most ability gone).
4. Pick up a BODY pickup. BODY 0 → 1.
5. Press `;` three times. The Enemies stat counter drops to 3. As enemies die, fewer respawn — population converges to 3.
6. Press `'` until the counter says 15. Enemies start swarming as the population converges up.
7. Confirm bordering values: `;` won't drop below 0; `'` won't go above 20.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Task 6: HUD splits BODY/ability counts + enemy-count tuning keys

- Side panel shows BODY (health) and Ability counts separately,
  plus a live Enemies counter.
- updatePanel() computes both counts from the snake array (skips head).
- game.targetEnemyCount controls the equilibrium enemy population;
  step phase 10 schedules respawns toward this target instead of
  1-for-1.
- ; / ' decrement / increment targetEnemyCount (clamped 0-20).
- Side panel documents the new keys.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Death overlay polish

**Files:**
- Modify: `index.html` — `render()` DEAD overlay block (replace the Task 1 stub with a styled version)

**Goal of this task:** Replace the stub DEAD overlay with a slightly more polished version: a translucent backdrop, a large red title, a subtitle showing how many enemies you killed and how long you survived, and the restart hint.

- [ ] **Step 1: Add a `deathTime` field captured at death moment**

In `reset()`, add:

```javascript
    deathTime: 0,    // NEW: performance.now() captured when snake dies
    startTime: 0,    // NEW: performance.now() captured when game starts
```

In `pickStartingSegment(type)` (added in Task 4), set the start time at the moment play actually begins:

```javascript
function pickStartingSegment(type) {
  addSegment(type);
  game.choosingStart = false;
  game.startTime = performance.now();
  for (let i = 0; i < 6; i++) spawnEnemy();
  for (let i = 0; i < 3; i++) spawnPickup();
}
```

In `applyDamage()` (from Task 1), set the death time when the dead flag is set. Replace the existing `if (game.snake.length === 1) game.dead = true;` with:

```javascript
  if (game.snake.length === 1) {
    game.dead = true;
    game.deathTime = performance.now();
  }
```

- [ ] **Step 2: Replace the stub DEAD overlay with the polished version**

Find the existing DEAD overlay block in `render()` (added in Task 1). Replace it with:

```javascript
  if (game.dead) {
    ctx.fillStyle = 'rgba(10,11,17,0.78)';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';

    ctx.fillStyle = '#ff4060';
    ctx.font = 'bold 56px monospace';
    ctx.fillText('DEAD', canvas.width / 2, canvas.height / 2 - 50);

    const seconds = Math.max(0, Math.floor((game.deathTime - game.startTime) / 1000));
    ctx.fillStyle = '#d4d7e3';
    ctx.font = '16px monospace';
    ctx.fillText(`survived ${seconds}s · ${game.kills} kills`, canvas.width / 2, canvas.height / 2 + 6);

    ctx.fillStyle = '#6a6f85';
    ctx.font = '14px monospace';
    ctx.fillText('press R to restart', canvas.width / 2, canvas.height / 2 + 40);
  }
```

- [ ] **Step 3: Verify behavior in a browser**

1. Reload. Pick FLAME. Play for ~20 seconds — kill some enemies, watch the kill count rise.
2. Deliberately get hit until you die (let enemies eat the snake).
3. Verify the overlay: large red "DEAD" title, subtitle reads e.g. "survived 18s · 7 kills", restart hint below.
4. Press `R`. Overlay clears, starting choice card appears, kill count resets to 0.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "$(cat <<'EOF'
Task 7: polished death overlay with run stats

- DEAD overlay shows seconds survived and kill count.
- startTime captured when player picks a starting segment;
  deathTime captured when head becomes alone.
- Visual polish: darker backdrop, larger title, run-stats line,
  restart hint.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: End-to-end smoke test

**Files:** none changed; this is a verification pass.

**Goal of this task:** Walk the full feel-test flow end-to-end with each starting segment to make sure nothing regressed during the build sequence. No code changes. If anything is wrong, find the task it belongs in, fix it there, and re-commit.

- [ ] **Step 1: Cold-start smoke**

1. Force-reload the page (Cmd+Shift+R) to clear any stuck state.
2. The starting choice card appears, dimmed playfield behind it, no enemies visible yet.
3. Side panel shows BODY 3, Ability 0, Kills 0.

- [ ] **Step 2: Try each starting segment**

For each of FLAME, MAGNET, SHIELD, ECHO:

1. Press `R`. Card appears.
2. Pick the segment under test.
3. Verify: starting snake reads head + 3 BODY + chosen segment in the chain visualization at the top of the panel.
4. Play for 15 seconds. Confirm the chosen ability does its thing (Flame shoots, Magnet pulses + pulls pickups, Shield shockwaves damage enemies, Echo just sits there since nothing is in front of it).
5. Confirm head's white shot fires every ~30 ticks regardless of starting pick.

- [ ] **Step 3: Build a chain via `C` card and watch combos**

1. Press `R`, pick FLAME.
2. Press `C`, pick ECHO. Chain reads `[head][BODY][BODY][BODY][FLAME][ECHO]`. Flame fires 2 projectiles per trigger.
3. Press `C`, pick ECHO again. Flame fires 3 projectiles per trigger.
4. Press `C`, pick MAGNET. Magnet is at the new tail, behind the 2 Echoes. It pulses and pulls pickups; no chain interaction with Flame (the Echoes are between Flame and Magnet).

- [ ] **Step 4: Damage flow + death overlay**

1. Walk into enemies one hit at a time. Watch BODY 3→2→1→0, then Ability decrements from tail (3→2→1→0).
2. Final hit: head alone, DEAD overlay appears with survive time + kill count.
3. Press anything other than `R` — overlay stays. Press `R` — restart card appears.

- [ ] **Step 5: Enemy tuning**

1. Restart with FLAME. Press `;` until Enemies stat reads 0. Enemy population drains.
2. Press `'` until Enemies reads 18. Population swarms.
3. Confirm the snake can still take hits and die normally under load.

- [ ] **Step 6: Final commit (if any fixes were made)**

If Steps 1–5 surfaced bugs, fix them in the appropriate task's file region and commit per-task. Run the full smoke again. If clean, no commit needed.

If everything passes:

```bash
git log --oneline
```

You should see 7 task commits + the initial commit. Confirm the working tree is clean:

```bash
git status
```

Expected: `nothing to commit, working tree clean`.

- [ ] **Step 7: Push to origin (optional)**

If you want the playground state on GitHub:

```bash
git push origin main
```

---

## What's NOT in this plan (per the spec)

These are deliberately deferred and will be planned in a future spec/plan cycle:

- Bounded walls + wall-damage (replaces torus)
- Room transitions, multi-room state, cooldown carry-over
- Doors as kill-gated segment-choice mechanic (replaces `C` card)
- Procedural generation for 5-room run
- Beefy chase boss for room 5
- Win/death overlays with full run-summary screens
- Meta-progression / segment unlocks
- Solid self-collision (snake body blocks its own head)
- Additional enemy types beyond the existing chase mob
- Coil / Mirror / Toxin / etc. segment types beyond the existing 4

After this plan's playground answers the combat-feel question, re-enter brainstorming for the MVP build (the broader scope locked in the earlier session).
