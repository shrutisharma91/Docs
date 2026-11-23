# Comsky Books: UX analysis for two kinds of user (October 2026)

> **Status:** analysis, written to decide what to build next. **Date:** 2026-10-19. **Author:** UX analyst pass (read-only: nothing under `src/`, `scripts/` or `electron/` was edited; this is the one file created in the repo).
> **Scope:** the whole experience (left menu, Dashboard, main window, right-hand F-key rail, top bar, status bar, first run) for two people: **A, the Tally switcher** (keyboard-first) and **B, the business owner from Zoho / Vyapar / Busy / Excel** (mouse-first). It sits beside the owner's request of 2026-10-19: **Wave A** = a "+" beside every menu row, "+ New" top-right, a universal "+", a Reports Center, saved filters / views, the navigation-stack fix. **Wave B** = configurable shortcut buttons and an admin Settings window (theme, accent, fonts, size). Wave B is analysed here only where it constrains Wave A.
> **Evidence tags:** **[E]** measured or seen in this session (headless Chrome through the `scripts/shots` launcher, sample company, 1440x900 unless stated, light/compact); **[C]** read from code (file and line given); **[Z]** Zoho fact from Zoho's public help pages fetched 2026-10-19, or from the owner's written description of his screenshots; **[M]** from memory, not re-checked this session; **[J]** my judgement. Anything not tagged is arithmetic on tagged facts.

---

## 0. The verdict

1. **The product is two products joined at a few seams, and the seams, not the features, are where both people lose time.** Tally's model is a *stack of screens with Esc as Back*; Zoho's is *destinations you replace, a list per destination, and a "+" to create*. The left menu speaks the second idiom but is wired with the first, which is exactly the breadcrumb bug.
2. **The breadcrumb bug is verified, and it is wider than the diagnosis.** One *untouched* F8 voucher is enough: after six left-menu clicks the crumb was eight screens deep, and once three Go To jumps were added it took ten Esc presses to get home [E]. Three things the diagnosis did not say: (a) **Go To always pushes**, with no voucher anywhere (`Gateway › Balance Sheet › Trial Balance › Day Book` after three jumps) [E]; (b) **the crumb names the component, not the object** ("Voucher", "Register", "Outstandings", "Menu" for every voucher / register / bill list / submenu) [C `router.tsx` CRUMBS]; (c) the "dirty" signal the fix wants to reuse **does not exist for masters and is switched off for alterations** (section 5.3).
3. **Clicking "Sales Invoice" opens an entry form; Zoho's "Invoices" opens a list.** And the register we have is a **monthly summary** (Month, No. of Vouchers, Amount) with a month drill, not a list of invoices, and it has no status, due date or balance [E, C `reports/financial.tsx:321`]. "Click Sales and see all sales" is therefore not a re-pointing of rows; it needs a **flat, status-aware document list** (N2). Without it Wave A ships a "+ New" button on a table nobody can use to find an invoice.
4. **There is no way to create anything from a report.** F8 and the other voucher keys fire only on the Gateway and its menus; on a report they are dead [E: F8 pressed on Receivables did nothing; spec `left-nav-spec.md` 4.2]. A (Tally) goes Esc, Esc, F8; B has no path except the rail.
5. **Day one of an own (non-sample) company is empty.** A 29-field company form, then a Gateway saying "No Vouchers Entered", a Dashboard of six `₹0.00` tiles, a Sales Register of twelve blank months, and **no Getting Started anywhere** (it is reachable only from the sample banner, Go To and the Alt+F3 company menu) [E, C]. The sample company is excellent; the real one is a cliff.
6. **The Dashboard has the right six numbers on the wrong clock.** Tiles print "as at 31-Mar-27" (period end, in the future) and cash / receivables / payables are balances at period end, not today [E, C `dashboardData.ts:239-243`]. There is **no profit or expense tile** and no chart by default (trend, top customers, GST payable, stock alerts exist but are off) [C `dashboardLayout.ts`].
7. **Trust leaks:** half the top bar is dead with a tooltip reading "Not available in MVP" (Data, Exchange, Import, Share) [E]; the sample banner says "never synced" while the status bar says "synced" [E]; the sync tooltip tells users to set `localStorage.TW_SYNC_URL` [C `widgets.tsx`]; there is **no `beforeunload` guard** and **a master form discards on Esc without asking** [C]. Each is small; together they read "demo".
8. **The shell does not respond below 938px** (212 + 574 + 152): at 768 and 390 the page scrolls sideways and the sample banner wraps to four lines [E]. B checks numbers on a phone; today that is not possible.
9. **Where we are already better:** speed of repeat entry (ghost recall, amount shorthand, bulk paste), a Go To that is genuinely good, offline-first, Indian compliance depth in one tree, operator rights with an audit trail, undo that keeps the audit trail, a sample company, self-ticking Getting Started. The right-hand rail, the hold-Alt reveal and the `?` sheet are a better bridge from Tally than anything Zoho offers a Tally user. Section 6.
10. **What I would do next, in order:** the Wave A items as scoped (with the three conditions in section 10), then a **guided start** (N1), **status-aware lists** (N2), an **owner Dashboard** (N4), a **post-save moment** (N3), the **trust and honesty fixes** (N5, N6), a **safe-leave kit** (N7), **role-based home** (N8), **dual-label vocabulary** (N9), a **responsive shell** (N11), **real buttons** (N12). Section 9 has the reasons.

### Scorecard (my judgement, with the evidence in section 4)

| Area | A: Tally switcher | B: owner from Zoho / Vyapar | Main reason |
|---|---|---|---|
| Navigation model | Strong | Weak | A: Gateway + letters + Esc intact. B: rail destinations stack, no Home, Esc is the only Back |
| First run / own company | OK | Weak | Welcome and sample are strong; own-company start is a 29-field form then nothing |
| Dashboard | OK | Weak | Right tiles, period-end clock, no profit, no trend by default |
| Create flows | Strong | Weak | F-keys everywhere, but no "+" on any list or report |
| Finding things | Strong | OK | Go To is excellent; no voucher lookup by number / amount; no saved row filter |
| Terminology | Strong | Weak | Search aliases exist; headings, columns and crumbs are Tally-first |
| Right-hand rail | Strong | Weak | A: muscle-memory bridge. B: unlabelled caps, not obviously clickable |
| Key consistency | Strong | Weak | Ctrl+A is Accept; no Ctrl+S; F-keys dead on reports |
| Data-safety cues | OK | Weak | Esc prompt on vouchers only; no draft recovery; jargon in sync state |
| Speed | Strong | Strong | Rail click to paint 18 ms shallow, 16 to 37 ms at 12 to 15 mounted screens (43-voucher sample) |
| Accessibility | OK | OK | Rail is a proper tree; the F-key rail, tiles and top bar are clickable divs |
| Responsive / mobile | OK (desktop) | Missing | Hard 938px floor |
| Help and learning | Strong | Weak | `?` sheet and Alt-reveal are great; nothing explains a voucher |

---

## 1. Evidence and its limits

**What was done.**
1. Read the code that matters: `AppShell.tsx`, `ui/nav/*` (all ten files), `navTree.ts`, `navStore.ts`, `useRail.ts`, `favouritesCore.ts` (`planNav`, `mayHoldWork`, destinations), `widgets.tsx` (TopBar, StatusBar, ButtonPanel), `MenuScreen.tsx` (the Gateway menus), `modern/Dashboard.tsx` + `dashWidgets.tsx` + `core/dashboardLayout.ts` + `core/dashboardData.ts`, `modern/CommandPalette.tsx`, `paletteRank.ts`, `onboarding/*`, `CompanyCreate.tsx`, `reportViews/*`, `core/savedViews.ts`, `router.tsx` (CRUMBS), `reports/financial.tsx` (RegisterReport), `VoucherScreen.tsx` (dirty / quit paths), `forms.tsx` (master Esc), `state/theme.ts`, `docs/design/left-nav-spec.md`, CLAUDE.md.
2. Viewed the re-captured BEFORE shots in `scratchpad/zx-before/` (gateway, dashboard, voucher-sales, daybook, balance-sheet, outstanding-receivables, nav-menu-all-open, key-sheet).
3. Ran four scratch probes on the harness's own launcher (`launchChrome`, `Page`, `A.*` from `scripts/shots`, a throw-away profile each; scripts live in the session scratchpad, not the repo; **nothing in the repo was changed and nothing saved except one voucher in a throw-away profile**): (1) stack growth through the left menu and Go To, with timings; (2) own-company first run and every empty state; (3) widths 1280 / 1024 / 768 / 390, Go To, F12, Alt+V, filter, Getting Started, dark; (4) top-bar state, F-key rail counts, the moment after Accept. Output: `scratchpad/ux-probe/out1 … out4` (Appendix C).
4. Fetched Zoho's public help for the Reports Center, custom reports and custom views (Appendix B).

**Limits, stated plainly.**
* **The owner's five screenshots could not be opened**: the temp folder had been cleaned (`images/` was empty). Everything about Zoho's Sales menu, "+ New" split button, mega "+" menu and Reports Center comes from **his written description** plus Zoho's help pages. The Marg reference is from `left-nav-spec.md`.
* **No live TallyPrime and no Zoho account were used.** Statements about how TallyPrime or Zoho behave are **[M]** unless tagged **[Z]**.
* **No users were observed.** The two personas are hypotheses built from the owner's description of his customers. Section 11 says how to test them in ten sessions.
* **Performance was measured on the 43-voucher sample only**, in headless Chrome on one Mac, against the dev server. Large books were not measured. The repo's own `npm run perf` covers the engines, not stacked screens.
* **The harness shows "160 pending"** in the status bar right after creating an own company. My reading is that the harness has a sync URL configured with nobody signed in (I did not verify it); the code says a signed-out user with no sync URL sees "local only · N unsynced". Either way the number is a raw change-log row count (section 4.9).

---

## 2. The two users

### A. The Tally switcher
An accountant, or an owner who keeps his own books, at a trading business or at a Tally reseller's customer. Knows the Gateway letters, F4 to F9, Alt+G, Ctrl+A, Esc as Back, F12 to configure a report. Wants identical key positions, dense screens, no mouse detours; will tolerate any new surface that does not move a key. Fears: a key that now means something else, a familiar word replaced by a fashionable one, losing a half-typed voucher. **What A needs from this wave:** nothing taken away; the stack to stop breaking Esc; the new surfaces to teach him nothing he did not ask to learn.

### B. The business owner from Zoho / Vyapar / Busy / Excel
Navigates by mouse and by left menu; expects a list per module, a "+ New" at top-right, a search box, a dashboard on arrival; thinks in documents and money ("invoice", "who owes me", "GST due"), not in vouchers and ledgers. On a phone half the day. Fears: jargon, deleting something, not knowing what to click next, "is it saved?". **What B needs:** to be able to use the product with no key at all, in his own words, from the first minute, on any screen size.

### The constraint that makes this hard: they share one set of books
In the SaaS shape (multi-user, operator rights) the owner (B) and his accountant (A) open the **same company**. The right design is **one product with two defaults**, chosen by role, not two products. That is why N8 (role-based home) and Wave B's presets matter more than any single screen.

---

## 3. The experience as built

```
┌ top bar: [≡] Comsky Books  K:Company  Y:Data  Z:Exchange  O:Import  E:Export  M:Share  P:Print  F1:Help   [Go To ⌘K] [Light][Compact] ┐
│ left rail 212px        │ main window (the screen)                                   │ F-key rail 152px      │
│ Menu | Quick access    │ Gateway (centred letter menu)  or  Dashboard  or  report   │ "Button bar": per-    │
│ filter box             │ or voucher / master form                                    │ screen key list       │
│ Dashboard W            │                                                             │ Gateway 12 · voucher  │
│ Sales > …              │                                                             │ 23 · reports 4 to 7 · │
│ …14 groups, 108 rows   │                                                             │ Dashboard 0           │
└ status bar: [Hide menu] crumb (Gateway › Voucher › …)            sync dot · Comsky Books ┘
```
Strips that can sit above or below: sample banner (top, every screen), trial / storage / backup banners (above the status bar). All figures [E].

### 3.1 What each surface is for, who it serves, what it duplicates

| Surface | Job | Serves | Duplicates |
|---|---|---|---|
| **Gateway** (home, centred letter menu) | The Tally tree: Create / Alter / Vouchers / Day Book / Reports | A | The left rail's groups; the F-key rail's voucher keys |
| **Left rail, Menu view** (108 rows, 14 groups) | Module map for people arriving from an online ERP | B first, A as accelerator | Gateway, Go To, Dashboard Quick Entry |
| **Left rail, Quick access** (Today / Pinned / Recent) | A's "where I just was" | A | The Menu view's Pinned group |
| **Dashboard** (6 tiles + Quick Entry + Recent by default) | Orientation and the first click | B | Rail, Gateway "Reports", the top "+ New Sales Invoice" and its own Quick Entry card (Sales Invoice appears twice on one screen) |
| **Main window** | The real work: forms, reports | both | n/a |
| **F-key rail** ("Button bar", 152px) | Truthful key list for this screen | A | Top-bar letters; rail keycaps; Go To's "New X" |
| **Top bar** | Brand, nine letter-labelled items, Go To pill, theme toggles | A (letters), B (pill) | Four of the nine items do nothing |
| **Status bar** | Crumb, Hide menu, undo toast lane, sync | both | n/a |

### 3.2 The transitions, and where each person is lost

| # | Transition | A | B | Evidence |
|---|---|---|---|---|
| T1 | Arrive (first screen after open) | Gateway: home | Gateway: a centred text menu with underlined letters, "Create / Alter / Chart of Accounts", no icons. Dashboard is one click away, not home | [E] gateway.png |
| T2 | Dashboard tile to report, then Esc | Fine: Esc returns to the Dashboard (it was pushed) | Fine, but Esc is the only Back; the F-rail's "Esc Quit" is a div labelled with a Tally word | [E] |
| T3 | Rail row to a voucher form ("Sales Invoice") | Fine | Lands on **"Accounting Voucher Creation"**: the title is not the label he clicked | [E] voucher-sales.png |
| T4 | Voucher, then Accept | Stays on a blank next voucher; toast "Voucher 19 saved ⌘Z Undo"; the ledger list pops open at once | "Did it save? What now?" No Print / Share / Record payment | [E] out4 |
| T5 | Report to "create one of these" | Esc until the Gateway, F8 | No path except the rail or Go To; no "+" | [E] F8 dead on a report |
| T6 | Rail click while a voucher is anywhere in the stack | Esc walks back (safe, but 10 deep) | Pushes silently on top; the crumb grows; Esc is the only way out | [E] probe 1 |
| T7 | Rail footer ("Books & registers…") into a Gateway submenu | The escape hatch A wants | A jolt: from a left-aligned tree into a centred letter menu with a different F-key rail | [C] `navTree.ts` `more` |
| T8 | Gateway Esc | "Quit ?" (Tally-faithful) | "Quit ?" with **Yes preselected**; Enter runs `window.close()` (it ends the desktop app; a plain browser tab usually ignores it [M]) | [E] w1024-gateway.png, [C] `MenuScreen.tsx:616` |
| T9 | Dashboard to anything with the keyboard | F12 = Customize (consistent with "Configure") | The Dashboard has **no F-key rail** (0 buttons) so the right 152px vanishes; the layout shifts between Dashboard and every other screen | [E] probe 4 |
| T10 | Any screen to the Dashboard | W on the Gateway, rail row, Go To | Rail row #1: fine | [C] |

### 3.3 What is redundant (and why it matters)
* **Six ways to start a sales invoice** (F8, F-rail F8, rail row, Dashboard CTA, Dashboard Quick Entry, Go To "New Sales") and **zero ways from a report**. Redundancy where it is not needed, absence where it is.
* **Two search boxes** with different scope: the rail's "Filter menu" (menu rows only) and the top-bar Go To pill (everything). The rail already says "press Ctrl+K to search everything" on a no-match [C `MenuView.tsx`], which is the right bridge; keep it.
* **Pinned** in the Menu view and in Quick access; **Quick Entry** card and the rail; **sample banner** and Getting Started.
* **Light / Compact** toggles on every screen of a product whose users will set them once (Wave B moves them into Settings; the labels name the *current* state, "Light", not the action).
* **Four dead top-bar items** (Data, Exchange, Import, Share) with the tooltip "Not available in MVP" [E, C `widgets.tsx` TOP_ITEMS].

### 3.4 What the common jobs cost today

Counted from the running UI and the code ([E] / [C]), not timed with users. "After A" is what Wave A should give if the conditions in section 10 are met.

| Job | A today | B today | After Wave A (expected) |
|---|---|---|---|
| New sales invoice from the Gateway or Dashboard | 1 key (F8) | 1 click (Dashboard "+ New Sales Invoice"); 2 via the rail if Sales is closed | unchanged |
| New invoice while looking at a report | Esc once per stacked screen, then F8; or Go To + a few letters + Enter | 2 clicks through the rail (open Sales, click Sales Invoice) | 1 click ("+ New"), or F8 on the list |
| "Who owes me?" | Go To + 3 to 4 letters + Enter (about 6 keystrokes) | 1 click (Receivables tile), 2 via the rail | unchanged |
| Find last month's invoice for one customer | K (Day Book), `/`, type the name | Sales, Sales Register, a month, then scan: 4 clicks and no search by number | list with search: 2 clicks and typing |
| Look something up mid-invoice, come back | Go To, then Esc (pushed, so safe) | 1 rail click (pushed silently on top), then Esc; the only on-screen Back is the F-key rail's "Esc Quit" | "Leave this entry?" dialog |
| Get home after browsing | 1 Esc per stacked screen: 10 from 11 deep | no Home control at all; the same Escs | 1 |
| Reopen "overdue 60+ days for one customer" next week | Set it up again | Set it up again | a saved row filter |
| First ten minutes in an own company | 29-field form, a blank Gateway, no next step | the same | N1 |
| Back up the data | Alt+F3, B | Data & Utilities, Backup / Restore (2 clicks); nothing prompts before the first reminder | unchanged |

---

## 4. Findings by area

Each finding: what is there, what it costs each person, what to do. Findings F-01 to F-12 are numbered because other sections cite them; the rest are cross-referenced to the register in section 8 by ID (A1 to A8, B1 to B3, N1 to N18).

### 4.1 Navigation model and mental map

**F-01. The rail navigates by destination but is wired like the Gateway.** `useRail.go()` asks `planNav`, which pushes on top when any screen above the Gateway `mayHoldWork`, and `mayHoldWork` is true for **every** entry form (`kindOf(scr) === 'create'`) whether or not anything was typed [C `favouritesCore.ts:456`]. Verified: **F8, type nothing, then click rail rows** gives `Gateway › Voucher › Register › Outstandings › Voucher › Balance Sheet › Day Book › Outstandings` after six clicks; **ten Esc presses** to reach the Gateway from eleven screens [E]. And one Esc from the pushed Balance Sheet lands back on the (empty) voucher [E]. Cost: A's Esc is no longer predictable; B has no Back at all. Do: A1, section 5.

**F-02. Home is the Gateway for everyone.** Esc from anywhere ends on the Tally tree. B arrives and returns to a screen he cannot read; the brand in the top bar is a plain span, not a link to Home [C `widgets.tsx`]. Do: N8.

**F-03. The crumb says which component, not which thing.** `CRUMBS`: `voucher: 'Voucher'`, `register: 'Register'`, `outstandings: 'Outstandings'`, `pickMaster: 'Select'`, any submenu `'Menu'` [C `router.tsx`]. "Gateway › Voucher › Voucher › Register" is unreadable even at four deep. Zoho has no breadcrumb: the page title and the lit menu row say where you are. Do: titles carry the object (F-04) and the crumb is shortened (A1).

**F-04. The label you click is not the title you see.** "Sales Invoice" lands on "Accounting Voucher Creation / Sales"; "Receivables" on "Bills Receivable"; "Payment Received" on a Receipt [E]. This is the single cheapest orientation fix: **title = the label clicked, with the Gateway word as the sub-title** (N9).

**F-05. Rail rows mean "open an entry form" for vouchers and "open a list" for masters.** Sales Invoice and Payment Received open forms; Ledgers, Groups, Stock Items open a pick list with a hover "+" (nine rows have it) [C `navTree.ts`]. The master rows already follow Zoho's idiom (`to` = list, `create` = form, `also` = the edit forms that keep the row lit). **Wave A is generalising a pattern that exists, not inventing one**: lower risk than it looks.

**F-06. Sales and Purchases carry rows that are really views of one list.** Sales has nine rows: Sales Invoice, Payment Received, Receivables, Sales Register, Sales Order, Credit Note, Delivery Note, Ageing Analysis, Reminder Letters. Receivables, Register, Ageing and Reminders are four ways to ask "which invoices, in what state". Zoho's equivalent group also has nine, but each is a *document* (Customers, Estimates, Sales Orders, Invoices, Delivery Challans, Payment Links, Payments Received, Credit Notes, e-Way Bills) [Z owner]. The owner's "decrease the number of submenus" is best met by **folding the four into views of one Invoices list** (All, Unpaid, Overdue, By month), giving about six rows. Condition: Receivables is bill-wise (an invoice can be part-paid, an advance is "On Account"); the list must read `outstandingBills`, and with the bill-wise feature off there is no Unpaid / Overdue view, which the row must say rather than hide silently.

**F-07. Auto-expansion accumulates.** Each landing opens its group for the session; after a short walk the rail had Sales, Accounting and Reports open and a scrollbar [E out1 stack-after-menu-walk]. Minor; Zoho does the same. An "accordion" preference would remove it (N18).

### 4.2 First run, onboarding, empty states

**F-08. The front door is excellent, the back door is a cliff.** The Welcome offers three honest paths (sample, bring my books, set up my own) and teaches three keys [C, E]. The sample company, its banner and the self-ticking Getting Started are better than Zoho's tour [J]. But:
* **Own company = one 29-field scrolling form** (Name, Mailing Name, Address, State, Pincode, E-mail, Phone, two dates, GST type, GSTIN, PAN, **TAN, Person responsible (TDS), passcode, GST periodicity, e-Invoicing date, e-Way date, LUT number, LUT date, CIN, Udyam, IEC**, Website, Second phone, Jurisdiction, City, Reverse-charge, Logo) [E own-company-form.png]. A business owner must understand LUT and TAN to start. Only Name is needed to open; everything else can be asked later in context.
* **After Accept: the Gateway, "No Vouchers Entered", "2 ledgers · 0 stock items · 0 vouchers", and nothing else** [E own-company-home.png]. No checklist, no "add your first customer", no import prompt. The sample banner (which carries Getting Started) is not shown for own companies.
* **Getting Started is reachable from: the sample banner, Go To, and Alt+F3 then G.** Not from the left Menu, not from the Dashboard [C `navTree.ts` has no row; `dashboardLayout.ts` has no widget].
* **Empty states are zeros, not invitations.** Dashboard: six `₹0.00` tiles, "No vouchers yet — press ⌘K or use Quick Entry" (a key hint). Sales Register: twelve blank month rows and a total of 0 [E empty-sales-register.png]. Receivables: "No outstanding bills" under a table header [E]. Zoho puts a one-line explanation and a "+ New" in every empty list [M].
Do: N1.

**F-09. The sample banner is a keyboard script.** "Press F8, type Sha, Enter, then ⌥R..." on every screen of the sample [E]. For A it is a perfect lesson; for B it is a sentence he cannot act on with a mouse. At 390px it takes four lines. Do: make the banner two-faced: the same sentence plus a clickable "Try it: new invoice for Sharma Coaching" that opens the form with the party chosen (N1).

### 4.3 The Dashboard (new for Tally users, the first screen B should see)

What a business owner needs on day one, against what is there:

| Need | Today | Gap |
|---|---|---|
| Cash and bank now | Tile "Cash & Bank", on | Balance **at period end** (31-Mar-27), not today [C `dashboardData.ts:239-243`] |
| Who owes me | Tile "Receivables", on | Period-end again; click opens the full bill list, not "overdue first" |
| Who is late | Tile "Overdue", on, red | Ages to min(period end, today) correctly, but the foot still says "as at 31-Mar-27"; click opens all receivables |
| What I owe | Tile "Payables", on | Same clock |
| Am I making money | Not present | **No net profit, expense or margin tile**; "Sales (period)" is the whole financial year, not this month |
| Trend | Card "Sales and purchase trend", **off** | Exists (`monthlyTrend`), hidden by default |
| Top customers / items, stock alerts | Cards, **off** | Exist, hidden by default |
| GST due | Tile "GST payable", **off** | Exists; hidden; no due-date countdown |
| TDS to deposit | Tile, **off** | Exists; hidden |
| What needs attention | Not present | No overdue-customer list, no unreconciled bank count, no backup / mismatch note (these live in banners and the status bar) |
| First-run guidance | Not present | N1 |
| Charts | One CSS trend card, off | No cash-flow or income / expense chart; Zoho's dashboard leads with these [M] |

Also: tiles are `div onClick` with no role, tab stop or focus ring [C `dashWidgets.tsx:91`]; "Customize" is F12 and the Dashboard has no F-key rail, so the 152px column vanishes only here [E]. The customise panel is good (keyboard-complete, per-operator layout [C `layoutOwner`]); **the defaults are the problem, not the machinery**. Do: N4 (default board, clock, profit, attention card) and a "Today | Period end" switch on the tiles.

### 4.4 Create flows (the "+" concept)

* Today: every create path assumes you are on the Gateway, a menu, the Dashboard or the palette (F-key table in `left-nav-spec.md`: F4 to F9, Alt+F5 to Alt+F7, Ctrl+F7 "fire on the Gateway and every submenu; they do nothing on a report or a list").
* The palette already has a complete, feature-gated, rights-gated **Create** list ("New Sales (invoice)", "New Receipt", "New Ledger"... with the F-key as keycap) [C `CommandPalette.tsx:203-260`]. **The universal "+" should read this list, not a third copy.** The drift guard that ties Go To to `favouritesCore.STATIC` shows a second copy will drift.
* Zoho's "+" mega menu groups by GENERAL / INVENTORY / SALES / PURCHASES / BANKING [Z owner]. A mapping for us (everyday word first, the F-key as keycap, the Gateway word in the search aliases):

| Column | Entries (everyday label, key) | Gate |
|---|---|---|
| Sales | Invoice F8, Payment received F6, Credit note ⌥F6, Sales order, Delivery challan | orders, inventory |
| Purchases | Bill F9, Payment made F5, Debit note ⌥F5, Purchase order, Goods received | orders, inventory |
| Banking | Transfer F4, Cheque, Import statement | none |
| Accounting | Journal entry F7, Customer / vendor / account (ledger), Group | none |
| Inventory | Item, Stock transfer ⌥F7, Stock count ⌘F7, Godown, Unit | inventory |
| People | Lead, Employee, Leave | payroll, none |

* **Where "+ New" should live and what it should do.** Top-right of every list / report that has a creatable object, **bound to that object's own F-key** (on the Sales list, F8 creates a sale). A report already ignores F8, so this is additive; it also matches what TallyPrime's Day Book does with voucher keys on its button bar [M]. The button label is everyday ("New invoice"), the keycap is the Tally key, the F-key rail shows it as one more truthful entry.
* **A chord for the universal "+".** `Ctrl+N` is browser-reserved on web [C `keys.ts` BROWSER_RESERVED]; bare `+` is already the rail's own key and must stay local to the rail; `Alt+N` means "After: new" inside a voucher. The existing **G-chord namespace** ("G then letter", already used for CRM and HR) is free of conflicts by construction; **G then N** is an option for the builder to verify against `keys.ts`, the `?` sheet and `overlayUp()`. I claim no key here.
* **After Accept** nothing offers the next step (T4). Do: N3.
* Where the "+" appears on the rail: nine master rows today, **on hover or focus only** (`navtree.css:150`). Zoho shows it on the hovered / active row too [Z owner: "Invoices (+)" on the lit row], so hover-reveal is faithful; add "always on the current row" and a first-visit hint, since a mouse user who never hovers never learns it.

### 4.5 Finding things

**F-10. Go To is a real strength.** Strict tiers (alias exact, alias prefix, title prefix, substring, fuzzy), frecency inside a tier, a destructive-command guard, a stable top row [C `paletteRank.ts`]; domain aliases in both vocabularies [C `CommandPalette.tsx:46`]. Typed "sales": Sales - GST (a ledger) first, then Sales Register, Sales Orders Outstanding, New Sales (invoice) F8 [E goto-sales.png]. It indexes screens, creates, ledgers, stock items and the five latest vouchers; **I found no index of voucher numbers, amounts or narrations** [C, E]. B's most common search ("find invoice 123 / Sharma's last bill") has no home. Condition on any change: A's muscle memory depends on the **stability of the top row**, so new result types go in as a **new scope** or a chip, never by re-tiering existing hits. Do: N13.

**F-11. "Save filter" is not what Saved Views do.** Saved Views capture **F12 options, sort, added columns, basis and (optionally) the period**, and explicitly **not the typed filter text** [C `core/savedViews.ts` header]. The dialog says so: "Set the report up the way you want it (F12 options), then press N" [E sales-register-views.png]. There is **no row-level filter model** (party, status, date range, amount). The typed filter (`/` or Ctrl+F) matches printed text only [C `useReportFilter.ts`]. Views are **kept on this device** [C `useReportViews.tsx` header], so an owner's view never reaches his accountant or his phone. Zoho's equivalent: system filters per module plus "+ New View": name, favourite, criteria (field, comparator, value, add criterion), column preference, visibility (me / shared) [Z custom-views page]. What the owner asked for is that; ours is a different thing with the same name. Do: A6 (with the three conditions in section 10).

**F-12. Three finding tools, three scopes, no map.** Rail filter (menu rows), Go To (everything), report `/` filter (rows on screen). Fine for A, who learned them; B needs one sentence of help at each ("search everything", "search this list"). Do: N14 (help) and chip labels in N13.

### 4.6 Terminology bridge

The bridge exists in **search** (108 rail rows carry invisible aliases such as udhaar, kharid, debtors, challan, GRN; the palette has 48 domain-alias entries) and in the rail's job words ("Payment Received", "Payment Made"). It stops at the **display** layer: page titles, column headers, field labels, the crumb, the F-key rail labels and the voucher form are Tally-first (T3, F-04). Section 7 has the glossary and where each treatment goes. The rule I recommend:
* **Never remove or rename a Gateway or F-key word** (A's training sheets point at them).
* **New surfaces (rail, "+", lists, Reports Center, Dashboard) are everyday-first**, with the Gateway word as a quiet sub-label or alias.
* **Old surfaces get dual labels, not new labels** (title plus sub-title, tooltip on column heads, field hints), never the word "Tally" in the UI (brand rule), only "also called".
* A global "vocabulary" switch is **not** worth it yet: two label sets double the strings every assertion suite reads. Dual labels first.

### 4.7 The right-hand shortcut rail (the "Button bar")

* **What it is:** a per-screen, truthful list of keys, 152px wide, always (except on the Dashboard). Counts [E]: Gateway **12**, Sales voucher **23**, Day Book **6**, Balance Sheet **7**, Receivables **5**, Sales Register **4**, Dashboard **0**. Each button is a `div onClick` with a `title` of "key — label" [C `widgets.tsx` ButtonPanel]; the F4 to F10 block is Tally's order, frozen.
* **For A:** exactly right, and the best thing the product does for a switcher. It also carries what Tally does not show: ghost recall, Alt+C new ledger, Alt+K hold field, hold-Alt reveal.
* **For B:** the header says "Button bar"; the caps are unreadable glyphs (⌥B, ⌘L, ⌥T, ⌘F7); labels are Tally shorthand ("Cmp Info", "Other Vch", "As Voucher", "Hold field", "After: new", "Esc Quit"); nothing says these are clickable; at rest it is the busiest column on the screen. On a four-button report it still costs 152px.
* **Relevance and clutter for A:** the voucher screen lists the ten voucher-type keys (F4 to F9, Alt+F5, Alt+F6, Alt+F7, Ctrl+F7) beside thirteen editing keys (23 in all); two entries (Batch, Line GST) are shown greyed when they do not apply. That is honest, and acceptable for A.
* **Do not "fix" it by listing keys that do not fire.** `left-nav-spec.md` made "a keycap never over-promises" a rule. Keep it: the rail shows what this screen accepts, nothing more.
* **Recommendations:** (1) make it **collapsible to a thin strip** with the state remembered (same pattern and placement as Hide menu); default **open** for A, **collapsed** for B by role preset (N8); (2) title it **"Shortcuts"** with a `?` that opens the sheet; (3) add **everyday tooltips** to labels (section 7); (4) B's primary actions ("New invoice", Print, Export, Filter, Period) belong in a **top-right action bar** (Wave A's "+ New"), not in a key list; (5) **Wave B (configurable)** must follow the frozen-key rule in section 10; (6) fix semantics (N12).

### 4.8 Consistency of keys

| Key | Means | Surprise for | Fix |
|---|---|---|---|
| F4 to F10, Alt+F5/F6/F7, Ctrl+F7 | New voucher, **only on Gateway and menus** | B and A on a report: nothing happens | "+ New" binds the list's own key (4.4) |
| Ctrl+A | **Accept**, no prompt (Tally-faithful, frozen) | B: Select All in a narration box **saves the voucher**; there is no Ctrl+S [C: no `Ctrl+S` binding anywhere; Ctrl+Enter only in Print Config] | N10: add Ctrl+S as a second Accept; keep Ctrl+A; the undo toast already exists as the safety net |
| Esc | Back / close; **on the Gateway "Quit ?" with Yes preselected** (`window.close()`) | B | Keep; default focus on **No** for the Gateway quit in the desktop build (low risk to A: Esc then Y still works) |
| Esc on a dirty voucher | "Quit ? (entries will be lost)" **Yes preselected** | A (fast), B (Enter = lose it) | Align copy with the new "Leave this entry?"; consider default "Stay" once A's habit is checked (N7) |
| Esc on a master form | **Silent discard** [C `forms.tsx:248`] | Both | N7 (prompt when any field changed) |
| Esc in an altered voucher | No prompt: the guard is `dirty && !editing` [C `VoucherScreen.tsx:2855, 3244`] | Both | N7 |
| Ctrl+K / Alt+G | Go To (the pill shows Ctrl+K on a Mac, Alt+G elsewhere) | none | none |
| `/` or Ctrl+F | Filter this report | none | none |
| Bare letters | Gateway letters; **the rail stops them while focus is inside it** (and they type into the filter) | none | Correct; keep (Phase 25 GOTCHA) |
| Ctrl+\ | Hide / show the left menu | AltGr layouts have no chord by design | none |

### 4.9 Data-safety cues

* **Unsaved work:** Esc prompt on Voucher, Payroll and Attendance vouchers and Bulk Entry; **none** on masters, alterations, Import Data. **No `beforeunload`** (grep over `src/` and `electron/`: none): closing the tab, refreshing, or an Electron window close drops a half-typed voucher without a word. **No draft recovery.**
* **After save:** a toast with Undo ("Voucher 19 saved ⌘Z Undo ×") and an audit-preserving Ctrl+Z [E]. Good. The word is "Voucher", the number is bare ("19"), no invoice identity.
* **Sync:** one dot and a word at the bottom right: synced / N pending / offline / local only · N unsynced [C]. Three problems: **(a)** the sample says "never synced" in the banner while the dot says "synced" [E gateway.png]; **(b)** the tooltip says "Set localStorage.TW_SYNC_URL to enable cloud sync." [C `widgets.tsx`]; **(c)** "160 pending" is a change-log row count, not something a person can act on [E].
* **Backups:** a "Backup due" banner and the Backup / Restore screen; the last backup time is not visible anywhere else. Storage-durability, trial and backup banners stack above the status bar; with the sample banner on top there are up to four strips plus two bars, a vertical cost not measured here.
* **Who am I and which company:** the top bar carries neither; the company name is in each screen's header and the Dashboard sub-title; **no operator name**, **no company switcher** [C `TopBar`]. For a multi-company, multi-operator product that is a trust gap (Zoho puts organisation and user at top-right [M]). Do: N5.

### 4.10 Speed and perceived performance

* **Fast where it counts [E]:** rail click to second animation frame **18 to 19 ms** at depth 2; **16 to 37 ms** at 12 to 15 mounted screens; the repo's own `npm run perf` reports 15 of 15 gated checks [CLAUDE.md].
* **The cost of the stack bug is mostly cognitive, with a real but modest CPU and memory tail.** `AppShell` keeps every stacked screen mounted (`display: none`) [C]; DOM nodes grew **342 to 1,810 at 11 deep and 2,606 at 15 deep** [E]; the JS heap moved 37 to 52 MB. On a 43-voucher sample that is invisible; **on a 20,000-voucher book each mounted report keeps its memoised rows, which I did not measure.** Once A1 keeps the stack shallow this stops mattering; unmounting hidden screens (N17) is a defensive follow-up, with Medium risk (scroll position and filters must survive).
* **Dev cold start** shows "Loading..." for the whole module fetch [README of the harness]; the production build was not measured.
* **Perceived speed gaps that are not about CPU:** the ledger list **opens on focus and covers half the voucher** right after Accept (T4); the Dashboard has no skeleton or last-updated stamp (`ScreenSkeleton` exists for routing only).

### 4.11 Accessibility

* **Strong:** the rail is a real tree (roving tab stop, `aria-label`, polite live region for filter counts, "Skip to left menu"), AA contrast measured in `left-nav-spec.md` (section 1, item 8), a 24px minimum target for rail "+" and star, forced-colors rules in four stylesheets [C].
* **Weak:** the F-key rail (`div` + `onClick`, no role, no tab stop), top-bar items (`span` + `onClick`), Dashboard tiles (`div` + `onClick`), the sync indicator (`span` + `onClick`) are mouse-only and invisible to a screen reader or a Tab key [C `widgets.tsx`, `dashWidgets.tsx:91`]. Three banners are `role="button"` divs with no tab stop.
* **Hover-only reveal** for the rail "+" and star (`navtree.css:150`) is also a keyboard issue only partly solved by `:focus-within`; on touch there is no hover at all.
* **Wave B hazard:** a free colour picker for the accent breaks the contrast table (accent is 5.8:1 on panel in light and 10.0:1 in dark by token); offer a **validated palette**, not a picker. Larger font sizes and the 20.9 px compact row are tied (`visual.assert` baselines).
Do: N12.

### 4.12 Responsive and mobile

* The shell grid is fixed at `212px 1fr 152px` (`styles.css:597`, `navtree.css`), and the content column would not shrink below **574px** on any of the three screens I tried (Gateway, Dashboard, Day Book). Measured **scroll width 938px (212 + 574 + 152) at both the 768 and the 390 viewport** [E probe 3]: **no horizontal fit below 938px**. The only responsive rules are the Go To pill hiding under 1100px and the Dashboard row stacking under 900px [C `styles.css:551, 1508`].
* At **1024px** the Gateway, rail and F-key rail all fit but the content column is 660px; at **1280px** it is 916px [E].
* At **390px** the rail alone is 54% of the screen and the sample banner wraps to four lines [E w390-dashboard.png].
* Zoho and Vyapar both have phone apps [M]. B's most frequent phone tasks are read-only (cash, who owes me, an invoice's status) plus "create a quick invoice". Do: N11 with a first slice that is CSS only.

### 4.13 Help and learning

* **A:** the `?` sheet is searchable and context-sorted; hold-Alt badges keys on the elements they drive; Getting Started ticks itself; Go To teaches the second vocabulary by showing it ("New Sales (invoice)") [C, E]. This is better than anything Zoho offers a Tally user.
* **B:** F1 is the key sheet (a list of shortcuts); there is **no contextual help, no tour, no "what is a voucher", no link to documentation**; the only guided path is the keyboard script in the sample banner. The Welcome promises "Everything is saved on this device first" but nothing explains sync, backup or what a Financial Year is.
* Do: N14 (help drawer, 60-second first-invoice tour on the sample, a "Where is X?" mapping page for Zoho / Vyapar / Busy users, a key map for Tally users), N10 (progressive key coaching).

### 4.14 Reports: the Reports Center

* **Today:** reports are reached from the Gateway's Reports section (six entries), "Display More Reports" (16 entries, five of them submenus), the rail's Reports group (eight plus "More reports…"), Go To, and Saved views. Counts from the spec's route audit: the five report submenus list **73 entries between them** (Account Books 11, Inventory Books 12, Statutory 25, Display 16, Payroll 9; a few repeat, and Display's 16 include links to four of the others) [C `left-nav-spec.md` evidence 5]. **No single catalogue, no categories, no favourites, no "last visited", no search that is *only* reports.**
* **What Zoho shows** [Z]: Reports Center with a left panel (Home, Favorites, Shared Reports, My Reports, Scheduled Reports, Advanced Analytics) and **report categories** (Business Overview, Sales, Inventory, Inventory Valuation, Receivables, Payments Received, Payables, Purchases and Expenses, Taxes, Banking, Projects and Timesheet, Accountant, Currency, Activity, Automation [owner]); a table "All Reports (102)" with star, name, category, created by, last visited; search; "Create New Report". Zoho's help adds: layouts for report PDFs, chart view, scheduling by email, sharing with users and roles, role-based access; custom reports from a parent module, from an existing report, with date range, group-by, advanced filters, columns added with "+" and reordered by drag, **permissions Only Me / Selected users and roles / Everyone**, admins can view, edit, clone or delete others' reports, and **plan limits (10 on Standard to unlimited)** [Z].
* **A mapping that fits what we have** (screens exist; this is a catalogue, not new engines):

| Category | Our reports (examples) |
|---|---|
| Business overview | Balance Sheet, Profit & Loss, Cash Flow, Funds Flow, Ratio Analysis, Top-N (MIS), Budget vs Actual, Profitability |
| Sales | Sales Register, Top customers / items (MIS), Item-wise vouchers, Profitability by item |
| Receivables | Receivables (bills), Ageing, Reminder letters, Interest calculation, Bill-wise on a ledger |
| Payables | Payables, Ageing (payables side) |
| Purchases and expenses | Purchase Register, Payment advice, Group Summary for expense groups |
| Inventory | Stock Summary, Godown summary, Stock ageing, Reorder status, Movement analysis, Stock query |
| Inventory valuation | Stock valuation methods, Stock summary with basis |
| Taxes | GSTR-1, GSTR-3B, GSTR-2B recon, HSN, GSTR-9, RCM register, TDS reports, TDS quarterly, Form 16A, TCS |
| Banking | Cash/Bank Book, Reconciliation, PDC register, Deposit slip, Cheque register |
| Accountant | Trial Balance, Day Book, Journal register, Ledger vouchers, Group summary, Exception reports, Data check |
| Activity | Edit Log (audit trail), Cancelled / deleted vouchers, Sync mismatch report |
| Payroll and HR | Pay sheet, Payslip, Statutory, Attendance, Leave |

* **"Create New Report"** should be **"Save as report"** on top of saved views plus the new row filters (A6): name, category, visibility. A free-form report builder is a large piece of work that competes with keyboard speed and would duplicate F12 plus added columns plus basis. Say so out loud to the owner.
* **Scheduling and sharing** (Zoho's scheduled / shared reports) need the backend to send mail and a visibility model: **later** (N15).
* **Don't copy Zoho's per-plan report cap**: our plan limits are display-only today and a cap here would be a new thing to explain.

---

## 5. The navigation stack: verification, and the policy to implement

### 5.1 Verified numbers [E, probes 1 and 1b]

| Scenario | Result |
|---|---|
| F8 (nothing typed), then 6 rail clicks (Sales Register, Receivables, Payment Received, Balance Sheet, Day Book, Receivables) | Crumb **8 deep**: `Gateway › Voucher › Register › Outstandings › Voucher › Balance Sheet › Day Book › Outstandings`; 8 screens mounted |
| Then 3 Go To jumps | **11 deep**, DOM nodes 342 to 1,810 |
| Esc presses to reach the Gateway (from 11 deep) | **10** |
| F8 (nothing typed), then one rail click on Balance Sheet | `Gateway › Voucher › Balance Sheet`; one Esc returns to the empty voucher |
| **No voucher at all**, Go To three times | `Gateway › Balance Sheet › Trial Balance › Day Book` (**Go To always pushes**) |
| Rail click at depth 2 vs 12 to 15 (second animation frame) | 18 to 19 ms vs 16 to 37 ms |
| Own company, Go To Dashboard then Sales Register then Receivables | `Gateway › Dashboard › Register › Outstandings` (the same pile-up with no voucher) |

### 5.2 Diagnosis: confirmed, with three additions
1. **Confirmed:** `useRail.go()` to `planNav` pushes whenever any non-Gateway screen `mayHoldWork`, which is true for every `create`-kind screen regardless of content. A voucher that was just **accepted** (it resets to a blank next voucher and stays in the stack) is the common trigger.
2. **Go To is a second, unguarded source.** `CommandPalette.go()` calls `st.push(to)` unconditionally [C `CommandPalette.tsx:179`]; the policy the task says Go To "shares" is only shared by the favourite slots and the rail. Fixing `planNav` alone leaves the pile-up in the Go To path.
3. **The crumb problem is two problems:** length (A1) and meaning (F-03, F-04).
4. **"Voucher stays after Accept" is intended** (batch entry, After: new / Alt+N); the fix is to stop treating a *pristine* form as work, not to change the screen.

### 5.3 The dirty signal: what exists and what must be added

| Screen | Has a dirty test? | Quit prompt on Esc? | What the stack needs |
|---|---|---|---|
| Voucher (new) | Yes: `dirty` (`VoucherScreen.tsx:1526`: any ledger / item / party / narration) | Yes ("Quit ? entries will be lost") | Reuse: expose `dirty` to a registry |
| **Voucher (alteration)** | The same `dirty`, but the Esc guard is `dirty && !editing` | **No** | A `touched` flag (changed since load); until then an alteration is dropped on Leave exactly as on Esc: consistent but unsafe; decide deliberately |
| Payroll / Attendance voucher | Yes (`payroll/vouchers.tsx`) | Yes | Reuse |
| Bulk Entry | `rows.length > 0 && !done` (`BulkEntry.tsx:1038`) | Yes | Reuse |
| **Master forms** (`useTallyForm`, every master screen) | **No**, but `values` and `initial` are in the hook (`forms.tsx:71,105`) | **No** (Esc calls `onCancel()`) | `dirty = values differ from initial`; free to add once, in the hook |
| Import Data (a plan being previewed) | `plan !== null` | Esc clears the plan | Treat a plan as dirty |
| Dashboard customise, Saved-views dialogs | Modal / panel owned by the screen | n/a | Out of scope |

Implementation shape (suggestion, builder's call): a module-level registry `markDirty(screenKey, () => boolean)` registered by those screens (the **global** key stack is not involved, and no global handler toggles `active`, per the Phase 25 GOTCHA); `planNav` consults it. Keep `mayHoldWork` as the **fallback for a screen that has not registered** (conservative, today's behaviour), so a forgotten screen can never silently lose work.

### 5.4 Policy (a decision table)

Sources: **R** = destination navigation (left rail rows, Pinned, saved views, favourite slots Ctrl+Alt+1..9, Dashboard tiles and Quick Entry, "+ New", universal "+"); **G** = Go To; **K** = Gateway letters and menu rows (Tally model); **D** = drill-down inside a screen (Enter on a row, F12 panels).

| Situation | R | G | K | D |
|---|---|---|---|---|
| Target is already on top | stay (focus the screen) | stay | n/a | n/a |
| Target already in the stack, everything above it clean | pop to it | pop to it | n/a | push |
| Target already in the stack, something above it dirty | ask (below) | **push** (peek and return, A's habit) | n/a | push |
| Stack is clean | **replace**: `[home, target]` | **push** | **push** | push |
| Something in the stack is dirty | **ask**: "Leave this entry? Entries will be lost." **Stay** (default) / **Leave**; Leave replaces, Stay does nothing | **push** on top (as today) | push | push |
| Depth would exceed ~6 | n/a (replace keeps it at 2) | collapse the clean middle entries, keep Home and the dirty one | n/a | n/a |

Why this split: A uses Go To to *peek and return* (Esc walks back), and the Gateway letters are his tree, so both stay Tally-faithful. B uses the rail to *go somewhere*, so it replaces. De-dup removes `Register › Outstandings › Register › Outstandings`. **Esc from a replaced destination goes Home** (and Home is the role's home once N8 lands).

### 5.5 Crumb
Last four segments with a leading ellipsis, full path in `title` (as specified); **segments stay non-clickable** (a click would pop several screens and bypass each screen's own Esc guard: spec 3, decision 6). Put the **object** in the segment (F-03): `Sales Register`, `Receivables`, `Sales No. 19`, not `Register`, `Outstandings`, `Voucher`.

### 5.6 Assertions and a harness recipe to add
* Update `favourites.assert` **deliberately**: `planNav` returns `replace / push / ask / pop-to / stay` and the table in 5.4 becomes the test table.
* A `rail-walk` recipe: pristine F8 then six rail clicks must leave the crumb at most `Gateway › X`; Go To the same screen twice must not grow the crumb; a half-typed voucher plus a rail click must show the dialog and **not** change the stack on Stay.
* A drift guard that every screen in `HOLDS_WORK` or of kind `create` registers a dirty test.

---

## 6. Where we are already better, and where we are not

### Better than Zoho, or than Tally, or than both [J, with [M] noted]
| Strength | Than whom | Evidence |
|---|---|---|
| Repeat-entry speed: ghost recall (Alt+R), amount shorthand (`50000/-`, lakh / crore), `;snippet` narration, Bulk Entry paste, batch hold (Alt+K) | Both | CLAUDE.md Phases 20 to 23, [E] voucher-sales.png |
| Go To: tiered ranking, aliases in both vocabularies, frecency, stable top row | Tally's Alt+G; Zoho's global search [M] | [C] `paletteRank.ts` |
| Offline-first, local data, scheduled backup, restore | Zoho (cloud only) [M] | [C] |
| One tree for compliance (GST returns with portal JSON, TDS / TCS registers, e-invoice records, bank-statement import, reconciliation) | Zoho India for TDS depth [M] | [C] parity board |
| Operator rights per screen area, an edit log, **undo that keeps the audit trail** | Both [M] | [C] `access.ts`, [E] toast |
| A migration path built in: Tally XML import, "Compare with Tally Data" checker | Zoho | [C] |
| Sample company, self-ticking Getting Started, welcome with three honest paths | Both | [E] |
| Dual surface: Gateway + rail + F-key rail on one screen; hold-Alt reveal | Both | [E] |
| Customisable Dashboard, per-operator, keyboard-complete | Tally (no dashboard) | [C] |
| Light and dark, two densities, AA contrast measured | Tally | [C] spec |

### Worse than Zoho (what B will notice in the first hour) [J]
Discoverability (no "+", no list-first modules), empty states, onboarding of own companies, document lists with status and balance, sending or sharing an invoice (**I found no WhatsApp, email, PDF-link or share path anywhere in `src/`**), Reports Center, charts, scheduled reports, mobile, search by invoice number, payment links, estimates / quotes (we have orders, not a quote chain), who-am-I chrome.

---

## 7. Terminology bridge: the glossary

How to read: column 2 is the word as printed in the Gateway, the F-key rail or a report today; column 3 is the everyday word B uses; column 5 is the treatment. **"alias"** means the search layer already knows the pair (nav aliases or palette `DOMAIN_ALIASES`); the gap is the **display** layer. No proposed string contains the word "Tally" (brand rule): the sub-label is "also called".

| # | Gateway-style word (as printed) | Everyday word | Seen in | Treatment |
|---|---|---|---|---|
| 1 | Gateway | Home | Gateway title, crumb, Esc target | Crumb "Home" when the role's home is the Dashboard; keep "Gateway" on the menu itself |
| 2 | Voucher | Entry, transaction | Crumb "Voucher", header "Accounting Voucher Creation" | Title = what was clicked ("New Sales Invoice"), sub-title "Sales voucher No. 19" |
| 3 | Sales (F8) | Sales invoice, tax invoice | Rail row (alias ok), form header | Header dual-label |
| 4 | Purchase (F9) | Bill, purchase bill | Rail "Purchase Invoice"; alias `bill` ok | "+" menu: "Bill" first, "Purchase" as sub-label |
| 5 | Receipt (F6) | Payment received | Rail ok; Go To "New Receipt" | "+" menu: "Payment received"; keep F6 |
| 6 | Payment (F5) | Payment made, expense | Rail ok | "+" menu: "Payment made" |
| 7 | Contra (F4) | Transfer between cash and bank | Rail "Transfer (Contra)" ok | Keep dual |
| 8 | Journal (F7) | Journal entry, adjustment | Rail, Gateway | Tooltip: "Adjustment between accounts" |
| 9 | Credit Note | Sales return, credit memo | Rail ok; alias ok | Sub-label |
| 10 | Debit Note | Purchase return, vendor credit | Rail ok; alias ok | Sub-label |
| 11 | Ledger | Account, customer, vendor | Crumbs "Ledger Vouchers", "Select" | Masters row "Ledgers" + hint "customers, vendors, accounts" (alias `customers` ok) |
| 12 | Sundry Debtors | Customers (owe you) | Group names, ledger forms | Field hint under "Under" |
| 13 | Sundry Creditors | Vendors (you owe) | Same | Same |
| 14 | Group / Under | Category / Belongs to | Ledger form | Field hint: "Which group this account belongs to" |
| 15 | Alter | Edit | Gateway "Alter", "Alter Company" | Rail already says "Company Details"; keep Alter on the Gateway |
| 16 | Create | New, add | Gateway "Create" | "+" menu says "New" |
| 17 | Particulars | Party or description | Day Book, registers | Column-head tooltip "Party or account" |
| 18 | Vch Type / Vch No. | Type / Number | Day Book | Tooltip |
| 19 | Narration | Notes, memo | Voucher, registers | Field hint "Notes (narration)" |
| 20 | Dr / Cr | Debit / Credit; "they owe you / you owe" | Balances ("95,939.00 Dr") | Party balance hint: "Dr: this customer owes you" |
| 21 | Outstandings | Unpaid invoices / bills | Crumb | Crumb "Receivables" / "Payables" |
| 22 | Bills Receivable / Payable | Receivables / Payables | Report header | Header + sub "unpaid invoices"; alias ok |
| 23 | New Ref / Agst Ref / On Account | New bill / Against bill / Advance | Bill-wise panel | Labels with hint |
| 24 | Ageing | Overdue by 30 / 60 / 90 days | Report | Alias `overdue` ok; sub-label |
| 25 | Day Book | Daily transactions | Rail, Gateway | Alias ok; sub-label "all entries by date" |
| 26 | Trial Balance | Trial Balance | Report | Keep; category "Accountant" |
| 27 | Profit & Loss A/c | Profit & Loss | Gateway, report | Drop "A/c" on new surfaces only |
| 28 | Stock Item | Item, product | Masters | Alias ok |
| 29 | Stock Summary | Inventory summary | Rail, report | Alias ok; sub-label |
| 30 | Godown | Warehouse, location | Masters | Alias ok |
| 31 | Stock Journal | Stock transfer, consumption | Voucher | Alias ok |
| 32 | Physical Stock | Stock count | Voucher | Alias ok |
| 33 | Delivery Note | Delivery challan | Voucher | Alias `challan` ok |
| 34 | Receipt Note | Goods received note (GRN) | Voucher | Alias ok |
| 35 | Cost Centre | Project, department, branch | Masters, reports | Alias ok |
| 36 | Voucher Type | Document series, numbering | Masters | Alias ok |
| 37 | Optional / Memorandum / Post-dated | Draft / not in books / future-dated | Registers | Sub-labels |
| 38 | Period (F2 / Alt+F2) | Date range, financial year | F-key rail | Tooltip |
| 39 | Accept (Ctrl+A) | Save | F-key rail, Welcome | Keep Accept; add Save as the tooltip and Ctrl+S as a second key (N10) |
| 40 | Quit (Esc) | Close, back | F-key rail, prompts | Tooltip "Close and go back"; keep the key |
| 41 | Configure (F12) | Customise, options | F-key rail | Tooltip |
| 42 | Features (F11) | Modules, settings | F-key rail | Rail already has Settings > Features |
| 43 | Cmp Info (Alt+F3) | Company menu | F-key rail | Tooltip |
| 44 | Other Vch (F10) | More entry types | F-key rail | Tooltip |
| 45 | Company | Business, organisation | Everywhere | Keep |
| 46 | Mailing Name | Billing name | Company form | Hint |

**Where the treatments go** (each is one place in the code): (a) rail rows: `title` plus the existing "via" hint (`nv-via`); (b) report column heads: `title` attributes in each report's header row (shared in `ReportShell`); (c) F-key rail labels: `title` in `ButtonPanel` (today `"key — label"`); (d) form fields: `hint` in `FieldSpec` (`useTallyForm`); (e) page titles: the screen header; (f) palette: `DOMAIN_ALIASES` (exists); (g) the "+" menu and Reports Center: everyday-first labels.

---

## 8. The recommendation register

**Columns:** Who: A = Tally switcher, B = owner, Both. Impact 1 to 5 (5 = changes whether the person succeeds). Effort S (days), M (a couple of weeks), L (more). **Risk to Tally users:** None / Low / Med / High with the reason. **Covered by** is the table it sits in: Wave A, Wave B, or NEW (in neither). "Where" names the component, not a design.

### Wave A (as asked), with the analyst's conditions

| ID | Recommendation | Who | Impact | Effort | Risk to Tally users | Where |
|---|---|---|---|---|---|---|
| A1 | Stack and breadcrumb: replace on destination, real dirty signal, de-dup, short crumb with object names (section 5) | Both | 5 | M | Low: Go To and Gateway letters keep push; `planNav` tests change on purpose | `useRail.ts`, `favouritesCore.planNav`, `store.ts`, `CommandPalette.go`, `StatusBar`, dirty registry |
| A2 | Rows open the list; "+" creates (Sales, Purchases, Banking, Accounting rows), "+" visible on hover **and** on the current row | B | 5 | M | Low-Med: the F8 keycap moves to the "+"; `sidenav.assert` literal table changes; the pattern (`to`/`create`/`also`) already exists for masters | `navTree.ts`, `NavTree.tsx`, `scripts/assert/sidenav.assert.ts` |
| A3 | "+ New" split button top-right of lists and reports, bound to that list's own F-key | B | 5 | S-M | Low: a report already ignores F8; additive | `ReportShell` (`reports.tsx:191`), `ButtonPanel` |
| A4 | Universal "+" in the top bar, fed by the palette's Create list | B | 4 | M | Low | `widgets.tsx` TopBar, `CommandPalette.tsx` (Create) |
| A5 | Reports Center: categories, search, favourites, recent, "Save as report" | B | 4 | L | Low | New screen: `SCREEN_AREA`, router, crumb, Gateway menu, palette, rail (new-screen checklist) |
| A6 | Saved filters: **row criteria** (party, status, date, amount), system views (All, Unpaid, Overdue, This month), favourite, visibility, **synced not device-local** | Both | 4 | L | Low-Med: F12 saved views must keep working unchanged | `core/savedViews.ts`, `reportViews/*`, `useReportFilter.ts`, backend change-log |
| A7 | Fewer rows: fold Receivables / Ageing / Reminders / Register into views of one Invoices (and one Bills) list | B | 4 | M | Low | `navTree.ts`, list screen |
| A8 | **Gap in A:** a flat "All" level on the registers (they are months, then a month's vouchers) | B | 4 | S-M | Low | `reports/financial.tsx:321` |

### Wave B (analysed only for its constraints)

| ID | Recommendation | Who | Impact | Effort | Risk to Tally users | Where |
|---|---|---|---|---|---|---|
| B1 | Configurable shortcut buttons: hide / reorder / relabel **optional** slots and add favourites; never remap a Tally key; offer presets, not free-form | Both | 3 | M | Med if the F4 to F10 block can be reordered; None if frozen | `ButtonPanel` / per-screen `Btn[]`, `favourites.tsx` |
| B2 | Admin Settings: theme, accent (validated palette), fonts (bundled, offline-safe), size | Both | 3 | M | Low-Med: density, row height and `visual.assert` baselines | `state/theme.ts`, `styles.css` tokens |
| B3 | Contract: **workspace** preferences (admin) are synced data; today `TW_THEME_*` is per-device `localStorage` | Both | 3 | M | None | New synced prefs doc, `SCREEN_AREA` for the Settings screen |

### NEW

| ID | Recommendation | Who | Impact | Effort | Risk to Tally users | Where |
|---|---|---|---|---|---|---|
| N1 | **Guided start**: two-step company creation (name, state, GST yes / no with GSTIN, financial year; the rest later), a persistent "Setup" card on the Dashboard until done, empty states with one sentence and a "+ New", clickable sample prompt | B | 5 | M | None | `CompanyCreate.tsx`, `Dashboard.tsx`, `GettingStarted.tsx`, empty states in `ReportShell` |
| N2 | **Status-aware document lists** (Invoices, Bills): number, party, date, due, amount, balance, status chip; row actions Record payment, Print, Share | Both | 5 | M | None | New list over `outstandingBills`; `billwise` feature gate |
| N3 | **Post-save moment**: non-modal strip "Invoice saved: Print · Share · Record payment · New"; then share (PDF, WhatsApp, email) | B | 5 | M (strip), L (share) | None: non-modal, so A can ignore it; the Ctrl+A flow is unchanged; offer an off switch in Settings | `VoucherScreen.tsx`, `feedback.tsx`, print, backend for links |
| N4 | **Owner Dashboard**: "as of today" with a Period-end switch; net profit and expenses; trend on; overdue customers (top 5); GST due with date; a "Needs attention" card (overdue, unreconciled bank, backup due, sync mismatches) | B | 5 | M | None | `dashboardLayout.ts` defaults, `dashboardData.ts`, `dashWidgets.tsx` |
| N5 | **Identity and status cluster** (top right): company switcher, operator, sync in plain words ("Saved on this device", "3 changes waiting", "Offline"), last backup; fix sample "never synced" vs "synced"; replace the `localStorage` tooltip | Both | 4 | S | None | `TopBar`, `StatusBar`, `sync.ts` status |
| N6 | **Dead controls**: wire or hide Data, Exchange, Import, Share; never ship "Not available in MVP" | Both | 3 | S | None | `widgets.tsx` TOP_ITEMS |
| N7 | **Safe-leave kit**: `beforeunload`, draft recovery ("Resume your unsaved invoice?"), master-form and alteration prompts, one copy and one default for Quit / Leave | Both | 4 | M | Low: A's Esc then Y still works | `forms.tsx`, `VoucherScreen.tsx`, IndexedDB drafts |
| N8 | **Role-based home**: owner = Dashboard home, Menu view, collapsed F-key rail; accountant = Gateway home, Quick access, rail open; brand click goes Home; Esc lands on the role's home | Both | 4 | S-M | None: defaults only, one click to switch | `navStore.ts`, `AppShell.tsx`, operator role, `widgets.tsx` |
| N9 | **Dual-label vocabulary** (section 7): titles carry the clicked word, tooltips on column heads and rail labels, "also called" sub-labels; no vocabulary switch yet | B | 4 | M | Low | each screen header, `ReportShell` heads, `ButtonPanel`, `FieldSpec.hint` |
| N10 | **Progressive key coaching** (a dismissible "next time press F8" after the third click on a keyed row) plus **familiar aliases**: Ctrl+S = Accept alongside Ctrl+A | B to A | 3 | S | None | rail click handler, `keys.ts`, entry screens |
| N11 | **Responsive shell**: under 1100px the F-key rail becomes a "Keys" popover; under 900px the rail is a drawer; under 640px read-first screens (Dashboard, Receivables, Invoices, quick invoice) | B | 4 | L (slice 1 is S: CSS only) | None above 1100px | `styles.css` grid, `AppShell.tsx`, `NavRail.tsx` |
| N12 | **Real buttons and names**: F-key rail, Dashboard tiles, top-bar items, sync indicator; `aria-current` on the lit row; keep `title` as hint, not as the name | Both | 3 | S-M | None | `widgets.tsx`, `dashWidgets.tsx` |
| N13 | **Go To for B**: voucher lookup by number / party / amount as a new scope; type chips (Report, New, Ledger); keep tiers and the stable top row | B | 4 | M | Med if tiers move; None as a new scope | `CommandPalette.tsx`, `paletteRank.ts` |
| N14 | **Help**: a help drawer (F1 keeps the key sheet), a 60-second first-invoice tour on the sample, a "Where is X?" page for Zoho / Vyapar / Busy users and a key map for Tally users | Both | 3-4 | M | None | New screen + palette + Gateway "Company" menu |
| N15 | **Report delivery**: schedule, email and share a report as PDF with visibility | B | 3 | L | None | Reports Center, backend mail |
| N16 | **Estimates / quotations**: a quote to order to invoice chain | B | 3 | L | None | New voucher base; check demand first |
| N17 | **Unmount hidden stack screens** beyond the top three, with scroll and filter snapshots | Both | 2 | M | Med | `AppShell.tsx` |
| N18 | **Rail polish**: an accordion option (one group open), Pinned shown once, first-visit hint for the "+" | B | 2 | S | None | `NavRail.tsx`, `MenuView.tsx` |

---

## 9. The top 10 NEW upgrades, in the order I would do them, and why

| # | ID | What | Why this, why here |
|---|---|---|---|
| 1 | **N1** | Guided start + setup card + real empty states | The first ten minutes decide whether B adopts. Today the best onboarding in the product (the sample) ends at the door of a 29-field form and an all-zero screen. Cheap, no risk to A, and it makes every later feature discoverable. |
| 2 | **N2** | Status-aware Invoices / Bills lists with Record payment | Wave A's "click Sales, see all sales" has nothing to land on without it: the register is months, with no status or balance. It is also what makes "+ New", the filters and the Reports Center worth using. Build it **with** A2 / A3, not after. |
| 3 | **N4** | Owner Dashboard on the right clock, with profit and attention | The Dashboard is the one screen B can use with no training; today it shows period-end figures in the future and hides the trend, the GST payable and the top customers. All the engines exist (`monthlyTrend`, `rankParties`, `gstPayableCash`): this is defaults and one new card. |
| 4 | **N3** | The post-save moment, then sharing | B's actual finish line is "send it to the customer", not "voucher saved". I found no share path at all. The strip is cheap; PDF / WhatsApp / email links are the long pole and need the backend. |
| 5 | **N5 + N6** | Honest status and no dead controls | Small effort (S) for the biggest credibility win: four disabled top-bar items, a banner and a dot that contradict each other, and a tooltip with a developer instruction. Do these first even though they rank fifth by impact. |
| 6 | **N7** | Safe-leave kit | After A1 removes the pile-up, the only protection against losing an entry is Esc on three screens. No `beforeunload`, no draft recovery, masters discard silently. One lost invoice costs more trust than any feature earns. |
| 7 | **N8** | Role-based home and defaults | The same company is opened by an owner and an accountant. One flag at first sign-in fixes the home screen, the rail view and the F-key rail for both, and it is the contract Wave B's presets need. |
| 8 | **N9 (+ N10)** | Dual-label vocabulary and progressive key coaching | The bridge exists in search and not on screen. Titles that echo what was clicked and tooltips on column heads turn each screen into a lesson, and the key coaching moves B toward A's speed without a manual. Ctrl+S as a second Accept removes the one surprise that can post a voucher by accident. |
| 9 | **N11** | Responsive shell | B checks cash on a phone; the product has a hard 938px floor. Slice 1 (hide the F-key rail under 1100px, make the left menu a drawer under 900px) is CSS and costs days; the read-first mobile screens come after. |
| 10 | **N12** | Real buttons and accessible names | Wave B will build the configurable rail on `ButtonPanel`; fixing its semantics once now avoids retrofitting a customisable control. It also gives B a Tab path and a screen reader a way in. |

**Next five, if capacity allows:** N13 (Go To by voucher number), N14 (help drawer and tours), N15 (scheduled / shared reports), N16 (estimates), N17 and N18 (stack tail, rail polish).

---

## 10. Notes to the Wave A and Wave B builders

### Wave A: three conditions and the traps

**Conditions (each is a place the wave will quietly disappoint if missed):**
1. **A list must exist to land on** (F-01, F-06, A8, N2). "Sales opens the Sales Register" opens a month matrix. Either add a flat "All" level (A8) with status columns (N2), or the new "+ New" lands on a screen B cannot use to find an invoice.
2. **Saved filters are not saved views** (F-11). The ask maps to Zoho's custom views, which need a **row-criteria model**, system views and **shared, synced storage**. Saved Views today are F12 options and are device-local. Keep the existing views working unchanged; do not rename the old feature.
3. **Go To needs the same fix as the rail** (5.2). `CommandPalette.go()` pushes unconditionally; a stack fix that only touches `useRail` leaves the pile-up on the other path.

**Traps found in the code:**
* `activeRow` highlights the row whose `to` equals the screen on top. If "Sales Invoice" now opens the list, the **voucher screen must keep the row lit through `also`**, as masters already do.
* The **F8 keycap** is printed on the Sales Invoice row today; once the row opens a list, the keycap belongs on the "+" (the key creates). `sidenav.assert.ts` is a literal 108-row table: change it in the same edit.
* **New screens** (Reports Center) need a `SCREEN_AREA` entry, a router case, a crumb, a Gateway or menu entry, a palette entry and the Go To list; the assertion suites fail otherwise.
* **Feature gates:** with the bill-wise feature off there is no Unpaid / Overdue view; the row should say why, not vanish.
* **`Register` is two-level** inside one screen (Esc returns from a month to the matrix without touching the nav stack). A flat "All" level must keep that Esc behaviour.
* **Do not claim bare `+` globally**: it is the rail's own key and the Gateway's keys must not run behind it.
* **Pinned rows and Recents** key on `screenKey`; the `stay` and de-dup rules must use the same key or a pin would reopen a screen that is already up.

### Wave B: the contracts Wave A should reserve

* **Configurable shortcuts.** The F-key rail's data is already one `Btn[]` per screen; the Favourite Shortcuts screen (Ctrl+Alt+1..9) is the existing configurable surface. Reserve a *presets* concept (Tally preset = today; Owner preset = New invoice, New bill, Record payment, Receivables, Reports) rather than free-form remapping. **Frozen:** no Tally key changes meaning or position; configuration can hide or reorder only the optional second block.
* **Admin Settings.** `state/theme.ts` holds scheme and density in per-device `localStorage` (`TW_THEME_SCHEME`, `TW_THEME_DENSITY`). "The administrator of the workspace" implies **synced workspace settings** (B3) with a per-user override. The accent is two tokens per scheme (`--accent`, `--on-accent` with soft and hot variants): offer a **validated palette**, never a free picker. Fonts must be **bundled** (the rupee sign exists only in Plex `latin-ext`, which the app already bundles); a font list that loads from the network breaks offline-first. Font size and the compact row (20.9 px) interact: the `visual.assert` baselines need a size dimension. The Settings screen needs its own `SCREEN_AREA` entry.
* **Where the top-bar toggles go.** Light / Compact leave the top bar for Settings; the identity and status cluster (N5) takes their place.

---

## 11. How to know it worked

### Measurable, with the tools already in the repo
| Check | Target | How |
|---|---|---|
| Crumb depth after pristine F8 plus 6 rail clicks | at most 2 (`Gateway › X`) | `rail-walk` harness recipe (5.6) |
| Esc presses from a rail-reached destination to Home | 1 | same |
| Rail click then Go To to the same screen twice | no growth | same |
| Half-typed voucher plus rail click | dialog; Stay changes nothing | same |
| Rail click to second frame at depth 12 | at most 40 ms on the sample | probe 1b logic |
| Own company first screen | a visible next step (checklist or CTA), no all-zero screen | `npm run shots`, new recipe |
| Top bar | zero controls reading "Not available" | `shots` plus DOM assertion |
| Width 768 and 390 | no horizontal scroll | geometry probe (probe 3) |
| Search by invoice number | result in Go To | recipe |

### Ten sessions, five per persona, before Wave C is scoped
* **B tasks (no keys allowed):** make an invoice for a new customer; find who is late; record a payment; find invoice 18; save "overdue customers" for later. **Measure:** time to first invoice (target under 3 minutes with the sample, under 6 with an own company), wrong clicks, whether he finds "+", whether he can say what "voucher" means.
* **A tasks (keys allowed):** repeat his day's ten entries; Go To a ledger mid-voucher and return; open Day Book, filter, back to Gateway; hit Esc ten times in a row. **Measure:** whether any key has moved, any Esc surprised him, any new surface got in his way.
* **Instrumentation worth adding (counts only, local):** rail vs Go To vs key vs "+" opens per screen, how often the "Leave this entry?" dialog is answered each way, how often the F-key rail is collapsed. These are what tell us whether role defaults (N8) were right.

---

## Appendix A. What the probes measured

| Probe | Result |
|---|---|
| 1. F8 then rail walk | Crumb 8 deep after six clicks; 10 Esc presses to Home from 11 deep |
| 1. Pristine F8 then rail Balance Sheet | `Gateway › Voucher › Balance Sheet`; one Esc returns to the voucher |
| 1b. Go To x3, no voucher | 4 deep (`Gateway › Balance Sheet › Trial Balance › Day Book`) |
| 1b. Rail click latency (2 frames) | 18 to 19 ms at depth 2; 16 to 37 ms at depth 12 to 15; DOM 342 to 2,606 nodes; heap 37 to 52 MB |
| 2. Own company, name only | 29 fields; Gateway "No Vouchers Entered", "2 ledgers · 0 stock items · 0 vouchers"; Dashboard six `₹0.00`; Sales Register 12 blank months; Receivables "No outstanding bills"; status bar "160 pending" (harness default) |
| 3. Geometry | 1280: rail 212, content 916, keys 152. 1024: content 660. **768 and 390: scroll width 938** |
| 3. Esc on the Gateway | "Quit ?" with Yes preselected |
| 4. Top bar | 8 items, **4 disabled** ("Not available in MVP"): Data, Exchange, Import, Share |
| 4. F-key rail | Gateway 12; Sales voucher 23; Dashboard 0 |
| 4. After Accept | Toast "Voucher 19 saved ⌘Z Undo ×"; blank No. 20 with the ledger list open; no Print / Share / Record payment |

## Appendix B. Sources

Zoho Books help pages, fetched 2026-10-19:
* Reports Center and report categories: https://www.zoho.com/us/books/help/reports/
* Custom reports (modules, filters, columns, layout, permissions, scheduling, plan limits): https://www.zoho.com/us/books/help/reports/custom-reports.html
* Manage reports (scheduling, sharing, access): https://www.zoho.com/books/help/reports/manage-reports.html (summary came from the search result; the page itself redirected)
* Custom views (default filters, "+ New View": name, favourite, criteria, columns, visibility): https://www.zoho.com/books/help/settings/customization/custom-views.html (from the search result, not fetched in full)

See also `docs/research/zoho-books-reference.md`, written in parallel by another pass (untracked when this was written). From its section 0 I rely on two points only, and they agree with this document: Zoho's saved view stores filter conditions while ours stores F12 options (so "save filter" is a real gap), and Zoho's own FAQ says its sidebar modules cannot be reordered, hidden or renamed (so Wave B should not copy that rigidity; our rail already hides by feature and right).

Owner's description of his Zoho screenshots (Sales group with nine entries, "+ New" split button, "..." menu, "All Invoices" title dropdown, universal "+" mega menu with GENERAL / INVENTORY / SALES / PURCHASES / BANKING, Reports Center with the left panel, 102 reports, category list, search, "Create New Report"): in his request of 2026-10-19; the images themselves were not available.

## Appendix C. Artefacts (session scratchpad, not in the repo)

`/private/tmp/claude-501/-Users-Akshay-tallyreplica/9aab9235-f990-4073-bbcb-e40015fe85ee/scratchpad/`
* `zx-before/`: the nine BEFORE shots of the current UI.
* `ux-probe/probe1.mjs … probe4.mjs`: the four scratch probes; `out1/ out1b/ out2/ out3/ out3b/ out4/`: their logs and screenshots (stack walk, empty states, widths 1280 / 1024 / 768 / 390, Go To, F12, Alt+V, Getting Started, dark Dashboard, post-Accept).
