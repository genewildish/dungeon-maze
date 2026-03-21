# Maze Game

A meditative browser-based maze explorer. Navigate procedurally-generated perfect mazes with checkpoint safety nets, optional trail tracking, and an atmospheric lamplight mode filled with spirits.

**No build step. No dependencies. Open `index.html` and play.**

**[🌐 Play in Browser](https://htmlpreview.github.io/?https://github.com/genewildish/dungeon-maze/blob/main/index.html)** · **📱 Mobile-friendly with on-screen D-pad**

---

## Quick Start

1. Open `index.html` in a modern browser (Chrome, Firefox, Safari, Edge)
2. Toggle options on the start screen:
   - **Show Trail** — visualize cells you've visited
   - **Checkpoints** — auto-advance through solution path for safe resets
   - **Lamplight** — see only around your character; encounter spirits and monsters
3. Press START and use arrow keys to move
4. Find the exit (glowing and undulating at the edge)

---

## ClockworkPi / Handheld Devices

The game runs on **ClockworkPi uConsole** and similar handheld Linux devices with Chromium.

### Launch on ClockworkPi uConsole

- **Desktop:** Double-click the **Maze** shortcut on the desktop
- **Terminal:** Run `~/games/maze/launch.sh`
- The game launches in fullscreen kiosk mode and scales to fill the 1280×720 screen

### Controls on uConsole

- **D-pad arrows** — move one cell at a time
- **D-pad center** (or **Enter** key) — start game / play again
- **Escape** key — exit to desktop (the game puns: *"press escape to escape"*)
- **Physical keyboard** — arrow keys work alongside D-pad

---

## Mobile / Phone / Tablet

The game detects touch devices and shows a **D-pad in the bottom-left corner** for easy mobile play.

### Launch on Your Phone

**Easiest:** Open this link in your mobile browser:
- [htmlpreview.github.io](https://htmlpreview.github.io/?https://github.com/genewildish/dungeon-maze/blob/main/index.html)
- Or download the repo and open `index.html` locally

### Touch Controls

- **Arrow buttons** (▲ ▼ ◀ ▶) — tap to move one cell
- **Checkboxes & START button** — tap to toggle options and begin
- **HELP button** — tap during play to reset to checkpoint
- **Escape key** — return to menu (on devices with a keyboard)

The D-pad only appears on touch-capable devices; desktop browsers show arrow keys and gamepad as before.

---

## Gameplay

### Core Loop
- **Arrow keys** move you one cell at a time
- **Walls block movement** — respecting the grid's solid boundaries
- **Exit** glows at a random edge each game (not always right side)
- **Entrance** glows at a different random edge

### Checkpoints (Optional)
- **Silent safety net** — as you move along the solution path, your checkpoint advances
- **At dead ends** (surrounded by 3 walls + only visited cells beyond) — auto-teleport back to your last checkpoint with shimmer effect
- **Manual reset** — press HELP button anytime to checkpoint-teleport during play
- **Reset counter** — each checkpoint cell shows how many times you've been sent back there; sidebar tallies total resets (lower is better)

### Trail (Optional)
- **Breadcrumb visualization** — colored squares mark cells you've walked through
- **Clears on restart** — fresh slate when you return to a checkpoint
- **Memory aid** — helps you recognize what you've already explored

### Win Condition
- **Reach the exit** cell and step out through its wall
- **Celebration** — sparkle bursts and "YOU WIN!" screen
- **Play Again** resets the entire game and generates a fresh maze

---

## Modes

### Standard Mode
Navigate the maze visually. All walls visible, trail optional. Pure exploration with a safety net.

### Lamplight Mode ✨

A solitary journey through darkness, lit only by a small lantern around you.

#### Mechanics
- **Invisible walls** except where they touch your recent trail (fading over ~20 steps) or within your lamplight glow radius (3 cells, fading by distance)
- **Random wall holes** — ~12 gaps punch through walls to create escape routes
- **Monsters** — 3 soot spirits with red eyes roam the maze
  - **In darkness**: Only see their glowing red eyes
  - **In your lamplight**: Dark soot balls with red eyes
  - **On touch**: Reset to the beginning (respawns monsters)
- **Fairies** — 3 ethereal spirits with green glowing eyes wander peacefully
  - **In darkness**: Glowing green eyes drifting
  - **In your lamplight**: Soft green translucent sprites
  - **Their own aura**: 2-cell green glow that illuminates walls nearby
  - **On touch**: Full-map illumination for 1.5 seconds (green light fading), then the fairy vanishes
  - **Purpose**: Help you memorize the maze layout before monsters catch you
- **Entry** always glows steady green
- **Exit** undulates with a beckoning pulse

#### Strategy
Use fairies to scout the maze layout, find shortcuts through wall holes, and evade monsters. The fewer resets, the better.

---

## How Mazes are Generated

Perfect maze algorithm in three phases:

### Phase 1: Solution Path
- **Biased random walk** from a random entry cell toward a random exit cell
- **Directional bias** adapts: favors directions that reduce Manhattan distance to the exit
- **Self-avoiding** — backtracks if cornered, never revisits a cell
- **Guaranteed minimum length** — if entrance and exit are too close, regenerate until the path reaches the diagonal distance (48 cells on a 25×25 grid)

### Phase 1.5: Wall Holes (Lamplight Only)
- Punch ~12 random gaps in walls to create alternative routes
- Helps players escape monsters and explore

### Phase 2: Fill Phase
- **Recursive backtracker** carves from shuffled solution path cells, ensuring every corridor branches off the solution
- **DFS exploration** creates naturally varied-length corridors — some long detours, some short dead ends
- **Result**: Organically deceptive alternate routes that feel like real options

### Perfect Maze Properties
- **Every cell is reachable** from the entrance
- **No loops** — only one path between any two cells
- **Visually diverse** — solution path winding, fill corridors branching in all directions

---

## Features

### Checkpoint Reset Tracker
Each time you return to a checkpoint via dead-end teleport or HELP button, that cell displays an incrementing counter (small pip with white number). The sidebar **RESETS** tally shows your total across all checkpoints—lower is better.

### Player Glow (Lamplight)
Your character emits a 3-cell radius of light, fading with Manhattan distance. Walls and monsters within this radius become visible.

### Exit Beacon
The exit portal gently pulses with a sine wave (alpha 0.3–1.0), beckoning you forward in both standard and lamplight modes.

### Particle Effects
- **Movement sparkles** — small amber flecks when you walk
- **Checkpoint teleport** — green shimmer burst when resuming
- **Fairy touch** — golden celebration burst (same as winning)
- **Win celebration** — large golden sparkle fireworks

---

## Project Structure

```
maze/
├── index.html              # Entire game (vanilla JS, ~35KB)
├── README.md              # This file
├── claude.md              # Developer collaboration guide
├── ideas.md               # Future feature concepts
└── docs/
    └── superpowers/
        ├── specs/
        │   ├── 2026-03-17-maze-game-design.md
        │   └── 2026-03-18-decoy-branches-design.md
        └── plans/
            └── 2026-03-18-decoy-branches.md
```

### Code Organization (in `index.html`)
- **Constants & State** — grid size, colors, game state variables
- **Grid & Maze Generation** — generation algorithms and helpers
- **Rendering** — canvas drawing (walls, maze, trails, particles, UI)
- **Player & Input** — movement, collision, input handling
- **Checkpoint System** — tracking progress along solution
- **Dead End Detection** — 3-wall structural analysis
- **Monsters & Fairies** — spawning, AI, collision
- **Particle System** — lightweight effects
- **Game Loop** — main animation frame and state updates

---

## Future Ideas

Concepts for future development (see `ideas.md`):

- **Mazes above and below** — multi-level dungeon with vertical traversal
- **Story mode** — narrative framing ("please help us", Stanley Parable vibes)
- **Dimensional shift monsters** — purple spiraling entities that, on touch, transform the maze to a new configuration while keeping entrance/exit; visual preview of the new solution path as they approach
- **Dynamic difficulty** — scaling monster/fairy counts based on player performance
- **Leaderboard tracking** — persistent high scores for fewest resets
- **Sound design** — ambient atmosphere, footsteps, monster sounds, fairy chimes
- **Accessibility modes** — colorblind-friendly palettes, high-contrast trails

---

## Technical

- **Single-file codebase** — ~35KB vanilla JavaScript in `index.html`
- **Canvas rendering** — 200×200 logical grid, scaled 3× to 600×600 CSS pixels for crisp pixel-art look
- **No dependencies** — pure browser APIs (ES6+)
- **Performance optimized** — efficient wall drawing, particle culling, minimal GC pressure
- **Modern browsers** — Chrome, Firefox, Safari, Edge (2020+)

### Development

Read `claude.md` for collaboration guidelines, code style, and testing practices.

For detailed architecture, see the design docs in `docs/superpowers/specs/`.

---

## License & Credits

A meditation on maze navigation, built with care. Share, fork, and remix freely.

Enjoy the quiet. Mind the shadows. Follow the green lights home.
