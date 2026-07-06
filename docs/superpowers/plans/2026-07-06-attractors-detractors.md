# Attractors / Detractors Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A single-file p5.js sketch simulating cyclic pursuit on a grid — a ring of pixels each chasing its successor and fleeing its predecessor, with captures, optional corpses, and live controls.

**Architecture:** Everything lives in one `index.html`: p5.js from CDN, an inline `<script>` holding the simulation (plain arrays + a cell-occupancy map), and plain HTML controls wired with `input` events. The sim advances on a time accumulator inside p5's `draw()` so tick rate is independent of frame rate.

**Tech Stack:** p5.js 1.9.x (CDN), vanilla HTML/CSS/JS. No build step, no dependencies to install.

**Spec:** `docs/superpowers/specs/2026-07-06-attractors-detractors-design.md`

## Global Constraints

- Single deliverable file: `index.html` at repo root. No build step, no npm.
- Ring structure: pixel *i* chases *(i+1) mod N*, flees *(i−1) mod N*. No retargeting after a catch.
- Move scoring is lexicographic on squared Euclidean distance (torus-aware in wrap mode). Aggressive = (closer to prey, then farther from hunter); cautious = swapped. Ties broken uniformly at random.
- Capture = moving onto prey's cell (the only exception to the occupancy rule). Corpses ON: cell becomes permanent obstacle, catcher stays put. Corpses OFF: catcher takes the cell.
- Settled = no living pixel has a living prey; sim stops ticking and shows a banner.
- Verification is visual/behavioral in the browser per spec — each task ends with explicit browser checks instead of unit tests.
- Defaults: count 100, grid 80×80, 10 ticks/sec, wrap ON, corpses OFF, shuffled order, aggressive, trails ON, chase lines OFF.

---

### Task 1: Scaffold and static render

**Files:**
- Create: `index.html`

**Interfaces:**
- Produces: `settings` object (fields: `count, gridSize, speed, wrap, corpses, shuffled, aggressive, trails, chaseLines`), pixel array `px` of `{x, y, hue, alive}`, `occ` Map of `"x,y" → pixel index`, `corpseCells` Set of `"x,y"`, counters `tick, captures`, flags `settled, paused`, functions `key(x,y)`, `seedSim()`, `render()`, `aliveCount()`. Task 2 and 3 rely on all of these names exactly.

- [ ] **Step 1: Write `index.html` with layout, styles, settings, seeding, and static rendering**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>attractors / detractors</title>
<script src="https://cdn.jsdelivr.net/npm/p5@1.9.4/lib/p5.min.js"></script>
<style>
  :root { color-scheme: dark; }
  body {
    margin: 0; display: flex; gap: 16px; padding: 16px;
    background: #0b0b0e; color: #cfcfd6;
    font: 13px/1.5 ui-monospace, SFMono-Regular, Menlo, monospace;
  }
  #sketch canvas { display: block; border: 1px solid #26262e; }
  #panel { width: 230px; display: flex; flex-direction: column; gap: 10px; }
  #panel h1 { font-size: 14px; margin: 0 0 4px; color: #ececf2; font-weight: 600; }
  #panel label { display: flex; justify-content: space-between; align-items: center; gap: 8px; }
  #panel label span.val { color: #8f8f9c; }
  #panel input[type=range] { width: 110px; }
  #panel button {
    background: #1c1c24; color: #cfcfd6; border: 1px solid #33333e;
    border-radius: 4px; padding: 5px 10px; cursor: pointer; font: inherit;
  }
  #panel button:hover { background: #26262f; }
  #stats { color: #8f8f9c; min-height: 1.5em; }
  #stats .settled { color: #e6c96b; }
  fieldset { border: 1px solid #26262e; border-radius: 6px; display: flex; flex-direction: column; gap: 6px; }
  legend { color: #8f8f9c; padding: 0 4px; }
  .row { display: flex; gap: 8px; }
</style>
</head>
<body>
<main id="sketch"></main>
<aside id="panel">
  <h1>attractors / detractors</h1>
  <div id="stats"></div>
  <div class="row">
    <button id="btn-pause">pause</button>
    <button id="btn-step">step</button>
    <button id="btn-restart">restart</button>
  </div>
  <fieldset>
    <legend>on restart</legend>
    <label>pixels <span class="val" id="val-count">100</span>
      <input id="in-count" type="range" min="10" max="500" step="10" value="100"></label>
    <label>grid <span class="val" id="val-grid">80</span>
      <input id="in-grid" type="range" min="40" max="120" step="10" value="80"></label>
  </fieldset>
  <fieldset>
    <legend>live</legend>
    <label>speed <span class="val" id="val-speed">10</span>
      <input id="in-speed" type="range" min="1" max="60" step="1" value="10"></label>
    <label>wrap edges <input id="in-wrap" type="checkbox" checked></label>
    <label>corpses block <input id="in-corpses" type="checkbox"></label>
    <label>shuffled order <input id="in-shuffled" type="checkbox" checked></label>
    <label>strategy
      <select id="in-strategy">
        <option value="aggressive" selected>aggressive</option>
        <option value="cautious">cautious</option>
      </select></label>
    <label>trails <input id="in-trails" type="checkbox" checked></label>
    <label>chase lines <input id="in-lines" type="checkbox"></label>
  </fieldset>
</aside>
<script>
const settings = {
  count: 100,
  gridSize: 80,
  speed: 10,          // ticks per second
  wrap: true,
  corpses: false,     // corpses block movement
  shuffled: true,     // shuffled turn order vs fixed ring order
  aggressive: true,   // vs cautious
  trails: true,
  chaseLines: false,
};

let px = [];                  // {x, y, hue, alive}
let occ = new Map();          // "x,y" -> index of living pixel
let corpseCells = new Set();  // "x,y"
let tick = 0, captures = 0;
let settled = false, paused = false;
let acc = 0;                  // ms accumulator for tick timing

const statsEl = document.getElementById('stats');

function key(x, y) { return x + ',' + y; }

function aliveCount() {
  let n = 0;
  for (const p of px) if (p.alive) n++;
  return n;
}

function seedSim() {
  const N = settings.count, G = settings.gridSize;
  px = []; occ = new Map(); corpseCells = new Set();
  tick = 0; captures = 0; settled = false; acc = 0;
  const cells = new Set();
  while (cells.size < N) {
    cells.add(key(Math.floor(random(G)), Math.floor(random(G))));
  }
  let i = 0;
  for (const k of cells) {
    const [x, y] = k.split(',').map(Number);
    px.push({ x, y, hue: (i / N) * 360, alive: true });
    occ.set(k, i);
    i++;
  }
  background(11, 11, 14);
}

function render() {
  if (settings.trails) {
    noStroke();
    fill(11, 11, 14, 40);
    rect(0, 0, width, height);
  } else {
    background(11, 11, 14);
  }
  const cs = width / settings.gridSize;
  noStroke();
  fill(70);
  for (const k of corpseCells) {
    const [x, y] = k.split(',').map(Number);
    rect(x * cs, y * cs, cs, cs);
  }
  colorMode(HSB, 360, 100, 100, 1);
  for (const p of px) {
    if (!p.alive) continue;
    fill(p.hue, 80, 100);
    rect(p.x * cs, p.y * cs, cs, cs);
  }
  if (settings.chaseLines) {
    strokeWeight(1);
    const N = px.length;
    for (let i = 0; i < N; i++) {
      const p = px[i], q = px[(i + 1) % N];
      if (N > 1 && p.alive && q.alive) {
        stroke(p.hue, 60, 100, 0.35);
        line(p.x * cs + cs / 2, p.y * cs + cs / 2, q.x * cs + cs / 2, q.y * cs + cs / 2);
      }
    }
    noStroke();
  }
  colorMode(RGB, 255, 255, 255, 255);
  statsEl.innerHTML = 'tick ' + tick + ' · alive ' + aliveCount() + ' · captures ' + captures
    + (settled ? ' · <span class="settled">settled</span>' : '');
}

function setup() {
  const c = createCanvas(640, 640);
  c.parent('sketch');
  seedSim();
}

function draw() {
  render();
}

document.getElementById('btn-restart').addEventListener('click', () => {
  settings.count = Number(document.getElementById('in-count').value);
  settings.gridSize = Number(document.getElementById('in-grid').value);
  seedSim();
});
</script>
</body>
</html>
```

- [ ] **Step 2: Verify in browser**

Open `index.html` in a browser (e.g. `open index.html`, or drive it with the chrome-devtools MCP tools and take a screenshot).

Expected:
- Dark page, 640×640 canvas on the left, control panel on the right.
- 100 colored squares scattered on the canvas, hues spanning the rainbow.
- Stats line reads `tick 0 · alive 100 · captures 0`.
- Clicking **restart** re-scatters the dots; moving the *pixels* slider to 500 and clicking restart shows visibly more dots.
- No console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Scaffold sketch: layout, controls markup, seeding, static render"
```

---

### Task 2: Simulation engine — movement, capture, corpses, settled

**Files:**
- Modify: `index.html` (the inline `<script>` only)

**Interfaces:**
- Consumes: `settings`, `px`, `occ`, `corpseCells`, `tick`, `captures`, `settled`, `paused`, `acc`, `key()`, `render()` from Task 1.
- Produces: `dist2(x1,y1,x2,y2)`, `stepPixel(i)`, `runTick()`, `isSettled()`, `OFFSETS`. A `draw()` that advances the sim on a time accumulator. Task 3 relies on `paused`, `runTick()`, and `settings` fields being live-readable.

- [ ] **Step 1: Add the engine functions**

Insert after the `aliveCount()` function in the script:

```js
const OFFSETS = [
  [-1, -1], [0, -1], [1, -1],
  [-1,  0], [0,  0], [1,  0],
  [-1,  1], [0,  1], [1,  1],
];

function dist2(x1, y1, x2, y2) {
  const G = settings.gridSize;
  let dx = Math.abs(x1 - x2), dy = Math.abs(y1 - y2);
  if (settings.wrap) {
    dx = Math.min(dx, G - dx);
    dy = Math.min(dy, G - dy);
  }
  return dx * dx + dy * dy;
}

function stepPixel(i) {
  const p = px[i];
  const N = px.length;
  const preyIdx = (i + 1) % N, hunterIdx = (i - 1 + N) % N;
  const prey = px[preyIdx], hunter = px[hunterIdx];
  const chasing = N > 1 && prey.alive;
  const fleeing = N > 1 && hunter.alive;
  if (!chasing && !fleeing) return;

  const G = settings.gridSize;
  let best = [], bestA = Infinity, bestB = Infinity;
  for (const [ox, oy] of OFFSETS) {
    let nx = p.x + ox, ny = p.y + oy;
    if (settings.wrap) {
      nx = (nx + G) % G;
      ny = (ny + G) % G;
    } else if (nx < 0 || ny < 0 || nx >= G || ny >= G) {
      continue;
    }
    const k = key(nx, ny);
    if (corpseCells.has(k)) continue;
    const isPreyCell = chasing && nx === prey.x && ny === prey.y;
    const occupant = occ.get(k);
    if (occupant !== undefined && occupant !== i && !isPreyCell) continue;

    // Inactive criteria contribute a constant 0, which never affects ordering.
    const dPrey = chasing ? dist2(nx, ny, prey.x, prey.y) : 0;
    const dHunt = fleeing ? -dist2(nx, ny, hunter.x, hunter.y) : 0; // negated: farther is better
    const a = settings.aggressive ? dPrey : dHunt;
    const b = settings.aggressive ? dHunt : dPrey;
    if (a < bestA || (a === bestA && b < bestB)) {
      best = [{ x: nx, y: ny }]; bestA = a; bestB = b;
    } else if (a === bestA && b === bestB) {
      best.push({ x: nx, y: ny });
    }
  }
  if (best.length === 0) return;
  const m = best[Math.floor(random(best.length))];
  if (m.x === p.x && m.y === p.y) return;

  if (chasing && m.x === prey.x && m.y === prey.y) {
    prey.alive = false;
    occ.delete(key(prey.x, prey.y));
    captures++;
    if (settings.corpses) {
      corpseCells.add(key(prey.x, prey.y));
      return; // corpse keeps the cell; catcher stays put
    }
  }
  occ.delete(key(p.x, p.y));
  p.x = m.x; p.y = m.y;
  occ.set(key(p.x, p.y), i);
}

function isSettled() {
  const N = px.length;
  if (N < 2) return true;
  for (let i = 0; i < N; i++) {
    if (px[i].alive && px[(i + 1) % N].alive) return false;
  }
  return true;
}

function runTick() {
  const order = [];
  for (let i = 0; i < px.length; i++) if (px[i].alive) order.push(i);
  if (settings.shuffled) {
    for (let j = order.length - 1; j > 0; j--) {
      const r = Math.floor(random(j + 1));
      [order[j], order[r]] = [order[r], order[j]];
    }
  }
  for (const i of order) {
    if (px[i].alive) stepPixel(i); // may have been caught earlier this tick
  }
  tick++;
  settled = isSettled();
}
```

- [ ] **Step 2: Replace `draw()` with the accumulator loop**

Replace the existing `draw()`:

```js
function draw() {
  if (!paused && !settled) {
    acc += deltaTime;
    const interval = 1000 / settings.speed;
    while (acc >= interval) {
      acc -= interval;
      runTick();
      if (settled) break;
    }
  }
  render();
}
```

- [ ] **Step 3: Verify in browser**

Reload `index.html`.

Expected:
- Dots move immediately, forming chains and swirling clusters; the alive count falls as captures accrue.
- With trails ON (default), moving dots leave fading streaks.
- Eventually the stats line shows `settled` in yellow and motion in the counters stops (captures and tick freeze).
- Check the corpse toggle: restart, enable **corpses block**, and confirm gray squares accumulate and living pixels never enter them.
- Uncheck **wrap edges** — note this control is not wired yet (Task 3); instead verify wrap behavior visually: dots fleeing across an edge reappear on the far side rather than piling in corners.
- No console errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add simulation engine: scored movement, capture, corpses, settled detection"
```

---

### Task 3: Wire the controls

**Files:**
- Modify: `index.html` (the inline `<script>` only)

**Interfaces:**
- Consumes: `settings`, `paused`, `runTick()`, `render()`, `seedSim()` from Tasks 1–2. All element ids from Task 1's markup (`in-count`, `in-grid`, `in-speed`, `in-wrap`, `in-corpses`, `in-shuffled`, `in-strategy`, `in-trails`, `in-lines`, `btn-pause`, `btn-step`, `btn-restart`, `val-count`, `val-grid`, `val-speed`).
- Produces: fully wired UI; nothing downstream.

- [ ] **Step 1: Add control wiring**

Replace the existing `btn-restart` listener block at the bottom of the script with:

```js
function bindRange(id, valId, onInput) {
  const el = document.getElementById(id);
  const val = document.getElementById(valId);
  el.addEventListener('input', () => {
    val.textContent = el.value;
    if (onInput) onInput(Number(el.value));
  });
}

bindRange('in-count', 'val-count', null);            // applied on restart
bindRange('in-grid', 'val-grid', null);              // applied on restart
bindRange('in-speed', 'val-speed', v => { settings.speed = v; });

function bindCheck(id, field) {
  document.getElementById(id).addEventListener('input', e => {
    settings[field] = e.target.checked;
  });
}

bindCheck('in-wrap', 'wrap');
bindCheck('in-corpses', 'corpses');
bindCheck('in-shuffled', 'shuffled');
bindCheck('in-trails', 'trails');
bindCheck('in-lines', 'chaseLines');

document.getElementById('in-strategy').addEventListener('input', e => {
  settings.aggressive = e.target.value === 'aggressive';
});

const pauseBtn = document.getElementById('btn-pause');
pauseBtn.addEventListener('click', () => {
  paused = !paused;
  pauseBtn.textContent = paused ? 'resume' : 'pause';
});

document.getElementById('btn-step').addEventListener('click', () => {
  if (!paused) { paused = true; pauseBtn.textContent = 'resume'; }
  if (!settled) runTick();
});

document.getElementById('btn-restart').addEventListener('click', () => {
  settings.count = Number(document.getElementById('in-count').value);
  settings.gridSize = Number(document.getElementById('in-grid').value);
  if (paused) { paused = false; pauseBtn.textContent = 'pause'; }
  seedSim();
});
```

- [ ] **Step 2: Verify every control in browser**

Reload `index.html` and check each:

- **pause** freezes motion, button reads `resume`; **resume** continues.
- **step** while running pauses first, then each click advances exactly one tick (tick counter increments by 1).
- **restart** reseeds with current slider values and un-pauses.
- **speed** slider visibly changes tick rate live (1 = crawl, 60 = frantic).
- **wrap edges** OFF: dots pile up against walls and corner captures happen; ON: they cross edges.
- **corpses block** ON after restart: gray obstacles accumulate.
- **shuffled order** OFF: sim still runs (bias is subtle — just confirm no errors and motion continues).
- **strategy** cautious: pixels prioritize fleeing; chases get noticeably less decisive.
- **trails** OFF: crisp dots, no streaks. ON: streaks.
- **chase lines** ON: thin lines from each pixel to its prey; lines disappear for dead pairs.
- No console errors after exercising everything.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Wire all controls: pause/step/restart, live toggles, speed"
```

---

### Task 4: End-to-end verification pass

**Files:**
- Modify: none expected (fixes only if checks fail)

**Interfaces:**
- Consumes: the complete sketch.

- [ ] **Step 1: Behavioral checklist against the spec**

Run through with the browser open (screenshots via chrome-devtools MCP if driving headlessly):

1. Fresh load: 100 pixels, rainbow hues, wrap ON, motion within a second.
2. Watch a capture: alive decreases by exactly 1 and captures increases by exactly 1, always in lockstep.
3. Corpses ON (after restart): every capture leaves a gray cell; gray cells never disappear; living pixels visibly route around them.
4. Corpses OFF: catcher occupies the prey's former cell (pause + step near an imminent capture to observe).
5. Walls mode: evaders get pinned at edges/corners; sim resolves faster than wrap mode.
6. Let a run finish: `settled` appears, tick counter stops, page stays responsive.
7. Edge case: pixels = 10, grid = 120 (sparse) and pixels = 500, grid = 40 (dense, 31% full) both run without errors or hangs.
8. Performance: 500 pixels at 60 ticks/sec keeps the page responsive.

- [ ] **Step 2: Fix anything that failed, re-verify, and commit fixes**

```bash
git add index.html
git commit -m "Fix issues found in end-to-end verification"
```

(Skip the commit if nothing needed fixing.)
