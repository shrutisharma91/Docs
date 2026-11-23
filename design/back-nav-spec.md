# Comsky Books: the Back control for mouse-only use (architecture and spec)

> **Status (updated 2026-10-25, after the rebase onto `main`):** BUILT, reviewed and fixed on branch `cf-back-nav`; BETA, not deployed, never used by a customer. B-1 (the Back control), B-2a and B-2b (a Close on every dialog, `onClose` required),
> B-3 (the rail's Esc button runs Esc) and the grid-independent part of B-4 are in the tree; **B-5, B-6 (the browser's Back, thumb button, swipe, `beforeunload`) and B-4's grid half are NOT built.** What was built, where it deviates from this
> text (the component is `BackButton.tsx`; `requestBack` returns a status string) and the gate are in `docs/design/cf-back-nav-build-notes.md`: trust that file over this one wherever they differ. The text below is the design as written, and the
> sentences in the future tense or about the tree "belonging to wave 3" were true on 2026-10-21; parity wave 3 is committed (`2cee555`) and merged under this work.
> **Original status (2026-10-21):** design only. Nothing was built and no source file was changed. Written 2026-10-21 against `HEAD` = `83d9876` plus the uncommitted parity-wave-3 tree (158 changed paths).
> This is the build contract for the owner's request item 4: "an option to go back in every report or view, which is currently not there, so that a person operating through a mouse only should be
> able to go back into the reports". Its sibling is `docs/design/custom-fields-spec.md`, which holds the combined **build order** (its section 17) that this document's section 13 summarises.
> **Method:** `docs/research/back-nav-audit.md` (written today; about 170 screen entries and 45 flows probed in a throw-away headless Chrome with real mouse and key events) was read in full, with
> `docs/design/zoho-ux-spec.md` section 5 (the navigation stack, the dirty guard, the crumb decision), `docs/design/left-nav-spec.md` (the full-screen toggle and the crumb), and the code the audit cites, which I
> re-read where a decision depends on it (files named). **I did not re-run the audit's mouse probes** (nothing may be run: the tree belongs to wave 3), so every "[V]" below is the audit's, tagged **[R]** here.
> **Evidence tags:** **[C]** read in this repo's code this session (file named; wave-3 line numbers drift, symbols do not). **[R]** stated in `docs/research/back-nav-audit.md`, which itself tags what it verified live
> as [V] in its own probes. **[W]** a web page (MDN and Chromium pages cited by the audit; I did not re-fetch them). **[J]** my judgement. **[unverified]** could not be checked. There was no live TallyPrime, Zoho Books or
> Pipedrive account, so nothing here claims how another product places Back.
>
> **Revision (2026-10-21, second pass).** An adversarial review raised 38 findings across this spec and `docs/design/custom-fields-spec.md`; 10 concern this one (its **Review log** is at the end; the other 28 are in the sibling's). Each was re-checked against the code first: **all reproduced** (one rests on the reviewer's own pixel measurement, which I did not redo), so none is rejected; one fix was adapted (finding 31). What changed: a Close that records a statutory decision is no longer treated as a neutral close (the two TDS dialogs, B5, 5.1 to 5.2); `planBack` says WHY it did nothing, and browser Back no longer reads a pending question as "at the root" (2.1, 10.1); Back is blocked while an Accept is in flight (2.2); the Custom Fields editor and its panels register dirty probes (4); the rail Esc override is proved screen by screen (6, 11); the top-bar control is chevron only below 1100 px and its tooltip no longer claims Esc equals Back (1); Group Summary rows get the mouse drill the audit found missing (3).
>
> **Note (2026-10-22).** The sibling spec gained an addendum for the owner's answers of that day. It adds no chord and no Back behaviour; it adds three screens or panels to the table in section 4 (the entry-rules panel, the five analysis screens, the Opening by variant panel) and widens the build order of section 13 (the sibling's section 17 is the source of truth).

---

## 0. The decisions, in one table

| # | Decision | Why (section) | Vetoable |
|---|---|---|---|
| B1 | **One persistent Back control, in the top bar, immediately after the hamburger.** A chevron, and the previous screen's real name when there is room ("Back to Day Book"); **chevron only below 1100 px**, the destination in its tooltip and accessible name. Hidden (not disabled) at the Gateway root, on the startup company picker and on the two gate screens | the audit's finding that the only visible way back is a per-screen right-rail "Quit" at a different height on every screen, and none at all on the 15 Gateway submenus (1) | yes (placement and wording) |
| B2 | Back is **one screen, never an internal level**, and goes through `core/navPlan.ts` as a new pure `planBack`, so **the dirty-leave guard applies** ("Leave this entry?", Stay armed); **`planBack` names why it did nothing** (root, a question pending, a gate, an Accept in flight) | a naive `st.pop()` loses an edited alteration or a typed master silently, which Quit and Esc do today (2); a browser Back that read every `none` as "at the root" would leave the page over typed work (10.1) | no |
| B3 | A small **session trail** (memory only, 10 entries) so Back from a screen entered by a **left-rail or top-bar click** returns to where the person was, not to the Gateway | one rail click replaces the stack, so "back" after it is the Gateway, which is useless to someone who lives in the rail (2.3) | yes |
| B4 | **No Forward** in v1 | a pop unmounts the screen, so Forward would reopen it fresh, the same as clicking where you came from; Quick access > Recent already serves "get me back to what I was doing" (7) | yes |
| B5 | **Every dialog and picker gets a visible Close, and a click outside closes the ones that hold no typed work and decide nothing.** `Box` gets an `onClose` (optional first, required last: 12.1); `ListPrompt` gets a Cancel button and a backdrop click, **with `scrim` and `cancelLabel` props so a picker whose Esc records a decision (the TDS nature picker) neither closes on a stray click nor labels its button Cancel**; the TDS prompt's button says "Skip TDS"; the hand-built `.overlay` dialogs follow | 12 of 12 dialogs the audit opened (F2 Change Period, F12, Saved Views, Filter rows, the voucher date, F10 Change Voucher Type, New Column, Basis of Values, Stock Query's item list, Share...) are mouse traps: no close button, no outside click, and a full-window shield over the rail's Quit (5); and two of the 41 dialogs record a statutory decision on Esc (A, item 9) | no |
| B6 | **The rail's `Esc Quit` button runs what the Esc key runs**, through one `runEscape()` in `ButtonPanel`, instead of 99 hand-written `st.pop()` handlers; **proved screen by screen: a table-driven test names every rail Esc button (120: 96 `st.pop()`, about 24 closures over screen state) and the key handler each must reach** | Quit and Esc disagree on a register's month list, Stock Query and a Day Book with marked rows (6) | no |
| B7 | **The Esc key keeps its meaning, the label "Quit" stays, and no chord is added.** Back asks before discarding unsaved work; Esc keeps the prompts it has today | the repo's rule: never change what a key means (Phase 20 superset rule); renaming "Quit" to "Back" on every screen is a visible default change for Tally switchers (8) | yes (the rename, Q1; Esc asking, Q2) |
| B8 | **The browser's own Back (thumb button, trackpad swipe, Alt+Left) becomes the app's Back in a second phase**, by a single sentinel history entry on the web and an IPC message in Electron; **built only if the owner confirms mouse users reach for it** | today it leaves the app and the person's position is lost [R]; but history code has real cross-browser risk (10) | yes (Q3) |
| B9 | **A mouse double-click sets the keyboard cursor, and a screen's scroll offset is restored on return**, inside the grid's `ListTable` (or a small edit of `ReportShell` if the grid is late) | a mouse user who drills from a Day Book scrolled to the bottom came back near the top (246 to 48 of 247) while the keyboard path restored exactly [R] (3) | no |
| B10 | The **Gateway submenus** need no rail change: the top-bar Back covers them. An `Esc Quit` rail button on them is an option for the owner | 0 of 15 had any back control [R]; whether TallyPrime's own submenus show one is [unverified] (4) | yes (Q6) |
| B11 | **The top-bar control and a rail button that is also labelled Back are told apart by the destination.** 17 rail `Esc` buttons in 13 files already read Back or Back to List and go up one INTERNAL level; the top-bar control always names a destination and never shows the bare word, and its tooltip never says Esc equals Back (1.1, 6.1) | two Backs with different meanings on one screen (Import Data's mapping step) | yes (wording) |
| B12 | **Anything that makes leaving unsafe registers itself:** a screen holding typed work registers a dirty probe (the Custom Fields editor and its panels now do), and an Accept in flight registers a busy probe that blocks Back | the dirty guard is only as good as its registry (4, 2.2) | no |

---

## A. What I verified in the tree, and where I differ from the audit

1. **The top bar** is `TopBar` in `src/ui/widgets.tsx`: `.topbar` (a flex row) holds the `.navtoggle` hamburger, `.brand`, the shedding `.topitems` group, a spacer, the Go To pill, `QuickCreateButton`, `ThemeControls`
   [C]. The hamburger's manners (`onMouseDown` preventDefault, `onKeyDown={ownActivate}`, `type="button"`, `aria-controls`) are the in-repo pattern the Back button copies. `railtop.css` already pins `.topbar .brand { flex: none }`
   and makes `.topitems` shed into a "..." menu when the bar is narrow [C], so a new flex-none child before the brand costs the shedding group some room and nothing else.
2. **The stack and its guard exist**: `useStore.nav` with `push`, `pop` (never below one), `goHome`; `requestNav` and `planNavV2` (`dest`, `page`, `goto`); `isDirtyAt`; `NavGuardHost` with its stale-question check (it clears a pending question when the stack changes)
   and `leaveAndGo` / `stayPut`; `heldWords` for the prompt's second line [C `state/store.ts`, `ui/navigate.ts`, `core/navPlan.ts`, `ui/navGuard.tsx`, `state/navGuard.ts`]. **There is no `requestBack`** (the audit and the code agree).
3. **`Box` has no way to close.** `export const Box: React.FC<{ title; children }>` draws `.overlay > .box` and nothing else; each caller's Esc handler lives in its own `useKeys`. `ListPrompt` draws a `position: fixed; inset: 0; z-index: 60` backdrop with a 25 % scrim and no
   click handler of any kind [C `widgets.tsx`]. Counted by `grep`: **24 `<Box` sites in 19 files, 17 `<ListPrompt` sites in 13 files, and 5 hand-built `.overlay` dialogs** outside `Box` itself (`VoucherScreen.tsx` twice, `dispatch/DispatchPanel.tsx`,
   `admin/BulkEntry.tsx`, `banking/brsImport.tsx`) [C]. `overlayUp()` does not list `.overlay`: the shield itself is what blocks the window [C `ui/overlays.tsx`].
4. **Only a right chevron icon exists** (`Icon n="chevron"` is `m9 6 6 6-6 6`) [C `ui/icons.tsx`]; a left chevron is a CSS rotation of it, so `icons.tsx` need not change.
5. **The light scheme overlay already restates `.navtoggle:hover`** (`html:root[data-scheme='light'] .navtoggle:hover { background: var(--hover-canvas) }` [C `styles/light/shell.css:33`]); the Phase 26 gotcha says a base `:hover` form that the light prefix does not
   restate is silently lost, so the Back button needs the same line.
6. **`RAIL_NEVER_HIDE = ['Esc', 'Ctrl+A']`** [C `core/uiPrefs.ts`]: the rail's Quit can never be hidden by Settings. The top-bar layout (`TOP_ITEM_IDS`: company, data, exchange, import, export, share, print, help) is a closed list that the Go To pill, the +, the theme buttons and the gear are outside of; **Back must be outside it too**
   (it is the pointer's escape hatch, like `Ctrl+,` and Go To are for the keyboard) [C].
7. **Wave 3 has already edited the files this touches**: `widgets.tsx` (+7: the Share item), `router.tsx`, `overlays.tsx`, `store.ts`, `VoucherScreen.tsx` (+341), `electron/main.cjs`, `electron/preload.cjs` [C `git status`, `git diff --stat`]. **No slice here may start before wave 3 commits.**
8. **The filter's Esc and the report's Esc already stack correctly in places** (a popover closes before the screen pops): untouched.
9. **Dialogs whose Esc records a decision** [C `VoucherScreen.tsx`, read in the second pass over all 24 `Box` and 17 `ListPrompt` sites]: `TaxPrompt`'s Esc runs `onDone(null)` and its own help line says "Esc to skip" (the withholding is skipped and the save goes on); the **Nature of payment** `ListPrompt` (`pickAsk`) says "Esc: no TDS on it" and its `onCancel` sets `tdsChoice.current[ledgerId] = TDS_SKIP`. The Bank Allocation panel's `onCancel` skips an advisory allocation and moves the walk on. Every other `onCancel` of the 17 `ListPrompt` sites is neutral (`setX(false)`, `setPick(null)`, `st.pop()`, `onClose`), and the Esc of every other `Box` dialog is a cancel (`onCancel`, `onDone(null)`, `onDone(undefined)` "leave as it was").
10. **`beforeunload`**: no handler anywhere in `src/`, `electron/` or `index.html` [C grep]; typed work is lost to any navigation away from the page.
11. **An Accept in flight**: `VoucherScreen.save` awaits `reserveVoucherNumber` (the cloud, up to 15 s, behind `numberBusy`) and then `saveVoucher`, with no check that the screen is still mounted; it then calls `st.pop()` for an alteration or duplicate, and for `afterAccept === 'list'` [C].
12. **Seventeen rail `Esc` buttons in 13 files are labelled Back or Back to List** and go up one internal level (`billofsupply`, `Snippets`, `SyncConflicts`, `ImportData`, `OperatorSignIn`, `form16a`, `chequeLayoutEditor`, `cheque`, `brsImport`, `payroll/masters`, `reminders`, `PasswordBits`, `SplitCompany`); of the 120 rail Esc buttons, 96 are a literal `st.pop()` and about 24 are closures over screen state [C grep; the review counted 16].
13. **Not re-measured:** the top bar's width budget at 1024 px (the reviewer measured its items at about 956 px, the shedding group at about 530 px). `railtop.css` confirms the structure (`.brand { flex: none }`, `.topitems` sheds), not the numbers.

**Where I differ from the audit:** it recommended a second Back "at the start of the breadcrumb or in the page header"; I take its own conclusion (the top bar only, one control: two Backs on one screen is worse than one). It put the history shim at "phase 2, optional"; I agree and design it as a **sentinel**, not one history entry per screen, because the per-screen design needs `history.go(-n)` bookkeeping that can race and that Chromium's history-manipulation intervention makes brittle (10).

---

## 1. The control

### 1.1 Placement and markup

`TopBar` renders `<BackButton />` **between the hamburger and the brand**, in a new file `src/ui/backNav.tsx` so `widgets.tsx` gains one import and one element:

```tsx
export const BackButton: React.FC = () => {
  const st = useBackState();                       // { show, label, full, enabled }
  if (!st.show) return null;                       // hidden, not disabled (1.3)
  return (
    <button type="button" className="backbtn" data-testid="back" aria-disabled={st.enabled ? undefined : true}
      aria-label={`Back to ${st.full}`} title={`Back to ${st.full}`}
      onMouseDown={(e) => e.preventDefault()} onKeyDown={ownActivate} onClick={() => requestBack()}>
      <Icon n="chevron" size={14} style={{ transform: 'rotate(180deg)' }} />
      <span className="backbtn-t">{st.label}</span>
    </button>
  );
};
```

`onMouseDown` preventDefault: the click must not take the cursor out of a voucher field the person was typing in (the hamburger's reason). `ownActivate`: Enter and Space belong to this button, not to the screen's key stack (`nav/ownKeys.ts`).
It is a real `button` in the Tab order (the hamburger precedent), so a keyboard user can reach it, but **no key is bound to it**.

**The tooltip does not name Esc.** The first draft's "Back to X (Esc)" was untrue: Esc is the screen's own key. It goes up one INTERNAL level on the 17 screens whose rail "Back" is such a button, and on a new voucher Esc raises "Quit ?" while Back raises "Leave this entry?". No keycap is shown. (The app writes `keyLabel('Esc')`; `keyLabel('Escape')`, which the draft used, would print "Escape".)

### 1.2 Geometry (the frozen three-column grid is not touched; Back is outside `.main`)

| Item | Rule |
|---|---|
| Position | `.topbar` is a flex row above `.main`; the Back button is a **flex-none child after `.navtoggle`**. `.main { 186px 1fr 152px }`, `.app.nv-wide` (212px) and `.app.navhid` (0 column) are not read or written, and `visual.assert.ts` (which reads the base `.main` rule) is unaffected. The `navhid` bug (hiding a grid child with `display: none` shifted the columns) cannot recur: nothing here is a grid child |
| Width | the label is cut with an ellipsis at `max-width: calc(24ch * var(--ui-k, 1))`; **below 1100 px only the chevron shows** (about 28 px instead of about 70; the destination moves to the tooltip and the accessible name); the control never wraps. `.topitems` already sheds into "..." when it runs out of room. **The reviewer measured the bar at 1024 px and 100 % text at about 956 px with no slack, so even the bare word "Back" would push Help behind "..." where nothing sheds today** [reviewer-measured, not re-measured here]: that is why the narrow form is the chevron alone. The recipes and suites that pin the bar at that width are named in 11.2 and are re-run (re-recorded only where they pin shedding): this is an intended consequence, not a discovery. Back takes room from the shedding group first, never from the Go To pill, the + or the gear (`min-content`, `flex: none`) |
| Text size | `font-size: var(--fs-sm)`; any literal size goes through `sz()` or `calc(Npx * var(--ui-k))` (Phase 27 gotcha); verified at 90 to 150 % with the "text drawn over text" probe the Phase 27 matrix lacked |
| Tokens | no hex: reuse `.navtoggle`'s colours (`--dim`, `--ink`, `--sunk`, `--radius-sm`, `--focus-ring`); the **light overlay** gets `html:root[data-scheme='light'] .backbtn:hover { background: var(--hover-canvas) }` (5 above); the accent contrast matrix is re-run (the control uses no accent) |
| Hold-Alt reveal, the `?` sheet | none: no chord |
| Print | the report tools clone the report table, not the shell, so the top bar is not on paper [unverified here: the print recipe confirms it with the button showing] |

### 1.3 States

| State | Condition | Draws |
|---|---|---|
| **Hidden** | stack depth 1 (the Gateway root, the startup company picker, first-run create), or the top is a gate screen (`isGateScreen`: company passcode, operator sign-in; they have their own "Back" and "Companies" buttons [C `CompanyUnlock.tsx`, `OperatorSignIn.tsx`]) | nothing (no layout shift: it is a flex child) |
| **Enabled** | depth at least 2, top not a gate, no guard question pending | chevron, "Back" or "Back to <name>" |
| **aria-disabled, dimmed** | a "Leave this entry?" question is pending (`useNavGuard.pending`), **or an Accept is in flight (a busy probe, 2.2)**, or a modal that does not shield (`.cmdp-overlay`, `.qc-overlay`, `.share-overlay` via `overlayUp()`) is up | same, dimmed; a click does nothing |
| **Under a dialog** | an `.overlay` shield is up | unreachable by mouse (the shield covers the bar): the dialog's own Close (5) is the way out |

`useBackState()` is a hook over `useStore((s) => s.nav)`, `useNavGuard((s) => s.pending)` and the name resolver; it recomputes only when the stack, the question or a name changes.

### 1.4 The label: `backLabel(scr)` (pure, `src/ui/backNav.ts`)

The crumb and `labelFor` cannot name a destination for this: the crumb prints "Menu" for every submenu, "Select" for every master list, "Voucher" and "Register" for the families [R, C `router.tsx` `CRUMBS`; `state/recents.ts` `labelFor`], so "Back to Menu" would be the result.
A purpose-built function with a test that fails on a bad answer:

| Screen | `backLabel` | Example |
|---|---|---|
| `menu` `gateway` | "Gateway" | Back to Gateway |
| any other `menu` | the menu's own title, shortened where it is long ("CRM", "HR") | Back to Display More Reports |
| `voucher`, new | "<Type> entry" | Back to Sales entry |
| `voucher`, alteration (`editId`) | "<Type> (editing)" (`heldWords`) | Back to Sales (editing) |
| `register` | "<Base> Register" | Back to Sales Register |
| `pickMaster` | "<Kind> list" | Back to Ledger list |
| a master form | `labelFor` ("New Ledger") or "Ledger (editing)" | |
| a screen with a master qualifier (`ledgerVouchers`, `groupSummary`, `stockItemVouchers`, `costCentreVouchers`, `billsOfLedger`...) | the qualifier when known, else the title ("Sharma & Co", "Sundry Debtors") | Back to Sharma & Co |
| `reportView` | the view's name | |
| anything else | `labelFor(scr).title` | Back to Day Book |

**The test (`backnav` suite):** for every key of `SCREEN_AREA` plus every `MenuId`, `backLabel` returns a non-empty string of at most 40 characters that is **not** "Menu", "Select", "Pick Master", "Voucher" or "Register" on its own. A new screen kind that falls to the
default title is caught the day it is added. The menu titles are checked against `MENUS` in `MenuScreen.tsx` by a source read (the table is module-private; `MenuScreen.tsx` is wave 3's, so it is not edited).

---

## 2. Behaviour

### 2.1 `planBack` (pure, in `core/navPlan.ts`, beside `planNavV2`)

```ts
export type BackPlan =
  | { kind: 'none'; why: 'root' | 'pending' | 'gate' | 'busy' }   // nothing to do, and WHY: only 'root' means "there is nowhere to go back to"
  | { kind: 'pop'; to: number }                           // stack becomes stack.slice(0, to + 1)
  | { kind: 'ask'; then: { popTo: number } | 'replace'; target: Screen }       // the top holds unsaved work: ask first
  | { kind: 'reopen'; screen: Screen };                   // depth 2 with a trail entry: replace the stack by it, as a rail click would

export function planBack(
  stack: readonly Screen[], dirtyTop: boolean, trail: readonly Screen[], pending: boolean, busyTop: boolean,
): BackPlan;
```

| Stack | Trail | Top dirty | Plan |
|---|---|---|---|
| depth 1 | any | any | `none`, why `root` |
| the top is a gate screen | any | any | `none`, why `gate` |
| a guard question is pending (`pending`) | any | any | `none`, why `pending` |
| an Accept is in flight on the top (`busyTop`) | any | any | `none`, why `busy` |
| depth at least 3 | any | no | `pop` to `n - 2` |
| depth at least 3 | any | yes | `ask { popTo: n - 2 }`, target `stack[n - 2]` |
| depth 2 | has an entry that is recordable and whose `screenKey` differs from the top's | no | `reopen` it |
| depth 2 | same | yes | `ask 'replace'`, target the trail entry |
| depth 2 | empty | no | `pop` to 0 (the Gateway) |
| depth 2 | empty | yes | `ask { popTo: 0 }`, target the Gateway |

`dirtyTop` is `isDirtyAt(stack, n - 1)` (the registered probe, else the conservative rule), exactly what the left rail asks. **Only the top is asked about**: popping to `n - 2` discards nothing else. A voucher or master below a clean top
is kept mounted by the pop (its cursor, filter, marks and typed state intact: [R] section 6).

`busyTop` comes from a second tiny registry beside the dirty probes (`state/dirty.ts` gains `registerBusy` and `busyAt`, same slot index, same pattern; B-1 adds it). The voucher screen registers `numberBusy.current !== 0` in B-2a's one-line pass of `VoucherScreen.tsx`.

### 2.2 `requestBack()` (in `ui/navigate.ts`, beside `requestNav`)

```ts
export function requestBack(): void {
  const st = useStore.getState();
  const stack = st.nav;
  const plan = planBack(stack, isDirtyAt(stack, stack.length - 1), trailOf(), !!useNavGuard.getState().pending, !!busyAt(stack.length - 1));
  switch (plan.kind) {
    case 'none': return;
    case 'pop':    useStore.setState({ nav: popToStack(stack, plan.to) }); refocusRevealed(); return;
    case 'reopen': requestNav(plan.screen, 'dest', undefined, { fromBack: true }); return;     // replace; not recorded in the trail
    case 'ask':    useNavGuard.getState().ask({ target: plan.target, source: 'dest', then: plan.then, losing: [heldWords(stack[stack.length - 1])], verb: 'back', focusEl: focusBeforeAsk() }); return;
  }
}
```

State transitions, step by step:

1. **Rights are not re-judged for a `pop`** (the screen below was already open); a `reopen` goes through `requestNav`, which judges `screenDenial` first and raises a refusal without touching the stack, exactly as for a rail click.
2. **`pop`** sets `nav` to the prefix and calls `refocusRevealed()` (a macrotask later, once the revealed screen is visible: the existing fix for "the first characters after a rail click were dropped"). The screen below was kept mounted, so nothing remounts.
3. **`ask`** stores a `PendingNav` with a new optional `verb: 'back'` and `losing` naming only the top ("Sales (editing)"). `NavGuardHost`'s second line becomes **"You have unsaved work in Sales (editing). Going back to Day Book would discard it."** (the text for the other sources is
   unchanged). **Stay** (armed first, as today) calls `stayPut()`: nothing changes, not the stack and not focus. **Leave** calls `leaveAndGo()`, which already re-validates a `popTo` slot against the target's `screenKey` and falls back to a replace.
4. **Stale question:** `NavGuardHost` already clears a pending question when the stack changes under it; unchanged.
5. **Esc keeps its own prompts.** A voucher in creation shows the **nav-guard** prompt ("Leave this entry?", Stay armed) when left by Back, and keeps its Tally-faithful **"Quit ?" (Yes armed)** when left by Esc or its rail Quit. That asymmetry is deliberate (B7) and is the one place the two differ; the owner may veto it (Q2).
6. **Never an internal level.** Back is exactly one stack screen, so its label stays true. A screen with an internal level a mouse user needs (a register's month list) already has a visible control (its "By month / All vouchers" toggle); an optional hook `useBackTarget({ label, run })` could later let such a screen
   say "Back to By month", and is **not** built here.
7. **An Accept in flight (`why: 'busy'`).** `VoucherScreen.save` awaits the cloud's numbering for up to 15 s and then saves. A Back pressed meanwhile would ask (the voucher is dirty), the person would answer Leave and the screen would unmount, the save would still complete (a numbered voucher the person chose to discard), and its trailing `st.pop()` would remove one more screen. So Back is a no-op and drawn aria-disabled while the busy probe is true; and B-2a's one-line edit of the same function guards the post-save `st.pop()` and `st.push` with a mounted flag (a rail click during the await had the same exposure and gains the same protection).

### 2.3 The session trail (B3)

After a left-rail click the stack is `[Gateway, X]`; "back" to the Gateway is useless to someone who lives in the rail. So `requestNav`'s **replace** moves remember the screen they replace.

```ts
// src/state/trail.ts (a tiny zustand store; memory only, never persisted)
interface TrailState { entries: Screen[]; noteLeft(stack: readonly Screen[], clean: boolean): void; take(): Screen | undefined; clear(): void }
export const TRAIL_MAX = 10;
```

| Event | Effect |
|---|---|
| a `dest` **replace** from depth at least 2 (a rail row, `+`, a pin, a favourite slot, the universal `+`), top is **clean** and `isRecordable` (state/recents.ts) | push the top screen on `entries` (de-duplicated by `screenKey`, the newest last, at most `TRAIL_MAX`) |
| a replace from depth at least 2 whose top is **dirty** and the person chose Leave (`leaveAndGo`) | **not recorded** (its work is gone; reopening it blank would pretend otherwise) |
| a replace from depth 1 (the person was at the root) | `clear()` (the root is the start of the trail) |
| the stack becomes depth 1 by any route (Esc to the Gateway, Quit, Back, a company switch) | `clear()` (a store subscription in `trail.ts`, so `AppShell` is not edited) |
| `reopen` (Back at depth 2) | `take()` removes the entry; the move is **not** recorded (`fromBack`), so Back never ping-pongs, and there is no Forward |
| an entry that is an alteration (`editId`) or `dupId`, a picker, a print screen, a gate | never recorded (`isRecordable`) |

**A worked sequence.** Gateway, then the rail's Day Book (`[G, DayBook]`, trail empty because the replace came from depth 1), then the rail's Receivables (`[G, Receivables]`, trail `[DayBook]`), then the rail's Balance Sheet
(`[G, BalanceSheet]`, trail `[DayBook, Receivables]`). Back: reopen Receivables (`[G, Receivables]`, trail `[DayBook]`). Back: reopen Day Book (trail empty). Back: the Gateway (`pop` to 0, which clears the trail). A reopened screen **starts fresh** (cursor, scroll, filter
gone, because the replace unmounted it; the period and F12 options are global and hold). Keeping replaced screens mounted would rebuild the eight-deep stack Phase 26 removed, so it is not done.

### 2.4 What Back never changes

Period, F12 options, added columns and the basis are **global stores** (`st.setPeriod`, the F12 stores), not per-screen state: changing the period in a drill level changes the screen underneath, and applying a saved view rewrites them [R, C `reportViews/apply.ts`]. **Back cannot restore them and does not pretend to.**
A saved view that moved the period says so in its own toast (the grid's job), not in Back.

---

## 3. What each level restores (and the two gaps to close)

**Restored today by `pop`** (the screen below stays mounted, [R] section 6): the keyboard cursor row, the row-filter text and narrowed rows, marked rows, an open internal level that was merely covered, the Dashboard's own scroll and as-at choice. **Not restored:** the **mouse-selected row**
(a double-click opens a row but never moves the keyboard cursor), the **scroll offset** on report screens (a "cursor row just visible" rule replaces it), the global period, F12 and columns, and an unsaved edit in the screen being left (lost silently for alterations and masters by Quit and Esc; asked by Back, 2.2).

| Gap | Fix (B9) | Where |
|---|---|---|
| A double-click or single click on a data row does not move the cursor, so on return the report re-reveals the cursor row (`revealCursorRow` in `useRowNav`) and a Day Book scrolled to the bottom came back near the top [R] | **a click on a data row sets the row-nav cursor before the drill**: `navLink` (`useRowNav`'s published channel) gains `select(i)`, and the shell calls it from **one delegated click handler**, not by editing the five `onDoubleClick` sites; Enter-to-open and the keyboard path are unchanged | the grid's `ListTable` (B-grid S1) if it is built, else `ReportShell` in `reports.tsx` after B-grid's Gate 1 |
| Scroll position | record `scrollTop` of `.scr-body` when a stacked screen stops being the top and restore it when it is the top again (`AppShell`'s wrapper knows both moments); `revealCursorRow` then runs **only if the cursor row is off screen**. The Dashboard already behaves this way [R] | same two homes |

A recipe `back-place` proves it: scroll a Day Book to the bottom, double-click a row, Back: the same row and offset.

**Group Summary rows have no mouse drill (audit F10).** `reports.tsx` gives the Trial Balance rows an `onDoubleClick` but the Group Summary's two tables have none, so a mouse-only person stops at "Sundry Debtors" [R, V two double-clicks did nothing]; the Dashboard's Sales and Purchases cards open exactly that report (Phase 28). The first draft carried the cursor fix (B9) and not this one. B-4 adds the missing handler on the two tables (one `onDoubleClick={() => drill(row)}` each, as the Trial Balance has) until B-grid's `ListTable` supersedes them, and the delegated click handler is also what gives a row that Enter would open a double-click open. Recipe `back-drill` clicks through Group Summary with the mouse.

---

## 4. Special screens and flows

| Screen or flow | Back |
|---|---|
| **Gateway root**, the startup company picker, first-run company creation | hidden (depth 1). The root's own "Quit" menu item and Esc keep asking "Quit ?" and calling `window.close()`, which in a normal browser tab does nothing for a tab the script did not open [unverified] |
| **The 15 Gateway submenus** (Create, Alter, Display More Reports, Account Books...) | **Back works** ("Back to Gateway", or to the report the person was in when a top-bar item pushed the menu over it: the stack after top-bar "K: Company" over a Day Book is Gateway, Day Book, Menu). This closes the audit's P0 for submenus with no rail change (B10) |
| **Gate screens** (company passcode, operator sign-in) | hidden; their own buttons stay |
| **Dashboard** | enabled. In Customize (edit) mode Back leaves the Dashboard like any push; the board's edit mode ends when another screen covers it (`useIsTop`, Phase 28), and its panel keeps "Cancel changes" and "Done" |
| **Settings** | enabled ("Back to <where you were>") |
| **Custom Fields editor** (and its Starter-set and Fill-blanks panels) | enabled; the editor registers a dirty probe (true while its docked panel's draft differs from the stored row, while the paste box holds text, or while a Starter or Fill preview holds choices: custom-fields spec 7.6), so Back and a rail click ask "Leave this entry?", and Esc on a changed panel asks "Discard changes?" |
| **Settings > Workspace > Custom-field entry rules** (custom-fields spec AD.1.5, added 2026-10-22) | enabled; the panel holds a draft of the company's rule for old records, so it registers a dirty probe (true while the draft differs from the stored policy), Back and a rail click ask "Leave this entry?", and Esc on a changed draft asks "Discard changes?" (No armed) |
| **The five analysis screens** (Summary, Income and expense, Bills, Balances and Stock by field; custom-fields spec AD.2.4, AD.8) | enabled; plain report screens, no typed work, no probe; their F12 option panels are `Box`es and pass `onClose` (B-2a's scan) |
| **Stock by field > Opening by variant** (custom-fields spec AD.3.7) | enabled; a typed split that is not yet saved is dirty work, so the panel registers a probe and is a `Box` with `onClose` |
| **Voucher entry with an Accept in flight** | Back is aria-disabled (2.2) |
| **Create New Report** (a builder session) | enabled; the session's bar registers a dirty probe, so Back asks like the rail does ("Report not saved" otherwise) |
| **A saved-view launcher** (`reportView`) | it pops itself and re-requests its target as a `dest` move; Back from the opened report goes to the screen the launcher replaced or the trail's last entry |
| **Voucher alteration with PgUp / PgDn** | the neighbouring voucher replaces the one on screen (depth unchanged), so Back goes to where the alteration was started, not to the previous voucher. The rail's "Prev/Next" button is **inert today** [R] and gets its `onClick` in B-2's `VoucherScreen.tsx` pass (a one-line edit of the same function the key calls, made while that file has no other writer) |
| **Print screens, pickers** | enabled; a `reopen` never lands on one (not recordable) |
| **A dialog or picker is up** | Back is unreachable by mouse (the shield); the dialog's own Close (5) is the way out |

---

## 5. The dialog trap (B5): `Box`, `ListPrompt`, the hand-built overlays

### 5.1 The API change in `widgets.tsx`

```tsx
export const Box: React.FC<{
  title: string;
  children: React.ReactNode;
  onClose: () => void;                       // what Esc does for this dialog. Where Esc records a DECISION (TaxPrompt: skip this deduction) the caller sets scrim 'ignore' and names it in closeLabel ('Skip TDS'). OPTIONAL in B-2a, required in B-2b (12.1)
  scrim?: 'close' | 'ignore';                // a click outside the box; default 'ignore'
  closeLabel?: string;                       // default 'Close'; 'Cancel' where the dialog confirms
  actions?: { label: string; onClick: () => void; primary?: boolean }[];   // an OK where Enter is the only way to confirm today
}>;
```
`onClose` is **required once B-2b lands** (24 compile errors, one per call site); before that B-2a fills every site with it optional in the type, and a source scan in `box-close` fails any `<Box` without it, so no dialog written in between can forget it. The close button is a real `button` in the title bar (the Settings key recorder's `Use, Remove, Cancel` buttons are the in-repo pattern, "for someone using the mouse or a finger" [C
`KeyRecorder.tsx`]; the print dialogs' "Cancel (Esc)" another [C `PrintVoucher.tsx`]). **The close button and Esc call the same function**, so there is one close behaviour per dialog, not two, **and where that function records a decision its button says so**. `ListPrompt` always draws a button under its list (**Cancel**, or the `cancelLabel` its caller gives) and, unless its caller sets `scrim: 'ignore'` (both props optional, defaults as today), closes (calls `onCancel`) on a
mouse-down on its **backdrop** (a `mousedown` whose target is the backdrop element itself, so a click inside the list never closes it). A picker holds no typed work other than its filter text, **but not every `onCancel` is neutral: the Nature of payment picker's records "no TDS on this" (5.2)**.

### 5.2 Which dialogs close on a click outside (the rule, then the table)

**Rule:** a click outside closes a dialog only if closing loses nothing a person typed **and decides nothing**. Everything else closes only through its button, so a stray click never discards a half-typed date, key or reason. (Sites and lines are as of the wave-3 tree.)

| Dialog (site) | `scrim` | `closeLabel` | `actions` |
|---|---|---|---|
| Report Configuration F12 (`reportConfig.tsx`), Voucher Configuration F12 (`VoucherConfig.tsx`), Basis of Values (`reportBasis.tsx`), Filter rows (`header/FilterPanel.tsx`), Saved Views list (`reportViews/useReportViews.tsx` first Box, while it shows the list; a name being typed is its second Box) | `close` (they apply live) | Close | none |
| **Change Period** (`DatePrompt`'s sibling `PeriodPrompt`, `widgets.tsx`), **the voucher Date** (`DatePrompt`) | `ignore` (typed dates) | Cancel | **OK** (runs the Enter handler: a mouse-only person could not confirm these at all today) |
| New Column (`reportColumns.tsx`), the Save view dialog (second Box of `useReportViews.tsx`), Batch Details, Bank Allocation (its Esc skips the allocation and continues the walk: advisory, so Cancel stays), `ReasonPrompt` (cancel a voucher), GST details and Applicable-from (`gstLine`, `gstDetails`), GST rate on an advance (`advances.tsx`), Shipping Bill (`gstr1.tsx`), the 26Q filed-on date (`tds/interest.tsx`), Challan details (`tds/challanPanel.tsx`), Cheque Book and Cheque (`banking/*`) | `ignore` (typed fields) | Cancel | OK where Enter confirms and no button does |
| API keys: New API Key, **Copy your API key now** (`admin/ApiKeys.tsx`) | `ignore` (the key is shown once: a stray click must not lose it) | Close | none |
| The Settings key recorder (`settings/KeyRecorder.tsx`) | `ignore` | has its own buttons | has its own |
| 16 of the 17 `ListPrompt` sites | backdrop closes | Cancel | none |
| **TDS and TCS: `TaxPrompt`** (`VoucherScreen.tsx`) | `ignore` | **Skip TDS** (or **Skip TCS**): closing skips this deduction and the save goes on, exactly as Esc does, so the button says so | an **OK** that runs the Enter handler, labelled "Deduct TDS" or "Collect TCS"; its nested nature `ListPrompt` is neutral |
| **TDS: the Nature of payment picker** (`pickAsk`, `VoucherScreen.tsx`) | `ignore` (a stray click must never decide "no TDS") | **No TDS on this** (its `onCancel` records `TDS_SKIP`) | none |
| Hand-built `.overlay` (`VoucherScreen.tsx` F10 **Change Voucher Type** and the second one near its end, `dispatch/DispatchPanel.tsx`, `admin/BulkEntry.tsx`, `banking/brsImport.tsx`) | a picker closes on a click outside; a panel with typed fields does not | Close or Cancel | the same rule |

**Audit of the decisions (second pass).** Reading the Esc handler and `onCancel` of all 24 `Box` and 17 `ListPrompt` sites: `TaxPrompt` (Esc = skip the deduction) and the Nature picker (Esc = `TDS_SKIP`) record a statutory decision; the Bank Allocation panel skips an advisory allocation and moves the walk on; every other site is a cancel (`setX(false)`, `setPick(null)`, `onDone(null)`, `onDone(undefined)` "leave as it was", `st.pop()`). The first draft's table put the TDS dialog under `scrim: ignore` with a button labelled **Cancel**, and called every `ListPrompt` "backdrop closes": a mouse-only person paying a contractor would have posted the voucher with no TDS deducted from a stray click, or by pressing a Cancel that did not cancel. The `box-close` suite carries this table as data; a new `Box` or `ListPrompt` whose Esc records a decision must be added to it with a label, or the suite fails.

Go To, the universal `+` and the `?` sheet **already** close on a click outside, and the Settings key recorder has real buttons [R]: untouched. The "is an OK needed" column is a mouse-completeness fix beyond Back, limited to the two dialogs every report and voucher opens (F2); the other dialogs have their own confirm
buttons or keys and are the owner's later question, not widened here.

---

## 6. The rail's `Esc Quit` runs what Esc runs (B6)

`ButtonPanel` already post-processes every rail (`applyRail`, hide, rename, order). It gains one rule: a `Btn` whose `k === 'Esc'` ignores its own `onClick` and runs **`runEscape()`**, a new function in `keys.ts`:

```ts
/** Press Escape the way a person would: a synthetic keydown on the element that has focus (or the body), so an element-level handler (the filter box) sees it first and then the key stack does. */
export function runEscape(): void {
  const target = (document.activeElement as HTMLElement | null) ?? document.body;
  target.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', code: 'Escape', bubbles: true, cancelable: true }));
}
```
The rail's `.pbtn` gets `onMouseDown={(e) => e.preventDefault()}` **for the Esc button only**, so a click leaves the field focused (today a click blurs it, and an element-level Esc handler would not see the key).

What this fixes in one place [R]: a register month list (the key goes up to the month matrix, the button left the whole register), Stock Query (key re-opens the picker, button leaves), a Day Book with marked rows (the key clears the marks, the button leaves and drops them), and every screen where the two drifted, with **no** edit of the 99 call sites. The 99 hand-written `onClick: () => st.pop()` become dead code; a **ratchet** in the new suite counts them and may only fall.
The Gateway's own Quit (it asks "Quit ?") is unchanged: it is not a rail `Esc` button. `RAIL_NEVER_HIDE` keeps the button on every rail.

**Why a synthetic event is safe here:** the keyboard layer's single `window` `keydown` listener sees it, `combo(e)` reads `e.key`, and Escape is a reserved named key (never remapped by a person's chord table, `RESERVED_NAMED_KEYS`), so the translation layer cannot interfere; the Proxy note in `keys.ts` ("why not a synthetic KeyboardEvent") is about re-dispatching a **remapped** press, which this is not [C `keys.ts`].

**Proof, screen by screen (a source scan alone cannot show that the key does what the button did).** Of the 120 rail `Esc` buttons, 96 are `st.pop()` and about 24 are closures over screen state (`ImportData`'s `mapApi.current?.cancel()` and `.back()`, `SyncConflicts`' `setOpenKey(null)`, `Security`'s `setEditing(null)`, form16a, cheque, brsImport...). The `backnav-esc` suite therefore has a **table, one row per rail Esc button** (file, label, and the Escape-key handler it must reach), built by a source scan that fails when a rail Esc button is in no row; a screen whose Escape handler is gated (`if (busy) return`) is listed with the matching `dis`; and the `back-register` recipe clicks each of the 24 closure buttons and presses Esc on the same state and compares the result. `ButtonPanel` keeps `b.dis` precedence (a disabled button does nothing). Where a screen's key and button truly differ by design, the row carries a reason and that `Btn` says `keepClick: true` so `ButtonPanel` leaves its `onClick` alone (none is expected; the table is how one would be found).

### 6.1 The 17 rail buttons labelled Back

Seventeen rail Esc buttons in 13 files read Back or Back to List and go up one INTERNAL level (the previous step of a wizard, the list from a detail). The first draft's section A said two Backs on one screen is worse than one, and then created that on these screens. The top-bar control is a different thing and is made distinguishable: it always names a destination ("Back to Gateway") where there is room, it never collapses to the bare word at any width (below 1100 px it is the chevron with the destination in its tooltip and accessible name), and the guard sentence names exactly what is lost ("unsaved work in Import Data"). **Residual, said plainly:** below 1100 px on those 13 screens the two controls are told apart by position and tooltip, not by words. The `back-register` recipe lists those screens and checks both controls on each.

---

## 7. Forward: decided no

A pop unmounts the screen above, so a Forward button would reopen it **fresh**, which is exactly what clicking where you came from does; and a stack that keeps "forward" screens mounted is the pile-up Phase 26 removed. Quick access > Recent (7 shown, `state/recents.ts`) is the existing "get me back to what I was doing", and the trail (2.3) covers the rail case. Not built, and
the owner may veto (Q4). Browser Forward after a browser Back is a no-op in the phase-2 design (10).

---

## 8. Keyboard parity and the registry

- **Esc is the keyboard's Back and is enough.** It is a reserved named key (`RESERVED_NAMED_KEYS`, never rebindable [C `core/shortcutRegistry.ts`]) and every screen handles it. **No chord is added**, so `core/shortcutRegistry.ts`, `zoho-scaffold.assert` and `shortcutprefs.assert` are untouched. `Backspace` is "Remove Column" on the Trial Balance and others, and `Alt+Left` is the browser's own Back in a tab (unverified whether a page can cancel it): neither is used.
- The `?` key sheet's existing Esc row gains "(or the Back button)" in its description (a prose edit in `ui/overlays.tsx`, wave 3's file, so after its commit).
- **Not hideable:** `TOP_ITEM_IDS` in `core/uiPrefs.ts` is unchanged and a suite assertion fails if Back appears in it or in `applyTop`'s input (the pointer's escape hatch).
- Focus manners: Tab order is hamburger, Back, brand (DOM order); a click never moves a field's focus; after a pop the revealed screen takes its keyboard back (2.2).

---

## 9. The frozen-layout constraints, in one table

| Thing | Touched? | Constraint kept |
|---|---|---|
| F-key rail (`.btnpanel`, column 3, 152 px) | **behaviour only** (6): no button added, moved, renamed or removed; the label stays "Quit" | the Tally keys and their order are frozen; Settings may hide a rail button but hiding never disables its key |
| `.main { 186px 1fr 152px }`, `.app.nv-wide`, `.app.navhid` | no | Back is a child of `.topbar` |
| `.topbar` and `.topitems` shedding | one flex-none child | `railtop.css` gets the sibling rule beside `.brand { flex: none }`; checked at 800, 1024, 1440 and 90 to 150 % text |
| Status-bar crumb and "Hide menu" | untouched | segments stay inert, by the standing decision (a segment click would pop several screens and bypass each screen's guard): `planBack` is exactly the guarded single step the decision lacked |
| Light overlay | one `:hover` rule | `html:root[data-scheme='light']` prefix; no hex in `:root` for a token dark does not override |
| `sidenav` suite (65 rows today), Go To golden (103 queries) | no | Back adds no menu row and no Go To command (it would re-rank queries: Phase 27 gotcha); the custom-fields spec adds one menu row, so the suite reads 66 after its CF-1 |
| `AppShell.tsx` | **no edit**: Back reads `st.nav`, `useNavGuard`, `overlayUp()`; the trail subscribes inside `state/trail.ts` | a new host would register a global handler; a button needs none |

---

## 10. Phase 2, optional: the browser's Back, thumb button and swipe (B8)

**Today** the app never writes a history entry (my own grep of `src/`, `electron/` and `index.html` finds no `pushState`, no `popstate` handler, no mouse back-button handler and no `app-command`; `index.html:21` has the one `replaceState`, for `?uireset=1`), so a real browser "back" (a mouse thumb button, a trackpad swipe, Alt+Left, Cmd+[) went to `about:blank` in the audit's probe and the person's
place in the app was lost [R, C]. In Electron nothing handles an `app-command` either, and its default is [unverified]. **Built only if the owner confirms mouse users reach for those gestures (Q3).** Two routes, because the shell differs:

### 10.1 The web: one sentinel entry, not one entry per screen

A history entry per pushed screen needs `history.go(-n)` bookkeeping on every pop that can race with a user's own presses, and Chromium's **history manipulation intervention** makes the browser's Back button skip same-document entries that were created without a user activation (and not those created after
interaction) [W: the Chromium doc the audit cites, https://chromium.googlesource.com/chromium/src/+show/08d828b920062e2b0be1f95b7d2416d163091745/docs/history_manipulation_intervention.md; MDN `popstate` and `pushState`: `popstate` fires on a user's Back or Forward and on `history.back()`, **not** on `pushState`]. So the design keeps **one** sentinel entry behind the current one:

| Event | Condition | Action |
|---|---|---|
| the **first trusted** `pointerdown` or `keydown` (a capture listener on `window`) | not armed | `history.replaceState({ tw: 'base' }, '')`, then `history.pushState({ tw: 'cur' }, '')`; armed. (Pushed **after** an activation, so the intervention cannot skip it) |
| `popstate` | `e.state?.tw === 'base'` (the person went Back) and a **dialog is up** (`.overlay`, a `ListPrompt`) | `runEscape()` (close the dialog), then re-push the sentinel: browser Back closes the top dialog first, as on a phone |
| `popstate` | `'base'`, no dialog, `planBack(...)` is **not** `none` | `requestBack()`, then `history.pushState({ tw: 'cur' }, '')` **at once**, whether it moved, asked or refused: the browser never actually left the page |
| `popstate` | `'base'`, and `planBack` is `none` **because `why === 'root'`** (the person is at the first screen) | **disarm and do nothing**: the next Back leaves the app, which is what a person at the first screen means |
| `popstate` | `'base'`, and `planBack` is `none` with `why` = `pending`, `gate` or `busy` | **re-push the sentinel at once and do nothing else**: a question is waiting, a gate screen is up or an Accept is in flight, none of which means "leave"; a second thumb press before the answer therefore cannot leave the page over typed work (the first draft treated every `none` as root) |
| `popstate` | `e.state?.tw === 'cur'` (a Forward from base) | re-arm; nothing else (**Forward is a no-op**, B4) |
| any later trusted activation while disarmed | the stack grew past depth 1 | arm again (as the first row) |

**A leave guard for the page.** Browser Back is not the only way out of a tab (a reload, a link, the close box), and there is no `beforeunload` handler anywhere today (A, item 10). While any registered dirty probe is true the web build installs a `beforeunload` handler (`event.preventDefault()`, so the browser shows its own generic confirmation) and removes it when none is. In Electron the equivalent (`will-prevent-unload`) is [unverified] and the OS window close is not guarded here. It ships with B-6, or earlier on its own if the owner agrees (question Q8).


`pushState` is called **only** from the activation handler, never from a store subscription, so no programmatic move (the saved-view launcher's replace, startup, a sync) can create an entry the intervention would skip; the `?uireset=1` cleanup in `index.html` runs before first paint and is unaffected. **Known limitations to recipe-test, not to hide:** a page reload keeps the
entries behind it, so one extra Back may be needed to leave after a reload; Firefox has a similar preference (`browser.navigation.requireUserInteraction`) [Ws, the audit's search summary] and Safari's behaviour is [unverified]; a company switch (which rebuilds the stack) disarms.

### 10.2 Electron: no history at all

`electron/main.cjs` (wave 3 has edited it) handles Windows' `app-command` (`browser-backward`) and macOS's `swipe` event on the `BrowserWindow` and sends an IPC message through `electron/preload.cjs`; the renderer's `src/ui/backHistory.ts` calls the same `requestBack()` (and `runEscape()` first when a dialog is up). No `pushState`, no popstate: the desktop shell never needs the web's trick. Electron 31 (Chromium 126)
cannot be driven in this environment (CLAUDE.md Phase 27), so this route is proved by a unit test of the handler and by the owner on a real Windows machine.

### 10.3 Proof

`history.back()` called from a recipe exercises the `popstate` path (it is **not** subject to the intervention); the real mouse thumb button can be simulated through the CDP `Input.dispatchMouseEvent` "back" button as the audit did. Recipes: browser Back at depth 3 pops one screen; at depth 2 with a trail reopens; at the root leaves; with a dialog up closes it; with a dirty top shows the guard and stays on the page; browser Forward does nothing.

---

## 11. Proving it

### 11.1 Pure suites (each ships deliberate mutations, each killed)

| Suite | Proves | Mutations it must kill |
|---|---|---|
| `backnav` (new) | **`planBack`** for every row of 2.1 (each `none` with its `why`; depth 1; a gate on top; a pending guard; clean and dirty tops at depth 2 and 3; a trail entry equal to the top is skipped; an unrecordable entry is skipped). **`backLabel` for every `SCREEN_AREA` key and every `MenuId`** (1.4: non-empty, at most 40 characters, never a bare "Menu", "Select", "Pick Master", "Voucher", "Register"; menu titles equal `MENUS`' by a source read). **The trail:** the recording rules of 2.3, the cap of 10, the de-duplication, `clear()` at depth 1 and on a company switch, `reopen` not re-recording, a dirty-then-Leave top not recorded. The guard sentence for `verb: 'back'`. `TOP_ITEM_IDS` does not contain Back | always pop to 0; ignore the dirty top; record a dirty top; keep the trail at the root; a label of "Menu"; Back listed in the top-bar layout; read every `none` as root; allow Back while busy |
| `backnav-esc` | **every rail Esc button equals the Esc key, one table row per button (120 rows: 96 `st.pop()`, about 24 closures; a rail Esc button in no row fails)**: `ButtonPanel` overrides a `k: 'Esc'` button's `onClick` with `runEscape()` unless the `Btn` says `keepClick` (source scan with a guard that it found the panel); `runEscape` dispatched through `pushKeyHandler` is seen as `combo(e) === 'Escape'` with the focused element as `target`; each row's screen has an Escape handler and, for the closures, the same effect as the button (the `back-register` recipe runs the pairs); a **ratchet** on the number of hand-written `st.pop()` Esc handlers (96 by a literal grep, 99 in the first count; it reads the scan's own count and may only fall) | restore `b.onClick` for Esc; dispatch on `window` instead of the focused element; add a rail Esc button that is in no row; a gated Escape handler with an ungated button |
| `box-close` | `Box` requires `onClose` (after B-2b the compiler proves it; before that a source scan fails any `<Box` without it); the close button calls it; a backdrop click closes only when `scrim === 'close'`; `actions` render; **the 5.2 table as data is matched against the callers (a new `Box` or `ListPrompt` not in the table fails), including the two decision-recording dialogs: `TaxPrompt` closes through a button labelled "Skip TDS" and ignores the backdrop, and the nature picker ignores the backdrop and its button reads "No TDS on this"**; every other `ListPrompt` draws Cancel and closes on a backdrop mousedown but not on a click inside | scrim closes an `ignore` dialog; a no-op close button; a picker that closes on an inside click; the TDS nature picker closing on the backdrop; a button labelled Cancel on `TaxPrompt` |
| `navstack` (extended) | the existing lines stay (the voucher `useDirty` literal and the Esc rule at `navstack.assert.ts:228-229` are **not** touched); a Back-versus-rail case table; **`customFields` joins the wired list (the editor registers a probe: a changed draft, a pasted option list, a Starter or Fill preview is dirty, an untouched panel is clean) and `planBack` asks for it; so do the entry-rules panel and the Opening by variant panel (a changed draft is dirty, an untouched panel is clean)**; the busy registry (`registerBusy` / `busyAt`) | |
| `visual` (extended) | the top bar with Back at 800, 1024 and 1440 wide, both densities, both schemes, 90 to 150 % text: no overflow, no cut label, the light `:hover`, the contrast pairs | |
| **Untouched, and the proof that no chord was added** | `zoho-scaffold.assert`, `shortcutprefs.assert`, the Go To golden, the sidenav literal table | |

### 11.2 Recipes (`scripts/shots/recipes/back.mjs`; real mouse events, never Accept; each in light and dark, compact and comfortable, and at 150 % text)

| Recipe | Steps |
|---|---|
| `back-submenu` | Gateway, Alter submenu, Back; and the top bar's "K: Company" over a Day Book, Back: 0 of 15 submenus lacked a control before |
| `back-drill` | Trial Balance, Group Summary, Ledger Vouchers, a voucher: Back three times; the screen below keeps its cursor, filter and marks |
| `back-dirty-master`, `back-dirty-alteration` | type a ledger name, Back: the guard shows, Stay changes nothing, Leave pops; the same for an altered voucher (the case Quit and Esc discard silently) |
| `back-rail` | Day Book, rail Receivables, rail Balance Sheet; Back, Back, Back follows the trail and ends at the Gateway |
| `back-dialog` | F12, F2, F10 Change Voucher Type, Stock Query's item list, Share: the visible Close works; a click outside closes the ones that hold no typed work and not the others |
| `back-tds` | a Payment to a contractor: `TaxPrompt`'s button reads "Skip TDS" and a click outside does nothing; the Nature picker ignores the backdrop and its button reads "No TDS on this"; neither posts a voucher without TDS from a stray click |
| `back-inflight` | slow the numbering call (the mock cloud's delay), Accept, click Back: it does nothing; the voucher saves once and the stack is not popped twice |
| `back-dirty-cf` | Custom Fields: paste options into a new field, Back: "Leave this entry?", Stay changes nothing |
| `back-register` | on a month list, the rail Quit equals Esc (up to the month matrix); the same on Stock Query and a Day Book with marked rows; **each of the 24 closure buttons against Esc on the same state, and on each of the 13 screens with a rail "Back": the rail button goes up one level, the top-bar control leaves the screen and asks when dirty** |
| `back-place` | scroll a Day Book to the bottom, double-click a row, Back: the same row and offset |
| `back-narrow` | 800, 1024 and 1100 wide at 100 and 150 % text: chevron only below 1100, the label truncates before the bar wraps above it; the Go To pill, the + and the gear keep their places; **and the recipes and suites that pin the bar at 1024 are re-run (`settings-shortcuts`, `settings-shortcuts-flow`, `crosscut-flow`, `dashboard-fix`, `dashboard-board.assert`, `uiprefs.assert`) and re-recorded only where they pin shedding** |
| `back-history` (phase 2) | 10.3 |

**Exit criteria:** all suites and recipes green; the 642-screen sweep (plus whatever the wave added) clean in three sweeps (light compact, dark compact, light comfortable) with zero console errors; `tsc` 0; `build` green; the Go To golden, `sidenav`, `zoho-scaffold` and `shortcutprefs` unchanged; a live bug-hunt on the sample company that walks the
audit's table (section 4 of `back-nav-audit.md`) and clicks Back on every screen kind it lists.

---

## 12. File ownership, order, slices

### 12.1 What each slice touches (one writer per file at any moment)

| Slice | What | Files | Effort [J, builder-days] |
|---|---|---|---|
| **B-1** | `planBack`, the trail, `requestBack`, `backLabel`, `useBackState`, `BackButton`, the guard wording, the top bar element | `src/core/navPlan.ts` (adds `planBack`), new `src/state/trail.ts`, `src/state/dirty.ts` (the busy registry, 2.1), `src/ui/navigate.ts` (adds `requestBack`, the trail notes in `requestNav`'s replace branch and `leaveAndGo`, a fourth `{ fromBack }` argument), `src/state/navGuard.ts` (`verb`), `src/ui/navGuard.tsx` (the sentence), new `src/ui/backNav.ts` and `backNav.tsx`, `src/ui/widgets.tsx` (**one import and one element** in `TopBar`), `src/styles/railtop.css`, `src/styles/light/shell.css`; suite `backnav`; recipes `back-submenu`, `back-drill`, `back-dirty-*`, `back-rail`, `back-narrow` | 3 to 4 |
| **B-3** | the rail Esc button runs Esc | `src/keys.ts` (`runEscape`), `src/ui/widgets.tsx` (`ButtonPanel`: the Esc rule and the mouse-down guard); suite `backnav-esc`; recipe `back-register` | 1 to 2 |
| **B-2a** | close affordances on dialogs and pickers, `onClose` OPTIONAL in the type and filled at every site | `src/ui/widgets.tsx` (`Box`, `ListPrompt` with `scrim` and `cancelLabel`, `DatePrompt`, `PeriodPrompt`); the 24 `Box` and 17 `ListPrompt` call sites in about 28 files (named in the 5.2 table); the five hand-built overlays; `VoucherScreen.tsx` (the two overlays, the inert PgUp/PgDn rail button, **the busy probe and the post-save mounted guard, 2.2**); a small `src/styles/` rule for `.box-x`; suite `box-close` (with the source scan that fails a `<Box` without `onClose`); recipes `back-dialog`, `back-tds`, `back-inflight` | 3 to 4 |
| **B-2b** | flip `onClose` to required | `src/ui/widgets.tsx`, and the dialogs CF-1 and B-grid S1 added meanwhile | 0.5 |
| **B-4** | cursor on click, scroll restore | **inside B-grid S1's `ListTable`** (preferred), else a standalone edit of `ReportShell` and `useRowNav` in `src/ui/reports.tsx` **after B-grid's Gate 1**; recipe `back-place` | 2 to 3 |
| B-5 | optional: an `Esc Quit` rail button on the Gateway submenus | `src/ui/MenuScreen.tsx` | 0.5, only if Q6 says so |
| **B-6** | optional phase 2: the browser's Back | new `src/ui/backHistory.ts`, `electron/main.cjs`, `electron/preload.cjs`; recipe `back-history` | 3 to 4, only if Q3 says so |

**B-1 to B-4 total: about 10 to 14 builder-days [J]** (B-1's sequence: B-1, then B-3, then B-2a, because all three edit `widgets.tsx`; B-2b last, once the dialogs of the custom-field and grid slices exist).

### 12.2 The shared header and crumb files, and the order they are touched in

| File | Touched by | Order |
|---|---|---|
| `src/ui/widgets.tsx` | B-1 (TopBar), B-3 (ButtonPanel), B-2 (Box, ListPrompt, DatePrompt, PeriodPrompt) | wave 3 commits first (it edited the file); then **B-1, B-3, B-2 in that order, one writer** |
| `src/core/navPlan.ts`, `src/ui/navigate.ts`, `src/ui/navGuard.tsx`, `src/state/navGuard.ts` | B-1 | after wave 3 (it has edited `store.ts`, `router.tsx`, `overlays.tsx`); the Phase 26 files are otherwise unmodified |
| `src/keys.ts` | B-3 | any time after wave 3 |
| `src/state/dirty.ts` | B-1 (the busy registry: `registerBusy`, `busyAt`) | any time after wave 3 (not modified there) |
| `src/ui/VoucherScreen.tsx` | B-2a (the two overlays, the inert rail button, the busy probe, the post-save mounted guard) | **before CF-2 starts** (custom-fields spec section 17), so the voucher file has one writer at a time |
| `src/ui/reports.tsx`, `src/ui/reports/*` | B-4 only, and only after B-grid Gate 1 | the grid owns these first |
| `src/ui/AppShell.tsx`, `src/ui/router.tsx`, `src/ui/pageHeader.tsx`, `src/ui/MenuScreen.tsx` | **not touched** (B-5 aside) | |
| `src/styles/railtop.css`, `src/styles/light/shell.css` | B-1 | |
| `src/ui/overlays.tsx` | B-6 aside, one prose edit of the `?` sheet's Esc row | after wave 3 |

---

## 13. Build order of everything pending (the same order as the custom-fields spec's section 17, compressed)

```
 wave 3 gates + COMMIT ─────────────────────────────────────  blocks every lane
   │
   ├─ B-1, B-3, B-2        top-bar Back, rail Esc, dialog close      about 8 to 11 d   FIRST: small, nearly disjoint from the rest
   ├─ CF-0                 custom-field model, entry-policy core, Dimension, new pure files   7 to 8 d   (was 5 to 6: widened 2026-10-22)
   ├─ A2-0 residual        report grid scaffold (table goldens, seams, chord)   about 3 d
   └─ CF-6a                backend rule, 5 to 6 d (deploy gated on the owner's VM secrets; carries the capability-keyed company-row rules for entryPolicy and ui)
        │
        ├─ CF-14 (entry policy)  ║  CF-9a (variant engine, no screens)  ║  CF-4b-lite (filter fields for four reports)   (disjoint, small)
        ├─ B-grid S0..S1   ║  CF-1 (editor, masters)       (B-4 lives inside B-grid S1)
        ├─ B-grid S2..S4   ║  CF-2 (voucher header)  ║  CF-4a (Summary, Bills, Balances: five screens registered once)
        └─ CF-3  ║  CF-5  ║  CF-9b..d (stock by field)  ║  CF-4b (catalogue integration, after B-grid S1 and S2)  ...
   (custom-fields spec section 17 is the source of truth; its release gates G1 and G2 are there)
```

**Where Back fits: first, because it is small.** It starts the moment wave 3 commits, it runs in parallel with the grid scaffold and the custom-field model (disjoint files), and its `VoucherScreen.tsx` edits are finished before CF-2 begins. **Parallel on disjoint files:** B-1 to B-3 (`widgets.tsx`, nav files, `keys.ts`) with CF-0 (new files, `types.ts`) with A2-0 (`savedViews.ts`, `reportActions.ts`,
`FilterBar.tsx`, `PageActions.tsx`, the registry) with CF-6a (`backend/`, one block of `sync.ts`). **Never in parallel:** two slices editing `widgets.tsx`, `VoucherScreen.tsx`, `types.ts`, the registry or the new-screen files.

**A cross-work finding the grid owner must act on:** wave-a2's `Alt+M` for the Columns panel is void (wave 3 gave `Alt+M` to Share; every Alt letter is taken: custom-fields spec section A item 2). Back adds no chord and needs none.

**B-2 ships in two steps (B-2a, B-2b; 12.1)**, which removes a dependency the first draft left unstated: the new dialogs of the custom-field slices (Starter sets, Fill blanks) and of B-grid S1 (the Columns panel) are written against an optional `onClose` that a source scan already enforces, and the type is flipped last. The custom-fields spec's section 17 carries the same item.

---

## 14. Questions that genuinely need the owner (each with the default taken)

| # | Question | Default |
|---|---|---|
| Q1 | Relabel the rail's `Esc Quit` to "Back" on every screen except the Gateway? (a visible default change for people who know Tally) | **no**: the label stays; the new control says Back |
| Q2 | Should the **Esc key** also ask before discarding an edited alteration or a typed master, as the left rail and Back do? (it changes what Esc means there) | **no**: Esc is unchanged; Back asks either way. A voucher in creation therefore shows "Leave this entry?" from Back and "Quit ?" from Esc |
| Q3 | Do your mouse users reach for the **thumb button or a swipe**? If yes, B-6 is built (10) | **no** until confirmed |
| Q4 | Is a mouse **Forward** wanted? | **no** (7) |
| Q5 | Back on **Settings** and in the Dashboard's **edit mode**? | yes on both (4) |
| Q6 | Add an `Esc Quit` button to the **Gateway submenus** as well? (whether TallyPrime's own submenus show one is unverified) | **no**: the top-bar Back covers them |
| Q7 | May a mouse user be asked **OK** and **Cancel** on the Change Period and Date dialogs (today only Enter confirms them)? | yes (5.2) |
| Q8 | Install the **`beforeunload` leave-guard** on the web build (a generic browser confirmation while any screen holds typed work), ahead of the rest of phase 2? | no: it ships with B-6, unless you want it sooner (10.1) |
| Q9 | Is the wording of the TDS dialogs' buttons right: **Skip TDS** on the deduction prompt and **No TDS on this** on the nature picker (each says what closing does)? | as written (5.2) |

---

## 15. What this specification did not verify

- **I did not re-run any probe.** The audit's live findings (the 12 dialogs, the 15 submenus, Quit versus Esc, the history result) are its [V], taken here as [R]; the code facts I re-read are tagged [C]. No suite, build, screenshot or perf run (the tree belongs to wave 3).
- No live TallyPrime, Zoho Books or Pipedrive: where each puts Back, and whether TallyPrime asks on Esc for an edited alteration or a typed master, is [unverified].
- Headless Chrome on macOS only (the audit's): not Windows, Linux, Firefox, Safari or Electron; no real touch; no screen reader. The thumb button was simulated by the CDP "back" button.
- Whether a page can cancel `Alt+Left` in a browser tab, the Windows/Electron default for `app-command`, `window.close()` on the Gateway's Quit in a tab, and Safari and Firefox history behaviour: [unverified].
- **Second pass:** the TDS dialogs' behaviour was read in code (the Esc handlers and `onCancel` bodies of all 24 `Box` and 17 `ListPrompt` sites), not clicked; the reviewer's width measurements of the top bar (about 956 px at 1024) were not re-measured; the 17 Back-labelled rail buttons and the 120 and 96 counts are a literal grep (the review counted 16 and 99); `beforeunload` in Electron is [unverified].
- That the delegated click handler of B9 (slice B-4) can map every rendered row to its nav index in every report (the audit says `ReportShell`'s DOM pass already does); the grid's `ListTable` is the safer home and is not built yet.
- Effort numbers are [J].

---

## 16. Sources

**Repo (read, not modified):** `CLAUDE.md`; `docs/research/back-nav-audit.md`; `docs/design/zoho-ux-spec.md` section 5; `docs/design/left-nav-spec.md` (as cited by the audit); `docs/research/{wave-a2-plan,custom-fields-code-audit}.md`;
`src/ui/widgets.tsx` (`TopBar`, `StatusBar`, `Box`, `ListPrompt`, `AcceptPrompt`, `ButtonPanel`, `DatePrompt`, `PeriodPrompt`), `src/ui/AppShell.tsx`, `src/ui/navigate.ts`, `src/ui/navGuard.tsx`, `src/state/{navGuard,dirty,recents,store}.ts`, `src/core/navPlan.ts`, `src/core/uiPrefs.ts` (`RAIL_NEVER_HIDE`, `TOP_ITEM_IDS`),
`src/core/shortcutRegistry.ts`, `src/keys.ts`, `src/ui/overlays.tsx` (`overlayUp`), `src/ui/icons.tsx`, `src/styles.css`, `src/styles/railtop.css`, `src/styles/light/shell.css`, `src/ui/reports.tsx` (`useRowNav`), `src/ui/router.tsx` (`CRUMBS`).
**Web (through the audit; not re-fetched) [W]:** MDN `popstate` event https://developer.mozilla.org/en-US/docs/Web/API/Window/popstate_event; MDN `History.pushState` https://developer.mozilla.org/en-US/docs/Web/API/History/pushState; Chromium history manipulation intervention (URL in 10.1).


---

## Review log (2026-10-21, second pass)

The 10 findings that concern this spec (four of them shared with `custom-fields-spec.md`), each re-checked against the code first; the other 28 of the review's 38 are in that spec's log. **Rejected: none.**

| # | Sev | Finding | Reproduced? | Disposition (where) |
|---|---|---|---|---|
| 12 | major | B5 treats the TDS prompt and the nature picker as neutral closes, but Esc records a decision | **Yes** [C `VoucherScreen.tsx`: `TaxPrompt` Esc is `onDone(null)` ("Esc to skip"); `pickAsk` `onCancel` sets `TDS_SKIP`; the other `onCancel` bodies of 17 `ListPrompt` and the Esc of 24 `Box` read, all neutral or advisory] | Accepted: `scrim` and `cancelLabel` on `ListPrompt`, "Skip TDS", "No TDS on this", the audit result in 5.2, `box-close` and `back-tds` (B5, A9) |
| 13 | major | no dirty probe for the Custom Fields editor, Starter sets or Fill blanks, so Back would not ask | **Yes** [C `mayHoldWork`; `CustomFields.tsx` registers nothing] | Accepted: the editor registers a probe, `navstack.assert` covers it, `back-dirty-cf` (4, 11, B12; custom-fields 7.6) |
| 14 | minor | browser Back read every `none` as "at the root"; no `beforeunload` | **Yes** (the spec's own 2.1 and 10.1 against row 1 of `planBack`; [C grep] no `beforeunload`) | Accepted: `none` carries a `why`, only `root` disarms, the others re-push; a leave guard (2.1, 10.1, Q8) |
| 15 | minor | Back is not disabled while an Accept is in flight | **Yes** [C `VoucherScreen.save`: awaits numbering, then `saveVoucher`, then `st.pop()`] | Accepted: a busy registry blocks Back; the post-save pop and push get a mounted guard (2.2, B12, `back-inflight`) |
| 26 | major (shared) | no mouse path to some controls, Group Summary rows in particular | **Yes** [R audit F10; C `reports.tsx`] | Accepted here for the Group Summary drill (3, B-4); the rest is the sibling's |
| 31 | minor | two Backs on one screen; the tooltip claims Esc equals Back; `keyLabel('Escape')` | **Yes** [C 17 rail Esc buttons in 13 files labelled Back; `keyLabel('Esc')` is the app's spelling] | Accepted, **ADAPTED**: the tooltip drops Esc, the top-bar control always names a destination, and the residual below 1100 px is stated, not hidden (1.1, 6.1, B11) |
| 32 | minor | B6 overrides 24 closures with no per-screen proof | **Yes** [C 120 rail Esc buttons, 96 literal `st.pop()`] | Accepted: a one-row-per-button table, `keepClick` as the explicit opt-out, the closures run against Esc in `back-register` (6, 11) |
| 33 | minor | Back costs width at 1024 and moves the shedding point | **Not re-measured** (`railtop.css` confirms the structure, not the numbers) | Accepted conservatively: chevron only below 1100 px; the recipes and suites that pin the bar are named and re-run (1.2, 11.2) |
| 35 | minor (shared) | `Box.onClose` required in one step collides with new dialogs | **Yes** (the plans' own ordering) | Accepted: B-2a (optional plus a source scan) then B-2b (required) (12.1, 13) |
| 28 | major (shared) | the editor is a "docked panel in the F12 family", which is self-contradictory | **Yes** [C the F12 panels (`reportConfig`, `VoucherConfig`) are `Box` modal shields] | Accepted in the sibling (7.2); this spec's B5 keeps `Box` as the dialog shell and counts the F12 panels among the traps it fixes |

Not taken from the review: nothing. Not verified in the second pass: the reviewer's pixel measurements, how TallyPrime or Zoho place Back, anything in Electron.

## Fix round (2026-10-24, after the build review)

Three confirmed findings against the built slices, each fixed at its root and proven by a suite that fails on the old behaviour and a recipe in `scripts/shots/recipes/cf-fixes.mjs` (real mouse events, never Accept).

- **A show-once secret is unsaved work (section 2.1 and 4).** The API-key dialog is a modal, so Back is held off by the modal itself; Webhooks draws its signing secret **inline**, so the top bar's Back (and a rail click, Go To, a favourite) popped the screen with the secret on it and no question, while Esc and the rail's Esc button had always dismissed the panel first ("losing it costs a new endpoint"). Webhooks now registers `useDirty(() => secret !== null)`: with the secret showing, Back asks "Leave this entry?" (Stay armed; Stay changes nothing, Leave pops); with none showing, Back pops with no question. Esc is unchanged. `navstack.assert` lists the screen among the hand-wired probes and pins the probe's exact shape; `fx-webhooks-secret` drives it with the real mouse (a stub of `/api/webhooks` in the page, no account).
- **A dialog whose body grows must stay inside the window (section 5).** `Box` is `overflow: hidden` and centred with no height limit, so a dialog with many rows lost its title-bar Close and its foot's Cancel and Save off the screen (40 variant rows made a 1,369 px box on a 900 px window): the mouse trap section 5 set out to remove, reintroduced by Opening by variant and Split Not set. The remedy is **opt-in**, `boxClass="box-fit"` (`styles/boxclose.css`): a flex column no taller than the window less a margin on each side, the title bar and the foot fixed, only the body scrolling. It is a class a dialog asks for and not a rule on every `.box`, so a dialog that already fits is pixel-for-pixel what it was and a pick-list popup that hangs out of a body is not clipped by a scroller it never had. The three Stock by Field dialogs (size run, Opening by variant, Split Not set) ask for it; any new dialog whose rows the person adds does the same (`box-close.assert` pins the class, its rules and the three uses). `fx-stock-panels-bounded` (25 rows at 1366x768) and `fx-stock-panels-150` (12 rows at 150 % text) measure the Close, Cancel and Save inside the window and click them with the mouse.
- **Unapplied ticks are typed work (section 5.2).** The "Ledgers under" picker of Balances by Field was classed a picker (`scrim="close"`), so a stray click on the backdrop discarded ticks Apply had not applied. It is now `scrim="ignore"` (Cancel and Apply are the ways out) and its row in the 5.2 table says `typed`; `fx-balances-scrim` presses the backdrop with the real mouse and finds the dialog and the ticks as they were.

Not done here: the other dialogs of the 5.2 table were not re-measured for height (only the three whose rows grow ask for `box-fit`); a dialog added later whose body can grow has to ask for the class, and nothing but review will notice if it does not.
