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
- **Share** — the URL hash tracks every setting; the share button copies a
  link that reproduces your setup.

## Parameters

Topology (wrap/walls), corpses as obstacles, turn order, prey inheritance,
movement neighborhoods (8-way / orthogonal / diagonal), aggressive/cautious
mix, random jitter, vision radius, speed classes, hunger, multiple
independent rings, trails, and chase lines. See
`docs/superpowers/specs/` for the full design.
