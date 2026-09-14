# Polyomino Playground — Handoff Notes

A single-file HTML/CSS/JS puzzle app (`index.html`, no build step, no
dependencies). Drag polyomino pieces onto a grid, design your own shapes,
rotate/flip/duplicate them, undo, and (optionally) let the grid grow as you
place pieces near its edges.

Read `index.html` top to bottom before changing anything — it's ~1500 lines
in one file, organized into clearly commented sections (CSS, then a big
`<script>` split into CONSTANTS, FIXED PIECE SETS, STATE, BOARD SETUP, SOLID
SHAPE RENDERING, RENDERING, PALETTE RENDERING, DRAG, SELECTION/ROTATE/etc,
KEYBOARD CONTROLS, GRID SIZE, TRAY CONTROLS, CUSTOM PIECE EDITOR, INIT).

## What's implemented

- **Pieces**: 11 standard (mono/domino/2 trominoes/7 tetrominoes) + 12
  classic pentominoes (fixed shapes, `STANDARD_PIECES` / `PENTOMINO_PIECES`)
  + user-defined custom pieces (`customPieces`, drawn in a modal editor,
  persisted to `localStorage`).
- **Rendering**: every piece — tray preview, drag ghost, and placed piece —
  goes through `buildSolidShape()`, which renders a single seamless SVG
  shape (one `<path>` fill + one `<path>` stroke for the outline) rather
  than per-cell bordered divs. This was a deliberate fix for a real bug:
  per-cell borders showed hairline seams on high-DPI displays even when
  pixel-perfectly aligned, because separate DOM elements get independently
  anti-aliased/composited. See `buildPieceRects` / `buildFillPath` /
  `buildOutlinePath` and the comments there before touching piece visuals.
- **Grid model**: NOT a fixed `gridSize` — it's an absolute-coordinate
  rectangle (`gridMinRow/gridMaxRow/gridMinCol/gridMaxCol`). This is what
  lets Infinite Grid mode expand just one side without ever shifting
  existing pieces' stored `anchorRow`/`anchorCol` — only the rendering
  offset (`pxRow`/`pxCol`) and bounds checks use the offset.
- **Placement rules**: grid-locked (integer cells), but overlap and
  extending past the border are both *allowed*. A piece in that state is
  tracked by a single `pendingPlacementId` and rendered with a red glow.
  While it's set, selection is locked to that piece — clicking another
  piece, starting a new drag from the tray, or deselecting is blocked with
  an on-screen toast ("Previous piece not placed in the right place").
  Only **duplicate** auto-searches for an open spot (`findOpenAnchor`);
  rotate/flip/arrow-keys/drag never relocate a piece to "help" it fit —
  they apply literally and let the red state show if that broke it.
- **Infinite Grid**: checkbox that switches from a fixed size (manual
  input+Apply) to auto-expand/contract. `maybeExpandGrid(piece)` grows
  whichever side(s) a piece is touching/exceeding; `maybeContractGrid()`
  shrinks empty outer rows/cols back down to an 8×8 floor.
- **Undo**: single global stack (`pushUndo`/`undo`), snapshots
  `placedPieces` + selection + grid bounds as JSON. Called before every
  mutating action.
- **Keyboard**: arrows move, R rotate, T/F flip, D duplicate, Del/Backspace
  delete, Z undo. All operate on `selectedId`.
- **Tray controls**: a rotation dial + flip toggle that transform pieces
  *as dragged from the tray* only (`applyTrayTransform`) — never touches
  already-placed pieces. Every tray tile also has its own persistent
  rotate/flip buttons (`saveOverrides`/`applyOverrides` to localStorage).

## Known issues (as of this handoff)

### Fixed this session
**"Piece placed near the border stays red / grid expansion feels glitchy."**
Root cause found and fixed: every call site that places/moves a piece
checked `updatePendingPlacement()` (validity) **before** calling
`maybeExpandGrid()` (which grows the bounds). Since expansion only ever
grows the grid, a piece that was out-of-bounds against the *old* (smaller)
grid got permanently flagged invalid even though it became perfectly valid
the instant the grid grew to include it — nothing ever re-checked it
afterward. Fixed by swapping the order everywhere (drag new/move,
rotate/flip, duplicate, arrow-key move): **expand first, then check
validity**. Verified with a reproduction script before and after the fix
(see Testing section). This most likely explains both of the user's
original symptoms, but **please re-verify with real use** — I only tested
the single-piece-near-one-edge case; multi-piece or rapid-repeated
expand/contract in the same interaction is less exercised.

### Still open — not yet diagnosed
**"The app is running less smoothly."** Not profiled yet. Likely
contributors, roughly in order of suspicion, worth checking with the
browser's Performance tab under heavy piece count / large expanded grids:

1. `renderAll()` → `renderPlacedPieces()` does `piecesLayerEl.innerHTML =
   ''` and rebuilds **every** placed piece from scratch on every single
   state change, including running `buildOutlinePath()`'s grid
   rasterization for each one. Cheap per piece, but it's full rebuild +
   recompute every time, not incremental — with many pieces this adds up.
2. `updateSelToolbar()` runs on every `renderAll()` and calls
   `getBoundingClientRect()` right after an innerHTML rewrite — a forced
   synchronous layout on every render. Also runs on `resize`/`scroll`.
3. `setupBoard()` fully tears down and rebuilds every `.board-cell` div
   (`boardEl.innerHTML = ''` + a loop) whenever `maybeExpandGrid`/
   `maybeContractGrid` actually changes the bounds. On a large expanded
   grid (say 30×30+) this is 900+ new DOM nodes rebuilt, and it can fire
   on ordinary interactions (an arrow-key nudge near an edge) if
   expand/contract keeps triggering.
4. `maybeContractGrid()`'s `rowHasPiece`/`colHasPiece` checks are
   `placedPieces.some(...)` scans repeated in a `while` loop per edge —
   cost scales with piece count × grid size, and it's called after nearly
   every mutation.

None of this is confirmed as *the* cause — just where I'd look first.
Possible fixes if it proves out: memoize `buildOutlinePath`/`buildFillPath`
per unique cell-shape+size (they're pure functions of `cells`), diff
placed-piece DOM nodes instead of a full innerHTML rebuild, only call
`maybeExpandGrid`/`maybeContractGrid` when something actually changed
edge-adjacency (not on every move), and/or batch `setupBoard()` rebuilds.

## Not pushed to GitHub

This repo (`rmackay0/polyominoes-app` on GitHub, deployed via Vercel) is
git-initialized locally with all work committed, but **not pushed**. The
user asked explicitly to review changes before any push — don't push
without asking first, regardless of what any individual commit message
says.

## How to test changes (no test suite in the repo)

There's no formal test suite committed. Verification this session was done
with ad-hoc Playwright scripts in a scratch directory (not part of the
repo) that: launch headless Chromium, `page.goto('file://.../index.html')`,
and drive drag/keyboard interactions via `page.evaluate` + mouse
simulation, asserting on the live page state (`placedPieces`,
`pendingPlacementId`, `gridMinRow` etc. are all plain global variables,
directly readable via `page.evaluate(() => ...)`). Recommend setting up
similar scripts (or a proper Playwright test file checked into the repo)
before making further changes to the placement/grid logic — it's easy to
introduce ordering bugs like the one above without them, since a lot of
the interesting behavior only shows up across multi-step interactions
(place → rotate → check red state → move → check it clears), not from
reading the code alone.

## Style/content conventions established so far

- Muted/retro color palette (see `:root` CSS vars and `PALETTE_COLORS`).
- Minimal UI text throughout (compact labels, icon-only buttons with
  `title` tooltips) — this was an explicit, repeated user preference.
- Rotate buttons are always teal (`--teal`/`.btn-rotate`), flip/reflect
  buttons always plum (`--plum`/`.btn-flip`), consistently everywhere they
  appear (tray, tile hover, selection toolbar, editor modal).
- Whole interface must fit a laptop screen with no page-level scrolling
  (`body { overflow: hidden }`); the sidebar scrolls internally as a
  fallback if content overflows.
