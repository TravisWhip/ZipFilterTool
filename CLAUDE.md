# ZipFilterTool / Stitch & Sort

Static site hosted on **GitHub Pages** from `main` (repo: https://github.com/TravisWhip/ZipFilterTool,
site: https://traviswhip.github.io/ZipFilterTool/). Pushing to `main` publishes, so only push when asked.
No build step, no dependencies, no server: every tool is a single self-contained HTML file.

| Path | What it is |
|------|-----------|
| `index.html` | Landing page "Stitch & Sort — ZIP Filter" (ZIP filtering tool). Has a `.tool-link` card linking to the JEF editor. |
| `viewer.html` | Companion viewer for the ZIP tool. |
| `mortgage-calculator.html` | 3D mortgage calculator. |
| `examples/` | Sample ZIPs for the ZIP tool. |
| `jef-editor/` | Janome `.JEF` embroidery tools (merged in with `git subtree` from a separate local repo, history kept). |

## Working conventions
- Commit only when the user asks; push only when asked. End commit messages with the co-author line Claude Code supplies.
- The user often says just "commit" — then commit everything relevant with a descriptive message.
- `jef-editor/` is the live copy. The old standalone folder `C:\Users\travi\OneDrive\Desktop\jef_file` is outdated
  (it lacks the hoop-photo feature); don't edit it.
- Platform: Windows, Git Bash + PowerShell. Node is available; Python is not. Headless Edge is at
  `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe`.

## JEF editor (`jef-editor/jef-editor.html`, ~5300 lines, vanilla JS + canvas)
Also `date.JEF` (sample: 15 runs, 1692 stitches), `README.md`, `TODO.txt`. (The old `jef-viewer.html` was removed; the editor
covers it.) Local-only, not committed: `SavedDesigns/`, `HoopImage.jpeg`, `SQ14b.webp`, `DirectoryFileViewer.drawio`.

### JEF format (little-endian)
- Header: stitch offset @0x00, flags @0x04, ASCII date @0x08 (14 bytes), thread count @0x18,
  record count @0x1C (= stitch bytes/2 + 1), hoop code @0x20 (0=110×110, 1=50×50, 2=140×200, 3=126×110, 4=200×200),
  extents @0x24 (left=-minX, top=maxY, right=maxX, bottom=-minY, in 0.1 mm), hoop extents 0x34–0x73 (-1),
  thread palette indexes @0x74, then thread types.
- Stitches: 2-byte signed dx/dy in 0.1 mm, **+Y is up**. `0x80` starts a 4-byte command: `80 02` jump/trim,
  `80 01` color change (machine stops), `80 10` end. Moves are limited to ±127, so long moves are split.
- Optional trailing "ENHANCE" block with thread RGB + names (rgb at rec+4, name at rec+12, stride 44).
- **JEF+** (`.JEF+`, Janome's newer software; local sample `jef-editor/SavedDesigns/3inchPatchTemplate.JEF+`, not committed): same header/stitches, but between
  the thread types and the stitches sits a thread block: u32 count, then per thread 44 bytes = u32 number (0..n-1), r g b a,
  u32 (unknown, usually -1), 32-byte name. Palette indexes go past 30 and the hoop code can be non-standard (48 in the sample).
  `parseJEF` keeps everything between types and stitches as `meta.pre` (`meta.plus` = it's a JEF+ block, `readPlus`);
  `buildJEF` writes `pre` back unchanged, or rebuilds it with `buildPlus` when threads changed (reusing each unchanged
  thread's original record `t.plusRec`). The file keeps its `.JEF+` extension (`jefExt()`). An unedited JEF+ round-trips
  byte-identically.
- A color change to the **same** thread = a pause (used for stops and appliqué).
- Saving an unedited file must be byte-identical. The hoop code is only rewritten when `meta.isNew || meta.hoopTouched`.

### Data model
- `cmds`: `[{t:'s'|'j'|'c', x, y, g?(group), sid?(source id), note?}]` in 0.1 mm; `threads`; `meta`.
- `shapes`: live shapes `{id, kind(rect|oval|line|tri), cx, cy, w, h, rot, style(satin|running|triple), thick, dens, len,
  under, lock, color, corner, stop, applique, appPlace, after, group}`.
- `compose()` interleaves shapes into the stitch runs (by `shape.after`) → `{all, th, marks}`; everything draws from it.
- A **stitch run** = block of stitches between jumps. `applyOrder(order)` rebuilds `cmds` from run objects.
- Copy/paste (`copyShapes`/`pasteShapes`, also Duplicate): the clipboard keeps each run with its thread (`runObjects()`), and paste
  appends the copies to the order and calls `applyOrder`, so a multi-color copy keeps its colors (color changes added as needed).
- `runSources[sid] = {def, stage, xf}` keeps shape settings editable after "To stitches" (regenerated with `fitAffine`).
- Undo/redo: JSON snapshots of `{cmds, threads, threadsDirty, shapes, measures, runSources}` via `snapshot()`/`restore()`.
- View: `view = {s, ox, oy}`; `toS(x,y)` design→screen, `toW(sx,sy)` screen→design. `changed()` refreshes panels + `draw()`.
- CSS gotcha: `[hidden]{display:none !important}` is required because `display:flex` rules override `hidden`.

### Features
Shape drawing with satin/running/triple stitches (rounded satin corners with sharp↔round slider, compressed inner side);
draw.io-style handles, snapping and guides (edges, centers, stitched inner/outer edges, group centers); inch/mm grid with
rulers and opacity; metric/imperial units (internally always 0.1 mm); hoop overlay with standard Janome hoops;
realistic thread preview with fabric tightness/pull simulation and a 100% ghost; editable stitch runs; stops/pauses;
KSP-style **Sew order** panel (thumbnails, drag reorder, pauses, horizontal groups, resizable, edge tabs to reopen);
groups (right-click, Ctrl+G); "Center on contents" with a 3-level spotlight; measure tool (M); sew-out player with
stops; appliqué (placement → pause → tack-down → pause → satin); points tool for single stitches.

**Shift while moving:** Shift+drag (or pressing Shift mid-drag) locks the move to the axis moved further along (`drag.lock`,
orange dashed axis line; snapping/guides only on the free axis). Shift+click still toggles selection: an unselected element
is added on pointerdown, a selected one is removed on pointerup only if it wasn't dragged (`drag.unselect`).

**Align (TinkerCad style):** toolbar `#alignBtn` / key A, needs 2+ elements selected (an element = a whole group, or one
ungrouped shape / stitch run; `alignEls`, keys from `itemKey`; boxes = stitched outer edges). The reference (`alignRef`,
default the biggest by area) stays put and is drawn in orange "Stays put"; clicking another selected element makes it the
reference. `alignDots` = a row under the selection (left/center/right) and a column left of it (bottom/middle/top) at the
reference's edges; hover (`alignHover`) previews, click = `alignTo(axis, at)` (one undo step; mode stays on). Esc, A,
clicking empty canvas, changing tool or tab leaves it. The side panel's align buttons call the same `alignTo`.
Right-click menu "Align to this" (`alignFromMenu`): `showMenu(x, y, anchor)` gets the right-clicked element's key (canvas or
Sew order card) as `menuAnchor`, which becomes `alignRef`. Right-clicking a selected element keeps the selection.
`selTarget()` returns null in align mode (no handles); `drawAlign` runs last in `draw()`, above the rulers.

**Text (embroidery fonts, `.bx`):** toolbar Text / key X. `.bx` = Embrilliance BX font (format reverse-engineered from
`SavedDesigns/fonts/`, not committed): header "BX001" with `installName` (font name) and the character list, then a bzip2 stream
whose "BZh9" is replaced (find the block magic `1AY&SY`; decompressed by our own `bunzip2`). Per glyph: `u32 2, u32 2, u32 n`,
n×4 zero bytes, n×4 flag bytes (byte 0 bit 1 = jump to this point), n int16 dx then n int16 dy (planar!) in ¼ of 0.1 mm, +Y up,
centered on the glyph box; then records incl. "outline" (6 floats; [1] = baseline y) and, after the "index" record and the
path record, the character string (uppercase stored as e.g. "AU": take the first code point). Duplicate characters (author
mistakes) are resolved by file name (`BX_NAMES`). `parseBX` → `{ name, glyphs: Map(ch → {st:[[x,y,jump]], w, top, bot}), capH,
lineH, space }`; `layoutText(font, text, ls%, lh%, align)`. Fonts live in `FONTS` (by name) and IndexedDB `kv/fonts`
(`idbKV`; stored in the background — headless Edge on file:// never answers IndexedDB). A text is a shape `kind:"text"`
(style "text") with `text, font, ls, lh, align` plus its laid-out stitches `tst` at natural size `nw × nh` (so it sews/saves
without the font); `w/h` scale them (`textStitches`, jump flags → `appendShape` emits j + s). `regenText` keeps the scale, the
top and the aligned side; `setTextSize` sets the capital height. The Properties panel shows the font's own size (`f.capH`, row `#txDefRow`, “Use” resets to it)
and a font list `#txFonts` (stitched “Abc” samples from `fontSample`; click = `chooseFont`). **Font preview:** Shift+point at a font in
that list or a `.bx` file in the Files panel (`previewFontFile`, parsed fonts cached in `fxFonts`, not loaded) → `previewFont(font)`
lays the selected texts out in it without an undo step (`fontPrev` keeps their JSON); pointing away, Shift up, any other key,
any pointerdown or window blur → `endFontPreview()` restores them. Corner handles keep proportions. "To stitches" makes one
group of runs (no runSources). Loading: Open…, drop (`useFontFiles(files, at)`), Files panel tile (`.bx` listed with a "font"
badge and an "Abc" thumbnail), or the Text tool asks (`pendingText`).

**Hoop photo:** View → Hoop photo → Upload (or drop an image on the canvas). Align mode
shows a pink ghost hoop; the photo is placed by 4 corners `photo.q` (perspective warp via `squareToQuad` + triangle
mesh `drawImageTri`, cached in `photo.warp`). Stored in `localStorage["jefHoopPhoto"]` (downscaled JPEG), never in the JEF.
Key functions: `loadPhoto`, `startAlign`/`endAlign`, `photoXform`, `flattenPhoto`, `drawGhostHoop`, `drawPhoto`.

**Project files (`.jefproj`):** JSON `{format:"jef-editor-project", version:1, fileName, meta (header/trailer as base64),
cmds, threads, threadsDirty, shapes, measures, runSources, nextId, nextMeasId, hoop, photo}`. `buildProject`/`loadProject`;
**Save** (Ctrl+S) writes it, **Open…**/drop reads `.jefproj` or `.JEF`, **Export JEF** = `buildJEF`. Export only marks the
design saved when there's no project-only data (`projectOnly()`). A JEF → project → reopen → export must stay byte-identical.

**Photo alignment modes:** "Mark hoop edge" (drag a dotted box onto the hoop's inside edge, Apply = `homog(markBox, hoopInnerQuad())`
applied to `photo.q`) or "Move photo" (manual). Hoops can carry real frame sizes (`iw/ih/band/rad`, SQ14 = 171.5 mm inside).

**Files panel (left):** `FX` state. Several folders from anywhere on the PC (`FX.roots`: `{path, handle | upload, tree, lost, scanning}`),
added with **Add folder…** or right-clicking the empty space (`fxAddFolder` → `showDirectoryPicker`, or the `webkitdirectory`
upload fallback), always at the top level; right-click an added folder → “Remove folder from list” (`fxRemoveRoot`, files untouched).
`fxRebuild()` puts them under an invisible root `FX.tree` (`virtual: true`) so filters/counts/`fxFileByPath` span all of them;
top-level paths are unique (`fxRootPath`: "Designs (2)") and `fxRootOf(node)` finds a node's root (its handle is used for saving).
Handles are kept in IndexedDB `jefEditor/kv/folders` (migrated from the old single `folder` key); a root the browser needs
permission for again shows as a stub row ("Click to reconnect" → `fxScanRoot(r, true)`). Nested folders are CSS-grid columns with spanning rows (`renderFiles`); thumbnails via
`fileStitches` + `drawThumb`, lazy with IntersectionObserver. Lists `.JEF`, `.jefproj` and `.bx` fonts; filter bar `#fxFilter` (All / Projects / JEF incl. JEF+ / Fonts, with counts; `FX.filter`, `FX_KINDS`, localStorage `jefFilesFilter`) renders `fxFiltered(FX.tree)` — a copy whose folders keep `orig` (the real node, used for the save-here menu); folders stay even when empty, and with a filter on, a folder with nothing matching (incl. subfolders) folds to one row (`none(n)`: "No fonts here (3 other files)", not clickable, right-click still saves). Layouts (`FX.mode`, `FX_MODES`, localStorage `jefFilesLayout`): horizontal / rows / small (CSS-only variants of the grid) and tree (`fxTreeList`: IntelliJ-style 24 px rows, folders first, 20 px thumbnail + size via `th.short`, hover → `#fxPeek` big preview beside the panel). Thumbnails are always drawn at 124 px and scaled by `fxPaint`; tiles and rows share `fxWire` (click, Shift+hover, drag).

**Tabs (bottom of canvas):** active design lives in the globals; others are parked as `docs[i].d` (`captureDoc`/`applyDoc`).
`openInTab` (file list, Open…, OS drop) reuses an untouched new tab (`blankDoc`) or switches to the tab already showing that
file. Dragging a tile onto the canvas = `insertDesign(fileStitches(file), dropPoint)`: appended to the sew order as one group,
jump first, color change only if the thread differs; undoable. `bounds()` always includes the origin, so don't use it for
centering.

**Panels:** Files (left), Sew order, canvas + tabs (`#center`), Properties (`aside#side`, right). Each side panel has a drag
handle (`#fileResize`, `#orderResize`, `#sideResize`; double-click resets; width in localStorage) and a slim vertical edge
tab (`#fileTab`, `#orderTab`, `#sideTab`) that is always shown and toggles the panel (`edgeTab()` sets its label/arrow; the
left tabs are rotated 180°, so their arrow character is flipped). There are no separate collapse buttons.

**Preview tab:** Shift+hover a file tile (or press Shift over one) → `queuePreview` (70 ms debounce) → `showPreview` loads it
into the single `docs[i].preview` tab. It reads the file first (`readFor`) and then calls `load(file, data)`, which needs
no I/O wait, so nothing is painted between clearing and loading (otherwise a blank canvas flashes between files). `dropPreview()` (called by switchTab/newTab/closeTab) removes it once it isn't
active; `keepPreview()` turns it into a normal tab on edit (`syncTab` sees `!saved`), rename, or clicking its file.

**Names:** the tab shows `fileName`; double-click to rename (`startRename` → `setName`, cleaned by `cleanName`). New designs get
`defaultName()` (new-design, new-design-2, …) and `untitled = true`; Save / Export JEF / Save PNG call `ensureNamed()` first
(popup: name it, keep the default, or cancel). A loaded file exports as name-edited.JEF unless `renamed`.
The popup is the generic `askName(o)` (also used by Save as; supports a live `warn(name)` that relabels OK to "Replace").

**Save as into a folder:** right-click a folder (or its file row) in the Files panel → `#fxMenu` → `saveToFolder(node, "proj"|"jef")`.
Folder nodes from `fxScanHandle` carry `handle`; write access is requested on the root handle at save time. Warns in the
dialog if the name exists in the scanned list, and re-checks on disk (confirm) in case the list is stale. Empty subfolders
are kept (as "right-click to save here" rows). Upload-fallback folders have no handle → items disabled. Each root is rescanned on its own after a save.

**Fit:** `fit(mode)` with "hoop" (whole frame + any design outside it) or "design" (true bbox). Toolbar switch `#fitHoop`
picks the Fit button's mode (`jefFitMode`); side panel "On open" `#openFit` (`jefOpenFit`, default hoop) is used by `load`/`loadProject`.

### Testing
- Node simulations with a stubbed DOM: extract the `<script>` from the HTML, `eval` it with fake `document`/canvas/
  `localStorage`, expose internals through a `globalThis` hook, then drive `pointerdown/move/up` and `keydown`.
  Assert results (PASS/FAIL lines). Run from a folder containing `date.JEF` when tests load it.
- Visual checks: inject a test `<script>` into a copy of the page and screenshot with headless Edge:
  `msedge --headless=new --disable-gpu --user-data-dir=<tmp> --window-size=1400,900 --virtual-time-budget=3000 --screenshot=<png> file:///<html>`
- Verify the unedited `date.JEF` round-trips byte-identically after changes to save/parse.

### Open to-dos
- Possibly a toolbar button for the hoop photo (the user had trouble finding it in the side panel).
- The editor has no link back to the landing page.
