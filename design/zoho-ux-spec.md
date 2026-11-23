# Comsky Books: the Zoho-style UX wave (Wave A), architecture and spec

> **Status:** design spec plus the scaffold it describes, ready for three builders working in parallel. **Author:** architect pass, 2026-10-19. **Scope:** Wave A of the owner's request of 2026-10-19: the module menu with a `+` per row, the `+ New` button on lists, the universal `+`, saved filters and views, the Reports Center with "Create New Report", and the navigation-stack bug (the breadcrumb that grows). **Wave B** (configurable shortcut buttons, the admin Settings window) is **not built here**; section 8 reserves its screens and contracts so it can land without touching Wave A's files.
> **Reads before this:** `docs/research/zoho-books-reference.md` and `docs/design/ux-analysis-2026-10.md` (both written minutes earlier; this spec decides what they only analyse), `docs/design/left-nav-spec.md` (Phase 25; Appendix D of this spec lists exactly what it supersedes), `CLAUDE.md`.
> **Frozen and untouched:** the Gateway tree and every letter in it (one row is appended, section 7.7), every existing shortcut, the F-key rail (152px, its keys, its order), field order, the brand rule (no Tally wording in the product except a migration screen's own name), operator rights (`core/access.ts`), company feature flags, light / dark / compact / comfortable through design tokens (`docs/design/light-theme-spec.md` overlay rules), printed paper. New things are ADDITIVE.
> **Evidence tags:** **[C]** read from code in this session (file named); **[V]** verified by running something (tsc, the assertion suites, the headless harness, a scratch probe); **[R]** from the two research documents; **[J]** my judgement.

---

## 0. The decisions, in one table

| # | Question | Decision | Section |
|---|---|---|---|
| 1 | Menu shape | **12 groups, 64 destinations (was 14 and 108), at most 6 rows a group.** A click on a **group label** opens that module's list (Sales opens the Sales Register, Banking the Cash/Bank Book, Accounting the Day Book, Inventory the Stock Summary, Taxes & Compliance GSTR-3B, Reports the Reports Center); the **chevron alone** expands. Every **row** opens its register or report; 26 rows carry a `+` that opens the matching entry screen. Rows with no entry screen have no `+`. Tally keys stay (9 dim keycaps on rows; the 10 voucher F-keys move to the `+` tooltip). | 1, App. A |
| 2 | One list behind every `+` | `src/ui/quickCreate.ts`: 28 entries, 33 page entries (`QC_PAGES`). The rail `+`, the page `+ New`, and the universal `+` read it; nothing else lists a create. Gating is the left menu's: company feature plus `screenDenial` (so a voucher is judged by `voucherDenial('create')`, the function the store applies). | 2, App. B |
| 3 | Page header | `src/ui/pageHeader.tsx`: **three slots** (`titleExtra`, `actions`, `belowHead`), drawn by ReportShell, PickMaster and the Chart of Accounts. `+ New ▾` (primary create plus its related creates) and a `...` menu (Print, Export CSV / Excel, Configure F12, Save as view, Filter, Import, Bulk Entry, Refresh) sit in `actions`. **No F-key is bound on a list**: the Tally keys inside the dropdown are information, with the same "where it fires" wording the rail uses. | 3 |
| 4 | Universal `+` | A primary-coloured 24px square in the top bar, **right of the Go To pill**, opening a five-column mega menu (Sales, Purchases, Banking + Accounting, Inventory, General + Payroll + CRM). **Chord: `Alt+Q`** (printed `⌥Q` on a Mac). Dead inside a text field, behind an overlay, on a gate screen. Listed on the `?` sheet and in the locked keys through the shortcut registry. | 4 |
| 5 | The breadcrumb bug | Verified, and wider than diagnosed (Go To is a second unguarded source). New policy: decide on **real unsaved work** (a probe each entry screen registers), **replace** the stack when clean, **ask** "Leave this entry?" (Stay default / Leave) when dirty, **pop back** to a screen already in the stack, cap depth at 6 for peeks, and show the **last 4 crumb segments** with the full path in the tooltip. Go To keeps its Tally "peek and return" push. | 5 |
| 6 | Saved filters | A **row filter** (text, number, date, enum conditions, AND / OR, one nested level) that runs on a report's rows **before** it totals them, so totals, percentages, Print and Export all follow it and are labelled "Filtered". Saved **views carry it**, personal (this device) or company-wide (synced, administrators write). Keys **`Alt+F12`** (range) and **`Ctrl+F12`** (value / name) are free and chosen. A live filter is **never persisted silently**. | 6 |
| 7 | Reports Center | Screen `reportsCenter`: Home, Favourites, Recently visited, Shared reports, My reports, then **14 categories** (99 reports, counted per category, gated by feature and rights), search, a star per row (the star **is** the left menu's pin), "Create New Report". The builder (`customReport`) is **a short form, then "set it up in the real report"** with a Save bar, so F12, columns and filters are the real ones. | 7, App. C |
| 8 | Wave B | Screen `settings` registered as a stub with `SCREEN_AREA 'admin'` and linked from nowhere; `Btn` gains reserved `id` / `movable`; every new chord is in one registry (`core/shortcutRegistry.ts`) a settings window can list and lock. | 8 |
| 9 | Proof | A scaffold suite (done, 484 assertions) plus per-builder suites and recipes; acceptance checklist in section 9. | 9 |

---

## A. What was verified (and what the scaffold already changed)

1. **The diagnosis is confirmed [C, R].** `useRail.go()` asks `planNav` (`ui/favouritesCore.ts`), which pushes ON TOP whenever any screen above the Gateway `mayHoldWork`, and `mayHoldWork` is true for every `create`-kind screen (`state/recents.ts` `FORMS`: every voucher, master and Bulk Entry) whether or not anything was typed. A voucher stays in the stack after Accept (it resets to a blank next voucher), so after ONE voucher every later rail click stacks another screen. Two things the diagnosis did not say: **Go To is a second, unguarded source** (`CommandPalette.go()` calls `st.push(to)` unconditionally, so it piles up with no voucher at all), and **the "dirty" signal does not exist** for masters (`useTallyForm` knows `values` and `initial` but never says so) and is switched off for alterations (`dirty && !editing`). The UX probe measured it: pristine F8 plus six rail clicks gave a crumb eight deep, ten Esc presses to get home from eleven.
2. **Every route in this spec compiles [V].** The menu tree, the quick-create tables and the report catalog (Appendices A to C) were written as TypeScript against the real `Screen` union and the real `QuickCreate` type, type-checked, and then checked by script against the Gateway's `MENUS` and the palette: 64 destinations and 28 creates resolve to a union member, a router case and a `SCREEN_AREA` entry; a voucher create's key is that voucher's own F-key; a row's feature equals the Gateway's effective feature for the same destination; every group's click target is its first row's (Reports excepted, by design); **all 108 rows of the old tree are still reachable** (55 stay on the rail, 14 became a `+`, 30 live in the Reports Center catalog, 9 in a Gateway submenu); every Gateway and palette report is in the catalog or on an explicit exclusion list with a reason (Appendix C).
3. **The chord audit [C].** Section 4.4. Alt letters taken today: A B C D E F G H I J K L N O P R S T V W Y and `Alt+2`; free before this spec: M Q U X Z. `Alt+Q` is chosen.
4. **The scaffold is in and green [V].** `npx tsc --noEmit` 0 errors; `npm run build` passes; `npm run assert` runs **65 suites, all green** (the 64 existing plus the new `zoho-scaffold`, 484 assertions); `npm run shots` (nine screens plus Welcome and ledger-create, pinned date, `--strict`) is **byte-identical to the BEFORE shots for every screen except three, and those three differ only by the one appended Gateway row**: `gateway`, `nav-menu-all-open` (the Gateway beside the rail) and `key-sheet` (the Gateway behind the scrim). A scratch probe confirmed: Gateway `O` opens the Reports Center stub, Go To finds it and "New Custom Report", a pushed `settings` renders, Esc leaves each, **the Day Book header is still `t | rf-hint | co` with zero `ph-*` elements**, and no console errors.

### What the scaffold added (so the builders do not touch these registrations)

| Area | Files (new unless marked) |
|---|---|
| Three screens, registered everywhere a screen must be | `Screen` union (`state/store.ts`, edited: `reportsCenter`, `customReport` with `editId?` and `base?`, `settings`, and `register` gained optional `all`), `router.tsx` (route + crumb, edited), `state/recents.ts` (TITLES; `customReport` is a FORMS and NEVER entry, `settings` a SETUP one; edited), `core/access.ts` (SCREEN_AREA: `reportsCenter` and `customReport` `'open'`, `settings` `'admin'`; edited), `ui/favouritesCore.ts` (Go To / Favourite Shortcuts destinations for `reportsCenter` and `customReport`; edited), `ui/modern/CommandPalette.tsx` (Go To entries + aliases + the Gateway key; edited), `ui/MenuScreen.tsx` (Gateway row "Reports Center", `O`, appended to the Reports section; edited) |
| Stubs (a `PlannedScreen`, not `ComingSoon`, so `sidenav.assert`'s one-entry stub allow-list is untouched) | `ui/reportsCenter/ReportsCenter.tsx`, `ui/reportsCenter/CustomReport.tsx`, `ui/admin/SettingsScreen.tsx`, `PlannedScreen` in `ui/scaffold.tsx` (edited) |
| The page header | `ui/pageHeader.tsx` (slots, `HeaderCtx`, `useHeaderCtx`), three stub slot components `ui/header/PageActions.tsx` (B1), `ViewsTitle.tsx` (B2), `FilterBar.tsx` (B2), all returning `null`; integrated once in `ReportShell` (`ui/reports.tsx`, edited), `PickMaster` and `COAScreen` (`ui/masters.tsx`, edited) |
| Slot plumbing | `SlotIndexCtx` / `SlotScreenCtx` in `ui/screenCtx.ts` (edited), provided by `AppShell` and `ScreenRouter`; `state/dirty.ts` (the unsaved-work registry, working); `ui/reportActions.ts` (what the open report offers its header, working) |
| Contracts | `ui/quickCreate.ts` (types, layout, lookups and gates working; tables empty), `core/reportFilter.ts` (types, limits, identity `applyFilter`), `core/savedViews.ts` (optional `filter`, `scope`, `category`, `by`, `custom` on `SavedView`, type-only; edited), `state/reportVisits.ts` (no-op API, called from one effect in `AppShell`), `core/shortcutRegistry.ts` (three entries, none live yet) |
| Hosts mounted in `AppShell` (edited), all `null` today | `ui/qc/QuickCreateHost.tsx` (Alt+Q), `ui/navGuard.tsx` (Leave this entry?), `ui/reportsCenter/ReportBuilderBar.tsx`; `ui/qc/QuickCreateButton.tsx` mounted in `TopBar` (`ui/widgets.tsx`, edited, between the Go To pill and the theme toggles) |
| Generated from the registry | `ui/overlays.tsx` (the `?` sheet rows and `overlayUp()` now also sees `.qc-overlay` and `.nav-guard`; edited), `ui/favouritesCore.ts` `LOCKED_KEYS` (edited) |
| Wave B reservations | `Btn.id?` and `Btn.movable?` in `ui/widgets.tsx` (type-only) |
| CSS | `styles/pageHeader.css`, `styles/light/header.css`, `styles/light/filters.css`, `styles/light/reportscenter.css` (empty overlay blocks, imported by `styles/light.css`, edited) |
| Assertions | `scripts/assert/zoho-scaffold.assert.ts` |

---

## B. Build plan: three builders, disjoint files

| | **B1: navigation and create** | **B2: filters and views** | **B3: Reports Center** |
|---|---|---|---|
| Delivers | Menu V2 (section 1), the quick-create tables (2), `PageActions` (3), the universal `+` (4), the nav-stack policy, guard, crumb (5) | Row filter engine, filter bar and panel, `Alt+F12` / `Ctrl+F12`, saved-view schema, views dropdown, company-shared views, the register's flat All level (6) | Report catalog, Reports Center screen, visits and favourites, the report builder and its bar (7), Gateway / Go To entries already registered |
| Owns (creates or edits) | `ui/navTree.ts`, `ui/nav/*`, `state/navStore.ts` and `core/navPrefs.ts` if needed, `ui/quickCreate.ts` (fill the tables), `ui/header/PageActions.tsx`, `ui/qc/*`, `ui/navGuard.tsx`, `ui/navigate.ts` (new), `core/navPlan.ts` (new), `state/navGuard.ts` (new), `ui/favouritesCore.ts` (**only** `planNav`, `decideChord`, `mayHoldWork`; not `STATIC`, not `LOCKED_KEYS`), `ui/favourites.tsx`, `ui/modern/CommandPalette.tsx` (**only** `go()`), `ui/widgets.tsx` (TopBar button wiring, StatusBar crumb, `AcceptPrompt` labels), `ui/AppShell.tsx` (**only** the crumb props), `ui/overlays.tsx` (**only** `NAV_KEYS`), the dirty probes in `VoucherScreen.tsx`, `forms.tsx`, `payroll/vouchers.tsx`, `admin/BulkEntry.tsx`, `admin/ImportData.tsx`, `inventory/manufacturing.tsx`, `admin/Budgets.tsx`, `hr/leaveapply.tsx` (one `useDirty` call each), `styles/header.css`, `styles/qc.css`, `styles/navtree.css`, `styles/light/header.css`, `styles/light/shell.css` (menu rules only), `scripts/assert/sidenav.assert.ts` (rewrite the literal table), `favourites.assert.ts`, new `navstack.assert.ts`, `quickcreate.assert.ts`, `scripts/shots/recipes/sidenav.mjs` plus a new recipe file | `core/reportFilter.ts` (the engine), `core/savedViews.ts`, `ui/reportViews/*`, `ui/reportConfig.tsx` (publish `ReportActions`, views), `ui/header/ViewsTitle.tsx`, `ui/header/FilterBar.tsx`, `ui/rowFilter.ts` (new hook), `ui/reports.tsx` (**only** the filter note on the `rpt-sub` line, and Day Book / Ledger Vouchers adoption), `ui/reports/financial.tsx` (`RegisterReport`), `ui/reports/outstanding.tsx`, `types.ts` (`Company.reportViews?`), `styles/rowfilter.css`, `styles/views.css`, `styles/light/filters.css`, `scripts/assert/dashboard-custom.assert.ts` if its source scan needs it, new `rowfilter.assert.ts`, `savedviews` additions, recipes | `ui/reportCatalog.ts` (new), `state/reportVisits.ts` (the body), `ui/reportsCenter/*`, `ui/reportsCenter/builderSession.ts` (new), `styles/reportscenter.css`, `styles/light/reportscenter.css`, new `reportcatalog.assert.ts`, recipes |
| Must not touch | `reports.tsx` (B2's), `router.tsx`, `store.ts`, `access.ts`, `recents.ts` (nobody: a new screen is a question for the architect) | `widgets.tsx`, `AppShell.tsx`, `navTree.ts`, `overlays.tsx` | everything of B1 and B2 above; `CommandPalette.tsx`, `MenuScreen.tsx` (already registered) |
| Needs from the others | Nothing. | The `register` flat level (6.7) is B2's; B1's rows already point at it with `all: true` and work with the month matrix until it lands. | B1's `PageActions` draws "Create New Report" on its own once `QUICK_CREATES` is filled (the `reportsCenter` page entry is in Appendix B). B2's `SavedView` fields for the views list and the saved builder output. |

**Rules for all three.**
1. **Hot files.** After this scaffold the registrations (screens, routes, crumbs, rights, Go To, Gateway, key sheet) are DONE. If a builder believes it needs a new screen or key row, it asks; it does not edit `router.tsx`, `store.ts`'s union, `access.ts`, `recents.ts`, `MenuScreen.tsx`, `favouritesCore.ts` `STATIC`.
2. **New chords** are appended to `core/shortcutRegistry.ts` (section 8.2), never to `overlays.tsx` or `LOCKED_KEYS`; flip `live: true` and name the `handler` when the handler lands.
3. **No git.** Nobody runs `git add`, `commit`, `stash`, `checkout` or `reset`; the orchestrator commits.
4. **Gate per builder:** `npx tsc --noEmit` 0, `npm run build`, `npm run assert` green (including the suites above), the builder's recipes in light and dark, compact and comfortable (`npm run shots`, never the shared Browser pane), and the screenshots of the untouched screens unchanged.
5. **Tokens only.** Every new rule uses tokens (`--accent`, `--on-accent`, `--sel`, `--panel`, `--border-lt`, `--sp-*`, `--fs-*`, `--radius*`, `--shadow-*`); a light-only look goes in the matching `styles/light/*.css` (one `@media screen` block, every selector starting `html:root[data-scheme='light']`, report-clone-safe rules rooted under `.screen`); no hex in `:root` for a token dark does not override; no `display: none` on a grid item; no `text-transform: uppercase` or tracked labels in the menus.
6. **Cycles.** `ui/widgets.tsx` imports `ui/qc/QuickCreateButton.tsx`; that file (and everything under `ui/qc/`) must not import `ui/widgets` at module level beyond what is hoisted (import `refocus` from a leaf, or dispatch the `tw-refocus` event itself).
7. **Hidden screens.** `AppShell` keeps every stacked screen mounted (`display: none`). A slot component, a probe or a key handler of a hidden screen must be inert: use `useScreenKeys` (top-gated), never a global handler with a toggled `active`.

**Order.** All three start together. Cross-dependencies are only the two listed above. Integration order at the end: B1, B2, B3, then the full sweep (`npm run shots` all screens, `npm run perf`).

---

## 1. Menu V2: fewer groups, a list per row, a `+` after it

### 1.1 What the owner asked, and the shape that answers it [R, J]
"After every button there is a `+`: by default it opens the report, and if you press `+` it creates the voucher. It will decrease the number of submenus." Zoho's Sales group is *documents*, each opening its list with a `+` beside it. Phase 25's tree was *tasks and reports mixed*: "Sales Invoice" opened an entry form and "Sales Register" was a separate row, so nine rows meant six things. Merging each entry row with its register row gives one row per document, which is what takes Sales from nine rows to six without removing a screen. The same merge takes the whole tree from 14 groups and 108 rows to **12 groups and 64 destinations**.

**Rules**
1. **At most 12 top-level groups, at most 6 rows in each** (Dashboard is a one-row leaf and counts as a group). Row labels stay at 26 characters or fewer (the measured room in the 212px rail); group labels at 20.
2. **A row means one thing: it opens the list, register or report.** If the thing has an entry screen, the row has a `+` that opens it. If it has none (Receivables, Reconciliation, Stock Summary) there is no `+`. A row that is only an entry screen (Manufacturing) keeps that screen as its row and has no `+`.
3. **A group label opens the module's list** (`NavGroup.to`); **the chevron alone expands**. When the group's `to` is hidden for this company or operator (a feature is off, a right is missing) the click opens the first visible row's `to`; a group with no visible row is not drawn. The group is lit (`has-on`) whenever the screen on top is its `to`, so Reports lights its header on the Reports Center, which has no row of its own.
4. **Everything removed from the rail stays reachable**: through the Reports Center (reports), the group's footer into a Gateway submenu, the `+` and the universal `+` (entries), and Go To. Section 1.4 lists all 53.
5. **Priority order** (by how often an owner opens it): Dashboard, Sales, Purchases, Banking, Accounting, Inventory, Reports, Taxes & Compliance, Payroll & HR, CRM, Data & Utilities, Settings.

### 1.2 The tree (final; the TypeScript is Appendix A, compiled against the real `Screen` union)

Routes are exactly the screens the Gateway or an F-key pushes, with two deliberate additions: a register row opens the **flat All level** (`all: true`, section 6.7) and carries the month matrix in `also`, so the Gateway's own `{ s: 'register', base }` still lights the row; and the registers of voucher types the Gateway never listed (Receipt, Payment, Contra, orders, notes, stock) use the same `register` screen with their base (it accepts any `VchBase`; `core/savedViews.ts` already names all of them).

**Master rows keep their old meaning.** "Ledgers", "Groups", "Cost Centres", "Items", "Godowns", "Employees", "Voucher Types" open the *picker that leads to the alter form* (`pickMaster`, goal alter), exactly as the Phase 25 master rows did, because that is what the Gateway's Alter menu does and a Tally switcher's hands know it. A ledger's *book* (its transactions) is the Reports Center's "Ledger" entry and Go To by name. (A "Book / Alter" choice inside the picker is a good next step and out of scope.)

**Dashboard** (`dashboard`). A click on the label opens `{ s: 'dashboard' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Dashboard | `{ s: 'dashboard' }` |  | `W` (gateway) |  |

**Sales** (`sales`). A click on the label opens `{ s: 'register', base: 'Sales', all: true }`. Footer: "Outstandings & ageing…" opens `{ s: 'menu', id: 'statementsAcc' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Invoices | `{ s: 'register', base: 'Sales', all: true }` | Invoice `{ s: 'voucher', base: 'Sales' }` (F8) |  |  |
| Payments Received | `{ s: 'register', base: 'Receipt', all: true }` | Payment Received `{ s: 'voucher', base: 'Receipt' }` (F6) |  |  |
| Receivables | `{ s: 'outstandings', side: 'receivables' }` |  |  | billwise |
| Credit Notes | `{ s: 'register', base: 'CreditNote', all: true }` | Credit Note `{ s: 'voucher', base: 'CreditNote' }` (Alt+F6) |  |  |
| Sales Orders | `{ s: 'register', base: 'SalesOrder', all: true }` | Sales Order `{ s: 'voucher', base: 'SalesOrder' }` |  | orders |
| Delivery Notes | `{ s: 'register', base: 'DeliveryNote', all: true }` | Delivery Note `{ s: 'voucher', base: 'DeliveryNote' }` |  | inventory |

**Purchases** (`purchases`). A click on the label opens `{ s: 'register', base: 'Purchase', all: true }`. Footer: "Outstandings & ageing…" opens `{ s: 'menu', id: 'statementsAcc' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Bills | `{ s: 'register', base: 'Purchase', all: true }` | Bill `{ s: 'voucher', base: 'Purchase' }` (F9) |  |  |
| Payments Made | `{ s: 'register', base: 'Payment', all: true }` | Payment Made `{ s: 'voucher', base: 'Payment' }` (F5) |  |  |
| Payables | `{ s: 'outstandings', side: 'payables' }` |  |  | billwise |
| Debit Notes | `{ s: 'register', base: 'DebitNote', all: true }` | Debit Note `{ s: 'voucher', base: 'DebitNote' }` (Alt+F5) |  |  |
| Purchase Orders | `{ s: 'register', base: 'PurchaseOrder', all: true }` | Purchase Order `{ s: 'voucher', base: 'PurchaseOrder' }` |  | orders |
| Receipt Notes | `{ s: 'register', base: 'ReceiptNote', all: true }` | Receipt Note `{ s: 'voucher', base: 'ReceiptNote' }` |  | inventory |

**Banking** (`banking`). A click on the label opens `{ s: 'cashBankBook' }`. Footer: "More banking…" opens `{ s: 'menu', id: 'banking' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Cash/Bank Book | `{ s: 'cashBankBook' }` |  |  |  |
| Transfers | `{ s: 'register', base: 'Contra', all: true }` | Transfer (Contra) `{ s: 'voucher', base: 'Contra' }` (F4) |  |  |
| Reconciliation | `{ s: 'brs' }` |  |  |  |
| Import Statement | `{ s: 'brsImport' }` |  |  |  |
| Cheques | `{ s: 'cheque' }` |  |  |  |
| Post-Dated Cheques | `{ s: 'pdcRegister' }` |  |  |  |

**Accounting** (`accounting`). A click on the label opens `{ s: 'daybook' }`. Footer: "Books & registers…" opens `{ s: 'menu', id: 'accountBooks' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Day Book | `{ s: 'daybook' }` |  | `K` (gateway) |  |
| Journals | `{ s: 'register', base: 'Journal', all: true }` | Journal Entry `{ s: 'voucher', base: 'Journal' }` (F7) |  |  |
| Chart of Accounts | `{ s: 'coa' }` | Ledger `{ s: 'ledger' }` | `H` (gateway) |  |
| Ledgers | `{ s: 'pickMaster', kind: 'ledger' }` | Ledger `{ s: 'ledger' }` |  |  |
| Groups | `{ s: 'pickMaster', kind: 'group' }` | Group `{ s: 'group' }` |  |  |
| Cost Centres | `{ s: 'pickMaster', kind: 'costCentre' }` | Cost Centre `{ s: 'costCentre' }` |  | costCentres |

**Inventory** (`inventory`). A click on the label opens `{ s: 'stockSummary' }`. Footer: "Inventory reports…" opens `{ s: 'menu', id: 'inventoryBooks' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Stock Summary | `{ s: 'stockSummary' }` |  | `S` (gateway) | inventory |
| Items | `{ s: 'pickMaster', kind: 'stockItem' }` | Item `{ s: 'stockItem' }` |  | inventory |
| Stock Journal | `{ s: 'register', base: 'StockJournal', all: true }` | Stock Journal `{ s: 'voucher', base: 'StockJournal' }` (Alt+F7) |  | inventory |
| Physical Stock | `{ s: 'register', base: 'PhysicalStock', all: true }` | Physical Stock `{ s: 'voucher', base: 'PhysicalStock' }` (Ctrl+F7) |  | inventory |
| Godowns | `{ s: 'pickMaster', kind: 'godown' }` | Godown `{ s: 'godown' }` |  | inventory |
| Manufacturing | `{ s: 'manufacture' }` |  |  | inventory |

**Reports** (`reports`). A click on the label opens `{ s: 'reportsCenter' }`. Footer: "Gateway report menus…" opens `{ s: 'menu', id: 'display' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Balance Sheet | `{ s: 'balanceSheet' }` |  | `B` (gateway) |  |
| Profit & Loss | `{ s: 'profitLoss' }` |  | `P` (gateway) |  |
| Trial Balance | `{ s: 'trialBalance' }` |  |  |  |
| Cash Flow | `{ s: 'cashFlow' }` |  |  |  |
| Ratio Analysis | `{ s: 'ratioAnalysis' }` |  | `R` (gateway) |  |
| Top-N Analysis (MIS) | `{ s: 'topMis' }` |  |  |  |

**Taxes & Compliance** (`taxes`). A click on the label opens `{ s: 'gstr3b' }`. Footer: "All statutory reports…" opens `{ s: 'menu', id: 'statutory' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| GSTR-3B | `{ s: 'gstr3b' }` |  |  | gst |
| GSTR-1 | `{ s: 'gstr1' }` |  |  | gst |
| GSTR-2B Reconciliation | `{ s: 'gstr2b' }` |  |  | gst |
| e-Invoice / e-Way Bill | `{ s: 'einvoice' }` |  |  | gst |
| TDS Outstanding | `{ s: 'tdsReport' }` |  |  | tds |
| TDS Challans & Returns | `{ s: 'tdsReturns' }` |  |  | tds |

**Payroll & HR** (`payroll`). A click on the label opens `{ s: 'pickMaster', kind: 'employee' }`. Footer: "Payroll reports…" opens `{ s: 'menu', id: 'payrollRep' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Employees | `{ s: 'pickMaster', kind: 'employee' }` | Employee `{ s: 'employee' }` |  | payroll |
| Attendance | `{ s: 'attendanceRegister' }` | Attendance `{ s: 'attendance' }` |  | payroll |
| Payroll | `{ s: 'paySheet' }` | Payroll Voucher `{ s: 'payrollVch' }` |  | payroll |
| Salary Disbursement | `{ s: 'salaryDisbursement' }` |  |  | payroll |
| Leave | `{ s: 'leaveRegister' }` | Leave `{ s: 'leaveApply' }` |  | payroll |
| PF / ESI / PT | `{ s: 'payrollStatutory' }` |  |  | payroll |

**CRM** (`crm`). A click on the label opens `{ s: 'leads' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Leads | `{ s: 'leads' }` | Lead `{ s: 'lead' }` |  |  |
| Follow-ups Due | `{ s: 'followUps' }` |  |  |  |
| Pipeline Summary | `{ s: 'pipeline' }` |  |  |  |

**Data & Utilities** (`data`). A click on the label opens `{ s: 'backupRestore' }`. Footer: "More utilities…" opens `{ s: 'menu', id: 'companyInfo' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Backup / Restore | `{ s: 'backupRestore' }` |  |  |  |
| Bulk Entry | `{ s: 'bulkEntry' }` |  |  |  |
| Import Data | `{ s: 'importData' }` |  |  |  |
| Export Data | `{ s: 'exportXml' }` |  |  |  |
| Data Check | `{ s: 'dataCheck' }` |  |  |  |
| API Keys | `{ s: 'apiKeys' }` |  |  |  |

**Settings** (`settings`). A click on the label opens `{ s: 'companyAlter' }`. Footer: "More settings…" opens `{ s: 'menu', id: 'companyInfo' }`.

| Row | Opens | `+` creates | Key on the row | Needs |
|---|---|---|---|---|
| Company Details | `{ s: 'companyAlter' }` |  |  |  |
| Switch Company | `{ s: 'companySelect' }` |  | `F3` (anywhere) |  |
| Features | `{ s: 'features' }` |  | `F11` (anywhere) |  |
| Users & Security | `{ s: 'security' }` |  |  |  |
| Print Configuration | `{ s: 'printConfig' }` |  |  |  |
| Voucher Types | `{ s: 'pickMaster', kind: 'voucherType' }` | Voucher Type `{ s: 'voucherType' }` |  |  |


**Totals.** 12 groups. 64 destinations: 1 + 6 + 6 + 6 + 6 + 6 + 6 + 6 + 6 + 3 + 6 + 6. 26 rows with a `+` (the Chart of Accounts and Ledgers both open the same New Ledger). 9 rows with a keycap: Dashboard `W`, Day Book `K`, Chart of Accounts `H`, Stock Summary `S`, Balance Sheet `B`, Profit & Loss `P`, Ratio Analysis `R` (Gateway letters, fire on the Gateway only), Switch Company `F3` and Features `F11` (fire everywhere). 10 voucher creates with an F-key: F4 to F9, Alt+F5, Alt+F6, Alt+F7, Ctrl+F7.

### 1.3 Behaviour (the deltas to `left-nav-spec.md` sections 7 and 10)

**Mouse**
| Target | Does |
|---|---|
| Group **label** (icon and text) | `requestNav(group.to, 'dest')` (section 5), and the group opens (the existing auto-open for the screen on top; no new state). |
| Group **chevron** | Toggles that group only (`toggleGroup`); never navigates; `stopPropagation`. |
| A **row** | `requestNav(row.to, 'dest')`. |
| A row's **`+`** | `requestNav(create.to, 'dest')`. |
| The **star** | Unchanged: pins the row's `to` (the same pin list the Reports Center's star uses). |

**Keyboard** (only while focus is in the tree; the rail still registers no global handler and still stops every bare key it uses)
| Key on | Was | Now |
|---|---|---|
| a group header: `Enter` | toggled the group | **opens the module's list** (as the label click) |
| a group header: `→` / `←` | open / close | unchanged |
| a group header: `Space` | nothing | **toggles** the group (so a keyboard user can still expand without navigating) |
| a row: `Enter` / `+` / `Space` / arrows / `Home` `End` / `PgUp` `PgDn` / printable (goes to the filter) / `Esc` | | unchanged. `+` opens the create form on rows that have one (a numpad plus too). |

The tree's hidden hint (`NAV_HINT`) and the `?` sheet's "In the left menu" rows (`NAV_KEYS` in `ui/overlays.tsx`, **B1's only edit there**) change to: "Enter opens the row or the module. Right and Left expand and collapse a group. Space on a group expands it; on a row it pins it. Plus on a row starts a new one."

**The `+` button.** Visible on row hover, on `:focus-within`, **always on the current row** (`.on`, the research's "Invoices (+)" on the lit row), and always on touch (`@media (hover: none)`). Same 24 x 24 target, same absolutely-positioned strip as today's master `+` (no layout shift), tokens only. Name: `aria-label="New: Invoices"`; title: `New Invoice (F8 on the Gateway and its menus)`, built with the shared where-it-fires wording (`menuModel.rowTitle`'s `WHERE` table), and `aria-keyshortcuts` carries the F-key chord exactly as today (`ariaShortcut`). **The create key is not drawn as a keycap on the row**: the voucher F-keys fire on the Gateway and its menus and not on a register, so a permanent `F8` beside "Invoices" would over-promise and would crowd the label at hover. A row's own keycap (the nine above) is unchanged.

**Row tooltips.** Label alone, or with a key: `Day Book: press K on the Gateway`. A row that opens the flat register says nothing extra.

**Active row.** `activeRow` order is: exact `to` key; then `within(to)` or an `also` form; then **the group's own `to`** (new: the group header is lit, `rowId = '<group>.home'`); then the footer submenu. With `also` carrying the month matrix and the edit forms (`ledger`, `group`, `stockItem`...), arriving by the Gateway, an alteration or a drill keeps the right row lit.

**Footers, Pinned, Saved views, the filter box, Quick access, width (212px), full-screen toggle, density, scheme, forced colours, focus handling:** unchanged from `left-nav-spec.md` (sections 7 to 10, Appendices C and D). Saved views keep their group between Reports and Taxes & Compliance (the old "between Reports and Masters"; Masters no longer exists); the group shows the operator's **pinned views first**, then the most recently saved, six at most.

**Types** (`ui/navTree.ts`, shown in Appendix A): `NavItem.create` becomes a **QuickCreate id** (a string), resolved through `QUICK_CREATES`; `NavItem.key` / `keyWhere` now describe only the row's own `to`; `createKey` does not exist (the registry holds it); `NavGroup` gains `to: Screen`. `visibleTree` resolves `create` ids, drops a `+` whose create is denied or whose feature is off (the row stays), and applies the group-`to` fallback of rule 3. `menuModel.RowModel.create` stays a `Screen` (`qc.to`) and gains `createTitle`.

**Defaults:** unchanged (Menu view, Pinned open, every module folded; the group of the screen on top auto-opens for the session and is never stored).

### 1.4 What left the rail, and where it lives now

All 108 old rows are reachable. 55 rows stayed (several renamed: Sales Register to **Invoices**, and so on). The other 53:

| Old rail row | Destination | Now reached by |
|---|---|---|
| Sales Invoice | `voucher:base=Sales` | the `+` on **Invoices** (and the universal `+`) |
| Payment Received | `voucher:base=Receipt` | the `+` on **Payments Received** (and the universal `+`) |
| Sales Order | `voucher:base=SalesOrder` | the `+` on **Sales Orders** (and the universal `+`) |
| Credit Note | `voucher:base=CreditNote` | the `+` on **Credit Notes** (and the universal `+`) |
| Delivery Note | `voucher:base=DeliveryNote` | the `+` on **Delivery Notes** (and the universal `+`) |
| Ageing Analysis | `ageing` | Reports Center (category in Appendix C), Go To |
| Reminder Letters | `reminders` | Reports Center (category in Appendix C), Go To |
| Purchase Invoice | `voucher:base=Purchase` | the `+` on **Bills** (and the universal `+`) |
| Payment Made | `voucher:base=Payment` | the `+` on **Payments Made** (and the universal `+`) |
| Purchase Order | `voucher:base=PurchaseOrder` | the `+` on **Purchase Orders** (and the universal `+`) |
| Debit Note | `voucher:base=DebitNote` | the `+` on **Debit Notes** (and the universal `+`) |
| Receipt Note | `voucher:base=ReceiptNote` | the `+` on **Receipt Notes** (and the universal `+`) |
| Transfer (Contra) | `voucher:base=Contra` | the `+` on **Transfers** (and the universal `+`) |
| Cheque Books | `chequeBooks` | Reports Center (category in Appendix C), Go To |
| Payment Advice | `paymentAdvice` | Reports Center (category in Appendix C), Go To |
| Deposit Slip | `depositSlip` | Reports Center (category in Appendix C), Go To |
| Journal | `voucher:base=Journal` | the `+` on **Journals** (and the universal `+`) |
| Ledger | `pickMaster:goal=book,kind=ledger` | Reports Center (category in Appendix C), Go To |
| Group Summary | `pickMaster:goal=book,kind=group` | Reports Center (category in Appendix C), Go To |
| Cost Centre Analysis | `costAnalysis` | Reports Center (category in Appendix C), Go To |
| Budgets | `budgets` | Gateway submenu `create` (reached from the group footer or letter keys), Go To |
| Interest Calculation | `interest` | Reports Center (category in Appendix C), Go To |
| Reorder Status | `reorderStatus` | Reports Center (category in Appendix C), Go To |
| Stock Journal | `voucher:base=StockJournal` | the `+` on **Stock Journal** (and the universal `+`) |
| Physical Stock | `voucher:base=PhysicalStock` | the `+` on **Physical Stock** (and the universal `+`) |
| Godown Summary | `godownSummary` | Reports Center (category in Appendix C), Go To |
| Stock Ageing | `stockAgeing` | Reports Center (category in Appendix C), Go To |
| HSN Summary | `hsnSummary` | Reports Center (category in Appendix C), Go To |
| GST Validation | `gstValidation` | Reports Center (category in Appendix C), Go To |
| GST Reconciliation | `gstRecon` | Reports Center (category in Appendix C), Go To |
| Reverse Charge Register | `rcmRegister` | Reports Center (category in Appendix C), Go To |
| GSTR-9 Annual Return | `gstr9` | Reports Center (category in Appendix C), Go To |
| TDS Quarterly Tracker | `tdsQuarterly` | Reports Center (category in Appendix C), Go To |
| TDS Deductee-wise | `tdsDeductee` | Reports Center (category in Appendix C), Go To |
| TDS Challan Recon | `tdsChallanRecon` | Reports Center (category in Appendix C), Go To |
| Form 16A Certificates | `form16a` | Reports Center (category in Appendix C), Go To |
| TDS Interest & Late Fee | `tdsInterest` | Reports Center (category in Appendix C), Go To |
| TCS Outstanding | `tcsReport` | Reports Center (category in Appendix C), Go To |
| TCS 206C(1H) Turnover | `tcs206c` | Reports Center (category in Appendix C), Go To |
| Funds Flow | `fundsFlow` | Reports Center (category in Appendix C), Go To |
| Profitability Analysis | `profitability` | Reports Center (category in Appendix C), Go To |
| Budget vs Actual | `budgetVsActual` | Reports Center (category in Appendix C), Go To |
| Units | `pickMaster:kind=unit` | Gateway submenu `alter` (reached from the group footer or letter keys), Go To |
| Price Levels | `priceLevels` | Gateway submenu `create` (reached from the group footer or letter keys), Go To |
| Payslip | `payslip` | Reports Center (category in Appendix C), Go To |
| Leave Balance | `leaveBalance` | Reports Center (category in Appendix C), Go To |
| Custom Fields | `customFields` | Gateway submenu `companyInfo` (reached from the group footer or letter keys), Go To |
| Webhooks | `webhooks` | Gateway submenu `companyInfo` (reached from the group footer or letter keys), Go To |
| Compare with Tally Data | `tallyCompare` | Gateway submenu `companyInfo` (reached from the group footer or letter keys), Go To |
| GST Rate Setup | `gstRateSetup` | Gateway submenu `statutory` (reached from the group footer or letter keys), Go To |
| Sync Settings | `syncLogin` | Gateway submenu `companyInfo` (reached from the group footer or letter keys), Go To |
| Subscription & Billing | `billing` | Gateway submenu `companyInfo` (reached from the group footer or letter keys), Go To |
| Edit Log | `auditLog` | Reports Center (category in Appendix C), Go To |

(`Switch Company` stays; `Batch & Expiry` was never on the rail and stays off because it is a stub.)

### 1.5 Assertions for this section (builder B1; `scripts/assert/sidenav.assert.ts` is rewritten, deliberately)
- Shape: 12 groups in the published order; each non-leaf group has 3 to 6 rows (CRM 3, the rest 6); 64 rows; 26 `create` ids; ids unique; labels at most 26 characters; aliases lower case; the only destination listed twice is none (Stock Items is now Inventory only).
- Every `to`, `also`, group `to`, footer `to` and every create's `to` is in the union, the router, `SCREEN_AREA`; footer menus exist in `MENUS`.
- Group `to` equals the first row's `to` for every group except Reports; Reports' `to` is `reportsCenter`.
- Every `create` id exists in `QUICK_CREATES`; for a row whose page has a `QC_PAGES` entry, **the page's primary equals the row's create**.
- Keys: a row `key` with `keyWhere: 'gateway'` is that screen's Gateway letter; `'anywhere'` is handled in `App.tsx`; a voucher create's key is its F-key (the registry assertion).
- The Gateway comparison of section 5 of the old suite is kept, comparing a register row on its **base** screen (the Gateway has no flat level).
- Gating: the old section 6 tests, rewritten for the new rows (feature off hides; a restricted role loses rows and `+`s; footers only for full-rights or security off; the group-`to` fallback when its first row is hidden).
- Active row: exact, `also`, group-lit (`reports.home`), footer.
- No row opens a `PlannedScreen` (already asserted by `zoho-scaffold`).
- The literal table `SPEC_ROWS` is regenerated from Appendix A (columns: group id, label, destination key, create id).
- Recipes (`scripts/shots/recipes/sidenav.mjs`): update `allOpen` and `openGroup` for the new ids; new: `nav-plus` (hover a row, `+` visible, click opens the entry, Esc returns), `nav-label-opens` (click "Sales" opens the register and the group), `nav-chevron-only` (chevron toggles, screen unchanged).

---

## 2. The quick-create registry: one list behind every `+`

`src/ui/quickCreate.ts` (scaffolded: types, layout, lookups and gates are real; the two tables are empty until B1 pastes Appendix B).

```ts
interface QuickCreate {
  id: string;            // 'sales.invoice'
  label: string;         // everyday words first: "Invoice", "Payment Received", "Bill"
  group: QcGroup;        // 'General' | 'Sales' | 'Purchases' | 'Banking' | 'Accounting' | 'Inventory' | 'Payroll' | 'CRM'
  to: Screen;            // EXACTLY what the Gateway / the F-key pushes for this creation
  key?: string;          // the Tally key, a raw combo ('F8')
  keyWhere?: KeyWhere;   // where it fires: a voucher F-key fires on the Gateway and its menus ONLY
  feature?: keyof CompanyFeatures;
  aliases: readonly string[];   // invisible search words: Tally names, Zoho names, trade words
}
```

### 2.1 What it holds

| id | Label | Column group | Opens | Tally key (where it fires) | Needs |
|---|---|---|---|---|---|
| `sales.invoice` | Invoice | Sales | `{ s: 'voucher', base: 'Sales' }` | `F8` (menus) |  |
| `sales.received` | Payment Received | Sales | `{ s: 'voucher', base: 'Receipt' }` | `F6` (menus) |  |
| `sales.creditnote` | Credit Note | Sales | `{ s: 'voucher', base: 'CreditNote' }` | `Alt+F6` (menus) |  |
| `sales.order` | Sales Order | Sales | `{ s: 'voucher', base: 'SalesOrder' }` |  | orders |
| `sales.delivery` | Delivery Note | Sales | `{ s: 'voucher', base: 'DeliveryNote' }` |  | inventory |
| `purchases.bill` | Bill | Purchases | `{ s: 'voucher', base: 'Purchase' }` | `F9` (menus) |  |
| `purchases.paid` | Payment Made | Purchases | `{ s: 'voucher', base: 'Payment' }` | `F5` (menus) |  |
| `purchases.debitnote` | Debit Note | Purchases | `{ s: 'voucher', base: 'DebitNote' }` | `Alt+F5` (menus) |  |
| `purchases.order` | Purchase Order | Purchases | `{ s: 'voucher', base: 'PurchaseOrder' }` |  | orders |
| `purchases.receiptnote` | Receipt Note | Purchases | `{ s: 'voucher', base: 'ReceiptNote' }` |  | inventory |
| `banking.transfer` | Transfer (Contra) | Banking | `{ s: 'voucher', base: 'Contra' }` | `F4` (menus) |  |
| `accounting.journal` | Journal Entry | Accounting | `{ s: 'voucher', base: 'Journal' }` | `F7` (menus) |  |
| `accounting.group` | Group | Accounting | `{ s: 'group' }` |  |  |
| `accounting.costcentre` | Cost Centre | Accounting | `{ s: 'costCentre' }` |  | costCentres |
| `inventory.item` | Item | Inventory | `{ s: 'stockItem' }` |  | inventory |
| `inventory.stockjournal` | Stock Journal | Inventory | `{ s: 'voucher', base: 'StockJournal' }` | `Alt+F7` (menus) | inventory |
| `inventory.physical` | Physical Stock | Inventory | `{ s: 'voucher', base: 'PhysicalStock' }` | `Ctrl+F7` (menus) | inventory |
| `inventory.mfg` | Manufacturing Journal | Inventory | `{ s: 'manufacture' }` |  | inventory |
| `inventory.godown` | Godown | Inventory | `{ s: 'godown' }` |  | inventory |
| `inventory.unit` | Unit | Inventory | `{ s: 'unit' }` |  | inventory |
| `general.ledger` | Ledger | General | `{ s: 'ledger' }` |  |  |
| `settings.vtype` | Voucher Type | General | `{ s: 'voucherType' }` |  |  |
| `general.report` | Custom Report | General | `{ s: 'customReport' }` |  |  |
| `payroll.employee` | Employee | Payroll | `{ s: 'employee' }` |  | payroll |
| `payroll.attendance` | Attendance | Payroll | `{ s: 'attendance' }` |  | payroll |
| `payroll.voucher` | Payroll Voucher | Payroll | `{ s: 'payrollVch' }` |  | payroll |
| `payroll.leave` | Leave | Payroll | `{ s: 'leaveApply' }` |  | payroll |
| `crm.lead` | Lead | CRM | `{ s: 'lead' }` |  |  |

`General` holds the master a customer, a vendor or an account is (a **Ledger**; Zoho's "customer" and "vendor" are a ledger with a party group, which is what the Tally switcher expects) and the report builder. `Banking` has one entry because Zoho's bank-account creates are ledgers; an **action** (Import Statement) is not a create and is not listed.

### 2.2 Gating (identical to the left menu's, and to what the store would refuse)
`visibleCreates(feats, ctx)` keeps an entry when its `feature` is on **and** `screenDenial(ctx, q.to) === null`. For a voucher `to`, `screenDenial` is `voucherDenial(... 'create')` (the voucher type and the role's allowed actions and date window), the very function `store.push` applies, so an entry that would be refused is never offered. Security off: everything for the open features. A `+` whose create is refused disappears; its row stays.

### 2.3 How a page chooses its "related creates"
Not inferred. **`QC_PAGES` is an explicit table** keyed by `pageKey(screen)` (the screen key without the parameters that only say *how* it is shown: `all`, an edit id, a drill-down group; so `register:base=Sales` is one page for the month matrix and the flat level). Each entry is a **primary** (the `+ New` button) and up to `MAX_RELATED` (5) **related** ids, in the order the dropdown lists them. `pageCreates()` returns `null`, and the page has **no `+ New` at all**, when the page has no entry, or when its primary is not offered to this operator (never a button that opens a refusal); a related entry the operator or company cannot use is dropped. Rules the table follows:
1. A page that lists a *document* (a register, Receivables) has that document's create as primary and the neighbouring documents of its flow as related (Sales Register: Invoice, then Payment Received, Credit Note, Sales Order, Delivery Note).
2. A page that lists a *master* has the master's create as primary and the master it is most often made with as related.
3. A report that is a statement (Balance Sheet, P&L, a return, an analysis) has **no** `+ New`: the universal `+` covers it.
4. **A row's `+` and its page's primary are the same entry**: the nav assertion fails on a mismatch.
5. The Reports Center's own entry is "Create New Report".

| Page (`pageKey`) | Primary ("+ New") | Related (dropdown, in order) |
|---|---|---|
| `register:base=Sales` | Invoice | Payment Received, Credit Note, Sales Order, Delivery Note |
| `register:base=Receipt` | Payment Received | Invoice, Credit Note |
| `register:base=CreditNote` | Credit Note | Invoice, Payment Received |
| `register:base=SalesOrder` | Sales Order | Invoice, Delivery Note |
| `register:base=DeliveryNote` | Delivery Note | Invoice, Sales Order |
| `outstandings:side=receivables` | Invoice | Payment Received, Credit Note |
| `register:base=Purchase` | Bill | Payment Made, Debit Note, Purchase Order, Receipt Note |
| `register:base=Payment` | Payment Made | Bill, Debit Note |
| `register:base=DebitNote` | Debit Note | Bill, Payment Made |
| `register:base=PurchaseOrder` | Purchase Order | Bill, Receipt Note |
| `register:base=ReceiptNote` | Receipt Note | Bill, Purchase Order |
| `outstandings:side=payables` | Bill | Payment Made, Debit Note |
| `register:base=Contra` | Transfer (Contra) | Payment Received, Payment Made, Journal Entry |
| `cashBankBook` | Transfer (Contra) | Payment Received, Payment Made |
| `register:base=Journal` | Journal Entry | Transfer (Contra) |
| `daybook` | Invoice | Bill, Payment Received, Payment Made, Journal Entry, Transfer (Contra) |
| `pickMaster:kind=ledger` | Ledger | Group |
| `coa` | Ledger | Group |
| `pickMaster:kind=group` | Group | Ledger |
| `pickMaster:kind=costCentre` | Cost Centre | (none) |
| `pickMaster:kind=voucherType` | Voucher Type | (none) |
| `stockSummary` | Item | Stock Journal, Physical Stock, Godown |
| `pickMaster:kind=stockItem` | Item | Godown, Unit |
| `pickMaster:kind=godown` | Godown | Item |
| `pickMaster:kind=unit` | Unit | Item |
| `register:base=StockJournal` | Stock Journal | Physical Stock, Item |
| `register:base=PhysicalStock` | Physical Stock | Stock Journal |
| `pickMaster:kind=employee` | Employee | Attendance, Payroll Voucher |
| `attendanceRegister` | Attendance | Employee |
| `paySheet` | Payroll Voucher | Attendance |
| `leaveRegister` | Leave | Employee |
| `leads` | Lead | (none) |
| `reportsCenter` | Custom Report | (none) |

### 2.4 Lookups the three surfaces share (already written)
`visibleCreates`, `pageKey`, `pageCreates`, `menuColumns` (the mega menu's columns: `QC_COLUMNS` places `Sales`, `Purchases`, `Banking + Accounting`, `Inventory`, `General + Payroll + CRM`; a group with nothing offered, and a column with nothing in it, are not drawn).

### 2.5 Assertions (builder B1; new `scripts/assert/quickcreate.assert.ts`; the generic invariants already run in `zoho-scaffold`)
Appendix B equals the registry (a literal table); every create `to` is what the Gateway Create menu pushes for the masters (and the F-key voucher for vouchers); a voucher's `key` is its own F-key and its `keyWhere` is `menus`; every nav row's `create` exists; each `QC_PAGES` primary matches its nav row; a clerk (Accounting) is offered no master; a reports-only operator none; `inventory` off drops the Inventory column; `orders` off drops the two orders; `pageCreates` is null for Balance Sheet.

---

## 3. The page header: `+ New ▾`, `...`, the views dropdown, the filter bar

### 3.1 One header, three slots (`ui/pageHeader.tsx`, scaffolded)
`.scr-head` is one flex row: title on the left, the company on the right. The shared shells now draw, in this order:

```
[title] [titleExtra]  [text-filter hint "Filter"]  [actions]  [company]
────────────────────────────────────────────────────────────────────
[belowHead]
[report body]
```

| Slot | Filled by | Draws | Where it is drawn |
|---|---|---|---|
| `titleExtra` | `ui/header/ViewsTitle.tsx` (B2) | the "All Sales Register ▾" views dropdown, only for a report that published `openViews` | `ReportShell` only |
| `actions` | `ui/header/PageActions.tsx` (B1) | `+ New ▾` and `...`, as below | `ReportShell`, `PickMaster`, `COAScreen` |
| `belowHead` | `ui/header/FilterBar.tsx` (B2) | the Filter button, the chips, the "Filtered" badge, the filtered count; the `Alt+F12` / `Ctrl+F12` handlers | `ReportShell` only |

**The rule that keeps every existing screen unchanged:** a slot component with nothing to draw returns `null` and adds no element. A slot with something draws **one** wrapper of its own with a class from `styles/header.css` or `styles/rowfilter.css` (`.ph-extra`, `.ph-actions`: inline-level, they sit in a flex row; `.ph-below`: block, `data-noprint="1"`). `HeaderCtx = { screen, slot, title, shell }` is all a slot is given; what the open report offers (its F12 key, the Configure opener, the saved view in force, its row filter) is read with `useReportActions(ctx.slot)` (`ui/reportActions.ts`), published by `useF12` (B2 wires it, so **every report that already has F12 gets Configure, views and the filter with no change of its own**).

Other shared list screens (Leads is a ReportShell; the rest of `scr-head` users are forms, statements and tools) get the header later, one line each; this wave covers the three shells above. The Dashboard keeps its own "+ New Sales Invoice" button and Quick Entry.

### 3.2 `+ New ▾` (builder B1)
- **Anatomy.** A split button at the right of the header: the main part reads **`+ New`** (icon plus the word), the chevron part opens the dropdown. Title and aria-label name the primary: "New Invoice". With no related creates there is no chevron part.
- **Main part:** `requestNav(primary.to, 'page')` (section 5: push, so Esc comes back to the list; a clean entry form of the same type already in the stack is popped back to, not duplicated).
- **Dropdown** (a popover, not a modal): the primary first (semibold), then the related creates; each row is the label plus the Tally key as a dim keycap, with a title that says where it fires ("press F9 on the Gateway and its menus"), never a bare key that does nothing from a list. Click or `Enter` runs it through `requestNav(.., 'page')` and closes it.
- **Keyboard.** The two parts are real `<button>`s in the Tab order; `Enter` and `Space` on them are stopped at the button (`ownActivate`, `ui/nav/ownKeys.ts`: the window key stack sees Enter before the click, and every list screen consumes Enter); `↓` or `Alt+↓` on the chevron opens the dropdown, `↑` `↓` move, `Enter` runs, `Esc` closes and returns focus to the button. While the dropdown is open it owns the keyboard through a `useKeys` handler **mounted only while open** (top of the stack, swallowing everything else); it never toggles a global handler's `active`.
- **No new F-key.** The ten voucher F-keys stay dead on a report exactly as today and exactly as the rail tells the user (`"on the Gateway and its menus"`). Binding "the list's own F-key" would change the truthful F-key rail and collide with screens that spend F7 (the bank screens). Revisit with the shortcut registry in Wave B.
- **Hidden** when `pageCreates()` is null; hidden screens (stack below the top) render nothing visible and register no keys.

### 3.3 The `...` menu (builder B1)
Same popover pattern; the trigger is a 24px icon button named "More actions". Items, in this order, each shown only when it applies and only when `screenDenial` allows it:
1. **Print** (`Ctrl+P`) and **Export CSV** (`Ctrl+E`) and **Export Excel**: only while a report table is on screen (`reportAvailable()`); they dispatch the existing `tw-print`, `tw-export`, `tw-exportxls` events that `App.tsx` already listens to, so Print and Export honour the filter by the mechanism in section 6.3 and nothing new is plumbed.
2. **Configure report** (`F12`): when `ReportActions.configure` exists.
3. **Save as view…** (`Alt+V`): when `openViews` exists.
4. **Filter…** (`Alt+F12`): when the report published a `filter`.
5. a separator, then, **only on a page whose primary create is a voucher**: **Import data…** (`importData`) and **Bulk entry…** (`bulkEntry`).
6. a separator, then **Refresh**: `useStore.getState().reloadActive()` (re-reads the company from IndexedDB; useful after a sync).
Keycaps shown here ARE live on the report (Print, Export, F12, Alt+V, Alt+F12), so they print without a "where" note.

### 3.4 The views dropdown and the filter bar
Specified in 6.5 and 6.4. They share this header and the same rules: `null` when empty, tokens only, light overlay in `styles/light/filters.css`.

### 3.5 Visual spec (B1 and B2)
Compact-compatible: the header row keeps its current height (the `+ New` button is 22px tall in compact, 26 in comfortable, text `--fs-sm`); the split button's main part is `--accent` fill with `--on-accent` text (5.8:1 light, 9.8:1 dark), the chevron part the same with a 1px `--on-accent` hairline at 30% between them; the `...` button is a ghost button (`--sunk` on hover); dropdown surfaces are `--panel` + `--border-lt` + `--shadow-2` + `--radius-lg`; focus is the existing 2px accent outline; forced-colours rules mirror `navtree.css`. In comfortable density `.scr-head .co` is already hidden, so the actions end at the right edge; in compact `.ph-actions { margin-left: auto }` keeps them beside the company name.

### 3.6 Assertions (B1)
`PageActions` source returns `null` or draws `ph-actions` (already in `zoho-scaffold`); with a filled registry: Sales Register shows `+ New` with the five related; Balance Sheet none; a clerk on the Ledgers picker none; the dropdown rows' keycaps carry a where-title; Print / Export / Configure items appear only when applicable; **recipes** `header-new-register` (click `+ New` on the Sales Register opens a Sales voucher, Esc returns to the register), `header-new-dropdown` (open, arrow, Enter), `header-more` (items per page), each light and dark.

---

## 4. The universal `+`

### 4.1 Where it sits [C, J]
The top bar is `[≡] brand · eight letter items · spacer · Go To pill · Light · Compact`. The universal `+` goes **immediately right of the Go To pill and left of the theme toggles** (`<QuickCreateButton />`, already mounted, drawing nothing). Zoho's order is search, then `+`; Wave B moves Light / Compact into Settings, which leaves the `+` as the right-most control, where Zoho puts it. A 24 x 24 target (WCAG 2.5.8) fits the 28px compact bar. It is a **solid `--accent` square with a `--on-accent` plus icon**, radius `--radius`, `aria-haspopup="dialog"`, `aria-expanded`, title `Quick create: start anything new (⌥Q)`.

### 4.2 The mega menu (`ui/qc/QuickCreateMenu.tsx`, B1)
- A fixed overlay (`.qc-overlay`: transparent scrim, click outside closes) anchored under the button; panel `.qc-pop`, `role="dialog" aria-label="Quick create"`, width `min(860px, 96vw)`, `--panel`, `--shadow-3`, `--radius-xl`.
- **Top:** a filter input, placeholder "Find something to create…" (matches label and aliases, every word must match, as `filterTree` does).
- **Body:** the columns of `menuColumns()`: **Sales | Purchases | Banking and Accounting | Inventory | General, Payroll and CRM**. Headings are plain 12px semibold dim text (no uppercase, no tracking). Each item is a button: the everyday label, and on the right the Tally key as a dim keycap with a where-it-fires title (the menu opens from anywhere, so the title says "press F8 on the Gateway and its menus" or, when the screen on top is a menu, "press F8 now"). On a phone-width viewport the columns collapse to one.
- **Empty state:** "Nothing to create here for your role."
- **Rights and features:** exactly section 2.2.

### 4.3 Keyboard model
| Key | Does |
|---|---|
| `Alt+Q` (`⌥Q`) or a click on the button | Opens; focus in the filter input. A second `Alt+Q` closes. |
| type | Filters across all columns; the first match is highlighted. |
| `↓` `↑` | Move within a column; `←` `→` jump to the nearest row of the neighbouring column (wrapping). |
| `Home` `End` | First / last visible item. |
| `Enter` | Creates the highlighted item: `requestNav(item.to, 'dest')` (replace when clean, ask when an entry is in the way, pop back to an existing blank one). |
| `Esc` | First press clears a non-empty filter, the next closes and gives focus back to the screen (`refocus()`). |
| anything else | Swallowed (modal). |

While it is open `overlayUp()` is true (`.qc-overlay`; registered), so the bare-letter bindings behind it (`?`, the G chord, a report's letters) are dead.

### 4.4 The chord: `Alt+Q`, with the audit
Audited against every quoted chord and every `altKey` / `ctrlKey` read in `src/` (grep), `keys.ts` (`BROWSER_RESERVED`, the G chord, hold-Alt), `favouritesCore.ts` (`LOCKED_KEYS`, the `Ctrl+Alt+1..9` slots), `App.tsx` (the generic F-key swallow), `docs/design/left-nav-spec.md` section 5 and `docs/research/attachments-drives.md`.

| Candidate | Verdict | Evidence |
|---|---|---|
| **`Alt+Q`** | **chosen** | Zero uses in `src/`. Not browser-reserved. Not a menu mnemonic in Chrome, Edge or Firefox (those are F, E, V, S, B, T, H). Outside the app's letter namespaces. On a Mac, Option+Q types `œ`; `combo()` recovers the physical `KeyQ`, so the chord is `Alt+Q` ([V] in `zoho-scaffold`). On a keyboard where AltGr+Q types `@` (German and others), Windows reports Ctrl+Alt, so `combo()` gives `Ctrl+Alt+Q`, which is not `Alt+Q` ([V]). Mnemonic: **Q**uick create. |
| `Alt+M`, `Alt+X`, `Alt+Z` | free, kept in reserve | the last free Alt letters after this spec |
| `Alt+U` | free but reserved | the attachments research (`docs/research/attachments-drives.md`) plans `Alt+U` for the Attachments panel |
| bare `+` | rejected | it is the left menu's own key (a row's `+`), and in a field it is amount arithmetic |
| `G` then `N` (the G-chord namespace) | rejected | the G chord is dead wherever a screen spends bare `G` (the Create menu: Group, Employee Group; Statutory Reports: GSTR-1; the e-invoice register), so it is not reachable everywhere |
| `Ctrl+N`, `Ctrl+Shift+N`, `Ctrl+T`, `Ctrl+W` | rejected | browser-reserved (`BROWSER_RESERVED`; Ctrl+Shift+N opens an incognito window) |
| `Alt+N` | taken | After Accept inside a voucher |
| `Ctrl+Alt+N` | rejected | `Ctrl+Alt` is the favourite slots' family and how Windows reports AltGr; `favourites.assert` forbids spelling it elsewhere |
| `Ctrl+.` | taken | `DISMISS_CHORD` (dismiss the message on screen) |
| `Ctrl+/`, `Ctrl+D`, `Ctrl+S`, `Ctrl+O`, `Ctrl+I`, `Ctrl+Y` | rejected | `?` is the sheet; the rest are browser commands (and `Ctrl+S` means Save to every Zoho user while this app accepts with `Ctrl+A`) |
| `C` then a letter (Zoho's) | rejected | collides with the Gateway's own letters; bare letters die inside fields |
| `Ctrl+Space` | taken | recall |
| every F-key, with or without Ctrl or Alt | taken | `App.tsx` swallows `^(Ctrl\|Alt\|)F\d{1,2}$` |

**Gate.** `QuickCreateHost` is ONE permanent `useFallbackKeys` handler (bottom of the stack: every screen is asked first, so it can never shadow a key a screen spends; it never toggles `active`; the pattern of `NavChordHost` and `FavouritesHost`). It answers "not mine" (`return false`) when: no company is open; a gate screen is on top (`isGateScreen`); `overlayUp()`; **`isTypingTarget(document.activeElement)`** (the brief's rule: never inside a text field; this also keeps Option+Q's `œ` typing normally on a Mac); `e.repeat` (swallowed once open, so a held key toggles once); or the chord carries Shift. **Consequence, stated plainly:** every voucher and master form keeps an input focused, so the chord is dead there. The keyboard routes from a form are Go To (`Ctrl+K` works inside fields: "New Sales", "New Ledger"... are its Create category) and the top-bar button with the mouse; from every report, menu and list the chord works.

**Registration.** `core/shortcutRegistry.ts` already holds the `quickCreate` entry (`Alt+Q`, `sheet: 'everywhere'`, `lockedGroup: 'Go anywhere'`, `live: false`). B1 flips `live: true` when `QuickCreateHost` binds it; the `?` sheet row and the Favourite Shortcuts locked row then appear by themselves, and `zoho-scaffold` checks that the handler file really names the chord and that nothing else in `src/` spells it.

### 4.5 Assertions (B1; `quickcreate.assert.ts`)
The gate as a pure function (no company, gate screen, overlay, typing, repeat, Shift each answer "not mine"); `combo()` of Option+Q on a Mac and of AltGr+Q; the menu model (filtering by label and alias; the grid walk; empty states); the host is a single `useFallbackKeys` that never names `active` (a source scan, as the nav chord host's); `overlayUp()` is true while `.qc-overlay` is mounted. **Recipes:** `qc-open` (chord on the Gateway, menu in light, dark, comfortable), `qc-create` (type "inv", Enter opens the Sales voucher, crumb `Gateway › Voucher`), `qc-in-field` (chord inside a voucher field does nothing), `qc-click` (the top-bar button).

---

## 5. The navigation stack: policy, dirty signal, de-dup, crumb

### 5.1 Root cause, restated with the two additions [C, V]
See section A, item 1, above. The fix keeps the one safety goal (**never silently drop a half-keyed entry**) and changes what it is decided on: not "an entry form exists in the stack" but "something has been typed in it and not saved".

### 5.2 Three kinds of move
| Source (`NavSource`) | Who | Meaning |
|---|---|---|
| **`'dest'`** | the left menu (rows, `+`, pins, Quick access, saved views), the Favourite slots `Ctrl+Alt+1..9`, the universal `+`, the Reports Center's rows and its builder's opens | *Go somewhere*: the destination replaces the stack. |
| **`'page'`** | the page's `+ New` and its dropdown | *Start one from here*: a drill from the list, so Esc comes back to the list. |
| **`'goto'`** | Go To (`Ctrl+K`, `Alt+G`) | *Peek and return*: Tally's own habit, kept. |
| (no source: plain `st.push`) | the Gateway's letters and rows, Enter on a report row, F-key panels, Dashboard tiles | drill-downs and the Tally model, **unchanged**. |

### 5.3 The decision (`core/navPlan.ts`, pure, B1)
```ts
export type NavSource = 'dest' | 'page' | 'goto';
export type NavPlan =
  | { kind: 'stay' }                                  // the target is already on top: do nothing (the caller refocuses)
  | { kind: 'pop-to'; index: number }                 // nav = nav.slice(0, index + 1)
  | { kind: 'replace' }                               // goHome(); push(target)
  | { kind: 'push'; collapse: number[] }              // push; first drop these clean middle entries (depth cap)
  | { kind: 'ask'; then: 'replace' | { popTo: number } };   // unsaved work is in the way: show the guard

export const MAX_DEPTH = 6;
export function planNavV2(stack: readonly Screen[], target: Screen, source: NavSource, dirty: (index: number) => boolean): NavPlan;
```
`dirty(i)` is `dirtyAt(i) ?? mayHoldWork(stack[i])`: **a registered probe is authoritative; a screen that has not registered stays conservative** ("an entry form may hold work"), so a screen nobody wired can never lose an entry.

| Situation | `dest` | `page` | `goto` |
|---|---|---|---|
| target key = top key | stay | stay | stay |
| target already in the stack at `k`, everything above `k` clean | `pop-to k` | `pop-to k` | `pop-to k` |
| target in the stack at `k`, something above `k` dirty | **ask** (then `pop-to k`) | **push** (peek, the dirty entry stays below) | **push** |
| target absent, nothing in `stack[1..]` dirty | **replace** | **push** | **push** |
| target absent, something dirty | **ask** (then `replace`) | **push** | **push** |
| a push that would exceed `MAX_DEPTH` (6) | n/a (replace keeps it at 2) | collapse | collapse |

*Collapse* removes the **oldest clean entries after index 0** until the depth is back at 6, never a dirty entry and never the home screen. *Key equality* is `screenKey`, so a voucher being altered (`editId`) is a different destination from a blank one of the same type, and the flat register differs from the month matrix. Pop-to unmounts only the clean screens above the target and **keeps the target mounted** (its scroll, its filter, its cursor). Esc from a replaced destination goes Home; from a `page` push it returns to the list.

### 5.4 Executing it (`ui/navigate.ts`, B1)
```ts
export function requestNav(target: Screen, source: NavSource): void
```
Reads `useStore.getState().nav`, builds `dirty` from `dirtyAt` and `mayHoldWork`, calls `planNavV2`, then:
1. **Rights first.** `screenDenial(accessCtxOf(st), target)`: if refused, `raise(denial)` and **return before touching the stack** (today's `goHome` then a refused `push` leaves the user at the Gateway with a toast).
2. `stay`: nothing (and `refocus()`, as the rail does today when the screen is already up).
3. `pop-to`: `useStore.setState({ nav: nav.slice(0, i + 1) })`. `replace`: `goHome(); push(target)`. `push`: drop the `collapse` indices, then `push(target)`.
4. `ask`: `useNavGuard.setState({ pending: { target, source, then } })` (`state/navGuard.ts`, a tiny zustand store); the **`NavGuardHost`** (mounted once in `AppShell`) shows the prompt.

Callers (each one line):
| Caller | Becomes |
|---|---|
| `useRail.go` (rows, `+`, pins, Quick access) | `requestNav(scr, 'dest')` |
| `reportViews/Launcher.tsx` (after its `pop`) | `requestNav(target, 'dest')` |
| `favourites.tsx` `FavouritesHost` and `openShortcutsScreen` | `requestNav(to, 'dest')`; `decideChord` keeps its pure job (ignore / unavailable / stay / go with the target) and **no longer returns a `plan`**; the plan is computed at execution, where the probes live |
| `CommandPalette.go` | `onClose(); requestNav(to, 'goto')` |
| `PageActions`, the Reports Center, the universal `+` | `'page'` / `'dest'` as in 4.3 and 3.2 |

### 5.5 The "Leave this entry?" prompt (`ui/navGuard.tsx`, B1)
- A modal in the same family as `AcceptPrompt` (the bar above the status line): **"Leave this entry? Entries will be lost."** with a second line naming the target ("Opening Balance Sheet") and the first dirty entry's crumb ("You are keying New Sales"). Two choices: **Stay** (default) and **Leave**. `AcceptPrompt` gains optional `yes`, `no` and `initial` props (backwards compatible) so the same component and look serve both.
- **Keys:** `S`, `N`, `Esc` and `Enter` on the default = Stay; `L` and `Y` = Leave; `←` `→` `Tab` toggle. It is a modal: a `useKeys` handler mounted only while it is up, swallowing every other key (F-keys included, so a browser never reloads behind it). `overlayUp()` sees it through the `.nav-guard` class (registered).
- **Leave** runs the stored plan (`replace`, or `pop-to k`): the dirty screens unmount and their entries are gone, which is exactly what the user agreed to. **Stay** clears `pending` and changes nothing: not the stack, not focus.
- Esc on a voucher keeps its own, Tally-faithful "Quit ? (entries will be lost)" with Yes preselected: **unchanged**. The new prompt exists only for the navigation sources that did not have one (the rail, a slot, `+`).

### 5.6 The dirty signal (`state/dirty.ts`, scaffolded and working; B1 wires the screens)
`useDirty(() => boolean)` registers a probe for the screen's stack slot (`SlotIndexCtx`); the probe reads current state **when asked**, only at navigation time (never during render, so it costs nothing per keystroke), and is removed when the screen unmounts; a probe that throws counts as dirty. Who registers, and the probe:
| Screen | Probe |
|---|---|
| `VoucherScreen` (new) | the existing `dirty`: any ledger, item, party or narration entered |
| `VoucherScreen` (alteration) | **changed since load**: a signature of rows, items, party, narration and dates taken once the editor has loaded, compared lazily (today's `dirty && !editing` guard makes Esc silent for alterations; the probe makes the rail honest, and Esc is left as it is) |
| `useTallyForm` (every master, Company Alteration, Print Configuration...) | `values` differ from `initial` or `ids` from `initialIds`: registered **inside the hook**, so ~18 screens get it with no change of their own |
| Attendance / Payroll vouchers | their existing `dirty` |
| `BulkEntry` | `rows.length > 0 && !done` (its own Esc guard's test) |
| `ImportData` | `plan !== null` |
| `Manufacture`, `BudgetEditor`, `LeaveApply`, `CustomReport` (B3) | their field state vs initial |

**A drift guard** (`navstack.assert.ts`): every screen of kind `create` or in `HOLDS_WORK` either imports `useDirty` / uses `useTallyForm`, or is on a named allow-list with a reason; the list is short and anything not on it fails the suite. The conservative fallback keeps an unwired screen safe meanwhile.

### 5.7 De-dup, depth, crumb
- **De-dup** is the `pop-to` rows of 5.3. It removes `Register › Outstandings › Register › Outstandings`; Go To "Gateway" pops to home.
- **Crumb.** `StatusBar` shows the **last four segments** with a leading `… › ` when the path is longer, and puts the **full path in the element's `title`**. A pure `trimCrumb(names: string[], max = 4): { text: string; full: string }` in `core/navPlan.ts`; `AppShell` passes `crumb` and a new `crumbTitle`. **Segments stay non-clickable** (a click would pop several screens and bypass each one's Esc guard, as before). **The crumb words are not renamed in this wave**: `waitCrumb` in the shots harness and dozens of recipes match `Voucher`, `Register`, `Outstandings` exactly, and the shortened crumb still ends with the screen on top, so `endsWith(' › ' + label)` keeps holding; richer labels (`labelFor`) belong in the tooltip only. A recipe that inspects an early segment of a deep stack is fixed by the builder.
- `MAX_DEPTH` stays at 6 for peeks; the replace policy keeps the common path at 2.

### 5.8 Assertions and recipes (B1)
- **`favourites.assert.ts` is updated deliberately**, where the policy changes: section 7 (`planNav` returning `home-push` / `push` / `stay`) becomes the table of 5.3; section 8 (`decideChord` with `plan`) loses `plan`; the "mayHoldWork" lines stay as the fallback test. State in the commit message which lines changed and why.
- **New `navstack.assert.ts`**: the full 5.3 table as data (every cell), including: a pristine voucher then a rail click replaces; a typed voucher then a rail click asks and Stay changes nothing; Go To three times without a voucher grows by three until the cap and never repeats a screen; `pop-to` keeps the target; collapse never removes a dirty or index-0 entry; a refused target changes nothing; `trimCrumb` (3 segments: whole; 6 segments: `… ›` plus the last four; the title carries all six); a probe that throws is dirty; an unregistered create-kind is dirty; the drift guard of 5.6.
- **Recipe `rail-walk`** (the owner's screenshot, as a test): pristine F8, then six rail clicks (Invoices, Receivables, Payments Received, Balance Sheet, Day Book, Receivables) leaves the crumb at most `Gateway › X` and one Esc gets home; Go To the same report twice does not grow the crumb; a half-typed voucher plus a rail click shows the dialog, Stay changes nothing, Leave replaces; after Accept (the voucher resets to a blank one) a rail click replaces without asking. New file `scripts/shots/recipes/navstack.mjs`.

---

## 6. Filters and views: "save the filter in reports"

### 6.1 The gap, precisely [C, R]
A Zoho custom view stores **filter conditions**, columns and who may see it. Our saved view (`Alt+V`, `core/savedViews.ts`) stores F12 options, added columns, basis and optionally the period, **explicitly not a filter**, and it lives on one device. The existing type-to-filter (`/`, `Ctrl+F`, `ui/useReportFilter.ts`) matches printed text only, hides rows with CSS, and says "totals unfiltered". The owner's "save filter in reports" is therefore a real gap. **The existing feature keeps working unchanged**; this adds a second, structured filter beside it and puts it into views.

### 6.2 The model (`core/reportFilter.ts`, scaffolded as types; B2 writes the engine)
```ts
type Condition =
  | { field; kind: 'text';   op: 'contains' | 'notContains' | 'startsWith' | 'equals'; value: string }
  | { field; kind: 'number'; op: 'eq' | 'neq' | 'gt' | 'gte' | 'lt' | 'lte' | 'between'; value: number; value2?: number }
  | { field; kind: 'date';   op: 'before' | 'after' | 'between'; value: string; value2?: string }   // ISO dates; between is inclusive
  | { field; kind: 'enum';   op: 'is' | 'isNot'; value: string };
interface FilterGroup { join: 'and' | 'or'; items: (Condition | FilterGroup)[] }
interface ReportFilter extends FilterGroup { v: 1 }
interface FieldDef<T> { id; label; kind; get: (row: T) => string | number | null | undefined; options?: { v; label }[] }
```
- **Limits (constants exist):** 12 conditions in all, nesting depth 2 (the root group plus groups inside it), text cut at 80 characters.
- **Text** compares case-insensitively after NFKC and with whitespace folded. **Numbers** are the report's own figures (rupees, quantities) as numbers, never display text. **Dates** are ISO strings.
- **A condition naming a field the report does not have matches nothing** (a stale or hand-edited view can narrow, never silently widen).
- **Fields are the report's own sort columns.** Each report already hands `useColumnSort` a list of `{ key, label, get }` (Day Book: date, particulars, type, number, debit, credit, narration; the register: date, particulars, number, amount; Bills: date, ref, party, pending amount, due, overdue days). `FieldDef` is that shape plus a `kind`, so filter and sort read the same accessors, and the filter runs **before** the sort.

### 6.3 Where it applies, and what the totals say (the decision the owner will notice)
**Opt-in per report.** A report that has not given the engine its rows and fields has no filter, no Filter button and no chips (a statement such as the Balance Sheet has no rows to filter). **Wave A allow-list: Day Book, the voucher registers (both levels), Ledger Vouchers, Receivables and Payables (Bills).** More are one hook call each, with a field list and a test.

The hook (`ui/rowFilter.ts`, B2): `useRowFilter(reportKey, rows, fields) → { rows, matched, total, filter, set, clear }`. The report passes its rows **after** its own guard (`postsToLedgers`, `!v.optional`, bill-wise outstanding), gets the narrowed rows back, and builds *everything else* from those: month counts and amounts, the Total row, the `%` columns, the selection, the sort, Print, Export. It publishes `{ value, matched, total, open, clear }` as `ReportActions.filter` so the header can draw the bar.

| Question | Answer |
|---|---|
| **Totals** | Computed over the filtered rows, by the report itself (never by summing printed text): the Total row, the month totals, the % of total. While a filter is active the total row's label gets ` (filtered)` (it still satisfies `labelsATotal`'s "Total (...)" form, so the shell's total emphasis and the text filter's never-hide-a-total rule keep working), the sub-line gains `· Filtered: <description>`, and the bar shows a **Filtered** badge with "N of M". |
| **Print and Export** | They honour the filter **by construction**: the filtered-out rows are not rendered, so the clone and the CSV never contain them. The printed header's sub-line carries the one-line description (e.g. `Filtered: Amount ≥ 1,00,000 and Party contains "sharma"`). (The text filter's old behaviour, hidden rows and whole-report print, is unchanged and still says so.) |
| **Optional vouchers** | The guard is unchanged and runs first; the filter only ever **narrows** what the report already decided to show (a property test: every output row is an input row). The Memorandum, Optional and Post-dated registers are unchanged. |
| **The two filters together** | They compose (the text filter then hides rows on screen). The "totals unfiltered" note of the text filter stays true of the text filter alone; when a structured filter is active the report's totals are the structured filter's. |
| **Persistence** | **A live filter is held in memory for the app session per company and report, and is never written to storage.** A filter that silently came back next week would be a report that silently misses rows. Only a **saved view** persists a filter; recalling "All" clears it. The bar is always visible while a filter is on and `Clear` is one click. |

### 6.4 The Filter button, chips and panel (`ui/header/FilterBar.tsx`, B2)
- **`belowHead` slot, draws only for a report that published `filter`.** A "Filter" button (funnel icon; tooltip "Filter rows (`Alt+F12` range, `Ctrl+F12` value or name)"), then a **removable chip per condition** ("Amount ≥ 1,00,000 ×", "Party contains sharma ×", with a plain `and` / `or` between), a **Clear** button, the "N of M" count and the **Filtered** badge. `data-noprint="1"`.
- **The panel** (a modal in the family of the F12 panel, `ui/reportConfig.tsx` `F12ConfigPanel`; read its keyboard conventions and match them): one row per condition `[Field ▾] [Operator ▾] [Value]`; **live apply** as the F12 panel does; `A` adds a row, `Alt+D` deletes the highlighted row (the app's delete chord), `J` switches the group between and / or, `G` adds a group (depth 2), `R` resets, `Esc` closes. The operator list follows the field's kind; a date value takes the app's date mnemonics (`parseDateInput`); an enum field offers its options.
- **Keys.** `Alt+F12` opens the panel with a **range** condition added (the first number or date field: amount, date, quantity, days overdue); `Ctrl+F12` opens it with a **value** condition added (the first text or enum field: party, type, narration). Both are free in `src/` (grep) and swallowed today by `App.tsx`'s generic F-key rule (which also stops a browser reloading). They are handled **only while the report on top published a filter** (`useScreenKeys`, top-gated; everywhere else they stay swallowed and do nothing, exactly as today). On a Mac `⌘F12` is the same chord (`combo()` folds ⌘ into Ctrl). Both are in `core/shortcutRegistry.ts` (`filterRange`, `filterValue`, `sheet: 'report'`, `live: false` until B2 binds them).

### 6.5 Saved views carry the filter (`core/savedViews.ts`, B2)
The `SavedView` type already carries the new optional fields (scaffold, types only). B2 makes them real, **additively and with the existing sanitising discipline** (a corrupt or older blob must never poison a report; an older build must still read a newer view and ignore the fields it does not know).

| Field | Meaning |
|---|---|
| `filter?: ReportFilter` | Sanitised against the report's own field list on load (unknown field, wrong kind or operator, too many conditions, over-long text: dropped condition by condition, never the whole view). |
| `scope?: 'me' \| 'company'` | Absent means `'me'`. **`me`**: this device, in the existing `tw-report-views-v1:<cid>` store (unchanged). **`company`**: a synced list on the company row, the Dashboard's precedent (`Company.dashboard`): `Company.reportViews?: SavedViewStored[]` (B2 adds the optional field to `types.ts`; a restore, a sync and a split must carry it through untouched, so `master-resave` / `companyAlter` copy it), written through `saveCompany`, which the write guard answers with **Administration** (`TABLE_AREA.companies = 'admin'`). Offered in the save dialog only to an operator who may write it (security off: everyone). At most 40 company views (60 personal, the existing `MAX_VIEWS`). |
| `category?: string` | The Reports Center category id; absent means the base report's own. |
| `by?: string` | The operator's display name at save time; absent shows as "You". Never used for rights. |
| `custom?: boolean` | Made by "Create New Report" (listed under My reports with a "Custom" tag). |
| *(no `favourite`)* | **A favourite view is a pinned view**: the star in the views dropdown and in the Reports Center is the left menu's pin (`reportViews/pin.ts` `togglePinnedView`, limit `PINS_MAX` = 8). One favourites list, not two. |

**Reading.** `useViews(cid)` returns the personal list plus the company list, tagged by scope. A company view is **read-only** for an operator without the right (rename, replace and delete are disabled; "Save a copy" makes a personal one). `captureView` takes `live.filter`; `viewMatches` compares the canonical JSON of the filter (so the dropdown's chip names the view the report is showing exactly); `planApply` returns the filter; `applyView` writes it through a new bridge (`readReportFilter` / `writeReportFilter`, the same pattern as `reportViews/bridge.ts`, because the report's filter store lives in `ui/rowFilter.ts` and importing it from the views code would be a cycle). A view saved **before** this change has no `filter` and recalls exactly as before.

**System ("default") filters.** Read-only, built per report (`SystemFilterView`, `build(env)` is evaluated each time because "This month" moves), listed under "Default filters" in the dropdown and applying the **filter only** (they never touch F12):
- Registers and Ledger Vouchers: **All**, **This month**, **Last month**.
- Day Book: **All**, **Today**, **This week**, **This month**.
- Receivables and Payables: **All**, **Overdue**, **Due this week**, **Not yet due**.

### 6.6 The views dropdown (`ui/header/ViewsTitle.tsx`, B2)
- **`titleExtra` slot, drawn only for a report that published `openViews`.** The button shows the current view: **All** (no view, no filter), the **view's name** (when `activeView` matches), or **Custom filter** (a filter with no matching view), with a caret. It sits beside the title ("Sales Register  All ▾").
- **The dropdown**, in Zoho's three bands: **Favourites** (the pinned views of this report), **Default filters** (the system views above), **My views** and **Shared views** (company); a search box once there are more than ten entries in all; at the bottom **+ New view…** (opens the existing Alt+V save dialog, extended with Scope, Category and the filter summary) and **Manage views…** (the existing Alt+V list). Recently used views are remembered per device (a small list in the same store).
- The Alt+V list and its keys **stay exactly as they are** (N U R P Delete 1-9): the dropdown is a second door onto the same store, for the mouse.

### 6.7 The register's flat "All" level (B2, `ui/reports/financial.tsx` `RegisterReport`)
A register is a month matrix with a month drill (Tally-faithful, and what the Gateway opens). "Click Sales and see all sales" needs a flat list to land on, and a filter on a month matrix would only narrow the counts. So:
- **Screen parameter `all?: boolean`** (already in the union). The rail rows and the page header push `{ s: 'register', base, all: true }`; the Gateway's `{ s: 'register', base }` still opens the month matrix. A saved view opens whichever it was saved from (`sanitiseScreen` keeps `base` only, so a view recalls the month matrix unless B2 extends `SCREEN_PARAMS.register.keep` with `all`; B2 decides and asserts).
- **The flat level** lists every voucher of the period in one table, with the same columns as today's month-voucher level (Date, Particulars, Vch No., Amount, plus narration and `%` per F12), the same sort (`useColumnSort`, `Alt+S`), selection, Enter-to-open and `Alt+2` duplicate, and a Total row.
- **A two-button toggle "By month | All vouchers"** in the register's own area (in the `chip` row), real buttons in the Tab order with Left / Right inside the pair. **No new chord** (the registers already spend Space, P, E, O, R on a marked selection). The state is the screen's: initial `scr.all ? 'all' : F12 extra level (default 'months')`; the toggle sets it and writes the F12 extra `level`, so **saved views capture it through `config.extras`** with no schema change.
- **Esc.** At the flat level Esc leaves the screen (it is the top level); at the month-voucher level Esc returns to the month matrix, exactly as now (the register's own two-level Esc is untouched).
- The filter (6.3) applies to vouchers **before** they are grouped, so month counts and amounts follow it.
- The fuller "status, due date, balance" columns an owner expects of an invoice list are the UX analysis's N2 and **out of scope** here; the flat level is the landing place they will extend.

### 6.8 Assertions and recipes (B2)
- **`rowfilter.assert.ts`**: the engine (every operator, each kind; AND, OR, a nested group; text folding; a stale field matches nothing; the 12-condition and depth limits; a property test that the output is an ordered subsequence of the input), `describeFilter` text, the sanitiser (corrupt blob, wrong shapes, an older view without `filter`), `viewMatches` with and without a filter, scope handling (a company view is read-only for a restricted role; `reportViews` of both scopes merge without id collisions), system views' dates (This month on 31 March, Overdue against `min(period end, today)` as the ageing already does), and the **totals contract** (a register with a filter has totals equal to the sum of its filtered rows, and the optional-voucher guard excludes the same vouchers with and without a filter).
- **Source scans**: `Alt+F12` and `Ctrl+F12` are bound only in the filter bar's file; no live filter is written to storage.
- **Recipes** (`scripts/shots/recipes/rowfilter.mjs`): `filter-register` (Sales Register, `Alt+F12`, Amount above a figure, the chips, the Filtered badge, Total row relabelled), `filter-print` (the clone carries only the filtered rows and the description line), `views-dropdown` (open, default filters, save a view with a filter as company-wide, recall), `register-all` (rail "Invoices" opens the flat level; the toggle; Esc), each in light and dark, compact and comfortable.

---

## 7. The Reports Center

### 7.1 What Zoho has, and what we build [R]
Zoho's Reports Center is a finding aid over a fixed catalogue, plus saved custom reports. Its left panel (Home, Favorites, Shared Reports, My Reports, Scheduled Reports, Advanced Analytics), a category list, a table (star, name, category, created by, last visited), a search box and "Create New Report". **We build everything that is real and nothing that is not:** no Scheduled Reports (no mail sender or job runner exists in `backend/`) and no Advanced Analytics (a separate product). The 15 categories of the owner's screenshot become **14** that fit what Comsky has (Appendix C): Zoho's "Projects and Timesheet", "Currency" and "Automation" have nothing behind them here (no projects, no multi-currency; Automation could not be confirmed to exist at all), and GST and TDS & TCS are each large enough to be their own category.

### 7.2 The screen (`ui/reportsCenter/ReportsCenter.tsx`, B3)
```
Reports Center                                         [Search reports]  [Create New Report]
┌ left panel ─────────┬ main ───────────────────────────────────────────────────────────────┐
│ Home                │ All Reports (99)                                                     │
│ Favourites       3  │ ☆  Report                       Category        Created by  Last visit│
│ Recently visited    │ ☆  Balance Sheet                Business Overv… System      Today    │
│ Shared reports   1  │ ★  Sales Register               Sales           System      Yesterday│
│ My reports       2  │ ☆  Monthly sales by party       Sales           You         12-Oct   │
│ ─────────────────   │ ...                                                                  │
│ REPORT CATEGORY     │                                                                      │
│ Business Overview 7 │                                                                      │
│ Sales             5 │                                                                      │
│ ...                 │                                                                      │
└─────────────────────┴──────────────────────────────────────────────────────────────────────┘
```
- **Left panel.** *Home* (the full table, with a Favourites strip and a Recently visited strip above it when they have rows), *Favourites*, *Recently visited*, *Shared reports* (company-scope views), *My reports* (personal views, built or saved), then the **14 categories with counts**. Counts and rows are **what this operator in this company may open**: each entry is dropped when a `needs` feature is off or `screenDenial(ctx, entry.to)` refuses it (the same two tests as the menu), so the counts differ by company and by role. **No** Scheduled Reports or Advanced Analytics entries.
- **Main table.** Columns: a star, the report name, its category, *Created by* (**System** for the catalogue; a view's `by`, else "You" for a personal view, "Company" for a company one), *Last visited* ("Today 11:20", "Yesterday", a date, or a dash). Sorted by name within the selected panel item; *Recently visited* by time; *Favourites* in pin order. A saved view shows its **base report** in a quiet sub-line and a "Custom" tag if it was built.
- **Star = pin.** The star toggles the left menu's pin (`state/recents.ts togglePin`, a report's `to`, or a view's `reportView` launcher via `togglePinnedView`). **One favourites list**; limit `PINS_MAX` (8): the 9th star is disabled with the title the menu already uses ("Pinned list is full (8)"). `state/reportVisits.ts` exposes `isFavouriteReport` / `toggleFavouriteReport` as thin delegates to the pin store.
- **Search** (autofocused on open): "Search reports". Type to filter the table across all categories (label and aliases, every word, ranked with `paletteRank.rankPalette` so the vocabulary and the tiers match Go To's), views included; `↓` moves into the table; `Enter` opens the highlighted row.
- **Open a row:** `requestNav(entry.to, 'dest')`; a view through its `reportView` launcher.
- **Header action:** the Reports Center's own `QC_PAGES` entry makes `PageActions` draw **Create New Report** (B3 renders its header with `PageHeaderSlot`; B1's component supplies the button), and the entry is also in the universal `+` (General: "Custom Report").

**Keys** (screen-scoped, `useScreenKeys`; no new chord): the search box is focused on open; `↓` / `↑` walk the table; `Enter` opens; `Space` stars; `Esc` clears the search, then leaves; `Tab` moves search, left panel, table; in the left panel `↑` `↓` choose and `Enter` selects. Each is listed on the `?` sheet through `core/shortcutRegistry.ts` (`sheet: 'report'`, B3 appends).

### 7.3 The catalogue (`ui/reportCatalog.ts`, B3; Appendix C is the whole file, compiled)
99 reports in 14 categories. Each entry: stable `id`, `label` (Tally's word where there is one; the Zoho word travels in `aliases` so "invoices", "bills", "debtors", "udhaar", "aging" all find it), `category`, **`to` exactly what the Gateway (or Go To) pushes**, `needs` (every company feature that must be on, as the Gateway gates it, nested submenus included: Sales Orders Outstanding needs `inventory` and `orders`), the Gateway `key` where it has one, `aliases`.

**Business Overview** (`business`, 7): Profit, position, cash and ratios.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Balance Sheet | `{ s: 'balanceSheet' }` |  | `B` |
| Profit & Loss A/c | `{ s: 'profitLoss' }` |  | `P` |
| Cash Flow | `{ s: 'cashFlow' }` |  |  |
| Funds Flow | `{ s: 'fundsFlow' }` |  |  |
| Ratio Analysis | `{ s: 'ratioAnalysis' }` |  | `R` |
| Top-N Analysis (MIS) | `{ s: 'topMis' }` |  |  |
| Profitability Analysis | `{ s: 'profitability' }` |  |  |

**Sales** (`sales`, 5): Invoices, returns and orders to customers.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Sales Register | `{ s: 'register', base: 'Sales' }` |  |  |
| Credit Note Register | `{ s: 'register', base: 'CreditNote' }` |  |  |
| Sales Order Register | `{ s: 'register', base: 'SalesOrder' }` | orders |  |
| Sales Orders Outstanding | `{ s: 'pendingOrders', base: 'SalesOrder' }` | inventory, orders |  |
| Delivery Note Register | `{ s: 'register', base: 'DeliveryNote' }` | inventory |  |

**Receivables** (`receivables`, 5): Who owes you, how late, and what you were paid.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Receivables (Bills Outstanding) | `{ s: 'outstandings', side: 'receivables' }` | billwise |  |
| Ageing Analysis (buckets) | `{ s: 'ageing' }` | billwise |  |
| Reminder Letters & Confirmation | `{ s: 'reminders' }` | billwise |  |
| Interest Calculation | `{ s: 'interest' }` |  |  |
| Receipt Register | `{ s: 'register', base: 'Receipt' }` |  |  |

**Payables & Purchases** (`payables`, 7): Bills, payments and what you owe suppliers.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Purchase Register | `{ s: 'register', base: 'Purchase' }` |  |  |
| Debit Note Register | `{ s: 'register', base: 'DebitNote' }` |  |  |
| Purchase Order Register | `{ s: 'register', base: 'PurchaseOrder' }` | orders |  |
| Purchase Orders Outstanding | `{ s: 'pendingOrders', base: 'PurchaseOrder' }` | inventory, orders |  |
| Receipt Note Register | `{ s: 'register', base: 'ReceiptNote' }` | inventory |  |
| Payables (Bills Outstanding) | `{ s: 'outstandings', side: 'payables' }` | billwise |  |
| Payment Register | `{ s: 'register', base: 'Payment' }` |  |  |

**Inventory** (`inventory`, 9): Stock on hand, movement and reorder.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Stock Summary | `{ s: 'stockSummary' }` | inventory | `S` |
| Godown Summary | `{ s: 'godownSummary' }` | inventory |  |
| Stock Item (book) | `{ s: 'pickMaster', kind: 'stockItem', goal: 'book' }` | inventory |  |
| Movement Analysis | `{ s: 'movementAnalysis' }` | inventory |  |
| Stock Query | `{ s: 'stockQuery' }` | inventory |  |
| Reorder Status | `{ s: 'reorderStatus' }` | inventory |  |
| Stock Ageing Analysis | `{ s: 'stockAgeing' }` | inventory |  |
| Stock Journal Register | `{ s: 'register', base: 'StockJournal' }` | inventory |  |
| Physical Stock Register | `{ s: 'register', base: 'PhysicalStock' }` | inventory |  |

**Inventory Valuation** (`valuation`, 2): Stock at cost under each method.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Stock Valuation Methods | `{ s: 'stockValuation' }` | inventory |  |
| Godown-wise Stock (valuation) | `{ s: 'godownStock' }` | inventory |  |

**Banking** (`banking`, 8): Cash, bank, cheques and reconciliation.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Cash/Bank Book(s) | `{ s: 'cashBankBook' }` |  |  |
| Bank Reconciliation | `{ s: 'brs' }` |  |  |
| Cheque Printing & Register | `{ s: 'cheque' }` |  |  |
| Cheque Books | `{ s: 'chequeBooks' }` |  |  |
| Post-Dated Cheques | `{ s: 'pdcRegister' }` |  |  |
| Payment Advice | `{ s: 'paymentAdvice' }` |  |  |
| Deposit Slip | `{ s: 'depositSlip' }` |  |  |
| Contra Register | `{ s: 'register', base: 'Contra' }` |  |  |

**GST** (`gst`, 14): Returns, reconciliation and e-invoicing.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| GSTR-1 | `{ s: 'gstr1' }` | gst |  |
| GSTR-3B | `{ s: 'gstr3b' }` | gst |  |
| GSTR-2B Reconciliation | `{ s: 'gstr2b' }` | gst |  |
| GSTR-9 Annual Return | `{ s: 'gstr9' }` | gst |  |
| HSN Summary | `{ s: 'hsnSummary' }` | gst |  |
| e-Invoice / e-Way Bill | `{ s: 'einvoice' }` | gst |  |
| GST Validation / Exceptions | `{ s: 'gstValidation' }` | gst |  |
| GSTR-3B vs 1 vs Books Recon | `{ s: 'gstRecon' }` | gst |  |
| Bill of Supply | `{ s: 'billOfSupply' }` | gst |  |
| Ineligible ITC (17(5)) | `{ s: 'itcReport' }` | gst |  |
| GSTR-1 Advances (11A/11B) | `{ s: 'gstAdvances' }` | gst |  |
| Documents Issued (Table 13) | `{ s: 'gstDocsIssued' }` | gst |  |
| Reverse Charge Register | `{ s: 'rcmRegister' }` | gst |  |
| ITC Reversal (Rule 37) | `{ s: 'itcReversal' }` | gst |  |

**TDS & TCS** (`tds`, 10): Deduction, collection, challans and certificates.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| TDS Outstanding | `{ s: 'tdsReport' }` | tds |  |
| TDS Challans & Returns | `{ s: 'tdsReturns' }` | tds |  |
| TDS Deductee-wise | `{ s: 'tdsDeductee' }` | tds |  |
| TDS Exceptions / Uncertain | `{ s: 'tdsExceptions' }` | tds |  |
| TDS Quarterly Tracker | `{ s: 'tdsQuarterly' }` | tds |  |
| TDS Challan Reconciliation | `{ s: 'tdsChallanRecon' }` | tds |  |
| TDS Interest & Late Fee | `{ s: 'tdsInterest' }` | tds |  |
| Form 16A Certificates | `{ s: 'form16a' }` | tds |  |
| TCS Outstanding | `{ s: 'tcsReport' }` | tcs |  |
| TCS 206C(1H) Turnover | `{ s: 'tcs206c' }` | tcs |  |

**Payroll & HR** (`payroll`, 11): Pay, attendance, leave and statutory.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Pay Sheet | `{ s: 'paySheet' }` | payroll |  |
| Payslip | `{ s: 'payslip' }` | payroll |  |
| Payroll Statement | `{ s: 'payrollStatement' }` | payroll |  |
| Attendance Register | `{ s: 'attendanceRegister' }` | payroll |  |
| Salary Disbursement | `{ s: 'salaryDisbursement' }` | payroll |  |
| Statutory (PF / ESI / PT) | `{ s: 'payrollStatutory' }` | payroll |  |
| Employee Profile / Head Count | `{ s: 'headCount' }` | payroll |  |
| Multi-Period Comparison | `{ s: 'payrollComparison' }` | payroll |  |
| Gratuity Provision | `{ s: 'gratuity' }` | payroll |  |
| Leave Register | `{ s: 'leaveRegister' }` | payroll |  |
| Leave Balance | `{ s: 'leaveBalance' }` | payroll |  |

**Accountant** (`accountant`, 9): Books, ledgers and trial balance.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Trial Balance | `{ s: 'trialBalance' }` |  |  |
| Day Book | `{ s: 'daybook' }` |  | `K` |
| Journal Register | `{ s: 'register', base: 'Journal' }` |  |  |
| Ledger | `{ s: 'pickMaster', kind: 'ledger', goal: 'book' }` |  |  |
| Group Summary | `{ s: 'pickMaster', kind: 'group', goal: 'book' }` |  |  |
| Chart of Accounts | `{ s: 'coa' }` |  | `H` |
| Memorandum Register | `{ s: 'vchRegister', mode: 'Memorandum' }` |  |  |
| Optional Vouchers | `{ s: 'vchRegister', mode: 'Optional' }` |  |  |
| Post-dated Vouchers | `{ s: 'vchRegister', mode: 'PostDated' }` |  |  |

**Cost Centres & Budgets** (`costing`, 5): Projects, departments and budget variance.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Cost Centre Analysis | `{ s: 'costAnalysis' }` | costCentres |  |
| Cost Category / Centre Report | `{ s: 'costCentreReport' }` | costCentres |  |
| Cost Centre Summary | `{ s: 'costCentreSummary' }` | costCentres |  |
| Cost Centre P&L Breakup | `{ s: 'costPl' }` | costCentres |  |
| Budget vs Actual | `{ s: 'budgetVsActual' }` |  |  |

**Activity & Audit** (`activity`, 4): Edit log, cancelled entries and checks.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Edit Log (Audit Trail) | `{ s: 'auditLog' }` |  |  |
| Cancelled / Deleted Vouchers | `{ s: 'cancelledRegister' }` |  |  |
| Exception Reports | `{ s: 'exceptions' }` |  |  |
| Sync Mismatch Report | `{ s: 'syncConflicts' }` |  |  |

**CRM** (`crm`, 3): Leads, follow-ups and the pipeline.

| Report | Opens | Needs | Gateway key |
|---|---|---|---|
| Leads | `{ s: 'leads' }` |  |  |
| Pipeline Summary | `{ s: 'pipeline' }` |  |  |
| Follow-ups Due | `{ s: 'followUps' }` |  |  |


**Scope, stated:** a *report* is a screen that displays books or compliance data. Not in the catalogue, by an explicit list with a reason (Appendix C): the Dashboard (home), tools and utilities (Bank Statement Import, Bulk Entry, Import and Export Data, Data Check, Split Company, Compare with Tally Data), setup (Company, Features, GST Rate Setup, Price Levels, Voucher Types, Security, Print Configuration), personal tools (Getting Started, Snippets, Favourite Shortcuts), the Budgets master list, choosers and drill-downs that need a parameter (Ledger Vouchers, Stock Item Vouchers, Group Summary by id, Bills of a ledger...; their entry is the picker listed under Accountant / Inventory), and the one stub (Batch & Expiry Summary, `ComingSoon`).

### 7.4 Last visited and favourites (`state/reportVisits.ts`, scaffolded as no-ops; B3 writes it)
- **Visits.** `noteReportVisit(scr)` is already called from one effect in `AppShell`, keyed on the screen key. B3 maps a screen to a catalogue id by `screenKey` equality **ignoring drill-down parameters and `all`**, and records `{ [id | 'view:<id>']: epoch ms }` in `localStorage` `TW_REPORT_VISITS_v1:<companyId>:<operator>`, newest 200, every read and write in `try / catch` (blocked storage reads as "no visits"). A screen the catalogue does not list is not a report and is not recorded. The page renders correctly with storage absent.
- **Recently visited** is this list, newest first, ten shown. It is separate from the rail's Recents (which record every screen, forms included, and are capped at 7 shown).

### 7.5 Shared reports and My reports
**My reports** = personal views (`scope: 'me'`) of every report, built or saved with Alt+V, each tagged with its base report. **Shared reports** = company views (`scope: 'company'`). Both are the `useViews` list of 6.5, opened through the `reportView` launcher (which applies the view's F12 options, columns, basis, period and filter and then replaces itself with the report). Per-role sharing, scheduling and mailing are not built (the research marks them as needing platform work).

### 7.6 "Create New Report" (`screen 'customReport'`, B3)
**The decision:** a free-form builder (data sources, parent and child modules, chart view, plan limits) is not built; "a register with my columns and my filter" is 80% of it and is exactly a saved view with a filter. The builder is therefore **a short form, then the real report**, so F12, added columns, basis and the filter are the real controls, not copies:

1. **Step 1: the form** (`CustomReport.tsx`, a `useTallyForm`, so it keeps the Tally field walk and registers its dirty probe): *Name* (the saved-view name rules), *Based on* (a popup of the reports that can have views: `LISTED_REPORTS` plus the registers, i.e. what `viewDef` knows; `base` pre-selects it from "Create from existing report"), *Category* (a popup of the 14, defaulting to the base report's), *Share with everyone in this company?* (Yes / No; offered only to an operator who may write company views), *Include the period?* (Yes / No, the existing choice). `Enter` on the last field offers **"Set up the report"**.
2. **Step 2: set it up in place.** The base report opens (`requestNav(base, 'dest')`) with a **build session** active (`ui/reportsCenter/builderSession.ts`, module state: name, base key, category, scope, includePeriod, editId). The **`ReportBuilderBar`** (mounted in `AppShell`, a strip above the status bar) reads: *Building "Name" on Sales Register. Set it up with F12, the filter and Alt+F12; then Save.* with **Save** and **Cancel** buttons. Save captures the live view (`captureView`) with the session's metadata (`custom: true`, `category`, `scope`, `by`) through `saveNewView` (personal) or `saveCompany` (company; the write guard decides), flashes "Report saved: Name", ends the session, and shows the report. Cancel ends the session. If the base report leaves the stack the session ends and says "Report not saved".
3. **Edit** (`customReport { editId }`): step 1 prefilled; step 2 recalls the view then Save replaces it (`refreshSavedView`). Delete lives in the views list (Alt+V) and in the Reports Center row menu.
4. **Rights.** The screen is `'open'`; the base list contains only reports this operator may open; a company-scope save is refused by the write guard for anyone without Administration (and the Yes / No is not offered to them).
5. **Limits.** 60 personal and 40 company views; a clear message at the limit.

### 7.7 Where it hangs off the Gateway and the menu (all registered by the scaffold; additive)
- **Gateway:** one row appended to the **Reports** section after "Display More Reports": **Reports Center**, letter **`O`** (R-e-p-**o**-rts; every other letter of the label is taken on that menu and first match wins). No row moved.
- **Left menu:** the **Reports** group's label opens it (section 1); there is no row of its own (the group header lights on it).
- **Go To:** "Reports Center" (Report) and "New Custom Report" (Create), with aliases; Favourite Shortcuts can point at both.
- **Universal `+` and `+ New`:** "Custom Report" (General) and the Reports Center page's primary.
- **Rights:** `reportsCenter` and `customReport` are `'open'` in `SCREEN_AREA` because every row is judged individually; a clerk with Accounting but not Reports still reaches the registers there.

### 7.8 Assertions and recipes (B3; `reportcatalog.assert.ts`)
Appendix C equals the catalogue (literal count per category: 7, 5, 5, 7, 9, 2, 8, 14, 10, 11, 9, 5, 4, 3 = 99); ids unique; every `to` is in the union, the router and `SCREEN_AREA`, is of kind `report`, and is not a stub; every `key` is that screen's Gateway letter; every entry's `needs` includes the Gateway's effective feature chain for the same screen; **drift guard**: every report-kind destination of the Gateway `MENUS` and of the palette is in the catalogue or on the exclusion list (so a new report cannot be forgotten); a counts-by-role test (Data Entry loses GST, payroll and the statements; a restricted role never sees a row it cannot open); search finds "udhaar", "debtors", "aging", "invoices", "bs"; the star is the pin (starring then reading the rail's Pinned shows it; the ninth is refused); `noteReportVisit` records a catalogued screen with and without `all` and never an uncatalogued one, and survives blocked storage; the builder's session ends on Cancel, on leaving the base report and on Save. **Recipes** `reports-center-home`, `reports-center-category`, `reports-center-search`, `reports-center-star`, `report-builder` (the form, set up, Save, it appears under My reports), each in light and dark, compact and comfortable; no horizontal scroll at 1280 and 1024.

---

## 8. Wave B reservations (not built; nothing here may block them)

The owner's last two paragraphs (configurable shortcut buttons; an admin settings window for theme, accent colour, fonts and size) are Wave B. What Wave A leaves in place for them:

### 8.1 The screen
`{ s: 'settings' }` is in the `Screen` union, routed to `ui/admin/SettingsScreen.tsx` (a `PlannedScreen` stub), has a crumb ("Settings") and a title, is a `SETUP`-kind screen, and is **`SCREEN_AREA 'admin'`** (administrators only; the security suite fails any screen without an entry). **Nothing links to it**: not the Gateway, not Go To, not a rail row, not `QUICK_CREATES`. `zoho-scaffold` asserts that. Wave B replaces the stub and then, in one edit each, points the rail's **Settings** group label at it (`NavGroup.to`, today `companyAlter`), adds the palette entry and the Gateway Company-menu row.

### 8.2 The shortcut registry (`core/shortcutRegistry.ts`, scaffolded)
Every new chord of Wave A is **one entry in one list** (`NEW_SHORTCUTS`: `id`, `keys`, `label`, `desc`, `find`, `sheet`, `lockedGroup`, `live`, `handler`, `configurable`). The `?` sheet and Favourite Shortcuts' locked keys are **generated from it** (only for `live` entries), so a key is never advertised before it works and nobody edits those tables again. Today: `quickCreate` (`Alt+Q`), `filterRange` (`Alt+F12`), `filterValue` (`Ctrl+F12`), all `live: false`, `configurable: false`. Wave B's settings window lists the registry, shows the locked ones, and may flip `configurable` for individual entries **never for a voucher F-key and never for a chord that predates the list**. The suite enforces: no browser-reserved chord, no `Ctrl+Alt` spelling, no duplicates, nothing else in `src/` binds a registered chord, a live entry's handler file really names it, a non-live entry is not on the sheet.

### 8.3 "Configurable shortcut buttons" means the right-hand rail, and the Tally block is frozen
`ui/widgets.tsx` `Btn` now has two **reserved, unread** fields: `id?` (a stable id so a choice is stored by id) and `movable?` (set only on optional, Comsky-only buttons, **never on the F4 to F12 block or Esc**). The rail's data is already one `Btn[]` per screen; the Favourite Shortcuts screen (`Ctrl+Alt+1..9`) is the existing configurable surface. Wave B's presets are the model: a *Tally* preset (today) and an *Owner* preset (New invoice, New bill, Record payment, Receivables, Reports); the rule is **hide or reorder only the optional second block; no Tally key changes meaning or position**.

### 8.4 Appearance (the contract, for the designer of Wave B)
- Today scheme and density are per-device `localStorage` (`state/theme.ts`: `TW_THEME_SCHEME`, `TW_THEME_DENSITY`). "The administrator of the workspace" implies **synced workspace settings** with a per-user override: a company-row field (the `Company.dashboard` / `Company.reportViews` precedent: optional, sanitised, an alteration must carry it through untouched, written under the Administration right).
- **Accent** is two tokens per scheme (`--accent`, `--on-accent`, with `--accent-soft` and `--hot`). Offer a **validated palette** (each choice pre-checked against `--panel`, `--sel` and its `--on-accent` at AA, light and dark: the contrast table in `left-nav-spec.md` section 1 item 8 is the method), **not a free colour picker**. Zoho's own control is a dark or light pane plus an accent (red, blue, green, yellow, custom hex) and **has no font or size setting** [R]; ours may exceed it.
- **Fonts** must be **bundled** (IBM Plex is; the rupee sign U+20B9 exists only in Plex `latin-ext`); a font list loaded from the network breaks offline-first. **Size** interacts with the compact row (20.9px) and `scripts/assert/visual.baseline.json`: the visual assertion needs a size dimension.
- Nothing in Wave B may hide a keycap, and nothing may put a hex into `:root` for a token that dark does not override (the light-theme rule).

---

## 9. The assertion plan and the acceptance checklist

### 9.1 What exists and what each builder adds
| Suite | State | Owner |
|---|---|---|
| `zoho-scaffold.assert.ts` (484) | **done**: the three screens everywhere, the Gateway row and its letter, Go To entries, no link to Settings, no left-menu row opens a stub, the three slots wired in order, the hosts mounted once, light overlay files, the quick-create invariants (gates, `pageKey`, `menuColumns`), the filter types, the dirty registry, report actions, the shortcut registry rules, `combo()` of the chosen chords | architect |
| `sidenav.assert.ts` | rewrite its literal table and group tests from Appendix A (1.5) | B1 |
| `favourites.assert.ts` | update section 7 and 8 deliberately (5.8) | B1 |
| `navstack.assert.ts`, `quickcreate.assert.ts` | new (5.8, 2.5, 4.5) | B1 |
| `rowfilter.assert.ts`, saved-view additions | new (6.8) | B2 |
| `reportcatalog.assert.ts` | new (7.8) | B3 |
| `visual.assert.ts` | extend, never relax: the new header row (title, slots, company) in both densities and both schemes; the rail's `+` strip does not move a row | B1 and B2 |

### 9.2 Acceptance checklist (reviewers run it; each item names its proof)
**Gates.** `npx tsc --noEmit` 0 errors. `npm run build` green. `npm run assert` all green (65 suites today; more after). `npm run perf` 15/15. `npm run shots` over **all** screens, light and dark, compact and comfortable, zero console errors and zero failed screens (`--strict`).

**Frozen things are frozen.**
- [ ] The Gateway: every row and letter as before, the one appended row (`O`), unique letters (`zoho-scaffold`); the F-key rail unchanged on every screen that had one (screenshots).
- [ ] Every existing chord still means what it meant: the voucher F-keys on menus; `Ctrl+A` Accept; `Esc`; the Esc "Quit ?" on vouchers and its Yes preselected; `Ctrl+\`; `Ctrl+K`; `Alt+G`; `Ctrl+Alt+1..9`; `/` and `Ctrl+F` on reports (and their "totals unfiltered").
- [ ] Field order of every master and voucher unchanged (the existing assertions; B1's `useDirty` is a hook call, not a field).
- [ ] No Tally wording introduced anywhere in UI copy except the existing migration row's own name (`sidenav.assert` brand section, re-run).
- [ ] No hex in any new CSS; every light rule is rooted at `html:root[data-scheme='light']`; dark unchanged where nothing new is drawn.
- [ ] Screens untouched by the wave are byte-identical in the screenshot sweep (the scaffold already proved the shared shells draw nothing extra).

**The owner's list.**
- [ ] **Left menu:** 12 groups, no group over 6 rows; a click on "Sales" opens the Sales Register (flat All level) and opens the group; the chevron alone toggles; every row opens its list; the `+` on Invoices opens a Sales voucher and Esc returns to the register; rows without a create have no `+`; the nine keycaps are right; hover, focus and the current row show the `+`.
- [ ] **`+ New` top right** on a register, Day Book, a ledger list and the Chart of Accounts, with its dropdown; none on the Balance Sheet.
- [ ] **Universal `+`** in the top bar and `Alt+Q` from the Gateway, a report and a list; dead in a voucher field; the menu respects features and rights; the `?` sheet lists it; Favourite Shortcuts shows it as taken.
- [ ] **The breadcrumb bug:** the `rail-walk` recipe; a half-typed voucher is never lost silently; Go To never repeats a screen; the status bar shows at most four segments and the tooltip the whole path.
- [ ] **Saved filters:** a filter on the Sales Register narrows its totals and is labelled Filtered; it prints and exports narrowed; it is saved in a view (personal and company); recalling the view restores it; "All" clears it; a reload shows no filter.
- [ ] **Reports Center:** reachable from the Gateway (`O`), Go To and the Reports group; 14 categories with counts that change with features and rights; search finds a report by its Zoho name; the star is the pin; "Create New Report" saves a custom report that then appears under My reports (and under Shared reports for a company one).
- [ ] **Operator rights:** a restricted role (Data Entry) sees no row, `+`, create or report it cannot open, and a company-view save is refused without Administration.
- [ ] **Wave B not blocked:** `settings` stub registered and unlinked; `Btn.id` / `movable` present and unread; the registry lists the new chords.

**Four combinations** (light and dark, compact and comfortable) for every new surface (rail `+`, header, mega menu, guard prompt, filter bar and panel, views dropdown, Reports Center, builder bar), with no truncated rail label and no horizontal page scroll at 1280 and 1024.

### 9.3 Risks and open decisions (stated, with the default taken)
1. **Hover-only `+` is invisible to a mouse user who never hovers.** Mitigated: always on the current row and on touch. A first-visit hint is the UX analysis's N18 and is left to later.
2. **`Alt+Q` is dead in forms.** By the owner's own rule (never inside a text field). The mouse button and Go To cover forms.
3. **Master rows open the alter picker, not the book.** Kept for the Tally switcher; a "Book / Alter" choice in the picker is the better answer and is a follow-up.
4. **`PINS_MAX` is 8, and the star is the pin**, so there are at most eight favourite reports. One constant if the owner wants more; two lists would be worse.
5. **The flat register level is a landing place, not the status-aware invoice list** the UX analysis wants (N2). Stated so nobody believes "All" has Due and Balance columns.
6. **Company views are a new synced field** on the company row; an older client ignores it and an alteration must carry it (B2 adds the master-resave assertion). Sync-conflict behaviour is the Company row's existing one.
7. **Crumb words are not renamed** (recipes depend on them); the tooltip carries the fuller path.
8. **Not built, by design:** scheduled and emailed reports, Advanced Analytics, a free report builder with data sources, charts, per-role sharing, an icon-only rail, responsive shell, post-save "what next" strip, role-based home. Each is named in the UX analysis's register.

---

## Appendix A. `src/ui/navTree.ts`: the Menu V2 tree (builder B1 pastes this and adapts the types)

Compiled against the real `Screen` union, `CompanyFeatures`, `VchBase` and `IconName` (a misspelt route does not build), then checked by script against the Gateway's `MENUS`, the palette, `SCREEN_AREA`, the router and the quick-create table (section A, item 2). `reg()` opens the flat All level of a register. The interfaces replace `NavItem` and `NavGroup`; `KeyWhere` is the existing type in the same file.

```ts
// src/ui/navTree.ts, Menu V2 (docs/design/zoho-ux-spec.md section 1). Compiled against the real Screen union, so a misspelt route does not build.
// KeyWhere, screenDenial, visibleTree, filterTree, activeRow and the rest of the file stay as they are (updated for the types below).
import type { Screen } from '../state/store';
import type { CompanyFeatures, VchBase } from '../types';
import type { IconName } from './icons';

export interface NavItemV2 {
  id: string;
  label: string;
  /** What the row opens: the list / register / report. */
  to: Screen;
  /** The "+": the id of a QuickCreate (ui/quickCreate.ts). Absent = no "+". */
  create?: string;
  /** Other screens that count as "being on this row". */
  also?: Screen[];
  /** The key that opens `to` (a Gateway letter or F3 / F11). NOT a voucher F-key: that is the QuickCreate's. */
  key?: string;
  keyWhere?: KeyWhere;
  feature?: keyof CompanyFeatures;
  aliases: string[];
}
export interface NavGroupV2 {
  id: string;
  label: string;
  icon: IconName;
  leaf?: boolean;
  /** What a click on the group's LABEL opens (the chevron alone expands). When it is not visible to this operator or company, the first visible row's `to`. */
  to: Screen;
  more?: { label: string; to: Screen };
  items: NavItemV2[];
}

/** The flat "All vouchers" level of a register: what the left menu and the page header open (the Gateway still opens the month matrix). */
const reg = (base: VchBase): Screen => ({ s: 'register', base, all: true });

export const NAV_TREE_V2: readonly NavGroupV2[] = [
  { id: 'dashboard', label: 'Dashboard', icon: 'grid', leaf: true, to: { s: 'dashboard' }, items: [
    { id: 'dashboard', label: 'Dashboard', to: { s: 'dashboard' }, key: 'W', keyWhere: 'gateway', aliases: ['home', 'overview', 'summary', 'kpi', 'cockpit'] },
  ] },
  { id: 'sales', label: 'Sales', icon: 'invoice', to: reg('Sales'), more: { label: 'Outstandings & ageing…', to: { s: 'menu', id: 'statementsAcc' } }, items: [
    { id: 'sales.invoices', label: 'Invoices', to: reg('Sales'), also: [{ s: 'register', base: 'Sales' }], create: 'sales.invoice', aliases: ['invoice', 'sales register', 'sales list', 'tax invoice', 'sales voucher', 'bill', 'sale', 'sales report', 'all invoices'] },
    { id: 'sales.received', label: 'Payments Received', to: reg('Receipt'), also: [{ s: 'register', base: 'Receipt' }], create: 'sales.received', aliases: ['receipt', 'receipts', 'payments received', 'collection', 'received', 'customer payment', 'invoice payment', 'receipt register'] },
    { id: 'sales.receivables', label: 'Receivables', to: { s: 'outstandings', side: 'receivables' }, feature: 'billwise', aliases: ['debtors', 'sundry debtors', 'outstanding', 'udhaar', 'who owes me', 'ageing', 'overdue', 'customer balance', 'bills receivable'] },
    { id: 'sales.creditnotes', label: 'Credit Notes', to: reg('CreditNote'), also: [{ s: 'register', base: 'CreditNote' }], create: 'sales.creditnote', aliases: ['credit note', 'sales return', 'cn', 'return', 'credit memo', 'credit note register'] },
    { id: 'sales.orders', label: 'Sales Orders', to: reg('SalesOrder'), also: [{ s: 'register', base: 'SalesOrder' }], create: 'sales.order', feature: 'orders', aliases: ['sales order', 'so', 'order', 'customer order', 'quotation', 'quote'] },
    { id: 'sales.delivery', label: 'Delivery Notes', to: reg('DeliveryNote'), also: [{ s: 'register', base: 'DeliveryNote' }], create: 'sales.delivery', feature: 'inventory', aliases: ['delivery note', 'challan', 'delivery challan', 'dispatch', 'challan out'] },
  ] },
  { id: 'purchases', label: 'Purchases', icon: 'bag', to: reg('Purchase'), more: { label: 'Outstandings & ageing…', to: { s: 'menu', id: 'statementsAcc' } }, items: [
    { id: 'purchases.bills', label: 'Bills', to: reg('Purchase'), also: [{ s: 'register', base: 'Purchase' }], create: 'purchases.bill', aliases: ['purchase', 'purchase invoice', 'purchase bill', 'bill', 'supplier invoice', 'inward', 'kharid', 'purchase register', 'bills list', 'purchase list'] },
    { id: 'purchases.paid', label: 'Payments Made', to: reg('Payment'), also: [{ s: 'register', base: 'Payment' }], create: 'purchases.paid', aliases: ['payment', 'paid', 'expense', 'expenses', 'kharcha', 'outgoing', 'vendor payment', 'bill payment', 'payment register'] },
    { id: 'purchases.payables', label: 'Payables', to: { s: 'outstandings', side: 'payables' }, feature: 'billwise', aliases: ['creditors', 'sundry creditors', 'dues', 'bills to pay', 'outstanding', 'vendor balance', 'bills payable'] },
    { id: 'purchases.debitnotes', label: 'Debit Notes', to: reg('DebitNote'), also: [{ s: 'register', base: 'DebitNote' }], create: 'purchases.debitnote', aliases: ['debit note', 'purchase return', 'dn', 'return', 'vendor credit', 'debit note register'] },
    { id: 'purchases.orders', label: 'Purchase Orders', to: reg('PurchaseOrder'), also: [{ s: 'register', base: 'PurchaseOrder' }], create: 'purchases.order', feature: 'orders', aliases: ['purchase order', 'po', 'order', 'supplier order'] },
    { id: 'purchases.receiptnotes', label: 'Receipt Notes', to: reg('ReceiptNote'), also: [{ s: 'register', base: 'ReceiptNote' }], create: 'purchases.receiptnote', feature: 'inventory', aliases: ['receipt note', 'grn', 'goods received', 'goods receipt', 'inward note'] },
  ] },
  { id: 'banking', label: 'Banking', icon: 'bank', to: { s: 'cashBankBook' }, more: { label: 'More banking…', to: { s: 'menu', id: 'banking' } }, items: [
    { id: 'banking.book', label: 'Cash/Bank Book', to: { s: 'cashBankBook' }, aliases: ['cash book', 'bank book', 'cash balance', 'bank balance', 'cash and bank', 'accounts'] },
    { id: 'banking.transfers', label: 'Transfers', to: reg('Contra'), also: [{ s: 'register', base: 'Contra' }], create: 'banking.transfer', aliases: ['contra', 'transfer', 'cash deposit', 'withdrawal', 'bank transfer', 'transfer between accounts', 'contra register'] },
    { id: 'banking.recon', label: 'Reconciliation', to: { s: 'brs' }, aliases: ['brs', 'reco', 'reconcile', 'bank reconciliation', 'bank statement match'] },
    { id: 'banking.import', label: 'Import Statement', to: { s: 'brsImport' }, aliases: ['bank statement', 'statement import', 'import bank statement', 'mt940', 'camt', 'auto match', 'upload statement'] },
    { id: 'banking.cheques', label: 'Cheques', to: { s: 'cheque' }, aliases: ['cheque printing', 'print cheque', 'cheque register', 'register', 'cheque leaf'] },
    { id: 'banking.pdc', label: 'Post-Dated Cheques', to: { s: 'pdcRegister' }, aliases: ['pdc', 'postdated', 'post dated'] },
  ] },
  { id: 'accounting', label: 'Accounting', icon: 'register', to: { s: 'daybook' }, more: { label: 'Books & registers…', to: { s: 'menu', id: 'accountBooks' } }, items: [
    { id: 'accounting.daybook', label: 'Day Book', to: { s: 'daybook' }, key: 'K', keyWhere: 'gateway', aliases: ['all vouchers', 'journal register', 'daily transactions', 'transactions'] },
    { id: 'accounting.journals', label: 'Journals', to: reg('Journal'), also: [{ s: 'register', base: 'Journal' }], create: 'accounting.journal', aliases: ['journal', 'jv', 'adjustment', 'adjustment entry', 'manual journal', 'journal register'] },
    { id: 'accounting.coa', label: 'Chart of Accounts', to: { s: 'coa' }, create: 'general.ledger', key: 'H', keyWhere: 'gateway', aliases: ['coa', 'account tree', 'accounts list'] },
    { id: 'accounting.ledgers', label: 'Ledgers', to: { s: 'pickMaster', kind: 'ledger' }, create: 'general.ledger', also: [{ s: 'ledger' }], aliases: ['customers', 'vendors', 'suppliers', 'parties', 'accounts', 'party master', 'ledger master'] },
    { id: 'accounting.groups', label: 'Groups', to: { s: 'pickMaster', kind: 'group' }, create: 'accounting.group', also: [{ s: 'group' }], aliases: ['account groups', 'ledger groups', 'group master'] },
    { id: 'accounting.cost', label: 'Cost Centres', to: { s: 'pickMaster', kind: 'costCentre' }, create: 'accounting.costcentre', also: [{ s: 'costCentre' }], feature: 'costCentres', aliases: ['cost center', 'project', 'branch', 'department'] },
  ] },
  { id: 'inventory', label: 'Inventory', icon: 'box', to: { s: 'stockSummary' }, more: { label: 'Inventory reports…', to: { s: 'menu', id: 'inventoryBooks' } }, items: [
    { id: 'inventory.summary', label: 'Stock Summary', to: { s: 'stockSummary' }, key: 'S', keyWhere: 'gateway', feature: 'inventory', aliases: ['stock', 'closing stock', 'stock on hand', 'inventory summary'] },
    { id: 'inventory.items', label: 'Items', to: { s: 'pickMaster', kind: 'stockItem' }, create: 'inventory.item', also: [{ s: 'stockItem' }], feature: 'inventory', aliases: ['stock items', 'products', 'item master', 'sku', 'new item'] },
    { id: 'inventory.journal', label: 'Stock Journal', to: reg('StockJournal'), also: [{ s: 'register', base: 'StockJournal' }], create: 'inventory.stockjournal', feature: 'inventory', aliases: ['stock transfer', 'consumption', 'transfer between godowns', 'stock journal register'] },
    { id: 'inventory.physical', label: 'Physical Stock', to: reg('PhysicalStock'), also: [{ s: 'register', base: 'PhysicalStock' }], create: 'inventory.physical', feature: 'inventory', aliases: ['stock count', 'physical verification', 'stock taking', 'stock audit', 'inventory adjustment', 'stock adjustment'] },
    { id: 'inventory.godowns', label: 'Godowns', to: { s: 'pickMaster', kind: 'godown' }, create: 'inventory.godown', also: [{ s: 'godown' }], feature: 'inventory', aliases: ['warehouse', 'location', 'store', 'godown master'] },
    { id: 'inventory.mfg', label: 'Manufacturing', to: { s: 'manufacture' }, feature: 'inventory', aliases: ['manufacturing journal', 'production', 'bom', 'bill of materials', 'assembly'] },
  ] },
  { id: 'reports', label: 'Reports', icon: 'trend', to: { s: 'reportsCenter' }, more: { label: 'Gateway report menus…', to: { s: 'menu', id: 'display' } }, items: [
    { id: 'reports.bs', label: 'Balance Sheet', to: { s: 'balanceSheet' }, key: 'B', keyWhere: 'gateway', aliases: ['bs', 'financial position', 'assets liabilities'] },
    { id: 'reports.pl', label: 'Profit & Loss', to: { s: 'profitLoss' }, key: 'P', keyWhere: 'gateway', aliases: ['pl', 'p&l', 'income statement', 'profit', 'loss'] },
    { id: 'reports.tb', label: 'Trial Balance', to: { s: 'trialBalance' }, aliases: ['tb', 'trial'] },
    { id: 'reports.cashflow', label: 'Cash Flow', to: { s: 'cashFlow' }, aliases: ['cash movement', 'cashflow'] },
    { id: 'reports.ratio', label: 'Ratio Analysis', to: { s: 'ratioAnalysis' }, key: 'R', keyWhere: 'gateway', aliases: ['ratios', 'key indicators', 'current ratio', 'margin'] },
    { id: 'reports.mis', label: 'Top-N Analysis (MIS)', to: { s: 'topMis' }, aliases: ['mis', 'top customers', 'top items', 'best sellers', 'top 10'] },
  ] },
  { id: 'taxes', label: 'Taxes & Compliance', icon: 'percent', to: { s: 'gstr3b' }, more: { label: 'All statutory reports…', to: { s: 'menu', id: 'statutory' } }, items: [
    { id: 'taxes.gstr3b', label: 'GSTR-3B', to: { s: 'gstr3b' }, feature: 'gst', aliases: ['3b', 'monthly return', 'gst payment', 'summary return', 'gst overview'] },
    { id: 'taxes.gstr1', label: 'GSTR-1', to: { s: 'gstr1' }, feature: 'gst', aliases: ['r1', 'outward return', 'sales return gst', 'gst return'] },
    { id: 'taxes.gstr2b', label: 'GSTR-2B Reconciliation', to: { s: 'gstr2b' }, feature: 'gst', aliases: ['2b', 'itc match', 'input credit', 'purchase match'] },
    { id: 'taxes.einvoice', label: 'e-Invoice / e-Way Bill', to: { s: 'einvoice' }, feature: 'gst', aliases: ['irn', 'eway', 'ewb', 'einvoice', 'qr', 'portal'] },
    { id: 'taxes.tds', label: 'TDS Outstanding', to: { s: 'tdsReport' }, feature: 'tds', aliases: ['tds payable', 'tds due', 'deducted'] },
    { id: 'taxes.tdsreturns', label: 'TDS Challans & Returns', to: { s: 'tdsReturns' }, feature: 'tds', aliases: ['challan', '26q', 'tds return', 'tds deposit'] },
  ] },
  { id: 'payroll', label: 'Payroll & HR', icon: 'people', to: { s: 'pickMaster', kind: 'employee' }, more: { label: 'Payroll reports…', to: { s: 'menu', id: 'payrollRep' } }, items: [
    { id: 'payroll.employees', label: 'Employees', to: { s: 'pickMaster', kind: 'employee' }, create: 'payroll.employee', also: [{ s: 'employee' }], feature: 'payroll', aliases: ['staff', 'people', 'new employee', 'hr'] },
    { id: 'payroll.attendance', label: 'Attendance', to: { s: 'attendanceRegister' }, create: 'payroll.attendance', also: [{ s: 'attendance' }], feature: 'payroll', aliases: ['attendance register', 'attendance voucher', 'present absent', 'overtime'] },
    { id: 'payroll.payroll', label: 'Payroll', to: { s: 'paySheet' }, create: 'payroll.voucher', also: [{ s: 'payrollVch' }], feature: 'payroll', aliases: ['pay sheet', 'salary', 'run payroll', 'process salary', 'salary sheet', 'salary register', 'payroll voucher'] },
    { id: 'payroll.salary', label: 'Salary Disbursement', to: { s: 'salaryDisbursement' }, feature: 'payroll', aliases: ['salary payment', 'pay salary', 'bank transfer salary'] },
    { id: 'payroll.leave', label: 'Leave', to: { s: 'leaveRegister' }, create: 'payroll.leave', also: [{ s: 'leaveApply' }], feature: 'payroll', aliases: ['apply leave', 'record leave', 'time off', 'absence', 'leave register'] },
    { id: 'payroll.statutory', label: 'PF / ESI / PT', to: { s: 'payrollStatutory' }, feature: 'payroll', aliases: ['provident fund', 'esi', 'professional tax', 'epf', 'statutory'] },
  ] },
  { id: 'crm', label: 'CRM', icon: 'funnel', to: { s: 'leads' }, items: [
    { id: 'crm.leads', label: 'Leads', to: { s: 'leads' }, create: 'crm.lead', also: [{ s: 'lead' }], aliases: ['prospects', 'enquiries', 'inquiries', 'new lead', 'create lead', 'add lead'] },
    { id: 'crm.followups', label: 'Follow-ups Due', to: { s: 'followUps' }, aliases: ['tasks', 'calls due', 'reminders', 'to do'] },
    { id: 'crm.pipeline', label: 'Pipeline Summary', to: { s: 'pipeline' }, aliases: ['funnel', 'stages', 'deals'] },
  ] },
  { id: 'data', label: 'Data & Utilities', icon: 'plug', to: { s: 'backupRestore' }, more: { label: 'More utilities…', to: { s: 'menu', id: 'companyInfo' } }, items: [
    { id: 'data.backup', label: 'Backup / Restore', to: { s: 'backupRestore' }, aliases: ['backup', 'restore', 'export data', 'auto backup', 'save a copy'] },
    { id: 'data.bulk', label: 'Bulk Entry', to: { s: 'bulkEntry' }, aliases: ['paste from excel', 'import vouchers', 'excel entry', 'many vouchers'] },
    { id: 'data.import', label: 'Import Data', to: { s: 'importData' }, aliases: ['migrate', 'import from tally', 'csv', 'xml', 'upload', 'switch from tally'] },
    { id: 'data.export', label: 'Export Data', to: { s: 'exportXml' }, aliases: ['export for tally', 'xml', 'send to ca', 'download'] },
    { id: 'data.check', label: 'Data Check', to: { s: 'dataCheck' }, aliases: ['integrity', 'repair', 'verify data', 'unbalanced', 'orphan'] },
    { id: 'data.api', label: 'API Keys', to: { s: 'apiKeys' }, aliases: ['token', 'integration', 'developer', 'rest api', 'webhooks'] },
  ] },
  { id: 'settings', label: 'Settings', icon: 'gear', to: { s: 'companyAlter' }, more: { label: 'More settings…', to: { s: 'menu', id: 'companyInfo' } }, items: [
    { id: 'settings.company', label: 'Company Details', to: { s: 'companyAlter' }, aliases: ['alter company', 'company info', 'gstin', 'address', 'profile', 'organisation'] },
    { id: 'settings.switch', label: 'Switch Company', to: { s: 'companySelect' }, key: 'F3', keyWhere: 'anywhere', aliases: ['select company', 'change company', 'another company', 'organisation'] },
    { id: 'settings.features', label: 'Features', to: { s: 'features' }, key: 'F11', keyWhere: 'anywhere', aliases: ['f11', 'modules', 'turn on', 'turn off', 'enable'] },
    { id: 'settings.security', label: 'Users & Security', to: { s: 'security' }, aliases: ['users', 'roles', 'permissions', 'pin', 'access'] },
    { id: 'settings.print', label: 'Print Configuration', to: { s: 'printConfig' }, aliases: ['invoice format', 'invoice layout', 'customise invoice', 'print setup', 'logo', 'template'] },
    { id: 'settings.vtypes', label: 'Voucher Types', to: { s: 'pickMaster', kind: 'voucherType' }, create: 'settings.vtype', also: [{ s: 'voucherType' }], aliases: ['numbering', 'invoice series', 'voucher series', 'new voucher type'] },
  ] },
];
```

## Appendix B. `src/ui/quickCreate.ts`: the two tables (builder B1 replaces the empty ones)

```ts
// src/ui/quickCreate.ts: replace the two empty tables with these (docs/design/zoho-ux-spec.md section 2). Compiled against the real types.

export const QUICK_CREATES: readonly QuickCreate[] = [
  // ── Sales
  { id: 'sales.invoice', label: 'Invoice', group: 'Sales', to: { s: 'voucher', base: 'Sales' }, key: 'F8', keyWhere: 'menus', aliases: ['sales invoice', 'tax invoice', 'sale', 'sales voucher', 'bill', 'new sale', 'sales'] },
  { id: 'sales.received', label: 'Payment Received', group: 'Sales', to: { s: 'voucher', base: 'Receipt' }, key: 'F6', keyWhere: 'menus', aliases: ['receipt', 'payments received', 'collection', 'received', 'customer payment', 'invoice payment'] },
  { id: 'sales.creditnote', label: 'Credit Note', group: 'Sales', to: { s: 'voucher', base: 'CreditNote' }, key: 'Alt+F6', keyWhere: 'menus', aliases: ['sales return', 'cn', 'return', 'credit memo'] },
  { id: 'sales.order', label: 'Sales Order', group: 'Sales', to: { s: 'voucher', base: 'SalesOrder' }, feature: 'orders', aliases: ['so', 'order', 'customer order', 'quotation', 'quote'] },
  { id: 'sales.delivery', label: 'Delivery Note', group: 'Sales', to: { s: 'voucher', base: 'DeliveryNote' }, feature: 'inventory', aliases: ['challan', 'delivery challan', 'dispatch', 'challan out'] },
  // ── Purchases
  { id: 'purchases.bill', label: 'Bill', group: 'Purchases', to: { s: 'voucher', base: 'Purchase' }, key: 'F9', keyWhere: 'menus', aliases: ['purchase', 'purchase invoice', 'purchase bill', 'supplier invoice', 'inward', 'kharid', 'new purchase'] },
  { id: 'purchases.paid', label: 'Payment Made', group: 'Purchases', to: { s: 'voucher', base: 'Payment' }, key: 'F5', keyWhere: 'menus', aliases: ['payment', 'paid', 'expense', 'kharcha', 'outgoing', 'vendor payment', 'bill payment'] },
  { id: 'purchases.debitnote', label: 'Debit Note', group: 'Purchases', to: { s: 'voucher', base: 'DebitNote' }, key: 'Alt+F5', keyWhere: 'menus', aliases: ['purchase return', 'dn', 'return', 'vendor credit'] },
  { id: 'purchases.order', label: 'Purchase Order', group: 'Purchases', to: { s: 'voucher', base: 'PurchaseOrder' }, feature: 'orders', aliases: ['po', 'order', 'supplier order'] },
  { id: 'purchases.receiptnote', label: 'Receipt Note', group: 'Purchases', to: { s: 'voucher', base: 'ReceiptNote' }, feature: 'inventory', aliases: ['grn', 'goods received', 'goods receipt', 'inward note'] },
  // ── Banking
  { id: 'banking.transfer', label: 'Transfer (Contra)', group: 'Banking', to: { s: 'voucher', base: 'Contra' }, key: 'F4', keyWhere: 'menus', aliases: ['contra', 'transfer', 'cash deposit', 'withdrawal', 'bank transfer', 'transfer between accounts', 'cash to bank'] },
  // ── Accounting
  { id: 'accounting.journal', label: 'Journal Entry', group: 'Accounting', to: { s: 'voucher', base: 'Journal' }, key: 'F7', keyWhere: 'menus', aliases: ['journal', 'jv', 'adjustment', 'adjustment entry', 'manual journal'] },
  { id: 'accounting.group', label: 'Group', group: 'Accounting', to: { s: 'group' }, aliases: ['account group', 'ledger group', 'new group', 'category'] },
  { id: 'accounting.costcentre', label: 'Cost Centre', group: 'Accounting', to: { s: 'costCentre' }, feature: 'costCentres', aliases: ['cost center', 'project', 'branch', 'department', 'new cost centre'] },
  // ── Inventory
  { id: 'inventory.item', label: 'Item', group: 'Inventory', to: { s: 'stockItem' }, feature: 'inventory', aliases: ['stock item', 'product', 'sku', 'item master', 'new item'] },
  { id: 'inventory.stockjournal', label: 'Stock Journal', group: 'Inventory', to: { s: 'voucher', base: 'StockJournal' }, key: 'Alt+F7', keyWhere: 'menus', feature: 'inventory', aliases: ['stock transfer', 'consumption', 'transfer between godowns'] },
  { id: 'inventory.physical', label: 'Physical Stock', group: 'Inventory', to: { s: 'voucher', base: 'PhysicalStock' }, key: 'Ctrl+F7', keyWhere: 'menus', feature: 'inventory', aliases: ['stock count', 'physical verification', 'stock taking', 'stock adjustment', 'inventory adjustment'] },
  { id: 'inventory.mfg', label: 'Manufacturing Journal', group: 'Inventory', to: { s: 'manufacture' }, feature: 'inventory', aliases: ['production', 'bom', 'bill of materials', 'assembly', 'manufacturing'] },
  { id: 'inventory.godown', label: 'Godown', group: 'Inventory', to: { s: 'godown' }, feature: 'inventory', aliases: ['warehouse', 'location', 'store', 'new godown'] },
  { id: 'inventory.unit', label: 'Unit', group: 'Inventory', to: { s: 'unit' }, feature: 'inventory', aliases: ['uom', 'unit of measure', 'nos', 'kg', 'new unit'] },
  // ── General
  { id: 'general.ledger', label: 'Ledger', group: 'General', to: { s: 'ledger' }, aliases: ['customer', 'vendor', 'supplier', 'party', 'account', 'new ledger', 'party master'] },
  { id: 'settings.vtype', label: 'Voucher Type', group: 'General', to: { s: 'voucherType' }, aliases: ['numbering', 'invoice series', 'voucher series', 'document series', 'new voucher type'] },
  { id: 'general.report', label: 'Custom Report', group: 'General', to: { s: 'customReport' }, aliases: ['report', 'new report', 'create report', 'report builder', 'saved report'] },
  // ── Payroll
  { id: 'payroll.employee', label: 'Employee', group: 'Payroll', to: { s: 'employee' }, feature: 'payroll', aliases: ['staff', 'people', 'new employee', 'hr'] },
  { id: 'payroll.attendance', label: 'Attendance', group: 'Payroll', to: { s: 'attendance' }, feature: 'payroll', aliases: ['attendance voucher', 'present absent', 'overtime'] },
  { id: 'payroll.voucher', label: 'Payroll Voucher', group: 'Payroll', to: { s: 'payrollVch' }, feature: 'payroll', aliases: ['salary', 'run payroll', 'process salary'] },
  { id: 'payroll.leave', label: 'Leave', group: 'Payroll', to: { s: 'leaveApply' }, feature: 'payroll', aliases: ['apply leave', 'record leave', 'time off', 'absence'] },
  // ── CRM
  { id: 'crm.lead', label: 'Lead', group: 'CRM', to: { s: 'lead' }, aliases: ['prospect', 'enquiry', 'inquiry', 'new lead', 'add lead'] },
];

export const QC_PAGES: Readonly<Record<string, { primary: string; related: readonly string[] }>> = {
  // Sales documents
  'register:base=Sales': { primary: 'sales.invoice', related: ['sales.received', 'sales.creditnote', 'sales.order', 'sales.delivery'] },
  'register:base=Receipt': { primary: 'sales.received', related: ['sales.invoice', 'sales.creditnote'] },
  'register:base=CreditNote': { primary: 'sales.creditnote', related: ['sales.invoice', 'sales.received'] },
  'register:base=SalesOrder': { primary: 'sales.order', related: ['sales.invoice', 'sales.delivery'] },
  'register:base=DeliveryNote': { primary: 'sales.delivery', related: ['sales.invoice', 'sales.order'] },
  'outstandings:side=receivables': { primary: 'sales.invoice', related: ['sales.received', 'sales.creditnote'] },
  // Purchase documents
  'register:base=Purchase': { primary: 'purchases.bill', related: ['purchases.paid', 'purchases.debitnote', 'purchases.order', 'purchases.receiptnote'] },
  'register:base=Payment': { primary: 'purchases.paid', related: ['purchases.bill', 'purchases.debitnote'] },
  'register:base=DebitNote': { primary: 'purchases.debitnote', related: ['purchases.bill', 'purchases.paid'] },
  'register:base=PurchaseOrder': { primary: 'purchases.order', related: ['purchases.bill', 'purchases.receiptnote'] },
  'register:base=ReceiptNote': { primary: 'purchases.receiptnote', related: ['purchases.bill', 'purchases.order'] },
  'outstandings:side=payables': { primary: 'purchases.bill', related: ['purchases.paid', 'purchases.debitnote'] },
  // Banking and accounting
  'register:base=Contra': { primary: 'banking.transfer', related: ['sales.received', 'purchases.paid', 'accounting.journal'] },
  cashBankBook: { primary: 'banking.transfer', related: ['sales.received', 'purchases.paid'] },
  'register:base=Journal': { primary: 'accounting.journal', related: ['banking.transfer'] },
  daybook: { primary: 'sales.invoice', related: ['purchases.bill', 'sales.received', 'purchases.paid', 'accounting.journal', 'banking.transfer'] },
  // Masters (the list pages)
  'pickMaster:kind=ledger': { primary: 'general.ledger', related: ['accounting.group'] },
  coa: { primary: 'general.ledger', related: ['accounting.group'] },
  'pickMaster:kind=group': { primary: 'accounting.group', related: ['general.ledger'] },
  'pickMaster:kind=costCentre': { primary: 'accounting.costcentre', related: [] },
  'pickMaster:kind=voucherType': { primary: 'settings.vtype', related: [] },
  // Inventory
  stockSummary: { primary: 'inventory.item', related: ['inventory.stockjournal', 'inventory.physical', 'inventory.godown'] },
  'pickMaster:kind=stockItem': { primary: 'inventory.item', related: ['inventory.godown', 'inventory.unit'] },
  'pickMaster:kind=godown': { primary: 'inventory.godown', related: ['inventory.item'] },
  'pickMaster:kind=unit': { primary: 'inventory.unit', related: ['inventory.item'] },
  'register:base=StockJournal': { primary: 'inventory.stockjournal', related: ['inventory.physical', 'inventory.item'] },
  'register:base=PhysicalStock': { primary: 'inventory.physical', related: ['inventory.stockjournal'] },
  // People
  'pickMaster:kind=employee': { primary: 'payroll.employee', related: ['payroll.attendance', 'payroll.voucher'] },
  attendanceRegister: { primary: 'payroll.attendance', related: ['payroll.employee'] },
  paySheet: { primary: 'payroll.voucher', related: ['payroll.attendance'] },
  leaveRegister: { primary: 'payroll.leave', related: ['payroll.employee'] },
  leads: { primary: 'crm.lead', related: [] },
  // The Reports Center itself: "Create New Report"
  reportsCenter: { primary: 'general.report', related: [] },
};
```

## Appendix C. `src/ui/reportCatalog.ts`: the Reports Center catalogue (builder B3)

The whole file, compiled. The exclusion list is the other half of the drift guard: **every report-kind destination of the Gateway `MENUS` and of the palette must be in the catalogue or in this table**, and the table's reasons are the scope statement of 7.3.

```ts
// src/ui/reportCatalog.ts (new, builder B3; docs/design/zoho-ux-spec.md section 7). Compiled against the real Screen union.
import type { Screen } from '../state/store';
import type { CompanyFeatures, VchBase } from '../types';

export type ReportCategoryId =
  | 'business' | 'sales' | 'receivables' | 'payables' | 'inventory' | 'valuation' | 'banking'
  | 'gst' | 'tds' | 'payroll' | 'accountant' | 'costing' | 'activity' | 'crm';

export interface ReportCategory { id: ReportCategoryId; label: string; blurb: string }

export const REPORT_CATEGORIES: readonly ReportCategory[] = [
  { id: 'business', label: 'Business Overview', blurb: 'Profit, position, cash and ratios' },
  { id: 'sales', label: 'Sales', blurb: 'Invoices, returns and orders to customers' },
  { id: 'receivables', label: 'Receivables', blurb: 'Who owes you, how late, and what you were paid' },
  { id: 'payables', label: 'Payables & Purchases', blurb: 'Bills, payments and what you owe suppliers' },
  { id: 'inventory', label: 'Inventory', blurb: 'Stock on hand, movement and reorder' },
  { id: 'valuation', label: 'Inventory Valuation', blurb: 'Stock at cost under each method' },
  { id: 'banking', label: 'Banking', blurb: 'Cash, bank, cheques and reconciliation' },
  { id: 'gst', label: 'GST', blurb: 'Returns, reconciliation and e-invoicing' },
  { id: 'tds', label: 'TDS & TCS', blurb: 'Deduction, collection, challans and certificates' },
  { id: 'payroll', label: 'Payroll & HR', blurb: 'Pay, attendance, leave and statutory' },
  { id: 'accountant', label: 'Accountant', blurb: 'Books, ledgers and trial balance' },
  { id: 'costing', label: 'Cost Centres & Budgets', blurb: 'Projects, departments and budget variance' },
  { id: 'activity', label: 'Activity & Audit', blurb: 'Edit log, cancelled entries and checks' },
  { id: 'crm', label: 'CRM', blurb: 'Leads, follow-ups and the pipeline' },
];

export interface ReportEntry {
  /** Stable, unique ('sales.register'). The star, the visit log and the tests use it. */
  id: string;
  /** What the table shows. Tally's own word where there is one, with the other vocabulary in `aliases`. */
  label: string;
  category: ReportCategoryId;
  /** EXACTLY the Screen the Gateway (or Go To) pushes for the same report. */
  to: Screen;
  /** Every company feature that must be on, exactly as the Gateway gates it (a nested submenu's feature counts). */
  needs?: readonly (keyof CompanyFeatures)[];
  /** The Gateway letter that opens it, printed as a dim keycap where it fires ("from the Gateway"). */
  key?: string;
  /** Invisible search vocabulary, lower case. */
  aliases: readonly string[];
}

const reg = (id: string, label: string, category: ReportCategoryId, base: VchBase, aliases: string[], needs?: ReportEntry['needs']): ReportEntry =>
  ({ id, label, category, to: { s: 'register', base }, aliases, ...(needs ? { needs } : {}) });

export const REPORT_CATALOG: readonly ReportEntry[] = [
  // ── Business Overview
  { id: 'business.bs', label: 'Balance Sheet', category: 'business', to: { s: 'balanceSheet' }, key: 'B', aliases: ['bs', 'financial position', 'assets liabilities'] },
  { id: 'business.pl', label: 'Profit & Loss A/c', category: 'business', to: { s: 'profitLoss' }, key: 'P', aliases: ['pl', 'p&l', 'profit and loss', 'income statement', 'profit', 'loss'] },
  { id: 'business.cashflow', label: 'Cash Flow', category: 'business', to: { s: 'cashFlow' }, aliases: ['cash movement', 'cashflow', 'cash flow statement'] },
  { id: 'business.fundsflow', label: 'Funds Flow', category: 'business', to: { s: 'fundsFlow' }, aliases: ['sources and uses', 'fund flow'] },
  { id: 'business.ratio', label: 'Ratio Analysis', category: 'business', to: { s: 'ratioAnalysis' }, key: 'R', aliases: ['ratios', 'key indicators', 'current ratio', 'margin', 'business performance'] },
  { id: 'business.mis', label: 'Top-N Analysis (MIS)', category: 'business', to: { s: 'topMis' }, aliases: ['mis', 'top customers', 'top items', 'best sellers', 'top 10'] },
  { id: 'business.profitability', label: 'Profitability Analysis', category: 'business', to: { s: 'profitability' }, aliases: ['margin', 'gross profit', 'item profit', 'profit by item'] },
  // ── Sales
  reg('sales.register', 'Sales Register', 'sales', 'Sales', ['invoices', 'sales list', 'sales report', 'sales by customer', 'sales summary']),
  reg('sales.creditnotes', 'Credit Note Register', 'sales', 'CreditNote', ['credit notes', 'sales return', 'credit memo']),
  reg('sales.orders', 'Sales Order Register', 'sales', 'SalesOrder', ['sales orders', 'customer orders', 'quotes'], ['orders']),
  { id: 'sales.pending', label: 'Sales Orders Outstanding', category: 'sales', to: { s: 'pendingOrders', base: 'SalesOrder' }, needs: ['inventory', 'orders'], aliases: ['pending sales orders', 'open orders', 'order fulfillment'] },
  reg('sales.delivery', 'Delivery Note Register', 'sales', 'DeliveryNote', ['delivery notes', 'challan', 'dispatch'], ['inventory']),
  // ── Receivables
  { id: 'receivables.bills', label: 'Receivables (Bills Outstanding)', category: 'receivables', to: { s: 'outstandings', side: 'receivables' }, needs: ['billwise'], aliases: ['receivables', 'debtors', 'sundry debtors', 'outstanding', 'udhaar', 'who owes me', 'customer balance', 'invoice details'] },
  { id: 'receivables.ageing', label: 'Ageing Analysis (buckets)', category: 'receivables', to: { s: 'ageing' }, needs: ['billwise'], aliases: ['ageing', 'aging', 'overdue', 'buckets', 'aging summary', 'aging details'] },
  { id: 'receivables.reminders', label: 'Reminder Letters & Confirmation', category: 'receivables', to: { s: 'reminders' }, needs: ['billwise'], aliases: ['dunning', 'payment reminder', 'follow up letter', 'balance confirmation'] },
  { id: 'receivables.interest', label: 'Interest Calculation', category: 'receivables', to: { s: 'interest' }, aliases: ['interest on overdue', 'interest calc', 'late payment interest'] },
  reg('receivables.received', 'Receipt Register', 'receivables', 'Receipt', ['payments received', 'receipts', 'collections']),
  // ── Payables & Purchases
  reg('payables.register', 'Purchase Register', 'payables', 'Purchase', ['bills list', 'purchase list', 'purchase report', 'purchases by vendor', 'bill details']),
  reg('payables.debitnotes', 'Debit Note Register', 'payables', 'DebitNote', ['debit notes', 'purchase return', 'vendor credit']),
  reg('payables.orders', 'Purchase Order Register', 'payables', 'PurchaseOrder', ['purchase orders', 'supplier orders'], ['orders']),
  { id: 'payables.pending', label: 'Purchase Orders Outstanding', category: 'payables', to: { s: 'pendingOrders', base: 'PurchaseOrder' }, needs: ['inventory', 'orders'], aliases: ['pending purchase orders', 'open purchase orders'] },
  reg('payables.receiptnotes', 'Receipt Note Register', 'payables', 'ReceiptNote', ['receipt notes', 'grn', 'goods received'], ['inventory']),
  { id: 'payables.bills', label: 'Payables (Bills Outstanding)', category: 'payables', to: { s: 'outstandings', side: 'payables' }, needs: ['billwise'], aliases: ['payables', 'creditors', 'sundry creditors', 'dues', 'bills to pay', 'vendor balance', 'bill payable'] },
  reg('payables.payments', 'Payment Register', 'payables', 'Payment', ['payments made', 'payments', 'expenses']),
  // ── Inventory
  { id: 'inventory.summary', label: 'Stock Summary', category: 'inventory', to: { s: 'stockSummary' }, needs: ['inventory'], key: 'S', aliases: ['inventory summary', 'closing stock', 'stock on hand', 'inventory'] },
  { id: 'inventory.godown', label: 'Godown Summary', category: 'inventory', to: { s: 'godownSummary' }, needs: ['inventory'], aliases: ['warehouse', 'location stock', 'godown stock'] },
  { id: 'inventory.item', label: 'Stock Item (book)', category: 'inventory', to: { s: 'pickMaster', kind: 'stockItem', goal: 'book' }, needs: ['inventory'], aliases: ['item vouchers', 'item ledger', 'item movement', 'stock item book'] },
  { id: 'inventory.movement', label: 'Movement Analysis', category: 'inventory', to: { s: 'movementAnalysis' }, needs: ['inventory'], aliases: ['stock movement', 'inward outward'] },
  { id: 'inventory.query', label: 'Stock Query', category: 'inventory', to: { s: 'stockQuery' }, needs: ['inventory'], aliases: ['stock lookup', 'item query'] },
  { id: 'inventory.reorder', label: 'Reorder Status', category: 'inventory', to: { s: 'reorderStatus' }, needs: ['inventory'], aliases: ['low stock', 'reorder level', 'what to buy', 'shortage'] },
  { id: 'inventory.ageing', label: 'Stock Ageing Analysis', category: 'inventory', to: { s: 'stockAgeing' }, needs: ['inventory'], aliases: ['slow moving', 'dead stock', 'old stock', 'inventory aging summary'] },
  reg('inventory.journals', 'Stock Journal Register', 'inventory', 'StockJournal', ['stock transfer', 'consumption'], ['inventory']),
  reg('inventory.physical', 'Physical Stock Register', 'inventory', 'PhysicalStock', ['stock count', 'physical verification', 'inventory adjustment summary'], ['inventory']),
  // ── Inventory Valuation
  { id: 'valuation.methods', label: 'Stock Valuation Methods', category: 'valuation', to: { s: 'stockValuation' }, needs: ['inventory'], aliases: ['inventory valuation summary', 'costing method', 'avg cost', 'fifo', 'lifo', 'market value'] },
  { id: 'valuation.godown', label: 'Godown-wise Stock (valuation)', category: 'valuation', to: { s: 'godownStock' }, needs: ['inventory'], aliases: ['stock value by godown', 'warehouse valuation'] },
  // ── Banking
  { id: 'banking.book', label: 'Cash/Bank Book(s)', category: 'banking', to: { s: 'cashBankBook' }, aliases: ['cash book', 'bank book', 'cash balance', 'bank balance'] },
  { id: 'banking.recon', label: 'Bank Reconciliation', category: 'banking', to: { s: 'brs' }, aliases: ['brs', 'reco', 'reconcile', 'reconciliation status'] },
  { id: 'banking.cheque', label: 'Cheque Printing & Register', category: 'banking', to: { s: 'cheque' }, aliases: ['print cheque', 'cheque register', 'cheque leaf'] },
  { id: 'banking.books', label: 'Cheque Books', category: 'banking', to: { s: 'chequeBooks' }, aliases: ['cheque leaves', 'cheque series'] },
  { id: 'banking.pdc', label: 'Post-Dated Cheques', category: 'banking', to: { s: 'pdcRegister' }, aliases: ['pdc', 'postdated', 'post dated'] },
  { id: 'banking.advice', label: 'Payment Advice', category: 'banking', to: { s: 'paymentAdvice' }, aliases: ['remittance', 'remittance advice', 'payment slip'] },
  { id: 'banking.slip', label: 'Deposit Slip', category: 'banking', to: { s: 'depositSlip' }, aliases: ['pay in slip', 'bank deposit', 'paying slip'] },
  reg('banking.contra', 'Contra Register', 'banking', 'Contra', ['transfers', 'cash deposit', 'withdrawal', 'bank transfer']),
  // ── GST
  { id: 'gst.gstr1', label: 'GSTR-1', category: 'gst', to: { s: 'gstr1' }, needs: ['gst'], aliases: ['r1', 'outward return', 'summary of outward supplies', 'gst return'] },
  { id: 'gst.gstr3b', label: 'GSTR-3B', category: 'gst', to: { s: 'gstr3b' }, needs: ['gst'], aliases: ['3b', 'monthly return', 'summary return', 'gst overview'] },
  { id: 'gst.gstr2b', label: 'GSTR-2B Reconciliation', category: 'gst', to: { s: 'gstr2b' }, needs: ['gst'], aliases: ['2b', 'itc match', 'input credit', 'purchase match'] },
  { id: 'gst.gstr9', label: 'GSTR-9 Annual Return', category: 'gst', to: { s: 'gstr9' }, needs: ['gst'], aliases: ['annual return', 'r9', 'yearly gst'] },
  { id: 'gst.hsn', label: 'HSN Summary', category: 'gst', to: { s: 'hsnSummary' }, needs: ['gst'], aliases: ['hsn', 'sac', 'hsn wise'] },
  { id: 'gst.einvoice', label: 'e-Invoice / e-Way Bill', category: 'gst', to: { s: 'einvoice' }, needs: ['gst'], aliases: ['irn', 'eway', 'ewb', 'einvoice', 'qr', 'portal'] },
  { id: 'gst.validation', label: 'GST Validation / Exceptions', category: 'gst', to: { s: 'gstValidation' }, needs: ['gst'], aliases: ['gst exceptions', 'missing gstin', 'pre filing check'] },
  { id: 'gst.recon', label: 'GSTR-3B vs 1 vs Books Recon', category: 'gst', to: { s: 'gstRecon' }, needs: ['gst'], aliases: ['3b vs 1', 'books vs return', 'gst difference'] },
  { id: 'gst.billofsupply', label: 'Bill of Supply', category: 'gst', to: { s: 'billOfSupply' }, needs: ['gst'], aliases: ['composition', 'exempt supplies', 'tax free invoice'] },
  { id: 'gst.itc', label: 'Ineligible ITC (17(5))', category: 'gst', to: { s: 'itcReport' }, needs: ['gst'], aliases: ['blocked credit', 'itc blocked', '17(5)'] },
  { id: 'gst.advances', label: 'GSTR-1 Advances (11A/11B)', category: 'gst', to: { s: 'gstAdvances' }, needs: ['gst'], aliases: ['advance receipts', '11a', '11b', 'tax on advances'] },
  { id: 'gst.docs', label: 'Documents Issued (Table 13)', category: 'gst', to: { s: 'gstDocsIssued' }, needs: ['gst'], aliases: ['table 13', 'invoice series issued', 'documents issued'] },
  { id: 'gst.rcm', label: 'Reverse Charge Register', category: 'gst', to: { s: 'rcmRegister' }, needs: ['gst'], aliases: ['rcm', 'reverse charge'] },
  { id: 'gst.reversal', label: 'ITC Reversal (Rule 37)', category: 'gst', to: { s: 'itcReversal' }, needs: ['gst'], aliases: ['rule 37', '180 days', 'unpaid supplier credit'] },
  // ── TDS & TCS
  { id: 'tds.outstanding', label: 'TDS Outstanding', category: 'tds', to: { s: 'tdsReport' }, needs: ['tds'], aliases: ['tds payable', 'tds due', 'deducted', 'tds payables summary'] },
  { id: 'tds.returns', label: 'TDS Challans & Returns', category: 'tds', to: { s: 'tdsReturns' }, needs: ['tds'], aliases: ['challan', '26q', 'tds return', 'tds deposit'] },
  { id: 'tds.deductee', label: 'TDS Deductee-wise', category: 'tds', to: { s: 'tdsDeductee' }, needs: ['tds'], aliases: ['party wise tds', 'deductee'] },
  { id: 'tds.exceptions', label: 'TDS Exceptions / Uncertain', category: 'tds', to: { s: 'tdsExceptions' }, needs: ['tds'], aliases: ['tds check', 'uncertain deductions'] },
  { id: 'tds.quarterly', label: 'TDS Quarterly Tracker', category: 'tds', to: { s: 'tdsQuarterly' }, needs: ['tds'], aliases: ['quarter', 'due dates', '26q tracker'] },
  { id: 'tds.recon', label: 'TDS Challan Reconciliation', category: 'tds', to: { s: 'tdsChallanRecon' }, needs: ['tds'], aliases: ['challan reconciliation', 'challan match', 'bsr', 'deposit match'] },
  { id: 'tds.interest', label: 'TDS Interest & Late Fee', category: 'tds', to: { s: 'tdsInterest' }, needs: ['tds'], aliases: ['late fee', '234e', 'tds interest'] },
  { id: 'tds.form16a', label: 'Form 16A Certificates', category: 'tds', to: { s: 'form16a' }, needs: ['tds'], aliases: ['16a', 'tds certificate'] },
  { id: 'tds.tcs', label: 'TCS Outstanding', category: 'tds', to: { s: 'tcsReport' }, needs: ['tcs'], aliases: ['tcs payable', 'tax collected at source'] },
  { id: 'tds.tcs206c', label: 'TCS 206C(1H) Turnover', category: 'tds', to: { s: 'tcs206c' }, needs: ['tcs'], aliases: ['206c', 'tcs turnover', 'tcs threshold'] },
  // ── Payroll & HR
  { id: 'payroll.paysheet', label: 'Pay Sheet', category: 'payroll', to: { s: 'paySheet' }, needs: ['payroll'], aliases: ['salary sheet', 'salary register', 'payroll register'] },
  { id: 'payroll.payslip', label: 'Payslip', category: 'payroll', to: { s: 'payslip' }, needs: ['payroll'], aliases: ['salary slip', 'pay slip'] },
  { id: 'payroll.statement', label: 'Payroll Statement', category: 'payroll', to: { s: 'payrollStatement' }, needs: ['payroll'], aliases: ['payroll summary', 'salary statement'] },
  { id: 'payroll.attendance', label: 'Attendance Register', category: 'payroll', to: { s: 'attendanceRegister' }, needs: ['payroll'], aliases: ['attendance', 'present absent', 'overtime'] },
  { id: 'payroll.disbursement', label: 'Salary Disbursement', category: 'payroll', to: { s: 'salaryDisbursement' }, needs: ['payroll'], aliases: ['salary payment', 'pay salary', 'bank transfer salary'] },
  { id: 'payroll.statutory', label: 'Statutory (PF / ESI / PT)', category: 'payroll', to: { s: 'payrollStatutory' }, needs: ['payroll'], aliases: ['provident fund', 'esi', 'professional tax', 'epf'] },
  { id: 'payroll.headcount', label: 'Employee Profile / Head Count', category: 'payroll', to: { s: 'headCount' }, needs: ['payroll'], aliases: ['head count', 'employees', 'staff list'] },
  { id: 'payroll.comparison', label: 'Multi-Period Comparison', category: 'payroll', to: { s: 'payrollComparison' }, needs: ['payroll'], aliases: ['payroll comparison', 'month wise salary'] },
  { id: 'payroll.gratuity', label: 'Gratuity Provision', category: 'payroll', to: { s: 'gratuity' }, needs: ['payroll'], aliases: ['gratuity'] },
  { id: 'payroll.leaveregister', label: 'Leave Register', category: 'payroll', to: { s: 'leaveRegister' }, needs: ['payroll'], aliases: ['leaves taken', 'leave history', 'time off'] },
  { id: 'payroll.leavebalance', label: 'Leave Balance', category: 'payroll', to: { s: 'leaveBalance' }, needs: ['payroll'], aliases: ['leaves left', 'leave register balance'] },
  // ── Accountant
  { id: 'accountant.tb', label: 'Trial Balance', category: 'accountant', to: { s: 'trialBalance' }, aliases: ['tb', 'trial'] },
  { id: 'accountant.daybook', label: 'Day Book', category: 'accountant', to: { s: 'daybook' }, key: 'K', aliases: ['all vouchers', 'journal register', 'daily transactions', 'transactions'] },
  reg('accountant.journals', 'Journal Register', 'accountant', 'Journal', ['journal report', 'jv register', 'adjustments']),
  { id: 'accountant.ledger', label: 'Ledger', category: 'accountant', to: { s: 'pickMaster', kind: 'ledger', goal: 'book' }, aliases: ['ledger book', 'account statement', 'general ledger', 'account transactions', 'customer statement', 'vendor statement'] },
  { id: 'accountant.group', label: 'Group Summary', category: 'accountant', to: { s: 'pickMaster', kind: 'group', goal: 'book' }, aliases: ['group', 'account group summary', 'account type summary'] },
  { id: 'accountant.coa', label: 'Chart of Accounts', category: 'accountant', to: { s: 'coa' }, key: 'H', aliases: ['coa', 'account tree', 'accounts list'] },
  { id: 'accountant.memo', label: 'Memorandum Register', category: 'accountant', to: { s: 'vchRegister', mode: 'Memorandum' }, aliases: ['memorandum', 'not in books'] },
  { id: 'accountant.optional', label: 'Optional Vouchers', category: 'accountant', to: { s: 'vchRegister', mode: 'Optional' }, aliases: ['optional', 'draft vouchers', 'draft'] },
  { id: 'accountant.postdated', label: 'Post-dated Vouchers', category: 'accountant', to: { s: 'vchRegister', mode: 'PostDated' }, aliases: ['post dated', 'future dated', 'postdated'] },
  // ── Cost Centres & Budgets
  { id: 'costing.analysis', label: 'Cost Centre Analysis', category: 'costing', to: { s: 'costAnalysis' }, needs: ['costCentres'], aliases: ['cost center', 'project profit', 'department'] },
  { id: 'costing.report', label: 'Cost Category / Centre Report', category: 'costing', to: { s: 'costCentreReport' }, needs: ['costCentres'], aliases: ['cost category', 'category centre'] },
  { id: 'costing.summary', label: 'Cost Centre Summary', category: 'costing', to: { s: 'costCentreSummary' }, needs: ['costCentres'], aliases: ['cost centre totals'] },
  { id: 'costing.pl', label: 'Cost Centre P&L Breakup', category: 'costing', to: { s: 'costPl' }, needs: ['costCentres'], aliases: ['cost centre profit', 'project p&l'] },
  { id: 'costing.budget', label: 'Budget vs Actual', category: 'costing', to: { s: 'budgetVsActual' }, aliases: ['variance', 'budget variance', 'plan vs actual', 'budget'] },
  // ── Activity & Audit
  { id: 'activity.log', label: 'Edit Log (Audit Trail)', category: 'activity', to: { s: 'auditLog' }, aliases: ['audit trail', 'who changed', 'history', 'audit', 'activity logs'] },
  { id: 'activity.cancelled', label: 'Cancelled / Deleted Vouchers', category: 'activity', to: { s: 'cancelledRegister' }, aliases: ['deleted', 'cancelled', 'void', 'removed entries'] },
  { id: 'activity.exceptions', label: 'Exception Reports', category: 'activity', to: { s: 'exceptions' }, aliases: ['exceptions', 'negative stock', 'negative cash', 'checks'] },
  { id: 'activity.sync', label: 'Sync Mismatch Report', category: 'activity', to: { s: 'syncConflicts' }, aliases: ['conflict', 'conflicts', 'clash', 'offline', 'mismatch'] },
  // ── CRM
  { id: 'crm.leads', label: 'Leads', category: 'crm', to: { s: 'leads' }, aliases: ['prospects', 'enquiries', 'inquiries'] },
  { id: 'crm.pipeline', label: 'Pipeline Summary', category: 'crm', to: { s: 'pipeline' }, aliases: ['funnel', 'stages', 'deals'] },
  { id: 'crm.followups', label: 'Follow-ups Due', category: 'crm', to: { s: 'followUps' }, aliases: ['tasks', 'calls due', 'reminders', 'to do'] },
];
```

### Not in the catalogue, and why

| Screen | Why it is not a report |
|---|---|
| `dashboard` | home screen, not a report (Gateway W, rail Dashboard) |
| `reportsCenter` | this screen |
| `reportView` | stand-in launcher |
| `printVoucher` | one-shot print preview |
| `brsImport` | tool: imports a bank statement (Banking > Import Statement) |
| `batchSummary` | stub (ComingSoon) |
| `budgets` | master list with its own create (Gateway Create > Budget) |
| `monthlySummary` | drill-down, reached by Group Summary |
| `ledgerVouchers` | drill-down, reached by Ledger |
| `stockItemVouchers` | drill-down, reached by Stock Item |
| `groupSummary` | drill-down from Trial Balance |
| `billsOfLedger` | drill-down |
| `costCentreVouchers` | drill-down |
| `menu` | a menu |
| `gstRateSetup` | setup (GST Rate Setup) |
| `priceLevels` | setup |
| `pickMaster` | chooser (the entries that matter are listed with goal=book) |
| `onboarding` | personal tool (Getting Started) |
| `snippets` | personal tool (Narration Snippets) |
| `shortcuts` | personal tool (Favourite Shortcuts) |
| `companySelect` | company switching (Switch Company) |
| `companyCreate` | company setup |
| `companyAlter` | company setup (Company Details) |
| `operatorSignIn` | sign-in gate |
| `bulkEntry` | tool: Bulk Entry (Data & Utilities) |
| `dataCheck` | utility: Data Check (Data & Utilities) |

---

## Appendix D. What this supersedes in `docs/design/left-nav-spec.md`

| left-nav-spec | Status |
|---|---|
| Section 4 (the 14 groups, 108 rows, 4.2 table, 4.3 footers, 4.5 leave-outs) | **Replaced** by section 1 and Appendix A of this spec. Footers survive per group. |
| 4.4 the two dynamic groups | Pinned unchanged; Saved views move to sit between Reports and Taxes & Compliance and list pinned views first. |
| 4.6 the name bridge | Unchanged; rows keep every alias they had (and gain Zoho words). |
| 5 the collapse chord `Ctrl+\` | Unchanged and still taken. `Alt+Q` is added beside it (section 4.4). |
| 6 data model | `NavItem.create` becomes a QuickCreate id; `NavItem.key` describes the row's own `to`; `NavGroup.to` is new; the rest stands. |
| 7.1 to 7.5 behaviour, focus, accessibility | Stand, with the two key deltas of 1.3 (Enter on a group opens it; Space on a group toggles it) and the group-click behaviour. |
| 8 full-screen view, 10 visual spec | Unchanged (212px, the `navhid` grid fix, tokens). |
| 9 rights and gating | Unchanged, plus the group-`to` fallback and the `create` id resolution. |
| 11 assertions | `sidenav.assert.ts`'s tree-specific parts are rewritten (1.5); the chord, CSS, brand and model parts stand. |
| Appendix C and D (build notes, fix round) | Stand as history. |

The deltas to the three-column grid, `navhid`, and `.nv-wide` are none.


---
