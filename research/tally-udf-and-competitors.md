# Custom fields, dimensions and the Tally customisation market: what a switcher expects, what competitors do

> **Status:** research only. Nothing here is built, and no source file was changed. Written 2026-10-21 (docs only; a parity
> wave 3 build was running in the same working tree, so every `[C]` note below describes the tree as read that day, not a
> committed state).
> **Why this exists:** the owner asked for (1) a Pipedrive-style customisable report grid, (2) quick-saved views, (3) object-wise
> custom fields (mandatory or not, a type, where visible, and whether they reach the reports), plus a look at Zoho's custom fields
> and "reporting tag" idea, and (4) a way back in every report for a mouse-only user. This doc answers the **demand and
> competitor** half of item 3. Items 1 and 2 are already researched in `pipedrive-list-ux.md` and `wave-a2-plan.md`; item 4 is
> not covered here (section 7 says what I noticed in passing).
> **Evidence tags:** **[W]** a vendor page I fetched and read; **[Ws]** a search summary only (the page itself not read);
> **[C]** read in this repo's code or docs (file named); **[J]** my judgement; **[unverified]** I looked and could not confirm.
> Everything is paraphrased; no vendor text is reproduced. Page summaries were produced by a fetch tool that condenses each page
> with a small model, so a page that "does not say" something may simply have lost it in the summary; where that matters I say so.

## 0. Read this first

### 0.1 What I could not verify

1. **No live account of anything.** I had no TallyPrime with a TDL loaded, no Zoho Books, QuickBooks, Xero, BUSY, Marg, Vyapar,
   Odoo or Pipedrive login. Everything about behaviour is from documentation, vendor pages and community threads.
2. **TallyPrime's own help was patchy.** The UDF mechanics I could read in full are in the TDL reference of Tally.ERP 9
   (release 5.4 docs) and the TallyPrime developer pages that repeat them. The UDF page's own examples cover only the Company
   and the ledger-entry level; the other levels in section 2.2 are inferred from neighbouring pages and marked. I found **no
   no-code UDF screen in TallyPrime**, but absence from help pages is not proof for the newest release **[unverified]**.
3. **Marg's own site could not be read** (the fetch failed on its certificate). Everything about Marg is from a third-party
   software directory listing, and that listing does not mention user-defined fields at all.
4. **Xero Central returned an empty page** to the fetch tool, so Xero's tracking limits rest on a search summary of Xero's own
   article plus third-party posts that disagree on the option cap (100 or 500).
5. **Zoho's two reporting-tag help pages describe different generations** of the feature (an older page and an "Advanced
   Reporting Tags" page), and plan caps came from the Australian edition's comparison table, not the Indian one.
   Whether Zoho custom fields can be a **report column** is not stated in any Zoho page I read.
6. **Vyapar's field counts come from blog posts of uncertain age** (one is about four years old) and they disagree with each
   other. **BUSY's "10 item fields" figure is from a search summary** I could not find on the page that discusses the feature.
7. **Frequency of customisations is not measured anywhere.** The "most common" list in 2.6 is built from what partner and vendor
   pages name, plus one Tally example. No source gave counts or a ranking.
8. **Odoo's Studio page was truncated** at 100,000 of 117,000 characters and says nothing about group-by on new fields; the claim
   that Studio is Enterprise-only comes from a vendor blog.

### 0.2 The findings that change what we build

1. **"Custom field" is three different jobs, and the owner's own examples hit all three** (section 1). *Zone on a Sales voucher* is
   a descriptive voucher field. *Colour on an item* is an item attribute, and if the dealer wants stock by colour it is really a
   variant. *The Zoho tag the owner calls a profitability tag* is a **dimension**: one tag applied across many document types so the
   profit statements can be cut by it. Building only the first would answer the first example and miss the other two.
2. **Tally has no end-user custom field. Every added field is a TDL development job** (section 2). That is the whole business
   behind the reseller demand the owner describes: each field touches several definitions (form, every invoice line, report,
   filter) and breaks on upgrades. A Tally switcher's minimum is simply "add a field on a screen, with no partner".
3. **Competitors cap and gate this feature, and almost none lets a custom field into the financial statements.** Vyapar gives a
   handful of fields; BUSY 20 per voucher series; QuickBooks 1 to 12 per transaction by plan; Zoho caps by field type and plan
   (5 dropdown fields on its lower paid plans in the table I read); Xero offers two active tracking categories and no custom
   fields at all, with the request open since 2012. QuickBooks states plainly that a P&L by custom field does not exist. **No
   technical cap and "the field is reportable" are two ways Comsky can be plainly better.**
4. **Where a tag is attached decides what it can report.** QuickBooks tags classes on the detail lines only, so a class cannot
   filter the balance sheet; its locations sit on the header and the lines, so they can. Zoho's line-level tags cannot be used in its
   Invoice Details report. A header tag that does not reach every ledger leg cannot cut the Balance Sheet; a line tag cannot appear
   on a one-row-per-invoice register. This is the design decision that matters most for "zone" and "project".
5. **The "does a new field appear in reports?" question has a consistent market answer: opt-in for display, easy to opt in.**
   Tally never (TDL per report); BUSY by a custom column you build; Vyapar and Zoho tick "show in print"; Pipedrive offers the field in
   the column picker the moment it exists. No competitor makes a new field show in standard layouts by itself. Section 6.4 turns this
   into a rule that also keeps Comsky's "nothing chosen, nothing changes" promise.
6. **A switcher's existing UDF data is a migration asset nobody else addresses.** Tally's export carries UDFs with their original
   index number (so data can be recovered when the TDL is gone) **[W]**; Comsky's Tally XML importer reads "the fields we understand
   and drops the rest" **[C `src/core/tallyImport.ts` header]**, and the exporter lists custom fields among what it does not carry
   **[C `src/core/tallyExport.ts` ~line 741]**. An importer that maps a file's UDFs to custom fields would be a real switch-day win
   (never tried on a live TallyPrime file **[unverified]**).
7. **Comsky's current custom field is thinner than it looks.** Masters only (Ledger, Stock Item); the **Required flag is stored and
   printed in the admin table but nothing enforces it**; no voucher or line level; no report reads it **[C]** (section 5).
8. **Vehicle no., LR no., transport details are being built as fixed fields in wave 3** (uncommitted `src/core/dispatchTypes.ts`)
   **[C]**, and TallyPrime has them built in too **[W/Ws]**. So the custom-field layer should not be sold on "vehicle number"; it
   is for what no vendor can foresee (zone, colour, jewellery weights, a site code).

## 1. Three things people mean by "custom field"

| Job | What it is | Owner's example | How competitors name it | What it must be able to do |
|---|---|---|---|---|
| **Descriptive field** | A label or value stored on one object (a voucher, an item, a party) | Zone on a Sales voucher; a licence number on a party | Tally UDF; BUSY optional field; Vyapar additional field; Zoho and QuickBooks custom field | Appear on the entry screen and the print; be a register column and a filter |
| **Dimension / tag** | A named list of options attached to many document types, usually per line, so profit and cost statements can be cut by it | The Zoho tag "created once and tagged to multiple transaction types"; zone or project when profit by zone is wanted | Zoho reporting tags; QuickBooks classes and locations; Xero tracking categories; Odoo analytic plans; Tally cost centres | Reach the ledger postings, so P&L (and ideally Balance Sheet) can be filtered and compared by it |
| **Variant** | The identity of a stock unit (this red, size M) with its own quantity, rate and barcode | A clothes dealer's colour at item level, if stock by colour matters | BUSY parameterised details; Pixel-style size-colour matrix; Tally's item-per-size workaround | Carry stock and price per combination; a label cannot do this |

[J] A single "Custom Fields" screen can host the first two, and the first two must share one option-list editor and one place in
the report picker. The third is a stock-engine feature that custom fields feed (the attribute list) but do not replace.

## 2. TallyPrime and TDL

### 2.1 What a UDF is
- **Tally's own explanation:** the data structures and storage are fixed, so a UDF adds storage to an existing object for
  information the product has no field for. Any Tally data type works (string, amount, quantity, rate, number, date, rate of
  exchange, logical) and an **aggregate** type groups several typed values into one row-like record **[W: TDL reference, "User Defined
  Fields"]**.
- **Declaration:** a name, a type and an **index number** in the system section. Index range 1 to 65,536 per data type; 1 to 9,999
  and 20,001 upward are open to customisers, 10,000 to 20,000 are reserved for Tally's own shared development, and 1 to 29 are used
  by default TDL **[W]**.
- **Attaching:** a field names the UDF in its **Storage** attribute and the value is stored against the object that is "in context"
  (the company, a ledger entry, and so on); the value is read back with a dollar prefix. A UDF **does nothing until a field or report
  uses it, and does not exist until a value is saved** **[W]**.
- **Shapes:** a simple UDF (one value, or several by repeating a line), and an aggregate UDF (a record, used for subforms such as
  bill-wise details) **[W]**.
- **Customisation cannot invent a new kind of master** (a new "Ledger"-like object); it only extends the existing ones **[Ws: TDL FAQ
  summary]**.

### 2.2 The levels a UDF can attach to

| Level | Evidence | Tag |
|---|---|---|
| Company | The UDF page's own example (a report on the company stores against the company) | [W] |
| Ledger entry (one line of an accounting voucher) | The UDF page's other example; Tally's voucher-structure page lists ledger entries as a first-level collection, each with bill allocations and cost allocations | [W] |
| Voucher header | The Day Book filter FAQ stores a method at the voucher level; the report-filter docs use Voucher as the primary object; Tally's own invoice-customisation example adds UDFs to the narration part for bank and cheque details | [W] |
| Inventory entry (an item line) | The voucher-structure page lists inventory entries as the other first-level collection; the invoice page says a new column must be added to every one of the line definitions. That a UDF is stored here follows the same object rule | [W] for the structure, [J] for the UDF |
| Batch allocation (inside an item line) | Stores **string UDFs only**; a number or quantity UDF needs a hand-written method on the batch object before a report can read it | [W] |
| Masters (ledger, stock item and their sub-lists) | A UDF added to the price-list report is not saved to the stock item by default; persisting it takes UI, object storage and a form-accept function. A plain ledger or stock item master form is the standard case | [W] for the price-list FAQ, [J] for the plain master |

### 2.3 How a field is shown, printed, reported and filtered
- **Shown:** a developer adds a field to the form and points its Storage at the UDF. A pick list of stored values is built from a
  collection and shown with the Table attribute **[W]**.
- **Printed:** nothing prints by itself. Tally's own page says to find **every** definition the new column touches (the heading
  line, item lines with and without a class, the alternate-quantity line, the batch lines, totals) and add the field in each **[W]**.
  Tally's own example adds "marks" and "number of packages" columns this way.
- **Reported:** the report's collection must fetch the stored value, and a column must read it. For a batch-level value the FAQ shows
  a columnar report fetching the batch methods explicitly **[W/Ws]**.
- **Filtered:** TallyPrime Release 3 documents a report-filter button and a collection of filter objects. A developer adds an object
  for the new UDF (its display name, the storage name searched, the data type that decides the match operators, a primary object
  type such as Voucher, and a chain of collections to search). The example adds a Salesman filter on vouchers **[W]**. Older pattern
  for the Day Book: hook the filter field, define an object, add a fetch **[W]**.
- **Validated:** TDL has field-level Validate, Unique, Notify and Control, and form-level Control; the page I read does not describe a
  dedicated "mandatory" attribute, so I do not claim Tally lacks one **[W]**.
- **Left behind:** if the TDL is removed, the values stay in the data. Tally's XML carries the **original index number** on the UDF
  tag, which Tally says helps recover the data when the TDL is unavailable **[W]**.

### 2.4 What Tally gives without TDL, and where it stops
- **Cost centres and categories** are the built-in dimension. Tally's own help uses salesmen as the cost-centre example and a job
  or site as the other **[Ws/W]**. Limits Tally states: allocation is against **ledger entries of accounting vouchers** (not for
  managing stock); a delivery note may carry one, a receipt note may not; **TallyPrime has no cost-centre-wise Profit & Loss or
  Balance Sheet** (the help points to add-ons on TallyShop); ledger reports do show the cost-centre detail; cost-centre classes cannot
  be imported between companies; no sub-category under a category **[W: TallyPrime cost-centre FAQ]**. Tally's job-costing guide
  concedes the per-job view is not a formal P&L and tells the user to export and structure it **[W]**.
- **Stock groups and categories** are Tally's own answer to colour and size: its public page tells users to organise stock by
  product type, brand, colour or size with as many groups and categories as they like **[W]**. A partner article admits Tally does not
  treat size and colour as separate stock, and lists three workarounds: an item per size, colour and size inside the item name, or
  batches **[W, vendor-authored]**.
- **Additional description** for stock items (a free-text line per item, switchable per invoice and per master) **[Ws]**.
- **Dispatch and e-way bill details** (LR/RR number, vehicle number, transporter, distance) are built in **[Ws/W]**.
- **Report filter / value filter** exists for users, but a **new** field must be registered with it by TDL **[W]**.

### 2.5 Why partners sell TDL customisation [J, with evidence]
- Tally itself points customers to authorised partners and to TallyShop add-ons for anything beyond configuration, and names
  TallyPrime Developer as the build tool **[W: Tally customisation page]**. There is no self-service alternative to point to.
- Each field is billable work because it is several edits (form, every invoice line, report, filter, sometimes validation) **[W]**.
- Upgrades re-open the work: Tally says old TDLs mostly load in TallyPrime but the default TDL changed a lot, some customisations
  may break and should be tested; buttons, menus, column-report buttons and the F11 add-on screen all changed **[W]**. A posted job
  on a freelance site asks for exactly this (migrating an invoice TDL from Release 2.1 to 7) **[Ws]**.
- The market is visible: partner pages advertise add-on catalogues (one claims 500+ built and a ready-made add-on from Rs 2,999;
  self-reported, unaudited **[Ws]**) and TallyShop sells partner add-ons in Silver and Gold editions **[Ws]**.
- **No counts of partners, TDL jobs or revenue were found** **[unverified]**.

### 2.6 The six customisations buyers ask for most (built from what partners and vendors name; no frequency data)

| # | Customisation | Level | What the sources say | Already built into TallyPrime? |
|---|---|---|---|---|
| 1 | Extra invoice header references: buyer order or PO number, a custom reference, bank or cheque details, declarations | Voucher header | Tally's own UDF example is cheque number, date and bank on payment and receipt vouchers with a print on/off switch; partner pages list buyer order or quotation reference and "custom reference fields" **[W]** | Partly (order details exist; a free reference is TDL) |
| 2 | Transport: transporter, vehicle number, LR/GR number, delivery note, e-way details | Voucher header | Partner invoice page lists transporter, vehicle, LR and e-way details **[W]** | **Yes, in dispatch and e-way bill details** **[Ws/W]**; BUSY has a separate transport window **[W]** |
| 3 | Extra item-line columns: marks and packages, remarks, size, colour, MRP, secondary unit, batch and expiry | Inventory line (and the stock item) | Tally's own example (marks, packages) and partner pages (batch, expiry, MRP, secondary unit, remarks) **[W]**; garment partners describe colour and size via item names or batches **[W]** | Batch, expiry, alternate units yes; colour, size, marks no |
| 4 | Salesman, zone, region, route, as a field **and** a report (sales and outstanding by it) | Voucher header, party master | Tally help uses salesman cost centres **[Ws]**; a partner lists a salesperson-wise performance report **[Ws]**; the Tally report-filter example is a Salesman filter **[W]** | Only as cost centres (accounting-voucher ledger entries); no cost-centre P&L |
| 5 | Project, site or job costing | Ledger entry (cost centre) | Cost centre per job, Cost Centre Breakup and Summary, net balance as the job's profit **[W]**; Tally's construction page lists project-specific cost centres, budgets, subcontractor tracking, site material transfer; retention, RA billing and works-contract GST are **not named** there **[W]** | Yes, with the limits in 2.4 |
| 6 | Extra party details: drug licence, FSSAI, extra GSTIN or PAN-like ids, contact roles, credit notes about the party | Ledger master | Partner invoice page names drug licence and distributor details for pharma **[W]**; Vyapar's own examples are licence number, PAN, drug licence **[W]** | No free fields |

Other requests partners advertise that are not fields but travel with them: custom print formats for invoices, purchase orders,
challans and receipts; summary, detailed and matrix reports; field-level validation; who may enter or view a field; import from other
software; WhatsApp or SMS **[W: tallyplanet.com]**.

### 2.7 What this means for a switcher
- A customer arriving with TDL fields has **data we cannot currently read**: the importer drops what it does not understand
  **[C]**, and the file's UDF tags carry a descriptive name, a type and the index **[W for the index; the tag shape is [unverified]]**.
- A TDL add-on also **stops working** when the customer leaves Tally, so the first thing they ask is where their zone, vehicle or
  colour went. Mapping UDFs in during import is the answer; asking them to retype is not.

## 3. Competitors

### 3.1 BUSY
- **Voucher optional fields** (the BUSY equivalent of a header UDF): configured per voucher type and series, up to **20** per
  voucher; a name and a type (text, numeric, yes/no, date, numeric with decimals); a flag to keep a **master list** so the values can
  be picked from a dropdown and created on the spot with F3; they appear after the narration; printing is a tick-box in the invoice
  print settings **[W for the 20, the setup path and the position; Ws for the types, F3 and the print tick]**. A recent release
  added a bulk "update voucher optional fields" option **[Ws]**.
- **Item additional fields:** configured in Master Configuration; one Q&A says up to **10**, text type, with the same master-list
  flag **[Ws; the feature is described on the BUSY Q&A page for master configuration, the 10 and the type are not on the pages I read]**.
- **Parameterised details** (variants): up to **five** parameters (size, colour, style, design, fit and so on), each **open** (typed
  in the voucher) or **predefined** (picked from a list), a character limit each, optional MRP, price and barcode; a window opens after
  the quantity on an invoice; purchases are typed, sales are picked; MRP can differ per variant; a **Parameterized Stock** report filters
  by item, group or combination; a separate barcode-number stock report tracks single units **[W]**.
- **Reports:** a user adds a **Custom Column** to a register (a name, heading, position, type, width, total on/off) and picks a data
  field such as a header-level voucher value; the Basic edition uses a "Columnar Reports" route instead **[W for the steps; Ws for the
  edition split and for the item-master and account-master categories]**. F7 opens a filter in a report **[Ws]**; I could not confirm
  that an optional field is filterable.
- **Transport** (GR/RR, vehicle) has its own window rather than optional fields **[W]**.
- **What I could not confirm:** whether optional fields can be mandatory, and any account-master field limits **[unverified]**.

### 3.2 Marg ERP
- **Pharma Distribution** edition listing: batch- and salt-wise stock, expiry management in every edition, credit limits; the **Silver**
  edition adds multi-store, order and challan entry, **salesman (M.R.), area and route-wise reporting**, self-configurable invoice
  formats and approval flows; Gold adds purchase import, push sale, live credit limits, claims, tagging and collection. The Retail edition
  lists salesman, area and route reporting in Silver and barcode and label printing **[W: third-party directory]**.
- **User-defined fields: not mentioned on either listing, and I found no Marg help page**, so I make no claim about whether Marg
  has them **[unverified]**.
- [J] The lesson is positional: Marg sells the salesman, area and route cut as **built-in reports on fixed fields** (and tiers them by
  edition). A Marg customer expects that cut to exist; a generic field screen is how a customer on another vertical gets the same.

### 3.3 Vyapar
- **Party additional fields:** Settings, Party; up to four, three taking text or numbers to 30 characters and one a date; each can be
  named and switched to print **[W: Vyapar blog]**.
- **Transaction additional fields:** Settings, Transaction, Additional field; the user names each and **chooses which transaction types
  show it**; they print on invoices **[W]**. Another Vyapar post says two at firm level and four at transaction level **[Ws]**; the two
  posts do not agree and neither is dated.
- **Item additional fields** (a tracking set): MRP or price, batch number, manufacturing and expiry dates, serial or IMEI, size, each
  switchable and renamable **[W]**. Vyapar's inventory page says custom fields for size, colour or batch can be added and lists
  "Item Additional Fields" under its **Silver** plan, but says **nothing about reports or filters on them** **[W]**.
- **Type set is tiny** (text up to 30 characters, number, date). No list type, no mandatory flag was mentioned **[W]**.

### 3.4 Zoho Books: custom fields
- **Per module**, from Settings (the pages use Customers and Invoices as the examples); a new field takes a label and type **[W]**.
  Types named: single and multi-line text, email, URL, phone, number, date; a pricing table adds decimal, amount, percent, date-time,
  dropdown, lookup, attachment, multi-select, formula, unique and AI fields **[W]**.
- **Options on a field:** help text; data privacy markers (PII, health data, with encryption); unique; an input pattern; a default
  value or **inherit from a related module** (a customer's value defaulting onto an invoice); **mandatory**; **show in PDFs**; **role
  access** (read and write, read only, hidden); active or inactive **[W]**. For invoices the KB shows the two switches "Is Mandatory" and
  "Show in All PDFs" **[W]**.
- **Item custom fields become columns in the transaction item table:** set "show when creating transactions", "show in all PDFs" and
  the transaction types **[W]**. Community notes (undated) say the column always sits before Quantity and cannot be hidden from a
  customer's PDF without a custom template **[Ws]**.
- **Caps:** by field type and plan, none on Free. In the Australian comparison table: Standard and Professional allow 5 dropdown, 20
  string, 12 number-or-date, 4 decimal, 5 lookup, 3 boolean fields; Premium, Elite and Ultimate allow 20 dropdown, 50 string, 30
  number-or-date, 10 decimal **[W, AU edition; India not read]**. [J] A clothes dealer with Colour, Size, Zone and Brand has used four
  of five dropdowns on a lower plan.
- **Reports:** the custom-reports page **does not mention custom fields at all**; the custom-views page does not say whether a custom
  field can be a column or a filter; a single unanswered, undated community post says custom fields cannot be added as columns in the
  estimate and invoice custom views **[W]**. So whether a Zoho custom field is reportable is **not documented**, and the one signal
  points to "not as a column in views" **[unverified]**.

### 3.5 Zoho Books: reporting tags (the "profitability tag" the owner asked about)
**Naming note [J]:** no Zoho page I read uses the words "profitability tag". Reporting tags are the feature that matches the owner's
description (create once, attach to many transaction types, report by it). Zoho's Profit by Item and project reports are separate.

- **What it is:** a custom category with options you define, attached to records and transactions so reports can be filtered and
  compared by department, location or similar **[W]**.
- **Two generations of the help page:**
  - *Older page (India edition):* up to **10 tags** per organisation and **500 options** per tag; applies to contacts, items, quotes,
    sales orders, invoices, credit notes, expenses, purchase orders, bills, vendor credits, journals; tagged **at line level** on
    transactions; edit, delete or mark inactive; **mandatory not stated**; filter by tag from Customise Report on the Business Overview
    reports (Profit and Loss is the page's example) **[W]**.
  - *Advanced Reporting Tags page (Canadian edition):* create a tag with a name and options, **choose the modules** (Sales, Purchases,
    Journals, Inventory Adjustments, Banking), **choose transaction level or line-item level** (the Items module must be line level),
    **mark it mandatory** (needs one active option), set a **default option**, nest options **up to five levels**; an item's tag
    **auto-applies to its lines** and a customer's or vendor's tag defaults onto their transactions, always changeable; **filter
    Profit and Loss by a tag option and compare across options**; **visibility mapping** (an option shows only when certain options
    of other tags or locations are chosen; up to five filters and 200 mapped options per tag); **access control** (restrict which records
    a role or user sees by tag option; one tag per organisation; up to 500 options per user); plan limits apply; **once Advanced is
    switched on it cannot be switched off** **[W]**.
- **Report reach is uneven:** a Zoho KB article says tags cannot customise the **Invoice Details** report because they sit on invoice
  lines, but the **Receivable Details** report can group by and filter by them; whether a tag can be a *column* is not stated **[W]**.
  Zoho's custom-report page lists "reporting tags" among the advanced filters of an existing report **[W]**.
- **Plan:** custom fields and reporting tags are both absent on Free and present on the paid tiers; the exact tag and option caps per tier
  were not on the table I read **[W]**.

### 3.6 QuickBooks Online
- **Custom fields:** limits per transaction by plan: Simple Start 1, Essentials 4, Plus 4 (sales forms; Plus also 4 on purchase orders),
  **Advanced 12**; no limit on how many transactions carry them; a name, a data type and a category (customer, transaction, supplier or
  project; the customer, supplier and project categories only on Advanced **[Ws]**); the user picks which forms show it and a **Print on form** switch decides whether it
  prints; hiding a field keeps its data and it still shows on old forms **[W]**. A dropdown holds up to 100 values **[Ws]**.
- **Reports:** a custom field can be ticked into some transaction reports through Customise and Filter (for example Transaction List by
  Customer) **[Ws, a third-party mirror]**; users report blank columns on others **[Ws]**.
- **The documented gap:** Intuit's team said on 20 April 2025 there is **no Profit and Loss by custom field** and suggested exporting
  and pivoting, or an App Store tool **[W]**. Separately, Intuit **retired tags for reporting from 16 May 2025** and migrated up to 100
  tags into custom fields, copying the names into a dropdown but **not** onto the already-tagged transactions; the team agreed a P&L
  by custom field would be useful and pointed to the feedback button **[W]**. [J] Intuit swapped a reporting dimension for a
  descriptive field and left users without the report; that is the clearest evidence in this doc that the two jobs need different
  reporting.
- **Classes and locations (the dimensions):** Plus and Advanced only; a class can be one **per whole transaction or one per row**; a
  location is **one per transaction**; classes are saved on the lines only, so **the Balance Sheet cannot be filtered by class** while
  locations (saved on header and lines) can filter both statements, with receivables and payables landing in "not specified" in some
  location views **[W]**. Intuit says classes and locations suit large categories and points to its Projects feature for jobs **[Ws]**;
  its Enterprise Suite uses the word "dimensions" **[W]**.

### 3.7 Xero
- **Tracking categories:** two active at a time and four in total; options up to 100 per category in third-party posts (a conflicting
  500 is also quoted); applied **per line** on invoices, bills and spend money; reported through the Profit and Loss tracking filter and a
  Tracking Summary; archive an option rather than delete to keep history **[Ws: Xero Central summary plus third-party posts]**.
- **Custom fields on invoices or contacts: not available.** The idea has been open since October 2012 with 704 votes; a Xero community
  manager wrote on 21 November 2025 that it is not on the roadmap, and earlier (2023) pointed to the contact Notes box. Users work
  round it with the reference field, the account number, extra description lines and per-client templates **[W]**.
- [J] Xero shows how far a product can get with one dimension and nothing else, and how loudly users complain about it.

### 3.8 Odoo
- **Studio** (the no-code builder) offers 20 field choices over 15 technical types: text, multiline text, integer, decimal, monetary,
  HTML, date, date-time, checkbox, selection, priority, file, image, signature, and relational fields (many-to-one, one-to-many, lines,
  many-to-many, tags, related) **[W]**. Studio appears to be an **Enterprise-only** product **[Ws, vendor blog]**.
- **Views:** Studio can set list defaults, add custom filters and autocomplete fields to the search view, and configure pivot (row and
  column grouping, measures) and graph views; whether a Studio-created field is offered for grouping is **not stated** **[W/unverified]**.
- **Analytic accounting (the dimension):** analytic **plans** (nestable) hold analytic **accounts**; each plan's **applicability** is
  Optional, Mandatory or Unavailable by default with per-line overrides (mandatory blocks confirming an entry that lacks it);
  **distribution** on invoice and bill lines splits an amount by percentage across accounts and **across several plans on one line**;
  **distribution models** apply a preset automatically from account prefix, partner, product or company, and can be chained and
  sequenced; distributions can be edited in bulk from journal items **[W]**. Analytic reports (pivot and graph by plan) were in the
  truncated part of the page **[unverified]**.

### 3.9 Pipedrive (the CRM the owner pointed to for the grid; useful for "where it is visible")
- Custom fields on deals and leads (shared), people, organisations, products and projects; a field's type cannot be changed after creation;
  an **Important** setting (Growth plan) only highlights missing data, a **Required** setting (Premium on one page, Professional and higher
  on another, so plan-gated and the page names disagree) blocks saving; per-field **places shown**: the detail view, the **list view
  columns**, the add view, per pipeline (Premium); read-only per user group (Premium); admins see field statistics (created, updated,
  where used) before editing or deleting; the field is offered to filters, Insights reports, automations, web forms, email templates
  and imports **[W]**. Plan caps exist and are not stated on the page **[W]**.
- [J] This is the best model for the owner's "where it should be visible" wording: **one field, a short list of places** (entry form,
  list column, print, filter, API), each a switch.

### 3.10 Does a field added to the object reach the reports by itself?

| Product | Entry screen | Print | Register or list column | Filter | Financial statements |
|---|---|---|---|---|---|
| Tally (TDL) | Developer adds it | Developer edits each invoice line | Developer fetches it in the collection | Developer registers a filter object | Not for UDFs; cost centres only, no cost-centre P&L |
| BUSY | Per voucher series, after narration | Tick in print settings | A **Custom Column** the user builds (edition gated) **[Ws]** | F7 filter, optional-field support unconfirmed | Not stated |
| Vyapar | Choose transaction types | "Show in print" | Not stated | Not stated | Not stated |
| Zoho custom field | "Show when creating transactions" | "Show in all PDFs" | Item field becomes an item-table column; otherwise **not documented** | Not documented | Not documented |
| Zoho reporting tag | On the line or header | Not stated | Not stated as a column; Receivable Details groups by it | Customise Report filters, Receivable Details | **P&L filter and compare by option** |
| QuickBooks custom field | Per form | "Print on form" | In some transaction reports via Customise | Via report filter | **No P&L by custom field** |
| QuickBooks class / location | Header or row | n/a | n/a | n/a | **P&L by class; P&L and Balance Sheet by location** |
| Xero tracking | Per line | n/a | n/a | n/a | P&L tracking filter |
| Pipedrive | Add view and detail view | n/a | **Column picker offers it at once** | Filters and Insights | n/a |
| Odoo Studio | You edit the view | Report templates separately | You add it to the view | Search view if added | Analytic reports by plan |

[J] Nobody makes a new field appear on standard layouts automatically. Pipedrive is closest (offered at once in the picker); the
accounting products make it a deliberate act.

## 4. Indian small-business cases

| Case | Typical fields | Where they attach | Type | What the market does | Comsky today **[C]** |
|---|---|---|---|---|---|
| **Garment, footwear** | Colour, size, design number, brand or collection, season, fabric, fit | Item master, the item line, and stock identity | Pick list (colour, size), text (design no.) | BUSY: up to five parameters with stock, MRP and barcode per variant **[W]**. Tally: item per size, colour in the name, or batches **[W]**. Vyapar: size and MRP in its item tracking set **[W]**. A size-colour grid (sizes against colours, 5 sizes by 4 colours is 20 cells) is the vendor-standard entry style **[Ws]** | Item alias, part no., category; batches with mfg and expiry; item custom fields exist but are not columns yet; **no variant stock** |
| **Pharma, FMCG** | Batch, expiry, MRP, salt or composition, drug licence number, schedule | Item line (batch, expiry), item master (salt, MRP), party (licence) | Date, text, number | Marg lists batch- and salt-wise stock and expiry in every edition **[W]**; Vyapar uses additional fields for licence numbers **[W]** | Batch, mfg and expiry on lines **[C `InventoryLine`]**; **no MRP, salt or licence field exists** (grep of `types.ts`) |
| **Hardware, building material** | Size or gauge, grade, brand, rack location, selling unit (kg, piece, metre), customer-wise price list | Item master, price list | Pick list, text | Vendors stress mixed units on one invoice, a different GST rate per line, contractor credit and godown stock **[Ws, vendor marketing]** | Alternate and compound units, price levels, godowns exist; size, grade, brand, rack would be item custom fields |
| **Jewellery** | Purity or karat, gross, stone and net weight, rate per gram, making charge (fixed, percent or per gram), wastage, HUID, old-gold exchange | Item line, with **amounts computed from the numbers** | Pick list, number, text | An India billing vendor lists these invoice fields **[W]**; I found no Tally or TallyShop listing for karat or weight **[Ws]**; an ERPNext consultancy proposes custom fields plus a daily metal-rate table because stock items carry one standard rate **[Ws]**. Zoho has a **formula** field type (a few per plan, Premium and up) **[W]** | Nothing; this is the case where a label is not enough (a calculation is needed) |
| **Distributor** | Zone, route or beat, salesman, area, target | Party master (the default), voucher header, report grouping | Pick list | Marg: salesman, area and route-wise reports built in **[W]**; Tally: salesman as a cost centre or a partner add-on **[Ws]** | No salesman, zone, route or area field anywhere **[C `types.ts`]**; cost centres exist |
| **Contractor, service project** | Site, project, work order, RA bill number, retention | Voucher header, every cost line (dimension) | Pick list, text | Tally: cost centre per job, cost-centre breakup and summary, per-job net as profit; no RA bill or retention on its construction page **[W]** | Cost centres on ledger entries plus a Cost-Centre P&L Breakup (P&L-nature lines only) **[C `ui/reports/costpl.tsx`]**; RA and retention would be custom fields |

[J] Three of the six cases (garment, jewellery, hardware) are **line-level** needs, and the planned report grid has a deliberate
rule that a one-to-many value never makes a column on a one-row-per-voucher report, so a line-level custom field needs an item-wise
register or a scalar projection ("Multiple") **[C `docs/research/pipedrive-list-ux.md` section 4 and 6.1]**.

## 5. What Comsky Books has today

All **[C]**, read in the working tree on 2026-10-21.

- **Definition:** `CustomFieldDef` (`src/types.ts` ~411): `objectKind` is `'ledger' | 'stockItem'` only; `key` (stable slug), `label`,
  `type` (text, number, date, toggle, select), `options` for select, `required`, `order`. Dexie v4 table `customFields`; values live in
  an optional `udf` record on `Ledger` and `StockItem` (`types.ts` ~366 and ~526), synced as part of the object payload and visible
  to the REST layer (`docs/ARCHITECTURE.md`).
- **Where it shows:** a block titled "Additional Details" at the end of the Ledger and Stock Item master forms
  (`src/ui/masters.tsx`, `src/ui/inventory/masters.tsx`, `src/ui/udf.ts`). The admin screen is `src/ui/admin/CustomFields.tsx`
  (Gateway Company menu D; Go To).
- **Required is not enforced.** The admin table prints a Yes and `udf.ts` builds form fields without any required rule; the other
  `required` properties in the tree (the CSV import mapper, the saved-view sanitiser) belong to those features, not to custom fields.
  A "mandatory" custom field today is cosmetic.
- **Small gaps in the same files:** select options are one comma-separated text box (no list editor, no rename or reorder);
  `collectUdf` stores a number only when non-zero and a toggle only when Yes; deleting a definition leaves the values on records with no
  guard; the key is derived from the label and the type cannot be edited.
- **No voucher or line level:** `Voucher` and `InventoryLine` carry no `udf` (a grep of `types.ts` finds two `udf` properties, both
  on masters). The roadmap addendum already lists `tdl-customization` as missing, priority P3, with a scoped slice "add voucher to
  UdfKind and Voucher.udf" (`docs/research/ROADMAP-ADDENDUM.md`). The owner's request lifts it.
- **No report reads a custom field.** The planned grid already reserves a "Your custom fields" group with ids `udf:ledger:<key>` and
  `udf:stockItem:<key>`, party and item fields reaching voucher and stock reports through the relation, flagged "(current)" because they are
  read at report time; ids are never renamed, a deleted field's column is dropped, a stale **filter** is kept and matches nothing
  (`docs/research/pipedrive-list-ux.md` sections 5 to 6.3 and risk R7; `docs/research/wave-a2-plan.md`). Saved views already store
  columns, filter and basis (`src/core/savedViews.ts`).
- **Import and export:** the Tally XML importer drops fields it does not read, and the exporter lists "masters with custom fields"
  among what the file does not carry (`src/core/tallyImport.ts`, `src/core/tallyExport.ts`). The exporter does open its messages with
  Tally's UDF namespace declaration (`xmlns:UDF="TallyUDF"`).
- **The dimension that exists:** `VoucherEntry.costs?: CostAllocation[]` (a cost-centre id and an amount **on a ledger entry**),
  `CostCentre.categoryId`; reports `costpl.tsx` (income and expense per cost centre, P&L-nature allocations only), `costanalysis.tsx`,
  `costcentrereport.tsx`. That is Tally's model, with one thing Tally says it lacks (a cost-centre P&L).
- **Fixed fields for what Tally partners most often add:** `Voucher.refNo`, `orderRef`, `supplierInvNo`, `placeOfSupply`; and, **uncommitted
  and in flux**, a per-voucher dispatch block (`deliveryNoteNo`, `dispatchDocNo`, `dispatchedThrough`, `destination`, `lrNo`, `vehicleNo`,
  consignee and dispatch-from addresses) in `src/core/dispatchTypes.ts` with its own tests.
- **Absent as fields anywhere:** MRP, salt, drug licence, salesman, zone, route, area, colour, size (grep of `types.ts`).

## 6. [J] Synthesis

### 6.1 Comparison (what each product lets a user do, from sections 2 and 3)

| Product | Object levels | Field types | Mandatory | In a report | Filterable | Caps and other limits |
|---|---|---|---|---|---|---|
| **TallyPrime (TDL)** | Any object: company, masters, voucher, ledger entry, item line, batch (string only) | String, amount, quantity, rate, number, date, logical, aggregate | Via TDL controls | Only if the developer adds it | Only if a filter object is registered (R3) | 65,536 per type; **needs a developer and re-testing per release** |
| **BUSY** | Voucher header (optional fields), item master, item line variants | Text, numeric, yes/no, date (voucher); text (item); five variant parameters | Not stated | Custom column on a register (edition gated) | F7; optional-field support unconfirmed | 20 voucher fields, about 10 item fields, 5 parameters |
| **Marg** | Not documented | Not documented | Not documented | Built-in salesman, area, route reports | Not documented | Edition tiers gate the reports |
| **Vyapar** | Party, transaction (by type), item tracking set | Text (30 chars), number, date | Not stated | Not stated | Not stated | About 4 party, 4 transaction fields; item fields on Silver |
| **Zoho Books custom fields** | Per module: contacts, items, invoices and others; item fields become line columns | Text, number, date, dropdown, lookup, formula, attachment and more | Yes | **Not documented** (one report of "no" for views) | List views by criteria | By type and plan (5 or 20 dropdowns) |
| **Zoho reporting tags** | Header or line, per module; items, customers, vendors default them in | Pick list with nested options | Yes (advanced) | Filter and compare in P&L; some reports group by it | Yes | 10 tags by 500 options (older page); five option levels |
| **QuickBooks** | Transaction (all plans); customer, vendor, project (Advanced); class per row; location per transaction | Dropdown (100 values), text, number, date | Class warn or require **[Ws]**; field flag **[Ws]** | Some transaction reports; **no P&L by custom field** | Report filters | 1, 4, 4, 12 fields by plan |
| **Xero** | Line, as a tracking category; no custom fields | Option list | No | P&L and Tracking Summary | Yes | Two active categories |
| **Odoo** | Any model (Studio); analytic plans on lines | 20 Studio kinds; plan applicability | Field required; plan Mandatory | You configure views; analytic reports **[unverified]** | Search view you configure | Studio is paid |
| **Comsky today** | Ledger, stock item | Text, number, date, yes/no, list | **Stored, not enforced** | **None yet** | **None yet** | None |

### 6.2 The minimum a Tally switcher expects [J]
A switcher is comparing against "my partner added it in a week". The floor is:
1. **Add a field on the Sales (and Purchase) voucher screen without a partner**: text, number, date, yes/no, and a pick list with a fixed set
   of values (North, South, East, West); choose which voucher types show it.
2. **Add a column on the invoice item line** (remark, marks, packages) and a **field on the stock item** (colour, size) and on the **party**
   (licence number, zone, salesman).
3. **Print it** on the invoice when the user says so (every competitor offers a print switch).
4. **See it as a column in the Sales Register and Day Book, filter by it, sort by it, export it**, and drill through. This is the step Tally
   makes expensive and the one Zoho does not document.
5. **Mandatory** and a **default value** (a recalled or batch-held value must be visible, per the existing gotcha).
6. **Vehicle, LR, transporter as ready-made fields** (wave 3) so the commonest TDL job needs no custom field at all.
7. **Their own TDL fields visible after import** (at least readable, ideally mapped).
8. **Salesman or zone wise sales and outstanding**, whether through a field plus filter or a dimension.
Not expected on day one: formulas, lookups to other masters, approvals, per-role field access. Zoho and Odoo have them; Tally users
did not have them either unless they paid for a larger TDL.

### 6.3 What would make Comsky Books clearly better [J]
1. **No ceiling.** Competitors cap by plan or by a small fixed number (Vyapar about 4, BUSY 20, QuickBooks 1 to 12, Zoho by type). Comsky
   plan limits are display-only today **[C]**; keep only technical limits and say so. A clothes dealer should not be told he has used his
   five dropdowns.
2. **Three levels on one screen:** master, voucher header and item line, with the same editor, per field: type, list values, mandatory,
   default, the **voucher types it applies to**, and the **places it shows** (entry, print, register column, filter, API). That is Zoho's
   per-module settings plus Pipedrive's "places shown", on the Tally keyboard model.
3. **Reportable by default, displayed by choice.** Every field is offered in the column picker and filter of each report that reaches its
   object, with Standard layouts left exactly as they are (section 6.4). Beats Zoho (undocumented), QuickBooks (not in P&L), Vyapar (not
   stated), Tally (never).
4. **A dimension that reaches the statements.** Tag a voucher with Zone or Project once and have it reach **every ledger leg**, so the P&L
   and, with a stated rule for receivables and payables, the Balance Sheet can be filtered and compared by it. Comsky already has a
   cost-centre P&L that TallyPrime lacks **[C vs W]**; QuickBooks' own help shows the failure mode to avoid (class not on the header, so
   no balance-sheet cut; "not specified" rows). Add Zoho's best parts: a tag's default from the party or the item, nested options, an
   "untagged" bucket, tag-restricted access; and Odoo's idea that a plan can be **mandatory** per voucher type.
5. **Defaults flow and are snapshotted.** Party zone defaults onto the voucher; item colour defaults onto the line; the value is **copied
   onto the document when it is posted** so editing the master later cannot re-label last year's sales (the same reason `InventoryLine`
   snapshots GST treatment **[C CLAUDE.md]**).
6. **A Tally import that keeps the customer's UDFs.** Read the namespaced UDF tags and their indexes, show a preview of "these 6 fields
   found, map to new custom fields", and fill the values. No competitor does this. Needs real sample files before it is trusted.
7. **History-safe by construction:** the type cannot change once data exists (Pipedrive's rule); deleting a field **deactivates** it and
   keeps the data (QuickBooks keeps data when a field is hidden); renaming a list value is blocked or propagated, never silent; a column for
   a removed field disappears while a saved filter on it matches nothing (the existing rule).
8. **The vertical starter packs (cheap, high perceived value):** a one-click set for garment (colour, size, design no.), pharma (MRP, salt,
   drug licence), distributor (zone, route, salesman), jewellery (karat, gross, stone, net weight, HUID), contractor (site, RA bill no.).
   No competitor ships these as data in a no-code product; Tally partners sell each as a TDL.
9. **A variant path, kept separate:** colour or size with stock per combination is a stock-engine feature (BUSY's five parameters is the
   benchmark). Do not promise it from custom fields.
10. **A later step:** a computed field (jewellery net weight times rate); Zoho offers a handful on its upper tiers.

### 6.4 The owner's open question: does a field added on a voucher type appear in reports by itself?
**Rule [J]:** *added to the object means available, not displayed.*
- At creation the field gets one setting, **Available in reports and filters**, **on by default**. On, the field is offered in the column
  picker and the filter builder of every report whose rows are that object or reach it (Sales Register, Day Book, Ledger Vouchers, Bills,
  Group Summary and the item reports for an item field), and in saved views, the API and exports.
- **No standard layout changes.** The Standard column set of every report stays exactly today's (the same promise as the planned grid:
  "Standard layout identical to today"). A person who defines nothing sees nothing new **[C Phase 27 gotcha: customisation is opt-in,
  asserted by identity]**.
- **Print is separate and off by default.** A second switch, "Show on the printed invoice", and a third, "Show as a column on the invoice
  item table" for line fields. A company can choose a default column set for everyone through the existing company-wide saved views, which
  already exist and are administrator-gated **[C `Company.reportViews`]**.
- **A field defined for Sales only is offered only in reports that can contain a Sales voucher**, and a Day Book that mixes types shows a
  blank for vouchers that do not carry it, so the column can never mislead.
- **One more screen for the owner's zone example:** if he wants the **P&L by zone**, the field must be a dimension (6.3 item 4), not just
  a field. That is a separate switch, "Use as a reporting dimension".
Evidence behind it: Pipedrive offers a new field in the picker immediately **[W]**; Zoho and Vyapar make printing a switch **[W]**; BUSY
makes a report column a deliberate build **[W]**; QuickBooks and Zoho show that a field kept out of the statements frustrates users **[W]**.

### 6.5 Traps to design against (each tied to something this repo already learned)
- **A label cannot hold stock.** Colour as a custom field on `StockItem` gives a column and a filter; it does not give "stock of red kurtas".
- **A line field breaks a one-row-per-voucher report.** Use the existing scalar-projection rule ("Multiple") or an item-wise register.
- **Header-only tags cannot cut the Balance Sheet; line-only tags cannot appear on a header register** (QuickBooks, Zoho **[W]**). Decide
  and document which legs a dimension reaches.
- **A required field must gate every path that can create the record**: voucher Accept, Bulk Entry, CSV import, Tally import, the REST API,
  Alt+2 duplicate, ghost recall, batch holds. The existing gotcha says a guard that gates only the commit lets a duplicate or an empty
  value through other doors **[C CLAUDE.md Phase 20 to 21]**. A mandatory rule must also be **prospective**: it never invalidates a saved voucher.
- **Frozen field order.** A new voucher field must not move any existing field in the Enter walk; give it its own step or an Alt-key panel and
  assert the whole walk with the new fields removed **[C Phase 24 gotcha]**; ROADMAP-ADDENDUM already sketches a "Custom Details" step.
- **A form that rebuilds the object drops what it does not own.** Alter Voucher with a new `udf` on it needs the same
  resave-preserves-everything check the masters have (`master-resave.assert`) **[C]**, and the older client that does not know `udf`
  must not erase it (the Phase 27 lesson about readers that rebuild).
- **Hostile or garbage values.** A custom field value or label is typed text: strip format characters, cap length and option count, never
  evaluate it; a CSV or Excel cell that starts with `= + - @` is a formula **[C Phase 24 and 27 gotchas]**.
- **Alias and search.** A pick list value must be searchable like other pickers; a select list with hundreds of values needs search, not a
  popup of everything.
- **Stale references.** A saved view, filter or dashboard card that names a removed field must degrade the way the grid plan already says
  (column dropped, filter kept and matching nothing) **[C R7]**.
- **Export mapping.** Exporting a custom field to Tally XML needs a stable index per field; derive it once and store it, never recompute **[J]**.

### 6.6 Questions for the owner, and what to check in a live account
1. **Zone, project, salesman: do you want Profit and Loss and Balance Sheet by it, or only registers and outstanding filtered by it?**
   The first needs a dimension, the second a field.
2. **Colour and size: does the dealer need stock and rate per colour (variants), or a label to filter by?**
3. **Should the number of fields or tags ever depend on the plan?** Competitors do this; Comsky's limits are display-only today.
4. **Get three real TallyPrime XML exports from partner customers who run UDFs** (a garment trader, a distributor, a pharma stockist) and
   look at the UDF tags before designing the import mapping.
5. **In a live Zoho Books trial:** do custom fields appear as columns in Customise Report or in Create New Report; does an item custom field
   appear in "Sales by Item"; can a reporting tag be a column. **In a QuickBooks Advanced trial:** which reports list custom fields.
   **Ask a Marg dealer** what user-defined fields exist.
6. **Check the India edition's Zoho plan table** for custom-field and reporting-tag caps.

## 7. Not covered here
- **Grid and quick-save (owner items 1 and 2):** `docs/research/pipedrive-list-ux.md`, `docs/research/wave-a2-plan.md`,
  `src/core/savedViews.ts`. Section 5 says where custom fields plug into them.
- **A way back for mouse-only users (item 4):** not researched. One fact for whoever does: the breadcrumb text is deliberately **not
  clickable**, because a segment click would pop several screens at once and bypass each screen's own Esc guard, and the only crumb
  control today is the Hide/Show menu button **[C `src/ui/widgets.tsx` `StatusBar` comment]**; `st.pop()` exists in the store
  (`src/state/store.ts`) and, for example, the Custom Fields screen's Quit button calls it. A Back control would need to go through the same
  dirty-state guard as Esc (`src/core/navPlan.ts`, `src/state/dirty.ts`).

## 8. Sources

**Read in full (fetched), [W]:**

*Tally*
- https://help.tallysolutions.com/docs/td9rel54/tdlreference/user_defined_fields.htm
- https://help.tallysolutions.com/developer-reference/tally-definition-language/what-are-user-defined-fields-validations-and-controls-in-tdl/
- https://help.tallysolutions.com/how-to-customise-voucher-and-invoice-using-tdl/
- https://help.tallysolutions.com/what-can-be-customised-in-report-filter/
- https://help.tallysolutions.com/article/DeveloperReference/faq/6177.html
- https://help.tallysolutions.com/article/DeveloperReference/faq/6677.html
- https://help.tallysolutions.com/article/DeveloperReference/td9/faq/udf_values_in_batch_object.htm
- https://help.tallysolutions.com/working-of-tally-erp-9-customisations-with-tallyprime/
- https://help.tallysolutions.com/cost-centres-in-tallyprime-faq/
- https://help.tallysolutions.com/?p=179361 (e-way bill transport fields)
- https://tallysolutions.com/customization
- https://tallysolutions.com/mena/?p=165143
- https://tallysolutions.com/tally/how-to-track-job-costs-and-profitability-in-tallyprime/
- https://tallysolutions.com/accounting-software/construction/
- Partner and vendor pages: https://tallyplanet.com/tally-customization/, https://www.nakshatrainfo.com/blog/tally-invoice-printing-customization,
  https://www.tallyatcloud.com/articles/how-to-customize-tally-prime-for-clothing-business-complete-garment-industry-guide-2026,
  https://precisiontech.in/apps/tally/tally-tdl-addons/
- Buyer example: https://www.freelancer.de/projects/software-development/tallyprime-tdl-customization

*BUSY*
- https://busy.in/accounting-software/product-variant-inventory/
- https://busy.in/faqs/use-the-parameter-feature-in-busy-for-bifurcating-items-based-on-color-size-etc-answerid-36535/
- https://busy.in/faqs/configurations/voucher-series-configuration/15, /21, /27/, /31/, /71/
- https://busy.in/faqs/i-am-unable-to-add-a-custom-column-to-my-sales-register-for-tcs-amount-how-can-i-do-this-answerid-31000/
- https://busy.in/faqs/customize-purchase-order-invoice-formats-busy-including-adding-editing-columns-fields-answerid-36479/
- https://busy.in/faqs/features/inventory-parameter-batch-serial-number/29/
- https://busy.in/faqs/configurations/master-configuration/118/ (read; it discusses the master additional field but gives no figure)

*Marg, Vyapar*
- https://www.softwaresuggest.com/marg-pharmaceutical, https://www.softwaresuggest.com/marg-counter
- https://vyaparapp.in/blog/how-to-open-medical-store-in-india/, https://vyaparapp.in/free/inventory-management-software

*Zoho Books*
- https://www.zoho.com/books/help/settings/customization/custom-fields.html
- https://www.zoho.com/ca/books/help/settings/reporting-tag.html (Advanced Reporting Tags)
- https://www.zoho.com/in/books/help/settings/reporting-tags.html (older page)
- https://www.zoho.com/books/kb/reports/reporting-tags-in-invoice-details-report.html
- https://www.zoho.com/books/help/settings/customization/custom-views.html
- https://zoho.com/books/help/reports/custom-reports.html
- https://www.zoho.com/en-fr/books/kb/invoices/invoice-additional-fields.html
- https://www.zoho.com/us/books/kb/templates/display-additional-item-info.html
- https://www.zoho.com/au/books/pricing/pricing-comparison.html
- https://help.zoho.com/portal/en/community/topic/custom-view-and-custom-fields-on-zoho-books

*QuickBooks, Xero, Odoo, Pipedrive*
- https://quickbooks.intuit.com/learn-support/en-ie/help-article/customise-forms/add-custom-fields-sales-forms-purchase-orders/L6saJYHdg_IE_en_IE
- https://quickbooks.intuit.com/learn-support/en-ca/reports-accounting/quickbooks-online-are-replacing-tags-with-custom-fields/00/1539619
- https://quickbooks.intuit.com/learn-support/en-us/reports-and-accounting/how-can-i-run-a-p-l-report-by-custom-field/01/1546310/highlight/true
- https://quickbooks.intuit.com/learn-support/en-us/help-article/class-list/track-transactions-class/L927QQfNV_US_en_US
- https://quickbooks.intuit.com/learn-support/en-us/help-article/balance-sheet/run-balance-sheet-class-location/L2wj2NQT2_US_en_US
- https://productideas.xero.com/forums/967115-invoices-quotes/suggestions/44960422-contacts-custom-fields-on-invoices-and-contacts
- https://www.odoo.com/documentation/18.0/applications/studio/fields.html
- https://www.odoo.com/documentation/18.0/applications/studio/views.html
- https://www.odoo.com/documentation/19.0/applications/finance/accounting/reporting/analytic_accounting.html
- https://support.pipedrive.com/en/article/custom-fields

*India cases*
- https://mybillbook.in/blog/how-to-create-gst-bills-for-gold-and-silver-jewellery/
- https://mybillbook.in/s/billing-software-for-hardware-store/
- https://pixeltech.ai/blog/garment-erp-explained-style-bom-size-colour-matrix-cut-to-ship/

**Search summary only, [Ws] (the page itself not read):**
- Tally: https://help.tallysolutions.com/article/te9rel65/Voucher_Entry/Accounting_Vouchers/Configuring_a_Invoice.htm (additional descriptions);
  https://help.tallysolutions.com/docs/te9rel65/TSS/TallyShop.htm; Tally cost-centre report pages under
  https://help.tallysolutions.com/article/te9rel64/Reports/MIS_Reports/ (the fetched copy held only navigation); the TDL FAQ on object limits,
  https://help.tallysolutions.com/developer-reference/developer-reference-faq/tdl-faq/
- Xero: https://central.xero.com/0/article/Set-up-tracking-categories (fetch returned empty); https://www.databuzz.com.au/category/api/page/4/;
  https://www.fieldpromax.com/blog/xero-tracking-categories-for-job-costing-setup-and-limits
- QuickBooks: https://quickbooks.intuit.com/learn-support/en-uk/class-list/tag-transactions-in-quickbooks-online/01/636608 (fetch returned a landing page);
  https://www.cleverence.com/articles/quickbooks-documentation/use-custom-fields-in-reports-in-quickbooks-online-advanced-2749
- Vyapar: https://vyaparapp.in/blog/how-to-use-vyapar-for-your-restaurant-business-in-simple-steps/
- Odoo Studio edition: https://ecosire.com/blog/odoo-studio-customization-guide
- Pipedrive: https://support.pipedrive.com/en/article/required-fields
- Jewellery ERP field proposal: https://ecosire.com/zh/apps/erpnext/erpnext-jewellery-bullion
- BUSY 21 release notes: https://download2.busy.in/Busy21/ReleaseNotesBusy21v14.0.pdf

**Repo files read [C]:** `src/types.ts`, `src/ui/udf.ts`, `src/ui/admin/CustomFields.tsx`, `src/ui/masters.tsx`, `src/ui/inventory/masters.tsx`,
`src/core/tallyImport.ts`, `src/core/tallyExport.ts`, `src/core/savedViews.ts`, `src/core/dispatchTypes.ts` (uncommitted), `src/ui/dispatch/dispatchForm.ts`
(uncommitted), `src/ui/reports/costpl.tsx`, `src/ui/widgets.tsx`, `docs/research/pipedrive-list-ux.md`, `docs/research/wave-a2-plan.md`,
`docs/research/zoho-books-reference.md`, `docs/research/ROADMAP-ADDENDUM.md`, `docs/ARCHITECTURE.md`, `CLAUDE.md`.
