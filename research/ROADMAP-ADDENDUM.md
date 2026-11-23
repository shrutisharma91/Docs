# Roadmap addendum: what Phase 24 left open

> Written 2026-10-19 at the end of Phase 24 (two parity waves, see `docs/FEATURE-PARITY.md`). One line for every item
> the waves left `partial` or not done, with the reason and the next step. It is the honest remainder of the
> numbers in `docs/parity-data.json` (258 covered, 44 partial, 79 missing of 381).
> Reasons come from the builders' own reports and from the code; nothing here has been checked against a live
> TallyPrime, the GST portal, the IRP or a bank unless a line says so. Row ids are the ids in `parity-data.json`;
> items marked (beyond) live under `beyond`, not in the parity percentage; items marked (no row) were scoped by a wave
> but have no audit row of their own.

> **Update 2026-10-22 (Phase 29, parity wave 3; built, NOT committed, NOT deployed).** This file is the remainder as at Phase 24 and was not rewritten. The board is now 250 covered, 58 partial, 73 missing
> (73.2%). Phase 29 touched these rows, so their lines below are out of date and their current remainder is in `docs/HANDOFF.md` §2f and in each row's note in `parity-data.json`: `cancelled-register`, `deleted-tracking`,
> `credit-limit`, `credit-limit-override`, `interest-simple`, `interest-bills`, `interest-report`, `price-levels`, `single-entry-mode`, `numbering-override-multiuser`, `numbering-advanced-series`, `eway-bill`, `eway-register`,
> `report-email`, `report-export`, `export-pdf`, `import-excel`, `vouchertype-flags`. Two lines here changed meaning: a native .xlsx reader now exists (`src/core/xlsxRead.ts`, used by Import Data) but is not wired into
> Import Bank Statement (`brs-auto-import` stays partial), and the Phase 24 note that nothing calls `/sync/next-number` is superseded by the Phase 29 client (backend half not deployed).

## 1. Partial or not done, by area

### GST returns and rates

- **gst-json-export** (partial, P0). The JSON has never been imported into the GST offline tool. Fields flagged `SCHEMA?` in
  `core/gstr1.ts` / `core/gstr3b.ts` rest on sample files and vendor notes: the `hsn_b2b` / `hsn_b2c` key names and entry
  fields, the WOPAY rate, CDNR's original-invoice reference, the 16-character document-number and 6-character port-code
  rules, and the GSTR-3B `itc_rev` / `itc_inelg` `ty` codes. Next: take one real GSTR-1 and GSTR-3B from the owner's CA,
  import them into the offline utility, fix each flagged field, then flip the row.
- **gst-rate-hierarchy** (partial). No GST Classification tier, no accounting-group tier, no "apply this group rate to all
  its items" action. The default source order (stock item first) is Tally's second documented option, kept so existing item
  invoices do not change; Tally's documented default is ledger first. Next: owner decides the default (ledger first would
  re-tax new item invoices wherever a Sales ledger carries a rate); any bulk reform tool must treat legacy lines with no
  snapshot with care (a naive "item states nothing from date X" entry reads 0% for them).
- **gst-classifications** (missing). Needs a new Dexie table (v9) and its own screens; the resolver in `core/gstRates.ts` is
  layered so a classification tier slots between the item and its groups. Next: table, master screen, resolver tier.
- **rcm** (partial). No self-invoice print (`invoiceModel.ts` shows no RCM tax figures); liability falls on the voucher or
  Journal date, not the statutory time of supply; with Manual posting a purchase flipped to Optional on the voucher screen
  leaves its Journal live (the return skips it and warns). Next: self-invoice print, a time-of-supply date, and make
  `rcmAfterSave` act for Manual companies too.
- **itc-reversal** (partial). The Rule 42/43 common-credit ratio is not computed (amounts are typed into a GST-flagged
  Journal), and s.50 interest, Rule 37A, 4(D)(2) and ISD are not recorded. Next: a common-credit ratio helper from exempt
  versus total turnover, then interest; ISD only if a customer needs it.
- **gst-cess** and **ledger-gst-details** are covered, with two stated limits worth a line: per-unit (specific) cess is not
  modelled, and whether compensation cess still applies after the September 2025 Council decisions (pan masala and tobacco
  versus everything else) is unconfirmed. Next: check the notifications before relying on cess figures for 2026-27.
- **einvoice-irn** (partial) and **einvoice-irp-api** (no row, not done). No call to an IRP or GSP, no credential vault, no
  queue or relay; cancel IRN, e-way by IRN and GSTIN lookup are not built; signed-QR signatures are not verified (no NIC public
  key held) so a hash-consistent forged portal row would pass; the results-file layout is built to documented fields, not to
  a real file. Next: Phase 2 of `docs/research/einvoice-gsp.md` (TLS, credential vault, GSP contract), a server-side
  denylist for client-writable e-invoice rows, NIC signature check, one sanitised real results file to fix the parser.
- **einvoice-bulk** (partial). Portal bulk-upload wrapper and the 2 MB split come from third-party guides; no bulk generation
  through an API, no status polling. Next: with the IRP work above; verify the wrapper from one real upload.
- **Debit Notes in the e-invoice register** (no row). The builder supports DBN but the register lists Sales and Credit Notes
  only, because a Debit Note voucher is often a purchase return. Next: confirm with the owner's CA, then list them.
- **GSTR-9 tables** (row `gstr9`, covered for Tables 4-6 and 9). Tables 8 and 10-14 (amendments, ISD, transitional credit)
  are absent and Table 7 reversals are one line. Next: build them only when a customer files GSTR-9 from the app.

### TDS / TCS

- **tds-interest-latefee** (partial). Not modelled: 194S for a specified person, a Government deductor, CBDT-extended due
  dates, s.234E on 26QB/QC/QD statements, and the Income-tax Act 2025 renumbering (section labels stay the 1961 ones).
  Estimates only, nothing is posted. Next: a due-date override table; a section map once the 2025 numbering is confirmed.
- **form-26q** (covered, but) no FVU file and no 26Q/27EQ file-format section codes (unverified; sections renumber from
  1 Apr 2026). **etds-fvu** (missing, P1, scout "skip"), **form-27q**, **form-27eq**, **form-26qb**: not started. Next: only with
  the NSDL FVU specification in hand.
- **tds-threshold-cumulative** (covered, limits). Payments made before the books began are not in the register; vouchers
  saved before this release on an "Any (choose at entry)" ledger that deducted nothing stay uncounted (a floor); a hand-posted
  deduction with no expense line is counted as the payment it implies and flagged estimated. Next: an opening
  year-to-date entry per deductee.
- **tds-reversal** (missing, scout "skip"). Alteration covers it by hand; a refund or return correction is not a book entry.

### Reports, inventory, manufacturing

- **report-f12-config** (partial, P0). GST returns, TDS, payroll statements and about 35 other report files are not on F12.
  Next: one `useF12(title, spec)` call per report file, and list each in `core/savedViews.ts` STATIC (the dashboard-custom
  suite fails on an unlisted F12 report).
- **report-manual-columns** (partial). No added columns on Ratio Analysis or Godown Summary; Cash/Funds Flow (month-per-row
  tables) do not take them. Next: add the column hook to the two reports; decide whether the flow reports ever should.
- **stock-summary** (covered, limits). The movement layout (Opening/Inwards/Outwards/Consumption, or Group by Godown) is one
  period, so added as-on columns are hidden there; "Show Gross Value for Outwards" is not built; a godown's opening and
  closing values are derived from the item's rate, not booked. Next: a per-period movement comparison.
- **cost-analysis** (covered, limits). Cost Centre Summary (`reports/financial.tsx`) and Cost Centre P&L (`costpl.tsx`) still
  show own-centre figures flat, and the cost-centre master's Book drill is own-only. Next: apply the same rollup there.
- **inventory-f12** and **f12-voucher-config** (partial). Actual-versus-billed quantity, MRP and rejection-note columns are not
  built (they change stock and tax arithmetic); the line Description is not written to the Tally XML export or the open API;
  only item invoices get the column (not Stock Journal or Physical Stock). Skip-field and warning switches are not built.
  Next: actual/billed quantity first (inventory engine), then Description in `tallyExport.ts` and REST.
- **item-description** (partial). Same limits as above (XML export, API, item invoices only).
- **valuation-standard** (partial). One undated standard rate, so editing it restates history; Tally keeps date-wise lists.
  Next: a dated standard-cost list on the item, the way GST history is dated.
- **valuation-other** (partial). Tally's four item behaviour options (ignore physical-count difference, ignore negative
  balances, treat all sales as new manufacture, treat all purchases as consumed) are researched in `core/valuation.ts` but
  not built (a new item-master field would break the frozen Enter walk); value-only debit/credit notes and rejection in/out
  are not modelled. Next: an Alt+key option grid on the item, as the GST history uses.
- **valuation-wac / valuation-lifo / valuation-fifo** (covered, but unverified). Nothing has been compared with a live
  TallyPrime. Section 4 lists the five cases the owner's consultants should run. Plain FIFO and LIFO that restart each year,
  and per-godown or per-batch lots, are not modelled; the financial year is assumed to start in April by every caller.
- **mfg-stock-posting** (partial, P0, not touched). Manufacture hardcodes Main Location, captures no batch/expiry for batch
  items, gives no shortage warning and values FIFO items at average cost. Next: extract `costManufacture()` to
  `core/manufacturing.ts` (scout approach), then godown and batch fields.
- **Job-work chain, multi-BOM, co-products, mfg cost allocation** (rows `bom-multiple`, `bom-nested`, `bom-wastage`,
  `coproducts-byproducts`, `mfg-cost-allocation`, `material-out`, `material-in`, `jobwork-*`, all missing). Untouched by
  Phase 24. Next: see section 3.

### Banking, security, platform

- **brs-auto-import** (partial). `.xlsx`/`.xls`/`.pdf` are not opened (no parser in node_modules; the screen explains Save As
  CSV or paste); bank fixtures are reconstructions of published layouts, not real exports; one-to-many matches (a deposit slip
  credited as one line) are not attempted; flag-batch undo lasts only while the screen is open. Next: a small `.xlsx` reader
  (zip + DecompressionStream + sheet scanner), one real statement per bank, one-to-many matching; live feeds follow
  `docs/research/live-bank-feeds.md` (the matcher already takes a neutral `StatementLine`).
- **brs-format-config** (covered, limit). The header synonym table is a best guess at HDFC/SBI/ICICI/Axis/Kotak/Yes/PNB
  layouts. Next: harden from real statements.
- **company-vault** (partial, P0). Nothing is encrypted: the books are readable in IndexedDB and backups are plain JSON minus
  the secrets; a forgotten company passcode cannot be reset. WebCrypto AES is missing on the http-only VM. Next: TLS first
  (HANDOFF Phase B), then a password-derived AES-GCM backup envelope with a key derived at unlock so automatic backups can
  encrypt; at-rest encryption of IndexedDB is a separate decision.
- **password-policy** (partial). No expiry and no reuse history (the brief said skip). The backend changes are written but not
  deployed. Next: deploy; expiry only if a customer asks.
- **access-restriction** and the operator model (covered, limits). A client-side control only: the sync server does not
  enforce roles; payroll, attendance, e-invoice and Alt+C ledger creation have no early access check (a refused save shows the
  reason and writes nothing); there is no per-report or Print right; users in non-admin roles may have no PIN, so their Edit
  Log stamps prove nothing. Next: enforce roles in `backend/` (the sync server does not).
- **export-xml** (partial). Never imported into a live TallyPrime; tags flagged unverified in `tallyExport.ts` (HSN inside
  `GSTDETAILS.LIST` versus `HSNDETAILS.LIST`, the P&L A/c with an empty `PARENT`, older ledger GST tags versus
  `LEDGSTREGDETAILS`, the stock-journal view name); real SEZ / Deemed Export / Overseas registration-type strings are unknown;
  no GUID is written, so importing the same file twice duplicates vouchers; no CSV/Excel layout (TallyPrime imports Excel
  through a mapping template whose field names live in the sample files' "Read Me" sheet). Next: import both files into a
  trial TallyPrime company, fix the tags, add the `REG_TYPE` values, then decide on GUIDs and the Excel template.
- **import-xml** (partial, P0). Tested on hand-built fixtures and our own export, never on a file from a live TallyPrime;
  SEZ / Deemed Export are not restored (`gstPartyTypeFromTally` needs the real values); not imported (counted and reported):
  orders, rejection and job-work types, payroll and attendance vouchers, multi-currency, a stock journal's additional-cost
  lines; parsing is on the main thread and undo is one voucher at a time. Next: one real export, then those voucher types.
- **multiuser-lan** (partial). Changes arrive in about a second, but there is no presence list and no on-prem LAN server.
  Next: presence over the live stream (the shared platform work is sequenced in `docs/research/README.md`); a LAN server only on customer demand.
- **record-locking** (covered, limits). Optimistic detection plus a report, not a hard per-voucher lock; duplicate-number
  detection is client-side and only for vouchers that arrived by sync (at most 5,000 per sync); older clients ignore the
  server's adjusted copy and keep no device mismatch queue; a restore's deleted records have no one-click recovery; the
  mismatch report needs backend migration 007. Next: server-side number check once numbering windows are server-known.
- **data-split** (covered, limits shown on its screen). FIFO/LIFO layers and the last purchase rate are carried at a blended
  rate; cost-centre openings, deductee-wise TDS/TCS detail, payroll year-to-date, leave balances and GST/TDS turnover
  aggregates do not carry (a mid-year split warns); new companies carry no passcode or sign-in. Next: carry those when a
  customer splits mid-year.
- **licensing** (partial). Subscription status gates sync, but payments are MOCK and plan `max_companies` / `max_users` are
  display-only. Next: enforce the caps in `backend/` and take live Razorpay keys (HANDOFF Phase B).
- **scheduled-backup** (covered, limits). The Electron backup bridge is checked only against a mocked electron and needs an
  installer rebuild; other browsers download on B and never back up automatically; the schedule is per device. Next:
  `npm run electron:build` and one real-window run.

### Beyond Tally (not in the percentage)

- **tally-compare-trial-balance** (partial) and **tally-compare-daybook-report** (not done). The Trial Balance layout comes
  from Tally's published sample pages only (closing columns, as-on date assumed to be the window end); a Day Book exported as
  a report is refused with a pointer to the voucher export because its tag names could not be confirmed. Next: one real
  Trial Balance and one real Day Book export from a reseller client.
- **report-views-filters** and **report-views-favourite-slots** (partial). The type-to-filter text and one-off toggles (F5
  detailed, All Items) are not part of a saved view, and the nine Ctrl+Alt+1..9 favourite slots cannot hold a view. Views are
  kept per device, not synced and not per operator. Next: a shared hook out of `ReportShell` for the filter; extend
  `favouritesCore.ts` to take a view id; a synced prefs mechanism that a non-admin can write does not exist.
- **Dashboard** (covered). The GST payable tile runs a full GSTR-3B build (off by default); the TDS tile names no due date
  because the deposit dates under the Income-tax Act 2025 were not researched. Next: add the date once confirmed.

## 2. Decisions waiting on the owner

1. **GST source order.** Keep stock-item-first as the default (current) or switch to Tally's documented ledger-first. Both
   exist (`Company.gstSource`, set on GST Rate Setup); ledger-first re-taxes new item invoices wherever a Sales ledger carries
   a rate. Sources: help.tallysolutions.com/set-up-gst-rate-hsn-sac-details/.
2. **Restating existing stock.** Every item saved before Phase 24 keeps Moving Avg. until the owner alters it or presses S on
   Stock Valuation; only new items, new sample companies and imports default to Avg. Cost. The owner's existing cloud sample
   will not match Tally until that is done.
3. **Migration 007 window.** `backend/sql/007_sync_conflicts.sql` builds `idx_change_doc` on `change_log` (a brief write lock
   on a large table). Re-running is safe. Pick a quiet window; deploy the backend before or with the SPA.
4. **Admin-only versus Audit for Compare with Tally Data.** It is an `audit` area screen (reads everything, saves a CSV);
   `admin` would be stricter.
5. **Which unverified statutory points to put in front of the CA** (section 5) before any figure is quoted to a customer.

## 3. Not attempted this phase (scout backlog)

Rows below were scoped by the research scouts and not built by either wave. Each line is the scout's own gap and approach,
shortened; value (1-5) and effort (S/M/L/XL) are the scout's. "Skip" rows were judged not worth building now.

### Accounting

- **voucher-class** (missing, P1, v4/L). No class concept. On an item invoice the user must pick every GST ledger row by hand; amounts prefill only after the ledger is chosen. No preset... Next: VoucherType.classes[] {name,itemLedgerId,taxLedgers{CGST,SGST,IGST,Cess},extras[{ledgerId,pct}]} stored in the voucherTypes doc. Pure...
- **numbering-override-multiuser** (missing, P2, v4/M). Only Automatic or Manual; no 'Automatic (manual override)'. Duplicate-number warning runs only for Manual, so two synced devices can both save... Next: Extract nextVchNo/formatVchNo from store into pure core/numbering.ts (keep overlays reservedFloor wrapper). Add numbering...
- **single-entry-mode** (partial, P2, v4/L). Payment/Receipt/Contra always show the Dr/Cr grid (particulars first, then bank via balancing row). TallyPrime's layout is Account first, then... Next: Per-type VoucherType.singleEntry plus in-voucher toggle on the existing Ctrl+H/Alt+V chord (free on these bases). View over the same...
- **line-narration** (partial, P2, v3/M). Voucher-level narration only; no per-ledger-line narration. Separately, no report (Day Book, Ledger Vouchers, Cash/Bank Book) shows narration at... Next: Optional VoucherEntry.narration; per-type VoucherType.lineNarration toggle (F12 in voucher). AccRow gains narr with its own focus column...
- **multi-address-party** (missing, P2, v3/L). Ledger holds one address/state/GSTIN. No address book, no per-voucher bill-to/ship-to choice; print Ship-to block just repeats the party address... Next: Ledger.addresses[] {id,label,address,pincode,state,gstin?} edited in a LedgerForm sub-panel (openingBills pattern). Voucher.shipToId...
- **multi-master-create** (missing, P2, v3/M). Only single-master Create screens. The sole bulk path is file import (CSV ledgers drop unknown groups silently into Suspense). No in-app tabular... Next: New multiMaster grid screen modelled on BulkEntry paste-from-Excel: Ledgers/Groups modes, columns Name/Under/Opening/Dr-Cr/GSTIN/State....
- **duplicate-insert-voucher** (partial, P2, v2/M). Alt+I Insert-within-date is not built. Alt+2 Duplicate is done in both Day Book and alteration. Skip: Tally's insert relies on renumbering following vouchers; renumbering issued invoices breaks GST Rule 46 serial integrity and audit trail, and...
- **numbering-advanced-series** (partial, P2, v2/L). Multiple named series inside ONE voucher type are not built. Restart, zero-pad, prefix/suffix tokens already exist. Skip: Functionally covered: a custom voucher type on the same base has its own counter/prefix/restart, switched with F10 (TypePicker), and GSTR-1 Documents...
- **vouchertype-flags** (partial, P2, v2/M). Only the effective-dates flag is missing; abbr, default-Optional and print-after-save are verified in code. Skip: Effective date has no downstream consumer here (ageing, interest and BRS use voucher date or bankDate; PDC and post-dated already exist). Needs...
- **zero-valued-entries** (partial, P2, v2/S). Accounting rows with amount 0 and item lines with qty 0 are dropped or rejected. Free items (qty>0, rate 0) are already accepted. Skip: The daily use case (free/sample items) already works: validInvLines needs only qty>0 and a zero-total invoice saves. Remaining nil accounting lines...
- **group-advanced-flags** (missing, P3, v3/M). Calc/sub-ledger/nett/allocation flags absent. Worse, Group.affectsGross is stored and editable but never read: P&L is hardwired to six reserved... Next: Extract pure plSections(groups,ledgers,totals) in core/accounting.ts: reserved slugs unchanged; any other top-level Income/Expense group...

### Advanced accounting

- **costcentre-parallel-allocation** (partial, P1, v3/M). CostPanel is one flat pool that must sum to the entry across all categories; with two categories a user cannot allocate 100% under each. Reports'... Next: CostPanel: group rows by centre.categoryId, track remaining per category, Ctrl+A requires each used category to be fully allocated; show...
- **credit-limit-override** (missing, P2, v3/M). Limit is warn-only on new Sales/DebitNote (VoucherScreen:1890, BulkEntry:572). No UI sets creditLimit; LedgerForm.onAccept rebuilds the Ledger and... Next: Add Credit limit field to LedgerForm (party ledgers) and spread ...editing so unknown fields survive. New core/credit.ts: creditCheck()...
- **currencies-master** (missing, P2, v3/M). No Currency entity; CLAUDE.md lists multi-currency as not built; importer has no forex handling. Next: New 'currencies' Dexie v8 table {symbol, formalName, decimals, prefix/suffix, rates[]} with embedded date-wise rates (covers...
- **forex-gain-loss** (missing, P2, v3/L). No fcy balance tracking, no unrealised (unadjusted) gain/loss vs period-end rate, no adjustment journal. Relevant to AS-11/Ind AS 21 restatement... Next: core/forex.ts fcyBalances(ledgers,vouchers,asOn) gives signed fcy and base per forex ledger; unrealised = fcyBal x closingRate -...
- **forex-voucher-entry** (missing, P2, v3/L). VoucherEntry holds only a base amount; no ledger currency, forex amount or rate fields. Exporters/importers cannot record USD receipts or invoices. Next: Ledger.currencyId; VoucherEntry.fcy {amount,rate}. Keep entry.amount as the BASE value so every engine and report is unchanged. In...
- **interest-advanced** (missing, P2, v3/L). Interest uses one rate and style per ledger, but no UI sets them (not in LedgerForm). Non-bill-wise mode charges the closing balance for the whole... Next: Ledger.interest {rate, per year|month|day, style, from, on balance|bills, side, rounding, slabs[]} edited in LedgerForm (spread...
- **interest-debit-note** (missing, P2, v3/M). InterestCalc is read-only; interest is never booked, so users retype it as a manual journal. Next: Row selection on InterestCalc plus 'Book Interest' action: one Journal per party (Dr party / Cr Interest Received via ensureLedger...
- **budget-costcentres** (missing, P2, v2/M). BudgetLine.targetKind is only 'group'|'ledger'; actualOf() reads periodTotals (ledger totals) so cost-centre actuals (VoucherEntry.costs) cannot... Next: Extend BudgetLine with targetKind 'costCentre' (+ optional ledgerId). Editor picker lists centres; actualOf sums e.costs Dr-Cr for the...
- **costcentre-class** (missing, P2, v2/M). No preset split: CostPanel always starts with one blank row at full amount; no voucher-class concept (CLAUDE.md: voucher class not built). Next: Skip voucher classes: add Ledger.costSplit [{costCentreId,pct}] edited in a LedgerForm sub-panel (like opening bills). CostPanel...
- **reversing-journals** (missing, P2, v2/M). No reversing base or applicable-upto date. Tally's reversing journal is non-posting and shows only in scenarios, so it is useless without a... Next: New VchBase 'ReversingJournal' (accounting:false so postsToLedgers already excludes it everywhere) in VCH_BASES/BASE_LABELS, acc-mode...
- **scenarios-master** (missing, P2, v2/L). No scenario entity or scenario-aware reporting. Optional vouchers exist but are only visible in the Optional register, never as an alternate view... Next: New 'scenarios' Dexie table {name, includeOptional, includeReversing, includeMemo, excludeActuals}. Pure core/scenario.ts...
- **base-currency-config** (partial, P2, v1/S). Base currency is implicit INR: ~34 hardcoded Rs/₹ sites, Indian lakh grouping and amountInWords are INR-only. Skip: Indian SMB base is always INR; configurability would touch every print and words engine for no switcher benefit. Treat INR as the fixed base constant...
- **payment-performance** (missing, P3, v3/M). No report of actual settlement speed per party; outstandingBills keeps only open balances, so settlement dates are not analysed. Next: Pure core/payperf.ts: walk postsToLedgers vouchers, map New Ref (and opening bills) to bill date/due, each Agst Ref on Receipt/Payment...
- **budget-period-distribution** (missing, P3, v2/S). A budget is one lump over from..to; Budget vs Actual compares whole-period only, no month-wise phasing or monthly variance. Next: Pure core/budget.ts distribute(amount,from,to,'equal'|'days') using largest-remainder to months, optional BudgetLine.monthly override...
- **forex-bill-settlement** (missing, P3, v2/L). BillAllocation is base-only; no fcy per bill, so Agst Ref at a different rate cannot compute realised gain/loss. Next: BillAllocation.fcy {amount,rate}. Extend billwise.ts to keep fcy balance per bill; on Agst Ref compute realised = settleFcy x...
- **forex-rate-table** (missing, P3, v2/S). No exchange-rate storage: no standard selling/buying rates by date and no last-voucher rate lookup. Next: Embedded in the currencies-master doc as rates[{date,std,selling,buying}] edited in a sub-panel. Pure rateOn(currency,date,kind) returns...
- **forex-report-columns** (missing, P3, v2/M). Outstandings and ledger reports show base only; no fcy column or fcy balance for forex ledgers. Next: Reuse core/forex.ts fcyBalances and per-bill fcy: add Forex column to Bills Receivable/Payable and Ledger Vouchers when...

### Inventory

- **additional-cost-purchase** (missing, P1, v4/L). Freight/loading/insurance on a purchase posts to its own ledger only; it never enters the item's stock cost, so closing stock, avg/FIFO cost and... Next: Ledger.apportion?:'Qty'|'Value' ('treat as additional cost of purchase'). Pure apportionAdditionalCost(lines, extras) in...
- **item-opening-multi** (partial, P1, v4/M). Item master takes one opening qty/rate/value; engine hard-books it to Main Location and never seeds byBatch. No per-godown or per-batch opening,... Next: Add StockItem.openingAllocs?:{godownId,batchName?,mfgDate?,expiryDate?,qty,rate,value}[]; openingQty/Value stay as the totals. Shared...
- **price-slab-discount** (missing, P2, v3/L). Price levels hold one flat rate per item per level (StockItem.prices); no quantity bands, no band discount %, no applicable-from date, and the... Next: Extend prices entries with optional slabs:{fromQty,rate?,discPct?}[] and from?:date. Pure priceFor(item,levelId,qty,date) in new...
- **rejections-in** (missing, P2, v2/M). No Rejections In voucher; a customer rejection can only be entered as a Credit Note, which posts accounting, instead of a stock-only inward note. Next: New VchBase RejectionsIn (stock in, accounting false, invoice true) in VCH_BASES/BASE_LABELS; reserved voucher type added to VT_DEFS...
- **rejections-out** (missing, P2, v2/S). No Rejections Out voucher; goods returned to a supplier can only be a Debit Note with accounting, not a stock-only outward note. Next: Same change as rejections-in with base RejectionsOut (stock out, no accounting); build in the same PR so the shared NOTE_BASES, VT_DEFS,...
- **cost-tracking** (missing, P2, v1/XL). No cost-tracking numbers, estimate-of-cost or landed-cost allocation to a tracking number across purchase, job and production vouchers. Skip: XL job-costing feature for niche manufacturing/job-work users; needs the Tally tracking-number spec and a manufacturing chain we lack....
- **actual-billed-qty** (missing, P3, v2/M). One qty per line; no delivered (actual) vs invoiced (billed) split, so free-scheme goods (10+1) cannot move stock without being billed. Next: InventoryLine.billedQty? (absent = qty). Stock engine keeps qty as actual; amount = rate x billedQty so cost per unit spreads over...
- **mrp-pricing** (missing, P3, v2/M). No MRP on stock item, batch or voucher line, so MRP cannot be recorded, printed or used for margin; pharma/FMCG invoices expect it beside... Next: Scoped to MRP capture/print only: StockItem.mrp?, optional per-batch MRP on the line, InventoryLine.mrp snapshot, F12 flag mrp adds MRP...
- **godown-price-override** (missing, P3, v1/L). No per-godown rate or reorder-level override on stock items. Skip: Niche for SMB; global reorder level plus Godown-wise Stock already cover most use. Revisit only if multi-warehouse customers ask.

### Manufacturing and job work

- **mfg-stock-posting** (partial, P0, v4/M). BOM-driven cost transfer works, but Manufacture hardcodes Main Location for every line (no godown pick), captures no batch/expiry for batch items,... Next: Extract costManufacture() to core/manufacturing.ts (components valued per item's method via stockPositions at date, overhead, outputs)....
- **bom-multiple** (missing, P1, v4/M). No BOM master: BOM is a flat per-unit array set only via 'Save as BOM' on first manufacture, then uneditable. No named/multiple BOMs or... Next: Add StockItem.boms?: Bom[] ({id,name,outputQty,components[{componentId,qty,wastagePct?}]}); bomsOf(item) wraps legacy bom as 'Standard'....
- **jobwork-godown-type** (missing, P1, v4/S). Godown has only name/parent/reserved. No Our Location vs Job Worker location flag, no linked job worker party, so stock lying with workers is... Next: Add Godown.kind?: 'Own'|'JobWorker' and Godown.jobWorkerId?: ledgerId. GodownScreen is the generic TreeMasterScreen whose onAccept...
- **material-in** (missing, P1, v4/L). No receipt of finished goods from a job worker that consumes the materials previously issued and absorbs job charges, with scrap/by-product return. Next: New base MaterialIn {stock:'both', accounting:false in v1}. 'jobWork' screen (kind:'material-in'): worker, optional order, goods...
- **material-out** (missing, P1, v4/M). No way to issue materials to a job worker without a sale or ownership change. Generic Stock Journal can move qty but the destination rate is... Next: New base MaterialOut {accounting:false, stock:'both'}. Dedicated 'jobWork' screen (kind:'material-out'): job worker party, challan no...
- **coproducts-byproducts** (missing, P1, v3/M). Manufacture screen yields exactly one 'in' line. Generic Stock Journal allows several dst lines but no co-product/by-product/scrap role, no... Next: Add InventoryLine.role?: 'component'|'product'|'coproduct'|'byproduct'|'scrap' and Bom.outputs[{itemId,qty,role,costPct?}]. Manufacture...
- **jobwork-out-order** (missing, P1, v3/L). No Job Work Out Order: nothing records materials to send and goods expected back from a job worker, with due date and a numbered series. Next: New base JobWorkOutOrder {order:true, stock:'none', accounting:false} (lineDir ignores it). Dedicated 'jobWork' screen...
- **mfg-cost-allocation** (missing, P1, v3/M). Finished value = components at avg cost + one typed lump 'additional cost'. No apportioning of input+overhead across multiple outputs by %, qty or... Next: Pure allocateCost(pool, outputs, basis) in core/manufacturing.ts: bases manual %, quantity, standard value, zero-cost; largest-remainder...
- **bom-nested** (missing, P2, v3/M). A component that is itself manufactured has no recursive explosion: no rolled-up raw-material requirement, no cost roll-up, no prompt to make the... Next: Pure explodeBom(itemId, qty, itemsById, {depth}) in core/bom.ts recursing via bomsOf(), compounding wastage, cycle detection (A to B to...
- **job-costing** (partial, P2, v3/M). Cost-centre P&L breakup, category summary and analysis exist for accounting entries (VoucherEntry.costs). Stock consumption lines carry no cost... Next: Add InventoryLine.costCentreId? (or voucher jobCentreId) on Stock Journal/MJ/Material Out. Pure jobCostSheet(centre, vouchers, ledgers,...
- **jobwork-order-tracking** (missing, P2, v3/M). pendingOrders() only tracks Sales/Purchase orders against invoices/notes by orderRef. No ordered vs issued vs received per job work order, nor... Next: Pure jobWorkOrderProgress(vouchers, upto) in core/jobwork.ts: per order x item, to-issue vs MaterialOut(orderRef), to-receive vs...
- **jobwork-reports** (missing, P2, v3/M). No stock-with-job-worker view, job work register, material movement or ageing of goods lying with workers; no ITC-04 style working. Next: Pure jobWorkLedger(vouchers,godowns,items,upto) in core/jobwork.ts: per worker x item issued/consumed/returned/balance valued, plus age...
- **jobwork-in-order** (missing, P2, v2/XL). No Job Work In Order: nothing records goods received from a principal for processing and materials to return. Skip: Niche inverse flow (we are the job worker). Needs a 'principal-owned stock, excluded from our valuation' semantic inside stockPositions, risking the...
- **mfg-print** (partial, P2, v2/S). Stock Journal prints as the generic Inward/Outward value table (invoiceModel isSJ branch). No cost sheet showing components, qty, rate, overhead,... Next: Voucher.mfg?: {finishedItemId, bomId?, overhead} set by Manufacture; buildCostSheetModel(v) in invoiceModel.ts feeds a third layout in...
- **bom-shortage-check** (missing, P3, v3/M). No check of component availability against the BOM before producing; Manufacture shows no on-hand column and no warning, and there is no... Next: Pure shortages(requirements, positions, godownId?) in core/bom.ts using stockPositions byGodown. Manufacture shows Available/Short...
- **bom-wastage** (missing, P3, v3/S). BOM lines carry no wastage/scrap %, so planned consumption equals net requirement and the allowance is invisible in cost. Next: Add BomLine.wastagePct? to the Bom model; gross consumption = qty x (1 + w%) applied in explodeBom and Manufacture prefill, shown as a...

### GST

- **multi-gstin** (missing, P2, v2/L). One GSTIN and one state per company; no per-branch registration, per-registration returns or voucher-to-registration binding. Skip: Multi-company is native, so one company per GSTIN is a workable path. True multi-registration touches isIntraState, numbering, returns and...
- **tax-ledger-config** (partial, P2, v2/M). Duty ledgers hold taxType plus an informational gstRate. No calculation basis (on value / on quantity) and no per-ledger rounding method. Skip: Tax amounts come from item and ledger rates via computeItemGst, and the invoice Round Off ledger already covers rounding. Per-quantity calculation...
- **composition-scheme** (missing, P3, v3/M). Reg type 'Composition' only adds a Bill of Supply print note. No CMP-08 quarterly liability, no category or rate (1/5/6%), GSTR-1 and 3B still... Next: core/composition.ts compositionTax(vouchers, category, quarter): turnover (Sales minus credit notes) x rate, CGST/SGST halves, plus RCM....

### TDS / TCS

- **etds-fvu** (missing, P1, v5/XL). No NSDL .txt export for FVU/RPU; current 26Q view is on-screen and CSV only. Skip: NSDL record-layout spec and the FVU validator are not in the repo and cannot be run offline; a hand-written file FVU rejects is fake statutory output....
- **form-27q** (missing, P2, v2/XL). No non-resident return or data. Skip: Return is an NSDL FVU file (no spec or validator in repo) and needs non-resident deductee data (country, TIN, DTAA, 15CA/CB) few SMBs hold. Cheaper...
- **tds-reversal** (missing, P2, v2/M). No guided 'reverse TDS' action; fix is by hand. Skip: Alteration already covers it: detectTds never re-runs on edit and the TDS and party rows are editable. Reversal after challan deposit is a refund or...
- **form-27eq** (missing, P2, v1/XL). No eTCS return or 27D certificate. Skip: Same NSDL FVU dependency as eTDS, and TCS volume is small (206C(1H) likely no longer applies, verify). No 27D data model. Defer behind etds-fvu.
- **form-27a** (missing, P3, v2/S). No return cover sheet. Skip: 27A is the control sheet FVU prints for the exact file it validated (counts, totals, checksum); a hand-made one has no standing. A totals panel falls...
- **form-26qb** (missing, P3, v1/L). No 26QB/QC/QD data sheet. Skip: These are filed online by individual payers on the TIN portal as challan-cum-statements; nothing to generate from books beyond a data sheet. P3 with...

### Payroll

- **calc-selected-components** (partial, P1, v4/S). Engine already sums PayHead.computeOnHeadIds[]; the pay head screen only lets you pick one head, and saving a multi-head head truncates it to the... Next: Replace the single 'Compute on' select in PayHeadScreen with a multi-select checklist popup (plus 'All Earnings'); persist the whole...
- **pay-period-config** (partial, P1, v4/M). Unmarked employees default to a hard-coded 26 days (vouchers.tsx:459) while 'period' proration divides by calendar days, so an unmarked employee... Next: Add Company.payrollConfig?: {daysBasis:'calendar'|'fixed'; fixedDays; unmarked:'full'|'zero'}. Pure resolvePayDays(cfg, from, to) drives...
- **payhead-statutory-types** (partial, P1, v4/L). statutory tag exists (PF/ESI/PT/Gratuity/Loan/Reimbursement) but is informational and offered only for Deductions. No employer-contribution head,... Next: Add PayHead.role ('employer') + contraLedgerId. Engine collects employer lines separately (excluded from gross/net); buildPayrollEntries...
- **payroll-autofill** (partial, P1, v4/M). runCompute processes every employee with a salary. No employee, group or subtree filter, no pay-head subset. Nothing stops processing the same... Next: Add a pre-step in PayrollVoucherScreen: all / employee group (descendants) / single employee, optional pay-head subset. Pure...
- **payroll-f11-statutory** (partial, P1, v4/M). Company holds only TAN and PAN. No PF, ESI or PT registration details, and PF/ESI/PT rates, ceilings and PT slabs are hard-coded constants in... Next: Add Company.payrollStat?: {pfRegn, pfCeiling, pfEeRate, pfErRate, esiCode, esiLimit, esiEeRate, esiErRate, ptRegn, ptState, ptSlabs}....
- **salary-it** (missing, P1, v4/XL). No TDS on salary (s.192): no projected annual income, regime choice, standard deduction, rebate, cess or declarations, and no per-month tax... Next: Pure core/salaryTax.ts: computeSalaryTds(profile, ytdGross, ytdTds, monthsLeft, slabTable). Slab tables are data per FY and editable,...
- **salary-revisions** (missing, P1, v4/M). Employee.salary is one structure, edited in place. No effective-dated history, so an increment rewrites the past for statutory/gratuity reports... Next: Add Employee.salaryHistory?: {from, salary}[]. Pure salaryOn(emp, iso) in core/payroll.ts picks latest row <= period end, else...
- **employee-code** (missing, P2, v4/S). Employee has only UUID + name. No employee code field, no auto-numbering, nothing shown in pickers, payslip, pay sheet or head-count. Next: Add Employee.code?; suggest next EMP-### (max numeric suffix+1) on create, editable, case-insensitive unique. Show via PopupItem.alt...
- **payhead-rounding** (missing, P2, v4/S). Every head amount is round2 only; no Normal/Upward/Downward rounding to the nearest N. ESI employee share must round up and PF to the rupee. Next: Add PayHead.round?: {mode:'normal'|'up'|'down'; to:number}. Pure roundAmount() in core/payroll.ts applied to each head amount before...
- **employee-categories** (missing, P2, v3/M). Payroll expense cannot be allocated to cost centres or departments by employee; no employee category dimension. Next: Add Employee.costCentreId? (picker from existing costCentres). buildPayrollEntries takes an employee-to-centre map and attaches...
- **reimbursement-heads** (missing, P2, v3/S). No reimbursement head. Phone, fuel and medical claims paid outside gross cannot be shown separately or excluded from the statutory and tax base. Next: PayHead.role 'reimbursement': engine adds it to net but not gross. Posts Dr head expense ledger, Cr Salary Payable. Payslip gets a...
- **salary-copy** (missing, P2, v3/S). Every employee's salary structure is keyed line by line; no way to clone another employee's structure. Next: In the Salary Details sub-panel add a 'Copy from' employee popup (panel-scoped useScreenKeys key; check keys.ts and the ? sheet for a...
- **salary-it-forms** (missing, P2, v3/XL). No Form 16, Form 12BA, 24Q or tax computation sheet. Skip: Form 16 Part A is TRACES-issued. 24Q needs the NSDL FVU file spec and validation, which we do not have. 12BA needs perquisite data. Revisit after...
- **salary-loans** (missing, P2, v3/L). No loan or advance handling. A recovery head posts Cr to an auto-created current-liability ledger (ensureLedger curliab), the wrong side for a... Next: Needs the ledger picker from payhead-statutory-types so recovery credits an Employee Advances asset ledger. Add Employee.loans?: {id,...
- **payslip-email** (missing, P2, v2/M). Payslips can only be printed. No individual or bulk email, and Employee has no email field. Skip: No mail transport: the app is browser/Electron and backend/ has no SMTP or provider. A mailto link cannot attach the PDF, so it would be a pale...
- **payroll-units** (missing, P2, v1/S). No user-defined or compound payroll units; only the four built-in attendance units. Skip: AttendanceType.unit (days/hrs/units/pcs) already covers day, hour and piece-rate work. Compound payroll units are rare and add a master for little value.
- **statutory-dashboards** (missing, P3, v3/M). The only PF/ESI/PT report projects from salary structure with fixed rates. No view from posted payroll of deducted vs employer share vs paid vs... Next: Pure statutoryDues(vouchers, payHeads, ledgers, period) groups posted payroll lines by PayHead.statutory tag: deducted, employer share,...
- **expat-reports** (missing, P3, v1/M). No passport, visa or permit expiry tracking for foreign employees. Skip: Niche for Indian SMBs (P3). If asked later, extend custom fields (UdfKind) to employees with a date field and an expiry report, instead of bespoke...

### Reports

- **report-range-filter** (partial, P1, v4/M). Phase 26 built the structured row filter (Alt+F12, `core/reportFilter.ts`, `ui/rowFilter.ts`) on the Day Book, every voucher register, Ledger Vouchers and Receivables/Payables, with totals that follow it and say 'Total (filtered)'; built and reviewed, not deployed, never compared with a live TallyPrime. Still open: it is not on Trial Balance, Group Summary, Balance Sheet, P&L, Stock Summary or the GST, TDS and payroll reports, so 'closing greater than 50,000' on a ledger or group cannot be asked. Next: one registry entry in `core/reportFilterFields.ts` and one `useRowFilter` call per report.
- **purchase-profitability** (partial, P2, v3/M). Purchase Register shows value only. No landed cost (freight and loading apportioned to items), no purchase-rate variance, no margin against sale rate. Next: Pure purchaseCostAnalysis in core/purchasecost.ts: for Purchase (minus Debit Note) lines, allocate each voucher's extra Dr ledgers (not...
- **report-value-filter** (partial, P2, v3/S). The Phase 26 panel also opens on Ctrl+F12 for name or value conditions (contains, does not contain, starts with, is exactly; is / is not for a fixed list) on the same five report screens (Day Book, voucher registers, Ledger Vouchers, Receivables, Payables). Still open: the same report coverage gap as the range filter, and no per-column exclude on a ledger or group list. Next: the same registry entries.
- **cancelled-register** (missing, P3, v4/M). CancelledRegister lists deleted vouchers from the change log, but a voucher cannot be cancelled retaining its number (Tally Alt+X). No cancelled... Next: Add Voucher.cancelled?: boolean set together with optional: true, so all 36 existing optional guards exclude it with no engine edits....
- **report-email** (missing, P3, v3/M). Top bar Share (M) is disabled and does nothing. No way to email or share a report from the screen. Next: Enable the Share top item with a tw-share event handled in App.tsx. shareReport() in reportTools.ts builds the .xls/.csv Blob from...
- **reversing-register** (missing, P3, v2/L). No reversing journal feature exists, so there is nothing to register. Skip: Depends on the unbuilt reversing-journals feature; auto-reversal would need expansion in every engine that walks v.entries (46 files) or books and...
- **scenario-columns** (missing, P3, v2/M). No scenario entity and no scenario column kind in reports. Skip: Depends on the missing Scenarios master. Optional vouchers are deliberately excluded from every total, and a provisional column is rarely used by...

### Banking

- **epayments** (missing, P2, v3/M). No bulk payment file. Payment Advice has beneficiary A/c and IFSC but nothing exports a NEFT/RTGS upload, and salary disbursement has none either. Next: Reduced scope: generic CSV only, not bank-certified. Pure core/paymentFile.ts: rows from bank Payments and salary net pay, validation...
- **multi-bank-allocation** (missing, P2, v2/L). Cannot split ONE bank line across several instruments inside a single allocation panel. Skip: Workaround already works: repeat the bank ledger line, each with its own instrument. Deposit slip, PDC, advice and BRS read per-entry. An allocations...
- **bank-interest** (missing, P3, v3/M). Interest report multiplies closing balance by whole-period days, not daily balances. No ledger-form fields set interestRate or interestStyle, so... Next: New pure core/interest.ts dailyBalanceInterest(): step function from postsToLedgers vouchers, sum of balance x days on the chosen side...
- **epayment-status** (missing, P3, v1/L). No tracking of e-payment status or download of bank statements. Skip: Needs bank host-to-host or corporate-banking APIs that we have no specs or credentials for; offline-infeasible. Manual status flags add little over...

### Admin

- **group-company** (missing, P2, v3/L). Multi-company exists but each report reads one company from the store. No group master, no merge of ledgers/groups by name, no consolidated Trial... Next: Pure src/core/consolidate.ts merges member datasets (groups by reserved slug or name+parent, ledgers by name+group, voucher ledger ids...
- **remote-users** (missing, P3, v1/L). No named remote users. Cloud tenants have membership(owner|admin|member) but no invite/accept flow, no per-member rights, and plan max_users is... Skip: Tally.NET remote access does not map to a hosted/synced product. The real equivalent is team invites on a tenant, which needs email delivery, enforced...

### Platform

- **f12-master-config** (partial, P2, v2/M). Master forms always show every optional field; F12 does nothing there. Users cannot hide GST, TDS or contact fields they never use. Next: Add optional flag to FieldSpec; useTallyForm filters flagged fields per form key from a per-company config store (reportConfig pattern)...
- **tdl-customization** (missing, P3, v4/L). Custom fields exist only on Ledger and Stock Item. The commonest reseller TDL job, extra invoice fields (vehicle no, PO no, dispatch doc), has no... Next: Scoped slice: add voucher to UdfKind and Voucher.udf (optional, REST sees jsonb). VoucherScreen adds a Custom Details step after...
- **export-other** (missing, P3, v2/S). Reports export CSV, XLS and print-to-PDF only. No standalone HTML or plain-text export; JPEG and SDF/ASCII absent. Next: In reportTools.ts add exportReportHTML (styled standalone page reusing the XLS builder's escaping) and exportReportTXT (fixed-width from...
- **http-xml-gateway** (missing, P3, v2/XL). No Tally-XML request/response surface for third-party connectors built against that protocol. Skip: Needs server-side XML engines (importer is browser DOMParser) and depends on export-xml/import-xml landing first; backend is deployed work. REST /v1...
- **data-path** (partial, P3, v1/S). Data lives in IndexedDB (browser) or the pinned Electron userData dir; no user-chosen data folder. Skip: Browser cannot choose a path. Electron userData is deliberately pinned to the Comhard Books literal so renames cannot strand IndexedDB; moving leveldb...
- **odbc** (missing, P3, v1/XL). No ODBC driver or data source for Excel/BI. Skip: Browser and Electron renderers cannot host an ODBC driver. Scoped REST /v1 API with keys, CSV/XLS export and webhooks already serve BI; a future OData...

## 4. Cases to run in a live TallyPrime (valuation)

None of Tally's own behaviour could be run here (the TallyPrime help pages returned HTTP 500; Tally.ERP 9 pages and search
summaries were used, and a few of Tally's own example tables are internally inconsistent). Run these on an Avg. Cost item and
compare Stock Summary closing at 30 April:

1. T1: buy 10 at 100 (1 Apr), sell 10 (5 Apr), buy 10 at 120 (10 Apr). We expect 1,100.00 (a perpetual average would show
   1,200.00).
2. T2: buy 10 at 100, buy 10 at 120, Debit Note 5 at 120. We show 1,650.00; if Tally lowers the inward pool it shows 1,600.00.
3. T3: buy 10 at 100, sell 5, Credit Note 2 at 150. We show 758.33; if Tally takes the return at cost it shows 700.00.
4. T4: a godown transfer by Stock Journal with the destination at a different rate: does Tally count the in-leg as an inward?
5. T5: the Stock Valuation FAQ example (5,335 units, 4,25,720.00 inward): we reproduce 4,25,719.98 at 31 March and
   4,25,719.93 on 1 April.

## 5. Statutory and schema points to confirm with a CA or against the source

Each of these is cited in a code comment or marked `SCHEMA?`; none was confirmed against the Act, the gazette or a live system.

- TDS/TCS: 206C(1H) omission from 1 Apr 2025 (Tally's help page and secondary sources only); the 1% no-PAN rate on 206C(1H);
  the Form 15G/15H section list (rent left out because sources disagree); a certificate rate applied with the nature's
  surcharge and cess on top; the catch-up on earlier payments at each payment's own-date rate; whether s.288B ten-rupee
  rounding applies to interest; interest months counted inclusively (TRACES) versus 30-day months; FY 2026-27 rates; the
  Income-tax Act 2025 renumbering; the 194IA/IB/194M 30-day deposit rule; the 195 rate (a placeholder); the TCS 206C(1) rates
  against any Budget 2026 change.
- GST returns: `at`/`txpd` key names and nesting; GSTR-3B `itc_rev` / `itc_inelg` mapping to 4(B)(1)/(2) and 4(D)(1)/(2);
  Rule 37 wording (from commentary citing Notification 26/2022-CT); QRMP facts (Rs 5 crore, optional IFF, PMT-06);
  GSTR-9 4F treatment of advances from an earlier year; Table 13 natures 2 and 7; HSN B2B/B2C split from May 2025; B2CL at Rs 1
  lakh (law effective 1 Aug 2024, portal change lagged); cess after September 2025; CGST Rule 46 clause letters and the B2C HSN
  exemption under Notification 78/2020-CT.
- e-Invoice: the IRN hash recipe (confirmed only against one vendor's worked example); buyer PIN optional in schema 1.1;
  6-digit HSN above Rs 5 crore; the IRP's allowed rate list; PIN-versus-state consistency; the 30-day reporting window; UQC
  list (GGR versus GGK); the 2 MB bulk limit.
- Cheque: the CTS-2010 leaf size (about 202 x 92 mm) was written from memory; physical alignment needs a test print.

## 6. Real files and live systems that would close several rows at once

- A real TallyPrime XML export (masters, Day Book, Trial Balance): `import-xml`, `export-xml`, `tally-compare-*`.
- One real bank statement per bank (and MT940/CAMT samples): `brs-auto-import`, `brs-format-config`.
- One sanitised IRP/portal results file and one real bulk-upload file: `einvoice-irn`, `einvoice-bulk`.
- A GSTR-1 and a GSTR-3B imported into the GST offline utility: `gst-json-export`.
- A real cheque leaf per bank laid over the ruler-grid test print: `cheque-config`.

## 7. Audit debt: covered rows whose "ours" text contradicts their status

These 40 rows are `covered` in `parity-data.json` but their `ours` text still starts "No ..." / "None" / "Not ...".
`docs/HANDOFF.md` section 5 lists most of the features as shipped (budgets, batches, FIFO, e-way, 26Q, payroll statutory, BOM,
interest, cheque, GSTR-2B, reminders, reorder), so the text is probably the stale half; if a row turns out not to have shipped,
its status is the wrong half instead. Not rewritten in this pass: no wave touched them and a note should not be rewritten without
reading the code. Next: a one-pass read of each against the code, a rewrite, then a drift assertion that fails on a `covered` row
whose `ours` begins "No ", the way the favourites suite guards its list.

`credit-limit`, `optional-vouchers`, `postdated-vouchers`, `budgets-master`, `budget-ledgers`, `budget-variance-report`, `interest-simple`, `batches`, `expiry-reports`, `price-levels`, `reorder-levels`, `bom-definition`, `bom-scaling`, `mfg-additional-cost`, `gstr2b-recon`, `eway-bill`, `eway-register`, `tds-deductor-details`, `employee-statutory-profile`, `employee-bank-details`, `calc-userdefined`, `calc-slabs`, `attendance-types`, `attendance-voucher`, `production-pay`, `pf-computation`, `pf-reports`, `esi-computation`, `esi-reports`, `pt-computation`, `pt-reports`, `report-auto-column`, `comparative-bs-pl`, `negative-ledgers-report`, `monthly-cashflow`, `reminder-letters`, `confirmation-accounts`, `exception-menu`, `audit-report`, `export-pdf`

## 8. Smaller defects and limits reported by the builders (not parity rows)

- **Menu placement.** `Cheque Books` (K) sits third in the Banking menu, so Post-Dated Cheques, Deposit Slip and Payment
  Advice each moved down one place (menu placement is frozen; hotkeys still work). Next: move it to the end of the list.
- **Cheque Books access area.** `chequeBooks` is classified `accounting` in `core/access.ts` but its actions save the bank
  Ledger doc, which `TABLE_AREA` puts under `masters`. Next: classify it `masters`.
- **Company menu hotkeys.** `Split Company Data` (Z) and `Sync Mismatch Report` (J) use letters that are not in their labels,
  so the hotkey is not underlined; reach them through Go To, the palette or the `?` sheet.
- **Backend.** `backend/src/rest.js` writes `change_log` without the new per-tenant push lock (a REST write committing out of
  order with a push could let a pull cursor pass an uncommitted row); the access token travels in the SSE query string (an
  EventSource cannot set a header) and shows in the console when a stream fails; the SSE hub and rate limiter are still
  in-memory single-process; plaintext passcodes or PINs pushed to the cloud change log before Phase 24 may still be in server
  history (the local outbox is cleaned, the server was not purged).
- **Sync semantics.** `importCompany` now writes outbox deletes for rows a restore removes, so restoring an older backup on a
  synced device deletes newer cloud records (the question names the count, the report records it, deleted records are not
  one-click recoverable). Older desktop installs keep working but ignore the server's adjusted copy of a Company row.
- **Samples.** Sample companies created before Phase 24, including the one loaded into the owner's cloud tenant by
  `scripts/seed`, carry GSTINs that fail the check character and no city/PIN, so GST Validation marks their vouchers Excluded
  and e-invoice reads "Needs fixing". Recreate the sample (the seed script refuses to run twice).
- **GST Validation.** A purchase from a Composition supplier that carries tax is flagged Uncertain but its input tax still
  flows into 3B 4(A) as booked.
- **Large books.** A 250,000-voucher company split takes 3 to 7 seconds of main-thread time; the Tally comparison reads files
  synchronously on the main thread with a 250 MB per-file refusal; the importer parses on the main thread.
- **Performance docs.** `docs/PERF.md` is stale: the `rpt.gst` row reads 142 ms against 1,159 ms measured at 250,000
  vouchers (identical at HEAD, so not a regression of this phase, and a reference-only row), its text still calls the row a
  "mirror", the trial-balance row's definition changed, and `dash.gst`, `dash.cards` and `rpt.stockFlow` have no baseline.
  Next: re-record PERF.md from `npm run perf`.
- **Printing.** Named `@page` rules (A5, Letter, landscape) are honoured by Chromium and Electron; Firefox and Safari may use the
  printer's paper. Thermal 80 mm paper is out of scope.
- **Branding.** The menu label "Compare with Tally Data" and the XML import/export screens use "Tally" nominatively on
  migration screens, which `docs/LEGAL-BRIEFING.md` permits; re-read that briefing before launch copy is written.
