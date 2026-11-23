# Batch, serial and IMEI tracking: how the strong systems do it, where TallyPrime is weak, and what would beat Busy

> **Status:** research and judgement only. Nothing here is built, and no code was changed. First written 2026-10-22 against the main tree
> (`/Users/Akshay/tallyreplica`, about 160 uncommitted wave-3 paths) and, read-only, the worktree `.wt-cf` (the custom-fields spec with its
> addendum of 2026-10-22, and the Back-control spec). **Second pass 2026-10-23:** wave 3 is now committed (`2cee555`); the web tools worked again, so the
> sources the first pass could not reach were read (the DoT/PIB IMEI release, the CDSCO drug-QR FAQ, the CBIC margin-scheme flyer, the BIS jeweller page, Marg's own
> help pages, Busy's label FAQ, Tally's Batch Summary and Stock Journal class pages, Zoho's batch-price, first-expiring and label pages, a page of Busy user reviews, the Zebra and TSC
> printer-language manuals), the worktree spec was re-read after its third review, and every code reference was re-checked. Section 0, item 9, lists what changed.
> **The owner's request (intent, 2026-10-22):** TallyPrime's batch tracking is weak; Busy's customers like its batch management; businesses such as mobile
> shops track an IMEI per unit; label printing and scanning are needed, where Tally is also weak; build these well.
> **Evidence tags.** **[W key]** a page I fetched and read (the key points at the source list at the end). **[Ws key]** a search summary only, the page
> itself was not read. **[C file:line]** read in this repository. **[J]** my judgement. **[unverified]** I could not confirm it. Every external fact is
> paraphrased. Statutory and regulatory facts carry the weakest tag I could honestly give them, and say so.
> **What I could not do (the full list is section 12):** I have no live Busy, Marg or TallyPrime to try and no hardware (no scanner, no label printer, no phone).
> GS1's own pages answered 403 (its position on the drug-pack code is a search summary only), I reached no forum or YouTube thread about Busy, and Marg's help pages are
> mostly navigation with the article body cut off. The Busy "customers appreciate it" premise is **only weakly confirmed**: one page of 20 user reviews mentions no batch work and
> one electronics seller praises serial and IMEI tracking (section 2.1).
> **Scope:** batch and lot tracking, serial and IMEI tracking, and the label and scan work they need. Variants (colour, size) are the custom-fields
> track's CF-9 slice and are only touched where the two meet (section 10).

---

## 0. Read this first

1. **Comsky today has the bones of batches and nothing of serials.** A batch is a name and an expiry typed into a two-field box that opens only on `Alt+B`
   [C `VoucherScreen.tsx:880-914`, `:3062`]. The engine counts stock per item and batch, **not per godown** [C `inventory.ts:14`, `:141-160`]. There is no mfg-date
   entry, no MRP, no batch picker, no expiry rule, no batch report (the "Batch & Expiry Summary" menu row opens a four-line stub [C `BatchSummary.tsx:1-4`;
   `MenuScreen.tsx:234`]), and no serial, IMEI, barcode or label code at all. The parity board nevertheless marks `batches` and `expiry-reports` as covered, and its notes still
   say "No batch field on item or line" [C `docs/parity-data.json`; `docs/FEATURE-PARITY.md:161`, `:171`]: the status and the code disagree and the status is the
   one that is wrong.
2. **TallyPrime is adequate for a pharmacist's batches and weak around them.** It has batch names with mfg and expiry, per-batch balances, a Batch Summary with
   expired-only and date-range filters, a Batch Vouchers report, an ageing view by days left to expiry, and a switch that hides expired batches at sale
   [W tally-batches, tally-batch-summary; Ws tally-expired]. It has **no serial or IMEI tracking** (the documented workaround is one batch per phone [W tally-erp9-unsold]),
   **no per-batch MRP** (the MRP help page I read never connects MRP and batch [W tally-mrp-page]), **manual batch choice**, a default that **lets expired stock be sold** [Ws tally-expired], **no expiry alert** [W tally-guide-godown],
   barcodes only as text in the item's Alias field (32 characters) and **no label printing of its own** [W tally-barcodes].
3. **Busy is the benchmark because it ships the whole chain, and its limits are visible in its own FAQ.** Batch: per-batch MRP and sale-price options, expiry warning days,
   an Expiring Batches report, an On-Account exception report, a FIFO auto-adjust utility. Serial: automatic or manual numbering, scanning, purchase, sale and installation
   warranty, a serial-wise stock report and an item serial query. Labels through an add-on [W busy-batch, busy-serial, busy-faq, busy-labels]. Its stated limits: batch and
   serial are **not in the Basic edition**; expired batches can be sold with a warning; batch choice is manual (no FEFO found); **duplicate serials are not
   supported**; label design is a **chargeable service**; barcode values are 5 to 20 characters; no scheduled report e-mails; batch numbers are added to old purchases one voucher at a time
   [W busy-faq 67, 72, 74; busy-labels].
4. **Odoo and Zoho each do things Busy does not document.** Odoo proposes lots by a removal strategy (FEFO, ranked by removal date) [W odoo-removal, odoo-fefo], takes an item's expiry from four day-counts
   (expiration, best before, removal, alert) [W odoo-expiry] and asks for "first serial plus a count" to generate a run [W odoo-serial]. Zoho keeps a price and an MRP **per batch** with four
   pre-fill methods [W zoho-batch-price], lists batches soonest-expiry first at invoice (a sort, not an auto-pick) [W zoho-first-expiring], opens any serial's full history by
   click [W zoho-returns], and forces one method per item (serial **or** batch) [W zoho-index].
5. **IMEI is not "a number you type".** It is 15 digits (an 8-digit type code, a 6-digit serial, one Luhn check digit), a dual-SIM phone has two, and **consecutive units do not differ by
   one**: from the check-digit rule, 490154203237518 is followed by 490154203237526, 490154203237534 and 490154203237542 (computed in section 3.1). A range tool that adds one
   to the whole number generates invalid IMEIs. Logic ERP's bulk entry (a start IMEI plus a quantity) is the only range feature I found, and its page says nothing about the
   check digit [W logic-imei].
6. **The integrity rules that matter most** (section 7): a serial is held by at most one thing at a time and is sold at most once until it comes back; a batch never goes below zero
   in a godown; an expired batch cannot be sold unless an administrator says so; quantity equals the serial count; a duplicated or recalled voucher never copies serials; and two
   devices must not both sell the same IMEI. The leaders are permissive on several of these by default; being strict by default is a real, cheap way to be better.
7. **Constraints in this repository** (section 9): the custom-fields slice CF-9 changes the same eight engine functions in the same weeks; there is **no link from a
   Delivery Note to its invoice**, so a serial delivered and then invoiced would be issued twice; an older client that re-saves a voucher drops any line field it does not know, which is
   exactly how serials would be lost; and a dialog that holds scanned work must not close on a stray Esc or click.
8. **Ranked bets** (section 10, which has the evidence and the dependencies): (1) a unit-level engine with real integrity, (2) per-godown batch positions with a batch master (mfg, expiry, MRP, cost), (3) proposed first-expiry-first-out at sale with a
   visible override, (4) expiry rules that actually protect, (5) scan-first capture with check-digit validation, (6) a way over from Tally (import batch openings, convert a batch-of-one to a serial), (7) a "where is this unit or batch" lookup and recall trace, (8) stock-take by scanning with a
   difference list, (9) warranty as data, (10) labels from the grid; then batch and serial on one item, pharma extras, unit and batch profit, and the used-phone margin scheme (accountant first). The first two are structural and everything else stands on them.
9. **What the second pass (2026-10-23) changed, so a reader of the first draft is not misled.**
   - *IMEI law, now read from the release itself* (PIB Release ID 2190763, 17 November 2025): the three offences, the used-device database check and the 2025 amendment are stated in section 3.1.2.
     One correction: the release says retailers "should be aware" of the consequences of procuring tampered devices; it does **not** say they "must ensure" valid IMEIs, as the first draft wrote. The used-device check **is** stated as a duty, with a fee per check [W pib-dot].
   - *Scanning a drug pack to fill batch and expiry is not something a pharmacy can rely on.* CDSCO's FAQ lists the fields the code must carry but leaves the code's identifier to each manufacturer, and names no standard; GS1's
     position paper says many of the top-300 packs carry a QR that only encodes a **web address** to a vendor's page [W cdsco-faq; Ws gs1-india-position]. The first draft assumed a parseable payload. Sections 3.2, 6.4 and 10.3 change.
   - *Used-phone margin scheme*: the rule is now cited from the CBIC flyer [W cbic-margin] (section 3.1.4); the first draft only "recalled" it.
   - *Jewellery*: the HUID mark is mandatory, but **showing it on the invoice is "voluntary at present" and keeping a HUID record is not mandatory** [W bis-jewellers]; the first draft said the invoice should list it. Section 3.5.
   - *Schedule H1*: the circulated register format has **no batch or expiry column** [W nhsrc-h1]; the first draft said it did. (The batch and expiry are still required on the Schedule C, H and H1 sale record under Rule 65(3), as the sibling document read it [R]; the register and the sale record are two records.)
   - *Marg* now has first-party evidence (an expiry-ordered batch list, a Near Expiry / Expired report, a Batch Stock report), section 2.3; *Busy's* label limits and the user-review evidence are read, section 2.1; Tally's MRP, Batch Summary and Stock Journal class pages and Zoho's batch-price, first-expiring and label pages are read.
   - *The worktree spec moved*: the custom-fields addendum had a third review; a Physical Stock count by variant is now a **count group** read from a `countPart` flag, not resolved against a variant balance as section 1.3 first said. The same device is the right way to count by batch (section 9.2).
   - *New*: section 6.2 gains two rules from a Busy user's complaints (correcting a serial, scanning on a return), section 8.7 records what the Zebra and TSC manuals say about printing labels, and section 12 is rewritten.

---

## 1. Where Comsky stands today [C]

### 1.1 What exists

| Area | What the code does | Where |
|---|---|---|
| Item flag | `StockItem.batches?: boolean` ("maintain in batches (mfg/expiry)") | `types.ts:520` |
| Line fields | `InventoryLine.batchName`, `mfgDate`, `expiryDate` (ISO strings) | `types.ts:953-955` |
| Opening stock | `openingAlloc.batches: { name, qty, expiryDate? }[]`, written only by Split Company Data; the item master has no batch opening entry | `types.ts:482-486`; `splitCompany.ts:547-552` |
| Entry | `Alt+B` opens "Batch Details" with two fields (batch or lot no., expiry). It **never opens by itself**, so the Enter walk is unchanged. `onDone` writes `batchName` and `expiryDate` only: **a manufacturing date cannot be typed** (it arrives only by import or copy). No list of existing batches, no per-batch stock, no check that the batch exists or has stock | `VoucherScreen.tsx:880-914`, `:3062`, `:3398-3410`, `:4106-4113` |
| Engine | `byBatch: Map<name, { qty, expiry? }>` on the item's position: **one map per item, no godown dimension.** The expiry kept is the last non-empty one seen in date order. Physical Stock resolves the counted figure against the (item, godown) balance only, so a count ignores batches | `inventory.ts:14`, `:141-160`, `:104-112` |
| Costing | Valuation is per item; the header comments say godown and batch are ignored. There is no per-batch cost | `valuation.ts:52`, `:111`, `:217`, `:248` |
| Guards | The only stock guard is a save-time "will go NEGATIVE" warning grouped **by item**, with Continue anyway. No batch guard, no expiry guard, no "batch exists" guard | `VoucherScreen.tsx:2489-2505` |
| Reports | Stock Query lists batch balances for one item. `BatchSummary` is a `ComingSoon` stub behind a live menu row. Stock Ageing has no batch or expiry reading | `stockquery.tsx:89-140`; `BatchSummary.tsx`; `MenuScreen.tsx:234`; `router.tsx:282` |
| Print, e-invoice | The invoice prints "Batch: X · Exp. date" under the item when the batch column is on. The e-invoice sends `BchDtls` (name and expiry) **only when the name is 3 to 20 characters, silently dropped otherwise**; it never sends the warranty date `WrDt` or a per-item serial `PrdSlNo` | `ui/print/invoiceModel.ts:617`; `einvoice.ts:358-359`; grep of `einvoice.ts` finds no `PrdSlNo` or `WrDt` |
| Tally XML | Export writes `BATCHNAME` per allocation. Import reads the batch **name** of each allocation (the default "Primary Batch" is dropped) but **not its manufacturing or expiry dates** | `tallyExport.ts:783-789`; `tallyImport.ts:1273-1304`; grep finds no date tag in the importer |
| Bulk Entry | **Refuses** a batch-tracked item ("enter it on the voucher screen") | `BulkEntry.tsx:486-489` |
| Sync identity | A voucher line is identified by (item, godown, batch) in the Mismatch Report | `syncDiff.ts:53` |
| Serial, IMEI, MRP, barcode, label | **None.** No field, no screen, no encoder for a linear barcode. Only a QR encoder exists, for the e-invoice | grep of `src/`; `core/qr.ts`, `ui/print/QrCode.tsx`; parity row `mrp-pricing` is missing |
| Paper | Print pages are A4, A5 and Letter only; the Electron PDF bridge offers the same three. There is no label-size page | `styles/print.css:34-43`; `electron/preload.cjs` (`pdf.save`) |
| Delivery and Receipt Notes | Both move stock exactly like an invoice (`stock: 'out'` and `'in'`), and **nothing links a note to the invoice that bills it**; only orders are netted (`pendingOrders`) | `types.ts:36-38`; `inventory.ts:305-316` |
| Copy and recall | Ghost recall and Alt+2 copy lines forward. Alt+2 deliberately drops bill and instrument allocations because a copied reference would merge with the source | `core/recall.ts`; `ROADMAP-ADDENDUM.md` (duplicate row); CLAUDE.md Phase 13 |

### 1.2 What is wrong with the board

`parity-data.json` carries `batches` and `expiry-reports` as **covered**, and the first says "Batch tracking, mfg/expiry, batch selection at entry" against code that has no
selection and no mfg entry; the second says "Near-expiry/expired reports; block expired sale" against code with neither. Their notes read "No batch field on item or line" and
"No batch/expiry concept", which are stale in the other direction. CLAUDE.md's own rule is that a status and its note must agree and that `covered` needs the builder and the
reviewer to agree; both rows should be re-stated as **partial** when the work starts. This document does not edit the board.

### 1.3 What is arriving next to this area (so a design does not collide with it) [C, `.wt-cf`]

The custom-fields addendum (decisions CF24 to CF26, section AD.3, slice CF-9) adds a **line-level variant that tracks stock**. I read AD.3 and the CF-9 rows in full and only
skimmed the rest of the addendum; on 2026-10-23 I re-read AD.3.3 after the addendum's **third review** (11 findings, all reproduced, which rewrote the Physical Stock rule). In the worktree the build has
started only at the type level: `countPart?: true` is in `.wt-cf/src/types.ts` and `byVariant` is not yet in `.wt-cf/src/core/inventory.ts` [C, grep on 2026-10-23]. What matters here:

- A choice field marked "Tracks stock" makes (item, variant) a stock position **beside** godown and batch: `byVariant` next to `byGodown` and `byBatch`, a `VariantOf` resolver
  that is `undefined` when no field exists so every output is byte-identical, derived at read time from the line's `udf` and never stored.
- **Costing stays per item** (pooled). A variant's value is the item's value shared by quantity, so no figure outside the new screen can move. The same stance is the safest for batch and
  serial (section 9).
- It edits `stockPositions`, `resolvedMovements`, `stockItemRows`, `pendingOrders`, `stockFlow`, `openingStock.ts`, `splitCompany.ts` and the negative-stock warning in
  `VoucherScreen.tsx`. **Physical Stock** is, after the review, **always a count of the whole item at its godown**: a count made by variant is one **count group**, the lines of one (item, godown)
  saved with `countPart` and merged by `stockMovements` into a single movement of their sum, so no item-level figure depends on whether a caller passed a resolver (a first text that resolved the count against the
  (godown, variant) balance moved qty, value and `byGodown` for some callers and not others, and was withdrawn). Variants held at the godown and not listed count 0; the entry screen pre-fills a row per held variant at its book
  quantity; a pure `countDeltas` spreads the item's delta over the variants [C `.wt-cf/docs/design/custom-fields-spec.md` AD.3.3]. A Stock Journal receipt row starts from its issue row's variant, and the warning becomes
  per (item, variant). Manufacturing is not variant-aware in v1.
- `OpeningStockAlloc` gains `variants?`; `Company` and the server get older-client protection for `udf` (`carryUdf` restores a line's `udf` only where its identity is unambiguous).
- The Back-control spec adds a visible Close to every dialog, and says a dialog that holds **typed work** must not close on a backdrop click and a dialog whose Esc records a
  decision must say so on its button (rule B5; I read B5, item 9 and review rows 12 and 35).

---

## 2. How each product does it

### 2.1 BUSY (the benchmark)

**Method note.** Busy's FAQ pages are numerous but uneven: several answers do not match their question (a "how do I print barcoded bills" answer describes the Outstanding Analysis menu;
an answer about a duplicate batch error recommends a report instead). I used the FAQ as evidence of what Busy says, I weighted it down, and I relied on a statement where the product page agrees or where it
is a limit Busy states against itself. Menu paths differ between pages ("Settings > Features/Options" against "Administration > Configuration > Features and Options").

**Batches [W busy-batch; W busy-faq 36, 44, 49, 50, 53, 57, 67, 72, 73, 74, 76].**

- *Switches.* Batch-wise details are enabled under the inventory features, **then configured**, **then turned on per item** (two switches).
- *Configuration knobs.* Manufacturing and expiry date input can each be **required or not required**; an **expiry warning period in days**; an **action for expired batches** (the product page
  does not say block or warn; an FAQ says the "batch adjust stock" setting reads "Allow with warning", so expired batches **can be sold, with a warning**); a **duplicate batch** switch;
  a **negative stock** warning; **batch-wise MRP** and **batch-wise sale price** options (an FAQ for POS says the price is missing until "Maintain Sales Price" is turned on);
  batch names **manual or auto-generated**; **expiry days set on the item** instead of a date (shelf life); optional **drug licence number** tracking.
- *Capture.* A purchase voucher opens a batch window when the quantity is typed (batch, quantity, manufacturing date, expiry). A sale lets the operator pick the batch and shows its expiry; a
  batch barcode can be scanned where configured. Opening batch stock is entered in the item master. A saved invoice's batch dates are edited by opening the voucher and pressing Enter on the item.
- *Reports (Display or Reports, Advanced Inventory).* **Batch-wise Stock** (quantity, manufacturing, expiry); **Expiring Batches** (on a date or within a range); **Batch On Account** (entries
  whose batch quantity is unallocated or mismatched); **Batch Wise Purchase Price** (an item's purchase price per batch); **batch reference details**; per-batch sales and profit **when the batch
  is on every sale voucher**. Stock ageing is on a FIFO basis [W busy-mobile].
- *Tools.* An **auto-adjustment utility** that allocates batches to past sales **first-in first-out** without changing the sale; "add stock to the batch" to merge two batches that share an id;
  a stock transfer carries a batch between warehouses **if the batch numbers were recorded** on it.
- *Limits (from Busy's own pages).* Not in the **Basic** edition (the "Standard", "Saffron", "Emerald" and "Enterprise" editions carry it), and batch numbers disappear from invoices after an upgrade to an edition without it;
  no quick way to add batch numbers to many old purchases (edit each voucher); **disabling batch tracking removes the batch numbers already on transactions**; a sale with no batch lands "on account" in the
  report; no limit on the number of batches; no scheduled report e-mail; custom reports need a partner or programming.
- *MRP.* The product page lists batch-wise MRP and sale price. An FAQ (seen only in a search summary) says batch-wise pricing is "not available right now" and that MRP and price can be kept with batch
  details by enabling them; two other FAQs route "several MRPs for one barcode" through **MRP-wise details in the parameterised-stock feature**. I read these as **one feature reached by two
  paths, with a stale FAQ** [W busy-batch, busy-faq 5, 53; Ws busy-search], and I flag it as unresolved without a trial.

**Serials and IMEI [W busy-serial, busy-mobile; W busy-faq 14, 21, 23, 36, 53, 67, 74, 76; W busy-serial-tips].**

- *Switches.* Serial No. under the inventory features, then a per-item flag with optional opening stock. **Standard and Enterprise editions only**; the Basic edition lacks it.
- *Numbering.* **Manual** (type or scan) or **automatic** from a configured structure (a renumbering-frequency option exists, for example yearly). In a purchase voucher, **F7 recalls the last serial
  and Alt+F7 generates the next**. One FAQ says serials are 6 to 8 digits, which cannot describe a 15-digit IMEI; I treat it as describing the automatic structure and flag it [unverified].
- *Per-serial data.* Optional **MRP and sale price per serial**; **up to 10 custom descriptions per serial**; **three warranty types: purchase, sale and installation**, set on the item and changeable at voucher entry.
- *Sale.* The serial is chosen from a list or scanned on the item line; the invoice prints it. Starting serial tracking late is allowed (it applies to later transactions); there is no per-item limit on serials.
- *Reports and queries.* **Serial No.-wise Stock** (Reports, Advanced Inventory; filters, search, summary) and Display, Sr. No. Wise Reports; **Item Serial No. query on Ctrl+Q**, which shows a serial's linked transactions
  and status; item ledger on Ctrl+D. A wrong serial on a saved invoice is corrected by editing the invoice.
- *Mobile shop positioning.* Per-IMEI traceability "for returns, warranty and theft claims", serial and IMEI wise stock grouped by brand and category, exchange and buyback with the old phone's value netted on the bill,
  3-inch slip printing, EMI, a UPI QR, a downloadable sample "mobile store" company with IMEI reports [W busy-mobile; Ws busy-search]. The page does not say which edition includes IMEI, warranty or exchange.
- *Integrity.* An FAQ answers "how do I allow duplicate serial numbers" with "not currently available" (send feedback to R&D), which I read as **Busy enforces uniqueness**. A typed duplicate is refused at entry [W busy-faq 67; Ws busy-search].
- *Not documented anywhere I read:* bulk import, a numeric range entry, a returns rule for serials, a Luhn or length check on IMEIs, dual-SIM handling, lookup by a second IMEI [W busy-serial: "not covered"].

**Barcodes and labels [W busy-labels; W busy-faq 5, 23, 50; Ws busy-labels-search].** Printing is an add-on (**Add-on, Item Barcode Printing, Configure Barcode Labels**): label height and width, margins, default printer,
sheet type, skip-lines; fields of type barcode, QR, data field, image or plain text taking values such as item name, alias, price, MRP; labels from the item master, a voucher, an Excel sheet or current stock status; the batch
expiry can be a label field when batches are on. **Designing a format is a chargeable service through a channel partner**; in demo mode and the Basic (B1) edition at most 20 barcodes can be printed, Standard and Enterprise have no limit; a barcode value must be 5 to 20 characters
[W busy-label-faq-5, busy-label-faq-8, read 2026-10-23]. "Print barcode from current stock status" prints stickers for remaining stock (the FAQ's use is a discount sale) [W busy-label-faq-5]. Scanning never creates an item: the barcode is saved
in the item's alias and the scan selects the item; hand-held scanners are supported for serial entry and no models are named [W busy-label-faq-8]. A label can show a batch's expiry when the "Item + batch" barcode type is chosen
and a data field named `EXP_Date` is added [Ws busy-labels-faq]. Barcodes
are generated through the **parameterised-details** feature (prefix, length, numeric or alphanumeric) and a barcode-number (BCN) stock report tracks single units. Scanning is turned on per POS voucher series.

**What Busy's customers say.** Two kinds of evidence, both thin.
- *A page of user reviews I read in full* [W busy-reviews-ss: the SoftwareSuggest listing, rated 4.6 of 5 from 124 reviews, of which the page shows 20, dated September and October 2026]. **None of the 20 mentions batch or expiry.** Nine
  talk about stock and reports in general terms (stock visibility, "stock-linked sales information", item movement and profitability for a distributor, production and BOM links for manufacturers); the recurring complaints are a dated interface, effort
  to set up masters, a learning curve and reports spread across many screens. **One review, from an electronics seller's counter staff, is about serial and IMEI tracking**: it praises warranty checks, tracing a unit to its invoice and finding which godown
  holds it, and it complains that **correcting a wrongly entered serial takes more steps than expected**, and asks for **faster barcode and IMEI scanning on returns** and an **easier screen for one serial's whole history**. One reviewer, one shop; but each complaint
  is a feature request this document already designs for (sections 6.2, 7 and 10).
- *Capterra and GetApp listings through search summaries* [Ws busy-reviews]: small teams and distributors like the batch-wise inventory and multi-location support; complaints about support, refunds, price rises, a dated interface, clumsy spreadsheet import, year-end handling and (in older reviews)
  no remote stock view.
I found **no independent complaint about batch tracking specifically**, no forum thread and no video (a search for Reddit, Quora and YouTube returned only listing sites and one vendor-adjacent blog); the batch and serial weaknesses above come from Busy's own FAQ.
[J] A shop that already uses Busy for batches is mostly praising "it exists and the reports work", which is the bar to clear; the one detailed serial review says the same, with the corrections and history screen as the gaps.

### 2.2 TallyPrime

**What it does [W tally-batches, tally-batch-summary, tally-guide-godown; Ws tally-expired, tally-stockjournal].**

- *Switches.* F11 (company features): enable batches, then maintain expiry dates for batches. Per item: Maintain in batches, Track date of manufacturing, Use expiry dates. A stock item with only a manufacturing date and a shelf-life
  period is supported. Disabling batches **company-wide cannot be undone per item afterwards**; disabling for one item leaves the others [W tally-batches].
- *Capture.* The Stock Item Allocations screen takes a batch name and dates; a new batch can be created by typing a name; the same screen at sale selects an **active** batch, takes mfg and expiry from it (editable), and
  reduces that batch. The allocation is per godown.
- *Expired stock.* A sales-voucher switch (F12, Advanced; older pages call it "Honor Expiry Dates usage for Batches") **defaults to allowing expired batches**, so that expired stock can be returned to the supplier (a debit note)
  or sold. Turned off, expired batches disappear from the active list and choosing one is an error [Ws tally-expired].
- *Reports.* **Batch Summary** per item and period (quantity, rate, value per batch; Ctrl+B for expired-only and for manufacturing or expiry date ranges; F12 shows mfg, expiry and days to expiry and includes zero rows; Enter
  to the monthly summary and **Batch Vouchers**; the page does not say whether the report can be filtered by godown [W tally-batch-summary, re-read 2026-10-23]); **Stock Ageing Analysis**, which the help page describes as grouping stock by days left to expiry [W tally-batches, tally-batch-summary].
- *Alerts.* **None.** Tally's own guide says it shows no automatic expiry pop-up; the user runs the reports and turns on the expiry column [W tally-guide-godown].
- *Godown transfer.* The Stock Journal **class** for inter-godown transfers mirrors items and batches to the destination with batch number, rate and value; the plain Stock Journal's documented steps mention godown,
  quantity and rate and not the batch [Ws tally-stockjournal]. The class article, read on 2026-10-23, needs Multi-Location Inventory, at least two godowns and the setting "Use Class for Inter-Godown Transfer", and **does not say whether a plain
  Stock Journal carries the batch** [W tally-stockjournal-class]. [J] That gap is a classic source of "the batch disappeared after a transfer" complaints, but I found no thread confirming it.
- *Serials.* **Not native.** The Tally.ERP 9 help describes using the IMEI as the batch number of a single unit, optionally prefixing a consignment or lot (its example reads like "Feb-654321543214321"), and reading unsold units from the Batch Summary, where a sold unit's closing figures are blank and an unsold unit shows a quantity of 1; the settings are "Maintain batch-wise details" under F11 Inventory Features and "Maintain in batches"
  on the item [W tally-erp9-unsold, read in full 2026-10-23; the article body follows about 200,000 characters of navigation]. "Tracking numbers" in Tally link a Delivery or Receipt Note to the invoice that bills it: the invoice pulls in the note's quantity, rate and amount, a new number can be typed to invoice first and deliver later, "Not applicable" skips the link and lets the accounting voucher move stock, and the option is "Use tracking numbers"
  under F11 Inventory Features; they are a document link, not a unit identity [W tally-tracking, read in full 2026-10-23]. The article does not say outright that stock moves once when a note and its invoice are both used.
  I found **no IMEI add-on** on the pages reachable to me; TallyShop add-ons for barcode print and scan exist [Ws tally-addons]. A search on 2026-10-23 aimed at a serial or IMEI TDL found only vendor blogs that describe what such an add-on would have to do, one
  voucher-numbering add-on, and mobile-shop packages that are not Tally; Tally's own guide says TallyPrime has no standalone mobile billing app [Ws tally-tdl-search].
- *MRP.* A stock-item setting (F12 "Provide MRP details", then an MRP with a date of applicability, optionally by state, optionally editable in the voucher) [W tally-mrp-page for the item setting and the "modify MRP in vouchers" option; Ws tally-mrp-state for the applicable-from date and the state-wise list, from the Tally help pages of the VAT era]. The page I read
  **never mentions batches**, so MRP and batch are unconnected there, and I found **no per-batch MRP**. The nearest thing is a per-batch selling price, mentioned once [Ws tally-mrp]. A batch UDF holds a **string only**; a number there needs a hand-written method before a report can read it [W tally-udf-batch via `docs/research/tally-udf-and-competitors.md:110`].
- *Barcodes and labels.* The item's **Alias** holds the code (up to 32 characters, unique); scanning into the item field fills the item; a generator and the scanner's software are external. TallyPrime releases 5 and 6 list nothing on barcodes [W tally-barcodes; Ws tally-release].
  A third-party article that claims native label printing contradicts itself and says batch barcodes need an add-on [W tallyatcloud-barcodes].

**Real gaps, in the order a trader feels them [J on the evidence above].**

| # | Gap | Who feels it |
|---|---|---|
| 1 | No serial or IMEI engine; one batch per unit is the workaround (slow typing, no warranty, no identity checks, no scan, no uniqueness) | mobile, electronics, appliances, jewellery |
| 2 | No per-batch MRP or price | pharma, FMCG, any trader whose MRP changes |
| 3 | Batch chosen by hand; no proposed first-expiry-first-out | pharma, food |
| 4 | Expired stock saleable by default; no expiry alert; no "short-expiry" rule | pharma, food |
| 5 | No batch trace report ("who got batch X"); Batch Vouchers is one batch of one item | recall, quality |
| 6 | Batch carried across a godown transfer only through the Stock Journal class | multi-godown traders |
| 7 | Barcode only as an alias; no label printing, no batch or serial label, no scan into a serial list | everyone with a scanner |
| 8 | No count check or stock-take by scanning | everyone |
| 9 | No unit warranty, no warranty-due report | electronics |

### 2.3 Marg ERP and the other Indian products

- **Marg.** Marg's own support site (`care.margcompusoft.com`) has how-to pages, and on 2026-10-23 I read what the fetch tool could extract of four (each page is mostly navigation, the article body was often cut off):
  an **Index Batch** setting (Master, Marg Setup, Control Room) that makes the billing batch list follow expiry [W marg-index-batch for the path only; the option name "Y-Expiry" and the nearest-expiry-first order are from a search summary, Ws];
  a **Near Expiry / Expired Report** (Daily Reports, Inventory Reports, then a store, a Simple or Columnar layout and a report window) with an expiry date range (From and Upto), code or name order, a **single supplier or supplier and company grouping**, and a company
  picker [W marg-near-expiry; its columns, and any return-to-supplier action, are not on the page]; a **Batch Stock** report (Stock, Batch Stock) whose page names product identification, expiry monitoring and recall management as its uses and whose filters I could not read [W marg-batch-stock];
  an expiry edit at billing (F4 on the item, Batch Detail, F3 on the batch, edit Exp, Ctrl+W) [Ws]; and a **Breakage / Expiry Receive** entry whose help index lists "create item batch" inside it (titles only; the return workflow is not described) [W marg-brk-exp].
  Marg's pharmacy marketing page lists near-expiry alerts and an expired and near-expiry indication at billing [Ws]; its blog calls the batch handling "First-Expiry First-Out" but names no setting by that name [Ws].
  Everything else is from third-party listings [Ws marg-listings]: pharma editions list batch and expiry management and scheme management, a "push sale and expiry
  management" tool for moving near-expiry stock to suitable customers, salesman, area and route reports in higher editions; the appliance edition lists warranty management and barcode integration; **no listing mentions IMEI or serial
  numbers**. The "Marg is strongest on pharma batches" premise matches Marg's own pages (an expiry-ordered batch list, a dedicated Near Expiry / Expired report with supplier grouping) and the listings; I cannot go further.
  [unverified] Whether Marg **proposes** the first batch or only lists it first (the Index Batch page body was not readable), scheme-on-batch behaviour, registers, any per-batch MRP rule, and anything about IMEI or serials.
- **Logic ERP** (a pharma distribution and electronics product with public KB pages) [W logic-imei, logic-pharma]. IMEI: a serial window at purchase; **F2 bulk entry from a start IMEI, a start row and a quantity**; at sale the item code is set to
  "serial mobile number" so the voucher prompts for an IMEI that is scanned or typed and fills item, model, price and warranty. Not covered on its page: validation, duplicates, returns, reports, dual-SIM. Pharma (vendor page): stock tracked
  by batch and MRP, a **prompt at billing for expired or near-expiry stock**, **batch-wise free schemes**, purchase returns for expired, damaged or normal stock with pending amounts per supplier, a block on Schedule H and narcotic sales
  until patient and doctor details are entered, a "last rate" lookup. A "price drop management" KB page exists but its body would not load [Ws logic-pricedrop].
- **Busy's medical-store page** [W busy-medical]: batch, expiry and MRP at purchase carried to sale; near-expiry flags (the FAQ is softer than the copy: reports "can highlight"); expiry returns against the **original supplier bill**; batch chosen at billing by search or scan. It does not mention a Schedule H register, drug licence tracking, or FEFO by name.
- **GoFrugal** (pharmacy POS) [W gofrugal-expiry, partial]: an Expiry Returns screen under Purchase that flags products near expiry for return to a distributor; the rest of the page was cut.
- **Vyapar** [W vyapar-inventory]: batch numbers with movement from purchase to sale and expiry alerts; IMEI mentioned only for mobile accessories, with no detail; barcode generation and import are **Gold-plan** features; no MRP mention.
- **myBillBook, HisabPlus, Accountune** (vendor blogs) [W mybillbook-imei, hisabplus-imei, hisabplus-importance, accountune-electronics]: the workflows they describe are the ones in section 3.1; their strongest verifiable statements are that a return should match the IMEI and not only the model,
  that a duplicate-IMEI warning is something "a well-configured system can" give, and that unsold IMEIs should be compared to the shelf. All are marketing claims.

### 2.4 Zoho Inventory [W zoho-*]

- **Plan-gated.** Enabled under Settings, Preferences, Items by choosing the stock-tracking method per item: **serial number or batch, not both**; an organisation can enable both methods [W zoho-index, zoho-serial].
- **Serial.** Serials are captured on the purchase bill; invoices choose the units; adjustments add or select serials. **Scanning** a serial into the field is supported (the cursor goes in the serial field and each scan is added to the transaction; the page does not say whether a scan can create a serial, what happens to one that is not available, or which scanner is needed
  [W zoho-barcode-serial, read 2026-10-23]; a search summary says an invoice can only pick serials already recorded and that new ones are entered on a bill, Ws); a transaction takes **up to 1,000 serials** [W zoho-serial-limit, re-read: that page states only this limit and
  says nothing of opening stock, scanning or paste], opening stock **up to 200 at a time** [Ws zoho-adv, carried from the first draft and not re-confirmed];
  a serial can be searched and shown on the PDF; clicking a serial shows its transaction history [W zoho-serial-limit, zoho-barcode-serial, zoho-returns, zoho-index]. The **Serial Number Details report** has item, serial, status (available or sold), inward and outward document and date, warehouse, batch columns [W zoho-reports]
  (the batch columns sit oddly with "one method per item"; I could not reconcile them).
- **Batch.** Batches carry manufactured and expiry dates; the report distinguishes an **internal batch number** from the **manufacturer's batch number**. At invoice, batches are **listed soonest-expiry first**, and the operator chooses [W zoho-first-expiring, re-read 2026-10-23: the page describes a sort and not an automatic pick, says an expiry date must be entered for every batch,
  and says nothing of overriding the order or of expired batches]. A batch's page shows its creation, initial and current quantity and all linked transactions [W zoho-batch].
- **Batch price management** (optional): a different MRP and selling price per batch; a default sales margin on cost for new batches; four methods decide what pre-fills: the item default, the **latest** batch's, the **highest**, or the price recorded **for the selected batch** (which needs a package or pick list first); existing batches' prices cannot be edited in a purchase receive; converting a sales order to an invoice lets the person keep the order's price or apply the batch's, and with the "price from inward transaction" method an order filled from several batches
  through packages or pick lists **splits the invoice by batch** [W zoho-batch-price, re-read 2026-10-23; the page states no plan requirement].
- **Returns.** A sales return needs the original sale to have gone through the whole cycle (serials or batches selected, paid, shipped); damaged or expired units go **credit-only** (not back to stock); a returned lot **cannot return to the old batch**: it gets a new reference while the manufacturer's number may stay [W zoho-returns].
- **Labels.** Barcode and QR label templates (Code 128 and others), paper from label sheets to custom sizes, **one label per selected serial** (a unique barcode each) and, for a batch-tracked item, the **batch number as the encoded value**; the generation field can also be item name, SKU, EAN, UPC or ISBN; bulk printing from the Items list takes a number of copies per item and has a selection cap whose
  figure the page does not give; a dedicated printer's label size must match the template's [W zoho-labels, re-read 2026-10-23]. A community thread calls the generator limited [Ws zoho-community-labels].
- **Composites, item groups, plan and "mandatory serials"** (read 2026-10-23).
  *Composite items* can carry serials or batches **on the composite item itself**: serial tracking is switched on for the composite, and serials are recorded on a purchase bill or, with opening stock, entered comma-separated on the same page; the page says components can include non-inventory items and services (labour or service time) and **does not say** whether components can be serial-tracked, how
  serials are assigned when a composite is assembled or sold, or what component stock is needed [W zoho-composite-serial]. The earlier summary that an assembly "needs full component stock" [Ws zoho-composite] is unconfirmed. A search summary of the Create Assemblies API says it accepts component serials and batches (a batch by ID with the quantity consumed) and a separate list of serials, or new batches, for the finished product [Ws zoho-assembly-api].
  *Item groups (variants):* batch tracking **cannot be set on the group**; it is enabled variant by variant [W zoho-batch-variants-composites]. The same page does not say whether a composite's batch belongs to the composite or to its components.
  *Plan:* serial tracking is available in the free trial and on the Professional plan or higher, switched on under Settings, Preferences, Items [W zoho-enable-serial]; the page does not say whether an item can be switched to serial tracking once it has transactions; a search summary says turning tracking **off** generally means deleting the item's transactions and clearing its opening stock, or inactivating the item and creating a new one [Ws zoho-disable],
  which is Busy's behaviour too (section 2.1). A second search summary repeats the opening-stock caps (200 serials, 100 batches) [Ws zoho-adv].
  *Mandatory serials:* **none of the three Zoho pages I read states whether serial selection is compulsory on an invoice** (whether an invoice can be saved without it); the returns page implies the serials or batches are selected when the sale is processed [W zoho-returns].
  *Zoho Books:* its inventory features depend on a connected Zoho Inventory account (a search summary: a Premium Books plan or higher plus a paid Inventory plan of Professional or higher) [Ws zoho-books]; I found no separate Books or Books India page on serials or batches, so everything above is Zoho Inventory.

### 2.5 Odoo [W odoo-lots, odoo-expiry, odoo-serial, odoo-removal, odoo-fefo; Ws odoo-trace]

- **Tracking** is per product (by lot or by serial), after enabling Lots and Serial Numbers. A receipt **cannot be validated** until its tracked lines have lot or serial numbers; each operation type decides whether **new** numbers may be created (receipts yes, deliveries normally no, so a delivery cannot invent a serial).
- **Generating and importing.** A "Generate" step takes the **first serial and a count** and infers the pattern; an import takes **one serial per line** (paste from a spreadsheet), with "keep current lines" to add or replace. **The number of serials drives the quantity** field.
- **Expiry.** Per product, four day-counts from receipt: **expiration**, **best before**, **removal**, **alert**; they compute a lot's dates, which can be overwritten. Perishables are meant to use the **FEFO** removal strategy; a red alert banner shows once the date passes; lots can be filtered by expiry alert. The page does **not** describe blocking an expired lot from sale.
- **Removal strategies** (re-read 2026-10-23): **FIFO** (earliest incoming date), **LIFO**, **FEFO** (earliest **removal date**, which is the expiration date less the product's removal day-count, not the expiration itself), closest location, and least packages; set on a product category (a "Force Removal Strategy", which wins) or on a storage location; FIFO and LIFO need Lots and Serial Numbers,
  FEFO also needs Expiration Dates [W odoo-removal]. The FEFO page's worked example has three lots and an order of six cartons: five come from the lot with the earliest removal date and one from the next [W odoo-fefo]. On a delivery the lot is chosen by the strategy, and the quantity taken from each lot can be changed to split a demand across lots [W odoo-lots]. **Neither strategy page says how the proposal is overridden or what happens to a lot already past its dates.**
- **Traceability.** From Inventory, Products, Lots/Serial Numbers, selecting a lot and pressing its Traceability button opens that lot's **stock moves**; the section describes the full history as origin, arrival, storage and who received it and when, and **does not describe an upstream and downstream view, a start from a receipt, a customer list or a recall procedure** [W odoo-lots, re-read 2026-10-23]. Third-party modules exist **specifically** to answer "who received this lot" and to print recall audit trails [Ws odoo-trace],
  which suggests the core report is a lot's stock moves and not a customer recall list [J].

### 2.6 NetSuite and SAP Business One (reference only) [Ws sap-b1, netsuite]

- **SAP B1.** "Manage item by" is serial **or** batch (exclusive) and changes only with no stock. The **management method** is either "on every transaction" (identity captured the first time it enters) or "on release only" (captured when it leaves, including transfers and credits);
  "automatic serial creation on receipt" makes successive numbers for a purchased quantity; **tracked items cannot go negative**; an "automatic allocation" button on the batch selection window chooses bins; Excel entry of serials and batches arrived in 9.3.
  Whether B1 itself offers FIFO or expiry-first auto-selection, I could not confirm: the sources that say so are add-ons.
- **NetSuite.** Lot-numbered and serialized item types; expiration belongs to the **lot**; lots and serials can be auto-generated; a Lot Traceability SuiteSolution traces forward to customers and back to suppliers. A forum thread says serialized items take no expiry date and the usual workaround is a lot-controlled item with a custom record per serial; I did not verify that against current NetSuite.

---

## 3. The domain: what each kind of business actually tracks

### 3.1 Mobile phones and consumer electronics

#### 3.1.1 The IMEI, exactly [W wiki-imei, wiki-luhn]

| Part | Digits | Meaning |
|---|---|---|
| Type Allocation Code (TAC) | 8 | the model and its origin; its first two digits name the body that allocated it |
| Serial | 6 | set by the maker |
| Check digit | 1 | Luhn (mod 10) over the first 14 digits |
| IMEISV (a variant) | 16 | the check digit replaced by a 2-digit software version; never printed on a box |

- A **dual-SIM phone normally has two IMEIs**; a device with neither a SIM slot nor an eSIM usually has none (rare exceptions). Dialling `*#06#` shows them; they are also printed on the box and the phone.
- **Luhn.** From the right of the 14-digit body, double every second digit starting with the rightmost, subtract 9 from any result above 9, add everything, and the check digit is what lifts the total to a multiple of 10. It catches every
  single-digit typo and almost every swap of adjacent digits; it **misses a swap of 09 with 90** and a few twin-digit errors (22 and 55, 33 and 66, 44 and 77) [W wiki-luhn]. So a Luhn-valid IMEI can still be mistyped; the real defence is scanning.
- **Worked example** (the body is the one used on the IMEI page): 49015420323751. Reversed: 1 5 7 3 2 3 0 2 4 5 1 0 9 4. Doubled positions give 2, 5 (from 14), 4, 0, 8, 2, 9 (from 18), sum 30; the others sum 22; the total 52 needs 8 more to reach 60, so the full IMEI is **490154203237518** [W wiki-imei agrees; I recomputed it].
- **Consecutive units are not consecutive numbers.** Adding 1 to the 14-digit body and recomputing the check digit gives 490154203237518, **490154203237526**, **490154203237534**, **490154203237542** (last digits 8, 6, 4, 2). Adding 1 to the **whole** 15-digit number gives 490154203237519, ...520, ...521, and **all three fail the check** (computed). A "from this IMEI, N units" generator must step the body and recompute the check digit; Logic ERP's F2 start-plus-quantity feature is the only range tool I found and its page says nothing about this [W logic-imei]. Whether a carton's phones really are sequential differs by maker [unverified], so a range tool is a convenience and never an assumption.
- **A non-IMEI serial exists on the same shelf.** Laptops, TVs, ACs, watches and some phone brands carry an alphanumeric serial with **no standard check digit**. The unit identity field must take both: Luhn-check a value only when the item is flagged "IMEI" (or the value is 15 digits and the person asks).
- **The TAC is a free consistency check.** Units of one item usually share a TAC (a colour or storage variant may not). A scan whose TAC differs from every other unit of that item deserves a "different model?" prompt; this needs no external database [J].

#### 3.1.2 What Indian rules say about IMEIs (read from the DoT release of 17 November 2025; the Act and the Rules themselves were not read)

The source is one government press release: **Press Information Bureau, Ministry of Communications, Release ID 2190763, posted 17 November 2025**, "Department of Telecommunications cautions manufacturers, importers and resellers about mandatory IMEI registration and consequences of IMEI tampering".
The live page answered 403 to the fetch tool, so I read a PDF print of the same page (header: Press Information Bureau, printed 17 November 2025), 4 pages, in full [W pib-dot]. It is an advisory summarising the law, **not the law**.

- **Tampering and possession.** Telecommunications Act, 2023: section 42(3)(c) prohibits tampering with telecommunication identifiers, IMEI included; section 42(3)(f) makes **wilful possession of radio equipment such as a mobile handset, modem, module or SIM box, knowing that it uses unauthorised or tampered identifiers**, an offence. Penalty: imprisonment up to three years, a fine up to Rs 50 lakh, or both; the offences are cognizable and non-bailable (section 42(7)); abetting or promoting them carries the same punishment (section 42(6)).
  Telecom Cyber Security Rules, 2024: Rule 8(3) bars anyone from intentionally removing, obliterating, changing or altering the unique equipment identification number, and from using, producing, trafficking in, having control or custody of, or possessing hardware or software related to an identifier **knowing it has been so configured**; the release says
  using devices **with programmable IMEIs** amounts to tampering; Rule 5 lets the Central Government direct telecom entities to block equipment with a tampered IMEI [W pib-dot].
- **Who must do what.** Manufacturers register the IMEI of every device made in India **before first sale, testing, R&D or any other purpose**, and importers register before importing, both on the Device Setu (ICDR) portal, where a brand is linked to its GSMA TAC; the registration scope includes smartphones, cellular smartwatches, mobile hotspots, tablets,
  USB modems, modules, dongles, laptops and SIM boxes. Under the **2025 amendment** the Central Government can direct manufacturers not to assign an IMEI already in use in Indian networks to a new device. **Entities buying or selling used mobile devices must check the national database of tampered or blacklisted IMEIs before completing a transaction, paying a fee per verification** [W pib-dot].
  For sellers, resellers and retailers the release says only that they "should be aware" that procuring, assembling or using devices with tampered or configurable IMEIs can have serious legal consequences. **It does not state a duty on a retailer to validate the IMEIs of new stock**; the first draft's "must ensure" came from a secondary report and is withdrawn.
- **Checking.** `*#06#` on the handset; the Sanchar Saathi portal or app; an SMS "KYM" plus the 15-digit IMEI to 14422; stolen or lost phones are blocked through the **CEIR** by the owner with a police report [Ws sanchar-saathi, from the first pass's search summaries, not re-read]. The release itself names only the ICDR portal for registration and **does not say how a used-phone dealer performs the database check**, what the fee is, or whether any API exists.
- **Caveats.** I read the release, not the Act, the 2024 Rules or the 2025 amendment (the release gives their address, `dot.gov.in/act-rules-content/3296`, which I did not fetch). Section 42(3)(f) turns on **knowledge**, and nothing I read names a retailer offence for selling a duplicate IMEI as such; so how it bears on a shop that buys, scans and records its stock is my reading and not a stated fact [J, unverified].
  I found no public API for the check, so building it into an accounting product is an open question, not a feature [unverified]. The release's own reasons for the rules include law enforcement and tax collection, which is an argument for keeping a unit's IMEI on every invoice.
- **What a product should do about it [J]:** keep the IMEI on the sale invoice (a buyer needs it for a CEIR block or a police report); for a **used phone received** (a buyback, an exchange), store "IMEI checked on date X, result Y, by whom" and the seller's identity as part of the receipt; keep every IMEI ever seen, so that a unit that reappears can be flagged; never delete a sold unit's history.

#### 3.1.3 The workflows, step by step

| Step | What happens in the shop | What the software must do |
|---|---|---|
| Buy | Cartons arrive with a supplier invoice; the IMEI is on each box (a barcode) and inside the phone | scan each box into the purchase against the invoice; count equals invoice quantity; refuse a unit already in stock; accept a unit previously sold (a trade-in coming back) as a new inward; flag a box whose IMEI differs from the phone's `*#06#` as a manual check, outside software |
| Stock | Boxes shelved by model, godown or counter | stock by IMEI; moves between counters or godowns change the unit's location, never its identity |
| Sell | The cashier scans the box being handed over; accessories by their own barcodes | the unit must be in stock at this location; the invoice prints the IMEI (both, for a dual-SIM) and the warranty end; one unit leaves, qty 1 per unit |
| Exchange and buyback | The old phone is valued and netted on the bill | a new inward of a **used** unit with its IMEI check, a cost equal to the credit given, its own item or condition grade; see 3.1.4 |
| Warranty | Brand warranty runs from the sale (the rule differs by brand [unverified]); the shop may add its own | warranty start defaults to the invoice date; types kept separately (Busy keeps purchase, sale, installation [W busy-serial]); expiry stored on the unit |
| Return | The customer brings it back within a window | find the sale **by IMEI** and compare the unit in hand; model alone is not enough, a different handset of the same model could come back [W mybillbook-imei]; the unit goes back to stock only if its status is "sold" |
| Repair and replacement | The unit goes to service; a replacement may be issued | a "with service centre" state; a replacement is a **new** unit linked to the old one so the warranty history survives |
| Supplier return (DOA) | A dead-on-arrival phone goes back | the unit must be in stock; it leaves with the supplier-return voucher; its status is "returned to supplier" |
| Brand scheme claims | Brands pay price-protection or sell-out schemes against IMEIs | a report of IMEIs sold in a period by model, with purchase and sale dates; whether brands require IMEI-level claims is [unverified] |
| Theft or loss claim | A customer or the police asks "who bought IMEI X?" | one lookup by either IMEI of a dual-SIM unit |

#### 3.1.4 Used phones and the margin question

A buyback makes the shop a seller of second-hand goods. GST has a **margin scheme**. The CBIC leaflet "Margin Scheme in GST" (Directorate General of Taxpayer Services, updated November 2022; a plain-language leaflet, **not the rule text**) says [W cbic-margin]:
under **Rule 32(5) of the CGST Rules, 2017**, where a taxable supply is made by a person dealing in buying and selling second-hand goods (used goods as such, or after minor processing that does not change their nature) **and no input tax credit was availed on the purchase**,
the value of supply is the **selling price less the purchase price**, and a **negative result is ignored**; value added by repair, refurbishing or reconditioning is added to the margin; a proviso covers goods repossessed from an unregistered defaulting borrower (the purchase price is reduced by 5 percentage points per quarter or part of it); Notification 10/2017-Central Tax (Rate)
exempts intra-state purchases of second-hand goods from an **unregistered** supplier by a dealer who pays tax on the margin; and a person who sells to such a dealer under the scheme issues no taxable invoice while the dealer claims no ITC. The leaflet's example is one car bought and sold.
It **does not say** whether the margin is worked out per item or across a period, what wording the margin-scheme invoice must carry, or what records must be kept, and the secondary articles I saw differ on whether the scheme is optional. Those are for a chartered accountant, not for this document.
What follows for the design [J]: the margin is defined by *this* sale against *this* purchase, so **a per-unit purchase price (and, for a refurbished phone, the repair cost added to it) is the natural input**; the unit engine's specific-identification cost (R19) is what a margin invoice would read from. Do not build the invoice before the treatment is confirmed.

#### 3.1.5 Fraud and error patterns and the control each needs

| Pattern | Control |
|---|---|
| The same IMEI on two invoices | refuse the second sale at entry; flag two devices doing it offline at sync (R14) |
| A return of a different unit of the same model | the return screen asks for the IMEI and compares it with the sale |
| A mistyped IMEI | scan first; Luhn check; show the unit found, not only "OK" |
| A unit in the shop that is not in the books | stock-take by scanning shows the extra |
| A unit missing from the shelf that was never invoiced | the same count shows the missing; each one needs a reason |
| A blacklisted or tampered used phone | an IMEI-check stamp on every used receipt (3.1.2) |
| A box IMEI that differs from the phone | a human check at receipt, because no software sees the phone |
| A unit sold from the wrong godown | location is part of the unit's state; a sale from a godown it is not in is refused |

#### 3.1.6 Appliances, computers and other durable goods

One serial per unit, alphanumeric, often several (a unit and a component); **three kinds of warranty** (from the supplier, to the customer, from installation: Busy models exactly these [W busy-serial]); installation dates; annual maintenance contracts are a separate record (a partner's Tally add-on exists for AMC dates [Ws tally-addons]). Consumables and cables in the same shop are
quantities, not units. A product may need **both** a lot (the production batch, for a recall) and a serial (the unit): Zoho and SAP B1 make you choose one [W zoho-index; Ws sap-b1], and Zoho's own serial report carries batch columns anyway, so the need is real.

### 3.2 Pharma, FMCG and food

- **The label facts a batch must hold.** Packaged goods must show the **month and year of manufacture, packing or import**, a **best-before or use-by** date where the goods spoil, and the **maximum retail price, stated as inclusive of all taxes**; a retailer may not sell above the printed price and may not alter it
  [W legal-metrology: Packaged Commodities Rules 2011, rule 6(1)(d), (da), (e) and 18(2), 18(5)]. For **food**, a **batch, code or lot number** must be declared (reg 5(9)); the **date of manufacture or packing and the use-by or expiry date** must be declared with best-before optional (reg 5(10)(a)); shelf life **up to three months** shows day, month and year, **longer** shows **month and year**
  (reg 5(10)(b)(i)); shelf life of seven days or less needs the use-by only (reg 8(3)); and a food past its use-by or expiry **shall not be sold** (reg 2(1)(u)) [W fssai-compendium: FSSAI's own compendium PDF of the 2020 regulations, Version-II of 29 December 2021, not the gazette]. **Design consequence:** an expiry is often a **month**, not a day. Store it as the **last day of that month**, show it as MM/YYYY, and treat the batch as expired on the first day of the next month. [J; the month-end reading of a printed drug expiry is the universal practice I know of, but I could not cite an Indian rule for it]
- **Drugs.** The Schedule H1 register records each supply at the time of sale: **prescriber's name and address, patient's name, drug, quantity**, kept for **three years** and open to inspection. The register format published by a government health body (NHSRC) under G.S.R. 588(E) of 30 August 2013 has these columns: serial number, prescriber's name and address, patient's name, medicine dispensed, quantity dispensed, and the patient's registration number and date;
  it has **no batch, expiry or bill-number column** [W nhsrc-h1, read 2026-10-23; the first draft said the model format added them, which came from a search summary and is withdrawn]. The same document reproduces the NDPS Forms 3D, 3E and 3H for essential narcotic drugs in healthcare facilities (kept two years); they are for facilities and were not studied further.
  **That does not mean the batch is optional on the sale record.** `tracking-verticals-india.md`, which read the Drugs Rules PDF, reports that Rule 65(3) has a retail licensee record, for Schedule C, H and H1 drugs, the **manufacturer, batch number and expiry date**, in the prescription register or in serially numbered credit memos, beside the separate H1 register [R, its section 3]; search summaries of secondary sources
  repeat it [Ws drug-rules-secondary]. I could not open the Rules PDF (the host answered 403), so the clause itself is unread by me. **Design consequence [J]:** the H1 register is a sale-line report that needs the prescriber and patient captured at the counter (Logic ERP blocks the sale until they are entered [W logic-pharma]); the batch and expiry belong on the H and H1 sale record,
  and the sale line already carries them, so they cost nothing to print. A 2026 High Court ruling cancelled a chemist's licences for a missing
  H1 register and out-of-order purchase records [Ws drug-records]. The retention of general sale and purchase records and the exact clause for batch numbers I could not confirm [unverified]; sale of an expired drug is an offence under the Act [unverified, not fetched].
- **QR on medicine packs.** For batches manufactured **on or after 1 August 2023**, manufacturers of the Schedule H2 formulations (the "top 300 brands"; notified 17 November 2022 as G.S.R. 823(E)) must print or affix a **barcode or QR code** carrying: a **unique product identification code**, the proper and generic name, the brand name, the manufacturer's name and address,
  the **batch number**, the **date of manufacturing**, the **date of expiry** and the **manufacturing licence number**; on the primary label, or on the secondary label where the primary has no room; Indian and foreign makers are both covered; an imported product is preferably labelled before import and may be labelled in India with the licensing authority's permission
  [W cdsco-faq: CDSCO's "FAQs on implementation of GSR 823(E)", eight questions, a PDF dated 25 July 2023, read in full 2026-10-23]. **The FAQ says the unique product identification code is decided by the manufacturer under its own procedure, and it names no encoding standard, no GS1 identifier, and no requirement that a retailer's software can read the code**
  (the first draft's "readable by an application" is not in the FAQ and is withdrawn). GS1's October 2023 healthcare position paper, **as a search summary only** (the page answered 403) [Ws gs1-india-position], says the requirement gives no clear guidance on the code type or on how the data is reached, that proprietary vendors are meeting it with a **QR code that encodes a web address**,
  which opens a page confirming or denying authenticity, and that GS1 recommends a **GS1 DataMatrix** carrying the GTIN, expiry date, batch and serial. A June 2026 report of an amendment extending the framework to more drug classes (vaccines, antimicrobials, narcotic and psychotropic drugs, anticancer drugs; nine data elements) is a single secondary source, headline and summary only [Ws onmanorama-2026].
  **Design consequence [J]:** a pharmacy **cannot assume** that a pack's code holds the batch and expiry as text; many will hold only a link. Scanning can fill the batch row **only when the payload parses** (a GS1 element string, or a text layout the shop has been taught); otherwise the batch and expiry come from the supplier's invoice or the pack's printed text, and the scan is at most an extra reference.
- **MRP changes by batch.** Two batches of one item carry different MRPs after a price revision, and the invoice shows the MRP of the batch sold. Busy, Zoho and Logic ERP model this; Tally keeps one dated MRP on the item [W tally-mrp-page; Ws tally-mrp-state]; Comsky has none.
- **First-expiry-first-out.** Pharma and food sell the soonest-expiring batch that still meets the customer's minimum remaining shelf life; **a retailer will refuse a pack with less than some months left**, so a distributor needs a "do not issue within N days of expiry" rule (a Busy case study describes exactly that rule [Ws busy-search]).
- **Returns to the supplier.** Near-expiry and expired stock goes back against the **original supplier bill** with a credit note, tracked per supplier [W busy-medical, logic-pharma; W gofrugal-expiry]. One stockist's notes put expiry returns near 5 percent of stock [Ws, via `docs/research/mobile-personas-and-jobs.md:432`].
- **Free schemes.** "10 plus 1" moves eleven and bills ten; the batch given free is the batch drawn. Comsky has a single quantity per line (parity row `actual-billed-qty`, missing) [C]; Logic ERP attaches schemes to batches [W logic-pharma].
- **Recall.** For a recalled batch the shop needs, in one screen: stock still held (by godown), quantities sold and to whom, the supplier bill it came in on, and a **block** so the batch cannot be sold while it is investigated. Tally's own pages say a returned item's batch lets you check whether others in the batch may be affected [W tally-batches]; Zoho says the point of batches is tracing a defective item back [W zoho-batch]; neither names a customer recall list.
- **Price drop.** When a maker lowers the price, stockists claim the difference on stock held that day, **by batch**; Logic ERP has a KB page by this name [Ws logic-pricedrop; body unreadable]. The statutory basis (price control orders) is [unverified].
- **Food retail.** Weighed goods carry a price-embedded barcode printed by a scale (3.4 and 8.2); Indian grocers also print their own MRP stickers.

### 3.3 Garments and footwear

Barcode **per size and colour** of a style, no serials and no batches. This is the custom-fields track's variant (CF-9) plus a barcode map from code to (item, variant). Busy's parameterised stock (up to five parameters, MRP and barcode per variant) is the benchmark and is where Busy's label feature starts [W busy-faq, `tally-udf-and-competitors.md`]. Batch and serial must **not** be used to fake variants, and variants must not be used to fake batches.

### 3.4 Grocery and weighed goods

A scale prints a label whose barcode embeds the item and a weight or a price, with a check digit. EAN-13 numbers beginning 020 to 029 are reserved by GS1 for a retailer's internal use, such as own-brand or weighed items [W wiki-ean13]; I believe the restricted block also includes 200 to 299 [unverified: the GS1 specification would not load]. Scale makers use different layouts, so the parser must be configured per scale and tested with the owner's scale [unverified for any named Indian maker].

### 3.5 Jewellery

A **tag is a unit** with attributes, not just a number: gross, stone and net weight, purity, a tag number, and, for gold, a **HUID**. Gold jewellery must carry the BIS hallmark of **three marks: the BIS logo, the purity (carat and fineness), and a six-character alphanumeric HUID that is unique to the article** (paired items and detachable parts each get their own [first pass, BIS blog read by `tracking-verticals-india.md`; not re-read]); the jeweller's own mark is optional; jewellers with annual turnover up to Rs 40 lakh are outside mandatory hallmarking.
**What the BIS jeweller page says is and is not mandatory** [W bis-jewellers: `bis.gov.in/hallmarking-jewellers`, read 2026-10-23, first 100,000 of 108,850 characters]: the bill or invoice for a hallmarked article must separately show **each article's description, net weight of precious metal, purity in carat and fineness, and hallmarking charges**; **showing the HUID number on the invoice is "voluntary at present"**;
and **maintaining a record of HUID numbers is not mandatory**; a customer can check a HUID in the BIS Care app ("Verify HUID"), which shows the jeweller's registration number (the consumer FAQ page confirms the app and the registration number and says nothing else about invoices [W bis-consumer-faq]). The first draft said the invoice should list the HUID; that is withdrawn. The HUID requirement was notified from 1 April 2023
and enforced in phases [Ws bis-huid-news]. One August 2026 report mentions a **proposal** to mark HUIDs "sold" and to link HUIDs to retailers' stock systems; it is single-source and not in force [Ws bis-huid-news]. **Design consequence [J]:** because the HUID on the invoice and a HUID record are both voluntary, capturing the HUID per tag is a service and fraud-control feature the owner chooses, not a statutory duty; what the invoice must carry is net weight, purity and hallmarking charges per article, which the tag's own attributes supply.
Serial-level attributes (a bag of per-unit fields, like Busy's up to ten per-serial descriptions [W busy-serial]) and a quantity in grams beside the count of one are needed; valuation by weight is a separate feature that this document does not cover.

### 3.6 Books

An ISBN-13 is an EAN-13 with prefix 978 or 979, the same check-digit rule; a 10-digit ISBN becomes a 13-digit one by prefixing 978 and recomputing the check digit [W wiki-gtin, wiki-ean13]. No serials, no batches; an edition is an item. The barcode on the cover is the item key.

### 3.7 Hardware, paint, tiles

Occasional batch use where **the lot matters to the customer** (a shade or dye lot for paint and tiles, so one job gets one lot) [J, unverified in the sources]; serials only for power tools and appliances.

---

## 4. The workflows, side by side

A cell says what a correct system does. "Leaders" names evidence; "Comsky" is the code today [C].

| Workflow | Batch rule | Serial rule | What the leaders do | Comsky today |
|---|---|---|---|---|
| Opening stock | per (godown, batch) with mfg, expiry, MRP, cost; sums to the item's opening quantity | a list of units with purchase cost and date; count equals the quantity | Busy and Tally take batches at opening [W busy-batch, tally-batches]; Zoho 200 serials at opening [Ws zoho-adv]; Odoo needs an adjustment to give existing stock lots [W odoo-lots]; SAP B1 needs zero stock to switch tracking on [Ws sap-b1] | no batch opening in the item master; only Split Company writes `openingAlloc.batches` |
| Purchase receipt or invoice | batch name, mfg, expiry, MRP, quantity per line; several batches per line; same name with a different expiry or MRP warns | scan or paste each unit; count equals quantity; a unit already in stock is refused; a unit sold before may come back as a new inward | Busy's window opens when the quantity is typed [W busy-batch]; Odoo refuses to validate without numbers [W odoo-lots]; Logic F2 start plus quantity [W logic-imei] | `Alt+B` name and expiry on a line; no checks |
| Sale or POS | proposed by first-expiry-first-out, override visible, expired blocked, a minimum remaining life honoured, selling above MRP warned; a line splits when one batch cannot cover it | scan the unit; it must be in stock at this location; quantity is the count; the invoice prints the serial and the warranty end | Busy and Tally: manual pick [Ws busy-search; W tally-guide-godown]; Zoho: list sorted by expiry [W zoho-first-expiring]; Odoo: proposed by removal strategy [Ws odoo-removal]; Busy and Zoho scan [W busy-serial, zoho-barcode-serial] | manual `Alt+B`; no check of any kind |
| Delivery Note then invoice | the invoice carries the note's allocation and stock moves **once** | same: the delivered units are the invoiced units | Tally links them with tracking numbers [W tally-tracking] | **no link: both move stock** (so a serial would be issued twice) |
| Sale return, credit note, rejection in | back to its batch if sellable, else a quarantine; the batch is named on the return | allowed only if the unit's state is "sold"; by default to the same customer; state becomes "returned" and may need a refurbish step | Zoho: a returned lot needs a new reference, damaged goes credit-only [W zoho-returns]; Busy: "returned batch still showing" is fixed by correcting batches in the On Account report [W busy-faq 53] | Credit Note only; no Rejection In voucher (parity row missing) |
| Purchase return, debit note, rejection out | from the original batch; **an expired batch may be returned even when sale is blocked** | the unit must be in stock; it leaves as "returned to supplier" | Tally: the expired-batch switch exists so expired stock can go back as a debit note [Ws tally-expired]; Busy and Logic: expiry returns against the supplier bill [W busy-medical, logic-pharma] | Debit Note only; no Rejection Out |
| Godown transfer (Stock Journal) | batch moves with its cost; a plain journal carries it too | the unit's location changes; one unit in one place | Tally: only the Stock Journal class mirrors batches [Ws tally-stockjournal]; Busy: needs batch numbers entered on the transfer [W busy-faq 50] | lines carry `batchName`, but `byBatch` has no godown, so a transfer changes no batch figure and per-godown batch stock is unknown |
| Physical stock | count per (godown, batch) | **scan every unit**; the system lists missing and extra units | Busy advises periodic reconciliation of serials [W busy-serial-tips]; its Physical Stock makes a stock journal [Ws]; no scan-based count seen anywhere | a count resolves against (item, godown) only, ignoring batch (CF-9 will make it (godown, variant)) |
| Adjustment, write-off | expiry write-off is an expense; damaged and shrinkage by reason | scrap with a reason; the history stays | Busy: expired stock is written off as a loss in the income statement [W busy-faq 5]; Zoho: adjustments add or pick serials and batches [W zoho-adjust] | Stock Journal only; no reason codes |
| Manufacturing: consumption | choose the component's batch (proposed first-expiry-first-out); per-batch cost feeds the output's batch cost | consumed units leave as "consumed" | Zoho and Odoo assemblies [Ws zoho-composite; W odoo-lots] | Manufacture hardcodes Main Location and takes **no batch or expiry** [C `ROADMAP-ADDENDUM.md:89`] |
| Manufacturing: output | a new batch with mfg date = voucher date and expiry = mfg plus the item's shelf-life days | serials generated (auto numbering) and their labels printed | shelf-life days: Tally (mfg date plus shelf-life period) [W tally-batches-setup via the help summary], Busy (expiry days on the item) [W busy-batch], Odoo (day-counts from receipt) [W odoo-expiry] | none |
| Job work out and in | stock lies at the job worker's godown but remains ours; the returned finished lot gets the worker's production batch | the unit's holder is the worker | Tally Material Out and In [C parity rows, not read in Tally] | job-work chain is missing (parity rows `material-in`, `material-out`, `jobwork-godown-type`) |
| Alter, cancel, delete a voucher | re-check the batch chain from that date | re-check the unit's chain; **refuse** when a later event depends on it (a purchase whose unit was sold) and **list the dependants** | Busy edits the voucher to fix serials and batches [W busy-faq 44, 53]; none documents a chain check | the engine derives from the log, so the **mechanism** exists; no check |
| Copy, recall, Bulk Entry, import | copy keeps the batch only as a suggestion | **copy never carries serials** | Alt+2 already drops bill allocations for the same reason [C] | Bulk Entry refuses batch items [C]; the importer drops batch dates |
| Two devices (sync) | per-batch negative caught at sync | the same unit sold on two devices is caught at sync | no leader documents it; Comsky's server already owns voucher numbering, which is the precedent [C `backend/src/numbering.js`] | nothing |

---

## 5. The reports people actually use

| # | Report | The question it answers | Who asks | Seen in | Comsky |
|---|---|---|---|---|---|
| 1 | Batch-wise stock, by godown | what batches do I hold, how much, expiring when | every batch user | Tally Batch Summary [W tally-batch-summary]; Busy [W busy-batch]; Zoho Batch Details [W zoho-reports] | Stock Query, one item, no godown |
| 2 | Expiry ageing in buckets (expired, 0-30, 31-60, 61-90, 91-180, later) | what do I act on this month | pharma, food | Tally's Stock Ageing by days to expiry [W tally-batches]; Odoo groups lots by expiry [W odoo-expiry] | none |
| 3 | Expiring or expired batches by date range and by supplier | what to return | pharma | Busy Expiring Batches [W busy-batch]; Marg Near Expiry / Expired report by date range, supplier and store [W marg-near-expiry]; Logic by supplier [Ws logic-expiry-title] | none |
| 4 | Batch trace: backward (supplier bill, receipt) and forward (customers, quantities) | a recall, a complaint | pharma, food | Tally Batch Vouchers, one batch [W tally-batch-summary]; Zoho batch page [W zoho-batch]; NetSuite Lot Traceability [Ws netsuite]; Odoo add-ons [Ws odoo-trace] | none |
| 5 | Batch-wise sales and profit | which lots earn | pharma, FMCG | Busy (needs the batch on every sale) [W busy-faq 73] | none |
| 6 | Batch-wise purchase price | what each lot cost | pharma, FMCG | Busy Batch Wise Purchase Price [W busy-faq 73] | none |
| 7 | Unallocated or on-account batch entries | where batch data is missing | any batch user | Busy Batch On Account [W busy-batch] | **latent defect:** a line with no batch is skipped in `byBatch`, so batch totals do not add up to the item [C `inventory.ts:141`, `:156`] |
| 8 | Serials in stock, by item, godown and age | what do I hold | mobile, electronics | Busy Serial No.-wise Stock [W busy-serial]; Zoho Serial Number Details [W zoho-reports] | none |
| 9 | Unit history: supplier, receipt, moves, sale, customer, returns, warranty | where is this unit | mobile, electronics | Busy Ctrl+Q query [W busy-faq 36], and a Busy user asks for an easier screen for one serial's whole history [W busy-reviews-ss]; Zoho click-through [W zoho-returns] | none |
| 10 | A customer's units | which phones did this person buy | warranty, theft | Accountune says search by customer or IMEI shows warranty status [W accountune-electronics, vendor] | none |
| 11 | Warranty due, expired and under claim | what to expect | electronics | Busy warranty history [Ws busy-search]; Accountune [W vendor] | none |
| 12 | Unit profit | margin per phone | mobile | not documented by any product I read [unverified] | none |
| 13 | Pending serials (received without identities) | what is not yet scannable | mobile, electronics | Odoo prevents it by validation [W odoo-lots]; Busy parameter entries land on account [W busy-faq 21] | none |
| 14 | Stock-take difference (scan against books) | what is missing or extra | everyone with units | no product documents it; Busy advises periodic reconciliation [W busy-serial-tips] | none |
| 15 | Return list by supplier with the original bill | what to send back | pharma | Busy [W busy-medical]; GoFrugal [W gofrugal-expiry]; Logic [W logic-pharma] | none |
| 16 | Dead and slow stock by batch age | what is stuck | all | Busy [W busy-mobile] | Stock Ageing by item only |
| 17 | Schedule H1 register | statutory | pharmacies | Logic blocks sale without patient and doctor [W logic-pharma, vendor] | none |
| 18 | Exceptions: IMEIs failing the check, sold twice, not in stock, TAC mismatch | data quality | mobile | none documented | none (the Data Check screen is the natural home [J]) |

---

## 6. Data-entry ergonomics: how units and batches get into the system

Capture is where these features are won or lost. A shop that has to key forty IMEIs per carton stops using the feature, and a pharmacy that has to scroll a list of twelve batches to find the right one at the counter goes back to typing a name. No product I read publishes timings, so everything below about speed is [J]; what the leaders *offer* is evidence.

### 6.1 The entry methods the leaders offer

| Method | What I found | Tag |
|---|---|---|
| Type one by one | Everyone. Tally's only route for a unit is typing its IMEI as a batch name | [W tally-erp9-unsold] |
| **Scan into the unit field** | Busy: serial scanning in item setup, purchase and sales vouchers, **only when barcode reading is enabled**; Zoho: a serial can be scanned into the field; Logic ERP: at sale the voucher's code field is set to "serial mobile number" and an IMEI scan fills model, price and warranty. Tally scans into the **item** field only (its alias) | [W busy-serial; W zoho-barcode-serial; W logic-imei; W tally-barcodes] |
| **Paste a list** | Odoo: one serial per line, pasted from a spreadsheet, with "keep current lines" to add or replace. SAP B1: Excel entry of serials and batches arrived in 9.3. Logic lists "bulk billing of serial and IMEI numbers from Excel" on a distribution page. Busy and Zoho: **none documented** | [W odoo-serial; Ws sap-b1; Ws logic-elec] |
| **Range or "first plus a count"** | Odoo: first serial plus a count, and it infers the pattern. Logic ERP: F2 bulk window with a start IMEI, a start row and a quantity. SAP B1: generate from a serial string, or "automatic serial creation on receipt". Busy: numbers generated from a configured structure, with F7 (repeat the last) and Alt+F7 (the next) on a purchase voucher | [W odoo-serial; W logic-imei; W sap-learning; W busy-serial] |
| Auto-numbered batches | Busy (a batch numbering mode on the item); SAP B1 creates a batch record with a system number when "on release only" | [W busy-batch; W sap-learning] |
| Receive now, name later | SAP B1's "on release only" method: nothing is asked at receipt, the identity must be chosen before the stock leaves. The opposite, Odoo, **refuses to validate a receipt** until every tracked line has its numbers | [W sap-learning; W odoo-lots] |
| Excel into a voucher | Tally: an older paid add-on that imports batch and serial numbers into purchase vouchers (a 2017 manual). Busy: an FAQ says batch details can be changed in bulk by "bulk updation" or an Excel import, another says there is **no shortcut**, edit each purchase voucher; the two answers disagree | [Ws tally-erp9-excel; W busy-faq 66; W busy-faq 67] |
| Copy from the last receipt | Busy: F7 for the last serial only. No leader documents "same batch and expiry as the line above" or a default drawn from the last receipt of the item | [W busy-serial] |
| Bulk edit of dates on a saved voucher | Busy: open the saved voucher, Enter on the item name, edit manufacturing or expiry in the batch window; the new date is not propagated anywhere else | [W busy-faq 44] |

**What this says [J].** The leaders offer scan, type and a *generator*; only Odoo and (half) SAP B1 offer a *paste*, and only Logic ERP offers an IMEI generator whose page says nothing about the check digit. Nobody documents "scan the whole carton, then reconcile against the invoice quantity".
For an Indian mobile trader the carton and the supplier's invoice already carry the IMEI, so scan and paste are the two that matter; a range is a convenience that is only safe when it steps the 14-digit body and recomputes the check digit (section 3.1.1).

### 6.2 The serial panel (the shape the evidence points to) [J]

The panel is a dialog opened from a tracked line, not a column in the grid, because one line can carry hundreds of units. Rules, each from a leader's behaviour or a trap in section 3 or 7:

1. **A counter, not a prompt.** "14 of 20 entered". The line's quantity is the target; the panel never closes by itself when the count is reached (a stray extra scan must not be lost) and never accepts the voucher.
2. **Every scan appends and is checked at once**, against four things in this order: *format* (Luhn for an IMEI item: a warning mark, never a block, since a real handset can carry an unusual number [J, consistent with the sibling document `tracking-verticals-india.md` section 3.3]); *duplicate in this panel* (a double scan is ignored with a quiet tone, because the second beep is the operator doing exactly what a barcode gun does when it is pulled twice); *duplicate in the books* (the unit is in stock, or sold and never returned: a red row naming the document and date); *wrong item* (the scanned type code differs from the item's other units: "different model?", a prompt not a refusal, section 3.1.1).
3. **Paste splits on any separator** (newline, comma, space, tab) and adds all rows, each judged by rule 2; Excel columns of `IMEI1, IMEI2` paste as pairs for a dual-SIM item.
4. **Range is "first plus count", per item type**: for an IMEI item it steps the 14-digit body and recomputes the check digit; for other serials a prefix, a numeric run and zero-padding (`SN-0009` to `SN-0012`); the preview shows the first and last generated value before anything is added.
5. **A second identifier is part of the unit.** One row holds `IMEI 1`, `IMEI 2` (and a serial for a handset that has one); searching either finds the unit; both are held unique in the company (Busy documents one serial field per unit and "up to ten descriptions" [W busy-serial]; the second IMEI of a dual-SIM phone is an attribute there at best [J]).
6. **Per-unit extras sit in the row, not in a second dialog**: cost (for used or margin-scheme goods), MRP, colour or storage variant, warranty months, and a free-text note. The columns shown are the ones the item turns on (Busy: MRP and sale price per serial, up to ten descriptions [W busy-serial]).
7. **Esc and click-away keep scanned work.** A panel holding typed or scanned rows asks before it closes, with a visible Close and "Keep editing" armed (the Back-control rule B5, section 1.3); a panel with nothing in it closes at once.
8. **The panel never takes the Enter that accepts the voucher.** The code audit shows a scan burst reaching the Narration field raises Accept [C, `batch-serial-code-audit.md` section 3.14]; the panel owns its own keys while open, and a scan outside any scan target says "scan into the item field" and is swallowed.
9. **A wrong identifier is corrected in the panel, not by re-walking the invoice.** The one detailed Busy serial review complains that fixing a wrongly entered serial takes more steps than expected [W busy-reviews-ss]. The row's identifier is editable in place; the edit is **re-judged as a new entry** (rule 2) and, if the unit has moved on since
   (sold, transferred, returned), goes through R7's chain check: the screen lists the later documents that name the unit, and the correction either carries through them, with a reason, or is refused. One correction should not cost four screens. [J]
10. **A return starts from a scan.** The same reviewer asks for faster scanning of the IMEI on a return [W busy-reviews-ss]. On a Credit Note or sales return, a scan in the return field finds the sale of that unit by **any** of its identifiers, shows the customer, date and warranty end, and fills the line; if the unit was never sold, or not to the party chosen, it says so and adds nothing (R5). [J]

### 6.3 Quantity and the serial count

| Case | Rule | Why |
|---|---|---|
| Sale, issue, transfer, purchase return | The quantity **is** the number of serials chosen; it is not typed. Mismatch cannot be saved | Odoo makes the quantity follow the serial count [W odoo-serial]; a unit sold twice or a quantity that is not a whole number is meaningless |
| Purchase receipt, opening stock | The quantity is typed first (as Busy's window opens after the quantity and unit [W busy-batch]); the panel then asks for that many. A shortfall may be **saved as "pending identities"** on a purchase only, shown in the grid as "3 not yet scanned", listed in a report (row 13 of section 5), and blocking any sale of those units because they have no identity | A carton often reaches the counter before it is scanned; Odoo's refusal to validate a receipt is the strict alternative and is wrong for a shop that must book the bill today [J] |
| Quantity edited later | Lowering it below the serial count asks which units to drop; raising it adds empty slots | The line must always agree with its list |
| A unit-less item that is serial-tracked in the item but has no stock | Allowed; nothing to pick | Busy lets a person start serial tracking late, for later transactions [W busy-faq 76] |
| Alternate unit (a box of 10) | A serial item is counted in single units; a box unit is refused with a sentence | A "box" has no one IMEI [J] |

### 6.4 Batch entry

**On a purchase line** (the row is a mini-grid, one row per batch, because one supplier line often covers several batches): batch number, manufacturing date (optional), expiry, MRP, quantity, free quantity, with the cursor order matching the printed supplier invoice, which in pharma reads batch, expiry, MRP, quantity, free quantity, rate [J; the column order is how I understand Indian pharma purchase invoices, not a documented fact].

- *Expiry formats.* Accept `08/27`, `0827`, `Aug-27`, `31-8-2027` and read a month-only value as the **last day of that month** (section 3.2). Today's prompt takes a d-m-yyyy date with the existing mnemonics and nothing month-only [C `VoucherScreen.tsx:897-902`]. Where the item carries a **shelf life in days**, the expiry is proposed from the manufacturing date (Busy: expiry days on the item; Tally: manufacturing date plus a shelf-life period; Odoo: day-counts from receipt [W busy-batch; W tally-batches; W odoo-expiry]).
- *Defaults.* Mfg and expiry may be required or not per item (Busy's two switches [W busy-faq 44]); MRP is proposed from the batch just before it, or the item's MRP when there is none.
- *Same name, different attributes.* A batch name already in stock with a different expiry or MRP raises "same batch number, different expiry: this is probably a different batch" (Busy has a duplicate-batch switch, and its FAQ says two batches with one id cause reporting trouble, resolved by consolidating them [W busy-faq 73]).
- *Scan.* A GS1 element string on a carton or pack (GS1-128 or DataMatrix: GTIN, batch, expiry, serial) fills the row **when it parses**. **The top-300 drug brands' codes are not a safe source**: CDSCO fixes the fields a code must carry but leaves its identifier to the manufacturer and names no encoding, and GS1 says many are QR codes that hold only a web address
  [W cdsco-faq; Ws gs1-india-position] (section 3.2). So a scan fills the batch row only if it parses; otherwise the batch and expiry are typed from the supplier's invoice or the pack's printed text. A batch name printed as a linear code can still be scanned into the batch field.

**On a sale or issue line**: the picker lists the batches **held in this godown**, each with quantity, expiry, MRP and age, **proposed in first-expiry-first-out order with the proposal pre-selected** (Zoho sorts the list soonest-expiry first and the operator picks [W zoho-first-expiring]; GoFrugal's sales bill has an option to list FIFO or shortest-expiry first [W gofrugal-batch]; Odoo proposes by the removal strategy [W odoo-fefo]; Busy's batch window lets the operator pick, and its FIFO utility allocates batches to past sales afterwards [W busy-faq 53]). Enter accepts the proposal; typing a few characters of a batch number narrows the list; a quantity larger than one batch splits into the next batch automatically (Zoho splits an invoice line when batches differ in price and quantity [Ws zoho-batch-price]; Odoo fills the remainder from the next lot [W odoo-expiry]). The proposal is never silent: the chosen batch and its expiry stay visible on the line (the repository's own rule that a carried value is always visible, CLAUDE.md Phase 20-21).

**On a return** the batch is the original one by default (section 4); a Zoho return gets a **new** batch reference because a returned lot cannot rejoin the old batch [W zoho-returns]. For Comsky the old reference is kept and the line says "returned" (section 7, R5): a second reference makes batch totals harder to read for a pharmacy that wants "how many of batch X did we sell and take back".

### 6.5 Bulk scan on purchase, step by step [J]

1. Open the purchase, pick the supplier, type the supplier's bill number and date.
2. Scan the first **item** barcode (EAN) or type the model; the row is created.
3. Type the quantity from the invoice, press Enter; the serial panel opens with the counter at 0 of N.
4. Scan every unit; each beep adds a row; the panel flags anything wrong in place.
5. Close the panel (Enter on an empty field or the Close button); the line shows "20 units".
6. Next item; repeat. At the end the Accept bar lists any line with fewer units than its quantity.
7. After Accept, a one-page summary says "20 units received, 18 IMEIs scanned, 2 pending" with a Print labels action (section 8).

Everything in 2 to 4 must work on a keyboard-wedge scanner, because the live site is HTTP and a camera needs a secure context [C `batch-serial-code-audit.md` section 3.14; Ws MDN].

### 6.6 What to design against (input mistakes)

| Mistake | What the system does |
|---|---|
| The same unit scanned twice in a row | Ignore the second, a quiet tone, the counter does not move |
| An item barcode scanned where a unit is expected | "That is an item code; scan the unit's IMEI" and nothing is added |
| The second IMEI of a dual-SIM phone scanned after the first | The row for that unit already holds it: "already on this line" |
| A box IMEI that differs from the phone's | Outside software; the receipt checklist says to dial `*#06#` on a sample, and the row carries a "checked by hand" tick for a used phone (section 3.1.2) |
| A typed batch name in a different case (`B-17` and `b-17`) | One batch: names are trimmed and compared without case; the engine today treats them as two [C `batch-serial-code-audit.md` section 3.2.2] |
| A 13-digit scan landing in the quantity field | Not accepted as a number above a sane limit; the screen says "scan into the item field" |
| A scan during the "Accept ?" bar | Ignored (Y, N and Enter answer that bar today [C `batch-serial-code-audit.md` section 3.14]) |
| Paste with a blank line, trailing spaces, or a header row | Blank lines dropped, values trimmed, a header row recognised by a non-numeric first line and skipped with a note |

---

## 7. Integrity rules: what must always be true

These are the rules a design has to enforce, ranked in the table by how much damage a breach does. For each: the rule, the default strictness, who may override, and the evidence. **Strictness is a design lever [J]:** the leaders are permissive on several of these by default (Tally lets expired stock be sold, Busy's batch setting reads "Allow with warning", neither documents a serial rule for returns), and being strict by default, with a named override and a recorded reason, is a cheap way to be better than both.

**The enforcement point [J, from `batch-serial-code-audit.md` sections 3.10 and 5].** The rules live in **one pure function** over the voucher log (state of a unit or a batch cell *as at a date*), called from four places: the entry screen at Accept, the write funnel (so Bulk Entry, imports, Mark Optional and REST are covered), the Data Check, and the post-pull scan. Comsky already derives every other stock figure from the log, so a unit's state is derived the same way and cannot drift from the vouchers.

### 7.1 Serial and unit rules

| # | Rule | Default | Evidence and reasoning |
|---|---|---|---|
| **R1** | A serial identity is **unique per item among live units**. It may reappear only after the unit has left (sold, returned to a supplier, scrapped) **and** come back as a *new inward* (a customer return, a trade-in, a repaired unit) | **Block** at entry; an administrator may allow one named exception (a legitimately reused number) with a reason | Busy's answer to "how do I allow duplicate serials" is "not currently available" (read as: it enforces uniqueness) and it offers its parameter feature as the alternative for goods that repeat [W busy-faq 67]; Zoho and Odoo describe serials as unique identifiers [W odoo-serial; Ws zoho-serial]. The case Busy does not document is the **second life** of one unit |
| **R2** | A unit is in **one place and one state** at a time: *in stock at godown G*, *with a customer*, *with a service centre*, *returned to supplier*, *scrapped*, *reserved to a sales order* | Derived, never typed | SAP B1: once a serial is allocated to a sales order it cannot go on another document until the allocation is cancelled [W sap-learning]; Busy shows serials "in stock" per material centre, and an option exists to "pick serial numbers from all material centres", which implies the default is the voucher's own centre [W busy-faq 46; J on the default] |
| **R3** | A sale, issue or transfer needs the unit **in stock at that godown as at the voucher date and not out again later** (a date-ordered rule, not a current-state flag) | **Block**, naming the blocking document | Busy's FAQ lists "the sale voucher's date is earlier than the purchase date" as one reason a purchased serial is not offered at sale [W busy-faq 46]; voucher dates are user-editable here and the engine orders by (date, number) [C `batch-serial-code-audit.md` section 3.10, "Date order" row]; a backdated purchase after its sale is a real case |
| **R4** | **Quantity equals the number of units** on the line; for a serial item the count *is* the quantity | **Block** at Accept (a purchase may park the shortfall as pending identities, section 6.3) | Odoo lets the quantity follow the serial count and warns before a receipt without numbers [W odoo-serial; W odoo-lots]; SAP B1 auto-creates successive serials for a purchased quantity [W sap-learning] |
| **R5** | **Returns.** A serial return is allowed only if the unit's last event is a *sale to that customer* and the IMEI in hand matches; the unit becomes "returned" and goes back to stock only after an explicit disposition (restock, refurbish, supplier return, scrap). A **batch return** goes back to the **original batch cell** unless the person chooses quarantine; damaged or expired returns go to a "not for sale" cell or are credit-only | **Block** (serial); **default to the original batch** (batch) | A return of "a different handset of the same model" is the fraud pattern the mobile vendors describe [W mybillbook-imei]; Zoho's rule is the opposite for batches: a returned lot cannot rejoin the old batch, it gets a new reference, and damaged or expired units are credit-only and never re-enter stock [W zoho-returns]; Busy's answer to "the returned batch still shows" is to fix the batch on the original entry [W busy-faq 53] |
| **R6** | **Supplier return, DOA and rejection out**: the unit or batch quantity must be in stock; an **expired batch may still be returned** to its supplier even where sale is blocked | **Block** unless in stock; expiry never blocks a return | Tally keeps expired stock usable *by default* precisely so it can be returned to the supplier or sold, and exposes a switch [W tally-expired]; Busy and Logic route expiry returns against the original supplier bill [W busy-medical; W logic-pharma] |
| **R7** | **Alter, cancel or delete** a voucher cannot orphan a later event: a purchase whose unit was sold afterwards, a sale whose unit was later returned, a transfer whose unit moved again. The screen **refuses and lists the dependants** | **Block**, with "Cancel instead" and the dependants' numbers | The mechanism exists (stock is derived from the log) but no leader documents a chain check; Busy fixes a wrong serial by editing the saved invoice [W busy-faq 53]. Wave 3's Cancel (Alt+X) keeps a shell with no lines, so units are released automatically [C `batch-serial-code-audit.md` section 3.12] |
| **R8** | **Copy, recall, Alt+2, Bulk Entry and import never carry units** (and an outward batch is only a suggestion). A copied serial would be sold twice | Dropped on copy; imports validate each row | Alt+2 already drops bill and instrument allocations for exactly this reason (CLAUDE.md Phase 13); Bulk Entry today refuses batch items [C `BulkEntry.tsx:60-61`, `:486-489`] |
| **R9** | **Identifier format.** An item flagged IMEI: 15 digits, Luhn check, a **warning** (never a block) when it fails; a non-IMEI serial takes any text of up to 20 characters (the e-invoice `PrdSlNo` limit, `tracking-verticals-india.md` section 3.2); all identifiers of a unit are **unique** in the company and **resolve to the same unit** | **Warn** (format), **Block** (duplicate) | IMEI structure, Luhn and its blind spots: section 3.1.1 [W wiki-imei, wiki-luhn]. Busy documents a 6 to 8 digit serial structure, which cannot be an IMEI [W busy-faq 76] |

### 7.2 Batch rules

| # | Rule | Default | Evidence and reasoning |
|---|---|---|---|
| **R10** | **A (godown, item, batch) cell never goes below zero.** The existing "will go NEGATIVE" warning becomes per cell and names the batch | **Block** for a tracked outward line; the company's negative-stock switch, if on, downgrades it to a warning | Busy: a negative-stock warning switch per batch [W busy-batch]; Busy also fails to carry a negative batch balance into the next year [W busy-faq 65]; SAP B1: tracked items cannot go negative [Ws sap-b1]. Comsky's only guard today is per item, with Continue anyway [C `VoucherScreen.tsx:2489-2505`] |
| **R11** | **Expired stock cannot be sold or issued**; an item can also carry "do not issue within N days of expiry" (and a customer can carry a minimum remaining life). Near-expiry inside the warning window **warns** | **Block** (expired), **warn** (near expiry); override by the Inventory right with a reason; purchase return and write-off exempt (R6) | Tally defaults to *allowing* expired batches and has **two** switches for it ("Allow use of expired batches" in the sales voucher's F12 Advanced, and "Honor expiry dates usage for batches" which hides expired batches from the sale list) [W tally-expired]; Busy's FAQ says expired stock can be sold with a warning when batch adjust is "Allow with warning" [W busy-faq 74]; Logic ERP's "prompt in billing if lot expiry is below N days" stops the bill being saved [W logic-expiry]; NetSuite warns N days before expiry at sale [W netsuite-lot]; the food rule is that a past use-by or expiry date "shall not be sold" [W fssai-compendium reg 2(1)(u)]; the Drugs Rules bar sale of an expired drug (see `tracking-verticals-india.md` section 3, Rule 65 row) |
| **R12** | **Batch identity.** (item, name) where the name is trimmed and compared without case. A name that already exists with a **different expiry or MRP** raises a warning ("probably another batch"). The batch's **attributes are snapshotted onto each line when it posts** (batch name, mfg, expiry, MRP), so editing a batch later never rewrites a printed invoice | **Warn**; snapshot always | Busy keeps a duplicate-batch switch and says two batches under one id break reports, fixed by consolidation [W busy-faq 49, 73]; the repository's own rule is to snapshot tax treatment on the line when posting (CLAUDE.md Phase 20-21); the engine today treats `B-17` and `b-17` as two batches [C `batch-serial-code-audit.md` section 3.2.2] |
| **R13** | **A held or recalled batch is not issuable.** A status (*active*, *on hold*, *recalled*, *quarantine*) with a reason; hold blocks sale and issue, never return or count | **Block** while on hold | Drugs rule: stock under withdrawal is kept apart, marked "not for sale" [R `tracking-verticals-india.md` section 3, Rule 65 row]; Insecticides Rules: expired stock is segregated and stamped [R same, INS row]; none of the leaders documents a recall status [J] |
| **R20** | **A month-only expiry is stored as the last day of that month** and the batch is expired on the first day of the next. It prints as MM/YYYY | Always | Food labels may state month and year only for shelf lives above three months, and **since 1 July 2022 may also use DD/MM/YY** [W fssai-compendium reg 5(10)(b)]; the month-end reading is the universal practice I know of and I could not cite an Indian rule for it [J] |

### 7.3 Rules across both

| # | Rule | Default | Evidence and reasoning |
|---|---|---|---|
| **R14** | **Two devices must not both sell the same unit** (or draw the same last pieces of a batch). The only honest promise is: *one device cannot do it*; *two online devices can be stopped by a server reserve at Accept*; *two devices with one offline cannot be stopped, only caught at sync*. The server records a `duplicate-serial` mismatch and **never rejects a push** (the stated policy of `conflict.js`); the resolution is an ordinary credit note or Alt+X, not a silent undo | **Detect and record**; a reserve at Accept narrows the window, with a provisional flag when the cloud is silent | `batch-serial-code-audit.md` sections 3.10 and 5 (risk 2): no existing conflict rule sees two different vouchers naming one IMEI; wave 3's server numbering is the pattern to copy (idempotent reserve, provisional flag, claim at sync, never renumbered by the server) [C]. Odoo's and SAP's serial controls assume one database and say nothing of offline devices [J] |
| **R15** | **A Delivery Note and the invoice that bills it issue the stock once.** Link them (a tracking number, as Tally does) or treat the note as the issue and the invoice as billing only; units on a note and on its own invoice are not a duplicate | **Link** | Tally's tracking number ties a delivery or receipt note to the invoice so the quantity, rate and amount carry; the article does not state outright that stock moves once, but its "Not applicable" choice is the case where the accounting voucher moves stock and no note exists [W tally-tracking]; Comsky has **no link** and both move stock [C `types.ts:36-38`, `inventory.ts:305-316`] |
| **R16** | **A transfer moves identity and cost, never creates or destroys.** The unit's location changes; the batch cell moves with its mfg, expiry and MRP; the receiving leg cannot be typed without the batch | **Block** if not in stock at source | Tally's inter-godown class mirrors items and batches to the destination, a plain Stock Journal's documented steps do not mention the batch [Ws tally-stockjournal; W tally-guide-godown]; Busy: a transfer is to be made with the serial numbers recorded so each unit's movement can be tracked [W busy-faq 49], and the first draft read that a batch moves **only if its numbers were recorded** on the transfer [W busy-faq 50, not re-read]; Comsky: the engine keeps batch stock without a godown, so a transfer changes no batch figure [C `inventory.ts:14`] |
| **R17** | **A count is per cell, and for serials a scan.** Physical Stock is counted per (godown, batch); a serial item is counted by scanning, and the screen lists **missing** and **extra** units, each needing a reason before the books move | **Block** silent adjustment | TallyPrime's Physical Stock picks the batch and fills its dates [W tally-physical]; Busy's physical stock entry creates a stock journal and asks for every item including zero [W busy-faq 57]; a scan-based count is documented by **no one** [section 5, row 14]; Comsky's count ignores batch, and a two-batch count turns 12 into 7 [C `batch-serial-code-audit.md` section 3.2.2] |
| **R18** | **An optional voucher never claims a unit or batch**; marking it regular later runs every check | Always | CLAUDE.md "optional vouchers must be excluded from every total"; Mark Optional / Regular writes outside `saveVoucher` [C `batch-serial-code-audit.md` section 3.12] |
| **R19** | **Costing.** Serial goods are costed by **specific identification** (each unit its own cost); batch goods keep the item's method with a per-batch cost *report* first; items with no tracking are untouched. No figure outside the new screens moves | Per item flag | The CF-9 draft's own stance (pooled per-item costing so no existing figure moves) is the safe one for variants and batches; serials need per-unit cost for used-phone margins and because 125,000 one-unit FIFO lots took 23.8 s in the engine [C `batch-serial-code-audit.md` sections 3.2.5 and 4.2, probes]; SAP B1 offers "every serial number has its cost" as a valuation method [Ws sap-b1-valuation] |
| **R21** | **Warranty is data on the unit**: type (purchase, sale, installation), start (defaults to the invoice date), months, end; editing needs a reason; it prints on the invoice and goes into the e-invoice `WrDt` when the buyer needs it | Set from the item, editable per unit | Busy keeps three warranty types and lets them change at voucher entry [W busy-serial]; the e-invoice schema has a warranty date and says nothing about start or end [`tracking-verticals-india.md` section 3.2] |
| **R22** | **A used handset received** (buyback, exchange, trade-in) carries "IMEI checked on date X, result Y, by whom" and the seller's identity as part of the receipt | **Warn** if missing | Entities buying or selling used mobile devices must check the government's database of tampered and blacklisted IMEIs before completing the transaction, paying a fee per check [W pib-dot, 17 November 2025]; section 3.1.2 |
| **R23** | **Overrides are rights and leave a trail.** Overriding R1, R3, R10 or R11 needs the Inventory right (or Administration for R1), asks for a reason, and writes the reason on the voucher | Reason required | Busy has "user rights for that field or feature" [W busy-faq 53]; Comsky's rights model already scopes Stock Journal and Physical Stock to the Inventory right [C `batch-serial-code-audit.md` section 3.15] |
| **R24** | **Nothing chosen is the app as it was.** Every rule above is keyed on an item or company switch that exists only after a choice; an untracked item is asserted identical by reference | Always | CLAUDE.md Phase 27 gotcha ("customisation must be opt-in, asserted by identity") |

### 7.4 The override ladder [J]

| Level | When | Who | Example |
|---|---|---|---|
| **Block** | the books would become false | nobody without a reason | sell a unit that is not in stock (R3); a duplicate live serial (R1) |
| **Block, administrator may allow with a reason** | the books are fine but the business may have a reason | Inventory right | sell an expired batch to a clearance buyer (R11) |
| **Warn, Continue armed** | a probable mistake | the operator | IMEI fails Luhn (R9); same batch name, different expiry (R12) |
| **Inform** | no action needed | nobody | near-expiry within the window |

### 7.5 What the leaders enforce, in one view

| Rule | Tally | Busy | Zoho | Odoo | SAP B1 |
|---|---|---|---|---|---|
| Serial unique | n/a (no serials) | **Yes** (duplicates "not available") | Yes (described) | Yes (described) | Yes |
| Unit in stock to sell | n/a | **Yes**, by voucher date and by material centre | Yes | Yes (delivery cannot create) | Yes; reserved by a sales order |
| Quantity equals count | n/a | Window after quantity | Selected units | Follows count | Yes |
| Return only of the same unit | n/a | not documented | needs the original sale | not documented | not documented |
| Batch cell not negative | not documented for batches | warning switch | not documented | not documented | **Yes** (tracked items cannot go negative) |
| Expired blocked | **default no**, switchable | **"Allow with warning"** | not documented | **alert only** | not documented |
| Held/recalled batch status | no | no | no | no | no [J] |
| Duplicate across devices | n/a | n/a | n/a | n/a | n/a |

---

## 8. Barcodes, scanning and labels: what the batch and serial work needs from them

**Scope and honesty.** The owner named labels and scanning as separate weak spots of Tally. This section covers what the *batch and serial design* needs from them and what the leaders do, and leans on two sibling documents for the rest: `batch-serial-code-audit.md` (sections 3.6, 3.13 and 3.14: the print path, what a scan resolves to, what a scan burst does to the key handler) and `tracking-verticals-india.md` (sections 3.3, 5.4, 6.5 and 6.6: identifier facts, the barcode registry, the label field catalogue). **The second pass added only what two makers' command manuals say about printing a label (section 8.7); I still could not research label-stock sizes, which printers Indian shops actually run, or scanner models, and I have no hardware** (section 12). Everything below about hardware, beyond 8.7, is [J] or [unverified], and the owner's own printer and scanner are the test.

### 8.1 One scan, many possible meanings

| Code | Typical form | Identifies | Who prints it | Tag |
|---|---|---|---|---|
| Item barcode | EAN-13 or UPC (India's company prefix is 890, issued by GS1 India) | an item (a retail pack) | the manufacturer | [R GS1IN, read in `tracking-verticals-india.md`] |
| Shop's own item or variant code | Code 128 or EAN-13 in the restricted block | an item, or a style-size-colour | the shop | [J] |
| Batch code | Busy's barcode type "Item+Batch No." | an item **and** a batch | the shop | [Ws busy-labels-faq] |
| Unit code (IMEI, appliance serial) | on the carton; the symbology is, I believe, Code 128 | one unit | the maker | [J, unverified] |
| GS1 code on a carton or pack (GS1-128, DataMatrix, QR) | application identifiers 01 (GTIN), 10 (batch or lot), 17 (expiry as YYMMDD), 21 (serial); variable-length fields end with a group separator | item, batch, expiry and serial **at once** | the maker | [Ws gs1-ai: the first pass's summary, re-checked 2026-10-23 against search summaries of three pages that are not GS1's own: AI 01 is 14 digits, AI 10 and AI 21 are variable up to 20 alphanumeric characters, AI 17 is six digits YYMMDD, and a separator follows a variable field unless it is the last; one source disagrees about 2D symbols and is not relied on] |
| Top-300 drug brands' QR or barcode | the **fields** are fixed (unique product code, generic and brand name, manufacturer, batch number, manufacturing and expiry dates, licence number; batches made from 1 August 2023); the **encoding is not**: the identifier is the maker's own and GS1 says many are QR codes holding only a web address | item, batch and dates **only if the payload parses; often just a link** | the maker | [W cdsco-faq; Ws gs1-india-position] |
| Price- or weight-embedded label | EAN-13 beginning 02 or 20 to 29 (reserved by GS1 for a retailer's internal use) with the item code and a weight or price block and a check digit | an item and a quantity or a price | a scale | [W wiki-ean13; Ws GS1-RCN] |
| ISBN-13 | EAN-13 with prefix 978 or 979 | a book edition | the publisher | [W wiki-gtin] |
| Case or pallet code | GTIN-14 | a case, not a piece | the maker | [R GS1IN] |
| Jewellery tag | the shop's own code; the **HUID** is a separate six-character alphanumeric on the article, not a barcode | a tag (a unit with weights) | the shop and the assaying centre | [W bis-jewellers; first pass bis-faq] |

### 8.2 The resolution rule [J]

A scan runs **one exact lookup through a registry**, in this order, and stops at the first hit: (1) a **unit** (any identifier of any live or past unit), (2) a **batch code**, (3) a **variant or item code**, (4) a **scale pattern** configured for the shop's scale, (5) a **GS1 string**, parsed into GTIN plus batch plus expiry plus serial and then resolved by (1) to (3). A miss says "code not found" and never appends anything (today an unknown scan is silent and the next one appends to it [C `batch-serial-code-audit.md` section 3.3]).

- **One barcode, several MRPs.** An item barcode with several live batches opens the batch picker (the proposed batch pre-selected) instead of picking one silently. Busy needs a "duplicate barcode" setting plus "allow multiple MRP for the same barcode" for exactly this case, and says so in three FAQs [W busy-faq 46, 53]. It is common in India because a maker reprints the same EAN with a new MRP [J].
- **A GS1 string is resolved, not trusted.** The batch and expiry in the code fill the row; the row is still validated against R11 and R12.
- **The scale layout is configuration, not code.** GS1 only reserves the prefixes; the position and length of the item and weight or price fields are the retailer's or the scale maker's choice [Ws GS1-RCN]. So the shop sets "2 prefix digits, 5 item digits, 5 weight digits, 1 check digit" once and the product parses against it; the worked example in `tracking-verticals-india.md` section 3.3 (`2100123012503` is item 00123 at 1.250 kg) shows the shape. No Indian scale maker's format was checked [unverified].
- **A check digit is not the same everywhere.** EAN-13 and GTIN use weights 1 and 3 over the first twelve digits; an IMEI uses Luhn (section 3.1.1). One validator for both is a bug [W wiki-ean13; W wiki-luhn].

### 8.3 What the leaders do

| Product | Scanning | Label printing | Tag |
|---|---|---|---|
| **Tally** | Scan into the item-name field; the code is the item's **Alias** (the help article says a scan records a 16-digit number there, Tally's blog says up to 32 characters; I could not reconcile them); the scanner's software is separate; unique per item; nothing for batches or units | The help page says nothing about creating or printing labels; Tally's product page says a generator matched to the scanner is needed; partner add-ons exist: a barcode and QR label printer in a partner's feature list, a barcode print module that prints item name, MRP, code and barcode per quantity, and an IndiaMART listing at Rs 5,000 for a lifetime licence; **no listing confirms a sticker that carries batch, manufacturing and expiry** | [W tally-barcode-help; W tally-barcodes; Ws tally-labels-addons] |
| **Busy** | Per **POS voucher series** (a switch); serial scanning in item setup, purchase and sales; a batch barcode can be scanned to pick the batch where configured | An **add-on** (Item Barcode Printing, then Configure Barcode Labels): label height and width, margins, default printer, sheet type; fields of type barcode, QR, data, image or text; values such as name, alias, price, MRP; sources: Busy data or an Excel sheet; from the item master or a voucher; the batch number can be a data field (barcode type "Item+Batch No."); generation needs the **Standard** edition or above; a barcode value must be **5 to 20 characters**; demo and Basic (B1) print at most 20 barcodes; **designing a format is a chargeable service** through a partner; QR on an invoice is B2C only, B2B is chargeable | [W busy-labels; W busy-faq 46; W busy-label-faq-5, busy-label-faq-8] |
| **Zoho Inventory** | A serial can be scanned into its field | Barcode and QR labels from an item or in bulk from the list, two default templates (linear and QR), Code 128 and others, a batch number as an encoded value for a batch-tracked item, paper from label sheets to custom sizes; the first draft read "one label per selected serial or batch number"; a re-search found no serial-specific steps; a community thread calls the generator limited (one field at a time, no labels at receiving or after choosing batches or serials) | [Ws zoho-labels; Ws zoho-community-labels; W zoho-barcode-serial] |
| **NetSuite** | n/a | Labels that show a lot number and its expiry date, printed from Print Checks and Forms | [W netsuite-lot] |
| **Vyapar** | barcode scanning (plan-dependent) | barcode generation and import are listed as **Gold-plan** features | [W vyapar-inventory] |
| **Odoo** | a separate barcode module (not researched) | not researched | [unverified] |

**What this says [J].** Nobody offers a label that is **born from the receipt**: print one sticker per unit received, carrying the batch, expiry and MRP just entered. Busy comes closest (labels from a voucher, a batch field), and puts the design behind a paid service. The leaders' label tools are all *item* tools; a unit or batch label is an afterthought. That is the room to be better, and the first cheap win is a handful of **fixed, tested label layouts** (a shelf label, a unit sticker, a batch sticker, a jewellery tag, a garment hang tag), not a free designer.

### 8.4 What a label must carry, by business (the field catalogue is in `tracking-verticals-india.md` section 6.6)

| Business | The label | Why |
|---|---|---|
| Mobile and electronics | model, storage and colour, price, an IMEI or serial **barcode**, warranty end; a shelf label with the model and price | the customer and the next cashier scan it; the IMEI on the invoice is what a CEIR block or a police report needs (section 3.1.2) |
| Pharma | a shelf label (name, batch, expiry, MRP, rack); a **loose-dispense label** (drug, quantity, dealer's name and address) for a drug supplied in a container other than the maker's | the Drugs Rules require the loose-dispense label [R `tracking-verticals-india.md` section 3, Rule 65 row] |
| FMCG and food packers | the **statutory pack label**: MRP inclusive of all taxes, month and year of manufacture or packing, net quantity, consumer-care details, and for food a batch or lot number, dates and licence number | Legal Metrology and FSSAI [W legal-metrology; W fssai-compendium] |
| Garments and footwear | style, size, colour, MRP, barcode, hang-tag layout | one code per variant, the CF-9 slice's partner |
| Jewellery | tag number, gross, stone and net weight, purity, **HUID**, a barcode | a tag is a unit with weights; one HUID per article; the HUID need not be on the invoice (3.5) [W bis-jewellers] |
| Grocery | a weighed-goods label from the scale (not from this software), and the shop's own MRP stickers | the scale owns the format |
| Books | an ISBN and a price sticker | the barcode on the cover is the item key |

### 8.5 The print path: what exists, what does not, what I could not check

- **In the repository** [C `batch-serial-code-audit.md` section 3.6]: a QR encoder with an independent decoder test; a millimetre-exact custom-page precedent (the cheque leaf); a print host and a PDF bridge; **no** 1-D barcode encoder, **no** label-size page (print pages are A4, A5 and Letter only), **no** silent print or printer list in the Electron bridge. Electron's `webContents.print` documents silent printing, a device name and a custom page size, and `getPrintersAsync` lists printers [R electron-webcontents, read in `batch-serial-code-audit.md`]; the unit of a custom page size was not stated on the page that was read.
- **A bar that does not scan is worse than no label.** The sibling audit's risk 10 is that a Windows driver resamples a bitmap barcode so module widths alias; the safe design draws bars as whole printer dots, checks them with an independent decoder in the test suite (as the QR code is), and ships a calibration sheet (the cheque pattern). [J, from that audit; no hardware was available to confirm it.]
- **Hardware not researched here [unverified]:** which command language (if any) the thermal printers Indian shops actually run take (section 8.7 gives what two makers' manuals say, not what a shop owns), which label sizes shops buy, whether a dedicated printer driver or a raw command path is the better first target, and how a Bluetooth scanner pairs. These need a printer and a scanner in the owner's hands before a build, not a desk study.

### 8.6 What this puts on the batch and serial design

1. A **barcode registry** that maps any code to a unit, batch, variant, item, pack or scale pattern, with a single exact-lookup function and a deterministic order (8.2).
2. **Check digits for each family** in pure, asserted functions: EAN and GTIN, Luhn for IMEI, the scale check digit.
3. A **scan target model**: only an item field or a serial panel takes a scan; anywhere else the scan is swallowed with a message (`batch-serial-code-audit.md` section 3.14).
4. **Print labels from the grid**: after a purchase (one per unit or per quantity received), from the batch list, from the serial list, from Stock Query; the quantity defaults to what was received.
5. **Fixed layouts first**, five or six, each tested on the owner's printer; a designer later.
6. **Text typing for long digit strings** in CSV and Excel exports (an IMEI or EAN opened in a spreadsheet becomes a number in scientific notation) [C `batch-serial-code-audit.md` risk 15].

### 8.7 Label printer languages: what the two manuals I read say [W zebra-bc, labelary-zpl, tsc-tspl; Ws printer-market]

Two text command languages cover most desktop label printers, from the two makers whose models appear in Indian marketplace listings (Zebra and TSC) [J for "most"; not counted]. Both are **plain-text command streams sent to the printer**, so a program can place every element in the printer's own dots, without a Windows driver redrawing the page.

- **ZPL (Zebra).** A label runs from `^XA` to `^XZ`; each field is a block ended by `^FS`; `^FO` places a field's top-left corner in dots; `^FD` carries the data (spaces inside it count); `^BY` sets the bar module width in dots (default 2) and the wide-to-narrow ratio; `^BC` makes a **Code 128** bar code with six parameters (orientation, height in dots,
  the printed interpretation line (default yes), that line above the bars (default no), a UCC check digit (default no; the mod-103 check digit is always present), and a mode); field data is limited to the label width. **Everything is in dots**: at 203 dpi a dot is 1/8 mm (8 dots per mm); 6, 12 and 24 dots per mm also exist, so **the same label prints at a different physical size on a different density** [W zebra-bc; W labelary-zpl].
- **TSPL / TSPL2 (TSC).** `SIZE` sets the label size in inches, millimetres or dots (the manual: 200 dpi is 8 dots per mm, 300 dpi is 12); `GAP` the gap between labels; `CLS` clears the buffer; `BARCODE x, y, "type", height, human readable, rotation, narrow, wide, "content"`; `PRINT` ends a label. The types listed include Code 128 (subsets switched automatically or by hand), EAN-13 and EAN-8 with optional 2- or 5-digit add-ons,
  UPC-A and UPC-E, Code 39, Code 93, Interleaved 2 of 5, Codabar and Postnet, with `QRCODE` and `DMATRIX` commands for the 2D codes [W tsc-tspl: the TSC programming manual, 189 pages, a 2009 to 2011 revision (the first 80 pages were extracted locally); current firmware may differ].
- **What this means for the design [J].**
  1. A label engine that emits ZPL or TSPL controls bar widths exactly, so a bar code is drawn in **whole printer dots** and checked by an independent decoder in the test suite (the code audit's risk 10). The cost: the app must send raw text to the printer, which a browser tab cannot do and which the Electron shell has no bridge for today
     (it saves a PDF only [C `electron/preload.cjs`]); a raw path (a network socket, a spooler raw job or a USB library) would have to be built, and I did not verify which is practical.
  2. The alternative, a PDF at a custom page size printed through the driver, works on any printer but leaves the bar widths to the driver; a bar that does not scan is worse than no label (8.5).
  3. Because positions are dots, **a label template should store millimetres and the printer's density, and convert at print time**; one template run on a 203 dpi and a 300 dpi printer is the likely failure.
  4. Marketplace listings show desktop label printers such as the TSC TE244 and the Zebra ZD220 at asking prices of roughly Rs 9,000 to Rs 19,000, and a 50 x 25 mm direct-thermal roll at about Rs 100 [Ws printer-market, IndiaMART listings]. Their resolution, which languages the shops' printers accept, and whether thermal-transfer or direct-thermal stock suits a jewellery tag or a pharmacy shelf label were **not verified**.
  5. Ask the first customer which printer and roll they own before choosing a target language; the doc's section 10 treats labels as "M plus hardware" for exactly this reason.

---

## 9. Constraints in this repository (what the research has to fit)

The full audit is `batch-serial-code-audit.md` (17 ranked risks, lanes and an order); this is the part of it a *product* decision depends on. Evidence tags: [C] read in code (symbols and sections are stable; line numbers are of the tree at the time of the first draft and of this session, wave 3 having been committed since as `2cee555`, with the lines named here unchanged), [J] judgement.

### 9.1 Thirteen constraints

| # | Constraint | What the design must therefore do |
|---|---|---|
| 1 | **The engine keeps batch stock without a godown** (`byBatch` is one map per item) and **Physical Stock counts against (item, godown) only** [C `inventory.ts:14`, `:104-127`]. The audit's probe turned a two-batch count of 12 units into 7 | Make the unit of stock a **cell** (godown, item, batch), with an explicit "no batch" cell so batch totals add up to the item; fix transfers, returns and counts to carry the batch. This **changes the Physical Stock result of existing batch books**, so it is a named, deliberate change with goldens, not a side effect |
| 2 | **The CF-9 slice (stock by a line field, in the worktree `.wt-cf`) changes the same engine functions in the same weeks**: `stockPositions`, `resolvedMovements`, `stockItemRows`, `pendingOrders`, `stockFlow`, `openingStock.ts`, `splitCompany.ts` and the negative-stock warning. Its spec (after a third review on 2026-10-22) keeps `byVariant` as a third parallel map beside `byGodown` and `byBatch`, and adds a `countPart` flag so a Physical Stock count made in parts is one movement; the engine part is not yet written in the worktree (only the `countPart` type is) [C `.wt-cf/docs/design/custom-fields-spec.md` AD.3.3; grep of `.wt-cf/src`, 2026-10-23] | Agree **one shared stock-cell key** with the CF lane-I owner **before CF-9a is written**: an item can be tracked by batch and by colour (a shade and its lot), and a third parallel map does not give (godown, batch, variant). **Reuse the reviewed count group for a batch count** (9.2, item 1) rather than inventing a second mechanism. Cost is cheap now and large after |
| 3 | **Where serials live decides everything.** A field on the voucher line rides undo, cancel, sync, backup, restore, the Mismatch Report and REST for free; a separate synced table must redo each of those, and at 1,000,000 serials means about 348 MB of outbox and 3,334 push requests (the audit's figures [C][P]) | **Serials on the line, as an array**; a derived, rebuildable **index** (in memory on the device, an event index on the server) instead of a source-of-truth table; no Dexie bump needed |
| 4 | **An older client that re-saves a voucher drops any line field it does not know**; the server's `udf` carry rule exists only for an unversioned write that states no capability [C `.wt-cf` §9.3] | A serial list needs that protection more than `udf` does, because dropping it frees the IMEIs silently. Design **one carry function with two bags** (`udf` and serials), one capability marker, one backup version number, one REST carry |
| 5 | **No rule in `conflict.js` sees two different vouchers naming one IMEI** (the unit of comparison is the document id), and the server's policy is "accept and record, never refuse a push" [C `batch-serial-code-audit.md` section 3.10] | R14: detect, record, resolve; a reserve at Accept narrows the window; a new mismatch kind needs no migration; a **backend deploy** gated on the owner's VM secrets, in one window with the pending wave-B rule and CF-6a |
| 6 | **Serial goods under FIFO or LIFO freeze every Accept**: lots are an array consumed with `Array.shift`, and 125,000 one-unit lots took 23.8 s in `stockPositions` (40,000 took 1.8 s) [C][P `batch-serial-code-audit.md` section 3.2.5] | Specific-identification costing for serial items; keep unit state out of `stockPositions`; replace `shift` before any serial item can be FIFO; a perf row with one-unit lots at 100k and 250k |
| 7 | **No link from a Delivery Note to its invoice**: both move stock [C `types.ts:36-38`, `inventory.ts:305-316`]; only orders are netted | R15: link them, or a serial delivered and then invoiced is issued twice and every duplicate check cries wolf |
| 8 | **A scan burst lands on whatever has focus**: letters are hotkeys on a non-input screen; Enter in Narration raises Accept; Y, N and Enter answer the Accept bar; an unknown item scan is silent [C `batch-serial-code-audit.md` section 3.14] | A scan detector at the key layer **before** the handler stack, and a scan target model (6.2, 8.6); a burst test as a gate |
| 9 | **A dialog that holds scanned work must not close on a stray Esc or click**; every `Box` gets an `onClose` and a visible Close; typed work registers a dirty probe [C `.wt-cf/docs/design/back-nav-spec.md` rule B5, B12] | The serial panel, the batch picker and the label dialog are `Box` with `onClose`, in that spec's dialog table, and register probes |
| 10 | **Nothing chosen must be the app as it was**, asserted by identity; the **Enter walk is frozen** and a new master field goes **last** [C CLAUDE.md Phase 24 and 27 gotchas]. Today `Alt+B` is the **only** way into the batch prompt, which never opens by itself [C `VoucherScreen.tsx:3059-3062`, `:3398-3410`] | Auto-opening the batch or serial panel in the walk changes the walk, so it happens **only for an item that opts in to a new tracking mode**; an existing `batches: true` item keeps `Alt+B` and gets a quiet "batch not set" hint until its owner chooses the new mode. Tracking fields are tail fields of the stock item form |
| 11 | **The e-invoice has one serial field per line (20 characters), a batch name of 3 to 20 characters, a warranty date and a 1,000-line cap** [R `tracking-verticals-india.md` section 3.2]; Comsky sends only the batch name and expiry, and drops a short batch name silently [C `einvoice.ts:358-359`, `:717-719`] | Keep one line per item with the serial array; expand to one line per unit only inside `einvoice.ts` for a buyer that demands it and only below the cap; warn at capture when a batch name will not go to the IRN |
| 12 | **Every interchange path loses batch detail**: the Tally importer reads one opening figure per item and drops per-batch openings and dates; the exporter never writes expiry [C `batch-serial-code-audit.md` section 3.7] | A switcher arrives with a broken book unless import reads batch allocations and dates and offers **"convert a batch-of-one to a serial"** for the IMEI-as-batch practice Tally itself documents [W tally-erp9-unsold] |
| 13 | **The parity board says `batches` and `expiry-reports` are covered while their notes say "No batch field on item or line"; the Batch & Expiry Summary menu row opens a four-line stub** [C `docs/parity-data.json`; `BatchSummary.tsx:1-4`; `MenuScreen.tsx:234`] | Re-state both rows as partial when work starts, and ship the report with the first release or remove its three entry points |

### 9.2 What to agree before building (the CF-9 meeting point)

1. **The shared cell key and one cell map**, with `byGodown`, `byBatch` and `byVariant` as projections of it (`batch-serial-code-audit.md` section 4.2). Name the Physical Stock behaviour change in the commit and in CF-9a's identity gate.
   **The reviewed CF-9 count rule is the template for counting by batch [J, from reading AD.3.3 on 2026-10-23].** The spec's rule is that a Physical Stock count is always a count of the **whole item at its godown**; a count by variant is one group of part lines, merged by `stockMovements` into one movement of their sum; variants held and not listed count 0; the entry screen
   pre-fills a row per held variant at its book quantity so an untouched row moves nothing; and a pure `countDeltas` spreads the item's delta over the variants, so that **no item-level figure depends on whether a caller passed a resolver**. A batch count has the same shape: the lines of one (item, godown) are the parts, each batch held and not listed counts 0, and the screen pre-fills a row per held batch.
   One thing differs and must be built, not assumed: the spec's merged movement keeps the **first part's line**, and today's `byBatch` update reads `batchName` from the movement's line, so merged batch parts would book the whole delta to one batch. The batch version needs its own spread of the delta over batches, generalising `countDeltas` from a variant key to a cell key.
2. **A "stock dimension" abstraction** for the voucher grid: a cell drawn only for tracked items, a stock hint in the picker, a negative-position warning that names the position, a Stock Journal row that starts from its source row's position, a Physical Stock line that counts exactly the position it names. CF-9c needs exactly this for a variant; a batch and a serial need the same machinery with a different editor and cardinality. Building it three times in one file gives three subtly different walks [J, same section].
3. **One carry function with two bags** (constraint 4) and one server release window (constraint 5).
4. **CF-9 stays out of manufacturing in v1**; batch and serial must still decide what manufacturing consumption and output do (section 4).

### 9.3 Where this document differs from its siblings (so a reader is not misled)

| Point | This document | Sibling |
|---|---|---|
| FSSAI sub-regulation numbers for the batch and date clauses | **5(9)** (lot, code or batch identification) and **5(10)** (date marking), as numbered in FSSAI's own compendium, Version-II of 29 December 2021 [W fssai-compendium] | `tracking-verticals-india.md` section 3 cites them as 4(9) and 4(10). Regulation 4 is "General requirements" in the compendium, so I read the sibling's numbers as a slip |
| How the batch prompt opens | **Only** with `Alt+B` or the Batch button [C `VoucherScreen.tsx:3059-3062`, button at `:3408`] | `tracking-verticals-india.md` section 2.1 says "or by Enter on a batch item"; the code does not |
| Month-and-year expiry on food | May also be written DD/MM/YY since 1 July 2022 for shelf lives above three months [W fssai-compendium reg 5(10)(b) proviso] | the sibling states month and year only |
| A Tally barcode's length | The help article says a scan records a 16-digit number, the blog says up to 32 characters [W tally-barcode-help; W tally-barcodes] | the sibling and the first draft said 32 |
| Scanning a drug strip or box to fill item, batch and expiry on the top-300 brands | Only where the payload parses. CDSCO's FAQ fixes the fields but leaves the identifier to the maker and names no encoding, and GS1 says many packs carry a QR that holds only a web address (3.2) [W cdsco-faq; Ws gs1-india-position]. The FAQ lists **eight** particulars | `tracking-verticals-india.md` section 4(b) takes a scan to fill item, batch and expiry (a "GS1 QR"), and section 3 says "nine data elements" (the FAQ of the 2022 notification lists eight; "nine" appears in a June 2026 report of the extension [Ws onmanorama-2026]) |

---

## 10. The ranked list: what would make this better than Busy [J]

**How I ranked.** Four things, in this order: (a) the damage to a switching customer if it is missing, (b) which business it serves (section 3), (c) how far it beats what Busy and the others *documented* (sections 2, 4 and 5), (d) what it depends on, because the first two are structural and everything else stands on them. A feature that is cheap and visible but stands on nothing is ranked by its dependency, not its charm. All ranks are judgement; the evidence is in the cited sections.

### 10.1 The parity floor: match this first, and do not call it better

What Busy's customers praise is concrete and copyable [W busy-batch; W busy-serial]. Of the evidence I reached, a page of 20 reviews mentions no batch work, and one electronics seller praises serial and IMEI tracking for warranty checks and finding a unit's invoice and godown [W busy-reviews-ss]; Capterra summaries say small teams and distributors like the batch-wise inventory [Ws busy-reviews]; the rest of the praise is GST updates and reports. So the bar is "it exists and the reports work":

- a batch with number, manufacturing date, expiry (or days), **MRP and sale price per batch**, and opening stock by batch;
- an expiry warning period, an action for an expired batch, duplicate-batch and negative-stock warnings;
- **Batch-wise Stock**, **Expiring Batches** and an exception report for lines with no batch (Busy's Batch On Account);
- serials by automatic or manual numbering, scanning, purchase, sale and installation warranty, per-serial MRP and price, a serial-wise stock report and an item serial query;
- barcode scanning at billing, and labels that can carry a batch.

Today Comsky has none of these as a working feature: a batch is a name and an expiry typed into a box that opens on `Alt+B` (section 1.1).

### 10.2 The ranking

| Rank | Feature | Serves | What it beats (evidence) | Stands on | Size [J] |
|---|---|---|---|---|---|
| **1** | **A unit engine with real integrity.** The serial state machine derived from the voucher log, as at a date (R1 to R9, R14, R18): unique live serials, one place and one state, in stock as at the voucher date, return only of the same unit, alteration refused with the dependants listed, copies never carry units | mobile, electronics, appliances, jewellery tags | Busy enforces uniqueness and offers units by centre and date [W busy-faq 67, 46] but documents no return rule, no format check, no dual-identifier handling and no cross-device check, and its one detailed user review calls correcting a serial clunky and asks for faster scanning on returns [W busy-reviews-ss] (6.2, rules 9 and 10); Tally has no serials at all and the IMEI-as-batch trick has none of these checks [W tally-erp9-unsold] | the shared cell key (9.2), the carry function (constraint 4), serials on the line | L |
| **2** | **Per-godown batch cells and a real batch master.** (godown, item, batch) with mfg, expiry, MRP, price, cost and status; opening stock by batch; transfers, returns and Physical Stock that carry the batch; MRP that follows the batch onto the invoice; the Batch-wise Stock, Expiring and exception reports in 10.1 | pharma, FMCG, food, chemicals | Parity with Busy and Zoho, and it **fixes the engine's own arithmetic** (a count that turns 12 into 7, a transfer that drops the batch) that a Busy user would find first [C `batch-serial-code-audit.md` section 3.2.2]. Tally has no per-batch MRP [W tally-mrp-page; Ws tally-mrp]; Zoho's four price methods are the model [W zoho-batch-price] | the same cell key; goldens for the changed Physical Stock result | L |
| **3** | **Proposed first-expiry-first-out at sale, with a visible override and an automatic split.** The batch picker lists batches held here, the soonest-expiring that meets the item's "do not issue within N days" rule is pre-selected, Enter accepts, a short batch rolls into the next; the line shows *why* | pharma, food, FMCG distributors | Busy and Tally: **manual pick** (Busy's FIFO utility works on past sales only) [W busy-faq 53; W tally-guide-godown]; Zoho: a sorted list, the operator chooses [W zoho-first-expiring]; GoFrugal: a sort option [W gofrugal-batch]; Odoo proposes by removal date [W odoo-fefo]. Nobody pairs the proposal with a per-customer minimum remaining life | 2 | M |
| **4** | **Expiry rules that protect.** Block an expired batch at sale unless an administrator allows it with a reason; warn inside the warning window; **never block a return** (R6, R11, R20); month-only expiry stored as month end; an expiry-ageing report (expired, 0 to 30, 31 to 60, 61 to 90, 91 to 180, later) and a Today and Dashboard card | pharma, food | Tally's default **allows** expired sale and has no pop-up [W tally-expired; W tally-guide-godown]; Busy "allow with warning" [W busy-faq 74]; Odoo: an alert, no documented block [W odoo-expiry]; only Logic's prompt stops the save [W logic-expiry] | 2 | S to M |
| **5** | **Scan-first capture with checks.** The serial panel (counter, check digit warning, duplicates, paste, range that steps the IMEI body, dual identifiers, pending identities, in-place correction, scan-driven returns, 6.2 to 6.5) and a scan detector with a scan target model; pack-scan for a drug's batch and expiry only where the payload parses (3.2) | mobile, electronics, appliances | Busy: type or scan, no paste or range documented; Logic: a range with no check digit documented; Odoo: paste and generate, no format check; **no one documents "scan the carton, then reconcile against the invoice"** | 1, the key-layer scan detector (9.1 row 8), a Back-control-compliant dialog | M to L |
| **6** | **A way over from Tally: import batch openings and dates, and convert a batch-of-one to a serial.** Tally documents IMEI-as-batch as its answer for phones, so a Tally mobile shop's book is full of one-unit batches | every Tally switcher with units | Busy offers a chargeable Tally-to-Busy conversion and its FAQ does not say what happens to batch fields [Ws busy-migration]; no product I read converts batches to serials. Comsky's importer today drops batch openings and dates (9.1 row 12) | 1, 2 | M |
| **7** | **"Where is it": one lookup and a trace.** An IMEI, a serial, a batch or a customer in; out comes the unit's history (supplier bill, receipt, moves, sale, customer, returns, warranty) or, for a batch, the **recall list**: stock held by godown, who bought it with their phone numbers, the supplier bill, and a **Hold** button (R13) | everyone with units or batches; pharma recall; theft and warranty claims | Busy: an Item Serial No. query in its Query system (`Ctrl+Q` is the query key; it shows a serial's linked transactions) and batch reference details [W busy-faq 13, 41, 57], while its one detailed user review asks for an easier screen for a serial's whole history [W busy-reviews-ss]; Tally: Batch Vouchers for one batch of one item [W tally-batches]; Zoho: click a serial or a batch for its history, and a Batch Transaction Summary naming customers [W zoho-reports, zoho-returns]; NetSuite and Odoo search a lot's transactions [W netsuite-lot; W odoo-lots]. None joins backward and forward and ends in a hold | 1, 2 | M |
| **8** | **Stock-take by scanning, with a difference list.** Scan the units on the shelf; the screen lists missing and extra units, each needing a reason before the books move; batches are counted per cell | every unit-tracked shop | Documented by **no product** I read; Busy only advises periodic comparison [W busy-serial-tips] | 1, 5 | M |
| **9** | **Warranty as data and the states around it.** Warranty type, start (invoice date), months and end on the unit; "with service centre"; a replacement linked to the old unit; DOA return; a warranty-due and expired report | electronics, mobile | Busy keeps three warranty types and shows warranty history [W busy-serial; Ws busy-search]; Zoho and Odoo have no warranty field of their own in what I read; the unit-profit and replacement-link parts are documented by none | 1 | M |
| **10** | **Labels from the grid, fixed layouts first.** After a purchase, one sticker per unit or per quantity received carrying the batch, expiry, MRP or IMEI barcode; from the batch list, the serial list and Stock Query; five or six tested layouts (8.4); a designer later | everyone with a printer | Busy: an add-on, paid design, 20-barcode limit below Standard [W busy-labels, busy-label-faq-8]; Zoho: one label per selected serial or batch, bulk copies per item, a community thread calls the generator limited [W zoho-labels; Ws zoho-community-labels]; Tally: an alias and partner add-ons, none confirmed to carry batch and expiry [Ws tally-labels-addons] | the barcode registry and encoders, a label-size print path (8.7), **the owner's printer and scanner** | M plus hardware |
| **11** | **A lot of units: batch and serial on one item** (appliances, batteries, tyres, a drug with a serialised pack) | appliances, auto parts, pharma serialisation later | Busy: "batch-wise serial number management is not supported" [W busy-faq 41]; Zoho: one method per item [W zoho-kb-both]; Odoo and SAP B1 choose lot **or** serial per product [W odoo-lots; Ws sap-b1] | 1, 2 | M |
| **12** | **Pharma extras**: expiry return list per supplier against the original bill, the Schedule H1 register, price-drop claims by batch | pharmacies, stockists | Parity with Busy's expiry returns, Logic's prompts and Marg's Near Expiry report [W busy-medical; W logic-pharma; W marg-near-expiry; Ws marg-listings]. The H1 register needs the prescriber and patient captured at the counter and has no batch column in the published format, while Rule 65(3) wants batch and expiry on the H and H1 sale record (3.2). Designed in the pharmacy-mode document, not here | 2, 4 | M each |
| **13** | **Unit and batch profit**: margin per phone, per batch | mobile, pharma | Busy: batch-wise sales and profit need the batch entered on the sale voucher [W busy-faq 73]; unit profit documented by none | 1, 2, R19 | S to M |
| **14** | **Used-phone margin scheme**: a per-unit purchase price, an IMEI-check stamp, a margin-scheme invoice | mobile | The rule is now cited from CBIC's leaflet (Rule 32(5): margin is sale less purchase, a negative margin ignored, no input credit; 3.1.4), which says nothing on per-item computation or invoice wording, so a chartered accountant must confirm **before anything is built** | 1 | L, accountant first |

### 10.3 What I would not build first [J]

- **An online IMEI blacklist check.** I found no public API; it is a link-out plus a recorded stamp (R22) until one exists [unverified].
- **Camera scanning.** The live site is HTTP and a camera needs a secure context; a keyboard-wedge scanner works today (9.1 row 8).
- **A free label designer.** Five tested layouts serve more shops than one designer a partner must configure.
- **Per-unit lines on the e-invoice.** One line per item with the serial array; expand only on demand and under the 1,000-line cap (9.1 row 11).
- **Serial or batch tracking inside manufacturing and job work.** Manufacturing hardcodes one godown and takes no batch, and the job-work chain does not exist [C `docs/research/ROADMAP-ADDENDUM.md:89`]; rows 1 and 2 first, with a plain statement of what manufacturing does not do.
- **Pack-scan as the way to fill a drug's batch and expiry.** CDSCO leaves the code's identifier to the maker and GS1 says many top-300 packs carry only a web address (3.2, 8.1); parse a GS1 element string when one appears, and never promise that a pack scan fills the row.

### 10.4 Where this beats Busy, by business

| Business | The headline | Because |
|---|---|---|
| Mobile and electronics | A phone cannot be sold twice, returned as a different unit, or lost without a trace; 40 phones are received by scanning in a minute or two | ranks 1, 5, 7, 8, 9 |
| Pharma, food | The right batch is proposed, an expired one cannot be billed, a recall is one screen | ranks 2, 3, 4, 7 (and 12) |
| FMCG distributors | MRP and price follow the batch; expiry returns are a list, not a hunt | ranks 2, 4, 13 |
| Garments | A barcode per size and colour (the CF-9 slice), kept apart from batches | rank 10 and the variant work |
| Jewellery | A tag is a unit with weights and a HUID | rank 1 with per-unit attributes |
| Tally switchers | Their batch-of-one phones become serials on import | rank 6 |

### 10.5 How the owner can tell it is better [J]

1. Receive 40 phones from a supplier bill by scanning, with two unscannable boxes: the bill books today with "2 pending identities".
2. Sell the same IMEI on two devices, one offline: the second is caught at sync and shown on the sync report.
3. Return a different phone of the same model: refused, and the screen says which unit was sold.
4. Bill a pharmacy counter with an item that has four batches: the soonest-expiring valid batch is pre-selected; an expired one cannot be billed without a reason.
5. Mark a batch recalled: it cannot be sold, and one screen lists who bought it.
6. Count 20 phones by scanning 18: the screen lists the 2 missing, and the books do not move until each has a reason.
7. Import a Tally company with IMEI-named batches: the shop is offered "convert 312 batches of one to serial units".
8. Print one sticker per unit after a purchase on the owner's own printer, and scan it back into the sale.

---

## 11. Where each product is strong or weak

Per the evidence in section 2 (and, for Marg, third-party listings only). **Strong** means the feature is documented and does what a trader needs; **Partial** means present with a gap I name; **Weak** means present in name or with a defect; **None** means I found no such feature in the pages I read (which is weaker than "cannot"); **Unknown** means I could not tell.

### 11.1 Batch capabilities

| Capability | Comsky today | TallyPrime | Busy | Marg (listings) | Zoho Inventory | Odoo | SAP B1 and NetSuite |
|---|---|---|---|---|---|---|---|
| Batch master (mfg, expiry) | **Weak** (name and expiry; mfg cannot be typed) | Strong (name, mfg, expiry, shelf life) | Strong | Strong | Strong (internal and manufacturer batch numbers) | Strong (four day-counts) | Strong |
| MRP and price per batch | None | **None found** | Strong (switchable; two FAQs disagree on the path) | Strong (listing) | Strong (four methods) | Not documented | Unknown |
| Batch picked at sale | Weak (`Alt+B`, typed, no list) | Partial (manual from the active list) | Partial (manual, or scan) | Partial (batch list in expiry order through Index Batch; the operator picks) | Partial (sorted list) | **Strong** (proposed) | Partial (an auto-allocation button) |
| First-expiry-first-out proposed | None | None (manual) | None documented | Partial (an expiry-ordered list; whether the first is pre-selected is unknown, and a blog calls it "First-Expiry First-Out") | Sort only | **Strong** (by removal date) | Unknown |
| Expired stock stopped | None | **Weak** (default allows; switch) | Weak ("allow with warning") | Unknown | Not documented | Partial (alert, no block) | NetSuite warns |
| Near-expiry report and alert | None | Partial (ageing by days to expiry; no pop-up) | Strong (Expiring Batches, warning days) | Strong (own help: a Near Expiry / Expired report by date range, supplier and store; alerts per the marketing page) | Partial (Batch Details by expiry) | Strong (alerts, filters) | NetSuite: warning days |
| Batch by godown and transfers | **Weak** (no godown dimension) | Partial (per godown; Stock Journal class mirrors) | Partial (needs batch on the transfer) | Unknown | Partial (transfer orders) | Strong | Strong |
| Count by batch (Physical Stock) | **Weak** (ignores batch) | Strong (batch chosen, dates filled) | Partial (stock journal, all items) | Unknown | Partial (adjustments pick batches) | Strong | Strong |
| Trace and recall | **None** (Stock Query only) | Partial (Batch Vouchers, one batch of one item) | Partial (batch reference details, batch-wise profit) | Unknown (its Batch Stock page names recall management as a use; no recall list seen) | Strong (batch page and transaction summary name customers) | Partial (Traceability report) | NetSuite: lot transactions |
| Held or recalled status | None | None | None | Unknown | None | None | None |
| Plan or edition gating | n/a | none | **Standard and above** | Edition tiers | selective plans | modules | licence |

### 11.2 Serial, IMEI and the rest

| Capability | Comsky today | TallyPrime | Busy | Marg (listings) | Zoho Inventory | Odoo | SAP B1 |
|---|---|---|---|---|---|---|---|
| Serial or IMEI engine | **None** | **None** (IMEI as a batch of one) | Strong (Standard and above) | Unknown (no listing mentions IMEI) | Strong (plan-gated) | Strong | Strong |
| Entry: scan, paste, range | None | scan of the item only | scan and a generator; no paste | Unknown | scan; limits per transaction | paste and generate | generate; Excel entry |
| IMEI format check, dual IMEI | None | None | **Not documented** | Unknown | Not documented | None | None |
| Unit history lookup | None | None | Strong (`Ctrl+Q` query) | Unknown | Strong (click a serial) | Partial (Traceability) | Strong |
| Warranty | None | None | Strong (purchase, sale, installation) | Strong (listing, appliance edition) | None in what I read | None native | Partial |
| Return of the same unit enforced | None | n/a | Not documented | Unknown | Partial (needs a processed sale) | Not documented | Not documented |
| Batch **and** serial on one item | n/a | n/a | **No** [W busy-faq 41] | Unknown | **No** | No (lot or serial) | No (serial or batch) |
| Cross-device duplicate detection | None | n/a | n/a | n/a | n/a | n/a | n/a |
| Stock-take by scanning | None | None | None (advice only) | Unknown | None | None | None |
| Barcode scan at billing | None (needs the alias) | Item only, via the alias | Strong (per POS series) | Strong (listing) | Strong | module | add-ons |
| Labels with batch or unit | **None** | **None** (add-ons) | Partial (add-on, paid design) | Strong (listing, counter edition) | Partial (item labels, limited) | Not researched | NetSuite: lot labels |

### 11.3 Reading the two tables [J]

- **Everyone is weak at the same things**: expiry blocking, a held or recalled status, a unit-format check, a scan-based count, a label born from the receipt, and the two together (batch and serial on one item). Those are the open ground, and they cost less than the structural work.
- **Busy is strong where the owner said and weaker than its fame in four places**: batch and serial cannot combine, expired stock is allowed with a warning, batch choice is manual, and batch carry-forward at year end depends on the BCN-wise feature being on and the stock not being negative [W busy-faq 41, 74, 73, 65].
- **Odoo is the best at *proposing*** (removal strategy, FEFO) and at *paste and generate*; it is not a trader's product and does not block expired lots.
- **Zoho is the best at price-per-batch and at history by click**, and the strictest about returns.
- **TallyPrime is adequate for a pharmacist's batches and absent for everything unit-level**, which is exactly the owner's observation; its Physical Stock does take a batch, which Comsky's does not.

---

## 12. What I could not verify (read this before relying on any claim)

1. **No live product.** I have no Busy, Marg, TallyPrime, Zoho, Odoo or SAP account. Every "what Busy does" is Busy's own help page or FAQ; every "what Tally lacks" is "not found in the pages I read", which is weaker than "Tally cannot". Busy's FAQ answers are numerous but uneven (several answers do not match their question, two disagree on bulk batch edits) and are used as evidence of what Busy *says*.
2. **No hardware.** No scanner, no label printer, no phone. Every statement about how a keyboard-wedge scanner behaves, about label sizes, driver aliasing or Bluetooth pairing is [J] or [unverified]; what two makers' command manuals say about printing a label (8.7) is a desk reading of the manuals, not a print test.
3. **Web access.** The first pass lost its web tools part-way through (a session limit on 2026-10-22). On 2026-10-23 most pages could be read, but not all. **Refused or missing:** GS1's own pages (403: its position on the drug-pack code is a search summary only), the live PIB page (403: read through a PDF print), the Sanchar Saathi copy of the advisory (404), Software Advice's reviews (403), and the Zebra and TSC pages
   that returned only navigation until read further. **Would not decode in the fetch tool and were read after local text extraction** (a text extractor only, nothing executed, each file in its own scratch directory): CDSCO's FAQ, the CBIC margin leaflet, the NHSRC register format, the PIB print and the TSC manual. **Marg's help pages** are 260,000 to 880,000 characters of navigation around a short article, and the tool reached the article on only some of them.
   **Still unread:** the Drugs Rules text itself (the H1 register format was read from NHSRC's document; the labelling, licence and retention clauses come from `tracking-verticals-india.md`), the Telecommunications Act, 2023 and the Cyber Security Rules (and the 2025 amendment), GS1's General Specifications, any scanner manual, Busy forum and YouTube threads, and the rest of 13.2.
4. **Busy's customers.** The independent evidence I reached: **one page of 20 user reviews read in full** (SoftwareSuggest, 4.6 of 5 from 124 reviews, September and October 2026): none mentions batch or expiry; one electronics seller praises serial and IMEI tracking and lists three gaps (2.1). Capterra and G2 search summaries add praise for batch-wise inventory from small teams and distributors
   and complaints about support, refunds, price rises, a dated interface and a learning curve; G2's support score for Busy against TallyPrime reads 7.6 against 8.5 [Ws busy-reviews]. A vendor-adjacent blog has a pharmacy anecdote and is not counted. I found **no forum thread and no video**, and no independent complaint about batch tracking itself. The premise that Busy's customers appreciate its batch management
   is consistent with Busy's pages and the Capterra summary and is otherwise **only weakly confirmed**.
5. **Marg.** First-party help pages were read only in part (2.3): the path and filters of the Near Expiry / Expired report, the path of the Batch Stock report, the existence of the Index Batch setting, and the titles of the Breakage / Expiry Receive articles. Whether Marg proposes the first batch or only lists it first, what it does about MRP per batch, and anything about IMEI or serials are **unknown**: no page or listing I read mentions IMEI or serial numbers.
6. **Statutory facts.** *Read in full or in the part named:* the FSSAI Labelling and Display compendium (Version-II of 29 December 2021, text extracted locally); the first 100,000 of 121,229 characters of a consolidated Legal Metrology (Packaged Commodities) Rules page (a third-party consolidation, not the gazette); **the DoT/PIB IMEI release of 17 November 2025**; **CDSCO's FAQ on G.S.R. 823(E)**;
   **the CBIC margin-scheme leaflet** (a leaflet, not the rule text); **the NHSRC Schedule H1 register format**; **the BIS jeweller page** (first 100,000 of 108,850 characters). *Taken from a sibling document that read them:* Drugs Rules Rule 65 and Rule 96 clauses, the e-invoice schema, the BIS blog on paired articles. *Not read:* the Telecommunications Act, 2023, the Telecom Cyber Security Rules, 2024 and 2025 amendment, the GST rule text itself,
   the G.S.R. 823(E) notification itself, the BIS Hallmarking Regulations and the July 2026 jeweller guidelines, GS1's specifications. The retailer's exposure for stocking a handset with a tampered IMEI is **my reading of the possession clause, not a stated fact**.
7. **Things the first draft got wrong or incomplete, corrected here:** Busy's mobile page describes **no** sample company (the first draft said a sample "mobile store" company exists); **batch and serial cannot be combined in Busy** (FAQ 41) and batch carry-forward at year end can fail (FAQ 65), neither of which the first draft had; Logic ERP's lot-expiry setting **stops the bill being saved** (the first draft called it a prompt); Odoo's FEFO ranks by **removal date** and the 16.0 docs warned that lots past expiry can still be picked; the FSSAI date-format proviso of 1 July 2022; the IMEI-as-batch Tally page and the Tracking Numbers page were read in full rather than from a search summary.
8. **Probe scope.** The engine measurements (the Physical Stock result of 7 for a count of 12, the 23.8 s for 125,000 one-unit lots) are from `batch-serial-code-audit.md`'s probes against synthetic books; I did not re-run them (item 12 lists the few figures I recomputed myself).
9. **CF-9 is a spec under a build.** Its addendum has had a third review (2026-10-22) and the worktree has only `countPart` at the type level on 2026-10-23; its statements here are about the `.wt-cf` file as of that day. Re-read AD.3.3 before building.
10. **Ranks and sizes in section 10 are judgement.** The owner's customer list is what would change them.
11. **Corrected in the second pass (2026-10-23)**, beyond the list in item 7: retailers "should be aware" under the DoT release, they are not told to validate new stock; the drug-pack code's encoding is the maker's choice and is often a web address; the Schedule H1 register format has no batch column; the HUID on a jeweller's invoice is voluntary; Busy's label limit is 20 below Standard; Tally's MRP page does not connect MRP and batch; a Physical Stock count by variant is a count group in the reviewed CF-9 spec.
12. **Recomputed by me on 2026-10-23, not copied:** the IMEI check digits and the three failing "whole number plus one" values of 3.1.1, the 09 and 90 blind spot of the Luhn check, and the EAN-13 check digits used in 8.2 and 3.6 (a throw-away Node snippet in the session scratchpad; no repository script was run). The Luhn page and the IMEI page were re-read and agree.

---

## 13. Sources

Tags: **[W]** read in full or in the part named; **[Ws]** a search summary only; **[R]** read in a sibling document of this workstream, not by me. Evidence is paraphrased throughout; nothing is quoted at more than a few words.

### 13.1 Read in the first pass and kept (URLs)

**Busy (vendor pages; marketing and help, treated as claims).**
- `busy-batch` https://busy.in/accounting-software/batch-expiry-inventory/ [W] (markdown copy `.md`)
- `busy-serial` https://busy.in/accounting-software/serial-number-warranty-tracking/ [W]
- `busy-mobile` https://busy.in/accounting-software/mobile-store/ [W]
- `busy-medical` https://busy.in/accounting-software/retail-shop/medical-store/ [W]
- `busy-serial-tips` https://busy.in/faqs/any-tips-or-best-practices-for-managing-serial-numbers-efficiently-answerid-59728/ [W]
- `busy-faq N` is the page `https://busy.in/faqs/features/inventory-parameter-batch-serial-number/N/` (each page holds ten FAQs): 13, 36 (first answer only), 41, 44, 46, 49, 50 (one answer), 53, 57, 65, 66, 67, 72, 73, 74, 75, 76 [W]
- `busy-alerts` https://busy.in/faqs/set-up-alerts-low-or-stock-expiry-unable-file-tax-return-busy-help-answerid-26474/ [W]
- `busy-labels` https://busy.in/faqs/barcode-qr-code-lable-printing/barcode (page 1 read [W]; pages 2 to 5 and `busy-labels-faq` as a search summary [Ws], the 5-to-20-character limit and the "Item+Batch No." type from the summary)
- `busy-reviews` https://www.capterra.in/software/71449/busy-accounting-software and https://www.g2.com/compare/busy-accounting-software-vs-tallyprime [Ws]
- `busy-migration` https://busy.in/migration and https://busy.in/faqs/data-conversion/tally-to-busy [Ws]
- `busy-search` search summaries of the pages above (first draft and this session) [Ws]

**TallyPrime and Tally.ERP 9.**
- `tally-batches` https://help.tallysolutions.com/tally-prime/inventory/batches-in-tallyprime/ [W] (an overview page, last updated 14 October 2026 by its own footer)
- `tally-expired` https://help.tallysolutions.com/docs/te9rel65/Creating_Masters/Inventory_Info/Batches_Expiry_Dates.htm [W]; the second switch ("Honor expiry dates usage for batches") [Ws] from https://help.tallysolutions.com/article/te9rel55/Creating_Masters/Inventory_Info/Manufacturing_date_Expiry_date_for_Batches.htm
- `tally-erp9-unsold` https://help.tallysolutions.com/article/Tally.ERP9/Advanced_Features/Advanced_Inventory_Features/Identifying_Unsold_Goods_Batch-wise_Details.htm [W] (the article body sits after about 200,000 characters of navigation)
- `tally-tracking` https://help.tallysolutions.com/article/te9rel65/Voucher_Entry/Advanced_Voucher_Entry/Using_Tracking_Numbers.htm [W] (same: body after 200,000 characters)
- `tally-guide-godown` https://tallysolutions.com/business-guides/tallyprime-inventory-management-godown-batch-tracking/ [W]
- `tally-guide-mfg` https://tallysolutions.com/business-guides/batch-manufacturing-and-expiry-tracking-for-small-manufacturers-tallyprime-vs-erp/ [W]
- `tally-physical` https://help.tallysolutions.com/?p=193740 [W]
- `tally-barcode-help` https://help.tallysolutions.com/?p=193523 [W]
- `tally-barcodes` https://tallysolutions.com/inventory/creating-using-barcodes-inventory-management-tallyprime/ [first draft W; a search summary this session]
- `tally-stockjournal` https://help.tallysolutions.com/article/te9rel55/Creating_Masters/Inventory_Info/Creating_Stock_Journal_Voucher_Class.htm [Ws]
- `tally-mrp` https://help.tallysolutions.com/tally-prime/sales/br-sales-at-mrp-tally/ and the VAT stock-item pages it sits beside [Ws]
- `tally-labels-addons` / `tally-addons` https://precisiontech.in/apps/tally/tally-tdl-addons/ , https://tallysolutions.com/partners/pearl-computer-lab/ , https://m.indiamart.com/proddetail/barcode-print-scan-tally-software-service-2851264707062.html , https://tallysolutions.com/customization [Ws]
- `tally-erp9-excel` https://tallysolutions.com/BSSDeploymentMgmtWS.php?ID=GETRELEASEUSERMANUAL&Releaseid=94666 [Ws, as cited in `tracking-verticals-india.md`]

**Zoho Inventory.**
- `zoho-serial` https://www.zoho.com/inventory/help/advanced-inventory-tracking/serial-number-tracking.html [W]
- `zoho-batch` https://www.zoho.com/inventory/help/settings/batch-tracking.html [W]
- `zoho-reports` https://zstatic.zohostatic.com/inventory/help/reports/advanced-inventory-reports.md [W]
- `zoho-returns` https://www.zoho.com/ae/inventory/kb/sales-returns/advanced-return-tracking.html [W]
- `zoho-kb-both` https://www.zoho.com/us/inventory/kb/advanced-inventory-tracking/batch-and-serial-number-tracking-for-items.md [W]; `zoho-index` https://www.zoho.com/en-ng/inventory/help/items/items-overview.html [Ws]
- `zoho-first-expiring` https://www.zoho.com/inventory/kb/advanced-inventory-tracking/first-expiring-batch.html [Ws]
- `zoho-batch-price` https://www.zoho.com/inventory/help/advanced-inventory-tracking/batch-price-management/batch-price-management.html [Ws]
- `zoho-labels` https://www.zoho.com/en-ng/inventory/help/items/qrcode-generation.html [Ws]; `zoho-community-labels` https://help.zoho.com/portal/en/community/topic/product-label-generator-is-must-have-more-fileds [Ws]

**Odoo (18.0 documentation).**
- `odoo-lots` https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/product_management/product_tracking/lots.html [W]
- `odoo-serial` https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/product_management/product_tracking/serial_numbers.html [W]
- `odoo-expiry` https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/product_management/product_tracking/expiration_dates.html [W]
- `odoo-removal` https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/shipping_receiving/removal_strategies.html [W]; `odoo-fefo` .../removal_strategies/fefo.html [W]; the 16.0 caution about expired lots [Ws] https://www.odoo.com/documentation/16.0/applications/inventory_and_mrp/inventory/routes/strategies/removal.html
- `odoo-trace` the Traceability section of `odoo-lots` [W]; add-on vendors [Ws] https://ecosire.com/apps/odoo/lot-serial-expiry-traceability

**SAP Business One and NetSuite (reference).**
- `sap-learning` https://learning.sap.com/courses/managing-logistics-in-sap-business-one/managing-serial-numbers-and-batches [W]
- `sap-b1` help.sap.com pages for 9.0 and 9.1, for example https://help.sap.com/doc/saphelp_sbo91/9.1/en-US/45/2366439e152b31e10000000a1553f7/content.htm and a community thread https://answers.sap.com/questions/8698155/manage-item-by-batches.html [Ws]; `sap-b1-valuation` https://vision33.com/hubfs/ARedesign%202018/Tips%20and%20Tricks/Tips%20and%20Tricks%20PDF/every-batch-serial-number-has-its-cost.pdf [Ws]
- `netsuite-lot` https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/article_163360798322.html [W]; `netsuite` https://www.sikich.com/insight/how-to-do-netsuite-item-tracking-with-serial-and-lot-numbers/ and https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/section_N3728128.html [Ws]

**Logic ERP, GoFrugal, Marg.**
- `logic-imei` https://kb.logicerp.com/logic-erp-in-action-a-feature-overview/distribution/billing/serial-number-imei-wise-billing-for-electronic-goods [W]
- `logic-expiry` https://kb.logicerp.com/logic-erp-learning-hub-tutorials-and-best-practices/user-access/logic-global-admin/sale/sale-settings-3/prompt-in-billing-if-lot-expiry-is-below [W]
- `logic-pricedrop` https://kb.logicerp.com/logic-erp-in-action-a-feature-overview/distribution/billing/price-drop-management (the page body is empty: title only) [unverified]; `logic-elec` https://logicerp.com/Distribution/Electrical-And-Electronics-Distribution-Software [Ws]
- `gofrugal-batch` https://community.gofrugal.com/portal/en/kb/articles/sales-bill-configurations-6-8-2020 [W]
- `marg-listings` https://www.softwaresuggest.com/marg-pharmaceutical , https://accountune.com/marg-erp-alternative , https://softwareconnect.com/pharmaceutical-distribution/marg-pharma-software/ , https://saasrat.com/products/marg-pharmaceutical-erp , https://www.industrybuying.com/brands/marg-16172 [Ws; third-party and competitor pages]

**Statutory.**
- `fssai-compendium` Food Safety and Standards (Labelling and Display) Regulations, 2020, FSSAI compendium Version-II (29 December 2021): https://fssai.gov.in/upload/uploadfiles/files/Compendium_Labelling_Display_04_01_2022.pdf [W] (text extracted with pdfminer in the session scratchpad; regulations 2(1)(l) and (u), 4(2), 5(8) to 5(10), 8(1), 8(3), 8(5), 8(6))
- `legal-metrology` Legal Metrology (Packaged Commodities) Rules, 2011, consolidated page: https://www.legitquest.com/act/legal-metrology-packaged-commodities-rules-2011/97B4 [W] (first 100,000 of 121,229 characters; rules 6(1)(d), (da), (e), 6(2), 6(3), 18)

**Repository (all [C] or [R]).** `CLAUDE.md` (Phases 13, 20 to 28 gotchas); `src/types.ts`, `src/core/inventory.ts`, `src/core/einvoice.ts`, `src/ui/VoucherScreen.tsx` (`BatchPrompt`, the Alt+B walk), `src/ui/inventory/BatchSummary.tsx`, `src/ui/admin/BulkEntry.tsx`; `docs/parity-data.json`, `docs/FEATURE-PARITY.md`; `docs/research/batch-serial-code-audit.md`, `docs/research/tracking-verticals-india.md`, `docs/research/tally-udf-and-competitors.md`, `docs/research/mobile-personas-and-jobs.md`, `docs/research/ROADMAP-ADDENDUM.md`; and, read-only, `.wt-cf/docs/design/custom-fields-spec.md` (CF24 to CF26, AD.3) and `.wt-cf/docs/design/back-nav-spec.md` (B5, B12).

**Computed by me, not copied:** the IMEI Luhn digits in 3.1.1 (recomputed in both passes with a throw-away Node snippet: body 49015420323751 gives check digit 8; the next three bodies give 490154203237526, 490154203237534 and 490154203237542; adding 1 to the whole number fails the check three times; two numbers that differ by a 09/90 swap have the same check digit).

### 13.1a Added in the second pass (2026-10-23)

Where a PDF would not decode in the fetch tool, the file was saved by the tool, copied into its own empty scratch directory and read with a text extractor only (nothing executed). [W] = read in full or the part named; [Ws] = search summary.

**Statutory and regulatory**
- `pib-dot` Press Information Bureau, Ministry of Communications, Release ID 2190763, 17 November 2025: https://www.pib.gov.in/PressReleasePage.aspx?PRID=2190763&reg=3&lang=1 [W] (the live page answered 403; read as a 4-page PDF print of it)
- `cdsco-faq` CDSCO, "FAQs on implementation of GSR 823(E) dated 17th November, 2022 with respect to bar code or QR code on top 300 brands" (PDF dated 25 July 2023): https://cdsco.gov.in/opencms/resources/UploadCDSCOWeb/2018/UploadPublic_NoticesFiles/Final%20FAQs%20on%20QR%20code%2021.07.2023.pdf [W]
- `gs1-india-position` GS1 Healthcare, India position paper, October 2023: https://www.gs1.org/docs/healthcare/position-papers/paper-gs1-hc-india-oct2023.pdf [Ws] (403; the summary came from a search); related, not read: https://gs1.org/sites/gs1/files/2024-10/gs1seg241001-01-global_harmonisation_domestic_indian_market_a4_01.pdf
- `onmanorama-2026` the June 2026 report of an extension of the drug-code mandate: https://www.onmanorama.com/upsc/current-affairs/2026/06/26/qr-codes-now-mandatory-for-vaccines-cancer-drugs.amp.html [Ws]
- `cbic-margin` CBIC / DGTS, "Margin Scheme in GST" (updated November 2022), https://www.gstcouncil.gov.in/sites/default/files/e-version-gst-flyers/margin-scheme-under-gst.pdf [W] (a leaflet quoting Rule 32(5); the rule text itself was not read)
- `bis-jewellers` BIS, jewellers page: https://www.bis.gov.in/hallmarking-jewellers/?lang=en [W] (first 100,000 of 108,850 characters); `bis-consumer-faq` https://www.bis.gov.in/hallmarking-overview/hallmarking-faqs/mandatory/?lang=en [W] (six questions; says nothing on invoices)
- `nhsrc-h1` NHSRC, "Attachment A, Schedule H1 register and NDPS Forms" (a six-page document created 31 May 2021): https://www.nhsrcindia.org/sites/default/files/2021-07/Attachment%20A%2C%20Schedule%20H1%20register%20and%20NDPS%20Forms.pdf [W] (G.S.R. 588(E) of 30 August 2013)
- `drug-rules-secondary` search summaries of https://accountune.com/pharmacy-billing-batch-expiry-guide , https://en.wikipedia.org/wiki/Drugs_and_Cosmetics_Rules,_1945 and a Goa FDA workshop document, on Rule 65(3) and Schedule H1 [Ws]; the CDSCO Drugs Rules PDF (https://cdsco.gov.in/opencms/resources/UploadCDSCOWeb/2022/drug_rules/Drugs%20Rules,%201945.pdf) answered 403 to a scripted request and was not retried by other means
- `gs1-ai` identifier facts (AI 01, 10, 17, 21; the separator) from search summaries of https://www.tec-it.com/en/support/knowbase/gs1-application-identifier/Default.aspx , https://gs1ca.org/documents/GS1-Canada-GS1-DataMatrix-Fact-Sheet-en.pdf and https://kb.cleverence.com/3.4/how-to-correctly-form-gs1-datamatrix [Ws]; gs1.org's own page answered 403

**Busy, Marg**
- `busy-reviews-ss` https://www.softwaresuggest.com/busy-accounting/reviews [W] (20 of 124 reviews shown; a Software Advice reviews page answered 403)
- `busy-label-faq-5`, `busy-label-faq-8` https://busy.in/faqs/barcode-qr-code-lable-printing/barcode/5/ and .../8/ [W]; best-practices and "change the barcode number or quantity" FAQs seen in search only [Ws]; the "enable batch or serial tracking" FAQ https://busy.in/faqs/how-do-i-enable-batch-or-serial-number-tracking-for-items-answerid-55954/ [W] (batch steps only; names no edition)
- `marg-index-batch` https://care.margcompusoft.com/margerp/batch/4451/1/How-to-Index-Batch-on [W, path only]; `marg-near-expiry` https://care.margcompusoft.com/margerp/inventory-reports/30305/1/null [W]; `marg-batch-stock` https://care.margcompusoft.com/margerp/batch-stock/3911/1/adler32 [W, partial]; `marg-brk-exp` https://care.margcompusoft.com/margerp/breakage-expiry-receive [W, titles only]; Marg's pharmacy page https://margcompusoft.com/pharmacy-software.html and blog https://margcompusoft.com/m/top-10-features-of-an-efficient-drug-inventory-management-software-for-your-business/ [Ws]

**Tally, Zoho**
- `tally-mrp-page` https://help.tallysolutions.com/tally-prime/sales/br-sales-at-mrp-tally/ [W]; `tally-mrp-state` https://help.tallysolutions.com/docs/te9rel63/Tax_India/VAT/Defining_and_using_State_wise_MRP.htm [Ws]
- `tally-batch-summary` https://help.tallysolutions.com/?p=193789 [W]; `tally-stockjournal-class` https://help.tallysolutions.com/docs/te9rel60/Creating_Masters/Inventory_Info/Creating_Stock_Journal_Voucher_Class.htm [W]; `tally-tdl-search` https://tallysolutions.com/business-guides/is-it-worth-using-tallyprime-for-micro-businesses/ and a vendor blog https://hisabplus.com/Blog/imei-history-tracking [Ws]
- `zoho-composite-serial` https://www.zoho.com/us/inventory/kb/composite-items/comp-items-serial-number.html [W]; `zoho-batch-variants-composites` https://www.zoho.com/us/inventory/kb/advanced-inventory-tracking/batch-tracking-variants-and-composites.html [W]; `zoho-enable-serial` https://www.zoho.com/us/inventory/kb/advanced-inventory-tracking/enabling-serial-number-tracking.html [W]; `zoho-serial-help` https://www.zoho.com/inventory/help/advanced-inventory-tracking/serial-number-tracking.html [W] (says nothing on mandatory serials, entry methods or composites); `zoho-assembly-api`, `zoho-disable`, `zoho-adv` and `zoho-books` are search summaries [Ws] of https://www.zoho.com/inventory/api/v1/compositeitems/ , https://www.zoho.com/us/inventory/kb/advanced-inventory-tracking/disable-batch-tracking-for-an-item.html and https://www.zoho.com/us/inventory/help/advanced-inventory-tracking/serial-number-tracking.html
- `zoho-batch-price` https://www.zoho.com/inventory/help/advanced-inventory-tracking/batch-price-management/batch-price-management.html [W]; `zoho-first-expiring` https://www.zoho.com/inventory/kb/advanced-inventory-tracking/first-expiring-batch.html [W]; `zoho-serial-limit` https://zstatic.zohostatic.com/inventory/kb/advanced-inventory-tracking/serial-number-limit.md [W]; `zoho-barcode-serial` https://zstatic.zohostatic.com/za/inventory/kb/advanced-inventory-tracking/barcode-scanning-for-serial-numbers.md [W]; `zoho-labels` https://www.zoho.com/us/inventory/help/items/qrcode-generation.html [W]

**Identifiers and printers**
- `wiki-imei` https://en.wikipedia.org/wiki/International_Mobile_Equipment_Identity [W]; `wiki-luhn` https://en.wikipedia.org/wiki/Luhn_algorithm [W]
- `zebra-bc` https://docs.zebra.com/us/en/printers/software/zpl-pg/c-zpl-zpl-commands/r-zpl-bc.html [W] (the page body began after 100,000 characters of navigation); `labelary-zpl` https://labelary.com/zpl.html [W]
- `tsc-tspl` the TSC "TSPL/TSPL2 Programming Language" manual, a copy hosted at https://cleversoftsolutions.com/descargas/utilidades/Impresoras/Tsc/TSPL_TSPL2_Programming2.pdf [W] (189 pages, copyright 2009, update history to 2011; the first 80 pages extracted; a copy hosted by a third party, and a second copy at ers-online.co.uk answered 403)
- `printer-market` IndiaMART listings, e.g. https://m.indiamart.com/impcat/tsc-barcode-printers.html [Ws] (seller asking prices; not a specification)

### 13.2 Read by the first draft and not (or only partly) re-read since (URLs were not preserved)

These claims are carried over as the first draft stated them. **Recovery pending** means a later pass should re-find the page and re-read it. The rows the second pass re-read say so; the rest are unchanged.

| Key | What it supports | Status |
|---|---|---|
| `busy-faq 5, 14, 18, 21, 23` | the Busy FAQ pages numbered 5, 14, 18, 21 and 23 in the first draft's reading: the barcode and label limits, "parameter entries land on account", the serial-number structure | partly resolved: the barcode-category pages 5 and 8 were re-read on 2026-10-23 (`busy-label-faq-5`, `busy-label-faq-8`) and support the label limits; 14, 18, 21 and 23 remain recovery pending, category not recorded |
| `tally-batch-summary` | Batch Summary: Ctrl+B for expired-only and for date ranges, F12 days to expiry, zero rows | **re-read 2026-10-23 [W]** (13.1a): confirmed; the page does not say whether it filters by godown |
| `tally-batches-setup`, `tally-release`, `tally-udf-batch` | shelf-life setup; TallyPrime releases 5 and 6 list nothing on barcodes; a batch UDF holds a string only (also in `docs/research/tally-udf-and-competitors.md:110`) | recovery pending |
| `tallyatcloud-barcodes` | a third-party article that contradicts itself on label printing | recovery pending |
| `zoho-adv`, `zoho-adjust`, `zoho-serial-limit`, `zoho-barcode-serial`, `zoho-composite` | opening stock of 200 serials; adjustments; 1,000 serials per transaction; scanning a serial; composites | `zoho-serial-limit` (1,000 per transaction, and nothing else) and `zoho-barcode-serial` re-read 2026-10-23 [W]; `zoho-adv` (the 200), `zoho-adjust` and `zoho-composite` remain recovery pending |
| `logic-pharma`, `logic-expiry-title`, `gofrugal-expiry` | Logic's pharma page (batch by MRP, schemes by batch, expiry returns, a Schedule H block); a Logic expiry-report page; GoFrugal's Expiry Returns screen | recovery pending |
| `vyapar-inventory` | batch and expiry alerts; barcode generation on the Gold plan | recovery pending |
| `mybillbook-imei`, `hisabplus-imei`, `hisabplus-importance`, `accountune-electronics` | vendor blogs on IMEI workflows, duplicate warnings, returns by IMEI | recovery pending (marketing claims) |
| `wiki-imei`, `wiki-luhn`, `wiki-ean13`, `wiki-gtin` | https://en.wikipedia.org/wiki/International_Mobile_Equipment_Identity , https://en.wikipedia.org/wiki/Luhn_algorithm , https://en.wikipedia.org/wiki/International_Article_Number , https://en.wikipedia.org/wiki/Global_Trade_Item_Number | `wiki-imei` and `wiki-luhn` re-read 2026-10-23 [W] and agree with 3.1.1; `wiki-ean13` and `wiki-gtin` remain recovery pending |
| `fonearena-dot`, `pib-dot`, `sanchar-saathi` | the Department of Telecommunications IMEI advisory and the Telecom Cyber Security Rules, 2024 (obligations of makers, importers, retailers, used-phone dealers); Sanchar Saathi and CEIR | `pib-dot` **read in full 2026-10-23** through a PDF print (13.1a) and now the basis of 3.1.2; `fonearena-dot` is superseded by it; `sanchar-saathi` remains a search summary [Ws] |
| `bis-faq`, `bis-huid-news` | hallmark marks and the six-character HUID; a single-source report on a proposed "sold" mark | `bis-faq` is superseded for the invoice and record rules by `bis-jewellers` (13.1a); the BIS blog `https://services.bis.gov.in/php/BIS_2.0/BISBlog/?p=560` was read by `tracking-verticals-india.md`; `bis-huid-news` remains a search summary [Ws] |
| `nhsrc-h1`, `drug-records` | Schedule H1 register contents and retention; a 2026 High Court ruling on an H1 register | `nhsrc-h1` **read in full 2026-10-23** (13.1a) and it corrects the first draft (no batch column); `drug-records` remains a search summary [Ws]; the labelling and licence clauses come from `tracking-verticals-india.md` section 3 |
| `cdsco-qr` | the QR on the top-300 drug brands from 1 August 2023 and its 2026 extension | the FAQ was read in full 2026-10-23 (`cdsco-faq`, 13.1a); the 2026 extension remains a single-source search summary [Ws]; earlier URLs are in `tracking-verticals-india.md` (Business Today; RAPS) |
| `GS1-AI`, `GS1-RCN`, `GS1IN`, `INS`, `Rule 65 row` | identifiers read in `tracking-verticals-india.md` | see that document's section 9 |
