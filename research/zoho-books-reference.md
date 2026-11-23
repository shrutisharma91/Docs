# Zoho Books reference: reports, quick create, list screens, left menu, shortcuts and appearance

> **Status:** research only. Nothing here is built. Written 2026-10-19 for the "Zoho-style" wave (Wave A: reports hub,
> universal `+`, per-row `+` in the left menu, list-screen "New" button, saved filters; Wave B: configurable shortcut
> buttons and the admin settings window).
> **Method:** Zoho's own public help, knowledge-base and release-notes pages (zoho.com/books, zoho.com/in/books,
> help.zoho.com community product-update posts), read through a web fetch that summarises each page. Everything below is
> paraphrased; no Zoho text or screenshot is reproduced. Comsky facts were read from the working tree.
> **Evidence rule:** a statement without a tag is from a Zoho page cited in the area's source list. **[owner]** means it
> comes only from the owner's written description of his screenshots. **[unverified]** means I looked and could not
> confirm it from Zoho's public pages. **[inferred]** is my reading, not a Zoho statement.

## 0. Read this first

**What could not be checked, and why it matters.**
1. **The five screenshots are gone.** The temp folder named in the brief was cleaned (the `images/` folder does not
   exist; only the re-captured "before" shots of our own UI remain). Everything about the Reports Center left panel
   (Home, Favorites, Shared Reports, My Reports, Scheduled Reports, Advanced Analytics), the `+` mega-menu columns
   (General, Inventory, Sales, Purchases, Banking), the `+ New` split button and the icon-only collapsed rail rests on
   the owner's description. Zoho's public help does not describe those screens in that form.
2. **Zoho's help is regional and uneven.** The India, US, Singapore and other editions differ, the India report pages
   are incomplete, and a few pages contradict each other (custom-report availability, section 1.5). The summarising
   fetch can also drop detail. Treat this as a map to confirm in a live Zoho Books organisation before a spec is locked.
3. **Not found anywhere in Zoho's public pages:** a "Recently visited" list inside Reports, a "Shared Reports" page as
   such, an "Automation" report category, a collapse control on the main app's sidebar, the contents of the top-bar
   `+` menu as a whole, and any font or font-size setting for the interface.

**The six findings that change what we build.**
1. **Zoho's reports home is mostly a finding aid, not new reports.** The 2024 redesign (list view, sidebar, search,
   cards) sits on top of a fixed catalogue of roughly 40 to 70 standard reports. Most of what the owner wants from
   it (categories, search, a star, my views) is navigation over reports we already have. That is cheap for us.
2. **A Zoho "custom view" and our saved view are the same idea, but ours misses the part the owner asked for.** A Zoho
   view stores filter conditions, columns and who can see it. Ours (`Alt+V`) stores F12 options, added columns, basis
   and optionally the period, but explicitly **not** a filter, and it lives on one device. "Save filter in reports" is
   therefore a real gap, not a rename.
3. **The `+` beside a module is documented; the `+` on hover per row is mostly the owner's picture.** Zoho's welcome
   guide says each module has a quick-create control beside it and the top bar has a universal one. How a module's
   header relates to its list is not documented beyond menu paths such as "Sales > Invoices".
4. **Zoho's quick create is gated by add-ons and custom modules, and (by inference) by role.** Nothing documents the
   role part. For us the gate already exists in two forms: company feature flags and operator rights.
5. **Zoho does not let anyone reorder, hide or rename sidebar modules** (its own FAQ says so). Our rail already hides
   by feature and by operator right, which is more than Zoho does. Do not copy the rigidity.
6. **Zoho's admin appearance control is small:** a dark or light pane and an accent colour (red, blue, green, yellow or
   a custom hex), set under Settings > Branding, carried to the portals. No font or size option was found. Wave B can
   exceed it, but it is a smaller target than the owner's wording suggests.

---

## 1. The Reports section

### 1.1 Shape of the module
Reports hold the business, accounting and tax reports (sales, purchases, inventory, activity). Zoho's marketing page
says "70+" built-in reports; its help says "40+"; the owner's screenshot says 102 listed [owner]. The count depends on
edition and add-ons, so no single number is safe to quote.
- **The Reports page was redesigned** (release note dated 8 May 2024): a list view, a sidebar, search and a card-style
  layout with simpler filters. A separate note (31 Jan 2024) added a search bar at the top-left of Reports that also
  finds your saved custom reports.
- **Favourites:** a report can be starred so it is easy to find again (notes of 26 Dec 2018 and 4 Jan 2019).
- **Draft transactions never appear in any report** (stated on the overview page).
- Activity history for reports (10 Dec 2025) logs print, export, share and schedule actions, plus create and edit
  history of custom reports. Chart-view reports got AI-written trend summaries (5 Mar 2026).

### 1.2 Categories and the reports in each
The help pages list these, edition by edition. The owner's screenshot names 15 categories [owner]: Business Overview,
Sales, Inventory, Inventory Valuation, Receivables, Payments Received, Payables, Purchases and Expenses, Taxes, Banking,
Projects and Timesheet, Accountant, Currency, Activity, Automation. Names below are as the help pages give them.

| Category | Reports named in Zoho's help | Notes |
|---|---|---|
| Business Overview | Profit and Loss, Cash Flow Statement, Balance Sheet, Business Performance Ratio, Movement of Equity; Cash Flow Forecasting on some plans; a Horizontal P&L with stock details (Oct 2019 note) | Statements. Compare-with and report basis live here. |
| Sales | Sales by Customer, Sales by Item, Sales by Salesperson, Sales Summary, Profit by Item, Order Fulfillment by Item | Drafts excluded. |
| Inventory | Inventory Summary, Inventory Aging Summary, ABC Classification, Inventory Adjustment Summary and Details, Shipment by Item, FIFO Cost Lot Tracking | Add-on dependent in places. |
| Inventory Valuation | Inventory Valuation Summary (and FIFO lot tracking on some pages) | Shown as its own category in the screenshot [owner]. |
| Receivables | Customer Balance Summary, Aging Summary, Aging Details, Invoice Details, Retainer Invoice Details, Sales Order Details, Quote Details; Receivables Summary and Details (2020 note) | Aging Summary drills to a customer. |
| Payments Received | Payments Received, Time to Get Paid, Credit Note Details, Refund History | |
| Payables | Vendor Balance Summary, Purchase Order by Item; Payable Summary and Details, Bill Details, Vendor Credit Details elsewhere | The India payables page lists only two. |
| Purchases and Expenses | Purchases by Vendor, Purchases by Item, Purchase Order Details, Purchase Orders by Vendor, Payments Made, Expenses by Category, Customer, Project and Employee | Taken from the "reports that can be customised" list, not from one category page. |
| Taxes | Tax Summary, TDS Receivables Summary, TDS Payables Summary; India: GSTR-3B Summary (in Reports), "Summary of Outward Supplies" as the GSTR-1 data export | Filing itself (GSTR-1, 3B, 2B reconciliation) is a separate GST Filing module, **not** in Reports. HSN summary location **[unverified]**. |
| Banking | Reconciliation Status (summary, matched, unmatched transactions, unmatched statement lines; PDF and CSV) | |
| Projects and Timesheet | Project Summary, Project Revenue Summary; Timesheet Details **[unverified]** | |
| Accountant | General Ledger, Detailed General Ledger, Trial Balance, Journal Report, Account Transactions, Account Type Summary, Day Book; Budget vs Actual | Year, quarter and month to date appear in the date-range list here and in Business Overview. |
| Currency | Realised Gain or Loss, Unrealised Gain or Loss | |
| Activity | Activity Logs | |
| Automation | **[unverified]**: no help page names such a category | |

### 1.3 The Reports left panel (Home / Favorites / Shared / My Reports / Scheduled / Advanced Analytics)
What Zoho's own pages do and do not support:
- **Scheduled Reports** is a real page (note of 18 Feb 2020): it shows frequency, who scheduled it, recipients, status, next
  due date and file format. It needs administrator or full report access.
- **My Reports** is the place your own custom reports are listed. Administrators also get an "Other User Reports" area.
- **Shared reports:** reports can be shared to a named user with view or export access through a Share icon. A page
  called "Shared Reports" is **[unverified]**.
- **Advanced Analytics** is a link out to Zoho Analytics ("Try Advanced Reporting"), a separate product with its own
  workspace, one-way data sync, 75-plus prebuilt reports, pivots, dashboards and embedding. Only the administrator who
  sets it up can open it by default.
- **Home, Favorites, "last visited" column:** the star is documented; a recently-visited list inside Reports is
  **[unverified]**. Zoho's main navigation does have a "Recent History" of recently opened transactions.

### 1.4 Customising a standard report
Opening a report gives a Customize panel, in two tabs plus a Run step:
- **General:** date range (a preset week, month, quarter, year, year/quarter/month to date, or a custom range),
  **report basis** (accrual or cash, statements only), **filter accounts** (all, with transactions, or non-zero),
  **compare with** (previous periods, months, quarters or custom ranges; also by location), **group by** (on reports that
  support it, such as Sales Summary), and **advanced filters** (projects, reporting tags, branches and more).
- **Show/Hide Columns:** move columns from an Available list to a Selected list and drag them into order, then
  **Run Report**.
- **Compare with exists only on the statements** (Profit and Loss, Cash Flow, Balance Sheet, Movement of Equity).
- On the statements, a "create from existing report" flow adds **customise rows**: a new account group, a text row, or a
  **formula row** (built from accounts, groups and operators, with a syntax check).
- Changes can be kept as a **Save as Custom Report**. Without saving they are for that run only [inferred from the flow].
- **Table density, table design (bordered, alternating rows or columns), paper size, orientation, font and margins**
  are a separate administrator-level layout setting applied to PDF print and export, plus a choice of header details
  (organisation name, report basis, page number, generated by, generated date and time). One report can override them
  for a single print.

### 1.5 Custom reports and the Create New Report flow
- **Where:** Reports > Custom Reports (or any category) > Create New Report; a dropdown beside it offers "Create From
  Existing Report".
- **Five steps for a new report:** (1) **data source**: a parent module, optionally one or more child modules, with a
  switch between "related records only" (default) and "all records"; (2) **General**: date range, group by, advanced
  filters; (3) **Show/Hide Columns**: add, reorder; (4) **Report Layout**: the same print-layout options as above;
  (5) **Report Preferences**: report name, the name used on export, and permissions.
- **Permissions (who sees it, exports it, schedules it):** only me, only selected users and roles, or everyone in the
  organisation. Administrators can edit, clone and delete every user's custom report (note of 5 May 2026), and
  custom reports can be cloned from the list (note of 18 Sep 2025). A date-range filter was added to custom reports in
  2026. A scheduled custom report with a fixed range reuses that same range every time.
- **Limits by plan:** 10 (Standard), 25 (Professional), 50 (Premium), unlimited (Elite, Ultimate).
- **Conflict between pages:** one regional edition says custom reports exist only for Profit and Loss, Balance Sheet,
  Cash Flow and Invoice Details; another knowledge-base page lists about fifty customisable reports (statements,
  ledgers, sales, purchases, expenses, projects, tax). Plan and edition decide, and I could not tell which is current.
- **Charts:** a Chart View exists only on a named subset (Profit and Loss, the sales-by reports, receivable and payable
  summaries, details reports for invoices, orders, quotes, bills, purchase orders and expenses). Options: chart type,
  configure axes, a forecast view and an anomaly view. The custom-report page does not mention charts.

### 1.6 Export, print, schedule, share, access
- **Export** to PDF or spreadsheet, optionally password-protected; **print**; exports of large reports run in the
  background and are fetched from a list (Detailed General Ledger).
- **Schedule** a report daily, weekly, monthly, quarterly or yearly; it is emailed to chosen addresses. Not every report
  can be scheduled. A schedule is paused by marking it inactive.
- **Weekly summary email** to administrators every Sunday (income and expenses, cash flow, alerts), switched on under
  Settings > General.
- **Role-based report access:** a role gets full access to all reports, or access by report group; per report an
  administrator picks view, export, schedule, share. New reports are not automatically granted to limited roles. A
  default "staff" user cannot see Reports until it is granted. Related modules must also be granted.
- **Report notes** (Nov 2020) attach a temporary note to a report; **Zia insights** (2026) summarise chart views.

### 1.7 Sources (area 1)
- Manage Reports: https://www.zoho.com/in/books/help/reports/manage-reports.html
- Custom Reports (India, US editions): https://www.zoho.com/in/books/help/reports/custom-reports.html and https://www.zoho.com/us/books/help/reports/custom-reports.html
- Reports overview: https://www.zoho.com/us/books/help/reports/ and https://www.zoho.com/en-sg/books/help/reports/
- Category pages: https://www.zoho.com/us/books/help/reports/sales.html, https://www.zoho.com/us/books/help/reports/business-overview.html, https://www.zoho.com/in/books/help/reports/receivables.html, https://www.zoho.com/in/books/help/reports/inventory.html, https://www.zoho.com/in/books/help/reports/payables.html, https://www.zoho.com/en-sg/books/help/reports/tax-reports.html
- Customisable report list: https://www.zoho.com/de-de/books/kb/reports/list-of-reports-that-can-be-customized.html
- Reconciliation Status and Currency: https://www.zoho.com/us/books/kb/reports/download-transactions-reconciliation-status-report.html, https://www.zoho.com/en-sg/books/help/reports/currency-reports.html
- Accountant reports (kb): https://www.zoho.com/us/books/kb/reports/detailed-general-ledger.html, https://www.zoho.com/in/books/kb/reports/can-i-view-the-trial-balance-report-with-opening-and-closing-balances.html
- GST reports (Reports vs filing): https://www.zoho.com/in/books/kb/gst/gst-3b-contents.html, https://www.zoho.com/in/books/kb/gst/download-gstr1-return.html
- Advanced Analytics: https://www.zoho.com/us/books/help/integrations/advanced-analytics.html
- What's New (reports redesign, search, favourites, clone, activity history): https://www.zoho.com/us/books/whats-new.html
- Release notes (favourites, scheduling, new reports): https://www.zoho.com/us/books/release-notes.html
- Reporting feature page: https://www.zoho.com/us/books/accounting-software/accounting-financial-reports/

---

## 2. The universal `+` (Quick Create)

**What is documented.**
- A `+` icon in the top band (top right) opens a list of modules; picking one starts a new record from **any page**.
  It first appeared in 2015, grew in 2017 to cover retail invoices, customer payments, recurring bills and expenses,
  vendor payments, journals and inventory adjustments, and has had additions since.
- It creates contacts (customers, vendors), sales and purchase transactions "and more". Release notes show two
  conditional entries: **purchase receives only for organisations with the Inventory add-on** (2023) and **custom
  modules** (June 2024).
- The welcome guide says each module also has its own quick-create control beside it, so the same action exists in the
  left navigation and in the top band.
- Keyboard route: a `C` then a letter creates a record from anywhere (section 5).

**Gating.** Only add-ons and custom modules are documented as changing the menu. Role gating is **[unverified]**: the
help says roles control module access and per-action permissions, and does not say whether a user without create rights
sees the entry. The sensible reading is that the menu follows create permission **[inferred]**; test with a restricted
user before relying on it.

**The mega-menu itself.** Five columns (General, Inventory, Sales, Purchases, Banking), each a list of "New X" entries,
is the owner's description **[owner]**. Zoho's public pages never print the menu's contents, so the entry list per
column is **[unverified]** and varies by plan, add-ons and custom modules.

**Sources:** https://www.zoho.com/us/books/help/personalize/more-actions.html, https://www.zoho.com/us/books/whats-new.html, https://www.zoho.com/books/whats-changed.html, https://www.zoho.com/qa/books/welcome-guide.html, https://www.zoho.com/us/books/kb/users-and-roles/custom-role.html

---

## 3. List screens (the "All Invoices" page)

### 3.1 The title dropdown and custom views
- The title of a list page ("All Invoices", "All Bills") is a **dropdown of views**, in three bands: **Favorites**
  (starred views, on top), **Default filters** (Zoho's own, per module, read-only: for example by status), and
  **your views plus views others shared with you**. A search box appears inside the dropdown once there are more than
  ten default filters or any custom view exists (note of 7 Oct 2024). Recently used views are remembered across
  sessions (2018 note).
- **Create a view:** open the dropdown, choose **+ New View** at the bottom.
- **What a view stores:** a name; a favourite flag; **filter criteria** (one or more rows of field, comparator, value);
  **columns** (an Available list and a Selected list, drag to order); **who may use it** (only me, selected users and
  roles, or everyone in the organisation); and whether the view is also offered in other Zoho apps. The help does
  **not** list sort order as part of a view, and does not list a limit on views or criteria.
- **Who sees it:** the creator, plus whoever the share setting names. Only the creator can edit or delete a view
  (edit icon beside the name, or the delete button on its edit page). Default filters cannot be changed.
- **Export current view** (notes of Mar 2019 and Jan 2024): the filtered, column-arranged list is what gets exported.

### 3.2 `+ New`, the `...` menu, the column chooser, per-row actions
- **`+ New`** (top right) opens the new-record form for that module. Whether it is a **split button** with a dropdown
  of related actions is **[unverified]** (the help just calls it a button; the owner sees a split) **[owner]**.
- **The `...` / More menu**, as far as module help pages show it: **Import** (invoices accept CSV, TSV or XLS up to 10 MB,
  with encoding and delimiter choice, field mapping and a preview before import), **Export Current View** (choose a
  format, optionally set a password), **Sort by** (pick a column; pick it again to reverse), **Refresh List**, and
  **Reset Column Width** (seen on the sales-receipts page). A **Preferences** entry is **[unverified]**; pages only
  link to a separate "preferences" article per module.
- **Table (columns) icon:** choose which fields show and drag them into order; mandatory fields cannot be hidden; up to
  two columns can be **pinned** on the left; text can wrap or clip. Clicking a column heading also sorts.
- **List-level extras:** a **calendar toggle** on Bills (month view of due and overdue); an **advanced search** behind
  the magnifier on most lists; a period dropdown on some lists.
- **Per-row and bulk actions:** the help documents actions on an invoice's own page (edit, email, share a link, print,
  download PDF, make recurring, create a credit note, clone, view journal, delete). Row hover actions and the bulk-select
  bar are **[unverified]**; the keyboard page lists module-level bulk shortcuts for email, PDF, import, export and select
  all, for sales and purchase modules only.

**Sources:** https://www.zoho.com/books/help/settings/customization/custom-views.html, https://www.zoho.com/us/books/help/invoice/, https://www.zoho.com/us/books/help/bills/manage-bills.html, https://www.zoho.com/au/books/help/sales-receipts/manage.html, https://www.zoho.com/en-fr/books/help/payments-received/manage-payments.html, https://www.zoho.com/us/books/whats-new.html

---

## 4. The left navigation and the dashboard

- **Layout:** a navigation panel on the left lists every module, with **Sales** and **Purchases** groups holding
  quotes, invoices, expenses, bills and so on. Help articles give paths in the form "Sales > Customers" or
  "Sales > Invoices": a group header opens in place, a row inside it opens that module's list. What a click on the
  group header itself does beyond expanding is **[unverified]**.
- **Quick add:** the welcome guide says each module has a quick-create control beside it. Showing it only on hover is
  the owner's picture **[owner]**.
- **Collapse:** no public page describes collapsing the main sidebar. The only documented hamburger collapse is in the
  **customer and vendor portals** (Aug 2022). The icon-only collapsed rail in the Reports screenshot is **[owner]**.
- **Customising it:** Zoho's FAQ says modules cannot be renamed, reordered, hidden or shown by the organisation; only
  transaction names on PDF templates can change.
- **Other regions on the page** (from Zoho's navigation overview): a dashboard (receivables and payables totals, top
  projects, getting started, announcements), **Recent History** of recently opened transactions, a global **search bar**
  with an advanced-search option and an AI search, an organisation switcher, **Quick Create**, notifications, Settings,
  profile and help, and a switcher to other Zoho apps.
- **Dashboards:** a locked system dashboard for everyone; **custom dashboards only on the Elite and Ultimate plans**,
  with panels (predefined, or charts built from a report) that can be dragged and resized, shared as only me, everyone
  or selected users with view or manage rights, and up to three marked favourite.

**Sources:** https://www.zoho.com/books/whats-changed.html, https://www.zoho.com/qa/books/welcome-guide.html, https://www.zoho.com/books/kb/general/rename-modules.html, https://www.zoho.com/us/books/help/home/custom-dashboards.html, https://www.zoho.com/us/books/whats-new.html

---

## 5. Keyboard shortcuts, appearance and admin customisation (context for Wave B)

**Shortcuts.**
- `Shift+?` opens the full list (also a link under the profile menu). The system dates from 2015 and was extended in 2017.
- **Groups:** general (`/` search, `Option+/` advanced search, `S` Settings, `O` organisation list); **navigation**
  (`Shift` plus a letter or digit); **module level** and **transaction level** (email, PDF, print, import, export,
  select all, edit, mail; only for sales and purchase modules); **creation** (`C` then a letter); **save** (`Option+S`
  save and send, `Option+P` save and print).
- **Navigation keys:** Shift+I Invoices, J Journals, E Quotes, N Credit Notes, B Bills, X Expenses, P Purchase Orders,
  S Sales Orders, Z Chart of Accounts, R Payments Received, M Payments Made; Shift+1 Dashboard, 2 Items, 3 Banking,
  4 Customers, 5 Vendors, 6 Reports, 7 Documents.
- **Creation keys:** C then I Invoice, J Journal, E Quote, N Credit Note, B Bill, X Expense, A account, T time log,
  5 Vendor, 4 Customer, R Retail Invoice, 1 Customer Payment, M Vendor Payment.
- **Customising:** a user can assign their own key to any action with an edit icon; the June 2024 note says shortcuts can
  be customised, disabled and reset to defaults. A "jump to main content" key (Aug 2024) and accessibility preferences
  (Apr 2026) exist. How conflicts with browser keys are handled, and Windows equivalents of the Option keys, are
  **[unverified]**.

**Appearance (what an administrator can change).**
- **Where:** Settings > Branding (under Organization). **Appearance:** a dark pane or a light pane. **Accent colour:**
  red, blue, green or yellow, or a custom colour by picker or hex code. It also applies to other connected Zoho Finance
  apps and to the customer and vendor portals. Who may change it is **[unverified]**.
- **Not found:** any setting for interface font, font size or density. Fonts appear only for PDF templates, where each
  template has a colour theme and fonts. There is no mention of a full-app dark mode.
- **Preferences** is a settings page grouped into General, Customer Portal, Vendor Portal, Customers and Vendors,
  Projects, Accountant, plus transaction number series and posting dates. It holds work-week start, PDF encryption,
  the weekly summary and similar; it has no appearance, language or date-format settings in what I could read.

**Sources:** https://www.zoho.com/books/help/getting-started/keyboard-shortcuts.html, https://www.zoho.com/us/books/whats-new.html, https://www.zoho.com/us/books/help/personalize/change-theme.html, https://www.zoho.com/us/books/kb/general/branding-themes.html, https://www.zoho.com/us/books/help/settings/preferences.html, https://www.zoho.com/us/books/help/personalize/

---

## 6. What this means for Comsky Books

### 6.1 The owner's asks, mapped to Zoho's concept and our code

| Owner's ask | Zoho concept | What we have today | Verdict |
|---|---|---|---|
| Save filter in reports | Custom view (filters + columns + sharing) | Saved views `Alt+V` (`core/savedViews.ts`): F12 options, added columns, basis, optional period; **not** filters; device-local per company | **Gap, Wave A.** Extend, do not replace. |
| `+` after each menu item: opens the report by default, `+` creates the voucher | Module list + quick-create beside it | Rail rows are split: entry rows (Sales Invoice) and list rows (Sales Register); master rows already have a `+`; a `+` key in the rail | **Gap, Wave A.** Merge entry and list rows into one list row with a `+`. |
| Click Sales and see the register | Module opens its list | Tally behaviour: `Sales Invoice` opens a blank entry; `Sales Register` is a separate row | **Gap, Wave A**, same change as above. |
| "Add New" button, top right of a list | `+ New` | None on registers or the Day Book; creation is by key (F8 on the Gateway), palette or menu | **Gap, Wave A.** |
| Fewer submenus | Flat module groups | 14 groups, 108 rows | Reduced by the merge above (section 6.3). |
| Reports section like Zoho's | Reports Center | No reports hub. Reports are reached by the Gateway's Display More Reports and submenus, the rail's Reports group (8 rows), `Ctrl+K` and Go To | **Gap, Wave A.** |
| Search reports | Reports search bar | `Ctrl+K` with the `r ` scope, the rail's filter box | Partly covered; a hub needs its own box (reuse `paletteRank`). |
| Create a new report | Create New Report | Nothing free-form. Saved views over an existing report are the nearest thing | **Do not build a builder in this wave** (6.5). |
| Universal `+` in the top bar | Quick Create | Palette "Create" category, Gateway Create (`C`), voucher F-keys. Top bar has brand, menu keys, Go To pill, density, scheme | **Gap, Wave A.** |
| Dashboard, right-hand shortcut keys, analyst view | (section 6.6) | Dashboard with a customiser and a company layer; F-key rail | Suggestions in 6.6. |
| Configurable shortcut buttons, admin settings window | Custom shortcuts; Branding | `?` sheet, favourite slots `Ctrl+Alt+1..9`, `ThemeControls` (light/dark, density) | **Wave B.** Contracts to reserve in 6.7. |

### 6.2 Reports Center: feature by feature

| Zoho feature | What we already have | Gap | Recommendation |
|---|---|---|---|
| Home with categories (Business Overview, Sales, Inventory, Receivables, Payables, Taxes, Banking, Accountant ...) | Rail "Reports" group of 8 rows; Gateway's Display More Reports; GST, TDS, Payroll under their own groups | One screen that shows **every** report by category does not exist | Build a **Reports hub**: a category list on the left, a table of reports on the right (name, category, favourite star, last opened). Derive its catalogue from `navTree.ts` and the Gateway `MENUS`, not a third hand-typed list; add it to the same drift guard style as `sidenav.assert.ts`. |
| A report catalogue grouped for a Zoho user | Names are Tally-style | Category names a Zoho user expects | Proposed categories (final names to be set against the real registry): **Business overview** (P&L, Balance Sheet, Cash Flow, Funds Flow, Ratio Analysis, Budget vs Actual, Top-N MIS, Profitability); **Sales and receivables** (Sales Register, Receivables, Ageing, Reminder Letters, credit notes); **Purchases and payables** (Purchase Register, Payables, debit notes, orders outstanding); **Inventory** (Stock Summary, Godown stock, Movement, Stock Query, Stock Ageing, Reorder Status); **Accounting** (Day Book, Trial Balance, Ledger, Group Summary, Cash/Bank Book, registers, cost centres, Interest); **Banking** (Reconciliation, Post-dated cheques, Payment Advice, Deposit Slip); **GST**; **TDS and TCS**; **Payroll and HR**; **Audit and checks** (Edit Log, Cancelled Vouchers, Exception Reports, Data Check, Sync Mismatch). Each hub row keeps its Tally name and shows the Tally key where one exists. |
| Favourites (star) | Rail Pinned (max 8) and Quick access; `Ctrl+Alt+1..9` slots | A star on each hub row | Reuse the existing pin store (`state/recents.ts`, `PINS_MAX`). Do **not** invent a second favourites list. A pin of a report with views should open it as the view sets it. |
| Recently visited | Quick access "Recent", per operator and company | Not shown inside a reports hub | Show a Recent band in the hub from the same memory. |
| Reports search | Palette `r ` scope; rail filter | No in-hub box | Add a search box that uses `paletteRank` with the aliases already carried (Tally and Zoho names). |
| My Reports / custom reports | Saved views (`Alt+V`), shown in the rail under "Saved views" (first 6) and in Go To as "View: name" | Not listed together with reports; no sharing | List **My views** in the hub next to the standard reports, each tagged with its source report. |
| Shared Reports, permissions (only me / selected / everyone) | None: views are device-local | Cannot hand a view to a colleague or the whole company | Use the **layering the Dashboard already has** (personal on this device, then a company layer that administrators write and that syncs, `Company.dashboard` precedent). Offer "Only me" and "Everyone in this company"; per-role sharing is not worth building yet. Needs a synced field, a sanitiser and an operator-rights rule. Phase it behind the local half. |
| Scheduled reports (email on a schedule) | None; only backup scheduling exists | Needs a sender and a job runner, neither of which exists in `backend/` | Do not build in this wave. It depends on the shared platform pieces in `docs/research/README.md` section 2 (S6 background jobs, S7 notifications and email). Offer "remind me to run this report" locally later if wanted. |
| Advanced Analytics | Dashboard customiser, saved views, CSV and Excel | None needed | Skip (section 6.8). |
| Customise a standard report: date range | `F2` period, date mnemonics | Covered | None. |
| Compare with | Added columns on Trial Balance, Group Summary, Balance Sheet, P&L, Stock Summary (as-on, period, auto months or quarters), `core/reportColumns.ts` | Not offered on the registers or non-statement reports; by location is moot (no locations; cost centres exist) | Already stronger than Zoho on the statements. Leave. Note: "Compare with Tally Data" is a migration checker, not this feature. |
| Report basis | `Ctrl+B` basis of values | Covered on the statements | None. |
| Filters and group by | `/` type-to-filter text; F12 core options (zero rows, scale, percent, sort) plus per-report extras | No multi-condition filter and no group-by on the registers and Day Book | Core of the "save filter" ask. Give registers a small, named **filter set** (period, party or group, voucher type or status such as unpaid or overdue, amount range) that can be saved in a view. Group-by (party, month, item) is a second step. |
| Show/hide columns | Added columns only on statements; most reports have fixed columns | Choosing columns on a register | Medium priority; a column chooser on the registers and Day Book, saved in the same view. |
| Custom report builder (data source, columns, filters, group, chart) | None | A free builder | Not now. The 80 percent case is "a register with my columns and my filter", which saved views cover once filters and columns are in them. |
| Chart view | Dashboard trend widget | No chart toggle on a report | Low to medium; one small chart toggle for the registers and P&L later. Not Wave A. |
| Export, print, password-protect | Print, CSV, Excel for any report (`reportTools.ts` reads the visible `.scr-body`); Export Data writes Tally XML | No password-protected exports; no report-page layout options beyond Print Configuration | Fine. Skip passwords; maybe expose paper size and orientation for report print later. |
| Activity history for reports | Edit Log records data changes | Print and export are not logged | Skip. |
| Role-based report access | `core/access.ts` `SCREEN_AREA`, `screenDenial`, feature flags | Covered | The hub must hide rows by the same checks the rail uses (`screenDenial`, company features), and a new Screen needs a `SCREEN_AREA` entry or it is admin-only. |

### 6.3 Universal `+` and the left menu `+`

**Universal `+` (top bar).** Build as a menu opened by mouse, with the entries grouped like the owner's columns and filled
from what Comsky can create. A reasonable first mapping, to be checked against `navTree.ts` `create` fields:
- **Sales:** Sales Invoice (`F8`), Payment Received (`F6`), Credit Note (`Alt+F6`), Sales Order, Delivery Note.
- **Purchases:** Purchase Invoice (`F9`), Payment Made (`F5`), Debit Note (`Alt+F5`), Purchase Order, Receipt Note.
- **Banking:** Transfer or Contra (`F4`), cheque entries; "Import bank statement" is an action, not a create, so keep it
  out or label it as such.
- **Inventory:** Stock Item, Stock Journal (`Alt+F7`), Physical Stock (`Ctrl+F7`), Manufacturing Journal.
- **General:** Journal (`F7`), Ledger (a party or an account), Group, Lead and Employee where those modules are on.
  Zoho's "customer" and "vendor" become a Ledger with the party group preset, which the Tally switcher also expects.
- **Gating** is the same pair as the rail: the company feature flag (hide Inventory when `inventory` is off, orders when
  `orders` is off) and the operator's right to write that voucher type (`screenDenial` and the write guard). An entry that
  would be refused is not shown. This matches `MenuScreen.offered()` and the rail, so the three can share one helper.
- **Printed keys** say where they fire (the rail already does this: the voucher F-keys fire on the Gateway and its menus,
  not on a report). Never print a key that will do nothing from where the button sits.
- **How to open it from the keyboard:** a new global chord needs the same audit the collapse chord got in
  `left-nav-spec.md` section 5. `Ctrl+K` already has a Create category and may be enough.

**`+` per row in the left menu.** Zoho's pattern is "the module opens its list; the `+` makes a new one". Applied to us:
- Merge entry and list rows so one row means one thing. Example: **Sales** group becomes **Invoices** (opens the Sales
  Register, `+` opens a Sales Invoice), **Payments Received** (Receipt register, `+` Receipt), **Credit Notes**, **Orders**
  (when `orders` is on), **Delivery Notes** (when `inventory` is on), **Receivables**, **Ageing**, **Reminders**. That is
  roughly nine rows down to six or seven, and the same trick takes Purchases from seven to five, which gives the owner's
  "fewer submenus" without removing any screen. Each screen stays reachable by Go To and the Gateway keys.
- **Rows with no list** (Journal, Contra) keep their entry screen as the row and need no `+`.
- **Keep what already works:** the rail's own `+` key and master rows' `+` buttons; the Tally key as a dim keycap; the
  rule that the rail handles only its own keys (CLAUDE.md "a rail that handles its own keys must not register a global
  handler").
- **Drift guard:** `scripts/assert/sidenav.assert.ts` holds the literal 108-row table. It must be changed deliberately with
  `navTree.ts`, as the nav spec requires.
- **Collapsed icon-only mode (owner's Reports screenshot).** We have full hide (`Ctrl+\`, breadcrumb toggle, hamburger).
  An icon-only third state is cheap because each of the 14 groups already has an icon, but it adds a width state to a
  grid that already broke once (the `navhid` bug). Optional, low priority.

### 6.4 List screens

| Zoho | Comsky today | Gap and recommendation |
|---|---|---|
| Title dropdown with **default filters** + **custom views** + **favourites** + search | A register is one list for the open period; saved views are reached by `Alt+V` | Add a **view chip or dropdown in the register's header** listing: predefined filters (all, this month, unpaid, overdue, cancelled where it applies), the operator's saved views for that register (starred first), and "Save current filter as view". The chip logic exists (`viewMatches` names the active view). |
| A view stores filters + columns + share + favourite | Our view stores F12 options, added columns, basis, optional period; pin = favourite | **Add filter criteria to the view schema**, additively, with the existing sanitising rules (a corrupt or older blob must never poison a report). Keep sort (Zoho does not store it; we do, which is better). |
| Who sees a view | Nobody else | Personal plus company layer (6.2). |
| `+ New` (top right) | Not present | Add a button in the screen header of every register and master list. Label with the voucher it creates; the dropdown holds related actions (Zoho's split button, if the owner sees one **[owner]**). Show the key as a keycap only if it fires from that screen. |
| `...` menu: Import | Import Data, Bulk Entry (Data and Utilities) | Surface both from a `...` menu on the register. |
| `...` Export current view | CSV and Excel from `reportTools` for any report | Label it "Export what is shown"; it already exports the visible table. |
| `...` Sort by | F12 sort | Mirror F12's sort choice in the menu; same setting. |
| `...` Preferences, Reset Column Width, Refresh List | F12 panel; Print Configuration; local-first data is always current | Preferences = open F12. Skip Refresh and Reset Width. |
| Column chooser, pinned columns | Fixed columns on registers | See 6.2. |
| Per-row actions | Enter to open, `Alt+2` duplicate, `Alt+D` delete, print, mark optional | Keys are the whole story. For mouse-first users a row-end menu mirrors those; low to medium priority. |
| Calendar view of bills | None | Skip. |

### 6.5 Navigation breadcrumb bug (not in scope here, stated for the record)
Zoho's pages do not describe a breadcrumb at all, so there is nothing to borrow. The fix is in the brief (decide on real
unsaved work, de-duplicate, shorten the status-bar path).

### 6.6 What else would improve the experience, for both kinds of user (analyst view)
Suggestions beyond what the owner listed, each tied to evidence in Zoho's behaviour or our own gaps:
1. **One find-anything box that is visible.** Zoho has a search bar in the top band on every page. We have `Ctrl+K` and a
   Go To pill; a Tally user never needs more, an owner from Zoho will not find a hidden chord. Keep the pill, make it read
   as a search box, show the keycap.
2. **A "What's next" nudge after saving a voucher** (Zoho added one in 2019: the usual next step after an invoice).
   Comsky's After Accept (`Alt+N`) is the keyboard form; a visible follow-up row ("record payment", "print", "new
   invoice") serves owners without taking a key away.
3. **A first-session path for owners:** the sample company and Getting Started exist; point the new hub and the universal
   `+` at them so a non-Tally user's first ten minutes are "click +, create invoice, open Sales, see it, open Receivables".
4. **Dashboard as the owner's home, reports hub as the analyst's home.** The Dashboard already has a customiser and a
   company layer, more generous than Zoho's (custom dashboards are Elite and Ultimate only there). Add "open report"
   drill-throughs from each tile to the saved view that explains it.
5. **Two-vocabulary search aliases** (already in place for the rail and palette): make the hub use them, so "invoices",
   "bills", "customers" land on Sales Register, Purchase Register, Ledgers.
6. **Recently opened on the hub and the rail** from the same memory, per operator.
7. **An empty state that teaches the key.** Anywhere a button replaces a key, show the key on hover; anywhere a key
   replaces a button, show the button. This is the product's main promise to the Tally switcher.
8. **Do not add features that change a key's meaning.** The superset rule applies to the new button, the hub and the `+`.

### 6.7 Wave B contracts to reserve (do not build now)
- **Shortcut customisation.** Zoho lets a user rebind any key, disable keys and reset. Ours must stay a **superset**:
  `LOCKED_KEYS` (`favouritesCore.ts`) and the `?` sheet already list what cannot move. Wave B's "configurable shortcut
  buttons" most naturally means **which actions sit on the right-hand F-key rail**, not changing key meanings. Reserve:
  a per-company (administrator) layer and a per-operator layer, same layering as the Dashboard; a settings screen id and
  a `SCREEN_AREA` entry; the Alt+key grid pattern for any new field.
- **Admin settings window.** Zoho's admin appearance is dark or light pane plus an accent colour, carried to portals.
  Ours has scheme and density (`ThemeControls`, `state/theme.ts`) per device. Wave B can add accent colour and font size
  as **design tokens** (override `--accent` and the type-scale root, never hex in `:root` for a token dark does not
  override, per the light-theme spec), with a contrast check on every chosen colour (the nav spec already computed the
  AA pairs), stored as a company default that an operator may override. Zoho shows **no** font or font-size setting, so
  this goes beyond Zoho.
- **Both** need a Screen entry, `SCREEN_AREA`, a feature of "who is an administrator" (security rights), and a rule that
  nothing here may hide a keycap.

### 6.8 Zoho features not worth copying for a keyboard-first Tally switcher

| Feature | Why not |
|---|---|
| Single-letter chord system (`C` then `I`, `Shift+letter` to navigate) | Collides with the Gateway's own single-letter keys and bare letters die inside fields (`isTypingTarget`). We already own `Alt+`, `Ctrl+`, the `G` chords and the F-keys; the `+` needs a button, not a new letter namespace. |
| Plan-limited custom reports (10, 25, 50, unlimited) | An artificial limit. Our plan caps are display-only today; do not add the first enforced one on a convenience feature. |
| Zia AI search, chart insights, anomaly and forecast views | A separate product investment; no evidence our users ask for it before the filter, hub and `+` exist. |
| Advanced Analytics (a separate BI workspace, one-way sync, embedding) | A second product with its own admin. The Dashboard customiser plus saved views and CSV cover the owner's need. |
| Free-form custom report builder with parent and child modules | Large, easy to get wrong over a voucher log with optional vouchers (every report must exclude them), and our users think in registers and ledgers, not modules. Filters and columns in a view reach most of it. |
| Email-scheduled reports and the weekly summary email | Needs an email sender and a job runner we do not have; do it with the platform work, not as part of a UI wave. |
| Password-protected report exports, report PDF layout screens | Low value for local-first books; Print Configuration already covers paper and layout for vouchers. |
| A fixed, non-customisable sidebar | Zoho cannot hide or reorder modules. Ours hides by company feature and by operator right; keep that. |
| Dark-pane versus light-pane choice for the sidebar alone | The nav spec rejected a navy slab on tokens, contrast and identity grounds; the app's scheme switch already does this. |
| Custom modules in quick create, portal-oriented settings, retainer invoices, estimates, projects and timesheets, mileage, multi-currency gain or loss reports | Out of scope (no multi-currency or projects module; CRM and HR have their own screens). |
| Zoho's names where they replace ours | Keep Tally names and keys as the primary label with Zoho names as aliases and tooltips (nav spec section 4.6). The brand rule also bars Tally wording in UI except on migration screens. |
| The "Home / Getting started / announcements" tabs | Marketing surface; our Getting Started and Welcome screens already do the onboarding job. |
| Calendar view for bills; Zoho's "Shipment by Item" style add-on reports | Niche; no demand shown. |

### 6.9 Gaps, ranked

**Wave A, in this order of value:**
1. Merged `+` rows in the left menu (open the list; `+` creates) and a register `+ New` button. Fixes the owner's main complaint.
2. Saved **filters** inside saved views, with a view chip or dropdown on registers and the Day Book. The owner's first sentence.
3. The **Reports hub**: categories, search, star, recent, My views.
4. The universal `+` in the top bar, gated by features and rights.
5. A `...` menu on registers collecting Import, Bulk Entry, Export what is shown, Sort.

**Later in or after Wave A:** company-shared views; column chooser on registers; group-by on registers; row-end menu; small chart toggle; icon-only rail.

**Wave B:** shortcut-button configuration; admin settings (accent, font size, density default, scheme default).

**Out for now:** scheduled email reports; report builder; Analytics; AI features.

### 6.10 Things to confirm in a live Zoho Books organisation before the spec is locked
1. The exact entries and columns of the top-bar `+` menu, and whether it hides entries a restricted role cannot create.
2. Whether `+ New` is a split button, and what its dropdown holds.
3. The `...` menu's full list on a list page (is there a Preferences item?).
4. The Reports Center left panel: does a Recently visited list exist, what does Shared Reports contain, what is under Automation?
5. What a click on a sidebar group header does when the group is already open, and how the hover `+` appears.
6. Whether the icon-only collapsed mode is a user control and whether it persists.
7. Whether a saved custom view can be re-shared or its columns edited after saving (the help does not say).

---

## 7. Source index (all URLs read for this document)

Reports: https://www.zoho.com/in/books/help/reports/manage-reports.html · https://www.zoho.com/in/books/help/reports/custom-reports.html · https://www.zoho.com/us/books/help/reports/custom-reports.html · https://www.zoho.com/us/books/help/reports/ · https://www.zoho.com/en-sg/books/help/reports/ · https://www.zoho.com/us/books/help/reports/sales.html · https://www.zoho.com/us/books/help/reports/business-overview.html · https://www.zoho.com/in/books/help/reports/receivables.html · https://www.zoho.com/in/books/help/reports/inventory.html · https://www.zoho.com/in/books/help/reports/payables.html · https://www.zoho.com/us/books/help/reports/payables.html · https://www.zoho.com/en-sg/books/help/reports/tax-reports.html · https://www.zoho.com/en-sg/books/help/reports/currency-reports.html · https://www.zoho.com/de-de/books/kb/reports/list-of-reports-that-can-be-customized.html · https://www.zoho.com/us/books/kb/reports/detailed-general-ledger.html · https://www.zoho.com/us/books/kb/reports/download-transactions-reconciliation-status-report.html · https://www.zoho.com/in/books/kb/gst/gst-3b-contents.html · https://www.zoho.com/us/books/help/integrations/advanced-analytics.html · https://www.zoho.com/us/books/accounting-software/accounting-financial-reports/

Product pages: https://www.zoho.com/us/books/whats-new.html · https://www.zoho.com/us/books/release-notes.html · https://www.zoho.com/books/whats-changed.html · https://www.zoho.com/qa/books/welcome-guide.html · https://www.zoho.com/us/books/accounting-software-features/ · https://help.zoho.com/portal/en/community/topic/zoho-books-product-updates-june-2026-7-6-2026 · https://help.zoho.com/portal/en/community/topic/zoho-books-product-updates-october-2026

Quick create, lists, views: https://www.zoho.com/us/books/help/personalize/more-actions.html · https://www.zoho.com/books/help/settings/customization/custom-views.html · https://www.zoho.com/us/books/help/invoice/ · https://www.zoho.com/us/books/help/bills/manage-bills.html · https://www.zoho.com/au/books/help/sales-receipts/manage.html · https://www.zoho.com/en-fr/books/help/payments-received/manage-payments.html

Navigation, dashboards, appearance, access: https://www.zoho.com/us/books/help/home/custom-dashboards.html · https://www.zoho.com/books/kb/general/rename-modules.html · https://www.zoho.com/us/books/help/personalize/ · https://www.zoho.com/us/books/help/personalize/change-theme.html · https://www.zoho.com/us/books/kb/general/branding-themes.html · https://www.zoho.com/us/books/help/settings/preferences.html · https://www.zoho.com/us/books/kb/users-and-roles/custom-role.html

Shortcuts: https://www.zoho.com/books/help/getting-started/keyboard-shortcuts.html

Comsky sources read: `CLAUDE.md`, `docs/design/left-nav-spec.md`, `docs/research/README.md`, `src/ui/navTree.ts`, `src/core/savedViews.ts`, `src/ui/reportViews/*`, `src/core/reportConfig.ts`, `src/ui/reportColumns.tsx`, `src/ui/reportTools.ts`, `src/ui/MenuScreen.tsx`, `src/ui/modern/CommandPalette.tsx`, `src/ui/favouritesCore.ts`, `src/ui/ThemeControls.tsx`, `src/state/theme.ts`, `src/state/recents.ts`, `src/core/dashboardLayout.ts`, `src/core/access.ts`.
