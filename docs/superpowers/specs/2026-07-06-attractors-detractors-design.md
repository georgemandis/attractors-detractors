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

**Candidate moves.** The 8 Moore neighbors plus staying put, minus cells that
are occupied by living pixels, corpses, or off-grid (walls mode).

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

No retargeting in v1: a successful catcher never chases again.

**End state.** When no living pixel has a living prey, nothing can change;
the sim shows a "settled" banner and stops advancing captures (pixels may
still be fleeing but no further deaths are possible, so the loop can pause).

## Grid topology

Toggleable: wrap-around (torus) or hard walls. Wrap keeps chases alive and
lets mill/spiral dynamics emerge; walls create corner traps and faster, more
brutal resolutions.

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
| Speed (ticks/sec) | slider | ~10 |
| Pause / single-step | buttons | running |
| Topology | toggle wrap/walls | wrap |
| Corpses as obstacles | toggle | off |
| Turn order | toggle shuffled/fixed | shuffled |
| Aggressive % | slider 0–100 | 50 |
| Trails | toggle | on |
| Chase lines | toggle | off |
| Restart | button (reseeds) | — |

Changing pixel count or grid size takes effect on restart; the rest apply live.

## Testing / verification

Art sketch — verification is visual and behavioral, in the browser:
captures happen and remove exactly one pixel, corpses block movement when
enabled, wrap distances behave at edges, fixed order shows its bias, the
settled state is detected.

## v2 ideas (out of scope)

- Retargeting variant: catcher inherits its victim's prey, so the ring
  shrinks but never breaks — runs down to two survivors in mutual pursuit.
- Adjacency-based or wear-down capture rules.
