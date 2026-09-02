# Label Sheet Printer

A single-file browser tool for printing SKU text and PNG logos onto sticker/label sheets at exact
millimetre positions. No install, no server, no dependencies — open the page and print.

**Live:** https://kizaialt.github.io/label-sheet-printer/

## Sheet sizes

| | |
|---|---|
| Page | A4, 210 × 297 mm |
| Grid | 6 columns × 17 rows = **102 labels** |
| Label | **32 × 15 mm** |
| Margins | 4 mm left / right, 5 mm top / bottom |
| Gaps | 2 mm between labels, both directions |
| Corner | 1.5 mm radius |

Paper is always A4. You pick the **label size** from the toolbar dropdown; nine standard A4 sticker
layouts ship with it, from 32 x 15 mm (102 up) to a full uncut sheet, and every one is computed to
land exactly on the page. Anything unusual can be fine-tuned by hand and saved as your own size.

**Printer offset** shifts the whole grid a fraction of a millimetre to match a printer that feeds
off-centre, without changing the label dimensions. Set it once, and it applies to every size.

## What it does

- **Pick labels** — click, drag a box, `Ctrl`+click to toggle, `Shift`+click for a block, or click a
  row/column header to take the whole line.
- **Text** — type it once and apply to every selected label; multi-line supported. Text that is too
  long shrinks itself to fit.
- **Auto-number** — `SKU-001, SKU-002, …` across the selection, with prefix, suffix and zero padding.
- **Paste a list** — one SKU per line, filled in reading order, optionally repeating.
- **Logo** — drop in a PNG/JPG/SVG. A label shows whatever it holds: text alone, logo alone, or
  both together (choose whether the logo sits above, below, left or right of the text). Logos and
  text always print. Images are downscaled and kept in the browser.
- **Colour tags** — mark labels with a small dot to track which stickers on a part-used sheet are
  already gone. Ten flat colours by default; flip the switch to **Gradient** and the dot fades from
  that colour to white, so you get twenty distinguishable tags without covering the label. Hover a
  colour to spotlight every label carrying it; each swatch shows how many it has. Dots are
  screen-only unless you tick *Print the dots too*. *Paint mode* tags by dragging.
- **Keyboard** — arrows move the cursor, Shift extends, Space toggles, Del clears, Ctrl+P prints.
- **Light or dark** — light by default, since you are usually comparing the screen against a real
  sheet under office light. The toggle is in the toolbar and is remembered.
- **Export / Import** — the whole sheet, images included, as a JSON file.

Everything autosaves to `localStorage`, so the sheet is still there next time you open the page.

## Printing

Browser print dialog → **Paper A4**, **Margins: None**, **Scale: 100%** (not "fit to page"),
headers/footers off. Tick *Print outlines* in Sheet setup for a first alignment test on plain paper:
hold it against the sticker sheet up to the light and nudge the margins by a few tenths of a mm if
your printer runs off-centre.

## Offline use

Save `index.html` anywhere and open it — it is fully self-contained.

## Design notes

The chrome is deliberately achromatic. The ten tag colours are the only saturated things on screen,
so they read as data rather than decoration, and the sheet is always the brightest element on the
page. Type runs on three sizes (10.5 / 13.5 / 17px) and the system UI face; there are no webfonts,
no gradients outside the tag dots, and one filled button per screen.
