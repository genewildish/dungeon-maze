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
- Add jitter: pick a random cell within ±3 indices of each division point.
- Each branch point must have at least one unvisited neighbor. If not, slide along the solution path until one is found.

This prevents branches from clustering together and ensures they span the full length of the solution.

### Target Edge Assignment

Each decoy targets a different edge of the grid. For variety and maximum length:

- For each branch point, calculate the farthest edge based on the branch point's grid position (e.g., a branch point near the top targets the south edge).
- If two branch points would target the same edge, reassign the closer one to its second-farthest edge.
- The target cell is placed 1-2 cells from the target edge at a random position along that edge (e.g., targeting the north edge means `y = 1`, random `x`).

This ensures the 3 branches fan out in different directions and each has room to grow long.

### Decoy Walk Algorithm

Each decoy uses the same biased random walk pattern as the solution path:

1. **Start** from the branch point on the solution path. First step goes to an unvisited neighbor.
2. **Directional bias** toward the target edge: 40% weight for the target direction, 20% each for the other three. Uses the same weighted-pool approach as `generateSolutionPath()`.
3. **Self-avoiding** — only walks through unvisited cells. Backtracks if cornered (same backtracking logic as the solution path).
4. **Stop conditions** (whichever comes first):
   - Walk reaches within 1-2 cells of the target edge (success).
   - Walk reaches 50-75% of the solution path length.
   - Walk exhausts all backtracking options (keep whatever was carved).
5. **Mark** all carved cells as visited so Phase 2 fills around them.

### Collision Handling & Edge Cases

- **Decoy-to-decoy collision:** Each decoy marks cells visited as it goes. Later decoys naturally avoid earlier ones. On a 25x25 grid (625 cells) with ~60 solution cells and ~30-45 cells per decoy, there is sufficient room.
- **Short decoy fallback:** If a decoy gets stuck early (under 10 cells), it remains as a shorter branch. No retry logic — Phase 2 may extend it organically.
- **No visual distinction:** Decoy corridors are indistinguishable from any other corridor after Phase 2 fills the grid.
- **Checkpoint system unaffected:** Checkpoints only track progress along `solutionPath`. Decoy branches are regular corridors from gameplay's perspective.

## Non-Goals

- Decoy branches do not need to connect to each other.
- No new UI, visual indicators, or gameplay mechanics.
- No changes to the existing solution path algorithm or fill algorithm.
- No difficulty settings or configurability for decoy count/length (hardcoded to 3 branches, 50-75% solution length).
