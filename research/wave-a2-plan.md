# Wave A2 build plan: the report grid (B-grid) and Dashboard v2 (B-dash)

> **Status:** plan only. Nothing here is built. Written 2026-10-20, to be executed by **two builders after Wave A is committed**.
> **Reads on top of (not repeated here):** `docs/research/pipedrive-list-ux.md` (the grid design: catalogue, layout, panel, header menu, quick filter), `docs/research/zoho-dashboard.md` (the dashboard design: widgets, engine, chart kit, layout engine), `docs/design/zoho-ux-spec.md` (Wave A: header slots, filter engine, saved views, the builder rules in its section B), `docs/design/ux-analysis-2026-10.md` (findings F-11, N2, N4, T9) and `docs/research/zoho-books-reference.md`. This document only **decides, sequences, assigns files, and fixes the contracts**; where a research doc already says how, it is cited by section.
> **Method:** the three research docs and the Wave A spec read in full; the working tree read for every claim that changes a decision (files named); three Pipedrive and Zoho pages re-fetched this session to close gaps the plan depends on (section 14). Paraphrased, never copied.
> **Evidence tags:** **[C]** read in code this session (file named) . **[V]** verified by running a grep or reading a suite's own rule this session . **[R]** from a research doc (named) . **[W]** a vendor page fetched this session . **[Ws]** a web-search summary only . **[J]** my judgement (all effort numbers are [J], in ideal builder-days of one engineer who knows the repo, tests and screenshots included, the unit both research docs use) . **[inferred]** / **[unverified]** as stated.

---

## 0. Read this first

**What A2 is.** Two owners working in parallel on disjoint files, after one short serial scaffold step (A2-0, section 3):

| | **B-grid** | **B-dash** |
|---|---|---|
| Delivers | Pipedrive-style report lists: a field catalogue per row type, columns from related objects, Columns panel, per-column header menu (sort, filter, hide, move), an inline quick-filter popover with editable chips, layout saved with views, selection actions | Zoho-style Dashboard v2: one period and one "as at", a chart kit, the Zoho home set of widgets, **customise mode on the board and a widget catalogue (gallery)**, widgets that can be built from a saved view |
| First reports / widgets | Day Book, registers (listing level), Bills Receivable and Payable, Ledger Vouchers (section 5.2) | Receivables, Payables, Cash and bank, Cash flow, Income and expense, Top expenses (section 6.2) |
| Research MVP it implements | `pipedrive-list-ux.md` section 11, P0 to P3, plus selection actions | `zoho-dashboard.md` section 4, Phases 0 and 1, plus the customise-mode half of Phase 3 and the view-widget join |
| Effort [J] | 24 to 29 builder-days | 26 to 32, plus 2 for the stretch widgets |
| New screens | none | none |

**The one dependency edge, and it runs one way.** B-dash's "widget from a saved view" and its Overdue-invoices list call **`core/listQuery.ts`, which B-grid writes** (section 7, C2). Nothing in B-grid needs B-dash. B-dash's other work (engine, charts, period, board, edit mode) does not wait for B-grid, so the edge costs no idle time: B-grid reaches `listQuery` about day 9 to 11; B-dash needs it about day 16 to 20 [J].

**Decisions taken in this plan** (each can be vetoed; the default if the owner is silent is the one written here):

| # | Decision | Why (section) |
|---|---|---|
| A2-1 | The Columns chord is **`Alt+M`**, not the `Alt+C` the Pipedrive research proposed | `Alt+C` is quoted in at least seven source files (new ledger on the fly in vouchers, New column on statements, Clear 2B, Bulk Entry, Stock Summary), and Wave A's registry suite fails any live chord that is spelled outside its handler file (0.1) |
| A2-2 | `Alt+F12` and `Ctrl+F12` keep their Wave A meaning (a range, a value) and, on a report that has a catalogue, open the **popover** instead of the modal; the modal stays as **Advanced** | the one Wave A behaviour the Pipedrive research re-specs; it is a seam in `FilterBar.tsx`, not a rewrite (4.4) |
| A2-3 | **No new screens in A2.** The Parties and Items lists (the strongest Pipedrive fit) are the first thing in A3, unless the owner says otherwise (D1) | a new screen touches nine hot files (router, store union, access, recents, MenuScreen, navTree, palette, Go To, QC_PAGES) and two assertion literals; the four report families already deliver the three behaviours the owner named |
| A2-4 | One scaffold step A2-0 before either builder, run by one writer, as Wave A did | it removes every file two builders would both need (section 3) |
| A2-5 | **Single writer per file at any moment.** Where two builders need the same file, one owns it and the other gets a seam made in A2-0 or a time slot in the integration window | the Edit tool refuses a file changed since it was read; a stale full-file write would silently drop the other builder's edit (section 4) |
| A2-6 | The **fail-closed filter reader** and the **cfg-preserving dashboard layout reader** are the first commits of the train, before the features that need them | older clients (Electron installs lag the web app) would otherwise widen a filter or strip a widget's config on re-save (7, C9) |
| A2-7 | Dashboard widgets that list rows are **view widgets**: a saved view (or a Wave A default filter) plus a row limit, evaluated by the same `listQuery` the report grid uses. No second filter or column implementation on the dashboard | the contract the owner asked for; it is also the answer to Zoho's report panels that `zoho-dashboard.md` 3.7 declined to build as a separate engine (7, C3) |
| A2-8 | Cash against accrual, windowing, model-driven Print and Export, pinned columns, company default columns, bulk edit of anything, and the GST due card are **out of A2** (section 1) | each is named with its reason and the phase it moves to |

### 0.1 What I checked in the tree that changes the research docs' advice

1. **`Alt+C` cannot be registered.** `scripts/assert/zoho-scaffold.assert.ts` section 8 requires, for every live registry chord, that no other source file quotes it (`binders.every(b => b === handler)`). A grep for `Alt+C` finds quoted uses in `ui/VoucherScreen.tsx`, `ui/reportColumns.tsx`, `ui/gst/gstr2b.tsx`, `ui/admin/BulkEntry.tsx`, `ui/inventory/stocksummary.tsx`, `ui/overlays.tsx`, `ui/favouritesCore.ts` and `ui/onboarding/GettingStarted.tsx`. `Alt+M` has **zero** hits in `src/` and is on Wave A's list of free letters (M Q U X Z; Q is now taken; U is reserved for the attachments research). Mac and AltGr behaviour is asserted the way `Alt+Q`'s is (section 8). [V]
2. **The perf harness has two rules that bite a refactor.** It bundles the real engines and **refuses** any that import `src/ui/` (except `paletteRank.ts`), `src/state/` or Dexie; and under `--enforce` it **fails on drift** of its "mirror" snippets, which are exact source lines in `ui/reports.tsx`, `ui/modern/dashWidgets.tsx` and others (`scripts/perf/run.mjs`, `scripts/perf/mirrors.mjs`). So `core/listQuery.ts` and `core/dashIndex.ts` must be pure `core/` files, the sort primitives that now live in `ui/reports/columnSort.ts` must move to `core/`, and any edit to a mirrored line must update `mirrors.mjs` in the same change (the Day Book's `st.ledgers.find(...)` line is itself a mirror row, `rpt.daybook.find`). [V, C]
3. **Selection is keyboard-only and voucher-only today.** `useVoucherSelection` (Space, Shift+Down, Esc) is used by the Day Book, the three special registers and the register's month listing; the Bills and Ledger Vouchers screens have **none**, and a bill has no voucher id (`BillState` is `{ledgerId, name, openedOn, dueDate, billAmount, balance, onAccount}`). There is no mouse marking. "Bulk selection actions" therefore means generalising the id-keyed selection to every catalogue report, not importing Pipedrive's bulk edit (4.4 and 12). [C `ui/reports/selectionBar.tsx`, `core/billwise.ts`]
4. **Ledger Vouchers has no column sort today** (no `useColumnSort`; its `th`s are plain), and its optional Balance column is a running balance keyed by voucher id (`ledgerSplit().running`). Sort arrives with the grid, and the running balance must stay id-keyed so a sort never rewrites it. [C `ui/reports.tsx:1394-1409`, `core/reportRows.ts`]
5. **`QC_PAGES` has no `dashboard` entry**, so the Dashboard has no `+ New` from Wave A's registry; its own "+ New Sales Invoice" button is still hard-coded. Adding the entry is a seam (it also adds a row to the literal table in `quickcreate.assert.ts`). The Dashboard renders no `ButtonPanel`, so the F-key rail column disappears there; the fix is a `<ButtonPanel>` inside `Dashboard.tsx` itself (every screen renders its own as a sibling). [C `ui/quickCreate.ts`, `ui/modern/Dashboard.tsx`, `ui/AppShell.tsx`]
6. **Both research docs want to edit `ui/reports.tsx` and `ui/reports/financial.tsx`** (the grid ports the Day Book, Ledger Vouchers and the register listing; the dashboard extracts `plFigures` from Profit and Loss and Ratio Analysis, and wants a Cash Flow option). Under rule A2-5 the `plFigures` extraction moves into A2-0, and the Cash Flow option waits for the integration window (section 4.3).

### 0.2 What could not be checked

- No live Pipedrive or Zoho account. Row selection, a header hide/pin menu and a footer total in Pipedrive are still **[unverified]**: the two pages re-fetched this session do not describe them either [W]. The grid's selection and header menu are therefore **our design**, not a copied behaviour.
- Nothing in this plan has been run. Effort numbers and the spike thresholds in 5.5 are [J] and are to be replaced by the P0 numbers.
- Wave A is uncommitted and moving. Every file and function name below is as of the working tree on 2026-10-20; A2-0 starts by re-checking them against the commit.

---

## 1. Scope: in A2, deferred, and never

| Item | A2 | Moves to | Reason |
|---|---|---|---|
| Field catalogue, ListTable, Columns panel, header menu, quick filter popover, layout in saved views | **in** (B-grid) | | the owner's three Pipedrive loves |
| Four report families (5.2) | **in** | | MVP |
| Selection: id-keyed marking on every catalogue report, modifier-click, selection total, Print/Export/Mark Optional | **in** | | the safe subset of "bulk actions" |
| New operators `isEmpty`, `notEmpty`, `anyOf` | **in, behind a constant** until the fail-closed reader has shipped (D10) | | 7, C9 |
| Dashboard v2: period, as-at, chart kit, Zoho home widgets, customise mode, gallery, view widgets | **in** (B-dash) | | |
| Today strip, Setup card | stretch, only after Gate 2 is green | A3 | no new engine; first-run value |
| Parties and Items lists | no | **A3, first** | 0, A2-3 |
| Memorandum, Optional, Post-dated registers (`ui/reports/vchregisters.tsx`), Stock Summary attributes, Leads | no | A3 | same catalogue, more ports |
| Windowed rendering, model-driven Print, CSV and `/` text filter, raw-number export | no | A3, **if the P0 spike says so** | all three read the DOM today (5.5) |
| Pinned columns, company default columns for everyone, hover card, item-wise register, bulk edit of masters | no | A3 or later | `pipedrive-list-ux.md` P5 and P6 |
| Cash/accrual toggle on Income and expense | **never as an approximation** | a Phase 4 coverage spike | `zoho-dashboard.md` 3.5 |
| GST due card with statutory dates, TDS date | no (the GST tile's **words** are corrected, D8) | A3 | needs a cited dates module and the QRMP state table |
| Incremental index update, Highlights, named boards, role defaults, synced personal layout | no | later | `zoho-dashboard.md` Phase 3 and 4 |
| Bulk edit or delete of vouchers, Pipedrive's owner filter, kanban, plan caps, URL-shared filters | **never** | | `pipedrive-list-ux.md` section 10 |

---

## 2. Ground rules (both builders; Wave A's rules 1 to 7 in `zoho-ux-spec.md` section B still apply)

1. **Single writer per file** (A2-5). Section 4 lists every file with exactly one owner. A builder who needs a file it does not own asks the orchestrator; it never edits it "just this once".
2. **No git** for builders. The orchestrator commits at Gate 0, Gate 1, Gate 2 and after the fix pass (section 10), and runs the ownership check at each (4.6).
3. **Pure engines.** Anything under `src/core/` imports no `src/ui/`, `src/state/`, React, Dexie or DOM (type-only imports are erased and allowed). The perf harness enforces it (0.1 item 2).
4. **New chords** only as entries in `core/shortcutRegistry.ts`; display a chord with `chordOf(id)`, never by quoting it, or the registry suite fails ("bound by its handler only").
5. **Never toggle a global handler's `active`**; popovers, panels and edit modes mount a `useKeys` handler **only while open**; screen handlers use `useScreenKeys`. A hidden stacked screen (`AppShell` keeps them mounted) renders nothing visible, registers no keys and **computes nothing** (B-dash adds `useIsTop` gating).
6. **A live filter is never written to storage** (Wave A). A layout and a dashboard layout may be (they cannot hide a row or change a total), always inside try/catch with a working fallback.
7. **Ids are forever.** Field ids, column ids, widget ids, view ids saved anywhere are never renamed; a new id is appended.
8. **Optional vouchers, totals and the books**: a column, a sort, a hidden column or a widget never changes which rows a report shows or what its totals say; only a filter does, and it says so ("Filtered", `(filtered)`).
9. **Tokens only** (Wave A rule 5): no hex in any new CSS except the `--viz-*` block B-dash owns; light-only looks in `styles/light/*.css` under `html:root[data-scheme='light']`.
10. **Perf mirrors move with the code** (0.1 item 2): a builder who edits a mirrored line updates its own mirrors file in the same edit.
11. **Docs are the orchestrator's.** Builders do not edit `CLAUDE.md`, `docs/HANDOFF.md`, `docs/PERF.md`, `docs/parity-data.json` or `docs/research/ROADMAP-ADDENDUM.md`; they report numbers and new `beyond` rows in their final summary. A2 is **not parity work** and its rows go under `beyond`, never into the parity percentage (CLAUDE.md, Phase 18 to 21 note).
12. **Every new suite ships with deliberate mutations**, each killed (the Phase 25 practice, 28 mutations on `sidenav`): the builder lists them in its summary.

---

## 3. Step A2-0: the scaffold (serial, one writer, before either builder starts)

**Prerequisite:** Wave A is committed and Gates are green on that commit (tsc 0, build, `npm run assert`, `perf` with `--enforce`, a full `shots --strict` sweep). A2-0 records the commit hash at the top of its output; if any name below has moved, A2-0 corrects this plan's table, not the builders.

| # | Deliverable | Files | Why it is serial |
|---|---|---|---|
| 0.1 | **Golden captures of today's tables** and of today's Profit and Loss and Ratio Analysis figures. A small script `scripts/shots/golden.mjs` that reuses the harness (`boot`, `openSample`, `goTo`, `page.eval`) in record and compare modes and writes, per family, the header text and the first 20 rows' text on the sample company (pinned date) and, for the Day Book and Bills, on a larger seeded company; JSON committed under `scripts/shots/golden/`. A node assertion suite cannot render the React tables, and recipes are screenshot targets, so this is a harness script [C `scripts/shots/page.mjs` has `eval`; `recipes/_helpers.mjs` describes recipes as screens to shoot] | new | B-grid's "Standard layout is unchanged" gate and B-dash's `plFigures` equality gate both need a before-picture taken **before** any code moves |
| 0.2 | **`core/plFigures.ts`** extracted from `ProfitLoss` (`ui/reports.tsx`, about line 1840) and `RatioAnalysis` (`ui/reports/financial.tsx`, about line 190), both screens calling it, with `scripts/assert/plfigures.assert.ts` proving the figures equal the goldens in 0.1 with stock, without stock, with a stock change and with a loss; the `rpt.profitLoss` (and `rpt.balanceSheet` if touched) mirror rows updated | `core/plFigures.ts`, the two call sites, `scripts/perf/mirrors.mjs` | the only file-level overlap between the builders (0.1 item 6); a behaviour-preserving refactor that must land before B-grid starts editing the same two files |
| 0.3 | **Type-only seams** (no behaviour; every one returns `null`/no-op so the screens are byte-identical): `SavedView.layout?` and `LiveView.layout?` in `core/savedViews.ts`; `ReportActions.columns?` and `ReportActions.grid?` plus a `publishReportGrid` stub in `ui/reportActions.ts`; `DashItemStored.cfg?` in `types.ts` (check `master-resave.assert` stays green); in `ui/header/FilterBar.tsx` a delegation point that renders `ui/grid/QuickFilterMount.tsx` (a stub returning `null`) and routes `Alt+F12`/`Ctrl+F12` to it when `actions.grid` is present, else to the existing panel; in `ui/header/PageActions.tsx` a "Columns..." item shown only when `actions.columns` exists | the listed Wave A files | each is a Wave A file that exactly one builder will later fill from a file it owns |
| 0.4 | **Registry**: append `columns` (`Alt+M`, label "Choose columns", sheet `report`, `live: false`, handler `src/ui/grid/ColumnsPanel.tsx`) to `core/shortcutRegistry.ts`. B-dash adds none (F2 and F12 predate the registry; edit-mode keys live inside the panel, as the Customize panel's do) | `core/shortcutRegistry.ts` | one shared append-only file, one writer |
| 0.5 | `QC_PAGES.dashboard` (primary Invoice; related Bill, Payment Received, Payment Made, Journal Entry) and its row in `quickcreate.assert.ts`'s literal table | `ui/quickCreate.ts`, assertion | Wave A file |
| 0.6 | **CSS imports**: empty `styles/grid.css`, `styles/light/grid.css`, `styles/dashviz.css`, `styles/light/dashviz.css`, imported from where `styles/light.css` and the app import their siblings | new plus `styles/light.css` | `light.css` is shared |
| 0.7 | **Perf files split by owner**: `scripts/perf/mirrors.grid.mjs` and `mirrors.dash.mjs` (the existing `rpt.daybook.find` row moves to the first, the `dash.*` rows to the second, `mirrors.mjs` imports both); `passes.grid.ts` and `passes.dash.ts` likewise, imported from `bench.ts`. `budgets.mjs` stays untouched (new rows are reference rows) | `scripts/perf/*` | otherwise both builders edit `mirrors.mjs` and `passes.ts` |
| 0.8 | **Ownership manifest** `scripts/a2-owners.json` (file or directory to owner: `grid`, `dash`, `a2-0`, `orchestrator`, `wave-a-frozen`) and a 20-line checker the orchestrator runs against `git status --porcelain` (4.6) | new | makes rule A2-5 checkable |

**Gate 0** (commit): tsc 0, build, all suites green with Wave A's suites **unmodified** except `quickcreate.assert.ts`'s one row, perf `--enforce` green (mirrors found), `shots --strict` over all screens **byte-identical** to the Wave A commit (every stub draws nothing). Effort: 3 to 4 builder-days [J].

---

## 4. File ownership

Legend: **grid** = B-grid, **dash** = B-dash, **a2-0** = the scaffold step, **orch** = orchestrator. "Edit zone" = the named function or region only.

### 4.1 B-grid owns (new files)
`src/core/reportFields.ts`, `reportFieldCtx.ts`, `columnSets/{daybook,register,ledgerVouchers,bills}.ts`, `listLayout.ts`, `listQuery.ts`, `sortRows.ts` (moved from the UI) . `src/ui/grid/*` (`ListTable.tsx`, `useListLayout.ts`, `ColumnsPanel.tsx`, `HeaderMenu.tsx`, `QuickFilter.tsx` and its mount, `SelectionBar.tsx`, `gridKeys.ts`) . `src/styles/grid.css`, `src/styles/light/grid.css` . `scripts/assert/{reportfields,listlayout,listquery,gridsort,grid-quickfilter,grid-keys,grid-selection}.assert.ts` . `scripts/shots/recipes/columns.mjs` . `scripts/perf/{mirrors,passes}.grid.*`.

### 4.2 B-dash owns (new files and the Dashboard's own)
`src/core/dashIndex.ts`, `dashBuckets.ts`, `dashScale.ts`, `dashListWidget.ts` (the view-widget cfg model) . `src/core/dashboardLayout.ts`, `dashboardData.ts` (existing, B-dash only) . `src/ui/modern/{Dashboard,DashCustomise,dashStore,useDashLayout,dashEditRequest}.ts*` and the split of `dashWidgets.tsx` into `src/ui/dash/widgets/*` . `src/ui/dash/charts/*`, `src/ui/dash/gallery/*`, `src/ui/dash/edit/*` . `src/styles/dashcustom.css`, `dashviz.css`, `light/dashviz.css` . `scripts/assert/{dashboard-v2,dashscale,viz-tokens}.assert.ts` and the deliberate rewrites of `dashboard-custom.assert.ts` and (if D2 is yes) `security-access.assert.ts`'s Dashboard line . `scripts/shots/recipes/{dashboard-custom,dashboard-v2}.mjs` . `scripts/perf/{mirrors,passes}.dash.*`.
`src/ui/modern/CommandPalette.tsx` is **not** B-dash's (Wave A's); B-dash touches only `Dash*`, `dashWidgets.tsx`, `dashStore.ts`, `useDashLayout.ts`, `dashEditRequest.ts` in that directory.

### 4.3 Shared files: one writer, and when

| File | Writer | Zone / timing | The other builder |
|---|---|---|---|
| `ui/reports.tsx` | **grid** | `DayBook`, `LedgerVouchers` (and imports). `ProfitLoss` already done in A2-0 | reads only |
| `ui/reports/financial.tsx` | **grid**, then **dash** | grid: `RegisterReport` listing level (S1). dash: the Cash Flow option, **only in the integration window after Gate 2** and only if D3 is "report option" | the other waits for its slot |
| `ui/reports/outstanding.tsx` | grid | the bill-wise view (the ledger-wise view is not touched) | none |
| `ui/reports/columnSort.ts` | grid | re-export of the moved primitives; `useColumnSort` itself unchanged (six other screens use it) | none |
| `styles.css` | **dash** | the `--viz-*` tokens (a light-only block 5b and a dark override), nothing else | grid must not edit it |
| `core/access.ts` | dash | one line `dashboard`, **last**, **only if D2 is yes** | none |
| `scripts/assert/zoho-scaffold.assert.ts` | a2-0 only | flips for `columns` happen when grid sets `live: true` in the registry; the suite itself is not edited | none |
| `docs/*`, `CLAUDE.md`, `parity-data.json`, `scripts/perf/budgets.mjs` | orch | at integration | builders report, do not edit |

### 4.4 Wave A files: the seams (everything else in Wave A's set is read-only to A2)

| Wave A file | A2 writer | Edit zone (and nothing else) | Purpose |
|---|---|---|---|
| `core/reportFilter.ts` | grid | `cleanCondition`/`sanitiseFilter` fail closed on an unknown operator (keep the condition, match nothing); later the three operators in `OPS`, `compileCondition`, `describeNode`, behind the constant of D10 | 7, C9 |
| `core/reportFilterFields.ts` | grid | `filterFieldsFor` returns the catalogue's filterable columns; the exported `DAYBOOK_FIELDS`, `REGISTER_FIELDS`, `LEDGER_VOUCHER_FIELDS`, `BILL_FIELDS` and every Wave A id, kind and **order** stay exactly as they are; new ids are appended | back-compat of saved views |
| `core/savedViews.ts` | grid | the bodies behind the A2-0 `layout` types: `captureView`, `planApply`, `viewMatches`, `viewPhrases`, `sanitiseViewList` | layout travels with a view |
| `ui/reportViews/apply.ts`, `bridge.ts` | grid | `liveViewOf` reads the layout; `applyView` writes it; `bridge.ts` gets `registerLayoutAccess`/`readReportLayout`/`writeReportLayout`, the same door as the filter's | |
| `ui/reportViews/viewStore.ts`, `useReportViews.tsx`, `Launcher.tsx`, `pin.ts` | **nobody** | | B-dash reads views through `getViews`/`useViews` and opens them through the existing `{ s: 'reportView', viewId, target }` push |
| `ui/rowFilter.ts` | grid | `useRowFilter` takes catalogue fields; the sort state moves into a live store keyed like the filter (`liveKey`); the storage rule (no live filter in storage) is unchanged | |
| `ui/reportActions.ts` | grid | the publish bodies for the A2-0 `columns`/`grid` types | |
| `ui/header/FilterBar.tsx` | **nobody** (A2-0 made the seam) | | grid fills `ui/grid/QuickFilter.tsx` |
| `ui/header/PageActions.tsx` | **nobody** | | |
| `ui/header/ViewsTitle.tsx` | grid | the label function only: with no filter but a non-Standard layout and no matching view the button reads "Custom view" | |
| `ui/header/FilterPanel.tsx` | **nobody** | | stays as Advanced, reached from the popover footer and the `...` menu |
| `scripts/shots/recipes/rowfilter.mjs` | grid | the steps that click the old chips (the chips become editable buttons) | the Wave A recipe would otherwise fail |
| `ui/quickCreate.ts`, `ui/navTree.ts`, `ui/nav/*`, `ui/qc/*`, `core/navPlan.ts`, `ui/navigate.ts`, `ui/navGuard.tsx`, `state/{navGuard,navStore,qcStore,dirty}.ts`, `ui/reportsCenter/*`, `ui/reportCatalog.ts`, `state/reportVisits.ts`, `ui/pageHeader.tsx`, `router.tsx`, `state/store.ts`, `state/recents.ts`, `MenuScreen.tsx`, `favouritesCore.ts`, `CommandPalette.tsx`, `overlays.tsx`, `widgets.tsx`, `AppShell.tsx`, `ThemeControls.tsx`, `masters.tsx`, `scaffold.tsx`, `screenCtx.ts` | **nobody** | | the "nav and quick-create" set; A2 uses `pageCreates`, `useIsTop`, `screenDenial` and `requestNav` as callers only |

A route to the Reports Center is not needed: `ui/reportCatalog.ts` already lists the four families and the saved views show under My reports and Shared reports by Wave A's own mechanism.

### 4.5 Never touched in A2
`backend/`, `electron/`, `src/data/db.ts` (no Dexie bump: layouts live in `localStorage` and saved views; the dashboard's `cfg` is inside `Company.dashboard` JSON), the Gateway tree, the F-key block of any voucher screen.

### 4.6 The ownership check
At every gate the orchestrator runs the A2-0 checker: every path in `git status --porcelain` since the previous gate must be owned by the builder whose gate it is, or sit in the seam table above, or be a file the builder created under its own prefix. A path outside that set is reverted or reassigned before the gate passes. [J]

---

## 5. B-grid

### 5.1 What a person gets (against the owner's words, `pipedrive-list-ux.md` section 2)

1. **"Add multiple columns in a report from a particular object."** `Alt+M` (or a button in the header's `...` menu and the filter bar) opens a Columns panel: search, tick fields grouped by the object they belong to (This report, Voucher, Party, Items, Calculated, Your custom fields), reorder, reset to Standard. Party attributes appear on a list of vouchers or bills; the party's custom fields do too. Standard is exactly today's table.
2. **"A quick filter on the top-left, very dynamic."** The bar under the header (Wave A's slot) gains an inline popover: type to search a field, pick a value, `Enter`; chips are editable in place; a funnel appears on a column header on hover or focus. The Wave A modal remains as Advanced for OR groups.
3. **"In columns there are options like sorting and filtration."** A caret on each header opens: Sort ascending, Sort descending, Clear sort, **Filter by this column**, Hide, Move left, Move right, Reset width; click still sorts, **Shift+click** adds a secondary sort (Pipedrive's documented gesture [R, W]).
4. **Saved with views.** A saved view (personal or company) carries the layout (columns, order, widths, sort) beside its filter; recalling a view restores both.
5. **Selection actions** on every catalogue report (5.4).
6. It also answers UX finding **N2** for the flat registers and Bills: Status, Due date, Paid, Balance and Days overdue of an invoice are columns, not a new screen; and the owner's "Create New Report" gets its 80 % case (a register with my columns, my filter, a name).

### 5.2 The reports, exactly, in build order

| # | Report | Where it is today | Row object and `rowId` | Standard layout (today's table, must not change) | Offered in A2, by object |
|---|---|---|---|---|---|
| 1 | **Day Book** | `ui/reports.tsx` `DayBook`; fields `DAYBOOK_FIELDS` | voucher, `v.id` | Date, Particulars, Vch Type, Vch No., then Amount **or** Debit and Credit (the F12 `DRCR_SHAPE` option), then Narration (the F12 `NARRATION` option). So Standard is a function of F12, not a static flag | **Voucher:** Ref. no., Status (Regular, Optional, Post-dated), Place of supply, Reverse charge, Supplier invoice no. and date. **Calculated:** Taxable value, GST amount, Age in days, Month, and (bill-wise on) Due date, Paid, Balance, Invoice status, Days overdue. **Party (current):** GSTIN, State, City, Phone, Email, PAN, Credit limit, Credit period, Ledger group, party custom fields. **Items (one-to-many collapsed):** Items (count), First item, Main HSN (the line's own snapshot), Total qty. **Compliance:** IRN, E-way bill no. |
| 2 | **Registers, listing level** (Sales, Purchase, Receipt, Payment, Journal, Contra, notes, orders) | `ui/reports/financial.tsx` `RegisterReport`, the flat level of Wave A section 6.7 (and a month's drill if it shares the listing code, **[inferred]**: A2 confirms) | voucher, `v.id` | Date, Particulars, Vch No., Amount, then Narration and % per F12; has a Total row | as the Day Book minus the Debit/Credit pair. `Taxable value` and `GST amount` may be summed in the Total row (additive on the voucher's own measure) |
| 3 | **Bills Receivable and Payable**, bill-wise view | `ui/reports/outstanding.tsx`; fields `BILL_FIELDS` | bill, `${ledgerId}\|${bill name or 'onaccount'}` | Date, Ref. No., Party's Name, Pending Amount, Due on, Overdue by days, Ageing; Total row | **Calculated:** Original bill amount (sum), Status (Open, Part-paid, Overdue, On account). **Party (current):** the party set above plus **Closing balance** and **Credit used %** (closing over credit limit; blank without a limit), **Last transaction date**; neither summable. Originating voucher no. needs a bill-to-voucher index (the engine has no voucher id): **A3** |
| 4 | **Ledger Vouchers** | `ui/reports.tsx` `LedgerVouchers`; fields `LEDGER_VOUCHER_FIELDS` | voucher as one ledger sees it, `v.id` | Date, Particulars (the counter ledger), Vch Type (the type's own name), Vch No., Amount or Debit and Credit, Balance (F12 `LV_RUNNING`), Narration (F12) | Voucher, Calculated and Items sets as the Day Book; **no Party set** (the row's other party is the counter ledger and can be several, so the relation is ambiguous; A3 decides) |

**Per-report rules the builder asserts:**
- **Column 0 is locked** (Date on all four): the Total row's label and the `/` text filter's "never hide a total" backstop read the first cell (`useReportFilter` `labelsATotal`).
- **Day Book has no Total row and does not get one** because a column was added. A report with no footer gets none.
- **Amount stays the report's own function** (`amountCells` on the Day Book, `registerAmount` on registers, `Math.abs(bill.balance)` on Bills): the catalogue imports or wraps it and a mirror test compares the two on every sample row. The Phase 20 to 21 gotcha (a voucher's amount is its party leg) is exactly this.
- **Party relation = `v.partyId`'s ledger master**, not the Day Book's lead-ledger name; a Journal or Contra has no party and shows blank cells.
- **Ledger Vouchers keeps `Opening + shown + not shown = Closing`** and its running balance **keyed by voucher id** whatever the sort.
- **Vch Type column vs the filter field "Voucher kind"**: the column shows the company's own type name (`Sales - Export`) and filters on the base; the picker says so.
- **`truth: 'now'`** columns (party state, GSTIN, credit limit, item group) carry "(current)" in the picker and the header text; HSN and GST rate read the **line's snapshot** (`InventoryLine.gstRate/hsn`), never the item master (CLAUDE.md Phase 20 to 21 gotcha).
- **Not in A2:** Memorandum/Optional/Post-dated registers, Stock Summary, Cancelled and Deleted Vouchers, Leads, statutory tables (class B), statements (class C), **BRS (excluded for good until rebuilt on the shared table)**.

### 5.3 Design decisions on top of `pipedrive-list-ux.md` section 5 to 9 (only the deltas)

| Topic | Decision |
|---|---|
| Catalogue home | pure `core/` (`reportFields.ts` types; `columnSets/*` per family; `reportFieldCtx.ts` builds the lookup context `X` once per render from the store arrays plus lazy indexes `ledgersById`, `bills`, `einvoice`, `closing`, `partyStats`, each linear and built only when a column that needs it is shown, filtered or sorted). Pure so `listQuery`, the perf harness and B-dash can use it |
| Filter ids | Wave A's ids (`date, party, type, number, amount, narration, debit, credit, ref, due, overdue, ageing`) keep their kind and their position in the list; a catalogue column that is filterable reuses the id; new ids are appended and typed by a helper `fid(family, id)` so a rename is a compile error for B-dash too (7, C1) |
| Sort | sort primitives move to `core/sortRows.ts` (so `listQuery` can use them under the perf rules); `ui/reports/columnSort.ts` re-exports them and `useColumnSort` keeps its behaviour for its six other callers; the grid uses a new `useGridSort` with up to two keys that **persists in the layout** and is dropped when its column is hidden (the existing rule) |
| Layout | `ListLayout { v:1, ids, w?, sort?, pin? }` as in `pipedrive-list-ux.md` 5.3; `pin` is **reserved and ignored** so a later release needs no schema bump; sanitiser caps 20 columns, widths 48 to 600, drops unknown ids, forces locked columns first, falls back to Standard |
| Live store and storage | in memory per company, report and (Ledger Vouchers) ledger, and `localStorage` `tw-report-layout-v1:<cid>` in try/catch, the precedent of `tw-report-cols-v1` |
| F12 write-through | the Day Book and Ledger Vouchers toggles (Narration, Amount versus Debit/Credit, Running balance) stay the **single source of truth**; the picker writes through them (`f12Key`) so there are never two controls with two states for one column |
| Saved views | `layout` is captured and applied with the filter; a view saved before A2 has none and **recalling it resets to Standard** (the same rule Wave A set for the filter: a view means the whole setup); an older client ignores `layout` and shows Standard, never a wrong table |
| Hidden columns | **not rendered** (never `display: none`), so Print and Export (which read the DOM) are right with no change; related-object headers carry the object in text ("Party GSTIN") |
| Widths | screen only, through a `colgroup` marked for removal in the print clone |
| Wide layouts | up to 20 columns will scroll sideways at 1280 px, which kills sticky headers if the table sits in `.hscroll` (CLAUDE.md gotcha: `overflow-x: auto` forces `overflow-y: auto`). S1 therefore renders the four tables in **one scroller that scrolls both axes**, and the recipe probes the header's position after a scroll |
| DOM stability | `ListTable` renders the **same element tree and class names** as today for the Standard layout (`tt pk-tbl`, `srt`, `r`, `sel`, `data-vid`, `data-pk`, the empty-row `colSpan`), so print CSS, `visual.assert` and the 478-screen sweep do not move |

### 5.4 Interaction (keys, gestures, selection)

| Surface | Gesture | Does | Notes |
|---|---|---|---|
| Columns panel | `Alt+M`; the `...` menu "Columns..."; the F-key rail button (its keycap printed through `chordOf('columns')`) | opens a modal in the F12 panel's family. Type to search; `Up/Down`; `Space` show or hide; `S` sort by the highlighted field (again reverses, a third time clears); `F` opens the filter popover on it; `Alt+Up/Alt+Down` move a shown column; `R` reset to Standard; `Enter` or `Esc` close | owns the keyboard only while open; dead inside a text field and behind an overlay; AltGr layouts use the button |
| Header menu | hover or focus a `th` shows a caret `button`; click or `Enter` opens | Sort asc, Sort desc, Clear sort, Filter by this column, Hide (not column 0), Move left, Move right, Reset width. Drag a header to reorder; drag the divider to resize | the caret stops propagation so it never also sorts; every action also exists in the panel, so **no mouse is ever required** |
| Header click | click | existing three-state cycle (ascending, descending, off) | unchanged |
| Header click | Shift+click | adds or changes the secondary sort | new; `Alt+S` unchanged and steps visible columns only |
| Quick filter | `+ Filter` button, `Alt+F12` (a range: first number or date field), `Ctrl+F12` (a value: first text or enum field), the header funnel | popover: step 1 field (typed search, grouped by object), step 2 operator and value; `Enter` commits and returns focus to the table on the same row; `Shift+Enter` commits and opens another; `Esc` cancels | smart value: a bare word offers "Particulars contains ..."; `>1L`, `<=50k`, `1L-5L` offer an Amount range through `parseFilterNumber`; a date word through the date mnemonics |
| Chips | click or `Enter` on a chip | reopens the popover prefilled; `Delete` or the x removes | real buttons in the Tab order |
| Advanced | popover footer, `...` menu "Filter..." | Wave A's modal, unchanged; its wording becomes "Match all / Match any" (D5) | stored form stays and/or |
| Selection | `Space`, `Shift+Down/Up`, `Esc` (unchanged) **plus** Ctrl or Command+click toggles a row and Shift+click extends | marking now works on all four families (id-keyed by `rowId`, intersected with the visible ids, Wave A's `visibleIds` rule) | a plain click still moves the cursor as today; the modifier-click is a mouse door, additive |
| Selection bar | appears with a selection | Print selected, Export selected, Mark Optional and Mark Regular (vouchers only, through the existing `markOptional.ts` Y/N and one undo step), and a **selection total**: "3 selected . Amount 4,52,000.00" for the first additive money column shown | counts only rows on screen; never deletes or edits a voucher; Print and Export keep finding rows by the `data-vid` attribute (the attribute name stays, the value is the `rowId`) |

Esc stacking: a popover or panel closes before the report's own `Esc`; the report never pops the screen while one is open (a recipe presses `Esc` in each state).

### 5.5 Slices, in order, with exit gates (effort [J])

| Slice | What | Effort | Exit |
|---|---|---|---|
| **S0** | **Spike and groundwork.** Measure first render and key response of the flat Day Book at 5k, 25k and 100k rows with the Standard layout, 15 and 20 columns, in the real app under the harness (`--cpu-factor` for an old PC); time the catalogue accessors per row and the lazy indexes with the perf mirrors; move the sort primitives to `core/sortRows.ts`; **fail-closed unknown operator** in `core/reportFilter.ts` with its assertion. **Rule the numbers must meet [J, to be ratified]:** the Standard layout costs at most 1.10 times today's at 25k rows, and 15 columns at most twice Standard. If not met, A2 still ships (hidden columns cost nothing) but windowing with model-driven Print, CSV and text filter becomes the first A3 item and the Columns panel shows a hint above 5,000 rows and 12 columns | 2 | numbers in the final summary (the orchestrator writes `docs/PERF.md`); the reader change green |
| **S1** | **Catalogue, context, `listQuery`, `ListTable`; port the four families.** Includes the sticky-header scroller fix and the `rpt.daybook.find` mirror row retired by the `ledgersById` lookup | 7 to 9 | `golden.mjs` compare is **identical** for all four families; `reportfields`, `listquery`, `gridsort` suites green; 478-screen sweep unchanged |
| **S2** | **Layout, Columns panel (`Alt+M`), header menu, Shift+click, sort in layout, persistence, `layout` in saved views, ViewsTitle label, print-fit hint** (`fitPlan` already chooses portrait, landscape or a scaled landscape; the panel shows "prints landscape at 78%") | 6 to 7 | recipes `columns-picker`, `header-menu`, `columns-views`, `columns-keyboard`, `columns-print`; `listlayout` suite; registry entry flipped `live: true` |
| **S3** | **Quick filter popover**, editable chips, header funnel, smart typing, Advanced wording; the new operators written but **off** behind the constant (D10) | 4 to 5 | recipe `quickfilter-bar`; `grid-quickfilter` suite; all Wave A filter suites green; `rowfilter.mjs` updated |
| **S4** | **Selection**: id-keyed marking on all four families, modifier-click, selection total, bar actions | 2 to 3 | `grid-selection` suite; recipe `grid-selection`; Mark Optional undo still one step |
| **S5** | Fix pass after the review lenses (section 9) | 3 | every confirmed finding fixed or recorded |

Total 24 to 29 builder-days. **Gate 1** takes S0 and S1; **Gate 2** takes S2 to S4.

### 5.6 Totals, Print and Export (the rules, short)
Footers are computed from the filtered rows by the report, never from printed text; `total: 'sum'` only for an additive measure of the row's own object and **never** on a column read through a relation; a non-additive cell in a total row is blank, never 0; sort, hide, reorder and resize never change a total; a filter does and says so. CSV export remains the cell's display text in A2 (a money cell leaves as "1,23,456.00"); raw numbers and ISO dates are an A3 item with the model-driven export.

---

## 6. B-dash

### 6.1 What a person gets (`zoho-dashboard.md` section 0 and 3)
A home that answers four questions without a click (who owes me and how late, what do I owe, what is my cash doing, am I making money), on the right clock (balances **as at today**, a switch for period end), one period control for the whole board (`F2` and presets), charts with table twins, every number opening the report it came from, and, new in this plan, **customise mode on the board itself with a widget catalogue** and list widgets built from a saved view.

### 6.2 The widget catalogue in A2

Ids already shipped are kept; "kit" means rebuilt on the chart kit. Defaults are `zoho-dashboard.md` 3.4's 12; a saved layout is untouched and an added widget arrives **off**.

| id | Widget | Form | Source | A2 slice | Needs |
|---|---|---|---|---|---|
| `receivables`, `payables` (kept) | Receivables, Payables | card: headline, current/overdue/unapplied stacked bar, four ageing steps, `+ New`; S tile | debtors or creditors balance at as-at (`dashIndex`); one lazy bills pass | S4 | accounting; bill-wise for the split |
| `cash` (kept) | Cash and bank | account list with balance and "N not reconciled"; S tile | `dashIndex` plus bank-date test of `brsPosition` | S4 | accounting |
| `cashflow` (new) | Cash flow | two-series columns, Opening to Closing headline | `dashIndex` per-voucher net on cash and bank; opening and closing cash | S4 | reports |
| `incomeExpense` (new) | Income and expense | two-series columns, headline Net profit, reconciling "Change in stock" line | `dashIndex` buckets; headline from `plFigures` | S4 | reports |
| `topExpenses` (new) | Top expenses | ranked bars, share %, "Other" | `dashIndex` | S4 | reports |
| `profit` (new) | Net profit | S tile | `plFigures` | S4 | reports |
| `trend`, `topCustomers`, `topItems` (kept) | Sales and purchases, top lists | on the kit | existing engines | S4 | accounting, inventory |
| `recent` (kept, "Latest vouchers") | Latest vouchers | list; `date <= today` (fixes G11) | change log tail | S4 | accounting |
| `stockAlerts`, `stock`, `tdsPayable`, `sales`, `purchases`, `overdue` (kept) | tiles and list | unchanged engines, honest foot dates | existing | S3 (as-at stamping) | as today |
| `gstPayable` (kept) | GST payable | **words corrected only**: the foot says "for <the open period>", not "current month" (D8) | unchanged | S3 | gst |
| **`vw-<id>` (new, instance)** | **List from a view** | table: the view's columns (up to 6), 5, 8 or 12 rows, "8 of 23" count | `listQuery` over a saved view or a Wave A default filter | S5, after B-grid S1 | per the view's report |
| presets of the above | **Overdue invoices** (Bills Receivable, Overdue), **Bills due this week** (Bills Payable, Due this week), **Today's vouchers** (Day Book, Today) | the same widget, a ready-made config in the gallery | `systemViewsFor(reportKey)` | S5 | bill-wise for the bill lists |
| `today`, `setup` (new) | Today strip, Get started | strips, L only | existing `useToday`, `onboarding/progress.ts` | S6 **stretch** | open |
| `quick` (kept, off by default) | Quick Entry and Reports | buttons | static | S3 | per button |

The Overdue list is therefore **not a bespoke widget**: it is the first preset of the view-widget engine, which is the point of A2-7.

### 6.3 Customise mode and the widget catalogue (the part the research docs left thin)

Evidence: Zoho's setup page has a left pane of **Predefined Panels** and **Report Panels** (grouped by category with a short description), a hover "Create Panel" icon that adds one, drag to reorder, resize only for report panels, and nothing is applied until "Save Changes"; the page does **not** mention search, previews or thumbnails [W, custom-dashboards page]. Comsky already beats that on machinery (free on every plan, per operator, keyboard-complete, company standard). A2 adds what is missing.

| Piece | Design |
|---|---|
| Entry | `F12` or the Customize button (unchanged): the board enters **edit mode** (`data-editing`), the existing docked panel opens |
| Panel tabs | **On this board** (today's list: checkbox, blurb, S/M/L, move, R reset, C company, X remove: keys and meanings **unchanged**, the superset rule) and **Add widgets** (new) |
| Gallery (Add widgets) | grouped sections (Money, Profit, Sales and stock, Taxes, Activity, **Lists from your views**, Setup); each card has the name, one line, a form tag (Tile, Chart, List), the sizes it offers, a "needs" badge (company feature, right) and a **static schematic thumbnail** (no live figures: a hidden widget must not be computed). A search box matches label and aliases (Tally and Zoho words: "debtors", "udhaar", "aging"). `Enter` or the Add button puts it on the board **off to the side of nothing**: after the current widget, on, default size |
| "Lists from your views" | the presets above, then **From a saved view...** (a picker over the operator's personal and company views of the four families); limit 5, 8 or 12 rows; at most **8 view widgets** and **24 widgets** on a board |
| On-board edit | every widget gets a drag handle, a focus ring and a one-line toolbar (hide, size, settings). **Pointer drag** reorders with pointer events (not the mouse-only HTML drag API), a previewed drop slot, `Esc` cancels. **Document order is visual order**: no `grid-auto-flow: dense`, so Tab order never lies |
| Keyboard model | `zoho-dashboard.md` 3.7 item 4 verbatim: `Space` grabs (announced "Grabbed, position 4 of 12"), arrows move by measured geometry, `Shift+Left/Right` resize one step (stops at the ends), `H` or `Delete` hides, `S` `M` `L` set the size, `Esc` cancels or drops; every move and resize announced through a live region |
| Per-widget settings (`cfg`) | a small sanitised object on the stored item: Top N, "include purchases", the list widget's `src` and `limit` (7, C3). Schema: `{v: 1}` stays `v: 1` unless an item carries `cfg` (an older reader ignores `v` anyway) |
| Apply semantics | **live apply** to the personal layout (as today), plus **Cancel changes**, which restores the layout in force when edit mode opened. Zoho's batch "Save Changes" is deliberately not copied: a live board that can be cancelled is faster, and `Ctrl+Z` stays the books' undo, not the board's |
| Who sets what | personal (this device, `tw-dash-v1`) beats company (`Company.dashboard`, an administrator writes it with `C`) beats built-in, as today. **Rule:** making a board the company layout is refused, with the widget named, if it holds a view widget bound to a **personal** view (other operators would see a broken widget); the message says to share the view first |
| Rights | `visibleItems` gains a rights argument; a widget the operator cannot open is not drawn, not listed in the gallery and **not computed** (`zoho-dashboard.md` 3.12). The screen-level area stays `reports` unless D2 is yes |

### 6.4 Engine, period and charts (decisions only; the design is `zoho-dashboard.md` 3.3, 3.6, 3.9)
- **`core/dashIndex.ts`**: one pass building a ledger-by-bucket debit and credit index plus the per-voucher cash net (prototype 55 ms at 250,000 vouchers [R, V by that doc]); bills (153 ms) and stock (124 ms) stay separate lazy passes, memoised on array identity, each in its own task, started only for a visible widget. **Reconciled to the engines at 2k, 50k and 250k** (closing balances, period totals, monthly trend, party totals, cash net against an independent oracle).
- **As at**: `asOn = min(period.to, max(today, period.from))`, switch Today (default) or Period end, printed on every balance widget; a midnight timer re-reads `today` and is cleaned up on unmount.
- **Period**: the open period (`st.setPeriod`), presets and `F2`; a chip "Not the financial year: reset" whenever the window is not the year; buckets by `bucketsFor(from, to)` (day, week, month, quarter).
- **Gating**: `useIsTop()` stops a stacked Dashboard recomputing on every save (G13), the cheapest big win; a stale flag recomputes when it returns to the top.
- **Chart kit** (`ui/dash/charts/*`, plain SVG): `Columns`, `StackBar`, `RankBars`, `Sparkline`, `ChartFrame`, each with a table twin, a tooltip that never gates a value, keyboard focus per bucket; pure helpers in `core/dashScale.ts`. Series colours are the four validated `--viz-*` slots (the existing `--tone-*` tokens stay KPI stripe accents); a drift test fails if a `--viz-*` value changes without the validator output recorded beside it.
- **Cash flow rule**: per-voucher net on cash and bank (own-account transfers vanish). The widget states it in a line under the chart. Whether the **report** changes too is D3; A2's default is "widget only, footnoted".
- **Income and expense** is accrual only and labelled; headline from `plFigures`, which A2-0 extracted, so it equals the Profit and Loss screen by construction.

### 6.5 Slices, in order (effort [J]; B-dash may fan out into the three lanes of `zoho-dashboard.md` section 4, one owner and one gate)

| Slice | What | Effort | Exit |
|---|---|---|---|
| **S0** | **cfg-preserving layout reader** (`resolveLayout`, `serialiseLayout`, the extras path keep every item field including `cfg`) and `useIsTop` gating of the existing Dashboard; both are first commits of the train (A2-6) | 1 | `dashboard-custom` extended: unknown item fields survive a round trip through both paths |
| **S1** | engine lane: `dashIndex`, `dashBuckets`, as-at rule, `dashScale`, perf rows, reconciliation suite | 3 to 4 | index equals engines at 2k, 50k, 250k; rows in `scripts/perf/*.dash.*` with reference lines |
| **S2** | chart lane: the kit, `--viz-*` tokens (light-only block and dark override), validator output recorded, table twins | 3 to 4 | `dashscale` and `viz-tokens` suites; chart recipes in all four combinations |
| **S3** | board shell: header band (greeting, company, window and as-at in words, Period, As at, Customize, `+ New` from `pageCreates` replacing the hard-coded button), `ButtonPanel` (F2 Period, F12 Customize, Esc Quit), midnight timer, drill helper `drillTo(screen, {period?, filter?})` (7, C4), honest foot dates on every kept tile, the GST tile's words, tiles become real buttons with names (G9), default layout v2 | 4 to 5 | `dashboard-custom` rewritten **deliberately** (the shipped default was pinned: "the default IS the dashboard the app always drew", G16); recipes `dash2-default`, `dash2-period`; zero console errors |
| **S4** | the Phase-1 widgets of 6.2 in the order **Receivables, Payables, Cash and bank, Cash flow, Income and expense, Top expenses**, then the kit rebuilds, Latest vouchers fix; first-run and empty states; per-widget rights | 5 to 6 | `dashboard-v2` reconciliation per widget (segments sum to the headline; Income and expense equals Profit and Loss; a Contra between two banks shows zero); recipes `dash2-empty`, `dash2-restricted`, `dash2-table-twins` |
| **S5** | customise mode, gallery, pointer drag, keyboard grab/move/resize, `cfg`, **view widgets** (needs B-grid S1's `listQuery`; if B-grid slips, S5 ships everything else and the view-widget family waits for Gate 3) | 7 to 9 | recipes `dash2-edit-pointer`, `dash2-edit-keyboard`, `dash2-gallery`, `dash2-viewwidget`; layout v2 reader and preserver tests |
| **S6** | stretch: Today strip, Setup card | 2 | only if Gate 2 is green with time left |
| **S7** | fix pass after the review lenses | 3 | every confirmed finding fixed or recorded |

Total 26 to 32 (+2 stretch). **Gate 1** takes S0 to S3; **Gate 2** takes S4 and S5.

### 6.6 Deliberate changes to pinned behaviour (flagged, not slipped in)
1. The built-in default board changes (12 on by default, Quick Entry off): `dashboard-custom.assert.ts` is rewritten, listed in the gate output, and saved layouts are untouched by the append-off rule (D4).
2. Balances are stamped with **today**, not the period end (finding 1): every tile's text changes in the screenshots.
3. If D2 is yes: `SCREEN_AREA.dashboard = 'open'` and the Data Entry "blocked from Dashboard" line in `security-access.assert.ts` is rewritten with the per-widget rights table beside it.
4. If D3 is "report option": the Cash Flow Summary report changes its default (own-account transfers not counted) with an F12 option to see the leg view; `financial.tsx` edit in the integration window.

---

## 7. Contracts

| # | Contract | Producer | Consumer | Exact form | Proof |
|---|---|---|---|---|---|
| **C1** | **Field ids are forever, and typed** | B-grid (`core/reportFilterFields.ts`, `core/columnSets/*`) | B-dash drill filters and view-widget presets; Wave A saved views | `fid(family, id)` returns the id only if the family's catalogue has it with the kind the caller expects; Wave A ids keep kind and order; new ids append | `reportfields`: every Wave A id exists with the same kind; `a2-contracts`: every filter B-dash builds survives `sanitiseFilter` with **no condition dropped** |
| **C2** | **`listQuery`: the one row-and-column path** | B-grid, `core/listQuery.ts`, pure | `ListTable` (B-grid) and view widgets (B-dash) | `listQuery({ report, filter, layout }, env, { limit?, ledgerId? }) -> { cols, rows: {id, cells, text}[], matched, total, totals, action: ViewScreen }`; `env` is plain arrays plus the period and `today` (no store); with `limit` it selects the top N in O(n log k); rights are **not** judged here | `listquery`: for each family the result's rows equal what `ListTable` draws (same code path); `matched` and `total` equal the filter engine's; the Overdue preset equals the Bills report rows with `overdue > 0` |
| **C3** | **View widget config** | B-dash, `core/dashListWidget.ts` | B-dash widget, gallery, layout | `cfg: { v: 1, src: { t: 'system', report, id } \| { t: 'view', viewId }, limit: 5 \| 8 \| 12 }`; resolved **live** (a widget never copies a view); a missing view shows "This view is not available (deleted, or not yet synced)" with Remove; rights judged by `screenDenial` on the view's own `screen`; a company layout may not hold a personal view (6.3) | `dashboard-v2`: sanitiser (hostile cfg, unknown source type, limit out of set, 9th view widget); resolution; the company rule |
| **C4** | **Drill with a filter** | B-dash `drillTo` | Wave A's filter bridge | `writeReportFilter(cid, reportKey, scope, filter)` then `setPeriod` when a window is given, then the existing push; the filter is built from `systemViewsFor(reportKey)` or `fid(...)` conditions, never hand-typed field names; the filter and period are **visible on arrival** (bar, badge, reset chip) and cleared by one click | `dashboard-v2`: each drill row of `zoho-dashboard.md` 3.8 lands on a report whose live filter equals the intended one; recipe `a2-flow` |
| **C5** | **Saved view schema** | B-grid (`SavedView.layout`) | B-dash reads only | layout is optional, sanitised on read, ignored by an older client; B-dash never writes a view; it opens one through `{ s: 'reportView', viewId, target }` | `savedviews-filter` extended |
| **C6** | **One clock for "overdue"** | Wave A `core/dates.ts ageingAsOn`; B-dash `core/dashBuckets.ts asOn` | the Bills report, the Days overdue column, the Overdue widget | the Bills report and the new "Days overdue" and "Status" columns use `ageingAsOn(period.to, today)`; the dashboard's `asOn` equals it whenever today is inside the window or the year is closed; a window wholly in the future is the one documented difference (`asOn` is the period start) | `a2-contracts`: Overdue widget total equals the sum of Bills report rows with `overdue > 0` for the same date, on the sample company and a 50k synthetic, across the four window cases |
| **C7** | **Registry and chords** | A2-0 pre-seeds `columns` | B-grid flips `live` | B-dash adds none; neither builder quotes a registered chord outside its handler | `zoho-scaffold` unmodified and green |
| **C8** | **Tokens and CSS** | B-dash owns `--viz-*` in `styles.css`; each builder owns its own css files | none | B-grid uses existing tokens only | the Wave A no-hex scan |
| **C9** | **Release-ahead readers** | B-grid: fail-closed unknown operator (`core/reportFilter.ts`). B-dash: cfg-preserving layout reader (`core/dashboardLayout.ts`) | older clients and company-synced data | both land in the **first commit(s) of the train**; the new operators (`FILTER_OPS_V2`) and the first stored `cfg` are only produced after the deploy order is confirmed (D10) | `reportfilter` and `dashboard-custom` extended |
| **C10** | **Perf and purity** | each builder | the harness | `core/listQuery.ts`, `core/dashIndex.ts` and everything they import pass the harness's forbidden-import scan; reference rows `grid.*` and `dash.*` exist; mirrors move with the code | `perf --enforce` green, no mirror drift |

The only cross-builder call graph: **B-dash S5 imports `core/listQuery.ts`, `core/reportFilterFields.ts` (the `fid` helper and `systemViewsFor`) and reads `ui/reportViews/viewStore.ts`**. Nothing flows the other way.

---

## 8. Assertion plan

Every new suite lists its deliberate mutations in the builder's summary and each is killed. All suites are auto-discovered (`scripts/assert/run.mjs` bundles every `*.assert.ts`).

### 8.1 B-grid suites (pure unless noted)
| Suite | Proves | Kills |
|---|---|---|
| `reportfields` | ids unique and typed; locked column first; `now` columns carry "(current)"; party labels prefixed; **every Wave A id exists with the same kind and order**; the catalogue's Amount, Days overdue and Ageing equal `amountCells`/`registerAmount`/`daysOver`/`ageingBucket` on every sample row; `total: 'sum'` only on additive own-object measures, never on a relation column (a blank, never 0); no getter returns more than one value per row (a voucher with 5 lines and 3 cost centres is still one row); running-balance style columns are keyed by id; indexes linear on a 250k synthetic set (reference time, not gated) | a summed credit limit; a blended GST rate; a relation column that repeats a row |
| `listlayout` | sanitiser (corrupt blob, wrong types, duplicates, 21 columns, widths out of range, a sort naming a hidden column, `pin` ignored); resolver (unknown id dropped, locked first, Standard fallback); a source scan that every storage access is in try/catch and that **no live filter is written to storage** | a column set that can hide a row; a throw on a private window |
| `listquery` | C2 as above, plus: optional and non-posting vouchers never reach any family; rows with and without a layout have the same ids and order for the same sort; top-N equals the head of the full sort; 250k reference time | the widget and the report disagreeing |
| `gridsort` | two-key sort, blanks last in both directions, natural number order, stable ties; `core/sortRows.ts` equals the old `columnSort.ts` results on a property test | a regression for the six `useColumnSort` callers |
| `grid-quickfilter` | the smart-value parser (`>1L`, `<=50k`, `1L-5L`, bare words, date words); operator lists by kind; chip model; the 12-condition message; the fail-closed reader (an unknown operator keeps the condition and matches nothing); with `FILTER_OPS_V2` off the new operators are not offered | a filter that widens |
| `grid-keys` | source scans: `Alt+M` appears only in `ColumnsPanel.tsx` (displayed elsewhere through `chordOf`); no `useKeys` handler names `active` toggling; popover and panel handlers are mounted only while open; the header caret and chips use `ownActivate`; `combo()` of Mac Option+M and of AltGr+M (`Ctrl+Alt+M` is **not** `Alt+M`) | a stolen key; a dead Enter |
| `grid-selection` | marking is id-keyed and intersected with the visible ids on all four families; a sort, a filter and a period change never change what an action sees; the selection total sums only additive columns of marked visible rows; Mark Optional changes one flag (the existing guarantee) | an action on a hidden row |
| **Extend** | `savedviews-filter` (a view with and without `layout`; an older blob; company scope), `reportfilter` (fail-closed), `reportfilter-rows` (totals contract with columns), `zoho-scaffold` (the `columns` entry), `visual.assert` (Standard unchanged; the new header row in both densities and schemes) | |

### 8.2 B-dash suites
| Suite | Proves |
|---|---|
| `dashboard-v2` | index equals `closingBalances`, `periodTotals`, `monthlyTrend`, `rankParties` at 2k, 50k, 250k and on the sample company; **reconciliation per widget** (Receivables segments sum to the headline, which equals the debtors group balance; Income and expense headline equals Profit and Loss with stock, without, with a stock change and with a loss; Cash flow with a Contra between two banks shows zero; Top expenses shares sum to 100 % with Other); the as-at rule and `bucketsFor` at every boundary (today before, inside and after the window; a closed year; first and last day; leap day; a year across a calendar year; one-day and 31-day windows); layout v2 (unknown ids and unknown item fields kept through both save paths, a size outside a widget's list falls back, hostile and oversized stores never throw, the default has the 12 ids in order, every widget's drill target exists in the union, router and `SCREEN_AREA`); **view-widget cfg** (C3); rights (the visible set for a clerk, an accountant, a stock keeper and a full-rights user equals the menus'; security off shows everything); a hidden (stacked) Dashboard computes nothing |
| `plfigures` | in A2-0 (above) |
| `dashscale` | nice ticks for all-zero, one bucket, negative and mixed ranges, a 366-bucket day window, 0.01 values; the label-fit rule |
| `viz-tokens` | the `--viz-*` values match the recorded validator output in both schemes |
| Rewrites | `dashboard-custom.assert.ts` (the pinned default, deliberately; the existing 794 lines stay except the default and the tile text), `security-access.assert.ts` (only if D2 is yes) |

### 8.3 Cross-builder suite (owner: orchestrator at Gate 2, written by B-dash)
`a2-contracts.assert.ts`: C1 (no filter dropped), C2 (the Overdue preset equals the Bills rows), C6 (one clock), C4 (each drill's live filter), and a source scan that no file outside the ownership manifest changed (4.6).

### 8.4 Recipes (`npm run shots`, never the shared Browser pane; each in light and dark, compact and comfortable, zero console errors)
B-grid: `columns-picker` (Day Book, `Alt+M`, search "gst", add Party GSTIN, close, the column is there, Print shows it, widths absent), `header-menu`, `quickfilter-bar` (including Esc stacking and `Alt+F12`/`Ctrl+F12` landing in the popover), `columns-views` (save as a company view, recall on a second profile), `columns-keyboard` (the same flow with no mouse), `columns-print`, `grid-selection`, plus the sticky-header geometry probe after a sideways scroll. B-dash: `dash2-default`, `dash2-period`, `dash2-empty`, `dash2-restricted`, `dash2-table-twins`, `dash2-edit-pointer`, `dash2-edit-keyboard`, `dash2-gallery`, `dash2-viewwidget`. Shared: `a2-flow` (save a view on Bills Receivable with Overdue and added party columns, add it as a dashboard widget, drill from the widget, the report shows that view's filter and layout), and `golden.mjs` in compare mode in every run.

### 8.5 Gates
**Gate 0** (A2-0, section 3). **Gate 1** (S0 and S1; B-dash S0 to S3): tsc 0, build, assert green, perf `--enforce` (no mirror drift), shots `--strict` over all screens, `golden.mjs` identical, untouched screens byte-identical. **Gate 2** (S2 to S4; B-dash S4 and S5): the same, plus every recipe above. **Final**: the review lenses (section 9), the fix passes, the live data bug-hunt, then the full sweep again.

---

## 9. Review lenses

Wave A and Phase 24 used four to six adversarial reviewers, each with a lens, then a skeptic verifier per finding; A2 does the same on the finished Gate 2 tree. **Plus one step review cannot replace:** the Phase 23 accuracy bugs were found by *viewing reports on real data*, so a **live bug-hunt** opens each ported report and each dashboard widget on the sample company and on a 50k synthetic company, compares Standard layouts with the goldens, and compares every dashboard figure with the report it opens.

| # | Lens | For | Attack with |
|---|---|---|---|
| L1 | **Accounting integrity** | grid | a column that changes a row or total; a summed relation column; the Amount mirror; an Optional voucher through a filter plus a column; Ledger Vouchers `Opening + shown + not shown = Closing` under every sort; snapshot versus current (edit an item's GST rate, read the column); Mark Optional through a selection after a sort |
| L2 | **Keyboard and focus** | both | `Alt+M` in a field, behind an overlay, on a statement; Esc stacking (popover over panel over report; edit mode over the board); Enter and Space on caret buttons and chips; a hidden stacked screen's handlers; any global handler toggling `active`; AltGr layouts; Tab order after a drag |
| L3 | **Persistence and compatibility** | both | mixed client versions (an older reader on a view with `layout`, on a filter with `isEmpty`, on a layout with `cfg`); company views synced to a second device; a deleted view behind a widget; `master-resave` for any new key; storage disabled; a stale column id; a corrupt blob |
| L4 | **Print, Export, DOM coupling** | grid | a hidden column in Print and CSV; widths in the print clone; the Total label's cell; the `/` text filter over added columns; `data-vid` on Bills rows; sticky header after a sideways scroll; `fitPlan` at 20 columns |
| L5 | **Performance** | both | the spike thresholds; accessor cost per row; lazy indexes linear; memo churn in `useRowFilter` and the sort store; a stacked Dashboard recomputing on a save; one pass versus several; timers cleaned up on unmount |
| L6 | **Visual, density, accessibility** | both | four combinations, 1280 and 1024 wide, no horizontal page scroll; contrast; forced colours; focus rings; names on tiles and carets; no hex; the light overlay rule (a base `:hover` form silently lost under the `html:root[data-scheme='light']` prefix) |
| L7 | **Number truth** | dash | every widget against its report on the same window and as-at; Receivables segments; Cash flow rule; the Income and expense reconciling line; bucket boundaries; a window in the future; a company with no bill-wise; a negative cash balance |
| L8 | **Rights and gating** | dash | each widget for a clerk, accountant, stock keeper; features off; a view widget over a report the operator cannot open; the company-layout write guard; D2 either way |
| L9 | **Layout engine and compatibility** | dash | the layout round trip through old and new code; instance ids; `cfg` hostile input; the 8 and 24 caps; a company layout holding a personal view; the append-off rule for new widgets |
| L10 | **Charts** | dash | label collisions and clipping; negative values; future buckets; the table twin carries every number the tooltip does; colour never alone; both schemes |
| L11 | **Seams and ownership** | cross | any file changed outside the manifest; any duplicate implementation of a filter, a sort or a column on the dashboard; `listQuery` and `ListTable` drifting; a registered chord quoted elsewhere |

---

## 10. Sequencing, effort, commits

```
A2-0 (serial, 3-4d) ──► Gate 0 commit
        ├─ B-grid  S0 ─ S1 ──────────┐ Gate 1 commit ─ S2 ─ S3 ─ S4 ─┐ Gate 2 commit ─┐
        └─ B-dash  S0 ─ S1 ∥ S2 ─ S3 ┘                  S4 ─ S5 (needs grid S1) ┘        │
                                                                                          ▼
   integration window (single writer by turn): grid done in financial.tsx -> dash Cash Flow option (if D3)
   -> dash access.ts line (if D2) -> orchestrator docs, beyond rows, PERF.md -> review lenses -> fix passes -> final sweep -> commit
```
- **Critical path:** A2-0, B-grid S0 and S1, then B-dash S5's view widgets. B-dash S1 to S4 (16 to 20 days) outlast B-grid's S0 and S1 (9 to 11), so the dependency edge does not stall B-dash.
- **Release-ahead:** the Gate 0 and Gate 1 commits carry the two readers of C9; the owner decides when they are deployed (the VM recipe is in `docs/HANDOFF.md` section 2); the features that produce new stored shapes follow (D10).
- **Totals [J]:** A2-0 3 to 4, B-grid 24 to 29, B-dash 26 to 32 (+2 stretch): about 53 to 65 builder-days across two owners. Calendar time depends on how many lanes each owner fans into; Wave A and Phase 24 ran three to eleven parallel lanes, so I give no calendar estimate I cannot support.
- **Docs the orchestrator updates at the end:** `CLAUDE.md` (a Phase 26 block and its gotchas), `docs/HANDOFF.md`, `docs/PERF.md` (the P0 table and the new reference rows), `docs/parity-data.json` (two `beyond` rows, never the parity percentage), `docs/research/ROADMAP-ADDENDUM.md` (the A3 list).

---

## 11. Risks specific to A2

| # | Risk | Mitigation |
|---|---|---|
| R1 | Wave A moves after this plan was written | A2-0 starts from the commit and corrects the tables; builders read A2-0's output, not this plan's file names |
| R2 | The Standard layout drifts and a Tally switcher notices | goldens (0.1), `ListTable` keeps the element tree, the 478-screen sweep, `visual.assert` |
| R3 | A column makes a total plausible and wrong | `total: 'sum'` rules, blank not 0, L1 |
| R4 | Rendering cost at 250,000 vouchers (no windowing exists in `src/`; columns multiply cells) | S0 measures and sets the A3 rule; hidden columns cost nothing; the hint above 5,000 rows and 12 columns |
| R5 | Two builders edit one file | A2-0 seams, the ownership manifest, the checker at every gate, the perf and CSS file splits |
| R6 | The dashboard grows a second filter or column engine | A2-7 and C2; L11 |
| R7 | Mixed client versions widen a filter or strip a widget's cfg | C9; release order; D10 |
| R8 | A new keyboard collision | `Alt+M` audited and asserted; F2 and F12 predate the registry; popovers own keys only while open |
| R9 | The dashboard's period change surprises a later report (the drill sets the **global** period and a live filter) | the reset chip, the always-visible Filtered bar and badge, and the fact that a live filter never persists to storage |
| R10 | Reversing the pinned Data Entry test | D2 default is "not in A2"; the widgets carry their rights regardless |
| R11 | Effort and thresholds are my estimates | marked [J]; S0's numbers replace them |
| R12 | View widgets turn into a mini report builder | no grouping, no aggregation, no custom columns beyond the view's own layout, at most 12 rows, 8 per board |
| R13 | Perf mirror drift fails `--enforce` after a refactor | rule 10; mirrors split per owner in A2-0 |
| R14 | Charts: clipped labels and tooltips are easy to get wrong | label-fit rule, `dashscale` suite, a screenshot review of every chart at three sizes in both schemes (the validator checks colour, not layout) |

---

## 12. Owner decisions (default taken if no answer, so no builder waits)

| # | Question | Recommendation and default |
|---|---|---|
| D1 | Do the **Parties and Items lists** (the strongest Pipedrive fit, a new screen each) go in A2 or first in A3? | **A3 first.** Default: A3. If yes, A2-0 registers two stub screens the way Wave A did, and B-grid builds them last |
| D2 | Open the Dashboard to every operator with each widget judged by its own right (a Data Entry operator then sees the accounting widgets; today they are refused the whole screen)? | Yes, in the end. Default: **not in A2**; widgets carry the rights; one line and one rewritten assertion flip it |
| D3 | Does the Cash Flow Summary **report** also stop counting transfers between your own accounts (with an F12 option to see the leg view)? | Yes (the widget and its drill then agree). Default: widget only, footnoted; the report in A3 |
| D4 | Accept the new default board (12 widgets, Quick Entry off) for every un-customised user? | Yes; saved layouts are untouched |
| D5 | "Match all / Match any" wording in Advanced, with and/or kept elsewhere? | Yes. Default yes |
| D6 | Which party fields matter most on a list of invoices, and is a salesperson needed (there is none on a voucher today)? | Default: offer the whole set in 5.2; no salesperson (it would be a new custom voucher field) |
| D7 | Company-wide default columns for a report (a synced `Company` field)? | Default: not in A2; a company **view** may carry a layout, which is the same effect by choice |
| D8 | The GST tile says "current month" but builds the whole open period | Default: correct the words now; the real "GST due" card (filing period, statutory dates, QRMP state table) is A3 and needs the owner's QRMP category |
| D9 | Pinned columns (Zoho has up to two [Ws]; Pipedrive documents none) | Default: no in A2; the layout reserves `pin` |
| D10 | Deploy order: may the fail-closed filter reader and the cfg-preserving layout reader be deployed **before** the features that produce the new stored shapes? `FILTER_OPS_V2` stays false until you say so | Yes |
| D11 | Selection: confirm Print, Export, Mark Optional/Regular and a selection total as the whole of "bulk actions"; and whether "Reminder letters for the selected parties" on Bills (about 2 days) belongs in A2 | Default: the four, not the letters |
| D12 | **View widgets** on the dashboard (Zoho's report panels, built on saved views)? | Yes (it is the join between the builders). If no, drop the Overdue list from A2 or build it bespoke (2 days) |
| D13 | Gate a 250,000-voucher dashboard row in `perf --enforce`? | Default: reference rows only in A2; propose a gate when the numbers settle |

Decided in this plan, veto only if you disagree: `Alt+M` (A2-1); the popover opens on `Alt+F12`/`Ctrl+F12` (A2-2); recalling a view saved before A2 resets columns to Standard (5.3); column 0 is locked (5.2); no new screens (A2-3); the dashboard's Cancel changes instead of Zoho's Save Changes (6.3).

---

## 13. What this plan did not do

- It did not run, build or edit anything; it wrote this one file.
- It did not open a live Pipedrive or Zoho account (0.2). Row selection, a column header hide or pin menu and a footer total remain **[unverified]** for Pipedrive; the quick filter's "ten conditions" and "funnel" gestures are from the earlier research [R].
- It did not verify whether the register's month drill and flat level share one listing component (5.2 row 2); A2 confirms it.
- It did not research the Income-tax Act 2025 TDS dates, the QRMP state categories or current GST extension notices (deferred with the GST due card).
- It did not time anything; every number tagged [J] or taken from `zoho-dashboard.md` section 2.3 is a plan input, not a result.

---

## 14. Sources

**Fetched this session [W]** (paraphrased, none reproduced):
- Pipedrive, bulk editing and filtering (selection mechanics not described; actions by item type; 100 activities at a time): https://support.pipedrive.com/en/article/bulk-editing-and-filtering
- Pipedrive, list view (toolbar, gear column picker, inline edit; no header menu, row selection or footer described): https://support.pipedrive.com/en/article/list-view
- Zoho Books, custom dashboards (Predefined Panels and Report Panels lists, Create Panel icon, drag to reorder, resize only report panels, Save Changes; no search or previews described): https://www.zoho.com/us/books/help/home/custom-dashboards.html

**Search summaries only [Ws]:**
- Zoho Books, list column customisation and up to two pinned columns (a Sales Receipts help page): https://www.zoho.com/au/books/help/sales-receipts/manage.html
- Zoho Books, custom views with Available and Selected columns: https://www.zoho.com/books/help/settings/customization/custom-views.html

**Inherited from the research documents [R]** (their URL lists are in `pipedrive-list-ux.md` section 15 and `zoho-dashboard.md` section 10; not re-fetched): Pipedrive's customising-columns, filtering, quick-filters, sorting-by-multiple-columns, default-columns, exporting, summary-box, custom-fields, Insights combined-data and Filters API pages; Zoho's Home, India dashboard, KB overdue, release notes, what's new and roles pages.

**Our own, read this session [C, V]:** `CLAUDE.md`; `src/core/{reportFilter,reportFilterFields,reportRows,savedViews,dashboardLayout,shortcutRegistry}.ts`; `src/ui/{rowFilter,reportActions,pageHeader,quickCreate}.ts*`; `src/ui/header/{FilterBar,FilterPanel}.tsx`; `src/ui/reportViews/{apply,bridge,Launcher}.ts*`; `src/ui/reports.tsx` (Day Book, Ledger Vouchers), `src/ui/reports/{columnSort,registerLevel,selectionBar,outstanding,financial}.ts*`; `src/ui/modern/{Dashboard,dashWidgets,dashStore,useDashLayout,dashEditRequest}.ts*`; `src/types.ts` (Voucher, Ledger, SavedViewStored, DashItemStored); `src/core/billwise.ts`; `scripts/assert/{run.mjs,zoho-scaffold.assert.ts}`; `scripts/perf/{run.mjs,budgets.mjs,mirrors.mjs}`; `scripts/shots/README.md`; `src/ui/selection.ts`; greps for `Alt+C`, `Alt+M`, `useColumnSort`, `useVoucherSelection`, `useRowFilter`, `ButtonPanel`, `QC_PAGES`, `dashboard` (2026-10-20).
