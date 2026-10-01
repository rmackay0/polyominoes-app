# Polyomino Playground — Handoff Notes

A single-file HTML/CSS/JS puzzle app (`index.html`, no build step, no
dependencies). Drag polyomino pieces onto a grid, design your own shapes,
rotate/flip/duplicate/delete them (single or multi-select), undo, toggle a
checkerboard overlay with live counts, box-select with a marquee, and
(optionally) let the grid grow as you place pieces near its edges, or resize
it manually by typing a number — no Apply button needed.

Read `index.html` top to bottom before changing anything — it's organized
into clearly commented sections (CSS, then a big `<script>` split into
CONSTANTS, FIXED PIECE SETS, STATE, BOARD SETUP, SOLID SHAPE RENDERING,
RENDERING, PALETTE RENDERING, DRAG, SELECTION/ROTATE/etc, KEYBOARD CONTROLS,
GRID SIZE, TRAY CONTROLS, CUSTOM PIECE EDITOR, INIT).

## What's implemented

- **Pieces**: 11 standard (mono/domino/2 trominoes/7 tetrominoes) + 12
  classic pentominoes (fixed shapes, `STANDARD_PIECES` / `PENTOMINO_PIECES`)
  + user-defined custom pieces (`customPieces`, drawn in a modal editor,
  persisted to `localStorage`).
- **Rendering**: every piece — tray preview, drag ghost, and placed piece —
  goes through `buildSolidShape()`, which renders a single seamless SVG
  shape (one `<path>` fill + one `<path>` stroke for the outline) rather
  than per-cell bordered divs. This was a deliberate fix for a real bug:
  per-cell borders showed hairline seams on high-DPI displays, and jagged
  "flag" artifacts at concave corners (T/S/Z pieces). The outline is traced
  via a fine-grid rasterization boundary-walk, not per-cell borders. See
  `buildPieceRects` / `buildFillPath` / `buildOutlinePath`.
- **Grid model**: NOT a fixed `gridSize` — it's an absolute-coordinate
  rectangle (`gridMinRow/gridMaxRow/gridMinCol/gridMaxCol`). This is what
  lets Infinite Grid mode expand just one side without ever shifting
  existing pieces' stored `anchorRow`/`anchorCol` — only the rendering
  offset (`pxRow`/`pxCol`) and bounds checks use the offset. Max manual grid
  size is 20×20.
- **Grid resize — manual, no Apply button**: `#grid-size-input` resizes live.
  Typing + Enter, typing + blur, and the native spinner/arrow-key
  interactions all trigger `attemptGridResize()` via a `change` listener
  (native inputs fire `change` immediately for spinner/arrow-key bumps, only
  on blur for typed text — this distinction is why both paths are wired).
  Validation is two-tier: `attemptGridResize()` first checks the value is a
  well-formed integer in range (1–20), then checks the new bounds via
  `piecesFitInBounds()` against every existing placed piece. On failure,
  `showGridSizeError(msg)` displays an inline error and the resize is
  **blocked** — existing pieces and grid bounds are left untouched. On
  success, existing pieces are preserved in place (bounds just grow/shrink
  around them). `dismissGridSizeError()` clears the error and reverts the
  input to `Math.max(gridRows(), gridCols())`. The Enter key is overloaded:
  if an error is currently showing, Enter **dismisses** it (does not
  re-attempt); otherwise it attempts a resize and only blurs the input on
  success — this was a self-caught bug (originally blurred unconditionally,
  which silently broke "press Enter again to dismiss" because focus was
  already gone for the second press).
- **Placement rules**: grid-locked (integer cells), but overlap and
  extending past the border are both *allowed*. Any piece in that state is
  tracked in `pendingPlacementIds` (a `Set`, supports multiple at once) and
  rendered with a red glow. While any piece is pending, selection is locked
  to pending pieces only (`isLocked()`/`canInteractWith(id)`) — clicking
  another piece, starting a new drag from the tray, or deselecting is
  blocked with an on-screen toast ("Previous piece not placed in the right
  place"). Only **duplicate** auto-searches for an open spot
  (`findOpenAnchor`); rotate/flip/arrow-keys/drag never relocate a piece to
  "help" it fit — they apply literally and let the red state show if that
  broke it. **Critical ordering rule**: `updatePendingPlacement(pieceId)`
  must always be called **after** `maybeExpandGrid()`, never before — doing
  it in the wrong order was a real, previously-shipped bug (see Known
  issues below) and it's an easy mistake to reintroduce at a new call site.
- **Infinite Grid**: checkbox that switches from a fixed size (manual
  input) to auto-expand/contract. `maybeExpandGrid(piece)` grows whichever
  side(s) a piece is touching/exceeding (`EXPAND_BUFFER = 2`);
  `maybeContractGrid()` shrinks empty outer rows/cols back down to an
  `INFINITE_MIN_DIM = 8` floor.
- **Selection model — multi-select**: `selectedIds` and
  `pendingPlacementIds` are both `Set`s (generalized from earlier
  single-id variables). Shift-click adds/removes a piece from the
  selection. `updateSelToolbar()` shows Rotate/Flip/Delete for any
  selection size, and adds Duplicate only when exactly one piece is
  selected; it positions itself over the bounding box of the whole
  selection.
- **Box (marquee) selection**: click-drag starting on empty board space
  draws a marquee rectangle (`#marquee-box`, driven by `onMarqueeMove`/
  `onMarqueeEnd`). A piece is selected if its on-screen center falls inside
  the dragged rectangle. Holding Shift at drag-start makes it additive
  (keeps the existing selection and adds to it); without Shift it replaces
  the selection. A plain click (no movement) on empty space still just
  deselects everything, as before.
- **Group transform**: `transformSelected(fn)` rotates/flips an entire
  multi-piece selection together as a rigid group — each piece's
  bounding-box corners are transformed within the group's shared frame
  using the same point-rule that `rotateCells`/`flipCells` use for single
  pieces (verified via a 360°-round-trip self-test). Single-piece selection
  still goes through the original, simpler single-piece path.
- **Trash zone**: dragging a piece onto `#trash-zone` deletes it
  (`isOverTrashZone(clientX, clientY)`). This **replaced** an earlier
  "drag off the edge of the grid to delete" mechanic, which was removed
  because it conflicted with Infinite Grid's edge-expansion gesture (same
  drag motion meant two different, conflicting things).
- **Checkerboard overlay**: `checkerboardEnabled` toggle colors the grid
  with `CHECKERBOARD_COLORS` (currently 2 colors, `['var(--checker-0)',
  'var(--checker-1)']` / labels `['Black','White']`), driven by a generic
  `patternIndexFor(r, c)` function — deliberately written to generalize
  beyond simple 2-color checkering for future, more intricate pattern
  designs (explicit user request). `updateCheckerCounts()` (called from
  `updateHolesCount()`) shows how many black/white squares are currently
  covered by placed pieces.
- **Onion skin overlay**: `onionSkinEnabled` toggle shows a semi-transparent
  checkerboard pattern over the board (`renderCheckerOverlay()`), restricted
  to only the cells actually covered by a placed piece (not the whole
  board — this was a deliberate refinement after an earlier version
  overlaid everywhere). Current opacity is `.checker-overlay-cell { opacity:
  0.5; }` after a couple of rounds of "make it more opaque" tuning (started
  at 0.3, went to 0.375, then to 0.5 at the user's explicit request to go
  "all the way to .5").
- **Selected-piece glow**: `.placed-piece.selected .piece-svg` uses a
  doubled, full-opacity white `drop-shadow` plus a dark drop-shadow for
  depth. This value has been tuned back and forth — see "Selection glow
  history" below before changing it again.
- **Undo**: single global stack (`pushUndo`/`undo`), snapshots
  `placedPieces` + selection + grid bounds as JSON. Called before every
  mutating action.
- **Keyboard**: arrows move, R rotate, T/F flip, D duplicate, Del/Backspace
  delete, Z undo. All operate on the current `selectedIds` (single or
  multi). The global keydown guard is narrowed to only ignore keystrokes
  when `e.target.matches('input[type=text], input[type=number],
  input[type=color]')` — see Known issues for why this matters.
- **Tray controls**: a rotation dial + flip toggle that transform pieces
  *as dragged from the tray* only (`applyTrayTransform`) — never touches
  already-placed pieces. Every tray tile also has its own persistent
  rotate/flip buttons (`saveOverrides`/`applyOverrides` to localStorage).

## Selection glow history (read before touching this again)

The CSS has been adjusted multiple times based on subjective "too
bright/too dim" feedback. Known values, in chronological order:

1. **Original (first shipped version)**:
   `drop-shadow(0 1px 2px rgba(0,0,0,0.4)) drop-shadow(0 0 3px #fff)
   drop-shadow(0 0 3px #fff);` — doubled, 3px blur, full-opacity white.
2. **Dimmed** (after an early "reduce brightness" request): single layer,
   `drop-shadow(0 1px 2px rgba(0,0,0,0.4)) drop-shadow(0 0 3px
   rgba(255,255,255,0.5));` — 3px, 50% opacity. Superseded.
3. **Over-brightened** (after a later "brighter" request): doubled, 4px
   blur, full opacity — `drop-shadow(0 0 4px rgba(255,255,255,1))` ×2. User
   feedback: "dramatically less obvious... go back to the original and then
   turn it up just a little." Reverted.
4. **Current**: `drop-shadow(0 1px 2px rgba(0,0,0,0.4)) drop-shadow(0 0
   3.5px #fff) drop-shadow(0 0 3.5px #fff);` — the original doubled/full-
   opacity look, with only the blur radius nudged from 3px to 3.5px as the
   "turn it up just a little" step. If this still reads as too strong or
   too subtle, adjust the blur radius in small increments (0.5px steps)
   rather than opacity or layer count, since those are what overshot last
   time.

## Known issues / bugs already fixed this project (don't reintroduce these)

- **Pending-placement ordering bug**: every call site that mutates a
  piece's position must call `maybeExpandGrid()` **before**
  `updatePendingPlacement()`. Doing it in the other order was a real,
  shipped bug — a piece that needed grid expansion stayed permanently
  flagged invalid (red) even after the grid grew to fit it, because nothing
  ever re-checked it afterward. Matched directly to user bug report
  ("expansion works sometimes but it's glitchy... it stays red").
- **Persistent highlight bug**: moving a piece could leave stale
  `hl-valid`/`hl-invalid` classes on board cells after the drag ended. Root
  cause: `onDragEnd` called `clearHighlights()` and then called
  `updateDrag()` again afterward (to read the piece's final anchor), and
  `updateDrag()` re-adds highlight classes as a side effect — nothing
  cleared them a second time. Fixed with an explicit final
  `clearHighlights()` call immediately before `drag = null; renderAll();`.
- **Keyboard shortcuts silently stopped working** after interacting with
  any input or checkbox. Root cause: the keydown guard used a blanket
  `e.target.tagName === 'INPUT'` check, and checkboxes/number inputs retain
  DOM focus after being clicked, with nothing to blur them back out. Fixed
  two ways: (1) narrowed the guard to only suppress shortcuts for
  `type=text/number/color` inputs specifically, and (2) added a
  capture-phase `pointerdown` listener that blurs any focused form control
  when the user clicks something that isn't one.
- **Grid-resize Enter-key bug** (self-caught during this project, never
  shipped): the Enter handler for the grid size input used to blur the
  input unconditionally after attempting a resize, including on failure.
  This broke "press Enter again to dismiss the error" because focus was
  already gone before the second press. Fixed to only blur on success.

## Still open — not yet diagnosed

**"The app is running less smoothly" (reported early on, not re-confirmed
since the rendering/selection work below).** Not profiled yet. Likely
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
   `maybeContractGrid` actually changes the bounds, or a manual resize
   happens. On a large grid (close to 20×20) this is hundreds of DOM nodes
   rebuilt, and it can fire on ordinary interactions (an arrow-key nudge
   near an edge) if expand/contract keeps triggering.
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

## Deployment status

Repo: `rmackay0/polyominoes-app` on GitHub, deployed via Vercel with a
custom domain through Namecheap (Vercel auto-deploys on every push to
`main`). `gh auth setup-git` has been run so plain `git push` works
directly (no need to use `gh` for the push itself).

**Standing rule (persistent across sessions): commits are fine to make
without asking, but pushing to GitHub always requires the user to
explicitly ask first, every time** — even if a previous message in the same
session already asked for a push, a *new* push still needs a fresh ask.
Multiple pushes have happened over the life of this project; always check
`git log --oneline origin/main..HEAD` to see what's pending rather than
assuming the remote is stale or current.

## How to test changes (no test suite in the repo)

There's no formal test suite committed. Verification throughout this
project has been done with ad-hoc Playwright scripts in a scratch directory
(not part of the repo, not persisted between sessions) that: launch
headless Chromium, `page.goto('file://.../index.html')`, and drive
drag/keyboard interactions via `page.evaluate` + mouse simulation,
asserting on live page state (`placedPieces`, `pendingPlacementIds`,
`selectedIds`, `gridMinRow`, etc. are all plain global variables, directly
readable via `page.evaluate(() => ...)`). If `node_modules/playwright` is
missing or stale in the scratch dir, reinstall with `npm install
playwright@latest && npx playwright install chromium`.

Recommend setting up similar scripts (or a proper Playwright test file
checked into the repo) before making further changes to the
placement/grid/selection logic — it's easy to introduce ordering bugs like
the ones documented above without them, since a lot of the interesting
behavior only shows up across multi-step interactions (place → rotate →
check red state → move → check it clears → box-select → group-rotate →
undo), not from reading the code alone. A few things that looked like app
bugs during testing turned out to be test-script artifacts instead (CSS
`rgb()` vs `rgba(...,1)` string-matching mismatches, imprecise click
coordinates landing on grid gaps instead of cell centers, a native spinner
button's hit-box being a few pixels narrower than assumed) — always confirm
directly (`document.elementFromPoint`, `document.activeElement`, raw CSS
rule text) before concluding either way.

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
- When a user asks to "turn up/down" an already-tuned value and later says
  it overshot, they tend to want you to recall and restore the exact prior
  value from history rather than just picking a number close to the
  current one — keeping this handoff's "history" sections up to date
  matters more than it might seem.
