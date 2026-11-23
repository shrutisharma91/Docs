# TallyPrime parity re-check, October 2026

> **Status:** proposal for the owner. Generated 2026-10-20 for snapshot `1a87871` (the audited tree; HEAD was `fcfafaa`). Machine-readable twin: `docs/research/tally-parity-recheck-2026-10.json` (as generated; not updated).
> **Update 2026-10-22:** `docs/parity-data.json` is still NOT edited from this file's corrections as a set, but Phase 29 (parity wave 3) re-judged the 15 board rows it touched, and **section (h) lists which pending items it addressed** (tags reading "Phase 29" sit on the rows in (b), (c) and (g)). The numbers in (a) are still the 2026-10-20 proposal; the board is now 250 covered, 58 partial, 73 missing (73.2%).
> **Method:** 13 auditors (11 modules, a release-notes sweep, an everyday-use sweep) re-read every board row against the code and against TallyHelp, then a second reader (skeptic) re-checked every correction and every big new gap. Refuted claims are dropped; `[S]` = skeptic re-read and confirmed, `[A]` = single auditor, code read only.
> **Not counted:** Phase 26 wave A (`c0d9aa3`, deployed) and the uncommitted Settings and Dashboard v2 work are not in the snapshot; section (f) says what they would change.

## (a) Executive summary

1. **Corrected parity:** covered 173, partial 126, missing 82 of 381 (was 258 / 44 / 79). Weighted 61.9% (was 73.5%), using the board generator's own formula, (covered + 0.5 x partial) / total.
2. **Corrections:** 103 proposed, 98 confirmed by a second read, 5 refuted and dropped (all-voucher-bases, itc-reversal, cheque-register, http-xml-gateway, report-value-filter). Net: 86 rows down from covered, 1 up (tds-interest-latefee), 6 missing to partial, 5 partial to missing.
3. **Why the board overstated:** the first slice shipped and the status was set to covered, but the row was never re-judged against Tally's full behaviour. Most downgrades are covered to partial (82), not to missing (4: expiry reports and the PF, ESI and PT reports).
4. **The sharpest pattern is engines with no input path:** credit limit, interest rate and style, and the party price level have no field on any form, so the shipped warning, the Interest report and price lists never fire for a user. Cheapest, highest-value fixes on the list.
5. **Not on the board at all:** 199 TallyPrime capabilities (171 real gaps: 111 missing, 60 partial; 9 we already have; 18 not applicable; 1 deliberate). Many are 3.0 to 7.2 additions: Connected GST, IMS, WhatsApp, payment links, e-way, MSME 43B(h), Act-2025 TDS forms.
6. **Pending to build:** 371 rows (200 board rows plus 171 new): P0 4, P1 95, P2 163, P3 109. Largest buckets: payroll 56, platform and everyday 43, GST 39, inventory 35, manufacturing 35, advanced accounting 35.
7. **No board-P0 row is missing, but 21 of the board's 83 P0 rows are now partial (was 5).** The re-audit rates four as P0 in the sense of blocking trust: the GST return file never imported into the portal's offline tool, the Tally XML import never run on a live file, the write-once BOM master, and manufacturing stock posting (FIFO valuation mismatch).
8. **What a Tally switcher notices first:** no Alt+X cancel keeping the number, Payment/Receipt/Contra open as a double-entry grid, only CSV export, no WhatsApp or UPI on invoices, no e-way bill, no per-report rights, a device-local Edit Log, and PF/ESI/PT as projections.
9. **Confidence:** every verdict is a code read of the snapshot by one auditor and one skeptic (a few were also run: BOM rounding, TDS reversal probe, the 2B parser). There was no live TallyPrime, GST portal, IRP or customer file. Tally-side facts come from TallyHelp read through summarising fetchers, often one page, with several marked unverified; priorities and efforts are auditor judgement.
10. **Honest limits:** Tally 7.2 (12 Oct 2026) rests on its own release-notes page; manufacturing, banking, payroll, and TDS release coverage is partial (table below). Beyond-Tally work (approvals, attachments, portal, Connect, dashboards) stays out of the parity percentage.

### Board numbers, old and corrected

| | Covered | Partial | Missing | Total | Weighted |
|---|---:|---:|---:|---:|---:|
| Board as it stands | 258 | 44 | 79 | 381 | 73.5% |
| Corrected (confirmed corrections only) | 173 | 126 | 82 | 381 | 61.9% |
| Change | -85 | +82 | +3 | 0 | -11.5 pts |

The board generator prints the weighted figure rounded (73% before, 62% after); CLAUDE.md says about 74%.

Row movements: covered to partial: 82; missing to partial: 6; partial to missing: 5; covered to missing: 4; partial to covered: 1.

| Category | Rows | Old C / P / M | Old % | Corrected C / P / M | Corrected % |
|---|---:|---|---:|---|---:|
| accounting | 38 | 27 / 6 / 5 | 79.0 | 21 / 12 / 5 | 71.0 |
| admin | 19 | 15 / 2 / 2 | 84.2 | 11 / 6 / 2 | 73.7 |
| advanced-accounting | 28 | 11 / 2 / 15 | 42.9 | 3 / 9 / 16 | 26.8 |
| banking | 28 | 23 / 1 / 4 | 83.9 | 17 / 8 / 3 | 75.0 |
| gst | 44 | 32 / 7 / 5 | 80.7 | 25 / 14 / 5 | 72.7 |
| inventory | 39 | 26 / 5 / 8 | 73.1 | 18 / 12 / 9 | 61.5 |
| manufacturing | 24 | 8 / 3 / 13 | 39.6 | 3 / 9 / 12 | 31.2 |
| payroll | 49 | 31 / 5 / 13 | 68.4 | 18 / 15 / 16 | 52.0 |
| platform | 22 | 11 / 7 / 4 | 65.9 | 7 / 11 / 4 | 56.8 |
| reports | 48 | 39 / 5 / 4 | 86.5 | 24 / 19 / 5 | 69.8 |
| tds-tcs | 42 | 35 / 1 / 6 | 84.5 | 26 / 11 / 5 | 75.0 |

### Pending rows by module and priority

| Module | P0 | P1 | P2 | P3 | Total |
|---|---:|---:|---:|---:|---:|
| Accounting and vouchers | 0 | 6 | 11 | 8 | 25 |
| Advanced accounting (credit, interest, cost centres, budgets, forex) | 0 | 3 | 19 | 13 | 35 |
| Inventory | 0 | 7 | 16 | 12 | 35 |
| Manufacturing and job work | 2 | 12 | 12 | 9 | 35 |
| GST | 1 | 7 | 21 | 10 | 39 |
| TDS and TCS | 0 | 3 | 10 | 15 | 28 |
| Payroll | 0 | 24 | 26 | 6 | 56 |
| Banking | 0 | 6 | 13 | 8 | 27 |
| Reports and MIS | 0 | 9 | 13 | 9 | 31 |
| Platform, data exchange, printing and everyday UX | 1 | 10 | 16 | 16 | 43 |
| Admin, audit and security | 0 | 8 | 6 | 3 | 17 |
| **All** | 4 | 95 | 163 | 109 | 371 |

Of 371 pending rows, 121 are skeptic-confirmed corrections or big gaps; the rest are single-auditor code reads. Priorities and efforts are the owning auditor's, not the board's (the board carries 83 P0 rows, mostly covered). Where two auditors disagreed, the module that owns the board row won.

### What could not be verified, by module

| Module | Tally documentation actually read |
|---|---|
| accounting | TallyPrime help pages on vouchers, groups, CoA, numbering, interest, party ledger, shortcuts, Edit Log, Audit, release-wise page; 7.0 notes. 7.2 adds nothing accounting-side. Several pages are Tally.ERP 9 era (reversing journal, memorandum, cost-centre class, credit limits, zero-valued). |
| advanced-accounting | Interest, budgets, scenarios, cost-centre, forex, credit-limit pages; mostly un-versioned or Tally.ERP 9 (te9rel*) URLs; post-dated and Payment Performance partly from search summaries; release notes 7.0-7.2 mention nothing here (unverified that nothing changed). |
| inventory | Inventory, orders, godowns, batches, price lists, reorder, cost tracking, stock query, ageing pages; no inventory feature found in 5.x to 7.2 notes. Several pages returned HTTP 500 (valuation methods, godown summary, third-party godowns): those are unverified. Serial-number tracking not found as a native feature. |
| manufacturing | Notes read for 4.0, 5.1, 6.0, 6.2, 7.0, 7.1, 7.2; NOT read for 1.x, 2.x, 3.x, 5.0, 6.1. BOM/MJ help partly read through search summaries; nested BOM, wastage % and shortage report not documented by Tally. |
| gst | Notes read for 1.1, 2.0, 3.0, 3.0.1, 4.0, 4.1, 5.0, 5.1, 6.0, 6.1, 6.2, 7.0, 7.1, 7.2 (best coverage). GSTR-1 amendments and ITC-04 only from Tally.ERP 9 or third-party pages; 7.2 date rests on one third-party blog. |
| tds-tcs | 7.1 confirmed in its notes; 7.2 TDS/TCS content only from a banner and snippet. Form 16A and 27D only in older Tally.ERP 9 docs. |
| payroll | Payroll help pages (not release-tagged) plus notes 3.0, 6.0, 7.2; 5.1 payroll compliance note from a snippet only; statutory bonus and the Form 138 effective year unverified. |
| banking | Notes 6.0, 7.0, 7.1 read; 1.x-5.x not retrievable, so older items come from current help pages. PDC and Cheque Register pages seen only as snippets. |
| reports | Notes 1.0, 2.0, 3.0, 4.0, 4.1, 5.0, 6.1, 7.0 and report help pages; 7.1 (TallyIra) only from press coverage. |
| platform | Notes 4.0-7.2 plus export, import, backup, licensing pages; whether Tally hard-locks a record and the TallyPrime company-delete steps are unverified. |
| admin | Notes 3.0, 4.0, 5.1, 6.0-7.2; Edit Log, TallyVault, Audit, group-company pages; per-switch F11 list and password-policy defaults not fetched. |
| release-sweep and everyday | All release notes 1.0 to 7.2 (paraphrased through a page summariser); WhatsApp, payment request and SmartFind pages seen as snippets only. |

Common to all: no live TallyPrime was available, so nothing is compared against a running Tally; no GST portal, IRP, NSDL FVU or bank was touched; most skeptics ran out of web-search budget and confirmed the Tally side from one help page or from knowledge, which is why unverified notes survive in the rows below.

## (b) The pending list: what is in TallyPrime and not here

Columns: feature, what exactly is missing (`partial` = part exists), who needs it, priority P0 (blocks trust) to P3 (rare), effort S/M/L/XL, planned or blocked by. Sorted by priority then effort within a module. `[S]` skeptic-confirmed, `[A]` single auditor. Rows that merge several board ids list them in the JSON `boardIds`.

### b1. Should build

#### Accounting and vouchers (25 rows: P0 0, P1 6, P2 11, P3 8)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| Group master integrity [S] | **partial**: No duplicate group-name check (also against ledger names); no block on re-parenting under own descendant; children keep old nature when a group moves | accountant | P1 | S | none |
| Prefix and suffix with applicable-from dates [S] | **partial**: Dated prefix/suffix; altering a voucher must keep its issued numberStr unless number or date changed (today an old invoice number is silently rewritten) | GST filer | P1 | S | none |
| Cancel voucher (Alt+X) keeping its number [S] | **partial**: Cancel action missing (register of deletions only; board row corrected to partial): a cancelled voucher must keep its number outside the books; delete leaves a gap in the GST invoice series | GST filer, accountant, auditor | P1 | M | briefed ROADMAP-ADDENDUM (Voucher.cancelled flag); voucher lifecycle model pending (research README) **Phase 29 (2026-10-22): BUILT (Alt+X, shell voucher, register, Table 13); board now covered; see (h)** |
| Cost-centre allocation on item invoices [S] | **partial**: Allocation on the sales/purchase ledger and charge lines of Sales/Purchase/Credit/Debit Note item invoices; keep imported allocations on alteration | accountant, project or multi-branch firm | P1 | M | none |
| Group nature and Affects Gross Profit drive the P&L [S] | **partial**: P&L and Trading must read the group tree; a user-made primary Income/Expense group is on the Balance Sheet profit but never on the P&L | accountant, manufacturer | P1 | M | briefed ROADMAP-ADDENDUM (group-advanced-flags, plSections) |
| Single-entry layout for Payment, Receipt, Contra (TallyPrime default) [A] | **partial**: Account first, then particulars with amounts, Ctrl+H to double entry; ours always shows the Dr/Cr grid (board says P2) | every Tally switcher, bookkeeper | P1 | L | briefed ROADMAP-ADDENDUM **Phase 29 (2026-10-22): BUILT as an opt-in layout, default still the grid; board stays partial; see (h)** |
| Alias on every master, several aliases, language alias [S] | **partial**: Only ledgers and stock items have one alias; groups, stock groups, cost centres, godowns, units, voucher types have none | accountant, regional-language user | P2 | S | none |
| Reference number and date on any voucher type (new) [A] | **partial**: Only Purchase item invoices carry supplier invoice no. and date; Credit/Debit Note, 'as voucher' and accounting vouchers have none | accountant | P2 | S | none |
| Change voucher type on alteration; convert a Memorandum to a real voucher (new) [A] | **missing**: F10 works only when creating; a Memorandum can only be duplicated, leaving the memo behind | accountant | P2 | M | none |
| Create or alter any master from a picker (Alt+C, Ctrl+Enter) (new) [A] | **partial**: Alt+C only on ledger fields; no create for stock item, unit, godown, cost centre, group, voucher type; no Ctrl+Enter alter anywhere | data-entry operator | P2 | M | Wave A '+' quick-create is menu-level, not in-voucher (not counted) |
| Narration for each ledger line [S] | **missing**: Per-line narration on VoucherEntry, voucher-type switch, grid column, Day Book and ledger display | bookkeeper (multi-line journals and payments) | P2 | M | briefed ROADMAP-ADDENDUM |
| Multi Create / Multi Alter / Multi Display (Alt+H) [S] | **partial**: No in-app grid to create or alter groups and ledgers, no bulk openings (Alt+B); CSV creates only 6 columns and cannot set address, PAN or credit fields | accountant, CA onboarding a client | P2 | M | briefed ROADMAP-ADDENDUM (grid modelled on Bulk Entry) |
| Automatic (Manual Override), Multi-user Auto, None numbering [A] | **missing**: Only Automatic and Manual; no provisional number, no None; duplicate check is an overridable warning and only for Manual; alter-last-number and keep-number-on-date-change missing | multi-user firm, GST filer | P2 | M | server /sync/next-number exists but no client calls it **Phase 29 (2026-10-22): PARTLY: server numbers, provisional numbers and clash reporting built (backend half not deployed); override and None not built; see (h)** |
| Voucher type options [A] | **partial**: Effective dates, allow zero-valued, narration per ledger, additional purchase costs, share-after-save, default accounting allocations | accountant, trader | P2 | M | effective dates: ROADMAP-ADDENDUM says skip **Phase 29 (2026-10-22): PARTLY: Entry layout only; see (h)** |
| Multiple addresses per party, Bill-to/Ship-to, dated party GST details [A] | **missing**: Address book per ledger, GSTIN per location, Bill-to/Ship-to on the voucher; dated party GST registration history is the P3 part | trader, distributor | P2 | L | briefed ROADMAP-ADDENDUM (Ledger.addresses[]) |
| Numbering series, restart table, starting number [A] | **partial**: Several series in one voucher type, restart by applicable-from date, Starting number field (workaround: one voucher type per series) | GST filer with several series | P2 | L | ROADMAP-ADDENDUM suggests skipping |
| Voucher classes [A] | **missing**: Saved template per voucher type: default allocations with % splits, tax and round-off ledgers, party-group restriction, POS modes, interest class (board says P1) | accountant, retailer | P2 | XL | briefed ROADMAP-ADDENDUM; ghost recall and auto round-off cover common cases |
| Party contact depth (new) [A] | **partial**: Several named mobiles, default WhatsApp number, CC e-mails, fax, website, country | trader | P3 | S | none |
| Related Reports (Ctrl+O) from a voucher, ledger or report (new) [A] | **missing**: Context menu of reports about the item under the cursor; our palette is global only | accountant | P3 | S | none |
| Unused-masters list with bulk delete; bulk party GSTIN and e-mail update (new) [A] | **missing**: Ctrl+J exceptions under Chart of Accounts | CA cleaning a client's masters | P3 | S | none |
| Zero-valued transactions [A] | **partial**: Zero-quantity lines and zero-amount accounting rows are dropped or refused; free items already work | trader | P3 | S | ROADMAP-ADDENDUM: skip |
| Bill-wise depth: non-party ledgers, sort, pre-allocate (new) [A] | **partial**: Bill-by-bill only on Debtors/Creditors; pending bills not sortable (due/bill date/balance), no pre-allocation, no table of bills | accountant tracking loans and instalments | P3 | M | none |
| Chart of Accounts as the masters workbench (new) [A] | **partial**: Group view, groups with no ledgers, all master types, F12 display names, edit-log filter, create inside | accountant | P3 | M | none |
| Insert (Alt+I) and Add (Alt+A) in Day Book [A] | **partial**: Duplicate works; insert-in-sequence and add not built | accountant | P3 | M | insert is a deliberate skip (renumbering breaks GST serials) |
| Group advanced flags [A] | **missing**: Behaves as sub-ledger, nett Dr/Cr, used for calculation, allocation method; Allow Advanced entries switch | accountant, trader | P3 | M | briefed ROADMAP-ADDENDUM |

#### Advanced accounting (credit, interest, cost centres, budgets, forex) (35 rows: P0 0, P1 3, P2 19, P3 13)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| Credit limit per party with enforcement [S] | **partial**: No ledger-master field to enter a limit (S); blocking instead of warn-only, accounting-mode vouchers, edited vouchers, creditor side, post-dated (M) | trader, distributor | P1 | S-M | briefed ROADMAP-ADDENDUM (credit-limit-override) **Phase 29 (2026-10-22): PARTLY: field, shared engine, reason on override built; blocking mode and accounting-mode vouchers not; see (h)** |
| Interest on settled-late and open bills, grace period, calculate-from [S] | **partial**: Only bills open at period end are charged; no settled-late bills, grace period, calculate-from or per-slab style | trader | P1 | M | briefed ROADMAP-ADDENDUM **Phase 29 (2026-10-22): PARTLY: open bills charged from the due date to the as-on date; settled-late and grace period not built; see (h)** |
| Interest parameters on the ledger master and Enable Interest Calculation [S] | **partial**: No way to set rate/style/balances on a ledger so the report is always empty; engine charges closing balance for the whole period, not the daily balance; no F11 switch; no bank-date basis | trader, lender, anyone charging on overdues | P1 | M | briefed ROADMAP-ADDENDUM (interest-advanced, bank-interest) **Phase 29 (2026-10-22): PARTLY: rate and style fields and a daily-balance engine built; F11 switch not; see (h)** |
| Base currency configuration [S] | **missing**: Company has no currency settings; INR, 2 decimals, Indian grouping and Rupees/Paise hard-coded | exporter or non-INR user | P2 | S | ROADMAP-ADDENDUM recommends not building (fixed INR) |
| Cost centres on balance-sheet ledgers [S] | **partial**: Flag offered only for Income/Expense ledgers | accountant (CWIP, project tracking) | P2 | S | none |
| Budget on cost centres [A] | **missing**: BudgetLine targets only group/ledger; cost-centre actuals not read | project or department head | P2 | M | briefed ROADMAP-ADDENDUM |
| Budget basis: closing balance vs nett transactions [S] | **partial**: Only nett transactions over the whole period, so balance-sheet budgets read wrong | owner, finance head | P2 | M | none |
| Overdue-party warning at entry; credit period in days, weeks, months or years (new) [A] | **missing**: Warn when the party already has bills past its credit period; units other than days | trader | P2 | M | none |
| Cost Centre Monthly Summary and column comparison (new) [A] | **missing**: Month-wise totals with drill, Ctrl+H view change, Alt+C/Alt+N columns on the three cost reports | accountant | P2 | M | none |
| Parallel allocation across cost categories [A] | **partial**: Allocate 100% under each category independently plus category flags; ours is one flat pool | larger firm (department + project) | P2 | M | briefed ROADMAP-ADDENDUM |
| Credit-limit override and exceeded handling [A] | **missing**: Per-party override using post-dated transactions, exceeded-limit report (board wording differs from Tally's documented override) | trader, credit controller | P2 | M | briefed ROADMAP-ADDENDUM; needs credit-limit entry first **Phase 29 (2026-10-22): PARTLY: reason-recorded override and a Credit Limits report built; see (h)** |
| Credit Limits bulk screen (Alter > Credit Limits) (new) [A] | **missing**: Group-wise grid for limit, credit period and check-overdue flag | trader | P2 | M | easy once credit-limit has a field **Phase 29 (2026-10-22): NOT built (the Credit Limits report is a report, not this grid); see (h)** |
| Currencies master [A] | **missing**: Symbol, formal name, ISO code, decimals, prefix/suffix, words wording | exporter, importer | P2 | M | briefed ROADMAP-ADDENDUM |
| Invoices printed in a foreign currency (new) [A] | **missing**: Invoice shows FCY amounts, rate and amount in words | exporter | P2 | M | depends on currencies master and forex entry |
| Book interest as a Debit/Credit Note (interest class) [A] | **missing**: A 'book interest' action creating one Debit/Credit Note per party; the report is read-only | trader | P2 | M | briefed ROADMAP-ADDENDUM (Book Interest journal action instead of classes) |
| Interest report depth [S] | **partial**: Flat ledger list only: no group view, bill-wise detail (Alt+F5), settle-bills, pending/overdue basis, period comparison | accountant, trader | P2 | M | none **Phase 29 (2026-10-22): NOT built (receivable and payable are now shown apart); see (h)** |
| Post-dated vouchers held out of the books until their date [S] | **partial**: Exclusion from ledger and balance reports until the date, Include-post-dated switch (Alt+F8); flag, badge and register exist | trader taking post-dated cheques | P2 | M | none |
| Cost centre class (preset % allocation) [A] | **missing**: Preset splits that auto-fill the cost panel (ERP 9 help; TallyPrime screens unverified) | multi-branch or project firm | P2 | L | briefed ROADMAP-ADDENDUM (Ledger.costSplit instead of classes) |
| Forex gain/loss: realised, unrealised, unadjusted [A] | **missing**: Ledger Forex Gain/Loss, unadjusted line in Balance Sheet, adjustment journal; AS 11 vs AS 21 treatment to be decided | exporter, importer, CA | P2 | L | briefed ROADMAP-ADDENDUM; needs voucher class or dedicated screen |
| Ledgers in a foreign currency (new) [A] | **missing**: Per-ledger currency, FCY opening balance with rate, FCY bank accounts | exporter, importer | P2 | L | none |
| Foreign-currency voucher entry [A] | **missing**: Amount + rate on entries with base value derived; XML import keeps only the INR figure | exporter, importer | P2 | L | briefed ROADMAP-ADDENDUM |
| Advanced interest parameters [A] | **missing**: Several rates with applicable dates, per annum/month/day, 365 or calendar, rounding, compound, per-voucher override | lender, trader | P2 | L | briefed ROADMAP-ADDENDUM |
| Budget period-wise apportionment [A] | **missing**: One lump from..to, no monthly phasing (Tally behaviour unverified) | owner | P3 | S | briefed ROADMAP-ADDENDUM |
| Budgets master: sub-budgets and scope [S] | **partial**: Under/sub-budget nesting and cost-centre scope | owner, finance head | P3 | S | none |
| Cost Category master flags [S] | **partial**: Alias and allocate-revenue / non-revenue flags | accountant | P3 | S | none |
| Cost centre master extras (new) [A] | **partial**: Alias, revenue opening balances, bank details and e-mail, create-on-the-fly | accountant | P3 | S | none |
| Credit period as a date, counted from voucher, invoice or effective date (new) [A] | **partial**: Days from voucher date only | trader | P3 | S | effective dates deliberately skipped |
| Ledger Vouchers with cost-centre details (F12) (new) [A] | **missing**: Cost allocation lines under each voucher | accountant | P3 | S | none |
| Rates of exchange table [A] | **missing**: Standard, buying, selling, last-voucher rates by date | exporter, importer | P3 | M | briefed ROADMAP-ADDENDUM |
| Forex columns in outstanding and ledger reports [A] | **missing**: No FCY balance anywhere | exporter, importer | P3 | M | briefed ROADMAP-ADDENDUM |
| View any report in another currency (new) [A] | **missing**: Column currency switch and side-by-side currencies with the rate used | exporter, CA | P3 | M | depends on rate table |
| Payment performance (average days to pay) [A] | **missing**: Per-party paid invoices with days due and delay; the drill target of Ratio Analysis 'Recv. Turnover in days' | credit controller | P3 | M | briefed ROADMAP-ADDENDUM (core/payperf.ts) |
| Reversing Journal with applicable-upto date, and its register [A] | **missing**: Non-posting journal that auto-reverses after a date and shows only in scenarios; register | CA, finance head | P3 | M | briefed ROADMAP-ADDENDUM; low value without scenarios |
| Bill settlement in foreign currency [A] | **missing**: BillAllocation is base-only; no FCY per bill (Tally itself does not track bills on non-base ledgers) | exporter, importer | P3 | L | briefed ROADMAP-ADDENDUM |
| Scenarios master and scenario columns / variance [A] | **missing**: Include/exclude optional, memorandum, reversing and actual vouchers; Alt+C scenario column with variance | CA, finance head | P3 | L | briefed ROADMAP-ADDENDUM |

#### Inventory (35 rows: P0 0, P1 7, P2 16, P3 12)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| Inventory F12 configuration [A] | **partial**: Actual vs billed qty, MRP, rejection columns, order-no-per-item and tracking-no options, negative-stock and availability warning switches | experienced Tally users migrating | P1 | M | briefed ROADMAP-ADDENDUM |
| Item opening per godown/batch [A] | **partial**: No item-master UI to split opening qty by godown or batch (engine supports openingAlloc) | multi-godown and batch businesses migrating mid-year | P1 | M | briefed ROADMAP-ADDENDUM (v4/M) |
| Sales/Purchase order tracking [S] | **partial**: Per-item order numbers, line prefill from the order, closed/partial status, duplicate-order warning | traders, makers selling on orders | P1 | M | none |
| Additional cost of purchase (apportioned overheads) [A] | **missing**: Freight/loading/insurance by qty or value into item cost, in invoice and voucher modes | importers, traders, makers wanting landed cost | P1 | L | briefed ROADMAP-ADDENDUM (v4/L) |
| Batches with mfg/expiry dates [S] | **partial**: No existing-batch picker with balances, mfg date never entered, no opening batches in item master, no Stock/Manufacturing Journal batches, Batch Summary is a ComingSoon stub in the menu, no Batch Vouchers/Negative Batches, no expired-sale warning | pharma, FMCG, food, chemicals traders and makers | P1 | L | none (board said covered) |
| Delivery/Receipt Note linked to the invoice (tracking numbers) and Bills Pending reports [S] | **partial**: No tracking link, so note + invoice moves stock twice and double-counts order fulfilment; no billed-not-delivered / delivered-not-billed reports; no rejection against a note | any trader or maker who delivers before billing | P1 | L | none; data-integrity risk, not only a gap |
| Price levels and price lists [S] | **partial**: Party default price level has no field and the voucher has no price-level pick, so lists never reach an invoice; no dated lists, group/category lists, slabs, discount, Alt+R | traders, distributors | P1 | L | price-slab part briefed; the unreachable party field is not briefed **Phase 29 (2026-10-22): PARTLY: party level field and the Alt+Z pick built; dated lists and slabs not; see (h)** |
| Inventory voucher registers (Stock Journal, Physical Stock, Delivery/Receipt Note, Rejections, Order Books) (new) [A] | **partial**: RegisterReport can already render these but the menus expose only Sales, Purchase, Journal, Debit/Credit Note | stock keepers, auditors | P2 | S | none |
| Rejections Out voucher [A] | **missing**: Stock-only outward voucher before the purchase invoice; also how Tally lowers stock value | traders and makers using receipt notes | P2 | S | briefed ROADMAP-ADDENDUM (v2/S) |
| Stock item / group monthly summary (new) [A] | **missing**: Month-by-month inwards/outwards/closing between Stock Summary and vouchers | accountants, stock keepers | P2 | S | none |
| Actual vs billed quantity [A] | **missing**: Separate delivered and billed columns; free items (10+1) move stock without billing | FMCG, pharma, scheme-based traders | P2 | M | briefed ROADMAP-ADDENDUM (v2/M) |
| Expiry reports / expired-stock handling [S] | **missing**: Near-expiry and expired batch report, ageing by expiry date, warn or block an expired sale | pharma, FMCG, food | P2 | M | none |
| Movement Analysis (group, category, party) [S] | **partial**: Stock group/category analysis, party-wise analysis, effective rate with additional cost, Stock Transfer Analysis, drill to party then voucher | traders analysing movers and who buys what | P2 | M | none |
| Order due dates, lot splits and pre-close (new) [A] | **missing**: Due date per lot with overdue warning; pre-close whole or part with reason and date | order-based sellers | P2 | M | none |
| Order outstanding views, ageing and Order Books (new) [A] | **partial**: Group/item/ledger/all views, days overdue, ageing, goods without order, monthly Order Book | sales and purchase managers | P2 | M | none |
| Rejections In voucher [A] | **missing**: Stock-only inward voucher against a delivery-note tracking number | traders and makers using delivery notes | P2 | M | briefed ROADMAP-ADDENDUM (v2/M); needs tracking numbers to be Tally-faithful |
| Reorder levels and status [S] | **partial**: Minimum order qty, order-to-be-placed, availability net of pending orders, group/category and consumption-based levels | traders, retailers | P2 | M | none |
| Stock Ageing styles and custom slabs [S] | **partial**: By expiry/mfg date, to-be-expired, custom slabs, expired-only, godown filter | any stock holder; pharma and food for expiry styles | P2 | M | none |
| Stock Journal additional cost on transfer, and wastage (new) [A] | **missing**: Freight/loading on a goods transfer; received-vs-issued shortage | multi-branch traders, makers | P2 | M | none |
| Stock Summary batch, alternate-unit and order-position columns (new) [A] | **partial**: Batch breakup, alternate-unit quantities, pending-order and net-available columns | batch and order-driven businesses | P2 | M | none |
| Standard cost / selling price valuation [A] | **partial**: Applicable-from dated standard cost and selling price lists; today one undated rate restates history | makers using standard costing | P2 | M | briefed ROADMAP-ADDENDUM |
| Quantity-slab / discount price lists [A] | **missing**: Quantity bands, band discount %, applicable-from date on a price list | distributors with volume pricing | P2 | L | briefed ROADMAP-ADDENDUM (priceFor design) |
| Item cost tracking (Item Cost Analysis / Cost Track Breakup) [A] | **missing**: Tracking numbers across purchase, sales, orders, payments; cost analysis and breakup | project and trading firms wanting per-lot profit | P2 | XL | briefed and marked Skip (niche) |
| Line discount as amount or both; discount ledger (new) [A] | **partial**: Percentage only today; Tally added amount/both in 7.1; per-item discounts into a Discount ledger | retailers, traders | P3 | S | none |
| Item description / remarks [A] | **partial**: Master-level description; not in XML export or API; not on Stock Journal | services, quotation-heavy traders | P3 | S | briefed ROADMAP-ADDENDUM |
| Stock group option: should quantities be added (new) [A] | **missing**: Per-group switch | mixed-unit groups | P3 | S | none |
| Stock Query completeness [S] | **partial**: Recent N purchases/sales with voucher drill, Top Buyers/Sellers, vertical/horizontal, nett rate | sales and purchase desk staff | P3 | S | none |
| Per-item market valuation method [S] | **partial**: Market method is a screen toggle not an item field and feeds no report | accountants wanting unrealised-profit views | P3 | S | none |
| Value-only stock items (no unit) (new) [A] | **missing**: Item with no unit adjusted by value only | accountants adjusting stock value | P3 | S | none |
| Bulk Revise Price (Alt+R), price-list export/import (new) [A] | **missing**: Raise/lower by amount or % from a date; copy lists between companies | distributors revising lists | P3 | M | needs dated price lists first |
| Godown attributes: storage flag, address on vouchers, third-party and consignment godowns (new) [A] | **missing**: Allow-storage switch, godown address on invoices, third-party godown type | multi-site traders, consignment sellers | P3 | M | third-party type is the job-work row jobwork-godown-type |
| Inventory-only company / accounts-inventory integration off (new) [A] | **missing**: No inventory-without-accounts mode | stock-only users (rare) | P3 | M | none |
| MRP and tax-inclusive MRP (tobacco deemed taxable value) [A] | **missing**: Date-wise MRP on item, MRP column on voucher/print, MRP edit switch, GST computed on MRP | pharma, FMCG, tobacco and pan distributors | P3 | M | briefed ROADMAP-ADDENDUM (v2/M); cess status after Sept 2025 unconfirmed |
| Item behaviour options and other valuation features [A] | **partial**: Four behaviour options, value-only debit/credit notes, rejection valuation, per-godown/batch FIFO lots, yearly-restart FIFO/LIFO | made-to-order makers, accountants reconciling to Tally | P3 | M | briefed; needs a live TallyPrime comparison |
| Per-godown rate and reorder override [A] | **missing**: Rate and reorder level per godown | multi-warehouse businesses | P3 | L | briefed and marked Skip |

#### Manufacturing and job work (35 rows: P0 2, P1 12, P2 12, P3 9)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| BOM master on the Stock Item (Set Components) [S] | **partial**: No screen to set, alter or delete a BOM; written once via Save as BOM on first manufacture; no unit of manufacture, godown per component, copy-BOM | manufacturer | P0 | M | briefed ROADMAP-ADDENDUM (Bom model); nothing built |
| Consumption/production posting with correct valuation [A] | **partial**: Components priced at average cost but FIFO/LIFO items are issued from lots, so values disagree; no batch, hard-coded godown, no negative-stock warning | maker on FIFO costing; batch-managed maker | P0 | M | briefed ROADMAP-ADDENDUM (costManufacture()); not built |
| BOM quantity precision [S] | **partial**: Ratios rounded to 2 decimals so 5 per 1000 doubles and 4 per 1000 vanishes; store exact or a manufacture-unit base | food, chemical, cosmetic makers (small ratios) | P1 | S | found by this audit; not briefed |
| Component and product godown in the Manufacturing Journal [S] | **partial**: Journal always consumes from and produces into Main Location | maker with raw-material, WIP and FG godowns | P1 | S | briefed with mfg-stock-posting |
| BOM carried in Tally migration (import and export) (new) [S] | **missing**: tallyImport has no BOM handling; tallyExport lists BOM as not carried; tag names unverified | maker switching from Tally; Tally partner running migrations | P1 | M | needs a live TallyPrime export |
| Multiple named BOMs per finished item [A] | **missing**: Named BOMs (Standard, customer) and a BOM choice at production | maker with customer-specific variants | P1 | M | briefed; depends on the BOM master |
| Third-party godown type [A] | **missing**: 'Our stock with third party' and 'third-party stock with us' godown types | principal manufacturer; anyone with stock at a third-party warehouse | P1 | M | briefed ROADMAP-ADDENDUM |
| Manufacturing Journal as a distinct voucher type [S] | **partial**: Saved as a plain Stock Journal (no own type, series, abbreviation); alteration opens the generic grid; no co-product section; no batch/optional/reference | manufacturer | P1 | M | not briefed; editId path exists but nothing routes to it |
| Batch, mfg date and expiry on Manufacturing Journal lines (new) [S] | **missing**: A batch-managed item produced through the MJ enters stock with no batch; consumed lines name no batch | pharma, food, chemical makers | P1 | M | mentioned ROADMAP-ADDENDUM (mfg-stock-posting) |
| Co-products, by-products and scrap output [A] | **missing**: One finished line only; no Component/By-product/Co-product/Scrap type and no % cost allocation | makers in sugar, oil, metal, food | P1 | L | briefed ROADMAP-ADDENDUM |
| Material In voucher [A] | **missing**: Receive finished goods from a job worker; consumption, scrap/by-product return, shortage/wastage, job charges | principal manufacturer, job worker | P1 | L | briefed ROADMAP-ADDENDUM |
| Material Out voucher [A] | **missing**: Move material to a job worker without a sale; order selection, challan, dispatch details | principal manufacturer | P1 | L | briefed ROADMAP-ADDENDUM |
| Cost allocation across multiple outputs [A] | **missing**: Apportion input plus overhead across outputs by %, primary takes the remainder | maker with co-products | P1 | L | briefed (allocateCost()); not built |
| Job Work Out Order [A] | **missing**: Order to a job worker with dispatch details, due date, Track Components, BOM allocation, nature of process | principal manufacturer (textile, engineering, garments) | P1 | XL | briefed; needs Material In/Out and a godown type |
| Job work setup: Enable Job Order Processing, Material In/Out voucher-type options (new) [A] | **missing**: No company switch or voucher-type options | principal, job worker | P2 | S | blocked by job-work vouchers |
| e-Way Bill on Material In / Out (new) [A] | **missing**: No e-way for job-work movements | principal moving goods by road | P2 | M | blocked by job-work vouchers |
| GST on Material In / Material Out vouchers (new) [A] | **missing**: No tax ledgers or GST applicability on job-work movements | principal, job worker, CA | P2 | M | blocked by job-work vouchers |
| Pre-close job work orders (new) [A] | **missing**: Close unfulfilled or damaged orders | principal manufacturer | P2 | M | blocked by job-work orders |
| Job work order and challan printing (new) [A] | **missing**: No print for orders, Material Out/In, co-product/by-product/scrap list | principal, transporter | P2 | M | blocked by job-work vouchers |
| Material In auto-consumption of raw material (new) [A] | **missing**: Consume material from a consumption godown using the BOM on receipt | principal manufacturer | P2 | M | blocked by Material In and BOM master |
| Additional manufacturing cost by expense ledger and % [S] | **partial**: One typed lump only; no cost heads, % of component cost, nothing remembered per BOM | maker loading labour and power | P2 | M | partly briefed (costManufacture()) |
| Manufacturing journal printing (cost sheet) [A] | **partial**: Prints as a generic Stock Journal: no overhead line, effective rate, BOM name | maker, auditor | P2 | M | briefed (buildCostSheetModel) |
| Job costing [A] | **partial**: Godown linked to a job, consumption by job, Job Work Analysis; cost-centre P&L exists | contractor, fabricator, project maker | P2 | L | blocked by voucher-class and cost-tracking |
| Job work order tracking [A] | **missing**: Ordered vs issued vs received per order, Job Work Orders Summary | principal manufacturer | P2 | L | briefed ROADMAP-ADDENDUM |
| Job work registers and analysis [A] | **missing**: Material registers, Material Movement Register, Stock With Job Worker, variance analysis | principal, job worker, CA doing ITC-04 | P2 | L | briefed ROADMAP-ADDENDUM |
| Job Work In Order [A] | **missing**: Order as job worker; principal-owned stock kept out of our valuation | job worker | P2 | XL | briefed and deprioritised |
| Copy BOM from another BOM or item (new) [A] | **missing**: Needs the BOM master first | maker | P3 | S | depends on bom-definition |
| Source godown stored on each BOM component (new) [A] | **missing**: BOM line holds componentId and qty only | maker with several stores | P3 | S | none |
| Unit of manufacture on a BOM (new) [A] | **partial**: Per-1-unit ratios only (precision defect is bom-scaling) | maker | P3 | S | none |
| BOM wastage and substitutes [A] | **missing**: No wastage % or alternate components; Tally basis thin | maker with process loss | P3 | M | briefed ROADMAP-ADDENDUM |
| Cost Estimation (Item Estimates) report (new) [A] | **missing**: BOM-based cost estimate by stock group/item | maker, costing accountant | P3 | M | depends on BOM master |
| Manufacturing Journal F12 options and voucher flags (new) [A] | **missing**: No F12, ref no/date, optional status, per-item description, alt/compound units, negative-stock toggle | maker | P3 | M | none |
| Purchase requisition for manufacturing (new) [A] | **missing**: BOM-driven requisition (optional voucher type on the MJ) | purchase officer in a factory | P3 | M | depends on BOM master and shortage check |
| Nested BOM roll-up [S] | **partial**: Sub-assemblies work via separate journals; no recursive explosion or roll-up (Tally's own behaviour undocumented) | assembler with semi-finished goods | P3 | L | briefed ROADMAP-ADDENDUM |
| Component availability and negative-stock warning on production [A] | **missing**: No on-hand column or warning when consumption exceeds stock (warning S, shortage view L) | maker, store keeper | P3 | L | briefed ROADMAP-ADDENDUM |

#### GST (39 rows: P0 1, P1 7, P2 21, P3 10)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| GST returns export formats (portal JSON, Excel/CSV) [A] | **partial**: JSON built but never imported into the offline utility or uploaded (SCHEMA? fields: hsn_b2b/b2c, WOPAY, itc_rev ty codes); no Excel/CSV layouts, no section-wise IFF JSON | every GST filer without portal API access | P0 | L | needs a real GSTR-1 and 3B from the owner's CA (ROADMAP-ADDENDUM); online route needs a GSP |
| GST exception resolution (Uncertain transactions) [S] | **partial**: No Accept as is, bulk correction, in-place fix, B2B-to-B2C for inactive GSTIN, Accepted-As-Valid list (name unconfirmed) | accountant preparing returns | P1 | M | none |
| GST rate source tiers (classification, group, apply-to-all) [A] | **partial**: No Classification or accounting-group tier; one order serves rate and HSN; default is item-first while Tally's is ledger-first | accountant, Tally migrators with ledger-first setups | P1 | L | owner decision on default order; needs Dexie v9 |
| GSTR-2A/2B reconciliation depth [S] | **partial**: b2b only; no CDNR/2A/imports, tax-head comparison, potential matches, ignore-prefix options, manual status, ITC accept/reject, reason, re-import, no effect on 3B, no portal download | accountant, CA, trader claiming ITC | P1 | L | offline parts need no contract; download needs a GSP |
| Connected GST: upload, file, download returns from the app (new) [S] | **missing**: GSTR-1, 3B, CMP-08 upload and filing (DSC/EVC), download of 1/3B/2A/2B | every GST filer | P1 | XL | needs GSP, TLS, credential vault (einvoice-gsp.md) |
| e-Invoice online IRN generation, cancel [A] | **partial**: No IRP call; INV-01 omits Dispatch-From, Ship-To, e-way and payment blocks; no cancel API, no signed-QR verification; offline export/import loop exists | every seller above the e-invoice threshold | P1 | XL | needs a GSP contract, TLS, credential vault (einvoice-gsp.md) **Phase 29 (2026-10-22): PARTLY: DispDtls, ShipDtls and EwbDtls are written; still no IRP call; see (h)** |
| e-Way Bill generation and management [A] | **missing**: No Part A/B capture (transporter, vehicle, distance), no EWB JSON, no cancel/extend/Part-B update/consolidated, no QR print; only an imported EWB number is stored | traders, makers, distributors | P1 | XL | offline bulk-upload JSON needs no contract (L); live needs a GSP **Phase 29 (2026-10-22): PARTLY: the offline path is built (Part A/B, bulk file, results import); see (h)** |
| IMS: Invoice Management System actions and reports (new) [S] | **missing**: Inward supplies, accept/reject/pending, IMS-ITC summary, 2B recompute, remarks, auto-reconcile, ITC reduction for amendments | accountant, CA | P1 | XL | file-based Supplier Inbox briefed (comsky-connect-network.md); live sync needs a GSP |
| On-screen GST Tax Analysis in voucher entry [S] | **partial**: Rate-wise figures appear only on the printed invoice | accountant checking an invoice | P2 | S | none |
| Outward debit notes in GSTR-1 (CDNR/CDNUR ntty D) [S] | **partial**: Loop takes Sales and CreditNote only; ntty fixed to C | sellers issuing price-increase debit notes | P2 | S | confirm with the owner's CA how outward debit notes should be booked |
| HSN/SAC digit length by turnover, description on masters, code validation (new) [A] | **partial**: Only a blank-HSN check; no 4/6/8 digit rule, no HSN description field, no code check on save | any GST filer, e-invoice users | P2 | S | turnover thresholds must be researched and cited |
| Duties & taxes ledger config (calculation, rounding) [A] | **partial**: Calculation method, rounding method and limit (from memory); computation always round2 | accountant matching paise to Tally | P2 | S | none |
| Bill of Entry details on import purchases (new) [A] | **partial**: BoE number, date, port and duplicate-BoE handling missing | importers | P2 | M | none |
| e-Commerce Summary and operator supplies (GSTR-1 tables 14/15) (new) [A] | **missing**: No operator GSTIN field, summary or ECO tables | sellers on Amazon, Flipkart, Myntra | P2 | M | none |
| e-Way Bill register [A] | **missing**: No register of generated, pending, cancelled bills, no status fetch | dispatch and accounts staff | P2 | M | depends on e-way capture **Phase 29 (2026-10-22): BUILT, fed by the imported results file only; see (h)** |
| GST on advance receipts: posting and opening balances [S] | **partial**: No output-tax posting or tax rows on the receipt; no opening balances for earlier-year advances | service providers, contractors | P2 | M | not posting is a recorded design choice |
| GST classifications / rate-setup master [A] | **missing**: Named classification (HSN + dated rate), bulk GST Rate Setup report | accountant managing many items after a rate change | P2 | M | Dexie v9 table and screens |
| GST payment voucher with challan details and Challan Reconciliation (new) [A] | **missing**: CPIN/CIN/BRN capture on GST payments and a reconciliation report (TDS has both) | accountant, CA reconciling the cash ledger | P2 | M | none |
| GSTR-3B remaining sections [S] | **partial**: 5.1 interest and late fee, ISD 4(A)(4), 3.1.1 e-commerce block, Excel for the offline utility (4(D)(2) is not captured by Tally either) | accountant, CA | P2 | M | none |
| GSTR-9 export to a portal file, annual HSN summary, GSTR-9C support (new) [A] | **partial**: Screen only: no JSON, Tables 8 and 10-18 absent, no 9C | CA at year end | P2 | M | build only when a customer files GSTR-9 from the app |
| QRMP: IFF month-wise GSTR-1 JSON and PMT-06 tracking (new) [A] | **partial**: B2B and CDNR-only export for months 1 and 2; PMT-06 and fixed-sum tracking; quarter picker exists | QRMP filers | P2 | M | none |
| Reverse Charge Mechanism [A] | **partial**: Self-invoice print; liability on voucher date not time of supply; Manual posting leaves the Journal live when a purchase turns Optional | buyers from unregistered suppliers, GTA, legal services | P2 | M | ROADMAP-ADDENDUM next steps |
| Return Effective Date (move a voucher to another return period) (new) [A] | **missing**: Per-voucher return-period override, bulk update | purchase accountants claiming late ITC | P2 | M | none |
| Return filing tracker: filed/signed status, Undo Filing, accepted-as-valid (new) [A] | **missing**: Per-period filed/signed state and pending-actions report | accountant tracking many periods or clients | P2 | M | a local mark-as-filed works without portal access |
| Value-based (slab) GST rates (new) [A] | **missing**: Rate that depends on unit value band, resolved per line | apparel, footwear, textile traders | P2 | M | none |
| Composition dealers: CMP-08, GSTR-4 [A] | **missing**: Composition company still gets full GST; no tax on turnover, CMP-08, GSTR-4 | small traders, restaurants (auditor: P2, board P3) | P2 | L | none briefed |
| e-Invoice bulk generation, status and cancel [A] | **partial**: Bulk export and results import work; no API generation, status polling, bulk cancel | high-volume B2B sellers | P2 | L | GSP needed; wrapper layout unverified |
| Portal validation of GSTIN/UIN and HSN/SAC; create a party from its GSTIN (new) [A] | **partial**: Offline structure and check-character only; no registration status, legal name, address lookup | accountant creating parties | P2 | L | taxpayer search in the GSP RFQ |
| Multi-GSTIN / branch return segregation [A] | **missing**: One GSTIN per company; Tally has several registrations with own periodicity, series, e-invoice/e-way setup (workaround: one company per GSTIN) | multi-state traders and makers, CA firms | P2 | XL | none briefed |
| Interest, penalty, late-fee adjustments to 3B 5.1/6.1; ISD credit adjustment (new) [A] | **partial**: Sub-natures routed to 5.1/6.1 and an ISD route missing | accountant on late filing | P3 | S | none |
| Master versus transaction GST conflict finder (new) [A] | **partial**: List of vouchers whose line GST differs from the master rate on that date | accountant after a rate change | P3 | S | snapshot design makes it less urgent |
| ITC ineligibility at stock-item and voucher level (new) [A] | **partial**: Ledger flag only today | accountant | P3 | S | none |
| ITC reversal: ISD credit received (Tally parity) and Rule 42/43 ratio helper, s.50 interest, Rule 37A (beyond Tally) [S] | **partial**: Reversal, re-claim and Rule 37 match or beat Tally, but ISD credit (Journal Increase in ITC, 3B 4(A)(4)) is always 0 and has no statutory-adjustment nature; the 'nothing missing vs Tally' correction was refuted | branch of an ISD-distributing company; CA | P3 | S | ROADMAP-ADDENDUM: ISD only if a customer needs it |
| s.16(4) time-limit ineligible credit and 3B 4(D)(2) (new) [A] | **missing**: No date-limit check on ITC claims | CA at year end | P3 | S | none |
| Bills Payable ITC-at-risk and Bills Receivable upload status (new) [A] | **missing**: Needs portal data | accountant managing ITC | P3 | M | blocked on the GSP work |
| Quantity-based (specific) cess [S] | **partial**: Value-plus-quantity valuation types; ad valorem complete | tobacco, pan masala traders | P3 | M | cess status after Sept 2025 unconfirmed |
| GSTR-1 amendment tables (B2BA, CDNRA) (new) [A] | **missing**: No amendment tables | accountant correcting filed returns | P3 | M | Tally support unverified |
| GSTR-1 reconciliation against portal-filed data; 2A/2B reset and re-import (new) [A] | **missing**: No GSTR-1 import or comparison with the portal copy | CA | P3 | M | none |
| Form GST ITC-04 (job work) (new) [A] | **missing**: Needs the job-work chain first | makers using job workers | P3 | L | blocked by job-work chain; Tally support unverified |

#### TDS and TCS (28 rows: P0 0, P1 3, P2 10, P3 15)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| Income-tax Act 2025 natures: sections 392/393/394, payment codes, Alt+H helper [S] | **partial**: 20 natures on 1961 numbering, no payment-code field; Tally lists 41 resident + 30 non-resident TDS and 25 TCS natures (codes 1004-1092); no Nature of Remittance (Form 144); not seeded: 194B, 194BB, 194DA, 194E, 194EE, 194K, 194LA, 194N | every deductor and collector from FY 2026-27 | P1 | M | statutory values must be researched and cited (CLAUDE.md); renumbering listed as unmodelled |
| Form 26Q (now Form 140 from 1 Apr 2026) return handling [S] | **partial**: No Included/Not-relevant/Uncertain triage in the view, no save/recompute/overwrite/save-as-revised, no deductor/address-change flags, no Form 140 label | accountant, CA | P1 | L | same blocker as etds-fvu |
| e-TDS/e-TCS return file for FVU, and Forms 138/140/143/144 by period [S] | **missing**: No return file (view + annexure CSV only): NSDL fixed-width export, receipt/token no., address-change flags, Act-2025 form labelling with period rule (that part alone is S), FVU 1.2 validations; running FVU itself is external | every deductor; accountants, CAs | P1 | XL | blocked on the NSDL/Protean record-layout spec and an FVU validator (not in repo); ROADMAP-ADDENDUM 'skip' |
| Section 206AB specified-person rate and 194Q exemptions (new) [A] | **partial**: Only emulable with an Alt+E certificate row; no 206AB flag, label or return remark (Tally is manual here too) | any deductor paying non-filers; 194Q buyers | P2 | S | none |
| TDS enablement set-up (F11 with deductor screens) [S] | **partial**: Switch only; no ignore-IT-exemption-limit option, TDS on stock items switch, dated surcharge/cess slabs | accountant | P2 | S | build with tds-deductor-details |
| Deductor details (TAN, type, branch, responsible person) [S] | **partial**: Only TAN and a responsible person's name, no TAN validation; no deductor type, branch/division, TAN registration no., designation, PAN, father's name, address, contact | anyone filing returns or issuing 16A | P2 | S | none |
| TCS challan details and reconciliation [S] | **partial**: No BSR/serial/date capture on a TCS deposit, no TCS challan reconciliation, no ITNS 281 print | TCS collectors | P2 | M | none |
| TCS Nature of Goods master depth [S] | **partial**: Dated rate history, explicit no-PAN rate, payment code, accrual/realisation basis, tax-on-excess toggle; Act-2025 list (25 natures) | TCS collectors; LRS/tour-package sellers | P2 | M | none |
| TDS on fixed-asset ledgers and 'treat as TDS expense' (gross-up) (new) [A] | **missing**: Nature settable only on Expenses-nature ledgers; no tax-borne-by-deductor gross-up (s.393(10)) | accountants booking capital contractor/professional/non-resident invoices | P2 | M | none |
| TDS reversal on purchase return / debit note [S] | **partial**: No auto-computed reversal prompt; the reversal is dropped from Deductee-wise, 26Q annexure and Form 16A (they show gross) | accountant handling purchase returns | P2 | M | ROADMAP-ADDENDUM says skip; the report leak is not mentioned there |
| Automatic TCS on receipts, advances, non-item sales [S] | **partial**: Only Sales item invoices on accrual; no Receipt/advance (realisation) path, per-party basis override, stock-item TCS, accounting-invoice mode | sellers of TCS goods paid in advance | P2 | L | none |
| Form 27EQ (now Form 143) e-TCS return and Form 27D [A] | **missing**: No TCS return, collection/payment view, exceptions triage or 27D (27D only in Tally's older FAQ) | scrap, timber, liquor, mineral, vehicle sellers collecting TCS | P2 | XL | ROADMAP-ADDENDUM skip, behind etds-fvu; volume lower since 206C(1H) omitted |
| Form 27Q (now Form 144) non-resident return [A] | **missing**: No country, TIN, nature of remittance, 37BC or DTAA fields; no 27Q view or export | importers, payers of royalties/SaaS | P2 | XL | ROADMAP-ADDENDUM skip: needs FVU spec and non-resident deductee data |
| Bulk 'Update TDS Nature of Payment' exception tool (new) [A] | **missing**: No bulk assignment from the exceptions report | accountants cleaning masters before filing | P3 | S | none |
| Ledgers Without PAN report (TDS and TCS) (new) [A] | **partial**: PAN gaps show only per voucher and per buyer; no party-level or TCS report | accountants fixing higher-rate exposure | P3 | S | none |
| TCS enablement with collector details [S] | **partial**: Collector type, branch, responsible person, surcharge/cess slabs by collectee status | TCS collectors | P3 | S | none |
| TCS exceptions (uncertain transactions) report (new) [A] | **missing**: No screen for missing PAN, collectee type, nature, rate | TCS collectors | P3 | S | none |
| TDS on an advance paid through a Journal or on a third party's behalf (new) [A] | **partial**: Advance TDS fires only in a Payment voucher | group-company or agent payments | P3 | S | none |
| Modify the assessable amount for TDS on part of an expense (new) [A] | **partial**: Only the TDS amount is editable; 26Q base is the full line | accountants deducting on part of a bill | P3 | S | none |
| TDS Tax Analysis voucher-level report (Ctrl+O) (new) [A] | **partial**: Explanation only on the accept prompt | accountants auditing a deduction | P3 | S | none |
| TDS/TCS Outstandings: party-wise, due and overdue days (new) [A] | **partial**: Nature-level only; due/overdue logic lives in Challan Reconciliation, none for TCS | accountant chasing deposit deadlines | P3 | S | none |
| Print ITNS 281 challan from the tax payment voucher (new) [A] | **missing**: No challan print (TDS side unconfirmed on a TallyPrime page) | offline payers depositing at a bank | P3 | M | none |
| Rule 37BC relaxation, DTAA, non-resident deductee data (new) [A] | **missing**: No 37BC flag, DTAA provision, country, TIN; the 195 seed is a placeholder rate | payers of non-residents | P3 | M | linked to form-27q |
| TCS exemptions: government buyers, buyer deducting TDS, lower-collection certificate (new) [A] | **missing**: No exemption path; certificates hard-coded empty | TCS collectors | P3 | M | lower urgency now 206C(1H) is omitted |
| TDS on a cash-paid expense with supplier PAN and deductee type typed at the voucher (new) [A] | **missing**: No party-details capture on a cash purchase | small businesses paying in cash | P3 | M | none |
| TDS uncertain transactions (master and payment families, resolve in place) [S] | **partial**: Duty ledger without nature, payment without bank/challan details, unlinked payment; in-place resolve | accountant preparing returns | P3 | M | none |
| TDS nature on stock items (item-wise TDS in a purchase invoice) (new) [A] | **missing**: One purchase ledger drives the whole invoice | traders mixing goods (194Q) and services | P3 | M | none |
| Surcharge and cess slabs by deductee status with dates [S] | **partial**: Flat undated % per nature; no deductee-class or amount slab | payers of non-residents and high-value payments | P3 | M | none |

#### Payroll (56 rows: P0 0, P1 24, P2 26, P3 6)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| Computation on selected pay-head components [A] | **partial**: Engine sums several heads but the screen picks one and saving truncates; no 'current deductions total' base | payroll accountant (Basic+DA bases) | P1 | S | briefed ROADMAP-ADDENDUM |
| Employee master (basic profile) [S] | **partial**: Date of leaving (also needed by gratuity and full-and-final), gender, employee no., location, address, contact, e-mail, blood group | payroll accountant, HR | P1 | S | none |
| Extended statutory profile [S] | **partial**: Gender, contact, PRAN, EPS a/c, PF date of joining, ESI dispensary, tax regime with history, PAN/Aadhaar/UAN format validation | payroll accountant, CA filing PF/ESI/TDS | P1 | S | none |
| Pay head rounding method [A] | **missing**: Normal/upward/downward rounding to nearest N per head; every amount is round2 | payroll accountant (PF and ESI returns depend on it) | P1 | S | briefed ROADMAP-ADDENDUM; raised to P1 by the payroll audit |
| Payroll report filters by employee category, group, employee (new) [S] | **missing**: Pay Sheet, Payroll Statement, attendance reports loop every employee | payroll accountant with several departments | P1 | S | group filter ships before categories exist |
| Employee tax regime (old / Sec 115BAC) with revision history (new) [S] | **missing**: Regime field on the employee with effective-dated changes | payroll accountant, CA | P1 | S | belongs with salary-it |
| Arrears on retrospective salary revision (new) [S] | **missing**: Arrears calculation and accrual when a revision is back-dated | payroll accountant | P1 | M | needs salary-revisions first |
| Attendance Sheet by type [S] | **partial**: Adds present, absent, overtime and units into one figure (defect); per-type columns, day grid, leave balances, group filter | payroll accountant | P1 | M | none; correctness defect not only a gap |
| Multiple pay slips (batch print by group or category) (new) [S] | **missing**: Payslip is one employee at a time | payroll accountant, HR (monthly task) | P1 | M | none |
| Calendar and pay period configuration [A] | **partial**: Unmarked employees default to a hard-coded 26 days; no calendar/fixed/user-defined days basis | payroll accountant | P1 | M | briefed ROADMAP-ADDENDUM |
| Pay Head master [S] | **partial**: Income type, payslip name, affect-net-salary, rounding, use-for-gratuity, calculation period, Under-group choice, effective dating | payroll accountant | P1 | M | prerequisite for employer heads, gratuity, rounding |
| Payroll autofill by group or single employee [A] | **partial**: No filter by category/group/employee, no head subset, no double-processing guard; only the Salary process exists | payroll accountant | P1 | M | briefed ROADMAP-ADDENDUM |
| Payroll F11 statutory setup [A] | **partial**: Only a Maintain Payroll toggle; no PF/ESI/NPS/PT registration details, rates or ceilings in company config | payroll accountant, CA | P1 | M | briefed ROADMAP-ADDENDUM (Company.payrollStat) |
| PF other charges (admin A/c 2, EDLI A/c 21, EDLI admin) (new) [S] | **missing**: Admin and EDLI heads and the Journal autofill that books them | payroll accountant, CA | P1 | M | depends on the employer-head work |
| Salary disbursement / payment voucher [S] | **partial**: Aggregate payment only: no employee-wise bank allocation, pay one employee or group, per-employee advice, bank upload file | payroll accountant, bank | P1 | M | bank-file part linked to banking row epayments |
| Effective-dated salary revisions [A] | **missing**: One structure edited in place; an increment rewrites past periods and gratuity | payroll accountant, CA | P1 | M | briefed ROADMAP-ADDENDUM (Employee.salaryHistory) |
| Employer statutory contribution heads and processing (EPF/EPS, ESI, NPS) (new) [S] | **missing**: Heads that do not reduce net pay plus the PF/ESI/NPS contribution autofill (Dr expense, Cr payable) | payroll accountant, CA | P1 | L | overlaps payhead-statutory-types; design in ROADMAP-ADDENDUM (PayHead.role employer) |
| ESI computation and heads [S] | **partial**: Employer ESI head and posting, round-up, ceiling/applicability config, dispensary; the report is a hard-coded projection | payroll accountant (staff under Rs 21,000) | P1 | L | depends on payhead-statutory-types and rounding |
| ESI statutory reports (Monthly Statement, Forms 3/5/6) [S] | **missing**: None exist (Tally lists four) | payroll accountant, CA | P1 | L | blocked: needs the ESIC return format; no research brief |
| Income tax declarations, tax configuration, overrides (new) [S] | **missing**: Declaration capture, tax configuration screen, override of exemption and tax values | payroll accountant, CA | P1 | L | belongs with salary-it |
| Statutory, gratuity, loan, reimbursement head types [A] | **partial**: Only a label on Deductions; no employer contribution or other-charges type, loans type, reimbursement type, gratuity type, payable ledger per head | payroll accountant, CA | P1 | L | briefed ROADMAP-ADDENDUM (payhead-statutory-types) |
| Provident Fund computation and heads [S] | **partial**: Employer EPF/EPS/admin/EDLI, voluntary PF, over-58 split, configurable ceiling, per-employee applicability, rounding; the one report uses hard-coded rates from the structure, not posted payroll | payroll accountant, CA, makers with staff | P1 | XL | needs payhead-statutory-types, rounding, F11 statutory first; rates must be researched and cited |
| PF statutory reports (ECR, Forms 3A/5/6A/10/12A, challan) [S] | **missing**: None exist | payroll accountant, CA | P1 | XL | blocked: needs the EPFO ECR spec and PF-from-posted-payroll; no research brief |
| Income tax (TDS on salary) computation [A] | **missing**: Projected annual income, regime, declarations, perquisites, previous employer, standard deduction, 87A rebate with marginal relief, cess, 'As Per Income Tax Slab' head type, monthly spread | payroll accountant, CA; every employer above the basic exemption | P1 | XL | briefed ROADMAP-ADDENDUM (core/salaryTax.ts); slabs must be researched and cited per FY |
| Attendance Autofill (new) [A] | **missing**: Default-value fill for a group, mixed types per voucher | payroll accountant (month-end bulk entry) | P2 | S | none |
| Employee bank details [S] | **partial**: Branch, transaction type, and any advice or bank file that reads them | payroll accountant | P2 | S | bank file blocked on a bank format |
| Employee number / auto-numbering [A] | **missing**: Employee code in pickers, payslip, reports | payroll accountant, HR | P2 | S | briefed ROADMAP-ADDENDUM |
| Group-level salary structure (new) [A] | **missing**: Define once on an Employee Group, apply to members | payroll accountant | P2 | S | none |
| Payroll voucher manual lines (new) [A] | **partial**: Cannot add an employee or head outside the computed set | payroll accountant | P2 | S | folds into payroll-autofill |
| Pay slip content and options (new) [A] | **partial**: Attendance/production lines, YTD option, logo, DOJ, UAN, bank | payroll accountant, employees | P2 | S | none |
| Reimbursement pay heads [A] | **missing**: Head paid outside gross, excluded from statutory and tax base | payroll accountant | P2 | S | briefed ROADMAP-ADDENDUM |
| Copy salary structure from another employee [A] | **missing**: Copy-from employee (F12) | payroll accountant | P2 | S | briefed ROADMAP-ADDENDUM |
| Bonus accrual with year-end payment, statutory bonus (new) [A] | **partial**: Monthly provision, year-end payout flow, statutory bonus calculation (details unverified) | payroll accountant | P2 | M | none |
| Employee / pay head breakup reports (new) [A] | **partial**: Employee-by-head by-month breakup (also tracks advances) | payroll accountant, auditor | P2 | M | none |
| Bulk create/alter employees; import employees and salary (new) [A] | **missing**: Multi create/alter and CSV/Excel import | anyone with more than about 20 staff, switchers | P2 | M | none |
| Dated pay head rates and slabs (new) [A] | **missing**: Effective-from on slabs and rates so changes keep history | payroll accountant, CA | P2 | M | none |
| Employee categories (payroll cost allocation) [A] | **missing**: Category master, revenue/non-revenue allocation, filters, payroll expense to cost centres | maker or multi-site business costing payroll by project | P2 | M | briefed ROADMAP-ADDENDUM |
| Gratuity provision and posting [S] | **partial**: Gratuity pay head with slabs, use-for-gratuity flag (Basic+DA), monthly provision posting, date of leaving and payout | payroll accountant, CA | P2 | M | needs payhead master fields and date of leaving |
| Leave balances and loss-of-pay feeding attendance and payroll (new) [A] | **partial**: Payroll never reads leave applications, yet the HR screens say LOP is deducted in payroll (not true today) | HR, payroll accountant | P2 | M | none; the beyond row b-hr claim is not wired |
| Payroll payment advice / bank transfer letter (new) [A] | **missing**: Per-employee transfer letter from the salary payment voucher | payroll accountant, bank | P2 | M | bank layouts are partner-only in Tally too |
| Tally XML import/export of payroll masters and vouchers (new) [A] | **missing**: Employees, pay heads, structures, attendance and payroll vouchers do not come across; export writes Journals | anyone switching from Tally with payroll | P2 | M | blocked: needs a real TallyPrime payroll XML export |
| Payslip e-mail / WhatsApp, employee e-mail field and report [A] | **missing**: No employee e-mail field, no send path (print only) | HR, payroll accountant | P2 | M | ROADMAP-ADDENDUM says skip: no mail transport; folds into share-email-whatsapp |
| Previous employer income, exemption and tax paid (new) [A] | **missing**: Income, exemptions and tax already deducted by an earlier employer | CA, payroll accountant | P2 | M | belongs with salary-it |
| Professional Tax computation [S] | **partial**: PT registration no., custom/half-yearly period, rounding, dated slabs (a user-entered slab head works) | payroll accountant | P2 | M | Tally ships no state slabs either |
| Professional Tax statement and computation report [S] | **missing**: Neither report exists | payroll accountant, CA | P2 | M | none |
| Statutory payment vouchers with autofill from payable ledgers (new) [A] | **partial**: PF/ESI/PT/NPS payment autofill; no statutory payable ledgers exist | payroll accountant | P2 | M | needs the employer-head work |
| NPS (Tier I/II) with PRAN and reports (new) [A] | **missing**: NPS heads, PRAN, NPS Summary, Subscriber Contribution, PRAN Not Available | companies offering NPS | P2 | L | none |
| Salary advance / loans and recovery [A] | **missing**: Loans head posting to an asset ledger, instalment recovery, balance by employee | payroll accountant, trader with staff advances | P2 | L | briefed ROADMAP-ADDENDUM; needs ledger-group choice |
| Salary TDS operational reports (projection, variance, challan reconciliation, PAN not available) (new) [A] | **missing**: Reports around s.192 deduction | CA, payroll accountant | P2 | L | after salary-it |
| IT statutory forms (Form 16, 12BA, 24Q/Form 138) [A] | **missing**: Income Tax Computation report, Form 16, 12BA, 24Q Annexures I/II, e-24Q, ITeTDS.txt | CA, payroll accountant | P2 | XL | blocked: needs the NSDL/Protean FVU spec; Form 16 Part A comes from TRACES |
| Labour Welfare Fund deduction (new) [A] | **partial**: No state tables, no half-yearly period | payroll accountant in LWF states | P3 | S | none |
| Payroll voucher paid directly to bank or cash (new) [A] | **missing**: Always posts net to Salary Payable | small employers paying on the day | P3 | S | none |
| Payroll Register and Employee Vouchers report (new) [A] | **partial**: Day Book only today | payroll accountant, auditor | P3 | S | none |
| Payroll units of measure [A] | **missing**: Simple and compound payroll units | makers with piece-rate or shift workers | P3 | S | ROADMAP-ADDENDUM skip |
| Expat / passport-visa reports [A] | **missing**: Passport, visa, contract expiry reports | company with foreign nationals | P3 | M | ROADMAP-ADDENDUM skip (niche) |
| Payroll Statutory Summary and Gratuity Summary from posted payroll [A] | **missing**: Deducted vs employer share vs paid vs payable by head | payroll accountant, CA | P3 | M | briefed ROADMAP-ADDENDUM |

#### Banking (27 rows: P0 0, P1 6, P2 13, P3 8)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| Create vouchers from statement rows (multi, consolidated, optional-first, bill-wise) [S] | **partial**: Consolidated voucher (Alt+F7), common ledger, optional-then-regularise, bill-wise/cost/GST/TDS allocation (customer receipts land On Account) | accountant, trader | P1 | M | none |
| Cheque printing from the payment voucher [S] | **partial**: No print from voucher or after save, no batch print of selected/all pending, no blank-date print, register limited to the F2 period | accountant paying vendors by cheque | P1 | M | none; physical alignment needs a real leaf test print |
| Partial-match reconciliation and amount tolerance (new) [S] | **missing**: A line whose amount differs (bank charges, TDS short-pay) cannot match, so users create a duplicate; no tolerance or user-set window | accountant, CA | P1 | M | none |
| Payment Advice (supplier advice letter) [S] | **partial**: Bill-wise amounts, TDS and deductions, salutation/signature, printed/e-mailed tracking, e-mail send; unlabelled instruments show as NEFT | accountant, trader | P1 | M | none (no e-mail transport) |
| Auto BRS via statement import: Excel files, stored statements, bank layouts [A] | **partial**: .xlsx/.xls statements not opened; statement not stored; no exact-match reconcile on voucher save; windows not configurable; no named-bank layouts (Tally lists 258) | every accountant reconciling monthly; CA firms | P1 | L | briefed ROADMAP-ADDENDUM (xlsx reader); needs one real statement per bank |
| One-to-many and many-to-many matching (new) [S] | **missing**: One bank line vs several vouchers, or several lines vs one voucher | trader depositing cheque batches, CA | P1 | L | named in ROADMAP-ADDENDUM, not started |
| Bank ledger master (SWIFT, client code, e-pay and recon configuration) [S] | **partial**: SWIFT, client code, e-payment enable, amount-range mode, per-bank recon defaults | accountant | P2 | S | waits on epayments |
| Vendor beneficiary bank profile and e-Fund Transfer mode rules (new) [A] | **partial**: Party has only a/c no and IFSC; no bank name/branch/holder/SWIFT, per-voucher override, amount-range IMPS/NEFT rule | accountant | P2 | S | waits on epayments |
| Cheque printing with the date left blank (new) [A] | **missing**: Date always prints | accountant issuing post-dated or held cheques | P2 | S | none |
| Instrument, transaction type and bank date columns in books (new) [A] | **missing**: Ledger Vouchers, Day Book and Cash/Bank Book show none of them | accountant, bookkeeper | P2 | S | needs an F12 option on the report-config engine |
| Post-dated cheque management [S] | **partial**: Notional Bank, PDC issue date, Post-dated Summary; two registers (instrument date vs Ctrl+T flag) can disagree | trader, distributor receiving PDCs | P2 | S | none |
| Auto-reconcile on voucher save; per-bank auto-reconciliation configuration (new) [A] | **missing**: Matching runs only at import; no per-bank voucher type or series | accountant | P2 | M | depends on the persisted statement store |
| Banking Activities overview, Reconciliation Summary, Potential Matches Summary (new) [A] | **partial**: Bank picker shows closing balance only; no cross-bank book-only/bank-only counts or statement-balance difference | accountant with several bank accounts | P2 | M | none |
| Deposit slip printing [S] | **partial**: Cash slip with denominations, slip layout and print settings, printed state, instrument edit | trader with daily bank deposits | P2 | M | none |
| e-Payments readiness report (beneficiary validation, hold or exclude) (new) [A] | **missing**: No check of missing or mismatched beneficiary a/c and IFSC before payment | accountant, payroll | P2 | M | can ship before any bank file format |
| Persisted Imported Bank Data report and Reset Bank Statement Data (new) [A] | **missing**: Statement lines not saved: no imported-data report, audit trail, reset | accountant, auditor | P2 | L | needs a Dexie table, sync and backup coverage |
| Pre-configured cheque formats by bank (new) [A] | **missing**: No bank format ships (Tally lists 619); each owner calibrates by test print | accountant printing on bank leaves | P2 | L | needs a real leaf per bank |
| Connected Banking: live balance and online statement fetch (new) [A] | **missing**: No live balance or on-demand statement fetch | accountant, owner | P2 | XL | live-bank-feeds.md: bank or aggregator contract and TLS first; HANDOFF trust map (HTTP-only single VM) blocks it |
| e-Payments bulk payment file export and reverse-file import [A] | **missing**: No NEFT/RTGS/IMPS file for any bank, no reverse-file import, no salary file; Tally supports 19 banks | accountant, payroll, traders paying many vendors | P2 | XL | ROADMAP-ADDENDUM proposes a generic CSV (M) first; certified formats need each bank's spec |
| Cheque layout extras: hide company name, vertical leaf, millions (new) [A] | **missing**: No toggles | firms sharing pre-printed cheque books | P3 | S | none |
| Unlink, modify match and Set Status (new) [A] | **partial**: No match record to unlink or re-pick | accountant | P3 | S | none |
| Interest on bank and OD accounts [A] | **missing**: No daily-balance OD interest or interest voucher; rate cannot be set from the UI | OD or CC borrower | P3 | M | briefed ROADMAP-ADDENDUM (dailyBalanceInterest) |
| Multiple bank allocations per entry [S] | **partial**: Single split panel; Payment Advice collapses a split payment to one instrument | trader depositing several cheques | P3 | M | ROADMAP-ADDENDUM says skip (repeat-the-line works) |
| Bharat Connect for Business (early access) (new) [A] | **missing**: No system-to-system invoice and payment exchange | B2B businesses on NPCI BBPS | P3 | L | needs NBBL onboarding |
| e-Payment status tracking and bank download [A] | **missing**: Status states, manual set-status, statement download | accountant | P3 | L | needs a bank or aggregator contract (live-bank-feeds.md) |
| Payment gateway links (PayU, Razorpay) and settlement reconciliation (new) [A] | **missing**: No gateway profile, link generation or settlement reconciliation | business with online collections | P3 | L | needs gateway merchant accounts |
| Online payments from the app (OTP, limits, status, auto advice) (new) [A] | **missing**: No payment initiation | accountant, owner | P3 | XL | needs approvals/workflows and a bank contract |

#### Reports and MIS (31 rows: P0 0, P1 9, P2 13, P3 9)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| Ratio Analysis depth [S] | **partial**: Operating Cost %, Receivable Turnover days, ROI %, Return on Working Capital %, Working Capital Turnover, Bank OD/Sales/Purchase figures, drill, F12 | owner, CA, banker-facing user | P1 | S | Receivable Turnover drill needs payment-performance |
| Budget variance inside statements [S] | **partial**: Budget/Actual/Variance columns in Trial Balance and statements, ledger-wise drill, cost-centre variance, monthly breakup | accountant, manufacturer, finance head | P1 | M | budget-costcentres and budget-period-distribution briefed |
| Cash Flow depth [S] | **partial**: Opening/closing balance, month drill to a summary by group and voucher type, graph, scale, quarterly/weekly columns, contra exclusion | accountant, owner | P1 | M | none |
| MSME: supplier Udyam status, 45-day tracking, 43B(h) unpaid list, MSME-1 export (new) [S] | **missing**: Only the company's own Udyam is stored; no supplier MSME flag, no credit-period basis, no report, no MCA Excel layout | trader, manufacturer, CA (tax audit) | P1 | M | none; needs the current MCA template (unverified) |
| Register comparison, periodicity and F4 [S] | **partial**: Comparison/auto columns, daily/weekly periodicity, F4 voucher-type switch, extract | trader, accountant | P1 | M | none |
| Auto columns breadth [S] | **partial**: Daily/weekly/fortnightly units, total column, member-company columns, 12-column cap; only 6 reports have them | accountant, finance head | P1 | M | member-company columns depend on group-company |
| Columnar Sales / Purchase Register with Extract (new) [S] | **missing**: Ledger- or group-wise tax and discount columns per voucher, party address/GSTIN/PAN columns, group-total extract, inventory-wise extract (F8, Alt+F8) | trader, CA doing GST reconciliation | P1 | L | none |
| Sales profitability: per invoice, cost at time of sale [S] | **partial**: Per-invoice gross profit in the Sales Register, cost from each item's own valuation method (ours uses current average cost for everything) | trader, manufacturer | P1 | L | none |
| F12 breadth across reports [A] | **partial**: About 18 of 67 report files are on F12; not on Cash Flow, Funds Flow, Ratio, Monthly Summary, Cash/Bank Book, Ageing, GST, TDS, payroll statements | accountant, CA | P1 | XL | board: one useF12(title, spec) call per report |
| Exception Reports hub and Ctrl+J [S] | **partial**: Negative Ledgers, Reversing Journals, E-Mail IDs; Memorandum/Optional/Cancelled/Post-dated inside one hub; in-report Ctrl+J | accountant | P2 | S | none |
| Monthly Summary breadth [S] | **partial**: Ledger-level monthly summary in the drill chain (only the group variant is wired), F12, budget variance | accountant | P2 | S | none |
| Negative Ledgers exception [S] | **partial**: Only cash ledgers in Cr are checked; any ledger on the wrong side for its group is missing | accountant, CA | P2 | S | none |
| Step through periods and days in a report (+ and -) (new) [A] | **missing**: No + / - stepping in any report; Day Book has no day navigation | accountant paging Day Book and monthly reports | P2 | S | none |
| Balance Sheet and P&L layout and F12 options (new) [A] | **missing**: Vertical layout, loss under liabilities, Source/Application method, show with gross profit, Income & Expense A/c, Alt+F5 detailed vertical | CA, NGO and non-trading books, auditor | P2 | M | none |
| Day Book context and actions (new) [A] | **partial**: F4 voucher-type switch, Alt+F5 expanded entries, Alt+A add, Alt+I insert (Alt+X is under cancelled-register) | accountant, data-entry operator | P2 | M | none |
| Funds Flow depth [S] | **partial**: Source/application lines behind each month, current asset/liability summary, scale, auto columns, Negative Ledgers | CA, finance head | P2 | M | none |
| Ctrl+B basis of values breadth [S] | **partial**: Only Trial Balance and Group Summary; no stock-valuation basis in Balance Sheet; multi-year roll-over missing | accountant, CA | P2 | M | none |
| Manual columns breadth [A] | **partial**: No columns on Ratio, Godown Summary, Cash/Funds Flow, registers; no budget, scenario or company-vs-company column | accountant | P2 | M | none |
| Report print setup [S] | **partial**: Editable titles, page numbers, page setup, in-app preview, multi-report print | accountant, CA | P2 | M | page numbers omitted by design (Electron 31 has no @page margin boxes) |
| Dashboard parity (Tally 4.0) (new) [A] | **partial**: User-built tiles from any report, per-tile period and filter, ratio/assets/cash-flow tiles, sales and purchase dashboards, drill, print/export of tiles | owner | P2 | L | counted under 'beyond' (not parity); Dashboard v2 is in flight |
| Report Filter: conditional, multi-criteria and advanced (range and value filters) [S] | **missing**: Only a type-to-filter on rendered text exists (never changes totals): field + condition filters, AND/OR, Filter Rows vs Calculate Values, Filter Details | accountant, CA | P2 | L | Wave A adds field filters on registers and Bills (live tree, not counted) |
| Schedule III (Division I) financial statements add-on (new) [A] | **missing**: Account mapping, disclosures, Balance Sheet, P&L, indirect cash flow, ratios in Schedule III format | CAs preparing company accounts | P2 | L | none |
| Ctrl+H Change View (new) [A] | **partial**: One key to flip a report's view or periodicity | accountant | P3 | S | none |
| Outstandings scoped to a group or ledger; Show Nett Balances (new) [A] | **partial**: Group/party scope, nett balances, remembered ageing buckets | credit controller | P3 | S | none |
| Remove and restore report lines (Ctrl+R, Alt+U) (new) [A] | **missing**: Exclude chosen rows from a report's figures and restore them | accountant | P3 | S | none |
| Scale factor options [S] | **partial**: Crores and Millions units; scale on reports with no F12 | CA, larger companies | P3 | S | none |
| Save View parity (Ctrl+L) (new) [A] | **partial**: Views for reports without F12, shared across devices/users, default-opening view, filter captured | accountant, team | P3 | S | Wave A saved/shared views carry filters (live tree, not counted) |
| Statistics report (new) [A] | **missing**: Voucher counts by type and account type with drill | accountant | P3 | S | none |
| Cash Flow Projection (new) [A] | **missing**: Whole report: bank balance plus receivables and payables due by bill due date | owner, finance head | P3 | M | none |
| Multi-account and batch printing of reports (new) [A] | **missing**: Print many ledgers, books or reminders in one run | accountant, collections clerk | P3 | M | none |
| Purchase cost analysis [S] | **missing**: Purchase analysis with additional-cost loading (Tally side unconfirmed) | trader, importer | P3 | M | overlaps cost-tracking and additional-cost-purchase |

#### Platform, data exchange, printing and everyday UX (43 rows: P0 1, P1 10, P2 16, P3 16)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| Import from Tally XML (masters and vouchers) [A] | **partial**: Never run against a live TallyPrime file; not imported: orders, rejection and job-work vouchers, payroll, multi-currency, stock-journal additional cost, BOM; no Tally JSON import (7.0) | every Tally switcher (accountant, trader, CA) | P0 | XL | blocked on a live TallyPrime file; briefed ROADMAP-ADDENDUM |
| Multi-user voucher numbering consistency (client uses server numbers) (new) [A] | **partial**: /sync/next-number is deployed but the client numbers by local max+1, so offline/concurrent users can duplicate an invoice number; only warnings and Data Check | multi-device or multi-user trader (GST numbers must be unique) | P1 | S | ROADMAP-ADDENDUM lists a server-side number check; CLAUDE.md claim that the cloud assigns numbers is not true of the client **Phase 29 (2026-10-22): BUILT, backend half not deployed; see (h)** |
| Bulk print, export and e-mail of vouchers for a party and period (new) [S] | **missing**: Voucher print takes one id; selection print covers report rows only | distributors and traders printing invoices in batches | P1 | M | registers already support selection (selectionBar.tsx) **Phase 29 (2026-10-22): PARTLY: bulk print built; bulk export and e-mail not; see (h)** |
| Dispatch, consignee, order and transport details on the invoice (new) [S] | **missing**: No consignee per voucher (Ship-to prints as a copy of the buyer), delivery note no., dispatched through, destination, LR/RR, vehicle, terms of delivery/payment | traders sending goods; anyone raising e-way bills | P1 | M | needs an Alt-key panel (voucher field order is frozen); also feeds e-way Part B **Phase 29 (2026-10-22): BUILT (Alt+S panel; no order number or date field); see (h)** |
| Direct PDF file export [S] | **partial**: Only the browser print dialog's Save as PDF; no one-key PDF to file, no PDF attach to mail, no digital signature | trader, accountant, CA | P1 | M | Electron printToPDF is a small route; web needs a generator **Phase 29 (2026-10-22): PARTLY: Alt+F9 built; the desktop write was never run for real; see (h)** |
| Payment Request: UPI QR and payment link on invoices, partial payment, reconciliation (new) [S] | **missing**: No UPI ID, QR on invoice or link; a static UPI QR can reuse core/qr.ts (S), a gateway link needs keys | traders and service firms collecting by UPI | P1 | M | briefed customer-vendor-portal.md (UPI deep link, Razorpay Payment Links) **Phase 29 (2026-10-22): PARTLY: a static UPI QR only; see (h)** |
| Report export in real Excel (.xlsx), JSON, XML, HTML, image [S] | **partial**: Only CSV reachable; the .xls code is dead (no dispatcher) and is an HTML table; no .xlsx/JSON/XML/HTML/JPEG; no export options (folder, open after, header) | accountant, CA, every Tally user (daily Excel task) | P1 | M | no xlsx or pdf library installed **Phase 29 (2026-10-22): PARTLY: .xlsx built; JSON, XML, HTML and image not; see (h)** |
| Share by e-mail and WhatsApp: invoices, vouchers, reports, statements, reminders [S] | **missing**: No send path of any kind; top-bar Share (M) is a disabled stub; needs PDF output, templates and a mailer or WhatsApp link | every invoicing user; collections staff; accountant | P1 | M-L | customer-vendor-portal.md: click-to-chat/mailto first (M); WhatsApp Business needs a BSP; e-mail needs a sender account (backend has none) **Phase 29 (2026-10-22): PARTLY: the Share panel opens the person's own WhatsApp or mail app, nothing is sent; see (h)** |
| Export to Tally XML [A] | **partial**: Tags unverified against a live TallyPrime import; importing the same file twice there duplicates vouchers (no GUID); no Excel/JSON layout | CA or reseller moving data back; auditors | P1 | L | needs a live TallyPrime file |
| Import from Excel with mapping templates and samples [S] | **partial**: Native .xlsx/.xls, user-defined mapping and saved templates, sample files, multi-line/item/GST vouchers, import log, pre-import exceptions for CSV | anyone migrating from Excel or another package; reseller onboarding in week one | P1 | L | small .xlsx reader proposed in ROADMAP-ADDENDUM (brs-auto-import); Bulk Entry covers part **Phase 29 (2026-10-22): PARTLY: .xlsx with mapping and a preview built; vouchers are two-ledger rows; see (h)** |
| Multi-user: presence and who-is-editing [A] | **partial**: Live sync works (SSE); no presence list, per-person cloud rights (see ADM remote-users for invites) | any business with more than one data-entry person; CA firms | P1 | L | briefed ROADMAP-ADDENDUM (presence, invites) |
| Party's previous and current balance on the printed invoice (new) [A] | **missing**: No balance block or print part | traders with credit customers | P2 | S | none |
| Top-menu chords Alt+K/Y/Z/O/E/M/P (new) [A] | **partial**: Y, Z, O, M are disabled stubs; K, E, P work by mouse only | Tally switcher relying on muscle memory | P2 | S | browser Alt-letter conflicts; Electron unaffected; Wave B shortcut customisation may touch this (not counted) |
| Export panel and options (Alt+E), export of a voucher, invoice or dashboard (new) [A] | **partial**: No panel; Ctrl+E acts only on a table report, not a voucher, invoice or Dashboard | accountant, CA | P2 | M | depends on report-export formats |
| F12 master-creation configuration [S] | **missing**: Users cannot hide GST, TDS, contact or alias fields | accountant, shops wanting a lean ledger form | P2 | M | briefed (FieldSpec optional flag plus per-company store); field order frozen so only hide or trailing add |
| Invoice content options: export marks and containers, godown address, ledger/item additional descriptions (new) [A] | **partial**: None of these are printable parts (see dispatch and previous-balance rows) | traders, exporters | P2 | M | none |
| Invoice template gallery, watermark, header/footer images, fonts, saved templates (new) [A] | **partial**: One design with owner-selectable blocks; no template choice, watermark (except mock-IRN mark), header/footer image, fonts, colours, signature/stamp image | any business wanting a branded invoice | P2 | M | none |
| More Details (Ctrl+I): add a field to one voucher or master (new) [A] | **missing**: No per-record extra-details panel; natural home for dispatch fields | accountant adding one-off details | P2 | M | none |
| Print on pre-printed stationery or letterhead (new) [A] | **missing**: No mode that prints only variable data on a ready-made form | counter-billing trader with an invoice book | P2 | M | none |
| SmartFind: find vouchers by number, narration, amount or party; cross-company (new) [A] | **partial**: Palette finds reports, create commands, ledgers, stock items (fuzzy); not vouchers, groups, cost centres, godowns, units; one company only (auditors disagreed on P1/P2/P3) | accountant ('invoice 1234', 'the 50,000 payment') | P2 | M | none |
| Custom fields on vouchers, printed and in reports (the TDL slice) [S] | **partial**: User fields only on Ledger and StockItem; none on vouchers (vehicle no., PO date), none printed or in reports | reseller customising without TDL | P2 | M | voucher slice briefed ROADMAP-ADDENDUM; credit only the master slice (also counted under beyond b-udf) |
| User-visible cloud backup with version restore (new) [A] | **partial**: No version list or point-in-time restore for the user; operator-only nightly pg_dump with untested restore | all cloud customers; recovering from a bad import or delete | P2 | L | HANDOFF Phase B lists restore testing |
| F12 voucher-entry configuration on all vouchers [A] | **partial**: Only item invoices have F12 (3 options); none on Payment/Receipt/Contra/Journal; no skip-field or warning switches, no per-ledger narration, short names | accountant, trader, pharma/FMCG | P2 | L | briefed ROADMAP-ADDENDUM |
| Marketplace import: Amazon shipments, refunds, settlements, TDS/TCS (new) [A] | **missing**: No importer; generic importer is CSV with fixed headers | e-commerce sellers | P2 | L | depends on Excel import with mapping |
| Phone and tablet layout for reports (new) [A] | **partial**: Shell is a fixed three-column grid: at 390px the rail stays open and the report is clipped to about 160px; no read-only mobile view | owner checking cash and receivables on a phone | P2 | L | mobile apps are an unbuilt epic (docs/ROADMAP.md) |
| Record locking / simultaneous-edit prevention [S] | **partial**: Detect-and-report only; no 'being edited by X' notice; Tally's own locking unverified | multi-user accounts teams | P2 | L | needs presence work first |
| Multi-lingual UI, masters, print and export (new) [A] | **missing**: English only: no language aliases, regional-language print, per-user language, amount in words in other languages | owners and staff not working in English; regional markets | P2 | XL | none briefed |
| About screen and per-screen help article (Ctrl+F1) (new) [A] | **missing**: No About page or build info | support and resellers | P3 | S | none |
| Calculator panel (Ctrl+N) (new) [A] | **partial**: Amount fields already evaluate expressions; no panel | anyone calculating between fields | P3 | S | none |
| Delete a company (new) [A] | **missing**: Only the sample company can be deleted | anyone cleaning up trial companies | P3 | S | deliberate omission; needs a cloud-delete design |
| Delete a voucher line (Ctrl+D) (new) [A] | **partial**: Blank the field is the only way | data-entry operator | P3 | S | none |
| Export to HTML, JPEG, ASCII, JSON [A] | **missing**: No standalone HTML, text, image or JSON report export | analysts, developers | P3 | S | briefed ROADMAP-ADDENDUM (HTML and TXT slice) |
| Go To: alter a master by name, Switch To (Ctrl+G), open-reports list (new) [A] | **partial**: Ledger/item rows only open books | keyboard-first accountant | P3 | S | none |
| Notification centre (new) [A] | **partial**: Per-feature banners only (backup due, subscription); no list; no Electron auto-updater | owner wanting due-date alerts | P3 | S | none; licence/upgrade alerts do not apply to SaaS |
| Single-company data split (new) [A] | **partial**: Split always makes two companies | accountants at year end | P3 | S | none |
| Bank SWIFT code on invoices (new) [A] | **missing**: No SWIFT field | exporters | P3 | S | none |
| Display settings: Stripe View, Auto Wrap Text, number format and base-currency format (new) [A] | **missing**: Only theme and density; fmtAmt hard-codes Indian grouping, 2 decimals, (-) negatives | exporter wanting million grouping; heavy report reader | P3 | M | Wave B Settings window may cover display options (not counted) |
| F11 granular feature switches (new) [A] | **partial**: 9 switches only; batches, expiry, price levels, interest, job order always visible | small shops wanting a simpler screen | P3 | M | none |
| Print setup extras: page range, start page, custom paper, letterhead mode (new) [A] | **partial**: No report page numbers (deliberate), page range, start page; only A4/A5/Letter | accountants printing statutory books | P3 | M | Chromium 126 has no @page margin boxes |
| Selective / one-way branch sync rules and snapshot exchange (new) [A] | **missing**: Sync is whole-tenant two-way | multi-branch businesses | P3 | L | none |
| Plan caps and live billing [A] | **partial**: max_companies and max_users are display-only; payments default to MOCK, no live Razorpay keys | the vendor; customers on plan limits | P3 | L | HANDOFF Phase B |
| Several companies loaded at once (F3 keeps others open, Ctrl+F3) (new) [A] | **partial**: One active company; a second browser tab is the workaround | CA or group accountant | P3 | L | none |
| AI document capture: scanned bill to draft voucher (new) [A] | **missing**: No document extraction (bank statements only); plug-in detail unverified | accounts clerk keying purchase bills | P3 | XL | later step in attachments-drives.md; needs the attachments store first |

#### Admin, audit and security (17 rows: P0 0, P1 8, P2 6, P3 3)

| Feature | Status and what is missing | Who needs it | P | Eff | Planned / blocked by |
|---|---|---|---|---|---|
| Edit Log report: filters, line-level diffs, version drill [S] | **partial**: Voucher amounts show as '[N items]'; no filter, period, user, search, paging; Resave/Renumber not distinguished | CA, auditor | P1 | M | none |
| Edit Log that follows the company [S] | **partial**: Pulled changes not logged, operator never synced, backups carry no history, a browser-data clear loses it | CA, auditor, any company under the MCA audit-trail rule | P1 | M | none |
| Edit Log per record (Alt+Q from a voucher or master in alteration) (new) [S] | **missing**: AuditLog has no document filter and no entry from a voucher or master | accountant, CA asking who changed this invoice | P1 | M | Wave A universal '+' claims Alt+Q (chord collision, not counted) |
| Password-protected backup with recovery key (new) [S] | **missing**: Backups are plain JSON: no password, key or emailed recovery key | anyone storing or e-mailing backups off-site | P1 | M | ROADMAP-ADDENDUM: AES-GCM envelope after TLS |
| Company password: encryption of books [A] | **partial**: Passcode is real (PBKDF2) but nothing is encrypted at rest; a forgotten passcode cannot be reset | owners on shared PCs; CA offices holding client books | P1 | L | ROADMAP-ADDENDUM: TLS first (WebCrypto absent on the http-only VM), then an AES-GCM envelope |
| Edit Log Summary (voucher type, master, user) with drill and filters (new) [S] | **missing**: Altered/untouched/deleted counts by type, master and user; filters by activity, period, user; company-level activity report | CA, auditor | P1 | L | none |
| Named users with their own rights (invite and accept) [A] | **missing**: Only the owner membership row is ever inserted; no invite or accept flow in app, backend or admin; a CA needs the owner's login (board says P3) | CA/auditor working on a client's books; owners with staff | P1 | L | ROADMAP-ADDENDUM: needs e-mail delivery, per-member rights, server-enforced roles |
| Rights matrix per report and action [S] | **partial**: No per-report allow/disallow, master create vs alter vs delete, Print/Export/Preview right, per-voucher-type view restriction | accountant, CA (restrict an operator to specific reports; auditor read-only plus print) | P1 | L | none briefed; every new screen still needs a SCREEN_AREA entry |
| Back-dated entry restriction by rolling 'days allowed' per role (new) [A] | **partial**: Only fixed dates (company lock, role window), not N days back from the last entry | owners controlling data-entry staff | P2 | S | none |
| Deleted-voucher register: operator and other devices [S] | **partial**: 'Deleted By' shows the sync account not the operator; deletes made on other devices absent | accountant, GST filer | P2 | S | operator fix small and not briefed; Cancel half is under cancelled-register **Phase 29 (2026-10-22): PARTLY: Deleted By is now the operator; deletions on other devices are still absent; see (h)** |
| Altered and deleted vouchers and masters visible in Day Book, Ledger Vouchers, Chart of Accounts (new) [A] | **missing**: Basis-of-Values view with drill to history | accountant, auditor | P2 | M | depends on the company-wide audit-trail fix |
| Password policy: expiry, forced change [A] | **partial**: Strength and change checks exist; expiry with reminder, forced change on first login, per-user exemption missing | companies with an internal-controls requirement | P2 | M | ROADMAP-ADDENDUM: expiry only if a customer asks |
| Group company and consolidated reports [A] | **missing**: No group master, no consolidated Balance Sheet/P&L/Trial Balance or member-company columns | owners of several entities, CA firms | P2 | L | briefed ROADMAP-ADDENDUM (core/consolidate.ts) |
| Tally Audit listings and voucher verification (new) [A] | **missing**: Audit statistics, accept/verify entries, sampling, related-party and clarification flows | CA, internal auditor | P2 | L | none |
| Default 'Data Entry' security level (new) [A] | **partial**: Only Owner is seeded | small firms setting up a first clerk | P3 | S | none |
| Define a role by copying another (new) [A] | **missing**: RoleEditor builds from blank toggles | accountant setting up several similar roles | P3 | S | none |
| Dashboard access by role with tiles hidden (new) [A] | **missing**: Dashboard is all-or-nothing under the Reports right | owner sharing the Dashboard with staff | P3 | M | Dashboard v2 widgets carry their own right (not counted) |

(new) = not on the board before this re-check.

### b2. Not applicable to a web / local-first product (by design)

| Module | TallyPrime capability | Release | Why not applicable |
|---|---|---|---|
| ACC | Mark Changed Vouchers | 6.2 | Moves changes between separate Tally files; a shared cloud company with the Edit Log and the audit role covers the need |
| PLT | ODBC connectivity (board row odbc) | n/d | Not meaningful for a browser or Electron renderer; REST /v1, CSV/XLS export and webhooks serve the job (an OData feed is noted in ROADMAP-ADDENDUM) |
| PLT | HTTP/XML (and JSON) integration in Tally's request envelope (board row http-xml-gateway) | 7.0 (JSON) | Tally's dialect is not applicable; our REST /v1 with API keys and webhooks is the replacement and is counted under beyond row b-api (the 'partial' correction was refuted) |
| PLT | TDL language and add-on loading | n/d | Not applicable to a web product; the voucher custom-field slice is applicable and listed as voucher-custom-fields |
| PLT | Data folder / path configuration (board row data-path) | n/d | Data lives in browser IndexedDB or the Electron profile; cloud sync replaces moving data folders |
| PLT | Backup schedule extras: once/monthly, running while the app is closed | 7.0 | Needs an OS scheduler; the web build cannot run while closed |
| PLT | Digital signature (DSC dongle) on PDFs, vouchers and reports | 2.1 | A USB token needs desktop middleware; a browser cannot reach it |
| PLT | Migrate directly from native Tally.ERP 9 / TallyPrime data files | 7.0 | Tally's data files cannot be read; the Tally XML import is the bridge |
| GST | 30-day IRP/e-way login sessions, Kerala gold e-way exemption, UT-merger ITC guide | 7.1, 7.2 | Session length matters only inside a future GSP relay; the others are niche or one-time |
| MFG | Excise manufacturing voucher (CENVAT) | ERP 9 | Pre-GST, obsolete |
| TDS | Form 26QB/26QC/26QD challan-cum-statements (board row form-26qb) | n/d | No TallyPrime documentation found, so not a gap against Tally (beyond Tally); in ROADMAP-ADDENDUM as skip |
| TDS | Form 27A return cover sheet (board row form-27a) | n/d | Produced by the FVU tool from the validated file, not a Tally-generated document |
| PAY | UAE payroll (labour card, Emirates ID, UAE gratuity, WPS) | n/d | India-first product |
| PAY | Convert cost centres to employees and back | n/d | Migration helper for Tally payroll-on-cost-centre history |
| PAY | ODBC export of payroll data | n/d | No ODBC; report export writes CSV/XLS |
| BNK | Tally.NET ID, TSS and OTP-mobile gate for Connected Banking | 6.0 | Licensing mechanism; ours is operator PIN and roles. SBI's proprietary TALLYPAY layout is also not applicable |
| RPT | SDF export format | n/d | Appears only after loading an SDF TDL file |
| ADM | Tally.NET remote and browser access, Control Centre users, Connectivity Status | n/d | The product is itself a web app with cloud sync; the real equivalent is remote-users |
| ADM | Auto-login, Educational-mode lock, per-user local TDL files, WhatsApp number restrictions, user-profile reminders | n/d | Tied to Tally's serial number and TDL layer |
| ADM | Dot-matrix / neat / draft print formats, printer page range (F8-F10), Simple vs Commercial invoice, optimise-for-paper | n/d | The browser print dialog supplies printer choice and page range |
| PLT | Tally vendor services: TallyDrive, Tally on Mobile, TallyPrime Server, subscriptions view, bell, plug-in management, TallyCapital, TallyEdge, Try Now, TallyIra onboarding | 1.1.3-7.2 | Licensing and marketing tied to Tally's own services; Comsky billing and trial screens cover subscriptions |
| PLT | GCC and international VAT, Arabic bilingual invoices, UAE e-invoicing, currency symbols | 1.1.2-7.2 | India-first product |
| RPT | Reports in Browser (TRiB) | 2.0 | A web product already is one (phone layout is listed as mobile-access) |

### b3. Deliberate skips (applicable, decided against)

| Module | Capability | Release | Reason |
|---|---|---|---|
| PLT | Reuse of deleted voucher numbers | 7.1 | Deliberate: a number is never recycled (core/undo.ts voided-number registry) to keep the audit trail unambiguous; cancel-with-number (cancelled-register) is the right answer |

Also deliberate, kept inside their rows above: Insert voucher Alt+I (renumbering breaks GST serials), voucher effective dates, per-godown rates, cost tracking (briefed as Skip in ROADMAP-ADDENDUM).

## (c) Top 25 pending items for a Tally switcher

Ranked by how soon a switcher hits the gap, how much trust it costs, and how cheap the fix is. A row may name companion keys that ship together.

| # | Pending item (keys) | Why it matters | Status | P | Eff | Blocked by |
|---:|---|---|---|---|---|---|
| 1 | GST returns export formats (portal JSON, Excel/CSV) (`gst-json-export`) | A GST filer cannot trust a return file nobody has imported into the offline utility; one real GSTR-1 and 3B from the owner's CA turns the doubt into proof (SCHEMA? fields remain). | partial | P0 | L | needs a real GSTR-1 and 3B from the owner's CA (ROADMAP-ADDENDUM); online route needs a GSP |
| 2 | Import from Tally XML (masters and vouchers) (`import-xml + export-xml, bom-migration, payroll-xml`) | Migration is the adoption weapon and has never met a live TallyPrime file; it also drops BOMs, orders, job work and payroll, so a manufacturer or payroll user loses data on day one. | partial | P0 | XL | blocked on a live TallyPrime file; briefed ROADMAP-ADDENDUM |
| 3 | Cancel voucher (Alt+X) keeping its number (`cancelled-register`) | Delete leaves a hole in the GST invoice series; every Tally accountant cancels instead, and e-invoice and e-way cancellation hang off it. | partial | P1 | M | briefed ROADMAP-ADDENDUM (Voucher.cancelled flag); voucher lifecycle model pending (research README) **Phase 29 (2026-10-22): built; see (h)** |
| 4 | Report export in real Excel (.xlsx), JSON, XML, HTML, image (`report-export + export-pdf, import-excel`) | Excel and PDF in and out are daily work: only CSV reaches the user, the .xls code is never dispatched, and Excel import is CSV with fixed headers. | partial | P1 | M | no xlsx or pdf library installed **Phase 29 (2026-10-22): .xlsx and Alt+F9 built, JSON/XML/HTML/image not; see (h)** |
| 5 | Share by e-mail and WhatsApp: invoices, vouchers, reports, statements, reminders (`share-email-whatsapp + payment-request-upi, bulk-invoice-print`) | Tally 4.0's headline is invoice to WhatsApp with a UPI QR; here there is no send path, no QR and invoices print one at a time. | missing | P1 | M-L | customer-vendor-portal.md: click-to-chat/mailto first (M); WhatsApp Business needs a BSP; e-mail needs a sender account (backend has none) **Phase 29 (2026-10-22): Share panel, static UPI QR and bulk print built; see (h)** |
| 6 | Credit limit per party with enforcement (`credit-limit + interest-simple, interest-bills, price-levels`) | Engines with no input path: a user cannot set a credit limit, an interest rate or a party price level, so shipped checks and reports never fire. | partial | P1 | S-M | briefed ROADMAP-ADDENDUM (credit-limit-override) **Phase 29 (2026-10-22): the fields now exist (credit limit, interest rate and style, party price level); see (h)** |
| 7 | Single-entry layout for Payment, Receipt, Contra (TallyPrime default) (`single-entry-mode`) | Payment, Receipt and Contra are the most frequent vouchers and TallyPrime opens them as single entry; a switcher sees the difference on day one. | partial | P1 | L | briefed ROADMAP-ADDENDUM **Phase 29 (2026-10-22): built as an opt-in layout; see (h)** |
| 8 | e-Way Bill generation and management (`eway-bill + dispatch-consignee`) | Nothing is built for e-way bills, and Ship-to prints as a copy of the buyer with no vehicle, LR or transport fields; goods above the limit cannot move compliantly. | missing | P1 | XL | offline bulk-upload JSON needs no contract (L); live needs a GSP **Phase 29 (2026-10-22): the offline path and the Alt+S panel are built; see (h)** |
| 9 | Income-tax Act 2025 natures: sections 392/393/394, payment codes, Alt+H helper (`tds-natures-seed + etds-fvu, form-26q`) | FY 2026-27 is current: Income-tax Act 2025 sections, payment codes and Forms 138/140/143/144. Natures are M and unblocked; the return file waits on the FVU spec. | partial | P1 | M | statutory values must be researched and cited (CLAUDE.md); renumbering listed as unmodelled |
| 10 | Named users with their own rights (invite and accept) (`remote-users + invoice-numbering-multiuser, multiuser-lan`) | A CA or second clerk cannot be added to a company, and offline users can issue duplicate invoice numbers because the client never calls the server's number endpoint. | missing | P1 | L | ROADMAP-ADDENDUM: needs e-mail delivery, per-member rights, server-enforced roles **Phase 29 (2026-10-22): only the numbering half; invites assessed, not built; see (h)** |
| 11 | Cost-centre allocation on item invoices (`costcentre-panel`) | Cost centres do not work on Sales and Purchase item invoices, which is where trading and project firms need them. | partial | P1 | M | none |
| 12 | Group nature and Affects Gross Profit drive the P&L (`group-nature-affectsgross + group-crud-nesting`) | A user-made Income/Expense group reaches the Balance Sheet profit but not the P&L, and group alteration allows duplicates and cycles: a correctness hit in any new chart of accounts. | partial | P1 | M | briefed ROADMAP-ADDENDUM (group-advanced-flags, plSections) |
| 13 | Prefix and suffix with applicable-from dates (`numbering-prefix-suffix`) | After a prefix change, altering an old voucher silently rewrites its printed invoice number, which breaks GST serial and e-invoice document integrity. Effort S. | partial | P1 | S | none |
| 14 | Delivery/Receipt Note linked to the invoice (tracking numbers) and Bills Pending reports (`delivery-receipt-note + order-tracking`) | Delivery Note then invoice moves stock twice and double-counts order fulfilment with no warning: a data-integrity risk. | partial | P1 | L | none; data-integrity risk, not only a gap |
| 15 | Employer statutory contribution heads and processing (EPF/EPS, ESI, NPS) (`employer-contribution + payhead-rounding, pf-computation, pf-reports, esi-computation, esi-reports`) | Payroll users cannot run a compliant PF/ESI month: projections from hard-coded rates, no employer heads, no rounding, no ECR or ESI statements. | missing | P1 | L | overlaps payhead-statutory-types; design in ROADMAP-ADDENDUM (PayHead.role employer) |
| 16 | Income tax (TDS on salary) computation (`salary-it + tax-regime, it-declarations`) | TDS on salary is the other half of payroll and nothing is computed: no regime, declarations, slabs or rebate. | missing | P1 | XL | briefed ROADMAP-ADDENDUM (core/salaryTax.ts); slabs must be researched and cited per FY |
| 17 | Batches with mfg/expiry dates (`batches + expiry-reports`) | Pharma, FMCG and food users get a free-text batch box, and Batch & Expiry Summary is a ComingSoon stub still listed in the menu. | partial | P1 | L | none (board said covered) |
| 18 | BOM master on the Stock Item (Set Components) (`bom-definition + bom-scaling, mfg-stock-posting, bom-multiple`) | A BOM cannot be edited after its first use, light-ingredient ratios are rounded (5 per 1000 doubles), and FIFO items are issued at a different value than the finished good carries. | partial | P0 | M | briefed ROADMAP-ADDENDUM (Bom model); nothing built |
| 19 | MSME: supplier Udyam status, 45-day tracking, 43B(h) unpaid list, MSME-1 export (`msme-43bh`) | Section 43B(h) disallowance for MSME suppliers is live; there is no supplier MSME flag, report or MSME-1 export. | missing | P1 | M | none; needs the current MCA template (unverified) |
| 20 | GSTR-2A/2B reconciliation depth (`gstr2b-recon + ims`) | ITC claim workflow: a b2b-only value matcher that misses real portal nesting, no credit/debit notes, no IMS accept/reject. | partial | P1 | L | offline parts need no contract; download needs a GSP |
| 21 | Edit Log that follows the company (`audit-trail + audit-report, edit-log-record, edit-log-summary`) | Under the MCA audit-trail rule the Edit Log is device-local, hides voucher amounts as '[N items]', and has no per-record or summary view. | partial | P1 | M | none |
| 22 | Company password: encryption of books (`company-vault + encrypted-backup`) | Books and backups are plain JSON; the passcode only locks the UI. Blocked on TLS (WebCrypto is absent on the http-only VM). | partial | P1 | L | ROADMAP-ADDENDUM: TLS first (WebCrypto absent on the http-only VM), then an AES-GCM envelope |
| 23 | Partial-match reconciliation and amount tolerance (`partial-match + many-to-many, brs-auto-import, brs-create-voucher`) | Bank reconciliation breaks when the amount differs by bank charges or TDS, cannot match a deposit batch, and does not open .xlsx statements. | missing | P1 | M | none |
| 24 | Cheque printing from the payment voucher (`cheque-print + payment-advice, cheque-blank-date`) | Vendors paid by cheque: no print from the voucher, no batch print, no blank-date print, and the advice is a one-line listing, not a supplier letter. | partial | P1 | M | none; physical alignment needs a real leaf test print |
| 25 | SmartFind: find vouchers by number, narration, amount or party; cross-company (`smartfind + create-on-fly`) | Find 'invoice 1234' or 'the 50,000 payment' and create an item mid-invoice: the palette finds masters and reports, not vouchers, and Alt+C works on ledgers only. | partial | P2 | M | none |

## (d) Verified board corrections

An overstated board is itself a finding: 98 rows were over- or under-claimed. All were re-read by a second reader. Evidence paths are relative to the snapshot root.

### d1. Confirmed corrections

| Row id | Board | Truth | Evidence |
|---|---|---|---|
| `costcentre-panel` (accounting) | covered | **partial** | Cost panel opens only in accounting mode (VoucherScreen.tsx:1995,2012); item-invoice save builds entries with no costs (:2429-2441); imported allocations would drop on next alter in item mode (inferred, not run). |
| `group-crud-nesting` (accounting) | covered | **partial** | masters.tsx:753-757 validate checks name+parent only (ledger form has a duplicate check, group form none); Under list excludes only itself (:735-739); nature not cascaded to children on a move. Tally one-namespace and descendant-refusal rules not cited from official docs. |
| `group-nature-affectsgross` (accounting) | covered | **partial** | Group.affectsGross is read by no report or engine; P&L hard-wired to 6 reserved groups (reports.tsx:1850-1878); netProfit sums every Income/Expense group so Balance Sheet profit and P&L disagree for user-made primary groups. |
| `ledger-costcentre-flag` (accounting) | covered | **partial** | Toggle offered only for Income/Expense ledgers (masters.tsx:308,346); no cost centres on balance-sheet ledgers. |
| `line-narration` (accounting) | partial | **missing** | VoucherEntry and AccRow carry no narration; only Voucher.narration (types.ts:980). Tally per-ledger narration is a voucher-type option, not F12. |
| `master-alias` (accounting) | covered | **partial** | alias exists only on Ledger and StockItem (types.ts:248,448), single value; Group, StockGroup, Category, Godown, Unit, CostCentre, CostCategory, VoucherType have none; no language alias. |
| `multi-master-create` (accounting) | missing | **partial** | ImportData.tsx ledger and stock-item CSV create many masters additively, but no in-app Alt+H grid, no Multi Alter, no zero-all-openings; COAScreen is read-only. Borderline call. |
| `numbering-prefix-suffix` (accounting) | covered | **partial** | VoucherScreen.tsx:2394-2397 recomputes numberStr from the type's CURRENT prefix/suffix/width on every alter; no applicable-from dates, so altering an old voucher rewrites its invoice number. |
| `audit-report` (admin) | covered | **partial** | AuditLog.tsx is a flat list; arrays render '[N items]' so voucher amounts are invisible; no filters, period, user; O(n^2) prior-put scan; no Alt+Q. Tally has summary, version drill and red diffs. |
| `audit-trail` (admin) | covered | **partial** | Edit Log is a device-local outbox: operator never synced, pulled changes not logged, backups exclude history, clearing browser data loses it. |
| `deleted-tracking` (admin) | covered | **partial** | Register lists deletions only; no Alt+X cancel-retain-number; 'Deleted By' reads row.user not operator; device-local. |
| `rights-matrix` (admin) | covered | **partial** | 8 areas plus voucher type and create/alter/delete; no per-report allow, master create vs alter vs delete, Print/Export/Preview right or per-voucher-type view restriction. (Hiding GSTR-3B but keeping Balance Sheet already works: different areas.) |
| `base-currency-config` (advanced-accounting) | partial | **missing** | Company has no currency fields; fmtAmt is fixed 2 dp Indian grouping and 'Rupees/Paise' is hard-coded (money.ts:213). A fixed INR default is not partial configuration. |
| `budget-ledgers` (advanced-accounting) | covered | **partial** | BudgetLine has no basis; actualOf() always uses nett transactions over the whole period; Tally has On Closing Balance. Monthly apportionment is a separate missing row. |
| `budgets-master` (advanced-accounting) | covered | **partial** | Budget = name/from/to/lines (types.ts:610-623); no Under/sub-budget nesting; targets only group or ledger, not cost centres. |
| `cost-category-master` (advanced-accounting) | covered | **partial** | Single Name field (masters2.tsx:156-200); no alias, no allocate-revenue/non-revenue flags (XML export hard-writes Yes). Tally side from knowledge, not re-read. |
| `credit-limit` (advanced-accounting) | covered | **partial** | No ledger-master field (masters.tsx:247-248); limit arrives only via Tally XML import or sample data; check runs only for NEW item-invoice Sales/DebitNote and only warns (VoucherScreen.tsx:2326-2334, BulkEntry.tsx:581); Tally blocks on Accept. |
| `interest-bills` (advanced-accounting) | covered | **partial** | Only bills open at period end are charged (interest.tsx:40-51); late-settled bills accrue nothing; no grace period, calculate-from or slabs; unusable until a rate can be set. |
| `interest-report` (advanced-accounting) | covered | **partial** | Flat per-ledger list; no group view, no bill-wise detail (Alt+F5), no settle-bills, no pending/overdue basis (Tally uses Alt+N for comparison columns). |
| `interest-simple` (advanced-accounting) | covered | **partial** | Report reads Ledger.interestRate/interestStyle (interest.tsx:33-53) but nothing writes them (masters.tsx:247-248); non-bill-wise charge is closing balance x whole-period days, not the daily balance; no F11 switch. |
| `postdated-vouchers` (advanced-accounting) | covered | **partial** | postDated drives a badge, the register and a print note only; postsToLedgers (accounting.ts:8) ignores it so the voucher posts at its date; no include switch. Tally's exact exclusion rule (and Alt+F8) not confirmed from docs. |
| `bank-ledger-master` (banking) | covered | **partial** | Name/branch/holder/BSR/a/c/IFSC exist; missing SWIFT, client code, and e-payments, bank-reconciliation and cheque-printing configuration on the bank ledger (row's own note said e-pay pending). |
| `brs-create-voucher` (banking) | covered | **partial** | One plain two-ledger voucher per statement line, born regular; no consolidated voucher (Alt+F7), common ledger, optional-then-regularise, or bill-wise/cost/GST/TDS allocation. |
| `cheque-print` (banking) | covered | **partial** | Nothing in VoucherScreen opens the cheque screen; one cheque at a time from the register (cheque.tsx:217); no batch print, no blank-date print, register bound to the F2 period. 'Confirm each cheque' not found in Tally docs. |
| `deposit-slip` (banking) | covered | **partial** | Receipts with Cheque/DD only; no cash slip with denominations, slip layout or print settings, printed state or instrument edit. |
| `multi-bank-allocation` (banking) | missing | **partial** | Repeating the bank ledger on several rows works (BRS, PDC, deposit slip read per entry) but Payment Advice collapses to bankCr[0] with the summed amount; no single split panel. |
| `payment-advice` (banking) | covered | **partial** | One-line-per-bank-payment listing (advice.tsx); no invoice-wise bill/TDS lines, per-supplier letter, salutation/signature, printed/e-mailed flag or e-mail; payments with no instrument type are labelled NEFT. |
| `pdc-management` (banking) | covered | **partial** | Two unreconciled models (instrument date later than voucher date vs the Ctrl+T flag); no Notional Bank, PDC issue date, Post-dated Summary or include/exclude in reports. |
| `gst-advance` (gst) | covered | **partial** | Advance tax not posted to ledgers by design (gstAdvance.ts); rate typed on the 11A row not on the receipt; no opening balances for earlier-year advances; in Tally posting is optional (select tax ledgers). |
| `gst-cess` (gst) | covered | **partial** | Ad valorem only (types.ts:431 says per-unit cess is not modelled); importer skips quantity-based cess rows; Tally has value, quantity and higher-of. Niche after the Sept 2025 cess changes (unconfirmed). |
| `gst-rate-analysis` (gst) | covered | **partial** | RateBreakup[] is consumed only by the printed invoice (invoiceModel.ts buildGstTable); no on-screen GST Tax Analysis in voucher entry (Tally: Ctrl+O then Alt+F5). |
| `gst-validation` (gst) | covered | **partial** | Detection and drill complete but no resolve: no Accept-as-is, bulk correction, or B2B-to-B2C for an inactive GSTIN. The 'Accepted As Valid' report name is unconfirmed. |
| `gstr1-cdn` (gst) | covered | **partial** | buildGstr1 keeps only Sales and CreditNote (gstr1.ts:545) and ntty is fixed 'C' (:549); outward debit notes never reach CDNR/CDNUR; Tally combines credit and debit notes in 9B. |
| `gstr2b-recon` (gst) | covered | **partial** | b2b array only, GSTIN+invoice number exact, Rs 1 value tolerance, 5 statuses; no CDNR, 2A, IMS, potential matches, manual status; does not feed 3B. Parser also misses the real portal nesting data.docdata.b2b (checked: 0 groups). |
| `gstr3b` (gst) | covered | **partial** | No 5.1 interest/late fee (gstr3b.ts:24,407,525), no ISD 4(A)(4), no 3.1.1 e-commerce block, JSON only (no Excel). 4(D)(2) is NOT captured by Tally either; Tally 3.1.1 unconfirmed. |
| `batches` (inventory) | covered | **partial** | Alt+B free-text batch name + expiry only (VoucherScreen.tsx:854-884,2861); no existing-batch picker or balances, mfgDate never entered, Batch & Expiry Summary is a ComingSoon stub still in the menu, no batch opening, no batch in Manufacture. |
| `delivery-receipt-note` (inventory) | covered | **partial** | Stock-only vouchers with no tracking link to the invoice: Delivery Note then Sales moves stock twice (inventory.ts:33-48) and double-credits the order (:316); no Bills Pending reports. |
| `expiry-reports` (inventory) | covered | **missing** | No near-expiry or expired report; Stock Ageing is purchase-date buckets only; no expired-sale warning (Tally: By Expiry Date, To be Expired, expired-only). |
| `order-tracking` (inventory) | covered | **partial** | Voucher-level orderRef only (types.ts:957); picking an order does not prefill lines; matching keys on the internal voucher number; flat report with no due date, close or ageing. |
| `price-levels` (inventory) | covered | **partial** | Item x level rate grid only; party default level (Ledger.priceLevelId) has no field and Voucher.priceLevelId is never written, so a price list never reaches an invoice; no dated lists, slabs or discount. |
| `reorder-levels` (inventory) | covered | **partial** | One reorderLevel and a shortfall (reorder.tsx); no minimum order qty, order-to-be-placed, net-of-orders availability (pendingOrders unused here), consumption-based levels. |
| `stock-query-panel` (inventory) | covered | **partial** | One last purchase and one last sale only; no recent-N with voucher drill, Top Buyers/Sellers, vertical/horizontal switch, nett rate. Pending orders and price-list view not confirmed in Tally docs. |
| `valuation-per-item` (inventory) | covered | **partial** | Market method is a screen toggle (valuation.tsx:43), not an item field, and feeds no other report (marketValue.ts is comparison only). |
| `bom-definition` (manufacturing) | covered | **partial** | No BOM editor in the item master; BOM is written once by 'Save as BOM' on first manufacture (manufacturing.tsx:187-190, canSaveBom needs !hasBom) and cannot be changed; flat per-unit list; no import or export. |
| `bom-nested` (manufacturing) | missing | **partial** | A sub-assembly can be made in one journal and consumed in a second; no recursive explosion or roll-up. Tally's own help does not document recursion either, so the gap is small or nil. |
| `bom-scaling` (manufacturing) | covered | **partial** | Ratios stored and exploded with round2 (manufacturing.tsx:188,101,121) though units allow 4 decimals: 5 per 1000 stored as 0.01 so a 1000 batch pulls double; 4 per 1000 drops to 0. Arithmetic verified by running it. |
| `manufacturing-journal` (manufacturing) | covered | **partial** | Saved under the reserved StockJournal type, no own type or series (manufacturing.tsx:163-185); alteration from Day Book opens the generic Stock Journal (editId path unreachable); one output line; no batch, optional or reference. |
| `mfg-additional-cost` (manufacturing) | covered | **partial** | One typed lump (manufacturing.tsx:96-98); no expense ledgers, no % of component cost, nothing posted, retyped each batch. Absorption into finished value does work. |
| `mfg-component-godown` (manufacturing) | covered | **partial** | Manufacturing Journal hard-codes mainGodown (manufacturing.tsx:168-171); per-line godown exists only in the generic Stock Journal, which is not BOM-aware. |
| `attendance-register` (payroll) | covered | **partial** | registers.tsx:46 sums every attendance type into one 'Total Days' (mixed-unit defect); no per-type columns, day grid, group filter or hide-zero. |
| `employee-bank-details` (payroll) | covered | **partial** | Bank name, a/c and IFSC only; no branch or transaction type; nothing consumes them (disbursement shows them, no advice, no bank file, not on the payslip). |
| `employee-master` (payroll) | covered | **partial** | No leaving date, gender, employee no., location, address, contact/email, blood group (types.ts:723). A separate payslip name is not a Tally field. |
| `employee-statutory-profile` (payroll) | covered | **partial** | PAN/Aadhaar/UAN/PF/ESI/DOB exist; missing gender, contact, PRAN, EPS a/c, PF date of joining, ESI dispensary, tax regime; no format validation. Tally gender/address/emp no./PRAN confirmed, the rest unconfirmed. |
| `esi-computation` (payroll) | covered | **partial** | statutory.tsx hard-codes 21,000 / 0.75% / 3.25% as a projection; no employer ESI head or posting, no round-up, ceiling config or dispensary. |
| `esi-reports` (payroll) | covered | **missing** | No Monthly Statement - ESI, Form 3/5/6 or e-return (Tally lists four ESI reports). |
| `gratuity` (payroll) | covered | **partial** | Read-only provision report on the 'basic'-named head; no gratuity pay head, use-for-gratuity flag, journal posting or date of leaving; Tally has a Provisions-group head with slabs and a Gratuity Summary. |
| `payhead-master` (payroll) | covered | **partial** | 2 head types; no income type, payslip name, affect-net, rounding, use-for-gratuity, calculation period, Under-group choice or effective dating; ledger group fixed (masters.tsx:190). |
| `pf-computation` (payroll) | covered | **partial** | Only statutory.tsx, a report with hard-coded 12% / Rs 15,000 read from the salary structure; PayHead.statutory is a label; no employer EPF/EPS/admin/EDLI heads or posting, no rounding. |
| `pf-reports` (payroll) | covered | **missing** | No ECR, Forms 3A/5/6A/10/12A, monthly statement or PF challan anywhere in src. |
| `pt-computation` (payroll) | covered | **partial** | One generic hard-coded slab (statutory.tsx:40-44); a user-entered slab head can model a state; no PT registration no., custom/half-yearly period, rounding or dated slabs. State selection is not a confirmed Tally field. |
| `pt-reports` (payroll) | covered | **missing** | No Professional Tax Statement or Computation report (Tally lists both); PT challan in Tally unconfirmed. |
| `salary-disbursement` (payroll) | covered | **partial** | One aggregate Payment (disbursement.tsx); per-employee bank rows are display only; cannot pay one employee or a group, no employee-wise bank allocation (Tally Ctrl+F autofill), no per-employee advice. |
| `export-pdf` (platform) | covered | **partial** | Only the browser print dialog's Save as PDF; no PDF library and no printToPDF in electron/; Tally Ctrl+E writes PDF (and supports digital signatures). |
| `f12-master-config` (platform) | partial | **missing** | No F12 on any master form; FieldSpec.visible is not user-settable. |
| `import-excel` (platform) | covered | **partial** | CSV only with fixed header aliases; no .xlsx/.xls, mapping templates or sample files; vouchers are one balanced 2-ledger row each and ledgers must already exist; no preview or exceptions for CSV. |
| `record-locking` (platform) | covered | **partial** | Detect-and-report only (mismatch.ts, conflict.js), no lock or being-edited notice; client never calls /sync/next-number so duplicate numbers are possible; duplicate warning only for Manual numbering. Tally lock behaviour unverified. |
| `report-export` (platform) | covered | **partial** | Only CSV is reachable (Ctrl+E, App.tsx:98-101); exportReportXLS has a listener but nothing dispatches it and it writes an HTML table named .xls; no .xlsx, JSON, XML, HTML or JPEG; no export options. |
| `tdl-customization` (platform) | missing | **partial** | User fields exist only on Ledger and StockItem; no voucher fields (the commonest reseller job), custom reports or prints. Same capability is counted under beyond b-udf, so credit only the master-field slice. Judgement call for the owner. |
| `budget-variance-report` (reports) | covered | **partial** | Only the standalone BudgetVsActual screen; no budget column in Trial Balance/Group Summary/P&L/BS, no F5 ledger-wise, no cost-centre variance (Tally: F10 in Trial Balance). |
| `cancelled-register` (reports) | missing | **partial** | CancelledRegister lists DELETED vouchers rebuilt from the outbox; reachable only via Go To; no Alt+X cancel-retain-number. |
| `cash-flow` (reports) | covered | **partial** | Month rows of Dr/Cr on cash and bank; no opening/closing, Enter opens the Day Book not a Cash Flow Summary, no scale, columns or graph; contras counted as both inflow and outflow. |
| `exception-menu` (reports) | covered | **partial** | Hub has 6 categories; Memorandum/Optional/Post-dated registers are separate and Cancelled is Go-To only; no Negative Ledgers, Reversing Journals, E-Mail IDs; no in-report Ctrl+J. |
| `funds-flow` (reports) | covered | **partial** | Simplified monthly working-capital delta (financial.tsx:107-186); no Sources/Applications breakup, Net Profit line, scale or columns; Sources/Applications convention may be inverted. |
| `item-profitability` (reports) | covered | **partial** | COGS at current average cost, not cost at time of sale (profitability.tsx); item and party views only, no per-invoice gross profit in the Sales Register. Stock Summary already has profit options. |
| `monthly-summary` (reports) | covered | **partial** | No F12/scale/basis; the ledger-level monthly summary is unreachable (only kind 'group' is built, masters.tsx:823) and Ledger Vouchers is reached directly. |
| `movement-analysis` (reports) | covered | **partial** | Per-item table only (movement.tsx); no stock-group/category analysis, no party-wise (Ledger/Group Analysis), no effective rate with additional cost, no Stock Transfer Analysis. |
| `negative-ledgers-report` (reports) | covered | **partial** | Exceptions 'negcash' checks Cash-group ledgers in Cr only; no any-ledger-on-the-wrong-side check. |
| `purchase-profitability` (reports) | partial | **missing** | profitability.tsx handles Sales/CreditNote only; no purchase cost analysis. Overlaps cost-tracking and additional-cost-purchase; Tally side unverified. |
| `ratio-analysis` (reports) | covered | **partial** | 6 ratios only (financial.tsx:253-260); missing Operating Cost %, Recv. Turnover days, ROI %, Return on Wkg Capital %, Wkg Capital Turnover, Bank OD/Sales/Purchase groups, drill-downs, F12. Payable turnover days is not a Tally ratio. |
| `register-summary` (reports) | covered | **partial** | Month matrix and drill; useReportColumns not wired; no Alt+C/Alt+N columns, periodicity change, F4 voucher-type switch or extract. |
| `report-auto-column` (reports) | covered | **partial** | 4 units (month, quarter, half, year) capped at 12 columns, wired to 6 reports; no day/week/fortnight, total column or member-company columns. |
| `report-change-basis` (reports) | covered | **partial** | Ctrl+B wired only on Trial Balance and Group Summary (reports.tsx:976,1176); multi-year roll-over limit admitted in reportBasis.ts. |
| `report-print` (reports) | covered | **partial** | window.print() of a cloned table; no page setup, titles, in-app preview or multi-report print; page numbers omitted by design. |
| `report-range-filter` (reports) | partial | **missing** | Only a type-to-filter on rendered text (useReportFilter.ts); no numeric or condition filter. Tally's conditional Filter is Ctrl+F/F6/F7 from 3.0. Wave A adds field filters on registers and Bills (not in this snapshot). |
| `report-scale` (reports) | covered | **partial** | Units/Thousands/Lakhs only (reportConfig.ts SCALES); Cash Flow, Funds Flow, Ratio Analysis and Monthly Summary have no scale; Tally lists hundreds/lakhs/millions/thousands. |
| `stock-ageing` (reports) | covered | **partial** | Purchase-date FIFO fixed buckets only (ageing.tsx); Tally adds By Expiry/Mfg date, To-be-expired, custom slabs, expired-only, godown filter. |
| `form-26q` (tds-tcs) | covered | **partial** | On-screen deductee view + annexure CSV that says 'not the return file: no FVU'; no Included/Not-relevant/Uncertain triage, save/recompute/revised states, e-TDS text file or Form 140 label. |
| `tcs-activation` (tds-tcs) | covered | **partial** | Feature switch only; no collector type, branch or responsible person for TCS, no surcharge/cess slabs by collectee status; TAN shared with TDS. |
| `tcs-auto` (tds-tcs) | covered | **partial** | Only Sales item invoices on accrual (VoucherScreen.tsx:2168-2184); no receipt or advance (realisation) path, no accounting-invoice mode, no stock-item TCS, no per-voucher nature override. |
| `tcs-challan` (tds-tcs) | covered | **partial** | Alt+Y challan row is TDS-only (VoucherScreen.tsx:3195-3199, tdsChallan.ts:94-96); no TCS challan capture, reconciliation or ITNS 281 print. |
| `tcs-nature-master` (tds-tcs) | covered | **partial** | Name/section/rate/threshold only; no dated rate history, explicit no-PAN rate, payment code, realisation/accrual basis or tax-on-excess toggle; 5 seeds. Tally's Act-2025 count of 25 natures unverified. |
| `tds-activation` (tds-tcs) | covered | **partial** | F11 switch only; TAN is free text with no check; no deductor type, branch, TAN registration no., responsible-person details, ignore-IT-exemption switch or surcharge/cess slabs. |
| `tds-deductor-details` (tds-tcs) | covered | **partial** | Only Company.tan and deductorName; no deductor type, branch/division, TAN registration no., responsible person's designation, PAN, father's name or address; no TAN format check. |
| `tds-exceptions` (tds-tcs) | covered | **partial** | Voucher-level reasons only; no master or payment families (nature missing on duty ledger, no challan, unlinked payment), no resolve in place, no bulk update. |
| `tds-interest-latefee` (tds-tcs) | partial | **covered** | UNDERSTATED. interest201/fee234E, three views and about 25 assertions exist; Tally documents only a 'Late Fee' payment type, no computation. Remaining 'not modelled' items are beyond Tally. |
| `tds-natures-seed` (tds-tcs) | covered | **partial** | 20 seeds on 1961 numbering with no payment codes; Tally 7.1+ has 41 resident + 30 non-resident Act-2025 natures (392/393, codes 1004-1067) via Alt+H. |
| `tds-reversal` (tds-tcs) | missing | **partial** | Register nets a hand-posted Dr TDS on a Debit Note and the next plan credits it (probe), but tdsDeductionItems (tdsRegister.ts:724) drops the negative event so Deductee-wise, 26Q and Form 16A still show the gross; no auto reversal prompt. |
| `tds-surcharge-cess` (tds-tcs) | covered | **partial** | Flat undated surchargePct/cessPct per nature (tds.ts:253-255); no dated, deductee-class or income-slab model. Tally's exact company-level slab wording unverified. |

### d2. Refuted corrections (dropped; board status stands)

| Row id | Board status kept | Why the correction failed |
|---|---|---|
| `all-voucher-bases` | covered | REFUTED. Row claims the 17 bases the app has; every missing Tally type (Reversing Journal, Rejections, Material In/Out, Job Work Orders) already has its own missing row, so downgrading double counts. Fix wording only: the row says 16, code has 17. |
| `itc-reversal` | partial | REFUTED. Reversal, re-claim and Rule 37 match or beat Tally, but ISD credit (Journal Increase in ITC, 3B 4(A)(4)) is a Tally capability we lack, so 'nothing missing vs Tally' is wrong. Stays partial; add an ISD row instead. |
| `cheque-register` | covered | REFUTED. Tally documents an issued-cheque register only; a received-instrument register with deposited/cleared/returned status is not a documented Tally feature. Fix the board's Tally text, not our status. |
| `http-xml-gateway` | missing | REFUTED. Our REST /v1 and webhooks are a different dialect and are 'beyond' row b-api; there is still no Tally XML/JSON envelope endpoint or local port. Stays missing; reword 'ours'. |
| `report-value-filter` | partial | REFUTED. A free-text row filter exists (all tokens, amounts match). Fix the stale note 'None.'; the real gaps (exclude, equals, field, numeric, recomputed totals) stay under the Report Filter item. |

### d3. Board text that contradicts its own status (fix the text)

| Row(s) | Issue |
|---|---|
| all-voucher-bases | Tally text says 16 base behaviours; code and defaults seed 17. Status stays covered (refuted correction). |
| cheque-register | Board's Tally text says 'issued/received with status'; Tally documents the issued-cheque register only. Fix the Tally text, keep covered. |
| http-xml-gateway | 'ours' still says 'Sync REST push/pull only'; REST /v1, API keys and signed webhooks exist (beyond row b-api). Status stays missing. |
| report-value-filter | 'ours' says 'None.' but a free-text row filter exists. Status stays partial. |
| itc-reversal | Tally side is a manual Journal; ours matches plus Rule 37. Add the ISD credit gap; status stays partial. |
| instrument-no, instrument-print | 'ours' says set only in BRS / instrument never entered, but BankInstrPanel captures and prints them. Status covered is right. |
| credit-limit, interest-simple, interest-report, optional-vouchers | 'ours' starts 'No ...' beside a covered status; the code settles it: optional is covered, the other three are partial. |
| budgets-master, budget-ledgers, budget-groups, budget-variance-report | 'ours' still reads 'No budget ...' beside covered. |
| batches, price-levels, reorder-levels, expiry-reports | Pre-build notes ('No batch field', 'No price level/list model', 'No reorder fields', 'No batch/expiry concept') beside a covered status: first slice built, row never re-judged. |
| cancelled-register vs deleted-tracking | One says missing, the other covered, and deleted-tracking's Tally cell claims 'cancelled retain number'. Reconcile the pair. |
| gstr2b-recon, import-excel, audit-report | 'ours' says no purchase reconciliation / .xlsx not parsed / no edit-log report UI beside 'covered'. |
| payroll: calc-attendance, calc-computed, calc-userdefined, calc-slabs, attendance-types, attendance-voucher, production-pay | Status right, 'ours' text stale ('None', 'Not supported'). |
| manufacturing: 8 covered rows | 'ours' text describes the pre-build state ('No BOM', 'Generic Stock Journal src/dst only'). |
| reports: comparative-bs-pl, monthly-cashflow, negative-stock-report, confirmation-accounts, reminder-letters, budgets-master | 'ours' says 'None' or 'No ...' beside covered. |
| cloud-sync, audit-report (admin), password-policy | cloud-sync 'ours' still describes the legacy Node+SQLite server; password-policy says both 'not deployed' and 'LIVE'. |
| deposit-slip, payment-advice | Board's Tally column understates what Tally does. |
| report-email, tdl-customization, bom-nested | Stale 'ours' text; tdl-customization moved to partial only for the master-field slice (the same capability is counted under beyond b-udf). |

### d4. Defects found along the way (not parity rows)

| Area | Defect | Related row |
|---|---|---|
| accounting | P&L is hard-wired to six reserved groups, so a user-made primary Income/Expense group is on the Balance Sheet profit but not the P&L (reports.tsx:1850-1878). | `group-nature-affectsgross` |
| accounting | Altering an old Automatic voucher after a prefix, suffix or width change rewrites its printed invoice number (VoucherScreen.tsx:2394-2397). | `numbering-prefix-suffix` |
| accounting | Group alteration allows duplicate names and a parent that is the group's own descendant; a cycle detaches the subtree from the statements. | `group-crud-nesting` |
| accounting, platform | The client never calls /sync/next-number; numbers are local max+1, so two devices can issue the same invoice number. CLAUDE.md and HANDOFF say the cloud assigns numbers; the client does not use them. | `invoice-numbering-multiuser` |
| accounting | Item-invoice save rebuilds entries without costs, so cost allocations on an imported item voucher look likely to drop on the next alteration (inferred, not run). | `costcentre-panel` |
| advanced-accounting | The Interest Calculation report's empty state tells the user to set a rate in Alter Ledger, which has no such field; non-bill-wise interest is closing balance times the whole period. | `interest-simple` |
| inventory | Delivery Note then Sales invoice reduces stock twice and double-credits the order; no warning. | `delivery-receipt-note` |
| inventory | Inventory Books lists 'Batch & Expiry Summary', which opens a ComingSoon stub (the new left menu correctly omits it). | `batches` |
| inventory | Price levels can be maintained but never reach an invoice: nothing sets the party's default level. | `price-levels` |
| inventory | An alternate- or compound-unit quantity typed with a suffix is rounded to 2 decimals though units allow 4 (VoucherScreen.tsx:3052,3057). | - |
| manufacturing | BOM ratios and exploded quantities are rounded to 2 decimals: 5 per 1000 is stored 0.01 so a batch pulls double; 4 per 1000 drops to 0 (arithmetic run). | `bom-scaling` |
| manufacturing | Altering a manufacture from Day Book opens the generic Stock Journal (the Manufacture editId path is unreachable); the screen has no assertion suite or shots recipe. | `manufacturing-journal` |
| gst | The GSTR-2B parser reads b2b, data.b2b and docdata.b2b but not data.docdata.b2b, the real portal nesting (skeptic ran it: 0 groups), so a genuinely downloaded 2B JSON may import nothing. | `gstr2b-recon` |
| tds-tcs | tdsSummary counts the whole Dr on the TDS ledger as 'paid'; the challan panel tells the user to put interest and fee inside that debit, so TDS Outstanding understates tax payable by the interest/fee. | - |
| tds-tcs | A Debit Note reversal is netted by the register but dropped from the deduction list that feeds Deductee-wise, the 26Q annexure and Form 16A (they show the gross). | `tds-reversal` |
| payroll | Attendance Register adds present days, absent days, overtime hours and units into one 'Total Days' figure. | `attendance-register` |
| payroll | HR leave screens and beyond row b-hr say unpaid leave is deducted in payroll; no payroll code reads leave applications. No payroll assertion suite or sample payroll data exists; an unmarked employee silently gets 26 days. | `leave-lop` |
| banking | Payment Advice labels any payment with no instrument type 'NEFT' and collapses a split payment to its first instrument. | `payment-advice` |
| banking | Post-dated cheques are modelled twice (instrument date later than voucher date, and the Ctrl+T flag), so the two registers can disagree. | `pdc-management` |
| reports | exportReportXLS has a listener but nothing dispatches 'tw-exportxls', so the .xls path is dead code (and writes HTML, not .xlsx). | `report-export` |
| everyday | The top-bar Share item has no handler and renders disabled ('Not available in MVP'); at 390 px the left rail stays open and the Trial Balance is clipped to about 160 px. | `share-email-whatsapp` |
| admin | Edit Log is device-local; 'Deleted By' reads the sync account not the operator; AuditLog prior-version scan is O(n^2); Shut Company is reachable only from the admin-only Backup/Restore screen. | `audit-trail` |
| everyday | Wave A's universal '+' claims Alt+Q, which is TallyPrime's Edit Log chord (collision for a future per-record Edit Log). | `edit-log-record` |

## (e) TallyPrime capabilities that were never on the board

199 capabilities, grouped by the earliest Tally release the auditors found for them. A row can bundle related sub-features, and auditors' overlapping lists were merged. `ours`: missing, partial, built (the board under-claimed us), not applicable, deliberate skip. Release strings come from Tally's notes; `n/d` means the sources do not date it.

#### 1.x (Sep to Dec 2020) (15)

| Capability | Module | Release | Ours | P | Check |
|---|---|---|---|---|---|
| Create or alter any master from a picker (Alt+C, Ctrl+Enter) | ACC | 1.0 | partial | P2 | [A] |
| Reference number and date on any voucher type | ACC | 1.0 | partial | P2 | [A] |
| QRMP quarterly return period | GST | 1.1.3, 4.0 | built | - | [A] |
| QRMP: IFF month-wise GSTR-1 JSON and PMT-06 tracking | GST | 1.1.3, 4.0 | partial | P2 | [A] |
| Inventory-only company / accounts-inventory integration off | INV | 1.0 | missing | P3 | [A] |
| About screen and per-screen help article (Ctrl+F1) | PLT | 1.0 | missing | P3 | [A] |
| Bank SWIFT code on invoices | PLT | 1.1 | missing | P3 | [A] |
| Dispatch, consignee, order and transport details on the invoice | PLT | 1.0, 4.0 | missing | P1 | [S] |
| GCC and international VAT, Arabic bilingual invoices, UAE e-invoicing, currency symbols | PLT | 1.1.2-7.2 | not applicable | - | [A] |
| Go To: alter a master by name, Switch To (Ctrl+G), open-reports list | PLT | 1.0 | partial | P3 | [A] |
| More Details (Ctrl+I): add a field to one voucher or master | PLT | 1.0 | missing | P2 | [A] |
| Tally vendor services: TallyDrive, Tally on Mobile, TallyPrime Server, subscriptions view, bell, plug-in management, TallyCapital, TallyEdge, Try Now, TallyIra onboarding | PLT | 1.1.3-7.2 | not applicable | - | [A] |
| Ctrl+H Change View | RPT | 1.0 | partial | P3 | [A] |
| Ledgers Without PAN report (TDS and TCS) | TDS | 1.1.4 | partial | P3 | [A] |
| TCS exemptions: government buyers, buyer deducting TDS, lower-collection certificate | TDS | 1.1.4 | missing | P3 | [A] |

#### 2.x (Jul 2021 to Mar 2022) (9)

| Capability | Module | Release | Ours | P | Check |
|---|---|---|---|---|---|
| Edit Log per record (Alt+Q from a voucher or master in alteration) | ADM | 2.1 | missing | P1 | [S] |
| Portal validation of GSTIN/UIN and HSN/SAC; create a party from its GSTIN | GST | 2.0, 5.0, 7.1 | partial | P2 | [A] |
| Bulk print, export and e-mail of vouchers for a party and period | PLT | 2.1, 4.0 | missing | P1 | [S] |
| Digital signature (DSC dongle) on PDFs, vouchers and reports | PLT | 2.1 | not applicable | - | [A] |
| Phone and tablet layout for reports | PLT | 2.0 | partial | P2 | [A] |
| Reports in Browser (TRiB) | RPT | 2.0 | not applicable | - | [A] |
| Save View parity (Ctrl+L) | RPT | 2.0 | partial | P3 | [A] |
| Saved report views (Save View) | RPT | 2.0 | built | - | [A] |
| Step through periods and days in a report (+ and -) | RPT | 2.0 | missing | P2 | [A] |

#### 3.x (Jun to Jul 2023) (8)

| Capability | Module | Release | Ours | P | Check |
|---|---|---|---|---|---|
| Payment gateway links (PayU, Razorpay) and settlement reconciliation | BNK | 3.0 | missing | P3 | [A] |
| GSTR-1 reconciliation against portal-filed data; 2A/2B reset and re-import | GST | 3.0 | missing | P3 | [A] |
| Return Effective Date (move a voucher to another return period) | GST | 3.0, 7.1 | missing | P2 | [A] |
| Return filing tracker: filed/signed status, Undo Filing, accepted-as-valid | GST | 3.0, 5.0, 5.1 | missing | P2 | [A] |
| Value-based (slab) GST rates | GST | 3.0 | missing | P2 | [A] |
| GST on Material In / Material Out vouchers | MFG | 3.0 | missing | P2 | [A] |
| Multi-user voucher numbering consistency (client uses server numbers) | PLT | 3.0 | partial | P1 | [A] |
| Payment Request: UPI QR and payment link on invoices, partial payment, reconciliation | PLT | 3.0, 4.0 | missing | P1 | [S] |

#### 4.x (to Mar 2024) (10)

| Capability | Module | Release | Ours | P | Check |
|---|---|---|---|---|---|
| Party contact depth | ACC | 4.0 | partial | P3 | [A] |
| Dashboard access by role with tiles hidden | ADM | 4.0 | missing | P3 | [A] |
| Bill of Entry details on import purchases | GST | 4.0 | partial | P2 | [A] |
| e-Commerce Summary and operator supplies (GSTR-1 tables 14/15) | GST | 4.1 | missing | P2 | [A] |
| Job work order and challan printing | MFG | 4.0 | missing | P2 | [A] |
| Invoice content options: export marks and containers, godown address, ledger/item additional descriptions | PLT | 4.0 | partial | P2 | [A] |
| Party's previous and current balance on the printed invoice | PLT | 4.0 | missing | P2 | [A] |
| Dashboard parity (Tally 4.0) | RPT | 4.0 | partial | P2 | [A] |
| Graphical dashboards | RPT | 4.0 | built | - | [A] |
| MSME: supplier Udyam status, 45-day tracking, 43B(h) unpaid list, MSME-1 export | RPT | 4.1, 6.1 | missing | P1 | [S] |

#### 5.x (Sep to Dec 2024) (12)

| Capability | Module | Release | Ours | P | Check |
|---|---|---|---|---|---|
| Bill-wise depth: non-party ledgers, sort, pre-allocate | ACC | 5.0 | partial | P3 | [A] |
| Bills Payable ITC-at-risk and Bills Receivable upload status | GST | 5.0 | missing | P3 | [A] |
| Connected GST: upload, file, download returns from the app | GST | 5.0, 5.1, 7.1 | missing | P1 | [S] |
| GSTR-9 export to a portal file, annual HSN summary, GSTR-9C support | GST | 5.1, 7.2 | partial | P2 | [A] |
| HSN/SAC digit length by turnover, description on masters, code validation | GST | 5.0, 5.1, 7.1 | partial | P2 | [A] |
| Master versus transaction GST conflict finder | GST | 5.1 | partial | P3 | [A] |
| s.16(4) time-limit ineligible credit and 3B 4(D)(2) | GST | 5.1 | missing | P3 | [A] |
| e-Way Bill on Material In / Out | MFG | 5.1 | missing | P2 | [A] |
| Display settings: Stripe View, Auto Wrap Text, number format and base-currency format | PLT | 5.0, 7.1 | missing | P3 | [A] |
| Multi-lingual UI, masters, print and export | PLT | 5.0 | missing | P2 | [A] |
| Notification centre | PLT | 5.0 | partial | P3 | [A] |
| Bulk 'Update TDS Nature of Payment' exception tool | TDS | 5.0 | missing | P3 | [A] |

#### 6.x (Jan to Aug 2025) (16)

| Capability | Module | Release | Ours | P | Check |
|---|---|---|---|---|---|
| Mark Changed Vouchers | ACC | 6.2 | not applicable | - | [A] |
| Edit Log Summary (voucher type, master, user) with drill and filters | ADM | 6.1 | missing | P1 | [S] |
| Auto-reconcile on voucher save; per-bank auto-reconciliation configuration | BNK | 6.0 | missing | P2 | [A] |
| Banking Activities overview, Reconciliation Summary, Potential Matches Summary | BNK | 6.0, 7.1 | partial | P2 | [A] |
| Connected Banking: live balance and online statement fetch | BNK | 6.0 | missing | P2 | [A] |
| Edit Log of bank date, instrument number and instrument date changes | BNK | 6.0 | built | - | [A] |
| Instrument, transaction type and bank date columns in books | BNK | 6.0 | missing | P2 | [A] |
| One-to-many and many-to-many matching | BNK | 6.0 | missing | P1 | [S] |
| Partial-match reconciliation and amount tolerance | BNK | 6.0, 6.1 | missing | P1 | [S] |
| Persisted Imported Bank Data report and Reset Bank Statement Data | BNK | 6.0 | missing | P2 | [A] |
| Reconciliation Beginning Date and Opening BRS list | BNK | 6.0 | built | - | [A] |
| Tally.NET ID, TSS and OTP-mobile gate for Connected Banking | BNK | 6.0 | not applicable | - | [A] |
| Unlink, modify match and Set Status | BNK | 6.0 | partial | P3 | [A] |
| Vendor beneficiary bank profile and e-Fund Transfer mode rules | BNK | 6.0 | partial | P2 | [A] |
| IMS: Invoice Management System actions and reports | GST | 6.1, 7.0, 7.1 | missing | P1 | [S] |
| Single-company data split | PLT | 6.0 | partial | P3 | [A] |

#### 7.x (Dec 2025 to Oct 2026) (14)

| Capability | Module | Release | Ours | P | Check |
|---|---|---|---|---|---|
| Password-protected backup with recovery key | ADM | 7.0 | missing | P1 | [S] |
| Bharat Connect for Business (early access) | BNK | 7.0 | missing | P3 | [A] |
| Online payments from the app (OTP, limits, status, auto advice) | BNK | 7.0, 7.1 | missing | P3 | [A] |
| 30-day IRP/e-way login sessions, Kerala gold e-way exemption, UT-merger ITC guide | GST | 7.1, 7.2 | not applicable | - | [A] |
| Line discount as amount or both; discount ledger | INV | 7.1 | partial | P3 | [A] |
| AI document capture: scanned bill to draft voucher | PLT | 7.1, 7.2 | missing | P3 | [A] |
| Backup schedule extras: once/monthly, running while the app is closed | PLT | 7.0 | not applicable | - | [A] |
| Invoice template gallery, watermark, header/footer images, fonts, saved templates | PLT | 7.1, 7.2 | partial | P2 | [A] |
| Marketplace import: Amazon shipments, refunds, settlements, TDS/TCS | PLT | 7.1 | missing | P2 | [A] |
| Migrate directly from native Tally.ERP 9 / TallyPrime data files | PLT | 7.0 | not applicable | - | [A] |
| Reuse of deleted voucher numbers | PLT | 7.1 | deliberate skip | - | [A] |
| Several companies loaded at once (F3 keeps others open, Ctrl+F3) | PLT | 7.0 | partial | P3 | [A] |
| SmartFind: find vouchers by number, narration, amount or party; cross-company | PLT | 7.0 | partial | P2 | [A] |
| Schedule III (Division I) financial statements add-on | RPT | 7.1 | missing | P2 | [A] |

#### Undated in the sources (long-standing or Tally.ERP 9 era) (115)

| Capability | Module | Release | Ours | P | Check |
|---|---|---|---|---|---|
| Change voucher type on alteration; convert a Memorandum to a real voucher | ACC | n/d | missing | P2 | [A] |
| Chart of Accounts as the masters workbench | ACC | n/d | partial | P3 | [A] |
| Related Reports (Ctrl+O) from a voucher, ledger or report | ACC | n/d | missing | P3 | [A] |
| Unused-masters list with bulk delete; bulk party GSTIN and e-mail update | ACC | n/d | missing | P3 | [A] |
| Altered and deleted vouchers and masters visible in Day Book, Ledger Vouchers, Chart of Accounts | ADM | n/d | missing | P2 | [A] |
| Auto-login, Educational-mode lock, per-user local TDL files, WhatsApp number restrictions, user-profile reminders | ADM | n/d | not applicable | - | [A] |
| Back-dated entry restriction by rolling 'days allowed' per role | ADM | n/d | partial | P2 | [A] |
| Default 'Data Entry' security level | ADM | n/d | partial | P3 | [A] |
| Define a role by copying another | ADM | n/d | missing | P3 | [A] |
| Dot-matrix / neat / draft print formats, printer page range (F8-F10), Simple vs Commercial invoice, optimise-for-paper | ADM | n/d | not applicable | - | [A] |
| Edit Log viewing restricted by rights | ADM | n/d | built | - | [A] |
| Inactive users, admin PIN reset, self-service PIN change | ADM | n/d | built | - | [A] |
| Switch user without closing the company | ADM | n/d | built | - | [A] |
| Tally Audit listings and voucher verification | ADM | n/d | missing | P2 | [A] |
| Tally.NET remote and browser access, Control Centre users, Connectivity Status | ADM | n/d | not applicable | - | [A] |
| Cost Centre Monthly Summary and column comparison | ADV | n/d | missing | P2 | [A] |
| Cost centre master extras | ADV | n/d | partial | P3 | [A] |
| Credit Limits bulk screen (Alter > Credit Limits) | ADV | n/d | missing | P2 | [A] |
| Credit period as a date, counted from voucher, invoice or effective date | ADV | n/d | partial | P3 | [A] |
| Invoices printed in a foreign currency | ADV | n/d | missing | P2 | [A] |
| Ledger Vouchers with cost-centre details (F12) | ADV | n/d | missing | P3 | [A] |
| Ledgers in a foreign currency | ADV | n/d | missing | P2 | [A] |
| Overdue-party warning at entry; credit period in days, weeks, months or years | ADV | n/d | missing | P2 | [A] |
| View any report in another currency | ADV | n/d | missing | P3 | [A] |
| Bank charges and interest booked from statement lines | BNK | n/d | built | - | [A] |
| Cheque layout extras: hide company name, vertical leaf, millions | BNK | n/d | missing | P3 | [A] |
| Cheque printing with the date left blank | BNK | n/d | missing | P2 | [A] |
| Pre-configured cheque formats by bank | BNK | n/d | missing | P2 | [A] |
| e-Payments readiness report (beneficiary validation, hold or exclude) | BNK | n/d | missing | P2 | [A] |
| Form GST ITC-04 (job work) | GST | n/d (unverified) | missing | P3 | [A] |
| GST payment voucher with challan details and Challan Reconciliation | GST | n/d | missing | P2 | [A] |
| GSTR-1 amendment tables (B2BA, CDNRA) | GST | ERP 9 doc; TallyPrime unverified | missing | P3 | [A] |
| ITC ineligibility at stock-item and voucher level | GST | n/d | partial | P3 | [A] |
| Interest, penalty, late-fee adjustments to 3B 5.1/6.1; ISD credit adjustment | GST | n/d | partial | P3 | [A] |
| Bulk Revise Price (Alt+R), price-list export/import | INV | n/d | missing | P3 | [A] |
| Godown attributes: storage flag, address on vouchers, third-party and consignment godowns | INV | n/d | missing | P3 | [A] |
| Inventory voucher registers (Stock Journal, Physical Stock, Delivery/Receipt Note, Rejections, Order Books) | INV | n/d | partial | P2 | [A] |
| Order due dates, lot splits and pre-close | INV | n/d | missing | P2 | [A] |
| Order outstanding views, ageing and Order Books | INV | n/d | partial | P2 | [A] |
| Stock Journal additional cost on transfer, and wastage | INV | n/d | missing | P2 | [A] |
| Stock Summary batch, alternate-unit and order-position columns | INV | n/d | partial | P2 | [A] |
| Stock group option: should quantities be added | INV | n/d | missing | P3 | [A] |
| Stock item / group monthly summary | INV | n/d | missing | P2 | [A] |
| Value-only stock items (no unit) | INV | n/d | missing | P3 | [A] |
| BOM carried in Tally migration (import and export) | MFG | n/d | missing | P1 | [S] |
| Batch, mfg date and expiry on Manufacturing Journal lines | MFG | n/d | missing | P1 | [S] |
| Copy BOM from another BOM or item | MFG | n/d | missing | P3 | [A] |
| Cost Estimation (Item Estimates) report | MFG | n/d | missing | P3 | [A] |
| Excise manufacturing voucher (CENVAT) | MFG | ERP 9 | not applicable | - | [A] |
| Job work setup: Enable Job Order Processing, Material In/Out voucher-type options | MFG | n/d | missing | P2 | [A] |
| Manufacturing Journal F12 options and voucher flags | MFG | n/d | missing | P3 | [A] |
| Material In auto-consumption of raw material | MFG | n/d | missing | P2 | [A] |
| Pre-close job work orders | MFG | n/d | missing | P2 | [A] |
| Purchase requisition for manufacturing | MFG | n/d | missing | P3 | [A] |
| Source godown stored on each BOM component | MFG | n/d | missing | P3 | [A] |
| Unit of manufacture on a BOM | MFG | n/d | partial | P3 | [A] |
| Arrears on retrospective salary revision | PAY | n/d | missing | P1 | [S] |
| Attendance Autofill | PAY | n/d | missing | P2 | [A] |
| Bonus accrual with year-end payment, statutory bonus | PAY | n/d | partial | P2 | [A] |
| Bulk create/alter employees; import employees and salary | PAY | n/d | missing | P2 | [A] |
| Convert cost centres to employees and back | PAY | n/d | not applicable | - | [A] |
| Dated pay head rates and slabs | PAY | n/d | missing | P2 | [A] |
| Employee / pay head breakup reports | PAY | n/d | partial | P2 | [A] |
| Employee tax regime (old / Sec 115BAC) with revision history | PAY | n/d | missing | P1 | [S] |
| Employer statutory contribution heads and processing (EPF/EPS, ESI, NPS) | PAY | n/d | missing | P1 | [S] |
| Group-level salary structure | PAY | n/d | missing | P2 | [A] |
| Income tax declarations, tax configuration, overrides | PAY | n/d | missing | P1 | [S] |
| Labour Welfare Fund deduction | PAY | n/d | partial | P3 | [A] |
| Leave balances and loss-of-pay feeding attendance and payroll | PAY | n/d | partial | P2 | [A] |
| Multiple pay slips (batch print by group or category) | PAY | n/d | missing | P1 | [S] |
| NPS (Tier I/II) with PRAN and reports | PAY | n/d | missing | P2 | [A] |
| ODBC export of payroll data | PAY | n/d | not applicable | - | [A] |
| PF other charges (admin A/c 2, EDLI A/c 21, EDLI admin) | PAY | n/d | missing | P1 | [S] |
| Pay slip content and options | PAY | n/d | partial | P2 | [A] |
| Payroll Register and Employee Vouchers report | PAY | n/d | partial | P3 | [A] |
| Payroll payment advice / bank transfer letter | PAY | n/d | missing | P2 | [A] |
| Payroll report filters by employee category, group, employee | PAY | n/d | missing | P1 | [S] |
| Payroll voucher manual lines | PAY | n/d | partial | P2 | [A] |
| Payroll voucher paid directly to bank or cash | PAY | n/d | missing | P3 | [A] |
| Previous employer income, exemption and tax paid | PAY | n/d | missing | P2 | [A] |
| Salary TDS operational reports (projection, variance, challan reconciliation, PAN not available) | PAY | n/d | missing | P2 | [A] |
| Statutory payment vouchers with autofill from payable ledgers | PAY | n/d | partial | P2 | [A] |
| Tally XML import/export of payroll masters and vouchers | PAY | n/d | missing | P2 | [A] |
| UAE payroll (labour card, Emirates ID, UAE gratuity, WPS) | PAY | n/d | not applicable | - | [A] |
| Calculator panel (Ctrl+N) | PLT | n/d | partial | P3 | [A] |
| Delete a company | PLT | n/d | missing | P3 | [A] |
| Delete a voucher line (Ctrl+D) | PLT | n/d | partial | P3 | [A] |
| Export panel and options (Alt+E), export of a voucher, invoice or dashboard | PLT | n/d | partial | P2 | [A] |
| F11 granular feature switches | PLT | n/d | partial | P3 | [A] |
| Print on pre-printed stationery or letterhead | PLT | n/d | missing | P2 | [A] |
| Print setup extras: page range, start page, custom paper, letterhead mode | PLT | n/d | partial | P3 | [A] |
| Selective / one-way branch sync rules and snapshot exchange | PLT | n/d | missing | P3 | [A] |
| TDL language and add-on loading | PLT | n/d | not applicable | - | [A] |
| Top-menu chords Alt+K/Y/Z/O/E/M/P | PLT | n/d | partial | P2 | [A] |
| User-visible cloud backup with version restore | PLT | TallyDrive | partial | P2 | [A] |
| Balance Sheet and P&L layout and F12 options | RPT | n/d | missing | P2 | [A] |
| Cash Flow Projection | RPT | n/d | missing | P3 | [A] |
| Columnar Sales / Purchase Register with Extract | RPT | n/d | missing | P1 | [S] |
| Day Book context and actions | RPT | n/d | partial | P2 | [A] |
| Multi-account and batch printing of reports | RPT | n/d | missing | P3 | [A] |
| Outstandings scoped to a group or ledger; Show Nett Balances | RPT | n/d | partial | P3 | [A] |
| Remove and restore report lines (Ctrl+R, Alt+U) | RPT | n/d | missing | P3 | [A] |
| SDF export format | RPT | n/d | not applicable | - | [A] |
| Statistics report | RPT | n/d | missing | P3 | [A] |
| Modify the assessable amount for TDS on part of an expense | TDS | n/d | partial | P3 | [A] |
| Print ITNS 281 challan from the tax payment voucher | TDS | n/d | missing | P3 | [A] |
| Rule 37BC relaxation, DTAA, non-resident deductee data | TDS | n/d | missing | P3 | [A] |
| Section 206AB specified-person rate and 194Q exemptions | TDS | n/d | partial | P2 | [A] |
| TCS exceptions (uncertain transactions) report | TDS | n/d | missing | P3 | [A] |
| TDS Tax Analysis voucher-level report (Ctrl+O) | TDS | n/d | partial | P3 | [A] |
| TDS nature on stock items (item-wise TDS in a purchase invoice) | TDS | n/d | missing | P3 | [A] |
| TDS on a cash-paid expense with supplier PAN and deductee type typed at the voucher | TDS | n/d | missing | P3 | [A] |
| TDS on an advance paid through a Journal or on a third party's behalf | TDS | n/d | partial | P3 | [A] |
| TDS on fixed-asset ledgers and 'treat as TDS expense' (gross-up) | TDS | n/d | missing | P2 | [A] |
| TDS/TCS Outstandings: party-wise, due and overdue days | TDS | n/d | partial | P3 | [A] |

## (f) In-flight work and the research docs

The audit counted the committed snapshot only. The three in-flight efforts and what they touch:

| Work | What it is | Parity rows it touches | Still open after it |
|---|---|---|---|
| Wave A: Zoho-style UX (c0d9aa3, committed and deployed) | left menu v2 with +, + New, universal + (Alt+Q), filters and saved views, Reports Center, nav-stack fix. NOT in the audited snapshot. | report-filter (field filters on Day Book, registers, Ledger Vouchers, Receivables/Payables: could lift the range-filter row from missing to partial once verified), save-view-parity (saved and shared views carry filters), create-on-fly (menu-level only, does not close the in-voucher gap). Reports Center and '+' are beyond Tally. | Still open: numeric/OR/advanced filter modes, Calculate-Values (totals recompute), in-voucher Alt+C beyond ledgers, Ctrl+Enter alter. Alt+Q collides with Tally's per-record Edit Log chord. |
| Wave B: Settings window and configurable shortcuts (uncommitted; spec docs/design/settings-spec.md) | Appearance (theme, accent, font, text size, motion, contrast), Shortcuts (rebind the F-key rail family, hide or reorder rail and top-bar buttons), Workspace (administrator defaults and locks). | topmenu-chords (customisable rail and top bar, but Alt+Y/Z/O/M still have no action), display-settings (appearance only: number format and Stripe View not listed in the spec). Beyond Tally. | Still open: Stripe View, number/base-currency format, F11 breadth, F12 on masters and accounting vouchers. |
| Dashboard v2 (uncommitted; docs/design/dashboard-v2-contracts.md calls it wave A2) | twelve-widget default board, per-widget rights, period and bucket rules, bespoke overdue list; plFigures.ts already committed in c0d9aa3. | dashboard-parity (user-built tiles from any report and per-tile period are NOT in scope), dashboard-tile-rights (widgets carry their own right; Data Entry still refused). Counted under beyond, never in parity. | Still open: tiles built from any report, per-tile filters, separate Sales and Purchase dashboards, print/export of tiles. |

Naming: the repo calls the Settings work "Wave B" (settings-spec.md) and the Dashboard work "wave A2" (dashboard-v2-contracts.md); the "wave D" label in the request is not in the files read, so the mapping of letters to efforts is the brief's, not the repo's.

None of this moves the corrected percentage until it is committed and audited. At most two or three report-filter and saved-view rows could move from missing or partial to partial or covered.

**Research docs (docs/research/README.md classification):** approvals, attachments, the party portal and Comsky Connect are Beyond Tally and belong under `beyond`, never in the parity percentage. e-Invoicing and statement import map to existing parity rows.

| Topic | Doc | Classification | Link to the pending list |
|---|---|---|---|
| Approvals and workflows | `docs/research/approvals-workflows.md` | Beyond Tally | Maker-checker on the optional bit; not a parity row. Do not fold into the percentage. |
| Attachments and drives | `docs/research/attachments-drives.md` | Beyond Tally | Local-first files; OCR-to-draft-voucher is a later step and is the nearest thing to Tally's Docs by Ira (row docs-by-ira, P3). |
| Customer and vendor portal | `docs/research/customer-vendor-portal.md` | Beyond Tally | Read-only party portal. Its UPI deep link, Razorpay Payment Links and WhatsApp click-to-chat are the briefed route for rows payment-request-upi and share-email-whatsapp. |
| Comsky Connect network | `docs/research/comsky-connect-network.md` | Beyond Tally | Supplier Inbox from files is the briefed offline route for IMS and 2B reconciliation depth (rows ims, gstr2b-recon). |
| Live bank feeds | `docs/research/live-bank-feeds.md` | Beyond in framing; parity in substance | README maps statement import to parity row brs-auto-import. Live feeds equal Tally's Connected Banking (6.0), row connected-banking (P2, XL), blocked on a bank or aggregator contract and TLS. |
| e-Invoice GSP relay | `docs/research/einvoice-gsp.md` | Parity (rows einvoice-irn, einvoice-bulk, eway-bill live, connected-gst) | README maps e-invoicing to existing parity rows. The same relay would carry e-way by IRN, 2B, IMS and taxpayer search; needs a GSP RFQ, TLS and a credential vault. The research roadmap's own Wave 0 fixes the mock-IRN print hazard. |

Honest reading of the request's grouping: bank feeds and the e-invoice GSP relay are briefed as cloud-track work, but each delivers a capability TallyPrime already has (Connected Banking, e-invoice IRN, e-way by IRN, Connected GST, IMS). Their parity slice is real, counted above, and blocked on the same external prerequisites: a GSP or bank contract, TLS, a credential vault. Approvals, attachments, the portal and Connect have no Tally counterpart and are not parity.

## (g) Recommended next three build waves

Start the owner-held unblockers now; they gate the biggest P0 and P1 rows and cost the owner little:

| Unblocker the owner holds | Unblocks |
|---|---|
| A real GSTR-1 and GSTR-3B from the owner's CA, imported into the GST offline utility | gst-json-export (P0) |
| A real TallyPrime XML export (masters, vouchers, a BOM, payroll) | import-xml (P0), export-xml, bom-migration, payroll-xml |
| NSDL/Protean e-TDS record layout and an FVU validator | etds-fvu, form-26q, form-27q, form-27eq, salary-it-forms |
| EPFO ECR and ESIC return formats | pf-reports, esi-reports |
| A GSP/IRP RFQ decision, TLS hostname plan, credential vault | einvoice-irn, connected-gst, ims (live), eway-bill (live) |
| A bank or aggregator contract, one real statement per bank | connected-banking, epayments, brs-auto-import |
| A decision on the voucher lifecycle and numbering model (cancelled, burned numbers) | cancelled-register, numbering rows, approvals and e-invoicing in the research docs |
| A mail transport and a WhatsApp Business provider (or click-to-chat only) | share-email-whatsapp, payslip-email |

### Wave 1: make what shipped reachable and correct

Internal build, mostly S and M, no external dependency. Closes mostly rows where a user cannot reach or trust a feature the board called covered. 23 items (11 S, 1 S-M, 9 M, 2 L; effort is the whole row's, a slice is smaller).

| Item | Feature | Slice or note | P | Eff | Depends on |
|---|---|---|---|---|---|
| `credit-limit` | Credit limit per party with enforcement | ledger-master field, then block on Accept | P1 | S-M | briefed ROADMAP-ADDENDUM (credit-limit-override) **Phase 29 (2026-10-22): PARTLY built; see (h)** |
| `interest-simple` | Interest parameters on the ledger master and Enable Interest Calculation | rate/style fields on the ledger, F11 switch, then the daily-balance engine | P1 | M | briefed ROADMAP-ADDENDUM (interest-advanced, bank-interest) **Phase 29 (2026-10-22): PARTLY built (no F11 switch); see (h)** |
| `price-levels` | Price levels and price lists | slice: party default level field and the voucher pick | P1 | L | price-slab part briefed; the unreachable party field is not briefed **Phase 29 (2026-10-22): PARTLY built; see (h)** |
| `group-crud-nesting` | Group master integrity | - | P1 | S | none |
| `group-nature-affectsgross` | Group nature and Affects Gross Profit drive the P&L | - | P1 | M | briefed ROADMAP-ADDENDUM (group-advanced-flags, plSections) |
| `numbering-prefix-suffix` | Prefix and suffix with applicable-from dates | keep numberStr unless number or date changed | P1 | S | none |
| `invoice-numbering-multiuser` | Multi-user voucher numbering consistency (client uses server numbers) | client calls /sync/next-number | P1 | S | ROADMAP-ADDENDUM lists a server-side number check; CLAUDE.md claim that the cloud assigns numbers is not true of the client **Phase 29 (2026-10-22): built, backend half not deployed; see (h)** |
| `cancelled-register` | Cancel voucher (Alt+X) keeping its number | Voucher.cancelled and Alt+X | P1 | M | briefed ROADMAP-ADDENDUM (Voucher.cancelled flag); voucher lifecycle model pending (research README) **Phase 29 (2026-10-22): built; see (h)** |
| `report-export` | Report export in real Excel (.xlsx), JSON, XML, HTML, image | dispatch the xlsx path, write a real .xlsx | P1 | M | no xlsx or pdf library installed **Phase 29 (2026-10-22): PARTLY built (.xlsx); see (h)** |
| `export-pdf` | Direct PDF file export | printToPDF on Electron, a generator on web | P1 | M | Electron printToPDF is a small route; web needs a generator **Phase 29 (2026-10-22): PARTLY built (Alt+F9); see (h)** |
| `bom-scaling` | BOM quantity precision | - | P1 | S | found by this audit; not briefed |
| `mfg-component-godown` | Component and product godown in the Manufacturing Journal | - | P1 | S | briefed with mfg-stock-posting |
| `attendance-register` | Attendance Sheet by type | per-type columns | P1 | M | none; correctness defect not only a gap |
| `tds-reversal` | TDS reversal on purchase return / debit note | slice: stop dropping reversals from Deductee-wise, 26Q and 16A | P2 | M | ROADMAP-ADDENDUM says skip; the report leak is not mentioned there |
| `audit-trail` | Edit Log that follows the company | sync the operator, log pulled changes, include history in backups | P1 | M | none |
| `audit-report` | Edit Log report: filters, line-level diffs, version drill | render voucher lines, add filters | P1 | M | none |
| `batches` | Batches with mfg/expiry dates | slice: replace the ComingSoon Batch Summary stub with a real one | P1 | L | none (board said covered) |
| `payhead-rounding` | Pay head rounding method | - | P1 | S | briefed ROADMAP-ADDENDUM; raised to P1 by the payroll audit |
| `calc-selected-components` | Computation on selected pay-head components | - | P1 | S | briefed ROADMAP-ADDENDUM |
| `employee-master` | Employee master (basic profile) | - | P1 | S | none |
| `employee-statutory-profile` | Extended statutory profile | - | P1 | S | none |
| `inventory-registers` | Inventory voucher registers (Stock Journal, Physical Stock, Delivery/Receipt Note, Rejections, Order Books) | menu entries for registers RegisterReport already renders | P2 | S | none |
| `negative-ledgers-report` | Negative Ledgers exception | - | P2 | S | none |

### Wave 2: the switcher's day-one workflow

Internal build, M and L. What a Tally user does in the first week and notices missing. 32 items (2 S, 22 M, 1 M-L, 7 L; effort is the whole row's, a slice is smaller).

| Item | Feature | Slice or note | P | Eff | Depends on |
|---|---|---|---|---|---|
| `single-entry-mode` | Single-entry layout for Payment, Receipt, Contra (TallyPrime default) | - | P1 | L | briefed ROADMAP-ADDENDUM **Phase 29 (2026-10-22): built as an opt-in layout; see (h)** |
| `costcentre-panel` | Cost-centre allocation on item invoices | - | P1 | M | none |
| `import-excel` | Import from Excel with mapping templates and samples | .xlsx reader, mapping templates | P1 | L | small .xlsx reader proposed in ROADMAP-ADDENDUM (brs-auto-import); Bulk Entry covers part **Phase 29 (2026-10-22): PARTLY built; see (h)** |
| `share-email-whatsapp` | Share by e-mail and WhatsApp: invoices, vouchers, reports, statements, reminders | slice: PDF + wa.me / mailto first | P1 | M-L | customer-vendor-portal.md: click-to-chat/mailto first (M); WhatsApp Business needs a BSP; e-mail needs a sender account (backend has none) **Phase 29 (2026-10-22): PARTLY built (Share panel); see (h)** |
| `payment-request-upi` | Payment Request: UPI QR and payment link on invoices, partial payment, reconciliation | slice: static UPI QR | P1 | M | briefed customer-vendor-portal.md (UPI deep link, Razorpay Payment Links) **Phase 29 (2026-10-22): PARTLY built (static QR); see (h)** |
| `bulk-invoice-print` | Bulk print, export and e-mail of vouchers for a party and period | - | P1 | M | registers already support selection (selectionBar.tsx) **Phase 29 (2026-10-22): PARTLY built (bulk print); see (h)** |
| `dispatch-consignee` | Dispatch, consignee, order and transport details on the invoice | - | P1 | M | needs an Alt-key panel (voucher field order is frozen); also feeds e-way Part B **Phase 29 (2026-10-22): built; see (h)** |
| `more-details` | More Details (Ctrl+I): add a field to one voucher or master | - | P2 | M | none |
| `smartfind` | SmartFind: find vouchers by number, narration, amount or party; cross-company | - | P2 | M | none |
| `create-on-fly` | Create or alter any master from a picker (Alt+C, Ctrl+Enter) | - | P2 | M | Wave A '+' quick-create is menu-level, not in-voucher (not counted) |
| `msme-43bh` | MSME: supplier Udyam status, 45-day tracking, 43B(h) unpaid list, MSME-1 export | - | P1 | M | none; needs the current MCA template (unverified) |
| `columnar-register` | Columnar Sales / Purchase Register with Extract | - | P1 | L | none |
| `delivery-receipt-note` | Delivery/Receipt Note linked to the invoice (tracking numbers) and Bills Pending reports | tracking numbers and Bills Pending | P1 | L | none; data-integrity risk, not only a gap |
| `batches` | Batches with mfg/expiry dates | remaining: picker, mfg date, opening batches | P1 | L | none (board said covered) |
| `expiry-reports` | Expiry reports / expired-stock handling | - | P2 | M | none |
| `bom-definition` | BOM master on the Stock Item (Set Components) | - | P0 | M | briefed ROADMAP-ADDENDUM (Bom model); nothing built |
| `bom-multiple` | Multiple named BOMs per finished item | - | P1 | M | briefed; depends on the BOM master |
| `mfg-stock-posting` | Consumption/production posting with correct valuation | - | P0 | M | briefed ROADMAP-ADDENDUM (costManufacture()); not built |
| `manufacturing-journal` | Manufacturing Journal as a distinct voucher type | - | P1 | M | not briefed; editId path exists but nothing routes to it |
| `employer-contribution` | Employer statutory contribution heads and processing (EPF/EPS, ESI, NPS) | - | P1 | L | overlaps payhead-statutory-types; design in ROADMAP-ADDENDUM (PayHead.role employer) |
| `salary-revisions` | Effective-dated salary revisions | - | P1 | M | briefed ROADMAP-ADDENDUM (Employee.salaryHistory) |
| `arrears` | Arrears on retrospective salary revision | - | P1 | M | needs salary-revisions first |
| `multi-payslip` | Multiple pay slips (batch print by group or category) | - | P1 | M | none |
| `payroll-report-filters` | Payroll report filters by employee category, group, employee | - | P1 | S | group filter ships before categories exist |
| `payroll-autofill` | Payroll autofill by group or single employee | - | P1 | M | briefed ROADMAP-ADDENDUM |
| `cheque-print` | Cheque printing from the payment voucher | - | P1 | M | none; physical alignment needs a real leaf test print |
| `payment-advice` | Payment Advice (supplier advice letter) | - | P1 | M | none (no e-mail transport) |
| `cheque-blank-date` | Cheque printing with the date left blank | - | P2 | S | none |
| `partial-match` | Partial-match reconciliation and amount tolerance | - | P1 | M | none |
| `brs-create-voucher` | Create vouchers from statement rows (multi, consolidated, optional-first, bill-wise) | - | P1 | M | none |
| `edit-log-record` | Edit Log per record (Alt+Q from a voucher or master in alteration) | - | P1 | M | Wave A universal '+' claims Alt+Q (chord collision, not counted) |
| `edit-log-summary` | Edit Log Summary (voucher type, master, user) with drill and filters | - | P1 | L | none |

### Wave 3: statutory depth and contract-dependent work

Needs a spec, real files, an external contract or TLS first. Start the unblockers now even though the build comes last. 29 items (1 S, 6 M, 12 L, 10 XL; effort is the whole row's, a slice is smaller).

| Item | Feature | Slice or note | P | Eff | Depends on |
|---|---|---|---|---|---|
| `gst-json-export` | GST returns export formats (portal JSON, Excel/CSV) | verify with the CA's real GSTR-1 and 3B, then Excel/CSV layouts | P0 | L | needs a real GSTR-1 and 3B from the owner's CA (ROADMAP-ADDENDUM); online route needs a GSP |
| `eway-bill` | e-Way Bill generation and management | offline bulk-upload JSON first | P1 | XL | offline bulk-upload JSON needs no contract (L); live needs a GSP **Phase 29 (2026-10-22): offline half built; see (h)** |
| `gstr2b-recon` | GSTR-2A/2B reconciliation depth | - | P1 | L | offline parts need no contract; download needs a GSP |
| `ims` | IMS: Invoice Management System actions and reports | offline file-based first | P1 | XL | file-based Supplier Inbox briefed (comsky-connect-network.md); live sync needs a GSP |
| `composition-scheme` | Composition dealers: CMP-08, GSTR-4 | - | P2 | L | none briefed |
| `gstr9-export` | GSTR-9 export to a portal file, annual HSN summary, GSTR-9C support | - | P2 | M | build only when a customer files GSTR-9 from the app |
| `tds-natures-seed` | Income-tax Act 2025 natures: sections 392/393/394, payment codes, Alt+H helper | Act-2025 natures and payment codes | P1 | M | statutory values must be researched and cited (CLAUDE.md); renumbering listed as unmodelled |
| `etds-fvu` | e-TDS/e-TCS return file for FVU, and Forms 138/140/143/144 by period | blocked on the NSDL/Protean spec | P1 | XL | blocked on the NSDL/Protean record-layout spec and an FVU validator (not in repo); ROADMAP-ADDENDUM 'skip' |
| `form-26q` | Form 26Q (now Form 140 from 1 Apr 2026) return handling | - | P1 | L | same blocker as etds-fvu |
| `tcs-nature-master` | TCS Nature of Goods master depth | - | P2 | M | none |
| `tcs-auto` | Automatic TCS on receipts, advances, non-item sales | - | P2 | L | none |
| `pf-computation` | Provident Fund computation and heads | - | P1 | XL | needs payhead-statutory-types, rounding, F11 statutory first; rates must be researched and cited |
| `pf-reports` | PF statutory reports (ECR, Forms 3A/5/6A/10/12A, challan) | blocked on the EPFO ECR spec | P1 | XL | blocked: needs the EPFO ECR spec and PF-from-posted-payroll; no research brief |
| `esi-computation` | ESI computation and heads | - | P1 | L | depends on payhead-statutory-types and rounding |
| `esi-reports` | ESI statutory reports (Monthly Statement, Forms 3/5/6) | blocked on the ESIC format | P1 | L | blocked: needs the ESIC return format; no research brief |
| `pt-computation` | Professional Tax computation | - | P2 | M | Tally ships no state slabs either |
| `pt-reports` | Professional Tax statement and computation report | - | P2 | M | none |
| `salary-it` | Income tax (TDS on salary) computation | - | P1 | XL | briefed ROADMAP-ADDENDUM (core/salaryTax.ts); slabs must be researched and cited per FY |
| `tax-regime` | Employee tax regime (old / Sec 115BAC) with revision history | - | P1 | S | belongs with salary-it |
| `it-declarations` | Income tax declarations, tax configuration, overrides | - | P1 | L | belongs with salary-it |
| `brs-auto-import` | Auto BRS via statement import: Excel files, stored statements, bank layouts | .xlsx statements, one real statement per bank | P1 | L | briefed ROADMAP-ADDENDUM (xlsx reader); needs one real statement per bank |
| `many-to-many` | One-to-many and many-to-many matching | - | P1 | L | named in ROADMAP-ADDENDUM, not started |
| `epayments` | e-Payments bulk payment file export and reverse-file import | generic CSV first | P2 | XL | ROADMAP-ADDENDUM proposes a generic CSV (M) first; certified formats need each bank's spec |
| `remote-users` | Named users with their own rights (invite and accept) | needs e-mail delivery and server-enforced roles | P1 | L | ROADMAP-ADDENDUM: needs e-mail delivery, per-member rights, server-enforced roles |
| `company-vault` | Company password: encryption of books | after TLS | P1 | L | ROADMAP-ADDENDUM: TLS first (WebCrypto absent on the http-only VM), then an AES-GCM envelope |
| `encrypted-backup` | Password-protected backup with recovery key | after TLS | P1 | M | ROADMAP-ADDENDUM: AES-GCM envelope after TLS |
| `einvoice-irn` | e-Invoice online IRN generation, cancel | blocked: GSP contract, TLS, credential vault | P1 | XL | needs a GSP contract, TLS, credential vault (einvoice-gsp.md) |
| `connected-gst` | Connected GST: upload, file, download returns from the app | blocked: GSP | P1 | XL | needs GSP, TLS, credential vault (einvoice-gsp.md) |
| `connected-banking` | Connected Banking: live balance and online statement fetch | blocked: bank contract | P2 | XL | live-bank-feeds.md: bank or aggregator contract and TLS first; HANDOFF trust map (HTTP-only single VM) blocks it |

Everything else on the pending list is Wave 4 and later: P3 rows, the job-work chain (one XL epic: material out and in, orders, godown type, registers, GST and e-way on them), multi-currency (one block, mainly for exporters), multi-GSTIN, multi-lingual, and mobile layout.

## (h) Phase 29 (the build programme's parity wave 3), 2026-10-22: what it addressed

> Phase 29 is not the "Wave 3" of section (g): it built a slice of that section's Wave 1 and Wave 2, plus the offline half of the e-way bill. It is built in the working tree on top of `83d9876` (three of its five clusters, cancel and party terms, e-way and dispatch, and export and share, also went through an adversarial review and fix pass; single entry and server numbering did not, in the reports this was written from) and was NOT committed or deployed when this was written. Sections (a) to (g) above stay as generated on 2026-10-20 (only the tags mentioned in the status note were added), and the JSON twin was not touched.

**What the board now says.** `docs/parity-data.json` carries the 15 rows Phase 29 touched and none of the other corrections in (d1): 258 / 44 / 79 (73.5%) became **250 / 58 / 73 (73.2%)**. Six rows went up: `cancelled-register` to covered, and `credit-limit-override`, `eway-bill`, `eway-register`, `report-email` and `numbering-override-multiuser` from missing to partial. Nine went down from covered to partial, which is the correction this file proposed for each of them, now applied because the wave rebuilt part of the row and the note had to agree with the status: `credit-limit`, `interest-simple`, `interest-bills`, `interest-report`, `price-levels`, `report-export`, `export-pdf`, `import-excel`, `deleted-tracking`. `report-export` is a P0 row, so six P0 rows are partial (it was five). The weighted figure fell by 0.3 point while the product gained; that is the honest arithmetic of re-judging first-slice rows. `cancelled-register` is the one place the board is above this file's corrected status (partial), because the missing piece, Alt+X, is what the wave built.

**Pending items addressed**

| Item (as named in (b), (c), (g)) | Phase 29 outcome | Board row (status now) | Still open |
|---|---|---|---|
| Cancel voucher (Alt+X) keeping its number | Built: Alt+X, a shell voucher every engine reads as absent (20 engines asserted), number never reused, Table 13 and the doc_issue JSON count it, read-only view, reason required, IRN and e-way warning, cloud keeps it cancelled | `cancelled-register` (covered) | un-cancel (as Tally), Alt+X on the Ledger Vouchers drill, bulk cancel, ISCANCELLED in Tally XML (skipped and counted on import, omitted and counted on export), portal cancel |
| Deleted-voucher register: operator and other devices | Deleted By is now the operator; the register shows Cancelled (number kept) beside Deleted | `deleted-tracking` (partial) | deletions made on another device are not listed |
| Single-entry layout for Payment, Receipt, Contra | Built as an OPT-IN layout (Voucher Type > Entry layout, or Ctrl+H / Alt+V per person); saved vouchers identical to grid vouchers; the default is asserted unchanged | `single-entry-mode` (partial) | the default is still the grid (Tally defaults to single entry); the Account side for Contra and Receipt is this build's convention |
| Credit limit per party with enforcement | The field exists (party ledger tail field); one shared engine for the voucher, Bulk Entry and the report; supplier side; altered vouchers; a reason on override | `credit-limit` (partial) | blocking mode; Journal, Payment, Receipt and Sales-as-voucher checks; Bulk Entry takes no reason |
| Credit-limit override and exceeded handling | Reason-recorded override kept on the voucher; Credit Limits report (over, near, within, every override) | `credit-limit-override` (partial) | authorisation by another user; Tally's post-dated override |
| Credit Limits bulk screen (Alter > Credit Limits) | Not built (the report is not the grid) | none | whole item |
| Interest parameters on the ledger master / Enable Interest Calculation | Rate and style fields; the report rewritten to a daily balance, receivable and payable apart, never past today | `interest-simple` (partial) | F11 switch, bank-date basis, compound or per-voucher override, bank/OD ledgers |
| Interest on settled-late and open bills | Each open bill charged from the day after its due date to the as-on date | `interest-bills` (partial) | settled-late bills, grace period, calculate-from, slabs |
| Interest report depth | Not built (two columns, two totals) | `interest-report` (partial) | group view, bill-wise detail, settle-bills, period comparison; booking as a Debit/Credit Note |
| Price levels and price lists | The party's default level (field) and Alt+Z on the voucher; the level reaches the rate and is saved | `price-levels` (partial) | dated and group lists, slabs, discounts, Alt+R; items already entered keep their rate |
| Report export in real Excel (.xlsx), JSON, XML, HTML, image | A real .xlsx for every report (60 then 71 mutations killed); the dead HTML .xls path is gone | `report-export` (partial, P0) | JSON, XML, HTML, image; the file was never opened in real Excel or LibreOffice |
| Direct PDF file export | Alt+F9: a browser opens its print window and names Save as PDF; the desktop app writes the file through printToPDF behind a validated bridge | `export-pdf` (partial) | the desktop write was NEVER run (no Electron binary here): rebuild the installers and run it once |
| Share by e-mail and WhatsApp | A Share panel (Alt+M): wa.me and mailto open the person's own app, copy text, share sheet, Excel or PDF offered first | `report-email` (partial) | nothing is sent or attached by the app; no templates; WhatsApp Business and a mail sender are not built |
| Payment Request: UPI QR and payment link on invoices | A static UPI QR (owner-chosen, default off, needs a valid UPI ID) | none (not a board row) | payment link, partial payment, reconciliation; the NPCI link format is cited from a third-party copy of the 2017 draft, to be re-checked |
| Bulk print, export and e-mail of vouchers for a party and period | Bulk print from a register selection (one job, each voucher's own paper and copies, at most 100) | none (not a board row) | bulk export and bulk e-mail |
| Import from Excel with mapping templates and samples | .xlsx and CSV with column mapping remembered per file shape, a preview with per-row errors, one additive balance-guarded write | `import-excel` (partial) | vouchers are two-ledger rows; .xls; sample files; checked on this app's own fixtures only |
| Dispatch, consignee, order and transport details on the invoice | An Alt+S panel (delivery note, dispatch, LR/RR, vehicle, terms, consignee, dispatch-from); Ship-to prints the consignee; the e-invoice carries DispDtls and ShipDtls; a consignee never moves the place of supply | none (not a board row; `delivery-receipt-note` and `tax-invoice-print` carry it) | no order number or date field; the Tally XML export does not carry the consignee |
| e-Way Bill generation and management | The offline path: Part A and B, cited validation (8 rules flagged uncertain), the portal's bulk JSON, a tolerant results import | `eway-bill` (partial) | the bulk file was never uploaded to the portal (key names from ERPNext and a Vayana sample); no API, cancel, extend or Part B update; no QR; needs a GSP |
| e-Way Bill register | Built: needs details, ready, generated, expiring, expired, cancelled, changed after the bill, not in the books | `eway-register` (partial) | fed only by the imported results file; no status polling |
| e-Invoice online IRN generation, cancel | Not addressed except that the payload now writes DispDtls, ShipDtls and EwbDtls | `einvoice-irn` (partial, unchanged) | everything in (b) that needs a GSP |
| Multi-user voucher numbering consistency / Automatic (Manual Override), Multi-user Auto, None numbering | The client asks the cloud for the number on Accept; offline numbers are provisional and claimed at sync; a clash is reported and never renumbered silently; Renumber asks first | `numbering-override-multiuser` (partial) | THE BACKEND HALF IS NOT DEPLOYED; Auto (manual override), None and named series not built; RCM and Rule 37 journals and bank-statement import still number locally |
| Voucher type options | Entry layout only | `vouchertype-flags` (partial, unchanged) | share-after-save and the rest |
| Named users with their own rights (invite and accept) | Assessed only, not built (see `docs/HANDOFF.md` section 2f) | `remote-users` (unchanged) | whole item; needs TLS, a mail sender, an invite table and server-enforced per-company roles |

**Not addressed by Phase 29:** every other pending item in (b) to (g), including all the P0 unblockers in (g), the job-work chain, payroll, TDS forms and the rest of Wave 1 and Wave 2 not listed above.

**Limits.** All of this was checked in headless Chrome on macOS only. None of it was compared with a live TallyPrime, the GST portal, the IRP, the e-way bill portal, a UPI app, WhatsApp or real Excel, and the desktop PDF bridge was exercised against a fake Electron. The statements about what Tally does remain this file's own reading (see the limits in (a)).

## Appendix: how the numbers were made

- Old and corrected counts are computed by script from the snapshot `docs/parity-data.json` (381 rows) with the confirmed corrections applied; every correction id was checked to exist with the board status the auditor stated (103 of 103 matched).
- Weighting is `scripts/gen-kanban.py` `renderCov()`: `round(100 * (covered + 0.5 * partial) / total)`; shown here to one decimal.
- Pending and never-on-board rows are the 13 auditors' lists merged by capability (duplicates across modules collapsed; where they disagreed the owning module's priority and effort were kept). All 208 board rows that are not covered after correction appear in b1 or b2 (merged ids are listed in the JSON).
- Skeptic verdicts were applied as stated: a refuted claim is dropped, and a stated correctedStatus overrides the auditor. Every confirmed claim kept its status; five were refuted.
