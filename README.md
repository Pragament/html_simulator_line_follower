# Line Follower Lab — README

An interactive, browser-based simulator that teaches block coding through a line-following robot. Mathematics and physics are woven directly into maze checkpoints so students learn that **academic knowledge becomes a practical problem-solving tool**:

> **Maths/Physics Knowledge → Decision → Robot Behaviour → Measurable Result**

The app is a single self-contained HTML file. No build step, no dependencies, no server required — just open it in a modern browser.

---

## Table of Contents

- [Part 1 — User Guide](#part-1--user-guide)
  - [Quick start](#quick-start)
  - [The screen](#the-screen)
  - [Block reference](#block-reference)
  - [Maze types & checkpoint meanings](#maze-types--checkpoint-meanings)
  - [Example lessons](#example-lessons)
- [Part 2 — Teacher Notes](#part-2--teacher-notes)
  - [Learning objectives](#learning-objectives)
  - [Curriculum coverage](#curriculum-coverage)
  - [Assessment ideas](#assessment-ideas)
- [Part 3 — Developer Guide](#part-3--developer-guide)
  - [Architecture at a glance](#architecture-at-a-glance)
  - [File layout](#file-layout)
  - [State model](#state-model)
  - [Maze generation](#maze-generation)
  - [Checkpoint & clue generation](#checkpoint--clue-generation)
  - [Block interpreter](#block-interpreter)
  - [Robot physics & route following](#robot-physics--route-following)
  - [Rendering pipeline](#rendering-pipeline)
  - [Main loop](#main-loop)
  - [Extending the app](#extending-the-app)
  - [Performance & accessibility notes](#performance--accessibility-notes)
- [Part 4 — Known Limitations](#part-4--known-limitations)
- [Glossary](#glossary)

---

## Part 1 — User Guide

### Quick start

1. Open `line-follower-lab.html` in Chrome, Edge, Firefox, or Safari.
2. Pick a **maze type** in the top-left dropdown.
3. Click **🎲 New Maze** to generate a fresh layout.
4. Drag blocks from the **left palette** into the **Script** area, or click a palette block to append it.
5. Press **▶ Run**. Watch the robot follow the line, stop at checkpoints, and (hopefully) reach the goal.
6. Press **↺ Reset** to try again. Press **⏹ Stop** to abort mid-run.

The starter script is already a minimal working program:

```
🚩 when flag clicked
  forever
    do line following until next checkpoint
```

This will drive to the first checkpoint, but **not** turn at it. Add `turn`, `if`, `set speed`, and `wait` blocks to make it finish the maze.

---

### The screen

| Area | What it does |
|---|---|
| **Header** | Maze type selector, New Maze, Run / Stop / Reset |
| **Left — Blocks** | Palette of draggable blocks, grouped by Control and Motion |
| **Left — Script** | Your program. Blocks stack vertically; containers (`forever`, `if`, `repeat`) hold a nested body |
| **Center — Canvas** | The maze. Yellow = the correct line, gold = a decoy branch, S = start, G = goal, numbered circles = checkpoints |
| **Center — HUD** | Live time, distance, checkpoints reached, current speed, speed limit, and cumulative speed-limit violation time |
| **Bottom — Maze card** | Name, description, and a worked example for the selected maze type |
| **Bottom — Clues card** | One clue per checkpoint: the question, its answer, and the rule that maps the answer to a turn direction. Turns green once visited |

---

### Block reference

Blocks are colour-coded: **orange = Control**, **blue = Motion**.

#### Control

| Block | Fields | Behaviour |
|---|---|---|
| `forever` | — | Repeats its body indefinitely. The main loop of any working robot. |
| `repeat [n] times` | `n` | Repeats its body `n` times, then continues. |
| `if [cond] then` | condition | Runs its body once if the condition is true. |
| `wait [n] seconds` | `n` | Pauses the whole program for `n` seconds. |

**Available conditions:** `has reached a checkpoint`, `at checkpoint 1–4`, `at the goal`, `speed is over the limit`, `speed is under 30 %`, `time > 30 seconds`.

#### Motion

| Block | Fields | Behaviour |
|---|---|---|
| `do line following until [mode]` | `next checkpoint` \| `the goal` | Drives along the line. At each junction it picks the branch whose direction best matches the robot's current heading. Pauses the script until arrival. |
| `turn [dir] [n] degrees` | `left`/`right`, `n` | Rotates the robot in place at `TURN_RATE` rad/s. **This is how you commit to a branch.** |
| `set speed to [n] %` | `n` | Sets the target speed as a percentage. `100 %` = 240 px/s. Values above the speed limit accumulate violation time. |
| `go forward [n] steps` | `n` | Creeps `n` cells straight ahead along the current route, ignoring the line. |
| `stop` | — | Ends the program. |

> **Key insight for students:** `do line following until next checkpoint` gets you *to* a checkpoint but does **not** turn. The correct turn direction comes from the checkpoint clue. So the canonical loop is:

```
forever
  do line following until next checkpoint
  if at checkpoint 1 then
    turn left 90 degrees        ← decided by the maths clue
  if at checkpoint 2 then
    turn right 90 degrees
  ...
  do line following until the goal
```

---

### Maze types & checkpoint meanings

Each maze type changes **how many checkpoints** exist and **what they mean**. The clue at each checkpoint still gives a direction; the maze type changes the narrative and the reasoning chain.

| Type | Checkpoints | Semantic meaning | Reasoning chain |
|---|---|---|---|
| **1 · Waypoint Run** | 1 | Mandatory waypoint — a place the robot *must* pass through. The clue decides a single left/right bend. | One problem → one decision. |
| **2 · Arithmetic Relay** | 2 | A relay: each checkpoint hands you a number, the second modifies the first. The combined value drives **both** turns. | Two linked problems → two decisions. |
| **3 · Pattern Trail** | 3 | Checkpoints reveal terms of a number sequence. Recognising the rule tells you which branch continues the trail. | Identify pattern → predict next term → three decisions. |
| **4 · Geometry Grid** | 4 | Checkpoints are points on a coordinate grid. Angles, distances, and coordinates drive four linked decisions. | Measure & compute → four decisions. |

#### Worked examples per type

**Waypoint Run (1 checkpoint)**
> A courier van must stop at the fuel depot before reaching the warehouse. The depot sign reads: *"3/4 of 80 — even → right, odd → left."*
> 3/4 of 80 = 60, which is even → **turn right**.

**Arithmetic Relay (2 checkpoints)**
> Depot A gives *"25 % of 240"* = 60. Depot B gives *"half of that"* = 30.
> Rules: multiple of 10 → **go straight** at A; answer < 30 → **turn left** at B.

**Pattern Trail (3 checkpoints)**
> The trail reads 2, 5, 10, 17, … The next term is 26. That value is used at the third gate.
> At earlier gates, the same sequence's intermediate terms (2, 5, 10) decide the branches.

**Geometry Grid (4 checkpoints)**
> Checkpoint A sits at (2, 3) and B at (2, 11). Distance = 8 → even → **turn right** at B.
> Checkpoint C at (7, 11): the angle from B to C is 90°, so the third clue asks for the third angle of a triangle with 65° and 45° (answer 70 → left, say).

> Checkpoint counts are fixed per type, but **the actual clue questions are drawn randomly** from a bank of 25+ problems across fractions, percentages, ratios, measurement, angles, geometry, coordinates, speed, time, force, friction, and patterns. Each run therefore tests a different subset.

---

### Example lessons

**Lesson 1 — "Follow the line" (ages 8–10, 20 min)**
Use only `forever` + `do line following until the goal`. No checkpoints reached. Discuss *why* the robot ignored the checkpoints. Introduce the idea of a "must-visit" place.

**Lesson 2 — "One clue, one turn" (ages 10–12, 30 min)**
Waypoint Run. Students read the single clue, compute the answer, and add an `if at checkpoint 1 then turn …` block. Compare runs: did everyone turn the same way?

**Lesson 3 — "Fractions on the road" (ages 11–13, 45 min)**
Arithmetic Relay. Students compute both clues, then write a script with two `if` blocks. Introduce `set speed` and challenge them to finish under 20 s without exceeding the speed limit.

**Lesson 4 — "Patterns and sequences" (ages 12–14, 45 min)**
Pattern Trail. Students must first identify the sequence rule *before* writing any code, then encode the decisions.

**Lesson 5 — "Coordinates and angles" (ages 13–15, 60 min)**
Geometry Grid. Students measure distances and angles on the canvas, then write a fully-specified script with four `if` blocks.

---

## Part 2 — Teacher Notes

### Learning objectives

By the end of a session, students should be able to:

1. **Decompose** a problem into a sequence of decisions.
2. **Translate** a mathematical result into a discrete control action (left / right / straight).
3. **Read and write** a control-flow program using forever loops and conditionals.
4. **Measure** the effect of their code (time, distance, violations) and iterate.
5. **Explain** why a physics concept (speed, friction, force) constrains a real robot.

### Curriculum coverage

The problem bank deliberately spans:

- **Number:** fractions, percentages, ratios, arithmetic.
- **Algebra & patterns:** sequences, squares, next-term prediction.
- **Geometry:** angles, triangle angle sums, perimeters, coordinates.
- **Measurement:** unit conversion (m ↔ cm).
- **Physics:** speed = distance / time, F = ma, friction = μN.

### Assessment ideas

- **Correctness** — Did the robot reach the goal with all checkpoints visited?
- **Efficiency** — Fastest time with zero speed-limit violations.
- **Explanation** — Ask students to annotate each `if` block with the clue it responds to.
- **Debugging** — Give a broken script and ask what clue it misread.
- **Extension** — "Add a `repeat` loop that slows the robot by 10 % every checkpoint."

---

## Part 3 — Developer Guide

### Architecture at a glance

The app is a single HTML file with no external dependencies. It is organised as:

```
┌──────────────────────────────────────────────────────────┐
│  Static data                                             │
│  MAZE_TYPES · PROBLEMS · RULES · BLOCK_DEFS · PALETTE    │
├──────────────────────────────────────────────────────────┤
│  Maze generation                                         │
│  genMaze() · solve() · buildMaze() · makeClue()          │
├──────────────────────────────────────────────────────────┤
│  Block model & UI                                        │
│  script[] · makeBlock() · renderBlock() · bodyDiv()      │
├──────────────────────────────────────────────────────────┤
│  Interpreter                                             │
│  runBlock() · tick() · evalCond()                        │
├──────────────────────────────────────────────────────────┤
│  Robot physics & routing                                 │
│  attachRoute() · pointAt() · updateRobot()               │
├──────────────────────────────────────────────────────────┤
│  Rendering & HUD                                         │
│  draw() · updateHud() · renderClues() · renderMazeInfo() │
├──────────────────────────────────────────────────────────┤
│  Main loop (requestAnimationFrame, fixed max dt)         │
└──────────────────────────────────────────────────────────┘
```

Everything lives inside one `<script>` block. Global mutable state is concentrated in the `state` and `program` objects.

### File layout

Single file. Key sections are marked with comment banners:

```
/* ---------------- constants ---------------- */
/* ---------------- maze types ---------------- */
/* ---------------- maths problem bank ---------------- */
/* ---------------- state ---------------- */
/* ---------------- maze build ---------------- */
/* ---------------- block definitions ---------------- */
/* ---------------- palette / script UI ---------------- */
/* ---------------- program interpreter ---------------- */
/* ---------------- robot physics / update ---------------- */
/* ---------------- run control ---------------- */
/* ---------------- rendering ---------------- */
/* ---------------- HUD / panels ---------------- */
/* ---------------- main loop ---------------- */
/* ---------------- init ---------------- */
```

### State model

```js
const state = {
  cells, cols, rows,           // maze grid (see below)
  path,                        // array of cells from start to goal
  checkpoints,                 // [{index, cell, required, clue, visited}]
  decoy,                       // Map<pathIndex, cell>  — dead-end stubs
  speedLimit,                  // percent
  typeKey,                     // 'waypoint' | 'relay' | 'pattern' | 'geometry'
  mode,                        // 'idle' | 'following' | 'turning' | 'creeping'
  elapsed, distance,           // measured in seconds / 0.5 cm units
  visited, nextCP, atCP,       // checkpoint tracking
  overSpeed, message,          // violation timer & last error text
  success, finished,
  robot: {
    x, y, heading,             // pose in pixels/radians
    speed, targetSpeed,        // px/s
    targetSpeedPct,            // percent (0–120)
    s,                         // arc length along current route (px)
    route,                     // {points, startIndex, isMain}
    turnDir, turnRemaining     // in-place turn state
  }
};

const program = {
  running, stack,              // stack of {body, index, loop?, repeat?, count?}
  blocked,                     // {done, kind, until} — pause gate
  waitTimer                    // seconds remaining for `wait`
};
```

A **cell** is `{x, y, walls:[N,E,S,W], visited}`. A wall is `true` if solid.

### Maze generation

```js
genMaze(cols, rows) → cells[]
```
A standard **recursive-backtracker** maze. Starts at `(0,0)`, carves passages by repeatedly visiting a random unvisited neighbour and removing the wall between them. Produces a perfect maze: exactly one path between any two cells.

```js
solve(cells, cols, rows, start, goal) → cell[]
```
Breadth-first search from start to goal, tracking predecessors, then reconstructing the path. Because the maze is perfect, this is *the* unique solution.

`buildMaze(typeKey)` orchestrates:

1. Regenerate until the solution path has ≥ 22 cells (avoids trivial layouts).
2. Find **turn indices** — cells where the incoming and outgoing directions differ.
3. Pick `cfg.cps` checkpoint indices, spread evenly across the turn list.
4. For each checkpoint, add a **decoy branch**: open the wall in the *incoming* direction so there is a fake "straight on" choice. Record it in `state.decoy`.
5. Compute the required turn direction with the 2D cross product of the in and out direction vectors.
6. Generate a clue whose rule evaluates to that required direction.

### Checkpoint & clue generation

```js
makeClue(requiredDirection) → {q, a, topic, rule, dir}
```

Shuffles the `PROBLEMS` bank and, for each problem, tries every `RULE`. The first `(problem, rule)` pair whose rule function returns the required direction is chosen. This guarantees the clue is always solvable and always agrees with the maze geometry.

**To add a problem:**

```js
{ q:'5/6 of 72', a:60, topic:'Fractions' }
```

**To add a rule:**

```js
{ text:'answer is prime → turn LEFT · otherwise RIGHT',
  fn:a => isPrime(a) ? 'left' : 'right' }
```

Rules must return `'left'`, `'right'`, or `'straight'`. If you introduce `'straight'`, no change is needed elsewhere — the checkpoint code already handles it (the robot simply does not turn, and line-following continues along the main branch).

### Block interpreter

The interpreter is a **stack of frames**, each frame holding a body array and an index:

```js
{ body: [...blocks], index: 0, loop: bool, repeat: n, count: k }
```

`tick(dt)` advances the top frame. When a block returns `'pause'`, the interpreter stops and waits for the block to signal completion via `program.blocked.done = true`. This is how long-running blocks (`wait`, `turn`, `lineFollow`, `forward`) hand control back to the physics loop.

- `forever` pushes a frame with `loop: true` — the frame resets `index = 0` instead of popping.
- `repeat` pushes a frame with `repeat: n` — the frame re-runs until `count === n`.
- `if` pushes a frame only when `evalCond` is true.
- `wait` sets `program.waitTimer`; `tick` decrements it before running the next block.
- `turn`, `lineFollow`, and `forward` set `program.blocked = {done:false}` and change `state.mode`. The physics update sets `done = true` on completion.

A `guard` counter (max 800 iterations per tick) prevents infinite tight loops from freezing the tab.

### Robot physics & route following

The robot has a pose `(x, y, heading)` and a scalar `speed`. Two motion modes:

**Turning in place.** `turnRemaining` decrements by `TURN_RATE * dt`. When it hits zero, `program.blocked.done = true`.

**Following a route.** A route is `{points: [{x,y}, …], startIndex, isMain}`. The robot tracks arc length `s` along the route:

- Speed ramps toward `targetSpeed` with a fixed acceleration (400 px/s² while following, 900 px/s² while creeping).
- The desired heading is the direction to a point ~26 px ahead on the polyline.
- The actual heading is exponentially smoothed toward the desired heading — the smoothing factor scales with speed, so **fast robots corner sloppily**, which is the physical intuition behind the speed limit.
- Speed-limit violations accumulate in `state.overSpeed` while `targetSpeedPct > state.speedLimit`.

**Branch selection at checkpoints.** When line-following ends at a checkpoint, `attachRoute()` compares the robot's current heading to each available branch (main continuation, decoy stub) and picks the closest one within 60°. If the robot's heading is wrong — i.e. the student turned the wrong way — it either drives down the decoy and fails at the dead end, or finds no branch and fails immediately.

### Rendering pipeline

`draw()` is called every frame and is fully stateless — it reads `state` and paints:

1. Background + cell floors.
2. Walls as a single `Path2D` stroke.
3. Decoy stubs in gold.
4. The main path in yellow.
5. Start and goal markers.
6. Checkpoint rings (colour changes when visited; thicker when current).
7. The robot as a rotated rounded rectangle with a triangular nose and two headlights.

Coordinates are in canvas pixels (13 × 52 = 676 wide, 9 × 52 = 468 high). The canvas has `max-width:100%` for responsive layout but no DPR scaling — see limitations.

### Main loop

```js
let lastT = performance.now();
function loop(t) {
  const dt = Math.min(0.05, (t - lastT) / 1000);  // clamp to 50 ms
  lastT = t;
  if (program.running) { state.elapsed += dt; tick(dt); }
  if (program.running || state.mode !== 'idle' || state.robot.speed > 0)
    updateRobot(dt);
  hudAcc += dt;
  if (hudAcc > 0.08) { hudAcc = 0; updateHud(); }
  draw();
  requestAnimationFrame(loop);
}
```

The `dt` clamp prevents a tab that was backgrounded from teleporting the robot through walls. HUD updates are throttled to ~12 Hz to avoid layout thrash.

### Extending the app

| Goal | Where to change |
|---|---|
| Add a new problem | Append to `PROBLEMS` |
| Add a new rule | Append to `RULES` |
| Add a new maze type | Add an entry to `MAZE_TYPES` with `name`, `cps`, `desc`, `ex`. No other code changes needed. |
| Add a new block | Add to `BLOCK_DEFS` with `cat`, `text`, optional `fields`, optional `body:true`. Add a `case` in `runBlock`. Add it to `PALETTE`. |
| Change physics feel | `PX_PER_PCT`, `TURN_RATE`, or the acceleration constants inside `updateRobot` |
| Change speed limit | `state.speedLimit` (currently 70) |
| Change maze size | `CELL`, `COLS`, `ROWS` — but also update the `<canvas>` width/height attributes (they are set from `W`/`H` in JS, so this is automatic) |
| Localise strings | All user-visible text is inline; there is no i18n layer |

**Example: adding a "sound" block**

```js
// 1. definition
BLOCK_DEFS.beep = { cat:'motion', text:'beep' };

// 2. handler
case 'beep':
  playTone();
  return 'ok';

// 3. palette entry
{ group:'Motion', items:['lineFollow','turn','setSpeed','forward','beep','stop'] }
```

### Performance & accessibility notes

- No memory allocation inside `draw()` except for the robot's `roundRect` path — safe for 60 fps on low-end hardware.
- No `innerHTML` is written every frame; the DOM is only re-rendered when the script or checkpoints change.
- Blocks are keyboard-focusable inputs and selects. The drag-and-drop palette also supports **click to append**, which is the primary path for keyboard and touch users.
- Colour contrast: all block text is white on saturated backgrounds; the clue panel uses a distinct yellow accent.
- `prefers-reduced-motion` is not currently honoured — animations are integral to the simulation.

---

## Part 4 — Known Limitations

- **No save/load.** Refreshing the page loses the script.
- **No undo/redo** in the script editor.
- **No DPR scaling** on the canvas — on very high-DPI displays the maze may look slightly soft. Fix by multiplying `cv.width/height` by `devicePixelRatio` and scaling the context.
- **Decoy dead-ends are shallow** — only one cell deep. They are visually convincing but not long.
- **Only one decoy per checkpoint.** Multiple fake branches are not generated.
- **`go forward` uses the current route**, so it only works immediately after a checkpoint or on the starting straight.
- **The interpreter is single-threaded and cooperative.** A script with a `forever` loop and no `lineFollow`/`turn`/`wait` inside will spin until the 800-iteration guard, then surrender the frame — it will not crash but will not progress either.

---

## Glossary

| Term | Meaning |
|---|---|
| **Block** | A single instruction in the script, e.g. `turn left 90 degrees` |
| **Body** | The nested list of blocks inside a container block |
| **Checkpoint** | A marked cell the robot must visit; carries a maths/physics clue |
| **Decoy** | A fake branch from a checkpoint that leads to a dead end |
| **Route** | The polyline the robot is currently following, plus its start index |
| **Arc length (`s`)** | Distance travelled along the current route, in pixels |
| **Speed limit** | Percentage above which the robot accumulates violation time |
| **Clue** | The `{question, answer, rule, direction}` tuple shown at a checkpoint |
| **Required direction** | The turn the maze geometry demands at a checkpoint |
| **Violation** | Cumulative seconds spent above the speed limit |

---

*Line Follower Lab is a single-file, offline-capable teaching tool. Fork it, remix it, and add your own problems and rules — the problem bank and maze types are deliberately data-driven so teachers can extend them without touching the engine.*
