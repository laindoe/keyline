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

Each character part (head, a single loc, an arm, etc.) is a "Part" / layer
in the app, and is built from three separate image uploads:

1. **art** — the clean linework/color for that part
2. **dots** — anchor points marking corners, as a transparent-background
   Procreate layer with just the dots on it (any color — see Key detection)
3. **lines** — a guide line connecting the dots along the path to trace,
   same transparent-background convention

The app keys dots/lines out of their frame, skeletonizes the guide line to
a 1px centerline, snaps the artist's anchors onto it, and fits bezier
segments between consecutive anchors. Multiple Parts compose together
RIGGR-style: select a part, drag/pinch to position it, export one SVG with
a named `<g>` per part.

## Current file

`keyline.html` — single file, open directly in a browser. No build/install
step. Uses Canvas 2D, vanilla JS, no external JS dependencies (Google Fonts
link only).

Supporting files: `index.html` (landing page linking into the tool),
`manifest.webmanifest` + `icons/` (PWA install support — add-to-home-screen
on iOS launches straight into `keyline.html` standalone; no service worker,
so it still needs network to load).

## Architecture notes

- `state.layers[]` holds one object per Part: `{art, dots, lines}` (Image
  elements), `origW/origH/scale/W/H` (working-resolution downscale, capped
  at `WORK_MAX`), `anchors[]`, `paths[]` (traced polylines), `curves[]`
  (fitted beziers), `tx/ty/s` (compose-mode transform).
- **Key detection** (`hueMask`): if the dots/lines frame has real alpha
  transparency, every non-transparent pixel is treated as a mark —
  color-agnostic, since transparency alone identifies marks. If the frame
  is fully opaque (art visible underneath), falls back to hue-based
  matching (red for dots, ~100° neon-green for lines) since color is the
  only signal left to separate marks from art in that case. This fallback
  path is unverified against real opaque-frame uploads — the artist's
  actual workflow uses transparent-only frames, so the opaque path only
  has synthetic test coverage so far.
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
- **Persistence** (`openDB`/`saveProject`/`restoreProject`): auto-saves
  the whole project to IndexedDB (db `keyline`, store `project`) — each
  frame's original file blob under `img:<layerId>:<slot>` written at
  upload time, plus one `meta` record with all serializable layer state,
  debounce-written (300ms) from every mutation point (upload, vectorize,
  rename, toggle, delete, move/pinch end, slider change). Restored
  silently on page load; "Clear saved project" button wipes the store.
  All IDB calls degrade silently to in-memory-only if unavailable
  (private browsing).
- **Auto-fill** (`polygonInteriorPoint`/`artPixelColor`): for each closed
  traced shape, finds a point guaranteed to be inside it via a
  horizontal-scanline test (centroid alone can land outside on concave
  shapes), then samples the artist's own `art` frame at that point and
  bakes the result in as the shape's fill — both in the canvas preview and
  the exported SVG (`fill="none"` stays the default for open paths). No
  manual color-picker override exists yet; if the sampled point lands on a
  boundary/anti-aliased pixel or a sliver shape, the fill can pick up the
  wrong region's color.

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
- **Project persistence is single-project only.** Auto-save/restore via
  IndexedDB now exists (original file blobs per frame + one meta record;
  see Architecture notes), but there's exactly one project slot — no way
  to keep several characters as separate saved projects, and no
  export/import of a project file for backup or moving between devices.
  `navigator.storage.persist()` is requested but the OS can still evict
  storage under pressure.
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
