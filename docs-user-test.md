# Peaklab Labels — hands-on usability test

**Tester persona:** owner of a small clothing shop in Ulaanbaatar. Two staff. Cheap inkjet in the back office, a pack of A4 self-adhesive sticker paper (plain full sheets, I cut them myself). Comfortable with Facebook and Excel. Never used design software. Reads Mongolian. Impatient — if something takes more than a minute I go back to writing price tags by hand.

**Tool:** https://kizaialt.github.io/label-sheet-printer/ (Mongolian UI)
**Method:** operated the live site in my own browser tab; four jobs attempted in order. Source consulted afterwards only to confirm *why* something behaved as it did.

---

## Summary

| Job | Result |
|---|---|
| 1. 30 price tags from Excel | **Completed** — good result, one wrong turn and one dead end on the way |
| 2. SKU labels with scannable barcode | **Failed on every path a shopkeeper would take.** Completed only via a 5-step chain I found by reasoning about the code, not the UI |
| 3. QR sticker for the Facebook page | **Completed** — fastest job of the four |
| 4. Reprint onto a part-used sheet | **Completed** — but nothing in the UI told me how; I worked it out |

Two findings dominate everything else:

1. **Bulk fill and auto-numbering silently ignore the barcode and QR settings.** There is no way to produce a sheet of *different* barcodes without a workaround nobody will find. Job 2 is the tool's own advertised use case and it does not work.
2. **Changing the label size permanently destroys labels.** I lost 7 of my 30 finished price tags with no warning and no undo.

---

## Job 1 — 30 price tags, name + price, readable from a metre

### Did I complete it?
Yes. And the end result is genuinely good — big bold price, thousands separator and ₮ added automatically, product name above it. This is better than anything I could do in Word.

### What I actually did
1. Clicked the first label on the sheet.
2. Typed `Цамц 89000` into **«Шошгоны текст»** and pressed **«Сонгосонд бичих»**. ← **wrong first thing.** I got a label with the name and the price at the *same small size*, no ₮, wrapped over two lines. Exactly what the brief asked me not to produce.
3. Only then did I read the tiny grey line **«Юу хийх вэ?»** above the four buttons and press **«Үнэ»**. A **«Үнэ»** field appeared.
4. Changed the label size from the default `32 × 15 mm` to `48 × 25 mm`. The default is far too small for a price you can read at a metre, and nothing suggests changing it.
5. Scrolled the sidebar down and opened the collapsed **«Олноор бөглөх»** section.
6. Pasted my 30 rows and pressed **«Жагсаалтаар бөглөх»** → toast: *«Эхлээд хуудсан дээрээс шошго сонгоорой»*. Nothing happened. **Dead end.**
7. Pressed **«Бүгд»**, then **«Жагсаалтаар бөглөх»** again → *«30 мөрөөр 30 шошго бөглөлөө»*. Done.

**≈9 interactions, one wrong output, one hard error.** It should be: paste, press one button. Two.

### The thing that annoyed me most
Step 3 was **completely wasted work.** Switching to «Үнэ» made no difference at all to the bulk fill — the paste box always treats the second column as a price whether or not you have pressed «Үнэ». I only found that out afterwards. So the tool made me fumble my first label, made me hunt for the mode picker to fix it, and then the fix turned out to be irrelevant to the actual job.

### The one that would have cost me a sheet of paper
The most natural thing for an Excel user is: select the two columns in Excel, Ctrl+C, click a label, Ctrl+V. **I tested this. It produces garbage:** the product name goes on one sticker and the bare price `89000` — no ₮, no comma, plain small text — goes on the **sticker next to it**. Four Excel rows produced eight ruined stickers. No warning, no toast saying "did you mean the list box?". On a small preview it looks plausible enough that I would have printed it.

The correct route (the «Олноор бөглөх» paste box) is behind a collapsed accordion two screens down the sidebar. The obvious route is wrong and silent.

### Other friction
- **«Сонгосонд бичих»** ("write to selection") is a strange verb for "apply". I hesitated over whether it would overwrite everything.
- The default `32 × 15 mm · хуудсанд 102` is an odd first impression — 102 tiny labels. Nobody's first job is 102 tiny labels.
- Long names (`Даашинз үдэшлэгийн`) auto-shrank and wrapped correctly. Good.
- Nothing tells me how much of the sheet 30 items will use, or that I have 10 spare positions, until after I fill.

---

## Job 2 — SKU labels with a scannable barcode

### Did I complete it?
**Not by any route I would have found on my own.** In real life I would have stopped after step 4 below and gone back to counting by hand.

### Where exactly I stopped and why

1. Pressed **«Бараа»** in the job picker. A checkbox **«Баркод»** appeared, unchecked, with the hint *«Шошгоны текстийг Code 128 баркод болгож хэвлэнэ»*.
   - First hesitation: **«Бараа»** means "goods". I would never guess that's where the barcode lives. I looked for the word "Баркод" in the sidebar first. If the button said «Баркод» I'd have found it instantly.
   - Second hesitation: if I've told the tool the job is "goods", why do I *also* have to tick a barcode box? What else would «Бараа» be for?
2. Ticked **«Баркод»**. Pressed **«Бүгд»**. Filled prefix/start/pad and pressed **«Дугаарлах»**.
3. **Result: 40 labels reading `001`–`040` in enormous type. No barcodes at all.** The Баркод box was still ticked. No warning, no toast, nothing.
   - I confirmed in the source: **«Дугаарлах» never writes the barcode flag.** Neither does **«Жагсаалтаар бөглөх»**. Both bulk tools write only text (and price), and silently drop the barcode tick and the QR field. So *neither* bulk path can ever produce a barcode.
4. Tried selecting a single label and applying a barcode to it. **The «Баркод» tick had silently un-ticked itself** — the checkbox re-syncs to whatever the selected label already has, so ticking it *before* selecting throws it away. I got `SKU-001` as plain text.
   - This is the point at which I would have given up. Nothing on screen explains it.
5. Re-ticked «Баркод» *after* selecting, pressed «Сонгосонд бичих». Got a barcode — but with the SKU text above it rendered at roughly **2pt**: a grey smudge about 0.8 mm tall, with 13 mm of empty white space below it. A barcode with no readable code under it is useless for stock counting the moment the scanner misreads.
6. Typed `9` into **«Хэмжээ (pt)»** under «Үсгийн тохиргоо». **Nothing happened.** No message. The reason is that **«Автомат»** was on and overrides the pt box; the hint text explains what Автомат does but never says it makes the pt field inert.
7. Turned **«Автомат»** off. Now the label looked right: readable `SKU-001` above a clean barcode.
8. Only then did I find a way to do all 40, by reasoning about which operations preserve the barcode flag:
   **«Хуулах» → «Бүгд» → «Буулгах» → «Дугаарлах»** — copy the one good label over the whole sheet, then renumber. Numbering rewrites the text but leaves the barcode flag alone, so all 40 barcodes come out different and correct.

That chain works and the result is good (40 distinct scannable Code 128 barcodes, ~0.41 mm module width, comfortably above the tool's own thin-barcode threshold). **But there is nothing anywhere in the interface that hints at it.** You have to know that one button preserves a property another button destroys.

**≈15 interactions plus a leap of logic**, versus what it should be: paste the SKU list, tick Баркод, one button.

### "This is not for me" moment
Step 3. I did the obvious thing, the tool said nothing, and gave me 40 labels that were confidently, completely wrong. That is worse than an error message. If I hadn't been looking closely I'd have printed a sheet of useless number stickers.

---

## Job 3 — QR sticker for the shop's Facebook page

### Did I complete it?
Yes — the smoothest job. Added a page, pressed **«QR»**, pressed **«Бүгд»**, typed the shop name in «Шошгоны текст» and the Facebook URL in **«QR-д хийх холбоос, текст»**, pressed «Сонгосонд бичих». 40 identical stickers, shop name above a clean QR. ~6 clicks.

The QR renders at 15.7 mm with a 2-module quiet zone — fine for a phone camera. The placeholder `https://facebook.com/...` told me immediately what to put there, which is why this job went well: the field is self-explanatory in a way «Бараа» is not.

### Where I hesitated / got caught
- **I wanted a bigger sticker.** A packaging sticker at 48 × 25 mm is small; I tried `70 × 37 mm`. **This destroyed Job 1.** See below — I had to change back and keep the small size.
- **Left-over text.** The «Шошгоны текст» box still held `SKU-001` from Job 2. If I hadn't cleared it, all 40 packaging stickers would have said SKU-001. The text box does not clear when you change pages or switch jobs.
- **«QR-д хийх холбоос, текст»** — good label. But I only knew to look under **«QR»** because the job picker happens to use the same three letters everyone knows. That's luck, not design.

### The destructive bug, in detail
Label size is a property of the **whole document**, not the page. Switching from `48 × 25` (4 columns) to `70 × 37` (3 columns) reflowed page 1 from 4 columns to 3 — and **every label that had been in column D vanished.** Seven finished price tags: Свитер хар, Жинс эмэгтэй, Бээлий арьсан, Хантааз, Гутал намрын, Юбка богино, Худи саарал.

- No confirmation dialog. No toast. No "7 labels will not fit".
- **Switching back to 48 × 25 did not restore them.** They are gone.
- The remaining 23 were left packed three-to-a-row on a four-column grid, i.e. in the wrong positions.
- **Ctrl+Z does not cover it.** I pressed undo twice; it skipped the size change entirely, jumped me to a different page, and undid two pieces of work I *wanted* to keep (my SKU numbering and an added page). The destroyed price tags stayed destroyed.

For a shop owner this is the difference between trusting the tool and not. I would not leave a half-finished sheet in this thing overnight.

---

## Job 4 — reprinting onto a part-used sheet (top two rows peeled)

### Did I complete it?
Yes. 10 new price tags placed from row 3 onwards, rows 1 and 2 left empty.

### What I did
1. Added a page, dragged a box over rows 1–2 to select the 8 peeled positions.
2. Clicked the red swatch under **«Өнгөт тэмдэг»** — red dots appeared on those 8.
3. Clicked **«Эсрэгээр»** (invert) → the remaining 32 were selected.
4. Pasted 10 rows into the list box → **«Жагсаалтаар бөглөх»** → *«10 мөрөөр 10 шошго бөглөлөө»*. The 10 landed neatly starting at A3; the rest stayed blank.

That is a good outcome, and the fact that a 10-row list into a 32-label selection fills exactly 10 and stops is exactly right.

### But
- **Nothing in the UI told me this was the workflow.** The colour-tag help text says *«Өнгөн дээр хулгана аваачихад тухайн өнгөөр тэмдэглэсэн шошго тодорно… доорх сонголтыг асаагаагүй бол хэвлэгдэхгүй»* — it explains what a dot *is*, never that this is the feature for a part-used sheet. The words "used", "peeled", "part sheet" appear nowhere. I found the route by guessing that «Эсрэгээр» meant "invert".
- **I did more work than I needed to.** Afterwards I realised I could have just dragged a selection starting at row 3 and skipped the dots entirely. The dot feature is presented as though it's the answer, and for this job it is optional ceremony.
- **Drag-select is fiddly.** My first drag got 6 labels instead of 8 — the rightmost column is hard to hit and dragging past the sheet edge clamps the selection instead of extending it. There is no way to type "labels 9 to 18" or "start at row 3".
- **The tool does not remember what I printed.** Next week I will again have to look at the physical sheet, count the missing rows and re-mark them by hand. The one thing a computer should be doing here — remembering which positions are used up — it doesn't do. Printing a sheet should offer to mark those positions as consumed.
- Cut lines still print across the peeled rows. Minor ink waste, but on an inkjet with a part-used sheet it looks messy.

---

## The job picker: is it discoverable, and does it help?

**Verdict: it is visible but not self-explanatory, it makes the panel simpler without making the job simpler, and worst of all it hides fields whose data then gets silently erased.**

### Discoverable — mostly yes
The four buttons sit in the first open section of the sidebar, on screen at load. I noticed them within about twenty seconds. But **I typed my price into the text box first anyway**, because the label above them is «Юу хийх вэ?» in tiny grey micro-type and the buttons themselves read as a style toggle, not as a mode switch. The first thing I did was exactly what you predicted: name and price both into «Шошгоны текст», and I got a small unformatted price for my trouble.

### Does it make the tool simpler?
Superficially yes — one relevant field at a time instead of four fields stacked. The panel is calm. That is a real improvement over "too many options".

But underneath, **it does almost nothing.** All the picker does is show or hide three rows. It doesn't set the label size, doesn't preselect a sensible layout, doesn't change what the bulk tools do. So:

- «Үнэ» is **redundant** for the main workflow — the Excel paste box formats prices as prices whether or not you ever press it.
- «Бараа» is **misnamed**. "Goods" is not a word I associate with barcodes; I looked for «Баркод» first. And after pressing it you still have to tick a separate barcode box, which then un-ticks itself when you select a label.
- «QR» is the only one that works the way the picker implies.

### Where it actively hurts — this is the important part
Because the picker *hides* fields rather than disabling them, **data you cannot see gets destroyed without warning:**

> I filled 30 labels with prices in «Үнэ» mode. I then pressed **«Энгийн»** — the Үнэ field disappeared but the prices still showed on the labels. I selected a label and pressed **«Сонгосонд бичих»**. **The price was wiped.**

The write button always writes *all four* fields, using empty values for whichever ones the current mode has hidden. So «Энгийн» silently erases prices, barcodes and QR codes; «Үнэ» silently erases barcodes and QR codes. Undo recovers it *if* you notice. On a 40-label sheet you would not notice.

**A mode picker that hides a field must not let a later action blank that field.** Either keep hidden values, or say "this will remove the price from 30 labels".

### One-line answer
The picker is discoverable enough and does reduce clutter, but it hides the barcode where I can't find it, it's irrelevant to the bulk workflow that actually does the work, and it turns invisible fields into silent data loss — so it simplifies the *panel*, not the *job*.

---

## Other things I hit

- **Печать guidance is buried.** The one piece of advice that decides whether the whole sheet aligns — *«захын зай None, масштаб 100%, «fit to page» сонгохгүй»* — lives in a collapsed section called **«Хэвлэх ба гарын товчлол»** at the very bottom of the sidebar. I pressed the big black **«Хэвлэх»** button in the top right and went straight to the browser print dialog with no reminder. First print on real sticker paper = one wasted sheet.
- It also refers to **"Background graphics"** — an English string — which a Mongolian-speaking shop owner has to find inside Chrome's own "More settings".
- **«Экспорт» produces a JSON file, not a PDF.** If I want to take the job to a print shop on a USB stick, I can't. This surprised me — Export means "give me a file I can use", and this file is only usable by this website.
- **Only one document exists.** Everything lives in one browser save. There is no "my price tags" / "my SKU sheet" library. Multiple pages help, but if I clear or re-size, that's it.
- **`x3` copies works, but the instructions are misleading.** The hint says *«Мөрийн төгсгөлд «x3» гэж бичвэл 3 ширхэг хэвлэнэ»* — "write x3 at the end of the row". I wrote `Цамц цагаан ⇥ 89000 ⇥ S x3` and got **one** label with the note "S x3". The multiplier has to be its own tab-separated column. (Credit where it's due: the Cyrillic `х3` works as well as Latin `x3` — someone thought about a Mongolian keyboard.)
- **The third line («Нэмэлт мөр») prints very small.** For S / M / L on a clothing tag it's too small to read across the shop.
- **On a phone (375 px) the label-size dropdown in the top bar collapses to an empty box** showing only an arrow — you can't read or pick a size from it. The duplicate dropdown further down still works, so it isn't fatal, but the top bar looks broken.
- Double-click to edit in place works, though the inline editor renders the text tiny in the corner rather than as it will print.
- Undo works within a page, but **it jumps you to another page without saying so** and it does not cover grid changes.

---

## What is still missing for ordinary small-shop work — ranked by how much it would change my day

1. **Barcodes and QR codes from a list.** «Дугаарлах» and «Жагсаалтаар бөглөх» must honour the Баркод tick and the QR field. Right now a sheet of different barcodes is impossible by any route a shopkeeper would find — which means Job 2, the reason I would install a label tool at all, does not work. Fixing this is worth more than everything else on this list combined.
2. **Make Ctrl+V from Excel do the right thing.** Two columns should become name + big price, not two ruined stickers. Third column = note, fourth = quantity. At minimum, detect a two-column paste and ask: "Is this name and price? Use the list box." This is the single most likely first action of every user who has their data in Excel.
3. **Stop the label-size change destroying labels.** Warn before dropping anything ("7 labels won't fit — continue?"), keep the size per page rather than per document, and put the change on the undo stack.
4. **Saved, named jobs.** "Үнийн шошго", "SKU", "Facebook QR" — reopen and reprint next week without rebuilding. Today there is one anonymous save and a JSON file I don't know what to do with.
5. **PDF export.** So I can take the file to a print shop, send it to my staff on Messenger, or reprint next month without the browser. «Экспорт» giving JSON is not what that word promises.
6. **A print checklist on the Хэвлэх button**, in Mongolian, with pictures of where those settings are in Chrome — plus a "print one test row on plain paper" button. Right now the advice that saves the sheet is hidden at the bottom of the sidebar and I never saw it before printing.
7. **Remember what I already printed.** After printing, offer to mark those positions used, so next week the part-used sheet is already set up and I select "the free ones" in one click. Also let me type "start at label 9" instead of dragging.
8. **Name + price + barcode on one label, laid out sensibly.** Today adding a barcode collapses the text to ~2pt with half the label empty, and the fix ("Автомат" off) is invisible. A clothing tag wants name, price *and* barcode together.
9. **Rename «Бараа» to «Баркод»**, and make picking it actually turn the barcode on. Also make «Хэмжээ (pt)» say why it's doing nothing while «Автомат» is on.
10. **A bigger note line for sizes (S/M/L)** and a way to say "3 of each size" without hand-typing a separate x-column.

### And one thing that is genuinely good
The price rendering. `89,000₮` big, bold, auto-fitted, comma and tugrik added for me from a bare `89000` in Excel — that is the exact thing I would otherwise be writing by hand 30 times, and the tool does it in one button. When the paste box works, this tool is better than what I do now. It just hides that button two screens down and lets me do the wrong thing first.
