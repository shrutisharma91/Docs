# Pharmacy retail and wholesale software: what the incumbents do, what the law asks of the counter, and what a pharmacy mode in Comsky Books should be

> **Status:** research and design input only. Nothing here is built and this document changes no code. **First written 2026-10-22 against `main` at `83d9876`;
> revised 2026-10-25 (the resume pass).** The code facts were re-checked on 2026-10-25 against `main` at `5aee765`, where the pure batch, serial, scan and label
> engines are merged as new files that no screen imports (`src/core/batches.ts`, `stockCell.ts`, `trackRules.ts`, `unitId.ts`, `scanResolve.ts`, `labelRules.ts` and
> `src/core/barcode/`). The custom-fields and stock-by-field build in the worktree `.wt-cf` was read (read-only): its tip is one squashed commit, `2517916`, whose message says it
> was reviewed and gated; I did not verify that, nor whether it has merged. Two documents of the same sweep were read for what a pharmacy counter needs from them:
> `docs/research/pos-code-audit.md` (what a POS bill touches in today's code) and `docs/design/batch-serial-labels-spec.md` (the batch, serial, label and scan
> spec; **it did not exist when this document was first written**, and section 6.4 is rewritten against it).
>
> **What this revision changed (2026-10-25), so a reader of the first version can find the differences:**
> 1. **Section 6.4 is rewritten against the batch / serial spec.** Its ten pharmacy requirements (R1 to R10) are mapped to the spec's decisions (T1 to T36) and rules (R1 to R31):
>    **five are delivered, four are partly delivered and one is not designed**; twelve further asks the first version could not make follow (A1 to A12). **Two findings there are
>    defects in waiting for that spec's lane, not wishes:** a revised MRP written through the batch overlay moves a sale off its batch's cell (F1; reproduced in a scratch script
>    that calls the merged functions, nothing written in the repo), and the spec's
>    pharmacy dispense label follows Rule 94(2), which is the wrong rule for a cut strip (F2).
> 2. **Items the first version called "not researched" were researched where a source could be read:** what the NDPS Rules ask of a retail seller (partly), the legal standing
>    of an electronic prescription (context only), the dates of the DPDP Rules, the physician-side rule on commissions, GST on free goods (CBIC Circular 92/11/2019-GST,
>    through secondary summaries), the June 2026 extension of the QR rule, pregabalin's entry in Schedule H1 (G.S.R. 377(E), May 2026) and NHCX. Each is marked where it
>    appears; what is still open is in section 7.
> 3. **Rule 65, Rule 94(2), Rule 96 and Schedules H and H1 were re-read in the CDSCO consolidated PDF** (its text layer extracted locally). The first version's paraphrases held;
>    three things were added (the provisos to 65(3), clause 65(4)(1) for Schedule C, and Rule 74(p) on advertising by manufacturers).
> 4. **New:** batch alerts from CDSCO's monthly NSQ lists (3.20); the point-of-sale findings of the POS code audit (3.15, 3.19); a refreshed Busy row; the RxPOS listing; a
>    re-check of the Marg review score (unchanged, and two new complaint themes); stale `file:line` references repaired.
>
> **The ask (owner, verbatim intent):** "Tally is not used in POS as it is weak ... I want to give a POS function in this, so retailers can be
> targeted. Check what is good in these software and add. Another example is Marg, which does well in pharma; also LogicERP."
> **This document's slice:** PHARMACY, retail (the chemist's counter) and wholesale (the stockist and distributor). Restaurant and
> general-retail POS, the batch / serial / IMEI / label-printing / barcode spec (`docs/design/batch-serial-labels-spec.md`, **written after the
> first version of this document; section 6.4 now reads pharmacy against it and does not redesign it**) and mobile are other documents.
> **Classification:** a vertical mode of POS is a *Beyond Tally* capability. Track it under `beyond` in `docs/parity-data.json`, never in
> the parity percentage (CLAUDE.md, "Phases 18-21").
> **Evidence tags (every claim carries one):** **[W]** a page I fetched and read (I paraphrase; I do not copy; at most one short quote per
> source) · **[Ws]** a search summary only, the page itself could not be read · **[C]** read in this repo (`file:line`) · **[J]** my
> judgement, not evidence · **[unverified]** I could not confirm it. `S<n>` points to the source list in section 8. **Labels:** "R1 to R10" in sections 3 and 6 are this
> document's ten requirements of the batch design (6.4); the batch spec's own rules are always written "spec R11" or "its R11", its decisions "T4"; "A1 to A12" and "F1, F2" are asks
> and findings in 6.4; "T1 to T28" in section 5 are this document's traps.
> **What I could not do, plainly:** I used no live account of any product and no hardware, so every statement about how a product *feels*
> or how fast it is comes from vendor pages and reviews, and a vendor page is marketing. Several review sites (Software Advice, Capterra, G2)
> returned HTTP 403 to the fetch tool, so their content is **[Ws]** only. Statutory text was read from the CDSCO-hosted consolidated PDF of the Drugs
> Rules, whose consolidation date I could not determine and which lags the gazette (it holds 48 entries in Schedule H1; one more was notified in May
> 2026, see 3.9). **Still open after the second pass (each is tagged [unverified] where it is used and listed in section 7):** whether a software
> printout satisfies a register written for paper; what an NDPS-specific retail register requires, if any (the NDPS Rules text I could read routes a retail
> seller back to the Drugs Rules, 3.9); whether a pharmacist may treat an e-prescription as satisfying Rule 65(10); TPA and NHCX flows at a retail
> counter; the chemist's side of a commission to a prescriber; whether the free-quantity treatment of Circular 92/11/2019-GST holds for every shape of
> pharma scheme (I read it only through secondary summaries, its own PDF would not yield text); and the PIB GST pages, which answered 403 on the
> second visit, so the GST table in 3.3 is as read on 22 October and is corroborated here only by search summaries. I looked for "RxPOS": one
> marketplace listing exists and says little (2.6).

---

## 0. TL;DR

1. **Pharmacy is the Indian retail vertical where an accounting-first product lost and a specialist won, and the reason is in the statute.**
   Rule 65 of the Drugs Rules makes batch number, manufacturer and the licence number part of the records of *every* purchase and wholesale
   sale and, for prescription drugs, of the retail record too [W S20]. A general POS that does not carry the batch cannot produce the
   records an inspector asks for. Marg (the dominant name; the vendor claims about half of Indian pharma and FMCG billing, **unverified**
   [W S1]) built the whole counter around that fact.
2. **What a chemist's counter must do, in order of how often it is used [J, from S1, S2, S11, S14, S15]:** find the product by brand,
   salt or barcode and see batch, expiry, MRP, stock and rack in one list; sell a strip or a loose tablet; refuse or warn on expired and
   near-expiry stock; take doctor and patient details on a scheduled drug; offer a same-salt substitute; show margin and warn when a
   bill loses money; park a bill and serve the next customer; record the medicine that was out of stock so it gets ordered. Then the back
   office: purchase entry with free-quantity schemes, expiry returns to the stockist, reorder, GST.
3. **Marg wins on workflow depth, a dealer network, one-time pricing and, newer, a chemist-to-distributor ordering network.** It loses on
   support and renewal cost (Trustpilot: 2.3 of 5 across 116 reviews, polarised 53% five-star and 36% one-star; re-read on 25 Oct on two country
   pages with the same figures, the recent complaints now also naming printing faults after an update and stock values that changed unexpectedly
   [W S9]), on a desktop file database that its own help pages say is damaged by power cuts [W S7], on a dated interface, and on a growing pile of
   side businesses (insurance, loans, a web shop, health checks) [W S1]. Those are the openings; the depth is what must be matched first.
4. **The law shapes the counter more than any competitor does.** Retail memo must show the dealer's licence number [W S20 65(4)(3)]; a
   Schedule H, H1 or X item is sold only on a registered practitioner's prescription (65(9)); H1 sales go in a separate register kept three
   years (65(3)(1)(h)); a repeat dispensing needs the prescriber's say-so (65(11)); **no other preparation may be supplied in lieu on such a
   prescription (65(11A))**; nothing is sold or stocked after expiry, except returns-pending stock kept apart and marked "Not for sale"
   (65(17)); a loose supply goes in a labelled container under a pharmacist (65(19)). Two drafts are in their comment windows now: CCTV for
   prescription supply (G.S.R. 791(E), 8 Sep 2026) and a ban on retailers advertising Schedule H, H1 and X drugs without Central approval
   (G.S.R. 861(E), 28 Sep 2026) [W S23, S24]. Neither is law yet. **One change is law with a date on it:** pregabalin was added to Schedule H1
   (entry 51) by G.S.R. 377(E) of 13 May 2026, in force 180 days after Gazette publication, which is about 16 November 2026 if the reported
   20 May publication date is right [Ws S37]; the H1 register duty then reaches every pregabalin sale.
5. **GST is simple in rate and messy in practice.** Since 22 Sep 2025 most medicines (heading 3004 and the neighbouring 3001 to 3006) are
   at 5%, 36 named drugs are nil, and a pharmacy still sells 18% items beside them [W S25]. The rate follows the date of supply, not the
   date of purchase, so stock on the shelf took the new rate on the day [W S25 FAQ]. Manufacturers had to cut MRPs from that date and
   need not re-sticker old stock if the cut reaches the counter, so **two MRPs for one product exist in the market** [W S26].
6. **Three well-selling features are legal or ethical traps if copied as shipped:** substitute suggestions ranked by incentive (Marg
   FLIP [W S1]) against 65(11A); WhatsApp "prescription refill reminders" against 65(11) and the draft advertising rule; doctor
   commission tracking [W S2, S14] (the physician's receipt of a commission is barred by clause 6.4.1 of the Medical Council's 2002 ethics
   regulations [W S38]; the chemist's side I could not settle, so legal review before any build).
7. **Data is half the product and half the liability.** Composition, schedule membership and ceiling prices are dated data, not code.
   Schedule H in the CDSCO copy I read has 551 numbered entries and extends to their salts, esters and derivatives with carve-outs for
   topical use; H1 has 48 there, and entry 51 (pregabalin) was notified in May 2026 [W S20; Ws S37]. A flag derived from a hard-coded list goes
   stale silently (the CDSCO consolidated copy was already behind the Gazette on the day I read it). Ship an editable, dated list with a stated
   source, and do not claim a drug database.
8. **Where Comsky stands today [C, re-checked 2026-10-25]:** an item line carries `batchName`, `mfgDate`, `expiryDate` and the engine keeps quantity per batch
   (`src/types.ts:953-955`, `src/core/inventory.ts:14,141-159`), but the voucher's batch prompt takes a typed batch name and expiry only (no
   pick from stock, no MRP, no mfg date: `src/ui/VoucherScreen.tsx:883-915`), the Batch & Expiry Summary is a stub
   (`src/ui/inventory/BatchSummary.tsx:3`), and there is no MRP field (`docs/FEATURE-PARITY.md:150`). **None of that has changed on `main`**: the
   engines of the batch spec (a derived batch master, FEFO proposal, expiry maths, the MRP ceiling, cell keys) were merged on 25 October (`5aee765`) but **no screen calls them**.
   **Two parity rows are stale and say "covered" beside "No batch/expiry concept"** (`batches`, `expiry-reports` in `docs/parity-data.json`, rows `:163` and `:173` of
   `docs/FEATURE-PARITY.md`); fix them when next edited (the spec's decision T30 does so when its work starts).
   Strong and reusable: effective-dated GST history, price levels, credit limits with a reasoned override, compound units, bill-wise
   ageing, cheque and PDC, e-way bill, WhatsApp and UPI links, an audit trail on every write.
9. **Build in three tiers (section 6).** Tier 0 is what makes it a pharmacy at all: batch pick with FEFO and an expiry block, per-batch
   MRP with tax-inclusive billing, strip and loose, schedule capture with a printable register, licence number and pharmacist on the bill,
   a "Not for sale" location, expiry return to supplier, free-quantity purchase schemes. Tier 1 is speed and money. Tier 2 is data
   licences and networks.
10. **How much hangs on the batch / serial spec (written since the first version; read against in 6.4).** *Dependency:* of the 14 Tier 0 items, 5 cannot work without it, 6
    work only in a degraded form without it and 3 do not depend on it. *Coverage:* it already designs 3 of the 14 (the batch pick with FEFO and the expiry block, and the
    near-expiry and expired reports), designs 6 in part and **does not design 5** (strip and loose, the schedule gate, the registers, licence numbers and the pharmacist,
    free-quantity schemes; it says in terms that the register and free quantity are not in its first version, its Q26 and section 18, and is silent on the other three,
    none of which conflicts with it). Of the ten requirements this document first
    asked of it, 5 are met, 4 are partly met and 1 (loose stock) is not designed. Plainly: **the spec gives a pharmacy a correct batch engine and a sale-time pick; it does not
    make a pharmacy product.** The core sale path (its slices T-0, T-1, T-2 and T-4) is about 52 to 64 builder-days by its own table [J], and the pharmacy-specific work beside it is
    about 60 to 80 more on the same scale (6.4) [J]. The sale path is not pharmacy-grade until both exist.
11. **Two findings for the spec's lane before its panel is built (6.4):** (F1) a lower MRP written through the batch overlay, the route the spec leaves for a revised price,
    would make the next sale line name a different cell from the batch's own, so the sale would hit a phantom cell; (F2) the spec's pharmacy "dispense" label implements Rule
    94(2), which covers a medicine made up for a patient, not a tablet cut from a strip (Rule 65(19)); the pharmacy preset's "loose-dispense" label can only be that template.
12. **Compliance wording:** say what the software prints (the columns Rule 65 lists), never that the shop "is compliant". The licensee
    carries the duty; inspectors differ by state; whether a software printout satisfies a register written for paper is state practice
    [unverified] (6.3).

---

## 1. What was studied and how much weight it carries

| Layer | What | Strength of evidence |
|---|---|---|
| Vendor pages, read in full | Marg retail, distribution, drug-inventory pages and four blogs; LogicERP pharma, retail and distribution pages and four knowledge-base articles; GoFrugal pharmacy pages and blogs; Busy pharmacy and medical-store pages; Medeil (via a marketplace page); Zoho and Odoo help pages | Marketing. Good for "what exists and how it is described"; says nothing about quality or speed. [W] |
| Vendor's own admissions | Marg help-centre pages on database damage, DBF errors, "Not Responding" | Better than marketing: the vendor describing its own failure modes. [W S7] |
| User voice | Trustpilot (116 reviews, re-read 25 Oct), SoftwareSuggest (small, 2015 to 2020), plus search summaries of Software Advice, Capterra, G2, Techjockey and, for GoFrugal, Capterra (a general-retail sample, not pharmacy) | Thin. Trustpilot's mix is polarised and the vendor does not solicit reviews; the others are small and old. [W S8, S9; Ws S10, S39] |
| Competitor and implementer pages | Accountune's comparison, a Ksolves ERPNext case study, Marg's comparison with eVitalRx | Written by a party with a stake. Used only for facts that cut against the writer, or tagged. [W S5, S19] |
| Statute and regulator | Drugs Rules 1945 (CDSCO PDF; Rules 65, 74(p), 94(2), 96, 97 and Schedules H and H1 re-read in its text layer on 25 Oct), a Punjab and Haryana High Court judgment of 18 Aug 2026, PIB release and FAQs on the 56th GST Council (read 22 Oct; 403 on 25 Oct), CBIC circular 72/2018, two draft notifications reported by Medical Dialogues and SCC Online, an NHSRC register format; added on 25 Oct: the NDPS Rules text (a case-law site's copy, summarised by the fetch tool, so a weaker read), a state medical council's copy of the 2002 ethics regulations, CBIC Circular 92/11/2019-GST through secondary summaries, news reports of G.S.R. 377(E), of the June 2026 QR amendment and of the DPDP Rules, and one CDSCO NSQ alert PDF | The strongest layer. Primary text where I read the PDF; the drafts and the 2026 notifications are secondary reports of the Gazette. [W S20 to S28, S37 to S49] |
| This repo | Types, engines (including the batch engines merged on 25 Oct and not wired), two worktree specs, the batch / serial spec, the POS code audit | Fact about the code. [C] |

Two reading rules used throughout. **"Y" in the matrix in 2.8 means a page I read lists the capability; a dash means I did not find it on
the pages I read, which is not the same as absent.** And every number that a vendor gives about itself (shares, user counts, "up to 40%
faster") is repeated only as a claim.

---

## 2. The market

### 2.1 Landscape

| Product | Who it is for | Delivery | Price signal | Note |
|---|---|---|---|---|
| **Marg ERP 9+** (Marg ERP Ltd) | Retail chemist, pharma distributor, stockist; also FMCG, retail, manufacturing | Windows desktop, a hosted Windows VM option, mobile apps | Retail: Nano Rs 5,550, Basic Rs 10,300, Silver Rs 13,900, Gold Rs 26,000, all plus 18% GST, one time; Rs 3,000 per extra user or company; AMC separate, reported Rs 2,000 to 5,500 a year; sold only through dealers [W S1; Ws S10] | Aggregator listings quote different figures; treat the vendor page as the list price |
| **LOGIC ERP** (Logic Software) | Pharma retail, wholesale and distribution; also FMCG and others | Cloud or on-premise | Not published [W S11] | A knowledge base that is unusually detailed about schemes [W S13] |
| **GoFrugal Pharmacy** | Single stores to chains | Cloud or on-premise, mobile billing | Not published; 15-day free trial [W S14] | Pitches inter-store control, doctor commission, SMS reminders |
| **BUSY** | General accounting with a pharmacy and medical-store edition | Desktop, cloud, mobile app | Plans Start, Smart, Power, Power+; prices on a page I did not read [W S15] | Strong GST depth; schedule-drug handling not named on the pages read |
| **Medeil** (Vanuston, Chennai) | Retail and hospital pharmacy | Described inconsistently (cloud, web, desktop) | From about Rs 6,500 plus GST, with a conflicting Rs 25,000 on one FAQ [W S16] | Two reviews only |
| **RxPOS** (Sekhon Technologies) | A pharmacy management and POS listing | Described both as cloud and as web or on-premises; a Windows and macOS client per its FAQ | Price on request | One marketplace listing that contradicts itself (its FAQ calls it POS software for restaurants); it names prescriptions, expiry and supplier management, and names no batch, schedule or insurance handling [W S41] |
| **eVitalRx** | Small and chain pharmacies, cloud-only | Cloud, ONDC-linked | Subscription | Described by Marg, a competitor [W S5] |
| **Vyapar, Accountune, Commmerce** | Small shops, cheap and simple | Cloud or app | Published, low | Batch and expiry described as thin for Vyapar, by a competitor [W S19; Ws S19] |
| **Zoho Inventory, Odoo** | General ERPs used by pharma distributors and chains with an implementer | Cloud | Public tiers (not read) | Batch, expiry, FEFO yes; pharmacy compliance is custom work [W S17, S18; Ws S17] |
| **Tally, Busy as accounting back-ends** | The accountant, not the counter | Desktop | One-time | Several pharma tools claim a Tally bridge [W S11, S5] |

### 2.2 Marg ERP: why it dominates, and what people say

**Why it dominates [J, resting on S1 to S6 and S9].** Six reasons, none proven by sales data I could see:

1. *Compliance by default.* Batch, expiry, doctor and patient, the Schedule H indicator, and the retail memo particulars are in the billing
   flow, so a chemist with no accountant still holds the records the rules ask for.
2. *Trade-economics tooling that turns into cash.* Free-quantity schemes, the last four purchase deals shown when ordering, a warning when a
   bill loses margin, expiry returns, reorder suggestions from sales: each one saves or earns money the owner can see.
3. *Keyboard speed on modest PCs.* The pages stress shortcuts, quick search and barcode scanning, with a claimed 40 to 50% faster billing
   (the two pages disagree: 40% on the retail and distribution pages, 50% on the drug-inventory page) [W S1, S2, S3]. Unverified.
4. *A price a chemist can say yes to,* one time, through a local dealer who installs and trains. The vendor claims 850+ sales and support
   centres and 8,500+ people [W S1].
5. *A network.* ERP-to-ERP ordering puts the chemist's order into the distributor's Marg with live stock, rates and schemes visible; the
   distributor side lists "ECOD Secure" for sharing stock and sales with a company's field staff, and a "Digital RCPA" market-survey service
   for pharma companies [W S1, S3]. A rival has to match not just features but who is already on the other end of the order.
6. *Money the chemist makes through the software,* not just saves: incentives on substitute sales (FLIP), health insurance and loan referrals,
   digital health services, a Rs 10,000 web shop [W S1]. This makes it sticky and is also the source of the ethical questions in 3.10.

**Retail feature inventory, grouped (all vendor claims) [W S1, S2]:**

- *Find and sell:* search by salt, substitute, rack, name, barcode, strip code or item code; sell in strips and tablets; sale by doctor,
  patient, salesman or operator; prescription-wise sale with pending reminders; cash, credit and *panel* bills and challans; flexibility in stock
  for government-employee bills; several sale series and a wholesale billing option; several customers billed at once on one PC; quick switch
  between sale, purchase and back; WhatsApp bills.
- *Warn:* a Schedule H or narcotic indicator to check the prescription; an expired or near-expiry indicator; a margin-loss indicator.
- *Customer:* card and points; family-wise consolidated ledger; "Promise Order" (save a medicine the shop could not supply, to fulfil later);
  automatic prescription reminders on WhatsApp; suggested items from purchase history.
- *Purchase:* automatic load from an order or an email; photo or PDF of the supplier bill turned into a purchase with item mapping; the last
  four deals of the item (rate, scheme, discount, tax, cost) shown at entry; sale rates fixed on MRP and purchase rates on a margin; stock
  adjustment without a purchase; pending breakage and expiry, debit and credit, and replacement notes.
- *Reorder:* suggestions from sale, minimum level, manual, today's sale or zero stock; supplier-wise company lists and day-wise ordering;
  a supplier-wise order limit; a central order system for a chain.
- *Control and accounts:* a log of bills not printed, number of prints and modifications; operator-wise log book and cash collection;
  Gross-profit reports by day, month, item, company, bill and supplier; ABC, fast and slow movers; doctor's and MR's incentives and
  commission; bank reconciliation; GST filing from the product (GSTR-1, 2, 3B, 4, 9; 2A and 2B reconciliation; clause 44 of the tax audit
  form); e-invoicing; ABHA linking in the bill [W S2, S4].
- One line in the list reads "Adjust Margins of Bills for personal sales". I could not tell what it does. **Do not copy it** without finding
  out: anything that edits the margin recorded on a bill sits badly with an audit trail [J].

**Wholesale inventory [W S3]:** GST, e-invoice and e-way bill; batch, expiry and MRP and scheme control; temporary and permanent credit limits
set from behaviour; "purchase and sale claim management" (reminders for claims owed by manufacturers against purchases); restricted-drug
records and reports for Narcotics, H and H1; rack and mobile picking; "Push Sale" (suggest a near-expiry or dump item to the right retailer
from purchase history); customer search on a map filtered by date, route, company and salesman; order calling by QR; dispatch and delivery;
Marg Wallet payment links with WhatsApp reminders; an API; partner loan and insurance offers.

**What users love [W S8, S9; Ws S10]:** the pharmacy-specific features; expiry and payment reminders; GST filing from the product; automatic
reordering and stock-and-price cross-checks; reports; a quick, touch-responsive counter with a search fallback when a barcode fails (one
2015 to 2020 reviewer); built-in cloud backup that restored data after a ransomware attack (a reviewer, **Ws**); long-term users calling it
reliable. Reviewers of the small sample are mostly chemists; distributors are under-represented.

**What users hate [W S9, S8; Ws S10], most consistent first:**

| Theme | Evidence |
|---|---|
| Support after the sale | The most repeated complaint: calls and callbacks unanswered, tickets closed unresolved, dealer, care and IT pointing at each other (Trustpilot, July 2025 to October 2026) [W S9]. A Software Advice support score of 2.90 of 5 is quoted in a search summary [Ws S10] |
| Renewal and AMC | An AMC that rose from about Rs 2,500 to over Rs 5,500 plus GST in ten years, and different dealers quoting differently [Ws S10]; renewal cost called high; a free-SMS promise not honoured [W S9] |
| Dealer lock-in | A reviewer found it hard to change service provider and worried about data access and portability [W S9]. The vendor says it does not sell or support through online portals [W S1] |
| Fragility of the desktop database | Marg's own help pages list a power cut or a shutdown without closing the program, junk and temp files, antivirus and file permissions as causes of "database damage", DBF, syntax and "Not Responding" errors, and the repair steps are deleting CDX and temp files, running error checking, recovering an MST file or reinstalling [W S7]. DBF and CDX are names from the xBase file family, so this reads as a file-based store; that is **my inference [J]**, not a vendor statement |
| Interface and learning curve | Dated look, laggy at times, new users struggle with navigation and bulk price changes (G2 and aggregator reviews) [Ws S10] |
| Regressions after an update | Recent Trustpilot reviews name printing faults after an update and stock values that changed unexpectedly, and slow handling of GST invoices and paperwork [W S9, re-read 25 Oct]. A stock figure that moves by itself is the worst fault a chemist's software can have; it is a requirement here that every stock and batch figure is a pure function of the voucher log (the repo's engines are) and that a new version proves it by identity on the sample company (the spec's `identity-track` suite is that proof) |
| Cloud and mobile (older reviews) | Two SoftwareSuggest reviewers wanted a web or mobile version [W S8]; the vendor now sells apps and a hosted VM [W S1] |
| Pharma-centric clutter | A reviewer said it is not for other trades [Ws S10] |

Where the praise and the complaint meet: **the same chemist praises the features and curses the support.** A product that matches the
features and answers the phone is a real opening, and the answer-the-phone part is a business, not a software, problem [J].

### 2.3 LOGIC ERP

Retail and wholesale pharma pages and a knowledge base [W S11, S12, S13]. Retail: patient management, prescription reminders and
**prescription image upload at billing**; configurable billing screens; expiry and near-expiry prompts; a **cashier and shift system**;
several payment modes; **salt-based substitutes at billing**; **Schedule H and narcotics checks that block the sale unless patient and doctor
details are entered**; **discount blocking for chosen product categories**; a **red alert when margin turns negative**; **out-of-stock
medicines recorded at billing for supplier-wise purchase orders**; batch- and MRP-wise stock; expired and damaged stock handling; rack
location; strip-to-tablet conversion; supplier-scheme comparison and alerts at purchase; landed cost from a user-written formula;
min/max supplier orders; a ready template of 16,000+ items; Excel import of items and purchases; FIFO and FEFO suggestions on sale; cheque
printing and post-dated cheques; TDS and TCS; e-invoice and e-way bill; scheduled email reports and a mobile reporting app.
Wholesale adds party-wise credit and bill limits and payment terms, multiple price lists, **batch-wise free schemes**, distributor and
retailer margin shown at billing, the last rate charged shown on the line, area-wise billing, sales analysis by salesman or MR, market,
party, company and division, and a field-order app. Several items are tagged "claimed" even by the page's own summary: mobile and offline
billing, Tally integration, insurance claim integration, dosage and duplicate-drug alerts.

The scheme knowledge base is the most useful single document found (3.6). It also shows the shape of a mature scheme engine: groups,
dated periods, quantity or value basis, slabs, free item or discount, customer scoping, a budget with stop, prompt or allow, and four apply
modes.

### 2.4 GoFrugal Pharmacy

[W S14] Substitute lookup from a drug index by generic or brand; short-expiry alerts and a distributor-wise **expiry return screen** that
values a return at MRP, purchase rate or landed cost; a short-expiry warning at *purchase* inward; reorder of items expiring within 60 days;
a built-in **scheduled-drug register**, an **NDPS report** for narcotic and psychotropic drugs, and a **lock on editing a bill after the
schedule register has been printed**; doctor-wise sales and **doctor commission by percentage or amount slab per medicine**; customer-to-medicine
mapping with purchase-interval tracking and **SMS reminders**; strip and loose through an **I/U (units per item)** value; rack, shelf and box
location; 1D, 2D and QS barcodes; purchase invoice import from Excel; min/max reorder with supplier-item mapping and a warehouse that
consolidates store indents into one purchase order; inter-store transfer, short expiry from all stores sent to a warehouse; a mobile
billing app; a branded online-ordering app with prescription upload and delivery management; transaction logging for audit.
*User voice (a search summary of Capterra reviews, a general-retail sample with little pharmacy content) [Ws S39]:* easy to learn, a wide feature set, fair pricing; the commonest
complaint is again support from the vendor's centre (one 2024 reviewer gave it 1 of 5), with good help reported through partners. The same shape as Marg's, at lower volume.

### 2.5 BUSY

[W S15] Batch with manufacturing and expiry dates, search by batch, **sale restriction on expired batches**, a short-life alert; MRP captured
on purchase; discount on MRP and tax-inclusive or exclusive billing; strip, pack, piece and box units; search by salt, chemical name or
alias; patient and doctor recording, prescription attachment and patient purchase history; reorder and dead-stock reports; UPI QR on bills and
WhatsApp sharing; GST depth (date-wise rates, GSTR-1, 3B, 4 and 9 dashboards, e-invoice, e-way bill). **Not named on the pages read:** FEFO
or FIFO allocation, and Schedule H, H1 or X handling (a "Maintain Drug Register" line exists but the page does not say what it covers). Busy is
the product a Tally-like accountant already trusts and its pharmacy depth is the thinnest of the specialists on the evidence read.
*Re-read on 25 Oct (the pharmacy billing and the pharma wholesale pages [W S40]):* the same set, and a little more. Near-expiry items are "prioritised for sale", which implies
expiry-first selling without naming FEFO; there is salesman-wise billing with commission by salesman or broker, automatic payment reminders, a free WhatsApp integration and a
free mobile app (Lite) for the first year; the page for distributors names e-invoice and e-way bill. Still **not named:** Schedule H, H1 or X handling, substitutes,
free-quantity offers (schemes and discounts are), PTR or PTS, a party's drug licence number, cheque printing and price lists. Pricing is a link to a section I could not read.

### 2.6 Medeil and the newer cloud tools

*Medeil* [W S16; Ws S16]: sales, inventory, finance and CRM; expiry and banned-drug alerts, digital prescription capture, a drug index with salt
compositions and interactions, barcode and QR, a cloud HQ view for chains, purchase orders from stock alerts. The deployment and price facts
conflict on its own pages. A competitor calls it hospital-pharmacy-first [W S19].

*eVitalRx, Vyapar, Accountune, Commmerce* [W S5, S19; Ws S19]: cloud-first, cheaper, simpler; each is described by a rival, so the
weaknesses named (offline, accounting depth, batch depth) are the rival's case. They matter as proof that a cloud-first chemist product
exists and is priced in public.

### 2.7 General ERPs: Zoho Inventory and Odoo

*Zoho Inventory* [W S17]: batch tracking records manufacture and expiry dates and a batch reference, traces a defect to its batch, has batch
reports, is plan-gated, and its help page describes **manual** batch selection on sales (no FIFO or FEFO logic named). A partner's case study of
a 17-outlet Indian pharmacy chain describes FEFO dispatch rules, a rolling 90-day near-expiry alert and a batch return that drafts a debit note;
the partner also says Schedule H-style controls were custom builds in Zoho CRM and Creator [Ws S17].
*Odoo* [W S18; Ws S18]: lots with expiration, removal, best-before and alert times, a **FEFO removal strategy**, an expiry alert on a lot,
traceability; India accounting, e-invoice, e-way bill and GST-return modules; a controlled-drug register is a third-party custom model.
**Reading:** a general ERP gets batch, expiry and FEFO right and leaves the pharmacy rules (schedule capture, registers, loose sale, MRP
inclusive pricing, schemes) to an implementer. That is the position Comsky will be in on day one if it only ships the batch spec, and it is
the position to leave quickly [J].

### 2.8 Capability matrix

Y = a page I read lists it (a vendor claim). - = not found on the pages read (not proof of absence). Shaded judgement is mine.

| Capability | Marg | LOGIC | GoFrugal | BUSY | Medeil | Zoho Inv | Odoo |
|---|---|---|---|---|---|---|---|
| Batch, mfg and expiry on stock | Y | Y | Y | Y | Y | Y (plan-gated) | Y |
| Expired sale blocked or flagged | flag | Y | Y | Y | alert | - | - |
| FEFO or FIFO suggestion on sale | - | Y | - | implied ("prioritises near-expiry") | - | manual pick | FEFO |
| MRP kept per batch | - | Y | - | Y | - | - | - |
| Strip and loose | Y | Y | Y | Y | - | - | - |
| Salt search and same-salt substitute | Y | Y | Y | search only | Y | - | - |
| Schedule H, H1, narcotic indication and report | Y | Y (blocks) | Y (+NDPS report, lock after print) | unclear | alerts | custom | custom |
| Doctor and patient on bill | Y | Y | Y | Y | Y | - | - |
| Prescription image attached | Y | Y | claimed | Y | Y | - | - |
| Free-quantity scheme | Y | Y (batch-wise) | - | schemes and trade discounts (free quantity not named) | - | - | - |
| Supplier return of expiry stock | Y | Y | Y | Y | - | partner | - |
| Auto reorder | Y | Y | Y | Y | Y | - | - |
| Purchase import from supplier file or photo | Y | Y (Excel) | Y (Excel) | - | - | - | - |
| e-invoice from the product | Y | Y | - | Y | - | - | Y |
| WhatsApp bill and reminders | Y | - | SMS | Y | - | - | - |
| Mobile order and owner apps | Y | Y | Y | Y (a free Lite app for a year) | - | - | - |
| Offline counter | claimed | claimed | on-premise option | desktop | unclear | - | - |
| Published price | Y | - | - | plans | Y (conflicting) | tiers | tiers |

**What the matrix says [J]:** every specialist has the same core; the differences are in (a) how deep the scheme engine goes (LOGIC and
Marg), (b) the ordering network (Marg), (c) accounting and GST (BUSY, Marg), and (d) the compliance edge (GoFrugal's register lock, LOGIC's
blocking). **No page I read describes a pharmacy product that is keyboard-first, touch-friendly *and* has a real double-entry ledger underneath
with a synced audit trail.** That is Comsky's claim to make, and it is a claim about the whole, not about any one feature.

---

## 3. Feature by feature

Each topic gives **Market** (what the products do), **Law and data** (what binds it), **Today** (what this repo has, [C]) and **Build** ([J]).

### 3.1 Item master: composition, manufacturer ("company"), pack

**Market.** Every specialist searches by brand, salt, barcode and alias; Marg and GoFrugal match by exact salt composition **and strength**
for substitutes [W S1, S2, S14]. Marg runs PharmaNXT, a free directory it says covers 4.5 lakh to 5 lakh+ products with salts, substitutes,
MRP and the nearest distributor stocking them (its own pages give both figures) [W S1, S3]; LOGIC ships a template of 16,000+ items; GoFrugal
an "area-wise" drug database and a generic and brand index [W S11, S14]. **"Company" means the manufacturer and is a first-class dimension of
the trade**: schemes, price lists, orders, margins and claims are all organised by company [W S2, S13 (scheme groups by item, name, company or
group)].

**Law and data.** The manufacturer's name and the batch number must appear in the purchase record, the wholesale memo and, for Schedule C, H
and H1, the prescription register line [W S20 65(3)(1)(f), 65(4)(4), 65(5)]. The composition database is somebody's licensed work: the one
commercial Indian source I found (MIMS and CIMS) licenses content to software commercially and I found no public terms [Ws S33]; Tata 1mg's
public material covers order integration only [Ws S33].

**Today [C].** `StockItem` has `alias` and `partNo` (searchable in every picker), `stockGroupId`, `categoryId`, `reorderLevel`, `udf`
(`src/types.ts:484-525`); a `StockCategory` exists (`:447`) and could carry a manufacturer. No composition, no manufacturer field, no pack
field, no schedule flag. The worktree build proposes a **Pharma starter set** of *custom* fields (MRP on item, a text "Salt" on item, drug
licence number on party) and item-attribute reporting ("Profit by Brand") [C `.wt-cf/docs/design/custom-fields-spec.md` 7.5, AD.3]; its own
text says the field lists are proposals, not researched requirements. **The batch spec, written since, adds two typed tail fields to the item:** `manufacturer` (text, up to
80 characters, printed on the batch line of a scheduled item) and `schedule` (none, C, H, H1 or X, switched on by the Pharma preset) [C spec 3.1, decision T34]. It adds no
composition, no pack and no rack, and its `schedule` is a typed field, not a derived one (see A7 in 6.4).

**Build [J].** Keep the starter set as a stopgap but do **not** let a text "Salt" field be the design: it can be searched but cannot be
matched safely ("Paracetamol 500" against "Paracetamol 500 + Caffeine 30"). First-class additions: a **manufacturer** (a master, or the
stock category), a **composition** as a list of `{substance, strength, unit}` plus dosage form and route, the **pack** (3.2), a **rack**, and a
**schedule** that is *derived* (3.9). Ship an **Excel and CSV item import** (LOGIC, GoFrugal and Busy all import items from Excel [W S11, S14, S15]) and no
bundled drug database in v1. If a data licence is wanted later, ask CIMS and MIMS for written terms on caching and display; do not scrape a
consumer site [Ws S33].

### 3.2 Strip, loose, box: the pack hierarchy

**Market.** GoFrugal's **I/U (units per item)** is the plainest design on record: the item is bought and priced per strip, the sale line is
entered in units, the unit rate is the strip price divided by the I/U, landed cost is held per unit, and a pack of 10 at Rs 10 sells at Rs 1 a
unit [W S14]. LOGIC buys in boxes and sells in strips, with strip-to-tablet conversion [W S11, S12]. Busy tracks strip, pack, piece and box [W S15].

**Law.** Rule 65(19): a supply in a container other than the manufacturer's is made only by a dealer who employs a registered pharmacist,
under that pharmacist's direct supervision, in an envelope or container labelled with the drug's name, the quantity and the dealer's name and
address [W S20]. The label content that law lists does not include batch or expiry; the register line for H and H1 does (65(3)(1)(f)) [W S20].
I found no rule on how much of a strip may be sold: a Health Ministry answer of 2011 said none is prescribed [Ws S29]. Policy is moving:
a patient grievance went to the Drugs Consultative Committee, which set up a labelling sub-committee at its 66th meeting on 17 Jun 2025; the
chemists' body (AIOCD) answers that a cut strip loses its batch and expiry, cannot be returned to the stockist and leaves the shop holding
part-strips [W S29]. A news report of August 2026 says the Committee has urged exact-quantity dispensing and that chemists cite the difficulty of cutting blister packs [Ws S49];
I found no rule text and no date for a change. A claim that a loose sale may not exceed the pro-rata MRP (DPCO para 27) came from a compliance blog and is
**unverified** [Ws S29].

**Today [C].** `Unit.baseUnitId` and `conversion` make a compound unit ("Box of 12 Nos"), stock is kept in the base unit and quantities
convert on entry; `altUnitId` and `altConversion` add a reporting unit (`src/types.ts:425-436, 518-519`). Unit-suffixed entry exists
(`5 box`). That is the engine for the hierarchy.

**Build [J].** Base unit = the smallest sold unit (tablet, ml, capsule). Hierarchy box, strip, unit as compound units. **Price per strip,
derived per-unit rate** shown and rounded per line (never above the pro-rata MRP; the rounding rule is a decision to record). **Fractional
strip balance per batch** is the real design problem: an "open strip" has no purchase invoice and no reverse chain, so the stock report
should show whole strips plus loose units per batch, and a count sheet must ask for both. A loose sale **prints a small label** with the
drug, quantity, dealer name and address (the statute's three particulars, 65(19)) and, as the shop's own additions, batch, expiry and MRP per unit. Loose sale is gated
by the pharmacist-on-duty (3.9). Depends on the batch spec: batch identity must survive under the loose unit (6.4, R5). **The spec does not design this.** It allows a compound unit
under a batch item (it refuses one only for serial items, 2.2 of the spec), so the engine accepts a batch cell counted in tablets, but nothing says how a half-used strip is shown
or counted, and its pharmacy "dispense" label is Rule 94(2), which covers a medicine made up ready for treatment on a prescription (read in the PDF: label with the supplier's name and
address, the patient's name and the quantity, the prescription-register serial number and the dose), **not** a tablet cut from a strip, which is Rule 65(19)'s three particulars. Both
rules exist and a pharmacy needs both labels (6.4, F2).

### 3.3 HSN and the GST rate classes in pharma

**Market.** Every product lists HSN and GST; Busy keeps date-wise rates; LOGIC tags HSN and Schedule H together [W S11, S15]. A pharmacy's
catalogue is mixed [J, from S25]: medicines at 5% or nil, devices and diagnostics at 5%, personal care now mostly 5%, supplements and others
still 18%.

**Law [W S25, PIB release and FAQs, Annexure I read as text].** From 22 Sep 2025 for goods:

| What | Heading | Change |
|---|---|---|
| Glands and organ extracts, antisera and blood fractions, mixed medicaments not in retail packs, **medicaments put up in measured doses or retail packs** (this explicitly includes Ayurvedic, Unani, Siddha, homoeopathic and bio-chemic), dressings, the 3006 pharmaceutical goods other than contraceptives | 3001, 3002, 3003, **3004**, 3005, 3006 | 12% to 5% |
| 33 named drugs (for example onasemnogene abeparvovec, daratumumab, alectinib, evolocumab, inclisiran) | 30 | 12% to nil |
| 3 named drugs (agalsidase beta, imiglucerase, eptacog alfa) | 30 | 5% to nil |
| Six named products in an "all drugs and medicines including" row | 30 | 12% to 5% |
| Diagnostic kits and reagents, glucometer and test strips, bandages and gauze and similar | various | 12% to 5% (the Council text; row-level rates not all read) |
| Medical apparatus and devices for medical, surgical, dental or veterinary use | various | 18% to 5% (Council text) |
| Talcum and face powder, hair oil and shampoo, dental floss and toothpaste | 3304, 3305, 3306 | 18% to 5%; tooth powder 12% to 5% |

*Re-check of 25 Oct:* the PIB pages answered 403 on the second visit, so the table above is as read on 22 Oct; independent search summaries of the same changes agree on 5% for most
medicines, nil for 36 drugs (33 plus 3) and the Ayurvedic, Unani, Siddha and homoeopathic inclusion, and one marketplace guide says 37 [Ws S25]. The notification text (9/2025-Central
Tax (Rate)) was still not read. The Council gave its reason for not making medicines exempt: an exempt supplier cannot claim input tax credit, which would raise cost [W S25
FAQ]. **The rate follows the date of supply: stock already on the shelf took the new rate on 22 Sep 2025** [W S25 FAQ]. The arithmetic
favours 36 over 37 [J]. A later notification may have moved
entries: the instrument is Notification 9/2025-Central Tax (Rate) and its amendments, which I did not read in the original; rates must be
re-checked at build time and kept as dated data, never as constants [J].

**MRP is inclusive of all taxes by the label rule, and a seller may not charge above the lower of the price list and the label price** [W S27].
So the counter works backwards: taxable value = MRP divided by (1 + rate), tax = the remainder, and a discount comes off the MRP.

**The two-MRP problem [W S26].** The price regulator told manufacturers to revise MRPs from 22 Sep 2025 and to issue Form V or VI price
lists to dealers, retailers, state controllers and the government; recalling or re-stickering stock released earlier is **not** mandatory if the
price benefit is ensured at the retailer, and a CDSCO order of 11 Sep 2025 let Class C and D device importers sticker revised MRPs within three
months. So one batch label may show an old MRP with the new rate applicable. If the base price is unchanged the arithmetic is mrp / (1 + old
rate) x (1 + new rate), but **the right figure is the manufacturer's revised price list, not that formula** [J].

**Today [C].** Dated item GST history with an Alt+H grid (`core/gstRates.ts`), a rate and HSN **snapshot on each posted line**
(`src/types.ts` `InventoryLine.gstRate/taxability/hsn`), nil, exempt and non-GST separated for GSTR-1, 3B and 9 (`core/gstNil.ts`), HSN
summary and portal JSON builders. The 22 Sep change is representable. **Gap:** I found no tax-inclusive line pricing in the voucher engine
(the only "amount is tax-inclusive" path is the advance receipt, `src/types.ts:869`) and there is no MRP field
(`docs/FEATURE-PARITY.md:150`). Both are shared with the general-retail POS (garments, groceries print an MRP too). The POS code audit adds the reason and the
consequence: `core/voucherConfig.ts` records MRP as deliberately not built because it changes the stock and tax arithmetic, a rate is held to two decimals, and so a basket of
MRP-priced items cannot be made to total the sum of its MRPs, only to be tidied by the final whole-rupee round-off [C `docs/research/pos-code-audit.md` section 1.3]. **The batch spec
does not fix this either:** its `mrpCeiling` correctly adds the tax back before comparing with the MRP, but the sale line is still entered as a tax-exclusive rate (A4 in 6.4).

**Build [J].** MRP per batch (3.4); a **tax-inclusive sale line** whose base is MRP; an "MRP changed from date" action that applies a dated
revision to all batches of selected companies (it must write a separate effective price and never the printed MRP that identifies the batch: 6.4, F1); a Data Check that lists items whose HSN is 3004 but whose rate is not 5% or nil, and items on
the nil list by name. Never assume 3004 for everything on a chemist's shelf.

### 3.4 Batch, mfg and expiry, MRP, FEFO

**Market.** The universal object is the (item, batch) with manufacturing date, expiry and MRP. LOGIC tracks stock "batch-wise and MRP-wise";
Busy blocks sale of an expired batch and flags a short-life one; GoFrugal blocks short-expiry drugs at *inward* and at sale; Odoo's FEFO
removal strategy takes the lot expiring soonest first; Zoho leaves the pick manual [W S11, S14, S15, S17, S18]. Counter behaviour that matters:
the sale list shows every live batch with expiry, MRP and quantity, the default is the earliest expiry that is still sellable, a
different batch is one keystroke away, and an expired batch cannot be chosen [J, consistent with the sources; no product's UI was driven].

**Law.** A batch number is on every purchase and wholesale record and, for H and H1, on the prescription register line; nothing is sold
or stocked after the expiry of potency printed on the label (65(17)) [W S20]. **A label's expiry is a month and a year and means the drug is recommended up to the last day
of that month, introduced by the words "Expiry date"** (Rule 96, its closing explanation) [W S20, re-read 25 Oct], so a month-only expiry is saleable through that month's last day
and is expired from the next day; the batch spec stores it that way (its 3.4, rule R20).

**Today [C].** The line carries `batchName`, `mfgDate`, `expiryDate` and the engine accumulates quantity and expiry per batch
(`src/types.ts:953-955`, `src/core/inventory.ts:14, 141-159`); `StockItem.batches` switches an item to batch mode (`:520`); Stock Query lists
per-batch quantity and expiry (`src/ui/inventory/stockquery.tsx:91,140`); print can show "Batch, Exp." on a line
(`src/ui/print/invoiceModel.ts:617`). **But** `BatchPrompt` takes a typed batch name and an expiry date only: no list of existing batches, no
mfg date, no MRP, no FEFO, no expired block, and on a sale a typo creates a phantom batch (`src/ui/VoucherScreen.tsx:883-915`); the Batch &
Expiry Summary is a "coming soon" stub (`src/ui/inventory/BatchSummary.tsx:3`); the item opening has no per-batch entry
(`docs/FEATURE-PARITY.md:155`). Costing stays per item (FIFO and LIFO lots are per item, "no per-batch lots": `docs/FEATURE-PARITY.md:142` and `:165`). The
worktree's stock-variants design keeps the same stance: costing per item, a variant is a position beside the batch [C `.wt-cf` AD.3].

**Build [J].** This is the batch spec's job, and it now does most of it: a batch master derived from the voucher lines (`foldBatches`), a FEFO proposal (`proposeAllocation`) with
an expired block and a minimum-life-at-issue rule, a picker panel after Quantity, an MRP ceiling, a batch status of hold, recalled or quarantine, and a Batch Trace and Recall
report; 6.4 maps each to pharmacy's requirements and lists what is missing. It is merged as engine code that no screen calls. The pharmacy-specific pieces on top: expiry stored as **month
and year** (strips print "08/2027"; the sale cut-off is the last day of that month, which Rule 96 and the spec's R20 both settle); a company policy for the
near-expiry window and for whether a near-expiry batch needs a confirmation; a purchase-side warning when a supplier's batch arrives with
less than N days (GoFrugal's "short expiry notification" at inward [W S14]).

**Codes on packs (added 25 Oct).** A purchase entry is slowest where batch, dates and MRP are copied off a strip, so a scan that fills them is the obvious speed-up, and what is
printed on the pack decides whether it can work. The rule binds the maker: for the top 300 brands a barcode or QR must carry a product identification code, the generic and brand
names, the maker's name and address, batch, manufacturing and expiry dates and the manufacturing licence number (Schedule H2, G.S.R. 823(E) of 17 Nov 2022, from 1 Aug 2023) [W, via
`tracking-verticals-india.md` 3]. **A June 2026 amendment adds vaccines, antimicrobials, narcotic and psychotropic drugs and anti-cancer drugs** to the framework, from 1 July 2027
for the first, third and fourth and 1 July 2028 for antimicrobials, per one government broadcaster's page; it asks nothing of a retailer [W S44]. The rule fixes content, not encoding:
GS1 India's position paper, seen only as a search summary, says GS1 healthcare does not recommend QR for product identification, prefers a GS1 DataMatrix, and notes that Indian
implementations often encode a web address that leads to a verification page, not the data itself [Ws S44]. So a pack scan can fill a batch row only when the payload is a parseable GS1
string; a URL carries no batch. The batch spec says the same and fills a row only when the payload parses (its 14.4), and a one-dimensional scanner cannot read a QR at all (its 14.7,
`reads2D`). **Treat scan-to-fill on purchase as a convenience that depends on what real strips carry (V10), not as a Tier 0 requirement.**

### 3.5 Near-expiry, expired stock, returns to the supplier, expiry claims

**Market.** LOGIC: purchase returns for normal, expired and damaged stock with supplier-wise pendency, an expired and near-expiry list by
supplier [W S11]. GoFrugal: a distributor-wise return screen valuing the return at MRP, purchase rate or landed cost, a reminder, a credit note
[W S14]. Marg: pending breakage and expiry, debit and credit, replacement notes; "return before expiry" [W S1, S2]. Return windows are
set by each manufacturer or stockist, commonly about three to six months before expiry, per one accounting glossary and one reconciliation
guide, **both secondary** [Ws]; I found no authoritative short-expiry standard. A vendor's "60 to 70% recovered if caught early" is marketing
[W S14]. A Tamil Nadu rule requiring disposal within 15 days of expiry is repeated by GoFrugal and **unverified** [W S14].

**Law.** 65(17): no sale or stocking after expiry, **except** drugs for which the licensee has taken steps with the manufacturer for
withdrawal, reimbursement or disposal, which may be kept after expiry if stored apart from trade stock in packages marked "Not for sale"
[W S20]. 65(18): free physician samples and government-scheme stock marked as such are not sold or stocked (approved ESI chemists and
authorised agents are excepted with separate storage and records); a retail licence was cancelled in August 2026 partly for stocking
samples marked "not for sale" that included a psychotropic, with no invoice trail [W S20, S22].
**GST (CBIC circular 72/2018) [W S28]:** a retailer or wholesaler returning time-expired drugs through the chain may (a) treat it as a fresh
supply with an invoice (value may be the original invoice value; the recipient may take credit; a composition dealer issues a bill of supply;
an unregistered person may use any document and charge no tax), or (b) return under a **delivery challan** and the supplier may issue a
**credit note** under section 34(1); the tax liability can be adjusted only within the section 34(2) time limit and only if the returning party
did not take credit or reversed it. Sources disagree on the exact date for that limit; check section 34(2) at build time. **An expired batch that is destroyed or written off instead of returned is a different
case:** two search summaries say credit on goods that are lost, stolen, destroyed, written off, or given away as gifts or free samples is blocked by section 17(5)(h) of the CGST
Act [Ws S43]; I did not read the section, and whether it reaches the write-off of expired stock is a CA question. (One source reports that the 57th GST Council on 8 Oct 2026 proposed
lifting the restriction for free samples; it was not operative on 20 Oct, and it concerns samples, not expiry [Ws S43].)

**Today [C].** Godowns exist per item line (`InventoryLine.godownId`), debit and credit notes exist, bill-wise engine and ageing exist.
Nothing links a return to its original purchase batch. **The batch spec adds** a godown kind `quarantine` (shown as not for sale; the Pharma presets create a "Returns (not for
sale)" godown), an Expiring and Expired report whose rows carry "Return to supplier" and fill a Debit Note, a rule that expiry, hold and recall never block a return to the supplier
(its R6), a credit note that defaults to the batch sold to that party, and a transfer that keeps a batch's identity into the quarantine godown [C spec 3.5, 7.3 to 7.5, 9 rows 2 and 3,
11.5]. It does not design the delivery challan, the pending-credit tracking or the choice of valuation basis.

**Build [J].** (1) An **Expired / Not for sale** godown created with the pharmacy mode, to which an expiry action moves a batch (a Stock
Journal, so the 65(17) proviso is satisfied by a location, not a deletion). (2) A **supplier return worksheet**: pick a supplier, list its
batches inside the window, value at MRP, purchase rate or landed cost, print a **delivery challan** (the circular's route (b)), and track the
return as *pending* until the supplier's credit note arrives; a pending-claims report by supplier and age is the money screen. (3) A
**near-expiry report** by window (30, 60, 90, 120 days, set per company), by supplier, with a "return by" date. (4) A sample flag set at
purchase that blocks sale. All depend on the batch spec (R6); (1) and (3) are largely the spec's own (its quarantine godown and its Expiring and Expired report), while (2) is not designed there (A11 in 6.4) and
for (4) its batch hold status is the nearest mechanism. A fifth, new in this revision, is a batch hold from a regulator's quality alert (3.20).

### 3.6 Schemes: free quantity, discounts, bill-wise against item-wise

**Market, in detail (LOGIC's knowledge base is the clearest account found) [W S13].**
- *Batch-wise free scheme on the purchase.* Three purchase columns: sale quantity, free quantity, minimum sale quantity. At sale, when the
  same item and batch reaches the minimum, the free units come automatically (the example is 10+1 with a minimum of 50: 20 units get none, 50 get 5).
- *Manual scheme at billing.* A key on the rate column opens a window; enter "10+1"; the system derives the **adjusted net rate per unit** and
  shows the free units' value under a scheme section.
- *Group schemes.* A group names items (or a company), a customer group, dated periods that cannot overlap, a quantity or value basis, a
  minimum, slabs (the bill gets the highest slab reached), a free item or a discount, a scheme budget with stop, prompt or allow, and four apply
  modes (none, prompt, silent, allow choice). A rollback undoes a scheme on a saved bill.
- *Supplier side.* A scheme alert when buying from the same supplier; a supplier-scheme comparison at purchase [W S11].
- Marg lists discount and scheme management, "claim management" on the wholesale side (a reminder of claims due from manufacturers on purchases)
  and the last four deals at purchase entry [W S2, S3]. Busy records supplier schemes and trade discounts and applies scheme discounts at
  billing [W S15].
- "Bill-wise against item-wise": a scheme is a property of the *item or batch* (10+1 on this batch), of the *bill* (a cash discount on the
  total, or a slab on bill value) or of the *customer group*. I found these three scopes across the pages; I found no product page that
  names the two terms.

**Law and tax.** The vendor pages do not say how the free units are posted or taxed [W S13]. **CBIC Circular 92/11/2019-GST of 7 March 2019 does, in outline** (read through
secondary summaries; the circular's own PDF on the GST Council site would not yield text [W S43, secondary]): an offer such as "buy one, get one free" is not a supply of free goods
but two or more supplies for a single price, so tax is charged on the one price actually recovered and input credit on the goods is not reversed; free samples given with no consideration
are not a supply (unless Schedule I applies) and credit on them is blocked. That supports the **net-rate method** (spread what was paid across all units, as LOGIC's manual scheme
does), which keeps tax on the consideration. What the summaries do not say, and what stays a **CA question [unverified]**, is the rate where the free units are a different item or
a different batch from the billed ones (the circular says composite or mixed supply rules decide), and how a distributor's invoice should show a free quantity so the retailer's credit follows.

**Today [C].** Line discount % with the amount already reflecting it (`InventoryLine.discountPct`, F11 column); price levels per party or
voucher (`src/core/priceLevel.ts`). No free-quantity, no scheme master, no purchase-side scheme.

**Build [J].** Tier 0: **free quantity on the line** (paid and free units on one line, the net rate computed from what was paid, the free units shown separately and moving stock like any other unit), on
purchase *and* sale, with the scheme remembered on the purchase batch and offered at sale (LOGIC's pattern). Tier 1: dated group schemes
by company with slabs and a customer group; "last four deals" at purchase; a supplier-scheme comparison. Confirm the posting with a CA
**before** coding it. Two repo facts make the net-rate shape the only workable one today: a free line cannot be entered as a zero-amount line (Enter on a zero amount is refused with
"Amount is required!" in the item step of `VoucherScreen.tsx`, [C pos-code-audit 1.4]), and the e-invoice schema has a `FreeQty` field that Comsky does not send [C
`docs/research/tracking-verticals-india.md` 3.2]. **The batch spec names free-quantity schemes as not in its first version** (its section 18, item 11) **and the free units are real
stock that belongs to a batch**, often a different batch from the billed one (the vertical research's list of what breaks a naive model [C `tracking-verticals-india.md` 4(b)]), so
the inward grid needs a Free column and the line a `freeQty` (A5 in 6.4).

### 3.7 Margin and the price types: MRP, PTR, PTS

**Market.** Marg fixes the sale rate on MRP and the purchase rate on a margin, cross-checks purchase rate, deal, discount and cost, flags
margin loss at billing, adjusts sales by "company-wise margins"; LOGIC shows distributor and retailer margin at billing and flags a negative
margin in red; GoFrugal reports margin by generic name [W S2, S11, S14].

**What the terms mean.** MRP is the printed maximum retail price, inclusive of all taxes [W S27]. PTR (price to retailer) is the price a
retailer pays; the price regulator averages PTR across brands to set a ceiling price [W S27]. PTS (price to stockist) and the usual trade
margins between PTS, PTR and MRP are trade usage that I did not find stated on any page [unverified]. A retailer margin of 16% is used in the
ceiling-price method by one summary [Ws S27].

**Law and data [W S27; Ws S27].** For "scheduled" formulations the regulator fixes a **ceiling price** (excluding GST; manufacturers add GST only
if payable on it); revised ceilings for 767 scheduled formulations took effect on 1 Apr 2026 (S.O. 1575(E) of 25 Mar 2026); a dealer must keep
the manufacturer's price list on show in a conspicuous part of the premises (para 24(4)), and the regulator has said "premises" includes a
website or app; a June 2026 amendment narrowed manufacturer liability for overcharging at the retail end [Ws]. **Ceiling prices change each
April.** Discounts a shop gives within the MRP are a commercial matter [W S27].

**Today [C].** Costing methods (`core/valuation.ts`), cost per item not per batch; price levels per item; no MRP, PTR or PTS fields. **In the batch spec:** a batch's MRP and an
optional per-batch selling price (`salePrice`, "the price follows the batch"), a batch purchase rate as a report column, and an MRP ceiling at the counter; **no PTR or PTS**. The
vertical research proposed PTR and PTS as the existing price levels applied per batch [C `tracking-verticals-india.md` 6.2]; the spec did not take that up (A6 in 6.4).

**Build [J].** Per-batch **MRP and PTR** (PTS optional for wholesale); sale price defaults to MRP; a **"price above MRP" hard stop** and
"below cost" soft stop (the second needs batch cost: R4); a retail margin shown on the line for the cashier role only; a manufacturer-level
price rule ("PTR = MRP less x%") for items without their own. Keep ceiling prices as an optional dated field with a source note and a
Data Check ("MRP above ceiling plus GST"), not as something the product claims to enforce.

### 3.8 Doctor and patient registers, prescription capture

**Market.** Marg: doctor, patient, salesman and operator-wise sales; prescription-wise sale with pending reminders; prescription photos attached
to the bill and kept on the vendor's servers; doctor account with MR incentives and commission; a family ledger [W S1, S2]. LOGIC: patient
management, prescription image at billing, stored patient details recalled at billing [W S11, S12]. GoFrugal: doctor-wise sales, doctor
commission by slab, customer-to-medicine mapping [W S14]. Busy: patient and doctor recording, prescription attachment [W S15]. Medeil: digital
prescription capture [W S16].

**Law [W S20].** Prescription (65(10)): in writing, signed with the prescriber's usual signature and dated, the patient's name and address,
the total quantity to be supplied and the dose. Register (65(3)(1)): for any drug *other than Schedule X* supplied on a practitioner's
prescription, the supply is recorded at the time in a prescription register and the **entry's serial number is written on the prescription**;
the register line holds serial number, date, prescriber's name and address, patient's name and address, drug and quantity, and for
Schedule C, H and H1 the manufacturer, batch number and expiry date, and the pharmacist's signature. **A serially numbered cash or credit memo
book may serve as the register** for drugs supplied in the original container, if that option was stated in writing to the licensing authority
when the licence was applied for (65(3)(2)). A repeat of a prescription may refer back to the earlier entry. At dispensing the seller notes
name, address and date on the prescription itself (65(11)(c)). The pharmacy's supply on prescription is only by or under a registered
pharmacist's personal supervision (65(2)). The Pharmacy Practice Regulations 2015 add a five-year patient-record duty per one secondary summary
and are adopted state by state [Ws S34]. **Re-read on 25 Oct in the PDF, three narrowing details:** the Schedule H1 register line (65(3)(1)(h)) asks for the prescriber's name and
address, the **name** (not the address) of the patient, the drug and the quantity, kept three years; the register is excused for drugs supplied on an ESI prescription that already
carries every particular, and **for a drug outside Schedules C, H and H1 supplied in the manufacturer's unopened container if the prescription is stamped at supply with the
supplier's name and the date** (the third proviso to 65(3)(1), items (i) and (ii), the second on condition that the retail-memo rule 65(4)(3) is met); and a retail supply *without* a prescription of a Schedule C drug (the biologicals) still needs a
register or memo entry with the manufacturer, batch and expiry (65(4)(1)) [W S20]. So the capture gate belongs on Schedules C, H and H1 lines, with the register for the rest
optional beyond the stamp.

**Electronic prescriptions and patient data, researched this pass but not settled.** *E-prescription:* the Telemedicine Practice Guidelines of 2020 expect the doctor to give the
patient a copy of the prescription and, if the doctor sends it straight to a pharmacy, to obtain the patient's explicit consent, which lets the patient fill it at any pharmacy;
they bar teleconsultation prescribing from a prohibited list that includes Schedule X and NDPS drugs [Ws S47]. Whether an e-prescription satisfies Rule 65(10) (in writing, signed with the
prescriber's usual signature, dated) is argued both ways in commentary from 2016 to 2018, and the 2026 drafts say nothing on it [Ws S47]. **I treat it as unsettled
[unverified]: the product must let a pharmacist attach any image or file, and must never state that an e-prescription is valid.** *DPDP:* the Digital Personal Data Protection
Rules, 2025 (G.S.R. 846(E), 13 Nov 2025; one source says 14 Nov) came into force in parts: the Board's rules at once, consent-manager rules after 12 months (13 Nov 2026) and **the duties that bind a
shop (notice, security safeguards, breach intimation, retention and erasure, rights) after 18 months (about 13 May 2027)**; Rule 6 lists encryption, access control, logging and
backups, and Rule 8 keeps logs for at least a year [W S42, secondary]. I did not find a relaxation for small businesses or a health-specific rule in what I read; none of it has
started to bind a chemist yet, and all of it will bind a product that stores patient names, diagnoses and prescription images.

**Today [C].** Party ledgers with GST details, credit limit and price level; a CRM and HR module; nothing for prescriber, patient or
prescription, and no staff roles beyond the operator model. **The POS code audit's warning applies to patients:** a customer who is a ledger enlarges the ledger list that is copied and
re-sorted on every change and that the Day Book searches row by row (436 ms at 250,000 rows with about 500 ledgers), and a chemist's walk-in patients run to tens of thousands
[C pos-code-audit 1.6; J].

**Build [J].** Doctor and Patient are **not ledgers**: keep them as pick-lists *derived from the vouchers' own `rx` records* (the way ghost recall derives from the ledger's history),
keyed on a phone number or registration number so a mis-spelt name is not a second person, and promote a doctor or patient to a stored master only when something needs one (a
doctor's registration number, a family link). A credit customer stays a ledger. **Store the register fields on the voucher** (`rx: { prescriber, patient, address, rxNo, rxDate, times, interval, image? }`) and
render the registers as *reports over vouchers*, so there is one source of truth; seal them at save and let a later edit leave a dated
amendment (the outbox already records who changed what); an administrator unlock after print is logged (GoFrugal's lock-after-print is
the precedent [W S14]). Keep the prescription image local to the device and the company, off any public link, with a retention setting.
Capture gates on the *schedule*, not on every bill (3.9).

### 3.9 Schedules H, H1, X and narcotics: what the law asks of the counter

**The rules [W S20, read in the CDSCO-hosted consolidated PDF; numbering cross-checked against a High Court judgment of 18 Aug 2026 that cites
65(3)(1), 65(4)(1), 65(4)(4), 65(6) and 65(18) [W S22]].** A paraphrase of Rule 65 as it bears on software:

| Clause | Duty | What a pharmacy mode must do |
|---|---|---|
| 65(2) | Supply on a practitioner's prescription only by or under the personal supervision of a registered pharmacist | A **pharmacist on duty** (name, registration number) on the bill; sign-in per shift |
| 65(3)(1) | Prescription register for any non-Schedule-X drug supplied on prescription; serial number on the prescription; fields (a) to (g) | Capture, serial series, print in those columns; batch, expiry and manufacturer for C, H, H1 |
| 65(3)(1)(h) | **Schedule H1 supply goes in a separate register at the time of supply**: prescriber's name and address, patient's name, drug, quantity; **kept three years**, open to inspection (the government register format also carries a patient registration number and date [W S21]) | A separate H1 report and a three-year retention that nothing prunes |
| 65(3)(2) | Choice of register or memo book declared to the licensing authority at licence time | A company setting that records which was declared |
| 65(3), provisos | The register is excused for an ESI prescription that carries every particular, and for a drug **outside** Schedules C, H and H1 supplied in the manufacturer's unopened container if the prescription is stamped at supply with the supplier's name and the date (read 25 Oct) | The capture gate keys on Schedules C, H and H1 lines; for the rest a "stamp the prescription" prompt, not a register line |
| 65(4)(1) | A retail supply **without** a prescription of a Schedule C drug (the biologicals) is recorded in a register or memo with the manufacturer, batch and expiry (read 25 Oct) | A Schedule C item needs manufacturer, batch and expiry even on an over-the-counter bill |
| 65(4)(3) | Every retail supply by a cash or credit memo with the dealer's **name, address and sale licence number**, a **serial number**, drug name and quantity; copies kept | Licence number mandatory on the receipt in pharmacy mode; serial numbers gap-free per series |
| 65(4)(4) | Purchase records: date, supplier name, address and licence number, drug, quantity, **batch number**, manufacturer; purchase bills serially numbered and filed in date order | A purchase screen that cannot save a stock line without batch and manufacturer; a chronological register |
| 65(5) | Wholesale memo: date, buyer's name, address and **sale-licence number**, drug, quantity and **batch number**, manufacturer, a competent person's signature; **copies kept three years** | Buyer's DL number and validity on the party; three-year retention |
| 65(6), (7) | Produce all registers on demand; keep records at least two years from the last entry unless a rule says longer | Export on demand; retention floor per register |
| 65(9)(a) | Schedule H, H1 and X **only on and in accordance with** a practitioner's prescription; **X prescriptions in duplicate, one copy kept two years** | Hard gate on the line |
| 65(9)(b) | Supply of H, H1, X to practitioners, hospitals, dispensaries and nursing homes only against a **signed written order kept two years** | An order reference and image on wholesale sales |
| 65(11) | H, H1 and X prescriptions **not dispensed more than once unless the prescriber says so**; follow any stated number of times or interval; note seller and date on the prescription | Track "times allowed" and "interval"; **no automatic refill nudge** |
| **65(11A)** | **No other preparation, containing the same substance or not, may be supplied in lieu** on such a prescription | **No substitution offer on a scheduled prescription line** |
| 65(12) | Schedule X stored locked or in a separate part with access limited | A storage note, not software |
| 65(17) | No sale or stock after expiry; segregated "Not for sale" stock pending return | Expired location (3.5); expired block |
| 65(18) | No free samples or government-scheme-marked stock (a physician's sample carries the words "Physician's sample, not to be sold" overprinted, Rule 96(1)(ix), read 25 Oct) | A sample flag blocking sale |
| 65(19) | Loose supply under a pharmacist in a labelled container | 3.2 |
| 65(21) | Schedule X register: bound, serially page-numbered, a page per drug; date, quantity received and supplier and licence, drug, quantity supplied, manufacturer, batch, patient, prescription reference, bill number and date, signature | A Schedule X report with a page per drug; rare in a chemist's shop, so Tier 2 |

**The lists are data.** In the CDSCO copy I read, Schedule H has **551 numbered entries**, with a note that the **salts, esters, derivatives
and preparations containing those substances** are covered, excluding topical or external preparations (except ophthalmic, ear and nose
preparations with antibiotics or steroids), and a separate note on steroid and hydroquinone topicals; **Schedule H1 has 48 entries**
(from alprazolam to tapentadol; Oxytocin and Tapentadol are late additions) [W S20]. That copy predates later changes. **Pregabalin was added to Schedule
H1 as entry 51 by the Drugs (Second Amendment) Rules, 2026, G.S.R. 377(E) dated 13 May 2026 (one report gives 20 May as the Gazette date), after a draft of 21 January 2026 (G.S.R.
54(E)); the draft's three months became 180 days after publication, so about 16 November 2026** [Ws S37]. Entry 51 implies two more additions (49 and 50) beyond the 48 in my copy,
which I did not identify; reports differ on dates and one page lists an enforcement date of 13 May, so the Gazette text must be read before anything is shipped. The labels carry
the signal too: Rx, NRx (a drug that is also under the NDPS Act)
and XRx symbols and boxed warnings are prescribed by the labelling rule [W S20 rule 97]. **So the flag must be derived from the composition
(3.1) against a dated list, with the note's salt and route logic, not stored as a hand-ticked checkbox per product.**

**Narcotics and psychotropics.** Marg reports on Narcotics, H, H1 and "TB" items; GoFrugal has an NDPS report and the register lock [W S2, S14].
The NHSRC register format I read gives the H1 register and, separately, **NDPS Forms 3D, 3E and 3H for essential narcotic drugs in
*healthcare facilities*** (two-year retention) [W S21]. **What the NDPS Rules ask of a retail seller, as far as I could read them
(second pass):** the only text I could read was a case-law site's copy of the NDPS Rules 1985, summarised by the fetch tool, so this is a weaker read than the Drugs Rules PDF. It says,
in its Rule 65-A, that no one may sell, buy, consume or use a psychotropic substance except as the Drugs Rules allow, and, in Rule 67, that a consignment of psychotropic
substances needs a Form 6 note unless it is covered by a signed bill, invoice or cash memo showing the consignor's and consignee's names, addresses and licence numbers, the
description, **batch**, quantity and transport details, kept two years (a practitioner, hospital or research body consignee need not give a licence number); a chemist who handles
*essential narcotic drugs* needs a licence under the State's rules (Rule 52-B in that copy) [W S46, secondary]. I read that as sending a retailer's records back to Rule 65, which is
how a Delhi sessions case and the 2026 High Court judgment treated them [Ws S46]; **I did not find an NDPS register that a retail chemist keeps in addition [unverified]**. For a
wholesaler it adds something concrete: an invoice for a psychotropic drug must carry both licence numbers, the batch and the transport particulars (the dispatch panel and the
party licence field of A10 would carry them). What is certain is enforcement: the August 2026 High Court case upheld cancelling two retail licences where a shop held
psychotropic physician samples, could not show invoices or register entries and kept no H1 register [W S22].

**Draft rules in their comment windows (both reported, neither law) [W S23, S24].**
- **G.S.R. 791(E), 8 Sep 2026:** a new Rule 65(2A): supply of prescription drugs other than by wholesale only under a CCTV system installed and
  maintained at the licensed premises, with recordings kept at least three months. It mentions no billing or software records.
- **G.S.R. 861(E), 28 Sep 2026:** a new Rule 65(22): retailers, wholesalers and distributors may not advertise Schedule H, H1 or X drugs
  without prior approval of the Central Government. This reaches **marketing messages, push-sale suggestions and loyalty offers that name a
  prescription product** (3.17 and 3.18). The same sentence already binds **manufacturers**: Rule 74(p) of the PDF says no advertisement of Schedule H, H1 or X drugs without the
  Central Government's previous sanction [W S20, re-read 25 Oct]; the draft extends it to sellers.
- A UP drugs administration directive for retail pharmacies to publish their own website with essential information (licences, stock and
  prices in one report) is reported with an effective date of 1 Nov 2026 in one report; both reports are paywalled and partial [W S32]. A
  public "stock and price" export may become a requirement in some states [J].

**Is a software printout a register?** Rule 65 is written for paper: a register or a serially numbered memo book, carbon copies, signatures.
Vendors print registers from software and lock them afterward [W S14]; whether every state's inspectors accept that is state practice
[unverified]. The safe design is a stable, serial, printable format in the rule's own columns with an immutable history, which is also what
the repo's outbox gives.

**Today [C].** Nothing: no schedule, no prescriber, no register. The repo's "master form" and "never print legal wording the owner did not
choose" rules apply (CLAUDE.md Phase 20-21): licence numbers print only when entered, with a warn-only shape check like `core/companyIds.ts`. **That file holds only the
CIN, Udyam and IEC lines** (`sellerIdLines`, `src/core/companyIds.ts:80`): there is **no field for the shop's drug licence numbers (Forms 20 and 21, or 20B and 21B), an FSSAI number or
the pharmacist's registration**, so the licence number Rule 65(4)(3) wants on every retail memo has no home yet. **The batch spec** adds the maker and the schedule to the item,
prints the manufacturer on a scheduled item's batch line and **says in terms that the invoice is not the Rule 65(3) record** (its T34, question 26 and section 18) [C].

**Build [J].** In order: the dated schedule list (editable, with a source and date shown on screen, a Data Check for "H flag from list
dated more than 12 months ago"); the derivation by composition (the spec's typed `schedule` field then holds the derived result, with a reasoned override; A7 in 6.4); the capture gate (a bill with a Schedule H or H1 line cannot be saved without
prescriber, patient and pharmacist on duty; X also needs the duplicate-prescription reference); the registers as reports with a retention
floor and a lock-after-print; Schedule X after the rest.

---

### 3.10 Substitute and alternate lookup by composition

**Market.** Marg suggests substitutes "with the exact salt composition and strength" at billing and sells the idea hard: a free directory
(PharmaNXT), a claim of "40 to 50% of sales" found through substitutes, and **FLIP, a scheme that pays the chemist higher incentives on
substitute sales** [W S1, S2]. GoFrugal recommends substitutes when a customer wants a cheaper drug or the prescribed one is out [W S14]; LOGIC
lists salt-based substitutes at billing [W S11]. All are vendor claims.

**Law.** **65(11A): on a prescription containing a Schedule H, H1 or X substance no other preparation may be supplied in lieu, whether it
has the same substance or not** [W S20]. A summary of the Pharmacy Practice Regulations 2015 says a pharmacist dispenses what is prescribed
and does not substitute [Ws S34], and the regulations apply state by state. For an OTC product, or a customer asking for a cheaper
equivalent without a prescription, I found **no rule either way [unverified]**. A list of 344 banned fixed-dose combinations from 2016 exists
and the current list must be taken from CDSCO [Ws S33].

**Today [C].** Alias and part-number search only.

**Build [J].** Three uses, three behaviours. (a) *Same-salt alternatives for stock the shop holds*, from the shop's own item master (with
batch, expiry, MRP and what is in stock), matched on **identical substances, strengths, dosage form and route**; sorted by **price to the
customer, ascending, with no incentive sort**; the choice is logged on the line (who, what was replaced). (b) On a **scheduled prescription
line** the panel is replaced by a one-line message ("No substitution on this prescription") and an action "ask the prescriber" that starts a
fresh prescription line. (c) An out-of-stock capture (3.12). **Do not build an incentive-ranked substitute programme in v1**; it puts the
pharmacist's margin between the patient and the prescriber's choice, and the product would be the one asking for it. A list of banned
combinations is a blocker flag, not a suggestion filter.

### 3.11 Drug interaction data

**Market.** Medeil's page lists a drug index with salt compositions and interactions; LOGIC's own summary marks "prescription dosage and
duplicate-drug alerts" as claimed; Marg and GoFrugal claim none [W S11, S16]. The commercial sources are licensed: the MIMS product has a
commercial web-service API and an interaction checker restricted to healthcare professionals, CIMS says it can be mapped to a formulary
[Ws S33]. No public interaction API from an Indian source turned up.

**Judgement [J].** An interaction check that says nothing is read as "safe". The first false all-clear is the shop's liability and the
vendor's reputation. **Not in v1.** In Tier 1, a **duplicate-therapy check** (the same substance twice on one bill) comes free from the
composition data without any licence. If a licensed feed is ever added: show its name and date, say "no data" rather than "no interaction",
require a pharmacist acknowledgement, and label the screen as decision support.

### 3.12 Supplier ordering: reorder, shortages, promise orders, purchase import

**Market.** Marg: reorder from sale, minimum, manual, today's sale or zero stock with automatic purchase generation; supplier-wise company lists and
**day-wise ordering**; a supplier-wise order limit; last purchase deals shown at ordering; a central order system for chains; **Promise Order**
(save an unavailable medicine with the customer's need and fulfil later); **photo or PDF of a supplier bill turned into a purchase**; and
**ERP-to-ERP ordering** [W S1, S2, S3]. LOGIC: **out-of-stock medicines recorded at billing for supplier-wise purchase orders**, min/max
orders, Excel purchase import [W S11]. GoFrugal: supplier-item mapping, supplier-wise planning, store indents consolidated at a warehouse,
Excel purchase import, non-moving flags [W S14].

**Today [C].** `reorderLevel` per item and a Reorder Status report (`src/ui/inventory/reorder.tsx`); purchase orders and pending-order
tracking in the inventory engine; **ghost recall**, which re-derives the shape of a party's last voucher against today's masters
(`src/core/recall.ts`), a seed for "repeat last order"; WhatsApp click-to-chat sharing with the message prefilled and the person pressing
Send (`src/core/share.ts:1-4`). Not present: sales-velocity suggestions, supplier-item mapping, promise orders, purchase import.

**Build [J].** Tier 1, in value order: (1) a **short-list at the counter**, one key adds the item and the customer's phone to a list that feeds
reorder; (2) **reorder suggestions from sales velocity** over a chosen window with a minimum, maximum and lead time, computed once over the
voucher log in the manner of `core/dashIndex.ts`; (3) the **last four deals** panel at purchase entry; (4) **purchase import from the
supplier's Excel or CSV** with a remembered item-mapping, then photo or PDF later (see `docs/research/attachments-drives.md`); (5) a PO
sent as PDF, Excel or a prefilled WhatsApp. **ERP-to-ERP ordering is a network moat that one vendor cannot copy alone**; a WhatsApp or
file exchange is the honest stand-in and costs nothing.

### 3.13 E-invoicing

**Law [W S30].** E-invoicing applies where aggregate turnover under one PAN exceeded Rs 5 crore in any year since 2017-18 (Notification
10/2023-Central Tax, from 1 Aug 2023); it covers B2B invoices, supplies to registered government recipients, exports, SEZ and deemed exports
and their credit and debit notes, and **not** B2C invoices, bills of supply or delivery challans; from 1 Apr 2025 a taxpayer at Rs 10 crore
or more cannot report an invoice more than 30 days old (GSTN advisory of 5 Nov 2024). So **a chemist's retail counter is mostly outside
e-invoicing, and a distributor selling to registered chemists is inside it** once over the limit. Marg, LOGIC and BUSY say they generate it
from the product [W S3, S11, S15].

**Today [C].** E-invoice records exist (Dexie v8) but **no IRN is generated here**: the portal's result is imported and the QR printed, and there
is no IRP or GSP call (CLAUDE.md, "Known limitations"; the GSP research is `docs/research/einvoice-gsp.md`).

**Judgement [J].** This is a **blocker for the distributor segment**, not for the chemist: a stockist above the limit would run a portal or a
GSP beside Comsky. It is not pharmacy-specific work; it belongs to the GSP track. Do not promise e-invoicing to wholesalers before then.

**Two schema facts that bite a pharma wholesaler [C `docs/research/tracking-verticals-india.md` 3.2, read from NIC's sandbox page; I did not re-read it].** The item's batch name must be 3 to
20 characters (a shorter one is left out of the IRN), and the printed date pattern for the batch expiry admits only the years 2010 to 2029. A drug's labelled life may be up to 60
months from manufacture unless the licensing authority extends it (Rule 96(1)(vii) [W S20, read 25 Oct]), so a batch made from the second half of 2025 can already expire in 2030 or
later. The batch spec's answer is to omit `ExpDt` above 2029 and say so on the e-invoice preview, never to send a made-up date (its 8.1, decision T16); whether the live IRP
enforces the pattern is untested, so a wholesaler above the threshold should see that note before relying on it. The spec's line grouping (one invoice line per item and batch cell) also suits
pharma, where a 50-line purchase is the daily unit of work.

### 3.14 Pharma wholesale: salesman, area, route, outstanding follow-up, discounting, cheques, price list by company

**Market [W S2, S3, S11, S12].** *Salesman and route:* sales by salesman or MR, market, party, company and division; area-wise billing (BUSY: bills by salesman, commission by salesman or broker [W S40]); field-order
apps; a map of nearby retailers filtered by date, route, company and salesman; mobile picking; dispatch and delivery management.
*Outstanding follow-up:* party-wise outstanding, ageing and follow-up tracking, overdue reminders, a billing block for defaulters; Marg Wallet
sends payment links and automatic WhatsApp reminders. *Credit:* temporary and permanent limits, party-wise bill limits and payment terms.
*Cheques:* printing and post-dated cheques (LOGIC and Marg). *Price lists:* several price lists, party- and supplier-wise defaults for discount,
scheme and rate, margins by company. *Claims:* reminders for claims due from manufacturers. *Push sale:* suggest a near-expiry or dead item to
the retailer likely to buy it.
**"Bill discounting"** is ambiguous in the owner's list: it can mean a trade or cash discount on the invoice, or invoice financing through
a bank or partner. Marg promotes partner loans [W S3]. **Searched on 25 Oct:** I found no bill-discounting module in Marg's pages or on the review sites; the hits for the phrase were invoice-financing
providers' pages (KredX is one) selling it to pharma companies as working-capital finance, not a feature of any ERP [Ws S51]. The nearest Marg features are "purchase and sale claim
management" and credit-limit control. **I read it as the trade meaning (a cash discount for early payment, carried on the price list and the bill)** unless the owner meant financing,
which I did not research [unverified].

**Today [C].** Bill-wise outstanding and ageing (`core/billwise.ts`) and a WhatsApp share of a party's outstanding bills (`core/share.ts`);
credit limits with a reasoned override and an exceeded report (`core/creditLimit.ts`); price levels (`core/priceLevel.ts`); cheque printing
with layouts, books and crossing and a PDC register; a dispatch and e-way bill panel (`core/dispatch.ts`, `core/ewb.ts`). **There is no
salesperson on a voucher, no beat or visit, no targets**, five objects the mobile research lists as missing [C
`docs/research/mobile-personas-and-jobs.md` section 0, point 3]; the worktree's *Distributor* starter set proposes Zone, Route and Salesman as
custom fields as a stopgap [C `.wt-cf/docs/design/custom-fields-spec.md` 7.5].

**Build [J].** Most of the wholesale back office already exists in the accounting product, which is Comsky's real advantage over Marg's
accounts. Gaps in value order: a **party drug licence number and validity** (block or warn on a scheduled line to a party whose licence is
missing or lapsed, since the wholesale memo carries the buyer's licence number [W S20 65(5)]; a Ksolves case study shows a customer asking for
exactly this, and it is a vendor claim [W S19]); **salesman and route as real dimensions** (the mobile track decides); a **claims tracker** for
manufacturer claims; a **promise-to-pay log** on outstanding follow-up; a **temporary credit limit with an end date**; manufacturer-level price
rules (3.7). Cheque, PDC and ageing need nothing.

### 3.15 The counter: speed workflows

**What the vendors claim [W S1, S2, S6, S11, S14].** 40 to 50% faster billing, "under 10 seconds" on a mobile bill (GoFrugal), touch-screen
billing, barcode and strip-code scanning, multiple counters, several customers parked on one PC, quick switch to purchase and back,
a "Promise Order" key, a cashier and shift system, discount blocking by category, the last rate shown on the line, a margin-loss warning.
None of these figures is independently measured.

**Design principles for a keyboard-first, touch-friendly pharmacy counter [J].**

1. **One search box** matches brand, salt, alias, barcode and rack; each result row shows the company, pack, MRP, stock and, for a
   batch-mode item, the earliest sellable batch and its expiry; Enter takes that batch, a second key opens the batch list, an expired batch
   cannot be taken.
2. **Quantity speaks the pack:** `2` strips, `5t` tablets (the unit-suffixed entry exists, `src/types.ts:425-436`), the unit rate derived.
3. **Warnings arrive on the line they concern**, in a fixed place and a fixed order: Rx gate, near-expiry, price above MRP, below cost.
4. **Park a bill** with one key and serve the next; a parked bill's lines hold their batches (requirement R2) so a parked bill cannot oversell. **No draft concept exists today** (the
   nearest things are an Optional voucher, which burns a number and drops out of the document-issue table, and a Sales Order): a held bill needs its own local store with no
   number [C pos-code-audit section 5], and **the batch spec reserves nothing**, so a parked bill must not move stock and must re-propose its batches when it resumes.
5. **The gate is conditional.** Prescriber, patient and pharmacist prompts appear only when a scheduled line is on the bill, so an OTC bill
   has no extra step (3.9). That alone is a speed feature.
6. **A short-list key** records an unavailable item and the customer's phone.
7. **Payment** is split-capable (cash, UPI, card, credit) and shows a UPI QR for the exact amount (`src/core/upi.ts`, an owner-VPA link
   that is not a merchant integration; Marg's MargPay is a registered-merchant dynamic QR [W S1]).
8. **Print and share:** a cash memo with the dealer's licence number and a serial; a WhatsApp bill as click-to-chat (`src/core/share.ts`).
   **Thermal 58 and 80 mm printing is out of scope in the app today** (`docs/research/ROADMAP-ADDENDUM.md:392`); every receipt goes through the browser's print
   dialog on A4, A5 or Letter, there is no silent print, printer list or cash-drawer kick, and the Electron bridge exposes only backup, save-as-PDF and open-a-WhatsApp-link [C
   pos-code-audit sections 7 and 8]; the general-retail POS work owns that, and the spec's label bridge is for labels, not receipts.
9. **A sale return picks the original bill's batch** (R6) and restores to it.
10. **Alt+K "hold" constants** (`VoucherScreen.tsx:2994`) already hold the pharmacist or the doctor across bills in a batch of entries;
    the hold-visible rule (a carried value must be visible, CLAUDE.md Phase 20-21) applies to a pharmacist on duty.

**Against today's code and the batch spec (added 25 Oct) [C pos-code-audit; C spec 6.2, 6.3, 14.5].**

- *Where the counter lives.* The POS audit's central finding is that the Sales voucher screen is the wrong front end for a counter (a canonical 10-line GST invoice is 115 key actions and
  four prompts come before the first item; a Sales voucher will not save without a party and a sales ledger) and that a POS needs **its own writer** that builds the same `Voucher`, as
  Bulk Entry already does [C pos-code-audit 0.2]. I agree for pharmacy, with a consequence the spec does not provide for: it builds its allocation panel inside the voucher-screen slices
  (T-2a, on `VoucherScreen.tsx`), and a pharmacy counter on its own writer must host the same panel (A2 in 6.4).
- *The panel costs one Enter per line, and it can be spared.* The spec adds a stop after Quantity for every tracked row: the panel opens and Enter accepts the pre-selected proposal. The
  audit counts nine key presses per line on today's screen (three characters and Enter for the item, two digits and Enter for the quantity, Enter on the rate, Enter on the amount) [C
  `docs/PERF.md` via pos-code-audit 1.2], so the stop is about 11% more per line and five more presses on a five-line bill [J, arithmetic]. When one batch covers the need and is
  neither short-dated nor inside a warning tier, the proposal can be taken **without opening the panel**, with the chip still showing batch, expiry and MRP (which keeps the spec's rule that
  a proposal is never silent); the panel opens on an exception or on a key (A1). The spec already does this for scans ("auto-allocate scans", default on, its 14.5).
- *Things outside the spec that sit on the counter's path.* `doAccept` asks "will go NEGATIVE ... Continue anyway?" on every bill, even on Ctrl+A [C pos-code-audit 6]; with a pick per batch
  and the spec's block on a cell going below zero for strict items, an item-level prompt is redundant at a pharmacy counter. Ghost recall keyed on a Cash party offers the last bill of *any*
  customer [C pos-code-audit 1.6], so a patient's repeat needs a recall keyed on the patient, derived from the `rx` records (3.8). There is no reprint counter or "duplicate" mark, and rights
  have no discount cap, price override, void, refund or reprint [C pos-code-audit 9]; the schedule gate's override (3.9) needs the credit limit's reasoned-override pattern. And because
  the server accepts any table and payload from any member of the tenant, **theft prevention is detective only** until the server enforces something (3.19).

**A target, as a hypothesis [J, no evidence]:** an experienced keyboard user rings a five-line mixed bill (two scheduled, one loose) in under
30 seconds with a scanner and under 60 without. Measure the current Sales voucher on the same bill first; the number to beat is ours today,
not a vendor's claim. Marg and the other specialists are not the counter's only competitor: a chemist's habit is.

### 3.16 Health insurance, TPA and panel billing

**Market.** Marg lists "cash, credit and **panel** bills and challans", flexibility in stock for **government-employee bills**, ABHA linking in
the bill, and sells health insurance as a product; LOGIC lists insurance-claim integration as a claim [W S1, S2, S4, S11]. Retail chemists mostly
serve **panels** (an employer, ESI, CGHS) on credit with a monthly statement; cashless hospital pharmacy is a different workflow.

**Law [W S20].** An ESI-scheme prescription need not be re-recorded if all the register particulars are on it (65(3)(1) proviso); approved ESI
chemists and authorised agents may stock scheme-marked drugs apart from trade stock with their own records (65(18) proviso). **TPA and the
National Health Claims Exchange flows I did not research [unverified]** (second pass: NHCX, under ABDM, is described as a standard message exchange for cashless claims, covering
eligibility, pre-authorisation, claim and payment notices between providers, insurers and TPAs; one explainer lists pharmacies among providers, the specification summaries name no
pharmacy workflow, and an early cashless OPD-medicines scheme of 2018 ran through a pharmacy company's own system, not through NHCX [Ws S48]; I found nothing on how a retail chemist
bills a TPA), and ABHA integration needs an ABDM facility registration that I did not check.

**Today [C].** Credit sale to a party ledger, bill-wise settlement, party price levels. No panel concept.

**Build [J].** Tier 1: a **panel** is a party with a billing cycle, a co-pay percentage, an "image required" flag and a monthly challan roll-up;
scheme-marked stock is a flag that blocks retail sale and keeps a separate record (65(18)). Tier 2, only after a pilot with a real panel:
TPA formats and claim submission.

### 3.17 Loyalty, refills and reminders

**Market.** Marg: card and points, family ledger, automatic prescription reminders and bill, order and payment reminders on WhatsApp; GoFrugal:
purchase-interval tracking, SMS reminders, offers, customers grouped by purchase history; Busy: patient purchase history [W S1, S2, S14, S15].

**Law and caution.** A reminder that nudges a refill of a **Schedule H, H1 or X** drug collides with 65(11) (not dispensed again unless the
prescriber says so, and then as stated) [W S20]; a message that **promotes** such a drug is what the draft 65(22) would restrict [W S24]; patient
data in a reminder list is personal data (the DPDP Rules' main duties bind from about May 2027, 3.8).

**Build [J].** Reminders are a Tier 1 feature with three rules: (1) for a scheduled item the message is "your medicine is due, please bring or
send a current prescription", never an offer, and only while the stored prescription's "times allowed" has not run out; (2) loyalty points and
offers apply to non-prescription goods; (3) each message is a click-to-chat the staff member sends (`core/share.ts`), with consent noted
on the patient. A business messaging API is not needed in v1; its cost and templates are **unverified**.

### 3.18 Delivery, online orders

**Market.** Marg: a delivery app with live tracking and route management, a courier connection, a prescription-and-order app that says it takes
no commission, a Rs 10,000 web shop; GoFrugal: a branded ordering app with prescription upload and delivery management; eVitalRx is tied to ONDC [W
S1, S2, S5, S14].

**Regulatory ground [Ws S32].** Nationally there is no enacted e-pharmacy rule, only draft rules, and a 2018 Delhi High Court order restrained
e-pharmacies without the required licences; the UP directive on retail pharmacies' websites (3.9) shows states acting alone. Whether taking a
prescription over WhatsApp and delivering is permitted in a given state is a question for counsel, not for the product [unverified].

**Build [J].** Home delivery as a **bill state** (packed, out, delivered) with a delivery person, a cash-on-delivery receipt and a map link; a
prescription image received by the shop is attached to the bill; no marketplace, aggregator or ONDC work in this track.

### 3.19 Controls: audit and theft

**Market.** Marg lists a log of bills not printed, number of prints and modifications, an operator-wise log book and cash collection, and
"control on pilferage, stock, expiry and claims" [W S2]; LOGIC a cashier and shift system and discount blocking by category [W S11]; GoFrugal
a log of every transaction and an edit lock after a schedule register is printed [W S14].

**Today [C].** The outbox stamps every write with the user (undo and the audit trail read it), a cancelled and deleted voucher register reads the
same log, an operator model with PIN hashes and rights by area exist (CLAUDE.md; `src/ui/reports/cancelled.tsx`, `src/ui/admin/AuditLog.tsx`).

**Build [J].** Tier 1: a reprint counter and a "duplicate" mark; **role caps** on discount and on price below the batch's rate; reasons on
void, return and scheduled-item override; a "modified after print" flag; the register lock (3.8); a shift close with expected against counted
cash (the general POS doc). **Do not build a way to edit the recorded margin of a bill** (the Marg line noted in 2.2). The POS audit sets the limit on all of this: rights are client-side and too coarse (a cashier
needs the Accounting area, which also opens the Day Book and cash book, and cancel counts as alter), and the server enforces nothing per voucher type, so every control above is
**detective**: an event with operator, terminal and time, a shift report that shows them and an exceptions list the owner reads; drawer opens and reprints are not data writes today
and need an event log of their own [C pos-code-audit section 9].

### 3.20 Batch safety alerts: CDSCO's monthly NSQ and spurious lists (added 25 Oct)

**What exists.** CDSCO publishes each month a list of drug samples declared Not of Standard Quality (a central-laboratory list and a separate state-laboratory list) and a list of
spurious drugs, as PDFs on its Latest Alerts page; the stated purpose is to make stakeholders aware of the NSQ batches identified in the market [W S45: one PDF, January 2025, read as
text; Ws for the page and the other months]. The table has the columns serial number, product name, **batch number, manufacturing date, expiry date, manufacturer**, the test the sample
failed and the reporting laboratory. It is a Word-made PDF of ten landscape pages whose table text extracts out of order (the cells of one page came out as columns), which is itself a
finding for anyone who would import it. News reports put the monthly counts in the tens to a few hundred samples (46 in the central labs' May 2026 list, 280 flagged NSQ or spurious in
July 2026) [Ws S45], and officials say an NSQ finding applies to the tested batch, not to all of a maker's products [Ws S45]. I found **no pharmacy product that advertises a batch
alert tied to these lists** (none of the Marg, GoFrugal or Busy pages I read mentions one; search summaries only) [Ws S45]. I did not research a retailer's legal duty on holding a
flagged batch [unverified]; nobody wants to sell one.

**Build [J].** The spec's batch hold status and its Batch Trace and Recall report are the mechanism (a hold is an overlay write with a reason, refused at sale, never at return or count).
The new piece is the **match**: a screen where a person pastes rows from an alert (product, batch, maker) or loads a cleaned CSV, and the app lists every batch in the company whose
number matches under the spec's own batch-key normalisation and whose item or maker looks alike, with the quantity held, who it was sold to and which supplier it came from; the
pharmacist ticks which to hold, and the hold records its source ("CDSCO NSQ alert, <month>"). **Never automatic and never a verdict:** batch numbers are free text and collide across
makers, so a number match is a candidate, not a finding, and the wording is "your stock has a batch with the same number as one on the alert", never "this batch is unsafe". **Do not
parse the PDF silently.** Tier 1; small to medium on top of the spec's hold and trace [J]. It is also the clearest answer to "why books software and not a billing app": the engine that
sold the batch can say who bought it.

---

## 4. Where Comsky stands against the pharmacy bar

"In the spec" is what `docs/design/batch-serial-labels-spec.md` designs (read 25 Oct); "merged" means its engine code is on `main` and no screen calls it.

| Capability | The bar (best of the specialists) | Comsky today [C] | In the batch spec | Gap [J] |
|---|---|---|---|---|
| Batch pick on sale from stock, FEFO, expired block | A list of live batches with expiry, MRP, quantity; earliest first; expired blocked | Typed batch name and expiry; no list, no FEFO, no block (`VoucherScreen.tsx:883-915`); the engines (`proposeAllocation`, `foldBatches`) are merged and unused | **Designed** (the panel, FEFO, expired block, minimum life); no rack, no parked-bill reservation | XL (the spec's T-0 to T-2) |
| MRP per batch, tax-inclusive billing | MRP on the batch; sale at or under MRP | No MRP field; no tax-inclusive line | MRP per batch in the batch's identity and a ceiling check; **no tax-inclusive sale line, no PTR or PTS, no dated revision** (F1, A4, A6) | L |
| Strip and loose | I/U-style conversion; loose label | Compound and alternate units exist | **Not designed** (a compound unit is allowed under a batch; the label is the wrong rule, F2) | M |
| Composition, manufacturer, substitutes | Salt and strength, company, same-salt lookup | `alias`, `partNo` only | The maker as text on the item; no composition | L |
| Schedule flag, capture gate, registers | Block without doctor and patient; printable registers; lock after print | None | A typed `schedule` field and the maker on the batch line; **registers and capture "not in v1"** | L |
| Licence number, pharmacist on bill | Dealer licence on the memo; pharmacist per shift | Company identity lines hold only CIN, Udyam and IEC (`companyIds.ts:80`); no licence or pharmacist field | **Silent** | M |
| Expiry return workflow | Supplier-wise return screen, credit-note tracking | Debit and credit notes exist; no link to batch | Expiring and Expired report fills a Debit Note; a quarantine godown; **no challan, no pending-credit tracking** | L |
| Free-quantity schemes | Purchase and sale; batch-wise; slabs | Line discount % only; a free line cannot be entered at zero | **Named a non-goal** | L |
| Batch alerts from a regulator's list | None found | None | Hold status and Batch Trace are the mechanism; no match screen | S to M |
| Reorder by velocity, short-list, purchase import | Yes, with supplier mapping | `reorderLevel` and a report; ghost recall | Importers for tracked lines and opening cells (T-8) | L |
| Wholesale back office | Credit, ageing, cheque, PDC, price lists | Present and strong | n/a | S |
| e-invoice for wholesalers | Generated in product | Imported only; no GSP | Batch name and expiry rules for the IRN (the 2029 limit, 3 to 20 characters) | XL (GSP track) |
| WhatsApp bills and reminders | Click or automatic | Click-to-chat for vouchers and outstanding | n/a | S |
| Panel and TPA | Panel bills; claims claimed | None | n/a | M |
| Controls and audit | Reprint log, discount caps, lock | Strong audit trail; no reprint counter, caps | Reasoned overrides on the voucher for its rules | M |
| Offline counter | Claimed | Offline-first app | Detect-and-record on sync for serial clashes; batch cells are judged per window | S |
| Accounting beneath | Variable; Marg's "billing to balance sheet" | A full double-entry engine | Batch cost is a report; valuation stays per item | an advantage |

**Two repo statements to fix [C].** `docs/parity-data.json` rows `batches` and `expiry-reports` read "covered" beside "No batch field on item or
line" and "No batch/expiry concept" (`docs/FEATURE-PARITY.md:163` and `:173`): the code does carry a batch name and expiry and Stock Query lists
them, and the Batch & Expiry Summary is a stub. Both rows need a status and a note that agree (the "a status field and its note must agree"
gotcha, CLAUDE.md Phase 20-21); the spec's decision T30 says it re-states both as partial when its work starts. I did not edit them: this task writes one document.

---

## 5. Data and regulatory traps

| # | Trap | Why it bites | Handling [J] |
|---|---|---|---|
| T1 | Schedule lists are dated data, and H extends to salts, esters and derivatives with topical carve-outs; **a list entry can have a future effective date** (pregabalin joins H1 about 16 Nov 2026) | A flat per-product checkbox goes stale and misses a salt form; a missed H1 sale is an unrecorded sale; the spec's typed `schedule` field is exactly such a checkbox | Editable dated list with source, date and an effective-from; flag derived from composition and stored with a reasoned override; Data Check on age |
| T2 | 5%, nil and 18% side by side; 3004 is not "all medicine"; the nil list is by name | Over- or under-charging tax on a mixed shelf | Dated GST history (exists); Data Check on 3004 items not at 5% or nil |
| T3 | MRP is tax-inclusive; ceiling prices move each April; two MRPs after 22 Sep 2025 | Selling above the label price is an offence for the seller as well as the maker [W S27] | Per-batch MRP; dated MRP revision action; hard stop above MRP |
| T4 | 65(11A) bars substitution on a scheduled prescription | An incentive-ranked substitute prompt invites a breach | Neutral price-sorted panel; disabled on scheduled lines; log the choice |
| T5 | 65(11): no repeat dispensing unless allowed | Refill reminders as marketing | "Times allowed" and "interval" tracked; reminder asks for a current prescription |
| T6 | Draft 65(22): no retailer advertising of H, H1, X without Central approval | Push-sale and loyalty offers naming Rx products | No promotional template for scheduled items; wait for the final text |
| T7 | Draft 65(2A) CCTV | Hardware and footage, not software | Note only; nothing built |
| T8 | Loose sale: a cut strip loses batch and expiry; no reverse chain | Fractional stock and unreturnable part-strips; label duty under 65(19) | Fractional strip balance per batch; label with batch and expiry |
| T9 | Doctor commission | Vendors ship it [W S2, S14]; the Medical Council's 2002 regulations bar a physician from receiving a commission or gift for referring or recommending a patient (6.4.1) [W S38]; the 2023 replacement regulations were suspended within weeks [Ws S38]; whether paying one is an offence of the chemist's I could not settle | **Legal review before any build**; a neutral doctor-wise sales report is fine |
| T10 | Patient and prescription data | Health data; images; staff access; DPDP Act [unverified] | Local to the company, retention setting, access by role, no public links |
| T11 | Drug database licence and the "all clear" | Licensed content; a false negative is a safety event | No bundled database; no interaction feature in v1; "no data" wording |
| T12 | Marg's ordering network | Cannot be matched feature-for-feature | File and WhatsApp exchange now; revisit with a pilot distributor |
| T13 | NDPS obligations of a retail chemist only partly read | The NDPS Rules text I could read routes sale and purchase records back to Drugs Rules 65 (Rule 65-A) and, for a psychotropic consignment, wants both licence numbers, batch and transport details on the invoice (Rule 67) [W S46, secondary]; enforcement is real [W S22] | Ask counsel; build the H1 register first; carry licence numbers and batch on wholesale invoices |
| T14 | Electronic register acceptability | Rule 65 is written for paper | Stable serial print in the rule's columns; immutable history; state practice [unverified] |
| T15 | Physician samples and scheme-marked stock | 65(18); a licence was cancelled in 2026 for it [W S22] | Sample flag blocks sale |
| T16 | Expired stock is kept apart, not deleted | 65(17) proviso | Expired location |
| T17 | Strip barcodes are not a standard; scanning gives the item, not the batch, unless a 2D code carries it **and the payload is parseable** | Marg says "scan batch numbers directly" [W S1]; the QR rule binds the maker for the top 300 brands, from 1 Aug 2023, and was extended in June 2026 to vaccines, antimicrobials, NDPS and anti-cancer drugs from July 2027 and 2028 [W S44]; it fixes content, not encoding, and GS1 says Indian QR codes often hold a web address [Ws S44]; a 1-D scanner cannot read a QR | Scan to item; batch from the list; parse a pack code only when it is GS1, say so when it is a URL (the spec's 14.4); V10 stays open |
| T18 | Marg's "adjust margins of bills" | Edits recorded margin | Not copied |
| T19 | Free goods in GST | Circular 92/11/2019-GST (secondary summaries) treats "buy one, get one free" as two supplies at one price with credit kept, and is silent on free units of a different item or batch; credit on free samples is blocked [W S43, secondary] | CA before coding; net-rate posting; free units are real stock in a named batch |
| T20 | Dated claims in this document | Several are from May to October 2026 press and drafts (pregabalin, the QR extension, two drafts, the 57th Council's sample proposal) | Re-check every item in section 3.9 and 3.3 at build time |
| T21 | **A revised MRP written through the batch overlay moves the sale off its cell (F1, 6.4)** | The cell key carries the line's MRP in paise; the batch row prefers the overlay's MRP; the picker copies the row's MRP onto the sale line | Two numbers: the printed MRP (identity, in the key) and the effective MRP (the ceiling); the key is built from the printed one only; a suite sells a batch after a revision and asserts the same cell |
| T22 | **The spec's dispense label is Rule 94(2), not the 65(19) loose-supply label (F2, 6.4)** | A cut strip would be labelled with a register serial number and a patient it may not have | A separate loose-supply label (drug, quantity, dealer name and address, plus batch, expiry and unit MRP as the shop's own addition) |
| T23 | Patients and doctors as ledgers | The ledger list is copied and re-sorted on each change and the Day Book searches it row by row; walk-in patients run to tens of thousands [C pos-code-audit 1.6] | Derive the pick-lists from the vouchers' `rx` records; promote only when needed |
| T24 | A free line cannot be saved at a zero amount | Enter on a zero amount is refused in the item step | The net-rate method plus a `freeQty` annotation; never a zero-rate line |
| T25 | Importing a regulator's NSQ list | A Word-made PDF whose table extracts out of order; batch numbers are free text and collide across makers | Paste or CSV, match as candidates, a person ticks; never automatic (3.20) |
| T26 | A 60-month shelf life against the IRP's 2029 limit | A batch made from late 2025 can expire in 2030 or later; a wrong date must never be sent | Omit `ExpDt` above 2029 with a visible note (the spec's 8.1); one sandbox test when access exists |
| T27 | DPDP Rules phase in (main duties about May 2027) | Patient names, diagnoses and prescription images are personal data; Rule 6 lists encryption, access control, logging; Rule 8 keeps logs a year [W S42, secondary] | Role-based access to `rx`, an access log and a retention setting designed in now; nothing public; no claim of compliance |
| T28 | The allocation panel's extra stop per line | About 11% more key presses per line [J] on a counter measured in seconds per bill | A no-stop pick when the proposal is unambiguous, with the chip showing it (A1) |

---

## 6. What to build: a pharmacy mode (judgement)

Everything in this section is **[J]** unless it names a source.

### 6.1 Principles

1. **A company feature switch, opt-in, and nothing changes when it is off.** Pharmacy mode is a `CompanyFeatures` flag beside the existing
   ones; with it off, no screen, field, key or default moves, and a suite asserts that by identity (the Phase 27 lesson: "nothing chosen
   must be the app as it was") [C CLAUDE.md]. With it on, new stock items default to batch mode, the near-expiry window is 90 days (the
   commonly cited three-month return window [Ws]; editable) and the scheduled-drug gate is on.
2. **A pharmacy bill is a GST Sales voucher.** No parallel store, and the same `Voucher` document, written by a counter-specific writer rather than the Sales voucher screen (3.15). The registers, the expiry reports and the substitute panel are readers of
   vouchers and masters; the schedule capture is a small sub-record on the voucher (3.8). That is what keeps books, stock and register
   reconcilable and what lets the same bill print as a tax invoice, a cash memo or a challan.
3. **Compliance data is dated, editable and carries its source.** The schedule list, the banned-combination list, the GST rate history, the
   ceiling prices: each shows "as of" and "from" on screen and a Data Check flags one that is old.
4. **One flow for keyboard and touch.** The same search, the same batch list, the same gates; touch targets are a layout, not a mode.
5. **Offline-first.** A counter must bill when the line is down; the provisional-number approach in the mobile research applies.
6. **Do not copy** (with the reason in 3.10, 3.8, 3.11, 3.17, 2.2): incentive-ranked substitution, a margin editor on bills, doctor
   commission payouts, automatic refill nudges on scheduled drugs, marketing messages that name a scheduled product, "no interaction"
   all-clears, a claimed drug database, insurance and loan resale.

### 6.2 Tiers

**Tier 0: what makes it a pharmacy.** Effort is a rough [J] scale in the repo's S/M/L/XL.

"Depends on the spec" is whether the item can work without the batch engine; "The spec designs it" is what `batch-serial-labels-spec.md` actually covers (6.4 has the detail).

| # | Item | Depends on the spec | The spec designs it | Effort |
|---|---|---|---|---|
| 1 | **Batch pick on sale from stock** (batch, expiry, MRP, quantity, rack), safe for parked bills | Fully | **Yes**, but not the rack nor a parked bill's reservation | XL (that spec) |
| 2 | **FEFO default, expired-sale block, near-expiry window and confirmation** (company policy), **and a no-stop pick when the proposal is unambiguous (A1)** | Fully | **Yes**; the no-stop pick is an ask | M on top |
| 3 | **Per-batch MRP, tax-inclusive sale line, hard stop above MRP, dated MRP revision** (the revision written as an effective price, never into the batch key: F1) | Fully | Partly: MRP in the identity and a ceiling check; no inclusive line, no dated revision | L |
| 4 | **Strip and loose**: pack hierarchy, per-unit rate, fractional strip balance, loose-supply label (65(19), not 94(2): F2) | Partly (batch identity under loose) | **No** (compatible: a compound unit is allowed under a batch) | M |
| 5 | **Batch-wise purchase entry**: batch, mfg, expiry, MRP, **free quantity**; a chronological purchase register with the supplier's licence number and the maker (65(4)(4)) | Fully | Partly: the inward grid (batch, mfg, expiry, MRP, qty); no free column, no register report | M |
| 6 | **Expired / Not for sale godown**, expiry move, **supplier return worksheet** with delivery challan and pending-credit tracking (65(17), circular 72/2018) | Partly (batch-linked return) | Partly: the godown kind and a report that fills a Debit Note; no challan, no pending credit | L |
| 7 | **Near-expiry and expired reports** by supplier and window | Fully | **Yes** (Expiry Ageing, Expiring and Expired) | S to M |
| 8 | **Dated schedule list + derived flag + capture gate** (prescriber, patient, prescription reference, pharmacist on duty) | No | **No**: a typed `schedule` field only | L |
| 9 | **Registers as reports**: prescription and H1 registers, memo particulars, lock after print, retention floors (65(3), 65(7)) | Partly (batch, expiry, manufacturer columns) | **No** (its Q26: not in v1) | L |
| 10 | **Licence numbers and the pharmacist**: company DL numbers, pharmacist on duty per shift, party DL number and validity | No | **No** (silent) | M |
| 11 | **Composition structure, manufacturer, same-salt lookup** from the shop's own master | No | Partly: the maker as text; no composition | L |
| 12 | **Free-quantity schemes** on purchase and sale (net-rate method) | Partly (batch-scoped free quantity) | **No** (a named non-goal) | L |
| 13 | **Item master import** (Excel or CSV) with batch openings, parties and outstanding | Partly (batch openings) | Partly: tracked-line and opening-by-cell importers (T-8); not salt, schedule or pack | M |
| 14 | **Price guards**: above-MRP stop, below-cost warning, margin on the line for the cashier role | Partly (batch cost, requirement R4) | Partly: the MRP ceiling and a batch-cost report; no below-cost rule, no margin on the line | M |

Counts, by the two columns: *depends* 5 fully (items 1, 2, 3, 5, 7), 6 partly (4, 6, 9, 12, 13, 14), 3 not at all (8, 10, 11); *designed by the spec* 3 yes (1, 2, 7), 6 partly (3, 5, 6, 11, 13,
14), 5 no (4, 8, 9, 10, 12).

**Tier 1: speed and money.** Parked bills; the counter short-list; reorder from sales velocity; the last four deals at purchase; purchase
import from a supplier's Excel or CSV with a remembered mapping; WhatsApp bills and reminders with the refill guards of 3.17; panel
billing; role caps on discount and price, a reprint counter and a modified-after-print flag; promise-to-pay log; salesman and route as
dimensions (the mobile track); a duplicate-therapy check; doctor-wise and patient-wise reports; delivery as a bill state; dated scheme
groups by company with slabs; a manufacturer claims tracker; batch alerts from the regulator's NSQ lists (3.20); a public stock-and-price export if a state requires it.

**Tier 2: data licences and networks.** A drug data licence and an interaction feature (3.11); a distributor ordering network or a file
exchange pilot; TPA and claim exchange; ABHA; the Schedule X register; photo or PDF purchase import; scan-to-fill a purchase from a pack's own code (3.4, only where the payload parses); marketplace or ONDC; e-invoice for
wholesalers (the GSP track, 3.13); doctor commission **only after legal review**.

**The first segment [J].** Start with the **retail chemist**. It needs no e-invoice, no routes and no salesman app, and the specialist
price it has to beat is Rs 5,550 to 26,000 one time with AMC [W S1]. The distributor segment needs the GSP, the salesman app and the party
licence control; sequence it behind the mobile and e-invoice tracks.

### 6.3 Compliance claims: what the product may and may not say

- **May say:** "Prints the prescription register in the columns Rule 65(3) lists." "Keeps Schedule H1 entries for three years and will not
  prune them." "Warns before an expired batch is sold." "Shows the schedule list's date and source." "The licensee is responsible for the
  register and for keeping the list current."
- **May not say:** "Drugs and Cosmetics Act compliant", "inspector-ready", "audit-proof", "approved by CDSCO", "legal in every state",
  "no interaction", "safe substitute", "this invoice is your prescription register" (the spec's own rule: the invoice is not the Rule 65(3) record), "a valid e-prescription"
  (unsettled, 3.8), "this batch is unsafe" for a regulator-alert match (3.20: it is a number match), "DPDP-ready" (the duties start about May 2027 and no one has audited anything here). The rules differ by state in practice and change by notification (two drafts are open as this is
  written).
- **Print only what the owner entered.** The licence number prints only when entered, with a warn-only shape check like `core/companyIds.ts`;
  the product ships no default legal wording (CLAUDE.md, Phase 20-21 gotchas).
- **Say what a gate is.** A gate that blocks is named in the sentence that blocks it ("Schedule H item: prescriber and patient are required
  by Rule 65(9) and 65(3)"), with the rule as a citation the owner can read and an administrator override logged with a reason (the credit
  limit override is the pattern, `core/creditLimit.ts`).

### 6.4 How much depends on the batch / serial spec, and what pharmacy asks of it (rewritten 2026-10-25 against the spec)

**Numbering, so nothing is confused.** In this section **R1 to R10 are this document's ten requirements** (the labels the first version gave them, which the vertical research and the
spec cite); the spec's own rules are always written "spec R11", "its R26" and so on, and its decisions T1 to T36. **A1 to A12** are asks that are new in this revision; **F1 and F2** are
findings.

**What the spec is, for a pharmacy reader [C `docs/design/batch-serial-labels-spec.md`, sections 0, 2, 3, 4.4, 5, 6.3, 7, 8, 9, 11.5, 13.5, 13.8, 14.4, 15, 17, 18].** Design only, reviewed
twice, 36 decisions, 31 rules, 30 owner questions with defaults taken, 145 to 180 builder-days across its slices T-0 to T-10 by its own table. It builds a **tracking layer**: one stock key
`(godown, batch, variant)` with a serial as a unit identity below it (T1); a batch identity of item, batch number and, where the item keeps MRP per batch, the printed MRP (T4); a batch
master derived from the voucher lines with a small stored overlay for status and corrections (3.4); FEFO proposals (T7); one rules function with severities and reasoned overrides (T8);
reports (T17); presets including "Pharma retail" and "Pharma wholesale" (11.5); labels and scanning. **Its pure engines are merged on `main` (`5aee765`) and no screen calls them.** The
integration is gated on the custom-fields lane: T-0b waits for the stock-by-field engine to be committed, reviewed and green on a rebased `main`, and today that build is one squashed
commit on its own branch (`.wt-cf`, `2517916`); I did not establish whether it has merged. The spec is written mainly for mobile and electronics (IMEI) and for batch goods; pharmacy is
one of five presets. **What pharmacy gets from it is a correct batch engine and a sale-time pick, not a pharmacy.**

**Dependency, by the Tier 0 table (6.2):** of the 14 items, 5 cannot work without the batch engine (1, 2, 3, 5, 7), 6 work only in a degraded form (4, 6, 9, 12, 13, 14) and 3 do not
depend on it (8, 10, 11). **Coverage, by the same table:** the spec designs 3 (1, 2, 7), designs 6 in part (3, 5, 6, 11, 13, 14) and does not design 5 (4, 8, 9, 10, 12).

**Pharmacy's ten requirements against the spec.**

| Ref | The spec's answer [C] | Verdict | What is missing or in the way |
|---|---|---|---|
| **R1** A batch master per item: number, expiry as month and year, MRP, optionally PTR, PTS and maker; **MRP revisable on a date** | A derived master (`foldBatches`); identity is item + number + **printed MRP** (its decision T4 and question Q11, vetoable); a month-only expiry is stored as that month's last day (3.4, spec R20), which Rule 96 confirms; the maker is on the item; the overlay can hold one corrected MRP, and a dated MRP history is "not modelled in v1" (its 18.11) | **Partial** | The identity differs from this document's first ask (unique per item and batch number): accepted, because it keeps the above-MRP check exact, **but see F1**. No PTR or PTS and no dated revision (A6). A maker on the item, not the batch, is right for a brand and wrong for a generic-named item stocked from two makers [J] |
| **R2** Sale-time pick: FEFO, one-key override, expired refused, near-expiry window, negative stock per batch | The allocation panel after Quantity, the proposal pre-selected, expired blocked (default for a new item), `warnDays` such as 90, 60, 30, a minimum life at issue, a cell below zero blocked on a strict item and warned on an upgraded one (spec R10), held and recalled batches excluded (spec R13) (its decision T7, 4.4, 6.3) | **Met** | Speed (A1), hosting on a counter writer (A2), a rack column (A9) |
| **R3** Purchase-time capture; a new batch typed on a sale is refused | The inward grid in the order of an Indian pharma purchase bill (batch, mfg, expiry, MRP, sale price, quantity), MRP proposed from the previous batch; the same number with another expiry informs (spec R12); a new batch typed on a sale is refused (spec R27; only a warning on an upgraded item) | **Met** | The free-quantity column (A5) |
| **R4** Batch purchase rate kept per batch; valuation stays per item | `purchaseRate` (quantity-weighted over the inward lines) and `valueAtCost` on every batch row, plus a "costing method difference" line (4.5); valuation per item | **Partial** | A report, not a rule: nothing warns at the counter that a price is below the batch's cost, and the rate is not landed cost. Cheap to add: put the batch rate into the pick row and add a warning (item 14) |
| **R5** Batch identity under unit conversion: whole strips plus loose, counts in both | A compound unit is allowed under a batch item (the spec refuses one only for serial modes, 2.2); its picker example reads "strip of 10"; nothing on a half-used strip | **Not designed** | Show whole strips plus loose units per cell, count by cell in both, price per strip with a derived unit rate (3.2); the loose-supply label (F2, A3) |
| **R6** Batch-linked returns: names the original batch; keeps identity into the Not-for-sale godown; a sale return restores to the same batch | A credit note defaults to the batch sold to that party and a returned lot keeps its batch (spec R5, 7.3); a supplier return starts from the Expiring and Expired report into a Debit Note, and expiry, hold and recall never block it (spec R6, 7.4); a transfer into a quarantine godown keeps the batch with its MRP, expiry and status (7.5) | **Met** | The challan and the pending-credit tracking (A11) |
| **R7** Batch-wise opening stock, carried by Split Company Data and by import | `OpeningStockAlloc.cells`, an "Opening by cell" panel, Split Company carrying cells (T-0b), an "Opening stock by cell" importer (T-8); the split already carries batches [C `docs/FEATURE-PARITY.md:470`] | **Met** | Nothing pharmacy-specific |
| **R8** Scan to item and, where a 2D code carries it, to batch; lookup by batch number; the loose-sale label | One exact-lookup resolver: unit, minted batch code, item, pack or variant code, parsed GS1 string, alias; a pack's QR fills a batch **only when its payload parses**; batch-number lookup in the picker and Batch Trace; templates `pharma-shelf-50x25` and `pharma-dispense-75x50` (14.4, 13.5) | **Partial** | The loose-supply label (F2). Pack scanning depends on a 2-D scanner and a parseable payload (3.4) |
| **R9** Batch on documents: bill lines, challans, register lines | A printed line `Batch: B · Exp. MM/YYYY · MRP`, plus `Mfr:` on a scheduled item (8.2); the e-invoice batch rules (8.1) | **Partial** | No register exists to print a batch on (A8); whether challans and debit notes share the invoice print model I did not check |
| **R10** Reports: batch-wise stock, near-expiry by supplier and window, batch movement and a recall trace | Batch-wise Stock, Expiry Ageing, Expiring and Expired, FEFO Pick List, Unbatched Stock, Batch Trace and Recall (backward and forward, with Hold, Recall and Release and a recall list), Batch Profit, and a batch-ledger drill (9) | **Met** | A regulator-alert match (A12) |

**Count: met 5 (R2, R3, R6, R7, R10), partly met 4 (R1, R4, R8, R9), not designed 1 (R5).**

**Two findings for the spec's lane, to settle before its panel is built (slice T-2a).**

*F1. A revised MRP written through the overlay moves the next sale off its cell [C: read in the merged code, then reproduced on 25 Oct by a scratch script that calls the merged functions; nothing was written in the repo].* The chain: (1) `batchKeyOf` builds a batch's cell key from the
**line's** `mrp` in paise whenever the item keeps MRP per batch (`src/core/stockCell.ts`, `batchKeyOf` and the resolver closure); (2) `foldBatches` sets a batch row's `mrp` to the
**overlay's** `mrp` when one exists (`mrp: rec?.mrp ?? a.mrp`, `src/core/batches.ts`), `pickCellsOf` passes that figure to the picker as the cell's MRP, and `splitToLines` writes it onto the
sale line (`line.mrp = info.mrp ?? cell.mrp`; the caller's `batchInfo` could supply another figure, but the panel that would is unwritten). So once a pharmacy lowers a batch's MRP in the
overlay, which is the only data route the spec leaves for a price cut **without re-stickering** ("corrected values win", 3.1; a re-stickered pack is a different printed MRP, hence a
different cell, and is handled by a transfer), the next sale of that batch carries the lower MRP and resolves to a **different cell key** from the batch's own: on a strict item the sale is blocked as below zero (spec R10) or as
a new batch (spec R27), on an upgraded item it warns and books a phantom cell. **The script's numbers:** batch B-1 received, 10 units, at a printed MRP of 22.50 with an overlay lowering
its MRP to 20.00; `splitToLines` over that batch's pick cell, with no `batchInfo` override, writes a sale line with MRP 20, and `cellOf` resolves that line to the batch part
`b-1` + MRP 2000 paise, while the 10 units sit in `b-1` + MRP 2250 paise; without the overlay both are 2250 and the sale lands on the stock. The panel could dodge this by passing a
`batchInfo` that returns the printed MRP, but the default path is the broken one and nothing yet says the panel must. There are **two MRPs in the pharma case**, the printed one that identifies the batch and the lower
effective one that caps the price (the vertical research named the second `mrpRevision` [C `tracking-verticals-india.md` 6.2]), and the engine has one field. This is not a corner case:
after 22 Sep 2025 a revised MRP on old stock is the ordinary case (3.3). **Fix:** carry both: the printed MRP stays in the key and is what `splitToLines` copies onto the line; an
effective price, with a from-date, feeds the ceiling check and the default rate; a suite sells a batch before and after a revision and asserts the same cell. It is cheap now (the
panel is unwritten) and expensive after sale lines exist.

*F2. The spec's dispense label is the wrong rule for a cut strip [W, Rule 94(2) and Rule 65(19) read in the PDF on 25 Oct; C, the merged code: `LABEL_KINDS` in `src/core/labelModel.ts:26` has `dispense` and no loose kind, `labelRules.ts:411-421` checks a dispense label against the five 94(2)(i) items, and `labelTemplates.ts:121-135` has no field for the drug's name].* Rule 94(2) exempts "a medicine made up ready for treatment"
and supplied on a prescription from the labelling Rules 96 to 101 if it is labelled with the supplier's name and address, the patient's name and the quantity, the serial number of the
prescription-register entry, and the dose for internal use (or "for external use only"), and Rule 65's condition (3) is met. Rule 65(19) is the loose-strip rule: a retail supply in a
container other than the manufacturer's goes only through a dealer who employs a registered pharmacist, under the pharmacist's direct supervision, in a wrapper or container labelled with
the drug's name, the quantity supplied and the dealer's name and address. The spec's template 8 and its labels research implement 94(2) and call it "the dispenser's own label"; its Pharma
retail preset pins a "loose-dispense" label, which can only be template 8, since the library of 14 has no other pharmacy dispense template. A shop that cuts a strip with template 8 is
asked for a patient and a register serial number it may not have, and it has no field for the drug's name, the first of the three particulars Rule 65(19) asks for. **Fix:** a `loose` label kind (drug, quantity, the dealer's name and address from the company, plus
batch, expiry and the unit MRP as the shop's own addition, labelled so), and keep template 8 for made-up medicines (A3).

**Twelve further asks.** What a pharmacy needs that R1 to R10 did not say, or that the spec leaves out. Sizes are [J] on the spec's own scale; "where" says which lane's files the ask touches
(the spec's lanes, 15.1).

| # | Ask | Why | Size, where |
|---|---|---|---|
| A1 | **A no-stop FEFO pick:** when one batch covers the need and is neither short-dated nor in a warning tier, take the proposal without opening the panel (the chip still shows batch, expiry and MRP); open the panel on an exception or a key | About 11% more key presses per line otherwise (3.15) | S; the panel's caller (lane V) |
| A2 | **The allocation panel as a self-contained component** (cells, need, policy in; lines out), not tied to the voucher screen's state; and parked-bill semantics: a parked bill moves no stock and re-proposes on resume | A counter on its own writer (3.15) must host the same panel, or duplicate it | S to M; lane V |
| A3 | **A loose-supply label** (F2) | The wrong statutory label otherwise | S; lane P (template data and a rules row) |
| A4 | **A tax-inclusive MRP sale line:** a quantity at MRP, discount off MRP, tax derived, the exclusive rate stored with enough precision | Baskets cannot total the sum of MRPs today (3.3); shared with general retail | L; the counter writer, not the spec |
| A5 | **Free quantity:** a Free column in the inward grid, a `freeQty` on the line (net-rate posting; the free units move stock in a named batch; the e-invoice `FreeQty`), the scheme remembered per purchase batch | The spec names it a non-goal; it is the trade's daily money (3.6) | L; lanes V and I |
| A6 | **PTR and PTS at batch grain, and an effective price with a from-date applied in bulk by company** (F1) | The two-MRP problem and the margin lines (3.3, 3.7) | M; lanes I and E |
| A7 | **Schedule derived from a dated list:** keep the spec's `schedule` field as the stored result; add an NDPS marker (the NRx label) and a derivation from composition against an editable list with its source and effective-from date, overridable with a reason | A typed field goes stale (trap T1 in section 5); H1 changed in May 2026 | L, with item 11; new pure files first, the field behind the one-writer `types.ts` |
| A8 | **Prescription capture and the registers** (the spec's Q26): an `rx` sub-record on the voucher, the gate on C, H and H1 lines, the prescription and H1 register reports, lock after print, retention floors | The law's own records (3.8, 3.9); the spec says the invoice is not the record | L plus L; a design of its own |
| A9 | **A rack text field on the item**, shown in the picker and on the shelf label | Every specialist shows rack; the spec's shelf label has a `rack` field with no source | S; lane E |
| A10 | **Licence numbers and the pharmacist:** the company's drug licences (Forms 20 and 21, 20B and 21B) and other identity lines, a party's drug licence number and validity (a warning on a scheduled line to a party with none or a lapsed one), the pharmacist's registration and who is on duty | Rule 65(2), 65(4)(3), 65(5); no field exists (3.9) | M; the CF "Pharma" starter set proposes the party field as a custom field, the company has no home |
| A11 | **The expiry-return worksheet:** a delivery challan (route (b) of Circular 72/2018), pending-credit tracking by supplier and age, a valuation basis (MRP, purchase rate, landed) | The largest recurring money job (3.5) | L; on top of the spec's report 3 |
| A12 | **Batch alerts from a regulator's list** (3.20): a match screen over the spec's hold and trace | A differentiator none of the products I read advertises | S to M; reports and the batch overlay |

**What the spec's own timing means for pharmacy.** The core sale path is its slices **T-0 (17 to 21 days), T-1 (7 to 9), T-2 (16 to 20) and T-4 (12 to 14): 52 to 64 builder-days** by its
table [J, summed]. A pharmacy also wants the shelf and loose labels (a subset of T-6) and, optionally, scanning (T-5); neither is needed to ring a bill. **Server work matters for one
reason:** in a synced company an older app version can erase a batch hold made in the overlay, and the spec's protection ("hold wins", capability `trk1`, slice T-9a) ships in the same
window as the custom-fields server work; **a recall or an NSQ hold is not trustworthy in a synced pharmacy until that has been deployed**, whereas serial-clash detection, the bulk of T-9,
is irrelevant to batch goods. **The pharmacy-specific work beside the spec** (items 4, 5's register report, 6's worksheet, 8, 9, 10, 11, 12, 13 and 14, plus half of item 3's inclusive line and
revision, since the general POS shares it) is **about 60 to 80 builder-days** on the same scale [J: item 4 5 to 7; 8 and 9 and 11 and 12 at 8 to 10 each; 10 at 4 to 6; 6's worksheet 6 to 8;
13 at 4 to 6; 14 at 3 to 4; item 3's half 4 to 5; the register report 2 to 3]. **Elapsed time is set by the spec's order, not by the sum:** T-0b waits for the custom-fields build to land,
and on `VoucherScreen.tsx` its own T-2a queues behind that lane's edits unless the owner moves it (spec question 22). **Two things can start now without waiting:** the pure parts of
pharmacy (the schedule list and its derivation from a composition, the register and alert builders as pure functions over vouchers) as new files, and asking the spec's lane for F1, F2 and
A1 to A3, which are cheapest before the panel exists. The fields and screens that touch `types.ts` and the item form queue behind those one-writer files (spec 15.1).

Serial and IMEI numbers do not matter to a chemist's counter except for a medical-device shelf (glucometers, BP monitors); the spec's serial
half can follow pharmacy's needs by a release.

### 6.5 Repo rules that apply (so the build does not relearn them)

All from CLAUDE.md and its gotchas [C]:

- **A new master table** (Doctor, Patient, a scheme, a short-list entry) means `types.ts`, a Dexie version bump with migration in `db.ts`,
  `store.ts` `COLLECTIONS` and `checkDelete`, a seed if needed. **A new field on `StockItem` (composition, manufacturer, pack, rack) must be
  carried by every writer that rebuilds an item from form fields**, or Alter Item erases it; `master-resave.assert` classifies every
  `types.ts` key and fails on a new one.
- **New master fields go last** in the frozen field order, or in an Alt+key grid; assert the whole Enter walk with the new keys removed.
- **Every new screen** needs a `SCREEN_AREA` entry (`core/access.ts`), the router, the crumb, the menu, the palette, Go To; a create goes in
  `ui/quickCreate.ts`; a report is a row in the Reports Center catalogue; a screen that can hold typed work registers a `state/dirty.ts`
  probe. A new rebindable action is a `REBINDABLE` row in `core/shortcutRegistry.ts`; a new rail button has a stable id.
- **Snapshot on the line**: tax treatment, HSN, and now the **batch's MRP and the schedule at the time of sale**, or an edit to the master
  re-writes last year's register line.
- **A "posted" guard on every path that can re-arm a block** (the register lock, the duplicate-post guard).
- **Statutory figures are researched and cited, never remembered:** every figure in 3.3 and 3.9 carries `SCHEMA?` or a source note in the code.
- **System writes must not look like user actions** (lazy seeds go through `putSeedDoc`): the schedule list's seed is a system write.
- **Opt-in and asserted unchanged when unset** for every new setting.
- **A new voucher field is erased on every alteration unless the save carries it** (added 25 Oct). The Sales save builds the voucher as a literal, field by field, so an `rx` record, a
  `freeQty` or an effective-price field must be added to the row and save functions and classified in `voucher-resave` [C pos-code-audit 0.3 and the custom-fields code audit's finding
  4; spec 3.1 does the same for its own fields].
- **The F11 features screen rebuilds the `features` object from its nine keys on Accept**, so a new `pharmacy` key not added to that literal is erased the next time anyone presses Ctrl+A
  there [C pos-code-audit 10.3]: keep pharmacy configuration in its own `Company` field (as `print` and `ui` do) and let F11 hold only the on or off switch. The spec's `tracking` switch
  is in the same position.
- **Patients and doctors are not ledgers** (3.8; trap T23 in section 5), and **a pharmacy counter is a second writer of the same `Voucher`** (3.15), which makes the Bulk Entry writer the precedent for
  tax rows, round-off and an all-or-nothing transaction [C pos-code-audit 0.2].

### 6.6 Sequence and how to test it

1. Give the **batch spec's lane** findings F1 and F2 and asks A1 to A3 now, and get its answer on A2 (a hostable panel) before slice T-2a is built (6.4).
2. In parallel, as new files that touch no one-writer file: the dated schedule list and its derivation from a composition, the composition matcher, the register builders and the
   alert matcher as pure functions over vouchers. The fields and forms (composition, rack, licences, pharmacist) queue behind `types.ts` and the item form, which the spec's lanes own.
3. The sale path: items 1 to 4, 14. The purchase path: items 5, 12, 13.
4. The registers and the capture gate: items 8, 9.
5. The expiry money screens: items 6, 7.
6. **Test with real counters**, not with this document: six chemists (observe a morning at the counter, time ten real bills on their software
   and on ours), two stockists (observe a billing day and a collections call), one registered pharmacist reading the register formats line by
   line, one drugs-law advocate reading the "may say" and "may not say" lists in 6.3, and one CA on free goods and expiry returns. The owner's work at a
   Tally partner is the likely route to a pharmacy customer who would share an item master export under an NDA for the importer. Whether
   Marg, Busy or GoFrugal exports give an item master with batches in Excel is **unverified**: ask a customer to try.

### 6.7 The verdict, in one place [J]

**What to build in a pharmacy mode.** One opt-in company mode, on the same `Voucher`, written through a counter-specific writer, with six things that together make it a pharmacy and not a
billing app: (1) the batch engine's sale-time pick, with FEFO and an expiry block, taken without a stop when it is unambiguous; (2) per-batch MRP with a tax-inclusive line and a price
cap, the revised price kept apart from the printed one; (3) strip and loose with its own label; (4) the schedule gate, built on an editable dated list with its source and date, with the
prescription and Schedule H1 registers as reports over the vouchers' own `rx` records; (5) licence numbers and the pharmacist on the memo; (6) free-quantity schemes and the expiry-return
worksheet, which are where the trade's money moves. Then speed, reorder and panel billing, and, as the one idea no product I read advertises, batch holds from the regulator's NSQ lists.
Start with the retail chemist; the stockist waits for the GSP, the salesman work and the party licence control.

**What is a data or regulatory trap, and how to speak about it.** The schedule lists and the GST rates are dated data, and one list changed in May with its effect still ahead; a typed or derived flag is only as
good as its source and date. Substitution on a scheduled prescription, refill nudges and anything promotional that names a scheduled drug collide with Rules 65(11), 65(11A) and the draft
65(22); doctor commission is barred on the physician's side; an e-prescription's standing, a software register's acceptability and the chemist's side of commissions are unsettled; a
free line, an MRP basket and a loose label each fail on a repo fact or a mis-chosen rule; and patient data becomes a statutory duty from about May 2027. **Say what the software prints and
what it checks, cite the rule, log every override with a reason, and never say "compliant", "valid", "safe" or "ready".**

**How much depends on the batch / serial spec.** The engine under items 1, 2 and the expiry reports is the spec's, and it is good; five of the fourteen Tier 0 items cannot work without
it and six work only in part. It does not design strip and loose, the schedule gate, the registers, licence numbers or free quantity, and it carries two defects for a pharmacy (F1 and F2) that
are cheap to fix before its panel exists. Plan on **about 52 to 64 builder-days of the spec's own work for the sale path and about 60 to 80 of pharmacy-specific work beside it** [J], and
on the spec's order, not the sum, deciding the calendar.

---

## 7. Open questions and what to verify

**For the owner.**

1. **First segment:** retail chemist first (the recommendation), or the distributor, whose needs pull in the GSP, a salesman app and the
   party licence control?
2. **Who is the pilot** (a chemist and a stockist the Tally partner already serves)?
3. **Drug data:** import-only in v1 (the recommendation), or a licensed database from the start (cost and terms unknown)?
4. **Offline:** must the counter work with the line down for a whole day? (The recommendation says yes.)
5. **Comfort with the "do not copy" list** in 6.1, in particular **no incentive-ranked substitution and no doctor commission**, since both
   are things Marg sells.
6. **Support model:** the loudest complaint about the leader is support after the sale (2.2). Is Comsky ready to answer a chemist's phone
   during a rush? That is a business decision no feature settles.
7. **The batch spec's decisions that a pharmacy lives with (added 25 Oct):** MRP in a batch's identity (its Q11), which this document accepts **provided F1 is fixed**; expired stock
   blocked by default for a new item with a reasoned override (its Q7), which suits a pharmacy; and its decision not to model the prescription register (its Q26). Do you accept all
   three, and will you pass F1, F2 and A1 to A3 (6.4) to the spec's lane?
8. **Register or memo book:** do the pilot pharmacies keep the prescription register or the serially numbered memo book (Rule 65(3)(2))? The spec's Q26 asks the same and the answer
   decides what the register report must print.
9. **Where the counter lives:** on its own writer (3.15, the POS audit's recommendation) or inside the Sales voucher screen with the spec's panel. The first is faster to ring a bill
   and needs A2; the second is cheaper to start and slower at the counter.

**To verify before any claim is made or any statute-driven field ships.** Status as of 25 Oct: V3, V4, V5, V6 and V10 were researched this pass and are **narrowed, not closed**; V2 is
partly answered; the rest are unchanged.

| # | Question | Status 25 Oct | Who |
|---|---|---|---|
| V1 | Does a software-printed register or memo satisfy Rule 65(3) and 65(4) in the target states, given the option declared at licence time (65(3)(2))? | Open | Drugs-law advocate, state Drugs Controller |
| V2 | What does the NDPS regime require of a **retail** chemist for psychotropic items that are also in Schedule H1 (the NRx label)? | **Partly:** the NDPS Rules text I could read points to Drugs Rules 65 (Rule 65-A) and, for a consignment, wants licence numbers, batch and transport details on the invoice (Rule 67) [W S46, secondary]; whether a separate retail register exists is open | Advocate |
| V3 | Does an **e-prescription** satisfy "in writing, signed, dated" (65(10)), and for which schedules? | **Context only:** the Telemedicine Practice Guidelines contemplate a prescription copy and sending it to a pharmacy with consent; commentary argues both ways on 65(10) [Ws S47]; open | Advocate |
| V4 | Patient and prescription data under the DPDP Act 2023 and its rules: consent, retention, access log | **Dates found:** main duties about 13 May 2027, consent managers 13 Nov 2026, logs kept a year [W S42, secondary]; what it means for a chemist is open | Advocate |
| V5 | Doctor commission and incentives: lawful and ethical for the chemist and the doctor? | **Physician side found:** 6.4.1 of the 2002 regulations [W S38]; the chemist's side is open | Advocate |
| V6 | GST on free goods in a scheme and on the buyer's input credit; expiry returns by challan against a credit note | **Narrowed:** Circular 92/11/2019-GST (secondary summaries) for the single-price case [W S43]; still open: free units of another item or batch, how to show them on the invoice, and a write-off of expired stock against section 17(5)(h) | CA |
| V7 | Whether a retailer's WhatsApp or push message naming a scheduled drug is an "advertisement" under the draft 65(22) when final | Open | Advocate, after the final text |
| V8 | Current Schedule H, H1 and X lists, the nil-rated drug list and ceiling prices from the Gazette and the regulator, **not** from this document. **Pregabalin (H1 entry 51, G.S.R. 377(E)) takes effect about 16 Nov 2026 by my arithmetic: read the Gazette for the date** | Open | Anyone, at build time |
| V9 | The two drafts (G.S.R. 791(E) CCTV and G.S.R. 861(E) advertising) and any final form | Open | Watch the Gazette |
| V10 | Barcodes and QR on strips: what real strips carry and whether a rule requires it | **Rule found** (the maker's duty, extended June 2026); what real strips carry is open, and GS1 says many hold a web address [Ws S44] | Handle real strips |
| V11 | Real counter speed: observe, do not assume | Open | Test sessions, 6.6 |
| V12 | Prices, AMC and feature lists on the vendors' own pages at the date of any comparison used in marketing | Open | Re-read |
| V13 | A retailer's legal duty on holding a batch named in a CDSCO NSQ or spurious alert (3.20) | Open | Advocate |
| V14 | Whether a delivery challan, debit note and credit note print through the same model as the invoice, so that batch, expiry and MRP appear on them (R9) | Open | Read the print code |
| V15 | F1 (6.4) pinned by a suite in the repo (a sale after a price revision lands on the batch's own cell); I reproduced it only in a scratch script, and not on a real pharmacy book | Open | The spec's lane |

---

## 8. Sources

Dates are the dates I read or the page's own date. A **[W]** I fetched and read; **[Ws]** I saw only in a search summary. Any [Ws] without an
`S` number comes from S36. Marg's own pages were read by direct HTTPS download with normal certificate checks because the fetch tool could not
verify the site's certificate chain; they are vendor marketing.

| # | Source | Tag |
|---|---|---|
| S1 | Marg ERP, pharmacy software page (prices, features, claims): https://margcompusoft.com/pharmacy-software.html | W |
| S2 | Marg ERP, drug inventory management software page: https://margcompusoft.com/retail/drug-inventory-management-software.html | W |
| S3 | Marg ERP, pharma distribution software page: https://margcompusoft.com/distribution/pharma_software.html | W |
| S4 | Marg blog, "Top Features Every Pharma Software Must Have in 2026" (28 Oct 2025; ABHA): https://margcompusoft.com/m/top-features-every-pharma-software-must-have-in-2026/ | W |
| S5 | Marg blog, Marg ERP vs eVitalRX (26 Mar 2026): https://margcompusoft.com/m/compare-pharmacy-software-marg-erp-vs-evitalrx/ | W |
| S6 | Marg blog, "How Marg Pharmacy Software Digitises Your Business" (26 Nov 2025): https://margcompusoft.com/m/how-marg-pharmacy-software-digitises-your-business/ | W |
| S7 | Marg help centre: database damage after power failure https://care.margcompusoft.com/margerp/error-queries/112/1/how-to-fix-database-damage-due-to-power-failure-margacc-which-appears-while-opening-marg-erp-software ; Syntax, DBF and program error https://care.margcompusoft.com/margerp/error-queries/1523/1/dbf ; "Not Responding" https://care.margcompusoft.com/margerp/general-queries/181111/1/How-to-Fix--Marg-Not | W |
| S8 | SoftwareSuggest, Marg ERP9+ Medical Billing https://www.softwaresuggest.com/marg-pharmacy-management and Pharmaceutical Distribution https://www.softwaresuggest.com/marg-pharmaceutical (reviews mostly 2015 to 2020; vendor-claimed profile) | W |
| S9 | Trustpilot, Marg ERP (2.3 of 5, 116 reviews, July 2025 to October 2026; **re-read on 25 Oct on the Spanish and the Austrian pages, same score, count and star split**; a search snippet showing 3.9 of 5 from 102 reviews is an older snapshot): https://es.trustpilot.com/review/margcompusoft.com and https://at.trustpilot.com/review/margcompusoft.com | W |
| S10 | Software Advice, Capterra, G2, Techjockey reviews of Marg (403 to the fetch tool; summaries only): https://www.softwareadvice.com/erp/marg-erp-9-plus-profile/reviews/ and the others | Ws |
| S11 | LOGIC ERP, pharma software: http://www.logicerp.com/Solutions/Pharma-Software | W |
| S12 | LOGIC ERP blogs: retail top 10 https://www.logicerp.com/blog/top-10-features-of-logic-erp-pharmacy-retail-software-1-pharmacy-billing-software/ ; distribution top 10 https://www.logicerp.com/blog/?p=7056 ; wholesale benefits https://www.logicerp.com/blog/top-10-best-benefits-of-logic-erp-pharma-wholesale-distribution-software-for-wholesalers-distributors/ ; distribution page https://www.logicerp.com/distribution/pharma-and-healthcare/pharmacy-distribution-software | W |
| S13 | LOGIC ERP knowledge base: batch-wise free schemes https://kb.logicerp.com/logic-erp-in-action-a-feature-overview/distribution/billing/batch-wise-free-schemes ; setup schemes https://kb.logicerp.com/logic-erp-learning-hub-tutorials-and-best-practices/setups/configurations/setup-schemes ; POS schemes https://kb.logicerp.com/logic-erp-in-action-a-feature-overview/retail/point-of-sale/flexible-schemes-and-promotions | W |
| S14 | GoFrugal: pharmacy highlights https://www.gofrugal.com/blog/pharmacy-solution-highlights ; pharmacy software https://www.gofrugal.com/retail/pharmacy-software ; medical store billing https://www.gofrugal.com/retail/pharmacy-software/medical-store-billing.html ; expiry management https://www.gofrugal.com/blog/pharmacy-expiry-management/ ; I/U concept https://community.gofrugal.com/portal/en/kb/articles/i-u-concept | W |
| S15 | BUSY: pharmacy https://busy.in/accounting-software/pharmacy ; medical store https://busy.in/accounting-software/retail-shop/medical-store/ | W |
| S16 | Medeil: Techjockey https://www.techjockey.com/detail/vanuston-medeil (W); FitGap, Capterra, IndiaMART summaries (Ws) | W, Ws |
| S17 | Zoho Inventory batch tracking help https://www.zoho.com/inventory/help/settings/batch-tracking.html (W); a partner's pharmacy-chain case study (Ws) | W, Ws |
| S18 | Odoo 18, expiration dates https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/product_management/product_tracking/expiration_dates.html (W); Odoo India localisation and pharmacy add-on summaries (Ws) | W, Ws |
| S19 | Accountune, six medical-store tools compared (a competitor) https://accountune.com/best-billing-software-medical-store-india (W); Ksolves, Marg to ERPNext pharma distributor case study (a vendor) https://www.ksolves.com/case-studies/erpnext/marg-erp-to-erpnext-pharma-distributor (W); Commmerce comparison (Ws) | W, Ws |
| S20 | **Drugs Rules, 1945, consolidated PDF hosted by CDSCO** (874 pages; Rule 65 at pages 41 to 50; Rule 97 labelling; Schedules H and H1 at pages 503 to 519; consolidation date not determined): https://cdsco.gov.in/opencms/resources/UploadCDSCOWeb/2022/drug_rules/Drugs%20Rules,%201945.pdf. **Re-read on 25 Oct from the PDF's text layer (extracted locally with PDFKit):** Rule 65 (pages 41 to 50), Rule 74(p) (page 71), Rule 94(2) (page 103), the explanation closing Rule 96 and 96(1)(vii) (pages 107 to 110), Rule 97 (pages 110 to 112), Schedule H (551 entries, the list ends on page 518) and Schedule H1 (48 entries, from page 519) | W |
| S21 | NHSRC, Schedule H1 register format (G.S.R. 588(E), 30 Aug 2013) and NDPS Forms 3D, 3E, 3H (G.S.R. 359(E), 5 May 2015): https://www.nhsrcindia.org/sites/default/files/2021-07/Attachment%20A%2C%20Schedule%20H1%20register%20and%20NDPS%20Forms.pdf | W |
| S22 | High Court of Punjab and Haryana, CWP-26127-2026, Harman Medical Store v State of Punjab, 18 Aug 2026: https://www.livelaw.in/pdf_upload/2026/09/02/18-697567.pdf | W |
| S23 | Draft CCTV rule, G.S.R. 791(E), 8 Sep 2026: SCC Online https://www.scconline.com/blog/?p=396823 ; Medical Dialogues https://medicaldialogues.in/amp/news/industry/pharma/cctv-at-pharmacies-health-ministry-proposes-surveillance-of-prescription-drug-sales-3-month-recording-mandate-179527 | W |
| S24 | Draft advertising rule, G.S.R. 861(E), 28 Sep 2026: https://medicaldialogues.in/amp/news/industry/pharma/health-ministry-proposes-ban-on-prescription-drug-advertisements-without-prior-approval-180904 | W |
| S25 | PIB, recommendations of the 56th GST Council (3 Sep 2025), Annexure I https://www.pib.gov.in/PressReleasePage.aspx?PRID=2163555 ; PIB FAQs https://www.pib.gov.in/PressReleasePage.aspx?PRID=2163560 (read 22 Oct; **both returned 403 on 25 Oct**, so the figures in 3.3 are as then read, with search summaries agreeing) | W |
| S26 | NPPA memoranda of 12 and 13 Sep 2025 on MRP revision, as set out by Lakshmikumaran and Sridharan https://www.lkslaw.com/insights/articles/revision-of-mrp-post-gst-2-0-reforms-mrp-labelling-norms-for-fmcg-and-pharma-sectors (W); news summaries of the NPPA direction (Ws) | W, Ws |
| S27 | DPCO 2013: Indian Pharma Post, norms for fixing prices https://www.indianpharmapost.com/policy/lite/norms-for-fixing-prices-of-medicines-15032 (W); IndiaLaw, 2026 ceiling-price analysis https://www.indialaw.in/blog/regulatory/dpco-2013-ceiling-price-2026-legal-analysis-of-nppa-order/ (W); para 24 and the June 2026 amendment in search summaries (Ws) | W, Ws |
| S28 | CBIC Circular 72/2018 (26 Oct 2018), return of time-expired drugs: https://gstzen.in/a/central-tax-circular-722018-circular-to-clarify-the-procedure-in-respect-of-return-of-time-expired-drugs-or-medicines.html | W |
| S29 | Loose tablets: Medical Dialogues (11 Apr 2026) https://medicaldialogues.in/news/industry/industry-perspective/pharmacies-deny-loose-tablets-govt-mulls-exact-prescription-quantity-drug-dispensing-aiocd-raises-concerns-168463 (W); a 2011 Lok Sabha answer and a compliance blog on DPCO para 27 (Ws) | W, Ws |
| S30 | e-invoice: BUSY explainer on the Rs 5 crore limit (updated 4 Aug 2026) https://busy.in/gst/e-invoicing-turnover-limit-vs-gst-threshold-whats-the-difference-and-why-it-matters/ ; Taxmann on the GSTN advisory of 5 Nov 2024 https://www.taxmann.com/post/blog/time-limit-of-30-days-for-reporting-e-invoice-on-irp-portal-for-taxpayers-with-aato-of-10-crores-and-above-gstn-update/ | W |
| S31 | Schedule H1 changes (pregabalin, May 2026; a pharmacy-software guide on the H1 count): search summaries (the first version's only source; superseded by S37) | Ws |
| S32 | UP FSDA directive on pharmacy websites (5 and 6 Oct 2026; paywalled, partial) https://www.uniindia.com/north-india/admin-up-drug-prices-websites/204706 and https://www.uniindia.com/north-india/health-up-medicines/204033 (W); e-pharmacy status and the 2018 Delhi High Court order (Ws) | W, Ws |
| S33 | Drug data licensing: MIMS, CIMS, Tata 1mg, the 2016 fixed-dose-combination ban (search summaries) | Ws |
| S34 | Pharmacy Practice Regulations 2015 (Pharmacy Council of India), summaries only | Ws |
| S35 | This repo and the `.wt-cf` worktree, read-only: `src/types.ts`, `src/core/inventory.ts`, `src/core/priceLevel.ts`, `src/core/creditLimit.ts`, `src/core/share.ts`, `src/core/upi.ts`, `src/core/recall.ts`, `src/ui/VoucherScreen.tsx`, `src/ui/inventory/BatchSummary.tsx`, `src/ui/inventory/stockquery.tsx`, `src/ui/inventory/reorder.tsx`, `docs/FEATURE-PARITY.md`, `docs/parity-data.json`, `docs/research/ROADMAP-ADDENDUM.md`, `docs/research/mobile-personas-and-jobs.md`, `.wt-cf/docs/design/custom-fields-spec.md` | C |
| S36 | Search summaries without a readable page: expiry return windows (an accounting glossary and a reconciliation guide), Zoho and Odoo partner pages, Marg review aggregators, Medeil marketplace listings | Ws |
| S37 | Pregabalin in Schedule H1: Drugs (Second Amendment) Rules 2026, G.S.R. 377(E), 13 May 2026 (draft G.S.R. 54(E), 21 Jan 2026): Cliniexperts https://cliniexperts.com/regulatory-update/2026-05-13-g-s-r-377e_notification-for-amendment-in-schedule-h1-for-inclusion-of-pregabalin-and-its-drug-formulations/ (W; entry 51 and "180 days after publication", but the page also lists an enforcement date of 13 May, which conflicts); Indian Pharma Post https://www.indianpharmapost.com/policy/lite/health-ministry-classifies-pregabalin-under-schedule-h1-to-curb-misuse-20322 (W; Gazette of 20 May; no commencement stated); ETV Bharat, Medical Dialogues, The Week (Ws). **The Gazette text was not read** | W, Ws |
| S38 | Medical Council regulations: the Gujarat Medical Council's copy of the Indian Medical Council (Professional Conduct, Etiquette and Ethics) Regulations 2002, "amended upto 8th October 2016", clauses 6.3 and 6.4.1: https://gmcgujarat.org/actnrules.aspx?id=29 (W); the NMC's 2023 regulations suspended in August 2023 with the 2002 regulations back in force, and their gift and inducement clauses: search summaries (Ws) | W, Ws |
| S39 | GoFrugal on Capterra and GetApp (general POS reviews, little pharmacy content), search summary: https://www.capterra.in/reviews/71693/gofrugal-pos-software | Ws |
| S40 | BUSY, read 25 Oct: https://busy.in/pharmacy-billing-software and https://busy.in/pharma/ | W |
| S41 | RxPOS (Sekhon Technologies), TechJockey listing: https://techjockey.com/detail/rxpos | W (the listing contradicts itself) |
| S42 | DPDP Rules 2025: SCC Online, key highlights (26 Dec 2025) https://www.scconline.com/blog/post/2025/12/26/digital-personal-data-protection-rules-2025-key-highlights/ (W; it says "eighteen months" without dates); the dates of 13 Nov 2025, 13 Nov 2026 and 13 May 2027 and the notification number come from search summaries of JSA, S.S. Rana and Deccan Herald (Ws) | W (secondary), Ws |
| S43 | CBIC Circular 92/11/2019-GST (7 Mar 2019), sales promotion schemes: TaxGuru https://taxguru.in/goods-and-service-tax/sales-promotion-schemes-taxability-itc.html (W, a secondary summary); **the circular itself**, https://gstcouncil.gov.in/sites/default/files/2024-06/circular-cgst-92.pdf, **was a PDF the fetch tool could not read**; Lexplosion and StudyCafe summaries (Ws). The 57th GST Council's proposal of 8 Oct 2026 on section 17(5)(h) and free samples, not operative on 20 Oct: https://taxguru.in/goods-and-service-tax/57th-gst-council-free-samples-itc-section-17-5-eligibility-practical-impact.html (W, a single source). Section 17(5)(h) itself was read only in summaries | W (secondary), Ws |
| S44 | The QR and barcode rule: the June 2026 amendment to Schedule H2 (vaccines, antimicrobials, NDPS and anti-cancer drugs; 1 July 2027 and 1 July 2028): All India Radio https://newsonair.gov.in/health-ministry-expands-qr-code-based-track-and-trace-framework-for-medicines/ (W); BioSpectrum, Indian Pharma Post and others (Ws). GS1 India's position paper of Oct 2023, https://www.gs1.org/docs/healthcare/position-papers/paper-gs1-hc-india-oct2023.pdf (403 to the fetch tool; a search summary only, Ws) | W, Ws |
| S45 | CDSCO NSQ alert for January 2025, PDF read as extracted text: https://cdsco.gov.in/opencms/resources/UploadCDSCOWeb/2018/UploadAlertsFiles/NSQ%20ALERT%20FOR%20THE%20MONTH%20OF%20JANUARY-2025.pdf (W); the Latest Alerts page https://cdsco.gov.in/opencms/opencms/en/Latest-Alerts/ and 2026 news counts (Ws) | W, Ws |
| S46 | NDPS Rules 1985 on a case-law site, https://www.casemine.com/act/in/5f5f11de9fca195c05cce7d4 (**the fetch tool's summary of the page, not my line-by-line reading**; Rules 52-B, 65-A, 66, 67); LiveLaw's report of the High Court judgment https://www.livelaw.in/high-court/punjab-and-haryana-high-court/ndps-act-pharmacist-licence-cancellation-excess-psychotropic-stock-548248 (W, partly paywalled); a Delhi case description of Rule 65-A (Ws) | W (weak), W, Ws |
| S47 | Telemedicine Practice Guidelines 2020: MediaNama's summary https://www.medianama.com/2020/03/223-summary-india-telemedicine-guidelines/ (W); law-firm and journal summaries, and 2016 to 2018 commentary on e-prescriptions and Rule 65(10) (Ws). The guidelines' own text was not read | W, Ws |
| S48 | NHCX and pharmacies: Outlook Money, the ABDM FHIR implementation guide and other explainers (search summaries) | Ws |
| S49 | Loose tablets, August 2026: Telangana Today https://telanganatoday.com/why-hyderabad-patients-are-forced-to-buy-more-medicines-than-prescribed (search summary) | Ws |
| S50 | In this repo, read 25 Oct: `docs/research/pos-code-audit.md`, `docs/design/batch-serial-labels-spec.md`, `docs/research/tracking-verticals-india.md`, `docs/research/labels-and-scanning.md`, `src/core/batches.ts`, `src/core/stockCell.ts`, `src/core/companyIds.ts`, `src/ui/VoucherScreen.tsx`, `docs/FEATURE-PARITY.md`, and the log of the `.wt-cf` worktree | C |
| S51 | Bill discounting and invoice discounting for pharma (KredX and others) and Marg's claim management: search summaries | Ws |

*End of document.*
