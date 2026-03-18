# Maze Game — Design Spec

## Overview

A browser-based maze game that procedurally generates mazes using a "squiggle method" — the solution path is drawn first, then the surrounding corridors and dead ends are filled in to obscure it. The player navigates the maze using arrow keys with optional trail visibility and checkpoint resume, presented in a retro pixel art style with sparkle particle effects.

## Technology

- **Platform:** Web browser (HTML5 Canvas)
- **Language:** Vanilla JavaScript, no dependencies
- **Delivery:** Single `index.html` file — open and play, no build step or server required

## Maze Generation

The maze is a 25x25 grid. Each cell stores its wall state as 4 bits (N/S/E/W).

All border cells have walls on their border-facing sides, except the entry opening (left edge) and exit opening (right edge).

### Phase 1: Solution Path (the squiggle)

Starting from a cell on the left edge, perform a biased random walk toward the right edge. The path must be **self-avoiding** — if the walk would revisit a cell, that direction is rejected and another is chosen. If the walk paints itself into a corner (all neighbors visited), backtrack along the path until a cell with unvisited neighbors is found and continue from there.

At each step, direction weights are: **East 40%, North 20%, South 20%, West 20%** (of available/unvisited directions, renormalized). This produces organic squiggly paths that trend rightward without being too direct.

The path is stored as an ordered array of `[x, y]` coordinates and never modified after generation. It is the solution to the maze. Walls between consecutive cells in the path are removed during this phase, marking those cells as visited.

### Phase 2: Fill the Rest

From every unvisited cell, run a **recursive backtracker** (randomized depth-first search) that connects into already-visited cells. This carves branching corridors and dead ends throughout the grid, producing a perfect maze (every cell reachable, no loops) where the solution path is hidden among many false paths.

## Player Movement

- **Arrow keys** move the player one cell at a time on the grid. Arrow key default browser behavior (scrolling) is suppressed with `preventDefault`.
- Before each move, the target cell's wall bits are checked. If a wall exists in the requested direction, the move is rejected.
- Movement is grid-snapped. A brief pixel-slide animation between cells is optional for polish but not required.
- The canvas/page captures keyboard focus on load.

## Trail

- **Toggle on:** The player leaves a visible trail of visited cells as they move (like Snake's body). Helps them see where they've been and avoid retreading.
- **Toggle off:** No trail — the player must remember their path.
- Controlled by a toggle on the start screen. Independent of the checkpoint setting.

## Checkpoint System

- The solution path (from Phase 1) is stored as an ordered array of cell coordinates.
- After each player move, the game checks whether the player's current cell is in the solution path. If so, `lastCheckpoint` is updated to that position — but only if this cell is **at or beyond the current checkpoint's position in the solution path sequence**. This prevents the checkpoint from jumping forward to a solution-path cell the player stumbled onto out of order. Checkpoints can only advance sequentially along the solution.
- The player never sees the solution path. The checkpoint is a silent safety net.

### Dead End Handling

A dead end is auto-detected when the player is in a topological dead end — a cell with only one passage, and that passage leads to an already-visited cell. In other words, the player literally cannot move to any unvisited cell from where they stand. This is distinct from being in a corridor where all immediate neighbors are visited but further unvisited cells are reachable through them. The player can also manually request help at any time via a button/key, regardless of whether auto-detection has triggered.

When triggered, the player is presented with two options:

- **Restart** — Player returns to the entry cell. Trail is cleared entirely.
- **Resume from checkpoint** — Player teleports to `lastCheckpoint`. Trail from the checkpoint onward is cleared; trail before the checkpoint is preserved.

If the checkpoint toggle is off, only the Restart option is shown.

## Difficulty / Settings

There is one maze size: 25x25. Difficulty is controlled by two independent toggles on the start screen:

| Setting | On | Off |
|---|---|---|
| **Trail** | Visited cells are visually marked | No trail visible |
| **Checkpoint** | "Resume from checkpoint" available on dead end | Only "Restart" available |

No difficulty labels (easy/medium/hard). The player picks their own challenge by toggling these options.

## Visual Style

- **Pixel art** — Each cell is 8x8 pixels, giving an internal canvas resolution of 200x200 for the 25x25 grid. The canvas is scaled up with CSS and `image-rendering: pixelated` for crisp, chunky pixels.
- **Color palette** — Game Boy-inspired but not restricted to 4 colors. Dark background, bright/glowing walls, distinct player character.
- **Retro pixel font** — Used for start screen, toggles, dead-end prompt, and win screen.

### Sparkle Particle System

A lightweight particle system manages sparkle effects:

- **Player movement** — Small sparkle particles trail behind the player as they move between cells.
- **Win (reaching the exit)** — Celebration burst of sparkles.
- **Checkpoint resume** — Teleport shimmer effect at the destination.

Each particle has position, velocity, lifetime, color, and size. The system updates and renders particles each frame, removing expired ones.

## Game Flow

1. **Start screen** — Game title, trail toggle, checkpoint toggle, "Start" button.
2. **Maze generation** — Instant at 25x25. Generate and render immediately.
3. **Gameplay** — Player enters from the left edge, exits on the right edge. Arrow key navigation with optional trail and sparkle effects.
4. **Dead end** — Modal/overlay prompt with "Restart" and optionally "Resume from checkpoint."
5. **Win** — Sparkle celebration, display completion message, "Play Again" button (generates a new maze).

## File Structure

```
maze/
  index.html    — the entire game: HTML, CSS, and JS inline
```

## Data Structures

### Cell Grid
```
grid[y][x] = {
  walls: { n: bool, s: bool, e: bool, w: bool },
  visited: bool  // used during generation
}
```

### Solution Path
```
solutionPath = [[x0, y0], [x1, y1], ..., [xN, yN]]
```

### Player State
```
player = {
  x: int,
  y: int,
  lastCheckpoint: { x: int, y: int },
  trail: [[x, y], ...]  // cells visited in order
}
```

### Particle
```
particle = {
  x: float, y: float,
  vx: float, vy: float,
  life: float,     // 0.0 to 1.0
  color: string,
  size: float
}
```
