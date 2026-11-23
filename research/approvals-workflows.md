# Voucher approvals and workflows: research and design

> Status: research and design only, nothing built. Written 2026-10-15 for the owner and the
> consultants' team. The ask came from the consultants on 2026-10-15.
> Classification: a **Beyond Tally** capability. Track it under `beyond` in
> `docs/parity-data.json`, never in the parity percentage (CLAUDE.md, "Phases 18-21").

**The ask, verbatim:** "There should be an option for creation of workflows and voucher
approvals, where admin should be able to define approvals. From their mobile or their web app
they should be able to approve or reject transactions, which is currently not there in Tally
also."

---

## 0. TL;DR

1. **Build it, as maker-checker on the voucher itself.** A voucher that matches an admin-defined
   rule saves as **Pending approval**. It carries the existing `optional` bit while pending, so
   the ~100 guards that already keep optional vouchers out of ledgers, stock, GST, TDS, payroll
   and bill-wise keep it out too. It posts only when the last approver approves. Rules are a
   synced master, and once a company syncs to the cloud the **server is the only writer of approval
   state**.
2. **Mobile means a PWA inbox on the existing web app, not a native app.** Approvals arrive by
   in-app SSE, web push and email. WhatsApp is an optional paid add-on later.
3. **You can build a working MVP with no external credentials.** Local maker-checker with PIN
   operators, plus a cloud in-app inbox on any synced device, needs no partner contract. Phone
   push needs only HTTPS, which is infrastructure, not a contract. WhatsApp needs a Meta/BSP
   account.
4. **The biggest blocker is not approvals.** The cloud today has **one login per tenant** (there
   is no invite or member flow) and runs **HTTP-only**. An approval by a *different person* on a
   *phone* needs both of those fixed first.
5. **Effort:** about **12-14 engineer-weeks raw** to reach desktop and phone approvals with web push and
   email. About 9.5 of those weeks are approval-specific; the other 3-4 are prerequisites that overlap the
   Phase-B backlog (TLS, idempotency, team invites). WhatsApp adds about 1-1.5 weeks plus Meta's
   lead time.

---

## 1. What the ask really needs

### 1.1 Interpretation

"Workflows" here means **approval routing**: who must approve which transactions, in what order,
and what happens meanwhile. It does **not** mean a general BPM or automation builder. Generic
"when X happens, do Y" automation is already possible through the shipped webhooks and REST API
(`backend/src/platform.js`, `backend/src/rest.js`). A no-code automation designer is a separate,
larger product and is not recommended now (section 5, option H).

### 1.2 User stories

| # | As a... | I want to... | Notes |
|---|---|---|---|
| U1 | Admin (owner, CA) | define rules: "Payments of ₹1,00,000 or more need the Accounts Head, then the Owner" | By voucher type, amount, party or party group, ledger, creator role or user, back-dated entry, source (desk, bulk, import, API). Multi-level, with any-of, all-of or N-of at each level. |
| U2 | Data-entry operator (maker) | enter vouchers exactly as today; if a rule applies I see **before I save** that it will go for approval, and to whom | **Keyboard-first:** no new key. Ctrl+A still accepts. The hold is shown on screen (CLAUDE.md gotcha: "A carried/held value must always be visible"). |
| U3 | Maker | see what I submitted, what was sent back to me and why, then fix and resubmit | "Submitted by me" tab. A sent-back voucher reopens in normal alteration. |
| U4 | Approver at a desk | clear my queue without a mouse | G then I opens the inbox. ↑↓ moves, Enter reviews, Ctrl+A approves, Alt+R rejects with a reason, Alt+B sends back, Space plus Ctrl+A bulk-approves. These keys are proposals and must be checked against `src/keys.ts` and the `?` sheet (section 6.4). |
| U5 | Owner on a phone | get a push or email ("Purchase ₹2.4L · Sharma Traders · by Ravi"), tap it, see the voucher, and approve or reject in 2 taps | PWA on the existing web app. Step-up (PIN or password again) above a threshold. Works on Android, and on iOS 16.4+ once installed to the Home Screen. |
| U6 | Owner | bulk-approve the 20 small items at day end, but never bulk-approve the big ones | Bulk is allowed only below the step-up threshold. |
| U7 | Admin | override a stuck chain ("Final approve"), with a mandatory reason that is logged | Zoho has the same idea ("Final Approve"). |
| U8 | Auditor | see who entered, who approved at each level, when, from which channel, and **exactly what was approved** | Hash of the approved content (section 6.8). |
| U9 | Integrator (REST / API key) | have API-created vouchers obey the same rules | QuickBooks routes third-party and imported transactions straight to Pending approval ([Intuit](https://quickbooks.intuit.com/learn-support/en-uk/help-article/manage-workflows/create-custom-workflows-quickbooks-online-advanced/L0SVebThg_GB_en_GB)). Ours must too: `POST /v1/vouchers` currently bypasses everything (section 2). |
| U10 | Offline operator | keep working offline; my vouchers wait as pending and go to approvers when I reconnect | Approvers cannot be notified until the device syncs. Say so in the UI. |

### 1.3 Non-negotiables

- **A pending voucher never touches the books.** That means ledgers, Trial Balance, P&L and BS,
  stock and valuation, bill-wise outstanding, GST returns (GSTR-1, 3B, 9, HSN), TDS/TCS,
  payroll, e-invoice IRN, e-way bill, and Tally export. The ask says this, and the market splits
  on it (BUSY even has a "Post Unapproved Vch" switch: [busy.in](https://busy.in/accounting-software/user-approval/)).
- **No silent loss** when approval and offline edits race (ARCHITECTURE.md §6.6 principle).
- **GST serial integrity** (Rule 46) survives rejection, resubmission and offline numbering.
- **Approval is a server fact once cloud-connected.** It is not something a client can assert by
  pushing a payload.

---

## 2. What exists today (codebase facts this design builds on)

| Area | Fact | File |
|---|---|---|
| Exclusion from books | `postsToLedgers(v) = baseInfo(v.base).accounting && !v.optional` is the canonical ledger guard. Engines also check `!v.optional` directly. | `src/core/accounting.ts:8`; `src/core/inventory.ts:35,297,305`; `src/core/billwise.ts:46`; `src/core/tds.ts:255,422`; `src/core/tcs.ts:36`; `src/core/payroll.ts:17`; `src/core/gstr1.ts:360,442` |
| Breadth of that guard | 101 references to `.optional` / `postsToLedgers` across 33 files, covered by the assertion suites. A missed guard was "the biggest bug" historically (HANDOFF §4). | `grep` over `src/` |
| Optional elsewhere | IRN is refused for optional vouchers. Tally export skips them. Print adds "Optional voucher, not part of the books". Docs Issued excludes them and counts **gaps as cancelled**. | `src/ui/gst/einvoice.tsx:175-178`, `src/core/tallyExport.ts:485`, `src/ui/print/invoiceModel.ts:368`, `src/ui/gst/docsissued.tsx:119` |
| Optional toggles | Ctrl+L / Alt+L toggles optional on a voucher. Mark Optional/Regular works as a batch, with `markBlock` rules. | `src/ui/VoucherScreen.tsx:1014,2451,2909`; `src/ui/reports/markOptional.ts`; `src/ui/selection.ts:165` |
| Operator model | Opt-in PIN operators (User/Role per company). Rights are by area, voucher bases, actions and a date window, plus a period lock. The code says it is **"a client-side control for people sharing a screen, not a data-security boundary"**. | `src/core/access.ts:12-14,180,266`; `src/ui/admin/Security.tsx` |
| Write enforcement | `checkWrite` runs in `putDoc`/`deleteDoc` **before** the transaction. `applyRemote` (pulled rows) bypasses it by design. | `src/data/writeGuard.ts`; `src/data/db.ts:296-334` |
| Audit fields | Outbox rows carry `user` (sync email) and `operator` (local label), and `origin:'flag'` keeps batch flag writes off the undo stack. **The push does not send `operator`** to the server. | `src/data/db.ts:31-49`; `src/sync/sync.ts:509-511` |
| Numbering | Per-device `max+1` over local vouchers, **including optional ones**. Restart is yearly or monthly, with a prefix/suffix token formatter. | `src/state/store.ts:541-573` |
| Server numbering | `POST /sync/next-number` exists but **the client never calls it**. The counter key is `(tenant_id, type_id)`, with **no FY and no series** (ARCHITECTURE §6.4 wanted `(tenant,cid,type,series,fy)`). | `backend/src/sync.js:48`; `backend/sql/005_p3.sql` |
| Sync wire | v1: full-doc LWW by server `seq`, **no `changeUid` idempotency, no per-row authorization, no `rejected[]`**. Every member of the tenant can push anything. | `backend/src/sync.js:8-45` |
| Realtime | The server has a per-tenant in-memory SSE "nudge". **The client does not subscribe** (no `EventSource` in `src/`) and polls every 30 s. | `backend/src/realtime.js`; `src/sync/sync.ts:593` |
| Team | `membership.role` is owner, admin or member, but the **only** membership ever created is the owner's at register. **There is no invite or member route.** | `backend/sql/002_p2.sql`; `backend/src/tenancy.js:10`; route list |
| REST | `POST /v1/:collection` appends straight to `change_log` and fires webhooks. Sync pushes fire **no** webhooks. | `backend/src/rest.js:40-57` |
| Transport | **HTTP only, no TLS.** Service workers and web push require a secure context. | HANDOFF §2 trust map |
| Shared pure module precedent | `backend/src/passwordPolicy.js` is ONE pure module used by both server and app (re-exported by `src/core/passwordPolicy.ts`). | `backend/src/passwordPolicy.js:1-10` |
| Prior approval precedent | Leave applications have `Pending/Approved/Rejected`, but status is just an editable field: no approver identity, no rule. | `src/types.ts:381-395`; `src/ui/hr/leaveapply.tsx` |
| Architecture stance | "Approval gating: honestly a soft control for the client engines". The `approvals` table is sketched. | `docs/ARCHITECTURE.md:830-838,1581-1592` |
| Roadmap | "maker-checker approval workflow with audit ties" is listed under the Team collaboration epic. | `docs/ROADMAP.md:29` |

---

## 3. Landscape

### 3.1 What competitors ship

| Product | Approvals? | Rules / levels | Pending in books? | Mobile / notify | Source and confidence |
|---|---|---|---|---|---|
| **Zoho Books** | Yes, per module: Quotes, SO, Invoices, Credit Notes, PO, Bills, Payments Made, Vendor Credits, Inventory Adjustments, Custom Modules | Simple (any approver), **Multi-level up to 10**, Custom (criteria trigger specific approvers, priority-ordered). Admin **Final Approve** skips levels. One rejection rejects the whole chain. Reject needs a reason. Resubmit is allowed. Self-approval is an optional preference. Bulk approve exists. | Transactions need approval before "further actions". Their KB says Draft/Approved items are not in reports; only Sent/Open are. | In-app + email documented. **Mobile approval not documented**: the app listing says "verify and approve" but this is unverified. | [configure](https://www.zoho.com/books/help/transaction-approval/configure-approvals.html), [multi-level](https://zstatic.zohostatic.com/books/kb/transaction-approval/configure-multi-level-approval.md), [custom](https://www.zoho.com/books/help/transaction-approval/custom-approval.html), [flow](https://www.zoho.com/en-de/books/help/transaction-approval-new-flow/transaction-approval-workflow.html), [reports KB](https://www.zoho.com/us/books/kb/banking/approved-transactions-in-reports.html). Official docs, high confidence. |
| **Zoho Books price (India)** | Professional ₹1,499/mo or Premium ₹2,999/mo (+18% GST) | Sources **disagree** on which tier includes approvals. | | | [Patron Accounting](https://www.patronaccounting.com/blog/zoho-books-pricing-india-2026), [reseller](https://codroiditlabs.com/zoho-books-reseller-india-2/). Third-party, unverified on zoho.com/in. |
| **QuickBooks Online Advanced** (not sold in India) | Yes: workflow templates for bills, invoices, estimates, POs | Conditions, one or more approvers with ordering, approver groups (up to 5 per group), minimum approvals. Imported or third-party transactions go straight to Pending approval. Bills unreviewed for 30 days are **auto-denied**. | Held until approved ("only sends it once approved"). Ledger effect is not stated. | Task + email + push options. | [Intuit custom workflows](https://quickbooks.intuit.com/learn-support/en-uk/help-article/manage-workflows/create-custom-workflows-quickbooks-online-advanced/L0SVebThg_GB_en_GB), [bill approval](https://quickbooks.intuit.com/learn-support/en-us/help-article/manage-workflows/set-use-bill-approval-and-payment-release-workflows/L1IOLL9hv_US_en_US). Official. |
| **Xero** | Bills/invoices: Draft → **Awaiting Approval** → Awaiting Payment | **Single step**, role-gated. No native multi-level or amount rules; customers buy ApprovalMax. | An awaiting-approval bill cannot be paid. | | [third-party guide](https://invoicedataextraction.com/blog/xero-bill-approval-workflow), [Xero idea forum: multi-level request](https://productideas.xero.com/forums/939198-for-small-businesses/suggestions/44960764-bills-ability-to-allow-multiple-levels-of-approv). Not verified against Xero's own help. |
| **SAP Business One** | Yes: approval **stages** (N approvals / N rejections required) + **templates** (originators, document types, terms such as amount over X) | Multi-stage. A rejected draft can be edited and re-run. On approval the originator **converts the draft to a real document**. | **Pending = draft document**, outside the books. | Internal messages + email. Reports: Approval Status, Approval Decision. | [SAP help 9.0](https://help.sap.com/doc/saphelp_sbo900/9.0/en-US/45/06dab4d8696c30e10000000a114a6b/content.htm), [SAP Learning](https://learning.sap.com/courses/implementing-sap-business-one/implementing-the-approval-processes-in-sap-business-one). Official. |
| **BUSY** (closest Indian desktop rival) | Yes, **Enterprise feature**: vouchers (by type and series) **and masters** (account, item) | Approver sub-users, Super User always included. "Selective approval" allows **up to two conditions**. FAQ claims multi-level / value matrix. Approve with remarks, bulk approve, Approval Report. | **Configurable:** "Post Unapproved Vch" and "Allow Printing" switches. | Desktop only per the page. Notifications not mentioned. | [Approval Management](https://busy.in/accounting-software/user-approval/), [multilevel FAQ](https://busy.in/faqs/how-to-handle-multilevel-and-matrix-approvals-in-busy-answerid-63057/), [voucher approval FAQ](https://busy.in/faqs/what-is-the-process-for-voucher-approval-answerid-67744). Vendor pages, medium confidence. |
| **RealBooks** (Indian) | Maker-checker pools by user, branch, voucher type and ledger group, with attachment review | | Unapproved entries **cannot be printed**. Email + dashboard notification. | | [realbooks.in](https://www.realbooks.in/?p=5019). Vendor blog. |
| **Marg ERP / Vyapar** | **Could not verify** either way. No documentation found. | | | | none |
| **TallyPrime** | **No native maker-checker found** in official help, up to and including Release 7.0 (Dec 2025: Auto Backup, PrimeBanking, SmartFind, GST updates) | Security levels (Owner / Data Entry; allow/disallow create, alter, display, print by voucher type), back-date cut-off, **Edit Log**. Tally.NET remote users are data-entry-only by default. Partners sell **TDL add-ons** for voucher approval. | n/a | n/a | [7.0 release notes](https://help.tallysolutions.com/release-notes-tallyprime-7-0), [security levels](https://help.tallysolutions.com/article/te9rel61/Data_Management/DM_Assign_Security_Level.htm), [Tally guide on audit trails](https://tallysolutions.com/business-guides/tallyprime-audit-trails-and-controls-to-satisfy-external-auditors/), [TDL listing](https://vikashamittech.odoo.com/shop/tally-tdl-vouchers-verify-module-l-vouchers-approval-with-password-protection-95). One partner article describes a "Use Voucher Approval" voucher-type option and a "Voucher Approvals" report ([tallyatcloud](https://tallyatcloud.com/articles/remote-access-with-approvals-in-tally-prime-gold)). It cites no Tally documentation and **could not be verified**; treat it as unconfirmed. |

### 3.2 What Tally lacks, and where we can be better than everyone

- **Tally:** no native rule-based routing, no mobile approval, no approver identity separate from
  a shared desktop login, no "what exactly was approved" proof. The consultants' claim holds
  for Tally. It does **not** hold for BUSY Enterprise, which is the competitor to beat in the
  reseller channel.
- **Our edge, if we build it as designed:**
  1. **keyboard-first approval at the desk** (nobody else does this);
  2. **phone approval** with push (BUSY and Tally have none; Zoho's mobile approval is undocumented);
  3. **offline makers**, which Zoho, QBO and Xero cannot have;
  4. **honest GST numbering** through rejection (section 4.2);
  5. **tamper-evident approval proof** (content hash per decision).

### 3.3 Regulatory frameworks that constrain the design

| Framework | What it requires | Design consequence |
|---|---|---|
| **CGST Rule 46(b)**: tax invoice serial | "consecutive serial number not exceeding sixteen characters", one or multiple series, **unique for a financial year** ([CBIC](https://taxinformation.cbic.gov.in/content/html/tax_repository/gst/rules/cgst_rules/active/chapter6/rule46_v1.00.html)) | A rejected invoice must not leave an unexplained hole. Two allocators (desk at entry, server at approval) must never share a series (section 4.2). |
| **GSTR-1 Table 13**: documents issued | Total issued + **cancelled** per series | Rejected invoice → shown as cancelled with its number kept. **Pending** invoices must not be counted as cancelled gaps, which today's `docsissued.tsx` gap logic would do. |
| **CGST s.31**: time of issue (general knowledge, not re-verified here) | Goods: on or before removal. Services: within a prescribed period of supply. | Holding a **sales** invoice for approval can make it legally late. Recommend gating Sales Orders, discounts or credit-limit breaches rather than every invoice, plus an "approval lag" warning. |
| **e-Invoice 30-day IRP window** | From 1 Apr 2025, taxpayers with AATO of ₹10 cr or more cannot report an e-invoice older than 30 days to the IRP ([Taxmann on GSTN advisory](https://www.taxmann.com/post/blog/time-limit-of-30-days-for-reporting-e-invoice-on-irp-portal-for-taxpayers-with-aato-of-10-crores-and-above-gstn-update/)) | Escalate pending outward documents at day 20+. Block IRN until approved (already blocked for optional). |
| **e-Way bill** (Rule 138, general knowledge) | Generated before movement | A pending invoice cannot carry an e-way bill. Gate at the order or dispatch stage instead. |
| **Companies (Accounts) Rules, Rule 3(1) proviso**: audit trail | Software must keep an edit log of every change with its date; the log **cannot be disabled**. In force from 1 Apr 2023 ([ClearTax](https://news.cleartax.in/mandatory-to-use-accounting-software-with-audit-trail-mca/6016/amp), [Mondaq](https://www.mondaq.com:443/india/accounting-standards/1071750/audit-trails-under-the-companies-act-a-hazy-path-ahead)) | Every approval transition is an edit-log event. Approval actions are append-only, with no UPDATE or DELETE grant. Books are retained 8 years under Companies Act s.128(5) (general knowledge). |
| **DPDP Act 2023 + DPDP Rules 2025** | Rules notified 13/14 Nov 2025. Most obligations (notice, security safeguards, breach notification) apply from about **May 2027**; consent managers from about Nov 2026 ([The Week/PTI](https://www.theweek.in/wire-updates/business/2025/11/14/del148-biz-dpdp-rules-ld-govt.html), [TCSA roadmap](https://www.tcsa.in/resources/dpdp-rules-2025-implementation-roadmap)) | Approver phone numbers and emails are personal data. So are party names of sole proprietors. Minimise notification payloads, disclose sub-processors (Meta, BSP, email provider), and get WhatsApp opt-in. Section 8 (general knowledge): erasure yields to a **legal retention** duty, so accounting approval records survive a user-erasure request. |
| **RBI** | Not directly applicable: our approval moves no money. | If connected-banking payment initiation is built later, **the bank's own authorisation and 2FA still applies**. Our approval is an internal control, never a payment authorisation. Do not market it as one. |
| **TRAI DLT** (SMS, general knowledge) | Commercial SMS in India needs DLT entity + template registration | SMS not recommended; push, email and WhatsApp cover the need. |

### 3.4 Notification channels and costs

| Channel | Needs | Cost | Caveats |
|---|---|---|---|
| **In-app (SSE)** | Already on the server (`backend/src/realtime.js`). The client must subscribe. | ₹0 | In-memory hub (single VM). Only reaches open tabs. |
| **Web push (VAPID)** | **HTTPS**, a service worker, a manifest, and the `web-push` npm lib ([GitHub](https://github.com/web-push-libs/web-push)) | ₹0. The browser push services are free (general knowledge). | **iOS: 16.4+ and only after "Add to Home Screen"** ([OneSignal](https://documentation.onesignal.com/docs/web-push-for-ios)). Reports say EU standalone PWAs were removed in iOS 17.4 ([MagicBell](https://www.magicbell.com/blog/pwa-ios-limitations-safari-support-complete-guide); irrelevant for India). Some iOS reliability complaints. |
| **Email** | SMTP or API provider. SPF/DKIM/DMARC on the sending domain (needs DNS access). | Amazon SES: $0.10 per 1,000 pay-as-you-go. AWS now also lists plan tiers ($0.16-0.23/1,000) ([AWS](https://aws.amazon.com/ses/pricing/)). Effectively ₹0 at SMB volume. | **Links must never approve on GET**: corporate mail scanners pre-click links. |
| **WhatsApp** (Meta Cloud API direct or via a BSP) | Meta Business verification, a WABA, display-name approval, **utility template approval**, recipient **opt-in**, and an **HTTPS webhook** | Per-message since 1 Jul 2025 ([Meta](https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing)). India utility ≈ **₹0.145/message** as quoted by a BSP, chargeable even in-window from 1 Oct 2026 per that BSP ([AiSensy](https://m.aisensy.com/blog/whatsapp-pricing-update-october-2026/)). **Not verified against Meta's INR rate card.** Meta's own page still says in-window utility is free. Plus BSP platform fees (vary, not researched). India WABAs must move to INR billing by 31 Dec 2026. | At 50 approvals/day ≈ 1,500 msgs/month ≈ ₹220/month, plus BSP fee. Quick-reply "Approve" buttons are possible, but recommend "Review" deep-links only (section 7). |
| **SMS** | DLT registration | ~paise per SMS (not researched) | Not recommended. |
| **Native app push (FCM/APNs)** | Store accounts: Apple $99/yr and Google $25 one-time (general knowledge), plus an RN app | Low | Only with a native app (ROADMAP mobile epic). Deferred. |

---

## 4. The design decisions that matter

### 4.1 How a pending voucher is represented: reuse `optional`, or a new state?

| Option | How | Pro | Con |
|---|---|---|---|
| **A. Reuse `optional` alone** | Pending = optional with a note | Zero new guards | Conflates "memo voucher" with "awaiting approval". Ctrl+L or Mark Regular would **approve by side door**. No routing state. |
| **B. New `approval.state` only** | Every engine checks `isInBooks(v)` | Clean semantics | Touches 33 files / ~100 sites. One miss = a pending voucher in GSTR-1. That is exactly the historic "optional-voucher leak" class of bug. |
| **C. Hybrid (recommended)** | `voucher.approval` carries the workflow state, **and** a held voucher **also** has `optional: true`. Invariant: `approval.mode==='hold' && approval.state!=='approved' ⇒ optional===true`, enforced by the client write guard **and** the server. The maker's own Ctrl+L choice is kept in `approval.userOptional` and restored on approval. | Every existing exclusion works on day 1 with its assertions. The new field adds only the routing, display and authority. | Two fields must stay coupled, so the invariant needs an assertion suite. The Optional Vouchers register must split "Optional" from "Pending approval". |

**Pick C.** `optional` becomes the mechanism ("not in the books") and `approval` becomes the
reason. A later refactor to a single `isInBooks(v)` predicate can happen behind the invariant
without risk.

### 4.2 Numbering, and GST Rule 46

| Policy | Mechanism | Gap-free? | Offline? | Risk |
|---|---|---|---|---|
| **N1. Number at entry; reject = cancel (default)** | Unchanged `nextVchNo` at save. A rejected voucher is **kept** with its number and state `rejected` (still optional), so Docs Issued shows it as cancelled. This is the Tally-style rule ARCHITECTURE §6.4 already chose. | Yes | Yes | Numbers are not in approval order. A pending invoice holds a number that may later read as cancelled. |
| **N2. Draft ref at entry; final number at approval (per voucher type, opt-in)** | The maker gets `DRAFT/n` (non-statutory). The **server allocates** the final number inside the approve transaction (`voucher_counter`, re-keyed to `(tenant,cid,type,series,fy)`). | Yes, and no cancelled noise | Approval is online anyway | **Only safe when *every* voucher of that type goes through approval** (rule with no threshold). Otherwise desk (`max+1`) and server allocate in one series and **collide**. Requires moving `formatVchNo` into a shared pure module. |
| N3. Number at entry, renumber at approval | | No | | Breaks references, prints and bill-wise. **Rejected.** |

**Pick N1 as default and N2 as an opt-in per voucher type.** The settings form must
refuse N2 unless the type's rule has no threshold and the company syncs to the cloud. Also:

- Docs Issued (`src/ui/gst/docsissued.tsx`) counts `approval.state==='rejected'` as cancelled and
  `pending` as a **separate "pending, not issued" column**, with a warning if any are pending in the
  filing period.
- A pending outward voucher **prints with a "PENDING APPROVAL, NOT A TAX INVOICE" watermark** (it
  extends the existing optional note in `invoiceModel.ts:368`). The admin can disable printing entirely,
  which is BUSY's "Allow Printing" switch and RealBooks' default. IRN and e-way generation stay blocked.

### 4.3 Hold vs post-and-review (per rule)

- **`hold` (default):** pending does not post. This is what the ask demands.
- **`review`:** the voucher posts immediately and approval is a sign-off checkmark (BUSY's "Post
  Unapproved Vch"). Recommend this for **Receipts and Contra**: the money has already moved, and
  holding it out of the books makes cash and BRS wrong. Pending-review items still show in the
  inbox and the audit trail.

### 4.4 Identity: who is the maker, who is the approver?

Reality in Indian SMB offices: several PCs share **one** cloud sync login (often the owner's),
with PIN operators on top (`access.ts`). So:

- **Maker identity = the company operator (`User`, PIN-signed-in).** Approvals require
  `company.security.enabled` (the settings form refuses otherwise). Push must start sending the
  outbox `operator` user **id**, not just its label (`sync.ts:509-511` drops it today).
- **Approver identity = a personal cloud account** (`app_user`) **bound** to a company `User`
  (`company_user_link`). This is the only identity the server can authenticate. It needs **team
  invites**, which do not exist yet (section 2).
- **Self-approval is compared on company `User` ids** (maker operator vs the approver's bound User),
  not on cloud accounts. Otherwise an owner whose login is also the office login could never
  approve anything.
- **Honest limit:** maker identity is only as strong as the PIN model. Approver identity is as strong as
  cloud auth (Argon2id + JWT; MFA is designed but not built, ARCHITECTURE §7).

### 4.5 Authority: who may write approval state?

- **Local-only company (never synced):** the client is the authority. The approver signs in as an
  operator on the same machine. This is a soft control, said plainly on screen, just as `access.ts` does.
- **Cloud-connected company:** **the server owns `voucher.approval` and, for held vouchers,
  `voucher.optional`** (field ownership).
  - On every voucher push, the server overwrites those fields from its own records.
  - If that changes the payload, it stores the corrected row with `device_id='server'` so the
    origin device also pulls the correction (normal echo suppression would otherwise hide it).
  - A crafted client can still compute its *own* local Trial Balance with the voucher posted. That
    is the residual soft spot ARCHITECTURE §6.7 already admits. It cannot make other devices,
    the REST API, webhooks or exports see an unapproved voucher as approved.
- **Optional rule flag `allowDeskApproval`** (default: on below the step-up threshold, off above it):
  this allows an offline PIN-operator approval at a desk to be accepted by the server on push, recorded as
  `channel='desk-pin'`.

### 4.6 Rule matching

- **First match by priority wins** (Zoho's model). It is explicit, and the screen can show which
  rule matched.
- **The rule is snapshotted onto the request at submission** (SAP B1 templates behave the same). Editing a
  rule never re-routes in-flight items. A resubmission re-evaluates.
- **The amount is `voucherHeadlineAmount(v)`** (`src/core/accounting.ts:117`). That is the party
  leg, not the debit sum, per the Phase 23 gotcha, so a round-down invoice cannot dodge a threshold.

---

## 5. Options compared

| Option | Scope | Effort | Running cost | External dependency | Risk |
|---|---|---|---|---|---|
| **A. Local maker-checker** | Rules master, pending = held/optional, desk inbox, PIN operators, audit in outbox | ~4 wk | ₹0 | None | Soft control. Same device only. No phone. |
| **B. A + server-authoritative cloud approvals** | approval_request/action tables + RLS, field ownership on push and REST, inbox API, SSE, webhooks | +3 wk, **plus ~3-4 wk prerequisites** (invites, user links, idempotency, `rejected[]`, client SSE) | ₹0 extra | None (own backend) | Medium: sync semantics, solved in section 6.6 |
| **C. B + PWA phone inbox + web push + email** | Manifest, service worker, VAPID, mobile layout, step-up, SES | +2.5 wk | Email ≈ ₹0 at SMB volume | **HTTPS (hard prerequisite)**, DNS for SPF/DKIM, SES account | iOS needs a Home Screen install. Push reliability. |
| **D. C + WhatsApp** | Utility templates with deep-links, opt-in, webhook | +1-1.5 wk after onboarding | ≈ ₹0.145/msg (BSP-quoted, unverified) + BSP fee | **Meta WABA + business verification + template approval** (partner/credentials) | Template rejection, policy changes 4x/year, phone-number identity |
| E. Native mobile app (RN) | Store apps, FCM/APNs | XL (8+ wk) | Store fees | Apple/Google accounts | Duplicates PWA. Belongs to the ROADMAP mobile epic. **Defer.** |
| F. "Review only" (post-then-sign-off everywhere) | Checkmark, no hold | ~1.5 wk | ₹0 | None | **Does not meet the ask** ("must not post"). Keep it only as a per-rule mode. |
| G. Partner TDL-style add-on approach | n/a | n/a | n/a | n/a | Not applicable: we own the product. |
| H. General no-code workflow / automation builder | Triggers, conditions, actions | XL | ₹0 | None | Scope creep. Webhooks + REST already cover integrations. **Not now.** |

**Recommendation: A → B → C as one epic, with D only when a paying customer asks.** A ships value on
its own: desktop maker-checker that Tally lacks. Its data model is exactly what B and C extend.

---

## 6. Recommended architecture

### 6.1 Client data model (`src/types.ts`, Dexie v7 → **v8**)

```ts
export type ApprovalState = 'pending' | 'approved' | 'rejected' | 'sentBack' | 'withdrawn';
export type ApprovalAction = 'submit' | 'approve' | 'reject' | 'sendBack' | 'withdraw'
  | 'finalApprove' | 'reopen' | 'escalate' | 'expire';
export type ApprovalChannel = 'desk' | 'desk-pin' | 'pwa' | 'email' | 'whatsapp' | 'api' | 'system';

export interface ApprovalLevel {
  userIds?: string[];          // company User ids
  roleIds?: string[];          // company Role ids
  need: 'any' | 'all' | number; // any-of / all-of / N-of
}

export interface ApprovalRule {            // new synced master table `approvalRules`
  id: string; cid: string; name: string;
  active: boolean;
  priority: number;                        // first match wins
  voucherTypeIds?: string[];               // absent = all types of `bases`
  bases?: VchBase[];
  minAmount?: number; maxAmount?: number;  // on voucherHeadlineAmount()
  partyIds?: string[]; partyGroupIds?: string[]; ledgerIds?: string[];
  creatorRoleIds?: string[]; creatorUserIds?: string[];
  backdatedDaysOver?: number;              // entry date vs voucher date
  sources?: ('entry' | 'bulk' | 'import' | 'api')[];
  mode: 'hold' | 'review';
  levels: ApprovalLevel[];                 // 1..10 (Zoho's cap is a sane bound)
  allowSelfApproval?: boolean;             // default false
  allowDeskApproval?: boolean;             // accept offline PIN approvals (section 4.5)
  stepUpAbove?: number;                    // re-auth + no bulk above this amount
  onEditAfterApproval: 'reapprove' | 'lock';
  numbering?: 'atEntry' | 'atApproval';    // N1 / N2 (section 4.2)
  escalateAfterHours?: number;
}

export interface ApprovalStep {
  at: string;              // server time when cloud-decided
  by: string; byName: string;   // company User id + display name
  action: ApprovalAction; level: number; note?: string; channel: ApprovalChannel;
  vchHash: string;         // content hash the actor saw
}

export interface VoucherApproval {
  state: ApprovalState;
  ruleId: string; ruleName: string; mode: 'hold' | 'review';
  level: number; levels: number; round: number;
  requestId?: string;      // server id once synced
  submittedBy: string; submittedAt: string;
  userOptional?: boolean;  // the maker's own Ctrl+L choice, restored on approval
  trail: ApprovalStep[];   // append-only; written by the authority (client if local-only, server if cloud)
}
// Voucher gains:  approval?: VoucherApproval;
// Company gains:  approvals?: { enabled: boolean };   // requires security.enabled
```

**One pure shared module** follows the `passwordPolicy.js` precedent.
`backend/src/approvalPolicy.js` is re-exported by `src/core/approvalPolicy.ts` and owns:

- `matchRule(voucher, rules, ctx)`;
- `materialHash(voucher)`: canonical JSON of date, typeId, number, partyId, entries, inventory,
  placeOfSupply, rcm and the GST fields;
- `nextState(request, action, actor)`;
- `canAct(actor, request, rule)`, including self-approval and any/all/N-of;
- `invariantOk(voucher)`.

Client and server can then never disagree on routing. A new standalone suite
`scripts/assert/approvals.assert.ts` covers it, and the existing optional-exclusion suites must
stay green.

### 6.2 State machine

```
            (rule matches on save)                     approve (last level)
  [new] ──────────────────────────► PENDING(L1) ──► … ──► PENDING(Ln) ─────────────► APPROVED
                                       │   ▲                 │                        │
                         send back     │   │ resubmit        │ reject                 │ edit by maker
                                       ▼   │ (round+1, L1)   ▼                        │ (onEditAfterApproval
                                    SENT BACK            REJECTED  (kept, number      │  = 'reapprove')
                                       │                  retained = cancelled)       ▼
                                       │ delete/withdraw                         PENDING(L1), round+1
                                       ▼
                                   WITHDRAWN
  Any pending level: admin "Final approve" (reason required) ──► APPROVED
  Edit while pending: approvals so far voided ──► PENDING(L1), same round, trail 'reopen'
```

| Transition | Who | Effect on voucher |
|---|---|---|
| submit | maker (any operator who may create that type) | `optional=true` (hold); number per N1/N2 |
| approve (non-final level) | an eligible approver at level k, not the maker | level k+1. Still held. |
| approve (final) / final approve | final-level approver / full-rights admin | `optional = approval.userOptional ?? false`; N2 assigns the number |
| reject | eligible approver (reason required) | stays held, state `rejected`; number retained (N1) |
| send back | eligible approver (note required) | state `sentBack`; the maker may edit and resubmit |
| withdraw | maker, or by deletion | state `withdrawn` (N1: delete leaves a gap, which Docs Issued already reads as cancelled) |
| reopen | the system, on a material edit | as per the diagram |

### 6.3 Behaviour of a pending (held) voucher in the books

| Surface | Behaviour | Mechanism |
|---|---|---|
| Ledgers, TB, P&L, BS, cash/bank, ratios | Excluded | `postsToLedgers` (`optional`) |
| Stock, valuation, godown, batches, pending orders | Excluded. **Re-check negative stock at approval** (stock may be consumed meanwhile). | `inventory.ts` `!v.optional`. Approval warning (section 6.6). |
| Bill-wise / outstanding / ageing / interest | Excluded. The bill stays open until approved. | `billwise.ts:46` |
| GSTR-1, 3B, 9, HSN, ITC, recon | Excluded | `gstr1.ts`, gst reports |
| Docs Issued (Table 13) | Pending is shown separately; rejected counts as cancelled | change in `docsissued.tsx` |
| TDS/TCS computation and returns | Excluded. The TDS assist threshold read ignores pending. | `tds.ts:255`, `tcs.ts:36` |
| Payroll / attendance | Approvals not offered in v1 (payroll vouchers are "edited on its own screen", `selection.ts:166`) | n/a |
| e-Invoice IRN / e-way bill | Blocked | `einvoice.tsx:178` |
| Print / share | Watermarked, or blocked per setting | `invoiceModel.ts:368` |
| Tally export | Skipped and counted as "pending approval" (the Tally export ask must state this) | `tallyExport.ts:485` |
| Day Book, registers, ledger vouchers (display) | Shown with a **PENDING** badge (like OPTIONAL, `VoucherScreen.tsx:2909`), excluded from totals | display only |
| Ghost recall / "last voucher" helpers | Skip pending | small guard |
| Ctrl+L on a held voucher | Toggles `approval.userOptional` (shown), never `optional` | VoucherScreen |
| Mark Regular batch | Blocked: "awaiting approval" | `markBlock`, `selection.ts:165` |
| Period lock (`lockedThrough`) | Approval of a voucher dated in a locked period needs full rights. Server reads the company's `security` from `change_log`. | `canAct` |

### 6.4 Client UX (keyboard-first)

**New screens.** Follow the CLAUDE.md "New screen checklist": the `Screen` union, router + crumb,
MenuScreen, CommandPalette, overlays GoTo, and `SCREEN_AREA` in `access.ts` (fail-closed).

- `approvals`: the inbox. Area `'open'`: any signed-in operator sees only what they may act on.
- `approvalRules`: the rule master. Area `'admin'`. It is a Tally-style `useTallyForm` list + form, like
  `Security.tsx`.

**Inbox keys.** These are proposals and must be verified against `src/keys.ts`, the frozen F-key
rail and the `?` sheet before build:

- **G then I** opens the inbox. `I` is free in `G_CHORDS`, `src/ui/overlays.tsx:438`.
- Tabs (Tab or ←→, like `Security.tsx`): **Awaiting me · Submitted by me · All pending (admin) · Decided**.
- ↑↓ moves. Enter opens the voucher **read-only** with an approval band: rule, level, trail and
  checks-at-submit.
- **Ctrl+A approves** (the app's universal "accept"; superset rule, no new meaning). Alt+R rejects
  (a reason prompt). Alt+R is screen-scoped elsewhere, for example the Read-scope toggle in
  `ApiKeys.tsx:75` and a binding in VoucherScreen, so it is free on a new screen. Alt+B sends back
  (also screen-scoped in VoucherScreen and reports). Space selects. Ctrl+A on a selection is a bulk
  approve under `stepUpAbove`, with one confirm.

**Maker side: no new key.**

- While entering, as soon as a rule matches, the status line shows "Needs approval · Payments ≥ ₹1,00,000 → Priya (L1 of 2)".
- After Ctrl+A, the toast reads "Payment 43 saved · sent for approval".
- Approval writes in local-only mode use `putDoc(..., { origin: 'approval' })` so they stay **off
  the maker's undo stack**. This is the CLAUDE.md `origin:'flag'` gotcha, extended. Ctrl+Z must never
  "unapprove".

**Sidebar.** Add a Today/Pinned/Recent block "Awaiting you (3)" (`src/ui/Sidebar.tsx`).

**Phone (PWA).**

- The same SPA has a code-split `approvals` route with a single-column mobile layout:
  - a card list (type, party, amount, maker, age);
  - a detail view (lines, GST, narration, trail, checks-at-submit; later, attachments from the
    document-attachment research);
  - three big buttons;
  - a reason sheet.
- It reads the **server inbox API**, not a full Dexie copy of the company. Phones should not carry
  the whole books.
- Offline it shows a cached list read-only, because decisions are server actions.
- Step-up above `stepUpAbove`: re-enter the password or PIN first; WebAuthn passkeys later.
- Install prompt plus push opt-in after the first approval.

### 6.5 Backend (Fastify + Postgres, migration `backend/sql/007_approvals.sql`)

```sql
-- Prerequisite (team): invites + member list
CREATE TABLE tenant_invite (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
  email text NOT NULL, role text NOT NULL DEFAULT 'member',
  token_hash text NOT NULL UNIQUE, expires_at timestamptz NOT NULL,
  accepted_by uuid REFERENCES app_user(id), created_by uuid, created_at timestamptz DEFAULT now());

-- Bind a company operator (User) to a person's cloud account
CREATE TABLE company_user_link (
  tenant_id uuid NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
  cid text NOT NULL, company_user_id text NOT NULL,
  app_user_id uuid NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
  PRIMARY KEY (tenant_id, cid, company_user_id),
  UNIQUE (tenant_id, cid, app_user_id));

CREATE TABLE approval_request (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
  cid text NOT NULL, object_type text NOT NULL DEFAULT 'vouchers', object_id text NOT NULL,
  rule_id text NOT NULL, rule_snapshot jsonb NOT NULL,
  state text NOT NULL CHECK (state IN ('pending','approved','rejected','sent_back','withdrawn')),
  level smallint NOT NULL DEFAULT 1, round smallint NOT NULL DEFAULT 1,
  amount numeric(18,2), vch_hash text NOT NULL,
  submitted_by_op text, submitted_by_user uuid, submitted_seq bigint,
  version int NOT NULL DEFAULT 1,                       -- optimistic concurrency for /decide
  created_at timestamptz NOT NULL DEFAULT now(), updated_at timestamptz NOT NULL DEFAULT now());
CREATE UNIQUE INDEX ux_approval_open ON approval_request(tenant_id, cid, object_type, object_id)
  WHERE state IN ('pending','sent_back');
CREATE INDEX ix_approval_inbox ON approval_request(tenant_id, cid, state, level);

CREATE TABLE approval_action (                           -- append-only audit
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id uuid NOT NULL, request_id uuid NOT NULL REFERENCES approval_request(id),
  action text NOT NULL, level smallint, round smallint,
  actor_user uuid, actor_op text, channel text NOT NULL, note text,
  vch_hash text NOT NULL, ip inet, user_agent text, at timestamptz NOT NULL DEFAULT now());

CREATE TABLE push_subscription (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(), tenant_id uuid NOT NULL, app_user_id uuid NOT NULL,
  endpoint text NOT NULL UNIQUE, p256dh text NOT NULL, auth text NOT NULL,
  ua text, created_at timestamptz DEFAULT now(), last_ok_at timestamptz);

CREATE TABLE notification_outbox (                      -- one row per (recipient, channel, event)
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, tenant_id uuid NOT NULL,
  app_user_id uuid NOT NULL, channel text NOT NULL, event text NOT NULL, payload jsonb NOT NULL,
  status text NOT NULL DEFAULT 'queued', attempts int NOT NULL DEFAULT 0, next_attempt_at timestamptz,
  dedupe_key text NOT NULL UNIQUE, created_at timestamptz DEFAULT now());

-- RLS on every table above, same policy shape as 002_p2.sql:
--   USING / WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid)
-- plus FORCE ROW LEVEL SECURITY on the new tables (the trust map notes ENABLE-not-FORCE today).
REVOKE UPDATE, DELETE ON approval_action FROM comhard_app;   -- edit log cannot be rewritten
```

**Endpoints.** All go through `withTenant()`, and all require membership + a `company_user_link` for the `cid`.

| Endpoint | Purpose |
|---|---|
| `POST /tenants/invites`, `POST /invites/:token/accept`, `GET /tenants/members`, `POST /companies/:cid/user-links` | Prerequisite team plumbing (owner/admin only) |
| `GET /approvals/inbox?cid=&tab=` | Items where `canAct(actor)` at the current level, or the maker's own submissions |
| `GET /approvals/:id` | Request + action history + current voucher payload (latest `change_log` row) |
| `POST /approvals/:id/decide` `{action, note, version, vchHash, stepUp?}` | Locks the request `FOR UPDATE`, then checks `version` and `vchHash` (**409** if the voucher changed or someone else acted). Appends to `approval_action`, updates the request, and **appends a server-authored voucher row to `change_log`** (`device_id='server'`, `user_id=actor`) with the new `approval`, `optional` and (N2) number. Then the notification outbox, SSE nudge and webhook. One transaction. |
| `POST /approvals/bulk-decide` | Per-id results; refuses any id above `stepUpAbove` |
| `POST/DELETE /push/subscription` | Web push subscriptions |
| Webhook events | `approval.submitted`, `approval.approved`, `approval.rejected`, `approval.sent_back` (`platform.js` `deliverEvent`) |

**Push path (`/sync/push`), per voucher row.** All of this happens in the existing push transaction,
and it needs `changeUid` idempotency first (ARCHITECTURE §6.1). Otherwise a retried push
double-submits.

1. Load the open request for the doc `FOR UPDATE`. Load the rules for the `cid` (latest `approvalRules`
   rows from `change_log`, cached by max seq). Load the ledger→group map for party-group rules.
2. No request and `matchRule` hits → create a request and force `approval={pending,L1,...}` and
   `optional=true` (hold). **This also catches old clients that predate approvals** (version skew).
3. Request open and `materialHash` unchanged → overwrite `approval`/`optional` with the server's values.
4. Content changed while pending → reopen at L1. Changed after approval → per `onEditAfterApproval`:
   reopen (new round), or for `lock`, refuse.
5. Delete of a held voucher → `withdrawn`. Delete of an **approved** voucher by a non-full role, or a
   `lock` violation, is refused through the v2 `rejected[]` response (ARCHITECTURE §6.6). Until v2
   exists, the fallback is a compensating re-put of the approved version plus a conflict entry in the
   sync mismatch report (the consultants' separate "mismatch report" ask).
6. If the stored payload differs from what the client sent, store it with `device_id='server'` so the
   origin device pulls the correction.
7. Desk-PIN approvals arriving in a payload are accepted only if `rule.allowDeskApproval` is set, the
   amount is below `stepUpAbove`, and `canAct` passes for the operator id. They are logged with
   `channel='desk-pin'`.

**REST (`POST /v1/vouchers`).** It runs the same transform with `source='api'`, so API-created
vouchers obey rules (U9).

### 6.6 Sync semantics and conflicts

| Scenario | Outcome | User sees |
|---|---|---|
| Maker saves offline; rule matches | Local `pending`, held. Approvers are notified only after the push. | "Will be sent for approval when online" |
| Approver approves on phone; maker offline | The server row reaches the maker on the next pull (`applyRemote`, no guard, by design) | Badge flips to Approved; toast |
| Maker edits a pending voucher offline while the approver approves | Maker push (stale base) → content hash differs from the approved hash → **reopen** (approval voided), trail "edited after approval by Ravi" | Approver is notified "changed after you approved"; maker sees "re-approval needed" |
| Two any-of approvers act at once | Row lock serialises them. The second gets 409 with the current state. | "Already approved by Priya at 14:02" |
| Approver acts on a stale view | `vchHash`/`version` mismatch → 409 | "This voucher changed. Review again." |
| Rule edited while items in flight | The snapshot governs in-flight items. Resubmission uses the new rule. | Rule form warns "N items in flight keep the old routing" |
| Approved purchase would now take stock negative or breach a credit limit | The server cannot compute stock (no server engines until the P4 materializer). **Checks-at-submit** are computed by the maker's device and stored on the request. Re-validation runs on the next device that loads the approved voucher; a failure lands in Exceptions, never as a silent revert. | Approver sees the submit-time checks with their age |
| Old client (no approval code) pushes a matching voucher | The server transform holds it (step 2). The old client then displays it as optional (it knows that field). | Degrades safely |
| Local-only company later connects to cloud | Local trail entries become `desk-pin` actions if `allowDeskApproval`; otherwise they are re-queued as pending | One-time migration notice |
| Clock skew | Decision times are server time. `submittedAt` is advisory only. | n/a |

### 6.7 Offline behaviour, summarised

- **Makers:** fully offline-capable. Pending is a local state; numbering follows N1.
- **Approvers:**
  - A cloud decision needs connectivity. That is acceptable, because the phone is online by
    definition when the push arrives.
  - Desk approval offline is possible only through `allowDeskApproval` (a soft control).
- **Notifications** start at the maker's first successful push. The inbox shows the "submitted
  (device time)" and "received by server" times separately, so an approver is never blamed for
  latency they could not see.

### 6.8 Audit trail

- **Three layers:**
  1. the client outbox (local edit log, with `operator`);
  2. the server `change_log` (every voucher version, including server-authored approval rows);
  3. `approval_action` (append-only, no UPDATE or DELETE grant).
- Each action stores the **`vch_hash` of exactly what the actor saw**. An auditor can prove the approved
  content equals the posted content. This is a "what exactly was approved" guarantee that none of the
  surveyed products document.
- The Audit Log screen (`src/ui/admin/AuditLog.tsx`) gains an "Approval" filter. A new
  **Approval History** report (SAP B1's "approval status / decision" reports) lists per voucher:
  maker, each level, decision, channel, time and lag.
- Satisfies the MCA Rule 3(1) edit-log expectation for the approval dimension. The hash chain over
  `change_log` (ARCHITECTURE §8) is still **designed, not built**. Until it is, "tamper-evident"
  holds against the app role only, not against the DB owner.

### 6.9 Notifications pipeline

- `notification_outbox` rows are written in the decision/submission transaction (no lost
  notifications) and drained by an in-process worker with backoff. Single VM for now; it moves to
  the queue in ARCHITECTURE §9 later.
- Channel preference is per user, plus a **digest mode** (one 9 am / 6 pm summary) so owners are
  not spammed.
- Optional escalation at `escalateAfterHours`, plus a GST-aware nudge for pending outward documents
  near the 30-day IRP window.
- **Payload minimisation:** the default lock-screen text is "Payment ₹1.2L awaiting your
  approval". There is a per-tenant "private notifications" switch ("1 voucher awaits approval"). Never
  send line items, GSTINs or bank details through third-party channels.

---

## 7. Security, privacy, legal

- **Segregation of duties:** no self-approval by default, compared on company User ids
  (section 4.4). "Final approve" is limited to full-rights roles, needs a mandatory reason, and is
  flagged in the Approval History.
- **Approver authentication:**
  - per-person cloud accounts only;
  - step-up re-auth above `stepUpAbove`;
  - refresh-token reuse detection already exists (`backend/src/tokens.js`);
  - recommend MFA (ARCHITECTURE §7) before marketing phone approval for large amounts.
- **No approve-on-GET.** Email and WhatsApp carry a **deep link to the inbox item**. The action
  is always an authenticated POST from the app. Link scanners and previewers cannot approve. The
  link carries an opaque id, **never personal data in the query string**.
- **WhatsApp quick-reply "Approve"** (if ever offered):
  - limited to below-threshold items;
  - phone↔user binding verified by OTP at opt-in;
  - webhook signature verified (`X-Hub-Signature-256`);
  - single-use, short-TTL action tokens.
  Default: off.
- **Tenant isolation:** RLS on all new tables, `FORCE ROW LEVEL SECURITY` on them, and `company_user_link`
  checked on every approval route, so a member of tenant X cannot act on company Y even within X if not
  linked. The SSE stream stays per-tenant. Recommend per-cid filtering with the v2 cursor map
  (ARCHITECTURE §6.2).
- **Transport:** web push and service workers **require HTTPS**. Bearer tokens on HTTP are
  already the top trust-map risk (HANDOFF §2). TLS is a hard gate for phase C, not a nice-to-have.
- **DPDP:**
  - Comhard acts as **processor** for tenants' books and as **fiduciary** for its own users' account
    data (emails, phones, push endpoints).
  - Collect phone numbers only on WhatsApp opt-in, with a notice.
  - List Meta, the BSP and the email provider as sub-processors in the DPA.
  - Push endpoints are personal data; delete them on logout or 410.
  - Accounting approval records are kept for the statutory period even after a user-erasure request.
  - Most DPDP obligations bite from about May 2027, so design them in now.
- **GST:**
  - Rule 46 integrity through N1/N2 (section 4.2).
  - Table 13 pending vs cancelled.
  - No IRN or e-way for pending.
  - "Approval lag" warnings against s.31 timing and the 30-day IRP window.
  - Recommend that clients gate **Sales Orders / discounts / credit-limit breaches** rather than
    every sales invoice. This is a CA question for the owner (section 10).
- **Not a payment control:** state in-product that approval does not authorise any bank movement
  (section 3.3, RBI row).

---

## 8. MVP without credentials vs what needs credentials

| Capability | External credential / contract? | Buildable now? |
|---|---|---|
| Rules master, held pending vouchers, desk inbox, PIN-operator approvals, audit (option A) | None | **Yes** |
| Cloud approvals, server authority, in-app inbox on any synced device, SSE, webhooks (B) | None (own backend). It needs internal prerequisites: invites, user links, `changeUid`, `rejected[]`, client SSE. | **Yes** |
| PWA install + **web push** (C) | **No contract**, but **HTTPS is required**. Needs a TLS cert (Let's Encrypt is free) via DNS-01 (a DNS API token for the domain) or a provider routing 443→VM. Today `books.comhard.in:80/443` serves a different app (HANDOFF §2). | **Blocked until TLS** |
| **Email** notifications (C) | An email provider account (SES or similar) + SPF/DKIM DNS records | Trivial once the account and DNS exist |
| **WhatsApp** (D) | **Yes:** Meta Business verification, WABA, display-name + template approval, BSP contract or direct Cloud API, recipient opt-in | **No.** Needs the owner to start Meta onboarding. |
| Native app push (E) | Apple Developer + Google Play accounts | Deferred |

**So yes: an MVP without credentials exists (A + B).** It delivers desktop and any-synced-device
approvals with server authority. **Phone push needs HTTPS only.** Nothing claimed here requires a
credential we do not list.

---

## 9. Phased plan with estimates

Assumes 1 senior full-stack engineer with the repo's AI-assisted fan-out method (HANDOFF §4).
"wk" means engineer-weeks, and each phase ends with the adversarial review + fix loop.

| Phase | Contents | Effort | Gate / exit test |
|---|---|---|---|
| **P0. Prerequisites** (worth doing anyway) | TLS on the app origin (1-3 d, plus DNS access lead time). **Team:** invites, member list, roles, `company_user_link` UI in Security (6-8 d). Push **`changeUid` idempotency** (3-4 d). Push sends the operator **id**. **v2 `rejected[]`** per-row response + client surfacing (3-5 d). Client **SSE subscription** (2 d). Fix the `voucher_counter` key to `(tenant,cid,type,series,fy)` (1 d). | **3-4 wk** | A second person can join a tenant and sign in on a phone over HTTPS. A replayed push creates no duplicate rows. |
| **P1. Local maker-checker** (option A) | Types + Dexie v8 `approvalRules`; shared `approvalPolicy.js` + assertion suite; save path + write-guard invariant; Ctrl+L / Mark Regular / undo-origin handling; Rules screen; Inbox screen + keys + `?` sheet; PENDING badges; split Optional vs Pending register; Docs Issued pending/cancelled; print watermark; Tally-export count; Approval History report; New-screen checklist + `SCREEN_AREA` | **4 wk** | A pending voucher appears in no total, return or export (assert it across every optional suite). Approve posts it. Reject keeps its number as cancelled. |
| **P2. Server authority** (option B) | `007_approvals.sql` + RLS/FORCE; push transform (field ownership, reopen, old-client catch); REST path; inbox/decide/bulk endpoints with 409 semantics; server-authored `change_log` rows; webhooks events; **first backend integration tests** (none exist today, per HANDOFF §2) | **3 wk** | A crafted push claiming `approved` is corrected on every other device. Concurrent decides serialise. An API-created voucher goes pending. |
| **P3. Phone + push + email** (option C) | Manifest + service worker + code-split approvals route + mobile layout; VAPID web push + subscription management; SES email + templates + SPF/DKIM; step-up re-auth; private-notification switch; digest; escalation | **2.5 wk** | Android + iOS 16.4+ (installed) receive a push and approve in 2 taps. The email link never approves on GET. |
| **P4. WhatsApp** (option D, on demand) | BSP integration, utility templates (deep-link), opt-in + OTP binding, webhook verify | **1-1.5 wk** + Meta onboarding lead time (unknown: days to weeks) | Template approved, opt-in recorded, messages delivered |
| **P5. Later** | Master approvals (new ledgers, **GST rate / HSN changes**, which ties to the consultants' effective-date ask); fold Leave applications into the same engine; delegation / out-of-office; server-side re-validation once the P4 report materializer exists; MFA/passkeys | n/a | n/a |

**To desktop + phone approvals (P0-P3): about 12-14 wk raw.** P0 overlaps heavily with the Phase-B
productionisation backlog (TLS, idempotency). The approval-specific work (P1-P3) is about 9.5 wk.

---

## 10. Open questions the owner must answer

1. **Which voucher types first?** Recommendation: Purchase, Payment, Journal, Debit/Credit Note in
   `hold`; Receipt/Contra in `review`. Sales by Sales Order or discount rule, not every invoice.
   **Get a CA's view on holding sales invoices** (s.31 timing).
2. **Numbering:** is N1 (number at entry; rejected = cancelled in Table 13) acceptable to your CAs as
   the default? Which voucher types, if any, want N2 (number at approval)?
3. **Can an approver edit before approving?** Recommendation: no. Send back instead; an approver who edits
   becomes the maker.
4. **Self-approval for owners, and "Final approve" for admins?** Recommendation: self-approval off,
   Final approve on with a reason.
5. **Per-person accounts and pricing.** Approvals make seats real. Plan `max_users` is
   display-only today (CLAUDE.md). Will approver seats be priced, enforced, or free?
6. **TLS and domain.** Who controls DNS for the product domain (still `books.comhard.in`, or a Comsky
   domain)? Can 443 be routed to the VM, or should we use DNS-01 on :1036?
7. **WhatsApp.** Is there budget for it? Should it run from one Comhard/Comsky platform number (simplest) or
   per-tenant numbers? Which BSP do we use?
8. **Local-only customers** (no cloud): is a same-device, PIN-based soft approval an acceptable
   product, labelled as such?
9. **Masters in scope?** Ledger creation, GST rate / HSN changes and credit-limit changes (BUSY approves
   masters). These are proposed for P5.
10. **SLA behaviour.** Escalate only, or auto-reject after N days like QuickBooks' 30-day auto-deny?
    Recommendation: escalate, never auto-reject an accounting document.
11. **Approval in a locked period.** Should it be refused, or allowed for full-rights users with a reason?
12. **Notification privacy default.** Should lock screens show the amount, or "1 voucher awaits approval"?

---

## 11. Sources

**Competitors:** Zoho Books
[configure approvals](https://www.zoho.com/books/help/transaction-approval/configure-approvals.html) ·
[multi-level](https://zstatic.zohostatic.com/books/kb/transaction-approval/configure-multi-level-approval.md) ·
[custom approval](https://www.zoho.com/books/help/transaction-approval/custom-approval.html) ·
[approval flow](https://www.zoho.com/en-de/books/help/transaction-approval-new-flow/transaction-approval-workflow.html) ·
[draft/approved not in reports](https://www.zoho.com/us/books/kb/banking/approved-transactions-in-reports.html) ·
India pricing (third-party): [Patron Accounting](https://www.patronaccounting.com/blog/zoho-books-pricing-india-2026), [reseller](https://codroiditlabs.com/zoho-books-reseller-india-2/).
QuickBooks: [custom workflows](https://quickbooks.intuit.com/learn-support/en-uk/help-article/manage-workflows/create-custom-workflows-quickbooks-online-advanced/L0SVebThg_GB_en_GB) ·
[bill approval](https://quickbooks.intuit.com/learn-support/en-us/help-article/manage-workflows/set-use-bill-approval-and-payment-release-workflows/L1IOLL9hv_US_en_US).
Xero (third-party): [guide](https://invoicedataextraction.com/blog/xero-bill-approval-workflow) ·
[multi-level request](https://productideas.xero.com/forums/939198-for-small-businesses/suggestions/44960764-bills-ability-to-allow-multiple-levels-of-approv).
SAP B1: [help 9.0](https://help.sap.com/doc/saphelp_sbo900/9.0/en-US/45/06dab4d8696c30e10000000a114a6b/content.htm) ·
[SAP Learning](https://learning.sap.com/courses/implementing-sap-business-one/implementing-the-approval-processes-in-sap-business-one).
BUSY: [approval management](https://busy.in/accounting-software/user-approval/) ·
[multilevel FAQ](https://busy.in/faqs/how-to-handle-multilevel-and-matrix-approvals-in-busy-answerid-63057/) ·
[voucher approval FAQ](https://busy.in/faqs/what-is-the-process-for-voucher-approval-answerid-67744).
RealBooks: [maker-checker](https://www.realbooks.in/?p=5019).
TallyPrime: [7.0 release notes](https://help.tallysolutions.com/release-notes-tallyprime-7-0) ·
[security levels](https://help.tallysolutions.com/article/te9rel61/Data_Management/DM_Assign_Security_Level.htm) ·
[audit-trail guide](https://tallysolutions.com/business-guides/tallyprime-audit-trails-and-controls-to-satisfy-external-auditors/) ·
unverified partner claim: [tallyatcloud](https://tallyatcloud.com/articles/remote-access-with-approvals-in-tally-prime-gold) ·
TDL add-on: [listing](https://vikashamittech.odoo.com/shop/tally-tdl-vouchers-verify-module-l-vouchers-approval-with-password-protection-95).

**Regulatory:** [CGST Rule 46 (CBIC)](https://taxinformation.cbic.gov.in/content/html/tax_repository/gst/rules/cgst_rules/active/chapter6/rule46_v1.00.html) ·
[e-invoice 30-day limit, AATO ≥ ₹10 cr (Taxmann)](https://www.taxmann.com/post/blog/time-limit-of-30-days-for-reporting-e-invoice-on-irp-portal-for-taxpayers-with-aato-of-10-crores-and-above-gstn-update/) ·
[MCA audit trail (ClearTax)](https://news.cleartax.in/mandatory-to-use-accounting-software-with-audit-trail-mca/6016/amp) ·
[Mondaq](https://www.mondaq.com:443/india/accounting-standards/1071750/audit-trails-under-the-companies-act-a-hazy-path-ahead) ·
[DPDP Rules notified (PTI via The Week)](https://www.theweek.in/wire-updates/business/2025/11/14/del148-biz-dpdp-rules-ld-govt.html) ·
[DPDP roadmap (TCSA)](https://www.tcsa.in/resources/dpdp-rules-2025-implementation-roadmap).
Items marked "general knowledge" (CGST s.31 timing, Rule 138 e-way, Companies Act s.128(5) 8-year
retention, DPDP s.8 retention carve-out, TRAI DLT, app-store fees, free browser push services)
were **not re-verified** in this research.

**Channels and costs:** [Meta WhatsApp pricing](https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing) ·
[AiSensy India rates / Oct 2026 change](https://m.aisensy.com/blog/whatsapp-pricing-update-october-2026/) (BSP-quoted, unverified vs Meta's INR rate card) ·
[Amazon SES pricing](https://aws.amazon.com/ses/pricing/) ·
[iOS web push (OneSignal)](https://documentation.onesignal.com/docs/web-push-for-ios) ·
[PWA iOS limits (MagicBell)](https://www.magicbell.com/blog/pwa-ios-limitations-safari-support-complete-guide) ·
[web-push library](https://github.com/web-push-libs/web-push).
