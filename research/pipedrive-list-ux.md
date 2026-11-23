# Pipedrive-style list views for Comsky reports: columns from related objects, the quick filter, header menus

> **Status:** research and design only. Nothing here is built. Written 2026-10-20 for the owner's ask of 2026-10-19 (verbatim in section 2).
> **Method:** Pipedrive's own public help and product pages (support.pipedrive.com, developers.pipedrive.com, pipedrive.com blog), read through a web fetch that summarises each page, plus our working tree. Everything is paraphrased; no Pipedrive text or screenshot is reproduced. No Pipedrive account was used and no screen was seen live.
> **Evidence tags:** **[P]** a Pipedrive page I fetched (URL in section 15); **[Ps]** only a web-search summary of a Pipedrive page, not fetched in full; **[owner]** the owner's written description; **[C]** read from code (file named); **[Z]** from `docs/research/zoho-books-reference.md`; **[J]** my judgement; **[inferred]** my reading, not a vendor statement; **[unverified]** I looked and could not confirm it.
> **Reads on top of (not repeated here):** `docs/design/zoho-ux-spec.md` (Wave A: `+ New`, filter bar, views dropdown, Reports Center, menu v2), `docs/design/ux-analysis-2026-10.md` (findings F-11, N2) and `docs/research/zoho-books-reference.md`. Wave A is uncommitted while this is written, so file and function names below may shift.

---

## 0. Read this first

**What could not be checked, and why it matters.**
1. **The fetch tool summarises each page with a small model**, so "the article does not say" means "the summary did not say". Items marked **[unverified]** below are worth ten minutes in a live Pipedrive trial before a spec is locked: column pinning, a per-column header menu beyond sort and filter, per-user versus shared column choice, and whether the column picker groups fields by object.
2. **Two Pipedrive pages disagree on scope.** The quick-filter article says it exists only on the deals list; the advanced-filtering article describes quick filters on the pipeline and on the deal and contact lists **[P]**. Treat the quick filter as "deals first, widening".
3. **One Pipedrive source is old.** The "new list views" blog post is from 2015 (its page header shows a 2026 modification date) and says its product descriptions were true only when written **[P]**.

**The ten findings that change what we build.**
1. **The owner's three loves are real and documented, with one correction.** Columns from related objects (deal list shows person and organisation fields), a quick-filter bar built by "add condition", and click-to-sort plus hover-to-filter on column headers are all in Pipedrive's docs **[P]**. A header menu that offers hide or pin is **not** documented **[unverified]**; hiding happens in the gear-icon column picker.
2. **Pipedrive's whole trick is one idea: every list row is an OBJECT, and a column is a FIELD of that object or of an object linked to it.** Our reports are mostly hand-written tables with hand-written accessors. The thing to build is not a picker; it is a **field catalogue per row type** that filter, sort, cell, total and export all read.
3. **We are closer than it looks.** Wave A's `FieldDef`/`useRowFilter` already give filter fields their own accessors, and `useColumnSort` already takes `{key, label, get}`. Today each report writes the same accessor three times (filter fields, sort columns, JSX cells); the catalogue merges them (section 5).
4. **The biggest gap is not in the reports, it is in the masters.** Ledgers and Stock Items open as pick lists (name plus one sub-label), not tables [C `ui/masters.tsx` `PickMaster`]. They are our true "People/Organisations/Products": 60-odd fields each, plus custom fields. A Pipedrive-style **Party list** and **Item list** is the best single use of the pattern (section 6.4).
5. **Our custom fields (UDF) become columns and filters everywhere for free.** Ledgers and stock items carry `udf` values with a `CustomFieldDef` registry [C `types.ts`, `ui/udf.ts`]; through the party and item relations they can appear on every voucher, bill and stock report. Pipedrive treats custom fields as ordinary picker fields **[P]**.
6. **A column picker answers the status-aware list the UX analysis asked for (N2).** Status, due date and balance of an invoice are three computed columns from the bill engine, not a new screen (section 6.1).
7. **Totals are the trap, not the picker.** Pipedrive has no per-column footer; its totals are a summary box that follows the filter **[P]**. Ours are statutory-grade Total rows. A column from a related object (a party's credit limit on a list of vouchers) must **never** be summable (section 7.1).
8. **Print and Export read the rendered DOM** [C `ui/reportTools.ts` `tablesToRows`, `printReport`]. That makes added columns appear on paper for free, as long as a hidden column is **not rendered at all** (never CSS-hidden). It also means the day we window long lists (no windowing exists in `src/`, grep), Print, Export and the `/` text filter must become model-driven first (section 12, R1).
9. **Pipedrive's list is mouse-first** (the only list keyboard shortcut I found is `/` for global search, **[Ps]**). Ours is keyboard-first. So every Pipedrive affordance below is added as a **mouse door onto a keyboard-complete core**: the Columns panel does everything the header menu does, with keys.
10. **This is also the answer to "Create New Report".** Columns plus filter plus sort plus a name is "a register with my columns and my filter", the 80% case the Zoho spec said saved views would cover once they hold filters and columns (zoho-books-reference 6.2). No free-form builder is needed.

---

## 1. How Pipedrive's list views work

### 1.1 What a list is
- **Where:** list views exist for Leads, Deals, Contacts (people and organisations), Activities, Projects and Products **[P list-view]**. The Leads Inbox is a similar list with its own article **[P leads-inbox]**.
- **Per-type:** a filter belongs to one item type; a deal filter cannot be used on the people list **[P bulk-editing]**.
- **Furniture:** a `+` at the top adds a record (reads "+ Person" on contacts); a filter dropdown (top right); a gear icon at the far right for columns; a refresh button; on contacts an alphabet dropdown (first letter of the name; Pipedrive says initials filtering has moved to quick filters) **[P list-view, P filtering]**. Hovering a record shows a small summary card (a person shows name, email, phone, organisation; a deal shows title, value, contacts, expected close, pipeline and stage) **[P list-view]**.

### 1.2 The column picker ("add columns from a particular object")
| Question | What the docs say | Tag |
|---|---|---|
| How do I open it? | Gear icon in the top right of the list; tick the fields to show; save. Closing the window without saving discards the change | [P customizing-the-columns] |
| Which objects' fields? | **Deal, person and organization fields** are available as columns on the deal list. Activity or product fields as list columns are not mentioned | [P customizing-the-columns]; activity/product columns [unverified] |
| Where does a new column land? | At the right end of the existing columns | [P] |
| Custom fields | Appear in the same picker once created; a system ID column is added the same way by searching it | [Ps], [P custom-fields: "optional columns in list views"] |
| Does the picker group by object? | Not stated. Pipedrive's Insights tables mark linked-object columns with an **entity icon in the header** | grouping [unverified]; icon [P insights-combined-data] |
| How are removals done? | The article does not describe removal; untick in the same picker | [Ps] |
| Column limit | Not stated | [unverified] |
| Which plan? | Number of custom fields depends on plan; the column picker itself is not plan-gated in anything I read | [P custom-fields] |

**Cross-object columns are the 2015 headline of the redesign:** a deal list can show the postal address of the deal's organisation, which the old lists could not, and the author's own use case is building a direct-mail list **[P new-list-views blog]**.

**Insights reports** (Pipedrive's "reports") use the same idea more formally. Each report has a main item and a stated set of linked items it may pull from (deal reports: people, organisations, leads, activities; activity reports: people, organisations, leads, deals, projects; and so on). Linked fields work in report **filters** and in the **table view**; the chart still measures the main item and linked items only group it. Limits: activity columns only in the activities-performance report; the organisation report cannot show contact-person columns; custom fields as table columns on any plan but in filters and the visual builder only on Premium or higher. **How one-to-many links (a deal with several contacts or products) are shown is not stated** **[P insights-combined-data]**. Cross-object fields in filters were released for list views first, then for Insights (Professional plan and above) **[P insights cross-entity blog]**.

### 1.3 Order, width, pin
- **Reorder:** press and hold a column title and drag **[P customizing-the-columns]**; the 2015 post calls drag-to-order new **[P blog]**.
- **Resize:** hover the divider between cells until the resize cursor shows, drag **[P]**.
- **Pin or freeze:** **not found** in any Pipedrive page I fetched or searched **[unverified]**. (Zoho documents up to two pinned columns **[Z]**.)

### 1.4 Per-user versus shared
- Columns can be **saved together with a filter** (a tick-box in the filter's edit window). Sharing that filter shares its columns, and that is the documented way to give every user the same default columns **[P how-can-i-set-default-columns]**.
- Whether a user's own gear-icon choice is private to that user, and whether it survives a reload, is not stated **[unverified]**. I read it as per user per list **[inferred]**, because each user sees their own selection.
- The article does not say who may set shared defaults (admin versus any user) or how a user's changes interact with a shared layout **[P, unstated]**.

### 1.5 The quick filter bar
| Question | What the docs say | Tag |
|---|---|---|
| Where is it? | A bar **between the view toolbar and the data table**, on the deals list (the owner calls it "top-left"; the bar is left-aligned under the toolbar) | [P quick-filters]; "top-left" [owner] |
| Empty state | A single add-condition button | [P] |
| Building a condition | Click add; choose a **field** (type to search or scroll; "most fields" are offered); give a **value** (a dropdown of options or a text box by field type); optionally change the **operator** | [P] |
| Operators | By field type. Default "is" or "is any of"; text adds "contains", numbers "is more than", dates "is before" | [P] |
| After the first condition | The button shrinks to a funnel icon that adds more. Click a condition to change value or operator; X removes it | [P] |
| Header shortcut | **Hover a column header and click the funnel** to add a condition for that column | [P] |
| Limit | **Ten** conditions at once | [P] |
| Sharing | The browser URL carries the filter; a colleague with the right visibility sees the same rows. Bookmarkable | [P] |
| Save it? | Not stated whether a quick filter becomes a named filter | [unverified] |
| Who picks the fields? | Not stated whether admins or users can pin or reorder the offered fields | [unverified] |

### 1.6 Saved (advanced) filters
- **Where:** the filter dropdown at the top right; an add-new-filter entry; add conditions, optionally name it and set who sees it, save **[P filtering]**.
- **Logic:** an **ALL** group (every condition must match) and an **ANY** group (at least one), used together; the help's example shows two conflicting owner conditions in ALL returning nothing, and the same two in ANY returning both owners' records **[P filtering]**. In the API the structure is stricter: one top-level AND group, one AND and one OR group beneath it, nothing else yet **[P developers Filters]**.
- **Operators (API):** is null, is not null, equal, not equal, the four comparisons, and pattern matching (starts with, contains, does not start with). Ends-with and the negated contains/ends-with forms are listed as deprecated **[P Filters API; deprecation line [Ps]]**. The UI shows more friendly labels per field type **[P quick-filters]**.
- **Limits:** **16 conditions** per filter, and fewer is advised for speed in large accounts; a cap of **5,000 filters** per user is stated **[P filtering]**.
- **Visibility:** **private** (only the creator; not even admins) or **shared** (everyone in the company); changed in the edit window **[P filtering]**.
- **Favourites:** a star in the dropdown puts a filter on a Favorites tab; leads can also be saved as favourites **[P filtering, P leads-inbox]**.
- **Edit, delete:** a pencil icon edits or deletes; clearing is an X beside the dropdown **[P]**. Duplicating is not mentioned **[unverified]**.
- **Cross-object conditions:** list-view filters can use fields of the linked person or organisation (the blog says they arrived "earlier in the year", no date) **[P Insights blog]**; the operator-per-type table is not in any help page I read **[unverified]**.
- **Personal filters work in any view** (the 2015 post's wording) **[P blog]**.

### 1.7 Column headers
- **Sort:** click a header to sort; **Shift+click** another header to add a secondary sort; a secondary sort only changes order where the first column ties. Works on deal and contact lists, not on the Projects list or the Leads Inbox. Sorting covers **all** rows, not only the loaded ones **[P sorting-by-multiple-columns, P blog]**. Whether the sort is remembered, how many levels, and whether an arrow shows are not stated **[unverified]**.
- **Filter this column:** the hover funnel above **[P quick-filters]**.
- **Hide or pin from the header:** not documented **[unverified]**.

### 1.8 Selection, bulk actions, inline edit
- **Selecting rows opens a bulk panel**: choose a field, set a new value, confirm. Editable fields vary by type (deals: pipeline and stage, status, organisation; people: name, email, phone; leads: title, labels, value, owner...). Bulk edit can change fields **not shown** in the list. **How rows are selected (checkboxes, select-all, across pages) is not documented** **[P bulk-editing, P blog]**.
- **Other bulk actions by type:** delete (all types), group email, schedule activities (up to **100** at once, plan-gated), convert lead to deal and back, archive, enrich, add to sequences, add products to a deal **[P bulk-editing]**.
- **Safety:** a bulk edit can fire automations; only global admins see the warning and may skip them; **changes cannot be undone after confirmation** **[P]**.
- **Inline edit:** hover a cell, a pencil appears; not every field is editable; on a deal list, changing the linked organisation or person changes the link only, not the other record **[P list-view]**.

### 1.9 Export
- With a filter applied, pick columns with the gear, then the "..." menu's export-filter-results entry: it exports the **displayed fields**, and on the deal list can mix deal, person and organisation fields **[P exporting-data]**. The file prompts a download; closing the prompt leaves it under generated exports for **28 days**. CSV or Excel. Needs the right permission set and covers only rows visible to the user. No row limit stated **[P]**.

### 1.10 Totals
- **No footer or per-column total row was found** **[unverified]**. The deal list has a summary box behind an **"i" icon** next to the sync button: total value, count, value by currency and a probability-weighted total, **following the current filter** **[P how-can-i-see-the-total-value]**.

### 1.11 Performance and limits (everything stated)
Fewer filter conditions load faster in big accounts **[P filtering]**; scrolling loads more rows automatically once a list passes 100 and sorting covers every row **[P blog, 2015]**; a refresh button reloads without a full page refresh **[P list-view]**; bulk edits of many records can trigger a performance warning **[P bulk-editing]**. **No row limits, column limits or query-time limits are published** **[unverified]**.

### 1.12 What I did not find
Pinned columns; a header menu with hide or pin; column choice private versus shared; a footer total; how to select rows; multi-sort persistence; any list-row keyboard navigation; activity or product columns on a deal list; operator tables for cross-object filter conditions.

---

## 2. What the owner said, and what the docs confirm

> "I like the UX of PipeDrive CRM, which gives us a feature where we can add multiple columns in a report from a particular object. Also there is a quick filter on the top-left side, which is very dynamic and easy to implement. In columns there are options like sorting and filtration, which is also very good. Research once if we can add UX and features of PipeDrive in the reports of this Comsky books." **[owner, verbatim]**

| Owner said | Docs say | Gap or note |
|---|---|---|
| Add multiple columns from a particular object | Deal list can show deal, person and organisation fields; custom fields too; Insights table has a stated link map per report | **Confirmed.** Pipedrive's objects are CRM objects; ours are voucher, party, item, bill (section 4) |
| Quick filter, top-left, dynamic, easy | A bar under the toolbar: add condition, pick field (searchable), pick value, optional operator; funnel to add more; click a condition to edit; header funnel; 10 conditions | **Confirmed**, location slightly different ("under the toolbar, left-aligned"). Wave A put our filter bar in the same place (`belowHead`) |
| Columns have sorting and filtration | Click sort, Shift+click secondary, hover funnel filters that column | **Confirmed.** Hide and pin from the header: **not documented** |

---

## 3. Where we stand: what Wave A already covers, and the delta

| Capability | Wave A (spec, in progress) | This research adds |
|---|---|---|
| Row filter engine | `core/reportFilter.ts`: 4 kinds, AND/OR groups, depth 2, 12 conditions, sanitiser, canonical form [C] | Fields come from a **catalogue** that also owns columns; add **is empty / is not empty** and **is any of** (section 8.5) |
| Filter UI | A **Filter button** opens a modal panel (rows of field, operator, value); removable chips; `Alt+F12` range, `Ctrl+F12` value [spec 6.4] | An **inline popover builder** (type to search fields grouped by object, value in place, Enter commits), **chips editable in place**, a **header funnel**, smart typing of a bare value. The modal stays as **Advanced** for OR groups |
| Saved filters | Saved views carry `filter`, personal and company scope, favourites = pins [spec 6.5, 6.6] | Views also carry the **layout** (columns, order, widths, sort) |
| Columns | Only the statements have added columns (time columns: as-on and period), device-local in `tw-report-cols-v1` [C `ui/reportColumns.tsx`]; the registers have one F12 "narration" toggle | **Attribute columns from related objects** on list reports; the time columns stay exactly as they are |
| Header | `th` click sort (`useColumnSort`), `Alt+S` cycles [C] | A per-column **caret menu**, drag reorder, resize, Shift+click secondary sort |
| Flat register list | `all: true` level, `By month / All vouchers` toggle [spec 6.7] | It is the landing place for the new columns |
| Totals | Computed over filtered rows, label gets " (filtered)" [spec 6.3] | Per-column total kinds; the same rule extended to added columns |

Nothing in this document asks Wave A to change a shipped meaning of a key; section 8.4 lists the one place where a Wave A chord opens a lighter surface.

---

## 4. Which of our reports are lists of objects

A column from a related object is meaningful only when **each row is one object and has a single related object of each kind**. Classes: **A** = list of objects (catalogue-ready), **B** = list with statutory fixed columns (filter and sort yes, columns no), **C** = aggregate or matrix (keep its own columns).

| Report (F12 key) | Row object | Single-valued relations | Class | Today |
|---|---|---|---|---|
| Day Book | voucher | party (lead ledger), voucher type | A | Filter fields wired; 4 to 7 columns hard-coded [C `ui/reports.tsx` DayBook] |
| Sales, Purchase, Journal, Debit and Credit Note registers, flat "All" level | voucher | party, voucher type | A | Filter wired; month matrix is a different (aggregate) level |
| Memorandum, Optional, Post-dated registers | voucher | party | A | Day-Book-shaped [C `ui/reports/vchregisters.tsx`] |
| Ledger Vouchers | voucher as seen by one ledger (Dr, Cr) | party, type | A | Running balance column is order-dependent (section 7.4) |
| Bills Receivable, Bills Payable | bill (`BillState`) | party ledger | A | Filter wired [C `ui/reports/outstanding.tsx`]; bill has no originating-voucher id |
| Receivables ledger-wise view, Ageing, Reminder Letters, Interest | party | party master | A (partial) | Row is a party with computed buckets |
| Stock Summary, Godown-wise Stock, Reorder Status, Movement | stock item (plus godown) | group, category, unit | A (hierarchical for Stock Summary) | Has as-on/period time columns already [C `savedViews.ts` STATIC] |
| Cancelled and Deleted Vouchers, Edit Log | audit row | voucher, operator | A | Reads the outbox |
| Post-dated cheques, Deposit Slip, Payment Advice | instrument row | party, bank ledger | A | Small |
| Leads, Follow-ups | lead | ledger when converted | A | Native Pipedrive-like list; its columns are fixed [C `ui/crm/leads.tsx`] |
| **Ledgers, Stock Items (masters)** | ledger, item | group, unit, category | **A, but no table today** | `PickMaster` pick list, name plus sub-label only [C `ui/masters.tsx`] |
| GSTR-1 tables, GSTR-3B, HSN summary, TDS 26Q, Form 16A, e-invoice list | invoice or deductee row | party | **B** | Columns are fixed by the portal or form |
| Balance Sheet, P&L, Trial Balance, Group Summary, Cash Flow, Cost Centre Analysis, Payroll sheets | account, group | n/a | **C** | Keep the existing time columns (`Alt+C`) |
| Bank Reconciliation (BRS) | (voucher, entry) pair | party | A **but excluded** | Hard-coded `colSpan` 8 and 5, inline bank-date editor indexes entries [C `ui/banking/brs.tsx`] |

**Rule: one-to-many never makes a column.** A voucher has many item lines, cost centres and bill references. They appear as **scalar projections** (count, first, main, "Multiple") or not at all; they never repeat the row (section 7.5). A line-level list (one row per item line, with voucher and party as related objects) is a **different report** (an item-wise register), which is where Pipedrive's Products list maps (phase P6).

---

## 5. The field catalogue (the core design)

### 5.1 One accessor, five consumers
Today a report writes "party name of a voucher" up to three times: in its filter fields, in its `useColumnSort` list and in its JSX (Day Book `leadName`, register `partyName`, bills `r.ledger.name`). A **column definition** is the single owner of that accessor, read by: the **filter** (value to compare), the **sort** (order), the **cell** (what is drawn), the **total** (what a footer adds up), the **export** (what a CSV row says).

```ts
// src/core/reportFields.ts  (pure; suggested name)
export type Group = 'self' | 'party' | 'item' | 'voucher' | 'calc' | 'custom';

export interface ColumnDef<R, X> extends FieldMeta {            // FieldMeta = Wave A: id, label, kind, options?, money?
  group: Group;                  // picker section and header glyph; 'party' labels read "Party GSTIN", never "GSTIN"
  get(row: R, x: X): string | number | null | undefined;        // filter value; sort value unless sortKey is given
  sortKey?(row: R, x: X): string | number | null | undefined;   // when the shown text orders differently (Vch Type shows "Sales - Export", filters on base)
  text?(row: R, x: X): string;   // shown + exported text; default derives from kind and fmt (fmtAmt, fmtDate, fmtQty)
  cell?(row: R, x: X): React.ReactNode;                          // rich cell (VtBadge); must render the same text as text()
  align?: 'l' | 'r';
  width?: number;                // px, screen only
  total?: 'sum' | 'none';        // 'sum' ONLY for an additive measure of the row's own object (7.1)
  truth?: 'posting' | 'now';     // 'now' = a master attribute read at report time; the label says "(current)" (7.6)
  standard?: boolean;            // in the Standard column set (today's table)
  locked?: boolean;              // cannot be hidden or moved (the identity column)
  f12Key?: string;               // existing F12 toggle this column writes through (narration)
  needs?: readonly string[];     // indexes it requires: 'ledgersById', 'bills', 'einvoice', 'partyStats' (built lazily, 5.2)
  since: 1;                      // ids are saved inside views and are NEVER renamed (Wave A rule)
}

export interface ColumnSet<R, X> {
  reportKey: string;             // the F12 key the views and filter are kept under
  rowId(row: R): string;
  columns: readonly ColumnDef<R, X>[];
  groups: readonly Group[];      // which relations this report may offer (the Insights-style link map, section 4)
}
```

- **The filter fields are the catalogue**: `filterFieldsFor(reportKey)` becomes `columns.filter(c => c.kind)`. Wave A's ids (`date, party, type, number, amount, narration, debit, credit, ref, due, overdue, ageing`) keep their meaning, so views saved by Wave A stay valid (section 13, test 2).
- **A filter may name a field that is not a visible column** (as in Pipedrive and Zoho), and a hidden column never changes which rows are shown.
- **Dependencies**: `X` is a context built once per render and memoised on the store arrays: `ledgersById`, `stockById`, `groupsById`, `voucherTypesById`, `today`, `periodTo`, plus lazy indexes. This removes an existing O(rows x ledgers) scan: the Day Book calls `st.ledgers.find` per row [C `reports.tsx` `leadName`], which the perf harness already records as `rpt.daybook.find` [C `scripts/perf/run.mjs:278`].

### 5.2 Lazy indexes (so a column costs nothing until it is shown, filtered or sorted)
| Index | Built from | Used by | One pass over |
|---|---|---|---|
| `ledgersById`, `stockById`, `groupsById` | store arrays | party/item attribute columns | masters |
| `bills` (`outstandingBills(...)` by bill name) | existing engine | Status, Due, Balance, Days overdue on vouchers | vouchers once |
| `einvoice` (`indexRecords`) | `core/einvoiceState.ts` (used by the register selection already) | IRN, e-way | records |
| `partyStats` | one pass: last voucher date, voucher count, period sales or purchases per ledger | Party list, "Last transaction" on bills | vouchers once |
| `closing` (`closingBalances`) | `core/accounting.ts` | Party closing balance, credit used % | vouchers once |
Each index is a pure function in `src/core`, built the first time a column that `needs` it is **visible, filtered or sorted**, and asserted linear.

### 5.3 The layout (what a user chooses)
```ts
// src/core/listLayout.ts  (pure; sanitiser and resolver here, UI elsewhere)
export interface ListLayout {
  v: 1;
  ids: string[];                    // column ids in order; absent or [] = the Standard set
  w?: Record<string, number>;       // widths, 48..600, screen only
  pin?: string[];                   // up to 2, only leading columns; phase P5
  sort?: { id: string; dir: 'asc' | 'desc' }[];   // up to 2 keys: primary, secondary
}
```
- **Named `layout`, not `cols`**, because `SavedView.cols` already means the statements' time columns [C `core/savedViews.ts`].
- **Resolve:** `resolveColumns(set, layout)` drops unknown ids (a deleted custom field) and duplicates, caps at 20, forces `locked` columns first and in place, falls back to Standard when nothing survives. Dropping a column can never change rows or totals, so unlike a filter condition it is safe to drop.
- **Sort lives in the layout** so "overdue, biggest first" is a view. Today `useColumnSort` keeps its state in `useState` only; it must move to a live store keyed like the row filter [C `ui/rowFilter.ts` `liveKey`]. A sort on a column that is no longer shown is dropped (existing rule in `columnSort.ts`).

---

## 6. Field catalogues per report

Notation: **[std]** is in today's table (the Standard layout); **sum** = additive total allowed; **now** = master attribute at report time. Fields without a source note read the row's own object.

### 6.1 Voucher reports (Day Book, registers flat level, Memorandum/Optional/Post-dated, Ledger Vouchers)
| Group | Columns |
|---|---|
| This report (self) | Date [std, locked], Particulars [std, locked], Vch Type [std], Vch No. [std], Amount [std, sum], Debit and Credit [std on Day Book], Narration (writes the F12 narration toggle), Ref. no. (`refNo`), Supplier inv. no. and date, Place of supply, Reverse charge (Yes/No), Status (Regular, Optional, Post-dated, Memorandum, read from the flags), Instrument no. and type (first entry that has one), Order ref., Price level |
| Voucher (compliance) | IRN and Ack no. (`einvoice` index), E-way bill no., Export: shipping bill no. |
| Item summary (one-to-many collapsed) | Items (count), First item, Main HSN (the highest-value line; **reads the line's own snapshot**, section 7.6), Total qty, Godown (the one, or "Multiple"), Cost centre (the one, or "Multiple") |
| Calculated | Taxable value, GST amount (sum of heads from `voucherGstSummary`), GST rate (the one slab, or "Mixed"; never a blended rate [C CLAUDE.md]), **Due date**, **Paid**, **Balance** and **Status of the invoice** (Open, Part-paid, Settled, Overdue; from the `bills` index; with bill-wise off these four are blank and the picker says so rather than hiding them), Days overdue, Age (days since voucher date), Month, Quarter |
| Party (one) | Party state (current), GSTIN, GST type, City, Pincode, Phone, Email, Contact, PAN, Credit limit, Credit period, Ledger group, TDS deductee type |
| Party custom fields | one column per `CustomFieldDef` of kind `ledger` (id `udf:ledger:<key>`; type text, number, date, select = enum, toggle = Yes/No) |

Notes: **Amount stays the existing function** (`registerAmount` / `voucherHeadlineAmount`: the party leg, not the sum of debits, per the Phase 20-21 gotcha), imported not re-derived (section 13, test 4). The Day Book's Amount has no footer today ("the Day Book has no totals" [C `core/reportRows.ts`]); adding a column does not add a footer to a report that has none.

### 6.2 Bills (Receivables, Payables)
Self: Bill date [std], Ref. no. [std], Pending amount [std, sum], Original bill amount (sum), Due on [std], Days overdue [std], Ageing bucket [std, enum]. Party: all the party fields in 6.1, **Party closing balance** (`closing` index, not summable), **Credit used %** (closing balance over credit limit; blank without a limit), **Last transaction date** (`partyStats`). Calculated: **Originating voucher no.** (needs an index from bill name to voucher; `BillState` has no voucher id today [C `core/billwise.ts`], so this is a small engine addition, phase P4).

### 6.3 Stock Summary and other item reports
Self: opening, inwards, outwards, closing quantity, rate and value (the existing F12 flags), plus today's as-on and period columns (unchanged, section 3). Item (one): Part no., Alias, HSN (now), GST rate (now), Taxability, Stock group, Category, Unit, Reorder level, Standard sell and cost rate, Costing method, Batches (Yes/No), Service (Yes/No), item custom fields (`udf:stockItem:<key>`). Calculated: Below reorder (Yes/No), Days of stock (needs the movement window; spike). Group rows (rank 0 in the sort) show blank attribute cells.

### 6.4 The new object lists
- **Parties** (ledgers under any group; opens from a rail row): Name [locked], Group, State, GSTIN, GST type, City, Phone, Email, Contact, PAN, Credit limit, Credit period, Closing balance (Dr or Cr, `closing`), Overdue amount and Oldest overdue days (`bills`), Last transaction (`partyStats`), Vouchers in period, Sales or Purchases in period, custom fields. Row Enter opens Ledger Vouchers (Tally behaviour: select, drill); `Alt+A`-style alter stays where it is today.
- **Items**: Name [locked], Part no., Group, Category, Unit, HSN, GST rate, Closing qty and value, Reorder level, Last sale date, custom fields.
- These are **new screens** (new-screen checklist in CLAUDE.md applies: `Screen` union, router, crumb, palette, GoTo, `SCREEN_AREA` in `core/access.ts` as `masters`). They are lists, so Pipedrive's **inline edit** and **bulk edit** have a home here later (section 10).

### 6.5 What each field costs
Every column declares `needs`. Scalar reads from the row's own object are O(1). A relation read is O(1) through `X`. An index column costs one linear pass the first time it is used. Anything that would need a scan per row (per-voucher margin; days of stock) is **not offered until it has an index and a test**: per-voucher margin needs the valuation engine's per-line cost, which I did not verify is exposed per voucher **[unverified]**.

---

## 7. Adding columns without breaking anything

### 7.1 Totals
- **Footers are computed from the filtered rows by the report, never from printed text** (Wave A rule, spec 6.3). A column contributes to a footer only when `total: 'sum'`.
- **`sum` is allowed only for an additive measure of the row's own object**: Amount, Debit, Credit, Pending amount, Quantity, Taxable value, GST amount. It is **never** allowed on a column read through a relation (credit limit, closing balance, party state), because summing a party's credit limit once per voucher is a number that looks right and means nothing. A non-additive cell in a total row is blank, never 0.
- **Sort, hide, reorder and resize never change a total.** Filter does, and says so: " (filtered)" in the label, the sub-line and the badge.
- **A report with no footer gets none** because a column was added.
- **Percent columns** (`% of total`) divide by the filtered total, as Wave A decided.
- **Selection summary (stretch, P6):** with rows marked (Space, Shift+Down), the selection bar can show "3 selected, total of the sorted money column", the way a spreadsheet status bar does. This is our answer to Pipedrive's summary box. It counts only rows on screen (`visibleIds` rule in `ui/selection.ts`).
- **The total row's label stays in column 0**, which is why column 0 is locked: `labelsATotal(firstCell)` and the text filter's "never hide a total" backstop read the first cell's text [C `ui/useReportFilter.ts`]. A numeric column dragged to the front would put "Total (filtered)" under an amount.

### 7.2 Print and Export
- Both read `table.tt` in the DOM [C `ui/reportTools.ts`]. So: **a hidden column is not rendered** (no `display: none`), a shown column prints and exports with no further work, and a filtered-out row is not in the DOM (Wave A already relies on this).
- **Related-object headers carry the object in text**, "Party GSTIN", not a glyph alone, so paper and CSV are unambiguous.
- **Widths are screen-only.** The print clone copies inline styles, so widths are applied through a `colgroup` marked for removal in the clone (the print engine then uses `fitPlan`'s natural widths, [C `ui/print/fit.ts`]).
- **How wide is too wide:** `fitPlan` already chooses portrait, landscape or a scaled landscape (floor zoom 0.4, legible above 0.7). The picker shows a live hint ("prints landscape at 78%") computed from the same function.
- **Known weakness this makes worse:** CSV export is the cell's display text, so a money cell leaves as "1,23,456.00" and a date as "5-Apr-26" (text in a spreadsheet). With 15 columns that matters more. A model-driven export (raw numbers, ISO dates) is a phase P5 item. There is no spreadsheet library in `package.json` (grep for xlsx, exceljs, sheetjs, papaparse found nothing), so the first step is a better CSV; real `.xlsx` is a dependency decision.

### 7.3 Drill-down, selection, Mark Optional
- Rows are unchanged by columns, so `useRowNav(rows.length ...)`, Enter to open (`voucherScreenFor(rows[i])`), `Alt+2` duplicate, `Alt+D` delete and the selection (a **set of voucher ids**, intersected with visible ids [C `ui/selection.ts`]) behave exactly as now. Selection, Print selected and Mark Optional find rows by `data-vid` on the `tr`, which the shared table keeps.
- **A cell link (party) is the one new click target:** ctrl-click or the cell's own button opens Ledger Vouchers for that party; row Enter still opens the voucher. Pipedrive does the same with a summary card on hover **[P list-view]**; a hover card is an optional S item.
- **Mark Optional** flips one flag by id inside one undoable transaction [C `ui/reports/markOptional.ts`]; it is untouched. A **Status** column reads `optional`/`postDated`/`base` directly, so the Optional and Post-dated registers (which deliberately do not exclude optional vouchers) are not affected by the guard.
- **BRS is excluded**: its `colSpan` arithmetic and its inline bank-date editor depend on fixed columns and on a (voucher, entry) index [C `ui/banking/brs.tsx`]. It joins only after it is rebuilt on the shared table with an explicit `rowId`.

### 7.4 Order-dependent columns
Anything computed from row order (a running balance, a cumulative %) must be keyed **by row id from an unfiltered pass**, never recomputed from the displayed order. Ledger Vouchers already does this (`ledgerSplit().running` is a map by voucher id over the whole period, and `Opening + shown + not-shown = Closing` always [C `core/reportRows.ts`]). A new column of this kind that is not keyed by id is rejected by the catalogue test.

### 7.5 One-to-many
No column may make a row repeat. Projections only (count, first, main, "Multiple"), each with a label that says what it is ("First item", "Main HSN"). A user who wants the lines opens the item-wise register (P6). Pipedrive's own docs do not say how they handle this **[P insights-combined-data]**, which is a reason to be explicit.

### 7.6 Snapshot versus current (the GST gotcha, applied)
CLAUDE.md records that editing an item's GST rate silently re-taxed every historical invoice until the tax treatment was **snapshotted on the line** (`InventoryLine.gstRate/taxability/hsn`). The same trap applies to columns:
- A column on a voucher report that reads the **posting-time fact** (HSN and GST rate from the line; place of supply from the voucher) is `truth: 'posting'`.
- A column that reads the **master now** (party state, GSTIN, credit limit, item group) is `truth: 'now'` and the picker and the header say "(current)", so nobody reads next year's state against last year's invoice.
- The catalogue test fails a `now` column whose label lacks the marker.

### 7.7 Keys, drill-through and the typed filter
- **The `/` and `Ctrl+F` text filter** works on rendered rows and rendered text, so a shown related column becomes searchable (a feature) and a hidden one is not. Its "totals unfiltered" note is unchanged and the structured filter's totals still win when both are on (Wave A).
- **`Alt+S`** keeps stepping through **visible** sortable columns only (existing rule), so a sort by an invisible column cannot exist.

---

## 8. Interaction design

### 8.1 The page row (Wave A's three slots, extended)
```
[title] [All Sales Register ▾ views]                      [+ New ▾] [...]            [company]
[+ Filter] [Party contains sharma ×] [Amount ≥ 1,00,000 ×] [Clear] [12 of 340] [Columns ▦]
 Date ▾   Particulars ▾   Vch No. ▾   Party GSTIN ▾   Amount ▾ ...                      (header: caret on hover/focus)
```
Only the **Columns ▦** button is new in the bar; `+ Filter` replaces Wave A's "Filter" button with the same position and the same funnel glyph. Standard layout renders **exactly today's table**.

### 8.2 The Columns panel (`Alt+C`, or the button, or `...` menu)
A modal in the F12 family [C `ui/reportConfig.tsx` `F12ConfigPanel`; match its conventions].
- **Left:** a search box (focused on open) over the report's catalogue, grouped by object (This report, Party, Item, Voucher, Calculated, Your custom fields); each row: checkbox, label, kind icon, "(current)" when `truth: 'now'`. **Right:** "Shown (n)" in order with drag handles.
- **Keys inside the panel (it owns the keyboard while open, via a `useKeys` handler mounted only while open, never toggling a global handler's `active`):** type to search; `Up/Down` move; `Space` shows or hides; `S` sorts by the highlighted field (press again to reverse, a third time to clear); `F` opens the filter popover on it; `Alt+Up` and `Alt+Down` move a shown column (builder to verify against `keys.ts`; not `Alt+Left/Right`, which browsers use for Back); `R` resets to Standard; `Enter` or `Esc` closes. Everything the header menu can do is here, so **no mouse is ever required**.
- **Chord:** **`Alt+C`**. It already means "New Column" on the statements [C `ui/reportColumns.tsx` header], so on a list report it extends that meaning to "choose columns"; on a statement it keeps opening the time-column prompt and the picker also offers that under "Compare periods". Bare `C` is **not** claimed on lists (Space, P, E, O, R already act on a marked selection, and `/` types into the filter). Fallback if a platform eats `Alt+C`: **`Alt+M`** (zero uses in `src/`, to be verified). AltGr layouts reach the panel through the button and the `...` menu.
- **Footer:** "Standard" and "Reset", the print-fit hint, and a note when a hidden column is in the active sort or filter ("Sort by Party GSTIN is cleared" / "Filter on Party GSTIN stays on").
- **F-key rail:** gains "Alt+C Columns" on list reports only, truthful per the keycap rule.

### 8.3 The header menu (mouse and touch door onto the same actions)
On hover or focus of a header cell a small caret appears (never on touch-less print). Click or `Enter`:
Sort ascending, Sort descending, Clear sort; **Filter by this column** (opens the filter popover at the value step); Hide column (not for locked); Move left, Move right (the touch alternative to drag); Pin left (phase P5); Reset width. **Drag** a header to reorder, drag the divider to resize, double-click the divider to fit. **Shift+click** a header adds a secondary sort (Pipedrive's pattern **[P sorting]**); plain click keeps the existing three-state cycle (ascending, descending, back). The caret is a real `button` inside the `th` and stops propagation so a click on it does not also sort.

### 8.4 The quick filter bar (keyboard-first and mouse)
This replaces the **popover half** of Wave A's filter UI and keeps its panel as **Advanced**.
- **Open:** click `+ Filter`; or `Alt+F12` (range: opens on the first number or date field) / `Ctrl+F12` (value: first text or enum field), with exactly Wave A's meaning but landing in the popover. *This is the one Wave A behaviour to re-spec* (the spec says these open the modal panel); the builder of Wave A's `FilterBar.tsx` should be told before it is finished.
- **Step 1, field:** a search box over the catalogue's filterable fields grouped by object; type to narrow, `Up/Down`, `Enter`. **Smart value:** if the text matches no field label it offers "Particulars contains ..." and "Narration contains ..." for the report's text fields; `>1L`, `<=50k` and `1L-5L` offer an Amount range through `parseFilterNumber`; a date word offers a date range through the date mnemonics.
- **Step 2, value:** operator defaults by kind (contains, at least, between, is) and is a small dropdown; the value input is the kind's own (options list for enums with type-ahead, Indian shorthand for money, date mnemonics). `Enter` commits and closes and **returns focus to the table with the cursor on the same row**; `Shift+Enter` commits and opens another. `Esc` cancels.
- **Chips are editable:** click or `Enter` on a chip reopens the popover prefilled; `Delete` or the x removes it. Chips are real buttons in the Tab order, joined by "and". A group (made in Advanced) draws as one chip, "(A or B)".
- **Header funnel:** hover a column header, click the funnel (the Pipedrive gesture **[P quick-filters]**), or `F` in the Columns panel: opens step 2 for that field.
- **Count and limits:** "12 of 340" and the Filtered badge are Wave A's. The cap stays 12 conditions and depth 2 (Pipedrive: 10 quick, 16 advanced **[P]**); the popover says "A filter holds at most 12 conditions" when full.
- **Labels for the non-accountant:** Advanced reads "Match all / any" instead of "and / or" in its UI (the stored form is unchanged).
- **No URL sharing.** Pipedrive shares a filter through the browser URL **[P quick-filters]**; our data is local-first, so sharing is a **company view**, never a link to local rows.

### 8.5 Operators we are missing (additive, with a safety step)
Pipedrive's API has null checks **[P Filters]**, and for a ledger the single most useful questions are "GSTIN is empty", "HSN is empty", "Phone is empty". Add to every kind: **is empty / is not empty**; to enums: **is any of**.
**Safety first:** Wave A's reader keeps a condition with an unknown *field* as match-nothing, but a condition with an unknown *operator* is **dropped** (`cleanCondition` returns null when `validOp` fails [C `core/reportFilter.ts`]), which on an older client would silently **widen** an AND. In a company-shared view with mixed client versions that is the failure the module was written to avoid. So: **release 1** makes the v1 reader fail closed on an unknown operator (keep the condition, match nothing); **release 2** introduces the new operators.

### 8.6 Keyboard conflicts, audited
| Key | Existing meaning | This design |
|---|---|---|
| `Alt+C` | New Column (statements) | Columns panel on lists; unchanged on statements |
| `Alt+F12`, `Ctrl+F12` | Filter range, value (Wave A) | Same intent, popover not panel |
| `Alt+S` | Step sort through visible columns | Unchanged; adds secondary only via Shift+click |
| `Alt+D` | Delete voucher (lists), delete column (statements) | Unchanged; not used in the panel |
| `Alt+V` | Saved views | Unchanged |
| `Space`, `Shift+Down/Up`, `P`, `E`, `O`, `R` | Selection and its actions on registers | Unchanged; no bare letter added on lists |
| `/`, `Ctrl+F` | Type-to-filter | Unchanged |
| `Esc` | Closes the topmost thing | Popover and panel close first; the report's own Esc must not pop the screen while one is open |

---

## 9. Persistence and sharing

| What | Where | Why |
|---|---|---|
| Live layout (columns, order, widths, sort) | In memory per company and report **and** device-local `tw-report-layout-v1:<cid>` (the precedent of `tw-report-cols-v1`), every read and write in try/catch | A layout cannot hide a row or change a total, so unlike the live filter it is **safe to persist** (the live filter is deliberately never stored [C `ui/rowFilter.ts`]) |
| A saved view | `SavedViewStored.layout` beside `filter`, `config`, `cols`, `basis` | The view is the whole setup (Wave A rule); a view saved before layouts existed recalls **Standard**, as an older view recalls no filter |
| Personal versus company | Wave A's scopes: `me` (device) and `company` (synced, administrator-written, `Company.reportViews`) | Pipedrive's private versus shared **[P filtering]** maps to ours one for one |
| **Default columns for everyone** | A company view flagged `default: true` for a report; applied when the operator has no personal layout; a personal layout overrides | Pipedrive reaches this by sharing a filter that carries columns **[P default-columns]**; making it explicit avoids the trap that switching filters silently changes your columns |
| Favourites | The existing pin (`PINS_MAX` = 8) | One favourites list (Wave A) |

**Rules:** the sanitiser clamps ids to the safe pattern, caps columns at 20 and widths at 48..600, drops unknown ids, and treats a corrupt blob as Standard. A view applied while the report is not mounted goes through the bridge (`ui/reportViews/bridge.ts`) as the filter does. F12 toggles that already exist (narration, Dr/Cr shape) stay the **single source of truth** for those columns and the picker writes through them (`f12Key`); two controls with two states for one column would be a bug.

---

## 10. What not to copy, and why

| Pipedrive thing | Verdict | Reason |
|---|---|---|
| Pipeline kanban, deal stages, probability-weighted totals, forecast **[P]** | **Do not** | CRM-specific. We already have a lead pipeline (`ui/crm/pipeline.tsx`); it is not accounting |
| "Owner" as the universal filter | **Do not** | A voucher has no owner. The nearest, "entered by", lives in the outbox change log (a scan); keep it in the Edit Log, not in the catalogue |
| Bulk edit and bulk delete of records **[P]** | **Do not for vouchers** | Pipedrive says bulk edits cannot be undone **[P]**. Ours are books: the register selection allows only Print, Export, Mark Optional/Regular, each undoable in one step. A Parties list may later get bulk edit through one master-save path that spreads the existing doc (the Phase 24 gotcha: forms that rebuild the object drop fields) with operator rights and one undo step |
| Inline edit of any cell | **Not in MVP; lists of masters later** | Vouchers change on the voucher screen with audit. The BRS bank date is the one existing inline editor |
| Archive, convert, sequences, group email, enrich, schedule activities, Mailchimp, map **[P]** | **Do not** | CRM actions |
| Infinite scroll with "load more" **[P, 2015]** | **Do not as is** | A report must be deterministic and printable. Windowing is acceptable only behind model-driven print and export (R1) |
| Plan caps on filters and custom fields (5,000 filters; plan-tiered custom fields) **[P]** | **Do not** | Our plan limits are display-only today; a cap here is a new thing to explain. Keep only technical limits (12 conditions, 20 columns) |
| Sharing a filter by browser URL **[P]** | **Do not** | Local-first data; sharing = company view |
| ALL/ANY words **[P]** | **Borrow the wording in Advanced only** | Friendlier to the owner persona; stored form stays and/or |
| Pipedrive's lists being mouse-first **[Ps]** | **Do not** | Our value to the Tally switcher is the keyboard; every new feature has a keyboard-complete core |
| A free report builder (Zoho "Create New Report") | **Not needed** | Layout plus filter plus sort plus a name covers it (finding 10) |
| Quick filter restricted to the deal list **[P]** | n/a | Ours is generic once a report has a catalogue |

---

## 11. MVP, phases, effort, and which screens first

**Effort scale (one engineer, including assertions and a screen sweep) [J]:** **S** up to 2 days, **M** 3 to 6 days, **L** 1.5 to 3 weeks.

### 11.1 MVP (P0 to P3, on four report families)
**Day Book, the voucher registers' flat level, Ledger Vouchers, Receivables and Payables.** The MVP delivers: the catalogue and `ColumnDef` for those families; a shared table renderer; the Columns panel; the header menu (sort, filter, hide, reorder, resize); the quick filter popover with editable chips; layout in views and device storage; party fields, voucher fields, item summary, status/due/balance and custom fields as columns. **Standard layout identical to today.**
**Not in the MVP:** pin, windowing, model-driven export, inline or bulk edit, group-by, company default layout, the new Party and Item lists, Stock Summary attributes, line-level lists, BRS, statutory tables, mobile.

### 11.2 Phases
| Phase | What | Effort | Depends on | Exit test |
|---|---|---|---|---|
| **P0** | **Spike and groundwork.** Measure first paint and key response of a flat Day Book at 5k, 25k and 100k rows with 7 versus 15 columns in the headless harness; measure the catalogue accessors per row with the perf mirrors; decide windowing; fix the sticky-scroller trap for pin (`.hscroll` makes the element the containing scroller and killed sticky headers in 14 reports [C CLAUDE.md gotcha]); fail-closed unknown operator in the filter reader | S | none | A table of numbers in `docs/PERF.md`; R1 decided |
| **P1** | **Catalogue, context, shared renderer** (`core/reportFields.ts`, `core/reportFieldCtx.ts`, `ui/list/ListTable.tsx`): port Day Book, registers flat, Ledger Vouchers, Bills; replace the four hand-written tables; colSpans from the visible count; total row from `total` kinds | M | P0 | The Standard layout's table text and the 478-screen sweep are **unchanged**; assertion suites in section 13 |
| **P2** | **Layout, panel, header menu**: `core/listLayout.ts`, Columns panel, header caret, drag reorder, resize, Shift+click secondary sort, sort into the live store, device persistence, `layout` in saved views, print-fit hint | M | P1 | Recipes `columns-picker`, `header-menu`, `columns-print` in light and dark, compact and comfortable |
| **P3** | **Quick filter bar v2**: the popover builder, editable chips, header funnel, smart typing, "Match all / any" wording; operators `isEmpty`, `notEmpty`, `anyOf` (after the fail-closed release) | M | P1 (fields), Wave A FilterBar | Recipe `quickfilter-bar`; the Wave A suites still green |
| **P4** | **Object lists and more attributes**: Parties and Items lists (`partyStats`, `closing` indexes; custom fields), Stock Summary and Reorder attribute columns, originating-voucher index for bills | L | P1, P2 | Closing balance of a party list equals the Trial Balance's ledger balance; the Stock Summary closing still equals Stock-in-Hand |
| **P5** | **Scale and output**: windowed rendering, **model-driven Print, CSV and the `/` text filter**, raw-number CSV export, pin (up to two), company default layout (synced field, sanitiser, `masters`/admin rights) | L | P0 numbers, P2 | Same figures printed from a windowed list as from the full DOM list; 250k-voucher budget met |
| **P6** | **Extras**: selection total, party hover card, item-wise (line-level) register, Parties inline edit and bulk edit (one undo step each) | M each | P4 | Each behind its own assertion and operator-rights test |

**Effort of the MVP (P0 to P3): roughly 3 to 4 engineer-weeks [J].**

### 11.3 Report screens to cover first (in order)
1. **Day Book** (everyone opens it; voucher fields, party fields, status).
2. **Sales and Purchase registers, flat level** (the owner's "click Sales and see all sales"; invoice status, due and balance fix UX finding N2).
3. **Receivables and Payables** (party GSTIN, state, phone, credit limit, credit used %, last transaction: the collections workflow).
4. **Ledger Vouchers**.
5. **Parties list (new)**, then **Items list (new)** (the true Pipedrive People and Products).
6. **Stock Summary** attributes and **Reorder Status**.
7. **Leads** (already a native list; its columns are its own object plus linked ledger).
Not now: GSTR and TDS tables (class B), statements (class C), BRS.

---

## 12. Risks

| # | Risk | Evidence | Mitigation |
|---|---|---|---|
| **R1** | **Performance at 250,000 vouchers.** No windowing exists in `src/` (grep for virtual, windowing, overscan finds one unrelated hit), so a flat list renders every row, and the default period is the whole year. Columns multiply cells per row (arithmetic, not measured: a 60,000-row year at 7 columns is about 420,000 cells; at 15 columns about 900,000). The engines are gated (Balance Sheet and Stock Summary 1,200 ms at 250k [C `scripts/perf/budgets.mjs`]); **no row-render budget exists** | [C], arithmetic | P0 measures; hidden columns cost nothing (not rendered); accessors O(1) through `X`; lazy indexes; windowing in P5 **only together with** model-driven Print, Export and text filter, because all three read the DOM today |
| **R2** | **Totals semantics.** A summed related column looks plausible and is wrong | section 7.1 | `total: 'sum'` only for own additive measures; catalogue test fails any `sum` on a `party`, `item` or `calc` non-additive column; blank, never 0 |
| **R3** | **Keyboard conflicts.** `Alt+C` reuse; two Wave A chords changing surface; Esc stacking with popovers | section 8.6 | One owner for each key; popovers mount their key handler only while open; recipes press each chord on a list and on a statement |
| **R4** | **Hiding the identity column breaks the total row, the text filter and drill.** | `labelsATotal`, `namesATotal` read the first cell [C `ui/useReportFilter.ts`] | `locked` column 0, not movable or hideable; the total label always in it |
| **R5** | **Wave A is uncommitted and moving**; this design touches its filter UI | git status at session start | Tell the Wave A builders (section 8.4) before `FilterBar.tsx` is finished; the catalogue is an additive layer under `FieldDef`, not a rewrite |
| **R6** | **Saved views from mixed client versions** (company views synced) | `cleanCondition` drops unknown operators [C] | fail-closed reader first (8.5); `layout` is ignored by an older build, which then shows Standard, never a wrong table |
| **R7** | **Stale columns** (a deleted custom field; a renamed label) | Wave A id rule | ids never renamed; resolver drops unknown ids; a stale **filter** matches nothing (widening is the dangerous direction) |
| **R8** | **Print and CSV with many columns** | `fitPlan` floor zoom 0.4 [C]; CSV is display text | print-fit hint; "print standard columns" later; raw-number export in P5 |
| **R9** | **Sticky header and pin collide with `.hscroll`** | CLAUDE.md gotcha | one scroller with both axes; fix in P0, before pin |
| **R10** | **Snapshot versus current** | CLAUDE.md "snapshot tax treatment on the line" | `truth` metadata and the "(current)" label (7.6) |
| **R11** | **Three sources for one value** (filter, sort, cell) drifting | today's hand-written accessors | one `ColumnDef`; catalogue test compares each column's `get` with the report's own function (Amount) on every sample voucher |
| **R12** | **Double control** (F12 narration toggle and the picker) | section 9 | `f12Key` write-through; one state |
| **R13** | **Standard layout drifts from today's table** and a Tally switcher notices | the "frozen" rule | golden header-text and body-text test per report plus the 478-screen sweep |
| **R14** | **New object lists** cost an index pass each and a new `SCREEN_AREA` entry | `core/access.ts` rule | `partyStats` asserted linear; the security-access suite names the screens |

---

## 13. Assertions and recipes to add (names are suggestions)

**`scripts/assert/reportfields.assert.ts`** (pure)
1. Per report: ids unique; every column has a `get`; `locked` columns come first; `kind` present; `now` columns carry the "(current)" label; party labels are prefixed.
2. **Back-compat:** every id in Wave A's `DAYBOOK_FIELDS`, `REGISTER_FIELDS`, `LEDGER_VOUCHER_FIELDS`, `BILL_FIELDS` exists in the catalogue with the same kind, so a Wave A saved view still means what it did.
3. **Golden Standard layout:** headers and the first 20 rows' text per report equal today's tables on the sample company.
4. **Mirror guard:** the catalogue's Amount, Days overdue, Ageing getters equal the report's own functions (`registerAmount`, `daysOver`, `ageingBucket`) for every sample row.
5. **Totals:** the footer equals the sum of `sum` columns over the filtered rows; no `sum` on non-additive; blank cell, not 0; "(filtered)" label; Ledger Vouchers keep Opening + shown + not-shown = Closing.
6. **Rows never change with columns:** for random layouts, the output row ids and order (given the same sort) are identical.
7. **One-to-many:** no catalogue getter returns more than one value per row (projection kinds only); a property test builds a voucher with 5 lines and 3 cost centres and checks the row count.
8. **Order-independent columns:** running-balance style columns are keyed by id.
9. **Indexes linear:** `partyStats`, `closing`, `bills` on a 250k synthetic set stay within a stated ms budget (reference, not gated, like most perf rows).

**`scripts/assert/listlayout.assert.ts`:** sanitiser (corrupt blob, wrong types, duplicates, 21 columns, widths out of range, pin on a non-leading column, a sort naming a hidden column) and resolver (unknown id dropped, locked first, Standard fallback), plus a source scan that every storage access is in try/catch and that no **live filter** is written to storage.

**Extend:** `savedviews-filter.assert.ts` (a view with and without `layout`), `reportfilter.assert.ts` (new operators, and the fail-closed reader), `zoho-scaffold.assert.ts` (the `Alt+C` registry entry; keycap truthful on lists only), `visual.assert.ts` baselines (Standard unchanged).

**Recipes (`scripts/shots/recipes/columns.mjs`):** `columns-picker` (Day Book, `Alt+C`, search "gst", add Party GSTIN, close, the column is there, Print shows it, widths absent), `header-menu` (hover caret, sort, filter this column, hide, reorder by drag), `quickfilter-bar` (`+ Filter`, type "sharma", Enter; `Ctrl+F12`; edit a chip; Advanced for an OR), `columns-views` (save as company view, recall on a second profile), `columns-keyboard` (the same flow with no mouse). Each in light and dark, compact and comfortable, zero console errors.

---

## 14. Questions for the owner

1. **Is the first target the voucher lists (Day Book, registers, Receivables) or the Parties list?** I would ship the first and start the second in the next phase; the Parties list is where the Pipedrive feeling is strongest.
2. **Which party fields matter most on a list of invoices** (GSTIN, state, phone, credit limit, salesperson)? There is no salesperson on a voucher today; a custom voucher field would be new work.
3. **Should an administrator be able to set the default columns for a report for everyone** (company default layout, phase P5)?
4. **Is "Match all / any" wording acceptable** for the non-accountant persona in Advanced, with "and / or" kept elsewhere?
5. **Pinning:** Zoho has it, Pipedrive does not (as far as I could find). Worth the sticky-scroller work in P5?

---

## 15. Sources

All fetched 2026-10-20 unless marked **[Ps]** (read through a search summary only).
- Customizing the columns in the list view: https://support.pipedrive.com/en/article/customizing-the-columns-in-the-list-view
- List view: https://support.pipedrive.com/en/article/list-view
- Advanced filtering: https://support.pipedrive.com/en/article/filtering
- Quick filters: https://support.pipedrive.com/en/article/quick-filters
- List view, filter and bulk edit items: https://support.pipedrive.com/en/article/bulk-editing-and-filtering (older address: https://support.pipedrive.com/hc/en-us/articles/206529689-Bulk-Editing-and-Filtering)
- Sorting by multiple columns in list view: https://support.pipedrive.com/en/article/sorting-by-multiple-columns-in-list-view
- Default columns for all users: https://support.pipedrive.com/en/article/how-can-i-set-default-columns-for-all-my-users-in-the-list-view
- Exporting data from Pipedrive: https://support.pipedrive.com/en/article/exporting-data-from-pipedrive
- Total value of deals (summary box): https://support.pipedrive.com/en/article/how-can-i-see-the-total-value-of-my-deals-by-stage-or-pipeline
- Custom fields: https://support.pipedrive.com/en/article/custom-fields
- Leads Inbox: https://support.pipedrive.com/en/article/leads-inbox
- Filtering activities in the list view: https://support.pipedrive.com/en/article/how-can-i-filter-my-activities-in-the-list-view
- Insights, combine linked item data: https://support.pipedrive.com/en/article/insights-combined-data
- Insights cross-entity fields in filters (blog): https://www.pipedrive.com/en/blog/insights-cross-entity-fields-in-filters
- New list views (blog, 2015): https://www.pipedrive.com/en/blog/new-list-views-powerful-filtering-of-contacts-and-deals
- Filters API reference: https://developers.pipedrive.com/docs/api/v1/Filters
- **[Ps]** Keyboard shortcuts and interface: https://support.pipedrive.com/en/article/quick-actions-in-pipedrive
- **[Ps]** Insights table-view articles (deal performance, product reporting): https://support.pipedrive.com/en/article/insights-reports-deal-performance, https://support.pipedrive.com/en/article/insights-products
- Our own: `docs/design/zoho-ux-spec.md`, `docs/design/ux-analysis-2026-10.md`, `docs/research/zoho-books-reference.md`, `src/core/reportFilter.ts`, `src/core/reportFilterFields.ts`, `src/core/reportRows.ts`, `src/core/reportColumns.ts`, `src/core/savedViews.ts`, `src/ui/rowFilter.ts`, `src/ui/reports/columnSort.ts`, `src/ui/reports.tsx`, `src/ui/reports/outstanding.tsx`, `src/ui/reports/financial.tsx`, `src/ui/reportTools.ts`, `src/ui/selection.ts`, `src/ui/banking/brs.tsx`, `src/ui/masters.tsx`, `scripts/perf/budgets.mjs`.
