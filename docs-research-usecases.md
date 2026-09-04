# Peaklab Labels — job map, competitive gaps, and a build order

Research date: 2026-09-04. Product: https://kizaialt.github.io/label-sheet-printer/
Source read: `C:\Users\kizai\Claude\label-sheet-printer\index.html` (read-only; not edited).

> **Note on a moving target.** The file changed while this research was underway. My first read
> (1853 lines) had no barcode and no price field. My second read (1912 lines) had both:
> a hand-written Code 128 encoder (`code128()` / `barcodeSVG()`), a per-cell `c.bc` flag, a
> per-cell `c.p` price field, and a `money()` formatter that adds thousands separators and `₮`.
> Everything below is written against the **1912-line state**. This matters a lot for
> prioritisation — see §4.1, because the two biggest wins on this list just got cheaper.

---

## 1. What the tool is today (architecture, stated precisely)

Facts that constrain every recommendation below.

**State shape** (`freshState()`, ~line 934):

```
S = {
  presetId, 
  g: {pw, ph, cols, rows, mt, mr, mb, ml, gx, gy, rad},   // ONE page geometry
  cells: [ ... cols*rows entries ... ],                    // ONE sheet, flat array
  images: { id: dataURL },                                 // logos, in localStorage
  style:  {fs, ff, b, up, lay, is, fit},                   // sheet-wide default style
  custom: [ ...user size presets... ],
  opts:   {guides, idx, printMarkers, theme, ox, oy}
}
```

**Cell shape**: `{ t, p, bc, img, m, fs, ff, b, up, lay, is, fit }`
— `t` text (multi-line), `p` price, `bc` barcode-on flag, `img` image id, `m` colour tag
(`"blue"` or `"blue.g"`), rest are per-cell style overrides.

**Render**: `cellHTML()` builds a `.cell` containing `.inner` → `.txt` which holds
`.nm` (the text) + `.pr` (price, `font-size:2em; font-weight:800`), then `.bc` (Code 128 SVG,
28% of label height), plus an `<img>` positioned by `lay` (top/bottom/left/right).
`autofit()` binary-searches the largest point size at which the whole `.txt` block fits, 10
iterations, cached by a content+geometry key.

**Print**: `@page{size: Wmm Hmm; margin:0}` injected into a `<style>`, then `window.print()`.
No PDF path, no rasterisation.

**Sizes**: 9 built-ins, all A4 — 32×15 (102-up), 38×21 (65), 48×25 (40), 63×38 (21),
70×37 (24), 99×68 (8), 105×42 (14), 105×149 (4), full sheet (1). Plus user presets.

**What it can do**: rectangular/ctrl/shift selection, row+column header select, filter chips
(all/none/invert/blank/filled/tagged), apply text to selection, auto-number with
prefix/suffix/zero-pad, "fill from list" (one line per label), grid-shaped tab-separated paste,
logo drop with 4 arrangements + size slider, autofit type, 10 colour tags × solid/gradient with
paint mode and hover spotlight, copy/cut/paste blocks, 50-deep undo, printer offset in 0.1 mm,
cut lines, export/import JSON, localStorage autosave, light/dark, Mongolian UI.

**What it structurally cannot do** (these are the gaps, stated as facts about the code):

| # | Limitation | Where it comes from |
|---|---|---|
| L1 | **One sheet, ever.** `count()` is `cols*rows`; `cells` is one flat array; one `.sheet` div. | state shape |
| L2 | **The list importer fills only `t`.** `applyList` does `c.t = line`. It cannot fill `p`, `bc`, or `img`. There is no column→field mapping anywhere. | line ~1556 |
| L3 | **No QR.** Code 128 only, and Code 128 subset B rejects Cyrillic (`tBadBarcode` toast). | `code128()` |
| L4 | **No in-cell editing.** Every edit is select → type in sidebar → click Apply. No `dblclick` handler exists. | grep: zero hits |
| L5 | **Two type levels max, and the second one must be a price.** `.nm` + `.pr`. A label cannot have "big name / tiny ingredients". | `cellHTML` |
| L6 | **No per-cell alignment, padding, border, or rotation.** No `transform:rotate` on cells. | CSS |
| L7 | **No date handling.** No date picker, no "+N days", no date format. | grep: zero hits |
| L8 | **No saved designs.** Export/import is a whole-sheet JSON blob; there is no named, reusable label *design* separate from the sheet contents. | export handler |
| L9 | **No copies-per-row.** Sequential numbering and list fill are 1:1 with labels. You cannot say "3 of each". | `applySeq`, `applyList` |
| L10 | **No text wrapping control or truncation warning** beyond autofit shrinking to 3 pt minimum. | `autofit()` |

---

## 2. PART 1 — The job map

**35 distinct jobs across 9 sectors.** For each: fields that appear on the label, the size that
suits it on A4, and what the tool would need that it does not have. Gap codes refer to §1.

### 2.1 Retail shop (5 jobs)

| # | Job | Fields on the label | Size | Missing |
|---|---|---|---|---|
| 1 | **Shelf-edge price tag** | Product name, price (₮), unit price ("/кг"), sometimes barcode | 48×25, 63×38 | Column import (L2), copies-per-row (L9), multi-sheet (L1) |
| 2 | **Price sticker on the item** | Price only, or price + SKU | 32×15, 38×21 | Column import (L2) |
| 3 | **SKU / stock label** | SKU code + Code 128 barcode + short name | 32×15, 38×21 | *Barcode now exists.* Needs column import (L2), multi-sheet (L1) |
| 4 | **Promo / discount sticker** | "-30%", "ХЯМДРАЛ", old price struck through, new price | 32×15, 48×25 | Strikethrough / two-price layout (L5), background colour fill (L6) |
| 5 | **Clothing size & variant tag** | Brand, size (XL), colour, price, barcode | 38×21, 48×25 | Column import (L2), 3 type levels (L5) |

### 2.2 Café, bakery, food producer (5 jobs)

| # | Job | Fields | Size | Missing |
|---|---|---|---|---|
| 6 | **Café / bakery item label** | Product name, price, sometimes allergen icon | 48×25, 63×38 | Column import (L2); ok otherwise |
| 7 | **Ingredient + allergen label** (deli, catering, cottage food) | Product name, full ingredient list in small type, **allergens in bold**, net weight, producer name + address | 63×38, 99×68 | **Three type levels in one label (L5)** — this is the blocker. Also bold-within-text, left alignment (L6) |
| 8 | **Date marking / best-before** | "Үйлдвэрлэсэн: 04.09.2026", "Дуусах: 11.09.2026", batch code, "нээсэн огноо" | 32×15, 38×21 | **Date fields with today + N days (L7)**, batch auto-number exists |
| 9 | **Jar / bottle product label** | Brand logo, product name, net weight, ingredients, contact, QR to Facebook page | 63×38, 99×68 | QR (L3), three type levels (L5), circular/oval shape (L6) |
| 10 | **Takeaway order seal / item label** | Order no, customer name, phone, items, time | 48×25, 70×37 | Column import (L2), date/time (L7) |

Regulatory grounding: cottage-food and packaged-food rules everywhere converge on the same six
fields — product name, ingredients, net weight, producer name+address, allergen declaration,
date. In Mongolia the reference is **MNS CAC 1:2007** for packaged-food labelling, and label
information for imported food must be in Mongolian (or EN/RU). All six fields on one 63×38 mm
sticker is a *typography* problem, not a data problem — which is exactly gap L5.

### 2.3 Pharmacy and clinic (2 jobs)

| # | Job | Fields | Size | Missing |
|---|---|---|---|---|
| 11 | **Dispensing label** | Patient name, drug name + strength, quantity, directions for use, date, Rx/serial no, pharmacy name + address + phone, cautionary text | 63×38, 70×37 | Three type levels (L5), date (L7), **a saved reusable design with fixed boilerplate** (L8) |
| 12 | **Repack / compounded medicine** | Drug, strength, batch, expiry, "Хүүхдээс хол байлга" | 32×15, 48×25 | Date + expiry (L7), copies-per-row (L9) |

Note: the pharmacy job is the strongest case for **saved designs (L8)** — the pharmacy's name,
address and phone are identical on every label forever, and retyping them is absurd.

### 2.4 Online sellers / parcel senders (3 jobs)

| # | Job | Fields | Size | Missing |
|---|---|---|---|---|
| 13 | **Parcel address label** | Recipient name, phone, district/khoroo/building address, order no, sender block, sometimes tracking barcode | 105×149 (4-up), 99×68, **210×148.5 (2-up) — not in the size list** | **Multi-sheet (L1)** — you ship 60 parcels, not 4. Column import (L2). Left alignment (L6). New size preset |
| 14 | **Thank-you / brand sticker** | Logo, "Баярлалаа!", Facebook/Instagram handle, QR to page | 48×25, 63×38 round | **QR (L3)**, round shape (L6) |
| 15 | **Care / return-instruction sticker** | Short paragraph of instructions, QR to a returns form | 63×38, 99×68 | QR (L3), left-aligned body text (L6) |

### 2.5 Warehouse / stockroom (3 jobs)

| # | Job | Fields | Size | Missing |
|---|---|---|---|---|
| 16 | **Bin / shelf location label** | Location code in large human-readable type (`A-03-02-C-01`) **above** a Code 128 barcode encoding the same string, zone colour | 63×38, 99×68 | **Multi-sheet (L1)** — a small warehouse is 200–2000 bins. Sequential *multi-level* numbering, i.e. aisle × bay × level (L9 generalised). Barcode exists |
| 17 | **Carton / pallet label** | SKU, description, quantity, date, PO number, barcode | 99×68, 105×149 | Column import (L2), multi-sheet (L1), three type levels (L5) |
| 18 | **Goods-received / QC label** | Date received, supplier, inspector initials, pass/fail | 32×15, 48×25 | Date (L7) |

Industry convention worth encoding as a preset: zone-aisle-bay-level-bin, zero-padded, Code 128,
human-readable text mirroring the barcode exactly, and one colour per zone — the tool's colour
tags map onto this beautifully but currently do not print by default.

### 2.6 Office and administration (5 jobs)

| # | Job | Fields | Size | Missing |
|---|---|---|---|---|
| 19 | **IT / equipment asset tag** | Asset number, "Property of <company>", QR or barcode encoding the asset number, sometimes department | 32×15, 38×21 | **QR (L3)**, fixed boilerplate line (L8), sequential + barcode together (works today only if barcode text == label text) |
| 20 | **File / binder spine label** | Title, date range, department, sometimes colour band | 105×42, and a **long narrow custom size** | **90° rotation (L6)** — a spine label is read sideways. Nothing else in the tool does this |
| 21 | **Visitor / meeting name badge** | Name (large), organisation, role, date, event logo | 99×68, 105×42 | Column import from a guest list (L2), three type levels (L5), multi-sheet (L1) |
| 22 | **Envelope / mailing address label** | Name, organisation, street, district, city, postcode | 99×68, 105×42, 63×38 | Column import with a **multi-field address block** (L2 + L5), multi-sheet (L1) |
| 23 | **Archive box label** | Contents description, date range, box number, destroy-by date | 105×42, 105×149 | Date arithmetic for destroy-by (L7), sequential box numbers (exists) |

### 2.7 Schools (3 jobs)

| # | Job | Fields | Size | Missing |
|---|---|---|---|---|
| 24 | **Pupil name labels** (books, equipment) | Pupil name, class, sometimes school logo | 32×15, 38×21 | **Copies-per-row (L9)** — 30 pupils × 8 labels each = 240 labels → **multi-sheet (L1)**. Column import (L2) |
| 25 | **Library / textbook spine + accession label** | Call number, accession number, barcode | 32×15 and narrow custom | Rotation (L6), multi-sheet (L1) |
| 26 | **Reward / certificate stickers** | Short praise text, star image, pupil name | 32×15 round, 48×25 | Round shape (L6), image library (low value, see §4.4) |

### 2.8 Workshops and trades (4 jobs)

| # | Job | Fields | Size | Missing |
|---|---|---|---|---|
| 27 | **Cable / port tag** | From → To, circuit id, often printed twice (wrap-around flag) | 32×15, custom narrow | **Rotation (L6)**, "print each label twice mirrored" (a wrap tag) — niche, see §4.4 |
| 28 | **Tool / equipment inventory tag** | Tool id, owner, calibration due date, QR | 32×15, 38×21 | QR (L3), date (L7) |
| 29 | **Service / maintenance sticker** | "Last service: dd.mm.yyyy", "Next due: dd.mm.yyyy", technician, phone | 38×21, 48×25 | **Date + N-months arithmetic (L7)** — this job is *entirely* a date job |
| 30 | **Electrical panel / breaker label** | Circuit number, what it feeds, amperage | 32×15 | Column import (L2). Otherwise served today |

### 2.9 Events and services (5 jobs)

| # | Job | Fields | Size | Missing |
|---|---|---|---|---|
| 31 | **Event name badge from a guest list** | Name (large), company, table/seat, QR for check-in | 99×68, 105×42 | Column import (L2), QR (L3), multi-sheet (L1) |
| 32 | **Raffle / cloakroom / entry ticket** | Sequential number, event name, date, barcode of the same number | 32×15, 48×25 | Barcode from a *different* field than the label text (L2/L5), multi-sheet (L1) |
| 33 | **Table numbers / place cards / favour labels** | Guest name, table number, date, monogram | 63×38, 99×68 | Column import (L2) |
| 34 | **Repair-shop / salon job ticket** | Job no, customer name, phone, item, date in, date promised | 63×38, 70×37 | Column import (L2), date (L7), three type levels (L5) |
| 35 | **Marketing QR sticker** (menu QR, wifi QR, review QR, Facebook page) | Call to action in Mongolian, big QR, logo, handle | 48×25, 63×38, 99×68 | **QR (L3)** — the whole job is the QR |

### 2.10 Frequency count of each gap across the 35 jobs

| Gap | Jobs blocked or crippled | Count |
|---|---|---|
| **L2 — column→field import** | 1,2,3,5,6,10,13,17,21,22,24,30,31,33,34 | **15** |
| **L1 — multi-sheet** | 1,3,13,16,17,21,22,24,25,31,32 | **11** |
| **L5 — three type levels / rich label body** | 5,7,9,11,17,21,22,34 | **8** |
| **L3 — QR** | 9,14,15,19,28,31,35 | **7** |
| **L7 — dates** | 8,10,12,18,23,28,29,34 | **8** |
| **L6 — alignment / rotation / shape / fill** | 4,7,9,13,14,15,20,25,26,27 | **10** (but split across four unrelated sub-features) |
| **L9 — copies per row** | 1,12,24 | **3** (but 24 alone is the entire school market) |
| **L8 — saved designs** | 11,19, plus every repeat job | **2 hard, ~15 soft** |
| **L4 — in-cell editing** | none blocked; all 35 slowed | **35 × friction** |

---

## 3. PART 2 — Competitive gap analysis

### 3.1 Avery Design & Print Online (the reference product; free, browser)

- **Templates** — thousands, organised by *job* (address, name badge, jar label, ticket,
  gift tag, business card, wedding), not by size. You pick "name badge", not "99×68 mm".
- **Mail & Data Merge** — import a spreadsheet; map columns onto the design; "Edit One /
  Edit All" toggles between changing every label and changing one.
- **Barcode generator** — barcodes *and* QR codes, with a **merge path**: the barcode content
  can come from a spreadsheet column **or** from a sequential number, chosen in the same
  dialog. Option to show/hide the human-readable text under the code.
- **Sequential numbering** — explicitly marketed for raffle tickets and price tags.
- **Image library** + upload; crop, size, position.
- **Curved text**.
- **Saved projects in the cloud** — reopen, edit, reorder, share from any device.
- **Custom shapes**, white underprint.

**What Peaklab lacks vs Avery**: job-named templates, spreadsheet merge, QR, barcode fed by a
column rather than by the label's own text, saved reusable designs, multi-sheet runs.
**What Peaklab has that Avery does not**: sub-millimetre printer offset calibration, colour
tags for part-used sheets, grid-first block selection and keyboard control, offline operation,
Mongolian UI. These are genuinely better for the actual moment of use.

### 3.2 Niimbot and Phomemo phone apps (the volume competitor)

These are the tools the target customer already has on their phone, because the printers are
cheap and everywhere.

- Niimbot: 10+ fonts, **100+ borders**, 1500+ symbols, barcodes, QR, **icons**, **time/date
  element**, images, **tables**, pre-made templates, **Excel import for batch printing**.
- Phomemo (Print Master / Labelife): 280+ fonts, 320+ borders, 4000+ icons, 800+ borders,
  6500+ materials, **880–8000 templates**, Excel batch import, barcode + QR, **sequential
  numbering for text and barcodes**, OCR, 29 languages, all permanently free.

**The pattern**: a **time/date element** and a **numbering element** are treated as first-class
label objects, alongside text and barcode. Peaklab has numbering (as a fill action, not an
element) and no date at all.

**The strategic read**: these apps print onto *rolls*, one label at a time. They do not compete
for A4 sheet work. But they set the customer's expectation of what "a label app" contains —
and a Mongolian shop owner who has used the Niimbot app will look for QR, date and Excel import
in Peaklab and not find them.

### 3.3 Canva

- Label templates organised by occasion and product, brand kit (fonts, colours, logo),
  **Bulk Create** (one design per spreadsheet row, autofill fields on a template), QR code app,
  resize, print service.
- **Weakness Peaklab should exploit**: Canva is a design canvas that produces *one* label
  beautifully; laying 40 different labels onto a physical A4 die-cut grid at exact millimetre
  positions is painful in Canva and nobody does it. Canva also cannot calibrate a printer that
  feeds 0.4 mm off-centre.

### 3.4 Free browser label makers (the direct substitutes)

| Tool | What it gives |
|---|---|
| **generatorbarcode.com/label-maker** | Tile N labels per page on A4 or Letter, all in-browser, no signup. Code 128, EAN, UPC, ITF-14, ISBN, ISSN, Data Matrix, QR |
| **barqrcodelabel.com** | A4/A5/A6/custom, adjustable rows, columns, margins, gaps; Code 128, EAN, UPC, QR, Data Matrix |
| **LabelInn** | Text (font, size, colour, **alignment**, bold), barcode (Code 128, EAN-13, UPC-A, Code 39, ITF-14, GS1-128), QR (URL / text / **Wi-Fi**, error-correction levels), image, **shapes — rectangles with fill, border, rounded corners, and lines**, real-millimetre canvas, **Excel merge**, PNG/PDF export, no watermark |
| **Maestro Label Designer** (OnlineLabels) | 9 barcode types, **7 QR content types**, mail merge with a choice of **"Simple Text Field"** (each column a separate movable text box) vs **"Combined Text Field"** (several columns in one box) |
| **Lableo** | A4 sheets, roll labels, barcodes, QR, Excel import |

**The uncomfortable finding**: the free browser competitors already do adjustable-grid A4 sheets
*plus* barcodes *plus* QR *plus* Excel merge. Peaklab's differentiators are the calibration
offset, the colour-tag part-used-sheet tracking, the block-selection interaction, offline use,
and Mongolian. Those are real and defensible — but the feature floor for "a credible free A4
label tool" in 2026 is: **grid + barcode + QR + spreadsheet merge + multi-page**. Peaklab is
currently 2 of 5. Today's barcode work made it 3 of 5.

### 3.5 The capability checklist, scored

| Capability | Avery | Niimbot/Phomemo | Canva | Free web tools | **Peaklab today** |
|---|---|---|---|---|---|
| Job-named templates | ✅ thousands | ✅ hundreds | ✅ | ⚠️ some | ❌ (sizes only) |
| Spreadsheet → field mapping | ✅ | ✅ Excel | ✅ Bulk Create | ✅ | ❌ (one column into one field) |
| Barcode | ✅ many | ✅ | via app | ✅ many | ✅ **Code 128 only** |
| Barcode content from a *column* | ✅ | ✅ | ⚠️ | ✅ | ❌ (only from the label's own text) |
| QR code | ✅ | ✅ | ✅ | ✅ | ❌ |
| Date / time element | ⚠️ | ✅ | ❌ | ⚠️ | ❌ |
| Currency formatting | ❌ | ❌ | ❌ | ❌ | ✅ **₮ + thousands, better than all of them** |
| Shapes / borders / fill | ✅ | ✅ 100–800 borders | ✅ | ✅ | ❌ |
| Text alignment | ✅ | ✅ | ✅ | ✅ | ❌ |
| Image library | ✅ large | ✅ 4000 icons | ✅ | ❌ | ❌ (own upload only) |
| Saved designs | ✅ cloud | ✅ | ✅ | ⚠️ | ⚠️ whole-sheet JSON only |
| Multi-sheet runs | ✅ | ✅ (roll) | ✅ | ✅ | ❌ |
| Sequential numbering | ✅ | ✅ | ⚠️ | ✅ | ✅ |
| Printer offset calibration | ⚠️ | n/a | ❌ | ⚠️ | ✅ **best in class** |
| Part-used sheet tracking | ❌ | n/a | ❌ | ❌ | ✅ **unique** |
| Offline | ❌ | ✅ | ❌ | ⚠️ | ✅ |
| Mongolian UI | ❌ | ⚠️ | ⚠️ | ❌ | ✅ |

---

## 4. DELIVERABLE — the prioritised build list

### 4.0 The one-paragraph argument

**Two features unlock more jobs than everything else on this list combined: column→field import
(15 of 35 jobs) and multi-sheet runs (11 of 35).** They are one feature in two halves — mapping
without pagination silently truncates the user's data at one sheet, and pagination without
mapping gives you more sheets you still have to fill by hand. Build mapping first because it is
cheaper, lower-risk, and the slot renderer it needs *already exists* as of today's price/barcode
work; build pagination immediately after, because the business model is selling paper and a tool
that can only ever fill one sheet at a time sells one sheet at a time. Everything below those two
is a rounding error by comparison.

---

### P0 — the tool is not credible without these

#### P0-1. Column-mapped data import ("paste your spreadsheet")

**What to build.** Replace/augment the "paste a list" box. On paste, detect the delimiter (tab
first — that is what a spreadsheet copy gives you — then comma, then semicolon). Show the first
3 rows as a table with a `<select>` above each column offering: *Label text · Price · Barcode ·
QR · Small print · Copies · Ignore*. Add a "first row is headers" checkbox. Add a **Copies**
mapping so one row can produce N identical labels. Fill in reading order across the selection,
with the existing "repeat to fill" behaviour retained. Report "42 rows → 126 labels; 24 did not
fit" instead of silently truncating.

**Jobs unblocked**: 15 directly (1, 2, 3, 5, 6, 10, 13, 17, 21, 22, 24, 30, 31, 33, 34), plus it
is the delivery mechanism for the price and barcode fields that *just* shipped and currently have
no bulk path at all.

**Difficulty**: **Low–medium.** Pure JS, ~150 lines of logic plus a small table UI. The renderer
already has `.nm`, `.pr` and `.bc` slots. `pasteText()` already splits on `\t` (line ~1319), so
the delimiter handling is half-written. The only real care needed is CSV quoting (`"a,b",c`) —
about 25 lines of a proper splitter. No dependency, no build step.

**Why first**: the single largest job-count unlock, the cheapest of the top three, and it closes
a hole the barcode/price work opened an hour ago — you can now put a price and a barcode on a
label, but only by selecting labels and typing one price at a time.

---

#### P0-2. Multi-sheet runs

**What to build.** Generalise `cells` to N pages. Cheapest honest implementation: keep one flat
array of length `cols*rows*pages`, add `S.pages` (integer), and render `pages` sibling `.sheet`
divs each with `break-after: page` in the print stylesheet. `@page{size}` already handles the
paper. Add: a page count control, "Add sheet / Remove sheet", a page strip in the toolbar,
`Ctrl+PgUp/PgDn`, and a "labels needed → sheets needed" readout ("284 labels = 3 sheets of
102, 22 spare"). Selection, undo snapshot (`snap()`), `resize()`, `ref()` and the row/column
headers all need the page dimension threaded through.

**Jobs unblocked**: 11 (1, 3, 13, 16, 17, 21, 22, 24, 25, 31, 32) — and it is the difference
between a demo and a production tool for every one of them. A class of 30 pupils needing 8
labels each is 240 labels; a small warehouse is 200–2000 bin labels; a seller ships 60 parcels.

**Difficulty**: **Medium.** No new libraries, but it touches the most code of anything on this
list — state, undo, selection maths, headers, print CSS, autofit cache keys, localStorage size.
Budget 250–350 lines and expect to find three off-by-one bugs in the index arithmetic.
The one genuine risk: **localStorage quota**. Images are stored as data URLs; five sheets each
carrying a logo will hit the 5 MB ceiling. Mitigate by deduplicating images by content hash
(they already live in a shared `S.images` map keyed by id — just hash the dataURL before
adding), and by capping the number of pages at something honest like 20.

---

#### P0-3. QR codes

**What to build.** A per-cell `c.qr` string plus a `qrSize` percentage, rendered as an SVG next
to the barcode slot. Content types worth offering, since they are just prefixes: plain text,
URL, **Wi-Fi** (`WIFI:T:WPA;S:<ssid>;P:<pass>;;`), phone, and vCard-lite. Must accept Cyrillic
(UTF-8 byte mode) — this is precisely where Code 128 fails and the tool already toasts an error.

**Jobs unblocked**: 7 (9, 14, 15, 19, 28, 31, 35), and job 35 (marketing QR stickers — menu QR,
Facebook page QR, review QR, QPay) is arguably the highest-volume sticker job in Mongolia right
now. Sheets of QR stickers is a paper-selling job in its own right.

**Difficulty**: **Medium.** A minimal QR encoder — byte mode, versions 1–10, ECC level M,
Reed–Solomon over GF(256) — is about 300–400 lines of dependency-free JS, which is the same
order as the Code 128 encoder already in the file. Alternatively `cdnjs.cloudflare.com/ajax/
libs/qrcode/1.5.1/qrcode.min.js` or `bwip-js/4.11.4`. **I recommend inlining it**, for
consistency with the hand-written Code 128 and because the README promises "save index.html
anywhere and open it — it is fully self-contained"; a CDN script silently breaks offline use,
which is a real feature for a shop with unreliable internet.

---

#### P0-4. Three type levels in one label (a "small print" slot)

**What to build.** Add `c.s` — a small-print block rendered below `.nm`/`.pr` at a fixed
fraction of the fitted size (`.nm` at 1em, `.pr` at 2em, `.s` at 0.6em), and a per-cell
alignment (`left | center`) since ingredient lists and addresses read badly centred. Autofit
already scales the whole `.txt` block, so relative sizes come free — this is genuinely a small
change to `cellHTML()` plus one textarea and one segmented control in the sidebar.

**Jobs unblocked**: 8 (5, 7, 9, 11, 17, 21, 22, 34) — and it is the *only* thing standing
between the tool and the entire food-producer, pharmacy and address-label markets, all of which
require a big line and a small line on the same sticker.

**Difficulty**: **Low.** ~60 lines. The highest value-per-line on this list after P0-1.

---

#### P0-5. In-cell editing (double-click to type)

**What to build.** `dblclick` on a `.cell` makes `.nm` contenteditable, Enter commits, Escape
cancels, Tab commits and moves to the next label. Everything else stays as-is.

**Jobs unblocked**: none *blocked*, all 35 *slowed*. A café with 14 different products currently
does 14 select→sidebar→apply round trips. Every competitor lets you click and type.

**Difficulty**: **Low.** ~60 lines, main care being that it must call `mark()` for undo and must
not fight the drag-select handler.

---

### P1 — unlocks whole new jobs

#### P1-1. Dates
Insert-date controls: **today**, **today + N days**, **today + N months**, with a format picker
(`dd.mm.yyyy` / `yyyy-mm-dd` / `dd/mm/yy`) and a label prefix ("Дуусах хугацаа: "). Best as a
button that writes a computed string into the text or small-print field rather than a live
element — simpler, and printed labels do not need to stay live.
**Jobs**: 8 (8, 10, 12, 18, 23, 28, 29, 34). Job 29 (service stickers) is *entirely* this.
**Difficulty**: **Low.** ~50 lines, zero dependencies.

#### P1-2. Job-named templates
Ship 10–14 built-in label designs, named by job in Mongolian, each of which sets the size preset
*and* the slot arrangement *and* placeholder text: price tag, SKU + barcode, best-before,
ingredient label, parcel address, thank-you QR, bin location, asset tag, name badge, file spine,
pupil name, service due, raffle ticket. This is what makes Avery and Niimbot feel complete —
the customer does not think "48×25 mm", they think "price tag".
**Jobs**: touches all 35; converts the tool from "a grid" to "a tool that knows my work".
**Difficulty**: **Low–medium** once P0-1/P0-4 exist — it is a data table of preset objects plus
a picker UI. Do **not** build this before the slot system, or you will ship 14 templates that
all look the same.

#### P1-3. Saved designs, separate from sheet contents
Name and store a *design* (size + slots + styling + fixed boilerplate text like a pharmacy's
address) in `localStorage`, listed in a picker, applied to any selection. Distinct from the
existing whole-sheet JSON export.
**Jobs**: 11 and 19 hard, ~15 more soft (any repeat job).
**Difficulty**: **Low.** The `custom[]` preset machinery already does exactly this for sizes.

#### P1-4. Copies-per-row / quantity
"Print 8 of each" as a global multiplier and as a mappable column in P0-1.
**Jobs**: 3 (1, 12, 24) — but job 24 (pupil name labels) is the entire school market and is a
seasonal paper-buying spike.
**Difficulty**: **Trivial** (~20 lines) once P0-1 and P0-2 exist. Nearly free; do it in the same
change as P0-1.

#### P1-5. Rotation (90° / 180° / 270°)
Per-cell `transform: rotate()` with the autofit box swapped.
**Jobs**: 3 (20 file spines, 25 book spines, 27 cable tags) — but these jobs are *completely*
impossible today and are entirely unserved by phone label apps on A4.
**Difficulty**: **Low–medium.** ~40 lines, but `availBox()` and the autofit binary search need
width/height swapped for the rotated case, and it must survive the print path.

#### P1-6. Alignment, borders and background fill per label
Left/centre/right, a hairline or solid border, and a solid background colour (which prints and
makes "-30%" stickers and zone-coloured bin labels work).
**Jobs**: 4, 7, 13, 14, 15, 16, 26 — 7, partially.
**Difficulty**: **Low.** ~50 lines. Alignment should be pulled forward into P0-4; the rest can
wait. Note that background fill requires the user to enable "Background graphics" in the print
dialog, which the tool already warns about for cut lines.

#### P1-7. More barcode symbologies (EAN-13, UPC-A, Code 39, ITF-14)
EAN-13 in particular, because a shop reselling manufactured goods labels with the
manufacturer's EAN, not an internal SKU.
**Jobs**: strengthens 3, 5, 17.
**Difficulty**: **Low–medium.** EAN-13 is ~80 lines of pure JS (three encoding tables plus a
check digit); Code 39 is ~40. Or add `jsbarcode/3.12.3/JsBarcode.all.min.js` from cdnjs — but
see the offline caveat under P0-3.

#### P1-8. Two missing A4 sizes
**210×148.5 (2-up, A5)** for parcel labels and **70×25.4 (33-up)** for address/small labels,
plus 63.5×38.1 (Avery L7160 21-up) and 99.1×38.1 (14-up) to match what people actually buy.
**Jobs**: 13, 22.
**Difficulty**: **Trivial.** Four rows in `BUILTIN`.

---

### P2 — nice

- **P2-1. Round / oval / rounded label shapes** (`border-radius:50%` plus a clip for content).
  Jobs 14, 26, 9. ~30 lines. Cosmetic but sells round sticker paper.
- **P2-2. A print-check page** — a single test sheet with registration crosses at the four
  corners and a mm rule, so the offset can be dialled in without wasting a sticker sheet.
  Cheap, and it makes the tool's best existing feature discoverable.
- **P2-3. Wi-Fi / vCard / phone QR wizards** — trivial once P0-3 lands, high delight for cafés.
- **P2-4. Strikethrough + dual price** for discount stickers (job 4).
- **P2-5. "Print only selected labels"** — blank everything unselected for one print run, so a
  part-used sheet can be topped up. This pairs with the colour-tag feature, which is the tool's
  most original idea and currently has no printing counterpart. Higher value than its position
  suggests if the colour tags are genuinely used.
- **P2-6. Symbol set** — a couple of dozen inline SVG glyphs (allergen icons, fragile, recycle,
  arrows, stars). Not an image library; a small curated set. Jobs 6, 7, 26.
- **P2-7. Sequential numbering across multiple levels** for bin locations (aisle × bay × level).
  Job 16 only, but that job is high paper volume.

---

### 4.4 Things that look obvious but are traps here

**PDF export.** Tempting, and every competitor lists it. But the tool is Mongolian: jsPDF's
built-in fonts have no Cyrillic, so you would have to base64-embed a Cyrillic-capable TTF
(100–300 KB) into a file whose whole identity is being one self-contained HTML page — and
you would still have to re-solve autofit against PDF text metrics. Meanwhile every browser
already offers "Save as PDF" in the print dialog, at exact mm, with the existing `@page` rule.
**Skip it.** If print fidelity is the actual worry, build P2-2 (the print-check page) instead.

**An image / clipart library.** Avery's is a headline feature and Phomemo claims 4000 icons.
For this product it is a trap: it needs hosting or a large inline payload (killing the
self-contained promise), it carries licensing exposure, it inflates localStorage, and the actual
demand is "my own logo", which already works. A curated 24-glyph inline SVG set (P2-6) captures
90% of the value at 1% of the cost.

**A freeform drag-and-drop design canvas.** This is the obvious "become Canva" move and it would
destroy the product. The reason this tool is good is that it is grid-first: block selection,
keyboard navigation, apply-to-many. A canvas makes 40 different labels *slower*, not faster, and
puts you into a fight with Canva that you lose. Keep slots, not free positioning.

**Direct thermal-printer support (Niimbot/Phomemo/Zebra).** LabelInn advertises it. But this
tool exists to sell A4 sticker paper — every label pushed to a thermal roll is paper you did not
sell. Actively out of scope.

**Cloud accounts / saved projects on a server.** Avery's differentiator, but it means a server,
which the constraint forbids and the business does not need. P1-3 (localStorage designs) plus
the existing JSON export gets 90% of it.

**Curved text, 280 fonts, 800 borders.** Phone-app feature-count theatre. Three faces and a
sensible border set is enough for print at 32×15 mm; more choices make small labels worse.

**Barcode symbology sprawl.** Beyond EAN-13 and Code 39, the long tail (GS1-128, Data Matrix,
PDF417, ITF-14) serves industrial supply chains that are not this customer. Adding bwip-js to
get them costs a heavy dependency and breaks offline use for jobs nobody in this market has.

---

### 4.5 Build order, one line each

| # | Feature | Priority | Jobs | Effort |
|---|---|---|---|---|
| 1 | Column-mapped data import (paste a spreadsheet, map columns to slots, copies column) | **P0** | 15 | Low–med |
| 2 | Multi-sheet runs (N pages, page navigation, "how many sheets" readout) | **P0** | 11 | Medium |
| 3 | QR codes (inline encoder, UTF-8, URL/Wi-Fi/text presets) | **P0** | 7 | Medium |
| 4 | Small-print slot + per-label alignment (three type levels) | **P0** | 8 | Low |
| 5 | In-cell editing (double-click to type) | **P0** | 35 × friction | Low |
| 6 | Dates: today / +N days / +N months, format picker | P1 | 8 | Low |
| 7 | Job-named templates (10–14 built-ins) | P1 | all 35 | Low–med |
| 8 | Saved designs with fixed boilerplate | P1 | 2 hard, 15 soft | Low |
| 9 | Copies-per-row / quantity | P1 | 3 | Trivial |
| 10 | Rotation 90/180/270 | P1 | 3 | Low–med |
| 11 | Borders + background fill | P1 | 7 | Low |
| 12 | EAN-13, Code 39 | P1 | 3 | Low–med |
| 13 | Four more A4 sizes (210×148.5, 70×25.4, 63.5×38.1, 99.1×38.1) | P1 | 2 | Trivial |
| 14 | Round/oval shapes | P2 | 3 | Low |
| 15 | Print-check / registration page | P2 | all | Low |
| 16 | Wi-Fi / vCard QR wizards | P2 | 2 | Trivial |
| 17 | Strikethrough + dual price | P2 | 1 | Low |
| 18 | Print only selected labels | P2 | all | Low |
| 19 | Curated 24-glyph symbol set | P2 | 3 | Low |
| 20 | Multi-level sequential numbering | P2 | 1 | Low |

---

## 5. Sources

- [Avery Design & Print](https://www.avery.com/software/design-and-print/) ·
  [QR and barcode merge](https://www.avery.com/help/article/qr-and-barcode-merge) ·
  [Free online label maker](https://www.avery.com/software/label-maker/)
- [Maestro Label Designer](https://www.onlinelabels.com/maestro-label-designer) ·
  [Mail merge guide](https://onlinelabels.com/support/maestro/mailmerge/mail-merge-guide.aspx)
- [LabelInn label maker](https://labelinn.com/en/label-maker) ·
  [generatorbarcode label maker](https://generatorbarcode.com/label-maker/) ·
  [barqrcodelabel](https://barqrcodelabel.com/) · [Lableo](https://labelo.me/) ·
  [MyLabelMaker](https://mylabelmaker.com/)
- [Niimbot label templates](https://openlabelmaker.com/labels/niimbot-labels/) ·
  [Niimbot support](https://www.niimbotlabel.com/pages/instruction) ·
  [Phomemo E50Pro](https://phomemo.com/products/phomemo-e50pro-industrial-label-maker) ·
  [Phomemo app](https://play.google.com/store/apps/details?id=com.quyin.phomemo)
- [Canva labels](https://www.canva.com/labels/)
- [Cottage food labeling requirements](https://findhomegrown.com/blog/cottage-food-labeling-requirements) ·
  [FDA small food business labels](https://foodscompliance.com/fda-food-label-requirements/)
- [Warehouse labeling guide](https://inventoryquick.com/blog/warehouse-labeling-guide) ·
  [Code 128 for bin locations](https://barcodx.com/articles/bulk-code128-for-warehouse-bins) ·
  [Warehouse location labeling](https://spartanpos.com/pages/warehouse-location-labeling-guide)
- [IT asset tagging best practices](https://www.assetpanda.com/resource-center/blog/it-asset-tagging-best-practices/) ·
  [QR codes for asset management](https://blog.invgate.com/qr-codes-for-asset-management)
- [Prescription labeling requirements](https://www.consumermedsafety.org/safety-tips/learn-to-read-a-prescription-label) ·
  [MN Rules 6800.3400](https://www.revisor.mn.gov/rules/6800.3400/)
- [Mongolian retail/food labelling guidance (MNS CAC 1:2007, MNS 5021-1:2019)](https://mongolia.gov.mn/news/view/8494)
- Library availability checked on cdnjs: `jsbarcode/3.12.3`, `qrcode/1.5.1`, `bwip-js/4.11.4`
