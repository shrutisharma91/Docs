# Mobile personas and jobs-to-be-done: owners, salespeople, accountants, CAs and parties

> **Status:** research and design input only. Nothing here is built and this document changes no code. Written 2026-10-22
> against `main` at `83d9876` (the working tree also carries about 160 uncommitted paths from the parity wave 3 gate and the MCP
> server build; none of them is read as a fact here unless a line says so).
> **The ask (owner, verbatim intent):** "research well, and develop great mobile apps, considering also business owners and sales
> people using the app, in mind."
> **What this is:** the persona and job layer that the mobile design, the platform work and the feature roadmap should hang from.
> It is not an architecture decision. Where a platform fact forces a product choice it says so and stops there.
> **Classification:** mobile is a *Beyond Tally* capability. Track it under `beyond` in `docs/parity-data.json`, never in the parity
> percentage (CLAUDE.md, "Phases 18-21").
> **Evidence tags:** **[W]** a page I fetched and read (the fetch tool summarised it; I paraphrase, I do not copy) · **[Ws]** a
> search summary only, the page itself could not be read · **[C]** read in this repo (`file:line`) · **[J]** my judgement, not
> evidence · **[unverified]** I could not confirm it. `S<n>` points to the source list at the end.
> **No user was observed and no competitor account was used.** Every persona below is a hypothesis built from what vendors ship
> for a role, public role descriptions, incident reports, India market data and the code in this repo. Section 15 says how to test
> them in twelve sessions. Most "field sales" numbers come from vendors that sell field-sales software and are labelled as such.

---

## 0. TL;DR

1. **Two primary users with opposite needs.** The **business owner** wants answers, exceptions and nudges (read-first, approve in two
   taps, never lose track of cash and who owes what). The **salesperson** wants speed and survival without a signal (an order in
   under a minute, the party's dues and credit in front of them, a payment collected and a WhatsApp sent, all offline). A
   good mobile product serves both from one set of books, with different defaults by role. The accountant, the CA and the party are
   real but secondary: the accountant needs an inbox, the CA a read-only multi-client view, the party a later portal.
2. **For the owner, mobile is mostly a delivery problem, not a new-engine problem.** The questions an owner asks first (cash and
   bank, who owes me, what sold today) are already answered by the pure engines behind Dashboard v2 [C `src/ui/dash/widgets/registry.ts`].
   The phone needs a layout, a freshness stamp, a cache and a way to reach the data.
3. **For the salesperson, five objects do not exist yet:** a salesperson on a voucher (only `Lead.owner`, free text [C
   `src/types.ts:581`]), a beat / stop / visit, targets and incentives, photo and location proof, and per-rep cash accountability.
   Several engines the rep needs do exist: credit-limit check with a reasoned override, ghost recall, price levels, WhatsApp
   click-to-chat share, UPI link and QR, server-assigned voucher numbers with provisional offline numbers [C, section 2].
4. **Four blockers are verified in code and decide what can honestly ship** (section 2): HTTP only (no service worker, push, camera,
   GPS or install); one login per tenant (no way to give a rep an identity or take it back); a whole-tenant `/sync/pull` that returns
   every row as stored, including operator PIN hashes; and a shell with a 938 px floor. The third one matters most: **today, any
   phone that syncs holds the entire ledger, purchase costs, every rep's customers and the PIN hashes.** A salesperson app cannot
   promise "reps do not see cost or other reps' customers" on top of that. It needs a server-built projection, not the books.
5. **Recommendation on data:** the owner and the trusted accountant read the books (cached). A salesperson reads a **projection**
   built for that rep: assigned parties with open bills and a credit headroom, items with that rep's price levels and a stock
   status, schemes, a block of voucher numbers. Section 8 sets out the options and the cost of each.
6. **Offline for a rep means a queue with honest states.** Orders, receipts and visits are saved on the phone with provisional
   numbers, checked against cached exposure plus the rep's own unsent orders, re-checked at sync and **held for the owner, never
   silently dropped or silently allowed.** A competitor's own help page says its offline orders skip the credit check [W S10]; another
   tells users to turn sync off before billing offline [W S4]. Those are the failure modes to design against.
7. **Collections are the money, and the leak.** A 2026 vendor report of about 1.1 lakh MSMEs puts the average invoice at 73 days to
   clear against a 45-day legal expectation [W S45]. A Pune drug distributor's employee is alleged to have kept about ₹29 lakh of
   collections over three months [W S51]. So per-rep cash accountability, payment to the company's own UPI QR and a same-day
   handover come **before** targets and incentives.
8. **India's rules shape the flows.** Cash of ₹2 lakh or more from one person is barred for the receiver (old s.269ST, now s.186 of
   the Income-tax Act 2025) [Ws S47]; late payment to micro and small suppliers costs the buyer its deduction (s.43B(h), now
   s.37(2)(g)) [W S46]; GST dates are the 11th and 20th (monthly) or the 13th and 22nd or 24th (QRMP) [W S48]; an e-invoicing
   company above ₹5 Cr cannot issue a valid invoice offline (an invoice that should carry an IRN and does not is not valid), so its reps take an
   *order* offline and the invoice follows [C `docs/research/einvoice-gsp.md:71`, `:75`; W S4; J].
9. **WhatsApp and UPI are the loop-closers.** WhatsApp or WhatsApp Business is on 97% of the phones in one MSME-owner survey [W S40];
   UPI ran 24.07 billion payments in September 2026 [W S41]. v1 can close every loop with a prefilled WhatsApp the person sends
   themselves (already built for vouchers and outstanding bills [C `src/core/share.ts:1-4`]), no messaging contract needed.
10. **Headline success criteria** (section 14, all targets to be proven in a spike): a known party's order of five lines saved **in
    60 seconds, offline**, median; the owner sees **today's cash and bank in 5 seconds from a cold start** (2 seconds from cache with a
    stale label); a party's outstanding in 2 seconds offline; a receipt against one bill in 30 seconds; **zero** lost or duplicated
    orders across a day of offline work.

---

## 1. Method, and what I could not verify

### 1.1 How the personas were built

| Input | What it gives | Weakness |
|---|---|---|
| What vendors ship for a role (BUSY, Marg, Vyapar, Tally, SalesOn, Bizom, BeatRoute, Zoho) | Revealed preference: a vendor builds what buyers asked for | Marketing copy; says nothing about quality or use |
| Public role descriptions (NSDC distributor salesman, Tally owner guides) | The tasks of the role | Old (the NSDC pack is 2016) or written by a vendor |
| Incident reports (salesman cash fraud, delayed payments) | Where the money leaks | Allegations, a handful of cases |
| India market and device data | Constraints: phone, network, language, channel | Survey samples are retail-heavy, not distributors |
| This repo | What exists, what is missing, what is blocked | A snapshot of a moving tree |
| Judgement | Gap filling | Marked **[J]** and to be tested |

### 1.2 What I tried and could not read or verify

| Item | What happened | Effect |
|---|---|---|
| Play Store pages (Vyapar and others) | The fetch returned truncated text | No star ratings or review themes from the Play Store |
| Capterra, G2, Trustpilot, MouthShut | Capterra returned 403; the rest only as search snippets | Competitor "review" evidence is a few excerpts [Ws S5], not an analysis |
| Marg eOrder, Marg eOwner, Bizom Next iOS listings | Readable, but each has 0 to 3 ratings | No usable review signal for them [W S7, S8, S16] |
| Opensignal India reports | 403 on the report pages | Network quality is a search summary only [Ws S44] |
| NPCI statistics page, incometaxindia.gov.in | 403 | UPI figures come from a press outlet citing NPCI [W S41]; tax rules from commentary [Ws S47] |
| NSDC distributor salesman handbook | The preview is image-only | Role description from a search summary of its stated tasks [Ws S59] |
| Kantar-IAMAI 2024 PDF | Unreadable as text | Figures via a news summary [W S39] |
| Two founder interviews (Tally 2026 fireside, Vyapar SMBStory) | 403 | Their content is search-summary only [Ws]; the one interview I could read is OkCredit's CEO (2021) [W S19] |
| Any independent study of Indian SMEs using mobile accounting | None found | Adoption claims are the vendors' own |
| Reddit and forum threads on salesmen and discounts | None found by search | The "voice of the owner" is thin; section 15 plans for it |
| Tally reseller and partner commentary on mobile | Searches returned Tally's own pages and third-party companion vendors, no reseller commentary | The partner voice is absent here; the owner's own reseller customers are the first people to ask (section 15) |
| Case studies by trade (pharma, garment, hardware) | Only vendor pages and one stockist's notes; the one named case with a customer quote, BeatRoute and Utkarsh, has no numbers [W S15] | Trade differences in section 7.9 are directional |

### 1.3 Scoring used in the job tables

Frequency **F** (1 = a few times a year, 3 = weekly, 5 = many times a day) times Pain **P** (1 = a mild nuisance, 5 = costs money or
a sale today) gives a score out of 25. Both numbers are **[J]**, calibrated on the evidence cited in the row. They rank jobs against
each other and nothing more. Re-score them after the sessions in section 15.

---

## 2. Ground truth: what this repo has, and what blocks mobile

Verified for this document. File and line references are to the working tree of 2026-10-22. `backend/src/conflict.js` and
`backend/src/sync.js` carry uncommitted edits from another session (the line of `isAdminRole` moved from 115 to 129 while this was
being written), so re-check any backend line number at build time.

| # | Fact | Evidence | Why it matters for mobile |
|---|---|---|---|
| 1 | **The live deployment is HTTP only.** | `docs/HANDOFF.md:62`, `:89` | Service workers, Push, Geolocation, camera (`getUserMedia`), Web Share, Background Sync, Barcode Detection, Clipboard and WebAuthn all need a secure context [W S24]. A plain-HTTP host with a port is not one; only `localhost` is [W S23]. So no installed app, no offline shell, no push, no scanner, no GPS proof, until HTTPS (README piece S1). |
| 2 | **No manifest, no service worker.** `index.html` carries only a viewport tag. | `index.html:5`; a `find` for manifests and service workers returned nothing | Installability needs HTTPS and a manifest with 192 and 512 px icons; a service worker is not on Chrome's list [W S31]. |
| 3 | **The shell cannot fit a phone.** Fixed `212px 1fr 152px`; scroll width **938 px** at both 768 and 390; the only responsive rules are the Go To pill and the Dashboard row. | `docs/design/ux-analysis-2026-10.md:274-279` | A phone view is a new layout, not a media-query patch. The same document's N11 proposes a CSS-only first slice. |
| 4 | **One login per tenant.** Only an `owner` membership is ever inserted; `isAdminRole` exists for the sync mismatch report only; no invite route. | `backend/src/tenancy.js:10`; `backend/src/conflict.js:129`; `backend/src/sync.js:59-61`; README S2 | A rep cannot have an identity, be attributed in the change log by name, or be revoked alone. The only per-person control is the client-side operator PIN. |
| 5 | **Operator rights are a client-side control.** The file's own header says it is for people sharing a screen, not a data boundary. Rights are areas, voucher bases and date windows; there is no field rule (hide cost) and no row rule (own customers only). | `src/core/access.ts:12-14`, `:108-178`, `:323-335`; `src/types.ts:683-697` | "A salesperson must not see cost, margin or other reps' customers" cannot be enforced by today's model at all. |
| 6 | **`/sync/pull` is whole-tenant and unredacted.** It takes `since` and `device`, returns every change row of the tenant (all companies, all tables) 500 at a time, with each payload as stored. PIN-hash stripping exists only on the REST read path. | `backend/src/sync.js:343-372`; `backend/src/restPolicy.js:29-38` | A synced phone holds the whole book and the operator PIN hashes, which the REST policy itself calls brute-forceable offline in seconds. Visit logs and GPS pings, if they rode the books' change log, would also land on every other device. |
| 7 | **The client loads a whole company into memory.** The perf harness times pure engines on a desktop, "not time-to-paint". Dashboard v2's default board costs about 718 ms of engine time at 250,000 vouchers on the dev machine. | `src/state/store.ts:306`; `docs/PERF.md:1-9`; CLAUDE.md Phase 28 "Cost, plainly" | On a phone with 3 to 4 GB of RAM the same load is slower by a factor nobody has measured. `npm run perf -- --cpu-factor=4` is only a projection [C `docs/PERF.md`]. |
| 8 | **No salesperson, beat, visit, target, location, photo or barcode objects.** The only owner-of-a-deal field is free text; a search of `src/` finds no `BarcodeDetector`, `getUserMedia` or `navigator.geolocation`. | `src/types.ts:571-600`; search of `src/` | Section 6's field jobs are mostly new data. `Voucher` would need a first-class salesperson field (and the master-resave gotcha applies: a form that rebuilds a document drops fields it does not own [C CLAUDE.md, Phase 24 GOTCHAs]). |
| 9 | **Purchase cost is on screen for anyone with Inventory rights.** | `src/ui/VoucherScreen.tsx:2067`; `src/ui/inventory/stockquery.tsx:153` | A rep screen must be built without it, and the server must hold the below-cost rule, because the rep device must not hold cost at all. |
| 10 | **Assets that do exist and matter.** Credit-limit check whose override needs a reason that stays on the voucher; ghost recall of a party's last voucher re-derived against today's masters; price levels per party and per item; WhatsApp click-to-chat, mailto and Web Share builder (sends nothing by itself); UPI link and QR (off unless the owner turns it on, owner's own VPA); server-assigned voucher numbers with a provisional local number offline and a claim on return; Sales Order base with pending orders; CRM leads, pipeline and follow-ups; voucher cancel with reason; dispatch panel and e-way bill capture (landing in the same tree); Dashboard v2 widgets Today, Receivables, Payables, Cash and Bank, Overdue invoices, GST payable; one quick-create registry; saved report views; Bulk Entry. | `src/core/creditLimit.ts:1-15`, `src/types.ts:1021`; `src/core/recall.ts:7-20`; `src/types.ts:362`, `:522`; `src/core/share.ts:1-4`; `src/core/upi.ts:1-30`; `backend/src/numbering.js:1-30`, `src/core/vchNumbering.ts:115-123`; `src/types.ts:13`, `src/core/inventory.ts:315`; `src/ui/crm/`; `src/ui/cancel/`; `src/ui/dispatch/`; `src/ui/dash/widgets/registry.ts`; `src/ui/quickCreate.ts` | The rep and owner jobs below reuse these engines. The risk is not missing logic, it is carrying the engines onto a device without carrying the data the rep must not hold. |
| 11 | **English-only UI.** No i18n library, no Devanagari string in `src/`; Hindi appears only as search aliases (udhaar, kharcha) on dashboard widgets. Lakh and crore formatting exists. | search of `src/`; `src/ui/dash/widgets/registry.ts`; CLAUDE.md `money.ts` | A rep or counter clerk who does not read English cannot use today's strings. Language is a v1 requirement for the rep, not polish. |
| 12 | **No notification path to a device.** The server has an SSE hub; the client never subscribes; there is no push or email sender. | README S7 | Owner approvals and rep alerts have no channel today. |
| 13 | **No partial sync.** A device pulls a whole company; delta `op:'patch'` is designed but not built. | `docs/HANDOFF.md` trust map; `backend/src/sync.js:343-372` | First sync on a mobile data plan is the whole book. More than half of the surveyed MSME owners spend under ₹500 a month on internet [W S40]. |

---

## 3. India context that constrains the design

| Dimension | Evidence | Reading for this product |
|---|---|---|
| **Platform** | Android held **93.89%** of Indian mobile OS web traffic in September 2026, iOS 6.05% (a share of browsing, not of devices) [W S36]. | Android first. iOS still matters for owners and managers: one Bizom reviewer said managers and above mostly use iPhones [Ws S16]. [J] |
| **The phone in the rep's hand** | IDC, Q2 2026: 33.2 M units sold (down 11.1%); phones under US$100 fell to a 4.5% share from 15.6%; the **US$100-200 band is the largest at 46.8%**; average price a record US$315, because memory costs rose [W S37]. | New phones in the field are mostly in the roughly ₹10,000 to ₹19,000 band; the installed base is older and cheaper than that. Design for 3 to 4 GB RAM and old Chrome. [J] A memory squeeze also means cheap phones may get *less* RAM [Ws]. |
| **Brands and background limits** | Counterpoint Q2 2026 share: vivo 21%, OPPO 16%, Samsung 16%, Xiaomi 13%, realme 9% [W S38]. Xiaomi's skin restricts background work so hard that, with defaults, background processing does not work right; the fix is user-side Autostart and battery settings [W S34]. Vivo, Oppo and Realme need similar settings [Ws S34]. | Six in ten new phones (vivo, OPPO, Xiaomi and realme: 21 + 16 + 13 + 9 = 59%) come from skins that need user-side Autostart or battery settings; I did not check Samsung's. Push and background sync on these phones are best effort. **A salesperson's "sync when signal returns" cannot rely on a background task alone**; sync on app open and a visible "N unsent" badge are the floor. [J] |
| **Network** | Opensignal (Feb 2026, data from Oct to Dec 2025) ranks India in the top ten large countries for download speed, with Jio far ahead; 5G availability above 70% for Jio in the June 2025 edition [Ws S44]. Rural users are 55% of India's 885 M internet users; 41% of people are offline [W S39]. 55% of MSMEs in one PayNearby study tether to a phone hotspot [Ws]. | The averages are good; the **beat** is where the signal drops (basements, godowns, rural stops). Design for "no signal at the outlet" as the normal case for a rep, "weak signal" for an owner on the road. [J] |
| **Data cost** | Over half of surveyed MSME owners spend less than ₹500 a month on internet access [W S40]. An older Flipkart PWA case study reports a third of the native app's data use, 63% of its users on 2G and 60% of visits from the home-screen icon [W S33] (the page is undated and the 2G share will have changed; the lesson stands). | A 100+ MB first sync is a real cost. Compress photos, never auto-download the whole book to a rep, cache on demand. |
| **Language** | 57% of users prefer content in regional languages; of the 41% of Indians still offline, 25% cite lack of awareness and 13% the lack of local-language content [W S39]. OkCredit's CEO says the app runs in "more than 9" languages, works offline and takes voice entry [W S19]. SalesOn calls its interface Hindi-ready [W S6]. | Hindi plus one regional language for the rep and counter screens at v1; voice entry is a v3 bet. Owners read English more often than reps do. [J] |
| **WhatsApp** | WhatsApp or WhatsApp Business is on 97% of the owner phones in one survey [W S40]. Meta and Bain count more than 65 million Indian SMBs, about 5 million of which sell online [W S42]. | Every persona's loop closes on WhatsApp. v1 uses click-to-chat the person presses Send on, which needs no Meta approval [C `src/core/share.ts:1-4`]. The Business Platform splits messages into marketing, utility, authentication and service categories, with replies inside 24 hours free; INR rates are on a rate card I could not read [W S43]. |
| **UPI** | **24.07 billion** payments worth ₹29.37 lakh crore in September 2026, about 802 million a day. From 15 Oct 2026 a 0.4% merchant discount rate applies to *specified* person-to-merchant payments above ₹2,000 (capped at ₹300 for ₹75,000 or more), with exemptions for small merchants [W S41]. | A rep collecting by UPI is normal. Which receipts the new rate touches (B2B? invoice payments?) is [unverified]; the owner should not assume UPI is free at ₹50,000. |
| **Collections** | Recordent's 2026 report (about 1.1 lakh MSMEs, vendor-published): 73 days on average to clear an invoice, 82.6% of invoices carry terms of 0 to 30 days, and the average SME has ₹3.83 crore unpaid beyond 360 days [W S45]. | Late payment is a discipline and follow-up problem more than a terms problem, which is exactly what a phone nudge workflow can move. |
| **Smartphone use by owners** | In the same PayNearby index, 36% spend 4 to 6 hours a day on the phone, 29% of tech-savvy MSMEs use accounting software, 17% POS, 14% CRM; 6% use no software [W S40]. | The owner is already on the phone for business; the question is whether our app is the first thing they open. |
| **Tally owners' phone today** | TallyPrime's own phone route is *Reports in Browser*: view-only, needs a current Tally Software Services subscription, a Tally.NET ID per viewer and the office computer online; recording entries remotely is a separate paid access [W S11]. | The incumbent leaves a clear gap: an owner whose office PC is off sees nothing. A cloud copy with a cached last-known view is a real advantage, if it is trustworthy. |
| **Statutory dates and limits** | GSTR-1 on the 11th and GSTR-3B on the 20th (monthly); quarterly filers 13th, and the 22nd or 24th by state; PMT-06 on the 25th; third-party calendars, the portal is final [W S48]. Cash receipts of ₹2 lakh or more from one person (day, transaction or event) are barred for the receiver [Ws S47]; cash payments over ₹10,000 a day lose the deduction [Ws S47]; amounts unpaid to micro and small suppliers past the MSMED Act limit are deductible only when paid [W S46]. | Each one is a job or an alert for the owner, and a guardrail at the rep's collection screen. A CA must confirm every threshold before it is coded [J]. |

---

## 4. What others ship on a phone, and where the gaps are

### 4.1 Landscape

| Product | Owner features on phone | Salesperson features on phone | Identity and control | Offline | Evidence |
|---|---|---|---|---|---|
| **TallyPrime (Reports in Browser)** | View P&L, balance sheet, trial balance, day book, ledgers, outstandings, stock; drill to vouchers; download invoices | None. Docs by Ira (separate) uploads bills into drafts for the accounts team | Tally.NET ID per person; roles Owner, User, Auditor; access per report (display, print, full) | None: the office PC must be online | [W S11, S12, S13]; Docs by Ira [Ws S14] |
| **BUSY mobile** | Sales, receivables and stock figures; overdue reminders; low-stock alarms; 100+ reports; approve vouchers and masters | Orders, quotations, invoices on the go; new party and item; share on WhatsApp, email, SMS; capture payments with instrument details; salesman tagging; e-way bill and e-invoice from the phone | The desktop administrator creates mobile users and permissions; OTP to change a device; needs the desktop product and a BDEP ID | Not stated | [W S1]; approvals are set up on desktop with conditions by voucher type and series [W S2]; whether approval acts on mobile is not stated |
| **Marg (eOrder, eOwner, eRetail)** | eOwner: collections, outstanding and PDC, multi-company view, GPS location of salesmen, alerts to them, temporary credit limits, debtor and creditor lists, P&L | eOrder: live orders, route tracking, save carts offline or online, bill-by-bill collection, outstanding and PDC, live stock, a credit-limit option, **Mark Visit**, target vs achievement, receipts on WhatsApp | Not described on the pages | "Save carts, modify offline or online"; nothing on the credit check offline | [W S7, S8, S9]; ratings absent on both iOS pages |
| **Vyapar** | Reports on the phone; user activity tracking | Salesperson role | Roles with a 4-digit passcode; restricted roles cannot edit or delete, back up, or see purchase price; invite by phone or email on a separate device | Offline-first locally, but its own desktop guide says to turn sync off before billing without internet; e-way bill, e-invoice and cloud sync need internet | [W S3, S4]; sync and mismatch complaints in a few reviews [Ws S5] |
| **Zoho Books** | Mobile app complements the web app; offline payments can be recorded | Role "Staff (Assigned Customers Only)" limits a user to their customers | Admin, Staff, Timesheet Staff, Assigned-Customers-Only staff; custom roles | Not stated | [W S17]; [Ws S18] |
| **SalesOn** (field-sales app, Tally sync) | Web dashboard for orders, collections, stock, field activity | GPS check-in, beat tracking, catalogue with schemes and live stock, orders, cash, cheque and UPI collection, GST invoices, thermal printing, end-of-day report, targets | Admin-created accounts only; no self-sign-up; the company must tell its staff about location use; about ₹208 per user per month billed annually | Not stated | [W S6] |
| **Bizom, BeatRoute** (brand and distributor SFA) | Dashboards for managers | Beat optimisation, offline selling, suggested orders, van sales, collections; BeatRoute routes special-price orders through approval workflows | Company-managed | Offline mode claimed | [Ws S16]; BeatRoute case study gives no numbers [W S15]; Bizom Next on iOS: 3.7 from three ratings [W S16] |
| **Khatabook, OkCredit** (khata apps) | Customer-wise balances, reminders over WhatsApp or SMS with a UPI pay option, voice entry, many languages | n/a | Single user | Offline records | [W S19]; features [Ws] |
| **Tally companions** (Bizeye, Accosync, Counts, Biz Analyst) | Tally reports and outstanding on the phone | Order booking written into Tally (Bizeye works with Tally.ERP 9 only) | Vendor-specific | Mostly not stated | [Ws S20] |
| **Price anchors** | BUSY mobile: ₹3,600 a year per device for one to four, ₹2,700 for five or more, plus GST [W S1]. SalesOn: about ₹208 per user per month [W S6]. | | | | The field seat in this market costs roughly ₹200 to ₹300 a month [J] |

### 4.2 Gaps that are a real opening, and the ones that are not

- **An owner whose office computer is off.** Tally's phone view stops [W S11]; BUSY's mobile needs the desktop to sync [W S1]. A cloud-synced, cached pulse is a clear gain. It is only a gain if the numbers equal the desktop report to the paisa (the Phase 28 rule [C CLAUDE.md]).
- **A credit check that still works offline, and says how stale it is.** One field-sales vendor's help page states its offline orders skip the limit and balances update only after sync [W S10]. Marg lists a credit-limit option on eOrder without saying what happens offline [W S7]. We own a credit engine with reasoned overrides [C `src/core/creditLimit.ts:1-15`].
- **Books-grade field sales in one system.** The field-sales specialists sync *to* an accounting package ("Tally sync" [W S6]); the accounting vendors bolt a salesman role on. Orders that book against the real ledger, with the real outstanding, are the thing neither side does cleanly. [J]
- **Not a gap: a beat planner or an SFA dashboard.** Bizom, BeatRoute, SalesOn and Marg all ship them. Competing on route optimisation is a v3 bet at best. [J]
- **Not a gap to claim: approvals on a phone.** BUSY ships voucher and master approval with conditions, though its mobile reach is unstated [W S2]; our own approvals design already says "PWA inbox, not native" [C `docs/research/approvals-workflows.md:23-24`].

### 4.3 How competitors give a salesperson access (and what we can learn)

| Pattern | Who | Lesson |
|---|---|---|
| Admin creates the user, the device activates with a code, OTP to change device | BUSY [W S1] | Device binding is normal in this market and owners accept it. |
| Admin creates the account; reps cannot self-register | SalesOn [W S6] | Same. |
| Invite by phone or email; separate device needs sync and a licence per device | Vyapar [W S3]; [Ws S5] | Per-device pricing makes owners ration seats. |
| Role presets with a fixed permission set, no per-permission editing | Vyapar (one reviewer complained) [Ws S5] | Owners want presets *and* per-rule tweaks. |
| Role limited to assigned customers | Zoho Books [W S17] | The row rule "own customers only" exists elsewhere and is understood. |

---

## 5. Design rules that follow for every persona

These are derived from sections 2 to 4 and the repo's own gotchas. Each persona section applies them and records only its deltas.

- **R1. Read-first for the owner, speed-first for the rep, inbox-first for the accountant, status-first for the CA, one-link for the party.** One product, defaults by role (the same idea as the UX analysis's "one product with two defaults" [C `docs/design/ux-analysis-2026-10.md` section 2]).
- **R2. Anyone but the owner and a trusted accountant holds a projection, not the books.** Sections 2 (5, 6, 9) and 8.
- **R3. Name the state of everything the phone holds.** Draft, Queued, Provisional (numbered offline), Confirmed, Held for approval, Rejected. A carried value must always be visible (the Phase 20 and 21 gotcha [C CLAUDE.md]): every figure carries an "as at" stamp and every unsent item a badge.
- **R4. Nothing posts without a person, and the server owns approval state** (the three principles in README section 2 and the approvals design [C `docs/research/README.md:136-138`]).
- **R5. Numbers come from the server; offline numbers are provisional** (the existing reserve, claim and release calls [C `backend/src/numbering.js:1-30`]). A company that must e-invoice takes **orders** offline and invoices after sync: an IRN and an e-way bill need the portal [W S4], and an invoice that should carry an IRN and does not is not valid [C `docs/research/einvoice-gsp.md:71`, `:75`; counsel to word it, README item 26].
- **R6. A collection is a legal and a fraud event.** Block or warn on cash at the ₹2 lakh line [Ws S47]; one same-day handover; UPI to the company's own QR only; a receipt number the company issues, not the rep.
- **R7. Track proportionately.** Location at check-in, check-out and collection, during a started working day, shown to the rep. No all-day trail in v1. The DPDP Act's employment ground (s.7(i)) is narrow, its notice duties start around 13 May 2027, and continuous tracking of staff is unsettled in law [W S49; Ws S49]; government staff in India have pushed back hard on always-on geo-fencing [Ws S50]. [J on the design]
- **R8. Speak the user's language and the user's channel.** Hindi first, lakh and crore, WhatsApp share, UPI, call buttons.
- **R9. A figure on the phone equals the report it opens on the desktop**, on the same window and as-at date (the Phase 28 blocker rule [C CLAUDE.md]). Otherwise the owner distrusts both.
- **R10. Budget data, memory and battery.** Compress photos; never push the whole book to a rep; cache on demand; no polling. Measure on a US$100 to 200 phone [W S37].
- **R11. Reuse the accessibility work.** Text-size tokens (`--ui-k`), validated accent contrast, high-contrast and reduced-motion already exist [C CLAUDE.md Phase 27]. A phone view built on them inherits the validation for outdoor, sunlight use; a rebuilt style sheet would not.
- **R12. Sequence by dependency, not by appeal.** TLS (S1), identity (S2), sync hardening (S3), notifications and PWA shell (S7), server engines (S10), numbering and drafts (S11) are the README's shared pieces [C `docs/research/README.md:98-113`]; section 13 maps each persona's versions onto them.

---

## 6. Persona 1: the business owner

### 6.1 Who

| Variant | Who | Share of attention |
|---|---|---|
| **O1 Trader, wholesaler or distributor owner** | Two to twenty staff, a counter, a godown, perhaps three to ten reps. Owns the credit decisions. Keeps Tally, BUSY or Marg at the office, an accountant to run it. | The core user. Every field-sales job in section 7 lands on this person as an approval or a worry. |
| **O2 Retail shop owner** | One or two counters, helpers, UPI-heavy, a khata for regulars. Mostly physically at the shop. | Wants the same pulse, fewer approvals, simple reminders. |
| **O3 Small manufacturer or MSME owner** | Fewer parties, larger invoices, raw-material vendors, MSME payment deadlines. | Payables and stock matter more than field sales. |

The UX analysis already names this person "B": mouse-first, from Zoho, Vyapar, BUSY or Excel, thinks in invoices and money, on the phone
half the day, afraid of jargon and of "is it saved?" [C `docs/design/ux-analysis-2026-10.md`, section 2]. The owner and the accountant
open the same books, so the product has to give them two defaults, not two products [C, same section]. Variants are **[J]**.

### 6.2 Context

| | |
|---|---|
| **Device** | Own Android, mid-range; an iPhone is more likely at higher turnover (a Bizom reviewer says managers use iPhones [Ws S16]) **[J]**. The phone is glanced at in front of customers, so amounts are visible to whoever stands at the counter. |
| **Network** | 4G or 5G at the shop and at home; weak in the godown and in transit; a phone hotspot often feeds the office PC [Ws]. Reads are tolerant of staleness if the stamp says so. |
| **Language** | Hindi or a regional language in conversation, English for software and numbers, lakh and crore everywhere. UI in English is acceptable to most owners, unlike for reps **[J]**. |
| **Shared phone** | Family firms hand the phone to a spouse, a son or a trusted clerk **[J, not evidenced here]**. Needs an app lock and a hide-amounts mode. |
| **Attention** | A glance of ten to thirty seconds between customers. One number, then back to the counter. |
| **Habit** | WhatsApp (97% in one survey [W S40]); phoning the accountant to ask a balance; asking staff to "send the ledger" [Ws]. |

### 6.3 A day in the life **[J, a composite; not observed]**

1. **Before the shop opens (8:40).** Opens the app at home: cash and bank, yesterday's sales, who is overdue. Five seconds. Decides whom to ring first.
2. **At the counter (9:30 to 13:00).** A customer asks for more stock on credit. The owner checks that party's outstanding and overdue on the phone; Tally's own owner guide uses exactly this example [W S12]. A rep rings for a bigger discount; the owner approves from behind the counter. A supplier chases a payment. The owner asks the accountant to WhatsApp a ledger to a party.
3. **Lunch (13:00).** Scans the overdue list, sends three reminders, notes one promise to pay.
4. **On the road (16:00).** Wants to know whether an item is in stock; clears two approvals from a notification.
5. **Closing (19:30).** Reads the day digest; matches the cash count to the book; sees what each rep handed over.
6. **Each month.** The 7th to the 11th is GST preparation; the 20th is the due date (or the 22nd and 24th for quarterly filers); a micro-supplier bill approaches 45 days [W S48, S46].

### 6.4 Top ten jobs, ranked by frequency times pain

| # | Job (when... I want... so...) | F | P | Score | Offline | Evidence | Today in this repo |
|---|---|---|---|---|---|---|---|
| 1 | **Cash and bank now.** Before I decide anything, show total cash and bank, split by account, with the time it is true as at. | 5 | 4 | 20 | Cached, stamped | Tally's owner list opens with cash [W S12]; BUSY sells "real-time" figures [W S1] | Dashboard Cash & Bank card, as-at date [C `registry.ts`] |
| 2 | **Who to chase today.** Show overdue parties, largest and oldest first, so I make the calls that matter. | 4 | 5 | 20 | Cached | 73-day average clearance [W S45]; BUSY overdue reminders [W S1]; Tally Bills Receivable [W S12] | Receivables and Overdue invoices cards; Reminders screen [C `access.ts:131`] |
| 3 | **Decide an exception.** A rep or accountant asks for a discount, a credit breach, a payment or a new party. Let me answer in two taps with enough context to be right. | 4 | 5 | 20 (for owners with reps; 8 without) | Needs the server | BeatRoute routes special-price orders through approvals [W S15]; BUSY approvals with conditions [W S2] | Credit override with a kept reason on desktop [C `creditLimit.ts:1-15`]; approvals designed, not built [C `approvals-workflows.md`] |
| 4 | **Party check before more supply.** Outstanding, overdue, last payment, credit limit, last three invoices, for one named party. | 4 | 4 | 16 | Cached | Tally's own example [W S12]; a Vyapar user's story of customers phoning for outstanding while he was on the road [Ws] | Ledger and outstanding screens; credit exposure engine |
| 5 | **Team oversight.** Which rep visited, ordered and collected today against target; what cash each is holding. | 4 | 4 | 16 (for owners with reps) | Cached | Marg eOwner locates salesmen and tracks field staff by orders [W S8]; SalesOn's owner dashboard [W S6]; the Pune case [W S51] | Nothing: no rep, visit or target object [C section 2, row 8] |
| 6 | **Sales against last month.** Today, this week and this month, against the same period before. | 5 | 3 | 15 | Cached | Tally's list [W S12] | Today widget, trend and Group Summary drill [C `registry.ts`] |
| 7 | **Nudge a party.** From the overdue list send a polite reminder or a statement on WhatsApp without retyping. | 4 | 3 | 12 | Compose offline, send when online | BUSY and Marg both list WhatsApp sharing [W S1, S7]; OkCredit reminders carry a UPI pay option [W S19 vendor] | WhatsApp click-to-chat for outstanding bills [C `share.ts:1-4`] |
| 8 | **What I owe, and when.** Bills due this week, and any micro or small supplier approaching its limit. | 3 | 4 | 12 | Cached | s.43B(h) [W S46]; Tally Bills Payable [W S12] | Payables card; no MSME flag (the company records its own Udyam number [C `src/types.ts:142`], but a supplier's class is not a ledger field) |
| 9 | **Stock watch.** What is low, what is fast, what is dead. | 3 | 3 | 9 | Cached | BUSY low-stock alarms [W S1]; Tally Stock Summary [W S12] | Stock widgets; reorder status screen [C `access.ts:136`] |
| 10 | **The compliance clock.** What is due, whether it is filed, roughly how much is payable. | 2 | 4 | 8 | Cached | Dates [W S48]; late fee per day [Ws] | GST payable card for the open period; no filing-month card yet [C CLAUDE.md Phase 28 "Deferred"] |

Owners without reps drop jobs 3 and 5 to the bottom and gain "forward an invoice or receipt" (F4, P2). Owners of O2 shops add "today's UPI
total against the bank" (F5, P3). **[J]**

### 6.5 What must work offline

An owner is online most of the day, so offline is a safety net, not a mode.

- **Opens with the last-known pulse in under two seconds, stamped** ("as at 09:12, 3 hours ago"). Never a blank screen, never a spinner on cold start.
- **Reads the cached overdue list, party cards and recent vouchers.**
- **Composes a WhatsApp reminder** from the cache; the person sends it when they have a signal (it is their own app).
- **Does not decide approvals offline.** An approval is a decision on server state; a queued one can meet an edited voucher. The screen says "needs a connection", not "saved" (R3, R4).

### 6.6 Privacy and control

The owner is the controller of everyone else's access (section 8). For their own phone:

- An app lock (device biometric or PIN) and an auto-lock after a short idle, because the phone is handed round.
- A **hide-amounts** mode on the home screen (tap to reveal) for use at the counter. **[J]**
- A device list with remote sign-out, so a lost phone is a one-tap problem (needs identity, S2).
- Lock-screen notification text private by default ("An approval is waiting"), with an owner switch for the detail, as the approvals design already proposes [C `approvals-workflows.md:57`].

### 6.7 Approval flows the owner receives

Full rule catalogue in section 12.1. What the owner needs on the phone:

- Two taps from the notification to a decision; the card shows who asked, the party, the amount, the reason, the party's current outstanding and the margin or cap the rule is guarding.
- **Step-up** (PIN again) above an amount the owner sets; **bulk approve** only below it [C `approvals-workflows.md:57`].
- **Delegate** to the accountant for a period (a holiday), with a cap.
- Never auto-reject; escalate or expire to "pending", so a rep's order is never lost (section 12.1).

### 6.8 Notifications the owner wants

| Event | Channel (v1 then later) | When | Default |
|---|---|---|---|
| Approval waiting | Push, in-app; WhatsApp later | Immediately | On |
| Day digest: sales, collections, cash and bank, top three overdue, open approvals | Push and in-app; WhatsApp later | A time the owner sets, around closing | On |
| A rep exceeds an approved limit or tries to collect cash at the legal line | Push | Immediately | On |
| A party crosses 30, 60, 90 days overdue | In-app list; digest | Daily | Digest only |
| A large receipt or payment above an amount | Push | Immediately | Off until set |
| GST due in three days and one day, with "not filed" | Push | Fixed dates | On |
| A micro or small supplier bill reaches day 35 of 45 (day 10 of 15 where there is no written agreement [Ws S46]) | Push | Daily check | On (needs the supplier class and the agreed terms) |
| Low stock of a flagged item | Digest | Daily | Digest only |
| A device has not synced for a day; a sync mismatch needs a decision | In-app; push for mismatches | On event | On |

Budget: no more than about five pushes a day; everything else rides the digest. Reliability and cost caveats are in section 12.2.

---

## 7. Persona 2: the salesperson

### 7.1 Who

| Variant | Where they work | Document they make | What differs |
|---|---|---|---|
| **S1 Field pre-seller (beat rep)** | A weekly beat of retail outlets; tens of calls a day | A **sales order**; another vehicle delivers, usually the next day | Order drift between booking and delivery (a vendor guide puts cancellations at 5 to 15%) [W S53 vendor]; collections are separate; the largest group **[J]** |
| **S2 Van-sales or delivery rep** | Sells from stock on a van | A **sales invoice** at the stop; cash or UPI at the stop | Load and unload, van stock and cash reconciliation, an e-way bill for a load over the threshold [Ws S52], fewer outlets a day, a thermal printer; same-day close is higher but outlets covered are lower [W S53 vendor] |
| **S3 Counter staff** | Wholesale or retail counter, godown | Invoice, return, held bill | Barcode and fast billing, price overrides, shift cash close, often a shared device on shop Wi-Fi with power cuts **[J]** |
| **S4 Key-account, project or dealer rep** | Fewer, larger accounts; sites, contractors, dealers (hardware, building materials, pharma stockists) | Quotation, then order; special prices | Special-price approvals [W S15]; a pipeline with follow-ups (leads, pipeline and follow-ups exist [C `src/ui/crm/`]); a long cycle |

All four: often on a personal phone, partly paid on collections or targets, often less comfortable in English than their owner. NSDC's
distributor-salesman pack (2016) describes the role as visiting outlets on a daily route plan with handheld selling aids, finding new
outlets, executing point-of-sale visibility, and learning credit management [Ws S59].

### 7.2 Context

| | |
|---|---|
| **Device** | Mostly a ₹10,000 to ₹19,000 Android, 3 to 4 GB of RAM, a screen in sunlight, one hand free [W S37; J]. Personal phones are common (BYOD), which is why tracking and data access need care (R7, section 8). |
| **Network** | "No signal at the outlet" is the normal case on a beat, not an edge case. Weak data plans (under ₹500 a month for half of surveyed owners [W S40]). |
| **Language** | Hindi or a regional language first; English numerals. Labels must fit a short word, not a sentence. **[J]** |
| **Shared phone** | S3 yes (the counter device), S1 and S2 usually no, but a phone may be reassigned when a rep leaves. **[J]** |
| **Attention** | Standing in a shop with the owner of the outlet waiting. A rep who is slower than a paper pad will go back to the pad. |
| **Incentives** | Pay depends on orders, collections and targets, so disputes about what was recorded are common. **[J]** |

### 7.3 A day in the life of a field pre-seller **[J, a composite built from S6, S7, S53 to S55 and S59]**

1. **Start (9:00).** Taps Start day. Sees today's stops in order with flags: overdue, last visit 9 days ago, new scheme, party over limit. The phone syncs while the rep has Wi-Fi at the depot.
2. **At each outlet (about 10 minutes of a 14-minute slot).** Checks in. Opens the party card: dues, overdue bills, credit headroom, last order. **If overdue, collects first** (cash, UPI or cheque) and issues a receipt. Then repeats last week's order, edits quantities, applies the price level and scheme, takes the order. Shares the order on WhatsApp with the retailer. Logs the outcome in one tap.
3. **A discount request** from the retailer is above the rep's cap. The rep sends it for approval and goes on to the next outlet; the order waits as pending.
4. **A new shop** on the way: the rep adds it as a draft party with the GSTIN and a photo; the office completes it.
5. **End (18:00).** Syncs. Hands the day's cash to the cashier against the app's expected cash figure. Sees today's orders, collections and the month-to-date against target.

**Arithmetic that sets the speed budget [J on vendor ranges]:** vendor-published ranges for urban general trade are 22 to 30 calls a day in one reference table [Ws S53] and 25 to 40 in another vendor's KPI blog [W S55], with five to six productive selling hours a day [Ws S53]. These vendors disagree with each other and none names its data, so read them as an order of magnitude. At 25 calls in 6 hours each call gets about 14 minutes including travel and talk. An order that takes 60 seconds is under a tenth of that; one that takes three minutes is a fifth, and a rep will skip it. This is why the target in section 14 is 60 seconds, not "fast".

**The van seller's day differs** in four ways: loads the van and records the load at the depot (stock and a delivery challan, an e-way bill if the load crosses the threshold [Ws S52]); sells and collects at every stop (an invoice, not an order); returns unsold stock; and reconciles cash and stock at the end. Their "day close" is the most important screen they have.

### 7.4 Top ten jobs, ranked by frequency times pain (field pre-seller)

| # | Job | F | P | Score | Offline | Evidence | Today in this repo |
|---|---|---|---|---|---|---|---|
| 1 | **Take an order for a known party, fast.** Repeat the last order, change quantities, apply price level and scheme, save. | 5 | 5 | 25 | **Yes** | Every field app lists order taking [W S1, S6, S7]; orders still reach many distributors by phone and WhatsApp [W S54] | Ghost recall, price levels, Sales Order base, Bulk Entry [C section 2, row 10] |
| 2 | **See dues, overdue and credit headroom before I order.** Stop me selling more to a party on hold, or tell me I need approval. | 5 | 5 | 25 | **Yes**, stamped, and re-checked at sync | Reps take orders from accounts on credit hold [W S54]; offline credit gap [W S10] | Credit exposure and check engine [C `creditLimit.ts`]; no cached projection |
| 3 | **Know my day.** My stops in order, with a flag on each, and the next stop one tap away. | 5 | 4 | 20 | **Yes** | Beat plans drawn by hand and rarely updated [W S54]; Marg's route tracking and SalesOn's beat plans [W S6, S7] | None (no beat object) |
| 4 | **Collect against a bill.** Cash, UPI or cheque against named bills; a receipt the retailer can keep. | 4 | 5 | 20 | **Yes** (UPI confirmation needs the network) | Marg lists bill-by-bill collection [W S7]; SalesOn lists cash, cheque and UPI [W S6]; the Pune fraud [W S51] | Receipt voucher, bill-wise engine, UPI QR [C `upi.ts`]; no rep cash ledger |
| 5 | **Close the day.** Expected cash against handed over; unsent items; today's totals. | 5 | 4 | 20 | **Yes** | The same fraud pattern; owners ask retailers to verify [W S51] | Nothing |
| 6 | **Share it.** The order, invoice, receipt or statement to the retailer on WhatsApp, in one tap. | 5 | 3 | 15 | Compose offline | BUSY, Marg and SalesOn all list WhatsApp [W S1, S6, S7] | Click-to-chat for vouchers and outstanding bills [C `share.ts`] |
| 7 | **Log the visit.** Visited, ordered or not (and why), with time and place, in under five seconds. | 5 | 3 | 15 | **Yes** | The admin burden of field software is about 21% of a rep's week in one vendor's research [W S56 vendor]; Salesforce's 2023 survey has reps selling only about 28% of the week [Ws S57] | Nothing |
| 8 | **Get an exception approved while I stand there.** | 3 | 5 | 15 | Queue; answer needs the network | BeatRoute special-price approvals [W S15] | Credit override reason, desktop only |
| 9 | **Check stock, price and scheme before I promise.** | 5 | 3 | 15 | Cached status | Marg live stock [W S7]; SalesOn stock before ordering [W S6] | Stock positions; price levels; no scheme engine |
| 10 | **See targets, achievement and incentive.** | 4 | 2 | 8 | Cached | Marg target vs achievement [W S7]; SalesOn targets [W S6] | None |

Next five **[J]**: add a new outlet (F2, P3); returns and damages (F2, P3); a quotation for a bigger buyer (F2, P3); a photo as proof of a
cheque, shelf or delivery (F3, P2); a promise-to-pay note (F3, P3).

**Deltas for the other variants:**

| Variant | Jobs that move up | Jobs added |
|---|---|---|
| **S2 Van sales** | Collect (to 1), day close (to 1), stock check | Load and unload the van; van stock count; delivery challan and e-way bill for the load; Bluetooth thermal print; returns of unsold stock |
| **S3 Counter staff** | Take an order becomes "bill in seconds" with barcode and item search | Hold and resume a bill; price override with approval; return against an invoice; shift cash close; reprint |
| **S4 Key-account rep** | Exception approvals (to 1), share a quotation | Quotation to order conversion; site and contact notes; follow-up reminders; dealer onboarding |

### 7.5 The offline contract

A rep's phone must keep working in four conditions: no signal, weak signal, a signal that returns mid-task, and a phone that restarts mid-order.

| Works with no network | Needs the server | Degrades, and says so |
|---|---|---|
| Open the day, the stops and each party card from cache; take, edit and save orders; record a receipt (cash, cheque, UPI claim with a reference); log visits; add a *draft* party; compose WhatsApp shares; day close totals from the local queue | Real approval decisions; confirming a UPI credit against the bank; a final server voucher number; invoice and e-way bill generation for e-invoicing companies; stock truth across reps | Credit check: against the cached exposure **plus the rep's own unsent orders**, marked "provisional, checked again at sync"; stock: shown as a status with a time; price and scheme: as of the last sync, with a stale flag after a set age |

Rules that follow:

- **A rep may edit an order only until the office accepts it.** After sync the server's copy rules and a clash becomes a visible conflict, never a silent overwrite (the sync guard and mismatch report already follow this idea [C CLAUDE.md Phase 24]).
- **Numbers.** Reserve on accept when online; take a provisional local number offline and claim it on return; a claim that finds the number held by another voucher keeps the voucher and records a mismatch [C `backend/src/numbering.js:1-30`]. A pre-reserved block per device is the alternative some vendors use, at the cost of wasted numbers [W S22].
- **Credit check at sync:** an order that would breach the limit when it reaches the server is **held for the owner**, not rejected and not posted. The rep sees "held" the next time the phone connects. The owner may set an "offline credit allowance" per party or per rep (extra headroom permitted offline, to be re-checked) **[J]**.
- **Time limits:** sensitive cached data (outstanding amounts) expires after a number of days the owner sets; after that the party card shows names only until the next sync.
- **Photos and notes** queue separately from orders so a large upload never blocks an order.
- **Storage:** a home-screen web app on iOS keeps its own count of days of use, which actual use resets, while a site opened in Safari can have its script-written storage (IndexedDB included) deleted after seven days of Safari use without interaction [W S29, S28]. An unsent day of orders must not live in a browser tab; the rep must install the app, and the app should refuse to call a queue "safe" if it is not installed (on iOS). Chromium browsers evict only under storage pressure and offer `navigator.storage.persist()` [W S28].
- **Sync order on reconnect:** receipts first (money), then orders, then visits and photos. **[J]**

### 7.6 Privacy limits and the business rules owners apply

What a salesperson should and should not hold. Matrix and enforcement are in section 8; this is the list of typical rules, each a pattern
seen in competitors or a standing owner concern.

**Typical rules (default on):**

1. No purchase cost, standard cost, margin or supplier rates. Vyapar's own page says restricted users cannot see purchase price [W S3].
2. Only assigned parties (Zoho has a role limited to assigned customers [W S17]). Other reps' customers, orders and targets are invisible.
3. No cash, bank or company-level balances; no payroll; no GST filing data; no users, PINs or settings.
4. Prices only from the party's own price level; no other level.
5. A party's dues as the owner decides: exact amount, a band ("within limit", "near limit", "over"), or hidden (section 8.2).
6. Stock as a status ("in stock", "low", "out") or an exact number, the owner's choice.
7. No editing a party's master data; "add a party" creates a draft that the office completes.
8. No deleting; cancel only with a reason, and only before dispatch.
9. Discounts only up to a cap per rep and per item group; above it is an approval (section 12.1).
10. Cash: a daily cap per rep, a hard stop at the ₹2 lakh line per party, a same-day handover (R6).
11. A receipt number is the company's, not the rep's.
12. Location is taken at check-in, check-out and collection, during a started day, and the rep can see what the boss sees (R7).
13. Party phone numbers are visible so the rep can call; the owner can switch to "call from the app only" so a leaving rep cannot take a contact list **[J, a standing owner worry, not evidenced here]**.
14. A leaving rep's access ends in one action, and the phone's cached data is wiped at the next open.

### 7.7 Approval flows (rep side)

- A request is made in one tap from the order or collection screen with a reason picker (competitor price, bulk, old stock, relationship).
- The order is saved as **Pending** (the existing optional bit keeps it out of every ledger, stock and GST total [C `docs/research/approvals-workflows.md:18-22`]) and the rep moves on.
- The rep sees the answer as a notification and a badge. On approval the order becomes live; on rejection the rep can edit and resubmit.
- When the owner cannot be reached, the order stays pending and appears on the day-close list; it is never auto-approved.

### 7.8 Notifications the rep wants

| Event | Channel | Default |
|---|---|---|
| Start-of-day nudge with the number of stops | Push, in the rep's language | On, set by the rep |
| Approval decision on a request | Push | On |
| A party you are about to visit has just paid or promised | Push | On |
| An order was held at sync (credit) | Push and badge | On |
| An order was dispatched or delivered | Push | Off |
| Target milestones (50%, 80%, 100%) | In-app | On |
| Cash handover reminder at a set time | Push | On |
| Items waiting to sync for more than a few hours | Badge, then push | On |
| A scheme or price list changed | In-app on next open | On |

No more than about five a day; local language; no amount in a lock-screen text by default.

### 7.9 Trade differences that change the item picker and the order screen

| Trade | What differs | Evidence | Consequence for v1 |
|---|---|---|---|
| **FMCG distribution** | Many SKUs, schemes by paper circular or WhatsApp, pre-sell with next-day delivery, strike rate and lines per call as the managers' measures | Schemes by circular and rep memory [W S54]; vendor ranges of 60 to 75% strike rate and 4 to 6 lines per call [W S55 vendor] | Last-order repeat, scheme display, lines per call as a manager metric |
| **Pharma stockist** | Batch and expiry, Schedule H records, retailer credit of 45 to 60 days against 15 to 20 days from suppliers, about 5% of stock returned for expiry (one stockist's notes) | One stockist's wiki page [Ws S21]; GoFrugal and LOGIC ERP describe batch and expiry in the rep app [Ws S21] | Batch and expiry shown in the item picker; credit days on the party card |
| **Garment and textile** | Size and colour grids, design catalogues shared on WhatsApp, orders as voice notes, double-selling when an order is lost in a group chat | Vendor pages [Ws S21] (marketing) | A matrix entry grid and a shareable catalogue are v2, not v1; the risk of double-selling argues for reserving stock at order |
| **Hardware, electrical, building materials** | Many SKUs by brand and size, frequent price-list changes, tiered discount approvals, project quotes | BeatRoute's Utkarsh case: special-price approvals, dealers, contractors and architects as accounts [W S15]; Odoo role-based discount caps and a minimum-threshold pattern [Ws S21] | Price-list versioning with an effective date; tiered approvals; quotation to order |

Our stock model already carries `batches`, `prices` per level and `altUnit` [C `src/types.ts:488-540`]; size-colour matrices do not exist.

---

## 8. Privacy, permissions and the decision about what sits on a phone

This section is shared by every persona. It answers the owner's first question about giving a rep the app: "what will they be able to
see, and what happens when they leave?"

### 8.1 Who may hold what

Legend: **F** full · **R** read-only · **A** assigned or own only · **B** a band or status instead of a number (owner's choice) · **L** look up one party at a time · **P** the party's own records only · **X** none.

| Data | Owner | Accountant | Field rep (S1, S2) | Counter staff (S3) | Key-account rep (S4) | CA (auditor) | Party |
|---|---|---|---|---|---|---|---|
| Company financials, cash and bank balances | F | F | X | X (own shift cash only) | X | R | X |
| Purchase cost, margin, supplier rates | F | F | **X** | **X** | **X** | R | X |
| Every party's ledger and outstanding | F | F | X | L | X | R | X |
| Assigned parties' bills, overdue, credit headroom | F | F | A (amounts exact; limit as a band) | L | A | R | P |
| Other reps' customers, orders, targets | F | F | **X** | X | **X** | R | X |
| Selling prices | F | F | A (own price level) | A (levels the owner allows) | A | R | P (on own invoices) |
| Stock quantity | F | F | B | exact | B | R | X |
| Payroll and HR | F | by right | X | X | X | by engagement | X |
| Users, PINs, roles, API keys, security settings | F | X | X | X | X | X | X |
| GST returns and filing data | F | F | X | X | X | by engagement | X |
| Audit and edit log | F | R | X | X | X | R | X |
| Party phone, address, GSTIN | F | F | A (or call-only) | L | A | R | own |
| Own orders, receipts, visits, target, incentive | F | F | **F (own)** | F (own shift) | F (own) | R | X |
| Rep location and visit trail | F during working windows | X | own (sees what the boss sees) | X | own | X | X |

This is the typical policy pattern in the products compared in section 4: hidden purchase price (Vyapar [W S3]), assigned-customers-only
(Zoho [W S17]), admin-created accounts only (BUSY, SalesOn [W S1, S6]). The matrix itself is **[J]**.

### 8.2 The settings owners want

Defaults are the safe choice for a business with reps; every one is the owner's to change. **[J]** except where a pattern is cited.

| Group | Setting | Default | Notes |
|---|---|---|---|
| **Visibility** | A rep's view of a party's dues: exact bills, limit hidden, headroom as a band ("within", "near", "over") | On | A rep needs bill amounts to collect, not the credit limit figure |
| | Stock as a status or an exact number | Status for reps, number for counter | |
| | Prices from the party's own price level only | On | |
| | Party phone visible, or call-from-app only | Visible | The second is for owners who fear a leaving rep taking the list |
| | How many past orders the rep sees (3, 10, 30) | 5 | |
| **Limits** | Maximum discount per rep and per item group | 0%, so any discount is an approval | Odoo modules and a Vtiger example cap discounts by role and route larger ones upward [Ws S21] |
| | Price edit: none, within a percentage, any | None | |
| | Maximum order value without approval | Owner sets | A US electrical distributor's app has a configurable threshold below which no approval applies [Ws S21] |
| | Offline credit allowance per party or per rep | 0 | Extra headroom allowed offline, re-checked at sync |
| | Cash collection: allowed or not; daily cap per rep; hard stop at the ₹2 lakh line per party per day | Allowed, capped, hard stop on | The ₹2 lakh line is the receiver's liability [Ws S47]; a CA must confirm whether any override should exist |
| | Cheques: post-dated allowed; photo of the cheque required | On | |
| | UPI: to the company's own QR only; reference mandatory | On | |
| | New party: draft only, completed by the office | Draft | |
| | Edit or cancel an order: only before dispatch, with a reason | On | Cancel with a kept reason already exists [C `src/ui/cancel/`] |
| | Back-dated entries: none, or up to N days | None | The period lock and date windows exist in roles [C `src/types.ts:683-697`] |
| **Tracking** | Working-day window; location only inside it | On | R7 |
| | Location at check-in, check-out and collection, not a continuous trail | On | A periodic trail is an opt-in with a written notice [Ws S49] |
| | Geofence at check-in: warn or block | Warn | Outdoor GPS drifts by metres and more indoors [Ws S50 vendor] |
| | Selfie at start of day | Off | Selfie check-in is standard in field apps; the sensitivity is the owner's call **[J]** |
| | Location and visit retention (days) | Owner sets; counsel to set the floor | |
| | The rep sees "what my manager sees" | Always on | |
| **Device and session** | Devices per person | 1 | |
| | Enrolment by owner-issued code; revoke; sign out and wipe at next open | On | BUSY uses an activation code and OTP on device change [W S1] |
| | Cached amounts expire after N offline days | 7 | Best effort: a rooted phone can read its own cache |
| | App lock and idle auto-lock | On | |
| | Rep may share PDFs only for own parties | On | |
| | Screenshot blocking | n/a on the web | A native Android feature; a PWA cannot do it **[J, not verified here]** |
| **Approvals and alerts** | Approver, delegate, escalation timeout, step-up amount | Owner, none, 30 min then pending, above ₹ set | Section 12.1 |
| | Quiet hours, digest time, language, per-event on or off | Quiet 22:00 to 07:00 | Section 12.2 |

### 8.3 What enforces each control, and what exists today

| Control | It must live | Today |
|---|---|---|
| Hide cost and other reps' customers; "own parties only" | **Server**: the rep's device must never receive the data | **No.** Rights are client-side [C `access.ts:12-14`]; the pull sends every row [C `sync.js:343-372`] |
| Below-cost and price-floor checks | **Server**, at sync (the rep has no cost) | No |
| Credit re-check of an offline order | **Server**, at sync | The mismatch machinery exists; no credit re-check [C CLAUDE.md Phase 24] |
| Approval state | **Server** is the only writer (the README principle) | Designed, not built [C `docs/research/README.md:136-138`] |
| Discount caps | Client for feedback, server re-check | Voucher guards are client-side |
| Working-day window | Client starts it; server accepts location only inside it | No |
| Device enrolment and revocation | **Server** | No (one login per tenant [C `tenancy.js:10`]) |
| Cached-data expiry and wipe | Client, best effort | No |
| Operator PIN for a shared counter device | Client | Yes, an opt-in model [C `access.ts`] |

The honest limit: anything cached on a phone can be read by someone who holds the phone and the skill. **Minimise what is cached** (a
projection), rather than hide what is cached.

### 8.4 The decision: how does a rep's phone get data?

| Option | What it is | For | Against |
|---|---|---|---|
| **A. Snapshot plus proposals (recommended for S1, S2, S4) [J]** | At start of day (and on demand) the phone downloads a **server-built snapshot**: assigned parties with open bills and a headroom, items with this rep's price levels and a stock status, schemes, a block of voucher numbers. Orders, receipts, visits and draft parties go back as **proposals** to dedicated endpoints; the server validates them (credit, floor, discount cap, cash line) and creates **pending** drafts in the books for a person (or an owner's auto-accept rule) to confirm | Real privacy; a small payload for a ₹10,000 phone and a thin data plan; no two-way merge; no cost on the device; fits "nothing posts without a person" [C README:136-138]; the credit and floor checks have a home | A second read model that must equal the books (R9); new server endpoints; needs identity (S2), server engines (S10) and TLS (S1); the largest single item |
| **B. Thin online client on REST `/v1` with a per-rep API key** | Reuse the open-platform API with a cache and an offline queue | No sync changes | Keys have only `read` and `write` scopes, no collection or row scope [C `backend/src/platform.js:83-89`]; any writable collection is open to a key [C `restPolicy.js:18-24`]; the offline queue must be invented |
| **C. Trusted-staff mode: full sync, operator PIN, client rights (today's model)** | What exists, once HTTPS is in | Earliest to ship | Every row of section 2 (5, 6, 9); PIN hashes on the device; a leaving rep carries the whole customer list and cost prices. Acceptable only for **counter devices the business owns** and for owner-run teams of two or three who accept the warning, never marketed as "restricted" |

The shared pieces this needs are in README section 2: S1 TLS (2 to 4 days), S2 team and identities (1.5 to 2 weeks), S3 sync hardening (2.5 to 3
weeks), S7 notifications and PWA shell (1.5 to 2 weeks), S10 server engines (1 to 1.5 weeks) [C `docs/research/README.md:98-113`]. My sum of
those as the README sizes them is **about 7 to 9 engineer-weeks, plus or minus 30%, before any rep screen or the snapshot endpoints**. S3
and part of S2 serve the six consultant asks too, so this is not all mobile's bill. **[J]**

### 8.5 Identity and shared phones

| Person | Mechanism | Why |
|---|---|---|
| Owner, accountant, CA | A tenant invite with a role (S2); the owner can see and revoke members | Today the owner shares one password, so there is no attribution and no revocation [C `tenancy.js:10`] |
| Field rep, key-account rep | An **owner-issued enrolment code or QR** for "Ravi, field rep, these parties", binding one device to a rep role; revoke from the owner's phone; no password to share. Like linking WhatsApp Web **[J]** | Matches how BUSY and SalesOn already gate access [W S1, S6]; works for people without an email address |
| Counter staff | Shared device with the existing operator PINs and a fast user switch; each sale attributed to the operator; auto-lock | The operator model exists [C `access.ts`] |
| Party | A signed, expiring link with an OTP; no account in v1 | See section 11 and `docs/research/customer-vendor-portal.md` |

### 8.6 Cash controls: the fraud lesson

Three Indian cases describe one pattern: an employee who sells and collects keeps the money. In the Pune and Ludhiana cases the shortfall
surfaced when retailers were asked or said they had already paid; the Bhopal case surfaced in a ledger audit. The Pune case (reported 30 Aug 2024, alleged, under police investigation): a seven-year employee handling orders,
deliveries and collections is said to have kept about ₹11.8 lakh from 14 retailers and another ₹17.2 lakh from a related business, over
three months [W S51]. Reports from Ludhiana (a commission salesman) and Bhopal (two sales agents) have the same shape [Ws S51]. They are allegations, not findings.

| Control | In the product |
|---|---|
| A receipt number issued by the company, never by the rep | Server-assigned numbers [C `backend/src/numbering.js`] |
| Cash with each rep is a **ledger** until handed over | A "cash with rep" ledger (or a first-class rep field on vouchers) and a day close that compares expected to handed over **[J]** |
| The retailer is told | A WhatsApp receipt to the retailer's number at collection; the retailer can dispute a wrong amount. The Pune and Ludhiana shortfalls surfaced only when retailers were asked or said they had paid [W S51; Ws S51] |
| UPI only to the company's QR | The owner's VPA already prints on invoices when enabled [C `upi.ts:1-30`]; a dynamic per-invoice QR with a reference needs a registered-merchant VPA and the `tr` field that `upi.ts` deliberately leaves out, so it is v2 |
| Reconcile UPI to the bank by UTR, not by the rep's screenshot | A screenshot can be edited; match to the company's bank credit [Ws S60] |
| Separation of duties | A rep cannot cancel or alter a receipt after it is shared |
| Aged-receivable anomalies and leave-taking | A report of receipts re-applied to older bills and of reps with no days off is the accountant's lapping check [Ws S51] **[J]** |
| Pay on cleared money | The incentive report counts banked collections, not entered ones **[J]** |

---

## 9. Persona 3: the accountant or bookkeeper on the move

### 9.1 Who

An in-house accountant at a trading business (often a Tally power user, keyboard-first: "A" in the UX analysis [C `docs/design/ux-analysis-2026-10.md`
section 2]) or an outsourced bookkeeper serving several small clients. "On the move" means the bank, a customer's office, the GST
office, home in the evening. The phone is for triage and capture, not data-entry marathons: Tally positions the phone as a place to read
reports and send documents, not to run accounting [W S12]; its Docs by Ira turns photographed bills into drafts that the accounts team
reviews [Ws S14]; BUSY lists accountants and CAs handling several clients from mobile [W S1].

### 9.2 Context

| | |
|---|---|
| **Device and network** | Own Android or iPhone; reliable data mostly; evenings on home Wi-Fi |
| **Language** | English for software, a regional language with colleagues; exact figures matter more than speed |
| **Shared phone** | No, but the phone is high-value: it can see every ledger (section 8.1) |
| **Habit** | Keyboard on the desktop; the phone only for what cannot wait |

### 9.3 A day in the life **[J]**

Morning at the bank (matches credits to open bills). Mid-day at a customer's office: a party's accounts clerk asks for a ledger; the
accountant WhatsApps a statement. Afternoon: three reps' day closes arrive; the accountant checks cash handed over and UPI credits
against what they entered. Evening: photographs vendor bills for tomorrow, accepts two orders into invoices, and sees GST and TDS dates.

### 9.4 Top ten jobs

| # | Job | F | P | Score | Offline | Evidence | Today in this repo |
|---|---|---|---|---|---|---|---|
| 1 | **Verify reps' day close.** Cash handed over against receipts entered; UPI credits against UTR. | 5 | 4 | 20 | Cached | The fraud pattern [W S51]; reconciliation by UTR [Ws S60] | None (no rep object) |
| 2 | **Post a receipt or payment from the phone.** From a bank SMS, a cash slip, a cheque photo. | 5 | 3 | 15 | **Yes**, as a draft | Quick-create registry [C `quickCreate.ts`] | Voucher screen is desktop-shaped |
| 3 | **Answer "send me my ledger".** Statement or outstanding as a PDF on WhatsApp. | 5 | 3 | 15 | Compose offline | BUSY shares ledgers and bill receivables [W S1] | Share builder for vouchers and outstanding bills [C `share.ts`] |
| 4 | **Work the inbox.** Drafts from reps, bills captured, approvals assigned to me, items sent back. | 5 | 3 | 15 | Cached | README S8 One Inbox [C `README.md:107`] | Not built |
| 5 | **Capture a vendor bill.** Photograph it; it becomes a draft voucher for review. | 4 | 3 | 12 | **Yes** (OCR needs the network) | Tally's Docs by Ira [Ws S14] | Not built (README attachments design, `docs/research/attachments-drives.md`) |
| 6 | **Turn orders into invoices and dispatch.** Accept a rep's order, create the invoice, an e-way bill. | 4 | 3 | 12 | Needs the server for e-way | BUSY lists e-way bill and e-invoice from the phone [W S1] | Sales Order, pending orders, dispatch panel [C section 2] |
| 7 | **Clear bank exceptions.** Unmatched credits and debits. | 3 | 4 | 12 | Cached | README bank design [C `docs/research/live-bank-feeds.md`] | Statement import and reconcile exist on desktop |
| 8 | **Check GST and TDS dues and exceptions.** | 3 | 4 | 12 | Cached | Dates [W S48] | GST validation and TDS reports on desktop |
| 9 | **Look up a voucher, fix a mistake, see who changed it.** | 4 | 2 | 8 | Cached | Edit log exists [C `access.ts:150`] | Desktop |
| 10 | **A stock query for a sales call.** | 3 | 2 | 6 | Cached | Tally Stock Query [W S12] | Desktop |

### 9.5 Offline, privacy, approvals, notifications

- **Offline:** capture drafts (bill photos, cash slips) and look up cached ledgers; no heavy reports offline.
- **Privacy:** can post, cannot delete; the period lock binds; payroll and the owner's drawings need their own right [C `access.ts:143-148`]. The phone must be an enrolled device with an app lock and remote wipe, because it holds the full books.
- **Approvals:** the accountant is the checker for rep receipts and the maker for vendor-bill drafts; the owner may delegate small approvals to them with a cap (section 12.1).
- **Notifications:** drafts waiting; a rep submitted a day close; an unmatched bank credit; GST and TDS due; an item sent back; a sync mismatch (the mismatch report is administrators-only today [C `access.ts:155`]). Email digest as well as push.

---

## 10. Persona 4: the chartered accountant reviewing

### 10.1 Who and context

A CA or tax consultant with 20 to 200 small clients, mostly at a desk; the phone is for quick checks, a client's question, and due-date
anxiety. Tally models this person as the **Auditor** role on a Tally.NET ID with report-by-report access, one ID can be authorised across several companies [W S11], and remote access lets an auditor verify the books [W S13]; BUSY says accountants and CAs can handle several clients from mobile [W S1]; Zoho Books offers accountants their own seat [Ws S18]. The evidence for how CAs use a
*phone* is thin: **[J]** for the day below.

Today the only way to give a CA access is to share the owner's login, so there is no read-only role, no access log and no instant revoke
[C `tenancy.js:10`]. Every CA job below depends on identity (S2).

### 10.2 Top ten jobs **[J]**

| # | Job | F | P | Score | Notes |
|---|---|---|---|---|---|
| 1 | **Pre-filing readiness across clients.** Draft GSTR-1 and 3B against books and GSTR-2B, TDS due, before the date. | 3 | 5 | 15 | The due dates are the 11th and the 20th [W S48]; the busy windows before them are **[J]** |
| 2 | **Ask or answer a query on a voucher** with the client, kept with the voucher. | 4 | 3 | 12 | Replaces WhatsApp threads with no record |
| 3 | **Review exceptions.** Validation, RCM, ineligible ITC, mismatches. | 3 | 4 | 12 | GST validation and reconciliation reports exist [C `access.ts:139-142`] |
| 4 | **Pull a report as a PDF.** Trial balance, P&L, ledger, GST summary. | 4 | 3 | 12 | |
| 5 | **Switch client in a few seconds.** | 5 | 2 | 10 | Tally allows one ID across companies [W S11] |
| 6 | **Due-date calendar across clients.** | 4 | 2 | 8 | |
| 7 | **Who changed what.** Audit trail on one entry. | 2 | 4 | 8 | Edit log exists [C `access.ts:150`] |
| 8 | **Sign off a period.** Request or approve a lock through a date. | 2 | 4 | 8 | The lock exists [C `types.ts:698-710`] |
| 9 | **Confirm a balance with a party.** | 2 | 3 | 6 | Portal [C `customer-vendor-portal.md`] |
| 10 | **Share a working paper** with the client. | 3 | 2 | 6 | |

### 10.3 Offline, privacy, approvals, notifications

- **Offline:** not needed in v1. Online-only is acceptable for a person who has a desk; a cache would put a client's books on a phone the client does not own.
- **Privacy:** read-only per company, time-boxed access, a visible access log the client can read, instant revoke. A processor, not a controller, under the DPDP Act (counsel item in README section 7E).
- **Approvals:** the CA approves a period lock and may be a named approver for specific voucher classes; never for day-to-day sales.
- **Notifications:** "ready for review" from the client; due-date reminders at three days and one day; a query answered. Email is the natural channel.

---

## 11. Persona 5: the party (customer or vendor), as a later portal

Only a later portal: the research already exists in `docs/research/customer-vendor-portal.md`, which recommends a cloud-only,
read-only, passwordless portal on its own origin, built after TLS, team identities and the isolation suite (Wave 3 in the README
sequence [C `docs/research/README.md:200-282`]). This section lists the jobs so the owner and rep screens leave room for them.

| # | Job | F | P | Score | Notes |
|---|---|---|---|---|---|
| 1 | **See what I owe and the bills behind it** (the ledger a retailer would otherwise ask for on WhatsApp) | 3 | 4 | 12 | Zoho ships a customer portal on every plan [C README:80] |
| 2 | **Pay**, with the right reference | 3 | 4 | 12 | A UPI link on the owner's own account; Comhard never holds the money (README principle [C `README.md:139-142`]) |
| 3 | **Reorder what I took last time** | 4 | 3 | 12 | The party-side twin of a rep's "repeat last order"; WhatsApp and a link **[J]** |
| 4 | **Get a copy of an invoice or receipt** | 3 | 3 | 9 | The e-invoice QR and IRN matter for the buyer's input credit |
| 5 | **Know the delivery status** | 3 | 3 | 9 | Dispatch panel data [C `src/ui/dispatch/`] |
| 6 | **Query a line** | 2 | 4 | 8 | |
| 7 | **Returns and credit note status** | 2 | 3 | 6 | |
| 8 | **See a scheme or offer** | 3 | 2 | 6 | |
| 9 | **Confirm a balance at period end** | 1 | 3 | 3 | |
| 10 | **Get GST documents for input credit** | 2 | 3 | 6 | Connect design [C `comsky-connect-network.md`] |

- **Privacy:** the party sees only a server-computed view of its own ledger, never raw vouchers: one bulk receipt or journal carries other parties' data [C README:82-88].
- **Offline:** none needed.
- **Notifications:** invoice issued; due in three days; overdue at one, seven and thirty days; payment received. The reminder carries the pay link (the khata apps do the same [W S19 vendor]).
- **v1 (portal):** a signed link and an OTP to a read-only statement and invoice copies. **v2:** pay link. **v3:** reorder.

---

## 12. Approvals and notifications across personas

### 12.1 Approval rule catalogue

Every threshold belongs to the owner; the defaults below are a safe starting point **[J]**. A pending item uses the existing `optional`
bit so every ledger, stock, GST and statutory total already ignores it [C `docs/research/approvals-workflows.md:18-22`].

| ID | Trigger | Asked by | Decided by | Default | If the answer is slow or the phone is offline | Pattern |
|---|---|---|---|---|---|---|
| A1 | Discount above the rep's cap, per line or per invoice | Rep, counter | Owner or sales manager | Cap 0% | Order saved **Pending**; rep moves on; never auto-approved | Role-based caps routed upward [Ws S21]; BeatRoute approvals [W S15] |
| A2 | Price below the minimum price (checked on the server, the rep never sees cost) | Rep | Owner | Block, request allowed | Pending | [J] |
| A3 | Order pushes a party over its credit limit | Rep, counter | Owner | Warn, reason required | Held at sync for the owner; the rep sees "held" | Credit override with a kept reason exists [C `creditLimit.ts:1-15`]; offline gap elsewhere [W S10] |
| A4 | A party past N days overdue asks for fresh credit | Rep | Owner | Block new credit orders | Held | Credit-hold accounts still get orders [W S54] |
| A5 | New party or outlet | Rep | Office | Draft only | Draft stays a draft | [J] |
| A6 | Return, credit note, damage replacement | Rep, counter | Accountant or owner above ₹ set | Approval above ₹ set | Pending | [J] |
| A7 | Cash receipt over the rep's daily cap, or aggregate to one person at or past ₹2 lakh | Rep | Not approvable at ₹2 lakh (the law); above the rep's cap, the owner | Hard stop at the line | Blocked on the spot: the screen says to take UPI, a cheque or a transfer | Receiver liability [Ws S47]; CA to confirm |
| A8 | Cancel or alter an order or invoice after dispatch | Rep, accountant | Owner | Approval | Pending | Cancel with reason exists [C `src/ui/cancel/`] |
| A9 | A back-dated entry | Anyone but the owner | Owner | Block | Blocked | Role date windows exist [C `src/types.ts:683-697`] |
| A10 | Free goods or a scheme the party does not qualify for | Rep | Owner | Block | Pending | Scheme compliance is a vendor KPI [W S55 vendor] |
| A11 | Payment out above an amount (supplier, expense) | Accountant | Owner | Above ₹ set, step-up PIN | Pending | Approvals design [C `approvals-workflows.md:57`] |
| A12 | Stock adjustment or write-off | Storekeeper, accountant | Owner | Approval | Pending | [J] |

Mechanics: one request card (who, what, party, amount, reason, the party's outstanding, the rule it breaks); two taps to decide;
step-up above an amount; bulk approve only below it; escalate to a delegate after a timeout; **never auto-reject** (the order stays
pending, never lost); a note when the owner edits the figure ("approved at 6%"); a log of who decided from which device.

### 12.2 Notification channels and what to expect from each

| Channel | Best for | Needs | Caveats | Evidence |
|---|---|---|---|---|
| **In-app inbox and badge** | Everything, as the source of truth | Nothing | Seen only when the app is open | [J] |
| **Web push** | Owner approvals, rep decisions | HTTPS, a manifest, a service worker, VAPID keys; on iOS 16.4 or later, the app must be added to the Home Screen and permission asked on a tap [W S30, S31] | On vivo, OPPO, realme and Xiaomi phones background delivery depends on user-side Autostart and battery settings [W S34]; Background Sync is supported by Chrome and Samsung Internet (browsers used by 76.66% of people worldwide) and not by Safari or Firefox [W S25] | Section 3 |
| **WhatsApp click-to-chat** | A rep or owner sharing to a party | Nothing | The person presses Send; it is not a notification [C `share.ts:1-4`] | [C] |
| **WhatsApp Business Platform** | Owner digest, party reminders, receipts | A BSP and Meta approval; templates by category (marketing, utility, authentication, service) | Replies within 24 hours are free; INR rates unverified; the README quotes ₹0.13 to 0.15 a message, unchecked [W S43; C `README.md:190-192`] | [W S43] |
| **SMS** | OTP, a fallback to a low-end phone | A sender and, in India, template registration (DLT) **[J, not verified here]** | Cost; language limits | |
| **Email** | Accountant and CA digests | A sender account with SPF and DKIM [C README S7] | Slow for urgent items | |

Rules: critical items **also** sit in the inbox, because push is a hint, not a guarantee; an app-icon badge counts open items (iOS supports it
once permission is given [W S30]); lock-screen text is private by default; no more than about five pushes a person a day; quiet hours;
the language is the user's.

---

## 13. Prioritised feature lists, v1, v2 and v3, by persona

How versions were ordered: by the job table's score first, then by dependency (what the code and the platform allow), then by risk
(money and privacy before polish). "Depends on" uses the README's shared pieces: **S1** TLS, **S2** team and identities, **S3** sync
hardening, **S7** notifications and PWA shell, **S9** blob store, **S10** server engines, **S11** numbering and drafts [C
`docs/research/README.md:98-113`].

### 13.1 The business owner

| Ver | Feature | Why now | Depends on | What exists |
|---|---|---|---|---|
| **v1** | **Pulse**: cash and bank, receivables and overdue, payables and due this week, today's sales, receipts and payments, a "needs you" count; each figure stamped "as at"; cached | Jobs 1, 2, 6 (scores 20, 20, 15). Closes the gap that Tally's phone view leaves when the office PC is off [W S11] | S1; a phone layout (the UX analysis's N11); a cloud copy of the books (exists) | Dashboard v2 engines and cards [C `registry.ts`] |
| v1 | **Party card**: search a party; bills, overdue, last payment, limit, last three invoices, call and WhatsApp buttons | Job 4 (16) | As above | Ledger and outstanding engines; credit exposure |
| v1 | **Overdue list with a prefilled WhatsApp reminder** the owner sends | Jobs 2 and 7; closes the loop with no Meta approval | As above | `share.ts` [C] |
| v1 | **Find an invoice or receipt and share it** (PDF through the share sheet where the browser allows, else a link) | Everyday | S1 (Web Share needs a secure context [Ws S35]) | PDF output [C `src/ui/pdfOut.ts`] |
| v1 | **App lock, hide-amounts, freshness stamp, device list** | R3 and section 6.6 | S2 for the device list | |
| v1 | **Day digest in-app and by email** | Gives the owner a reason to open the app | S7 | |
| **v2** | **Approvals inbox with push**, step-up, delegate, bulk below a limit | Job 3 (20 with reps); the approvals design [C `approvals-workflows.md`] | S2, S3, S7 | Designed |
| v2 | **Team board**: visits, orders, collections and cash with each rep against target | Job 5 (16) | Rep objects (section 7), S2 | None |
| v2 | **Quick entry**: receipt, payment, expense, invoice | Owners at the counter or on the road | S11 drafts | Quick-create registry [C] |
| v2 | **Promise-to-pay notes and follow-up reminders** on overdue bills | Job 2 | None | CRM follow-ups [C `src/ui/crm/followups.tsx`] |
| v2 | **Micro and small supplier flag** and a "pay by" list | Job 8; s.43B(h) [W S46] | A supplier-class field on the ledger (new) | None |
| v2 | **Filing-month GST card** with the due date and status | Job 10 | None | GST payable for the open period; the filing-month card is deferred [C CLAUDE.md Phase 28] |
| v2 | **Multi-company switcher**; WhatsApp digest through a Business Platform provider | | A BSP and Meta [W S43] | |
| **v3** | Hindi voice questions ("Sharma Traders ka outstanding?"); cash forecast and anomaly alerts; payment links on the owner's own gateway; bank-feed balances; scheduled reports to the accountant; branch consolidation | Each is a bet on habit, not a gap in a core job | Bank feeds, gateway contract, voice engine | Designs exist for bank feeds and payments [C `docs/research/live-bank-feeds.md`] |

### 13.2 The salesperson (field pre-seller first)

| Ver | Feature | Why now | Depends on | What exists |
|---|---|---|---|---|
| **v1** | **Enrolment by an owner-issued code; start day; today's stops** (assigned parties in the owner's order; no optimiser) | Job 3 (20); identity and a snapshot are the gate for everything else | S1, S2, S10, the snapshot endpoint (section 8.4) | None |
| v1 | **Party card offline**: bills, overdue, headroom band, last order, last payment | Job 2 (25) | Snapshot | Credit and exposure engines |
| v1 | **Order capture**: repeat last order, search by name, alias or part number, qty, price level, scheme shown, save as a pending sales order with a provisional number | Job 1 (25), the daily task | S11 numbering; snapshot | Ghost recall, aliases and part numbers, price levels, numbering [C section 2, row 10; `types.ts:488-540`] |
| v1 | **Collect**: cash, cheque or UPI against named bills, a receipt, a hard stop at the ₹2 lakh line, UPI to the company's QR | Jobs 4 and 5 (20); the fraud lesson (section 8.6) | Snapshot, proposals | Receipt voucher, bill-wise, UPI QR [C `upi.ts`] |
| v1 | **WhatsApp share** of the order, receipt and statement | Job 6 | S1 for the share sheet | `share.ts` [C] |
| v1 | **Visit log**: outcome, reason for no order, time and place, one tap | Job 7; the data owners and managers ask for | A visit object (new) | None |
| v1 | **Day close**: expected cash against handed over, unsent items, today's totals | Job 5 (20) | A rep cash ledger (new) | None |
| v1 | **Hindi and English strings; large targets; an "N unsent" badge** | Language is the first barrier for a rep [W S39] | An i18n layer (new) | English-only [C section 2, row 11] |
| v1 | **Ask the owner** for a discount or credit exception as a Pending order with a reason; the status shows on next open | Job 8, without the push dependency | Pending orders | Optional bit [C] |
| **v2** | **Push approvals**; targets and incentive view; photo proof (cheque, delivery, shelf) with compression; new-outlet onboarding with a GSTIN check | Jobs 8, 10; proof against disputes | S7, S9 | GSTIN check digit exists [C `src/core/gstin.ts`] |
| v2 | **Barcode scan**, **Bluetooth thermal print**, **van-sales mode** (load, unload, van stock, delivery challan, e-way) | Needed by the van and counter variants (S2, S3). Both are Chromium-on-Android features in practice: `BarcodeDetector` is experimental, absent in Firefox and behind a flag in Safari, and Web Bluetooth is absent in Safari and in Firefox on Android [W S26, S27] | A fallback scanner library; or a thin native shell | None |
| v2 | **Scheme engine** (effective-dated, rule-based), **quotation to order**, returns and damages | FMCG and hardware needs [W S54, S15] | | None |
| v2 | **Dynamic UPI QR per invoice** | Reconciliation by reference | A registered-merchant VPA and the `tr` field `upi.ts` omits | Static QR [C] |
| **v3** | Suggested orders from history; route optimisation; Hindi voice order entry; claims and scheme settlement; incentive statements; a retailer self-order link; offline maps | The specialists already ship suggested orders and beat optimisation [Ws S16]; compete only after the core is trusted | Data volume | Ghost recall is the seed of suggested orders [C `recall.ts`] |

### 13.3 The accountant

| Ver | Feature | Why now | Depends on |
|---|---|---|---|
| **v1** | **Inbox**: rep day closes, receipts to verify, drafts; statement or outstanding as a WhatsApp PDF; quick receipt and payment drafts offline; GST and TDS due list; voucher lookup with the edit log | Jobs 1 to 4 (20, 15, 15, 15) | S2 identity, rep proposals, S8 |
| **v2** | Bill photo to a draft; order to invoice and e-way bill; bank exceptions list; bulk verification of day closes; delegated approvals with a cap | Jobs 5 to 7 | S9 blob store, the bank design, OCR (attachments design) |
| **v3** | WhatsApp capture (forward a bill to a number to make a draft); lapping and anomaly reports; GSTR-2B match on the phone | Habit bets | WhatsApp Business Platform; Connect design |

### 13.4 The CA

| Ver | Feature | Why now | Depends on |
|---|---|---|---|
| **v1** | **Client list with readiness chips**; report viewer and PDF share (trial balance, P&L, ledger, GST summary); due-date calendar; read-only auditor role with an access log | Jobs 1, 4, 5, 6 | S2 (an auditor role, multi-tenant switch), S10 for the chips |
| **v2** | Queries on a voucher with the client; an exception list; period-lock request and sign-off | Jobs 2, 3, 8 | S2, S8 |
| **v3** | Cross-client dashboard; notices tracker; balance confirmation through the portal | | Portal |

### 13.5 The party

| Ver | Feature | Why now | Depends on |
|---|---|---|---|
| **v1 (portal)** | A signed link with an OTP to a read-only statement and invoice copies | Job 1 | S1, S2, S4, S10 and the external security review [C README:200-282] |
| **v2** | Pay link on the owner's own gateway; raise a query on a line | Jobs 2, 6 | A gateway account; the owner's own keys |
| **v3** | Reorder what I took last time; delivery status; scheme view | Jobs 3, 5, 8 | Dispatch data |

### 13.6 The ladder: what unblocks what

| Rung | Needs | Unlocks | Note |
|---|---|---|---|
| **0** | S1 TLS, a responsive phone layout, a manifest and a service worker | **Owner v1** (pulse, party card, overdue plus WhatsApp reminder, share) | The owner **is** the single tenant login today, so no identity work is needed. This is the cheapest rung and it is the buyer. |
| **1** | S2 identities and device enrolment, S3 sync hardening, S7 push | Accountant v1, CA v1, **Owner v2** (approvals with push) | The most shared rung; S2 and S3 serve all six consultant asks |
| **2** | S10 server engines, S11 numbering and drafts, the snapshot-and-proposals endpoints (8.4) | **Salesperson v1** | The largest single piece after S2 |
| **3** | Rep objects (visit, beat, target, rep cash), S9 blob store | Salesperson v2, accountant v2 | |
| **4** | S4 isolation suite, the external security review | Party portal | Wave 3 in the README |

**Cheap and useful before rung 0 is finished (no regret) [J]:** the shell's first responsive slice (hide the F-key rail under 1100 px,
the left menu as a drawer under 900 px [C `ux-analysis-2026-10.md:513`]); "as at" and "N unsent" stamps; extracting English strings so Hindi
can follow; defining the rep snapshot and the proposal payloads on the sample company and measuring their size; and the rep object
model (a first-class salesperson field on a voucher against a cost-centre stand-in).

**Platform notes the personas imply, as input to the architecture work, not a decision.** A PWA first is already the plan [C `README.md:351`].
It can be put on the Play Store as a Trusted Web Activity (which needs a PWA that meets Chrome's criteria and a Digital Asset Links
file) [W S32]. Features that work on Chromium browsers on Android and not on iOS or Firefox (barcode, Bluetooth printing, Background Sync) argue for a thin native
shell **only if** the field pilot shows scanning, printing or background sync is a must-have [W S25, S26, S27]. iOS is under 7% of web
traffic [W S36], but owners and managers are likelier to hold iPhones, so owner v1 must work there; web push on iOS needs a Home Screen
install [W S30].

---

## 14. The single most important screen of each persona, and how to know it works

### 14.1 The screens

**Owner: the Pulse (home).** Opens from cache, never blank. Everything is a tap to the party card, the list or the approval.

```
 Sharma Traders  v        as at 09:12 (3 min ago)    [lock] [eye]
 ---------------------------------------------------------------
  CASH + BANK                          Rs 4,82,300
  Cash 61,200  .  HDFC 4,21,100
  ---------------------------------------------------------------
  TO COLLECT  Rs 18.4 L      6.2 L overdue > 30 days   [Chase 7]
  TO PAY      Rs  9.1 L      1.8 L due this week
  TODAY       Sales 1.42 L . Received 0.86 L . Paid 0.31 L
  ---------------------------------------------------------------
  NEEDS YOU (3)
   > Discount 8% . Verma Stores . Ravi            [Approve] [No]
   > Order over limit . Gupta Hardware . Anil     [Review]
   > Cash short Rs 2,000 . day close . Anil       [Review]
 [ Party ]  [ Overdue ]  [ Approvals 3 ]  [ + New ]
```
The figures are illustrative. Design notes: a "stale" chip when the stamp is old; "hide amounts" masks every figure until tapped; a figure
that cannot be computed offline says so instead of showing zero (R3).

**Salesperson: the Visit screen (party card and actions together).**

```
 Verma Stores            Stop 4 of 22        [offline] [3 unsent]
 ---------------------------------------------------------------
  Owes Rs 42,300     Overdue Rs 18,000 (27 days)    Limit: NEAR
  Last order 6 days ago . Rs 21,400 . 7 lines
 ---------------------------------------------------------------
   [ TAKE ORDER ]     [ REPEAT LAST ORDER ]
   [ COLLECT ]        [ SHARE STATEMENT ]
   [ NO ORDER: why? ]
 ---------------------------------------------------------------
  Open bills                              Rs        Days
   INV-1042                             18,000        27   [Collect]
   INV-1077                             24,300         9   [Collect]
```
Design notes: collect sits next to order because overdue is the first conversation; limit is a band by default; every button works with
no signal; large targets, two languages, no menu to dig through.

**Accountant: the Inbox.** One list: rep day closes to verify, drafts to post, approvals assigned to me, items sent back, bank exceptions;
a count in the app badge. This is README S8's One Inbox on a phone [C `README.md:107`].

**CA: the Clients list.** One row per client with chips: GSTR-1 and 3B status and due date, TDS due, queries open, last synced; a tap
opens that client's read-only reports. Switching is one tap back.

**Party: the Statement.** The balance first, the bills behind it, a Pay button when enabled, a copy of each invoice. No login screen.

### 14.2 Measurable success criteria

Every target is **[J], to be proven in a spike on the lab rig in 14.3**; none is a measurement. Tests run on a device in the US$100 to 200
band [W S37] with an old Chrome, 4 GB of RAM and a throttled connection.

**Owner**

| Metric | Target | How measured |
|---|---|---|
| **Cold start to today's cash and bank on screen** | **5 s** on throttled 4G; **2 s** from cache with a stale label | Time from the icon tap (process killed) to the figure drawn with its stamp; median and p90 over 20 runs per device |
| Party search to outstanding and overdue | 10 s cold, 3 s warm | Scripted flow, the same runs |
| **Figure agreement with the desktop report** | **100% to the paisa**, same window and as-at date | An automated reconciliation suite on the sample company and a 50,000-voucher synthetic company, as the Phase 28 suites already do for the dashboard [C CLAUDE.md] |
| Approval | At most 2 taps from the notification; median time to decision to be learned in the pilot, a first target of 10 minutes in business hours; at most 10% of requests older than an hour | Server timestamps: requested, opened, decided |
| Reminder | At most 3 taps from the overdue list to the WhatsApp composer | Scripted flow |
| Adoption | At least 60% of enrolled owners open the pulse on 4 or more days a week from week 4 | Pilot telemetry (privacy-reviewed) |
| Outcome | Days to clear an invoice falls against the company's own baseline (the market vendor figure is 73 days [W S45]) | Receivables over sales times days, per company, month on month; no target until a baseline exists |

**Salesperson**

| Metric | Target | How measured |
|---|---|---|
| **Order of five lines for a known party, offline** | **60 s median, 90 s p90**; at most 25 taps | Scripted flow in airplane mode from the party tap to "saved"; then timed in the field against the diary |
| Party card with dues and headroom, offline | 2 s from the tap | Cache read, 20 runs |
| Beat list from a cold start, offline | 3 s | As above |
| Collection against one bill, then a WhatsApp receipt | 30 s, then at most 3 taps | Scripted flow |
| **A day with no loss** | 0 lost orders, 0 lost receipts, 0 duplicate numbers across a simulated day: 60 orders, 40 receipts, three reconnects, two restarts mid-order | An extension of the existing simulated-device numbering harness [C `backend/src/numbering.js:1-4`, `scripts/assert/vchnumbering.assert.ts`] |
| Credit honesty | 100% of offline orders re-checked at sync; any breach held for the owner within a minute | Server test with a seeded over-limit party |
| Queue clears | At least 99% of queued items reach the server within 10 minutes of connectivity; the unsent count is visible at all times | Device logs in the pilot |
| **Privacy by construction** | A rep device receives **zero** rows of cost, other reps' parties, users or PIN hashes | An automated check over the snapshot payload, run on every build |
| Adoption | At least 80% of beat visits logged in the app by week 4, against the diary | Pilot telemetry |
| Admin burden | Time in the app that is not an order, a collection or a visit log: 10 minutes a day at most. For scale, one vendor's research puts rep admin at about 21% of the week [W S56 vendor], and another vendor's survey says nearly 40% of reps think field technology hinders more than it helps [Ws S58 vendor] | Time-in-screen counters |
| Cash discipline | At least 95% of rep cash handed over the same day; unexplained variance at day close of at most 0.5% of collected | Day-close records; calibrate in the pilot |
| First snapshot size | Stated in MB and met on a throttled connection | Measured in the snapshot spike; no figure exists yet |

**Accountant, CA, party**

| Metric | Target | How measured |
|---|---|---|
| Verify ten reps' day closes | 10 minutes | Timed in the pilot |
| Post a receipt draft from the phone | 45 s | Scripted flow |
| A statement as a WhatsApp PDF | At most 5 taps, 20 s | Scripted flow |
| CA opens one client's filing readiness | 10 s; switching client 3 s | Scripted flow |
| CA access granted or revoked | Effective within 1 minute, and every access logged, 100% | Server tests |
| Party sees the correct outstanding from the WhatsApp tap, no login | 15 s | Scripted flow |
| Reminder effect on payment within 7 days | Uplift against a no-reminder group, size to be learned | A controlled pilot with the owner's consent |

### 14.3 The rig, and who may stop the release

- **Devices:** two Android phones in the US$100 to 200 band with 4 GB of RAM (one vivo, OPPO or Xiaomi to expose background limits [W S34]), one Android Go-class, one iPhone.
- **Networks:** offline; throttled 3G; flaky (a drop every 30 seconds); and a Wi-Fi to 4G handover during an order.
- **Instrumentation:** local performance marks, anonymised. Today's perf harness times engines on a desktop and says it is not time to paint [C `docs/PERF.md:1-9`], so a **mobile time-to-paint harness is a gap to build first**; the `--cpu-factor=4` projection is only a guess.
- **Field pilot:** three distributors, three reps each, four weeks, against each rep's diary baseline.
- **A release is stopped if:** any order is lost or duplicated; any pulse figure differs from its desktop report; a rep device holds a forbidden row; the offline order median stays above 90 seconds after two design iterations (then the rep app needs a native shell, not more polish).

---

## 15. Validation plan, decisions needed and what is unverified

### 15.1 Twelve sessions

| Who | How many | Method |
|---|---|---|
| Owners: two distributors with reps, one retail shop, one small manufacturer | 4 | Contextual inquiry at the counter: "show me the last three times you approved a discount, chased a payment, checked a balance"; their real phone and apps |
| Reps: two field pre-sellers, one van seller, one counter clerk | 4 | A ride-along or a day diary; "show me your last order pad"; time an order on paper and on the prototype; ask about their own phone, data plan, language and what annoys them about the owner's rules |
| Accountants | 2 | The day close and the bill pile; how they get bills; shared logins |
| A CA with small clients | 1 | How a client's books are reached today; what a CA would not allow on a phone |
| A party (a retailer who buys from one of the owners) | 1 | "How do you find out what you owe?" |

Questions that would change this document: What is the first number an owner looks at? What did a rep once do that made the owner stop
trusting the books? Do reps use their own phones and a shared one at the counter? Would an owner pay ₹200 to ₹300 a month per rep [W S1, S6]?
Which language does each rep read? What do owners hide from reps, and what do reps say they need that the owner would hide?

### 15.2 Decisions for the owner

1. **Sequence:** the owner's phone first (needs TLS and a layout, no identity), the rep's second (needs identity and the snapshot)? **[J, recommended]**
2. **Rep data:** snapshot and proposals (section 8.4 A), with trusted-staff mode (C) only for counter devices the business owns?
3. **Identity:** owner-issued device enrolment for reps, tenant invites for the accountant and CA?
4. **Languages:** Hindi plus which regional language first, driven by the pilot's trade and state?
5. **Rep model:** a first-class salesperson field on a voucher, or cost centres as a stand-in for the pilot?
6. **Cash at the ₹2 lakh line:** a hard stop with no override, or an override with a reason? (A CA to decide.)
7. **Location policy and retention:** defaults in section 8.2, reviewed by counsel before the pilot (the DPDP notice duties start around 13 May 2027 [W S49]).
8. **Shell:** PWA with a Play Store wrapper, and a thin native shell only if the pilot demands scanning, printing or background sync?
9. **Pilot trade:** FMCG, pharma, hardware or garment first? Each changes the item picker (section 7.9).
10. **Pilot partners:** which three distributors, and are they Tally, BUSY or Marg users today?

### 15.3 Unverified, or only as good as a vendor's page

- Which UPI receipts the new merchant discount rate touches, in particular B2B invoice payments [W S41; unverified for B2B].
- Meta's INR price per message; the README's ₹0.13 to 0.15 is a third-party figure [C `README.md:190-192`].
- The DLT registration requirement for commercial SMS in India [J, not checked here].
- Whether a PWA can block screenshots (I believe not; not verified).
- The penalty wording under s.186 of the Income-tax Act 2025 [Ws S47].
- Every Marg, Bizom, SalesOn and BUSY claim about offline behaviour, credit handling and approvals on mobile; the pages are silent or promotional [W S1, S6, S7, S8].
- All "field sales" benchmarks (strike rate, lines per call, calls per day, admin time, cancellations): vendor-published, unattributed, and they disagree [W S53, S55, S56].
- The Pune, Ludhiana and Bhopal cases: allegations in press reports [W S51; Ws S51].
- The pharma stockist figures (margin, 5% expiry returns, credit days): one stockist's notes [Ws S21].
- Opensignal's India figures: search summaries only [Ws S44].
- The Vyapar sync complaints: a few review excerpts [Ws S5].

### 15.4 Spikes worth doing before building (read-only or in the user's own tree)

1. Measure time to paint of the Dashboard pulse on a US$150 phone with the whole-company load of the sample company and of a 50,000-voucher synthetic company.
2. Build the rep snapshot for the sample company and measure its size and its build time on the server.
3. Prototype the offline order flow (party card, repeat last order, save) with the numbering simulation, and time it with a person who is not a developer.
4. List every field a rep screen would read, against the data classes in section 8.1, to size the projection.

---

## 16. Sources

Tags: **[W]** fetched and read · **[Ws]** search summary only, the page itself was not read. All checked 2026-10-21 or 2026-10-22. Everything is paraphrased.

**Products and competitors**

| ID | Source | Tag |
|---|---|---|
| S1 | BUSY Mobile App, https://busy.in/mobile-accounting-software | W |
| S2 | BUSY voucher and master approval, https://busy.in/accounting-software/user-approval.md | W |
| S3 | Vyapar user roles, https://vyaparapp.in/blog/user-roles-permissions-vyapar/ | W |
| S4 | Vyapar offline billing guide, https://vyaparapp.in/guides/how-to-make-bills-offline-in-vyapar | W |
| S5 | Vyapar review excerpts (Capterra, Trustpilot), https://www.capterra.com/p/180579/Vyapar/reviews/ and search snippets; the pages returned 403 | Ws |
| S6 | SalesOn mobile app, https://saleson.co.in/mobile-app | W |
| S7 | Marg eOrder (App Store), https://apps.apple.com/us/app/eorder/id1631309291 | W |
| S8 | Marg eOwner (App Store), https://apps.apple.com/us/app/-/id6476924466 | W |
| S9 | Marg FMCG Android listing, https://www.softwaresuggest.com/fmcg-distribution/mobile-app | W |
| S10 | FieldPro credit management, https://learn.fieldproapp.com/en/article/credit-management-80cqe5 | W |
| S11 | Tally: TallyPrime on the phone, https://tallysolutions.com/tally/tallyprime-mobile-experience-for-business-owners/ (published 30 Sep 2026, updated 14 Oct 2026) | W |
| S12 | Tally: what owners can do on a phone, https://tallysolutions.com/smart-moves/expert-insights/using-accounting-software-on-your-phone-what-business-owners-can-actually-do/ | W |
| S13 | Tally Remote Access, https://help.tallysolutions.com/tally-prime/connected-services/remote-access-tally/ | W |
| S14 | Tally Docs by Ira, https://tallysolutions.com/ai-for-business/tallyprime-bulk-invoice-upload-docs-by-ira/ | Ws |
| S15 | BeatRoute, Utkarsh India case study, https://beatroute.io/resources/case-studies/utkarsh-india/ | W |
| S16 | Bizom Next (App Store), https://apps.apple.com/us/app/bizom-next/id6446275531 (W: 3.7 from three ratings); feature list and G2 and Capterra excerpts from search summaries | W and Ws |
| S17 | Zoho Books users and roles, https://www.zoho.com/en-fr/books/help/settings/users.html | W |
| S18 | Zoho Books offline payments and plans, https://www.zoho.com/books/videos/payments/offline-payments.html and https://www.zoho.com/globalpayments/ca/books/ | Ws |
| S19 | OkCredit kirana page, https://okcredit.in/khata-app-for-kirana-store (vendor claims) [W]; MediaNama, OkCredit CEO interview, 2021, https://www.medianama.com/2021/05/223-okcredit-merchants-interview/ [W]; Khatabook and OkCredit feature summaries [Ws] | W and Ws |
| S20 | Tally companions (Bizeye, Accosync, Counts, Biz Analyst) from search summaries, e.g. https://apps.apple.com/us/app/-/id1029363563 | Ws |
| S21 | Trade vendors: LOGIC ERP, GoFrugal and RichAutomate (pharma), App4Sales and RichAutomate (garment), SalesOn, Odoo discount modules, Vtiger, Gross Electric (hardware); a stockist's notes at https://increff.atlassian.net/wiki/x/V4G0H | Ws |
| S22 | Swipe, offline billing for field teams, https://getswipe.in/blog/collections/best-mobile-billing-app-offline-field-sales-teams (vendor) | W |

**Platform**

| ID | Source | Tag |
|---|---|---|
| S23 | MDN, Secure contexts, https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts | W |
| S24 | MDN, Features restricted to secure contexts, https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts/features_restricted_to_secure_contexts | W |
| S25 | MDN Background Synchronization API, https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API; MDN browser-compat-data for `ServiceWorkerRegistration`; caniuse background-sync data (76.66% global) | W |
| S26 | MDN Barcode Detection API, https://developer.mozilla.org/en-US/docs/Web/API/Barcode_Detection_API; browser-compat-data for `BarcodeDetector` | W |
| S27 | browser-compat-data for Web Bluetooth (`api/Bluetooth.json`) | W |
| S28 | MDN, Storage quotas and eviction, https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria | W |
| S29 | WebKit, https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/ (seven-day cap; home-screen web apps count separately) | W |
| S30 | WebKit, Web Push for web apps on iOS and iPadOS, https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/ | W |
| S31 | web.dev, Installation criteria, https://web.dev/articles/install-criteria | W |
| S32 | web.dev, Using a PWA in your Android app (Trusted Web Activity), https://web.dev/articles/using-a-pwa-in-your-android-app | W |
| S33 | web.dev, Flipkart Lite, https://web.dev/case-studies/flipkart (an older case; 63% on 2G) | W |
| S34 | dontkillmyapp, Xiaomi, https://dontkillmyapp.com/xiaomi [W]; vivo, OPPO and realme pages from search summaries [Ws] | W and Ws |
| S35 | Web Share API, https://developer.mozilla.org/docs/Web/API/Web_Share_API; web.dev, Web Share (files, secure context, a user gesture) | Ws |

**Market and people**

| ID | Source | Tag |
|---|---|---|
| S36 | StatCounter, India mobile OS share (Sept 2026), https://gs.statcounter.com/os-market-share/mobile/india | W |
| S37 | IDC, India smartphones Q2 2026, https://www.idc.com/resource-center/press-releases/india-smartphone-shipments-decline-q2-2026/ | W |
| S38 | Counterpoint, India smartphone share, https://www.counterpointresearch.com/india-smartphone-share/ | W |
| S39 | Business Today on the Kantar-IAMAI Internet in India 2024 report, https://www.businesstoday.in/amp/technology/news/story/indias-internet-revolution-key-insights-from-kantar-and-iamai-report-461043-2025-01-16 (the PDF was unreadable) | W |
| S40 | Storyboard18 on PayNearby's MSME Digital Index 2024, https://www.storyboard18.com/digital/over-85-of-small-business-owners-using-smartphones-for-biz-ops-report-35559.htm (sample size not stated); the hotspot figure from a CRN summary [Ws] | W |
| S41 | Outlook Money, UPI September 2026, https://www.outlookmoney.com/banking/upi-processes-2407-billion-transactions-worth-rs-2937-lakh-crore-in-september | W |
| S42 | TechCrunch, WhatsApp Business in India, https://techcrunch.com/2024/09/12/whatsapp-brings-meta-verified-customized-messages-to-small-businesses-in-india | W |
| S43 | WhatsApp Business Platform pricing categories, https://whatsappbusiness.com/hi/products/platform-pricing/ (INR rates not shown) | W |
| S44 | Opensignal India reports, https://insights.opensignal.com/reports/2026/02/india/mobile-network-experience (403; search summaries) | Ws |

**Business, tax and law**

| ID | Source | Tag |
|---|---|---|
| S45 | MediaBrief on Recordent's Indian SME Receivables Report 2026, https://mediabrief.com/recordent-report-highlights-working-capital-stress-for-indian-smes-as-average-overdue-receivables-cross-%E2%82%B93-83-crore/ (vendor-published) | W |
| S46 | TaxGuru on the Rajya Sabha reply of 21 July 2026 on s.43B(h), https://taxguru.in/income-tax/finance-ministry-clarifies-section-43bh-45-day-msme-payment-rule.html; the 15-day no-agreement rule from TaxGuru and Taxmann summaries [Ws] | W and Ws |
| S47 | Cash rules: s.269ST now s.186 of the Income-tax Act 2025 (in force from 1 April 2026), and s.40A(3): TaxGuru, Taxmann, ebizfiling summaries; incometaxindia.gov.in returned 403 | Ws |
| S48 | SoftwareSuggest, GST due dates 2026-27, https://www.softwaresuggest.com/blog/?p=133460 (a third-party calendar; the portal is final) | W |
| S49 | King Stubb and Kasiva on employee monitoring under the DPDP Act, https://ksandk.com/data-protection-and-data-privacy/can-employers-monitor-employees-dpdp-act/ [W]; other law-firm commentary [Ws] | W and Ws |
| S50 | Geo-fencing attendance pushback (Tribune, Amar Ujala, MediaNama on ASHA workers) | Ws |
| S51 | Bridge Chronicle, Pune drug distributor, 30 Aug 2024, https://www.thebridgechronicle.com/news/wholesale-drug-distributor-defrauded-by-employee-29-lakh-missing [W]; Ludhiana (Tribune) and Bhopal (Hitavada) [Ws]. All are allegations | W and Ws |
| S52 | e-way bill and van sales: ICAI rule text, KnowYourGST forum, BUSY, Gimbooks | Ws |

**Field-sales practice and research (mostly vendors)**

| ID | Source | Tag |
|---|---|---|
| S53 | Rework, van sales vs pre-sell, https://resources.rework.com/ms/libraries/fmcg-distribution-growth/van-sales-vs-pre-sell [W]; Rework field-sales economics and field force sizing tables [Ws] | W and Ws |
| S54 | MAssistCRM, FMCG distribution challenges, https://blog.massistcrm.com/fmcg-distribution-challenges-how-smart-field-operations-fix-what-manual-processes-break | W |
| S55 | MAssistCRM, ten FMCG KPIs with benchmarks, https://blog.massistcrm.com/10-kpis-every-fmcg-sales-manager-must-track-with-benchmarks | W |
| S56 | SPOTIO, mobile CRM for field sales, https://spotio.com/blog/mobile-crm/ | W |
| S57 | Salesforce State of Sales (reps spend about 28% of the week selling in the 2023 edition) | Ws |
| S58 | Skynamo field-sales survey claims (nearly 40% say technology hinders; nearly 90% distrust their captured data) | Ws |
| S59 | NSDC and RASCI distributor salesman qualification pack (RAS/Q0604), https://www.nsdcindia.org/scmp/assets/image/1884524164-8._Distributor_Salesman_English_PH_Preview.pdf (unreadable; summary from search) | Ws |
| S60 | UPI collection and UTR reconciliation guidance (Razorpay, PayU, Cashfree explainers) | Ws |

**Repo documents** (read): `CLAUDE.md`; `docs/HANDOFF.md` section 2; `docs/PERF.md`; `docs/design/ux-analysis-2026-10.md`;
`docs/research/README.md`, `approvals-workflows.md`, `customer-vendor-portal.md`, `einvoice-gsp.md`.

