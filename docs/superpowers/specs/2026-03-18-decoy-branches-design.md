# Decoy Branches — Design Spec

## Problem

The maze currently generates a solution path (Phase 1) then fills remaining cells with a recursive backtracker (Phase 2). The fill phase creates corridors and dead ends, but they're short and random — the player quickly learns to ignore them. The maze feels deterministic once you develop an intuition for the generation patterns.

## Goal

Add 3 deliberately long decoy branches that fork off the solution path and head toward different edges of the grid, stopping 1-2 cells short. These branches are 50-75% the length of the solution path, making them major time commitments that feel like plausible alternative routes. The maze should feel non-deterministic — like there are multiple real paths competing for the player's attention.

## Design

### Generation Flow

Three phases instead of two:

1. **Phase 1** — `generateSolutionPath()` (unchanged). Biased random walk from left edge to right edge.
2. **Phase 1.5** — `generateDecoyBranches()` (new). Carves 3 long branches off the solution path toward different edges.
3. **Phase 2** — `fillMaze()` (unchanged). Recursive backtracker fills all remaining unvisited cells.

No changes to Phase 1 or Phase 2. Decoy branches mark their cells as visited, so Phase 2 naturally carves around them.

### Branch Point Selection

The solution path is an ordered array of ~40-80 cells. To place 3 well-distributed branch points:

- Divide the solution path into 4 equal segments, creating division points at ~25%, ~50%, and ~75% of the path length.
- Add jitter: pick a random cell within ±3 indices of each division point. Clamp the result to `[2, solutionPath.length - 3]` to avoid placing branches at the very start or end of the solution.
- Each branch point must have at least one unvisited neighbor. If not, slide forward along the solution path (toward the exit) until one is found. If the end is reached without finding one, slide backward from the original point. If no valid branch point exists in that segment (extremely unlikely), skip that decoy.

This prevents branches from clustering together and ensures they span the full length of the solution.

### Target Edge Assignment

Each decoy targets a different edge of the grid. The right (east) edge is excluded since the solution path already exits there — a decoy running parallel to the solution near the exit would be confusing rather than fun. That leaves 3 edges (north, south, west) for 3 branches — one each, no conflicts.

Assignment uses a greedy loop: process branch points in solution-path order. Each picks the farthest available edge from its grid position, measured by Manhattan distance from the branch point to the nearest cell on that edge (e.g., distance to north edge = branch point's `y`, distance to west edge = branch point's `x`). Ties are broken randomly. Since there are exactly 3 edges for 3 branches, each gets a unique target.

Note: the west-targeting decoy starts from a solution path cell that trends eastward, so it may travel farther and is more likely to hit its length cap before reaching the edge. This is acceptable — it still creates a long, convincing detour.

The target cell is placed 1-2 cells from the target edge at a random position along that edge (e.g., targeting the north edge means `y = 1`, random `x`).

### Decoy Walk Algorithm

Each decoy uses the same biased random walk pattern as the solution path:

1. **Start** from the branch point on the solution path. First step goes to an unvisited neighbor.
2. **Wall removal** — each step removes the wall between the current cell and the next, exactly like the solution path walk. This is what creates passable corridors.
3. **Directional bias** toward the target edge: 40% weight for the target direction, 20% each for the other three. Weights are applied only to available (unvisited) neighbors, then sampled from the resulting pool — same filtering approach as `generateSolutionPath()`.
4. **Self-avoiding** — only walks through unvisited cells. Backtracks if cornered. When backtracking reaches the branch point, try the branch point's other unvisited neighbors before giving up. If no unvisited neighbors remain, terminate the decoy with whatever it carved.
5. **Target length** — before each walk begins, pick a random target length uniformly from `[0.5 * solutionPath.length, 0.75 * solutionPath.length]`.
6. **Stop conditions** (whichever comes first):
   - Walk reaches within 1-2 cells of the target edge (success).
   - Walk reaches the target length.
   - Walk exhausts all backtracking options back to the branch point (keep whatever was carved).
7. **Mark** all carved cells as visited so Phase 2 fills around them.

### Collision Handling & Edge Cases

- **Decoy-to-decoy collision:** Each decoy marks cells visited as it goes. Later decoys naturally avoid earlier ones. On a 25x25 grid (625 cells), worst case is ~80 solution cells + 3 decoys at ~60 cells each = ~260 cells (42% of grid), leaving 365 cells for Phase 2. Typical case is ~60 solution cells + ~30-45 per decoy = ~150-195 cells, leaving ample room. In practice, decoys will often be shorter than their target length due to collisions and backtracking.
- **Short decoy fallback:** If a decoy gets stuck early (under 10 cells), it remains as a shorter branch. No retry logic — Phase 2 may extend it organically.
- **No visual distinction:** Decoy corridors are indistinguishable from any other corridor after Phase 2 fills the grid.
- **Checkpoint system unaffected:** Checkpoints only track progress along `solutionPath`. Decoy branches are regular corridors from gameplay's perspective.

## Non-Goals

- Decoy branches do not need to connect to each other.
- No new UI, visual indicators, or gameplay mechanics.
- No changes to the existing solution path algorithm or fill algorithm.
- No difficulty settings or configurability for decoy count/length (hardcoded to 3 branches, 50-75% solution length).
