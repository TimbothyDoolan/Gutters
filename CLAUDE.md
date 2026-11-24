# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What's here

Two unrelated codebases in one repo:

| Path | What it is |
|---|---|
| `PrimeGutterEstimator/PrimeGutterEstimator/gutter-estimator.html` | **The actual product.** A ~5,000-line single-file offline app (one `<style>`, one `<script>`, no build step, no dependencies). |
| `PrimeGutterEstimator/PrimeGutterEstimator/*.swift` + `PrimeGutterEstimator.xcodeproj` | A thin SwiftUI/`WKWebView` shell that bundles that HTML as an iPad app. ~3 small files. |
| `concepts/` | A React/Vite gallery of 5 throwaway UI/UX concept demos built against `concepts/prd.md`. Exploration only — **not** the shipping app, and it shares no code with it. |

Nearly all real work happens in `gutter-estimator.html`. Treat the Swift shell as
plumbing and `concepts/` as a sketchpad unless asked otherwise.

## Environment constraints

**There is no `node`, `npm`, `deno` or `bun` on this machine.** This shapes everything:

- `concepts/`'s scripts (`npm run dev`, `npm test` — which uses `node --test`) **cannot be run
  here.** Don't propose them as a verification step; reason about that code by reading it.
- To execute the estimator's JS, use the JavaScriptCore CLI that ships with macOS.

### Testing `gutter-estimator.html` headlessly

The whole app is one `<script>` that runs DOM setup on load, so it can't be `require`d. Extract
it and run it under `jsc` with a DOM shim (`document`, `localStorage`, `indexedDB`, timers,
`getComputedStyle`, and a permissive `Proxy` for `getElementById`/canvas `ctx` — ~70 lines):

```bash
python3 -c "import re; open('app.js','w').write(re.findall(r'<script>([\s\S]*?)</script>', open('gutter-estimator.html').read())[0])"
/System/Library/Frameworks/JavaScriptCore.framework/Versions/A/Helpers/jsc shim.js app.js test.js
```

`init()` will throw partway through DOM setup — that's expected and harmless; every function is
still defined and callable. You can then drive the real engine: set `P.objects`, call
`compute()`, `materialList()`, `runCorners()`, `resizeRunSegment()`, `draw()`,
`buildEstimatePane()`. Assertions are plain scripts. **Re-extract `app.js` after every edit** —
a stale `app.js` silently tests the old code.

For visual checks: `python3 -m http.server` from the HTML's directory, then open in Chrome
(`file://` URLs are blocked from `chrome://newtab`).

### Building the iPad app

Run from the repo root (the `.xcodeproj` lives there, sources live one level down):

```bash
xcodebuild -project PrimeGutterEstimator.xcodeproj -scheme PrimeGutterEstimator \
  -destination 'id=<simulator-udid>' -configuration Debug \
  -derivedDataPath <scratch>/dd build
```

The project uses Xcode 16 `PBXFileSystemSynchronizedRootGroup` (objectVersion 77): **files in
`PrimeGutterEstimator/PrimeGutterEstimator/` are picked up automatically.** The bundled
`README.md` predates this and still describes manually dragging files in and fixing "Target
Membership" — that advice no longer applies. The filename `gutter-estimator.html` is load-bearing
(`WebView.swift` looks it up by name via `Bundle.main.url`).

## Architecture of the estimator

Navigate the file by its banner comments (`/* ==== STATE ==== */`, `CAMERA / CANVAS`,
`GEOMETRY HELPERS`, `TAKEOFF / COMPUTE`, `RENDERING`, `TOOLS + DOCK`, `HIT TESTING`,
`POINTER EVENTS`, `LEFT RAIL`, `PHOTO MARKUP PAGE`, `SAVE / LOAD / NEW`, `PRINT`, …).
Subsystem notes split out of this file live in `docs/claude/`; each section below says when to read which.

**Global mutable state, no framework.** `P` is the live project (`P.meta`, `P.settings`,
`P.objects`, `P.photos`). Mutate it directly, then call `refreshAll()` (redraw + rebuild the
visible rail pane) or `draw()` for canvas-only changes. `recordHistory()` *before* a mutation
snapshots for undo.

**Everything drawn is one flat `P.objects` array** discriminated by `o.t`: `gutter`, `ds`
(downspout), `valley`, `splash`, `custom`, `demo`, `screen`, `text`, `arrow`, `stamp`, `hl`, `underlay` (a plan image to trace over —
see **Plan underlay**). A `ds` may carry `memo` (free-text note like "next to brick", its own chip and
`memoOff`), kept apart from `notation` so `parseDS` never sees words. Every label chip is draggable through
`segChip()` + a label-box kind, its offset stored on the object (`noteOff`, `memoOff`, `sizeOff`, `tagOff`,
`existOff` for an existing run's EXISTING tag, `segLabelOff[i]`); double-tap snaps one back.
Line-ish objects carry `pts` (world coords) plus a parallel `lengths` array — one entry per
*segment*, so `lengths.length === pts.length - 1`. Keep them in sync on any geometry edit.
A `text` box (`{t:"text",id,x,y,w,h,text,fill,color,font,size,stroke,bold,italic,border}`) has
`w:null` for auto-width (sized to the longest line) or a number for a fixed, word-wrapped width;
`h` is always recomputed by `drawText()` on every draw, never trust a stale one. See **TEXT BOX**
below — it's shared with the Photos page.

**Text-box selection is two taps, not one.** With the Select tool, the first tap on a text box
selects it (handles + the style strip); a second tap on the box that's already the sole selection
opens the editor. `EDITABLE` (HIT TESTING) deliberately excludes `"text"` for this reason — it's
special-cased in `onDown`/`onUp` instead of the generic single-tap-opens-editor path that
`seglen`/`dsnote`/`stamp` still use.

**`compute()` is the single source of truth for the takeoff.** It walks `P.objects` and returns
quantities (`q`), amounts, and totals; `materialList()` calls it and re-derives the same
quantities as human-readable BOM rows. **Any pricing rule change must be made in both** or the
worksheet and the printed estimate disagree. Two deliberate pricing-only exceptions:
`settings.newConstruction` zeroes end caps/outlets (and their flat-mode LF allowances) in `compute()`
but `materialList()` still lists them; `settings.priceOverride` (>0) replaces the turnkey (`r.overridden`,
`r.calcTurnkey`), deposit/card fee follow it, and `pdfPriceRows()` then prints only deposit + card fee. Acute corners
(≤ 90−`bayDeviation`, inside or outside) bill the style's strip miter (never bay) **plus** `prices.acute`
(hand-edited add-on, $60 default, charged in flat mode too); `materialList()` splits miter rows
"— inside"/"— outside" and adds one style-less "Acute miter" row. Gutter bills whole feet: every *gutter*
segment rounds up (`segFt()`/`runFt()`), at read time — stored lengths keep their measured precision, so read
gutter footage through those helpers, never a raw `lengths` sum.

**Products are data: the catalog / price book** (PRODUCT CATALOG banner, Products page; `CATALOG.products`,
device-wide). `applyCatalog()` refills `GUTTER_STYLES`, `DS_STYLES`, `SCREEN_TYPES`, `PRICE_KEYS`, `DERIVED_KEYS`,
`LINES` and `BOOK` **in place** — they're consts other code holds; never reassign them, and call `catalogChanged()`
after any catalog edit. Every product's job price is `settings.prices[priceKey]`, copied from the book the first time
`ensurePrices()` meets it and then kept. **Before touching products, pricing, product dock buttons, highlight
erasers or Convert style, read `docs/claude/catalog-pricing.md`.**

**Jobs** (JOBS banner) live in IndexedDB as well as the `localStorage` autosave. `curJobId` is serialized as
top-level `jobId`, deliberately not in `P.meta` (undo swaps meta), and `idbGC()` must never delete a blob a listed
job names. **Before touching jobs, save/load or IndexedDB, read `docs/claude/jobs-storage.md`.**

**Dialogs** are the app's own: `uiConfirm(msg,opts,yes)`, `uiAlert(msg)`, `uiPrompt(msg,opts,done)` (DIALOGS,
`#dlgWrap`, a layer above the modal). They're callbacks, not blocking — don't add native `confirm()`/`alert()`/
`prompt()`. Long how-it-works text goes in `infoNote(key,title,html)` (a folded `<details>`; `infoOpen`
remembers which are open across pane rebuilds), not a bare `.note`.

**Options are estimate variants.** `OPTIONS[]` each own their own `settings`, `objects` and
undo `history`; `meta` (customer), `photos` and stamp presets are *shared*. `captureActive()` /
`activate(i)` swap the live `P` between them.

**Coordinates:** world units with `GRID = 20`; `cam {x, y, scale}`; `worldToScreen` /
`screenToWorld`. Job-content line weights, dashes and miter marks go through `wpx(v)` (= `v*cam.scale`,
its look at 100%), so they stay proportional to the drawing on the canvas at any zoom and in the
PDF. Gutter runs and what rides on them (casing, existing/custom runs, miter marks, chip/arrow clearances) use
`gpx(v)` instead — `wpx` × `settings.lineWeight`, the ⚙ line-thickness slider. Selection chrome and handles stay fixed screen px. Don't add a bare px `lineWidth` to job content. Feet are derived, not stored — `P.settings.feetPerSquare` (or calibration)
converts via `feetToWorld` / `autoLen`.

**Selection.** `hitTest()` picks the closest thing, by distance past its painted edge (a tie goes to the narrower
target, then the topmost), never just the topmost within tolerance. Text chips under 8px tall don't catch taps, and
markers shrunk by the zoom reach out to a minimum target (not `dsfront`: a stray tap flips the gutter). `hit.loose`
= in tolerance but off the ink: a tap selects it, a drag starts a box instead of moving it, or nothing could be
boxed on a dense plan. Touch needs 10px of travel to count as a drag. `selectInMarquee()` tests what's painted
(`labelBoxes`, arrow lines, a stroke's band), never `objBounds()` boxes, and a tap with no travel isn't a box.

**Arrows** on both pages are `paintArrowShaft()`: a solid arrow is one filled outline, so nothing can poke past
the tip and part opacity shows no overlap; a dashed shaft is a stroke stopping at the head's base.

**Run connectivity is geometric, not stored.** Endpoints within `CAP_EPS` (11 world units) count
as joined. `isClosedRun()`, `freeEnds()`, `runCorners()` and `junctionCorners()` all re-derive
this from the points every time. Don't cache a "closed"/"joined" flag on an object — extending,
splitting and dragging all invalidate it, and a stale flag silently drops end caps or invents
miters. Known limitation: only *endpoint-to-endpoint* joins are recognized; a run ending on
another run's interior vertex counts as neither a miter nor a junction.

**Front side → inside/outside miters.** `gutter.front` (+1 = left of the run's point order, −1 =
right, world y-down; unset = auto) is the only stored piece; `runGroups()` re-derives which runs
share it from the joins, and `runFronts()` resolves every run (a rep flip wins, else `autoFront()`:
outside of the group's shoelace area, else away from the drawing's centroid, else page-bottom).
Every corner from `runCorners()`/`junctionCorners()` carries `inside` (turn toward the front) and
`acute`. `flipFront(run)` rewrites the whole group so explicit values never disagree. Each
downspout's front arrow (`dsFrontOf()`, nearest segment within 2 GRID) prints; the I/O letters
on miter marks (`drawCornerSides()`) are screen-only. Segment length chips default to the run's *back* (`segBackOffset()`,
re-derived from the front every draw); `segLabelOff` (a rep's drag, measured from the midpoint) wins, and
`flipFront()` mirrors those across their segment so they stay on the back.

**Photo blobs never live in `P`.** `P.photos` and `P.meta.titleImage` hold only ids; base64 sits
in the `photoBlobs` Map, mirrored to IndexedDB. This is deliberate — blobs in `P` made undo
snapshots ~27 MB and blew WKWebView's localStorage quota silently. Exported "Save" JSON *does*
embed images so backups stay portable (`serialize(true)`). Every photo enters through
`commitPhotos()` and is capped at `PHOTO_MAX_EDGE` (2000px) on the way in.

**Photos page** (PHOTO MARKUP PAGE banner): photo slots are fixed (no auto-reflow across pages), one painter
(`paintPhotoMark`) serves the card and the PDF, card canvases cover only their on-screen window, the eraser only
touches ink, and the camera needs native-side support (no simulator camera). **Before touching anything on the
Photos page — layout, zoom, marks, tools, text/callouts, camera — read `docs/claude/photos-page.md`.**

**Native bridge + signatures.** `geNative(name)` returns a `webkit.messageHandlers` handler or `null`; handlers
are registered in `WebView.swift` and **both sides must change together**. The native PDF overlay sits above ALL
web content, so anything modal on PDF View must `hide` it first. **Before touching the native bridge, PDF
import/export, PDF View, signatures or page 1's legend/summary, read `docs/claude/pdf-native.md`.**

**The PDF diagram is vector.** `diagramSVG()` replays `drawScene()` into `makeSvgCtx()`, a recorder
that turns the 2D-context calls RENDERING uses into SVG. Put job content in `drawScene()` (it prints)
and editor chrome in `draw()` (it doesn't); a new `ctx` method in RENDERING must also be added to
the recorder or it silently won't print. `makeSvgCtx().fillText` parses `ctx.font` as
`"[italic ][weight ]<size>px <family>"` — both the style and weight tokens are optional, in that
order — and emits `font-style="italic"` when present; miss either token and bold/italic text
prints plain.

**TEXT BOX (near RENDERING) is one engine for two pages.** `textStyleFont(o,px)`,
`layoutTextBox(ctx,o,px,maxWidthPx)`, `paintTextBox(ctx,o,x,y,px,maxWidthPx)` and
`applyTextSizeChange(o,newSize)` are shared by the Diagram's `{t:"text"}` (`drawText()`) and a
photo's `{type:"text"}` mark (`paintPhotoMark()`'s text branch, `paintPhotoText()`). Only the
caller's coordinate conversion differs: the Diagram passes world-px scaled by `cam.scale`, a
photo passes normalized 0..1 coords scaled by the card's `cw`/`chh`. `maxWidthPx===null` means
auto-width (sized to content, never wrapped); a number means fixed-width (word-wrapped, padding
proportional to `px`). `paintTextBox` never draws selection chrome — the Diagram replays it
through `makeSvgCtx()` for the PDF, so a baked-in handle would print. A photo callout is a text
mark with an added `tip` point; `paintPhotoText()` draws its leader line before the plate so the
plate sits on top. The floating editor is `openEditorFor()`'s text branch (Diagram) and
`openPhotoTextEditor()` (Photos) — both drive the same `#floatEdit` element (a child of `#stage`,
which `#photoStage` also sits inside), just computing its screen rect differently; open it
**synchronously inside the pointer-up** that created or tapped the box, or iOS won't raise the
keyboard.

**Plan underlay + extra pages** (DOCUMENT IMPORT banner). Measure only through `fpsAt()`/`segFps()` (`autoLen`,
`pathFeetRaw`, `feetToWorld(ft,at)`) — never a bare `drawingFps()`, and never copy a plan's scale into settings.
Any new blob-backed field must go in `extraBlobIds()` or `idbGC()` deletes it at startup. **Before touching plans,
Plan mode, scales, the trace loupe, tap-draw or extra pages, read `docs/claude/plans.md`.**

**Profile vs settings vs pricing.** The rep's name/phone/e-mail are device-wide (`localStorage["primeGutterProfile"]`),
not in the job. The ⚙ Settings modal holds profile, takeoff rules and drawing settings, written to every option; the
Pricing tab holds money only. Don't put a setting in both places.

## Conventions

- **Touch sizes live in one `@media (pointer:coarse)` block at the end of the `<style>`** (44pt targets for
  tabs, top-bar buttons, the undo/zoom rows, worksheet inputs); the mouse layout stays dense. The dock is 62px
  wide for its labels, and 42px-tall tools under `max-height:880px` so 13 fit an 11" iPad in landscape.
- **iPad touch hygiene lives in CSS near the top of the `<style>`:** `user-select:none` +
  `-webkit-touch-callout:none` on `body` (inputs/textareas opt back in, or iOS won't let you type),
  transparent tap highlight, `touch-action:manipulation`, and every `:hover` rule wrapped in
  `@media (hover:hover)` — iPadOS leaves `:hover` stuck on the last thing tapped. New hover styles go
  inside that media query too.

- The estimator must stay **offline and single-file** — no CDN links, no external assets, no
  build step. Inline anything new.
- Comments in this file explain *why*, usually citing the bug the code prevents. Match that when
  touching non-obvious logic; keep the existing terse, dense style rather than reformatting.
- Domain vocabulary is load-bearing: *miter* (corner) is strip or bay by angle, and every gutter style has its own of each (no box miters, no shared miters), *end cap* only
  on free ends, *run* is one polyline of gutter, *outlet/elbow/offset* belong to downspouts.
