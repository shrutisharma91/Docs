# Mobile market research: what Indian owners and salespeople use on a phone today, where it fails, and what Comsky Books mobile should be best at

> **Status:** research and design input only. Nothing here is built. Written 2026-10-22. This file is the only thing the task touched in the repo.
> **Ask (owner, verbatim intent):** "research well, and develop great mobile apps, considering also business owners and sales people using the app, in mind."
> **Method:** about 70 web fetches and searches on 2026-10-21 and 2026-10-22 (official vendor pages, Apple App Store listings, Tally's help site, Odoo and Pipedrive documentation, WebKit/MDN/Chrome/Google Play platform pages, IDC/StatCounter/NPCI-reporting pages), plus a read of this repo and one read-only look at the running dev app at phone width. Everything is paraphrased; no vendor text or screenshot is reproduced.
> **Evidence tags:** **[W]** a page I fetched (the fetch tool summarises the page, so "not mentioned" means "not found on that page", never "does not exist"). **[Ws]** a search summary only, I did not read the page itself. **[C]** read in this repo (file, line where useful). **[J]** my judgement. **[unverified]** I looked and could not confirm.
> **Related repo docs:** `docs/design/ux-analysis-2026-10.md` section 4.12 and item N11 (the responsive shell), `docs/research/approvals-workflows.md` (phone approvals as a PWA), `docs/research/attachments-drives.md` (phone camera capture), `docs/research/customer-vendor-portal.md` (party side on a phone), `docs/ROADMAP.md` line 32 (the React Native "mobile epic", sized XL).

---

## 0. Read this first

### What I could not verify, and why it matters

1. **Google Play pages did not fetch.** Every attempt (Zoho Books, Vyapar, Biz Analyst, Marg eOrder) returned an empty shell. So Android ratings and download counts below are vendor claims, search summaries or tracker figures **[Ws]**, and they disagree with each other (Vyapar's Android rating is quoted between 4.5 and 4.8 while its India App Store rating is 3.5). Apple App Store pages did fetch **[W]**, and I use them for ratings, app size, languages and review excerpts. India is about 94% Android, so the iOS numbers describe the minority.
2. **No hands-on use of any competitor.** There were no live accounts. Offline behaviour, sync behaviour and role behaviour come from documentation, vendor guides, forum posts and reviews. Where a vendor page was silent on offline I say "not documented", which is weaker than "does not work offline".
3. **No Reddit and no video.** Two targeted searches for Reddit threads returned only vendor comparison pages. I did not watch YouTube. The "complaints" below come from App Store reviews I could read, Capterra and Software Advice aggregates seen through search summaries, vendor-run comparison pages (biased by construction) and forum posts. Treat every complaint as a lead for a user interview, not a measured rate.
4. **Prices conflict.** Indian software prices on aggregator sites differ from vendor pages and from each other. I give the vendor page where I could read it and a range where I could not.
5. **Statistics from vendors are marketing.** Field-sales vendors quote adoption, attrition and payroll-leakage numbers that I could not trace to a primary source; I cite them only as claims.
6. **The dev app was looked at in Chrome emulation only**, at a 375 px phone preset. No real device was used.

### Ten findings that change what we build

1. **Tally's own phone story is a window, not a tool.** The official options are Reports in Browser (about 37 reports, drill to voucher, PDF/Excel, view only, and the office PC running TallyPrime must be online and connected), a full TallyPrime session through paid Cloud Access, and the TallyIra iOS app that turns invoice photos and PDFs into draft vouchers **[W]**. I found no official order-booking, collection or approval app. A Tally switcher who lived with that is being offered a lot of room to improve. **[W]**
2. **The Indian phone market is two camps.** Ledger and billing apps (Khatabook, OkCredit, Vyapar, myBillBook) are free or cheap, offline-first, vernacular and WhatsApp-centred. ERP companions (Busy, Marg, Tally's third parties) are paid per device and tied to a desktop. The global tools (Zoho Books, Xero) are online-first and heavy (iOS listings 385 to 475 MB), QuickBooks stopped serving India in 2023, and Xero has no India edition. Nobody holds the middle: an accounting-grade app, in the hands of the owner and the salesperson, that works without the office PC. **[W/Ws]**
3. **Salesperson tools are bought separately and rarely touch the trader's books.** Bizom and FieldAssist serve consumer-goods brands and quote prices on request. Marg eOrder has the deepest link to an accounting package and draws sync and offline-failure complaints. Vyapar and Busy bolt a salesman role onto a billing app. A Tally-class distributor with five to twenty reps has no clean option. **[W/Ws]**
4. **Role leakage is a live complaint, and we would be worse today.** A Vyapar user reports that salesman-mode staff can still see purchase price, stock value and profit **[W]**. Comsky's operator rights are enforced on the client only (the file says so itself, `src/core/access.ts:12`), the cloud has one login per tenant, and a device pulls the whole company **[C]**. A salesperson's phone would carry every purchase rate and every ledger. This is the largest piece of real work in any mobile plan.
5. **Sync trust is the most repeated complaint class.** Biz Analyst App Store reviews from 2020, 2022 and 2025 report sync failures, one of them specifically when Tally is closed on the PC **[W]**; Marg eOrder reviews report offline-saved orders that cannot be sent **[Ws]**; one long-time Vyapar user reports missing records **[Ws]**; Intuit's community says QuickBooks mobile needs a connection and users report poor reconnect behaviour **[Ws]**. Comsky already has an outbox change-log, revision-based mismatch detection and a Sync Mismatch Report **[C]**; surfacing that in plain words on a phone is a differentiator, not a chore.
6. **The shell cannot be used on a phone today.** At about 410 px wide the three-column grid computes to `212px 11px 152px`, so the working screen is 11 px wide; with the rail hidden the Gateway draws in a column about 120 px wide beside a 152 px F-key rail. There is no manifest and no service worker. The single JS chunk is 2.74 MB (about 873 KB gzip) and the backend registers no compression. The page is served over plain HTTP. **[C]** (section 7).
7. **HTTPS gates everything mobile.** Service workers, install, web push, geolocation, Web Bluetooth and camera barcode detection are secure-context features **[W]**, and the VM serves HTTP only **[C]**. The earlier approvals research reached the same conclusion **[C]**.
8. **Android first is right; iOS needs a plan, not a promise.** Android is 93.9% of India's mobile web traffic in September 2026 **[W]**. On iPhone, a PWA in a Safari tab can lose its storage after seven days of Safari use without interaction, a Home Screen web app is tracked separately, and web push works only for Home Screen apps on iOS 16.4 or later **[W]**. Most owners and reps will be on Android; a minority of owners will be on iPhone and must be told to install.
9. **There are two different data modes, and the repo has only one.** The owner wants a thin, online-first, always-fresh read plus approvals (do not ship the whole books to a phone, the approvals research says the same **[C]**). The salesperson needs an offline-first, scoped replica of just their parties, prices and own documents. The sync protocol today is one mode: pull the whole tenant log **[C]**.
10. **Field orders and collections fit the books without new accounting rules.** A Sales Order is already a non-posting voucher (`accounting: false`, `src/types.ts:35`), pending orders already exist (`core/inventory.ts:315`), the credit-limit check with a recorded override reason is built (`core/creditLimit.ts`), and an `optional` voucher is excluded from every ledger total (`core/accounting.ts:9`). A rep's receipt can sit as optional until the office confirms the cash. Most of the "business" is built; the missing parts are the shell, the scoped server door and the language layer. **[C]**

### Decisions I need from the owner

| # | Decision | My recommendation | Why |
|---|---|---|---|
| D1 | Delivery shape: PWA, PWA plus a store wrapper, or React Native | PWA first, Capacitor (or a Trusted Web Activity) only when a named trigger is hit (section 8) | One codebase and the same Dexie/outbox; instant fixes; the engines are pure TypeScript either way **[J]** |
| D2 | TLS and an app hostname | Decide this week | Blocks install, push, location and durable storage for every mobile feature **[C]** |
| D3 | First persona to win | The salesperson of a trading or distribution business, with the owner as the buyer and approver | The owner's pulse is table stakes; the order-to-cash loop is where incumbents are weakest **[J]** |
| D4 | Pricing posture for field seats | Cheap or bundled field seats, no per-device lock | Per-device add-ons and licence errors are named complaints **[W]** |
| D5 | Language scope for v1 | English plus Hindi, with an i18n layer so others can follow | No i18n library exists today **[C]**; OkCredit ships 11 languages **[W]** |
| D6 | Pilot partners | Three distributors or traders with 5 to 15 reps, on the cheapest phones they own | The only way to settle the offline, battery and adoption questions **[J]** |

---

## 1. Who the mobile users are and what they are trying to get done

### 1.1 Jobs by persona

| Persona | Job on a phone | How often | What must be true |
|---|---|---|---|
| **Business owner** (proprietor, partner, finance head of a Tally-class firm) | See cash, sales today, who owes me, what is overdue | Several times a day | Numbers equal the books to the paisa, carry an "as at" time, and do not need the office PC |
| | Approve or refuse an exception (credit-limit override, large payment, discount, big purchase) | A few per day | Two taps from a notification; the reason is recorded |
| | Chase a payment: call or WhatsApp a party with the right bills listed | Daily | One tap; the message is correct and editable |
| | Check a party's statement, an invoice's status, stock of one item | Ad hoc | Fast search by name; share as PDF |
| **Field sales rep** (beat or visit-based) | Know today's visit list and each party's outstanding, credit headroom, last order | Every visit | Works with no signal; shows only what the rep may see |
| | Book an order (repeat the last one, adjust quantities) | Every visit | Under a minute; price list applied; stock visible; credit warning |
| | Collect money (cash, UPI, cheque) against named bills | Several per day | Bill-wise; receipt shared on WhatsApp; cash is accounted for later |
| | Log the visit, a note, the next follow-up | Every visit | One tap; no forced background tracking |
| **Van-sales rep** | Sell from stock on the vehicle and bill on the spot | All day | Invoice, print on a small Bluetooth printer, end-of-day stock and cash settlement |
| **Counter staff** | Bill walk-in customers, scan barcodes, take UPI | All day | Fast item search; works if the shop network drops |
| **Distributor's back office** | Turn the day's rep orders into dispatches and bills | End of day | Orders arrive as clean pending documents, not messages |

The first three rows are the v1 design target. Van-sales and counter billing are served well by Vyapar and myBillBook (section 2) and need hardware (printers, scanners) that a browser handles unevenly (section 8).

### 1.2 India context that shapes the design

| Fact | Figure | Source | What it means for us |
|---|---|---|---|
| Mobile OS share | Android 93.89%, iOS 6.05% of mobile web traffic, September 2026 | StatCounter **[W]** | Android is the product; iOS is a supported minority, probably owners |
| Phone market | Q2 2026 shipments 33.2 M, down 11.1%; average price a record US$315; sub-US$100 phones fell from 15.6% to 4.5% of shipments, the US$100 to 200 band is still the largest at 46.8%; 4G share rose to 11.1% as makers kept 4G models | IDC **[W]** | New phones are getting dearer, but the phones in reps' hands are older and cheaper than the sales mix suggests **[J]**; design for a 4 GB-class Android |
| Low-end memory | Launches of 4 GB variants in the sub-Rs 10,000 and Rs 10,000 to 20,000 bands rose sharply in Q1 2026 as memory costs squeezed makers | Counterpoint **[Ws]** | Do not assume 6 to 8 GB; budget memory and storage |
| Internet users | 886 M active in 2024, 55% rural; 98% access content in Indic languages and 57% prefer regional languages (two different measures) | IAMAI-Kantar via Business Today **[Ws]** | Language is a feature, not a polish item |
| WhatsApp | Estimates of Indian users range from about 535 M to 854 M depending on source; Meta and Bain count more than 65 M small businesses, about 5 M of which sell online | Search summaries **[Ws]** | WhatsApp is the document and reminder channel for every user in this research |
| WhatsApp Business API cost | List rates in India: about Rs 0.86 per marketing message and Rs 0.115 per utility message outside the 24-hour window, plus 18% GST and a BSP markup of 10 to 60% depending on the vendor; billing is per message since 1 July 2025 | Search summaries **[Ws]** | v1 should send with the person's own WhatsApp (click-to-chat), which costs nothing and is what Vyapar does **[Ws]** |
| UPI | 24.07 billion transactions worth Rs 29.37 lakh crore in September 2026, 802 M a day | NPCI data as reported **[W]** | UPI QR on an invoice and a UPI link in a reminder are expected, not optional |
| Connectivity | I could not verify a current median mobile speed. Flipkart's PWA case study (about 2016) is the standard proof that flaky networks and small phones reward an app-shell design | web.dev **[Ws]** | Design for the basement godown, the rural beat and the dead zone, not for the average Speedtest number |
| Data protection | DPDP Rules notified 13 Nov 2025; consent-manager framework from about 13 Nov 2026; core duties from about 13 May 2027; an employment "legitimate use" exists but monitoring beyond clearly work-related purposes may need consent | Law-firm commentary **[Ws]** | Location and attendance features should be opt-in per use, with a plain notice, in v1 |
| Android location policy | Background location is allowed only when core to the app's main purpose, needs a Play Console declaration and a video, and may be refused; foreground use is the preferred route | Google Play policy **[W]** | A background "breadcrumb trail" of salespeople is a store-review risk; check-in on tap is not |

---

## 2. The competitive landscape

I read each product's public pages and store listings. "Missing for a Tally switcher" is judged against what a TallyPrime user does all day: bill-wise outstanding, voucher types and series, GST invoice with place of supply, batch and godown stock, TDS, and the speed of the keyboard.

### 2.1 Platforms, personas and offline at a glance

| Product | Platforms | Owner / pulse | Salesperson / field | Offline |
|---|---|---|---|---|
| **TallyPrime (official)** | Phone browser (Reports in Browser); full app through paid Cloud Access; TallyIra (iOS listing found, Android not checked) **[W]** | About 37 reports, drill to voucher, change period, PDF/Excel; remote approval via Optional Vouchers is a desktop pattern the owner uses from anywhere **[W/Ws]** | None official. TallyIra captures invoices to drafts (back office) **[W]** | **No.** Reports are pulled live from the office PC, which must be online **[W]** |
| **Biz Analyst** (Tally companion, owned by Khatabook per a search summary) | Android, iOS **[W]** | Live reports, outstanding reminders, dashboards, seven security levels **[W]** | Location check-in and check-out, GST invoice creation, entry of transactions and receipts **[W]** | Cached view only; needs a connector on the PC **[Ws]** |
| **Vyapar** | Android, iOS, Windows, Mac **[W]** | 50+ reports, party reminders **[W]** | Salesman role; a "salesman tracking" page claims GPS check-in, visit log, orders, collections, offline check-ins and notes **[W, vendor]** | Strong on-device billing offline (the reviewers' main praise); sync documentation thin **[W]** |
| **myBillBook** | Android, iOS, web, desktop **[W]** | Dashboard cards, GSTR-1/3B, P&L **[W]** | Multi-user; staff attendance with check-in/out times and payroll; no field-sales module found **[W]** | Not stated **[W]** |
| **Khatabook** | Android, iOS; desktop announced **[Ws]** | Ledger totals, reports **[Ws]** | None | Basic ledger works offline **[Ws]** |
| **OkCredit** | Android, iOS **[W]** | Ledger, statements, collection reports **[W]** | None | Offline use and cloud backup **[W]** |
| **Busy** | Android, iOS companion to Busy desktop **[W]** | 100+ reports, daily-summary push, warning alerts, approve or reject vouchers **[W]** | Orders, quotations, invoices, receipts, barcode billing, salesman tagging, filter by broker **[W]** | Not stated; "auto cloud sync" **[W]** |
| **Marg ERP** | eOwner, eOrder, eRetail, SFAXpert apps **[Ws]** | eOwner: collections, outstanding and PDC, salesman location, credit caps **[Ws]** | eOrder: route tracking, cart editable offline or online, bill-by-bill collection, outstanding and PDC, live stock **[W]** | Claimed for carts; reviewers report failures **[Ws]** |
| **Zoho Books (and Zoho Invoice)** | Android, iOS, Apple Watch, widgets **[W]** | 50+ reports, push for payments and approvals, bank feeds, Siri approve **[W]** | Estimates, invoices, expenses, receipt scan, mileage; no field-sales tools **[W]** | **Not documented**; Zoho staff told a Zoho Invoice user the app is cloud-only **[Ws]** |
| **Zoho CRM / Bigin** | Android, iOS **[W]** | Quick view of team performance **[W]** | CRM: check-in at a client, nearby customers with navigation, voice notes on call logs; Bigin: pipelines, in-app calls, WhatsApp channel **[W]** | CRM: offline claimed by the vendor, users mixed **[W/Ws]**; Bigin: not stated |
| **QuickBooks** | Not available to new Indian customers since 2023 **[W/Ws]** | (global) invoices, reports, receipt capture, GPS mileage | n/a | Online only per Intuit's community **[Ws]** |
| **Xero** | Android, iOS **[W]** | Invoices, quotes, cash flow, reconcile, AI assistant **[W]** | n/a | Not confirmed **[Ws]** |
| **Odoo** | A thin native wrapper app (2 MB on iOS) for Online and Enterprise databases **[W]** | All apps through the wrapper | CRM, Inventory through the same wrapper | Short interruptions only; 2 GB cap; hard refresh clears it **[W]** |
| **Pipedrive** | Android, iOS **[W]** | Pipeline, activities | Deals, activities, calls, notes | **Yes** for moving deals, notes and activities, synced later **[W]** |
| **Field-sales suites** (Bizom, FieldAssist, SalesJump) | Android SFA apps, DMS apps, retailer apps **[W/Ws]** | Dashboards for brand managers | Attendance, beat plans, order booking, collections, van sales, merchandising by image **[W]** | Claimed (Bizom: works in low or no connectivity; FieldAssist: SKU list cached on the device) **[W/Ws]**; SalesJump: not found |

### 2.2 Pricing, ratings and scale

| Product | Pricing model (INR, excluding 18% GST unless said) | Rating and scale | iOS size and languages |
|---|---|---|---|
| **TallyPrime** | Silver (single user) Rs 22,500 lifetime or Rs 750 a month; Gold (multi-user) Rs 67,500 lifetime or Rs 2,250 a month; WhatsApp sharing and Cloud Access sold separately (a Silver add-on example is Rs 1,800) **[W]**. A vendor blog quotes TSS at Rs 4,500 (Silver) and Rs 13,500 (Gold) a year; Tally's page says Gold includes TSS **[Ws]** | TallyIra iOS 4.9 from 14 ratings **[W]** | TallyIra 61 MB; English documents only; handwritten not supported per Tally's help page, though a Tally marketing article says it reads handwriting (they conflict) **[W]** |
| **Biz Analyst** | Free to install; a vendor blog puts third-party Tally companions at about Rs 2,200 to 3,600 per device per year **[Ws]** | iOS 4.6 from 16 ratings **[W]**; company claims of 50,000 to 100,000 users **[Ws]** | 158 MB |
| **Vyapar** | Free mobile tier; mobile plans about Rs 699 to 999 a year, desktop about Rs 3,400 to 3,700, desktop plus mobile about Rs 4,000 to 4,300 (aggregators disagree); India iOS in-app prices Rs 999 (Gold Mobile) up to Rs 12,700 (Platinum Combo); multi-device sync needs a licence on every device **[Ws/W]** | Vendor: 10 M+ downloads, 4.7 to 4.8 on Play, 1.5 crore businesses; tracker: 4.77 from about 180,000 ratings; India iOS **3.5 from about 1,100 ratings**; Capterra 4.3 from 181 **[W/Ws]** | 247 MB; English and Hindi |
| **myBillBook** | Free plan capped at 15 invoices a month; a banner says plans start at Rs 399 a year; its own comparison shows Rs 2,599 a year for the cross-platform plan **[W/Ws]** | Vendor: Play 4.7, "1 crore+ businesses"; founder interview: 5 M+ downloads; US iOS 4.1 from 14; Capterra 4.5 from 52 **[W/Ws]** | 125 MB (US store); English |
| **Khatabook** | Free basic ledger; premium price not published **[Ws]** | Listing claims 5 crore users; about 10 M+ downloads; 4.5 to 4.8 **[Ws]** | Not fetched; about 13 languages per G2 **[Ws]** |
| **OkCredit** | Free basic; a paid Premium (GST billing, multi-device) **[W]** | Site: 1 crore+ downloads, 4.6; iOS 4.8 from 36 **[W]** | 103 MB; English plus 10 Indian languages **[W]** |
| **Busy** | Mobile Rs 3,600 per device for 1 to 4 devices, Rs 2,700 each for 5 or more; term not stated; a Busy desktop subscription is also needed **[W]**. Desktop: Rs 4,999 to 9,999 (single user, subscription) or Rs 9,999 to 14,999 perpetual **[Ws]** | iOS **3.3 from 7 ratings** **[W]** | 38.6 MB |
| **Marg** | ERP from about Rs 9,999 per an aggregator; app prices not published **[Ws]** | eOrder on Play at least 100,000 installs; Marg's 19 Android apps average 3.79 from about 6,600 ratings (tracker); iOS eOrder: too few ratings to show **[Ws/W]** | eOrder 12.7 MB; English |
| **Zoho Books** | Free (1 user plus an accountant); Rs 749 (3 users), 1,499 (5 users, custom roles), 2,999 (10), 4,999 (15), 7,999 (25) per organisation per month billed yearly; extra user Rs 150 a month **[W]** | India iOS **4.8 from about 6,000**; Capterra India 4.4 from 674 **[W/Ws]** | **475 MB**; 27 languages including Hindi, Gujarati, Kannada, Marathi, Tamil, Telugu **[W]** |
| **Zoho CRM / Bigin** | CRM about Rs 800 per user per month at Standard (press); Bigin Rs 400, 720 or 1,080 per user per month, billed yearly **[Ws]** | CRM iOS 4.3 from 351 (UK store), tracker 4.4 from 3,800; Bigin iOS 4.7 from 969 **[W/Ws]** | CRM 581 MB, Bigin 371 MB; 26 to 28 languages |
| **Xero** | Not sold with an India edition **[Ws]** | iOS 4.6 from about 1,700 **[W]** | 386 MB; English only |
| **Odoo** | Mobile app only for Online and Enterprise; India list about Rs 580 to 725 (Standard) and Rs 890 to 1,150 (Custom) per user per month **[W/Ws]** | iOS **2.6 from 146** **[W]** | 2 MB; English |
| **Pipedrive** | US dollars, about $12.5 to 14.9 (Essential) up to about $50 (Professional) per user per month; no free tier **[Ws]** | Play about 4.0 from 3,600; iOS about 3.8 to 3.9 **[Ws]** | n/a |
| **Bizom / FieldAssist / SalesJump** | Quote only for Bizom and FieldAssist; SalesJump listed near Rs 300 per user per month by one aggregator **[Ws]** | FieldAssist claims 190,000 users, 75,000 distributors, 8.9 M outlets; Bizom claims 750+ brands **[W, vendor]** | FieldAssist site: English and Indonesian |

**What six phones cost a year, for a feel (an owner plus five reps, excluding GST):** Busy mobile about Rs 16,200 (six at Rs 2,700) on top of the desktop subscription **[W]**; Zoho Books Professional about Rs 19,800 (Rs 1,499 plus one extra user, times twelve) for the whole product with roles **[W]**; Vyapar mobile about Rs 4,000 to 6,000 if every device needs its own licence **[Ws/W]**; a Tally Gold licence is Rs 67,500 once, with phone access a separate purchase **[W]**. The spread shows the pricing wedge: per-phone add-ons cost real money, and a vendor blog claims field-team attrition of 30 to 40% a year **[Ws, vendor]**, so seat friction recurs.

### 2.3 How each handles the India specifics

| Product | GST invoice | WhatsApp | UPI | Party ledger and outstanding reminders | Multi-user and salesperson roles | Missing for a Tally switcher |
|---|---|---|---|---|---|---|
| **TallyPrime** | Full | Outbound sharing sold separately; a third-party vendor markets a bot that answers owner questions from live data **[W/Ws]** | Not checked | Full bill-wise; phone view only | Tally.NET users, security levels, browser access per user **[W]** | Everything that is not viewing |
| **Biz Analyst** | GST invoice with logo **[W]** | Automatic reminders by WhatsApp, SMS, email **[W]** | Online collection by UPI, cards, wallets **[W]** | Outstanding reminders, inactive-customer tracking **[W]** | Seven security levels **[W]** | Reliable sync; full entry; depends on Tally |
| **Vyapar** | GST and VAT, 50+ reports **[W]** | Reminder from the person's own WhatsApp or SMS, recurring on a schedule; one reviewer says invoices arrive as blank files **[Ws]** | Not found | Party ledger, reminders, credit limits **[W]** | Secondary Admin, Salesman, Viewer, Biller, Accountant, Stock Keeper **[Ws]**; a reviewer says salesman mode can see purchase price, stock value and profit **[W]** | Journals and bank reconciliation are shallow, multi-branch is weak **[W, competitor review]** |
| **myBillBook** | GST invoice, e-invoice, e-way bill, GSTR-1/3B **[W]** | WhatsApp reminders with UPI links, live shareable ledger **[W]** | UPI QR on invoices **[W]** | Outstandings, credit period and limit **[W]** | Multi-user, attendance **[W]** | Depth of accounting; Tally-style vouchers |
| **Khatabook** | GST and non-GST bills (promotional text) **[Ws]** | Payment links **[Ws]** | Payment links **[Ws]** | The ledger is the product | None found | Everything accounting-grade |
| **OkCredit** | GST billing on Premium **[W]** | Automatic SMS to the customer, WhatsApp reminder with a payment link **[W]** | UPI QR, payments recorded automatically (vendor claim) **[W]** | Ledger, PDF statements **[W]** | Multi-device on Premium **[W]** | Everything accounting-grade |
| **Busy** | Yes (the desktop product) | Share quotations, invoices, ledgers on WhatsApp **[W]** | Not stated | 100+ reports with outstanding **[W]** | Salesman tag, broker filter; mobile needs a paid subscription (a login error says so) **[W]** | A mobile that works without its own subscription; offline unknown |
| **Marg** | Yes (the desktop product) | Order sharing to retailers reported failing in one review **[Ws]** | Not stated | Bill-by-bill collection, outstanding, PDC **[W]** | Per-salesman eOrder; eOwner for the owner **[Ws]** | Reliable offline and sync; English only |
| **Zoho Books** | India edition with e-invoice from the Standard plan **[W]** | Not a WhatsApp-first product **[unverified]** | Online payment gateways **[Ws]** | Customer statements; reminders | Custom roles from the Rs 1,499 plan **[W]** | Voucher speed, bill-wise depth, offline |
| **Zoho CRM / Bigin** | n/a (CRM) | Bigin lists WhatsApp as a channel **[W]** | n/a | n/a | Per-user pricing, roles | They do not hold the books |
| **Xero / QuickBooks** | No India edition / gone | n/a | n/a | n/a | n/a | Not options in India |
| **Odoo** | India localisation exists **[unverified here]** | n/a | n/a | n/a | Enterprise per user | Cost and weight; wrapper app is weak |
| **Pipedrive** | n/a | n/a | n/a | n/a | Per-user | They do not hold the books |
| **Field-sales suites** | Not accounting | Bizom lists a WhatsApp ordering bot **[W]** | n/a | Credit and outstanding views (vendor blogs) **[Ws]** | Built for many reps | They do not hold the trader's books; quote-only; English-first **[W]** |

### 2.4 Product notes: praise, complaints, gaps

**TallyPrime on a phone.**
- Reports in Browser is the official view tool: a Tally.NET ID, the company connected from TallyPrime, the PC online, companies shown as Connected or Offline, about 37 reports, owner and auditor see all, other users a smaller set, custom levels can block reports **[W]**. Excel download works on mobile without Excel on the phone **[W]**. Its help page does not mention a Tally Software Services subscription; Tally's marketing article says one is needed **[W/Ws]**.
- Cloud Access runs the whole application on a remote server and lets a person record entries; it is not in the plan price **[W]**. A vendor article calls it cumbersome beyond quick checks and says the server must be online **[Ws, vendor]**.
- TallyIra is the first official app. It reads invoices from the camera, a share from WhatsApp or Gmail, or a file; extracts party, number, date, amounts, GST and lines; maps to existing masters; flags duplicates; creates draft vouchers; added bank-statement upload in July **[W]**. Limits per Tally's help page: one transaction per file, up to 5 MB through the app, nine voucher types, English documents, usage metered in page packs valid a year **[W]**. Reviews are few (14) and kind **[W]**. A third-party guide says access is being rolled out in phases to eligible licences **[Ws]**.
- Praise is thin (the app is new). The complaint is structural: nothing here lets the owner or a rep act on the books from a phone.

**Biz Analyst.** An independent companion that syncs with Tally and shows reports, outstanding, sales analysis, with reminders, invoices, entry, a sales-team check-in and online collection **[W]**. Listed as free to install on iOS at 4.6 from 16 ratings; the visible reviews are mostly about sync: one (2022) says data does not sync when Tally is closed on the PC, one (2020) says sync fails even when Tally is synced, and a 2025 review calls sync very poor **[W]**. A Capterra reviewer in 2023 liked the real-time mobile access and flagged delays, fewer features than the desktop and limited dashboard customisation **[Ws]**. Its dependency on a connector and an open PC is the Tally problem again.

**Vyapar.** The mass-market leader for Android billing. Praise: speed of GST invoicing, usable by non-accountants, offline billing, reminders, barcode, WhatsApp share **[W, competitor review; Ws]**. Complaints: a five-year user reporting records going missing and earlier entries no longer matching, thin support in several reviews, hangs during business hours, WhatsApp invoices arriving blank **[Ws]**; an iOS reviewer says a secondary admin is pushed to a subscription page and that salesman-mode staff see purchase price, stock value and profit, which cost them staff, and the developer's reply says the app is device-based and multi-device use needs a licence **[W]**. Its own role page says staff cannot see purchase price and cannot delete or edit transactions or take backups **[W]**; the two statements contradict, which is itself the finding: the role model is shallow or inconsistent. Sync is an invite-and-download model with an admin device **[W]**. A competitor's review rates it 3.8 out of 5 and says it fits single-location shops and is outgrown **[W, bias noted]**.

**myBillBook.** Broad billing app with GST, e-invoice, e-way bill, WhatsApp reminders with UPI links, UPI QR on invoices, staff attendance and payroll, bill scanning, and a recent dashboard-card update **[W]**. It sells on a 14-day trial after which a subscription is needed (iOS) **[W]**. Reviews I could read were too few to characterise; a competitor claims it syncs in real time and a rival claims its accounting is lighter **[Ws, both biased]**. No field-sales module was found.

**Khatabook and OkCredit.** These own the "udhar" habit: record credit given and received, send reminders, share statements, in many Indian languages, offline, free to start. OkCredit's site describes SMS to the customer the moment an entry is made, WhatsApp reminders with a payment link, UPI QR with automatic recording, backups so a lost phone loses nothing, and GST billing on Premium **[W]**. Praise (OkCredit iOS): it helps people remember dues; a long-time user dislikes the move to a subscription **[W]**. Khatabook's privacy policy copies I found (dated April 2020) say contact lists are uploaded to its servers and installed apps may be inspected **[Ws]**; it laid off about 6% of staff in 2023 and closed a commerce product in 2021 **[Ws]**; OkCredit reportedly laid off 30 to 35% of staff in 2022 and, in the reports I found, has raised no round since 2019 **[Ws]**. Lesson: the free ledger habit is real and the monetisation behind it is fragile; neither covers invoices, stock or a salesperson.

**Busy.** A mobile companion that needs a paid mobile subscription alongside the desktop, so a user without one hits an error saying no active subscription was found **[W]**. Features include approving or rejecting vouchers, daily summary and warning alerts by push, salesman tagging on vouchers, 100+ reports, orders and quotations; India iOS rating is 3.3 from 7 **[W]**. A directory places salesman reporting in a higher desktop tier **[Ws]**. Gap: pay-per-device mobile and an unclear offline story.

**Marg ERP.** The most complete salesperson apps among accounting packages: eOrder for field reps, eOwner for the owner, eRetail for the retailer to order from a supplier **[W/Ws]**. eOrder's listing promises a cart editable offline, route tracking, bill-by-bill collection, outstanding and PDC tracking, live stock and multi-company orders **[W]**. Play reviews quoted in a search summary describe orders saved offline that could not be retrieved or sent, crashes, stock and transaction mismatches against the server, poor sync and orders that cannot be shared with retailers; the developer replied to one that it was fixed **[Ws]**. Pharma and FMCG distributors are its core. Gap: reliability, language, and a modern interface.

**Zoho Books and Zoho Invoice.** Polished and broad, with 27 languages and AI reports, Siri approvals, widgets, and bank feeds **[W]**. The India iOS listing is 4.8 from about 6,000; reviewers praise a clean, easy app and responsive support, one complains of unreliable bank feeds and a GSTR-3B view missing the company name **[W]**. Capterra reviewers call the app less capable than the web and say it is cloud-only **[Ws]**. The app is 475 MB on iOS **[W]**. Roles beyond the basics need the Rs 1,499 plan **[W]**. Gap for a switcher: voucher-style speed, bill-wise depth, offline and a salesperson role.

**Zoho CRM and Bigin.** CRM's iOS app does check-in at a client, nearby customers with navigation, voice notes on call logs and claims offline access **[W]**; visible reviews are dated 2017 to 2021 and mention analytics lagging the web, fixed record layouts and weak telephony **[W]**. Forum posts say the app must reconnect to pull fresh data and that calls and emails cannot be made offline **[Ws]**. Bigin is the small-team version (pipelines, in-app calls, WhatsApp as a channel) at 4.7 from 969 **[W]**. There is no standalone "Zoho Sales" product that I found **[unverified]**.

**QuickBooks and Xero.** Intuit told Indian customers in July 2022 that QuickBooks would leave the country; sign-ups stopped, and a search summary of Intuit's own post gives 30 April 2023 as the end of service **[W/Ws]**. Observers speculate that cheaper local and offline competitors played a part; Intuit gave no reason **[Ws]**. Xero's iOS app is 4.6 from about 1,700; reviewers say it lacks web functionality and a custom date range for profit and loss **[W]**; it has no India edition and filings run outside the software **[Ws, sold by CA firms with an interest]**. For this market they are context, not competition.

**Odoo.** The mobile app is a 2 MB wrapper around the web client for Online and Enterprise databases, rated 2.6 from 146 on iOS; reviewers list missing basics such as adding notes to orders and a barcode scanner that does not scan **[W]**. Odoo's documentation describes offline mode as a cushion for short interruptions, limited to 2 GB with the oldest data dropped first, wiped by a hard refresh, and (in the 19.0 text I read) read-only for records opened earlier; later versions per a search summary allow edits with conditions **[W/Ws]**. It is the cautionary tale for "just wrap the web app": the wrapper inherits every weakness of the web client.

**Pipedrive.** The best-documented offline story among the CRMs: the mobile apps let a rep move deals, add notes and activities offline and sync on reconnect, with no manual mode switch; the web app needs a connection **[W]**. Call logging on iOS only captures calls placed from inside the app, Android logs more **[Ws]**. The mobile complaints are about depth versus the web and slow lists on large accounts, from a development agency's blog **[Ws]**; store ratings sit near 4 **[Ws]**.

**Field-sales suites.** Bizom: sales force automation, distributor management, direct retailer ordering, beat optimisation, van sales, image-based merchandising, a WhatsApp ordering bot, AI coaching, 750+ brands claimed, low or no connectivity claimed **[W]**. FieldAssist: SFA, DMS, retailer app, van sales, geo-verified adherence, 190,000 users claimed **[W]**; a profile says the app works offline and caches the SKU list **[Ws]**. SalesJump: Chennai, FMCG and dairy, SFA plus DMS plus retailer app **[Ws]**. All three sell to brands that run thousands of reps; none holds a trader's GST books, prices are on request, and the sites are English-first **[W]**. Vendor blogs describe the real workflow gaps: paper and WhatsApp orders re-keyed by the office, reps blind to outstanding and credit at booking, English-only apps, offline needs, and distributors who run Tally or nothing **[Ws, vendor]**.

---

## 3. What users praise and complain about, across the field

| Theme | Evidence | Reading |
|---|---|---|
| **The office PC must be on** | Tally Reports in Browser needs the PC online **[W]**; Biz Analyst needs a connector **[Ws]**; Busy mobile needs a desktop subscription **[W]**; Vyapar sync needs an admin device **[W]** | A cloud-held book is a structural advantage over all of them; we have it **[C]** |
| **Sync trust** | Biz Analyst reviews from 2020, 2022 and 2025 **[W]**; Marg eOrder offline orders **[Ws]**; Vyapar missing records **[Ws]**; QuickBooks reconnect **[Ws]** | The first bad experience is a lost order. Visible status and no silent loss is the feature |
| **The phone is a lesser web** | Xero, Zoho Books, Pipedrive, QuickBooks reviews and notes **[W/Ws]** | Do fewer things, completely, rather than a thin version of everything |
| **Heavy apps** | iOS listings 247 to 581 MB for Vyapar, Zoho, Xero, Bigin **[W]** vs 13 to 61 MB for Marg, Busy, TallyIra **[W]** | On a phone with little free storage the large apps are the ones that get uninstalled **[J]** |
| **Subscription and per-device friction** | OkCredit reviewer on the subscription move; Busy login error; Vyapar secondary admin pushed to a paywall **[W]** | Never block a working rep behind a payment screen mid-day |
| **Support** | Vyapar reviews (mixed), a Zoho Books app review **[Ws/W]** | Support by WhatsApp in the user's language is expected |
| **Praise: simple, fast, cheap** | Vyapar, OkCredit, Zoho Books reviews **[W/Ws]** | The winners are measured in seconds per bill and rupees per month |
| **Praise: reminders and WhatsApp** | OkCredit, myBillBook, Vyapar **[W/Ws]** | Outstanding to a WhatsApp message in one tap is the habit-forming loop |
| **Praise: vernacular** | OkCredit 11 languages **[W]**; Zoho Books 27 **[W]** | Language is a reason to switch |
| **Role leakage** | Vyapar iOS review **[W]** | A salesperson must not be able to read cost |

---

## 4. The five jobs where incumbents are weakest

**Job 1. Book an order in the field, offline, with the party's money position in front of me.**
- *Why they fail:* the global tools are online-first or undocumented offline **[Ws]**; Tally has no field order tool **[W]**; Busy and Marg are paid per device and tied to the desktop and Marg's offline orders draw failure reports **[W/Ws]**; Vyapar's offline billing is good but its sync and its role limits are weak **[W]**; the brand-side suites do not hold a trader's books **[W]**.
- *What great looks like:* an order for a known party in under a minute (the folk rule in vendor material is that a routine order taking more than a minute or two loses to a phone call **[Ws, vendor]**), repeat-last-order, the party's price level applied, stock quantity shown without cost, credit headroom shown as a warning with a recorded reason rather than a wall, and the order arriving in the office as a normal pending Sales Order with the rep's name.
- *Comsky advantage today:* Sales Order is a non-posting voucher **[C types.ts:35]**; pending orders exist **[C inventory.ts:315]**; the credit-limit check with an override reason is built **[C creditLimit.ts]**; price levels and ghost recall exist **[C]**; server-assigned voucher numbers can be claimed ahead for offline use **[C sync.js:319]**.
- *Missing:* the scoped replica, the phone UI, item search tuned to a thumb, a language layer.

**Job 2. Turn outstanding into cash during the visit or the call.**
- *Why they fail:* Khatabook and OkCredit own reminders but not GST bills, stock or reps **[W/Ws]**; Vyapar and myBillBook reminders are party-level and the WhatsApp send is manual or needs the paid API **[Ws]**; Tally's WhatsApp is a paid extra **[W]**; Marg has bill-by-bill collection but unreliable sync **[W/Ws]**.
- *What great looks like:* a party card with bills by age, one tap to call or WhatsApp with the right bills in the message, a promise-to-pay date, and "collect" as a receipt against named bills by cash, UPI QR or cheque. A rep's receipt stays pending until the office confirms the cash; the owner sees collections by rep as they happen.
- *Comsky advantage today:* bill-wise engine and ageing **[C billwise.ts]**, the outstanding message builder and WhatsApp/e-mail links **[C share.ts]**, UPI link and QR from the owner's own VPA **[C upi.ts]**, PDC register, credit-limit report, optional vouchers excluded from totals **[C accounting.ts:9]**.
- *Missing:* promise-to-pay and a follow-up list, collection by rep, the confirm-deposit step, a way to confirm a UPI payment without a payment-service integration (a human marks it received and the bank-statement import matches it later **[J]**).

**Job 3. See the pulse and decide, without the office PC.**
- *Why they fail:* Tally's phone view is read-only and depends on the PC **[W]**; Biz Analyst sync complaints **[W]**; Busy has summaries and approvals but needs its desktop subscription **[W]**; Zoho's push and widgets are generic and its mobile approval flow is not documented in the pages I read **[W/C]**.
- *What great looks like:* five or six cards (sales, collections, receivables overdue, payables due, cash and bank, stock alerts), each stamped with "as at" and the time of the last sync, drill to the party or voucher, push only for exceptions, approve or refuse in two taps with a reason.
- *Comsky advantage today:* Dashboard v2 engines whose figures are reconciled to the paisa against the reports they open **[C CLAUDE.md Phase 28]**, a cloud store that does not depend on a desk PC **[C]**, and a written approvals design **[C approvals-workflows.md]**.
- *Missing:* a server-side summary endpoint (so the phone does not need the whole company), push infrastructure (none exists **[C]**), TLS, the phone layout.

**Job 4. Give a salesperson a safe slice of the books, and give the owner a trustworthy visit record without surveillance.**
- *Why they fail:* Vyapar's salesman sees cost **[W]**; Tally's access levels govern viewing **[W]**; Zoho's roles need a mid-priced plan **[W]**; vendor blogs report GPS tracking drains batteries (a tracking vendor measured 13% with good signal and 38% with weak signal), draws resistance when rolled out without explanation, and is easy to spoof on Android **[Ws, vendors]**; Google Play restricts background location and DPDP raises the bar for monitoring **[W/Ws]**.
- *What great looks like:* a server-enforced slice (the rep's parties, sale prices and stock quantities, own documents; no purchase cost, no profit, no other users' data), revocable from the owner's phone in seconds, a check-in on tap that records place and time for that visit only, visit notes and next follow-up, and attendance as check-in and check-out events.
- *Comsky today:* operator rights are a client-side control **[C access.ts:12]**; one login per tenant; no invite flow (the only `INSERT INTO membership` is the register path, `backend/src/tenancy.js:10`); a whole-company pull **[C sync.js:343]**; API keys are scoped only to read or write for the whole tenant **[C platform.js:81]**. This is the largest build item and, if done well, the clearest differentiator.

**Job 5. Trust the sync, in my language, on my cheap phone.**
- *Why they fail:* the complaint list in section 3; apps of 400+ MB; English-only Marg, Busy, Tally and FieldAssist **[W]**; Odoo's wiped-on-refresh cache **[W]**.
- *What great looks like:* every record shows one of four plain states (saved on this phone, sent, confirmed by the office, needs your attention), automatic retry whenever the app is open, a visible queue count, a one-tap "send the order as a WhatsApp message" fallback if the cloud is unreachable for a day **[J]**, conflicts shown only where money or quantity differs and in words a rep understands, and Hindi labels.
- *Comsky advantage today:* an outbox change-log, revision-based mismatch detection and a Sync Mismatch Report **[C CLAUDE.md Phase 24]**, a durable-storage request with a visible warning **[C persist.ts]**.
- *Missing:* an i18n layer (no library in `package.json`, `src/state` has no locale **[C]**), compression and code-splitting, a phone conflict screen, plain-language statuses.

---

## 5. What "great" looks like in this market

These are design rules, each tied to the evidence above. The numeric budgets are **[J]** and must be measured on the cheapest handset in the pilot before they are promised.

1. **Two apps in one installable shell, chosen by role at sign-in.** An Owner view and a Field view, not a shrunken desktop. The desktop stays keyboard-first and untouched.
2. **Thumb-first.** A bottom bar of four, one primary action per screen, large touch targets, no F-keys, no hover, no right rail. The keyboard rail does not appear on a phone.
3. **The party card is the salesperson's home.** Outstanding by bill, credit headroom, last order, last receipt, phone and WhatsApp on one screen.
4. **Offline is the normal state, not a mode.** A whole shift (8 hours, 100 documents) with no signal; the queue is visible; nothing is dropped silently (Pipedrive's no-toggle offline behaviour is the model **[W]**).
5. **Honest status on every record and every number.** "As at" on balances (the Dashboard v2 rule **[C]**), a sync state on every document, staleness shown in words.
6. **Same numbers as the books.** The pure engines in `src/core` are shared, so the phone and the desktop agree to the paisa (the Phase 28 rule **[C]**).
7. **WhatsApp is the output channel, with the person's own number.** Prefilled click-to-chat messages and the share sheet for PDFs. No paid API in v1. This matches the Indian norm and costs nothing **[C share.ts, Ws]**.
8. **UPI from the owner's own VPA.** Show an amount QR; a human marks it received; the bank-statement import matches it later **[J]**. iOS cannot learn which UPI app opened or the result, and Android returns a status that should still be verified on the server, per the link specification summaries **[Ws]**.
9. **Language.** English and Hindi in v1 behind an i18n layer; Indian number grouping (lakh and crore) already exists in `core/money.ts` **[C]**; terms translated by a person, not a machine, because an accounting word translated badly is a wrong figure **[J]**.
10. **Light and fast.** Budgets to hold: field shell under about 300 KB gzip before first useful paint (the whole app is about 873 KB gzip today **[C]**); cold start to "new order" in about 3 seconds on a 4 GB Android over 4G; no background GPS in v1; local data kept small by design (parties, items, own documents only).
11. **Safe by construction.** A server-scoped token, no cost data on a rep's device, revoke from the owner's phone, a plain notice for anything location-related, no contact-list upload (the Khatabook policy is the example to avoid **[Ws]**).
12. **A price that does not punish a rep.** Field seats cheap or bundled, no per-device lock, no payment screen in the middle of a visit.
13. **A way back.** Export or WhatsApp-share any unsent order; owner can see every device and sign one out (a lost phone is the common case **[J]**).

---

## 6. What to copy and what to avoid

| Copy | From | Evidence | How |
|---|---|---|---|
| Offline with no mode switch; sync on reconnect | Pipedrive | **[W]** | The Field shell works from local data and flushes the outbox on open and on network return |
| Bill-by-bill collection, outstanding and PDC on the rep's screen | Marg eOrder | **[W]** | Receipt against named bills; PDC visible in the party card |
| Daily summary and warning alerts by push; approve or reject on the phone | Busy | **[W]** | Exceptions-only push; two-tap approvals |
| Drill to the voucher; PDF/Excel from a phone | Tally Reports in Browser | **[W]** | Owner cards drill to party and voucher; share as PDF |
| Maker-checker through a pending state | Tally Optional Vouchers (as the owner uses it) | **[Ws]** | A rep's receipt and big orders are optional until confirmed **[C]** |
| Share a bill from WhatsApp into the app and get a draft | TallyIra | **[W]** | Later: receive a shared file into a draft purchase (needs the Android share target and so a wrapper) |
| One-tap reminder with a UPI link; shareable live ledger | OkCredit, myBillBook | **[W]** | Outstanding message with a UPI link and a statement PDF |
| Many Indian languages, a free start | OkCredit, Khatabook | **[W/Ws]** | Hindi first, i18n for the rest; a free solo tier is a business decision |
| Repeat-last-order, one-tap reorder | Vendor field-sales material | **[Ws, vendor]** | Ghost recall already shapes a party's last voucher **[C]** |
| Staff attendance as check-in and check-out | myBillBook | **[W]** | Visit check-in doubles as attendance, on tap |

| Avoid | From | Evidence | Why |
|---|---|---|---|
| A phone app that needs the office PC online | Tally Reports in Browser, Biz Analyst | **[W]** | Our cloud makes this unnecessary |
| Per-device mobile add-ons and licence errors | Busy, Vyapar | **[W]** | Named complaints; seats churn |
| Salesman mode that exposes cost and profit | Vyapar | **[W]** | Trust and staff-poaching risk |
| 400 to 600 MB apps | Zoho Books, Zoho CRM, Xero | **[W]** | Storage-poor phones |
| A web wrapper with no offline design | Odoo mobile | **[W]** | Wiped caches, missing basics |
| Uploading the contact list or reading installed apps | Khatabook policy copy | **[Ws]** | DPDP exposure and a trust cost |
| Always-on location trails by default | Field-sales suites | **[Ws, vendors]** | Battery, resistance, Play review, consent |
| English-only | Marg, Busy, Tally, FieldAssist | **[W]** | Adoption "suffers silently" per vendor blogs **[Ws]** |
| Mid-day paywalls | OkCredit, Busy, Vyapar reviews | **[W]** | A rep blocked at a customer's counter does not come back |

---

## 7. Where Comsky stands today: a phone-readiness audit (read in this repo)

| Area | State | Evidence | Consequence |
|---|---|---|---|
| Layout at a phone width | **Unusable.** At about 410 px the `.main` grid is `212px 11px 152px` (screen column 11 px); hiding the rail leaves the Gateway about 120 px wide beside a 152 px F-key rail; the durable-storage warning also takes two lines. Earlier measurement (before the later phases) found a 938 px floor with sideways scroll | Dev app probe, 2026-10-22 **[C]**; `docs/design/ux-analysis-2026-10.md` section 4.12 | A real mobile entry point is needed; a CSS tweak to the shell is not enough |
| Installability | None. No `public/` folder, no manifest, no service worker. The viewport meta tag is present | `index.html`; grep of `src` found no `serviceWorker` or `webmanifest` **[C]** | PWA work starts from zero |
| Transport | Plain HTTP on port 1036; `books.comhard.in` ports 80/443 serve a different app | `docs/HANDOFF.md` section 2 **[C]** | No service worker, install, push, geolocation, Web Bluetooth, barcode detection (all secure-context) until TLS **[W]** |
| First-load weight | One JS chunk of 2.74 MB raw, about 873 KB gzip, about 683 KB brotli; CSS 291 KB (47 KB gzip); a small realtime chunk. The backend registers `@fastify/static` with no compression plugin | `dist/assets` as built 2026-10-21 21:05 (a snapshot, may lag the tree); `backend/package.json`; `backend/src/server.js:62`. I did not request the live host **[C]** | About 3 MB on first load over whatever network the rep has, unless something in front compresses; code-split a mobile entry |
| Storage durability | `navigator.storage.persist()` is requested and a warning shows if denied | `src/data/persist.ts` **[C]** | Good start; on iOS an installed Home Screen app is the safer home **[W]** |
| Identity | One login per tenant. `membership` has `owner | admin | member` but only the register path inserts a row; no invite or member endpoint | `backend/sql/002_p2.sql:12-19`, `backend/src/tenancy.js:8-11`, grep for "invite" in `backend/src` found none **[C]** | A second person cannot sign in separately; a rep would share the owner's login |
| Authorisation | Operator rights (roles, PIN) are enforced where data is written on the client; the file states it is a control for people sharing a screen, not a data boundary | `src/core/access.ts:12` **[C]** | The server cannot stop a rep reading or writing anything |
| Sync shape | A device pulls the tenant's whole change log in pages of 500, suppressing its own echoes; there is no per-user or per-collection filter | `backend/src/sync.js:20, 343-370` **[C]** | A rep's phone would hold every ledger and purchase rate; "partial sync" does not exist |
| API keys | Scopes are `read` or `write` for the whole tenant; some tables are write-denied (companies, users, roles, e-invoices, custom fields) | `backend/src/platform.js:81-90`, `backend/src/restPolicy.js` **[C]** | No per-collection or per-row scope for a rep |
| Push | No web-push, FCM or APNs code anywhere. SSE exists with the access token in the query string and an in-memory hub | `grep` of `backend/src` and `src`; `src/sync/realtime.ts`, `backend/src/realtime.js` **[C]** | Push is a new subsystem; the approvals design estimates it **[C]** |
| Language | No i18n library in `package.json`; UI is English | `package.json` **[C]** | An i18n layer must precede any translation |
| Domain engines | Pure TypeScript with no UI, store or Dexie imports; bundled by esbuild in the perf harness | `CLAUDE.md`; `scripts/perf` **[C]** | Portable to a phone shell, a Capacitor build or React Native |
| Assets that already match the job | Sales Order voucher (non-posting); pending orders; credit-limit check with override reason; bill-wise engine; outstanding message and WhatsApp/e-mail links; UPI link and QR; CRM leads and follow-ups; ghost recall; price levels; bulk entry; server-claimable voucher numbers; Sync Mismatch Report; Dashboard v2 engines | `src/types.ts:13,35,363,571`; `core/creditLimit.ts`; `core/share.ts`; `core/upi.ts`; `core/inventory.ts:315`; `backend/src/sync.js:319` **[C]** | The "business" is mostly built; the shell, the scoped door and the language layer are not |
| What the earlier research already decided | Phone approvals as a code-split PWA route reading a server inbox API, TLS a hard prerequisite, team invites a prerequisite, native app deferred | `docs/research/approvals-workflows.md` lines 23, 94, 410-420, 598-650 **[C]** | This document should extend that plan, not contradict it |

---

## 8. Delivery shape: PWA, a store wrapper, or React Native

### 8.1 What the platform facts say

- **iOS Safari tab versus Home Screen app.** Safari applies a seven-day limit across IndexedDB, local storage, service-worker caches and similar storage after seven days of Safari use without interaction on the site; Home Screen web apps are not part of Safari, track their own use and WebKit does not expect their data to be deleted **[W]**. WebKit's storage policy page adds that persistent mode is requested through `navigator.storage.persist()` and WebKit decides by heuristics such as being a Home Screen app **[W]**. Web push on iOS and iPadOS requires 16.4 or later, a Home Screen install with a manifest, and a permission prompt from a tap **[W]**.
- **Background Sync** is "limited availability" on MDN (Chromium-centred, **[J]** not in Safari or Firefox), so a queued order must be flushed when the app is open or opened, with a visible queue **[W]**.
- **Web Bluetooth, barcode detection and geolocation** are secure-context features; Web Bluetooth and the barcode API are marked limited availability and experimental on MDN **[W]**; the browser list is Chromium-centred by my general knowledge **[J]**. Van-sales printing to a Bluetooth thermal printer and camera barcode scanning are therefore weak in a browser, especially on iPhone.
- **Capacitor** wraps an existing web app in a native shell and adds native plugins; its storage guide warns that WebView storage can be reclaimed by the OS on iOS, may be persisted on Android, and recommends native SQLite for durable data **[W]**.
- **Trusted Web Activity** shows a PWA full screen in an Android app from the Play Store, requires Chrome 72 or later and a digital-asset link, and the host app cannot read the page's storage or state **[W]**.
- **Android background location** needs a declaration and review and the foreground route is preferred **[W]**.

### 8.2 Options compared

| | A. PWA only (the existing SPA, a code-split mobile entry) | B. PWA plus a store wrapper (Capacitor or TWA) | C. React Native app sharing `src/core` | D. Responsive web only (no install) |
|---|---|---|---|---|
| Reuse of today's code | Highest: same React, Dexie, outbox, sync | High: same web code inside a shell | Engines yes; every screen and the sync client rewritten | Highest |
| Offline durability | Good on Android Chrome once installed; iOS needs Home Screen install **[W]** | Native SQLite available for the durable store; WebView storage has an iOS eviction risk **[W]** | Native SQLite | Weak, especially iOS Safari tabs **[W]** |
| Push | Android web push needs HTTPS; iOS only for Home Screen apps **[W]** | FCM and APNs | FCM and APNs | None |
| Location, share-target, printers, scanners | Foreground location only; no share target; printers and scanners weak **[W/J]** | Plugins for each | Plugins for each | As A |
| Store presence | None | Play Store (and App Store) presence | Yes | None |
| Update speed | Instant | Web layer instant, shell on store review | Store review for everything | Instant |
| Build and run cost | Lowest | Moderate: signing, two store accounts, WebView on old Android | Highest (the roadmap sizes it XL **[C]**) | Lowest |
| Main risk | iOS install friction and push reliability; no TLS yet | WebView version spread on old phones; store review for location features | A second UI to keep in step with the first | The shell is unusable today and nothing is safe offline |

### 8.3 Recommendation **[J]**

**Ship A first, designed so B is a packaging step.** Reasons: one codebase and the same Dexie and outbox; instant updates matter while the team iterates with a pilot; the earlier approvals research already chose a PWA for phone approvals **[C]**; iOS is about 6% of traffic **[W]**; and the engines are portable if C is ever needed. Move to **B** only when one of these triggers is met, each one a measurable event: (1) customers or channel partners will not adopt without a Play Store listing; (2) push reliability on Android fails the pilot; (3) a pilot rep needs a Bluetooth printer, a camera barcode scan on a low-end phone, or a "share to Comsky" target from WhatsApp; (4) iOS owners will not install to the Home Screen. Choose **C** only if a low-end phone in the pilot fails the performance budget inside a WebView. Do not choose **D**: Odoo's wrapper and Zoho's online-only apps show what no-offline costs a rep.

---

## 9. [J] Positioning for Comsky Books mobile v1

### 9.1 The sentence

**Comsky Books mobile is the order-to-cash app for a Tally-class trading business: a salesperson books orders and collects payments offline against live party outstanding and credit, the owner sees and approves it on their phone, and every figure is the same as in the books.**

### 9.2 What to be best at (three bets)

1. **The field loop.** Party card, order in under a minute, bill-wise collection, visit note, with an honest sync state. Beats Marg on reliability and language, Vyapar on role safety, Zoho on offline, and the brand-side suites on owning the books.
2. **The owner's pulse and decisions.** Five or six "as at" cards, who owes me with a one-tap WhatsApp, approvals in two taps. Beats Tally's PC-dependent view and Biz Analyst's sync.
3. **Trust.** Server-scoped access (no cost on a rep's phone), revocation, a visible queue, a recoverable order. This is where every incumbent leaks and where our existing mismatch machinery can show up in words.

### 9.3 What to be merely adequate at

Party lookup and statement sharing, stock lookup by item (quantity only), invoice PDF share, a quick GST invoice for counter use (copy of the desktop template, not new tax logic), English and Hindi.

### 9.4 What not to build in v1

All fifteen voucher types on a phone; the Reports Center; GST return preparation; bank reconciliation; payroll; background location trails; the WhatsApp Business API; a native store app; offline for the owner beyond a cached last snapshot; van-sales printing and counter barcode billing (they need hardware that a browser handles unevenly, section 8); multi-company switching.

### 9.5 Architecture consequences (so the plan is honest)

| Need | Smallest honest design | Reuses | New |
|---|---|---|---|
| Rep identity | Invite flow: a second `app_user` joins an existing tenant with role `member` and a new `field` role; revoke by owner | `membership` table, `tokens.js` **[C]** | Invite endpoints, a member list, a device list (the approvals research sizes the team-invite work at about 6 to 8 days **[C]**) |
| Scoped read | A server projection per rep: assigned parties, items with sale rates and stock quantity, outstanding per party computed with the shared engines, own recent documents | `src/core` engines, tenant RLS **[C]** | A `field` pull endpoint and an assignment model (party to rep); no cost, no profit, no other users |
| Scoped write | A constrained push accepting only Sales Order drafts, Receipt vouchers saved as `optional` (pending), and visit notes | `optional` flag, `restPolicy` pattern, number claiming **[C]** | A `field` push endpoint, a visit table (a Dexie and sync allow-list change) |
| Owner thin client | A server summary endpoint and an approvals inbox endpoint, not a full local copy | Dashboard v2 engines run server-side | Endpoint and caching; push after TLS |
| Phone shell | A code-split mobile entry in the same SPA, role-routed, with a bottom bar and a service-worker app shell | `src/ui/quickCreate.ts` (one create registry), pure engines | Manifest, service worker, mobile routes, an i18n layer |
| Delivery | HTTPS, compression, long cache headers on hashed assets | `@fastify/static` options | Config and a certificate |

### 9.6 Sequence and rough size **[J]**

The sizes below are my estimate to frame a decision, not a plan; whoever builds it should re-size them. The shared prerequisites overlap the approvals research (TLS, team invites) and should be built once.

| Step | Content | Rough size |
|---|---|---|
| M0 | TLS and hostname; compression and cache headers; invite flow and member list; field role in the server; an i18n layer | 3 to 4 weeks, mostly shared with the approvals prerequisites **[C]** |
| M1 | Phone entry, manifest, service-worker shell; owner pulse (online-first) and party lookup with share | 3 to 4 weeks |
| M2 | Scoped field replica; party card, order, bill-wise collection, visit note; visible queue and plain-language status; pilot with three distributors | 6 to 8 weeks |
| M3 | Approvals and push; Hindi complete; WhatsApp polish; hardening from the pilot | 3 to 4 weeks |
| Trigger-based | Capacitor or TWA packaging; printer and scanner; share target | 2 to 4 weeks each, only when a trigger fires |

### 9.7 Success measures for the pilot **[J]**

Share of the pilot reps' orders booked through the app; median seconds to book a five-line order; orders lost or duplicated (target zero); sync failures per 100 documents and median time to "confirmed by the office"; owner opens per week and approvals made from the phone; rep weekly retention at week four; support requests per rep; cold start and battery on the cheapest handset in the group; the number of times a rep asked "did it go through?".

### 9.8 Risks

| Risk | Why it is real | Mitigation |
|---|---|---|
| TLS slips | It gates install, push, location, durable storage **[C/W]** | Make it D2 and start now |
| A scoped server door is bigger than it looks | No per-user identity, no projection, rights client-only **[C]** | Build the `field` endpoints first and test with a hostile client |
| The office does not confirm orders and receipts | An "optional" backlog erodes trust in numbers | An owner-facing "pending from the field" count, and a daily nudge |
| iOS install friction | Safari tabs can lose storage; push needs Home Screen **[W]** | Install coaching; do not promise offline on iOS in a tab |
| Low-end performance | Single 873 KB gzip chunk today; WebView limits **[C]** | Code-split; measure on a 4 GB phone; fall back to a store wrapper or RN if it fails |
| Reps reject tracking | Vendor material reports resistance and battery fears **[Ws]** | No background tracking in v1; check-in on tap, explained |
| Scope creep toward "the whole desktop on a phone" | Every incumbent ended up a lesser web **[W]** | Hold section 9.4 |
| Plan limits are display-only | `max_users` is not enforced **[C CLAUDE.md]** | Decide the field-seat price and enforce it before launch |

---

## 10. Validation plan and open questions

**Interviews and ride-alongs (about 8 sessions).** Three distributors or traders with 5 to 15 reps (pharma, FMCG, hardware or electricals), two owners who use Biz Analyst or Tally Reports in Browser today, three reps. Watch an actual order, an actual collection, and the evening when the office keys the day in. Ask: what happens when there is no signal, who is allowed to see the rate list, how is cash handed over, what gets said on WhatsApp.

**Open questions (nobody could answer them from public pages).**
1. How many of our existing customers are distributors with reps versus owner-operated traders? (Our own customer list; the target persona depends on it.)
2. Will owners accept a rep's receipt as "pending until confirmed", or do they treat a rep's receipt as final? (Interview.)
3. Which UPI flow do reps actually use at a counter, an owner QR or a personal QR? (Ride-along; it decides how much of the UPI work is a QR and how much is reconciliation.)
4. What are the real phones? (Ask for the cheapest three handsets in each pilot, then test on them.)
5. Is Hindi enough for the pilot, or do the first customers need Gujarati, Marathi, Tamil or Telugu? (Customer list by state.)
6. Are bill-wise receipts needed on day one, or is "on account" enough for the first release? (Ask the accountants, Tally users expect bill-wise.)
7. What will the owner pay per field seat? (Price test; compare Rs 700 to 1,000 a year at the Vyapar end with Rs 2,700 to 3,600 at the Busy end **[W/Ws]**.)
8. Does DPDP guidance on employee location need counsel before any pilot? **[Ws]**
9. Real Android ratings and installs for Vyapar, Marg eOrder and Biz Analyst: open the live Play pages in a browser; I could not fetch them.

---

## 11. Sources

**Tally and companions** (all **[W]** unless noted)
- Reports in Browser, help: https://help.tallysolutions.com/tally-reports-in-browser-tally/
- Docs by Ira and TallyIra, help: https://help.tallysolutions.com/docs-by-ira/
- TallyIra, App Store: https://apps.apple.com/in/app/tallyira/id6740851785
- Tally on owners and mobile: https://tallysolutions.com/tally/tallyprime-mobile-experience-for-business-owners/
- TallyPrime pricing: https://tallysolutions.com/buy-tally/
- Vendor comparison of Tally mobile options (vendor, biased): https://www.aiaccountant.com/blog/tally-on-mobile
- Biz Analyst, App Store: https://apps.apple.com/us/app/biz-analyst-app-for-tally-user/id1164789740
- Biz Analyst, **[Ws]**: https://www.capterra.in/software/203226/biz-analyst, https://www.cbinsights.com/company/biz-analyst, https://yourstory.com/companies/biz-analyst

**Ledger and billing apps**
- Vyapar about, salesman tracking, roles, multi-device, comparison with Busy: https://vyaparapp.in/about-us, https://vyaparapp.in/business-management-software/salesman-tracking, https://vyaparapp.in/blog/user-roles-permissions-vyapar/, https://vyaparapp.in/guides/how-to-use-vyapar-app-on-multiple-devices, https://vyaparapp.in/comparison-with-other-apps?brand=busy (vendor pages)
- Vyapar India App Store: https://apps.apple.com/in/app/vyapar-billing-invoice-app/id6478382307
- Vyapar competitor review: https://profitbooks.net/vyapar-review/
- Vyapar **[Ws]**: https://www.capterra.com/p/180579/Vyapar/reviews/ (the page returned 403 to the fetch tool), https://www.techjockey.com/detail/vyapar, https://www.appbrain.com/dev/Invoicing%2C+Billing%2C+Inventory%2C+GST%2C+Accounting+app/, https://vyaparapp.in/guides/how-to-send-payment-reminders-in-vyapar
- myBillBook: https://mybillbook.in/, https://mybillbook.in/s/mybillbook-vs-swipe/, https://apps.apple.com/app/id1612498254; **[Ws]** https://digitalterminal.in/amp/story/news/24127.html
- OkCredit: https://okcredit.in/en/, https://apps.apple.com/app/id1488748286
- Khatabook **[Ws]**: https://khatabook.com/en/privacy/, https://startuptalky.com/khatabook-business-model/, https://www.g2.com/sellers/khatabook, https://inc42.com/buzz/khatabook-fires-over-40-employees-restructuring-exercise; OkCredit layoffs **[Ws]**: https://inc42.com/buzz/tiger-global-backed-okcredit-lays-off-around-40-employees

**ERP companions and field sales**
- Busy: https://busy.in/busy-mobile-app-pricing, https://apps.apple.com/app/id1662346018; **[Ws]** https://www.capterra.in/software/71449/busy-accounting-software
- Marg: https://apps.apple.com/us/app/eorder/id1631309291; **[Ws]** https://www.appbrain.com/dev/Marg+ERP+Limited/, https://apps.apple.com/cd/app/eretail/id1217414784, https://www.techjockey.com/detail/margerp-9, Play listing (not fetchable) https://play.google.com/store/apps/details?id=com.marg.newmargorder
- Bizom: https://bizom.com/; FieldAssist: https://fieldassist.com/; **[Ws]** https://www.techjockey.com/detail/fieldassist, https://www.softwaresuggest.com/salesjump
- Field-sales vendor blogs (biased) **[Ws]**: https://www.fieldassist.com/blog/the-ultimate-guide-to-retailer-ordering-apps-for-fmcg-brands, https://www.pepupsales.com/blog/order-taking-app-for-field-sales-teams/, https://spirestock.com/blog/fmcg-salesman-attrition-distributor-india, https://blog.petpooja.com/people-payroll/field-sales-attendance-tracking-petpooja-payroll/, https://hubstaff.com/blog/employee-gps-tracking-battery-life/

**Global and CRM tools**
- Zoho Books mobile, pricing, India App Store: https://www.zoho.com/in/books/mobile/, https://www.zoho.com/in/books/pricing/, https://apps.apple.com/in/app/id710446064
- Zoho CRM and Bigin, App Store: https://apps.apple.com/gb/app/zoho-crm-sales-marketing/id444908810, https://apps.apple.com/us/app/-/id1454212098; **[Ws]** https://www.bigin.com/articles/bigin-india-pricing-clarification.html, https://help.zoho.com/portal/en/community/topic/access-to-zoho-invoice-offline
- QuickBooks exit: https://thepaypers.com/fintech/news/quickbooks-will-exit-the-indian-market-in-2023; **[Ws]** https://feedback.qbo.intuit.com/forums/920245-quickbooks-mobile-feedback-forum/suggestions/50473014-add-offline-mode-for-quickbooks-mobile-ios-andr
- Xero, App Store: https://apps.apple.com/us/app/xero-accounting-for-business/id441880705; **[Ws]** https://www.patronaccounting.com/blog/xero-for-indian-businesses-gst
- Odoo offline mode and App Store: https://www.odoo.com/documentation/19.0/applications/general/offline_mode.html, https://apps.apple.com/us/app/odoo/id1272543640; **[Ws]** https://oec.sh/odoo-pricing/india
- Pipedrive offline: https://support.pipedrive.com/en/article/do-the-mobile-apps-work-offline; **[Ws]** https://support.pipedrive.com/en/article/calling-and-logging-calls-in-the-mobile-app, https://appgoblin.info/apps/921456160, https://chrome-stats.com/d/com.pipedrive, https://www.lowcode.agency/blog/pipedrive-mobile-app-performance-issues

**India context**
- StatCounter, India mobile OS: https://gs.statcounter.com/os-market-share/mobile/india
- IDC, India Q2 2026 smartphones: https://www.idc.com/resource-center/press-releases/india-smartphone-shipments-decline-q2-2026/
- UPI, September 2026: https://startuptalky.com/india-upi-transactions-september-2026/
- **[Ws]** Counterpoint Q1 2026: https://counterpointresearch.com/cn/insights/india-smartphone-market-q1-2026; IAMAI-Kantar via Business Today: https://www.businesstoday.in/amp/technology/news/story/indias-internet-revolution-key-insights-from-kantar-and-iamai-report-461043-2025-01-16; WhatsApp: https://hyperleap.ai/blog/whatsapp-statistics-india-2026; WhatsApp API prices: https://montymobile.com/blogs/whatsapp-business-api-pricing-in-india-inr-rates-gst-and-the-2026-currency-migration-deadline; UPI link specification copy: https://www.labnol.org/files/linking.pdf; DPDP Rules: https://www.deccanherald.com/india/centre-notifies-dpdp-rules-implementation-planned-in-phases-spread-over-12-18-months-3798465; Flipkart PWA: https://web.dev/case-studies/flipkart

**Platform facts** (**[W]**)
- WebKit storage policy: https://webkit.org/blog/14403/updates-to-storage-policy/
- WebKit seven-day cap: https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/
- WebKit web push for web apps: https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
- MDN Background Synchronization, Web Bluetooth, Barcode Detection, Geolocation: https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API, https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API, https://developer.mozilla.org/en-US/docs/Web/API/Barcode_Detection_API, https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API
- Trusted Web Activity: https://developer.chrome.com/docs/android/trusted-web-activity/overview
- Capacitor: https://capacitorjs.com/docs, https://capacitorjs.com/docs/guides/storage
- Google Play location policy: https://support.google.com/googleplay/android-developer/answer/9799150

**This repo** (**[C]**): `src/core/access.ts:12`; `src/types.ts:13,35,363,571`; `src/core/creditLimit.ts`; `src/core/share.ts`; `src/core/upi.ts`; `src/core/accounting.ts:9`; `src/core/inventory.ts:315`; `src/data/persist.ts`; `index.html`; `package.json`; `backend/src/sync.js:20,319,343-370`; `backend/src/tenancy.js:8-11`; `backend/sql/002_p2.sql:12-19`; `backend/src/platform.js:81-90`; `backend/src/restPolicy.js`; `backend/src/server.js:62`; `backend/package.json`; `backend/src/realtime.js`; `src/sync/realtime.ts`; `dist/assets/*` (build of 2026-10-21 21:05); `docs/HANDOFF.md` section 2; `docs/ROADMAP.md:32`; `docs/design/ux-analysis-2026-10.md` section 4.12 and N11; `docs/research/approvals-workflows.md`; `docs/research/attachments-drives.md`; `docs/research/customer-vendor-portal.md`.
