# Dashboard v2: a Zoho-style, customisable home for Comsky Books (research and design)

> **Status:** research plus design. Nothing here is built. Written 2026-10-20 for the owner's ask of 2026-10-19: "make the
> dashboard of Comsky Books similar to Zoho Books so that it is intuitive, and make it customizable".
> **Reads before this (not repeated here):** `docs/research/zoho-books-reference.md` (section 4 has the two-line version of
> Zoho's dashboard), `docs/design/zoho-ux-spec.md`, `docs/design/ux-analysis-2026-10.md` (section 4.3 and N4 are the Dashboard
> findings this document builds on).
> **Wave A is in flight and uncommitted** (page header, filter bar, Reports Center, menu v2). Nothing below needs a new
> `Screen`; the files it touches that Wave A also touches are listed in section 9 so the order can avoid a collision.
> **Method:** Zoho's own public help, knowledge-base, release-note and blog pages, read through a fetch that summarises each
> page (so detail can be dropped; see 1.7). Everything is paraphrased; no Zoho text or screenshot is reproduced. Our side
> was read from the working tree, run in the headless harness, and timed with the repo's engines.
> **Evidence tags:** **[Z-key]** a Zoho page (key in section 10) . **[C]** read in our code (file named) . **[E]** seen in a
> harness screenshot (light and dark, 1440 and 1100 wide, pinned date 2026-10-12) . **[V]** verified by running something
> (tests, the perf harness, a scratch benchmark or the palette validator) . **[J]** my judgement . **[inferred]** and
> **[unverified]** as stated. Effort numbers are ideal builder-days for one engineer who knows the repo, tests and screenshots
> included **[J]**.

---

## 0. Read this first

**What the owner asked for is two things, and Zoho answers only one of them.**
1. *Intuitive, like Zoho:* a home page that answers four questions without a click: who owes me and how late, what do I
   owe, what is happening to my cash, and am I making money. Zoho's default home is exactly that: Total Receivables and Total
   Payables with a current-versus-overdue split, a Cash Flow chart, an Income and Expense chart, Top Expenses, the bank
   accounts, an account watchlist, and (as a separate tab) a Getting Started guide. [Z-home][Z-india]
2. *Customisable:* here **Comsky is already ahead of Zoho in machinery and behind in defaults and charts.** Zoho's default
   dashboard cannot be edited, reordered or deleted; custom dashboards exist only on its two top plans, and only report-built
   panels can be resized. [Z-custom] Ours is free on every plan, per operator, with a company standard that an administrator
   sets, fully keyboard-driven (`F12`). [C `core/dashboardLayout.ts`, `ui/modern/DashCustomise.tsx`] What it lacks is a board
   worth customising: no profit, no cash-flow or income chart, no overdue list, no period control, and a clock that is wrong.

**The nine findings that shape the design.**
1. **The Dashboard prints the wrong date on every balance.** Every tile says "as at 31-Mar-27" (the period end, five months in
   the future on the harness date), and cash, receivables, payables and overdue are balances at that date. [E, C
   `core/dashboardData.ts:239-262`] Zoho's receivables and payables come from its aging reports **as of the day you look**. [Z-kb-overdue]
   Fix: one rule, `asOn = min(period end, today)` (never earlier than the period start), shown on every widget.
2. **There is no period control on the Dashboard at all.** The screen binds only `F12` and `Esc`; the period is whatever a
   report last set with `F2`, and the hero line calls any period "FY". [C `Dashboard.tsx:58-64,77`] Zoho puts a period
   selector in the top right of every chart panel. [Z-home]
3. **The GST tile does not show what its own blurb says.** The blurb says "net of input credit, for the current month"; the
   code builds GSTR-3B over the **whole open period** (the financial year by default). [C `dashboardLayout.ts:59` against
   `dashboardData.ts:266-268`] A year-to-date figure on a "due" tile is a wrong number on the page owners check most.
   Building the *filing period* instead is also 6x cheaper: **69 ms for one month against 406 ms for the year at 250,000
   vouchers** [V].
4. **There is no profit anywhere on the Dashboard, and the profit maths lives inside a React component.** The Profit and Loss
   screen computes sales, stock change, purchases and expenses in a `useMemo` in `ui/reports.tsx`, and Ratio Analysis holds a
   second copy (with a leftover placeholder line) in `ui/reports/financial.tsx`. [C] A dashboard tile that says "Net profit" must
   equal the statement it opens, so a pure `core/plFigures.ts` has to be extracted first (section 3.9). This is the one real
   prerequisite.
5. **The default Dashboard costs 393 ms of main-thread time at 250,000 vouchers, and a fuller Zoho-style board on today's
   pattern would cost over a second.** Each widget makes its own full pass over the vouchers (balances, period totals, bills,
   stock). [V, table in 2.3] **One pass that builds a ledger-by-bucket debit/credit index costs 55 ms at 250,000 vouchers**
   (prototype, same cost as `closingBalances` alone, 52 ms) and answers the balances, the totals, the trends, the cash flow and
   the profit inputs together. [V]
6. **A Zoho-style cash-flow chart needs a rule the current Cash Flow report does not have.** The report counts every debit leg
   on a cash or bank ledger as inflow and every credit leg as outflow, so a transfer between two of your own accounts is both.
   [C `ui/reports/financial.tsx:48-62`] In the synthetic 250,000-voucher book, transfers inflate both sides by about 773 million
   of 1,570 million gross inflow [V] (the synthetic mix is an assumption; real books will differ, the mechanism will not).
   Zoho's help does not say how it treats transfers **[unverified]**. Decision D3.
7. **A cash/accrual toggle on Income and Expense is not computable correctly from our data without a bill-by-bill engine, so it
   is not in v2.** Zoho has the toggle. [Z-home] Section 3.5 gives the three options and the reason.
8. **A donut for Top Expenses is the wrong chart, and the chart colours we already ship are not chart colours.** The five
   `--tone-*` tokens are KPI stripe accents; used as a five-series palette they **warn in light and fail in dark** (dark
   lightness band; red against green at delta-E 3.6 under deutan simulation). [V, 3.6] A four-slot series palette that
   passes in both schemes is proposed and validated. [V]
9. **The Dashboard is the only screen a restricted operator cannot see for the wrong reason.** `SCREEN_AREA.dashboard` is
   `reports`, so a clerk with Accounting but not Reports gets a gate screen as the home page. [C `core/access.ts:123`] Zoho
   limits dashboard insights by role instead (watchlists and dashboard reports per role). [Z-notes][Z-new] Design: the
   screen is open to everyone and **each widget is judged by the right its drill target needs**, the pattern the Reports
   Center already uses.

**The decisions, in one table** (full reasoning in the sections named; open ones are repeated in section 8 for the owner).

| # | Question | Decision | Section |
|---|---|---|---|
| 1 | Shape of the home | A header band (period, as-at, Customize, `+ New`), a Today strip, a conditional Setup card, then **cards in a 12-column grid**: Receivables and Payables (current/overdue bar, ageing), Cash flow, Income and expense, Cash and bank accounts, Top expenses, Overdue invoices, GST due, Latest vouchers, Stock alerts. 22 widgets in the catalogue, 12 on by default. | 3.2, 3.4 |
| 2 | Period | **One selector, the existing open period** (`F2` plus presets), because a tile must equal the report it opens. Balances are "as at" a separate switch: **Today (default) or Period end**. | 3.3 |
| 3 | Income and expense basis | **As booked (accrual) only, labelled.** The cash view is the Cash flow widget. No toggle. | 3.5 |
| 4 | Cash flow rule | Per-voucher net on cash and bank, so own-account transfers vanish; the Cash Flow Summary report gains the same option, default on. **Owner to confirm** (D3). | 3.4 |
| 5 | Charts | Inline SVG kit, no dependency: columns, a stacked bar, ranked bars, sparkline; every chart has a table twin, keyboard focus per mark and a tooltip that never gates. Donut replaced by ranked bars. | 3.6 |
| 6 | Layout engine | Keep the shipped list-of-`{id,on,size}` model (12 columns, S/M/L, personal beats company beats built-in); add per-widget size limits, in-board edit with pointer drag, a keyboard grab/move/resize model, per-widget config, optional role defaults. | 3.7 |
| 7 | Drill-down | Every number and mark opens the report it came from, with the period set; the filter arrives with Wave A's engine. Esc returns, and a chip offers to restore the period. | 3.8 |
| 8 | Speed | **One memoised index pass**, bills and stock as separate lazy passes, nothing computed unless the Dashboard is on top. | 3.9 |
| 9 | Rights | Dashboard open to all; each widget gated by area and feature; unseen widgets are neither drawn, listed nor computed. | 3.12 |
| 10 | MVP | Phase 0 and 1: about 17 to 20 builder-days, three builders in parallel in about two weeks. | 4 |

---

## 1. How the Zoho Books home dashboard works (what Zoho documents)

### 1.1 The page and its widgets
The Home page opens on a **Dashboard** tab, with sibling tabs for **Getting Started**, **Announcements** and **Recent
Updates**. [Z-home] The Dashboard tab is a set of panels (Zoho calls them panels or components). The India edition lists the
same set and shows **no GST or other India-specific panel**. [Z-india]

| Panel | What it shows | Click-through | Period | Notes |
|---|---|---|---|---|
| Total Receivables | What customers owe, split **Current** (not yet due) and **Overdue** | Either figure opens the AR Aging Details report by invoice due date | A dropdown beside Overdue picks a time range | `+ New` menu inside the panel: Invoice, Recurring Invoice, Customer Payment [Z-home][Z-india] |
| Total Payables | What you owe vendors, Current and Overdue | AP Aging Details by bill due date | Same dropdown | `+ New`: Bill, Vendor Payment, Recurring Bill [Z-home] |
| Cash Flow | Cash at the start and end of the fiscal year, plus incoming and outgoing, drawn as a graph | Not stated [Z-india] | "This Fiscal Year" control, top right, opens a timeline dropdown | Incoming is the debit side of bank, card, cash and payment-clearing accounts, outgoing the credit side; drawn by the **direct** method, so it differs from the indirect-method Cash Flow Statement report [Z-home] |
| Income and Expense | Income (sales and bank deposits) against expenses (purchases and bank withdrawals) | Clicking the bars opens Profit and Loss | Same selector; a **Cash / Accrual** toggle, top right | [Z-home][Z-india] |
| Top Expenses | A pie of the largest expense categories | A slice opens the matching transactions in reports | Same selector | Built from the Expense by Category report (stated for the sibling Zoho Invoice product) [Z-home][Z-invoice] |
| Projects | Projects with client, unbilled hours, unbilled expenses; tabs switch between the two; "Show All Projects" | A project opens its details in Timesheets | none stated | We have no projects module |
| Bank and Credit Cards | The bank accounts and cards added to the organisation | An account opens its transactions | none stated | Balances shown **[unverified]**: the help names the list, not the figures [Z-home] |
| Account Watchlist | Near-real-time balances of accounts you choose, with an Accrual / Cash toggle | not stated | none | Chosen by ticking a watchlist box on the account in Chart of Accounts (the same flag appears in Zoho's API as a show-on-dashboard boolean, from a search-result summary, page not opened) [Z-home] |

**Where the receivables numbers come from.** Zoho says the Total Receivables and Total Payables figures are taken from its AR
and AP Aging Details reports, which show data **as of the date you view them**; a payment dated in the future still counts as
overdue until that date arrives. [Z-kb-overdue] So the dashboard figure is reconcilable to a named report, which is the
property our tiles must keep.

**Smaller dashboards elsewhere (the same idea, per module).** A "mini dashboard" on the invoices list gives outstanding
receivables, due dates and average days to get paid (release notes, September 2020; the two Zoho pages that carry it differ
by one day). A dashboard section in Bills shows outstanding payables (January 2023). [Z-notes][Z-new] So Zoho's pattern is
a few headline figures **at the top of the list they summarise**, not only on Home. That is a Wave A+ idea for us, not v2.

### 1.2 Period, basis and refresh
- **Period:** a per-panel "This Fiscal Year" dropdown on Cash Flow, Income and Expense and Top Expenses; a time-range dropdown
  on the Overdue figures. The **list of options is not in the help** **[unverified]**; the screenshots the owner described
  are gone (zoho-books-reference.md, section 0). One selector per panel, not one for the page.
- **Basis:** Cash or Accrual on Income and Expense and on the Watchlist. A Zoho community answer (via a search summary, not
  opened) says cash-basis balances cannot be shown directly on the dashboard except through the watchlist **[unverified]**.
- **Refresh:** nothing in the help describes a refresh control or how often panels recompute **[unverified]**. Bank feeds
  refresh when you sign in (banks with multi-factor sign-in need a manual refresh), which is about feeds, not panels (from a
  search summary of a Zoho banking help page, not opened).
- **Empty and first-run states:** the help does not describe how panels look with no data **[unverified]**. Setup guidance is
  the separate **Getting Started** tab (company profile, bank accounts, first invoice, tax settings). [Z-home]

### 1.3 Customising: what is and is not allowed
- **The default dashboard is locked.** It cannot be edited, reordered or deleted. [Z-custom]
- **Custom dashboards** (Elite and Ultimate plans): create from **+ New Dashboard** with a name and optional description;
  add **predefined panels** (Zoho's built-in ones) or **report panels** (a chart built from a report such as Profit and
  Loss, Sales by Customer, Receivable Summary, Bills Details, or a custom version). A report panel has a time period, a
  chart type (pie, bar or line), X and Y data points, a sort order and a maximum number of points. [Z-custom]
- **Layout:** drag panels on the Dashboard Setup page to reorder; **only report panels can be resized**, by dragging a corner;
  nothing is saved until Save Changes. [Z-custom]
- **Sharing:** Only Me, Everyone (all users may view and manage), or Only Selected Users and Roles with View Only or View and
  Manage. Administrators can see and manage all custom dashboards by default. A dashboard list groups Created by Me, Shared
  With Me, Created by Other Users (admins only) and System (the default). Up to three can be marked favourite; the tab shows
  two of them beside the default. [Z-custom]
- **Later additions:** custom dashboards built from predefined components, marketplace widgets and reports with reorder,
  resize and permissions (1 July 2024); sortable report panels with a data-point limit, Ultimate only (19 June 2025);
  inventory panels such as sales-order summaries and top-selling items when the Inventory add-on is on (10 June 2026). [Z-new]

### 1.4 Admins against everyone else
- Admins have full access; **Staff** can use all modules except reports, settings and accountant features. [Z-users]
- "Dashboard insights" can be restricted by role-based permissions: a September 2019 note calls it limiting access to the
  dashboard **watchlist** and dashboard **reports** per role. [Z-notes][Z-new] A role's dashboard section picks which custom
  dashboards it sees, each needing View Access, plus an **Allow Dashboard Management** switch. [Z-custom]
- Whether the *default* dashboard hides the Receivables or Payables panel from a role without report access is **not
  documented** **[unverified]**. The two role pages I opened (users, custom role) say nothing about the dashboard. [Z-users][Z-roles]

### 1.5 Newer and adjacent things
- **AI Summary dashboard** (Zoho's blog calls it Zia Summary): an auto-generated tab beside the default and custom
  dashboards that pulls profit-and-loss trend, projected cash balances, current against overdue receivables and payables,
  days-to-pay, near-term dues and vendor concentration into one view, with suggested actions. **Early Access, Books only**,
  published 17 December 2025, updated 17 April 2026. Whether it appears on the default Home is **not confirmed**. [Z-blog]
- **Mobile app:** the screenshots show tiles for bank summary (cash on hand, payments clearing, bank balance), total
  receivables and overdue invoices and bills; the page does not describe the home screen in text. [Z-mobile]
- **Not found in any Zoho page I read:** a refresh control, an empty-state description, the period dropdown's options, and
  what a restricted role sees on the default Home.

### 1.6 What this means for our design
Zoho's dashboard is **a fixed set of seven or eight answers with a period dropdown per chart and a one-click path from
every number to a named report**. It is intuitive because of three habits we copy: (1) a number is always split into the
two parts a person acts on (current and overdue); (2) a chart always opens the report behind it; (3) a create action
sits inside the panel it relates to. Its weaknesses are our openings: a locked default, per-panel periods that let two panels
disagree about what "this year" means, and nothing for the first week.

### 1.7 Limits of this research
Summaries from the fetch can drop detail; the Zoho help is regional and uneven; the help page `zoho.com/books/help/dashboard/`
returned an access error when opened (a search summary of it matched the pages above). Confirm in a live Zoho Books
organisation before locking: the period dropdown's options, whether `+ New` inside Receivables is gated by role, and what a
Staff user sees on Home. The dashboard figures in sections 2 and 3 are ours and do not depend on those answers.

---

## 2. Our Dashboard today (read, run, timed)

### 2.1 What exists
[C] `src/ui/modern/Dashboard.tsx` (123 lines), `dashWidgets.tsx` (340), `DashCustomise.tsx` (206), `useDashLayout.ts`,
`dashStore.ts`, `src/core/dashboardLayout.ts` (253, pure), `dashboardData.ts` (271, pure), `src/styles/dashcustom.css`, and the
suite `scripts/assert/dashboard-custom.assert.ts` (794 lines).

| Aspect | Today |
|---|---|
| Widgets | **15**: 9 tiles (cash, receivables, payables, sales, stock, overdue, purchases, GST payable, TDS to deposit), 6 cards (trend, Quick Entry and Reports, Recent Vouchers, top customers, top items, stock alerts). **8 on by default**: the six tiles, Quick Entry, Recent Vouchers. |
| Grid | 12 columns; a tile takes 3/6/12 for S/M/L, a card 4/6/12; narrower rules by container query at 860 and 460 px; a run of tiles and a run of cards always start separate rows. |
| Layers | personal (this operator, this device, `localStorage`) beats company (`Company.dashboard`, synced, an administrator writes it with `C`) beats built-in. Unknown saved ids are kept and written back; a widget the saved list lacks is appended **off**. |
| Keys | `F12` opens the Customize panel (docked at the right): up/down pick, space show/hide, left/right or S/M/L size, shift+up/down move, `R` reset, `C` company layout, `X` remove it, `Esc` done. `Esc` otherwise leaves the screen. **`F2` is not bound.** |
| Figures | Each is "a thin reading of an engine a report already uses"; a tile that is not on the board is never computed. |
| Tiles | `div` with `onClick`: no role, no tab stop, no focus ring [C `dashWidgets.tsx:91`]. |
| Trend card | CSS flex columns scaled to the tallest bar, a "Tallest bar" note instead of an axis, a `title` tooltip, every column drills to the Sales register (not to the month). [C `dashWidgets.tsx:166-215`] |
| Recent Vouchers | `st.vouchers.slice(-n).reverse()`: the latest **dated** vouchers (the log is sorted by date), not the latest entered, and future-dated ones included. [C `dashWidgets.tsx:135`] |
| Rights | The whole screen is `reports` area. [C `core/access.ts:123`] |
| F-key rail | None: the right-hand 152 px column disappears on this screen only. [E, ux-analysis T9] |

### 2.2 What the harness shows
`npm run shots -- --screens dashboard,dc-rich,dc-panel,dc-empty,dc-trend` (light, compact, 1440x900) and the dark/comfortable and
1100-wide variants; all ran clean. [E, V]
- **Default board (sample company):** four tiles in one row (Cash and Bank 2,56,994.00; Receivables 2,40,838.00; Payables
  51,599.00; Sales 3,85,768.00), a second row of two (Stock Value; Overdue 1,61,068.00 in red), Quick Entry beside Recent
  Vouchers. Every tile foot reads "as at 31-Mar-27". There is a "+ New Sales Invoice" button **and** a Sales Invoice button in
  Quick Entry on the same screen. No chart, no profit, nothing that says what is late or who.
- **With every widget on:** nine tiles wrap 4 + 4 + 1, leaving one orphan tile; the trend card shows seven coloured months
  and five empty ones (the months after the pinned date), no axis, no gridlines, no marker for "not yet". The GST tile shows
  1,991.36 "net of input credit" (see finding 3). TDS shows 0.00 with no date.
- **Customize panel:** 15 rows with a checkbox, a blurb, a Tile/Card tag and S/M/L; when it docks, the board reflows to two
  tiles per row. It is good. It lists widgets that exist; it cannot offer one that does not.
- **1100 px wide:** tiles go two up and Quick Entry takes the full row, pushing Recent Vouchers below the fold. The shell
  itself does not fit below 938 px (ux-analysis, section on responsiveness), so true phone layout is outside this document.
- **Dark:** the tiles' uppercase, tracked labels and the coloured left stripes are the existing KPI look; the trend bars use
  the dark tone tokens (lighter than the chart band, see 3.6).

### 2.3 Measured cost (`npm run perf -- --only=dash --sizes=50k,250k --runs=3`, Apple M4, load average 4.8, engine time only)
[V] Pure-engine time on the main thread; no React render, layout or paint. The harness itself says the 100 ms line is a reference
line, not a gate for these rows.

| Row (what the Dashboard runs today) | 50,000 vouchers | 250,000 vouchers |
|---|---|---|
| Default tiles (`computeDashMetrics`: closing balances, period totals, bills, stock value) | 77 ms | **393 ms** |
| GST payable tile (a whole GSTR-3B over the open period) | 96 ms | **446 ms** |
| Opt-in cards (trend, top customers, top items, stock alerts) | 45 ms | **217 ms** |
| All three together, if a user switches everything on | | **about 1,060 ms** |

And the same engines timed one by one at 250,000 vouchers with a scratch script (p50 of 5 runs, same synthetic book): closing
balances 52 ms, period totals 53 ms, outstanding bills **153 ms**, stock positions **124 ms**, stock value 117 ms, GSTR-3B
for one month 69 ms against 406 ms for the year. [V] Because AppShell keeps every stacked screen mounted and `useDashMetrics`
depends on `st.vouchers`, **the Dashboard recomputes on every save while it sits anywhere in the navigation stack**, even
hidden. [C `dashWidgets.tsx:33-44`, perf harness note `dash.kpis`] An old office PC may be several times slower (the harness has
`--cpu-factor` for that projection; I did not run it).

### 2.4 Gaps found, numbered (each with its evidence)
| # | Gap | Evidence |
|---|---|---|
| G1 | Balances stamped with the period end, not today | E, C |
| G2 | No period control; hero says "FY" for any period | C `Dashboard.tsx:77` |
| G3 | GST tile builds the open period, blurb says the current month | C |
| G4 | No profit, expense, income or cash-flow widget | C, E |
| G5 | P&L maths is not a pure function (two copies in components) | C `reports.tsx:1840+`, `financial.tsx:190+` |
| G6 | Receivables tile is the Sundry Debtors **ledger balance**; overdue is **bill-wise positive balances**: they need not add up (on-account credits and non-bill-wise parties sit in one and not the other), and the page never reconciles them | C `dashboardData.ts:244-245,262` against `billwise.ts:18-97` |
| G7 | Cash Flow Summary counts own-account transfers as flow | C, V |
| G8 | No overdue list, no bank-account list, no "what needs attention" | C, E |
| G9 | Tiles and trend columns are not buttons (mouse only) | C |
| G10 | Trend chart: no axis, no future-month marker, drills to the register not the month | C, E |
| G11 | "Recent" means latest dated, future-dated included | C |
| G12 | Dashboard needs the `reports` right | C |
| G13 | Whole-board recompute on every save while stacked; each widget its own pass | V |
| G14 | A tab left open overnight keeps yesterday's "today" (no midnight refresh) | C (no timer in `dashWidgets.tsx`) |
| G15 | Empty company: six zeros and a keyboard hint | E, ux-analysis F-08 |
| G16 | The shipped default is pinned by a test ("the default IS the dashboard the app always drew") | C `dashboard-custom.assert.ts` |

---

## 3. The design

### 3.1 Principles
1. **A tile never disagrees with the report it opens.** Every widget declares its engine and its drill target, and a test
   reconciles the two (the shipped suite already does this for the tiles; v2 extends it to every widget).
2. **Nothing is decided by colour alone,** and no number is only in a tooltip (table twin, direct labels).
3. **Split every headline into the parts a person acts on** (current and overdue; in and out; income and expense).
4. **One period, one "as at".** The page has one range selector; balance widgets say what date they are at.
5. **Calm by default, deep on demand.** Twelve widgets on by default, ten more one checkbox away.
6. **A widget you may not see is not drawn, not listed and not computed.**
7. **Compute once, only when the Dashboard is on top,** and never block a keystroke.
8. **Keys never change meaning** (the superset rule): `F12` stays Customize, `Esc` stays leave, `F2` becomes Period as on every
   report. The Gateway `W` still opens the Dashboard.
9. **Everyday words first** (Receivables, Money in, Money out), Tally names as aliases in the Customize search.

### 3.2 Page anatomy (1,012 px board at 1440 wide, 212 px rail)
```
Good morning                          [ Period: FY 2026-27 v ]  [ As at: Today v ]  [ Customize F12 ]  [ + New v ]
Lakshmi Office Supplies (Sample) . 1-Apr-2026 to 31-Mar-2027 . balances as at 12-Oct-2026 (today)
+------------------------------------------------------------------------------------------------------------+
| Today, Mon 12 Oct   Sales x (n)   Received x   Paid x   n vouchers dated today   n entered today           |
+------------------------------------------------------------------------------------------------------------+
| Get started   3 of 5 done  [======----]   Next: take a backup                                  [ Hide ]    |   (until done)
+-------------------------------------------------+----------------------------------------------------------+
| Receivables                       [ + New v ]   | Payables                                  [ + New v ]    |
| x                                               | x                                                        |
| [ Current x ][ Overdue x ][ Unapplied x ]       | [ Current x ][ Overdue x ][ Unapplied x ]                |
| 0-30   31-60   61-90   90+   (four steps)       | 0-30   31-60   61-90   90+                               |
+-------------------------------------------------+----------------------------------------------------------+
| Cash flow         Opening x > Closing x         | Income and expense              Net profit x             |
|  columns: Money in / Money out by month         |  columns: Income / Expenses by month, "as booked"        |
|  table twin                                     |  table twin                                              |
+-------------------------------------------------+----------------------------------------------------------+
| Cash and bank                                   | Top expenses                                             |
|  Bank account      x      n not reconciled      |  Ledger       ========   share %                         |
|  Cash              x                            |  Ledger       ======     share %  (ranked bars, no pie) |
+-------------------------------------------------+----------------------------------------------------------+
| Overdue invoices (oldest first)                 | GST due: GSTR-3B for <month>, by <due date> (n days)     |
| Latest vouchers                                 | Stock alerts                                             |
```
The sketch carries **no figures** (x and n are placeholders): the numbers the harness showed are balances at the period end,
which the new as-at rule changes, so none is reused here. It is a layout intent, not a pixel spec; spans are in 3.7.

**Header.** Left: greeting, company, the window and the "as at" date in words. Right: **Period** (a button that opens a
list: This month, Last month, This quarter, Last quarter, This financial year, Last financial year, Custom `F2`),
**As at** (Today or Period end), **Customize `F12`**, **`+ New`** (the same split button the list pages get in Wave A: primary
"Invoice", related Receipt, Bill, Payment, Journal, from `ui/quickCreate.ts`; the existing "+ New Sales Invoice" button becomes
it, and the Quick Entry card is off by default because the universal `+` and `+ New` do that job).
**The Dashboard gets a button bar** (F2 Period, F12 Customize, Esc Quit) so the 152 px column stops vanishing. [ux-analysis T9]

### 3.3 The period selector and "as at"
- **The window is the open period** (`st.period`), the same value every report uses. Presets call `st.setPeriod`. This is
  decision D1: the alternative (a dashboard-local range, with drills setting the period at the moment of drilling) keeps the
  reports undisturbed but lets the Dashboard and a report disagree about "this year" after an `F2` elsewhere. A Tally user
  expects one global period (`F2`), and the Cash Flow report already sets it when you press Enter on a month. [C
  `financial.tsx:69-71`]
- **Presets** are computed from the company's financial year (`Company.finYearFrom`) and `today`: month and quarter are
  clamped inside the open financial year; custom opens the existing period prompt. A chip shows the range in words and
  **"Not the financial year: reset"** whenever the window is not the whole year, so a narrowed window can never be mistaken
  for the year after a drill.
- **"As at"** is `asOn = min(period.to, max(today, period.from))`: today when today is inside the window; the period end for a
  closed year; the period start for a window wholly in the future. **Default Today** (this is what Zoho's aging-based figures
  do [Z-kb-overdue]); **Period end** for an accountant closing a year. The switch is stored per operator per company on this
  device. The Overdue tile already measures lateness against `min(period end, today)` (`ageingAsOn`), but reads its bill
  balances at the period end, so a future-dated receipt is still counted; the new rule reads both at the same date. Every
  balance widget prints its date ("as at 12-Oct-26"); no widget prints a date in the future.
- **Which widgets follow which:** flow widgets (Income and expense, Cash flow, Top expenses, Sales and purchases trend, top
  customers and items, period tiles) use the **window**; balance widgets (Cash and bank, Receivables, Payables, Stock value,
  Overdue, TDS) use **as at**; the Today strip ignores both; GST due uses its own **filing period** (3.4).
- **Chart buckets follow the window:** up to 31 days, one column per **day**; 32 to 120 days, per **week** (Monday start);
  longer, per **month**; more than 24 months, per **quarter**. One function, `bucketsFor(from, to)`, pure and asserted at the
  boundaries (leap day, a financial year crossing a calendar year, a one-day window, a 31-day month).
- **Keys:** `F2` opens the period list (existing prompt below the presets; underlined letters pick a preset: `M` this month,
  `L` last month, `Q` quarter, `Y` financial year, `C` custom). The midnight rule: a timer sets `today` again at local
  midnight so an overnight tab is correct (G14).

### 3.4 The widget catalogue
Each row is a contract: what it reads, how it draws, what the period does to it, where it goes, who may see it. **Sizes:** S, M, L
per widget as listed (3.7 gives the columns). **Default** is the built-in layout for a company with nothing saved; a saved
layout is untouched (an added widget arrives **off**, the shipped rule). Rights are the `RightArea` the figure reveals (3.12).

| id (existing = kept) | Widget | Reads (engine) | Form | Window / as at | Drill target | Rights . feature | Default |
|---|---|---|---|---|---|---|---|
| `today` (new) | Today strip | index day bucket for today; `useToday` (change log) for "entered today" | stat strip, no chart | always today | Day Book at today | accounting | L, on |
| `setup` (new) | Get started | `onboarding/progress.ts` `stepsDone` plus three owner steps (company details, first customer, first invoice) | checklist with progress meter | n/a | each step's action | open | L, on until done or hidden; not drawn in the sample company |
| `receivables` (kept) | Receivables | debtors group balance at as-at (index); bills pass for the split | **S** tile; **M** card: headline, current/overdue/unapplied bar, `+ New`; **L** adds four ageing steps | as at | whole card: Receivables; ageing steps: Ageing Analysis; `+ New`: Invoice, Receipt, Credit Note (`QC_PAGES['outstandings:side=receivables']`) | accounting . billwise for the split | M, on |
| `payables` (kept) | Payables | creditors group at as-at; bills pass | as above | as at | Payables; `+ New`: Bill, Payment, Debit Note | accounting . billwise | M, on |
| `overdue` (kept) | Overdue (tile) | bills pass | S tile, red | as at | Receivables | accounting . billwise | off (the card above carries it) |
| `overdueList` (new) | Overdue invoices | bills pass, oldest first | list: customer, bill, due, days late, balance; bucket chip | as at | row: that party's bills (`billsOfLedger`); "Reminders": `reminders` | accounting . billwise | M, on |
| `cash` (kept) | Cash and bank | cash, bank, bank OD ledgers at as-at (index); entries without a bank date per bank ledger | **S** tile (total); **M/L** account list: name, balance, "N not reconciled" | as at | row: that ledger's vouchers; the count: Reconciliation | accounting | M, on |
| `cashflow` (new) | Cash flow | index: per-voucher net on cash and bank per bucket; opening and closing cash | two-series columns Money in / Money out, plus a headline "Opening > Closing, net" | window | a column: Cash Flow Summary at that bucket; headline: Cash Flow Summary | reports | M, on |
| `incomeExpense` (new) | Income and expense | index: Income and Expense groups per bucket; **headline from `plFigures`** | two-series columns Income / Expenses, headline Net profit | window; labelled "as booked" | a column or the headline: Profit and Loss at that bucket | reports | M, on |
| `profit` (new) | Net profit | `plFigures` | S tile with change against the previous equal window | window | Profit and Loss | reports | off |
| `topExpenses` (new) | Top expenses | index: expense ledgers (direct and indirect expenses; Purchase Accounts left out unless asked) ranked | **ranked horizontal bars**, share %, "Other" row | window | row: that ledger's vouchers; "All": Group Summary | reports | M, on |
| `sales`, `purchases` (kept) | Sales, Purchases (period) | index | S tile | window | the register | accounting | off |
| `trend` (kept) | Sales and purchases | index | two-series columns (the shipped card rebuilt on the chart kit) | window | a column: the register at that bucket | accounting | off |
| `topCustomers`, `topItems` (kept) | Top customers, Top items | `rankParties`, `rankItems` (unchanged engines) | ranked bars | window | the party's or item's vouchers; "Top-N Analysis" | accounting; inventory | off |
| `gstPayable` (kept, renamed "GST due") | GST due | `buildGstr3b` over the **filing period** (month, or the quarter under QRMP via `qrmpPeriod`) | **S** tile (cash payable); **M** card: period, taxable value, output tax, ITC, cash payable after Rule 88A set-off, the statutory due date | own period | GSTR-3B (the GSTR-1 line opens GSTR-1) | gst . gst | M, on |
| `tdsPayable` (kept) | TDS to deposit | ledger closing balances (`tdsPayable`) | S tile; M card by section later | as at | TDS Outstanding | gst . tds | off |
| `stock` (kept) | Stock value | `stockValueAsOn` | S tile | as at | Stock Summary | inventory . inventory | off |
| `stockAlerts` (kept) | Stock alerts | `stockAlerts` over `stockPositions` | list with Negative / Low tags | as at | row: that item's vouchers; "Reorder Status" | inventory . inventory | M, on |
| `recent` (kept, relabelled "Latest vouchers") | Latest vouchers | log tail with `date <= today` | list | none | the voucher | accounting | M, on |
| `quick` (kept) | Quick Entry and Reports | static | buttons | none | various | per button | off |
| `watchlist` (new, Phase 3) | Account watchlist | closing balance of up to 8 chosen ledgers | list | as at | the ledger's vouchers | reports | off |
| `highlights` (new, Phase 4) | Highlights | rule-based sentences over the index (no model) | list | window | the report each names | per rule | off |

Counts: **22 widgets by the end of Phase 2**: the 15 that exist plus seven new ones (`today`, `setup`, `overdueList`,
`cashflow`, `incomeExpense`, `profit`, `topExpenses`); `watchlist` and `highlights` arrive in Phases 3 and 4, making 24.
**On by default: 12** (`today`, `setup`, `receivables`, `payables`, `cashflow`, `incomeExpense`, `cash`, `topExpenses`,
`overdueList`, `gstPayable`, `recent`, `stockAlerts`). Three are hidden when the company feature is off (Overdue invoices
needs bill-wise, GST due needs GST, Stock alerts needs inventory) and Setup is hidden once it is done. The mapping to
Zoho: Total Receivables and Payables > `receivables`, `payables`; Cash Flow > `cashflow`; Income and Expense > `incomeExpense`;
Top Expenses > `topExpenses`; Bank and Credit Cards > `cash`; Account Watchlist > `watchlist`; Getting Started tab > `setup`;
Projects > **no equivalent** (closest are the CRM follow-ups and pipeline screens, a Phase 4 candidate); Zia Summary >
`highlights`, rule-based and named plainly (the Zoho brand is not ours).

**The rules behind the tricky rows.**

*Receivables must add up (G6).* The headline is the Sundry Debtors balance at as-at, the same figure the Receivables report
and the old tile show. The bar splits it into **Current** (bills not yet due), **Overdue** (bills past due, the shipped
`overdueTotal` rule) and, when non-zero, **Unapplied and not bill-wise** (what is left: on-account receipts, credit notes
not set against a bill, parties that are not bill-wise). The three segments therefore sum to the headline, and the third
segment is a link to the explanation, not a hidden gap. Without the billwise feature the card shows the headline only, with
one line saying how to see the split. A test fails if the segments do not sum to the headline.

*Cash flow must not count your own transfers (G7).* Rule: for each voucher, `net = sum(Dr legs on cash, bank and bank OD)
minus sum(Cr legs on the same ledgers)`; a positive net is money in, a negative net is money out, a zero net is nothing. That
cancels a Contra between accounts and a same-voucher pair, and it is exact for a Receipt, a Payment and a multi-line
voucher. It **differs from today's Cash Flow Summary**, which counts legs. Options: (A) give the report the same rule as an
F12 option, "Count transfers between my own accounts: No" (default), keeping the leg view one keypress away, so the widget
and its drill agree; (B) leave the report and footnote the widget. **Recommendation A** (D3) because owners read a
doubled bar as a mistake. Opening and closing cash come from the index (cash and bank balances at window start minus one
day and at the window end).

*Income and expense must reconcile to Profit and Loss (finding 4).* Monthly columns are **Income** (Sales, Direct and Indirect
Incomes: credit less debit) and **Expenses** (Purchases, Direct and Indirect Expenses: debit less credit), as booked. The
headline Net profit is the **Profit and Loss figure for the window**, which also carries the change in closing against
opening stock. When there are stock items the card prints one reconciling line, "Change in stock +x", so Income minus Expenses
plus that line equals the headline; without stock items the line is absent and the two agree. A test asserts equality with
the Profit and Loss screen's own numbers.

*GST due uses the filing period, not the open period (G3).* Rule: the **next return whose statutory due date has not passed**.
Monthly companies: GSTR-1 on the 11th and GSTR-3B on the 20th of the following month. QRMP companies: quarterly GSTR-1 by the
13th, GSTR-3B by the 22nd or 24th of the month after the quarter (24th or 22nd by the state's notified category), with the
first two months' tax paid by challan (PMT-06) by the 25th. [GST portal help pages, via search summaries, section 10] The day
after a due date the card rolls to the next period and says "month in progress" until it ends. **Two honest limits, printed on
the card:** the books cannot know whether the return was filed, and due dates can be extended by notification, so the card
says "statutory due date; check the portal for extensions". The dates live in a new pure module with a source note on each
constant (the Phase 24 rule: statutory figures are researched and cited, never remembered), and the state-category question
for QRMP is a data table the owner confirms. Cost: one GSTR-3B build of a month (69 ms at 250,000 vouchers [V]), cached by
(vouchers identity, period).

*TDS keeps no date until the open item closes.* `ROADMAP-ADDENDUM.md` (the Dashboard row) records that the deposit dates
under the Income-tax Act 2025 were not researched. `core/tdsInterest.ts` has `depositDueDate`, but until that item is closed
the tile shows the amount and a link to the TDS quarterly tracker, no date.

*Top expenses is a definition, not a given.* Expense ledgers are those under Direct Expenses and Indirect Expenses; Purchase
Accounts are cost of goods and appear in the Income and expense card and the Purchases tile, not here, unless the widget's
config says "include purchases". Ranked by window movement (debit less credit), top 5, 8 or 12 by size, the remainder as
"Other".

*Cash and bank "not reconciled".* For each bank ledger, the count of entries with no bank date (or one after as-at), counted in
the index pass (the same test `brsPosition` uses for "reflected"). The count opens Reconciliation for that bank.

*The Today strip counts what is dated today*, not what was entered today; the second number ("entered today") is the existing
change-log count the Quick access panel uses. [C `nav/QuickAccess.tsx:67-100`] They differ whenever someone back-dates.

*Latest vouchers.* Dated up to today, newest first, so a future-dated voucher no longer heads the list (G11).

### 3.5 Cash against accrual: decided, with the reasons
The owner's list said "with a cash/accrual-style toggle only if we can compute it correctly". We cannot, yet.

| Option | What it takes | Verdict |
|---|---|---|
| **Cash-basis profit** (recognise income when the receipt arrives, expense when the payment leaves) | For every Sales voucher, find the receipts that settled it (bill-wise `Agst Ref` allocations), recognise each income and tax leg pro rata, handle advances, on-account receipts, credit notes, journals that settle bills, discounts, TDS deducted by the customer, and parties that are not bill-wise at all. Correct only when every debtor and creditor ledger is bill-wise and fully allocated. | **Not in v2.** A 3 to 4 week engine with an audit **[J]**; to my knowledge TallyPrime has no cash-basis P&L either **[unverified]**. A Phase 4 spike may measure how many real books are fully allocated; the toggle would then appear only above a coverage threshold and say so. |
| **A toggle that approximates** (income = receipts, expense = payments) | Cheap | **Rejected.** It is not cash-basis profit (it counts transfers, loans, capital and tax payments as income or expense) and would sit one click from the Profit and Loss it contradicts. |
| **Two honest views side by side** | The Cash flow card (money in and out, exact) next to the Income and expense card (as booked, labelled) | **Chosen.** The cash view is exact because it is a count of cash movements; the accrual view is exactly what the books say. The two cards answer "did I get paid" and "did I make money" without pretending to be one chart. |

A small addition for owners who want the intuition: the Receivables card already shows how much of what you invoiced is
still unpaid. (Zoho's own description of its income figure, sales plus bank deposits, suggests its toggle is not a strict
accounting basis either **[inferred]**. [Z-home])

### 3.6 Charts without a dependency
**What the repo does today** [C]: no chart library and no chart component; the trend card is CSS flex bars (`.dtr`,
`dashcustom.css`), the ranked list is CSS bars (`.dtop`), coloured with `--tone-violet` and `--tone-amber`; inline SVG
appears only for icons, tick marks and the QR code. Tokens live in `src/styles.css` (surfaces `--panel`, `--sunk`; text `--ink`,
`--ink-2`, `--dim`; status `--crit`, `--warn`; spacing `--sp-*`; radii; elevation), with the rule that **a hex never goes
in `:root` for a token dark does not override** (light-only tokens go in the `:root[data-scheme='light']` block 5b).

**Method.** I used the repo's `dataviz` skill (form first, colour by job, validate, marks, hover, accessibility) and ran its
validator on candidate palettes against the card surface (`--panel`: `#ffffff` light, `#141c19` dark). [V]

**Findings from the validator.**
- The five `--tone-*` tokens as a categorical series set: **light WARN** (red against green at delta-E 7.3 under deutan,
  allowed only with direct labels), **dark FAIL** (all five sit above the dark chart lightness band, and red against green is
  3.6). They are KPI stripe accents ("which metric, not how worried should I be", the file's own comment) and stay that.
- The brand accent `#0b7266` **fails the chroma floor as a series colour** (reads gray, 0.086 against 0.10).
- A **four-slot series palette passes every check in both schemes** (lightness band, chroma, adjacent CVD separation, normal
  vision floor, contrast of at least 3:1 against the surface):

| Slot | Role | Light | Dark |
|---|---|---|---|
| `--viz-1` | the first series: Income, Money in, Sales | `#2a63bd` (blue) | `#4a8bea` |
| `--viz-2` | the second: Expenses, Money out, Purchases | `#8f6206` (amber) | `#c08a1c` |
| `--viz-3` | a third when needed (a third bucket, Other-by-name) | `#07907e` (teal) | `#17a091` |
| `--viz-4` | a fourth (rare) | `#5d4bd0` (violet) | `#8a76ea` |

  Worst adjacent CVD separation in the set is above 11 in both schemes, normal-vision floor above 17. Pairs actually used
  (1 and 2) separate at delta-E 24.8 light and 27.7 dark.
- **Ageing buckets are ordinal, not categorical:** one amber ramp, four steps, validated with the ordinal check. Light
  `#d9a640 > #bd8420 > #9a650a > #6f4408`, dark `#7a5a1c > #a8801f > #d1a23a > #f0cb74` (0 to 30, 31 to 60, 61 to 90, over 90
  days). The overdue segment of the receivables bar takes `--crit`, the status colour, with a text label.
- **Never reuse a status colour for a series** (`--crit`, `--warn`, `--ok` mean state) and never colour text with a series colour.

**Form, by the data's job.**
| Data | Form | Why |
|---|---|---|
| Money in against money out, income against expense, sales against purchases, per bucket | **two-series columns, one axis** | two series, one scale, trend over a short ordered axis |
| Current against overdue (and ageing) | **one horizontal stacked bar** with direct labels | part to whole, two to four parts |
| Top expenses, customers, items | **ranked horizontal bars** (the existing `RankList` look, on the kit) | the donut is the wrong form for comparing close values and past about six slices; part-to-whole at a glance is served by the share % beside each bar |
| One current value with a trend | stat tile with an optional 12-point sparkline | a single value is not a chart |
| Never | dual-axis charts, a one-bar chart, a two-slice pie | the closing balance goes in the headline row, not on a second axis |

**Components** (new, `src/ui/dash/charts/`, plain SVG and HTML, about 600 lines in all **[J]**):
- `Columns` : categories on x, one to three series, grouped; y axis with round ticks (compact Indian units: `0`, `50 K`... via
  the existing `fmtCompactInr`, `L` and `Cr` from a lakh), hairline solid gridlines one step off the surface, zero baseline,
  columns at most 24 px with a 4 px rounded top and a square base, a 2 px surface gap between neighbours, **negative values
  drawn below the baseline** (a month of net returns, an overdrawn account), future buckets drawn as a tick with a muted
  label and no bar, an end-of-series value label only on the last non-empty bucket, no number on every bar.
- `StackBar` : one bar, 2 to 4 segments, 2 px surface gaps, a label inside a segment only when the measured text fits
  (otherwise outside the end or in the legend), amounts in a labelled legend row beneath. A zero segment is omitted.
- `RankBars` : label, bar, value, share %; the value at the bar tip; a row opens its drill target.
- `Sparkline` : 12 points, 2 px line, current point 8 px with a 2 px surface ring.
- `ChartFrame` : title, one-line "what and when" subtitle, the legend (always for two or more series; none for one), the
  **View as table** toggle, the empty and loading states, and the `role`/label wiring.
- A pure helper module `core/dashScale.ts` (nice ticks, band geometry, label-fit rule, compact INR) with its own assertion
  suite: ticks for all-zero, one bucket, negative and mixed ranges, a 366-bucket day window, and 0.01 values.

**Interaction.** A pointer move over a column group shows one tooltip listing every series at that bucket, value first and
name second (rows keyed by a short line of the series colour); keyboard focus shows the same. Hit areas are the whole
bucket band, not the painted column. Text is inserted with `textContent` (party and ledger names are user data). A refetch
(period change) holds the previous render at reduced opacity, with no skeleton and no layout jump.
**Accessibility** (3.11 repeats the checklist): each chart is `role="img"` with a one-sentence summary label; the **table twin**
is the accessible equivalent and the print and forced-colours fallback; marks are focusable in one tab stop (roving tabindex,
left and right between buckets, `Enter` drills, `Esc` leaves the chart); direct labels and the legend carry identity so colour
is never alone; the dark scheme uses its own validated steps, not an inversion.

### 3.7 The layout engine (keep the model, extend it)
**Keep:** the stored list of `{id, on, size}` in display order; three layers (personal, company, built-in); the 64-entry cap;
the rule that unknown saved ids are carried through and a widget the saved list lacks arrives off; `layoutOwner` keyed by
company and (with security on) operator. These are shipped, tested and correct. [C]

**Extend:**
1. **Per-widget size limits.** `WidgetDef.sizes` lists the sizes a widget offers (charts: M and L; `today` and `setup`: L;
   tiles: S, M, L; lists: S, M, L with row counts by size as `LIST_ROWS` does). The panel greys out sizes a widget does not
   offer; a stored size outside the list falls back to the widget's own (the existing tolerant reader).
2. **A single `span` table per widget** replaces the tile/card row-break rule: `S` 3 or 4, `M` 6, `L` 12 columns of 12, so
   natural pairs always fill a row (6 + 6, 4 + 4 + 4, 3 + 3 + 3 + 3). Document order is visual order: **no `grid-auto-flow:
   dense`**, because a reordered visual sequence would break Tab order.
3. **Edit mode on the board itself.** `F12` and the Customize button keep opening the docked panel (discoverability, and the
   place hidden widgets live, like Zoho's panel gallery). In edit mode the board also becomes direct: every widget gets a
   drag handle, a focus ring and a one-line toolbar (hide, size). **Pointer drag** reorders (pointer events, not the mouse-only
   HTML drag API); the drop slot is previewed; Esc cancels.
4. **Keyboard grab, move, resize** (the keyboard alternative to dragging, with live-region announcements, the same grammar as
   the panel):

| Key (a widget focused in edit mode) | Does |
|---|---|
| `Tab` / `Shift+Tab` | next / previous widget |
| `Space` | **grab** the widget (announced "Grabbed, position 4 of 12"); `Space` or `Enter` drops, `Esc` cancels and restores |
| `Left` / `Right` (grabbed) | earlier / later in the order |
| `Up` / `Down` (grabbed) | to the slot in the row above / below, by measured geometry |
| `Shift+Left` / `Shift+Right` | one size smaller / larger (stops at the ends, never wraps, as `stepSize` does) |
| `H` or `Delete` | hide (it moves to the panel's list; `Ctrl+Z` is the undo) |
| `S` `M` `L` | set the size directly |
| `R`, `C`, `X` | as today in the panel: reset, make this the company layout, remove it (each asks first) |

   Every move and resize is announced ("Receivables moved to position 2 of 12, size Medium"), as the panel's live region already does.
5. **Per-widget config** (`cfg`), a small sanitised object on a stored item: Top N, "include purchases" on Top expenses,
   the ledger list of the watchlist, the GST widget's GSTR-1 line. Schema `v: 2`. **A real cost:** an older build re-saving a
   layout rewrites items as `{id,on,size}` and would drop `cfg` (the shipped serialiser). Phase 3 therefore ships with the
   reader tolerating and *preserving* unknown item fields in the older code path first (a one-line change released ahead of
   the feature), or accepts the loss for config only.
6. **Who sets what.**
   - **Personal:** today per operator per device (`tw-dash-v1`, key `company|operator`). Honest limit, unchanged from
     `ROADMAP-ADDENDUM.md`: it does not follow the operator to another computer, because "a synced prefs mechanism that a
     non-admin can write does not exist". Phase 4 option, not v2.
   - **Company standard:** `Company.dashboard`, written by an administrator (the `C` key), as today.
   - **Role defaults (Phase 3):** a layer between personal and company, `Company.dashboardByRole?: Record<roleId, layout>`,
     administrator-written and synced; `pickLayout` stays pure with one more argument. It gives the clerk a Today-and-latest-
     vouchers home and the owner the full board (ux-analysis N8), and Zoho's role-limited dashboard is the precedent. [Z-notes]
     A new optional `types.ts` field must be carried by restore, sync and split: `master-resave.assert` classifies every key
     and fails on a new one, so it is listed in the work.
   - **Lock (Phase 3, optional):** an administrator can lock the company layout so operators cannot change theirs, the one
     thing Zoho's locked default has that ours lacks.
   - **Named boards (Phase 4, stretch):** tabs ("Overview", "Sales", "Inventory", "Compliance"), each its own layout; Zoho's
     custom dashboards with favourites are the precedent. [Z-custom]
7. **Where Zoho still beats us, said plainly:** multiple named, shareable dashboards (above), and report-built panels (any
   report as a chart). We deliberately do not build the second: the saved report views and the Reports Center cover it.

### 3.8 Drill-down everywhere
**Mechanism.** One helper, `drillTo(screen, { period?, filter?, basis? })`: (1) if a `period` is given, `st.setPeriod`; (2)
`st.push(screen)`; (3) the optional filter is delivered by a one-shot request the target report reads on mount (the same
pattern as `ui/modern/dashEditRequest.ts`). Step 3 needs Wave A's filter engine (`core/reportFilter.ts`, `useReportFilter`)
and is **not** in the MVP: until it lands, a drill opens the right report for the right window and the person sees the
overdue rows there; the card says so rather than promising a filter. Rights: a drill the operator may not open is not
drawn (the widget is judged by `screenDenial(ctx, drillTarget)`).

| From | To | Window set | Filter (when Wave A's engine is in) |
|---|---|---|---|
| Receivables headline / Current / Overdue | Receivables (`outstandings`) | none (as at) | overdue only for the Overdue segment |
| An ageing step | Ageing Analysis | none | that bucket |
| Payables headline / segments | Payables | none | as above |
| A bill row in Overdue invoices | that party's bills (`billsOfLedger`) | none | none |
| Cash and bank row | that ledger's vouchers | the window | none |
| "N not reconciled" | Reconciliation for that bank | the window | none |
| A Cash flow column | Cash Flow Summary | that bucket | none |
| An Income and expense column / the headline | Profit and Loss | that bucket / the window | none |
| A Top expenses row | that ledger's vouchers | the window | none |
| "All expenses" | Group Summary of the expense groups | the window | none |
| A Sales/Purchases trend column | the register | that bucket | none |
| A Top customers or items row | the party's or item's vouchers | the window | none |
| GST due | GSTR-3B (and GSTR-1 for its line) | the filing period | none |
| TDS to deposit | TDS Outstanding | none | none |
| A Stock alert row | that item's vouchers; "Reorder Status" | none | none |
| A voucher in Latest vouchers | the voucher, in alteration | none | none |
| Today strip, each figure | Day Book | today | the voucher type |

**Return.** `Esc` returns to the Dashboard (the screen is pushed, as today). Because a drill that sets a window changes the
**global** period (precedent: the Cash Flow report), returning shows the "Not the financial year: reset" chip with `R` to
restore, never a silent restore and never a silent leftover. A drill never changes the as-at switch.
**Every drillable thing is keyboard-reachable:** tiles, cards, rows and chart marks are real buttons or links in the Tab order
(G9), and `Enter` opens.

### 3.9 Data engine and performance
**The pure engine, `core/dashIndex.ts`** (no React, no Dexie, no store, so the perf harness can bundle and time it; the
harness refuses any engine that imports UI or store code).

```
buildDashIndex(inputs: { groups, ledgers, vouchers, window: {from, to}, asOn, today, bucketsFor(window) })
  -> DashIndex {
       classes: Uint8Array        // per ledger: cash | bank | sales | purchase | income | expense | debtor | creditor | duty
       open[l], asOn[l]           // signed balance before the window, and at as-at (opening balances included)
       dr[l], cr[l]               // window debit and credit
       bucketMove[class][b]       // per-bucket sums for the classes the charts need (income, expense, sales, purchase, cash in, cash out)
       cashNet                    // per-voucher net on cash and bank, split into in and out per bucket
       customers: Map<party, amt>, suppliers   // the existing partyOf rule, in-window
       bankUnreconciled[ledger]   // entries with no bank date at or before as-at
       today: { sales, received, paid, count }
     }
```
- **One pass over the vouchers**, applying the single optional/non-accounting guard (`postsToLedgers`) once per voucher, then a
  tight loop over entries with typed arrays and a ledger-index map. Everything the card needs afterwards is arithmetic over
  the index (`O(ledgers x buckets)`), so adding a widget does not add a pass.
- **Measured prototype** (scratch code, not shipped; the repo's synthetic book, 250,000 vouchers, 500 ledgers, 858,863 entries):
  a pass that fills a ledger-by-month debit/credit matrix plus the per-voucher cash net took **55.5 ms (p50)**, against 52.3 ms
  for `closingBalances` alone; and it reproduced the engines' sales figure exactly (751,204,931.96 against 751,204,931.96). [V]
  Day- and week-level buckets change the lookup key (a small map of at most 400 dates against a 12-entry month map), not the
  loop; I did not time that variant **[inferred]**.
- **Separate lazy passes, each memoised on the array identity of what it reads:** bills (`outstandingBills`, 153 ms), stock
  (`stockPositions`, 124 ms, which both Stock value and Stock alerts share), GSTR-3B for the filing period (69 ms). They are
  started only for a visible widget that needs them, in that order of value (Receivables first), each in its own task with
  a yield between, so the first paint carries the index-only widgets.
- **Budget at 250,000 vouchers, everything on:** index 55 + bills 153 + stock 124 + GST 69 = **about 400 ms** of engine time
  for a board that carries every widget in the catalogue, against about 1,060 ms for today's three groups of widgets
  (tiles, GST tile, opt-in cards), which carry fewer; none of it in one block longer than 160 ms. The sum is arithmetic
  over separately timed pieces, not a measurement of the finished board **[inferred]**. At 50,000
  vouchers all of it is under 80 ms. Old PCs: multiply (the harness's `--cpu-factor`). Rows `dash.index`, `dash.bills`,
  `dash.v2.all` are added to `scripts/perf` with reference lines, and `dash.v2.all` is proposed as a **gated** row once the
  numbers have settled (D8).
- **Gating, the cheap change that matters most:** `useIsTop()` already exists. A hidden (stacked) Dashboard does not
  recompute; it marks itself stale and recomputes when it returns to the top. This removes the "every save while stacked"
  cost (G13) even before the index.
- **Sync pulls and saves:** recompute is debounced to one per animation burst; the previous render stays on screen at reduced
  opacity meanwhile.
- **Later (Phase 4):** an incremental update (subtract the old voucher's contribution, add the new one's) makes a save O(entries
  of one voucher) instead of one 55 ms pass; the store replaces the vouchers array on each save (`sortVouchers([...])`,
  `store.ts:693`), so the hook needs the old and new voucher from the save path. A Web Worker is **not** proposed: the data
  lives in the main-thread store and copying 250,000 vouchers into a worker costs more than the pass.
- **The `plFigures` prerequisite.** Extract the Profit and Loss arithmetic (sales, direct and indirect incomes, purchases,
  direct and indirect expenses, opening and closing stock through `stockDelta`, gross, nett) from `ProfitLoss` and
  `RatioAnalysis` into `core/plFigures.ts`, taking a `moveOf(slug)` provider so it can run over `periodTotals` (the screens) or
  the index (the Dashboard). Both screens call it; the suites for both and `perf` rows for them guard the refactor. It
  touches `ui/reports.tsx` and `ui/reports/financial.tsx`, **both Wave A's files**, so it waits for Wave A's integration.

### 3.10 States
| State | What the person sees |
|---|---|
| **New company, no vouchers** | The Today strip reads "No vouchers yet". The Setup card is first. Receivables and Payables show "Nothing owed yet" and a `+ New` (Invoice, Bill) rather than a row of zeros; the charts show the frame with "Your first invoice will appear here" and no bars; Cash and bank lists the ledgers with their opening balances (an opening balance is data, not nothing). |
| **Data, none in this window** | The chart keeps its frame and says "No sales in 1-Oct to 31-Oct. Try Last financial year", with the preset one keypress away. |
| **Feature off** | Not drawn, not listed in Customize (the shipped rule). |
| **Right missing** | Not drawn, not listed, **not computed** (3.12). |
| **Computing** | The previous render at reduced opacity; first render a fixed-height placeholder (no layout jump). |
| **A widget throws** | A per-widget boundary: "Could not work this out. Retry." The board stays. The error goes to the console and, in the sample, nowhere else. |
| **All widgets hidden** | The shipped "Nothing on the dashboard" card, plus `R` for the standard layout (exists). |
| **Sample company** | Setup card hidden (the sample banner and Getting Started cover it); everything else as for a real company. |
| **Window in the future or `today` outside the year** | The as-at rule of 3.3; the header says "as at 31-Mar-26 (period end)" when it is not today. |

### 3.11 Accessibility checklist (each is a test or a probe)
- Every drillable thing is a `button` or link with a visible focus ring and an accessible name that includes the figure
  ("Receivables, 2,40,838 rupees, open the receivables report"); the tiles stop being `div` + `onClick` (G9).
- Each widget is a labelled region; the board has a heading order (h1 greeting, h2 per widget).
- Each chart has a summary label and a **table twin** reachable by a button; keyboard focus per bucket shows the same tooltip as
  hover; `Enter` drills; the tooltip text never carries a value that is not also in the table.
- Colour is never alone: the overdue segment has a text label and its amount; ageing steps are labelled; tags (Negative, Low)
  have words.
- Contrast: series colours at least 3:1 against the card in both schemes (validated); text in text tokens only.
- Forced colours and print: the table twin shows, marks use system colours, texture is available for print (45 and 135
  degrees only) as the dataviz rules say.
- Edit mode announces grab, move, size and drop through a live region; `Esc` always backs out one level.
- No motion beyond the existing 90 to 120 ms feedback; nothing animates data in.
- Phone width: the container queries already collapse the board at 860 and 460; the shell's 938 px minimum is the blocker and
  is a separate item (ux-analysis N11).

### 3.12 Rights and feature gating
Today the whole screen needs the `reports` right. New rule: **`SCREEN_AREA.dashboard = 'open'`**, and each widget carries the
`RightArea` of the figure it reveals plus the company feature it needs; `visibleItems` gains a rights argument beside the
features argument. A widget the operator cannot open is not drawn, not offered in Customize and not computed; with security
off everything is offered. The check that decides a drill is the same `screenDenial` the menus use, so a card never offers a
click that opens a refusal.

**This reverses a pinned decision, so it is flagged, not slipped in.** `scripts/assert/security-access.assert.ts` (section 5,
"screens by role") asserts that the Data Entry role **is blocked from `dashboard`** along with the P&L, Balance Sheet and Cash
Flow. Under the new rule that clerk opens the Dashboard and sees only the widgets whose area they hold: Today, Receivables,
Payables, Cash and bank, Overdue invoices and Latest vouchers (accounting), and nothing from Reports. Every one of those
figures is already one click away for that role (the Day Book, Cash/Bank Book and Outstandings are `accounting` screens they
may open), so no figure becomes newly visible; what changes is that the home page stops being a refusal. That assertion is
rewritten with the change, and the owner confirms the policy (D4).

| Widget | Area | Feature |
|---|---|---|
| Today, Receivables, Payables, Cash and bank, Overdue list, Latest vouchers, Sales and Purchases tiles, Top customers | accounting | billwise for the overdue split |
| Cash flow, Income and expense, Net profit, Top expenses, Watchlist | reports | none |
| Stock value, Stock alerts, Top items | inventory | inventory |
| GST due, TDS to deposit | gst | gst, tds |
| Setup | open | none |

Honest limit (as `core/access.ts` says of itself): this is a screen-sharing control in a client app, not a data boundary.
The Customize **company** layout still needs the Administration right through the company write guard, unchanged. The
change to `SCREEN_AREA` is one line plus the assertions that pin it (`security-access`, `dashboard-custom`, `zoho-scaffold`
and `sidenav` read the table).

---

## 4. MVP, phases, effort and build order

**Three builders on disjoint files** (the Wave A pattern), starting together after Wave A's integration:
- **D1, engine:** `core/dashIndex.ts`, `core/dashBuckets.ts`, `core/plFigures.ts` (and the two screens' edits), `core/dashDue.ts`
  (GST dates), `core/dashScale.ts`, perf rows, the engine assertion suites.
- **D2, chart kit:** `ui/dash/charts/*`, `styles/dashviz.css`, the `--viz-*` tokens in `styles.css` (a light-only block and
  a dark override, per the token rule), the chart recipes and the dataviz validator run recorded in the spec.
- **D3, board:** `Dashboard.tsx`, `ui/dash/widgets/*` (the split of `dashWidgets.tsx`), `core/dashboardLayout.ts` v2,
  `DashCustomise.tsx`, the period control, the drill helper, rights, the default layout, empty states, the assertion
  rewrites, `SCREEN_AREA.dashboard`.

| Phase | Delivers | Effort (builder-days) | Gate |
|---|---|---|---|
| **0. Foundations** | `dashIndex` plus buckets plus as-at, reconciled to the engines at 2k, 50k and 250k; `plFigures` extracted with P&L and Ratio Analysis calling it; `--viz-*` tokens; `useIsTop` gating of the existing Dashboard | **5 to 6** | tsc 0, build, `npm run assert`, `perf` rows, shots of the untouched screens byte-identical |
| **1. MVP: the Zoho-like home** | Period selector and `F2`, as-at switch, midnight timer, honest foot dates; the chart kit; Receivables and Payables cards (current, overdue, unapplied, ageing, `+ New`); Cash and bank list; Cash flow; Income and expense with Net profit; Top expenses; default layout v2; first-run and empty states; per-widget rights; the Dashboard button bar | **12 to 14** | as above plus the Dashboard recipes in light and dark, compact and comfortable; table twins and keyboard probes |
| **2. Complete the catalogue** | Overdue invoices, Today strip, Setup checklist, GST due with the cited dates module, TDS card, Stock alerts and Latest vouchers refinements, Trend and top lists on the kit, drill with filters (when Wave A's engine is in) | **7 to 9** | the reconciliation suite covers every widget |
| **3. Layout engine** | In-board edit mode with pointer drag, the keyboard grab/move/resize model, per-widget `cfg` and the Watchlist, role defaults, lock | **8 to 10** | edit-mode keyboard recipes; layout v2 reader and preserver tests |
| **4. Depth** | Incremental index update, rule-based Highlights, a cash-basis coverage spike (research only), a print summary, named boards (stretch), a synced personal layout (if a prefs mechanism exists by then) | **6 to 8** | each behind its own owner decision |

**MVP = Phase 0 plus Phase 1: about 17 to 20 builder-days, roughly two calendar weeks with three builders.** It delivers
everything Zoho's default home shows except the watchlist and the Projects panel (no equivalent), and it makes the existing
customiser worth using. Everything after is additive; nothing in Phases 2 to 4 changes a Phase 1 contract.

**Order to build the widgets** (value first, each reusing what the one before built):
1. Infrastructure: index, buckets, as-at, period control, chart frame.
2. **Receivables** (the owner's first question; builds the bills pass, the stacked bar and `+ New`).
3. **Payables** (the same code, mirrored).
4. **Cash and bank** (list; adds the unreconciled count).
5. **Cash flow** (adds Columns; adds the transfer rule and its report option).
6. **Income and expense** (adds `plFigures` use and the reconciling line).
7. **Top expenses** (RankBars; nearly free after the index).
8. **Overdue invoices** (reuses the bills pass).
9. **Today strip** and **Setup** (no new engine; first-run value).
10. **GST due** (the dates module and the filing-period build).
11. **Stock alerts**, **Latest vouchers**, **TDS** tile, then **Trend**, **Top customers/items** onto the kit.
12. Watchlist (needs `cfg`), Highlights (needs the whole index).

---

## 5. Test and acceptance plan
A new suite `scripts/assert/dashboard-v2.assert.ts` (and a rewrite of the pinned-default assertions in
`dashboard-custom.assert.ts`, deliberately, as `sidenav.assert` was rewritten for menu v2):
- **Index against engines:** at 2k, 50k and 250k synthetic vouchers and the sample company, the index's per-ledger closing
  balances equal `closingBalances`; window sums equal `periodTotals`; bucket sums equal `monthlyTrend`; party totals equal
  `rankParties`; cash net equals an independent per-voucher oracle; optional and non-posting vouchers never reach it.
- **Reconciliation per widget:** Receivables segments sum to the headline and the headline equals the debtors group balance;
  Income and expense headline equals the Profit and Loss screen's net profit on the same window (with stock, without,
  with a stock change, with a loss); Cash flow with a Contra between two banks shows zero for it; GST due equals a direct
  `buildGstr3b` of the filing period; Top expenses shares sum to 100% with "Other".
- **As-at and buckets:** the rule at every boundary (today before, inside and after the window; a closed year; the first and
  last day; leap day; a financial year across a calendar year; one-day and 31-day windows).
- **Layout v2:** `resolveLayout` keeps unknown ids and unknown item fields; a size outside a widget's list falls back; a
  hostile or oversized store never throws; the default has the 12 on-by-default ids in order; every widget has a drill target
  that exists in the union, the router and `SCREEN_AREA`.
- **Rights:** every widget's area and feature produce the same visible set as the menus for a clerk, an accountant, a
  stock keeper and a full-rights user; with security off everything shows.
- **Chart helpers:** `dashScale` ticks and label fitting at the edges listed in 3.6.
- **Harness recipes (`npm run shots`):** the default board in light and dark, compact and comfortable, at 1440 and 1100; the
  empty company; a restricted operator; the period list and presets; edit mode (pointer and keyboard); every chart's table
  twin and tooltip focus; zero console errors; the DOM probes for roles, names and tab order.
- **Performance:** `perf` rows added; at 250,000 vouchers the first paint's engine time is the index only.
- **Palette:** the validator output for both schemes is recorded next to the tokens, and a drift test fails if a `--viz-*` value
  changes without it.

---

## 6. Risks
| # | Risk | Mitigation |
|---|---|---|
| R1 | A widget disagrees with its report (the worst failure for an accounting product) | Principle 1; the reconciliation suite; `plFigures` shared by the screens and the board; no widget ships without its drill-and-reconcile test |
| R2 | Collision with Wave A (`reports.tsx`, `AppShell.tsx`, `router.tsx`, `access.ts`, `store.ts`, `quickCreate.ts` are modified or new in the working tree) | Phase 0 waits for Wave A's integration; D3's own files are `ui/modern/*` and `core/dash*`; the one `access.ts` line goes in last; no new `Screen` |
| R3 | Changing the built-in default alters what every un-customised user sees, and the shipped test pins the old default | It is the point (ux-analysis N4); saved layouts are untouched by the append-off rule; the assertion is rewritten deliberately and listed in the PR |
| R4 | The Cash Flow report's meaning changes (transfers excluded by default) | Decision D3; the leg view stays one F12 option away; the change is stated on the report |
| R5 | Statutory dates go stale or wrong (GST extensions, QRMP state category, the Income-tax Act 2025 TDS dates) | A cited, isolated module; the card says "statutory, check the portal for extensions"; TDS shows no date until its open item closes |
| R6 | Main-thread cost grows with a bigger default board | One index pass, lazy passes, gating on `useIsTop`, gated perf rows; the numbers in 2.3 and 3.9 are the baseline |
| R7 | Per-operator layouts do not follow a person across devices (local storage) | Stated in Customize ("kept on this device"); company standard and role defaults cover the common case; a synced mechanism is Phase 4 |
| R8 | Setting the global period from the Dashboard surprises a user later on a report | The "Not the financial year: reset" chip; the status bar already shows the period; precedent in the Cash Flow drill |
| R9 | SVG chart text and tooltips are easy to get subtly wrong (clipping, overflow, label collisions) | The label-fit rule, `dashScale` tests, and a screenshot review of every chart at the three sizes in both schemes (the validator checks colour, not layout) |
| R10 | The `reports`-to-`open` change reverses a pinned test (a Data Entry operator is blocked from the Dashboard today) | Each widget's own area, so no figure is newly visible; the honest limit is stated on the screen; `security-access.assert.ts` is rewritten with the change and the owner signs off (D4) |
| R11 | An older build re-saving a v2 layout drops `cfg` | Ship the tolerant-reader change ahead of Phase 3 |
| R12 | A dense default feels busier than Zoho's seven panels | 12 on by default of which three need a feature; "Hide" on Setup; every widget one checkbox from off; the owner reviews the default in the first screenshots |

---

## 7. Open decisions for the owner
| # | Decision | My recommendation |
|---|---|---|
| D1 | One global period (the open period, changed from the Dashboard) or a dashboard-local range | **Global**, so a tile equals its report; the reset chip covers the surprise |
| D2 | The default set: the 12 listed (including GST due and Stock alerts) | Yes; Quick Entry off because `+ New` and the universal `+` replace it |
| D3 | Cash flow counts own-account transfers or not, and whether the report changes too | **Exclude, and give the report the same option (default on)** |
| D4 | Dashboard open to every operator with per-widget rights (a Data Entry operator then sees the accounting widgets; today they are refused the whole screen) | Yes |
| D5 | Income and expense: accrual only, no toggle | Yes; the Cash flow card is the cash view |
| D6 | Ranked bars instead of a donut for Top expenses | Yes |
| D7 | GST card rule: "next return not yet due"; QRMP state category supplied by the owner | Yes |
| D8 | Gate a v2 performance row at 250,000 vouchers | After the numbers settle, and with the 100 ms reference line re-argued for a 250,000-voucher book |
| D9 | Personal layouts stay device-local in v2 | Yes; revisit with a synced prefs mechanism |

---

## 8. What this document did not do
- It did not open a live Zoho Books organisation; the owner's five screenshots are gone (zoho-books-reference.md, 0). Section
  1.7 lists what to confirm there.
- It did not research the Income-tax Act 2025 TDS deposit dates, the QRMP state categories, or the current GST extension
  notifications; they are named as the dependency of the cards that would show them.
- It did not run the shipped perf harness at a CPU factor or measure day or week buckets; both are labelled.
- It did not build or edit anything in the repo. The scratch benchmarks live in the session scratchpad.

---

## 9. Files the work would touch (so the order can avoid Wave A)
**Owned by this work (new or already the Dashboard's):** `src/ui/modern/Dashboard.tsx`, `dashWidgets.tsx` (split into
`src/ui/dash/widgets/*`), `DashCustomise.tsx`, `useDashLayout.ts`, `dashStore.ts`, `src/ui/dash/charts/*`,
`src/core/dashboardLayout.ts`, `dashboardData.ts`, new `core/dashIndex.ts`, `dashBuckets.ts`, `dashScale.ts`, `dashDue.ts`,
`plFigures.ts`, `src/styles/dashcustom.css` and a new `dashviz.css`, `scripts/assert/dashboard-custom.assert.ts` and a new
`dashboard-v2.assert.ts`, `scripts/perf/passes.ts`, `bench.ts`, `mirrors.mjs`, `run.mjs` (rows), `scripts/shots/recipes/dashboard-custom.mjs` and a new recipe.
**Shared with Wave A (edit last, one line or one function each):** `src/core/access.ts` (`dashboard: 'open'`),
`src/ui/reports.tsx` and `src/ui/reports/financial.tsx` (`plFigures` call sites), `src/ui/quickCreate.ts` (read only:
`pageCreates`), `src/styles.css` (the `--viz-*` tokens), the Wave A filter engine (read only until it lands).
**Not touched:** `router.tsx`, `store.ts`'s `Screen` union, `AppShell.tsx`, `MenuScreen.tsx`, `navTree.ts`.

---

## 10. Sources and evidence index
**Zoho (paraphrased; none reproduced).** Keys used above:
- **[Z-home]** Home (the Dashboard panels): https://www.zoho.com/us/books/help/home/
- **[Z-india]** India edition of the same page: https://zoho.com/in/books/help/dashboard
- **[Z-custom]** Custom dashboards (plans, panels, drag and resize, sharing, favourites): https://www.zoho.com/us/books/help/home/custom-dashboards.html
- **[Z-nav]** Navigating Zoho Books (regions of the page; no panel detail): https://www.zoho.com/us/books/help/getting-started/navigating-zoho-books.html
- **[Z-works]** How Zoho Books works (the module order; names the dashboard): https://www.zoho.com/us/books/help/getting-started/zoho-books.html
- **[Z-kb-overdue]** Payments shown as overdue in the dashboard (receivables come from the aging reports, as of the day viewed): https://www.zoho.com/qa/books/kb/reports/payments-included-as-overdue-in-dashboard.html . Outstanding receivables (invoice statuses counted; the mini dashboard on Invoices): https://www.zoho.com/us/books/kb/contacts/outstanding-receivables.html
- **[Z-notes]** Release notes (mini dashboard on invoices, September 2020; role-limited dashboard insights, 12 September 2019): https://www.zoho.com/us/books/release-notes.html
- **[Z-new]** What's new (the same notes, plus custom dashboards 1 July 2024, sortable report panels 19 June 2025, inventory panels 10 June 2026, outstanding payables in Bills 20 January 2023): https://www.zoho.com/us/books/whats-new.html
- **[Z-users]** Users and roles (Admin and Staff defaults): https://www.zoho.com/us/books/help/settings/users.html . **[Z-roles]** Create a custom role (no dashboard option described): https://www.zoho.com/us/books/kb/users-and-roles/create-custom-roles.html and https://www.zoho.com/us/books/kb/users-and-roles/custom-role.html
- **[Z-blog]** AI Summary dashboard ("Zia Summary"), Early Access, 17 December 2025, updated 17 April 2026: https://www.zoho.com/blog/general/zoho-finance-ai-features.html
- **[Z-mobile]** Mobile apps page: https://www.zoho.com/en-fr/books/accounting-mobile-apps/
- **[Z-invoice]** Sibling product's dashboard page (used only for the Top Expenses source and the Receivables split, labelled as Zoho Invoice): https://www.zoho.com/us/invoice/help//dashboard/
- Appeared in search results, **not opened** (so only as search summaries): https://www.zoho.com/us/books/help/accountant/chart-of-accounts.html and https://www.zoho.com/books/api/v3/chart-of-accounts/ (the watchlist box and the API's show-on-dashboard flag); https://www.zoho.com/books/help/dashboard/ (returned an access error).
- **GST due dates** (search summaries of the GST portal's own help pages, not opened; the portal says extensions by notification apply): https://tutorial.gst.gov.in/userguide/returns/GSTR3B.htm and https://tutorial.gst.gov.in/userguide/returns/FAQs_change_profile.htm

**Comsky (read in the working tree, commit 1a87871 plus uncommitted Wave A):** `CLAUDE.md`, `src/ui/modern/Dashboard.tsx`,
`dashWidgets.tsx`, `DashCustomise.tsx`, `useDashLayout.ts`, `dashStore.ts`, `src/core/dashboardLayout.ts`, `dashboardData.ts`,
`accounting.ts`, `billwise.ts`, `gstPeriod.ts`, `brs.ts`, `access.ts`, `src/ui/reports.tsx` (Profit and Loss),
`src/ui/reports/financial.tsx` (Cash Flow, Ratio Analysis), `src/ui/nav/QuickAccess.tsx`, `src/ui/onboarding/*`,
`src/ui/quickCreate.ts`, `src/ui/screenCtx.ts`, `src/state/store.ts`, `src/styles.css`, `src/styles/dashcustom.css`,
`scripts/assert/dashboard-custom.assert.ts`, `scripts/perf/*`, `scripts/shots/README.md`, `docs/research/ROADMAP-ADDENDUM.md`.
**Run:** `npm run shots -- --screens dashboard,dc-rich,dc-panel,dc-empty,dc-trend` (light and dark, 1440 and 1100); `node
scripts/perf/run.mjs --only=dash --sizes=50k,250k --runs=3 --no-stress` (Apple M4, 16 GB, load average 4.8); scratch
benchmarks in the session scratchpad over `scripts/perf/synth.ts` (one-pass index, per-engine timings, a gross-against-net cash
flow comparison by voucher base, GSTR-3B month against year); the `dataviz` skill's palette validator on the candidate series
and ageing palettes against the light (`#ffffff`) and dark (`#141c19`) card surfaces.
