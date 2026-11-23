# Comsky Books: custom fields and reporting tags, object-wise (architecture and spec)

> **Status (updated 2026-10-25, after the rebase onto `main`):** BUILT, reviewed and fixed on branch `cf-back-nav`; BETA, not deployed, never used by a customer. In the tree: CF-0, CF-1, CF-14, CF-2, CF-3, CF-4a, CF-4b-lite,
> CF-4c, CF-5, CF-6a, CF-7, CF-8 and CF-9 a to d. **NOT built:** CF-4b (the field as a column and sort through the report grid, which does not exist), CF-6b, CF-9e, CF-9f, CF-10, CF-11, CF-12, CF-5b. What was built, where it deviates from this
> text and the gate are in `docs/design/cf-back-nav-build-notes.md`: trust that file over this one wherever they differ. The text below is the design as written (its own fix-round sections at the end record what review changed).
> **Original status (2026-10-21):** design only. Nothing was built and no source file was changed. Written 2026-10-21 against `HEAD` = `83d9876`
> plus the uncommitted parity-wave-3 tree (158 changed paths when read: 64 modified, 94 untracked). This is the build contract
> for the owner's request item 3 (object-wise custom fields: mandatory or not, a type, where visible, reaching the reports;
> Zoho's custom fields and its reporting tags) and for the custom-field half of items 1 and 2 (the report grid and quick-saved
> views). Its sibling is `docs/design/back-nav-spec.md` (item 4).
> **Method:** the five research documents written today were read in full (`docs/research/zoho-custom-fields.md`,
> `zoho-reporting-tags.md`, `tally-udf-and-competitors.md`, `custom-fields-code-audit.md`, `back-nav-audit.md`), plus
> `docs/research/wave-a2-plan.md` (all 479 lines), `pipedrive-list-ux.md`, `docs/design/zoho-ux-spec.md` sections 5 and 6, and the
> code they cite, which I re-read where a decision depends on it (files named). Three vendor pages were re-fetched to check the
> facts the design leans on hardest (section A, item 11). Vendor facts are paraphrased; nothing is copied.
> **Evidence tags:** **[C]** read in this repo's code or data this session (file named; line numbers in files the wave-3 build
> is editing will drift, the symbol names will not). **[R]** stated in one of the repo research documents above, which tags its
> own source: I did not re-check it against the vendor. **[W]** a vendor page I fetched this session (through a tool that
> summarises each page, so "not stated" means "the summary did not say"). **[J]** my judgement. **[unverified]** could not be
> checked. There was no live Zoho Books, TallyPrime or Pipedrive account, so every statement about vendor behaviour is a
> reading of documents.
> **What this document does not do:** run any gate (the wave-3 builders own `assert`, `shots`, `build`, `perf` on this tree),
> edit any file but itself and its sibling, or settle anything that needs the owner (section 18 lists those, each with the
> default that lets work proceed).
>
> **Revision (2026-10-21, second pass).** An adversarial review of the first draft raised 38 findings (1 blocker, 19 major, 18 minor) across this document and
> `docs/design/back-nav-spec.md`. Each was re-checked against the code, or the vendor page, before it was accepted. **All 38 reproduced** (two rest on the reviewer's own pixel measurements, which I did not redo), so none is rejected
> outright; in five cases (findings 1, 2, 5, 26 and 31) the fix was adapted rather than copied, and the **Review log** at the end of each spec says which and why. What changed most: the server
> keep-rule now has a stated I/O contract (the first draft's version could not fire in the commonest case, 9.3); an older app is no longer shown a field it would
> round-trip lossy (CF16, 1.1); exempting a writer from the mandatory check is its own narrow option and not `system` (CF17, 3.3); Fill blanks has its own undo (10);
> history is bucketed by the stored value, not by today's scope (CF18, 5.5, 6.3); definitions and options are never removed from a document (CF8, 10); and
> `BACKUP_SCHEMA_VERSION` goes to 3 when custom data exists (CF5, 12). Section numbers are unchanged; the sections named carry the new text.
>
> **Revision (2026-10-22, owner answers).** The owner answered questions 1 to 3 of section 18: (a) Colour and Zone were *examples*, so every analogous scenario must be served (stock split by a line attribute, profit by any field, and so on); (b) whether a mandatory field binds an **old** voucher when it is edited is an **administrator's choice per company** (a Settings > Workspace default plus a per-field override; the default stays "new entries only"). The new section **Addendum: owner decisions of 2026-10-22** follows the decisions table and **overrides any older passage it differs from**; the older passages it changes (decision rows CF2, CF3, CF6, CF9 and CF11; sections 1.3, 1.6, 2.1 to 2.4, 3.1 to 3.3, 4.1, 4.2, 5.1, 5.2, 5.4 to 5.6, 6.2 to 6.7, 7.2, 7.4, 8, 9.3, 9.6, 13, 14, 15, 16, 17, 18, the appendix) were edited in place and are listed in AD.7. **The addendum is design only; it has had one adversarial review (the third pass, next paragraph), and its findings are applied in place.**

**Revision (2026-10-22, third pass: review of the addendum).** An adversarial review of the addendum (lenses L1 accounting integrity, L3 persistence and older clients, L2 keyboard, L8 seams) raised 11 findings (1 blocker, 5 major, 5 minor). Each was re-checked against the code first; **all 11 reproduced**, so none is rejected, and in three cases (the blocker, the server keep-rule and the line-pairing rule) the fix was adapted rather than copied: the **Review log (third pass)** at the end says which and why. What changed most: **a Physical Stock count made by variant no longer moves any item-level figure with the resolver** (the first text resolved a tracked item's count against the variant balance, which changed qty, value and `byGodown` for every caller that passed a resolver; it is now a count GROUP read from the lines' own `countPart` flag in the one movement generator every stock figure uses, AD.3.3); **the `entryPolicy` keep-rule is keyed on the capability a writer states, not on "states no base"** (a marker that is wrong in both directions, AD.1.2, 9.3); **Bills by field ties to the group balance of its side through a Pending / Unapplied / Not bill-wise split**, and says plainly that the Bills reports decide a side by sign (AD.2.7); **the `party` source honours `partyId` only for the bases whose entry screen sets it** (the Tally importer sets it on every type, AD.2.1); **pending orders of a tracked item are allocated per order line, blank variants included** (AD.3.3); **the writers that do not draw the variant cell are listed with what each does** (AD.3.4b); and the value allocation, the opening-by-variant trust rule and the line-pairing rule of the server carry were tightened (AD.3.5, AD.3.7, 9.3). Section numbers are unchanged.

---

## 0. The decisions, in one table

Each can be vetoed; the default if the owner is silent is the one written here. "Vetoable" marks the ones where I expect the
owner to have a view.

| # | Decision | Why (section) | Vetoable |
|---|---|---|---|
| CF1 | **One field definition can live in several places** (a primary place plus `also`): ledger, stock item, voucher header, item line, employee, lead. A voucher-entry-line (`VoucherEntry`) field is **reserved and not built**: cost centres stay the entry-level dimension. The written primary place follows the older-app rule of CF16 | the owner's cases are exactly "one field, several places" (Colour on the item AND on the sale line; Zone on the party AND on the invoice) and one definition means one option list (1.1) | yes |
| CF2 | **A reporting tag is not a new object.** It is a single-valued field (choice, yes/no, lookup) on the voucher header or the item line, and every such field is groupable. A family of analysis reports reads it: **Summary by field** (any voucher family, any measure), **Income and expense by field** (the ledger postings, tied to the Profit and Loss), **Bills by field**, **Balances by field** and **Stock by field** (AD.2, AD.3). Cost categories are left as Tally's amount-splitting dimension | one screen in front of a Tally customer who asked for "add a column"; Zoho's own history of faking a mandatory tag with a required dropdown (6) | yes |
| CF3 | **Answer to "does a field added to Sales appear in the Sales report?": it is automatically AVAILABLE (Columns panel, filter, sort, every analysis report, saved views, exports) and never SWITCHED ON in any standard layout.** A per-field "Available in reports" switch (default on) is the one control; Print is a separate switch (default off). **The "sees exactly today's app" claim covers entry screens, saved documents, print (the Print Configuration form included), standard report layouts and Go To ranking; the navigation additions are deliberate (4.2, AD.7)** | no competitor makes a field show in standard layouts by itself; Zoho's per-report patchwork is its 15-year complaint (4) | yes |
| CF4 | **Option lists have ids that never change.** Rename changes the label only; a hidden option stays valid on old data; **an option is never removed from its list, only marked `gone`, and a new option's label may not equal any existing option's id or label**; a legacy option's id is its original label text, so stored v1 values need no rewriting | history must stay reportable, a saved filter "Zone is North" must survive a rename, and a value that arrives late from a device that had not synced must still be labelled (1.4, 10) | no |
| CF5 | **Values live on the document they describe** (`Voucher.udf`, `InventoryLine.udf`, masters' `udf`), under the key `fid ?? key`. Absent means unanswered; `0` and `No` are values. **No Dexie bump, no new table. `BACKUP_SCHEMA_VERSION` goes to 3, written only when the company holds custom-field v2 data**, so an older app refuses that file instead of eating its values on the next save | every consumer (reports, print, REST, backup, split, undo, change log) already reads the document (12); the backup refusal is the only guard on a restore round trip | no |
| CF6 | **Mandatory is enforced in one pure function at every write path, for v2 definitions only. It binds every NEW entry; what it does to an alteration of an OLD voucher or record is the administrator's choice (AD.1, CF19):** a field marked `req: 'policy'` follows the company setting (default "new entries only", so altering an old voucher is never blocked), `'new'` and `'always'` override it per field. **A mandatory field is asked in the Enter walk, after Narration** (a company that never makes one mandatory sees today's walk, asserted by identity); optional fields sit in a strip below Narration reached by Tab, click or Down. **No new chord.** **The store is the backstop, not the front door:** it judges `create` requireds and CHANGED values only and throws a typed `UdfRefusal`; the entry screens enforce a binding alteration (an `always` field, or a policy that binds, 3.3, AD.1) | the frozen walk, the Phase 24 gotchas, and Accept-on-Enter (2.1, 3) | yes (where it sits) |
| CF7 | **Older apps cannot be allowed to erase or garble values.** Four layers: (0) an older app is shown only fields it round-trips without loss (CF16); (1) the client merges instead of rebuilding (`mergeUdf`) and every voucher writer goes through one carry funnel; (2) the **server** gets a capability marker (`caps: ['udf2']`) and, for a writer lacking it, restores what it omitted, read from the document as it stands (its own `current` read: the first draft's reuse of `latestOther` could not fire, 9.3); (3) a backup holding v2 data carries schema 3. The backend deploy (owner's VM secrets) must precede the release of any field that needs it (`needsServerCap`, 9.6) to a synced company | the server accepts a stale whole-document write as an ordinary edit (9) | yes (merge versus refuse) |
| CF8 | **Delete means hide.** A field with values is hidden; **a definition is never hard-deleted from the table: "Delete" writes `status: 'deleted'` (a tombstone that keeps its labels and id forever), is offered only on a fresh sync, and the server turns a delete from an older editor into a hide**; the type cannot change after creation; there is no force-delete in v1 | Zoho's force-delete removes data for good; Pipedrive's delete removes the data too [W]; "zero use" decided on a stale replica can be wrong (10) | no |
| CF9 | **Report ids are `cf.<key>`, `cfp.<key>`, `cfi.<key>`, `cfl.<key>`, `cfa.<key>`** (own value, party's current value, item's current value, line projection, the posting ledger's current value; AD.2.1), at most 40 characters, so `SAFE_FIELD` in `core/reportFilter.ts` accepts them unchanged | the sanitiser drops a longer id and an AND silently widens (5.1) | no |
| CF10 | A line value is a **snapshot** copied from the item when the line is entered; ghost recall does not carry line values; Alt+2 duplicate copies a field unless its "Copy when duplicated" switch is off | the GST-snapshot gotcha; "a recalled value that skips a prompt must be held" (2.4) | no |
| CF11 | Every by-field report states its basis and ends in a **Not set** row, so the amount rows plus Not set equal the signed unfiltered figure to the paisa; at line grain amounts partition exactly, counts overlap and say so, and returns carry the sign of their base in a net view (and read as the register shows them when notes stand alone); the P&L report adds a **Change in stock** line so it ties to the Profit and Loss nett. **Every report of the family (Summary, Income and expense, Bills, Balances, Stock) ties to the report it stands beside, to the paisa, for every voucher family and every source of the value (AD.2, AD.3)** | the Phase 26 and Phase 28 gotchas (5.5, 6, AD.2) | no |
| CF12 | An administrator **Fill blanks** tool sets one field on old vouchers that have none (from a fixed option or from the party's value) **with its own batch undo (a result bar, not the global Ctrl+Z), atomic per chunk, and only on a synced company that has nothing waiting to send and has just pulled** | the retroactive-tagging cost Zoho's practitioners warn about (10; R zoho-reporting-tags 3.4); a stale whole-document write must not overwrite another device's edits | yes |
| CF13 | **No plan caps.** Technical caps only (table 1.5), each naming what it protects | Zoho's per-type, per-plan caps generate complaints [R]; our plan limits are display-only (CLAUDE.md) | no |
| CF14 | A custom value **never reaches** a GST return, an e-invoice or an e-way payload, and prints only where the field's Print switch is on | Phase 20 to 21 gotcha: never print what the owner did not choose (11) | no |
| CF15 | The wave-a2 plan's Columns chord `Alt+M` is **void**: wave 3 gave `Alt+M` to Share. This spec needs no chord; the grid owner must re-pick (A, item 2) | the registry rule that a chord is spelled in one file | n/a |
| CF16 | **An older app is shown only what it can round-trip.** The written `objectKind` is the first ticked place that an older app either does not know or edits losslessly (a text or choice field on ONE master place); every other master-only shape is written `'v2only'`, which no older editor, form or count reads. `needsServerCap(def)` is true for every other shape, and in a synced company the editor **refuses** to save such a field while the pull answer lacks `udf2` | the older master forms rebuild `udf` for the fields they own, rounding a number to 2 dp, dropping `0` and `No`, flattening an array to text; a server merge can restore an omitted key but never a mutated one (1.1) | yes |
| CF17 | **Exempting a writer from the mandatory check is `{ udf: 'exempt' }`, never `system: true`.** `system` skips the operator write guard and the user stamp; the new option skips only `checkUdf`. The store derives create or alter from the stored row, throws a typed `UdfRefusal`, and every `saveVoucher` and `saveDoc` caller is classified in a table the suite enforces (3.3, 9.1) | cancel, Ctrl+Z and restore would otherwise be refused, or would bypass the date window and lose the operator stamp | no |
| CF18 | **History is bucketed by the stored value.** A voucher holding a value is reported under that value (or under "Removed option") whatever its type is scoped to today; Not asked is only for an out-of-scope voucher with no value | narrowing a scope must not move a closed year's figures (4.3, 5.5, 6.3) | no |
| CF19 | **What a mandatory field does to an OLD voucher or record when it is edited is the administrator's choice, per company, with a per-field override.** The company setting (`Company.entryPolicy`, Settings > Workspace, Administration right plus the write guard) has two knobs: vouchers and their lines (new entries only, the default / dated on or after a date / always) and masters (new entries only, the default / always). A field's `req` is `'policy'` (follow the company: the editor's default), `'new'` or `'always'` (override). Creation is always checked; the store never judges an alteration; undo, cancel, restore and every importer are unaffected | the owner's answer to question 2 of section 18 (2026-10-22); a company that changes nothing keeps the prospective rule, asserted by identity (AD.1) | yes (the three voucher modes; no warn-only tier) |
| CF20 | **The owner's examples are shapes, not features.** A scenario is (where the value lives: party, item, voucher header, item line, posting ledger) x (what is measured) x (what is done with the field: group by, filter, column, sort, sum). The report engines are built over that grid through ONE **Dimension** abstraction, not over Colour and Zone; the cells with no honest meaning and the scenarios that need a different feature are listed plainly (AD.5, AD.6) | the owner's answers to questions 1 and 3 of section 18 (2026-10-22; AD.2) | no |
| CF21 | **Summary by field serves every voucher family** (sales, purchases, receipts, payments, journals and contra, orders, delivery and receipt notes, stock vouchers) with counts and amounts; a sign convention that is net only when a base and its notes are chosen together; party-side legs for receipts and payments; pending measures for orders; quantity by unit | "Receipts by Zone" is as natural a question as "Sales by Zone" (AD.2.5) | no |
| CF22 | **Income and expense by field serves five sources of the value** (voucher header, party now, item now, line, posting ledger now). Attribution is per posting entry and never guessed (several parties, or lines that cannot be tied, get their own row). The Change in stock line is attributed exactly when the source is an item attribute and by allocation when the field tracks stock; otherwise it stays one unattributed line | "profit by X" for any X (AD.2.6) | no |
| CF23 | **Receivables and payables by a party field** (bills, ageing, on account) and **balances of any ledgers by the ledger's own field**, each tied to the group balance it stands beside, through explicit Pending / Unapplied / Not bill-wise columns for the bills (the Bills reports decide a side by sign and may list items on the other side: said in a memo, not promised equal) | "ageing by Region", "debtors by Salesman" (AD.2.7) | no |
| CF24 | **A field marked "Tracks stock" makes (item, variant) a stock position, exactly as a batch makes (item, batch) one.** It is a choice field on the item line; up to three per item (Colour and Size and one more); the position is the combination. **Costing stays per ITEM (pooled, one cost for all variants)**: a variant's value is the item's value shared by quantity, so no figure outside the Stock by field screen can ever move (**the one input that depends on how a line was entered is a Physical Stock count made by variant: a count GROUP, read from the lines' own `countPart` flag in the one movement generator every stock figure uses, so every caller reads the same item-level count with or without a resolver**, AD.3.3); per-variant costing is a later, gated slice | the owner's answer to question 1 of section 18; the smallest safe change that serves it (AD.3) | yes (pooled costing: question 18) |
| CF25 | **The variant is read, never stored as a key.** It is derived from each line's value at read time, so history before the field existed is simply Not set, switching tracking on or off rewrites nothing, and an item outside the field's scope behaves exactly as today (a Physical Stock count made by variant carries a `countPart` structure flag on its lines, which is not the key: it says "these lines are the parts of one count", AD.3.3). Opening stock by variant is an optional list on the item (`openingAlloc.variants`), trusted only while it adds up **and agrees with the godown split** | migration without a migration (AD.3.7) | no |
| CF26 | **Stock by field is one screen with two readings:** an item attribute (whole items are grouped; every figure exact; Stock Summary gains the same grouping) and a tracking variant (quantity exact, value allocated); both tie to the Stock Summary total | (AD.3.8) | no |
| CF27 | **A company-aware filter field list for the four existing filterable reports is built now (CF-4b-lite), with no edit to `reportFilter.ts`, `savedViews.ts` or `rowFilter.ts`;** the report-grid wave overwrites exactly the functions named in section 16 and keeps the ids | the owner's headline "filter the Sales Register by Zone" must not wait for the grid (16, 17) | no |
| CF28 | **A custom column sorts by what it shows** (a choice by its label, a number by value, a date by ISO date, text folded) with **Not set always last**, in either direction; its filter value stays the option id | a sorted Zone column must read alphabetically and a blank must never lead (AD.2.8) | no |
| CF29 | **Text, date and number fields can be analysed too** through banded dimensions (a distinct text value, capped at 200 with an "Others" row; a month, quarter or financial year; a number band the person types), chosen in the report's own options and never stored on the definition | "sales by month of delivery", "by credit-days band" (AD.2.2, CF-4c) | yes |
| CF30 | **Employee and lead fields reach the payroll and CRM reports later (CF-12).** Not part of this wave; said, not hidden | (AD.6) | yes |

---

## Addendum: owner decisions of 2026-10-22

> **Status:** design only; the build lanes of the first text (CF-6a, the back-navigation work) are in the tree, so where a finding below names code that already exists, the spec says what must be reworked. **Reviewed once on 2026-10-22 (the third pass; its findings are applied in this section and logged at the end)**; the two review passes of 2026-10-21 covered the text before this section. Written 2026-10-22 in the worktree `.wt-cf` (branch `cf-back-nav`, snapshot `baa0c45`) after re-reading the code each part names: `core/inventory.ts`, `valuation.ts`, `openingStock.ts`, `stockSummary.ts`, `billwise.ts`, `plFigures.ts`, `reportRows.ts`, `accounting.ts`, `policyWrite.ts`, `appearance.ts`, `state/companyUi.ts`, `state/policyMirror.ts`, `backend/src/conflict.js`, `reportFilterFields.ts`, `ui/rowFilter.ts`, `ui/settings/workspacePanels.ts`, `ui/VoucherScreen.tsx` (where `partyId` is set, the negative-stock check, the voucher literal) and `core/splitCompany.ts` (the opening-alloc carry). Tags as in the header: **[C]** read in code this pass, **[J]** judgement, **[unverified]**. Nothing was run.
> **Precedence:** where this addendum and an older passage differ, the addendum wins. The older passages it changes were edited in place so the two agree; AD.7 lists them. **Numbering:** parts AD.0 to AD.8; new decisions CF19 to CF30 (table in section 0); new slices CF-9, CF-14, CF-4b-lite, CF-4c and CF-12, and widened CF-0, CF-4a and CF-5 (section 16).

### AD.0 What the owner said, and the rule drawn from it

| Question (section 18) | The owner's answer, 2026-10-22 | What it changes |
|---|---|---|
| 1. Do you need stock by colour, or only a colour label on items and lines? | "this was example and other such scenarios shall be fulfilled" | A line field can **track stock**: (item, variant) becomes a stock position like a batch (AD.3). Colour is only the first user |
| 2. Should a mandatory field apply when someone edits an old voucher? | "there shall be an option at admin level to decide this, as per their use case" | An administrator policy with a per-field override (AD.1). The default stays "new entries only" |
| 3. Do you want profit by Zone, or only sales by Zone? | "this was example and other such scenarios shall be fulfilled" | Profit, sales, receipts, bills, balances and stock are all analysable by **any** groupable field, wherever its value lives (AD.2) |

**The rule drawn (CF20).** Colour, Zone and the "business line" tag are *shapes*. A scenario is a triple: **where the value lives** (the party, the item, the voucher header, the item line, the posting ledger) x **what is measured** (an invoice value, a quantity, a count, a P&L posting, a bill outstanding, a closing balance, a stock position) x **what is done with the field** (group by, filter, show as a column, sort, sum). Before this addendum the spec served the triples its three examples needed. It now builds the engines over the grid of triples, so that "receipts by Zone", "profit by Brand", "ageing by Salesman", "pending orders by Colour" and "stock by Size" are the same machinery with different arguments, and it says in AD.5 and AD.6 which cells have no honest meaning and which scenarios need a different feature. Mandatory-on-edit is treated the same way: a company's use case decides, not the spec.

### AD.1 Mandatory fields and old records: an administrator's choice (CF19)

#### AD.1.1 Two layers and their values

| Layer | Where | Values | Default |
|---|---|---|---|
| **Company setting** | Settings > Workspace > "Custom-field entry rules" (AD.1.5), stored on the company row (AD.1.2) | **Vouchers and their lines:** `new` (new entries only) / `since` (vouchers dated on or after a date) / `always`. **Masters** (parties, items, employees, leads): `new` / `always` | `new` and `new`: exactly the prospective rule of the first draft |
| **Per-field override** | the field editor's Required block (7.2) | `req: 'policy'` (follow the company setting; **the editor's default for a field made mandatory**), `'new'` (only new entries are checked, whatever the company says), `'always'` (checked on every alteration, whatever the company says). `req` absent = not mandatory | `'policy'` |

*Create is never a question:* a new voucher, ledger, item, employee or lead is checked for every mandatory field, in every mode. The setting decides only what an **alteration** of an existing record must do. There is no warn-only tier (question 15 stands) and no per-field date: a field that needs its own cut-off is `'always'` with Fill blanks run first.

#### AD.1.2 Where it is stored, and the older-client protection

Not inside `Company.ui`: `ui` is deliberately "not books data" (its writer's header says so) and is read by a Phase 27 sanitiser that rebuilds a known shape. An entry rule changes whether a save is refused, so it gets its own field, **with the same machinery** (the Phase 27 lesson, "company defaults live on the synced company doc, so they need older-client protection from day one").

```ts
// src/types.ts (one optional field on Company; no Dexie bump, no migration)
export type EntryEdit = 'new' | 'since' | 'always';
export interface EntryPolicyStored {
  v: 1;
  rev: number;                  // every save spends one; an all-default policy is stored as { v, rev }, NEVER as a deleted key (absence is never a reset)
  vouchers?: EntryEdit;         // existing vouchers and their lines; absent = 'new'
  since?: string;               // ISO date, read only when vouchers === 'since'
  masters?: 'new' | 'always';   // existing parties, items, employees, leads; absent = 'new'
}
// Company.entryPolicy?: EntryPolicyStored
```

| Piece | Where | Rule (each mirrors its Company.ui twin; none imports it, so the Phase 27 files stay untouched) |
|---|---|---|
| Pure core | `src/core/entryPolicy.ts` | `EntryPolicy` is the sanitised, defaults-filled reading of `EntryPolicyStored` (`{ rev, vouchers, since?, masters, extras }`). `DEFAULT_ENTRY_POLICY` (frozen; the SAME reference is returned for "nothing set", so identity is asserted by reference); `sanitiseEntryPolicy(raw)` never throws: an unknown `vouchers` or `masters` value reads as `new`, a `since` that is not an ISO date reads as absent (so `since` behaves as `new` and the panel says "choose a date"), unknown fields are **carried** (capped at 4 KB and three levels deep, `__proto__` dropped: the Phase 28 "a reader must preserve what it does not know" lesson); `serialiseEntryPolicy`; `effectiveEntryPolicy(stored, mirror)` with the same four states as `appearance.ts detectErased` (`current`, `reset`, `erased`, `older`: a row that arrives WITHOUT `entryPolicy` after this device remembered one is an older app erasing it, never a reset); `planEntryPolicyWrite(stored, mirror, update)` (nothing changed writes nothing; the new `rev` is above both the stored and the remembered one, so Restore and Reset work on an erased or older device) |
| The one writer | `src/state/entryPolicyWriter.ts` `writeEntryPolicy(update)` | modelled on `writeCompanyUi` [C]: write guard first (`writeProblem` on `companies`, Administration with security on), then `updateDoc('companies', id, stored => ...)` as a read-modify-write of the STORED row, `{ ...base, entryPolicy }`, never a rebuilt company (the vault-overwrite lesson) |
| Device mirror | `src/state/entryPolicyMirror.ts` | `localStorage` `TW_ENTRY_POLICY_MIRROR:<cid>` (cid pattern-checked, at most 20 KB, every access wrapped, the app is correct without it): the last non-empty policy this device saw, so a row that loses the field does not make the device forget; the Workspace panel offers **Restore** |
| Reading it | `useEntryPolicy()` (a hook beside the writer) | the effective policy from the store's company and the mirror; it writes the mirror when `remember` is true. The voucher screen, the master forms and Bulk Entry read it once per mount and **pass it to `checkUdf` as an argument**: the pure engine never reads storage |
| Server | `backend/src/conflict.js`, `backend/src/histories.js` | **Keyed on the capability a writer states, never on "states no base".** (The first text put `entryPolicy` in `UNVERSIONED_PROTECTED`, whose marker is wrong in both directions: every client since Phase 24 stamps a base while signed in [C `sync/baseline.ts stampBase`], so a live older build is never "unversioned" and the rule never fires for it, while a CURRENT client's rows made while signed out, before its first pull, or in another account's namespace carry no base [C `baseline.ts` header], so the rule would put the previous policy back over an administrator's legitimate edit.) **`CAP_KEPT = { companies: { entryPolicy: UDF_CAP } }`**: for a company-row `put` from a writer that does **not** state `udf2`, the server keeps the CURRENT value of `entryPolicy` **whatever base the write states**, because an app that cannot state the capability can never legitimately edit the field: a changed value from it is a stale carry or an erasure, and is restored (removed if `current` has none) and recorded as a `protected-fields` mismatch. A writer that **states** `udf2` is never touched by it: its signed-out edit, or an edit from a stale base (an ordinary recorded mismatch, as for `ui`), is applied. The value is read from **`current`** (the newest row of the document, any device, as 9.3 item 2 loads it), so `needsUdfCurrent` gains "a `put` to a table in `CAP_KEPT`": one extra indexed read per company-row push from an older writer, and the company row is pushed rarely. **`OMIT_KEPT = { companies: ['ui'] }`** fixes the Phase 27 field in the same deploy (its rule has the same flaw, and its backend half is still undeployed): for a writer without `udf2`, an incoming company row that **omits** `ui` while `current` holds one gets the current `ui` back. It is **not** keyed on the capability, on purpose: a Phase 27 or 28 build states none and legitimately edits `ui`, so a capability rule would silently revert its administrator's edit; but absence is never a reset (an empty policy is stored as `{ v, rev }`, above), so a build that omits `ui` is a build that never knew it. `UNVERSIONED_PROTECTED.companies` keeps `['ui']` as built (it still covers a pre-Phase-24 build) and does **not** list `entryPolicy`. Neither new list is in `PROTECTED_FIELDS` (a current administrator one revision behind must be able to save; that collision is an ordinary recorded mismatch). The deploy window of CF-6a carries it; `syncconflict.assert.ts` gets both lists and the cases of AD.1.6 (5); the deploy-time smoke (9.6) gets two company-row pushes, one from a device that states `udf2` and one from a device that states nothing. **The CF-6a code built from the first text (`entryPolicy` inside `UNVERSIONED_PROTECTED`, no `current` read for the company row) must be reworked to this before the backend deploy; nothing live is affected, the VM still runs the `a0fffd6` backend.** An erasure stored before the deploy is repaired by the client's Restore (above), not by the server |
| Carry | `core/companyAlter.ts`, `core/splitCompany.ts` | an alteration of the company spreads the stored row, so `entryPolicy` survives it; the company classification in `master-resave.assert` gains `entryPolicy` as "off-form, preserved"; Split Company Data is checked to copy the row's fields (a spread does; a rebuilt company would not, and the same classification catches it) |
| Backup | `core/backupMigrate.ts` | no change: a policy matters only beside v2 definitions, and those already write schema 3 (CF5, 12) |

**The fallback is the loose direction, so the net is not optional.** A policy that cannot be read falls back to "new entries only": an administrator who chose `always` would stop being enforced without a sign. That is why the rev, the mirror, `detectErased`, Restore and the capability-keyed server rule exist; until the backend deploy, the client defence alone stands, exactly as for `Company.ui` today.

#### AD.1.3 Exact semantics

`modeFor(def, place, policy)` (pure, in `core/entryPolicy.ts`) returns what an alteration of a record of that place must do:

| `def.req` | place `voucher` or `line` | place `ledger`, `stockItem`, `employee`, `lead` |
|---|---|---|
| absent | not mandatory | not mandatory |
| `'new'` | `never` | `never` |
| `'always'` | `always` | `always` |
| `'policy'` | `policy.vouchers`: `new` is `never`, `since` is `since`, `always` is `always` | `policy.masters`: `new` is `never`, `always` is `always` |

`checkUdf` then judges (this replaces the `req: 'new'` and `req: 'always'` rows of 3.2):

| Operation | mode `never` | mode `always` | mode `since` |
|---|---|---|---|
| **create** (including an F10 change into a base or type where the field newly applies, `beforeKind`) | empty is refused | empty is refused | empty is refused |
| **alter**, the record holds no value | allowed | empty is refused | empty is refused **when the voucher's stored date or its new date is on or after `policy.since`**; otherwise allowed |
| **alter**, the record held a value and it is cleared | refused ("a value cannot be removed") | refused | refused |
| **alter**, the field is filled | judged only for type, option and range, and only when the value CHANGED (3.2) | same | same |

- **`since` reads the voucher's date, for the header field and for every line field of it** (a line has no date of its own). It tests **both** the stored date and the new date, so an old voucher moved INTO the window is judged as a window voucher, and a window voucher moved OUT of it is not a way to dodge the rule. Masters have no date, so `since` does not exist for them: the master knob has only `new` and `always`, and the panel says so.
- **Optional, post-dated and memorandum vouchers** are judged like any other (an entry rule is not a posting rule). A **cancelled** shell has no editor.
- **PgUp / PgDn in an alteration:** each voucher is judged as it is opened and saved; the walk per voucher is AD.1.5.
- **The store backstop is unchanged (3.3):** it judges `create` requireds and CHANGED values only, so a binding mode is enforced by the entry screens (the voucher's `doAccept`, the master form's `validate`) and a cheque screen resaving an old party under `masters: 'always'` is not refused. That is deliberate: a policy must never make a housekeeping write fail.

#### AD.1.4 Every door, under each setting

| Door | `new` (default) | `since` | `always` |
|---|---|---|---|
| Voucher screen, Accept or Ctrl+A on a **new** entry | required fields enforced | same | same |
| Voucher screen, **alteration** (including PgUp / PgDn and Alt+2 on the way to Accept) | field not required (a value the voucher already holds cannot be removed) | required when the date rule binds | required |
| Alt+2 duplicate, "create from" (credit note from an invoice) | judged as a new entry | same | same |
| Master forms (ledger, item; employee, lead in CF-8), alteration | not required | not applicable (masters follow `masters`) | per `masters` |
| Alt+C quick create | required fields only, as 2.3 | same | same |
| **Bulk Entry** | creates only: required fields enforced at voucher-screen severity (3.3) | same | same |
| CSV Import Data, Tally XML import | creates only: history, counted not refused (3.3; the "Enforce mandatory fields" tick is unchanged) | same | same. A later **edit** of an imported voucher meets the rule above |
| REST `/v1` | not enforced in v1 (3.3). CF-6b enforces **create**; enforcing alterations there needs the server to read the company's policy from its own change log, recorded as CF-6b scope | same | same |
| **Cancel** (Alt+X), **Ctrl+Z** of an alteration, delete-undo, restore, Salary Disbursement, Manufacturing, RCM and ITC journals, numbering writes | `exempt` (3.3): never refused | same | **same: an `always` policy never stops a cancel, an undo or a restore** |
| Below-the-store writers (BRS bank date, Mark optional, GSTR-1 export info, advances, e-invoice strip) | not checked | same | same |
| **Fill blanks** | sets a value, never refused | same | same. It is the tool that makes `always` painless: the Workspace panel and the field editor offer it before the setting is saved (AD.1.5) |
| **F10** (Change Voucher Type) into a base or type where a field newly applies | judged as a create (3.2) | same | same |

#### AD.1.5 What the person sees

- **Walk and tag.** On an alteration where the field is bound, it is in the walk set W (2.1), so Enter on Narration walks into it, and its tag reads "required by company rule for existing vouchers" (dim text; the tag joins the `visual.assert` contrast matrix for both schemes and all eight accents). Where it is not bound it is in the strip, optional, as today.
- **A bound alteration never fills the cell for the person.** On a new entry the cell starts from the default or the party's value, visibly tagged (2.1). On an old voucher the party's CURRENT value may restate history, so the cell starts **blank**; when the party holds a value a visible one-click "Use North (party now)" is offered. Enter on a required blank is refused, never "picks the first row" (2.1).
- **Refusal wording:** a new entry says "Zone is required!"; an alteration says "Zone is required to edit this voucher (company rule)." and the cursor goes to the field. Nothing is saved.
- **Settings > Workspace > "Custom-field entry rules"** (a new panel, `ui/settings/panels/EntryRulesPanel.tsx`, one more entry in `WORKSPACE_PANELS`, whose `owner` literal gains `'C'`): two radio groups (Vouchers and their lines: "New entries only" / "Vouchers dated on or after [date]" / "Every voucher, always"; Masters: "New entries only" / "Always"), a plain-language summary ("Mandatory fields that follow the company setting are asked when an existing voucher dated on or after 1 April 2026 is edited"), the list of mandatory fields with where each follows the policy or overrides it (each a link to the editor), and a **consequence line**: "If you choose Always: 3 fields, 1,204 existing vouchers and 38 parties would need completing when edited. Fill blanks first." (a count of in-scope records with no value, from `usageOf`; the link opens Fill blanks). **Save (Ctrl+A)**, Cancel (Esc asks "Discard changes?", No armed), **Reset to defaults** (asks first, No armed). The date box uses `parseDateInput` with mnemonics off (the CSV lesson). The panel registers a dirty probe (it holds a draft), so Back and a rail click ask "Leave this entry?" (the back spec's rule).
- **Rights.** Anyone sees the panel; **only an administrator can change it**: `canAdminister(ctx)` (security off: everyone, as every company setting; on: the Administration right) **and** the write guard on the company row (a stale tab or a console is refused at the write by `writeEntryPolicy` and again by `putDoc`). Everyone else sees it read-only with the Workspace tab's existing note.
- **The field editor's Required block (7.2)** becomes: *Required: No / Yes.* When Yes: **"When an existing voucher or record is edited: (o) Follow the company setting (now: New entries only; change it in Settings > Workspace)   ( ) Only new entries are checked   ( ) Always checked: an old voucher must be completed to be edited."** A live count under it: "412 existing Sales vouchers have no Zone." and, for Always, the Fill blanks link.

#### AD.1.6 Proof

Suite **`entrypolicy`** (new; mutations in the same file): (1) *identity:* `effectiveEntryPolicy(undefined, null).policy` is `DEFAULT_ENTRY_POLICY` by reference; for a generated set of definitions (every type, every place, every `req`) and records, `checkUdf` with the default policy deep-equals `checkUdf` with the 3.2 prospective rules, the old `req: 'new'` meaning, on create and on alter; (2) *the semantics matrix* as a recorded vector file (about 150 rows): `req` x policy mode x place x operation x (record held a value, never held one) x (date before, on, after `since`; date moved into the window; moved out) x (field filled with a changed, an unchanged, an invalid value); (3) *every door* of AD.1.4, driven through the real callers by the source scan of 3.3, which fails on a caller in no row; cancel, undo, delete-undo and restore succeed under `always`; the store backstop does not refuse an alteration under `always` while the form check does; (4) *persistence:* hostile and garbage stored values (`vouchers: 'bogus'`, `since: 'tomorrow'`, 1 MB of extras, `__proto__`) read as the default and never throw; the rev rule; `{ v, rev }` is a stored empty policy; erased and older are detected and Restore works; `planEntryPolicyWrite` writes nothing when nothing changed; a property test runs `detectErased` and the entry-policy twin over the same synthetic histories and requires the same classification; `writeEntryPolicy` refuses a non-administrator and never rebuilds the company; (5) *server (the capability rule of AD.1.2, in `syncconflict.assert.ts` and `customfields-sync`):* `CAP_KEPT.companies` contains `entryPolicy` and `OMIT_KEPT.companies` contains `ui`; a company-row write that **states `udf2`**, with no base (signed out, before the first pull, another account's namespace), changing the policy is **applied**, and so is one from a stale base (a recorded mismatch, not a keep); the same write from a writer that states **nothing** is **kept** (the current policy comes back, recorded) with no base, with a stale base, and with a base at or past the current row; a hostile `caps` (not an array, 100 entries, a 17-character string, a non-string) reads as "states nothing", so the policy is kept; an erasure (the key absent) from a writer without `udf2` is restored; for `ui`: an older writer that omits it gets the current one back, a writer without `udf2` that carries a changed `ui` is applied (a Phase 27 administrator's edit is never reverted); `needsUdfCurrent` is true for a company-row `put` without `udf2` and false with it, and costs exactly one indexed statement. **Mutations it must kill:** read `policy.vouchers` for masters; treat `since` as `always`; test only the stored date; test only the new date; apply the policy on create; enforce an alteration in the store backstop; judge an exempt writer; auto-fill the cell on a bound alteration; treat an erased policy as a reset; a rev below the remembered one; drop unknown fields on sanitise; return a fresh object for "nothing set"; **key the server keep on "states no base"; apply the keep to a writer that states `udf2`; read `anchor` instead of `current`; apply the capability rule to `ui`.** **Recipe `cf-policy`** (real keys; light and dark, compact and comfortable, 150 % text; never Accept on a live company): as administrator set Always for vouchers, alter an old Sales voucher with no Zone (the field is in the walk, Accept is refused with the company-rule wording, "Use North (party now)" fills it); set `since` and alter one voucher on each side of the date; as a non-administrator see the panel read-only; an older-app save of the company row (mock cloud) keeps the policy once the server rule is deployed and shows Restore before it is.

*Honest limit:* the policy is enforced by the **client**, as every mandatory rule here is (REST is not enforced in v1, 3.3). An older app, or a console, can write a voucher without the field; the Mismatch Report cannot flag it, because the server holds no definitions until CF-6b. Said, not hidden.

### AD.2 Generality: one family of reports over every shape of scenario (CF20 to CF23, CF28, CF29)

#### AD.2.1 The five sources of a value (the "where the value lives" axis)

A report row (a voucher, a line, a posting entry, a bill, a ledger, an item) can reach a field's value in five ways. Each is a **source**; each has a report id prefix (CF9) and a **truth**: `posting` (the value as entered on the document, never moved by later edits of a master) or `now` (read from a master as it stands today, so a later edit restates history, and every report prints "(current)").

| Source | Prefix | The value lives on | Read from | Truth | Reads as |
|---|---|---|---|---|---|
| `own` | `cf.` | the voucher header (or the master a master list shows) | the document's own `udf` | posting | "Zone" |
| `party` | `cfp.` | the party ledger | the voucher's party, read by **one function, `partyOf(v)`** (`core/fieldDimension.ts`): for the bases whose entry screen sets `partyId` (**`baseInfo(base).invoice`: Sales, Purchase, Credit Note, Debit Note, both orders, Delivery Note, Receipt Note** [C `VoucherScreen.save`: `vPartyId` is assigned in invoice mode only]) the ledger of `partyId`; for **every other base (Receipt, Payment, Journal, Contra) and for an invoice base written in accounting mode with no `partyId`, the party-side legs (AD.2.5), even when `partyId` is set**. The Tally importer sets `partyId` on every voucher type that carries a party name [C `tallyImport.ts`: `if (partyId) v.partyId = partyId`], and `voucherHeadlineAmount` already refuses it for a non-invoice base for exactly that reason (a Payment of Dr 6,000 party + Dr 4,000 Rent read 6,000) [C `accounting.ts`], so an imported Payment and its hand-entered twin must bucket identically; for a bill its ledger | now | "Party Zone (current)" |
| `item` | `cfi.` | the stock item | the line's `stockItemId` | now | "Colour (items)" |
| `line` | `cfl.` | the item line | the line's own `udf` | posting | "Colour (lines)" |
| `acct` | `cfa.` | the ledger an entry posts to (any group), or the ledger a balance is of | `Ledger.udf` of that ledger | now | "Account type (ledger)" |

`SAFE_FIELD` still accepts every id: `cfa.` is four characters like the others (1.3). **Choosing a source is part of choosing the dimension**, and a field that lives in two places offers both: "Zone (on the invoice)" and "Zone (party, current)". **Fallback source (an option on every analysis screen):** *"The invoice's value; where blank, the party's current value"* (`['own', 'party']`). It answers the old-voucher problem without Fill blanks (the default of AD.1 leaves old vouchers blank), and the report says how many rows read the party ("412 vouchers read the party's current Zone"); Fill blanks remains the way to freeze it.

#### AD.2.2 The Dimension: the one place a field becomes a set of buckets (`src/core/fieldDimension.ts`, pure)

Every analysis engine (`fieldSummary`, `fieldPl`, `fieldBills`, `fieldBalances`, `stockByField`) takes a **`Dimension`** and never a `CustomFieldDef`, so a new kind of field is one change here and no change in five engines.

```ts
export type DimSource = 'own' | 'party' | 'item' | 'line' | 'acct';
export type DimKind = 'choice' | 'text' | 'period' | 'band';
/** What a row of any analysis report can be. One vocabulary for all of them. */
export type RowKind = 'option' | 'value' | 'removed' | 'unset' | 'notAsked' | 'noParty' | 'mixed' | 'charge' | 'other';
export interface Dimension {
  fid: string; kind: DimKind; label: string;
  sources: readonly DimSource[];           // one, or two tried in order: ['own', 'party'] is the fallback of AD.2.1
  truth: 'posting' | 'now' | 'mixed';
  declared: readonly { key: string; label: string }[];   // choice: the options in list order (hidden ones included, labelled "(hidden)"); band and period: the buckets of the window; text: empty (found in the data)
  keyOf(v: UdfValue | undefined): string | null;          // null = the document holds no value
  labelOf(key: string): string;
}
export type DimSpec = { period?: 'month' | 'quarter' | 'fy'; edges?: readonly number[] };
export function dimensionFor(def: CustomFieldDef, sources: readonly DimSource[], spec?: DimSpec): Dimension | { refused: string };
```

| Field type | Kind | Buckets | Slice |
|---|---|---|---|
| choice, yes/no | `choice` | the options (id keyed, list order), then **removed** rows for ids not in the list, then the engine's own rows | CF-4a |
| lookup to a ledger | `choice` over the target's current names | later | CF-8 |
| text | `text` | one bucket per distinct cleaned value (NFKC, case-folded, whitespace collapsed, **length capped before folding**), labelled with its first spelling, in label order; **at most 200 buckets, the rest in one `other` row "Others (n values)" whose amounts are kept**, so totals still tie | CF-4c |
| date | `period` | month (`2026-04`), quarter, or financial year by the company's first month (`dates.ts`) | CF-4c |
| number, amount, percent | `band` | `edges` (ascending, at most 12, typed in the screen's options): "below e0", "e0 to e1" (lower end inclusive, upper exclusive), "e_n and above" | CF-4c |
| multi-choice, long text, email, phone, URL | **refused** ("a record in two groups would add to more than the whole", 1.2; and an email has no honest bucket) | none | n/a |

The engine decides the **non-value rows**, in this fixed order after the value rows: `unset` ("Not set": the field applies and nothing was entered), `notAsked` ("Not asked on this voucher type": out of the field's scope and no value; a value always wins, CF18), `noParty` ("No party": the voucher or entry has no party leg), `mixed` ("Several parties", or lines that cannot be tied: counted in a footnote, never guessed), `charge` ("Whole-invoice charges": line and item sources only). Every report prints exactly the rows it has. **Never-guess rule:** where an attribution is ambiguous the amount goes to `mixed` or `unset`, because a wrong bucket is worse than a missing one (R4).

#### AD.2.3 Roles of a field: group by, filter, column, sort, measure

| Type | Group by | Filter (v1 operators, 5.3) | Column | Sort (CF28) | Measure (summed) |
|---|---|---|---|---|---|
| choice | choice | is, is not | the current label | by label | no |
| yes/no | choice (Yes, No) | is Yes, is No (an unanswered row matches neither) | Yes / No | by label | no |
| text | text bucket (CF-4c) | contains, not contains, starts with, equals | the text | folded | no |
| number | band (CF-4c) | the numeric operators | the number | by value | **yes only when `rules.additive`** (a footer of "credit days" is plausible and wrong) |
| amount | band (CF-4c) | the numeric operators | money | by value | **yes** |
| percent | band (CF-4c) | the numeric operators | the number | by value | never |
| date | period (CF-4c) | before, after, between | the date | ISO | no |
| lookup | choice (CF-8) | text on the current name | the name | folded | no |
| multi-choice | never | contains (joined labels) | joined labels | none | no |
| long text | never | contains | not a column | none | no |
| email, phone, URL | never | text | text | folded | no |

| Where the value lives | As a group-by | As a filter, column, sort | As a measure |
|---|---|---|---|
| Party (ledger) | `party` source, current value; Not asked outside the field's ledger groups | `cfp.` on every voucher-grain list and on bills; `cfa.` on ledger lists | never: a relation is blank in a total row, not 0 |
| Item (stock item) | `item` source: whole items; current value | `cfi.` on item-grain and line-grain lists | never |
| Voucher header | `own` source, the posting value | `cf.` on every voucher list | an own amount, or an own number marked additive, on voucher-grain reports |
| Item line | `line` source, the posting value; a field that **tracks stock** also splits the stock (AD.3) | `cfl.` projection on voucher lists (one value, "Multiple", or blank), a real column on line-grain lists | an additive number or amount, on line-grain reports |
| Posting ledger | `acct` source in Income and expense and in Balances | `cfa.` | never |
| Employee, lead | later (CF-12) | their own lists (CF-8) | later |

#### AD.2.4 The family: which report takes which sources

| Report (screen id) | Grain of a row | Sources it accepts | Measures | Ties to, to the paisa | Slice |
|---|---|---|---|---|---|
| **Summary by field** (`fieldSummary`) | voucher; line; party-side leg of a receipt or payment | `own`, `party`, `item`, `line`, and the fallback | per family (AD.2.5) | the register total of the same bases and period; the sum of lines; `pendingOrders` | CF-4a |
| **Income and expense by field** (`fieldPl`) | posting entry | `own`, `party`, `item`, `line`, `acct` | Income, Expense, Change in stock, Net | `plFigures` nett | CF-5 |
| **Bills by field** (`fieldBills`) | bill, and the unapplied and not-bill-wise rest of each ledger | `party` | pending by ageing bucket, unapplied, not bill-wise | the **group balance** of the side as at the same date (the Dashboard card's headline and the Balance Sheet), through the three columns Pending, Unapplied and Not bill-wise; **not** the Bills Receivable and Payable totals, which decide a side by sign (AD.2.7) | CF-4a |
| **Balances by field** (`fieldBalances`) | ledger | `acct` (a party field on a debtor is the ledger's own field) | closing balance, Dr and Cr | the closing balances of the **union** of the chosen groups' ledgers (equal to `groupBalance` when the choice does not overlap), to the paisa | CF-4a |
| **Stock by field** (`fieldStock`) | stock position | `item` (a whole-item attribute), `line` (a tracking variant) | quantity, value, movements | the Stock Summary total | CF-9 |
| Listing reports: Day Book, voucher registers, Ledger Vouchers, Bills; later the item-wise register, Parties and Items lists | row | `own`, `party`, `line` projection, `item` (as 5.4) | not a total | n/a | filter now (CF-4b-lite), column and sort with the grid (CF-4b) |
| Stock Summary | item | `item` as a group mode | the Stock Summary's own | itself | CF-9b |

The five analysis screens share one shell (`ui/reports/fieldShell.tsx`: field and source pickers, the fallback option, the basis line, Open/drill, Print and Export through the report shell), and **one writer registers all five in the new-screen checklist, once** (CF-4a; CF-5 and CF-9 add only their routes after it, never in parallel).

#### AD.2.5 Summary by field, over every voucher family (`core/fieldSummary.ts`, CF21)

This **generalises 5.5**: the rules there (rows, Not set, Not asked, header and line grain, reconciliation, drill) hold for every family below; what changes is the family list, the measures, the sign rule and the party legs.

```ts
export type FamilyId = 'sales' | 'purchases' | 'receipts' | 'payments' | 'journals' | 'orders' | 'deliveries' | 'stock';
export type Measure =
  | 'invoice' | 'taxable' | 'itemValue' | 'qty' | 'count' | 'amount'
  | 'orderedValue' | 'orderedQty' | 'pendingValue' | 'pendingQty'
  | 'qtyIn' | 'qtyOut' | 'valueIn' | 'valueOut'
  | { sum: string };                         // the fid of an additive number or amount: an own field (voucher grain) or a line field (line grain)
export type SignMode = 'net' | 'asRegister';
export interface SummaryRow { kind: RowKind; key: string | null; label: string; count: number; amount: number; qtyByUnit: [unitId: string, qty: number][]; share: number }
export interface Summary { rows: SummaryRow[]; total: { count: number; amount: number; qtyByUnit: [string, number][] }; countOverlaps: boolean; legsFromParty: number; signMode: SignMode; basis: string }
export function fieldSummary(a: { family: FamilyId; bases: VchBase[]; measure: Measure; dim: Dimension; from: string; to: string;
  vouchers: Voucher[]; ledgers: Ledger[]; groups: Group[]; stockItems: StockItem[]; cid: string; signMode: SignMode; showZero: boolean }): Summary;
```

| Family | Bases it may span (any non-empty subset) | Grains | Measures (what the rows add up) | Sign |
|---|---|---|---|---|
| Sales | Sales, Credit Note | voucher, line | Invoice value (taxes in), Taxable value, Item value before tax (line), Quantity (line, per unit), Vouchers, Sum of a field | **net** when both bases are chosen (a Credit Note is minus); a Credit Note alone is `asRegister` (positive, "Returns, as the register shows them") |
| Purchases | Purchase, Debit Note | voucher, line | as Sales | as Sales |
| Receipts | Receipt | voucher | Amount received, Vouchers, Sum of a field | none |
| Payments | Payment | voucher | Amount paid, Vouchers, Sum of a field | none |
| Journals and contra | Journal, Contra | voucher | Amount (the voucher's debits), Vouchers, Sum of a field | none |
| Orders | Sales Order or Purchase Order, **never both** | voucher, line | Ordered value, Ordered quantity, **Pending** value, **Pending** quantity, Orders | none |
| Delivery and receipt notes | Delivery Note, Receipt Note | voucher, line | Item value, Quantity, Vouchers | none |
| Stock vouchers | Stock Journal, Physical Stock | line (a Physical Stock **count group** is one movement, AD.3.3) | Quantity in, Quantity out, Value in, Value out (read from the stock engine's own observed movements, **not** from line amounts: a Physical Stock count is its variance against all earlier history, and a Stock Journal issue line with no value is priced at cost, exactly as the Stock Summary does; see the **Stock vouchers** bullet below), Vouchers | none |

- **Sign.** `baseSign` (5.5) applies **only in `net` mode**, which is entered exactly when a base and its notes are chosen together. A Credit Note or Debit Note alone, and every other family, is `asRegister`: the amounts equal the register's own figures. The basis line says which ("Net of credit notes" or "As the register shows them"). This replaces "every amount is multiplied by `baseSign`" in 5.5 and CF11's first reading.
- **Which source works at which grain.** Voucher grain accepts `own` and `party` (and the fallback). Line grain accepts all four: a header or party value is **broadcast** to every line of the voucher ("Quantity sold by Zone"), an item or line value is per line. A voucher-grain measure (Invoice value, Taxable value, Amount) with an `item` or `line` source is **refused** ("Colour belongs to lines: choose Item value or Quantity"), except **Vouchers (count)**, which then reads "vouchers having a line of this option" and overlaps (5.5).
- **Party legs, for every voucher the `party` source reads by legs (AD.2.1 `partyOf`).** `partyId` is set by the voucher screen only for the invoice bases in item-invoice mode [C], and the Tally importer also sets it on a Receipt or a Payment [C `tallyImport.ts`]; the `party` source therefore **honours `partyId` only for `baseInfo(base).invoice` bases**, and a Receipt, Payment, Journal, Contra and an invoice base written in accounting mode (a Sales voucher with no party and no items) read legs **whether or not `partyId` is set**. For them the party source reads **legs**: the entries on ledgers under Sundry Debtors or Sundry Creditors (the `gid(cid, 'debtors')` and `gid(cid, 'creditors')` subtrees, as `subtreeIds` finds them [C `data/defaults.ts`]) **on the base's party side: debit for Sales, Debit Note and Payment, credit for Purchase, Credit Note and Receipt** (the side `voucherHeadlineAmount` already uses for the invoice bases [C `accounting.ts`]). Each leg takes its own ledger's value and its own amount; the rest of the voucher's amount (`registerAmount` less the legs) goes to **No party**, so the rows still sum to the register (a Receipt of 10,000 with a 3,000 customer leg and a 7,000 cash sale is 3,000 under the customer's bucket and 7,000 under No party). The remainder is shown as it falls, **never clipped**: for a Receipt or a Payment it cannot be negative (the legs are part of the voucher's credit or debit total and a balanced voucher's two totals are equal), and for an accounting-mode invoice a negative remainder, if `invoiceValue` is not its party legs, is shown in No party so the rows still tie. A voucher with legs in two buckets is counted under each (`countOverlaps`). Journal and contra refuse the `party` source (a journal between two parties has two sides to net: that is what **Balances by field** is for). **Fixture:** an imported Payment (`partyId` = the supplier, Zone North: Dr Supplier 6,000, Dr Rent 4,000, Cr Bank 10,000) and the same Payment typed on the voucher screen (no `partyId`) bucket **identically**, 6,000 under North and 4,000 under No party, never 10,000 under North.
- **Orders.** Ordered measures sum the order lines (orders never move stock). Pending measures come from `pendingOrders(base, vouchers, to, variants?)` (AD.3.3: the fourth argument is passed by every caller, and for a **tracked** item the fulfilment is allocated per order line, blank variants included), whose row gains `line` (the order line's index) and `variantKey` so a pending line has a bucket; the pending value is `pendingQty x rate`.
- **Stock vouchers tie to the stock engine, not to line amounts.** A Physical Stock line carries the COUNTED quantity, so its measure is a variance that depends on every earlier movement and not only on the window, and a Stock Journal issue line entered with no value is priced at cost by the Stock Summary (`outwardValue`, `stockSummary.ts`) while `line.amount` is 0. The family therefore reads the same stream the Stock Summary reads: **quantities** are the observed movement quantities of `stockPositions`' `observe` callback (equal to `resolvedMovements`, which resolves a count the same way; with a tracking field a count group's per-variant deltas come from `countDeltas`, AD.3.3); **Value in** is the movement's `inValue`; **Value out** is what the Stock Summary prints (a line's own value, or the cost the engine issued it at, `-dValue`, or for a period-average method that settles at close the item's issue rate). It uses `stockFlow` with two additive options: `FlowOpts.bases` (only the chosen bases reach the flow cells; every base still books the valuation) and `FlowOpts.cellOf?: (m: StockMove) => string | null` (a bucket function for a **non-tracking** line field or an own or party field, built on the godown cells' precedent: one `apply`, one `snap`), so a valueless issue line reads at cost here exactly as there. **For a tracking field the family offers the quantity measures only** (Quantity in, Quantity out, from the per-variant deltas of AD.3.3): Value in and Value out by variant need a per-variant cost, which pooled costing does not have (AD.3.1, AD.3.5). The window needs a **full-history walk** up to `to` (a count's variance needs the running balance): its cost is a perf row, `cf.summary.stock`, at most 800 ms at 250,000 vouchers [J], reference not gated. **The tie** is: with every stock-moving base chosen the quantities and values equal the Stock Summary's Inwards and Outwards for the same items and window (quantities exactly, values to the paisa); with a base filter they equal the same stream summed over those bases (asserted at 2,000 and 50,000 vouchers, with a Physical Stock count, a valueless Stock Journal issue line and a period-average item in the fixtures).
- **Quantity** is shown **per unit** ("120 Nos, 30 Kg"): a quantity summed across units is meaningless, so a row's `qtyByUnit` is a list and a Total row is the same list (the Stock Summary leaves the cell blank when units differ; here the units are listed, which costs nothing).
- **Sum of a field** is allowed only for an own amount or an own additive number (voucher grain) and a line amount or additive number (line grain); the Total row sums it, a relation column never does (5.2).
- **Reconciliation (CF11), per measure, proved for every family and source** on the sample company and a 50,000-voucher synthetic company: voucher measures: the rows (value rows, removed, Not set, Not asked, No party) equal the **register total** of the same bases and period (signed in `net` mode, unsigned otherwise); line measures of the invoice and note families: the rows equal the signed sum of the lines in scope (the **Stock vouchers** family is the exception: it ties to the stock engine's observed movements, the bullet above); pending measures: the rows equal `pendingOrders`' total; a count at voucher grain sums to the vouchers, a count at line grain or with legs has a Total of **distinct** vouchers and says "a voucher with several options is counted under each". Optional and cancelled vouchers are excluded by the register guard with and without a filter.
- **Drill** is 5.5's, extended: a voucher-grain row opens the family's register flat level with a live filter `cf.<k>` or `cfp.<k>` (both exist after CF-4b-lite for Day Book and registers); a line-grain row opens the item-wise register (disabled until it exists); **Not set** is a disabled row with a tooltip until `isEmpty` ships.

#### AD.2.6 Income and expense by field, over five sources (`core/fieldAttribution.ts`, `core/fieldPl.ts`, CF22)

This **generalises 6.3 and 6.4**. R1 to R6 stand (R4's machinery is reused). The new rules:

| # | Rule |
|---|---|
| R7 | **Party source.** Each counted P&L entry of the voucher takes the bucket of the voucher's party, by the one function `partyOf` (AD.2.1). **Invoice-base vouchers** (`baseInfo(base).invoice`: Sales, Purchase, Credit Note, Debit Note, the orders and the notes) **with a `partyId`**: `partyId`'s ledger value, now. **Every other voucher, a Receipt, Payment, Journal or Contra with or without a `partyId`** (the Tally importer sets it on every type), and an invoice base with none: the **distinct** ledgers under Sundry Debtors or Creditors among the entries; exactly one takes its value, none is **No party**, several is **Several parties** (`mixed`, counted in the footnote, never guessed). The report heading says "current". The imported and the hand-entered twin of one Payment bucket identically (fixture) |
| R8 | **Item source.** R4 with the line's bucket read from the item master now (`item.udf`) instead of the line's own bag: the same item-entry rule (index 1, or the one unpointed candidate), the same largest-remainder split to the paisa, the same `charge` row and the same "untied" footnote |
| R9 | **Account source.** The bucket of an entry is its own ledger's value now (`Ledger.udf`): a P&L ledger with no value is `unset`, outside the field's ledger groups (`scope.groups`) with no value is `notAsked`. "Fixed against variable cost" is a ledger field and this source |
| R10 | **Change in stock.** `plStock` gives one opening and one closing figure. **Item source:** attributed **exactly**: each item's opening and closing value (`stockDeltaByItem`, new in `inventory.ts`, and `stockDelta` is re-expressed as its sum so there is one computation, asserted equal) goes to the item's attribute bucket, the residual of at most a few paise to the largest item, so the buckets add to the total to the paisa; "Profit by Brand" then carries a real stock movement and is a true gross figure. **`stockDeltaByItem` reads the opening the way `stockDelta` does, not the way `stockPositions` does:** when `from <= booksFrom` an item's opening is the raw `openingValue || openingQty x openingRate` of `stockDelta`'s own branch (the total is `round2` of their sum), never a `stockPositions` read, whose opening state is settled to qty x a rate truncated to five places and is valued at 0 under Zero cost (`valuation.ts newValState`), so a P&L whose opening came from the position would differ from the Profit and Loss by paise or more; when `from > booksFrom` it is `stockPositions` at the day before; the closing is `stockPositions` at `to`. `stockDelta` is asserted bit-equal to its old output on every sample, on a company with openings, one with Zero-cost items and one whose opening rates truncate. **Tracking line field (AD.3):** attributed **by allocation** (the pooled item value shared by variant quantity, AD.3.5) and labelled "allocated"; an item whose allocation is **Unallocated** (a negative variant, AD.3.5) puts its stock change in an "Unallocated" bucket, so the buckets still add to the total. **Every other source:** one line "Change in stock (not attributed)", as 6.4 says today. The `stock` column of a row is `null` (drawn "n/a") for a source that cannot attribute it |

```ts
export type PlSource = DimSource;                  // own, party, item, line, acct
export interface FieldPlRow { kind: RowKind; key: string | null; label: string; income: number; expense: number; stock: number | null; net: number }   // net = income - expense + (stock ?? 0)
export interface FieldPl {
  rows: FieldPlRow[];
  total: { income: number; expense: number; stock: number; net: number };
  stockChange: { total: number; attributed: number; unattributed: number };   // unattributed is drawn as its own line
  nett: number;                                    // sum of rows' net + stockChange.unattributed
  plNett: number;                                  // plFigures(...).nett for the same window
  diff: number;                                    // plNett - nett: computed and SHOWN, must be 0.00 ("Agrees with the Profit and Loss statement")
  footnotes: { mixedVouchers: number; untiedVouchers: number };
  /** Entries on Income- or Expenses-NATURE ledgers that lie outside the six reserved statement groups (a user-created primary group of nature Expenses): NOT in `nett` and NOT in `diff`; shown as the memo row "Outside the statement: X (not in nett)" */
  outside: { income: number; expense: number; entries: number };
}
```
Rules carried from 6.4: basis printed ("postings in the window, opening balances excluded, accrual"); **what counts is decided by the SAME six reserved subtrees `plFigures` reads** (`moveFromTotals` over `ledgersUnder(gid(cid, slug))`: sales, directinc, indirectinc = income, purchase, directexp, indirectexp = expense), **not by a ledger's group nature**, because the tie target is `plFigures`' nett (Split Company's nature test is a different rule and is not borrowed): an Income- or Expenses-nature ledger outside those subtrees is **not in the Profit and Loss nett, so it is not in this report's nett either**, and it is shown as the memo row above, never added to a bucket and never hidden; the suite asserts the tie (`diff` 0.00) on a company that HAS such a group; **margin by option is still not offered** (`profitability.tsx` costs at current average cost), but an item-source gross figure with the exact stock movement is. **Proof:** the tie to `plFigures` nett for each source on a company with a loss, with a stock change, without stock items and on the 50,000-voucher synthetic company; for a header field, a line field and a two-type scope; for the fallback source; fixtures for R7 (a journal with two parties), R8 (an unpointed Tally-imported voucher), R9 (a ledger outside the field's groups) and R10 (the item buckets add to the stock change to the paisa under all eight costing methods).

#### AD.2.7 Receivables and payables by a party field; balances by a ledger field (`core/fieldBills.ts`, CF23)

**Bills by field.** Input: `outstandingBills(ledgers, vouchers, asOn)` for the bills, and `closingBalances(ledgers, vouchers, asOn)` for what each ledger holds (the two engines the Bills reports and the Group Summary already call). The screen has a Receivable / Payable switch (an F12 enum, no chord, **B6**) and covers the ledgers under the side's own group (the `debtors` or the `creditors` subtree; another group's bill-wise ledger is not on this screen, as it is not on the Dashboard card). **The honest tie target is the group balance of the side, not the Bills reports' total.** The Bills Receivable and Payable reports decide a bill's side by the **sign** of what is left and list on-account rows [C `ui/reports/outstanding.tsx`: `side === 'receivables' ? b.balance > 0 : b.balance < 0`, and the grand total sums `|balance|` of every row listed, on-account rows included], so a customer's receipt on account (a credit on a debtor) is listed under Payable and a supplier's advance (a debit on a creditor) under Receivable; `ui/dash/widgets/compute/receivables.ts` documents the same disagreement and CLAUDE.md lists it as open (the Bills reports are Phase 26's and are not touched here). A screen built to the group rule cannot show `diff` 0.00 against that report except on books with none of those entries, so the first text's promise ("`diff` ... the Bills report's own total for the side: 0.00") is withdrawn and replaced by an explicit reconciling set of columns:

| Column | What it holds, for the ledgers of one bucket (side-signed: a receivable ledger's Dr balance and a payable ledger's Cr balance are positive) |
|---|---|
| Ledgers | the count of the group's ledgers in the bucket |
| **Pending** (Not due, 0-30, 30-60, 60-90, Above 90; Bills = the count) | every `outstandingBills` entry of the group's bill-wise ledgers, **a named bill or the on-account entry**, whose side-signed balance is above zero (the Dashboard card's own `owedC > 0` rule, so a sale posted to a bill-wise debtor with no bill reference is a pending item aged from its date, exactly as the card and the Bills report both show it), aged against `ageClock = ageingAsOn(asOn, today)`, never the future (**B3**) |
| **Unapplied** (signed) | for a bill-wise ledger: its side-signed closing balance less its Pending, which is what has the OTHER sign or no bill: **an on-account receipt on a debtor and an advance on a creditor (B4), an overpaid named bill (a New Ref with more settled than billed), and an opening balance not itemised by bills**; the Dashboard card sends the same entries to its Unapplied (`owedC <= 0`). Negative when more has been received on account than is billed |
| **Not bill-wise** (signed) | for a ledger that is **not** bill-wise (`led.billwise` false: it is in no `outstandingBills` list): its whole side-signed closing balance |
| **Balance** | Pending + Unapplied + Not bill-wise, which is the bucket's side-signed closing balance |

**Rules.** **B1** a bill's side is its ledger's **group**, never the sign of what is left; **B2** the bucket is the ledger's value now (the `party` source; a bill carries no voucher id, so a voucher field joins when the originating-voucher index lands, 4.3); **B3** ageing against `ageingAsOn(asOn, today)`; **B4** an entry of the OTHER sign than its group's side (a customer's on-account receipt, a supplier's advance, an overpaid bill) is inside Unapplied, never Pending, so no customer becomes a payable; a same-sign on-account entry is Pending, as in the card and the reports; **B5** a Not set row always exists; **B6** the Receivable / Payable switch.

```ts
export interface BillsRow { kind: RowKind; key: string | null; label: string; ledgers: number; bills: number; notDue: number; d0_30: number; d30_60: number; d60_90: number; d90p: number; pending: number; unapplied: number; notBillwise: number; balance: number }
export interface BillsByField {
  side: 'debtor' | 'creditor'; asOn: string; ageAsOn: string; rows: BillsRow[]; total: Omit<BillsRow, 'kind' | 'key' | 'label'>;
  groupBalance: number;   // side-signed `groupBalance` of the side's group from `closingBalances` at asOn: the oracle, read independently of the rows
  diff: number;           // groupBalance - total.balance: computed and SHOWN, must be 0.00 ("Agrees with the group balance")
  /** The memo: entries of THESE ledgers that the Bills report (which decides by sign) lists on the other side, counted by this engine's own sign test; no equality with the report is claimed */
  otherSideInBillsReport: { onAccount: { entries: number; amount: number }; overpaid: { bills: number; amount: number } };
}
```
**The memo line** under the table says it in words: "Bills Receivable and Payable decide a side by sign, so they list N on-account or advance entries (X) and M overpaid bills (Y) of these ledgers on the other side; this report keeps every ledger on its group's side and agrees with the group balance." **Proof:** (a) `diff` is 0.00 for both sides on the sample company and a 50,000-voucher synthetic company, with fixtures: **a debtor with an on-account receipt, a creditor with an advance, a non-bill-wise debtor, an overpaid New Ref on a debtor, a bill-wise ledger whose opening balance is not itemised, a bill-wise ledger in another group (not on the screen)**; (b) the Dashboard's Receivables or Payables card (`computeAr`, the suite builds the env as `dashboard-v2.assert` does) on the same `asOn`: `total.pending` equals the card's `current + overdue`, and `unapplied + notBillwise` equals the card's Unapplied, to the paisa; (c) **the one case where equality with the Bills report is promised**: every bill-wise ledger sits under the side's own group and none holds an entry of the opposite sign (no on-account receipt on a debtor, no advance on a creditor, no overpaid bill); then `total.pending` equals the report's grand total for that side, computed by the report's own rule (a test oracle written from the sign rule, not the report's component); (d) the memo's counts equal a by-sign count over `outstandingBills` on the fixtures above.
Drill: a Pending cell opens Bills Receivable or Payable with a live filter `cfp.<k> is <id>` (the bills family gets `cfp.` ids in CF-4b-lite; the report may list more rows than the cell, by its sign rule, and says "filtered"); an Unapplied or Not bill-wise cell, and Enter on a row, expand the row in place to its ledgers (Pending, Unapplied, Not bill-wise each), and Enter on a ledger opens Ledger Vouchers.

**Balances by field.** Closing balances of the ledgers under chosen groups (default: Sundry Debtors and Creditors) as at a date, bucketed by the ledger's **own** field (`acct` source), with ledgers count, Dr, Cr and net. It covers what Bills cannot: ledgers that are not bill-wise, any other group ("bank balances by bank type"), and the net party movement a journal makes. **Overlapping choices are one set:** the chosen groups may nest or overlap (Current Assets and Sundry Debtors; a sub-group chosen with its parent), and `ledgersUnder` per group would count a debtor twice (`groupBalance` sums one group), so the engine takes the **union of the ledger ids** before it buckets, and a ledger counts once however many chosen groups contain it. **Ties:** the rows equal the closing balances of that union to the paisa; when the choice does not overlap this is `groupBalance` over the same groups, and the suite asserts both (the oracle for an overlapping choice is the sum of `bal.get(id)` over the union). A row expands in place to its ledgers; Enter on a ledger opens Ledger Vouchers. This is **not a Balance Sheet by field** (6.7 stands: a header tag does not reach every leg); a ledger's own field reaches the ledger by definition. **Fixture:** Current Assets and Sundry Debtors both ticked: no debtor is counted twice.

#### AD.2.8 The same field as a column, a filter and a sort in the listing reports (CF27, CF28)

- **`CfColumn` (5.2) gains `sortKey(row, ctx)` and the `acct` role.** A choice sorts by its **label** (what the cell shows), a number by value, a date by ISO date, text folded; **a row with no value sorts last in both directions** (a blank must never lead the list); the filter value stays the option id (`get`).
- **Which listing report offers which source** is 5.4, with two additions: Bills (and later the Parties list) offer `cfa.`/`cfp.` of the ledger, and the Stock Summary offers `cfi.` as a group mode (AD.3.8).
- **Filter now, column and sort with the grid.** The four existing filterable reports (Day Book, the voucher registers, Ledger Vouchers, Bills) get a company-aware filter field list **now** in slice **CF-4b-lite**; columns and sort need the grid's table and wait for CF-4b. Exact file contract is in section 16 ("What CF-4b-lite adds, and what the grid wave overwrites").

### AD.3 Stock by field: an item attribute, and a line-level variant that tracks stock (CF24 to CF26)

#### AD.3.1 Two readings, and the one decision that keeps every other figure safe

1. **An item attribute** ("Brand", "Material", "Season": every item has one value). Whole items are grouped, every figure is already per item, so **nothing in the engines changes**: the value is exact. It shows as a Stock Summary group mode and as a reading of the Stock by field screen (AD.3.8).
2. **A line-level variant** ("Colour", "Size": one item is sold, bought and counted in several). The stock itself must split: a position per (item, variant), the way the engine already keeps a position per (item, batch) and per (item, godown) [C `inventory.ts`: `byGodown`, `byBatch`].

**Decision (CF24): costing stays per item.** Valuation is a per-item state machine (`valuation.ts`: one pool, one set of lots, one average; lots are per item "so a transfer re-queues its lot", `inventory.ts`); a variant does not get its own. A variant's quantity is exact; its value is **the item's value shared by quantity** (AD.3.5). The reasons are the safest available: no figure outside the Stock by field screen can ever move (the Balance Sheet, the Profit and Loss, the Stock Summary, the valuation reports and every Dashboard card call `stockPositions` without a resolver, 23 call sites by grep, and read the same item pools), the eight costing methods need no variant-aware twin, and switching tracking on or off is reversible. **That sentence is true only if no item-level figure depends on whether the caller passed a resolver, and one rule of the first text broke it** (found by review, reproduced in `stockPositions`: a Physical Stock line's `cur` feeds `bookIn` and `bookOut`, so resolving a tracked item's count against the (godown, variant) balance changed the item's qty, value, `avgCost` and `byGodown` for every caller that passed a resolver and for none that did). **The fix is a rule, not a patch: a Physical Stock count is always a count of the WHOLE item at its godown, and a count made by variant is one count GROUP whose total is read from the lines' own `countPart` flag in the single movement generator all callers share (AD.3.3, "The count group").** The group is therefore the same for every caller, with or without a resolver, and the resolver only adds `byVariant`. This is the one place where how a line was entered (a flag on it) moves an item-level figure, and it moves it for every reader alike; it is said here, tested (AD.3.9) and listed in the release note. **What it costs, said plainly:** where cost really differs by variant (a jeweller's 22 K against 18 K, a fabric that costs more in one colour) the allocated variant value is an approximation; the screen's header says so, and the two honest answers are one item per variant (which works today) or the gated per-variant costing slice (CF-9e, question 18).

#### AD.3.2 The tracking field (a definition rule, not a new object)

`CustomFieldDef.tracksStock?: boolean` (a v2 property, 1.3). Rules the editor and `normaliseDef` enforce:

- type **`select`** only (a bounded, id-stable list), and the **Item line** is one of its places (`also` holds `'line'`); the Stock item place is optional and gives "the item's usual variant" through `inherit.item`;
- it applies to the items in `scope.stockGroups` (sub-groups included; absent = every item), and **the cell is drawn only for those items** (a service or an item in another group books as today);
- it is scoped to **every base that moves or counts stock**: Sales, Purchase, Credit Note, Debit Note, Delivery Note, Receipt Note, Stock Journal, Physical Stock. The editor ticks and **locks** them (Sales Order and Purchase Order are optional, on by default, because they never move stock). A tracking field asked on Sales but not on Purchase would book purchases to Not set and sales to Red, and the position would drift;
- **at most three** tracking fields apply to any one item (the editor refuses a fourth that overlaps an item's scope); the position is the **combination**, in the fields' `order` (Colour then Size);
- `req` defaults to `'policy'` (AD.1) with the editor's sentence "A blank books the stock as Not set";
- **Hide is refused while the field tracks stock.** A hidden tracking field is not drawn, so every later line would book Not set: the first text allowed it and so let an administrator who only wanted to be asked less turn off the variant on every sale after (the drift of AD.3.4b: Red and Blue never fall, Not set goes negative). The editor offers **Stop tracking** instead (clear the flag: the stock returns to whole items, nothing is rewritten, and the field keeps working as an ordinary line label), and **Hide and Delete are offered only after it** (a Delete is the tombstone of 10, with its own rules). The resolver still reads a `hidden` tracking field as tracking, so a row written by anything else (an import, a sync from a build that allowed it) is read the way its lines were entered;
- **Copy when duplicated is on and locked** for a tracking field: a credit note made from an invoice, or an Alt+2 duplicate, must return the shirt to the variant it left from; a copy that dropped the variant would book Not set (AD.3.4b);
- **Stop tracking** (and a `deleted` tombstone) returns the screen to the item view; nothing is rewritten; no figure outside Stock by field was ever different, because every item-level figure is resolver-independent (AD.3.1, the count group).

```ts
/** The option ids of the item's tracking fields, in field order, joined by U+0001; '' is a segment with no value; all segments '' = Not set.
 *  Ids carry no control characters (1.4), so the separator is safe. Derived at read time from the line's udf, NEVER stored (CF25). */
export type VariantKey = string;
/** null = this item is not tracked: the engine does nothing for it. */
export type VariantOf = (item: StockItem, line: InventoryLine) => VariantKey | null;
export interface VariantPos { qty: number; byGodown: Map<string, number> }
```
`variantResolver(defs, stockGroups): VariantOf | undefined` (in the new `core/stockByField.ts`) returns **`undefined` when no tracking field exists** (the identity reference; a field counts while its status is `active` or `hidden`, a `deleted` tombstone does not), else a function memoised per item.

#### AD.3.3 The minimal safe engine change (lane I, slice CF-9a)

Nothing below touches `bookIn` or `bookOut`, and **no item-level figure depends on whether the caller passed a resolver**: the variant map is updated **beside** `byGodown` and `byBatch`, after the valuation call, so qty, value, `avgCost`, `byGodown` and `byBatch` of every item are what they would have been, and the one rule that reads how a line was entered (the count group, below) lives in the movement generator that every caller shares, not in the resolver. (The first text resolved a tracked item's Physical Stock count against the (godown, variant) balance inside `stockPositions`. That made `cur`, which feeds `bookIn` and `bookOut` [C `inventory.ts`], depend on the resolver, so the item's qty, value, `avgCost` and `byGodown` changed for a caller that passed a resolver (Stock by field) and for none of the 23 existing call sites, which pass none, so the same item read two different quantities on two screens. It is withdrawn.)

| Function (file) | Change | With no resolver, or no `countPart` line |
|---|---|---|
| `ValuationOpts` (`inventory.ts`) | adds `variantOf?: VariantOf` | absent: every branch below that reads it is skipped |
| `InventoryLine` (`types.ts`) | one optional member, `countPart?: true`: "this Physical Stock line is one part of a count of the whole item at its godown" (the count group below). Written only by the voucher screen, only for a tracked item on a Physical Stock voucher; a structure flag, **not the variant key** (CF25), classified by `voucher-resave` (9.1) and carried by the 9.3 server carry | absent on every line written before it exists; ignored on any voucher but Physical Stock |
| **`stockMovements` (`inventory.ts`)**, the one generator behind `stockPositions`, `resolvedMovements`, `stockItemRows`, `stockFlow`'s observe stream and `ui/inventory/ageing.tsx` | on a **Physical Stock voucher**, the `countPart` lines that name the same item and the same godown become **ONE movement**: `line = { ...first part (without its udf and countPart), qty: the sum of the parts' qty, rate: the first part's positive rate or 0, amount: round2(qty x rate) }` and `parts: InventoryLine[]` (the original lines, voucher order); the other part lines yield nothing; the merged movement sits where the first part sat. Every other line is yielded exactly as before (the same `line` object) | **deep-equal output for every voucher with no `countPart` line** (asserted on the sample company and a 50,000-voucher synthetic one) |
| `StockPosition` / `ValState` (`inventory.ts`, `valuation.ts`) | one optional field, `byVariant?: Map<VariantKey, VariantPos>`, present on a tracked item **from the start** (seeded Not set at the opening, or by the opening list, so the invariant below holds before the first movement) | absent on every item |
| `stockPositions` (`inventory.ts`) | beside the `byGodown` and `byBatch` updates, a tracked item updates `byVariant` after the valuation call: an ordinary line moves its own variant by its quantity; **a Physical Stock movement books its ITEM-level delta exactly as it does today (against `byGodown`, never against a variant balance) and then spreads that same delta over the variants with `countDeltas`**. It tells `observe` two additive things about a movement: `StockMove.parts?` (the count group's original lines) and, for a tracked item, `StockMove.variants?: ReadonlyArray<{ key: VariantKey; dQty: number }>` (the signed per-variant change: one entry for an ordinary line, the `countDeltas` entries for a count) | byte-identical output (deep-equal asserted), and **with a real resolver every existing field of every position is identical, Physical Stock included** |
| `openingStock.ts` | `OpeningStockAlloc.variants?`, `openingVariantMap(it)` (a fresh Map per call; trusted only under the sum rule **and the per-godown rule** of AD.3.7), and `applyOpeningAlloc` seeds `byVariant` when asked | untouched for an item without the list |
| `stockItemRows` (`inventory.ts`) | optional `variant?: { of: VariantOf; item: StockItem; key: VariantKey }` (the caller passes the item, which the function does not hold today): only that variant's movements; a Physical Stock row of a variant is that variant's `countDeltas` entry | unchanged |
| `resolvedMovements` (`inventory.ts`) | optional `variantOf`: each row carries `variantKey`; a Physical Stock count emits one row per variant it moves (from `countDeltas`), amounts as today | unchanged |
| `ui/inventory/ageing.tsx` | **no change of logic**: it keeps its own running godown balance and its own lots, and reads `stockMovements`, so a count group is one movement there too (the first text did not mention this file at all, though it is the fourth place that resolves a count). Its lot walk lives inside the component's `useMemo`, where no suite can drive it, so the builder **moves that walk, unchanged, into a pure `core/stockAgeing.ts`** that the component calls; a golden of the sample company's rows before and after proves the move, and the vectors above read Stock Ageing through it | unchanged |
| `pendingOrders` (`inventory.ts`) | optional fourth parameter `variants?: { of: VariantOf; items: ReadonlyMap<string, StockItem> }` (it takes no items today); **a tracked item's fulfilment is allocated per order line (below)**; the row gains `line` and `variantKey` | unchanged, including today's quirk that two order lines of ONE untracked item each subtract the whole fulfilled quantity (recorded, **not fixed here**: nothing changes for a company with no tracking field) |
| `stockFlow` (`stockSummary.ts`) | `FlowOpts.variants?: VariantOf`, `ItemFlow.byVariant?: Map<VariantKey, VariantFlow>`, where `VariantFlow = { opening: { qty, value: number \| null }, inward: { qty }, outward: { qty }, closing: { qty, value: number \| null } }`: **quantities** from the per-variant deltas the engine observed (`StockMove.variants`, one entry for an ordinary line, the `countDeltas` entries for a count), **Opening and Closing values from the allocation of AD.3.5 (`null` when Unallocated), no Inwards or Outwards value and no consumption at variant level** (pooled costing has no per-variant cost to price a movement with, and a count that raises one variant and lowers another has no honest gross value); `FlowOpts.bases` and `FlowOpts.cellOf` are the additive options of AD.2.5 | unchanged |
| `stockDeltaByItem` (new, `inventory.ts`) | per-item opening and closing value for a P&L window; `stockDelta` is re-expressed as its sum, so there is one computation (asserted equal on every sample), **with the opening read the way `stockDelta` reads it** (AD.2.6, R10) | `stockDelta` identical |
| `core/stockByField.ts` (new, pure) | `variantResolver`, **`countDeltas`**, `allocateValue`, `stockByVariant`, `stockByItemAttribute`, key projection (a segment of a combination), the size-run grid | new file |

**The count group (Physical Stock; the rule the first text got wrong).**

- A Physical Stock line carries the COUNTED quantity and each engine resolves it to a delta against the godown balance (the same arithmetic is written in `stockPositions`, `resolvedMovements`, `stockItemRows` and `ui/inventory/ageing.tsx`). That arithmetic is **not changed and is never given a variant**: a count is **always a count of the WHOLE item at its godown**.
- A line with **no `countPart`** is exactly today's line: the item at that godown becomes the counted quantity. For a tracked item the variant effect of its delta goes to the variant the line names (**Not set** when it names none, because the line did not say which variant; this can drive Not set negative, and the Stock by field header says so, AD.3.4b).
- A count made **by variant** is one **count group**: the voucher screen saves the lines of one (item, godown) with `countPart: true` and `stockMovements` merges them into one movement of their sum, so every reader, with or without a resolver, sees one whole-item count of that total. **Variants held at that godown and not listed count 0**; the entry screen pre-fills a row for every variant held at the godown with its book quantity at the voucher's date (so an untouched row moves nothing, and only a deliberate change moves a variant), and a row deleted by hand counts 0 (the screen says so before Accept). A group is **either part lines or one whole-item line** for an (item, godown): the screen refuses a mix; a voucher that mixes them anyway (an import) merges only its part lines and reads the whole line as it would alone, in voucher order.
- **`countDeltas`** is the one place the variant logic of a count lives (`stockPositions`, `resolvedMovements` and the variant path of `stockItemRows` all call it):

```ts
/** The per-variant deltas of ONE Physical Stock movement of a tracked item at one godown. Pure. Their sum equals `itemDelta` EXACTLY. */
export function countDeltas(
  held: ReadonlyMap<VariantKey, number>,                 // this godown's variant balances before the count
  itemDelta: number,                                     // the item-level delta the engine already resolved (round2 of counted less byGodown): never recomputed here
  parts: ReadonlyArray<{ key: VariantKey; qty: number }> | null,   // the movement's parts (keys read through the resolver); null for a whole-item line
  wholeKey: VariantKey,                                  // the variant a whole-item line names (the Not set key when blank)
): Map<VariantKey, number>;
```
  With `parts`: each listed key (summed if it repeats) goes to its counted quantity, delta `counted - held`; every other key held at this godown with a non-zero balance goes to 0, delta `-held`; the difference between the sum of these and `itemDelta` (at most the rounding of the engine's own `round2`) goes to the listed variant of the largest absolute delta (the first in list order on a tie), so the sum is `itemDelta` to the last bit. Without `parts`: the whole `itemDelta` goes to `wholeKey`.
- **The vectors that pin it** (Red 10 and Blue 10 held at Main; the item is 20): (1) parts Red 8 and Blue 10 give item 18 (Red 8, Blue 10) in the Stock Summary, `stockValueAsOn` (the Balance Sheet and the Profit and Loss), Stock Ageing, Godown-wise and Stock by field, **the same number in every one of them**; (2) parts Red 8 alone give item 8, Red 8, Blue 0 (the screen would have pre-filled Blue 10); (3) a **blank-variant whole-item count of 20** moves nothing at item level and nothing in `byVariant` (the first text read it as "counts Not set only": Not set held 0, so delta +20 and item 40 with a resolver, 20 without one, and a double count of every shelf); (4) a blank whole-item count of 8 gives item 8 and Red 10, Blue 10, Not set -12; (5) the same four with and without a resolver give identical `qty`, `value`, `avgCost`, `byGodown` and `byBatch`.
- **Older readers.** An older build's `stockMovements` does not know `countPart`, so it reads a count group last line wins: the item quantity of that godown differs in an older install until it is updated. This is the one place an older reader's figure can differ, and it exists only after a company has tracked stock and counted by variant; it is said in the release note, and Data Check lists "Physical Stock counts made by variant: N" (CF-9d).

**The invariant (asserted after every movement on the 2,000- and 50,000-voucher companies).** For a tracked item the variants add to the item: the sum of `byVariant[k].qty` is the position's `qty`, and for every godown the sum of `byVariant[k].byGodown[g]` is `byGodown[g]`, to 1e-9. Every movement preserves it by construction (an ordinary line moves one variant by what it moves the item; a count moves the variants by deltas that sum to the item's delta).

**Pending orders by variant (a tracked item only; an untracked item keeps today's keys).** Fulfilment is today keyed on (order number, item), and the voucher screen's pending-order picker only sets `orderRef` (it copies no lines) [C `VoucherScreen.tsx`], so an invoice line cannot be tied to an order line; a variant in the key would leave an order line with no variant (placed before tracking was switched on, or with Sales Order unticked in the field's scope, which AD.3.2 allows) impossible to clear. Per (order number, item), the invoice lines that name the order contribute **pools keyed by their variant** (a blank variant is the pool "any"), and the order's lines for that item are processed in their order: (1) each order line that names a variant takes `min(its quantity, the pool of that variant)`; (2) each order line with a **blank** variant takes from whatever is left in the pools of any variant, in invoice order; (3) what is left in the "any" pool (invoice lines with a blank variant) is then applied to the order lines still pending, in order, whatever their variant. A specific pool is **never** applied to an order line of another specific variant: an order for Red delivered as Blue stays pending for Red (a substitution is a deliberate act, recorded by amending the order). The pending quantity of a line is its quantity less what it took. **Every caller passes the fourth argument** (the picker's per-order total, the Pending Orders report, Summary by field's pending measures, Split Company), and a source scan lists the `pendingOrders(` call sites and fails on one that passes none; **Split Company maps each carried order line by the row's `line`** (it matched the first row of the item by `stockItemId` [C `splitCompany.ts`], which gave every variant line of an order the first line's pending quantity), and does so for a tracked item only, so an untracked company's carry is byte-identical.

#### AD.3.4 How each kind of voucher treats the variant

| Voucher or event | Treatment |
|---|---|
| **Opening stock** (item master) | booked at the item's opening godown (Main Location unless split) as **Not set**, unless `openingAlloc.variants` is present **and passes both trust tests of AD.3.7** (it adds up to `openingQty`, **and** its per-godown totals equal the item's opening by godown), then by variant, each at its `godownId` |
| Purchase, Receipt Note, Credit Note (a sales return comes back in) | the line's variant: inward to that variant's position |
| Sales, Delivery Note, Debit Note | the line's variant: outward from it; the negative-stock warning is per (item, variant) (AD.3.5) |
| **Stock Journal** issue lines (out) and receipt lines (in) | each line its own variant. A receipt row for an item that also has an issue row **starts from the issue row's variant** (tagged "from source line"), so a godown transfer is variant-neutral by default; a different destination variant is a deliberate variant change (rework, dyeing). The transfer value rule (a receiving line with no rate is valued at what the same voucher issued the item at) is per item and unchanged, so a variant change is value-neutral too |
| **Physical Stock** | **a count is always a count of the WHOLE item at its godown, and the item-level figures never depend on a resolver** (AD.3.3, "The count group"; the first text counted "exactly the (godown, variant) position a line names", which moved qty, value and `byGodown` with the resolver and broke the identity claim, and counted a blank-variant line against Not set alone, which double-counted every habitual whole-shelf count). A line with **no variant** is today's line (the item at that godown becomes the counted quantity; its variant effect goes to Not set). **Variant lines are the parts of one count:** the screen saves them with `countPart` and `stockMovements` merges them into one movement of their sum; variants held at the godown and not listed count 0 (the screen pre-fills a row per held variant at its book quantity, so only a deliberate change moves a variant). Items not listed on the voucher are untouched, as today |
| **Manufacturing journal** (`manufacturing.tsx`) | **not variant-aware in v1:** it books Not set (the editor warns when a tracked item has a bill of materials); reclassify with a Stock Journal. Variant-aware manufacture is CF-9f, gated |
| Sales Order, Purchase Order | the line may carry the variant (reporting: pending by variant); no stock effect |
| Cancel (Alt+X), delete, Ctrl+Z | the engine reads the vouchers that remain, so positions follow exactly as they always did |
| Optional, memorandum | never move stock (`stockMovements`), unchanged |
| Alt+2 duplicate, "create from" | the line's variant is copied, and **`use.copy` is forced on for a tracking field** (AD.3.2), so a duplicate or a credit note made from an invoice never books Not set by an unchecked switch (a retired option is still not copied, 2.4: that line starts blank and is refused or books Not set like any blank, tagged "retired option not copied") |
| Costing of any of the above | the item's, unchanged (AD.3.1) |

#### AD.3.4b The writers that do not draw the cell, and what each does about a tracked item

The table above is what the engine does with a line's variant. It is silent about every writer that builds a voucher WITHOUT the voucher screen's cell, and for a tracked item each of them would split stock wrongly in silence: the first text refused a tracking field scoped to fewer bases because "the position would drift", then accepted the same drift for these writers. A line of a tracked item with no variant books **Not set**; the item total stays right while Red and Blue never fall, so the negative-stock warning (a voucher-screen check, only on a new entry) never fires and Stock by field overstates the variants.

| Writer | What happens to a tracked item's line | What the product does about it |
|---|---|---|
| **Bulk Entry** (`BulkEntry.tsx`) | the tracking field is a column of the paste (a column per applicable line field, 3.3); a blank for a tracked item is **refused at the voucher screen's severity when `req` applies** (the rule of 3.3) and books Not set only when `req` was removed, and the row says "books Not set"; **its own negative-stock check** (it keeps a per-item running delta over `posAt(date)` [C `BulkEntry.tsx`]) is made per (item, variant) for a tracked item, with the voucher screen's wording (AD.3.5) | `bulkentry` extended |
| **CSV Import Data and the mapped importer** | the tracking field is a mapping target; an unmapped or blank cell books Not set | the preview counts "N lines of tracked items will be booked as Not set"; the "Enforce mandatory fields" tick (3.3) refuses them as rows; a Data Check row |
| **Tally XML import** | the file carries no variant (UDFs are not mapped until CF-10); lines of tracked items book Not set | the preview and the import report say "N lines of tracked items booked as Not set"; a Data Check row |
| **REST `/v1`** | an integration may send `udf: { <fid>: <optionId> }` on a line; a line without it books Not set. The server holds no definitions until CF-6b, so it **cannot** warn (said, not hidden) | `ApiDocs.tsx` documents the field id and the consequence; CF-6b's create response adds a `variant_missing` warning per line; until then Data Check and the Stock by field banner (below) show the drift |
| **Alt+2 and "create from"** | the variant is copied; the copy switch is locked on (AD.3.2) | none needed |
| **Manufacturing Journal** (`manufacturing.tsx`) | books Not set both ways (components out of Not set, finished goods into Not set); **drift consequence: Red and Blue are not reduced and Not set goes negative** | the editor warns when a tracked item has a bill of materials (7.4); the banner and Data Check below; reclassify with a Stock Journal; CF-9f is the real fix (gated) |
| **Hide / Delete of the tracking field** | refused while it tracks (AD.3.2) | "Stop tracking" is offered instead |
| **An older app** | it cannot draw the cell, so a voucher it **creates** books Not set; a voucher it **edits** keeps its variants through the 9.3 carry (the strong-identity pairing) | the banner and Data Check below; the release order of 9.6 |

**One detector for all of them: a negative Not set.** A sale of a variant draws that variant down; a writer that booked the sale as Not set draws Not set down instead, below zero, which is how the drift always shows. The Stock by field screen carries a banner when any tracked item's Not set position is negative: "Not set holds N units less than zero (M items): stock was booked without a variant by a voucher that did not ask for one. Reclassify with a Stock Journal (Split Not set)." (A whole-item Physical Stock count with no variant does the same on purpose, AD.3.3.) Data Check gains the rows "Tracked items with a negative Not set position: M" (the drift) and, beside the one AD.3.7 names, "Tracked items that still hold stock as Not set: N" (the split still to do) (CF-9d). Both read `byVariant`, so they cost the one `stockPositions` pass the screen already pays.

#### AD.3.5 Value, negative stock, reorder

- **Allocation (`allocateValue(total, qtys)`).** With Q the item's total quantity and q_i the variants': **the value is allocated only when every q_i is zero or positive and Q is above 1e-9**; then `v_i = round2(total x q_i / Q)` and the residual `total - sum(v_i)` (a few paise) goes to the variant of the largest quantity (the first in list order on a tie), so **the variants add to the item's value exactly**. **Otherwise** (any variant negative, or |Q| under 1e-9) every variant's value is 0 and the item's whole value is one **Unallocated** row (a row, never hidden) with the note "a variant is negative: quantities are exact, the value is not split"; the quantities stay exact either way. (The first text gave a negative variant a negative share and said the sum still ties. It ties, but with a small non-zero Q and one negative variant, `total x q_i / Q` swings to absurd figures, e.g. Red +10 and Blue -9.9999 over Q = 0.0001; the conservative reading is the Unallocated row, and an oversold variant already warns at entry.) **The variant cells carry exactly two value figures, Opening and Closing, each allocated by this function at its own date, and no Inwards or Outwards value, no consumption and no value at the (variant, godown) level.** The first text also valued inward and outward cells and derived consumption "as for godowns"; but a godown cell is itself not value-exact (`stockFlow` rounds each godown cell and values waiting at-cost outwards per cell with `round2`, so its cells do not add to the item for Outwards value), pooled costing has no per-variant cost to price a movement with, and a count that raises one variant and lowers another has no honest gross value. So "Sales by Colour" and "Purchases by Colour" in value are **Summary by field's** job (line grain, item value), and Stock by field shows quantities in every column and allocated value at the two ends. **Quantities** of Inwards and Outwards per variant are the per-variant deltas the engine observed; they add to the item's Inwards and Outwards **except where one count group raises one variant and lowers another** (variant A +3 and B -2 against an item delta of +1), where each variant books its own signed adjustment and the footnote says so; **Opening + Inwards - Outwards = Closing holds per variant, and the closings add to the item's closing exactly.** If the allocation is Unallocated at either end for an item, that item's value figures read "n/a" for the window (its value is in the Unallocated row) and its quantities are unaffected.
- **Negative stock.** The save-time warning today groups outward lines by item and compares with `positions.get(itemId).qty` [C `VoucherScreen.tsx`, "negative-stock warning"]. For a tracked item it groups by (item, variant) and compares with `byVariant.get(key)?.qty`, and names the variant ("Shirt (Red) will go NEGATIVE (3 short)!"), still a warning with Continue; for an untracked item the existing line and message are untouched (byte-identical, asserted).
- **Reorder levels are per item and stay so** (the reorder report and Stock Query read `reorderLevel`): a per-variant level is a different feature (AD.6).

#### AD.3.6 Entry on the voucher (slice CF-9c; the variant is a CF-3 line field, so this lists only what is specific to a tracking field)

1. The cell is drawn **only for items in the field's stock groups**; an item outside it shows no cell and books as today.
2. **In the walk when required** (the default): Enter on Amount walks into it (one more Enter per row, only for a company that turned tracking on); with `req` removed it is reached by Tab, and a skipped cell books Not set (the editor says so).
3. **Picker rows show stock.** For an outward line each option row reads "in stock 12" for that item and godown, from the same `positions` memo the screen already builds (`stockPositions` once per screen; the resolver is passed only when one exists, so a company with none runs the identical call). A zero is dimmed, never hidden.
4. **Stock Journal:** the destination row starts from the source row's variant (AD.3.4). **Physical Stock:** choosing a tracked item at a godown **pre-fills one row per variant held there**, each at its book quantity at the voucher's date (the cell of each reads its variant and the row's tag says "part of the count of <item>"); changing a row is the count, an untouched row moves nothing, and **a row deleted by hand counts 0**, which the screen says before Accept ("Black L is not listed: it will be counted as 0"). The lines are saved with `countPart` (AD.3.3). The screen refuses to mix part rows and a whole-item row for one (item, godown) ("Count by variant or the whole item, not both"). With `req` removed, a blank variant is a whole-item count and its delta goes to Not set (the tag says so).
5. `inherit.item` starts a line from the item's usual variant when the field also lives on the item; ghost recall never carries it (2.4).
6. **A size run typed as a grid is not built:** each size is a line (AD.6).
7. Every control has a pointer path (a click target with Clear, like every strip field, 2.1).

#### AD.3.7 History, items without values, switching on and off (CF25)

- **History is Not set, with no migration.** The key is derived from the line's value; a voucher written before the field existed has none, so it reads Not set. No document is rewritten, no Dexie bump, no `BACKUP_SCHEMA_VERSION` effect beyond what v2 data already causes (12).
- **Opening stock:** `OpeningStockAlloc.variants?: { key: VariantKey; qty: number; godownId?: string }[]`, a list that is **independent of `openingAlloc.godowns`** and so can disagree with it (Red 50 and Blue 50 at Main against godowns Main 60, Branch 40: a Physical Stock count at Branch would then resolve against a balance the item never had). It is therefore trusted only while it passes **two tests**, and otherwise **ignored whole, never half trusted**: (a) the quantities add up to `openingQty` within `ALLOC_EPS`; **and (b) its per-godown totals equal `openingGodownMap(item)`** (rows grouped by `godownId`; a row with none counts at the opening godown only when the item opens in exactly one godown, otherwise a row must name its godown, and the panel always writes it; every godown's total must equal the item's opening quantity there within `ALLOC_EPS`, and a godown the map does not hold must total 0). When either fails the whole opening is Not set at the godown map, so `byVariant` and `byGodown` always agree (the invariant of AD.3.3 holds from the first movement), and the panel says which test failed and by how much; the panel checks (b) as it is typed. Written by the **Opening by variant** panel on the Stock by field screen (an administrator or Masters-right action; it registers a dirty probe and is a `Box` with `onClose`) and carried by Split Company Data (`splitCompany.ts` writes `alloc.variants` from `byVariant` beside `godowns` and `batches`).
- **Reclassifying stock already booked as Not set** (the first step after switching tracking on): the Stock by field screen builds, for one item, a **Stock Journal draft** ("Split Not set: 40 Red, 30 Blue, 30 Black") with Not set issue lines and variant receipt lines, value-neutral by the existing transfer rule. The person Accepts it like any voucher (its own rights, its own undo).
- **Switching on, with a preflight in the editor:** "N items in these groups hold stock worth X. Their stock will read Not set until you split it; a sale of a variant with no stock will warn NEGATIVE." **Data Check** gains "Items tracked by Colour that still hold stock as Not set: N" and the negative-Not-set row of AD.3.4b.
- **Stopping** (**Stop tracking**, which AD.3.2 makes the only way to retire the field; or a deleted tombstone after it): the screen returns to the item view; nothing is rewritten; no figure outside Stock by field was ever different (AD.3.1), **including the item quantity of a Physical Stock count made by variant, which is read from `countPart` and not from the flag** (a count group stays one count after tracking stops).
- **An item outside the field's scope, or with no value on any line:** the engine path is today's. The screen lists the tracked items and counts "N items are not tracked by this field".

#### AD.3.8 The Stock by field screen, and Stock Summary's group mode (CF26)

`fieldStock` `{ s: 'fieldStock'; fid?: string; asOn?: string; item?: string }`. Rights: `SCREEN_AREA.fieldStock = 'reports'` (it writes nothing; the Opening by variant panel and the Stock Journal draft are guarded writes, their buttons shown only to a holder of the Masters or Inventory right). Catalogue row under "Analysis by field", gated by the company's inventory feature, a Go To late command. It shares the shell of AD.2.4.

- **Reading 1, an item attribute** (a stock-item choice or yes/no field; banded kinds later): rows are the options and Not set; columns Opening, Inwards, Outwards and Closing quantity (per unit) and Closing value, **all exact** (the sum of the items' own flows). Enter on a row lists its items; Enter on an item opens Stock Item Vouchers.
- **Reading 2, a tracking variant:** rows are the options of the chosen tracking field (marginal over the other tracking fields; with two chosen, Colour then Size as a two-level view), then Not set and removed; quantities **exact** (`byVariant`) in all four columns, Opening and Closing value **allocated** (AD.3.5; "n/a" for an item whose allocation is Unallocated, whose value sits in an **Unallocated** row), **no Inwards or Outwards value** (AD.3.5); a banner for a negative Not set (AD.3.4b) and a footnote where a count raised one variant and lowered another. Enter on a row lists the tracked items holding that variant (quantity, allocated value); Enter on an item opens **Stock Item Vouchers filtered to the variant** (`stockItemRows` with `variant`), and, where two tracking fields apply, a **size-run grid** of the item (the first field down, the second across, quantities).
- F12: as-on date or period, show zero rows, show value, a second level by godown (`byVariant.byGodown`: **quantity only**, Opening, Inwards, Outwards and Closing, **no value at that level**, AD.3.5). The header line states "Stock is valued per item (one cost for all variants); variant values are the item's value shared by quantity."
- **Ties, proved on the sample company and a 50,000-voucher one:** closing quantity per unit equals the Stock Summary's for the same items; closing value equals the Stock Summary's total for the tracked items to the paisa (Unallocated rows included, so the screen total never differs from the Stock Summary's); opening + inwards - outwards = closing in quantity for every row, exactly; **the item-level figures are the same with and without the screen** (the screen reads positions the Stock Summary reads, and a Physical Stock count made by variant is the same whole-item count in both, AD.3.3).
- **Stock Summary gains the same grouping for an item attribute.** `SSMode` gains `'field'` (the field id rides in the F12 `groupBy` enum); `SS_GROUPBY` keeps its static options and positions, and `ssGroupByOptions(defs)` appends one option per stock-item choice or yes/no field after Godown; `rowBuilder.children` gains `viaField` (options and Not set, then the items: the shape of `viaTree` for categories). With `groupBy` at its default nothing changes (the `stocksummary` suite compares the rows deep-equal). Stock Query also lists a tracked item's quantity per variant (optional, CF-9b).

#### AD.3.9 Proof

Suite **`customfields-stock`** (and `inventory`, `stocksummary` and `split` extended on purpose, the commit naming the lines): (1) **identity:** with no tracking field the resolver is `undefined` and the outputs of `stockMovements`, `stockPositions`, `stockFlow`, `resolvedMovements`, `stockItemRows`, `pendingOrders` and the rows of `ui/inventory/ageing.tsx` deep-equal their pre-change outputs on the sample company and a 50,000-voucher synthetic one (**`stockMovements` is deep-equal for every voucher with no `countPart` line**), also with a resolver that answers `null` for every item, and **with a real resolver every existing field of every position (qty, value, avgCost, byGodown, byBatch) is identical and only `byVariant` is added, on a company that HAS Physical Stock counts made by variant** (the count group is read from `countPart`, not from the resolver: the pooled-costing proof); (2) **conservation and the invariant:** after every movement on the 2,000- and 50,000-voucher companies the variants' quantities sum to the item's and, per godown, to `byGodown`, to 1e-9; the allocated values sum to the item's value to the paisa under each of the eight costing methods, for an item of zero net quantity, **and a negative variant gives the Unallocated row (never a negative share), including the small-Q case (Red +10, Blue -9.9999, Q = 0.0001) that produced absurd figures**; (3) **every voucher type of AD.3.4**, including a godown transfer that is variant-neutral, a variant change that is value-neutral, a manufacture that books Not set, cancel and delete-undo, **and the five Physical Stock vectors of AD.3.3 (Red 10 and Blue 10 held; parts Red 8 and Blue 10; parts Red 8 alone; a blank whole-item count of 20; a blank whole-item count of 8) read through the Stock Summary, `stockValueAsOn` (the Balance Sheet and the Profit and Loss), Stock Ageing, Godown-wise and Stock by field, each with and without a resolver: the item-level numbers are identical everywhere**; the count-group edge cases (parts in two godowns of one item are two groups, part lines of two vouchers are not merged, a mix of part and whole lines, a part of quantity 0, a repeated variant summed, an untracked item whose voucher carries `countPart` still merges, tracking stopped afterwards still reads one count); (4) **opening:** the list is trusted only while it adds up **and** its per-godown totals equal the item's opening by godown (a list that adds up but disagrees with `openingAlloc.godowns`, Red 50 and Blue 50 at Main against Main 60 and Branch 40, is ignored whole and the opening is Not set), a stale list is ignored, a split round trip carries variants that pass both tests; (5) **pending orders:** an order for Red fulfilled by a Blue invoice stays pending, by a Red one it clears; **a blank order line (placed before tracking, or with Sales Order unticked) is cleared by invoices of Red 12 and Blue 8; a variant order is cleared by a blank invoice, earliest line first; Red and Blue lines of one order do not each subtract the whole fulfilment; an UNTRACKED item's output is deep-equal to today's, quirk included; Split Company carries each variant line at its own pending quantity (by `line`)**; the source scan that every `pendingOrders(` call passes the fourth argument; (6) **entry:** the negative-stock warning names the variant and fires per (item, variant), an untracked item's message is byte-identical; the Physical Stock screen pre-fills one row per held variant and refuses a mix of part and whole rows; (7) **the writers of AD.3.4b:** Bulk Entry refuses a blank variant for a tracked item when `req` applies and books Not set with the row's message when it was removed; the import previews count their Not set lines; **Hide and Delete are refused on a field that tracks stock and allowed after Stop tracking**; Copy when duplicated is locked on; the banner and the Data Check rows appear on a negative Not set; (8) the screen and the Stock Summary group mode tie, Unallocated rows included. **Mutations it must kill:** key valuation by variant (`bookIn` per variant); **resolve a tracked item's Physical Stock count against the variant balance (the first text's rule); read `countPart` only when a resolver is passed; merge part lines across godowns or across vouchers; merge lines that carry no `countPart`; read a part group last line wins; send a whole-item count's delta to a variant other than Not set; drop the unlisted variants of a count**; store the key on the line; trust `openingAlloc.variants` without the sum check; **trust it without the per-godown check**; allocate by value instead of quantity; drop the residual; **allocate with a negative variant; let the (variant, godown) level carry value; give a variant cell an Inwards or Outwards value**; return a key for an untracked item; sum variants across godowns wrongly; match a pending order across variants; **match a blank order line only by exact variant; subtract the whole fulfilment from each of two lines; map a carried order line by item instead of by `line`**; sort Not set first; **allow Hide on a tracking field; let a Bulk Entry blank through when `req` applies**. **Recipes:** `cf-stock-by-colour` (Case 4), `cf-variant-entry` (the walk with the variant cell, the stock hint, the Stock Journal default, **a Physical Stock count by variant with its pre-filled rows**, the negative warning), each in light and dark, compact and comfortable, and at 150 % text. **Performance:** `stockPositions` with a resolver at most 1.3 x the plain call at 250,000 vouchers [J]; reference row `cf.stock`, not gated.

### AD.4 What this adds to the slice list (the full table, with files, gates and effort, is section 16)

| Slice | New or widened | In one line |
|---|---|---|
| CF-0 | widened | `core/entryPolicy.ts`, `core/fieldDimension.ts`, the type additions (`Company.entryPolicy`, `req: 'policy'`, `tracksStock`, `OpeningStockAlloc.variants`), the policy parameter of `checkUdf` |
| **CF-14** | new | the entry policy end to end: writer, mirror, Workspace panel, the editor's per-field control (its server half, the capability-keyed company-row rules, is CF-6a's) |
| **CF-9** (a to d) | new | stock by field: the variant engine, the screen and Stock Summary's group mode, the variant on the voucher, the migration tools |
| CF-4a | widened | Summary by field over every family, plus Bills by field and Balances by field, plus the new-screen checklist once for five screens |
| **CF-4b-lite** | new | a company-aware filter field list for the four existing filterable reports, now |
| **CF-4c** | new | banded dimensions: text, date and number fields |
| CF-5 | widened | Income and expense by field over five sources, with the Change in stock attribution |
| CF-4b | narrowed | columns and sort through the grid (the Stock Summary group mode moved to CF-9b); it absorbs CF-4b-lite |
| **CF-12** | new, later | employee and lead fields in payroll and CRM reports |
| CF-9e, CF-9f | new, gated | per-variant costing; variant-aware manufacture |

### AD.5 The scenario atlas: the shapes the owner's examples stand for

Each row is a scenario someone will ask for next. "Status" is what this design delivers once the named slice ships; **Different feature** means the field machinery cannot honestly do it and AD.6 says why.

| # | Scenario | Field and where it lives | Served by | Status |
|---|---|---|---|---|
| 1 | Sales by Zone | Zone on the party and/or the invoice | Summary by field, Sales family, Invoice or Taxable value | CF-4a |
| 2 | Collections (receipts) by Zone | Zone on the party | Summary, Receipts family, party legs | CF-4a |
| 3 | Payments to suppliers by supplier Category | Category on the creditor ledger | Summary, Payments family, party legs | CF-4a |
| 4 | Purchases by supplier Region; by item Brand | Region on the party; Brand on the item | Summary, Purchases family; `party` and `item` sources at line grain | CF-4a |
| 5 | Returns by reason | "Return reason" on the Credit Note header | Summary, Credit Note alone (`asRegister`, positive) | CF-4a |
| 6 | Net sales after returns by Colour | Colour on the line | Summary, Sales with Credit Note (`net`), Item value | CF-4a |
| 7 | Pending sales orders by Salesman; by Colour | Salesman on the order header; Colour on the order line | Summary, Orders family, Pending value or quantity | CF-4a |
| 8 | Profit by Zone, by Branch, by Business line | a header field on any voucher types | Income and expense by field, `own` | CF-5 |
| 9 | Profit by item Brand or Category, with the stock movement | an item attribute | Income and expense by field, `item` (the Change in stock is attributed exactly, R10) | CF-5 |
| 10 | Profit by customer Category or Salesman of the party | a party attribute | Income and expense by field, `party` (current) | CF-5 |
| 11 | Fixed against variable cost | a field on the expense ledgers | Income and expense by field, `acct` | CF-5 |
| 12 | Profit by Colour (when Colour tracks stock) | a tracking line field | Income and expense by field, `line`, stock change allocated | CF-5 and CF-9 |
| 13 | Receivables and payables ageing by Region or Salesman | a party attribute | Bills by field | CF-4a |
| 14 | Debtor balances by Region, including ledgers that are not bill-wise; bank balances by bank type | a ledger attribute | Balances by field | CF-4a |
| 15 | Closing stock by Brand, Material, Season | an item attribute | Stock by field, reading 1; Stock Summary group mode | CF-9b |
| 16 | Closing stock by Colour | a tracking line field | Stock by field, reading 2 | CF-9 |
| 17 | Stock by Size and Colour, a size run for one item | two tracking line fields | Stock by field, combination and size-run grid | CF-9b |
| 18 | Sales by month of delivery (a date field) | a header date | Summary with a period dimension | CF-4c |
| 19 | Sales by credit-days band; by order-value band | a number field | Summary with a band dimension | CF-4c |
| 20 | Sales by Project code typed as free text | a text field | Summary with a text dimension (capped at 200 values and "Others") | CF-4c |
| 21 | Filter the Sales Register, Day Book, Ledger Vouchers or Bills by any field | any field | the four existing reports' filter | CF-4b-lite |
| 22 | The same field as a column and a sort in those reports | any field | the grid's Columns panel and header menu | CF-4b |
| 23 | An old invoice with no Zone is edited | a mandatory field | the entry policy: allow, require from a date, or require always | CF-14 |
| 24 | Employee Department in the payroll statement; Lead source in the pipeline | an employee or lead field | **Different feature** for now | CF-12 |
| 25 | A per-Size reorder level; a per-Colour selling price; a per-variant barcode or SKU | | **Different feature** (AD.6) | not built |
| 26 | Jeweller's stock by Purity with a different cost for each | a tracking line field | quantity: yes; **value by variant needs per-variant costing** (or one item per purity) | CF-9e, gated |
| 27 | Manufacture a finished good in a chosen colour | | **Different feature** until CF-9f | gated |
| 28 | Budget against actual by field | | **Different feature** (budgets are by ledger and cost centre) | not built |
| 29 | Balance Sheet or Trial Balance by a header or line field | | **No honest meaning** (6.7); Balances by field covers a ledger's own field | not offered |
| 30 | One invoice split 60/40 across two business lines; an invoice for two projects | | **Different feature**: a cost centre or category splits an amount; a field labels a record (CF2, 6.7) | not offered |
| 31 | A custom value in a GST return, an e-invoice or an e-way bill | | **Never** (CF14) | not offered |

### AD.6 What still needs a different feature, plainly

- **Value by variant where cost differs by variant.** Pooled costing (CF24) shares the item's value by quantity. A separate cost per variant is per-variant costing (CF-9e): it would change the item-level closing value, so it is opt-in per field, gated on question 18, and never the default. Until then: one item per variant.
- **Per-variant reorder levels, price lists, GST rates, barcodes and SKUs.** `reorderLevel`, `prices` and the GST history live on the item; none is keyed by variant. They need their own slices if asked.
- **Manufacturing by variant** (a bill of materials per colour; a finished good in a chosen variant): the manufacturing journal books Not set (AD.3.4). CF-9f, gated.
- **Typing a size run as a grid** (one row per colour, one column per size): each size is a line today. A grid entry is a voucher-screen feature, not a stock one.
- **A voucher field on a bill** (Bills by field reads the party's field): needs the originating-voucher index (4.3).
- **A second dimension in one report** (Zone and Salesman together as rows and columns): the engines group by one dimension. A second dimension across the columns exists only for Income and expense (Layout B, CF-5b); no other report pivots.
- **A warn-only tier for old records** ("ask, but let me skip"): question 15's default stands; the three modes of AD.1 are never, from a date, always.
- **Server-side enforcement of mandatory fields** (REST and older apps): CF-6b enforces creates; alterations need the server to read the policy and definitions. Until then enforcement is the client's (AD.1.6).
- **Employee and lead reports by field**: CF-12.
- **Accrual or cash basis choices, multi-currency, split of one amount across options**: not part of this family (6.7, CF2).

### AD.7 Older passages the addendum changed in place (so the two never disagree)

| Where | What changed |
|---|---|
| Header | a "Revision (2026-10-22, owner answers)" paragraph |
| Section 0 | CF2, CF3, CF6, CF9, CF11 reworded; CF19 to CF30 added |
| 1.3 | `req` gains `'policy'`; `tracksStock` added; the `FieldScope.stockGroups` comment; `reportFieldId` gains `cfa.` |
| 1.6 | the v1 `required: true` conversion writes `req: 'policy'` |
| 2.1 to 2.4 | the walk set W per voucher follows the policy (2.1); a line field labels stock unless it tracks it (2.2); a required master field on alteration (2.3); a bound alteration starts blank (2.4) |
| 3.1 to 3.3 | `CheckArgs` takes `policy` and `dates`; the two `req` rows of 3.2 become one `req` row and one mode row; the backstop sentence of 3.3 |
| 4.1, 4.2 | the analysis row of the matrix; **the navigation additions, complete (below)** |
| 5.1, 5.2, 5.4 | the `acct` source and `cfa.`; `CfColumn.role` and `sortKey`; a note after the 5.4 table |
| 5.5, 5.6 | 5.5 gets a banner and the `net` sign rule (AD.2.5 supersedes its signatures); 5.6 is rewritten for stock by colour |
| 6.2 to 6.7 | the P&L row of 6.2; banners on 6.3 and 6.4; the margin bullet; four rows added to 6.5; the Balances exception in 6.7 |
| 7.1, 7.2, 7.4 | the sample row; the Required row; the new Stock row; two warnings |
| 8 | three rights rows |
| 9.3, 9.6 | the company-row rule bullet (**rewritten by the third review: capability-keyed**), the line-pairing rule (**strong identity first**); the deploy window; the release gate now names CF-4b-lite |
| 10 | Fill blanks is the companion of the `always` policy |
| 13 to 15 | performance rows; new suites and the widened ones; new recipes |
| 16, 17 | lanes (I added), the slice table, effort, the CF-4b-lite contract; the build-order diagram, items 6, 7, 7a, 7b, 10, 11 and the parallel-files paragraph |
| 18 | questions 1 to 3 marked ANSWERED; 18 to 23 added |
| 19, 20 | what the addendum did not verify; what it read |
| Appendix | Case 1 step 6, Case 2 (JSON, step 1, step 8), Case 3's `req`, Case 4 added |
| `docs/design/back-nav-spec.md` | section 4 gains the new screens and panels; section 13's order and numbers follow section 17 here (and, after the third review, the CF-6a line of its build-order diagram) |
| **The third review (2026-10-22), in place** | the header's third Revision paragraph and the addendum's Status; CF23, CF24, CF25 (section 0); AD.1.2's Server row and AD.1.6 (5); AD.2.1's `party` row (**`partyOf`**), AD.2.4's Bills and Balances rows, AD.2.5 (the party-leg bullet, the Orders bullet, the new **Stock vouchers** bullet, the reconciliation bullet), AD.2.6 (R7, R10, the `FieldPl` shape and the carried-rules paragraph), AD.2.7 (**rewritten**); AD.3.1, AD.3.2 (Hide, Stop tracking, Copy locked), **AD.3.3 (rewritten: the count group, `countDeltas`, the invariant, pending orders by variant)**, AD.3.4 (Physical Stock, Opening, Alt+2) and the new **AD.3.4b**, AD.3.5 (allocation, cell values), AD.3.6 item 4, AD.3.7 (opening trust, stopping), AD.3.8, AD.3.9; 1.3 (`countPart`); 5.4 (the Day Book party row); 6.3 R2 and 6.4 (the six reserved subtrees, the memo row); 7.2 (the Stock row, Actions) and 7.4; 9.1 A2; 9.3 (the line-pairing rule, the `UDF_CARRIERS` block, the company-row bullet); 9.6 step 2; 10 (Hide, Delete); 11 and 12 (`countPart` in the schema-3 rule); 14 (six rows and two paragraphs); 15 (five recipes); 16 (lane I and S, CF-0, CF-4a, CF-6a, CF-14, CF-9a, CF-9c, CF-9d, the totals); 17 (the diagram, 7a, 7b); 18 (question 24); 19 (items c and the P&L line); Case 4 step 4; the Review log (third pass) |

**The navigation additions, complete (they are visible to every company and are the one deliberate exception to the identity claim, CF3):** (1) the left menu's "Custom Fields" row (65 to 66 rows) and its `+` entry "New custom field"; (2) **five Reports Center rows** in a new "Analysis by field" group; (3) **late Go To commands** for the five screens, the entry-rules panel and Custom Fields (each `late`, so the 103-query golden does not move); (4) the Settings > Workspace panel "Custom-field entry rules"; (5) the Stock by field row is shown only when the company keeps stock items. No Gateway row, no new chord, no change to the F-key rails.

### AD.8 The new-screen checklist for the five analysis screens (done once, in CF-4a, by one writer)

| Checklist item (CLAUDE.md) | Decision for `fieldSummary`, `fieldPl`, `fieldBills`, `fieldBalances`, `fieldStock` |
|---|---|
| `Screen` union (`state/store.ts`) | five members, each with its params (`{ s: 'fieldSummary'; fid?; family?; bases?; measure? }`, and so on); registered once, in CF-4a |
| Route and crumb (`ui/router.tsx`) | one route each; crumbs "Summary by field", "Income and expense by field", "Bills by field", "Balances by field", "Stock by field" |
| `SCREEN_AREA` (`core/access.ts`) | all five `reports` (they write nothing); `security-access.assert` lists them |
| `MenuScreen.tsx` (Gateway) | no row of their own: reached from the Reports Center, Go To, the palette and the field editor's "Open Summary" button. If `reportscenter.assert` or the new-screen rule insists on a Gateway entry, one "Analysis by field" entry in the Display More Reports submenu opens the Reports Center group (decided at CF-4a; not a reason to add five rows) |
| `CommandPalette.tsx` and the `overlays.tsx` Go To | five **late** commands (the `late` flag of `paletteRank.ts`), so no existing query is re-ranked |
| `state/recents.ts` `TITLES` | five titles |
| `ui/favouritesCore.ts` | the Reports Center star is the pin; the drift guard reads the five rows |
| `ui/navTree.ts` and the `sidenav.assert` literal | **no new row** (the menu stays at 66, the Custom Fields row only); the pin and feature maps (`DRILL_FEATURE`, `needsOf`) say `fieldStock` needs the inventory feature |
| `ui/quickCreate.ts` and `QC_PAGES` | none: a report creates nothing, and the Opening by variant panel is not a create |
| Reports Center catalogue (`ui/reportsCenter/catalogue.ts`) | five rows in "Analysis by field"; `EXCLUDED_REPORT_SCREENS` is not touched |
| Dirty probe (`state/dirty.ts`) | none for the five screens (they hold no typed work); **the Opening by variant panel and the Entry rules panel register theirs** (`navstack.assert` extended) |
| Back control | the five are plain screens: Back enabled, no probe (back spec section 4) |
| Dialogs | every new `Box` (Opening by variant, the F12 option panels, a Reset confirmation) passes `onClose` (back spec B-2a's scan) |
| Light, dark, compact, comfortable, 150 % text | tokens only; sizes through `sz()` or `calc(Npx * var(--ui-k))`; the new tags join the `visual.assert` contrast matrix; the screens join the 642-screen sweep |

### AD.9 CF-4c as built (2026-10-24, the reports fix round), and where it differs from the text above

**Built:** the text, period and band kinds (`core/fieldBands.ts`, reached through `dimensionFor(def, sources, spec)`), on all five analysis screens: Summary, Income and expense, Bills, Balances and Stock by Field (item attribute). A typed text is one bucket per cleaned value (NFKC, case-folded, whitespace collapsed, read to 200 characters before the fold), labelled by its first spelling, at most 200 buckets and the rest in one `other` row "Others (n values)" whose amounts are kept (the 200 kept are the values held by most documents); a date is a month, a quarter or a financial year by the company's first month, the span the book holds declared so an empty month can be shown; a number, amount or percent is banded by typed edges (at most 12; a band holds its lower end). A stored value that is not of the field's shape is one row ("Not a date", "Not a number"). The engines changed in two places only: `kindOfKey` (the one rule that says what kind of row a bucket key is) and `extraOrder` (the order of rows no list declares: a typed or banded dimension by key, Others last); a choice field reads exactly as before.

**Differences from the text above:**
1. **The period and the band edges are chosen with a chip and the key B on the screen, not in F12.** F12 offers cycle rows only and cannot take typed text, and an F12 row that does nothing for a field with no period or bands is the kind of option `reportConfig.ts` forbids. The chip (and the rail button) appears only when the chosen field is a date (Month / Quarter / Financial year) or a number, amount or percent (a box in which the edges are typed, the bands read back as they are typed). The choice is remembered for that field on that device (`TW_FIELD_DIM`, a list of pairs; never the definition, the company or another person).
2. **A number field starts with round bands** worked out from the values the book holds (`suggestEdges`: about five bands at 1, 2 or 5 times a power of ten), so the report answers at once; the screen says they were worked out. They are never stored until the person types edges.
3. **Not built, said plainly:** column and sort for these types in the listing reports (they wait for the grid, CF-4b); the Stock Summary's own group mode still offers choice and Yes/No attributes only (it has no control for a period or bands); a lookup field cannot group yet (CF-8; the screen says "a link to another record cannot be used to group yet"); a multi-choice field is still refused by decision (a record in two groups would add to more than the whole).
4. **"Available in reports" off now removes a field from every picker of the five screens** (spec 4.1 and 4.4), including a field that keeps stock (Stock by Field); the field editor says so beside the switch.
5. **Proof:** the suite `customfields-bands` (the pure kinds worked on paper; a differential of all five engines, banded against a choice field written from the same values by an independent oracle, on 2,000 and 50,000 vouchers; the 200-value cap through Summary, Income and expense, Bills and Balances with their ties) and the recipes `cf-bands-*` (the spec's `cf-dimension-bands`, in `scripts/shots/recipes/cf-bands.mjs`: text, date, number, the edge box by key and by pointer, the other four screens, 150 % text). The same file holds the Physical Stock recipes for the count rows that follow the date and the godown (`cf-bands-count-date`, `cf-bands-count-godown`) and the one for the squeezed hint of a custom text field (`cf-bands-hint-150`).

---

## A. What I verified in the tree (and what changes the research docs' advice)

1. **The tree is moving.** `HEAD` is `83d9876`; `git status --short` showed 158 paths (64 modified, 94 untracked) [C]. Files this
   design must edit that the wave-3 build is also editing: `src/ui/VoucherScreen.tsx` (+341 lines, 4,119 lines in the tree),
   `src/ui/masters.tsx`, `src/types.ts` (+69), `src/state/store.ts`, `src/ui/reports.tsx`, `src/ui/router.tsx`, `src/ui/overlays.tsx`,
   `src/ui/widgets.tsx`, `src/sync/sync.ts`, `backend/src/sync.js` (+30), `src/core/shortcutRegistry.ts` (+62) [C `git diff --stat`].
   **No slice of this spec may start before wave 3 is committed** (section 17).
2. **`Alt+M` is Share, and there is no free Alt letter.** The registry now lists `share` on `Alt+M`, `cancelVoucher` on `Alt+X`,
   `priceLevel` on `Alt+Z`, `quickCreate` on `Alt+Q`, `pdf` on `Alt+F9` [C `core/shortcutRegistry.ts`]. Counting the registry's
   `REBINDABLE`, `RAIL_FIXED` and `NEW_SHORTCUTS` lists, every letter A to Z is either bound (A alter or add, B batch, C new ledger,
   D delete, E relief, F forgot PIN, G Go To, H GST history, I new item, J GST adjustment, K hold, L optional or login, M share, N after
   accept, O logout, P print, Q quick create, R recall or API read, S sort or dispatch, T post-dated, V saved views, W API write,
   X cancel, Y challan, Z price level) or held (U for attachments) [C]. A literal grep of `src/` finds `Alt+8` and `Alt+9` once each,
   `Alt+1`, `Alt+3`, `Alt+4` on the Webhooks screen, and **no use at all of `Alt+F1`, `Alt+F8`, `Alt+F10`, `Alt+F11`** [C]. So: the
   wave-a2 plan's `Alt+M` for the Columns panel (decision A2-1) is void, and this spec adds no chord (2.5).
3. **The report grid does not exist and neither do its before-pictures.** `src/core/reportFields.ts`, `listQuery.ts`, `listLayout.ts`,
   `src/ui/grid/` are absent [C `ls`]; `scripts/shots/golden/` holds only `dash-*`, `plfigures.*` and `goto.*` captures [C `ls`], so
   the "golden captures of today's tables" the wave-a2 plan made its step A2-0 item 0.1 have not been recorded. The Dashboard half of
   A2 (B-dash) **did** ship, as Dashboard v2, in `5c94342` [C CLAUDE.md]. So what remains of A2 is the grid (B-grid) and its scaffold.
4. **REST writes replace the whole document and validate nothing**: `POST /v1/:collection` inserts the body as the new current
   payload [C `backend/src/rest.js:58-75`]; `customFields` is in `REST_DENIED` [C `restPolicy.js:18`]. Webhooks fire only from REST writes.
5. **The server's per-field protection is two lists**: `PROTECTED_FIELDS` (passcode and PIN hashes, on a stale or unversioned write) and
   `UNVERSIONED_PROTECTED` (`companies.ui`, only on a write that states no base) [C `backend/src/conflict.js:38-56`, `:196-206`]. An older
   app that already sends a base (everything since Phase 24) is covered by neither, so a new voucher field would be erased by it and the
   server would accept the write as an ordinary edit. `backend/src/sync.js` is already modified by wave 3 (+30 lines).
6. **Dexie is at version 8 and wave 3 did not bump it** [C `src/data/db.ts`: the diff against `HEAD` is one option on `updateDoc`].
7. **The existing UDF code is thin, exactly as the audit says**: `collectUdf` rebuilds `udf` from the current definitions only (an orphaned
   value is dropped on the next save), stores a number only when non-zero and a yes/no only when Yes, keeps a date as typed text, and
   keeps a list value as the option's label; `required` is read nowhere [C `src/ui/udf.ts`, `src/ui/admin/CustomFields.tsx`]. Its editor
   assigns `id: crypto.randomUUID()` and `key: slug(label)` of at most 40 characters [C `CustomFields.tsx`].
8. **A button name's cleaner is 24 characters** (`MAX_LABEL = 24`, `cleanLabel` in `core/uiPrefs.ts`), so a field label needs its own
   cleaner (40 characters), built on the same invisible-character set [C].
9. **Chords inside the voucher**: nothing to reuse; all 26 letters are spoken for (item 2). The voucher's narration handler already lets
   a bare Tab fall through to the browser when no snippet blank is open (`case 'narr'` in `VoucherScreen.tsx`) [C], which is what makes the
   strip reachable with no chord (2.1).
10. **The Esc rail button and Tab order**: not this document's business; see the Back spec.
11. **Vendor facts re-checked today [W]:** (a) Zoho's item custom field becomes a column of the item table of transactions, with three switches
    (show when creating transactions, which transaction types, show in all PDFs); the page does not say whether it can be mandatory or whether
    a line defaults from the item master. (b) Pipedrive: a field's "places shown" setting controls the add and detail views, list-view
    columns are chosen separately with the gear, the type cannot change after creation, deleting a field deletes its data, and an admin can open
    field statistics (created, last updated, how often it is filled, whether a filter uses it) before editing or deleting. (c) Zoho's Profit and
    Loss by reporting tag is either "Compare With" tags (columns) or a More Filters condition on the tag; the page does not say how untagged
    amounts are handled. Everything else vendor-side below is **[R]**, taken from the research documents.
    (d) The review quoted Pipedrive's "quality rules" and I re-read the page in the second pass: a field can be marked **Important** (highlighted in the deal view when empty, the deal can still be saved; the page ties it to a mid-tier plan) or **Required** (the deal cannot be saved until it is filled; a higher tier) [W https://support.pipedrive.com/en/article/custom-fields]. Not adopted in v1 (question 15).
12. **Code facts re-read in the second pass** (each was a premise of a review finding; all [C], symbols stable, line numbers will drift):
    (a) `backend/src/sync.js loadHistories` loads `latestOther` only for tables in `PROTECTED_FIELDS` or `UNVERSIONED_PROTECTED` (the `protect` flag) and `others` only for rows NEWER than the writer's base, so `classifyPush`'s `anchor` is null when the writer's base is at or past a document's newest foreign row; the push route reads no `caps` today; a mismatch past `LIMITS.openRows` (2,000) is not recorded (a `log.warn`, `up.capped`). The shipped client pushes at most 300 documents per push (`PUSH_BATCH`, `sync/sync.ts`) and `change_log` has `idx_change_doc (tenant_id, tbl, doc_id, seq)`; the pull answer is `{ changes, cursor }`.
    (b) The older master forms rebuild `udf`: `masters.tsx` and `inventory/masters.tsx` set `udf: collectUdf(...)` after the `...editing` spread; `udfDefsFor` keeps only definitions whose `objectKind` equals the form's kind; `collectUdf` runs a number through `parseAmt` (rounds to 2 dp, so `0` is dropped by `if (n)`), stores only `Yes` for a toggle and keeps a trimmed string otherwise; `udfInitial` does `String(v)`.
    (c) `putDoc`'s `system: true` skips `checkWrite` (the operator guard: base rights, the date window) and stamps no user; `saveVoucher` takes no options. Callers of `saveVoucher`: `VoucherScreen.save`, `useCancelVoucher`, the Ctrl+Z applier in `overlays.tsx`, `BulkEntry`, `ImportData`, `manufacturing.tsx`, `payroll/vouchers.tsx` (twice), `payroll/disbursement.tsx`.
    (d) `core/undo.ts record()` never puts an `origin: 'flag'` row on the undo stack (it hands it to listeners and returns); `ui/reports/markOptional.ts` runs its batch inside ONE outer `db.transaction`, collects entries with `onMutation`, and owns `undoMark` and `batchIsLatest`; `updateDoc` opens one transaction per document.
    (e) `core/backupMigrate.ts` refuses a file whose `schemaVersion` is above the build's ("an older app silently dropping fields a newer one wrote would lose data on the next save"); `backupHeader` always writes `BACKUP_SCHEMA_VERSION`.
    (f) The v1 editor builds `taken` per `objectKind`, so a key repeats across kinds; `slug()` cuts at 40 and the collision loop appends `_2` and up, so a key can have 41 to 43 characters.
    (g) Voucher entry: `validInvLines` never merges rows, so one item on several lines has one identity (`stockItemId`, `godownId`, `batchName`) several times; the item grid draws one `<input>` (the focused cell) and a `<span onClick>` for every other cell, `ItemCol` is a string union compared with `===`, and `case 'item'` has no Tab branch (today Tab is the browser's and lands on Narration); Enter on an empty Place of Supply keeps it blank while Enter on an empty party picks the highlighted row; `useTallyForm`'s `pickCurrent` picks the highlighted row (row 0) of any select and a toggle has only Y, N and Space.
    (h) `tallyImport.ts` sets `itemLedgerId` only when every line posts to ONE allocation ledger (`core/einvoice.ts` and `ui/print/invoiceModel.ts` carry an `unpointed` branch for the rest); `extraLedgerPopup` offers every ledger but the party, so one ledger can appear twice on an invoice.
    (i) `PrintConfigScreen` builds one toggle row per `PRINT_PARTS` entry inside the Enter walk and `signature` is last. `reportFilter.describeCondition` prints the raw field id for a field it has no metadata for and the raw value for an unknown option. `styles.css` fixes the item table's `qty`, `rate`, `amt` and `gdn` widths at 110, 110, 140 and 150 px, unscaled by `--ui-k`. `CustomFields.tsx` registers no dirty probe and `mayHoldWork` knows only `bulkEntry`, `importData` and the create forms.
    (j) The F12 panels (`reportConfig`, `VoucherConfig`) are `Box` modal shields. The Esc of `TaxPrompt` records a decision (skip this deduction); that is the back spec's finding, recorded there.
    **Measurements the review quoted** (the 1024 px grid of 212 / 660 / 152 px, the item table at 650 px, the top bar's items summing to about 956 px) are the reviewer's, or arithmetic on the CSS: **not re-measured here** (the dev server belongs to the wave-3 builders).

---

## 1. The model

### 1.1 Objects ("places"), and why one definition can have several

| Place (`UdfKind`) | What carries the value | Status | Asked where |
|---|---|---|---|
| `ledger` | `Ledger.udf` | exists | the Ledger master form (and the Alt+C quick-create overlay, only if required) |
| `stockItem` | `StockItem.udf` | exists | the Stock Item master form |
| `voucher` | `Voucher.udf` (**new**) | **new, slice CF-2** | a strip under Narration on the voucher types the field is scoped to (at most 12 header fields per base, 1.5) |
| `line` | `InventoryLine.udf` (**new**) | **new, slice CF-3** | a field of each item row on the voucher types scoped: an inline column after Amount when the voucher body is wide enough, else a sub-row under the item (at most 6 per base, 2.2) |
| `employee` | `Employee.udf` (**new**) | slice CF-8 | the Employee master form |
| `lead` | `Lead.udf` (**new**) | slice CF-8 | the Lead form |
| `entry` | would be `VoucherEntry.udf` | **reserved, never offered** | none |

**Why `entry` is not built.** The accounting grid's rows are system-shaped in invoice mode (party, item ledger, taxes, round off), they are
rebuilt at four sites (`AccRow` and the three entry literals in `VoucherScreen.save`), and the entry-level dimension already exists as cost
centres (`VoucherEntry.costs`, with amounts, the Tally model, round-tripping through Tally XML) [C audit 2.2 A3, A4, 5]. A custom field on a
payment line ("purpose") is a real but rare need; it is a later slice if asked, the type union already reserves the name so ids stay forever.
**Not offered anywhere (and why):** unit, godown, cost centre, voucher type, company, group. Each needs its own form wiring for little demand;
the editor says so rather than listing them.

**One definition, several places (CF1).** `objectKind` stays the **primary** place (what an older app reads); `also` lists the others.

- *Colour for a clothes dealer:* `objectKind: 'stockItem'`, `also: ['line']`, `inherit.item: true`. One option list (Black, White, Red...),
  asked on the Stock Item form (an attribute of the item) and as a column on the sale line (a choice per line, starting from the item's value).
- *Zone for a distributor:* `objectKind: 'ledger'`, `also: ['voucher']`, `inherit.party: true`, `scope.vch.bases: ['Sales','CreditNote']`. One
  option list (North, South, East, West), asked on the party and on the invoice (starting from the party's value).
- *A "profitability tag" such as Business line:* `objectKind: 'voucher'` with `scope.vch.bases` = Sales, Purchase, Credit Note, Debit Note,
  Journal, Payment, Receipt: one list, created once, "tagged onto" seven voucher types. This **is** Zoho's tag shape inside the voucher world (6).

**The primary-place rule (CF16), for the written row.** `objectKind` is what an older app reads, and an older app that reads a definition **owns** it: its master form rebuilds that field's value on every save (A, item 12b), rounding a number to 2 dp, dropping `0` and `No`, flattening a multi-choice array to text, turning an amount or percent into a string. A merge on the server can restore a key that an older writer OMITTED; it cannot repair a value it MUTATED (9.3). The first draft ordered the primary kind "so that an older app shows as much as it can" and called master fields safe at any time: that is false (the review's finding 2). The written `objectKind` is now chosen to keep the older app away from anything it would damage:

1. Take the places the admin ticked, in the order ledger, stock item, voucher, line, employee, lead.
2. If the first is a **master place** (ledger or stock item) and the field's type is `text` or `select` (the two types an older form round-trips without loss: it keeps a trimmed string, and an option id it does not touch comes back unchanged), that place is `objectKind`.
3. Otherwise the first ticked **non-master** place is `objectKind` (an older app ignores voucher, line, employee and lead), and the master places go in `also`.
4. If nothing else is ticked (a number, date, yes/no, amount, percent, multi-choice, lookup, long text, email, phone or url field on a master place only) `objectKind` is the sentinel **`'v2only'`**, which no older editor lists, no older form renders and no older count reads, and `also` holds every place.

`placesOf(def)` is the one reader of a definition's places: `def.objectKind === 'v2only' ? (def.also ?? []) : [def.objectKind, ...(def.also ?? [])]`; nothing else reads `objectKind` to place a field. A **v1 row** (no `v: 2`) keeps its `objectKind`, and the new master forms write its values with the **v1 profile**, the same bytes the older `collectUdf` writes (a 2-dp number with `0` dropped, a Yes-only toggle, trimmed text), so a company that never converts a field sees no change in what its master forms store (asserted against `collectUdf` on the same inputs). "Update older fields" (1.6) converts a v1 `number`, `date` or `toggle` field to the v2 rules and, by this rule, away from the older editors.

What an older app still does to a field it does not own: it **omits** that field's key on every resave of that master (its form rebuilds the whole bag). Omission, unlike mutation, is repairable: by the server rule for a synced company (9.3) and, for a restore onto an older install, by the backup marker (12). The remaining exposure is every field for which `needsServerCap` is true, and the editor deals with it at the door (7.4, 9.6):

```ts
/** false only for the one shape an older app both owns and round-trips without loss: a text or choice field on exactly one master place. */
export const needsServerCap = (def: CustomFieldDef): boolean => {
  const p = placesOf(def);
  return !(p.length === 1 && (p[0] === 'ledger' || p[0] === 'stockItem') && (def.type === 'text' || def.type === 'select'));
};
```
An older app also shows the **raw option id** (not the label) for a v2 option it has not renamed in `options`, since it reads the stored string; an untouched value round-trips unchanged and a value it picks is a label, which the reader of 1.4 resolves.

### 1.2 Types, and the stored value of each

Stored value `UdfValue = string | number | boolean | string[]`. **Absent key = unanswered.** The writer deletes a key when a field is cleared:
never `''`, never `null`. `0`, `false` and an empty-but-answered Yes/No are values.

| `UdfType` | Name in v1? | Stored | Entered as | Report `kind` | Groupable | Slice |
|---|---|---|---|---|---|---|
| `text` | yes | string, cleaned, at most `rules.maxLen` (cap 200) | typed | text | no | CF-1 |
| `longText` | new | string, at most 2,000 | multi-line box; **never a column** (a preview in the picker only) | text (filter only) | no | CF-7 |
| `number` | yes | finite number, at most 3 decimals (the row filter compares quantities to 0.0005), **exact** (no 2-dp rounding) | arithmetic, **no magnitude shorthand** (like a quantity, `parseQty`) | number | no | CF-1 |
| `amount` | new | number rounded to paise | arithmetic and Indian shorthand (`parseAmtInput`) | number, `money` | no | CF-1 |
| `percent` | new | number 0 to 100 (rules may narrow) | arithmetic | number | no | CF-1 |
| `date` | yes | **ISO** `YYYY-MM-DD` | the voucher date's letters and mnemonics (`parseDateInput`, mnemonics on) | date | no | CF-1 |
| `toggle` (Yes/No) | yes | `true` or `false`; three states (unanswered, Yes, No) | Y, N, Space, or the three buttons; Delete or the blank button clears | enum Yes, No | yes | CF-1 |
| `select` (choice) | yes | option **id** | type-ahead popup of active options | enum, options = the list | yes | CF-1 |
| `multiSelect` | new | `string[]` of option ids, de-duplicated, in list order, at most 50 | popup that ticks several | text (the joined labels) | **never** | CF-7 |
| `email`, `phone`, `url` | new | string, at most 200: a shape check (`url`: scheme `http`, `https`, `mailto` or `tel` only), linear scan. **Kept and drawn as text in v1, never as an anchor**: a later link renderer must re-check the scheme at render time, because REST and older apps can write any string (`javascript:...` must never become clickable) | typed | text | no | CF-7 |
| `lookup` | new | a master id (`to`: ledger, stock item, godown, cost centre, employee) | the master picker | text (the master's **current** name; a deleted master reads "(deleted master)") | ledger only, later | CF-8 |
| attachment, image | not built | n/a | n/a | n/a | n/a | out |

`select` and `toggle` keep their v1 names; every new type has a new name, so an older app that does not know it falls to its `default` branch
and treats the field as text. **Why multiple values are never groupable (T11):** a line in two groups makes the groups add to more than the
whole. If an amount genuinely belongs to several places, that is what a cost category is for [R zoho-reporting-tags 8.3 item 2]. **A v1-named type does not make a field safe for an older form** (1.1): only `text` and `select` are round-tripped without loss.

### 1.3 The definition, in TypeScript (`src/types.ts`; every v2 property optional, so every row written before this spec is valid as it stands)

```ts
export type UdfKind = 'ledger' | 'stockItem' | 'voucher' | 'line' | 'employee' | 'lead';   // 'entry' reserved, see 1.1
export type UdfType =
  | 'text' | 'number' | 'date' | 'toggle' | 'select'                                   // v1 names, unchanged
  | 'longText' | 'amount' | 'percent' | 'multiSelect' | 'email' | 'phone' | 'url' | 'lookup';
export type UdfValue = string | number | boolean | string[];
export type UdfValues = Record<string, UdfValue>;                                     // was string | number | boolean

export interface FieldOption { id: string; label: string; active?: boolean; gone?: boolean }   // active absent = true; gone = removed from the editor's list, never offered, label kept for history (10)

/** Where a field is asked on a voucher: these bases, and (optionally) only these voucher types under them. */
export interface VchScope { bases: VchBase[]; typeIds?: string[] }
export interface FieldScope {
  groups?: string[];       // ledger: ledger-group ids, sub-groups included; absent = every ledger
  stockGroups?: string[];  // stockItem: stock-group ids, sub-groups included; absent = every item. For a field that tracks stock (AD.3.2) it also names the items whose lines carry the variant, even when the field has no stockItem place
  vch?: VchScope;          // voucher header
  line?: VchScope;         // item lines (bases that carry inventory: invoice or stock bases only)
}
export interface FieldRules {
  maxLen?: number;                                          // text, at most 200
  pattern?: 'digits' | 'alnum' | 'alnumDash' | 'letters';   // PRESETS ONLY (linear char-class scans); no user regular expression
  min?: number; max?: number; decimals?: number;            // number, amount, percent (decimals 0..3)
  unique?: boolean;                                         // voucher header only; RESERVED, not built in v1
  additive?: boolean;                                       // number only: a total row may sum it (default false; amount defaults true; percent never sums)
}

export interface CustomFieldDef {
  // ── v1 (unchanged) ─────────────────────────────────────────────────────────────────────
  id: string;                  // the document id (a UUID in v1 rows)
  cid: string;
  objectKind: UdfKind | 'v2only';   // the written primary place (CF16, 1.1): 'v2only' is a sentinel no older app reads; read places through placesOf()
  key: string;                 // v1: slug stored in udf; v2: equal to fid
  label: string;
  type: UdfType;
  options?: string[];          // v1 list options; for v2 kept IN STEP with `opts` (the active labels, in order) so an older app still works
  required?: boolean;          // v1: cosmetic. v2: kept in step with `req` for the old admin table
  order?: number;
  // ── v2 ─────────────────────────────────────────────────────────────────────────────────
  v?: 2;                       // present = this row was written by the new editor; enforcement applies only to v2 rows
  fid?: string;                // 'f' + 7 base36 characters, unique in the company; absent on v1 rows (then fid is key)
  also?: UdfKind[];            // the other places the same field lives
  status?: 'active' | 'hidden' | 'deleted';  // absent = active; 'deleted' is a tombstone (10): never offered anywhere, its label and id kept forever
  scope?: FieldScope;
  opts?: FieldOption[];        // select / multiSelect: authoritative when present
  req?: 'policy' | 'new' | 'always';  // present = mandatory (3). 'policy' (the editor's default) follows Company.entryPolicy for the alteration of an old voucher or record; 'new' = only new entries are checked; 'always' = every alteration is checked (AD.1)
  tracksStock?: boolean;       // a select with the Item line place only: (item, variant) becomes a stock position, like a batch (AD.3)
  walk?: boolean;              // ask in the Enter walk (forced on when req is set)
  dflt?: UdfValue;             // static default, typed per `type`; ignored at entry when it names a hidden or gone option
  inherit?: { party?: boolean; item?: boolean };  // a voucher starts from the party's value; a line starts from the item's
  rules?: FieldRules;
  hint?: string;               // one line shown under the field, at most 80 characters
  use?: { report?: boolean; print?: boolean; copy?: boolean };  // defaults: report true, print false, copy true
  lookup?: { to: 'ledger' | 'stockItem' | 'godown' | 'costCentre' | 'employee'; groupIds?: string[] };
  access?: 'all' | 'admin';    // RESERVED (section 8): admin-only fields
  tallyIndex?: number;         // RESERVED (section 11): the UDF index assigned at the first Tally export, never recomputed
}

// `UDF_KINDS` (the editor's tab list) gains Voucher, Item line, Employee and Lead beside Ledger and Stock Item.
// the carriers (optional; absent on every record written before a field exists, which is how "nothing defined, nothing changes" is asserted)
Voucher.udf?: UdfValues;  InventoryLine.udf?: UdfValues;  Employee.udf?: UdfValues;  Lead.udf?: UdfValues;   // Ledger.udf and StockItem.udf exist
InventoryLine.countPart?: true;   // a Physical Stock line that is one part of a count of the whole item at its godown (AD.3.3); written only by the voucher screen, only for a tracked item; a structure flag, never the variant key
```

**Identity.** Two ids are never confused (this is the trap the audit found: the id the report filter reads is not the id the document has).

| Name | Value | Forever? | Used by |
|---|---|---|---|
| `id` | the document id | yes | the `customFields` table, undo, sync |
| `valueKey(def)` | `def.fid ?? def.key` | **yes** | the key in every `udf` bag |
| `reportFieldId(def, role)` | `'cf.' / 'cfp.' / 'cfi.' / 'cfl.' / 'cfa.'` + `shortKey(valueKey)` | **yes** | report columns, filter conditions, saved views, the grid's layout ids |

`shortKey(k)` is `k` when it has at most 32 characters, else `k.slice(0, 23) + '_' + fnv1a36(k)` (an eight-character hash), a pure function of the
key, so every client agrees; it is at most 32 characters, so with a prefix of at most four every id is at most 36 and **`SAFE_FIELD = /^[A-Za-z0-9_.-]{1,40}$/`
in `core/reportFilter.ts` accepts every id unchanged** (the audit found `udf:ledger:<key>`, planned in the grid research, would have been
dropped by it because of the colon). New definitions get a fid of eight characters, so `shortKey` only ever alters a v1 key of 33 to 43
characters (the v1 slug is cut at 40 and the collision suffix `_2` to `_99` adds up to 3, A item 12f). A property test asserts: every id matches `SAFE_FIELD`;
**within one place no two definitions share a `valueKey`, and within one report no two columns share an id** (a v1 key is unique only within its kind, so a ledger
"Region" and a stock item "Region" both have key `region`: they stay apart because the place and the role prefix differ, `cfp.region` against `cfi.region`, and
`CfCtx` holds one map per place, 5.2); and the id of a definition never changes across `normaliseDef`. Fixtures: the colliding v1 pair, a 43-character key,
`__proto__` and `constructor` as keys and as option ids.

### 1.4 Option lists (select and multiSelect)

```ts
export const OPTION_CAP = 500;       // per field. Protects the picker (type-ahead over a list is O(n) per key) and the document size.
```

| Operation | Rule | Effect on stored data |
|---|---|---|
| Add | an id `'o' + 6 base36 characters`, unique in the field; label cleaned (40 above), unique after NFKC and case folding **against every label AND every id of the field's options, hidden and gone ones included** (a legacy id is a label: if "Red" were renamed Crimson and a new "Red" were then allowed, an older app that writes the label `Red` would be read back as Crimson by step 1 below) | none |
| Rename | the **label** only | none: values hold the id, a saved filter holds the id |
| Hide (deactivate) | `active: false` | old values stay valid and reportable; the option is not offered on new entries, except where it is the current value of the record being altered. **Hiding the option that is `dflt` is refused until the default is cleared or moved**, and a `dflt` that is hidden or gone is ignored at entry |
| Reorder | the order of `opts` | none |
| Make default | `dflt` = the option id | new entries only |
| Delete | offered only when no record on this device holds the id AND the company is fresh (10); it marks the option **`gone: true`** and keeps it in `opts` (listed under "Removed options", never offered): the label survives so a value that arrives from a device that had not synced is still labelled; otherwise "Hide it instead" | none, nothing to lose |
| Paste a block | one label per line, de-duplicated, up to the cap, with a count of those skipped | adds options |
| Preset | Indian states and union territories, months, days, zones (North, South, East, West), priority (High, Medium, Low), sizes (XS to XXL), colours (a dozen common) | adds options |

**Reading a stored value of a choice field** (`readValue`), in this order, so a record written by any version of the app is never lost:
1. the stored string equals an option id: that option (hidden or not);
2. else it equals an option label after NFKC folding (a value an older app wrote as text, or a legacy v1 value): that option, state `legacy`;
3. else the value is kept untouched and shown as typed with the note "(not in the list)"; a filter on the field can still match it by text
   and a report row for it reads "(removed option)" when it looks like a generated id. Step 1 can never capture a label written by an older app, because the Add rule keeps every label distinct from every id.

**Legacy ids.** A v1 definition has `options: string[]` and its values are those labels. `normaliseDef` builds `opts` with **`id` equal to the
legacy label text** (any characters, at most 80, no control or format characters; ids are values, never keys, never put in a pattern, a CSS id or a
DOM id; **every per-option structure, summary row, count and picker index is a `Map`, never a plain object**, because a legacy id is arbitrary label text, `__proto__` and `constructor` included). Stored v1 values are therefore already ids: no data is rewritten, and a rename later changes the label only. New options get generated
ids. The editor writes both `opts` and `options` (the active labels in order); an older app reads `options` and keeps working.

**Option colours: not in v1.** A coloured chip needs a contrast check against both schemes (Phase 27 gotcha: every colour offered is validated
by script for every pair); the `--viz-*` series tokens are the validated set if the owner wants it later. Recorded as a deferral, not an omission.

### 1.5 Caps and text hygiene

Every cap names what it protects (CF13). All lengths are checked **before** any pattern test, and every scan is linear (Phase 24 gotcha: a
quadratic regex froze the whole backend for minutes).

| Cap | Value | Protects |
|---|---|---|
| Definitions per company | 200 | the editor list, the Columns panel, `normaliseDefs` memoisation |
| Per place (defined) | ledger 60, stock item 60, voucher 40, line 12, employee 40, lead 40 | the editor list; the master form length |
| Asked on ONE voucher base | header fields 12; line fields 6, of which at most 2 inline (decided by the measured width, 2.2) | the strip height; the item grid width (a column costs a fixed 110 to 140 px that the text-size setting does not scale). The editor refuses a 13th header field or a 7th line field scoped to one base |
| Options per field | 500 | the picker, the document size |
| Field label | 40 characters | the strip, the column header, the picker |
| Option label | 60 | the same |
| Hint | 80 | the strip height |
| `text` value | 200; `longText` 2,000; `multiSelect` 50 selections | every voucher payload and every change-log row carries the value |
| A `udf` bag | 64 keys and 8 KB serialised | a hostile or corrupt payload cannot make one document a megabyte. **The order is fixed and nothing is truncated silently:** a write that would pass the cap is refused with a message naming the first key over; a read sees the first 64 keys in sorted key order (`udfView`, below) |

**Cleaning order for any typed or stored text** (`cleanDisplayText(raw, max)` in the new `core/customFields.ts`): `typeof` is a string; length
at most 4 times the cap (else rejected, not truncated); NFKC; remove every Unicode format control, the C1 controls and the "filler" letters
(the set `core/uiPrefs.ts` `INVISIBLE` removes; copied, not imported, because `uiPrefs` is Phase 27's, and a property test asserts both strip the
same code points); trim; collapse whitespace for single-line types; **then** the cap; then the pattern preset; then type checks. A name needs a
letter or a digit. `Object.create(null)` is used for every bag read from storage (`__proto__` can never be a key; keys match
`/^[A-Za-z0-9_-]{1,48}$/`, 48 covers the longest v1 key, 43, and others are dropped).

**Where the guards run (stated once).** *Reading:* the pull-apply and every report read the bag through a **view** (`udfView(doc)`, a pure function that never writes back): it ignores a key that fails the key pattern and a value of the wrong shape, and reads at most the first 64 keys. A read never rewrites the stored document, so a pull cannot make the local copy differ from the server's and a later save by a current client cannot persist a sanitised bag by accident. It is O(keys) for one document, not a pass over all rows: a report reads one key per column. *Writing:* `mergeUdf` and `checkUdf` (3) enforce the caps on every write path the app owns and refuse rather than truncate. *The server* validates nothing until CF-6b: a REST client can store a 70-key bag, which readers then see through the view, and the editor's usage count reports "N documents have more than 64 values".

### 1.6 The v1 rows that already exist

There is no customer data on v1 definitions (CLAUDE.md: no customer has used custom fields), so this is a compatibility reader, not a migration
project. It must not break the owner's own test companies and it must never surprise them.

- `normaliseDef(row)` reads a v1 row as a v2 row: `fid` is its `key`, `status` active, `opts` from `options`, no scope (a v1 field applies to
  every ledger or every item), `use.report` true.
- **A v1 `required: true` is NOT enforced** until the administrator opens the new editor, which lists the v1 fields with "Marked required, never
  enforced. Update" and converts them in one confirmed step (`required: true` becomes `req: 'policy'` (follow the company's entry rule, whose default is new entries only, AD.1), `v: 2`; a `number`, `date` or `toggle` field is also moved to the v2 rules and, by CF16, away from the older editors, which is why this is a confirmed step and never automatic). Enforcing a flag nothing ever read
  would refuse saves for a company that ticked it in good faith.
- v1 values are read leniently (1.4 for lists; a date as typed text is parsed as day-month-year or shown as typed with "(not a date)"; a
  number stays a number; an absent yes/no is "unanswered", **not** back-filled to No).
- The first v2 edit of a definition writes `v: 2`, `fid`, `opts` and keeps `options` in step. Nothing else about the row changes.
- **Key collisions across kinds are expected, not an error.** A v1 key is unique only within its kind (`CustomFields.tsx` builds `taken` per kind), so a ledger "Category" and a stock item "Category" both have key `category`. Nothing relies on company-wide uniqueness of a v1 key: values live in the bag of their own document, `CfCtx` holds one map per place (5.2), and report ids carry a role prefix. A new v2 field's `fid` IS unique company-wide (the generator checks every definition, tombstones included). A v1 key of 41 to 43 characters (slug plus collision suffix) is read as is.

---

## 2. Where a field is asked: the entry screens

### 2.1 Voucher header: a strip under Narration (slice CF-2)

**Rule (CF6): the Enter walk of a voucher is frozen, and the strip is appended after its last field.** The strip is `ui/vch/CustomDetails.tsx`, a
new component rendered by `VoucherScreen` immediately after `narrationRow` at all four sites that draw it (the four voucher shapes: accounting,
invoice, stock journal, physical stock; the audit lists them as `{narrationRow}` at four places). It draws **nothing** when no active field
applies to this voucher's base and type (the same constant empty array `NO_FIELDS` every time, so identity is by reference, not by eye).

Let **S** be the fields that apply (active, in scope for this base and type), in `order`; let **W** be those in S that are required or have
`walk: true`, in the same order. The Focus union gains `{ a: 'cfv'; fid: string }`. **W is computed per voucher:** on an alteration where the field does not bind (its mode is `never`, or `since` and the voucher is dated before the policy date, and the voucher never held a value: 3.2, AD.1.3), the field is not in W, so the walk of an old voucher is exactly what it was; **where it does bind (mode `always`, or `since` on or after the date) it is in W**, tagged "required by company rule for existing vouchers" and starting blank (AD.1.5); **when F10 (Change Voucher Type) moves a voucher into a base or type where a field newly applies, that field is in W and is judged as on a new entry** (3.2, `beforeKind`).

| From | Key | To | Notes |
|---|---|---|---|
| Narration | Enter | W empty: `doAccept()`, **exactly as today**; W non-empty: `cfv(W[0])` | an expanded snippet or open blank takes the key first, as today |
| Narration | Tab (no open blank) | the browser's Tab: the first strip input, whose `onFocus` sets `cfv(S[0])` | DOM order does the work; no handler |
| Narration | ArrowDown | `cfv(S[0])` when S is non-empty, else nothing | |
| `cfv(k)` | Enter | commit the field; next is the first `w` in W after `k`; none left: `doAccept()` | a field not in W is skipped by Enter and reached by Tab |
| `cfv(k)` | Tab, Shift+Tab | next or previous field in S; Shift+Tab from `S[0]` returns to Narration; **Tab from the last field of S also returns to Narration** (the cycle stays inside the voucher: Tab on the last field would otherwise fall to the browser and out of the page) | Shift is read off the raw event (`combo()` does not encode it) |
| `cfv(k)` | ArrowUp | previous in S, or Narration from the first | |
| `cfv(k)` | Backspace in an empty text box | the same as ArrowUp | |
| `cfv(k)` | Escape | the voucher's own Esc, unchanged ("Quit ?" for a new entry); an open choice popup closes first | |
| any | Ctrl+A or the rail's Accept | `doAccept(true)`: **refused** with "<label> is required!" if a required field is empty, the cursor moves to it, nothing is saved | the same pattern as "Party A/c name is required!" |
| choice field | a letter, or a click | opens the type-ahead popup. **With no stored value, no default and nothing inherited, no row is highlighted** (`popSel` starts at -1): Enter on a **required** field is refused with "<label> is required!" and the cursor stays; Enter on an **optional** field leaves it blank and moves on; arrows or typing highlight a row and Enter picks it | **not the party's rule** (Enter picks row 0): a habitual Enter, Enter, Enter must never book "North" on an invoice nobody chose a zone for (the "required" check would pass, so no check could catch it); Place of Supply already keeps a blank on Enter |
| yes/no field | Y, N, Space, or the three buttons | sets Yes, No, or toggles; Delete or the blank button clears to unanswered | three states, so "not asked" is never confused with No (T12) |
| any field | Delete (a choice or yes/no) or an empty text, or the **Clear (x)** shown on a filled field | clears the key | |

**With the mouse only (every control has a pointer path; the keyboard walk above is unchanged).** A click on a strip cell focuses it and opens its popup (choice), date box or text box; a filled optional field shows **Clear (x)**; a yes/no field is drawn as three buttons **Yes · No · blank** with the current state pressed, not as a text cell; every picker row is clickable and the popup closes on a click outside. The review found the first draft said only that a *letter* opens the popup, so a mouse-only clerk could tag a voucher only if a click happened to open it, and could never clear a wrong value. `cf-mouse-only` (15) enters a whole voucher with pointer events alone.

**Layout and accessibility.** The strip is a CSS grid of `repeat(auto-fill, minmax(200px, 1fr))` with the label above its box, no scroll region, and at most 12 fields per base (1.5), so its height is bounded (about three rows at 1280 px [J]; `cf-geometry` measures it). Each field is a `label` bound to its input (`htmlFor`), carries `aria-required` when `req` is set, and its hint is the `aria-describedby`. The "from party" and "default" tags are real text in `--dim` and join the contrast matrix of `visual.assert` for both schemes and all eight accents (Phase 27: every pair is validated by script). When a party change recomputes an inherited cell, the new value is announced through the existing status lane (`role="status"`: "Zone is now East, from the party"), never for a cell the person typed.

**Why this respects the frozen walk (the Phase 24 gotcha, both halves):** a field is never inserted before an existing one, and a field never
takes Enter's Accept away from a company that did not ask for it. With nothing required and nothing marked `walk`, Enter on Narration still
raises Accept and the strip is reached only by Tab, Down or a click: the habitual keystrokes mean what they meant. **Making a field mandatory is
the administrator's explicit act, and the definition screen says so beside the switch** ("Mandatory fields are asked after Narration; Enter
walks into them"). Asserted three ways: a source scan of the voucher walk with the new keys removed (as `gst-identity.assert.ts` does for the
ledger walk), a recipe that types a whole sales voucher by the old keystrokes in a company with only an optional field, and the zero-field
identity checks of section 14.

**State of one strip field** (so "a defaulted value must always be visible", Phase 21 gotcha):

```ts
interface CfCell { value: UdfValue | undefined; src: 'stored' | 'typed' | 'party' | 'default' }
```
`stored` = loaded from the voucher being altered; `typed` = a person entered or cleared it; `party` = inherited from the party when it was chosen;
`default` = the field's static default. A cell whose `src` is `party` or `default` is drawn with a dim "from party" or "default" tag beside the
value. Choosing or changing the party recomputes every cell whose `src` is `party` or `default` and never one that is `typed` or `stored`.

**Everything a header field touches inside `VoucherScreen.tsx`** (from the audit, section 3.4, with the wave-3 line numbers drifting): state
initialised from `src` beside the other header `useState` calls; `resetLines` (after Accept: cells return to default or inherited, **not carried**,
so nothing travels invisibly to the next voucher); the dirty `signature()` and `typedNotEntered` (a typed cell is unsaved work: the `navstack.assert`
line that pins the `useDirty` literal is updated in the same change); the `modal` and panel gating; the validation at the top of `doAccept`; the
`const v: Voucher = {` literal (the carry funnel, section 9.1); the duplicate, recall and PgUp/PgDn decisions (2.4); the rail (no button: the strip is
not a panel).

### 2.2 Item lines: a column or a sub-row after Amount (slice CF-3)

A `line`-place field scoped to a voucher base (`scope.line.bases`, optionally `typeIds`) is **a field of that voucher's item rows**: a column after Amount (after Description when F12 turned it on) when the voucher body is wide enough, otherwise a **sub-row** under the item (like Description). It is always shown on a scoped voucher: "applies to the item line" is what turns it on, so there is no second visibility switch (Zoho's "show when creating transactions" is that switch, [W] 11a). At most 6 line fields on one base (1.5).

- **Inline or sub-row (the measured width decides).** Each inline column costs a fixed 110 to 140 px that the text-size setting does not scale (`styles.css`: `qty` and `rate` 110, `amt` 140, `gdn` 150), and the grid's fixed columns already total 650 px, so six inline columns cannot fit at any ordinary window (A, item 12i; the review's measurement of a 650 px table in a 660 px body at 1024 px is not re-measured here). The rule is a formula, not a number: `inlineCount = clamp(floor((bodyWidth - builtInMinWidth - 16) / 120), 0, 2)`, with `bodyWidth` measured by the `useNarrow` ResizeObserver hook (Phase 27: no container query, no CSS `zoom`) and `builtInMinWidth` the sum of the visible built-in columns. The first `inlineCount` line fields (in `order`) are columns of 120 px; the rest draw on a **sub-row** under the item (`<tr class="inv-sub">`, one `colspan` cell holding a label-above-box grid) with the same Enter, Tab and click behaviour. At 1024 px every line field is on the sub-row, so the table's width does not change at all.
- **Walk.** The item row's walk is name, quantity, rate, discount, amount, description. Enter on Amount goes to the next **walk** line field when W-lines is non-empty (required or `walk`), in order, and Enter on the last of them goes to the next row, exactly as Enter on Amount does today. With no walk line field Enter on Amount is unchanged and the field is reached by Tab or a click. **The focus is `{ a: 'item', sec, i, col }` where `col` for a custom field is the string `'cf:' + fid`** (the union `ItemCol` gains the template-literal member `` `cf:${string}` `` and stays a string, so `isF` and `keyOf` keep comparing with `===`; an object-shaped column would have broken every comparison).
- **Tab (an explicit contract; the first draft assumed it).** Today the item grid draws one `<input>` (the focused cell) and `case 'item'` has no Tab branch, so Tab on an item input is the browser's own and lands on Narration. `case 'item'` gains a Tab and Shift+Tab branch **only when the base has at least one line field**, behind a `lineFieldCount > 0` guard, so a company with none runs no new code path (asserted by a source scan and by the identity suite). Tab on Amount (or on Description when shown) moves to the first line field of the row, inline cell or sub-row; Tab inside the line fields moves to the next; Tab on the last line field falls through to the browser's Tab as today; Shift+Tab reverses; Tab on name, godown, quantity, rate and discount is unchanged; Tab never adds a row (Enter does).
- **Mouse.** A click on a line-field cell focuses it and opens its popup; a filled optional cell shows Clear (x); the sub-row needs no hover to appear.
- **Value.** The line starts from the item's value of the same field when `inherit.item` is on (a copy: the line holds its own value from then on, CF10),
  else from `dflt`, else blank. Only a **valid** line (an item chosen) is checked for a required field, never an empty row.
- **Snapshot (T5).** Editing the item's Colour afterwards never re-labels last year's sales: the line's `udf` is its own.
- **Never merged (T6).** Any place that combines lines of one item (the print grouping, the e-invoice item list, the HSN summary) stays a function of
  money and quantity only and ignores `udf`, so a field can never move a statutory figure; `scripts/assert` scans those files for any read of `.udf`.
- **Carried through a rebuild.** `ItemRow` (the grid state), `itemRowFromLine`, `validInvLines` (the line literal) and the stock-journal source and
  destination grids all carry `udf`; the audit's A2 row lists them (and **the server restores a line's `udf` only where the line's identity is unambiguous**, 9.3). The line's identity in the Mismatch Report is unchanged (item, godown, batch):
  two lines that differ only by colour are labelled `(2)` there, which is accepted.
- **A line field labels the stock unless it is marked to track it (AD.3).** An ordinary line field does not change a stock position or its valuation; the engines key on item, godown and batch [C audit 4.5]. A choice field marked **Tracks stock** does: (item, variant) becomes a position exactly as a batch makes (item, batch) one, with costing still per item (CF24, AD.3.1), so "closing stock by Colour" is answered. **The product says which of the two a field is** where the field is defined: a field with the Item line place and no "Tracks stock" carries the sentence "This labels the stock; it does not split it. Tick Tracks stock to keep stock by Colour." (the first draft's "BUSY's parameter feature is the benchmark [R]" is now the fallback for per-variant cost: one item per variant, AD.3.1).

### 2.3 Masters (ledger, stock item; employee and lead in CF-8)

`ui/udf.ts` is rewritten around `core/customFields.ts`. Three behaviours change, all asserted:

1. **Placement.** An optional custom field is a `tail` field of the master form (`FieldSpec.tail`): the walk ends where it always did, Enter on the
   last built-in field raises Accept, and the tail is reached by Tab, a click or Down. (Today custom fields join the walk for any company that defines
   one, which already bends the frozen order: audit finding 6.) A **required** field is a walk field appended after the last built-in walk field: the
   administrator chose it, and the master form's `validate` also refuses Accept with the field named. **On an alteration of an existing master the field is in the walk and refused only where it binds** (`req: 'always'`, or `'policy'` with the company's masters setting at `always`: AD.1.3); otherwise it is a tail field, as an optional one.
2. **Scope.** `FieldsSource` is already a function of the form values, so a field scoped to ledger groups appears when the group chosen on the form is
   under one of `scope.groups`, and a stock-item field when the item's group is under `scope.stockGroups`.
3. **Collect never drops.** `mergeUdf(stored, edits, defs)` starts from the stored `udf` and writes only the keys the form owns: orphaned keys survive
   (T3), `0` and `No` are stored, dates are ISO, choice values are option ids, the bag is capped. `master-resave.assert` gets a new case: an alteration
   whose definition list lacks a key keeps that key. For a **v1 definition** `mergeUdf` uses the v1 profile of 1.1 (the older `collectUdf`'s bytes).

4. **What `forms.tsx` needs (lane E, CF-1; the first draft listed it as optional).** `useTallyForm` picks the highlighted row of any select on Enter (`pickCurrent(popSel)`, `popSel` starts at 0), and a toggle has only Y, N and Space, so the strip rules of 2.1 need three additions, each behind a `FieldSpec` flag that only custom fields set (so no built-in form changes; a source scan asserts the flags appear only in `ui/udf.ts`): `blankOk` (an empty select starts with no highlighted row, Enter leaves it blank or, with `required`, is refused, Delete clears), a three-state toggle (unanswered, Yes, No; Delete clears) and `percent` and `amount` types that reuse the number field's calculator. `udfInitial` stops back-filling `No` for an unanswered toggle in a v2 row (a v1 row keeps today's back-fill, the v1 profile). A master form's click path is the strip's (2.1).

**The Alt+C quick-create overlay** (a ledger made in the middle of a voucher) shows **only required custom fields**, so creating a party mid-voucher
is never blocked by an optional field and never silently skips a mandatory one (audit 1.4 row 9).

### 2.4 Defaults, inheritance, copy, recall, Accept (the per-flow rules)

| Flow | Header field | Line field |
|---|---|---|
| New entry | `dflt`, then `inherit.party` when the party is chosen | `dflt`, then `inherit.item` when the item is chosen |
| Alteration | the stored value; a party change does not overwrite a stored value. **A bound alteration (AD.1.5) starts a never-valued field blank**, never from the party or the default (the party's current value may restate history), and offers "Use North (party now)" | the stored value (a bound line field of a never-valued line likewise starts blank) |
| **Alt+2 duplicate** | copied unless `use.copy` is false; **a value that is a hidden or gone option is NOT copied**: the cell starts blank or at its default, tagged "retired option not copied" (hidden options are not offered on new entries, and a carried value must be visible) | copied unless `use.copy` is false; the same rule |
| **Ghost recall** (a party's last voucher) | **not recalled**: the default and inheritance rules apply, visibly | **not recalled**: the line starts from the item (a recalled value that skipped its prompt would be accepted blind, the Phase 20 to 21 gotcha) |
| After Accept (next blank voucher) | back to default or inherited, not carried | n/a |
| PgUp / PgDn in an alteration | each voucher shows its own stored values | same |
| Credit or debit note made from an invoice, recurring entry (any "create from") | per `use.copy`, with the hidden-option rule above | per `use.copy`, same |
| Cancellation (Alt+X) | **dropped** from the shell (the shell keeps identity, number and the numbering story, and drops lines, party and flags by design; the change log keeps the prior version); the cancel is an **exempt** writer (3.3), so a mandatory field never refuses it | dropped |

**Alt+K hold is not offered for custom fields in v1.** A held value must always be visible (the Phase 21 gotcha), and inheritance from the party already covers "the same zone all day" with no keystroke; a per-field "keep for the next voucher" switch is a later option if the owner asks.

### 2.5 Chord audit (this spec adds no chord)

The strip is reached by Tab, Down or a click; line columns by Tab or a click; the editor's keys are screen-local and reuse existing meanings
(Alt+A add, already the editor's own; Alt+Up and Alt+Down move, already the Dashboard editor's and listed in `RAIL_FIXED`; Esc quits). Nothing is
appended to `core/shortcutRegistry.ts`, so `zoho-scaffold` and `shortcutprefs` are untouched. If the owner later wants a chord for "go to the details
strip", the only unused chords in `src/` are `Alt+F8`, `Alt+F10`, `Alt+F11` (greps in A, item 2); each must be checked on Windows and in Electron
(`Alt+F10` can focus a menu bar in some shells, [unverified]) and appended per the registry's rules 1 to 4.

---

## 3. Mandatory, defaults, validation: one function, every door

### 3.1 The function

```ts
// src/core/customFields.ts (pure: no React, no store, no Dexie; the perf harness refuses an engine that imports any of them)
export type Source = 'form' | 'bulk' | 'import' | 'tally' | 'api' | 'exempt';   // 'exempt' skips THIS check and only this check (3.3)
export interface CheckArgs {
  defs: readonly CustomFieldDef[];
  place: UdfKind;
  base?: VchBase; typeId?: string;                       // voucher and line: the base and type being saved
  groupId?: string; stockGroupId?: string;               // masters
  op: 'create' | 'alter';
  before?: UdfValues; after: UdfValues | undefined;
  /** the effective company entry policy (AD.1): the CALLER passes it, the engine never reads storage. Omitted = DEFAULT_ENTRY_POLICY, the prospective rule */
  policy?: EntryPolicy;
  /** voucher and line places, alter only: the stored date and the date being saved, for the policy's `since` mode (AD.1.3) */
  dates?: { stored?: string; next?: string };
  /** alter only: the base and type the STORED voucher had, so a field that newly applies after F10 (Change Voucher Type) is judged as on a new entry */
  beforeKind?: { base: VchBase; typeId?: string };
  source: Source;
}
export interface Problem { fid: string; label: string; code: 'required' | 'type' | 'option' | 'range' | 'pattern' | 'length'; message: string; severity: 'refuse' | 'warn' }
export function checkUdf(a: CheckArgs): Problem[];       // [] when nothing applies (the NO_FIELDS fast path)
```

### 3.2 What it decides (prospective by default, T1; an administrator may bind old records, AD.1)

| Question | Rule |
|---|---|
| Which fields are checked | active v2 fields in scope for `place`, `base`, `typeId`, group |
| `req` present (any value) | **create:** an empty field is refused. **alter:** decided by `modeFor(def, place, policy)` (AD.1.3). Mode `never`: refused only when `before` held a value and `after` cleared it (a value cannot be removed), never when the record never had one. Mode `always`: an empty field is refused, so an old voucher must be completed to be edited. Mode `since`: refused when the voucher's stored or new date is on or after the policy date. **A field that does not apply to the stored voucher's base and type but applies to the new one (F10, `beforeKind`) is judged as on a create**, so turning a Journal into a Sales cannot dodge a mandatory Zone |
| `req: 'policy'` (the default), `'new'`, `'always'` | which mode an alteration gets: `'policy'` follows `Company.entryPolicy` (default `never`, so the owner's answer to question 2 (2026-10-22) is a setting, not a rule), `'new'` is always `never`, `'always'` is always `always` (AD.1.1). **A binding alteration is enforced by the entry screens' own validation (the voucher's `doAccept`, the master form's `validate`), not by the store backstop** (3.3) |
| Types | a wrong type, an id not in the list (and not a legacy label), a number outside `min` and `max`, a text over `maxLen` or failing its preset: refuse. A value that is a hidden option is allowed. **On an `alter` only a value that CHANGED is judged: a value equal to the stored one is never re-judged**, so an old record with an unknown or out-of-range value can still be re-saved by a cheque, BRS or undo writer |
| Warnings | none in v1 beyond "(not in the list)" shown for an unknown stored value |
| Defaults | a `dflt` that is hidden or gone is ignored at entry (the cell starts blank), and the editor refuses to hide it without clearing it (1.4) |
| v1 rows | never enforced (1.6) |

### 3.3 Which door asks, and how hard

**Two things the first draft conflated are now separate.** `{ system: true }` (on `putDoc`, `deleteDoc`, `saveDoc`) skips the **operator write guard** (`checkWrite`: base rights, the date window) and stamps **no user**, so the write is not an undo step and is not counted in Today; it exists for the app's own housekeeping (A, item 12c). Exempting a write from the **mandatory-field check** is a different decision and gets a different option: **`{ udf: 'exempt' }`**, accepted by `saveDoc` and `saveVoucher`, which skips `checkUdf` and nothing else: the write guard still judges the write and the operator stamp is still written. An operator who may not write in a closed period is therefore still refused when cancelling there, and a Ctrl+Z or a cancel carries the operator's name.

**The store derives op and before; the caller never states them.** `saveVoucher` and `saveDoc` read the stored row (`get().vouchers`, or the collection array) and set `op = stored ? 'alter' : 'create'`, `before = stored.udf`, `beforeKind = the stored base and type`. Ctrl+Z of a **deleted** voucher finds no stored row and would be judged a `create`, so the undo applier and the restore path pass `{ udf: 'exempt' }`: **a field that became mandatory after a voucher was written can never keep that voucher from being brought back.**

**The refusal contract.** `checkUdf` returns `Problem[]`. The **store** backstop turns a non-empty `refuse` list into a thrown typed error, `UdfRefusal extends Error { problems: Problem[] }` (`saveDoc` and `saveVoucher` stay `Promise<void>`). Form callers call `checkUdf` first and never see the throw; **every non-form caller catches `UdfRefusal` and raises the first problem through the message lane** (`raise`, the lane `deleteVoucher` uses for the IRN block), so a cheque layout or an importer that cannot point at a field still says why and nothing is half saved. **The store judges less than the form:** only `req` on a `create`, and the type, option and range of a value that CHANGED (3.2). A binding alteration (a field at `always`, or at `policy` when the company setting binds: AD.1) is the entry screens' job, so a cheque screen resaving an old party that has no Zone is not refused, **whatever the company's entry policy says** (and a ledger-place field with no `scope` applies to every ledger, so an unscoped `always` field would otherwise refuse every non-form ledger write).

| Writer (every `saveVoucher` and `saveDoc` caller is named; a `putDoc` caller is below the store and is never checked) | Source and flag | Required | Type, option, range |
|---|---|---|---|
| `VoucherScreen.save` (Accept, Ctrl+A, the rail's Accept), every base the screen draws | `form` | refuse; cursor to the field | refuse |
| Master forms through `useTallyForm` `validate` (ledger, stock item; employee and lead in CF-8) | `form` | refuse | refuse |
| **Bulk Entry** (`BulkEntry.tsx`, per row; custom columns `cf.<fid>` are columns of its paste) | `bulk` | **same severity as the voucher screen** (its own rule 1: "not a validation bypass") | refuse the row |
| **CSV Import Data and the mapped importer** (`ImportData.tsx`, `MappedImport`; custom fields offered as mapping targets) | `import` | **not enforced for vouchers: imported vouchers are history.** The preview counts "N rows lack Zone" and imports them; an "Enforce mandatory fields" tick (default off) refuses them as rows. **Master imports and Bulk Entry always enforce** | refuse the row (an invalid value is a row problem) |
| Tally XML import | `tally` | not enforced (history); the preview counts "N vouchers lack <field>" | a value that is not valid is dropped and counted |
| REST `/v1` | `api` | **not enforced in v1** (the server holds no definitions: `field_defs` was never created, audit finding 11); documented. CF-6b adds a server read of the current definitions and enforces on create | the same |
| Alt+2 duplicate, "create from" | `form` | at Accept, as new | as above |
| Cancellation shell (`useCancelVoucher`), the Ctrl+Z and restore applier (`overlays.tsx`), Salary Disbursement (`disbursement.tsx`: a Payment the system writes), Manufacturing Journal (`manufacturing.tsx`: it asks no custom field in v1; the editor warns when a required field is scoped to Stock Journal), Attendance and Payroll vouchers (`payroll/vouchers.tsx`: not offered as a scope, so `checkUdf` takes its fast path), RCM and ITC journals, the sample company seed, Split Company Data, numbering sync writes (`updateDoc`) | `exempt` (`{ udf: 'exempt' }` where the call goes through the store) | **exempt** | the shape is still sanitised by `mergeUdf` |
| Cheque layout and cheque book saves (`cheque.tsx`, `chequebooks.tsx`: `saveDoc` of a spread ledger) | `exempt` | exempt | not re-judged (the value is unchanged, 3.2) |
| BRS, mark optional, GSTR-1 export info, advances (`putDoc` below the store) | none | not applicable | not applicable |
| **Fill blanks** | below the store (`putDoc` inside its own transaction, 10) | not applicable: it only ever sets a value | the option must exist; nothing else is written |
| Sync pull | none | not a write | read through `udfView` (1.5) |

`data/writeGuard.ts`'s rights check is unchanged and still applies to everything above except a `system: true` caller. A guard that gates only the commit lets another door through (Phase 20 to 21 gotcha), so the suite **enumerates the callers by a source scan of `saveVoucher(` and `saveDoc(`, classifies each by this table, and fails on a caller that is in no row**; and a test proves that cancel, undo, delete-undo and restore all succeed while a mandatory field is defined, and that an operator locked out of a period is still refused when cancelling there.

---

## 4. Where a field is visible, and the owner's question answered

### 4.1 The matrix: one field, each surface, which control, which default

The owner wrote "define ... where it should be visible". Zoho spreads that over four controls and none of them is "reports" [R zoho-custom-fields 0.2 item 3];
Pipedrive's best idea is one field with a short list of places, each a switch [R tally-udf 3.9; W 11b]. Ours, per field:

| Surface | Controlled by | Default | Notes |
|---|---|---|---|
| **Asked on the entry screen** | the places (`objectKind` plus `also`), `scope`, `status` | the places ticked at creation | a field outside a voucher's scope is not drawn there; hidden fields are not asked |
| **Printed voucher** | `use.print`, plus the template's "Custom fields" part (`PrintPart 'custom'`) | **off** | header fields print as a "Details" line, line fields as a sub-line under the item (11); the Print Configuration form shows the part only when some field has Print on (11) |
| **Report column** | `use.report` ("Available in reports") | **available**; **never shown by default** | offered in the Columns panel of every report whose rows reach the object (5.4) |
| **Report filter** | `use.report` | available | `cf.<key>` ids (CF9); the filter's "Match nothing on a stale id" rule (4.4) |
| **Summary, Income and expense, Bills, Balances and Stock by field** | groupable type (choice, yes/no; lookup to a ledger later; text, date and number through banded dimensions, CF-4c) and `use.report` | available | no separate "use for analysis" switch (6, AD.2) |
| **Saved views** | carries the ids in its layout and filter | n/a | stale ids degrade safely (4.4) |
| **CSV, Excel, Print of a report** | whatever columns are displayed | follows the columns | the report tools read the visible table, so a shown custom column exports for free; a hidden one does not |
| **REST, webhooks, backup file** | always: the values ride on the document | always | cannot be switched off; `access: 'admin'` (reserved, 8) would not hide it from the API either, and the document says so |
| **Dashboard** | not in v1 | n/a | view widgets wait for `listQuery` (wave-a2 plan C3) |
| **A GST return, an e-invoice, an e-way bill** | never | never | CF14, asserted by a source scan (11) |

### 4.2 The question: when a field is added on a voucher type, is it in that type's reports automatically?

**Answer (CF3): available automatically, shown never unless chosen.** Evidence for each half:
- *Why available at once.* Zoho never made this true and it is the oldest complaint in its forum (a "Stock#" field that cannot be put on the sales
  reports; a customer field missing from Payments Received, fixed with an export and a VLOOKUP) [R zoho-custom-fields 2.8 to 2.9, 2.14]; Pipedrive
  offers a new field in the column picker the moment it exists [R pipedrive-list-ux 1.2; W 11b]; QuickBooks and Zoho show that a field kept out of
  the statements frustrates [R tally-udf 6.4]. A field you can enter but cannot report is the Tally switcher's reason to call a partner.
- *Why never switched on.* No competitor does it, and it would change every report of a company that merely defined a field, so "a company with no
  custom fields sees exactly today's app" would become "a company with one field sees every register change" (Phase 27 gotcha: customisation is opt-in,
  asserted by identity). A company that wants a column on everyone's standard layout makes a **company view** carrying it (`Company.reportViews`
  already exists, administrator-written).
- *The one control* is "Available in reports" (`use.report`, default on). Turning it off removes the field from every picker, filter and summary
  for new use; a saved view that already names it keeps working until it is deleted, because history stays reportable.

**What the identity claim covers (CF3, narrowed in the second pass).** For a company that defines no field, **the entry screens, the saved documents, the printed voucher (the Print Configuration form included), the standard report layouts and Go To's ranking are unchanged, asserted by identity** (14). **Navigation additions are made on purpose and are visible to every company** (the addendum widened the list, AD.7): the left menu's "Custom Fields" row (66 rows, as Phase 27's "Preferences" row took it from 64 to 65), the `+` entry "New custom field" that row resolves to in the universal `+` registry, **five Reports Center rows in a new "Analysis by field" group** (Summary by field, Income and expense by field, Bills by field, Balances by field, Stock by field; the last gated by the company's inventory feature), late Go To commands, and **a new Settings > Workspace panel, "Custom-field entry rules" (AD.1.5), which also carries the "Custom fields..." link**. Where the registry already gates by right, the editor-related ones are shown only to a holder of the Administration right.

### 4.3 How a field behaves where it does not apply

| Situation | Behaviour |
|---|---|
| A field scoped to Sales and Credit Note, opened in the **Day Book** (which mixes types) | the column is offered; the cell is **blank** for a Journal, Payment or Purchase **that holds no value** (the type is not in scope) and blank for a Sales voucher that has no value; **a voucher that holds a value shows it whatever its type is scoped to** (CF18). The picker row's sub-label reads "asked on Sales, Credit Note" so a blank is never misread |
| ... in the **Sales Register** flat level | the column is offered; a Sales voucher with no value is blank |
| ... in the **Purchase Register** | **not offered**: no voucher of that report can carry the field (the catalogue lists only what a report can compute) |
| A **voucher field on Bills Receivable or Payable** | **not offered in v1**: a bill carries no voucher id (`BillState` has `ledgerId, name, openedOn, dueDate, billAmount, balance, onAccount` [C `core/billwise.ts`]); the party's current value (`cfp.`) is offered instead. The originating-voucher index is wave-a2's A3 item; the voucher field joins Bills when it lands |
| A field **removed from a type's scope** (Zone no longer asked on Credit Notes) | stays offered, stays filled and **stays reported under its stored value** for the vouchers that hold one (CF18): the scope decides entry, never history, so "Profit by Zone" for a closed year does not move when a scope is edited. The summary's Not asked bucket only ever holds vouchers with no value |
| A **hidden** (inactive) field | not asked; **still offered** in the picker under "Hidden fields" and still resolving in saved views, so last year stays reportable |
| A **deleted** field (a tombstone, 10) | its column is dropped from layouts on load (dropping a column never changes a row or a total); its filter conditions are **kept and match nothing** (dropping one would widen an AND, the dangerous direction); it still resolves labels for a late-arriving value and a filter chip |

### 4.4 Saved views, stale ids, mixed client versions

- The grid's `ListLayout.ids` carries `cf.*` ids; `resolveColumns` drops an id the report no longer offers (safe). A stored filter condition on a missing
  field or option is **kept and matches nothing**, and its chip reads "<label> (not in this report)" or "<label> (removed option)". **`describeCondition` cannot say that today** [C `core/reportFilter.ts`]: for a field it has no metadata for it prints the raw id (`cf.fz8q1m4k (not in this report)`), and for an option it does not know it prints the raw option id. CF-4b therefore gives `describeCondition` and `describeNode` a **label resolver** parameter, `resolve(field, value) => { fieldLabel, optionLabel? } | undefined`, supplied from the definitions **including hidden, gone and deleted ones** (a tombstone keeps its labels, 10); `core/reportFilter.ts` joins lane R for that slice (16).
- A `select` condition stores `{ field: 'cf.f8k3m2qa', kind: 'enum', op: 'is', value: '<option id>' }`. The sanitiser does not check the value against the
  options (it never did, which is why per-company lists work at all [C `cleanCondition`]).
- An older app reading a view with a `layout` ignores it and shows Standard (wave-a2 contract C5); one reading a condition it does not know keeps it and
  matches nothing once the **fail-closed unknown-operator reader** (wave-a2 A2-6) has shipped. That reader is a precondition of CF-4b, not something this
  spec can assume: if it has not shipped, CF-4b may use only the operators already in the engine (`is`, `isNot`, text, number, date), and "Not set"
  drill-downs wait for `isEmpty` (5.5).

---

## 5. Reports: the integration contract

### 5.1 Ids (CF9, the "ids are forever" rule of the grid plan, C1)

| Role | Id | Reads | Truth |
|---|---|---|---|
| `own` | `cf.<shortKey>` | the report row's own `udf` (a voucher's header value, or a master row's) | posting |
| `party` | `cfp.<shortKey>` | the voucher's party ledger's `udf` | **now**, labelled "(current)" |
| `item` | `cfi.<shortKey>` | the item master's `udf` | **now**, labelled "(current)" |
| `line` | `cfl.<shortKey>` | a projection of the voucher's line values | posting |
| `acct` | `cfa.<shortKey>` | the ledger an entry posts to, or the ledger a balance is of (AD.2.1) | **now**, labelled "(current)" |

The `now` marker is the grid plan's catalogue rule (a `truth: 'now'` column whose label lacks "(current)" fails the catalogue test): a party's Zone
read next year must not be mistaken for the Zone on last year's invoice.

**A name clash to keep straight.** The grid plan's typed helper `fid(family, id)` (its contract C1) returns a report-field id only if the family's catalogue offers it with the kind the caller expects; this document's `CustomFieldDef.fid` is the **definition's own stable key** (1.3). They are different things that share a word. The grid's helper accepts the `reportFieldId(def, role)` values that `customColumns` registers, and nothing else.

### 5.2 What the grid imports from this spec (`src/core/customFieldColumns.ts`, pure; the grid's `columnSets/*` import it, never the reverse)

```ts
import type { FieldKind } from './reportFilter';
export type CfFamily = 'voucher' | 'bill' | 'itemRow' | 'lineRow';     // the report's row grain

export interface CfColumn {
  id: string;                          // reportFieldId(def, role), at most 40 characters
  fid: string;                         // valueKey(def)
  label: string;                       // "Zone", "Party Zone (current)", "Colour (items)"
  sub?: string;                        // picker sub-label: where the field is asked ("Sales, Credit Note")
  group: 'custom' | 'party' | 'item';  // the picker section: "Your custom fields" / "Party" / "Items"
  role: 'own' | 'party' | 'item' | 'line' | 'acct';
  kind: FieldKind;                     // text | number | date | enum (yes/no and choice are enum)
  options?: readonly { v: string; label: string }[];   // v = option id; hidden options included, labelled "(hidden)"
  money?: boolean;                     // amount fields
  truth: 'posting' | 'now';
  /** 'sum' ONLY for an own amount, or an own number with `rules.additive` on (default false), on a voucher-grain report, or the same on a line-grain
   *  report. A percent never sums, and neither does a number not marked additive (credit days, a rating, a distance: a footer of 4,320 "credit days"
   *  is plausible and wrong, the grid plan's R2). A relation column and a projection are 'none': a blank cell in a total row, never 0. */
  total: 'sum' | 'none';
  hidden: boolean;                     // status 'hidden' (still offered, grouped last)
  get: (row: unknown, x: CfCtx) => string | number | null;   // filter and sort value; an option id for enum
  text: (row: unknown, x: CfCtx) => string;                   // what is drawn and exported: the option's CURRENT label
  sortKey: (row: unknown, x: CfCtx) => string | number | null; // CF28: a choice by its label, a number by value, a date by ISO date, text folded; null (no value) sorts LAST in both directions
}
export interface CfCtx {
  ledgersById: Map<string, Ledger>; stockById: Map<string, StockItem>;
  /** ONE map per place, keyed by valueKey: a v1 key repeats across kinds (1.6), so a single map would hand a ledger field's options to a stock item field's column */
  defs: Readonly<Record<UdfKind, ReadonlyMap<string, CustomFieldDef>>>;
}
export function customColumns(family: CfFamily, defs: readonly CustomFieldDef[]): CfColumn[];
```

A `CfColumn` is a `ColumnDef` of the grid plan (`pipedrive-list-ux.md` 5.1) in everything but name; B-grid's `columnSets/*` wraps it. **Fixed by this contract:** (1) the
catalogue never lists a column the report cannot compute (5.4); (2) a `select` column's `text` is the label **read at render time**, so a rename shows
everywhere at once while the stored id never moves; (3) a hidden or unknown option still has text ("(hidden)", "(removed option)"); (4) `get` is O(1).

### 5.3 Filter kinds and operators

| Field type | `kind` | Operators in v1 (engine today) | After the fail-closed reader ships (wave-a2 D10) |
|---|---|---|---|
| text, email, phone, url, multiSelect (joined labels) | text | contains, not contains, starts with, equals | + is empty, is not empty |
| number, percent, amount | number | equals, not equals, greater, at least, less, at most, between (a row with no figure matches none of them, **including** "not equals" [C `compileCondition`]) | + is empty |
| date | date | before, after, between | + is empty |
| yes/no, choice | enum | is, is not (`isNot` **does** match a row with no value [C: `val(r) !== want`, null reads as `''`]) | + is empty, is any of |

So "invoices with no Zone" needs `isEmpty` (or `isNot` of every option, which the picker does not offer): **that is the one reason the fail-closed reader
is a precondition of the "Not set" drill-down**. Yes/No: `is Yes` matches an explicit true, `is No` an explicit false; an unanswered voucher matches
neither, which is the point of three states (T12).

### 5.4 Which report offers which role (what is computable; nothing else is listed)

| Report (row grain) | `own` | `party` | `item` | `line` |
|---|---|---|---|---|
| Day Book, the voucher registers' flat level, Memorandum / Optional / Post-dated registers (voucher) | voucher fields: blank outside the scope only when the voucher holds no value (CF18) | the party ledger's fields (current) via `partyOf(v)` (AD.2.1: `partyId` for the invoice bases, else the one party ledger among the entries, "Multiple" for several, blank for none, so an imported and a hand-entered Payment read alike) | no | the **projection** (below) |
| Ledger Vouchers (voucher, as one ledger sees it) | voucher fields | none (the relation is ambiguous: A2 plan 5.2 row 4) | no | projection |
| Bills Receivable / Payable (bill) | **not in v1** (no voucher id on a bill) | party fields | no | no |
| Stock Item Vouchers and the item-wise register (line grain, wave-a2 A3) | the voucher's header fields | none | item master fields (current) | the **line's own value** (a real column here, not a projection) |
| Stock Summary (item) | none | none | item fields: a choice or yes/no field becomes a **group-by mode** (like godown), others a column | no |
| Parties and Items lists (new screens, wave-a2 A3) | the master's own fields | n/a | n/a | n/a |
| Statements, GST, TDS, payroll | not offered | | | |

**Added by the addendum (AD.2.8):** Bills and, later, the Parties list also offer the ledger's own fields as `cfa.`; the Stock Summary's group mode and the Stock by field screen read `cfi.` and a tracking `cfl.` (AD.3.8); the five analysis screens are not listing reports and take a `Dimension` (AD.2.2). The filter lists of the first four rows are company-aware from **CF-4b-lite**; columns and sort arrive with the grid (CF-4b).

**The line projection** (a voucher-grain report cannot repeat a row per line, the grid's one-to-many rule): the column shows the one value when every
valued line agrees, **"Multiple"** when they differ, and is blank when no line has a value. Its filter is the ordinary `is`, which therefore matches only
**single-valued** vouchers; "invoices with any Red line" is "Any item's Colour is Red", an `anyOf`-over-lines condition that needs the V2 operators and is
the item-wise register's job. This limit is said in the picker, not discovered.

### 5.5 Summary by field ("Sales by Zone"), `core/fieldSummary.ts` and the screen `fieldSummary`

> **Generalised by AD.2.5 (CF21).** The rules below hold for **every** voucher family. What AD.2.5 replaces: the `Measure`, `SummaryRow`, `Summary` and `fieldSummary` signatures in the code block (families, more measures, `RowKind`, a `Dimension` instead of `fid` and `place`, `qtyByUnit`, `legsFromParty`), the screen parameters (now `{ s: 'fieldSummary'; fid?: string; family?: FamilyId; bases?: VchBase[]; measure?: Measure }`) and the sign sentence under "Bases and sign". The row, grain, reconciliation and drill rules stand.

A new report screen (checklist in CLAUDE.md: `Screen` union, router, crumb, `SCREEN_AREA.fieldSummary = 'reports'`, a Reports Center catalogue row "Analysis by
field", a Go To late command; no rail row, so it goes on `EXCLUDED_REPORT_SCREENS` only if it is not in the catalogue, and it will be). Parameters
`{ s: 'fieldSummary'; fid: string; place: 'voucher' | 'line'; bases?: VchBase[] }` (superseded, see the note above); the field is chosen inside the screen, so there is one catalogue row, not one per field.

```ts
export type Measure = 'invoice' | 'taxable' | 'count' | { sum: string /* fid of a number or amount field */ };
export interface SummaryRow {
  kind: 'option' | 'removed' | 'unset' | 'notAsked';
  optionId: string | null; label: string;
  count: number; amount: number; share: number;           // amount is SIGNED by base (below); share of the total, 0 to 1
}
export interface Summary {
  rows: SummaryRow[]; total: { count: number; amount: number };
  countOverlaps: boolean;     // true at line grain: a voucher with several valued lines is counted under each option, and total.count is the DISTINCT vouchers
  basis: string; measure: Measure;
}
export const baseSign = (b: VchBase): 1 | -1 => (b === 'CreditNote' || b === 'DebitNote' ? -1 : 1);   // a return carries the negative of its sale or purchase
export function fieldSummary(a: { defs; fid; place; bases; from; to; measure; vouchers; ledgers; showZero }): Summary;
```

**Bases and sign.** A summary may span bases **only inside one family**: Sales with Credit Note, or Purchase with Debit Note, or any single base (the picker offers exactly those groupings, never Sales with Purchase). **In `net` mode (a base chosen together with its notes, AD.2.5)** every amount is multiplied by `baseSign(v.base)`, so a 10,000 return of Red shirts reads -10,000 under Red, as R5 reads a credit note in the P&L report; a note alone, and every other family, is `asRegister` (unsigned, equal to the register's own figures). `InventoryLine.amount` is positive on every base (direction comes from the base), which is why the sign is applied here and not assumed.

| Rule | Detail |
|---|---|
| Rows | the field's options in list order (hidden or gone ones only if they hold values), then "(removed option)" rows for ids not in the list, then **Not set** (in scope, no value), then **Not asked on this voucher type** (**a voucher of a base the field is scoped away from AND holding no value**, only when the summary spans bases). **A voucher holding a value is bucketed by the value first** (CF18) |
| Rows come from | `registerRows(vouchers, base, from, to)` (base, period, never optional) with the same guard as the register, so the total equals the register's |
| Header grain, `invoice` measure | `baseSign(v.base) * registerAmount(v, ledgersById)` per voucher: the GST-inclusive invoice value (the Dashboard's Sales card opens the Group Summary instead because register totals include tax: Phase 28 gotcha, so **the basis is printed in the sub-title**: "Invoice value, taxes included") |
| Header grain, `taxable` | the invoice's taxable value from `voucherGstSummary` |
| Line grain | `baseSign * Σ InventoryLine.amount` per option (after line discount, **before tax**, labelled "Item value, before tax"); offered for the four bases whose lines are value (Sales, Credit Note, Purchase, Debit Note). **Amounts partition exactly:** each line belongs to one bucket (its option, Removed, or Not set), so the option rows plus Removed plus Not set equal the signed sum of the lines of the vouchers in scope, to the paisa; it reconciles to the sum of lines, not to the register. **`count` overlaps, and the screen says so:** a voucher with Red and Blue lines is counted under each ("vouchers having a line of this option"), so the option rows do not sum to the total, and **the Total row's count is the number of DISTINCT vouchers** (printed: "A voucher with several options is counted under each") |
| **Reconciliation (CF11), per measure** | **amount measures** (header or line grain): option rows plus Removed plus Not set plus Not asked equal the **signed** unfiltered total to the paisa; **count at header grain**: the rows sum to the number of vouchers; **count at line grain**: the Total row equals the distinct vouchers and the rows may exceed it. The suite proves each on the sample company and a 50,000-voucher synthetic company, and these deliberate mutations are killed: drop the Not set row, treat a credit note as positive, sum the line-grain count rows into the Total, bucket by scope before value |
| Multi-valued types | not groupable, so they cannot appear in the picker for this screen |
| Drill | **Enter on a row, a double-click, or the row's visible "Open" button** (a mouse-only person must be able to open the North drill: Group Summary rows have no double-click today, and the back spec fixes that report separately) opens the register's flat level (`{ s: 'register', base, all: true }`) with the period and a live filter `cf.<key> is <id>` written through the existing filter bridge (`writeReportFilter`, C4 of the wave-a2 plan), so the filter and period are **visible on arrival** with a reset chip; "Not set" needs `isEmpty` and is a disabled row with a tooltip until the V2 operators ship; line-grain rows drill to the item-wise register (A3), disabled until it exists |
| F12 options | measure; show zero rows; sort by list order or by amount; scale (the existing `fmtScaled`) |
| Optional vouchers | excluded by the register guard; the suite asserts it with and without a filter |
| Print, Export | the report shell reads the table, as every report does |

### 5.6 Colour, in three readings, and the honest limit (revised by AD.3)

1. *Item attribute* ("this kurta is Blue"): Colour is a `stockItem` choice. Offered as an item column on the Stock Item Vouchers and item-wise reports (`cfi.`, current), and as a group-by mode on **Stock Summary** and a reading of **Stock by field**: whole items are grouped, every figure is exact (AD.3.8; `stockSummary.ts` groups by stock group, category or godown today, and a flat group mode by a choice field is the same shape as the category tree [C audit 4.5]).
2. *Per-sale choice* ("this shirt, sold in red, blue and black on one invoice"): Colour is also on `line`; the value is the line's own, and **Summary by field** at line grain reads `Σ item value by Colour`.
3. *Stock by Colour* ("how many Red shirts are left"): the line field is marked **Tracks stock** (CF24). (Item, Colour) becomes a stock position like a batch; **Stock by field** reads it, the voucher asks the colour on every stock-affecting line, a sale of a Red shirt with none in stock warns, and Physical Stock counts per colour (AD.3).
**The limit that remains, said in the product:** a variant's quantity is exact, its **value** is the item's value shared by quantity (costing is per item, CF24), so where cost differs by colour the allocated value is an approximation, and the honest alternatives are one item per colour or the gated per-variant costing. A field with the Item line place and no "Tracks stock" **labels** the stock and does not split it; the definition screen says so, and CF-3's exit recipe reads the sentence.

---

## 6. Reporting tags: the decision

### 6.1 Three options, and why the third

| Option | What it is | Verdict |
|---|---|---|
| A new **Tag** master beside custom fields | Zoho's shape: a company dimension with its own list, attached to record types | **No.** It duplicates the option list, the type, the mandatory flag, the default and the "where visible" settings a field already needs for the owner's own examples, and puts two screens in front of a Tally customer who asked for "add a column". Zoho's own workaround, before it built mandatory tags (an undated post, about May 2023 by its image names), was a required dropdown field plus a function that copied it into a tag [R zoho-reporting-tags 0, finding 6] |
| **Stretch cost categories** into tags | a label is a category whose every allocation is one centre at 100% | **No.** A cost centre is an amount split on an accounting entry, it round-trips through Tally XML (`CATEGORYALLOCATIONS`), and a Tally partner already speaks it. Using it for an item's colour or a party default blurs that meaning and risks the round trip. Imported Tally categories used as labels stay what they are |
| **A groupable single-valued field** | a choice, yes/no or lookup field on the voucher header or the item line, with analysis reports that group by it | **Yes.** It covers the owner's three examples (Zone on a header, Colour on a line, a profitability tag across several voucher types), grows the object that already exists (definitions, sync, editor), and needs no second master |

### 6.2 Zoho's tag, feature by feature, against this design

| Zoho reporting tag [R zoho-reporting-tags 1.2] | Here |
|---|---|
| Created once, attached to many record types | one definition, `scope.vch.bases` of any length (1.1) |
| Header or line level, per module | place `voucher` or `line` |
| Attached to items, customers, vendors as a **default** that copies onto new lines | `inherit.party` and `inherit.item`, copied at entry and shown "from party" (2.1, CF10) |
| Mandatory | `req` |
| A default option | `dflt` |
| Nested options up to five levels | **deferred** (a flat list; a hierarchy needs a roll-up rule in every report: 6.7) |
| Conditional option lists, access control by option | **deferred / reserved** (`access`, section 8) |
| Filter and compare in reports (P&L, Balance Sheet, Trial Balance, Cash Flow) | filter everywhere (5); **P&L by field over five sources** (6.4, AD.2.6); Balance Sheet and Trial Balance **not** offered (6.7); the balances of a ledger's OWN field are (Balances by field, AD.2.7) |
| An "untagged" bucket | **always present**, two distinct rows (6.3), which Zoho's pages do not describe [W 11c; R] |
| Plan caps (10 tags, 500 options) | none (CF13) |

### 6.3 The attribution rule and what "reconciling to the books" means (`core/fieldAttribution.ts`, pure)

> **Generalised by AD.2.6 (CF22).** R1 to R6 below stand. R7 (party source), R8 (item source), R9 (account source) and R10 (the Change in stock line) are added there, and `attributeEntry` takes a `Dimension` (AD.2.2) in place of `place: 'voucher' | 'line'` (its bucket type gains the `RowKind` rows `noParty` and `mixed`).

A P&L is built from ledger **postings**, not vouchers. "Profit by Zone" therefore needs a rule saying which part of which posting belongs to which option, and a
test that the parts add up. This is the question neither Zoho's pages nor Tally's answer [R zoho-reporting-tags 0 finding 5, 8.3]; Tally's own job-costing guide says an unallocated entry is posted to its ledger but
is missing from the cost-centre reports, so job cost reads too low [R].

```ts
export type Bucket =
  | { t: 'option'; id: string }      // an option id (a removed id keeps its raw value and is labelled "(removed option)")
  | { t: 'unset' }                   // the field applies and nothing was entered
  | { t: 'notAsked' }                // the voucher's type is outside the field's scope AND it holds no value
  | { t: 'charge' };                 // line place only: a whole-invoice entry (freight, discount, round off)
export interface Share { bucket: Bucket; amount: number }   // amount: positive, in the entry's own P&L sign convention
export function attributeEntry(v: Voucher, entryIndex: number, def: CustomFieldDef, place: 'voucher' | 'line',
                               groups: Map<string, Group>, ledgers: Map<string, Ledger>): Share[];
```

| # | Rule |
|---|---|
| R1 | A voucher is counted only if `postsToLedgers(v)` holds (not optional, memorandum, cancelled; post-dated only from its date) and it is dated in the window: the same guard `core/costCentreTree.ts` and `ui/reports/costpl.tsx` use |
| R2 | **Only entries on ledgers under the six reserved statement groups are attributed** (`sales`, `directinc`, `indirectinc` as income, `purchase`, `directexp`, `indirectexp` as expense: the subtrees `plFigures`' `moveFromTotals` reads, `data/defaults.ts` slugs), **not every entry whose group `nature` is Income or Expenses**: the tie target is the Profit and Loss nett, which counts only those six subtrees. An Income- or Expenses-nature ledger outside them (a user-created primary group) is shown as the memo row "Outside the statement" and is not in nett (AD.2.6, R-rules carried); Assets and Liabilities entries are not part of a P&L (as in the Cost-Centre P&L) |
| R3 | **Header field:** every counted P&L entry of the voucher takes the voucher's value: its option (or `removed` for an id not in the list), **whatever the voucher's type is scoped to today (CF18)**; with no value, `unset` when the type is in `scope.vch`, else `notAsked` |
| R4 | **Line field.** (a) *Which entry is the item entry.* If `itemLedgerId` is set and the voucher has an entry at **index 1** on that ledger, that entry, and only it, is the item entry (`VoucherScreen.save` writes `[party, item entry, ...extras]`, so an extras row on the SAME ledger sits at index 2 or later and is a `charge`). Otherwise (an **imported voucher whose lines post to more than one allocation ledger**: `tallyImport.ts` sets `itemLedgerId` only for a single ledger, and `core/einvoice.ts` and `ui/print/invoiceModel.ts` already have an `unpointed` branch for it) the candidates are the counted P&L entries that are not the party, a tax ledger, TCS, TDS or Round Off: **exactly one candidate** is the item entry, and only the part of it up to `Σ InventoryLine.amount` is item value (any excess is a `charge`, which is how e-invoice reads an unpointed voucher); **more than one candidate** cannot be tied to lines, so every P&L entry of that voucher goes to `unset` and the report's footnote counts "N vouchers could not be tied to their lines" (never guessed: a wrong colour is worse than a missing one). (b) *The split.* The item entry's item value is split by each option's share of `Σ InventoryLine.amount` (lines with no value weigh into `unset`), **by largest remainder to the paisa so the shares add to the entry exactly**; zero total weight goes wholly to `unset`. (c) Every other P&L entry of the voucher (freight, a discount ledger, Round Off, an extras row) is `charge`. A voucher with no inventory is `notAsked`; one of a base outside `scope.line` is `notAsked` unless it holds line values, which then attribute as above (CF18) |
| R5 | Income takes `Cr − Dr`, Expense `Dr − Cr`, as `costpl.tsx` does, so a credit note reverses a sale inside its own option |
| R6 | Charges are **not** spread over lines by default; they are their own row, so nothing is hidden by a rule a reader cannot see. (A company switch "Spread charges over lines" is a later option; it changes no total.) |

### 6.4 The report: Income and expense by field (`core/fieldPl.ts`, the screen `fieldPl`)

> **The `FieldPl` shape below is superseded by AD.2.6** (five sources, a per-row `stock` column, `stockChange: { total, attributed, unattributed }`, footnotes). What does not change: the layout, the tie to `plFigures`, the basis, the "Other" line and the rights. It is kept here as the v1 reading for a header or line source with no stock attribution.

```ts
export interface FieldPlRow { bucket: Bucket; label: string; income: number; expense: number; net: number }
export interface FieldPl {
  rows: FieldPlRow[];                                   // options (list order), removed, Not set, Not asked, Whole-invoice charges
  total: { income: number; expense: number; net: number };
  stockChange: number;                                  // closing stock less opening stock, from core/plFigures.ts plStock: NOT attributable to an option (v1 reading; AD.2.6 R10 attributes it for an item source and a tracking field)
  nett: number;                                         // total.net + stockChange
  plNett: number;                                       // plFigures(...).nett for the same window
  diff: number;                                         // plNett - nett: must be 0.00
}
```

- **Layout A (v1):** one row per bucket with Income, Expense, Net and % of net; below the total a **Change in stock (not attributed)** line, then **Profit and Loss
  nett** and a check line "Agrees with the Profit and Loss statement" with the figure. The row set is exactly Zoho's "Compare With" turned on its side; **Layout B**
  (ledger rows by option columns, up to eight chosen options plus Others, Not set and Total) is a later slice (CF-5b).
- **Reconciliation proof (CF11):** `rows` summed plus `stockChange` equals `plFigures` nett to the paisa on the sample company, on a company with a loss, with a stock
  change, without stock items and on a 50,000-voucher synthetic company, for a header field, a line field and a field scoped to two voucher types; `diff` is **computed
  and shown**, never assumed. **What counts is the six reserved subtrees `plFigures` reads (R2); an Income- or Expenses-nature ledger outside them is a memo row "Outside the statement: X (not in nett)", excluded from `nett` and from `diff`, so the tie holds on a company that has such a group** (the first text said the report "adds an Other line", which either put the amount inside nett and made `diff` non-zero by construction or was undefined; the Dashboard's Income and expense card keeps its own Change-in-stock line, Phase 28).
- **Basis, said on the page:** postings in the window, opening balances excluded (as `plFigures`' `moveFromTotals`), accrual only.
- **Margin by option is not offered.** `ui/reports/profitability.tsx` costs goods at **current** average cost, not at sale time [C audit 5]; a margin by Colour would
  silently re-state last year. Said in the report's footnote and in the open questions. (An **item-source gross figure with the exact stock movement is offered**, AD.2.6 R10: it uses the same opening and closing stock as the Profit and Loss, not a re-costing.)
- **Rights:** `SCREEN_AREA.fieldPl = 'reports'`.

### 6.5 What this does for the owner's examples

| Case | How |
|---|---|
| Zone on a Sales invoice, filter in the Sales Register, a "Sales by Zone" summary | Case 2 |
| "Profit by Zone" | Income and expense by field, header field Zone; Purchases of the same Zone appear when the field is scoped to Purchase too |
| A profitability tag across Sales, Purchase, Expenses, Journal | a header field scoped to those bases; the report's **Not asked** row is every voucher of a type outside the scope that holds no value, **Not set** every in-scope voucher left blank |
| Colour at item level | Case 1 |
| "Profit by Brand" or by item Category, with the stock movement | Income and expense by field, `item` source: the Change in stock is attributed exactly by item (AD.2.6 R10) |
| "Profit by customer Category" or by the party's Salesman | `party` source (current value, labelled so); several parties on one journal go to Several parties (R7) |
| "Fixed against variable cost" (a field on the expense ledgers) | `acct` source (R9) |
| "Profit by Colour" when Colour tracks stock | `line` source; the Change in stock is allocated by variant (R10, AD.3.5), and labelled so |
| Any other field, any other shape | AD.5, the scenario atlas |

### 6.6 Cost centres: one thing to hand on, not part of this spec

The cost-centre side has defects found while researching tags (an item invoice carries no cost allocation at all; parallel allocation cannot be entered; the Tally importer
would double count a parallel-allocated file; none of the cost reports has a "not allocated" row) [R zoho-reporting-tags 6.2]. They are parity work for the orchestrator. Adding
**Not allocated** to the existing cost reports is the same seam as the Not set row above and is flagged as a small separate item.

### 6.7 What the tag design does not do (said plainly)

No Balance Sheet or Trial Balance by field (a header tag does not reach every balance-sheet leg, and a line tag reaches only the item entry; the same limit QuickBooks documents for
classes [R]; **Balances by field** (AD.2.7) groups ledgers by the ledger's own field, which does reach every leg of that ledger, and is not a Balance Sheet); no option hierarchy and no conditional options (each needs a roll-up rule in every report); no access control by option (reserved); no multi-valued tag (it would
break the reconciliation); no split of one amount across options (that is a cost category's job).

---

## 7. The definition screen

`ui/admin/CustomFields.tsx` is rewritten (`SCREEN_AREA.customFields = 'admin'`, unchanged). Keyboard-first **and** fully mouse-operable, opened from: the Gateway's Company menu (key D, as
today), Go To (existing aliases kept), **a new row "Custom Fields" in the left menu's Settings group** (appended after Preferences: the menu goes from 65 to 66 rows, so `navTree.ts` and the
literal row table in `scripts/assert/sidenav.assert.ts` change together, and a `+` on that row resolves through `quickCreate.ts` to "New custom field"), the "Custom fields..." link in
Settings > Workspace, a link at the foot of the grid's Columns panel ("Add a custom field", administrators only), and a small "Manage fields" link at the end of the voucher strip (administrators only).
It is not a Settings-window tab: that window is for a person's look and keys, which are not books data.

### 7.1 Layout

```
 Custom Fields                                                          <company>
 [ Parties ] [ Items ] [ Vouchers ] [ Item lines ] [ Employees ] [ Leads ]          (object tabs: Left and Right)
 ----------------------------------------------------------------------------------------------------------
  Field        Type      Asked on                    Required   Reports   Print   Used by
  Zone         Choice    Sales, Credit Note, Party   Policy     yes       no      412 vouchers
  Salesman     Choice    Sales                       no         yes       no      0
  [Hidden fields (2) v]
 ----------------------------------------------------------------------------------------------------------
  Starter sets...   Fill blanks...   Update 3 older fields...                                 [ + New field ]
```

A field that lives in several places appears on each tab it belongs to, marked "also on Items". The right half of the edit panel is a **live preview**.

### 7.2 The edit panel (a docked pane inside the screen, not a `Box`)

| Block | Controls |
|---|---|
| Name | label (cleaned, at most 40, unique after folding); a duplicate like "Colour" and "Color" is caught by the folded comparison |
| Type | the 13 types of 1.2, **locked after creation** with the sentence "To change the type, hide this field and make a new one" |
| Options (choice) | an editor with add, paste a block, a preset menu, rename in place, Hide/Show, reorder, Make default, Delete (only at zero use, with the count); the count of records holding each option |
| Type rules | text: max length and a preset pattern; number, amount, percent: min, max, decimals; lookup: the master and, for ledgers, the group |
| **Where it is used** | check boxes: Party or ledger (pick groups), Stock item (pick stock groups), Voucher (pick voucher types: a tree of bases with their types), Item line (pick voucher types), Employee, Lead. At least one |
| Required | **No / Yes.** When Yes (AD.1.5): "When an existing voucher or record is edited: **(o) Follow the company setting** (now: New entries only; change it in Settings > Workspace)  ( ) Only new entries are checked  ( ) Always checked: an old voucher must be completed to be edited." Writes `req: 'policy'`, `'new'` or `'always'`. Under it a live count ("412 existing Sales vouchers have no Zone") and, for Always, a link to Fill blanks. A field made Yes is asked in the Enter walk (the sentence of 2.1) |
| Ask while entering | "Ask in the Enter walk" (greyed on, and explained, when Required) |
| Starts from | static default (typed per type); "Start from the party's value" (when the field is also on a party); "Start from the item's value" (when also on a stock item) |
| Where it shows | Available in reports (on); Print on the voucher (off); Copy when a voucher is duplicated (on) |
| **Stock** (a choice field with the Item line place) | **Tracks stock** (off): "Keep stock separately for each Colour." Ticking it ticks the Item line place, **locks** the eight stock-moving voucher bases (AD.3.2), **locks Copy when duplicated on**, sets Required to Yes (follow the company setting) with the sentence "A blank books the stock as Not set", and shows the **preflight**: "N items in these groups hold stock worth X; their stock will read Not set until you split it (Opening by variant, or a Stock Journal); a sale of a variant with no stock will warn NEGATIVE." At most three tracking fields apply to one item. **While it is ticked, Hide and Delete are not offered** ("This field keeps stock by <label>. Stop tracking first: stock returns to whole items and nothing is rewritten") and **Stop tracking** is the control (AD.3.2). Without it, a field with the Item line place carries "This labels the stock; it does not split it." |
| Help text | one line, at most 80 |
| Actions | Save (Ctrl+A), Hide / Show (**not for a field that tracks stock: Stop tracking first, AD.3.2**), **Delete (no use here, and on a synced company a fresh sync only; it writes a tombstone, 10; not for a field that tracks stock)**, Cancel (Esc: **asks "Discard changes?" when the draft differs from the stored row, No armed**), Open Summary (when groupable) |

**Layout.** The first draft called this "a docked panel in the F12 family"; that was self-contradictory, because the F12 panels (`reportConfig`, `VoucherConfig`) are `Box` modal shields, which the Back spec counts as mouse traps. This pane reuses the Settings window's `.st-layout` (the list on the left, the panel on the right, the live preview under the panel) with the `useNarrow` hook: **below 1100 px the panel stacks under the list and the preview moves below the panel** (Phase 27 needed the same hook and rejected a CSS container query for its layout containment). It leaves the rail's Quit and the top bar's Back reachable.

**Minimum path and progressive disclosure.** The panel opens **prefilled**: the place of the tab you are on is ticked, the voucher types of that place are preselected as "every type of the bases", Required is No. **Name, Type and (for a choice) Options are the only blocks open, and Ctrl+A saves with just those**; everything else sits under **More options** (collapsed, remembered per device): type rules, the group and voucher-type scoping tree, Required and Ask-while-entering, Starts from, Where it shows, Help text. The plain-language summary under the controls is the confirmation step. The type is locked after creation, so the Type control carries a one-line description of each type and the "wrong type? hide this field and make a new one" note **at the point of choice**, not after. The vocabulary ("Asked on", "Starts from", "Where it shows") is Zoho's; TallyPrime's own wording could not be checked, [unverified], so it is a veto item (question 10).

**Plain-language summary under the controls**, so the owner's "where is it visible" is answerable at a glance and never needs the matrix: *"Asked on: Sales and Credit Note vouchers and on
the party. Required: for new entries. Starts from: the party's Zone. In reports: available as a column, filter and summary, not shown until someone chooses it. Printed: no."*

**Live preview** renders the real components (`CustomDetails` for the voucher strip, the grid's header and three sample cells, the print sub-line) from the draft definition, never a
second drawing of them, so the preview cannot drift.

### 7.3 Keys (screen-local; **no chord is added to the registry**)

| Key | Does |
|---|---|
| Up, Down | move between fields; Left, Right change the object tab (while the list has the keyboard) |
| Enter | edit the field |
| Alt+A | new field (the editor's existing meaning: "Add Field") |
| Alt+Up, Alt+Down | move the field up or down (the Dashboard editor's meaning, listed in `RAIL_FIXED`; the order is `order`) |
| H | hide or show the highlighted field (a bare letter on a list screen with no text field focused, as the Gateway does) |
| Ctrl+A, Esc | accept the edit panel, leave it (Esc on the list quits; **Esc on a changed panel asks "Discard changes?"**; the Esc rail button runs the same handler, per the Back spec) |
| buttons | **every key above has a visible button**: Add, Save, Cancel, Hide / Show, Delete, **Move up and Move down** (for fields and for options), Paste options; the rail keeps Alt+A Add Field and Esc Quit |

### 7.4 Warnings the screen must say

- **Older apps (a refusal, not a warning).** In a **synced** company, when the server's pull answer lacks `caps: ['udf2']` (9.3), the editor **refuses to save** a definition for which `needsServerCap(def)` is true (every field except a text or choice field on one master place, 1.1): "The cloud server has not been updated to protect these values from older copies of the app. Update the server first." A definition that already exists is not touched, and entering values is not blocked (values are what the server rule protects). Local-only companies are unaffected and are not told anything (a restore onto an older install is covered by the backup marker, 12).
- **Stock is not split by a label** (2.2), on a field with both `stockItem` and `line` places.
- **Mandatory** (3.2, AD.1): the walk sentence of 2.1, and the sentence on old vouchers that names what the company setting is NOW ("Existing vouchers: new entries only"), so the person making a field mandatory sees what editing an old voucher will do.
- **Tracks stock** (AD.3.7): the preflight above, the three-field limit, and, when a tracked item has a bill of materials, "A Manufacturing Journal books this item as Not set; the components of a build come out of Not set and the finished goods go into it, so Red and Blue do not fall and Not set can go negative". **Hide and Delete are refused while it tracks** (AD.3.2), and a count made by variant is one whole-item count (AD.3.3): "A Physical Stock count of a tracked item is a count of the whole item; variants you leave out are counted as 0."
- **v1 fields** (1.6): "Update 3 older fields" converts them in one confirmed step.

### 7.5 Starter sets (cheap, high perceived value [R tally-udf 6.3 item 8], data not code)

A "Starter sets..." menu adds definitions from a table in `core/customFieldPresets.ts`, previewed before anything is written, each field editable afterwards, all
off by default for printing. **The field lists below are my proposals for the owner to confirm (question 10), not researched requirements.**

| Set | Fields |
|---|---|
| Garment | Colour (choice; item and line), Size (choice; item and line), Design no. (text; item), Season (choice; item) |
| Distributor | Zone (choice; party and Sales invoice), Route (choice; party), Salesman (choice; Sales invoice) |
| Pharma | MRP (amount; item), Salt (text; item), Drug licence no. (text; party) |
| Contractor | Site (choice; voucher), RA bill no. (text; voucher) |
| Jewellery | Purity (choice; item and line), Gross weight and Net weight (number; line), HUID (text; line) |

Vehicle number, LR number, transporter and e-way fields are **not** custom fields: wave 3 builds them as fixed dispatch fields (`src/core/dispatchTypes.ts`, uncommitted) and TallyPrime has them
built in [R tally-udf 0.2 item 8]; the starter sets never duplicate them.

A starter set is refused whole in a synced company when any of its fields needs the server capability and the server lacks it (7.4); the preview says so.

### 7.6 Unsaved work: the editor registers a dirty probe

Neither the editor nor its panels registered one in the first draft, so both "Leave this entry?" on a rail click and the Back control (back spec 2.1) would have treated them as clean: `isDirtyAt` falls back to `mayHoldWork`, whose set is `bulkEntry`, `importData` and the create forms, and `CustomFields.tsx` registers nothing (A, item 12i). The editor calls `useDirty` (`state/dirty.ts`), true while **(a)** the edit panel's draft differs from the stored row (a field-by-field compare after `normaliseDef`), **(b)** the paste box holds text, or **(c)** a Starter-set or Fill-blanks preview holds choices the person made. `navstack.assert` adds `customFields` to its wired list and asserts the probe is true for each of (a) to (c) and false for an untouched panel (CLAUDE.md: a screen that can hold typed work registers a probe). Esc on a dirty panel asks "Discard changes?" (7.3).

---

## 8. Rights

| Who | May | Where it is decided |
|---|---|---|
| Define, edit, hide, delete a field; Fill blanks; Starter sets | holders of the **Administration** right | `SCREEN_AREA.customFields = 'admin'` and `TABLE_AREA.customFields = 'admin'`, **unchanged** [C `core/access.ts`]; the write guard refuses a stale tab or a console too |
| Enter or change a value | whoever may write that record: the voucher's base right (`writeDenial` by base, action and date window), the Masters right for a ledger or item, Payroll for an employee, Accounting for a lead | the existing guards; **no per-field right in v1** |
| See a value in a report | whoever may open the report | the report's own area; the two new screens are `reports` |
| Fill blanks | Administration **and** the write right of each voucher base touched (the batch goes through `putDoc`, which the guard judges per document); **on a synced company only when the device has nothing waiting to send and has pulled within the last minute (10)** | `data/writeGuard.ts` unchanged |
| **Change the company's entry rules for old records** (AD.1) | holders of the **Administration** right (`canAdminister`) **and** the write guard on the company row; everyone else sees the panel read-only | `writeEntryPolicy` (`writeProblem` first), then `putDoc` again at the write |
| Open Summary, Income and expense, Bills, Balances or Stock by field | whoever may open reports: `SCREEN_AREA` `fieldSummary`, `fieldPl`, `fieldBills`, `fieldBalances`, `fieldStock` are all `reports` (they write nothing) | `core/access.ts` (each screen needs its entry, or `security-access` fails) |
| Set opening stock by variant; build a variant-split Stock Journal (AD.3.7) | the Masters right for the item write, and the Stock Journal base right for the voucher | the existing guards, per document; the buttons are not drawn for anyone else |
| With security off | everyone, as everywhere | the operator model is opt-in |

**Reserved: `access: 'admin'` (field-level hiding), CF-11.** Zoho can hide a field per role but I could not confirm that columns, filters, exports and the API honour it [R zoho-custom-fields
T8]. If built here, "hidden" must be **one decision** (`core/access.ts` gains `fieldDenied(ctx, def)`) read by every surface, or it is cosmetic: the entry strip and master forms (not drawn), the
Columns panel, the filter field list, the summary pickers, saved-view apply (the stored id is kept, the column is dropped for that operator), CSV and Excel (a hidden column is not rendered, so
nothing leaks), Go To, and the print switch (an administrator's choice, independent). **The REST and webhook payloads and the backup file carry the value regardless, and the screen says so**:
`access` is an in-app control, not encryption. Not built until the owner asks (question 4).

---

## 9. Sync, older clients, the server

### 9.1 Every writer of a voucher, and what it must do about `udf` (the master-resave lesson at the voucher layer)

`VoucherScreen.save` builds its `Voucher` as a **fresh object literal** (`const v: Voucher = {`), so a field it does not name is erased on every alteration; three more screens and one
form do the same [C audit 2.2]. There is a resave suite for masters (`master-resave.assert.ts`) and **none for vouchers**; CF-2 adds it.

| # | Writer | What it must do | Test |
|---|---|---|---|
| A1 | `VoucherScreen.save` (every base the voucher screen draws) | `udf: mergeUdf(editing?.udf, stripEdits, defs)`; each line `udf` from its `ItemRow`; a sibling of the existing carry lines (`exportInfo`, `irn`, `...carriedNumbering(editing)`); calls `saveVoucher(v)` with no flag (`form`, 3.3) | `voucher-resave` |
| A2 | line rebuild: `ItemRow`, `itemRowFromLine`, `validInvLines`; the stock-journal source and destination grids | carry `udf`; **on a Physical Stock voucher also `countPart`** (written only for a tracked item's count by variant, AD.3.3; dropping it would turn a count group into a last-line-wins count of the whole item) | `voucher-resave` |
| A3, A4 | the entry literals of the accounting grid (`AccRow`) and of **invoice mode** (party, item-ledger, extras: the extras keep only ledger, side and amount today) | **nothing**: the `entry` place is reserved (1.1), so no custom value lives on an entry and these literals are not touched; the dimension at entry level stays the cost centre | `voucher-resave` (a key of `VoucherEntry` is classified too) |
| A5 | Stock Journal from Manufacturing (`manufacturing.tsx`) | **pass through** `editing?.udf`; no strip in v1; `saveVoucher(v, { udf: 'exempt' })` | `voucher-resave` |
| A6, A7 | Attendance and Payroll vouchers (`payroll/vouchers.tsx`) | pass through; **Attendance and Payroll bases are not offered** as a voucher scope | `voucher-resave` |
| A8 | Salary Disbursement (always a new Payment) | `saveVoucher(v, { udf: 'exempt' })` (**not** `system: true`, which would skip the operator guard, 3.3) | |
| A9 | Voucher **Type** form (`masters2.tsx`) | **nothing**: the scope lives on the field, not on the type, so this site is not touched (the reason the scope is `field.scope.vch.typeIds` and not a flag on `VoucherType`) | `master-resave` unchanged |
| A10 | cancellation shell (`voucherCancel.ts`, `useCancelVoucher`) | **drops** header and line values by design (2.4); `saveVoucher(shell, { udf: 'exempt' })` | `voucher-cancel` extended |
| A11 | the Ctrl+Z and restore applier (`overlays.tsx` `applier.put`, which routes ledgers through `saveDoc` too) | `{ udf: 'exempt' }` on both; a deleted voucher has no stored row and would be judged a `create` (3.3) | `voucher-resave` plus the cancel, undo and delete-undo test |
| A12 | cheque layout and cheque book saves (`cheque.tsx`, `chequebooks.tsx`: `saveDoc` of a spread ledger) | `{ udf: 'exempt' }`; they spread the stored ledger, so `udf` survives | `master-resave` |
| B1 | Bulk Entry | builds the voucher with its cells; mandatory at voucher-screen severity (3.3) | `bulkentry` extended |
| B2, B3 | CSV Import Data, the mapped importer | custom columns as mapping targets (11) | `importmap` extended |
| B4 | Tally XML import | UDF tags mapped later (CF-10); until then dropped, counted in the preview | |
| B5 | system journals: RCM, ITC (`rcmWrite`, `itcWrite`) | exempt; rebuild from `{...keep}` where they repost | |
| B6 | sample company and onboarding seed | none | |
| C | spread or patch writers: GSTR-1 export info, advance rate, cheque, BRS, mark optional, e-invoice strip, numbering, undo, restore | safe already: they spread the stored document, so an unknown field survives; the suite proves it for `udf` | `voucher-resave` |

The suite classifies **every key of `Voucher`, `InventoryLine`, `Employee`, `Lead`** as "written by the form", "carried by the funnel" or "off-form, preserved" and fails on a new key that is none of
them (the way `master-resave.assert` reads `types.ts`), and a mutation that removes the carry line of A1 is killed.

### 9.2 The client's own layers

1. **Merge, never rebuild** (`mergeUdf`, 2.3): a save keeps every stored key it does not own.
2. **One carry funnel**: the table above; no site writes a `Voucher` literal without it.
3. **A capability marker.** The push body gains `caps: ['udf2']` (beside `deviceId`, `deviceLabel`, `changes[]`), sent by every build that carries the 9.1 funnel; the server reads and validates it (9.3). `sync.ts` is already modified by wave 3, so this lands after it.
4. **The server's answer** (the pull AND the push response) gains `caps: [...]`; the client keeps the last answer per server, and the editor refuses what needs a missing capability (7.4).
5. **Definitions and options are never removed from a document** (10), so a stale save or an older editor cannot make a value unlabelled.

### 9.3 The server rule (`backend/src/conflict.js`, `sync.js`, `rest.js`; CF-6a)

**What is wrong today** (A, items 5 and 12a): the two protection lists cannot tell an older app that sends a base from a current one, so an older app's whole-document write of a voucher without `udf` is accepted as an ordinary edit. **The rule:** a write from a device that did **not** state the `udf2` capability must not erase custom values it does not know.

**The I/O contract (the first draft left it out, and without it the rule cannot fire).** `classifyPush` is pure and reads the `others` and `latestOther` its caller loaded. `loadHistories` loads `others` only for rows NEWER than the writer's base, and `latestOther` only for tables in `PROTECTED_FIELDS` or `UNVERSIONED_PROTECTED`. For a voucher written by a new device and then pulled by an older one, the older device's cursor and stamped base are already at or past it, so `others` is empty and `latestOther` is never loaded: **a rule that read `classifyPush`'s `anchor` would be inert in exactly the commonest case**, while a suite that injects `latestOther` would stay green. The rule therefore reads its own input, **`current`**: the newest `change_log` row of the document, of any device and any age, which is what this write is about to replace.

1. **The push body states its capabilities.** `caps` is an array of at most 8 strings of at most 16 characters of `[A-Za-z0-9_-]`; anything else is ignored (a hostile body cannot grow the query or the log). `planPush` and `loadHistories` gain a `caps` argument (a `Set`).
2. **`loadHistories` marks a document `need.current`** when the writer lacks `udf2` and the change is (a) a `put` to a table in `UDF_CARRIERS` or (b) a `delete` of `customFields`. It then issues ONE extra indexed read for the batch, `SELECT DISTINCT ON (tbl, doc_id) tbl, doc_id, seq, op, payload FROM change_log WHERE tbl = ANY($1) AND doc_id = ANY($2) ORDER BY tbl, doc_id, seq DESC` (the protect query without its device filter; it uses `idx_change_doc`), and stores each row in `histories.get(k).current`. `planPush` hands it to `classifyPush({ ..., current })`. `loadHistories` is exported (it is module-private today) or split into a pure planner and a thin executor in `backend/src/histories.js`, so a suite can drive it (14).
3. **Cost, stated.** Zero for a writer that states `udf2`. For one that does not: one extra indexed read per push returning the whole current payload of each distinct document in it, at most 300 documents per push from the shipped client (`PUSH_BATCH`), inside the tenant lock that already serialises the push. The first draft's "O(lines)" was wrong: it reads whole documents. A company that never defines a voucher or line field still pays the read when an older device pushes (a pre-check on the tenant's definitions would cost a read too), so the read is bounded and the merge is skipped when `current` carries no `udf` anywhere. The cost stops mattering once every device has updated [J, an estimate from the query's shape; the first measurement replaces it].

```js
export const UDF_CAP = 'udf2';
/** Tables whose documents carry custom values, and where. */
export const UDF_CARRIERS = Object.freeze({
  ledgers: { bag: 'udf' }, stockItems: { bag: 'udf' }, employees: { bag: 'udf' }, leads: { bag: 'udf' },
  vouchers: { bag: 'udf', lines: { path: 'inventory', bag: 'udf', keep: ['countPart'],
    identity: ['stockItemId', 'godownId', 'batchName'],                    // the WEAK identity
    strong: ['stockItemId', 'godownId', 'batchName', 'qty', 'rate', 'amount'] } },   // the STRONG one: what an older app does not touch is unchanged

});
export const udfCarrierOf = (t) => (typeof t === 'string' && Object.hasOwn(UDF_CARRIERS, t) ? UDF_CARRIERS[t] : null);   // the prototype lesson of protectedFieldsOf

/** The incoming document with the keys an older writer could not have known put back. Pure; { payload, kept } where kept names what was restored. */
export function carryUdf(table, current, incoming) { /* header: { ...current.udf, ...incoming.udf }; lines: see below */ }
```

- **Header bag:** the merged bag is `{ ...current.udf, ...incoming.udf }` (keys the old writer sent win; keys it omitted survive). An older app therefore **cannot clear** a value (clearing is omission); that is accepted and recorded. It cannot repair a value an older form MUTATED, which is why CF16 keeps older forms away from the fields they would mutate (1.1).
- **Lines: restore a line's `udf` (and its `countPart`) only where the pairing is unambiguous, pairing by a strong identity first.** The **weak identity** is (`stockItemId`, `godownId`, `batchName`), and `validInvLines` never merges rows, so one invoice can hold the same shirt on three lines (Red, Blue, Black) with ONE weak identity three times. The first draft matched "preferring index `i`", which pairs by position: an older client that deleted line 1 of those three would have had Blue relabelled Red and Black relabelled Blue, and Summary by field would then have moved item value between colours with no sign anything was wrong. The second draft paired a weak identity only when it occurs exactly once in BOTH lists, which is safe but, for a **tracked** item, loses everything: a size run is the SAME item on N lines by design (AD.3.6.6), so its weak identity is ambiguous every time, and an older app that added or removed ONE line (a freight-free extra line) or changed ONE quantity moved every line of that voucher to Not set, shifting 40 units from Red, Blue and Black to Not set, recorded in the Mismatch Report but not undone. **The pairing is now iterative, and every step is exact or absent.** The **strong identity** adds what an older app does not touch on a line it did not edit: `qty`, `rate` and `amount` as stored. The rules, in order: (i) if the incoming and current line lists are **equal in length and line-for-line equal on the weak identity, in order**, every line takes the `udf` and `countPart` of the line at its own index (an older client that changed only the narration, or a quantity or rate in place); (ii) otherwise pair every line whose **strong** identity occurs **exactly once in BOTH lists** (the lines the older app did not change); (iii) then, among the lines still unpaired **on each side**, pair a **weak** identity that occurs exactly once on BOTH remaining sides (the lone line whose quantity the older app edited: the other two of a size run paired in step ii); (iv) anything else, a line whose identity is still duplicated or absent on either side, gets nothing, and the loss is recorded. A paired line takes the current line's `udf` when it has none (and merges when it has some). **A wrong colour is still worse than a missing one:** two lines identical in every stored field but the variant are never paired (an older app that deletes one of them leaves the other unlabelled and recorded, because which one it deleted is unknowable), and a step (iii) pairing needs ONE candidate on each side. The same function runs for REST (9.4), where the writer never states caps. **Vectors:** [Shirt Red 10 @100, Shirt Blue 5 @100, Shirt Black 7 @100] plus an appended freight-free line keeps all three variants; with one quantity edited keeps all three (the edited one by step iii); with the first line deleted keeps Blue and Black on Blue and Black, never Red and Blue; two lines identical but for the variant with one deleted come out unlabelled and recorded; an older app that changed no line keeps everything by step i. **Cost and hygiene:** the pairing is O(lines) with a `Map` keyed by the identity string (no nested scan; a hostile body with 100,000 lines is still linear), keys are built by joining the fields with U+0001 after cutting each to 200 characters, and nothing in it is a pattern test (the "cap length before any pattern test" lesson).
- **Lines, fix round (review of 2026-10-24): what step (i) compares, what is not paired, and the cost bound.** (a) Step (i)'s "the rest of the line is unchanged" test (needed where a weak identity repeats) compares the lines **without `udf` and without the `keep` flags (`countPart`)**: an older app drops both, so a comparison that stripped only the bag called every Physical Stock count line "changed" and a count made by variant with REPEATED counts (10/10/5, 10/10/10) never paired by index and collapsed into whole-item counts (it moved the item's stock on every device); counts 10/7/5 were rescued by step (ii) only because their strong identities are unique. Vectors: 10/7/5, 10/10/5 and 10/10/10 saved again with a changed narration keep every variant and flag; three equal counts with the middle one edited in place keep all three. (b) **A cancelled voucher is exempt from the whole carry** (`carryUdf` returns the document untouched, header and lines): the shell drops its values on purpose (2.4, A10), so putting the header value back and reporting the (nonexistent) lines as lost made every Alt+X cancel of a voucher with custom values by an older build a `protected-fields` mismatch about a voucher with no lines. A shell is a voucher whose `cancelled` is an OBJECT (the same test as the cloud's cancel rule); `true` or `null` is not one, and a master with a field of that name is not a voucher. (c) **Bounded work.** The pairing costs about 4 to 20 microseconds a line, synchronously inside the tenant lock on the one event loop, so a voucher of 200,000 lines (a body near the 20 MB limit) stalled every tenant for 3 to 4 seconds. A document whose either list exceeds `CARRY_LINES_MAX` (5,000) is not paired (the header bag still is; the lines stand as sent; every current line that held a value is reported with `why: 'too-many'`, capped at 50 like every list), and one push may spend `PUSH_CARRY_BUDGET` (20,000) line-units (the two lists' lengths together) on ALL its documents, a document that does not fit being skipped the same way. A 200,000-line voucher now answers in about 15 ms; real vouchers are tens of lines and are never near either limit. (d) **Residual risk, accepted and stated:** step (ii) pairs on the STRONG identity as stored, so it is exact only while an older app's edit does not make one line's quantity, rate and amount coincide with another line's. Example: current Red 10, Blue 5, Black 7 (one item, rate 100); an older app deletes Blue AND edits Black from 7 to 5 in one save: the surviving 5 @ 100 line is paired with the deleted Blue line's strong identity and is labelled Blue although it was Black. It needs a delete plus an in-place edit in ONE older-app save that lands on another line's exact figures, it is recorded in the mismatch row like every carry, and refusing it would also refuse the common case (an older app deletes one line of a size run and touches nothing else: every surviving line pairs by its strong identity), so the spec does not claim "exact or absent" beyond that: it is exact unless an edit coincides with a sibling's figures.
- **Definitions:** a `delete` of a `customFields` document whose `current` payload has `v: 2`, from a writer without `udf2`, is **kept as a hide**: the server stores a `put` of `{ ...current.payload, status: 'hidden' }` instead and records it. (An older editor deletes through `removeDoc`, and its `checkDelete` has no custom-fields case, so no client-side guard exists; a v1 row is not covered, because it carries no enforcement and an older app may delete it as today.) A delete from a current client is the tombstone `put` of 10 and never reaches this branch.
- **Applied** in `classifyPush` beside the passcode rule: only for `op: 'put'` (and the definition delete above), only when the writer's `caps` lack `udf2`, only when `current` is a `put` that holds any value, and **whichever device wrote `current`**. There is no same-device exception to make: a device's capability is a property of its build, so a device that states `udf2` never reaches the rule. It is **inert for a current client** and stops mattering once every device has updated.
- **Recorded, and counted even when it cannot be recorded.** What was restored is pushed to `protectedKept` (as `udf`, `inventory[i].udf` and `delete`), which the machinery records as a `protected-fields` mismatch with `theirs: current`, so the administrator sees "an older copy of the app edited this voucher; its custom values were kept" in the Sync Mismatch Report. **The values are always kept; the row is recorded only while the tenant holds fewer than `LIMITS.openRows` (2,000) open mismatches** (`sync.js`: past that, `up.capped` is only a `log.warn`), so "recorded, not silent" was false of every row in the first draft. The push answer therefore gains **`udfKept: n`** (documents whose values were restored, counted before and independently of the mismatch log) beside the existing `adjusted`; the server logs it, and the report's existing `full` flag tells the administrator the list is full.
- **Not a refusal.** Refusing (`426`, "update the app") the writes of an app without the capability is the stricter option; it strands an Electron install that nobody updated and so is **owner question 9** (default: merge and record).
- **The editor enforces the order of release (7.4):** while the pull answer lacks `caps: ['udf2']` the editor refuses (it does not merely warn) to save a definition with `needsServerCap` true in a synced company.
- **The company row's entry policy rides in the same deploy, keyed on the capability (AD.1.2).** `CAP_KEPT.companies = { entryPolicy: UDF_CAP }` keeps the current policy against **any company-row write from a writer that does not state `udf2`, whatever base it states**, and `OMIT_KEPT.companies = ['ui']` restores a `ui` that such a writer omits. The first text put `entryPolicy` in `UNVERSIONED_PROTECTED`, keyed on "states no base", which this section's own opening sentence says cannot tell an older app that sends a base from a current one: every client since Phase 24 stamps a base while signed in, so the live older builds were never covered, and a current client's rows made signed out, before its first pull, or in another account's namespace carry no base, so an administrator's legitimate edit would have been reverted. `needsUdfCurrent` gains the company-row `put` (one extra indexed read per such push, rare); neither list is in `PROTECTED_FIELDS`. **The CF-6a code built from the first text must be reworked to this before the backend deploy.** A tracking field (AD.3) needs no server rule beyond `udf2`: its values are line `udf` values and a count group's `countPart` flags, which `carryUdf` protects (a line whose pairing stays ambiguous after the strong-identity steps loses its variant, recorded, which for a tracking field means the stock of that line reads Not set until it is corrected: the Mismatch Report entry says so; an older app that edits a Physical Stock voucher of a tracked item is rare; its count group keeps `countPart` by the same pairing, and a line that cannot be paired loses both, recorded).

### 9.4 REST

`rest.js` inserts the body as the new current payload without going through `classifyPush`, so it needs the same rule in its own place: on `POST /v1/vouchers` (and the other carriers) the current
document is read **inside the same `withTenantLock` callback as the insert** (two requests cannot both read the old document) and `carryUdf` applied (REST never states caps, so the rule always applies, with the line rule of 9.3: an integration that re-posts a voucher with its lines reordered keeps the `udf` of lines whose identity is unambiguous and loses the rest, recorded), so a body that omits `udf` **keeps** the current values; a body with `udf` merges by key; **a key set to `null` clears it** (REST only; the apps clear by omission
on the same device). Webhooks are unchanged (they fire only from REST writes, audit finding 11): the payload simply carries `udf`. `customFields` stays in `REST_DENIED`; definitions are readable.
Server-side **validation** of values (types, option ids, mandatory on create) needs the server to read the current definitions from its own change log; it is the API slice (CF-6b) and until then REST
accepts any `udf` shape (documented in `ApiDocs.tsx`).

### 9.5 Definitions are synced documents: edits, concurrency, labels

- A definition is an ordinary row of the existing `customFields` table, synced whole; two administrators editing one definition collide as any document does (recorded mismatch, last push wins).
- **The editor never saves from a stale copy and never drops what it did not see**: like `writeCompanyUi`, a save is a read-modify-write of the **stored** row. **Options are merged by id:** the saved `opts` is the union of the stored list and the edited list; an option the saver never saw (added by another administrator) survives; the `active` and `gone` flags of an edit apply only to the ids the editor had loaded (it carries that set), and **an option is never removed from `opts`** (10). A value reads "(removed option)" only when its id was never in any list; a `gone` option keeps its label.
- **The Mismatch Report** labels `udf` changes by field, not by key: `core/syncDiff.ts humanize` resolves `udf.<key>` through the definitions (a generated fid is meaningless to a person), and a line
  custom value changes the line's displayed leaf, not its identity (`ROW_KEYS` stay item, godown, batch; two lines of one item differing only by colour are `(2)` there).
- **Per-field merge of values is not built** (the server's model is whole-document last-push-wins with a recorded mismatch, [C `conflict.js:177-232`]). Two clerks editing different custom fields of one voucher
  produce a recorded mismatch and one document wins. A server-side three-way merge of `udf` maps would need the base version, which `classifyPush` does not fetch; recorded as not built, with its reason.

### 9.6 Release order (the constraint that needs the owner)

1. Wave 3 commits.
2. **CF-6a (backend rule, `caps`, the `current` read, REST keep, the definition-delete rule) is built and deployed first**: it needs the VM, whose SSH and admin passwords the owner supplies at session start (HANDOFF section 2; the VM runs the `a0fffd6` backend and Phase 27's `ui` rule is still waiting to be deployed too, so **one backend deploy can carry both, plus the capability-keyed company-row rules of AD.1.2 (`CAP_KEPT` for `entryPolicy`, `OMIT_KEPT` for `ui`), reworking the entry built from the first text**). A **deploy-time smoke** closes the gate, because the suites cannot (`loadHistories` needs Postgres): against a throwaway tenant on the VM's own database, push a voucher carrying `udf` from a device that states `udf2`, then push it again from a device that states nothing with a base at or past the first push, and read back that `udf` survived; **and push the company row twice, once from a device that states `udf2` (changing `entryPolicy`: it must apply) and once from a device that states nothing, with a base at or past the current row (the policy must come back)**. The owner flips the release only after it passes.
3. **CF-1 (masters) is not "safe at any time"** (the first draft said so; it is false: A, item 12b). Safe before the deploy: a **local-only** company, and in a synced company a definition for which `needsServerCap` is false (a text or choice field on ONE master place), which an older app owns and round-trips without loss. Everything else (more than one place, a number, date, yes/no, amount, percent, multi-choice or lookup type, any voucher, line, employee or lead place) needs step 2, and **the editor refuses it in a synced company while the pull answer lacks `udf2`** (7.4).
4. **CF-2 and CF-3 (voucher and line fields) must not reach a synced company before step 2, and must not reach any customer before CF-4a and CF-4b-lite**: a company that can tag vouchers it cannot yet filter or summarise is only collecting data it cannot use, and the owner's headline case (filter the Sales Register by Zone) is CF-4b-lite, which is built now and does not wait for the grid (CF27). The same field as a **column and a sort** arrives with CF-4b after the grid; the release note says so. For a local-only company there is no older copy; a backup restored onto an older install is stopped by the schema-3 marker (12), which the first draft dismissed as "no risk".

The `caps` pull answer lets the editor check this at runtime, so a mistaken order is refused rather than silent.

---

## 10. Deletion and hiding

```
active ──Hide──▶ hidden ──Show──▶ active
active | hidden ──Delete (no use here, fresh sync)──▶ deleted   (a tombstone: the row stays)
deleted ──Restore──▶ hidden
```

**A definition is never removed from the `customFields` table, and an option is never removed from `opts`.** "Delete" is a state. The first draft decided "zero use" on the local replica, which a device that has not synced can get wrong (it offers Delete for a field other devices have filled), and an older app's editor deletes through `removeDoc`, which has no guard at all (`checkDelete` has no custom-fields case): the row would vanish everywhere with its values still on the vouchers, unlabelled, unfilterable and unhideable, and the same field id could not be recreated. A tombstone keeps the label and the id, so a value that arrives late is still labelled, filterable and reportable, and no id is ever reused.

| Operation | Rule |
|---|---|
| **Hide** | always allowed **except for a field that tracks stock** (a hidden tracking field would book every later line as Not set: it must be stopped first, AD.3.2); asks "Hide <label>? Values already entered are kept and stay in reports. It will no longer be asked on new entries." Hidden fields are not drawn on entry screens, are offered under "Hidden fields" in the report picker, and still resolve in saved views |
| **Delete** | offered only when (a) no record on this device holds a value, **and** (b) **the field does not track stock (Stop tracking comes first, AD.3.2), and** (c) for a synced company the device has nothing waiting to send (`status.pending === 0`) and has pulled within the last minute (`status.lastSync`, `sync/sync.ts`), else "Sync first, then delete". It writes `status: 'deleted'` and does NOT remove the row. A deleted field is not asked, not offered in a picker or a filter, and does not count toward the 200-definition cap (a hard ceiling of 1,000 rows including tombstones), but it **still resolves labels** (a saved view, a filter chip, a value that arrives late) and is listed under **Deleted fields (n)** with **Restore** (to hidden). With use it is not offered: "Used by 412 vouchers and 9 stock items. Hide it instead." `checkDelete` gains a `customFields` case that **always refuses a hard delete** ("Fields are hidden or deleted from Custom Fields; the row is kept"), so every caller of `removeDoc` is held to it |
| **Older editors** | an older app's editor hard-deletes a v2 definition; in a synced company the **server keeps it as a hide** (9.3, Definitions); a local-only company has no older copy |
| **Lookup targets** | a `lookup` field stores a master id, so `checkDelete` for ledgers, stock items, godowns, cost centres and employees also refuses to delete a master that a lookup value points at (via `usageOf`), Data Check reports a lookup whose master is gone, and a report cell for it reads "(deleted master)" (CF-8) |
| **Type** | **never changes after creation** (Pipedrive's rule [W 11b]); "hide it and make a new one" |
| **Places** | may be added; a place may be **removed only at zero use in that place**; narrowing a voucher scope (dropping a base or a type) is always allowed, because scope decides entry and never history (CF18: a voucher that holds a value keeps being reported under it) |
| **Options** | "Delete" marks the option `gone: true` on the same terms as a field (no use here, fresh sync); otherwise hide (1.4). The label is kept; the editor lists it under "Removed options" |
| **Force-delete** | **not in v1.** Zoho's removes the field and all its data permanently [R]; Pipedrive's delete removes the data [W 11b]. A later "Erase values" administrator tool would require a backup confirmation and one batch undo, like Fill blanks |

**Usage** (`usageOf(defs, data) → Map<fid, Usage>`, pure, one pass over vouchers, lines and masters, memoised on the arrays' identity): counts of vouchers, lines, ledgers, items, employees, leads and per-option
counts. It reads the **local replica**; the screen says "on this device" when the company is synced (a device that has not synced may hold more), which is the same honesty as the Data Check (and why Delete on a synced company needs a fresh sync). It also counts lookup references and documents whose bag is past 64 keys, and it also powers Pipedrive's
"field statistics" idea [W 11b]: created, last changed, how often it is filled, whether a saved view uses it.

**Fill blanks (CF12)**, `core/customFieldFill.ts` (pure) and `ui/admin/cf/applyFill.ts` (the writer, modelled on `ui/reports/markOptional.ts`):

```ts
export type FillSource = { kind: 'option'; id: string } | { kind: 'party' };         // a fixed value, or the party's value of the same field
export interface FillPlan {
  fid: string; source: FillSource; scope: { bases: VchBase[]; typeIds?: string[]; from: string; to: string };
  ids: string[]; skipped: { noParty: number; partyHasNone: number };
  perOption: { optionId: string; count: number }[];                                  // how many vouchers each option would get
}
export function planFill(def, source, scope, data): FillPlan;        // pure: the vouchers in scope that hold no value; counts and a preview, nothing written
```

**What the first draft got wrong, and what is specified instead.** (1) It said the whole fill is "one Ctrl+Z step" through `origin: 'flag'`. `core/undo.ts record()` hands a flag row to listeners and never puts it on the undo stack, so the global Ctrl+Z would do nothing: a flag batch owns its undo, as Mark Optional's register bar does. (2) It said `updateDoc` makes the batch one unit. `updateDoc` opens one transaction per document; atomicity comes from an outer `db.transaction`, as `applyMark` does. (3) The fill writes whole voucher documents from the local replica, so on a device that has not pulled it can push stale documents over another device's edits (last push wins, and past 2,000 open mismatches the overwrite is not even recorded). (4) The "from the party" source applies the party's CURRENT value to last year's vouchers, which the preview must make visible.

- **Gate.** In a synced company the Fill is offered only when `status.pending === 0` and the device has pulled within the last minute (`status.lastSync`); otherwise the panel says "Sync first". It is an administrator action (8).
- **Preview.** Counts, a sample of ten vouchers and, for the party source, **the count per option** ("North 12,400 · South 9,100 · East 3,000 · West 800 · party has none 1,100 · no party 40"), because a fill from the party's current value restates history by today's master.
- **Apply.** `applyFill(cid, plan)` works in **chunks of 500 vouchers**, each chunk one outer `db.transaction('rw', db.vouchers, db.outbox, ...)` with `putDoc(..., { origin: 'flag' })` inside it (the `applyMark` shape: re-read each voucher inside the transaction; skip and count one that now holds a value, one that is optional or cancelled, and one that is gone). A chunk is atomic; a refusal or a crash between chunks leaves the completed chunks written, and the panel says "Filled N of M; stopped". It writes only `udf[fid]`, never another field, never an amount, and it collects its entries with `onMutation` exactly as `applyMark` does, returning a `FillBatch`.
- **Undo is the fill's own bar, not the global stack.** The result panel shows "Filled N vouchers · Undo", and Ctrl+Z means "undo the fill" only while `batchIsLatest(batch)` holds (no ordinary step recorded since), as for Mark Optional. `undoFill(batch)` restores a voucher **only if it is still exactly what the fill wrote** (`sameDoc`), one transaction per chunk, and counts "edited since, left as they are"; it never overwrites later work.
- **Stamped as the administrator's own action** (not `system: true`, which stamps "System" and hides it from undo and the audit); the audit log shows the fill. It is below the store (3.3) and only ever sets a value, so it needs no exemption.

It is bulk **tagging**, not bulk editing of amounts; the wave-a2 plan's "never bulk edit vouchers" is about the latter. This is the answer to "retroactive tagging is expensive" [R zoho-reporting-tags 3.4] and to T1's "a way to fill blanks in bulk". It is also the companion of the `always` entry policy (AD.1): complete the old vouchers first, then bind them (the Workspace panel and the field editor both link to it).

---

## 11. Import, export, print, API

| Surface | Rule |
|---|---|
| **CSV Import Data and the mapped importer** (`importMap.ts` `FIELDS`, a closed record today) | each applicable field is offered as a mapping target `cf.<key>`; a value is parsed per type: a date by `parseDateInput` **without mnemonics** (the Phase 20 to 21 gotcha: a mis-mapped column holding `R` must not import as the year end), a number with arithmetic only, a choice by option id or by label after folding; **an unknown option is a row problem** ("Zone: 'Nort' is not in the list (North, South, East, West)"), never silently created; a required field missing is **counted in the preview and imported** for vouchers (imported vouchers are history, 3.3), and a row problem only when the "Enforce mandatory fields" tick is on or the import is a master import |
| **Bulk Entry** | a column per applicable field, the voucher screen's severity (3.3) |
| **Backup, restore, Split Company Data** | whole rows: custom values and definitions travel; **`BACKUP_SCHEMA_VERSION` becomes 3 with a no-op step 2 to 3, and `backupHeader` writes 3 only when the company holds a v2 definition (`v: 2`) or any voucher, line, employee or lead `udf`, or a `countPart` line** (12), so a company that never used the feature restores in an older app as today and one that did is refused by it ("made by a newer version") instead of losing values on its next save. Split re-keys ids by value: a lookup value is a master id and is re-keyed correctly, a choice value is an option id that no voucher id equals |
| **Report CSV, Excel, Print** | follow the displayed columns (the shell reads the table); a custom cell starting `= + - @` is neutralised by the single writer `core/csvOut.ts` like every cell |
| **Printed voucher** | per field `use.print` (default **off**) and a new `PrintPart 'custom'` ("Custom fields", `layout: 'both'`, default **on**, so turning a field's Print on prints it unless a template turns the part off). Header fields draw as a "Details" line of label-value pairs under the buyer block on an invoice (and under the heading on an accounting voucher); line fields as **a sub-line under the item name** with Batch and Description (`invoiceModel.ts` already builds `sub` lines, so the column plan and its millimetre budget do not move). `PrintPart`, `PRINT_PARTS`, `DEFAULT_SHOW` each gain the member and `print.assert.ts` is updated in the same change. **With no printable field the invoice model is identical to today** (deep-equal). **The Print Configuration form shows the "Custom fields" row only when at least one active field has `use.print`, and places it BEFORE Signature block**: the form builds one toggle row per part inside its Enter walk, `signature` is the last, and a new row after it would steal Accept-on-Enter from every company (the Phase 24 gotcha: new fields go last in the *walk*, or out of it); so a company with none sees today's rows in today's order, and the identity suite compares the form's field list with a recorded baseline |
| **GST returns, e-invoice, e-way** | never. A source scan fails if `core/gstr1.ts`, `core/gstr3b.ts`, `core/gst.ts`, `core/einvoice*.ts`, `core/ewb*.ts`, `core/dispatch.ts` or `core/tds*.ts` import the custom-field modules or read `.udf` (T15, CF14) |
| **Tally XML export** | **not written in v1**; the exporter keeps listing "masters with custom fields" among what the file does not carry. CF-10 writes them as UDF tags with `tallyIndex` (assigned once, at the first export, from a **free index**: the exporter reads the UDF indexes already present in the target file when comparing with Tally, or the owner supplies the starting index, and never hard-codes 20001, because a partner's own TDL for the same customer may already use 20001 upward and two meanings would merge into one stored slot on import; Tally keeps 1 to 29 for its own default TDL and 10,000 to 20,000 for shared development [R tally-udf 2.1]; the index is stored on the definition and never recomputed) |
| **Tally XML import** | the importer drops what it does not read (header of `tallyImport.ts`). CF-10 reads namespaced UDF tags and their indexes, previews "6 fields found, map to new custom fields", and fills values. **Gate: three real TallyPrime exports from partner customers who run UDFs** (a garment trader, a distributor, a pharma stockist), because the tag shape is **[unverified]** and nothing here may be built on a guess (question 6) |
| **REST, webhooks** | values ride on the document under `udf`; definitions readable at `/v1/customFields`; `null` clears a key and an omitted `udf` keeps (9.4); `ApiDocs.tsx` gains the paragraph |
| **Go To and the palette** | no new command that could re-rank a query (the 103-query golden `goto.capture.mjs` is the proof: any new entry is `late`) |

---

## 12. Migration, Dexie, backup

- **No Dexie bump.** Definitions stay in the existing `customFields` table with extra properties; values are optional properties of existing documents. A new table would be version 9 (additive, like v6 to v8) and
  would add the eleven registration points and six drift suites of audit 6.3 for nothing the model needs. **The one thing that would force a bump** is a decision to keep values in a side table keyed by
  voucher id (rejected: every report would need a join and the older-app problem is solved by the server rule, 9.3).
- **The backup schema goes to 3, conditionally (CF5).** `core/backupMigrate.ts` says a file whose `schemaVersion` is above the build's is "refused outright: an older app silently dropping fields a newer one wrote would lose data on the next save", and that refusal is the only guard on the restore round trip. This feature introduces the first row fields an older voucher form DOES drop on its next alteration (`Voucher.udf`, `InventoryLine.udf`), so opting out of the guard (the first draft, with "no risk" for a local-only company) was wrong: a company backed up on one PC and restored on a second that still has the previous installer would lose every Zone on the first alteration there. `BACKUP_SCHEMA_VERSION` becomes 3, `MIGRATIONS` gains a no-op step 2 to 3 (nothing is invented or rewritten), and **`backupHeader` writes 3 only when the rows being written hold custom-field v2 data** (a `customFields` row with `v: 2`, or any `udf` on a voucher, an inventory line, an employee or a lead, or any `countPart` line, which implies a v2 tracking definition anyway), else it keeps writing 2, so a company that never used the feature restores in an older app exactly as today. `backupmigrate.assert` is extended on purpose: its "an export carries the schema version" check (`file.schemaVersion === BACKUP_SCHEMA_VERSION`) becomes "carries 2 for a plain company and 3 for one with v2 data", its "one step per version" check counts the new step, and a case proves an older build's reader refuses a 3-file.
- **The v1 reader** (1.6) is the whole "migration": no data is rewritten at any time.
- **`UdfValues` widens** to include `string[]`; `collectUdf`'s old return type is replaced by `mergeUdf`'s, a compile-time sweep of its two callers (`masters.tsx`, `inventory/masters.tsx`).

---

## 13. Performance budgets

The perf harness bundles the real engines and refuses any that import `src/ui/`, `src/state/` or Dexie; under `--enforce` it fails on drift of its mirrored source lines [C wave-a2 plan 0.1 item 2].
So every file in lane M is a pure `core/` file, and an edit to a mirrored line in `VoucherScreen.tsx`, `store.ts` or `reports.tsx` updates `scripts/perf/mirrors.mjs` in the same change. **All numbers
below are plan inputs [J], to be replaced by the first measurements; the new rows are reference rows, not gated** (the wave-a2 D13 decision).

**First measurements (2026-10-24, build review; `npm run perf -- --sizes=250k --only=cf`, min / p50 ms, on a machine at load 25 to 50 on 10 cores, so absolute numbers are pessimistic and the ratios are what to trust; an earlier run at load 40 to 300 was 2 to 10 times worse).** The plan numbers above were inputs; where they are not met, the measured figure stands here and the plan figure is NOT claimed:

| Row | Plan | Measured at 250,000 vouchers | Met? |
|---|---|---|---|
| `cf.summary` (voucher grain) | 200 | 156 / 161 | yes |
| `cf.summary` legs (party legs) | 400 | 126 / 133 | yes |
| `cf.summary` line grain | 400 | 217 / 373 | yes (p50 close) |
| `cf.summary` orders (`pendingOrders`) | 400 | 41 / 59 | yes |
| `cf.summary.stock` (Stock vouchers family) | 800 | 740 / 1,127 | min yes, p50 no |
| `cf.bills` (Bills by field) | 250 | 1,326 / 2,430 | **no**: `outstandingBills` alone is about 300 ms at this size, so 250 ms cannot be reached by construction; the working figure is about 1.3 s |
| `cf.bal` (Balances by field) | 250 | 288 / 335 | close (a `closingBalances` pass plus the union set) |
| `cf.pl` (Income and expense by field) | 500 | 1,378 / 1,834 | **no**: working figure about 1.4 s |
| `cf.pl.item` (an item source) | 800 | 2,329 / 2,557 | **no**: working figure about 2.3 s |
| `stockPositions` with a variant resolver against the plain call | 1.3 x | 1.8 to 3.3 x over three runs (the builder saw 1.17 to 1.67 x) | **no**: the working ratio is about 2 x; a company with no tracking field pays nothing |
| `cf.stock.flow` | (none) | 3,466 / 3,965 | reference only |
| The Stock by field screen table | (none) | 5,294 / 6,775 | reference only |

What the screens do about it: the Analysis screens hold their books while another screen is on top (`useLiveWhenOnTop`, `ui/reports/fieldShell.tsx`), so a covered screen computes nothing; Bills by field, Income and expense by field and the Stock by field screen are the three whose first paint at 250,000 vouchers takes seconds, and the next step if an owner with a company that size asks is a progress line while they run and a per-period memo of the whole-book pass (consider also caching `registerAmount`, the per-invoice `voucherGstSummary`, for the voucher-grain Summary), not a change of what they compute. **Rows at 50,000 vouchers were not re-measured in this review** except `usageOf` (35 ms against the plan's 150). The 250,000 rows were measured under load and should be re-run on an idle machine before any number here is used as a promise.

**The item grid with line fields, per key (build review, 2026-10-24; DECIDED NOT TO CHANGE YET).** A long invoice with six line fields (two inline columns and four sub-row cells per line) draws 2,485 DOM nodes on a 40-line altered Sales voucher against 983 without them, and every key re-renders the whole grid: keydown to paint over Enter and 24 arrow presses read p50 4.4 to 7.7 ms and p95 14 to 25 ms in a production build (dev build 9 to 19 and 20 to 42 ms), at load 20 to 30. A 12-field header strip adds nothing measurable (p50 7.2 against 7.5 ms). Both figures are inside a keystroke's budget, so the optional fix (memoise the item rows and the sub-row cells so only the cursor row re-renders) was **not made**: a cell reads its value, its focus, its tags and its handlers through the `LineCfApi` closures that the voucher screen rebuilds on every render, so a memo keyed on the row object alone would hand a skipped cell a stale handler (a click would act on the old row list). The safe form is a refactor of `VoucherScreen` that gives the line API stable, ref-backed handlers first; that is a change to the shared 4,800-line screen and belongs to a wave that owns it. A company with no line field pays nothing (`lineDefs` is the one frozen `NO_FIELDS`; the grid draws exactly what it always drew).

| Operation | Budget | Why it matters |
|---|---|---|
| `normaliseDefs`, `fieldsFor(defs, place, ctx)` | O(definitions), memoised on the array identity; **returns the same frozen `NO_FIELDS` when nothing applies** | the voucher screen calls it on every render; with no definition it must cost one comparison and draw nothing |
| One strip field on a keystroke | no work beyond that field | a voucher is typed all day; the harness's `--cpu-factor` run checks an old PC |
| `checkUdf` per voucher | at most 0.1 ms | Bulk Entry calls it per row |
| A catalogue getter per row (`CfColumn.get`) | O(1) through the lookup context | a 250,000-row Day Book; hidden columns cost nothing (not rendered) |
| `usageOf` | one pass; at most 150 ms at 50,000 vouchers and 150,000 lines; memoised | the editor opens it |
| `fieldSummary` | one pass over `registerRows`; at most 200 ms at 250,000 vouchers for any voucher-grain family; **line grain, party legs and the orders' `pendingOrders` pass at most 400 ms** (AD.2.5); **the Stock vouchers family needs a full-history stock walk (a count's variance depends on every earlier movement), at most 800 ms** | reference rows `cf.summary` and `cf.summary.stock` |
| `fieldPl` | one pass over vouchers and entries; at most 500 ms at 250,000; **an item source adds one `stockPositions` pass (as the P&L already pays for its own stock), at most 800 ms in all** | reference row `cf.pl` (the Dashboard's whole default board is about 718 ms at that size [C CLAUDE.md Phase 28]) |
| `fieldBills`, `fieldBalances` | one `outstandingBills` **and one `closingBalances`** pass (Bills by field needs both: the bills and what each ledger holds), which the Bills and Group Summary reports already pay; the union of the chosen groups' ledger ids is a set built once; at most 250 ms at 250,000 vouchers | reference rows `cf.bills`, `cf.bal` |
| `stockMovements` with the count-group merge | one extra scan of a Physical Stock voucher's lines for `countPart` (O(lines of Physical Stock vouchers)); nothing for any other voucher | every stock figure reads this generator, so the merge must cost nothing for a company with no count by variant: the 50,000-voucher deep-equal and the perf check run on it |
| `stockPositions` with a variant resolver; `stockByVariant` | at most **1.3 x** the plain call at 250,000 vouchers (the resolver adds one key per movement of a tracked item and nothing for the rest; with no resolver the call is identical) | reference row `cf.stock`; a company with no tracking field pays nothing |
| `checkUdf` with a policy, `modeFor` | O(fields in scope); at most 0.01 ms beyond the 0.1 ms above | the policy is read once per mount, never per keystroke |
| A `udf` bag | at most 8 KB, typically under 200 bytes | every saved voucher and every change-log row carries it |
| Server `current` read (CF-6a) | one indexed `= ANY` read per push from a writer without `udf2` (**including a company-row push, AD.1.2**); at most 300 documents per push; `carryUdf` per document O(keys + lines), **and the strong-identity pairing O(lines) with a map keyed by the strong identity (no nested scan)** | the price of protecting older apps; zero for a current client; an estimate [J] |
| Standard layouts | **unchanged**: the A2-0 table goldens and the Go To golden | a field that is defined but not chosen changes no pixel of a report |

---

## 14. The assertion plan

Every suite is auto-discovered (`scripts/assert/run.mjs` bundles every `*.assert.ts`) and **ships a deliberate-mutation list, each killed** (the Phase 25 practice).

| Suite (new) | Proves | Deliberate mutations it must kill |
|---|---|---|
| `customfields` | `normaliseDef` reads every v1 row; `valueKey`, `reportFieldId` (property: at most 40 characters, matches `SAFE_FIELD`, unique within a place and within a report, stable across `normaliseDef`; fixtures: a ledger and a stock item sharing the v1 key `region`, a 43-character key, `__proto__` and `constructor` as keys and as option ids); **`placesOf`, the written `objectKind` and `needsServerCap` for every shape of 1.1, and an older `udfDefsFor` never lists a `'v2only'` row or a non-lossless type**; option semantics (rename keeps the value, hidden stays valid, delete marks `gone`, **a new label may not equal an existing id or label**, **legacy id equals label**, the three-step reader, every per-option structure a `Map`); `coerce` per type (`0`, `false`, ISO date, exact numbers, no magnitude shorthand on `number`, shorthand on `amount`, `url` schemes); hostile input (`__proto__`, a one-megabyte string, bidirectional and zero-width characters, `Object.create(null)` bags) and **linear time** on a million-character string; the cleaner strips the same code points as `uiPrefs` `cleanLabel`; every cap of 1.5; `udfView` never writes back | id from the lower-cased label; length gate after the pattern test; store `''` for a cleared field; drop `0`; rename rewrites the id; random legacy ids; one `defs` map shared across places; a plain-object option count |
| `customfields-rules` | the 3.2 table and the 3.3 matrix: `req: 'new'` does not bind an old voucher, `'always'` does at the entry screens and not at the store, `'policy'` follows the company setting (AD.1.3; the policy argument is passed, never read from storage), a v1 `required` is not enforced; **`exempt` skips only `checkUdf` (an operator locked out of a period is still refused when cancelling; cancel, Ctrl+Z, delete-undo and restore all succeed with a mandatory field defined)**; **an F10 change into a base where a field newly applies binds as a create**; a changed value is judged and an unchanged one is not; a hidden `dflt` is ignored; a hidden field is not checked; scope matching by base, type and group; **every `saveVoucher(` and `saveDoc(` caller found by a source scan is classified by the 3.3 table, and an unclassified one fails**; `UdfRefusal` is thrown by the store and caught by each non-form caller | an alteration refused for a field the record never had; v1 enforced; `exempt` implemented as `system`; the F10 case judged as an alter; an unclassified caller |
| `voucher-resave` | the 9.1 classification of **every key** of `Voucher`, `InventoryLine`, `Employee`, `Lead`; each writer A1 to A10 carries or drops `udf` as the table says; an alteration with a definition list that lacks a key keeps the key | remove A1's carry line; `validInvLines` drops a line's `udf`; collect rebuilds |
| `customfields-identity` | **a company with no custom fields sees today's app, by identity** (the claim of 4.2, which covers entry screens, documents, print and standard layouts, not the navigation additions, AD.7): the saved `Voucher`'s key set per base equals a recorded baseline; `fieldsFor` returns the `NO_FIELDS` reference; `CustomDetails` renders `null` for it; the narration handler has no new branch when W is empty (source scan of `case 'narr'`); **`case 'item'` has no Tab branch when the base has no line field (source scan)**; the `signature()` and `typedNotEntered` are unchanged; the voucher walk with the new keys removed equals the frozen sequence; the ledger form's field list equals the baseline; **the Print Configuration form's field list equals the recorded baseline when no field has `use.print`**; `buildInvoiceModel` deep-equals; the `blankOk`, three-state toggle and `percent`/`amount` flags of `forms.tsx` appear only in `ui/udf.ts`; the v1 profile writes the bytes the older `collectUdf` writes; the Go To golden is unchanged | draw the strip with no fields; default `udf: {}` on a voucher; a new branch on Enter; a Custom fields row in the Print Configuration form for a company with no printable field; a Tab branch with no line field |
| `customfields-report` | ids pass `sanitiseFilter` untouched; the catalogue offers only what a report can compute (5.4); a `text` cell uses the current label; a stale option matches nothing and a deleted field's column is dropped while its condition is kept, and the chip names it through the label resolver; the line projection (one value, "Multiple", blank); the Day Book of mixed types is blank outside scope only for vouchers with no value; `total` rules (a party column is `none`, a number not marked additive is `none`, a percent never sums); `isNot` matches an unset row and `is No` an explicit No only; **`fieldSummary` reconciliation per measure (5.5), header and line grain, on the sample company and a 50,000-voucher synthetic one** (amount rows plus Not set plus Not asked equal the **signed** total to the paisa; line-grain count: the Total is the distinct vouchers; optional vouchers excluded with and without a filter; the drill's filter equals its row); **narrow the field's scope, then re-run: a voucher holding a value stays under its option**; a Sales-with-Purchase line summary is refused | drop the Not set row; bucket by scope before value; treat a credit note as positive; sum the line-grain count rows into the Total; sum a party column; store a label in a saved view; share one `defs` map across places |
| `customfields-pl` | `attributeEntry` rules R1 to R6; **the tie to `plFigures` nett** for a header field, a line field and a two-type scope, on a company with a loss, with a stock change, without stock items and the 50,000-voucher synthetic; **fixtures for R4: a Tally-imported voucher with `itemLedgerId` unset (one candidate, and several candidates), an extras row on the item ledger, and the narrow-the-scope-then-re-run case**; largest-remainder splits sum to the entry exactly; a credit note reverses inside its option; `diff` is computed, not assumed | spread charges silently; attribute an Assets entry; skip the guard; round per line; read an unset `itemLedgerId` as "every entry is a charge"; attribute by `notAsked` before value |
| `customfields-sync` | **I/O level:** `loadHistories` (exported, or split into a pure planner and a thin executor) is driven through a fake `client.query` that answers each tagged query shape from an in-memory change log. With `others` empty (the writer's base at or past the newest foreign row) and `latestOther` never loaded, a `put` of a voucher without `udf` from a writer lacking `udf2` still comes out with the current `udf` (the case the first draft could not reach); no `current` read is issued for a writer that states `udf2`. **Pure:** `classifyPush` with and without `udf2`; the header union; the line rules (i) to (iv) of 9.3 incl. **the strong-identity pairing: a size run of three lines of one item plus an appended line keeps all three variants, one edited quantity keeps all three, the first of three deleted keeps Blue and Black on Blue and Black (never Red and Blue), and two lines identical but for the variant with one deleted come out unlabelled and recorded, not scrambled**; `countPart` is restored with the line it belongs to; **the company-row capability rule (AD.1.6 (5)): a writer that states `udf2` is applied, one that states nothing is kept, with no base, a stale base or a current base, and `ui` is restored only on omission**; inert when `current` holds no value; a `customFields` delete from a capability-less writer becomes a hide; prototype-named tables are safe; `caps` parsing (not an array, long strings, 100 entries); `udfKept` is returned **with the mismatch log full** (a fake `upsertConflicts` that caps); `carryUdf` in the REST route inside the lock. The deploy-time smoke of 9.6 is the Postgres-level proof | apply the rule to a current client; match lines by index only; clear on omission; read `latestOther` only; drop `udfKept` when the log is full; skip the line identity check; **pair a duplicated strong identity; pair by weak identity when two candidates remain; key the company-row keep on "states no base"; apply the keep to a writer that states `udf2`; revert a Phase 27 administrator's `ui` edit** |
| `customfields-print` | a field with `print` prints (header "Details" pair, line sub-line) and one without does not; the `custom` part off hides both; no statutory file imports the modules or reads `.udf` (source scan, with a guard that the scan found the files) | print a field whose switch is off; leak into `gstr1.ts` |
| `customfields-import` | CSV and mapped-import targets, an unknown option is a row problem, **no date mnemonics**, Bulk Entry holds a required field at voucher severity | create an option from an import; enable mnemonics |
| **`entrypolicy`** (AD.1.6) | the policy store and its older-client protection (sanitise, rev, mirror, erased and older, Restore, the one writer, the **capability-keyed** server rule of AD.1.2); **identity** (nothing set is the same frozen reference and the prospective rule, by deep-equal on a generated set); the semantics matrix of about 150 vector rows (`req` x mode x place x operation x record state x date relative to `since`); every door of AD.1.4 through the real callers, cancel, undo and restore succeeding under `always`, the store backstop not refusing an alteration | read the voucher knob for masters; `since` as `always`; test only the stored date; test only the new date; apply the policy on create; enforce an alteration in the store; judge an exempt writer; auto-fill a bound alteration; an erased policy as a reset; a rev below the remembered one; drop unknown fields; a fresh object for "nothing set" |
| **`customfields-dimension`** (AD.2.2) | `dimensionFor` for every type (refused kinds refused), choice order and removed rows, the text kind (NFKC and case fold, length capped before folding, 200 buckets then "Others" with the amounts kept), period and band edges (lower inclusive, upper exclusive, at most 12), the fallback chain, `__proto__` and `constructor` as values, the sort rule of CF28 (a choice by label, Not set last in both directions) | fold before the length cap; drop the Others amounts; a band's upper end inclusive; sort Not set first when descending; a plain-object bucket map |
| **`customfields-bills`** (AD.2.7) | Bills by field: **`diff` against the side's group balance is 0.00** for the receivable and the payable side on the sample company and a 50,000-voucher one, with the fixtures **a debtor with an on-account receipt, a creditor with an advance, a non-bill-wise debtor, an overpaid New Ref on a debtor, a bill-wise ledger whose opening is not itemised, a bill-wise ledger in another group**; a bill's side is its ledger's group, **an on-account receipt does not make a customer a payable** (it is in Unapplied); **Pending equals the Dashboard card's `current + overdue` and Unapplied plus Not bill-wise equals its Unapplied**, to the paisa, on the same `asOn`; the age clock is `ageingAsOn`, never the future; **equality with the Bills report's total is asserted only on a book where every bill-wise ledger sits under the side's own group and none holds an entry of the opposite sign** (an oracle written from the report's sign rule), and the memo's counts equal a by-sign count on the fixtures; Balances by field ties to the union of the chosen groups' closing balances and to `groupBalance` when the choice does not overlap, **and a choice of Current Assets plus Sundry Debtors counts no debtor twice**; a ledger with no value is Not set | side by sign; age against the period end; drop Not set; sum on-account into pending; **promise `diff` 0.00 against the Bills report; send a negative named bill to Pending; drop the Not bill-wise column; sum `ledgersUnder` per chosen group without deduplicating** |
| **`customfields-stock`** (AD.3.9) | the variant engine: identity with no resolver, every existing position field identical with a real one **on a company that has counts by variant** (pooled costing; the count group is read from `countPart`), the five Physical Stock vectors read through the Stock Summary, the Balance Sheet, Stock Ageing, Godown-wise and Stock by field, conservation and the invariant after every movement, allocation to the paisa under all eight costing methods (a negative variant gives Unallocated), every voucher type of AD.3.4, the writers of AD.3.4b, the two opening-alloc trust tests, pending orders by variant (blank order lines, blank invoices, no double subtraction, Split Company by `line`), the variant negative-stock warning, the screen and Stock Summary group mode tie at 2k and 50k | key valuation by variant; **resolve a count against the variant balance; read `countPart` only with a resolver; merge across godowns or vouchers**; store the key on the line; trust the alloc without either check; allocate by value or with a negative variant; drop the residual; a key for an untracked item; match a pending order across variants or a blank order line only exactly; **allow Hide on a tracking field**; a value on the (variant, godown) level |
| **`customfields-filterlite`** (CF-4b-lite) | with no definitions `filterFieldsFor(key)` returns the same array reference and every static id keeps its kind and position; with definitions the custom ids are appended after the last static id, in definition order; the `cf.`, `cfp.`, `cfl.` getters; stale and deleted fields get synthetic metas ("(not in this report)", "(removed option)") so a chip never prints a raw id; `sanitiseFilter` is untouched | reorder a static id; prepend custom ids; print a raw id; key a getter by label |
| `customfields-usage` | `usageOf` counts (incl. lookup references and bags past 64 keys); `checkDelete` refuses every hard delete of a definition and a lookup target in use; Delete writes a tombstone and a tombstone still labels; the type is immutable; a place with use cannot be removed; options merge by id and never drop (a stale save keeps an option it did not see); `planFill` touches only blanks and reports its skips and per-option counts; **the Fill is chunked and atomic per chunk, `undoFill` restores only documents still identical to what the fill wrote and counts the rest, and the global Ctrl+Z does not reverse it** | hard-delete a definition; delete on a stale device; fill a non-blank; undo over a later edit; drop an option on a stale save |

**Widened by the addendum:** `customfields-rules` takes the policy argument (the 3.2 table is the AD.1.3 table); `customfields-report` covers **every family and source of AD.2.5** (the party legs of a receipt and a payment, orders' pending measures, quantity by unit, `net` against `asRegister`, the fallback source, the reconciliation per measure at 2k and 50k) with the added mutations "apply `baseSign` to a note alone", "count a party leg's remainder twice", "drop the No party row", "sum a quantity across units"; `customfields-pl` covers the five sources and R7 to R10 with the added mutations "attribute the Change in stock by value instead of by item", "read the party's value from the voucher's `itemLedgerId`", "guess a party for a journal with two", "attribute an Assets entry through the account source".

**Widened again by the third review (2026-10-22).** `customfields-report` and `customfields-pl` add: **`partyOf`** (an imported Payment with `partyId` and its hand-entered twin bucket identically in Summary and in R7; `partyId` is honoured only for `baseInfo(base).invoice`; the mutation "honour `partyId` on a Payment"); the **Stock vouchers** family tie to the stock engine (a valueless Stock Journal issue line reads at cost as in the Stock Summary; a Physical Stock count is its variance over a full-history walk; a base filter; the mutation "take Value out from `line.amount`"; with a tracking field only the quantity measures are offered); **`fieldPl` on a company that has an Expenses-nature group outside the six reserved groups: `diff` is 0.00 and the amount is the "Outside the statement" memo row** (the mutation "attribute by group nature instead of by the six subtrees"); and **`stockDeltaByItem` equals `stockDelta` bit for bit** on a company with openings, one with Zero-cost items and one whose opening rates truncate (the mutation "read the opening from `stockPositions`").

**Extended deliberately** (the commit message names the lines and why): `inventory.assert` (the variant identity and conservation lines, **the count-group vectors and the `stockMovements` deep-equal for a voucher with no `countPart`**, AD.3.9), and a **`stockageing` golden** (the lot walk of `ui/inventory/ageing.tsx` moved UNCHANGED into a pure `core/stockAgeing.ts` that the component calls; the golden proves the move on the sample company before and after, and the count-group vector reads **a count group as one movement there**), `stocksummary.assert` (the group mode, deep-equal at the default; `stockFlow` variant cells), `split.assert` (`openingAlloc.variants` round trip), `master-resave.assert` (an alteration keeps an orphaned key; the v1 profile equals `collectUdf`; **`Company.entryPolicy` is "off-form, preserved"; `StockItem.openingAlloc.variants` and `InventoryLine.udf` are classified**), `gst-identity.assert` (custom fields are `tail`, and required ones are walk fields), `navstack.assert` (the voucher `useDirty` literal: a typed cell is unsaved work; **`customFields` joins its wired list, and the editor's probe is true for a changed draft, a pasted option list, a Starter preview and a Fill preview, false for an untouched panel**), `crosscut.assert` (the signature mentions the cell signature), `sidenav.assert` (66 rows), `quickcreate.assert` (one row), `reportscenter.assert` (the catalogue rows; `EXCLUDED_REPORT_SCREENS`), `security-access.assert` (`fieldSummary`, `fieldPl`, `fieldBills`, `fieldBalances`, `fieldStock`), `syncconflict.assert` (`UDF_CARRIERS`, the `customFields` delete rule, **the `CAP_KEPT` rule for `entryPolicy` and the `OMIT_KEPT` rule for `ui`**), `print.assert` (the part; the Print Configuration baseline), `reportfilter.assert` (the ids; the `describeCondition` resolver), **`backupmigrate.assert` (schema 3 conditional, the no-op step, an older reader refuses a 3-file)**, `visual.assert` (the strip's tag colours and `aria` contrast pairs, both schemes, eight accents). **Untouched, and the proof that no chord was added:** `zoho-scaffold.assert`, `shortcutprefs.assert`, the 103-query Go To golden.

---

## 15. Recipes (`scripts/shots/recipes/customfields.mjs`; real keys, never Accept on a live company; each in light and dark, compact and comfortable, and at 150 % text)

| Recipe | Steps |
|---|---|
| `cf-editor` | create Zone (choice, party and Sales) with only Name, Type and Options (Ctrl+A saves), options by paste, preview matches the strip, hide, the delete refusal with a count, the v1 conversion banner; **Back or a rail click with a changed draft asks "Leave this entry?"; the editor stacks below 1100 px** |
| `cf-master-walk` | a ledger with an optional field: Enter on the last built-in field raises Accept (tail); with a required field Enter walks into it |
| `cf-voucher-walk` | the old keystrokes for a whole Sales voucher in a company with only an optional field: nothing moves; with a required field: Enter after Narration walks into it, Ctrl+A with it empty is refused and the cursor lands on it |
| `cf-inherit` | pick a party with Zone North: the cell reads "North (from party)"; change the party; type East, change the party again: East stays |
| `cf-line-column` | Colour as a line field: at 1280 px the first two line fields are inline columns and Tab on Amount reaches the first; at 1024 px they are on the sub-row and Tab still reaches them; Enter on Amount does not (not in walk), required puts it in the walk; **in a company with no line field Tab on Amount lands on Narration exactly as today**; the item's colour is the start; editing the item later leaves a saved line alone |
| `cf-report-picker` | Day Book and Sales Register: the field is in the Columns panel under "Your custom fields" with its sub-label, not in the standard layout; filter by it; "Total (filtered)"; save a view; the Day Book of mixed types is blank outside scope |
| `cf-summary-zone` | Summary by field: Sales by Zone, totals equal the register, drill to the register with the filter visible |
| `cf-pl-by-field` | Income and expense by Zone, the Change in stock line, "Agrees with the Profit and Loss statement" |
| `cf-print` | Print on: header "Details" pair and line sub-line appear; with the part off they do not |
| `cf-fill-blanks` | preview with per-option counts, apply in chunks, the result bar's Undo reverses all and leaves a voucher edited since, **the global Ctrl+Z does not reverse it**, a synced company with rows waiting to send is refused |
| `cf-old-client` | with the mock cloud (`scripts/shots/flows/_mockCloud.mjs`): an older-app save of a voucher keeps its `udf` **when the older device had already pulled the newer version** (the case the first design missed), an older-app edit of a voucher with a duplicated item comes back unlabelled and the Mismatch Report says so, **an older-app edit that appends one line to a Sales invoice holding a Red, Blue and Black size run keeps all three variants (strong identity)**, `udfKept` is shown, **and a company-row save from a current client (signed out, no base) changes the entry policy while one from an older client does not** |
| `cf-geometry` | the strip (12 fields) and the item grid with line fields at 1024 and 1280 wide and at 150 % text, **measured against the no-field baseline**: the table's `scrollWidth` never exceeds its baseline, no cut label, no text drawn over text (the check the Phase 27 matrix lacked) |
| `cf-mouse-only` | enter a whole Sales voucher with pointer events only (party, items, Zone and Colour by click, Clear (x), the Yes / No / blank buttons, the rail's Accept); in the editor reorder fields and options with Move up and Move down; open a Summary by field row with its Open button |
| `cf-policy` (AD.1.6) | as administrator set the voucher rule to Always: an old Sales voucher with no Zone is altered, the field is in the walk, Accept is refused with the company-rule wording, "Use North (party now)" fills it; set "from a date" and alter one voucher each side of it; as a non-administrator see the panel read-only; the field editor's three-way choice and its live count; an older-app save of the company row (mock cloud) keeps the policy once the server rule is deployed and shows Restore before it is |
| `cf-summary-families` (AD.2.5) | Summary by field for Receipts by Zone (party legs, the No party row), Credit Notes alone (positive) and with Sales (net), Pending sales orders by Colour, Quantity by unit; each Total equals its register or `pendingOrders`; drill with the filter visible |
| `cf-pl-sources` (AD.2.6) | Income and expense by Zone (header), by the party's Category (party, "current"), by Brand (item, with the exact Change in stock), by a ledger's Cost type (account); every "Agrees with the Profit and Loss statement", **including on a company with an Expenses-nature group outside the six reserved groups (the "Outside the statement" memo row)** |
| `cf-bills-by-field` (AD.2.7) | Receivables ageing by Region with a Not set row and the Pending, Unapplied and Not bill-wise columns, "Agrees with the group balance" at 0.00, the memo line about the sign rule of the Bills reports (on a book with an on-account receipt and a supplier advance), the Payable switch; the Pending drill opens Bills with `cfp.` visible; Balances by field expands to ledgers, and ticking Current Assets with Sundry Debtors counts no debtor twice |
| `cf-stock-by-colour` (AD.3.8, Case 4) | the garment company: Colour and Size track stock; the preflight, Opening by variant, the Stock Journal split, Stock by field (Colour, then Colour and Size, the size-run grid), the Stock Summary group mode by Brand; every total equals the Stock Summary |
| `cf-variant-entry` (AD.3.6) | a Sales voucher in that company: the variant cell is in the walk, the picker shows stock, a sale with none warns "(Red) will go NEGATIVE", a Stock Journal destination starts from the source row's variant, **a Physical Stock voucher for a tracked item pre-fills one row per held variant at book quantity, changing one row moves only that variant, a row deleted by hand counts 0 with the warning, a mix of part and whole rows is refused, and the item quantity then equals the Stock Summary's, the Balance Sheet's and Stock Ageing's**, Hide is not offered on the field while it tracks (Stop tracking is), an item outside the group shows no cell; **with tracking off the voucher is exactly today's** |
| `cf-filter-lite` (CF-4b-lite) | the four filterable reports (Day Book, Sales Register flat level, Ledger Vouchers, Bills) list the custom fields in the filter panel after the static ones; filter by Zone; chips read by label; a stale field's chip says "(not in this report)"; a company with no field sees the panel exactly as before |
| `cf-dimension-bands` (CF-4c) | Summary by a date field (month), a number field (bands typed in F12) and a text field (200 values then Others); the totals tie |

Plus a **live bug-hunt** before the final gate (the Phase 23 lesson: accuracy defects were found by viewing reports on real data, not by review): define Zone and Colour on the sample company, tag a month of
vouchers, and compare every figure of Summary by field and Income and expense by field with the register and the Profit and Loss they open.

---

## 16. File ownership, slices, exit gates, effort

Single writer per file at any moment (wave-a2 rule A2-5). **Lanes:** **M** model (new pure files), **E** editor and masters, **V** voucher entry and print, **R** reports (five new screens and the grid
integration), **S** server, **X** exchange, **I** inventory engine (added by the addendum: `inventory.ts` and its neighbours have one writer too).

| Lane | Files it owns |
|---|---|
| M | `src/types.ts` (the additions of 1.3, one writer), new `src/core/{customFields,customFieldColumns,fieldAttribution,fieldSummary,fieldPl,fieldBills,fieldDimension,entryPolicy,customFilterFields,filterLabels,customFieldFill,customFieldPresets}.ts`, `src/core/backupMigrate.ts` (the schema-3 step), **`src/state/store.ts` for CF-0 only: the `saveDoc` / `saveVoucher` options, `UdfRefusal`, and the `checkDelete` case**, `scripts/assert/customfields*.assert.ts` and their mutation files, `scripts/assert/vectors/udf.json` |
| E | `src/ui/admin/CustomFields.tsx` (+ `src/ui/admin/cf/*`, incl. `applyFill.ts`), `src/ui/udf.ts`, `src/ui/masters.tsx`, `src/ui/inventory/masters.tsx`, **`src/ui/forms.tsx` (CF-1 deliverable: `blankOk`, the three-state toggle, `percent` and `amount`, 2.3)**, **`src/ui/settings/workspacePanels.ts` and the new `src/ui/settings/panels/EntryRulesPanel.tsx` (the entry rules panel, which also carries the "Custom fields..." link), `src/state/entryPolicy{Writer,Mirror}.ts`, and the carry line in `src/core/companyAlter.ts` (CF-14)**, `src/ui/navTree.ts` with the literal table of `sidenav.assert.ts`, `src/ui/quickCreate.ts` (lane E no longer edits `store.ts`) |
| V | `src/ui/VoucherScreen.tsx` (CF-2, then CF-3, then CF-9c: **never two of them at once**), new `src/ui/vch/{CustomDetails,CustomLineCol}.tsx`, `src/ui/admin/BulkEntry.tsx`, `src/ui/inventory/manufacturing.tsx` and `src/ui/payroll/vouchers.tsx` (pass-through lines only), `src/core/voucherCancel.ts`, `src/ui/print/{invoiceModel.ts,PrintVoucher.tsx}`, `src/core/printConfig.ts`, `src/ui/print/PrintConfigScreen.tsx` |
| R | new `src/ui/reports/{fieldShell,fieldSummary,fieldPl,fieldBills,fieldBalances,fieldStock}.tsx`, `src/ui/cf/useCustomFilterFields.ts` (CF-4b-lite); **the shared navigation files** `src/state/store.ts` (the `Screen` union only, once, after CF-0), `src/ui/router.tsx` (route and crumb), `src/core/access.ts`, `src/state/recents.ts`, `src/ui/reportsCenter/catalogue.ts`, `src/ui/overlays.tsx` and `src/ui/modern/CommandPalette.tsx` (late commands): one writer for the whole new-screen checklist, once; **`src/core/reportFilter.ts` for the CF-4b label resolver (CF-4b-lite does not touch it: its chip labels come from synthetic metas, below)**; the grid integration files belong to B-grid until its Gate 1 |
| I | `src/core/inventory.ts`, `src/core/openingStock.ts`, `src/core/valuation.ts` (one optional field on `ValState`), `src/core/stockSummary.ts` (`stockFlow` variant cells in CF-9a; the `'field'` group mode in CF-9b), `src/core/splitCompany.ts` (the variants carry, and **the pending-order carry mapped by `line`**, CF-9d), new `src/core/stockByField.ts` (incl. `countDeltas`), **new `src/core/stockAgeing.ts` (the lot walk of `ui/inventory/ageing.tsx` moved unchanged; the component's edit is that one call)**, `scripts/assert/customfields-stock.assert.ts`; **`InventoryLine.countPart` is the one `types.ts` addition of this lane (surgical, with lane M's)**; **one writer at a time on each**: CF-9a before CF-5 uses `stockDeltaByItem` and before CF-9b |
| S | `backend/src/{conflict,sync,rest}.js` (**`conflict.js` also takes the two company-row lists, `CAP_KEPT` and `OMIT_KEPT`, and `needsUdfCurrent` grows a case**), `backend/src/histories.js` (if `loadHistories` is split), `backend/src/udfRules.js` (CF-6b), `src/sync/sync.ts` (the `caps` field and the stored server caps), `scripts/assert/syncconflict.assert.ts`, the deploy-time smoke script |
| X | `src/core/importMap.ts`, `src/ui/admin/ImportData.tsx`, `src/ui/admin/MappedImport.tsx`, `src/core/tallyImport.ts`, `src/core/tallyExport.ts`, `src/ui/admin/ApiDocs.tsx` |

| Slice | What | Lane | Effort [J, builder-days, tests and screenshots included] | Exit gate |
|---|---|---|---|---|
| **CF-0** (widened) | the model: `types.ts` additions (1.3, and **`Company.entryPolicy`, `EntryEdit`, `req: 'policy'`, `tracksStock`, `OpeningStockAlloc.variants`, `InventoryLine.countPart`**), `core/customFields.ts` (normalise, `placesOf`, `needsServerCap`, ids, coerce, read, `checkUdf` **with the `policy` and `dates` arguments**, `mergeUdf`, `udfView`, `usageOf`, caps, **`trackingFieldsOf`**), **`core/entryPolicy.ts`** (pure: `DEFAULT_ENTRY_POLICY`, sanitise, serialise, `modeFor`, `bindsAlteration`, `effectiveEntryPolicy`, `planEntryPolicyWrite`), **`core/fieldDimension.ts`** (the `Dimension`, `RowKind`, `partyOf`, `dimensionFor` for the choice kind; text, period and band join in CF-4c), the legacy reader, **the store plumbing (`saveDoc` and `saveVoucher` options, `UdfRefusal`, the `checkDelete` case) and the backup schema-3 step**, suites `customfields`, `customfields-rules`, **`entrypolicy`, `customfields-dimension`** and the `backupmigrate` extension | M | **7 to 8** | tsc 0, suites green with mutations killed, **no screen changes by a pixel** |
| **CF-1** | editor rewrite (object tabs, options editor, scope, required **with the per-field choice of AD.1.5 and the Tracks stock block of 7.2**, minimum path and More options, preview, hide, tombstone delete, v1 conversion, starter sets, "Custom Fields" menu row and `+`, **the dirty probe**), masters rewrite (tail, scope, `mergeUdf`, Alt+C overlay rule, **the master half of the entry policy**), **`forms.tsx` flags**; types stage A (text, number, amount, percent, date, yes/no, choice). The Workspace link moved to CF-14 | E | 7 to 8 | `cf-editor`, `cf-master-walk`, `cf-mouse-only` (editor part); `master-resave`, `gst-identity`, `sidenav`, `quickcreate`, `navstack` updated; identity suite for masters |
| **CF-6a** | server: `caps`, the `current` read, `UDF_CARRIERS`, `carryUdf` with the **strong-identity line pairing and the `countPart` carry** (9.3), the definition-delete rule, REST keep, pull and push `caps` and `udfKept`, **the capability-keyed company-row rules `CAP_KEPT` (`entryPolicy`) and `OMIT_KEPT` (`ui`), and `needsUdfCurrent` extended to a company-row `put` (AD.1.2). This REWORKS the code already built from the first text (`entryPolicy` inside `UNVERSIONED_PROTECTED`, keyed on "states no base"); it must land before the deploy (CF-14 writes the client half)**, `syncconflict` extension; the deploy-time smoke (**with two company-row pushes, one that states `udf2` and one that states nothing**); **deploy** | S | **5 to 6** | `customfields-sync` (incl. the I/O-level cases); the smoke passes on the VM; deployed (the owner's secrets) together with Phase 27's backend rule |
| **CF-2** | voucher header: strip, funnel and `voucher-resave`, dirty and signature, `resetLines`, duplicate and recall rules, cancel shell, print block (the form row only when a field prints), Bulk Entry and CSV columns, **the entry policy on alterations (W per voucher, the blank start, the refusal wording: AD.1.5)**. **Not released to a customer before CF-4a and CF-4b-lite (9.6)** | V (+X for the importers) | **10 to 12** | `cf-voucher-walk`, `cf-inherit`, `cf-print`, `cf-old-client`, `cf-geometry`, `cf-mouse-only`, `cf-policy` (voucher part); **`customfields-identity` green** |
| **CF-3** | item lines: inline columns and the sub-row by measured width, the Tab contract, the `cf:` focus, walk, snapshot, stock-journal grids, print sub-line. Same release gate as CF-2. (What is specific to a tracking field is CF-9c) | V | 6 to 7 | `cf-line-column`; the statutory-leak scan |
| **CF-4a** (widened) | **`core/fieldSummary.ts` over every voucher family (AD.2.5), `core/fieldBills.ts` (Bills by field with its Pending / Unapplied / Not bill-wise split and the group-balance tie, and Balances by field over the union of the chosen groups, AD.2.7), the party-leg and fallback-source logic (**`partyOf`**, AD.2.1), the shared shell `ui/reports/fieldShell.tsx`, the five screens' whole new-screen checklist registered ONCE** (`Screen` union, router and crumb, `SCREEN_AREA`, recents, five Reports Center rows in an "Analysis by field" group, Go To late commands; CF-5 and CF-9 add only their routes after it), the additive `line` property of `pendingOrders` rows | R + M | **10 to 12** | `cf-summary-zone`, `cf-summary-families`, `cf-bills-by-field`; `customfields-report`, `customfields-bills`; reconciliation at 2k and 50k for every family; perf reference rows |
| **CF-4b-lite** (new) | a company-aware filter field list for the four existing filterable reports, and chip labels that never print a raw id (the contract is below this table) | R | **2 to 3** | `cf-filter-lite`, `customfields-filterlite`; `reportfilter`, `savedviews-filter`, `reportfilter-rows` unchanged and green; with no definitions `filterFieldsFor` returns the same array references |
| **CF-4b** (narrowed) | catalogue integration through the grid: `customColumns`, the Columns panel, header filter and **sort (`sortKey`, CF28)**, **the `describeCondition` label resolver**, saved-view stale rules; **it absorbs CF-4b-lite** (below). The Stock Summary group mode moved to CF-9b | R (after B-grid S1 and S2) | 4 to 5 | `cf-report-picker`; `reportfilter`, `savedviews-filter` extended |
| **CF-4c** (new) | banded dimensions: the text, period and band kinds of `fieldDimension.ts`, the F12 controls on the shell (period unit, band edges) | R + M | **3 to 4** | `cf-dimension-bands`; `customfields-dimension` extended |
| **CF-5** (widened) | `fieldAttribution.ts` R1 to R10, `fieldPl.ts` over five sources, the screen, Layout A; the item-source and tracking-variant stock attribution uses CF-9a's `stockDeltaByItem` and `byVariant`, so that part ships after CF-9a | R + M | **8 to 9** | `cf-pl-by-field`, `cf-pl-sources`; the tie to `plFigures` for every source |
| **CF-7** | types stage B (long text, email, phone, URL, multi-choice), Fill blanks **with its own batch undo**, usage and field statistics, preset polish | E + M | 6 to 7 | `cf-fill-blanks`; `customfields-usage` |
| **CF-8** | employee and lead places; the lookup type, **with `checkDelete` on the five master tables and a Data Check line** | E | 4 to 5 | the same masters recipes on both forms |
| **CF-14** (new) | **the entry policy end to end (AD.1):** `state/entryPolicyWriter.ts` (the one writer), `state/entryPolicyMirror.ts`, `useEntryPolicy`, the Settings > Workspace panel `EntryRulesPanel.tsx` (the new `workspacePanels.ts` entry, whose `owner` literal gains `'C'`, carrying the "Custom fields..." link), the carry line in `core/companyAlter.ts` and `master-resave` classification, `backend/src/conflict.js` `CAP_KEPT.companies` gains `entryPolicy` and `OMIT_KEPT.companies` gains `ui` (the server half, built and deployed with CF-6a; the client half does not depend on it), the panel's dirty probe and consequence count | E (+M pure) | **4 to 5** | `cf-policy`; `entrypolicy` (mutations killed); `syncconflict`, `master-resave`, `navstack` extended; **identity: a company that never opens the panel stores no `entryPolicy` and behaves exactly as before** |
| **CF-9a** (new) | **stock by field, the engine (AD.3.3):** `VariantOf`, `byVariant` in `stockPositions`, **the count group (`countPart`, the merge in `stockMovements`, `countDeltas`) so no item-level figure depends on a resolver, and the lot walk of `ui/inventory/ageing.tsx` moved unchanged into `core/stockAgeing.ts`**, `openingVariantMap` and `OpeningStockAlloc.variants` **with both trust tests**, `stockItemRows` / `resolvedMovements` variant arguments, **`pendingOrders` per-line allocation and its fourth argument at every call site**, `stockFlow` variant cells (**quantities, and Opening and Closing value only**), `FlowOpts.bases` and `cellOf`, `stockDeltaByItem` (**opening read as `stockDelta` reads it**), `core/stockByField.ts` (`variantResolver`, `countDeltas`, `allocateValue` **with the negative-variant rule**, `stockByVariant`, `stockByItemAttribute`) | I (+M) | **6 to 7** | `customfields-stock` (identity, **the five Physical Stock vectors through five readers**, conservation and the invariant, allocation under all eight methods, every voucher type, opening, pending orders); `inventory`, `stocksummary` unchanged at the default; **no screen changes by a pixel**; perf reference `cf.stock` |
| **CF-9b** (new) | **the screen:** `ui/reports/fieldStock.tsx` (both readings, the combination view, the size-run grid, the godown level, F12), **Stock Summary's `'field'` group mode** (`ssGroupByOptions`, `viaField`), Stock Query's variant lines | R + I | **4 to 5** | `cf-stock-by-colour`; the screen ties to the Stock Summary at 2k and 50k; `stocksummary` deep-equal at the default |
| **CF-9c** (new) | **the variant on the voucher (AD.3.6):** the per-item cell, the walk, stock in the picker rows, the per-(item, variant) negative-stock warning, the Stock Journal destination default, **the Physical Stock screen (rows pre-filled per held variant, the unlisted-counts-0 warning, `countPart` on save, no mixing of part and whole rows)**, the manufacturing warning, **the Bulk Entry column and its severity, Copy locked on, Hide and Delete refused with Stop tracking (the editor, lane E), AD.3.4b**. **After CF-3** (one writer of `VoucherScreen.tsx`) | V (+E, +X for the importers' previews) | **7 to 8** | `cf-variant-entry`; **identity with tracking off (the voucher is exactly today's, byte-identical warning text)**; `customfields-stock` entry cases |
| **CF-9d** (new) | **migration tools (AD.3.7):** the Opening by variant panel (dirty probe, `Box` with `onClose`), the "Split Not set" Stock Journal draft, `core/splitCompany.ts` carries variants (**rows that pass both trust tests**) **and maps a carried pending order line by `line`**, the Data Check rows (**a negative Not set, still holding Not set, counts made by variant**, AD.3.4b) | I + E + V | **3 to 4** | `split` round trip; `cf-stock-by-colour` steps; `navstack` probe |
| CF-5b | Layout B (options as columns) | R | 3 | optional |
| CF-6b | API validation: server reads the current definitions **and the company's entry policy**, `udfRules.js`, a shared test-vector file proving it equals the client's `coerce` | S | 3 to 4 | `customfields-sync` vectors |
| CF-9e | per-variant costing: an opt-in per tracking field that values each variant by its own state (the item's value becomes the sum of its variants'), so it **does** move the item-level closing value; never the default | I | 4 to 5 | **gated on question 18**; a separate proof that the opt-in is the only thing that moves a figure |
| CF-9f | variant-aware manufacture (a bill of materials by variant; the finished good in a chosen variant) | V + I | 2 to 3 | gated (AD.6) |
| CF-10 | Tally UDF export and import | X | 4 to 5 | **gated on three real exports** (question 6) |
| CF-11 | field-level `access` | E + R | 3 | gated on question 4 |
| CF-12 | employee and lead fields in the payroll and CRM reports (the Dimension over those rows) | R | 3 to 4 | gated on the owner asking (question 21) |

**Total for CF-0 to CF-8 plus CF-4b-lite, CF-4c, CF-14 and CF-9: about 96 to 115 builder-days; the optional and gated slices (CF-5b, CF-6b, CF-9e, CF-9f, CF-10, CF-11, CF-12) add about 22 to 27 [J].** The earlier total, 55 to 66, covered CF-0 to CF-8 before the addendum; the addendum adds about 41 to 49 (the third review added 3: CF-9a +1, CF-9c +1, CF-6a +1): CF-9 is 20 to 24 of it, CF-14 4 to 5, CF-4a +5 to +6, CF-5 +3, CF-0 +2, CF-4b-lite 2 to 3, CF-4c 3 to 4, CF-2 +1 (the wave-a2 plan estimated the grid at 24 to 29 for comparison). The first draft quoted 55 to 70 against a table whose slices summed to 50 to 61. No calendar estimate is given: it depends on how many lanes run at once. **The owner's three scenarios, as examples, are reachable much earlier than the total:** the owner beta gate G1 (section 17) needs CF-0, CF-14, CF-1, CF-6a, CF-2, CF-4a and CF-4b-lite, about 45 to 54 days of the above.

#### What CF-4b-lite adds, and what the grid wave overwrites (CF27)

CF-4b-lite exists so that "filter the Sales Register by Zone" does not wait for the report grid. It is deliberately **additive and removable**:

| Added (new files) | `src/core/customFilterFields.ts` (`companyFilterFields(reportKey, defs, ctx)`: the `FieldMeta` list and the getters per row type), `src/core/filterLabels.ts` (**as built: `labelResolverFor(defs)`, the memoised `LabelResolver` that turns a custom field id, and a choice's option id, back into words, tombstones included, plus `roleLabel`, `optionLabelOf` and the `FilterLabels` shape `{ extra, resolve }`; there is no `staleMetas`**), `src/ui/cf/useCustomFilterFields.ts` (the hooks the reports call: `useCustomFilterFields`, `useFilterResolver`, `useFilterLabels`) |
|---|---|
| **Edited (as built; the first draft of this table said four of these were "not touched", which was wrong)** | `core/reportFilterFields.ts`: `filterFieldsFor(reportKey, extra?)` gains an optional second parameter and returns the static list **by the same reference** when `extra` is empty (the four exported lists, every id, kind and position are unchanged; custom ids are appended after the last static id, in definition order, `cf.` then `cfp.` then `cfl.`); `ui/reports.tsx`: the `dayFields` and `lvFields` blocks gain one spread; `ui/reports/financial.tsx`: the `fields` block of `RegisterReport` gains one spread; `ui/reports/outstanding.tsx`: the `billFields` block gains one spread (the bills family takes `cfp.` only). **Also edited, the label resolver's plumbing:** `core/reportFilter.ts` (`describeCondition`, `describeNode` and `describeFilter` gain an optional last `resolve` parameter; the new exports are the type `LabelResolver` and the two words-makers `staleFieldText` and `staleOptionText`; `sanitiseFilter` and every existing signature position are unchanged, and with no resolver every sentence reads as before); `core/savedViews.ts` (`viewPhrases` gains a 4th parameter `labels?: FilterLabels`, so a saved view that filters by a custom field reads "Zone is North" in the views list); `ui/rowFilter.ts` (`useRowFilter` re-filters when the SET of fields changes (`setKey`: a field deleted or switched out of reports under a waiting report), re-describes when a field or option is renamed (`metaKey`), passes a `resolve` to its descriptions and publishes it on the slot); `ui/header/FilterPanel.tsx` (a stale condition's `<option>` prints `staleFieldText` / `staleOptionText`, never an id); `ui/reportActions.ts` (`ReportActions.filter` gains an optional `resolve`); `ui/reportViews/useReportViews.tsx` (the three views dialogs call `useFilterLabels(def.report)` and pass it to `viewPhrases`). `ui/header/FilterBar.tsx` is not touched |
| **Known gap, said plainly** | none now: the saved-views list's one-line description of a view that filters by a custom field reads by label (the `labels` parameter of `viewPhrases`); a company with no custom field passes nothing and every phrase reads as it always did |
| **The grid wave overwrites** | the body of `filterFieldsFor` and the constant `FIELDS_BY_FAMILY` (they become the catalogue's filterable columns); the four `fieldDefs(...)` blocks named above (`dayFields`, `lvFields`, `fields`, `billFields`) and their `useRowFilter` calls; the parameter list of `useRowFilter` (it takes catalogue fields). With them go lite's `extra` parameter, its four spreads, `useCustomFilterFields` and `customFilterFields.ts`, **replaced by `customColumns` of CF-4b**. `filterLabels.ts` (`labelResolverFor`, `FilterLabels`) survives: it is the resolver CF-4b's `describeCondition` parameter already takes |
| **The grid wave must preserve** | the ids `cf.`, `cfp.`, `cfl.` and their kinds (saved views hold them, CF9), the rule "static ids first, custom ones appended", and the stale-id behaviour (kept, matches nothing, chip says "not in this report"). **And the resolver plumbing it must NOT overwrite when it rewrites `ui/rowFilter.ts`:** the `setKey` / `metaKey` re-filter and re-describe, `useFilterResolver()` and the `resolve` member of the slot's `filter`; in `core/reportFilter.ts` the `resolve` parameter of the three `describe*` functions with `LabelResolver`, `staleFieldText` and `staleOptionText`; in `core/savedViews.ts` the `labels` parameter of `viewPhrases`; in `ui/header/FilterPanel.tsx` the stale `<option>` words; and `useFilterLabels` in the views dialogs. A grid wave that took the older text of this table literally would overwrite `rowFilter.ts` and lose the stale-field re-filter and the chip wording |

**Review lenses at each gate** (the wave-a2 method: adversarial reviewers, each with a lens, then a skeptic verifier per finding): **L1 accounting integrity** (a field moving a figure; the P&L tie; a summed
relation column), **L2 keyboard and the frozen walk** (every Enter path with fields on and off; Accept-on-Enter; `Esc` stacking over a choice popup), **L3 persistence and older clients** (the writers of 9.1; the
server rule; a stale definition list; a backup restored in an older app), **L4 print and statutory leakage**, **L5 performance**, **L6 visual** (density, schemes, 150 % text, the strip and column geometry), **L7 rights**
(the writers of 3.3), **L8 seams** (any file touched outside its lane; a registered chord quoted elsewhere).

---

## 17. Build order of everything pending, and what can run in parallel

**Pending work and its state today [C]:** (a) parity wave 3, **running** in this tree (five builders; 158 paths uncommitted); (b) the report grid, i.e. wave-a2's B-grid and what remains of its scaffold A2-0 (the
Dashboard half, B-dash, shipped as Dashboard v2); (c) custom fields (this document); (d) the mouse Back control and the dialog fixes (the Back spec).

```
 wave 3 gates + COMMIT  ───────────────────────────────────────────────  blocks every lane (shared files, section A item 1)
   │
   ├─ Back B-1, B-3, B-2 (top bar Back, planBack, trail, rail Esc, dialog close)   8 to 11 d   small, first
   ├─ CF-0               (model, entry policy core, Dimension; new pure files + types.ts)  7 to 8 d
   ├─ A2-0 residual      (table goldens, grid seams, the Columns chord, perf and css splits)   about 3 d
   └─ CF-6a              (backend rule, caps, REST keep, capability-keyed company-row rules; reworks the first-text code)   5 to 6 d    deploy gated on the owner's VM secrets
        │
        ├─ CF-14 (entry policy: writer, mirror, Workspace panel)  ║  CF-9a (variant engine: inventory.ts and friends, no screens)  ║  CF-4b-lite (filter fields for four reports)
        │
        ├─ B-grid S0, S1 (catalogue, ListTable, listQuery)  ║  CF-1 (editor, masters, per-field choice)  ║  (Back B-4 inside B-grid S1)
        │
        ├─ B-grid S2..S4 (layout, panel, quick filter, selection)  ║  CF-2 (voucher strip, funnel, print, policy on alterations)  ║  CF-4a (Summary, Bills, Balances; the five-screen checklist, once)
        │        (CF-2 must wait for the backend deploy before it reaches a SYNCED company)
        │          ==> G1, owner beta: CF-0, CF-14, CF-1, CF-6a, CF-2, CF-4a, CF-4b-lite
        │
        ├─ CF-3 (lines)  ║  CF-5 (fieldPl, needs CF-9a for item and variant stock)  ║  CF-9b (Stock by field screen, Stock Summary group mode)  ║  CF-4c (banded dimensions)
        │        then CF-9c (the variant on the voucher; after CF-3: one writer of VoucherScreen.tsx)  and  CF-9d (opening by variant, split, Data Check)
        │
        ├─ CF-4b (columns and sort through the grid; needs B-grid S1 and S2; absorbs CF-4b-lite)
        │          ==> G2, customer release: G1 + CF-3, CF-4b, CF-5 (and CF-9 for a company that tracks stock)
        │
        └─ CF-7, CF-8, CF-6b, then CF-5b, CF-9e, CF-9f, CF-10, CF-11, CF-12 (all gated)
```

**Why this order**
1. **Nothing starts before wave 3 commits**: `VoucherScreen.tsx`, `widgets.tsx`, `masters.tsx`, `types.ts`, `store.ts`, `reports.tsx`, `router.tsx`, `sync.ts`, `backend/src/sync.js` and the registry are all modified there, and the Edit tool refuses a stale file.
2. **Back (B-1, B-3, B-2) goes first because it is small and nearly disjoint**: it touches `widgets.tsx`, `navigate.ts`, `navPlan.ts`, `navGuard`, `keys.ts`, `railtop.css`, plus one-line `onClose` edits at the hand-built overlays in `VoucherScreen.tsx`, which it makes **before CF-2 starts** (so
   the voucher file has one writer at a time).
3. **CF-0 first in the custom-field line** because every other slice imports it; it adds files and one writer's edit to `types.ts`, which nothing else in phase 1 touches (A2-0 does not).
4. **CF-6a runs alongside** (disjoint: `backend/` and one block of `sync.ts`) and is the long pole because of the deploy: do it with Phase 27's backend rule in one window.
5. **B-grid and CF-1 run in parallel** (disjoint: reports, `listQuery`, saved views against editor, masters, `udf.ts`); **B-grid and CF-2 run in parallel** (reports against `VoucherScreen`, print, Bulk Entry).
6. **CF-4b waits for B-grid S1 and S2 on purpose (columns and sort need the grid's table), and CF-4b-lite does not wait (CF27, revised by the addendum).** The first draft let the owner's "filter the Sales Register by Zone" wait for the grid, because B-grid S1 rewrites `filterFieldsFor` and the four field lists and a stopgap would be overwritten. The owner has asked for every analogous scenario, so the filter is built now as **additive and removable** (section 16: new files, one small edit in each of five places, nothing in `reportFilter.ts`, `savedViews.ts` or `rowFilter.ts`, and a written list of what the grid wave overwrites). The cost, stated: CF-4b-lite is about 2 to 3 days of work that the grid deletes later; the benefit is that the headline filter, and Summary by field and Bills by field's drills, work at G1.
7. **One writer for the new-screen checklist** (`Screen` union, router, access, recents, catalogue, Go To, palette): CF-4a, once, **for all five screens** (`fieldSummary`, `fieldPl`, `fieldBills`, `fieldBalances`, `fieldStock`); CF-5 and CF-9b add only their routes to the same files after CF-4a, never in parallel.
7a. **CF-9a (the variant engine) is early on purpose:** it is pure, has no screen, and is the only slice that edits `inventory.ts`; it unblocks CF-5's item-source stock attribution, CF-9b and CF-9c. It must land before anything else edits `inventory.ts`, `openingStock.ts` or `stockSummary.ts`'s `stockFlow`. **It changes what every stock figure reads in exactly one case, a Physical Stock voucher with `countPart` lines, and only after CF-9c can such a voucher exist** (the voucher screen is the only writer of the flag), so CF-9a can land and be gated with no company affected; **CF-9c is not released to a customer before the build carrying CF-9a is the one running everywhere that the company's stock is read** (an older build reads a count group last line wins, AD.3.3). It also moves `ui/inventory/ageing.tsx`'s lot walk, unchanged, into `core/stockAgeing.ts` (one call in the component).
7b. **CF-14 (the entry policy) runs beside CF-1, not inside it:** disjoint files (the Workspace registry, two state files, one company-alter line against the editor and masters; its server half, the two company-row lists, is CF-6a's), and CF-2 and CF-1's master half read the policy through `useEntryPolicy`, so CF-14's reading hook lands first or with them.
8. **The Columns chord:** the wave-a2 plan's `Alt+M` is void (A, item 2). The grid owner re-picks in A2-0; this spec needs nothing from it.
9. **`Box.onClose` ships in two steps (Back B-2a, B-2b; back spec 12.1).** The first draft made it required at once (about 24 compile errors), which collides with the new dialogs that CF-1 (Starter sets, Fill blanks) and B-grid S1 (Columns panel) add. B-2a makes it optional in the type, fills every existing site and adds a source scan that **fails any `<Box` without `onClose`** whatever the type says, so a dialog written meanwhile is held to it from day one; B-2b flips the type to required once those slices have landed.
10. **Two release gates.** **G1, owner beta:** CF-0, CF-14, CF-1, CF-6a (deployed, smoke passed), CF-2, CF-4a and CF-4b-lite: the owner can define Zone, enter it, make it mandatory with the rule of their choice for old vouchers, filter the four reports by it, and see Summary, Bills and Balances by it. **G2, customer release:** G1 plus CF-3 (line fields), CF-4b (the same field as a column and a sort), CF-5 (profit by field) and, for a company that keeps stock by a variant, CF-9. Voucher and line fields (CF-2, CF-3) are never released to a customer before CF-4a and CF-4b-lite (the owner would see Zone entry without a filter or a summary), and not to a synced company before CF-6a is deployed and its smoke has passed (9.6).
11. **`store.ts` has three writers in this plan, in this order:** CF-0 (the check plumbing and `checkDelete`), then CF-4a (the `Screen` union, once for all five new screens); lane E no longer edits it. `forms.tsx` is lane E (CF-1), and the Settings > Workspace panel and link are lane E (CF-14). `VoucherScreen.tsx` has three writers after Back's overlay edits: CF-2, then CF-3, then CF-9c.

**Disjoint file sets that can run at the same time:** Back B-1 to B-3 (`widgets.tsx`, nav files, `keys.ts`) with CF-0 (new files, `types.ts`) with A2-0 (`savedViews.ts`, `reportActions.ts`, `FilterBar.tsx`, `PageActions.tsx`, registry) with CF-6a (`backend/`, `sync.ts`);
then CF-14 with CF-9a with CF-4b-lite (disjoint: Workspace and state files, the inventory engine, the filter lists); then B-grid with CF-1; then B-grid with CF-2 with CF-4a. **Never in parallel:** two slices that both edit `VoucherScreen.tsx` (CF-2, CF-3, CF-9c, Back B-2's overlay edits), `inventory.ts` or `stockSummary.ts` (CF-9a then CF-9b), `types.ts`, the registry, or the new-screen files.

---

## 18. Questions that genuinely need the owner (each with the default taken so work can proceed)

| # | Question | Default |
|---|---|---|
| 1 | **ANSWERED 2026-10-22.** **Colour:** is it an attribute of the item (each shirt has a colour), a choice on each sale line, or does the dealer need **stock by colour**? These are three different features | **Answer: "this was example and other such scenarios shall be fulfilled".** All three are built: an item attribute and a per-sale choice (one field, both places), and a line field marked **Tracks stock** keeps stock by Colour, Size or any such variant (CF24, AD.3). Value by variant is the item's value shared by quantity (question 18). The first draft's "do not promise stock by colour" is withdrawn |
| 2 | **ANSWERED 2026-10-22.** Should a mandatory field bind an **old** voucher when someone alters it? | **Answer: "there shall be an option at admin level to decide this, as per their use case".** A company setting in Settings > Workspace (vouchers: new entries only / from a date / always; masters: new only / always) plus a per-field override; **the default stays "new entries only"** (CF19, AD.1) |
| 3 | **ANSWERED 2026-10-22.** Do you need **profit** by Zone or only **sales** by Zone? | **Answer: "this was example and other such scenarios shall be fulfilled".** Summary by field and Income and expense by field are **both** built, over every voucher family and every place a field lives (party, item, header, line, posting ledger), with Bills, Balances and Stock by field beside them (CF20 to CF23, AD.2). Balance Sheet by field is still not offered (6.7) |
| 4 | May some operators be **kept from seeing** a field (a salesman not seeing a cost field)? | not in v1; reserved (`access`, CF-11) |
| 5 | Should a field **print on the invoice**? | per-field switch, off |
| 6 | Do Tally switchers need their **TDL fields to come across on import**? | later, **gated on three real TallyPrime exports** from partners who run UDFs |
| 7 | Where should a mandatory header field sit, and is "Enter walks into it" acceptable (it changes Enter on Narration for a company that makes a field mandatory)? | after Narration, in the walk; optional fields never change the walk |
| 8 | **Backend deploy:** when can the VM be deployed (it needs the SSH and admin passwords you paste at session start)? Voucher and line fields cannot reach a synced company before it | one deploy window carrying CF-6a and Phase 27's pending backend rule |
| 9 | Older apps: **merge and record** their writes, or **refuse** them with "update the app"? | merge and record (refusing strands an Electron install nobody updated) |
| 10 | Are the **starter field lists** (7.5: garment, distributor, pharma, contractor, jewellery) the right ones? | as proposed; every field editable |
| 11 | Is the **Fill blanks** tool (tagging old vouchers in bulk, one undo step) acceptable, given the plan's rule against bulk editing vouchers? | yes: it writes one custom field only, never an amount |
| 12 | A salesperson on a voucher (the grid research's question D6): a built-in field or a custom one? | custom (a Distributor starter field) |
| 13 | Option **colours** on a choice? | no in v1 (needs a validated palette) |
| 14 | A **chord** for the Columns panel, now that `Alt+M` is Share? | no letter chord: the page header's "..." menu and a Columns button; `Alt+F10` is the only unused candidate (verify in Electron) |
| 15 | Should the product offer a non-blocking **Recommended** tier (highlighted when empty, never blocks) as Pipedrive does with its "Important" quality rule, a tier below Required [W https://support.pipedrive.com/en/article/custom-fields]? | not in v1: `walk: true` already asks for a field in the Enter walk without blocking |
| 16 | A switcher imports last year's invoices by CSV with a field mandatory: count the ones lacking it and import them, or refuse them? | import and count; an "Enforce mandatory fields" tick (default off) refuses them (3.3) |
| 17 | A mandatory field is not asked on a **Manufacturing Journal** or a **Salary Disbursement** (system writers) in v1: acceptable? | yes; the editor warns when a required field is scoped to Stock Journal or Payment (3.3) |
| 18 | **Cost per variant (new, 2026-10-22):** when Colour or Size tracks stock, is **one cost per item** acceptable (each variant's value is the item's value shared by quantity), or must each variant carry its own cost (a jeweller's purities, a fabric that costs more in one colour)? | one cost per item (CF24): it can never move a figure outside Stock by field. Per-variant costing is the gated slice CF-9e, opt-in per field |
| 19 | How many **stock-tracking fields** may one item have (Colour and Size and what else)? | up to three; the position is the combination (AD.3.2) |
| 20 | Should analysis read an old voucher's blank through the **party's current value** ("the invoice's Zone; where blank, the party's")? | yes, as an option on every analysis screen, labelled with the count of rows that used it; Fill blanks remains the way to freeze it (AD.2.1) |
| 21 | **Employee and lead fields** in the payroll and CRM reports? | later (CF-12); defined and filterable on their own lists (CF-8) meanwhile |
| 22 | Should an administrator get a **warn-only** tier for old records ("ask, but let me skip")? | no (question 15 stands): the three voucher modes are never, from a date, always (AD.1.1) |
| 23 | Does an `always` policy need a **grace period** after it is switched on (a day when old vouchers may still be edited unprompted)? | no: the consequence line ("1,204 vouchers would need completing when edited") and Fill blanks are the way in; a date mode (`since`) is the grace tool |
| 24 | **Physical Stock by variant (added by the third review):** is a count of Red a count of the **whole shirt** at that godown (variants left out count 0; the screen pre-fills a row per held variant so only a deliberate change moves a variant), or a count of **Red only** (every other variant untouched)? | the whole item (AD.3.3). It is the only reading that leaves every other report (the Balance Sheet, the Profit and Loss, the Stock Summary, Stock Ageing) reading the same item quantity as Stock by field, because those callers do not know variants; "Red only" would need every one of them to be given the variant balance. The pre-filled rows make the whole-item reading feel like the Red-only one |

---

## 19. What this specification did not verify

- No live Zoho Books, TallyPrime or Pipedrive account; the vendor behaviour in sections 4 to 6 is documents (three pages re-fetched today, the rest read in the research docs, whose own tags I kept).
- **Nothing was run.** No suite, build, screenshot or perf run: the tree belongs to wave 3. Every line number in a file wave 3 edits will drift; the symbols are stable.
- **The shape of TallyPrime's UDF XML** and whether a Tally customer's UDFs map onto these types: [unverified]; CF-10 is gated on real files.
- Whether `Alt+F10` is safe in Electron and on each platform; whether `window.close()` or any shell key interacts: [unverified].
- That the Income and expense by field rows plus Change in stock always equal the Profit and Loss nett on every company shape: this is the **claim the suite must prove**; a ledger of Income or Expenses nature outside the six reserved groups is a memo row, not part of nett (6.4, R2; the tie is asserted on a company that has one). I did not scan the sample company for one.
- The per-line identity rules of 9.3 (the weak and the strong identity, the pairing order) are conservative by construction (a duplicated or absent identity gets nothing, recorded), not proven rare.
- Layout numbers (the 1024 and 1280 px grids, the item table's 650 px, how many columns fit) come from the CSS and the review's own measurements and were **not re-measured** (the dev server belongs to the wave-3 builders); the width rule of 2.2 is a formula for that reason.
- The cost of the server's `current` read is an estimate from the query's shape and the existing index [J]; the deploy-time smoke checks correctness, not timing.
- Older-app behaviour beyond what its source says: an older client's reading of a multi-choice value is inferred from `ui/udf.ts` (`String(v)` then trimmed text), not run.
- CF effort numbers are [J] ideal builder-days of one engineer who knows the repo, tests and screenshots included.
- **The addendum (2026-10-22) is a design read of code, not a run.** Specific claims it rests on and that its suites must prove: (a) **pooled costing leaves every existing position field unchanged, with or without a resolver**: I read `stockPositions` and `valuation.ts` end to end and the `byVariant` update sits after the `bookIn` and `bookOut` calls, and (third review) the one input that could depend on the resolver, a Physical Stock count's `cur`, is now resolved against `byGodown` only and the count group is merged in `stockMovements` from `countPart`; nothing was executed, and the claim that **four places resolve a count** (`stockPositions`, `resolvedMovements`, `stockItemRows`, `ui/inventory/ageing.tsx`) and that `stockMovements` is the only generator they share is read from the code, not run; (b) **the allocation ties to the item's value to the paisa under all eight costing methods**, including a negative variant and zero net quantity: argued from `value = qty x rate`, `lotsValue` and the `Avg` path, not run; (c) **`partyId` is set by the voucher screen only for the invoice bases in item-invoice mode** [C `VoucherScreen.save`: `vPartyId` is assigned in invoice mode], and the Tally importer sets it on every type that carries a party name [C `tallyImport.ts`]; **`partyOf` therefore honours `partyId` only for `baseInfo(base).invoice` bases** and reads the party-side legs for a Receipt, Payment, Journal and Contra whether or not it is set (the first text let `partyId` win everywhere, which put an imported Payment wholly under the party's bucket and its hand-entered twin in two); the legs rest on the Sundry Debtors and Creditors subtrees; (d) **the party side per base** (debit for Sales, Debit Note and Payment, credit for Purchase, Credit Note and Receipt) is taken from `voucherHeadlineAmount`'s invoice logic and extended to Receipt and Payment by me: a Receipt with a debit on a creditor (a refund) falls into No party, not into a bucket, and an accounting-mode invoice's `invoiceValue` against its party legs was not checked; (e) `Company.entryPolicy` surviving Split Company Data and `core/companyAlter.ts` is **asserted by a classification test, not read**: I read only the opening-stock block of `splitCompany.ts` and did not open `companyAlter.ts`; (f) the Workspace panel's fit at 1024 px and 150 % text, and the Opening by variant panel's, were not measured; (g) `workspacePanels.ts`'s `owner` literal (`'A' | 'B'`) gaining `'C'` is read from the file, not compiled; (h) the claim that **`companies` already takes the server's `protect` read because of `ui`** is read from `conflict.js` and `sync.js`'s description in 9.3, not run; **(i) (third review) that every client since Phase 24 stamps a base while signed in, and that a current client's rows made signed out, before its first pull or in another namespace carry none, is read from `sync/baseline.ts`, not run; that a Phase 27 or 28 build states no capability is read from `sync/caps.ts` having arrived with this feature**; (j) that the Bills reports decide a side by sign is read from `outstanding.tsx` and `dash/compute/receivables.ts`, and the Bills by field columns tie to the group balance by construction, with the Dashboard tie proved only by the suite.
- **What the addendum does not decide:** the visual design of the entry-rules panel, the size-run grid and the stock hint in the picker (the first recipes will); the exact wording of the new refusal and tag texts (the strings above are proposals); the Stock Query's variant lines (optional in CF-9b).

---

## 20. Sources

**Repo (read, not modified):** `CLAUDE.md` (Phases 20 to 28 gotchas); `docs/research/{zoho-custom-fields,zoho-reporting-tags,tally-udf-and-competitors,custom-fields-code-audit,back-nav-audit,wave-a2-plan,pipedrive-list-ux}.md`;
`docs/design/zoho-ux-spec.md` (sections 5 and 6); `docs/HANDOFF.md` section 2; `src/types.ts`, `src/ui/udf.ts`, `src/ui/admin/CustomFields.tsx`, `src/ui/forms.tsx`, `src/ui/VoucherScreen.tsx` (the narration handler, the save literal, the four `narrationRow` sites),
`src/ui/widgets.tsx` (`Box`, `ListPrompt`, `ButtonPanel`, `TopBar`), `src/core/{reportFilter,reportFilterFields,savedViews,reportRows,accounting,plFigures,costCentreTree,printConfig,uiPrefs,navPlan,access,shortcutRegistry,money,dates}.ts`,
`src/ui/{navigate.ts,navGuard.tsx,AppShell.tsx,reports/costpl.tsx}`, `src/state/{dirty,navGuard,recents,store}.ts`, `src/keys.ts`, `src/data/db.ts`, `backend/src/{conflict,rest,restPolicy}.js`, `backend/src/sync.js` (diff stat only).
**Read for the 2026-10-22 addendum (worktree `.wt-cf`, snapshot `baa0c45`):** `src/core/{inventory,valuation,openingStock,stockSummary,billwise,plFigures,reportRows,accounting,policyWrite,appearance,uiPrefs,reportFilterFields,reportFilter,savedViews,splitCompany}.ts` (the parts named in AD), `src/state/{companyUi,policyMirror}.ts`, `src/ui/settings/{WorkspaceTab.tsx,workspacePanels.ts}`, `src/ui/rowFilter.ts`, `src/ui/dash/widgets/compute/common.ts` (`partySideOf`, `ageClock`), `src/ui/VoucherScreen.tsx` (the `positions` memo, `vPartyId`, the negative-stock warning, the voucher literal), `src/data/defaults.ts` (group slugs), `backend/src/conflict.js` (the protection lists), `docs/research/wave-a2-plan.md` (4.4 seams, 5.2 to 5.3), `docs/design/back-nav-spec.md` (4, 13).

**Re-read for the third review (2026-10-22, same worktree):** `src/core/{inventory,valuation,openingStock,stockSummary,billwise,plFigures,accounting,tallyImport,splitCompany}.ts` (the Physical Stock arithmetic in `stockPositions`, `resolvedMovements` and `stockItemRows`; `newValState`; `stockFlow`'s godown cells; `outstandingBills`; `moveFromTotals`; `voucherHeadlineAmount`; the importer's `partyId`; the pending-order carry), `src/types.ts` (`VCH_BASES`), `src/ui/inventory/ageing.tsx`, `src/ui/reports/outstanding.tsx`, `src/ui/dash/widgets/compute/{common,receivables}.ts`, `src/ui/VoucherScreen.tsx` (`isPhysical`, `vPartyId`, the pending-order picker), `src/ui/admin/BulkEntry.tsx`, `src/sync/{baseline,caps,sync}.ts`, `backend/src/{conflict,histories}.js` (the built CF-6a code). Nothing was run.

**Web, fetched this session [W] (paraphrased):**
- Zoho Books, adding a column to the item table of transactions: https://www.zoho.com/books/kb/items/add-new-column-in-item-table.html
- Pipedrive, custom fields (re-read in the second pass for its Important and Required quality rules): https://support.pipedrive.com/en/article/custom-fields
- Zoho Books, Profit and Loss by reporting tag: https://www.zoho.com/in/books/kb/reports/profit-and-loss-report-reporting-tag-wise.html

**Web, through the research docs only [R]:** the URL lists of `docs/research/zoho-custom-fields.md` section 5, `zoho-reporting-tags.md` section 10, `tally-udf-and-competitors.md` section 8 and `pipedrive-list-ux.md` section 15 (Zoho help, KB, API and community pages; Tally TDL reference and cost-centre help; BUSY, Vyapar, QuickBooks, Xero, Odoo, Pipedrive pages). Each claim attributed to one of them carries its [R] tag; none was re-fetched.

---

## Appendix. The owner's cases, worked

### Case 1: A clothes dealer: Colour at item level, with options, visible in the reports

**Definition** (written by the editor; the row is a normal `customFields` document):

```json
{ "id": "…uuid", "cid": "…", "objectKind": "stockItem", "also": ["line"], "v": 2,
  "fid": "f3k9x2ab", "key": "f3k9x2ab", "label": "Colour", "type": "select",
  "opts": [ { "id": "o1a2b3c", "label": "Black" }, { "id": "o4c5d6e", "label": "White" }, { "id": "o7e8f9g", "label": "Red" } ],
  "options": ["Black", "White", "Red"],
  "scope": { "line": { "bases": ["Sales", "Purchase", "CreditNote", "DebitNote"] } },
  "inherit": { "item": true }, "use": { "report": true, "print": true, "copy": true } }
```

1. *Custom Fields > Items > Alt+A*: name Colour, type Choice, paste the colours, tick Stock item and Item line, tick the four voucher types, Print on. The preview shows the item form field, the sale-line column and
   the printed sub-line; the screen says "This labels stock; it does not split it".
2. *Stock Item form*: Colour is a **tail** field (Tab or Down reaches it; Enter on the last built-in field still raises Accept). The dealer sets each item's usual colour.
3. *Sales voucher*: the item grid has a **Colour** column after Amount. Choosing an item starts the line from the item's colour ("from item"); Enter on Amount goes to the next row as always; **Tab** reaches Colour (a column at 1280 px, a sub-row under the item at 1024 px); picking Red writes
   `inventory[i].udf = { f3k9x2ab: 'o7e8f9g' }`. The printed invoice carries "Colour: Red" under the item name.
4. *Reports:* Stock Item Vouchers (line grain): a **Colour** column and a filter "Colour is Red" (the line's own value). Sales Register: "Colour (items)", the one colour, "Multiple" or blank. **Summary by field**, Item line, Colour: "Item value, before
   tax" by Black, White, Red, Not set, equal to the sum of the lines. **Stock Summary**: group by Colour (the item attribute).
5. *Edit the list:* rename Red to Crimson: every report, the picker and the print show Crimson at once; the saved view "Crimson lines" still matches. Hide White: old lines still read White and report; new lines are not offered it.
6. **Stock by colour, when the dealer needs it (revised by the addendum):** tick **Tracks stock** on Colour (the editor ticks the Item line place, locks the eight stock-moving voucher bases, sets Required to follow the company setting and shows the preflight). "How many Red shirts are left" is then **Stock by field, Colour**: Black, White, Red, Not set, with exact quantities and the item's value shared by quantity: Case 4. **Without the tick** the field labels the stock and does not split it, and the definition screen says so. The limit that remains is the cost: one cost per item, so a colour that costs more is an approximation (AD.3.1).

### Case 2: A distributor: Zone (North, South, East, West) on Sales invoices, filterable in the Sales Register

```json
{ "objectKind": "ledger", "also": ["voucher"], "v": 2, "fid": "fz8q1m4k", "key": "fz8q1m4k", "label": "Zone", "type": "select",
  "opts": [ {"id":"on7k2xq","label":"North"}, {"id":"os3m9ap","label":"South"}, {"id":"oe5t1vd","label":"East"}, {"id":"ow8h4zr","label":"West"} ],
  "options": ["North","South","East","West"],
  "scope": { "groups": ["<Sundry Debtors>"], "vch": { "bases": ["Sales", "CreditNote"] } },
  "req": "policy", "inherit": { "party": true }, "use": { "report": true, "print": false, "copy": true } }
```

1. Required applies **wherever the field is asked**: a new debtor needs a Zone, and a new Sales or Credit Note needs one. An old debtor or old invoice without a Zone is handled **by the company's entry rule (AD.1)**: by default editing it is not blocked; if the distributor wants every old invoice completed when it is next edited (or only invoices dated from 1 April 2026), the administrator sets that in Settings > Workspace > Custom-field entry rules, ideally after Fill blanks (step 6), and a Zone field that must always be checked regardless sets its own `req: 'always'`.
2. *Sales voucher:* choosing the party fills Zone with "North (from party)"; Enter on Narration **walks into Zone** (required means in the walk); Enter accepts the value and then Accept; one extra Enter per voucher.
   A party with no Zone leaves the cell blank, and Ctrl+A is refused with "Zone is required!" and the cursor on it; Enter on the blank cell is refused the same way (it never picks North for a habitual Enter). Duplicating (Alt+2) copies Zone.
3. *Sales Register, flat level* (`Alt+F12` or the filter bar): Columns panel > Your custom fields > Zone (sub-label "asked on Sales, Credit Note"; not in the standard layout); filter "Zone is North"; the total row reads "Total (filtered)";
   **Save view** "Sales, North, this quarter" (a company view if the administrator shares it).
4. *Day Book* (mixed types): the Zone column is blank on Purchases and Journals and on Sales that predate the field; the **Not set** row of the summary is those Sales.
5. *Summary by field*, Voucher, Sales Register, Zone, invoice value: North, South, East, West, Not set, **Total = the register's total**; Enter on North drills to the register with "Zone is North" visible; the Not set drill waits for `isEmpty`.
6. *Fill blanks* on the old Sales vouchers from the party's Zone (preview with the count per option; the result bar's Undo reverses it).
7. *Income and expense by field*, Zone: North, South, East, West income and expense (a Purchase of the same Zone appears if the field is scoped to Purchase), Not set, Not asked, the **Change in stock** line, and the check "Agrees with the Profit and Loss statement".
8. *Beyond Sales (addendum):* **Summary by field**, Receipts family: collections by Zone (the party's Zone, party legs, a No party row for the cash part); **Bills by field**: Receivables ageing by Zone with a Not set row and an on-account column; **Balances by field**: debtor balances by Zone including ledgers that are not bill-wise; **Income and expense by field**, party source: "Profit by the party's Zone (current)" beside step 7's invoice-level reading. Where the invoice holds no Zone the "invoice's value, else the party's" option reads the party's (AD.2.1).

### Case 3: A profitability tag across voucher types

`label: "Business line"`, `objectKind: 'voucher'`, `scope.vch.bases`: Sales, Purchase, Credit Note, Debit Note, Journal, Payment, Receipt; options Retail, Wholesale, Service; `req: 'policy'`. Created once, "tagged onto" seven voucher types.
*Income and expense by field*, Business line: Retail, Wholesale, Service, **Not set** (an in-scope voucher left blank on an old entry), **Not asked** (a Contra, a Stock Journal, a Payroll voucher: a type outside the scope), and the tie to the Profit and Loss. If a
line-level tag is wanted too (Business line per item), it is a **second** field on `line` with the same option labels: two fields, one list each, because one list shared by two fields is the "Tag master" this spec declined to build (6.1).

### Case 4 (addendum): A garment dealer who needs stock by Colour and by Size

```json
{ "objectKind": "stockItem", "also": ["line"], "v": 2, "fid": "f3k9x2ab", "label": "Colour", "type": "select", "tracksStock": true, "req": "policy",
  "opts": [ {"id":"o1a2b3c","label":"Black"}, {"id":"o4c5d6e","label":"White"}, {"id":"o7e8f9g","label":"Red"} ],
  "scope": { "stockGroups": ["<Shirts>"], "line": { "bases": ["Sales","Purchase","CreditNote","DebitNote","DeliveryNote","ReceiptNote","StockJournal","PhysicalStock"] } },
  "inherit": { "item": true } }
{ "objectKind": "line", "v": 2, "fid": "f8m2q7zd", "label": "Size", "type": "select", "tracksStock": true, "req": "policy",
  "opts": [ {"id":"oxs1","label":"S"}, {"id":"om2","label":"M"}, {"id":"ol3","label":"L"} ],
  "scope": { "stockGroups": ["<Shirts>"], "line": { "bases": ["Sales","Purchase","CreditNote","DebitNote","DeliveryNote","ReceiptNote","StockJournal","PhysicalStock"] } } }
```

1. *Editor:* ticking **Tracks stock** ticks the Item line place, locks the eight bases, sets Required and shows the preflight ("212 items in Shirts hold stock worth 38,40,000; their stock will read Not set until you split it"). Size is the second tracking field for the same items (the limit is three).
2. *Opening stock:* **Stock by field**, Colour, item "Cotton shirt", **Opening by variant**: Black 40, White 30, Red 30 (adds to the item's opening 100; the sum is checked). Stock booked since tracking was switched on, still Not set, is split by **"Split Not set"**, which builds a Stock Journal draft (Not set issue lines, variant receipt lines, value-neutral) that the person Accepts like any voucher.
3. *Purchase and Sales vouchers:* the item row's walk is name, quantity, rate, discount, amount, **Colour, Size** (required means in the walk); on a Sales line the Colour picker shows "in stock 12" per option; selling 15 of Black M warns "Cotton shirt (Black, M) will go NEGATIVE (3 short)!" and Continue still works; an item outside Shirts shows no Colour or Size cell.
4. *Physical Stock:* choosing "Cotton shirt" at Main pre-fills one row for every variant held there at its book quantity (Black M 12, Black L 8, White M 6 ...); the clerk changes Black M to 11 and Accepts. That is **a count of the whole shirt at Main** (the lines are saved with `countPart`, the item becomes the sum of the rows), so Black L and White M keep their pre-filled book quantities and only Black M moves (-1); a row deleted by hand would count 0, and the screen says so before Accept. The item quantity is the same in the Stock Summary, the Balance Sheet and Stock Ageing as in Stock by field (AD.3.3). *Stock Journal:* moving 5 Black M from Main to Shop starts the destination row at Black M ("from source line"); changing it to White M is a deliberate variant change.
5. *Stock by field:* Colour: Black, White, Red, Not set with Opening, Inwards, Outwards and Closing quantity (exact) and closing value (allocated), then Colour and Size as two levels; Enter on "Cotton shirt" opens the **size-run grid** (Colour down, Size across); Enter on a cell opens Stock Item Vouchers filtered to that variant. The totals equal the Stock Summary for the Shirts group, quantity exactly and value to the paisa; the header says "Stock is valued per item".
6. *Other reports on the same fields:* **Summary by field**, Sales and Credit Note (net), Item value by Size (line grain); **Pending sales orders by Colour** (Orders family); **Income and expense by Colour** (line source, the Change in stock allocated and labelled so); the Stock Summary grouped by Brand (an item attribute, exact).
7. *What it does not do (AD.6):* a different cost for each colour (one cost per item; or one item per colour; or the gated CF-9e), a reorder level per size, a price or GST rate per variant, typing a size run as a grid, a Manufacturing Journal in a chosen colour (it books Not set; CF-9f).


---

## Review log (2026-10-21, second pass)

The review's 38 findings, each re-checked against the code or the vendor page before it was accepted. **Rejected: none.** Five fixes were adapted (marked ADAPTED). "Reproduced" names what I read; a measurement I did not redo is said so.

| # | Sev | Finding | Reproduced? | Disposition (where) |
|---|---|---|---|---|
| 1 | blocker | the server keep-rule cannot fire in the main older-client case; the cost and "recorded" claims were off | **Yes** [C `sync.js loadHistories`: `latestOther` only for protected tables, `others` only newer than the base; the push route reads no `caps`; `upsertConflicts` caps at `LIMITS.openRows`] | Accepted, **ADAPTED**: a separate `current` read of the newest row of any device, instead of widening `protect`/`latestOther` (which excludes the writer's own rows and carries passcode semantics); `caps` parsed; cost stated; I/O-level test plus a deploy-time smoke; `udfKept` counted when the log is full; the editor refuses (9.3, 9.6, 7.4, 14, 15, CF7) |
| 2 | major | CF-1 "safe at any time" is false: older master forms rebuild `udf` and mutate values | **Yes** [C `masters.tsx`, `inventory/masters.tsx`, `udf.ts`, `money.ts parseAmt`] | Accepted, **ADAPTED**: neither "gate everything" nor "accept the exposure": the primary-place rule (`'v2only'` sentinel, text and choice on one master are the only lossless shape) plus `needsServerCap` refused at the door (CF16, 1.1, 9.6, 7.4) |
| 3 | major | restoring `inventory[i].udf` by identity is ambiguous when one item is on several lines | **Yes** [C `validInvLines` never merges rows] | Accepted: restore only where the identity is unique in both lists or the lists are equal line for line; mutation "match by index only" (9.3, 14) |
| 4 | major | the exemption conflated `system` with a udf exemption | **Yes** [C `putDoc`: `system` skips `checkWrite` and the user stamp; `saveVoucher` has no options; the caller list] | Accepted: `{ udf: 'exempt' }`, store-derived op and before, every caller classified (CF17, 3.3, 9.1 A8, A10 to A12) |
| 5 | major | Fill blanks: a flag batch has no global undo, `updateDoc` is not atomic, stale pushes, no per-option preview | **Yes** [C `undo.ts record()`; `markOptional.ts`] (the atomicity point holds for `updateDoc` only: `applyMark`'s outer transaction shows it is available) | Accepted, **ADAPTED**: own undo bar modelled on `markOptional`, chunks of 500 each atomic, a sync gate, per-option preview (10, CF12, 15) |
| 6 | major | "scope decides entry, never history" contradicted R3's `notAsked` | **Yes** (the spec's own text, 4.3 against 5.5 and 6.3) | Accepted: bucket by the stored value first (CF18, 4.3, 5.5, 6.3; test) |
| 7 | major | line-grain summary: count overlaps, sign undefined across bases | **Yes** [C `InventoryLine.amount` is positive on every base; `registerAmount`] | Accepted: amounts partition, counts overlap and the Total is distinct vouchers, `baseSign`, one family per summary (5.5) |
| 8 | major | "no backup-version bump" opts out of the only restore guard | **Yes** [C `backupMigrate.ts`, `backupHeader`] | Accepted: schema 3, no-op step, written only when v2 data exists; `backupmigrate.assert` extended on purpose (CF5, 12) |
| 9 | major | v1 keys are unique only within a kind; slug plus suffix reaches 43 characters | **Yes** [C `CustomFields.tsx`: `taken` per kind, `slice(0, 40)` then `_n`] | Accepted: one map per place in `CfCtx`, uniqueness stated per place and per report, key pattern 48, shortKey range 33 to 43, fixtures (1.3, 1.5, 1.6, 5.2) |
| 10 | major | "delete only at zero use" is decided on a stale replica; older editors have no guard | **Yes** [C `store.ts checkDelete` has no custom-fields case; no server rule] | Accepted: tombstones for definitions and options, a fresh-sync gate, a server rule that turns an older editor's delete into a hide, options merged by id (CF8, 10, 9.3, 9.5) |
| 11 | major | R4 assumed one item entry and a set `itemLedgerId` | **Yes** [C `tallyImport.ts` sets it only for one allocation ledger; `extraLedgerPopup` offers every ledger but the party] | Accepted: item entry at index 1, an `unpointed` reading like e-invoice's, several candidates go to `unset` and are counted (6.3 R4, 14) |
| 12 | major | TDS `TaxPrompt` and the nature picker record a decision on Esc | **Yes** [C `VoucherScreen.tsx`: `onDone(null)` "Esc to skip"; `onCancel` sets `TDS_SKIP`] | Accepted, in the back spec (B5, 5.1 to 5.2, 11) |
| 13 | major | no dirty probe for the Custom Fields editor or its panels | **Yes** [C `CustomFields.tsx` registers none; `mayHoldWork` knows three kinds] | Accepted (7.6, 14 `navstack`; back spec 4) |
| 14 | minor | browser Back read every `none` as "at the root"; no `beforeunload` | **Yes** [C grep: no `beforeunload`] | Accepted, in the back spec (2.1, 10.1) |
| 15 | minor | Back during an Accept in flight | **Yes** [C `VoucherScreen.save`] | Accepted, in the back spec (2.2) |
| 16 | minor | `url` scheme and rendering unspecified | **Yes** | Accepted: four schemes, text only in v1 (1.2) |
| 17 | minor | a user number is not necessarily additive | **Yes** | Accepted: `rules.additive` (1.3, 5.2) |
| 18 | minor | F10 type change dodges `req: 'new'`; duplicates copy hidden options; hidden `dflt` | **Yes** [C F10 `typePick` exists] | Accepted: `beforeKind`, not copying retired options, `dflt` rules (3.1, 3.2, 2.4, 1.4) |
| 19 | minor | where `sanitiseUdf` runs, truncation order, `Map` for option ids | **Yes** | Accepted: `udfView` reads, writers refuse, `Map` everywhere (1.4, 1.5) |
| 20 | minor | the `lookup` type has no delete protection | **Yes** (not in the first draft) | Accepted: `checkDelete` on five tables, Data Check, "(deleted master)" (10, CF-8) |
| 21 | minor | `tallyIndex` 20001 may collide with a partner's TDL | **Yes** | Accepted: a free index read from the target or supplied (11) |
| 22 | minor | `readValue` resolves an older app's label as a legacy id | **Yes** | Accepted: a new label may not equal any existing id or label (1.4) |
| 23 | major | "Tab reaches the line column" cannot be built; `ItemCol` is a string union | **Yes** [C `VoucherScreen.tsx`: no Tab branch in `case 'item'`, `<span onClick>` cells, `isF` compares with `===`] | Accepted: an explicit scoped Tab contract and `'cf:' + fid` focus (2.2, 15) |
| 24 | major | the new print part changes the Print Configuration form for every company | **Yes** [C `PrintConfigScreen` walk; `signature` last] | Accepted: the row only when a field prints, before Signature, baseline compared (11, 14) |
| 25 | major | Enter on an empty choice picks row 0 | **Yes** [C `pickCurrent`; Place of Supply keeps a blank] | Accepted: `popSel` -1, refuse or blank, `forms.tsx` flags a CF-1 deliverable (2.1, 2.3, 16) |
| 26 | major | no mouse path for several controls | **Yes** [C; the audit's F10 on Group Summary] | Accepted per control (2.1, 2.2, 5.5, 7.3, `cf-mouse-only`). **ADAPTED:** Pipedrive's "Important" tier is real [W] but plan-gated and not adopted in v1 (question 15) |
| 27 | major | the line-column cap and recipe cannot hold at 1024 and 1280 | **Yes by the CSS** [C fixed 110 / 110 / 140 / 150 px, unscaled]; the quoted pixel widths **not re-measured** | Accepted: inline count by measured width (at most 2), the rest on a sub-row, 12 header fields per base, editor stacks below 1100 (2.2, 1.5, 7.2, `cf-geometry`) |
| 28 | major | the editor holds typed work with no probe; "F12 family" is contradictory | **Yes** [C F12 panels are `Box` shields] | Accepted (7.2, 7.6) |
| 29 | major | the store-level check has no refusal contract | **Yes** [C `store.ts`: both return `Promise<void>`; non-form writers named] | Accepted: `UdfRefusal`, a narrowed backstop, a classified caller table (3.3, CF6) |
| 30 | minor | the identity claim was wider than what the suite asserts | **Yes** | Accepted: claim narrowed, five navigation additions listed (4.2, CF3, 14) |
| 31 | minor | two Backs with different meanings; the tooltip said Esc | **Yes** [C 17 rail Esc buttons labelled Back; `keyLabel('Esc')`] | Accepted in the back spec, **ADAPTED**: a residual below 1100 px is said, not hidden |
| 32 | minor | B6 overrides 24 closures with no per-screen proof | **Yes** [C 120 rail Esc buttons, 96 `st.pop()`] | Accepted in the back spec (6, 11) |
| 33 | minor | Back costs width at 1024 | **Not re-measured**; arithmetic on `railtop.css` agrees in direction | Accepted conservatively in the back spec (chevron only below 1100; recipes named) |
| 34 | minor | CSV import enforced `req: 'new'` per historical row | **Yes** | Accepted: history, counted, an optional tick (3.3, question 16) |
| 35 | minor | build-order gaps | **Yes** | Accepted: B-2a/B-2b, customer release gate, store.ts and forms.tsx and Workspace ownership (16, 17) |
| 36 | minor | the editor has no minimum path | **Yes** (design) | Accepted: prefilled, three blocks, More options (7.2) |
| 37 | minor | accessibility edges of the strip | **Yes** | Accepted: Tab cycle, aria, contrast pairs, status announcement (2.1) |
| 38 | minor | `describeCondition` cannot print the stale-id wording | **Yes** [C `reportFilter.ts`] | Accepted: a label resolver, `reportFilter.ts` in lane R (4.4, 16) |

Not taken from the review: nothing. Not verified in the second pass: the reviewer's own pixel measurements (rows 27 and 33), and anything that needs a live Zoho, Tally or Pipedrive account.

---

## Review log (2026-10-22, third pass: the addendum)

The review's 11 findings (lenses L1 accounting integrity, L3 persistence and older clients, L2 keyboard, L8 seams), each re-checked against the code before it was accepted. **Rejected: none** (all 11 reproduced). Five fixes were adapted (marked ADAPTED), because the fix as worded would have been wrong or insufficient in this tree. "Reproduced" names what I read; **nothing was run**.

| # | Sev | Finding | Reproduced? | Disposition (where) |
|---|---|---|---|---|
| 1 | blocker | the Physical Stock rule for a tracked item contradicts the claim that a resolver changes no existing figure: `cur` feeds `bookIn` and `bookOut`, the count is resolved in four places (the addendum named three), and a blank-variant whole-shelf count double-counts | **Yes** [C `inventory.ts stockPositions`: `cur = p.byGodown.get(...)`, `delta = round2(m.line.qty - cur)`, then `bookIn` / `bookOut` with it; the same arithmetic in `resolvedMovements`, `stockItemRows` and `ui/inventory/ageing.tsx` (lines 74 to 88), all fed by `stockMovements`; `grep stockPositions(` finds 24 matches, 23 call sites and the definition, none passing a resolver] | Accepted, **ADAPTED**: the reviewer's option A (the lines of one count are summed at item level and `byVariant` records the split) cannot group the lines "without a resolver" from the lines alone, because the engine does not know which `udf` key tracks stock; option B (thread the resolver through 23 call sites and the Split, Tally-compare and valuation boundaries) leaves a missed caller silently disagreeing and lets switching tracking off move item-level figures. **Built instead:** a `countPart` flag on the lines the voucher screen writes for a count by variant, and **one merge in `stockMovements`** (the single generator all four engines read), so every caller, with or without a resolver, reads one whole-item count of the sum and **none of the four engines is edited**; `countDeltas` spreads the same item delta over variants; the five vectors of AD.3.3 and the mutations of AD.3.9 pin it; older readers are the one stated exception (AD.3.1, AD.3.3, AD.3.4, AD.3.6, AD.3.9, 1.3, 9.1, 9.3, 12, question 24, Case 4) |
| 2 | major | Bills by field is specified to the dashboard's side rule but its proof target is the Bills report, which decides by sign and lists on-account rows; `outstandingBills` covers only bill-wise ledgers | **Yes** [C `ui/reports/outstanding.tsx`: `side === 'receivables' ? b.balance > 0 : b.balance < 0`, `grand` sums `|balance|` including on-account rows; `core/billwise.ts`: only `led.billwise` ledgers, and the opening remainder is not added as On Account; `dash/compute/receivables.ts` header documents the disagreement; CLAUDE.md lists it as open] | Accepted: the tie is the **group balance** through Pending / Unapplied / Not bill-wise columns (the card's own definition of Unapplied as the plug), a memo says the Bills reports decide by sign without promising equality, an overpaid New Ref goes to Unapplied as the card does, equality with the report is asserted only on a book where it holds, and the six fixtures are listed (AD.2.4, AD.2.7, 14, 15, CF23) |
| 3 | major | `partyId` wins wherever present, but the importer sets it on every type, so an imported Payment and its hand-entered twin bucket differently | **Yes** [C `tallyImport.ts`: `if (partyId) v.partyId = partyId` for any base; `accounting.ts voucherHeadlineAmount`: refuses `partyId` for a non-invoice base "the Tally importer sets partyId on every voucher type"; `types.ts VCH_BASES`: `invoice: true` for exactly Sales, Purchase, Debit and Credit Note, both orders, Delivery and Receipt Note] | Accepted, widened: one reader `partyOf(v)` honours `partyId` only for `baseInfo(base).invoice`; used by AD.2.1, AD.2.5, R7 and the Day Book's `cfp.` column (5.4), which had the same ambiguity; fixture and mutation added; section 19(c) rewritten (AD.2.1, AD.2.5, AD.2.6 R7, 5.4, 14, 19) |
| 4 | major | the server keep-rule for `entryPolicy` keys on "states no base", which is wrong in both directions (every client since Phase 24 stamps a base; a current client's signed-out rows carry none) | **Yes** [C `sync/baseline.ts stampBase`: stamps while signed in and a known cursor of the current namespace, and the header says rows made signed out carry none; `backend/src/conflict.js classifyPush`: `if (soft && own === null)`; the finding's own point that 9.3 says the lists cannot tell the two apart] | Accepted, **ADAPTED**: for `entryPolicy` the rule is keyed on the capability (`CAP_KEPT`, keep whatever the base, never for a writer that states `udf2`), read from `current`; for **`ui` the capability rule would be wrong** (a Phase 27 or 28 build states none and legitimately edits `ui`, so it would revert its administrator's edit), so `ui` gets an **omission** rule (`OMIT_KEPT`: restore only a `ui` the write omits, since absence is never a reset). The CF-6a code already built from the first text must be reworked before the deploy (AD.1.2, AD.1.6 (5), 9.3, 9.6, 14, 16 CF-6a and CF-14) |
| 5 | major | `pendingOrders` with the variant in the fulfilment key makes an order line with a blank variant impossible to clear | **Yes** [C `inventory.ts pendingOrders`: key `${orderRef}|${itemId}`, summed per (order, item); `VoucherScreen.tsx` `orderRefPopup` only `setOrderRef` and copies no lines; `splitCompany.ts` ~583 maps a carried line with `lines.find(x => x.stockItemId === ...)`] | Accepted, with two things found while checking: **two order lines of one item each subtract the whole fulfilled quantity today** (the normal shape for a tracked item), and Split Company's carry would give every variant line the first row's pending quantity. Fix: a per-order-line allocation for tracked items (specific first, then blank, then the "any" pool), an untracked item byte-identical including the old quirk (recorded, not fixed), the row's `line` used by Split Company, a source scan that every call site passes the fourth argument (AD.3.3, AD.3.9, AD.2.5) |
| 6 | major | writers that bypass the voucher screen (Bulk Entry, imports, REST, Alt+2, Hide, Manufacturing) silently split a tracked item's stock | **Yes** [C `BulkEntry.tsx` builds Sales and Purchase lines with no variant; the importers carry none; the REST route holds no definitions; `use.copy` can be off; Hide is allowed by the first AD.3.2] | Accepted: the new **AD.3.4b** table says what each writer does and what the product does about it; **Hide and Delete are refused while a field tracks (Stop tracking instead)**; Copy is locked on; Bulk Entry refuses a blank at voucher severity; the import previews count their Not set lines; a negative Not set is the one detector (banner and Data Check). **REST cannot warn until CF-6b** (the server holds no definitions): said, not hidden (AD.3.2, AD.3.4, AD.3.4b, 7.2, 7.4, 10, 14, 16) |
| 7 | minor | the P&L-entry guard is the group nature, but the tie target counts only the six reserved subtrees | **Yes** [C `plFigures.ts moveFromTotals` over `ledgersUnder(gid(cid, slug))`; `splitCompany.ts` uses the nature test, so the app has two rules] | Accepted: classification by the six subtrees; an Income- or Expenses-nature ledger outside them is a memo row, not in nett or `diff`; the tie is asserted on a company that has one. The opening pin also taken: `stockDeltaByItem` reads the raw opening of `stockDelta`, [C `valuation.ts newValState` settles the opening to qty x a rate truncated to five places and values it 0 under Zero cost] (6.3 R2, 6.4, AD.2.6 R10, 19) |
| 8 | minor | the Stock vouchers family cannot tie to "the sum of the lines" (a Physical Stock line is a count; a valueless Stock Journal issue line is priced at cost by the Stock Summary) | **Yes** [C `stockSummary.ts outwardValue` and `stockFlow`'s at-cost rule; `inventory.ts resolvedMovements` amount is `line.amount` for a non-count line] | Accepted: the family reads the observed movements through `stockFlow` with two additive options (`FlowOpts.bases`, `cellOf`); quantities tie to `resolvedMovements`; a full-history walk is a perf row; for a tracking field only the quantity measures are offered (AD.2.5) |
| 9 | minor | allocation gaps: no value for a (variant, godown) cell and godown cells are not value-exact; a small non-zero Q with a negative variant swings absurdly; the opening variant list can disagree with the godown list | **Yes** [C `stockSummary.ts`: godown cell `round2(oq * oRate)`, `snap(cell, rate)` per cell; arithmetic of `total x q_i / Q`; `openingStock.ts`: `variants` would be a third independent list beside `godowns` and `batches`] | Accepted, **ADAPTED** for (1): instead of the reviewer's residual rule for Outwards value cells, the variant cells carry **no Inwards or Outwards value at all** (pooled costing has no per-variant cost, a count that raises one variant and lowers another has no gross value, and "Sales by Colour" in value is Summary by field's job): only quantities, and allocated Opening and Closing value; (2) **Unallocated** whenever a variant is negative; (3) **two trust tests** for the opening list (sum, and per-godown agreement), ignored whole otherwise (AD.3.3, AD.3.5, AD.3.7, AD.3.8) |
| 10 | minor | Balances by field with overlapping chosen groups double counts | **Yes** [C `accounting.ts ledgersUnder` per group; `groupBalance` sums one group] | Accepted: the union of ledger ids, a fixture, the tie to the union (AD.2.7, 14) |
| 11 | minor | the server carry loses every variant of a size run when an older app adds or removes a line, because the identity is the same item N times | **Yes** (9.3's own rule (iii) leaves a duplicated identity unlabelled, and a size run is that by design, AD.3.6.6) | Accepted, **ADAPTED**: not "keep the current lines" (the older writer rebuilt the voucher's ENTRIES for its own lines, so keeping the current lines would desynchronise the postings from the stock), and not "extend the identity with the variant" (the older writer has none to give); instead an **iterative strong-identity pairing** (item, godown, batch, qty, rate, amount) before the weak one, which keeps the lines the older app did not touch and still never pairs two lines identical but for the variant (9.3, 14, 15) |

Not taken from the review: nothing. Not verified in the third pass: anything that needs a run (every figure, vector and mutation above is a design to be proven by the suites it names), and the 23-call-site count of `stockPositions(` is a grep, not a proof that no caller builds its positions another way.

---

## Addendum status (2026-10-22): reviewed once

The review log above covers the addendum (AD.0 to AD.8) and the passages edited to match it (AD.7) **once**. It has not been through a second pass. The lenses the next review should apply, in order of where the third pass found the addendum weakest: **L1 accounting integrity** (the count group and `countDeltas` against every reader of a stock quantity, including any that this spec did not name because it does not call `stockMovements`; Bills by field's three columns; Unallocated value); **L3 persistence and older clients** (the capability-keyed company-row rules and their deploy order; an older reader of a `countPart` voucher; the strong-identity pairing against a real older build's line rebuild); **L2 keyboard and the frozen walk** (the Physical Stock screen's pre-filled rows against Accept-on-Enter and the habitual Enter walk of a count; a tracking field's cell in the item row, AD.3.6; a bound alteration in the strip's W, AD.1.5); **L8 seams** (`inventory.ts` and `stockSummary.ts` having one writer at a time; the `ageing.tsx` move; the CF-4b-lite edit list against the grid wave's seam table). Findings are to be logged in a further table with the same columns.

## Fix round (2026-10-24, after the build review): what changed against the text above

- **Go To is lent none of the new words (section 7: "Go To (existing aliases kept)").** The Custom Fields menu row and its "+" entry carry discovery words (tags, zone, colour field, size field, attributes, extra fields, add a field ...), and `ui/paletteVocab.ts` lends every other surface's words to Go To's command for the same destination, where an alias outranks every title: `size` opened Customize Dashboard and would have opened the field editor, `user`, `tag`, `tags`, `ta`, `extra`, `attr` lost their first row, and `colour` and `zone` answered what they never answered, against the Phase 27 rule (a new alias must never move a first row; golden `goto.ranking`). The two menus keep their words (their filters find the editor by "zone"); the two rows are on `MENU_ONLY` in `paletteVocab.ts` and Go To finds the editor by the words it always had (`udf`, `custom field`, and the title). The suite `goto-vocab` pins the pairing (words stay on the menus, none reaches Go To, every other row still lends all its words) and its mutations are all killed; the golden `goto.ranking` compared on the branch differs only on `crm` and `edit`, both because of wave 3's own `Credit Limits` command, which the main tree already makes late.
- **Merge seam with parity wave 3.** This branch moved `loadHistories` out of `sync.js` into `histories.js`; wave 3 (2cee555) added `|| needsCancelCheck(c)` to that function's `protect` test, so a textual merge conflicts on `sync.js` and a resolution that keeps this side would drop the OR (the cancel rule would still fire for a writer whose base predates the shell, so no ordinary suite would notice). `histories.js` now reads the predicate off the `conflict.js` namespace and ORs it in the moment it exists (`planReads(changes, cursor, caps, cancelCheck = needsCancelCheck)`), and `customfields-sync` section 8 proves the load with wave 3's own predicate passed in, on both sides of the merge. After the merge, replace the namespace read with a plain named import (the note at the top of `histories.js` says so). The merge of `classifyPush` still needs a human: wave 3's rule 1b (the cancel rule) and this spec's 1b and 1c (custom values, the two company policies) are inserted at the same place; wave 3's goes first, and a cancelled shell needs no custom-value carry (see 9.3, fix round).
- **9.3 line pairing:** the `keep`-flag comparison, the cancelled-shell exemption, the bounded work and the residual risk of step (ii) are stated in the paragraph that follows the pairing rules in 9.3.

### Fix round, cluster "model-entry" (2026-10-24): what the entry screens and the definition model now do

- **A panel opened earlier never writes over what changed under it (9.5, completed).** `mergeForSave` took the draft's every member over the stored row's, so a save from a panel opened before another administrator's Delete, Hide, narrowed scope or Print-off brought a tombstoned field back to life and reverted the rest. The save is now a THREE-WAY merge: the screen passes the row the untouched draft would have written (`draftToRow(panel.opened)`), a member the draft left alone is whatever is stored now, and only a member the draft changed is written; `status` is never written from a draft. `staleSaveProblem` refuses, in words, the two cases that cannot be merged: the status moved (hidden, shown or deleted elsewhere) and one member (or one option) changed to two different answers; a member the stored row does not hold at all is not a rival answer. Options merge by id as before, and now three-way too (an option the draft left alone keeps another device's rename or Hide).
- **Option order is saved (found while doing the above).** `mergeOptionLists` always returned the STORED order, so the Move buttons of the option editor changed the draft and nothing else. The editor's order is now the saved order, an option the edit does not hold keeps its place after its stored predecessor, and a save that did not reorder keeps another administrator's reorder.
- **A rename keeps the old name (1.4).** `FieldOption.was` (at most 8 earlier labels, cleaned like a label) lets a value an older app wrote as the OLD label text still read as the renamed option (`readValue` step 2, after every label an option carries now); a new option may not take another option's earlier name (the sentence says whose); a name taken back leaves the list. `foldText` also drops invisible characters now, so a stored text with a zero-width character reads as its option.
- **"Update older fields" never moves a stored value (1.6).** An older choice field with an option the id rule would change (over 80 characters, or holding a control character) stays an older field: `v1UpgradeProblem` names the option, the banner says so, the button counts only what it will update, and the message says how many were left.
- **Esc and a click outside close the strip's open choice list (2.1).** The closing rule sat below the voucher's own Esc and was never reached: Esc on an open list asked "Quit ?" (or left an alteration with no question). It sits above it now, for the strip as for a line cell; a click outside the cell, the line cell and the list closes it as well.
- **A pick list stays where its rows can be seen, also under reduced motion.** `placePopup` measured the popup right after removing its inline placement; a computer that asks for reduced motion (and the app's own Reduce motion) gives every element a 0.01 ms transition, so the measurement read the OLD place, left the popup unmoved and it settled above the visible edge under the voucher's header bands. Transitions are switched off for the measuring and the write.
- **The refusal's one-click fix is clickable (2.1).** The cursor brings the WHOLE strip cell into view, and the strip leaves the height of the message lane (two lines, a token calc) below it, so a toast no longer covers "Use North (party now)" at 150 % or on a 1366 x 768 window.
- **Fill blanks asks the write right per (base, date) (10).** A role's date window or the period lock can cut across a plan; one probe per voucher type missed it and the fill stopped in the middle.
- **Go To reaches the entry rules (AD.7 item 3):** a late command "Custom-Field Entry Rules" opens Settings > Workspace > Custom-field entry rules; the `?` sheet lists the editor's keys. **The Analysis screens name their field** in recents, pins and Back ("Summary by Field: Zone"). **The editor** opens at the top with the tab strip and toolbar in view, scrolls the keyboard's choice into view, reads its tab parameter safely, and holds its books while a screen is pushed over it (no usage pass for a screen nobody can see).
- **Not done:** the six-line-field invoice re-renders every line per key (about 1.7 x the latency of the same invoice with none, 7.7 ms p50 against 4.4); memoising the item rows is an optimisation the review marked optional and a risky edit of the voucher grid, so it is left. The sticky tab strip and toolbar of a long field list (the review's second option) are not done either: the list now simply opens at the top.

