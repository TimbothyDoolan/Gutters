# Plan underlay, scales and extra pages

<!-- Split out of the root CLAUDE.md, which points here. Read before working on this area. -->

**Plan underlay + extra pages** (DOCUMENT IMPORT banner). Both store their images in the
photo blob store by id, like photos. `{t:"underlay",img,x,y,w,h,opacity,print,hidden,pageIn,scale,check,lastFps}`
is one per **sheet** — several plan pages can share one canvas (`plansOf()`; `planOf()` is the sheet Plan
mode is editing, `planCur`) — per option; it's skipped by the takeoff, by marquee, and by hit testing — it's only edited in
**Plan mode** (top bar's Plan button → `enterPlanMode()`/`exitPlanMode()`, `#planPanel`, which routes
canvas pointers to `planDown`/`planMoveEvt`/`planUp` and hides the dock/optbar via `body.plan-mode`).
The scale is an object on the plan: `scale` = `{mode:"line",a,b,ft}` (a dimension line, ends as
plan-local `{u,v}` fractions of the plan's width, so they ride with the sheet) or
`{mode:"ratio",ftPerIn,label}` (needs `pageIn`, the PDF page size, sent by `gutterPdfImport`).
**`fpsAt(w)` is the scale reader**: the scaled sheet under the point (hidden sheets included), else
`drawingFps()` — the first scaled sheet (`planFps()`) or `settings.feetPerSquare`. Never copy a plan's scale
into settings. **Each sheet keeps its own scale**: a segment measures at the sheet under its midpoint
(`segFps()`), so the same drawn length reads differently on a 1/8" and a 1/4" sheet. `autoLen`, `pathFeetRaw`
and `feetToWorld(ft,at)` all go through it — don't measure with a bare `drawingFps()`. Moving a sheet carries
what's traced on it (`overSheet()`: bounds center on the sheet) and the PDF frames over it; PDF pages' "Page per
sheet" frames each one. Plan points never snap to
the grid. Stored run lengths don't follow a scale change, so `commitPlanScale()` re-measures segments whose
scale moved (`segFps(a,b,[sheet,o.lastFps])` vs now) that still equal their old auto length, and keeps hand-typed ones. `normalizePlanWorld()` resizes
the sheets (while nothing else is drawn) to 2 ft/square on Done (camera-compensated) so symbols aren't out of proportion. `print:false`
draws from `draw()` and `objBounds()` returns null for it, so it neither prints nor stretches the PDF
fit. `print:true` draws inside `drawScene()`, which is why the recorder has `drawImage`. Scale/check
lines and the loupe are `draw()` chrome only. The same loupe (`drawPlanLoupe`) magnifies under the finger while the
Gutter tool draws or a gutter corner is dragged (`drawTraceLoupe()`, gated by `tracePress`/`vertexDrag` +
`settings.gutterLoupe` — off for plain drawings, on for plans: a plan import switches it on, removing the
last plan switches it off, and `applyLoaded()` sets it from whether a job has a plan when the save predates
the key; renamed from `planLoupe`, whose old default of true was saved into jobs). Over a
plan it re-draws the plan image; elsewhere it copies the canvas as painted so far, so it's called late in `draw()`. Finger-riding chrome
(the length chip, loupes) stays inside `canvasClearRect()` — the canvas minus dock, options bar, Plan panel. `settings.tapDraw`
(point-by-point drawing, touch only; a new plan turns it on) keeps a touch run open across lifts — each lift
commits a corner, ✓ Finish run (`drawBar`) or closing on its own start ends it — via `drawing.tap`/`tapPress`. `P.meta.extraPages` (shared, carried over by **New**) print as full
pages just before the customer info page. `serialize(true)` embeds both under `blobs`, and
`extraBlobIds()` keeps `idbGC()` from deleting them at startup. Add any new blob-backed field there.
