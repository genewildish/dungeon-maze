# 3D First-Person Raycaster View — Design Spec

**Date:** 2026-03-21
**Status:** Draft

## Overview

Add an always-on-top first-person 3D view to the maze game using Wolfenstein-style DDA raycasting. The view sits above the existing 2D maze as a wide banner, updates in real-time as the player moves, and renders using the existing wall grid data. Toggled via a start-screen checkbox (off by default).

Future versions will add information to wall surfaces (text, icons, etc.), so the rendering approach must treat walls as drawable surfaces, not just colored lines.

## Decisions

| Question | Decision | Rationale |
|----------|----------|-----------|
| Purpose | Always-on companion (split-screen) | 3D view visible alongside 2D maze during gameplay |
| Camera orientation | Last-move facing | Camera snaps to face the direction of the player's last movement (N/S/E/W). No manual rotation controls. |
| Layout | Wide banner above maze | Same width as maze, ~40% of maze height. Enough wall surface for future content. |
| Visual style | Procedural textures + distance fog | Brick patterns drawn in code (no image assets) with brightness falloff by distance. Performant combo: fog limits detail work on far walls. |
| Toggle | Start-screen checkbox | Matches existing toggle pattern (trail, checkpoint, lamplight). Off by default. ClockworkPi can leave it off. |
| FOV | 90° | Classic Wolfenstein feel. Good balance of immersion and performance for 1-cell-wide corridors. |

## Architecture

### Raycasting Engine

The 3D view uses a DDA (Digital Differential Analyzer) raycaster adapted for edge-based walls.

**Key difference from classic Wolfenstein:** The original Wolfenstein used solid/empty cells — a cell is either a wall or open space. This maze stores walls on cell *edges* (`grid[y][x].walls.n/s/e/w`), meaning each cell is open space with walls selectively placed on its four sides. The DDA must be modified accordingly: instead of checking "is this cell solid?", each grid-line crossing checks "does this edge have a wall?"

**Modified DDA collision test:**

When a ray crosses from cell `(x, y)` into an adjacent cell, it checks the wall on that shared edge:
- Moving east (x → x+1): check `grid[y][x].walls.e`
- Moving west (x → x-1): check `grid[y][x].walls.w`
- Moving south (y → y+1): check `grid[y][x].walls.s`
- Moving north (y → y-1): check `grid[y][x].walls.n`

If the wall exists, the ray stops and we know both the hit distance and which face was struck. The face orientation (horizontal vs vertical grid line) is determined by whether the last DDA step was in X or Y — this is used for wall shading (see Rendering).

**How it works:**

1. A second `<canvas>` element sits above the maze canvas.
2. Each frame, the raycaster casts one ray per column of the 3D canvas (~200 rays for 200px native width).
3. Each ray steps through the grid, checking wall edges at each grid-line crossing as described above.
4. When a ray hits a wall edge, the perpendicular distance determines the wall column height (closer = taller).
5. The procedural brick texture is drawn per-column with brightness modulated by distance (fog effect).
6. The raycaster runs inside the existing `requestAnimationFrame` game loop — no separate loop.

**Camera state:**

- Position: center of `player.x, player.y` cell (i.e., `player.x + 0.5, player.y + 0.5` in grid coordinates)
- Facing: stored in a new `player.facing` field — set to the direction of each successful move
- Initial facing: opposite of `entrance.wall` (e.g., if entrance is on the west edge, face east). Set at game start.
- Facing angle mapping: E→0°, S→90°, W→180°, N→270°
- FOV: 90°
- No player rotation controls — direction changes only via movement

**Performance characteristics:**

- 200 rays × DDA traversal on a 25×25 grid: each ray hits a wall in at most ~25 steps
- Procedural textures computed per-column (vertical strip), not per-pixel
- Distance fog means far columns rendered as solid dark color — texture work skipped
- Estimated addition: ~250-350 lines of JS

### Canvas & Layout

**3D Canvas:**

- Native resolution: 200×80 pixels (same width as maze canvas, 40% height)
- `image-rendering: pixelated` / `crisp-edges` to match pixel art aesthetic
- CSS sizing: uses `aspect-ratio: 200 / 80` with width matching the maze canvas via the same `min()` expression, so the 3D canvas always tracks the maze width regardless of viewport
- When toggle is off: `display: none`, layout collapses to current look

**DOM structure:**

```
#game-container
  #canvas-3d        ← NEW (200×80, above maze)
  #maze              ← existing (200×200)
  .overlay           ← existing overlays (positioned over #game-container)
```

**Overlay behavior:** Existing overlays use `position: absolute` on `#game-container`. With the 3D canvas added, overlays will cover both canvases. This is acceptable — the start screen, dead-end dialog, and win celebration are full-game-state overlays. When the 3D toggle is off, the 3D canvas is `display: none` and overlays cover only the maze as before.

**Responsive behavior:**

- The 3D canvas width is coupled to the maze canvas width via the same CSS `min()` sizing; `aspect-ratio` handles height automatically
- Portrait phone media query accounts for the taller combined layout when 3D is active
- ClockworkPi sizing logic accounts for the extra canvas height when `fpViewEnabled` is true

### Rendering

**Walls:**

- Procedural brick pattern: horizontal mortar lines within each wall column, with alternating horizontal offset per row to create brick coursing. Exact brick dimensions tuned during implementation (the 80px native canvas height means wall columns range from ~10px at far distances to ~80px up close).
- Drawn as `fillRect` calls — no `drawImage`, no texture image assets
- Wall face shading determined by DDA step direction: vertical grid-line hits (E/W faces) are full brightness, horizontal grid-line hits (N/S faces) are ~70% brightness. This is the classic Wolfenstein depth cue.
- Brightness further modulated by distance: `brightness = 1 / (1 + distance * fogFactor)`
- Beyond ~8 cells distance, walls are nearly black (fog limit)

**Floor & Ceiling:**

- Solid colors, no floor-casting or ceiling-casting (expensive and unnecessary for v1)
- Ceiling: `COLORS.bg` (#0f0f23)
- Floor: `COLORS.corridor` (#1a1a2e)

**Color palette:**

- Walls: derived from existing `COLORS.wall` (#4a6741) and `COLORS.wallHighlight` (#7ec875), tinted by distance and face orientation
- Integrates with the existing COLORS object — no separate color definitions

### Integration with Existing Systems

**New state:**

- `fpViewEnabled` boolean — toggle state, `false` by default
- `player.facing` — last movement direction, initialized to opposite of `entrance.wall`

**Lamplight mode:**

- When lamplight is active, fog distance shortened dramatically (e.g., 3-4 cells instead of 8)
- Creates a claustrophobic first-person feel that matches the restricted 2D visibility
- Thematic consistency for free

**Monsters/Fairies:**

- Not rendered in the 3D view for v1
- The 2D view remains the authoritative gameplay view

**Start screen:**

- New checkbox "3D View" in the toggle group
- Off by default
- State stored in `fpViewEnabled`

**Game state transitions:**

- 3D canvas renders during `gameState === 'playing'` and during the win celebration interval (before the win overlay appears), so the player sees their final view
- When `gameState` transitions to `'start'` (new game / play again), the 3D canvas is cleared to black
- Rendering skipped entirely when `fpViewEnabled` is false (no wasted cycles)

**Game loop integration:**

- `render3DView()` called from `gameLoop()`, not interleaved with `render()` — since the 3D view draws to its own canvas, ordering relative to 2D draw calls is irrelevant
- Gated on `fpViewEnabled && (gameState === 'playing' || gameState === 'win')`

## Future Considerations

This design intentionally leaves room for:

- **Wall content:** The procedural texture approach renders wall faces as surfaces. Future versions can replace or overlay the brick pattern with text, icons, or other information per-wall-face.
- **Camera rotation:** If manual look-around is needed later (e.g., to read wall content), adding Q/E rotation controls would require only changing `player.facing` to a continuous angle and adding key bindings — the raycaster already works with arbitrary angles.
- **Entity rendering:** Monsters/fairies could be rendered as billboard sprites in the 3D view using standard raycaster sprite-casting techniques.

## Estimated Impact

- **File size:** ~250-350 new lines of JS, ~20 lines of CSS, ~5 lines of HTML
- **Performance:** Negligible on modern hardware. The raycaster is O(rays × grid_size) per frame, which is ~5000 grid cell checks — trivial compared to canvas draw calls.
- **Layout:** Reversible via toggle. No impact when disabled.
