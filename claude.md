# Claude.md — Maze Game Developer Guide

## Quick Overview

A browser-based maze game built with vanilla JavaScript in a single `index.html` file. No build step, no dependencies. Open and play.

**Tech:** HTML5 Canvas, vanilla JS, ~2000 lines
**Visual Style:** Pixel art (8x8 cells, 25x25 grid) with particle effects
**Play Target:** Modern browsers (Chrome, Firefox, Safari, Edge)

---

## How We Work Together

### 1. Keep the Single-File Structure Clean
- **No refactoring into modules.** The single `index.html` is the constraint.
- Organize code into logical sections with clear comment headers (see existing pattern: `=== SECTION NAME ===`).
- Group related functions together.
- Use function names that are clear about what they do and what system they belong to.

### 2. Verify Before Claiming Done
When you ask me to add a feature, fix a bug, or make a change:
- I **will** run the game and test it works before telling you it's complete.
- I will confirm the specific behavior you requested actually happens.
- If something breaks, I'll fix it before reporting back.

### 3. Code Style: Match What You See, Comment Forward
- **Naming:** camelCase for functions and variables. System-related functions have implicit prefixes (e.g., `generateSolutionPath`, `drawMaze`, `updateParticles`).
- **Organization:** Constants at top, helper functions near their consumers, game loop last.
- **Comments:** Be generous. Explain *why* code does something, not just *what* it does. Especially for algorithmic logic (maze generation, collision detection, particle updates).
- **Code formatting:** Match the existing style (spacing, brace placement, ternary operators).

### 4. Performance: Browser-Lean, Hardware-Conservative
- **Liberal with browser APIs:** Use whatever Canvas APIs, DOM methods, and JavaScript features work well in modern browsers. Don't worry about IE11.
- **Conservative with hardware:** Don't waste CPU cycles or memory. Optimize particle counts, canvas redraws, and garbage generation.
- Example: It's fine to create new arrays in the game loop if they're small; don't create thousands of particles per frame.

### 5. Git: Commit Freely, But No Pushes or PRs
- I will commit my changes with clear messages.
- **You handle pushes and pull requests.** I won't push or open PRs without your explicit approval.

---

## The Three Main Systems (Quick Reference)

### 1. Maze Generation
- **Solution Path Phase:** Biased random walk (East 40%, North/South/West 20% each) from left to right edge. Self-avoiding, backtracks if cornered.
- **Fill Phase:** Recursive backtracker from unvisited cells carves corridors and dead ends.
- **Result:** Perfect maze (every cell reachable, no loops) with hidden solution.
- **See:** `generateSolutionPath()`, `generateMaze()` in index.html; full spec in `docs/superpowers/specs/2026-03-17-maze-game-design.md`.

### 2. Gameplay Loop
- **Movement:** Arrow keys move player one cell at a time (checked against wall bits before move).
- **Trail:** Optional visited-cell tracking (can be toggled at start).
- **Checkpoint:** Silent safety net — advances when player's cell matches next solution path cell in sequence.
- **Dead End:** Auto-detected when player has only one passage to a visited cell. Offers "Restart" or "Resume from Checkpoint."
- **Win:** Reaching the exit cell (right edge) triggers celebration.
- **See:** `handleInput()`, `movePlayer()`, `checkDeadEnd()`, `checkWin()`, game loop in `requestAnimationFrame`.

### 3. Particle System
- **Lightweight:** Particles have position, velocity, lifetime (0–1 fade), color, size.
- **Uses:** Movement trail sparkles, win celebration burst, checkpoint teleport shimmer.
- **Update:** Every frame, particles age, move, and are removed when `life <= 0`.
- **See:** `Particle` data structure, `updateParticles()`, `drawParticles()`, particle spawning in movement/win/checkpoint code.

---

## Running & Testing the Game

1. **Open the game:**
   Double-click `index.html` or run a local web server (`python -m http.server` in the repo root).

2. **What "verified working" means:**
   - Game loads without console errors.
   - Start screen appears with toggles and START button.
   - Arrow keys move the player; walls block movement correctly.
   - Trail shows visited cells (if enabled) and clears on restart.
   - Checkpoint system advances silently and "Resume from Checkpoint" works.
   - Dead end detection triggers correctly.
   - Win condition fires when reaching the exit; sparkles play.
   - New game generates without errors.

---

## Where to Find Things

| What | Where |
|------|-------|
| **Game Architecture & Systems** | `docs/superpowers/specs/2026-03-17-maze-game-design.md` |
| **Implementation Milestones** | `docs/superpowers/plans/2026-03-17-maze-game.md` |
| **The Actual Code** | `index.html` (single file, ~2000 lines) |
| **Collaboration Style** | This file (`claude.md`) |

---

## Notes

- The canvas is 200x200 pixels (25×25 cells at 8 px each), scaled to 600×600 via CSS for pixel-perfect rendering.
- Player starts at the left edge entry; exits at the right edge.
- All wall state is stored as booleans in `grid[y][x].walls { n, s, e, w }`.
- Solution path is an ordered array `solutionPath = [[x0, y0], ...]` — never shown to the player, only used for checkpoint logic.
