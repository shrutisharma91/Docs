# Comsky Books: left menu spec ("Menu rail")

> **SUPERSEDED IN PART (2026-10-20): menu v2 replaces this spec's tree.** Phase 26 (`docs/design/zoho-ux-spec.md`, built and reviewed,
> not yet deployed) replaced the 14 groups and 108 rows of section 4 with **12 groups and 64 rows, a `+` on 26 of them and a click on a
> group label that opens that module's list**; the compiled tree is Appendix A of `zoho-ux-spec.md`, and its Appendix D lists, section by
> section, what of this document it replaces and what still stands. Everything in this file about the rail's two views, the 212px width,
> the `navhid` grid fix, the `Ctrl+\` chord, focus and key isolation, rights and gating, and the visual spec **still applies**. Read the
> counts below (14 groups, 108 rows, 19 keycaps, 8 footers) as the Phase 25 tree, which is what the live site still has. The rest of this file is kept as written.

> **Status:** design spec, ready for four implementation agents. **Author:** spec author (read-only pass: nothing under `src/`, `scripts/` or `electron/` was edited; this is the one file created). **Date:** 2026-10-19.
> **Scope:** the left rail (`.navrail`), the full-screen toggle in the breadcrumb, the one collapse chord, the `?` sheet rows for them, and the grid fix for the hidden state. **Frozen and untouched:** the Gateway tree and every letter in it, every shortcut, the F-key rail (152px, its metrics, its keys), field order, the Create / Alter routing, the dark scheme's existing look, printed paper.
> **Evidence rule:** every number below was measured in this session (headless Chrome through `scripts/shots`, the sample company, 1440x900) or computed from the real source. Where something is from memory (competitor products) it says so.

---

## 0. The decision

**Yes, we can, and the cost is small and bounded.** A persistent left module tree, in the vocabulary of Zoho Books, QuickBooks and Marg, sits **beside** the Gateway and the F-key rail; it never replaces either. Tally users keep every key; owners and accountants arriving from an online ERP get the navigation they already know, with the Tally key printed beside each row so the two vocabularies teach each other.

| What ships | Detail |
|---|---|
| **Two views in the rail** | A segmented control at the top: **Menu** (new, default for everyone) and **Quick access** (today's Today / Pinned / Recent, unchanged). Remembered per device. |
| **Menu view** | 14 priority-ordered groups, **108 rows**, every row a verified real screen (115 distinct destinations pushed through the live store: 115 render, 0 console errors; one placeholder screen found and left out). Collapsible groups, current row highlighted and its group opened, a **Filter menu** box, a **Pinned** group on top, **Saved views** under Reports, a quiet **footer link** per group into the matching Gateway submenu. |
| **Full-screen view** | A toggle at the **start of the breadcrumb** ("Hide menu: full screen" / "Show menu"), the existing top-bar hamburger, and **one chord: `Ctrl+\` (`⌘\` on a Mac)**. Hidden: the screen and the F-key rail use the whole width on every screen. |
| **Bug fixed at its root** | `.app.navhid` used `display: none` on the rail, which removes it from the grid and re-places every sibling one column left: the screen fell into the 0px column and the F-key rail stretched across 1288px at the far left (reproduced, screenshot, section 1). The rail now stays in the grid and its column collapses. |
| **Width** | The rail is **212px in both views**, hidden is **0** (the first draft kept Quick access at 186px; that re-flowed the open screen on every view switch: Appendix D, 2). The 26px is paid for by the full-screen toggle; the arithmetic is in section 10.2. |

**What does not change:** no Gateway letter, no F-key, no menu placement, no field order; no global key is claimed except the one chord; the rail registers **no** global handler (it handles its own keys and stops them from reaching the screen's key stack); nothing is toggled on a global `useKeys` handler.

**Documents to update when this lands** (a docs agent owns them; this spec edits none): `CLAUDE.md` (status paragraph, the four GOTCHAs in section 14.3), `docs/HANDOFF.md`, `docs/design/light-theme-spec.md` (section 0 table row "Grid `186px 1fr 152px`", section 7.2, checklist 10.3: they must now read "186px in Quick access, 212px in Menu, 0 when hidden"), `docs/parity-data.json` `beyond` (a "Module menu" row; **not** parity).

---

## 1. Evidence

1. **Read:** `Sidebar.tsx`, `AppShell.tsx`, `widgets.tsx` (TopBar, StatusBar), `sidebar.css`, `styles.css` (tokens, `.app`, `.main`, `.navrail`, `.navhid`, `.btnpanel`), `styles/light/shell.css`, `docs/design/light-theme-spec.md`, `state/store.ts` (the `Screen` union, `push`, `goHome`), `ui/router.tsx` (every `case`, `CRUMBS`), `MenuScreen.tsx` (every `MENUS` entry, `hk`, `feature`, `offered()`), `modern/CommandPalette.tsx` (command list, `DOMAIN_ALIASES`), `overlays.tsx` (key sheet data, `GLOBAL_KEYS`, `KeyHelpHost`, `overlayUp`), `core/access.ts` (`SCREEN_AREA`, `screenDenial`), `state/recents.ts` (`screenKey`, `isRecordable`, `PINS_MAX = 8`), `keys.ts` (the LIFO stack, `combo`, `useFallbackKeys`, `BROWSER_RESERVED`), `favourites.tsx` / `favouritesCore.ts` (the host pattern and `LOCKED_KEYS`), `scripts/assert/visual.assert.ts` + `visual.baseline.json`, `scripts/shots/*`, the Marg Books screenshot.
2. **The current rail** is Today / Pinned / Recent only; its header comment explains it was slimmed because it used to be a second copy of the Gateway tree. The Gateway reaches `Sales Register` in three keystrokes after Home (`D`, `A`, `S`), `GSTR-2B Reconciliation` in three (`D`, `U`, `2`): a Tally operator's hands know this, an owner from Zoho does not, and neither can see the whole product at once.
3. **Reproduced the `navhid` bug** (`TW_NAV_COLLAPSED=1`, light, compact): `.main` columns `0px 1288px 152px`; the screen's `.content` is **0px wide at x=0**; `.btnpanel` is **1288px wide at x=0**. Cause: `.app.navhid .navrail { display: none }` removes the rail from the grid, auto-placement then puts the screen in column 1 (0px) and the F-key rail in column 2 (`1fr`). Present on every screen with a `.btnpanel`.
4. **Validated the fix** with the CSS in section 8 injected live: content **1288px at x=0**, F-key rail **152px at x=1288**, on Gateway, Day Book, Sales voucher (with its ledger popup open), Trial Balance, GSTR-1; the Dashboard (a single `.dash` child, no F-key rail, as in the shown state) 1288px at x=0. No horizontal page scroll. In dark: 0px / 1288px / 152px, content 1288px at x=0.
5. **Verified every route.** All 116 destinations of the draft tree (item `to` plus `+` create forms) were pushed with `goHome(); push(screen)` through the real Zustand store (a scratch CDP script built on `scripts/shots`' `launchChrome` / `Page` / `openSample`, the store module imported by its served URL) in the sample company: **115 render with no console error; 1 does not**: `batchSummary` renders `ComingSoon` ("This screen is being built…", `ui/inventory/BatchSummary.tsx`), the only stub in the product (`grep ComingSoon`). It is **left out** (section 4.5). The seven Gateway submenus used by the footer links were pushed too: Account Books (11 rows), Inventory Books (12), Statutory Reports (25), Display More Reports (16), Create (19), Payroll Reports (9), Company (24).
6. **Prototyped the design** as injected DOM + the CSS in Appendix A, in the live page: light and dark, compact and comfortable, Quick access with the segmented control, hidden state with the breadcrumb toggle, filter results (`gst`, `receipt`, no match). So the pictures in this spec were seen, not imagined.
7. **Measured label widths** (IBM Plex Sans 400, 13px, canvas `measureText` in the app's page after its fonts loaded): widest row label **142px** ("Reverse Charge Register"), widest keyed row 142px (Physical Stock + the `Ctrl+F7` cap). With the metrics of section 10.2 the room per row is `--nv-w` minus 68: a **186px rail leaves 118px and would truncate 26 of the 108 rows**, 202px would still truncate 9, **212px leaves 144px and truncates none**. The 68 includes the classic Windows scrollbar, drawn at 11px in this app the moment the tree scrolls. Probe at 212px: all 14 groups open, Windows key spellings, 11px scrollbar present, compact and comfortable: **0 truncated labels** (at 208px with looser paddings the same probe still caught 9, which is how the paddings were tightened).
8. **Contrast** (computed from the real tokens): ink on `--panel` 16.4 / 14.9 (light / dark); accent on `--sel` **5.00 / 7.03**; `--dim` on `--panel` 5.84 / 6.71; `--dim` on `--sel` 5.03 / **4.69**; `--dim` on `--hover` 5.29 / 5.68; accent on accent-soft-over-panel 5.10 / 7.63. All >= 4.5. `--border` on `--panel` is only 1.41 / 1.63, which is why the filter box is ringed with `--field-line` (3.27 / 3.9), exactly as the report filter is.
9. **Proved the key isolation** the design relies on: a focusable element inside the React root whose root-level `keydown` handler calls `stopPropagation()` for ArrowDown / letters keeps the Gateway's cursor on "Create" through two ArrowDowns while `F9` (not consumed) still reaches the stack and opens Purchase. React 18 delegates at the root container, which sits below `window`, where `installKeyboard` listens; a React `onKeyDown` + `e.stopPropagation()` therefore has exactly this effect.
10. **Compiled the reference module.** `src/ui/navTree.ts` and `src/core/navPrefs.ts` from sections 6.1 / 6.2, copied into a scratch copy of the repo with the six new icon names added, pass `tsc --noEmit` with **0 errors** (every `to`, `create` and `also` is checked against the real `Screen` union, so a misspelt route cannot compile) and **84 behavioural assertions** (gating, filter, active row, persistence, chord).

---

## 2. Evaluation

### 2.1 Does a left menu bar suit business owners and people leaving Zoho Books and other online ERPs?
Yes, for three reasons.
* **It is the idiom they already use.** From memory (not re-checked this session): Zoho Books keeps a persistent left module tree (Dashboard, Items, Banking, Sales, Purchases, Accountant, Reports, ...) whose groups open in place and which collapses from a hamburger; QuickBooks Online has the same shape (Dashboard, Banking, Sales, Expenses, Reports, Taxes, Accounting). The Marg Books screenshot the owner shared shows it for an Indian ERP: a search box on top, icon + label rows with chevrons, a group opened in place with its children indented, a hamburger in the header, and a "Shortcut Keys" panel at the right. The Gateway is a **Tally** idiom: a centred list that replaces the screen, three levels deep, driven by letters. Someone who has never used it cannot see what the product contains.
* **Owners think in jobs and documents, not in voucher types.** "Payment Received", "Bills", "Receivables" are how they say it; Tally says "Receipt", "Purchase", "Outstandings". The rail uses the job vocabulary, prints the Tally key on the row, and carries the Tally and Zoho names as invisible search aliases (section 4.6), so a user typing either finds the row.
* **Everything it needs already exists.** 115 destinations, no new screen, no new route, no new engine.

### 2.2 What it costs, and why that is acceptable
* **Width:** 26px more than the old 186px sidebar (212, in both views). The full-screen toggle is the answer: a user doing a long voucher or a wide report takes the width back with one click or one chord, and it persists.
* **A second path to each screen** (the reason the rail was slimmed in the first place). Mitigations: it is **additive and optional** (Quick access, the Gateway, Go To and every key are unchanged); each keyed row **teaches the Gateway key** instead of competing with it; and the footer links go **into** the Gateway's own submenus, so the rail leads toward the Tally tree instead of away from it.
* **Maintenance:** one typed table (`navTree.ts`) with a drift-guard suite that fails when a route, a feature flag, a key or a right drifts from what the Gateway says (section 11).

### 2.3 Why we do not copy Marg's dark navy panel
We take Marg's **structure** (search on top, icon + label + chevron, children indented) and leave its **paint**. Reasons:
1. **Tokens.** Everything here is `--panel`, `--sel`, `--ink`, `--accent`: two schemes and two densities for free. A navy slab would need its own tokens in both schemes and an exception in the contrast table.
2. **Calm canvas.** The light scheme's rule is "chrome recedes, the work is the brightest thing; teal means intent". A saturated dark rail beside a white page is the highest-contrast object on screen and pulls the eye from the data. The rail is a white raised card on the canvas, like the screen card and the Gateway cards.
3. **Identity.** Our trade dress is teal on neutral (`docs/LEGAL-BRIEFING.md`); a competitor's colour block is the wrong thing to echo.
4. **Dark scheme.** In dark our rail is already a dark `--panel` card on the dark chrome; nothing of Marg's paint would add to it.

### 2.4 What a Tally user gets
Nothing is taken away. They gain: all of the product in one scrollable list; the key printed beside the 19 rows that have one (`F8` on Sales Invoice, `Alt+F6` on Credit Note, `K` on Day Book, ...); type-to-filter that matches their own words (`receipt`, `contra`, `brs`, `r1`); and Quick access exactly as it was. The default view is Menu for everyone (owner's decision); a Tally purist is one click from Quick access and the choice sticks.

---

## 3. Decisions: the owner's eight, and where this spec changes one

| # | Decision | How it is implemented | Change, and why |
|---|---|---|---|
| 1 | Two views behind a segmented control; Menu default; Pinned group at top of Menu | `Menu` / `Quick access`, `TW_NAV_VIEW` (absent = menu). Pinned group = `mem.pins` filtered by rights, max `PINS_MAX` (8), shown first, open by default. | none |
| 2 | Module tree, icon + label, chevron, several groups open, current row highlighted and its group opened, state remembered, keycaps | Groups carry the icon and chevron; **item rows carry no icon** (108 icons would be noise; the keycap is the right-hand cue). Default open: Pinned only (every module folded, so the first view is the whole product in 14 rows; see Appendix D, 3). Opening for the current screen is session-only (section 7.2). | Icons on group rows only: the reading of "icon + label rows" that keeps 13px labels legible in 212px. |
| 3 | Filter box: substring over label + aliases, auto-expand, Enter opens first match, Esc clears and returns focus, no global key | As decided. Also matches the **group's name** ("inventory" lists the module), every typed word must match (AND), Pinned is filtered too, footer rows are never results. Reached by click or Tab; Ctrl+K / Alt+G stays the global search. | Two additions, both reversible: group-name match, and word-AND (so "sales reg" finds Sales Register). |
| 4 | Proper keyboard tree, keys act only inside the rail, nothing leaks to or is stolen from the key stack | Roving tabindex; the rail handles its own keydown and stops propagation of exactly the keys it consumes (section 7.3). | Adds `+` (new), `Space` (pin), type-to-filter, PgUp/PgDn. Reason: a non-modal focusable region must consume every key it uses **and** its stray letters, or the Gateway's single-letter navigation and `Enter` run behind it. |
| 5 | Hide what the operator cannot open and what the company has switched off | `screenDenial` + `feature` flags exactly as `MenuScreen.offered()`; empty groups vanish. | Footer links shown only to full-rights operators or with security off (they could otherwise open an empty submenu). |
| 6 | Full-screen: breadcrumb toggle + hamburger + one chord, persistent, grid valid | Toggle at the start of the breadcrumb; chord **`Ctrl+\`** (audit in section 5); `TW_NAV_COLLAPSED`; the grid fix in section 8. | **Breadcrumb segments are not clickable.** A segment click would pop several screens at once and bypass each screen's own Esc guard ("Quit? entries will be lost" on a half-typed voucher). Esc semantics are untouched. |
| 7 | Light and dark, compact and comfortable, tokens only, white raised card, 13px labels, 16px stroke icons, stronger group headers, no tracked caps, compact-compatible rows | Appendix A. Labels `--nv-fs: 13px` in both densities; icons 16px; six new stroke icons; rows 24px compact (follows `--row-y`). | **Width 212px, in BOTH views** (measured, section 1.7 and 10.2; Quick access used to stay 186px, which re-flowed the open screen by 26px on every view switch: Appendix D, 2). **Active row keeps a 2px accent bar** (the light spec removed the bar for pills on the canvas; here the row sits in a white card where the `--sel` wash alone is 1.12:1, so wash + accent label + bar carry it). |
| 8 | Priority order of 13 groups | Section 4. | **14 groups**: GST & Taxes is split into **GST** and **TDS & TCS** (a single group would be 18 rows against a 4-9 rule, and the TDS / TCS half is hidden as a unit by two company features). **Contra moves** from Accounting to Banking & Cash ("Transfer (Contra)": cash deposits and withdrawals are banking). **Receivables & Ageing** becomes two rows. **Employees** lives in Payroll & HR, not Masters. **Custom Fields** moves to Data & Utilities, **GST Rate Setup** and **Switch Company** go to Settings. Added: footer links, a dynamic **Saved views** group. Dropped: Batches & Expiry (stub), API Reference, Split Company Data (Gateway only). All in 4.5. |

---
## 4. The menu: groups, rows, order

Order inside the rail, top to bottom: **Pinned** (personal, dynamic) then the 14 groups below, with **Saved views** (dynamic) between Reports and Masters. A group with no row left for this company and operator is not drawn. "Rows" counts the real destinations; the footer link is extra and never counts toward the 4 to 9 rule.

### 4.1 Group order and why

| Rank | Group | Icon | Rows | Company features that can hide rows | Why it sits here |
|---|---|---|---|---|---|
| 1 | Dashboard | `grid` | 1 (a row, no folder) | none | The first thing an owner opens each morning; one row, no folder. |
| 2 | Sales | `invoice` | 9 | `billwise`, `orders`, `inventory` | Money in: the most-entered and most-checked work of a trading business. |
| 3 | Purchases | `bag` | 7 | `billwise`, `orders`, `inventory` | Money out: bills, payments and what is owed to suppliers. |
| 4 | Banking & Cash | `bank` | 9 | none | Where the cash is, and whether the bank agrees. |
| 5 | Accounting | `register` | 8 | `costCentres` | The books themselves: day-to-day checks first, year-end tools last. |
| 6 | Inventory | `box` | 8 | `inventory` | Stock on hand and stock movements, for businesses that hold stock. |
| 7 | GST | `percent` | 9 | `gst` | Monthly filing, in the order the month unfolds. |
| 8 | TDS & TCS | `doc` | 9 | `tds`, `tcs` | Deduction and collection at source; hidden when the company does not use them. |
| 9 | Reports | `trend` | 8 | none | The statements an owner or banker asks for. |
| 10 | Masters | `tree` | 9 | `inventory`, `costCentres` | Create and alter the lists everything else points at; set up once, touched rarely. |
| 11 | Payroll & HR | `people` | 9 | `payroll` | The monthly pay cycle in the order it is run. |
| 12 | CRM | `funnel` | 4 | none | Sales follow-up; the four screens of the pipeline, daily action first. |
| 13 | Data & Utilities | `plug` | 9 | none | Moving data in and out, and keeping it safe; used in bursts. |
| 14 | Settings | `gear` | 9 | `gst` | Configuration, last: an owner visits it a few times a year. |

Ordering rule used throughout: **daily money movement first** (Sales, Purchases, Banking), then **the books and stock behind it**, then **compliance on its calendar** (GST, TDS & TCS), then **statements**, then **set-up-once data**, then **monthly cycles and tools**, then **settings**. Reports sit below the compliance groups because the Dashboard already carries the daily numbers and period statements are consulted monthly; GST and TDS have deadlines. Within a group: entering things first, reading things next, rarely-run things last.

### 4.2 The final table: menus and submenus, in priority order

Columns: **Rank** is the group's position; **Opens** is exactly the `Screen` the row pushes (the same one the Gateway or Go To pushes; the rail pushes it with `go()`, which is the Favourite shortcuts' `planNav` policy: `goHome()` then `push()`, except that when an entry form is anywhere in the stack it pushes ON TOP, as Go To does; section 7.1); `+ x` is the create form behind a master row's **+** button; **Key** is the Tally key printed on the row: **(menus)** fires on the Gateway and every submenu (`MenuScreen` binds the ten voucher F-keys; they do nothing on a report or a list, and the tooltip says "on the Gateway and its menus"), **(Gateway)** only on the Gateway, **(anywhere)** on every screen, and the tooltip says which, so a keycap never over-promises.

| Rank | Group | Row | Opens | Key | Why it ranks there |
|---|---|---|---|---|---|
| 1 | Dashboard | Dashboard | `dashboard` | `W` (Gateway) | Single-row group: cash, receivables, sales at a glance. |
| 2 | Sales | Sales Invoice | `voucher` base=Sales | `F8` (menus) | Entered many times a day; the F8 voucher. |
| 2 | Sales | Payment Received | `voucher` base=Receipt | `F6` (menus) | Every collection is entered; second-most frequent entry. |
| 2 | Sales | Receivables | `outstandings` side=receivables<br>feature `billwise` |  | The owner's daily "who owes me" check. |
| 2 | Sales | Sales Register | `register` base=Sales |  | The list of invoices; the way back to one for alteration. |
| 2 | Sales | Sales Order | `voucher` base=SalesOrder<br>feature `orders` |  | Used by order-driven businesses only; behind the Orders feature. |
| 2 | Sales | Credit Note | `voucher` base=CreditNote | `Alt+F6` (menus) | Occasional: returns and rate corrections. |
| 2 | Sales | Delivery Note | `voucher` base=DeliveryNote<br>feature `inventory` |  | Only when goods ship before they are billed. |
| 2 | Sales | Ageing Analysis | `ageing`<br>feature `billwise` |  | Weekly collections review. |
| 2 | Sales | Reminder Letters | `reminders`<br>feature `billwise` |  | Monthly, and only for the overdue few. |
| 3 | Purchases | Purchase Invoice | `voucher` base=Purchase | `F9` (menus) | The F9 voucher; entered as stock and bills arrive. |
| 3 | Purchases | Payment Made | `voucher` base=Payment | `F5` (menus) | Every payment and expense is entered here. |
| 3 | Purchases | Payables | `outstandings` side=payables<br>feature `billwise` |  | Weekly "what do I owe" before paying. |
| 3 | Purchases | Purchase Register | `register` base=Purchase |  | The list of bills; the way back to one for alteration. |
| 3 | Purchases | Purchase Order | `voucher` base=PurchaseOrder<br>feature `orders` |  | Order-driven businesses only; behind the Orders feature. |
| 3 | Purchases | Debit Note | `voucher` base=DebitNote | `Alt+F5` (menus) | Occasional: returns to suppliers. |
| 3 | Purchases | Receipt Note | `voucher` base=ReceiptNote<br>feature `inventory` |  | Only when goods arrive before the bill. |
| 4 | Banking & Cash | Cash/Bank Book | `cashBankBook` |  | Checked daily: "how much cash and bank do I have". |
| 4 | Banking & Cash | Bank Reconciliation | `brs` |  | Weekly or monthly close of every bank account. |
| 4 | Banking & Cash | Import Bank Statement | `brsImport` |  | Monthly, right before reconciling; has its own screen. |
| 4 | Banking & Cash | Transfer (Contra) | `voucher` base=Contra | `F4` (menus) | Cash deposits and withdrawals: banking activity, not an adjustment. |
| 4 | Banking & Cash | Cheque Printing | `cheque` |  | Used whenever a supplier is paid by cheque. |
| 4 | Banking & Cash | Post-Dated Cheques | `pdcRegister` |  | Weekly look at cheques due to clear. |
| 4 | Banking & Cash | Cheque Books | `chequeBooks` |  | Set up when a new book is issued. |
| 4 | Banking & Cash | Payment Advice | `paymentAdvice` |  | Sent to a supplier after a payment run. |
| 4 | Banking & Cash | Deposit Slip | `depositSlip` |  | Printed when cash or cheques go to the bank. |
| 5 | Accounting | Day Book | `daybook` | `K` (Gateway) | Every voucher of the day; where corrections start. |
| 5 | Accounting | Journal | `voucher` base=Journal | `F7` (menus) | The accountant's adjustment entry (F7). |
| 5 | Accounting | Trial Balance | `trialBalance` |  | The accountant's first check of the books. |
| 5 | Accounting | Ledger | `pickMaster` kind=ledger goal=book |  | Statement of one party or account; asked for constantly. |
| 5 | Accounting | Group Summary | `pickMaster` kind=group goal=book |  | Totals of a whole group such as expenses or debtors. |
| 5 | Accounting | Cost Centre Analysis | `costAnalysis`<br>feature `costCentres` |  | Only for businesses that track by project or branch. |
| 5 | Accounting | Budgets | `budgets` |  | Set once a year. |
| 5 | Accounting | Interest Calculation | `interest` |  | Occasional; mostly year-end. |
| 6 | Inventory | Stock Summary | `stockSummary`<br>feature `inventory` | `S` (Gateway) | The "what do I have" report, opened daily. |
| 6 | Inventory | Stock Items | `pickMaster` kind=stockItem<br>+ `stockItem`; feature `inventory` |  | The item list, as a Zoho user expects to find it under Inventory. |
| 6 | Inventory | Reorder Status | `reorderStatus`<br>feature `inventory` |  | Tells the owner what to buy this week. |
| 6 | Inventory | Stock Journal | `voucher` base=StockJournal<br>feature `inventory` | `Alt+F7` (menus) | Routine transfers between godowns. |
| 6 | Inventory | Physical Stock | `voucher` base=PhysicalStock<br>feature `inventory` | `Ctrl+F7` (menus) | Periodic counts; month or year-end. |
| 6 | Inventory | Godown Summary | `godownSummary`<br>feature `inventory` |  | Matters only with more than one godown. |
| 6 | Inventory | Manufacturing Journal | `manufacture`<br>feature `inventory` |  | Manufacturers only. |
| 6 | Inventory | Stock Ageing | `stockAgeing`<br>feature `inventory` |  | A periodic clean-up report. |
| 7 | GST | GSTR-1 | `gstr1`<br>feature `gst` |  | Filed monthly or quarterly. |
| 7 | GST | GSTR-3B | `gstr3b`<br>feature `gst` |  | Filed monthly, with the tax payment. |
| 7 | GST | GSTR-2B Reconciliation | `gstr2b`<br>feature `gst` |  | Monthly, before 3B: what input credit is claimable. |
| 7 | GST | e-Invoice / e-Way Bill | `einvoice`<br>feature `gst` |  | Daily for businesses above the e-invoice threshold. |
| 7 | GST | HSN Summary | `hsnSummary`<br>feature `gst` |  | Required inside GSTR-1. |
| 7 | GST | GST Validation | `gstValidation`<br>feature `gst` |  | Run before filing; lists what the return would reject. |
| 7 | GST | GST Reconciliation | `gstRecon`<br>feature `gst` |  | Monthly tie-out of 3B, GSTR-1 and the books. |
| 7 | GST | Reverse Charge Register | `rcmRegister`<br>feature `gst` |  | Needed when unregistered or specified supplies are bought. |
| 7 | GST | GSTR-9 Annual Return | `gstr9`<br>feature `gst` |  | Once a year, but never skipped. |
| 8 | TDS & TCS | TDS Outstanding | `tdsReport`<br>feature `tds` |  | What has been deducted and not yet deposited. |
| 8 | TDS & TCS | TDS Challans & Returns | `tdsReturns`<br>feature `tds` |  | Monthly deposit and quarterly return. |
| 8 | TDS & TCS | TDS Quarterly Tracker | `tdsQuarterly`<br>feature `tds` |  | Quarter-by-quarter due dates. |
| 8 | TDS & TCS | TDS Deductee-wise | `tdsDeductee`<br>feature `tds` |  | Party-wise detail behind the return. |
| 8 | TDS & TCS | TDS Challan Recon | `tdsChallanRecon`<br>feature `tds` |  | Matches deductions to deposits. |
| 8 | TDS & TCS | Form 16A Certificates | `form16a`<br>feature `tds` |  | Issued each quarter after filing. |
| 8 | TDS & TCS | TDS Interest & Late Fee | `tdsInterest`<br>feature `tds` |  | Only when something was late. |
| 8 | TDS & TCS | TCS Outstanding | `tcsReport`<br>feature `tcs` |  | Collected on specified sales; fewer businesses. |
| 8 | TDS & TCS | TCS 206C(1H) Turnover | `tcs206c`<br>feature `tcs` |  | Threshold tracker for TCS on sales. |
| 9 | Reports | Balance Sheet | `balanceSheet` | `B` (Gateway) | The most-requested statement after P&L. |
| 9 | Reports | Profit & Loss | `profitLoss` | `P` (Gateway) | The owner's "am I making money" report. |
| 9 | Reports | Cash Flow | `cashFlow` |  | Asked for by banks and investors. |
| 9 | Reports | Ratio Analysis | `ratioAnalysis` | `R` (Gateway) | Quick health check; used monthly. |
| 9 | Reports | Funds Flow | `fundsFlow` |  | Accountant and lender report. |
| 9 | Reports | Top-N Analysis (MIS) | `topMis` |  | Management view: best customers, items, suppliers. |
| 9 | Reports | Profitability Analysis | `profitability` |  | Margin by item or party. |
| 9 | Reports | Budget vs Actual | `budgetVsActual` |  | Only where budgets are set. |
| 10 | Masters | Ledgers | `pickMaster` kind=ledger<br>+ `ledger` |  | Customers, suppliers and accounts: the master touched most. |
| 10 | Masters | Chart of Accounts | `coa` | `H` (Gateway) | The first screen an accountant from any package looks for. |
| 10 | Masters | Groups | `pickMaster` kind=group<br>+ `group` |  | Changed when the chart is restructured. |
| 10 | Masters | Stock Items | `pickMaster` kind=stockItem<br>+ `stockItem`; feature `inventory` |  | Tally users look here; the same screen as under Inventory. |
| 10 | Masters | Voucher Types | `pickMaster` kind=voucherType<br>+ `voucherType` |  | Numbering and series; set up once. |
| 10 | Masters | Units | `pickMaster` kind=unit<br>+ `unit`; feature `inventory` |  | A short list, set up once. |
| 10 | Masters | Godowns | `pickMaster` kind=godown<br>+ `godown`; feature `inventory` |  | A handful per business. |
| 10 | Masters | Cost Centres | `pickMaster` kind=costCentre<br>+ `costCentre`; feature `costCentres` |  | Only for businesses that track by project or branch. |
| 10 | Masters | Price Levels | `priceLevels`<br>feature `inventory` |  | Optional price lists. |
| 11 | Payroll & HR | Attendance | `attendance`<br>feature `payroll` |  | Step 1 of the month. |
| 11 | Payroll & HR | Payroll Voucher | `payrollVch`<br>feature `payroll` |  | Step 2: processes the month. |
| 11 | Payroll & HR | Pay Sheet | `paySheet`<br>feature `payroll` |  | Step 3: the month's summary. |
| 11 | Payroll & HR | Payslip | `payslip`<br>feature `payroll` |  | Step 4: handed to each employee. |
| 11 | Payroll & HR | Salary Disbursement | `salaryDisbursement`<br>feature `payroll` |  | Step 5: the payment. |
| 11 | Payroll & HR | PF / ESI / PT | `payrollStatutory`<br>feature `payroll` |  | Monthly deposits. |
| 11 | Payroll & HR | Leave | `leaveApply`<br>feature `payroll` |  | Entered as it happens. |
| 11 | Payroll & HR | Leave Balance | `leaveBalance`<br>feature `payroll` |  | Looked up on request. |
| 11 | Payroll & HR | Employees | `pickMaster` kind=employee<br>+ `employee`; feature `payroll` |  | Master data; changes a few times a year. |
| 12 | CRM | Follow-ups Due | `followUps` |  | The daily to-do list of a sales team. |
| 12 | CRM | Leads | `leads` |  | The working list. |
| 12 | CRM | New Lead | `lead` |  | Entered as enquiries arrive. |
| 12 | CRM | Pipeline Summary | `pipeline` |  | The weekly review. |
| 13 | Data & Utilities | Backup / Restore | `backupRestore` |  | The one chore an owner must not skip. |
| 13 | Data & Utilities | Bulk Entry | `bulkEntry` |  | Daily for data-entry teams. |
| 13 | Data & Utilities | Import Data | `importData` |  | Used heavily in the first week, then rarely. |
| 13 | Data & Utilities | Export Data | `exportXml` |  | Monthly hand-off to the accountant. |
| 13 | Data & Utilities | Data Check | `dataCheck` |  | Run when something looks wrong. |
| 13 | Data & Utilities | Custom Fields | `customFields` |  | Set up once. |
| 13 | Data & Utilities | API Keys | `apiKeys` |  | Set up by whoever integrates. |
| 13 | Data & Utilities | Webhooks | `webhooks` |  | Set up by whoever integrates. |
| 13 | Data & Utilities | Compare with Tally Data | `tallyCompare` |  | A one-time migration check. |
| 14 | Settings | Company Details | `companyAlter` |  | Name, GSTIN, address. |
| 14 | Settings | Switch Company | `companySelect` | `F3` (anywhere) | Daily for multi-company owners. |
| 14 | Settings | Features | `features` | `F11` (anywhere) | Turns whole modules on or off, which also shapes this menu. |
| 14 | Settings | Users & Security | `security` |  | Set up when staff are added. |
| 14 | Settings | Print Configuration | `printConfig` |  | Set up once; tweaked rarely. |
| 14 | Settings | GST Rate Setup | `gstRateSetup`<br>feature `gst` |  | Where a Zoho user looks for "Taxes". |
| 14 | Settings | Sync Settings | `syncLogin` |  | Once per device. |
| 14 | Settings | Subscription & Billing | `billing` |  | Visited at renewal. |
| 14 | Settings | Edit Log | `auditLog` |  | An audit read, on demand. |

### 4.3 Footer links (the way to everything not listed)

Each is the last row of its group, quiet (`--dim`, chevron), opens the Gateway submenu with `go()`. Shown only when security control is off or the operator has full rights (otherwise a restricted role could land on an empty submenu: `MenuScreen.offered()` empties them). Never a search result, never pinnable, highlighted when the screen on top is that submenu.

| Group | Footer row | Opens | Gateway path it replaces |
|---|---|---|---|
| Accounting | Books & registers… | `menu` id=accountBooks | D, then A |
| Inventory | Inventory reports… | `menu` id=inventoryBooks | D, then I |
| GST | Statutory reports… | `menu` id=statutory | D, then U |
| TDS & TCS | Statutory reports… | `menu` id=statutory | D, then U |
| Reports | More reports… | `menu` id=display | D |
| Masters | All masters… | `menu` id=create | C |
| Payroll & HR | Payroll reports… | `menu` id=payrollRep | D, then P |
| Settings | More settings… | `menu` id=companyInfo | Alt+F3 |

### 4.4 The two dynamic groups

* **Pinned** (first, `id: 'pinned'`, open by default): the operator's pins from `loadSidebar(cid).pins`, filtered the way Quick access filters them (`screenDenial`, deleted saved views removed), at most `PINS_MAX` (8). Row label: a destination that has a tree row is named by that row ("Sales Invoice", not Quick access's "New Sales": the user pinned it from there); a party's or item's vouchers lead with the record's own name ("Bhosale Engineering Works", the kind in the tooltip, and after a middle dot only when two pins share a name); a saved view shows its own name; anything else keeps `labelFor`'s "Title: qualifier". The label reserves the star's width, so a long name ends in an ellipsis beside the star. Star always lit; clicking the star unpins. A pin (and a saved view) of a screen whose company feature is switched off is not shown (`featureOf`, the same flag as the tree row), as the Gateway and Go To hide it; Quick access keeps showing every pin. A destination that is both pinned and listed shows in both places (the tree's star is lit). Hidden when empty. The star on any tree row pins `item.to` when `isRecordable(item.to)`; rows whose route is not recordable (`pickMaster` and the other `NEVER` screens in `recents.ts`: Ledgers, Groups, Units, Godowns, Cost Centres, Voucher Types, Employees, the Ledger / Group Summary pickers, footer links) show **no star** rather than a dead one. The star is disabled with the title "Pinned list is full (8)" at the limit, as in Quick access.
* **Saved views** (`id: 'views'`, icon `rows`, closed by default, between Reports and Masters): the operator's saved report views for this company, `useViews(cid)` filtered to those with `launcherFor(view) !== null`, labelled with the view's own name, tooltip = the report's name (`viewDef(view.report).label`), opens `launcherFor(view)` (`reportView`, which applies the view then replaces itself with the report; rights are judged on the target, `areaOfScreen`). The first **6** are listed; the rest are reachable as "View: name" in Go To. Hidden when there are none. Pinning a view from its row works like any row (`reportView` is recordable; `recordVisit` already ignores it).

### 4.5 What was left out, renamed or moved, and why

**Left out (reachable from the footer link, the Gateway and Go To):**
* **Batches & Expiry** (`batchSummary`): a `ComingSoon` placeholder today. Rule: **no row may open a stub.** Add the row the day the screen ships (the guard in section 11 fails when `batchSummary` stops being a stub, as a reminder).
* GST: ITC Reversal (Rule 37), Ineligible ITC 17(5), GSTR-1 Advances, Documents Issued, Bill of Supply. TDS: Exceptions / Uncertain. (Held at 9 rows; GSTR-9 outranks them.)
* Accounting: the Memorandum / Optional / Post-dated registers, Journal / Debit Note / Credit Note registers, Exception Reports, Cancelled Vouchers, Cost Category / Centre reports, Cost Centre P&L. Inventory: Godown-wise Stock, Stock Valuation Methods, Movement Analysis, Stock Query, Sales / Purchase Orders Outstanding.
* Payroll: Payroll Statement, Attendance Register, Head Count, Multi-Period Comparison, Gratuity, Leave Register, Leave Type. Masters: Stock Group, Stock Category, Employee Group, Pay Head, Attendance / Production Type, TDS Nature, TCS Nature (all inside "All masters...").
* Company menu: Getting Started, Narration Snippets, Favourite Shortcuts, API Reference, Split Company Data (year-end, administrator only, found by Go To "year end"), Sync Mismatch Report, Switch Operator.

**Renamed for the job vocabulary** (the Tally name stays as an alias and, where one exists, the key on the row): Sales to **Sales Invoice**; Purchase to **Purchase Invoice**; Receipt to **Payment Received**; Payment to **Payment Made**; Profit & Loss A/c to **Profit & Loss**; Security (Users & Roles) to **Users & Security**; Alter Company to **Company Details**; Select Company to **Switch Company**; Cheque Printing & Register to **Cheque Printing** (its own crumb says the same; "register" is an alias); TDS Challan Reconciliation to **TDS Challan Recon** (156px would not fit); Contra to **Transfer (Contra)**.

**Moved:** Contra to Banking & Cash; Employees to Payroll & HR; Custom Fields to Data & Utilities; GST Rate Setup to Settings (where a Zoho user looks for "Taxes"). **Duplicated on purpose (one destination, two rows):** **Stock Items** is under Inventory (where Zoho users look for Items) and under Masters (where Tally users look). Only the first row in tree order is ever highlighted or auto-opened (section 7.2).

**Brand rule:** the only visible "Tally" is the existing migration utility's own name, **Compare with Tally Data** (nominative, on a migration screen: allowed by `CLAUDE.md`). Aliases (never drawn) may say "tally"; the guard enforces both.

### 4.6 The name bridge (what each audience will type or expect)

| Row says | Zoho Books (from memory) | Tally | Key |
|---|---|---|---|
| Sales Invoice | Invoices | Sales voucher | `F8` |
| Payment Received | Payments Received | Receipt | `F6` |
| Credit Note | Credit Notes | Credit Note | `Alt+F6` |
| Delivery Note | Delivery Challans | Delivery Note | |
| Purchase Invoice | Bills | Purchase voucher | `F9` |
| Payment Made | Payments Made, Expenses | Payment | `F5` |
| Debit Note | Vendor Credits | Debit Note | `Alt+F5` |
| Transfer (Contra) | Banking: transfer between accounts | Contra | `F4` |
| Journal | Manual Journals | Journal | `F7` |
| Ledgers | Customers, Vendors, accounts | Ledger | |
| Stock Items | Items | Stock Item | |
| Physical Stock | Inventory Adjustments | Physical Stock | `Ctrl+F7` |
| Profit & Loss, Balance Sheet | Profit and Loss, Balance Sheet | same | `P`, `B` (Gateway) |

All of these are search aliases too (section 6.1 `aliases`).

---

## 5. The collapse chord: audit and decision

**Chosen: `Ctrl+\` (printed `⌘\` on a Mac).** Audited against every quoted chord in `src/` (grep of `'Ctrl+…'` / `'Alt+…'` literals, `e.key ===` reads, the key sheet's data), `keys.ts` (`BROWSER_RESERVED`, the G-chord, hold-Alt), `electron/main.cjs` and the F-key rail.

| Candidate | Verdict | Evidence |
|---|---|---|
| `Ctrl+B`, `Alt+B` | taken | `reportBasis.tsx` (Basis of values); key sheet rows "Batch allocation", "Basis of values" |
| `Alt+H`, `Alt+V`, `Alt+P`, `Alt+K`, `Alt+N`, `Alt+R`, `Alt+S`, ... | taken | wave 1/2: GST details, saved views / As Voucher, Print, hold, After Accept, reuse, sort. Taken Alt letters: A B C D E F G H I J K L N O P R S T V W Y and `Alt+2` |
| `Alt+M` | free but rejected | one of the last free Alt letters; spending it on a view toggle is wasteful; on Windows / Linux Alt+letter shares a namespace with browser menu mnemonics; on a Mac Option+M types a character in a field |
| `Ctrl+M`, `Ctrl+H`, `Ctrl+T`, `Ctrl+N`, `Ctrl+W`, `Ctrl+Q`, `Ctrl+L` | reserved / taken | `BROWSER_RESERVED` (⌘M minimises on a Mac); Alt aliases exist for the app's own uses |
| `Ctrl+.` | taken | `DISMISS_CHORD` (`feedback.tsx`) |
| `Ctrl+/` | rejected | free, but `/` is the report filter key and ⌘/ conventionally means "shortcuts" |
| `Ctrl+D`, `Ctrl+S`, `Ctrl+G`, `Ctrl+O`, `Ctrl+U`, `Ctrl+Y` | rejected | unbound in the app but browser commands (bookmark, save page, find next, open, view source); Ctrl+S means "save" to every Zoho user while this app accepts with Ctrl+A |
| `Ctrl+[`, `Ctrl+]` | rejected | ⌘[ / ⌘] are Back / Forward in Safari and Chrome on a Mac |
| `Ctrl+Shift+x` | rejected | `combo()` does not encode Shift (a documented GOTCHA) and most are browser commands |
| `Ctrl+Alt+1..9` | taken | Favourite shortcuts |
| `Ctrl+Space`, `Ctrl+Enter`, `Ctrl+↑↓` | taken | recall, accept variants, report moves; `Ctrl+←→` is word-jump in a field |
| bare `[` or `\` | rejected | bare keys are dead inside fields (`isTypingTarget`), which is exactly where the operator is when they want the width back: a voucher |
| F1 to F12 with Alt / Ctrl | all taken | the App handler swallows `^(Ctrl\+\|Alt\+)?F\d{1,2}$` |
| **`Ctrl+\`** | **chosen** | zero uses in `src/`; not in `BROWSER_RESERVED`; Electron has no menu (`Menu.setApplicationMenu(null)`); no macOS system shortcut; outside every letter namespace, so it can never collide with a future screen's letter; **works while typing** (a Ctrl chord inserts no text); precedent (from memory): Notion toggles its sidebar on it, Figma hides its UI on it |

**Edge cases decided:**
* Layouts where backslash needs AltGr (for example German): Windows reports AltGr as Ctrl+Alt, so the matcher refuses it (`altKey`) and the user uses the status-bar toggle or the hamburger. Intended: that keystroke is a character being typed.
* Some Windows / Linux layouts report Ctrl+\ with `e.key` as the control character U+001C: the matcher then accepts `e.code === 'Backslash'`. `Ctrl+Shift+\` and `Alt+\` stay free.
* A held key repeats: the handler swallows repeats and toggles once.
* **Wiring:** a `NavChordHost` component mounted once in `AppShell` registers ONE permanent `useFallbackKeys` handler (bottom of the stack: every screen is asked first, the pattern `FavouritesHost` documents), and **never** toggles `active`. A modal overlay that consumes unknown chords (Go To, the key sheet) shields it, which is correct.
* **Listed in:** the `?` sheet (`GLOBAL_KEYS`, "Everywhere": "Hide or show the left menu: full-screen view"), the in-rail rows (section 7.3), and `LOCKED_KEYS` in `favouritesCore.ts` ("Go anywhere") so Favourite shortcuts shows it as taken. The favourites-style scan in section 11 keeps it unique.

---
## 6. Data model and state

Two pure modules (no React, no store instance, no Dexie; same shape as `ui/paletteRank.ts`, `state/recents.ts`, `ui/favouritesCore.ts`) so every rule is assertable headlessly, plus one small Zustand store. Both reference modules below were compiled against the real project and run (section 1, item 10); implementers copy them, then change only what a reviewer asks.

### 6.1 `src/ui/navTree.ts` (pure)

The tree **is** the published table of section 4.2, as typed data. `to`, `create` and `also` are typed `Screen`, so a route that does not exist, or lacks a required parameter, does not compile. It needs the six new icon names from section 10.4 (`grid`, `bag`, `funnel`) to type-check.

```ts
import type { Screen } from '../state/store';
import type { CompanyFeatures } from '../types';
import type { IconName } from './icons';
import { screenDenial, securityOn, isFull } from '../core/access';
import type { AccessCtx } from '../core/access';
import { screenKey } from '../state/recents';

// ─── types ───────────────────────────────────────────────────────────────────

/** Where a printed key actually fires: it decides the tooltip, so a keycap never over-promises. */
export type KeyWhere = 'anywhere' | 'menus' | 'gateway';

export interface NavItem {
  /** Stable, unique across the tree ('sales.invoice'). Used for aria ids and by the tests; never persisted. */
  id: string;
  /** The row's text, 13px. Measured to fit the rail (section 10.2): keep it under 26 characters. */
  label: string;
  /** EXACTLY the Screen the Gateway pushes for the same destination. */
  to: Screen;
  /** The "+" action on a master row: the create form. Needs the same right and feature as the row. */
  create?: Screen;
  /** Other screens that count as "being on this row" (the edit forms a master list leads to). */
  also?: Screen[];
  /** Raw combo as keys.ts spells it ('F8', 'Alt+F6', 'W'); printed through keyLabel(). */
  key?: string;
  keyWhere?: KeyWhere;
  /** The company feature that must be on, exactly as MenuScreen / the palette gate the same destination. */
  feature?: keyof CompanyFeatures;
  /** Invisible search vocabulary: Tally names, Zoho names, Hindi-English trade words. Lower case. */
  aliases: string[];
}

export interface NavGroup {
  id: string;
  label: string;
  icon: IconName;
  /** One-row group (Dashboard): drawn as a row with an icon, no chevron, no children. */
  leaf?: boolean;
  /** Footer row that opens the matching Gateway submenu: the way to everything not listed here. */
  more?: { label: string; to: Screen };
  items: NavItem[];
}

// ─── the tree, in priority order (section 4.2 is the same data, with the reasons) ──────────────────

export const NAV_TREE: readonly NavGroup[] = [
  { id: 'dashboard', label: 'Dashboard', icon: 'grid', leaf: true, items: [
    { id: 'dashboard', label: 'Dashboard', to: { s: 'dashboard' }, key: 'W', keyWhere: 'gateway', aliases: ['home', 'overview', 'summary', 'kpi', 'cockpit'] },
  ] },
  { id: 'sales', label: 'Sales', icon: 'invoice', items: [
    { id: 'sales.invoice', label: 'Sales Invoice', to: { s: 'voucher', base: 'Sales' }, key: 'F8', keyWhere: 'menus', aliases: ['invoice', 'bill', 'tax invoice', 'sale', 'sales voucher', 'new sale'] },
    { id: 'sales.received', label: 'Payment Received', to: { s: 'voucher', base: 'Receipt' }, key: 'F6', keyWhere: 'menus', aliases: ['receipt', 'payments received', 'collection', 'received', 'customer payment', 'invoice payment'] },
    { id: 'sales.receivables', label: 'Receivables', to: { s: 'outstandings', side: 'receivables' }, feature: 'billwise', aliases: ['debtors', 'sundry debtors', 'outstanding', 'udhaar', 'who owes me', 'ageing'] },
    { id: 'sales.register', label: 'Sales Register', to: { s: 'register', base: 'Sales' }, aliases: ['invoices list', 'sales list', 'sales report'] },
    { id: 'sales.order', label: 'Sales Order', to: { s: 'voucher', base: 'SalesOrder' }, feature: 'orders', aliases: ['so', 'order', 'customer order', 'quotation'] },
    { id: 'sales.creditnote', label: 'Credit Note', to: { s: 'voucher', base: 'CreditNote' }, key: 'Alt+F6', keyWhere: 'menus', aliases: ['sales return', 'cn', 'return', 'credit memo'] },
    { id: 'sales.delivery', label: 'Delivery Note', to: { s: 'voucher', base: 'DeliveryNote' }, feature: 'inventory', aliases: ['challan', 'delivery challan', 'dispatch', 'challan out'] },
    { id: 'sales.ageing', label: 'Ageing Analysis', to: { s: 'ageing' }, feature: 'billwise', aliases: ['ageing buckets', 'aging', 'overdue', 'bucket'] },
    { id: 'sales.reminders', label: 'Reminder Letters', to: { s: 'reminders' }, feature: 'billwise', aliases: ['dunning', 'payment reminder', 'follow up letter', 'overdue letter'] },
  ] },
  { id: 'purchases', label: 'Purchases', icon: 'bag', items: [
    { id: 'purchases.invoice', label: 'Purchase Invoice', to: { s: 'voucher', base: 'Purchase' }, key: 'F9', keyWhere: 'menus', aliases: ['purchase bill', 'bill', 'supplier invoice', 'inward', 'kharid', 'new purchase'] },
    { id: 'purchases.made', label: 'Payment Made', to: { s: 'voucher', base: 'Payment' }, key: 'F5', keyWhere: 'menus', aliases: ['payment', 'payments made', 'paid', 'expense', 'kharcha', 'outgoing', 'vendor payment', 'bill payment'] },
    { id: 'purchases.payables', label: 'Payables', to: { s: 'outstandings', side: 'payables' }, feature: 'billwise', aliases: ['creditors', 'sundry creditors', 'dues', 'bills to pay', 'outstanding'] },
    { id: 'purchases.register', label: 'Purchase Register', to: { s: 'register', base: 'Purchase' }, aliases: ['bills list', 'purchase list', 'purchase report'] },
    { id: 'purchases.order', label: 'Purchase Order', to: { s: 'voucher', base: 'PurchaseOrder' }, feature: 'orders', aliases: ['po', 'order', 'supplier order'] },
    { id: 'purchases.debitnote', label: 'Debit Note', to: { s: 'voucher', base: 'DebitNote' }, key: 'Alt+F5', keyWhere: 'menus', aliases: ['purchase return', 'dn', 'return', 'vendor credit'] },
    { id: 'purchases.receiptnote', label: 'Receipt Note', to: { s: 'voucher', base: 'ReceiptNote' }, feature: 'inventory', aliases: ['grn', 'goods received', 'goods receipt', 'inward note'] },
  ] },
  { id: 'banking', label: 'Banking & Cash', icon: 'bank', items: [
    { id: 'banking.book', label: 'Cash/Bank Book', to: { s: 'cashBankBook' }, aliases: ['cash book', 'bank book', 'cash balance', 'bank balance'] },
    { id: 'banking.brs', label: 'Bank Reconciliation', to: { s: 'brs' }, aliases: ['brs', 'reco', 'reconcile', 'bank statement match'] },
    { id: 'banking.import', label: 'Import Bank Statement', to: { s: 'brsImport' }, aliases: ['bank statement', 'statement import', 'mt940', 'camt', 'auto match', 'upload statement'] },
    { id: 'banking.contra', label: 'Transfer (Contra)', to: { s: 'voucher', base: 'Contra' }, key: 'F4', keyWhere: 'menus', aliases: ['contra', 'cash deposit', 'withdrawal', 'bank transfer', 'transfer between accounts'] },
    { id: 'banking.cheque', label: 'Cheque Printing', to: { s: 'cheque' }, aliases: ['print cheque', 'cheque register', 'register', 'cheque leaf'] },
    { id: 'banking.pdc', label: 'Post-Dated Cheques', to: { s: 'pdcRegister' }, aliases: ['pdc', 'postdated', 'post dated'] },
    { id: 'banking.books', label: 'Cheque Books', to: { s: 'chequeBooks' }, aliases: ['cheque leaves', 'cheque series'] },
    { id: 'banking.advice', label: 'Payment Advice', to: { s: 'paymentAdvice' }, aliases: ['remittance', 'remittance advice', 'payment slip'] },
    { id: 'banking.slip', label: 'Deposit Slip', to: { s: 'depositSlip' }, aliases: ['pay in slip', 'bank deposit', 'paying slip'] },
  ] },
  { id: 'accounting', label: 'Accounting', icon: 'register', more: { label: 'Books & registers…', to: { s: 'menu', id: 'accountBooks' } }, items: [
    { id: 'accounting.daybook', label: 'Day Book', to: { s: 'daybook' }, key: 'K', keyWhere: 'gateway', aliases: ['all vouchers', 'journal register', 'daily transactions', 'transactions'] },
    { id: 'accounting.journal', label: 'Journal', to: { s: 'voucher', base: 'Journal' }, key: 'F7', keyWhere: 'menus', aliases: ['jv', 'adjustment', 'adjustment entry', 'manual journal'] },
    { id: 'accounting.tb', label: 'Trial Balance', to: { s: 'trialBalance' }, aliases: ['tb', 'trial'] },
    { id: 'accounting.ledger', label: 'Ledger', to: { s: 'pickMaster', kind: 'ledger', goal: 'book' }, aliases: ['ledger book', 'account statement', 'general ledger', 'account transactions'] },
    { id: 'accounting.group', label: 'Group Summary', to: { s: 'pickMaster', kind: 'group', goal: 'book' }, aliases: ['group', 'sundry debtors total', 'account group'] },
    { id: 'accounting.cost', label: 'Cost Centre Analysis', to: { s: 'costAnalysis' }, feature: 'costCentres', aliases: ['cost center', 'project profit', 'department'] },
    { id: 'accounting.budgets', label: 'Budgets', to: { s: 'budgets' }, aliases: ['budget', 'plan', 'target'] },
    { id: 'accounting.interest', label: 'Interest Calculation', to: { s: 'interest' }, aliases: ['interest on overdue', 'interest calc'] },
  ] },
  { id: 'inventory', label: 'Inventory', icon: 'box', more: { label: 'Inventory reports…', to: { s: 'menu', id: 'inventoryBooks' } }, items: [
    { id: 'inventory.summary', label: 'Stock Summary', to: { s: 'stockSummary' }, key: 'S', keyWhere: 'gateway', feature: 'inventory', aliases: ['stock', 'closing stock', 'stock on hand', 'inventory summary'] },
    { id: 'inventory.items', label: 'Stock Items', to: { s: 'pickMaster', kind: 'stockItem' }, create: { s: 'stockItem' }, also: [{ s: 'stockItem' }], feature: 'inventory', aliases: ['items', 'products', 'item master', 'sku', 'new item'] },
    { id: 'inventory.reorder', label: 'Reorder Status', to: { s: 'reorderStatus' }, feature: 'inventory', aliases: ['low stock', 'reorder level', 'what to buy', 'shortage'] },
    { id: 'inventory.stockjournal', label: 'Stock Journal', to: { s: 'voucher', base: 'StockJournal' }, key: 'Alt+F7', keyWhere: 'menus', feature: 'inventory', aliases: ['stock transfer', 'consumption', 'transfer between godowns'] },
    { id: 'inventory.physical', label: 'Physical Stock', to: { s: 'voucher', base: 'PhysicalStock' }, key: 'Ctrl+F7', keyWhere: 'menus', feature: 'inventory', aliases: ['stock count', 'physical verification', 'stock taking', 'stock audit', 'inventory adjustment', 'stock adjustment'] },
    { id: 'inventory.godown', label: 'Godown Summary', to: { s: 'godownSummary' }, feature: 'inventory', aliases: ['warehouse', 'location stock', 'godown stock'] },
    { id: 'inventory.mfg', label: 'Manufacturing Journal', to: { s: 'manufacture' }, feature: 'inventory', aliases: ['production', 'bom', 'bill of materials', 'assembly', 'manufacturing'] },
    { id: 'inventory.ageing', label: 'Stock Ageing', to: { s: 'stockAgeing' }, feature: 'inventory', aliases: ['slow moving', 'dead stock', 'old stock'] },
  ] },
  { id: 'gst', label: 'GST', icon: 'percent', more: { label: 'Statutory reports…', to: { s: 'menu', id: 'statutory' } }, items: [
    { id: 'gst.gstr1', label: 'GSTR-1', to: { s: 'gstr1' }, feature: 'gst', aliases: ['r1', 'outward return', 'sales return gst', 'gst return'] },
    { id: 'gst.gstr3b', label: 'GSTR-3B', to: { s: 'gstr3b' }, feature: 'gst', aliases: ['3b', 'monthly return', 'gst payment', 'summary return'] },
    { id: 'gst.gstr2b', label: 'GSTR-2B Reconciliation', to: { s: 'gstr2b' }, feature: 'gst', aliases: ['2b', 'itc match', 'input credit', 'purchase match'] },
    { id: 'gst.einvoice', label: 'e-Invoice / e-Way Bill', to: { s: 'einvoice' }, feature: 'gst', aliases: ['irn', 'eway', 'ewb', 'einvoice', 'qr', 'portal'] },
    { id: 'gst.hsn', label: 'HSN Summary', to: { s: 'hsnSummary' }, feature: 'gst', aliases: ['hsn', 'sac', 'hsn wise'] },
    { id: 'gst.validation', label: 'GST Validation', to: { s: 'gstValidation' }, feature: 'gst', aliases: ['gst exceptions', 'missing gstin', 'gst errors', 'pre filing check'] },
    { id: 'gst.recon', label: 'GST Reconciliation', to: { s: 'gstRecon' }, feature: 'gst', aliases: ['3b vs 1', 'books vs return', 'gst difference'] },
    { id: 'gst.rcm', label: 'Reverse Charge Register', to: { s: 'rcmRegister' }, feature: 'gst', aliases: ['rcm', 'reverse charge'] },
    { id: 'gst.gstr9', label: 'GSTR-9 Annual Return', to: { s: 'gstr9' }, feature: 'gst', aliases: ['annual return', 'r9', 'yearly gst'] },
  ] },
  { id: 'tds', label: 'TDS & TCS', icon: 'doc', more: { label: 'Statutory reports…', to: { s: 'menu', id: 'statutory' } }, items: [
    { id: 'tds.outstanding', label: 'TDS Outstanding', to: { s: 'tdsReport' }, feature: 'tds', aliases: ['tds payable', 'tds due', 'deducted'] },
    { id: 'tds.returns', label: 'TDS Challans & Returns', to: { s: 'tdsReturns' }, feature: 'tds', aliases: ['challan', '26q', 'tds return', 'tds deposit'] },
    { id: 'tds.quarterly', label: 'TDS Quarterly Tracker', to: { s: 'tdsQuarterly' }, feature: 'tds', aliases: ['quarter', 'due dates', '26q tracker'] },
    { id: 'tds.deductee', label: 'TDS Deductee-wise', to: { s: 'tdsDeductee' }, feature: 'tds', aliases: ['party wise tds', 'deductee'] },
    { id: 'tds.recon', label: 'TDS Challan Recon', to: { s: 'tdsChallanRecon' }, feature: 'tds', aliases: ['challan reconciliation', 'challan match', 'bsr', 'deposit match', 'reco'] },
    { id: 'tds.form16a', label: 'Form 16A Certificates', to: { s: 'form16a' }, feature: 'tds', aliases: ['16a', 'tds certificate'] },
    { id: 'tds.interest', label: 'TDS Interest & Late Fee', to: { s: 'tdsInterest' }, feature: 'tds', aliases: ['late fee', '234e', 'tds interest'] },
    { id: 'tds.tcs', label: 'TCS Outstanding', to: { s: 'tcsReport' }, feature: 'tcs', aliases: ['tcs payable', 'tax collected at source'] },
    { id: 'tds.tcs206c', label: 'TCS 206C(1H) Turnover', to: { s: 'tcs206c' }, feature: 'tcs', aliases: ['206c', 'tcs turnover', 'tcs threshold'] },
  ] },
  { id: 'reports', label: 'Reports', icon: 'trend', more: { label: 'More reports…', to: { s: 'menu', id: 'display' } }, items: [
    { id: 'reports.bs', label: 'Balance Sheet', to: { s: 'balanceSheet' }, key: 'B', keyWhere: 'gateway', aliases: ['bs', 'financial position', 'assets liabilities'] },
    { id: 'reports.pl', label: 'Profit & Loss', to: { s: 'profitLoss' }, key: 'P', keyWhere: 'gateway', aliases: ['pl', 'p&l', 'income statement', 'profit', 'loss'] },
    { id: 'reports.cashflow', label: 'Cash Flow', to: { s: 'cashFlow' }, aliases: ['cash movement', 'cashflow'] },
    { id: 'reports.ratio', label: 'Ratio Analysis', to: { s: 'ratioAnalysis' }, key: 'R', keyWhere: 'gateway', aliases: ['ratios', 'key indicators', 'current ratio', 'margin'] },
    { id: 'reports.funds', label: 'Funds Flow', to: { s: 'fundsFlow' }, aliases: ['sources and uses', 'fund flow'] },
    { id: 'reports.mis', label: 'Top-N Analysis (MIS)', to: { s: 'topMis' }, aliases: ['mis', 'top customers', 'top items', 'best sellers', 'top 10'] },
    { id: 'reports.profitability', label: 'Profitability Analysis', to: { s: 'profitability' }, aliases: ['margin', 'gross profit', 'item profit'] },
    { id: 'reports.budget', label: 'Budget vs Actual', to: { s: 'budgetVsActual' }, aliases: ['variance', 'budget variance', 'plan vs actual'] },
  ] },
  { id: 'masters', label: 'Masters', icon: 'tree', more: { label: 'All masters…', to: { s: 'menu', id: 'create' } }, items: [
    { id: 'masters.ledgers', label: 'Ledgers', to: { s: 'pickMaster', kind: 'ledger' }, create: { s: 'ledger' }, also: [{ s: 'ledger' }], aliases: ['customers', 'vendors', 'suppliers', 'parties', 'accounts', 'new ledger', 'party master'] },
    { id: 'masters.coa', label: 'Chart of Accounts', to: { s: 'coa' }, key: 'H', keyWhere: 'gateway', aliases: ['coa', 'account tree', 'accounts list'] },
    { id: 'masters.groups', label: 'Groups', to: { s: 'pickMaster', kind: 'group' }, create: { s: 'group' }, also: [{ s: 'group' }], aliases: ['account groups', 'ledger groups', 'new group'] },
    { id: 'masters.items', label: 'Stock Items', to: { s: 'pickMaster', kind: 'stockItem' }, create: { s: 'stockItem' }, also: [{ s: 'stockItem' }], feature: 'inventory', aliases: ['items', 'products', 'item master', 'new item'] },
    { id: 'masters.vtypes', label: 'Voucher Types', to: { s: 'pickMaster', kind: 'voucherType' }, create: { s: 'voucherType' }, also: [{ s: 'voucherType' }], aliases: ['numbering', 'invoice series', 'voucher series', 'new voucher type'] },
    { id: 'masters.units', label: 'Units', to: { s: 'pickMaster', kind: 'unit' }, create: { s: 'unit' }, also: [{ s: 'unit' }], feature: 'inventory', aliases: ['uom', 'unit of measure', 'nos', 'kg'] },
    { id: 'masters.godowns', label: 'Godowns', to: { s: 'pickMaster', kind: 'godown' }, create: { s: 'godown' }, also: [{ s: 'godown' }], feature: 'inventory', aliases: ['warehouse', 'location', 'store'] },
    { id: 'masters.cc', label: 'Cost Centres', to: { s: 'pickMaster', kind: 'costCentre' }, create: { s: 'costCentre' }, also: [{ s: 'costCentre' }], feature: 'costCentres', aliases: ['cost center', 'project', 'branch', 'department'] },
    { id: 'masters.pricelevels', label: 'Price Levels', to: { s: 'priceLevels' }, feature: 'inventory', aliases: ['price list', 'wholesale price', 'rate list'] },
  ] },
  { id: 'payroll', label: 'Payroll & HR', icon: 'people', more: { label: 'Payroll reports…', to: { s: 'menu', id: 'payrollRep' } }, items: [
    { id: 'payroll.attendance', label: 'Attendance', to: { s: 'attendance' }, feature: 'payroll', aliases: ['attendance voucher', 'present absent', 'overtime'] },
    { id: 'payroll.voucher', label: 'Payroll Voucher', to: { s: 'payrollVch' }, feature: 'payroll', aliases: ['salary', 'run payroll', 'process salary'] },
    { id: 'payroll.sheet', label: 'Pay Sheet', to: { s: 'paySheet' }, feature: 'payroll', aliases: ['salary sheet', 'salary register'] },
    { id: 'payroll.payslip', label: 'Payslip', to: { s: 'payslip' }, feature: 'payroll', aliases: ['salary slip', 'pay slip'] },
    { id: 'payroll.disburse', label: 'Salary Disbursement', to: { s: 'salaryDisbursement' }, feature: 'payroll', aliases: ['salary payment', 'pay salary', 'bank transfer salary'] },
    { id: 'payroll.statutory', label: 'PF / ESI / PT', to: { s: 'payrollStatutory' }, feature: 'payroll', aliases: ['provident fund', 'esi', 'professional tax', 'epf', 'statutory'] },
    { id: 'payroll.leave', label: 'Leave', to: { s: 'leaveApply' }, feature: 'payroll', aliases: ['apply leave', 'record leave', 'time off', 'absence'] },
    { id: 'payroll.balance', label: 'Leave Balance', to: { s: 'leaveBalance' }, feature: 'payroll', aliases: ['leave register', 'leaves left'] },
    { id: 'payroll.employees', label: 'Employees', to: { s: 'pickMaster', kind: 'employee' }, create: { s: 'employee' }, also: [{ s: 'employee' }], feature: 'payroll', aliases: ['staff', 'people', 'new employee', 'hr'] },
  ] },
  { id: 'crm', label: 'CRM', icon: 'funnel', items: [
    { id: 'crm.followups', label: 'Follow-ups Due', to: { s: 'followUps' }, aliases: ['tasks', 'calls due', 'reminders', 'to do'] },
    { id: 'crm.leads', label: 'Leads', to: { s: 'leads' }, aliases: ['prospects', 'enquiries', 'inquiries'] },
    { id: 'crm.newlead', label: 'New Lead', to: { s: 'lead' }, aliases: ['add lead', 'create lead', 'new enquiry'] },
    { id: 'crm.pipeline', label: 'Pipeline Summary', to: { s: 'pipeline' }, aliases: ['funnel', 'stages', 'deals'] },
  ] },
  { id: 'data', label: 'Data & Utilities', icon: 'plug', items: [
    { id: 'data.backup', label: 'Backup / Restore', to: { s: 'backupRestore' }, aliases: ['backup', 'restore', 'export data', 'auto backup', 'save a copy'] },
    { id: 'data.bulk', label: 'Bulk Entry', to: { s: 'bulkEntry' }, aliases: ['paste from excel', 'import vouchers', 'excel entry', 'many vouchers'] },
    { id: 'data.import', label: 'Import Data', to: { s: 'importData' }, aliases: ['migrate', 'import from tally', 'csv', 'xml', 'upload', 'switch from tally'] },
    { id: 'data.export', label: 'Export Data', to: { s: 'exportXml' }, aliases: ['export for tally', 'xml', 'send to ca', 'download'] },
    { id: 'data.check', label: 'Data Check', to: { s: 'dataCheck' }, aliases: ['integrity', 'repair', 'verify data', 'unbalanced', 'orphan'] },
    { id: 'data.fields', label: 'Custom Fields', to: { s: 'customFields' }, aliases: ['udf', 'extra fields', 'custom field'] },
    { id: 'data.api', label: 'API Keys', to: { s: 'apiKeys' }, aliases: ['token', 'integration', 'developer', 'rest api'] },
    { id: 'data.webhooks', label: 'Webhooks', to: { s: 'webhooks' }, aliases: ['events', 'integration', 'callback'] },
    { id: 'data.compare', label: 'Compare with Tally Data', to: { s: 'tallyCompare' }, aliases: ['reconcile', 'verify migration', 'match tally', 'migration check'] },
  ] },
  { id: 'settings', label: 'Settings', icon: 'gear', more: { label: 'More settings…', to: { s: 'menu', id: 'companyInfo' } }, items: [
    { id: 'settings.company', label: 'Company Details', to: { s: 'companyAlter' }, aliases: ['alter company', 'company info', 'gstin', 'address', 'profile', 'organisation'] },
    { id: 'settings.switch', label: 'Switch Company', to: { s: 'companySelect' }, key: 'F3', keyWhere: 'anywhere', aliases: ['select company', 'change company', 'another company', 'organisation'] },
    { id: 'settings.features', label: 'Features', to: { s: 'features' }, key: 'F11', keyWhere: 'anywhere', aliases: ['f11', 'modules', 'turn on', 'turn off', 'enable'] },
    { id: 'settings.security', label: 'Users & Security', to: { s: 'security' }, aliases: ['users', 'roles', 'permissions', 'pin', 'access'] },
    { id: 'settings.print', label: 'Print Configuration', to: { s: 'printConfig' }, aliases: ['invoice format', 'invoice layout', 'customise invoice', 'print setup', 'logo', 'template'] },
    { id: 'settings.gstrates', label: 'GST Rate Setup', to: { s: 'gstRateSetup' }, feature: 'gst', aliases: ['tax rates', 'gst rate', 'hsn default', 'taxes', 'slab'] },
    { id: 'settings.sync', label: 'Sync Settings', to: { s: 'syncLogin' }, aliases: ['cloud', 'account', 'login', 'online sync'] },
    { id: 'settings.billing', label: 'Subscription & Billing', to: { s: 'billing' }, aliases: ['plan', 'renew', 'licence', 'invoice for subscription'] },
    { id: 'settings.log', label: 'Edit Log', to: { s: 'auditLog' }, aliases: ['audit trail', 'who changed', 'history', 'audit'] },
  ] },
];

export const NAV_GROUP_IDS: readonly string[] = NAV_TREE.map((g) => g.id);

// ─── gating ──────────────────────────────────────────────────────────────────

/**
 * What THIS operator in THIS company is offered. Same three tests MenuScreen's offered() applies, in the same
 * order: the feature switch, then screenDenial (the operator's rights), then, for a footer link, "would the
 * submenu be empty for this role" (answered conservatively: only full-rights operators, or security off, get it).
 * A group with no row left disappears. Pure: no store, no React.
 */
export function visibleTree(tree: readonly NavGroup[], feats: CompanyFeatures, ctx: AccessCtx): NavGroup[] {
  const openAll = !securityOn(ctx) || isFull(ctx.operator?.role);
  const out: NavGroup[] = [];
  for (const g of tree) {
    const items: NavItem[] = [];
    for (const it of g.items) {
      if (it.feature && !feats[it.feature]) continue;
      if (screenDenial(ctx, it.to) !== null) continue;
      const create = it.create && screenDenial(ctx, it.create) === null ? it.create : undefined;
      items.push(create === it.create ? it : { ...it, create });
    }
    if (items.length === 0) continue;
    const more = g.more && openAll && screenDenial(ctx, g.more.to) === null ? g.more : undefined;
    out.push({ ...g, items, more });
  }
  return out;
}

// ─── search ──────────────────────────────────────────────────────────────────

/** Every word typed must appear somewhere in label + aliases + the group's own name (so "inventory" lists the module). */
export function filterTree(groups: readonly NavGroup[], query: string): NavGroup[] {
  const words = query.toLowerCase().split(/\s+/).filter(Boolean);
  if (words.length === 0) return [...groups];
  const out: NavGroup[] = [];
  for (const g of groups) {
    const items = g.items.filter((it) => {
      const hay = `${it.label} ${it.aliases.join(' ')} ${g.label}`.toLowerCase();
      return words.every((w) => hay.includes(w));
    });
    if (items.length > 0) out.push({ ...g, items, more: undefined }); // a footer link is never a search result
  }
  return out;
}

/** What Enter opens from the filter box: the first row on screen, top to bottom. */
export const firstMatch = (groups: readonly NavGroup[]): NavItem | undefined => groups[0]?.items[0];

// ─── where am I ──────────────────────────────────────────────────────────────

/** True when every parameter `item` names is present, equal, on `top` (so `top` may carry more: an editId, a drill-down). */
function within(item: Screen, top: Screen): boolean {
  if (item.s !== top.s) return false;
  const a = item as unknown as Record<string, unknown>;
  const b = top as unknown as Record<string, unknown>;
  return Object.keys(a).every((k) => k === 's' || a[k] === undefined || a[k] === b[k]);
}

export interface NavHere { groupId: string; rowId: string }

/**
 * The ONE row to highlight for the screen on top of the stack, or null.
 * Pass 1 an exact destination; pass 2 the same screen with more parameters (a voucher being altered, a Stock
 * Summary drilled into a group) or one of an item's `also` forms; pass 3 a group's footer submenu. Within a pass the
 * first row in tree order wins, so a destination listed twice (Stock Items) lights only its higher-priority row.
 */
export function activeRow(groups: readonly NavGroup[], top: Screen | undefined): NavHere | null {
  if (!top) return null;
  const k = screenKey(top);
  for (const g of groups) for (const it of g.items) if (screenKey(it.to) === k) return { groupId: g.id, rowId: it.id };
  for (const g of groups) for (const it of g.items) {
    if (within(it.to, top) || (it.also ?? []).some((a) => within(a, top))) return { groupId: g.id, rowId: it.id };
  }
  for (const g of groups) if (g.more && screenKey(g.more.to) === k) return { groupId: g.id, rowId: g.id + '.more' };
  return null;
}
```

### 6.2 `src/core/navPrefs.ts` (pure) and `src/state/navStore.ts` (store)

```ts
// ─── src/core/navPrefs.ts : the pure half of the rail's per-device preferences ───────────────────────────────────

export type NavViewId = 'menu' | 'quick';

export interface NavPrefs {
  /** Full-screen: the rail is hidden. */
  collapsed: boolean;
  /** Which of the two views the rail shows. */
  view: NavViewId;
  /** Group ids the user left open. Auto-expansion for the current screen is NOT stored here. */
  open: string[];
}

/** localStorage keys. The first is the one AppShell has always used: an existing preference survives. */
export const NAV_STORAGE = { collapsed: 'TW_NAV_COLLAPSED', view: 'TW_NAV_VIEW', open: 'TW_NAV_GROUPS' } as const;

/** What a device that never chose sees: Pinned (when it has any rows) and Sales open, the rest folded. */
export const DEFAULT_OPEN: readonly string[] = ['pinned', 'sales'];

/**
 * Read the stored preferences. `get` is localStorage.getItem (the caller wraps it: it can throw in a private window,
 * with blocked site data, or while a thumbnail is captured). Anything unreadable falls back to the default; a stored
 * group id that is not in `known` (a renamed group, a hand-edited value) is dropped rather than trusted.
 */
export function readPrefs(get: (k: string) => string | null, known: ReadonlySet<string>): NavPrefs {
  let open: string[] = [...DEFAULT_OPEN];
  try {
    const v: unknown = JSON.parse(get(NAV_STORAGE.open) ?? 'null');
    if (Array.isArray(v)) open = v.filter((x): x is string => typeof x === 'string' && known.has(x));
  } catch { /* corrupt blob: keep the default */ }
  return {
    collapsed: get(NAV_STORAGE.collapsed) === '1',
    view: get(NAV_STORAGE.view) === 'quick' ? 'quick' : 'menu',
    open,
  };
}

/** The structural slice of a KeyboardEvent the chord needs (so the suite can test it without a DOM). */
export interface ChordEvent { key: string; code?: string; ctrlKey: boolean; metaKey: boolean; altKey: boolean; shiftKey: boolean }

/**
 * Ctrl+\ (⌘\ on a Mac): hide / show the left menu. combo() cannot be used: it does not encode Shift, and it reads
 * e.key, which a few Windows / Linux layouts report as the control character U+001C for Ctrl+\ . So:
 *  - Ctrl or ⌘ held, Alt NOT held (Ctrl+Alt is how Windows reports AltGr, i.e. a character being TYPED), Shift not held;
 *  - the key prints as a backslash, or, when the platform hands us a control character instead, the physical
 *    Backslash key.
 */
export function isNavCollapseChord(e: ChordEvent): boolean {
  if (!(e.ctrlKey || e.metaKey) || e.altKey || e.shiftKey) return false;
  if (e.key === '\\') return true;
  return e.code === 'Backslash' && (e.key.length !== 1 || e.key.charCodeAt(0) < 32);
}
```

```ts
// src/state/navStore.ts: the one place the rail's per-device preferences live.
// AppShell, the rail, the status-bar toggle and the chord host all read it; none keeps its own copy.
import { create } from 'zustand';
import { NAV_STORAGE, readPrefs, NavPrefs, NavViewId } from '../core/navPrefs';
import { NAV_GROUP_IDS } from '../ui/navTree';

const KNOWN = new Set<string>([...NAV_GROUP_IDS, 'pinned', 'views']);   // the two dynamic groups persist their open state too
const get = (k: string): string | null => { try { return localStorage.getItem(k); } catch { return null; } };
const put = (k: string, v: string): void => { try { localStorage.setItem(k, v); } catch { /* blocked storage: this session still works */ } };

interface NavState extends NavPrefs {
  setCollapsed: (v: boolean) => void;
  toggleCollapsed: () => void;
  setView: (v: NavViewId) => void;
  /** `shownOpen` is what the user SAW (auto-expansion and filtering included), so a click does what it looked like it would. */
  toggleGroup: (id: string, shownOpen: boolean) => void;
}

export const useNavStore = create<NavState>((set) => ({
  ...readPrefs(get, KNOWN),
  setCollapsed: (collapsed) => set(() => { put(NAV_STORAGE.collapsed, collapsed ? '1' : '0'); return { collapsed }; }),
  toggleCollapsed: () => set((s) => { put(NAV_STORAGE.collapsed, s.collapsed ? '0' : '1'); return { collapsed: !s.collapsed }; }),
  setView: (view) => set(() => { put(NAV_STORAGE.view, view); return { view }; }),
  toggleGroup: (id, shownOpen) => set((s) => {
    const open = shownOpen ? s.open.filter((x) => x !== id) : s.open.includes(id) ? s.open : [...s.open, id];
    put(NAV_STORAGE.open, JSON.stringify(open));
    return { open };
  }),
}));
```

### 6.3 Persistence keys (all per device, all wrapped in try / catch, all optional: the page renders correctly with storage blocked)

| Key | Values | Written by | Read by |
|---|---|---|---|
| `TW_NAV_COLLAPSED` | `'1'` hidden, `'0'` shown (the key `AppShell` has always used: an existing choice survives) | `useNavStore` | store init |
| `TW_NAV_VIEW` | `'menu'` (absent / anything else) or `'quick'` | `setView` | store init |
| `TW_NAV_GROUPS` | JSON array of open group ids; absent = `['pinned','sales']`; `[]` = the user closed everything (respected) | `toggleGroup` | store init (unknown ids dropped) |
| *(not stored)* | filter text; the group auto-opened for the current screen; the keyboard "current" row | | |
| existing, untouched | `TW_SIDEBAR_v1:<cid>` (pins, recents), `tw-report-views-v1:<cid>` (saved views) | `recents.ts`, `viewStore.ts` | the rail |

---

## 7. Behaviour

### 7.1 Component structure (files and contracts)

```
src/ui/Sidebar.tsx            keeps `export const Sidebar`, now a re-export of NavRail, so AppShell's import is unchanged.
                              Its header comment is rewritten: two views, why the tree is back (a module tree for the
                              online-ERP audience, additive, with keys), and why Quick access stays.
src/ui/nav/NavRail.tsx        the <nav>; owns useRailMemory() (the recordVisit / SIDEBAR_REFRESH / name-lookup effects
                              moved VERBATIM out of Sidebar so they keep running while the rail is hidden), the segmented
                              control, the view switch, the single onKeyDown for the whole rail.
src/ui/nav/QuickAccess.tsx    today's Sidebar body, moved verbatim (Today block, Pinned, Recent, empty state, foot).
src/ui/nav/MenuView.tsx       .nv-card: NavFind + NavTree + live region.
src/ui/nav/NavTree.tsx        rows, groups, roving tabindex, scroll-into-view; memoised rows.
src/ui/nav/NavFind.tsx        the filter input and its clear button.
src/ui/nav/NavChordHost.tsx   the one useFallbackKeys handler (section 5).
src/ui/navTree.ts             data + pure functions (section 6.1).      src/core/navPrefs.ts, src/state/navStore.ts
```

`NavRail` always renders the `<nav id="navrail" class="navrail">` element (the grid needs it, section 8) but renders **no children while hidden or with no company open**; its effects still run. `AppShell` still renders the empty `<nav class="navrail" aria-hidden>` while a gate screen (company passcode / operator sign-in) is on top.

DOM contract (class and attribute names are shared by CSS, tests and recipes; do not rename):

```html
<nav id="navrail" class="navrail" aria-label="Left menu" data-view="menu">
  <div class="nv-seg" role="group" aria-label="Left menu view">
    <button type="button" aria-pressed="true">Menu</button>
    <button type="button" aria-pressed="false">Quick access</button>
  </div>
  <div class="nv-card">
    <div class="nv-find">
      <svg class="nv-find-ic">…search…</svg>
      <input class="nv-find-in" placeholder="Filter menu" aria-label="Filter menu" spellcheck="false" autocomplete="off">
      <button class="nv-find-x" type="button" tabindex="-1" aria-label="Clear filter">…x…</button>   <!-- only with text -->
    </div>
    <div class="nv-tree" role="tree" aria-label="Menu">
      <div class="nv-g nv-pins has-on" role="none">                                   <!-- Pinned -->
        <div class="nv-row nv-gh" role="treeitem" aria-level="1" aria-expanded="true" tabindex="-1">…star… Pinned …chevron…</div>
        <div class="nv-kids" role="group"> <div class="nv-row nv-it" role="treeitem" aria-level="2" tabindex="-1"> New Sales …acts… </div> </div>
      </div>
      <div class="nv-g" role="none">                                                  <!-- Dashboard (leaf) -->
        <div class="nv-row nv-gh [on]" role="treeitem" aria-level="1" [aria-current="page"] tabindex="-1" data-nv-id="dashboard">…grid… Dashboard <kbd class="nv-k">W</kbd></div>
      </div>
      <div class="nv-g [has-on]" role="none">                                         <!-- a group -->
        <div class="nv-row nv-gh" role="treeitem" aria-level="1" aria-expanded="true|false" tabindex="-1" id="nv-sales">
          <svg class="nv-ic">…</svg><span class="nv-lbl">Sales</span><svg class="nv-chev">…</svg></div>
        <div class="nv-kids" role="group">                                            <!-- rendered only while open -->
          <div class="nv-row nv-it [on]" role="treeitem" aria-level="2" [aria-current="page"] tabindex="0|-1"
               data-nv-id="sales.invoice" title="Sales Invoice: press F8 on the Gateway and its menus" aria-keyshortcuts="F8" aria-label="Sales Invoice">
            <span class="nv-lbl">Sales Invoice</span><kbd class="nv-k">F8</kbd>
            <span class="nv-acts"><button class="nv-act" tabindex="-1">+</button><button class="nv-act nv-star [on]" tabindex="-1">★</button></span>
          </div>
          <div class="nv-row nv-it nv-more" role="treeitem" aria-level="2" tabindex="-1" data-nv-id="gst.more">Statutory reports…<svg class="nv-chev"/></div>
        </div>
      </div>
    </div>
    <div class="nv-sr" role="status" aria-live="polite">12 items</div>               <!-- only while filtering -->
  </div>
</nav>
```

Row behaviours: a click on a group header toggles it; a click on a row calls `go(item.to)`; the `+` calls `go(item.create)`; the star calls `togglePin` (and `setMem`, as Quick access does). `go(scr)` is `planNav(st.nav, scr)` from `favouritesCore.ts` (the policy Favourite shortcuts already use): `stay` (already there) does nothing; `home-push` is `st.goHome(); st.push(scr);` so Esc walks back to the Gateway, as from the menu tree; **`push`** (an entry form is anywhere in the stack: a half-keyed voucher, a master being altered, Bulk Entry) pushes ON TOP and leaves the stack alone, because `goHome()` would unmount the entry with no "Quit ? (entries will be lost)" that Esc asks. Quick access's rows and its last-voucher button go through the same `go()`. `title` on every row is the label, then a colon and the tooltip: `press F8 on the Gateway and its menus`, `press B on the Gateway`, `press F3 from anywhere` (the key through `keyLabel()`); a row with no key has `title` = its label.

### 7.2 How filter, pins and the current group interplay

* **Active row** = `activeRow(visibleGroups, top)` (section 6.1): one row, exact destination first, then the same screen with more parameters (a voucher being altered keeps Sales Invoice lit; a drilled Stock Summary keeps Stock Summary lit), then an item's `also` forms (the Ledger edit form lights Ledgers), then a group's footer submenu. A destination listed twice lights only the first row in tree order. The Gateway lights nothing. The row gets `.on` and `aria-current="page"`; its group gets `.has-on` (icon in accent).
* **Auto-expand:** when the top screen changes (keyed on `screenKey`, not the object) and its group is folded, that group opens **for the session only** (`autoOpen` state, never written to `TW_NAV_GROUPS`, so the tree does not slowly open itself), and the row scrolls into view (`scrollIntoView({block:'nearest'})`, no smooth scrolling). A manual collapse of that group is respected until the screen changes again.
* **Toggling** a group calls `toggleGroup(id, shownOpen)` with what was **visible** (stored state, auto-open or filter), then drops `autoOpen` for it, so clicking an auto-opened group closes it.
* **Filter text** is session-only and local to the Menu view; switching to Quick access clears it. While it is non-empty: only matching rows show (`filterTree`), every group with a match is open regardless of stored or auto state, **stored state is not modified**, Pinned shows only matching pins (a pin matches by its label), footer rows are hidden, the current row stays highlighted when it is a match, and clearing the filter restores exactly the previous open / closed state. The matched letters of a label are underlined (never bolder: Plex 600 is wider and would truncate); a row that matched only through an invisible alias shows it dim after the label ("Company Details · gstin") and gives up its keycap meanwhile; a match on the group's own name needs no note. A group header is drawn inert while a filter is typed (`aria-disabled`, default cursor, no hover wash, dimmed chevron). One live region, mounted for the life of the Menu view, announces "N items" / "No items" (empty when no filter).
* **Enter in the filter** opens `firstMatch(...)`: the first row on screen, top to bottom, Pinned first when it matches (a pin is the user's own best match). With no match, **or with nothing typed**, Enter does nothing (`enterTarget`: a habitual Enter after clicking into the box must not open the Dashboard). **Esc in the filter** clears it, blurs, and hands focus back to the screen (`refocus()`); with an empty filter Esc only hands focus back. Esc never pops the nav stack while focus is in the rail.
* **No results** shows: "No menu item matches "xyz". Press ⌘K to search everything." (the key through `keyLabel`; the key text is a small button that dispatches the existing `tw-goto` event, as Quick access's empty state does).
* **Pins:** pinning from a tree row calls `togglePin(cid, item.to)` and refreshes the rail memory; the pin appears in Pinned immediately. Unpinning from Pinned removes it there and clears the star in the tree. **Unpinning the pin that has keyboard focus (Space) hands focus to a neighbour that still exists** (`focusAfterUnpin`: the next pin, else the previous, else the tree row of the same destination, else the neighbour in the walk); otherwise focus falls to `<body>` and the next arrow key moves the Gateway behind the rail.
* **Switching company** re-reads pins and recents (`useRailMemory` keyed on `cid`); group open state is per device, not per company.

### 7.3 Keyboard model

The rail registers **no** global handler. One React `onKeyDown` on the `<nav>` handles keys **only while focus is inside the rail**, and stops propagation of exactly the keys below (React 18 delegates at `#root`, below `window`, so `stopPropagation()` keeps them off `installKeyboard`'s stack; proven in section 1, item 9). Everything not listed (F-keys, Ctrl / Alt chords, `?` on a tree row, Ctrl+K, Alt+G, Esc inside a field the rail does not own) propagates untouched, so every global chord works from inside the rail.

| Where focus is | Key | Does | Propagation |
|---|---|---|---|
| tree row | `↓` `↑` | next / previous **visible** row (group headers, rows and footer rows alike); `↑` on the first row goes to the filter box | stopped |
| tree row | `→` | closed group: open it; open group: first child; row: nothing | stopped |
| tree row | `←` | open group: close it; row (or closed group with a parent): focus its group header | stopped |
| tree row | `Enter` | row: open it (`go`), then blur and hand focus to the screen; group: toggle; footer row: open the submenu | stopped |
| tree row | `Home` `End` | first / last visible row | stopped |
| tree row | `PgUp` `PgDn` | 8 rows up / down | stopped |
| tree row | `+` (also numpad +) | open the create form, on rows that have one | stopped |
| tree row | `Space` | pin / unpin the row (where pinnable); no scroll | stopped |
| tree row | any bare printable character except `+` and `?`, and `Backspace` | moves focus to the filter box and types there (so the Gateway's `C` / `A` / `V` letters never run behind the rail) | stopped |
| tree row | `Esc` | clears the filter, blurs, hands focus to the screen | stopped |
| tree row | `Tab` / `Shift+Tab` | native: the tree is one tab stop (roving tabindex); the key never reaches a screen handler (the Security screen uses Tab) | stopped, default kept |
| filter box | printable characters, `Backspace`, `Delete`, `←` `→`, `Home` `End` | edit the text (native) | stopped |
| filter box | `Ctrl+A` `Ctrl+C` `Ctrl+V` `Ctrl+X` `Ctrl+Z` (and ⌘) | the field's own select-all, copy, paste, cut, undo (so Accept and the app's Undo do not fire behind it) | stopped, default kept |
| filter box | `↓` | focus the first visible row | stopped |
| filter box | `Enter` / `Esc` | open the first match / clear, blur and return | stopped |
| segmented control | `Enter` `Space` | native button | stopped, default kept |
| any rail control | `Tab` / `Shift+Tab` | native focus move | stopped, default kept |
| any rail control | `Esc` | clears the filter, blurs, returns focus to the screen | stopped |
| anywhere in the rail | `Ctrl+\` `F1`..`F12` `Ctrl+K` `Alt+G` `?`(tree) | their normal meaning | **propagates** |

The two `+` and `Space` row actions are advertised on the `?` sheet under **"In the left menu"** (section 11.3), a section listed once something is typed so the resting sheet stays short.

### 7.4 Focus rules (the trap this design must not fall into)

1. **Activation hands focus back.** Enter on a row that navigates (and a click that did take focus, for example on the filter box's clear button) calls `(document.activeElement as HTMLElement | null)?.blur()` and then pushes. Many screens (reports, menus) have no input to take focus; without the blur the focus stays on a rail row and the next arrow key moves the rail instead of the report's cursor. Screens that auto-focus an input (`useAutoFocus`) simply take it. **And whenever the top screen changes while focus is still inside the rail** (a mouse click on a Quick access button, a Tab-focused row and then F3 / F8 / Ctrl+K, anything that is not a rail row) `NavRail` blurs the active element in an effect keyed on the screen key; the rail is later in the DOM than the screens, so a screen that focuses its own input keeps it.
2. **A mouse click on a row or an action does not take focus:** `onMouseDown={(e) => e.preventDefault()}` on rows, `.nv-act`, the clear button, both toggles and the segmented buttons, so a voucher field the operator was typing in keeps its caret and its focus. Only the filter input takes focus on click. The Quick access buttons (`sb-item`, `sb-pin`, `sb-last`, `sb-cta`) get the same `onMouseDown` rule: a clicked button left focused handed the rail every bare key of the screen it had just opened. (The existing hamburger, which stays focused after a click and then re-toggles on Enter, gets the same fix.) **Enter and Space on the hamburger and the breadcrumb toggle** are stopped at the button (`ownActivate`, `nav/ownKeys.ts`; not `preventDefault`, so the native click still fires): the window key stack sees a keydown before the click and a screen that consumes Enter would otherwise run its own Enter (on the Gateway: open Create) and the toggle would never fire.
3. **Hiding the rail while focus is inside it** blurs the active element (`NavRail` does it when `hidden` flips true).
4. **Tab order**: the rail is the LAST child of `.main` (placed in grid column 1 / row 1 explicitly), so from the top of the page it is: "Skip to left menu", the practice-data banner's links, the top bar, the screen's own fields, the rail (segmented control, filter box, the tree's one stop), the breadcrumb toggle. With the rail ahead of the screen, a stray Shift+Tab from a voucher's first field landed on a rail row, which then owned Enter, the arrows, letters and the first Esc; now it goes to the top bar, as it does in an app with no rail. **"Skip to left menu"** (`NavSkip`, mounted first in `App.tsx`; `position: fixed`, clipped until focused) is the keyboard route in: it focuses the tree's tab stop (Quick access: its first button). It binds no key.
5. **Roving tabindex:** exactly one row has `tabindex="0"`: the keyboard "current" row, initialised to the active row if visible, else the first visible row; arrow keys set it and call `.focus()`; clicking a row sets it; filtering resets it to the first visible row.

### 7.5 Accessibility
`nav` landmark labelled "Left menu"; the tree is a real ARIA tree (`role="tree"`, `treeitem`, `group`, `aria-level` 1 and 2, `aria-expanded` on group headers only, `aria-current="page"` on the active row); **every treeitem is named by its own label** (`aria-label`: "Ledgers", not "Ledgers New: Ledgers"; a pin: the full "Ledger Vouchers: Name"), the `+` and star inside it keep their own names ("New: Ledgers"; the star by its ACTION, "Pin to the top" / "Unpin", with no `aria-pressed`: a flipping label plus a state read "Unpin, toggle button, pressed"); the tree is `aria-describedby` a visually hidden hint (arrows, Right / Left, Enter, Space pins, + creates, type to filter, Esc), because the `+` and star are `tabindex=-1` and a screen reader user could not otherwise learn they exist; `aria-keyshortcuts` is set only for F-key chords, in ARIA's spelling (`Control+F7`, `Meta+F7` on a Mac, `Alt+F6`), never for a bare Gateway letter (typed on a focused row, it goes to the filter); the `<kbd>` is `aria-hidden`; the clear button has a name ("Clear filter"); the rail is `aria-hidden` while it has no company to show; focus is a 2px accent outline inside the row (`:focus-visible`), never removed; no state is colour alone (wash + accent label + 2px bar for "here"; the chevron's rotation for open); `prefers-reduced-motion` kills the chevron and hover transitions (global rule); **forced colours** (`@media (forced-colors: active)` in `navtree.css`): the current row is `Highlight` / `HighlightText`, the selected segment and the hidden-menu cue keep a `Highlight` marker, the filter box gets a `CanvasText` border (dark has no light overlay to add one), and the transparent 1px row border is `Canvas` so it is not painted as a box around every row; the outline survives as before. **Target size:** the `+`, star and clear buttons and the breadcrumb toggle are at least 24 x 24 CSS px (WCAG 2.5.8).

---

## 8. Full-screen view

### 8.1 Root cause and the fix (precisely)

`styles.css` today:

```css
.app.navhid .main { grid-template-columns: 0 1fr 152px; }
.app.navhid .navrail { display: none; }
```

`.main` is a three-column grid whose items are the rail, the screen's `.content`, and the screen's `.btnpanel` (the wrapper around each screen is `display: contents`). `display: none` takes the rail **out of the grid**, auto-placement fills column 1 (0px) with the `.content` and column 2 (`1fr`) with the `.btnpanel`. **Replace both rules with:**

```css
/* The rail is hidden by collapsing its column, NOT by removing it from the grid: a display:none grid child makes
   every sibling re-place one column to the left (the screen fell into a 0px column and the F-key rail stretched
   across the page). visibility:hidden also takes it out of the tab order and the accessibility tree. The selector
   outranks the light scheme's `.navrail { padding-top: 4px }` ((0,4,0) over (0,3,1)), so nothing keeps the 0px
   column from being exactly 0px. */
.app.navhid .main { grid-template-columns: 0 1fr 152px; }
.app.navhid .main > .navrail { visibility: hidden; overflow: hidden; padding: 0; border: 0; }
```

**DOM order (added in the fix round):** the rail is the last child of `.main`, with `.navrail { grid-column: 1; grid-row: 1; }` in `styles.css`, so every screen's `.content` / `.btnpanel` still auto-place into columns 2 and 3 on the same row (the geometry probe and the flow check it on every screen). DOM order is Tab order (7.4, rule 4); the gated placeholder `<nav>` is last too.

The base `.main` rule stays **character for character** `grid-template-columns: 186px 1fr 152px;` (`visual.assert.ts` reads it by regex and the F-key rail's 152px is solved against it). The rail's 212px is a separate, later rule that only exists while the rail is shown (Appendix A): `.app.nv-wide:not(.navhid) .main { grid-template-columns: var(--nv-w) 1fr 152px; }`; `AppShell` now adds `nv-wide` in BOTH views. `:not(.navhid)` keeps it from tying with the hidden rule at (0,3,0).

Light scheme only, so the card gets the breathing room every other card has (the rail normally supplies the gap, and the screen card's left margin is 0): inside `@media screen` in `light/shell.css`, `html:root[data-scheme='light'] .app.navhid .main { padding-left: var(--frame-gap); }`. Dark has flush panes and needs nothing.

### 8.2 `AppShell.tsx`

* Delete the local `useState` / `localStorage` effect; read `collapsed` and `view` from `useNavStore`. Class: `'app' + (collapsed ? ' navhid' : '') + (view === 'menu' ? ' nv-wide' : '')`.
* `<TopBar onToggleNav={toggleCollapsed} navHidden={collapsed} />`; `<Sidebar hidden={collapsed} />`; mount `<NavChordHost />` next to `<FavouritesHost />`; `<StatusBar crumb={...} navHidden={collapsed} onToggleNav={toggleCollapsed} />`.
* The gated placeholder `<nav className="navrail" aria-hidden="true" />` keeps its `id="navrail"` so `aria-controls` resolves.

### 8.3 The toggles

* **Status bar** (`widgets.tsx`, `StatusBar`): the **first child after `<StatusLane />`**, before `.crumb`:
  `<button type="button" className="crumb-nav" aria-controls="navrail" data-hidden={navHidden ? '1' : undefined} title={(navHidden ? 'Show menu' : 'Hide menu: full screen') + ' (' + keyLabel('Ctrl+\\') + ')'} onMouseDown={(e) => e.preventDefault()} onKeyDown={ownActivate} onClick={onToggleNav}><Icon n="sidebar" size={14} /><span className="crumb-nav-t">{navHidden ? 'Show menu' : 'Hide menu'}</span></button>` The words are the visible label (the owner's headline control must be findable without hovering) and the accessible name; they flip with the state, so the button carries **no `aria-expanded`** (a flipping name plus a state reads as a contradiction). The hamburger keeps its stable name and `aria-expanded`. Both stop Enter / Space at the button (`ownActivate`).
  Icon `sidebar` (a panel with a left pane). When the menu is hidden the button turns accent (`[aria-expanded='false']`): the way back is the one lit thing in the bar. The breadcrumb text is **not** made clickable (section 3, decision 6).
* **Top bar** (`TopBar`): the hamburger stays, `title="Show / hide the left menu (⌘\)"`, `aria-controls="navrail"`, `aria-expanded={!navHidden}`, `onMouseDown` preventDefault.
* **Chord:** section 5.

### 8.4 What "full screen" means, per surface (acceptance, section 13.3)

| Surface | Menu hidden |
|---|---|
| Any screen with `.content` + `.btnpanel` | `.content` starts at x = 0 (light: `--frame-gap`) and ends at `viewport - 152`; `.btnpanel` is 152px wide at `viewport - 152` |
| Gateway, Create / Alter, other menus | the two cards keep their place in the wider column |
| Dashboard (a single `.dash` child) | fills columns 1 and 2; the right 152px column is empty exactly as when the rail is shown |
| Voucher, master forms, reports, Print Configuration (a wide two-pane screen), popups and overlays | unchanged behaviour; `position: fixed` overlays are unaffected |
| Print / export | `.app` is `display: none` under `@media print`; unaffected |
| Comfortable density | same rule; the 8 / 12px frame gap is the light `padding-left` above |
| Dark | the same grid; no padding |

---

## 9. Rights and feature gating

`visibleTree(NAV_TREE, featuresOf(company), accessCtxOf(state))` is the only gate (section 6.1), evaluated in a `useMemo` keyed on the company, users, roles and `useSession` operators (as `MenuScreen` keys it):

1. `item.feature` switched off in the company: the row is not drawn. The flags are copied from `MenuScreen` for every destination that appears there, and from the palette (`orders` for Sales / Purchase Order, `inventory` for Delivery / Receipt Note) where the Gateway reaches the voucher only through F10; the guard (section 11) compares them.
2. `screenDenial(ctx, item.to) !== null`: the row is not drawn. With security control off `screenDenial` is always `null`. A `+` whose create form is denied is dropped and the row stays (a role with `reports` but not `masters` cannot reach a master list either, so in practice both go).
3. A group with no row left disappears (a Data Entry role does not see an empty Masters or GST).
4. Footer links only for full-rights operators or security off.
5. Pinned and Saved views are filtered by the same `screenDenial`; a pinned saved view whose view was deleted is dropped (Quick access's rule).
6. With security on and nobody signed in, `AppShell` is showing the sign-in gate and renders the empty rail; no tree is drawn.
7. With no company open the rail draws only its frame. (A new screen would need `SCREEN_AREA` or it is administrator-only and invisible to every restricted role; that is the existing fail-closed rule, and the guard fails on an unlisted route.)

---
## 10. Visual spec

Light and dark, compact and comfortable, **tokens only**. The rail is a white raised card on the canvas in light (the screen card's and the Gateway cards' language: `--panel`, `1px --border-lt`, `--radius-lg`, `--shadow-1`); in dark it is a `--panel` card on the dark chrome. The reference CSS (Appendix A) was injected into the live page and looked at in all four combinations; it is the starting file, not a sketch.

### 10.1 Tokens used (none is new except the five `--nv-*`, all defined in `:root` of `navtree.css`)

| Part | Tokens |
|---|---|
| Card | `--panel`, `--border-lt`, `--radius-lg`; light only: `--shadow-1`, and `margin-bottom: calc(var(--frame-gap) - var(--sp-2))` so its bottom edge lines up with the screen card in comfortable (12px gap) as it does in compact (8px) |
| Segmented control | track `--sunk` + `--border-lt`, pill `--panel` + `--border-lt` + `--shadow-1`, text `--dim` / `--ink` 600, `--fs-sm` |
| Filter box | `--field`, ring `inset 0 0 0 1px var(--field-line)` (1px, costs no height; 3.27:1 light / 3.9:1 dark on the card, which is what the field boundary needs and `--border` cannot give), focus ring `--accent` + `--focus-ring`, `--fs-sm`, placeholder `--dim` |
| Group header | `--ink` at weight 600, icon `--dim` (accent when it holds the current screen; Pinned always accent), chevron `--dim` |
| Row | `--ink`, `--nv-fs` 13px, padding `var(--row-y) var(--nv-pad)`, hover `--hover`, radius `--radius`, hairline guide `--border-lt` |
| You are here | background `--sel`, label + icon `--accent`, `box-shadow: inset 2px 0 0 var(--accent)`, **weight unchanged** |
| Keycap | the shared `kbd` recipe (`--fs-key`, mono, `--kbd-*`), softened here: transparent face, `--border-lt` edge, `--dim` ink (5.84 / 6.71 on `--panel`) |
| Footer link | `--dim`, `--ink` on hover, accent when current |
| Breadcrumb toggle | `--dim` -> `--ink` on hover with `--sunk` (light: `--hover-canvas`, the canvas hover), accent while the menu is hidden; icon + the words "Hide menu" / "Show menu", `min-height: 24px` |
| New, `:root` | `--nv-w: 212px` `--nv-fs: 13px` `--nv-pad: 5px` `--nv-ic: 16px` `--nv-gap: 7px`; added in the fix round: `--nv-act: 24px` (a `+` / star / clear target) and `--nv-row-h` (a row's height, which is also the sticky header's and the scroll padding) |
| Filter match | underline in `--accent`, 1.5px, never a weight change; the alias note `--dim` at `--fs-xs` |
| Dark selected segment | `--sel` wash + `--border` edge (the `--panel` pill was 1.07:1 from its `--sunk` track), the one scheme branch in the file, still tokens |

No hex anywhere in `navtree.css` (lint in Appendix B). Group headers are **sentence case at 600 weight**, 13px, not uppercase and not tracked; the light overlay's rule that turns every section label into a small `--dim` 11px sentence-case label is **not** applied to them (they are rows, not eyebrows), and the new classes are in no list of that overlay.

### 10.2 Metrics and the width arithmetic (so a reviewer can re-derive 212)

Space for a row label = **`--nv-w` minus 68**, where 68 = rail padding 14 (7 + 7) + card border 2 + tree padding 6 (3 + 3) + scrollbar 11 (drawn at 11px on Windows the moment the tree scrolls; macOS overlay scrollbars take none, so this is the worst case) + row border 2 + left inset 28 (5 padding + 16 icon + 7 gap, so a child's label starts under its group's label) + right padding 5. At 212 that is **144px**. Measured widest label **142px** ("Reverse Charge Register"); the widest **keyed** row is Physical Stock: label 84 + gap 4 + `Ctrl+F7` cap 54 = 142. Margin: 2px. Do not shrink `--nv-w` or widen a label without re-running the fit assertion (section 11.2) and probe B2.

Heights follow the density axis: compact rows `min-height: 24px` (`--row-y: 2px`, 13px at `--lh` 1.25), comfortable rows about 33px (`--row-y: 6px`). The compact pitch of every table is untouched (the rail has its own rows). Segmented control 22px buttons in a 2px track; filter box 22px. Icon box 16px (the `Icon` default), strokes 1.6, `currentColor`; chevron 12px; action icons 14px.

Widths: shown **212 in both views** (the `.main` base rule still reads 186 and applies only to a device that has not added `nv-wide`, i.e. never in the shell; `visual.assert` reads it), hidden **0**. At 1440px the screen column is 1076 (shown) / 1288px (hidden).

### 10.3 States

| State | Look |
|---|---|
| Rest | transparent row, `--ink` label, `--dim` icon and keycap |
| Hover | `--hover` wash; the keycap hides and the `+` / star appear in its place (so the row never changes width); the action strip has **no background of its own** (a grey tile behind the star of a keyboard-focused row, which has no wash) and the label **reserves the strip's width** (`padding-right`: 26px for a star, 50px for `+` and star; none on a keyed row, whose hidden keycap already holds the room), so a long label ends in an ellipsis beside the buttons instead of being cut mid-letter under them |
| Keyboard focus | 2px `--accent` outline **inside** the row (`outline-offset: -2px`) |
| Current screen | `--sel` wash, `--accent` label and icon, 2px accent bar; `aria-current="page"`; the group's icon is accent |
| Open / closed group | chevron rotated 90 degrees / not (`aria-expanded`); children are not rendered while closed; the header is **sticky** (`position: sticky; top: 0`, opaque `--panel`) so the rows on screen never lose their group name; the tree has no top padding (a sticky header would show rows through the gap) and `scroll-padding-top` keeps a keyboard-focused row clear of it |
| Filtering | groups with a match open, others vanish, footer rows hidden, filter box accent ring while focused |
| No match | the quiet text block of section 7.2 |
| Pin disabled (list full) | star at 45% with the title "Pinned list is full (8)" |
| Hidden | the rail's column is 0px, `visibility: hidden` (section 8) |

### 10.4 Icons (append to `src/ui/icons.tsx`, same 24 grid and 1.6 stroke, `currentColor`)

`IconName` gains `'grid' | 'bag' | 'funnel' | 'chevron' | 'sidebar' | 'x'`:

```tsx
grid:    <><rect x="3.5" y="3.5" width="7" height="7" rx="1.5" /><rect x="13.5" y="3.5" width="7" height="7" rx="1.5" /><rect x="3.5" y="13.5" width="7" height="7" rx="1.5" /><rect x="13.5" y="13.5" width="7" height="7" rx="1.5" /></>,
bag:     <><path d="M5 8h14l-1 12H6z" /><path d="M9 8V6.5a3 3 0 0 1 6 0V8" /></>,
funnel:  <><path d="M3.5 5h17l-6.5 7.5V19l-4 2v-8.5z" /></>,
chevron: <><path d="m9 6 6 6-6 6" /></>,
sidebar: <><rect x="3" y="4.5" width="18" height="15" rx="2" /><path d="M9 4.5v15" /></>,
x:       <><path d="M6 6l12 12M18 6 6 18" /></>,
```

Rendered at 64px and 16px next to `invoice`, `bank`, `register` and `tree`: same weight and optical size. Everything else reuses the existing set. Group icons: Dashboard `grid`, Sales `invoice`, Purchases `bag`, Banking & Cash `bank`, Accounting `register`, Inventory `box`, GST `percent`, TDS & TCS `doc` (not `percent` again: the Gateway already separates the two for the same reason), Reports `trend`, Saved views `rows`, Masters `tree`, Payroll & HR `people`, CRM `funnel`, Data & Utilities `plug`, Settings `gear`, Pinned `star`, filter `search`, clear `x`, the status-bar toggle `sidebar`, the hamburger `menu` (unchanged), `+` `plus`. (The existing `scales` icon reads as a plant at 16px, which is why Accounting uses `register`.)

### 10.5 Where the rules live

* `src/styles/navtree.css` (new; imported by `NavRail.tsx`, as `Sidebar.tsx` imports `sidebar.css`): everything in Appendix A except the light block. Base rules, all schemes.
* `src/styles/light/shell.css`: the four light-only rules of Appendix A appended inside its existing `@media screen { }`, every selector starting `html:root[data-scheme='light']`. They do only what a token cannot: the card's shadow and comfortable margin, the keycap's flat shadow (the shared `kbd` light rule would otherwise out-rank `.nv-k`), the canvas hover on the breadcrumb toggle, and the hidden-state gap.
* `src/styles.css`: **only** the two `navhid` lines of section 8.1. `sidebar.css`, `.sb-*`, `.navrail .nitem` and every class in `visual.baseline.json` keep their rules (the suite fails if one is deleted).

---

## 11. Assertions, drift guards and recipes

### 11.1 New suite `scripts/assert/navtree.assert.ts` (pure; `npm run assert -- navtree`)

It imports `NAV_TREE`, `visibleTree`, `filterTree`, `activeRow`, `firstMatch` (`ui/navTree`), `readPrefs`, `isNavCollapseChord`, `NAV_STORAGE` (`core/navPrefs`), `screenDenial` / `SCREEN_AREA` (`core/access`), `VCH_BASES`, `DEFAULT_FEATURES` (`types`), and reads source text for the rest (the pattern `security-access.assert.ts` uses for the `Screen` union). Required checks:

1. **Order and shape.** `NAV_TREE.map(g => g.id)` equals the published order, written out in the test: `dashboard, sales, purchases, banking, accounting, inventory, gst, tds, reports, masters, payroll, crm, data, settings`. The Dashboard group is a leaf with exactly 1 row; every other group has **4 to 9** rows; **no duplicate label inside a group**; every `id` unique; every label <= 26 characters; aliases lower case; the only destination appearing twice is `inventory.items` / `masters.items` (an explicit allow-list, so a third duplicate fails).
2. **Every route is real.** For every `to`, `create`, `also` and footer `to`: `to.s` is in the `Screen` union (parse `store.ts` between `export type Screen =` and `const ACTIVE_CID_KEY`), has a `case '<s>':` in `router.tsx`'s `renderScreen`, and has an entry in `SCREEN_AREA`. A `menu` footer's `id` is a key of `MENUS` (parse `MenuScreen.tsx`).
3. **No stub.** A source scan finds every file containing `<ComingSoon`; the set equals the allow-list `{ ui/inventory/BatchSummary.tsx }`; the screens those files render (`batchSummary`) do not appear in `NAV_TREE`. A new stub fails the first half; a stub that gets implemented fails the allow-list, which is the reminder to add the Batches & Expiry row.
4. **The rail never over-promises.** For `keyWhere: 'menus'` the key equals `VCH_BASES.find(b => b.base === to.base)?.fkey`; for `'gateway'` the Gateway block of `MENUS` has a row with the same `to.s` and `hk === key` (parse the `gateway:` block of `MenuScreen.tsx` with `/\{ label: '([^']+)', hk: '(\w)', to: \{ s: '(\w+)'/g`); for `'anywhere'` the key is handled in `App.tsx`'s global handler (`c === 'F3'`, `'F11'`). `key` and `keyWhere` appear together or not at all.
5. **The rail never offers what the Gateway hides.** Every `feature` is a key of `DEFAULT_FEATURES`; for every row whose `to` appears in a `MENUS` row (same `s` and same parameters) the item's `feature` equals the **effective** Gateway feature: that row's own `feature`, else the feature on the Gateway row that opens its submenu (Inventory Books and Statements of Inventory: `inventory`; Payroll Reports and HR & Leave: `payroll`). (Rows that exist only in the palette, `orders` and Delivery / Receipt Note, are checked against `CommandPalette.tsx`'s `feats.orders` / `feats.inventory` conditions.)
6. **Gating** (the cases of section 9): security off; each of `inventory`, `gst`, `tds`, `tcs`, `billwise`, `orders`, `payroll`, `costCentres` off in turn (the group disappears when it empties, rows disappear when it does not); a role with only `accounting`, only `reports`, only `masters`, `full`; signed out. Group removal, `+` removal, footer links only for full / security-off.
7. **Filter:** empty keeps all; case-insensitive; alias hit (`receipt` finds Payment Received, `contra`, `brs`, `r1`, `grn`, `vendor credit`); word-AND (`sales reg`); a module name lists the module (`tds`); no match returns `[]`; footer rows never match; `firstMatch` is top to bottom.
8. **Active row:** exact; altering a voucher; drilled Stock Summary; `pickMaster` with and without `goal: 'book'`; an edit form via `also`; the duplicate destination lights the Inventory row; a footer submenu; Gateway and an unlisted drill-down light nothing.
9. **Persistence:** `readPrefs` defaults, the legacy `TW_NAV_COLLAPSED` key, garbage view, unknown group ids dropped, corrupt JSON, `[]` respected.
10. **The chord:** `isNavCollapseChord` accepts Windows `Ctrl+\`, Mac `⌘\`, the `U+001C` fallback; rejects AltGr, `Ctrl+Shift+\`, bare `\`, `Alt+\`, `Ctrl+B`. **And it is free**: the `favourites.assert.ts` scan, aimed at `'Ctrl+\\'`, `e.key === '\\'` and `'Backslash'` in every `src/` file outside `keys.ts`, `core/navPrefs.ts`, `ui/nav/NavChordHost.tsx`, `ui/overlays.tsx` (the sheet rows) and `ui/favouritesCore.ts` (`LOCKED_KEYS`) finds nothing; `BROWSER_RESERVED` does not contain it; it **is** listed in `GLOBAL_KEYS` and `LOCKED_KEYS`.
11. **Brand:** no `label` matches `/tally/i` except the allow-list `['Compare with Tally Data']`; aliases may.
12. **CSS contracts:** `.main` base rule still reads `grid-template-columns: 186px 1fr 152px`; no stylesheet in `src/styles*` sets `display: none` on `.navrail` (the regression test for the bug); `.app.navhid .main > .navrail` declares `visibility: hidden`; `navtree.css` contains no `#hex` outside comments and no `display: none` on a grid item.

### 11.2 Extend `scripts/assert/visual.assert.ts` (extend, never relax)

* `PAIRS`: add `['dim', 'sel', 4.5, 'keycap on the current menu row']` (dark 4.69, so it passes; light is already in `LIGHT_PAIRS`, move it, do not keep two). All the other pairs the rail uses are already there (`ink` / `panel`, `accent` / `sel`, `dim` / `panel`, `dim` / `hover`, `accent` / `accent-soft/panel`).
* **Label fit**, in the same style as the F-key rail block: import `NAV_TREE`, read `--nv-w`, `--nv-pad`, `--nv-ic`, `--nv-gap` from `navtree.css`, take Plex Sans 400 advances at 13px from the shipped woff (the helper already there), and assert for every row `width(label) + (key ? 4 + capWidth(Windows spelling) : 0) <= nvW - 68`, and for every footer label `width + 4 + 12 <= nvW - 68`.
* The `.main` base rule still matches the existing regex (152px read from it); the wide rule `.app.nv-wide:not(.navhid) .main` reads `var(--nv-w) 1fr 152px`; `--nv-w` is 212px.

### 11.3 The `?` key sheet (`src/ui/overlays.tsx`)

```ts
// GLOBAL_KEYS ("Everywhere")
{ k: ['Ctrl+\\'], d: 'Hide or show the left menu: full-screen view', find: 'sidebar navigation menu rail collapse expand hide show full screen' },

// new constant, listed once something is typed (search: true), so the resting sheet stays short
const NAV_KEYS: KeyRow[] = [
  { k: ['ArrowUp', 'ArrowDown'], d: 'Left menu: move between rows' },
  { k: ['ArrowRight', 'ArrowLeft'], d: 'Left menu: open / close a group; ← on a row goes to its group' },
  { k: ['Enter'], d: 'Left menu: open the row (a group: open / close it)' },
  { k: ['Home', 'End'], d: 'Left menu: first / last row' },
  { k: ['+'], d: 'Left menu: new, on rows that have a + button (Ledgers, Groups, Stock Items...)' },
  { k: ['Space'], d: 'Left menu: pin or unpin the row' },
  { k: ['A…Z'], d: 'Left menu: type to filter the menu', find: 'filter search menu' },
  { k: ['Escape'], d: 'Left menu: clear the filter and return to the screen' },
];
// buildSections: after the 'Go anywhere' section
{ title: 'In the left menu', rows: NAV_KEYS, search: true },
```

`favouritesCore.ts` `LOCKED_KEYS`, group "Go anywhere": `{ keys: ['Ctrl+\\'], what: 'Hide or show the left menu' }`.

### 11.4 Screenshot recipes (`scripts/shots/recipes/left-nav.mjs`, so no shared table is edited)

Names (each asserts its crumb or a DOM probe before shooting; run in all four scheme x density combinations):
`nav-menu` (Gateway, default view), `nav-menu-active` (Sales Register: Sales open, row lit, Pinned shows one pin made in the recipe), `nav-filter` (click the box, type `gst`), `nav-filter-alias` (`receipt`), `nav-filter-empty` (`zzz`), `nav-quick` (click Quick access), `nav-hidden-gateway`, `nav-hidden-daybook`, `nav-hidden-voucher` (F8, its ledger popup open), `nav-hidden-dashboard`, `nav-hidden-gstr1` (the widest report), each hidden one after `p.key('Ctrl+\\')`. The hidden recipes also eval the geometry probe B1 and fail on a mismatch, so the bug cannot come back un-noticed.

---

## 12. Build plan (disjoint files, four agents; none runs `git add`/`commit`)

| Agent | Owns (creates / edits) | Needs from others |
|---|---|---|
| **N1 data and logic** | `src/ui/navTree.ts`, `src/core/navPrefs.ts`, `src/state/navStore.ts`, `scripts/assert/navtree.assert.ts` | the six icon names (N3) for `tsc` |
| **N2 UI** | `src/ui/Sidebar.tsx`, `src/ui/nav/*` (NavRail, QuickAccess, MenuView, NavTree, NavFind), `src/ui/AppShell.tsx`, `src/ui/widgets.tsx` (TopBar tooltip + aria, `StatusBar` toggle) | N1's API; N3's class names |
| **N3 CSS and icons** | `src/styles/navtree.css` (new), `src/styles.css` (the two `navhid` lines and nothing else), `src/styles/light/shell.css` (append), `src/ui/icons.tsx` (+6), `scripts/assert/visual.assert.ts` (+ its `visual.baseline.json` untouched) | none |
| **N4 keys, sheet, recipe** | `src/ui/nav/NavChordHost.tsx`, `src/ui/overlays.tsx` (sheet rows), `src/ui/favouritesCore.ts` (`LOCKED_KEYS`), `scripts/shots/recipes/left-nav.mjs` | N2 mounts `<NavChordHost />` (one line) |

Order: N3 and N1 start together; N4 starts together; N2 last (needs both). Then three adversarial reviewers (keyboard / gating / visual) work from section 13. The `.sb-*` rules, `Today` logic, `recents.ts`, `MenuScreen.tsx`, `router.tsx`, `store.ts` and `access.ts` are **not edited** (no screen is added, so the new-screen checklist does not apply).

---

## 13. Acceptance checklist (what reviewers run)

Stop at the first red. Screens: Gateway, Day Book, Trial Balance, GSTR-1, Sales voucher (F8), Ledger create, Dashboard, Bank Reconciliation, Print Configuration.

### 13.1 Hard gates
- [ ] `npx tsc --noEmit` 0 errors; `npm run build` passes; `npm run assert` green (all existing suites plus `navtree`; `visual` includes the new pair and the fit check).
- [ ] `git diff --stat` touches only the files of section 12 plus the doc updates; **no** change to `MenuScreen.tsx`, `router.tsx`, `store.ts`, `access.ts`, `keys.ts` combos, `btnpanel` rules.
- [ ] Lint (Appendix B5): no hex in `navtree.css`; every `var(--x)` defined; no `display: none` on `.navrail`; `visual.baseline.json` satisfied.

### 13.2 Frozen things are frozen
- [ ] Every Gateway letter, F-key, Esc behaviour and field order behaves as on `main` (spot-check: Gateway `C` `L`, `F8`, `Alt+G`, `?`, Esc out of a voucher).
- [ ] `.btnpanel` is exactly 152px and its keys and order are unchanged on every screen; Quick access view is identical to today except the segmented control above it and its column being 212px wide like the Menu's (so a view switch never re-flows the screen).
- [ ] Printing (Ctrl+P on a report, a voucher print) is unchanged.

### 13.3 Full-screen view
- [ ] Probe B1 on every screen listed, light and dark, compact and comfortable, at 1440x900 and 1280x720: `.content` begins at 0 (light: `--frame-gap`) and `.content + 152 = viewport`; `.btnpanel` is 152px at `viewport - 152`; no horizontal page scroll; the Dashboard fills its columns.
- [ ] The breadcrumb toggle is the first thing in the status bar, reads "Hide menu" / "Show menu" in words, and answers Enter and Space; tooltip "Hide menu: full screen (⌘\)" / "Show menu (⌘\)" (Windows: `Ctrl+\`); it turns accent while hidden; the hamburger does the same job; the chord does the same job; all three keep one state.
- [ ] State survives reload and is shared: hide, reload: still hidden; show via any of the three, reload: shown. Storage blocked: works for the session, no crash.
- [ ] While hidden, `Tab` never lands in the rail; shown again, the tree is one tab stop.
- [ ] The chord works in a browser tab on Chrome, Edge, Firefox (Windows / Linux) and Chrome, Safari, Firefox (Mac), and in the Electron app, **with focus in a voucher field**; none shows browser UI; held down it toggles once; with the key sheet or Go To open it does nothing. Esc behaves exactly as before on every screen (the breadcrumb is not clickable).

### 13.4 Keyboard and focus
- [ ] Tab from the page start reaches "Skip to left menu" first; Shift+Tab from a voucher's first field never lands in the rail; the rail's stops are: segmented control, filter box, the tree (one stop). Arrows, Home / End, PgUp / PgDn, `→` `←`, Enter, `+`, Space, type-to-filter, Esc all as the table in 7.3.
- [ ] With focus on a tree row on the Gateway, pressing `C`, `A`, `V`, `Enter`, `↓` does **not** move or fire the Gateway (letters go to the filter box); F8 still opens Sales; Alt+G still opens Go To; `?` opens the sheet.
- [ ] With focus in the filter box on a voucher screen, `Ctrl+A` selects the text and does **not** accept the voucher; `Ctrl+Z` undoes typing and does not undo a save; typing never reaches the voucher's fields.
- [ ] After Enter or a click on a row that navigates, the rail is blurred: on a report, `↓` moves the report's cursor, not the rail.
- [ ] A mouse click on a row, a star, a `+` or a toggle does not move the caret out of a voucher field the operator was typing in.
- [ ] Probe B3: exactly one `[tabindex="0"]` inside `.nv-tree` at any time.

### 13.5 The menu itself
- [ ] **All 108 rows** open the screen in section 4.2 (a script walks the tree and clicks each; crumb check), and all 8 footer rows open their Gateway submenu; no row opens `ComingSoon`.
- [ ] Group order equals 4.1; each group lists its rows in 4.2's order; Pinned first when non-empty; Saved views between Reports and Masters when the company has views.
- [ ] The current row is highlighted and its group opened when navigating by Gateway, Go To, the F-keys or the rail; altering a voucher keeps its row lit; the Gateway lights nothing; a manual collapse of the current group sticks until the screen changes; auto-opening is not stored (inspect `TW_NAV_GROUPS`).
- [ ] Filter: `receipt` shows Payment Received, `inventory` lists the whole module, `sales reg` finds Sales Register, `zzz` shows the empty text; Enter opens the first match; Esc clears and returns; clearing restores the previous open / closed groups exactly.
- [ ] Keycaps appear on exactly the 19 keyed rows, in the platform's spelling (`⌘F7` on a Mac); each tooltip says where it works; none appears on a row whose key would not fire where it says.
- [ ] `+` on Ledgers, Groups, Stock Items, Voucher Types, Units, Godowns, Cost Centres, Employees opens the create form; stars appear only on recordable rows; pinning works and appears in Pinned at once; at 8 pins the star is disabled with the title.

### 13.6 Look (four combinations: light / dark x compact / comfortable)
- [ ] Probe B2: **0 truncated labels** with all groups open, on a Windows-style classic scrollbar, compact and comfortable.
- [ ] The rail card's top and bottom edges line up with the screen card (light, both densities); no layout shift when hovering a row; the filter ring, segmented control and keycaps match the report filter and the F-key rail's keycaps.
- [ ] Focus is visible and >= 3:1 on: segmented buttons, filter box, a row, a star, the breadcrumb toggle. The current row is distinguishable in a grayscale screenshot (bar + wash + label colour).
- [ ] Group headers are 600 weight, sentence case, not tracked or uppercase; labels 13px in both densities.
- [ ] Reduced motion: no transitions; forced colours: the current row, the selected segment and the filter box are visible (`nav-forced-colors`, both schemes); the `+`, star, clear and breadcrumb buttons are at least 24px.

### 13.7 Rights and features
- [ ] Features off (`inventory`, `gst`, `tds` + `tcs`, `billwise`, `orders`, `payroll`, `costCentres`): the matching rows and emptied groups vanish; the Gateway agrees.
- [ ] Security on: a role with only Accounting rights sees no GST, Masters, Payroll or Settings rows except Switch Company; a Reports-only role sees Reports and no entry rows; footer links absent for restricted roles; signed out shows the sign-in gate and an empty rail.

### 13.8 Docs
- [ ] `CLAUDE.md`, `HANDOFF.md`, the light spec (three places) and the Sidebar header comment describe the two views and the 212 / 0 widths (212 in both views).

---

## 14. Risks, open decisions, out of scope

### 14.1 Risks
1. **Width 212 vs 186.** Measured, not guessed (1.7, 10.2), but it is 26px from every screen's content (now in both views). One known casualty: a five-period Cost Centre Analysis (and the four reports that already overflowed) scrolls sideways; its header text is `nowrap`, so only a narrower rail or a shorter header would fix it. If the owner vetoes any width change, `--nv-w: 186px` leaves 118px per row and 26 of the 108 rows truncate (full text stays in each row's tooltip); shortening them to fit costs more clarity than 26px costs space. A softer option is to default new devices to hidden below 1200px viewport width. Neither is recommended.
2. **`Ctrl+\` on AltGr layouts.** Unavailable by design (AltGr is Ctrl+Alt, a character being typed); the two toggles remain. If a fast chord is wanted there too, a second spelling can be added to `isNavCollapseChord` later without touching anything else.
3. **`e.key` for Ctrl+\ is not identical on every OS / browser.** The matcher has a `code` fallback; the acceptance list requires a real press on each browser and the desktop app.
4. **Vocabulary choice.** "Payment Received", "Bills" style names are the owner's Zoho-leaning call (decision 8). If the owner prefers Tally names on the rows, only `label` changes; aliases already cover both.
5. **Stock Items twice.** Deliberate; the active highlight and auto-open use the first. If it confuses, drop the Masters row (one line).
6. **`pickMaster` rows cannot be pinned** (`recordable` rule). Pinning a master list would need a new `recents.ts` rule; out of scope.
7. **Today count is not shown in Menu view.** It is one click away in Quick access. A one-line Today under the segmented control is an easy later addition.
8. **Saved views group** shows 6; more via Go To "View: name".
9. **A new feature flag or screen** that is not added to `navTree.ts` is simply absent from the rail: nothing breaks, the guard only protects rows that exist.

### 14.2 Out of scope (do not do)
Clickable breadcrumb segments; a second collapse chord; resizing the rail by dragging; icons on item rows; a mini (icons-only) collapsed rail; a rail on small phones; changing the Gateway, any shortcut, any field order; copying the Marg palette; touching `docs/parity-data.json` parity numbers (the module menu is a `beyond` row, never parity); the Quick access view's logic.

### 14.3 GOTCHAs to record in `CLAUDE.md` when it ships
* **A `display: none` grid child re-places its siblings.** Collapse the track and keep the item (`visibility: hidden; padding: 0; border: 0`); never remove a column's occupant. The F-key rail landed at the far left of the page because of this (found by a theme reviewer, present in light and dark).
* **A non-modal focusable region must stop every key it uses and its stray letters, then hand focus back after activation.** Otherwise the Gateway's single letters and Enter run behind it, and an arrow key keeps moving a rail the user has already left.
* **A width budget must include the scrollbar** (11px on Windows once a list scrolls); the fit is asserted with the real font advances (`visual.assert.ts`), not by eye.
* **No menu row may open a `ComingSoon` stub**; the `navtree` suite keeps an allow-list of stub files.
* **DOM order is Tab order, and the key stack sees Enter before a button does.** A region that owns bare keys (this rail) must come after the screen in the DOM, or a stray Shift+Tab lands in it and it silently takes the screen's keys. A real `<button>` outside the rail needs `ownActivate` (stop Enter / Space, never `preventDefault`) or a screen that consumes Enter runs its own Enter and the button never fires.
* **Anything that navigates for a click must be as safe as Esc.** `goHome()` + `push()` drops a half-typed entry with no prompt; use `planNav` (push on top when an entry form is in the stack), as Favourite shortcuts and Go To do.
* **A control that takes focus on a mouse click must not be able to keep it.** The rail owns every bare key while it has focus, so a clicked Quick access button left focused took the keyboard from the screen it had just opened. `onMouseDown` preventDefault on every rail control, and blur on any screen change.
* **A sticky header sticks at the scroller's PADDING edge** (give the scroller no top padding), and its background and every wash drawn over it must be opaque.
* **Forced colours drops backgrounds, shadows and accent text:** every state drawn that way (a "you are here" wash, a selected pill) needs a `Highlight` / `HighlightText` twin, at a specificity that beats the scheme's own override.
* **A treeitem is named by its content**, which folds nested buttons into its name: set `aria-label` on the row, give a flipping action label no `aria-pressed`, and describe the keys the nested `tabindex=-1` controls need.

---

## Appendix A: reference CSS (injected into the live page and looked at: light and dark, compact and comfortable)

### A.1 `src/styles/navtree.css` (new)

> The shipped file supersedes this reference CSS (the fix round of Appendix D added the sticky header, the 24px targets, the strip's reserved width, the filter marks, the skip link, the dark segment and the forced-colours block). Read `src/styles/navtree.css`; this listing is the starting point it grew from.

```css
/* ─── Left menu: the module tree (Menu view) ─────────────────────────────────
   src/styles/navtree.css. Token-only: no hex, no scheme branch. Every colour,
   radius and rhythm below is an existing token, so light, dark, compact and
   comfortable all follow without this file knowing which is on. */

:root {
  --nv-w: 212px;      /* the rail while the Menu view is showing (Quick access stays 186px) */
  --nv-fs: 13px;      /* row labels: 13px in BOTH densities; comfortable gets air from --row-y, not a bigger face */
  --nv-pad: 5px;      /* row side padding */
  --nv-ic: 16px;      /* icon box */
  --nv-gap: 7px;      /* icon-to-label gap */
}

/* The wide variant only exists while the rail is shown. :not(.navhid) is what keeps this rule from tying
   with `.app.navhid .main` (both are (0,3,0) otherwise, and source order would decide). */
.app.nv-wide:not(.navhid) .main { grid-template-columns: var(--nv-w) 1fr 152px; }

/* ── segmented control: Menu | Quick access ────────────────────────────── */
.nv-seg {
  flex: none; margin-bottom: 6px; display: flex; gap: 2px; padding: 2px;
  background: var(--sunk); border: 1px solid var(--border-lt); border-radius: var(--radius);
}
.nv-seg button {
  flex: 1; min-height: 22px; padding: 0 var(--sp-2);
  border: 1px solid transparent; border-radius: calc(var(--radius) - 1px);
  background: transparent; color: var(--dim);
  font-size: var(--fs-sm); line-height: 1; white-space: nowrap; cursor: pointer;
  transition: background var(--dur) var(--ease), color var(--dur) var(--ease);
}
.nv-seg button:hover:not([aria-pressed='true']) { color: var(--ink); }
.nv-seg button[aria-pressed='true'] {
  background: var(--panel); border-color: var(--border-lt); color: var(--ink);
  font-weight: 600; box-shadow: var(--shadow-1);
}

/* ── the card ──────────────────────────────────────────────────────────── */
.nv-card {
  flex: 1 1 auto; min-height: 0; display: flex; flex-direction: column;
  background: var(--panel); border: 1px solid var(--border-lt);
  border-radius: var(--radius-lg); overflow: hidden;
}

/* ── filter box (same recipe as the report filter, .rf-in) ─────────────── */
.nv-find { position: relative; flex: none; padding: 6px 6px 4px; }
.nv-find .nv-find-ic { position: absolute; left: 14px; top: 50%; margin-top: -5px; color: var(--dim); pointer-events: none; }
.nv-find-in {
  width: 100%; padding: 2px 22px 2px 24px;
  font-size: var(--fs-sm);
  background: var(--field); border: 0; border-radius: var(--radius);
  box-shadow: inset 0 0 0 1px var(--field-line);
  color: inherit;
}
.nv-find-in::placeholder { color: var(--dim); }
.nv-find-in:focus { outline: none; box-shadow: inset 0 0 0 1px var(--accent), var(--focus-ring); }
.nv-find-x {
  position: absolute; right: 9px; top: 50%; margin-top: -9px; width: 18px; height: 18px;
  display: grid; place-items: center; border: 0; background: transparent; color: var(--dim);
  border-radius: var(--radius-sm); cursor: pointer;
}
.nv-find-x:hover { color: var(--ink); background: var(--hover); }

/* ── the tree ──────────────────────────────────────────────────────────── */
.nv-tree { flex: 1 1 auto; min-height: 0; overflow-y: auto; padding: 2px 3px 6px; outline: none; }
.nv-g { position: relative; }

.nv-row {
  position: relative; display: flex; align-items: center; gap: var(--nv-gap);
  min-height: 24px; padding: var(--row-y) var(--nv-pad);
  border: 1px solid transparent; border-radius: var(--radius);
  color: var(--ink); font-size: var(--nv-fs); line-height: var(--lh);
  cursor: pointer; user-select: none; outline: none;
  transition: background var(--dur) var(--ease), color var(--dur) var(--ease);
}
.nv-row:hover { background: var(--hover); }
.nv-row:focus-visible { outline: 2px solid var(--accent); outline-offset: -2px; }

.nv-ic { flex: none; width: var(--nv-ic); height: var(--nv-ic); color: var(--dim); }
.nv-lbl { flex: 1 1 auto; min-width: 0; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

/* group header: a touch stronger than an item (600, ink), chevron on the right */
.nv-row.nv-gh { font-weight: 600; }
.nv-chev { flex: none; width: 12px; height: 12px; color: var(--dim); transition: transform var(--dur) var(--ease); }
.nv-gh[aria-expanded='true'] .nv-chev { transform: rotate(90deg); }
/* the group that holds the current screen, and Pinned (always the user's own), carry the accent on their icon */
.nv-g.has-on > .nv-gh .nv-ic, .nv-g.nv-pins > .nv-gh .nv-ic { color: var(--accent); }

/* children: the label lines up under the group's label; a hairline runs under the icon column */
.nv-kids { position: relative; }
.nv-kids::before {
  content: ''; position: absolute; left: calc(var(--nv-pad) + var(--nv-ic) / 2); top: 0; bottom: 0;
  width: 1px; background: var(--border-lt);
}
.nv-row.nv-it { gap: 4px; padding-left: calc(var(--nv-pad) + var(--nv-ic) + var(--nv-gap)); }

/* you are here: wash + accent label + a 2px bar. No weight change (Plex 600 is ~7% wider and would
   truncate the longest labels), and never colour alone. */
.nv-row.on { background: var(--sel); color: var(--accent); box-shadow: inset 2px 0 0 var(--accent); }
.nv-row.on .nv-ic { color: var(--accent); }

/* the Tally-familiar key: dim, right-aligned, ignorable. kbd already carries the shared keycap recipe. */
.nv-k {
  flex: none; margin-left: auto; font-weight: 500; color: var(--dim);
  background: transparent; border-color: var(--border-lt); border-bottom-width: 1px;
}
.nv-row.on .nv-k { color: var(--dim); border-color: var(--border); }

/* hover / focus actions (+ new, pin) sit over the key, so a row never changes width */
.nv-acts {
  position: absolute; right: 2px; top: 50%; transform: translateY(-50%);
  display: none; align-items: center; background: var(--hover); border-radius: var(--radius-sm);
}
.nv-row:hover .nv-acts, .nv-row:focus-within .nv-acts { display: flex; }
.nv-row:hover .nv-k, .nv-row:focus-within .nv-k { visibility: hidden; }
.nv-row.on .nv-acts { background: var(--sel); }
.nv-pins .nv-acts { display: flex; background: var(--panel); }            /* Pinned: the lit star is always there */
.nv-pins .nv-row:hover .nv-acts { background: var(--hover); }
.nv-act {
  width: 20px; height: 20px; display: grid; place-items: center; padding: 0;
  border: 0; background: transparent; color: var(--dim); border-radius: var(--radius-sm); cursor: pointer;
}
.nv-act:hover { color: var(--accent); }
.nv-act.on { color: var(--accent); }
.nv-act.on svg { fill: currentColor; }
.nv-act:focus-visible { outline: 2px solid var(--accent); outline-offset: -2px; }
.nv-act:disabled { opacity: 0.45; cursor: default; }
.nv-act:disabled:hover { color: var(--dim); }

/* footer link to the matching Gateway submenu: quiet, last in its group */
.nv-row.nv-more { color: var(--dim); }
.nv-row.nv-more:hover { color: var(--ink); }
.nv-row.nv-more .nv-chev { margin-left: auto; transform: none; }
.nv-row.nv-more.on { color: var(--accent); }

/* no results */
.nv-none { padding: var(--sp-3) var(--sp-2); font-size: var(--fs-xs); line-height: var(--lh-prose); color: var(--dim); }
.nv-none b { color: var(--ink); font-weight: 600; }

/* ── the full-screen toggle at the start of the breadcrumb ─────────────── */
.crumb-nav {
  flex: none; display: inline-grid; place-items: center; width: 20px; height: 18px; margin-right: -8px; padding: 0;
  border: 1px solid transparent; border-radius: var(--radius-sm); background: transparent; color: var(--dim); cursor: pointer;
  transition: background var(--dur) var(--ease), color var(--dur) var(--ease);
}
.crumb-nav:hover { background: var(--sunk); color: var(--ink); }
.crumb-nav[aria-expanded='false'] { color: var(--accent); }   /* the menu is hidden: the way back lights up */

/* announced to a screen reader, never drawn */
.nv-sr { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; }

@media (prefers-reduced-motion: reduce) {
  .nv-row, .nv-seg button, .nv-chev, .crumb-nav { transition: none; }
}
```

### A.2 `src/styles.css`: replace the two `navhid` lines (section 8.1), nothing else in the file

```css
.app.navhid .main { grid-template-columns: 0 1fr 152px; }
.app.navhid .main > .navrail { visibility: hidden; overflow: hidden; padding: 0; border: 0; }
```

### A.3 `src/styles/light/shell.css`: append inside its existing `@media screen { }`

```css
/* ── left menu (docs/design/left-nav-spec.md) ───────────────────────────────────────────────── */
html:root[data-scheme='light'] .nv-card { box-shadow: var(--shadow-1); margin-bottom: calc(var(--frame-gap) - var(--sp-2)); }   /* 0 in compact (8 - 8), 4px in comfortable (12 - 8): lines up with the screen card */
html:root[data-scheme='light'] .nv-k { box-shadow: none; border-bottom-width: 1px; }                  /* the shared light kbd rule adds a drop shadow at (0,2,2); this is quieter */
html:root[data-scheme='light'] .app.navhid .main { padding-left: var(--frame-gap); }                   /* the rail normally supplies this gap; hidden, the screen card needs it */
html:root[data-scheme='light'] .crumb-nav:hover { background: var(--hover-canvas); }                  /* the status bar sits on the canvas */
```

### A.4 Why no other light rule is needed
The filter box is a boxed field in light by the generic input rule (`entry.css`, white, `inset 1px --field-line`, accent ring on focus): the same values `navtree.css` already states, so base and overlay agree. Row hover and the current-row wash are `--hover` and `--sel`, which are the white-card values. The segmented control, rows and footer links are token-only.

---

## Appendix B: probes (paste into the page; they are what the numbers above came from)

**B1. Geometry with the menu hidden** (press `Ctrl+\` first, on each screen):

```js
(() => {
  const main = document.querySelector('.main');
  const top = [...main.children].filter((c) => c.style.display !== 'none').pop();            // the screen on top of the stack
  const box = (el) => el && [Math.round(el.getBoundingClientRect().left), Math.round(el.getBoundingClientRect().width)];
  const content = [...top.children].find((c) => c.classList.contains('content') || c.classList.contains('dash'));
  const rail = [...top.children].find((c) => c.classList.contains('btnpanel'));
  return { cols: getComputedStyle(main).gridTemplateColumns, content: box(content), btnpanel: box(rail), viewport: innerWidth,
           scrollX: document.documentElement.scrollWidth > innerWidth };
})()
// pass: cols[0] is 0px; btnpanel is [viewport - 152, 152] (null on the Dashboard); content[0] + content[1] === viewport - 152
// (+152 when there is no btnpanel); content[0] is 0, or --frame-gap in light; scrollX false.
// before the fix (bug): content [0, 0], btnpanel [0, 1288].
```

**B2. Truncated labels** (all groups open: `localStorage.TW_NAV_GROUPS = JSON.stringify(<every id>)`, reload; repeat in comfortable, and with Windows key spellings):

```js
[...document.querySelectorAll('.nv-lbl')].filter((e) => e.scrollWidth > e.clientWidth + 0.5).map((e) => e.textContent)   // must be []
```

**B3. One tab stop:** `document.querySelectorAll('.nv-tree [tabindex="0"]').length === 1`.

**B4. Key isolation** (harness, Gateway): focus a tree row (`document.querySelector('[data-nv-id="sales.invoice"]').focus()`), then `p.keys('ArrowDown a Enter')`: the Gateway's `.tmenu-item.sel` text is unchanged and the crumb is unchanged; then `p.key('F9')`: the crumb ends `Voucher` (a global chord still passes through).

**B5. Lint:**

```sh
grep -nE '#[0-9a-fA-F]{3,8}\b' src/styles/navtree.css | grep -v '^\s*[0-9]*:\s*/\*'      # no output
grep -rnE 'navrail[^{]*\{[^}]*display:\s*none' src/styles.css src/styles src/ui          # no output: the bug cannot return
grep -o 'var(--[a-z0-9-]*)' src/styles/navtree.css | sort -u                              # every one defined (visual.assert checks)
```

**B6. Walk every row** (the acceptance script for 13.5; harness, sample company): for each row in section 4.2 whose company feature is on, open its group (`TW_NAV_GROUPS`), click `[data-nv-id="<id>"]`, wait for the crumb, assert the crumb equals `crumbFor(item.to)` and that `.nv-row.on` has the id of the **first** row whose destination matches (`inventory.items`, not `masters.items`); then `goHome()`.

**B7. The prototype's own counters** (what was measured): all 115 destinations rendered with 0 console errors in the sample company (light, compact); hidden-state geometry on Gateway, Day Book, Sales voucher, Trial Balance, GSTR-1 and Dashboard matched B1 after the fix and failed it before; truncation probe returned `[]` at 212px with every group open.

---

## Appendix C: Build notes (builder, 2026-10-19): where the build differs from this spec, and why

The menu was built as specified (14 groups, 108 rows, 8 footer links, `Ctrl+\`, 212 / 186 / 0 widths, the grid fix at its root). Everything below is an addition or a deviation, each with its reason. Nothing else was changed in `MenuScreen.tsx`, `router.tsx`, `store.ts`, `access.ts` or `keys.ts`.

1. **Test file names** follow the task, not section 11: `scripts/assert/sidenav.assert.ts` (spec: `navtree.assert.ts`), `scripts/shots/recipes/sidenav.mjs` (spec: `left-nav.mjs`; screens are named `nav-*`). One file the spec did not list was added: `scripts/shots/flows/sidenav-flow.mjs`, an end-to-end drive (real key and mouse events) of the keyboard model, the chord, persistence, rights, saved views and a walk of every row and footer link. It is the in-browser counterpart of the pure suite, the way the other `flows/` files are, and is not part of `npm run assert`.
2. **Two files beyond section 7.1:** `src/ui/nav/menuModel.ts` (pure: which sections and rows are drawn, open and lit, from the gated tree, the filter, the stored and auto-opened groups, pins and saved views) and `src/ui/nav/useRail.ts` (the rail memory and `go`, shared by both views). The model is pure so the filter, auto-expand, pin and ordering rules are asserted headlessly instead of by screenshot.
3. **`isChordEvent` in `core/navPrefs.ts`.** `favourites.assert.ts` fails any source line that reads `ctrlKey` and `altKey` within 40 characters (it guards the Ctrl+Alt+digit Favourite chord), and the rail has to test "is a chord held" in three places. The test therefore lives once, written a modifier per line; `isNavCollapseChord` is written the same way. The scan itself was not touched.
4. **A pin lights when no tree row does.** A pinned screen that is in no group (a Ledger book, say) is lit as the current screen; a destination that is both pinned and listed lights only its tree row (the spec's rule). Pinned is auto-opened for it like any group.
5. **No results replaces the tree.** With a filter that matches nothing the quiet text sits directly under the box instead of at the bottom of an empty card.
6. **A pinned saved view shows its own name only** (the report it opens is in the tooltip); every other pin keeps "Title: qualifier".
7. **`visual.assert.ts`:** `dim` on `sel` moved from `LIGHT_PAIRS` to `PAIRS` (both schemes, as 11.2 says); the label-fit check measures every row, footer and group header with the shipped Plex advances (widest row **142.9px of 144**, "Reverse Charge Register"; "Compare with Tally Data" is 142.6) and asserts that 186px would truncate at least 20 rows, so the extra 26px is earned, not decorative.
8. **Quick access: Enter on a focused row now activates it.** Before, a Quick access button reached by Tab had its Enter taken by the screen's key stack first (the Gateway's Enter opens the highlighted menu item); the rail's own keydown rule (section 7.3) now keeps every bare key inside the rail, which makes those buttons keyboard-operable. No Quick access content, order or label changed.
9. **Measured in this build** (headless Chrome, sample company): row height 24px compact / 32.8px comfortable, labels 13px, group headers 600 and sentence case in both schemes, `.btnpanel` 152px everywhere, hidden-state geometry (probe B1) on Gateway, Day Book, Balance Sheet, Trial Balance, Sales voucher, Dashboard, GSTR-1, Ledger create, Print Configuration and Bank Reconciliation in light, dark, compact and comfortable at 1440x900, and the same ten at 1280x720 (comfortable, light and dark); zero truncated labels with every group open.
10. **A group header is inert while a filter is typed.** The spec says a click calls `toggleGroup(id, shownOpen)` with what was visible, "filter" included; but while filtering every match is held open and the stored state is meant to come back untouched when the box is cleared, so a click would silently rewrite what returns. Header click and Enter do nothing while there is filter text; the groups hold open, and clearing the filter restores exactly the previous open / closed state.

---

## Appendix D: Fix round (2026-10-19): what four reviewers confirmed, and what changed

The build was reviewed by four independent lenses (keyboard and frozen behaviour; visual quality in light, dark, compact and comfortable; accessibility, rights and data; integration and drift). Every confirmed finding was fixed at its root; each fix is pinned by `scripts/assert/sidenav.assert.ts` (sections 14 to 16: the published table row by row, the review findings, the wiring read from source) and by `scripts/shots/flows/sidenav-flow.mjs` (the `review fixes` section, real key and mouse events). Deviations from this spec, each with its reason:

1. **The rail is the LAST child of `.main`** (section 7.4, rule 4; 8.1). Reason: with the rail first, Shift+Tab from a screen's first field landed on a rail row, and from then on the rail owned Enter, the arrows, letters and the first Esc; at `HEAD` the same Shift+Tab hit an inert sidebar button and every key still reached the screen. It is placed in grid column 1 / row 1 explicitly, so every screen auto-places as before (checked on every screen by the geometry probe and the flow). Cost: Tab no longer reaches the rail before the screen's fields, so **"Skip to left menu"** was added (a clipped, focus-only first stop; no key). Probes and recipes that read "the last child of `.main`" now read "the last `<div>`".
2. **212px in both views** (decision 7). Reason: Quick access at 186 re-flowed the open report by 26px on every view switch (measured 1076 to 1102 on Trial Balance), and the 212 column is the default view anyway. The `.main` base rule is unchanged; `AppShell` adds `nv-wide` always.
3. **Default open: Pinned only** (was Pinned and Sales). Reason: Sales is two thirds of the tree at comfortable density and pushed Masters .. Settings below the fold on a 1366x768 laptop with only a scrollbar to say so. The group of the current screen still opens itself, so nothing needed is hidden. A fresh device now shows the whole product in 14 rows.
4. **The breadcrumb toggle shows words** ("Hide menu" / "Show menu") and has no `aria-expanded`; the hamburger keeps its stable name and `aria-expanded`. Reason: a 14px icon at the extreme bottom-left with its only explanation in a tooltip was undiscoverable and under the 24px target; a label that flips plus a state reads as a contradiction.
5. **The star has no `aria-pressed`** and is named by its action. Reason: "Unpin, toggle button, pressed" contradicts itself. Every treeitem is named by `aria-label` (its own label), the tree is described by a hidden hint, and `aria-keyshortcuts` is set only on F-key chords.
6. **`go()` is `planNav`** (7.1). Reason: the Menu view made the unguarded `goHome()` a one-click way to lose a half-typed voucher. The rule is the Favourite shortcuts' (`favouritesCore.planNav`), so the three navigators (rail, Go To, Favourite shortcuts) now agree.
7. **A voucher key's tooltip says "on the Gateway and its menus"**, and the `?` sheet's heading "New voucher: from the Gateway or a menu" (it said "from a menu or a report", wrong at `HEAD` too). Reason: `MenuScreen` binds the ten voucher F-keys for the Gateway and every submenu (the voucher and Bulk Entry screens reuse them to switch type); on a report or a list they do nothing, so the rail's rule that a keycap never over-promises was being broken by its own wording. A pin of a screen is labelled by its tree row, and a party's pin leads with the party.
8. **Pins and saved views obey the company's feature switches in the Menu view** (the tree already did). Quick access is unchanged and still shows every pin.
9. **Smaller things:** the `?` sheet leads with "In the left menu" when it was opened from a row of the menu (it stays search-only otherwise) and lists the skip link; the pin limit counts STORED pins (`togglePin` refuses at `PINS_MAX`, whatever rights or switched-off modules hide); Enter in an empty filter opens nothing; unpinning the focused pin keeps focus in the rail; the filter underlines what matched and names an alias that explains a row; headers are drawn inert while filtering; the action strip reserves its width and has no tile; headers are sticky; the dark selected segment has a wash; forced colours has its own block; the `+`, star and clear targets are 24px; the API Keys and Subscription screens let `Ctrl+\` through their catch-all key handlers (so "Everywhere" is true); `npm run perf`'s `sidebar.today` mirror points at `QuickAccess.tsx`.

**Known and accepted:** a five-period Cost Centre Analysis scrolls sideways at 212px (its header text is `nowrap`; four other reports already overflowed in both views); `Batches & Expiry` is still a stub and stays out; Real Windows, Linux, Firefox, Safari and Electron key presses, and a real Windows High Contrast theme (emulation only so far), are not yet exercised; `security-signin*` recipes time out in dark / comfortable on `HEAD` and on this build alike (the wait for "Who is working?" never matches there; not caused by this feature).

**Harness:** the `nav-forced-colors` recipe emulates `forced-colors`, and Chrome's media emulation is browser-wide and survives the reload between screens, so every screen after it in a run was being shot under it. `scripts/shots/shoot.mjs` now restores the run's own scheme and reduced-motion after each screenshot (see `scripts/shots/README.md`); the full recipe sweep (478 screens, light / compact, four parallel shards: zero failures, zero console errors) passed; the 32 screens that had followed it in one shard, and the 29 `nav-*` recipes in all four scheme and density combinations, were re-run after the fix.
