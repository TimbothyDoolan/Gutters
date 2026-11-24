# Native bridge, signatures, PDF legend

<!-- Split out of the root CLAUDE.md, which points here. Read before working on this area. -->

**Native bridge.** `geNative(name)` returns a `webkit.messageHandlers` handler or `null`, so the
same code runs in a plain browser and in the app. `gutterPdfImport` ({id,data:base64,maxEdge}) →
PDFKit renders each page to JPEG and calls back `pdfImportPage()`/`pdfImportDone()`/`pdfImportFailed()`
(DOCUMENT IMPORT banner); a browser has no PDF rasterizer, so there only images import. `gutterSave` → iOS share sheet; `gutterPdf`
(`build`/`frame`/`hide`/`show`/`share`/`print`) → native pagination into a real PDF shown in a
`PDFView` laid over the PDF View tab's `#pdfHost`, with Share/Print sending that same PDF data. The
overlay sits above ALL web content, so anything modal on that tab must `hide` it first. Handlers
are registered in `WebView.swift`; both sides must change together.

**Signatures** (`P.meta.signatures`, shared across options; see the SIGNATURES banner) are vector
ink signed on a white pad in `openSignModal()`. `build` carries `sigs` (each with a web-rendered
`sigCardPNG()`), and native stamps them on as `SignatureAnnotation`s the rep can press-and-hold to
drag, even across pages. Native reports every landing spot back through `pdfSigMoved(id,page,x,y)`
(PDF points, top-left origin). Custom annotations aren't serialized, so Share/Print use
`exportData()`, which redraws the pages flattened. Signatures stay out of undo: `restore()` carries
them across the `P.meta` swap. The card (`SIG_W`×`SIG_H`, 170×44pt) is sized to fit the band under
page 1's footer. In the native PDF that footer ends at ~735pt of 792 on every job, because the diagram
box absorbs page 1's slack. New signatures land there (`SIG_FOOT_Y`). Changing page 1's layout means
re-measuring that band with the simulator harness.

**Profile, settings, legend.** The rep's name/phone/e-mail live in `localStorage["primeGutterProfile"]`,
not the job. `applyProfile()` fills only blank salesperson fields. `repLine()` builds "Prepared by"
for both `buildEstimateHTML` and `pdfBoxFor`'s height model. The ⚙ Settings modal holds profile,
takeoff rules and drawing settings, written to every option. The Pricing tab holds money only. Don't
put a setting in both places. The legend (`legendItems()`/`legendHTML()`) is an HTML overlay, both on
the stage (`#diagLegend`) and in the PDF's `.dg` box, so it costs no `pdfBoxFor` height. Page 1's Job summary
cards show footage + the quoted style(s) on one nowrap line — it must not grow past `PDFL.summary`. Page 1's
legend placement is per option (`P.pres.legend` {x,y,s}: fractions of the box's free space + scale,
`legendPlaceStyle()`). The native PDFView can't host draggable HTML, so PDF View's ◳ Legend opens
`openLegendArranger()`, which replays the diagram box with that same CSS. Text boxes
scale from their corners (size and width together) and set width from their sides. The job details
quiz writes a normal `{t:"text",quiz:true}` box.
