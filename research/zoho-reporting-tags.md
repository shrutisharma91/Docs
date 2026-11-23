# Zoho Books reporting tags, set against TallyPrime cost centres and Comsky's own

> **Status:** research and design input only. Nothing here is built. Written 2026-10-21 for the owner's custom-fields ask
> (an object-wise field with a type, mandatory or not, and a place to show; a colour on an item, a zone on a sales invoice;
> reports that can filter and group on it; and the "profitability tag" idea from Zoho).
> **Method:** Zoho's and Tally's own public help, knowledge-base, API and community pages, read through a web fetch that
> summarises each page (so a page's fine print can be missed), plus a reading of this repo's working tree. Every vendor fact is
> paraphrased; no vendor text is reproduced. There was no live Zoho Books, TallyPrime or Pipedrive account to try anything on.
> **Evidence tags:** **[W]** a vendor page fetched and read this session (URL in section 10) . **[Ws]** a web-search summary
> only, page not opened . **[C]** read in this repo's code or data (file named) . **[J]** my judgement . **[inferred]** my
> reading of what a page implies . **[unverified]** I looked and could not confirm it.

## 0. Read this first

**What a "reporting tag" is, in one paragraph.** It is a named dropdown (a name and a list of options, optionally nested) that an
administrator creates once, attaches to many kinds of record, and then picks a value of on each record or each line. It carries no
amount: a tagged line belongs wholly to its option. Its reason to exist is reporting: Zoho lets the Profit and Loss, Balance Sheet,
Trial Balance, Cash Flow and a few other reports be filtered by a tag, or laid out in one column per option **[W]**. A cost centre in
TallyPrime is the opposite shape: a piece of an amount, where one posting is split into pieces that must add up to it **[W]**.

**The eight findings that matter.**
1. **Zoho has two generations of the feature and its help pages disagree with each other.** The older pages (US and EU) describe
   tags on line items only, 10 tags and 500 options per organisation, and filtering by tag. The newer "advanced" pages add a
   transaction-or-line choice, mandatory tags, option hierarchy five levels deep, a default option, conditional options, access
   control by tag and a "compare by tag" layout. Enabling the advanced mode cannot be undone **[W]**. Section 1.1.
2. **A tag is a label, not an allocation.** The pages describe picking one option from a dropdown; none describes splitting an
   amount across options. Whether one tag can take several options on one line is not stated **[unverified]**. So a tag cannot say
   "35,000 of this expense: 20,000 to Anil, 15,000 to Ravi"; a cost centre can **[W, J]**.
3. **Where a tag is more reachable than a cost centre.** A tag can sit on items and on customers or vendors (as a default that is
   copied onto new lines), on quotes, orders and other non-posting documents, on payments, banking and fixed assets, and on both
   income and balance-sheet accounts' lines. A Tally cost centre exists only on an accounting posting and, by default, only on
   income and expense ledgers **[W]**.
4. **Where cost centres already beat tags.** Splitting by amount, parallel categories at 100% each, sub-centre roll-ups, a
   Centre x Ledger breakup, Tally's percentage "class" and (in Comsky) a Cost-Centre P&L Breakup. Zoho's tag reports are filters and
   compare columns on existing reports, not a breakup engine **[W, C]**.
5. **Neither vendor documents how a partly tagged amount reconciles to the books.** Zoho's pages say nothing about an untagged
   column or a total check (the API can list an "untagged" entry among options, and the filter for it is called "is empty") **[W]**.
   Tally's own job-costing guide says plainly that an unallocated entry is posted to its ledger but missing from the cost-centre
   reports, which understates job cost and overstates job profit **[W]**. Comsky's cost reports behave the same way today **[C]**.
   This is the design point that decides the whole feature (section 8.3).
6. **Zoho's own history shows the overlap between tags and custom fields.** Until mandatory tags were built in (a Zoho community
   post, undated, whose image names suggest about May 2023), the recommended workaround was a *required dropdown custom field* plus a
   workflow function that copied the choice into a tag, one tag per invoice **[W, unverified date]**. A tag is, in practice, a
   dropdown field with reporting powers.
7. **Comsky has real gaps in the thing it already has.** (a) A Sales or Purchase entered as an **item invoice**, which is the
   default mode, carries **no cost allocation at all**: the posting code for that mode builds its entries without `costs`, so a
   person must switch the voucher to accounting mode to allocate. (b) Tally's *parallel allocation* (the same amount allocated 100%
   under each cost category) is a partial row in `docs/parity-data.json`. (c) By reading the code, the Tally XML importer flattens
   every category of a posting into one list with no sum check, so a file that uses parallel allocation would import with the
   allocations adding up to a multiple of the line (section 6.2) **[C, J: read, not run]**.
8. **The owner's "colour" example hides a fork.** If colour is a property of an item (this item is red), the stock-item custom field
   that exists today already stores it and needs only to become a column, a filter and a group in item reports. If colour is chosen
   per sale (the same shirt, sold in three colours on one invoice), it is a *line-level* value with the item's colour as its default,
   which is exactly Zoho's line-level tag with an item default **[C, W, J]**. The owner should say which.

**What could not be checked.** No live account of any of the three products. The summarising fetch sometimes disagreed between two
renderings of the same Zoho page (flagged where it matters). Zoho's pages do not state: tags in multi-currency, tags on printed
documents, import or export of tag values, what happens to existing records when an option is deactivated, or a reconciliation or
"untagged" column in any report. Tally's cost-centre report pages were reachable only as an introduction with links; their bodies
were not read, so what each Tally report prints (and whether one prints a "Not Applicable" line) is **[unverified]**.

---

## 1. What a Zoho reporting tag is

### 1.1 Two generations of one feature

Zoho publishes the feature on several regional help sites, and the text differs. Reading them side by side:

| Point | Older pages (US, EU) **[W]** | Newer "advanced" pages (AU, MX) **[W]** |
|---|---|---|
| Switch | none | "Enable Advanced Reporting Tags" on the Reporting Tags screen; **cannot be turned off**; existing tags are upgraded and an organisation that never enables it keeps the old flow. The MX page calls the feature early access and plan-limited |
| Tag form | name, then options | name, optional description, which modules it applies to, transaction-or-line choice, mandatory switch |
| Level on a transaction | line items only (the page says tags go on line items, not on the whole transaction) | for Sales, Purchases, Journals and Inventory Adjustments the tag is set to **transaction level or line-item level**; other modules get no choice |
| Mandatory | not stated | yes: a "make mandatory" switch; a mandatory tag must have at least one active option |
| Options | a flat list, up to 500 per tag | nested up to five levels, reorderable, a default option, count depends on plan |
| Conditional options | none | an option can be shown only when chosen options of *earlier* tags (or a Location) are selected (5 filters per tag, 200 mapped options) |
| Access control | none | one tag per organisation can restrict which records a user sees, at role level and up to 500 options per user |
| Status | none | a new tag is saved "Not Ready" and must be marked "Ready" (needs at least one option) |
| Reports | pick tags under "Customize Report" in a Business Overview report | Advanced Filters (tag, comparator, option), "Compare Based on Reporting Tags", an optional parent-child layout, save as custom report |

I treat the advanced pages as the direction of travel and the older ones as the floor. Which one an individual Zoho organisation
sees cannot be told from the public pages **[unverified]**. One of the older-page renderings said tagged "additional information"
is chosen from a transaction's "More" menu while another said tags are line-item only; the two KB answers about detail reports (1.3)
agree with line-item only, and I rely on that.

There is **no feature called a "profitability tag"** in anything I read. The owner's description matches Reporting Tags. Zoho's own
example is a travel agency tagging records by destination and season to see which locations and seasons earn most; the profit
view is simply the Profit and Loss report filtered or compared by tag **[W, J]**.

### 1.2 Creating a tag

Path: Settings, Organisation Settings, Customisation, Reporting Tags, then the new-tag button **[W]**.

- **Name** (required) and, in the advanced form, an optional **description**.
- **Modules it applies to.** Older pages: 14 record types (contacts, items, quotes, sales orders, invoices, recurring invoices, credit
  notes, expenses, recurring expenses, purchase orders, bills, recurring bills, vendor credits, journal entries). Advanced pages: 22
  (those plus retainer invoices, customer and vendor payments, inventory adjustments, sales receipts, fixed assets and banking).
  Customers and vendors are listed as separate modules there **[W]**.
- **Level** (advanced only, and only for Sales, Purchases, Journals, Inventory Adjustments): once per transaction, or once per line.
  A tag can be attached to **Items** only if it is line-level in one of those four **[W]**.
- **Mandatory** (advanced only). Zoho's API exposes the same flag as `is_mandatory` **[W]**.
- **Options**, added by name, or above, below or under another option; reorder; delete; mark one as the **default**, which is
  pre-selected on new transactions that use the tag **[W]**.
- **Visibility conditions** (off by default): an option appears only when a mapped option of an earlier tag, or a Location, is
  selected. "All" maps to every option; "Untagged" lets the option show when the earlier tag has nothing selected. The order of tags
  matters, and a tag used as a filter cannot be dragged above the tag it filters **[W]**.
- **Access control**: switch on for at most one tag; role settings decide whether a role may see records outside the user's
  assigned options and records with no tag at all **[W]**.

### 1.3 How a tag gets onto a record

- **Items:** pick an option on the item; it is the default for that item and is copied onto a new line when the item is added, and
  the person can change it there. **Customers and vendors:** the same, as a default for their transactions **[W]**. Zoho's pages do
  not say which default wins when an item's and a contact's disagree **[unverified]**.
- **Transactions:** at the level the tag was set to: a dropdown on the form, or a dropdown in each line of the item table **[W]**.
  Expenses use their own place at the bottom of the form, and an itemised expense takes a tag per line **[Ws, release note]**.
- **Detail reports and the line-level fact.** Zoho's own answers to "why can't I filter my Invoice Details (or Bills Details) report by
  tag?" say the tag sits on the line items and not on the invoice or bill, so those two reports cannot be customised by it; the
  Receivable Details and Payable Details reports can group by tag and filter by it **[W]**.

### 1.4 Limits (what is stated)

| Limit | Value | Source |
|---|---|---|
| Tags per organisation | 10 (older pages) | [W] |
| Options per tag | 500 (older pages); "depends on the plan", no figure (advanced pages) | [W] |
| Option nesting | 5 levels; at most 500 child options under one option | [W] API |
| Conditional-option filters | 5 per tag, 200 mapped options | [W] |
| Access control | 1 tag per organisation; 500 options per user | [W] |
| Plan | not available on the Free plan; included from Standard upward; the comparison table has no row giving a tag count | [W] pricing comparison |

I found no per-plan tag count anywhere; the older pages' 10 and 500 are the only numbers.

### 1.5 Editing, deactivating, deleting

- **Edit** from the tag list; **reorder** with a "change order" drag (the order is used in transactions and reports) **[W]**.
- **Deactivate** a tag, and (older page's edit screen) **deactivate single options** by clearing a box; a tag that is already on
  records **cannot be deleted**, deactivating is the way out **[W]**. The API agrees: delete fails if the tag is used in
  transactions, in custom views or in workflows **[W]**.
- Not stated anywhere: whether renaming an option rewrites old records (the API gives options a stable `option_id`, so it is
  presumably a label change **[inferred]**), what a deactivated option looks like on an old record or in a report, and what
  happens to children when a parent option is deactivated **[unverified]**.

---

## 2. How reports use tags

| Report behaviour | What Zoho documents |
|---|---|
| Filter | In a Business Overview report such as Profit and Loss: choose the tag, a comparator ("is in" on the older page), the options, run. Newer UI: Customise, General, Advanced Filters; the filter can be saved as a custom report **[W]** |
| Compare | "Compare With", then compare based on reporting tags: pick one or more tags and, for each, some options or all of them; an optional switch lays the chosen tag out by its parent-child hierarchy **[W]**. The Profit and Loss help also lists branch-wise, project-wise and month or year-wise comparisons beside it |
| Untagged | the old "Untagged" choice was renamed: pick the tag and use the comparator "is empty" **[W, community post, undated]**. The API's "all options" call has a flag to include an "untagged" entry **[W]** |
| Which reports | the community post that moved tag filters into Advanced Filters names the Balance Sheet (and its two Indian-edition forms), Cash Flow, Trial Balance, Sales by Item and the Schedule III Profit and Loss; the Account Transactions report has tag, project and account filters; the post says more reports would follow **[W]**. The Profit and Loss tag KB page itself lists no other report **[W]** |
| Detail reports | line-level tags cannot customise Invoice Details or Bills Details, but do work in Receivable Details and Payable Details **[W]** |
| Columns shown, untagged column, totals check | **not stated** on any page I read **[unverified]** |

Two things follow for a design **[J]**. First, Zoho's tag reporting is *attached to existing reports as a filter or a column set*; it
is not a separate "report by tag" screen, and it includes balance-sheet reports, which means tags are carried on the postings of
every account, not only income and expense. Second, Zoho never says what happens to the figure a tag does not cover, which is the
exact question an accountant asks first.

---

## 3. API, import, templates, multi-currency

| Topic | What is documented |
|---|---|
| API | a reporting-tags resource: create, list (in set order), update, delete, update options, update visibility conditions, activate or deactivate a tag, activate or deactivate one option, set or clear the default option, list all options (with the untagged flag), reorder **[W]**. A tag has id, name, order, description, active, mandatory, draft, and a per-module list; its transaction-or-line preference is a field on the tag. Options carry id, name, active, default, children, depth, order, parent and conditions. Two tag fields (`is_class`, `is_draft`) are shown without explanation |
| Import | not stated for tags. A 2019-era release note found through search says custom fields on the item table can be imported with each module's import **[Ws]**; nothing about tag values **[unverified]** |
| Templates and PDFs | not stated for tags. For custom fields there is a per-field "show in all PDFs" switch and an image field can be placed in a template by placeholder **[W]** |
| Multi-currency | not stated **[unverified]** |
| Recurring documents | recurring invoices, expenses and bills are listed among taggable modules; how a recurrence copies the tag is not stated **[W, unverified]** |

---

## 4. Tags, custom fields, projects and branches in Zoho

| | Reporting tag | Custom field | Project | Branch (Locations) |
|---|---|---|---|---|
| What it is | named dropdown reused across modules, with optional hierarchy | a field added to one module's form | a customer engagement with a billing method, budget and cost rates | a business location, with its own tax registration in India |
| Types | dropdown only | about 20: single-line and multi-line text, email, URL, phone, number, decimal, amount, percent, date, date-time, check box, auto-number, dropdown, multi-select (up to 30 options), lookup, multi-select lookup, attachment, formula, image, external lookup; AI fields on some plans **[W]** | n/a | n/a |
| Mandatory | advanced mode: yes | yes, per field **[W]** | n/a | not stated |
| Printed documents | not stated | per-field "show in all PDFs" **[W]** | n/a | address and tax follow the branch **[Ws]** |
| Used in reports | first-class: filters and compare columns in the reports above | **not stated** in the custom-field or custom-report help; a Zoho community question about getting an invoice custom field into the analytics add-on has no answer on its page, and another, about filtering invoices by a custom field through the API, is unanswered too **[W, unverified]** | profit and loss filtered by project; Projects Cost Summary and Projects Revenue Summary reports **[W]** | profit and loss compared by branch **[W]** |
| Reach | 14 or 22 modules, items, contacts | one module at a time | quotes, invoices, orders, credit notes, bills, expenses, purchase orders, vendor credits, manual journals, time logs **[W]** | sales, purchases, manual journals, bank transactions **[W]** |
| Plan | Standard and up | Standard and up, with per-type caps (dropdown 5 on Standard and Professional, 20 on Premium and above; 3 yes/no; 2 or 5 multi-select) **[W]** | Professional and up **[W]** | Premium 3, Elite and Ultimate 8 **[W]** |

Notes: the 2020 launch post for branches says that once a GSTIN is tied to a branch, tags are no longer needed for tracking
locations **[W]**, so Zoho itself separates "a legal or location split" (branch) from "any other cross-cutting view" (tag) **[J]**.
The pricing page says Locations while the help says branches; I read them as one feature **[J, unverified]**. The dropdown cap and the
custom-report cap (10, 25, 50, unlimited by plan) are plan limits Zoho sells; the page does not say what each counts.

What this table shows **[J]**: a custom field and a tag are two screens with overlapping capability. Zoho's field list is far
richer than its tag, and its tag reaches reports where its field is silent. Neither is the whole of what the owner described.

---

## 5. TallyPrime cost centres and cost categories (what Tally documents)

**Model.** Any number of **cost categories** besides the built-in Primary one; each holds its cost centres; categories cannot have
sub-categories; **cost centres** nest under another centre (a person under a department) **[W]**. A category has two switches:
allocate revenue items (income and expense) and allocate non-revenue items (assets and liabilities), at least one on **[W]**.

**Entry.** The allocation screen opens after an income or expense ledger and its amount. The total must be allocated across one or
more centres; when the split does not equal the ledger amount Tally refuses with a "break-up total does not match" error, which
also appears on imports **[W]**. **Parallel allocation:** choose a category first, then its centres, and repeat for every category
you want to report on; the worked example allocates a 35,000 travel expense once by employee and once by sales scheme, each adding to
35,000 (the page gives the example, not a stated rule) **[W]**. A **cost centre class** stores percentage splits across categories
and centres and applies them when chosen in a voucher (enabled per voucher type); classes cannot be imported between companies **[W]**.

**Which ledgers and vouchers.** Income and expense ledgers prompt by default, assets and liabilities do not unless the category
allows non-revenue items; a ledger or all ledgers can be switched off **[W]**. Tally's FAQ says cost centres go on **accounting
vouchers only, not inventory vouchers**, yet also says a Delivery Note can carry them and a Receipt Note cannot; the two statements
sit in tension on the same page **[W, unverified which holds for an item invoice]**.

**Reports.** Cost Category Summary (compares parallel sets), Cost Centre Break-up (ledger groups for one centre, with comparison
columns), Cost Centre Summary and Monthly Summary (totals and months, drilling to vouchers), Ledger Break-up and Group Break-up of
a centre, Cost Centre Vouchers, and ledger reports that can show the centre detail **[W]**. Tally's FAQ states that TallyPrime does
**not** show a Profit and Loss or Balance Sheet by cost centre and points to add-ons; the job-costing guide adds that a formatted
per-job profit and loss needs the data exported and restructured **[W]**.

**Unallocated amounts.** Tally's guide to job costing is explicit: an entry with no allocation is posted to the correct ledger but
is absent from the cost-centre reports; job expenses read too low and profit too high, and the remedy is an allocation journal. It
advises that the allocated total for a ledger should match that ledger's closing balance in the trial balance, and does not mention
tying to the profit and loss account **[W]**. Opening balances are allocated through a temporary adjustment ledger and a journal
**[W]**.

**Guards.** A centre with allocations cannot be deleted until they are moved; a category can be deleted only when empty; turning
cost centres off means deleting them first, after which the breakup reports are unavailable until re-enabled **[W]**.

**How Tally's own material uses them.** Verticals of one business (new cars, workshop, parts), a second dimension such as geography
as a category beside the verticals, projects and jobs, salespeople grouped under a department **[W]**. That is the practical
answer a Tally partner gives today to "tag a sale with its zone".

---

## 6. What Comsky has today (code reading, nothing run)

### 6.1 Cost centres

- **Model.** `CostCategory { id, cid, name, reserved? }` and `CostCentre { id, cid, name, categoryId, parent }`
  (`src/types.ts:633-646`); an allocation is `CostAllocation { costCentreId, amount }` on `VoucherEntry.costs`
  (`types.ts:875-886`). So the attribution unit is the **accounting entry**, never the voucher header or an inventory line. The
  centre has no active flag.
- **Which ledgers.** The ledger form offers "Cost centres are applicable" only for ledgers in an Income or Expenses group and only
  when the company feature is on (`ui/masters.tsx:334, 372`). Tally's "allocate non-revenue items" has no counterpart.
- **Entry.** `CostPanel` (`ui/VoucherScreen.tsx:498-660`) opens after the amount (and after bill-wise) on a flagged ledger when at
  least one centre exists (`:2125-2150`). It is one list of Cost Centre and Amount rows that must sum to the entry (`:538`, the
  Ctrl+A guard `:567`); the picker lists every centre of every category with the category name as a sub-label (`:544-547`).
  **Esc leaves the line unallocated and the walk carries on** (`:566`); I found no save-time check that a flagged ledger line was
  allocated.
- **Only in accounting mode.** A voucher has one of three shapes (`shapeFor`, `:104-110`): accounting (`acc`), item invoice
  (`inv`) or stock journal. Sales, Purchase and the notes default to item invoice, and the save code for that shape builds the
  party, item-ledger and extra-charge entries with no `costs` (`:2580-2590`; only the accounting branch copies `r.costs`, `:2560`).
  So an item invoice cannot carry cost centres; the person has to enter the sale "as a voucher" to allocate. Tally's FAQ hints at
  the same split for purchases (it advises switching to voucher mode when a break-up will not match) **[W, inferred]**.
- **Reports.** Cost Category Summary (`ui/reports/costcentrereport.tsx`), Cost Centre Summary and Cost Centre Vouchers
  (`ui/reports/financial.tsx:718, 808`), Cost Centre Analysis with Category-wise and Centre x Ledger views, sub-centre roll-up and
  added comparison periods (`ui/reports/costanalysis.tsx`), and the **Cost-Centre P&L Breakup** (`ui/reports/costpl.tsx`), which splits
  each centre's allocated amounts by the ledger's group nature into income and expense with a net. All use
  `core/costCentreTree.ts`, whose `allocationsByCentre` and `centreVoucherRows` skip anything `postsToLedgers` rejects (optional,
  memorandum). They sit in the Reports Center under "Cost Centres & Budgets" (`ui/reportsCenter/catalogue.ts:173-177`). Their
  period totals are of allocated amounts only: **there is no "not allocated" row** in any of them (a search of the cost report files,
  `costCentreTree.ts` and the Cost Centre Summary finds none).
- **Not wired to the new grid or filter.** The row filter's allow-list is the Day Book, the voucher registers, Ledger Vouchers and
  Bills (`core/reportFilterFields.ts`), none of which has a cost-centre field, and the planned column catalogue (`docs/research/
  wave-a2-plan.md`) has no cost-centre group. A person cannot filter the Sales Register by cost centre today.
- **Guards.** A centre with allocations or sub-centres, and a category with centres, cannot be deleted (`state/store.ts:832-838`).
- **Tally XML.** Export groups a posting's allocations by the category of each centre into `CATEGORYALLOCATIONS` blocks
  (`core/tallyExport.ts:809-826`); import reads them (`core/tallyImport.ts:1242-1255`).
- **Parity rows** (`docs/parity-data.json`): `costcentre-parallel-allocation` is **partial** ("single flat pool summing once; no
  per-category 100% enforcement"), `costcentre-class` is missing, `budget-costcentres` is missing, `job-costing` is partial,
  `employee-categories` is missing.

### 6.2 Defects and limits the comparison turned up (not fixed here; for the orchestrator) **[C, J: by reading, not run]**

0. **An item invoice cannot be allocated to cost centres** (section 6.1, "Only in accounting mode"). For a business whose sales are
   item invoices, the main sales document has no cost-centre or tag-like dimension today.
1. **Parallel allocation cannot be entered.** The panel forces *one* list to sum to the entry across all categories, so a person
   cannot allocate Rs 35,000 under Employees and again under Sales Schemes; the second block fails the sum. This is the known
   partial row.
2. **The importer would double count a parallel-allocated file.** `readCosts` collects *every* `COSTCENTREALLOCATIONS.LIST` under a
   posting, across all its `CATEGORYALLOCATIONS.LIST` blocks, into one flat list with **no check that the list sums to the
   posting's amount** (the bill allocations next to it do have one) (`core/tallyImport.ts:1242-1255`, merged again at
   `:1466-1469`). With two categories each at 100% the entry would carry about twice its own amount in allocations, and every cost
   report would count each rupee once per category. No live TallyPrime file has been through the importer (CLAUDE.md), so this is
   an inference from the code, worth a fixture test before anyone relies on the Tally migration for a customer that uses categories.

### 6.3 Custom fields (UDF) today

- `CustomFieldDef { id, cid, objectKind, key, label, type, options?, required?, order? }` with `objectKind` limited to **ledger**
  and **stockItem**, types text, number, date, yes/no and list (`types.ts:400-420`); values live in `udf` on the Ledger and Stock
  Item (`types.ts:366, 526`). **No voucher, voucher line, entry or other master carries a value.**
- A list field stores the option's **text** in the value (`ui/udf.ts collectUdf`), and the options are typed once as a
  comma-separated string at creation; the admin screen offers add and delete only, so I found no way to edit a field or rename an
  option (`ui/admin/CustomFields.tsx`). A rename would orphan stored values, and deleting a field leaves them behind.
- `required` is stored and shown in the admin list, but `ui/udf.ts` builds form fields without it and `ui/forms.tsx` has no
  `required` handling, so I found nothing that enforces it **[C, J]**.
- The planned report grid reserves a "Your custom fields" group with ids like `udf:ledger:<key>` (`docs/research/wave-a2-plan.md`
  section 5), and distinguishes a value **read live from a master** (the column says "(current)", `truth: 'now'`) from one **stored
  on the posting** (`pipedrive-list-ux.md` section 5).

---

## 7. Side by side: where tags add something, where cost centres already cover it

| Capability | Zoho reporting tag | Tally cost centre / category | Comsky today | Read |
|---|---|---|---|---|
| Nature of the value | a label on a record or line; no amount | a piece of an amount; pieces sum to the posting | the Tally model, one flat pool | different tools **[J]** |
| Split one amount across values | no (not stated) | yes, by amount, and by percentage class | yes, by amount; no class | cost centres cover it |
| Same amount under several dimensions | one tag per dimension, each its own dropdown | parallel allocation, 100% under each category | **not possible** (single pool) | covered by tags more simply; cost-centre gap is ours |
| On the voucher header | yes (advanced, 4 module groups); older pages no | no, the posting only | no | tags add |
| On an inventory line | yes (line level; items too) | no (FAQ: not on inventory vouchers) | no; and an item invoice, the default Sales and Purchase shape, takes no allocation at all | tags add |
| On a master as a default that copies to lines | items, customers, vendors | no (a class is percentages, not a default) | no | tags add |
| On non-posting documents (quotes, orders) | yes | not for receipt notes; delivery note yes | n/a | tags add |
| On balance-sheet accounts | yes (reports include Balance Sheet, Trial Balance) | only if the category allows non-revenue | no | tags add |
| Mandatory | advanced: per tag | the allocation must balance once started; whether it can be forced is not stated | not enforced | tags add, cost centre unclear |
| Default option | yes | no | no | tags add |
| Hierarchy | 5 levels of options, per tag | centres nest without stated depth; categories flat | centres nest; roll-up | equal |
| Conditional options | yes (advanced) | no | no | tags add, low priority **[J]** |
| Access by value | one tag per organisation | no | operator rights are by area, not by value | tags add, later **[J]** |
| Deactivate without losing history | options and tags | delete blocked while used | no active flag | tags add |
| Report: P&L by value | filter or compare columns on P&L, Balance Sheet, Trial Balance, Cash Flow | Cost Centre Breakup (ledger groups); no P&L or Balance Sheet by centre (Tally's FAQ) | Cost-Centre P&L Breakup (income, expense, net per centre) | each covers part; Comsky's is ahead of Tally |
| Report: filter any list by value | detail reports by tag where tags are line-level | no | no (not in the row filter) | tags add |
| Roll-ups and drill | by option, hierarchy layout | sub-centre roll-up, drill to vouchers | roll-up, drill, Centre x Ledger, comparisons | cost centres cover it |
| Untagged / unallocated shown | an "is empty" filter, no stated column | absent from reports (Tally's own caveat) | absent | neither covers it **(section 8.3)** |
| Limits | 10 tags, 500 options (older pages) | none stated | none | do not copy |

The shortest reading **[J]**: a tag adds *reach* (more places a value can live, defaults that copy, mandatory, reports beyond P&L);
a cost centre adds *arithmetic* (splits, parallel dimensions, roll-ups). A clothes dealer's colour and a distributor's zone need
reach. A job-costing builder who shares one expense between three jobs needs arithmetic.

---

## 8. What this means for Comsky Books [J]

### 8.1 The answer to the three options

**Treat a tag as a custom field of type dropdown that is also groupable, and do not build a separate Tag master. Keep cost
categories as they are, fix their parallel-allocation gap, and let both feed one "by this dimension" report seam.**

Why, option by option:

1. **A new tag dimension beside custom fields.** It would duplicate the option list, the type, the mandatory flag and the "where is
   it visible" settings that a custom field already needs for the owner's own examples, and it would put two screens in front of a
   Tally customer who asked for one ("add a column"). Zoho's own history, a mandatory dropdown field plus a workflow to fake a
   mandatory tag, is the cost of keeping them apart (finding 6).
2. **Generalise cost categories into tags.** A cost centre's meaning is an amount split on an accounting entry, it round-trips
   through Tally's XML (`CATEGORYALLOCATIONS`), and a Tally partner already speaks it. Stretching it to carry an item's colour, a
   contact default or an unposted quote would blur that meaning and risk the round-trip. The one useful half-truth is that a
   category whose every allocation is a single centre at 100% *is* a label; Tally's own guide uses a category for geography that
   way. So imported Tally categories used as labels will look like dimensions, which is a reason for the shared report seam below.
3. **A groupable dropdown custom field.** It covers all three of the owner's examples: Zone on a Sales voucher (header level), Colour
   on an item (master default, line-level value), and a profitability tag across several voucher types (header or line, several
   types). It grows an object that already exists (`CustomFieldDef`, its table, its sync, its admin screen) instead of adding a
   second.

### 8.2 The model, in outline (what the build should be shaped like)

- **Where a field lives** (`appliesTo`): Voucher header (chosen voucher types, so "visible on Sales only" is a list), voucher line
  (an inventory line, or an accounting line which is a `VoucherEntry`), Ledger (party), Stock Item. Today only the last two exist.
- **What it is:** a type (text, number, date, yes/no, dropdown first; multi-select later and *never groupable*), a stable option
  list with ids, an active flag and an optional default per option, a mandatory flag, a "show in reports" flag, and a "use for
  analysis" (groupable) flag that is offered for dropdown and yes/no only.
- **Options have ids, not text.** A rename is a label change; a deactivated option stays on old records and in saved views; a
  delete is blocked while any record uses it (the same rule the cost-centre master already follows, `state/store.ts:835`).
  The current list field, which stores the option text, cannot do this.
- **Snapshot at entry, never read live.** An item's or party's default option is *copied* onto the header or line when it is entered
  and then belongs to that document. Changing the default later must not re-tag history (the same reason the GST rate is
  snapshotted on the line, CLAUDE.md Phase 20-21 gotchas).
- **The attribution rule for reports** is per **ledger entry**, because that is what a P&L, Trial Balance or Balance Sheet adds up:
  a header value applies to every entry of the voucher; an entry-level value applies to that entry; an inventory-line value
  applies to the voucher's item-ledger entry in proportion to the line amounts (in an item-invoice voucher that entry is posted at
  the lines' total, `ui/VoucherScreen.tsx:1469, 2585`). Freight, discount, round-off and other extra-charge
  entries carry no line and take the header value if there is one, else fall into Untagged. Pro-rating them across lines is a later
  option, not a default.
- **One seam for "by this dimension" reports.** A dimension reports, for each entry, a list of `{ option, share }`: a cost category
  answers with the allocation amounts, a groupable field answers with the whole entry. Every by-dimension report then gets an
  automatic **Not tagged / Not allocated** remainder, so the two never disagree about arithmetic. New comparison columns need a third
  column kind beside `asOn` and `period` (`core/reportColumns.ts:18`) for "one column per option" (Zoho's compare-by-tag).

### 8.3 What "reconciling to the books" has to mean

Neither vendor states this, so these are the Comsky rules **[J]**, drawn from the Phase 26 gotcha that a filter must say it is filtered
and must not move a figure it did not filter, and from Tally's own warning about unallocated entries:

1. **Every by-option report ends with a Not tagged row, and the options plus Not tagged equal the report without the dimension**
   (net profit, trial balance total, ledger balance). The row is hidden only when it is exactly zero. The same row is **added to the
   existing cost reports** ("Not allocated"), which today print allocated amounts only.
2. **A groupable field is single-valued per record or line.** A multi-select cannot be grouped on, because one line would count
   in two columns and the options would add up to more than the whole. If an amount genuinely belongs to several places, that is
   what a cost category is for.
3. **Same guard as the books:** only entries `postsToLedgers` accepts (not optional, memorandum or cancelled; post-dated only from
   their date), exactly as `core/costCentreTree.ts` does.
4. **A filter reports as a filter.** A register filtered by a field shows "Total (filtered)" (the Phase 26 rule), and a saved view
   that names a deleted or unknown option matches nothing rather than everything.
5. **Say whether a column is live or posted.** An item's colour read through the item is "(current)"; the value stored on the line
   is the posted one. Both can be columns; they must be labelled differently (`truth: 'now'` in the planned catalogue).
6. **Balance Sheet and Trial Balance by option are a choice, not a default.** The default analysis is the P&L ledgers only, as in
   the Cost-Centre P&L today; letting an administrator include balance-sheet ledgers is the equivalent of Tally's "allocate
   non-revenue items".

### 8.4 The owner's question: should a field automatically appear in reports?

**Make availability automatic and display opt-in.** Zoho's pattern is close to this: once a tag exists it is offered as a filter in
the listed reports, and nobody's existing report changes **[W, J]**.

- A field marked "show in reports" (default **yes**) is *offered* in the Columns panel and the filter of every report whose rows are
  that object or reach it: voucher fields on voucher lists and registers, party fields on anything that lists parties or bills,
  item fields on item and stock lists. It appears under "Your custom fields".
- It is **never** added to anyone's default columns, and no saved view changes, so a person who never creates a field sees exactly
  what they saw before (the Phase 27 opt-in rule, asserted by identity).
- "Use for analysis" (groupable) is a separate switch. It makes the field a grouping and a compare column, not merely a column.
- "Show on printed documents" is a third switch, default **no** (Zoho's custom fields have the same per-field switch **[W]**).
- Voucher type chooses where the field is *entered*; reports choose where it is *shown*. A zone entered on Sales only is still
  offered in a report that lists several voucher types, blank on the types that never carry it.

### 8.5 The two examples, worked

- **Zone on a Sales invoice.** A dropdown field, voucher header, applies to Sales (and Credit Note if wanted), North, South, East,
  West, mandatory if the owner wants. Entered in an Alt-key panel or appended to the walk (section 8.6). Appears in the Sales
  Register's column picker and filter; with "use for analysis" on, the Sales-by-zone view and a P&L by zone exist, with a Not tagged
  row for vouchers that predate it.
- **Colour for a clothes dealer.** If colour belongs to the item: a stock-item dropdown field (exists today), which then needs to
  become an item-report column and filter. If colour is chosen on each sale: a line-level dropdown field on Sales and Purchase
  lines with the item's colour as the default copied at entry, mandatory or not, shown as a column on the item lines of the invoice,
  in the Sales Register's item level, and in stock movement reports (which need the line value to flow into the stock ledger, an
  extra piece of work for inventory reports). **Ask the owner which one he means.**

### 8.6 Constraints and risks

- **The voucher Enter walk is frozen.** A new header field cannot be inserted before existing fields; the Phase 24 gotcha records
  Opening-Balance and Accept-on-Enter walks that broke when fields went in the wrong place. A mandatory header field must stop the
  walk before Accept, and `Y` and Ctrl+A must not save without it; the owner decides where it sits (after Narration, or in an Alt-key
  panel like Dispatch, Alt+S).
- **Opt-in, asserted.** With no custom voucher field defined, every voucher screen, report and print must be byte-identical to
  today (Phase 27 rule).
- **The row filter needs per-company options.** The filter engine's enum fields were kept to fixed lists because a saved view cannot
  check a company's own options (`core/reportFilterFields.ts`, the "Voucher kind" note). A dropdown field's options differ per
  company, so the filter needs an option-id kind and the "stale value matches nothing" rule.
- **Storage.** Header, line and entry values are new properties on three document shapes that sync; unknown properties must survive
  an older client's save (the Phase 28 layout-reader gotcha) or an old app would erase the new values.
- **Do not copy:** the 10-tag and 500-option caps (a Comsky tenant's plan limits are display-only today, CLAUDE.md), the "Not Ready"
  state, an irreversible switch, and access control limited to one tag. Conditional options and access-by-value are real ideas and
  belong later.
- **Fix parallel allocation separately** (section 6.2). It is a parity item, but the importer double count makes it the first
  thing to test before any customer imports a Tally file that uses cost categories.

### 8.7 Questions for the owner

1. Colour: a property of the item, or chosen on each line of a sale?
2. Is a P&L by zone (or by tag) wanted, or only filtering and grouping on the register? If a P&L, is balance-sheet analysis ever needed?
3. Should a mandatory voucher-level field block saving, and where in the entry walk should it sit?
4. Do any customers need to *split* one amount across zones, jobs or people? If yes, that is a cost category, not a tag.
5. Is hiding a zone's data from a branch manager (access by value) a near-term need?
6. Are Tally customers, today, using cost categories as labels (zone, salesman)? A look at two customers' companies would settle how
   many imported categories would become dimensions on day one.

---

## 9. What was not verified

- No live Zoho Books, TallyPrime or Pipedrive organisation. Every statement about behaviour is from documentation or code.
- The summarising fetch **disagreed with itself** on two renderings of the older Zoho tag page (line-item only versus a
  "More" menu); I followed the page text plus the two detail-report answers. A re-check in a live organisation would settle it.
- **Undated pages:** the community post that moved tag filters into Advanced Filters, the "is empty" rename, and the mandatory-tag
  workaround post (date inferred from image names, about May 2023). The advanced help describes itself as early access and
  plan-limited; I could not tell how widely it has rolled out.
- Not found in Zoho's pages: multi-currency behaviour, tag values on PDFs, import or export of tag values, retroactive tagging,
  what a deactivated option does on an old record, whether one tag takes several options on a line, whether a report prints an
  untagged column or ties out, any per-plan tag count.
- Not found in Tally's pages that I could read: what each cost-centre report prints in detail, a "Not Applicable" line, and
  whether the allocation screen can be skipped. The Tally FAQ's account of inventory vouchers is internally inconsistent.
- Code facts in section 6 are from reading, not running. The importer double-count (6.2) is a prediction from reading `readCosts`,
  not a test result.
- Nothing here checks pricing or plan structure as of today beyond the single comparison page, whose table the fetch could read
  only as text.

---

## 10. Sources

Zoho (all [W] unless marked):
- Reporting tags, older help: https://www.zoho.com/books/help/settings/reporting-tags.html and https://www.zoho.com/us/books/help/settings/reporting-tags.md and https://zstatic.zohostatic.com/en-de/books/help/settings/reporting-tags.md
- Reporting tags, advanced help: https://www.zoho.com/au/books/help/settings/reporting-tag.html and https://www.zoho.com/es-mx/books/help/settings/advanced-reporting-tag.html
- Profit and Loss by reporting tag: https://www.zoho.com/in/books/kb/reports/profit-and-loss-report-reporting-tag-wise.html
- Detail reports and tags: https://www.zoho.com/us/books/kb/reports/reporting-tags-in-invoice-details-report.html and https://www.zoho.com/en-fr/books/kb/reports/reporting-tags-in-bills-details-report.html
- Advanced Filters in reports (community post, undated): https://help.zoho.com/portal/en/community/topic/enhancement-to-advanced-filters-in-reports?page=1
- Mandatory-tag workaround (community post, date inferred): https://help.zoho.com/portal/en/community/topic/function-3-assign-a-mandatory-reporting-tag-to-invoices
- Reporting Tags API: https://www.zoho.com/books/api/v3/reporting-tags
- Custom fields: https://www.zoho.com/en-ng/books/kb/invoices/invoice-additional-fields.html and https://www.zoho.com/books/help/settings/customization/custom-field-data-types.html; two unanswered community questions: https://help.zoho.com/portal/es/community/topic/zoho-books-invoice-custom-fields-are-not-available-in-analytics and https://help.zoho.com/portal/zh/community/topic/filter-invoices-by-custom-fields
- Custom reports: https://zoho.com/books/help/reports/custom-reports.html
- Projects: https://www.zoho.com/en-fr/books/help/projects/functions.html
- Branches: https://www.zoho.com/in/books/kb/branches/compare-profits-bwn-branches.html and the launch post https://www.zoho.com/blog/books/introducing-branches-and-multiple-gstin-management.html (2020-07-16)
- Plan comparison: https://www.zoho.com/au/books/pricing/pricing-comparison.html
- [Ws] only (search summaries, pages not opened): the expense-itemise release note and the 2019 custom-field-import release note, both on help.zoho.com community "What's New" posts.

Tally (all [W]):
- Cost centres FAQ: https://help.tallysolutions.com/tally-prime/financial-controlling/cost-centres-in-tallyprime-faq/
- Cost centre allocation: https://help.tallysolutions.com/tally/cost-centre-allocation-tally/
- Cost centre or profit centre: https://help.tallysolutions.com/tally-prime/financial-controlling/cost-centre-or-profit-centre-tally/
- Cost Centre Reports (introduction and links only; bodies unread): https://help.tallysolutions.com/docs/te9rel52/Reports/MIS_Reports/Cost_Centre_Reports.htm
- Business verticals with cost centres: https://tallysolutions.com/business-guides/manage-multiple-business-verticals-tallyprime-cost-centres/
- Job costing and profitability: https://tallysolutions.com/tally/how-to-track-job-costs-and-profitability-in-tallyprime/

This repo ([C]): `src/types.ts`, `src/core/costCentreTree.ts`, `src/core/tallyImport.ts`, `src/core/tallyExport.ts`,
`src/core/reportFilterFields.ts`, `src/core/reportColumns.ts`, `src/state/store.ts`, `src/ui/VoucherScreen.tsx`, `src/ui/masters.tsx`,
`src/ui/udf.ts`, `src/ui/admin/CustomFields.tsx`, `src/ui/reports/cost*.tsx`, `src/ui/reports/financial.tsx`,
`src/ui/reportsCenter/catalogue.ts`, `docs/parity-data.json`, `docs/research/wave-a2-plan.md`, `docs/research/pipedrive-list-ux.md`,
`docs/research/zoho-books-reference.md`, CLAUDE.md (Phase 20 to 28 gotchas).
