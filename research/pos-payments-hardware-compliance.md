# POS infrastructure for Indian retail: payments, hardware and compliance (research for a Comsky Books POS module)

> **Status:** research only. Nothing here is built, no gate (assert, shots, build, perf) was run, and this is the only repo file written. Written 2026-10-25 against `main` at `5aee765`
> (the working tree also carries other workstreams' uncommitted research notes; a fact about an untracked file may move).
> **Scope:** the infrastructure a POS needs in India that is not the counter screen itself: **(1) payments** (UPI static and dynamic QR, confirmation, card terminals, cash, wallets, credit, gift
> vouchers, split tender, rounding, what the app must never store), **(2) hardware** (thermal printers, cash drawer, displays, scales, scanners, kitchen printers, tablets, and how a web or Electron
> app reaches each), **(3) compliance** (GST invoicing at the counter, numbering, the B2C dynamic QR rule, e-invoice thresholds, composition bills, Legal Metrology, retention, receipt content), and
> **(4) e-bills, WhatsApp and customer privacy**. It ends with **[J] an integration plan**: what to build in-house now, what needs a partner or an API, and what the owner must decide.
> **What it leans on, and does not repeat:** the code audit `docs/research/pos-code-audit.md` (what a POS bill touches in today's engine, tender shapes S1 and S2, series per terminal, lanes A to H),
> the label and scanning study `docs/research/labels-and-scanning.md` and its spec `docs/design/batch-serial-labels-spec.md` (print paths, the Electron bridge pattern, the scan detector: **read its
> sections 13.6 and 14 before section 2.6 here**), `docs/research/pos-retail-market.md` (written alongside this one; its section 6 reached the same statutory readings for the Rs 200 slice, MRP, the HSN digits, e-invoice and the dynamic QR), `docs/research/pos-restaurant-market.md` and `pos-pharma-market.md` (vertical needs; their GST and Drugs-rules sections are cross-referenced, not
> re-derived), `docs/research/einvoice-gsp.md`, `docs/research/live-bank-feeds.md`, and `docs/research/mobile-tech-options.md`.
> **Method.** Statutory text was read from primary documents where I could get them (the CBIC rules and notifications as PDFs, the RBI circulars, the PIB note on the DPDP Rules, the NPCI deep-link
> specification, vendor developer documentation); everything is **paraphrased**. A PDF the fetch tool could not read was saved and its text layer extracted locally (this is how Rule 46, Notifications 14, 71,
> 78 and 89 of 2020, 06 and 28 of 2021, 05/2025 (Rate), Circulars 146 and 167, the GST service-rates table, the Packaged Commodities Rules, the NPCI linking specification, the NPCI UPI MDR FAQ and the PIB note on the DPDP Rules were read in full).
> **Evidence tags:** **[W key]** a page or document I fetched and read (source key in section 7) · **[Ws key]** a web-search summary only, the page itself not read · **[C]** read in this repo
> (`file:line`) · **[R]** stated in a sibling repo document and not re-checked here · **[M]** arithmetic or a measurement I ran (Appendix A) · **[J]** my judgement · **[unverified]** could not be checked.
> **What this study could not do:** no live account of any PSP, card acquirer, WhatsApp provider or product (Petpooja, Marg, LogicERP, GoFrugal and the rest are read from their public pages only); **no
> hardware** (no printer, drawer, scale, scanner, EDC or soundbox was touched, so every device statement is documentation or code, never a test); no legal advice. The web-search budget ran out
> part-way (Cashfree's soundbox question, and a few further checks, were left unanswered; section 6 lists them), and several authoritative hosts refused the fetch tool (section 6).

---

## 0. Findings that decide the design (read this first)

1. **A QR on the counter proves nothing; confirmation needs a channel the POS does not own.** The NPCI link specification itself says that a merchant app must check the final status with its server or
   PSP, and the response it describes goes back to an app on the payer's phone, not to the till [W npci-link §1.4]. The five ways to know a UPI payment arrived are a PSP callback or poll, an integrated
   terminal, a soundbox the cashier listens to, a next-day bank statement, and an Android SMS reader (restricted by Google Play policy): section 1.3. **The honest default is a tender with a confirmation
   state** (confirmed by PSP, by terminal, by bank statement, or attested by the cashier) and a shift-close list of the unconfirmed ones. [J]
2. **The link the repo prints today is unsigned and carries no reference.** `upi.ts` builds `upi://pay?pa&pn&am&tn&cu` from the owner's own VPA and deliberately leaves out `tr`, `mc`, `mid`, `msid`, `mtid`,
   `url`, `mode` and `sign` [C `src/core/upi.ts:16-22`]. The NPCI draft says a payer app that finds no signature should warn that the source could not be verified and ask for the PIN [W npci-link §1.3
   step 7c]; `tr` is mandatory "for merchant transactions and dynamic URL generation" [W npci-link §1.2]. A **PSP-issued dynamic QR** (registered, signed, reference-carrying) is the route that both
   avoids the warning and gives a confirmation. [W][J]
3. **UPI stopped being free for most shops on 15 October 2026, on paper.** NPCI's FAQ of 15 September 2026 introduces a 0.4% merchant discount rate on person-to-merchant UPI payments **above Rs 2,000**,
   capped at Rs 300 from Rs 75,000, zero for payments up to Rs 2,000, zero for small merchants taking up to Rs 1 lakh a month on a QR into their own accounts (the "P2PM" tier), a flat Rs 5 for named sectors,
   and **merchants may not pass it on to customers** [W npci-mdr]. One vendor page reports that NPCI said on 13 October that the start moves to after Diwali, with a date to follow [W terra-mdr]; I could not corroborate
   that, so **the start date is unverified and the rates are not**. A POS therefore needs a cost-per-tender view and must never offer a "UPI surcharge". [W][J]
4. **The statutory dynamic QR does not apply to the target customers.** Notification 14/2020-CT requires a dynamic QR on a B2C invoice only for a registered person whose aggregate turnover exceeds
   **Rs 500 crore** (in any preceding year from 2017-18, Notification 71/2020), with carve-outs for banks, insurers, goods transport, passenger transport, multiplex cinema and OIDAR; the penalty was waived
   until **30 September 2021** (Notification 28/2021) and nothing later was found [W n14, n71, n28-21, c146]. Circular 146/02/2021 lets a **payment cross-reference on the invoice** (transaction id, date, time, amount,
   mode; for cash the amount and date) stand in where the customer paid without the QR or before the invoice [W c146 Q3 to Q5]. So **pay first, then issue the invoice with the reference printed** is a design that
   satisfies even the strictest reading, abandons no invoice numbers, and is a feature for every shop. [W][J]
5. **Comsky must never hold a customer's money or a card number.** The RBI's Payment Aggregator Directions of 15 September 2025 make proximity (physical) aggregation a regulated category with an RBI
   authorisation and net-worth floor (Rs 15 crore at application per a law-firm note) [W rbi-pa-pr, azb-pa, lex-pa]; only card issuers and card networks may store card data, and anyone else may keep only the **last four
   digits and the issuer's name** for reconciliation [W rbi-cof21]. A closed-system gift card (for the issuer's own goods) needs no RBI authorisation but cannot be used for third parties or cash-out
   [W rbi-ppi]. Whether a software vendor that only shows a QR or routes to a partner is itself an aggregator is **not answered by any source I read** [unverified]: take advice before going beyond "the
   partner collects and settles". [W][Ws][J]
6. **Every counter bill is a numbered invoice, except a small slice.** Section 31(3)(b) of the CGST Act lets a registered person skip the per-bill invoice when the supply is under Rs 200, the buyer is
   unregistered and does not ask for one, and the supplier issues **one consolidated tax invoice at the close of each day** (Rule 46, fourth proviso) [W cgst-31, cbic-rules]. Everything else needs its own
   number from a series that is consecutive, at most **16 characters** of letters, digits, `/` and `-`, and unique in the financial year (Rule 46(b)) [W cbic-rules]. A series per terminal fits; 16 characters
   is tight for a chain (budget table, section 3.3); and the e-invoice portal upper-cases a document number, so `t1/1` and `T1/1` are one document [R einvoice-gsp §3]. [W][R][J]
7. **A mixed basket is an "invoice-cum-bill of supply".** Rule 46A lets a registered person issue one document for taxable and exempt supplies to an unregistered buyer [W cbic-rules]. A grocery counter
   (milk and vegetables beside packaged goods) is that case every day; a composition dealer issues a **bill of supply** that must carry the legend "composition taxable person, not eligible to collect tax on
   supplies" (Rule 5(1)(f)) [W cbic-rules]. [W]
8. **Buyer details are rule-driven.** For an unregistered buyer the name, address, delivery address and state with code are required when the taxable supply is **Rs 50,000 or more**, and on request below
   that (Rule 46(e), (f)); a taxpayer up to Rs 5 crore may omit the HSN digit count on invoices to unregistered buyers (Notification 78/2020: 4 digits up to Rs 5 crore, 6 above) [W cbic-rules, n78]. [W]
9. **A B2B bill from a taxpayer above Rs 5 crore is not valid without an IRN.** E-invoicing applies above Rs 5 crore aggregate turnover from 1 August 2023 (Notification 10/2023-CT), with banks, insurers,
   goods transport, passenger transport, multiplexes, SEZ units and government departments outside it; an invoice that should have carried an IRN and did not is "not treated as an invoice" (Rule 48(5));
   B2C is not an e-invoice document [W cbic-rules, tally-einv, taxheal-n10; R einvoice-gsp §3]. The app has no IRP call [C `CLAUDE.md`, Known limitations], so a **GSTIN entered at the counter by an
   e-invoice-liable seller is a hard stop or a partner call**, not a print. [W][R][C]
10. **Never sell above the printed maximum retail price.** Rule 18(2) of the Packaged Commodities Rules bars any sale of a packed commodity above its retail sale price, and Rule 18(3) caps a price after a
    tax revision [W lm-pc]. A POS that stores MRP (the code audit says today's engine cannot) needs an MRP guard, and **a bill-total round-up must never take a basket of MRP-priced goods above the sum of its
    MRPs** [W lm-pc; J]. [W][J]
11. **Of the web and Electron builds, only the Windows Electron app can reach every device without extra software.** It can print silently, send raw bytes by TCP 9100 or the Windows spooler, read a serial scale, drive a second screen
    and act as a store hub. A browser needs HTTPS and a chooser for each device, cannot reach a USB printer that has a vendor driver on Windows without swapping in WinUSB, and an Android browser cannot
    reach a CDC-ACM serial device through WebUSB [W chrome-webusb, electron-devices, electron-print]. An HTTPS page cannot fetch an `http://` LAN address (mixed content), which is the trap in every
    "tablet prints through the counter PC" design (section 2.5). [W][J]
12. **Receipts are a data-volume problem as well as a layout problem.** The rupee sign and any Indian script are not reliably in a thermal printer's code pages, so those lines go as **raster images**
    [Ws thermal-unicode]. At 203 dpi an 80 mm receipt of 150 mm is 84 KiB raster and a 58 mm one 56 KiB [M]; that is milliseconds over USB and **45 seconds over a 19,200-baud serial cable** [M], while the
    mechanism itself prints at about 160 mm/s [Ws tvs-store]. Hence a hybrid: native text for the body, raster only where a line needs it (section 2.2). [M][J]
13. **The label bridge in progress is the template for the receipt bridge.** `electron/labelLang.cjs` already emits TSPL from a structured job so a hostile page can never reach a printer command
    [C `electron/labelLang.cjs:1-12`]; the receipt path adds an **ESC/POS emitter and a job validator** of the same shape (a receipt, a drawer pulse, a cut), never bytes from the renderer (section 2.6).
    [C][J]
14. **Plain HTTP on the backend blocks the payment route.** PSP webhooks, bank feeds, the Android secure-context features and any hosted e-bill page all need HTTPS; the live site is HTTP only
    [R pos-code-audit §0 item 11; R live-bank-feeds TL;DR 5]. **TLS and a hostname come before the first PSP or WhatsApp provider is called.** [R][J]
15. **E-bills are three different things.** A `wa.me` link opens the customer's chat with text and cannot attach a file [C `src/core/share.ts:127-129`; R pos-code-audit §8]; a hosted bill page plus a link
    (and the UPI `url` field, the draft's Bill in the Box, which lets the payer's app show the invoice before paying [W npci-link §1.4]) needs only our backend; an attached PDF or an automatic message needs the
    WhatsApp Business Platform through Meta or a provider, with an opt-in, templates and a per-message price [W meta-wa]. SMS needs DLT registration of the sender and every template [W exotel-dlt]. [W][C]
16. **Privacy law arrives later than the product.** The DPDP Rules were notified on 14 November 2025 with an eighteen-month phase-in, so most duties on a business that collects a phone number bite from
    about mid-May 2027 [W pib-dpdp; Ws dpdp-news]. Collecting a phone number for a bill is personal data collection **today** in design terms: a separate notice, purpose limitation, a way to withdraw, a
    contact point, breach notification, and a 90-day window to answer access, correction or erasure requests are all in the PIB note [W pib-dpdp]. How erasure meets the 72-month GST retention duty was
    **not read** [unverified]. [W]

---

## 1. Payments

### 1.1 The tenders a counter needs, and how each reaches the books

The code audit's shapes apply unchanged: **S1** a single tender whose party IS the tender ledger, and **S2** a Sales voucher plus same-transaction Receipt vouchers for a mixed tender; S3 (tender legs
inside the Sales voucher) corrupts GST until the readers are audited [R pos-code-audit §2.3]. This section adds what each tender needs **besides** its amount.

| Tender | What it is at the till | Where confirmation comes from | What may be stored (the rule) | Books shape [R pos-code-audit §2.3] |
|---|---|---|---|---|
| **Cash** | notes and coin, change given | the cashier counts; the drawer total at shift close | amount, tendered, change; the cash-drawer open event | party or Receipt on a per-terminal Cash ledger |
| **UPI, static QR** | the shop's own QR; the payer types the amount | none by the till (soundbox, phone, bank statement) | the bank's UTR or RRN if the cashier enters it; the payer VPA only if the provider returns it and the owner needs it [J] | S1 with a UPI collections ledger, or S2 Receipt |
| **UPI, dynamic QR** | an amount-locked QR per bill, usually from a PSP | the PSP's callback or poll, or an integrated terminal | PSP order id, UTR/RRN, status, time, amount, `tr` (our bill reference) | as above |
| **UPI collect / intent** | a request pushed to the payer's app (collect), or a link opened on the payer's own phone (intent) | the PSP | as dynamic | as above |
| **Card, standalone EDC** | the cashier keys the amount on the machine, the customer taps or inserts | the slip and the machine; the cashier types an approval code | **last four digits, issuer name** (the RBI's limited data), approval code, RRN, terminal id, acquirer; **never the card number, expiry, CVV, PIN or track data** [W rbi-cof21] | S1 or S2 on a Card collections (bank) ledger |
| **Card, integrated EDC** | the till pushes the amount to the machine | the acquirer's cloud or the wired link answers with status and references | as above | as above |
| **Wallet (a PSP's)** | in practice a QR or app flow | as UPI or the PSP | as UPI | as UPI |
| **Store gift voucher / store credit** | a value the shop issued | the shop's own register | voucher code, balance, issue and redeem events | a liability ledger (section 1.7) |
| **Third-party voucher** (a brand's gift card, a payment partner's voucher) | accepted as tender, settled by the issuer later | the issuer's redemption response or a printed code | code and issuer | a receivable from the issuer (Pine Labs lists several voucher modes) [W pine-cloud] |
| **Credit (udhaar)** | the customer's account is debited | the customer, by the shop's own ledger | the bill-wise reference | S1 with the customer as party; the bill-wise engine is complete [R pos-code-audit §1.6] |
| **Loyalty points** | a discount or a payment in points | the shop's own register | points debited, rate | a discount line before tax, not a tender (section 1.7) |

Each Receipt in S2 can reuse the fields a voucher entry already has: `instrument` (a reference) and `instrumentType` including `UPI`, `Card` and `Cash` [C `src/types.ts:888-890`], captured today by the bank-instrument
panel on the accounting grid [R pos-code-audit §2.5]. A tender-confirmation field is new and belongs to the code audit's lane C contract.

### 1.2 UPI: what the specification says, and what the repo does with it

**The specification read.** The NPCI "UPI Linking Specifications" version 1.6, marked "(Draft)", November 2017, is the document the repo cites. NPCI's own hosted copy could not be fetched; I read a third-party
copy of the same file [W npci-link] (the repo's header says the same of its source, `src/core/upi.ts:3-8`). It defines `upi://pay?` links used for QR codes, intents and proximity transfers.

| Parameter | Static QR | Dynamic QR | What it means (paraphrased from the draft) |
|---|---|---|---|
| `pa` | mandatory | mandatory | payee address (VPA) |
| `pn` | mandatory | mandatory | payee name |
| `mc` | optional | optional | payee merchant category code; passed through as given |
| `tid` | optional | optional | a transaction id that must be **PSP-generated** when present |
| `tr` | optional | conditional | a transaction reference (order, bill, booking id); **mandatory for merchant transactions and for dynamic links** |
| `tn` | optional | optional | a short note |
| `am` | optional | mandatory | amount; if absent the payer's field is editable |
| `mam` | optional | conditional | a minimum amount where the payer may edit the amount; absent or `null` means not editable |
| `cu` | optional | optional | currency, INR only |
| `url` | optional | optional | a link that shows the bill or invoice for this transaction (the draft's Bill in the Box section: the payer's app offers a view-invoice option; the link must relate to this transaction and carry no unsolicited content) |
| `mode`, `sign`, `orgid` | marked mandatory in the draft's table for the signed forms (the table is garbled in my text extraction) | | `mode` 01 QR, 02 secure QR, 04 intent, 05 secure intent and others; `sign` a base64 RSA signature over the whole link; `orgid` is `000000` for a merchant-created QR |
| `mid`, `msid`, `mtid` | optional | | merchant, store and terminal ids (max 20 each), **echoed in all messages for the acquiring bank's and merchant's reconciliation** |

Other points in the draft that matter here [W npci-link]: a **merchant-created signed QR** is signed with the merchant's private key and the public key is registered by the acquiring bank; **a link with no
signature should make the payer's app warn that the source could not be verified and ask for the PIN** (§1.3); a space is written `%20` (§1.4 note 4); the standard response to a merchant **app** is a
`txnId`, response code, approval reference and status, and the draft says plainly that a merchant app **must check the final status with its own server or PSP server** (§1.4); and the draft says a one-person shop may simply print a static QR
with the payee address and name, the payer then entering the amount (§1.1, example 2). NPCI's 2017 Circular 18, seen only in a search summary, adds that the critical parameters of a
dynamic QR are not editable by the payer and that the acquiring PSP fills `tr` (up to 35 digits, as the summary renders it) [Ws npci-c18].

**What the repo does.** `upiLink` writes `pa`, `pn`, optionally `am` and `tn`, then `cu=INR`; the QR is drawn by `core/qr.ts`; printing is off by default and only with a valid VPA the owner typed; the code says in
terms that `tr` and the registered-merchant fields are left out, that a merchant code the payee is not registered under can be refused by the payer's app, and that the NPCI text was read from a third-party
copy and must be re-checked [C `src/core/upi.ts:3-22`, `:112-130`]. It prints with the bank block of an A4 invoice [C `src/types.ts:253-256`; R pos-code-audit §2.5]. That is right for an invoice that asks
for a bank transfer; it is **not a payment channel a till can confirm**.

**Collect, intent and limits** (the first two bullets are [Ws] search summaries; the others [W]):
- NPCI's circular of 29 July 2025 ended UPI peer-to-peer **collect** requests from 1 October 2025; **merchant** collect requests continue [Ws p2p-collect].
- From 15 September 2025 per-transaction limits for named person-to-merchant categories rose to Rs 5 lakh, with a Rs 10 lakh daily total, for verified merchants and subject to each bank's own lower
  limit [Ws p2m-limits]. The NPCI FAQ of September 2026 says daily caps are typically Rs 1 lakh to Rs 5 lakh depending on the category [W npci-mdr Q20]. A jewellery or electronics till should therefore
  expect a **UPI decline on a large bill** and offer a second tender. [J]
- **The 2026 merchant discount rate** (above): 0.4% on P2M above Rs 2,000, cap Rs 300 from Rs 75,000; Rs 5 flat above Rs 2,000 for railways, telecom, insurance, fuel, utilities and similar; zero up to Rs 2,000;
  zero for P2PM micro-merchants (inward UPI QR credit up to Rs 1 lakh a month into their own accounts, moved to P2M after three consecutive months above it); AutoPay mandates and credit-linked UPI (RuPay credit
  on UPI, credit lines) are outside this MDR; merchants cannot pass it to buyers; existing QR stands and soundboxes keep working [W npci-mdr Q3, Q22 to Q36]. The FAQ is a copy hosted by a broker; the
  official copy was not fetched [W npci-mdr]. Worked figures [M]: Rs 2,000 pays nothing, Rs 3,000 pays Rs 12, Rs 50,000 pays Rs 200, Rs 75,000 and Rs 1,00,000 both pay Rs 300.
  The same FAQ puts typical credit-card MDR at 1.5% to 2.5% and debit-card MDR at up to 0.90% [W npci-mdr Q4] (NPCI's own comparison, a source with an interest).
- **A personal QR is a legitimate micro-merchant tool only up to the P2PM line.** NPCI describes P2PM as small vendors receiving up to Rs 1 lakh a month through UPI QR directly into personal accounts [W
  npci-mdr Q23 to Q24]. Above that the acquirer moves them to P2M. The repo's "owner's own VPA" link therefore suits the smallest kirana and nobody else [J].

### 1.3 The confirmation problem

**What a till can know without a provider: nothing.** The payer's phone talks to the payer's PSP and the payee's bank; neither tells the till. Everything below is a way to get that answer from somewhere.

| # | Route | How the till learns | Evidence | Latency | What it needs | Offline day |
|---|---|---|---|---|---|---|
| A | **Look at the customer's phone** | the cashier reads a success screen | none, and a screenshot is not a payment [J, unverified as a fraud pattern: no source read] | immediate | nothing | works, and is the weakest control |
| B | **Static QR plus a soundbox** | a separate speaker announces a credit; the till is not told | Razorpay's Dynamic Soundbox pairs an invoice-level QR with an audio and screen confirmation and says it integrates with billing systems, without naming an API [W rzp-soundbox]; Petpooja supports soundboxes for integrated QR and voice alerts for static QR [W petpooja-fin]; NPCI says soundboxes keep working [W npci-mdr Q25] | seconds | a soundbox rental; no software | the soundbox has its own SIM or Wi-Fi; the till works |
| C | **A PSP's dynamic QR with a callback or poll** | the PSP answers `PAID` for our reference | Razorpay: a UPI QR through the QR Codes API (`type upi_qr`, `usage single_use`, `fixed_amount`, `close_by` two minutes to two hours, closes itself when paid, on-demand activation) or through a virtual-account QR (`close_by` at least 15 minutes, polling and webhooks) [W rzp-qr, rzp-va]; PhonePe: a static-QR callback server-to-server on a terminal state, or a Transaction-List pull plus a Metadata push that tags a bill number onto a transaction [W phonepe-sq]; Cashfree lists static and dynamic QR and a softPOS app, its page names no soundbox and no webhook [W cashfree-upi]; Zoho POS shows a Paytm dynamic QR that expires in ten minutes [W zoho-paytm]; GoFrugal says its POS shows a dynamic QR and watches the bank for success [W gofrugal-pay, a vendor] | seconds | a PSP merchant account, **HTTPS callback**, keys, a server | needs internet at the till or a backend; offline falls to A, B or E |
| D | **An integrated terminal** (card and UPI on the EDC) | the till pushes the bill to the machine and polls or receives status | Pine Labs cloud: upload a billed transaction, get a reference (PTRID) the cashier selects on the machine, then poll `GetStatus` or take a callback; split tenders go as separate uploads with a sequence number; `AutoCancelDurationInMinutes`; payment modes include card, UPI, Bharat QR, wallets, vouchers and points [W pine-cloud]; Paytm: `POST /edc-integration-service/payment/request` and `GET .../txn/status` with states IN_QUEUE, PENDING, FAILED, COMPLETED, EXPIRED, CANCELLED, a six-digit payment-request id the cashier keys on the machine, and no callback described [W paytm-edc]; Paytm also lists a wired ECR over USB with a DLL for Windows and Linux on Pax and Ingenico models and an Android app-invoke [Ws paytm-edc-search]; Petpooja's Pay+ routes through Paytm, Razorpay, Pine Labs, Mosambee and BharatPe and matches each payment to the bank payout [W petpooja-fin] | seconds | an acquirer relationship, terminal ids, a backend or a local link | cloud variants need internet at the till and at the machine |
| E | **Bank statement match** | the next day's statement (or the feed) shows the credit | the repo imports statements and matches lines to vouchers [C `src/core/bankStatement.ts`, `src/core/brsMatch.ts`]; live bank feeds are polled and "not real-time" [R live-bank-feeds TL;DR 4] | T+1 | the shop's own bank file | works; slow |
| F | **A bank-mapped static QR, with the bank's SMS read on an Android device** | the QR credits the shop's own account and an app parses the credit SMS | Google Play lists UPI and financial-transaction verification among the narrow, reviewed exceptions to its SMS-permission ban; the default SMS handler is the main permitted use [W play-sms] | seconds | an Android native app, a Play declaration, a fragile parser per bank [J] | works if the phone is the till or paired with it |
| G | **A bank's collections or virtual-account API** | each customer payment lands in a virtual account with auto-reconciliation | ICICI's programme lists collections with virtual accounts and auto-recon [R live-bank-feeds §3.3] | seconds to minutes | a bank partnership | internet |

**Recommended pattern [J].** A tender has `state`: `pending`, `confirmed`, `expired`, `failed`, or `unconfirmed` (cashier-attested, offline or no provider), and `source`: `psp`, `terminal`, `statement`,
`cashier`. **Pay first, issue second:** the QR (route C) or the terminal request (D) is created for a *draft* with its id as `tr`; the invoice number is taken only after `confirmed` (the code audit already says
"take the number when the tender is finalised, not when the cart opens" [R pos-code-audit §4]); the printed invoice carries the **payment cross-reference** (transaction id, date, time, amount, mode) that
Circular 146 accepts [W c146 Q3 to Q5]. An abandoned QR then burns no invoice number and leaves no cancelled shell. Offline, the cashier may attest a static-QR payment: the bill prints as paid with
`unconfirmed`, appears in the shift-close list, and is cleared when the statement matches (route E) or the cashier confirms against the soundbox. Restaurants pay after eating: the table bill printed before
payment is a **proforma without an invoice number** (or the final number is reserved only at settlement), because a printed number is an issued document [J].

**Sandbox and reality gaps.** None of the vendor flows was run: no PSP account exists, and the repo's backend cannot receive a webhook (HTTP only). The Razorpay webhook event names for a QR credit, the Paytm
request fields and signature, and PhonePe's dynamic-QR and EDC APIs were not read (the pages named them without the details I needed) [unverified].

### 1.4 Who is an aggregator, and what Comsky must not do

- The RBI consolidated its payment-aggregator rules on **15 September 2025** into the Regulation of Payment Aggregators Directions, 2025; the press release lists the definitions of the categories, the
  authorisation process, merchant due diligence and escrow operations as the covered points [W rbi-pa-pr]. A law-firm note says the categories are PA-O (online), PA-CB (cross-border) and **PA-P (physical, where
  the acceptance device and the payment instrument are physically close)**, that a non-bank PA-P needs RBI authorisation, KYC and FIU registration, and that PA-P-only entities had to apply by **31 December 2025**
  or stop by **28 February 2026** [W azb-pa]; another note puts the net worth at **Rs 15 crore** at application and **Rs 25 crore** by the end of the third financial year [W lex-pa]. I could not open the final
  text [unverified].
- **Rule for Comsky [J]:** never collect, hold or settle a customer's payment, never run a pooled account, never be the party the customer's bank pays. The customer is the merchant of record at a PSP or an
  acquirer who is authorised; Comsky passes references and statuses. Whether merely generating a QR for a partner's merchant account crosses the line is a legal question [unverified]. The live-bank-feeds study
  reached the same rule for payouts [R live-bank-feeds §3.4].
- **Stored value.** A **closed-system PPI** (a gift card or credit issued by an entity for purchases from that entity only, with no cash withdrawal) is outside RBI authorisation and supervision, and cannot
  be used for third-party payment or settlement [W rbi-ppi §2.1]. A shop's own gift card or wallet fits that; a card accepted at other shops, or a wallet that can pay out, does not. The directions page I read
  is dated "updated as on September 30, 2026" [W rbi-ppi].

### 1.5 Card terminals: integration models and the card-data rules

**Three integration models** (from the vendor pages above):
1. **Standalone**: the cashier types the amount on the machine and then types what the slip says into the till (approval code, last four digits, RRN). Cheapest, no partner, **wrong-amount and wrong-bill errors are
   the cashier's**; reconciliation is the acquirer's settlement report against the till's card total [J].
2. **Cloud-integrated** (Pine Labs, Paytm, Razorpay's POS Bridge, Petpooja Pay+): the till posts the bill to the acquirer's cloud, the machine fetches it (Pine Labs by a reference the cashier selects, Paytm by a
   six-digit request id), the till polls or is called back. Needs internet at both ends and one partner agreement per acquirer, but **no local driver** [W pine-cloud, paytm-edc, petpooja-fin]. Razorpay's POS Bridge was
   seen only in a third-party write-up (start by API, poll, cancel) [Ws rzp-pos-bridge].
3. **Wired ECR**: a serial or USB link from the PC to the machine (Paytm lists a DLL for Windows and Linux on named Pax and Ingenico models) [Ws paytm-edc-search]. Fast and offline-capable on the LAN side, but a
   native driver per acquirer and a Windows-only practical path [J].

**Behaviours a vendor integration forces on the UI** [W zoho-paytm; W pine-cloud]: the QR or request **expires** (ten minutes in Zoho's Paytm flow; `AutoCancelDurationInMinutes` at Pine Labs); a cancel is a
**two-step** act (cancel on the machine, then in the POS, since the status does not update by itself); a split tender is one request per part. The till needs timers, a manual "mark failed / retry" and a way
to recover a payment taken but not recorded (the machine charged, the PC crashed).

**What may never be stored, logged, synced or sent.** From 1 January 2022 only card issuers and networks may store actual card data; anything else already held was to be purged; the **limited data** others
may keep for tracking and reconciliation is **the last four digits and the issuer's name** [W rbi-cof21]. The July 2022 circular repeats the bar from 1 October 2022 and lets a merchant or aggregator keep guest-
checkout data for settlement for up to T+4 days or the settlement date, whichever is earlier, then purge it [W rbi-cof22]. An EDC takes the card; the till should never see it. [W]
Design consequence [J]: a **PAN sentinel** (a 13 to 19 digit run that passes the Luhn check) is refused or masked in every free-text field a cashier can type into (reference, narration, customer note) and is
asserted absent from the outbox and backups; a vendor SDK that returns a masked card number is stored as last four plus issuer only.
**Data in India.** The RBI's FAQ on storage of payment system data (2019) says payment system providers and the intermediaries, gateways and vendors they engage must keep the **entire payment data in systems
located only in India**; it does not name merchants [W rbi-localisation]. Comsky's backend would hold payment references and customer phone numbers; where the VM sits is a fact I have not checked
[unverified]. PCI DSS and the card brands' own merchant rules were **not** researched; the safe design is to hold no card data at all [J].

### 1.6 Cash, caps, rounding and the digital-acceptance duty

- **Cash cap.** Section 269ST of the Income-tax Act, 1961 bars receiving Rs 2 lakh or more in cash from one person in a day, in one transaction, or for one event or occasion; the penalty is the amount received
  [Ws it-269st]. The Income-tax Act, 2025, in force from 1 April 2026, renumbers it, reportedly as section 186 (the page I read maps the Bill's clause 186; one search summary says 221) [W it-2025; Ws it-269st]. A jewellery or electronics till should **warn at the cash
  tender, aggregated per customer per day** where the customer is known. [Ws][J]
- **Digital-acceptance duty.** Section 269SU (Rule 119AA) requires a business whose turnover in the preceding year exceeded **Rs 50 crore** to offer a RuPay debit card, UPI and UPI QR, in addition to any other
  mode, with a penalty of Rs 5,000 a day; a business-to-business seller with no retail customers and at least 95% non-cash receipts is exempt per CBDT Circular 12/2020 [W cleartax-269su]. A supermarket chain
  crossing Rs 50 crore needs UPI and RuPay on every till. The CGST Act also has a section 31A on a digital-payment facility for registered persons above a prescribed turnover; the retail study could not find the notification that fixes the class [R pos-retail-market §6 row 18]. How the 2026 MDR interacts with the duty and with the statute that once barred charges on these modes (section 10A of the Payment and Settlement Systems
  Act) is **not settled in anything I read**: one source says 10A was amended in 2026 [W terra-mdr]. [W][unverified]
- **Rounding.** Section 170 of the CGST Act rounds tax, interest, penalty, refunds and other sums payable or due under the Act to the nearest rupee, fifty paise or more going up [Ws cgst-170]. I found no rule
  that prescribes rounding of an **invoice total**; the repo rounds the party leg to the whole rupee and books the difference to a Round Off ledger found by name [R pos-code-audit §1.5]. A cash till may
  want to round to the nearest 5 or 10 paise or the rupee; that is a shop choice, not a statutory one, **except** that it must not push a bill above what the law allows (Legal Metrology, section 3.7) [J].
  Round-down is always safe; round-up on MRP goods is not [J].
- **Change and over-tender.** Cash over-tender gives change; UPI, card and voucher tenders must **not exceed the balance due** (an over-tendered UPI is a refund problem), a rule the till enforces by capping
  the non-cash tender at the amount outstanding [J].

### 1.7 Credit (udhaar), gift vouchers, store credit and loyalty

- **Credit sales** are the bill-wise engine's strength: the customer is the party and a New Ref is named after the voucher number [R pos-code-audit §1.6]. Collection of dues by UPI should print or show a QR whose
  `tr` is the bill reference, so the payment can be matched to the open bill (route C or E). No regulatory source for trade credit was needed; the credit-limit prompt exists [R].
- **A store gift voucher** is, for GST, **not itself a supply**: CBIC Circular 243/37/2024-GST (31 December 2024) treats the sale of a voucher as a transaction in an actionable claim, taxes the **underlying
  supply when the voucher is redeemed**, values it under Rule 32(6) at the money value of what it is redeemed for, and leaves breakage on unredeemed vouchers untaxed; the Finance Act 2025 omitted the old
  time-of-supply sections 12(4) and 13(4) from 1 October 2025 (Notification 16/2025-CT) [W taxclue-voucher]. I read these through commentary, not the circular. So the till books a voucher sale to a
  **liability ledger**, books the redemption as a tender against a taxable invoice, and reports the liability at close [J]; **a CA must confirm the treatment** before it ships.
- **Store credit from a return** is a credit note (Rule 53(1A): nature of document, a consecutive serial, and the serial number and date of the original invoice) issued by the 30 November following the end of the
  financial year of the supply (Section 34(2), amended by the Finance Act 2022 from 1 October 2022) [W cbic-rules; Ws s34-time]. The credit note's value is a liability the customer can spend; the same liability
  ledger carries it [J]. The code audit already requires the return to store the original bill's id [R pos-code-audit §5].
- **Loyalty points.** Points redeemed against a bill are best booked as a **discount shown on the invoice before tax**: section 15(3)(a) leaves out of the value of supply a discount given at or before the
  supply and recorded on the invoice [W cgst-15]. Whether points are a discount or a payment for GST is **not settled in anything I read** [unverified]; earning points is not a tender. A points wallet that
  can be bought or transferred is stored value (section 1.4).

### 1.8 What the tender record should hold

[J, built on the code audit's contract lane] A `Tender` is `{ mode, amount, at, state, source, ref, provider?, terminalId?, tr?, cardLast4?, issuer?, approval?, rrn?, voucherCode?, mdrPaise? }`; `ref` is the bank
reference (UTR or RRN) as the provider or the cashier gave it; `tr` is our reference (the draft id then the bill number); `mdrPaise` is filled when the provider reports the deduction or computed from a **dated rate
table** (the 2026 framework as data with an effective date, never in code, the repo's rule for statutory figures). Split tender is a list of tenders that sum to the bill; the invoice prints each with its
reference; each UPI tender is one payment (the Rs 2,000 MDR line applies **per payment**, and the till must not split one payment to game it) [W npci-mdr Q35; J]. The record is part of the voucher's `pos`
object and must be carried on every alteration (the resave rule) [R pos-code-audit finding 3].

---

## 2. Hardware, and how a web or Electron app reaches it

**The code today.** Every receipt is `window.print()` on A4, A5 or Letter [C `src/ui/reportTools.ts:275`, `:303`; `src/ui/print/PrintVoucher.tsx:608`, `:676`; `src/types.ts:186`]. The desktop bridge exposes only
backup, `pdf.save` and `share.open` [C `electron/preload.cjs:6-43`], each a pure validator in `exportBridge.cjs` because the renderer is not trusted [C `electron/exportBridge.cjs:8-16`]. The window loads
`dist/index.html` from disk (or `localhost:5173` in development), with context isolation on and Node integration off [C `electron/main.cjs:188-190`, `:212-214`]; Electron is `^31.4.0` [C `package.json:32`].
No grep finds a serial, USB, HID or Bluetooth call, a kiosk flag or a single-instance lock [R pos-code-audit Appendix B]. The pure scan engines (`scanInput.ts`, `scanResolve.ts`, `scanDevice.ts`) and the label
engines (`labelJob.ts`, `labelRender.ts`) are at `HEAD` [C `ls src/core`]; `electron/labelLang.cjs` is the TSPL emitter and `electron/labelBridge.cjs` (slice T-7a) does not exist yet [C].

### 2.1 The devices, their interfaces, and what I could confirm

| Device | Interfaces and protocol | What I found (tag) | Notes |
|---|---|---|---|
| **80 mm thermal receipt printer** (TVS RP 3160 Gold is the Indian example) | USB and serial, a cash-drawer port; some listings add Ethernet; 203 dpi; 4 MB flash; full, partial and tear-off cut; 160 mm/s | the maker's store page lists USB / Serial, 203 dpi, 80 mm, 160 mm/s, a drawer port; Ethernet is **contested** between the store page and a locator page; ESC/POS compatibility is **not stated** on the pages I reached; store prices Rs 10,499 to Rs 14,995 on different listings [Ws tvs-store, tvs-store2] | confirm the command set from the datasheet before a promise [unverified] |
| **58 mm thermal printer** (counter or handheld) | USB, Bluetooth (classic serial or low energy), sometimes built into an Android POS (Sunmi V2 Pro lists a built-in 58 mm printer at 70 mm/s) | printable width 48 mm, 384 dots a line at 8 dots/mm on several spec sheets [Ws spec-58]; Sunmi's sheet lists the printer and Android 7.1-based SUNMI OS [Ws sunmi-v2] | cheap Bluetooth units are commonly classic serial [R mobile-tech-options §3.10, unverified] |
| **Cash drawer** | an RJ11 or RJ12 cable to the printer's drawer port; the printer pulses a solenoid | Epson's pulse command takes a pin selector (pin 2 or pin 5) and on and off times in units of 2 ms; do not fire it repeatedly, it can overheat the solenoid [Ws epson-pulse]; a third-party code list gives `1B 70 00` and `1B 70 01` and says commands vary by model [W tabshop-esc] | the drawer has no "open" signal of its own; a drawer-open **event log** is the control |
| **Customer-facing display** | a 2 x 20 character pole display on serial or USB driven by ESC/POS-like or CD5220 commands; or a second screen | pole displays listed as ESC/POS and CD5220 devices, clear-screen codes vary by model [Ws pole-display]; **second screen:** Electron's `screen.getAllDisplays()` and `display-added` events let a `BrowserWindow` open at another display's bounds [W electron-screen]; the web Window Management API (`getScreenDetails`) is experimental, secure-context only and not Baseline [W mdn-wm] | a second Electron window is the cheap route [J] |
| **Weighing scale** | RS-232 serial (called the most common in India by one vendor), USB as a virtual serial port, Ethernet; a stable-reading flag, tare, a price-per-kg multiply; price-embedded barcode labels | LogicERP's page (vendor marketing) lists RS-232, USB-as-COM and Ethernet and names Mettler-Toledo, Avery, Essae, Phoenix and AND, and says compatibility lists are on request [W logic-scale]; **no maker's serial protocol was found** [Ws scale-search]; the scan spec already defines scale label profiles [R batch-serial-labels-spec §14.4] | a scale is a per-model profile; the owner's scale decides |
| **Barcode scanner** | keyboard wedge (USB or Bluetooth HID), 1-D laser, CCD or 2-D imager | the detector, the resolver and the setup screen are specified and mostly built as pure engines [R batch-serial-labels-spec §14; C `src/core/scanInput.ts`] | a camera scan needs a secure context [R batch-serial-labels-spec §14.8] |
| **Card terminal, soundbox** | see section 1 | Paytm's wired ECR lists Pax A910, Pax A50 and Ingenico DX8000 [Ws paytm-edc-search]; Razorpay's soundbox models are named on its page [W rzp-soundbox] | partner hardware, leased by the acquirer |
| **Kitchen printer** | a network ESC/POS printer on TCP 9100, or an impact printer | TCP 9100 is the usual raw port for network receipt printers [R labels-and-scanning §2.4 path 4] | impact (dot-matrix) kitchen printers are common in kitchens for heat and grease [J, unverified] |
| **KDS tablet** | a browser or app on the store LAN | not a device; a screen of the app | needs the store hub (section 2.5) |
| **Touch PC or Android tablet** | Windows all-in-one or tablet; Android tablet | see the feasibility matrix in section 2.5 | Windows first (below) |

### 2.2 The receipt itself: widths, commands, the rupee sign and Indian scripts

**Widths.** A 58 mm roll prints 48 mm, 384 dots a line; an 80 mm roll commonly prints 72 mm, 576 dots (one maker's sheet offers 64 mm, 72 mm or 54 mm settings) [Ws spec-58]. Characters per line depend on the
font cell: 12-dot cells give 32 columns at 384 dots and 48 at 576; narrower fonts give 42 or more [Ws spec-58; M]. The repo's narrowest print layout is an A5 compact set; there is **no 58 or 80 mm sheet, no
automatic-height receipt model and no thermal font** [R pos-code-audit §8]. A receipt model is therefore new work in `print/` and `printConfig.ts` (the `PaperSize` union is `'A4' | 'A5' | 'Letter'`
[C `src/types.ts:186`], and the bridge's own list repeats it [C `electron/exportBridge.cjs:24`, `electron/preload.cjs:36`]).

**The commands a receipt uses.** ESC/POS is Epson's command language that most receipt printers imitate; **how much each model implements varies** [W tabshop-esc]. Epson's own reference refused my fetch (HTTP
403), so the command facts below come from a third-party code list [W tabshop-esc], an open-source library's documentation [W python-escpos] and search summaries of Epson's pages [Ws epson-pulse, epson-cr]:

| Function | What I could confirm | Tag |
|---|---|---|
| Initialise | `ESC @` (`1B 40`) | [W tabshop-esc] |
| Drawer pulse | `ESC p m t1 t2`: `m` 0 or 1 picks pin 2 or pin 5, on-time `t1 x 2 ms`, off-time `t2 x 2 ms`; pins 2 and 5 cannot pulse together; some makers use `ESC x` instead | [Ws epson-pulse]; [W tabshop-esc] |
| Cut | a full or partial cut exists on most models; the byte form differs (a code list gives `ESC i` and `ESC m` for partial cuts on some printers; `GS V` is the Epson-style cut I know but did not see confirmed on a fetched page) | [W tabshop-esc]; [J, unverified for `GS V`] |
| Text styles | print mode bits for double height, bold; alignment, font A or B, size multipliers 1 to 8, underline, invert | [W tabshop-esc]; [W python-escpos] |
| Code pages | a code-table selector and an international character set exist (`ESC t` is the selector I know; it was not seen on a fetched page [J, unverified]); Epson lists India-specific code page numbers (Devanagari and Marathi among them) without showing their glyphs; **the rupee sign was not confirmed in any standard table**; a developer thread reports it missing | [Ws epson-cr]; [Ws rupee-thread] |
| Raster image | `GS v 0` is the default raster implementation in the library; `GS ( L` graphics and `ESC *` column images are the others; tall images are fragmented (default 960 rows) | [W python-escpos] |
| QR | printers may draw a QR natively or the host sends it as an image | [W python-escpos]; the native command number not confirmed here [J, unverified] |
| Status | real-time status commands exist on Epson-compatible printers (paper out, cover open, drawer state); **not read** | [J, unverified] |

**The rupee sign and Indian scripts [J].** A receipt in Hindi, Marathi, Tamil or Bengali, or with a rupee sign, cannot rely on the printer's own tables. The proven pattern is to **render the line to a bitmap
and send it as a raster**; the Flutter package `thermal_unicode_print` states the reason (most receipt printers have only a few Latin code pages, so other scripts go as bitmaps) and renders line by line to keep
memory flat [Ws thermal-unicode]. The repo already loads IBM Plex (including the rupee glyph) for paper [R labels-and-scanning §3.6]; the label engine already turns a drawing into a 1-bit bitmap with a
printer-dots layout [R batch-serial-labels-spec §13.1]. Two honest alternatives for a receipt that is ASCII only: print "Rs." in native text (fast, always right), or draw only the rupee-bearing lines as
raster.

**Data volume and time [M, Appendix A].** At 8 dots/mm an 80 mm receipt is 72 bytes a row; a 150 mm receipt is 1,200 rows, **86,400 bytes (84 KiB)**; 58 mm is 48 bytes a row and **57,600 bytes (56 KiB)**; a 300 mm
80 mm receipt is 168 KiB. A native-text receipt of 40 lines by 48 columns is about **2 KB**, forty times smaller.

| Link | 150 mm, 80 mm raster (84.4 KiB) | 150 mm, 58 mm raster (56.3 KiB) |
|---|---|---|
| USB (12 Mbit/s raw rate, ignoring overhead) | 58 ms | 38 ms |
| Serial at 115,200 baud | 7.5 s | 5.0 s |
| Serial at 19,200 baud | **45 s** | 30 s |
| Serial at 9,600 baud | 90 s | 60 s |
| Mechanism at 160 mm/s | about 0.9 s of paper | about 0.9 s |

Baud defaults of Indian printers were **not found** [unverified]; Bluetooth throughput was not measured (low-energy writes are chunked and plausibly several seconds for 84 KiB [J, unverified]). **Rule [J]: native
text for the body (an item row costs about 50 bytes), raster for the logo, the QR (or the printer's own QR), and any line whose characters the printer cannot print; a serial or Bluetooth printer defaults to
text-only with "Rs.", USB and LAN may use full raster.** The printer profile carries `columns`, `codepage`, `rupee: 'text' | 'raster'`, `cut`, `drawerPin`, `link`, `baud`.

### 2.3 The cash drawer

The drawer is opened by the printer, usually after a cash tender (the printer's driver can be set to open it before or after a job; a raw job sends the pulse itself). Three controls matter more than the
pulse [J]: (1) **drawer opens are events** (sale with cash, "no sale" with a reason, shift open and close) written to a device log with operator and terminal, because a drawer open is **not a data write
today** and so is invisible to the change-log [R pos-code-audit §9]; (2) a "no sale" open is a **right** a cashier does not have by default [R pos-code-audit §9]; (3) the pulse timing is a profile constant
from a short allow-list, never a value the renderer sends (an on-time too long overheats the solenoid [Ws epson-pulse]).

### 2.4 Kitchen printers, the customer display, scales and scanners

- **Kitchen printers.** A KOT is an ESC/POS job to a **network** printer on TCP 9100 from the machine that holds the order; the kitchen has several (one per station). From a counter PC (Electron) this is a
  main-process TCP write to an **allow-listed** host, exactly the label bridge's `addNetwork` pattern (a private address only, port 9100 to 9109, stored by main, never named by the renderer) [R
  batch-serial-labels-spec §13.6]. From a **tablet browser** it is impossible directly (section 2.5). KOT routing, courses and modifiers are the restaurant study's matter [R pos-restaurant-market §3.3].
- **Customer-facing display.** In Electron a second `BrowserWindow` on the second display, fed by the main window over IPC (`screen.getAllDisplays`, `display-added`, `display-removed`) [W electron-screen]; it
  shows items, the total, the **dynamic QR** and the payment state. A pole display is a serial device with model-specific codes [Ws pole-display] and is a later, lower-value add. A browser-only second screen
  needs the experimental Window Management API on HTTPS [W mdn-wm].
- **Scales.** Read in Electron through **Web Serial**, which Electron supports through the main process answering `select-serial-port` (and keeping the grant itself, since by default a grant lives only as long as
  the `WebContents`) [W electron-devices], rather than a native `serialport` module that must be rebuilt for each Electron version [J]. Each scale needs a profile: baud and framing, a line parser that extracts
  weight and the stable flag, tare handling, a unit, and **a simulator** for tests because no maker's protocol was found. The price-embedded label formats are the scan spec's `scale` profiles [R
  batch-serial-labels-spec §14.4]. **Legal Metrology:** an instrument used in trade must be verified and stamped; reverification is due every 12 or 24 months by type (a counter machine is on the 24-month list,
  other weighing instruments on the 12-month list as one secondary source tabulates it), with first-offence fines of Rs 2,000 to Rs 10,000 for using an unverified one and state fee schedules [W ssrana-lm].
  The POS does not certify a scale; it should **record the scale's verification due date** and warn near it [J]. Rule 18(7) of the Packaged Commodities Rules also requires retailers of goods sold by weight or
  volume to keep an electronic machine of accuracy class III with a printed receipt for customers to check [W lm-pc].
- **Scanners.** Cross-reference only: the detector, the resolver, the Alt-keypad and Caps Lock traps and the setup screen are in `batch-serial-labels-spec.md` §14 [R]. A POS-specific point: a scan on the POS screen
  must be handled **outside the per-keystroke React path** (the code audit's finding 5 [R pos-code-audit §0]); the spec's `ScanInput` already coalesces a burst into at most two state changes [R
  batch-serial-labels-spec §14.3].

### 2.5 How an app reaches a printer: the paths, and which works where

The label study's seven paths are the right catalogue [R labels-and-scanning §2.4]; the receipt-specific points are below, and a new one (the printer pulls its jobs) is added.

| Path | How it works | Silent? | Needs | Evidence |
|---|---|---|---|---|
| **R1 Browser dialog** with `@page { size: 80mm auto }` | `window.print()` to the thermal driver | no, a dialog per bill | the driver installed; **a roll is one long page, and drivers differ on honouring the size** | [R labels-and-scanning §2.4 path 1, "anecdotal"] |
| **R2 Chrome `--kiosk-printing`** | the shortcut flag prints to the default printer with no dialog | yes, on Windows | the printer set as the default; behaviour **changed between Chrome versions**, one vendor says it is not recommended | [Ws kiosk-printing; R labels-and-scanning §2.4 path 2] |
| **R3 Electron `webContents.print({ silent: true, deviceName })`** | the system driver, no dialog; `getPrintersAsync()` lists printers | yes | options include `silent`, `deviceName` (the system name), `pageSize` (a name or `{ width, height }`, **units not stated**, minimum 353 microns), `margins` (pixels), `copies`, `dpi`, `printBackground`; a callback gives success and a failure reason | [W electron-print]; whether it honours a CSS `@page` size is **untested** [R labels-and-scanning §2.4 path 3] |
| **R4 Electron raw ESC/POS** | main writes bytes: **TCP 9100** to a network printer; the **Windows spooler with the RAW data type** to a USB or driver printer (the spec decided on a bundled, hash-checked PowerShell helper); **Web Serial** to a COM or paired Bluetooth-serial printer | yes, with status | a printer profile; the helper's cost per job (below) | [R batch-serial-labels-spec §13.6]; [W electron-devices] |
| **R5 Browser WebUSB / Web Serial / Web Bluetooth** | a chooser per device, HTTPS only | no | **WebUSB on Windows needs the WinUSB driver bound to the device** (a vendor printer driver blocks it; the page suggests Zadig for development); **on Android the CDC-ACM class is not available** to WebUSB; Web Bluetooth is low-energy only; Web Serial reaches paired Bluetooth-serial printers on desktop | [W chrome-webusb]; [R labels-and-scanning §2.4 path 5] |
| **R6 Local agent (QZ Tray)** | a helper app the page reaches over a local WebSocket; prints raw ESC/POS and pixel jobs; silent only when requests are signed | yes, signed | a per-PC install and a certificate; the source is LGPL 2.1, the qz.io binaries carry a restriction that steers shops to paid support, and the vendor store listed Premium Support at USD 599 for the first year (a 2020 licence page and a store snapshot) | [W qz-licence; Ws qz-search]; [R labels-and-scanning §2.4 path 6] |
| **R7 The printer pulls jobs** | **Star CloudPRNT:** the printer polls an HTTP(S) server (default poll 5 s on the listed models, 10 s recommended for remote use; an MQTT variant cuts the delay), fetches the job and confirms with a DELETE; **Epson Server Direct Print** is the same idea on TM-Intelligent printers with ePOS-Print XML | no dialog at all | **CloudPRNT or Server-Direct-Print printers** (Star mC-Print and TSP lines; Epson TM-Intelligent), internet at the shop, a server | [W star-about; Ws star-protocol; Ws epson-sdp] |
| **R8 Epson ePOS SDK for JavaScript** | the page talks to the printer's own web service | yes | a certificate on the printer: an HTTPS page **cannot** call a plain-HTTP printer (mixed content), so Epson publishes secure hostnames of the form `<hash>.omnilinkcert.epson.biz` on port 8043 for models with automatic certificates | [Ws epson-sdk]; [W mdn-mixed] |
| **R9 Android print service or native shell** | RawBT presents Bluetooth, USB and 9100 printers as a print destination, and says it can be driven from a site; a Capacitor or React Native shell uses a plugin | dialog or silent by plugin | an app per tablet; **classic Bluetooth is Android-only** (iOS needs MFi) | [Ws rawbt]; [R mobile-tech-options §3.10] |

**The mixed-content trap (it kills most "tablet prints through the counter PC" ideas).** A page served over HTTPS may not fetch an `http://` URL; loopback addresses (`localhost`, `127.0.0.1`) count as secure origins, and the page lists `fetch()` and XHR to an `http://` URL among the blocked requests; it does not discuss a LAN address, which is neither loopback nor HTTPS, so it is blocked too [W mdn-mixed; J]. A tablet that loads the app from `https://books.example` therefore cannot call `http://192.168.1.20` on the store LAN. The routes out [J]: (a) the tablet loads the app **from the
hub itself** over plain HTTP on the LAN (then service workers, install and the camera, which need a secure context, are off); (b) the hub serves HTTPS with a certificate the tablets trust, which needs a real
hostname (the Epson secure-hostname trick is the precedent [Ws epson-sdk]); (c) every device talks **outbound only** to the cloud and a printer or hub pulls its work (R7), which fails offline; (d) a native shell
that is not subject to browser rules. **This is the single hardest decision for a multi-device restaurant and it belongs to the owner (section 5.5).**

**Feasibility, by platform [J on the sources above].**

| Capability | Windows Electron | Chrome desktop on HTTPS | Android Chrome / PWA | Android native shell | iPad or iPhone Safari | Plain HTTP site (today) |
|---|---|---|---|---|---|---|
| Print dialog to a thermal driver | yes | yes | clumsy (system print sheet) | yes by plugin | AirPrint printers only | yes |
| **Silent** print, named printer | **yes** (R3) | no | no | by plugin | no | no (kiosk flag on Windows) |
| Raw ESC/POS to a USB printer with a vendor driver | **yes** (spooler RAW, R4) | no (WebUSB needs WinUSB) | no | by plugin | no | no |
| Raw to a LAN printer on 9100 | **yes** (TCP) | no | no | by plugin | no | no |
| Raw to a paired Bluetooth-serial printer | **yes** (Web Serial) | Web Serial | no | **yes** | no | no |
| Cash drawer pulse | yes (raw job or driver setting) | no | no | by plugin | no | no |
| Serial scale | **yes** (Web Serial) | Web Serial | no (CDC-ACM) | by plugin | no | no |
| Second screen | **yes** | experimental API | no | no | no | no |
| Camera scan | check the secure context in the packaged build [R batch-serial-labels-spec §14.8] | yes | yes | yes | yes | **no** |
| Acts as a store hub for tablets | **yes** (mixed-content cost above) | no | no | no | no | no |

**Conclusion [J]:** the **counter is a Windows machine running the Electron app**; an Android or iPad device is an order-taking or owner device, printing nothing itself except through a hub, a CloudPRNT-class
printer, or a native shell. This matches the code audit's platform order (Windows first) [R pos-code-audit §15 item 9] and the mobile study's note that classic Bluetooth printing belongs to a native shell
[R mobile-tech-options §3.10]. It is a hypothesis about the owner's customers, **to be tested by asking them what they run** (section 5.7).

### 2.6 The device bridge: one pattern for the receipt, the drawer, the display and the scale

Model: `electron/exportBridge.cjs` and the label bridge's design [C; R batch-serial-labels-spec §13.6]. The renderer is not trusted; a handler is a pure factory with its collaborators passed in, tested against a
fake `electron`; a request is **validated, not repaired**; one job at a time; the answer says "sent to the printer", never "printed" [C `electron/exportBridge.cjs:8-16`; R batch-serial-labels-spec §13.6].

**Proposed handlers [J] (all new, none built):**

| IPC | Renderer sends | Main does | Refuses |
|---|---|---|---|
| `devices.printers()` | nothing | returns `{ id, name, kind: 'system' \| 'network' \| 'serial', status? }` from the live `getPrintersAsync()`, the allow-list and the enumerated ports | any host, path or port not in main's own list |
| `devices.addNetwork` / `removeNetwork` | nothing (main opens its own small window for host and port) | stores a private-address host and port 9100 to 9109 in `userData/printers.json`, written only by main | a public address (also at connect time), a hostname that resolves to one, any other port |
| `receipt.print(job)` | `{ printerId, kind: 'text' \| 'raster' \| 'page', widthDots, copies, cut: 'none' \| 'partial' \| 'full', drawer: bool, lines: [...] \| bitmap: { widthBytes, heightDots, dataB64 } }`, a **structured receipt**, never a byte | main builds the ESC/POS bytes itself (`escposLang.cjs`, the twin of `labelLang.cjs`), writes them in one spooler write, TCP stream or serial write, **polls status before and after** where the printer reports it | unknown keys, wrong types, a bitmap whose length is not `widthBytes x heightDots`, widths other than 384 or 576 (or a profile-listed width), copies over a ceiling, a total over a size ceiling, any command string |
| `drawer.open(printerId, reason)` | a printer id and a reason code | sends the profile's fixed pulse (pin and times from the allow-list) and writes a **drawer event** to a local log | any timing from the renderer; more than one pulse inside a short interval |
| `display.open()` / `display.update(state)` | the customer-facing view's state (items, total, QR payload, payment state) | opens a window on the second display and relays the state | a URL, a script, any node integration |
| `scale.open(portId)` / `scale.onWeight` | a port id | opens the Web Serial port main selected, parses by the profile, emits `{ kg, stable }` | a baud or a parser from the renderer |

**The emitter allow-list [J].** `escposLang.cjs` writes only: initialise, text in the profile's code page (printable ASCII plus the profile's extra characters), line feed, alignment, bold, size, a raster
block, a QR block if the profile says it is native, a cut, a drawer pulse, a feed. It never writes the printer's **memory-programming and macro commands**, a logo-download command, a baud or
code-page *persisting* setting, or any byte the renderer supplied outside a text run or a bitmap (the TSPL manual lists commands that change or restart the printer and the label spec closes that path the same
way [R labels-and-scanning §2.6]; for ESC/POS I have no manual to enumerate them [unverified]). The renderer-side job builder is a pure `src/core/receiptJob.ts`, **mirrored** in the CJS validator and
asserted equal on one corpus, as `labelJob.ts` is [C `electron/labelLang.cjs:26-27`].

**Costs the label spec does not have.** A label job is occasional; a receipt is **every bill**, so latency matters: (1) a Windows spooler write through a **PowerShell process per job** pays PowerShell's start
(on the order of hundreds of milliseconds [J, unmeasured]); a **long-lived helper** reading jobs on stdin, started once, hash-checked once and restarted on failure, avoids it [J]; (2) the first byte to paper
should be under a second after Accept, a figure to gate on real hardware; (3) a receipt must be **reprintable** from the saved voucher with a "DUPLICATE" mark and a reprint count, neither of which exists
[R pos-code-audit §8]; (4) printing must never block Accept: the sale is saved first and the print is a queued job with a visible retry [J].

**Packaging and honesty.** Only a **rebuilt installer** carries any of this; the Phase 29 PDF bridge **has never run on a real Electron binary** and the Windows installer is unsigned [C `CLAUDE.md`, Phase 29]; so
the receipt bridge's first real run is a gate, not an assumption. Windows code-signing and SmartScreen reputation are a shop-PC support issue for any helper binary; the spec's PowerShell route avoids an
unsigned executable at the cost of antivirus suspicion [R batch-serial-labels-spec §13.6].

### 2.7 What the owner can test on a bench before anything is sized (the hardware list)

One Windows PC with the Electron app; one 80 mm USB receipt printer with a drawer port (the TVS RP 3160 Gold or an Epson TM-T82-class unit) and a drawer; one 58 mm Bluetooth printer; one network ESC/POS
printer; one serial or USB scale with its manual; one wedge scanner (and a 2-D imager); one card machine on each acquirer's test mode; one soundbox. For each: the **exact driver settings** that give one clean
receipt, the time from Accept to first paper, whether the cutter and drawer fire, how `Rs.`, the rupee glyph and a Hindi line print, and what a power cut mid-job leaves. Nothing in this study substitutes for
that list.

---

## 3. Compliance

> **Not legal advice, and not reviewed by a CA.** Statutory text below is paraphrased from what I read: the CBIC copy of the CGST Rules "as on 24 September 2021" (a convenience copy that says it has no legal force), a
> reproduction of the current Rule 46 on gstzen, the notifications as PDFs, and commentary pages where the primary text was out of reach. **Rates, thresholds and wording must live in dated data, never in code**
> (the repo's own rule for statutory figures), and every row in section 5.6 needs a CA's sign-off before a POS ships. The CBIC web page for Rule 46 failed a certificate check, so the amendments after 2021 (the
> web page lists some, for example one effective 1 November 2024) were **not** read [unverified].

### 3.1 What a counter bill is in law

| Situation | The document | Basis |
|---|---|---|
| A taxable supply, any buyer | a **tax invoice** with the Rule 46 particulars, issued before or at the time of delivery (goods) | s.31(1); Rule 46 [W cgst-31, cbic-rules] |
| A supply **under Rs 200**, buyer **unregistered** and **not asking** for an invoice | no per-bill invoice; **one consolidated tax invoice at the close of each day** for all such supplies (not available to a multiplex cinema's admission service) | s.31(3)(b) sets the Rs 200 line; Rule 46, fourth proviso, sets the conditions [W cgst-31, cbic-rules] |
| An exempt supply, or any supply by a **composition** dealer | a **bill of supply** (Rule 49); a composition dealer's bill carries the legend "composition taxable person, not eligible to collect tax on supplies" at the top, and a signboard says "composition taxable person" (Rule 5(1)(f), (g)); a bill of supply is not needed under Rs 200 on the same conditions | s.31(3)(c); Rules 5, 49 [W cgst-31, cbic-rules] |
| **Taxable and exempt goods on one bill** to an unregistered buyer | a single **invoice-cum-bill of supply** | Rule 46A [W cbic-rules] |
| A **return or price reduction** | a **credit note** with its own serial, naming the original invoice's serial and date (Rule 53(1A)); issued by 30 November following the financial year of the supply, or the annual return date if earlier | s.34; Rule 53 [W cbic-rules; Ws s34-time] |
| **Advance received** (an order deposit, a booking) | a **receipt voucher**; a **refund voucher** if the supply never happens | s.31(3)(d), (e) [W cgst-31] |
| **Restaurant service through an e-commerce operator** | the **operator** issues the invoice; the restaurant reports the supply in Table 8 of GSTR-1 and 3.1(c) of GSTR-3B; if the same order has goods, the operator is advised to bill them separately and the supplier remains liable for them | Circular 167/23/2021-GST, Q8 to Q11 [W circ167] |
| A **B2B** bill from a taxpayer above the e-invoice threshold | an invoice **with an IRN and signed QR**; one issued any other way is not treated as an invoice | Rule 48(4), (5) [W cbic-rules] |

**What the "Rs 200 slice" means for the till [J].** The consolidated daily invoice is an option for the smallest, unregistered, no-invoice-asked sales, not a licence to batch the day's bills: **every other bill
needs its own number**. The code audit reached the same position and called the summary-posting lane optional and later [R pos-code-audit §3.4 to §3.5]. A shop that wants to use it needs a separate series of
non-tax-invoice receipts below Rs 200 that roll up into one daily invoice, and the roll-up must show in Table 13 of GSTR-1 as one document. Because Rule 48(3) requires the serial numbers of invoices issued in a
tax period to be furnished through GSTR-1 [W cbic-rules], the **daily consolidated invoice is itself a numbered document in the series**. The simplest and safest default is: **every bill is an invoice**.

**Copies.** Rule 48(1) and (2) say an invoice for goods is prepared in triplicate (original for recipient, duplicate for transporter, triplicate for supplier) and for services in duplicate (original for
recipient, duplicate for supplier) [W cbic-rules]. The repo already has the `goods` and `services` copy presets and a `single` default [C `src/types.ts:190`; R pos-code-audit §8]. For a till, the supplier's copy is the saved voucher; a reprint is a **duplicate** and should say so [J].

### 3.2 Rule 46: the receipt fields, as the rule lists them

| Rule 46 clause | Particular | At a B2C counter [J] |
|---|---|---|
| (a) | supplier's name, address, GSTIN | the company's, printed always |
| (b) | a consecutive serial number, at most 16 characters of letters, digits, `-` and `/`, in one or several series, unique for a financial year | section 3.3 |
| (c) | date of issue | the device clock, shown, never carried over from the last voucher [R pos-code-audit §4] |
| (d) | recipient's name, address, GSTIN if registered | **ask for the GSTIN**, validate its check character [R CLAUDE.md Phase 24] |
| (e) | for an unregistered recipient and a taxable supply of **Rs 50,000 or more**: name, address, delivery address, state and code | **force capture** at or above the line |
| (f) | the same, below Rs 50,000, **if the buyer asks** | an optional capture |
| (g) | HSN code of goods or services; Notification 78/2020: **4 digits up to Rs 5 crore, 6 above**, and a taxpayer up to Rs 5 crore **may omit the digit count** on invoices to unregistered buyers | print the HSN per line anyway: the GSTR-1 HSN table needs it [R pos-code-audit §1.5] |
| (h), (i) | description; quantity and unit (UQC) | per line |
| (j), (k) | total value; taxable value after discount or abatement | a discount recorded on the invoice is left out of the value of supply (s.15(3)(a)) [W cgst-15]; a bill-level discount is **allocated across lines before tax** [R pos-code-audit §1.4] |
| (l), (m) | rate and amount of tax by head (central, state, integrated, cess) | per real slab, never a blended rate [R CLAUDE.md gotchas] |
| (n), (o) | place of supply with state name for an inter-state supply; delivery address if different | a walk-in is the supplier's own state [R pos-code-audit §1.5] |
| (p) | whether tax is payable on reverse charge | "No" |
| (q) | signature or digital signature | **not required on an electronic invoice** under the Information Technology Act (a proviso) [W cbic-rules] |
| (r) | a QR code with the IRN, where Rule 48(4) applies | the IRN QR is a different QR from a payment QR (section 3.4) |

Value of supply also includes incidental expenses the supplier charges for the supply, such as packing and commission (s.15(2)(c)) [W cgst-15]; the restaurant study applies this to packaging and
delivery charges [R pos-restaurant-market §4.5].

### 3.3 Numbering: Rule 46(b) and a multi-terminal, offline shop

**The rule (primary text).** A consecutive serial number, not exceeding sixteen characters, in one or multiple series, containing letters, numerals, hyphen and slash in any combination, unique for a financial
year [W cbic-rules, Rule 46(b); the same words in Rules 49 and 53]. The GSTN reading, via a search summary, is that a taxpayer starts a new series each 1 April [Ws iris-numbering].

**What follows [J].**
1. **A series per terminal satisfies the rule.** Several series are allowed; each must be consecutive and each number unique in the year. One voucher type per terminal gives one series per terminal with no
   clash and nothing for the claim machinery to do [R pos-code-audit §4]. Credit notes get their own series per terminal; Table 13 groups by type and window, so per-terminal types give clean rows [R
   pos-code-audit §4].
2. **"Consecutive" means no number is skipped silently.** The repo's cancel keeps the number as a shell, which is what the rule's logic needs; Undo, Delete and an Optional "held bill" each leave a hole or burn
   a number and must be closed off in POS mode [R pos-code-audit §4 table]. A **held bill is not a voucher** and takes no number. A number is taken **at tender finalisation** (the pay-first design in 1.3).
3. **Case does not distinguish numbers.** Since 1 June 2025 the e-invoice portal upper-cases the document number before computing the IRN, so `inv/1` and `INV/1` are the same document [R einvoice-gsp §3]. A
   shop that may one day be e-invoice-liable should generate **upper-case** numbers only and treat a case-insensitive duplicate as a clash. The repo's own check is `^[A-Za-z0-9/-]{1,16}$` with a
   `SCHEMA?` flag [C `src/core/gstr1.ts:313-315`].
4. **The 16-character budget** is in the table after this list; the FY need not be written in the number, and a number should start with a letter or a non-zero digit if the shop may become e-invoice-liable (details below the table).
5. **A replaced or reset terminal continues its series.** The counter is read from the vouchers themselves [R pos-code-audit §4], so a new machine must **pull the terminal's history before it bills** or ask
   the server for the next number; an offline new machine would restart at 1 and clash. This is the one place server numbering helps a device-owned series [J].
6. **Table 13.** The serial numbers issued in a tax period are furnished in GSTR-1 (Rule 48(3)) [W cbic-rules]; the repo already builds the documents-issued table by type and window and counts cancelled
   numbers [R pos-code-audit §0 item 1; C `src/core/gstr1.ts:424-477`]. A hundred series (twenty stores by five terminals) is a hundred rows, fine.
7. **Rule 47** (a time limit of 30 days for services; 45 for banks and similar) does not bind a till, which issues at the time of supply.

**The 16-character budget.** The table is arithmetic [M, Appendix A]; "ok" means inside Rule 46(b) and the repo's check.

| Pattern | Example | Length | Verdict |
|---|---|---|---|
| terminal, FY, 6-digit sequence | `T1/26-27/000123` | 15 | ok; the code audit's own example |
| store (5), terminal, 6-digit sequence | `MUM01/T3/000123` | 15 | ok, **no FY in the text**; the series window is the FY in the data |
| store (5), terminal, FY, 4-digit sequence | `MUM01/T3/26-27/0001` | 19 | **too long** |
| store (2), terminal, FY (4), 7-digit sequence | `S12T3/2627/0001234` | 18 | **too long** |
| store (2), terminal, hyphens, 7-digit sequence | `S12-T3-0001234` | 14 | ok; 7 digits reach 99,99,999 bills a year |
| store (2), terminal, `/`, 7-digit sequence | `S12/3/0001234` | 13 | ok |


A terminal doing 3,000 bills a day makes 1,095,000 a year, seven digits [M]. The **FY need not be written in the number**: the uniqueness is per financial year, and a counter that restarts on 1 April
with the FY held in the voucher's date is enough; writing it costs 5 characters a chain cannot spare [J]. A number should **start with a letter or a non-zero digit** if the shop may become e-invoice-liable;
the repo's check allows a leading zero, which is a schema question to confirm against NIC's current definition [unverified].

### 3.4 The B2C dynamic QR: the timeline, who it binds, and what it asks

| Date | Instrument | What it did | Source |
|---|---|---|---|
| 21 March 2020 | Notification 14/2020-CT, under the sixth proviso to Rule 46 | an invoice issued by a registered person whose aggregate turnover **in a financial year exceeds Rs 500 crore**, to an unregistered person (a "B2C invoice"), **shall have a dynamic QR code**, other than those under Rule 54(2), (3), (4), (4A) and a person under section 14 of the IGST Act; a **digital display** of the QR with a cross-reference of payment on the invoice is deemed compliance; in force **1 October 2020** | [W n14] |
| 30 September 2020 | Notification 71/2020-CT | "a financial year" became "**any preceding financial year from 2017-18 onwards**"; the start moved to **1 December 2020** | [W n71] |
| 29 November 2020 | Notification 89/2020-CT | **penalty waived** (section 125) for non-compliance from 1 December 2020 to 31 March 2021, on condition of compliance from 1 April 2021 | [W n89] |
| 23 February 2021 | Circular 146/02/2021-GST | clarifications (below) | [W c146] |
| 30 March 2021 | Notification 06/2021-CT | the waiver to **30 June 2021**, compliance from 1 July 2021 | [W n06-21] |
| 30 June 2021 | Notification 28/2021-CT | in supersession of 89/2020, the waiver for **1 December 2020 to 30 September 2021** | [W n28-21] |

Nothing later was found; the circular and notifications were read from the Council's and CBIC's PDFs [W]. **No change to the Rs 500 crore threshold was found** [unverified for after 2021].

**What Circular 146 clarifies [W c146].** The notification applies to a tax invoice to an unregistered buyer from a registered person with annual aggregate turnover above Rs 500 crore in any year from 2017-18; it
does **not** apply to a supplier that is an insurer, a banking company or other financial institution (including an NBFC), a goods transport agency, a passenger transport service or a multiplex cinema
admission, nor to OIDAR supplies by a person registered under section 14 of the IGST Act, nor to **exports** (they are e-invoiced and treated as B2B). The QR must contain, among other things, the supplier's
GSTIN, the supplier's UPI id, the payee's bank account number and IFSC, the invoice number and date, the total invoice value and the GST amount with its CGST, SGST, IGST and cess breakup, and **must be
scannable to pay**. Three alternatives count as compliance: (i) a dynamic QR displayed for payment and the customer pays another way, **if the invoice cross-references the payment** (transaction id with date, time,
amount and mode for UPI, card or net banking; for cash, the amount paid and the date); (ii) a payment made **before** the invoice is issued, with the same cross-reference; (iii) a collect or intent
payment through an app, with the cross-reference. **Where payment is made after the invoice, the supplier must put a dynamic QR on the invoice.**

**Meaning for the product [J].** The target customers (a single shop to a small chain) are not caught, because their turnover is far below Rs 500 crore; a hypermarket group is. The pay-first flow with a
printed payment reference (1.3) satisfies the rule in the pre-paid case and costs nothing; a **bill-first** flow (a restaurant, a credit sale) for a taxpayer above Rs 500 crore needs a real dynamic QR *on the
invoice* with the five content items, which is a PSP job. **Do not build the statutory QR now**; keep the invoice's `paymentRef` block (id, date, time, amount, mode) so it can print the cross-reference, and keep
`upi.ts`'s optional QR. Do **not** confuse the three QRs a bill can carry: the statutory dynamic QR (payment, large taxpayers), the **IRN QR** (e-invoice, Rule 46(r)), and an owner's static UPI QR (optional).

### 3.5 E-invoicing and the counter

| Question | Answer | Source |
|---|---|---|
| Who must e-invoice | aggregate turnover **above Rs 5 crore** in any year since 2017-18, from **1 August 2023** (Notification 10/2023-CT amending 13/2020: "ten crore" became "five crore"); no lower threshold had been notified as of the dates I could check, and talk of Rs 2 or 3 crore is unnotified; one 2026 page claiming a rise is AI-flagged and contradicts the rest, **ignored** | [W taxheal-n10, tally-einv]; [R einvoice-gsp §3] |
| Who is outside it | insurers, banks, financial institutions and NBFCs, goods transport agencies, passenger transport, multiplex cinema admission, SEZ **units** (not developers), government departments and local authorities | [W tally-einv]; [Ws n13-amend]; [R einvoice-gsp §3] |
| Which documents | B2B, SEZ and export supplies, as invoices, credit and debit notes; **B2C is not handled by the IRP** | [R einvoice-gsp §3] |
| 30-day limit | from 1 April 2025 the IRP will not take a document older than 30 days for aggregate turnover of **Rs 10 crore and above** | [W tally-einv]; [R einvoice-gsp §3] |
| An invoice without an IRN where one is due | **not treated as an invoice** (Rule 48(5)); and the e-invoice, once issued, carries the signed QR | [W cbic-rules] |

**For the till [J].** (1) Hold the company's e-invoice status as dated data (`liable from`). (2) A B2C walk-in bill is outside e-invoicing at any turnover. (3) The moment a GSTIN is entered on a bill for an
**e-invoice-liable** seller, the bill is a B2B invoice that needs an IRN: the POS must either call an IRP through a GSP (a partner: `docs/research/einvoice-gsp.md`) or refuse to print it as a tax invoice and say why.
The app has no such call [C `CLAUDE.md`, Known limitations]. (4) Each IRN is tied to an invoice number and cannot be reissued, so the pay-first rule (a number only after confirmation) matters more here.

### 3.6 Restaurant service: the options a restaurant has, and what a receipt must say

The restaurant study derived the detail; the key facts were re-read here from the primary documents [W gstservice-rates, ctr05-2025, circ167]:

| Option | Rate and condition | Receipt consequence |
|---|---|---|
| **Restaurant service, not at "specified premises"** | **2.5% central + 2.5% state (5%)**, a mandatory rate that cannot be replaced by the general entry, on the condition that **credit of input tax on goods and services used in supplying the service has not been taken** (the condition points to an Explanation I did not follow through); takeaway, delivery, cloud and central kitchens are restaurant service [W gstservice-rates; R pos-restaurant-market §4.1] | a tax invoice at 5%; no input credit, so input tax is a cost |
| **Restaurant service at "specified premises"** | the residual entry, **9% + 9% (18%)** with credit. From **1 April 2025** a premises is "specified" if hotel accommodation above **Rs 7,500 a unit a day** was supplied from it in the **preceding** financial year, or a registered hotel supplier filed an opt-in declaration between 1 January and 31 March of the preceding year, or a new registrant declared it within 15 days of the registration acknowledgment; an opt-out declaration uses the same window [W ctr05-2025] | the rate is a **company or outlet setting with an effective date**, never hard-coded |
| **Composition** | up to 5% of turnover with no input credit, a **bill of supply** with the composition legend, tax not collected from the customer, no inter-state outward supply; the turnover limit and the alcohol and aggregator edge cases are open in the restaurant study [R pos-restaurant-market §4.2] | **"Bill of Supply"** print, no tax lines, the Rule 5(1)(f) legend |
| **Through an aggregator** | the operator pays the tax and **issues the invoice** for restaurant service (from 1 January 2022); the restaurant reports it in Table 8 of GSTR-1 and 3.1(c) of GSTR-3B; goods in the same order stay the supplier's own invoice with the operator collecting TCS [W circ167] | a **"tax paid by the operator under 9(5)"** taxability class; the restaurant must **never print its own tax invoice** to the diner for that order [R pos-restaurant-market §4.3] |

Also on a restaurant or any food bill: the **14-digit FSSAI licence or registration number** must be declared on cash receipts, invoices, cash memos and bills. The commentary I read gives 1 January 2022 as the
start and says a 13 September 2022 order told officers to run special drives [W foodsafetyworks]; the restaurant study dates the original order 8 June 2021 [R pos-restaurant-market §4.7]; the FSSAI PDFs themselves
were not reachable. Service charge and the Consumer Protection (CCPA) guidelines of July 2022, and the collision with GST valuation, are in the restaurant study and **unsettled**; v1 ships the charge off and
with no GST treatment preselected [R pos-restaurant-market §4.4].

### 3.7 Legal Metrology at the counter

Read from the Department of Consumer Affairs' consolidated Packaged Commodities Rules (Rules 2011 as amended to 2022) [W lm-pc], paraphrased:

| Rule | Counter consequence [J] |
|---|---|
| **18(1)** no retailer may sell, display or store for sale a packaged commodity unless the package complies | outside the POS; a stock-intake concern |
| **18(2)** **no retail dealer or other person may sell a packed commodity at a price exceeding its retail sale price** | an **MRP guard**: a line whose selling price exceeds the stored MRP is blocked or needs a reason and a right; MRP is *inclusive of all taxes* (Rule 2(m), 6(1)(e)) [R labels-and-scanning §3.1] |
| **18(3)** when a tax on a pre-packed commodity is revised, the retailer may not sell above the revised retail price notified by the maker; the difference is capped at the tax change; a lower revised price always binds | when GST rates change (as on 22 September 2025 [R pos-restaurant-market §4.1]), packed stock bought earlier keeps its printed MRP; the guard uses the **batch's** MRP, not today's rate card [J] |
| **18(5)** no one may obliterate, smudge or alter the printed retail price | a shop sticker must not cover it [R labels-and-scanning §3.1]; a till cannot enforce it |
| **18(7)** retailers of goods sold by weight or volume, covered by GST, must keep a class III electronic weighing machine with a printed receipt for customers to check | the shop's own duty; the POS may remind |
| **6(11)** a unit sale price on the package (from 1 October 2022) | a shelf-label matter [R labels-and-scanning §3.1] |

**Rounding.** The older text of Rule 6(1)(e) spoke of rounding a printed MRP to the nearest rupee or 50 paise; the 2022 substitution says only that the MRP is stated in Indian currency [W lm-pc;
R labels-and-scanning §3.1]. **A bill-level round-up is the risk:** a basket of ten MRP-priced items totalling Rs 997.40 rounded up to Rs 998 would, read strictly, charge more than the sum of the
retail sale prices; rounding to the nearest rupee **down** only, or only when no packed item is on the bill, is the cautious rule [J]. Whether Rule 18(2) is applied to a bill total or to each pack was **not
answered by anything I read** [unverified].

### 3.8 Retention, records and the audit trail

- **GST records.** Section 36: books and records are kept until **72 months** after the due date of the annual return for the year, and longer for an appeal, revision or investigation (a year after final
  disposal if later) [Ws cgst-36]. Rule 57: records kept electronically need a **proper electronic back-up** from which information can be restored within a reasonable time, must be producible on demand in hard
  copy or an electronically readable form, and the person must supply file details, passwords and explanations of codes on demand [W cbic-rules]. The repo's **restore has never been tested** and auto-backup
  stops working near 290,000 bills [R pos-code-audit §0 item 7]: those are compliance gaps at POS volume, not only engineering ones.
- **Companies' accounting software.** Proviso to Rule 3(1) of the Companies (Accounts) Rules, 2014, for financial years beginning on or after **1 April 2023**: accounting software must record an **audit trail
  of each transaction, an edit log of each change with dates, and the audit trail cannot be disabled**; the auditor reports on it under Rule 11(g) [Ws icai-audit-trail]. A company customer (Pvt Ltd) will ask. The
  POS must therefore keep void, discount override, price override, return, no-sale and **reprint** as logged events with operator, terminal and time, and the log must not be switchable off [J]. Rule 3(5)
  (back-up of books on servers physically in India) and an eight-year retention figure were **not confirmed** [unverified].
- **Payment data and customer data.** Sections 1.5 and 4.

### 3.9 What a receipt must show, by vertical

| Item | Retail (GST) | Restaurant | Pharmacy | Source |
|---|---|---|---|---|
| Supplier name, address, GSTIN | yes | yes | yes | Rule 46(a) [W cbic-rules] |
| Document title: Tax Invoice / Bill of Supply / Invoice-cum-Bill of Supply / Credit Note | by situation (3.1) | Tax Invoice at 5% or 18%; Bill of Supply if composition | Tax Invoice; Bill of Supply for exempt items | Rules 46, 46A, 49, 53 [W cbic-rules] |
| Serial number, date | yes | yes | yes (and a **serial number** on the cash memo) | Rule 46(b), (c); Drugs Rule 65(4)(3) as read by the pharmacy study [R pos-pharma-market §3.9] |
| Buyer details | GSTIN if B2B; name, address, state when Rs 50,000 or more | same | same | Rule 46(d) to (f) [W cbic-rules] |
| Per line: description, HSN, quantity and unit, taxable value, rate, tax | yes | yes | yes, with batch, expiry and MRP on the line | Rule 46(g) to (m) [W cbic-rules]; [R pos-pharma-market] |
| Total, tax by head, round-off | yes | yes | yes | Rule 46 [W] |
| Composition legend | if composition | if composition | if composition | Rule 5(1)(f) [W cbic-rules] |
| FSSAI licence number | food items | **yes** | no | [W foodsafetyworks] |
| Drug **sale licence number** | no | no | **yes** | Drugs Rule 65(4)(3) [R pos-pharma-market §3.9] |
| Payment reference (txn id, date, time, amount, mode; cash amount and date) | optional; **required form of compliance** above Rs 500 crore | same | same | Circular 146 [W c146] |
| "Duplicate" on a reprint, original or duplicate copy marks | yes | yes | yes | Rule 48 [W cbic-rules]; none exists today [R pos-code-audit §8] |
| Consumer-complaint contact | on packed goods by the maker | no | no | [R labels-and-scanning §3.1] |

---

## 4. E-bills, WhatsApp and customer privacy

### 4.1 The channels, and what each needs

| Channel | In the repo today | What it needs | Cost and rules | Offline day |
|---|---|---|---|---|
| **Paper** (thermal, A4, A5) | A4, A5 and Letter only, through the print dialog [C `src/types.ts:186`] | the receipt model of section 2 | paper | yes |
| **`wa.me` click-to-chat** | `waLink` builds `https://wa.me/<7-15 digits>?text=...`; the app **opens** the person's own WhatsApp and sends nothing; the text is a letter, capped (3,500 characters), not an itemised receipt; a link cannot attach a file; the desktop bridge accepts only this link shape [C `src/core/share.ts:127-130`; `electron/exportBridge.cjs:146-176`; R pos-code-audit §8] | the customer's number (the phone normaliser exists, `share.ts:33`); the cashier's WhatsApp on the same machine | free; the **cashier's** account sends | works when the cashier's phone or WhatsApp Web is online |
| **E-mail (`mailto:`)** | the same Share panel, validated addresses, no attachment [C `electron/exportBridge.cjs:146-166`] | an address | free | as above |
| **Hosted e-bill page with a link** | none | our backend over HTTPS, a random, unguessable token per bill, an expiry, no login, the bill as HTML or PDF; the link goes into a `wa.me` text, an SMS, an e-mail or the UPI `url` field | our hosting; **personal data in a page** (section 4.5) | the bill is created locally and published when online; the link is printed as a QR only after publish [J] |
| **UPI `url` (the draft's Bill in the Box)** | not written (`upi.ts` leaves `url` out) [C `src/core/upi.ts:16-22`] | a QR that carries the hosted-bill link; the draft **requires payer apps to show a view-invoice option** when `url` is present, and the link must concern that transaction only [W npci-link §1.4] | free | needs a PSP-style QR and the hosted page; whether today's payer apps still show it is **unverified** |
| **WhatsApp Business Platform** (Meta's Cloud API, directly or through a provider) | none | a verified business account, an approved **template** for any message sent outside the 24-hour window, the customer's **opt-in**, a server that calls the API, a phone number dedicated to the account | per-message price (below) | needs internet at the sender |
| **SMS** | none | a DLT-registered **entity**, **sender header** and **content template** (below) | a provider's price (not researched) | needs internet at the sender |

### 4.2 Is an electronic bill a valid bill?

Rule 46's fifth proviso says a **signature or digital signature is not required on an electronic invoice** issued under the Information Technology Act, 2000, and Rule 49 says the same for a bill of supply [W cbic-rules].
Rule 48's copy marks (original for recipient, duplicate for supplier) and Rule 57's duty to keep a restorable electronic record and produce it on demand still apply [W cbic-rules]. **I read nothing that says a
WhatsApp message or an image counts as the "original for recipient"** [unverified]; the practice, and the safe reading, is that the **saved voucher is the supplier's record, the customer's copy may be paper
or electronic, and a reprint carries "duplicate"** [J]. A CA should confirm for each vertical (section 5.6).

### 4.3 WhatsApp Business Platform: what I could read

Meta's developer page [W meta-wa], paraphrased:
- Charging is **per delivered message** since 1 July 2025, by category and by the recipient's country code; messages from users to the business are free; undelivered messages are not charged.
- Categories: **marketing**, **utility** (transactional, for example order confirmations), **authentication**, **service** (non-template replies to a user), and a token-priced agent product.
- The **24-hour customer service window** opens when the user writes to the business and resets with each user message; inside it, non-template messages are allowed; outside it, **only template messages**; **you may
  only message users who have opted in**.
- A free tier of 1,000 service messages a month per business phone number; **the page says charging for service messages after that tier began on 1 October 2026**, and that **utility messages sent inside an open
  window, free since 1 July 2025, became chargeable again on 1 October 2026**. A product page I also fetched still says an in-window utility reply is free [W meta-wa; W wa-pricing-hi]. **These two statements conflict;
  treat in-window utility messages as chargeable until confirmed.**
- India is a standalone market on the rate card; its marketing rate rose on 1 January 2026; eligible India businesses can bill in INR, and **all accounts must be in INR by 31 December 2026** or Meta stops
  delivering from January 2027. The rate card itself is interactive and was **not readable**; third-party blogs put India **utility at about Rs 0.115 and marketing at about Rs 0.86 a message before 18% GST**
  [Ws wa-rates]. At that utility rate a shop sending one message per bill at 800 bills a day, 30 days, spends about **Rs 2,760 a month, Rs 3,257 with GST** (24,000 messages) [M, on blog rates].
- I did not confirm whether a utility template may carry a **document** (an invoice PDF) as its header, a size limit, or the exact opt-in wording Meta accepts for "gave a phone number for a bill" [unverified].

### 4.4 SMS and DLT

India's commercial-SMS rules require three registrations on a DLT platform: the **entity** (a Principal Entity ID), a **header** (six letters for transactional), and each **content template** with placeholders;
operators scrub every message against the template, and a mismatch is **blocked and never delivered**. The categories are transactional, service-implicit, service-explicit (needs opt-in) and promotional (barred from
9 pm to 9 am, needs opt-in) [W exotel-dlt]. A bill SMS most likely belongs to transactional or service, but **the page does not say so** [unverified]. A shop therefore needs a provider that owns the DLT work; the
POS supplies the template text and the variables [J].

### 4.5 Personal data: the DPDP Act and Rules, as far as I read them

The DPDP Rules were notified on **14 November 2025**; the PIB note says the Rules bring an **eighteen-month phased compliance period**; law-firm summaries put the main data-fiduciary duties at about **mid-May 2027** (one gives 12 May 2027)
and consent-manager duties at 12 months [W pib-dpdp; Ws dpdp-news]. What the PIB note says every data fiduciary must do [W pib-dpdp]:
- issue a **separate, clear consent notice** that says the specific purpose the data is collected and used for; consent can be withdrawn at any time;
- act on a request to **access, correct, update or erase** within **ninety days**; let the person nominate someone to exercise rights;
- **tell affected people of a breach** without delay, in plain language, with what happened, the likely impact, the steps taken and a contact; show a **contact point** for data queries;
- extra duties for **significant data fiduciaries** (independent audits, impact assessments) and verifiable parental consent for a child's data;
- penalties up to Rs 250 crore for failing reasonable security safeguards, up to Rs 200 crore for failing to notify a breach or breaching the children's duties, up to Rs 50 crore for other breaches.
The Act's seven principles named there are consent and transparency, purpose limitation, data minimisation, accuracy, storage limitation, security safeguards and accountability.

**What this means for a POS [J].** The shop is the **data fiduciary**; Comhard is a **data processor** (the PIB note defines the term) and needs a processing agreement with each shop and with its own
sub-processors (the WhatsApp provider, the SMS aggregator, the PSP, the hosting). Design now for the date that is coming:
1. **Phone and name are optional** for a B2C walk-in; Rule 46 asks for buyer details only at Rs 50,000 and above or on request [W cbic-rules]. Do not make them mandatory to bill.
2. **A notice at the point of capture**, on the customer-facing display or the receipt: what the number is for, by whom, and how to withdraw. "To send you this bill" and "to send you offers" are **two purposes and two
   consents**; marketing is off unless the customer says yes. Record who consented, when, for what, on which terminal [J].
3. **Minimise and expire.** The tax records the shop must keep (72 months for GST, section 3.8) do not need the customer's phone number; a **phone number kept only to deliver a bill can be dropped after delivery**
   unless the customer opted into loyalty or offers [J]. How the erasure right meets a retention duty was **not read** in the Act's text [unverified].
4. **A hosted bill is a page of personal data.** A random 128-bit token, an expiry, no listing endpoint, no search-engine indexing, no other customer's data on the page, and a delete on request [J].
5. **Payment-provider data.** A webhook usually returns the payer's VPA or phone; the till needs the **UTR or RRN** and a status, not the payer's identity. Store the reference; mask or drop the VPA [J]. The RBI's
   payment-data FAQ expects payment data handled by providers and the vendors they engage to sit in systems **in India** [W rbi-localisation]; where our VM runs is unchecked [unverified].
6. **A breach plan** with the three contacts a shop needs (the owner, Comhard, the Data Protection Board's portal described by PIB) and a template message, tested before the date [J].

---

## 5. [J] The integration plan

**This section is judgement.** It turns sections 1 to 4 into slices, partner conversations and decisions. It assumes the code audit's lanes (A the hot path, B the pure POS engine, C the contract and one Dexie bump, D the
counter UI, E devices and print, F server and rights, G reports) and the label spec's slices, and adds the payment, compliance and e-bill pieces they do not cover.

### 5.1 Principles

1. **Never hold money, never see a card.** A partner collects and settles; the till stores references and last four digits (1.4, 1.5).
2. **Pay first, issue second, print the reference.** A number is taken at tender finalisation; the invoice prints the payment cross-reference (1.3, 3.4).
3. **A tender always has a confirmation state and a source.** Anything unconfirmed is listed at shift close (1.3).
4. **A renderer never sends a printer byte.** Receipts, the drawer, the display and the scale go through a validated bridge in the style of `exportBridge.cjs` (2.6).
5. **With POS off, the app is exactly what it is today, asserted by identity** (the Phase 27 rule [C `CLAUDE.md`]); every new field is opt-in and carried through alteration.
6. **Statutory figures are dated data** (MDR, the Rs 200 line, thresholds, rates, FSSAI), cited, never in code.
7. **Every online dependency has a degraded path:** offline UPI is "unconfirmed, cashier-attested", offline card is "standalone", offline WhatsApp is "queue and open the link later".
8. **Windows desktop first**, with the browser dialog as the universal fallback (2.5).
9. **"Sent to the printer", never "printed"** and **"opened WhatsApp", never "sent"** (the Phase 29 rule [C `CLAUDE.md`]).

### 5.2 What to build in-house now (no partner, no hardware beyond a bench printer)

Size: S is days, M is one to two weeks, L is several weeks, for one engineer on pure code [J, rough]; nothing is estimated for hardware debugging.

| Slice | What | Indicative files (new unless marked) | Exit test | Size |
|---|---|---|---|---|
| **PI-1 Tender and confirmation** | tender types, the state machine (`pending`, `confirmed`, `expired`, `failed`, `unconfirmed`), `source`, split tender, change and over-tender caps, the **cash-cap warning**, the **MDR table as dated data**, the **PAN sentinel**; pure | `src/core/pos/tender.ts`, `payState.ts`, `mdrTable.ts`, `cashCap.ts`, `panGuard.ts` (the code audit's lane B) | 10,000 random baskets and tender mixes equal the Sales-screen build to the paisa (lane B's own exit); every illegal transition refused; a Luhn-passing 13 to 19 digit run refused or masked in every typed field and **absent from the outbox and backups**; MDR vectors 2,000 to 0, 3,000 to 12, 75,000 to 300 | M |
| **PI-2 Invoice compliance** | document kind (tax invoice, bill of supply, invoice-cum-bill, credit note), the series planner and the **16-character budget** validator (letters, digits, `/`, `-`; upper-case; case-insensitive uniqueness; no skipped number), the **payment cross-reference** block, the **MRP guard**, the **Rs 50,000 buyer-details rule**, the **e-invoice gate** for a GSTIN on a liable seller's bill, composition legend | `src/core/pos/docKind.ts`, `invoiceNo.ts`, `payRef.ts`, `mrpGuard.ts`, `buyerRule.ts`, `eInvoiceGate.ts` | table-driven suites from 3.1 to 3.7 with mutation lists; the budget table of 3.3 as vectors; GSTR-1 B2CS, HSN and Table 13 equal the same bills through the Sales screen (lane B) | M |
| **PI-3 Receipt model and renderers** | the receipt as data (header, lines, tax summary, payment lines with reference, footer, QR payload, language), a **48 or 32 column text layout**, a **raster renderer** (canvas to 1-bit rows at 384 or 576 dots), a duplicate mark and a reprint count | `src/core/pos/receipt.ts`, `receiptText.ts`; `src/ui/print/receiptRaster.ts`; `src/styles/receipt.css` (browser path R1) | golden receipts for five sample bills at 58 and 80 mm in text and raster; a 40-line text receipt under 4 KB; a Hindi line and the rupee sign legible on a real printer (bench) | M |
| **PI-4 Device bridge, Windows** | `devices.printers`, `addNetwork`, `receipt.print`, `drawer.open`; paths R3 (silent driver print), R4 (TCP 9100, spooler RAW by a **long-lived** hash-checked helper, Web Serial); the ESC/POS emitter and its mirrored validator; the drawer event log; status before and after | `electron/deviceBridge.cjs`, `escposLang.cjs`, `src/core/receiptJob.ts`, `scripts/assert/devicebridge.assert.ts` (coordinate with the label bridge T-7a: **one bridge, not two**) | the suite against a fake `electron` on one corpus (the validator twin); **a real printer on a packaged build**: one clean receipt, cutter and drawer fire, first paper under a second after Accept on USB, an unplugged printer fails visibly without losing the sale | L, hardware-gated |
| **PI-5 UPI link and unconfirmed flow** | extend `upi.ts` with optional `tr`, `mc`, `mid`, `msid`, `mtid`, `url` **written only when configured**; a QR on the customer display; the cashier-attested state; statement matching that recognises a UPI UTR and an MDR deduction line | `src/core/upi.ts` (modified: with nothing configured the bytes are unchanged), `src/core/bankStatement.ts` and `brsMatch.ts` (modified), a UI panel | golden link strings; existing invoices print byte-identically; a UTR in a statement clears an `unconfirmed` tender; a net settlement (gross minus MDR) matches | M |
| **PI-6 Customer display and scale** | a second `BrowserWindow` on the second display fed over IPC; a Web Serial scale reader with a per-model profile and a **simulator** | `electron/displayWindow.cjs`, `electron/scale.cjs`, `src/core/pos/scale.ts` | the simulator emits stable and unstable weights, tare, a unit change and a corrupt frame; the display shows items, total, QR and state | M |
| **PI-7 Hosted e-bill** (after TLS) | a bill page and a link: a random token, expiry, no listing; link in the `wa.me` text, e-mail and the UPI `url`; consent and notice fields | `backend/src/ebill.js` (conflicts with the CF lane's backend edits: sequence it), a bill HTML template over the pure invoice model | tokens not guessable (a test with a million guesses finds none), the page carries one customer's bill and nothing else, expiry honoured | M |
| **PI-8 Privacy basics** | consent record, notice text, phone minimisation setting, export and erase for one customer's contact data | `src/core/pos/consent.ts`, a settings panel | a bill saved with no phone; a withdrawn consent stops the link; an erase removes the phone from every row but keeps the bill | S |

**Not now, on purpose.** The statutory dynamic QR (nobody in the target band is bound, 3.4); a wallet or any top-up stored value (1.4); WebUSB, Star CloudPRNT, Epson ePOS and a QZ Tray dependency (each adds a
support burden for one printer family, 2.5); Chrome kiosk-printing as a feature (it is a deployment trick, document it only); a pole display; the PA-P business itself.

### 5.3 What needs a partner or an API

| Need | Candidates (public documentation I read) | What to ask | Prerequisite |
|---|---|---|---|
| **UPI dynamic QR with confirmation** | Razorpay (QR Codes API and virtual-account QR; on-demand activation; polling and webhooks) [W rzp-qr, rzp-va]; PhonePe (static-QR callback; dynamic QR and EDC unverified) [W phonepe-sq]; Paytm (EDC request API read; dynamic QR seen only via Zoho) [W paytm-edc, zoho-paytm]; Cashfree (static and dynamic QR, softPOS; **no soundbox found**) [W cashfree-upi] | webhook event names and signatures, settlement time, MDR and fee schedule after the 2026 framework, sandbox, whether a software vendor can be a **technology partner** with OAuth for merchants (Razorpay lists a technology-partner programme with OAuth for RazorpayX payouts, a precedent only [R live-bank-feeds §3.4]) | **TLS and a hostname**; a callback endpoint; the `PaymentProvider` interface and the manual provider first |
| **Card and UPI on a terminal** | Pine Labs cloud API [W pine-cloud]; Paytm EDC payment request [W paytm-edc]; Razorpay POS Bridge (seen second-hand) [Ws rzp-pos-bridge]; Mosambee and BharatPe (named by Petpooja, nothing read) [W petpooja-fin] | terminal ids per outlet, the merchant onboarding flow, split-tender behaviour, timeouts, a **test terminal**, who bears a chargeback | partner agreement per acquirer; the owner's customers' existing acquirers decide the order |
| **Soundbox integration** | Razorpay Dynamic Soundbox (names no API on its page) [W rzp-soundbox]; Petpooja-style integrated QR [W petpooja-fin] | the API behind "integrate with your billing system" | a PSP agreement |
| **WhatsApp attachments and automatic messages** | Meta's Cloud API directly or a provider; no provider chosen or compared | template approval time, document-header support, INR billing and GST, opt-in evidence Meta accepts | TLS; the owner's decision on who pays |
| **SMS** | a DLT-registered aggregator (none compared) | template registration for a bill, categories, price | a registered entity per shop or the aggregator's own |
| **E-invoice for B2B above Rs 5 crore** | a GSP, per `docs/research/einvoice-gsp.md` | IRN latency, offline handling, the 30-day rule | the e-invoice research's own prerequisites |
| **Settlement reconciliation by bank feed** | ICICI first per `docs/research/live-bank-feeds.md` | collections or virtual accounts | the bank partnership and TLS |
| **Hardware bundles** | a distributor of TVS, Epson, Posiflex-class printers, drawers, scanners, scales; scale makers' serial manuals (**none found**) | certified models, warranty, install | the bench list (2.7) |
| **A lawyer and a CA** | n/a | section 5.6 | before launch spend |

### 5.4 Order and dependencies

1. **TLS and a hostname** (HANDOFF's trust map): gates every PSP callback, the hosted bill, WhatsApp, bank feeds and the camera. Start it first and in parallel with the pure slices.
2. **The code audit's lane C contract** (`Voucher.pos`, tender list, one Dexie bump): PI-1 and PI-2 read from it; keep one editor of `db.ts`, `types.ts` and `store.ts`.
3. **PI-1, PI-2 and PI-3 start now** and run in parallel (pure files, no shared-file edits beyond the contract).
4. **PI-4 needs a rebuilt installer and a bench printer.** Share the printer list, the network allow-list and the Windows helper with the label bridge; agree the names (`devices.*` for the shared part, `labels.print` and `receipt.print` as job kinds) with whoever builds T-7a.
5. **Provider adapters** wait for TLS and for the owner's choice of partner; the manual provider and the interface ship first so the tender screen does not wait.
6. **PI-7 conflicts with the CF lane's backend edits**; schedule after its deploy [R pos-code-audit §14 lane F].

### 5.5 What the owner must decide (each with the default I would take so work can proceed)

| # | Decision | Default | Why it matters |
|---|---|---|---|
| 1 | The counter platform | **Windows desktop (Electron) first**; the browser dialog as fallback; Android and iPad are order-taking devices | only Windows reaches every device (2.5) |
| 2 | Multi-device restaurants: the **store hub** | **no tablet printing in v1**; the counter PC is the only printing machine; revisit after TLS | the mixed-content trap (2.5): a hub means HTTP on the LAN (no PWA features) or a trusted certificate on a real hostname |
| 3 | First payment partner or partners, and the commercial model | a provider-neutral interface; the manual provider, **one** PSP QR provider and **one** integrated terminal; **referral, never custody** | decides commission, TLS urgency, and legal exposure (1.4) |
| 4 | Show per-tender cost and net settlement | **yes**, from a dated MDR table, after checking whether the 15 October 2026 start moved | the 2026 framework changes shop economics; never a customer surcharge |
| 5 | Printer scope for v1 | **ESC/POS only**, certify two or three models after the bench | each printer family is a support burden |
| 6 | The sub-Rs 200 slice | **every bill is an invoice**; the consolidated daily invoice is a later opt-in | 3.1 |
| 7 | Series format | `<store 2 or 3 letters><terminal>/<sequence 6 or 7 digits>`, at most 14 characters, FY not in the text, upper-case | 3.3 |
| 8 | A GSTIN on a bill of an e-invoice-liable seller | **hard stop with a reason** until a GSP partner is chosen | an invoice without an IRN is not an invoice |
| 9 | E-bill v1 | print, `wa.me` text with a hosted link, e-mail; the WhatsApp Business API later; the owner chooses who pays message costs | cost, consent, TLS |
| 10 | Customer data | phone and name **optional**; two consents (bill, offers); phone dropped after delivery unless loyalty | DPDP, about May 2027 |
| 11 | Stored value | store gift vouchers and credit-note store credit **only**; no wallet, no top-up | 1.4 and 1.7 |
| 12 | Cash cap warning | warn at Rs 2 lakh aggregated per known customer per day | 1.6 |
| 13 | Scale support | a simulator and **one real model** chosen by the first customer who needs it | no maker protocol was found |
| 14 | The statutory dynamic QR for Rs 500 crore taxpayers | **not now** | 3.4 |
| 15 | The bench kit and who buys it | the list in 2.7, bought by Comhard | nothing in this study replaces it |

### 5.6 What a CA or a lawyer must sign before launch

1. How the **daily consolidated invoice** (Rs 200 line) appears in Table 13, and whether to offer it at all.
2. An **invoice-cum-bill of supply** as the default title for a mixed grocery basket.
3. **Rounding** of a bill total against Rule 18(2) (Legal Metrology), and whether it is judged per pack or per bill.
4. GST on **store gift vouchers**, credit-note store credit and **loyalty points** (discount or payment); the 2025 omission of sections 12(4) and 13(4).
5. The restaurant list in the restaurant study: service charge and tips, packaging charge, composition with an aggregator, mixed bills [R pos-restaurant-market §4.10].
6. Whether showing a PSP's QR or **referring** a PSP makes Comhard a payment aggregator or imposes disclosures (the RBI's 2025 Directions), and the **MDR no-pass-through** wording for any "convenience fee".
7. The **DPDP** notice and consent wording, the processor agreement, retention against erasure, breach duties.
8. Whether an **electronic bill** by WhatsApp satisfies Rule 48's copy marks and Rule 57's record duty for each vertical.
9. The **Income-tax Act 2025** section numbers for the cash cap and the electronic-modes duty (the 269ST and 269SU provisions are renumbered).
10. WhatsApp **opt-in** adequacy for "gave a number for a bill", and the **DLT** category for a bill SMS.

### 5.7 Questions to put to the owner's customers before sizing (cheap, and they change the plan)

1. Which receipt printer, drawer, scanner and scale do you own today, and over what link (USB, serial, LAN, Bluetooth)? (Decides printer models and whether serial speed matters, 2.2.)
2. Which card machine and UPI soundbox do you have, from whom, and do you use the machine's integration? (Decides partners, 5.3.)
3. Bills a day per till, the share on **credit**, the share **B2B**, and **annual turnover** against Rs 5 crore (e-invoice), Rs 50 crore (digital-acceptance duty) and Rs 1.5 crore (composition). (Decides 3.5, 1.6.)
4. Do you sell **packed goods** (MRP), **loose goods by weight**, **food** (FSSAI), **medicines** (licence)? (Decides which of 3.7 and 3.9 apply.)
5. Is the shop on a single machine or a network, and is there a stable internet connection? How often does the power or the line fail on a busy day? (Decides offline weight, the hub question, 2.5.)
6. Do you already send bills on WhatsApp, and from which number? Do customers ask for a bill? (Decides 4.1.)

---

## 6. What I could not verify (read this before relying on any claim)

**Hardware and accounts.** No printer, cash drawer, scale, scanner, card machine or soundbox was touched, and no account exists with Razorpay, PhonePe, Paytm, Cashfree, Pine Labs, Meta, an SMS aggregator or any
competitor product. Every device and vendor statement is documentation, a vendor's own marketing, or a search summary. The ESC/POS byte facts, the thermal widths and the transfer times (a calculation) are not
test results.

**Pages that refused or failed.**
- Epson's ESC/POS reference (every `download4.epson.biz` and `support.epson.net` URL I tried answered HTTP 403): command semantics come from a third-party list, an open-source library and search summaries.
- NPCI's own hosted copies of the UPI linking specification and of Circular 18 (the fetch returned only a site tagline): the specification was read from a third-party copy of the November 2017 **draft**; Circular 18
  is a search summary only. **NPCI's current QR and BharatQR specifications were not read.**
- The CBIC web page for Rule 46 (a certificate error): Rule 46 was read from the CBIC convenience PDF of **24 September 2021** and a gstzen reproduction of the current text; **amendments after 2021 were not read**.
- The RBI's final Payment Aggregator Directions (the URL redirected to the home page): read through two law-firm notes and the press release.
- The official copy of the NPCI UPI MDR FAQ (financialservices.gov.in): read from a broker's copy dated 15 September 2026.
- The Income Tax Department page on cash rules (HTTP 403): section 269ST and 269SU facts are from commentary; the Income-tax Act 2025 section numbers are from a Bill summary.
- `rawbt.ru`, `gst.kar.nic.in`, `fintaxblog.com`, `lkslaw.com` (DNS or 404), and the Epson `ePOS` reference: not read.

**Search budget.** The web-search budget was exhausted before I could ask **whether Cashfree has a soundbox** (its UPI page names none), whether Razorpay publishes **webhook event names** for a QR credit (its
pages named webhooks without them), or to cross-check the MDR start-date postponement with a second source. I did not work around the limit.

**Statutory points I could not close.**
1. **The MDR start date:** 15 October 2026 per the FAQ; "after Diwali" per one vendor page of 14 October; a Supreme Court petition was reported pending [W npci-mdr, terra-mdr]. One source.
2. **Whether a software vendor that shows a QR or routes to a partner is a payment aggregator** (PA-P): not addressed.
3. **How erasure meets retention** under the DPDP Act: not read; **the exact start of data-fiduciary duties** (about mid-May 2027) is from summaries.
4. **Whether Rule 18(2) is judged per pack or per bill** when a total is rounded up.
5. **The GST treatment of loyalty points**, and whether a WhatsApp message counts as the "original for recipient" under Rule 48.
6. **Section 31(3)(b)'s Rs 200 line** and **Rule 46's fourth proviso** were read together from the Act text on one site and the CBIC rule text; the **Rs 200 figure is not in the rule**, only in the Act.
7. **The Rule 46 amendments after 2021**, the **amended Notification 13/2020** (the exempt-entity list was assembled from a vendor page, a search summary and the e-invoice study), and the **Notification 14/2020 threshold after 2021**.
8. **Whether the Legal Metrology weighing-instrument periods** are 12 or 24 months for an electronic counter scale (one secondary source, tabulated by type).
9. **Meta's price for an in-window utility message** (two Meta pages conflict) and the **India rate card** (blog rates only).
10. **A utility template with a document header**, its size limit, and Meta's accepted opt-in wording.
11. **The DLT category for a bill SMS.**
12. **Any card-network or PCI DSS rule** on a merchant: not researched.
13. **Where the Comsky VM sits** against the RBI's payment-data localisation FAQ.

**What this study did not cover.** Marg ERP's and LogicERP's own payment and hardware integrations (beyond LogicERP's scale page) and a survey of which printer models Indian shops actually run: the pharmacy and restaurant studies carry the product comparison, and the bench list in 2.7 and the questions in 5.7 are how to close the model question.

**What I did not do in the repo.** No code ran, no test ran, no file other than this one was written. Line anchors are from the working tree at `5aee765` (the tree also holds other workstreams' uncommitted files).

---

## 7. Sources

**Statute, rules, notifications and circulars (read from the PDF's text layer unless noted) [W]**
- `cbic-rules` CGST Rules 2017, CBIC convenience copy as on 24 Sep 2021 (Rules 5, 46, 46A, 47, 48, 49, 53, 56, 57): https://cbic-gst.gov.in/pdf/24092021-CGST-Rules-2017-Part-A-Rules.pdf
- `gstzen-r46` Rule 46 as reproduced (current structure): https://gstzen.in/a/tax-invoice-cgst-rule-46.html
- `n14` Notification 14/2020-CT, 21 Mar 2020: https://gstcouncil.gov.in/sites/default/files/2024-05/notfctn-14-central-tax-english-2020.pdf
- `n71` Notification 71/2020-CT, 30 Sep 2020: https://gstcouncil.gov.in/sites/default/files/2024-05/notfctn-71-central-tax-english-2020.pdf
- `n89` Notification 89/2020-CT, 29 Nov 2020: https://gstcouncil.gov.in/sites/default/files/2024-05/notfctn-89-central-tax-english-2020.pdf
- `n06-21` Notification 06/2021-CT, 30 Mar 2021: https://gstcouncil.gov.in/sites/default/files/2024-05/notfctn-06-central-tax-english-2021.pdf
- `n28-21` Notification 28/2021-CT, 30 Jun 2021: https://gstcouncil.gov.in/sites/default/files/2024-05/notfctn-28-central-tax-english-2021.pdf
- `c146` Circular 146/02/2021-GST, 23 Feb 2021: https://gstcouncil.gov.in/sites/default/files/2024-06/circular_refund_146.pdf
- `n78` Notification 78/2020-CT, 15 Oct 2020 (HSN digits): https://www.gstcouncil.gov.in/sites/default/files/2024-05/notfctn-78-central-tax-english-2020.pdf
- `circ167` Circular 167/23/2021-GST, 17 Dec 2021: https://cbic-gst.gov.in/pdf/Circular-167-17-12-2021-GST.pdf
- `gstservice-rates` GST Council consolidated service rates (notification 11/2017 table): https://www.gstcouncil.gov.in/sites/default/files/2024-02/gstservicerates.pdf
- `ctr05-2025` Notification 05/2025-CT (Rate), 16 Jan 2025: https://gstcouncil.gov.in/sites/default/files/2025-01/ctr05-2025.pdf
- `lm-pc` Legal Metrology (Packaged Commodities) Rules 2011 with amendments, Department of Consumer Affairs: https://consumeraffairs.gov.in/public/upload/admin/cmsfiles/whatsnews/Book_on_Legal_Metrology_Packaged_Commodities_Rules,2011_with_all_amendments_whatsnews.pdf
- `cgst-31` section 31: https://www.knowyourgst.com/gstlaw/the-central-goods-and-services-tax-act-2017/31-tax-invoice-31/ · `cgst-15` section 15: https://www.knowyourgst.com/gstlaw/the-central-goods-and-services-tax-act-2017/15-value-of-taxable-supply-15/
- `rbi-cof21` RBI/2021-22/96, 7 Sep 2021: https://www.rbi.org.in/scripts/BS_CircularIndexDisplay.aspx?Id=12159 · `rbi-cof22` RBI/2022-23/95, 28 Jul 2022: https://www.rbi.org.in/scripts/BS_CircularIndexDisplay.aspx?Id=12363
- `rbi-localisation` RBI FAQ, storage of payment system data: https://rbi.org.in/scripts/FAQView.aspx?Id=130 · `rbi-pa-pr` press release, 15 Sep 2025: https://www.rbi.org.in/scripts/BS_PressReleaseDisplay.aspx?prid=61218 · `rbi-ppi` Master Directions on PPIs (updated to 30 Sep 2026): https://www.rbi.org.in/scripts/bs_viewmasdirections.aspx?id=12156
- `pib-dpdp` PIB, DPDP Rules 2025 notified, 17 Nov 2025: https://static.pib.gov.in/WriteReadData/specificdocs/documents/2025/nov/doc20251117695301.pdf
- `npci-link` NPCI UPI Linking Specifications 1.6 (Draft), Nov 2017, third-party copy: https://www.labnol.org/files/linking.pdf
- `npci-mdr` NPCI FAQ, MDR on select UPI P2M transactions, 15 Sep 2026, broker's copy: https://flattrade.in/kosh/wp-content/uploads/2026/09/FAQs-Merchant-Discount-Rate-MDR-on-Select-UPI-P2M-Transactions_0-1.pdf

**Commentary and vendor pages fetched and read [W]**
- `scc-mdr` https://www.scconline.com/blog/post/2026/09/16/npci-released-upi-mdr-faqs-explained/ · `terra-mdr` https://www.terra-insight.com/insights/upi-mdr-p2m-merchant-charges-india/ (a reconciliation-software vendor)
- `azb-pa` https://www.azbpartners.com/bank/rbi-issues-consolidated-reserve-bank-of-india-regulation-of-payment-aggregators-directions-2025/ · `lex-pa` https://lexplosion.in/rbi-master-direction-2025-compliance-mandate-for-payment-aggregators-and-pa-p-deadline/
- `taxheal-n10` https://www.taxheal.com/notification-no-10-2023-central-tax-gst-einvoice-limit-reduced-to-rs-5-crore-from-01-08-2023.html · `tally-einv` https://tallysolutions.com/gst/e-invoicing-limit-india/ (Updated 9 Dec 2025) · rejected: https://legalsuvidha.com/latest-updates/gst-e-invoice-threshold-updated-for-fy-2026-27-210926 (flags its own content as AI generated)
- `taxheal-single` https://www.taxheal.com/rules-to-issue-single-tax-invoice-in-gst-for-all-b2c-supplies-in-a-day.html · `cleartax-ch6` https://cleartax.in/s/cgst-rules-chapter-6-tax-invoice-credit-and-debit-notes
- `taxclue-voucher` https://www.taxclue.in/blog/gst-on-vouchers-after-finance-act-2025/ · `it-2025` https://www.taxheal.com/new-cash-transaction-limits-under-the-income-tax-act-2025.html · `cleartax-269su` https://cleartax.in/s/section-269su
- `foodsafetyworks` https://foodsafetyworks.com/news/fssai-order-mandatory-declaration-of-fssai-license-registration-number-on-cash-receipts-invoices-cash-memo-bills-etc-by-all-food-businesses/ · `ssrana-lm` https://ssrana.in/articles/verification-of-weights-measures-legal-metrology-act/
- `rzp-qr` https://razorpay.com/docs/api/qr-codes/create.md · `rzp-va` https://razorpay.com/docs/payments/payment-methods/upi-qr/api.md · `rzp-soundbox` https://razorpay.com/docs/pos/dynamic-soundbox.md
- `pine-cloud` https://developer.pinelabs.com/in/instore/cloud-integration (the first 100,000 of 142,214 characters read) · `paytm-edc` https://paytmpayments.com/docs/pos-connection-via-payment-request · `phonepe-sq` https://developer.phonepe.com/offline-integration/integrated-static-qr/introduction · `cashfree-upi` https://www.cashfree.com/upi-payment-gateway/ · `zoho-paytm` https://www.zoho.com/en-in/pos/resources/help/paytm-integration.html
- `petpooja-fin` https://blog.petpooja.com/industry-business-guides/petpooja-finance-money-layer-in-pos/ (3 Jul 2026) · `gofrugal-pay` https://cdn.gofrugal.com/blog/billing-software-with-upi-card-cash/ (31 Mar 2026; vendor content) · `logic-scale` https://www.logicerp.com/retail-pos-software/features/pos-integrated-weighing-scale (vendor marketing)
- `play-sms` https://support.google.com/googleplay/android-developer/answer/9047303 · `chrome-webusb` https://developer.chrome.com/docs/capabilities/build-for-webusb
- `electron-print` https://www.electronjs.org/docs/latest/api/web-contents (the first 100,000 of 133,946 characters) · `electron-devices` https://www.electronjs.org/docs/latest/tutorial/devices · `electron-screen` https://www.electronjs.org/docs/latest/api/screen
- `mdn-mixed` https://developer.mozilla.org/en-US/docs/Web/Security/Mixed_content · `mdn-wm` https://developer.mozilla.org/en-US/docs/Web/API/Window_Management_API
- `python-escpos` https://python-escpos.readthedocs.io/en/latest/user/methods.html · `tabshop-esc` https://tabshop.smartlab.at/help-topics/help-esc-pos-codes.html
- `qz-licence` https://qz.io/docs/licensing (revision of 12 Nov 2020) · `star-about` https://star-m.jp/products/s_print/sdk/StarCloudPRNT/manual/en/about.html
- `meta-wa` https://developers.facebook.com/docs/whatsapp/pricing/ · `wa-pricing-hi` https://whatsappbusiness.com/hi/products/platform-pricing/ · `exotel-dlt` https://developer.exotel.com/docs/sms-support/what-is-dlt

**Search summaries only [Ws]**
- `npci-c18` https://www.npci.org.in/PDF/npci/upi/circular/2017/Circular18_BankCompliances_to_enbaleUPIMerchantecosystem_0.pdf · `p2p-collect` https://www.theweek.in/wire-updates/business/2025/08/14/dcm65-biz-npci-collect.html, https://www.etvbharat.com/en/!business/npci-to-stop-upi-p2p-collect-requests-from-oct-1-to-combat-fraud-enn25081404610 · `p2m-limits` https://www.newsonair.gov.in/npci-raises-upi-person-to-merchant-transaction-limit-to-%E2%82%B95-lakh, https://www.outlookmoney.com/banking/npci-changes-upi-transaction-limits-for-key-categories-from-september-15-2025-know-the-details
- `rbi-pa` / `lex-pa` summaries: https://lexplosion.in/rbi-issues-new-master-directions-to-consolidate-all-regulations-related-to-payment-aggregators-effective-immediately/
- `cgst-36` https://www.knowyourgst.com/gstlaw/cgst-act/36-period-of-retention-of-accounts-36/, https://gstgyaan.com/section-36-of-the-cgst-act-period-of-retention-of-accounts · `cgst-170` https://www.knowyourgst.com/gstlaw/cgst-act/170-rounding-off-of-tax-etc-170/ · `s34-time` https://bdo.in/en-gb/insights/alerts-updates/indirect-tax-alert-cbic-clarifications-on-the-time-limit-for-compliances-pursuant-to-union-budget · `iris-numbering` https://irisgst.com/?p=54609 · `n13-amend` https://icmai.in/upload/Taxation/Top_Stories/ITN/ITN_1205_2023_2.pdf
- `it-269st` https://www.taxscan.in/?p=280078 · `icai-audit-trail` https://cajournal.icai.org/article-details/audit-trail-requirements-responsibilities, https://www.taxmann.com/post/blog/faqs-statutory-auditors-duty-to-report-audit-trails · `dpdp-news` https://www.azbpartners.com/?p=87799, https://www.ey.com/en_in/insights/cybersecurity/transforming-data-privacy-digital-personal-data-protection-rules-2025
- `wa-rates` https://myoperator.com/blog/whatsapp-business-api-pricing-india-2026, https://montymobile.com/blogs/whatsapp-business-api-pricing-in-india-inr-rates-gst-and-the-2026-currency-migration-deadline
- `scale-search` a web search for makers' serial protocols (Essae, Avery, CAS) that returned vendor marketing and an Odoo add-on page only; no URL is relied on
- `epson-pulse`, `epson-cr` https://download4.epson.biz/sec_pubs/pos/reference_en/escpos/esc_lp.html, https://download4.epson.biz/sec_pubs/pos/reference_en/escpos/esc_cr.html (HTTP 403 to the fetch tool) · `thermal-unicode` https://pub.dev/packages/thermal_unicode_print · `rupee-thread` https://static.codeproject.com/script/comments/View.aspx?cmt=911012
- `tvs-store`, `tvs-store2` https://store.tvselectronics.in/products/rp3160-gold-thermal-receipt-printer, https://tvs-e.in/rp-3160-gold · `spec-58` https://www.zkteco.com/en/Printer/ZKP5804, https://www.posiflex.com/ja-jp/product/index/POS-Printer/POS_Printer/AURA_PP_7900 · `sunmi-v2` https://file.cdn.sunmi.com/newebsite/downloads/specs/en/v2-pro-new.pdf · `pole-display` https://forum.sambapos.com/t/vfd220-pole-display/8602
- `kiosk-printing` https://pikr.helpscoutdocs.com/article/58-disable-print-previews-of-shipping-labels-in-chrome, https://support.phppointofsale.com/hc/en-us/community/posts/4409909930132-print-directly-without-preview-dialog · `qz-search` https://buy.qz.io/Premium-Support-_p_13.html, https://github.com/qzind/tray/wiki/Signing
- `star-protocol` https://star-m.jp/products/s_print/sdk/StarCloudPRNT/manual/en/protocol-guide.html · `epson-sdp` https://download4.epson.biz/sec_pubs/pos/reference_en/technology/server_direct_print.html · `epson-sdk` https://download4.epson.biz/sec_pubs/pos/reference_en/epos_js/ref_epos_sdk_js_en_eposdeviceobject_connectmethod.html, https://dev.to/rubenruvalcabac/printing-to-tm-t88vi-from-an-azure-static-app-coh · `rawbt` https://www.apkmirror.com/apk/402d/rawbt-print-service, https://support.dealpos.com/en/articles/11980815-how-to-print-customer-invoices-on-mobile-from-dealpos-web-using-rawbt
- `rzp-pos-bridge` https://dev.to/ashiqu_ali/razorpay-pos-bridge-integration-in-flutter-trigger-payments-without-app-on-pos-device-1lo1 · `paytm-edc-search` https://paytmpayments.com/docs/pos-edc-wired-connection-wired-ecr-dll, https://www.paytmpayments.com/docs/pos-connection-via-app-invoke

**Repo [C] and sibling documents [R]**
- `src/core/upi.ts`, `src/core/share.ts`, `src/core/qr.ts`, `src/core/gstr1.ts:313-315`, `src/types.ts:186-190`, `:253-256`, `:888-890`, `src/ui/reportTools.ts:275`, `:303`, `src/ui/print/PrintVoucher.tsx:608`, `:676`, `electron/exportBridge.cjs`, `electron/preload.cjs`, `electron/main.cjs:62`, `:143-155`, `:188-190`, `:212-214`, `electron/labelLang.cjs`, `package.json:32`, `CLAUDE.md`.
- `docs/research/pos-code-audit.md` (sections 0, 1.4 to 1.6, 2, 4, 5, 8, 9, 14, 15), `docs/research/labels-and-scanning.md` (2.4 to 2.6, 3.1, 3.6), `docs/design/batch-serial-labels-spec.md` (13.1, 13.6, 14), `docs/research/pos-restaurant-market.md` (3.3, 4), `docs/research/pos-pharma-market.md` (3.9), `docs/research/einvoice-gsp.md` (3), `docs/research/live-bank-feeds.md` (TL;DR, 3.3, 3.4), `docs/research/mobile-tech-options.md` (3.10), `docs/HANDOFF.md` (trust map, through the code audit).

---

## Appendix A. The numbers I computed [M]

Method: a scratch Node script (session scratchpad, not committed); no repo file read as input beyond the facts cited. 203 dpi is taken as 8 dots a millimetre; serial links are 8N1 (ten bits a byte); the USB figure is the
12 Mbit/s raw rate with no protocol overhead (a lower bound on time).

```
receipt raster at 8 dots/mm       rows   bytes     KiB    9600 baud  19200   115200  USB 12 Mbit/s
58 mm (384 dots, 48 B/row) 100 mm  800   38,400    37.5     40.0 s    20.0 s   3.3 s   25.6 ms
58 mm                      150 mm 1200   57,600    56.3     60.0 s    30.0 s   5.0 s   38.4 ms
58 mm                      200 mm 1600   76,800    75.0     80.0 s    40.0 s   6.7 s   51.2 ms
58 mm                      300 mm 2400  115,200   112.5    120.0 s    60.0 s  10.0 s   76.8 ms
80 mm (576 dots, 72 B/row) 100 mm  800   57,600    56.3     60.0 s    30.0 s   5.0 s   38.4 ms
80 mm                      150 mm 1200   86,400    84.4     90.0 s    45.0 s   7.5 s   57.6 ms
80 mm                      200 mm 1600  115,200   112.5    120.0 s    60.0 s  10.0 s   76.8 ms
80 mm                      300 mm 2400  172,800   168.8    180.0 s    90.0 s  15.0 s  115.2 ms
mechanism at 160 mm/s: 100 mm 0.63 s, 150 mm 0.94 s, 200 mm 1.25 s, 300 mm 1.88 s
text receipt, 40 lines x 48 columns: 1,960 bytes
invoice number lengths: T1/26-27/000123 = 15; MUM01/T3/000123 = 15; MUM01/T3/26-27/0001 = 19; S12T3/2627/0001234 = 18; S12-T3-0001234 = 14; S12/3/0001234 = 13
bills a year at 3,000 a day: 1,095,000 (seven digits)
WhatsApp utility at Rs 0.115 (blog rate), 18% GST: 100/day 3,000 msgs Rs 345 (Rs 407); 300/day 9,000 Rs 1,035 (Rs 1,221); 800/day 24,000 Rs 2,760 (Rs 3,257)
UPI MDR (0.4% above Rs 2,000, Rs 300 from Rs 75,000): 1,999 -> 0; 2,000 -> 0; 2,001 -> 8.00; 3,000 -> 12; 12,500 -> 50; 50,000 -> 200; 75,000 -> 300; 1,00,000 -> 300
```

## Appendix B. Checks behind the "does not exist" and anchor statements (run 2026-10-25 from the repo root)

```
ls electron                       -> exportBridge.cjs  labelLang.cjs  main.cjs  preload.cjs   (no labelBridge.cjs, no device or printer bridge)
ls src/core | grep -i label|scan  -> labelJob labelLayout labelModel labelRender labelRules labelTemplates scanDevice scanInput scanResolve unitId (+ barcode/)
grep -n "window.print()" src/ui/reportTools.ts src/ui/print/PrintVoucher.tsx -> reportTools.ts:275,303  PrintVoucher.tsx:608,676
grep -n "PaperSize" src/types.ts  -> 186: 'A4' | 'A5' | 'Letter'
grep -n "contextIsolation|nodeIntegration" electron/main.cjs -> 188-189
grep -n "loadFile|loadURL" electron/main.cjs -> 212 (localhost:5173), 214 (dist/index.html)
grep -n "\"electron\"" package.json -> 32: "^31.4.0"
grep -n "waLink" src/core/share.ts -> 127
sed -n 1,26p src/core/upi.ts      -> the NPCI draft, the fields left out on purpose, the default-off rule
```
