# 3D Raycaster View Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Wolfenstein-style first-person 3D view above the existing 2D maze, rendering in real-time as the player moves.

**Architecture:** A second canvas (200×80) sits above the maze canvas. A DDA raycaster adapted for edge-based walls casts 200 rays per frame, drawing textured wall columns with distance fog. Camera faces the player's last movement direction. The whole system is gated behind a start-screen toggle (off by default).

**Tech Stack:** HTML5 Canvas 2D, vanilla JS (no dependencies)

**Spec:** `docs/superpowers/specs/2026-03-21-3d-raycaster-view-design.md`

---

### Task 1: Add 3D canvas element and CSS

**Files:**
- Modify: `index.html:49-57` (CSS — canvas styles)
- Modify: `index.html:122-135` (CSS — portrait media query)
- Modify: `index.html:170-171` (HTML — add canvas before maze)

This task adds the hidden 3D canvas to the DOM and its CSS. It should be invisible by default (`display: none`), producing zero visual change until toggled.

- [ ] **Step 1: Add CSS for the 3D canvas**

Insert after the existing `canvas { ... }` block (after line 57). The 3D canvas needs its own rule since it has different height behavior (aspect-ratio instead of square). The existing `canvas` rule still applies shared properties (display, image-rendering).

```css
#canvas-3d {
  display: none; /* hidden until toggled on */
  aspect-ratio: 200 / 80;
  /* Width matches maze canvas — same min() logic, height derived from aspect-ratio */
  width: min(600px, calc(100vw - 1rem), calc(100dvh - 1rem));
}
#canvas-3d.active {
  display: block;
}
```

- [ ] **Step 2: Add the canvas element to HTML**

Insert a new `<canvas>` just before the existing maze canvas (line 171), inside `#game-container`:

```html
<canvas id="canvas-3d" width="200" height="80"></canvas>
```

- [ ] **Step 3: Verify no visual change**

Open `index.html` in a browser. The game should look and behave identically — the 3D canvas is hidden. Inspect the DOM to confirm `#canvas-3d` exists with `display: none`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add hidden 3D canvas element and CSS"
```

---

### Task 2: Add start-screen toggle and state wiring

**Files:**
- Modify: `index.html:184-187` (HTML — toggle group in start screen)
- Modify: `index.html:266-268` (JS — state variables, after `lamplightEnabled`)
- Modify: `index.html:1205-1208` (JS — `startGame()` reads toggle)
- Modify: `index.html:1257-1262` (JS — `playAgain()` hides 3D canvas)
- Modify: `index.html:245-256` (JS — ClockworkPi sizing)

- [ ] **Step 1: Add toggle checkbox to start screen HTML**

Insert a new toggle group after the lamplight toggle and before the START button (between lines 186 and 187):

```html
<div class="toggle-group">
  <input type="checkbox" id="fp-toggle">
  <label for="fp-toggle">3D View</label>
</div>
```

- [ ] **Step 2: Add `fpViewEnabled` state variable**

Insert after `let lamplightEnabled = false;` (line 268):

```js
let fpViewEnabled = false;
```

- [ ] **Step 3: Wire toggle into `startGame()`**

In `startGame()`, after the line that reads `lamplightEnabled` (line 1208), add:

```js
fpViewEnabled = document.getElementById('fp-toggle').checked;
```

Then after `gameState = 'playing';` (line 1234), add the logic to show/hide the 3D canvas and set initial facing:

```js
// Show/hide 3D view canvas
const canvas3d = document.getElementById('canvas-3d');
if (fpViewEnabled) {
  canvas3d.classList.add('active');
  // Initial facing: opposite of entrance wall (face inward)
  const oppositeDirs = { n: 's', s: 'n', e: 'w', w: 'e' };
  player.facing = oppositeDirs[entrance.wall];
} else {
  canvas3d.classList.remove('active');
}
```

- [ ] **Step 4: Wire `playAgain()` to hide 3D canvas**

In `playAgain()`, after `particles = [];` (line 1259), add:

```js
document.getElementById('canvas-3d').classList.remove('active');
```

- [ ] **Step 5: Add `player.facing` to player object**

In the player object initialization (line 263), add `facing: 'e'`:

```js
let player = { x: 0, y: 0, lastCheckpointIndex: 0, trail: [], visitedSet: new Set(), health: 2, invulnerable: 0, facing: 'e' };
```

- [ ] **Step 6: Set `player.facing` on each move**

In `movePlayer()`, after `player.y = ny;` (line 721), add:

```js
player.facing = dir;
```

- [ ] **Step 7: Verify toggle appears and state works**

Open the game. Confirm: the "3D View" checkbox appears on the start screen. When checked and START pressed, the 3D canvas element should become visible (empty/transparent). When unchecked, it stays hidden.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add 3D View toggle and fpViewEnabled state wiring"
```

---

### Task 3: Implement the DDA raycaster core

**Files:**
- Modify: `index.html` (JS — new section before `// === GAME LOOP ===`)

This is the core rendering engine. It adds a `render3DView()` function that casts rays and draws wall columns. No textures yet — flat colored walls with distance fog and face shading.

- [ ] **Step 1: Add the raycaster section**

Insert a new section before `// === GAME LOOP ===` (before line 1180):

```js
// === 3D RAYCASTER ===

const canvas3d = document.getElementById('canvas-3d');
const ctx3d = canvas3d.getContext('2d');
const FP_W = canvas3d.width;   // 200
const FP_H = canvas3d.height;  // 80
const FOV = Math.PI / 2;       // 90 degrees
const HALF_FOV = FOV / 2;
const MAX_DEPTH = 25;           // max ray distance in cells

// Direction angles: E=0, S=π/2, W=π, N=3π/2
const DIR_ANGLES = { e: 0, s: Math.PI / 2, w: Math.PI, n: Math.PI * 1.5 };

// Parse hex color to RGB components for brightness modulation
function hexToRgb(hex) {
  const n = parseInt(hex.slice(1), 16);
  return [(n >> 16) & 255, (n >> 8) & 255, n & 255];
}
const WALL_RGB = hexToRgb(COLORS.wall);
const WALL_HI_RGB = hexToRgb(COLORS.wallHighlight);

function render3DView() {
  // Clear with ceiling (top half) and floor (bottom half)
  ctx3d.fillStyle = COLORS.bg;
  ctx3d.fillRect(0, 0, FP_W, FP_H / 2);
  ctx3d.fillStyle = COLORS.corridor;
  ctx3d.fillRect(0, FP_H / 2, FP_W, FP_H / 2);

  const playerAngle = DIR_ANGLES[player.facing] || 0;
  // Camera position: center of the player's cell
  const px = player.x + 0.5;
  const py = player.y + 0.5;

  // Fog factor: tighter in lamplight mode for claustrophobic feel
  const fogFactor = lamplightEnabled ? 0.6 : 0.15;

  for (let col = 0; col < FP_W; col++) {
    // Ray angle: sweep from -HALF_FOV to +HALF_FOV across the screen
    const rayAngle = playerAngle - HALF_FOV + (col / FP_W) * FOV;
    const sinA = Math.sin(rayAngle);
    const cosA = Math.cos(rayAngle);

    // --- DDA setup ---
    // Current grid cell
    let mapX = Math.floor(px);
    let mapY = Math.floor(py);

    // Ray direction sign: which way we step through the grid
    const stepX = cosA >= 0 ? 1 : -1;
    const stepY = sinA >= 0 ? 1 : -1;

    // Distance along ray to next vertical / horizontal grid line
    // tMaxX/tMaxY: parametric t where ray crosses next grid line
    // tDeltaX/tDeltaY: parametric t between consecutive grid lines
    const tDeltaX = cosA !== 0 ? Math.abs(1 / cosA) : 1e30;
    const tDeltaY = sinA !== 0 ? Math.abs(1 / sinA) : 1e30;

    let tMaxX, tMaxY;
    if (cosA >= 0) {
      tMaxX = (mapX + 1 - px) * tDeltaX;
    } else {
      tMaxX = (px - mapX) * tDeltaX;
    }
    if (sinA >= 0) {
      tMaxY = (mapY + 1 - py) * tDeltaY;
    } else {
      tMaxY = (py - mapY) * tDeltaY;
    }

    // --- DDA traversal ---
    let hit = false;
    let side = 0; // 0 = vertical grid line (E/W face), 1 = horizontal (N/S face)
    let dist = 0;

    for (let step = 0; step < MAX_DEPTH * 2; step++) {
      if (tMaxX < tMaxY) {
        // Step in X direction — check wall on the vertical grid edge
        const wallDir = stepX > 0 ? 'e' : 'w';
        if (mapX >= 0 && mapX < COLS && mapY >= 0 && mapY < ROWS &&
            grid[mapY][mapX].walls[wallDir]) {
          dist = tMaxX;
          side = 0;
          hit = true;
          break;
        }
        tMaxX += tDeltaX;
        mapX += stepX;
      } else {
        // Step in Y direction — check wall on the horizontal grid edge
        const wallDir = stepY > 0 ? 's' : 'n';
        if (mapX >= 0 && mapX < COLS && mapY >= 0 && mapY < ROWS &&
            grid[mapY][mapX].walls[wallDir]) {
          dist = tMaxY;
          side = 1;
          hit = true;
          break;
        }
        tMaxY += tDeltaY;
        mapY += stepY;
      }

      // Out of bounds — treat as wall at max depth
      if (mapX < 0 || mapX >= COLS || mapY < 0 || mapY >= ROWS) {
        dist = MAX_DEPTH;
        hit = true;
        side = 0;
        break;
      }
    }

    if (!hit) continue;

    // Fix fisheye: use perpendicular distance
    const perpDist = dist * Math.cos(rayAngle - playerAngle);
    if (perpDist < 0.01) continue;

    // Wall column height (projection)
    const wallHeight = Math.min(FP_H, Math.floor(FP_H / perpDist));
    const wallTop = Math.floor((FP_H - wallHeight) / 2);

    // Brightness: face shading (N/S darker) + distance fog
    const faceBrightness = side === 1 ? 0.7 : 1.0;
    const fogBrightness = 1 / (1 + perpDist * fogFactor);
    const brightness = faceBrightness * fogBrightness;

    // Choose wall color: blend wall and wallHighlight based on brightness
    const rgb = brightness > 0.5 ? WALL_HI_RGB : WALL_RGB;
    const r = Math.floor(rgb[0] * brightness);
    const g = Math.floor(rgb[1] * brightness);
    const b = Math.floor(rgb[2] * brightness);

    ctx3d.fillStyle = 'rgb(' + r + ',' + g + ',' + b + ')';
    ctx3d.fillRect(col, wallTop, 1, wallHeight);
  }
}
```

- [ ] **Step 2: Wire `render3DView()` into the game loop**

In `gameLoop()`, after `render();` (line 1197), add:

```js
if (fpViewEnabled) render3DView();
```

- [ ] **Step 3: Clear 3D canvas in `playAgain()`**

In `playAgain()`, after the line that removes the `active` class (added in Task 2), add:

```js
ctx3d.clearRect(0, 0, FP_W, FP_H);
```

Since `playAgain()` is defined before the raycaster section in the file, the raycaster constants (`ctx3d`, `FP_W`, `FP_H`) aren't declared yet at that point. However, they are `const` declarations (not function-scoped), so they ARE accessible at call time via hoisting within the `<script>` block. But to be safe and explicit, reference the element directly:

```js
const c3d = document.getElementById('canvas-3d');
c3d.getContext('2d').clearRect(0, 0, c3d.width, c3d.height);
```

- [ ] **Step 4: Verify flat-colored raycaster works**

Open the game, check "3D View", start. You should see a first-person corridor view above the maze with flat green-tinted walls. Move around — the view should update, walls should get darker with distance. Verify:
- Walls render at correct heights (close walls tall, far walls short)
- Camera direction changes when you move N/S/E/W
- No fisheye distortion (walls should appear straight, not bowed)
- Performance is smooth (60fps)

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Implement DDA raycaster with flat walls and distance fog"
```

---

### Task 4: Add procedural brick texture

**Files:**
- Modify: `index.html` (JS — inside `render3DView()`, replace the flat wall column drawing)

Replaces the single `fillRect` per column with a brick-textured column. Each column is drawn as a series of small rectangles representing brick rows with mortar gaps.

- [ ] **Step 1: Replace flat wall drawing with brick texture**

In `render3DView()`, replace the wall drawing block (the section after brightness calculation, starting with `// Choose wall color`) with:

```js
    // --- Procedural brick texture ---
    // Compute where the ray hit on the wall surface (0..1 across the face)
    let wallX; // fractional position along the wall face
    if (side === 0) {
      // Hit a vertical grid line (E/W face) — use Y position
      wallX = (py + dist * sinA) % 1;
    } else {
      // Hit a horizontal grid line (N/S face) — use X position
      wallX = (px + dist * cosA) % 1;
    }
    if (wallX < 0) wallX += 1;

    // Brick parameters (in wall-texture space, mapped to column pixels)
    const BRICK_ROWS = 4;     // number of brick rows per wall height
    const MORTAR_PX = 1;      // mortar thickness in screen pixels
    const brickH = wallHeight / BRICK_ROWS;

    // Face shading + distance fog
    const faceBrightness = side === 1 ? 0.7 : 1.0;
    const fogBrightness = 1 / (1 + perpDist * fogFactor);
    const brightness = faceBrightness * fogBrightness;

    // Skip detailed texture for very far/dim walls — just draw solid
    if (brightness < 0.08) {
      const r = Math.floor(WALL_RGB[0] * brightness);
      const g = Math.floor(WALL_RGB[1] * brightness);
      const b = Math.floor(WALL_RGB[2] * brightness);
      ctx3d.fillStyle = 'rgb(' + r + ',' + g + ',' + b + ')';
      ctx3d.fillRect(col, wallTop, 1, wallHeight);
      continue;
    }

    // Brick color
    const br = Math.floor(WALL_HI_RGB[0] * brightness);
    const bg2 = Math.floor(WALL_HI_RGB[1] * brightness);
    const bb = Math.floor(WALL_HI_RGB[2] * brightness);
    ctx3d.fillStyle = 'rgb(' + br + ',' + bg2 + ',' + bb + ')';
    ctx3d.fillRect(col, wallTop, 1, wallHeight);

    // Mortar lines (darker) drawn over the brick
    const mr = Math.floor(WALL_RGB[0] * brightness * 0.4);
    const mg = Math.floor(WALL_RGB[1] * brightness * 0.4);
    const mb = Math.floor(WALL_RGB[2] * brightness * 0.4);
    ctx3d.fillStyle = 'rgb(' + mr + ',' + mg + ',' + mb + ')';

    for (let row = 0; row < BRICK_ROWS; row++) {
      // Horizontal mortar line between brick rows
      const mortarY = Math.floor(wallTop + row * brickH);
      if (mortarY >= wallTop && mortarY < wallTop + wallHeight) {
        ctx3d.fillRect(col, mortarY, 1, MORTAR_PX);
      }

      // Vertical mortar: offset every other row by half a brick
      const offset = (row % 2 === 0) ? 0 : 0.5;
      const brickPos = (wallX + offset) % 1;
      // Draw vertical mortar when near the brick edge (within ~10% of face width)
      if (brickPos < 0.08 || brickPos > 0.92) {
        const rowTop = Math.floor(wallTop + row * brickH);
        const rowBot = Math.floor(wallTop + (row + 1) * brickH);
        ctx3d.fillRect(col, rowTop, 1, rowBot - rowTop);
      }
    }
```

- [ ] **Step 2: Verify brick texture renders correctly**

Open the game with 3D View enabled. Walls should now show a brick pattern:
- Horizontal mortar lines divide walls into rows
- Vertical mortar lines alternate offset per row (running bond pattern)
- Bricks are brighter green (`wallHighlight`), mortar is darker
- Far walls fade to solid dark (fog optimization skips texture detail)
- Close walls show clear brick detail

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add procedural brick texture to 3D raycaster walls"
```

---

### Task 5: Handle edge cases and polish

**Files:**
- Modify: `index.html` (JS — `resumeFromCheckpoint()`, `monsterCaughtPlayer()`)
- Modify: `index.html` (CSS — ClockworkPi sizing)

This task handles game state transitions that should update `player.facing`, and ensures the layout works on ClockworkPi.

- [ ] **Step 1: Maintain `player.facing` on checkpoint resume**

In `resumeFromCheckpoint()` (around line 817), after `player.y = cp[1];` (line 821), add:

```js
// Keep current facing direction — don't reset it on checkpoint resume
```

No code change needed — `player.facing` persists through checkpoint resume, which is the correct behavior (the camera stays pointed the same direction).

- [ ] **Step 2: Reset `player.facing` on monster death reset**

In `monsterCaughtPlayer()`, after the full-reset block sets `player.lastCheckpointIndex = 0;` (line 959), add:

```js
const oppositeDirs = { n: 's', s: 'n', e: 'w', w: 'e' };
player.facing = oppositeDirs[entrance.wall];
```

- [ ] **Step 3: Update ClockworkPi sizing for 3D canvas**

In the ClockworkPi sizing block (around line 249-256), after the existing canvas sizing, add handling for the 3D canvas:

```js
if (isClockworkPi) {
  // ... existing code ...
  // 3D canvas sizing handled via CSS aspect-ratio + matching width
  // No additional JS needed — the CSS width rule already uses min()
}
```

Review the CSS: the existing ClockworkPi JS sets `canvas.style.width` and `canvas.style.height` directly on the maze canvas. The 3D canvas uses CSS `aspect-ratio`, so only its `width` needs to match. Add after the maze canvas sizing:

```js
const canvas3dEl = document.getElementById('canvas-3d');
canvas3dEl.style.width = size + 'px';
```

- [ ] **Step 4: Verify edge cases**

Test the following scenarios with 3D View enabled:
- Enable lamplight mode: fog should be much thicker (3-4 cells visibility)
- Get caught by a monster and lose all health: camera should reset to face entrance direction
- Resume from checkpoint: camera should keep its current facing
- Win the game: 3D view should show the final corridor view during celebration
- Play again: 3D canvas should clear when returning to start screen

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Handle edge cases: monster reset facing, ClockworkPi sizing"
```

---

### Task 6: Final verification

- [ ] **Step 1: Full playthrough test**

Play a complete game with 3D View enabled and all features on (trail, checkpoints, lamplight):
- Start screen shows 4 toggles, all checkable
- 3D view appears above maze when toggled on
- Corridor view updates fluidly with each move
- Brick texture visible up close, fading to fog in distance
- Lamplight mode has shorter fog distance
- Win celebration plays while 3D view shows final position
- Play Again returns to clean start screen

- [ ] **Step 2: Test with 3D View OFF**

Play a complete game with 3D View unchecked. Verify zero visual or behavioral difference from the pre-feature game.

- [ ] **Step 3: Test responsive layout**

Resize the browser window. The 3D canvas should scale in lockstep with the maze canvas. Test portrait phone layout if possible (use browser dev tools responsive mode).
