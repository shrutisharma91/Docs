# Custom fields, reporting by field, stock by field and the Back control: build notes

Branch `cf-back-nav`, worktree `/Users/Akshay/tallyreplica/.wt-cf`, cut from the snapshot `baa0c45` (parity wave 3 as it stood, unreviewed). Written 2026-10-25 for the
orchestrator. **BETA: built, reviewed, fixed and gated in the worktree; rebased onto `main` (`5aee765`, 2026-10-25) and fixed again after the rebase (section 12); NOT merged into `main`,
NOT deployed, never used by a customer.** Sections 0 to 11 were written BEFORE the rebase: where a figure there differs from section 12, section 12 is the one for the rebased tree. The sources are the two specs
(`custom-fields-spec.md` with its owner addendum of 2026-10-22, `back-nav-spec.md`), the builder and fixer reports of each slice, the final gate, and the checks
made while writing this file (section 11 says which). Nothing here edits `CLAUDE.md`, `docs/HANDOFF.md`, `COMSKY_BOOKS_PRODUCT.md` or `docs/parity-data.json`:
sections 8 to 10 are the text to fold in after the merge.

## 0. Status in ten lines

1. **Built:** the Back control and its dialog fixes (B-1, B-2a, B-2b, B-3, the grid-independent part of B-4); custom fields v2 (CF-0 model, CF-1 editor and masters,
   CF-14 entry rules, CF-2 voucher strip, CF-3 item-line fields, CF-7 stage B types and Fill blanks, CF-8 employee and lead places and the lookup type); the five
   "Analysis by field" reports (CF-4a Summary, Bills and Balances by field; CF-5 Income and expense by field; CF-9 Stock by field); banded dimensions (CF-4c); the
   company-aware row filter on four reports (CF-4b-lite); the server half (CF-6a); stock by a variant that a field tracks (CF-9 a to d).
2. **Not built:** CF-4b (the same field as a column and a sort through the report grid, and the grid, B-grid, itself), CF-6b, CF-9e, CF-9f, CF-10, CF-11, CF-12,
   CF-5b, B-5 and B-6 (the browser's Back, thumb button, swipe, `beforeunload`). Section 4.
3. **Gate (2026-10-24 on one fixed tree, then re-run here):** tsc 0; build green; `npm run assert` **137 suites / 65,967 assertions, all green** (re-run on the exact
   tree committed with this file); perf 15/15 gated; **938 screens**: light compact 938/938, dark compact 938/938, **light comfortable 936/938** (two screens failed
   because of a harness viewport leak, not the app: section 1; repaired after the merge, section 12). Zero console errors in all 2,814 captures.
4. **Frozen defaults held:** no Dexie bump, no SQL migration, no dependency, no new chord, `electron/` untouched. A company with no custom field and a person who
   changes no setting see what they saw before, asserted by reference (`customfields-identity` and others, section 3).
5. **The owner's answers of 2026-10-22 are built, not just designed:** an administrator chooses how a mandatory field treats an OLD voucher (company default in
   Settings > Workspace plus a per-field override; default stays "new entries only"); the three examples (Colour, Zone, tags) are served as shapes: Summary by field
   AND Income and expense by field both exist, over every voucher family and every place a field lives, Bills and Balances by a party field, Stock by an item
   attribute AND by a line-level variant field that "tracks stock".
6. **Deploy:** a backend deploy (JS only, no migration) must come BEFORE or WITH the SPA, and the deploy smoke must be run by the owner on the VM. Section 6.
7. **Merge:** done as a rebase onto `main` (`git rebase --onto main wave3-wip cf-back-nav`; the plan said 13 files conflict textually, 27 hunks; the rebase stopped on 15 files and
   took the specs from this branch). Section 7 is the file-by-file list and its "Merge outcome"; the review and fix round after the rebase is section 12.
8. **Parity:** unchanged, 250 / 58 / 73 (73.2%). Five **beyond** rows to add (41 to 46). Section 9.
9. **Open defects worth knowing before release:** a six-line-field invoice re-renders every line per key; the disk was at 98 to 100 percent the whole time. Three items that were open when this file was
   first written are closed (section 12): the harness viewport leak is fixed; `ledger-tail-reachable` was never a defect of the app (the recipe was loosened on this branch against wave 3's unfixed tail, `main`
   has since fixed the tail itself, so the recipe is main's again); `sidenav-flow` was repaired in the branch commit and passes 355 of 355 on the rebased tree.
10. **Checked in headless Chrome on macOS only**; nothing was compared with a live TallyPrime, the GST portal, an older shipped build on the real server, or the VM.

## 1. Gate record

| Gate | Result | Notes |
|---|---|---|
| `npx tsc --noEmit` | 0 errors | re-run 2026-10-25 on the committed tree |
| `npm run build` | green (final gate, 2026-10-24) | JS `index-BKH2UTuR.js` 3,154 kB (1,008 kB gzip), CSS `index-jv3-B248.css` 322 kB; only the usual chunk-size warning. Not re-run for this file |
| `npm run assert` | **137 suites / 65,967 assertions, all green** | re-run 2026-10-25 on the committed tree (49 s). 37 suites are new (15,240 assertions): `customfields` 3,337, `customfields-stock` 1,253, `customfields-report` 1,204, `customfields-pl` 860, `backnav-esc` 731, `box-close` 581, `customfields-bills` 562, `customfields-bands` 447, `customfields-sync` 399, `customfields-editor` 346, `backnav` 258, `customfields-filterlite` 228, `customfields-attribution` 227, `customfields-screens` 226, `customfields-masters` 224, `customfields-draft` 192, `cf-line-column` 188, `customfields-lookup` 187, `cf-fill-blanks` 184, `customfields-stageb` 173, `customfields-reportview` 164, `customfields-voucher` 136, `customfields-usage` 128, `customfields-rules` 119, `customfields-columns` 116, `customfields-dimension` 105, `customfields-stockentry` 106, `voucher-resave` 106, `customfields-import` 83, `customfields-print` 55, `customfields-syncdiff` 49, `customfields-identity` 42, `goto-vocab` 37, `customfields-heldrefresh` 36, `customfields-stockexport` 22, `customfields-stockfuzz` 7, `entrypolicy` 2,122. Pinned suites other lanes touched, all green: `visual` 6,609, `sidenav` 1,807, `reportscenter` 2,007, `zoho-scaffold` 1,031, `navstack` 798, `master-resave` 35, `security-access` 181, `syncconflict` 306, `uiprefs` 315 |
| mutations | every deliberate mutation list killed | 43 new mutation files; CF-1 alone 193 mutations. Not re-run for this file |
| `npm run perf -- --enforce` | **15/15 gated pass**, exit 0 (final gate, 89 s) | 12 reference rows over their line, ungated (Dashboard v2 all-widgets 678 ms at 250k; GST returns 1,218 ms at 250k; Voucher save 201 ms at 250k). The stale `vch.altWalk` mirror of wave 3 is fixed on this branch. CF-9 reference: `stockPositions` with a resolver is 1.17 to 1.67x the plain call at 250k on a busy machine (plan: at most 1.3x); CF-1 editor path: `usageOf` 61 ms at 50k but 690 ms at 250k, `missingCounts` 271 ms at 250k, memoised, editor only |
| `npm run shots`, 938 screens, three configs | light compact **938/938**, dark compact **938/938**, light comfortable **936/938** | 0 console errors in all 2,814 runs. The two failures, `cf-hr-editor-panel` ("timed out waiting for More options") and `cf-masters-from-editor` ("nothing to click: label Sundry Debtors"), are a recipe viewport leak: `cf-voucher.mjs` (lines 395 and 760) and `cf-line-column.mjs` (lines 42 and 309) call `Emulation.clearDeviceMetricsOverride`, which drops the harness's 1440x900 window to the native 1440x813 for every later screen; both screens fail every time at 813 in comfortable density and pass at 900 (re-run: 138/138). **Verified again while writing this file: the leak was NOT fixed at that time (it is, after the merge: section 12).** A single unchunked comfortable sweep fails the same two screens. The fix is one line in each recipe (`clearSize` should call `setSize(p, p.opts.width, p.opts.height)` as `cf-fixes.mjs` `restoreSize` does) plus scrolling the two targets into view. Sweeps ran in chunks of at most 100 screens (938 outlasts the old budgets; `--timeout 7200`), with a reconstructed manifest for one dark-compact chunk |
| tree stability | fingerprint identical before and after the gate (`5c75790b...`) | 348 files changed against the snapshot at the gate, +51,292 / -1,872; that tree is `1d23168` plus the five modified files and one new recipe (`cf-nav-labels.mjs`) committed with this file |
| deploy smoke on a real Postgres | passes (throwaway Postgres 17 on macOS) | `scripts/pg/customfields-pg.mjs`. NOT run on the VM; the VM runs Postgres 16 |

Footprint against the snapshot: about 80 new and 101 modified files under `src/`, 112 under `scripts/assert`, 34 under `scripts/shots`, 7 under `backend/src`, 3 under
`scripts/smoke`, 3 under `scripts/pg`, 6 under `scripts/perf`, 13 under `src/styles`. `docs/design/*-spec.md` carry the full text, review logs and fix-round records.

## 2. What was built, slice by slice

Status words: **done** (built, reviewed, fixed, gated), **partial** (built, a stated part is not), **not built**. Every slice's recipes ran in light and dark, compact and
comfortable, and at 150 percent text; an exception is named.

### B-1 and B-3, the Back control and the rail's Esc button (lane Back): done
- **Built:** `planBack` (pure, `core/navPlan.ts`, a `why` on every refusal); the session trail (`state/trail.ts`; `TRAIL_MAX` and the trail rules live in `core/navPlan.ts`);
  `requestBack` (`ui/navigate.ts`); `backLabel` (`ui/backNav.ts`); the control `src/ui/BackButton.tsx`, mounted in `TopBar` by one import and one element in `widgets.tsx`;
  the busy registry (`registerBusy`, `useBusy` in `state/dirty.ts`). **B-3:** the rail's Esc button runs the Esc key (`runEscape` in `keys.ts`), so a mouse user gets what
  Esc does. Suites `backnav` 258 and `backnav-esc` 731 (the contrast pairs of the Back control are cross-read from `visual.assert` inside `backnav`). Recipes `back.mjs`
  (`back-submenu`, `back-drill`, `back-dirty-*`, `back-rail`, `back-narrow`, `back-register`: 16 generated recipes over existing set-ups), flow `back-inflight-flow.mjs`,
  probe `rail-esc.mjs`. Behaviour changes where the key and the old button differed, now the key's: Stock Query goes back to the item picker; a register goes from a month
  to the month list; a Day Book clears marks first; Reports Center clears a typed search first; Data Check returns to the list with the cursor; Settings closes the open
  recorder first; Create New Report asks instead of discarding silently.
- **Deviations:** the component is `BackButton.tsx` (not `backNav.tsx`: two files with one basename resolve ambiguously); `requestBack` returns a status string
  (`pop`, `reopen`, `ask`, `none-root`, `none-pending`, `none-gate`, `none-busy`) instead of void; judged again at the click (`backAllowedNow`: pending question, busy
  probe, DOM) while the dimmed look follows a 250 ms poll (a permanent 4 Hz timer while the top bar is mounted: a MutationObserver would be quieter); the first-run
  company form counts as a gate like the two sign-in screens; `runEscape` returns a boolean and, when focus is inside the left rail (which swallows its own Esc),
  dispatches on the body; `visual.assert.ts` is not edited (the pairs are proved in `backnav`); `back-register` is generated recipes, not one hand-built walk of 24
  closures (browser-covered: month list, Stock Query, marked Day Book rows, Data Check, Sync Mismatch, Cheque preview and test print, bank-statement mapping, Split
  Company review, Reports Center search, Create New Report and the security screens; source-table only, no harness state reaches them: Webhooks secret (since proved by
  `fx-webhooks-secret`), Snippets editor, form16a certificate, payroll masters and vouchers, Bill of Supply, reminders, Import Data's mapped steps). The rail Esc table is 120
  rows as the builder reported it (90 literal pops by that scan, the spec had counted 96; 33 closure rows; 1 hook row, the shared GST period hook with its 7 importers
  checked; 0 `keepClick`); those class counts add to more than 120 and the report does not say why (presumably overlap): `backnav-esc` holds the table, trust it over
  the prose; the literal-pop ratchet sits at 90. `scripts/shots/shoot.mjs` gained `SHOTS_PROBE` and `SHOTS_SCREEN_TIMEOUT_MS` (additive, default off).
- **Open:** (`scripts/shots/flows/sidenav-flow.mjs` was listed here as failing "only Switch Company under Settings", stale since Phase 27 added Preferences; the branch commit updates its
  expectations (the Preferences row, 27 `+` buttons) and it passes 355 of 355 on the rebased tree, light compact: section 12); the builder counted 108 harness states with a dialog shield over the rail's Quit (the same mouse trap; B-2a's Close on every dialog is the remedy and the states were not re-counted afterwards); headless Chrome on macOS only (real thumb
  button, touch, screen reader, Firefox, Safari, Windows and Electron unchecked).

### B-2a, B-2b and the grid-independent part of B-4: done (B-2b done in a later round; the first B-2a report still says "not flipped")
- **Built:** `Box` takes `onClose` (REQUIRED in the type: `widgets.tsx` says "since B-2b", verified in the source), draws a title-bar close and a foot Close or Cancel, and has
  a `scrim` rule per kind (`close` only for a dialog that applies live and holds no typed work; `ignore` otherwise); `ListPrompt` takes `scrim` and `cancelLabel`; the TDS
  wording ("Skip TDS", "Skip TCS", "No TDS on this": each says what closing records); `src/ui/boxClose.ts` and `styles/boxclose.css`; the hand-built overlays became
  Boxes or got a Close (the Share panel keeps its own `.share-overlay` with a Close and the same backdrop rule); the voucher's busy probe (`useBusy(numberBusyNow)`) and
  post-save mounted guard, so Back refuses during an Accept in flight; the rail's Prev/Next button (click = next, Shift-click = previous); cursor on click and scroll restore
  inside `ReportShell` and `useRowNav` in `reports.tsx`; the Group Summary mouse drill. Suite `box-close` 581 (source scan that fails a `<Box` without `onClose`, plus
  the 5.2 table: **one row per Box or ListPrompt site; any lane that adds one must add its row, the failure prints the row to paste**; the literal counts 29 Box and 17
  ListPrompt are lower bounds). Recipes `back-dialog`, `back-tds`, `back-inflight`. Fix round: the show-once secret on Webhooks is unsaved work (`useDirty(() => secret !== null)`;
  `fx-webhooks-secret`); a dialog whose body grows must stay inside the window (opt-in `boxClass="box-fit"`; the three Stock by Field dialogs ask for it;
  `fx-stock-panels-bounded`, `fx-stock-panels-150`); Balances by Field's "Ledgers under" tick list is typed work, `scrim="ignore"` (`fx-balances-scrim`).
- **Deviations:** `Box` has pass-through props beyond the spec (`boxClass`, `boxStyle`, `bodyClass`, `bodyStyle`, `ariaLabel`) so hand-built overlays could become Boxes
  without a layout change; scroll restore and click-sets-cursor live in `ReportShell` and `useRowNav`, not in a grid `ListTable` (B-grid does not exist), so screens that do
  not use `ReportShell` do not get scroll restore.
- **Open:** the other dialogs of the 5.2 table were not re-measured for height; a dialog added later whose body can grow has to ask for `box-fit` and only review will
  notice if it does not.

### CF-0, the model (lane M): done
- **Built:** `types.ts` additions (v2 definition properties, `Company.entryPolicy`, `EntryEdit`, `req: 'policy'`, `tracksStock`, `OpeningStockAlloc.variants`,
  `InventoryLine.countPart`); `core/customFields.ts` (normalise, places, `needsServerCap`, ids, `coerce`, `checkUdf`, `mergeUdf`, `udfView`, `usageOf`, caps,
  `trackingFieldsOf`); `core/entryPolicy.ts` and `entryPolicyWrite.ts`; `core/fieldDimension.ts`; the legacy v1 reader; the store plumbing (`saveDoc` and `saveVoucher` options,
  `UdfRefusal`, the `checkDelete` case); the backup schema-3 step (silent, written only when v2 data exists); `core/customFieldEdit.ts`, `customFieldColumns.ts`;
  `scripts/assert/srcscan.ts` (a one-pass comment stripper). No screen changes by a pixel.
- **Deviations:** `UDF_KINDS` left at two entries (the six places are `PLACES` / `PLACE_KINDS` in `core/customFields.ts`; CF-1 switched the editor); `checkDelete` refuses a
  hard delete of a v2 definition only (a v1 row stays deletable so the old editor's Delete still works); `UdfRefusal` and `udfRefusalFor` live in `core/customFields.ts` and
  `store.ts` imports and re-exports them (a pure suite can drive them); **`mergeUdf` returns `{ ok, udf, problems }` and never truncates** (callers use `.udf` and must not save
  when `!ok`); `checkUdf` gained `level` (`'form'` or `'store'`) and `groups`; `partyOf(voucher, partyLedgerIds)` takes the set (`partyLedgerIdsOf` builds it); a line
  field's `scope.stockGroups` narrows which lines ask for it when the item group is known; `normaliseDef` returns a frozen def and DROPS a row whose type, key or place this
  build cannot read; `EntryPolicyStored` carries unknown fields back (4 KB, 3 levels) and keeps a valid `since` when the radio moves off it; `store.ensureLedger` passes
  `{ system: true, udf: 'exempt' }`; a global `unhandledrejection` fallback says a `UdfRefusal` once; an item column is labelled "Colour (items, current)".
- **Writers:** `customfields-rules` classifies every `saveVoucher(` / `saveDoc(` caller: **24 of 24 writer groups wired, 0 pending** at the final gate (the ratchet that stood at 20
  pending after CF-0 went to 0 as CF-1, CF-2 and CF-8 landed).
- **Open:** the entry-policy writer, mirror and Settings panel were CF-14 (below); banded dimensions were CF-4c (below).

### CF-6a, server side of custom values (lane S): done (CF-6b not built)
- **Built (backend, JS only, no migration):** `caps` parsed with bounds; a separate per-document `current` read (`backend/src/histories.js`, a lateral `LIMIT 1` per (table,
  document) pair, no device filter, sliced at 500) used only for writers that do not state `udf2`; `UDF_CARRIERS` (own-property lookup); `carryUdf` with the unambiguous-line
  rule (identity item / godown / batch; a repeated identity is paired only when the rest of the line is unchanged; losses are reported as `udfLost`); a v2 definition delete from a
  writer without `udf2` is stored as a hidden put; REST keep (`rest.js`, `restWrite.js`); `udfKept` in the push answer; the capability-keyed company-row rules **`CAP_KEPT`
  (`entryPolicy`) and `OMIT_KEPT` (`ui`)** (built in the CF-2 pass because `cf-old-client` is in its gate); `SERVER_CAPS`. New files `histories.js`, `pushRun.js`, `mismatch.js`,
  `restWrite.js` (the push body was split out of `sync.js`). Client: `src/sync/caps.ts` (`UDF2_BUILD`, `clientCaps()`, the stored server caps), `core/syncDiff.ts`. The Sync
  Mismatch Report list says "Custom" for a custom-values row. Suites `customfields-sync` 399 (including the in-process deploy scenario), `customfields-syncdiff` 49,
  `syncconflict` extended. Scripts: `scripts/smoke/customfields-deploy-smoke.mjs` (HTTP, run by the owner), `udf-scenario.mjs`, `shipped-caps.mjs` (reads what the shipped build states),
  `scripts/pg/customfields-pg.mjs` (SQL half against a real Postgres, needs `psql` and a server it may create a database on).
- **Fix round:** `UDF2_BUILD` is **true** now (the funnel landed): while false the cloud treated this build as an older app and put every omitted value back, so a Zone
  could never be cleared and an administrator's entry-policy change would be reverted. The tripwire reads the funnel where it really is (`core/voucherFields.ts`, `ui/udf.ts`, and
  the screens that call them) and fails when flag and funnel disagree in either direction; section 11b pushes what `clientCaps()` states through the real server rule.
- **Deviations:** the SQL is a lateral per pair, not the spec's `DISTINCT ON` over `tbl = ANY AND doc_id = ANY` (which matches the cross-product of two arrays); line rule (i) is
  stricter than the spec (never mislabels after "delete line 1 and retype it at the bottom", but can lose a label the older app also normalised differently); the push body and
  report helpers moved out of `sync.js`; `pushAnswer` exposes `udfKept` but not `udfLost` (logged and written into the mismatch row).
- **Open:** **CF-6b (`backend/src/udfRules.js`, a shared coerce vector file) is NOT built**, so REST cannot warn about a blank variant; the **deploy smoke has not been run on the VM**;
  the cost of the current read is an estimate (planner uses `idx_change_doc`; under 200 ms for 300 documents among 30,000 rows on a throwaway Postgres 17, not measured on the VM);
  until a build that states `udf2` is installed, the CURRENT shipped v1 master form clears a custom value by omission (`ui/udf.ts collectUdf` skips blanks) and, once this
  backend is deployed, the server puts such a cleared value back (the device is handed the corrected document and a protected-fields row is recorded): see the deploy order.

### CF-1 and CF-14, the editor, the masters and the entry rules (lane E): done
- **Built:** the editor rewrite (`ui/admin/CustomFields.tsx` and `ui/admin/cf/*`: object tabs, options editor, scope, required with the per-field choice of AD.1.5, the Tracks stock
  block, minimum path and More options, live preview through `ui/udf.ts specForDef`, hide, tombstone delete, v1 conversion, starter sets, the "Custom Fields" menu row and its `+`,
  the dirty probe); the master forms (`masters.tsx`, `inventory/masters.tsx`, tail fields, scope, `mergeUdf`); the `forms.tsx` flags (`blankOk`, the three-state toggle, `percent`,
  `amount`, `requiredMsg`, fresh-value fix, cursor-to-field, a clickable ledger Accept); **CF-14**: `state/entryPolicyWriter.ts`, `entryPolicyMirror.ts`, the Settings > Workspace
  panel `ui/settings/panels/EntryRulesPanel.tsx` (the `owner` literal gained `'C'`), a consequence count ("N vouchers would need completing when edited"), the carry line in
  `core/companyAlter.ts`, the panel's dirty probe. Suites `customfields-editor` 346, `customfields-masters` 224, `entrypolicy` 2,122, `customfields-draft` 192,
  `customfields-identity` 42; recipes `cf-editor-*` (16 screens), `cf-masters*`, `cf-policy`, the write-path flow `cf-policy-flow.mjs`. Four older suites had pinned text that
  moved on purpose and were updated: `einvoice-wiring`, `party-terms` (custom tail follows the party terms), `syncguard`, `uiprefs` (four Workspace panels, owner C).
- **Fix round (model-entry, 2026-10-24):** an editor panel opened earlier never writes over what changed under it (a THREE-WAY merge; `staleSaveProblem` refuses only when the status
  moved or one member changed to two answers); option order is saved; a rename keeps the old name (`FieldOption.was`, at most 8, so a value an older app wrote as the old label
  still reads as the renamed option); "Update older fields" never moves a stored value; Esc and a click outside close the strip's open choice list; a pick list stays where its
  rows can be seen under reduced motion; the refusal's one-click fix is clickable at 150 percent; Fill blanks asks the write right per (base, date); Go To reaches the entry rules.
- **Deviations:** no Go To late command for the entry rules in the CF-1 pass (added in the fix round: "Custom-Field Entry Rules"); "Open Summary" is lane R's toolbar button on the
  field list, not a button in the edit panel's Actions row; the preview draws the master-form row, not the strip or item-grid components (they did not exist then); the voucher halves
  of `cf-policy` and `cf-mouse-only` waited for CF-2.
- **Open:** on a master form a select's open popup is a wide panel anchored at the right that covers the
  Yes/No/blank buttons and the Clear of the row it belongs to (a mouse user clicks the field first; the keyboard is unaffected); Fill blanks covers voucher-header fields only.

### CF-2, the voucher header strip (lane V): done
- **Built:** the strip under Narration, the carry funnel and `voucher-resave` (`core/voucherFields.ts`), dirty and signature, `resetLines`, the duplicate (Alt+2) and recall rules,
  the cancel shell, mandatory enforcement per company policy with the W (whether an old voucher is bound) per voucher, the refusal wording, the print block (the form row only when a
  field prints), Bulk Entry (`bulkPlan.ts`, the pure half) and the mapped import. Suites `customfields-voucher` 136, `customfields-print` 55, `customfields-import` 83,
  `voucher-resave` 106; recipes `cf-voucher-walk`, `cf-inherit`, `cf-print`, `cf-old-client`, `cf-voucher-geometry`, flow `cf-voucher-flow.mjs`.
- **Deviations:** the server company-row rework (`CAP_KEPT` and `OMIT_KEPT` in `conflict.js`) was built by this lane (CF-6a/CF-14 scope); the Postgres smoke was not run in this lane
  (the in-process scenario runs inside `customfields-sync`); F10 (Change Voucher Type) is disabled on an alteration in this app, so the spec's "F10 into a base where a field newly
  applies" clause is unreachable from the screen (`checkUdf`'s `beforeKind` still handles it for non-UI callers); the strip's minimum column width is 150 px scaled by the text-size
  token, not 200 px (12 fields at 150 percent on a 1024 px window collapse to one column, 902 px tall: measured by `cf-voucher-geometry`, never wider than the no-field baseline,
  nothing cut off); an accounting-grid voucher (Payment, Receipt, Journal, Contra) has no party, so "inherit from party" applies to the invoice shapes only; the legacy fixed-format CSV
  importers (Ledgers, Stock, Vouchers CSV) offer no custom column (the mapped Excel / CSV importer does; the legacy voucher CSV only counts vouchers that lack a required field).
- **Open:** per the spec CF-2 is not released to a customer before CF-4a and CF-4b-lite (both built); Esc on an open header choice list is fixed in the fix round.

### CF-3, custom fields on item lines (lane V): done
- **Built:** pure `core/lineFields.ts` (which fields apply to a voucher type and an item's stock group; `inlineCount` 0 to 2 columns by measured width, the rest on a sub-row; seeding
  from the item or default; the Tab contract), `ui/vch/CustomLineCol.tsx`, `useLineInline.ts`, the `cf:` focus, the walk, the snapshot, the stock-journal grids and the print sub-line.
  Suite `cf-line-column` 188; recipes `cf-line-geometry` and others, flow `cf-line-flow.mjs`.
- **Deviations:** the Tab and Shift+Tab walk lives in a `lineKey` closure called from `case 'item'` behind `if (lineFieldCount > 0)` (same behaviour as the spec's in-case branch;
  `customfields-identity` still asserts there is no `'Tab'` in the case); Tab on Amount goes straight to the first line field even when Description is shown, and Shift+Tab from the
  first field returns to Description when shown else to Amount; an inline column has room for only the tag that explains a refusal; typing a choice's whole label resolves to that
  option (a small superset of the header strip); `ItemRow` gains two screen-state members `lt` and `was`, never saved; `partyValue` now calls a new `offeredValue` (header behaviour
  unchanged, suites green). No new `visual.assert` contrast pair (the cells reuse pairs it checks).
- **Open:** a focused date cell shows `dd-Mon-yy` text to edit (consistent with the strip, not ideal); Bulk Entry and CSV line columns and the server-side line carry for older writers
  belong to other slices and the older-app flow was not re-run against line cells made by this UI; the six-line-field invoice re-renders every line per key (about 1.7x the latency of
  the same invoice with none: 7.7 ms p50 against 4.4); memoising the item rows is optional and was judged a risky edit of the voucher grid, so it is left.

### CF-4a, Summary, Bills and Balances by field and the shared shell (lane R): done
- **Built:** `core/fieldSummary.ts` over every voucher family, `core/fieldBills.ts` (Bills by field with Pending / Unapplied / Not bill-wise and the group-balance tie; Balances by field
  over the union of the chosen groups), `core/fieldAttribution.ts`, `core/fieldReportView.ts` (choices, measures, row order, register tie, group list, link parameters, all pure),
  `ui/reports/fieldShell.tsx`, the screens `fieldSummary`, `fieldBills`, `fieldBalances`, and the **new-screen checklist registered once for all five analysis screens** (the `Screen`
  union with plain-string parameters, router and crumb, `SCREEN_AREA` = `reports`, recents titles, favourites, five Reports Center rows in a new LAST category "Analysis by field",
  Go To late commands, `DRILL_FEATURE fieldStock: 'inventory'`; **no left-rail row and no Gateway row** on purpose, AD.8). The Reports Center is now 106 reports in 15 categories
  (it was 101 in 14). Suites `customfields-report` 1,204, `customfields-bills` 562, `customfields-attribution` 227, `customfields-reportview` 164, `customfields-screens` 226; the
  suites tie to the paisa at 2k and 50k and on the sample company. Recipes `cf-summary-*`, `cf-bills-by-field`.
- **Deviations:** screen parameters are plain strings (`fieldSummary { fid, src, set, measure }`, not `{ fid, family, bases, measure }`) because `screenKey` stringifies a value; drill
  opens an in-screen list of the vouchers behind the row built by the engine's own `bucketVouchers` walk (not the register's flat level with a live `cf.` filter, which needs CF-4b-lite
  integration); the Bills receivable / payable switch is a chip plus the S key, the Summary's measure a chip plus M (not F12 rows); the keys F, V, M, S, G are screen-local letters, not
  registry chords (documented on the `?` sheet); the unavailable-fields line sits under the table; `scripts/assert/mutations/run.mjs` now clones the tree copy-on-write after the shared
  disk filled; `scripts/shots/golden/goto.capture.mjs` treats the five Analysis rows as late.
- **Open:** saved views (Alt+V) on these reports keep only F12 settings (they are not in `core/savedViews`' registry, which the grid wave owns); print preview of these reports was not
  exercised (CSV was); 250k-voucher perf was not run (2k and 50k only; the `cf.*` perf rows are reference rows).

### CF-5, Income and expense by field (lane R): done, with one live gap
- **Built:** `core/fieldPl.ts` over five sources (own, party, item, line, posting ledger), `core/fieldPlStock.ts`, the screen `fieldPl`, Layout A, recipe `cf-pl`, suite
  `customfields-pl` 860 (the tie to `plFigures` for every source; the remainder guard). The change in stock for an item attribute is attributed exactly (R10) through a local per-item
  reader `itemStockDeltas`, injectable as `FieldPlArgs.stockByItem`; the suite asserts it adds back to `stockDelta`.
- **Deviations:** `stockDeltaByItem` of CF-9a is not used (the local reader does the job and CF-9a's function is a one-line swap if wanted); the Unallocated row uses `RowKind 'other'` with
  key `unallocated` (no new `RowKind`); a remainder guard beyond the spec (a residual larger than pieces plus 2 paise is not absorbed and shows as the not-attributed line); **Layout B
  (options as columns, CF-5b) is not built** (optional in the spec).
- **Open:** the split by a TRACKING line field is engine-complete and proved with hand-worked stub readers, but the screen passes no reader, so a tracking field's stock stays on its own line
  with the "not split yet" note: wire CF-9's per-variant opening and closing value by option id into `src/ui/reports/fieldPl.tsx`. The variants basis has no UI recipe for the same reason.

### CF-4b-lite, the company-aware row filter (lane R): done
- **Built:** a company-aware filter field list for the Day Book, the voucher registers, Ledger Vouchers and Bills (`cf.` the voucher's own field, `cfp.` the party's value now labelled
  "(current)", `cfl.` the line projection labelled "(all lines)"; Bills offers `cfp.` only), chip labels that never print a raw id (`core/filterLabels.ts labelResolverFor`), saved views that
  read "Zone is North", and a re-filter when the field set changes. Files `core/customFilterFields.ts`, `ui/cf/useCustomFilterFields.ts`. Suite `customfields-filterlite` 228; recipe
  `cf-report-filter`. With no definitions `filterFieldsFor` returns the same array references (asserted).
- **Deviations:** the spec says lite does not touch `reportFilter.ts`, `savedViews.ts`, `rowFilter.ts` or `FilterPanel.tsx`; the slice contract said otherwise and they each gained an
  optional parameter whose absence changes nothing (the spec table was corrected; `staleMetas` was not built, the resolver gives the same words); `useRowFilter` now re-filters when the
  offered set changes (fixes a stale-rows bug when a field is deleted under a waiting report).
- **Open:** a field switched out of reports is not offered, and a saved filter that names it is kept and matches nothing (chip "not in this report"; safe direction, but the spec's "keeps
  working" needs a grid-wave change); "any line is Red" needs the any-of operators and the item-wise register; shared (company) saved views holding a custom filter were covered by the pure
  suite but not driven in the harness (they write the company row); the Memorandum, Optional and Post-dated registers are not among the four.

### CF-4c, banded dimensions (lane R + M): done (built in the reports fix round; AD.9 is the record)
- **Built:** text, period and band kinds (`core/fieldBands.ts`) on all five analysis screens; a typed text is one bucket per cleaned value (at most 200, the rest in "Others (n values)"
  with amounts kept); a date is a month, a quarter or a financial year; a number is banded by typed edges (at most 12; round bands suggested from the data and never stored until typed).
  Suite `customfields-bands` 447 (a differential of all five engines against a choice field written from the same values, at 2,000 and 50,000 vouchers); recipes `cf-bands-*`.
- **Deviations (AD.9):** the period and the band edges are chosen with a chip and the key B on the screen, not in F12 (F12 takes cycle rows only); the choice is remembered per field per device
  (`TW_FIELD_DIM`).
- **Open:** column and sort for these types in the listing reports wait for the grid (CF-4b); the Stock Summary's own group mode still offers choice and Yes/No attributes only; a lookup cannot
  group yet; a multi-choice field is refused by decision (a record in two groups would add to more than the whole).

### CF-7, stage B types and Fill blanks (lane E + M): done
- **Built:** long text, multi-choice, email, phone and link on the master forms, the strip, the editor and its preview, reports, filters and importers; Fill blanks with its own batch undo
  (chunks of 500, each atomic; per-option counts; a synced-company gate; `ui/admin/cf/applyFill.ts`, `core/customFieldFill.ts`, `customFieldFillRun.ts`), usage and field statistics, preset
  polish. Suites `cf-fill-blanks` 184, `customfields-stageb` 173, `customfields-usage` 128.
- **Deviations:** the fill batch keeps a 53-bit fingerprint of each document it wrote rather than the document (a few MB instead of the whole vouchers; a collision of about 1 in 10^16 per
  document would fail to notice a voucher edited since and so undo it); the result bar is remembered for the session only; the importer's multi-choice rule is stricter than `coerce` (a hidden or
  removed option in a sheet cell is a row problem).
- **Open:** Fill blanks covers voucher-header fields only (a lookup field waits for CF-8; no fill for lines or masters); the printed "Details" pair draws a long text as stored; the transaction cost
  of a 500-voucher chunk on Dexie is a browser cost the perf harness cannot time; the stage B cell on an item line is CF-3's (built).

### CF-8, employee and lead places and the lookup type (lane E): done
- **Built:** custom fields on the employee form (`payroll/masters.tsx`) and the lead form (`crm/leads.tsx`) through `ui/udf.ts` (optional fields a tail, mandatory fields walk fields after the last
  built-in one: bank IFSC on the employee, Notes on the lead); the **lookup** type (a link to a ledger, item, employee or lead); `checkDelete` on five master tables with the "(deleted master)"
  reading; a Data Check line; masters recipes on both forms. Suites `customfields-lookup` 187, `customfields-masters` 224.
- **Deviations:** a lookup is asked on the master forms only (the editor refuses the voucher and item-line places for a lookup; a lookup an API client writes onto a voucher is carried untouched and
  never printed); grouping by a lookup stays refused ("a later release"); "filterable on their own lists (CF-8)" is NOT built for the Employees and Leads lists (neither has a row filter or Columns
  panel; that is CF-12 / CF-4b); Data Check reads lookup values held by ledgers, items, employees and leads, not values a REST client wrote onto vouchers (but `usageOf`, and so delete protection, does count them).
- **Open:** the Alt+D Delete buttons on the employee and lead forms still have no click handler (already so; the Accept buttons got one); a lookup list renders every matching master with no row cap;
  a synced company where the server lacks `udf2` (the editor refuses a lookup field then, by `needsServerCap`) and an older app resaving an employee or lead on the real server are covered only by mock-server suites.

### CF-9 (a to d), stock by field (lane R + M + I): done, with a perf figure over the plan
- **Built:** the variant engine (`core/variantCore.ts`, `core/stockByField.ts`, `core/variantEntry.ts`, `core/stockAgeing.ts`: the ageing lot walk moved unchanged out of `ui/inventory/ageing.tsx`);
  `byVariant` in `stockPositions`; the **count group** (`InventoryLine.countPart`, merged once in `stockMovements`, `countDeltas`) so no item-level figure reads a resolver; `openingVariantMap` and
  `OpeningStockAlloc.variants` with both trust tests; `pendingOrders` per-line allocation with its fourth argument at every call site; `stockFlow` variant cells (quantities, and Opening and Closing
  value only); the screen `ui/reports/fieldStock.tsx` (both readings, the combination view, the size-run grid, the godown level) and Stock Summary's `'field'` group mode (`ssGroupByOptions`); the
  variant on the voucher (per-item cell, walk, "in stock N" in the picker, per-(item, variant) negative-stock warning, Stock Journal destination default, Physical Stock rows pre-filled per held
  variant, Copy locked on, Hide and Delete refused while a field tracks stock); migration tools (Opening by variant, "Split Not set" Stock Journal draft, Split Company carries variants, Data Check rows);
  Tally XML export merges a count by variant. Suites `customfields-stock` 1,253, `customfields-stockentry` 106, `customfields-stockexport` 22, `customfields-stockfuzz` 7; recipes `cf-stock-by-colour`,
  `cf-variant-entry`, flow `cf-stock-flow.mjs`. Fix round: the Physical Stock rows are a derived default, not typed work.
- **Deviations:** `FlowOpts.bases` and `FlowOpts.cellOf` were NOT added to `stockFlow` (CF-4a's `stockMovementsByBucket` reads the same observed stream and ties to the Stock Summary); names differ from
  the text (`ssGroupByOptions(defs)` replaces `ssGroupByEnum` + `ssSpecFor`; `stockItemRows` `variant` also takes a segment selection `seg`); AD.3.4b's "N lines of tracked items will be booked as Not set"
  preview is built for the Tally XML importer, Bulk Entry and the REST docs but NOT for CSV Import Data or the mapped importer (they make two-ledger vouchers and never touch `inventory`); the Tally
  export merge and a counts-only Data Check row are additions; a printed Physical Stock voucher counted by variant prints one row per part (the variant shows only if the field's Print switch is on).
- **Open:** `stockPositions` with a resolver measured 1.17 to 1.67x the plain call at 250k on a busy machine (plan: at most 1.3x; steady about 220 ms against 150 to 190 ms): re-measure on an idle
  machine before quoting; per-variant costing (CF-9e) and variant-aware manufacture (CF-9f) are not built (the Manufacturing Journal books Not set and warns); an older build reads a count made by
  variant last-line-wins until updated, so **CF-9c must not reach a customer before the build carrying CF-9a runs everywhere that company's stock is read**; the voucher's "in stock" hint reads the
  positions memo including the voucher being altered; the Tally merge follows the app's own reading, never a real TallyPrime import.

### Review and fix rounds (2026-10-24)
Four fix clusters after the build review (5, 5, 20 and 9 findings; two of the nine were duplicate majors). Each was fixed at its root and proved by a suite that fails on the old behaviour; the
full records are the "Fix round" sections at the end of both specs and AD.9. In short: (1) stock entry (the Physical Stock rows as a derived default) and the report fixes, **one scope gap left: the
grid column and sort for text, date and number fields (CF-4b)**; (2) **`UDF2_BUILD` flipped to true** with a funnel-reading tripwire, and the deploy smoke now passes on a real Postgres 17; (3) 18 of 20
findings (4 major, 14 minor) fixed, **two minors open: memoising the item rows, and re-measuring perf on an idle machine**; (4) Go To: the Custom Fields menu row and its `+` keep their discovery words,
which are no longer lent to Go To (`MENU_ONLY` in `ui/paletteVocab.ts`; suite `goto-vocab`), one line in `ui/modern/CommandPalette.tsx` (the late command "Custom-Field Entry Rules").

## 3. Spec deviations that matter across slices (one list)

1. **Backend shape:** `sync.js` lost its push body to `pushRun.js`, `mismatch.js`, `histories.js`, `restWrite.js` (the spec allowed `histories.js` only). This is what makes the merge with wave 3's `sync.js` a hand job.
2. **CF-6a's company-row rules were built by lane V** (CF-2), not S or E; they are the reworked version (capability- and omission-keyed), not the first text (`entryPolicy` inside `UNVERSIONED_PROTECTED`).
3. **Screen parameters are plain strings** for the five analysis screens (no arrays or objects: `screenKey` stringifies).
4. **No chord was added.** The F, V, M, S, G, B keys on the analysis screens are screen-local letters. `shortcutRegistry.ts` shows a 2+/2- edit; check it is prose or live flags only when merging.
5. **The new reports have no left-rail or Gateway row** (AD.8); the menu gained one row, "Custom Fields" (65 to 66 rows, 27 `+` creates), and `sidenav.assert` carries it.
6. **CF-4b-lite edited six shared files** with optional parameters (`reportFilter.ts`, `savedViews.ts`, `rowFilter.ts`, `FilterPanel.tsx`, `reportActions.ts`, `useReportViews.tsx`); the grid wave must preserve them (spec section 16's list).
7. **B-2b was done** (the type is required) although the B-2a report says it was left for later.
8. **Identity is asserted by reference**, as in Phase 27: with no definitions the filter lists are the same array objects, `ssGroupByOptions` returns the same `SS_GROUPBY`, a company that never opens the entry-rules panel stores no `entryPolicy`, and the voucher with tracking off is byte-identical to today's (`customfields-identity`, `customfields-filterlite`, `customfields-stock`).
9. **Spec text left behind:** `FlowOpts.bases` / `cellOf`, `staleMetas`, `useBackState` as sketched, `stockDeltaByItem` as CF-5's reader: names in the spec that the code does not have (the spec's own fix-round sections say which).

## 4. Not built, and what is partial

**Not built (all optional or gated in the spec):**
- **CF-4b**: the same field as a column, header filter and **sort** in the listing reports through the report grid, saved-view stale rules; it **needs the grid (B-grid S1 and S2), which does not exist**. Until then the field works as a filter on four reports (CF-4b-lite) and as a dimension on five analysis screens, but not as a column or a sort. The grid column and sort for text, date and number are the same gap. **This is the only slice standing between the build and the spec's G2 customer release.**
- **CF-6b** (`backend/src/udfRules.js`, the server validating against the company's definitions and entry policy): REST cannot warn about a blank variant or a mandatory field.
- **CF-9e** (per-variant costing, gated on owner question 18), **CF-9f** (variant-aware manufacture), **CF-10** (Tally UDF export and import, gated on three real exports), **CF-11** (field-level `access`), **CF-12** (employee and lead fields in the payroll and CRM reports; the Employees and Leads lists have no filter), **CF-5b** (Layout B).
- **B-5** (an `Esc Quit` button on the Gateway submenus, only if the owner says so) and **B-6** (the browser's Back, the thumb button, a swipe, `beforeunload`; owner questions Q3 and Q8): not built. The in-app Back and the dialog Close are what shipped.
- B-4's grid half (cursor on click and scroll restore inside a grid `ListTable`): only the `ReportShell` / `useRowNav` part exists.

**Partial:**
- **Profit by a tracking line field** (Colour that tracks stock) is engine-complete, not live on the screen (CF-5, above).
- **Lookup** works on masters (party, item, employee, lead), is a column, filter and sort source, but not on the voucher strip or an item line and not a grouping.
- **Saved views on the analysis screens** keep F12 settings only; the chosen field and vouchers are not recalled.
- **Import:** CSV Import Data and the legacy CSV importers have no custom column and no "booked as Not set" preview; the mapped Excel / CSV importer has custom columns.
- **Stock by field** costs by the item's pooled value (each variant's value is the item's value shared by quantity); only quantities, Opening and Closing value are shown per variant.
- **The six-line-field invoice** costs about 1.7x the keystroke latency (memoising the item rows was left).

## 5. Release gates (spec section 17) against what is built

- **G1, the owner beta (CF-0, CF-14, CF-1, CF-6a deployed and smoked, CF-2, CF-4a, CF-4b-lite): met in code.** CF-6a's deploy and smoke are the owner's step.
- **G2, the customer release (G1 plus CF-3, CF-4b, CF-5, and CF-9 for a company that keeps stock by a variant): not met, CF-4b is missing.** CF-3, CF-5 and CF-9 are built. The spec's rule stands: voucher and line fields are not released to a synced company before CF-6a is deployed and the smoke has passed; the editor refuses a field that needs `udf2` while the stored server caps lack it (`needsServerCap`), which is the safety net, not the plan.
- The "visible stub" rows CF-4a reported (Income and expense by Field, Stock by Field) are gone: `fieldPl` and `fieldStock` are real screens (checked: no "Not available yet" in `src/ui/reports/field*.tsx`).
- **CF-9c must not reach a customer before CF-9a runs everywhere that company's stock is read** (an older build reads a count by variant last-line-wins).

## 6. Deploy order

State of the VM on 2026-10-23 (main's HANDOFF 2f): web app and backend are the `687c062` tree (Phase 29 and wave B's backend rule live, migration 008 applied). This branch changes the backend in JS only
(`conflict.js`, `sync.js`, `rest.js`, new `histories.js`, `pushRun.js`, `mismatch.js`, `restWrite.js`; **no new SQL migration, `backend/sql` is untouched**) and the SPA. Deploy secrets are the owner's.

1. **Merge and re-gate first** (section 7): tsc, build, `npm run assert`, `npm run perf -- --enforce`, the 938-screen sweeps in all three configs (chunked), on main's tree after the rebase.
2. **Backend first, in the same window as the SPA** (the owner supplies the VM SSH and database passwords): the recipe is in `docs/HANDOFF.md` section 2 (a `pg_dump` first, rsync of `backend/`, restart of the `comhard-backend` unit). No migration to run (harmless if you do). The new server announces `udf2` in its pull answers (`SERVER_CAPS`).
3. **Run the deploy smoke once, before releasing any custom field to a synced company:** `node scripts/smoke/customfields-deploy-smoke.mjs --url http://books.comhard.in:1036 --yes`. It registers its own throwaway account and tenant (a 7-day trial), walks the scenario over the real HTTP routes (a current and an older app push the same voucher; a definition deleted by an older editor; a REST re-post; two company-row pushes, one stating `udf2`, one not), prints PASS or FAIL per step, exits 0 only if all pass, and prints the SQL that removes its tenant (run as the owner role, tenant documents first: `change_log` has no foreign key to tenant). It refuses to run if `src/sync/caps.ts` says the build states nothing. Optionally run `scripts/pg/customfields-pg.mjs` against a scratch Postgres of the same major version (16), never the production database.
4. **Deploy the SPA right after** (the same HANDOFF recipe; keep a rollback copy as before). **Do not leave a gap:** between the backend deploy and an SPA that states `udf2`, the server treats today's shipped build as an older app, and today's v1 master form clears a custom value by omission, so the server would put a cleared value back on ledgers and items that carry v1 custom values (the corrected document is handed back and a protected-fields row is recorded; nothing is lost, but a person cannot clear a value until the SPA updates). Likewise an un-reloaded web tab or an Electron install that was not rebuilt behaves as an older app afterwards.
5. **Rebuild the Electron installers** (also still owed from wave 3's PDF bridge, which was never run against a real Electron). `electron/` is unchanged by this branch.
6. **Release to customers only after the G2 items** (CF-4b), or release as an owner beta (G1) and say so. Run the smoke's cleanup SQL when done.
7. **Rollback:** the SPA rollback copy; a backend rollback copy; there is no migration to undo. A rolled-back backend ignores `caps` (harmless); `Company.entryPolicy` written by new clients stays in the company document as an unknown field, which an older client's resave can erase (the client mirror and Restore are the defence, as for `ui`).

## 7. Merge plan

**Command (the orchestrator's, once parity wave 3 is committed; it is: `2cee555` and later on `main`, tip `d941298`):** `git rebase --onto main wave3-wip cf-back-nav` (`wave3-wip` is `baa0c45`).

**Before rebasing:**
1. **Commit the 7 untracked design documents in `main` first** (`docs/design/back-nav-spec.md`, `docs/design/custom-fields-spec.md`, `docs/research/{back-nav-audit,custom-fields-code-audit,tally-udf-and-competitors,zoho-custom-fields,zoho-reporting-tags}.md`). The snapshot tracks all seven; `main` has them untracked. They are byte-identical to the snapshot's copies (checked), so as an ordinary commit on `main` the branch's later edits to the two specs replay as clean modifications. If they are not committed, the rebase reports a modify/delete conflict on each spec (resolve by taking the branch's version) and a later fast-forward into `main`'s working tree is refused because the untracked copies would be overwritten.
2. **Squash the branch's commits above `baa0c45` into one** (the seven WIP checkpoint commits, the build and fix-round commits and the commit that carries this file) before rebasing, so each conflict below is resolved once, not once per replayed commit. The numbers below were measured as one three-way merge (base `baa0c45`, ours `main`, theirs this tree) with `git merge-file` per overlapping file; a commit-by-commit rebase may show intermediate conflicts that this measure does not.

**The files where this branch's diff overlaps wave 3's changes made after the snapshot** (`baa0c45` to `main`: 299 paths, of which 28 are also touched here; `mcp/`, `mobile/` and `backend/src/mcpRoute.js` belong to other workflows and do not overlap):

| File | main since snapshot | this branch | Conflict hunks | Resolution |
|---|---|---|---|---|
| `backend/src/conflict.js` | +32 / -1 | +411 / -17 | **2** | Both sides insert at the same place in `classifyPush`. Wave 3's rule 1b (a cancelled voucher stays cancelled: `cancelKept`, `protectedKept.push('cancelled')`) goes FIRST; this branch's custom-value rule and the two company-policy rules follow (renumber to 1c and 1d in the comments). Keep wave 3's `if (cancelKept) {...} else if (!conflict && (kept.length \|\| udfLost.length)) {...}` chain. A cancelled shell needs no custom-value carry: `carryUdf` must not run when `cancelKept`. Prove with `syncconflict` (wave 3's cancel cases) and `customfields-sync` |
| `backend/src/sync.js` | +2 / -2 | +18 / -219 | **2** | Take THIS branch's side (the push body and `loadHistories` left the file). Wave 3's two edits are the `needsCancelCheck` import and the `\|\| needsCancelCheck(c)` in the protect test: both live on in `histories.js` (`planReads(changes, cursor, caps, cancelCheck = needsCancelCheck)`, read off the `conflict.js` namespace so a textual merge cannot drop the OR). **After the merge replace the namespace read with a plain named import** (the note at the top of `histories.js` says so) and re-run `customfields-sync` section 8, which loads with wave 3's predicate on both sides |
| `src/core/importMap.ts` | +332 / -97 | +67 / -9 | **7** | Wave 3's fix round rewrote the mapped import (hardening); re-apply this branch's custom-column mapping by hand onto main's version; `customfields-import` and `importmap` suites decide |
| `src/ui/admin/MappedImport.tsx` | +31 / -15 | +33 / -11 | **3** | as above: take main's, re-apply the custom-column edits |
| `src/ui/admin/SyncConflicts.tsx` | +6 / -3 | +21 / -9 | **3** | wave 3 added the cancel-conflict kind to the list; this branch added the "Custom" badge and humanised custom values; keep both |
| `src/ui/VoucherScreen.tsx` | +4 / -5 | +785 / -108 | **2** | by hand; wave 3's change is small. Afterwards re-check the perf keystroke anchors (`scripts/perf/mirrors.mjs`) |
| `src/ui/forms.tsx` | +9 / -3 | +186 / -39 | **2** | by hand; keep both (wave 3: party-terms tail fields; this branch: the CF-1 flags) |
| `src/core/syncGuard.ts` | +59 / -2 | +29 / -3 | **1** | wave 3 added `cancelConflict` and `RemotePlan.cancelled`; keep both and run `syncguard` and `customfields-syncdiff` |
| `src/ui/overlays.tsx` | +2 / -0 | +15 / -4 | **1** | keep both |
| `src/ui/print/PrintConfigScreen.tsx` | +5 / -10 | +7 / -2 | **1** | keep both; re-run `print-config` and `customfields-print` |
| `scripts/assert/syncconflict.assert.ts` and `mutations/syncconflict.mjs` | +127 / -1 and +21 / -0 | +134 / -16 and +40 / -0 | **1 each** | keep both sets of cases and mutations |
| `scripts/assert/einvoice-wiring.assert.ts` | +2 / -1 | +2 / -1 | **1** | the same pinned line moved on both sides; take main's text and re-apply the party-terms tail wording |
| `docs/design/back-nav-spec.md`, `docs/design/custom-fields-spec.md` | n/a (untracked in `main`) | +22 / -5 and +870 / -102 | 1 each (modify/delete) | gone if step 1 above is done; else take the branch's version |
| *Textually clean, still coupled (read the merged result):* `src/core/voucherCancel.ts` (+19 vs +2: the cancel shell is exempt from the custom-value carry; `useCancelVoucher` passes `udf: 'exempt'`), `src/state/store.ts` (+3 vs +46 / -13: the five `Screen` entries, `UdfRefusal`, `checkDelete`), `src/sync/sync.ts` (+13 vs +16: caps against wave 3's cancel notice), `src/core/shortcutRegistry.ts` (+3 vs 2 / 2: no chord may have been added by this branch), `src/core/splitCompany.ts` (+4 vs +27 / -7: variants carry), `src/ui/modern/CommandPalette.tsx` (+1 vs +28: late commands), `src/ui/print/{BulkPrint,PrintVoucher}.tsx`, `src/ui/share/SharePanel.tsx` (wave 3's share panel against this branch's Close), `scripts/assert/party-terms.assert.ts` and `mutations/party-terms.mjs`, `scripts/perf/mirrors.mjs` (both sides fixed the `altWalk` mirror), `scripts/assert/mutations/run.mjs` (copy-on-write clone) | | | 0 | one pass of `git diff main..` on each, then the suites named |

`src/data/db.ts` changed in `main` (wave 3's `cancelConflict` in `applyRemoteGuarded`) and not here; neither side bumps Dexie.

**After the merge, run:** `npx tsc --noEmit`; `npm run build`; the full `npm run assert` (expect main's suites plus this branch's 37 new ones); `npm run assert -- syncconflict customfields-sync customfields-rules customfields-identity voucher-cancel party-terms einvoice-wiring sidenav zoho-scaffold reportscenter navstack box-close`; `npm run perf -- --enforce`; the golden compare (`goto.ranking`: this branch's capture differed only on `crm` and `edit`, both from wave 3's own "Credit Limits" command, which `main` already makes late); the three 938-plus-screen sweeps in chunks. **Two harness items were expected to need repair on the merged tree and were repaired (section 12):** the viewport leak (section 1), and `scripts/shots/recipes/einvoice-safety.mjs` `ledger-tail-reachable`, whose loosened Enter loop is replaced by `main`'s single Enter that must raise Accept (wave 3's fix round made Enter on the PIN code, the end of the first tail group, raise Accept; the loosened version this branch carried would have passed even if that regressed).

**Merge outcome (done: `git rebase --onto main wave3-wip cf-back-nav`, onto `5aee765`).** All 15 conflicted files were resolved with both intents kept; the two spec files were kept as this branch's versions. What was re-applied BY HAND because the code moved or was rewritten:
- `backend/src/sync.js`: this branch's side was taken (the push body and `loadHistories` left the file). Wave 3's `|| needsCancelCheck(c)` is now a plain named import in `backend/src/histories.js` (`planReads(..., cancelCheck = needsCancelCheck)`); the namespace seam note is gone. `syncconflict` reads `histories.js` for it now.
- `backend/src/conflict.js` `classifyPush`: wave 3's rule 1b (a cancelled voucher stays cancelled) FIRST, then this branch's custom values (now 1c) and the two company policies (now 1d); the report chain is `if (cancelKept) ... else if (!conflict && (kept.length || udfLost.length))`, and the cancel row reports `protectedKept: kept, udfLost`. A cancelled shell is exempt from the custom-value carry because the carry runs on the shell the rule put in place.
- `src/core/importMap.ts`: main's hardened version with the custom-column edits laid onto it. A custom problem refuses the row through main's `fail()` BEFORE the repeat note, the outside-year count and the number are taken, and a refused row is not counted as "used" by the file (it must not make a later identical row read as a repeat of one that was never imported).
- `src/ui/VoucherScreen.tsx`: `walkVoucher` (this branch: PgUp / PgDn and the rail's Prev/Next share it) now calls wave 3's `adjacentVoucher` and `walkable` (core/voucherWalk.ts).
- `src/ui/forms.tsx`: `tailGroup` / `groupEnd` (wave 3) and the custom flags (this branch) side by side; `next()` reads `if (idx >= visible.length - 1 || tailAhead || groupEnd) requestAccept(false, fresh)`.
- Suite pins re-pointed to the merged text (each one pinned a literal the OTHER branch had changed): `einvoice-wiring`, `party-terms`, `syncconflict`, `customfields-sync` (a voucher put now loads `latestOther` once; the list chip), `cancel-fixes` (`walkVoucher`; `saveVoucher(shell, { udf: 'exempt' })`), `importmap` (`ctxOf(..., enforce)`), `box-close` (the Share party list's shield is `share-overlay sp-pick`), `share` (the dispatch panel and the reason box are `Box` now), `voucher-resave` (the walk rule is `walkable`), and five mutations (`customfields-import` I27 and I28, `customfields-sync` M45, `importmap` N32, `syncconflict` the title and the histories.js load).

## 8. Proposed additions for CLAUDE.md

**Status bullet (Phase 30, wording ready to adapt):** "Phase 30: custom fields v2, reporting by field, stock by field and the Back control (2026-10-24; branch `cf-back-nav`; NOT parity; five `beyond` rows).
BUILT, REVIEWED and FIXED in a worktree; not merged, not deployed. An administrator defines fields on ledgers, stock items, vouchers (header), item lines, employees and leads, with 12 types (text, long text, number,
amount, percent, date, yes/no, choice, multi-choice, email, phone, link) plus a lookup; a company policy (Settings > Workspace) says whether a mandatory field binds an OLD voucher (new entries only, from a date, always;
default new entries only). Five analysis reports (Reports Center, 'Analysis by field'): Summary by field, Income and expense by field (ties to the Profit and Loss nett), Bills by field and Balances by field (tie to the
group balance), Stock by field. A field marked 'Tracks stock' keeps stock by Colour, Size or any variant. The row filter on the Day Book, registers, Ledger Vouchers and Bills offers the company's fields. The top bar has a Back
control that names its destination and asks before it discards work; every dialog has a Close; the rail's Esc button runs Esc. Server half: `backend/src/{conflict,histories,pushRun,mismatch,restWrite,rest}.js`, no migration;
capability `udf2`. Spec: `docs/design/custom-fields-spec.md`, `docs/design/back-nav-spec.md`; build notes `docs/design/cf-back-nav-build-notes.md`. Gate: tsc 0; build green; 137 suites / 65,967 assertions; perf 15/15;
938 screens, light and dark compact clean, light comfortable clean except a recipe viewport leak. Not built: CF-4b (column and sort through the grid; no grid), CF-6b, CF-9e, CF-9f, CF-10, CF-11, CF-12, B-5, B-6. Checked in
headless Chrome on macOS only; the deploy smoke was never run on the VM."

**Gotchas (Phase 30, proposed text; each is a lesson the build paid for):**
- **A capability flag must say what the build does, and its tripwire must read the funnel, not the flag.** `UDF2_BUILD` stays false until every writer carries custom values; while false the cloud treats the build as an older app and puts every omitted value back (a Zone could never be cleared). The suite reads the funnel (`core/voucherFields.ts`, `ui/udf.ts`) and fails when flag and funnel disagree either way; whoever removes the funnel flips the flag in the same change.
- **Every writer of a document that carries a new bag is classified, and the unclassified count is a ratchet.** `customfields-rules` lists every `saveVoucher(` / `saveDoc(` caller (24 groups, 0 pending). Entry forms call `checkUdf`; system writers pass `{ udf: 'exempt' }`, which is NOT `system: true` (that skips the write guard and the user stamp). `mergeUdf` returns `{ ok, udf, problems }` and never truncates; a caller that gets `!ok` must not save.
- **Key a server keep-rule on the capability or on omission, never on "states no base"; read the newest row of ANY device.** Every client since Phase 24 stamps a base while signed in and a signed-out row carries none. `CAP_KEPT` (`entryPolicy`) keeps whatever the base for a writer that does not state `udf2`; `OMIT_KEPT` (`ui`) restores only a `ui` the write omits (a Phase 27 administrator's edit must stand). In SQL, `DISTINCT ON` over `tbl = ANY(...) AND doc_id = ANY(...)` matches the cross-product of two arrays: use a lateral `LIMIT 1` per pair, sliced at 500.
- **A repeated identity is not an identity.** The server pairs a line by strong identity; for a repeated one (a size run is that by design) only when the rest of the line is unchanged, else it records the loss (`udfLost`). Never label by position after "delete line 1 and retype it at the bottom".
- **Scope decides entry, never history.** A field not asked on a voucher type still buckets by its stored value. `partyOf` honours `partyId` only for the invoice bases (the Tally importer sets it on every type); the party side per base is the invoice logic extended to Receipt and Payment.
- **A report by field must tie to the book, to the paisa, on the book's own basis, and say what it left out.** Income and expense by field ties to the Profit and Loss nett (an Income- or Expenses-nature ledger outside the six reserved groups is a memo row); Bills by field ties to the GROUP balance through Pending / Unapplied / Not bill-wise (the Bills reports decide a side by sign and list on-account rows, so equality with them is asserted only on a book where it holds); a remainder shows as a visible not-attributed or Unallocated row and a residual larger than pieces plus 2 paise is never absorbed.
- **Stock by variant has one merge point.** A count by variant writes `countPart` lines that `stockMovements` merges into one whole-item count, so none of the four engines that resolve a count reads a resolver; pooled costing means the variant cells carry quantities and Opening and Closing value only; Hide and Delete are refused while a field tracks stock; a negative Not set is the one detector for a writer that bypasses the screen (REST cannot warn until CF-6b).
- **An editor panel opened earlier must save THREE-WAY.** Writing the draft's every member over the stored row resurrected another administrator's Delete or Hide; the draft is compared with the row the untouched draft would have written and only changed members are written (`staleSaveProblem` refuses when the status moved or one member changed to two answers). A rename keeps the old name (`FieldOption.was`) so an older app's label still reads as the option.
- **A new surface's discovery words are not lent to Go To.** `paletteVocab.ts` lends every surface's words to Go To's command for the same destination, where an alias outranks every title: `size`, `user`, `tag`, `colour`, `zone` moved first rows (golden `goto.ranking`). Menu-only words go on `MENU_ONLY`; the five analysis rows and the entry-rules command are late.
- **A dialog needs a way out for a mouse and must stay inside the window.** `Box.onClose` is required; `scrim="close"` only for a dialog that holds no typed work (a tick list is typed work); a dialog whose body can grow asks for `boxClass="box-fit"`; every new `Box` or `ListPrompt` adds one row to the 5.2 table in `box-close.assert.ts`. A show-once secret is unsaved work (`useDirty`).
- **A control that refuses must re-judge at the click and say why.** `planBack` carries a `why` for each refusal; `backAllowedNow` reads the pending question, the busy probe and the DOM at the click, the dimmed look is a poll; the voucher registers `useBusy` so Back cannot leave during an Accept; only `none-root` may disarm the browser's Back in phase 2.
- **A rail button that says Esc must run Esc** (`runEscape`); the rail swallows its own Esc, so with focus inside it the event is dispatched on the body. `backnav-esc` holds the 120-row table.
- **A harness recipe must restore the viewport it changed.** `Emulation.clearDeviceMetricsOverride` drops the sweep's 1440x900 window to the native 1440x813 for every later screen; two comfortable-density screens fail only then (a 938-screen sweep outlasts every old budget: chunk it, `--timeout 7200`).
- **A popup measured under reduced motion reads the OLD place** (transitions of 0.01 ms): switch transitions off for the measuring and the write (`placePopup`).
- **When a function moves out of a file another branch edits, a textual merge keeps one side and silently drops the other's one-line OR** (`loadHistories` and `needsCancelCheck`). Read the other side's predicate through a namespace and prove the load with a suite on both sides of the merge; replace it with a named import after.
- **On a full disk, a mutation runner must clone copy-on-write** (`COPYFILE_FICLONE`); each headless Chrome profile grows about 55 MB a minute in Code Cache. The disk sat at 98 to 100 percent for this whole build.

## 9. Parity and beyond rows

**Parity (`docs/parity-data.json` `features`): no row moves; 250 / 58 / 73 (73.2%) stands.** A search of the 381 rows for udf, user-defined field and custom field finds none (`calc-userdefined` is
unrelated). `tdl-customization` (P3, missing) STAYS missing: a no-code custom field is not TDL, and the row's note may say so. `report-range-filter` and `report-value-filter` stay partial (the filter now offers the
company's fields on four reports, still not on the main statements, Stock Summary, GST, TDS or payroll). Nothing was compared with a live TallyPrime. Do not fold any of this into the percentage.

**Beyond (`beyond`, 41 rows at the last read of `main`; add five, giving 46), each a draft note that must keep the honest limits:**

| id | name | phase | note (draft) |
|---|---|---|---|
| `b-custom-fields` | Custom fields on masters, vouchers and item lines | 30 | Define fields on ledgers, stock items, employees, leads, voucher headers and item lines (12 types plus a lookup), with scope, defaults, inheritance from the party or item, print switches, hide and tombstone delete, starter sets and Fill blanks (one undo). A mandatory field is enforced at every door per the entry rules; the server keeps custom values an older app omits (capability `udf2`). BUILT, REVIEWED, NOT DEPLOYED; deploy smoke never run on the VM; headless Chrome on macOS only. Not built: the same field as a grid column or sort (CF-4b), Tally UDF import (CF-10), field-level access (CF-11) |
| `b-entry-rules` | Company entry rules for mandatory fields on old records | 30 | An administrator decides, per company and per field, whether a mandatory field binds an old voucher or master (new entries only, from a date, always; default new entries only), with a consequence count. Settings > Workspace; synced with older-client protection (`Company.entryPolicy`, `CAP_KEPT`). BUILT, NOT DEPLOYED |
| `b-field-reports` | Analysis by field: Summary, Income and expense, Bills, Balances | 30 | Five reports in the Reports Center ("Analysis by field") that group by any field wherever it lives (party, item, header, line, posting ledger) over any voucher family; profit ties to the Profit and Loss nett, receivables and payables to the group balance; banded dimensions for text, date and number fields; the same fields filter the Day Book, registers, Ledger Vouchers and Bills. BUILT, NOT DEPLOYED; no grid column or sort yet |
| `b-stock-variants` | Stock by a variant field (Colour, Size, any) | 30 | A line field marked Tracks stock keeps stock by up to three variant fields: Stock by Field screen, size-run grid, variant on the voucher, Physical Stock by variant, Opening by variant, "Split Not set", Stock Summary group mode. One pooled cost per item; per-variant costing and variant-aware manufacture NOT built. BUILT, NOT DEPLOYED |
| `b-back-control` | Back control, dialog Close and rail Esc for mouse users | 30 | A top-bar Back that names its destination, asks before discarding typed work and refuses during an Accept; a Close on every dialog; the rail's Esc button runs Esc. The browser's Back, thumb button and swipe are NOT built (B-6). BUILT, NOT DEPLOYED |

**Other counts to update:** menu 66 rows with 27 `+` creates; Reports Center 106 reports in 15 categories; five new screens (`fieldSummary`, `fieldPl`, `fieldBills`, `fieldBalances`, `fieldStock`); sweep 938 screens before the rebase and 947 after it (it was 697 on wave 3); 137 suites / 65,967 assertions before the rebase, **162 suites / 168,165 assertions on the rebased tree** (section 12).

## 10. Docs for the orchestrator to update after the merge

- `CLAUDE.md`: the status bullet and gotchas of section 8; the sweep, suite and menu counts; the Server paragraph (new backend files, capability `udf2`); `docs/design/` index mentions.
- `docs/HANDOFF.md`: a new section (2h) with the file map, the deploy order of section 6, the smoke and the open list of section 4; fix §2f's "Deployed" line only if the merge changes it.
- `COMSKY_BOOKS_PRODUCT.md`: the five features as beyond-Tally capabilities with the honest status (BETA, not deployed).
- `docs/parity-data.json`: the five beyond rows (section 9); `docs/research/ROADMAP-ADDENDUM.md`: the not-built list of section 4.
- Owner questions still open (custom-fields-spec section 18): 4 (field-level access), 6 (Tally UDF import, needs three real exports), 18 (per-variant costing), 21 (employee and lead fields in reports); back-nav Q3 and Q8 (build B-6?).

## 11. What this pass verified, and what it did not

**Verified while writing this file (2026-10-25):** `npx tsc --noEmit` exit 0 and `npm run assert` 137 suites / 65,967 assertions, all green, on the tree committed with this file; the 37 new suites and
their sum (15,240); `UDF2_BUILD = true` and the funnel comment in `src/sync/caps.ts`; `Box.onClose` required in `widgets.tsx`; "Skip TDS" and `useBusy` present in `VoucherScreen.tsx`; no `backend/sql`, `src/data`, `electron`
or `package.json` change against the snapshot; the menu literal (66 rows, 27 creates); no "Not available yet" in `src/ui/reports/field*.tsx`; the viewport leak still present in `cf-voucher.mjs` and
`cf-line-column.mjs`; the overlap list and the conflict-hunk counts of section 7 (a temporary index and a throwaway commit object, nothing in the real index or refs; `git merge-file` per overlapping file; this
git has no `merge-tree --merge-base`); the diff of `src/data/db.ts` in `main`; that the 7 untracked docs in `main` are byte-identical to the snapshot.

**Not re-run for this file:** `npm run build`, `npm run perf`, the shot sweeps, the mutation lists, any recipe or flow. They ran in the final gate on a tree whose fingerprint (`5c75790b...`) was identical before and after the gate;
the five modified files and the one new recipe committed with this file were already part of that tree, and the only file added since is this one. **Never done:** the deploy smoke on the VM; any run on Postgres 16 or Node 22; any real older shipped build against the real server;
a live TallyPrime, the GST portal, the IRP or an e-way bill portal; real key presses on Windows, Linux, Firefox, Safari or Electron; real touch or a screen reader; a time-to-paint measurement in a browser. The perf figures at 250,000 vouchers
came from a busy machine (load 3 to 19). The dev server on port 5181 was left running; port 5173 was never touched.

## 12. The rebase onto `main` and the fix round after it (2026-10-25)

**State.** `git rebase --onto main wave3-wip cf-back-nav` is done: one commit on `5aee765` (`main` with wave 3 as committed and fixed, `d941298`'s perf mirror fix and the batch/serial engine merge). A review of the
rebased tree asked one question, "do the wave 3 fix hunks survive?" (`git diff wave3-wip 2cee555`, 100 files), and answered yes: 67 of the 98 code, suite and doc files are byte-identical to `main`, the 31 this branch also
touches were read hunk by hunk, and the moved server code (`pushRun.js`, `histories.js`, `mismatch.js`, `restWrite.js`) was driven through the real `runPush` and `loadHistories` (an older writer with no base, no cursor or a
capability list without `udf2` still cannot un-cancel a voucher; a write that carries `cancelled`, and a restore, are untouched). It found no app defect: one major finding about a harness recipe and four minor ones, all
below. Only files under `scripts/` and `docs/design/` changed in this round; `src/`, `backend/`, `electron/` and `package.json` are identical to the commit that went into it.

**What was fixed, and how it was shown.**
1. **`ledger-tail-reachable` (major).** This branch had loosened the recipe (Enter up to 8 times until Accept or the PIN caution showed) against wave 3's unfixed tail, and the rebase carried it through without a conflict
   because the wave 3 fix round never touched the recipe. With the tail fixed on `main` (Enter on the PIN code, the end of the first tail group, raises Accept), the loose recipe would have passed even if that regressed. The
   recipe is now `main`'s byte for byte (`git diff main` on the file is empty). Proof it bites: with `groupEnd` in `src/ui/forms.tsx` forced false for one run, `ledger-tail-reachable` failed ("timed out waiting for Enter at
   the end of the tail asks to Accept"), and it passes on the real tree; the edit was reverted (the file is identical to the commit). The three places in this file that listed the recipe as an open defect are corrected.
2. **Five stale mutations (minor; none a merge effect).** A mutation whose text is no longer in the file kills nothing. Re-pointed at the current code: `cancel-fixes` "the Cancel button cannot be hidden or renamed"
   (`railLayout.ts` now reads `chord: chordOf('cancelVoucher')`; stale in `main` too), `customfields-usage` D8 and D9 (the merge in `mergeOptionLists`), `uiprefs` C5 (`platformLabel(e.chord ?? id)`; stale in `main`) and
   `viz-tokens` V19 (the bar path now carries `dv-ahead`; stale in `main`). All four lists run on the rebased tree: `cancel-fixes` 34 of 34 killed, `customfields-usage` 33 of 33, `uiprefs` 31 of 31, `viz-tokens` 27 of 27.
3. **The numbering flow's coin toss (minor).** `numbering-flow.mjs` built its "new voucher" from the first Payment in the table (random id order); when that was the voucher T5 had renumbered, the copy carried a
   `numberLog` and "signed out: a new voucher comes back as it was" read `hasLog` true. The copy now drops `numberLog` as well as `numberStatus`. 3 runs in a row: 64 checks, 0 failures. A coin toss is not shown fixed by 3 passes;
   the cause (the copy inheriting a log) is gone whichever Payment comes first.
4. **The two specs' status lines (minor).** `back-nav-spec.md` and `custom-fields-spec.md` still opened with "design only. Nothing here is built". Each now says what is built on this branch, what is not (B-5, B-6, B-4's grid half; CF-4b,
   CF-6b, CF-9e, CF-9f, CF-10, CF-11, CF-12, CF-5b) and that the build notes win where they differ; the original status is kept below it.
5. **The harness viewport leak (known defect).** `cf-line-column.mjs` (`clearSize`, used at the end of `cf-line-geometry`) and `cf-voucher.mjs` (two screens, `cf-voucher-geometry` and `cf-voucher-policy-lane`) called
   `Emulation.clearDeviceMetricsOverride`, which drops the run's 1440x900 window to the browser's own 1440x813 for every later screen. `clearSize` and `restoreSize` now put the run's own size back (`p.opts.width`,
   `p.opts.height`, as `cf-fixes.mjs` already did). Two more layers so a recipe that FAILS cannot leak either: `shoot.mjs` sets the run's size again after every screen (the same call `page.init` makes, so it changes nothing
   for a screen that did not touch the size), and `_cf.mjs` gained `reveal(p, sel, { text })` (scroll the target to the middle, settle), used by `cf-hr-editor-panel` (More options) and `cf-masters-from-editor` (the More options
   button and the Sundry Debtors tick), plus a fix in `click`: a point answered by a clipping ancestor that also holds the target is no longer a "hit". Proof: the two screens pass in light comfortable inside the same run that
   first calls `cf-line-geometry` (44 screens, 44 of 44 ok), and on their own at 813 px high (the window that used to fail them), both with `--strict`.

**Gate on the rebased tree after these fixes.**
- `npx tsc --noEmit`: 0 errors. `npm run build`: exit 0. `npm run perf -- --enforce`: **15 of 15 gated pass**, exit 0 (224.7 s; the machine's load was 7 to 9; 24 reference rows over their line, ungated).
- `npm run assert`: **162 suites / 168,165 assertions, all green.** All 125 suites `main` has are present (none lower; 17 higher, +1,057, because both sides pinned the same screens), plus this branch's 37 new suites
  (15,241 assertions); `main` alone is 151,867. Sections 0 to 11 quote 137 suites / 65,967, the number on the tree BEFORE the rebase.
- Screens, `--strict` (any console error fails the screen), against the dev server on 5181: **all 187 `cf-*` screens in light comfortable and all 187 in light compact, 374 of 374 ok** (4 chunks of 94 and 93);
  the 44-screen light-comfortable stretch above; the three `ledger-tail-*` screens in light compact. `--list` on this tree prints 947 screens (938 before the rebase).
- Real-key flows on the rebased tree: `cf-policy-flow` 55 of 55, `cf-line-flow` 55, `cf-voucher-flow` 70, `numbering-flow` 64 (three runs), `sidenav-flow` 355 of 355 (light compact; the branch commit had updated its expectations
  for the Preferences row and 27 `+` buttons, so the "stale since Phase 27" note in section 2 was out of date).
- The review's own runs on the same tree (before these fixes; the fixes do not touch what they exercised): `syncconflict` 32 of 32 mutations, `party-terms` 37, `importmap` 89, `vchnumbering` 78, `voucher-cancel` 34, `share` 62,
  `popup-inview` 11; the 74 wave 3 screens 74 of 74; `cancel-party-flow` 77, `eway-flow` 87, `export-share-flow` 105, `single-entry-flow` 111, `numbering-flow` 64.

**Not done in this round.** The three full sweeps (947 screens each, light compact, dark compact, light comfortable): only the `cf-*` family, the 44-screen stretch and the screens named above were re-run, in light, and nothing in dark,
so the shoot.mjs change (one extra size call after each screen) is proven on those screens and not on all 947. The other mutation lists were not re-run here (the review ran seven; the five stale ones are the four above, counted
over all 5,544 mutations by the review). The deploy smoke on the VM, Postgres 16 and Node 22, real key presses on other platforms, and a real older build against the server: still never done. Headless Chrome on macOS only.
