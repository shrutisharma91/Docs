# Reconcile note for the orchestrator: the tracking spec against the running CF-9 build

> **Case that applied: B. Slice CF-9 HAS started** (and is further along than the tracking spec's last read of it). Nothing under `/Users/Akshay/tallyreplica/.wt-cf` was edited; this note carries the adjustments instead. Written 2026-10-23, about 07:20 IST, against the worktree as it stood then (it is still moving: its files carried times up to 07:08). Nothing was gated or run, with one exception: a read-only calculation of CF's own `countDeltas` (bundled by `esbuild` into the scratchpad, section 5.3). Evidence tags: **[C file:symbol]** read in the worktree, **[run]** that calculation, **[J]** my judgement, **[unverified]**.
> Companion to `docs/design/batch-serial-labels-spec.md` (called "the tracking spec" below), whose Appendix R is the earlier reconcile note this one supersedes where they differ.

---

## 1. The case, and the evidence

- Worktree `.wt-cf`: branch `cf-back-nav`, HEAD `f446674` ("WIP checkpoint 2 ... unreviewed"), snapshot `wave3-wip` = `baa0c45`; `git status --porcelain` lists 115 entries. [C]
- The task's own test: `git diff --stat wave3-wip -- src/core/inventory.ts src/core/valuation.ts` reports 284 and 6 lines changed. Against HEAD the CF-9 work is **uncommitted**: `inventory.ts` 279, `openingStock.ts` 97, `stockSummary.ts` 168, `splitCompany.ts` 34, `valuation.ts` 6, `VoucherScreen.tsx` 413 lines. Untracked and new: `core/variantCore.ts` (139 lines), `core/stockByField.ts` (676), `core/variantEntry.ts` (215), `core/stockAgeing.ts` (137), `ui/reports/stockPanels.tsx`. `types.ts` already carries `InventoryLine.countPart` and `OpeningStockAlloc.variants` and is clean against HEAD. [C]
- What exists, by slice: **9a** the engine (`byVariant`, `variantOf`, the count group, `countDeltas`, `allocateValue`, `openingVariantMap` with both trust tests, `pendingOrders` per-line allocation, `stockDeltaByItem`); **9b** the screen (`ui/reports/fieldStock.tsx`, `SizeRunPanel`, the Stock Summary `'field'` group mode); **9c** the entry rules (`core/variantEntry.ts`) and their wiring in `VoucherScreen.tsx`; **9d in part** (`splitCompany.ts` carries variants and maps pending order lines by `line`; `trackedStockHealth` and `splitNotSetPlan` exist as pure functions). **Not written:** the Opening-by-variant panel, the Split Not set screen, the Data Check rows. [C]
- Proof artefacts in the tree: suites `customfields-stock` (81 KB) and `customfields-stockentry`, the fixture `cfstock-fixture.ts`, **frozen copies of the pre-CF-9 engine** (`scripts/assert/refs/inventory.pre-cf9.ts`, `stockflow.pre-cf9.ts`), a mutation list of 64 entries, recipes `cf-stock` and `cf-variant-entry`. I did not run any of it.
- The CF spec in the worktree (`docs/design/custom-fields-spec.md`, 399 KB, last written 22 Oct 04:36) is the evolved copy the tracking spec cites; the main-tree copy (200 KB) is older. Its CF-9 text is at AD.3.1 to AD.3.9 and the slice rows CF-9a to CF-9f.

**Consequence for the order of work (tracking spec 15.3, T27, question 22).** The CF lane's voucher-screen work (CF-2, CF-3 and now CF-9c's first version) is already in the tree, unreviewed. The plan "Back overlays, CF-2, CF-3, **T-2a**, CF-9c, T-2b" can no longer be followed literally, and there is no `useStockDims` hook for CF-9c to adopt (section 4, row 3). The wait before tracking reaches a voucher is now the review, commit and rebase of that diff, not 23 to 27 more days of CF voucher building. [J: the review time is unknown.]

---

## 2. The agreed key, in the words a CF builder needs

1. **A stock position is a cell: `(godown, batch, variant)`, each part optional.** A serial or IMEI is a **unit identity under a cell**, never a fourth part (tracking spec 2.1, T1).
2. **CF-9's `byVariant` is the variant projection of that cell map and keeps exactly its shipped shape**, `Map<VariantKey, { qty, byGodown }>`. The tracking work adds a sibling `StockPosition.cells`, computed **only when a cell resolver is passed**, beside `byVariant`, so a caller that passes only `variantOf` gets exactly what CF-9a gives it today. No item-level figure (qty, value, avgCost, `byGodown`) ever depends on either resolver.
3. A cell key is **length-prefixed** (`<len>:<godown><len>:<batch><variant>`), never separator-joined, because CF's own variant key already contains U+0001. It is never stored; a stored row keeps its parts in separate fields.
4. **Costing stays per item** for batch and variant (CF24). A serial item may opt into a per-unit method `Specific`; per-variant costing (CF-9e) is not part of this agreement (section 3.4).
5. **A count is always a count of the whole item at its godown, made in parts.** The tracking work reuses CF's `countPart` flag and merge unchanged, adds one flag (`eod`, "the close of its date") read by the same generator, and treats `countDeltas` as the variant-only case of a generalised `countSpread` over opaque keys.
6. The variant is read from the line's own `udf` through CF's resolver and is **never stored on the line** (CF25). A cell's variant part is read the same way, so Stop tracking and a deleted field need no tracking-side change.

**Paste-ready text** for the worktree's CF spec, if the orchestrator chooses to add it when the CF builder is quiescent (targets: AD.3.1 item 2, "A line-level variant", about line 383; the CF-9a row about line 1827; the CF-9c row about line 1829 of the 04:36 file). I did not apply it.

> **Reconcile with the tracking spec (batch, serial and IMEI tracking, 2026-10-23).** A stock position is a CELL: (godown, batch, variant), each part optional; a serial or IMEI is a unit identity under a cell and never a fourth part. `byVariant` is therefore the variant projection of that cell map and keeps exactly the shape written here; the tracking work adds a sibling `cells` and a `stockCellOf` resolver option beside `variantOf`, and no CF reader changes. CF-9 keeps: (1) `VariantKey` an opaque string, its encoding (option ids joined by U+0001, Not set = the empty string) frozen once `OpeningStockAlloc.variants[].key` or a screen parameter has been stored; (2) `StockMove.parts` the whole original lines, never stripped, with the merged count line dropping only `udf` and `countPart`; (3) `countDeltas` free of variant logic, a pure function over opaque keys; (4) every item-level figure independent of the resolver; (5) the variant read only from `line.udf` through `variantResolver`. CF-9 avoids: naming anything `cells`, `stockCellOf`, `CellKey`, `CellParts`, `heldCells` or `eod`; refusing a tracking variant on a batch-tracked item; per-variant costing (CF-9e) on an item the tracking work tracks by batch or serial until the two are reconciled; reading batch fields (`batchName`, `expiryDate`, `mrp`, `serials`) off a merged count line, which holds the first part's.

---

## 3. What CF-9 must change, keep, and avoid

### 3.1 Required in CF-9 code: none

Appendix R's fallback was written for exactly this situation and the shipped code satisfies its preconditions: the key is an opaque string (`variantCore.ts` `VariantKey`), `countDeltas` is key-agnostic and holds no variant logic, `StockMove.parts` keeps the whole original lines and the merge drops only `udf` and `countPart` (`inventory.ts` `mergeCountParts`), `openingVariantMap` takes the item, the resolver is absent for every old caller (`ValuationOpts.variantOf`), and nothing refuses a variant on a batch item. No batch field is read by any CF-9 reader (grep of `stockByField.ts`, `variantEntry.ts`, `variantCore.ts`, `fieldPlStock.ts`, `stockAgeing.ts`, `fieldStock.tsx`: none). [C]

### 3.2 Recommended inside CF's own review pass (comments or a pure move; no behaviour, no new scope)

| # | Where | What | Why | If skipped |
|---|---|---|---|---|
| R1 | `inventory.ts` `stockPositions` | Optional: name the per-movement variant effect (the inline ordinary-line `vmoves = [...]`, `countVariantMoves`, `applyVariantMoves`) as one exported function | Appendix R item 1; T-0b's `movementCellEffects` then sits beside it | T-0b adds the sibling next to the three existing call sites. **Prefer skipping R1: see 5.1** (27 mutation anchors pin that inline text) |
| R2 | `variantCore.ts` `countDeltas`; `inventory.ts` `StockMove.parts` | Two comment lines: "the variant-only case of `countSpread`; keep it key-agnostic" and "whole original lines; the tracking work reads `batchName`, `expiryDate`, `mrp`, `serials`, `eod` from them" | the contract is then written where the next editor looks | none; the paste-ready text carries it |
| R3 | CF-9d's Opening-by-variant panel (not yet written) | See 3.5 | cheapest moment to fix its shape | rework when the Opening-by-cell panel arrives |

### 3.3 Keep stable after review (the surface T-0b compiles against)

| Symbol (file) | Shape to keep | Who reads it |
|---|---|---|
| `VariantKey`, `NOT_SET_KEY`, `VARIANT_SEP`, `segmentOf`, `keyOfSegments`, `segmentAt`, `segmentsOfKey`, `canonKey` (`variantCore.ts`) | opaque string; segments joined by U+0001; Not set is the empty string | every CF reader; stored in `OpeningStockAlloc.variants[].key` and in screen parameters, so **frozen once any of those ships** |
| `countDeltas(held, itemDelta, parts \| null, wholeKey)` (`variantCore.ts`) | key-agnostic; residual to the listed key of the largest absolute delta | `stockPositions`, `resolvedMovements`, `stockItemRows`; T-0b calls it |
| `StockPosition.byVariant`, `ValState.byVariant`, `ValuationOpts.variantOf`, `StockMove.variants`, `StockMove.parts` (`inventory.ts`, `valuation.ts`) | as shipped; present only on a tracked item and only with the resolver | `stockFlow`, `fieldStock.tsx`, `reports.tsx`, `splitCompany.ts`, the voucher screen |
| the count group: `InventoryLine.countPart`, `stockMovements` merge, `mergeCountParts` | merge only `countPart` lines of one (item, godown) of one Physical Stock voucher; merged `line` keeps everything of the first part except `udf` and `countPart`; `parts` untouched | every stock reader; `eod` and batch counts ride on it |
| `variantResolver` and `VariantReader.fieldsOf` (`stockByField.ts`), `isTracked`, `variantUdf`, `rowKey`, `trackingFids` (`variantEntry.ts`) | returns `undefined` when nothing tracks; `fieldsOf` gives the fields in order | the allocation panel must **set a line's variant through `variantUdf`**, never write `udf` itself |
| `openingVariantMap`, `openingVariantCheck`, `seedVariantPositions`, `applyOpeningAlloc(p, it, withVariants)` (`openingStock.ts`) | map from the item; both trust tests | `openingCellMap` is read first when `cells` exist (tracking spec 3.6) |
| `outwardVariantWarnings`, `heldVariantRows`, `countProblems`, `isCountPart`, `sourceVariantStart`, `stockHints` (`variantEntry.ts`) | pure; the untracked-item sentence byte for byte | the voucher screen; T-2 widens, never forks |
| `encodeVariantParam` and `decodeVariantParam` (`stockByField.ts`) | CF owns the prefixes `v:` and `s:`; anything else decodes to null | Stock Item Vouchers; the tracking work adds other prefixes (`b:`) and never parses `v:` |
| `UDF_CARRIERS.vouchers.lines.{keep, identity, strong}`, `CAP_KEPT`, `UDF_CAP`, `SERVER_CAPS`, `cleanCaps` (`backend/src/conflict.js`) | tables, as shipped (`keep` is `['countPart']`; capabilities at most 8 names of 16 characters) | T-9a extends in place; there is no separate `mergeRules` list, `carryUdf` is the one merge |

### 3.4 CF-9 must avoid (until the tracking work has merged on top)

- Adding any member or name from this list: `cells`, `stockCellOf`, `CellKey`, `CellParts`, `cellKey`, `cellParts`, `heldCells`, `movementCellEffects`, `countSpread`, `eod`, `StockItem.track`, `serials`, `serialKind`. (No collision exists today in the worktree; grep of `src`, `scripts`, `backend/src`, `electron`.) [C]
- Refusing or warning against a tracking variant on a batch-tracked item (Appendix R item 7). A warning that the combination multiplies cells is acceptable.
- **CF-9e (per-variant costing, gated) on an item tracked by batch or serial.** It would change item-level value, and the `Specific` method values by unit; the two have no defined precedence. When CF-9e is built, the editor must refuse it for such an item. Not a CF-9 change today.
- Reading `batchName`, `expiryDate`, `mrp` or `serials` off a **merged count line** in any new reader: it holds the first part's.
- Storing the variant key on a line, or changing `VARIANT_SEP`, after the first stored `OpeningStockAlloc.variants` row exists.

### 3.5 CF-9d, not yet written: three constraints (all inside its existing scope)

1. The Opening-by-variant panel writes **only** `openingAlloc.variants`, reads trust through `openingVariantCheck`, is a `Box` with `onClose` and a dirty probe (Back rules B5 and B12, as the spec already says).
2. The Split Company writer (`splitCompany.ts`, the block that builds `alloc.variants` from `p.byVariant`) stays as written; the tracking work adds a `cells` writer beside it for an item that has `track`, and `openingCellMap` then wins.
3. The Data Check rows read `byVariant` only.

---

## 4. Corrections the orchestrator should make to the tracking spec (I did not edit it)

| # | Where | Stale or wrong now | Reality (read 07:13) | Edit |
|---|---|---|---|---|
| 1 | header, 1, 2.1, 15.3, 18 item 5, Appendix R status | "CF-9a is written (variants only)" | 9a, 9b and 9c are written, 9d in part, all uncommitted and unreviewed | restate as "CF-9 (a to c, d in part)"; the T-0b gate stays "CF-9 engine files committed, reviewed and green on a rebased main"; those files are also edited by 9b (`stockSummary.ts`) and 9d (`splitCompany.ts`) |
| 2 | 2.1 and 4.2, `ValuationOpts.cellOf` | the resolver option is named `cellOf` | `cellOf` already has **three** meanings in the CF tree or its spec: `lineFields.ts` exports `cellOf(def, row)` (a custom-field cell on a line row, line 103); `stockSummary.ts` `stockFlow` has a local `cellOf(itemId, godownId)` for godown cells (line 180) and forwards options to `stockPositions` beside it (`vopts`, line 228); CF AD.2.5 plans `FlowOpts.cellOf(m)` | rename the **option** to `stockCellOf` everywhere (keep the noun "cell", which generalises `stockFlow`'s godown cell). No other name in the tracking spec collides |
| 3 | Appendix R rows 5 and 6, 15.4, T27, question 22 | "CF-9c is not written; T-2a creates `useStockDims`, CF-9c adopts it" | CF-9c is written **inline**: no hook exists. The variant cell rides CF-3's `cf:` focus; the sites are listed in 5.4. The pre-fill function is `heldVariantRows(of, item, pos, godownId)` in `variantEntry.ts`, not `heldVariants(positions, itemId, godownId, asOn)` | re-scope T-2a: after CF-9c is reviewed and committed, add the `'alloc'` stop and widen the five inline sites; extract a hook only if the CF review wants one. Name the widened function `heldCellRows`. Question 22's "jump T-2a ahead of CF-2" is moot |
| 4 | T1, 2.1, Appendix R fallback, 15.3 | "T-0b re-points `byVariant`'s update to `byVariantOf(cells)` for a batch item" | CF's suite pins the **literal source text** of that update (27 of its 64 mutations are anchored in `inventory.ts`, 4 in `stockSummary.ts`, 5 in `openingStock.ts`, 2 in `splitCompany.ts`), and a separate residual problem exists (5.3) | say **"add `cells` beside `byVariant`, leave CF's inline update alone, and assert `byVariantOf(cells)` deep-equals `byVariant` on every book that has both"**; see 5.1 and 5.3 |
| 5 | 4.2 | "the legacy `byBatch` update is skipped for a merged count movement" | true to the code: CF-9a still books the merged line's first part's batch (`inventory.ts` lines 212 and 227). CF's suites do not pin it: no CF test or fixture names `batchName` or `byBatch` (grep), and the with-resolver identity compares the live engine with itself | keep the skip, but name it in T-0b's commit and the release note as a deliberate change for merged count movements; it depends only on `parts`, so identity with and without a resolver holds |
| 6 | 15.5, T29 | the MCP re-lock is listed for the tracking slices only | `mcp/engine-lock.json` pins 52 files, **five of which CF-9 edits** (`inventory.ts`, `valuation.ts`, `openingStock.ts`, `splitCompany.ts`, `types.ts`; `stockSummary.ts` is not pinned; the CF lane also edits the pinned `integrity.ts`, but not for CF-9), and the CF spec never mentions the lock. The worktree has no `mcp/` (base predates it), so the drift appears at the merge | add to the merge checklist: CF merges, drift check, re-record the lock, **then** T-0b rebases and re-records again |
| 7 | Appendix R row 8 | "T-9a adds `CAP_KEPT.stockItems`, a `mergeRules` list" | `UDF_CARRIERS` and `CAP_KEPT` tables exist as asked; there is no merge-rules list | state the extension point as the two tables |
| 8 | Appendix R row 3 | `StockMove.parts` "keeps the whole lines" | confirmed. The merged `line` also keeps the first part's `eod`, `mrp`, `serials` and batch fields, which is what T-0b needs for `eod` and what it must not use for cells | no edit beyond the warning in 3.4 |

---

## 5. T-0b's day-one sheet (what the builder meets in the CF tree)

### 5.1 Pinned text: add beside, never edit a pinned line

`scripts/assert/mutations/customfields-stock.mjs` anchors on exact source text. A pattern that no longer matches the source is stale and kills nothing (CLAUDE.md, Phase 29 gotcha), and the tracking spec's own 16.1 counts a stale pattern as a failure. The lines T-0b would most naturally edit are pinned:

- `inventory.ts`: the count-branch lines (V2, V3), the `applyOpeningAlloc(... !!variantOf && variantOf(it, BLANK_LINE) !== null)` line (V4), the group lines of `stockMovements` and `mergeCountParts` including `const { udf: _udf, countPart: _cp, ...first } = parts[0];` (C1 to C7), the zero-delta block `if (pv && vmoves && vmoves.length) {` with its comment (C11), `applyVariantMoves`' body (C12, C13), the ordinary-line `vmoves = [{ key: variantOf!(...` line (C14, C15).
- `stockSummary.ts`: the guard `if (m.qty === 0 && m.variants) return;` **together with the next line** `apply(byItem.get(m.itemId)!.total, m);` (A9).
- `openingStock.ts`: the sum and per-godown checks (O1 to O5). `splitCompany.ts`: the `if (p.byVariant && qty !== 0) {` line and the `trackedRows` line (G2, G3).

So: insert new lines **above** a pinned pair (the guard becomes two guards, one added above the pinned one), write `cells` code as a parallel block, never fold CF's three inline pieces into `movementCellEffects`, and where a pinned line truly must change, change its mutation anchor **in the same commit** and say so. This is a second reason, beyond Appendix R's, to prefer "add beside".

### 5.2 CF's oracles, as written

- Identity: with no resolver, or a resolver answering null for every item, `stockMovements`, `stockPositions`, the observed stream, `resolvedMovements`, `stockDelta`, `stockFlow` deep-equal the frozen pre-CF-9 engine on the sample company and 2,000 and 50,000 voucher books. The with-resolver check strips **only** `byVariant` from a position before comparing, so a `cells` member must not appear for a caller that passed only `variantOf`, and `'variants' in m || 'parts' in m` is asserted false on a null-resolver stream, so a `cells` member on a movement must be as absent as those. [C `customfields-stock.assert.ts`]
- The observed stream with a resolver equals the plain one except zero-quantity moves that carry `variants`. A batch-only count that moves stock between batches leaves the item unmoved: T-0b emits a zero-quantity movement carrying `cells` and `stockFlow` must skip it exactly as it skips one carrying `variants`.
- A source scan fails any `pendingOrders(` call in `src/` with fewer than four arguments, so any tracking code that calls it passes the fourth.
- The five Physical Stock vectors, the invariant after every movement (variants add to the item and, per godown, to `byGodown`, to 1e-9) and the allocation checks stay the oracle: T-0b must pass them **unchanged**.

### 5.3 The residual rule (a real disagreement, found by running CF's function)

`countDeltas` gives the part of the engine's `round2` rounding (up to 0.005) to the listed key of the largest absolute delta. Run over cell keys and over variant keys, the two choices can differ. [run] Three cells held at one godown, in kg: (B1, Red) 5.002, (B2, Red) 5.002, (B1, Blue) 5.003 (item 15.007); a count lists all three at 5 each (15.000). The engine's item delta is round2(-0.007) = **-0.01**. Over cells: -0.002, -0.002, -0.006, whose variant marginals are Red -0.004 and Blue -0.006. CF's variant-level call gives **Red -0.007, Blue -0.003**. So `byVariantOf(cells)` would differ from `byVariant` by 0.003 in two variants (integer-quantity items are unaffected; the rounding remainder is then exactly zero).

**Rule for T-0b:** compute the variant deltas with CF's `countDeltas` unchanged, and give the rounding remainder of the cell spread **first to the variant `countDeltas` chose, then within it to the listed cell of the largest absolute delta (first in list order on a tie)**. With it the example becomes -0.005, -0.002, -0.003 (marginals Red -0.007, Blue -0.003, equal to CF). Pin it as a vector in `trackcell`, with 3-decimal and 4-decimal quantities, and assert equality on every book that has both resolvers. Without it, the Stock by field screen (which passes only `variantOf`, `fieldStock.tsx` line 336) and the voucher screen (which would pass both) would print different variant quantities for one book.

### 5.4 Edit-site map (symbols; line numbers are of the 07:06 to 07:08 files and drift)

| File | Site | T-0b / T-2 action |
|---|---|---|
| `inventory.ts` | `stockMovements` sort (line 72) | add the `eod` rank; keep `seq` assignment so no-`eod` books deep-equal |
| | `ValuationOpts` (117), `StockMove` (96) | add `stockCellOf`, `cells` |
| | `stockPositions` init (147), zero-delta count block (182), `byBatch` updates (212, 227), after-valuation apply (233), observe payload | seed cells; add a parallel cells block; skip `byBatch` when `m.parts`; add `cells` to the payload |
| | `resolvedMovements` (291), `stockItemRows` (400, `effV` 443) | only if a batch report reads them; the `variant` argument is positional and stays |
| `stockSummary.ts` | `stockFlow` guard (233), `vObserve` (211), `vopts` (228) | widen the zero-quantity guard by an added line; pass the cell resolver through `vopts` |
| `openingStock.ts` | `applyOpeningAlloc` third parameter, `seedVariantPositions` | `openingCellMap` first; derive the variant seed from cells when present |
| `splitCompany.ts` | the `alloc.variants` block (about 556 to 568), `variantSplit` counter, the `positions` call | write `cells` for an item with `track`; pass the resolver |
| `variantEntry.ts` | `heldVariantRows`, `countProblems`, `isCountPart`, `outwardVariantWarnings`, `stockHints` | widen to cells in T-2; `outwardVariantWarnings` must stay the one writer of the negative-stock sentence (route batch cells through R10, byte-identical text for an untracked item) |
| `VoucherScreen.tsx` | positions memo (1440 to 1443); line literal (1232); save-time `countPart` (1544); Stock Journal destination start (1820); stock hint (1904); Physical Stock pre-fill (2512 to 2525); `countProblems` (2880); negative warning (2999) | add the cell resolver to the memo; the allocation stop; write `eod`, `serials` into the literal; extend the count save, pre-fill and problem checks to batch cells |
| `ui/inventory/reports.tsx` | `decodeVariantParam` use (line 35) | add a batch selector with its own prefix |
| `backend/src/conflict.js` | `UDF_CARRIERS.vouchers.lines.keep` (245), `UDF_CAP` and `SERVER_CAPS` (184 to 185), `CAP_KEPT` (217) | T-9a: extend `keep`, add `trk1`, add `CAP_KEPT.stockItems` |

### 5.5 Performance

CF's budget is `stockPositions` with a resolver at most 1.3 times the plain call at 250,000 vouchers (reference row `cf.stock`, not gated). A company that passes only `variantOf` must pay nothing for cells; the tracking spec's `trk.cells` row (at most 1.15 times) is measured on a book where 1 item in 20 is batch-tracked.

---

## 6. Other CF interfaces the tracking work meets (not CF-9; one line each)

- **Stock item form.** CF-1 puts mandatory custom fields in the Enter walk after the last built-in walk field and optional ones in "the very last tail" (`ui/inventory/masters.tsx`, `...cf.walk`, `...cf.tail`). The tracking work's four-way selector keeps the "Maintain in batches" slot; its tail group goes **between `cf.walk` and `cf.tail`** so the walk still ends where CF ends it.
- **Server carry.** The strong identity `(item, godown, batch, qty, rate, amount)` cannot tell two serial lines of one model at one price apart once an older writer drops `serials`, exactly as for a size run; CF's iterative pairing leaves such lines unpaired and records the loss. T-9a should say so in 10.5 rather than assume the carry restores them.
- **Back spec table 5.2.** Appendix R item 11 (the tracking dialogs) is a Back-spec edit, outside CF-9; CF-9d's own panels belong in the same table.

## 7. What I could not verify

- Nothing was run except the one calculation in 5.3 (CF's `countDeltas` called from a bundle in the scratchpad; the rule's result is arithmetic, not a run). I did not run any CF suite, the mutation runner, the type checker or a build; whether the CF tree is currently green is unknown to me.
- The tree is moving. Every statement here is true of the files as read between 07:13 and 07:20; a later CF edit (a hook extraction, a commit, a rename) can overtake rows 3 and 4 of section 4.
- At 07:22, after my reads, the CF builder was still editing `scripts/assert/customfields-stock.assert.ts` and `scripts/assert/mutations/customfields-stock.mjs` (and had created a temporary `mutations/run-subset.tmp.mjs`). The mutation counts and anchor lists in sections 4 and 5.1 are as read before that; the suite and its anchors may have shifted since, which makes the "add beside, never edit a pinned line" rule more important, not less.
- I did not inspect the sample company's fixtures for `batchName` (the CF test files and `cfstock-fixture.ts` contain none); row 5 of section 4 rests on that grep.
- The tracking spec's own statements about code outside CF-9 (Dexie behaviour, the Electron bridge, hardware) are untouched by this note and unverified here.
