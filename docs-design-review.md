# Peaklab Labels — design review

Reviewed live at https://kizaialt.github.io/label-sheet-printer/ against source
`C:\Users\kizai\Claude\label-sheet-printer\index.html` (2425 lines, single file).
Exercised at 1440×900, 1280×800, 920×700, 860×640 and 420×700, light and dark.
All four job modes, barcodes, QR, logo drop, colour tags, paint mode, drag-select,
multi-page, size switching, undo, export/import inspected.

Counts: **4 P0 · 14 P1 · 15 P2**

---

## Verdict on the three questions you led with

**Did the declutter work?** Partly, and the part that worked is the part you can
see in a screenshot. The default sidebar is genuinely calmer than six open
sections: two sections open, three closed, two sub-drawers shut. But the depth
did not go anywhere. Fully expanded the sidebar is **2918px of content in a
752px viewport — 3.9 screens**, and Content alone is 982px. What changed is that
there are now *three* levels of hiding in one column (section → sub-section →
mode-conditional field), so a control can be invisible for three different
reasons and the user cannot tell which. And the section that most needed the
declutter — **Хуудас** — was not touched: it still holds seven unrelated
concerns (size readout, size picker, printer offset, grid fine-tune,
save/delete preset, two display checkboxes, export/import/empty).

Worse, the mode picker didn't just hide fields — it made them **destructive**.
See P0-3.

**First-run.** The first minute is *legible* but not *obvious*, and it opens
with a small lie: cell A1 carries the accent cursor ring while the sidebar says
"Юу ч сонгоогүй". The empty-sheet card tells you the right two things (drag to
select; type in Агуулга) but neither of the things it names is visually
connected to it, and the first object in the sidebar is a row of six selection
chips, three of which cannot do anything yet.

**The job picker.** It is not currently a segmented control. It renders as a
**2×2 button pad** because of a CSS source-order bug (P0-1). As a concept it
reads as a choice — "Юу хийх вэ?" is a good question and Энгийн/Үнэ/Бараа/QR
are good answers — but as built it looks like a toggle grid, and because the
mode is global rather than a property of the label, it silently deletes data.

---

## P0 — broken or embarrassing

### P0-1. The job picker is not a segmented control. It renders as a 2×2 pad.

**Where:** `index.html:212–213`.

```
212: .seg4{grid-template-columns:repeat(4,1fr)}
213: .seg{display:grid;grid-template-columns:1fr 1fr; ...}
```

Equal specificity, `.seg` declared last, so `.seg` wins.
`getComputedStyle(#modeSeg).gridTemplateColumns` returns `"149px 149px"`, and
the four buttons lay out at y=207,207,234,234. Confirmed on the live site.

Two consequences beyond the wrong shape: `.segb+.segb{border-left:...}` (line
219) puts a stray vertical rule down the **left edge** of the second row, and
there is no horizontal rule between rows, so the two rows read as one blob.
The `.on` state then looks like a highlighted cell in a keypad rather than the
selected segment of a switch.

**Why it hurts:** this is the single element the whole restructure hangs on. A
shop owner sees four boxes in a square with one shaded and reads "four buttons,
one is on" — that is a toggle pad, not "which of these four am I making". The
adjacent `Бүтэн / Градиент` control *does* render as a proper 2-up segmented
control, so the same page shows both the intended pattern and the broken one
side by side.

**Fix:** change line 212 to `.seg.seg4{grid-template-columns:repeat(4,1fr)}` or
move it below line 213. Then, because `.segb+.segb` is now correct for a single
row, nothing else changes. Verify "Энгийн" (the widest label) at the 312px
sidebar width — at 10.5px it needs ~44px and each segment gets ~74px, so it
fits; if you raise the type size per P1-7, drop the segment padding to 0 8px.

### P0-2. Changing the label size in the toolbar silently destroys labels, and is not its own undo step.

**Where:** `pickPreset()` at `index.html:2261–2271`, `resizeOne()` at `1119–1132`.

`pickPreset` calls `resize(S, oc, or)` and never calls `mark()`. `resizeOne`
does a **positional** remap: it copies only `min(oldRows,newRows) ×
min(oldCols,newCols)` cells and drops the rest.

Measured on the live site: a sheet with **30 filled labels** on 6×17 switched to
the 99×68 preset (2×4) → **6 labels survived, 24 destroyed**, no confirm, no
toast, no undo entry.

Then, because there is no snapshot, pressing Ctrl+Z afterwards pops whatever the
*previous* edit was. In my session one Ctrl+Z after the size change reverted the
geometry *and* deleted the second page I had added — the user gets a random
earlier state, not "put my size back".

**Why it hurts:** the size dropdown is the second-most prominent control on the
page, and "let me see what this looks like on bigger stickers" is a completely
normal thing to try. Losing 90 labels for it is the kind of thing that makes
someone close the tab and use Word.

**Fix, in order of value:**
1. Call `mark()` as the first line of `pickPreset()` so the size change is one
   undo step. (Two-line fix, do it today.)
2. Change the remap from positional to **reflow**: collect the non-empty cells
   in reading order, lay them into the new grid in reading order, and push the
   overflow onto additional pages (`S.pages.push(...)`). A shopkeeper thinks
   "my 30 milk labels", never "cell B4". This also makes "design at 32×15,
   reprint at 48×25" a feature instead of a hazard.
3. If you keep positional remap, at minimum count what would be dropped and
   `confirm()` with the number, the way `clearAll` already does.

### P0-3. Switching job mode silently strips price / barcode / QR from labels, and hides data the label is visibly carrying.

**Where:** `$("applyText")` handler, `index.html:1929–1951`.

```js
var pv = mode==="price" ? $("priceInput").value : "";
var bc = mode==="sku"   ? $("bcChk").checked    : false;
var qv = mode==="qr"    ? $("qrInput").value    : "";
...
c.t=v; c.p=pv; c.s=sv; c.bc=bc?1:0; c.qr=qv;
```

Anything not owned by the *current* mode is written as empty. Reproduced live:
made a price label (`{t:"Сүү 1 л", p:"89000"}`), switched to Энгийн, selected it,
pressed Сонгосонд бичих → stored cell became `{t:"Сүү 1 л", p:""}`. The toast
said "1 шошго дээр бичлээ". Nothing said a price had been deleted.

The companion problem: while in Энгийн mode, `applyMode()` (line 1012) hides
`#priceRow`. So the user is looking at a label with **89,000₮ printed on it in
24pt**, the sidebar header says "A1 Сүү 1 л", and there is no price field
anywhere on screen. The data exists, is visible on the paper, and is
unreachable and unexplained.

**Why it hurts:** it converts the declutter into a data-loss mechanism. A café
does a price sheet, then wants one plain "Цэс" label, switches to Энгийн, later
re-selects a block to fix a typo, hits Apply, and the prices are gone from every
label they touched.

**Fix (choose one, (a) is closest to the current design):**

(a) Make the mode a property of the selection, not a global switch.
  - In `selCount()` (line 1603), when exactly one cell is selected, derive the
    mode from the cell — `c.qr ? "qr" : c.bc ? "sku" : c.p ? "price" : mode` —
    set `S.opts.mode` and call `applyMode()`. Selecting a price label then
    shows the price field, which is what the user expects from an inspector.
  - In the apply handler, **never write a field whose row is hidden**. Only set
    `c.p` when `#priceRow` is visible, etc. Clearing a price then becomes an
    explicit act (empty the field, or Арилгах), not a side effect of a mode.

(b) Drop the mode picker entirely and show Text / Price / Small line always,
    with Barcode and QR behind one "Код нэмэх ▾" toggle. Fewer states, no
    hidden data. This is a smaller UI than the current one for the two common
    jobs and honest about the other two.

I would ship (a). But note (b) is worth considering seriously: with the picker
fixed to four segments you have spent a full-width control and a question
("Юу хийх вэ?") to hide, at most, two fields.

### P0-4. The instruction that decides whether the print succeeds is in a collapsed drawer, and is never shown at print time.

**Where:** `printHint` string at `index.html:~836`, inside the
`Хэвлэх ба гарын товчлол` section — the **last** `<details class="sec">`, closed
by default. `doPrint()` (line 2356) goes straight to `window.print()`.

The hint is genuinely good copy: A4, margins **None**, scale **100%** (not "fit
to page"), headers/footers off, test on plain paper first, then use Принтерийн
зөрүү. Every one of those is required for the stickers to land on the die-cut.

**Why it hurts:** this is the whole commercial proposition. Peaklab gives the
tool away so their paper works. If the first print comes out 4mm low because
Chrome defaulted to "Fit to printable area", the customer concludes *the paper
is wrong*. You have written the fix and then hidden it below three other
sections in a drawer that says "Хэвлэх ба гарын товчлол" — which reads like
keyboard-shortcut trivia, so nobody opens it.

**Fix:** intercept the Хэвлэх button once. Show a small in-page confirm panel
(not a browser dialog) with the three settings as a checklist, a line about
test-printing on plain paper, a link that opens Принтерийн зөрүү, a
"Дахиж бүү үзүүл" checkbox persisted in `S.opts`, and one primary
"Хэвлэх" button that calls the existing `doPrint()`. Cost: ~40 lines. It is the
highest-leverage change in this document.

---

## P1 — clearly worth fixing

### P1-5. Undo after a size change leaves the size selectors lying about the sheet.

`snap()` (line 1639) stores `{pages, page, g}` but **not `S.presetId`**.
`restore()` then calls `syncPresetSel()`, which sets both selects to the stale
`S.presetId`.

Observed live and visible in one screenshot: the sheet is 6×17, the `Хуудас`
well correctly reads "Нэг шошго **32 × 15 mm** · A4 хуудсанд **102** ширхэг",
and the select immediately beneath it reads "**99 × 68 mm · 2×4 · хуудсанд 8**".
The toolbar select says the same wrong thing. Re-picking that preset from the
dropdown then triggers P0-2 again.

**Fix:** include `presetId` in `snap()` and restore it. One line each.

### P1-6. The Content panel is an inspector for one cell and a stale form for many.

`selCount()` populates `#txtInput`, `#priceInput`, `#subInput`, `#bcChk`,
`#qrInput` **only** in the `arr.length===1` branch. With 2+ selected the fields
keep whatever was last typed.

Observed: three price labels reading "Сүү 1 л" were selected while the text box
still read "ABC-1234" from a previous operation. Pressing Сонгосонд бичих would
have rewritten all three to ABC-1234 and (per P0-3) blanked their prices.

**Why it hurts:** the panel teaches you it is a mirror (single select) and then
stops being one without saying so. That is exactly how people destroy work.

**Fix:** on multi-select, show each field's value if it is uniform across the
selection, and a distinct "олон утга" placeholder if not (grey italic, empty
value). On apply, skip fields still showing "олон утга". If that is too much,
at least clear the fields on multi-select so nothing false is on screen.

### P1-7. Almost the entire sidebar is set at 10.5px.

`--fs-micro:10.5px` (line 40) is used by `.lbl`, `.hint`, `.chip`, `.segb`,
`.check`, `.well`, `.sw .n`, `.pgnum`, `.count span` and `details.sub>summary`.
Only inputs and section titles get 13.5px. So in practice the sidebar is a
10.5px interface with a few 13.5px exceptions.

**Why it hurts:** the audience is shop and pharmacy owners on a laptop in a shop,
not designers on a 27". 10.5px Cyrillic at 4.7:1 contrast (`--ink-3` #6f6d67 on
`--panel` #faf9f8) is right on the AA floor and below the comfort floor.

**Fix:** `--fs-micro: 11.5px` and introduce `--fs-nano: 10px` used *only* for
the swatch counts and the cell index. Bump `.segb` and `.check` to
`--fs-body` (13.5px) — those are controls, not annotations. Raise `--ink-3`
to about `#63615b` in light so hint text lands nearer 5.5:1.

### P1-8. Three separate rows of unlabelled number boxes.

- Auto-number: `#seqPrefix` / `#seqStart` / `#seqPad` (line ~560) — rendered as
  `SKU-` `1` `3` with nothing to say the `3` is zero-padding. Labels exist only
  as `title`/`aria-label`.
- Printer offset: `#ox` / `#oy` — two boxes both showing `0`, no visible X/Y.
- Fine-tune margins (4 boxes) and gaps (3 boxes) — the group label enumerates
  them in order, which is better but still asks the user to count.

**Why it hurts:** a shopkeeper will not hover to discover what a box means, and
on the offset row there is no ordering cue at all, so the two most consequential
numbers in the app are a coin flip.

**Fix:**
- Offset: prefix the inputs visibly — `→` and `↓` glyphs as static adornments,
  or micro-labels "Баруун (mm)" / "Доош (mm)". Better still, replace with a
  4-arrow nudge pad that steps 0.5mm and shows the current shift as text; the
  `shiftedFmt` string already exists for the readout.
- Auto-number: replace the three `title`s with micro-labels and add a live
  preview line under the row — `SKU-001, SKU-002, SKU-003 …` computed from the
  three fields. That removes the need to explain "orон" at all.
- Suffix currently sits on the *next* row beside the Дугаарлах button, which
  splits one operation across two visual groups. Put prefix/start/digits/suffix
  in one 4-up grid, button on its own line.

### P1-9. Printer offset — the control that makes their paper work — is buried.

It is the fourth item inside the closed `Хуудас` section, below a size select
and a calc well, with the unlabelled inputs from P1-8.

**Why it hurts:** "your printer feeds slightly off, nudge the grid" is the answer
to the single most common support complaint for die-cut sticker paper, and it is
the thing that differentiates this tool from a Word table. It should be one of
the first things a returning user can reach.

**Fix:** surface it in the pre-print panel from P0-4 ("Өмнөх хэвлэлт хазайсан
уу? → Зөрүү тохируулах"), and put a compact indicator in the topbar when a
non-zero offset is active (currently the only sign is a line inside the closed
`#calcInfo` well). Keep the full control where it is.

### P1-10. Autofit sizes every label independently, so a set of matching labels prints at wildly different sizes.

`autofit()` (line 1558) binary-searches per cell. Measured on one sheet at 60%
zoom: A1 `.txt` = 33.5px computed (≈42pt), its neighbour B1 = 14.5px (≈18pt),
a barcode cell = 4px (≈5pt). All three are "the product name".

Look at the screenshots and you can see it — the first label shouts and the
label beside it whispers. It reads as a rendering accident, not a decision.

**Why it hurts:** shelf labels are a set. Ragged type across a set is the most
visible "made in a free tool" signal there is.

**Fix:** add a third state to the Автомат control, or a checkbox under it:
**"Сонголтыг ижил хэмжээтэй болгох"** — run the existing binary search over
every cell in the selection and apply the **minimum** result to all of them.
The cache key machinery is already there; you only need to take the min before
writing `t.style.fontSize`. Make it the default for `Жагсаалтаар бөглөх` and
`Дугаарлах`, which by definition produce a set.

Also raise the search floor from `lo=3` (3pt is not printable) to 5, and warn
when the fit lands on the floor.

### P1-11. Paint mode has no persistent on-state.

`$("paintBtn")` toggles `.on` → `background:var(--selected)` — a barely
perceptible grey shift — changes its own label, sets the sheet cursor to
crosshair, and fires a 2.1s toast. That is all. Verified: with paint on, the
screen looks essentially identical to paint off.

**Why it hurts:** it silently reassigns the meaning of clicking on the sheet.
Someone who turns it on, serves a customer, comes back and clicks a label tags
it instead of selecting it, and has no idea why.

**Fix:** while paint is on — (1) fill the button with the active tag colour and
add an explicit "✕ Гарах"; (2) turn the sticky `.selbar` into a coloured strip
reading "Будах горим · улаан" so it is on screen no matter where the sidebar is
scrolled; (3) give `.sheet` a 2px outline in the active colour. Esc should exit
paint mode (it currently only clears selection).

### P1-12. Six selection chips are the first thing in the sidebar, above everything.

`.selbar` is sticky at the top and holds the count line plus
Бүгд / Цуцлах / Эсрэгээр / Хоосон / Бөглөсөн / Тэмдэгтэй. On an empty sheet
three of them can only produce the "Тохирох шошго алга" toast. They also wrap to
two rows at the 312–340px sidebar width, with "Тэмдэгтэй" alone on row two,
which reads as a layout mistake.

**Why it hurts:** prime real estate, above the actual first step, spent on a
power-user selection console. It is the clearest example of "the confusion
moved" — Content got tidied and the top of the column stayed an expert panel.

**Fix:** keep the count line sticky. Collapse the chips to
**Бүгд · Цуцлах** plus a "Шүүх ▾" popover holding the other four, and show that
popover only when the page has at least one filled or tagged label. On an empty
sheet the whole strip should be the one-line prompt and nothing else.

### P1-13. Хуудас is a junk drawer.

Contents: `#calcInfo` well, size select (duplicate of the toolbar one), printer
offset, `Энэ хэмжээг нарийн тохируулах` sub, Save/Delete preset, "Тайрах шугам
хэвлэх", "Хоосон шошгыг дэлгэц дээр дугаарлах", Export/Import/Empty. Seven
different concerns, 613px tall, and two of them (the cut-lines and index
checkboxes) are *display and print* options that have nothing to do with sheet
geometry.

Also `#sizeSel` (line ~640) and `#presetSel` (topbar, line ~428) are the same
control kept in sync by `syncPresetSel()`. Having the size in two places is
exactly the kind of duplication a declutter is supposed to remove — and it is
what makes the P1-5 desync show up as a self-contradicting panel.

**Fix:**
- Delete `#sizeSel` from the sidebar. The topbar select is always visible; keep
  `#calcInfo` as the readout beneath the offset control.
- `Хуудас` becomes: calc readout → printer offset → fine-tune → save/delete
  preset. One idea: "does this grid match my paper".
- Move "Тайрах шугам хэвлэх" and "Цэгийг бас хэвлэх" (currently stranded in
  Colour tags) into the pre-print panel from P0-4 — they are print decisions.
- Move "Хоосон шошгыг дэлгэц дээр дугаарлах" to a view menu or drop it (see P2-21).
- Move Export / Import / Хоослох to a "…" overflow next to Хэвлэх in the topbar.

### P1-14. The only revenue link on the page is styled exactly like the theme toggle.

`#buyLink` (line 441) is `class="btn sm ghost"`; `#themeBtn` (line 443) is
`class="btn sm ghost"`. Identical. So "Цаас авах" and "Бараан" sit side by side
as two anonymous grey words.

**Why it hurts:** the entire reason this tool exists is to sell A4 sticker paper.
It should not be indistinguishable from a display preference.

**Fix:** give `#buyLink` a border and a small paper/cart glyph so it reads as a
destination rather than a setting — still quiet, still not a CTA. And add one
line at the very bottom of the sidebar, below the last section:
"Наалтын цаас дуусах гэж байна уу? — Peaklab дэлгүүр →". The bottom of a
sidebar someone has just scrolled through while preparing 102 stickers is the
single most qualified moment you will ever get.

### P1-15. Three filled "primary" buttons compete.

`Хэвлэх` (topbar), `Сонгосонд бичих` (Content), `Сонгосонд тавих` (Logo) all use
`.btn.primary` — solid `--ink`. At least two are on screen at once, three when
Logo is open.

**Fix:** primary weight belongs to the terminal action. Keep `Хэвлэх` filled.
Make `Сонгосонд бичих` an accent-filled or accent-outlined button — visually
strong, categorically different. Demote `Сонгосонд тавих` to a normal `.btn`;
it is the only enabled action in its section anyway.

### P1-16. Barcode/QR labels: the human-readable line collapses, and QR gets no size warning.

On a 32×15mm label with a Code 128 barcode, the `.txt` computed to 4px at 60%
zoom ≈ **5pt** — technically printable, practically unreadable, and it sits
*above* the bars where no scanner-label convention puts it.

Meanwhile `applyText` warns properly when bars get thin
(`tThinBarcode`, threshold `labelW()*0.96/enc.width < 0.25`) — a genuinely
thoughtful touch — but the QR path has **no equivalent**. `qrSVG` sizes at
`min(labelW,labelH)*0.62`; on the default 32×15 preset that is a 9.3mm QR, and a
28-character URL is version 3 (29 modules) → ~0.32mm per module, below the
~0.4mm most phone cameras want at arm's length. The user finds out after
printing 102 of them.

**Fix:**
1. Mirror the barcode warning for QR: compute `sizeMm / moduleCount` and toast
   when it is under 0.4mm, with the same "Илүү том шошго сонговол найдвартай"
   wording.
2. Put the human-readable text **below** the bars for barcodes (swap the order
   when `c.bc` is set) and give it a hard floor of 6pt rather than letting
   autofit take it to 3.
3. Consider not rendering `c.t` at all under a barcode when the label is under
   ~20mm tall — the bars encode it; a 5pt smear helps nobody.

### P1-17. Adding a page looks like losing your work.

`$("pgAdd")` inserts a blank page and navigates to it. The blank page then shows
the identical first-run card ("Хуудас хоосон байна"), and the only indication
you are on page 2 is "2 / 2" in a small pill under the sheet — off-screen at
default zoom on a 900px-tall window. Verified: after clicking +, the visible
result is an empty sheet and a first-run hint.

**Fix:** put the page indicator where the eye already is — a "Хуудас 2 / 2"
chip in the topbar next to zoom, or above the sheet's column headers. Change the
blank-page copy when `S.pages.length > 1` (e.g. "2 дахь хуудас — хоосон.
Өмнөх хуудас руу буцах" with a link). Long term the right control is a small
vertical page-thumbnail strip on the left of the stage, which also uses the dead
canvas space (see P2-19).

### P1-18. Below 900px the sidebar controls stretch with no max-width.

At 860px the sidebar stacks under the stage at full width. The mode pad becomes
two rows of ~380px buttons; the textarea becomes 830px wide for a 20-character
product name; `.g2`/`.g3` button rows become enormous slabs. Meanwhile
`.stage{max-height:52vh}` cuts the sheet at row 6 and the Apply button is below
the fold, so you cannot see the result of applying.

**Fix:** in the `max-width:900px` block, add
`.sidebar .body{max-width:560px;margin:0 auto}` (or a 2-column grid for the
section bodies — there is plenty of width) and make the Apply row a sticky
footer inside the sidebar: `position:sticky;bottom:0;background:var(--panel)`.
Nothing else about the stacked layout needs to change; at 420px it already
behaves.

---

## P2 — polish

- **P2-19. The page bar is always on, even at 1 / 1.** `pageUI()` ends with
  `$("pagebar").style.display="flex"` unconditionally, so a first-run user gets
  a `‹ 1/1 › | + −` pill under the sheet with three of five controls dimmed.
  Show a single quiet "＋ Хуудас нэмэх" when `S.pages.length===1`, and expand to
  the full bar from page two.

- **P2-20. The toast lands on the page bar.** `#toast` is `position:fixed;
  left:50%; bottom:22px`; the page bar is centred under the sheet, which at the
  default sidebar width is near the same x. Confirmed overlapping at 1280×800.
  Offset the toast to `bottom:22px; left:calc(50% - 170px)` on wide screens, or
  raise it above the page bar.

- **P2-21. Blank-label index numbers are noise at fit zoom.** `.cell .idx` is
  `font-size:1.9mm`, so it scales with the sheet transform: at the default 60%
  fit it renders as ~4px grey specks in 100 cells. They are on by default
  (`idx:true`). Either size them in px with a floor (`font-size:max(9px,1.9mm)`)
  and hide them below ~45% zoom, or default the option off. As shipped they add
  a grey stipple across the whole sheet and communicate nothing.

- **P2-22. The cursor ring and the selection ring look the same.** `.cell.cur`
  and `.cell.sel` both use `--accent` outlines; on first load A1 has `cur` while
  `sel` is empty, so the sheet says "this one is chosen" and the sidebar says
  "nothing is chosen" simultaneously. Make `cur` a 1px dotted `--ink-3` ring, or
  don't paint it until the sheet has been focused/clicked once.

- **P2-23. English strings leak into a Mongolian-only UI.** `aria-label="Undo"`,
  `"Redo"`, `"prev"`, `"next"`, `"add"`, `"delete"` (pagebar),
  `aria-label="Top margin" / "Right margin" / "Horizontal gap" / "Corner radius"`
  (fine-tune), `title="Use this image"` and `title="Delete image"`
  (`renderLib`, line ~2168). Everything else is carefully localised, which makes
  these stand out. Add the strings to `STR` and use the existing
  `data-i18n-aria` / `data-i18n-title` machinery.

- **P2-24. Хоослох looks like Экспорт and Импорт until you hover it.**
  `.btn.warn` only colours on `:hover`. Give the destructive one a persistent
  `--danger` text colour at rest; it already confirms, so the styling is the
  only at-rest signal.

- **P2-25. `Сонгосонд тавих` is a primary button that cannot work yet.** With no
  image loaded it toasts "Эхлээд зураг нэмээрэй". Set `disabled` until
  `Object.keys(S.images).length` in `renderLib()`; same for `Хасах`.

- **P2-26. The pt field lies while Автомат is on.** `#fontSize` shows `8` and has
  no effect whenever `fit` is true (which is the default). Dim and disable it
  when Автомат is pressed, or show the computed size as its value.

- **P2-27. Double-click edits only the name.** The inline `.celledit` reads and
  writes `c.t` only. Double-clicking a price label to fix the price puts you in a
  box that cannot change it, with the price hidden behind the editor. Either
  extend the editor to the fields the label actually has, or, cheaper, on
  double-click of a label with `p`/`qr`/`s`, select it and scroll/flash the
  matching sidebar field instead of opening the inline box.

- **P2-28. The colour spotlight can stick.** `spotlight()` is cleared only by
  `#sheet` `mouseenter` and `#swatches` `mouseleave`. Leave the swatch grid
  upward into the section header (no `mouseleave` on some paths) or move the
  pointer out of the window and the sheet stays at `opacity:.25`. Clear it on
  any `pointerdown` on `document` and on `sidebar` `mouseleave`.

- **P2-29. Logos are data-URL'd into localStorage next to the sheet.**
  `readImage` downsamples to 800px and stores a PNG data URL in `S.images`; the
  whole payload goes through `JSON.stringify` into one 5MB key. Three or four
  logos will hit quota; the handling is graceful (`tFull` toast, stops saving)
  but the design invites it. Drop the resample cap to 400px for label use, and
  consider IndexedDB for images so the sheet JSON stays small.

- **P2-30. Three near-synonyms for "clear".** `Арилгах` (clear the selected
  labels' content), `Хоослох` (empty the whole page), `Цуцлах` (deselect), plus
  `Тэмдэг арилгах` (untag) and `Хасах` (remove image). Consider `Цэвэрлэх` for
  the selection clear and reserving `Арилгах` for one meaning.

- **P2-31. Shortcut list is incomplete, and Space double-fires.** The table omits
  Enter (focuses the text box — a nice touch nobody will find), Esc, Ctrl+A,
  Ctrl+X, Home/End. Separately, the keydown guard exempts input/textarea/select
  but not `button`, so pressing Space on a focused button both activates it and
  toggles the cell under the cursor.

- **P2-32. Text runs edge to edge on large labels.** `.cell .inner{padding:.8mm}`
  is fixed, so on a 99×68mm label autofit pushes type to within 0.8mm of the die
  cut. Make it proportional — `padding: max(.8mm, 3%)` — so big labels get
  optical margin and survive a 1mm feed drift.

- **P2-33. Nothing says the work is saved, or where.** Everything persists to
  `localStorage` and there is no reassurance and no warning that clearing browser
  data loses it. After the first successful apply, a one-time quiet line under
  the count ("Энэ хөтөч дээр автоматаар хадгалагдана") would remove a real worry
  for someone laying out 102 labels, and would give Экспорт a reason to exist in
  the user's mind.

---

## The structural recommendation

The current structure is *nearly* right and one decision inside it is wrong.

**The wrong decision: the job picker is a global mode.** Everything painful in
P0-3, P1-6 and P2-27 follows from it. A label in this tool already *is* a
typed thing — it has `t`, `p`, `s`, `bc`, `qr`. The picker should read that type
off the selection and write only what it shows. Make that change and the picker
stops being a mystery toggle and becomes what its label promises: a description
of what you are making, which follows you as you move around the sheet.

**The structure I would ship**, in order down the column:

1. **Topbar = sheet-level facts.** Label size (one select, delete the sidebar
   duplicate), page indicator, zoom, `Цаас авах` visibly distinct, Хэвлэх. A "…"
   overflow for Export / Import / Хоослох.
2. **Sticky selection line.** One line, no chip wall. "Бүгд · Цуцлах" plus a
   "Шүүх ▾" popover that appears only once the sheet has content.
3. **Юу хийх вэ? — four real segments**, full width, driven by the selection.
4. **The fields for that job**, with **Сонгосонд бичих sticky at the bottom of
   the sidebar** so it is reachable from any scroll position. This is the action
   the user performs fifty times in a session; it should never be off screen.
5. **Өнгөт тэмдэг** — the swatch strip and paint toggle only. The five-line
   explanation becomes a one-liner plus a `?`.
6. **Лого.**
7. **Дэлгэрэнгүй** — one collapsed section holding bulk fill, type settings,
   fine-tune, presets. Two drawers become one.
8. **Print help leaves the sidebar entirely** and becomes the pre-print panel
   (P0-4) plus a `?` in the topbar.

That is five things in the column instead of five sections plus two sub-drawers,
and the one control that is used constantly (Apply) stops moving.

**On the sheet vs the sidebar.** The sheet does get the weight it deserves —
white paper, real shadow, achromatic chrome, colour reserved for the tags. That
core decision is right and it is the best thing about the design. What competes
with it is not the sidebar, it is the *stipple*: dashed outlines on 102 empty
cells, a 4px index number in each, hover tints, cursor ring, selection ring,
colour dots, row and column headers, and a page pill. Turning the index numbers
off by default (P2-21) and softening the cursor ring (P2-22) removes most of it
at no cost. The row/column headers earn their place — they look like a
spreadsheet, so clicking one to select a whole row is genuinely discoverable
without hint text, which is more than can be said for drag-select, paint mode,
copy/paste, double-click-to-edit or the printer offset.

**On discoverability of the powerful bits**, honestly ranked by what a
non-technical user will find unaided:
- Row/column header click — **found**, looks like a spreadsheet.
- Drag-select — **probably found**, it is named in the empty-sheet hint.
- Colour spotlight on swatch hover — **found by accident**, and it is delightful.
- Double-click to edit — **not found**, it is line 8 of a table in a collapsed
  drawer. Add a one-time inline nudge after the first successful apply.
- Multi-page — **not found**, the pill is small and below the fold (P1-17).
- Paint mode — **found but not understood** (P1-11); the button is visible, the
  state is not.
- Copy / paste / Ctrl+D — **not found**. The three `Хуулах / Буулгах / Бөглөх`
  buttons are visible but unlabelled as to what they do to a *block*; the fact
  that you can paste a spreadsheet column straight in is a genuinely great
  feature and is documented nowhere on screen. Put "Excel-ээс хуулж буулгаж
  болно" as the placeholder hint on the list textarea.
- Printer offset — **not found** (P1-9), and it is the one that decides whether
  the customer buys paper again.

**What reads as template-built rather than designed for this job:** the six
selection chips, the three unlabelled numeric rows, the always-on page bar, and
`Хуудас` as a settings dumping ground. Everything else — the money formatter
adding `₮` and thousands separators, the hand-written Code 128 and QR so the
file stays dependency-free, the Cyrillic-`х` accepted as a `×N` copies
multiplier, the thin-barcode warning, the offset clamped so label geometry stays
exact, the paper staying white in dark mode — is specific, considered work that
somebody clearly thought about. The gap between the quality of that thinking and
the four P0s above is the thing to close.
