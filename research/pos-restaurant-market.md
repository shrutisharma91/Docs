# Restaurant, QSR and café POS: market study and design input for Comsky Books

> **Status:** research and design input only. Nothing here is built. First written 2026-10-22; **revised 2026-10-25 (revision 2, summarised after the related-docs
> note below)**. The only file written is this one; no code, no git write command, no root `npm run` script. Since the first draft: parity wave 3 is committed
> (`2cee555`) and deployed; the batch, serial/IMEI, label and scanning spec now exists (`docs/design/batch-serial-labels-spec.md`, its pure engines merged in `5aee765`
> and wired into no screen); the POS code audit exists (`docs/research/pos-code-audit.md`); and the custom-fields / stock-variants / Back-control build in `.wt-cf` is
> committed on its own branch (`2517916`) and not merged. I read all of those and wrote none of them.
> **Ask (owner, verbatim intent):** "Tally is not used in POS as it is weak. There are many known strong POS tools that are its alternative. In restaurants
> there is a tool called Petpooja ... I want to give a POS function in this, so retailers can be targeted. Check what is good in these software, and add."
> This document is the **restaurant / QSR / café** slice. Retail and pharmacy are other documents; where I touch them I say so.
> **Method:** **round 1 (2026-10-21 and 22):** about 95 web fetches and searches over vendor sites, help centres and API docs (Petpooja, Restroworks, UrbanPiper,
> GoFrugal, Limetray, DotPe, Odoo, Square, Lightspeed), government PDFs (CBIC circulars, GST Council notes and notifications, the CGST Act) which I text-extracted and
> **read in full**, review aggregators, and the consumer-law and food-law news that changes a restaurant bill; then a read of this repo. **Round 2 (2026-10-25):** about 80
> more, aimed at what round 1 could not reach because its search cap had run out: Toast, Lightspeed and Square help documents, Petpooja's own architecture pages, UrbanPiper's
> order contract, the CBIC FAQ on specified premises (read as a copy on a third-party host), the FSSAI order, the CGST Rules text of Rules 46 and 54, the composition and
> QR-code notifications, Circular 240, the CCPA's Chaayos order, and the 57th GST Council's recommendations of 8 October 2026; then a re-read of this repo against the POS code
> audit and the batch spec. Everything is **paraphrased**; no vendor or government text is reproduced (PDFs were read as extracted text).
> **Evidence tags.** **[W]** a page or PDF I fetched and read (the fetch tool summarises HTML pages, so "not mentioned" means "not found on that page",
> never "does not exist"; PDFs I read as extracted text). **[Ws]** a search summary only: I did not read the page. **[C]** read in this repo (file, line where
> useful). **[J]** my judgement. **[unverified]** I looked and could not confirm it, or I am saying it from memory: do not build on it without checking.
> **Related repo docs:** `docs/research/pos-code-audit.md` (what in today's code a POS bill touches; its lanes A to H are the build frame I map onto in 7.5),
> `docs/design/batch-serial-labels-spec.md` (print bridge, scan layer, Devices tab, barcode encoders: reused, not duplicated, 7.5), `docs/research/batch-serial-code-audit.md`,
> `docs/research/mobile-market.md` and `mobile-backend-readiness.md` (phone shell, HTTPS gate, thin vs offline replicas), `docs/research/einvoice-gsp.md` (e-invoice thresholds),
> `.wt-cf/docs/design/custom-fields-spec.md` (tracking variants, AD.3) and `back-nav-spec.md`.
>
> **Revision 2 (2026-10-25): what changed, in one place.**
> 1. **Closed gaps:** Torqus found (sections 1 and 9); Toast, Lightspeed and Square help documents read for voids, courses, tables and offline (3.3, 3.4, 3.13); Petpooja's four POS
>    types and its Bridge Server read (new 2.6); UrbanPiper's order contract read (3.7); the CBIC FAQ (as a copy), Annexure VII, Rule 46, the composition notification, the FSSAI
>    order and Circular 240 read (section 4).
> 2. **Findings that changed a recommendation:** the leaders gate multi-device offline behind an on-premises server, and Toast itself says orders do not cross devices offline (7.4);
>    aggregator webhooks are unsigned, time-limited and public, so the cloud must receive them (3.7, 7.4); a stand-alone restaurant cannot declare itself "specified premises" (the
>    CBIC FAQ says so, 4.1); the CCPA has now ordered a chain to change what its billing software does by default (4.4); the 57th Council (8 Oct 2026) recommended same-line-of-business
>    credit for restaurant and catering services and 5% without credit for platform delivery services, **not yet notified** (4.11).
> 3. **Corrections to round 1:** posting shapes now follow the audit (a tip or a charge must not be an extra credit leg on the Sales voucher: it would be taxed, 7.3); the portal's
>    reporting slot for 9(5) supplies is Table 14(b) and 3.1.1(ii), not the circular's "Table 8 and 3.1(c) for the time being" (4.3, search summaries); the stock engine does **not**
>    skip service items, so a dish needs a stockless rule (0.2 finding 10; 7.2); the small-bill rule sits in Rule 46's proviso (4.7); the composition limit is read from its notification (4.2).
> 4. **New:** 7.5 maps restaurant mode onto the audit's lanes and the batch spec's bridge; 3.7 gains the aggregator contract; 4.8 and 4.9 gain payment-data and consent notes; 4.11 lists
>    what is pending.

---

## 0. Read this first

### 0.1 What I could not verify, and why it matters

1. **No live account and no hardware.** I used no product. Offline behaviour, printer behaviour, role behaviour and speed come from vendor pages, release
   notes, API docs and reviews. A feature a vendor page does not mention is "not found", not "absent".
2. **Round 1's search cap ran out; round 2 ran into the same shared per-turn cap of 200 searches near its end (after which I could only fetch pages by URL, and did not work around it), and several sources stayed closed.** Still **search-summary only [Ws]** after round 2: Zomato's and Swiggy's partner API
   documents (partner-gated; I found no public Swiggy page at all), ONDC's GST position, Capterra, G2 and Software Advice (403 on fetch), the CCPA's own guideline PDF (the host
   refused the connection; PIB returned 403), the PIB release of the 57th Council (403: I used law-firm and news summaries), Epson's ESC/POS reference (403: third-party guides),
   and the operative text of Notification 15/2025 (not read). The CBIC's own FAQ PDF was not on a CBIC host I could open: I read a **copy** on a third-party host, and the CBIC rule
   pages failed a certificate check, so Rules 46 and 54 come from the CBIC's consolidated PDF as amended to 1 Jun 2021. I say per claim which kind of evidence it is.
3. **Review sites are mostly walled.** Capterra, GetApp, G2, Software Advice and MouthShut returned 403 again in round 2; I have search summaries of G2 and Capterra only. What I
   read: SoftwareSuggest, Techjockey, one Trustpilot page and the Apple App Store listing of Petpooja's captain app. Ratings on these sites come from vendor-invited reviews and run
   high: weigh the *complaints*, not the stars. No Reddit or YouTube result surfaced in either round.
4. **Torqus (found in round 2, with a conflict).** A Pune company founded in 2013 and bought by Dineout, reported 27 Nov 2018 **[W thenewsminute]**; its POS is listed with 4.3 from
   12 reviews, all dated 2020 or 2021 **[W softwaresuggest]**. Search summaries say it is now sold as **inresto**, while the 2018 report names inResto as a *separate* Dineout product,
   so which is which I did not settle **[Ws]**. I could not open a vendor page, so I have no first-hand feature list (section 1).
5. **Statutory primaries I could and could not read.** Read in full (as extracted text): CBIC circulars 164/20/2021, 167/23/2021 and **240/34/2024**, the GST Council's
   23rd-meeting decisions note (Nov 2017), Notifications 05/2025-Central Tax (Rate), **14/2019-Central Tax (composition limit)** and **14/2020-Central Tax (QR code)**, the
   Council's consolidated service-rate table, the 56th Council press release (Sep 2025), the CGST Act as consolidated on 1 Aug 2021, **the CGST Rules as amended to 1 Jun 2021
   (Rules 46 and 54)**, the 2017 composition FAQ, **the declaration form Annexure VII**, **the FSSAI orders of 8 Jun and 30 Sep 2021**, a **copy of the CBIC FAQ of Mar 2025 on
   "specified premises"**, and a state tax department's restaurant training deck. **Not read:** the CCPA guidelines PDF (commentary only), Notification 15/2025 (the Sep 2025 rate
   amendment), any amendment of the composition notification after Mar 2019, and every text issued after the CGST Act and Rules copies' dates (2021): later amendments are not in them.
6. **Prices conflict.** Petpooja's own pricing page gives rupee figures with no billing period; an aggregator listing says the same SKUs are per year plus
   GST and structures them differently; another listing says ₹10,000 plus a ₹7,500 renewal. I give all three and do not pick.
7. **Vendor scale numbers are marketing.** "25,000 restaurants", "1,50,000 businesses", "40,000 restaurants" are claims, counted differently (outlets,
   businesses across all sectors, brands). I cite them only as claims.

### 0.2 Sixteen findings that change what we build (twelve from round 1, several revised; four new, 13 to 16)

1. **A restaurant bill is a GST sales voucher; the *order* is not.** The books engine can already post the bill. What is missing is everything before it:
   tables, an open check that lives for hours across several kitchen tickets, modifiers, void control. **[J]** Model the check as **append-only kitchen-ticket
   events** and write the Sales voucher(s) only at settlement. Append-only events merge cleanly across devices; an editable shared check does not (section 7.3).
2. **Petpooja wins on price and breadth, not on accounting.** Its India list prices run from ₹12,000 (base: cloud billing, inventory, 90+ reports, unlimited users
   and terminals, aggregator hooks) up to ₹40,000 for marketing bundles, plus an aggregator-listed ₹6,000 and ₹9,000 entry tier **[W]**. It sells a sales-led
   package with installation visits. We will not out-cheap it. **[J]** The opening is **books in the same box**: correct GST (5% with no credit, the e-commerce
   operator rule, composition), real ledgers, a CA-friendly export, no per-terminal fee.
3. **Several rules that touch the bill moved recently and some are contested.** A default service charge is now treated as an unfair trade practice: the CCPA's
   2022 guidelines were upheld by a single judge of the Delhi High Court on 28 Mar 2025, an appeal was still pending in the latest report I found, and the CCPA
   has since ordered at least one restaurant to **change its billing software** so charges are not added by default (the Chaayos order of 14 Jul 2026, read in
   round 2: 4.4) **[Ws][W]**. A POS that adds 10% service charge by default is a liability. **Default OFF.**
4. **Aggregator orders are not ordinary sales.** Since 1 Jan 2022 the aggregator (the e-commerce operator) pays the GST on restaurant service and issues the
   invoice; the restaurant reports those supplies (the circular says GSTR-1 Table 8 and GSTR-3B 3.1(c) "for the time being"; the portal now has Table 14(b) and
   3.1.1(ii), 4.3) and counts them in its turnover **[W circ167][Ws]**. Comsky has no tax class for that **[C]**. The aggregator's commission is a separate 18% service whose tax a 5%-no-credit restaurant cannot claim back **[J]**.
5. **5% is a *mandatory* rate with a no-credit condition**, not an option **[W rates]**. Restaurants in hotel premises that qualify as "specified premises"
   pay 18% with credit; from 1 Apr 2025 that status depends on the *previous* year's room values or a declaration by a hotel-accommodation supplier **[W ctr05]**;
   the CBIC's own FAQ and the declaration form say only a registered person supplying hotel accommodation can declare, so **a stand-alone restaurant cannot**
   **[W faq-copy Q5][W annexure7]** (the Council's Sep 2025 recommendation of an Explanation says the same **[W gstc56]**). Takeaway, door delivery and
   cloud or central kitchens are all "restaurant service" **[W circ164]**. A 57th-Council recommendation (8 Oct 2026) would allow limited same-line-of-business credit
   for restaurant and catering services; it is **not notified**, and I could not tell whether it touches the no-credit condition on the 5% entry (4.11) **[W commentary]**.
6. **Direct Zomato and Swiggy integration is a partner programme, and the market goes through middleware.** Zomato gates its POS API behind onboarding (NDA, IP
   whitelist, partner-only docs per a third-party SDK) **[Ws]**; UrbanPiper sits between aggregators and POS software, and Odoo itself uses UrbanPiper for India's
   two big platforms **[W]**. **[J]** v2 should do the same; v1 is a fast manual "aggregator order" entry plus payout reconciliation. UrbanPiper's order webhooks are
   **unsigned**, must be answered with a 2xx inside about five seconds, are tried four times over a few minutes and then dropped, and an endpoint that keeps timing out is
   switched off **[W]**: a restaurant's own PC cannot be the receiver, the cloud has to be (3.7, 7.4).
7. **The leaders' loudest complaints are peak-hour slowness and uneven support**, repeated for Petpooja, Restroworks and Limetray **[W]**, and their selling point is
   that captain devices **stay in sync offline** **[W]**. **Round 2 narrowed that claim:** Petpooja sells four POS types and only "Local" (a Bridge Server on the outlet's own
   network, installed by Petpooja) carries the captain app, the KDS and several counters through an outage; its Electron and Android types hold up to five days of
   single-device offline billing **[W petpooja-pos-types, offline-pos]**; Toast tells staff to keep to one device during an outage and its help says that orders taken on one terminal do
   not appear on others **[W][Ws]**. Comsky syncs through the cloud only; there is no on-premises hub **[C]**. Single-device offline is therefore the market norm, and a hub is the gate for
   everything multi-device: this is the biggest architecture risk for any restaurant with more than one device (sections 7.4, 8).
8. **Cashier and staff theft is a purchase reason, and our operator rights are a client-side control by the code's own admission** **[C access.ts:12]**.
   v1 can ship honest controls (reason, manager PIN, no deletes, KOT-versus-bill reconciliation); real enforcement needs server-side rights, which is backend work.
9. **The menu is the hard part, and each channel constrains it.** Swiggy allows at most four variant groups per item and caps items per store; Zomato allows two
   nesting levels and treats names that differ only by digits as the same item **[W]**. The menu needs a per-channel layer, not one flat list.
10. **Recipes exist as a field and nothing uses them at sale time.** `StockItem.bom` is read only by the Manufacturing screen **[C]**. Multi-level recipes (a gravy
    base inside a dish) are listed as not built **[C CLAUDE.md]**, and a competitor's reviewers complain about exactly semi-processed items **[W]**. Ingredient
    stock is a v2 job. **And a dish cannot be a plain stock item today:** the stock engine never reads `StockItem.isService` (only the e-invoice, e-way bill and inward-kind
    code do) and there is no no-stock flag, so a dish sold from zero stock raises the negative-stock prompt on every bill **[C types.ts:525; grep of `src/core/inventory.ts`;
    VoucherScreen.tsx:2490-2503]**. v1 needs a stockless-item rule in the engine (7.2).
11. **Print is a gap.** The repo has no thermal 58/80 mm receipt, no ESC/POS, no kitchen-ticket output **[C]**. It does have an NPCI-style UPI link builder and a
    QR encoder, so a UPI QR on the bill is cheap **[C upi.ts, qr.ts]**; confirming a payment is not.
12. **FSSAI and per-outlet identity.** Food businesses must print their 14-digit FSSAI licence or registration number on bills: read in round 2 from the FSSAI's own
    PDF (the 8 Jun 2021 order, enforcement moved to 1 Jan 2022 by the 30 Sep 2021 order; it names restaurants and mithai shops) **[W fssai-order]**. `Company`
    has no such field **[C]**, and a licence is per premises, so it belongs on the outlet, not the company **[J]**.
13. **Void control is a permission, a reason and an approver, and the leaders record the approver.** Toast: voiding is a permission on a job or an employee; without it a
    manager's access code or card is entered; a reason comes from a configured list; a check can be voided only until its payment is captured, then it is a refund **[W]**; and
    its reports list voids by server and by approver **[Ws]**. Petpooja's own glossary: a void keeps the bill number at zero value with reason, user and time, belongs to a
    supervisor, the person who benefits must not approve it, and after the day is closed a credit note replaces it **[W]**. Our rights have no cancel, discount, override,
    refund, reprint or no-sale action, and cancel counts as alter **[C access.ts; pos-code-audit section 9]** (3.4, 7.2).
14. **Engine constraints decide how a restaurant bill is posted.** `voucherGstSummary` adds any non-tax, non-party leg of a Sales voucher to taxable value, so a tip, a
    service charge or a split-tender leg cannot be an extra leg of the bill **[C gst.ts:161-177, via pos-code-audit finding 4]**. The shapes that need no engine change are a
    party that *is* the tender ledger, or a Sales voucher plus same-transaction Receipts; charges are service lines with their own taxability; a tip is a Receipt (7.3). A voucher
    also has no time, terminal or cashier, and the working date defaults to the last voucher's date **[C types.ts; store.ts:357, via the audit]**.
15. **The cloud, not the outlet, must take aggregator orders, and the REST API is the wrong door.** Public HTTPS, unsigned calls, a few seconds to answer, retries and
    dead-lettering (3.7) all point at the backend, which is HTTP-only today **[C HANDOFF.md]**; the REST API writes raw vouchers with no numbering or balance check **[C
    restPolicy.js:18-24, via the audit]**. Intake belongs in a non-posting pending-orders table, accepted at the counter, then turned into a bill by the POS writer (7.4).
16. **Defaults are now a legal exposure.** The CCPA's Chaayos order (14 Jul 2026) found that commands embedded in the billing software added the charge to every bill, held
    that relabelling it "optional" did not cure the earlier default, and directed the software to be changed **[W scconline]**. A POS vendor's defaults are part of the
    restaurant's compliance (4.4).

### 0.3 Decisions I need from the owner

| # | Decision | My recommendation | Why |
|---|---|---|---|
| D1 | Who v1 is for | Counter service (QSR, café, bakery, sweet shop) and small dine-in, one to five outlets, owner who also keeps books with a Tally-using CA | That is where books-in-the-box is a reason to switch; fine dining, hotels and 100-outlet chains need a hub, KDS and middleware first **[J]** |
| D2 | Multi-device restaurants in v1 | One counter device per outlet in v1; extra tablets only while online. A local hub over the outlet's LAN in v2 | Cloud-only sync cannot keep two offline tablets agreeing on a table **[J]**; Petpooja gates multi-device offline behind its on-premises "Local" type and Toast says orders do not cross devices offline **[W][Ws]** |
| D3 | Aggregators | Start the UrbanPiper-style middleware conversation and the Zomato and Swiggy partner applications now; ship manual aggregator entry in v1; decide that the **cloud backend** receives the webhooks (public HTTPS first) | Partner calendars are the long pole, not engineering **[Ws][J]**; the webhooks are unsigned, time-limited and retried **[W]** |
| D4 | Service charge | Ship it **off by default**, as a visible, removable per-bill line, and do not turn on any GST treatment for it until a CA signs the wording | Law is split between GST (part of value) and consumer law (voluntary, no GST) **[W][Ws]** |
| D5 | Payments | UPI QR on the bill and cash, UPI and card as recorded tenders in v1; no card data ever in the app; auto-confirmation via a payment provider later | Since 1 Oct 2022 only issuers and networks may store card data **[Ws]** (4.8); the safe design needs no card number **[J]** |
| D6 | Liquor | Out of v1 (a plain Non-GST line is possible; state excise and VAT are not) | Alcohol sits outside GST and composition **[W]**; state rules vary **[J]** |
| D7 | Who signs the GST and consumer-law items in section 4.10 | A practising CA, before v1 ships | Several rows are my reading of a circular, not a ruling **[J]** |
| D8 | Pricing posture | Published, per-company, no per-terminal fee, standard printers and Android devices, full export | The review complaints are about oversold bundles, hardware and contract lock-in **[W][Ws]** |
| D9 | The dish item model | A **stockless rule** on service-like items in the stock engine (a small, additive engine change that retail and pharmacy modes also want for service lines); dishes stay item lines, not ledgers | The engine ignores `isService`; a ledger-only bill loses the per-line GST snapshot and the item-wise reports a restaurant lives on **[C][J]** |
| D10 | Who owns the device bridge | One bridge, one printer allow-list and one Devices tab for labels, receipts and kitchen tickets | The audit proposes `electron/deviceBridge.cjs`, the batch spec `electron/labelBridge.cjs`: two names for one discipline **[C]**. The architect's call, not mine |
| D11 | The outlet hub's hardware | The counter PC is the hub; ask for no separate server in v1 | Petpooja's own spec for its server is an 8th-generation i5, 8 GB and a 1 TB SSD up to 25 outlets, installed by its team **[W]**; we cannot match an installer-led rollout, so the hub must be a role of a machine they already have **[J]** |

---

## 1. The market in one table

"Claimed scale" is the vendor's own number. "Price" is what I could read. Tags are on each cell.

| Product | What it is | Claimed scale | Price as read | Notable facts |
|---|---|---|---|---|
| **Petpooja** (India) | Restaurant POS suite ("Poss"), captain app, KDS, online-order hub, inventory and recipes, marketing and CRM, plus payroll (Attendo), GST billing for retail and B2B (Invoice), task app (Stepwise) **[W]** | "1,50,000+ businesses", 14+ years, country selector for India, UAE, Singapore, Malaysia, Thailand, South Africa, US, Canada **[W]**; its UAE page says 1,00,000+ clients **[Ws]** | ₹12,000 base; ₹20,000 or ₹30,000 "Operations Manager"; ₹20,000 to ₹40,000 "Marketing Manager"; period not stated **[W]**. An aggregator lists the same family as per year plus 18% GST, from ₹6,000 **[W]** | Unlimited users and terminals in the base **[W]**; aggregator integrations carry no Petpooja commission **[W]**; **four POS types**: Lite (cloud, one phone, no offline), Android (up to 5 days offline, one device), Electron (full, up to 5 days offline) and Local (a Bridge Server on the LAN), 2.6 **[W]** |
| **Restroworks (was Posist)** | Chain-grade platform: POS, KDS and kitchen production, central ("Base") kitchen, recipes, supply chain, delivery management, kiosk, QR ordering, branded apps, open API, marketplace **[W]** | "25,000+ restaurants", 500+ integrations, 200+ reports **[W]** | Quote only; depends on modules, size, terminals and hardware **[W]** | Anti-theft tools and a mobile "Cockpit" app **[W]**; a Brand, Format, Cluster, Deployment hierarchy for multi-outlet reporting **[W]** |
| **Limetray** | Cloud POS, online ordering site and app, CRM **[W]** | "4,500+ restaurants" **[W]** | On request; yearly subscriptions **[W]** | Reviews I could read are from 2020 to 2022 **[W]**; Capterra's 2026 pages still list it (4.4 from 5 users, reviews 2018 to 2023) and I found no 2026 news of a sale or closure **[Ws]** |
| **DotPe** | POS, QR ordering, online store, delivery and marketplace integrations in one dashboard, aimed at all small merchants **[W]** | "75 lakh+ businesses" (all sectors) **[W]** | Not shown; third-party listings say the core products need a sales contact **[W][Ws]** | A burger chain's own claim of a 98% cut in third-party commissions through QR **[W]** (a merchant's number, not a measured one) |
| **GoFrugal Restaurant** | POS, steward and KOT app, KDS, QR menu, recipes and food costing, central kitchen, loyalty, Swiggy, Zomato and ONDC hooks, built-in accounts with GST, 350+ MIS reports; cloud or on-premise; offline **[W]** | No total on the page **[W]** | From ₹500 per user per month on its page **[W]**; ₹500 and ₹1,100 per store per month on a listing **[W]**; the listing's own FAQ says $7 and $15, so it conflicts | The closest to Comsky in shape: POS plus accounts in one product **[J]** |
| **UrbanPiper** | Middleware: aggregator orders, menus and status into **any** POS (Hub, Meraki branded ordering, a Prime POS, an AI call-taking product) **[W]** | "40,000+ restaurants" **[W]** | Not shown **[W]** | Public POS-side API docs, webhooks, Swiggy and Zomato constraint pages **[W]** |
| **Torqus** | Pune POS and supply-chain suite for restaurants, cafes, bars, bakeries and cloud kitchens: tables, orders, menu, invoicing, inventory, KDS, loyalty, online ordering, reservations, multi-location, mobile POS, an offline mode, and a cashbook in the Standard plan, per its listing **[W softwaresuggest]** | Reported at 3,500+ outlets when Dineout bought it (27 Nov 2018; a combined Dineout, Dineout Pay and inResto claim of about 10,000 restaurants in 70+ cities) **[Ws][W]** | ₹15,000 a year on its listing, $208 in the same page's FAQ; other listings say ₹4,000 a month and $56 a month per outlet **[W][Ws]**: they conflict | Its reviews are 2020 to 2021 (4.3 from 12): inconsistent support, complex accounting workflows, a cluttered interface, an app that kept logging out, and a wish for Tally integration **[W]**. Search summaries say it is now sold as **inresto** under Dineout; the 2018 report lists inResto as a separate product: unsettled **[Ws]**. Also one of UrbanPiper's channel names **[W]** |
| **Toast** (US) | All-in-one restaurant platform tied to its own payments **[Ws]** | n/a | From $0 starter kit, $69 per month plan; card rates about 2.5% to 3.1% plus 15 cents; add-ons extra; 1 to 3 year terms; proprietary hardware **[Ws]** | Reviewers complain of support delays, billing disputes and processor lock-in; a 99-cent online-order fee was dropped in 2023 after restaurant protest **[Ws]**. Useful as a pattern, not a competitor in India. **Round 2 read its help documents:** voids by permission with a manager override and configured reasons, four ways to fire courses, a printed or on-screen changes ticket, and an offline mode in which devices do not share orders (3.3, 3.4, 3.13) **[W]** |
| **Square for Restaurants** (US) | POS with an Orders API (order, line items, modifiers, taxes, discounts, service charges, pickup and delivery fulfilments) **[W]** | n/a | Page unreadable; its coursing help says the Plus or Premium plan or a KDS subscription is needed **[W]** | Its order model is a clean reference for ours (section 3.1). Round 2 read its coursing help (hold and fire from the check; KDS on Android tablets only) and, as search summaries, its general offline card payments (up to 24 hours, one hour on a Reader or Stand, the merchant carries the risk) and a line in its restaurant help that normal operation needs the internet **[W][Ws]** |
| **Lightspeed Restaurant** (global) | Restaurant POS with floor plans, tables, open checks by table, local (dine-in) and to-go orders in its API **[W]** | n/a | Not read | **Round 2 read its K-Series help articles:** floor plans of up to 150 tables with seats, an order profile and printing profiles; production centres with order, course and transfer tickets; courses fired by the server; reprints by order, course or item; void reasons set in the back office **[W]** |
| **Odoo POS Restaurant** | Open-source-core ERP's restaurant mode: floors, tables, courses and fire, preparation printers and display, split, merge, tips, self-order, UrbanPiper delivery **[W]** | n/a | Enterprise licence needed for delivery integration **[W]** | The best **documented** feature list I could read; used below as the neutral reference |

**Channels, not vendors:** Zomato and Swiggy (aggregators, with their own POS partner programmes) and **ONDC** (an open network: buyer apps and seller apps, built on
a common protocol). ONDC's food share is small nationally and larger in a few cities by 2024 reports **[Ws]**; every commission figure comparing ONDC with the
two aggregators in my sources came from a vendor blog **[Ws]**.

---

## 2. Petpooja in depth

### 2.1 What it sells, and the pricing puzzle
- **Base plan ₹12,000 (India, excluding tax; period not stated on the page).** Cloud billing, menu management, inventory and a stock manager, 90+ configurable
  reports, aggregator and purchase-manager hooks (Hyperpure, Swiggy Assure, and DeliverIT as a self-set-up option), unlimited users and terminals, "unlimited AI
  agent queries", online installation and training, 24 by 7 online support **[W petpooja-pricing]**.
- **"Operations Manager" add-on, Growth ₹20,000:** captain app, token management, Zomato and Swiggy reconciliation, Tally integration, scan-and-QR ordering, waiter calling
  (with an annual maintenance charge), **KDS**, API and SAP/ERP integration, call-centre module, one installation visit; **excludes** table reservations, kiosk and
  e-invoicing. **Scale ₹30,000** adds reservations, kiosk software, **1,000 e-invoice credits** and a second visit **[W]**.
- **"Marketing Manager" add-on: Core ₹20,000** (CRM with loyalty, virtual wallet, feedback QR, reputation management, e-bills, 4,000 WhatsApp Business credits), **Growth ₹30,000**
  (adds a restaurant website and ordering page, digital display, 8,000 credits), **Scale ₹40,000** (adds ads or two influencers, 10,000 credits) **[W]**.
- **The puzzle.** SoftwareSuggest lists the SKUs differently (Poss Lite ₹6,000, Android Poss ₹9,000, Electron Poss Core ₹12,000, Growth ₹20,000, Scale ₹30,000, Local Poss Core
  ₹20,000, Growth ₹30,000, Scale ₹40,000), **per year plus 18% GST**, updated 2 Jul 2026, with a 7-day trial; its FAQ quotes dollar figures that contradict its own table **[W]**.
  So I cannot tell from the pricing page whether Growth stacks on Base or replaces it, and whether the figures renew yearly. Techjockey's older listing said ₹10,000 plus a ₹7,500
  renewal and no free trial **[Ws]**. **Round 2:** the SKU names in that listing are Petpooja's four POS types (Lite, Android, Electron, Local; see 2.6), and in the same table
  Local sits ₹8,000 to ₹10,000 above the matching Electron tier **[W softwaresuggest]**: what one would expect of a tier that also needs an on-premises server **[J]**.
  **Lesson [J]:** the buyer cannot see the real price without a sales call; a published price sheet is itself a feature.

### 2.2 Why restaurants pick it (my reading, **[J]** on the evidence above)
1. **Price and a bundle that matches the checklist of a first-time buyer:** billing, KOT, inventory, reports, aggregator hooks and support at one price.
2. **India-shaped details the global tools lack:** Zomato and Swiggy hooks with platform-wise reconciliation against payout sheets, commission and margin per platform, GST
   reports by item, HSN and category, Tally export, UPI and wallets in the captain app **[W]**.
3. **Hardware breadth:** it claims support for more than 200 printers, scales and scanners **[Ws]**, which matters when a café owns a 58 mm Bluetooth printer already.
4. **Sales-led rollout:** installation and training visits are written into the plan **[W]**.
5. **An embedded partner portal:** the actual Swiggy partner portal opens inside its POS **[Ws]**, so the owner does not leave the app to run offers or check ratings.

### 2.3 What its captain app shows about the roadmap (Apple App Store release notes, Dec 2023 to Dec 2025 **[W]**)
The order of additions is a free feature-priority list. Failed kitchen tickets auto-sync after a connection returns (Aug 2024); items take three decimals for weighed goods
(Sep 2024); kitchen-display priority per menu item and **area-wise variation pricing** (Dec 2024); category scheduling for offline menus (May 2025); order-level and
ticket-level comments, a colour-coded table view like the POS, table and ticket numbers on Bluetooth printouts (Jun to Jul 2025); accept, reject, hold or modify
**dine-in QR orders** from Petpooja and from Swiggy (Sep 2025); acceptance of **Zomato dine-in QR orders** and of table-reservation requests (Nov to Dec 2025). The
listing shows a 3.0 average from two ratings: too few to say anything.

### 2.4 Complaints (all dated where the page gave a date)
- **Peak-hour slowness, glitches and delays**, slow support with no resolution, features the reviewer felt were oversold, poor value (1-star, 19 Aug 2025,
  SoftwareSuggest); the same reviewer alleges hardware was sold on commission, which I cannot verify **[W]**. The page's own recurring cons: peak-time slowness, uneven support,
  gaps in inventory and CRM, cumbersome menu updates **[W]**.
- **Support lines hard to reach** with long holds (2 Oct 2020) and a generally poor experience (11 Oct 2024) **[W Techjockey]**; **costly for small businesses**, no free trial
  or plan; **menu updates are manual**; SMS sender-ID approvals tedious; more dashboards wanted in the mobile app **[W]**.
- **A refund promised in writing for 26 Dec 2025 and still unpaid on 9 Jan 2026**, with shifting timelines (Trustpilot, one review, 3.2 score) **[W]**. One anecdote; I list it only
  because refunds and billing disputes were also a theme for Toast **[Ws]**.
- G2 reviewers praise fast replies and note slower resolution of complex issues **[Ws]**.
- **Round 2, from search summaries of G2, Capterra and SoftwareSuggest (the pages themselves returned 403) [Ws]:** support response of 24 to 48 hours on non-critical issues
  (a figure that comes from a competitor's blog), modules such as KDS and QR ordering costing extra, slowdowns at peak, advanced inventory hard to learn, and **many syncing issues
  when connectivity is poor** (one G2 reviewer). The same competitor blog claims a 1.5 to 2% fee on online orders; **I could not verify it** and Petpooja's page says its
  aggregator integrations carry no commission of its own **[Ws][W]**. G2's own score is reported as 4.7 from 289 reviews, vendor-invited like the others.

### 2.5 What to copy and what to avoid **[J]**
Copy: per-platform reconciliation against payout sheets; token numbers for QSR; captain-side accept, reject, hold or modify for customer-placed orders; area-wise price lists;
failed-ticket retry. Avoid: unpublished prices, add-on staircases, and a menu editor that is slow to change (build a grid, an Excel import and a diff-before-publish).

### 2.6 Petpooja is four products, and only one keeps several devices working with the network down (round 2)
Petpooja's own pages describe four POS types **[W petpooja-pos-types, offline-pos, without-internet]**:

| Type | Architecture | Offline | Devices and scope | Left out |
|---|---|---|---|---|
| **Lite** | Cloud; an Android phone and a Bluetooth printer for kitchen tickets | None: it needs the internet to bill | One phone; no multi-terminal, no multi-outlet; a takeaway window or juice bar | Everything else |
| **Android POS** | Cloud; any Android device of 7 inches or more, no PC | Up to 5 days of offline billing | One billing device per outlet; aggregators, ONDC and Tally sync | Captain app, KDS, token display |
| **Electron POS** | Cloud-based desktop build (laptop, desktop or billing terminal); the page calls it the most common | Up to 5 days of offline billing | Table management with floor plans, inventory, captain app, KDS, kiosk, loyalty, 15+ delivery apps, ONDC; multi-terminal and multi-outlet | None stated |
| **Local** | A laptop or desktop **plus a Bridge Server**: a machine on the outlet's network that holds the data, installed by Petpooja's team | Billing continues on the local network if broadband drops; the page says up to 5 days before it reconnects | Table management, captain app, KDS, token display, UPI payments; up to 50 outlets from one system; for owners who want data on their premises or have poor broadband | Cloud-only extras: loyalty and WhatsApp CRM |

- **The server's stated hardware:** an 8th-generation Intel i5, 8 GB and a 1 TB SSD up to 25 outlets; an i7 or better, 16 GB or more and 1 TB or more up to 50; custom beyond **[W]**.
  Its page says the owner owns the bills, orders and sales data; menu and price updates, Zomato and Swiggy orders and app-setting changes still need the internet **[W]**.
- **What works with no internet (its "without internet" article) [W]:** billing and bill holding, KOT printing at the right station, table merge and settle, cash settlement and day-end,
  the saved menu. **What waits:** Swiggy and Zomato orders, card and UPI through the card machine (it routes through Petpooja's finance cloud; a direct UPI QR is suggested as a
  fallback without confirming it works offline), online-menu toggles and edits, cloud reports. Offline bills, KOTs and settlements sync without re-entry; the article tells the
  owner to check pending aggregator orders and to **confirm the bill numbering has no gaps** before closing the day. One Petpooja blog says billing runs fully offline when the
  software and database share a machine and up to 5 days when they are separate, another that Local "holds for 5 days": the descriptions do not match and I did not reconcile
  them **[Ws]**.
- **Lessons [J]:** (1) Single-device offline is what the cheap tiers sell; multi-device offline is a separately priced on-premises tier that Petpooja installs. (2) The leader
  trades cloud features (loyalty, WhatsApp) for local data in that tier: a hub and cloud features pull in opposite directions, so our hub must be a **relay and a cache**, not a
  fork of the product. (3) The "no gaps in bill numbering" line is the same Rule 46 discipline our cancel-keeps-the-number design already serves (4.7). (4) The hub hardware
  floor is a mid-range PC; our counter-PC-as-hub idea (D11) asks less of the owner and gives up the installer.

---

## 3. What the products do, feature by feature

For each area: what vendors do, what India adds, what our books need, and a **Comsky stance [J]**. "Not found" means not on the pages I could read.

### 3.1 Floors, tables and order types
- **Vendors.** Petpooja has a floor and table view and, in its captain app, a colour-coded table screen with customer names **[W]**. Odoo draws floors in an editor, shows live table
  status (occupied, booked, running late), lets a person drag one table onto another so both share one order until paid, and has a transfer-or-merge action **[W]**. GoFrugal
  tracks vacancy, steward assignment, table-wise orders and **floor-wise pricing** **[W]**. Restroworks lists table layout, reservations and queue **[W]**. Lightspeed's API lists
  floor plans and their tables and finds open checks by table **[W]**; its K-Series help adds the model: a floor plan holds up to 150 tables, each with a shape, a seat count (used to
  split by seat and for reports) and a name of up to four characters that must be unique across all floor plans; a floor plan can carry an order profile that its orders inherit,
  a prompt for a cover (guest) count, and printers for receipts and draft checks **[W]**; the article on moving or merging tables was not among the pages I retrieved.
  Square's order has fulfilment types for pickup, delivery and in-store, and a source channel **[W]**.
- **Order types.** Odoo can set different taxes for eat-in and take-out **[W]**; in India that is not needed for a stand-alone restaurant, because Circular 164 treats takeaway and
  door delivery as restaurant service at the same rate **[W circ164]**. What does differ by type is the **packaging charge** (takeaway and delivery) and the **delivery charge**.
  QSRs also need a **token number** (Petpooja sells token management in Growth) **[W]**.
- **Stance.** Order types: dine-in (table), takeaway (token), own delivery (address, phone, rider), and aggregator (channel, external order id). Tables are a floor, a name and
  seats; status is **derived** from open orders, never stored by hand. v1 has no reservation; v2 reservations are a Scale-tier feature at Petpooja, so they are not urgent.

### 3.2 The menu: variants, add-ons, combos, special pricing
- **Vendors.** Petpooja's captain app searches by name and by variation name, takes open-price items in add-on pop-ups, area-wise variation prices, three-decimal quantities, and scheduled
  categories **[W]**. GoFrugal lists recipes with **modifiers and variants** and menu engineering **[W]**. Petpooja's ONDC page claims multiple menus with combos, offers and discounts
  **[W]**.
- **Channel constraints (from UrbanPiper's public docs) [W]:**
  - **Swiggy:** up to 400 items per store (one line elsewhere says 1,000: confirm with Swiggy); up to four variant groups per item; option groups need distinct names; the same option
    cannot sit in two groups of one item; nested option groups are allowed; nutrition information is required for every item; descriptions are required for meals, platters, combos, boxes
    and thalis; non-veg items must say the meat; there is no "eggetarian" type (items with no food type read as non-veg); packaged goods carry a `packaged-good` tag.
  - **Zomato:** two levels of nested option groups; item names must be unique and **Zomato ignores digits, so "Chicken 65" and "Chicken 95" collide and the menu sync fails**; category
    timing windows start and end on the hour or half hour; an order-subtotal charge must be a percentage and a store may have only one; fixed charges apply per item quantity; tax titles are
    a fixed set (CGST, SGST, Municipality, Kerala); nutrition information is required; packaged goods are tagged.
- **Time and area pricing.** Floor-wise (GoFrugal) and area-wise (Petpooja) price lists are documented **[W]**. I found **no vendor page that documents time-of-day (happy hour)
  *pricing***; Petpooja's scheduling is about *visibility* of categories **[W]**. **Round 2:** Toast's help documents two ways to do happy hour, an item that reprices itself by
  time of day and day of week, or a separate menu with its own prices **[Ws]**; Odoo's own documentation builds happy hour on pricelists, and a recurring daily window comes from
  third-party modules (one listing says the built-in pricelists support date ranges but not recurring daily windows) **[Ws]**. I still found no Indian vendor page that says it prices by time
  of day, so happy-hour pricing stays **[unverified]** as an *Indian* market expectation and is a v2 item here (7.2).
- **Stance.** `MenuItem` (name, category, station, veg or non-veg, photo, short code, channel flags) holds **variants** (each a priced, GST-carrying sellable, e.g. Half and Full), **option
  groups** (required or not, minimum and maximum picks, price delta, optional recipe delta), and **combos** (fixed bundle, or slots with choices). Channel price lists reuse
  `PriceLevel` and `StockItem.prices` **[C types.ts:650, 522]**. A per-channel layer validates the platform rules above *before* a menu is published ("this name differs only by digits").
  Mixed-tax combos (a dish plus a packaged drink) are refused in v1: apportioning one price across two rates is a legal question I did not research **[J]**.
  The custom-fields **tracking variant** (`CF24`) is a quantity-only split with pooled value **[C .wt-cf custom-fields-spec AD.3]**: it fits "stock by colour", **not** a priced Half/Full dish.

### 3.3 Kitchen tickets, stations, display, courses
- **Vendors.** Petpooja assigns a printer to each cooking station and prints category-wise tickets **[Ws]**; KDS is a Growth-tier add-on and tickets can carry a display priority per
  item **[W]**. GoFrugal routes tickets to the right kitchen by cuisine, shows table number and special notes on a KDS, pushes real-time notices when an order changes or is cancelled, and
  carries on without internet **[W]**. Odoo routes by product category to preparation printers by IP, labels later courses **Pending** until someone presses **Fire**, and prints a cancellation
  ticket when a whole sent order is cancelled; its docs say nothing about a single-line change **[W]**. Restroworks adds a kitchen production system and demand forecasting **[W]**.
  **Round 2, the three Western vendors' documents [W]:** *Lightspeed* routes by **production centres**: an order profile decides which centres see an order, each centre needs a printing
  profile (without one it stays disabled), and the ticket types a centre prints are order tickets, course tickets and **transfer tickets**; courses after the first are not fired automatically by
  default, the server fires the next course, a sent course cannot be reassigned, and a reprint can cover the whole order, one course or one item. *Toast* splits a ticket per course when course
  pacing is on, marks held courses HOLD and shows them only on the expediter's screen, and offers four ways to fire (manual rules, the server's Send, Hold and Stay, automatic timing, and
  expediter control, the last two needing a KDS per a search summary of its setup article **[Ws]**); a voided item shows struck through as VOIDED on a KDS, and a setting decides whether a changes ticket prints. *Square* holds and fires
  courses from the check; fired courses print to kitchen or bar printers or show on a KDS, and coursing needs its Plus or Premium plan or a KDS subscription.
- **Stance.** A kitchen ticket (KOT) is an **immutable document**: station, lines, notes, who, when. A change is a *new* ticket of kind ADD, EDIT, CANCEL or **TRANSFER** (a table move, as in
  Lightspeed) that prints with a loud banner; a cancelled line stays on the screen struck through, not removed; a reprint prints **DUPLICATE**. Item to station routing lives on the menu item.
  Courses and Fire are v2; the first screen is one station per item. KDS is v2 (same ticket events, a screen instead of a printer).

### 3.4 Edits, cancellations, voids, comps: who may do what
- **Vendors.** Petpooja lets an owner configure what billers may do and assigns tasks **[W]** and updated "user rights" in the captain app **[W]**; Restroworks advertises anti-theft tools that
  let an owner act remotely, details unread **[W]**; GoFrugal notifies on change or cancel **[W]**; Odoo prints a cancellation ticket **[W]**. Aggregator rules add constraints: Zomato
  needs a cancellation reason and allows a customer-requested cancel only until dispatch; Swiggy allows a direct cancel only when the payload says so **[W]**; with one POS, voids
  did not sync to UrbanPiper and had to be repeated by hand **[Ws]**. **No Indian vendor publishes a void-approval matrix that I could read; round 2 found the pieces in three other places [W]:**
  *Toast:* voiding is a permission on a job or an employee, on top of an access mode; an employee without it gets another employee's POS access code or card swipe; reasons come from a
  list the owner configures (with none configured, staff just confirm); a check can be voided **only until its payment is captured**, after which it is a refund; and, per a search summary of its
  support pages, its Voided orders report breaks voids out by server and by approver **[Ws]**. *Lightspeed K-Series:* void reasons are set in the back office so cancellations use only reasons the owner allows (the
  approval and report pages were not retrieved). *Petpooja's own glossary:* a void keeps the bill number and records zero value with the reason, the user and the time; limit it to a
  supervisor login; **the person who benefits from a void must not be the one who approves it**; an order cancellation is logged separately from a bill void; once the day's sales are
  closed a credit note replaces the void (it adds that an e-invoice's IRN can be cancelled only within 24 hours); review voids daily at day close, and watch for one till carrying
  most of them.
- **What the controls must catch [J]** (my list, from how theft actually works at a till; the UP tax deck also records restaurants billing on a side desk and reporting only what that software showed **[W]**):
  a ticket fired and never billed; an item voided after the kitchen cooked it; a cheap item rung and an expensive one served; discount abuse; a bill printed, collected in cash, then cancelled;
  reprinted bills reused; a drawer opened with no sale; bills never entered at all.
- **Stance.** States and the least that may undo them: *before the kitchen ticket* (anyone, no trace); *after the ticket, inside a short grace window* (captain, with a reason);
  *after the grace window or after serving* (manager PIN, reason, and the line is tagged **comp** or **wastage**); *after the bill printed* (manager PIN; the invoice is **cancelled, not deleted**:
  the number stays in the series and the document is a shell with a reason, exactly as `voucherCancel` already does **[C voucherCancel.ts:1-12; types.ts:1065]**); *after payment* (credit note).
  Reasons use the existing cleaned reason text (`reasonText`, 200 characters **[C types.ts:1068]**). The manager override copies the credit-limit pattern: it is a warning that can be overridden, the
  override needs a reason, and the reason is kept and reported **[C creditLimit.ts:1-16]**. Reports: voids, comps, discounts and reprints by person, **KOT-versus-bill reconciliation** (every ticket
  line is billed, voided with a reason, or flagged), no-sale drawer opens. **Round 2 adds:** the reason list is configured by the owner (not free text only); the approver is recorded and
  must differ from the person who benefits; an order cancellation and a bill void are two events; **no Undo on a printed bill** (Undo deletes the voucher and leaves a hole in the series, so
  "take back" must route to Cancel) **[C undo.ts:118-119, via the audit]**; and a day-close screen lists the day's voids, comps, discounts and reprints for the manager before the Z report is
  accepted. **Honest limit:** rights are enforced on the client only today **[C access.ts:12-14]**, and the role model has no cancel, discount, override, refund, reprint or no-sale action (cancel counts
  as alter) **[C types.ts:681; audit section 9]**; v1 must say so on screen, and v2 needs server-side rights (a per-user table the cloud checks).

### 3.5 Split, merge, move
- **Vendors.** Odoo: split by selecting products into a sub-order that is paid on its own (the order needs two lines; a sub-order is paid before returning to the main one); the docs describe
  splitting by product, not by seat; merge or transfer to another table; drag-to-link for big parties **[W]**. Petpooja splits, merges, moves and edits bills at the table **[Ws]**. GoFrugal merges and
  splits tickets and converts a ticket to a bill **[Ws]**.
- **Law and books.** Each separately paid bill is its **own tax invoice** with a consecutive number (the repo already states the Rule 46 consecutive-serial reason **[C voucherCancel.ts:1-3]**).
  A split is therefore N Sales vouchers from one order. Discount, service charge and packaging apportion across them in paise with the largest-remainder rule the app already uses for
  GST slabs **[C CLAUDE.md, Phase 20-21 gotchas]**; round-off is per bill.
- **Stance.** v1: split by item and split equally; merge and move; no seat model. The order keeps an `orderRef` link on each Sales voucher **[C types.ts:1005]**.

### 3.6 Service charge, tips, packaging, delivery, discounts, coupons
- **Vendors.** Odoo records a tip as a product line, takes it at payment (even by a different method from the bill), and after payment only in the US; its docs say nothing on tax for a tip **[W]**. Zomato and
  Swiggy constrain how packaging and other charges are expressed (section 3.1) **[W]**. Petpooja configures packaging charges, store timings and discounts per outlet for the online menu **[W]**; coupons and
  loyalty points apply on the bill, including in the captain app **[Ws][W]**. Swiggy's own share of a discount is **not** in the order payload; discounts are set on Swiggy's side **[W]**.
- **The law, in short (section 4):** a discount shown on the invoice at or before the supply reduces the taxable value; one given later does not, unless it meets the Act's conditions **[W act s.15(3)]**.
  Packing and other incidental charges billed to the customer are part of the value **[W act s.15(2)(c)]**. Service charge is the contested one (section 4.4). **Round 2:** Toast separates a
  charge that is a **mandatory gratuity paid out to the check owner** from a **service charge the business keeps and counts in net sales**, and can apply a gratuity automatically by service area
  and party size; its tax options for service charges differ by country and the richer ones are in limited release **[Ws]**. That split is the distinction the Indian dispute turns on
  (voluntary payment to staff against a price component), so our charge record keeps *who the money belongs to* as a field **[J]**.
- **Stance.** One **charge rule** record (kind, percent or fixed, order types it applies to, GST treatment, **who it belongs to**, default state). Service charge defaults **off**; if on, it is a visible line
  the cashier removes with one key and the bill says it is voluntary. Packaging is on for takeaway and delivery by default (owner choice), in the supply's value at the supply's rate. Tips go to a **liability**
  ledger, never revenue, never in the GST base **[J; GST on tips is [unverified]]**. Discounts carry a reason and a role limit. **How these post is constrained by the engine** (a charge is a service
  line, a tip is a Receipt: 7.3).

### 3.7 Aggregators and ONDC
- **How the integrations work (UrbanPiper's public API docs, as a stand-in for the whole class) [W].** The POS creates each brand's **stores**; then a **menu** (categories, items, option groups, options, taxes,
  charges) is created and assigned to each store; aggregator orders arrive as an **order-relay webhook** to an endpoint the POS registers; the POS sends **status** changes back; rider status arrives by webhook; the POS can switch
  an **item**, an **option** or the whole **store** on and off. If an order is not accepted in time the aggregator may cancel it. Orders can be **auto-accepted**. Activation takes up to 48 hours (Monday to Friday) and a menu change takes five to ten
  minutes to show **[W]**. Round 1 did not retrieve the payload, status values and retry rules; round 2 did (next bullet).
- **UrbanPiper's order contract, read in round 2 [W order-components, order-relay, order-status-update, setting-up-webhooks, webhook-circuit-breaker, certification].**
  *Statuses:* Placed, Acknowledged, Confirmed, Food Ready, Dispatched, Completed, Cancelled, plus a Zomato-only customer-cancel request; orders only move forward (an earlier status is refused
  with a named error), Confirmed and Food Ready need Acknowledged first, and for an aggregator-delivered order in India only the aggregator can mark it Completed. *Cancelling:* an aggregator-delivered order
  can be cancelled up to Confirmed, a self-delivery order up to Completed; Zomato lists five reasons for merchant-delivered orders and three for its own riders; Swiggy allows a direct cancel only when
  the order says it can be rejected *and* the request asks for it, otherwise Swiggy phones the store and the POS must not retry; a cancel carries a reason code from a fixed list (item or option out of
  stock, store closed or busy, rider not available, outside the delivery radius, connectivity, total mismatch, invalid item or option, special instructions, unspecified). *Order body:* store id, order id,
  channel, state, type, times, customer (the phone is often a dummy), items with option adds and removes, price, quantity, taxes, charges and discounts at order and item level, payment entries, a
  subtotal, a total and a payable amount; **each tax entry carries a `liability_on` field saying whether the aggregator or the merchant owes it, and discounts carry an `is_merchant_discount` flag**;
  not every field is present for every aggregator. *Delivery to us:* the POS registers a webhook per event (order placed, status update, rider status, store and item toggles, menu publish and others);
  calls are **not signed** (the page says to put your own secret in a header); the endpoint must answer 2xx (a 422 is final); an event is tried **four times, about 1, 1 and 2 minutes apart, then dropped**,
  with a retry call to fetch missed orders; the **circuit breaker** switches all of a business's webhooks on a host off after more than 15 connection failures in a minute (3 second connect and 5 second
  read windows), and re-pushes the missed orders when it re-enables after one or three minutes, in whatever state they have reached. *Idempotency:* resending the status an order already has succeeds without
  telling the aggregator again; the status call is limited to 100 a minute. *Certification:* sandbox testing, then a sign-off form, about one to two days to review, no fee stated on the page. The page does
  not say whether the webhook URL must be HTTPS or public; I infer that it must be reachable from UrbanPiper's servers **[J]**.
- **Zomato direct [Ws].** Partner onboarding with an NDA and an IP whitelist, a partner-only API reference, webhooks for order relay, menu and serviceability, and a per-item stock call that does not need a full menu push; a
  "hybrid logistics" order says whether the restaurant (`self`) or Zomato (`partner`) delivers, and the POS must show it **[W urbanpiper-zomato]**. **Swiggy direct:** the only account I found is a former Swiggy product manager's of
  the 2017-era build (a generic partner API, staging accounts, a menu-sync API that cut changes from days to seconds) and no current public partner page **[Ws]**. Odoo's own docs route both through UrbanPiper and need an Enterprise
  licence, an UrbanPiper subscription and a reseller account with each platform; **Swiggy orders cannot be rejected from Odoo** **[W]**.
- **Petpooja's offer [W]:** one screen for all platforms, menu, packaging and offers pushed once, stock-out toggles from POS inventory, **platform-wise reports reconciled to payout sheets**, per-platform commission, surplus charges, margin
  and taxes. GoFrugal lists Swiggy, Zomato and ONDC with inventory synced to the POS **[W]**.
- **ONDC [Ws].** An open network with buyer apps, seller apps and a gateway; F&B was among its largest non-mobility categories by 2024 reports (national share of food orders reported near 3%, a city like Bengaluru far
  higher); every commission comparison I saw came from a vendor blog; how GST under the e-commerce operator rule applies when buyer and seller apps are different firms is **not settled by anything I read**.
- **Stance.** v1: **manual aggregator order** (channel, order id, amount as per the aggregator's invoice, no output GST charged by us, paid-by-aggregator receivable) and a **payout reconciliation** screen on the bill-wise
  machinery the app already has **[C billwise.ts, brs.ts]**. v2: middleware integration (menu push with the channel rule checks above, order relay to a "pending customer orders" queue with accept, reject and hold,
  status push, stock toggles). v3: direct partner APIs and ONDC.
- **What the contract means for our architecture [J].** (1) The receiver must be an address UrbanPiper's servers can reach that answers in a few seconds: an outlet's PC cannot be, so **the cloud backend must be**,
  and today it is HTTP-only on one VM **[C HANDOFF.md]**. (2) A webhook can arrive twice, late, or in any state after a circuit-breaker re-push, so the aggregator's order id is the **idempotency key** and intake
  writes a **non-posting pending-orders table**; the counter accepts, rejects or holds an order, and only then does the POS writer make the bill. (3) The REST API is not the door: it writes raw vouchers with no
  numbering or balance check **[C restPolicy.js:18-24, via the audit]**. (4) Each tax entry's `liability_on` decides whether our bill carries output GST: "aggregator" means none is posted (4.3). (5) The counter
  needs a **late-orders view** ("3 orders arrived after the retry window") and a visible **store snooze**, because the missed-order and circuit-breaker rules mean orders can wait unseen. (6) Our own endpoint
  should carry a shared secret in a header (the contract's advice) and never log it: the repo's logger redacts only a few key names **[C CLAUDE.md, MCP GOTCHAs]**.

### 3.8 Delivery and riders
- **Vendors.** Petpooja supports third-party rider apps (its FAQ names Dunzo, Shadowfax and Pidge) and does not explain assignment or tracking **[W]**. UrbanPiper delivers rider assigned, arrived and picked-up status by webhook **[W]**.
  Zomato's hybrid logistics, rider OTP and bag-return OTP flows are in the Rider Status Update webhook **[W]**. UrbanPiper's index also lists self-delivery rider management and live-tracking
  endpoints, which I did not read **[W]**. The 57th Council (8 Oct 2026) recommended 5% without credit for delivery services supplied through an e-commerce operator; whether that reaches a restaurant's
  delivery fee I could not tell (4.11) **[Ws]**.
- **Stance.** v1: own delivery is an order type with address, phone, rider name and cash-to-collect. v2: rider assignment and status. v3: dispatch and third-party rider APIs.

### 3.9 QR, self-ordering, online ordering, captain app
- **Vendors.** Petpooja sells QR ordering and a digital menu as an add-on **[W]**; the captain app is described as an Android app on Petpooja's fine-dine page and as an iPad-class app on the App Store, takes orders offline, syncs
  several devices with the POS, and pays by UPI, wallets and contactless card **[W]**. GoFrugal has a QR table menu and a steward app **[W]**. Odoo has QR menus, kiosks and self-order **[W]**. Restroworks has a kiosk, QR, a branded app and website **[W]**. Limetray
  sells an ordering site and app **[W]**. DotPe sells QR ordering and a store **[W]**.
- **Stance.** The captain role is a phone or tablet job; the shell cannot be used on a phone today and the VM is HTTP-only, which blocks install, push and durable storage **[C mobile-market.md findings 6, 7]**. Public QR ordering needs a public
  HTTPS origin, abuse limits and a rule that a diner sees only their own table. Both are **v2 and depend on the mobile and TLS decisions**, not on this document.

### 3.10 Recipes, ingredient stock, wastage, purchase, central kitchen
- **Vendors.** GoFrugal: a recipe bank that maps ingredients in different units, food-cost reports with variance, recipe cost from end-of-day usage and wastage, substitutes, expiry alerts, day-end physical audits to measure
  pilferage, minimum and maximum levels with lead times, production planning and BOMs **[W][Ws]**. Petpooja: inventory, multi-stage recipes and a central-kitchen module for consistent menus across outlets **[Ws]**. Restroworks: a central
  kitchen ("Base Kitchen"), recipe management, supply chain; reviewers call its inventory **complex for semi-processed items** **[W]**.
- **A tax-side hint.** The state training deck tells officers to ask for a **separate inventory of goods used in cooking and goods sold as they are**, because a restaurant that also sells packaged goods raises a separate tax invoice for them and
  must reverse credit on common inputs **[W updeck]**. A restaurant item therefore has a **class**: dish (cooked), ingredient (stocked), resale goods (sold as bought, with a rate and an MRP).
- **Stance.** v1: dishes only, **no ingredient stock** (the cheapest true thing: sales, GST, cash). v2: ingredient and recipe stock with **day-end backflush**: one Stock Journal per outlet per day computed as sold quantity times recipe, then a physical count
  (the Physical Stock voucher exists **[C types.ts:13-14]**) to price the variance; wastage is a Stock Journal to a wastage head; purchases are ordinary Purchase vouchers, with kg-to-gram conversion through compound units **[C types.ts:425-435]**.
  Live depletion on sale is v3 (it makes every sale touch many stock positions). Sub-recipes need the **multi-level BOM that is not built** **[C CLAUDE.md]**. Expiry and FEFO are not enforced today **[C batch-serial-code-audit.md:109]**.
  **No-credit purchases:** a 5%-no-credit restaurant must not take input credit; the app's `itcBlocked` purchase-ledger flag and its "post the blocked tax to cost" helper are the nearest mechanism **[C itc.ts:8-30; types.ts:339]**, but they mean "blocked under section 17(5)", and
  whether a 5% restaurant's purchases should read as "ineligible credit" in GSTR-3B 4(D) is a CA question **[unverified]**.

### 3.11 Multi-outlet, central kitchen, loyalty
- **Vendors.** Restroworks manages menus across regions with real-time sync and a Brand, Format, Cluster, Deployment hierarchy for reports **[W]**. GoFrugal: centralised purchase and inventory, stock transfers between outlets, franchise
  tools **[W]**. Petpooja: a central-kitchen module and a Marketing Manager (CRM, loyalty, virtual wallet, feedback QR, reputation, WhatsApp credits) **[W]**.
- **Stance.** An outlet needs its own **FSSAI number**, bill series and often GSTIN **[J]**. Use one company per GSTIN with the outlet as a godown and cost-centre pair, FSSAI stored on the outlet; separate companies when the GSTIN differs. Menu
  master shared across outlets with outlet overrides is v2; central kitchen indents are v3; loyalty is v3. A **stored-value wallet** may fall under RBI prepaid-instrument rules, which I did not research **[unverified]**: take counsel before building it.

### 3.12 Reports and the day-end
- **Vendors.** Petpooja: 90+ reports, staff reports (sales, tables served, cash-register activity), item-wise tax, HSN-wise and category-wise tax **[W]**; Restroworks 200+ **[W]**; GoFrugal 350+ MIS **[W]**.
- **Stance.** Build on the Reports Center and the row filter **[C CLAUDE.md Phase 26]**. v1 list: item-wise, category-wise, hourly, day-wise, order-type and channel-wise, payment-mode, captain-wise, table turn, void and comp and discount and reprint audit,
  KOT-versus-bill reconciliation, GST summary through the existing return builders, and a **day-end (Z) report** with opening cash, tenders, expected and counted cash, variance, number range of invoices issued and number cancelled.
  v2: food cost and variance, aggregator reconciliation, outlet comparison.

### 3.13 Offline, hardware, receipts, payments, WhatsApp
- **Offline.** Petpooja's captain app runs without internet and keeps several handhelds in step with the POS **[W]**; GoFrugal continues order processing offline **[W]**; Limetray lists an offline mode **[W]**; Odoo's general docs say a session survives a
  short outage and syncs payments afterwards, without saying which restaurant functions survive **[W]** (2.6 narrows the Petpooja claim to its Local tier for several devices). Peak-time slowness is the review theme,
  so a **local-first POS that never waits for the cloud** is a differentiator **[J]**. **Round 2, Toast's offline mode [W][Ws]:** orders, tickets and receipts continue (printers on a wired network, a hardwired KDS if one device
  acts as the local hub, card payments held as Pending), but **devices do not share data, so an order taken on one terminal does not appear on another**, quick edits (86'ing an item, price overrides) are blocked,
  gift cards and loyalty do not work, online ordering stops, shifts cannot be closed or tips declared until reconnection, and reports lag; staff are told to stay on one device and never to sign out or uninstall during
  the outage. Square's general offline card payments run up to 24 hours (one hour on a Reader or Stand), the merchant carries the risk of declined or expired payments, and a sign-out or reset loses what is pending **[Ws]**.
  **[J]** The offline contract should be one sentence a cashier can read (what still works, what waits, what to do), and sign-out or clear-data must never discard an unsent bill.
- **Printing.** Thermal 58 or 80 mm tickets and bills, a Bluetooth option in the captain app **[W]**, and a cash-drawer kick are table stakes. The repo has no thermal layout, no ESC/POS and no kitchen ticket **[C: grep of `src/`; `print.css` has one 58 mm *table column*]**.
  Silent printing from a browser is not available; the Electron desktop shell can print without a dialog but I did not test a thermal printer **[unverified]**. **Round 2:** ESC/POS (Epson's command set)
  is the common language of receipt printers; its reference lists a drawer-kick command (ESC p) and marks the older raster-image command (GS v 0) obsolete in favour of newer graphics commands, so a driver should
  not assume the old one; a 58 mm roll prints about 48 mm (384 dots at 203 dpi) and an 80 mm roll about 72 mm **[Ws]** (my figure for 80 mm: 576 dots, from memory, **[unverified]**). **Regional-language
  tickets:** I found one vendor that lists Hindi besides English for kitchen tickets and none that lists Tamil or Marathi **[Ws]**; a thermal printer's built-in character tables are unlikely to carry Devanagari or
  Tamil glyphs **[unverified]**, so such a ticket is printed as a picture. The label engine already turns text and codes into bitmaps for its TSPL path **[C batch spec 13.3 to 13.6]**, so a KOT can take the same road,
  and the spec lists a bundled Devanagari face as not built **[C batch spec section 18]**: a v2 item with a real cost, not a v1 promise.
- **Payments.** UPI is the default tender; dynamic QR on the bill and contactless card limits appear in vendor material **[Ws][W]**. The repo can build a `upi://pay` link with an amount and encode it as a QR **[C upi.ts:1-25, qr.ts:1-4]**, but a printed QR is **not** a received payment:
  confirmation needs a payment provider or a soundbox (the audit adds that the link carries no transaction reference, so a bank-statement match is the control **[C audit 2.5]**). Card numbers must never reach the app (4.8) **[J]**.
- **WhatsApp bills.** Petpooja's Marketing Manager bundles e-bills with WhatsApp Business credits **[W]**. Sending needs a Meta business provider (the research README already lists that dependency **[C docs/research/README.md]**).
  Our `wa.me` link only opens the person's own app and cannot attach a file or carry an itemised receipt **[C share.ts, via the audit section 8]**. Meta's per-message price for a utility template in India is quoted
  at roughly ₹0.11 to ₹0.16 by two third-party blogs that disagree on the marketing rate and on a change due on 1 Oct 2026; I could not read Meta's own India rate card **[Ws]**: budget from Meta's table, not from this line.

---

## 4. GST and compliance for restaurants

> Every statutory statement below says what I read. **I am not a tax adviser and nothing here has been reviewed by one.** Rates and conditions must live in **dated data** (the app's dated GST history
> **[C gstRates.ts:1-20]**), never in code, and every row in 4.10 needs a CA's sign-off before the POS ships.

### 4.1 The rate and its condition
| Supply | Treatment | Evidence |
|---|---|---|
| Restaurant service **other than** at "specified premises", including takeaway | **5% (2.5 central + 2.5 state; 5 integrated)**. The table calls it a **mandatory rate** that cannot be replaced by the general 18% entry. Condition: credit of tax on goods and services used in the supply is **not taken** (the condition points to Explanation (iv), which I did not read; the CBIC FAQ's reading of it is in the common-inputs row below) | **[W rates]** item 7(ii) and its conditions column; the 2017 decision that stand-alone restaurants, AC or not, take 5% without credit and that parcels do too **[W gstc23]** |
| Restaurant service **at specified premises** (some hotel premises) | **18% with credit** (the residual entry 7(vi)) | **[W rates]**; **[W gstc23]** |
| "Specified premises" **from 1 Apr 2025** | A premises where, in the **preceding financial year**, a unit of hotel accommodation was supplied above ₹7,500 a day; or one for which a **registered person supplying hotel accommodation** filed a declaration between 1 Jan and 31 Mar of the preceding year; or one declared within 15 days of a registration acknowledgment. Opt-out by declaration on the same calendar | **[W ctr05]** (Notification 05/2025-CT(Rate), 16 Jan 2025). **Round 2, read from a copy of the CBIC FAQ (28 questions, Mar 2025) [W faq-copy]:** the declaration is **Annexure VII** (a registered hotel-accommodation supplier) or **VIII** (a new applicant, within 15 days of the registration acknowledgment) and **Annexure IX** opts out; it is filed **per premises**, between 1 Jan and 31 Mar, takes effect from 1 April, and **holds until opted out**; the status cannot change inside a year; the officer only checks completeness (self-assessment); a **mall owner cannot file**; a restaurant *inside* the declared hotel premises charges 18% with credit while one elsewhere in the same mall charges 5% without; and the revised definition also governs **catering** (Q23) |
| Can a stand-alone restaurant declare itself specified (to take 18% and credit)? | **No.** The CBIC FAQ says anyone not supplying or intending to supply hotel accommodation cannot file the declaration, and the form (Annexure VII) is itself headed as a declaration by a registered person supplying hotel accommodation. The Council's Sep 2025 recommendation of an Explanation is consistent with that; I did not read the notified wording and the POS rule does not depend on it | **[W faq-copy Q5]**; **[W annexure7]**; **[W gstc56]**; a vendor blog (18 Jul 2026) says the same **[W petpooja-gst-blog]** |
| Cloud kitchen, central kitchen, takeaway, door delivery | **Restaurant service**, 5% without credit, even if sold only by takeaway or delivery | **[W circ164]** para 3.3 to 3.4 |
| Ice cream parlour selling made ice cream, not cooking | **Goods, not restaurant service** (18% in the Oct 2021 circular). Goods rates moved on 22 Sep 2025; **the current ice-cream rate is [unverified]** | **[W circ164]** para 4 |
| Outdoor catering away from specified premises | 5% without credit under item 7(iv), with its own conditions; in 2017 the Council said 18% with credit | **[W rates]**; **[W gstc23]**. Out of scope for v1 |
| Hotel rooms up to ₹7,500 a day | Cut to 5% on 22 Sep 2025 per the 56th Council release (condition not read) | **[W gstc56]** |
| Hotel rooms above ₹7,500 | 18% | **[W rates]** residual entry |
| Credit on **common inputs** (a 5% restaurant that also makes other supplies) | Credit on inputs used **only** for 5% restaurant service is not taken; credit on inputs used **partly** for it and partly for other services (including restaurant service at specified premises) is **reversed as if the 5% supply were exempt** (the FAQ's reading of Explanation (iv), with the Rules 42 and 43 mechanics left to the CA) | **[W faq-copy Q17]** |
| Heading / SAC | Heading **9963** (accommodation, food and beverage services). Compliance sites give the six-digit SAC **996331** for restaurant service other than at specified premises (takeaway and door delivery included) and **996332** for specified premises; the notification table I read gives only the heading. An advance ruling is reported to hold that ready-made food bought in and sold over the counter is **not** restaurant service | **[W rates]**; **[Ws]** for the SAC and the ruling |

**History in one line [W gstc23, ctr05, circ167, gstc56; J]:** before 15 Nov 2017 the rate depended on air-conditioning and liquor on one account (12% for a non-AC or no-liquor restaurant, 18% for an AC or liquor one, with full credit; TaxGuru, 19 Nov 2017) while another contemporaneous report says 18% for both **[W taxguru][W businesstoday]**: **the two disagree, and the Council note I read does not describe the earlier rates**, so treat the pre-2017 rates as **[unverified]** and unneeded by the POS; from the Council's Nov 2017 decision (15 Nov 2017; one report says 1 Nov) stand-alone restaurants take 5% with no credit, the reason given being that restaurants had not passed earlier credit on to diners; from
1 Jan 2022 aggregator-routed restaurant service moved to the aggregator; from 1 Apr 2025 "specified premises" is read from last year's room values or a declaration; from 22 Sep 2025 room rates changed and (per a vendor, not the Council) food rates did not.
**Pending, not law:** the 57th Council's recommendations of 8 Oct 2026 (4.11). A search for a 2026 change to the restaurant rate itself found none **[Ws]**.

### 4.2 Composition scheme
- Restaurants are in the scheme's service carve-out: s.10(1) allows **up to 2.5% central plus an equal state share** (5%) of turnover for "supplies referred to in clause (b) of paragraph 6 of Schedule II", which is the restaurant limb **[W act s.10(1)(b)]**; the 2017 FAQ confirms 5% for restaurant service **[W compfaq]**.
- **Turnover limit, read in round 2.** Notification 14/2019-Central Tax (7 Mar 2019, in force 1 Apr 2019) lets an eligible registered person whose aggregate turnover in the preceding year did not exceed **₹1.5 crore** opt in, and
  **₹75 lakh** in Arunachal Pradesh, Manipur, Meghalaya, Mizoram, Nagaland, Sikkim, Tripura and Uttarakhand; it also bars a manufacturer of ice cream, pan masala or tobacco **[W ntf14]**. (The Act text opens at ₹50 lakh and lets the Government
  raise it **[W act]**; the 2017 FAQ's ₹75 lakh and the state deck's ₹1 crore for special-category states are older or different **[W]**.) I did not search for a later amendment, so the figure is **[unverified] after March 2019**: keep it in
  dated data. The separate ₹50 lakh service-provider scheme under section 10(2A) at 6% is not the restaurant route **[Ws]**.
- **Conditions I read:** no inter-state outward supply (s.10(2)(c)); no supply of goods or services not leviable to tax, which is why **alcohol breaks eligibility** (s.10(2)(b); the deck lists it **[W]**); no credit on purchases; a **bill of supply**, not a tax invoice; tax cannot be collected from the customer; quarterly return **[W act][W compfaq][W updeck]**. Because s.31(3)(c) makes a composition supplier issue a bill of supply instead of a tax invoice **[W act]**, the app's Bill of Supply print **[C CLAUDE.md Phase 16]** is the right output.
- **Through an aggregator?** s.10(2)(d) bars supplies through an e-commerce operator **required to collect tax at source under section 52** **[W act]**. Circular 167 says that for restaurant service under 9(5) the operator no longer collects that tax **[W circ167 Q1]**, so on my reading a composition restaurant is **not** barred. That is **inference, not a statement in the circular [J]**: ask the CA.
- **POS meaning:** company setting "Composition" turns off tax lines, prints "Bill of Supply", forbids inter-state and alcohol lines, and counts aggregator sales in turnover for the limit **[W circ167 Q4]**.

### 4.3 Aggregator orders: the e-commerce operator rule
Read from Circular 167/23/2021 (17 Dec 2021) **[W circ167]**, which implements Notification 17/2021 (18 Nov 2021) under section 9(5) **[W act s.9(5): the Government may name services whose intra-State tax the operator pays, as if it were the supplier]**:
1. From **1 Jan 2022** the operator is liable for GST on restaurant service supplied through it, **even for an unregistered restaurant**, and pays it **in cash**. It no longer collects tax at source or files GSTR-8 for that service; it still does for other goods and services (Q1, Q3, Q7).
2. **The operator issues the invoice** for the restaurant service. If the same order has other items (goods), the operator is advised to raise a separate bill for them, and the restaurant remains the supplier of those other items (Q8 to Q10).
3. The restaurant **reports** these supplies in **Table 8 of GSTR-1 and Table 3.1(c) of GSTR-3B "for the time being"**, and they count in its **aggregate turnover** for thresholds and any other purpose (Q4, Q11). **The portal's layout has since changed** (next bullet).
4. The operator is not the "recipient", so these are not reverse-charge inward supplies, and it need not reverse its credit (Q5, Q6).
- **POS meaning [J]:** a new **taxability class "tax paid by the operator under 9(5)"** on the sale: the value is posted, **no output GST is posted or printed**, and the return builders put it where the circular says. The repo already reports nil, exempt and non-GST value in Table 8 and 3.1(c) **[C gstNil.ts:1-5; gstr1.ts:670-711]**, so the reporting slot exists; the class and its tag do not. **A restaurant must never issue its own tax invoice to the diner on an aggregator order.**
- **Commission:** the aggregator's fee is its own service with its own GST, outside the 5% supply; a 5%-no-credit restaurant takes **no credit** on inputs, so the GST on commission is a **cost** **[J, from the no-credit condition in 4.1 and a vendor blog's statement W petpooja-gst-blog]**.
- **Where the supply is reported now (round 2) [Ws, from GST-software blogs and GSTN advisories; not read on the portal].** Since GSTR-1 gained Tables 14 and 15, a registered supplier reports supplies made through an operator **that pays the tax under 9(5)** in
  **Table 14(b)** (net of credit and debit notes), which auto-fills GSTR-3B **Table 3.1.1(ii)**; the operator reports its side in Table 15, which feeds 3.1.1(i); Table 14(a) is the older case where the operator collects tax at source. In practice this replaces the
  circular's "Table 8 and 3.1(c) for the time being". **A CA must confirm the current layout and that restaurant service sits in 14(b)** (the sources I found do not name restaurants). The repo's return builders know nil, exempt and non-GST value in the old slot
  **[C gstNil.ts:1-5; gstr1.ts:670-711]** and nothing about 14(b): it is a builder change, not just a tag.
- **The operator's own credit.** Circular 240/34/2024-GST (31 Dec 2024) restates that an operator paying under 9(5) need not reverse input credit for restaurant service and answers the question for the other 9(5) services **[W circ240]**: not a restaurant's concern, but it
  is why platforms keep absorbing the liability.
- **Pending (57th Council, 8 Oct 2026):** an amendment of section 9(5) so operators are liable on notified services "irrespective of business model" **[W trilegal commentary]**; recommendation only (4.11).

### 4.4 Service charge: two laws that disagree
- **Consumer law.** The CCPA's guidelines of **4 Jul 2022** (as summarised in news, PIB's page returned 403 **[Ws]**): do not add service charge by default or automatically; do not collect it under any other name; say clearly it is voluntary; do not refuse entry or service for not paying;
  do not add GST on it by folding it into the food total. A single judge of the Delhi High Court **upheld the guidelines on 28 Mar 2025** (W.P.(C) 10683/2022 and 10867/2022, 2025:DHC:2084) and fined the two industry bodies; they appealed; a division bench heard the matter in 2025, asked on 22 Aug 2025 why a charge was added when prices already exceeded MRP, and asked the GST
  department to attend; **I found no appellate decision up to June 2026 [Ws]**. The CCPA took suo motu action against **27 restaurants (Jan 2026)** and **41 more (Jul 2026)**, with penalties of ₹30,000 to ₹50,000, refunds, and, in two orders I read about, a direction to **change software-generated billing so default charges are not added** **[W etvbharat 10 Jan 2026][W moneylife 20 Jul 2026]**.
  A **25 Mar 2026 advisory** extends the logic to "LPG", "gas", "fuel recovery" and similar add-ons: menu prices must be final apart from taxes **[Ws]**.
- **The Chaayos order (14 Jul 2026; SCC Online, 21 Jul, updated 12 Aug) [W scconline].** The CCPA found the charge was **on by default, added by commands in the billing software to every bill** until 1 May 2025; held that relabelling it "Optional Service Charge"
  did not cure the earlier default; and directed the chain to refund the complainant, **immediately modify the billing software so no service charge or similar charge is added by default**, pay ₹50,000 and file a compliance report within 15 days. In Feb 2026 the
  CCPA directed refunds in other cases and restated the 2022 guidelines in a Barbeque Nation order **[Ws]**. The Delhi High Court's single judge (Mar 2025) held that service charge and tips are voluntary, that a mandatory service charge on food bills is contrary
  to law, that a voluntary tip is not barred but should not be added by default, suggested terms such as "voluntary contribution", and ordered each association to deposit ₹1 lakh **[W livelaw]**; the division bench was listing the appeals in May 2025 and
  asked about GST in August 2025 **[W][Ws]**, and the latest report I found (June 2026) still calls them pending **[Ws]**. **Design rule [J]:** the default of every charge is a *setting the owner chose and dated*, never a vendor default; an "Optional" label does not make
  a default optional.
- **GST law.** Section 15(2)(c) puts any amount charged for anything done in respect of the supply into its value **[W act]**, so a service charge billed as part of the restaurant's supply is, on its face, taxable at the supply's rate; a state tax deck presents a case where 10% service charge was taxed while the food was not, as an anomaly officers should look for **[W updeck]**. That deck is one state's training material, not law.
- **The collision.** The CCPA says no GST on service charge; the Act's valuation rule says the opposite for a charge that is part of the price. The division bench raised it and I did not find an answer **[Ws]**. **[J]** v1 ships service charge off, as a per-bill voluntary line, and with **no GST treatment preselected**; a CA decides before any outlet turns it on. A tip handed over freely is a different thing again (a liability, outside the price): **GST on tips is [unverified]**.

### 4.5 Packaging, delivery, takeaway
- Packing and incidental charges billed to the recipient are part of the value of supply **[W act s.15(2)(c)]**; takeaway and door delivery are inside "restaurant service" **[W circ164]**. **[J]** A packaging or delivery charge on the restaurant's own bill is taxed at the supply's rate (5%); I read no ruling on it **[unverified]**. On aggregator orders the aggregator's invoice governs.

### 4.6 Mixed bills: packaged goods, liquor, made-to-order goods
- A restaurant that sells **goods as bought** (a packaged drink, a bakery pack) raises tax on them at their own rate, may take credit on those, and reverses common credit: the state deck says a separate tax invoice and a separate inventory are expected **[W updeck]**. The CBIC FAQ's answer on common inputs (4.1) now confirms the reversal "as if the 5% supply were exempt" **[W faq-copy Q17]**; the Rules 42 and 43 mechanics are left to the CA. An advance ruling reported in search summaries holds that readily available food bought in and sold over the counter is not restaurant service **[Ws]**: so a packaged item is its own class on the bill (3.10), and a v1 bill that mixes classes needs the audit's separate-lines rule (7.3).
- **Alcohol is outside GST** and falls under state excise and VAT **[W petpooja-gst-blog][W updeck]**. v1 refuses alcohol; a Non-GST line type exists **[C types.ts:462]** but taxes and permits are state-specific.
- The repo's rule stands: **never report GST at a blended effective rate**; split per real slab **[C CLAUDE.md gotchas]**. The per-line **snapshot** of rate and HSN **[C types.ts:956-969]** is what keeps last year's bills intact when the 5% or 18% status changes.

### 4.7 The bill itself
- **Rule 46, read in round 2 from the CBIC's consolidated Rules as amended to 1 Jun 2021 (later amendments not read) [W cgstrules].** A tax invoice carries the supplier's name, address and GSTIN; a **consecutive serial number of at most 16 characters (letters,
  digits, hyphen and slash), in one or several series, unique for the financial year**; the date; the recipient's name, address and GSTIN if registered; for an **unregistered** recipient the name, address and state only where the taxable value is **₹50,000 or more**,
  or, below that, **if the recipient asks**; the HSN or SAC; description, quantity and unit; total value and the **taxable value after discount**; the rate and amount of tax by head; the place of supply for inter-state supply; the reverse-charge flag; and a
  signature (not needed on an electronic invoice). **The small-bill rule is in Rule 46's proviso, not Rule 54:** where the recipient is unregistered and does not want an invoice, a registered person may **not issue one** under s.31(3)(b) but **must issue a
  consolidated tax invoice at the close of each day** for all such supplies (the Act's text puts the line at ₹200 **[W act]**; round 1 had this as unverified). The number of HSN digits is set by notification: **4 digits up to ₹5 crore of previous-year turnover, 6 above,
  and a supplier up to ₹5 crore may omit the digit count on supplies to unregistered persons** (Notification 78/2020, from 1 Apr 2021) **[Ws]**; the SAC above is six digits anyway. A buyer who gives a GSTIN gets a full tax invoice: **GSTIN capture at the bill is a
  v1 field**. The 16-character cap is a design constraint for us: outlet code, counter, year token and zero-padded number must fit (the audit's `T1/26-27/000123` is 15) **[C audit section 4]**.
- **QR on a B2C bill.** Notification 14/2020-Central Tax (21 Mar 2020) requires a **dynamic QR code** on invoices to unregistered persons from a registered person with **aggregate turnover above ₹500 crore**, and treats a dynamic QR shown to the buyer on a digital
  display, with a cross-reference on the invoice, as compliant; the text says from 1 Oct 2020, and later relaxations are in search summaries only **[W ntf14-2020][Ws]**. No v1 customer is near it.
- **E-invoice** applies above ₹5 crore aggregate turnover (Notification 10/2023, from 1 Aug 2023) and **B2C is not handled by the IRP** **[C einvoice-gsp.md:71-73]**; Petpooja bundles 1,000 e-invoice credits only in its top Operations tier **[W]**. For most v1 customers this is out of scope; B2B bills over the limit reuse the existing e-invoice screens.
- **FSSAI, read in round 2 from the FSSAI's own PDF [W fssai-order].** The order of 8 Jun 2021 (file 15(31)2020/FoSCoS/RCD/FSSAI) requires every food business to print its **14-digit licence or registration number** on cash receipts, purchase invoices, cash memos
  and bills; it names restaurants, mithai shops, caterers and retail stores as the gap it closes; **no new document is needed** (a GST invoice can carry it); when an operator issues two documents the number goes on both; the only exemption is the GST e-way bill and other
  system-generated government documents; the 30 Sep 2021 order moved enforcement to **1 Jan 2022**. A 13 Sep 2022 drive order is commentary only **[Ws]**; I found no withdrawal. **v1 prints it, per outlet, on every bill.**
- **Consecutive numbers and cancellations.** The repo's cancel-keeps-the-number design is exactly right for a till **[C voucherCancel.ts:1-12]**, and Petpooja's own glossary teaches the same rule (3.4). The server issues per-series numbers with offline provisional
  numbers and a clash report **[C vchNumbering.ts:1-17; backend/src/numbering.js:1-30, sql/008; committed in `2cee555`, deployed 2026-10-23, never exercised signed-in on the VM per CLAUDE.md]**; one **voucher type per counter** gives each counter its own series. **But** a signed-in,
  online Accept waits up to 4 seconds for a server number before falling back, which for a series no other device writes is pure overhead on patchy shop internet; the audit proposes marking a counter's type device-owned and numbering locally (the cost: a replaced terminal must
  pull its history before it bills) **[C vchNumbering.ts:20, 79-82, via the audit section 4]**.

### 4.8 Payment-data rules
Round 2: from **1 Oct 2022** no party in the card chain other than the card issuer and the card network may store actual card data, and data stored earlier had to be purged, with tokenisation as the replacement (RBI circulars of 2021 and 2022 as summarised in search results) **[Ws]**.
I did not read how that applies to a POS that records a tender, and PCI scope was not researched. The only safe design for v1 stays: **no card number is ever entered, stored or logged; card payments are taken on an external terminal and recorded as a tender, with an approval code or the last four
digits at most [J]**; UPI by QR. The audit adds that the UPI link the repo builds carries no transaction reference, so nothing confirms that a payment arrived: the controls are a reference per tender and a bank-statement match **[C audit 2.5]**. Any card-on-file, wallet or provider integration needs a cited read of
the RBI rules and the payment provider's own terms first **[unverified]**.

### 4.9 What this means for the POS settings **[J]**
A restaurant company profile with: scheme (Regular or Composition), premises class (standard 5% or specified 18%), rate effective dates, service-charge posture (off), packaging default, FSSAI per outlet, GSTIN capture on, and aggregator class on. Changing any of them is a dated, audited change.
**Customer contact data (round 2).** A phone number or e-mail captured for an e-bill or loyalty is personal data. The Digital Personal Data Protection Rules, 2025 (notified Nov 2025) are being phased in; the notice-and-consent duties are reported to start on **13 May 2027** (an 18-month period counted from
publication, from law-firm commentary, not from the Gazette) **[Ws]**. Cheap to do now, costly later: a purpose line beside the phone field, the consent stored with the number, and a way to delete it **[J]**.

### 4.10 What a CA must sign before v1 ships (revised in round 2: items marked "narrowed" had evidence added, none is closed)
1. The treatment of a **voluntary service charge** and of **tips** (taxable or not, under which head), and the wording on the bill.
2. **Packaging and delivery charge** at the supply's rate on the restaurant's own bill.
3. Whether a **composition** restaurant may sell through an aggregator under 9(5), and how those sales count.
4. The **current composition turnover limit** (narrowed: Notification 14/2019 says ₹1.5 crore, ₹75 lakh in eight named states; any later amendment unchecked).
5. How a **5%-no-credit** restaurant records **input GST** (narrowed: the CBIC FAQ says no credit on exclusive inputs and reversal as if exempt on common ones; what GSTR-3B shows, and Rules 42 and 43 mechanics, are open) and the GST on **aggregator commission**.
6. **Mixed bills** (packaged goods on a restaurant bill; narrowed by the FAQ's common-input answer and the advance ruling on over-the-counter food).
7. The **small-invoice and consolidated-invoice** conditions for counter sales (narrowed: Rule 46's proviso says a daily consolidated invoice; the amended text after 1 Jun 2021 was not read), and what a B2C bill must carry.
8. The **current table** in GSTR-1 and GSTR-3B for 9(5) supplies (narrowed: 14(b) and 3.1.1(ii) per search summaries) and whether restaurant service is reported there.
9. ~~Whether a stand-alone restaurant can be "specified premises"~~: answered by the CBIC FAQ and Annexure VII (it cannot); a CA need only confirm.
10. The current **ice-cream** and packaged-goods rates (changed on 22 Sep 2025).
11. **After notification only:** the 57th Council's same-line-of-business credit for restaurant and catering services, the 5% treatment of platform delivery services, and the section 9(5) amendment (4.11).
12. Whether the **dated defaults** the POS ships (every charge off, every class owner-chosen) satisfy the CCPA's reading in the Chaayos order (4.4).
13. **Comps, wastage and staff meals**: whether a free meal carries GST or a credit reversal, and under which head a staff meal is booked (a CA question I did not research).

### 4.11 Recent and pending, as of 25 Oct 2026 (none of this is built or assumed)
| Item | What I read | Status and my reading |
|---|---|---|
| 57th GST Council, **8 Oct 2026**: limited input credit "in the same line of business" for **restaurant and outdoor catering services**, hotel accommodation up to ₹7,500 a day and gym or fitness services; outdoor catering credit unblocked under s.17(5) | Law-firm and news commentary; the PIB release returned 403 **[W trilegal][W caclubindia][Ws]** | A **recommendation**; no effective date given for the s.17(5) items. The commentary says it follows the treatment already given to passenger transport, tour operator and vehicle rental services; **it does not say whether the no-credit condition on the 5% restaurant entry changes, and I could not tell [unverified]**. Keep ITC rules in dated data; do not hard-code "5% means no credit" anywhere but the rate history |
| Same Council: **5% without credit** for delivery services (other than courier and postal) supplied through an e-commerce operator, and for delivery of goods ordered through one | **[Ws caclubindia]**, which says food is not named | Recommendation. Whether it reaches a restaurant's own delivery fee or an aggregator's delivery fee is **unclear [unverified]**; matters for 3.8 and the aggregator commission note in 4.3 |
| Same Council: amend s.9(5) so operators are liable on notified services "irrespective of business model" | **[W trilegal]** | Recommendation; no effect on the circular's reading today |
| Refunds of accumulated input-service credit from credit availed on or after 1 Nov 2026; return-form changes from the April 2027 return; faceless administration from FY 2027-28 | **[W trilegal]** | Not restaurant-specific; listed so nobody is surprised by April 2027 return changes |
| CCPA suo motu actions on default service charge (27 restaurants Jan 2026; 41 in Jul 2026; Barbeque Nation; Chaayos 14 Jul 2026) and the Delhi High Court appeals | 4.4 | Live enforcement; the POS default rule in 4.4 follows from it |
| DPDP Rules 2025: consent managers 13 Nov 2026; core duties 13 May 2027 (reported) | 4.9 | Capture consent from day one |

---

## 5. Reviews and complaints, by theme

| Theme | Where it shows up | What it means for us **[J]** |
|---|---|---|
| **Slow or glitchy at peak** | Petpooja 1-star (19 Aug 2025) and the page's own cons; Restroworks cons (peak-time slowness); Limetray lag (2020) **[W]** | Festival-day peaks are the test. Local-first, no network wait on the add-item path, KOT print independent of the cloud, a published performance budget. Test at several thousand open and closed orders per outlet |
| **Support is uneven or hard to reach** | Petpooja (2020, 2024, 2025); Restroworks (2022, 2023); Torqus (2020 to 2021, with a cluttered interface, complex accounting workflows and an app that kept logging out); Limetray (one Capterra review); Toast, slow major-issue escalation **[W][Ws]** | Do not promise 24 by 7 unless staffed. Ship remote diagnostics, an in-app "send logs" and an offline-capable POS that does not need support to keep selling |
| **Menu changes are manual or slow** | Petpooja ("manual", cumbersome); aggregators need a publish after every change and 5 to 10 minutes to show **[W]** | Menu grid, Excel import, bulk price change, per-channel price list, a diff-before-publish screen |
| **Inventory and recipe gaps; semi-processed items** | Petpooja and Restroworks reviewers **[W]** | Do not claim food costing in v1; build multi-level BOM before v2 recipes |
| **Oversold bundles, value, hardware sold on commission, hidden add-ons** | Petpooja 1-star **[W]** (the hardware claim is unverified); Toast hardware, add-ons and contract terms **[Ws]** | Published prices; standard printers and Android devices; no lock-in; full export |
| **Contract lock-in, processor lock-in** | Toast **[Ws]** | No payments tie-in; the app is the books, not a payments business |
| **Aggregator fragility** | Voids not synced with UrbanPiper from one POS; manual item-ID mapping on Swiggy; Swiggy orders not rejectable in Odoo **[Ws][Ws][W]** | A reconciliation view and clear "this action must also be done on the platform" prompts |
| **Mobile weakness** | Limetray "no mobile support" (2021); Petpooja wants more mobile dashboards **[W]** | Phone is a v2 job on the mobile track |
| **Refunds and billing disputes** | Petpooja (one 2026 review); Toast **[W][Ws]** | Boring billing hygiene |
| **Compliance exposure** | CCPA orders to change billing software; the Chaayos order names "commands embedded in the billing software" **[W]** | Defaults matter more than features; every charge is an owner-dated setting (4.4) |
| **Syncing trouble on poor connectivity; multi-device offline is a premium tier** | A G2 reviewer on poor-connectivity sync **[Ws]**; Petpooja's own four POS types and the Bridge Server **[W]**; Toast's offline mode where devices do not share orders **[W][Ws]** | Say plainly what works offline and on how many devices; build the hub before promising a captain app (7.4) |
| **Add-ons that should be in the box** | KDS and QR ordering as extras on Petpooja (a competitor's blog and Capterra summaries) **[Ws]** | Decide the bundle in public and keep KDS out of the staircase |

**Rating caveat [J].** SoftwareSuggest shows Petpooja at 4.7 from 157 reviews and Restroworks at 4.8 from 94 (with feature and ease sub-scores of 2.7 that the page does not explain); Techjockey shows Petpooja 4.2 from 50 and Limetray 4.6 from 27, with reviews from 2020 to 2022; GoFrugal 4.0 from six reviews and, per a search summary, 4.4 from 198 on Capterra **[W][Ws]**. Vendor-invited reviews skew high and small samples decide nothing: use the complaints as a checklist.

---

## 6. What Comsky has today, against what a restaurant needs

| Need | Today **[C]** | Gap |
|---|---|---|
| A bill that posts to the books with GST, round-off and a party | `Voucher`, `InventoryLine` with GST snapshot, Round Off ledger (`types.ts:944-1049`; `gst.ts:152`) | None for the bill; **no open-check concept** |
| Non-posting pre-document | Sales Order is non-posting and numbered (`types.ts:35`); `orderRef` links a sale to an order (`types.ts:1005`) | A numbered voucher is the wrong home for a check that changes dozens of times an hour; use dedicated, append-only tables |
| Dated GST rates and per-line snapshot | `gstRates.ts` dated history; `InventoryLine.gstRate/hsn` | Add a restaurant premises class and a 9(5) taxability class |
| Cancel keeping the number, with reason | `voucherCancel.ts`, `VoucherCancellation` | Reuse as is; add "after print" flow and manager PIN |
| Roles and rights | Operator model with rights by area; **client-side only** (`access.ts:12-14`) | Server-side enforcement; per-action rights for void, comp, discount, reprint |
| Override with a recorded reason | Credit-limit override pattern (`creditLimit.ts`) | Copy the pattern |
| Outlet, kitchen, bar | `Godown` (`types.ts:454`), `CostCentre` (`:640`), `Company` per GSTIN | Decide the outlet model; FSSAI on the outlet |
| Price lists per channel | `PriceLevel`, `StockItem.prices` (`:650, :522`) | Time-of-day schedule (v2) |
| Units and weights | `Unit` with decimals and compound units (`:425-435`) | None |
| Recipes | `StockItem.bom` (`:523`) read only by Manufacturing (`manufacturing.tsx:105,124`); no multi-level BOM | Day-end backflush and sub-recipes (v2) |
| Stock count | Physical Stock voucher (`types.ts:13`) | Reuse for variance |
| Server numbering, offline provisional numbers | `vchNumbering.ts`, `backend/src/numbering.js`, `sql/008`: committed (`2cee555`) and deployed 2026-10-23, never exercised signed-in on the VM | One voucher type per counter; a signed-in Accept can wait up to 4 s (`NUMBER_TIMEOUT_MS`), so mark a counter's type device-owned and number locally |
| UPI QR | `upi.ts`, `qr.ts` | Payment confirmation not built |
| Thermal receipt, KOT print, cash drawer | **None** | New output layer |
| Bill of supply | Phase 16 print | Reuse for composition |
| FSSAI, GSTIN capture at counter | No field | Add |
| Table, floor, KOT, station, charge rule, day close | **None** | New, section 7.3 |
| Local hub for several offline devices | **None**; cloud sync only (`sync/sync.ts`) | The biggest gap (section 8) |
| Batch and expiry | Hidden, no FEFO, no expiry check today (`batch-serial-code-audit.md:109`); now **specified** in `docs/design/batch-serial-labels-spec.md`, its pure engines merged in `5aee765` and wired into no screen | Only matters for v2 raw materials; read that spec, do not redo it |
| e-invoice for B2B over ₹5 crore | Exists (`einvoice-gsp.md`) | Link from a settled bill |
| A dish as a stock item | `StockItem.isService` (`types.ts:525`) is read only by the e-invoice, e-way bill and inward-kind code; the stock engine does not read it; no no-stock flag | A stockless rule in the engine (D9), or every dish raises the negative-stock prompt (`VoucherScreen.tsx:2490-2503`) |
| Time, terminal, cashier and shift on the bill | None on `Voucher` (`types.ts` voucher block; the change-log row carries `ts` and operator but an alteration overwrites it) | A `Voucher.pos` stamp (the audit's lane C), needed for the Z report and the audit reports |
| A held check | None: an Optional voucher burns a number and is left out of Table 13; a Sales Order is numbered | The restaurant `Order` with its events is the hold; no number until tender |
| Tips, charges and split tenders on one bill | Any non-tax, non-party leg of a Sales voucher counts as taxable (`gst.ts:161-177`) | Charges as service lines, a tip as a Receipt, split tender as S1 or S2 (7.3) |
| The date a counter bills on | The working date defaults to the last voucher's date (`store.ts:357`) | Device clock, shown, never carried (the audit's rule) |
| Per-action rights | A role is areas plus create, alter, delete; cancel counts as alter (`types.ts:681`; `access.ts`) | Void, comp, discount, reprint and no-sale rights, enforced server-side (audit lane F) |
| Sales ledger by category | One sales ledger per bill (audit 1.1) | The P&L shows one sales line; item and stock-group reports carry the split; a later "sales ledger by group" item |
| Silent thermal print and drawer | `window.print()` on A4, A5 or Letter; the Electron bridge exposes backup, PDF save and share only | The device bridge (D10): ESC/POS emitter, listed printers, drawer kick |
| Customer contact for e-bill and loyalty | Customers are ledgers; no consent field | A light customer record with a purpose line and consent (4.9) |

---

## 7. Recommendation **[J]**

### 7.1 Positioning
Build the restaurant mode as **"Counter and table service on books you can hand your CA."** Win where a single-outlet or small-chain owner already has a Tally-using CA, wants correct GST (5% no credit, composition, the aggregator rule),
wants cash and UPI tied to the ledger, and is annoyed by add-on staircases. Do **not** chase chains, hotels or enterprise KDS in v1.

### 7.2 What ships when
**v1 (single outlet, one counter device, optional extra devices while online)**
0. **Foundation shared with the other POS modes (the audit's lanes A to C and E; not restaurant-specific and not costed again below):** the hot-path fixes so a bill's save and each keystroke stop scaling with the book; a POS writer that builds the same `Voucher` (Bulk Entry is the
   precedent); `Voucher.pos` (time, terminal, cashier, shift, tenders) and one additive Dexie bump; a device-owned numbering series per counter; the **stockless item rule** (D9); a device-clock date; holds as local drafts, never Optional vouchers; the device bridge for silent print, printer
   list and drawer. Without it the restaurant pack cannot start; with it, retail and pharmacy start too.
1. Menu: categories, items, **variants**, **option groups**, simple **combos**, short codes, veg or non-veg, station routing; grid and Excel import; channel price lists (dine-in, takeaway, delivery, aggregator).
2. Order types: dine-in with a table, takeaway with a token, own delivery with address and rider, **manual aggregator order**.
3. Floors and tables with derived status; move, merge.
4. **KOT**: per-station thermal printing (80 mm), ADD, EDIT, CANCEL and TRANSFER banners, DUPLICATE reprint; no KDS. Printed through the device bridge from the counter PC.
5. Bill and settlement: hold and settle, multi-tender (**one tender: the party is the tender ledger; mixed tender: the Sales voucher plus same-transaction Receipts**, the audit's shapes S1 and S2; credit to a customer ledger), UPI QR on the bill, **split by item or equally**, discount with reason and role limit,
   optional charge lines (packaging, delivery, **service charge off by default**) **as service lines with their own taxability**, a tip **as a Receipt to a liability ledger, never a leg of the bill**, round-off, GSTIN capture, FSSAI printed, Bill of Supply for composition, 58 and 80 mm bill, share by WhatsApp link.
6. Controls: roles (captain, cashier, manager), **manager PIN override** with an owner-configured reason list and the approver recorded (never the beneficiary), cancel-not-delete, **no Undo on a printed bill**, KOT-versus-bill reconciliation, void, comp, discount and reprint reports, **day close** with cash count and a
   voids-and-comps review; a fast cashier switch by PIN (a PIN check is 260 ms or more where WebCrypto is missing, as on the plain-HTTP site **[C audit section 9]**), because staff turn over.
7. Posting: each settled bill becomes one Sales voucher (one voucher type per counter series) with `orderRef`; aggregator orders post with the 9(5) class and the payout reconciliation on bill-wise.
8. Reports: section 3.12 v1 list.
9. A **keyboard-first counter screen** (type a short code and a quantity) **and** a touch grid, both on the same state; touch is not an afterthought in a restaurant.

**v2**
- Local **hub** for several devices in one outlet (7.4); captain app on phone and tablet (mobile track); **KDS**; courses and Fire; reservations.
- **Middleware aggregator integration** (a cloud webhook receiver with idempotent intake into a pending-orders table, the accept, reject and hold queue, status push, stock toggles, menu push with the channel rule checks, a late-orders view) and a reconciliation report; partner applications to Zomato and Swiggy. **Needs TLS and a public hostname first.**
- **Recipes and ingredient stock**: multi-level BOM, day-end backflush Stock Journal, variance on a physical count, wastage, purchase.
- **QR table ordering** and a branded ordering page (public HTTPS origin first).
- **Server-side rights**; time-of-day pricing (happy hour; I found no Indian vendor page for it, 3.2); customer profiles with consent; outlet comparison; WhatsApp bills through a provider; regional-language tickets as pictures (3.13).

**v3**
- Central kitchen and indents; multi-outlet menu inheritance and franchise; loyalty and wallet (after counsel on stored value); direct Zomato, Swiggy and ONDC; kiosk; waiter-call hardware; rider dispatch; forecasting; liquor with state excise and VAT.

**Rough size [J, low confidence; two engineers, no external dependency waits; round 1's figures, kept, and now stated to EXCLUDE the shared foundation of item 0, which neither the audit nor this document estimates]**

| Slice | Engineer-weeks |
|---|---|
| v1 data model, order and KOT events, menu with variants and options | 5 to 7 |
| v1 counter and table UI (keyboard and touch), settlement, split, posting | 5 to 7 |
| v1 print layer (KOT and bill, thermal, drawer): an ESC/POS emitter and job kind on the shared bridge, plus the hardware test matrix; the bridge itself is the audit's lane E and the batch spec's T-7a, so only the emitter and the receipt model are counted here | 2 to 3 |
| v1 controls, day close, reports, GST profile, tests | 3 to 4 |
| **v1 total** | **about 15 to 21** |
| v2 hub and captain app | 8 to 12 (depends on the mobile track) |
| v2 middleware integration | 6 to 9, plus partner calendars outside our control |
| v2 recipes and backflush | 4 to 6 (multi-level BOM included) |
| v2 QR ordering | 4 to 6 (after HTTPS and a public origin) |

### 7.3 How the data model differs from retail POS

| Area | Retail POS | Restaurant POS |
|---|---|---|
| Unit of work | A **cart** that lives for a minute and becomes a bill | An **open check** that lives for hours, takes several kitchen tickets, moves tables and splits |
| What is sold | Stock items with barcode, batch, serial, MRP | **Menu items** with variants, option groups and combos, mostly cooked to order; some **resale goods** |
| Stock effect | Direct, at sale (quantity out of a godown) | None at sale in v1; ingredient depletion by **recipe**, usually at day-end; wastage and variance matter |
| Catalogue size and change rate | Tens of thousands, scanned | Tens to hundreds, **edited constantly**, and pushed to channels with their own rules |
| Identity of a line | Item (and batch or serial) | Item + variant + chosen options + course + station + note |
| Time | A bill has one time | A line has an **ordered**, **fired**, **served** and **billed** time; the kitchen sees the first two |
| Kitchen | None | **Stations**, **tickets**, **display**, **courses**, change and cancel tickets |
| Discounts | Price, scheme, customer | Discount **with reason**, comp, wastage, staff meal, coupon, channel discount |
| Charges | Rare | **Service charge**, packaging, delivery, tip; GST rules differ per charge |
| Bill splitting | Rare | **Routine**: split, merge, move; each paid bill is its own tax invoice |
| Returns | By bill, with stock back in | Almost none; a cancelled bill, a credit note, or a **comp** |
| Parties | Customer optional | Table, guest count, captain, platform, rider |
| Peak shape | Steady; scanner-fast | **Rush** (lunch, festival, match day); many people touching one check |
| Offline unit | One till | A **room of devices** that must agree on every table |
| Channels | Online store, marketplaces | **Aggregators** (tax paid by the operator), QR, own site, phone |
| Compliance | Legal metrology, MRP, GST | **5% without credit**, composition, 9(5), service charge law, **FSSAI** |
| Reports | Item, category, margin, stock | Item, category, **hour**, **captain**, **channel**, void and comp audit, food cost, day-end |

**Proposed v1 entities (names indicative).**
`Outlet` (godown + cost centre + FSSAI + printers). `Floor`, `DiningTable` (floor, name, seats). `MenuItem` (category, station, flags, short code, channel flags) with `variants[]`
(each pointing to the accounting `StockItem`: rate, HSN, tax snapshot), `OptionGroup{min,max,options[{label,delta,stockItemId?}]}`, `Combo`. `Station` (printer or display).
`Order` (outlet, channel, order type, tables, guests, captain, token, `ext{platform,id}`, status). **`Kot`** (order, seq, station, lines, kind ADD|EDIT|CANCEL|TRANSFER, by, time; **immutable**). `KotVoid` (kot line, qty, reason, by, approvedBy, kitchen state at the time).
`Settlement` (the Sales vouchers it created, tenders, tip Receipt). `ChargeRule` (kind, rate, order types, taxability, who it belongs to, default state, effective dates). `DayClose`. A bill is **derived**: the sum of live KOT lines minus voids, priced by channel list, taxed by the line's snapshot.
`Order` **is** the held bill: the audit's `PosDraft` is a cart, the restaurant's draft is an order with events, and it takes **no voucher number until tender** (an abandoned check burns nothing, which keeps Rule 46's series gap-free **[C audit section 4]**).
`PendingOrder` (aggregator or QR intake: channel, external order id as the idempotency key, raw body kept, state Placed, Acknowledged, Accepted, Rejected, Held, plus `liability_on` per tax entry).

**Why append-only KOT events [J].** Two captains adding to table 5 on two devices create two ticket documents and the check is their union; no field-level conflict, no lost edit. A voided line is another event.
This fits the app's rule that a pull must never overwrite an unsent local edit **[C CLAUDE.md Phase 24 gotchas]** and its outbox change log **[C]**.

**Posting rules [J].**
- *Settle (dine-in, takeaway, own delivery):* one Sales voucher per paid bill. **One tender:** the party leg is the tender ledger itself (Cash, UPI collections, Card) or the customer for a credit sale (the audit's S1). **Mixed tender:** the Sales voucher takes the larger tender as its party and
  same-transaction Receipt vouchers move the rest, each naming the final bill number, and both are cancelled together (S2) **[C audit 2.3]**. Sales and CGST and SGST per line slab. **Packaging, delivery and any service charge are service lines** (service items carrying a dated taxability snapshot), **not extra
  ledger legs**: the engine adds any non-tax, non-party leg to taxable value, so a leg cannot be "outside GST" **[C gst.ts:161-177, audit 2.1]**; a charge the CA rules outside GST is a Non-GST line (the engine leaves Non-GST lines out **[C CLAUDE.md Phase 7]**). **A tip is a Receipt** (Dr the tender ledger,
  Cr Tips payable), never a leg of the bill. One cash ledger per counter gives drawer-level books. The voucher has **one sales ledger per bill**, so food, beverage and packaged goods share a P&L line and the item and stock-group reports carry the split **[C audit 1.1, 1.3]**. An 80 mm bill prints the
  per-rate tax lines the voucher already carries.
- *Aggregator order:* Sales voucher to an aggregator receivable, class "tax paid by operator under 9(5)", **no output GST**; the payout bank line clears the receivable on bill-wise, with commission and its GST to expense. Reported in Table 14(b) and 3.1.1(ii) (4.3, to be confirmed). **Never a tax invoice from us.**
- *Void before the ticket:* nothing. *After the ticket:* a `KotVoid` and, if cooked, a wastage Stock Journal in v2. *After the bill printed:* cancel (shell, number kept; a mixed-tender bill cancels both documents). *After payment, or after the day is closed:* credit note (Petpooja's glossary draws the same line **[W]**).
  A reprint prints DUPLICATE and counts. **A comp** is a line at zero value with a reason and approver, reported, never deleted.
- *Day-end (v2):* a Stock Journal consuming ingredients by recipe times sold quantity.

### 7.4 The architecture risk to settle first
Everything above assumes one device is the source of truth for an outlet. The moment two tablets and a counter must agree **while the internet is down**, cloud sync is not enough: Petpooja and GoFrugal both advertise offline devices that stay in step **[W]**, and the Comsky client only talks to the cloud **[C]**. The v2 answer is an **outlet hub**: the counter's Electron app serves the outlet's LAN over WebSocket, devices write ticket events to it, and the hub syncs to the cloud. The append-only event design makes that hub simple (a log, not a merge engine). Decide this before promising a captain app.

**What round 2 adds [W][J].**
- *The evidence for the shape.* Petpooja's "Local" type is a Bridge Server on the LAN with the counter, captain devices and KDS attached, installed by its team, and it is the only type that carries several devices through an outage (2.6); its Electron and Android types are single-device offline. Toast's offline mode keeps one device
  working and tells staff not to switch, and its offline KDS needs a hardwired device acting as the local hub (3.13). The audit reaches the same conclusion from our own code: sync pulls reload the whole book per page and nudges arrive every few seconds, so live tables need "a LAN or low-latency channel of its own" **[C audit section 5 and ranked risk 17]**.
- *Three tiers.* **Devices** (counter, captain tablets, KDS) talk only to the **outlet hub**, a role of the counter PC's Electron process: the KOT-event log, the **print server for every ticket and bill**, the day-close, the offline queue, and the app shell served to tablets over the LAN so the outlet runs with no internet. The hub syncs to the **cloud**, which also receives aggregator and QR webhooks, publishes menus to channels, holds
  the pending-orders table and relays accepted orders to the hub when it is reachable (3.7). The hub is a **relay and a cache, not a fork** (2.6 lesson 2).
- *Why the tablet never prints.* Web Bluetooth and WebUSB need HTTPS, and the live site is HTTP-only **[C batch spec 13.6, path P4; HANDOFF.md]**: a captain tablet cannot reach a Bluetooth printer from a web page today. Printing from the hub, as Petpooja Local does, sidesteps that, and it also keeps one place where tickets are numbered, retried and logged.
- *What has to be spiked before anyone promises it* **[J, none of this was run]:** a tablet's browser loading the shell and talking WebSocket to a PC on the LAN without mixed-content blocks (a page served over HTTP from the hub is the simplest path); service-worker and install rules on plain HTTP; a hub whose PC sleeps, restarts or is replaced (the replacement pulls the log from the cloud before it
  serves, the audit's rule for a replaced terminal); two hubs started by mistake in one outlet (one lease, held in the cloud and cached); clock skew between devices (the audit's working-date rule needs a checked sync clock); and the Electron app's missing kiosk, single-instance and power-save settings **[C audit 7]**.
- *Volume.* A bill is one voucher, but it is several KOT events: at 800 bills a day and about four events a bill, the change-log outbox, which is never pruned, grows by roughly 1.2 MB a day from the vouchers alone **[C audit 3.1, measured on an M4]**. KOT events must not ride the voucher outbox at full payload for ever: keep them in their own append-only tables, ship them to the cloud compactly,
  and archive them after the day is closed and reconciled **[J]**.

### 7.5 Where restaurant mode sits in the existing plans (round 2)
**Lanes.** The audit's lanes A to H are the build frame **[C pos-code-audit section 14]**. Restaurant mode is lane H (vertical packs) on top of the others:

| Audit lane | It owns | Restaurant mode needs from it | Restaurant mode adds |
|---|---|---|---|
| A, hot path | Speed of save, of the number and of a pull | A bill save and a keystroke that do not scale with the book; a pull that does not reload the whole book per page | A published budget for the add-item path (below) |
| B, POS engine (pure) | Cart, pricing, tender, bill, shift, holds, receipt, numbering, the detective event log | Tender shapes S1 and S2, discounts allocated across lines before tax, the shift and Z logic, the event log | The order and KOT event engine, charge rules, the split engine, kitchen routing, menu variants and options |
| C, data and contracts | `Voucher.pos`, `PosShift`, `PosDraft`, `Customer`, one additive Dexie bump | That bump | Tables for floors, tables, menu, orders, KOT events, pending orders and day close: in the same bump or a second, not a third |
| D, counter UI | An own POS root for touch and keyboard | The root | The table map, the order screen with modifiers, the KOT and KDS screens |
| E, devices and print | The device bridge: silent print, printer list, drawer, kiosk flags | The bridge | An ESC/POS emitter and a receipt or KOT job kind |
| F, server and rights | A per-terminal credential, server-side rights | Void, comp, discount, reprint and no-sale rights | A server route for aggregator and QR intake; rights per station |
| G, reports | Z, shift, tender, cashier, hour, voids | The Phase 28 rule that every figure equals the report it drills to | Item, category, captain, channel, table-turn, KOT-versus-bill |

**Add-item budget [J].** The audit measured 66 ms for one `nextVchNo` call at 120,000 vouchers and a whole Accept at 223 ms p50 at 250,000 on an M4, and says an owner's PC is three to five times slower **[C audit finding 5 and ranked risk 1; PERF.md]**. A captain adding an item must wait on none of it: add-item and its
KOT must be O(1) in the size of the book, never touch the voucher array, and meet a published budget (a target to test, for example tap to ticket in the printer's buffer inside 300 ms on the owner's slowest PC). **I measured nothing; the number is a target, not a result.**

**Reuse from the batch, serial and label spec (committed; engines merged; nothing wired) [C].** (1) **The bridge discipline:** a main-process bridge where the renderer sends a structured job and never a command, a printer list built live in main, a network-printer allow-list that only main writes (private addresses, ports 9100 to 9109), a Windows
RAW helper checked by hash, one job at a time, and an answer that says "sent to the printer", never "printed" (spec 13.6; `electron/labelLang.cjs` is its pure TSPL emitter and the bridge itself is a later slice). Receipts and kitchen tickets need a **sibling ESC/POS emitter on the same bridge, not a second bridge**; ESC/POS jobs are text lines, rules and bitmaps, so the
ceilings differ. macOS raw printing is outside that spec's v1, consistent with the audit's Windows-first decision. (2) **The Devices tab and `TW_DEVICE`** for printer profiles; a station is a name that points at a printer profile. (3) **The barcode and QR encoders** (`src/core/barcode/*`, pure) for the UPI QR on a 58 mm bill, a token QR and a table QR. (4) **The scan layer** only for a counter that sells packaged
goods or weighs items; a dine-in restaurant needs none. (5) **MRP and batch** only matter for resale goods and v2 raw materials. (6) **A gap:** a grep of that spec finds no ESC/POS, KOT, restaurant or cash-drawer content (its "thermal" is label stock); receipts, tickets, drawer kick, cut and 58 and 80 mm paper are this document's hand-off to lane E.

**The custom-fields build (`.wt-cf`) [C].** Its tracking variant keeps stock by a line field with value pooled (AD.3, CF24): good for "stock by colour", not for a Half or Full portion, which is a *priced menu variant that points at its own sellable* and lives in the menu layer. The "carry funnel" for new voucher fields must be used by the POS writer or custom values vanish on alteration; the mouse Back control and dialog Close
buttons are shell work that a POS root outside the shell avoids **[C audit section 12]**.

**Conflicts and gaps between the documents [J].** (a) Two names for one bridge, `deviceBridge.cjs` (audit lane E) and `labelBridge.cjs` (batch spec): decide once (D10). (b) The audit's lane H says "aggregator intake over the REST API": I advise against it (3.7). (c) The audit's "live-table channel" is the hub of 7.4. (d) The audit and this document each cost the shared foundation as zero: someone must own it before any vertical's estimate
means anything. (e) The audit's Dexie bump and the CF build's choice to avoid one must be sequenced by one person.

### 7.6 The short answer (restated for round 2, **[J]**)
**v1, worth shipping (counter service and small dine-in, one device per outlet):** a menu with variants, option groups and simple combos on **stockless** item lines; four order types (dine-in, takeaway with a token, own delivery, manual aggregator order); floors and tables with derived status, move and merge; **immutable KOT events**
(ADD, EDIT, CANCEL, TRANSFER, DUPLICATE) printed per station from the counter PC; settlement into Sales vouchers with the audit's S1 and S2 tender shapes, **charges as service lines (service charge off by default)** and **a tip as a Receipt**, split by item or equally; an owner-configured reason list, a recorded approver, cancel-not-delete, no Undo on a printed bill, a day close
with a cash count, a voids-and-comps review and KOT-versus-bill reconciliation; the GST profile (5% with no credit, composition, premises class, the 9(5) class, FSSAI per outlet, GSTIN capture); keyboard-first and touch on the same state. **Not in v1:** KDS, courses, the hub and captain app, aggregator APIs, QR ordering, recipes, loyalty, liquor, reservations, happy-hour pricing.
**v2:** the hub, captain tablets, KDS and courses; the cloud webhook receiver with aggregator middleware (after TLS); recipes with day-end backflush and multi-level BOM; QR ordering; server-side rights; time-of-day pricing; regional-script tickets. **v3:** central kitchen and menu inheritance, loyalty and wallet (after counsel), direct partner APIs and ONDC, kiosk, dispatch, liquor.
**How the data differs from retail POS:** (1) **tables and an order that lives for hours** against a cart that lives for a minute: held as an event stream with no voucher number until tender; (2) **KOT**: immutable tickets per station with change and cancel kinds, where retail has no kitchen; (3) **modifiers, variants and combos**: a line's identity is item, variant, chosen options, course and note, and an option can carry a
price delta (later a recipe delta), where retail's identity is item, batch, serial or barcode and its price is an MRP; (4) **recipes**: ingredients depleted by recipe times quantity, at day-end first and with sub-recipes later, where retail depletes the sold item itself; (5) **charges and tax classes** (service charge, packaging, tip, 9(5), composition, FSSAI) where retail has MRP-inclusive pricing, batches, Legal Metrology and, for pharmacy,
the drug rules.

---

## 8. Risks

| Risk | Why it matters | Mitigation |
|---|---|---|
| No LAN hub | Multi-device restaurants lose the offline promise; even Toast does not share orders across devices offline, and Petpooja sells this only in its on-premises tier **[W][Ws]** | D2: single counter in v1; hub in v2; say the offline contract on screen; test with the network unplugged |
| Peak-hour performance | The leaders' top complaint; our own per-keystroke work scales with the book **[C audit]** | A published add-item budget (7.5), tested at festival volumes on a slow PC before launch |
| Service-charge and billing-default exposure | CCPA ordered a chain's billing software to change (Chaayos, Jul 2026) | Every charge off by default and an owner-dated setting; wording signed by a CA |
| Aggregator class wrong | Charging GST twice or filing in the wrong table | The 9(5) class driven by `liability_on`, tests against the circular's rules, CA review of Table 14(b) |
| Aggregator receiver is on a plain-HTTP single VM | Unsigned, time-limited webhooks need a public HTTPS endpoint that answers in seconds; a trip of the circuit breaker switches off all of a business's webhooks **[W]** | TLS and a hostname first; idempotent intake; a late-orders view; a shared secret in a header, never logged |
| Posting a restaurant bill the wrong way | A tip, a charge or a split-tender leg on the Sales voucher is added to taxable value **[C]** | The 7.3 shapes; a golden suite that equals GSTR-1 for the same bills entered through the Sales screen (the audit's lane B exit) |
| Dish as a plain stock item | Negative-stock prompt on every bill; stock figures meaningless **[C]** | The stockless rule (D9) before any menu import |
| The shared foundation is nobody's estimate | Restaurant, retail and pharmacy all wait on lanes A to C and E | Name an owner; sequence the single Dexie bump |
| Two bridge names for one discipline | Two bridges means two allow-lists and two ways to print | D10 |
| KOT events inflate the change log | An unpruned outbox already reaches 1.2 MB a day at 800 bills **[C audit]** | Own tables, compact shipping, archive after day close (7.4) |
| Partner calendars | Zomato, Swiggy and middleware timelines are outside our control; Zomato's and Swiggy's partner docs are not public **[Ws]** | Apply now; manual entry in v1 |
| Print hardware matrix | Thermal printers differ; the repo has no thermal layer | A short supported-printer list; a test rig |
| Client-only rights | A manager PIN is not a security boundary | Say so on screen; server-side rights in v2 |
| Menu rules per channel | A name or option that one platform rejects breaks a publish | Validate before publish |
| Statutory drift | Rates, premises rules and limits moved in 2022, 2025 and 2026 | Dated data, quarterly review |
| Scope creep into hotels, banquets, liquor | Different compliance and workflows | Out of v1 |

---

## 9. Open questions

1. D1 to D11 above.
2. **Torqus and inresto:** is inresto the new name of Torqus's POS or a separate Dineout product, and what do they do today? (Found in round 2 but not settled; no vendor page opened.)
3. UrbanPiper: price, whether the webhook URL must be HTTPS and public, and the exact retry timing (ask POS support). *(Round 2 answered the payload, statuses, idempotency and certification; see 3.7.)*
4. Zomato's and Swiggy's current POS partner terms, fees and timelines (partner-gated; no public Swiggy page found).
5. How the 9(5) rule applies on ONDC when buyer and seller apps are separate firms.
6. The division bench's decision on service charge, and the GST department's position.
7. Current GSTR-1 and GSTR-3B tables for 9(5) supplies: 14(b) and 3.1.1(ii) per search summaries; confirm on the portal and that restaurant service is in them.
8. ~~Does the stock engine skip `isService` items~~: **answered, it does not** (`types.ts:525`, grep of `inventory.ts`); D9 decides the fix.
9. Outlet model: company per outlet or godown and cost centre, and how group reports read across outlets.
10. Which thermal printers and which Bluetooth printers to support first; whether 58 mm is v1 or only 80 mm; whether a USB or LAN printer is the default; whether the owner's kitchens need Hindi or other-script tickets in v1.
11. Whether a manager PIN should be enforced server-side before any real customer relies on it.
12. What the 57th Council's recommendations become once notified (4.11), and whether the no-credit condition on the 5% entry moves.
13. Who owns the shared POS foundation (7.2 item 0) and who sequences the Dexie bump.
14. Does a tablet's browser reach a hub on the LAN without mixed-content or service-worker blocks (7.4 spike)?
15. Meta's own India rate card for WhatsApp utility messages (the blogs disagree).

---

## Appendix A. Sources

**Government and statute (read) [W]**
- Circular 167/23/2021-GST, 17 Dec 2021 (restaurant service through e-commerce operators): https://cbic-gst.gov.in/pdf/Circular-167-17-12-2021-GST.pdf (key `circ167`)
- Circular 164/20/2021-GST, 6 Oct 2021 (cloud and central kitchens, ice cream parlours): https://cbic-gst.gov.in/pdf/Circular-No-164-2021-GST.pdf (`circ164`)
- GST Council, decisions on services at the 23rd meeting (Nov 2017): https://www.gstcouncil.gov.in/sites/default/files/2024-02/gst-services-23rd_gst-council.pdf (`gstc23`)
- Notification 05/2025-Central Tax (Rate), 16 Jan 2025 (specified premises): https://gstcouncil.gov.in/sites/default/files/2025-01/ctr05-2025.pdf (`ctr05`)
- GST Council consolidated service rates (the notification-11/2017 table): https://www.gstcouncil.gov.in/sites/default/files/2024-02/gstservicerates.pdf (`rates`)
- 56th GST Council press release (Sep 2025): https://gstcouncil.gov.in/sites/default/files/2025-09/press_release_press_information_bureau_0.pdf (`gstc56`)
- CGST Act as consolidated on 1 Aug 2021 (sections 9(5), 10, 15, 31): https://cbic-gst.gov.in/pdf/CGST-Act-Updated-01082021.pdf (`act`)
- Composition levy FAQ, July 2017 (out of date on the limit): https://cbic-gst.gov.in/pdf/faq-manual/faq-composition-levy-revised.pdf (`compfaq`)
- State tax department training deck on hotel and restaurant services (one state's material, not law): https://gstcouncil.gov.in/sites/default/files/2024-04/restaurantservices_vns-i.pdf (`updeck`)

**Government and statute read in round 2 [W] (extracted text unless noted)**
- Notification 14/2019-Central Tax, 7 Mar 2019 (composition limit ₹1.5 crore; ₹75 lakh in eight states): https://gstcouncil.gov.in/sites/default/files/2024-05/notfctn-14-central-tax-english-2019.pdf (`ntf14`)
- Notification 14/2020-Central Tax, 21 Mar 2020 (dynamic QR code on B2C invoices above ₹500 crore turnover): https://gstcouncil.gov.in/sites/default/files/2024-05/notfctn-14-central-tax-english-2020.pdf (`ntf14-2020`)
- Annexure VII, opt-in declaration for a registered person supplying hotel accommodation: https://cbic-gst.gov.in/pdf/Annexure-VII.pdf (`annexure7`)
- CBIC FAQ on 'restaurant service' supplied at 'specified premises', Mar 2025, 28 questions, **read as a copy** (the CBIC host was not opened): https://cdn.taxo.online/wp-content/uploads/2025/03/8930681f-bb7c-4fab-b5c9-b59ccdd3c200.pdf (`faq-copy`); cross-checked against Mondaq https://www.mondaq.com/india/tax-authorities/1606898/frequently-asked-questions-on-restaurant-services-supplied-at-specified-premises
- CGST Rules, 2017, Part A as amended to 1 Jun 2021 (Rules 46 and 54): https://cbic-gst.gov.in/pdf/01062021-CGST-Rules-2017-Part-A-Rules.pdf (`cgstrules`)
- Circular 240/34/2024-GST, 31 Dec 2024 (credit of operators paying under section 9(5)): https://gstcouncil.gov.in/sites/default/files/2025-01/circular-no-240-2024.pdf (`circ240`)
- FSSAI orders of 8 Jun 2021 and 30 Sep 2021 (licence number on bills), one PDF: https://www.fssai.gov.in/upload/advisories/2021/10/6156936ac80bbOrder_License_Number_Print_01_10_2021.pdf (`fssai-order`)
- CCPA, Chaayos order of 14 Jul 2026 (SCC Online): https://www.scconline.com/blog/?p=390990 (`scconline`); CCPA 2022 guidelines as summarised: https://www.scconline.com/blog/post/2022/07/05/ccpa-issues-guidelines-to-prohibit-automatic-levy-of-service-charge-in-hotels-and-restaurants/
- Delhi High Court, single judge, Mar 2025 and the appeal listing (LiveLaw, 9 May 2025): https://www.livelaw.in/amp/high-court/delhi-high-court/delhi-high-court-restaurant-mandatory-service-charge-291763 (`livelaw`)
- 57th GST Council, 8 Oct 2026, commentary: Trilegal https://trilegal.com/knowledge-repository/analysis/key-recommendations-of-the-57th-gst-council-meeting/ (`trilegal`); CAclubindia https://www.caclubindia.com/news/57th-gst-council-recommendations-relating-to-services-26850.asp (`caclubindia`); Taxscan https://www.taxscan.in/gst/gst-council-proposes-amending-s-175-to-give-itc-relief-to-certain-sectors. The PIB release (PRID 2320934) returned 403.

**Government and law, summaries only [Ws] or commentary [W]**
- CBIC FAQ of 28 Mar 2025 on specified premises, via Taxmann: https://www.taxmann.com/post/blog/cbic-issued-faqs-to-explain-criteria-procedures-and-gst-implications-for-restaurant-services-supplied-at-specified-premises/
- Pre-Nov 2017 restaurant rates, two contemporaneous reports that disagree: TaxGuru, 19 Nov 2017: https://taxguru.in/goods-and-service-tax/gst-on-restaurants-all-you-needs-to-know.html (`taxguru`) **[W]**; Business Today, 15 Nov 2017: https://businesstoday.in/current/economy-politics/restaurants-gst-5-percent-ac-non-hotels-tax-restaurant-bill-cheaper/story/264054.html (`businesstoday`) **[W]**
- CCPA 2022 service-charge guidelines (PIB 403; consumeraffairs.nic.in refused the connection in round 2): https://www.pib.gov.in/PressReleasePage.aspx?PRID=1839133 **[Ws]**
- Notification 78/2020-Central Tax (HSN digits), via the ICAI copy and GST Council PDF search results; SAC 996331 and 996332 via compliance sites (busy.in, incorpx.io, blog.petpooja.com/free-tools/sac-code-finder-free/); GSTR-1 Tables 14 and 15 and GSTR-3B 3.1.1 via ginesys.in, caclub.in, taxscan.in, cleartax.in **[Ws]**
- RBI card-on-file circulars and the 1 Oct 2022 date via law.asia, amsshardul.com and rbi.org.in index pages (search results); DPDP Rules 2025 phasing via scconline.com, azbpartners.com, mondaq.com **[Ws]**
- Meta WhatsApp pricing (developers.facebook.com page not read in full) and India rates via aisensy.com and robylon.ai blogs **[Ws]**
- CCPA action, 10 Jan 2026 (27 restaurants): https://www.etvbharat.com/en/bharat/mandatory-levy-of-service-charge-by-restaurants-violates-consumer-law-ccpa-enn26011003854 **[W]**
- CCPA action, 20 Jul 2026 (41 restaurants, Chaayos): https://moneylife.in/article/service-charges-chaayos-fined-50000-as-ccpa-launches-action-against-41-restaurants/81100.html **[W]**
- CCPA advisory on LPG and similar charges, 25 Mar 2026: https://www.scconline.com/blog/post/2026/04/08/ccpa-advisory-on-lpg-charges-explained/ **[Ws]**
- Delhi High Court division-bench hearing of Aug 2025 (Storyboard18) and the June 2026 Moneylife report that the appeal is pending: https://www.storyboard18.com/how-it-works/delhi-hc-pulls-up-restaurants-for-levying-service-charge-on-inflated-mrp-rates-79541.htm **[Ws]**
- FSSAI licence number on bills (commentary): https://foodsafetyworks.com/news/fssai-order-mandatory-declaration-of-fssai-license-registration-number-on-cash-receipts-invoices-cash-memo-bills-etc-by-all-food-businesses/ **[W]**; https://ssrana.in/articles/fssai-license-no-mandatory-receipts-restaurants/ **[Ws]**; the earlier mail.fssai.gov.in link returned 404 and the fssai.gov.in link above worked.

**Vendors [W]**
- Petpooja pricing: https://petpooja.com/pricing (`petpooja-pricing`); home: https://petpooja.com/; fine dine: https://www.petpooja.com/fine-dine-restaurant-software; online ordering: https://petpooja.com/online-order-management-software;
  ONDC: https://www.petpooja.com/ondc; GST blog (18 Jul 2026): https://blog.petpooja.com/finance-compliance/gst-billing-for-restaurants/ (`petpooja-gst-blog`);
  captain-app blog: https://blog.petpooja.com/operations-workflows/use-petpooja-captain-app-to-enhance-customer-experience-restaurant-business/; captain app (App Store): https://apps.apple.com/us/app/-/id6470435905
- Restroworks: https://www.restroworks.com/ ; https://www.restroworks.com/restaurant-pos ; https://www.restroworks.com/pricing
- UrbanPiper: https://www.urbanpiper.com/ ; API overview https://api-docs.urbanpiper.com/downstream/getting-started/overview ; Zomato constraints https://api-docs.urbanpiper.com/downstream/aggregator-constraints/zomato (`urbanpiper-zomato`);
  Swiggy constraints https://api-docs.urbanpiper.com/downstream/aggregator-constraints/swiggy ; help: https://help.urbanpiper.com/ordering-channels/swiggy/integration-overview and https://help.urbanpiper.com/zomato/getting-started
- GoFrugal: https://www.gofrugal.com/restaurant/restaurant-pos-software ; https://cdn.gofrugal.com/restaurant/kitchen-order-ticket-kot-software.html
- Limetray: https://www.limetray.com/ ; DotPe: https://www.dotpe.in/
- Odoo: https://www.odoo.com/documentation/20.0/applications/sales/point_of_sale/restaurant.html ; https://odoo.com/documentation/18.0/applications/sales/point_of_sale/online_food_delivery.html
- Square Orders API: https://developer.squareup.com/docs/orders-api/what-it-does ; Lightspeed K-Series API index: https://api-docs.lsk.lightspeed.app/ ; help centre: https://k-series-support.lightspeedhq.com/hc/en-us
- Reviews and listings: SoftwareSuggest https://www.softwaresuggest.com/petpooja , /posist , /gofrugal-restaurant-pos ; Techjockey https://www.techjockey.com/reviews/petpooja-pos , https://www.techjockey.com/detail/limetray ; Trustpilot https://www.trustpilot.com/review/petpooja.com

**Vendors read in round 2 [W]**
- Petpooja POS types and architecture: https://blog.petpooja.com/industry-business-guides/petpooja-pos-types-lite-android-local-electron-comparison/ (`petpooja-pos-types`); on-premises POS and Bridge Server https://www.petpooja.com/offline-pos (`offline-pos`); https://blog.petpooja.com/industry-business-guides/restaurant-pos-without-internet/ (`without-internet`);
  glossary on voids https://blog.petpooja.com/glossary/void-bill/ ; billing page https://www.petpooja.com/billing (the captain-app page amd.petpooja.com/captain-ordering-app returned 404)
- Torqus: SoftwareSuggest https://www.softwaresuggest.com/torqus-pos (`softwaresuggest`); the Dineout acquisition, 27 Nov 2018: https://www.thenewsminute.com/atom/dineout-acquires-cloud-based-point-sale-system-restaurants-torqus-92306 (`thenewsminute`)
- UrbanPiper order contract: https://api-docs.urbanpiper.com/downstream/llms.txt (index); https://api-docs.urbanpiper.com/downstream/order/order-components.md ; https://api-docs.urbanpiper.com/downstream/api/endpoints/order-management/order-relay.md ; https://api-docs.urbanpiper.com/downstream/api/endpoints/order-management/order-status-update.md ;
  https://api-docs.urbanpiper.com/downstream/api/endpoints/setting-up-webhooks.md ; https://api-docs.urbanpiper.com/downstream/resources/webhook-circuit-breaker.md ; https://api-docs.urbanpiper.com/downstream/integration-certification/certification.md
- Toast: https://doc.toasttab.com/doc/platformguide/adminVoidingOrders.html ; https://support.toasttab.com/en/article/Course-Firing-Options ; https://support.toasttab.com/article/Offline-Mode-Using-Toast-During-a-ISP-Outage
- Lightspeed K-Series: https://k-series-support.lightspeedhq.com/hc/en-us/articles/360051089273-Adding-orders-in-Table-Service-mode ; .../1260804658689-Managing-production-centers ; .../1260804656709-Creating-and-managing-floor-plans-and-tables
- Square: https://squareup.com/help/us/en/article/6428-coursing-with-square-for-restaurants

**Search summaries only [Ws]**
- Round 2: G2, Capterra, Software Advice and GetApp pages for Petpooja (all 403 on fetch); Toast happy-hour pricing, service charges and offline help titles; Lightspeed void reasons; Square offline payments and restaurant features; Odoo happy hour (official 16.0 pricelist page and third-party modules); Zomato's POS developer platform (YourStory, Sep 2024, and an unofficial PHP SDK readme); Swiggy POS onboarding (a Petpooja how-to and a former Swiggy product manager's Substack);
  Limetray and DotPe 2026 status and pricing (Capterra, FitGap, TrustRadius listings); ONDC food share (outlookbusiness.com, inc42.com, dineopen.com); KOT in regional languages (a BillByteKOT listing); ESC/POS (Epson's reference, 403) and 58 mm printer specifications (datasheets); Torqus and inresto (techimply.com, trustradius.com, itvoice.in).
- Round 1: Petpooja features and Techjockey pricing; Toast pricing, contracts and complaints (chowly.com, upmenu.com, owner.com, nerdwallet.com summaries); GoFrugal features and Capterra rating; Zomato POS developer platform (YourStory and BW Disrupt, September 2024; an unofficial Zomato PHP SDK readme);
  Swiggy integration history (a former Swiggy product manager's Substack); ONDC food share and commissions (outlookbusiness.com, dineopen.com, uengage.in, indbiz.gov.in); GST on notified services for operators (cleartax.in, taxguru.in, livelaw.in).

**Repo [C]**
- `src/types.ts` (31-37, 91-105, 425-435, 454-460, 462, 522-526, 640-654, 880-902, 944-1074); `src/core/voucherCancel.ts:1-12`; `src/core/access.ts:12-14`; `src/core/creditLimit.ts:1-16`; `src/core/upi.ts:1-25`; `src/core/qr.ts:1-4`; `src/core/gstRates.ts:1-20`;
  `src/core/gst.ts:152, 161-177`; `src/core/gstNil.ts:1-5`; `src/core/gstr1.ts:151, 670-711`; `src/core/itc.ts:8-30`; `src/core/vchNumbering.ts:1-17, 18-22, 77-83`; `backend/src/numbering.js:1-30` and `backend/sql/008_voucher_series.sql` (committed in `2cee555`); `src/ui/inventory/manufacturing.tsx:105,124`; `src/styles/print.css:297`;
  `docs/research/einvoice-gsp.md:71-73`; `docs/research/batch-serial-code-audit.md:109`; `docs/research/mobile-market.md` (findings 6 and 7); `.wt-cf/docs/design/custom-fields-spec.md` (AD.3, CF24 and CF26); `CLAUDE.md` (Phase 16, 20-21, 24 and 26 notes).
  **Round 2 repo reads:** `docs/research/pos-code-audit.md` (findings 1 to 14, sections 2, 4, 5, 7, 8, 9, 11 to 14); `docs/design/batch-serial-labels-spec.md` (13.6, 14, 18; headings and greps); `electron/labelLang.cjs` (header); `src/types.ts:525` (`isService`), `:681` (`VoucherAction`), `:1005`, `:1019`, `:1065-1068`; `src/state/store.ts:357`; `backend/src/restPolicy.js:16-26`; `src/ui/VoucherScreen.tsx:2490-2503`; a `git log` and `git status` (read only).
