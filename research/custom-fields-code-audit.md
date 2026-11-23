# Custom fields and reporting tags: a code audit of what exists and what a build must touch

> **Status:** audit only. Nothing here is built. Written 2026-10-21 (about 21:30 IST) against `HEAD` = `83d9876`.
> **Scope:** the repo's existing Custom Fields (UDF) feature, and every place a new object-wise custom-field feature (voucher header, voucher line, masters) plus a Zoho-style cross-document "reporting tag" would have to touch. The owner's four requests are in the task brief; this file answers "what is already in the code, what would silently break, and where would the new code go".
> **Method:** every claim below was read in the working tree this session (files named, line numbers given) or proven absent by a grep whose command is listed in Appendix B. No code was run and no gate (assert, shots, build, perf) was run: a wave-3 build owns the tree right now.
> **Evidence tags:** **[C]** read in this repo's code (file:line). **[R]** stated in a repo research or design doc and not re-checked against the vendor. **[J]** my judgement. **[unverified]** could not be checked here.
> **Companion docs:** three vendor-research files appeared in this folder while this was being written (`zoho-custom-fields.md`, `zoho-reporting-tags.md`, `tally-udf-and-competitors.md`); they were **not read** for this audit, so nothing here depends on them.
> **What this audit did NOT do:** no web research (it is a code audit; the Zoho, Pipedrive and Tally behaviour the owner asked about is the job of the vendor-research companion). No vendor behaviour is claimed below except where a repo doc is cited as [R]. There was no live Zoho, Tally or Pipedrive account and no running app.

### How to read the line numbers (the tree is moving)

The working tree held about 150 changed paths at the first read (`git status --short`): 64 modified files and the rest untracked (wave-3 parity build: dispatch and e-way, voucher cancel, single entry, server voucher numbering, export and share, mapped import). Two marks are used on file names:

- **†** modified against `HEAD` at read time. Line numbers for a † file are the **working-tree** numbers; they have already drifted from `HEAD` (for example `VoucherScreen.tsx` `const v: Voucher = {` is line 2603 at `HEAD` and 2736 in the tree). Appendix A gives both numbers for the anchors cited most, and every † citation also quotes an anchor string you can grep for.
- **‡** a new, untracked file at read time (wave 3). It does not exist at `HEAD`.

Unmarked files are identical to `HEAD`. Note for anyone planning against CLAUDE.md: it calls `VoucherScreen.tsx` "~2100 lines"; it is **3,882 lines at `HEAD` and 4,119 in the tree** [C `wc -l`].

---

## 0. The findings that decide the design (read this first)

1. **The feature is master-only and unfinished.** Custom fields exist for exactly two kinds, Ledger and Stock Item (`UdfKind`, `types.ts:402`; `UDF_KINDS`, `types.ts:406-409`). Five field types (text, number, date, yes/no, list) (`types.ts:403`). Nothing else in the app reads a custom field value: a repo-wide grep for `udf` finds six consumers in total (the editor, the form helper, the two master forms, one palette entry, one export warning) [C, Appendix B.1]. No report, print, filter, saved view, picker, import, API validation or webhook knows they exist.
2. **"Required" is a label only.** `CustomFieldDef.required` is written when a field is defined (`CustomFields.tsx:48`) and drawn in the list (`CustomFields.tsx:121,130`); no code ever reads it back. `FieldSpec` has no `required` property (`forms.tsx:46-65`) and the master `validate` functions never look at a custom value [C, Appendix B.2].
3. **The stored values are loose.** A custom date is stored as the text the person typed, not an ISO date (`udf.ts:23`, `udf.ts:51`); a custom number goes through `parseAmt`, which rounds to two decimals, and `0` is dropped (`udf.ts:49`, `money.ts:132-136`); "No" on a yes/no is not stored at all (`udf.ts:50`); a list field accepts any typed text at Ctrl+A (`udf.ts:25-26`, `forms.tsx:264`) and stores the label, not an id; there is no length cap. Any "Sales by Zone" built on these values today would split `North`, `north ` and `Nort`. See section 1.4.
4. **A voucher field has no home yet, and the voucher save is a from-scratch rebuild.** `Voucher` has no `udf` (`types.ts:991-1049`). `VoucherScreen.save` builds the saved `Voucher` as a **fresh object literal** (`const v: Voucher = {` at `VoucherScreen.tsx:2736`†), carrying through only the fields it names. Any new voucher field that is not added to that literal is **erased silently on every alteration**. Three more screens rebuild an altered voucher the same way (Manufacturing journal, Attendance, Payroll), and the voucher TYPE form does it for voucher types (section 2.2). There is a regression suite that guards this for Ledger and Stock Item (`master-resave.assert.ts`) and **none for vouchers**.
5. **Older clients will erase new fields, and the server cannot stop them.** The server stores whole documents as jsonb and merges nothing per field (`conflict.js:177-232`). Its only protection is for named fields of two tables (`conflict.js:38-41`) and, for `companies.ui` only, against a write that states no base revision (`conflict.js:54-56`, `196-206`). The push request carries no app version or capability marker (`sync.ts:615-632`†). A client that predates the feature but already sends a base revision (everything since Phase 24) will rewrite a voucher without the new field and the server will accept it as an ordinary edit. The backend half of Phase 27 is itself still undeployed (CLAUDE.md "Deployed" paragraph).
6. **The Enter walk is frozen, and custom fields already bend it.** The master forms append custom fields without `tail: true` (`udf.ts:20-28`), so for a company that defines even one ledger field, Enter on what used to be the last field no longer raises "Accept ?": it walks into the custom fields first (`forms.tsx:229-237`; `masters.tsx:528`†; `inventory/masters.tsx:452`). That is opt-in, but it is the precise failure mode the Phase 24 gotchas describe.
7. **The report filter cannot carry a custom field as written.** Filter field ids must match `/^[A-Za-z0-9_.-]{1,40}$/` (`reportFilter.ts:546`, applied at `:553`). The id format the planned catalogue reserves for custom fields, `udf:ledger:<key>` (`pipedrive-list-ux.md:244` [R]), contains a colon and would be dropped by the sanitiser; a `cf.<key>` id with a 40-character key would be 43 characters and also dropped. The field lists are static `as const` arrays joined to getters (`reportFilterFields.ts:29-96`), and `filterFieldsFor(reportKey)` has no company argument.
8. **The "report grid" the owner wants to hang custom columns on does not exist yet.** `core/reportFields.ts`, `listQuery.ts`, `listLayout.ts`, `columnSets/` and `ui/grid/` are all absent [C `ls`, Appendix B.3]. Saved views carry F12 config, added period columns, basis, period and a row filter, but **no column layout and no sort** (`SavedView`, `savedViews.ts:228-259`; sort is `useState`-only, `columnSort.ts:123`). The plan to build the grid is `docs/research/wave-a2-plan.md` (unbuilt).
9. **There is no group-by-an-attribute engine.** "Sales by Zone" needs a new summary. Stock Summary groups by stock group, category or godown only (`stockSummary.ts:237`, `:416-420`); the registers bucket by month (`reportRows.ts:36-41`); Top-N ranks customers or items (`mis.tsx:11-18`). A bill has no voucher id (`BillState`, `billwise.ts:8-16`), so a voucher-level field cannot reach Receivables and Payables without an engine change.
10. **Cost centres are the nearest existing dimension, and they do not tag invoices.** Allocations live on **entries** of the accounting grid (`VoucherEntry.costs`, `types.ts:885`), asked only for a ledger flagged `costCentres` (`VoucherScreen.tsx:2124-2150`). The invoice save builds its entries without `costs` (`VoucherScreen.tsx:2582-2591`†). A reporting tag that must mark a sales invoice cannot reuse the allocation panel as it stands.
11. **Webhooks and the REST API do not see in-app saves the way the architecture doc says.** Webhook events fire only from the REST write routes (`rest.js:74,88`); `sync.js` never calls `deliverEvent` [C, Appendix B.4]. There is no server-side custom-field validation: the `field_defs` table in `ARCHITECTURE.md:1575-1578` was never created (`backend/sql/` has no such table) and the architecture doc itself lists "UDF-over-API" as an unbuilt P4 nicety (`ARCHITECTURE.md:1767-1768`).
12. **For the "back for mouse users" item:** the breadcrumb is deliberately not clickable (comment at `widgets.tsx:258-261`†), the top bar has no Back (`widgets.tsx:99-111`), and the only back is each screen's own Esc handler plus a per-screen "Quit" button in the right-hand rail. A naive Back button that calls `st.pop()` would bypass the voucher's own "Quit ?" prompt (`VoucherScreen.tsx:3054-3058`). Section 7.

---

## 1. The existing feature, end to end

### 1.1 Data model

| What | Where | Detail |
|---|---|---|
| Kinds | `types.ts:402` | `UdfKind = 'ledger' \| 'stockItem'` |
| Types | `types.ts:403` | `'text' \| 'number' \| 'date' \| 'toggle' \| 'select'` |
| Value bag | `types.ts:404` | `UdfValues = Record<string, string \| number \| boolean>`: no arrays, so no multi-select; no nested values |
| Kind list for the editor | `types.ts:406-409` | two entries |
| Definition | `types.ts:411-421` | `id, cid, objectKind, key, label, type, options?, required?, order?`. Nothing for: where visible, show in reports, default value, help text, applies-to voucher type, searchable, active/archived |
| Values on Ledger | `types.ts:366` | `udf?: UdfValues` |
| Values on Stock Item | `types.ts:526` | `udf?: UdfValues` |
| Not supported kinds | `types.ts:402` | Voucher, VoucherEntry, InventoryLine, Employee, Lead, Unit, Godown, Cost Centre, Voucher Type, Company |

### 1.2 Storage, state and sync registration

| What | Where |
|---|---|
| Dexie table `customFields: 'id, cid'`, added in version 4, additive | `db.ts:229-252`, table property `db.ts:99`, repeated in v5's store list `db.ts:276` |
| In `TableName` and `ALL_TABLES` (so backed up, synced, split) | `db.ts:23-39` |
| Loaded with the company | `db.ts:567-598` (`customFields` read at `:590`) |
| Store state, collection name, empty default | `store.ts:214` (COLLECTIONS), `:242` (State), `:301` (EMPTY_DATA) † |
| Written through the generic `saveDoc` / `removeDoc` | `store.ts:674-695` † |
| **No delete guard**: `checkDelete` has no `customFields` case, so the default `return null` applies and a field in use can be deleted | `store.ts:797-884` † (default at `:881-882`) |
| Write right: the `customFields` table needs Administration | `access.ts:324` (TABLE_AREA); the screen needs it too, `access.ts:164` (SCREEN_AREA) |
| Server: synced as an ordinary table; REST can read but never write it | `conflict.js:69` (DATA_TABLES), `restPolicy.js:18` (REST_DENIED) |
| Labels in the undo, mismatch and audit screens | `undo.ts:107` †, `syncGuard.ts:545`, `AuditLog.tsx:18` † |

### 1.3 The registry editor (`ui/admin/CustomFields.tsx`, 151 lines)

What it does: pick a kind (`:81-91`), enter a label, type, options and a Required tick (`:94-116`), press Add Field (Alt+A, `:65`); a table lists the fields with a Delete button each (`:119-140`). The key is a slug of the label capped at 40 characters, falling back to `field`, made unique **within the kind** by appending `_2`, `_3` (`:17-18`, `:39-41`). `order` is set to the current count (`:48`). Esc quits (`:59-62`).

What it cannot do, all [C] from the same file:
- **No edit.** There is no way to change a label, type, options, required flag or order after creation. Delete and re-add is the only route, and because the key is a slug of the label, re-adding the same label reuses the key (`:39`), so old orphaned values re-attach to the new field.
- **No reorder.** `order = fields.length` (`:48`); after deleting a middle field two fields can share an order and `udfDefsFor` falls back to label order (`udf.ts:14`).
- **List options are a comma-separated string** (`:42-44`): an option cannot contain a comma, duplicates are not removed, there is no length cap and no cleaning of invisible or bidirectional characters. Compare the sanitisers used elsewhere for names that become display text: `cleanLabel` (`core/uiPrefs.ts:325`, 24 characters, strips format characters) and `cleanName` (`core/savedViews.ts`, 60 characters).
- **No "where visible" setting, no "show in reports" setting, no type-specific rules** (min, max, decimals, format).
- **The screen is hard to find:** it is not a row of the left menu (`navTree.ts` has no entry; `grep customFields src/ui/navTree.ts` is empty). It is reachable from the Gateway Company submenu (`MenuScreen.tsx:395` †, key D), Go To (`overlays.tsx:60` †), the palette (`CommandPalette.tsx:372` †) and favourites (`favouritesCore.ts:128` †).

### 1.4 How a definition becomes a form field and comes back (`ui/udf.ts`, 54 lines)

`udfFields` (`:17-30`) maps each definition to a `FieldSpec`; `udfInitial` (`:33-41`) seeds the form from `udf`; `collectUdf` (`:44-54`) reads the form back. The master forms call them at: ledger `masters.tsx:118,155,290,528` †, stock item `inventory/masters.tsx:338,361,539,452`. Behaviours that matter for any reporting built on the values:

| # | Behaviour | Evidence | Consequence for reports and filters |
|---|---|---|---|
| 1 | **Date stored as typed text**, hint `d-m-yyyy`, never parsed or converted to ISO | `udf.ts:23`, `udf.ts:51`; `forms.tsx:21,145-152` normalises only amount, qty and number | The filter's date kind needs ISO (`reportFilter.ts:138`); a custom date column cannot be filtered or sorted as a date until writes are normalised |
| 2 | **Number goes through `parseAmt`**: rounded to 2 decimals; and `if (n)` drops `0` | `udf.ts:49`; `money.ts:132-136` | 3.14159 becomes 3.14; `0` and blank are indistinguishable; a mandatory number field could never hold 0 |
| 3 | **Yes/No stores only Yes**; No is absence | `udf.ts:50` | "is No" and "never answered" are the same; a mandatory yes/no cannot tell |
| 4 | **List accepts any text at Ctrl+A** and stores the **label** | `udf.ts:25-26`, `udf.ts:51`; `forms.tsx:264` (`Ctrl+A` goes straight to `requestAccept(true)`); `masters.tsx:170-197` validate never reads `cf_` keys | `North`, `north`, `Nort` are different groups; a renamed option orphans old values (no id) |
| 5 | **Required is never read** | `CustomFields.tsx:48,121,130` are the only uses; `FieldSpec` has no such flag `forms.tsx:46-65` | See finding 2 |
| 6 | **No length cap on a stored value** | `udf.ts:51` | Every voucher payload and every change-log row carries it; the filter caps its own text at 80 (`reportFilter.ts:88`) and the mismatch screen shortens at 120 (`syncDiff.ts` `show`) but nothing caps the write |
| 7 | **Orphaned values**: `udf` is rebuilt from the **current** definitions only, and written as an own key | `udf.ts:44-54`; `masters.tsx:290` †; `inventory/masters.tsx:539` | Delete a definition, then Alter and Accept the master: that value is dropped silently. Until then it sits on the record, invisible |
| 8 | **Not a `tail` field**, so it joins the Enter walk | `udf.ts:20-28` has no `tail`; walk logic `forms.tsx:229-237`; placement before the other tails `masters.tsx:528` vs `:529-555`†, `inventory/masters.tsx:452` vs `:456` | Finding 6. `gst-identity.assert.ts:558` pins only that custom fields follow every built-in field, not that they are tails |
| 9 | **Appears in the Alt+C quick-create overlay too** (no `overlay` guard on the line) | `masters.tsx:528` † (compare `!overlay` at `:555`) | A mandatory ledger field would be demanded inside a voucher's quick-create |
| 10 | **Label is only trimmed** | `CustomFields.tsx:37` | invisible characters could make two columns read identically |

### 1.5 Where values reach (the reach matrix)

| Surface | Carries custom values? | Evidence |
|---|---|---|
| Ledger and Stock Item master forms | **Yes** | section 1.4 |
| Voucher entry, alteration, duplicate | No (no kind exists) | `types.ts:402`; `VoucherScreen.tsx` has no `udf` |
| Printed invoice and other prints | No | no `udf` under `src/ui/print/` or `core/printConfig.ts` (B.1); `PrintPart` is a closed union `types.ts:192-194` |
| On-screen reports | No | no report reads `.udf` (B.1) |
| Row filter and saved views | No | static field lists `reportFilterFields.ts:29-96` |
| Type-to-filter (Ctrl+F or `/`), CSV, XLSX, Print of a report | Would carry **any displayed column** automatically; no custom column is displayed today | `reports.tsx:200-330` † (ReportShell reads the DOM table); `core/reportSheet.ts:1-5`‡; `reportTools.ts:65,106` † |
| Ledger and item pickers, palette search | No | picker match fields `VoucherScreen.tsx:2866` (name, alias, part no.); palette aliases `CommandPalette.tsx:424` † (alias, part no.) |
| Backup file | **Yes**, whole rows plus the definitions table | `db.ts:443-466` (`exportCompany`) |
| Sync | **Yes**, whole document | `db.ts:332-346` (`putDoc` logs `payload: doc`) |
| REST read | **Yes**, payload spread into the answer | `rest.js:31-44` |
| REST write | Stores whatever body it is given, replaces the whole document; the definitions collection cannot be written | `rest.js:58-75`; `restPolicy.js:18` |
| Webhooks | Only for REST writes, not for in-app saves | `rest.js:74,88`; `platform.js:38`; no call in `sync.js` (B.4) |
| Server validation of values | **None** | no `field_defs` in `backend/sql/*.sql` (B.5); `ARCHITECTURE.md:1767-1768` |
| Tally XML export | **Not written**; only counted in a "not exported" note | `tallyExport.ts:741-742` † |
| Tally XML import | **Not read** | no `udf` in `tallyImport.ts` beyond a namespace comment at `:25` |
| CSV import and the mapped importer | No | `ImportData.tsx` † voucher row `:277-284`; `importMap.ts:40-70`‡ `FIELDS` is a closed record |
| Split Company Data | Rows copied and re-keyed; label-valued fields survive | `splitCompany.ts:298-352`, `:656-676` |
| Data Check (integrity) | Not checked | `integrity.ts:61-66` `BOOK_TABLES` excludes `customFields` |
| Dashboard | No | no `udf` in `src/ui/dash/` or `core/dash*` (B.1) |
| Mismatch Report and audit log | As a generic changed field with a humanised key (`udf — Zone`) | `syncDiff.ts:59-78` (`humanize`), leaves `:108-140` |
| Undo | Verbatim previous document | `undo.ts:221` †; applier `overlays.tsx:785` † |

### 1.6 Test and screenshot cover

There is **no suite and no screenshot recipe dedicated to custom fields** (B.6). The only incidental pins: `gst-rate-history.assert.ts:706,719` (the keys the form writes stay own keys, `udf` among them for StockItem and Ledger), `gst-identity.assert.ts:558` (position of `...udfFields` after `favouring`), `split.assert.ts:444` (the secret-key regex leaves `udf` alone). `scripts/assert/master-resave.assert.ts` guards that an Alter of a Ledger or Stock Item keeps every field the form does not own (the `...editing` spread must be first; `OFF_FORM` lists the keys no control writes); it reads the two literals from source.

---

## 2. Vouchers: where a header field and a line field would live

### 2.1 The types

- `Voucher` (`types.ts:991-1049` †): identity and number, `partyId`, `itemLedgerId`, `entries: VoucherEntry[]`, `inventory?: InventoryLine[]`, `payroll?`, a run of optional "sidecar" objects, `narration`. **No `udf`.**
- `VoucherEntry` (`types.ts:880-902`): `ledgerId, type, amount`, `bills?`, `costs?`, bank and instrument fields, cheque print marks, `tdsNatureId`, `challan`. This is the unit a cost centre attaches to.
- `InventoryLine` (`types.ts:944-973`): item, godown, qty, rate, amount, `dir?`, `discountPct?`, `altQty?`, batch fields, a posted **GST snapshot** (`gstRate, taxability, hsn, cessRate, gstOvr`) and `desc?`. `desc` is the existing precedent for an optional line field that is carried through alteration and printed (`VoucherScreen.tsx:3296-3304`†, `invoiceModel.ts:616`†).
- The precedent for a voucher-level optional object that rides along untouched: `exportInfo` (`types.ts:1033`), `dispatch` and `ewb` ‡ (wave 3, `types.ts:1037-1038` †), `cancelled`, `creditOverride`, `numberStatus`, `numberLog` (all wave 3). Each has an explicit carry line in the voucher save (section 2.2).

### 2.2 Every writer of a voucher, and what it does to a field it does not know

This is the master-resave lesson at the voucher layer. Rebuild-from-scratch sites drop an unknown top-level field on every alteration. Spread or patch sites keep it.

**A. Rebuild from scratch (an unknown field is lost on alteration unless the site carries it)**

| # | Site | Anchor | What is carried through today | What a build must do |
|---|---|---|---|---|
| A1 | **Voucher screen save** (all of Sales, Purchase, notes, orders, Journal, Payment, Receipt, Contra, Stock Journal, Physical Stock) | `const v: Voucher = {` `VoucherScreen.tsx:2736-2780` † | explicit carry lines: `exportInfo` (`:2761`), `irn, ackNo, ackDate, ewayBillNo` (`:2764-2767`), `...carriedNumbering(editing)` (`:2769`), `creditOverride` (`:2754`), dispatch and ewb from component state (`:2772-2773`). Everything else comes from state | add the new field here; seed it from `src` in the `useState` block (`:1120-1137`); reset it in `resetLines` (`:1640-1664`) |
| A2 | Line rebuild: `validInvLines` | `VoucherScreen.tsx:1446-1468` † | `stockItemId, godownId, qty, rate, amount`, the GST snapshot, `discountPct, altQty, batchName, mfgDate, expiryDate, desc` | a line field needs `ItemRow` (`:167-199`), `itemRowFromLine` (`:1152-1174`), `validInvLines`, the Stock Journal src/dst grids (`:1285-1296`, `:2593-2598`) |
| A3 | Entry rebuild, accounting grid | `VoucherScreen.tsx:2556-2567` † | `bills, costs, instrument*, chq, tdsNatureId, challan` via `AccRow` (`:118-141`, seeded `:1221-1242`) | an entry-level field needs `AccRow` and the seeding |
| A4 | Entry rebuild, **invoice mode** | `VoucherScreen.tsx:2582-2591` † | party entry keeps `bills`; item-ledger entry keeps `tdsNatureId`; **extras keep only `{ledgerId, type, amount}`** | an invoice entry cannot carry `costs` or any new entry field today |
| A5 | Stock Journal from Manufacturing | `manufacturing.tsx:178` † | nothing but identity, lines, narration; `numberedForSave(v, editing)` then carries numbering only | carry or consciously exclude |
| A6 | Attendance voucher | `payroll/vouchers.tsx:137` † | identity, payroll lines, narration | same |
| A7 | Payroll voucher | `payroll/vouchers.tsx:562` † | identity, entries, payroll lines, narration | same |
| A8 | Salary Disbursement (always a new Payment) | `payroll/disbursement.tsx:108` † | n/a (new each time) | system-generated: exempt from any mandatory rule |
| A9 | **Voucher TYPE form** (not a voucher, same trap) | `masters2.tsx:121-138` † | names `numberFloor` and `reserved` explicitly because the literal drops the rest (`types.ts:546` comment) | any per-type "these custom fields apply here" setting must be named here |
| A10 | Cancellation shell | `voucherCancel.ts:180-213` ‡ (`cancelledShell`), `sealCancelled` `:222-238` | keeps identity, number, legacy e-invoice fields, numbering story; drops lines, party and flags **by design** | a decision: do a cancelled voucher's tags survive (for a Cancelled register filtered by zone), or not |

**B. Create-only writers (no source document; the question is default and mandatory policy)**

| # | Site | Anchor | Notes |
|---|---|---|---|
| B1 | Bulk Entry (paste a block from a sheet) | literal `BulkEntry.tsx:652` † (`const voucher: Voucher \| null = ok`); columns `FIELD_DEFS` `:82-101`; rule 1 "not a validation bypass" `:37-63` | any mandatory rule has to apply here at the same severity |
| B2 | CSV Import Data, vouchers | `ImportData.tsx:277-284` † | fixed two-ledger row; no custom column |
| B3 | Mapped import (wave 3) | `importMap.ts:495` ‡; closed `FIELDS` record `:40-70` | natural place to add custom columns as targets |
| B4 | Tally XML import | `tallyImport.ts:1380`, `:1497` | UDF namespace tags are ignored |
| B5 | System journals: reverse-charge and ITC | `data/rcmWrite.ts:105-108` (new), `:97` (repost via `{...keep}`), `data/itcWrite.ts:49` | generated by the app; must be exempt from "mandatory" |
| B6 | Sample company and onboarding seed | `state/onboarding.ts:192-195` (`data.vouchers` at `:194`), `core/demo.ts` builds the sample | no custom fields today |

**C. Spread or patch writers (safe for an unknown top-level field)**

`ui/gst/gstr1.tsx:170` (`{ ...v }` then `exportInfo`), `core/gstAdvance.ts:247-259` (`withAdvanceRate`), `ui/banking/cheque.tsx:209-239`, `ui/banking/brsWrite.ts:96`, `ui/reports/markOptional.ts:99` via `ui/selection.ts:203` † (`withOptional`), `core/einvoiceState.ts:383-391` (`stripLegacyMock`), `data/rcmWrite.ts:97`, `sync/numbering.ts:183,192,242` ‡ (`updateDoc` of the stored row), undo (`undo.ts:221` replays the stored payload verbatim through `overlays.tsx:785`), backup restore (whole rows). All these write through `putDoc`, whose change-log row carries the whole document (`db.ts:340-344`).

**D. State seeding inside the voucher screen (each must start from the source and reset)**

| Flow | Anchor | Note |
|---|---|---|
| Alteration | `editing` `VoucherScreen.tsx:1078`; `src = editing ?? dupSrc` `:1085` | header state is `useState(src?.field ?? default)` (`:1120-1137`) |
| Duplicate (Alt+2) | `dupSrc` `:1084`; push `:3040`; button `:3454` | copies **content**; drops bills, instruments, cheque marks, TDS tags (`:1226-1238`) and never the IRN (pinned by `einvoice-wiring.assert.ts:46-47`); a header tag would be copied unless excluded |
| Ghost recall, invoice | `findRecall` `recall.ts:48`; `useGhost` `VoucherScreen.tsx:1730-1752` | copies item rows (via `itemRowFromLine(l, false)`), extras and the item ledger; **no header field is recalled** |
| Ghost recall, accounting grid | `findAccRecall` `recall.ts:143`; `recallAccRows` `VoucherScreen.tsx:1779` | rows only |
| PgUp / PgDn | `VoucherScreen.tsx:3043-3053` | remounts the screen on the neighbour (`editId`) |
| After Accept | `resetLines` `:1640-1664`; pins `PinKey` `:162`, state `:1696-1712` | the header fields it resets today: party (unless held), narration, place of supply (unless held), reverse charge, supplier invoice no. and date, dispatch and e-way. A new header field must be added here or it carries **invisibly** to the next voucher (the Phase 21 gotcha "a carried value must be visible") |

### 2.3 Undo, change log and sync conflicts

- **Whole document everywhere.** `putDoc`, `updateDoc`, `deleteDoc` log `payload: doc` or `null` (`db.ts:332-388`); undo replays the stored payload (`undo.ts:221`). An outbox row carries its `base` revision, stamped by the creating hook (`db.ts:322-324`, `sync/baseline.ts:50-53`).
- **Server conflict model is last-push-wins, recorded.** `classifyPush` (`conflict.js:177-232`): a stale write is still accepted; a mismatch is recorded when another device wrote the same document after the writer's base, unless the two payloads are canonically equal (`:213-215`). **There is no per-field merge.** Two people editing different custom fields of one voucher produce a recorded mismatch and one whole document wins.
- **Client pull is whole-document too.** `planRemote` (`syncGuard.ts:170-187`) holds a pulled change when an unsent local row exists for the same document.
- **Field-level only for display.** `syncDiff.ts` flattens both versions to leaves for the Mismatch screen (`:108-140`); inventory rows are matched by `stockItemId, godownId, batchName` (`ROW_KEYS`, `:51-57`), so a line custom field is **not** part of a line's identity there: two lines of one item and godown that differ only by colour would be labelled `(2)`.

### 2.4 Older-client protection: what exists, what does not

- **Masters:** an older client's master form spreads `...editing` first (`masters.tsx:235` †), so unknown top-level fields survive; but it then writes `udf: collectUdf(...)` from **its own copy of the definitions** (`masters.tsx:290`), so any stored value that has no definition on that client (a definition not yet synced to it, or one deleted) is dropped on that save.
- **Vouchers:** nothing. The older literal (A1 above) drops the field and the server accepts it.
- **Server rules available:** `PROTECTED_FIELDS` is `companies: vault, vaultLegacy, security` and `users: pinHash` (`conflict.js:38-41`), applied to **stale** writes (`:205`). `UNVERSIONED_PROTECTED` is `companies: ui` (`:54-56`), applied only when the writer states **no base** (`:196-206`, `own === null`). An app version that sends a base but predates the feature is covered by neither.
- **No capability marker:** the push body is `{deviceId, deviceLabel, changes[]}` with per-change `base`, `restoreId`, `sys` (`sync.ts:615-632` †). The server cannot tell an old client from a new one except by the absence of `base`, which only pre-Phase-24 clients lack (and rows any client wrote while signed out, `sync/baseline.ts:15-17` and `:46-53`).
- **The company-row defence pattern (Phase 27)** is a client mirror plus Restore for `Company.ui` only (`types.ts:174-181`); it does not extend to per-voucher data.
- **Deploy gate:** the backend VM runs the `a0fffd6` code; `conflict.js`'s `ui` rule is committed but not deployed, and a backend deploy needs secrets the owner supplies (CLAUDE.md "Deployed", HANDOFF 2d [R]). Electron installers lag the web app (CLAUDE.md Phase 28 gotcha on `cfg`).
- **A new table is safe for older clients but invisible to them:** `isKnownTable` filters pulled rows to `ALL_TABLES` (`sync.ts:583`, `:693`).

### 2.5 Server schema, REST payload, webhook payload

- **Storage:** one `change_log` table, `payload jsonb` (`001_p1.sql:34-45`); the only document index is `(tenant_id, tbl, doc_id, seq)` (`007_sync_conflicts.sql:54`). There is no index on any field of a payload and no server-side query by field.
- **Push:** `/sync/push` stores `JSON.stringify(ins.payload)` (`sync.js:177`, `:229`). Pull returns the payload whole (`sync.js:343-367`).
- **REST:** `GET /v1/:collection` takes only `cid` as a filter and returns every current document (`rest.js:31-44`); `POST` takes the whole body as the document and **replaces** the previous one (`rest.js:58-75`), so an integration that posts a voucher without `udf` erases it (as it would any field). `customFields` is read-only (`restPolicy.js:18`; documented at `ApiDocs.tsx:51`).
- **Webhook payload:** `{ event, data: { id, cid, payload }, ts }` for `<collection>.created`, `{ id }` for `.deleted`, HMAC-signed (`platform.js:38-70`, `rest.js:74,88`). It is only raised by REST writes. A "custom field changed" event does not exist and cannot without projecting `/sync/push`.

### 2.6 Backup, restore, split, Data Check, Dexie

- Backup writes every row of every table in `ALL_TABLES` (`db.ts:443-466`); restore **replaces** the company: `existing` rows of each table are deleted and the file's rows put (`db.ts:538-550`). A file made before a **new table** existed has no rows for it (`prepareRestore` tolerates the absence, `backupMigrate.ts:150-151`), so a restore deletes whatever the company holds in that table. Optional fields and new tables need no `BACKUP_SCHEMA_VERSION` bump by the file's own rule (`backupMigrate.ts:24-29`, `:30`).
- Split Company Data copies every table in `ALL_TABLES` and re-keys every id **by value, deeply** (`splitCompany.ts:298-352`, `:656-676`): a custom value that is a master id is re-keyed correctly, one that is a label is untouched, and a value equal to a voucher id of the other half is dropped as a cross-side link. `stripSecrets` removes only **top-level** keys that look like secrets (`:192-205`), so a nested `udf.token` survives.
- Dexie: `vouchers: 'id, cid, date, base'` (`db.ts:233`) has no index a custom field could use; **every report filters the in-memory store arrays** (`loadCompanyData`, `db.ts:567-598`). New optional voucher fields need no Dexie version; a new table needs v9 (additive, like v6, v7, v8: `db.ts:295-314`).

---

## 3. Entry screens: layout, the frozen walk and Accept-on-Enter

### 3.1 The master engine (`ui/forms.tsx`, unchanged since `HEAD`)

- Enter walks `visible` fields; the last walk field's Enter calls `requestAccept()` which validates and then raises "Accept ?"; **Ctrl+A** calls `requestAccept(true)` which validates and saves without the prompt (`forms.tsx:218-227`, `:262-266`).
- `tail: true` (`:64`) marks fields **appended after the walk**: "the walk ends where it always did"; a tail is reached by Tab, a click or Down, and Enter inside it walks it and then raises Accept (`:229-242`). Tail fields must come last in the list.
- `validate` returns a refusal band message; `warn` is an overridable caution (`:71-90`). `requestAccept` does **not** move focus to an offending field (no `setIdx`), so a required field in a tail would show an error band while the cursor stays on the last walk field.
- The master walk is **asserted from source** for the ledger form (`gst-identity.assert.ts:529-559`: the pre-Phase-24 key sequence `FROZEN`, appended keys after `drcr`, custom fields after `favouring`).

### 3.2 Precedents for "extra details that never enter the Enter walk"

| Precedent | Where | Pattern |
|---|---|---|
| **Alt+H dated GST history grid** on masters | `ui/gst/gstDetails.tsx:27` (`GST_KEY`), `:421-424`; wired at `masters.tsx:73-93` † | flat fields keep their place in the walk; a sub-panel opened by a chord holds the rest; `active: !gst.busy` gates the form's keys; `settle()` runs at Accept |
| **Alt+S dispatch / e-way panel** on vouchers (wave 3, uncommitted) | `VoucherScreen.tsx:3067-3068` †, state `:1133-1137`, `DispatchPanel` `ui/dispatch/DispatchPanel.tsx` ‡, rail button `:3447-3448`, `dspOpen` in `modal` `:1494`, in the dirty probe `:1635`, in the signature `:1615` | "never in the Enter walk, absent unless a person filled them", normalised so an emptied panel leaves no key (`:2772-2773`) |
| **F12 "Description" column**, appended after Amount, off by default | `core/voucherConfig.ts:1-50`; `ItemCol 'desc'` `VoucherScreen.tsx:202`; walk `:3296-3304`, `:3315`; print sub-line `invoiceModel.ts:616` † | opt-in per company (`Company.voucherView`, `types.ts:152`); Enter on Amount reaches it only when on |
| **Cost allocation panel** (a panel after the amount, per entry) | `VoucherScreen.tsx:498` `CostPanel`; chain `:2124-2150` | asked only for a ledger flagged for cost centres |
| **Alt+K held constants** (`PinKey = party \| godown \| pos`) | `VoucherScreen.tsx:162-166`, `:1696-1712`, `resetLines` `:1641` | a header tag typed over a run of vouchers would want this |

**Chords:** no Alt letter is free globally. The Wave A spec listed M, Q, X, Z as the last free ones (`zoho-ux-spec.md:30`, `:516` [R]) and U as reserved (`:517`); since then Alt+Q is the universal `+`, Alt+X cancels a voucher, Alt+Z is the price level and Alt+M is Share (`outputChords.ts:20`‡). Inside `VoucherScreen.tsx` the literal chords in use are 2 B C D E J K L N P R S T V Y plus Alt+H (GST) and the recall key [C chord list, Appendix B.7]; A, F, I, O and W are unbound in that file but each is bound on other screens or globally and would need a check against `keys.ts` and the registry's "no other file quotes a live registry chord" rule (`wave-a2-plan.md:39` [R]). A rail button is the mouse route in any case.

### 3.3 How a mandatory field meets Ctrl+A and Enter-accept

- **Masters:** put the check in `validate`; it is honoured on both paths (`forms.tsx:218-227`). A required field **inside the walk** is visited by Enter anyway; a required field in a tail is reached only by Tab/Down, so Enter-through-the-end would raise a refusal rather than a prompt. The ledger **opening bill-wise panel** has a second save path `obSave` (`masters.tsx:598-610` †) that re-validates only the name; the `led` object was already built after `validate` passed, so the custom value is checked once but the panel's save does not re-run it.
- **Quick-create overlay (Alt+C inside a voucher):** custom fields are in it (finding 1.4 row 9), so a mandatory ledger field blocks creating a party mid-voucher.
- **Vouchers:** the hard stops sit at the top of `doAccept` (`VoucherScreen.tsx:2348-2412` †, for example "Party A/c name is required!" `:2400`), and the screen moves the cursor to the offender with `go({...})` (`:2382`). Ctrl+A is `doAccept(true)` (`VoucherScreen.tsx:3006`), Enter on the narration is `doAccept()` (`:3368`), and the rail's Accept button is `doAccept(true)` (`:3451`). A mandatory header field is one more check there. A mandatory **line** field whose column is hidden (the F12 pattern) must either force the column on or move the cursor and say why.
- **Everything that writes vouchers outside this screen** (B1 to B6, REST) needs a stated rule: `BulkEntry.tsx:37-63` already requires the same severity as the voucher screen; `importMap.ts:154,165` ‡ has a `required` flag on import columns (a required column must be mapped before the import can run, and it is how the file's kind is guessed): the only other "mandatory" notion in the repo, and it is about columns, not values.
- **Old records:** making a field mandatory leaves every older voucher without it; unless the rule is date- or creation-scoped, fixing a typo in an old narration would be refused for a field the person never saw. [J]
- **System vouchers** (RCM and ITC journals, salary payment, cancellation shell) must be exempt.

### 3.4 What a new voucher header field touches inside `VoucherScreen.tsx` †

State init from `src` (`:1120-1137`); reset in `resetLines` (`:1640-1664`) and the pin set if it can be held (`:162`, `:1696`); the dirty signature (`:1613-1621`) and `typedNotEntered` (`:1626-1631`); `modal`/panel gating (`:1494`); the `Focus` union and `keyOf` (`:204-225`) if it enters the walk, the walk handlers and their Backspace chain (`:1996-2011`, `:3157-3209`); the validation in `doAccept` (`:2348-2412`); the save literal (`:2736-2780`); the rail buttons (`:3445-3454`); the header render block; duplicate (`:3040`), recall (`:1730-1752`) and PgUp/PgDn (`:3043-3053`) decisions.

### 3.5 Suites that pin `VoucherScreen.tsx` and the master walk by source text

A build that edits these lines must update the suite in the same change. [C each]

- `navstack.assert.ts:225` (the master form's dirty probe literal) and `:228` (the exact voucher line `useDirty(() => subOpen || dspOpen || (editing ? signature() !== loadedSig.current : dirty || typedNotEntered))`, with the Esc rule at `:229`); `crosscut.assert.ts:215` pins the `subOpen` definition (`panel !== null || batchFor !== null || gstFor !== null || adjPick || taxQ !== null || pickAsk !== null`), `:216` that the alteration signature mentions `r.bills`, `r.costs`, `r.instrument`, `partyBills`, `statAdj`, `numberText`, `r.desc`, `r.ovr`, and `:217` `typedNotEntered`. A new header or line field has to be added to the signature, and these literals updated, together.
- `einvoice-wiring.assert.ts:41`: the multi-line `irn: editing?.irn, ackNo: ..., ewayBillNo: ...` carry block; `:46-47` a duplicate never copies an IRN.
- `vchnumbering.assert.ts:910-914`: `...carriedNumbering(editing)` present and `reserveVoucherNumber(` located between `const taxPlans = plansIn.filter` and `const v: Voucher = {`.
- `party-terms.assert.ts:279-283`: the `creditOverride:` line and the `raised` expression; `:256-262`: price-level lines.
- `gst-identity.assert.ts:540-559` (ledger walk), `gst-rate-history.assert.ts:706,719` (own keys), `master-resave.assert.ts` (Ledger and StockItem spread-first and `OFF_FORM`), `shortcutprefs.assert.ts:647` (Alt+K shift), `singleentry.assert.ts:49-54` (`FIRST_DRCR`).
- Perf mirrors that quote source lines of `VoucherScreen.tsx`, `store.ts`, `reports.tsx`: `scripts/perf/mirrors.mjs:12-40` (for example `vch.save` quotes the `sortVouchers` line at `store.ts:294`†).

---

## 4. Reports

### 4.1 How rows get their fields today

| Report | Row type | Where fields are defined | Anchor |
|---|---|---|---|
| Day Book | the `Voucher` itself | `DAYBOOK_FIELDS` joined to getters by `fieldDefs` | `reports.tsx:696-700` †; fields `reportFilterFields.ts:29-36` |
| Voucher registers (`<Base> Register`) | `Voucher` | `REGISTER_FIELDS` | `reports/financial.tsx:369-372` †; `reportFilterFields.ts:38-44` |
| Ledger Vouchers | `Voucher` of one ledger | `LEDGER_VOUCHER_FIELDS` | `reports.tsx:1355-1359` †; `reportFilterFields.ts:46-54` |
| Receivables / Payables | `BillRow = { bill: BillState, ledger }` | `BILL_FIELDS` | `reports/outstanding.tsx:94-99`; `billwise.ts:8-16` |
| Stock Summary | `SSRow` over items and groups | none (no row filter); F12 group by `group \| category \| godown` | `stockSummary.ts:237`, `:416-420` |
| Stock Item Vouchers | `StockVoucherRow { voucher, inQty, outQty, rate, amount, balanceQty }` (the **line is not on it**; `Movement` has `line`, `inventory.ts:17-23`) | none | `inventory.ts:243-250`, `ui/inventory/reports.tsx` |

Four families have a row filter: day book, register, ledger, bills (`filterFamily`, `reportFilterFields.ts:77-86`). Every other report has none.

### 4.2 What a custom field must expose to be a filter

- A `FieldMeta { id, label, kind, options?, money? }` with `kind` one of `text | number | date | enum` (`reportFilter.ts:31`, `:61-72`). Yes/No maps to `enum`; a list field to `enum` with options from the registry; a date field needs **ISO** values (`:138`, `:147-152`); a number compares to the paisa when `money`, else to 0.0005 (`:94`, `:226-250`).
- A getter on the report's row type: for Day Book, registers and Ledger Vouchers it is `(v) => v.udf?.[key]`, O(1); for Bills it is not reachable (no voucher id on a bill).
- **Id constraints:** the stored condition's `field` must match `SAFE_FIELD = /^[A-Za-z0-9_.-]{1,40}$/` (`reportFilter.ts:546`, `:553`) or the condition is dropped when a view is read. `SAFE_REF` in saved views allows a colon but is for screen parameters (`savedViews.ts:122`).
- **Fail behaviour is safe in one direction:** a stored condition naming a field the report does not have is **kept** and matches nothing (`sanitiseFilter`, `reportFilter.ts:595-619`; compile `:280-300`), so a deleted custom field makes a view show an empty report with its chip on screen, never a widened one. Its chip would read `<id> (not in this report)` (`describeCondition`, `:466-469`) unless the caller passes the field list.
- **The field list is static and company-blind:** `filterFieldsFor(reportKey)` (`reportFilterFields.ts:93-96`) is called at `savedViews.ts:348, 378, 412, 553` and the views code needs the company's definitions threaded in to label and describe a custom condition.
- **Closed lists:** enum options live on `FieldMeta`; a saved view's stored enum value is not validated against them (`cleanCondition` `:576-579`), which is the only reason a per-company option list works at all.
- **Limits:** at most 12 conditions, depth 2, text cut to 80 characters (`:84-92`).
- **Older readers** drop a condition whose operator they do not know, which widens an AND (`cleanCondition` `:555-556`); the Wave A2 plan lists a "fail-closed reader" as its own first commit for that reason (`wave-a2-plan.md` rule A2-6 [R]). No new operator is needed for custom fields, so this is only a constraint on later work.

### 4.3 What a saved view holds today

`SavedView` (`savedViews.ts:228-259`): `id, name, report, screen, config` (the F12 options), `cols?` (added **period** columns, not data columns), `basis?`, `period?`, `at`, `filter?`, `scope?`, `category?`, `by?`, `custom?`. Company-wide views live on `Company.reportViews` (`types.ts:162`, at most 40) and are written only by an administrator. **There is no layout or sort field**; column sort is component state (`columnSort.ts:118-123`). "Quick-save the view" for a grid therefore needs a `layout` on the view, which the plan already specifies (`wave-a2-plan.md` "Layout" row; `pipedrive-list-ux.md` section 5.3, `:215` [R]).

### 4.4 The planned catalogue is unbuilt, and it already reserves a custom-field group

The plan: a pure `core/reportFields.ts` catalogue per row type, a lookup context built once per render, `ListLayout { v, ids, w?, sort?, pin? }`, a Columns panel, per-column header menu, and `fid(family, id)` so "field ids are forever" (`wave-a2-plan.md` contract C1 [R]). Party custom fields and item custom fields are listed as catalogue groups with ids `udf:ledger:<key>` and `udf:stockItem:<key>` (`pipedrive-list-ux.md:244`, `:252` [R]). `docs/research/pipedrive-list-ux.md:22` already notes custom fields would "become columns and filters everywhere for free" **if** the catalogue exists, and its Columns panel groups the catalogue by object ending in a "Your custom fields" group (`:318` [R]). None of `core/reportFields.ts`, `listQuery.ts`, `listLayout.ts`, `columnSets/`, `ui/grid/` exists [C, B.3]. The Dashboard's Overdue list is marked `REPLACE-WITH-LISTQUERY` in 16 places (grep) for the same reason.

### 4.5 Column, sort and summary: what is cheap and what is not

| Need | Status | Anchor |
|---|---|---|
| A custom **column** on Day Book, a register or Ledger Vouchers | cheap once a catalogue or a per-report column hook exists; every printed column then flows to type-to-filter, CSV, XLSX and Print for free | `reports.tsx:200-330` †, `reportSheet.ts:1-5`‡ |
| A custom **sort key** | one more entry in the report's `useColumnSort` list (`reports.tsx:704-716` †); `compareValues` handles numbers and natural text order, blanks last (`columnSort.ts:41-60`) | |
| A custom **filter** | section 4.2 | |
| **Party-level** field on a voucher report | a join through `ledgersById`; truth is "now", not as posted. The Day Book still does a linear `st.ledgers.find` per row (perf mirror `rpt.daybook.find`, `mirrors.mjs:39`) | `reports.tsx:~665` † |
| **Item-level** field on a line report | `Movement.line` exists (`inventory.ts:17-23`); `StockVoucherRow` does not carry it | `inventory.ts:243-250` |
| **Bills** by a voucher field | needs an originating-voucher index; `BillState` has no voucher id; the engine is perf-gated (`outstandingBills`, rows `rpt.outstanding`, `vch.billPanel`, `prim.outstandingBills` in `docs/PERF.md`) | `billwise.ts:8-22`; `pipedrive-list-ux.md` section 6.2, `:248` [R] |
| **Summary "Sales by Zone"** | a new engine: one pass over `registerRows` (guard: base, period, not optional, `reportRows.ts:16-18`) grouping by the field and summing `registerAmount` (`:25-33`), so it reconciles with the Sales Register. **Basis must be stated**: `registerAmount` is the GST-inclusive invoice value; the Dashboard's Sales card opens the Group Summary instead because register totals include tax (CLAUDE.md Phase 28 gotchas) | `reportRows.ts` |
| Stock **by Colour** at **item-master** level | extend the Stock Summary group-by with a flat mode like godown (`stockSummary.ts:237`, `:376-390`) | |
| Stock **by Colour** at **line** level (same item sold in several colours) | does not exist: `stockPositions` and `stockFlow` are keyed by item (and godown, batch); the only sub-item dimension is the batch (`InventoryLine.batchName`, `inventory.ts` byBatch). A line custom field would **not** change stock positions or valuation unless the engines are re-keyed | `inventory.ts:82-` [J] |

### 4.6 Print, export, import

- **Print (invoice):** the model builds item rows with sub-lines under the name for description, batch and alternate quantity (`invoiceModel.ts:616-620` †), so a line custom field can print as a sub-line **without changing the column plan** (`columnPlan`, `:293`). A header field needs a block; the printed parts are a closed union `PrintPart` (`types.ts:192-194`) with `PRINT_PARTS` and `DEFAULT_SHOW` (`printConfig.ts:95-119`), so a new switch is a union member plus a row there. Print layout is pinned by `scripts/assert/print.assert.ts`.
- **Report export:** CSV, XLSX and Print read the visible DOM table (`reportTools.ts:65,106` †; `reportSheet.ts:1-5`‡); cells starting `= + - @` are neutralised by the single writer `core/csvOut.ts` (CLAUDE.md Phase 24 gotcha). A displayed custom column therefore exports as text, and as a number only if right-aligned and a figure the app prints.
- **Import:** `ImportData.tsx` † is a fixed-header CSV importer; `importMap.ts` ‡ adds a column-mapping step with a closed `FIELDS` record (`:40-70`), which is where custom columns would be offered as targets. Both are additive and never overwrite (header comment `importMap.ts:1-20`).
- **Tally exchange:** neither direction reads or writes custom values (section 1.5). The XML importer's header comment names the `UDF` XML namespace (`tallyImport.ts:25`) but nothing reads a UDF tag. What a real TallyPrime export of TDL user fields looks like is [unverified].

---

## 5. Cost centres and categories: reuse points for a reporting tag

| Piece | Where | Reusable for a tag? |
|---|---|---|
| Masters: `CostCategory`, `CostCentre { categoryId, parent }` | `types.ts:633-646`; forms in `masters2.tsx`; creation entries in `quickCreate.ts` | the master pair, the category-of-centre idea (a tag **group** and tag **options**) |
| Company switch | `CompanyFeatures.costCentres` `types.ts:95`; set in `CompanyCreate.tsx:341-358` | the F11 pattern for "nothing changes unless switched on" |
| Per-ledger flag | `Ledger.costCentres` (written `masters.tsx:246`) | the "ask only where it matters" pattern; not enough for invoices |
| Allocation shape | `CostAllocation { costCentreId, amount }` `types.ts:875-878`; `VoucherEntry.costs?` `:885` | amounts split across centres on one entry; a tag on a line or header has no amount split |
| Entry UI | `CostPanel` `VoucherScreen.tsx:498`; `afterBills` and `continueAfterAmount` chain `:2124-2150` | accounting-grid entries only; **invoice entries are built without `costs`** (`:2582-2591`) and the party entry would take none |
| Engines | `core/costCentreTree.ts:105-125` (`allocationsByCentre`), `:129-148` (`centreVoucherRows`, a drill that equals the figure) | the "drill equals the figure" discipline; the engines read `entries[].costs` only |
| Reports | `reports/costcentrereport.tsx`, `costpl.tsx` (`:60-95`), `costanalysis.tsx`, `financial.tsx:723`, saved-view keys in `savedViews.ts:88-93` | a tag report can copy the report shell, F12 and views wiring |
| Referential rules | `store.ts:832-839` (category has centres, centre has allocations), `integrity.ts:197` (link check), `syncDiff.ts:56` (row identity) | a new master table needs the same three |
| Rights | `access.ts:325-327` (`costCategories`, `costCentres` under Masters) | same area for a tag master |
| Tally exchange | `tallyImport.ts:1454-1466` reads allocations; `tallyExport.ts:806` writes them | cost centres are the dimension that round-trips with Tally; a tag would not |
| Profit by tag | `ui/reports/profitability.tsx:36-37`: gross profit by item and by party only, COGS **at current average cost**, not at sale time | a margin-by-tag report cannot be built on it without a per-line cost basis; per-voucher margin was marked unverified in the earlier research (`pipedrive-list-ux.md` section 6.5, `:259` [R]) |

---

## 6. Rights, features, Dexie, performance

### 6.1 Rights

- Defining a field: Administration (`access.ts:164,324`). **There is no per-field right**: who may fill or see a custom value follows the voucher rights (`writeDenial` by base, action and date window, `access.ts:351-374`) or the Masters right.
- A new Screen needs a `SCREEN_AREA` entry or it is administrator-only and the access suite fails (CLAUDE.md Phase 24 gotcha; `access.ts:108-210`). A new table needs `TABLE_AREA` (`security-access.assert.ts:305-312` fails on a missing or stale entry).

### 6.2 Company feature switches

`CompanyFeatures` (`types.ts:91-101`) has no custom-field switch; the editor is always available. A reporting-tag feature could follow `costCentres` or the optional `discount?`, `tcs?` pattern (optional boolean, absent = off) so a company that never asked sees nothing new. [J]

### 6.3 Dexie and the registration checklist if a new table is added

Version 9 (additive) in `db.ts`; `TableName`, `ALL_TABLES`, the class property and `loadCompanyData` (`db.ts:23-39`, `:99`, `:567-598`); `store.ts` COLLECTIONS, State, EMPTY_DATA, `checkDelete` (`:208-218`, `:242`, `:301`, `:797-884`); `access.ts` `TABLE_AREA`; `undo.ts:107`, `syncGuard.ts:545`, `AuditLog.tsx:18` labels; `recents.ts`; `backend/src/conflict.js` `DATA_TABLES`; `ApiDocs.tsx:51` and `restPolicy.js` if the table is read-only to the API. **Suites that fail on drift:** `syncconflict.assert.ts:585-586` (`DATA_TABLES` equals `ALL_TABLES`), `security-access.assert.ts:305-312`, `einvoice-wiring.assert.ts` (REST_DENIED against ApiDocs), `backupmigrate.assert.ts:20-22`, `split.assert.ts:518`, `syncguard.assert.ts`. Extending `CustomFieldDef` instead (same table, extra properties) needs **no** Dexie version.

### 6.4 Performance

- Per-row custom getters are O(1); a group-by summary is one linear pass. The harness refuses any engine that imports `src/ui/`, `src/state/` or Dexie (`docs/PERF.md`; `wave-a2-plan.md` section 0.1(2) [R]), so the field catalogue and any summary engine must be pure `core/` files, and a change to a mirrored source line must update `scripts/perf/mirrors.mjs` in the same change.
- Reports read whole in-memory arrays (section 2.6); the row filter memoises on rows, filter and listed dependencies (`rowFilter.ts:119-140` (`useRowFilter`)), so a getter that reads a lookup table must list it in `deps`.
- A bill-to-voucher index touches the perf-gated bill engine. Value size is uncapped today (section 1.4 row 6) and every saved voucher and change-log row carries it.

---

## 7. The "way back" for mouse-only people (owner item 4): what the code does today

- **Back = Esc, per screen.** `st.pop()` (`store.ts:435`) is called from each screen's own Escape handler: reports pop directly (Day Book `reports.tsx:~733` †, CustomFields `CustomFields.tsx:60-61`); the voucher has its own guard, "Quit ?" when dirty and new (`VoucherScreen.tsx:3054-3058`); masters call `onCancel` (`forms.tsx:265`).
- **On-screen "Quit" exists only as a rail button each screen supplies.** Reports built on `usePeriodControls` get `{ k: 'Esc', label: 'Quit' }` from one place (`reports.tsx:584`); other screens have their own (`reports.tsx:1023, 1799, 1967`; `CustomFields.tsx:66`). 79 files carry a Quit, Back or Close button literal, 141 files handle Escape [C counts, B.8]; a complete list of routed screens with **no** mouse way back was not produced [unverified]. A person or the administrator can hide rail buttons in Settings (`ButtonPanel` applies the layout, `widgets.tsx:82-87` †): hiding never disables the key, but it removes the only mouse route.
- **The breadcrumb is plain text on purpose.** The comment at `widgets.tsx:258-261` † says a segment click "would pop several screens at once and bypass each screen's own Esc guard". The status-bar button at its start toggles the left menu only (`widgets.tsx:262-272`).
- **The top bar** lists Company, Data, Exchange, Go To, Import, Export, Share, Print, Help (`widgets.tsx:99-111`); no Back.
- **Navigation plumbing that a Back button should use:** `requestNav(target, source, after?)` judges rights and dirty work before moving (`ui/navigate.ts:54`), `leaveAndGo` / `stayPut` answer the "Leave this entry?" prompt (`:101`, `:124`), `popToStack` and `dirtyIndexes` are the pure pieces (`core/navPlan.ts:106-113`), dirty probes are registered per stack slot (`state/dirty.ts:32-80`). There is **no** `requestBack`. A Back control that calls `st.pop()` directly is safe on a report (reports hold no unsaved work) and unsafe on an entry screen.
- The crumb shows the last four segments with the whole path as a tooltip (`navPlan.ts:125-143`, `AppShell.tsx:77`); each screen's crumb name comes from `crumbFor` (`router.tsx:382` †).

---

## 8. What I could not verify

- Nothing was run: no assert suite, build, screenshot or perf run (a wave-3 build owns the tree). All behaviour is read from source.
- No web research and no vendor account: nothing here describes how Zoho, Tally or Pipedrive behave beyond what repo docs say, tagged [R].
- How a real TallyPrime export carries TDL user-defined fields, and whether Tally customers' UDFs map onto the five types here: [unverified].
- Whether any routed screen lacks a mouse way back (section 7): needs a sweep.
- Line numbers in † files will keep moving until wave 3 commits; Appendix A gives the `HEAD` numbers.
- Whether the VM backend currently deployed (`a0fffd6`) accepts a payload whose documents carry an extra field in every table: by construction yes (jsonb, `sync.js:229`), but it was not tested.

### 8.1 Decisions the code cannot make (each is tied to a fact above) [J]

1. **Where do voucher values live?** On the voucher document (`Voucher.udf`, simplest, but exposed to risk 2 and to the whole-document conflict model) or in a side table keyed by voucher id (an old client's rewrite cannot touch it, at the cost of a join in every report and a new table, section 6.3).
2. **Header only in the first release, or lines too?** A line field means `ItemRow`, `itemRowFromLine`, `validInvLines`, ghost recall, the Stock Journal grids, an F12 column switch and a print sub-line (sections 2.2 A2, 3.2, 4.6); and it will not change stock positions (section 4.5).
3. **What does "where it is visible" mean?** A voucher **type** (the per-type form is a rebuild site, A9), a voucher **base** (registers are keyed by base, `reportFilterFields.ts:75-96`), or both; and whether the same switch also decides report and print visibility.
4. **Report visibility default.** The repo's opt-in rule (Phase 27, and "Standard layout identical to today" in `wave-a2-plan.md`) points to: always **available** in the Columns panel and the filter, **not shown** in the standard layout unless the field says so.
5. **Field id format and length** (risk 4), decided before the first view is saved.
6. **List options by label or by id** (finding 3 and risk 6): renaming an option is either impossible or orphans history.
7. **Reporting tag: new table or a new kind of definition?** And on which unit: header, item line or ledger entry (cost centres sit on entries, section 5).
8. **Back for mouse users:** one `requestBack` that goes through the dirty probes (section 7), against adding a labelled Back button to each screen's rail.

---

## 9. Risks, ranked by blast radius

"Blast radius" = how many users and records are hit if it goes wrong, weighted by how silent the failure is.

| # | Risk | Why it is large | Where | What contains it [J] |
|---|---|---|---|---|
| 1 | **Silent erasure of a new voucher field on alteration** | every company, every alteration, no error; four voucher rebuild sites (A1, A5, A6, A7) plus the voucher-type form (A9) in section 2.2; no voucher resave suite exists | `VoucherScreen.tsx:2736` †, `manufacturing.tsx:178` †, `payroll/vouchers.tsx:137,562` †, `masters2.tsx:121` † | one shared "carry extras from `editing`" funnel (the new `numberedForSave` already is one for four writers), and a `voucher-resave.assert` modelled on `master-resave.assert.ts` that classifies every `Voucher` key |
| 2 | **Older clients erase the field and the server accepts it** | web tabs not reloaded and Electron installs lag; whole-document last-push-wins; no capability marker; backend deploy is gated on the owner | `conflict.js:38-56,196-206`, `sync.ts:615-632` † | a push-side capability marker plus a server rule, **or** accept and document, **or** store the values apart from the voucher document (own table keyed by voucher id) so an old rewrite cannot touch them |
| 3 | **Frozen Enter walk and Accept-on-Enter** | every Sales voucher if a header field enters the walk; masters already bend it for any company with a field; mandatory adds refusals on Ctrl+A | `forms.tsx:229-237`, `masters.tsx:528` †, section 3 | tail or panel placement, off by default, an assertion of the whole walk with the new keys removed (as `gst-identity.assert.ts:529-559`) |
| 4 | **Id contract for report fields** | ids are saved in views, some synced company-wide (`Company.reportViews`); a wrong format is un-fixable later; today's sanitiser rejects the planned format | `reportFilter.ts:546,553`, `pipedrive-list-ux.md:244` [R] | decide id format and length before the first stored view; widen `SAFE_FIELD` or cap the key |
| 5 | **Source-pinned suites and a moving tree** | at least eight suites quote `VoucherScreen.tsx`; wave 3 has 341 changed lines in that file, 31 in `masters.tsx`, and edits `types.ts`, `store.ts`, `reports.tsx`, `router.tsx`; the Edit tool refuses a stale file | section 3.5; Appendix A | start only after wave 3 commits; one owner per file at a time |
| 6 | **Value quality** (date text, rounded numbers, lost zero and No, free-text lists, no cap, orphans) | any report, filter or "by Zone" summary built on the values is wrong or fragmented from day one; mandatory cannot even be tested for number and yes/no | section 1.4 | typed, validated, canonical writes (ISO date, exact number, option membership, length cap) in one pure module; a `legacy value` rule for what is already stored |
| 7 | **Unenforced "mandatory"** | creation paths outside the keyboard screens bypass it (B1 to B6, REST), so reports show blanks the owner was promised away | section 3.3 | state the rule per path (error, warning, exempt) |
| 8 | **A new table's registration surface** | 11 code points and six drift suites; restore of an older backup deletes the new table's rows; older clients ignore it | section 6.3, `db.ts:538-550` | prefer extending the existing definitions table if definitions are the only new data |
| 9 | **Whole-document conflicts on header edits** | two people tagging different fields of one voucher still collide into a recorded mismatch | section 2.3 | none needed beyond the existing report; bulk tagging would write N documents |
| 10 | **Perf and purity** | summary and catalogue code must be pure `core/`; bill index touches perf-gated engines; mirrored source lines | section 6.4 | `perf --enforce` in the gate; mirrors updated with the edit |
| 11 | **Print layout** | closed `PrintPart`, column plan budget in mm, `print.assert.ts` | section 4.6 | line fields as sub-lines, header fields in one block |
| 12 | **API and webhooks** | REST POST replaces a document, no validation, webhooks never fire for app saves | section 2.5 | document it; a projector is a separate epic |
| 13 | **Two dimension systems** (cost centres and tags) | product confusion, two sets of reports; only cost centres round-trip with Tally | section 5 | a clear owner-facing rule for which to use |
| 14 | **Back control that bypasses guards** | a naive `st.pop()` on an entry screen loses typed work | section 7 | route through the dirty probes |

---

## 10. The exact files a build would touch, by lane

Marks: † modified in the tree at read time (wave-3 overlap: coordinate), ‡ new untracked wave-3 file, **new** = does not exist yet. Lanes are chosen so each owns a disjoint file set; the seams between them are named.

### Lane A. Definitions and value rules (the model)
`src/types.ts` † (`UdfKind`, `UdfType`, `CustomFieldDef`, `Voucher.udf?`, `InventoryLine.udf?`, `VoucherType` setting; also if kept on the company row: `Company`); **new** `src/core/customFields.ts` (pure: key rules, label and option cleaning, value coercion and validation, the legacy-value reader, mandatory evaluation, the id format); `src/ui/udf.ts` (rewrite to use it); `src/ui/admin/CustomFields.tsx` (edit, reorder, options, required, where-visible, delete guard); `src/state/store.ts` † (`checkDelete` case for `customFields`); `src/core/access.ts` † (only if a new screen or table). New suite `scripts/assert/customfields.assert.ts` with mutation list.

### Lane B. Masters
`src/ui/masters.tsx` † (ledger form: placement as tail or panel, `validate`, overlay behaviour), `src/ui/inventory/masters.tsx` (stock item), `src/ui/forms.tsx` (optional: a `required` flag and a "focus the offender" step), `src/ui/masters2.tsx` † (voucher type form: carry the per-type setting and the `numberFloor` it already carries), `src/ui/admin/ImportData.tsx` †, `src/core/importMap.ts` ‡ and `src/ui/admin/MappedImport.tsx` ‡ (custom columns as targets). Suites to update: `gst-identity.assert.ts`, `master-resave.assert.ts`, `gst-rate-history.assert.ts`.

### Lane C. Voucher entry
`src/ui/VoucherScreen.tsx` † (every point in section 3.4); `src/ui/VoucherConfig.tsx` and `src/core/voucherConfig.ts` (F12 switch for a line column, as `itemDesc`); **new** panel under `src/ui/vch/` ‡ directory (header details panel, following `src/ui/dispatch/DispatchPanel.tsx` ‡); `src/core/recall.ts` †; `src/ui/admin/BulkEntry.tsx` †; `src/ui/inventory/manufacturing.tsx` †, `src/ui/payroll/vouchers.tsx` †, `src/ui/payroll/disbursement.tsx` †; `src/core/vchNumbering.ts` ‡ (`carriedNumbering`) and `src/sync/numbering.ts` ‡ (`numberedForSave`) as the carry funnel; `src/core/voucherCancel.ts` ‡ (decide whether the shell keeps tags); `src/ui/credit/`, `src/ui/cancel/` ‡ are neighbours only. Suites to update: `navstack.assert.ts`, `crosscut.assert.ts`, `einvoice-wiring.assert.ts`, `vchnumbering.assert.ts`, `party-terms.assert.ts`; **new** `voucher-resave.assert.ts`.

### Lane D. Reports, filters, views, the grid
`src/core/reportFilter.ts` (`SAFE_FIELD`), `src/core/reportFilterFields.ts` (company-aware `filterFieldsFor`), `src/core/savedViews.ts` (calls at `:348,378,412,553`; `layout` on the view), `src/ui/rowFilter.ts`, `src/ui/reports.tsx` † (Day Book `:696-700`, Ledger Vouchers `:1355-1359`), `src/ui/reports/financial.tsx` † (registers `:369-372`), `src/ui/reports/outstanding.tsx` and `src/core/billwise.ts` (originating-voucher link), `src/core/stockSummary.ts` and `src/ui/inventory/stocksummary.tsx` (group by an item field), `src/core/inventory.ts` and `src/ui/inventory/reports.tsx` (line on `StockVoucherRow`), `src/ui/reports/columnSort.ts`, `src/ui/header/FilterBar.tsx`, `src/ui/reportViews/*`, `src/ui/reportsCenter/catalogue.ts` † (a summary report row; `EXCLUDED_REPORT_SCREENS`), `src/ui/router.tsx` †, `src/state/store.ts` † (`Screen` union), `src/core/access.ts` † (`SCREEN_AREA`), `src/ui/navTree.ts`, `src/ui/MenuScreen.tsx` †, `src/ui/modern/CommandPalette.tsx` †, `src/ui/overlays.tsx` †, `src/ui/quickCreate.ts` (new-screen checklist in CLAUDE.md). The planned **new** files: `src/core/reportFields.ts`, `reportFieldCtx.ts`, `columnSets/*`, `listLayout.ts`, `listQuery.ts`, `sortRows.ts`, `src/ui/grid/*`; a **new** pure summary engine (for example `src/core/groupBy.ts`) reconciled to `registerRows` and `registerAmount`. Suites: `reportfilter.assert.ts`, `reportfilter-rows.assert.ts`, `dashboard-custom.assert.ts` (reads source for F12 adopters), `sidenav.assert.ts` if a row is added, new `reportfields` and `listquery`.

### Lane E. Output and exchange
`src/ui/print/invoiceModel.ts` † and `src/ui/print/PrintVoucher.tsx` †, `src/core/printConfig.ts`, `src/ui/print/PrintConfigScreen.tsx`, `src/types.ts` † (`PrintPart`), `src/core/tallyImport.ts` and `src/core/tallyExport.ts` † (only if Tally interchange is wanted), `src/ui/admin/ApiDocs.tsx`. Report export needs no change (it reads the DOM). Suite: `print.assert.ts`.

### Lane F. Sync, server, older clients
`backend/src/conflict.js` (`PROTECTED_FIELDS`, `UNVERSIONED_PROTECTED`, `DATA_TABLES`), `backend/src/sync.js` † (a capability marker or a keep-current rule), `backend/src/restPolicy.js` and `backend/src/rest.js` (validation or documentation of `udf`), a **new** `backend/sql/00N_*.sql` only if server tables are added, `src/sync/sync.ts` † (push marker), `src/core/syncGuard.ts` and `src/core/syncDiff.ts` † (labels, line identity), `src/data/db.ts` † (v9 only if a new table). Suites: `syncconflict.assert.ts`, `syncguard.assert.ts`, `backupmigrate.assert.ts`, `split.assert.ts`. **Gate:** needs the owner's VM secrets to deploy.

### Lane G. Reporting tags (only if built as a new table)
`src/types.ts` †, `src/data/db.ts` † (v9), `src/state/store.ts` † (COLLECTIONS, State, `checkDelete`), `src/core/access.ts` † (`TABLE_AREA`), `src/core/undo.ts` †, `src/core/syncGuard.ts`, `src/ui/admin/AuditLog.tsx` †, `src/state/recents.ts` †, `src/core/integrity.ts` (links), `src/ui/masters2.tsx` † (a master form modelled on cost centres), `src/ui/quickCreate.ts`, the navigation and Go To files in lane D, `src/core/costCentreTree.ts` as the pattern, `backend/src/conflict.js`.

### Lane H. Back for mouse users
`src/ui/widgets.tsx` † (`StatusBar` `:231-300`, `ButtonPanel` `:82-97`, top bar `:99-111`), `src/ui/reports.tsx` † (`ReportShell` `:200-330`), `src/ui/AppShell.tsx`, `src/ui/pageHeader.tsx` and `src/ui/header/PageActions.tsx` †, `src/ui/navigate.ts` (a `requestBack`), `src/core/navPlan.ts`, `src/state/dirty.ts`, `src/core/railLayout.ts`, styles. Suites: `navstack.assert.ts`, `visual.assert.ts`, a new "every routed screen has a mouse way back" scan.

### Lane I. Gates
`scripts/assert/*` (the suites named above, plus mutation lists under `scripts/assert/mutations/`), `scripts/shots/recipes/*` (new recipes for the field editor, a voucher with fields, a report with a custom column), `scripts/perf/mirrors.mjs` and `scripts/perf/passes.ts` (any mirrored line moved).

**Seams between lanes:** A to C (the value module and the `Voucher.udf` type), A to D (the id format and `FieldMeta` shape), C to F (what an older client may rewrite), D to H (`ReportShell` is shared), and every lane that edits `types.ts`, `store.ts`, `router.tsx`, `VoucherScreen.tsx` or `reports.tsx` (wave-3 overlap).

---

## Appendix A. Line drift: `HEAD` against the working tree (read 2026-10-21 about 21:35)

| File | Anchor | HEAD | Tree |
|---|---|---|---|
| `src/ui/VoucherScreen.tsx` | `type Focus =` | 190 | 204 |
| | `type PinKey` | 148 | 162 |
| | `interface AccRow` | 104 | 118 |
| | `interface ItemRow` | 153 | 167 |
| | `const CostPanel` | 484 | 498 |
| | `const itemRowFromLine` | 1133 | 1152 |
| | `const validInvLines` | 1414 | 1446 |
| | `const signature = ` | 1558 | 1613 |
| | `useDirty(() => subOpen` | 1578 | 1635 |
| | `const resetLines` | 1583 | 1640 |
| | `const doAccept` | 2261 | 2348 |
| | `const save = async` | 2439 | 2540 |
| | `const v: Voucher = {` | 2603 | 2736 |
| `src/ui/masters.tsx` | `...udfInitial(st.customFields, 'ledger', editing.udf)` | 112 | 118 |
| | `...editing,` | 216 | 235 |
| | `udf: collectUdf(st.customFields, 'ledger', v)` | 266 | 290 |
| | `...udfFields(st.customFields, 'ledger')` | 504 | 528 |
| | `const obSave` | 571 | 598 |
| `src/types.ts` | `export interface Ledger {` | 283 | 288 |
| | `export interface StockItem {` | 483 | 488 |
| | `export interface VoucherEntry {` | 871 | 880 |
| | `export interface InventoryLine {` | 935 | 944 |
| | `export interface Voucher {` | 982 | 991 |
| `src/state/store.ts` | `const COLLECTIONS` | 203 | 208 |
| | `saveDoc: async` | 667 | 674 |
| | `saveVoucher: async` | 690 | 697 |
| | `const sortVouchers` | 287 | 294 |
| | `function checkDelete` | 790 | 797 |
| `src/data/db.ts` | `exportCompany` | 442 | 443 |
| | `importCompany` | 498 | 499 |
| `src/ui/reports.tsx` | `const dayFields = fieldDefs` | 694 | 696 |
| | `const lvFields = fieldDefs` | 1344 | 1355 |
| `src/ui/reports/financial.tsx` | `const fields = fieldDefs<Voucher, typeof REGISTER_FIELDS>` | 368 | 369 |
| `src/ui/print/invoiceModel.ts` | `if (l.desc?.trim()) sub.push(` | 605 | 616 |
| `src/core/tallyExport.ts` | `master${udfN === 1` note | 739 | 742 |

Files cited above that are **unchanged** (same lines at `HEAD`): `src/ui/udf.ts`, `src/ui/admin/CustomFields.tsx`, `src/ui/forms.tsx`, `src/core/reportFilter.ts`, `src/core/reportFilterFields.ts`, `src/core/savedViews.ts`, `backend/src/conflict.js`, `backend/src/rest.js`, `backend/src/platform.js`, `backend/src/restPolicy.js`, `src/core/splitCompany.ts`, `src/core/reportRows.ts`, `src/core/backupMigrate.ts`, `src/ui/inventory/masters.tsx`, `src/core/inventory.ts`, `src/core/stockSummary.ts`.

## Appendix B. The searches behind the "absent" claims

Run from `/Users/Akshay/tallyreplica`, 2026-10-21.

1. **Every consumer of a custom value:** `grep -rIl "udf" src backend scripts electron` returned seven source files, `src/types.ts` (the declarations), then the six consumers `src/ui/udf.ts`, `src/ui/masters.tsx`, `src/ui/admin/CustomFields.tsx`, `src/ui/inventory/masters.tsx`, `src/ui/modern/CommandPalette.tsx` (a Go To alias for the screen) and `src/core/tallyExport.ts` (a "not exported" note), plus three assertion suites (`gst-identity`, `gst-rate-history`, `split`). Nothing under `src/ui/print/`, `src/ui/reports*`, `src/ui/dash/`, `src/core/report*.ts`, `src/core/savedViews.ts` or `backend/`. The wiring of the **definitions table** is a separate grep, `grep -rIl "CustomFieldDef\|customFields" src`: `types.ts`, `udf.ts`, `router.tsx`, `MenuScreen.tsx`, `overlays.tsx`, `favouritesCore.ts`, `masters.tsx`, `CustomFields.tsx`, `ApiDocs.tsx`, `AuditLog.tsx`, `inventory/masters.tsx`, `CommandPalette.tsx`, `undo.ts`, `access.ts`, `recents.ts`, `syncGuard.ts`, `store.ts`, `db.ts`.
2. **`required` reads:** `grep -rn "\.required\|required:" src --include=*.ts --include=*.tsx` shows `CustomFields.tsx:48,130` as the only custom-field uses; the other hits are the importer's own `FieldDef.required`, e-way and unrelated.
3. **Planned grid files:** `ls src/core/reportFields.ts src/core/listQuery.ts src/core/listLayout.ts src/core/columnSets src/ui/grid` all "No such file". `grep -rn REPLACE-WITH-LISTQUERY src` gives 16 lines.
4. **Webhook callers:** `grep -n deliverEvent backend/src/*.js` shows `platform.js:38`, `rest.js:3,74,88` only.
5. **Server field definitions:** `grep -n "udf\|customFields\|field_defs" backend/src backend/sql` shows only `conflict.js:69` and `restPolicy.js:18`.
6. **Dedicated suites or recipes:** `grep -n udf scripts/assert/*.ts` and `grep -rn "udf\|customField" scripts/shots` show only the three incidental pins named in section 1.6.
7. **Chords inside `VoucherScreen.tsx`:** `grep -oE "'(Alt|Ctrl)\+(Shift\+)?[A-Z0-9]'|GST_KEY|RECALL_KEY|'F[0-9]+'"`.
8. **Back-button counts:** `grep -rlE "label: '(Quit|Back|Close)'" src/ui | wc -l` = 79; `grep -rl "'Escape'" src/ui | wc -l` = 141; `grep -rl ButtonPanel src/ui | wc -l` = 58.
