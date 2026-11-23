# Live bank feeds — research and decision

**Ask (consultants, 2026-10-15):** "We should work on the online real-time integration of various banks."
**Status:** research only, nothing built. Written 2026-10-15 against the working tree on `main`
(note: `src/core/brs.ts`, `src/ui/banking/brs.tsx` and `brsWrite.ts` are **uncommitted** Phase 24
work at the time of writing; this design builds on them).
**Reader:** the owner (decision), and whoever builds it (design).

---

## TL;DR

1. **Build statement import + auto-matching first.** It needs no credentials, no contract and no
   regulator, it works offline, and it is what TallyPrime's "Auto BRS" actually is for 138 Indian banks.
   About 6 engineer-weeks. Every later live feed lands in the same pipeline.
2. **Do not plan around the RBI Account Aggregator (AA) network for bookkeeping.** Comhard cannot be a
   Financial Information User (FIU) because it is not a regulated entity. Bank data for companies,
   LLPs and partnerships appears not to be live on AA. Its fair-use caps and per-fetch pricing do not
   fit a ₹499–₹3,999/month product.
3. **For live feeds, use the bank's own "connected banking" API.** Pilot one bank, **ICICI**
   (recommended). It has the most documented ERP-partner programme, and small Indian ISVs are already
   on its partner list. Run an **Open (open.money) aggregator** conversation in parallel as the
   multi-bank fallback. Build the provider-agnostic feed platform against a **mock provider** before
   any contract, so the bank adapter is about 2–3 weeks once UAT credentials arrive.
4. **"Real-time" is not what any Indian bank gives accounting software.** Feeds are polled.
   ICICI's feed into Zoho excludes today's transactions. Kotak's refreshes on login and needs
   re-consent every 30 days. We should promise "same-day or next-day, automatically", not
   "real-time".
5. **The biggest blockers are internal and contractual, not technical.** A bank will not connect to an
   HTTP-only, single-VM deployment with no monitoring (HANDOFF §2 trust map). TLS, India hosting
   confirmation and a security review come before the first bank call. Payment initiation comes
   later still, after approvals/workflows exist.

---

## 1. What the ask really needs

"Real-time integration of various banks" bundles four different capabilities with very different
costs. They must not be built, or sold, as one thing.

| # | Capability | What the user sees | Credentials / contract needed? |
|---|---|---|---|
| A | **Statement in** | bank lines appear in Comsky Books without typing | File import: no. Live feed: yes (bank or aggregator). |
| B | **Live balance** | "Bank says ₹X as at 11:42" on the bank ledger / BRS | Yes (same API as A). |
| C | **Reconcile + post** | matched lines get their bank date stamped; unmatched lines become vouchers in a keystroke | No. This is pure engine/UI work on top of A. |
| D | **Pay out** | pay a supplier / salary batch from inside the books | Yes, plus a higher risk class (money moves). |

C is where the bookkeeper's time goes, and C works the same whether A came from a file or a feed.
That is why the order below is A(file) + C, then A(feed) + B, then D.

### User stories

**Keyboard-first (desktop/web):**
- *Accountant, month end:* "Inside Bank Reconciliation I press **Alt+O**, pick the SBI statement I
  downloaded and press **Ctrl+A**. The 212 cheque/NEFT lines that match a voucher on amount and
  cheque/UTR are stamped automatically. The 9 ambiguous ones show ranked candidates and I accept each
  with **Enter** or **1–9**. The 14 unmatched ones (bank charges, UPI receipts) I turn into vouchers
  with **N**, or into a rule with **R** so next month they post themselves. The footer's *Balance as
  per Bank Statement* fills itself and the Difference reads 0." (The key choices are in §5.6.)
- *Same accountant, daily (live feed):* "When I open the bank ledger, yesterday's lines are already
  there and the exact matches are already proposed. I only see what needs a decision."
- *CA with 40 client companies:* "One screen lists every connected account: last fetch, unmatched
  count, any feed error (consent expired, password changed at the bank)."
- *Bookkeeper:* "If I import the same statement twice, or an overlapping one, or the feed and my file
  cover the same week, I never get duplicate lines. If a day is missing, I am told which dates."

**Owner / mobile (web app at phone width, no native app assumed):**
- "On my phone I see each account's live balance, unreconciled count and the last feed time."
- *Payments (phase 5 only):* "My accountant prepares a supplier payment batch. I approve it in Comsky
  (the approvals/workflow ask), then authorise it **in the bank's own app or net banking**
  (ICICI CIB / InstaBIZ, Axis Neo, and so on). Comsky never holds the payment OTP or token."

**Explicit non-goals:** screen-scraping net banking. Storing anyone's net-banking password. Moving
money through a Comsky-controlled account (that would be payment-aggregator territory). Promising
sub-minute "real-time".

---

## 2. What exists in our codebase (and must not be broken)

| Piece | File | Why it matters here |
|---|---|---|
| BRS identity engine | `src/core/brs.ts` (`brsPosition`, `applyBankDates`, `brsKey`) | Its header already names "statement import" as a sibling feature that addresses rows by `brsKey(voucherId, entryIndex)`. It writes nothing, and its callers persist. Matching must only ever set `bankDate` through this. |
| Batch writes + one-step undo | `src/ui/banking/brsWrite.ts` (`stampBankDates`, `undoBankDates`) | One Dexie transaction, re-reads inside it, `origin: 'flag'` so no per-voucher undo spam. A match batch is the same shape. |
| BRS screen | `src/ui/banking/brs.tsx` | Keys Enter / Space / D / C / Ctrl+Z / F7 / F2 are taken. There is already a manual *statement balance* box (`stmtBal`) and a Difference row, which an import can fill automatically. |
| Entry-level bank fields | `src/types.ts` `VoucherEntry.bankDate / instrument / instrumentDate / instrumentType` | `instrumentType` (Cheque/NEFT/RTGS/IMPS/UPI…) drives the per-mode date windows in matching. |
| Bank ledger master | `src/types.ts` `Ledger.bankAcNo / bankIfsc / bankName / openingBankBalance` | `openingBankBalance` gives the opening difference when a statement starts mid-year. |
| CSV parser | `src/ui/admin/ImportData.tsx` `parseCSV` (dependency-free, BOM/quotes/CRLF) | Move it to `src/core/` and reuse it; do not write a second one. |
| Paste-a-block + full guards | `src/ui/admin/BulkEntry.tsx` | Its rule is that it is **not a validation bypass**, it commits **all rows or none**, and it never auto-deducts TDS. Vouchers created from statement lines must obey the same rules, ideally through the same validation code. |
| XML carrier repair + DOMParser shim | `src/core/tallyXml.ts` | Reuse for CAMT.053 (ISO 20022 XML) and its tests. |
| Pure-JS SHA-256 | `src/core/vault.ts` `sha256Pure` | Line IDs must be hashable on the plain-HTTP deploy, where `crypto.subtle` does not exist. |
| One module shared by server and app | `backend/src/passwordPolicy.js` + `.d.ts`, re-exported by `src/core/passwordPolicy.ts` | Use the same pattern for the line normaliser / ID function, so the feed worker and the file importer produce identical IDs. |
| Server-injected change-log rows | `backend/src/rest.js` (`device_id 'api'`, `user_id NULL`) + `realtime.js` `notifyTenant` | Precedent: the server can append docs to a tenant's change log and nudge devices over SSE. Feed lines use exactly this path. |
| Provider adapter, mock vs live | `backend/src/billing/provider.js` (`mock` / `razorpay`) | Use the same pattern for bank providers, so everything can be built and tested with `mock` before a contract exists. |
| Envelope encryption, KMS | `docs/ARCHITECTURE.md` §8 | Designed, **not built**. Bank tokens are the first secret that really needs it. |
| Trust map | `docs/HANDOFF.md` §2 | HTTP only, single VM, in-memory SSE/rate limits, no monitoring, untested restore. Each item is a bank-security-review question. |

**Parity rows this closes (and three that are already wrong):** `docs/parity-data.json`.
- `brs-auto-import` (P1, XL) has status **partial** but note "Fully manual; no import/matching". The
  status and the note disagree, which is the exact case CLAUDE.md warns about. It should read
  `missing` until Phase 1 ships.
- `bank-opening-recon` reads **missing** but `Ledger.openingBankBalance` + `bankOpeningOf()` exist.
- `brs-bulk-date` reads **missing / "Per-row only"** but `brs.tsx` ships D / C bulk stamping.
- Phase 1 below also closes `brs-format-config` (P2) and "Create/link vouchers from statement rows"
  (P2). Phase 1b closes "e-Payments / bank e-banking export" (P2, XL).

---

## 3. Landscape (verified October 2026 where stated)

### 3.1 What competitors actually ship in India

| Product | Live bank connection | Statement import / auto-BRS | Payments | Notes |
|---|---|---|---|---|
| **TallyPrime 6.0** (released 24 Jan 2025, [release notes](https://help.tallysolutions.com/release-notes-tallyprime-6/)) | "Connected Banking" for **Axis, SBI, Kotak**. ICICI appears in Tally's own bank list but not in the help text ([help](https://help.tallysolutions.com/tally-prime/banking-utilities/connected-banking-feature/)). Marketing says balances and transactions via Axis/SBI/ICICI/Kotak and payments via Axis and SBI only ([page](https://tallysolutions.com/tally/tallyprime-6-0-connected-banking-automating-banking-and-accounting)). **The sources contradict each other.** Needs valid TSS + Tally.NET ID. | Import for **138 Indian banks**, mostly Excel, ~70 CSV, **MT940 for only 7 banks** (Kotak, Yes, DBS India, Federal, StanChart, …) ([bank list](https://help.tallysolutions.com/tally-prime/banking-utilities/list-of-banks-supported-by-tallyprime-for-e-payments-auto-brs-and-cheque-formats/)). Exact matches are auto-reconciled and "potential match" rules are configurable (amount, amount tolerance, instrument number with zero/space normalisation, date range) ([Tally](https://tallysolutions.com/ssa/tally/transforming-banking-accounting-for-smes/)). Can create vouchers in bulk from statement lines, merge lines into one voucher, and auto-fill narration/instrument. | e-Payment files for **19 banks** incl. RazorpayX; online payments via connected banks | SBI linking is done in **YONO Business → API Banking → New Application → ERP Type "Aggregator ERP", ERP Provider "TallyPrime"**, pasting Tally's connection ID ([Tally SBI help](https://help.tallysolutions.com/connected-banking-feature/connection-sbi/)). Statements are **requested and then imported** ("Get Statement"), not streamed. |
| **Zoho Books / Zoho ERP (India)** | Direct integrations documented for **ICICI** ([help](https://www.zoho.com/en-in/erp/help/vendor-payments/icici.html)), **Kotak** ([help](https://www.zoho.com/in/books/help/online-payments/kotak.html)), **Axis** ([help](https://www.zoho.com/en-in/erp/help/vendor-payments/axis.html)) and **YES Bank** ([help](https://www.zoho.com/en-in/erp/help/vendor-payments/yes-bank.html)). Yodlee feeds are referenced for other banks. | CSV/OFX/TSV import | ICICI: IMPS/NEFT/RTGS from inside Zoho. Each payment needs an **OTP plus a CIB-generated token**, and CIB's own approval workflow still applies | ICICI: needs a corporate current account + CIB + FED ID. The link is approved **inside ICICI CIB → Connected Banking Approvals** with OTP. It is **bound to the one Zoho user who set it up**. **Fetched feeds exclude today's transactions.** Zoho charges nothing extra; ICICI fees follow its schedule and may include a document-processing fee. Kotak: CRN + net-banking login + OTP consent, **90-day backfill only**, **re-authentication every 30 days**, refresh on login; early-access/beta at one point. |
| **Marg ERP** | Via partner banks: ICICI (listed partner), IndusInd ([IndusInd](https://www.indusind.bank.in/in/en/personal/other/indusInd-bank-and-marg-erp.html)) | Auto-match per bank | Vendor payments | Bank-led integrations |
| **Busy** | None found | Official FAQ describes manual clearing ([Busy FAQ](https://busy.in/faqs/accounting-reports/bank-reconciliation/2/)). Excel auto-recon in Release 11.6 is claimed by a third party only (**unverified**) | — | |
| **Vyapar** | Advertises bank feeds, but its own page also says it "no longer provides" a BRS app ([Vyapar](https://vyaparapp.in/free/small-business-accounting-software/bank-reconciliation)). **Contradictory, unverified.** | | | |
| **QuickBooks** | Exited India in 2023 ([The Paypers](https://thepaypers.com/fintech/news/quickbooks-will-exit-the-indian-market-in-2023)) | | | |
| **Xero** | No evidence of Indian bank feeds. Indian users import OFX/CSV (secondary sources only) | | | |
| **ICICI BankPlugin** (bank-run, for Tally desktop) | ICICI CIB/InstaBIZ inside Tally. **Other banks via AA, with ICICI acting as the FIU**; data "stored on technology providers' cloud" ([bankplugin.com](https://www.bankplugin.com/)) | Pre-filled vouchers, reconciliation status | Payments | **₹1,200 + GST/yr, rising to ₹1,800 + GST from 1 Sep 2026**, paid by the customer to ICICI. This is the only precedent found for AA multi-bank data inside an accounting tool, and it works only because **a bank is the FIU**. |
| **Open (open.money)** | Connected-banking platform claiming **40+ banks** and "direct API integrations, no screen-scraping", 12-second median refresh (marketing, **unverified**) ([multi-bank](https://open.money/product/multi-bank)). Has **Enterprise Connected Banking APIs for ERPs**: link-account API → bank redirect → `account_linking` webhook ([docs](https://docs.enterprise-connected-banking.open.money/docs/digital-linking)). Its docs name Axis/SBI/Kotak/IDFC for digital linking and add ICICI/YES/HSBC/StanChart/HDFC elsewhere (inconsistent) | | Payouts with maker-checker | Pricing not public. Also a competitor (it ships invoicing and a Tally plugin). |
| **Suvit** | — | Bank statement (Excel/PDF) → Tally vouchers with AI ledger mapping; CA plan "from ₹10,000"/yr per a third-party listing ([Techjockey](https://www.techjockey.com/detail/suvit)) | — | Shows that "statement → vouchers" alone is a product people pay for. |

**Takeaway:** every serious Indian product does live feeds through **bilateral bank partnerships**,
not AA. Every one of them still ships **file import** as the universal path. Nobody ships true
real-time feeds for SMB accounting.

### 3.2 RBI Account Aggregator framework: what it allows us to do

- **Who can receive data.** FIUs and FIPs must be entities regulated by RBI, SEBI, IRDAI, PFRDA (or
  the Department of Revenue). **Technology Service Providers (TSPs) are "the sole unregulated entity"
  in the flow and do not receive the consented encrypted data themselves.** AAs are data-blind
  ([Sahamati FAQ](https://sahamati.org.in/faq/)). **So Comhard cannot be an FIU, and calling Setu,
  Finvu or OneMoney does not change that.** (Correction to the brief: Finvu, OneMoney, Anumati and
  Setu AA are themselves **licensed NBFC-AAs**. Setu also sells an FIU-side gateway, but only to
  regulated FIUs. Its FIU onboarding asks for the FIU's licence, Sahamati certification and a
  Central Registry entry, per [Setu go-live docs](https://docs.setu.co/data/account-aggregator/licenses-and-go-live/go-live).)
- **Governance.** RBI recognised **Sahamati as the SRO for the AA ecosystem on 5 June 2026**
  ([RBI press release](https://rbi.org.in/Scripts/BS_PressReleaseDisplay.aspx?prid=62870)). As of
  June 2026: 17 operational AAs, 176 FIPs, 1,076 FIUs (Sahamati site, via search).
- **Consent artefact.** Purpose code, profile/summary/transactions, one-time or periodic, data range,
  fetch frequency, consent validity and data life. **Fair-use templates have been enforced by AAs
  since 1 June 2025** ([FAQ](https://sahamati.org.in/faq/)). Example: the personal-finance template
  CT008 caps periodic fetches at **45 per month** ([CT008](https://sahamati.org.in/aa-consent-template-library/consent-template-id-ct008-personal-finance-management/)).
  No template covers "business bookkeeping". Purpose codes 103 ("aggregated statements") and 104
  ("account monitoring") are the closest ([purpose codes](https://sahamati.org.in/purpose-codes/)).
  AA is pull-based: there is no push of new transactions.
- **Data you would get.** ReBIT Deposit schema: per transaction `type` (DEBIT/CREDIT), `mode`
  (CASH/ATM/CARD/UPI/FT/OTHERS), `amount`, `currentBalance`, `transactionTimestamp`, `valueDate`,
  `txnId`, `narration`, `reference`. Account type SAVINGS or CURRENT
  ([ReBIT schema](https://specifications.rebit.org.in/api_schema/account_aggregator/documentation/deposit.html)).
  That is a good shape: it has a running balance and a reference, both of which matching needs.
- **The coverage gap that matters most to us.** The Holder element **requires DOB and PAN of a
  person**, and `Holders/@type` is SINGLE or JOINT only. A Sahamati deck said **sole-proprietor**
  current accounts were live at 19 of 23 banks and that **non-sole-proprietor current accounts would
  follow "once ReBIT releases technical clarifications"**
  ([deck](https://sahamati.org.in/wp-content/uploads/2022/12/GST-as-FIP-on-AA_Deep-Dive_Session.pdf)).
  A March 2026 vendor blog says partnerships, companies and trusts are **still excluded**
  ([CASParser](https://casparser.in/blog/state-of-account-aggregator-2026/)). **I could not find a
  primary source confirming either way. Treat it as a probable blocker for every Pvt Ltd, LLP and
  partnership customer**, which is a large share of a Tally reseller's base.
- **Reliability.** Sahamati tracks FIP discovery success (healthy above 85%) and missed notifications
  (healthy below 5%), but publishes per-bank numbers only to members
  ([dashboard](https://sahamati.org.in/aa-dashboard/)). Not verifiable from outside.
- **Cost.** Setu AA: "₹0.01 to ₹25 per fetch" depending on use case
  ([Setu AA pricing policy](https://setu-aa.com/pricing-policy)). Anumati (Perfios): **₹5–₹25 per
  statement fetch** ([Anumati](https://www.anumati.co.in/pricing/)). OneMoney: ₹0.30–₹9.90 per account
  (undated doc, [PDF](https://www.onemoney.in/docs/resources/FinTech%20Group%20Very%20Brief%20Business%20Summary.pdf)).
  A fresh consent is "a little over ₹5" ([Estabizz](https://www.estabizz.com/account-aggregators-price-wars/)).
  First-year integration via a TSP is estimated at **₹5–25 lakh** (vendor blog, CASParser).
  **Unit economics: one daily fetch at ₹5 is about ₹150/month per bank account, which is 30% of our
  Starter plan (₹499, `backend/sql/003_saas.sql`).**
- **The only legal route to AA for us** is a regulated partner (a bank or NBFC) acting as FIU under
  its own licence and purpose, with Comhard as its TSP. ICICI BankPlugin is a live example. This is a
  partnership and legal-structuring project, and **purpose limitation** probably confines it to that
  partner's purpose (for example a lending journey). Revisit only if a lending partnership appears.

### 3.3 Bank corporate API programmes ("connected banking")

| Bank | What is public | ERP-partner evidence | How the customer authorises | Verdict |
|---|---|---|---|---|
| **ICICI** | Developer portal with **250+ APIs**: self-register, manual review "up to 5 days", free and paid plans (unpriced) ([ICICI blog](https://www.icici.bank.in/blogs/api-banking/how-to-apply-for-api-banking-services)). Account services (balance, transaction status), payments (NEFT/RTGS/IMPS/UPI), collections (virtual accounts, auto-recon). Eligibility "corporates, fintechs, ERPs"; flow is sandbox → UAT with documents → security checks → live ([API suite](https://www.icici.bank.in/corporate/transact/online-platform/corporate-api-suite)). | **Partner list:** Zoho, TallyPrime, Marg, Kredily, Profitnx, Aakash Infoway, Ornate, Ledgers, Miracle, Paybooks, GoFrugal, AlignBooks ([partners](https://www.icici.bank.in/business-banking/cms/connected-banking/connected-banking-partners)). **Small ISVs are admitted.** | Customer approves our link in **CIB → Connected Banking Approvals** with OTP. Integrators hold CIB credentials (API key, Corp ID, User ID, aggregator name/ID), a **private key**, and a per-customer **URN** ([Mysa guide](https://support.mysa.io/en-US/mysa/article/ufoyIJ_T-setting-up-your-icici-bank-account-on-mysa)) | **Pilot #1** |
| **Kotak** | Connected-banking API suite "specifically designed for Fintechs, ERP service providers…", sandbox, "paperless customer onboarding" ([Kotak](https://api.kotak.bank.in/connectedbanking)). No public terms. | Zoho (direct feeds) | Net-banking login + OTP consent; Zoho reports **30-day re-auth** and a 90-day backfill | Pilot #2 candidate |
| **Axis** | "Neo Connect" / neo API suite, ERP-agnostic | Strategic partners include **Tally ERP**, Bank Open, Hylobiz, IndiaFilings ([Axis](https://axisbank.com/corporate/neo-by-axis-bank/neo-api-suite)); Zoho documents an integration | Approval through **Neo for Corporates** | Pilot #2 candidate |
| **YES Bank** | API portal (the public URL is a UAT domain); account details + fund transfer APIs | Zoho ERP integration; account must be **registered for API banking** with YES | Bank-side registration | Later |
| **SBI** | YONO Business → API Banking → "Aggregator ERP" | TallyPrime (listed as an ERP provider inside SBI's own portal) | Customer files the application in YONO Business | Later; we would need SBI to list Comsky as an ERP provider |
| **HDFC** | Developer portal for corporates/fintechs/ERPs; onboarding is gated (RM contact, UAT app, SSL certificate upload) ([HDFC dev FAQ](https://developer.hdfc.bank.in/faqs), [Open's HDFC flow](https://docs.enterprise-connected-banking.open.money/docs/hdfc-account-linking-flow)) | No Tally or Zoho HDFC connector confirmed | Via RM | Later / via Open |
| **IDFC FIRST** | **No public developer portal found** | — | — | Via Open, if at all |

**What a bank partnership will demand from us (expected, not yet confirmed; ask each bank):**
- NDA before UAT (ICICI: stated).
- A production security review. RBI's IT-outsourcing directions keep **banks** accountable for their
  third parties, so banks push audit rights, VAPT evidence (typically from a CERT-In-empanelled
  auditor), BCP and incident clauses into partner contracts
  ([RBI IT outsourcing 2023 summary](https://vinodkothari.com/2023/04/rbi-regulates-outsourcing-of-it-services-by-financial-entities/)).
- A static, whitelisted egress IP and client-certificate/encrypted payloads (the private key
  requirement in the ICICI integrator flow points this way).
- HTTPS callback URLs.

**None of the banks publish partner commercial terms. Every cost line for a direct bank feed in this
document is "unknown until asked".**

### 3.4 Payment initiation (capability D)

| Route | What it needs | Cost (published) | Notes |
|---|---|---|---|
| **RazorpayX via Partner OAuth** | Comhard registers as a Razorpay **Technology Partner**. The customer grants OAuth scopes `rx_read_write` or `rx_partner_read_write`. Tokens are server-side only, the access token expires every 90 days and the refresh token after 180 days ([Razorpay OAuth](https://razorpay.com/docs/partners/technology-partners/onboard-businesses/integrate-oauth)). The customer needs a RazorpayX account with a **current account through a partner bank**. Docs say RBL/Yes/IDFC FIRST/Axis; the pricing page names ICICI/RBL/Yes ([account types](https://razorpay.com/docs/x/account-types/)). RazorpayX Lite is closed to new merchants. | **Plans ₹1,499 / ₹4,199 / ₹9,799 per month + GST**, API Payouts bundle ₹4,000/month, **plus per-payout fees** ([pricing](https://razorpay.com/x/pricing/)). Payouts page: IMPS/UPI ₹4/₹6/₹9 by slab, NEFT/RTGS ₹5 flat ([payouts](https://razorpay.com/x/payouts/)). The two pages differ in shape; confirm. | Simplest legally: Razorpay and the bank carry the regulated parts. Our existing Razorpay adapter is billing-only (MOCK). |
| **Cashfree Payouts** | Business KYC (PAN-led, Aadhaar OTP for the owner); first credit must come from the customer's own KYC'd account | Published slabs ₹3–₹15 per transfer + GST + bank charges ([Cashfree](https://www.cashfree.com/en/payouts/)). The IMPS cap is stated as both ₹2 lakh and ₹5 lakh in different docs | Confirm whether payouts run from the customer's own account or a Cashfree-held balance before choosing it |
| **Bank payment APIs** (ICICI/Axis/Kotak) | Same partnership as feeds + beneficiary registration + bank-side approval workflow (CIB maker-checker) | Bank schedule of charges | Most "native", and the bank's own approval remains the control |
| **e-Payment bulk-upload file** (Tally's 19-bank route) | Nothing from a bank partner. We generate the bank's bulk-upload file and the customer uploads it in net banking | ₹0 | **Can be built now.** Needs each bank's template, which customers can download from their CIB |

Rule for every route: Comsky **never holds customer funds** (a pooled account would make us a
payment aggregator) and **never captures the bank OTP or token**. Bank-side authorisation stays the
final control. Comsky's own approval (from the workflows ask) is an extra gate before it, never a
replacement.

### 3.5 Statement file formats in practice
- Excel/CSV dominate. MT940 is rare for SMB accounts (Tally lists only 7 banks). CAMT.053 adoption in
  India is uneven, and the only source is a vendor blog
  ([Terra Insight](https://www.terra-insight.com/insights/mt940-vs-camt-053-vs-mt942-india/), **unverified**).
- From engineering experience (verify on real samples): Indian bank "Excel" downloads often start
  with 10–25 rows of account metadata before the header. Some `.xls` files are really HTML or
  tab-separated text, so **sniff the content, not the extension**. Amounts may carry `Cr`/`Dr`
  suffixes or come as a single signed column. Date formats vary (`dd/mm/yy`, `dd-MMM-yyyy`).
- Many SMBs only receive **password-protected PDF** statements by e-mail. PDF parsing is fragile.
  Deferred (see options).

---

## 4. Options compared

| Option | Delivers | Our effort | External cost | Dependency / contract | Risk | Verdict |
|---|---|---|---|---|---|---|
| **A. Statement file import + matching** (CSV/TSV/paste, XLSX, MT940, CAMT.053) | A (file) + C, every bank, offline | **5–7 eng-weeks** | ₹0 | None | Low. Parser drift per bank format (mitigated by saved column mappings) | **Do first** |
| **B. Direct bank API (ICICI pilot)** | A (feed) + B (+ D later), one bank | 2–3 wk adapter + **3–4 wk shared feed platform** | Unknown partner fees. Customer pays bank charges (and possibly a processing fee) | ICICI developer registration, NDA, UAT, security review, partner agreement | Medium: calendar time, security review, per-bank rework | **Pilot #1** |
| **C. Connected-banking aggregator (Open Enterprise Connected Banking)** | A (feed) + B for many banks with one integration | 2–3 wk adapter on the same platform | Not public | Commercial + data-processing agreement with Open | Medium: intermediary concentration risk, competitor, coverage claims inconsistent | **Parallel conversation; fallback** |
| **D. Account Aggregator via a regulated FIU partner** | A (feed) for individuals and sole proprietors, multi-bank | 3–4 wk + legal structuring | ₹0.01–25/fetch + ₹5–25 lakh first year (estimates) | A bank/NBFC FIU willing to carry our use case; Sahamati certification via that FIU | **High**: purpose limitation, company accounts not live, fair-use caps, cost per fetch | **Not now.** Revisit with a lending partner |
| **E. Screen-scraping / storing net-banking passwords** | A (feed) | — | — | Violates bank terms. Credential custody liability | Unacceptable | **Rejected** |
| **F. E-mailed PDF statements** (forward address + PDF parse) | A (file) for PDF-only customers | 2–4 wk per few banks, ongoing upkeep | ₹0 (OCR would cost) | Customer must give the PDF password | Medium-high: brittle, holds a secret | Phase 4+, demand-driven |
| **G. Payouts via RazorpayX Partner OAuth** | D | 4–6 wk (after approvals exist) | Customer: RazorpayX plan + per-payout | Razorpay Technology Partner registration | Medium: money movement, support load | Phase 5 |
| **H. e-Payment bulk-upload file export** | D (semi-manual), every bank | 1.5–2 wk for 3 banks | ₹0 | None (bank templates from customers) | Low | **Phase 1b** (parity P2) |

---

## 5. Recommended architecture on our stack

### 5.1 Principles
1. **Statement lines are facts; matches are decisions.** A line is immutable once received and is
   never edited by a user. A match lives on the voucher entry (`bankDate` + `bankLineId`), exactly
   where `brsPosition` already reads reconciliation from. Matching is **derived** by joining the two.
   So deleting a voucher automatically un-matches its line, and nothing else can drift.
2. **The server never writes vouchers.** The feed worker writes only statement-line docs. Every
   voucher change (stamp, link, create) is made on a device by a person through `putDoc`, so it is
   user-stamped in the outbox like everything else. This keeps the server payload-opaque
   (ARCHITECTURE §9) and avoids a new whole-document last-write-wins collision class on vouchers.
3. **One canonical line model and one ID function for every source** (file, feed, aggregator). The
   same transaction from a file and from a feed must collapse to one line.
4. **Secrets never reach a device.** No bank token, key or customer identifier secret is ever sent to
   the SPA or Electron. Devices see connection *status* only.
5. **HTTPS before any bank traffic.** It is non-negotiable and is Phase B item #1 anyway.

### 5.2 Client data model (Dexie v8, additive)

```ts
// src/types.ts
export interface BankLine {
  id: string;            // bankLineId(): deterministic, see 5.3
  cid: string;
  ledgerId: string;      // the Bank ledger this statement belongs to
  date: string;          // ISO posting/transaction date -> becomes VoucherEntry.bankDate on match
  valueDate?: string;
  signed: number;        // BOOKS sense, same as core/brs.ts: + = money INTO the account (bank "credit",
                         // a Dr on the bank ledger), - = money out. Never store the bank's Cr/Dr wording.
  balance?: number;      // running balance after this line, BOOKS sense (+ = funds with the bank)
  narration: string;
  ref?: string;          // cheque no / UTR / RRN exactly as the bank printed it
  mode?: 'CHEQUE' | 'NEFT' | 'RTGS' | 'IMPS' | 'UPI' | 'CASH' | 'CARD' | 'NACH' | 'CHARGES' | 'INTEREST' | 'OTHER';
  ord: number;           // position within the bank's day (tie-break for identical lines)
  source: { kind: 'file' | 'feed'; ref: string; at: number; by?: string }; // file sha256 | connection id
  providerTxnId?: string;
}
export interface BankLineState { id: string /* = line id */; cid: string; status: 'ignored'; reason?: string; by: string; at: number }
export interface BankRule {      // a master: "narration contains AIRTEL -> Telephone Expenses, Payment"
  id: string; cid: string; ledgerId?: string /* bank ledger, or all */;
  when: { contains?: string[]; refPattern?: string; dir?: 'in' | 'out'; min?: number; max?: number };
  then: { typeId: string; counterLedgerId: string; narration?: string; costCentreId?: string };
  priority: number; hits: number; lastUsed?: number;
}
// VoucherEntry gains:  bankLineId?: string   (optional field; no index needed)
// Ledger (bank) gains: stmtFormat?: StatementMapping  (saved column mapping; embedded like chequeLayout)
```

- `db.ts` v8: `bankLines: 'id, cid, [cid+ledgerId+date]'`, `bankLineStates: 'id, cid'`,
  `bankRules: 'id, cid'`. Additive only, like v7.
- **Do not put `bankLines` in the global Zustand store.** `store.ts` loads whole collections into
  memory. A UPI-heavy retailer can produce thousands of lines a day. Query Dexie per
  (ledger, period) through the compound index instead. `bankRules` is a normal master: follow the
  "New master table" checklist in CLAUDE.md (`COLLECTIONS`, `checkDelete`, …).
- All three tables join `ALL_TABLES`, so they sync and back up like everything else.

### 5.3 Line identity, dedupe and continuity: `backend/src/bankLine.js` (+ `.d.ts`, re-exported by `src/core/bankLine.ts`)

One pure module with no imports, shared exactly the way `passwordPolicy.js` is shared:
- `normalise(raw, mapping) -> BankLine[]`: dates, `signed` in books sense, `ref` (UTR/RRN/cheque).
  Typical shapes: CTS cheque numbers are 6 digits, NEFT UTR 16 characters, RTGS UTR 22, IMPS/UPI RRN 12
  digits. Extract them from the narration when there is no reference column.
- `bankLineId(l) = 'bl_' + hex(sha256(cid | ledgerId | date | signed.toFixed(2) | balance?.toFixed(2) ?? '' | normRef | ordAmongIdentical))`,
  where `ordAmongIdentical` counts earlier lines that are identical on every other component the
  same day. The balance is in the key because two identical amounts on one day normally leave
  different balances. The ordinal is still needed because +X, −X, +X repeats a balance. Hash with
  `sha256Pure` (`src/core/vault.ts`), because `crypto.subtle` does not exist on the HTTP deploy.
  `providerTxnId` is an attribute, not the key, so a file line and a feed line for the same
  transaction collapse to the same ID.
- **Partial days.** A file downloaded mid-day ends with a partial "today", which can shift ordinals.
  Default: drop lines dated on the download date unless the user opts in, and mark the last day
  "possibly partial". (ICICI's own feed into Zoho excludes today's transactions, probably for the same
  reason.)
- `continuity(lines)`: for consecutive lines, `prev.balance + line.signed === line.balance` to the
  paisa. Otherwise report a **break** (missing or reordered lines). Across imports, the first new
  line's opening (`balance − signed`) must equal the last known balance, otherwise report a **gap**
  ("12-Jun → 15-Jun not imported"). The statement screen header reads "Continuous from 1-Apr-26 to
  14-Oct-26 · closing 12,34,567.89" or names the gaps.
- **Auto-filled statement balance.** When continuity holds up to the BRS period end, the existing
  `stmtBal` box in `brs.tsx` is filled with the last line's balance on or before `to`, so the existing
  Difference row works without typing.

Re-importing the same file, or two devices importing it offline, produces identical IDs and therefore
idempotent puts. No duplicates by construction.

### 5.4 Matching engine: `src/core/bankMatch.ts` (pure, asserted like `brs.assert.ts`)

Input: `brsPosition(...).rows` that are unreflected (or reflected without a `bankLineId`), the
unmatched non-ignored `BankLine`s, and the rules. Output: a plan that writes nothing.

```ts
interface Pair { lineIds: string[]; keys: string[] /* brsKey */; tier: 1|2|3|4; why: string }
interface MatchPlan { auto: Pair[]; suggest: { lineId: string; candidates: Pair[] }[]; conflicts: Conflict[] }
```

| Tier | Rule | Default action |
|---|---|---|
| 1 | Same direction, same amount (paisa-exact), and the normalised cheque/instrument number equals the line's ref (leading zeros/spaces stripped, last 6 digits for CTS) | **auto** |
| 2 | Same direction and amount, and the line's UTR/RRN appears in the entry's `instrument` or the voucher narration, with line date within the window | **auto** |
| 3 | Same direction and amount, line date within the window for the entry's `instrumentType`, and **unique on both sides** | suggest (auto is an opt-in per bank ledger, as in Tally's "exact matches automatically") |
| 4 | Many-to-one: a set of ≤ 6 unreflected Dr entries (deposits) within the window that sums exactly to one credit line ("CHQ DEP", "BY CLG") | suggest only |

- **Windows** (configurable per bank ledger): Cheque from `instrumentDate ?? v.date` to +7 days;
  NEFT/RTGS/IMPS/UPI 0 to +2 days; Cash 0 to +1 day. A line *before* the voucher date is allowed only
  for tier 1/2, which mirrors `applyBankDates`' "early" count.
- **Never auto on ambiguity.** If two candidates tie at the same tier, propose both and auto
  neither. Ten ₹500 UPI receipts on one day is the normal case, not an edge case.
- **Tolerance is never auto.** A receipt short by ₹17.70 (charges plus GST) is suggested as "match
  plus create a Bank Charges voucher for the difference".
- **Self-transfers (Contra).** When both bank ledgers have statements, an out-line on A and an in-line
  on B with the same amount within 0–2 days propose one Contra matched to both lines.
- **Invariants to assert** (the target is hundreds of assertions, like `brs.assert.ts`): matching only
  changes `bankDate` and `bankLineId`; `brsPosition`'s identity holds before and after; with every
  line up to `to` matched and continuity intact, `statement balance − brsPosition.bank =
  openingDifference`-adjusted 0; a plan applied twice changes nothing; a line can never end up linked
  to entries whose signed sum differs from it (otherwise it is reported as a *conflict*, never written);
  optional and non-posting vouchers never match (`postsToLedgers`).

**Writing a match** extends `applyBankDates` with an optional `lineId` per key. When it stamps, it
sets `bankDate = line.date` and `bankLineId`. When it clears (key C, or a blank Enter), it removes
**both** fields. A manual Enter/D stamp leaves `bankLineId` unset, because the user asserted the
date without a line. Everything goes through `stampBankDates`' transaction / `origin: 'flag'` /
batch-undo path unchanged. So **A** (accept all exact) and **Ctrl+Z** are one step, and a voucher
edited since is left alone.

**Derived line state:** *matched* (linked entries sum to the line), *partial/conflict* (linked
entries sum to something else, e.g. a voucher amount edited after matching, or two offline devices
matching the same line to different entries), *ignored*, or *unmatched*. Conflicts also feed the
separate "sync mismatch report" ask.

### 5.5 Creating vouchers from unmatched lines
- **Proposal order:** an explicit `BankRule` (highest priority wins). Otherwise a **learned** hint from
  this ledger's history: the same UPI VPA, IMPS name or narration stem was last posted to ledger X. That
  is the same idea as ghost recall (`src/core/recall.ts`). Otherwise base-type defaults: in-line →
  Receipt, out-line → Payment, own-bank counterpart → Contra, "CHRG"/"GST ON CHG" → Payment to
  Bank Charges, "INT.PD"/"INT CR" → Receipt from Bank Interest.
- **Same guards as BulkEntry.** The commit is one Dexie transaction. It is not a validation bypass:
  unknown ledger, unbalanced or locked-period rows are hard stops, and negative cash / duplicate
  number are overridable once. **TDS is never auto-deducted.** Each created voucher's bank entry is
  born with `bankDate = line.date` and `bankLineId`, so it is reconciled at creation.
- **Bill-wise parties:** offer the party's open bills (`openBillsFor`, `core/billwise.ts`). Per the
  Phase 20-21 gotcha, a recalled or defaulted allocation must **go through the bill prompt** or be
  explicitly chosen as On Account. It must never silently post On Account.
- **GST on bank charges:** post the GST line to the input-tax ledger only if the user's rule says so,
  and flag that ITC needs the bank's invoice (GSTR-2B). Never infer eligibility.
- **R (make rule from this line)** pre-fills `when.contains` with the narration's stable stem, with
  digits, dates and reference numbers stripped.

### 5.6 UI and keys (superset rule: no existing key changes meaning)

**BRS screen** (`brs.tsx`), additions only. Enter / Space / D / C / Ctrl+Z / F7 / F2 keep their
current meaning.

| Key | Action |
|---|---|
| **Alt+O** | Import statement: file picker, or paste a block like BulkEntry. Mapping preview, Ctrl+A to accept. This mirrors TallyPrime's Import key. Today Alt+O means *Log out* only inside Sync Settings (`company/screens.tsx`), which is a different screen, so it is safe. Confirm against the frozen-key list. |
| **M** | Toggle the match pane. The right half lists statement lines; the cursor row shows ranked candidates. |
| **A** | Accept all tier-1/2 proposals shown. The bar names the count first ("Accept 212 matches?"). One undo step. |
| **1–9 / Enter** (in the match pane) | Accept candidate n / the top candidate |
| **N** | New voucher from the highlighted unmatched line (pre-filled; Ctrl+A to save) |
| **R** | Rule from this line |
| **I** | Ignore line (with reason). Shift+I un-ignores |
| **F7** | Filter also cycles: Unmatched lines / Conflicts |

**New screen "Bank Statement"** (Imported Bank Data): the lines, match state, continuity and gaps,
source (file/feed), and for feeds the connection status and last fetch. Follow the new-screen
checklist: the `Screen` union in `store.ts`, a route and crumb in `router.tsx`, `MenuScreen.tsx`,
`modern/CommandPalette.tsx` and `overlays.tsx` GoTo. Add every new chord to the `?` sheet.

**Bank connections** (cloud accounts only): under Banking. Connect / Refresh now / Disconnect, plus
the status with plain reasons ("consent expired, re-approve in ICICI CIB").

### 5.7 Backend (live feeds) — `backend/sql/007_bankfeeds.sql` + `backend/src/bankfeeds/`

```sql
CREATE TABLE bank_connection (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
  cid text NOT NULL, ledger_id text NOT NULL,            -- the client's bank Ledger.id
  provider text NOT NULL,                                 -- mock | icici | kotak | axis | open
  account_masked text,                                    -- 'XXXXXX1234' only
  status text NOT NULL CHECK (status IN ('pending','active','expired','revoked','error','paused')),
  consent_expires_at timestamptz, cursor jsonb,           -- provider pagination / last complete day
  last_success_at timestamptz, last_error text, fail_count int NOT NULL DEFAULT 0,
  created_by uuid, created_at timestamptz NOT NULL DEFAULT now(), revoked_at timestamptz, revoked_by uuid,
  UNIQUE (tenant_id, cid, ledger_id)
);
CREATE TABLE bank_secret (                                -- envelope-encrypted per-connection secrets
  connection_id uuid PRIMARY KEY REFERENCES bank_connection(id) ON DELETE CASCADE,
  tenant_id uuid NOT NULL, ciphertext bytea NOT NULL, iv bytea NOT NULL, tag bytea NOT NULL,
  wrapped_dek bytea NOT NULL, kek_version int NOT NULL, updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE bank_fetch_run (                             -- audit of every pull; no raw payload kept
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, tenant_id uuid NOT NULL,
  connection_id uuid NOT NULL REFERENCES bank_connection(id) ON DELETE CASCADE,
  trigger text NOT NULL, range_from date, range_to date, started_at timestamptz NOT NULL DEFAULT now(),
  finished_at timestamptz, lines_received int, lines_new int, status text, error_code text,
  provider_request_id text, response_sha256 text
);
CREATE INDEX IF NOT EXISTS idx_change_doc ON change_log(tenant_id, tbl, doc_id);  -- dedupe (also speeds REST GET /v1/:c/:id)
-- RLS: ENABLE + FORCE on all three, policy tenant_id = current_setting('app.tenant_id', true)::uuid.
-- bank_secret: REVOKE ALL FROM comhard_app. Only a new role comhard_bankfeed (no BYPASSRLS) may SELECT it.
-- The API process writes it only through SECURITY DEFINER bank_secret_put()/bank_secret_drop():
-- the web process can store or destroy a secret but can never read one back.
```

The ARCHITECTURE notes RLS is `ENABLE` (owner bypasses) today. New bank tables should be `FORCE`
from day one.

**Provider interface** (`bankfeeds/provider.js`, same shape as `billing/provider.js`):
`beginLink({cid, ledgerId, hint}) → {redirectUrl?|instructions}`, `completeLink(params) →
{maskedAccount, secrets}`, `fetchStatement(secrets, {from, to, cursor}) → {raw[], nextCursor, asOf}`,
`fetchBalance(secrets) → {balance, asOf}`, `revoke(secrets)`. **`mock`** replays a fixture statement,
including intra-day arrivals, an expired consent and a gap, so the whole pipeline and the UI are built
and tested before any bank signs.

**Endpoints** (owner/admin membership role for connect/revoke; any member may read status):

| Method | Path | Notes |
|---|---|---|
| POST | `/bank/connections` | `{cid, ledgerId, provider}` → link instructions / redirect. Blocked under admin impersonation. |
| GET | `/bank/connections` | Status only. Never secrets or the full account number. |
| GET | `/bank/callback/:provider` | OAuth/consent return. `state` + PKCE where supported. HTTPS only. |
| POST | `/bank/connections/:id/refresh` | Enqueue a fetch. Rate-limited (1/min per connection, 10/hour per tenant). |
| GET | `/bank/connections/:id/balance` | Live balance, cached 60 s |
| DELETE | `/bank/connections/:id` | Revoke at provider → `bank_secret_drop` (crypto-shred) → status `revoked` |
| GET | `/bank/connections/:id/runs` | Fetch history (the audit trail the CA sees) |
| POST | `/webhooks/bank/:provider` | Raw-body HMAC verify, same pattern as `/webhooks/razorpay` |

**Worker** (`comhard-bankfeed`, its own systemd unit, DB role and file-permissioned secrets).
`bank_due_connections()` (SECURITY DEFINER) returns `(tenant_id, id)` pairs. For each, it runs
`withTenant`. It decrypts the secret in memory only and fetches `cursor → yesterday` (plus today
where the bank allows it). It normalises with `bankLine.js`, skips IDs that already exist
(`idx_change_doc`), and appends the new lines to `change_log` as `tbl='bankLines'`,
`device_id='bankfeed:<connection>'`, `user_id NULL` (the REST precedent). It then records a
`bank_fetch_run` row, calls `notifyTenant(seq)` and advances `cursor` only after commit.
- Schedule: every 30–60 min in banking hours, plus once after midnight for the complete previous
  day, plus manual refresh. Bounded by each bank's rate limits once known.
- Backoff on failure. On consent expiry the status becomes `expired` and the user is told why.
- **Entitlement:** if `getEntitlement(tenant).canSync` is false, stop fetching (status `paused`) and
  incur no bank cost.
- Tenant webhooks for bank lines are **off by default**. Bank data leaving to a customer URL is the
  governed egress case in ARCHITECTURE §8.
- Single-VM/in-memory scheduling is acceptable for the pilot, the same tier as SSE today. Move it to
  the BullMQ/`server_outbox` design in ARCHITECTURE §9 at scale.

### 5.8 Sync semantics and offline behaviour

| Situation | Behaviour |
|---|---|
| Desktop/Electron with no cloud account | File import, matching and voucher creation all work fully offline. No feeds. |
| Cloud account, device offline | The server keeps fetching. Lines wait in `change_log` and arrive on the next pull (500-row pages). Matching runs locally on the lines already pulled. The BRS header shows "Feed last received 14-Oct 23:10". |
| Same file imported on two offline devices | Same IDs → idempotent puts → one line each |
| Feed and file cover the same days | Same canonical ID → one line. The later write updates only `source` (whole-doc LWW, harmless because the facts are identical). |
| Two devices match the same line differently while offline | Both voucher writes land (different docs) and the line becomes a **conflict**: shown in BRS and in the mismatch report. Nothing is silently dropped. |
| Device 1 edits a voucher offline; device 2 matches the same voucher | Today's whole-document LWW on vouchers applies, exactly as it does for a manual bank-date stamp now. Fixed properly by the planned `baseRev` quarantine / `op:'patch'` (ARCHITECTURE §6.3). Until then, a match **re-reads inside the transaction** (as `stampBankDates` does) and the mismatch report flags edits that were superseded. |
| Subscription lapsed (402 on push) | Feeds paused. File-imported lines stay local until sync resumes. Nothing is lost. |

Volume note: a busy current account may add 100k–200k lines a year, and `change_log` is never
compacted (ARCHITECTURE notes compaction as future work). That is fine for Postgres, but it lengthens
first-device bootstrap. Plan per-(company, table) snapshots before onboarding UPI-heavy retailers at
scale.

### 5.9 Audit trail
- Every match, unmatch and created voucher is an outbox row stamped with the operator, which syncs
  into `change_log`. The Cancelled/Deleted register and audit views already read this.
- Every feed line carries `source` (connection, fetch time). Every fetch is a `bank_fetch_run` with a
  provider request ID and a response hash (provenance without keeping the raw bank payload).
- Connection lifecycle (created, approved, expired, revoked, by whom) is recorded as rows in a
  tenant-visible `bank_event` log (or `bank_connection` columns + runs). Platform staff actions stay in
  `admin_audit`.

---

## 6. Security, privacy and legal notes

- **No credential custody, no scraping.** We never ask for, store or proxy net-banking passwords.
  Banks authorise us through their own consent screens (CIB approval, OAuth, OTP).
- **Two very different secrets:**
  1. **Comhard-level partner credentials** (API keys, the private key for signing/encryption, client
     certificate). If these leak, an attacker could pull the statements of *every* customer who
     approved us, with only the bank's IP whitelist as the second wall. Keep them out of `.env` in the
     repo tree, in a file readable only by the `comhard-bankfeed` service user (later KMS/HSM), and
     rotate on staff change.
  2. **Per-customer tokens and identifiers** (OAuth tokens, URN, Corp/User ID). Envelope-encrypt them
     with AES-256-GCM, a per-tenant DEK and a KEK (env now, AWS KMS `ap-south-1` later per ARCHITECTURE
     §8), with AAD = `tenant_id‖connection_id` so ciphertext cannot be swapped across tenants. Redact
     them from logs.
- **Prerequisites the current deploy fails** (HANDOFF §2): HTTP only (banks require HTTPS callbacks,
  and bank data must not travel in clear text), no monitoring or alerting, untested restore, single VM.
  The SSE endpoint takes the access token in the query string (`/sync/events?access=`, which lands in
  logs). Do not copy that pattern to any bank endpoint.
- **Data residency.** Confirm the VM ("Internal-VM7") is in India. Banks will ask, and the
  ARCHITECTURE residency stance assumes it. **Not verified in this research.**
- **DPDP Act 2023 / DPDP Rules 2025** (notified 13 Nov 2025; core fiduciary duties from **13 May
  2027**; [AZB](https://www.azbpartners.com/bank/update-indias-digital-personal-data-protection-framework-comes-into-effect/)):
  - For business customers, Comhard is broadly a **data processor**. But a sole proprietor's account
    holder *is* an individual, and narrations carry counterparties' names, VPAs and account
    fragments, so this is personal data.
  - **Purpose limitation:** use lines only for that customer's bookkeeping. No cross-tenant
    analytics, no credit scoring and no model training without separate consent.
  - **Minimisation:** keep normalised lines, not raw bank payloads.
  - **Retention:** lines are part of books (statutory retention under GST/IT). Connection secrets are
    shredded on disconnect. Logs are kept at least 1 year (DPDP Rules) and at least 180 days in India
    (CERT-In 2022 directions, [summary](https://psalegal.com/new-cert-in-directions-overview-and-implications/)).
  - **Breach reporting:** "without delay" plus a detailed report within 72 hours to the Data
    Protection Board, CERT-In within 6 hours, and the bank's contractual clock, whichever is
    stricter.
  - Write the data-processing terms into the customer ToS before the pilot.
- **RBI.**
  - AA Master Directions: we cannot be an FIU, and a TSP does not receive data.
  - We must never hold or route customer funds (payment-aggregator authorisation territory).
  - Banks' IT-outsourcing obligations will reach us by contract (audit rights, BCP, sub-processor
    disclosure). Budget for a VAPT before the pilot.
- **Access control in the app.** Client `RightArea` has no banking right (`types.ts`). Import and match
  can sit under `accounting`. **Connect/disconnect a bank** should need the tenant `owner`/`admin`
  membership role on the server **and** company `admin` on the client. Payments get their own right
  later. Admin-portal impersonation must not be able to connect, revoke or pay.
- **Trademark/copy:** the UI says "Bank Statement", "Bank connections", never "Tally". Comparative
  mentions belong only in migration copy (CLAUDE.md).

---

## 7. MVP without external credentials: **yes**

Everything below can be built, tested and shipped **without signing anything**:

1. **Statement import:** CSV/TSV/clipboard paste; XLSX/XLS (needs a parser library, see §10 Q10);
   MT940; CAMT.053. Header-row detection, a column-mapping preview saved per bank ledger
   (`stmtFormat`), presets for the top banks built from real sample files.
2. **Line store, dedupe, continuity and gaps,** with the statement balance filled into BRS
   automatically.
3. **Matching engine** (tiers 1–4) with the BRS match pane, accept-all and one-step undo.
4. **Vouchers from unmatched lines,** rules, learned hints and the Contra pairing.
5. **(1b) e-Payment bulk-upload file export** for 2–3 banks, using templates customers download from
   their own net banking.
6. **(2) The whole live-feed platform on the `mock` provider:** tables, worker, vault, endpoints,
   connection UI, SSE, entitlement pause, audit. It is real code exercised end to end; only the bank
   adapter is missing.

**Acceptance for 1–4:**
- Real statements from at least 5 banks (SBI, HDFC, ICICI, Axis, Kotak) import without manual
  column fixes after one saved mapping.
- An overlapping re-import adds 0 lines.
- On a sample company month, at least 80% of cheque/NEFT lines auto-match at tier 1/2 with **zero**
  wrong auto-matches. Measure it; do not assume it.
- Difference = 0 whenever continuity holds and everything is matched.
- `npx tsc --noEmit` = 0 and the assertion suite is green.

**What needs a contract or credentials (and who signs):**

| Item | Counterparty | What the owner signs / provides | Cost known? |
|---|---|---|---|
| ICICI connected banking (feeds, balance, later payments) | ICICI Bank | Developer-portal registration (Comhard entity), **NDA** for UAT, partner/aggregator agreement, security-review evidence. A Comhard ICICI current account is useful for UAT. | Partner side: **no**. Customer side: ICICI schedule of charges (+ possible processing fee). |
| Kotak or Axis (second bank) | Bank | Same shape. Kotak via its connected-banking portal, Axis via Neo Connect / RM | No |
| Open Enterprise Connected Banking (multi-bank fallback) | Open Financial Technologies | Commercial agreement + DPA (Open becomes a sub-processor) | No (not published) |
| RazorpayX payouts | Razorpay | Technology Partner registration (OAuth app); the customer holds the RazorpayX plan | Customer: published plans + per-payout |
| AA (only if a lending partnership appears) | A bank/NBFC FIU + its AA/TSP | FIU-side agreements are the partner's. Ours is a TSP/processing agreement plus legal review of purpose | Per-fetch ₹0.01–₹25 (published ranges) |
| Security posture | CERT-In-empanelled auditor; TLS certificate | VAPT engagement; DNS/TLS setup (HANDOFF §2) | Market-priced; not researched |

---

## 8. Phased plan

| Phase | Scope | Effort (1 senior dev who knows this codebase) | Calendar / gate |
|---|---|---|---|
| **0. Prerequisites** (parallel, not bank-specific) | TLS on `:1036`; confirm VM region; collect anonymised sample statements (2–3 per bank, Excel/CSV/PDF) from customers with consent; fix the 3 stale parity rows | 2–4 days dev + owner time | Start now |
| **1. Statement import + matching + vouchers** | §7 items 1–4; `bankLine.js` shared module; Dexie v8; Bank Statement screen; BRS keys; assertion suite + adversarial review + live test on a real company | **5–7 eng-weeks** (parsers 2, store/continuity 1, matching + BRS 1.5–2, voucher creation + rules 1.5, review/test 1) | ~2 months with review. **Closes 3 parity rows.** |
| **1b. e-Payment bulk files** | 2–3 banks' bulk-upload formats from the Payment Advice data (`banking/advice.tsx`) | 1.5–2 weeks | Optional; parity P2 |
| **2. Feed platform on mock** | §5.7–5.9 with the `mock` provider; connections UI; worker; vault; FORCE RLS; runs log | **3–4 eng-weeks** | Start bank conversations at the same time |
| **3. ICICI pilot adapter** | Link/approval flow, statement + balance APIs, encryption/signing per ICICI spec, UAT, 3–5 friendly customers | **2–3 eng-weeks** once UAT access exists | Bank calendar **unknown**: portal review "up to 5 days" (ICICI), NDA, UAT, security checks. A vendor claims 3–4 weeks for integration. **Plan 2–3 months elapsed; treat as an estimate.** |
| **4. Second source** | Kotak or Axis direct, **or** Open (if the commercial terms are acceptable) | 2–3 eng-weeks each | Bank/partner calendar |
| **5. Payments** | Only after the approvals/workflow feature exists. RazorpayX Partner OAuth or bank payment APIs; beneficiary management; status tracking feeds back into matching | 4–6 eng-weeks | Partner agreement; higher-risk review |
| AA | No build. Revisit trigger: a confirmed live non-individual current-account flow **and** a regulated partner FIU with a compatible purpose | — | — |

**Total to a live pilot on one bank:** about 6 weeks (Phase 1, which ships value by itself) plus about
6 weeks (Phases 2–3 development), with the bank's calendar as the real critical path.

---

## 9. Why ICICI first (and what could change that)

- **For ICICI:**
  - The most public ERP programme: self-registration on the portal, a documented UAT path, and a
    partner list that already includes very small accounting ISVs (Ledgers, AlignBooks, Profitnx).
  - Customers already know the CIB "Connected Banking Approvals" step from Tally and Zoho.
  - One integration later covers feeds, balance, payments and collections.
- **Against ICICI:** ICICI's BankPlugin (₹1,800 + GST/yr) competes for the same desktop-Tally users.
  There is no public partner commercial model, and the feed excludes "today".
- **What would flip the choice:** the owner's customer base. If most of the reseller's customers bank
  with Kotak, Axis or SBI, pilot that bank instead; the platform does not change. If the banks are slow
  but Open signs quickly, Open first gives the widest coverage for one adapter, at the price of an
  intermediary and unknown fees.

---

## 10. Open questions the owner must answer

1. **Which banks and entity types do your customers actually have?** Count your Tally customer base
   by bank (top 3) and by proprietorship vs partnership/LLP/Pvt Ltd. This picks the pilot bank and
   settles AA once and for all.
2. Does Comhard (as a Tally partner) already have a relationship manager at ICICI, Kotak or Axis, or
   with their connected-banking teams? Does Comhard hold a current account at the pilot bank for UAT?
3. Who signs the NDA and partner agreement for Comhard Technologies Pvt Ltd? Who is the named security
   contact and DPO/grievance officer the bank and DPDP will ask for?
4. Budget for a VAPT (and possibly ISO 27001 later) before the pilot: yes or no?
5. **Is the VM hosted in India?**
6. Pricing: are feeds a paid add-on (passing bank fees through) or included in Pro/Business?
7. Do you want payments at all in the first year? They are a separate risk and support class. Feeds
   alone is a defensible v1.
8. Is a lending/NBFC partnership on the roadmap? It is the only realistic AA route.
9. Is it acceptable that live feeds need a cloud account (desktop-only installs get file import only)?
10. XLSX parsing needs a library. The npm `xlsx` package is stale; SheetJS now ships from its own CDN,
    and older npm builds have published CVEs. `exceljs` is an alternative. Approve one dependency, or
    ship CSV-first and ask users to "Save as CSV" in v1.
11. Can you get 2–3 anonymised real statements per bank (with customer consent) as parser fixtures?
    Without them, the presets are guesses.
12. Retention: keep feed lines for the full statutory books period (8 years), or prune feed-only lines
    that never matched after N years?

---

## Sources

Competitors and banks:
- TallyPrime: [connected banking help](https://help.tallysolutions.com/tally-prime/banking-utilities/connected-banking-feature/),
  [supported banks / formats](https://help.tallysolutions.com/tally-prime/banking-utilities/list-of-banks-supported-by-tallyprime-for-e-payments-auto-brs-and-cheque-formats/),
  [6.0 marketing](https://tallysolutions.com/tally/tallyprime-6-0-connected-banking-automating-banking-and-accounting),
  [6.0 release notes](https://help.tallysolutions.com/release-notes-tallyprime-6/),
  [SBI connection](https://help.tallysolutions.com/connected-banking-feature/connection-sbi/),
  [matching rules](https://tallysolutions.com/ssa/tally/transforming-banking-accounting-for-smes/)
- Zoho: [ICICI](https://www.zoho.com/en-in/erp/help/vendor-payments/icici.html), [Kotak](https://www.zoho.com/in/books/help/online-payments/kotak.html),
  [Axis](https://www.zoho.com/en-in/erp/help/vendor-payments/axis.html), [YES Bank](https://www.zoho.com/en-in/erp/help/vendor-payments/yes-bank.html)
- ICICI: [partners](https://www.icici.bank.in/business-banking/cms/connected-banking/connected-banking-partners),
  [API suite](https://www.icici.bank.in/corporate/transact/online-platform/corporate-api-suite),
  [how to apply](https://www.icici.bank.in/blogs/api-banking/how-to-apply-for-api-banking-services),
  [BankPlugin](https://www.bankplugin.com/), [integrator flow (Mysa)](https://support.mysa.io/en-US/mysa/article/ufoyIJ_T-setting-up-your-icici-bank-account-on-mysa)
- Other banks: [Kotak](https://api.kotak.bank.in/connectedbanking), [Axis neo API suite](https://axisbank.com/corporate/neo-by-axis-bank/neo-api-suite),
  [HDFC developer FAQ](https://developer.hdfc.bank.in/faqs), [IndusInd–Marg](https://www.indusind.bank.in/in/en/personal/other/indusInd-bank-and-marg-erp.html)
- Open: [multi-bank](https://open.money/product/multi-bank), [digital linking](https://docs.enterprise-connected-banking.open.money/docs/digital-linking),
  [HDFC linking](https://docs.enterprise-connected-banking.open.money/docs/hdfc-account-linking-flow)
- Others: [Busy FAQ](https://busy.in/faqs/accounting-reports/bank-reconciliation/2/), [Vyapar](https://vyaparapp.in/free/small-business-accounting-software/bank-reconciliation),
  [QuickBooks exit](https://thepaypers.com/fintech/news/quickbooks-will-exit-the-indian-market-in-2023), [Suvit listing](https://www.techjockey.com/detail/suvit)

Account Aggregator:
- [Sahamati FAQ](https://sahamati.org.in/faq/), [purpose codes](https://sahamati.org.in/purpose-codes/),
  [CT008 template](https://sahamati.org.in/aa-consent-template-library/consent-template-id-ct008-personal-finance-management/),
  [ecosystem dashboard](https://sahamati.org.in/aa-dashboard/), [GST-as-FIP deck](https://sahamati.org.in/wp-content/uploads/2022/12/GST-as-FIP-on-AA_Deep-Dive_Session.pdf)
- [RBI SRO press release](https://rbi.org.in/Scripts/BS_PressReleaseDisplay.aspx?prid=62870), [ReBIT deposit schema](https://specifications.rebit.org.in/api_schema/account_aggregator/documentation/deposit.html)
- [Setu AA pricing](https://setu-aa.com/pricing-policy), [Setu FIU go-live](https://docs.setu.co/data/account-aggregator/licenses-and-go-live/go-live),
  [Anumati pricing](https://www.anumati.co.in/pricing/), [OneMoney summary](https://www.onemoney.in/docs/resources/FinTech%20Group%20Very%20Brief%20Business%20Summary.pdf),
  [Estabizz on pricing](https://www.estabizz.com/account-aggregators-price-wars/), [CASParser 2026](https://casparser.in/blog/state-of-account-aggregator-2026/)

Payments:
- RazorpayX: [pricing](https://razorpay.com/x/pricing/), [payouts](https://razorpay.com/x/payouts/), [account types](https://razorpay.com/docs/x/account-types/),
  [Partner OAuth](https://razorpay.com/docs/partners/technology-partners/onboard-businesses/integrate-oauth)
- [Cashfree Payouts](https://www.cashfree.com/en/payouts/)

Regulation, security and formats:
- [DPDP Rules 2025 (AZB)](https://www.azbpartners.com/bank/update-indias-digital-personal-data-protection-framework-comes-into-effect/),
  [CERT-In 2022 directions](https://psalegal.com/new-cert-in-directions-overview-and-implications/),
  [RBI IT outsourcing 2023](https://vinodkothari.com/2023/04/rbi-regulates-outsourcing-of-it-services-by-financial-entities/)
- [MT940 vs CAMT in India (vendor, unverified)](https://www.terra-insight.com/insights/mt940-vs-camt-053-vs-mt942-india/)

**Could not verify:**
- Whether non-individual current accounts are live on AA.
- Any bank's partner fees or timelines.
- Open's bank coverage and pricing.
- Zoho's Yodlee coverage in India.
- Busy's statement-import release.
- Per-bank AA success rates.
- Which RazorpayX partner banks are current (the pages disagree).
