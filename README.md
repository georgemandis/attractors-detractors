# attractors / detractors

Cyclic pursuit on a grid: a ring of pixels where each one chases its
successor and flees its predecessor — and the pixel you fear is exactly the
one hunting you. Captures shrink the ring; parameters change everything.

**Play it: https://georgemandis.github.io/attractors-detractors/**

A single-file [p5.js](https://p5js.org/) sketch. No build step — open
`index.html` in a browser, or use the link above.

## The rules

- N pixels form a ring: pixel *i* chases *i+1* and avoids *i−1*.
- Each tick a pixel picks its best move lexicographically: *aggressive*
  pixels prioritize closing on prey, *cautious* ones prioritize escaping
  their hunter.
- Moving onto your prey's cell is a capture. With *inherit prey* on, the
  catcher takes over its victim's chase and the ring never breaks.
- Dead pixels linger as gray bodies; optionally they block movement.

## Things to try

- **Recipes** — preset parameter sets (battle royale, ecosystem panic,
  graveyard maze, …) from the presets dropdown.
- **Shapes** — draw a circle, spiral, or lattice as one chain and set it
  loose; the pixels slider controls its density.
- **Draw your own** — pause, drag a shape onto the grid, resume.
- **Share** — the URL hash tracks every setting *and the run seed*; the
  share button copies a link that replays your exact run, capture for
  capture.
- **Go 3D** — run the same rules on the surface of a cube, sphere, gem, or
  torus (the torus is literally the flat wrap-mode grid, finally visible as
  the donut it always was). Drag to orbit.

## Determinism

Every run draws a seed (`sd=` in the share hash), and **all** simulation
randomness flows from that one seeded stream:

- initial pixel placement (seed and restart)
- each pixel's temperament (the aggressive/cautious draw) — at seeding,
  when click/drag painting, when drawing shape presets, and for
  evolution births
- shuffled turn order, every tick
- tie-breaks between equally good moves
- jitter moves
- wander moves (searching for out-of-sight targets)
- evolution: which survivor parents a newborn, its temperament mutation,
  its hue drift, and its spawn cell

The stream is re-anchored at three points: **page load** (uses the hash
seed if present, otherwise draws fresh), **restart** (always draws a fresh
seed), and **clear** — including shape presets, which clear first —
(rewinds to the current run's seed, so "clear → shape → resume" replays
identically no matter when you click). What is *not* covered: live human
intervention. Painting cells or changing sliders mid-run perturbs a replay,
because your timing and pointer aren't part of the seed.

## Parameters

Topology (wrap/walls), corpses as obstacles, turn order, prey inheritance,
movement neighborhoods (8-way / orthogonal / diagonal), aggressive/cautious
mix, random jitter, vision radius, speed classes, hunger, multiple
independent rings (with optional rock-paper-scissors rivalry between them),
evolution (deaths spawn mutated children — watch temperament drift under
selection), drawable walls, sonification, click-to-follow a pixel, WebM
recording, trails, chase lines — and 3D surfaces: run the same rules on the
surface of a cube, sphere, octahedral gem, or torus (drag to orbit). Works
on phones. See `docs/superpowers/specs/` for the full design.
