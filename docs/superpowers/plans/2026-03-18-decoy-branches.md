# Decoy Branches Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 3 long decoy branches that fork off the solution path toward different grid edges, making the maze feel non-deterministic.

**Architecture:** A new `generateDecoyBranches()` function runs between Phase 1 (solution path) and Phase 2 (fill). It picks 3 spread-out branch points on the solution path, assigns each a target edge (north, south, or west), and carves biased random walks toward those edges. Phase 2 fills around them naturally.

**Tech Stack:** Vanilla JavaScript, HTML5 Canvas (single `index.html` file)

**Note:** This project has no test framework. Verification is done by opening `index.html` in a browser and confirming behavior visually + via console.

---

## Chunk 1: Implementation

### Task 1: Write `generateDecoyBranches()` function

**Files:**
- Modify: `index.html:226` (insert new function after `generateSolutionPath()`, before `fillMaze()`)

- [ ] **Step 1: Add the function skeleton and branch point selection**

Insert after the closing `}` of `generateSolutionPath()` (line 225) and before `function fillMaze()` (line 227):

```javascript
    // === DECOY BRANCH GENERATION ===
    // Phase 1.5: Carve 3 long branches off the solution path toward different
    // grid edges. These create convincing alternative routes that make the maze
    // feel non-deterministic. Each branch is 50-75% the length of the solution
    // path and stops 1-2 cells short of its target edge.

    function generateDecoyBranches() {
      const NUM_DECOYS = 3;
      const pathLen = solutionPath.length;

      // --- Branch Point Selection ---
      // Pick 3 points at ~25%, ~50%, ~75% of solution path with ±3 jitter.
      // Each must have at least one unvisited neighbor to start carving from.
      const branchPoints = [];
      for (let i = 1; i <= NUM_DECOYS; i++) {
        const baseIndex = Math.floor(pathLen * i / 4);
        const jitter = Math.floor(Math.random() * 7) - 3;
        const startIndex = Math.max(2, Math.min(pathLen - 3, baseIndex + jitter));

        let found = false;

        // Slide forward along solution path to find cell with unvisited neighbor
        for (let j = startIndex; j < pathLen - 2; j++) {
          const [bx, by] = solutionPath[j];
          if (getNeighbors(bx, by, true).length > 0) {
            branchPoints.push({ x: bx, y: by, index: j });
            found = true;
            break;
          }
        }

        // If forward slide failed, slide backward from original point
        if (!found) {
          for (let j = startIndex - 1; j >= 2; j--) {
            const [bx, by] = solutionPath[j];
            if (getNeighbors(bx, by, true).length > 0) {
              branchPoints.push({ x: bx, y: by, index: j });
              found = true;
              break;
            }
          }
        }
        // If no valid branch point in this segment (extremely unlikely), skip it
      }

      // --- Target Edge Assignment ---
      // Exclude east edge (solution already exits there). Assign each branch its
      // farthest available edge by Manhattan distance; break ties randomly.
      const availableEdges = new Set(['n', 's', 'w']);
      const assignments = [];

      for (const bp of branchPoints) {
        const candidates = [...availableEdges].map(edge => {
          let dist;
          if (edge === 'n') dist = bp.y;
          else if (edge === 's') dist = ROWS - 1 - bp.y;
          else dist = bp.x; // west
          return { edge, dist };
        });

        // Sort farthest first, pick randomly among ties
        candidates.sort((a, b) => b.dist - a.dist);
        const maxDist = candidates[0].dist;
        const ties = candidates.filter(c => c.dist === maxDist);
        const chosen = ties[Math.floor(Math.random() * ties.length)];

        availableEdges.delete(chosen.edge);
        assignments.push({ ...bp, targetEdge: chosen.edge });
      }

      // --- Carve Decoy Walks ---
      // Each walk uses a biased random walk (40% toward target edge, 20% others)
      // with self-avoiding backtracking, same pattern as generateSolutionPath().
      for (const { x: startX, y: startY, targetEdge } of assignments) {
        // Random target length: 50-75% of solution path
        const minLen = Math.floor(pathLen * 0.5);
        const maxLen = Math.floor(pathLen * 0.75);
        const targetLength = minLen + Math.floor(Math.random() * (maxLen - minLen + 1));

        // Biased random walk from branch point toward target edge
        const decoyPath = [[startX, startY]];
        let cx = startX, cy = startY;

        while (decoyPath.length < targetLength) {
          const neighbors = getNeighbors(cx, cy, true);

          if (neighbors.length === 0) {
            // Backtrack — pop current cell and return to previous
            decoyPath.pop();
            if (decoyPath.length === 0) break; // exhausted back past branch point
            const prev = decoyPath[decoyPath.length - 1];
            cx = prev[0];
            cy = prev[1];
            continue;
          }

          // Weighted direction: 40% toward target edge, 20% each for others
          const weighted = [];
          for (const [nx, ny, dir] of neighbors) {
            const weight = dir === targetEdge ? 40 : 20;
            for (let w = 0; w < weight; w++) weighted.push([nx, ny]);
          }
          const [nx, ny] = weighted[Math.floor(Math.random() * weighted.length)];

          removeWall(cx, cy, nx, ny);
          grid[ny][nx].visited = true;
          decoyPath.push([nx, ny]);
          cx = nx;
          cy = ny;

          // Stop if within 1-2 cells of target edge (never on the edge itself)
          if (targetEdge === 'n' && cy <= 1) break;
          if (targetEdge === 's' && cy >= ROWS - 2) break;
          if (targetEdge === 'w' && cx <= 1) break;
        }
      }
    }
```

- [ ] **Step 2: Verify no syntax errors**

Open browser dev console, reload `index.html`. Confirm no errors. The function exists but isn't called yet.

### Task 2: Wire `generateDecoyBranches()` into maze generation

**Files:**
- Modify: `index.html:258-262` (the `generateMaze()` function)

- [ ] **Step 3: Add the call between Phase 1 and Phase 2**

Change `generateMaze()` from:

```javascript
    function generateMaze() {
      createGrid();
      generateSolutionPath();
      fillMaze();
    }
```

To:

```javascript
    function generateMaze() {
      createGrid();
      generateSolutionPath();
      generateDecoyBranches();
      fillMaze();
    }
```

- [ ] **Step 4: Browser verification**

Open `index.html` in browser and verify:
1. Game loads without console errors.
2. Start screen appears; click START.
3. Maze generates and is playable — walls block movement, no broken corridors.
4. Maze has noticeably longer corridors branching off the main route (compare with previous behavior by commenting out `generateDecoyBranches()` and regenerating).
5. Play through to win — checkpoint and dead-end systems still work.
6. Click "Play Again" — new maze generates cleanly.
7. Repeat 3-4 times to confirm variety in branch directions.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add decoy branches: 3 long paths branching from solution toward grid edges"
```
