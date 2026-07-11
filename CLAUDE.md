# KEYLINE

Chroma-key guided vectorizer. Part of the Grandure Studio / Ten Grand tools
ecosystem (same pattern as RIGGR, Grandure Cue, Bag Check, etc.): single-file
HTML + vanilla JS, no build step, no dependencies, deployed as a standalone
tool at tools.tengrand.co.

## What it does

Illustrator's Image Trace over-segments hand-drawn line art into hundreds of
tiny nodes because it has to *guess* where corners vs. curves are. KEYLINE
removes the guessing: the artist marks their own anchor points and guide
lines by hand (in Procreate, as separate layers), and the app uses those
marks to fit clean bezier curves exactly where the artist intended them —
same core curve-fitting math Illustrator uses (Schneider/Graphics-Gems-style
least-squares bezier fit), just applied per-segment against artist-placed
anchors instead of an automated corner-detection heuristic.

## Workflow (as the artist actually uses it)

The artist separates their character into pieces in Procreate (head base,
hair, each eye state, glasses, ...). Each piece is a layer row in the app,
built from three separate image uploads:

1. **art** — the clean linework/color for that piece
2. **dots** — anchor points marking corners, as a transparent-background
   Procreate layer with just the dots on it (any color — see Key detection)
3. **lines** — a guide line connecting the dots along the path to trace,
   same transparent-background convention

The app keys dots/lines out of their frame, skeletonizes the guide line to
a 1px centerline, snaps the artist's anchors onto it, and fits bezier
segments between consecutive anchors (auto-runs when both marker frames
are present). Pieces are often exported from different source docs/crops,
so they don't arrive pre-aligned — positioning is a per-piece touch
gesture (see Architecture notes) rather than assumed.

The UI is two tabs over one shared canvas (iPhone-portrait-first):

- **Geometry tab** — icon-only layer rows (grip drag-handle, visibility
  eye, thumbnail, name, yellow art / pink dots / cyan lines upload
  buttons, green per-piece SVG download, and a +/chevron for variants).
  Rows swipe left to reveal a delete (trash) button — this is the *only*
  place a piece can be deleted; Appearance rows have no delete affordance
  at all, since the two tabs render the same `state.layers` array —
  deleting a piece from Geometry removes it everywhere, nothing to keep
  in sync separately. The grip handle drags a row to reorder pieces
  (`wireDragHandle`): other rows shift via CSS transform only while
  dragging (classic translate-based reorder, matching the swipe/move
  gestures' pattern of "visual feedback first, mutate state once on
  release"); `state.layers` and the DOM are only touched on pointerup, as
  a single undo step. Scoped to Geometry — Appearance renders the same
  array, so order changes show up there too without its own handle.
  A piece can hold **variants** (eye open / closed / sad...)
  managed on a separate full-screen page — never nested in the main
  list. Exactly one variant (or the default) is active per piece; the
  eye icons on the variant page behave like radio buttons. Top bar:
  hamburger (project name/new/delete, trace-settings sliders, copy-SVG),
  logo, undo, redo. Project row: project switcher + Export SVG (whole
  character: visible pieces, active variants).
- **Appearance tab** — same rows (no delete), but each piece gets exactly
  ONE fill swatch, ONE stroke swatch (native color inputs), and a
  stroke-width select ("2 px" style; rendered width scales with doc
  resolution via `strokePx()` so the numbers stay display-scale). No
  per-shape fills, no region detection — the artist separates color
  regions into pieces during Geometry. Vectorizing seeds the piece fill
  by sampling the art frame inside the first closed traced shape (this
  sampling reads the art image's pixels directly and is unaffected by
  the art-underlay display toggle below).
- **Art-underlay toggle** (`state.showArt`, one bool per tab) — a small
  red dot (top-right of the canvas, `#bToggleArt`, pure CSS `::after`
  circle — no icon/label) shows/hides the raw art frame behind the
  traced geometry, tracked independently per tab: Geometry defaults OFF
  (so it reads as pure line/shape work, no color at all — `draw()` only
  fills closed curves and uses the piece's real stroke color when
  `state.tab==='app'`; Geometry always strokes in a neutral
  bone/selection-green regardless of the piece's assigned stroke),
  Appearance defaults ON (since that's where you're color-matching
  against the source art). Not persisted — resets to the defaults each
  load.
- **Workspace controls** — a background-color swatch (top-left of the
  canvas, `#bBgColor`) sets `--stage-bg` (persisted per-project,
  `bgColor` in the meta record; defaults to `--ink` on a new project);
  the grid overlay stays visible on top of any chosen color since it's
  a separate `background-image` layer. A vertical zoom panel (left edge,
  `.zoompanel`) has +/−/fit buttons (`zoomBy()`, `fitView()`) as a
  discoverable alternative to pinch-zoom.
- **Per-piece transform gestures** (Geometry tab, piece selected):
  one-finger drag moves the piece (`tx/ty`, see Positioning gesture
  above); simultaneously twisting two fingers rotates it around its own
  center (`L.rot`, radians) — computed as the angle delta between the
  two touch points each frame, independent of the same two-finger
  gesture's pinch-distance zoom (which still always controls the shared
  view). `draw()` applies translate→rotate(around origW/2,origH/2)
  →scale, in that order; `buildSVG()` mirrors it with `rotate(deg,cx,cy)`
  in the `<g transform>`, and `pieceBBox()` computes the true rotated
  AABB for the export viewBox (an unrotated w×h box would clip a
  rotated piece — e.g. ~41% wider at 45°).

Undo/redo covers structural + appearance mutations via full-state
snapshots (`takeSnap`/`applySnap`, capped at 25; images held by
reference).

## Current file

`keyline.html` — single file, open directly in a browser. No build/install
step. Uses Canvas 2D, vanilla JS, no external JS dependencies (Google Fonts
link only).

Supporting files: `index.html` (landing page linking into the tool),
`manifest.webmanifest` + `icons/` (PWA install support — add-to-home-screen
on iOS launches straight into `keyline.html` standalone; no service worker,
so it still needs network to load).

## Architecture notes

- A **content** (`newContent`) is one traceable unit: `{art, dots, lines}`
  (Image elements), `origW/origH/scale/W/H` (working-resolution downscale,
  capped at `WORK_MAX`), `anchors[]`, `paths[]` (traced polylines),
  `curves[]` (fitted beziers). A **layer** (`newLayer`) extends that with
  `visible`, appearance (`fill/stroke/strokeW`), `tx/ty` (per-layer
  position, doc-space; `s` reserved for a future per-layer scale gesture
  but not yet wired to touch input), and `variants[]` (content objects) +
  `activeVar` (-1 = default). `activeContent(L)` resolves what
  renders/exports. `vectorizeContent(L,C)` runs the trace pipeline on
  either the layer's own frames or a variant's.
- **Positioning gesture**: pieces are frequently exported from separate
  source docs/crops and don't arrive pre-aligned, so placement is a
  finger-count-driven touch convention (matches Procreate/Photoshop
  rather than a Pan/Move mode toggle) — one-finger drag on the canvas
  moves *only* the selected layer's `tx/ty` (Geometry tab only; falls
  back to panning the shared view when nothing's selected), while
  two-finger pinch always pans/zooms the shared view regardless of
  selection, so you can navigate while positioning. The move is pushed
  to undo once per drag gesture, not per pixel.
- **Key detection** (`hueMask`): if the dots/lines frame has real alpha
  transparency, every non-transparent pixel is treated as a mark —
  color-agnostic, since transparency alone identifies marks. If the frame
  is fully opaque (art visible underneath), falls back to hue-based
  matching (red for dots, ~100° neon-green for lines) since color is the
  only signal left to separate marks from art in that case. This fallback
  path is unverified against real opaque-frame uploads — the artist's
  actual workflow uses transparent-only frames, so the opaque path only
  has synthetic test coverage so far.
- **Confirm modal** (`askConfirm`): every destructive action (delete
  layer/variant/project) routes through this in-app modal, never
  `window.confirm()` — iOS silently no-ops `alert`/`confirm`/`prompt`
  for web apps launched from the home screen (standalone display
  mode), which is exactly how this app is meant to be used, so a
  native `confirm()` gate could pass through with no dialog ever
  shown and no way to cancel.
- **Skeletonization**: Zhang–Suen thinning (`thin()`) reduces the guide
  line's blob mask to a 1px centerline before tracing.
- **Path tracing** (`tracePaths`): walks the skeleton pixel-by-pixel,
  preferring the straightest continuation at junctions. This is a greedy
  walk — it does **not** correctly handle true branching guide lines
  (Y-splits, crossings). See Known Issues.
  Anchor mapping is currently done via nearest on-path pixel with a fixed
  distance cutoff (`Math.max(14, L.W*0.02)`) — not yet exposed as a
  setting.
- **Curve fitting** (`fitCubic`/`genBezier`/etc.): classic recursive bezier
  fit with Newton-Raphson reparameterization, ported from the
  Schneider/Graphics-Gems approach. Segments are cut at each on-path
  anchor; tangents are estimated from a small window of neighboring
  skeleton points (`tanAt`), not from the anchor markup itself — there's no
  current mechanism for the artist to mark "smooth" vs. "corner" anchors
  differently (see Known Issues / earlier design intent below).
- **Persistence** (`openDB`/`saveProject`/`loadProject`): auto-saves to
  IndexedDB (db `keyline`, store `project`) with multiple named
  projects. Keys: `projects` (index of `{id,name,updated}`), `cur`
  (active id), `proj:<pid>:meta` (all serializable layer state incl.
  variants + appearance, debounce-written 300ms from every mutation
  point), and `proj:<pid>:img:<contentId>:<slot>` (frame file blobs,
  written at upload time; content ids cover layers and variants). The
  active project restores silently on page load. Pre-multi-project saves
  (bare `meta` + `img:*` keys) migrate to project 1 via `migrateLegacy()`
  on first load; older metas without variants/appearance load with
  defaults, adopting the first per-curve `fill` (pre-v3 format) as the
  piece fill. All IDB calls degrade silently to in-memory-only if
  unavailable (private browsing).
- **Fill seeding** (`polygonInteriorPoint`/`artPixelColor`): on first
  trace of a piece, finds a point guaranteed inside the first closed
  traced shape via a horizontal-scanline test (centroid alone can land
  outside on concave shapes), samples the art frame there, and seeds the
  piece's single `fill`. The artist owns the value afterward via the
  Appearance tab; open paths always export `fill="none"`. If the sampled
  point lands on a boundary/anti-aliased pixel, the seeded fill can be
  off — that's what the Appearance override is for.

## Known issues / open items

- **Branching guide lines aren't supported.** If a Part's guide line has a
  Y-split or crosses itself, the skeleton becomes a graph rather than a
  path, and the greedy walk in `tracePaths` will pick one branch and
  silently drop the other. Confirmed in practice: a hair part with
  multiple crossing strands fragmented into 8 disconnected paths with a
  garbled fit, while splitting each strand into its own Part traced
  cleanly. No detection/warning exists yet for when a part's guide line
  fragments into more paths than expected — worth adding given how easy
  this mistake is to make.
- **No corner-vs-smooth anchor distinction.** An earlier design pass
  explored two dot colors (red = hard corner, blue = tangent-continuous
  smooth point) so fitted curves could match the artist's intent at each
  anchor, not just its position. This was dropped when the workflow moved
  to transparent-only mark frames where color is deliberately not
  meaningful. Worth revisiting as a *shape* distinction instead (e.g. dot
  size, or a second mark layer) if curve quality at anchors becomes an
  issue in practice.
- **Opaque-background fallback (hue-based) is synthetic-tested only.**
  Confirmed via synthetic multi-color-background tests in this chat, but
  never against a real opaque Procreate export. Low priority since the
  artist's real workflow is transparent-only, but don't assume it's
  battle-tested.
- **Export is single-resolution SVG only.** No PDF/AI-native export path,
  no per-part stroke-width override, no way to re-import a previously
  exported SVG to keep refining.
- **No project export/import.** Multiple named projects now persist via
  IndexedDB (see Architecture notes), but there's no way to export a
  project as a file for backup or to move it between devices — storage
  is per-browser-per-device, and clearing Safari website data deletes
  everything. `navigator.storage.persist()` is requested but the OS can
  still evict storage under pressure.
- **Mobile gesture conflicts.** Pan/zoom and move/scale share the same
  two-finger pinch gesture disambiguated only by a Pan/Move-part UI toggle;
  hasn't been stress-tested on an actual iPad session yet.

## Testing approach used so far

No formal test suite. Verification in this chat was done by extracting the
pure algorithm functions (`hueMask`, `thin`, `tracePaths`, `fitCurveSeg`,
`findBlobs`) out of the DOM-coupled script and running them directly in
Node against synthetic pixel data (a generated ring + anchors, and a
synthetic busy-background scenario) to confirm masking/skeletonization/
curve-fit correctness independent of browser quirks. `node --check` was
used for syntax validation before every shipped version. If you add a test
setup, extracting these pure functions into their own module (rather than
one monolithic inline `<script>`) would make this much less awkward —
that's probably the natural first refactor if this repo grows.

## Conventions carried over from the rest of the Grandure tools ecosystem

- Single-file HTML + vanilla JS + no build step, matching Bag Check, Dream
  Catcher, Care Remote, RIGGR, etc.
- `node --check` syntax validation before every delivered version.
- Dark UI, IBM Plex Mono for data/mono text, Archivo variable font for
  display — reuse this rather than introducing a new type system unless
  there's a reason to diverge.
