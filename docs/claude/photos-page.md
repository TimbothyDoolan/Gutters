# Photos page

<!-- Split out of the root CLAUDE.md, which points here. Read before working on this area. -->

**The camera needs the native side.** "Take photo" opens an in-app `getUserMedia` viewfinder
(`openCamera()`) so a rep can shoot a burst; iOS's `<input capture>` sheet closes after one
frame, and that input is now only the fallback. Three things gate it and all live outside the
HTML: `INFOPLIST_KEY_NSCameraUsageDescription` (without it iOS kills the app the moment the
camera is touched), `requestMediaCapturePermissionFor` in `WebView.swift` (without it WKWebView
auto-denies `getUserMedia`), and `mediaTypesRequiringUserActionForPlayback = []` (without it the
preview stays black). **The simulator has no camera at all** — this path can only be tested on a
real iPad.

**Photo pane layout.** The Photos page is the PDF's photo pages: one `.photo-sheet` per printed
page. Every photo has a fixed slot: `P.meta.photoPages = [{n:1..4, ids:[id|null]}]` (shared),
page order = print order, a null slot prints blank. A drop swaps two slots (`swapPhotoSlots()`)
and nothing else moves — deliberately, the rep asked for no auto-reflow across pages. Unplaced
photos (new, or older saves) are appended at `P.meta.photoLayout` per page (the "All pages"
picker, unset = 4, which also repaginates everything). Always read pages through
`photoPagePlan()` (it normalizes stale ids/empty pages), write via `commitPhotoPages()`.
`photoSlotRects()` gives the slot rects — `renderPhotos()` scales them onto the sheets and
`buildEstimateHTML()` prints them.
Each photo is contain-fit into its slot, as the PDF's `img` is. Reorder is the numbered grip
(`photoDragStart()` → `movePhoto()`), per-photo clear/remove live in its ⋯ menu.
Sheets are one column wide (`photoSheetMetrics()` splits the pane into ~560px columns), so
`#photoScroll` needs `scrollbar-gutter:stable` — `renderPhotos()` measures
after clearing the pane, and without a reserved gutter it sizes columns a scrollbar too wide and
the last one wraps. Marks are stored against a 520px preview (`PHOTO_REF_W`); every renderer
scales by `cw/PHOTO_REF_W` via `photoStrokePx()`/`photoTextPx()`. The left rail belongs to the
Diagram page; `setPage()` parks it on Photos/PDF View and restores the rep's choice on the way
back.

**Photo page zoom** (`photoZoom`, 0.1–16×, session only) is one zoom for all the sheets, not a zoom inside each
photo. The sheets are always ONE column, top to bottom (the rep asked: pages on a table, one below the next — never
regridded into the corner when zoomed out); 100% = one sheet as wide as the pane. `#photoPages` sits in `#photoTable`, padded almost a screen each
way (`photoPanPad`, leaving `PHOTO_PAN_KEEP` px on screen), so the pages are always scrollable and a pan/pinch leaves
them where the rep let go — the rep asked for no snap back to the centre. `renderPhotos()` shifts the scroll by any
change in that padding; Fit (`pzFit`) also lines the sheet back up horizontally. Nothing scales at paint time: marks size off `cw`, so zooming is `renderPhotos()` laying the
sheets out at `photoCardW` = fit width × zoom. Small cards shed chrome (`pc-tight` < 170px, `pc-tiny` < 90px, short
captions). `#photoScroll` is `touch-action:none` and its one-finger pan + coast is JS (`photoPan`/`photoFling` in
`wirePhotoZoom`): with native pan on the gaps, a finger off the photos let iOS start a scroll mid-pinch, cancelling
that finger and scrolling under the preview — a pinch behaved differently depending on where the fingers landed.
A finger on a card's canvas still belongs to the photo tool; a pencil off the photos pans but never pinches.
A pinch (capture-phase pointers on `#photoScroll`, so it can span two photos) or ctrl+wheel previews with a
CSS transform and lays out on release, re-finding the sheet under the fingers (`photoAnchorAt()`) to set the scroll.
A rebuild can land mid-line, so `photoLine` redraws through `photoRedraws`, never a card's closure.
**A card's canvas covers only its on-screen window.** The card div holds the size (cw+2 with its 1px border); the
canvas is absolutely placed at `win` {x,y,w,h} in card px and `redrawPhoto()` paints through a `-win` transform, so a
16× card (~10000×8000) never needs a canvas iOS won't allocate. `card._view(vp)` (from `photoPlaceControls()`, on every
scroll and layout) gives cards ≤ `PHOTO_CANVAS_FULL` px all of themselves, bigger ones what's visible padded half a
screen, and releases (1×1) cards over two screens away. Pointer → image coords (`pos()`) and the text editor's origin
come from the card's rect, never the canvas's. Size against `cw`, never `cv.width`.

**Photo mark schema** (`P.photos[i].strokes[]`, normalized 0..1 image coords): the full table — fields per
type, what `pts` holds for each — is the banner comment at PHOTO MARKUP PAGE in the HTML; read it there.
**`ensurePhotoMarks(ph)`** is the migration hook (called from `absorbPhotoBlobs()` on load and defensively at
the top of `renderPhotos()`): it back-fills `type`/`id`/`opacity` on legacy marks and migrates legacy
prompt()-authored text to the shared fields. After it runs, no reader needs a `||` fallback — don't add new ones.

**One painter for every mark, on screen and in the PDF.** `paintPhotoMark(ctx,st,cw,chh)` is
called from both `drawPhotoMarks()` (the live card) and `photoSnapshot()` (the full-res PDF
flatten) — `cw` is the card's on-screen width or `ph.w` in the PDF; everything else scales off
it. Pen/highlight strokes are smoothed at render time (`paintSmoothPath`, `quadraticCurveTo`
through segment midpoints) — the stored points stay a raw polyline, this is paint-only.
Arrowheads (`paintArrowShaft`) are filled triangles sized from the stroke width. Selection
chrome (bbox handles, vertex dots, the marquee) is NOT in `paintPhotoMark` — it's drawn
separately in `renderPhotos()`'s per-card `redrawPhoto()`, same split as the Diagram's
`drawScene()`/`draw()`.

**Photo tools** live in `photoTool` (`select|pen|highlight|line|rect|ellipse|arrow|polygon|
text|callout|eraser`), each with its own remembered style in `photoToolStyle[tool]` (line/rect/ellipse/polygon share the toolbar's one Shapes button, `#ptShapes` + `#ptShapesMenu`) (or
`photoTextStyle` for text/callout) so switching tools doesn't clobber another tool's width/
opacity. `#photoOptbar` (`buildPhotoOptbar()`/`wirePhotoOptbar()`) is the Photos equivalent of
the Diagram's `buildOptbar()`/`#optbar` — contextual per-tool controls that, with a mark
selected, edit *that mark* live instead of the tool's defaults (`photoOptTarget()` resolves
which). A freshly drawn rect/ellipse/arrow/polygon/callout **stays selected** while its tool
remains active (`photoSelMarks` holds it); a pointerdown on one of its handles
(`photoHandleHit()`) reshapes it, any other pointerdown releases it and starts a new mark. Handle
reshaping is deliberately single-selection only — see the comment at `scaleMarkByHandle()`. The
5-chip palette (`photoPalette()`, persisted in `P.meta.photoPalette`) is shared by every tool's
stroke/fill/text color; long-press or right-click a chip to redefine it.

**The eraser only touches ink.** `photoEraseAt()` (ported from the Diagram's
`eraseStrokesAt()`) splits `pen`/`highlight` strokes into surviving runs and leaves every other
mark type untouched, so scrubbing out a stray scribble can't delete a carefully placed arrow.
Shapes, lines, text and callouts are removed with Select → Delete.
