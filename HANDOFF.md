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
GRID SIZE, TRAY CONTROLS, CUSTOM PIECE EDITOR, HOW TO PLAY, INIT).

## Public "puzzle mode" direction (in progress, started 2026-10-15)

This app is being adapted into a public-facing deployment modeled on NYT
Games — land directly on a specific daily puzzle, no sandbox navigation —
while **keeping the current free-form sandbox** as the internal/admin tool
for building and testing those puzzle configs (not being replaced).
Confirmed decisions so far, all from direct user conversation (none of this
is derivable from the code yet, since most of it isn't built):

- **Grid shape customization** = a rectangular grid with specific cells
  blocked out per puzzle (holes), not a free-form painted outline. Simplest
  data model: board bounds + a list of blocked cells.
- **Win condition** (confirmed 2026-10-15 — **not** classic exact-cover,
  don't assume "every cell must be covered"): a puzzle is solved when (1)
  every piece in that puzzle's given set has been placed on the board, validly
  (no overlaps, nothing off-grid or on a blocked cell, none left in the tray),
  AND (2) every row and every column has an **even number of covered cells**.
  Empty cells are fine — the board does not need to be fully covered. Count
  **individual covered squares**, not pieces: e.g. a U-pentomino placed so
  its top edge covers 2 squares of one row and its bottom edge covers 3
  squares of the row below adds 2 to that row's tally and 3 to the next
  row's — not "1" to each. This is cheap to check live (just tally covered
  cells per row/column after each placement and test evenness) — no solver
  needed for in-app feedback.
- **Solver** (not built yet): an **offline Node script**, never shipped to
  players, that enumerates valid full-piece-set placements for a given
  board+piece-set config and checks which satisfy the row/column-parity win
  condition above — used by the admin to confirm a puzzle is actually
  solvable (and ideally how many solutions it has) before publishing it.
  Deliberately not an in-app feature, to keep the public bundle small and
  the solution out of the page source.
- **Sequencing** (user's choice, given the size of this): How to Play modal
  (done), piece-visual rework (done), grid-shape data model (done), then
  the solver (not started). Mobile optimization is explicitly deferred to a
  later phase.
- **How puzzle mode is entered — resolved 2026-10-15**: **tabs**, not a
  URL param/route. A tab bar at the top of the app lists "Sandbox" plus
  one tab per entry in the `PUZZLES` registry; clicking a puzzle tab swaps
  the shared board/tray UI over to that puzzle's fixed config. See "Puzzle
  tabs" below for the full mechanism. This was built directly off a
  concrete request ("make a tab titled test puzzle #1"), not a separate
  architecture discussion — it's the answer to what used to be an open
  question here.
- Still open / not yet decided: whether this ever needs more than a
  same-page tab bar (e.g. a real URL per puzzle for sharing/bookmarking a
  specific one) if the puzzle list grows large, and whether "Sandbox"
  itself stays visible/accessible in the eventual public deployment or
  only exists for admin use. Don't assume an answer — ask before building
  further on this.

## What's implemented

- **Pieces**: 11 standard (mono/domino/2 trominoes/7 tetrominoes) + 12
  classic pentominoes (fixed shapes, `STANDARD_PIECES` / `PENTOMINO_PIECES`)
  + user-defined custom pieces (`customPieces`, drawn in a modal editor,
  persisted to `localStorage`).
- **Rendering**: every piece — tray preview, drag ghost, and placed piece —
  goes through `buildSolidShape()`, which renders one `<path>` fill + one
  `<path>` outline stroke for the whole piece, rather than per-cell
  bordered divs. This was a deliberate fix for a real bug: per-cell borders
  showed hairline seams on high-DPI displays, and jagged "flag" artifacts at
  concave corners (T/S/Z pieces). The outline is traced via a fine-grid
  rasterization boundary-walk, not per-cell borders. See `buildPieceRects` /
  `buildFillPath` / `buildOutlinePath`.
  **Connected square units** (added 2026-10-15, part of the public
  puzzle-mode direction above): a piece needs to visibly read as joined
  unit squares, not a solid blob, so `buildInternalDividers()` adds a third
  `<path>` — thin stroke-only lines (`rgba(0,0,0,0.25)`, 1.5px) down the
  middle of every swallowed internal gap between two orthogonally-adjacent
  same-piece cells. It's still all vector strokes inside the same SVG, so
  it does **not** reintroduce the old hairline-seam bug — the outer
  silhouette is still the one smooth `buildOutlinePath()` trace, drawn on
  top, completely unaffected by the interior lines. Draw order in
  `buildSolidShape()`: fill → dividers → outer outline.
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
- **Placement rules**: grid-locked (integer cells), but overlap, extending
  past the border, and landing on a blocked cell are all *allowed*. Any
  piece in that state is tracked in `pendingPlacementIds` (a `Set`, supports
  multiple at once) and rendered with a red glow, purely from membership in
  that set — independent of `selectedIds`, so it persists whether or not
  the piece is currently selected. Only **duplicate** auto-searches for an
  open spot (`findOpenAnchor`); rotate/flip/arrow-keys/drag never relocate a
  piece to "help" it fit — they apply literally and let the red state show
  if that broke it. **Critical ordering rule**: `updatePendingPlacement(pieceId)`
  must always be called **after** `maybeExpandGrid()`, never before — doing
  it in the wrong order was a real, previously-shipped bug (see Known
  issues below) and it's an easy mistake to reintroduce at a new call site.
  **No global lock** (removed 2026-10-15 at explicit user request): earlier
  versions blocked clicking/dragging/deselecting anything else while any
  piece was pending (`isLocked()`/`canInteractWith(id)`, with a blocking
  toast). That's gone — a pending piece just sits there red; every other
  piece (new tray drags, selecting, moving, right-click delete, clicking
  empty space to deselect) works completely normally regardless of what
  else is pending. Don't reintroduce a global interaction lock here; if a
  future change needs to restrict interaction with a specific piece, scope
  it to that piece only, not the whole board.
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
- **Group drag**: starting a mouse drag on any piece that's already part of
  a multi-selection (built via shift-click or marquee) moves the whole
  group together, by the same delta, exactly like arrow-key movement does —
  it does **not** collapse the selection down to just the piece you grabbed.
  `startDragFromPlaced` captures each selected piece's starting anchor in
  `drag.groupStart`; `onDragEnd` applies the dragged piece's final delta to
  all of them. Starting a drag on a piece that *isn't* part of a current
  multi-selection still behaves as a plain single-piece drag/select, as
  before. The floating rotate/flip/duplicate/delete toolbar is explicitly
  hidden for the whole duration of any drag (new piece or move) —
  `updateSelToolbar()` bails out early whenever the global `drag` variable
  is set, and both `startDragFromTile`/`startDragFromPlaced` call it
  immediately after setting `drag` so it disappears right away rather than
  waiting for the next `renderAll()` (which doesn't happen again until the
  drag ends).
- **Trash zone**: dragging a piece onto `#trash-zone` deletes it
  (`isOverTrashZone(clientX, clientY)`). This **replaced** an earlier
  "drag off the edge of the grid to delete" mechanic, which was removed
  because it conflicted with Infinite Grid's edge-expansion gesture (same
  drag motion meant two different, conflicting things). Note: dragging one
  piece of a multi-selection to the trash only deletes that one piece, not
  the whole group — intentional, since the UI only ever shows that one
  piece's ghost heading toward the trash.
- **Grid shape (blocked cells)**: `blockedCells` (a `Set` of `"r,c"`
  absolute-coordinate keys, persisted to `localStorage` as
  `poly_blockedCells`) lets specific cells be removed from the board
  entirely — the "variable layouts" part of the public puzzle-mode
  direction. Deliberately **not** called "holes" anywhere in code/UI: that
  word was already taken by `#holes-count` for a different, pre-existing
  concept (an uncovered-but-placeable cell — the same sense the pentomino-
  coloring reference doc uses it in). A blocked cell behaves like
  out-of-bounds: `isValidPlacement` rejects it, and it's excluded from the
  total/open tallies (`setupBoard`'s `total-count`, `updateHolesCount`'s
  `holes-count`) and from the checkerboard scheme's per-color totals
  (`updateCheckerCounts`) — but a piece can still be *dropped* on one, same
  as any other invalid placement (soft-invalid/pending, not a hard
  rejection — consistent with how overlap and off-grid already work).
  Toggled via the "Edit grid shape" sidebar checkbox
  (`editingGridShape`/`#edit-shape-toggle`), which repurposes a plain
  board-cell click (normally the start of a marquee drag) into
  `toggleBlockedCell(r,c)` instead. While that mode is on,
  `body.editing-grid-shape` is set, which does two things via CSS alone —
  no extra JS gating needed in the piece-interaction code: `.placed-piece
  .blk { pointer-events: none; }` so clicks reach the board cell
  underneath even where a piece is currently sitting (you can block/unblock
  a cell regardless of what's on it), and a pointer cursor on every cell.
  Starting a **new** piece drag from the tray while editing is explicitly
  blocked with a toast (`startDragFromTile` checks `editingGridShape`
  first) — editing the board's shape and placing pieces are kept as two
  separate modes, not mixed. Toggling a cell calls `pushUndo()` first and
  revalidates every placed piece afterward (`updatePendingPlacement`) since
  blocking/unblocking a cell can invalidate or un-invalidate whatever's
  sitting on it; `blockedCells` is included in the undo snapshot alongside
  grid bounds. CSS: `.board-cell.blocked` (dashed border, `--bg` fill) plus
  a `::after` pseudo-element showing a red X (`\2715`, `#e05252`) centered
  on the cell — originally a checkmark (`\2713`) per an explicit "red
  checkmarks instead of holes" request, changed to an X one message later
  when the user clarified that was actually what they meant (they'd drawn
  X's in their sketch the whole time). If asked to change this glyph
  again, it's this one `content` value in `.board-cell.blocked::after`.
- **Checkerboard / color scheme overlay**: `checkerboardEnabled` toggle
  colors the grid using one of three schemes in the `COLOR_SCHEMES`
  registry — `checkerboard` ((r+c) mod 2), `stripes2` (r mod 2), `stripes3`
  (r mod 3) — picked via the `#color-scheme-select` dropdown (only enabled
  when the checkerboard toggle is on). Each scheme supplies its own
  `colors`/`labels`/`indexFor`; `patternIndexFor(r,c)` just forwards to
  `currentScheme().indexFor(r,c)`, so rendering, the onion skin overlay and
  the count display all automatically follow whichever scheme is active.
  Persisted to `localStorage` as `poly_colorScheme`. These three were
  chosen out of a ~10-scheme catalog in a coloring-theory reference
  document (see "Checkerboard scheme scope" below) — the rest were
  explicitly descoped as not applicable to this app.
  `updateCheckerCounts()` (called from `updateHolesCount()`) shows two
  things: the whole board's **uncovered/remaining** cells by color (always
  shown whenever checkerboard mode is on), and, only while at least one
  piece is selected, a second "Selected: ..." line showing **covered**
  cells by color for just the selected pieces. These are deliberately
  different statistics (remaining vs. covered) — see that section before
  changing either one.
- **Onion skin overlay**: `onionSkinEnabled` toggle shows a semi-transparent
  checkerboard pattern over the board (`renderCheckerOverlay()`), restricted
  to only the cells actually covered by a placed piece (not the whole
  board — this was a deliberate refinement after an earlier version
  overlaid everywhere). Current opacity is `.checker-overlay-cell { opacity:
  0.5; }` after a couple of rounds of "make it more opaque" tuning (started
  at 0.3, went to 0.375, then to 0.5 at the user's explicit request to go
  "all the way to .5").
- **Puzzle tabs**: a tab bar (`#tab-bar`, built by `renderTabBar()`) lists
  "Sandbox" plus one button per entry in the `PUZZLES` registry (currently
  `test-puzzle-1` and `test-puzzle-2`). Clicking a puzzle tab
  (`switchToPuzzle(key)`) takes over the shared board/tray UI: grid bounds
  set to the puzzle's `rows`/`cols` (always 0-based), `blockedCells`
  replaced with the puzzle's `blocked` list, `placedPieces`/selection/
  pending/undo stack all reset fresh, Infinite Grid forced off
  (in-memory only), and the grid-size input / Infinite Grid / Edit-grid-
  shape controls disabled (a puzzle's shape is fixed while viewing it).
  `#sandbox-controls` (a wrapper div around every sandbox-only sidebar row
  and the Standard/Pentomino/Custom tray sections) is hidden and
  `#section-puzzle` shown instead, populated by `renderPuzzleTiles(key)` —
  just `renderFixedTiles('puzzle-tiles', PUZZLES[key].pieces,
  'puzzle_'+key)`, the same renderer Standard/Pentomino tiles use, so
  drag/rotate/flip-tile-preview all work identically. **Critically, none of
  this touches `localStorage`**: `switchToPuzzle` never calls
  `saveGridBounds()`/`saveBlockedCells()`, and `infiniteGrid`/
  `editingGridShape` are only mutated in memory, specifically so a puzzle
  session can never clobber the sandbox's persisted state. Leaving Sandbox
  calls `captureSandboxSnapshot()` first (deep-copies grid bounds,
  `blockedCells`, `placedPieces`, selection, `infiniteGrid` into
  `sandboxSnapshot`); `switchToSandbox()` restores from it exactly. The
  checkerboard/stripes overlay is deliberately **not** part of any of this
  snapshotting — it's treated as a global display preference that applies
  to whichever board is currently shown, not per-tab state.
  **Puzzle data format**: each `PUZZLES` entry is `{title, rows, cols,
  blocked: [[r,c],...], pieces: [{name,color,cells},...]}` — `cells` are
  relative coordinates exactly like `STANDARD_PIECES`/`PENTOMINO_PIECES`.
  `test-puzzle-1`'s 6 pieces were extracted directly from a user-provided
  sketch image (colors eyeballed/approximated, shapes verified cell-by-
  cell — 4 of the 6 matched an existing `PENTOMINO_PIECES` shape exactly,
  confirming the extraction was accurate; the other 2 were the same
  pentomino type in a different rotation, expected and fine since puzzles
  define their own exact cells rather than referencing the catalog). Its
  blocked cells went through several revisions, documented here in full
  since this took real back-and-forth to get right:
  1. Originally `[[2,3],[3,2]]` (green X's in the first sketch).
  2. Changed 2026-10-15 to `[[1,1],[4,2]]` (red X's in a follow-up sketch,
     green-X cells reopened). **This made the puzzle unsolvable** —
     confirmed by an exhaustive brute-force search (all 6 pieces, every
     rotation/reflection, every non-overlapping placement) that found zero
     arrangements satisfying the row/column-even-parity win condition, for
     *both* this and the original blocked-cell pair. The same piece set
     solves instantly with **zero** blocked cells (24 raw solutions, 3
     distinct up to rotating/flipping the whole board — see
     `scratchpad/pwtest/count_solutions.js`-style scripts from that
     session if this needs re-deriving), so the piece set itself was never
     the problem.
  3. **Current** (2026-10-15, same day): `[[1,1],[1,5],[5,0],[5,5]]`. Found
     by taking one of the 3 zero-blocked solutions (anchors: L (0,0), I
     (0,1), X (1,1), N (2,4), P (4,1), U (1,3) — all cells relative, listed
     in reading order of the board) and blocking 4 of its 6 naturally-
     uncovered cells. This is guaranteed not to break that solution (the
     pieces never needed those cells), and an exhaustive check of **all 15
     ways** to pick 4-of-6 from that solution's empty cells found every
     single one collapses the puzzle to **exactly 1 solution** — so this
     particular choice of 4 wasn't forced, any of the 15 would have worked
     equally well. Verified end-to-end in the app itself (injecting the
     solution's placements and confirming `#congrats-modal` fires), not
     just in the standalone solver. **Difficulty**: with these 4 blocked
     cells, 74 ways exist to place all 6 pieces without overlap; only 1
     satisfies the parity rule — about 1-in-74 odds for a "looks fine"
     placement to actually be right. (An earlier 2-blocked-cell version of
     this same fix, `[[1,1],[5,5]]`, was looser — 2,088 total placements,
     1-in-1,044 — before the user asked to tighten it to 4 blocked cells;
     that number is stale anywhere else it might still appear.) The app
     only evaluates the parity rule once every piece is placed and valid,
     so there's no partial/warmer-colder feedback along the way — discussed
     directly with the user, who chose to keep it this way for now rather
     than add live row/column counters (noted as the natural follow-up
     below). This is a genuinely hard puzzle, not a casual one — expected
     and accepted, not a bug.
  If this ever needs revisiting again: the pattern that worked was picking
  a known zero-blocked solution first, then blocking only cells that
  solution doesn't use — trying to pick blocked cells first and hoping a
  solution exists is what caused step 2's failure. **Possible future
  follow-up, discussed but not built**: live per-row/column covered-cell
  counters while placing pieces (not just the end-state pass/fail check),
  which would shift the puzzle from "mostly trial-and-error, a little
  logical deduction" toward more logical deduction — the user chose to
  keep the current all-or-nothing-at-the-end feedback for now.
- **Test Puzzle #2 (added 2026-10-16)**: a second `PUZZLES` entry
  (`test-puzzle-2`), extracted the same way as #1 — from a user-provided
  screenshot, verified cell-by-cell before shipping (reconstructed the
  drawn arrangement in a standalone script and confirmed: no overlaps, no
  out-of-bounds, 30/30 open cells covered, every row/column even — the
  user had drawn a *complete, pre-solved* layout this time, not just a
  piece reference, and it checked out). Introduced two new, generalized
  `PUZZLES` fields beyond what #1 uses:
  - **`hiddenBlocked: [[r,c],...]`** — a subset of `blocked` that's
    functionally blocked (placement/counts/parity all treat it exactly
    like any other blocked cell — `isBlocked()` is unchanged) but rendered
    with **zero visual difference from a normal open cell**: no dashed
    border, no X, nothing. `isVisiblyBlocked(r,c)` is the new check
    (`isBlocked(r,c) && not in hiddenBlocked`); `setupBoard()`'s per-cell
    className now branches on `isVisiblyBlocked`, not `isBlocked`, for
    both the `.blocked` class and whether to suppress the checkerboard
    pattern class — a hidden cell shows the checkerboard pattern like any
    other open cell would, on purpose, since full indistinguishability is
    the point. The open/total counts (`#holes-count`/`#total-count`)
    deliberately still use the *true* `isBlocked` count, not just the
    visible one — a player who does the arithmetic can infer hidden cells
    exist, just not where; that's an accepted/intended mystery mechanic,
    not an oversight. Test Puzzle #2 blocks 6 cells total, shows 4 (the
    "right side" ones, matching which half of the board they're in — `c`
    index 3,4,5) and hides 2 (the "left side" ones).
  - **`hintOutline: [[r,c],...]` + `hintText: "..."`** — an optional
    static dashed outline traced around an arbitrary region of absolute
    board cells, with a caption. `renderHintOverlay()` (called from
    `setupBoard()`, so it refreshes on cell-size/grid changes same as the
    checker overlay) reuses `buildOutlinePath`/`SVG_NS` — the exact same
    boundary-walk geometry already used for piece outlines — fed the hint
    region instead of a piece's cells, so it traces one smooth connected
    boundary (not per-cell borders) regardless of the region's shape. Lives
    in `#hint-overlay`, a new layer in `#board-outer` stacked above
    `#pieces-layer`/`#checker-overlay` (z-index 500) so it's never hidden
    by a placed piece. For #2, the outlined region is the union of the X
    and T pieces' cells in the solution (10 cells, one connected shape
    since the two pieces' cells happen to touch), captioned "The outlined
    cells can only be covered by two of the six pieces." — a logical
    anchor (narrows the search meaningfully) without fully revealing the
    solution for that region.
  - `test-puzzle-1` leaves both fields undefined; `renderHintOverlay()` and
    the `isVisiblyBlocked` hidden-cell check both no-op cleanly when a
    puzzle doesn't define them (checked via `PUZZLES[activeTab]` directly,
    not a separate per-tab flag), so neither feature needed any
    puzzle‑#1‑specific changes.
- **Puzzle progress (timer + win check)**: live-checked inside `renderAll()`
  whenever `activeTab !== 'sandbox'` (`checkPuzzleProgress()`), so it
  re-evaluates after every discrete mutation (place/move/rotate/delete/
  undo) without needing calls scattered at each site. Implements the win
  condition exactly: returns early (no highlight, no popup) unless
  `placedPieces.length === puzzle.pieces.length`, `pendingPlacementIds` is
  empty, and there's exactly one placed piece per puzzle piece name (guards
  against e.g. Duplicate creating a second copy of one type — that's
  correctly treated as "not done," not crashed on). Once genuinely
  complete, tallies covered cells per row/column: all even → **solved**
  (stops the timer via `stopPuzzleTimer()`, freezes `#puzzle-timer`'s text
  at the exact elapsed time, shows `#congrats-modal` with that same time
  string — computed once and reused for both, not read twice, to avoid a
  tick of drift between them); any odd → adds `.parity-odd` to every cell
  in the offending row(s)/column(s) (`box-shadow: inset`, reuses
  `--invalid-border` red rather than a new color). `puzzleSolved` latches
  true on solve so the popup doesn't re-fire on later interaction (e.g.
  clicking to select the solved pieces) — reset to `false` by
  `startPuzzleTimer()` on each fresh puzzle-tab entry.
  **Timer**: `startPuzzleTimer()`/`stopPuzzleTimer()`, `#puzzle-timer`
  (shown only while a puzzle tab is active, mirrors `#section-puzzle`'s
  visibility toggling in `switchToPuzzle`/`switchToSandbox`), updates every
  500ms via `setInterval`, `formatElapsed(ms)` → `"M:SS"`. Resets fresh on
  every puzzle-tab entry (consistent with the tab already resetting
  `placedPieces` etc. fresh each time — see "Puzzle tabs" above).
  **Tray consumption**: `renderPuzzleTiles()` grays out
  (`.tile.consumed { opacity:0.3; pointer-events:none; }`) any tile whose
  name already has an instance in `placedPieces`, and un-grays it the
  moment that piece is deleted (recomputed fresh every call, not tracked
  as separate state) — this is what makes "every piece placed" legible to
  the player as the tray visibly draining, not just an invisible rule.
- **Selected-piece glow**: `.placed-piece.selected .piece-svg` uses a
  single white `drop-shadow` (2px blur, 75% opacity) plus a dark
  drop-shadow for depth. This value has been tuned back and forth — see
  "Selection glow history" below before changing it again.
- **Undo**: single global stack (`pushUndo`/`undo`), snapshots
  `placedPieces` + selection + grid bounds + `blockedCells` as JSON. Called
  before every mutating action. Cleared (not persisted across) on every
  puzzle-tab switch — see "Puzzle tabs" below.
- **How to Play modal**: `#how-to-play-btn` (top-right of the topbar) opens
  `#how-to-play-modal`, reusing the existing `.modal-backdrop`/`.modal`
  pattern from the custom piece editor. Explains the actual win condition
  (see "Public puzzle mode direction" above) including the "count squares,
  not pieces" U-pentomino example. Auto-opens once on a visitor's first
  load (`openHowToPlay()` called at the end of INIT, gated on
  `localStorage.poly_hideHowToPlay !== 'true'`); a "Don't show this again"
  checkbox in the modal sets that flag on close. No backdrop-click-to-close
  or Escape-to-close — kept consistent with the editor modal's existing
  explicit-button-only pattern, not a new convention.
- **Keyboard**: arrows move, R rotate, T/F flip, D duplicate, Del/Backspace
  delete, Z undo, Escape deselects all. All operate on the current
  `selectedIds` (single or multi). Escape is deselect rather than D because
  D was already taken by Duplicate (flagged and confirmed with the user
  rather than silently overloading D). The global keydown guard is narrowed
  to only ignore keystrokes when `e.target.matches('input[type=text],
  input[type=number], input[type=color]')` — see Known issues for why this
  matters.
- **Tray controls**: a rotation dial + flip toggle that transform pieces
  *as dragged from the tray* only (`applyTrayTransform`) — never touches
  already-placed pieces. Every tray tile also has its own persistent
  rotate/flip buttons (`saveOverrides`/`applyOverrides` to localStorage).

## Checkerboard scheme scope (read before adding more colorings)

A document titled "Grid Colorings for Pentomino Puzzles" (shared 2026-10-01)
catalogs roughly 10 periodic colorings for pentomino-tiling math — checker,
row/column stripes in 2 and 3 colors, several `(x+y) mod k` diagonals,
four-color parity, 2×2-block checker, sparse dots — plus a validator/
solver/printed-piece-generator spec built around them, written for a
puzzle-*solving* app. This app has no solver, no exact-tiling concept, and
no "does this board admit a tiling" question — it's a free-form drag-and-
drop board. Asked explicitly which parts applied here, the user confirmed:
**only the top 3 recommended-for-building-first schemes** (checkerboard,
2-color stripes, 3-color stripes), as plain overlay/paint options, with
none of the validator/solver/printed-piece machinery. That's what
`COLOR_SCHEMES` implements. If asked to add more schemes from that document
later, the remaining ~7 (and their mirror variants) are still undone by
design, not by oversight — re-confirm scope before adding them, since nothing
in this app's architecture currently needs the multi-color-mix math the
document's later sections describe (reachable-sum validators, signature
vectors per piece, etc.) — none of that has a clear use here without a
solver to attach it to.

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
4. **3.5px doubled/full-opacity retry**: `drop-shadow(0 1px 2px
   rgba(0,0,0,0.4)) drop-shadow(0 0 3.5px #fff) drop-shadow(0 0 3.5px
   #fff);`. This was committed and pushed, but a local editor autosave
   reverted the working file back to the 4px version (#3) before the user
   actually saw it in a browser — they reported "still far too dramatic,
   looks unchanged," which was literally true. **Lesson**: after editing
   this CSS, re-grep the file immediately before telling the user to look,
   since this file is being actively edited in an IDE and can get reverted
   out from under a change.
5. **True original restored**: `drop-shadow(0 1px 2px rgba(0,0,0,0.4))
   drop-shadow(0 0 3px #fff) drop-shadow(0 0 3px #fff);` (exact value from
   #1). User still found this "very bright and noticeable" and asked for
   something "very subtle" — i.e. the doubled/full-opacity white glow was
   never actually subtle; the user's mental model of "the original" as
   dim didn't match its real CSS value.
6. **Subtle rewrite**: dropped the doubled-layer/full-opacity approach
   entirely in favor of a single white shadow layer with reduced opacity —
   `drop-shadow(0 1px 2px rgba(0,0,0,0.4)) drop-shadow(0 0 2px
   rgba(255,255,255,0.6));` (2px blur, 60% opacity). Confirmed subtle
   enough by the user.
7. **Nudged up twice** from that subtle baseline, blur and opacity tuned
   independently: 2px/0.6 → 3px/0.75 → **current: 2px/0.75** (blur brought
   back down while keeping the higher opacity). **Current (final, pushed)
   value**:
   `drop-shadow(0 1px 2px rgba(0,0,0,0.4)) drop-shadow(0 0 2px
   rgba(255,255,255,0.75));`
   If asked to adjust again, tune blur and opacity as independent, small
   (±0.25–0.5 opacity, ±1px blur) increments from this single-layer base —
   do not go back to the doubled-layer/full-opacity style from #1/#3/#4,
   since that was confirmed too strong across multiple rounds.

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
