# Customer & Vendor Portal — research and design

> **Ask (consultants, 2026-10-15):** "Similar to Zoho Books, we should be able to give customers
> and vendors a login portal from where they can log in and see their ledger details."
>
> **Status:** research + design only. Nothing here is built. Written 2026-10-15 against `main`
> (backend migrations `001`–`006`, client Dexie v7). External facts carry a source URL. Anything
> I could not verify is marked as unverified.

---

## 1. TL;DR — recommendation

1. **Build it as a cloud-only, read-only party portal.** Serve it from its **own origin**, with
   **passwordless** login: an invite the owner sends, then a passkey or email OTP. Give portal
   users a **separate identity namespace and token audience** from tenant members.
2. **Compute what a party sees on the server.** Run our own `src/core` engines (bundled for
   Node with esbuild, as `scripts/assert/run.mjs` already does) to build an **allow-listed,
   per-party projection**. Serve it through a **least-privilege Postgres role** whose RLS is keyed
   on `(tenant, company, ledger)` and which has **no grant at all on `change_log`**. Do not ship
   raw voucher documents to a portal client, however sandboxed it is.
3. **The MVP (statement, outstanding bills, invoice copies via print-to-PDF) needs no partner
   contract and no paid credential.** **Going live does need TLS**, which the deployment does
   not have today (HTTP only). Setting up TLS needs owner action: a DNS API token, or routing
   443 to the VM.
4. **Payment links need each tenant's own KYC'd gateway account**, connected either with the
   tenant's own keys or through Razorpay partner OAuth. Comhard must never sit in the flow of
   funds: that is payment aggregation, which needs RBI authorisation. Today's Razorpay adapter
   is MOCK and belongs to Comhard's *own* subscription billing, so it cannot be reused.
5. **Effort:** about **10–13 engineer-weeks** to a hardened MVP, plus 1–1.5 weeks of
   prerequisites (TLS, origin, token audience). Comments and balance confirmation take about
   5–7 weeks more. Payments take 3.5–5 weeks after a gateway decision.

---

## 2. What the ask really needs

"See their ledger details" is the core. The consultants are reacting to the Zoho experience,
where a customer opens a link and sees **what they owe, why, and the documents behind it**,
without phoning the accounts team. Under the ask there are three jobs:

| Job | What the party wants | What the owner gets |
|---|---|---|
| **Reconcile** | Statement of account for a period, matching their own books. | Fewer "please send ledger" calls and WhatsApps. |
| **Collect / pay** | Open bills, due dates, ageing, invoice copies, (later) a Pay button. | Faster collection, fewer disputes. |
| **Prove** | A copy of the invoice exactly as issued, and a record that they saw it. | Evidence: "customer viewed INV/26-27/041 on 12 Oct". |

### User stories

**Owner / accountant, in the Comsky app (keyboard-first).** Shortcuts, menu placement and field
order are frozen (CLAUDE.md). So everything below lives on a **new screen**, reached through the
new-screen checklist (`Screen` union → `router.tsx` → crumb → `MenuScreen.tsx` +
`modern/CommandPalette.tsx` + `overlays.tsx` Go To). There are no new fields in the Ledger
master and no new global chords.

- **O1** — On a *Portal Access* list screen, I pick a party with the arrow keys and Enter. I add
  a contact (prefilled from `Ledger.email` / `contact` / `phone`) and press one screen-local key
  to **invite**. The invite link is copied, or opened as a WhatsApp click-to-chat
  (`https://wa.me/<phone>?text=…`) on my own device, so no messaging API is needed. Proposed
  screen-local keys: `I` invite, `R` revoke, `V` view as party, `A` activity. They are registered
  with `useScreenKeys` and must be checked against the `?` sheet before they are fixed.
- **O2** — I select many Sundry Debtors that have an email and enable them in **one action**. The
  selection-scoped pattern already exists (Phase 23 "Mark Optional/Regular"; `src/ui/selection.ts`).
  The action is one audited server call.
- **O3** — **View as party** before enabling: the same screen the party will see, built by the
  same projection function, computed locally from Dexie. It is labelled "local preview; the party
  sees the cloud copy as of hh:mm".
- **O4** — I **revoke** a contact. Their sessions die on their next request (revocation is
  checked on every request, as `tokens.js` `verifyAccess` already does for tenant families).
- **O5** — A *Portal Activity* report (ReportShell, printable, drillable) answers: who logged in,
  what they viewed or downloaded, and which **version** of each document they saw.
- **O6** — Company-level defaults: portal on or off, which sections are visible, banner text,
  which bank accounts and terms appear on invoice copies, and dispute contact.
- **O7** (phase 2) — From mobile, I approve or answer a party's **query** ("rate mismatch on line
  3"), **balance confirmation** or **"I've paid, UTR …" claim** in a portal inbox. The consultants'
  approvals ask would reuse this inbox.

**Customer (party) — mobile-first, keyboard-friendly:**

- **P1** — I open the invite on my phone, confirm with a passkey or a code sent to my email, and
  land on **Summary**: balance due, overdue, last payment, credit limit (only if the owner
  chooses to show it).
- **P2** — **Statement** for a date range (default: current financial year). I can print it or
  save it as PDF. It is stamped "As per <Company>'s books synced on <date time>".
- **P3** — **Open bills** with due date and ageing. Tapping one opens the **invoice copy**, which
  I can print or save as PDF.
- **P4** — If two businesses (or two companies of one business) gave my email access, I
  **switch** between them. One sign-in, separate scopes.
- **P5** (phase 2) — Raise a query on a line. **Confirm balance as on a date.** This is the
  online form of the "Confirmation of Accounts" letter that `src/ui/reports/reminders.tsx`
  already prints.
- **P6** (phase 3) — Pay a bill online, in part or in full.

**Vendor (party):**

- **V1** — See the invoices of mine that they have **booked** (my invoice no./date from
  `Voucher.supplierInvNo/supplierInvDate`, their reference), payments made to me, and the
  balance they owe me.
- **V2** (phase 2) — See **TDS deducted** per payment, which helps my Form 26AS reconciliation,
  and a **payment advice** for each payment, restricted to my own bills.
- **V3** (phase 4) — Upload my invoice (PDF/image) for them to book. This depends on the
  attachments epic and object storage.

**Comhard platform admin:**

- **A1** — A per-tenant kill switch and per-tenant OTP send budget (anti-abuse).
- **A2** — Platform impersonation (`admin.js` `/admin-api/tenants/:id/impersonate`) must **not**
  extend to portal identities.

**Non-goals for v1:** parties editing the books; parties seeing anything the printed document or
the statement does not show; offline portal; native mobile app; a "network" between Comsky
tenants (a separate ask, though this identity layer is its stepping stone).

---

## 3. Where we start — facts from the codebase that shape the design

| Fact | Where | Consequence for the portal |
|---|---|---|
| The server stores **only an append-only `change_log`** of whole jsonb docs. It has **no materialised doc tables** and **no report engine**. REST reads current state with `DISTINCT ON (cid, doc_id) … ORDER BY seq DESC` scans. | `backend/sql/001_p1.sql`, `002_p2.sql`; `backend/src/rest.js` | A server portal needs its own projector. ARCHITECTURE §5 "Where the engines run" already defers a server engine "until a concrete need"; this ask is that need. |
| RLS is **tenant-only** (`app.tenant_id`). It is `ENABLE`, not `FORCE`. There is **no company scope** (`app.cids` is designed, not built) and **no role check** on any `/api` route: any member can pull the whole tenant and manage keys. | `002_p2.sql`; `backend/src/db.js` `withTenant`; `platform.js` | Party scope needs **new** GUCs and policies. Portal admin endpoints need a membership role check, which does not exist yet. |
| **Tenant access tokens, admin tokens and (if we were careless) portal tokens would share one HS256 secret with no `aud` claim.** `verifyAccess` accepts any token signed with `JWT_SECRET` whose `fam` is absent, or present and unrevoked. | `backend/src/tokens.js` (`signAccess`, `verifyAccess`); `admin.js` `adminToken` | **Critical design rule:** if portal tokens were signed with `JWT_SECRET` and carried a `tid` without `fam`, they would pass `app.authenticate`, and a customer could call `/sync/pull` and **download the entire tenant**. Portal tokens need a separate secret **and** audience, and `verifyAccess` should start requiring `aud`. |
| `trustProxy: true`, and `clientIp()` takes the **first `X-Forwarded-For` value**. The VM is reached directly on `:1036`, with no proxy in front. | `backend/src/server.js:27`; `backend/src/ratelimit.js:41-45` | Per-IP rate limits can be **bypassed by sending a fake `X-Forwarded-For`**. The portal is internet-facing to non-customers, so it must read the socket address unless a known proxy sits in front. |
| CORS reflects **any** origin by default. | `backend/src/config.js` `corsOrigin`; `server.js:31` | Portal routes that use cookies need an explicit origin allow-list with `credentials: true` for the portal origin only. |
| The rate limiter and SSE hub are **in-memory**. | `ratelimit.js`, `realtime.js` | Limits reset on restart. That is acceptable on one VM for the MVP; it is a known limit (HANDOFF §2 trust map). |
| **HTTP only, no TLS.** | `docs/HANDOFF.md` §2 | **Hard go-live blocker.** Magic links, OTPs, session cookies (`Secure`) and passkeys (WebAuthn needs a secure context) all assume HTTPS. |
| The backend **sends no email** (no SMTP or provider dependency). | `backend/package.json`, `backend/src/*` | Email OTP and invites need an email provider credential. The MVP can avoid it with invites the owner delivers plus passkeys. |
| The owner app keeps its sync token in **`localStorage`** on the app origin. | `src/sync/sync.ts` (`TOKEN_KEY`) | If the portal were served **same-origin**, any XSS on a portal page (party-supplied comment text, an item description rendered unsafely) could read an owner's token in the same browser. **A separate origin is required.** |
| Invoice printing already has a **pure data model**: "a number on paper is a number on screen". `buildInvoiceModel` takes company, voucher, the ledgers/items/units it references, banks and terms. It has no React, DOM or Dexie. | `src/ui/print/invoiceModel.ts`; asserted by `scripts/assert/print.assert.ts` | This is the ideal **allow-list for invoice copies**: the portal shows the printed document and nothing more. |
| The print model emits **`IRN` / `Ack. no.`** when `v.irn` is set, and IRN is **MOCK**. | `invoiceModel.ts` (meta push of `irn`, `ackNo`); `src/ui/gst/einvoice.tsx` header ("IRN / QR here are MOCK") | A portal copy must **strip** mock IRN/Ack. An external party must never receive a fake IRN on an invoice that looks genuine. |
| The bank accounts and terms printed on invoices are **device-local** (`localStorage`), not synced. | `src/ui/print/bankStore.ts`, `termsStore.ts` | A server-rendered copy cannot know them. Portal settings must hold them (or they become a synced company setting). |
| `PrintVoucher.tsx` imports the Zustand store, keys and feedback. `PagedDoc.tsx` + `paginate.ts` + `print.css` are clean. | `src/ui/print/*` | The portal needs the **presentational invoice sheet extracted** into a store-free component that takes an `InvoiceModel`. |
| `outstandingBills(ledgers, vouchers, asOn)` and the Ledger Vouchers logic (party legs only, `postsToLedgers` guard, opening + period movement) are pure. | `src/core/billwise.ts`; `src/ui/reports.tsx` `LedgerVouchers` | Statement and bills can reuse the exact on-screen math. Passing **only the party's ledger** to `outstandingBills` is both correct and minimising, because it skips non-billwise and unknown ledgers. |
| `scripts/assert/run.mjs` already **esbuild-bundles `src/core` for Node**. | `scripts/assert/run.mjs` | Proven path to run the same engines server-side, with no rewrite. |
| Party contact fields live on `Ledger` (`email`, `phone`, `contact`). **Any member** can edit them, and it syncs. | `src/types.ts` `Ledger` | Portal access must **not** follow `Ledger.email` silently. Otherwise anyone who can edit a ledger could redirect a party's access. Portal contacts are a separate, audited, admin-only record. |
| The mock billing provider's `verifyWebhook()` returns `true`. | `backend/src/billing/mock.js:10` | Fine for demoing Comhard's own billing. **Unacceptable** for an endpoint that marks a customer's bill paid. Portal payments get no mock path in a production build. |
| Plan limits are display-only. Only `canSync` is enforced. | CLAUDE.md "Cloud account tooling"; `subscription.js` | If the portal is a paid-plan feature, the backend must enforce it, which is new. |

---

## 4. Landscape (checked 2026-10-15)

### Zoho Books — the reference the consultants named

**Customer portal** ([help, IN](https://www.zoho.com/in/books/help/client-portal/);
[product page](https://www.zoho.com/in/books/customer-portal/)):

- **Access:** per **contact person**, through "Invite to Portal" / "Configure Portal Access". The
  contact accepts an emailed invitation and **sets a password**. The help page describes no OTP
  or magic link. Invitations can be re-sent. When a customer forwards a document, the recipient
  becomes a contact person.
- **Home:** outstanding invoices, available credits, retainer balances, last payment, pending
  quote and timesheet approvals, account and contact details.
- **Modules:** Quotes (accept, or decline and negotiate in comments), Invoices (regular and
  retainer), Payments (with receipts), Sales Orders and Projects/Timesheets (each optional),
  **Statement of accounts for a chosen date range**. Documents can be printed, downloaded as
  PDF and forwarded.
- **Actions:** Pay Now (cards, saved cards; ACH where enabled; partial payments where allowed on
  the invoice; bulk payment), approve timesheets, comment, **upload documents**, update profile,
  company and card details, email the business, write reviews.
- **Owner side:** notified on views, quote decisions, comments, payments and detail updates;
  portal URL configurable; banner and portal colours; a custom menu item that embeds web pages;
  SalesIQ chat.

**Vendor portal** ([help, IN](https://www.zoho.com/in/books/help/vendor-portal/)):

- Invite per vendor or contact, and the vendor sets a password.
- Home: outstanding bills, credits, last payment received.
- Purchase Orders, which the vendor can accept or decline with a reason (opt-in).
- "Invoices" = documents the vendor uploaded that the business accepted as bills.
- Payments Received, and a statement for a date range.
- **Uploads: images/PDF only, max 5 MB each.** They arrive as *Pending* until the business
  converts, rejects or deletes them. Optional autoscan consumes the plan's scan quota.
  "Upload against PO" and some onboarding features need the **BillPay add-on**.

**Plan gating** ([pricing, IN](https://www.zoho.com/in/books/pricing/), as rendered by a fetch on
2026-10-15; prices are ₹/org/month, annual vs monthly billing, before tax):

| Plan | Annual | Monthly | Customer portal | Vendor portal |
|---|---|---|---|---|
| Free | — | — | yes | no |
| Standard | ₹749 | ₹899 | yes | no |
| Professional | ₹1,499 | ₹1,799 | yes | no |
| Premium | ₹2,999 | ₹3,599 | yes | **yes** |
| Elite / Ultimate | ₹4,999 / ₹7,999 | ₹5,999 / ₹9,599 | yes | yes |

**Reading:** the customer portal is table stakes at Zoho, offered even on its free plan. The
vendor portal is a premium differentiator.

### Where the Indian desktop incumbents are

- **TallyPrime** offers, among its *connected services*, WhatsApp sharing of invoices, reports and
  ledgers (bulk too), **Payment Request** links and UPI QR via Razorpay/PayU, read-only
  **browser access** to reports *for the business itself*, connected banking, and *Bharat
  Connect for Business* invoice exchange
  ([connected services](https://tallysolutions.com/features/connected-services/);
  [payment request help](https://help.tallysolutions.com/tally-prime/banking-utilities/payment-request/);
  [Razorpay–TallyPrime](https://razorpay.com/blog/announcing-razorpay-tallyprime-integration/)).
  WhatsApp onboarding goes through Interakt, "starting at just ₹1300/year"
  ([Tally web story](https://tallysolutions.com/web-stories/tallyprime-with-whatsapp/)).
  **I found no party self-service login** on Tally's connected-services page. That absence is
  inferred, not proven. Tally's model is *push a PDF to the party*, not *let the party pull*.
- **Busy:** a mobile app for the business owner, which needs a Busy Desktop subscription. Users
  share ledgers and invoices over WhatsApp/SMS. I found no customer login
  ([App Store listing](https://apps.apple.com/app/id1662346018); third-party
  [SoftwareSuggest](https://www.softwaresuggest.com/busy-accounting/mobile-app)).
- **Marg:** the *eRetail* app lets **retailers view distributor-wise outstanding**, order, and pay
  via Marg Pay, synced with Marg ERP 9+
  ([App Store](https://apps.apple.com/cd/app/eretail/id1217414784); details from a third-party
  listing). This is the closest Indian analogue: a B2B distributor-to-retailer channel, not a
  general portal. Logic ERP has a similar B2B portal where "customer outstanding and ledger can
  be viewed" ([Logic ERP](https://logicerp.com/Modules/B2B-Order-Management-Software)).
- **Vyapar:** the party statement is an internal report. It has an online-store catalogue link,
  and an undated "Vyapar Network" release note describes in-app invoice sharing between Vyapar
  users. I found no portal ([feature list](https://vyaparapp.in/blog/vyapar-app-features/)).
- **QuickBooks Online:** I found no official customer portal. Intuit's feedback forum carries user
  requests for one, and third-party portals fill the gap
  ([Intuit feedback](https://feedback.qbo.intuit.com/forums/591862-quickbooks-invoice-payments-portal/suggestions/40247722-customer-portal)).
- **Xero:** online statements that customers can view and pay through Stripe. A Xero
  representative on the ideas forum says they stay "live for 90 days". A live statement portal is
  still an open idea
  ([Xero ideas](https://productideas.xero.com/forums/967115-invoices-quotes/suggestions/50499696-customer-statements-allow-customers-to-view-live)).
- **Receivables fintechs** (CredFlow and similar) sit on top of Tally: WhatsApp/SMS/IVR
  reminders and ledger sharing. Third-party pricing is contradictory and unverified
  ([Techjockey](https://www.techjockey.com/detail/credflow)).

**Conclusion:** among tools Indian SMBs actually switch between, **only Zoho ships a true
self-service party portal**. Tally, Busy and Vyapar push documents, and Marg/Logic ship
distributor-ordering apps. A **live, always-reconciled portal driven by a desktop-grade ledger** is
real differentiation for a Tally switcher. That is exactly why its numbers must equal the owner's
screens.

### Frameworks, partners and costs that bite

| Topic | What applies | Source / verification |
|---|---|---|
| **Payment aggregation** | RBI *Regulation of Payment Aggregators Directions, 2025* (15 Sep 2025) consolidates the 2020/21 PA-PG guidelines. A PA "facilitates aggregation of payments made by customers to merchants… and subsequently settles the collected funds", and non-banks need RBI authorisation. **So Comhard must never receive a tenant's customers' money.** The tenant must be the gateway's merchant. | [AZB summary](https://www.azbpartners.com/bank/rbi-issues-consolidated-reserve-bank-of-india-regulation-of-payment-aggregators-directions-2025/); [Taxmann](https://www.taxmann.com/post/blog/rbi-issues-new-rules-for-payment-aggregators-and-banks). Secondary sources, so read the RBI text before contracts. |
| **Razorpay Payment Links** | Create a link with `reference_id`. Webhooks: `payment_link.paid`, `payment_link.partially_paid`, `payment_link.cancelled`, `payment_link.expired` (verified). Partner model: partner auth with `X-Razorpay-Account: <sub-merchant account_id>`, or Technology-Partner OAuth with per-merchant tokens that expire after 90 days plus a refresh token. Standard domestic pricing: 2% + 18% GST on the fee, per Razorpay's blog. | [Webhooks](https://razorpay.com/docs/webhooks/payment-links.md); [partner-auth links](https://razorpay.com/docs/partners/aggregators/partner-auth/payment-links.md); [OAuth integration](https://razorpay.com/docs/partners/platform/onboard-businesses/integrate-oauth/integration-steps); [pricing blog](https://razorpay.com/blog/razorpay-payment-gateway-pricing-explained/). **Unverified:** whether an OAuth token can call Payment Links, or only partner auth can. Ask Razorpay partner support. |
| **UPI without a gateway** | `upi://pay?pa=<VPA>&pn=<name>&am=<amt>&cu=INR&tn=<note>&tr=<ref>`. **There is no payment confirmation back to us**; reconciliation happens in BRS. The public spec copy is the 2017 v1.6 draft, and whether `mc` is mandatory for P2M is disputed. | [NPCI linking spec mirror](https://www.labnol.org/files/linking.pdf); [NPCI circular 37/2017](https://www.npci.org.in/PDF/npci/upi/circular/2017/Circular%20on%20Non-compliance%20of%20UPI%20apps%20to%20Deeplinking%20specs%20Circular%2037-%201.11.2017.pdf). Behaviour across PSP apps needs device testing. |
| **DPDP Act 2023 + Rules** | Rules notified **13 Nov 2025**, phased. Some rules apply immediately. Consent-manager provisions apply at about 12 months (Nov 2026). **Most fiduciary duties** (notice, security safeguards, breach intimation, retention, rights, DPO) apply at about **18 months, 13 May 2027**. Breach: inform the Board without delay, with a detailed report within 72 hours. Processors have no direct duties; the fiduciary must bind them by contract. Some logs must be kept for at least 1 year for specified purposes. | [Hogan Lovells](https://www.hlc.com/en/publications/indias-digital-personal-data-protection-act-2023-brought-into-force-); [AZB](https://www.azbpartners.com/bank/update-indias-digital-personal-data-protection-framework-comes-into-effect/). **Secondary sources disagree on which rule applies when. Verify against the Gazette.** |
| **SMS OTP** | TRAI TCCCPR 2018 **DLT** registration (entity, header, templates; PE–TM chain from 1 Dec 2024). About ₹0.10–0.45 per OTP. | [Plivo DLT guide](https://www.plivo.com/docs/sms/concepts/dlt-registration-process); [Message Central pricing](https://www.messagecentral.com/blog/sms-otp-pricing-india). Vendor-sourced prices. |
| **WhatsApp OTP / notifications** | Needs a WABA via a BSP or the Meta Cloud API, plus template approval. Per-message pricing since 1 Jul 2025. Rate cards effective 1 Apr 2026, with an INR card. | [Meta pricing](https://developers.facebook.com/docs/whatsapp/pricing). **The India authentication rate is unverified** (a third party says about $0.0014 per message). |
| **Email OTP / invites** | Any transactional provider. Amazon SES is about $0.10 per 1,000 (check ap-south-1). Zoho ZeptoMail: 10,000 emails per credit, first credit free, about $2.50 per credit (third-party). SPF/DKIM/DMARC on the sending domain. | [ZeptoMail pricing](https://www.zoho.com/zeptomail/pricing.html); SES figure from [third party](https://www.saaspricepulse.com/blog/amazon-ses-pricing-per-1000-emails-2026). |
| **Email as an authenticator** | NIST SP 800-63B-4 (final, Jul 2025) does not accept email as an out-of-band authenticator at higher assurance. Email OTP is fine for a low-assurance, read-only portal, but it caps what we may later unlock (payments, uploads). | [NIST SP 800-63B-4](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-63B-4.pdf). The exact clause wording was not verified in this session. |

---

## 5. The central decision: where does the party's ledger get computed?

The server has the data (the `change_log`) but no engine. The engines live in `src/core` and
run in the owner's browser. There are three ways to put a statement in front of a party.

| | **A. Server-side projection** (recommended) | **B. Owner-device "publish"** | **C. Ship filtered docs to a sandboxed portal client** |
|---|---|---|---|
| How | A projector reads `change_log` for portal-enabled tenants. It runs `src/core` (esbuild-bundled for Node) on **only** the party ledger + vouchers touching it + the masters those invoices print. It writes an **allow-listed projection**. The portal reads only that. | The owner's app computes the same projection from Dexie and uploads it (manually, or on each sync). The server stores only the projection. | The server sends the party's vouchers, ledger and masters to a portal SPA, which runs `src/core` in the party's browser. |
| What reaches the party | Only projection fields (§7.3). | Only projection fields. | **Raw voucher docs**: other legs, narration, cost-centre allocations, godown/batch, internal ledger ids, other parties on a multi-party Journal/Receipt. |
| Data minimisation | **Structural**: the portal role has no grant on `change_log`. | Structural: the server never has more for the portal than what was published. | **Fails** unless the server redacts docs, which needs full schema knowledge. Then the server has done the hard part, and the remaining math (sum + running balance) is trivial. |
| Freshness | Seconds to a minute after any device syncs. | Only when an owner device is online, fully pulled and pushed, and publishes. Forgotten publishes give stale statements. | Fresh. |
| Correctness risk | Server engine drift → mitigated by sharing `src/core`, stamping `engine_version`, and the owner-side preview diff (§7.8). | A device with unsynced or partially pulled data publishes wrong figures. Two devices race. | Same engines, but on a hostile client. Confidentiality is already lost. |
| New server capability | Payload-aware projector (breaks the "opaque log" tenet *for portal tenants only*; ARCHITECTURE §9 already plans a typed projector). | Upload endpoint + storage. | A redaction layer that is effectively A minus the math. |
| Verdict | **Choose.** | Fallback if the projector is delayed. Could also be the explicit "send statement as on date" action in phase 2. | **Reject.** It discloses other parties' data by construction. |

**Why C is a confidentiality bug, concretely:** a Receipt that settles two customers in one bank
voucher, or a Journal that transfers a balance from party A to party B, is **one doc**. Shipping
"vouchers that touch A" ships B's name, amount and bill references to A. Narrations ("hold
supplies, cheque bounced twice") and `udf` values on the party's own ledger are internal. Even
`Ledger.creditLimit`, `interestRate` and `tdsDeductee` are the owner's business, not the party's.

---

## 6. Options compared (product level)

| Option | Effort (eng-weeks) | External cost | Dependency | Risk | Notes |
|---|---|---|---|---|---|
| **1. Full party portal, server projection (A)** — recommended | MVP 10–13, + phase 2 5–7 | about ₹0 for the MVP; email about $0.10–0.25 per 1k | TLS + separate origin; SMTP for self-service re-login | Medium: projector correctness, new external attack surface | Matches Zoho on the core. A live ledger beats Tally's push model. |
| **2. Signed statement link, no login** ("send statement link") | 3–4 | about ₹0 | TLS | Medium: **anyone holding a forwarded link sees the statement**. Expiring links mitigate, not solve. | What Xero does (90-day online statements). A good *phase-0 teaser*, not what was asked. |
| **3. Owner-device publish (B)** | 7–9 | about ₹0 | Owner device discipline | Stale or partial-sync statements; races | A degraded version of 1. |
| **4. Sandboxed client with raw docs (C)** | 9–11 | about ₹0 | — | **High: cross-party disclosure** | Rejected (§5). |
| **5. Third-party client-portal SaaS** (generic portals that front QuickBooks etc.) | 4–6 integration | Per-seat SaaS fees (unverified) | A vendor, with data outside our residency | High: a DPDP processor chain; no Indian bill-wise/GST semantics | Not recommended. |
| **+ Payment links** (add-on to 1) | 3.5–5 | Tenant pays MDR (about 2% + GST standard) | Tenant KYC'd gateway account; Razorpay partner agreement for OAuth | Medium-high: money movement, webhooks, reconciliation | §7.11 |
| **+ UPI intent QR** (add-on to 1) | 0.5–1 | ₹0 | Tenant's merchant VPA | Low; **no confirmation** | Good interim before a gateway |

---

## 7. Recommended architecture (on our stack)

### 7.1 Shape

```mermaid
flowchart LR
  subgraph Owner devices
    APP["Comsky app (Dexie + outbox)"]
  end
  subgraph Backend :1036 (Fastify)
    SYNC["/sync/push (comhard_app, tenant RLS)"]
    CL[("change_log")]
    HUB["notifyTenant() — in-process"]
    PROJ["portal projector\n(src/core bundle, per-tenant withTenant)"]
    PT[("portal_* projection tables\nFORCE RLS: tenant+cid+ledger")]
    OAPI["/api/portal/* (members, role-checked)"]
    PAPI["/portal-api/* (portal tokens, aud=portal)"]
  end
  subgraph Portal origin (separate host, HTTPS)
    PSPA["Portal SPA (no Dexie, no store)"]
  end
  APP -->|push| SYNC --> CL
  SYNC --> HUB --> PROJ
  CL -->|read new rows| PROJ --> PT
  APP -->|settings, invites, preview| OAPI --> PT
  PSPA -->|bearer + refresh cookie| PAPI -->|role comhard_portal\nSELECT on portal_* only| PT
```

The process boundary stays the single Fastify process (matching the single-VM tier). The
**privilege** boundary is the database role: `/portal-api/*` uses a separate `pg.Pool` connected
as `comhard_portal`.

### 7.2 Portal identity and auth (passwordless, separate namespace)

- **Identity:** `portal_identity(email_norm)`. It is global, holds no tenant data, and is never an
  `app_user`. A person who is both a Comsky tenant member and someone's customer has **two
  unrelated identities**. Nothing is shared, so there is no password to reuse.
- **Grant:** `portal_grant` = (tenant, company `cid`, party `ledger_id`, contact, role
  customer|vendor, status). One identity may hold many grants, which gives the P4 switcher.
- **Invite (MVP, no credentials):** the owner creates a grant and the server returns a
  **single-use invite URL once**: `https://<portal-origin>/i#<token>`. The token sits in the URL
  **fragment**, so it never reaches server logs, proxies or `Referer`. It is stored as SHA-256,
  expires in 7 days, and is bound to the invited email. The owner delivers it through their own
  channel (copy, `mailto:`, `wa.me` click-to-chat). The SPA posts the token to
  `/portal-api/auth/invite/accept`. A GET never consumes a token, so email-scanner prefetch
  cannot burn it.
- **First login:** accepting the invite creates the session and offers **passkey enrolment**
  (WebAuthn: no third party, no credential for us; needs HTTPS; library e.g.
  `@simplewebauthn/server`). If the party skips the passkey, the session's refresh token
  (30 days sliding) is the only way back.
- **Returning login:**
  - With a passkey: passkey.
  - Otherwise, once SMTP is configured: **email OTP**. Six digits, HMAC-hashed at rest, 10-minute
    life, 5 attempts. The same email also carries a magic link. `/auth/start` always answers
    "if this address has access, we sent a code", so it never reveals whether an address exists.
  - Until SMTP exists: "Request a new link", which drops into the owner's portal inbox, and the
    owner re-sends an invite. That is acceptable for a pilot of tens of parties.
  - SMS or WhatsApp OTP comes later and needs DLT or WABA (§4).
- **Tokens:**
  - Access JWT: 10 minutes, `aud: "comsky-portal"`, signed with **`PORTAL_JWT_SECRET`** (new env
    variable; never `JWT_SECRET`). Claims: `{sub: identity, gid: grant, fam}`.
  - Refresh: opaque, rotating with reuse detection. Copy `tokens.js` `rotateRefresh` into
    `portal_session_family` / `portal_refresh`. It travels in an `HttpOnly; Secure;
    SameSite=Strict; Path=/portal-api/auth` cookie.
  - The SPA holds the access token **in memory only**.
  - **Every request** re-reads the grant (active? not revoked?) and derives `(tenant, cid,
    ledger)` from the **database row**, not from claims, so revocation is immediate.
- **Tighten tenant tokens in the same change:** add `aud: "comsky-app"` in `signAccess`, and
  make `verifyAccess` reject a token with a different or missing `aud` unless it has a valid
  `fam`. Every token `issueTokens` mints has `fam`, so this tightening does not log anyone out.
  The admin token (`admin.js`) should get its own secret or audience for the same reason
  (related hardening).

### 7.3 Data minimisation — the allow-list

The portal shows **an allow-list of derived fields**. Nothing is shown merely because it is
"not on a deny-list".

| Shown to the party | Derived from | Rule |
|---|---|---|
| Business letterhead: mailing name, address, state, pincode, GSTIN, PAN, business phone/email, logo | `Company` | Same fields as `CompanyBlock` on the printed invoice. **Never** `vault`, `security`, `features`, `tan`, `deductorName`, `gstSetup`. |
| Party's own name, address, state, GSTIN (so they can spot errors) | Party `Ledger` | Read-only; correction requests go to the owner's inbox. **Never** `creditLimit` (unless enabled), `interestRate`, `priceLevelId`, `tdsDeductee`, `udf`, `openingBankBalance`, bank fields. |
| Statement line: date, **party-facing kind**, voucher number, party's own Dr/Cr, bill refs, document link | `Voucher` | Amount = sum of **the party's own entries only** (as `LedgerVouchers` `vAmt` does). Excludes `!postsToLedgers(v)` (optional, Memorandum) and dates after as-on (post-dated). |
| Kind label | `Voucher.base` (and direction) | Customer view: Sales → *Invoice*, Receipt → *Payment received*, CreditNote → *Credit note*, DebitNote → *Debit note*, Payment → *Refund paid*, Journal and others → *Adjustment*. Vendor view: Purchase → *Your invoice <supplierInvNo>*, Payment → *Payment made*, and so on. **Never the owner's voucher-type names** ("Sales – Export Mumbai" reveals internal structure). |
| "Particulars" (counter-ledger name, as in the owner's Ledger Vouchers) | other entries | **Off by default.** If enabled, only non-party ledgers (e.g. "HDFC Bank", "Sales"), and **never** another Sundry Debtor/Creditor. |
| Narration | `Voucher.narration` | **Off by default** (free text, often internal). Per-company opt-in, with a warning. |
| Open bills: name, bill date, due date, original amount, balance, ageing bucket | `outstandingBills([party], vouchers, asOn)` | As-on = `ageingAsOn(today, today)` in IST, per the "never age into the future" gotcha. On Account shown as one line. |
| Invoice copy | `buildInvoiceModel(...)`, then `stripForPortal()` | Only invoice-shaped vouchers where **`v.partyId === party.id`**: Sales, CreditNote, DebitNote for customers; PurchaseOrder later for vendors. **Strip** `IRN`/`Ack. no.` while e-invoicing is mock. Banks and terms come from portal settings, not device storage. **No copy of Receipts/Payments** in v1: their accounting print model lists every ledger in the voucher, including other parties on a bulk receipt. |
| Summary: opening, closing, overdue, last payment | above | Computed from the same lines, so the page foots. |
| "As per books synced on …" | `as_of_seq` → `change_log.ts` | Always printed. |

**Never shown, by construction (the leak test asserts it):** any other ledger's name, balance or
GSTIN; cost-centre allocations; godown/batch; stock cost or valuation; `optional` vouchers;
internal ids (documents are addressed by an opaque `doc_key = HMAC(tenant_secret, voucher_id)`);
the owner's users or operators; anything from a company the grant does not name.

### 7.4 The projection engine (shared code) and the projector (server process)

**`src/core/portalProjection.ts`** is pure, like `invoiceModel.ts`. It is the **single source** used
by the server projector (via an esbuild bundle) **and** the owner app's "View as party" preview:

```ts
export interface PortalInput {
  company: Company; party: Ledger; role: 'customer' | 'vendor';
  asOn: string;                                   // IST date; ageingAsOn(today, today)
  vouchers: Voucher[];                            // current versions that have an entry on party.id
  voucherTypes: VoucherType[];
  printCtx: { ledgersById: Map<string, Ledger>; itemsById: Map<string, StockItem>; unitsById: Map<string, Unit> };
  settings: PortalVisibility;                     // owner choices: narration, particulars, banks, terms, …
}
export interface PortalProjection {
  summary: PortalSummary; lines: PortalLine[]; bills: PortalBill[];
  docs: { docKey: string; kind: string; number: string; date: string; model: InvoiceModel; hash: string }[];
}
export function partyProjection(inp: PortalInput): PortalProjection;
```

**Assert suite `scripts/assert/portal.assert.ts`** (runs with `npm run assert`). It is the gate for
every change:

1. **Leak test.** Seed canary strings into everything that must not leak: another party's name
   and GSTIN on a two-party Journal and a bulk Receipt, a narration, a cost-centre name, a godown,
   a batch, a UDF value, a voucher-type name, the company `vault` hint. Serialise the **entire**
   projection and assert that none of the canaries appear.
2. Optional and Memorandum vouchers are excluded; a post-dated voucher after as-on is excluded.
3. **Reconciles:** projection closing = `LedgerVouchers` closing = `closingBalances()` for the same
   ledger and date.
4. Opening bill-wise breakup (`Ledger.openingBills`) is aged from original dates.
5. An edit that **removes** the party from a voucher removes the line; a delete marks the document
   *withdrawn*.
6. Mock IRN never appears in `docs[].model`.
7. A round-down invoice shows its party-leg amount (`voucherHeadlineAmount` gotcha).

**Projector** (`backend/src/portal/projector.js`, in-process, guarded by a Postgres advisory lock
so a second instance cannot double-run):

- **Trigger.** Both the sync push and REST writes already call `notifyTenant(tenantId, seq)`.
  Add a sibling hook that marks the tenant dirty **only if it has ≥1 enabled grant**. Tenants
  without the portal cost nothing. On boot, catch up every portal tenant from its
  `portal_cursor(tenant_id, seq)`.
- **Per dirty tenant**, inside `withTenant` (the existing comhard_app path, tenant RLS; **no
  BYPASSRLS needed**), read `change_log WHERE seq > cursor`. Mark parties dirty:
  - `vouchers` put or delete → ledger ids in the **new** payload's `entries` **and** in the
    **previous** version's entries (an edit can move a voucher off a party).
  - `ledgers` → that ledger.
  - `companies`, `voucherTypes`, `stockItems`, `units`, tax ledgers → every enabled party of that
    `cid`. Coarse, but these are rare.
- **Load current docs** for a dirty party:
  - Vouchers ever touching the party come from a **partial GIN index**:
    `CREATE INDEX ON change_log USING gin (payload jsonb_path_ops) WHERE tbl = 'vouchers'`, queried
    with `payload @> '{"entries":[{"ledgerId":"<id>"}]}'`.
  - Keep only doc ids whose **latest** version (new index `(tenant_id, tbl, doc_id, seq DESC)`) is
    a put that still touches the party. This is the same "highest seq wins" rule the clients and
    REST converge on.
  - Then load the referenced masters by the same index.
- **Write** the party's projection in one transaction: replace lines, bills and summary; upsert
  documents by `content_hash`, bumping `version` and `changed_at` when the hash changes. Stamp
  `as_of_seq`, `computed_at` and `engine_version` (the bundle hash, so an engine upgrade
  re-projects everything).
- **Disable** a party → delete its projection rows (minimisation at rest). Party ledger deleted →
  suspend its grants.
- **Targets:** portal reflects a push in under 60 s; a 2,000-voucher party re-projects in tens of
  milliseconds of JavaScript (estimate, to be confirmed with `npm run perf`).
- **Migration notes:** `migrate.js` wraps each file in a transaction, so `CREATE INDEX
  CONCURRENTLY` needs a small runner change (a `-- no-transaction` file marker). Today's tables
  are small enough to build the index in a short maintenance window instead. A GIN index adds
  write cost on push; if that shows up, replace it with a projector-maintained
  `portal_vch_index(tenant_id, cid, ledger_id, doc_id)` side table.

### 7.5 Postgres: tables, roles, RLS (proposed `backend/sql/007_portal.sql`)

```sql
-- Roles (created by ops; passwords in backend/.env, never in the repo)
--   comhard_portal : LOGIN, NOBYPASSRLS, not owner. Serves /portal-api only.
-- comhard_app keeps serving members; the projector runs as comhard_app inside withTenant().

-- ── owner-managed config (tenant RLS, via withTenant from /api/portal/*) ──
CREATE TABLE portal_settings (
  tenant_id uuid NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
  cid text NOT NULL,
  enabled boolean NOT NULL DEFAULT false,
  visibility jsonb NOT NULL DEFAULT '{}'::jsonb,   -- narration, particulars, creditLimit, historyFrom, …
  bank_ledger_ids text[] NOT NULL DEFAULT '{}',    -- what invoice copies print (owner-chosen, never "first N")
  terms text, banner text, dispute_contact text,
  updated_by uuid NOT NULL, updated_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, cid));

CREATE TABLE portal_identity (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email_norm text UNIQUE NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(), last_login_at timestamptz);

CREATE TABLE portal_grant (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
  cid text NOT NULL, ledger_id text NOT NULL,
  role text NOT NULL CHECK (role IN ('customer','vendor')),
  invite_email text NOT NULL, contact_name text,
  identity_id uuid REFERENCES portal_identity(id) ON DELETE SET NULL,  -- null until accepted
  status text NOT NULL CHECK (status IN ('invited','active','suspended','revoked')),
  created_by uuid NOT NULL, created_at timestamptz NOT NULL DEFAULT now(),
  revoked_by uuid, revoked_at timestamptz,
  UNIQUE (tenant_id, cid, ledger_id, invite_email));

CREATE TABLE portal_token (            -- invite / magic / otp; only hashes stored
  token_hash text PRIMARY KEY, kind text NOT NULL CHECK (kind IN ('invite','magic','otp')),
  grant_id uuid, email_norm text, attempts int NOT NULL DEFAULT 0,
  expires_at timestamptz NOT NULL, used_at timestamptz, created_at timestamptz NOT NULL DEFAULT now());
CREATE TABLE portal_session_family (id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  identity_id uuid NOT NULL, grant_id uuid NOT NULL, revoked boolean NOT NULL DEFAULT false,
  created_at timestamptz NOT NULL DEFAULT now(), last_seen_at timestamptz);
CREATE TABLE portal_refresh (token_hash text PRIMARY KEY, family_id uuid NOT NULL,
  expires_at timestamptz NOT NULL, revoked boolean NOT NULL DEFAULT false);
CREATE TABLE portal_passkey (id bytea PRIMARY KEY, identity_id uuid NOT NULL,
  public_key bytea NOT NULL, sign_count bigint NOT NULL DEFAULT 0, created_at timestamptz NOT NULL DEFAULT now());

-- ── projections (written by the projector as comhard_app; read by comhard_portal) ──
CREATE TABLE portal_company_profile (tenant_id uuid, cid text, letterhead jsonb NOT NULL,
  PRIMARY KEY (tenant_id, cid));
CREATE TABLE portal_party_summary (tenant_id uuid, cid text, ledger_id text,
  party jsonb NOT NULL, opening numeric(18,2), closing numeric(18,2), overdue numeric(18,2),
  as_of_seq bigint NOT NULL, as_of_ts timestamptz NOT NULL, engine_version text NOT NULL,
  PRIMARY KEY (tenant_id, cid, ledger_id));
CREATE TABLE portal_line (tenant_id uuid, cid text, ledger_id text, line_no int,
  vdate date NOT NULL, kind text NOT NULL, number text, bill_refs text[],
  debit numeric(18,2) NOT NULL DEFAULT 0, credit numeric(18,2) NOT NULL DEFAULT 0,
  narration text, particulars text, doc_key text,
  PRIMARY KEY (tenant_id, cid, ledger_id, line_no));
CREATE INDEX ON portal_line (tenant_id, cid, ledger_id, vdate);
CREATE TABLE portal_bill (tenant_id uuid, cid text, ledger_id text, bill_key text,
  name text, opened_on date, due_date date, amount numeric(18,2), balance numeric(18,2), doc_key text,
  PRIMARY KEY (tenant_id, cid, ledger_id, bill_key));
CREATE TABLE portal_document (tenant_id uuid, cid text, ledger_id text, doc_key text,
  kind text, number text, vdate date, model jsonb NOT NULL, content_hash text NOT NULL,
  version int NOT NULL DEFAULT 1, status text NOT NULL CHECK (status IN ('current','withdrawn')),
  first_published_at timestamptz NOT NULL, changed_at timestamptz NOT NULL,
  PRIMARY KEY (tenant_id, cid, ledger_id, doc_key));

-- ── audit (append-only) ──
CREATE TABLE portal_audit (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id uuid NOT NULL, cid text, ledger_id text, grant_id uuid, identity_id uuid,
  actor_kind text NOT NULL CHECK (actor_kind IN ('party','member','system','platform')),
  actor_id uuid, action text NOT NULL, target text, content_hash text,
  ip inet, ua text, at timestamptz NOT NULL DEFAULT now(), detail jsonb);
```

**RLS, written once and applied to every projection table** (`portal_line` shown):

```sql
ALTER TABLE portal_line ENABLE ROW LEVEL SECURITY;
ALTER TABLE portal_line FORCE  ROW LEVEL SECURITY;          -- even the owner role is subject

-- the party: exactly one (tenant, company, ledger), read-only
CREATE POLICY party_read ON portal_line FOR SELECT TO comhard_portal
  USING (tenant_id = current_setting('app.tenant_id',     true)::uuid
     AND cid       = current_setting('app.portal_cid',    true)
     AND ledger_id = current_setting('app.portal_ledger', true));

-- members + projector: their tenant
CREATE POLICY tenant_rw ON portal_line TO comhard_app
  USING      (tenant_id = current_setting('app.tenant_id', true)::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);

GRANT SELECT ON portal_line, portal_bill, portal_document, portal_party_summary,
                portal_company_profile TO comhard_portal;   -- company_profile policy: tenant+cid only
GRANT INSERT ON portal_audit TO comhard_portal;
-- Deliberately NO grant to comhard_portal on: change_log, company, tenant, membership, app_user,
-- api_key, webhook_*, subscription, invoice, voucher_counter, portal_settings, portal_grant.
```

- Unset GUCs give `NULL`, and the comparison fails closed (the same idiom as `002_p2.sql`).
- **Login lookups that cross tenants**, from email to that identity's grants, go through 3–4
  `SECURITY DEFINER` functions with a pinned `search_path`:
  - `portal_auth_begin`
  - `portal_grants_for(identity)`, which returns display labels only
  - `portal_grant_scope(grant, identity)`, which returns `(tenant, cid, ledger)` only for an
    active grant
  - `portal_session_check`
- comhard_portal gets `EXECUTE` on those and nothing on the auth tables directly.
- `withPortalScope(scope, fn)` mirrors `db.js` `withTenant`: `BEGIN`, three `set_config(…, true)`
  calls, `fn`, `COMMIT`. It runs on the portal pool.
- The existing pattern of metadata on the owner pool (`platform.js` api keys) must **not** be
  used anywhere on the portal request path.
- **CI isolation suite** (new; HANDOFF lists CI as missing, so it starts as an assert script run
  against a scratch database), as `comhard_portal`:
  - with party A's GUCs: zero rows for party B in the same tenant; zero rows for another
    tenant; zero rows with GUCs unset
  - `SELECT` on `change_log` → *permission denied*
  - a forged JWT signed with `JWT_SECRET` → 401 on `/portal-api`
  - a portal JWT → 401 on `/sync/pull`

### 7.6 Endpoints

**Members (owner app)**, under `/api/portal/*`. They use `app.authenticate` plus a **new
`requireTenantRole('owner','admin')`** check against `membership.role`, and run through
`withTenant`:

| Method | Path | Purpose |
|---|---|---|
| GET/POST | `/api/portal/settings?cid=` | Company defaults (§7.3 switches, banks, terms, banner). |
| GET | `/api/portal/parties?cid=` | Grants + status + `as_of_seq`/closing per party (for the preview diff). |
| POST | `/api/portal/grants` | Create grant(s) (bulk for O2) → invite URL(s), **returned once**. |
| POST | `/api/portal/grants/:id/reinvite` · `/revoke` · `/suspend` | Lifecycle; revoke kills session families. |
| GET | `/api/portal/preview/:cid/:ledgerId/{summary,statement,bills,documents/:key}` | Exactly the portal responses, served through the portal pool with that party's GUCs. Audited as `preview`. |
| GET | `/api/portal/activity?cid=&ledgerId=` | Audit feed for the *Portal Activity* report. |
| GET | `/api/portal/inbox` (phase 2) | Link requests, queries, confirmations, payment claims. |

**Party**, under `/portal-api/*` (added to `API_PREFIXES` in `server.js`). Body limit 16 KB, not
the global 20 MB. `Cache-Control: no-store` on every response.

| Method | Path | Purpose |
|---|---|---|
| POST | `/portal-api/auth/start` `{email}` | Always 202. Sends OTP + magic link when SMTP is configured. |
| POST | `/portal-api/auth/verify` `{email, code}` or `{token}` | Session: access token in the body, refresh in a cookie. |
| POST | `/portal-api/auth/invite/accept` `{token}` | First login. |
| POST | `/portal-api/auth/passkey/{options,register,login}` | WebAuthn. |
| POST | `/portal-api/auth/refresh` · `/logout` · `/switch {grantId}` | Session lifecycle; `switch` re-scopes like `/auth/switch`. |
| POST | `/portal-api/auth/request-link` `{email}` | Pre-SMTP fallback → owner inbox. Always 202. |
| GET | `/portal-api/me` | Identity + grants (display labels only). |
| GET | `/portal-api/summary` · `/company` | Summary, letterhead. |
| GET | `/portal-api/statement?from&to&cursor` | Opening = party opening + Σ lines before `from`; running balance via a SQL window over `(vdate, line_no)`. Max 3 financial years per call. |
| GET | `/portal-api/bills` · `/documents?kind&cursor` · `/documents/:docKey` | Bills; document list; one document (logged with `content_hash`). |

### 7.7 Clients

- **Portal SPA** (`portal/`, a third Vite app beside `admin/`):
  - It imports **only** pure modules (`src/core/money`, `dates`) and an **extracted** store-free
    `InvoiceSheet` component (PrintVoucher's presentational half + `PagedDoc` + `print.css`).
  - **No Dexie, no Zustand store, no `sync.ts`.**
  - Mobile-first; IBM Plex and the teal tokens from `styles.css` so it reads as the same family.
  - The tenant's letterhead and logo lead; a small "Powered by Comsky Books" footer, with
    **no "Tally" anywhere** (CLAUDE.md branding rule).
  - Keyboard-friendly (`/` search, Enter opens, Esc back), though not a Tally keymap.
  - **PDF:** "Download PDF" opens the browser print dialog on the paginated sheet (Save as PDF on
    desktop and Android; iOS goes through share → print). A true server-generated PDF (headless
    Chromium or Gotenberg rendering the same HTML, cached by `content_hash`) is phase 2.
  - **CSP:** `default-src 'self'`, no inline script, `frame-ancestors 'none'`,
    `Referrer-Policy: no-referrer`.
- **Owner app** — new screens via the new-screen checklist (CLAUDE.md):
  - *Portal Access*: list + contact sub-panel + invite/revoke/preview. It follows the
    `src/ui/admin/ApiKeys.tsx` pattern of a server-backed screen with a first-class signed-out
    state.
  - *Portal Settings.*
  - *Portal Activity*: a ReportShell, so Print/Export work automatically.
  - *View as party* runs `partyProjection` locally on Dexie data.

### 7.8 Sync semantics and offline behaviour

- **The projection is a pure function of `change_log` up to `as_of_seq`.** Recomputing is
  idempotent. The party always sees "As per books synced on <as_of_ts>".
- **Unsynced work is invisible to the party**, by design. Each owner device keeps its own Dexie,
  and the portal reflects the cloud. The owner's *Portal Access* screen compares the server's
  `as_of_seq` and closing per party with the local preview, and shows **"Portal is behind this
  device — N vouchers not yet synced"** when they differ. This is the same kind of honest
  mismatch report the consultants asked for on sync.
- **The portal never writes to `change_log`.** Phase-2 party input (queries, confirmations,
  payment claims) lands in `portal_*` tables, reaches owners online through the inbox, and enters
  the books **only** when an owner acts, for example by creating a Receipt with a pre-filled
  Agst Ref. External parties cannot create books data, and there is no conflict surface.
- **Portal config is server-authoritative and online-only** (like API keys). An offline "revoke"
  sitting in an outbox for days would be a security bug. Offline, the screen shows the last
  fetched state read-only, with "Offline — changes need a connection".
- **Who can use it:** only companies that sync to the cloud. A purely local or Electron-only
  company has no portal. Say so in the UI.
- **The party side is online-only.** No PWA or offline cache of financial data (shared office
  PCs and phones).

### 7.9 Audit trail

- `portal_audit` is append-only: comhard_portal has INSERT only, and app roles get no
  UPDATE/DELETE grants.
- Events:
  - party: `invite_accepted`, `login`, `otp_sent`, `otp_failed`, `passkey_added`,
    `view_statement {from,to}`, `view_document {doc_key, content_hash, version}`,
    `download_document`, `switch`, `logout`
  - member: `settings_change {diff}`, `grant_create`, `reinvite`, `revoke`, `preview`
  - system: `projection_failed`, `grant_suspended`
- **Document versions are evidence.** When the hash of a document the party has already viewed
  changes, the portal shows "Revised on <date>". The audit holds the hash the party actually saw,
  so a dispute ("you changed the rate after sending") is answerable. Phase 2 can keep prior
  model JSON in `portal_document_version`. Silently re-issuing a GST invoice is a compliance
  smell worth surfacing.
- **Retention (proposal, needs counsel):**
  - Keep IP and user agent for 1 year (DPDP Rules require some logs for at least 1 year), then
    null them.
  - Keep the event rows as long as the books, which are retained for GST/Income-tax.

### 7.10 Abuse and rate limits

| Surface | Limit (proposal) | Notes |
|---|---|---|
| `auth/start`, `request-link` | 3 per 15 min and 10 per day per email; 20 per hour per IP; **per-tenant send budget** (e.g. 200 per day) + global circuit breaker | Uniform 202 and constant-ish timing (no enumeration); protects the sender's domain reputation and our bill. |
| OTP verify | 5 attempts per code, then dead; per-IP and per-email lockout with `ratelimit.js` backoff | Codes HMAC-hashed with a server pepper. |
| Invite tokens | Single use, 7 days, bound to email; reinvite invalidates earlier ones | |
| Reads | 120 per minute per session; documents 60 per hour; statement ≤ 3 FY per call, paginated | Stops scraping by a compromised party account. |
| IP source | **Socket address**, unless a known reverse proxy sets XFF (then trust only that hop) | Fixes the spoofable `clientIp()` for this surface. |
| Limiter store | In-memory for the MVP (single VM, matches today); Postgres or Redis counters at the next tier | It resets on restart, which is a known and documented limit. |
| Platform | Per-tenant kill switch; anomaly alert on a burst of `otp_failed` | Needs the monitoring that HANDOFF lists as missing. |
| Bots | No CAPTCHA by default. Cloudflare Turnstile is an option later and needs a (free) site key. | |

### 7.11 Payment links — what is actually needed

**Today:** `backend/src/billing/*` is **Comhard's own** subscription billing. `RAZORPAY_KEY_ID`
is Comhard's merchant account, the default provider is MOCK, and mock `verifyWebhook()` returns
`true`. **None of it can collect a tenant's receivables.** If the tenant's customer paid into
Comhard's account, Comhard would be aggregating payments (RBI PA Directions 2025) and mixing
client money.

**What a real implementation needs:**

1. **Each tenant's own KYC'd merchant account** with a gateway. Funds settle straight to the
   tenant; MDR is the tenant's cost.
2. **Connection model.** Choose one:
   - **(a) Bring your own keys.** The tenant pastes its Razorpay key ID, secret and webhook
     secret. This needs **encrypted secret storage**, which does not exist yet: AES-256-GCM
     under a server master key with a key id, moving to the KMS envelope of ARCHITECTURE §8
     later.
   - **(b) Razorpay Technology Partner / OAuth.** Comhard signs the partner agreement and gets a
     client ID and secret. Tenants click "Connect Razorpay". Tokens are per merchant (90 days +
     refresh). Better UX, and possibly partner revenue share, but it adds a contract dependency
     and an open question about API scope (§4).
   - A **provider interface** like `billing/provider.js` keeps Cashfree or PayU open. It is a
     *separate* interface, because a different merchant is collecting.
3. **Flow:**
   1. The party taps Pay on an open bill.
   2. `POST /portal-api/bills/:billKey/pay` creates a Payment Link on the tenant's account:
      amount = current balance (partial payment only if the owner allows it),
      `reference_id = <opaque payment id>`, customer email or phone only if the party consents.
   3. The party is redirected to the gateway.
   4. The `payment_link.paid` / `partially_paid` webhook arrives at
      `/portal-api/pay/webhook/:endpointId`. Its HMAC is verified with the **tenant's** webhook
      secret (the raw-body parser pattern in `webhooks.js`).
   5. A `portal_payment` row is written, and an **inbox item** is created for the owner.
4. **No automatic posting in v1.** The owner accepts the item in the app, which creates a Receipt
   locally: Dr gateway clearing (or bank) / Cr party Agst Ref bill. Gateway charges and GST on
   the fee are posted at settlement.
   - The server **cannot** safely pick the bank ledger, voucher type and numbering, or the
     MDR/TDS split, for an offline-first book.
   - Auto-posting rules can come later.
   - The party sees "Payment received by gateway, pending posting" until the books catch up.
5. **Interim, no credentials:**
   - **UPI intent / QR** with the tenant's *merchant* VPA, `am` = bill balance and
     `tr`/`tn` = bill reference.
   - Then "I've paid — UTR ____", which creates a *payment claim* for the owner to match in
     BRS.
   - This must be labelled honestly: payment is **not confirmed** until the owner reconciles it.

---

## 8. Security, privacy and legal notes

- **Who is who under DPDP.**
  - The **tenant business is the Data Fiduciary** for its parties' personal data.
  - **Comhard is its Data Processor.** The ToS needs a data-processing clause, which needs
    counsel.
  - Personal data in scope:
    - **proprietorships** (name, address, phone, email; a GSTIN embeds a PAN)
    - **contact persons** of company parties
    - portal **identity emails** and **IP/UA logs**
  - Data about a *company* party is not personal data; its contact person's details are.
- **Notice and consent at first login.** An itemised notice in plain language. It covers:
  - who the fiduciary is: the tenant, with its grievance contact
  - what is processed and why: providing account statements and documents
  - the processor: Comhard
  - rights and how to withdraw
  - the Board complaint route

  Record the acceptance with the notice version (the `consent_records` idea in ARCHITECTURE §8).
  Whether §7(a) "voluntarily provided for a specified purpose" also covers this is a legal
  judgement. Recording consent is the safe default.
- **Purpose limitation — the growth trap.** Using portal identities to market Comsky to
  parties ("your supplier uses Comsky — sign up!") makes **Comhard a fiduciary** for that data
  and needs its own notice and consent. v1 does not do it. A static footer link is fine.
- **Rights:**
  - *Access*: the portal itself.
  - *Correction*: a request routed to the owner, because the ledger master is the books.
  - *Erasure*: delete the identity, passkeys and sessions, and pseudonymise audit rows. **Books
    are retained** under GST and Income-tax law, and the response says so (ARCHITECTURE §8
    statutory carve-out).
- **Breach readiness.** The portal is our first **internet-facing surface for people who are not
  our customers**. DPDP breach intimation (Board without delay + a detailed report within 72
  hours, per secondary sources) applies from the 18-month mark (May 2027). Build the runbook now.
  An **external security review before go-live** is already on the HANDOFF Phase B list; the
  portal makes it mandatory.
- **Transport and origin.** HTTPS only (HSTS), a separate origin, `Secure` cookies, a strict CSP,
  and an explicit CORS allow-list. **Do not launch on the HTTP deployment.**
- **Residency.** Projections live in the same Postgres in India. If the email provider is outside
  India, the OTP email (address + code) is a cross-border transfer. DPDP uses a negative-list
  approach, so this is not banned, but it should be a conscious choice. Amazon SES in Mumbai
  avoids the question.
- **GST.**
  - A portal copy of a tax invoice is a copy of an already-issued document. Its content must
    match what was issued, hence the version stamping.
  - **Mock IRN/QR is stripped.** Once real IRP integration exists, Rule 48(4) puts the QR code on
    the e-invoice, so the copy must carry the real one.
  - A statement of account is not a statutory document. It carries "E&OE; subject to
    reconciliation".
- **Balance confirmation (phase 2).** A party's in-portal "I confirm the balance as on …" is
  click-wrap **audit evidence** (useful to statutory auditors), **not** a digital signature. Copy
  must not overclaim. Evidentiary weight under the Bharatiya Sakshya Adhiniyam, 2023 is a counsel
  question.
- **Brand.** The portal carries the tenant's brand plus "Powered by Comsky Books". Zoho is
  mentioned only in internal documents like this one. No "Tally" in any portal string.

---

## 9. MVP without external credentials vs what needs a contract or credential

| Capability | Needs a partner contract or paid credential? | Notes |
|---|---|---|
| Projector, projection tables, RLS role, isolation + leak tests | **No** | Pure engineering. |
| Portal SPA: summary, statement, open bills, invoice copy, print-to-PDF | **No** | |
| Owner screens: settings, access, bulk enable, preview, activity | **No** | |
| Invite links delivered by the owner (copy, `mailto:`, `wa.me`) | **No** | `wa.me` click-to-chat needs no API. |
| Passkeys (WebAuthn) | **No contract**, but **needs HTTPS** | Free library. |
| UPI intent QR + "I've paid" claim | **No** (tenant's own VPA) | Not confirmed; reconciled in BRS. |
| **TLS certificate** | **No contract** (Let's Encrypt is free), but **owner action**: a DNS API token for DNS-01, or 80/443 routed to the VM, plus a portal hostname or extra port | **Go-live blocker.** |
| Email OTP / invite emails | **Yes, a credential**: SMTP or SES/ZeptoMail account + SPF/DKIM/DMARC on the sending domain | Cheap (about $0.10–0.25 per 1k emails); no contract beyond provider ToS. |
| SMS OTP | **Yes**: DLT entity, header and template registration + an SMS provider | Weeks of paperwork; about ₹0.10–0.45 per OTP. |
| WhatsApp notifications / OTP | **Yes**: WABA via Meta or a BSP + template approval | Per-message fees. |
| Online payment links | **Yes**: tenant gateway KYC (each tenant) + either tenant keys or a Comhard–Razorpay partner agreement | Never through Comhard's own merchant account. |
| Server-generated PDFs | **No credential**; needs CPU/RAM headroom on the single VM (headless Chromium/Gotenberg) | Phase 2. |

**Bottom line:** **Yes**, the read-only MVP can be built and fully tested on a dev or staging
host without any external credential. **It must not be exposed to real parties until TLS and a
separate origin exist.** The only paid credential worth adding early is a transactional email
account, so parties can sign back in without the owner's help.

---

## 10. Phased plan and effort

Estimates are for one senior full-stack engineer who knows this codebase, working AI-assisted,
in engineer-weeks. They include tests but not calendar slack. A second engineer roughly halves
phases 1–2 because the client and server tracks parallelise. HANDOFF's advice ("audit-first,
don't blind-trust the AI-built code") applies: the review lines are not optional.

| Phase | Scope | Effort | Exit criteria |
|---|---|---|---|
| **0. Prerequisites** | TLS on a portal hostname/origin (Caddy or nginx with automatic certificates, or DNS-01). `aud` on tenant tokens + `verifyAccess` tightening. Socket-IP for limits. CORS allow-list for the portal origin. `requireTenantRole`. | 1–1.5 (+ owner DNS/port action) | `curl -I https://<portal-host>` gives a valid certificate; a token minted with `JWT_SECRET` but `aud=comsky-portal` is refused by `/sync/pull`. |
| **1a. Projection core** | `src/core/portalProjection.ts`, `stripForPortal`, the party-label map, `portal.assert.ts` (leak test + reconciliation), extraction of a store-free `InvoiceSheet`. | 2–2.5 | Leak test green with canaries; projection closing = Ledger Vouchers closing on the sample company. |
| **1b. Server** | `007_portal.sql` (tables, roles, FORCE RLS, SECURITY DEFINER auth functions); esbuild engine bundle in the backend deploy; projector (dirty hook, cursor, GIN lookup, atomic write); isolation suite. | 3–3.5 | Isolation suite green as `comhard_portal`; a push reflects in the portal in under 60 s; disabling purges rows. |
| **1c. Portal auth + API** | Invite accept, passkeys, refresh-cookie sessions, switch, request-link, read endpoints, audit writes, rate limits. | 2–2.5 | Reuse/replay of invite → 401; revoked grant → 401 on the next request; no enumeration difference on `/auth/start`. |
| **1d. Clients** | Portal SPA (summary, statement, bills, document viewer + print). Owner screens: Access (incl. bulk), Settings, Activity, View-as-party, behind-device indicator. | 2.5–3 | A live pilot on staging with the seeded sample company (`scripts/seed/`). |
| **1e. Hardening** | Internal adversarial review (the project's finder → refuter method), CSP and headers, load test of the projector on a 50k-voucher tenant, runbook. | 1 | No open high-severity findings. |
| **= MVP** | | **≈ 10–13** (+ phase 0) | Real parties onboarded **only after phase 0 + an external security review**. |
| **1.5 Email** | Provider adapter, OTP + magic link, invite emails, domain authentication. | 1 | Party signs back in without the owner. |
| **2. Engagement** | Queries/comments (sanitised, size-limited) + owner inbox; balance confirmation; vendor view with TDS deducted + payment advice; document version history; server PDF; notifications to owners (SSE nudge + email). | 5–7 | |
| **3. Payments** | UPI intent + claims (0.5–1); tenant-key Razorpay with encrypted secrets, webhook, inbox → Receipt (3–4); partner OAuth after the agreement (2–3). | 3.5–5 (+2–3 for OAuth) | A paid link shows in the inbox; accepting it books a Receipt Agst Ref; signature verified; no mock path in production. |
| **4. Vendor uploads, SO/quote acceptance, WhatsApp** | Needs object storage (ARCHITECTURE §9 attachments), malware scanning, image/PDF ≤ 5 MB (Zoho parity); Sales Order accept/decline; BSP notifications. | 6–8 | Shared with the attachments and approvals asks. |

---

## 11. Open questions the owner must answer

1. **Portal address.** Which hostname? It cannot share `books.comhard.in:1036` with the owner app
   (§3, `localStorage` tokens). It needs 443 routed to the VM, or another forwarded port, plus a
   certificate route (DNS API token, or 80/443). Should it carry the Comsky name or a neutral
   domain?
2. **Customers first, or customers and vendors together?** Zoho gates the vendor portal at
   Premium. Do we use the same commercial split?
3. **Plan gating.** Which plan includes it, and is there a cap on parties or contacts? This needs
   backend enforcement, which does not exist today (only `canSync` is enforced).
4. **When a tenant's subscription lapses:** does the portal go dark, stay read-only, or continue?
   Parties are not our customers, but they will notice.
5. **Default visibility:** narration off? "particulars" off? credit limit hidden? history from
   books start or only the last N financial years?
6. **Who may manage portal access:** owner and admin only (proposed), or every member?
7. **Email sender:** which provider and sending domain (SES Mumbai vs ZeptoMail vs existing
   mail)? Who owns the DNS records?
8. **Payments:** tenant-pasted keys, or a Razorpay partner agreement for one-click Connect? Do we
   also want Cashfree or PayU?
9. **Invoice bank details and terms** are device-local today (`bankStore.ts`, `termsStore.ts`).
   May they become a synced company setting? (That also helps the consultants'
   "customize the invoice" ask.)
10. **Balance confirmation:** do the consultants' audit clients want it? It changes phase-2
    priority.
11. **Counsel review:** the processor clause in the ToS; the portal notice text; whether consent
    or §7(a) legitimate use is the basis; the audit-log retention period.
12. **Pilot:** which 3–5 real customers of the consultants' clients will try it, and on which
    real (synced) company?

---

## 12. Sources

Zoho: [customer portal help (IN)](https://www.zoho.com/in/books/help/client-portal/) ·
[customer portal product page](https://www.zoho.com/in/books/customer-portal/) ·
[vendor portal help (IN)](https://www.zoho.com/in/books/help/vendor-portal/) ·
[pricing (IN)](https://www.zoho.com/in/books/pricing/).
Tally: [connected services](https://tallysolutions.com/features/connected-services/) ·
[WhatsApp web story](https://tallysolutions.com/web-stories/tallyprime-with-whatsapp/) ·
[payment request help](https://help.tallysolutions.com/tally-prime/banking-utilities/payment-request/) ·
[Razorpay–TallyPrime announcement](https://razorpay.com/blog/announcing-razorpay-tallyprime-integration/).
Others: [Busy app](https://apps.apple.com/app/id1662346018) ·
[Marg eRetail](https://apps.apple.com/cd/app/eretail/id1217414784) ·
[Logic ERP B2B](https://logicerp.com/Modules/B2B-Order-Management-Software) ·
[Vyapar features](https://vyaparapp.in/blog/vyapar-app-features/) ·
[Intuit feedback: customer portal](https://feedback.qbo.intuit.com/forums/591862-quickbooks-invoice-payments-portal/suggestions/40247722-customer-portal) ·
[Xero ideas: live statements](https://productideas.xero.com/forums/967115-invoices-quotes/suggestions/50499696-customer-statements-allow-customers-to-view-live) ·
[CredFlow (Techjockey)](https://www.techjockey.com/detail/credflow).
Payments: [Razorpay Payment Links webhooks](https://razorpay.com/docs/webhooks/payment-links.md) ·
[partner-auth Payment Links](https://razorpay.com/docs/partners/aggregators/partner-auth/payment-links.md) ·
[OAuth integration](https://razorpay.com/docs/partners/platform/onboard-businesses/integrate-oauth/integration-steps) ·
[Razorpay pricing explainer](https://razorpay.com/blog/razorpay-payment-gateway-pricing-explained/) ·
[RBI PA Directions 2025 — AZB](https://www.azbpartners.com/bank/rbi-issues-consolidated-reserve-bank-of-india-regulation-of-payment-aggregators-directions-2025/) ·
[UPI linking spec (mirror)](https://www.labnol.org/files/linking.pdf).
Privacy and messaging: [DPDP Rules — Hogan Lovells](https://www.hlc.com/en/publications/indias-digital-personal-data-protection-act-2023-brought-into-force-) ·
[DPDP — AZB](https://www.azbpartners.com/bank/update-indias-digital-personal-data-protection-framework-comes-into-effect/) ·
[DLT process (Plivo)](https://www.plivo.com/docs/sms/concepts/dlt-registration-process) ·
[SMS OTP pricing (Message Central)](https://www.messagecentral.com/blog/sms-otp-pricing-india) ·
[WhatsApp pricing (Meta)](https://developers.facebook.com/docs/whatsapp/pricing) ·
[ZeptoMail pricing](https://www.zoho.com/zeptomail/pricing.html) ·
[NIST SP 800-63B-4](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-63B-4.pdf).
