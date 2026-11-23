# Back-navigation audit for mouse-only use (2026-10-21)

**What this is.** An audit of how a person goes back today in Comsky Books, written for owner request item 4 ("create an option to go back in
every report or view, which is currently not there, so that a person operating through a mouse only should be able to go back into the
reports"), plus a recommendation. **No code was changed.** The only file written is this one.

**Evidence tags.** `[C]` read in this repo (file:line; the working tree on 2026-10-21 is HEAD `83d9876` plus about 157 uncommitted paths from the
parity-wave-3 build, so line numbers can drift: the symbol names are stable). `[V]` verified by me in the running app, in my own throw-away
headless Chrome (`scripts/shots` launcher, sample company opened through the welcome's T key, real CDP mouse and key events, **never Accept**).
`[W]` a page I fetched (URL given). `[Ws]` search summary only. `[J]` my judgement. `[unverified]` could not be checked.

---

## 1. Verdict

A mouse-only person is **not** without a way back everywhere, but the way back is accidental, hidden, inconsistent and, in two places, a trap.

1. **The only visible, clickable way back is the `Esc Quit` button in the right-hand F-key rail** (99 call sites in 73 files). [C, V]
   It is a Tally button, not a navigation control: it is called "Quit" (which on the Gateway means close the app), it sits at a different height on
   every screen (4th of 7 on the Day Book, 2nd of 2 on a master, 24th of 24 on voucher entry), it names no destination, and on a short window the
   voucher's copy is below the fold. [V]
2. **The 15 Gateway submenus (Create, Alter, Display More Reports ...) have no back control at all.** 15 of 15 probed: the rail is the Gateway's
   own (F2 to F11), there is no Esc button and no link in the menu box. The top bar's own "K: Company" pushes one of them over whatever you were
   doing. A mouse user is stranded there except through the left rail (which replaces the stack). [V]
3. **Dialogs and pickers trap the mouse.** Of the dialogs I opened, every one built on the shared `Box` / `.overlay` shell (Change Period F2,
   Report Configuration F12, Saved Views Alt+V, Filter rows, the voucher date, the F10 Change Voucher Type picker, New Column, Basis of Values) and
   both `ListPrompt` pickers I opened (Stock Query's item list, Share) have **no close button and ignore a click outside**, and a full-window shield
   covers the rail's Quit. Only the keyboard (Esc) leaves. There are 24 `<Box` sites, 6 hand-built `.overlay` dialogs and 17 `<ListPrompt` sites. Go To,
   the universal `+` and the `?` sheet, by contrast, close on a click outside, and the Settings key recorder has real buttons. [C, V]
4. **Browser Back (mouse thumb button, trackpad swipe, Alt+Left) leaves the app.** The app never writes a history entry, so a real "back" went
   to `about:blank` in my probe and the position in the app is lost. [C, V]
5. **The breadcrumb is plain text** (a `span`, default cursor, three real clicks did nothing), by an explicit earlier decision. [C, V]
6. **The Quit button is not the Esc key** on screens with internal state: from a Sales Register month list the key goes up to the month matrix,
   the button leaves the whole register; Stock Query and a Day Book with marked rows differ the same way. [V]
7. **Quit and Esc skip the "Leave this entry?" guard that the left rail uses.** An edited voucher alteration and a half-typed master are thrown away
   silently by Quit or Esc, while a left-rail click on the same state asks. [V]
8. **Entering from the left rail or the top bar replaces the stack**, so "back" after a rail click goes to the Gateway, not to where the person
   was; one rail click erased a three-level drill. [V]
9. **Going back loses the mouse user's place.** A mouse double-click opens a row but does not move the keyboard cursor, and the report re-reveals
   the cursor row on return: a Day Book scrolled to the bottom came back near the top (246 to 48 of 247). The keyboard path restored exactly. [V]
10. **Adjacent mouse gaps**: Group Summary rows cannot be opened by mouse, and the voucher alteration's `PgUp/PgDn Prev/Next` rail button does
    nothing when clicked. [C, V]

**Recommendation in one paragraph [J]:** add one persistent **Back** control in the **top bar, right after the hamburger**, that goes through
`core/navPlan.ts` as a new `back` move (so the dirty-leave guard applies exactly as it does for the left rail), labelled with the previous screen's
real name; hide it at the root and on gate screens; give every `Box` and `ListPrompt` a visible Close that runs the same handler as Esc; make the
rail's `Esc` buttons run what the Esc key runs; make a mouse double-click set the cursor; keep a small session trail so Back from a rail-entered
screen returns to where the person was. Esc stays the keyboard key; no new chord. Section 8 has the detail, section 9 the frozen-layout
touchpoints, section 10 the tests.

---

## 2. How a person goes back today, by mechanism

| Mechanism | Exists? | Mouse-reachable? | What it does | Evidence |
|---|---|---|---|---|
| `Esc` key | Yes, on nearly every screen | No (keyboard) | Each screen's own `useScreenKeys` handler decides: close a panel, clear marks or a filter, step up an internal level, ask "Quit?", or `st.pop()`. It is the real "back" contract | [C] `reports.tsx:735`, `financial.tsx:439`, `stockquery.tsx:71`, `VoucherScreen.tsx:3054`, `forms.tsx:265`; dozens of other handlers |
| Right rail `Esc Quit` button | On all screens except the Gateway and its submenus | Yes | `onClick: () => st.pop()` written out per screen, **not** the Esc handler | [C] 99 sites, e.g. `reports.tsx:584`, `masters.tsx:724`, `VoucherScreen.tsx:3461`; [V] 130+ screens |
| Breadcrumb in the status bar | Drawn, last 4 segments | **No**: plain `span`, `cursor: default`, no role or tabindex | Nothing. Decided on purpose: a segment click "would pop several screens and bypass each screen's own Esc guard" | [C] `widgets.tsx:258-261,273`, `left-nav-spec.md:83`, `zoho-ux-spec.md:618`; [V] three real clicks, no change |
| "Hide menu" at the start of the crumb | Yes | Yes | Toggles the left rail; it is not a back control | [C] `widgets.tsx:265-272` |
| Left rail (Menu / Quick access) | Yes | Yes | `requestNav(..., 'dest')`: **replaces** the stack (or pops to a screen already in it); asks first if something above is dirty | [C] `navigate.ts:54-95`, `navPlan.ts:53-85`; [V] |
| Quick access > Recent | Yes (7 shown) | Yes, after switching the rail view | Reopens a recent screen. The nearest thing to a history list | [C] `state/recents.ts:36-40,262`; [V] listed Trial Balance, Reports Center, Day Book |
| Go To (`Ctrl+K`, `Alt+G`, top-bar pill) | Yes | Yes (pill) | `goto` source: **push** (a peek) or pop back if already open; the oldest clean middle entries are dropped past depth 6 | [C] `navPlan.ts:38,92-103`; [V] pushed 4 deep, Quit returned one level |
| Drill-down (Enter / double-click) | Yes | Partly | `st.push`; the screen below stays mounted | [C] `AppShell.tsx:85-99`; [V] |
| `Backspace` | No global meaning | n/a | Bound to Remove Column in the Trial Balance and others, so it cannot be Back | [C] `reports.tsx:458-461` |
| `Alt+Left`, mouse thumb button, swipe | **Not handled** | Yes (hardware) | The app never touches `history`; the browser leaves the page | [C] grep: no `pushState`, `popstate` or `button === 3` in `src/` (only `index.html:21` `replaceState` for `?uireset=1`); [V] CDP mouse "back" went to `about:blank`, `history.length` 2, URL stayed `/` |
| `PgUp/PgDn` in a voucher alteration | Yes | **No**: the rail button "Prev/Next" has no `onClick` | `st.pop(); st.push(adjacent voucher)`, stack depth unchanged | [C] `VoucherScreen.tsx:3044-3052,3455`; [V] click did nothing, key walked No. 1 to No. 2 |

Where the navigation guard lives, and where it does not:

- **Forward moves** (left rail, Go To, `+`, favourites) go through `requestNav` -> `planNavV2` -> "Leave this entry?" (Stay armed) when
  `isDirtyAt` says there is typed, unsaved work. [C] `navigate.ts:25,54-95`, `navGuard.tsx:22-50`, `state/dirty.ts`.
- **Quit button and Esc key** never consult it. Vouchers have their own local "Quit ? (entries will be lost)" (Yes armed) and only for a **new**
  entry: `if (dirty && !editing) setQuitAsk(true) else st.pop()`. [C] `VoucherScreen.tsx:3054-3057,3461,3984`. Master forms and alterations pop with no
  question. [C] `forms.tsx:265`, `masters.tsx:805`.
- The probes are already good: the voucher probe covers alterations (`VoucherScreen.tsx:1635`), `useTallyForm` registers one for every master
  (`forms.tsx:122`). They are simply not asked when you leave by Quit or Esc.

---

## 3. The navigation stack, as built (what Back has to work with)

- `useStore.nav` is a `Screen[]`. `push` appends (after the rights check), `pop` drops the top (never below one entry), `goHome` keeps entry 0.
  [C] `state/store.ts:425-436`. Entry 0 is always the Gateway (or a gate / company picker / first-run screen).
- **Every stacked screen stays mounted**: `AppShell` renders each in a wrapper that is `display: contents` for the top and `display: none` below,
  keyed by object identity. So after a pop the screen underneath still has its cursor, row-filter text, marks, open panel state and any local
  state. [C] `AppShell.tsx:85-99`, `state/stackId.ts`. [V] filter "Sharma" (9 rows) and a marked row survived drill-and-back.
- A rail click (`dest`) **replaces**: `goHome` then push. A drill (`st.push`) and Go To (`goto`, `page`) **stack**. This was the Phase 26 fix for
  the eight-deep crumb. [C] `navPlan.ts:1-20`.
- **Period, F12 options, added columns and basis are global stores, not per-screen state.** Changing the period in a drill level changes the screen
  underneath. [V] F2 on a Group Summary (as at 31-Mar-27 to 30-Jun-26), Quit, and the Trial Balance below now read 30-Jun-26. Applying a saved
  view also calls `setPeriod`. [C] `reportViews/apply.ts:49`. Back therefore cannot "restore the period".
- The crumb names the **component**, not the object: "Menu" for every submenu, "Select" for every master list, "Voucher", "Register". Neither
  `crumbFor` nor `labelFor` gives a usable name for a submenu or a picker. [C] `router.tsx:340-383`, `state/recents.ts:177`; [V] both returned
  "Menu" for Alter and Display, "Select" / "Pick Master" for the master list. This matters for a "Back to X" label (section 8.3).

---

## 4. The table: every screen kind, what a mouse user gets

How I probed: about **170 screen entries**, plus about 45 driven flows. I entered each routed screen kind (about 130 of the ~150 `renderScreen` cases, all
15 submenu ids, 26 parameterised variants) with the store's own `push` (rights check included), read the rail, tested whether the Esc button was reachable
(`elementFromPoint` at its centre), and then **clicked it with a real mouse event**, recording the crumb and any prompt. The flow probes
drove real flows (drills by mouse, rail walks, panels, dirty states, Esc versus click). Not driven live: the two gate screens and company
create (setup needed), `reportView` (needs a saved view; read in code), pickers other than the ledger list.

"Way back" means a visible, clickable, labelled control. "Quit" below always means the rail's `Esc Quit`.

### A. Roots and menus

| Screen kind | Probed | Visible clickable way back? | What happens | Pitfall |
|---|---|---|---|---|
| Gateway root (`menu:gateway`) | 1 | None, by design. A "Quit" menu item | Item and Esc key both ask "Quit ?"; Yes calls `window.close()` | "Quit" here means leave the app; elsewhere it means go back. In a browser tab `window.close()` normally does nothing for a tab the script did not open [unverified]. [C] `MenuScreen.tsx:510-511,623-630` |
| **Gateway submenus** x15 (`create alter display accountBooks statementsAcc inventoryBooks statementsInv statutory payrollRep banking companyInfo orders otherVouchers crm hrms`) | 15 | **No.** Rail is the Gateway's (F2 to F11), no Esc button, no link in the box | Esc key pops (`MenuScreen.tsx:512`); a mouse user can only use the left rail (replaces the stack), Go To, or a menu item | **Mouse-only dead end.** Reached from Gateway letters, the rail's "More reports...", the 10 footer links, top-bar **K: Company** (stack became `Gateway > Day Book > Menu`) and `Alt+F3`. The crumb says only "Menu". [V] `menu:*` sweep, 0 of 15 had an Esc button |

### B. Reports and statements

| Screen kind | Probed | Way back | What happens | Pitfall |
|---|---|---|---|---|
| Statements: Trial Balance, Balance Sheet, P&L, Ratio, Cash Flow, Funds Flow | 6 | Yes, Quit (7th of 7 on the Trial Balance rail) | `st.pop()`; screen underneath intact | Label "Quit"; no destination named. Period edits made here persist underneath (global) |
| Day Book | 3 | Yes, Quit | Pops. With marked rows the **Esc key clears the marks first**, the button leaves and drops them | [V] Space-mark then Esc -> stayed, marks gone; Space-mark then click -> Gateway. [C] `reports.tsx:730-735` |
| Voucher registers (Sales/Purchase Register month matrix, month list, "All vouchers"; Optional / Memorandum / Post-dated registers) | 7 | Yes, Quit; plus the "By month / All vouchers" toggle buttons | **Quit from a month list pops the whole register** (to the Day Book in my stack). The Esc key goes to the month matrix | [V]. The sub-title says "Esc to return to months" (keyboard-only text). The toggle is the mouse route up, but unlabelled as back. [C] `financial.tsx:439,460`, `registerLevel.tsx:43` |
| Drill levels: Group Summary, Ledger Vouchers, Monthly Summary (then Ledger Vouchers), Stock Item Vouchers, Cost Centre Vouchers, Ledger Bills | 8 | Yes, Quit | Pops one level | Cursor row and filter restored, **mouse place not** (section 6). Trial Balance rows open by double-click, but **Group Summary rows have no double-click**: a mouse user stops there. [C] `reports.tsx:1260,1289` vs `:1087`; [V] two double-clicks on "Sundry Debtors" did nothing |
| Receivables / Payables, Ageing, Bills | 4 | Yes, Quit | Pops | Same |
| Stock Summary, Godown, Stock Valuation, Movement, Batch, Reorder ... | 9 | Yes, Quit | Pops | Same |
| **Stock Query** | 1 | Quit exists but is **covered** by the opening item picker | The `ListPrompt` list sits over the rail: my click on Quit selected an item. After choosing, **Esc key re-opens the picker, Quit leaves** | [V] `escReach = COVERED:pitem`. [C] `stockquery.tsx:71,80,216` |
| GST (GSTR-1, 3B, 9, 2B, HSN, validation, ITC, RCM, advances, docs issued ...), TDS/TCS (about 14), payroll statements | about 40 | Yes, Quit | Pops | Same. A few have a local "Back to List" (Form 16A, Sync Mismatch) that matches their Esc |
| Banking: BRS, Import Bank Statement, Cheque printing, PDC, Deposit Slip | 7 | Yes (BRS import swaps Quit for "Back" between stages) | Pops, or steps a stage | Consistent with Esc here. [C] `brsImport.tsx:820` |

### C. Entry screens

| Screen kind | Probed | Way back | What happens | Pitfall |
|---|---|---|---|---|
| **Voucher entry (new)** | 6 | Yes, Quit (24th of 24) | Untouched: pops. With typed work: **"Quit ? (entries will be lost)" with Yes armed**; Yes/No are clickable | Rail Quit is below the fold on a short window: at a 1366x680 viewport its bottom is at 654 of 656 usable, at 600 it is off screen and the rail scrolls [V]. Rail click on the same state gives the other prompt ("Leave this entry?", Stay armed) |
| **Voucher alteration** | 4 | Yes, Quit | Pops **without any question, even after an edit** | [V] changed the date 9-Apr to 3-May, clicked Quit and pressed Esc: no prompt, reopening showed 9-Apr. The left rail on the same state asks: "You have unsaved work in Sales (editing)". [C] `VoucherScreen.tsx:3054-3057,3461` vs `:1635`. `PgUp/PgDn` also replaces the voucher without asking [C `:3044-3052`]; its rail button is inert |
| Master create forms (ledger, group, stock item, unit, godown, cost centre, employee, pay head, TDS/TCS nature, voucher type ... 16 kinds) | 18 | Yes, Quit (2nd of 2) | Pops with no question | [V] typed a ledger name, Quit and Esc both discarded it silently; the rail click asked. Stack looks like `Gateway > Menu > Ledger` (the "Menu" is the Create submenu, which has no Back) |
| Master pick list (`pickMaster`: Alter > Ledger) then alteration | 2 | Yes, Quit | A single click on a row opens the alteration; Quit returns to the list | Crumb `Gateway > Menu > Select > Ledger` names nothing useful |
| Payroll, HR, CRM entry (employee, attendance, payroll voucher, leave apply, lead) | 7 | Yes, Quit | Pops | Same as masters |

### D. Hubs

| Screen kind | Probed | Way back | What happens | Pitfall |
|---|---|---|---|---|
| **Dashboard** | 4 | Yes, Quit. In **Customize** mode the rail is empty and the panel has "Cancel changes" and "Done" | Drill tiles push; Quit returns | [V] after a drill and Quit the as-at toggle (Period end) and scroll (700 of 1068) were intact. Edit mode is the one place with proper mouse buttons. [C] `Dashboard.tsx:178` |
| Reports Center | 2 | Yes, Quit | Opening a report (a single click on the name) is a **push**, so Quit returns to the Center | Good model. Esc clears the search first. [C] `ReportsCenter.tsx:220-224`; [V] |
| Create New Report (builder + Save bar) | 2 | Quit | Leaving the base report **ends the session** ("Report not saved") | Silent loss with a toast; the bar registers a dirty probe so the rail asks but Quit/Esc do not [C] `ReportBuilderBar.tsx` header, `registerDirty` import. Not driven live |
| Saved view launcher (`reportView`) | 0 live | n/a | Pops itself and `requestNav(target, 'goto')`: the report is pushed (or popped to) | Applies the view's period, F12, columns and filter to **global** stores, so Back does not undo them. [C] `Launcher.tsx`, `apply.ts:45-56` |
| Settings | 2 | Yes, Quit | Esc closes the tab's inner state first, then leaves | The key recorder dialog is the **good pattern**: Use, Remove and Cancel buttons "for someone using the mouse or a finger". [C] `SettingsScreen.tsx:72-80,105`, `KeyRecorder.tsx:96-102` |

### E. Admin and utility screens

| Screen kind | Probed | Way back | What happens | Pitfall |
|---|---|---|---|---|
| Data Check, Exception Reports, Sync Mismatch (internal levels) | 3 | Yes, Quit or "Back To List" | Step up one level, then leave: Quit and Esc agree | These are the consistent ones. [C] `DataCheck.tsx:153-156,173`, `Exceptions.tsx:239`, `SyncConflicts.tsx:401` |
| Import / Export Data, Bulk Entry, Edit Log, Custom Fields, Security, Features, Budgets, API Keys, Webhooks, Narration Snippets, Favourite Shortcuts, Price Levels, Split Company, Compare with Tally | about 22 | Yes, Quit | Pops (Bulk Entry asks first when it holds rows) | Same |
| Company select, Backup/Restore, Sync login, Subscription | 4 | Yes, Quit | Pops | The startup company picker swallows Esc and its Quit does nothing at depth 1 (correct). [C] `company/screens.tsx:52-56` |
| Print voucher, Bulk print, Print configuration | 3 | Yes, Quit | Pops | The in-screen dialogs (UPI, bank details, edit) have visible **"Cancel (Esc)"** buttons, another good pattern. [C] `PrintVoucher.tsx:838,881,916` |
| Gate screens (company passcode, operator sign-in) | 0 live | "Back" / "Companies" button | Returns to the company list | Code only. [C] `CompanyUnlock.tsx:93`, `OperatorSignIn.tsx:249-259`; `access.ts:181-182` |

### F. Overlays and panels

| Overlay | Probed | Visible close? | Click outside? | Rail Quit reachable? | Notes |
|---|---|---|---|---|---|
| Go To palette | Yes | none | **closes** | n/a | [V] opened from the top-bar pill |
| Universal `+` menu | Yes | none | **closes** | n/a | [V] |
| `?` key sheet / F1 Help | Yes | none | **closes** | n/a | [V] |
| Header dropdowns "All v" (views) and "+ New v" | Yes | none | **closes** | not checked (a popover, not a full-window shield) | [V] |
| **`Box` / `.overlay` dialogs**: F2 Change Period, F12 Report Configuration, Alt+V Saved Views, Filter rows (Add condition, Add group, Reset only), Voucher Date, Voucher F12, New Column, Basis of Values, **F10 Change Voucher Type** (a hand-built `.overlay`, `VoucherScreen.tsx:657-690`) | 10 | **None** (hint text "Esc close" only; Change Period and Voucher Date show no hint) | **No** | **No**: `elementFromPoint` returns `.overlay` over the whole window; clicking Quit did nothing | **Trap.** [V]. 24 `<Box` sites in 19 files plus 6 hand-built `.overlay` dialogs in 5; `.overlay` is `position: fixed; inset: 0; z-index: 50` [C] `styles.css:1226-1229`, `widgets.tsx:502`. The Settings key recorder is a `Box` that **does** have buttons (`KeyRecorder.tsx:97-102`), so the shell allows it; most callers do not use it |
| **`ListPrompt` pickers**: Stock Query items, Share panel (also bank, price-level, cheque, TDS-nature pickers, not opened) | 2 | **None** | **No** (Share: [V]; Stock Query: the scrim click did nothing) | **No**: Stock Query's list covers the rail's Esc slot | **Trap.** [V]. 17 sites in 13 files; `position: fixed; inset: 0; z-index: 60`, 25% scrim, Esc only. [C] `widgets.tsx:400-440` |
| `AcceptPrompt` (Accept ?, Quit ?, Leave this entry?) | Yes | Yes/No (or Leave/Stay) are clickable | n/a | yes | [C] `widgets.tsx:476-486`; [V] |
| Voucher sub-panels (bill-wise, cost centre, batch, bank instrument, GST line, dispatch ...) | 0 live | not seen | not seen | not seen | Mostly `Box`-based [C]; assume the Box trap until shown otherwise |
| Left-rail "Leave this entry?" | Yes | Leave / Stay buttons | n/a | n/a | [V] text "Opening X would discard it" |

---

## 5. Findings, in order of how much they hurt a mouse-only person

**F1. Gateway submenus: no back control (P0).** 0 of 15 have an Esc button. The Gateway's rail is reused as-is for submenus, and its list never
had one because the Gateway is the root. [C] `MenuScreen.tsx:532-545` (no `Esc` entry), `:510-513` (the key pops for a non-root menu). A top-bar click
on "K: Company" is enough to land on one. [V]

**F2. Modal dialogs and pickers cannot be closed with a mouse (P0).** No close button, no scrim click, and a full-window shield over the rail, on all
12 dialogs and pickers I opened. Esc-only is a keyboard assumption built into the shared shells (24 `<Box`, 6 hand-built `.overlay`, 17 `<ListPrompt`
sites; I opened a dozen of them and the rest are the same component). [C, V] Contrast with those done properly: Go To, `+` and `?` close on a click
outside; the key recorder, print dialogs and Dashboard edit mode have real buttons.

**F3. The browser's own Back leaves the app (P0 if mouse users have a thumb button, which most do).** No `history.pushState`, no `popstate`
handler, no mouse back-button handler. Real history navigation went to `about:blank` and the app's position is gone on return. [C, V] In Electron
nothing handles an `app-command` browser-backward either [C `electron/main.cjs`, grep]; what Electron does by default I could not check
[unverified].

**F4. The one visible control is a Tally button with a Tally label (P1).** "Quit" + a key cap, in a 152px rail, at a different slot per screen,
shown in the same weight as Duplicate or Delete; its tooltip reads "Esc, Quit". [C] `widgets.tsx:91`. "Quit" is the app-exit word on the Gateway.

**F5. Quit is not Esc (P1).** The 99 rail buttons call `st.pop()` directly while the Esc handlers do more. Found live: Register month list,
Stock Query, Day Book marks. Screens where the two agree (Data Check, Exceptions, Sync Mismatch, BRS import, Webhooks) wrote the button's
`onClick` by hand to mirror the key. [C] `reports.tsx:584`, `financial.tsx:327,439`, `stockquery.tsx:71,80`.

**F6. Quit and Esc skip the dirty guard (P1).** Edited voucher alteration and typed-in master forms are discarded without a word, while the same
state asks when you click the left rail. The probes exist and are right; the exits do not ask them. [V] Whether TallyPrime itself asks on Esc in
these two cases I could not check [unverified]; note that the repo's own rule is "never change what a key means", so a prompt on Esc is an owner
decision (section 11).

**F7. A rail or top-bar entry replaces the stack (P1).** After TB > Group Summary > Ledger Vouchers, one rail click on Accounting left
`[Gateway, Day Book]`; Quit then went to the Gateway. The drill is gone, and so is its cursor and filter. The Quick access > Recent list is the only
trail, one view-switch away. [V]

**F8. Crumb segments are inert, by decision (P2).** The reason given (a segment click pops several screens and bypasses the per-screen guard) is
real; it goes away if a segment click is built as repeated guarded single steps, but I recommend not doing that in v1. A labelled Back covers
the one-level case that matters.

**F9. A mouse user loses their place on return (P1).** `onDoubleClick` opens the row but does not set the keyboard cursor; on return the report
re-reveals the cursor row (`revealCursorRow`, "a hidden screen loses its scroll offset"). [C] `reports.tsx:110-116`, `reportNav.ts`. [V] Day Book
wheel-scrolled to 246 of 247, double-click a row, Quit: back at 48 of 247 with the first row selected. With the keyboard (45 Down, Enter, Quit) scroll
206 and the selected row came back exactly. I did not separate "the engine kept the offset" from "reveal moved it"; the effect is the same. The
Dashboard came back exactly (it is not a row-nav screen).

**F10. Mouse drill gaps next to Back (P2).** Group Summary rows have no `onDoubleClick` (the comparative and the single-period tables).
`PgUp/PgDn` is a rail button with no handler. A single click on a report row does not select it. [C, V]

**F11. Back cannot restore global settings (P2).** Period, F12 options, columns, basis are global and a saved view rewrites them (section 3). [C, V]

**F12. Names for a label (P2).** `crumbFor` and `labelFor` return "Menu", "Select", "Voucher", "Register". A "Back to <crumb>" label would read
"Back to Menu". A purpose-built `backLabel()` is needed. [V]

**F13. Rail height (P2).** The voucher rail has 24 buttons with Quit last; on a window under about 680px high it is below the fold. [V]

---

## 6. What a level restores on Back today

All of this is for a **pop** (the screen underneath is still mounted). A **rail replace or `goHome`** unmounts the screens, so none of it survives.

| State | Restored on pop? | Evidence |
|---|---|---|
| Keyboard cursor row | Yes | [V] |
| Row-filter text and narrowed rows ("type to filter") | Yes (component state) | [V] "Sharma", 9 rows |
| Marked rows (selection bar) | Yes | [V] |
| Open internal level (register month list) | Yes, if the screen is only covered | [C] state lives in the component |
| Scroll offset | Replaced by "cursor row just visible" on report screens; Dashboard restores its own | [V] 246 -> 48 (mouse), 206 -> 206 (keyboard), 700 -> 700 (Dashboard) |
| Mouse-selected row | **No** (a double-click never moves the cursor) | [V] |
| Period, F12 options, columns, basis | **Not restored: global**, whatever a lower level set stays | [V] F2 in Group Summary changed the Trial Balance |
| Unsaved edit in the screen you leave | **Lost silently** for alterations and masters; asked for new vouchers | [V] |
| An open dialog | n/a (cannot be reached by mouse) | [V] |

---

## 7. Is Esc enough for keyboard parity?

Yes for the keyboard: Esc is already a reserved key ("Escape" is in `RESERVED_NAMED_KEYS`, never rebindable) and every screen handles it.
[C] `core/shortcutRegistry.ts:297-300`. Options for an extra chord, all poor: `Backspace` is bound to Remove Column; `Alt+Left` is the browser's own Back
in a tab (the repo's Pipedrive notes already avoid it for that reason, `pipedrive-list-ux.md:319`) and its `preventDefault` behaviour I did not
test [unverified]. **No new chord is needed. If one is wanted, it must be appended to `core/shortcutRegistry.ts` (rules 1 to 4 at the top of
that file) and printed with `keyLabel`, never typed into a string.**

---

## 8. Recommendation [J]

### 8.1 One Back control, in the top bar, right after the hamburger

| Candidate | Reaches every screen kind? | Visible without scrolling? | Notes |
|---|---|---|---|
| Start of the breadcrumb, next to Hide menu (status bar, bottom) | Yes (`AppShell` renders it for all) | Yes, but a 24px strip at the bottom edge in the smallest type | Far from the title and the eye; it sits beside text that is not clickable |
| Page header `.scr-head` | **No**: only `ReportShell`, `PickMaster` and the Chart of Accounts draw it. Not menus, vouchers, masters, Settings, Dashboard | Yes (the scroller is `.scr-body` below it) | Best for lists, but needs a second solution for everything else |
| **Top bar, after the hamburger** | **Yes** (outside every screen) | **Yes**, always, same place | Browsers put Back at the top left; the hamburger is already there; the bar sheds low-priority items itself |
| Both | n/a | n/a | Two Backs on one screen is worse than one |

**Choose the top bar only.** Geometry at 1440 wide [V]: hamburger 12-44, brand 50-156, items 170-699, spacer, Go To pill, `+`, theme controls. Insert
a flex-none child between the hamburger and the brand (about 70 to 160px). The existing item group (`.topitems`) already sheds into a "..." menu
when width runs out (at 800 wide it shed three items), so the Back control never has to. Keep the rail's `Esc Quit` and the crumb's "Hide menu" where they are.

### 8.2 Behaviour

1. **Visible when** the stack has at least 2 entries and the top is not a gate screen (`isGateScreen`, `access.ts:181`). Hidden (not disabled) on the Gateway
   root, the startup company picker, first-run company create, passcode and operator sign-in. `aria-disabled` and visibly dim while any modal is up
   (`overlayUp()`, `.overlay`, `.nav-guard`).
2. **Action** `requestBack()` in `ui/navigate.ts`. A new `NavSource` value `'back'` in `core/navPlan.ts`: target is stack entry `n-2`, plan is `pop-to`
   when the top is clean and `ask` (then `{ popTo: n-2 }`) when `isDirtyAt` says the top holds unsaved work. It can be built today as
   `requestNav(stack[n-2], 'dest')` (the planner already finds `k = n-2` and asks if the top is dirty), but a dedicated source avoids key matching and
   a "stay" when the top and the previous screen share a key. This reuses `NavGuardHost` ("Leave this entry?", Stay armed), its check that clears a
   question gone stale, and focus hand-back (`refocusRevealed`). Reword the note for this source: "Going back to Day Book would discard it" instead of "Opening ...".
3. **Back does exactly one screen, never an internal level.** That keeps the label true. A screen that has an internal level a mouse user needs
   (register months) already has a button for it (the "By month" toggle); an optional opt-in hook `useBackTarget({ label, run })` could let the
   register show "Back to By month" later. Not needed for v1.
4. **The dirty-leave guard applies, so Back is safer than Quit/Esc today** (alteration, masters, builder session all ask). A voucher in creation will show
   "Leave this entry?" (Stay armed) instead of its own "Quit ?" (Yes armed): acceptable, the rail already does this for the same state.
5. **Focus**: use the hamburger's manners: `onMouseDown preventDefault` (a field being typed in keeps its caret), `onKeyDown={ownActivate}` (Enter and
   Space belong to the button, not the screen's key stack), `type="button"`, accessible name "Back to Day Book", `title` "Back to Day Book (Esc)".
   Do not make it a global `useKeys` handler (the standing gotcha): a button needs no key handler.

### 8.3 Label

Always show a chevron and the word **Back**; add the destination after it when there is room ("Back to Day Book", truncated with an ellipsis, the full name
in the tooltip). The visible words are the control's name, so it can be found without hovering (the same reasoning as "Hide menu", `left-nav-spec.md:839`).
It needs a new `backLabel(screen)` (pure, in `core/navPlan.ts` or `state/recents.ts`, not `crumbFor`): menus -> the menu's own title ("Display More
Reports", "Create"), master list -> "Ledger list", voucher -> its type, ledger or group screens -> the master's name, otherwise `labelFor(...).title`. A test must
fail on any `Screen` kind that comes back "Menu" or "Select". Under Settings > Appearance text size 150% the label must truncate before the bar wraps.

### 8.4 A session trail, so a rail-entered screen has somewhere to go back to

After a rail click the stack is `[Gateway, X]`, and Back to the Gateway is useless to someone who lives in the rail. Keep a **session-only** trail
(`state/trail.ts`, in memory, 10 entries, de-duplicated, never persisted: persisted Recents would offer a stale screen on the first click after launch).
Record the screen being left whenever a `dest` move replaces the stack, using `isRecordable`/`screenKey` from `state/recents.ts` (so edit screens and one-shot
pickers are skipped). Rule: **depth >= 3 -> pop one (a drill); depth 2 with a trail -> reopen the last trail entry as a `dest` move (the guard applies);
otherwise the Gateway.** Be honest in the tooltip: a trail reopen starts the screen fresh (cursor, scroll and filter are gone because the rail replace unmounted
it; period and F12 are global so they still hold). The alternative, keeping replaced screens mounted, re-creates the eight-deep stack Phase 26 removed.

### 8.5 Forward

**No mouse Forward in v1.** A pop unmounts the screen, so a Forward would reopen it fresh, which is the same as clicking where you came from. The rail's
Quick access > Recent already serves "go back to what I was doing". Revisit only if users ask.

### 8.6 What each level should restore (and the two small fixes)

- Keep today's pop behaviour (cursor, filter, marks, internal level).
- **Mouse parity for the cursor**: make a double-click (and a single click) on a data row set the row-nav cursor to that row before the drill. The shell's DOM
  pass already maps rendered rows to nav indices (`ReportShell.pass`, `mappingReliable`); give `navLink` a `select(i)` and have the shell call it from one delegated
  click handler, instead of editing the five `onDoubleClick` sites. Then Back lands on the row the person opened.
- **Scroll**: record `scrollTop` for the `.scr-body` when a screen stops being top and restore it when it is top again (the wrapper in `AppShell` knows both), and make
  `revealCursorRow` run only if the cursor row is then off screen. The Dashboard already behaves this way.
- Period, F12, columns and basis are global: do not pretend to restore them. If a saved view changed the period, say so in its own toast; do not add Back magic.

### 8.7 Fix the dialog trap before, or with, the Back button

A Back control in the top bar does not help while a modal is up (the shield covers the whole window). So:

1. `Box` gets an optional `onClose` and draws a close button in the title bar (the key recorder's `stk-rec-btns` is the in-repo pattern; print dialogs use "Cancel (Esc)").
   It calls the **same handler as Esc** for that dialog (`onCancel`, `onDone(null)`, `onClose`), so keyboard and mouse can never differ.
2. `ListPrompt`: a visible Cancel and a click on the scrim calls `onCancel` (nothing is lost by cancelling a pick).
3. A click on the scrim closes only dialogs that cannot hold typing (F12, Saved Views, Basis, Filters, Help-like). Change Period, Voucher Date and the entry
   dialogs close only through the button, so a stray click never drops half-typed dates.
4. One change in `Box` and `ListPrompt` covers 24 + 17 call sites; pass `onClose` from the call site only where its Esc handler is not the obvious one.

### 8.8 Make the rail's Esc buttons do what Esc does

`ButtonPanel` (`widgets.tsx:82-97`) already post-processes every rail (`applyRail`). Make a `Btn` with `k === 'Esc'` run `pressEscape()` (dispatch a keydown `Escape`
at `document.activeElement ?? window`, so an element-level handler such as the filter box's sees it first), instead of each screen's hand-written `st.pop()`.
That fixes the Register, Stock Query and Day Book marks in one place and removes the drift between the key and the button for good (99 sites). Screens that already
mirror their key keep working (they handle the same event). Add the guard test: every rail Esc button, clicked, equals pressing Esc, on every screen kind
that the sweep reaches.

### 8.9 History shim (phase 2, optional)

To make the mouse's thumb button, a trackpad swipe, `Alt+Left` and `Cmd+[` mean "Back" inside the app: `history.pushState` on every user-initiated push, a `popstate`
listener that calls `requestBack()`, and if the guard asks, `pushState` again to cancel the browser's step. Facts from the standard docs: `popstate` fires when the user
traverses history (Back or Forward) and when script calls `history.back()`, and **not** when `pushState` is called [W]
(https://developer.mozilla.org/en-US/docs/Web/API/Window/popstate_event, https://developer.mozilla.org/en-US/docs/Web/API/History/pushState). **Caveat:** Chromium's
"history manipulation intervention" makes the browser's Back **button** skip same-document entries created with no user activation, and does not affect
`history.back()`; once the document has had a user interaction, its entries are not skippable [W] (https://chromium.googlesource.com/chromium/src/+show/08d828b920062e2b0be1f95b7d2416d163091745/docs/history_manipulation_intervention.md;
a second copy of that page returned HTTP 503). So push only from clicks and keys, and use `replaceState` for programmatic moves (the saved-view launcher, startup).
Forward is treated as a no-op (push again). Risks to test: the `?uireset=1` cleanup in `index.html:21`, Electron's `file://` origin, the sync reload, and Firefox's
similar setting [Ws]. Do this only if the owner confirms mouse users reach for the thumb button; the top-bar Back already answers the written request.

---

## 9. Touchpoints of the frozen layout

| Thing | Touched? | Why it matters |
|---|---|---|
| **F-key rail** (`.btnpanel`, grid column 3, 152px, `Btn` list, Esc last with `gap`) | **Behaviour only** (8.8): no button added, none moved, label unchanged | The Tally keys and their order are frozen; Settings can hide a rail button but "hiding never disables its key". Do not rename "Quit" to "Back" without the owner (open question) |
| **Three-column grid** `.main { 186px 1fr 152px }`, `.app.nv-wide` 212px, `.app.navhid` 0 column | **Not touched** | The Back control is in the top bar, outside `.main`. `visual.assert.ts` reads the base `.main` rule. The `navhid` bug (hide a grid item with `display: none`) is a standing warning: never remove a grid child to hide Back; the control is not in the grid at all |
| Top bar `.topbar` (flex row) | **Yes**: one new flex-none child after `.navtoggle` | `railtop.css` already sets `.topbar .brand { flex: none }`, `.topbar .gotopill { min-width: min-content }`; add the sibling rule there. Check at 800, 1024, 1440 wide and at 90 to 150% text size (`--ui-k`; use `sz()` or `calc(Npx * var(--ui-k))` for any literal) |
| Top-bar layout setting (`core/railLayout.ts applyTop`, `TOP_ITEMS`) | Back is **not** an item | It must not be hideable: it is the pointer's escape hatch, in the same class as `Ctrl+,` and Go To (`settings-spec`) |
| Status bar crumb and "Hide menu" | Untouched (optionally a tooltip "Back: Day Book") | Segments stay inert (F8) |
| Light scheme overlay | New class needs a `html:root[data-scheme='light']` rule only if it differs from base | Dark inherits; never put a hex in `:root` for a token dark does not override. Use the `.navtoggle` colours and re-run the accent contrast matrix in `visual.assert` (8 presets x 2 schemes) |
| `?` key sheet, hold-Alt reveal | Add a row "Esc: go back (the Back button too)" | No new chord, so `shortcutRegistry` is untouched unless the owner wants `Alt+Left` in Electron |
| `AppShell.tsx` hosts | Back reads `st.nav`, `useNavGuard.pending`, `overlayUp()`; no new host | A host would register a global handler; a button needs none |
| Left-rail suites (`sidenav`, 65 rows) and Go To golden (103 queries) | Untouched | Back adds no menu row and no palette entry (do not add a Go To command: it would re-rank queries, see the Phase 27 gotcha) |

---

## 10. How to prove it (assertions and recipes, for the builder)

Pure (node) suites:
- `backLabel` over every `Screen` kind and every `MenuId`: none is "Menu", "Select", "Pick Master", empty, or longer than 40 characters.
- Plan: depth 1 and gate screens give no Back; clean top gives `pop-to n-2`; dirty top gives `ask` with `then: { popTo: n-2 }`; a deep stack after a Go To collapse
  still pops one entry; the trail rule (8.4) for `[Gateway, X]`.
- Every `Esc` rail button, when clicked, equals the Esc key (a table walk across the sweep); each mutation list kills a revert to `st.pop()`.
- `Box` and `ListPrompt` render a close control that calls the Esc handler, and a scrim click closes only the listed dialogs.

Recipes (`scripts/shots/recipes/back.mjs`, real mouse events, never Accept):
`back-submenu` (Gateway > Display More Reports > click Back > Gateway; also top-bar K: Company), `back-drill` (TB > Group Summary > Ledger Vouchers > voucher; Back x3),
`back-dirty-master` and `back-dirty-alteration` (type, click Back: prompt, Stay changes nothing, Leave pops), `back-rail` (rail click, then Back, then the trail),
`back-dialog` (F12 and F10: the close button works, Quit is not needed), `back-register` (month list: Quit equals Esc), `back-place` (scroll, double-click a row, Back: same row
and offset), `back-narrow` (800 and 1024 wide, 150% text: no wrap, no overflow), light and dark, comfortable density. A `nav-forced-colors` pass for the control.

Suggested exit criteria: all the above green; 642 + n screens clean in three sweeps; the Go To golden and `sidenav` unchanged; `keys` and `shortcutprefs` untouched.

Phasing [J] (S/M are my size guesses, not estimates from the team): (1) Back control + `backLabel` + `'back'` source + guard wording: M. (2) `Box`/`ListPrompt` close: M (41 sites,
one change plus a few overrides). (3) rail `Esc` parity in `ButtonPanel`: S. (4) cursor-on-click and scroll restore: S to M. (5) trail: S. (6) history shim: M, optional.

---

## 11. Open questions for the owner

1. Should the rail's `Esc Quit` button be relabelled "Back" on every screen except the Gateway? It is a visible default change for Tally switchers; the Back control makes it less necessary.
2. Should the **Esc key** also ask before discarding an edited alteration or a typed master (as the left rail already does)? That changes what Esc means on those screens; the Back button will ask either way.
3. Do mouse users in your customer base reach for the thumb button or a swipe? If yes, do phase 6.
4. Is a mouse **Forward** wanted? (Recommended no.)
5. Should the top-bar Back also appear on Settings and the Dashboard in edit mode (which have their own Cancel/Done)? Recommended: yes on Settings, and disabled in Dashboard edit mode.

## 12. What I could not verify

No live TallyPrime, Zoho Books or Pipedrive account: I did not check how any of them place Back, or whether TallyPrime asks on Esc for an edited alteration or a typed master. Headless Chrome on macOS only: not Windows, Linux, Firefox, Safari or Electron
(Electron 31 cannot be driven here); no real touch, no screen reader. The mouse thumb button was simulated by the CDP "back" button, which navigated the page to `about:blank`; a real browser with a real prior page is expected to go there. I entered screens with the store's `push`
(rights check included) rather than by clicking to them, so the **route** a person takes was walked separately in about 45 flows; the Quit/back click itself was always a real mouse event. The sample company is small (Day Book scrolls 247px). Voucher sub-panels, the
report-builder session, the two gate screens and `reportView` were read in code, not driven. `window.close()` on the Gateway's Quit in a normal tab, and Electron's default for a back command, are unchecked. Probe screenshots were written to the scratchpad and are not committed.

## 13. Sources

Repo: files and lines cited inline. Prior decisions: `docs/design/left-nav-spec.md:83,839-840` and `docs/design/zoho-ux-spec.md:538-626` (crumb segments stay inert; the stack policy), `docs/design/ux-analysis-2026-10.md` (the eight-deep crumb),
`docs/research/zoho-books-reference.md:376` (Zoho's pages describe no breadcrumb, nothing to borrow), `docs/research/pipedrive-list-ux.md:319` (browsers use Alt+Left for Back).
Web [W]: MDN `popstate` event (https://developer.mozilla.org/en-US/docs/Web/API/Window/popstate_event); MDN `History.pushState` (https://developer.mozilla.org/en-US/docs/Web/API/History/pushState); Chromium, "History manipulation intervention"
(https://chromium.googlesource.com/chromium/src/+show/08d828b920062e2b0be1f95b7d2416d163091745/docs/history_manipulation_intervention.md). [Ws] one web search on the same intervention, including Firefox's similar preference.

## Appendix A. Probe log (what was clicked, in the running app)

- **Sweep, Quit by real mouse click, screen entered with `push`:** statements and reports (about 60: TB, BS, P&L, Day Book, Ratio, Cash/Bank Book, Cash and Funds Flow, Stock, Godown, HSN, GSTR-1/3B/9/2B, TDS family, payroll family, ageing, cost centre,
  movement, valuation, batch, reorder, credit limits, cancelled register, deposit slip), admin and hubs (about 30: Dashboard, Reports Center, New Report, Settings, COA, Features, Company alter/select, Backup, Sync, Billing, CRM, HR, API, Webhooks, Snippets, Shortcuts, Edit Log, Import, Export, Compare, Bulk Entry, Data Check, Exceptions, Sync Mismatch, Security,
  Custom Fields, Price Levels, Budgets), 18 master create screens, 15 submenus, and 26 parameterised screens (ledger vouchers, group summary, stock group summary, stock item vouchers, pending orders, register months and all, monthly summary x2, outstandings x2, ledger bills, cost-centre vouchers, voucher registers x3,
  voucher new x2, alteration, duplicate, print, bulk print, print config, ledger and stock item alteration, builder). Result: Quit present and working on every screen except the 15 submenus (no button) and Stock Query (covered by its picker).
- **Real flows:** Day Book Quit; register month list (key versus click); TB double-click drill; Group Summary double-click (inert); Day Book scroll then drill then Quit (246 to 48), and keyboard cursor (206 to 206); Dashboard drill and Quit (700 kept); Reports Center open and Quit; Create Ledger dirty then Quit, Esc and rail click; alteration with an edited date then Quit, Esc, PgDn and rail click; "Prev/Next" click;
  F8 untouched and typed then Quit; rail walk (Accounting, Day Book, Reports); Go To peek then Quit; rail replace then Quit; period change in Group Summary then Quit; Alt+Left, Backspace, Esc and the CDP mouse back button; crumb clicks at start, middle and end; top-bar K: Company; Go To, `+` and `?` scrim clicks; F2, F12, Alt+V, Filters, voucher F2 and F12, New Column, Basis (Box), F10 (a hand-built overlay), the Stock Query picker and the Share panel (ListPrompt) with a scrim click and a Quit click;
  header "All" and "+ New" dropdowns; Dashboard Customize; Quick access Recent; voucher rail height at 900, 768, 680 and 600; top bar geometry at 1440, 1024 and 800; filter text and marks across a drill.
- Method: a scratchpad Node script imported `scripts/shots/chrome.mjs`, `page.mjs` and `app.mjs` unchanged, kept one private Chrome with the sample company open, and evaluated snippets; the page's store was reached with a dynamic `import('/src/state/store.ts')` (the same module instance as the app). No repository file other than this document was edited, no git command that changes state was run, and nothing was accepted or saved to a company.
