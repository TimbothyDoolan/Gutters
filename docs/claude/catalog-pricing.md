# Product catalog, price book and Convert style

<!-- Split out of the root CLAUDE.md, which points here. Read before working on this area. -->

**Products are data: the catalog / price book** (PRODUCT CATALOG banner, Products page). Every
quotable product — gutters, downspouts, screens, services, accessories, custom runs, the shared
valley/acute parts — is one entry in `CATALOG.products`, device-wide in
`localStorage["primeGutterCatalog"]` (built-ins in `BUILTIN_PRODUCTS` are merged in on every load, can be
hidden, never deleted). `kind` decides behaviour and components (`COMPONENTS`: a gutter gets end caps +
its own strip + bay miters (`stripKey`/`bayKey`, e.g. `strip6hr`/`bay6hr`), a downspout gets outlet/elbows/offsets; each is priced "follow" — parent $/ft ×
the Settings allowance, as always — or "fixed"). `applyCatalog()` refills `GUTTER_STYLES`, `DS_STYLES`,
`SCREEN_TYPES`, `PRICE_KEYS`, `DERIVED_KEYS`, `LINES` and `BOOK` **in place** — they're consts other code
holds; never reassign them, and call `catalogChanged()` after any catalog edit. Pickers go through
`pickable()` so hidden products drop out while jobs using them still draw and price. A drawn product
(`isToolProduct()`) gets a dock button unless `dock:false` (`onDock()`/`setOnDock()`): it stays in the catalog
and the dock's ＋ (`openQuickProduct()`) lists it to put back; unlike `hidden`, it's still drawable
(`toolDefById()` resolves `custom:<id>` for it, and `customTools()` keeps the active one's button).
Every dock tool has a `lbl` (shown under its icon). `PAGES_TOOL` (PDF pages) has no dock button: PDF View's
"Diagram pages" starts it and it's appended to the dock only while active. `setTool()` commits a run in
progress with 2+ points (`settleDrawing()`) instead of discarding it. The page tabs are the job flow (Diagram →
Photos → PDF View); Products is reached from ⚙ Settings or Pricing and returns via `prodReturn`. The option
tabs are hidden on Photos and Products.
Every highlight-style tool (Demo, Screen, Highlighter, a Highlight-behaviour product: `isHighlightTool()`) has an
Eraser toggle in its option bar, kept per dock button in `eraseFor`; `eraseMatch()` limits it to that tool's own
strokes (a product's eraser only touches that product), and `eraseStrokesAt()` splits them so footage drops. The catalog is a
**price book**: a job copies a key's book price into `settings.prices` the first time `ensurePrices()` meets
it and keeps it; `applyPriceBook()` is the explicit push, `applyBookKeys(keys)` the per-item one (a product
plus its fixed-price parts via `productBookKeys()`; the Products editor and each Pricing row that differs show a ⇣).
Compare a job to the book through `keyDrifts()` — a style's book list is `bookList()`, lifted to its min as
`ensurePrices()` does, or the job never "matches". Every product's job price — demo, screens, custom
products too — is `settings.prices[priceKey]`; flat/hourly service quantities are `settings.qty[priceKey]`.
`migrateLegacyPricing()` (inside `ensurePrices`) converts old saves' `demoRate`/`screen*Rate` and per-job
`customProducts`; `normalizeProduct()` upgrades older product shapes (the retired one-"own miter" gutter). Saves embed the non-built-in products they use (`jobProducts()` → `adoptProducts()` on
load, added hidden), so a job opens on an iPad whose catalog never had them.

**Product formulas** (`p.formulas = [{id,per,every,add,qty}]`, "every N ft/corners/… of this, add M of that";
helpers after `newProductId()`). They're book data that a job **freezes** like a price: `ensurePrices()` copies a
product's list into `settings.formulas[productId]` the first time it meets the product (an empty list too, so a
formula added later never reaches a job already quoted), and `bookKeyInto()` pushes the book's copy with the
product's own key, so `applyPriceBook()` and the per-item ⇣ carry formulas, and `keyDrifts()` reports a formula
drift on that key. `compute()` counts each formula per run (gutter run, downspout, custom run, highlight stroke;
junction miters join their run), rounding the "every N" up within the run; placed items and typed services pool
over the job. One level only: added quantities never fire the target's own formulas. Targets are
`formulaCanAdd()` (item / line / service / screen, never a gutter or downspout) and land in the ordinary custom,
service or screen rows, so `materialList()` lists them with no code of its own. A job formula naming a product
counts in `productUsage()` (embedded in saves, can't be deleted); deleting a product strips book formulas that add it.

**Convert style** (`openConvertSystem()` → `convertGutterStyle()`, Select bar) only changes sizes, since caps,
miters and footage all derive from them. Selection = whole runs, existing runs are never touched. Converting a
whole option also moves its `gutterType`/`defaultDownspoutSize`, pins skipped existing items to their old size so
they don't follow, and rewrites the job details box's gutter line.
