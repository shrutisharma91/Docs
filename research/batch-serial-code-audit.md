# Batch, serial (IMEI), barcode and label: a code audit of what exists and what a full build must touch

**Status:** research, docs only. First written 2026-10-22 against `83d9876` plus the then-uncommitted parity wave 3; **re-based and re-verified on 2026-10-23 against `main` at `7074445`**
(`/Users/Akshay/tallyreplica`). Wave 3 is no longer in flight: it was committed as `2cee555` (2026-10-22) and its web app and backend, migration `008` included, were deployed on 2026-10-23
(`docs/HANDOFF.md:48,546`). The MCP server is committed (`d7846ce`) and **not** deployed (`HANDOFF.md:671`). The custom-fields and Back-control build in the worktree
`/Users/Akshay/tallyreplica/.wt-cf` (branch `cf-back-nav`, `f446674`, "WIP checkpoint 2 ... unreviewed") was read, never written. Nothing in the repo was changed except this file.

**The owner's request, in one line:** batch tracking in Tally is weak and Busy's customers like its batch management; mobile traders track IMEI; label printing and barcode scanning are
needed and Tally is weak at them too; build these well.

**What this document is.** A map of the ground a design has to stand on. For every area the brief named, it says what the code does today (with `file:line`), what that means for a build, and
what I measured. It ends with a ranked risk list and a file list grouped into lanes that can run in parallel. It does **not** choose the design. Where the code forces a choice I say which way
it leans and why, tagged `[J]`. Section numbers and risk numbers are unchanged from the first version on purpose: three sibling documents cite them
(`docs/research/batch-serial-tracking.md`, `tracking-verticals-india.md`, `pos-restaurant-market.md`); the last also cites this file **by line** (`:109`), which no revision can keep: the row it means is "Batch name + expiry on a line" in section 2.

### What changed in this revision (read this if you read the first version)

1. **Every line number was re-anchored** to `7074445`. About a third had drifted (wave 3 kept changing `VoucherScreen.tsx`, `db.ts`, `sync.ts`, `syncGuard.ts` between the first read and its commit).
2. **One finding was wrong and is corrected (section 3.2.5).** The first version blamed `Array.shift` for the FIFO cliff. LIFO, which pops from the end, is just as slow: the quadratic term is
   `lotsValue`, a full `reduce` over every open lot on **every** inward and outward (`src/core/valuation.ts:305,360,392`). Replacing `shift` would not fix it.
3. **The custom-fields draft the first version audited was withdrawn in review (section 4.2).** The CF author's third pass dropped "Physical Stock resolves against the (godown, variant) balance"
   and replaced it with a **count group** (`countPart` flag, one merged movement in `stockMovements`). That removes part of the clash this audit warned about and gives the batch fix a ready
   mechanism; it also shows a **fourth** place that resolves a Physical Stock count (`ui/inventory/ageing.tsx:74-78`), which the first version missed.
4. **The worktree has moved.** `InventoryLine.countPart` and `CustomFieldDef.tracksStock` already exist there (types only); the Back control's `Box` `onClose` is built (the batch dialog passes it);
   `backend/src/sync.js` is split into `pushRun.js` / `histories.js` / `mismatch.js` / `restWrite.js`. The CF-9 **engine** is still not written (`.wt-cf/src/core/inventory.ts` differs from main by one additive field).
5. **New coupling: the MCP server** (section 4.5). It pins SHA-256 hashes of 52 `src/` files, `inventory.ts`, `valuation.ts`, `openingStock.ts` and `types.ts` among them, and classifies every key of
   the loaded interfaces of `src/types.ts`. Any lane that edits those files owes the MCP gate a re-lock.
6. **New facts found while re-reading:** the negative-stock warning does not run on an alteration (`VoucherScreen.tsx:2491`); `setBatchFor` has two openers, not three; an e-way bill warns above
   250 lines and an e-invoice refuses above 1,000 (so one-line-per-unit has a ceiling); an invoice row is atomic, so a long serial list is one unbreakable row (3.6); a scanned EAN or IMEI in the quantity
   field becomes a 13- or 15-digit quantity (probe 5); the server's change log has no index on payload, so a serial check needs its own table (3.9); a new mismatch kind also touches the MCP formatter (3.10).
7. **The probes were re-run on the current engine** and reproduce (Appendix A). The payload-size probe was redone with realistic UUID ids and reads about twice the first figure.

---

## 0. How to read this

**Evidence tags.**

| Tag | Meaning |
|---|---|
| `[C]` | read in this repo; `file:line`, at `7074445` unless a worktree path is given. Where a worktree file is cited it is `.wt-cf/...`; **its builder is editing it live** (two files moved by two lines while I wrote this), so a worktree line number is of 2026-10-23 and is always accompanied by a symbol. |
| `[P]` | a probe I ran against the real engine code (bundled with the repo's own `esbuild`, run with Node 25, scratch files only, no root `npm run` script). Inputs and outputs are in Appendix A. |
| `[W]` | a web page I fetched and read; `[Ws]` = a search summary only (the page was not read in full). |
| `[J]` | my judgement. |
| `[unverified]` | said without a source or a test; do not build on it without checking. |

All outside material is paraphrased. URLs are in Appendix B.

**Tree state.** `git status` at `7074445` shows only untracked docs (the CF and Back research) and an edited `.claude/settings.local.json`: the source tree is clean, so every `src/`, `backend/`, `electron/`
and `mcp/` line number below is a committed one and will not move until someone else commits. The task text that commissioned this revision still said "wave 3, ~160 uncommitted paths"; that was already
stale and is not true of the tree I read. The worktree `.wt-cf` had 36 uncommitted paths when I looked (`git status` there) on top of two checkpoints, and 249 files changed against its base `baa0c45`.

**CF-9 and the 2026-10-22 addendum.** I re-read `.wt-cf/docs/design/custom-fields-spec.md` (2,126 lines now; it was 1,410 when the first version of this audit began) at AD.3 (stock by field, CF24 to CF26), the CF-9a to
CF-9f slice rows, sections 9.3 and 16 to 17, and the status and review-log paragraphs. The file is **design**: its status paragraph says the whole spec had two adversarial passes and the addendum one (the third, whose findings are applied in place), and a second look at the addendum has not happened; I cite it by section (`AD.3.3`, `§9.3`), never by line,
because it is still being edited. Re-read it before building.

**Out of scope here.** Legal Metrology, FSSAI, Drugs and Cosmetics, HUID and the other regulatory research belong to the other documents of this workstream (`tracking-verticals-india.md`). This audit asserts
**no** statutory fact except two that I fetched or searched (the e-invoice batch / serial / barcode fields, section 3.8; and the accounting-standard rule on specific identification, section 3.2.5).

**What I did not have:** a live Busy, Marg or TallyPrime account; any scanner, label printer or phone; a browser run of the entry screen under a scan burst; a Postgres. Section 7 lists every consequence of that.

---

## 1. The findings on one page

Ranked list with fixes is in section 5; the lanes are in section 6. These are the facts a design must be built around.

1. **A batch is a free-text name on an inventory line, nothing more.** There is no batch master, no MRP, no manufacturing date that any screen can enter, no per-godown batch balance, no barcode on a
   batch. `InventoryLine.batchName` (`src/types.ts:953`), `byBatch: Map<name, {qty, expiry}>` per item (`src/core/inventory.ts:14`). [C]
2. **The voucher walk never asks for a batch.** The only ways in are Alt+B (`src/ui/VoucherScreen.tsx:3059-3062`) and the rail button (`:3409`); the item grid draws no batch column (`:754-760`); nothing requires a
   batch on a batch-tracked item; an outward line cannot pick from existing batches; no expiry check; no FEFO. Changing the item on a row leaves the old item's batch on it (`:2047-2075` never clears it). [C]
3. **The stock engine breaks its own batch arithmetic in ordinary cases.** I ran it again on `7074445`: opening stock is not in any batch (15 on hand, 5 in batches); a Physical Stock sheet with one line per
   batch turns 12 units into **7**; a godown transfer or a sales return with no batch on the receiving line drops that quantity out of every batch; `B-17` and `b-17` are two batches; a batch goes negative
   without a word; closing value follows the item's FIFO layers, not the batch that was sold. [P]
4. **Serial goods will freeze the app under FIFO or LIFO.** Both keep lots in an array and recompute the item's value by summing **every open lot on every movement** (`src/core/valuation.ts:305,360,392`);
   FIFO also removes the oldest lot with `Array.shift` (`:301`). One lot per unit is the natural shape of IMEI stock. With 125,000 one-unit lots `stockPositions` took **16 to 24 s** (two runs, a quiet and a
   busy machine; **LIFO, which pops, took 20 s**); 40,000 lots took 1.5 to 3 s; Avg. Cost took 27 ms on the same book. [P] The voucher screen re-runs `stockPositions` over every voucher whenever a voucher is saved
   or the date changes (`VoucherScreen.tsx:1371-1374`; 223 ms at 250k vouchers by the repo's own harness, `docs/PERF.md:178`). [C]
5. **Nothing for serials, barcodes, labels or scanning exists.** No field, no encoder (except QR), no scan handling, no printer path, no camera code (the greps are in section 2 and were re-run). The item picker
   matches name, alias and part number only (`VoucherScreen.tsx:2865-2868`); typing something that matches nothing and pressing Enter does nothing at all, silently (`:3223` with `:2047-2049`). [C]
6. **"Batch & Expiry Summary" is a stub that three entry points open.** `BatchSummary.tsx` renders "This screen is being built..."; it is reached by Gateway > Inventory Books > B (`MenuScreen.tsx:234`), Go To
   (`overlays.tsx:65`) and Ctrl+K (`CommandPalette.tsx:394`). The parity board nevertheless marks `batches` and `expiry-reports` **covered**, with `ours` text that says the opposite. [C]
7. **Where serials live decides everything else.** Only five code paths build an `InventoryLine` (section 3.4); a field on the line rides the existing undo, cancel, sync, backup, restore, Mismatch Report diff
   and REST paths for free. A separate serial table must re-do each of those by hand, and at 100k to 1M rows it multiplies the sync outbox (about 700 bytes a serial counting the table row and the outbox
   row, 3,334 push requests per million), the backup file, the pull-and-reload loop and the undo stack. [C][P]
8. **The same IMEI sold on two offline devices is invisible to every conflict rule that exists**, because the two sales are two different documents and the rules are per document
   (`backend/src/conflict.js:191-278`, `docKey` at `:173`). Only voucher *numbers* have a duplicate detector, and it is a client scan capped at 5,000 arrived vouchers (`src/sync/sync.ts:668-670`).
   Six writers bypass `saveVoucher` and one (REST) bypasses the app altogether. The server's policy is "accept and record, never refuse a push" (`conflict.js:15-16`). So the design is **detect, record,
   resolve**, not prevent. [C]
9. **Wave 3 built, and has now deployed, the pattern to copy** for the cloud half: an idempotent reserve, a provisional flag when the cloud cannot be reached, a claim when the device syncs, "never
   renumbered by the server", all keyed on a ledger whose primary key is the thing itself (`backend/src/numbering.js`, `backend/sql/008_voucher_series.sql`, `src/core/vchNumbering.ts`). It is live on the VM
   (BETA, authenticated cloud flows not exercised, `HANDOFF.md:620`). A serial is not a number (it cycles in and out), so the ledger is a lease, not a counter. [C][J]
10. **Printing has the pieces but not the path.** The app has a pure QR encoder with an independent decoder test (`src/core/qr.ts`), a millimetre-exact custom-page-size precedent (the cheque leaf,
    `src/ui/banking/chequeLeaf.tsx:166-172`), a print host, a PDF bridge and a bulk-print selection flow. It has **no** 1-D barcode encoder, **no** custom page size in Print Configuration
    (`PAPERS = A4, A5, Letter`, `src/core/printConfig.ts:32`), and **no** silent print or printer list in the Electron bridge (`electron/preload.cjs:6-43`, `exportBridge.cjs`). [C]
11. **The live site cannot use a camera.** It is HTTP only (`docs/HANDOFF.md:63,90`; `crypto.randomUUID` is polyfilled for exactly that reason, `src/polyfills.ts:1-13`); camera capture needs a secure
    context [Ws]. A keyboard-wedge (HID) scanner works on HTTP, which is the one scanner path that is not blocked. [C][Ws]
12. **A scan burst is dangerous on three kinds of focus**: a non-input screen (letters are hotkeys), the narration field (Enter raises Accept) and the "Accept ?" bar (Y, N and Enter answer it). [C]
13. **Every interchange path loses batch detail**: Tally import reads one opening figure per item and drops per-batch openings (`src/core/tallyImport.ts:959-967`); Tally export never writes expiry (it counts such lines as
    "not carried", `tallyExport.ts:731`); the e-invoice drops a batch name shorter than 3 characters without a warning and never carries a warranty date, serial number or barcode (`src/core/einvoice.ts:358-359`)
    although the schema has fields for all three [W]. [C]
14. **Four pieces of in-flight work want the same files, and one of them wants the same engine lines.** Wave 3 is no longer one of them. The custom-fields slices CF-2, CF-3 and CF-9c all edit the voucher grid, one
    after another. The Back control's `Box` `onClose` is built in the worktree, `BatchPrompt` included. **The CF-9 draft still changes the same `stockMovements` / `stockPositions` lines a batch fix must change**, but
    after its third review it does so through a **count group** that a batch count can reuse rather than through a re-keyed resolver (section 4.2). The **MCP server** pins `inventory.ts`, `valuation.ts`, `openingStock.ts`
    and `types.ts` by hash (section 4.5). [C]

---

## 2. Inventory of every touch point today

| Capability the owner named | Exists? | Where, and how much |
|---|---|---|
| Item "maintain in batches" switch | yes | `StockItem.batches?: boolean` `src/types.ts:520`; item master `src/ui/inventory/masters.tsx`: initial values `:352,361`, the toggle field `:426`, saved `:530` |
| Batch name + expiry on a line | yes, hidden | `InventoryLine.batchName/mfgDate/expiryDate` `types.ts:953-955`; `BatchPrompt` `VoucherScreen.tsx:883-914` (name and expiry only) |
| Manufacturing date | field only | `mfgDate` exists on the line and in the dirty probe (`:1618`); **no screen writes it**: `BatchPrompt.onDone(name, expiry)` `:886,4111` |
| Batch quantity per item | yes | `byBatch` in `stockPositions` `inventory.ts:141-146,156-161`; shown in Stock Query only (`stockquery.tsx:89-92,135-145`) |
| Batch report | **stub** | `src/ui/inventory/BatchSummary.tsx:1-4` (`ComingSoon`); allow-listed `scripts/assert/sidenav.assert.ts:131-133`; excluded from the Reports Center `src/ui/reportsCenter/catalogue.ts:205` |
| Expiry report / expired-sale block / FEFO | none | no code reads `expiryDate` except Stock Query's label, the print line and the e-invoice |
| Opening stock by batch | engine only | `OpeningStockAlloc.batches` `types.ts:483-486`, `core/openingStock.ts:41-64`; written only by Split Company Data `splitCompany.ts:547-552`; **no screen** (`masters.tsx:535-536`) |
| Batch on a printed invoice | yes | one small line "Batch: x · Exp. y", switchable (`print/invoiceModel.ts:617`, `printConfig.ts:103`) |
| Batch in the e-invoice | partial | `BchDtls {Nm, ExpDt}` only when 3 to 20 characters (`einvoice.ts:61,358-359,460,717-719`) |
| Batch in Tally import / export | partial | name on voucher lines both ways; no opening batches in; no dates out (3.7) |
| Serial / IMEI / warranty | **none** | no field anywhere. `grep -rniE "\bimei\b\|barcode\|\bgtin\b\|ean-?13\|ean-?8\|upc-?a\|code ?128\|code ?39\|datamatrix\|\bhuid\b"` over `src electron backend/src admin/src mcp/src scripts/seed` returns nothing (re-run 2026-10-23); the word "serial" appears only for invoice serial numbers and Excel date serials |
| Barcode field / search / generate / scan | **none** | `alias` and `partNo` are the only code-like fields (`types.ts:492-493`) |
| Label printing | **none** | `grep -rniE "sticker\|zebra\|tspl\|zpl\|esc/?pos\|thermal\|label (sheet\|printer\|print)\|print ?label"` over `src electron` returns nothing |
| MRP | **none** | parity row `mrp-pricing` is `missing` (`docs/parity-data.json`) |
| QR encoder | yes | `src/core/qr.ts` (ISO 18004, v1-40, L and M, an independent decoder test in `scripts/assert/einvoice-qr.assert.ts` per its header), `src/ui/print/QrCode.tsx` |
| 1-D barcode encoder (Code 128, EAN-13) | none | `src/core` has `qr.ts` only; `package.json` has no barcode or PDF dependency (dexie, react, zustand and two fontsource packages) |
| Custom page size / label stock | partly | the cheque leaf only (`chequeLeaf.tsx:166-172`) |
| Silent / direct printing, printer list | none | `electron/preload.cjs:6-43` exposes backup, `pdf.save`, `share.open` only; `grep -rniE "getPrinters\|silent:\|deviceName\|webContents\.print\("` over `electron src` returns nothing |
| Camera / WebHID / Web USB / Web Serial | none | `grep -rniE "getUserMedia\|navigator\.(hid\|usb\|serial)\|BarcodeDetector\|mediaDevices"` over `src electron mcp/src` returns nothing; `electron/main.cjs` installs no permission handler |
| A test or screenshot recipe with a batch item | none | the sample company has no batch item (`src/core/demo.ts`); the only batch tests are engine-level: `scripts/assert/{perf,split,print,tallyimport,tallyexport,inventory}.assert.ts` and the fixtures `split-fixture.ts`, `scripts/perf/synth.ts` |
| MCP read and write of batches | partial | read keeps `batchName` and drops `mfgDate`/`expiryDate` (`mcp/src/data/fields.ts:98`); the write planner refuses a batch item (`mcp/src/write/buildVoucher.ts:281`) (4.5) |

**Parity board.** `docs/parity-data.json` rows `batches` ("Batches with mfg/expiry dates") and `expiry-reports` are `covered`; their `ours` fields read "No batch field on item or line." and "No batch/expiry
concept." `item-opening-multi` is `partial` ("no per-godown/batch split", also stale for the engine), `mrp-pricing` is `missing` (re-read 2026-10-23). The gotcha "a status field and its note must agree"
(`CLAUDE.md`, Phase 20-21) applies; the two `covered` rows are not covered in any sense a Busy user would accept (sections 3.1 to 3.5). Whoever builds this must correct the board in the same change, or the
250 / 58 / 73 count stays overstated. [C]

---

## 3. Audit by area

### 3.1 The data model (`src/types.ts`)

| Type | What it holds | Line |
|---|---|---|
| `StockItem` | `alias`, `partNo`, `unitId`, `valuationMethod`, `batches?: boolean`, `reorderLevel`, `prices[]`, `bom[]`, `openingQty/Rate/Value`, `openingAlloc?`, `udf?` | 488-527 |
| `InventoryLine` | `stockItemId`, `godownId`, `qty`, `rate`, `amount`, `dir?`, `discountPct?`, `altQty?`, `batchName?`, `mfgDate?`, `expiryDate?`, the GST snapshot (`gstRate`, `taxability`, `hsn`, `cessRate`, `gstOvr`), `desc?` | 944-973 |
| `OpeningStockAlloc` | `godowns[]` and `batches[] {name, qty, expiryDate?}` as **two independent lists** ("not a godown x batch grid", `core/openingStock.ts:9-11`) | 483-486 |
| `Godown` | `name`, `parent`, `reserved` (Main Location) | 454-460 |
| `Unit` | `name`, `decimals` (0 to 4), `uqc`, compound `baseUnitId/conversion` | 425-436 |
| `CompanyFeatures` | `inventory`, `gst`, `billwise`, `costCentres`, `tds`, `payroll`, `orders`, optional `discount`, `tcs`; **no batch or serial switch** | 91-101 |
| `VoucherType` | `printAfterSave?`, numbering fields, `entryLayout?` (wave 3) | 531-553 |
| `Voucher` | `inventory?: InventoryLine[]`, `optional`, `cancelled`, `numberStatus`, `dispatch`, `ewb` (wave 3); **no time of day, no terminal, no operator** (only the change-log row carries one, `db.ts:43-69`) | 991-1049 |

**Facts that shape a build**

- A batch has no identity beyond `(itemId, name)`. Anything per batch (MRP, mfg date, shelf-life, a barcode, a rate) would have nowhere to live. [C]
- The line has no place for a serial list, a warranty, an MRP, or a barcode that was scanned. [C]
- `Unit.decimals` is used only to **format** a quantity (`fmtQty(q, decimals, ...)`, e.g. `VoucherScreen.tsx:787,2503,2876` and `stockquery.tsx:63-65`); entry accepts decimals regardless (`parseQty`
  carries up to 4). A serial-tracked item needs an integer rule that does not exist today. [C]
- **Quantity is always in the stocking unit.** `InventoryLine.altQty` is display-only ("qty is stocking-unit", `types.ts` `InventoryLine`), and a compound or alternate unit converts on entry ("5 box" on a Box-of-12 item, `VoucherScreen.tsx:3243-3263`, `Unit.baseUnitId/conversion` `types.ts:434-435`). A pharma pack ladder (strip, box, case) therefore exists only as a unit conversion, never as a per-batch pack size or a loose-unit balance. [C]
- `parseQty` evaluates arithmetic ("Indian shorthand" and expressions) and shows a live "= value" hint on every keystroke (`VoucherScreen.tsx:150-153,745-746`; `core/money.ts:162-178`). A scan that lands in the
  quantity field is read as a number, with no sanity limit. [C]
- The voucher dirty probe lists every line field by name (`VoucherScreen.tsx:1614-1621`). A new line field that is not added there makes typed work invisible to the navigation guard, which decides "Leave this
  entry?" from that probe (Phase 26 gotcha, `CLAUDE.md`). [C]
- **Three suites make a new `types.ts` key a decision, in three places.** `scripts/assert/master-resave.assert.ts` classifies every `StockItem` and `Ledger` key (`:12-14,100-118`; a new field needs a form control
  or an `OFF_FORM` entry); the worktree adds a `voucher-resave` suite that does the same for `Voucher` and `InventoryLine` (`.wt-cf/scripts/assert/voucher-resave.assert.ts`, built, unreviewed; spec `§9.1`); and the
  MCP server's `mcp/test/fields.test.ts:22-30` fails on any key of a loaded interface that is in neither the `keep` nor the `drop` list of `mcp/src/data/fields.ts` (4.5). [C]

### 3.2 The stock engine (`src/core/inventory.ts`, `valuation.ts`)

#### 3.2.1 How batches are keyed

`stockPositions` walks every movement in date order and keeps, per item, a `ValState` (`valuation.ts:211-235`). `byBatch` is created only when `it.batches` (`valuation.ts:248`) and updated on each line that
names a batch (`inventory.ts:141-146` inward, `:156-161` outward). The key is the raw `batchName` string. The map is per **item**, not per godown: the engine keeps `byGodown` and `byBatch` as separate maps,
so "how much of batch B1 is in godown G2" cannot be answered from a position (`openingStock.ts:9-11`, `inventory.ts:14`). [C]

**One generator feeds everything.** `stockMovements(vouchers, upto, from)` (`inventory.ts:33-48`) is the only place movements are made: it skips `optional` vouchers (`:39`), asks `lineDir` for each line's direction
(`:26-31`), numbers them in voucher-array order and sorts by `(date, seq)` (`:46`). The array is already sorted by `(date, number)` by the store (`store.ts:295-296`), so **within one date the voucher number
decides which movement is first**: that is the tie-break a serial "in stock as at" rule inherits. The movement carries the whole `line` and the whole `voucher`, so anything a line holds (a batch, a serial list)
is available to every reader without a signature change. `StockMove` (`:61-73`) is what `observe` hears; it carries the whole `line` but has no field of its own for a batch cell or a serial. The engine's callers are 22 files:
`core/{dashIndex,splitCompany,stockSummary,tallyCompare,tallyCompareRead,valuation,valuationCautions,valuationCompare}.ts`, `ui/VoucherScreen.tsx`, `ui/admin/{BulkEntry,Exceptions}.tsx`,
`ui/dash/widgets/{env,types}.ts`, `ui/inventory/{ageing,godownstock,manufacturing,movement,reorder,reports,stockquery}.tsx`, `ui/reports/profitability.tsx` (grep, 2026-10-23). [C]

#### 3.2.2 What the probes showed `[P]` (Appendix A, probes 1 and 3; re-run on `7074445`, identical)

| Case | Result | Why |
|---|---|---|
| 10 opening (no batch) + 5 purchased as B1 | item qty 15, batches sum to 5 | opening stock is booked to Main Location only (`newValState` `valuation.ts:247`), never into `byBatch` unless `openingAlloc` exists (`openingStock.ts:57-64`) |
| Physical Stock sheet, one line per batch (B1 = 5, B2 = 7; book 5 and 7) | item qty becomes **7** (should stay 12); B1 -2, B2 9 | the count is resolved against the **item's godown balance**, line by line (`inventory.ts:119-127`): line 1 sees 12, books -7; line 2 sees 5, books +2 |
| Physical Stock listing only B1 = 5 | item qty becomes 5; B2's 7 vanish from the item total | same |
| Stock Journal transfer, source line names B1, receiving line has no batch | batch sum 3 against item qty 5 | the receiving leg adds nothing to `byBatch` (`:141`), the source leg takes 2 from B1 |
| Credit note (sales return) with no batch | item qty 4, batch sum 3 | same reason |
| `B-17` purchased, `b-17` sold | two batches: `B-17` +5, `b-17` -5 | no normalisation |
| Sell 5 of B1 which holds 2 | B1 = -3, no message | no engine or screen check |
| Same batch bought twice with two expiry dates | last write wins | `inventory.ts:143` |
| FIFO: buy B1 at 100, buy B2 at 150, sell 10 of B2 | issued at **B1's** cost (-1,000); stock left is 10 of B1 but the closing value is 1,500 (B2's cost) | FIFO layers are per item (`valuation.ts:110-112` says per-godown/batch lots are "not modelled") |

None of these is covered by a test: `scripts/assert/inventory.assert.ts:375-382` checks that godown balances add up to the quantity, **not** that batch balances do, and nothing asserts a batch balance after a
Physical Stock voucher. [C]

**Consequence.** A batch design has to say what the invariant is. The cheapest one that also fixes the table above: every quantity of a batch-tracked item sits in exactly one `(godown, batch)` cell, with an
explicit "no batch" cell for opening and unbatched movements, and `Σ cells = item qty`, asserted. Physical Stock must count by cell. Transfers and returns must carry the batch on both legs (or refuse). [J]

#### 3.2.3 Physical Stock, transfers, Stock Journal

- Direction per base (`types.ts:26-44`): Sales out, Purchase in, DebitNote out, CreditNote in, DeliveryNote out, ReceiptNote in, StockJournal both (per line `dir`), PhysicalStock both (resolved to a delta).
  Orders and memoranda never move stock. `optional` vouchers are skipped (`inventory.ts:39`). [C]
- A godown transfer is a Stock Journal with source and destination sections ("Source (Consumption)", "Destination (Production)", `VoucherScreen.tsx:3725-3739`); a receiving line with no rate is valued at what
  the same voucher issued the item at (`inventory.ts:97-103,128-137`; the screen pre-fills the same rate at `VoucherScreen.tsx:2057-2063`). Nothing there knows about batches. [C]
- A **Delivery Note followed by a Sales invoice** both move stock out (`types.ts:31,37`); the app does not link them ("tracking-number links" are listed as not modelled, `valuation.ts:110-112`; only orders
  are netted, `pendingOrders` `inventory.ts:315-338`). For serials this means the same IMEI legitimately appears on two outward vouchers unless a link is built. [C][J]
- **Four places resolve a Physical Stock count against the item's godown balance**, not three as the first version said: `stockPositions` (`inventory.ts:119-127`), `resolvedMovements` (`:202-208`),
  `stockItemRows` (`:268-274`) and the Stock Ageing report, which keeps its own running balance and its own lots in a component `useMemo` (`ui/inventory/ageing.tsx:74-88`). A fix that changes how a count is
  read must land in all four, or the Stock Summary, Stock Item Vouchers, Movement and Stock Ageing will disagree. The CF-9 draft found the same four independently (4.2). [C]

#### 3.2.4 Manufacturing

`ui/inventory/manufacturing.tsx` builds a Stock Journal with the components as `dir: 'out'` lines and the finished good as one `dir: 'in'` line, all at Main Location, with **no batch and no serial**
(`:170-177`). The BOM is `StockItem.bom: {componentId, qty}[]` (`types.ts:523`) with no batch inheritance. A batch-tracked finished good therefore never gets a batch from production, and a batch-tracked
component is consumed with none. [C]

#### 3.2.5 Costing for serial goods

- The costing methods are Avg. Cost (default for new items), Monthly Avg., FIFO, LIFO, Last Purchase, Std., Zero and the earlier moving average (`valuation.ts:117-151`). There is **no
  specific-identification method** (each unit carried at its own cost). [C]
- The in-repo comment cites AS 2 as allowing FIFO or weighted average (`valuation.ts:84-85`). The standard also has a separate rule: items that are not ordinarily interchangeable, and goods segregated for
  specific projects, are costed by specific identification of their individual costs; for large numbers of interchangeable items specific identification is inappropriate because profit could be influenced by
  choosing which units remain (AS 2 paragraphs 14 to 16 per the search summary). [Ws] Whether a handset retailer's phones count as "not ordinarily interchangeable" is an accounting judgement the owner's CA
  decides; the build should offer the method, not impose it. [J]
- **The FIFO and LIFO structure is quadratic, and `shift` is not the main cause (correction of the first version).** After every inward and every outward, FIFO and LIFO call `lotsValue(lots)`, a `reduce` over
  **every open lot** (`valuation.ts:305`, called at `:360` and `:392`); FIFO also removes the front lot with `lots.shift()` (`:301`), LIFO with `pop()`. Cost per movement is therefore proportional to the number of
  open lots, whichever end is consumed. [C][P] probe 2 (one item, 1 unit per line, `stockPositions` wall time, this machine, two runs on different load):

| Book (one item, 1 unit per voucher) | `stockPositions` time |
|---|---|
| Avg. Cost, alternating buy and sell, 250,000 vouchers | 44 ms |
| Avg. Cost, buy 125,000 then sell 125,000 | 27 ms |
| FIFO, alternating buy and sell, 250,000 vouchers (one lot open at a time) | 26 ms |
| FIFO, 10,000 lots open (20,000 vouchers) | 0.4 s |
| FIFO, 20,000 lots open | 0.4 to 1.2 s |
| FIFO, 40,000 lots open | 1.5 to 3.1 s |
| FIFO, 80,000 lots open | 9.3 s |
| FIFO, 125,000 lots open | **16.3 s** (first run), 23.8 s (the first version's run, busier machine) |
| LIFO, 40,000 lots open | 1.4 to 2.0 s |
| LIFO, 80,000 lots open | 8.5 s |
| LIFO, 125,000 lots open | **20.5 s** |

  Wall times moved by up to 3x between my two sessions with machine load; the shape (growth with the number of open lots, LIFO equal to FIFO) did not. What matters is the number of lots standing at once, not the number of vouchers: an item that sells as fast as it is bought stays cheap, which is why the repo's harness never sees this (its synthetic book is
  about 2 lines per voucher spread over 300 items, `docs/PERF.md:108-113`) and why a **handset or jewellery dealer who holds thousands of single units** is exactly the book that hits it. A batch label adds
  almost nothing (`Avg. Cost + 500 batch names` 44 ms against 44 ms); the lot array is the cost. [C][P]
- **The pass is on the keystroke path.** `VoucherScreen.tsx:1371-1374` memoises `stockPositions(st.stockItems, st.vouchers, date)`; `st.vouchers` changes on every save (`store.ts:698-702`), so each Accept pays it
  again (`vch.save` 223 ms p50 at 250k, `docs/PERF.md:178`; `prim.stockPositions` 145 ms, `:192`). The item picker also reads `positions` for every row of its list per keystroke
  (`VoucherScreen.tsx:2871-2877`). [C]
- **Consequence (revised):** serial goods must not ride the per-item lot array, and serial state must **not** be added to `stockPositions` (the hot, always-recomputed pass). If a serial item may also be FIFO or
  LIFO, `lotsValue` has to become an incrementally maintained sum, **and** `shift` an index or deque; replacing only `shift` buys nothing. The cheaper route is the one the lane list assumes: a serial item is
  costed by specific identification (each unit at its own purchase cost) and never builds lots; a serial index is its own lazy pass built only for serial-tracked items, like the dashboard's separate bills and
  stock passes (`docs/research/wave-a2-plan.md:289`). A perf row with one-unit lots at 100k and 250k belongs in the gate. For scale, **a lazy serial pass over a 250,000-voucher book holding 1,000,000 serial events (500,000 distinct units) took about
  0.24 s and a point lookup 0.4 microseconds** (probe 6, Appendix A), against the 1,200 ms p95 the repo allows a full-period stock report (`docs/PERF.md:80`): the index is affordable, the lot array is not. [J][P]

### 3.3 The voucher entry screen (`src/ui/VoucherScreen.tsx`, 4,118 lines)

**The walk today.** Party, then per item row: name (Enter picks the highlighted match), godown (only when the company has more than one godown, `multiGdn`, `:1354`), quantity, rate, optional discount, amount,
optional description (`:3211-3322`). Qty Enter is guarded ("Quantity is required!" `:3267`); an item row's amount is guarded the same way (`:3310-3313`, which is also why a free-of-charge unit cannot be entered).
Nothing in the walk opens the batch panel: **`setBatchFor` opens it from exactly two places**, Alt+B (`:3059-3062`) and the rail button (`:3409`); the third mention is the dialog's own mount, which closes it
(`:4105-4113`). (The first version said three openers; the mount is not one.) [C]

**What `BatchPrompt` is.** A modal `Box` with two fields, "Batch / Lot no." and "Expiry date" (placeholder "d-m-yyyy (optional)"), Enter on the first goes to the second, Enter or Ctrl+A on the second saves
(`:883-914`, keys `:895-905`). It writes `{batchName, expiryDate}` and nothing else (`:4111`); it cannot list existing batches, cannot show stock in a batch, cannot validate. [C]

**Gaps, each read in code**

| Gap | Evidence |
|---|---|
| The batch is invisible on the entry grid | the table header is Name, Godown, Qty, Rate, Disc %, Amount, Description (`:754-760`); no cell prints `batchName`. A carried value must always be visible (`CLAUDE.md`, Phase 20-21 gotchas). |
| No required batch, no expiry check, no negative-batch warning | `doAccept` (`:2349-2512`) warns on negative **item** stock only (`:2490-2506`), **only for a new entry** (`!editing`, `:2491`) and never for Physical Stock; no batch test anywhere in the file. An alteration that swaps the batch (or later the serial) of an old invoice is therefore never warned. `BulkEntry.tsx:486-489` refuses batch items because "a batch-tracked item posted without a batch breaks the batch/expiry reports" |
| Changing the item keeps the old batch | typing in the name box sets `{text, itemId: null}` (`:771`); `pickItem` sets the new item and clears GST fields and description but not `batchName/mfgDate/expiryDate` (`:2047-2075`) |
| A duplicate or recall copies the batch | `itemRowFromLine` copies `batchName/mfgDate/expiryDate` unconditionally (`:1153-1175`, batch at `:1163-1165`); Alt+2 (`:1275,1288,1295`, handler `:3039-3043`) and "Use last" (`:1735`) call it. Bill allocations and instrument numbers are deliberately dropped there (`:1227-1231,1736-1739`); batches are not. |
| Typing a code that matches nothing | Enter on the item name calls `pop?.pick(popSel)` (`:3223`), `pickItem` returns when `filtered[itemIdx]` is undefined (`:2047-2049`): no message, the text stays, and the next typed characters **append** (the focus effect only selects when the focus key or a row count changes, `:1496-1503`) |
| Item popup matches name, alias, part no by substring | `:2865-2868`; each row also reads `positions.get(x.id)` (`:2872`) |
| Batch is not in the stock-journal seam either | transfers: both sections use the same `BatchPrompt`, but nothing prompts (`:3397-3409` `onBatchItem` works for any section) |

**Where a build hooks in (existing seams, all in this one file)**

- the row type `ItemRow` (`:168-193`) and the focus type `Focus` / `ItemCol` (`:203-218`); CF-3 adds `` `cf:${string}` `` to `ItemCol` in the same place (in the worktree `ItemCol` is still the seven columns, `.wt-cf/src/ui/VoucherScreen.tsx:216`, so CF-3 is not built yet);
- `freshItemRow` (`:1145`), `itemRowFromLine` (`:1153`), `validInvLines` (`:1447-1469`, the **only** place a line is built from the grid; the literal starts at `:1452`, batch fields at `:1463-1465`), the dirty `signature` (`:1614-1624`);
- the walk (`:3211-3322`), the rail buttons (`:3407-3460`), the popup spec (`:2862-2881`), the hold/pin constants (`PinKey` `:163`, `pinned` `:1698`);
- **F12 voucher configuration**, the existing per-company switch for optional item-grid columns: `VoucherView` (`types.ts:241-246`, stored as `Company.voucherView` `:152`) holds the Godown mode (automatic, always, never) and the Description column, appended **after** Amount so no key moves (`core/voucherConfig.ts:1-20`, `ui/VoucherConfig.tsx`). A Batch or Serial column is a further switch of the same kind; the file's header says MRP and actual-versus-billed quantity were left out because they "change the stock and tax arithmetic, not just a column" (`:16-17`), and `normaliseView` drops every key it does not know (`:52`), so an older build re-saving the view would erase a new key (the older-client class of 4.2 item 3);
- the chord registry: Alt+B is already a registered chord (`id: 'batch'`, `core/shortcutRegistry.ts:241`); a new chord is a row there first, and `keyLabel` prints it;
- the Accept chain: `doAccept` warnings (`:2349-2512`) then `save` with the cloud reservation "after every question that can stop the save and immediately before the write" (`:2708-2735`, written for server
  numbering, **now deployed**). A serial reserve belongs beside it; the numbering call has a 4 s timeout and a 30 s breaker (`src/core/vchNumbering.ts:20,28`), so two separate cloud calls on Accept would double
  the worst case (merge them into one request). [J]
- `useDirtySince` / the signature: a serial list and a batch pick must be in it.

### 3.4 Every writer of an inventory line, and every reader of one

**Constructors of `InventoryLine` (five, found by `grep "stockItemId:"` over `src` and `scripts/seed`, re-run)**

| Writer | File:line | Batch today |
|---|---|---|
| the voucher screen | `VoucherScreen.tsx:1453` (inside `validInvLines`) | name, mfg, expiry carried |
| Manufacturing Journal | `manufacturing.tsx:172,175` | none |
| Bulk Entry | `BulkEntry.tsx:521` | refuses batch items (`:486-489`); in the worktree its pure half moved to `.wt-cf/src/core/bulkPlan.ts:462` with the same refusal |
| Tally XML import | `tallyImport.ts:1290-1304` | name only; "Primary Batch", "any", "not applicable" mean none (`:1120`) |
| the sample-company generator | `src/core/demo.ts:363` | none |

Other code that **creates vouchers from whole cloth** without a line constructor: the mapped CSV/Excel importer builds two-ledger vouchers only (`importMap.ts:646` `buildVouchers`; `FIELDS.vouchers` is date, type,
debit, credit, amount and narration only, `:72-79`; its item fields are name, unit, group, HSN, GST rate, taxability, cess, opening qty and rate, `:61-71`), Split Company Data builds items with
`openingAlloc` (`splitCompany.ts:528-552`), and REST `POST /v1/:collection` stores whatever document it is sent (`backend/src/rest.js:58-77`). **A purchase from a supplier's Excel sheet of 500 IMEIs has no
import path today.** [C]

**Readers.** `grep "\.inventory\b"` finds uses in 43 files (`grep -rl "\.inventory\b" src`, 2026-10-23). Adding fields to the line is safe for all of them. **Splitting one line into one line per unit is not**:
it changes the e-invoice item count and `SlNo` (`einvoice.ts:446`, 1,000-item cap `:414,670`), the printed rows, the **e-way bill item list** (`core/ewb.ts:309-319` reads `v.inventory` line by line and warns above
250 lines, `:436`), the Mismatch Report's line labels (identity is `(stockItemId, godownId, batchName)`, `syncDiff.ts:53`, so N identical lines read "(2)", "(3)"), and CF's carry rule for `udf` on lines, which drops a
value when a line identity repeats (`.wt-cf` spec `§9.3`, the weak-identity discussion). GSTR-1 Table 12 sums quantity by HSN, unit and rate, so it is unaffected (`gstr1.ts:628-634`). [C][J]

### 3.5 Reports and screens (`src/ui/inventory/*`, Reports Center, dashboard)

- Files in `src/ui/inventory/`: `BatchSummary` (stub), `PriceLevels`, `ageing`, `godownstock`, `manufacturing`, `masters`, `movement`, `reorder`, `reports`, `stockquery`, `stocksummary`, `valuation`,
  `valuationSwitch`. The only batch output is the "Batch-wise Balance" block of Stock Query: name, expiry, quantity, **no value, no godown, no age, no filter**, one item at a time (`stockquery.tsx:135-145`). [C]
- **Stock Ageing** ages closing stock by purchase-date FIFO layers per item, "regardless of the item's valuation method" (`ageing.tsx:1-8`); it does not look at batch or expiry. [C]
- **Stock Summary, Godown Summary, Movement, Reorder** read `stockPositions` or `resolvedMovements` and have no batch dimension (`godownstock.tsx:41`, `reorder.tsx:33`, `inventory/reports.tsx:37,143`). The
  admin **Exceptions** report lists negative stock per item as at a date (`ui/admin/Exceptions.tsx:40-60`); a negative batch or serial cell would need its own line there. [C]
- **To replace the stub** a build edits, in one change: `BatchSummary.tsx`, `router.tsx:282,365`, `scripts/assert/sidenav.assert.ts:131-133` (the allow-list must become empty, the suite tells you so),
  `catalogue.ts:205` (remove from `EXCLUDED_REPORT_SCREENS`, add a row), the three entry points above, and decide on the "Batches & Expiry" rail row (`docs/design/left-nav-spec.md:251`: "add the row the day the
  screen ships"; the literal row table in `sidenav.assert.ts` must be edited with `navTree.ts`, 65 rows today). [C]
- **The report grid is not built.** `docs/research/wave-a2-plan.md:191` puts Stock Summary outside A2; CF-4b waits for B-grid S1 and S2. A batch / serial / expiry report built now is bespoke (the row filter
  `Alt+F12` / `Ctrl+F12` is on the Day Book, voucher registers, Ledger Vouchers and Receivables / Payables only, `CLAUDE.md` Known limitations). [C]
- **Dashboard.** Widgets are a static registry (`src/ui/dash/widgets/registry.ts`); `stockAlerts` (reorder and negative) and `stock` use the `'stock'` pass of `env.lazy` (`registry.ts:88-90,131-133`; the pass is
  `stockPositions` `ui/dash/widgets/env.ts:89`). An "Expiring soon" or "Serials in stock" widget is a new registry row plus a compute file under `widgets/compute/`; a hidden widget is never computed. [C]
- **Print rows / exports.** The report exporter types a right-aligned cell as a number when it is up to 15 digits with no leading zero (`src/core/reportSheet.ts:73,106,117-131`): a 15-digit IMEI in a
  right-aligned column would be written to Excel as a number, and the CSV writer defuses only text that could be a formula and treats a plain digit string as a number, with no text marker
  (`src/core/csvOut.ts:17-29`). Serial and barcode columns must be declared text, left aligned. [C][J]

### 3.6 Print (`src/ui/print/*`, `reportTools.ts`, `printConfig.ts`, Electron)

**What is there**

| Piece | Facts |
|---|---|
| Print host | a DOM node outside `#root` that `window.print()` snapshots: `PrintVoucher.tsx:608,676`, `BulkPrint.tsx:128,131`, `reportTools.ts:257-280` (`window.print` at `:275`) |
| Pages | `@page pv-sheet` (A4 portrait) and five more named pages, all `margin: 0`, plus two report pages with margins (`src/styles/print.css:34-43`); "@page cannot read a custom property, so each size is its own named page" (`print.css:35-36`) |
| Sheet geometry | `PAPERS = ['A4','A5','Letter']`, `sheetGeometry()` (`printConfig.ts:32-60`); `pageName` maps to the named pages (`:76-79`) |
| Custom size precedent | `PrintLeaf` injects `<style>@page { size: {w}mm {h}mm; margin: 0 }</style>` only while printing (`chequeLeaf.tsx:166-172`); fields placed in millimetres with a per-printer offset, a dashed focus frame, a "Print test sheet" key (`chequeLayoutEditor.tsx:212,238`); `Cheque` types in `types.ts:906-942` |
| Pagination | `PagedDoc.tsx` lays a document into explicit sheets from browser-measured heights because Chromium 126 has no `@page` margin boxes (`PagedDoc.tsx:3-12`; `print.css:21`) |
| Many documents, one job | `BulkPrint.tsx` (selection `I` key, `selectionBar.tsx:161-168,296`), `MAX_BULK = 100` (`core/bulkPrint.ts:14`; more than that is refused whole, `:53`) |
| PDF | `electron/exportBridge.cjs` `pdf:save`: `printToPDF({ preferCSSPageSize: true, printBackground: true, ... })` (`:108-114`); fallback sizes limited to A4, A5, Letter (`:24`, `preload.cjs:36`); a browser has only the print dialog (`ui/pdfOut.ts:1-12`) |
| Fonts | `ensurePrintFonts()` loads IBM Plex first so the rupee sign prints (`print/printFonts.ts`) |
| QR | `core/qr.ts` + `ui/print/QrCode.tsx` (`QrSvg`: black on white, size in mm, ECC L or M) |
| Invoice item rows | each line is an `InvRow` whose `sub: string[]` prints one `.sub` div per entry (`invoiceModel.ts:614-621`, `PrintVoucher.tsx:88`); the batch line is one of them, switchable (`printConfig.ts:103`); height is estimated at one line per entry (`invoiceModel.ts:337-340`) and then measured; **a row is atomic and is never split across pages** (`paginate.ts:12-14`). A line with 40 IMEIs is one tall unbreakable row unless the serials are chunked into several `sub` strings and capped. |
| Voucher-type hook | `VoucherType.printAfterSave` opens the print preview after Accept (`types.ts:543`; `VoucherScreen.tsx:2816,2826`) |

**What is missing**

- No 1-D barcode encoder (Code 128, EAN-13, EAN-8, UPC-A, Code 39); `package.json` has no barcode or PDF library and the project's habit is a pure, asserted `src/core` file instead of a dependency (`qr.ts`,
  `xlsx.ts`, `zip.ts`, `inflate.ts`). [C]
- No label sheet (N labels across and down, gaps, a start cell for a part-used sheet), no roll label, no custom page size in Print Configuration, no thermal-receipt width. [C]
- The Electron bridge cannot print silently or choose a printer: `webContents.print` supports `silent`, `deviceName`, a custom `pageSize` and `getPrintersAsync` exists [W], and the app uses none of them. A new IPC
  would be validated like `pdf:save` (renderer untrusted, no paths, option allow-list, `exportBridge.cjs:8-18,49-53`); note that wave 3's bridge **has never been run against a real Electron binary** (HANDOFF §2f). [C][W]
- Windows label printers normally print through a driver, which means CSS-to-driver output, so bar width has to be a whole number of printer dots (203 dpi is 8 dots per mm; a 0.25 mm module is 2 dots) or bars
  alias. [J][unverified]
- A browser tab cannot do any of the silent part and ends in the print dialog. [C]

**Consequence.** Labels are buildable from existing parts. The unusual work is (a) a pure label layout engine with a deterministic dot-aligned barcode, (b) a custom-size page path that is not the named-page
table, (c) an Electron `print:label` IPC if silent printing is wanted, (d) a one-label-per-page vs sheet N-up decision, and (e) the entry flow ("print labels for the lines of this purchase", the `I` key's
sibling). The calibration idea from the cheque leaf (offset X/Y, test sheet) is the right precedent. [J]

### 3.7 Tally and file interchange

**Tally XML in** (`src/core/tallyImport.ts`): `ISBATCHWISEON` sets `item.batches` (`:987`); opening stock is one `OPENINGBALANCE`, `OPENINGRATE`, `OPENINGVALUE` per item (`:959-967`); **batch openings under a
stock item are not read** (there is no `BATCHALLOCATIONS` read outside voucher lines, `:1273`). On vouchers, `BATCHALLOCATIONS.LIST` splits a line into one line per godown / batch part when the parts add up to
the line quantity (`:1273-1285`, `stockSplit++`), the batch name is kept unless it is "Primary Batch", "any" or "not applicable" (`:1304`, `:1120`); **no `MFGON`, no expiry, no serial**. The assertion suite
treats "&#4; Primary Batch" as no batch (`tallyimport.assert.ts:367`). [C]

**Tally XML out** (`src/core/tallyExport.ts`): `ISBATCHWISEON` (`:482`), one opening figure (`:519-523`), `BATCHALLOCATIONS.LIST` per line with the batch name or "Primary Batch" (`:783-789`), and a "batch lines
with manufacturing / expiry dates" entry in the "not carried" notice (`:569,628,731`). [C]

**Practice to expect `[unverified]`:** Tally has no serial-number object, so traders commonly record an IMEI as a batch name of quantity 1. If so, a migrating handset dealer arrives with `batchName = IMEI` on
thousands of lines, and the importer's own split rule (`:1281-1285`) already turns a multi-allocation line into one line per allocation. A migration step that offers "convert batch-of-one lines to serials" is
worth designing in; I could not verify the practice against a real Tally file. [J][unverified]

**CSV and Excel in** (`ImportData.tsx` and the mapped importer `importMap.ts`, `MappedImport.tsx`): items have name, unit, group, HSN, GST rate, taxability, cess, **opening qty and rate** only
(`importMap.ts:61-71`, builder `:548`); no barcode, no batch, no serial, no opening by batch. The mapped importer is additive and balance-guarded, with a preview before saving and a remembered column
mapping (`importMap.ts:1-18`), which is the right shape to extend. Its voucher kind makes **two-ledger vouchers only** (`:72-79`, `:646`); a purchase of 500 IMEIs from a supplier's sheet is an item voucher
and has no path. [C]

**Backup JSON** carries every table row (`db.ts:445-472`); a restore replaces all rows of the company and logs every one to the outbox (`:505-568`). Backup schema is version 2
(`core/backupMigrate.ts:30`); the custom-fields build takes version 3, written only for a company that holds custom data, so that an older app refuses the file instead of eating the values on its next save
(`.wt-cf/src/core/backupMigrate.ts`, +61 lines against its base; spec CF5 in `§0` and `§12`). **The next schema number is already contested**, and serials embedded in vouchers have exactly the same older-app
exposure. [C]

### 3.8 e-invoice, e-way bill, GST quantity

- `buildInv01` (`einvoice.ts`) emits `BchDtls {Nm, ExpDt}` only when the batch name is 3 to 20 characters (`:358-359`) and nothing otherwise, with **no warning** that a shorter name was left out; the validator
  errors only if a `BchDtls` object is present with a bad `Nm` (`:717-719`). It emits no `WrDt`, no product serial number, no barcode (`grep -rnE "PrdSlNo|Barcde|WrDt|AttribDtls" src` returns nothing; the only per-item
  free text it adds is `PrdDesc`, `:45,447,716`). [C]
- The official schema (Form GST INV-1, version 1.1, Rule 48; the CGGST copy) has, for each item: a **barcode** (optional string, at most 30 characters), a **unique product serial number** (optional, single
  value, at most 20 characters, "in case of each item having a unique number"), **batch details** (optional, once per item: batch number at most 20 characters, expiry date and a **warranty date**, both
  `DD/MM/YYYY`) and product attribute details (name and value, up to 100 characters, repeatable). [W] The extracted PDF text gives the parameter descriptions and lengths; the JSON tag names `PrdSlNo` and
  `Barcde` are the common ones and the batch tags are the ones the code already uses, but the tag spellings of the first two were not in the extracted text. [unverified]
- Because the serial field takes **one** value per item row, a line of 5 handsets with 5 IMEIs cannot carry them: a design either emits one item per IMEI (the invoice then has as many items as units; the
  validator caps `ItemList` at 1,000, `einvoice.ts:414,670`, and the e-way bill warns above about 250 lines, `ewb.ts:436`) or leaves the serial blank. That choice sits in `einvoice.ts` and not in the entry
  screen. [C][J]
- GSTR-1 Table 12 and the HSN summary aggregate quantity by HSN, UQC and rate and are indifferent to per-unit lines (`gstr1.ts:603-646`). The e-way bill reads the item lines (`core/ewb.ts:309-319`,
  `ui/gst/ewayPanels.tsx`). [C]

### 3.9 The Dexie schema, indexes and load path (`src/data/db.ts`)

- Version 8 is the latest; versions add tables with `'id, cid[, field]'` keys, never a compound index (`db.ts:312-314`). `vouchers` is indexed `id, cid, date, base` (`:124,209,233`); there is **no index on any
  line field**. [C]
- Every table's rows for the company are loaded into memory on open (`loadCompanyData` `db.ts:573-610`) and again after **each applied pull page** (`reloadActive` `sync.ts:615,713`). All engines then run over
  arrays. The app does not query IndexedDB for business reports. [C]
- A new table must be added in at least these places: `TableName` and `ALL_TABLES` (`db.ts:23-39`), the Dexie class and a `version(9)` block, `loadCompanyData`, `store.ts` `COLLECTIONS` (`:209-219`) and
  `State` and `loadInto` (`:307`), `checkDelete` (`store.ts:800-830`), `core/access.ts` `TABLE_AREA` (`:323-335`; an unlisted table is **admin-only** for writes, `:366`), `core/undo.ts` `NOUNS` (`:124`),
  `core/integrity.ts` (`BOOK_TABLES` `:62`), `backend/src/conflict.js` `DATA_TABLES` (`:79-87`; `syncconflict.assert.ts` fails when the lists drift), `backupMigrate.ts`, and `restPolicy.js` (derived,
  `:18-20`). [C]
- A multi-entry index would need a derived top-level array on each voucher (IndexedDB key paths do not traverse arrays of objects) and a Dexie hook to fill it, because `applyRemote` does a plain `put` of
  whatever another device wrote (`db.ts:394-411`) and an older client's document would arrive without it. A hook already exists for the outbox (`db.ts:322-324`) and the rule for hooks is "work must be O(1)"
  (Phase 24 gotchas). Not tested here: IndexedDB is not available to Node. [C][J][unverified]
- **Order of magnitude** `[P]` (probe 4, Node, one synthetic book, re-measured 2026-10-23): an in-memory `Map` of 1,000,000 serials (item + serial key) built in **0.4 to 0.7 s** with about **95 to 105 MB**
  extra heap; 100,000 is about a tenth of that. A voucher line with 4 serials is **181 bytes** of JSON, with 5,000 serials **90 KB** (`conflict.js:98-103` keeps a version up to 512 KB). One-row-per-serial with
  realistic UUID ids: **250 bytes** for the document plus **457 bytes** for the outbox row that embeds it, **about 707 bytes a serial, about 0.7 GB per million** before IndexedDB and the server's own copy,
  and **3,334** push requests at 300 per request (`sync.ts:47`; the first version's 365 bytes was a leaner row). The server accepts 20 MB per request (`backend/src/server.js:28`). The existing book already
  reaches 277 MB of heap at 250k vouchers (`docs/PERF.md:112`). [C][P]
- `checkDuplicates` (client scan after a pull) is skipped when more than 5,000 vouchers arrived at once (`sync.ts:668-670`), so a second device's first sync is never scanned. [C]
- **The server's change log is an append-only jsonb table with no index on payload content** (`backend/sql/001_p1.sql:34-45`, `007_sync_conflicts.sql:54`): "which other vouchers name IMEI X" cannot be asked
  of it without scanning every voucher payload, so a server-side serial check needs its own table written inside the push (migration `009`), as `voucher_number_issue` is for numbers. [C][J]

### 3.10 Sync and the hard problem: one serial, two devices

**What the rules do today**

1. *Revision / baseline.* Every outbox row is stamped with `base`, the highest server seq the device had applied (`db.ts:322-324`, `sync/baseline.ts`). `POST /sync/push` (`backend/src/sync.js:177-297`) takes
   the per-tenant lock (`:205`, `db.js:42-48`), loads for each pushed document the revisions other devices wrote after its base (`loadHistories` `sync.js:66-106`), and `classifyPush`
   (`conflict.js:191-278`) records an **edit-edit / edit-delete / delete-edit** mismatch when another device wrote the *same document* after the pusher's base. The write is still accepted: "last push wins...
   nothing blocks sync" (`conflict.js:15-16`), and a refused batch "would strand its whole outbox" (`conflict.js:73-77`). [C]
2. *The unit of comparison is the document id.* `docKey(table, docId)` (`conflict.js:173`, `syncGuard.ts:131`). Two different vouchers that both name IMEI X never meet. **No rule fires.** [C]
3. *Pull guard.* `planRemote` (`syncGuard.ts:183`) stops a pulled version overwriting an unsent local edit of the same document. Same unit, same blindness. [C]
4. *The only cross-document duplicate detector is for voucher numbers:* `findPullDuplicates` (`syncGuard.ts:311`) builds a map over **all** vouchers and tests the ones that arrived; `checkDuplicates` runs it
   after a pull (`sync.ts:669-688,722`, cap 5,000 at `:668`); the device reports it as kind `duplicate-number`; the report endpoint accepts the kinds in `KINDS` (`conflict.js:112-115`) and the admin may
   acknowledge, renumber or dismiss (`:117-124`). The `kind` column is plain text with no CHECK (`007_sync_conflicts.sql:15`), so a new kind needs no migration. `'duplicate-number'` appears in 15 files
   (`grep`, re-run; the first version counted 14): `conflict.js`, `sync.js`, `numbering.js`, `syncGuard.ts`, `vchNumbering.ts`, `sync/numbering.ts`, `integrity.ts`, `types.ts`, `SyncConflicts.tsx`, the MCP
   formatter `mcp/src/format/issues.ts` and five suites. A new kind is therefore a change in about as many places, the MCP's among them. [C]
5. *Wave 3's server numbering (`backend/src/numbering.js`, `008_voucher_series.sql`)* is the closest pattern, and it is **deployed**. Reserve, claim and release take one advisory lock per series
   (`numbering.js:169-171`); a ledger row's **primary key is the number itself**, so a duplicate cannot be written whatever the app does (`008_voucher_series.sql:11-17`); a reserve is idempotent on a key; a
   device that could not reach the cloud saves a *provisional* number and claims it at the next sync; a clash is **reported, never renumbered by the server** (`numbering.js:1-40`,
   `vchNumbering.ts:20-41`). The client reserves "immediately before the write" with a 4 s timeout and a 30 s pause after a silent cloud (`VoucherScreen.tsx:2708-2735`, `vchNumbering.ts:20,28`). The routes
   are `/sync/next-number`, `/claim` and `/release` (`sync.js:319-325`). One claim per request today; the mobile readiness note already asks for a batch claim
   (`docs/research/mobile-backend-readiness.md:471`). [C]

**What a uniqueness check can and cannot promise** `[J]`, from those rules:

| Situation | What a check can promise |
|---|---|
| One device, sync off or on | **Everything.** A local scan sees the whole book. A restore replaces the book, so it re-establishes the truth. |
| Two devices, both online | A server reserve at Accept (like numbering) can promise it, under a per-tenant or per-serial lock. |
| Two devices, one offline | Only "this device's own book has no earlier sale of it". The other device's unsynced sale cannot be known. Both Accepts succeed. |
| After both sync | The conflict is knowable: both vouchers are in the server's change log. It can be **detected and recorded**, not undone: the invoice may already be printed and handed over. |
| Ordering | `base` tells concurrent from knowing: if the other sale's seq is above the pusher's base, the pusher could not have seen it (an offline clash); if at or below, it was seen (a deliberate duplicate, or a return and resale). That is the existing baseline rule used for a new question. |
| Any writer that is not the voucher screen | Not covered by a check inside the screen. `markOptional` flips optional to regular with `putDoc` (`ui/reports/markOptional.ts:87-110`), `BulkEntry` and `TallyImportPreview` write with `putDoc` (`BulkEntry.tsx:943-944`, `TallyImportPreview.tsx:91`), `MappedImport` writes with `putDoc` inside its own transaction (`MappedImport.tsx:132-133`), `gstr1.tsx:174` and `advances.tsx:87` re-put vouchers, and REST `POST /v1/vouchers` appends any document (`rest.js:58-77`). Only a check at the **write funnel** (`putDoc`, using the same hook point as the operator guard, `data/writeGuard.ts:43-46`, which is also skipped by `applyRemote`) or on the **server push** sees all of them. |
| Date order | Voucher dates are user-editable; engines order by `(date, voucher number)` (`store.ts:295-296`, `inventory.ts:46`), so two sales of one unit **on the same date** are ordered by voucher number, and a voucher has no time of day (`types.ts:991-1049`). A backdated purchase after its sale, or a backdated sale of a unit sold later, are real cases. The serial rule must be "in stock as at the voucher's date and not out again later", not a current-state flag. |
| Cancel and delete | A cancelled voucher keeps no lines (`voucherCancel.ts` `shellProblems`: "a cancelled voucher has no item lines", `:244-249`; written by `cancelledShell` only, `:180-221`; the store re-seals every voucher it holds, `store.ts:295-296`); a deleted voucher is gone. Both release embedded serials on the next recomputation with no extra code. A server-held claim would need releasing on the same push. |
| Optional and post-dated | `optional` vouchers never move stock (`inventory.ts:39`); a serial on one must not claim; marking it regular later must. |

**The shape that fits the repo's conventions** `[J]`: keep the serials on the line (one source of truth); detect duplicates with a pure, asserted function used by three callers (the entry screen at Accept,
the Data Check, and the post-pull scan); add a server **index of serial events** written under the same tenant lock that `/sync/push` already takes (after `planPush`, `sync.js:222-241`), with a new mismatch kind
(`duplicate-serial`) whose resolution is *acknowledge or dismiss* (the real fix is an ordinary credit note or Alt+X, which wave 3 now provides); never reject a push, which would break `conflict.js`'s stated
policy and mobile's "reject the row, not the batch" (`docs/research/mobile-backend-readiness.md:250`, R2). The reserve call at Accept is an **optimisation that narrows the window**, with a provisional flag
when the cloud is silent, exactly as numbering does. Alternatives and their costs are in section 5, risk 2.

### 3.11 Voucher numbering interplay

A serial claim is **not** a number: a voucher number is issued once and never reused (`numbering.js:1-40`); a serial goes in, out, in (a return), out. So the ledger is a *current-holder lease* keyed on
`(tenant, company, item, serial)` with the holding voucher and a direction, not a monotonic counter. What can be copied: the idempotency key (the voucher id), the advisory lock, the provisional status, the
claim queue worklist stored in `meta` (`sync/numbering.ts:29-33`, a `vnoPending:` key family with no schema change), the 250-per-run claim cap (`vchNumbering.ts:38`) and "reported, never undone". [C][J]
A voucher that is renumbered after a clash (`numberLog`, `types.ts:1047`) keeps the same id, so a serial lease keyed on the voucher **id** survives it. [C]

### 3.12 Cancel, delete, undo, duplicate, recall, alteration

| Flow | What happens to lines today | Consequence for serials |
|---|---|---|
| Cancel (Alt+X, wave 3) | `saveVoucher(cancelledShell(...))` writes a shell with `entries: []` and no `inventory`; `optional: true` (`voucherCancel.ts:180-221`; `useCancelVoucher.tsx:99`) | embedded serials are released automatically; a separate table must be updated in the same transaction as the shell |
| Delete | `deleteVoucher` hard-deletes the document (`store.ts:708-722`) | same |
| Undo (Ctrl+Z) | reverses the **document** (`core/undo.ts`: classify, `reverseOf`); a write to a second table with a user stamp is a second undo step unless `origin: 'flag'` or `system: true` (`db.ts:332-346`; `CLAUDE.md` Phase 20-21 gotcha) | embedded: one step, correct; separate table: two steps or a missing release |
| Alteration | the screen rebuilds the voucher from its fields and carries listed fields through (`VoucherScreen.tsx:2737-2781`); a field it does not name is erased | a new line field must be in `itemRowFromLine(l, true)` and `validInvLines` |
| Duplicate Alt+2 | `itemRowFromLine(l, false)` (copies batch and dates) | **drop serials** (and batch for an outward voucher), as bill and instrument allocations are dropped |
| Ghost recall "Use last" | same function (`:1735`) | same |
| PgUp / PgDn walk | pushes the adjacent voucher by id (`:3045-3052`) | none |
| Mark Optional / Regular | bulk `putDoc` with `origin: 'flag'`, outside `saveVoucher` (`markOptional.ts:99,110`) | a serial check must also run here, or detect later |
| Restore | `importCompany` replaces all rows and re-logs them (`db.ts:505-568`) | a restore carries embedded serials; a table must be in `ALL_TABLES` |

### 3.13 Item search and what a scan resolves to

- The voucher item popup filters by substring on name, alias, part number (`VoucherScreen.tsx:2865-2868`); the master picker and Stock Query use `PopupItem.alt = alias + partNo` (`ui/masters.tsx:859`,
  `stockquery.tsx:43`, `widgets.tsx:356-361`); Bulk Entry resolves items by exact name, alias or part number (`BulkEntry.tsx:275`); the palette indexes alias and part number
  (`CommandPalette.tsx:425`); Tally import reads `PARTNUMBER` (`tallyImport.ts:972`). [C]
- A barcode is an **exact** lookup, not a substring filter, and an item can have several (EAN plus a vendor code, one per size and colour for garments, one per pack size). Putting a barcode in `partNo` is a
  working stopgap for exactly one code per item and nothing else. A new `barcode` field must reach every site above plus the master tail, the mapped importer's `FIELDS`, and the palette. [C][J]
- New master fields go last or in an Alt+key grid, never before Opening Balance: a habitual Enter walk typed into the wrong box in Phase 24 (`CLAUDE.md`, Phase 24 gotchas). The stock item form already has a tail
  mechanism (`ui/inventory/masters.tsx:453-457`, `tail: true`, the e-Invoice "Service item" toggle), and wave 3 added the stronger precedent: a **`tailGroup`**, so that Enter on what used to be the last field still
  raises Accept and Down steps into the new group (`ui/partyTermsFields.ts:1-15`; `party-terms.assert.ts` reads the source to hold the order). Tracking mode, barcodes and MRP are tail fields in their own group. [C]

### 3.14 Keyboard and what a HID scanner burst does (`src/keys.ts`)

**A keyboard-wedge scanner types the code as ordinary key events, fast, then usually Enter (some are set to Tab).** [J] In this app:

- One global `keydown` listener runs a last-in-first-out stack of handlers (`keys.ts:11-34`); a handler returns true to consume. The first handler on the stack is the open dialog or the top screen. Printable
  keys are **not** consumed by default: they reach the focused input. Several handlers read bare letters: the item grid's `D` and `C` on the Dr/Cr column of accounting rows
  (`VoucherScreen.tsx:3095-3096`), every Gateway and report hotkey, and the `AcceptPrompt`, which answers Y, N, Enter and Esc, toggles on Tab and Arrow keys and **swallows every other key**
  (`widgets.tsx:474-485`). `combo()` upper-cases a single character (`keys.ts:216`), so a hotkey letter fires on either case. [C]
- A person's own shortcut table translates the pressed chord before any handler sees it, with a `Proxy` over the real event, only when a table exists (`keys.ts:46-125`); printable keys and Ctrl+Alt are never
  rebound (`CLAUDE.md`, Phase 27). A scan detector has to sit **at this layer, before the stack**, or letters have already fired. [C][J]
- There is an existing capture-phase observer that never consumes (`installAltReveal`, `keys.ts:303-327`); it is the pattern for observing a burst, but it cannot retract the first characters that already
  reached a field. [C]

| Where the cursor is when the burst arrives | What happens today |
|---|---|
| Item name (invoice or stock journal) | text filters the popup; Enter picks the first match; **unknown code: nothing, silently, and the next scan appends to it** (3.3) |
| Godown, party, ledger | filter popup; party Enter says "Ledger does not exist! (Alt+C to create)" (`:3205`) |
| Quantity, rate, amount, discount | the text is evaluated as an expression and a "= value" hint is drawn on every keystroke (`:150-153,745-746`); digits become a number; Enter advances. A 13-digit EAN becomes the quantity 8,901,234,567,890 and a 15-digit IMEI 356,938,035,643,809 with no limit, and a leading zero is lost (`parseQty`, probe 5, Appendix A). |
| Description | free text, Enter adds a row (`:3298-3302`) |
| **Narration** | Enter raises **Accept** (`:3367`); Space, Tab and Enter also try a snippet expansion first (`:3360-3366`) |
| "Accept ?" bar | the bar blurs the active input and answers Y, N or Enter (`widgets.tsx:471-479`); a scan that ends in Enter answers **Yes** |
| A screen with no input (reports, menus) | letters are hotkeys; Enter activates the highlighted row |
| Tab as the scanner suffix | the item grid has no Tab branch (`'Tab'` appears in the file only in the narration snippet code, `:3360-3365`), so Tab goes to the browser's next element, Narration (the CF spec says the same, `§2.2`) |

- **Not measured:** how many keystrokes per frame the voucher screen can take. The entry screen is one 4,118-line component whose state changes re-render it; the existing picker budget is 16 ms p95 for the
  *filter and row build* only, and explicitly excludes React rendering (`docs/PERF.md:3-9`). A scanner types about 5 to 10 characters per frame [J][unverified]. A burst test with synthetic key events at 2 to
  5 ms spacing in headless Chrome should be a gate. [J]
- **The camera is a different route**: `getUserMedia` and the Barcode Detection API need a secure context, and `BarcodeDetector` is missing or partial on several desktop browsers (Chromium on Windows and Linux,
  Firefox, Safari by default) [Ws]. The live deployment is plain HTTP (`docs/HANDOFF.md:63,90`); Electron loads from `file://` (`electron/main.cjs:214`), which counts as a secure context [Ws]. The mobile research
  reached the same conclusion (`docs/research/mobile-personas-and-jobs.md:112`). No permission handler is installed in `main.cjs` (`grep`). [C][Ws]

### 3.15 Rights, operator model, company switches

- Voucher writes are judged by the voucher's **base** and date window only (`core/access.ts:356-364` for the table case; vouchers via `voucherDenial` `:249`); Stock Journal and Physical Stock need the Inventory
  right, everything else Accounting (`voucherArea` `:216-222`). Serials embedded in a voucher inherit that. [C]
- A new **screen** needs a `SCREEN_AREA` entry or it is admin-only (`access.ts:108-210`; the inventory screens are at `:136-137`, `batchSummary: 'inventory'` is already there). A new **table** needs `TABLE_AREA`
  or its writes are admin-only (`:366`). [C]
- "Who may reprint a label", "who may mark a serial damaged or returned for service" and "who may edit a batch's expiry or MRP" are not covered by any existing right and need a decision (Inventory right is the
  natural home). The POS audit reaches the same conclusion for cancel, discount and reprint rights (`docs/research/pos-code-audit.md` section 0, item 10). [J]
- F11 company features have no batch or serial switch (`types.ts:91-101`); the per-item `batches` flag is the only opt-in. The Phase 27 rule is that a person who never opted in sees the app exactly as before,
  asserted by identity: new rail rows, a new column on the entry grid, a new Gateway row and any new walk step must all be gated on something a company chose (an item with tracking on, or an optional
  `CompanyFeatures` key which, like `discount?` and `tcs?`, is absent for old companies). [C][J]
- The write guard is the one pre-write hook every `putDoc` / `deleteDoc` / restore passes (`db.ts:336,365,379,511`) and which **sync pulls skip** (`writeGuard.ts:6-8`); it throws `AccessDenied`, which screens
  already catch and show (`VoucherScreen.tsx:2804`). A serial rule there would reach `BulkEntry`, imports and `markOptional`. [C]

### 3.16 The reporting platform

- F12 report configuration is on Trial Balance, Group Summary, Balance Sheet, P&L, Day Book, Ledger Vouchers, Stock Summary, the registers and Bills; the row filter on a narrower set; the grid
  (`core/listQuery.ts`, `ListTable`, catalogue `columnSets/*`) is planned and unbuilt (`CLAUDE.md` Known limitations; `docs/research/wave-a2-plan.md:163-238`). Saved views carry the filter
  (`Company.reportViews`). CF-4b adds the custom-field filter fields after B-grid S1 and S2. [C]
- A batch or serial report should be written **as a grid report from the start** if the grid is near; otherwise it is bespoke and will be re-ported (the plan lists "Stock Summary attributes" as class A3,
  `wave-a2-plan.md:65`). Which one is a sequencing decision for the owner. [J]
- Reports Center drift guard: a report is a catalogue row or an entry on `EXCLUDED_REPORT_SCREENS` with a reason, or `reportscenter.assert.ts` fails (`catalogue.ts:1-10`, `:195-215`). [C]

### 3.17 What pins today's behaviour (new in this revision)

A build changes pinned behaviour on purpose in several places; these are the suites and fixtures that will say so, so the commit can name the lines it moved. [C]

| Pinned thing | Where | Moves when |
|---|---|---|
| Physical Stock resolution, per-godown balances, FIFO lots | `scripts/assert/inventory.assert.ts` (batch-related cases; godown sum `:375-382`), `stocksummary.assert.ts`, `split.assert.ts` / `split-fixture.ts` | the count-group rule, a batch cell, the lot structure |
| Sample-company stock goldens and the 250k synthetic book | `scripts/assert/perf.assert.ts:111,126`, `scripts/perf/synth.ts:300,396-397` (one item in 20 is batch-tracked with names `B-<item>-1..4`, one in 10 is FIFO; the book's fingerprint hashes `batchName`, `:615`), `scripts/perf/passes.ts` | any engine change; a serial-heavy or lot-heavy synthetic book is not in it |
| Tally round trip of batches | `tallyimport.assert.ts:367`, `tallyexport.assert.ts` | importing opening batches, writing dates |
| Print and invoice model | `scripts/assert/print.assert.ts` | a serial line, MRP, a label page |
| New-screen checklist | `sidenav.assert.ts:131-133` (stub allow-list), `reportscenter.assert.ts`, `quickcreate.assert.ts`, `navstack.assert.ts`, the security-access suite (`SCREEN_AREA`) | any new screen or create |
| Master and line field classification | `master-resave.assert.ts`; `.wt-cf` `voucher-resave.assert.ts`; **`mcp/test/fields.test.ts`** | any new key on `StockItem`, `InventoryLine`, `Voucher` |
| Sync rules | `syncconflict.assert.ts` (`DATA_TABLES` drift), `syncguard.assert.ts`, `vchnumbering.assert.ts` | a new mismatch kind, a serial index, a table |
| Data Check kinds | `core/integrity.ts:69-71,106-120` (eight kinds today; a new kind is an `IssueKind`, a title and an order entry), `integrity.assert.ts`; MCP-locked | duplicate serial, negative cell, expired in stock, old multi-line counts |
| **MCP engine lock** | `mcp/engine-lock.json` + `mcp/scripts/drift.mjs` | any edit of the 52 locked `src/` files, `inventory.ts`, `valuation.ts`, `openingStock.ts`, `types.ts`, `integrity.ts`, `splitCompany.ts`, `syncGuard.ts`, `ewb.ts`, `vchNumbering.ts`, `voucherCancel.ts` among them (4.5) |
| A UI recipe with a batch or serial item | none exists | the sample company has no batch item; a build must add one (and `scripts/seed/` for a cloud tenant) |

---

## 4. Interplay with work in flight

### 4.1 Parity wave 3 (committed `2cee555`, deployed 2026-10-23)

No longer in flight. What it left that this work leans on: cancel (`voucherCancel.ts`, `ui/cancel/*`), server numbering v2 (`backend/src/numbering.js`, `008`, `src/sync/numbering.ts`,
`core/vchNumbering.ts`), single-entry, dispatch / e-way (`core/ewb.ts`, `ui/dispatch/*`, `ui/gst/ewayPanels.tsx`), share and PDF (`electron/exportBridge.cjs`), the mapped importer (`importMap.ts`), the Excel writer
(`core/xlsx.ts`, `zip.ts`), UPI QR on invoices (`core/upi.ts`), bulk print. **The first version's rule "nothing starts before wave 3 commits" is satisfied**; every line number in this revision is a committed one.
Two cautions from wave 3's own record that apply here: the Electron PDF write and share bridge were **never run on a real Electron binary** (so a `print:label` IPC has no proven sibling), and the deployed numbering
flows were not exercised with an authenticated cloud session on the VM (`HANDOFF.md:620`). [C]

### 4.2 Custom fields and CF-9 (the draft as of 2026-10-23, `.wt-cf` `f446674`)

**What the draft says now** (AD.3, read in full; by section, in `.wt-cf/docs/design/custom-fields-spec.md`; the file is 2,126 lines and its own status paragraph says the addendum has had **one** review, the third pass, and
names the lenses a second pass should apply):

- A choice field marked **Tracks stock** makes `(item, variant)` a stock position "exactly as a batch makes `(item, batch)` one" (CF24 in §0, AD.3.1); at most three per item; the key is derived at read time
  from the line's `udf` and never stored (CF25, `VariantKey`). **Costing stays per item** (pooled): a variant's value is the item's value shared by quantity (AD.3.5).
- **Engine (AD.3.3), as rewritten by the third review:** `ValuationOpts.variantOf`; `StockPosition.byVariant?: Map<VariantKey, {qty, byGodown}>` updated "beside `byGodown` and `byBatch`, after the valuation
  call", so **no item-level figure depends on whether a resolver was passed**. The first draft resolved a tracked item's Physical Stock count against the (godown, variant) balance; the review reproduced that this
  changed an item's qty, value, `avgCost` and `byGodown` for callers that passed a resolver and for none that did, and **withdrew it**. The replacement: a count is always a count of the **whole item at its
  godown**; a count made by variant is one **count group**, written as lines flagged `InventoryLine.countPart: true`, merged by **`stockMovements`** (the one generator behind `stockPositions`,
  `resolvedMovements`, `stockItemRows`, `stockFlow`'s observe stream and Stock Ageing) into **one movement** of their summed quantity; `countDeltas` spreads the item-level delta over the variants; `StockMove`
  gains additive `parts?` and `variants?`; `ui/inventory/ageing.tsx`'s lot walk moves unchanged into a pure `core/stockAgeing.ts`. `OpeningStockAlloc.variants` and `openingVariantMap` (trusted only under the sum
  rule and the per-godown rule), optional variant arguments on `stockItemRows`, `resolvedMovements`, `pendingOrders` (now per order line), `stockDeltaByItem`, `core/stockByField.ts`. A **lane I** owns
  `inventory.ts`, `openingStock.ts`, `valuation.ts`, `stockSummary.ts`, `splitCompany.ts` and `stockByField.ts`, "one writer at a time on each" (§16). The `InventoryLine.countPart` member is "the one `types.ts`
  addition of this lane".
- **Acceptance (AD.3.9, the `customfields-stock` suite):** with a real resolver "every existing field of every position (qty, value, avgCost, byGodown, byBatch) is identical and only `byVariant` is added" on a
  company that has counts made by variant; `stockMovements` deep-equal for every voucher with no `countPart` line; five named Physical Stock vectors read through five readers; mutations include "merge lines that
  carry no `countPart`", "read `countPart` only when a resolver is passed" and "drop the unlisted variants of a count".
- **Entry (AD.3.6, CF-9c; after CF-3, and §16 says never two of CF-2, CF-3 and CF-9c on `VoucherScreen.tsx` at once):** the cell only for tracked items; in the walk when required; picker rows that show stock
  ("in stock 12"); the negative-stock warning per `(item, variant)` naming the variant; the Stock Journal destination row starts from the source row's variant; Physical Stock **pre-fills one row per variant held
  at the godown** and saves them as a count group; manufacturing books Not set.
- **Migration (AD.3.7, CF-9d):** an "Opening by variant" panel (a `Box` with `onClose` and a dirty probe), a "Split Not set" Stock Journal draft, Split Company carries variants, Data Check lines.
- **Release order (§17, 7a):** CF-9a (the engine, pure, no screen) is early on purpose and "must land before anything else edits `inventory.ts`, `openingStock.ts` or `stockSummary.ts`'s `stockFlow`";
  **CF-9c is not released before the build carrying CF-9a is the one running everywhere that stock is read**, because an older build's `stockMovements` does not know `countPart` and reads a count group "last
  line wins". Per-variant costing (CF-9e) and variant-aware manufacture (CF-9f) are gated on the owner.

**What exists in the worktree today** (`f446674`, "WIP checkpoint 2 ... unreviewed"): `InventoryLine.countPart?: true` (`.wt-cf/src/types.ts:1062`) and `CustomFieldDef.tracksStock` (`:471`) as **types**;
`ItemRow.countPart` carried through `itemRowFromLine` and `validInvLines` (`.wt-cf/src/ui/VoucherScreen.tsx:205,1203,1508`); the Custom Fields editor's "keeps stock" tag and switch
(`.wt-cf/src/ui/admin/CustomFields.tsx`, the `cf-tag` cell; `.wt-cf/src/ui/admin/cf/FieldPanel.tsx`, the `tracksStock` handlers). **Not written:** `core/stockByField.ts`, `stockAgeing.ts`, any `byVariant`, any `variantOf`; `.wt-cf/src/core/inventory.ts` differs from
main by one additive field on `PendingOrderRow` (`line`). So the CF-9 **engine is still design only**, which is the one window in which a batch cell can still be agreed with it. [C]

**Where it meets this audit (revised)**

1. **The first version's main clash is largely resolved by the CF author's own review, and the resolution is reusable.** The first draft re-keyed Physical Stock by `(godown, variant)` while a batch needed
   `(godown, batch)`; that is gone. What stands is a **count-group mechanism that is not specific to variants**: `countPart` is "a structure flag, not the variant key", and the merge lives in
   `stockMovements`, which every reader shares. A batch count made as one line per batch has exactly the bug of section 3.2.2 (12 becomes 7) and exactly the cure: save the lines as parts of one count, merge
   them into one whole-item movement, and spread the delta over the batches. So **the batch build should ride the same flag and the same merge, and generalise `countDeltas` over a cell key** (batch, or
   batch plus variant) rather than add a second flag or a second merge. The same applies to a serial count (a stock-take that scans the units on the shelf). Two consequences to settle with the lane-I
   owner **before CF-9a is written**: (a) `stockPositions` updates `byBatch` from `m.line.batchName` (`inventory.ts:141-146,156-161`); a merged movement carries only the first part's line, so the per-batch
   deltas must come from `parts`, the same way `byVariant` does; (b) an item can be tracked by batch **and** variant (a shade and its lot; a shoe size and its consignment), which needs a
   `(godown, batch, variant)` cell: with `byGodown`, `byBatch` and `byVariant` as three parallel maps each updated beside the valuation call, the combined cell has no home. One shared **cell key** with the
   three maps as projections is still the cheaper design, and still cheap only before CF-9a is written (it touches four resolvers, the opening alloc, `stockFlow` and five suites). [J]
2. **Existing batch books are a data question the draft does not ask.** Old Physical Stock vouchers with one line per batch (and no `countPart`) were booked line by line against the item's balance and are
   in the books as 12 becoming 7 (3.2.2). Adding the count-group rule does not touch them (it reads only flagged lines), and **changing how they read would restate history**. The build needs a Data Check line
   ("Physical Stock vouchers with more than one line for an item at a godown: N") and an owner decision, not a silent fix. [J]
3. **The line is the carrier, and an older app erases what it does not know.** CF stores variants in the line's `udf` and `countPart` beside it; a serial list would be a second line bag. The draft's **server
   rule** restores values an older writer omitted, for a push that does not state `caps: ['udf2']`: `UDF_CARRIERS.vouchers.lines` names `inventory` as the line path, `udf` as the bag, `keep: ['countPart']`, a weak
   identity `['stockItemId','godownId','batchName']` and a **strong** identity that adds `qty`, `rate`, `amount` (§9.3; built in `.wt-cf/backend/src/conflict.js`: `UDF_CARRIERS` with that `keep` and both identities, `carryUdf`, `UDF_CAP = 'udf2'`; the client marker is `.wt-cf/src/sync/caps.ts`). **Serials need that protection more than `udf` does**: an older client that alters a voucher and drops the serial list changes stock and frees the IMEIs, silently. `keep` is the natural
   place for a `serials` key, and the strong identity pairs a line whose quantity an older app left alone; a line whose quantity it **changed** cannot be paired, which for a serial list also means the count
   and the quantity disagree. So the capability marker, the `current` read, the line-carry rule, the REST carry (`restWrite.js`) and the backup version (CF5: version 3) must be designed for both at once, as one
   carry function with two bags. A serial **array** on one line is unambiguous under the weak identity, which is one more reason the audit leans to arrays over one line per unit. [C][J]
4. **The backend is being restructured, and none of it is deployed.** In the worktree `backend/src/sync.js` goes from 496 to 295 lines and its push body moves into new `pushRun.js` (takes the tenant lock, driven
   by a fake client in `scripts/assert/customfields-sync.assert.ts`), `histories.js` (the reads, incl. the new `current` read), `mismatch.js`, and `restWrite.js` (the REST carry); `conflict.js` changes by 343 lines against its base. That base, `baa0c45`, is the **pre-gate snapshot** of wave 3: against the committed `2cee555` it lacks 84 files and 3,338 inserted lines of gate fixes
   (`undo.ts`, `syncGuard.ts`, `ewb.ts`, `importMap.ts` ... `git diff --stat baa0c45 2cee555`), though only 9 lines of `VoucherScreen.tsx` and 33 of `backend/src/conflict.js`; the worktree will need a rebase before it
   merges. Every `backend/src/sync.js` line number in section 3.10 is the **main** file, which is the one on the VM. Serial detection belongs in the restructured shape, behind the same fake-client test
   seam. [C]
5. **One writer for the voucher grid, in this order: Back's `Box` edits (built), CF-2, CF-3, CF-9c, then batch / serial entry**, or better, **one mechanism**. CF-9c needs exactly what batch and serial entry
   need: a cell drawn only for tracked items, a "stock in this position" hint in the picker, a negative-position warning that names the position, a Stock Journal row that starts from the source row's
   position, a Physical Stock line that counts exactly the position it names (now: pre-filled rows saved as a count group), a manufacturing rule, an "opening by" panel, a "reclassify" Stock Journal draft, a
   Data Check line. Building these three times (variant, batch, serial) in one file is how the walk gets three subtly different behaviours. A "stock dimension" abstraction generalised over variant (a choice),
   batch (a name with attributes) and serial (a list whose quantity is its length) is the cheaper path; the differences are the cell's editor and cardinality, not the machinery. [J]
6. **Reports.** `stockFlow`'s godown cells (`stockSummary.ts:122-160,216-222`) and the Stock Summary's group modes are where a batch dimension would sit next to CF-9's; its `'field'` mode (AD.3.8) has a
   `viaBatch` sibling. CF-4b waits for the report grid; a batch report written now is bespoke. [C][J]
7. **Costing.** CF24's pooled per-item costing is consistent with what the engine does for batches today (cost per item, not per batch, 3.2.2) and means no figure outside the new screens moves. It does not
   serve serials (each unit its own cost) and does not remove the lot cost of 3.2.5: CF-9e's per-variant costing would multiply lot arrays and would hit the same cliff. [C][P][J]
8. **The reclassification trick has a batch-sized hole.** CF-9d moves "Not set" stock into variants with a value-neutral Stock Journal. The same trick for batches would run into the Stock Journal's missing
   batch on the receiving leg (3.2.2, transfer row), unless that is fixed first. [P]
9. **Older readers include the MCP server and any un-updated Electron install.** A count group read by a build that does not know `countPart` gives a different item quantity (the draft says so for its own
   release note). The MCP bundle is such a reader until its engine lock is re-recorded (4.5). [C][J]

**Other overlaps.** CF-2 and CF-3 (the strip and the line columns, the `cf:` focus, the Tab and Shift+Tab branch in `case 'item'` behind `lineFieldCount > 0`) edit the same functions as a serial panel or batch
picker (`itemRowFromLine`, `validInvLines`, the stock-journal grids); neither is built in the worktree yet (`cf:` appears nowhere in `.wt-cf/src/ui/VoucherScreen.tsx`). CF-0's `saveVoucher(v, { udf })` options
object (§9.1, rows A5 and A8) is the funnel a serial check would join. CF's own writer table (§9.1: A1 to A12, B1 to B6) is the list of writers a serial field must also be carried through; it is the same list as
section 3.4 plus the cheque, BRS and e-invoice patchers that spread the stored document. Bulk Entry's pure half moved to `core/bulkPlan.ts` in the worktree, so the batch refusal now lives at
`.wt-cf/src/core/bulkPlan.ts:462` and the screen `BulkEntry.tsx` is a drawing shell. [C]

### 4.3 The Back control (built in the worktree, unreviewed)

B-2a is built there: `Box` takes `onClose` (`.wt-cf/src/ui/widgets.tsx:566`), the batch dialog passes it (`.wt-cf/src/ui/VoucherScreen.tsx:919`, `onClose={onCancel} closeLabel="Cancel"`), and `src/ui/boxClose.ts`,
`backNav.ts`, `BackButton.tsx` exist. The rule in the spec is that every dialog a batch / serial / label build adds (a serial panel, a batch picker, a label dialog) must pass `onClose`, be listed in the spec's
dialog table (§5.2), and register a dirty probe if it holds typed or scanned work (B5, B12); a dialog that holds **scanned** work must not close on a stray click or Esc. The main-tree `Box` is
`({ title, children })` (`widgets.tsx:502`) and `BatchPrompt` passes nothing, so the first merge of the worktree changes every dialog's signature. [C]

### 4.4 Mobile research

Camera scanning is a phone strength and needs HTTPS first (section 3.14). A mobile device writing vouchers is **another offline device** for the serial problem
(`docs/research/mobile-backend-readiness.md:250`, R2: mobile may write vouchers, orders, receipts and van-sale invoices). The same document asks for a batch-claim call (`:471`); a serial claim should use the
same call shape. REST list and read scan the whole change log with `DISTINCT ON` (3.18 s at 250k vouchers, `docs/research/mcp-data-audit.md:88`): serials embedded in vouchers add nothing to that, a separate
collection would add another such scan. [C]

### 4.5 The MCP server (committed `d7846ce`, not deployed): four hard couplings

The MCP server (`mcp/`, `backend/src/mcpRoute.js`) lets a customer's own AI tool read the books. It is built and reviewed, not released, not deployed (`HANDOFF.md:671`), and it is **bound to the app's source** in
ways that bite this work. [C]

1. **A hash lock on 52 `src/` files.** `mcp/engine-lock.json` records the SHA-256 of every `src/` file the stdio bundle imports, among them `src/types.ts`, `core/inventory.ts`, `valuation.ts`,
   `openingStock.ts`, `integrity.ts`, `splitCompany.ts`, `syncGuard.ts`, `ewb.ts`, `einvoiceDoc.ts`, `vchNumbering.ts` and `voucherCancel.ts`. The lock's own note says a changed engine file fails the drift check until the lock is re-recorded, which forces a re-reconciliation before the next release (`mcp/scripts/drift.mjs:10` says the same of its `engine-lock` check). The one import door is `mcp/src/engines/index.ts` (it re-exports `stockPositions`, `stockValueAsOn`,
   `stockItemRows`, `openingGodownMap`, `StockPosition`, `index.ts:27-28,49`). **Every lane that edits one of these files owes the MCP gate a re-lock**, and the lock records the commit it was made at (`687c062`).
2. **A classification test on `src/types.ts`.** `mcp/test/fields.test.ts` parses every loaded interface and fails on any key that is in neither `keep` nor `drop` in `mcp/src/data/fields.ts`
   (`:22-30`). New keys on `StockItem`, `InventoryLine` or `Voucher` force a decision there; an unknown key is never copied in by the projection (the file's own header). Today `InventoryLine` keeps `batchName`
   and drops the rest of the batch fields (`fields.ts:98,294`).
3. **The write planner refuses batch items** (`mcp/src/write/buildVoucher.ts:281`; the tool description says "entries involving ... batches ... are refused", `descriptions.ts:65`). A serial-tracked item
   needs the same refusal, or a real implementation, in the same release; otherwise an AI-drafted sale of a serial item would carry no serial.
4. **A `stock_report` tool** reads `stockPositions` with no options and `stockItemRows` (`mcp/src/tools/stockReport.ts:2,43,67`): a batch or serial dimension in the app has no MCP counterpart until
   someone adds one, and **a count group's quantity is read by the locked copy of the engine** (4.2, item 9).


---

## 5. Ranked risks

Rank = (damage if it ships wrong) x (how hard it is to fix later). **The numbers are the first version's, kept because three sibling documents cite them (2, 10 and 15)**; the **Rank** column is the current order.
Rows 18 and 19 are new in this revision. Each row has the mitigation the code points to.

| # | Rank | Risk | Why it ranks here | Mitigation the audit supports |
|---|---|---|---|---|
| 1 | 1 | **Serial rows stored as their own synced table** | Every one of ~12 mechanisms (outbox, push batching, pull plus full reload, backup, restore, undo, cancel release, Mismatch Report, REST scans, `TABLE_AREA`, access, drift suites) would need a bespoke path; at 1M serials about 0.7 GB of table and outbox rows and 3,334 push requests (3.9). A table cannot be taken back after customers have data in it. | Serials on the line (array or per-unit sub-records); a derived, rebuildable **index** (in memory; and server-side events) rather than a source-of-truth table. |
| 2 | 2 | **Duplicate serial across devices and writers goes unnoticed, then an invoice is wrong** | No rule sees it (3.10); six local writers and REST bypass `saveVoucher`; policy forbids rejecting a push. Customer-facing and legally awkward (two invoices, one handset). | Detect at the write funnel (a second guard beside the operator guard, pure function, three callers), index serial events on the server inside `/sync/push`'s lock (migration `009`, after `planPush`, `sync.js:222-241`), new kind `duplicate-serial`, `base` to classify concurrent vs knowing, reserve-at-Accept with provisional fallback (numbering's deployed pattern). Honest copy: "offline devices cannot be stopped, only caught". |
| 18 | 3 | **An older app, a locked copy of the engine or a REST writer erases or misreads serials and count groups (new)** | The draft carry rule exists only for a writer that fails to state a capability (`§9.3`), the line-pairing rule cannot pair a line whose quantity an older app changed, and an older `stockMovements` reads a count group "last line wins" (4.2 items 3 and 9). The MCP bundle is such a reader until re-locked (4.5). A dropped serial list frees IMEIs silently. | One carry function with two bags (`udf` and serials, `keep: ['countPart', 'serials']`), one capability marker, one backup schema number, one REST carry; a release order that puts the engine build everywhere before the first count group or serial is written; Data Check lines for both. |
| 3 | 4 | **Two extensions of the same Physical Stock lines: CF-9a's count group and a batch count (reworded)** | The CF draft now handles a count made in parts with `countPart` and a merge in `stockMovements` (4.2); a batch count has exactly the problem of 3.2.2 (12 becomes 7) and a cure that is the same mechanism. If the two are built separately they will collide in `stockPositions`, `resolvedMovements`, `stockItemRows` and `ageing.tsx` (four readers) and an item that is both batch- and variant-tracked has no cell. Cheap to align before CF-9a is written, expensive after. | Reuse `countPart` and the `stockMovements` merge for batch (and serial) counts, generalise `countDeltas` over one **cell key**, update `byBatch` from `parts`; agree it with the CF lane-I owner before CF-9a starts; name the behaviour change for batch counts in the commit and in CF-9a's identity gate; leave history alone and add a Data Check line (risk 5). |
| 4 | 5 | **Serial goods under FIFO or LIFO freeze every Accept (cause corrected)** | 16 to 24 s at 125k lots [P]; the pass runs on each save and date change (3.2.5). The cost is `lotsValue`'s full sum on every movement, so LIFO is as slow; replacing `shift` alone does not help. | Specific-identification costing for serial items (no lots at all); or an incrementally maintained lot value **and** a deque before any serial item can be FIFO; keep serial state out of `stockPositions`; gate on a perf row with one-unit lots at 100k and 250k. |
| 5 | 6 | **Batch arithmetic is wrong in common flows, and a Busy user will find it first** | Physical Stock by batch turns 12 into 7 [P]; transfers and returns drop batch quantity; opening stock is in no batch; no per-godown batch cell; case-sensitive names (3.2.2). **Old Physical Stock vouchers with a line per batch are already booked that way** (4.2 item 2): a fix that re-reads history restates figures. | One `(godown, batch)` cell model with an explicit no-batch cell; counts by cell through the count group; transfer and return legs must carry the batch; normalise and pick batch names; an assertion that cells sum to item quantity at every checkpoint (the godown assertion is the template, `inventory.assert.ts:375-382`); a Data Check line for old multi-line counts and an owner decision on them. |
| 6 | 7 | **The voucher screen is the single hottest shared file and four efforts want it** | CF-2 / CF-3 / CF-9c; this work; Back's edits (built in the worktree); and the worktree's base is 84 files behind the committed wave 3 (it needs a rebase). A second writer means a lost edit (the Edit tool refuses stale files, the CF plan's own words). (3.3, 4.2-4.3) | Sequence: rebase the worktree, merge Back, then ONE voucher-screen writer for CF-2, CF-3, CF-9c and batch / serial entry, ideally as one "stock dimension" mechanism rather than three (3.3 seams listed; 4.2 item 5). |
| 7 | 8 | **A scan lands in the wrong place and does something** | narration Enter accepts; the Accept bar answers Yes to a scan's Enter; letters are hotkeys; an unknown scan is silent and appends (3.14). | A scan detector at `keys.ts` before the stack (timing and terminator), a scan target model (only an item or serial field accepts a scan, anything else shows "Scan the item field" and swallows), an explicit "not found" message, a burst test as a gate. |
| 8 | 9 | **The batch is invisible and optional at entry** | a carried value must always be visible; hidden batches are how stock goes wrong silently (3.3). | Auto-open at the point the walk reaches a tracked item (only for an item that opted in), show the batch on the line, require it when the item says so, and clear it on item change; `BatchPrompt` becomes a picker for outward lines. |
| 9 | 11 | **Status honesty: the stub and the parity board** | Gateway > Inventory Books > B opens "This screen is being built..." in front of a pharma prospect; two rows say `covered` (2). | Ship the report with the first release or remove the three entry points; correct `parity-data.json` and `FEATURE-PARITY.md` in the same commit; do not count batch rows covered until the master, picker, validation and report exist. |
| 19 | 10 | **The MCP lock and classification gates (new)** | Four hard couplings (4.5): a SHA-256 lock on 52 `src/` files, a classification test on `types.ts` keys, a write planner that refuses batch items, a `stock_report` over the old engine. Not user damage, but a lane that forgets fails the MCP gate late, and the MCP is a count-group reader of the locked copy. | Every lane that edits a locked file lists the re-lock step in its exit gate; `mcp/src/data/fields.ts` and `buildVoucher.ts` are one-writer files owned by the lane that adds the keys; decide serial support or refusal in the MCP in the same release. |
| 10 | 12 | **Label printing has no custom-size, no silent, no 1-D path, and Windows drivers alias bars** | 3.6. A label that does not scan is worse than none. The only existing bridge to a printer is unrun on real Electron. | A pure label layout and barcode module with dot-aligned modules (asserted by an independent decoder, as `qr.ts` is), a custom `@page` path that is not the named table, an Electron IPC modelled on `pdf:save`, a calibration sheet (the cheque pattern), a real-hardware test the owner must supply. |
| 11 | 13 | **Interchange loses batch data, so a migrating customer arrives with a broken book** | Tally opening batches dropped, dates not exported, IMEI-as-batch not understood (3.7), e-invoice drops short batch names silently (3.8). | Read `BATCHALLOCATIONS` under a stock item, import `MFGON` and expiry when present, a "convert batch-of-one to serial" step, an e-invoice warning for a dropped batch, serial and barcode in the mapped importer. |
| 12 | 14 | **The unit of a serial line (one line or many) leaks into print, e-invoice, e-way, Mismatch Report and CF** | 3.4; and one line per unit meets a 250-line e-way caution and a 1,000-item e-invoice cap. | Decide once; the audit leans to one line with a serial array, expanded to one item per unit only inside `einvoice.ts` if the owner wants IMEIs on the IRN and only below the cap. |
| 13 | 15 | **Next backup schema and Dexie version collide with CF-0** | `BACKUP_SCHEMA_VERSION = 2` on main, the worktree takes 3 (3.7). | One owner for the number; if serials are embedded no new table and no Dexie bump are needed, which removes the clash. |
| 14 | 16 | **Rights and opt-in rules are forgotten for new screens, tables and rows** | admin-only fallbacks; "nothing chosen = app as it was" is asserted by identity (3.15). | Gate every new surface on a company or item opt-in; `SCREEN_AREA` for each screen in the same commit. |
| 15 | 17 | **Exports corrupt long digit strings** | IMEI and EAN become numbers or scientific notation (3.5). | Serial and barcode columns typed as text in `reportSheet.ts` and the CSV writer. |
| 16 | 18 | **No test or recipe exercises a batch item in the UI** | none exists (2, 3.17). | A sample company with batch and serial items (the sample is `src/core/demo.ts` plus `scripts/seed/`), recipes that walk Purchase, Sale, Return, Transfer, Physical Stock and Cancel with real keys. |
| 17 | 19 | **Delivery Note plus invoice double-moves stock; serials would false-alarm** | no tracking link (3.2.3). | Decide link or document; do not flag a serial that is on a Delivery Note and its own invoice. |

---

## 6. What a build would touch, grouped into lanes

**Rule (the repo's own, `docs/research/wave-a2-plan.md` A2-5, quoted by the CF plan):** one writer per file at any moment. Lanes below are disjoint file sets that can run at the same time; the **one-writer
files** are listed after them. **Any lane that edits a file in the MCP engine lock (4.5) adds one step to its exit gate: run the MCP drift check and, once the reconciliation suites pass, re-record the lock.**

Names in *italics* are proposals `[J]`; everything else exists today.

### Lane M: pure model and engines (new files; the safest to start)

| File | Change |
|---|---|
| `src/types.ts` | **one writer** (also CF lane M; MCP-locked): item tracking mode (none / batch / serial), barcodes, MRP and shelf-life fields, `InventoryLine` serials and batch fields, `CompanyFeatures` key, opening alloc by cell and serial, `VoucherType` label option; each key is classified in `master-resave`, `voucher-resave` (worktree) and `mcp/src/data/fields.ts` |
| `src/core/inventory.ts`, `src/core/valuation.ts`, `src/core/openingStock.ts`, `src/core/stockSummary.ts` | **the CF plan's lane I (one writer per file; all MCP-locked except `stockSummary.ts`); the shared cell key and the count-group reuse are agreed with it BEFORE CF-9a starts**: cell balances, the count group for batch counts, `stockFlow` cells, the lot structure (no per-movement sum), specific-identification costing |
| *`src/core/stockAgeing.ts`* | CF-9a moves Stock Ageing's lot walk here unchanged; a serial age and a batch age read it |
| *`src/core/stockCell.ts`, `batches.ts`, `serials.ts`, `serialIndex.ts`* | cell keys, batch rules (normalise, FEFO, expiry, suggest), the serial state machine as of a date, the lazy index, the duplicate detector |
| *`src/core/barcode/{code128,ean,gs1,scale}.ts`* | encoders, EAN / GTIN check digits, scale-embedded weight or price parse; with an independent decoder for tests (as `qr.ts`) |
| *`src/core/labelLayout.ts`, `labelSheet.ts`* | label geometry, dot alignment, N-up sheet maths, calibration |
| `src/core/splitCompany.ts` (also lane I: CF-9d carries variants; MCP-locked), `recall.ts`, `integrity.ts` (MCP-locked), `syncDiff.ts`, `syncGuard.ts` (MCP-locked), `tallyImport.ts`, `tallyExport.ts`, `importMap.ts` | carry serials and cells in a split; drop on recall; Data Check lines (duplicate serial, negative batch cell, expired in stock, old multi-line counts); a `serial` row label; the new kind; opening batches in; dates out; item columns |
| `scripts/assert/*` | new suites for each file above with mutation files; extend `inventory`, `master-resave`, `split`, `tallyimport`, `tallyexport`, `syncguard`, `syncconflict`, `integrity`, `stocksummary` (3.17) |

### Lane V: voucher entry (**single writer for `VoucherScreen.tsx`**)

`src/ui/VoucherScreen.tsx`; new `src/ui/vch/{BatchPicker,SerialPanel}.tsx`; `src/ui/inventory/manufacturing.tsx` (batch / serial on consumption and output); `src/ui/admin/BulkEntry.tsx` (and, once the worktree merges,
`src/core/bulkPlan.ts`: batch and serial columns, remove the refusal); `src/core/shortcutRegistry.ts` (any new chord, one writer); `src/core/voucherConfig.ts` and `src/ui/VoucherConfig.tsx` (a Batch or Serial column switch, appended after the last field); `src/state/dirty.ts` probes. **Order on the file: Back's `Box` edits (built in the worktree), CF-2, CF-3, CF-9c,
then this**, unless the owner of CF-9c builds the generic stock-dimension cell (4.2 item 5) and batch and serial ride in it. Must follow Back's `onClose` on every new `Box` and CF-3's Tab contract and `ItemCol` union.

### Lane S: scanning input

`src/keys.ts` (**one writer**; the worktree adds `runEscape()` there, `.wt-cf/src/keys.ts:131-146`), *`src/core/scan.ts`*, *`src/ui/scan/*`* (a scan target model, a "not found" lane message through `ui/feedback.tsx`), the item
resolver (exact barcode lookup) in `VoucherScreen.tsx` (lane V's file; coordinate), the palette. A burst-test recipe under `scripts/shots/`.

### Lane R: reports and screens (**one writer for the new-screen checklist**)

Replace `src/ui/inventory/BatchSummary.tsx`; new *`src/ui/inventory/{batches,serials,expiry}.tsx`*; extend `stockquery.tsx`, `stocksummary.tsx`, `godownstock.tsx`, `ui/admin/Exceptions.tsx` (negative cell lines); a serial history (the
life of one IMEI) screen; the dashboard widget (`src/ui/dash/widgets/registry.ts` + a compute file); the checklist files, once: `src/state/store.ts` (`Screen` union only), `src/ui/router.tsx` (route and crumb), `src/core/access.ts`,
`src/state/recents.ts`, `src/ui/reportsCenter/catalogue.ts`, `src/ui/overlays.tsx`, `src/ui/modern/CommandPalette.tsx`, `src/ui/MenuScreen.tsx`, `src/ui/navTree.ts` with the literal table in `scripts/assert/sidenav.assert.ts`,
`src/ui/quickCreate.ts`. Also `src/ui/inventory/masters.tsx` (tail fields in their own `tailGroup`: tracking mode, barcodes, MRP; **frozen field order**), the batch master screen if there is one, and `docs/parity-data.json` +
`docs/FEATURE-PARITY.md`.

### Lane P: print and labels

`src/core/printConfig.ts` (custom sheet), `src/styles/print.css` (named page or injected style), `src/ui/print/invoiceModel.ts` and `PrintVoucher.tsx` (batch, serial, MRP lines), new *`src/ui/labels/*`* (designer, sheet preview,
print), `src/ui/reports/selectionBar.tsx` and `src/core/bulkPrint.ts` (a "print labels for the selected" sibling of the `I` key), `src/core/einvoice.ts` (batch warning, warranty date, serial, barcode),
`electron/main.cjs`, `electron/preload.cjs`, `electron/exportBridge.cjs` (**one writer**; a `print:label` IPC validated like `pdf:save`), `scripts/assert/exportbridge.assert.ts`, `print.assert.ts`.

### Lane X: exchange

`src/ui/admin/ImportData.tsx`, `MappedImport.tsx`, `src/core/importMap.ts` (serial purchase from Excel, opening by batch and serial, barcode column), `src/core/tallyImport.ts` / `tallyExport.ts` (**CF-10 also owns these**:
sequence), `src/core/reportSheet.ts` and `src/core/csvOut.ts` (text typing for long digit strings).

### Lane B: server and sync (**gated on the owner's VM secrets**, deploy with CF-6a)

**Build on the CF-6a structure, not the file as it is in the main tree**: the worktree splits `backend/src/sync.js` into `pushRun.js` (the push body, testable with a fake client), `histories.js` (the reads), `mismatch.js` and
`restWrite.js` (the REST carry), adds `caps` / `udf2` to the push body (`src/sync/caps.ts`) and a `carryUdf` rule for lines. Files: `backend/src/conflict.js` (kind `duplicate-serial`, resolutions, a serial bag beside `udf`
in `UDF_CARRIERS`, `DATA_TABLES` only if a table), `pushRun.js` (serial events under the push lock), `histories.js` (a `current` read that also serves serials), `restWrite.js` / `rest.js` / `restPolicy.js` (REST writes of vouchers
with serials), new *`backend/src/serials.js`*, new *`backend/sql/009_*.sql`* (008 is taken and deployed), `src/sync/sync.ts` (post-pull serial scan beyond the 5,000 cap, claims) and `src/sync/caps.ts` (a serial capability marker), new
*`src/sync/serials.ts`*, `src/ui/admin/SyncConflicts.tsx`, `src/sync/mismatch.ts`, `scripts/assert/syncconflict.assert.ts` and CF's `customfields-sync.assert.ts`. **One deploy window with CF-6a**, gated on the owner's VM
secrets (the pending wave-B rule is already deployed). A deploy-time smoke on a throwaway tenant (the CF plan's pattern, `scripts/smoke/customfields-deploy-smoke.mjs` in the worktree) closes the gate, because the suites cannot
cover the SQL.

### Lane D: data, backup, store

`src/data/db.ts` (**one writer**; only if a table or index is added: `version(9)`), `src/core/backupMigrate.ts` (the step number), `src/state/store.ts` (`COLLECTIONS`, `loadInto`, `checkDelete`, the `saveVoucher` funnel and its
options), `src/data/writeGuard.ts` (a second guard), `src/core/undo.ts`. If serials are embedded none of `db.ts`, `backupMigrate.ts` and the table plumbing changes; that is the single biggest lane-saving the embedded choice buys.

### Lane Q: the MCP server (new)

`mcp/src/data/fields.ts` (keep or drop for each new key, **one writer**), `mcp/src/write/buildVoucher.ts` (refuse or support a serial item; line 281 today), `mcp/src/tools/stockReport.ts` (a batch or serial reading, optional),
`mcp/src/tools/descriptions.ts` and the tool-list golden if a tool changes (the changelog says the list is a contract), `mcp/engine-lock.json` (re-recorded by `mcp/scripts/drift.mjs --write-lock` **only** after the
reconciliation suites pass), `mcp/test/fields.test.ts`. Runs after every other lane has settled its `src/` files.

### Files that must be **one writer at a time** (the list)

`src/types.ts` · `src/ui/VoucherScreen.tsx` · `src/core/inventory.ts`, `openingStock.ts`, `valuation.ts`, `stockSummary.ts`, `splitCompany.ts` (the CF plan's lane I, with CF-9) · `src/state/store.ts` (`Screen` union,
`saveVoucher` options) · `src/ui/router.tsx` · `src/core/access.ts` · `src/core/shortcutRegistry.ts` · `src/ui/navTree.ts` + `scripts/assert/sidenav.assert.ts` · `src/ui/quickCreate.ts` ·
`src/ui/reportsCenter/catalogue.ts` · `src/ui/overlays.tsx` · `src/ui/modern/CommandPalette.tsx` · `src/data/db.ts` · `src/core/backupMigrate.ts` · `src/keys.ts` · `src/ui/widgets.tsx` (Back adds `onClose` here) ·
`src/sync/sync.ts` · `backend/src/conflict.js`, `sync.js`, `pushRun.js`, `histories.js`, `restWrite.js` and `src/sync/caps.ts` (with CF-6a) · `electron/main.cjs`, `preload.cjs`, `exportBridge.cjs` ·
`src/styles/print.css` · `src/core/tallyImport.ts` / `tallyExport.ts` (CF-10) · `src/core/einvoice.ts` · `src/ui/dash/widgets/registry.ts` · `docs/parity-data.json` · **`mcp/engine-lock.json`, `mcp/src/data/fields.ts`**.

### Order `[J]`

1. ~~wave 3 gates and commits~~ **done** (`2cee555`, deployed 2026-10-23); **rebase the `.wt-cf` worktree onto it** (84 files and 3,338 inserted lines of gate fixes behind; 9 lines in `VoucherScreen.tsx`, 33 in `conflict.js`);
2. Back's slices are largely built in the worktree (`Box` `onClose`, `runEscape`, `BackButton`; unreviewed): review and merge them first, since they edit the voucher file's dialogs;
3. in parallel: **Lane M** (pure files, no UI) with **Lane B's design and migration** (no deploy) and **Lane P's pure parts** (barcode encoders, label layout);
4. **before CF-9a is written**: agree the shared cell key and the reuse of `countPart` with the CF lane-I owner, the backup schema number and the carry function with two bags (CF-0 and CF-6a); then CF-0, CF-9a and CF-6a land
   (the `saveVoucher` options included);
5. **Lane V** (one writer for the voucher screen: CF-2, CF-3 and batch / serial entry as one sequence) with **Lane R** (screens) and **Lane X**;
6. **Lane S** (scan detector) once the item field it targets is settled;
7. **Lane Q** (re-lock the MCP engine, classify the new keys) after the `src/` lanes settle;
8. Lane B deploy and Lane P Electron IPC last (they need the owner: VM secrets, a label printer, a scanner).

---

## 7. What I could not verify

- **No hardware.** No scanner, no label printer, no phone. Every statement about how a keyboard-wedge scanner behaves (speed, suffix, Caps Lock, Alt-codes) is `[J]` and `[unverified]`; every statement about thermal printers
  and drivers is `[J]` and `[unverified]`.
- **No browser run.** The scan-burst behaviour in section 3.14 is read from the key dispatcher and the field handlers, not driven. Whether the entry screen keeps up with a burst, and what the focus effect does when a scan
  arrives mid-render, is unmeasured. IndexedDB behaviour of a multi-entry index is likewise untested. I did not start the dev server (it is not mine) and ran no screenshot recipe.
- **No competitor account.** Nothing here says what Busy, Marg or Tally actually do beyond what the repo's own notes and the Tally help citations already say; I make no claim about Busy's screens. The brief's description of Busy
  is the owner's.
- **No Postgres.** The server-side serial-event design is reasoned from reading `sync.js`, `conflict.js`, `numbering.js` and the migrations; I did not run it. `loadHistories` and the advisory locks are unit-untestable without
  a database (the CF plan says the same).
- **No real Tally file.** The IMEI-as-batch practice, and what a real export contains under a stock item's batch list, are unverified.
- **The CF-9 design is a draft** the CF author marks "reviewed once" (the addendum), and the worktree has no CF-9 engine code. I read AD.3, the slice and lane rows, §9.3 and §16 to §17 in full on 2026-10-23, and the rest of
  the addendum by heading. The file keeps changing; the alignment in 4.2 must be re-checked against it when the build starts.
- **The worktree's code is unreviewed and ungated** ("WIP checkpoint 2"). Where I cite it (`.wt-cf/...`) I say what is there, not that it works.
- **The MCP coupling** is read from `mcp/engine-lock.json`, `mcp/scripts/drift.mjs`, `mcp/test/fields.test.ts` and `mcp/src/data/fields.ts`; I did not run the MCP gate or its tests.
- **The e-invoice tag spellings** `PrdSlNo` and `Barcde` are not in the text I extracted from the schema PDF, which gives parameter names and lengths; the schema's own item-count limit and the exact IRP validation of the
  serial and barcode fields were not read (the repo's own validator caps `ItemList` at 1,000, `einvoice.ts:414,670`).
- **AS 2** paragraph numbers come from a search summary; the ICAI PDF would not open (certificate error). The accounting judgement on whether handsets are "ordinarily interchangeable" is for the owner's CA.
- **Probe scope.** The engine probes use synthetic books I built (Appendix A). They show the engine's behaviour for those inputs; they are not a measure of a customer's book, and wall times move 1.5x to 2x with machine load.
  The 250k-voucher harness numbers are the repo's own (`docs/PERF.md`), not re-run.
- **Web research.** None this revision: every external fact is carried from the first version's fetches and searches (Appendix B), and the statutory research belongs to the sibling documents.

---

## Appendix A. The probes `[P]`

Scratch only: written under the session scratchpad, bundled with the repo's `node_modules/.bin/esbuild` (`--bundle --platform=node --format=esm`), run with Node 25.2.1. They import `src/core/inventory.ts` as it is at `7074445`;
nothing in the repo was edited or run through `npm`. Each builds a company by hand and calls `stockPositions` (and, for the first version, `resolvedMovements`) from `src/core/inventory.ts`. **Re-run on 2026-10-23: the batch results
below are identical to the first version's; the timing results moved with machine load and one conclusion changed (3.2.5).**

**Probe 1: batch arithmetic** (items are `FIFO` or `AvgCost`, `batches: true`, godown `<cid>-gd-main`)

| Case | Setup | Output |
|---|---|---|
| 1 | item opening 10 @ 100, Purchase 5 as B1 (exp 2027-01-01) | `qty 15`, `byBatch [B1: 5]` |
| 3 | Purchase 10 B1 @ 100, Purchase 10 B2 @ 150, Sales 10 of B2 (FIFO) | moves: purchase +1000, purchase +1500, sale **-1000**; closing value 1500, qty 10; `byBatch B1 10, B2 0` |
| 4 | Purchase 2 B1, Purchase 20 B2, Sales 5 of B1 | `qty 17`, `B1 -3`, `B2 20` |
| 5 | Purchase 5 `B-17`, Sales 5 `b-17` | `B-17: 5`, `b-17: -5`, item qty 0 |
| 7 | Purchase 5 B1 exp 2027-01-01, Purchase 5 B1 exp 2026-06-01 | `B1: 10, expiry 2026-06-01` |

**Probe 3: Physical Stock, transfer, return** (AvgCost)

| Case | Setup | Output |
|---|---|---|
| count by batch, both lines | Purchase B1 5 and B2 7; Physical Stock lines B1 = 5, B2 = 7 | `qty 7`, `B1 -2`, `B2 9` |
| count lists only B1 | Purchase B1 5, B2 7; Physical Stock B1 = 5 | `qty 5`, `B1 -2`, `B2 7` |
| transfer | Purchase B1 5; Stock Journal out 2 B1 from Main, in 2 at `g2` with no batch | `qty 5`, `B1 3`; godowns `Main 3, g2 2` |
| return | Purchase B1 5; Sales 2 B1; Credit Note 1, no batch | `qty 4`, `B1 3` |

**Probe 2: cost of the lot array and of batch labels** (one item, 1 unit per voucher, same date; `stockPositions` wall time on an Apple M-series laptop, one run each, two sessions)

| Method / shape | ms |
|---|---|
| AvgCost alternating, 250,000 vouchers | 44 |
| AvgCost alternating + 500 batch names | 44 |
| FIFO alternating (one lot open at a time) | 26 |
| FIFO + 500 batch names, alternating | 34 |
| AvgCost buy 125,000 then sell 125,000 | 27 |
| FIFO, 10,000 / 20,000 / 40,000 / 80,000 lots open | 391 / 1,213 / 3,071 / 9,302 (second run, busier machine); 362 and 1,481 for 20,000 and 40,000 (first run) |
| LIFO, 10,000 / 20,000 / 40,000 / 80,000 / 125,000 lots open | 174 / 518 / 1,988 / 8,477 / 20,484 |
| FIFO 125,000 lots open | 16,285 (first run) |

The first version's alternating FIFO figure (3,767 ms) came from a book whose purchases clustered ahead of its sales (more lots open at once) and is not comparable with the alternating row above; the open-lot rows are.

**Probe 4: payload and index sizes** (plain Node, synthetic)

| Quantity | Value |
|---|---|
| in-memory `Map` of 1,000,000 serial keys (`item|imei`) | 393 ms build, about 94 MB heap (first version: 720 ms, about 104 MB) |
| JSON of an inventory line with 4 serials | 181 bytes |
| JSON of an inventory line with 5,000 serials | 90 KB |
| one serial as its own document (UUID ids, 7 fields) / its outbox row embedding it | 250 / 457 bytes; **about 707 bytes a serial; 1,000,000 rows about 0.7 GB**; 3,334 push requests at 300 per request |

**Probe 5: what a scanned code does in the quantity parser** (`parseQty` and `parseAmt`, `src/core/money.ts`, new in this revision)

| Typed or scanned text | `parseQty` | `parseAmt` |
|---|---|---|
| `8901234567890` (an EAN-13) | 8901234567890 | 8901234567890 |
| `356938035643809` (an IMEI) | 356938035643809 | 356938035643809 |
| `0123456789012` | 123456789012 (the leading zero is lost) | 123456789012 |
| `12/3` | 4 | 4 |
| `5 L` | 5 (a quantity never reads L as lakh) | 500000 |
| `(1000)` | -1000 | -1000 |

**Probe 6: a lazy serial index over a big book** (plain Node, synthetic, new in this revision)

| Quantity | Value |
|---|---|
| 250,000 vouchers (125,000 purchases, each followed by the sale of the same 4 units), 1,000,000 serial events, 500,000 distinct units: one pass building a `Map` keyed `item|serial` and flagging in-after-in / out-after-out | 237 to 248 ms (two runs); 0 duplicates |
| 100,000 point lookups | 44 to 47 ms |

To re-run: write each probe as a `.ts` file that imports from `/Users/Akshay/tallyreplica/src/core/inventory` and `types`, bundle with the esbuild command above, run the `.mjs`. The shapes of `StockItem`, `Voucher` and
`InventoryLine` used are the ones in `src/types.ts:488-527, 944-973, 991-1049`.

## Appendix B. Sources

- Form GST INV-1, "Format/Schema for e-Invoice", version 1.1, Rule 48 (the CGGST copy): https://www.cggst.com/uploads/document/1748345405-form-gst-inv-1.pdf `[W]` (text extracted locally; item barcode, unique serial number,
  batch details with expiry and warranty date, attribute details).
- A search over the same schema's JSON field names, including a GSP's and an invoicing vendor's published samples: https://docs.logitax.in/Docs/Standard_Create_E_invoice and
  https://sleekbill.in/images/articles/Json-format-e-invoice.pdf `[Ws]`.
- ICAI Accounting Standard 2 (Valuation of Inventories), paragraphs on specific identification and on FIFO or weighted average: https://kb.icai.org/pdfs/69232asb55316-as2.pdf `[Ws]` (the page failed to load with a
  certificate error; paragraph numbers 14 to 16 are the search summary's, with the Expert Advisory Committee opinions at https://eacopinion.icai.org/html/20/27.htm).
- Electron `webContents`: `print` (silent, deviceName, custom pageSize, margins, copies), `getPrintersAsync`, `printToPDF` (`preferCSSPageSize`): https://www.electronjs.org/docs/latest/api/web-contents `[W]` (first
  100,000 of 133,867 characters read; the unit of a custom `pageSize` for `print` is not stated on the page).
- MDN, `getUserMedia` (secure contexts only): https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia `[Ws]`; Barcode Detection API support: https://caniuse.com/mdn-api_barcodedetector `[Ws]` (the
  search snapshots disagreed on version numbers; check the live tables before relying on a cut-off).
- In this repo: `CLAUDE.md` (Phase 20 to 29 gotchas), `docs/HANDOFF.md` (§2f, §2g), `docs/PERF.md`, `docs/research/wave-a2-plan.md`, `docs/research/mobile-personas-and-jobs.md`,
  `docs/research/mobile-backend-readiness.md`, `docs/research/mcp-data-audit.md`, `docs/research/batch-serial-tracking.md`, `tracking-verticals-india.md`, `pos-code-audit.md`, `docs/design/left-nav-spec.md`, `mcp/*` (read-only),
  and, read-only, `/Users/Akshay/tallyreplica/.wt-cf/docs/design/custom-fields-spec.md` and `back-nav-spec.md` plus the worktree's source at `f446674`.
