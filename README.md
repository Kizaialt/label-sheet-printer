# Label Sheet Printer

A single-file browser tool for printing SKU text and PNG logos onto sticker/label sheets at exact
millimetre positions. No install, no server, no dependencies — open the page and print.

**Live:** https://kizaialt.github.io/label-sheet-printer/

## The sheet it ships with

| | |
|---|---|
| Page | A4, 210 × 297 mm |
| Grid | 6 columns × 17 rows = **102 labels** |
| Label | **32 × 15 mm** |
| Margins | 4 mm left / right, 5 mm top / bottom |
| Gaps | 2 mm between labels, both directions |
| Corner | 1.5 mm radius |

Other layouts are in the preset dropdown, and **Sheet setup** lets you type any geometry and save it
as your own preset (stored in the browser).

## What it does

- **Pick labels** — click, drag a box, `Ctrl`+click to toggle, `Shift`+click for a block, or click a
  row/column header to take the whole line.
- **Text** — type it once and apply to every selected label; multi-line supported. Text that is too
  long shrinks itself to fit.
- **Auto-number** — `SKU-001, SKU-002, …` across the selection, with prefix, suffix and zero padding.
- **Paste a list** — one SKU per line, filled in reading order, optionally repeating.
- **Logo** — drop in a PNG/JPG/SVG and place it alone, above the text, or beside it. Images are
  downscaled and kept in the browser.
- **Colour tags** — mark labels with a small colour dot to track which stickers on a part-used
  sheet are already gone. Ten colours, each a colour-to-white gradient bead that reads at a glance
  without covering the label. Hover a colour to spotlight every label carrying it; each swatch shows
  how many it has. Dots are screen-only unless you ask for them. *Paint mode* tags by dragging.
- **Keyboard** — arrows move the cursor, Shift extends, Space toggles, Del clears, Ctrl+P prints.
- **Export / Import** — the whole sheet, images included, as a JSON file.

Everything autosaves to `localStorage`, so the sheet is still there next time you open the page.

## Printing

Browser print dialog → **Paper A4**, **Margins: None**, **Scale: 100%** (not "fit to page"),
headers/footers off. Tick *Print outlines* in Sheet setup for a first alignment test on plain paper:
hold it against the sticker sheet up to the light and nudge the margins by a few tenths of a mm if
your printer runs off-centre.

## Offline use

Save `index.html` anywhere and open it — it is fully self-contained.
