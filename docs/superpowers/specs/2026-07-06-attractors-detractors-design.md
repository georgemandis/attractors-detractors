# Attractors / Detractors — Design

A p5.js sketch exploring cyclic pursuit on a grid: a ring of pixels where each
one chases its successor and flees its predecessor. Date: 2026-07-06.

## Concept

N pixels are placed on random distinct cells of a W×H grid, forming one ring:
pixel *i* chases pixel *i+1* and avoids pixel *i−1* (mod N). Because the ring
is cyclic, the pixel you avoid is exactly the pixel hunting you. Captures
break the ring apart: each capture produces one pure evader (the catcher, whose
own hunter still lives) and one pure chaser (the victim's target, whose hunter
is now dead). The simulation winds down as chase pairs are eliminated.

## Deliverable

A single self-contained file: `index.html` at the project root. p5.js loaded
from CDN, one inline `<script>` containing the simulation, plain HTML controls.
No build step; open in a browser and it runs.

## Simulation rules

**Tick loop.** Each tick, every living pixel takes one move, resolved per the
turn-order setting:

- *Shuffled* (default): pixels move one at a time in a freshly shuffled order
  each tick. No cell conflicts are possible.
- *Fixed ring order*: always pixel 0 first. Deliberately biased — an
  observable experiment in first-mover advantage.

**Candidate moves.** The selected neighborhood plus staying put, minus cells
that are occupied by living pixels, corpses, or off-grid (walls mode). The
*moves* setting picks the neighborhood: all 8 (default), orthogonal only, or
diagonal only. Diagonal movers can never change checkerboard parity, so in
diagonal mode landing orthogonally adjacent to the prey also counts as a
capture (the body stays at the prey's cell; the catcher completes its move).

**Jitter.** Each pixel has a per-tick chance (the *jitter %* slider) of
ignoring strategy and taking a uniformly random legal move. Random blunders
onto the prey's cell still capture. 0 is fully deterministic strategy;
100 is a random walk.

**Speed classes.** With *fast hunters* (or *fast cowards*), the other
temperament class moves only on even ticks — half speed. Class membership
follows the live aggressive % slider.

**Multiple rings.** The *rings* slider (applied on restart) splits the seeded
population into K independent chains. Rings never interlink — captures and
inheritance stay within a ring — so they interact only by competing for
space. Each ring gets its own hue family (~60° band); a single ring keeps
the full rainbow. Click-added pixels splice into ring 0's seam.

**Hunger.** With *starve after* set, any pixel with a living prey runs a
starvation clock; a kill resets it, and exceeding the limit kills the hunter
(body follows the usual ghost/corpse rules; with inherit on, the ring closes
around the starved pixel). Good evasion becomes a weapon. The stats line
shows a starved count. With inherit + hunger on, runs collapse to a single
survivor.

**Vision.** Pixels only react to prey/hunters within the *vision* radius
(Euclidean, torus-aware; ∞ by default). A pixel whose living targets are all
out of sight wanders randomly in search of them; chases ignite on encounter.
Capture requires an in-sight chase — a wanderer cannot blunder onto unseen
prey.

**Distance.** Euclidean. In wrap mode, torus-aware (minimum over wrapped
offsets on each axis).

**Move scoring — lexicographic.**

- *Aggressive*: sort candidates by (1) smaller distance to prey, then
  (2) larger distance to hunter. So "closer to prey, same distance to
  hunter" beats "same distance to prey, farther from hunter."
- *Cautious*: same, with the two priorities swapped.
- Ties after both criteria are broken uniformly at random.
- A pixel whose prey is dead scores only on fleeing its hunter.
- A pixel whose hunter is dead scores only on chasing its prey.
- A pixel with neither stands still.

Strategy is per-pixel: each pixel draws a fixed random temperament in [0,1)
at seed time and plays aggressive while `temperament × 100 <` the
*aggressive %* slider (0 = all cautious, 100 = all aggressive, default 50).
The slider applies live and converts pixels monotonically as it moves.

**Capture.** If a pixel's chosen move is its prey's cell, that is a catch and
the prey dies. Moving onto the prey's cell is the single exception to the
occupancy rule.

- Corpses OFF: the catcher takes the vacated cell.
- Corpses ON: the prey's cell becomes a permanent obstacle (corpse) and the
  catcher stays where it was.

Prey/hunter relations are explicit per-pixel links (seeded as the ring
`i → i+1 mod N`), and the *inherit prey* toggle picks what a capture does
to them:

- Inherit ON (default): the catcher inherits its victim's prey, and that
  prey now fears the catcher — the ring shrinks but never breaks, so every
  living pixel always has someone to chase and someone to fear, down to a
  final mutual pursuit.
- Inherit OFF: retire-on-catch — the catcher never chases again and its
  victim's prey loses its hunter, so captures fragment the ring.

**End state.** When no living pixel has a living prey, nothing can change;
the sim shows a "settled" banner and stops advancing captures (pixels may
still be fleeing but no further deaths are possible, so the loop can pause).

## Grid topology

Toggleable: wrap-around (torus) or hard walls. Wrap keeps chases alive and
lets mill/spiral dynamics emerge; walls create corner traps and faster, more
brutal resolutions.

## 3D surfaces

The *surface* select (on restart) offers flat plus three solids that share
one topology: the cube surface — six G×G faces stitched at the edges, every
cell with exactly 4 orthogonal neighbors and 3–4 diagonal ones (diagonals
are cells reachable via two distinct orthogonal intermediates, which stays
valid across edges and cube corners). Cell positions on the unit cube are
projected radially — *sphere* (L2 normalize) or *gem*/octahedron (L1
normalize) — and move scoring uses squared distance in the projected space,
so the solid changes the dynamics, not just the look. Rendered in WEBGL with
orbit controls; trails become fading per-pixel history; the wrap toggle is
inert (closed surfaces); click-to-add and shape presets are flat-only.
Vision scales by cell size (2/G) to stay comparable. Switching surfaces
writes the state hash and reloads the page, since p5 cannot reliably swap
P2D and WEBGL renderers on a live canvas.

A fifth surface, *torus*, embeds the flat wrap-mode grid as a donut. Its
metric is the same wrapped grid distance as flat mode (scaled by 2/G), so
dynamics are identical to flat+wrap — only the view changes.

## Seeded runs

Every seed draws a run seed (`randomSeed`), carried in the share hash as
`sd=`. Opening a link replays the exact run tick-for-tick (placement,
tempers, shuffles, tie-breaks all flow from the seeded RNG); restart draws
a fresh seed. Live interaction (clicks, mid-run slider changes) perturbs a
replay, which is accepted.

## Later additions (2026-07-08)

- **Sonification** (toggle, default off): captures ping pentatonic notes
  pitched by the victim's hue; starvations rumble low. Raw WebAudio, max 8
  voices per tick; the checkbox click is the audio-unlock gesture.
- **Wall paintbrush**: a brush select (pixels / walls / erase walls). Walls
  are steel-blue terrain cells that block movement and seeding, survive
  restarts, and are cleared by the clear button. Flat mode only; not in the
  share hash.
- **Ring rivalry** (toggle): with K>1 rings, any pixel may trample any
  pixel of ring (r+1) mod K by landing on it — opportunistic (movement is
  still chain-driven). The victim's own ring closes around it; normal
  corpse rules apply. Spatial rock-paper-scissors.
- **Click to follow**: clicking a living pixel (pixels brush) highlights it
  with a white ring, a green line to its prey and red to its hunter, plus a
  bio line (temperament, kills, hunger). Click again to unfollow.
- **Evolution** (toggle): every death spawns a child of a random survivor
  at a random empty cell, spliced in behind its parent. Temperament mutates
  ±0.1, hue drifts ±12° (lineages become color families). Population stays
  constant; stats gain a born counter. First observed result: under hunger
  pressure the population evolves cowardice (mean temperament 0.50 → 0.88
  in 800 ticks).
- **Recording**: a rec button captures the canvas to a downloadable WebM
  via MediaRecorder (30 fps).
- **Touch / responsive**: below 900px the panel stacks under a full-width
  CSS-scaled canvas; p5 corrects pointer coordinates for the scaling, and
  `touch-action: none` keeps drawing/orbiting from fighting the scroll.

## Sparkline and winner card

The panel shows a small population-over-time canvas: alive (green),
cumulative captures (red), cumulative starved (yellow), scaled to the
population high-water mark; history decimates by 2 beyond 4096 ticks.
When a run settles with exactly one survivor, a winner card shows its
color, index, temperament, kill count (tracked per pixel), and the tick.

## Visuals

- Each pixel colored by ring position mapped around the hue wheel, so the
  chain is visible: every pixel chases the next hue over.
- Dead pixels stay visible at their death spot regardless of the corpses
  toggle; the toggle only controls whether they block movement, and it
  applies retroactively (toggling ON solidifies every existing body,
  toggling OFF releases them all). Walkable bodies render as faint gray
  ghosts; blocking corpses render solid gray. Because ghosts are walkable,
  a pixel can die on top of an older body — blocking is tracked per cell.
- Optional fading trails (toggle).
- Optional thin lines from each chaser to its living prey (toggle).
- Stats line: tick count, alive count, capture count.

## Controls

| Control | Type | Default |
|---|---|---|
| Pixel count | slider 10–500 | 100 |
| Grid size | slider (square, e.g. 40–120) | 80 |
| Rings | slider 1–8 (on restart) | 1 |
| Surface | select flat / cube / sphere / gem (on restart) | flat |
| Speed (ticks/sec) | slider | ~10 |
| Pause / single-step | buttons | running |
| Topology | toggle wrap/walls | wrap |
| Corpses as obstacles | toggle | off |
| Turn order | toggle shuffled/fixed | shuffled |
| Inherit prey | toggle | on |
| Aggressive % | slider 0–100 | 50 |
| Jitter % | slider 0–100 | 0 |
| Moves | select all 8 / orthogonal / diagonal | all 8 |
| Vision | slider 2–41 cells, or ∞ | ∞ |
| Speed classes | select uniform / fast hunters / fast cowards | uniform |
| Starve after | slider 20–400 ticks, or off | off |
| Trails | toggle | on |
| Chase lines | toggle | off |
| Restart | button (reseeds) | — |

Changing pixel count or grid size takes effect on restart; the rest apply live.

## Presets

Two selects compose like the manual workflow (recipe first, then shape):

- *Recipe* applies a complete named parameter set (battle royale, ecosystem
  panic, cautious standoff, graveyard maze, drunken brawl, tortoise hunt),
  syncs the UI, and reseeds. Every recipe spells out all sim parameters so
  leftovers from play never leak in; count/grid/speed/trails/lines stay
  user-controlled.
- *Shape* clears the grid, pauses, and draws a preset shape (circle, square,
  spiral, line, lattice) as one chain in path order with a smooth rainbow
  along the path. The pixels slider sets the sampling density along the
  path — 10 on a circle is a decagon of distant chasers, 500 the solid ring
  (capped at the path length). Tweak parameters, then resume. The select
  snaps back to "—" after drawing so the same shape can be re-picked.

## Shareable state

The URL hash mirrors all sim settings (short keys, e.g. `#n=100&g=80&a=50…`)
plus the active shape preset, updated on every control change via
`history.replaceState` (no history spam). Opening a hashed link applies the
settings, syncs the UI, reseeds, and re-draws the shape paused. Values are
clamped and validated on parse. The *share* button copies the link
(clipboard with prompt fallback). Hosted via GitHub Pages so links work
anywhere.

## Click to add / clear

A *clear* button empties the grid. Clicking any empty cell (walkable ghost
cells count as empty; blocking corpses and living pixels don't) adds a pixel
there, at any time — into an empty grid or a running sim. Holding the button
and dragging paints pixels continuously: every cell the pointer crosses is
added, with interpolation between drag samples so fast strokes leave no gaps.
(Drawing into a running sim is chaotic by design — each painted pixel appears
adjacent to its hunter — so pause first to draw clean shapes.) New pixels
chain in click order: each is chased by the previously added pixel and chases the
sticky "chain head" (the first pixel), so the loop always closes. On a seeded
run the head is pixel 0, so additions splice into the ring's seam. Added
pixels draw a random temperament (governed live by the aggressive % slider)
and get golden-angle-spaced hues. Adding a chase pair to a settled or empty
grid resumes ticking automatically.

## Testing / verification

Art sketch — verification is visual and behavioral, in the browser:
captures happen and remove exactly one pixel, corpses block movement when
enabled, wrap distances behave at edges, fixed order shows its bias, the
settled state is detected.

## v2 ideas (out of scope)

- Adjacency-based or wear-down capture rules.
