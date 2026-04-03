
# 🧪 Color Sort Puzzle — Optimal Solver

<h4>
    <a href="https://skylum1.github.io/color_puzzle_solver/">View Demo</a> 

</h4>

A browser-based solver for puzzles posted on [r/ColorPuzzleGame](https://www.reddit.com/r/ColorPuzzleGame/). Drop in any puzzle layout from the subreddit, get the **guaranteed minimum move solution**, and step through it interactively.



---
## ✏️ Entering a Puzzle from Reddit

Use the **Level Editor** to input any puzzle you see posted on the subreddit.

### Step 1 — Open the Editor

Click the **Level Editor** button in the controls bar. The game board switches to edit mode and a palette + settings panel appears below.

### Step 2 — Paint the Board

- **Click a color swatch** from the palette to select it (a ✓ badge confirms the active color).
- **Click any cell** in a tube to paint it with the selected color. Cells map left-to-right to top-to-bottom visually.
- Select the **✕ Eraser** swatch to clear any cell you mis-painted.
- Click **🗑 Clear** to wipe the entire board and start fresh.

### Step 3 — Use Raw Text Input (fastest method)

Expand **⚙️ Layout Settings** to access the text area. This is the quickest way to enter a puzzle — type or paste the layout directly.

**Format:** one tube per line, colors as single-letter codes separated by spaces, `_` for empty slots.

```
R G B C
Y P O L
W M G R
_ _ _ _
_ _ _ _
```

The board updates live as you type. Click **⟳ Apply Text** to force a re-parse if needed.

### Color Letter Reference

| Label | Color   | Label | Color    |
| ----- | ------- | ----- | -------- |
| `Y`   | Yellow  | `O`   | Orange   |
| `P`   | Purple  | `R`   | Red      |
| `C`   | Cyan    | `W`   | Grey     |
| `M`   | Magenta | `L`   | Lavender |
| `B`   | Blue    | `G`   | Green    |

### Step 4 — Validate

The editor shows live validation:
- ✅ **Green** — every color appears exactly 4 times. Puzzle is valid and ready.
- 🟡 **Orange** — color counts are uneven. The message tells you exactly how many of each color are placed so you can spot typos quickly.

### Step 5 — Launch the Solver

Click **Play Custom Level** to exit the editor. The board is now locked as your starting state. Then click **Show Optimal Play** to run the solver.

---
## 🧠 Optimal Strategy — How the Solver Works

The solver guarantees the **minimum possible number of moves** for any valid puzzle. Here is the full approach.

### Algorithm: Breadth-First Search (BFS)

BFS explores the puzzle state graph level by level — all states reachable in 1 move, then all states reachable in 2 moves, and so on. Because every pour operation has equal cost (1 move), the **first time BFS reaches the solved state it is provably via the shortest path**. No heuristic or approximation is involved — this is an exhaustive search guarantee.

### Canonical State Hashing

A key challenge in BFS is avoiding revisiting equivalent board positions. Two board positions are **logically identical** if they contain the same multiset of tube contents, regardless of which physical slot holds which tube (since you can pour between any two tubes freely).

The solver exploits this with a canonical hash:

```js
function getCode(state) {
    let s = state.map(tube => tube.join(''));
    s.sort();               // order-independent
    return s.join('|');
}
```

Sorting tube strings before joining produces a single key that treats all physically permuted but logically equivalent boards as the same state. This dramatically shrinks the state space and is mathematically valid for this puzzle type.

### Memory-Efficient Parent-Pointer Map

Naïve BFS copies the entire move path into every node in the queue — for 200,000 nodes this creates hundreds of thousands of growing arrays and dominates memory usage.

The solver uses a **parent-pointer map** instead:

```js
// Map: stateCode → { parentCode, move }
visited.set(code, { parentCode: currCode, move: { from: i, to: j } });
```

Only one `{parentCode, move}` object is stored per state. Once the solved state is found, the optimal path is reconstructed in a single backward walk through the map. This reduces peak memory by ~80% compared to path-copying BFS.

### Web Worker (Non-Blocking)

The BFS runs inside a **Web Worker** (an inline Blob worker — no separate `.js` file needed). The UI stays fully responsive while the solver computes. A **12-second wall-clock timeout** on the main thread terminates and recreates the worker if no answer arrives, preventing the browser from hanging on pathological inputs.

### Stale-Result Guard

A `solveGeneration` counter increments every time a new puzzle is entered or the board is reset. The generation is captured when the worker is dispatched and compared when it responds — stale results from a previous puzzle are silently discarded, so switching puzzles mid-calculation is always safe.

### Complexity & Practical Limits

| Metric                     | Typical value   |
| -------------------------- | --------------- |
| States explored per puzzle | 20,000 – 40,000 |
| Optimal move count range   | 25 – 35 moves   |
| Hard cap (`MAX_STATES`)    | 200,000         |
| Solver timeout             | 12 seconds      |

The 200,000 state cap means the solver returns `null` (no solution displayed) rather than a wrong answer if the cap is hit. For the standard 12-tube / 10-color configuration this cap is never reached in practice.

---

## 🎛️ Solver Controls

| Control                | Description                                                                       |
| ---------------------- | --------------------------------------------------------------------------------- |
| **Show Optimal Play**  | Runs BFS and enters step-by-step Playback Mode                                    |
| **Hint**               | Highlights only the *next* optimal source tube (amber pulse) without auto-playing |
| **Playback slider**    | Scrub to any move in the optimal solution                                         |
| **◀ / ▶ step buttons** | Navigate one move at a time                                                       |
| **▶ Play / ⏸ Pause**   | Auto-play the full solution                                                       |
| **Exit**               | Leave playback and return to free play                                            |
| **Undo**               | Step back one move in free play                                                   |
| **Restart Level**      | Reset board to the original entered state                                         |
| **New Game**           | Generate a random puzzle (ignores Reddit input)                                   |

---
> ✨ **Vibe coded** — this entire solver was built through an AI-assisted, conversational development session. The gameplay mechanics, BFS solver, level editor, playback system, and all UI/UX were designed and iterated using natural language prompts.