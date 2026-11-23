# e-Invoicing (IRP / GSP) integration: research and decision

> **Ask (consultants, 2026-10-15):** "There should be an option for e-invoicing integration, for
> which we have to check GSP."
> **Researched:** 2026-10-15, against this repo at `main` (7927626 + working tree) and the web.
> **Evidence tags:** **[V]** checked against an official or primary source (GSTN, NIC, the vendor's
> own docs). **[S]** secondary sources only (blogs, vendor marketing, reseller pages). **[U]** could
> not verify. Treat [S] and [U] as things to confirm before relying on them.

---

## 1. TL;DR

1. **Recommendation:** build e-invoicing as a **server-side relay (we act as the ASP)** behind a
   **provider-adapter interface** that does not depend on any one GSP. Buy IRP access from **one
   partner**, chosen by RFQ between (a) a GSP that routes to NIC1/NIC2 (Masters India, Vayana,
   Adaequare, Webtel, GSTZen, ClearTax...) and (b) a private IRP's own direct-integrator programme
   (IRIS IRP, Clear IRP). Do not try to become a GSP. Do not go direct to NIC, which requires
   AATO of ₹100 Cr or more.
2. **We can do a lot without any contract, and should start now:** a correct INV-01 v1.1 payload
   builder and pre-validator, a portal-assisted flow (bulk JSON export, upload on the portal,
   import the results), real QR printing, and the whole server adapter, queue and idempotency
   layer tested against a mock provider. **One fix is urgent:** the current mock writes a
   fabricated IRN and Ack No onto real vouchers, and the tax-invoice print shows them with nothing
   marking them as mock (§6).
3. **Hard prerequisites before we accept any taxpayer's IRP API password:** HTTPS on the backend
   (today it is HTTP-only) and an encrypted credential vault for each tenant.
4. **Effort:** about **7–11 engineer-weeks** to production for one senior engineer with AI help.
   The calendar time depends on the partner's contract and sandbox. Variable cost at public rates
   is about **₹0.18–₹2.40 per IRN**, but vendors set annual minimums, so negotiate one pooled
   ASP minimum across all our tenants.

---

## 2. What the ask really needs

"E-invoicing integration" means: a B2B, SEZ, export or deemed-export **tax invoice, credit note
or debit note** must be registered on a government-notified **Invoice Registration Portal (IRP)**
before it counts as a valid tax invoice. The IRP returns an **IRN**, an **Ack No/Date** and a
**signed QR**, and the QR must be printed on the invoice. For taxpayers who are required to
e-invoice, an invoice without an IRN is not a valid invoice (Rule 48(5) CGST Rules) [S]. That
exposes the seller to penalty and the buyer to ITC risk. The data then flows on to GSTR-1 and the
e-way bill system.

### User stories

| # | As a... | I want... | Notes / acceptance |
|---|---|---|---|
| U1 | billing clerk, keyboard only | to save a Sales invoice with **Ctrl+A** and have the IRN generated in the background, with the status visible on the voucher | Never steal focus. A status chip reads "e-Invoice: queued → IRN ✓ Ack 1123…". No new meaning for any existing key (superset rule, CLAUDE.md). |
| U2 | clerk | the invoice to **print with the QR, IRN and Ack** without reprinting by hand | print-after-save waits briefly for the IRN, then prints a "pending: not a valid tax invoice" banner if it has not arrived |
| U3 | clerk working offline | to keep billing offline and have pending e-invoices go out automatically when back online | the 30-day reporting window (AATO ≥ ₹10 Cr) is shown as an ageing warning |
| U4 | clerk | when the IRP rejects an invoice, to see **which field in which master** is wrong and jump to it with Enter | e.g. "Party 'Sharma Traders': PIN code missing → Enter opens the ledger at PIN" |
| U5 | accountant | to **bulk-generate** the day's pending invoices from the e-Invoice register (Space to select, one key to send) | progress row by row; failures do not block successes |
| U6 | accountant | to **cancel an IRN within 24 h** with a reason code, and have the books follow | after 24 h the app steers to a credit note (the IRP will not cancel) |
| U7 | owner | to enter the IRP API username and password **once**, have them stored encrypted on the server, and never on a laptop | "Test connection" runs an auth call |
| U8 | owner, on mobile | a **notification** when IRNs fail or invoices near the 30-day limit, and optionally to **approve an IRN cancellation** from the phone | links to the approvals-workflow ask, but the 24 h window is tight, so approval must be optional |
| U9 | owner without the cloud plan (desktop only) | a way to comply anyway | portal-assisted path: export JSON → upload at einvoice1 → import the results file |
| U10 | Tally switcher | the same mental model as TallyPrime | TallyPrime generates on save and supports offline JSON export and bulk generation ([Tally help](https://help.tallysolutions.com/tally-prime/e-invoice-tallyprime/one-time-setup-for-e-invoicing/)) [V] |
| U11 | dispatcher | an **e-way bill by IRN** from the same register (W) | needs transport details that we do not capture today (§8.12) |

**Out of scope here:** B2C dynamic QR (a separate, non-IRP mandate for very large taxpayers), the
standalone e-way bill API for non-invoice movements (needs its own research), and GSTR-1 filing
through a GSP.

---

## 3. Regulatory facts as of 2026-10-15

| Topic | Position | Tag / source |
|---|---|---|
| Who must e-invoice | Aggregate turnover **> ₹5 Cr in any FY since 2017-18** (Notification 10/2023-CT, w.e.f. 1 Aug 2023). Measured at PAN level. Once crossed, the mandate stays even if turnover falls. **No lower threshold had been notified as of mid-2026.** Talk of ₹2–3 Cr is unnotified. | [S] [gimbooks 2026](https://www.gimbooks.com/blog/5-crore-e-invoice-turnover-rule-2026/amp/), [busy.in](https://busy.in/gst/e-invoicing-turnover-limit-vs-gst-threshold-whats-the-difference-and-why-it-matters/) |
| Exempt entities | Insurers, banks/financial institutions/NBFCs, GTA (road), passenger transport, multiplex admission, **SEZ units** (not developers), government departments. The exemption covers the whole entity (Circular 186/18/2022). | [S] [taxguru](https://taxguru.in/goods-and-service-tax/e-invoicing-system-gst-turnover-exceeds-rs-10-cr-w-e-f-01-10-2022.html), [CBIC circular 186](https://cbic-gst.gov.in/pdf/circular-186.pdf) |
| Documents | INV, CRN, DBN. Supply types B2B, SEZWP, SEZWOP, EXPWP, EXPWOP, DEXP. **B2C is not handled by the IRP.** | [V] [NIC Generate IRN](https://einv-apisandbox.nic.in/version1.03/generate-irn.html) |
| 30-day reporting limit | For **AATO ≥ ₹10 Cr from 1 Apr 2025**, the IRP rejects a document older than 30 days (lowered from ₹100 Cr). Applies to INV, CRN and DBN. It is a portal validation, not a statutory deadline. | [V] [GSTN advisory PDF (mahagst mirror)](https://www.mahagst.gov.in/public/uploads/gstnadvisory/1760531580_293%20Advisory%20Time%20Limit%20for%20Reporting%20e%20Invoice%20on%20the%20IRP%20Portal%20%20Lowering%20of%20Threshold%20to%20AATO%2010%20Crores%20and%20Above.pdf), [taxmann](https://www.taxmann.com/post/blog/time-limit-of-30-days-for-reporting-e-invoice-on-irp-portal-for-taxpayers-with-aato-of-10-crores-and-above-gstn-update/) |
| Validity | An invoice that should carry an IRN but does not is **not a valid invoice** (Rule 48(5)). The October 2020 penalty waiver is long gone. | [S] [taxguru](https://taxguru.in/goods-and-service-tax/e-invoicing-system-gst-turnover-exceeds-rs-10-cr-w-e-f-01-10-2022.html) |
| HSN digits | **6-digit HSN minimum** on e-invoices for AATO ≥ ₹5 Cr since 15 Dec 2023, 4-digit below that. Where no valid 6-digit code exists, use an 8-digit code. | [S] [taxo.online](https://taxo.online/latest-news/22-03-2023-advisory-issued-on-hsn-code-reporting-in-e-invoice-on-irps-portal-in-gst-gstn/), [ginesys](https://www.ginesys.in/blog/6-digit-hsn-code-mandate-in-e-invoices-from-december-2023) |
| Document number case | Since **1 Jun 2025** the IRP upper-cases the document number before generating the IRN, so "inv/1" and "INV/1" are the **same** document. | [S] [taxmann](https://www.taxmann.com/post/blog/gstn-issued-advisory-on-case-insensitivity-in-irn-generation/), [taxguru](https://taxguru.in/goods-and-service-tax/gst-advisory-case-sensitivity-irn-generation.html) |
| 2026 API changes | Mandatory **Ship-To GSTIN** validation in the e-invoice and EWB-by-IRN APIs, plus a new **EWB Closure API** (planned for 15 Jun, then 1 Aug 2026), were **put on hold on 29 Jul 2026 (GSTN Advisory 668)**, with no new date. Keep Ship-To GSTIN optional but carry it through the data model. | [S] [taxupdate.in](https://taxupdate.in/gst/831/gstn-advisory-668-eway-bill-enhancements-on-hold-ship-to-gstin-closure-august-2026/) |
| MFA | OTP-based MFA is mandatory on e-invoice and EWB **portal logins** for all taxpayers from 1 Apr 2025. **Whether MFA touches API users could not be verified.** The NIC auth API is username + password + app key. | [S] [taxmann](https://taxmann.com/post/blog/multi-factor-authentication-on-e-invoice-and-e-way-bill-portals-will-be-mandatory-for-all-taxpayers-users-gstn); API impact [U] |
| E-way bill | EWB cannot be generated for a document **older than 180 days** (from 1 Jan 2025, including EWB by IRN). Extension is capped at 360 days from generation. | [S] [cleartax](https://cleartax.in/s/time-limit-for-e-way-bill-generation) |
| Record retention | The IRP lets you **download e-invoice JSON only for about 6 months**. Books must be kept for 72 months (s.36 CGST Act), so **we must keep the signed invoice and QR ourselves.** | [S] [taxheal](https://www.taxheal.com/?p=94652), [sovos](https://sovos.com/regulatory-updates/vat/india-new-e-invoice-download-functionality/) |
| Cost of the IRP itself | **All core e-invoice APIs are free of cost on all IRPs.** GSPs charge for their gateway and value-added services. | [V] [GSTN "API Details of all 6 IRPs" PDF](https://tutorial.gst.gov.in/downloads/news/e-invoice_api_integration_guide_irps.pdf) |

---

## 4. IRP architecture

### 4.1 The six notified IRPs (GSTN, as of 29 Nov 2023; still six per 2026 sources)

| IRP | Portal | Direct-API eligibility (GSTN's own table) |
|---|---|---|
| NIC1 | einvoice1.gst.gov.in | "Direct API" offered only if turnover **≥ ₹100 Cr**; otherwise through a GSP |
| NIC2 | einvoice2.gst.gov.in | same as NIC1. NIC1 and NIC2 interoperate ("criss-cross") through an `irp` header |
| Cygnet IRP | einvoice3.gst.gov.in | same ₹100 Cr rule as NIC |
| Clear IRP (Defmacro/ClearTax) | einvoice4.gst.gov.in | "All taxpayers eligible for e-Invoicing can access the ClearIRP APIs" |
| EY IRP | einvoice5.gst.gov.in | "Turnover is verified in Sandbox upon Direct API user creation" |
| IRIS IRP (now Sovos-owned) | einvoice6.gst.gov.in | "Open to direct integration for all B2B and SEZ taxpayers with e-invoicing enabled"; offers an **API Integrator** sign-up role for solution providers |

Sources: [V] [GSTN PDF](https://tutorial.gst.gov.in/downloads/news/e-invoice_api_integration_guide_irps.pdf).
[V] [IRIS API integrator registration](https://einvoice6.gst.gov.in/content/kb/api-integrator-registration/).
[S] IRIS's GST/ASP business was bought by Sovos in 2025
([BusinessWire](https://www.businesswire.com/news/home/20250812153263/en/Sovos-Expands-Global-eInvoicing-and-Tax-Compliance-Coverage-with-Acquisition-of-IRIS-APAC-Tax-Technology-Business)).
No seventh IRP (e.g. Alankit) could be confirmed [U]. An IRN registered on one IRP blocks the same
document on another: ClearTax documents an error **2295** for an IRN already generated through
another IRP [S] ([cleartax](https://cleartax.in/s/gst-e-invoice-error-code-2150-wrong-vehicle-number)).
**The ₹100 Cr rule is an API-access rule, not the mandate threshold.** Older blogs cite ₹500 Cr
or ₹10 Cr. GSTN's own November 2023 table is the authority used here.

### 4.2 API flow (NIC spec; GSPs mirror it)

All facts below are [V] from the [NIC API sandbox docs](https://einv-apisandbox.nic.in/), except where tagged.

1. **Auth** `POST {base}/v1.04/auth`. Headers: `client_id`, `client_secret` (issued to the direct
   integrator or the GSP), `Gstin`. The body `Data` is the JSON `{UserName, Password, AppKey,
   ForceRefreshAccessToken}`, Base64-encoded and encrypted with the **e-invoice public key**
   ([auth doc](https://einv-apisandbox.nic.in/version1.04/authentication.html)). RSA is the usual
   choice, but the padding is not stated on the page [U]. `AppKey` is 32 random bytes. The
   response carries `AuthToken`, a **`Sek`** (session key, AES-256/ECB/PKCS7, encrypted with
   the AppKey) and `TokenExpiry`. The token is **valid 360 min (60 min on sandbox)**.
   Re-authenticating within that window returns the same token; `ForceRefreshAccessToken` gets
   a new one in the last 10 min.
2. **Generate IRN** `POST {base}/api/Invoice`. Headers: `client_id`, `client_secret`, `Gstin`,
   `user_name`, `AuthToken`. Body: `{Data: Base64(AES_SEK(json))}`. Decrypted response:
   `AckNo`, `AckDt`, `Irn`, **`SignedInvoice`** (JWS, SHA256RSA), **`SignedQRCode`** (JWS),
   `Status` (`ACT`/`CNL`), and optionally `EwbNo`/`EwbDt`/`EwbValidTill`
   ([doc](https://einv-apisandbox.nic.in/version1.03/generate-irn.html)). **The IRN is the SHA-256
   of GSTIN, document number, document type and the FY of the document date.** The IRP refuses a
   second IRN for the same (supplier, type, number, FY) and **refuses to regenerate a cancelled
   IRN**. On a duplicate the error is **2150** [S]. Whether NIC returns the existing IRN inline
   (often called `DUPIRN` in `InfoDtls`) could not be verified [U]. ClearTax's wrapper does
   return it, as `DUPIRN_CLEARTAX` [V] ([ClearTax doc](https://docs.cleartax.in/cleartax-docs/e-invoicing-api/e-invoicing-api-reference/govt-compatible-apis/generate-irn)).
3. **Get IRN by IRN** `GET /api/Invoice/irn/{irn}` and **by document** `GET /irnbydocdetails?doctype&docnum&docdate`.
   Both work only **within 3 days of generation** ([doc](https://einv-apisandbox.nic.in/version1.03/Get_IRNdetailsbyDocDetails.html)).
   This window drives our idempotency design (§8.5).
4. **Cancel** `POST /api/Cancel` `{Irn, CnlRsn: 1 Duplicate | 2 Data entry mistake | 3 Order cancelled | 4 Others, CnlRem ≤100}`.
   Allowed **within 24 h**, and **blocked while an active e-way bill exists** ([doc](https://einv-apisandbox.nic.in/version1.03/cancel-irn.html)).
5. **EWB by IRN** `POST /api/einvewb/ewaybill` `{Irn, Distance 0–4000 (0 = auto PIN-to-PIN), TransMode, TransId, TransName, TransDocNo/Dt, VehNo, VehType, DispDtls, ExpShipDtls}`.
   Transporter ID alone gives a Part-A-only bill. Not allowed for CRN/DBN or services-only invoices
   ([doc](https://einv-apisandbox.nic.in/version1.03/ewaybill-generation-irn.html)). Cancel EWB and
   get EWB by IRN also exist.
6. **Get GSTIN details** (v1.04) can auto-fill and check party masters.

**Schema in force: INV-01 `Version: "1.1"`.** NIC validation 2 reads "must be 1.1 currently". The
API versions are auth v1.04 and the IRN APIs v1.03. Key rules from the same page [V]:
- `DocDtls.No` is **1–16 characters** matching `^[a-zA-Z1-9][a-zA-Z0-9/-]{0,15}$`. It **cannot
  start with 0, / or -**.
- `Dt` is `dd/MM/yyyy`.
- `Stcd` is a **numeric state code**. `Pin` is six digits and is checked against the state.
- `RndOffAmt` must be within ±99.99.
- Each tax figure may differ from the computed value by about ₹1 either way.
- An invoice has 1–1000 items, and the payload is at most 2 MB.
- HSN needs at least 4 digits.
- GSTIN prefix must equal the state code (96 for exports).

The sandbox page also says it accepts only document dates from 01/04/2025. That looks like a
sandbox reset [U].

**Signed QR contents** [S]: SellerGstin, BuyerGstin, DocNo, DocTyp, DocDt, TotInvVal, ItemCnt,
MainHsnCode, Irn, IrnDt ([Vayana verify-QR doc](https://docs.enriched-api.vayana.com/routes/enriched/Trade-Verification-Service/Verify%20EInvoice/apis/Verify-Qrcode/verify-qrcode/)).
We print the JWS string as a QR image. We do not re-encode it.

### 4.3 Sandbox access

- The NIC sandbox (einv-apisandbox.nic.in) is for direct-eligible taxpayers and GSPs. An ASP gets
  sandbox access **through its GSP**.
- Clear IRP ([irp-sandbox.clear.in](https://irp-sandbox.clear.in/app/onboarding)) and IRIS IRP
  ([manage sandbox access](https://einvoice6.gst.gov.in/content/kb/manage-sandbox-access/)) run
  their own sandboxes with self sign-up. IRIS production access requires test GSTINs first
  ([production access](https://einvoice6.gst.gov.in/content/kb/manage-production-access/)) [V].
- **Whether a software vendor can get IRIS or Clear sandbox credentials with no taxpayer GSTIN and
  no contract is unverified [U].** It is the cheapest experiment available (§11, Phase 2 spike).

### 4.4 How a GSP/ASP deployment works for the customer

1. The taxpayer logs in at einvoice1 and goes to **API Registration → Create API User → "Through
   GSP"**. They pick the GSP's name and set an API username and password. Tally has users pick
   "Tally (India) Private Ltd" ([Tally](https://help.tallysolutions.com/tally-prime/e-invoice-tallyprime/one-time-setup-for-e-invoicing/)) [V].
   Zoho Books has them pick "Zoho Corporation" ([Zoho](https://www.zoho.com/in/books/help/e-invoicing/set-up.html)) [V].
   Busy has them pick "Webtel Electrosoft" or "Pinnacle Finserv Advisors" ([Busy FAQ](https://busy.in/faqs/taxation/e-invoice-generation/17/)) [V].
2. The taxpayer types that API username and password into the accounting software.
3. The software calls the GSP with the **software's own GSP credentials** plus the **taxpayer's
   API user**. The GSP calls the IRP using its whitelisted gateway.

Because we are not a GSP, **the customer will select our partner's name on the portal, not
"Comsky"**. A Tally switcher must create a **new** API user under our GSP. Whether NIC allows
several API users per GSTIN for different GSPs at once is likely but unverified [U]. Zoho
reuses an existing e-way bill API profile when one exists.

---

## 5. Landscape

### 5.1 What competitors ship

| Product | Route | Notes |
|---|---|---|
| **TallyPrime** | **Built in; Tally (India) Pvt Ltd is itself a GSP** | Generates on save, bulk, and an offline route: export JSON, upload on the portal, then fetch the IRN back with "Get IRN Info". That fetch still needs internet and Tally's GSP ([offline help](https://help.tallysolutions.com/generate-e-invoice-offline-tallyhelp/)). Needs active **TSS**: Silver ₹4,500/yr, Gold ₹13,500/yr + GST ([Tally TCO](https://tallysolutions.com/business-guides/tallyprime-total-cost-of-ownership-india/), [connected services](https://help.tallysolutions.com/connected-services-in-tallyprime/)) [V/S]. On a duplicate-IRN error it uses Get IRN. After 72 h it advises a new number or a credit note ([help](https://help.tallysolutions.com/duplicate-irn-error/)) [V]. |
| **Zoho Books** | **Built in; Zoho Corporation is a GSP** | Connect once with the API user, push and cancel, role-restricted ([help](https://www.zoho.com/in/books/help/e-invoicing/set-up.html)) [V]. Listed from the Standard plan, ₹899/mo [S] ([patron](https://www.patronaccounting.com/blog/zoho-books-pricing-india-2026)). |
| **Busy** | **Through partner GSPs** (Webtel, Pinnacle) | Needs party PIN/station and item HSN/UQC in the masters. Offers "generate after save" per voucher series. Prints QR, IRN and Ack ([FAQ](https://busy.in/faqs/taxation/e-invoice-generation/17/)) [V]. |
| **Marg** | Through GSPs (Masters India lists Marg among integrations) | Marg's own default GSP was not found [U] ([Masters India](https://www.mastersindia.co/blog/e-invoicing-api-integration-modes/)) [S] |
| **Vyapar** | Claims GSP-API IRN generation and cancellation | Which GSP it uses is not disclosed [S] ([vyapar](https://vyaparapp.in/free/invoicing-software/e-invoicing)) |
| **QuickBooks / Xero** | QuickBooks India shut down in 2023, with no e-invoicing. No Xero India e-invoice support found. | [S] ([patron](https://www.patronaccounting.com/blog/zoho-books-vs-quickbooks-india-2026)) |

**Takeaway:** the two products we most need to match (Tally, Zoho) are **their own GSPs**, so
e-invoicing costs them nothing extra per call. Mid-market tools (Busy, Marg, Vyapar) use partner
GSPs. That is the model open to us.

### 5.2 GSPs and public pricing

The core IRP APIs are free [V]. What you pay for is the GSP gateway, plus storage and value-added
services.

| Provider | Role | Public price found | Tag |
|---|---|---|---|
| GSTZen | GSP/ASP | e-invoice API **18 paise/e-invoice** (no storage; deleted after 10 days; **50,000/yr minimum**) or **50 paise** (7-year storage; 25,000/yr minimum), excluding GST. EWB API 18/35 paise. Document signer ₹15,000/yr. | [V] [pricing](https://gstzen.in/e-invoicing-api-integration-pricing), [EWB](https://gstzen.in/e-way-bill-api-integration-pricing-details) |
| Adaequare (Tera) | GSP | 0.25 IRN-only / 0.40 IRN+EWB / 1.50 with GSTR-1. Currency and unit not stated. Their TLS certificate had expired when fetched. | [S/U] [ugsp.adaequare.com](https://ugsp.adaequare.com/e-invoice-api) |
| GSTINAPI | API reseller | 3 credits per successful document at ₹0.40–0.80 per credit, so about **₹1.20–2.40/doc**. Failures are free. | [S] [gstinapi](https://www.gstinapi.in/pricing) |
| Masters India | GSP | "Start small and pay as you grow". No rate card; bulk IRN API available. | [S] [mastersindia](https://www.mastersindia.co/e-invoicing-api/), [docs](https://docs.mastersindia.co/einvoicing/llms.txt) |
| ClearTax (Defmacro) | GSP + ASP + **Clear IRP** | No rate card. Its "govt-compatible" API takes **plain JSON** (no SEK handling for us) and returns the existing IRN on duplicate. | [V] docs, price [U] |
| IRIS GST (Sovos) | GSP + **IRIS IRP** | "Subscription / enterprise custom" | [S] |
| Cygnet | GSP + **Cygnet IRP** | none found | [U] |
| Vayana | GSP (runs an ASP onboarding portal) | none found | [U] [asponboarding.gsp.vayana.com](https://asponboarding.gsp.vayana.com/) |
| Webtel, Pinnacle | GSPs (Busy's partners) | none found | [U] |

Building a direct NIC integration is estimated at **₹5–7 lakh** for an enterprise
([cleartax](https://cleartax.in/s/e-invoicing-api-integration-modes), [Masters India](https://www.mastersindia.co/blog/how-can-you-decide-which-type-of-e-invoicing-integration-to-opt-for/)) [S].
That figure does not apply to us, because NIC direct needs ₹100 Cr AATO.

**Worked cost example (illustrative):** a customer with 500 B2B invoices a month makes 6,000 IRNs
a year. At 18–50 paise that is **₹1,080–3,000/yr**. But if a 25k–50k document **minimum applies
per GSTIN**, the floor becomes ₹4,500–12,500 per GSTIN, which **costs more than Tally's TSS**.
The make-or-break term in the RFQ is an **ASP-pooled minimum across all our tenants**.

### 5.3 Becoming a GSP ourselves

Not realistic. GSTN selected GSPs in batches between 2016 and 2019. The criteria included paid-up
capital of at least ₹5 Cr and average turnover of at least ₹10 Cr
([gstzen](https://gstzen.in/a/selection-of-gsp-3rd-batch.html)) [S]. **No open application window
was found for 2025–26 [U].**

---

## 6. Where the codebase is today (audit)

> Line numbers are from the 2026-10-15 working tree. **Another session was editing these files
> during this research**: `einvoice.tsx` changed between two reads. Treat the symbol names as
> authoritative and the line numbers as approximate (~).

What exists:
- The `Einvoice` register screen `src/ui/gst/einvoice.tsx` (G = mock IRN, W = mock EWB, J = JSON
  export), reachable from the menu, palette and router.
- `Company.gstSetup.einvoiceFrom` (`src/types.ts` ~L80), with a GST Validation rule "past
  e-invoice date with no IRN" (`src/core/gstCheck.ts` ~L113).
- IRN fields on the voucher (`Voucher.irn/ackNo/ackDate/ewayBillNo`, `src/types.ts` ~L759),
  printed by `src/ui/print/invoiceModel.ts` (~L633-636).
- A guard that stops an e-invoiced voucher being marked optional (`src/ui/selection.ts` ~L167).
- The backend has none of this. `ARCHITECTURE.md` §8 reserves an "egress proxy with KMS-held
  credentials" for IRP calls, and §9 reserves an `irp` queue.

**Defects and gaps this work must fix, most serious first:**

1. **Fabricated IRNs print as real (compliance hazard).** `generateIrn` (`einvoice.tsx` ~L193)
   writes a hash-derived 64-hex `irn` and a fake `ackNo` onto the **real voucher**. The
   tax-invoice print then shows "IRN / Ack. no." (`invoiceModel.ts` ~L634-635) with **no mock
   marker**. A customer who presses G gets a printed tax invoice carrying an IRN the government
   never issued. **Fix now** (Phase 0): gate mock generation to the demo/sample company or a dev
   flag, or stamp "MOCK, NOT REGISTERED" on print and keep it out of exports.
2. **Altering an e-invoiced voucher erases its IRN.** The save path in `VoucherScreen.tsx`
   (`const v: Voucher = {`, ~L2266) rebuilds the voucher field by field. It carries
   `exportInfo: editing?.exportInfo` through an alteration (~L2285), but not
   `irn/ackNo/ackDate/ewayBillNo`. Any edit silently drops them. The structural fix is to move
   e-invoice state off the voucher into a server-owned collection (§8.6).
3. **IRN state on a full-document, last-write-wins voucher cannot survive sync.** Vouchers sync as
   whole documents. A stale offline edit from another device overwrites a freshly attached IRN
   (ARCHITECTURE §6.3; the client still sends v1 full-doc puts).
4. **The payload is still not INV-01-valid.** The in-flight edit already switched `Stcd`/`Pos` to
   state **codes** (via `stateCodeOf`/`posCodeOf`) and derives `SupTyp` SEZWP/SEZWOP/DEXP from
   `supplyKind`. What remains:
   - `Loc` carries the **state name** where the IRP wants the locality.
   - Buyer `Pin` is missing, because **`Ledger` has no pincode or city field** (`src/types.ts`,
     `interface Ledger` ~L196).
   - `Company` has no city.
   - `ItemList` is built only from inventory lines, so **service and accounting-mode invoices
     produce an empty item list**. `Ledger.hsn/gstRate` exist and should be used.
   - `IsServc` is hard-coded "N", `Unit` (UQC; `Unit.uqc` exists) is omitted, and discounts,
     other charges and round-off are not mapped.
   - Eligibility still means "party has a GSTIN" (`eligible:` ~L188), so **exports (Overseas
     parties have no GSTIN; buyer `Stcd` must be 96) never reach the builder**, and EXPWP/EXPWOP
     are never emitted. `GstPartyType` plus `gstSetup.lutNo` give us WP vs WOP.
   - `DocDtls.Typ` is never DBN.
5. **Number formats the IRP rejects:**
   - `formatVchNo` (`src/state/store.ts` ~L576) zero-pads with `width`, so an unprefixed series
     yields "001", which **validation 8 rejects** (no leading 0).
   - Prefixes and suffixes can contain spaces or other characters outside `[A-Za-z0-9/-]`, or
     exceed 16 characters.
   - The duplicate-number warning compares case-sensitively and per voucher type
     (`VoucherScreen.tsx` ~L2055, `v.numberStr === numberStr`). The IRP treats numbers as case-insensitive and unique per
     **(GSTIN, FY, document type)** across all our voucher types.
6. **Number reuse after delete.** `nextVchNo` is max+1 over **live** vouchers (`store.ts` ~L546-560).
   Delete the latest e-invoiced invoice and its number is re-issued, which the IRP rejects
   (cancelled IRNs cannot be regenerated, and duplicates are refused).
7. **Per-device numbering collides offline.** The server numbering endpoint (`backend/src/sync.js`,
   `/sync/next-number`) exists, but `src/` does not call it. Two offline devices can both issue
   "INV/43". The second IRN request then hits a duplicate, and attaching the first device's IRN to
   the second invoice would be **wrong** (§8.5 fingerprint check).
8. **No QR encoder.** The "QR placeholder" box shows the IRN as text (`einvoice.tsx` ~L318), and
   `package.json` has no QR library.
9. **The parity board flatters us.** `docs/parity-data.json` row `einvoice-irn` is `"status":
   "covered"` beside `"ours": "No IRN/QR anywhere."` (the same row is in `docs/FEATURE-PARITY.md:233`).
   The rule in CLAUDE.md ("a status field and its note must agree") says this should be
   `partial`. *(Not edited here; this research must not touch other files.)*
10. **Backend prerequisites (HANDOFF §2 trust map):**
    - HTTP only, no TLS.
    - Push has no idempotency key (`change_uid` designed, not built).
    - The server is payload-opaque and has no materialised docs, so it cannot build IRP JSON itself.
    - No KMS.
    - In-memory SSE, a single VM, and no Redis/BullMQ.
    - `/sync/push` and REST `POST /v1/:collection` accept **any** table name, so a client or API
      key could forge e-invoice records unless server-owned tables are fenced off.

---

## 7. Options compared

| Option | What it is | Engineering effort | Cost | External dependency | Main risks | Verdict |
|---|---|---|---|---|---|---|
| **A. Portal-assisted** | We build a valid bulk JSON. The user uploads it on einvoice1 with their own login and imports the results file back. | S (8–12 d incl. payload + validator) | ₹0 | None (the user's own portal login) | Manual steps. The layout of the results file must be confirmed from a real download [U]. Fetching later details after the 6-month portal window is impossible. | **Build now.** It is the floor for desktop-only and no-contract customers, and the payload work is reused by every other option. |
| **B. ASP through one GSP (server relay)** | Our backend holds our GSP credentials, stores each tenant's IRP API user, and calls the GSP. | M–L (≈25–35 d incl. A) | GSP fees ≈ ₹0.18–2.40/IRN plus minimums; partner onboarding | GSP contract, sandbox, test certification | Vendor lock-in (mitigated by the adapter). Minimum-commitment economics. GSP outage (mitigate with NIC1/NIC2 failover via the GSP). | **Recommended production path.** |
| **C. Direct to a private IRP as an "API integrator"** (IRIS IRP / Clear IRP) | Same relay, but the IRP is the provider; no GSP in between. | Same as B | Core APIs free [V]; IRIS lists a "Pricing Plan" for extras [U] | That IRP's integrator terms. The customer must register on **that** IRP (Clear requires fresh credentials [V]). | Concentration on one private IRP. Customers already on NIC must onboard again. Commercial terms unverified. | **RFQ in parallel with B.** If the terms are good it is the cheapest; the adapter makes it a swap. |
| **D. Direct to NIC** | Our own client_id with NIC | M | Free | **Only taxpayers with AATO ≥ ₹100 Cr** [V]; credentials are per taxpayer, not per vendor | Not available for our customer base. | Reject (keep an adapter slot for large customers who bring their own NIC direct credentials). |
| **E. Become a GSP** | Apply to GSTN | XL + compliance | Capital and turnover thresholds; audits | No open window found [U] | Years, not weeks | Reject |
| **F. Client-direct** | The browser or Electron app calls the GSP with credentials stored locally | S | as B | as B | **Our GSP secret would ship in every client bundle.** Taxpayer passwords sit in IndexedDB, and we lose audit and single-flight across devices. | Reject (ARCHITECTURE §8 already says IRP calls go through a server egress proxy) |

---

## 8. Recommended architecture on our stack

### 8.1 Principles

1. **Never two IRNs for one supply. Never an IRN attached to the wrong invoice.** The IRP already
   guarantees one IRN per (GSTIN, FY, type, number). Our own risks are (a) a renumbered or
   re-issued document and (b) attaching a duplicate's IRN to a *different* invoice that collided on
   number. We defend with server unique keys, a number lock after IRN, and content-fingerprint
   comparison.
2. **The server owns e-invoice state. The books stay local-first.** The client builds the payload
   (it owns the GST engines: `voucherGstSummary`, `splitByRate`, the masters). The server
   validates it, relays it and records the result, then publishes the result back to every
   device through the ordinary change log.
3. **No IRP or GSP credential ever reaches a client.** Credentials are written once over HTTPS and
   kept encrypted at rest.
4. **Offline is a queue, not an error.**
5. **Every IRP interaction is audited** in an append-only record.

### 8.2 Provider adapter (backend, GSP-agnostic)

```ts
// backend/src/einvoice/provider.d.ts (sketch; the real file would be JS like billing/provider.js)
export interface EinvCreds { gstin: string; apiUser: string; apiPassword: string; irp?: 'NIC1'|'NIC2'|string }
export interface EinvSession { token: string; sek?: Buffer; expiresAt: number } // provider-opaque
export type EinvOutcome =
  | { kind: 'ok'; irn: string; ackNo: string; ackDt: string; signedInvoice: string; signedQr: string;
      status: 'ACT'|'CNL'; ewb?: { no: string; dt: string; validTill?: string } }
  | { kind: 'duplicate'; existing?: Omit<Extract<EinvOutcome,{kind:'ok'}>,'kind'> } // 2150 / DUPIRN*
  | { kind: 'rejected'; errors: EinvError[] }          // validation failure: fix data, resubmit
  | { kind: 'auth'; errors: EinvError[] }              // bad API user/password, token expired
  | { kind: 'transient'; errors: EinvError[]; retryAfterMs?: number } // 5xx, timeout, IRP down
  | { kind: 'unknown' };                                // sent, no usable answer: MUST reconcile
export interface EinvError { code: string; message: string; source: 'IRP'|'GSP'|'NETWORK'; path?: string }

export interface EinvProvider {
  name: string;
  caps: { bulk: boolean; plainJson: boolean; ewbByIrn: boolean; gstinLookup: boolean; irps: string[] };
  authenticate(c: EinvCreds): Promise<EinvSession>;
  generate(s: EinvSession, c: EinvCreds, payload: object): Promise<EinvOutcome>;
  getByDoc(s: EinvSession, c: EinvCreds, d: { typ: 'INV'|'CRN'|'DBN'; no: string; dt: string }): Promise<EinvOutcome>;
  getByIrn(s: EinvSession, c: EinvCreds, irn: string): Promise<EinvOutcome>;
  cancel(s: EinvSession, c: EinvCreds, irn: string, rsn: 1|2|3|4, rem?: string): Promise<EinvOutcome>;
  ewbByIrn?(s: EinvSession, c: EinvCreds, req: object): Promise<EinvOutcome>;
  gstinDetails?(s: EinvSession, c: EinvCreds, gstin: string): Promise<object>;
}
```

Implementations:
- **`mock`** (default, like `backend/src/billing/mock.js`): deterministic. It simulates success,
  2150 duplicates, the 24 h cancel window, timeouts with the request actually registered (the
  dangerous case), auth expiry, and validation errors on specific GSTINs and PINs, so the whole
  pipeline can be tested without credentials.
- **`nicCompatible`**: implements the NIC wire protocol (RSA-wrapped AppKey and password; SEK
  AES-256-ECB; Base64 `Data`). It is reused for any GSP that passes NIC-shaped calls through with
  its own extra headers, and for NIC direct if a large customer brings their own client_id.
- **One concrete partner adapter** chosen by RFQ. ClearTax-style "plain JSON" wrappers bypass the
  SEK layer, which `caps.plainJson` signals.
- Provider selection: `config.einvoiceProvider` per environment (the same pattern as
  `billing/provider.js`), with an optional per-tenant override (e.g. a customer on Clear IRP).

### 8.3 Backend tables (new migration `backend/sql/007_einvoice.sql`)

```sql
-- Credentials: one IRP API user per (tenant, company, GSTIN). Secret envelope-encrypted.
CREATE TABLE einv_credential (
  tenant_id   uuid NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
  cid         text NOT NULL,
  gstin       text NOT NULL,
  provider    text NOT NULL,              -- 'mock' | 'nic' | '<partner>'
  irp         text,                       -- preferred IRP, e.g. NIC1
  api_user    text NOT NULL,
  secret_ct   bytea NOT NULL,             -- AES-256-GCM(dek, password)
  dek_wrapped bytea NOT NULL,             -- dek wrapped by master key (KMS later, ARCH §8)
  key_ver     int  NOT NULL,
  session_ct  bytea,                      -- cached AuthToken+SEK, same envelope; survives restarts
  session_exp timestamptz,
  thirty_day  boolean NOT NULL DEFAULT false,   -- owner-declared AATO ≥ ₹10 Cr
  verified_at timestamptz, created_by uuid, created_at timestamptz DEFAULT now(), updated_at timestamptz,
  PRIMARY KEY (tenant_id, cid, gstin)
);

-- One row per IRP document identity, forever (a cancelled row keeps its number burned).
CREATE TABLE einv_document (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id     uuid NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
  cid           text NOT NULL,
  voucher_id    text NOT NULL,
  seller_gstin  text NOT NULL,
  fy            text NOT NULL,               -- '2026-27', from DocDtls.Dt
  doc_type      text NOT NULL CHECK (doc_type IN ('INV','CRN','DBN')),
  doc_no_norm   text NOT NULL,               -- upper(DocDtls.No): the IRP is case-insensitive
  doc_date      date NOT NULL,
  payload       jsonb NOT NULL,              -- exact INV-01 JSON last submitted
  payload_sha   text NOT NULL,
  fingerprint   text NOT NULL,               -- sha256(buyerGstin|docDt|TotInvVal|AssVal|ItemCnt|mainHsn)
  status        text NOT NULL CHECK (status IN
                 ('queued','sending','unknown','active','rejected','needs_review',
                  'cancel_queued','cancelled')),
  irn text, ack_no text, ack_dt timestamptz, signed_invoice text, signed_qr text,
  ewb_no text, ewb_dt timestamptz, ewb_valid_till timestamptz,
  cancel_rsn smallint, cancel_rem text, cancelled_at timestamptz,
  source        text NOT NULL DEFAULT 'api', -- 'api' | 'portal_import'
  last_errors   jsonb, attempts int NOT NULL DEFAULT 0, next_try_at timestamptz DEFAULT now(),
  idem_key      text NOT NULL,               -- client UUID per user action
  requested_by  uuid, created_at timestamptz DEFAULT now(), updated_at timestamptz DEFAULT now(),
  UNIQUE (tenant_id, idem_key),
  UNIQUE (tenant_id, seller_gstin, fy, doc_type, doc_no_norm)       -- mirrors the IRP's own key
);
CREATE UNIQUE INDEX einv_one_live_per_voucher ON einv_document (tenant_id, cid, voucher_id)
  WHERE status NOT IN ('cancelled','rejected');

-- Append-only audit of every outbound call (secrets and PII-heavy bodies redacted).
CREATE TABLE einv_attempt (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id uuid NOT NULL, document_id uuid REFERENCES einv_document(id),
  op text NOT NULL,                     -- auth|generate|get_by_doc|get_by_irn|cancel|ewb
  provider text NOT NULL, request_sha text, outcome text NOT NULL, codes text[],
  http_status int, latency_ms int, actor uuid, at timestamptz DEFAULT now()
);
-- RLS on all three (tenant_id = current_setting('app.tenant_id')::uuid), ENABLE + FORCE.
-- REVOKE UPDATE, DELETE ON einv_attempt FROM comhard_app;   (append-only, ARCH §8)
```

The worker claims jobs across tenants with one `SECURITY DEFINER` function
(`einv_claim_next()` → `(tenant_id, id)`, `FOR UPDATE SKIP LOCKED`). Everything after the claim
runs inside `withTenant()`, so RLS still scopes every read and write. This avoids the
`006_p4.sql` pattern, where integration tables sit outside RLS.

### 8.4 Endpoints (Bearer JWT, tenant from token; `gst` right checked server-side)

| Method / path | Purpose |
|---|---|
| `PUT /api/einvoice/credentials` `{cid, gstin, provider, apiUser, apiPassword, irp?, thirtyDay?}` | Runs a live `authenticate()` and stores the credential only if that succeeds. **Refused over plain HTTP.** |
| `GET /api/einvoice/credentials?cid` / `DELETE …` | Shows status masked (user, verified_at). The password is never returned. |
| `POST /api/einvoice/generate` `{cid, voucherId, idemKey, payload, fingerprint}` | Validates schema, arithmetic tolerances and the number regex, then upserts `einv_document` → `queued`. Returns `202 {docId, status}`. |
| `POST /api/einvoice/generate-bulk` `{cid, items:[…]}` | Same as above, per item, with partial success. Max N items per call. |
| `POST /api/einvoice/:id/cancel` `{rsn, rem}` | Allowed only if `active`, `ack_dt` < 24 h, and no active EWB. Goes to `cancel_queued`. |
| `POST /api/einvoice/:id/refresh` | Calls get-by-IRN or get-by-doc (only inside the 3-day window). |
| `POST /api/einvoice/:id/ewb` `{transport…}` | EWB by IRN (Phase 5) |
| `POST /api/einvoice/import` `{cid, rows:[{docType, docNo, docDate, irn, ackNo, ackDt, signedQr?}]}` | Portal-assisted import (`source='portal_import'`), with the same fingerprint and uniqueness checks |
| `GET /api/einvoice?cid&status&from&to` | Register feed (the client normally reads the synced collection instead) |

### 8.5 State machine and idempotency (the core of "never two IRNs")

```
queued ──claim──▶ sending ──ok──────────────▶ active ──cancel(≤24h, no EWB)──▶ cancel_queued ──ok──▶ cancelled
   ▲                 │ rejected (validation) ─▶ rejected  (user fixes data → new generate call reuses the row)
   │                 │ transient/auth ───────▶ queued (backoff; auth → pause GSTIN, notify owner)
   │                 │ timeout / crash ──────▶ unknown ──getByDoc (≤3 days)──▶ active | queued | needs_review
   │                 └ duplicate (2150) ─────▶ reconcile ─┬ fingerprint equal ──▶ active (idempotent replay)
   │                                                      └ differs / not found ▶ needs_review
```

- **Single-flight:** the claim moves `queued → sending` atomically. The unique key on (GSTIN, FY,
  type, number) means two devices or tabs can never queue the same document twice. `idem_key`
  makes HTTP retries of the *client's* request safe.
- **Never blind-resend after `unknown`.** A request that was sent but got no answer is resolved
  with **get-by-doc first**. The IRP keeps that open for only **3 days** [V], so `unknown` rows
  page the owner if they are still unresolved after 24 h. Past 3 days they go to
  `needs_review`, and the user checks the portal or uses the import.
- **On duplicate, compare before you attach.** The IRP's existing record (`SignedInvoice` JWS body)
  is decoded and its fingerprint compared with ours. Equal means our earlier lost response:
  attach it. Different means another invoice (another device, a renumbered voucher, an old
  Tally-era document) holds that number. **Never attach.** Mark `needs_review` with the
  explanation "The IRP holds a different invoice under INV/43: renumber this one, or cancel the
  other within 24 h." This is the multi-device collision case from §6.7.
- **Payload freeze:** once a row has ever reached `sending`, its payload can change only after a
  definitive `rejected`. Otherwise the IRP may already hold the earlier content.
- **Number lock (client):** once a row is `active`, the voucher's number, date, party and taxable
  content are read-only (§8.11). Server-side the unique key makes a renumber-and-resubmit
  produce a *new* document identity, and the server flags that as **drift** (§8.6).
- **Backoff:** exponential with jitter. Per-GSTIN concurrency of 2–3. The auth session is cached
  per GSTIN and re-auth happens only on expiry or an auth error (token life 360 min [V]).
- **Queue tech:** a Postgres table polled by an in-process worker, because there is no Redis
  today. It moves to the ARCHITECTURE §9 `irp` BullMQ queue at P4+ scale without changing the
  table contract.

### 8.6 Sync semantics and tenant isolation

- **New client collection `einvoices`** (Dexie v8, `TableName` += `'einvoices'`), keyed by
  `voucherId`: `{id: voucherId, cid, docId, status, irn, ackNo, ackDt, signedQr, ewbNo,
  ewbValidTill, cancelledAt, errors[], fingerprint, updatedAt}`.
  - **Only the server writes it.** After each state change the worker inserts a `change_log` row
    (`device_id='irp'`, `tbl='einvoices'`) inside the same `withTenant` transaction, then calls
    `notifyTenant()`. Every device receives it through the normal `/sync/pull` and SSE nudge.
    No new sync protocol is needed.
  - **`/sync/push` and `POST /v1/:collection` must reject `einvoices` rows** (a server-owned
    table denylist). Otherwise a client or API key could forge "registered" IRNs.
  - `signed_invoice` stays server-side (it is large and sensitive). The client gets `signedQr`,
    which it needs for printing.
- **Fixes defects 2 and 3:** IRN state no longer lives on the last-write-wins voucher, so
  alterations and stale puts cannot erase it. Alt+2 duplicates cannot copy it either.
  `Voucher.irn/ackNo/ackDate/ewayBillNo` become **legacy, read-only** fields: shown only if
  `einvoices` has no row, and print-flagged as unverified.
- **Drift detection:** when a voucher `put` for an `active` document arrives with a fingerprint
  that differs, the server **does not reject it**. Rejecting would break local-first, and
  ARCHITECTURE §6.7 says client gating is a soft control. Instead it records drift on the
  document. The client also computes drift locally, and `gstCheck.ts` raises it in GST
  Validation as "Books differ from the registered e-invoice". The fix is a credit or debit note,
  or a cancel inside 24 h.
- **Isolation:** every table carries `tenant_id` with RLS FORCE. Credentials are per
  (tenant, cid, GSTIN). The worker re-enters `withTenant()` per job. Audit rows carry the tenant.
  Cross-tenant tests are a release gate (ARCHITECTURE STRIDE row 1).

### 8.7 Offline behaviour

| Situation | Behaviour |
|---|---|
| Voucher saved offline, e-invoice applicable | Local derived status **"Awaiting connection"**: applicable, with no `einvoices` row and no submit receipt. A small local `einvPending` outbox in Dexie holds `{voucherId, idemKey, payloadSha}`, so the click is not lost. |
| Back online | The client first pushes the voucher (so the server's change log has it), then calls `generate` per pending item with its **original** `idemKey`. A retried call is idempotent. |
| Print while pending | Prints with a banner: "e-Invoice pending: not a valid tax invoice until registered". The banner never appears on a non-applicable voucher. |
| 30-day limit (owner-declared AATO ≥ ₹10 Cr) | Dashboard and sidebar Today show "e-Invoices pending: N (oldest 27 days)". The banner turns red from day 25. |
| Desktop-only (no cloud plan) | Portal-assisted path (§8.10). Online generation needs the cloud relay, which is stated in the UI. |
| Provisional numbers | If server numbering (ARCH §6.4) is adopted, **a provisional number is never sent to the IRP**; generation waits for `final`. Until then, e-invoicing companies with several devices should use **per-device series prefixes**, or the duplicate check in §8.5 will produce `needs_review` rows. |

### 8.8 Cancellation and the books

- **≤ 24 h, no active EWB:** the user presses **X** on the register (or acts from the voucher),
  picks a reason with keys **1–4** (Duplicate / Data entry mistake / Order cancelled / Others),
  adds an optional remark and confirms. The server cancels and publishes `cancelled`.
- **What happens to the voucher?** The app has **no "cancelled" voucher state** (`interface Voucher`
  in `src/types.ts` has only `optional`/`postDated`). Today's options are to delete it (its
  number is still burned by the `einv_document` row) or to add a Tally-style `cancelled` flag that
  keeps the number with zero effect. **We recommend the flag** (ARCH §6.4 already assumes
  "number retained, marked Cancelled"). Implement it as one exclusion helper used everywhere
  `!v.optional` is checked today (CLAUDE.md "optional vouchers" rule), not as new scattered
  checks. *(Owner decision, §13.)*
- **> 24 h:** the cancel key is disabled with the reason shown. The app offers **"Raise credit
  note"**, pre-filled against the original. This matches Tally's guidance [V].
- **Deleting an e-invoiced voucher is blocked** (client) and flagged as drift (server).
- `nextVchNo` must treat numbers burned in `einvoices` as used, which fixes defect 6.

### 8.9 Bulk generation

The register gains **Space** (toggle select), a select-all-pending key, and **Alt+G** (generate
selected). The selection pattern already exists in `src/ui/selection.ts`. One `generate-bulk`
call queues all of them, and rows update live over SSE. If the provider has `caps.bulk`
(e.g. Masters India's bulk IRN API), the worker batches; otherwise it fans out under the
per-GSTIN concurrency cap. Failures never block successes, and a summary toast reports "38
registered · 2 need fixing".

### 8.10 Pre-validation and error mapping to fields

- **New pure engine `src/core/einvoice.ts`:** `buildInv01(voucher, company, masters) → {payload,
  issues[]}` and `validateInv01(payload) → issues[]`. It mirrors the NIC validations in §4.2:
  - number regex and length, case-insensitive uniqueness;
  - GSTIN vs state code, PIN vs state, numeric `Stcd`;
  - HSN digits (6 for AATO > ₹5 Cr);
  - ±₹1 tax tolerance, `RndOffAmt` ±99.99, 1–1000 items, 2 MB;
  - SupTyp from `GstPartyType` + LUT;
  - IsServc from SAC/HSN `99…` or an explicit flag.
  - Each issue = `{path:'BuyerDtls.Pin', message, fix:{screen:'ledger', id, field:'pincode'}}`.
  - It gets a standalone assertion suite in `scripts/assert/` like the other 15.
- **IRP and GSP errors:** the adapter normalises them to `EinvError`. A table
  `IRP_ERROR_PATHS: Record<code, jsonPath>` maps known codes to the same paths, seeded from the
  chosen GSP's published catalogue (ClearTax publishes one:
  [error codes](https://docs.cleartax.in/cleartax-docs/e-invoicing-api/e-invoicing-api-reference/resources-and-master/error-codes)).
  Unknown codes are shown verbatim.
- **UI:** a failed row shows the first error inline. **Enter** opens the issue list, and Enter on
  an issue drills to the master or voucher field (existing `voucherScreenFor` and master routes).
  The pre-validator runs **at voucher save** for applicable vouchers as a *warning*, never a
  block (superset rule), so most rejections never reach the IRP.
- **Portal-assisted path (Option A):**
  - **J** exports all selected or pending documents as one INV-01 bulk JSON, split under 2 MB
    (the portal limit [S]).
  - **I** imports the portal's downloaded results (IRN, Ack No, Ack Date and, if the file has it,
    the signed QR) into `einvoices`. Without the cloud this is local-only; with it, it goes
    through `POST /api/einvoice/import`.
  - **The exact layout of the results file is unverified [U].** Build the parser from a real
    download (§13).

### 8.11 Voucher and print flow changes

- **Company → GST Setup:**
  - e-Invoice mode: `Off / Portal (manual JSON) / Connected (API)`.
  - Generate: `On save / Ask after save / Manually from register`.
  - "Turnover ≥ ₹10 Cr (30-day limit)" toggle.
  - "Connection…" sub-screen that posts credentials to the server and never stores them locally.
- **Masters (field order frozen; append under existing GST/Address sections):**
  - `Ledger.pincode`, `Ledger.city`, optional ship-to address(es);
  - `Company.city`;
  - `StockItem.isService?` (defaults from HSN `99…`);
  - UQC is required on units used in e-invoiced items.
- **Voucher save (Ctrl+A unchanged):** for an applicable voucher in "On save" mode the client
  enqueues after the local save commits. The status chip in the voucher footer reads
  `e-Invoice: queued | IRN ✓ | needs fixing (Enter) | awaiting connection`. "Ask" mode raises a
  Y/N prompt after save, as TallyPrime does. Any new key goes into the `?` sheet; existing keys
  keep their meaning.
- **Alteration of an `active` e-invoice:** e-invoice-relevant fields are read-only: number, date,
  party, place of supply, item/ledger lines, rates, taxes, totals. Narration and internal fields
  stay editable. Trying to change a locked field offers *Cancel IRN (if ≤24 h)* or *Raise credit
  note*. **PgUp/PgDn alteration walking** (Phase 13) must respect the lock.
- **Print (`invoiceModel.ts`):**
  - Read IRN, Ack and QR from `einvoices` (fall back to legacy fields marked "unverified").
  - Render the **QR from `signedQr`** with a small pinned QR encoder, bundled (no CDN, offline).
  - Title "Tax Invoice" plus an "e-Invoice" marker.
  - Add the pending banner.
  - **Never print a mock IRN** outside the sample company.
  - Print-after-save (`vtypeDef?.printAfterSave` in `VoucherScreen.tsx`, ~L2305) waits up to
    about 8 s for `active` when online.
- **Register (`src/ui/gst/einvoice.tsx`):**
  - Columns gain Status (Pending / Queued / Registered / Needs fixing / Cancelled / Drift), Age
    (days since document date) and IRP.
  - Keys: G generate, Alt+G bulk, X cancel, R refresh, W EWB (Phase 5), J export JSON, I import
    results, Enter drill.
  - Eligibility uses `supplyKind`/`GstPartyType`, not "party has GSTIN".

### 8.12 E-way bill by IRN (Phase 5)

This needs fields we lack:
- transporter GSTIN/ID and name, mode, vehicle number and type, transport document number and date;
- distance (0 = auto);
- dispatch-from and ship-to address.

Put them in an optional **Transport** sub-panel on a new chord (like `BankInstrPanel`), so the
frozen field order is untouched. EWB cancel must come before IRN cancel. The standalone EWB API
(B2C, delivery notes, inward) is a separate integration with its own API user. Keep Ship-To
GSTIN optional (Advisory 668 hold).

### 8.13 Audit trail

- `einv_attempt` is append-only, with the actor, the operation, the outcome and the IRP codes.
  Bodies are redacted.
- Every state change also appears in the client Edit Log via the `einvoices` change-log rows,
  with user `irp`, plus the human who triggered it.
- Cancellations record reason, remark and approver (if the approvals module is used).
- When ARCHITECTURE §8's hash chain lands, these rows are chained like any other.

---

## 9. Security, privacy and legal

- **TLS first.** Today the backend is HTTP-only (HANDOFF §2). Collecting a taxpayer's IRP API
  password over HTTP is unacceptable. The credentials endpoint must **refuse** non-TLS requests
  (check `x-forwarded-proto`, `trustProxy: true`).
- **Credential vault:**
  - AES-256-GCM per row with a random DEK, wrapped by a master key held outside the database
    (systemd `EnvironmentFile`, never in git). Move to KMS when ARCHITECTURE §8 lands.
  - Our own GSP client secret lives in the same place.
  - Rotate on staff change. Write-only API; never logged; masked in admin impersonation views.
  - Admin impersonation (`backend/src/admin.js`) must **not** be able to trigger generate or
    cancel, or that path must be audited and gated.
- **Who may act:** generate needs the `gst` right area (`src/core/access.ts`, `einvoice: 'gst'`), enforced
  server-side through the membership role (owner/admin by default; members by grant). Cancel is
  stricter (owner/admin), optionally with approval.
- **DPDP Act 2023 / DPDP Rules 2025:**
  - The Rules were notified 14 Nov 2025, and most fiduciary duties apply from about May 2027
    ([The Week/PTI](https://www.theweek.in/wire-updates/business/2025/11/14/del148-biz-dpdp-rules-ld-govt.html)) [S].
  - Invoice payloads contain the buyer's legal name and address. For proprietorships that is
    personal data.
  - The tenant is the Data Fiduciary, Comsky is the **Processor**, and the GSP is our
    **sub-processor**. We need a DPA with the GSP covering India-only processing and storage,
    breach notice, retention and deletion.
  - **Prefer the GSP's "no storage" tier** (GSTZen deletes after 10 days [V]) and keep the
    statutory copy ourselves. That minimises copies, and we must keep it anyway for 72 months
    (s.36).
  - List the GSP in our sub-processor list and privacy notice.
  - Crypto-shredding (ARCH §8) must **not** shred `einv_document` content under the statutory
    carve-out.
- **GST law:** we must never present an unregistered or mock IRN as real (defect 1). The
  validity, penalty and ITC consequences fall on our customer, and liability for wrong output
  comes back to us. The ToS must say: IRN status shown is what the IRP returned; pending
  documents are not valid invoices; the customer is responsible for applicability (AATO,
  exemptions) and the 30-day limit.
- **RBI:** not engaged. There is no payment flow. A B2C UPI dynamic QR, if ever built, is
  separate.
- **Branding:** the GSP's name appears on the government portal, not ours. Nothing here touches
  the Tally trademark constraints in `docs/LEGAL-BRIEFING.md`. Use "import from Tally"-style
  nominative wording only in migration help ("create a new API user; your Tally GSP user is
  separate").
- **GSP contract checklist:**
  - per-IRN and per-EWB price, and an ASP-pooled annual minimum;
  - which IRPs they route to (NIC1/NIC2 failover), SLA and uptime, status page;
  - India data residency, storage and retention options, DPA and sub-processors;
  - sandbox access and certification steps, rate limits;
  - whether they accept plain JSON or require SEK pass-through;
  - bulk API;
  - support for e-commerce operator, SEZ and export cases;
  - termination and data return.

---

## 10. MVP without credentials, and what needs a contract

**MVP without credentials: YES**, and most of the engineering fits in it:

| Can build and ship without any external credential | Needs a partner contract, credentials or a real account |
|---|---|
| Phase 0 safety fix (mock IRN) and the parity-row correction | Any **real** IRN, sandbox or production |
| Master fields, number-format guards, burned numbers, `cancelled` decision | The partner's actual auth headers, wire quirks and error catalogue |
| `src/core/einvoice.ts` builder + validator + assertion suite | Sandbox **test-case certification** with the GSP or IRP |
| Portal-assisted bulk JSON export (the customer uses **their own** portal login) | Our **GSP client_id/secret** (ASP onboarding) |
| Results import (**needs one real results file from a customer's portal** to fix the layout) | Each customer's **IRP API user** created under our GSP |
| QR rendering from a signed-QR string, and print changes | Production go-live, SLA, DPA |
| Backend tables, endpoints, worker, state machine, idempotency, `einvoices` sync, drift, audit, all exercised by the **mock provider** | Confirmation that IRIS or Clear sandbox sign-up works for a vendor without a taxpayer GSTIN [U] |
| `nicCompatible` crypto layer unit-tested on its own (round-trip, key sizes) | Verifying the IRP public key used to check signed QR/invoice JWS (NIC publishes keys; rotation [S]) |

**What the owner must procure:**
1. A **GSP or IRP-integrator agreement** (RFQ: Masters India, Vayana, Adaequare, Webtel, GSTZen,
   ClearTax, and the IRIS/Clear IRP integrator programmes). Ask for an **ASP-pooled minimum**,
   not one per GSTIN. Budget about ₹0.18–₹2.40 per IRN at public rates (§5.2); expect a minimum
   commitment.
2. **Sandbox credentials** from that partner, with test GSTINs.
3. **HTTPS** on books.comhard.in:1036, or wherever the API moves (HANDOFF §2: a DNS-01
   certificate needs a comhard.in DNS API token, or the hosting provider routes 443).
4. A **pilot customer** with AATO > ₹5 Cr who will (a) send a real portal bulk-results file and
   (b) create an API user under our GSP.
5. **Counsel review** of the ToS wording (pending invoices not valid; responsibility for
   applicability) and of the DPA with the GSP.

---

## 11. Phased plan and estimates

Engineer-days are for one senior full-stack engineer with AI assistance. Calendar time in
brackets where it depends on others.

| Phase | Scope | Needs credentials? | Estimate |
|---|---|---|---|
| **0. Safety + honesty** | Stop mock IRNs reaching real prints and exports (demo-company/dev flag + print marker). Correct `einvoice-irn` to `partial` in `parity-data.json` / `FEATURE-PARITY.md`. Carry `irn/ackNo/ackDate/ewayBillNo` through alteration until Phase 2 replaces them. | No | 1–2 d |
| **1. Correct payload + portal-assisted** | Master fields (ledger PIN/city, company city, item service flag). Number guards (IRP regex, 16 chars, no leading 0, case-insensitive per GSTIN+FY+type across voucher types, burned numbers). `core/einvoice.ts` builder for INV/CRN/DBN × B2B/SEZWP/SEZWOP/EXPWP/EXPWOP/DEXP, services from ledger lines, UQC, discount, other charges, round-off, cess, and state **codes**. Validator + assertion suite. Bulk JSON export (≤2 MB split). Results import. QR encoder + print. `einvoices` table (Dexie v8) in local mode. Register keys and statuses. `cancelled` voucher flag (if approved). | No | 10–14 d |
| **Prereq. TLS** | HTTPS for the API (HANDOFF Phase B item 1) | DNS token / provider | 2–3 d [external] |
| **2. Server relay on the mock provider** | `007_einvoice.sql` (RLS FORCE). Vault. Endpoints. Worker + `SECURITY DEFINER` claim. State machine, idempotency, duplicate reconciliation by fingerprint. Server-owned-table denylist on `/sync/push` and `/v1`. `einvoices` change-log publication + SSE. Client queue, status chip, generate-on-save, cancel flow, bulk. Drift detection in `gstCheck.ts`. Audit. `nicCompatible` crypto. **Spike:** try IRIS/Clear sandbox sign-up. | No (mock); spike needs a sign-up | 12–16 d |
| **3. Partner integration + certification** | Partner adapter, error-code map, sandbox test cases, failover (NIC1/NIC2), load test of the worker | **Yes**: sandbox credentials | 5–8 d [2–4 wk with partner] |
| **4. Production pilot** | Production credentials, 2–3 pilot customers, monitoring and alerts on `unknown` / `needs_review` / auth failures, runbook | **Yes**: contract + production | 3–5 d [2–4 wk pilot] |
| **5. EWB by IRN + extras** | Transport sub-panel, EWB by IRN and cancel, GSTIN lookup to fill party masters, 30-day ageing on Dashboard, mobile notifications (and approval hook once the approvals module exists) | Yes (same partner) | 6–9 d |
| **Total** | | | **≈ 39–57 engineer-days (≈ 8–11 weeks)**; Phases 0–2 (≈ 25–35 d) need no contract |

**Sequencing:**
- Phase 0 now.
- Phases 1 and 2 in parallel with the RFQ.
- Phases 3–4 start when a contract is signed and TLS is live.
- Phase 1 alone already gives Tally-like offline compliance for customers willing to upload on
  the portal.

---

## 12. Risks

| Risk | Mitigation |
|---|---|
| A collided number gets the wrong IRN attached | Fingerprint comparison on duplicates; `needs_review`; per-device series or server numbering for e-invoicing companies |
| Lost response, then a re-send | `unknown` state; get-by-doc before any re-send; 3-day alarm |
| Books edited after IRN | Client field lock + server drift flag + credit-note path |
| GSP minimums make small tenants unprofitable | Pooled ASP minimum; price e-invoicing into Pro/Business plans; Option C as a fallback |
| IRP or GSP outage on deadline days | Queue + backoff; NIC1/NIC2 failover through the GSP; portal-assisted fallback (Option A) always available |
| Regulatory churn (Ship-To GSTIN, closure API, MFA, thresholds) | Adapter + schema version field; quarterly check of GSTN advisories; keep new fields optional until notified |
| Credential leak | TLS, vault, no client storage, audit, impersonation gate |

---

## 13. Open questions for the owner

1. **Partner route:** RFQ both a NIC-routing GSP and the IRIS/Clear integrator programmes, or do
   you already have a GSP relationship through the Tally-partner business (e.g. Webtel, Masters
   India)? An existing relationship could shorten Phase 3 a lot.
2. **Commercials:** pass the per-IRN cost through, include N IRNs per plan, or keep e-invoicing
   Business-plan only? Which customers on the current list exceed ₹5 Cr AATO, and how many IRNs
   a month do they issue?
3. **Desktop-only users:** is "online e-invoicing requires a Comsky cloud account (portal-assisted
   otherwise)" acceptable? Tally users expect it to work from the desktop with just TSS.
4. **Voucher cancel model:** approve a Tally-style `cancelled` voucher flag (keeps the number,
   zero effect, one exclusion helper), or keep "delete + burned number"?
5. **Numbering:** for e-invoicing companies with more than one device, mandate per-device series
   prefixes now, or prioritise wiring the existing server numbering into the client?
6. **Debit Note semantics:** in this app a Debit Note voucher is often a **purchase return**,
   which is not an outward-supply DBN. Should DBN e-invoicing apply only to debit notes against
   *sales* (price revisions)? Please confirm with the consultants' CA.
7. **Pilot:** can you get one real **portal bulk-results file** (sanitised) and one pilot
   customer willing to create an API user under our partner?
8. **Approvals:** should IRN cancellation require owner approval (mobile) by default, given the
   24 h window?
9. **Scope of EWB:** is EWB-by-IRN enough for phase 5, or do customers need standalone e-way
   bills (delivery notes, B2C, inward) soon? That is a separate integration.
10. **Hosting:** the backend is a single VM in India (DPDP residency is fine). Confirm the GSP
    DPA can say "India only" end-to-end.

---

## 14. Sources (consulted 2026-10-15)

- GSTN, *E-Invoice Portal: API Details of all 6 IRPs (29 Nov 2023)*: https://tutorial.gst.gov.in/downloads/news/e-invoice_api_integration_guide_irps.pdf
- NIC e-Invoice API sandbox docs: https://einv-apisandbox.nic.in/ ([auth](https://einv-apisandbox.nic.in/version1.04/authentication.html), [generate](https://einv-apisandbox.nic.in/version1.03/generate-irn.html), [cancel](https://einv-apisandbox.nic.in/version1.03/cancel-irn.html), [get by doc](https://einv-apisandbox.nic.in/version1.03/Get_IRNdetailsbyDocDetails.html), [get by IRN](https://einv-apisandbox.nic.in/version1.03/get-eInvoicedetails.html), [EWB by IRN](https://einv-apisandbox.nic.in/version1.03/ewaybill-generation-irn.html))
- GSTN 30-day advisory (AATO ≥ ₹10 Cr from 1 Apr 2025): https://www.mahagst.gov.in/public/uploads/gstnadvisory/1760531580_293%20Advisory%20Time%20Limit%20for%20Reporting%20e%20Invoice%20on%20the%20IRP%20Portal%20%20Lowering%20of%20Threshold%20to%20AATO%2010%20Crores%20and%20Above.pdf
- IRIS IRP: https://einvoice6.gst.gov.in/content/kb/api-integrator-registration/ · https://einvoice6.gst.gov.in/content/kb/manage-production-access/
- Clear IRP FAQ: https://einvoice4.gst.gov.in/frequently-asked-questions/
- Tally: https://help.tallysolutions.com/tally-prime/e-invoice-tallyprime/one-time-setup-for-e-invoicing/ · https://help.tallysolutions.com/generate-e-invoice-offline-tallyhelp/ · https://help.tallysolutions.com/duplicate-irn-error/ · https://tallysolutions.com/business-guides/tallyprime-total-cost-of-ownership-india/ · https://help.tallysolutions.com/connected-services-in-tallyprime/
- Zoho Books: https://www.zoho.com/in/books/help/e-invoicing/set-up.html
- Busy: https://busy.in/faqs/taxation/e-invoice-generation/17/
- ClearTax: https://docs.cleartax.in/cleartax-docs/e-invoicing-api/e-invoicing-api-reference/govt-compatible-apis/generate-irn · https://cleartax.in/s/gst-e-invoice-error-code-2150-wrong-vehicle-number · https://cleartax.in/s/e-invoicing-api-integration-modes
- GSTZen pricing: https://gstzen.in/e-invoicing-api-integration-pricing · https://gstzen.in/e-way-bill-api-integration-pricing-details
- Masters India: https://www.mastersindia.co/e-invoicing-api/ · https://docs.mastersindia.co/einvoicing/llms.txt
- Sovos / IRIS acquisition: https://www.businesswire.com/news/home/20250812153263/en/Sovos-Expands-Global-eInvoicing-and-Tax-Compliance-Coverage-with-Acquisition-of-IRIS-APAC-Tax-Technology-Business
- Advisory 668 (Ship-To GSTIN / EWB closure on hold): https://taxupdate.in/gst/831/gstn-advisory-668-eway-bill-enhancements-on-hold-ship-to-gstin-closure-august-2026/
- Case-insensitive document numbers: https://www.taxmann.com/post/blog/gstn-issued-advisory-on-case-insensitivity-in-irn-generation/
- MFA: https://taxmann.com/post/blog/multi-factor-authentication-on-e-invoice-and-e-way-bill-portals-will-be-mandatory-for-all-taxpayers-users-gstn
- EWB 180/360-day rules: https://cleartax.in/s/time-limit-for-e-way-bill-generation
- Thresholds 2026: https://www.gimbooks.com/blog/5-crore-e-invoice-turnover-rule-2026/amp/
- Exemptions: https://taxguru.in/goods-and-service-tax/e-invoicing-system-gst-turnover-exceeds-rs-10-cr-w-e-f-01-10-2022.html · https://cbic-gst.gov.in/pdf/circular-186.pdf
- GSP selection history: https://gstzen.in/a/selection-of-gsp-3rd-batch.html
- DPDP Rules 2025: https://www.theweek.in/wire-updates/business/2025/11/14/del148-biz-dpdp-rules-ld-govt.html
- e-invoice JSON download window: https://sovos.com/regulatory-updates/vat/india-new-e-invoice-download-functionality/
- Signed QR fields: https://docs.enriched-api.vayana.com/routes/enriched/Trade-Verification-Service/Verify%20EInvoice/apis/Verify-Qrcode/verify-qrcode/
