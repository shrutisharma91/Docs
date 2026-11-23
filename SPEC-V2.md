# Comsky Books v2 — module implementation spec (for parallel agents)

Comsky Books is a keyboard-first TallyPrime replica (React + TS + Vite + Dexie + Electron).
v2 adds: inventory, GST, bill-wise, cost centres, TDS, payroll, orders, multi-company,
banking/BRS, printing, backup, real sync. The core (types, engines, store, form
framework, menus, routing) is DONE. Your job: implement the screens in YOUR files only.

## Hard rules
1. Touch ONLY the files assigned to you. Shared files (types.ts, store.ts, db.ts,
   engines in core/, forms.tsx, widgets.tsx, App.tsx, MenuScreen.tsx, styles.css,
   keys.ts, screenCtx.ts, scaffold.tsx) are read-only for you.
2. `npx tsc --noEmit` must pass for YOUR files before you finish (other agents'
   stub files may still be stubs — ignore errors outside your files, but there
   normally are none).
3. Never run `npm run dev`, git commands that mutate state, or `npm install`.
4. All UI must be fully keyboard-operable in the Tally idiom (see below). Mouse is secondary.
5. Look at `src/ui/masters.tsx` (LedgerForm) and, for reports, `git show b62ba97:src/ui/reports.tsx`
   (the v1 report implementations) as style references.

## The Tally keyboard idiom
- `useScreenKeys(handler, active)` from `ui/screenCtx.ts` — handler gets KeyboardEvent,
  return true to consume. Only fires when your screen is top-of-stack. `combo(e)` from
  `keys.ts` normalizes ("Alt+F2", "Ctrl+A", "Enter", "Y").
- Esc pops (st.pop()), Enter drills down / next field, ↑↓ move selection,
  F2/Alt+F2 change period (use `usePeriodControls` pattern from v1 reports),
  Ctrl+A accepts forms, Alt+D deletes (with AcceptPrompt "Delete ?"), Y/N confirm.
- Every report row navigation: selected row gets class `sel` (yellow bar).

## Building blocks (read these files first)
- `src/ui/forms.tsx` — `useTallyForm({fields, initial, initialIds, validate, onAccept, onCancel, deletable})`.
  Field types: text, amount, qty, number, date, select (right-side popup w/ filter),
  drcr, toggle (Y/N). `fields` can be `(values, ids) => FieldSpec[]` for dynamic visibility.
  Use `MasterShell` for the screen chrome. This is MANDATORY for master screens.
- `src/ui/widgets.tsx` — ButtonPanel (right side buttons), SelectPopup, AcceptPrompt,
  DatePrompt, PeriodPrompt, ErrBand, Box, Hl, useAutoFocus/refocus.
- `src/state/store.ts` — `useStore()`: state arrays (ledgers, stockItems, vouchers, ...),
  `push/pop`, `period`, `curDate`, `saveDoc(table, doc)`, `removeDoc(table, id)` (returns
  error string or null — show it in ErrBand), `saveVoucher`, `deleteVoucher`,
  `nextVchNo(typeId)`, `formatVchNo(typeId, n)`, `ensureLedger(name, groupSlug, extra?)`.
  Screen union type lists every navigable screen.
- `src/core/*` — accounting.ts (closingBalances, periodTotals, subtreeIds, ledgersUnder,
  groupBalance, netProfit, openingDifference, plOpening), inventory.ts (stockPositions,
  stockValueAsOn, stockDelta, stockItemRows, pendingOrders, lineDir), gst.ts
  (isIntraState, computeItemGst, computeLedgerGst, taxAmountFor, voucherGstSummary),
  billwise.ts (outstandingBills, openBillsFor, ageingBucket), tds.ts (computeTds,
  tdsSummary), payroll.ts (attendanceDays, computeEmployeePay, buildPayrollEntries),
  money.ts (fmtAmt Indian format, parseAmt, round2, fmtQty, parseQty, amountInWords),
  dates.ts (fmtDate, fmtDateMed, parseDateInput, monthsBetween, monthStart/End).
- `src/data/defaults.ts` — `gid(cid, slug)` well-known group ids (slugs: capital, loans,
  curliab, fixedassets, investments, curassets, branch, miscexp, suspense, sales,
  purchase, directinc, directexp, indirectinc, indirectexp, bank, cash, deposits,
  loansadv, stock, debtors, bankod, secured, unsecured, duties, provisions, creditors,
  reserves), `vtid(cid, base)` default voucher-type ids, `mainGodownId(cid)`,
  `primaryCostCatId(cid)`.
- Types: `src/types.ts` — Voucher {typeId, base, number, numberStr, date, refNo?,
  partyId?, itemLedgerId?, entries[], inventory?[], payroll?[], orderRef?, narration}.
  VoucherEntry {ledgerId, type Dr|Cr, amount, bills?, costs?, bankDate?, instrument?}.
  `featuresOf(company)` gates modules.

## Report screen pattern (copy from v1)
Title row + `rpt-title`/`rpt-sub`, `table.tt` rows, `useScreenKeys` arrows/Enter/Esc,
F2 period prompt, ButtonPanel with F2:Period / Esc:Quit. Amounts right-aligned via
`td.r` + `fmtAmt`. Drill-down pushes the next screen. Totals row class `total`.
Two-sided statements use `.twoside`. Grand totals must actually reconcile.

## Sign conventions
Ledger balances signed: positive = Dr, negative = Cr. `closingBalances` includes
opening. Income ledgers normally carry Cr (negative) balances. Bill balances:
positive = receivable (Dr), negative = payable. Inventory: qty positive; direction
from voucher base via `lineDir`.

## Printing (Alt+P)
`{ s: 'printVoucher', voucherId }` renders a GST tax-invoice preview (`.print-preview`),
`P`/Ctrl+P triggers `window.print()` — CSS already hides the app shell in @media print
when content is inside `.print-root`. Reports may offer Alt+P similarly (optional
except where your module spec says).
