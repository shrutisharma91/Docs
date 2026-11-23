# Document attachments and drive integrations: research and recommendation

**Ask (consultants, 2026-10-15):** "There should be an option for document attachment, which
is currently not there in Tally, and we can explore integration with Comsky Drive, OneDrive
or Google Drive."

**Status:** research and design only. No code has been written. Web sources were checked on
2026-10-15. Where a claim could not be verified it says so. Effort figures are estimates for
one senior full-stack engineer who knows this repo, and they carry roughly ±30%.

---

## TL;DR

1. **Build attachments natively and local-first.** Each file is a content-addressed blob
   (SHA-256). A new synced `attachments` document type rides the existing outbox and
   `change_log`. The bytes travel on a **separate upload channel** to object storage, never
   through `/sync/push`. Attachments are separate rows and never an array inside `Voucher`.
   Full-document last-write-wins sync would otherwise lose one device's attachment.
2. **An MVP needs no external credentials.** It covers attaching files (picker, drag-drop,
   clipboard paste, phone camera), link-only attachments (a pasted Drive or OneDrive URL), a
   viewer, an `Alt+U` panel, a "vouchers without documents" report, an audit-pack export, and
   cloud sync to the VM's disk behind a storage adapter. The first OCR step can also ship
   without credentials: decoding the GST e-invoice signed QR code to pre-fill a draft.
3. **Google Drive and OneDrive pickers are feasible but blocked by HTTPS.** The light scopes
   are enough: Google `drive.file` is non-sensitive, so no CASA audit is needed, and OneDrive
   needs only delegated read. However, Google and Microsoft both refuse non-localhost
   `http://` origins and redirect URIs, and the live app is HTTP-only. The owner must also
   create a Google Cloud project with brand verification and an Entra app registration
   (publisher verification is advisable).
4. **"Comsky Drive" does not exist yet.** `comsky.ai/drive` says it is "in design — no build,
   no beta, nothing to sign into". `products.comsky.in` and `tally.comsky.in` do not resolve
   in DNS. Build a storage-provider seam now and do not wait for Drive.
5. **Effort:** the MVP (local plus cloud sync) is about 4–5 engineer-weeks. The provider
   pickers add about 1.5–2 weeks, plus external verification lead time. OCR to draft vouchers
   comes later: 1–1.5 weeks for the credential-free version, and 2–3 weeks for cloud or LLM
   extraction.

---

## 1. What the ask really needs

### 1.1 Is it true that Tally does not have it?

Mostly true. The TallyPrime 7.0 release notes (launched 19 Dec 2025) do not mention attaching
documents to vouchers or masters. Their "TallyDrive" is **backup storage**, not attachments:
"Schedule backup of your data to TallyDrive or local storage"
([release notes](https://help.tallysolutions.com/release-notes-tallyprime-7-0),
[product update](https://tallysolutions.com/darpan/product-update-tallyprime-7-0/)).
Tally partners fill the gap with paid TDL add-ons sold on IndiaMART (for example
[1](https://m.indiamart.com/proddetail/dynamic-document-attach-module-in-tally-prime-2853254767830.html),
[2](https://m.indiamart.com/proddetail/document-management-system-in-tally-2851973485997.html);
these are seller claims and were not verified). Tally's own move is the opposite direction:
**Docs by Ira** turns PDFs and images *into* vouchers. It requires TallyPrime 7.1 or later,
gives 100 free pages for two months, and "files can't be pulled directly from Google Drive
or OneDrive"
([Tally article](https://tallysolutions.com/smart-moves/expert-insights/how-tallyprimes-latest-ai-feature-docs-by-ira-helps-businesses-reduce-manual-accounting-work/),
[release notes](https://help.tallysolutions.com/release-notes-docs-by-ira-tallyprime)).
Neither source says whether the source document stays linked to the voucher.

So two jobs are hiding in one ask. (a) **Evidence**: keep the bill with the entry, for audit,
GST, and the CA. (b) **Capture**: turn the bill into the entry. Build (a) first. It is cheap,
needs no credentials, and every later step (OCR, approvals, the customer portal) consumes it.

### 1.2 User stories

| # | As a… | I want to… | Keyboard path / surface |
|---|---|---|---|
| U1 | Data-entry operator keying a purchase bill | attach the supplier's PDF or photo without leaving voucher entry or breaking the frozen field order | `Alt+U` opens the Attachments panel. Inside it: `A` add file (OS picker), `Ctrl+V` paste a screenshot, or drag a file anywhere on the screen. `Esc` returns to the field I left. The header shows "2 attached". |
| U2 | Owner on a phone | photograph a bill and attach it to a voucher, or drop it in an inbox to attach later | Web app on mobile, `<input type="file" accept="image/*,application/pdf" capture="environment">`. No native app is needed for v1. |
| U3 | Auditor or CA | open a voucher from the Day Book and see its evidence; list vouchers that have none | Day Book shows an attachment glyph column. `Alt+U` then `Enter` views. New **Vouchers without Supporting Documents** exception report. |
| U4 | Approver (ties to the approvals research) | see the bill before I approve a payment | Approval card shows thumbnails or a page-1 preview through a short-lived signed URL. |
| U5 | Accountant whose files live in Google Drive or OneDrive | link (v1) or pick and copy (v3) the file instead of downloading and re-uploading it | `Alt+U` then `L` paste link (v1). `Alt+U` then `G` / `O` opens the provider picker (v3). |
| U6 | Accountant on a master | keep the GST certificate, PAN, or cancelled cheque on the party ledger; spec sheets or photos on stock items; KYC on employees | Same `Alt+U` panel on master alteration screens. Employee and KYC files are tagged **sensitive** and need the payroll right to view. |
| U7 | Operator working offline | attach now and sync later; another device shows "pending upload from Ravi's laptop", not a broken file | Local blob store plus an upload queue (section 5.4). |
| U8 | Company admin | see storage used, stay inside the plan quota, and be unable to destroy evidence inside the retention period | Storage line in Company/Subscription screens. "Remove" is a soft-delete with a reason, audited. |
| U9 | Owner preparing for GST audit or scrutiny | export vouchers with their documents in one pack | **Audit pack** export (zip): register CSV plus files named `date_type_number_n.ext` plus SHA-256 index. |

**Non-goals for v1:** editing documents, file versioning, folder management (that is a Drive
product's job), an email-in inbox, and OCR.

---

## 2. Fit with what exists today (codebase reading)

- **No attachment concept exists.** The only binary in the data model is `Company.logo`, a
  base64 data URL (`src/types.ts:127`, read with `FileReader.readAsDataURL` and **no size
  limit** in `src/ui/CompanyCreate.tsx:56-63`). It is pushed whole on every company edit.
  `docs/ARCHITECTURE.md` §9 ("Redis, attachments") and the Decision Log ("Blobs →
  object storage, key in payload") already decided that blobs leave row payloads. This
  design implements that decision and should migrate the logo in the same project.
- **Sync is full-document last-write-wins over a JSON outbox.** `putDoc` writes the row plus
  an outbox record (`src/data/db.ts:296-310`). `pushOutbox` sends 300 rows per JSON POST
  (`src/sync/sync.ts:36,525-551`). The server caps bodies at 20 MB
  (`backend/src/server.js`, `bodyLimit`) and stores `payload jsonb`
  (`backend/sql/001_p1.sql`). Bytes must not ride this path. One 15 MB scan would blow the
  batch, and every edit would re-push it. **Corollary:** attachments must be separate rows.
  If they were `voucher.attachments[]`, device A editing the narration while device B
  attaches a bill would lose one of the two changes under last-write-wins.
- **Older clients tolerate a new table.** `pullRemote` drops unknown tables
  (`isKnownTable`, `src/sync/sync.ts:523`). Adding `attachments` therefore does not break a
  device that has not upgraded.
- **Everything in `ALL_TABLES` is loaded into memory.** `loadCompanyData`
  (`src/data/db.ts:376`) loads every table into Zustand. Attachment *metadata* can live there
  because it is small. The **blob table must be outside `ALL_TABLES` and `COLLECTIONS`**.
- **Backups iterate `ALL_TABLES`.** `exportCompany` and `importCompany`
  (`src/data/db.ts:337-374`) would carry metadata automatically. Blobs need a container
  format (section 5.9). The on-disk tag `tallyweb-backup-v2` must stay readable.
- **The REST surface is generic.** `GET /v1/:collection` (`backend/src/rest.js`) will expose
  `attachments` metadata to API keys and fire `attachments.created` webhooks with no new
  code. Binary content needs one dedicated route.
- **Tenancy goes through `withTenant()` and the `app.tenant_id` GUC with RLS**
  (`backend/src/db.js`, `backend/sql/002_p2.sql`). New tables follow the same policy shape.
- **The HTTP-only deploy has no `crypto.subtle`.** WebCrypto SHA-256 is unavailable on
  `http://books.comhard.in:1036`. A pure-JS `sha256Pure` already exists in
  `src/core/vault.ts:158` (and `src/polyfills.ts` shows the same pattern for
  `randomUUID`). Hash with WebCrypto when available and fall back to `sha256Pure`.
- **Voucher ids are minted at save** (`src/ui/VoucherScreen.tsx:2252`,
  `id: editing?.id ?? crypto.randomUUID()`). The panel needs a draft id allocated when the
  screen mounts and reused at save.
- **`Alt+2` duplicate deliberately drops bill and instrument allocations** (CLAUDE.md Phase 13).
  Attachments must also **not** be copied. A duplicated voucher carrying the source's bill is
  false evidence.
- **Keys:** `Alt+U` is unbound in `VoucherScreen.tsx`, `overlays.tsx`, `App.tsx` and
  `AppShell.tsx` today (grep). Confirm against the frozen TallyPrime keymap before shipping.
  Avoid `Ctrl+U`, which is View Source in browsers.
- **Undo gotcha (CLAUDE.md):** anything written through `putDoc` without
  `origin: 'flag'` becomes a "Voucher N saved · Undo" step. Attachment writes need their own
  origin tag so the undo toast and the Today count skip them.
- **Backups:** `backend/deploy/backup.sh` is a `pg_dump` on the **same VM**. A blob directory
  on that disk would not be backed up at all.
- **Durable storage:** `src/data/persist.ts` already calls `navigator.storage.persist()` and
  warns when it is denied.
- **Electron** pins `userData` to `"Comhard Books"` (`electron/main.cjs`), and IndexedDB blobs
  will live there. Keep the pin. The preload exposes no native file APIs today.
- **Plan limits are display-only** (`backend/sql/003_saas.sql`, CLAUDE.md). A **storage quota
  should be the first enforced limit** because it protects the VM disk. It is easy to enforce
  at the one upload endpoint.

---

## 3. Landscape

### 3.1 What "Comsky Drive" is (finding)

- `products.comsky.in` and `tally.comsky.in` **do not resolve** (DNS lookup on 2026-10-15
  returned no A record for either). `comsky.in` 301-redirects to `comsky.ai`.
- [comsky.ai](https://comsky.ai/) (Comhard Technologies Pvt Ltd, Noida) lists 14 products.
  Comsky Drive is "Building": "One place for the files the company actually shares."
- [comsky.ai/drive](https://comsky.ai/drive) is explicit that it is "in design — no build, no
  beta, nothing to sign into". It also says "Drive has no infrastructure behind it at all".
  Storage country, versioning, sharing links, capacity, price, sync and mobile are all
  "not settled". **No API is mentioned and no Books integration is mentioned.**
- Two Comsky products are relevant today:
  - [Comsky Cloud](https://comsky.ai/cloud) is live and offers **"S3-compatible object
    storage"** in India ("ZONE01 is the only zone Comsky runs"; there is no second region).
    Per-unit prices are not published.
  - [Comsky Backup](https://comsky.ai/backup) is live: encrypted, device-keyed backups into
    S3-compatible storage, with no S3 Object Lock claimed. It is a candidate for off-VM backup
    of the attachment store.

**Implication:** there is nothing to integrate with. The sensible move is to put Books'
attachment store on **Comsky Cloud S3-compatible storage**, through an adapter, in a layout
Drive could later adopt (`tenant/company/...`). Later, add a `comsky-drive` provider to the
same picker seam as Google and Microsoft, once Drive has an API and shared identity.

### 3.2 Competitors

| Product | Attachments on entries | Cloud drives | Capture / OCR | Notes |
|---|---|---|---|---|
| **TallyPrime 7.x** | Not native (TDL add-ons only) | TallyDrive is backup, not attachments | Docs by Ira (7.1+): PDF/image to vouchers, 100 free pages / 2 months, English, no Drive/OneDrive pull | [7.0 notes](https://help.tallysolutions.com/release-notes-tallyprime-7-0), [Ira](https://tallysolutions.com/smart-moves/expert-insights/how-tallyprimes-latest-ai-feature-docs-by-ira-helps-businesses-reduce-manual-accounting-work/) |
| **BUSY** | "Maintain Images/Notes with Masters/Vouchers" (inventory config); per-series image path | Google Drive save for sales invoices; **needs active BLS subscription** | — | [BUSY FAQ](https://busy.in/faqs/how-do-i-upload-documents-like-invoices-and-bills-on-busy-answerid-56043.md) |
| **Marg ERP** | Could not verify | Could not verify | Could not verify | No Marg-specific source found |
| **Vyapar (TaxOne) / Suvit** | Not verified for the core Vyapar app | — | OCR upload of sales and purchase bills, then sync to Tally; both state **50–80%** first-pass capture | [Vyapar TaxOne](https://taxone.vyapar.com/help/articles/ocr-purchase), [Suvit](https://help.suvit.io/articles/ocr-purchase). The two help sites carry the same OCR articles; their relationship was not verified. |
| **LOGIC ERP** (Indian mid-market) | "Attach Documents" on account vouchers, incl. camera / F4 | — | — | [LOGIC ERP KB](https://kb.logicerp.com/logic-erp-version-release-notes/november-2020/accounts/attach-documents-option-added-in-setup-account-voucher-and-its-sub-categories) |
| **Zoho Books** | Up to **10 files × 5 MB** per invoice | Upload from WorkDrive, Google Drive, Dropbox, OneDrive, Box, Evernote | Documents module with a per-org **email-in address**; Autoscan add-on **₹499/month per 50 scans** (₹419 annual) | [limits](https://zstatic.zohostatic.com/books/kb/invoices/multiple-attachments.md), [Documents](https://www.zoho.com/in/books/help/documents/documents.html). Inbox items cleared from the inbox cannot be retrieved. |
| **QuickBooks Online** | Transactions plus customer/supplier profiles (not accounts or employee lists); a central **Attachments list** | Not mentioned | Not in that article | Types: PDF, JPEG, PNG, DOC, XLSX, CSV, TIFF, GIF, XML. Max size **not stated** ([Intuit](https://quickbooks.intuit.com/learn-support/en-us/help-articles/attachments-in-quickbooks-online/00/261783)). |
| **Xero** | Files on invoices and bills, plus a Files library | — | — | Per-file limit **unverified**: user posts say 7 MB when emailing invoices ([Xero ideas](https://productideas.xero.com/forums/967115-invoices-quotes/suggestions/46678708-invoices-increase-attachment-size)); ApprovalMax documents 10 MB / 9 files ([ApprovalMax](https://support.approvalmax.com/en/articles/414533-what-file-types-can-be-uploaded-via-approvalmax-capture)) |

**Takeaways:**
- **Zoho Books is the template:** attach from local or cloud, a document inbox, and OCR as a
  paid add-on.
- **BUSY proves the Indian market will pay** a subscription for Drive linkage.
- **Nobody in the Tally/BUSY/Marg segment is local-first with attachments that work offline
  and sync.** That is our angle.

### 3.3 Provider integrations: what Google and Microsoft actually require

**Google Drive**

- **Scope.** `drive.file` is **non-sensitive** and "recommended for most use cases". It
  lets the app open only the files the user picks. `drive.readonly` and `drive` are
  **restricted**. Restricted scopes require restricted-scope verification, and "if you store
  restricted scope data on servers (or transmit), then you must go through a security
  assessment"
  ([Google scopes](https://developers.google.com/workspace/drive/api/guides/api-specific-auth)).
- **Verification.** With only non-sensitive scopes, "it is not mandatory for your app to
  complete the app verification process". Showing the app name and logo on the consent
  screen requires **brand verification**
  ([Google](https://support.google.com/cloud/answer/13463073)). Brand verification needs a
  homepage and privacy policy on a **domain verified in Google Search Console**
  ([Google](https://support.google.com/cloud/answer/13464321)).
- **CASA.** Restricted scopes trigger an annual CASA assessment. Third-party lab quotes are
  roughly USD 540 to 4,500+ per year
  ([Switch Labs](https://www.switchlabs.dev/post/casa-tier-2-tier-3-security-review-providers-pricing-and-the-cheapest-option),
  [Leviathan](https://leviathansecurity.com/programs/google-casa-cloud-application-security-assessment)).
  **Using only `drive.file` avoids this entirely.**
- **Picker (web).** It needs an OAuth access token, a developer **API key**, and an **App ID
  (the Cloud project number)** via `setAppId`
  ([web picker guide](https://developers.google.com/workspace/drive/picker/guides/web-picker)).
- **Picker (desktop and mobile).** Only `drive.file`, which "cannot be combined with other
  scopes". It opens in the **system browser** and "can no longer be displayed within an
  embedded webview"
  ([Picker overview](https://developers.google.com/workspace/drive/picker/guides/overview)).
  Electron therefore uses the desktop flow: a *Desktop app* OAuth client with a
  **loopback** redirect, which Google still supports for desktop clients
  ([loopback guide](https://developers.google.com/identity/protocols/oauth2/resources/loopback-migration)).
- **HTTPS (hard blocker).** "JavaScript origins must use the HTTPS scheme… Localhost URIs…
  are exempt". Raw IP hosts are not allowed, and non-default ports must be written
  explicitly ([Google](https://support.google.com/cloud/answer/15549257)).
  `http://books.comhard.in:1036` cannot be registered. `https://books.comhard.in:1036` could
  be, once TLS exists.
- **Cost.** The Cloud project, Picker and Drive API, Search Console and brand verification
  are free. Usage is quota-limited, not billed. No paid line item was found for this path.

**Microsoft OneDrive / SharePoint**

- **Picker.** File Picker v8 is a Microsoft-hosted page in a popup or iframe, driven by
  `postMessage`. It needs an **Entra (AAD) app registration** using SPA redirects
  ([OneDrive picker docs](https://learn.microsoft.com/en-us/onedrive/developer/controls/file-pickers/)).
- **Permissions (delegated).**
  - Minimum for work or school accounts is SharePoint `MyFiles.Read`; the sample also uses
    `Files.Read.All` and `Sites.Read.All`.
  - Personal accounts use authority `consumers` with token scope `OneDrive.ReadOnly`, while
    the app registration declares `Files.Read`.
  - "The control relies on SharePoint tokens and not Graph… you cannot reuse tokens for
    Graph calls".
  - Each pick returns `id`, `parentReference.driveId` and `@sharePoint.endpoint`, which are
    used to `GET` the item.
- **Verification.** Multi-tenant apps registered after Nov 2020 show "unverified" unless
  **publisher-verified**. With risk-based step-up consent, users "can't consent to most newly
  registered multitenant apps that aren't publisher verified". Publisher verification needs a
  verified **Microsoft AI Cloud Partner Program (Partner One ID)** account and a matching
  publisher domain
  ([publisher verification](https://learn.microsoft.com/en-ca/entra/identity-platform/publisher-verification-overview),
  [how to](https://learn.microsoft.com/ga-ie/entra/identity-platform/mark-app-as-publisher-verified)).
  A customer's M365 admin may still have to grant admin consent.
- **HTTPS (hard blocker).** Redirect URIs "must begin with the scheme https", with exceptions
  only for localhost ([Microsoft](https://learn.microsoft.com/entra/identity-platform/reply-url)).
- **Cost.** App registration is free. Whether Partner Center enrolment carries a fee was
  **not verified**.
- **Electron.** Whether Picker v8 works in Electron (its `messaging.origin` against a
  `file://` page) was **not verified**. Plan v1 Electron support as local files plus links.

**Link-only attachments (no provider API)** need no credentials. The trade-off is link rot.
Links "can break if permissions, tokens, or folder paths change" (Sage documentation,
[via Cleverence](https://www.cleverence.com/amp/articles/sage-documentation/about-file-and-link-attachments-on-records-sage-4927/)),
and a linked file is **not backed up by us** (section 6.3).

### 3.4 Storage backends

| Backend | India residency | Price signal | Notes |
|---|---|---|---|
| **VM local disk** (`/home/ubuntu/comhard/blobs`) | Yes (owner VM) | Already paid | Single disk, single VM, not covered by `backup.sh`. Free space is unknown (open question). Acceptable only with an off-box backup. |
| **Comsky Cloud S3-compatible** | India, single zone "ZONE01", no second region | Unpublished | In-house. Same company as Drive. Needs pricing and an S3 feature check (presigned URLs, CORS, versioning). |
| **AWS S3 ap-south-1 (Mumbai)** | India region | ~₹2.1/GB-month Standard; egress ~₹8–9/GB after 100 GB/month (reseller figures, **not verified on AWS's page**: [source](https://precisiontech.in/cloud/amazon-aws-cloud/aws-pricing/)) | Mature. Object Lock and versioning available. A second India region (Hyderabad) exists for DR. |
| **Cloudflare R2** | **No India jurisdiction** (jurisdictions: EU, FedRAMP, US; `apac` hint is "best effort and not a guarantee") | $0.015/GB-month, zero egress, 10 GB free ([pricing](https://developers.cloudflare.com/r2/pricing/), [location](https://developers.cloudflare.com/r2/reference/data-location/)) | Cheap, but cannot promise India storage. That conflicts with the Companies (Accounts) Rules requirement for in-India backup (section 6.3). |
| **Self-hosted MinIO** | Whatever disk it runs on | Free | **Avoid.** The community edition went source-only, entered maintenance mode on 3 Dec 2025, and was reportedly **archived 12 Feb 2026** (secondary sources: [Pigsty](https://blog.vonng.com/en/db/minio-resurrect/), [RAGFlow issue](https://github.com/infiniflow/ragflow/issues/13840)). Garage or SeaweedFS are alternatives if a self-hosted S3 is ever needed. Not evaluated here. |

**Sizing (estimate, not measured):**
- An SMB attaching about 300 bills a month at about 300 KB each (after the client downscales
  phone photos) produces roughly **1.1 GB per company per year**.
- 100 tenants × 3 companies is about 330 GB per year. At about ₹2/GB-month that is under
  ₹1,000 a month in year 1.
- Storage is not the cost driver. Engineering and the backup discipline are.

### 3.5 Legal and regulatory frameworks that touch attachments

- **Companies (Accounts) Rules 2014, Rule 3, as amended by the 4th Amendment Rules 2022
  (G.S.R. 624(E), 5 Aug 2022):**
  - Electronic books and papers must be "accessible in India, at all times".
  - Backups must be kept on **servers physically located in India on a daily basis**.
  - Service-provider and cloud-address details are disclosed annually in AOC-4.
  - The audit-trail proviso applies from FY 2023-24.
  - Sources: [KNAV](https://in.knavcpa.com/insights/flash-alert-mca-amends-the-companies-accounts-rules-2014-on-back-ups-of-the-books-of-accounts-for-indian-companies),
    [TaxHeal](https://www.taxheal.com/companies-accounts-fourth-amendment-rules-2022.html).
  - Supporting bills are "books of account and other relevant books and papers". **Therefore
    attachment storage, and its daily backup, should be in India.**
- **CGST Act s.36:** retain accounts and records until **72 months after the due date of the
  annual return**, or longer if under appeal or investigation
  ([text](https://www.knowyourgst.com/gstlaw/cgst-act/36-period-of-retention-of-accounts-36/)).
  **CGST Rule 57:** keep electronic back-ups that can be restored within a reasonable
  period, and produce authenticated records on demand
  ([text](https://gstzen.in/a/generation-and-maintenance-of-electronic-records-cgst-rule-57.html)).
  The audit pack (U9) is the "produce on demand" feature.
- **Companies Act 2013 s.128(5):** books preserved for 8 financial years. This is standard
  knowledge and was **not re-verified in this research**.
- **Income-tax Act 2025:** s.62 replaces s.44AA; draft Rule 46 replaces Rule 6F, with
  **7 tax years** retention and daily India backups for electronic books (draft Rule 46(8)).
  The final notified text was **not confirmed**
  ([TaxHeal](https://www.taxheal.com/?p=115206), [myHQ](https://myhq.in/blog/maintenance-of-books-of-accounts-in-india)).
- **DPDP Act 2023 and DPDP Rules 2025** (notified 13/14 Nov 2025):
  - The main Data Fiduciary obligations (notice, security safeguards, breach reporting,
    erasure, log retention) apply **18 months after publication, around May 2027**
    ([PIB](https://static.pib.gov.in/WriteReadData/specificdocs/documents/2025/nov/doc20251117695301.pdf),
    [Khaitan](https://www.khaitanco.com/sites/default/files/2025-11/ERGO%20-%20Digital%20Personal%20%20Data%20Protection%20Rules%20-%2015%20November%202025.pdf)).
  - Erasure is required when consent is withdrawn or the purpose is served, **unless another
    law requires retention**.
  - Logs must be retained for at least one year
    ([KSK summary](https://ksandk.com/data-protection-and-data-privacy/data-retention-deletion-dpdp-act-india/)).
  - Section 6.2 maps this to attachments.

---

## 4. Options compared

| Option | What the user gets | Effort | Running cost | External dependency / credentials | Risk | Works offline | Satisfies India backup |
|---|---|---|---|---|---|---|---|
| **A. Link-only** (store a URL) | A clickable reference to a Drive or OneDrive file | 2–3 d | 0 | None | Link rot; permissions drift; not evidence we hold | Metadata yes; file no | **No** |
| **B. Local-only blobs** (IndexedDB, no sync) | Attach and view on one device | 6–8 d | 0 | None | Lost with the device or browser data; Safari ITP eviction; not shared with the CA | Yes | Only via the user's own backup |
| **C. Native local-first + Comsky object store** | Attach anywhere, sync to every device, offline, audit pack | B + 10–14 d | Disk or S3 (small) | None for VM disk; an S3 account for Comsky Cloud or AWS | Same-origin content security; backups of the blob store | Yes | **Yes** (India store + daily backup) |
| **D. Provider-as-storage** (file lives only in the user's Drive; we keep a pointer) | Pick from Drive; no storage cost to us | 8–12 d | 0 | Google Cloud project, brand verification; Entra app, publisher verification; **HTTPS** | Each viewer needs their own Drive access; offline view impossible; files may leave India; the token-handling surface grows | No | **No** |
| **E. Provider pick → copy into C** (keep provider ref) | Pick from Drive; we hold the evidence | C + 6–9 d | As C | As D | Lowest of the provider options; tokens stay in the browser | Yes after copy | **Yes** |
| **F. Wait for Comsky Drive** | Unknown | Unknown | Unknown | Drive does not exist | Indefinite delay | — | — |

**Recommendation:** ship **C + A** as the MVP, then **E** as the provider phase. Build a
provider seam so **F** plugs in when Drive has an API. Do **not** adopt D as the primary
model: it fails the offline promise and the India-backup rule, and every viewer needs their
own access to the owner's Drive.

---

## 5. Recommended architecture

### 5.1 Client data model

```ts
// types.ts (sketch; names indicative)
export type AttachOwnerTable = 'vouchers' | 'ledgers' | 'stockItems' | 'employees' | 'companies' | 'leads';

export interface Attachment {            // SYNCED doc (outbox → change_log), in ALL_TABLES
  id: string; cid: string;
  ownerTable: AttachOwnerTable;          // flat fields so Dexie can index them
  ownerId: string;
  kind: 'file' | 'link';
  name: string;                          // display name, editable
  mime?: string;                         // SNIFFED from magic bytes, never trusted from the OS
  size?: number;
  sha256?: string;                       // file kind: lowercase hex content address
  url?: string;                          // link kind: https only
  source?: { via: 'file' | 'camera' | 'paste' | 'drop' | 'gdrive' | 'onedrive' | 'email' | 'comsky-drive';
             providerId?: string; webUrl?: string };   // provider reference kept for "open in Drive"
  caption?: string;
  tags?: string[];                       // 'supplier-bill' | 'gst-cert' | 'pan' | 'cheque' | …
  sensitive?: boolean;                   // payroll/KYC: needs the payroll right to view
  addedAt: string; addedBy?: string;
  removedAt?: string; removedBy?: string; removedReason?: string;   // soft delete only
  pages?: number;
}

export interface AttachmentBlob {        // LOCAL ONLY: NOT in ALL_TABLES, NOT in COLLECTIONS, never in the outbox
  sha256: string;
  blob: Blob;                            // IndexedDB stores Blobs natively; fall back to ArrayBuffer if a browser refuses (test Safari)
  size: number; mime: string;
  state: 'staged' | 'pending' | 'uploaded' | 'cached';
  lastUsed: number;                      // LRU eviction of 'uploaded'/'cached' only
  thumb?: Blob;                          // generated locally, never synced
}
```

Dexie **v8** is additive only, which matches the v6/v7 pattern:

```ts
this.version(8).stores({
  attachments: 'id, cid, ownerId, sha256',
  attachmentBlobs: 'sha256, state, lastUsed',
});
```

Checklist (CLAUDE.md "New master table"):
- `types.ts`, `db.ts`, `store.ts` `COLLECTIONS` (attachments only), `ALL_TABLES`
  (attachments only).
- `checkDelete`: deleting a voucher or ledger does **not** delete its attachments. They
  become orphans under retention and are shown in the Cancelled/Deleted register.

### 5.2 Write path and keyboard UX

- **Panel.** `Alt+U` on VoucherScreen and master alteration screens opens an overlay
  registered with `useScreenKeys`. It is never a global toggle (CLAUDE.md keyboard gotcha).
  Keys:
  - `A` add, `L` link, `Enter`/`V` view, `R` rename/caption, `D` remove (asks for a reason),
    `G` Google, `O` OneDrive, `Esc` close.
  - `Ctrl+V` anywhere on the voucher screen pastes a clipboard image as an attachment.
  - Dropping a file anywhere attaches it.
  - The `?` sheet and the palette ("Attach file to this voucher", "Vouchers without
    supporting documents") document all of these.
- **Ingest pipeline**, the same for every source:
  1. Sniff magic bytes against the allow-list: PDF, JPEG, PNG, WebP; XLSX/DOCX/CSV as
     download-only. Reject HTML, SVG, JS, executables and archives in v1.
  2. Downscale camera images to a 2,000 px long edge as JPEG (a "keep original" toggle
     exists).
  3. Compute SHA-256 with WebCrypto, falling back to `sha256Pure` on HTTP.
  4. Store the blob as `staged`.
  5. Show the item in the panel immediately.
- **Size defaults:** 10 MB per file (server hard cap 25 MB) and 20 files per object. These
  are configurable per plan.
- **New voucher.** Allocate `draftIdRef = crypto.randomUUID()` on mount and use it at save
  instead of minting a new id (`VoucherScreen.tsx:2252`). Attachment rows are written in the
  **same Dexie transaction as the voucher on `Ctrl+A`**, and staged blobs flip to `pending`.
  When the voucher is discarded, staged blobs are dropped. The header shows the staged count,
  per the CLAUDE.md rule that a carried value must be visible.
- **Existing voucher or master.** Attach and remove write immediately with
  `putDoc('attachments', …, { origin: 'attach' })`. The voucher doc is not re-saved, so
  nothing else about the voucher changes. Extend the undo, toast and Today filters to skip
  this origin, as `'flag'` already does.
- **Permission.** Add a `writeGuard` rule for table `attachments`:
  - **Adding** needs the right to *create* the owner's type.
  - **Removing** needs *alter* and is refused inside a locked period.
  - Whether **adding** inside a locked period is allowed is an open question. It does not
    change the books, but it changes the evidence.
- **Alt+2 duplicate and Bulk Entry** never copy attachments.

### 5.3 Viewing

- In-app viewer: fetch with `Authorization: Bearer` into a Blob, then a `blob:` object URL.
  - PDFs render in an `<iframe>`. Images render in `<img>` with zoom, rotate and page keys.
  - **Never create a `blob:` URL for a type outside the inline allow-list.** A `blob:` URL
    runs in the app's origin, and the app keeps its tokens in `localStorage`.
  - Other types are download-only.
- "Open in new tab" and the mobile approver use a **short-lived signed URL** (section 5.5).

### 5.4 Sync semantics (bytes vs metadata)

```mermaid
sequenceDiagram
  participant UI as Device A (UI)
  participant L as Dexie (attachments + attachmentBlobs + outbox)
  participant S as Backend (/sync + /attachments)
  participant O as Object store (VM disk → S3-compatible)
  participant B as Device B
  UI->>L: ingest → blob 'staged' → Ctrl+A → meta row + outbox (one tx), blob 'pending'
  Note over L,S: syncNow(): pushOutbox() first (books before files)
  L->>S: POST /sync/push (attachments meta rides change_log like any doc)
  L->>S: POST /attachments/check {hashes} → {missing}
  L->>S: PUT /attachments/blob/:sha256 (stream, ≤25 MB, time-boxed per tick)
  S->>O: stream to temp → verify SHA-256 → put t/<tenant>/<sha256>
  S-->>L: 201 → blob state 'uploaded'
  B->>S: GET /sync/pull → meta row arrives
  B->>S: GET /attachments/blob/:sha256 (lazy on view; prefetch recent on Wi-Fi/Electron)
  S-->>B: 200 bytes | 404 {pending:true} → "Uploading from device A"
```

- **Two queues, one order of importance.** The existing outbox carries metadata. A new
  `pushBlobs()` runs after it inside `syncNow()` (`src/sync/sync.ts:583`), with a per-tick
  byte budget so a 20 MB scan on a slow link never delays the books.
- **A metadata row can arrive before its bytes.** That is normal and shown as pending. It is
  never treated as an error.
- **Idempotent and deduped.** Uploads are idempotent by SHA-256. The same file attached to
  ten vouchers is stored once per tenant.
- **Concurrent attach and remove** are separate rows, so they never collide. A remove and a
  caption edit on the same row resolve by the existing last-write-wins. That is acceptable
  because removal is soft.
- **Entitlement.** Uploads follow the `/sync/push` rule: **402** when `canSync` is false, and
  blobs stay `pending` locally. **Downloads always work**, so a lapsed tenant's evidence is
  never stranded (the same principle as pull in `backend/src/sync.js`).
- **Restore caveat.** `importCompany` does not emit deletes to the outbox (known limitation,
  HANDOFF §7). A restore that drops attachment rows will not propagate those drops. Document
  this, and do not paper over it.

### 5.5 Backend: tables, endpoints, storage adapter

New migration `backend/sql/007_attachments.sql`:

```sql
CREATE TABLE IF NOT EXISTS attachment_blob (
  tenant_id    uuid NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
  sha256       text NOT NULL CHECK (sha256 ~ '^[0-9a-f]{64}$'),
  size_bytes   bigint NOT NULL,
  mime         text NOT NULL,                  -- server-sniffed
  storage_key  text NOT NULL,                  -- t/<tenant>/<sha[0:2]>/<sha256>
  scan_status  text NOT NULL DEFAULT 'skipped',-- skipped | pending | clean | infected
  uploaded_by  uuid,
  uploaded_at  timestamptz NOT NULL DEFAULT now(),
  legal_hold   boolean NOT NULL DEFAULT false,
  purge_after  date,                           -- computed by the retention job, never by a client
  purged_at    timestamptz,
  PRIMARY KEY (tenant_id, sha256)
);
ALTER TABLE attachment_blob ENABLE ROW LEVEL SECURITY;
CREATE POLICY attachment_blob_isolation ON attachment_blob
  USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
ALTER TABLE plan ADD COLUMN IF NOT EXISTS max_storage_mb integer;   -- NULL = unlimited; ENFORCED at upload
```

Attachment **metadata needs no new server table.** It is `tbl='attachments'` in `change_log`,
so pull, REST `/v1/attachments`, SSE nudges and webhooks all work unchanged. A materialized
table can come later with the P4+ report materializer.

Endpoints (a new `backend/src/attachments.js` plugin, registered in `server.js`; add
`/attachments` to `API_PREFIXES` so it never falls through to the SPA):

| Method + path | Auth | Behaviour |
|---|---|---|
| `POST /attachments/check` `{hashes[]}` | Bearer | Returns `{missing[]}` for this tenant only. |
| `PUT /attachments/blob/:sha256` | Bearer, `canSync` | Route-level `bodyLimit` of 25 MB. `Content-Length` is required. The server checks the **quota before reading** and streams to a temp file while hashing. **On hash mismatch it returns 422** and deletes the temp file. It sniffs the type against the allow-list, then calls `store.put` and inserts `attachment_blob`. It never accepts a client-claimed MIME. |
| `GET /attachments/blob/:sha256` | Bearer | Streams the bytes. Sends `X-Content-Type-Options: nosniff` and `Content-Disposition: attachment` unless the type is PDF or an image. |
| `POST /attachments/blob/:sha256/link` `{disposition, ttl≤600}` | Bearer | Returns a signed URL: `HMAC(secret, tenant‖sha‖exp‖disposition)`, single blob, ≤10 min. Used for new-tab view and mobile approvals. It avoids putting the JWT in the query string, the way `/sync/events?access=` does today. |
| `GET /attachments/s/:token` | Signed token | Serves exactly one blob and expires. |
| `GET /v1/attachments/:id/content` | API key `read` | Binary for integrations. |
| `GET /attachments/usage` | Bearer | Bytes used and the plan cap, for U8. |

`backend/src/blobstore.js` is the **storage adapter**, behind one interface: `put`, `get`
(stream), `head`, `delete`.
- `FsBlobStore(root)` ships first (VM disk, `/home/ubuntu/comhard/blobs`).
- `S3BlobStore({endpoint, bucket, region, keys})` works with any S3-compatible endpoint:
  Comsky Cloud, AWS Mumbai, or later Comsky Drive's backing store if it exposes one.
- Switching is a config change plus a copy job, not a redesign.
- **Direct-to-S3 presigned uploads** are a later optimisation. Proxying through Node keeps
  hash verification and quota enforcement in one place, which is the right first trade.

**Jobs** (cron next to `backup.sh`):
- **Back up the blob store off-box nightly.** Use Comsky Backup or `rclone` to a second India
  location. This is mandatory before real customers use it, because of the "daily, servers
  in India" rule.
- **Retention and GC.** A blob is purgeable only when all of these hold:
  - no live (non-removed) `attachments` row references it;
  - `purge_after < today`;
  - `legal_hold` is false.

  `purge_after` defaults to **end of the FY of the owner's date + 8 years**, which covers the
  Companies Act 8 FY rule and the GST 72-months-after-annual-return rule. The tenant can
  extend it but not shorten it below the default.
- **Optional scan worker** (Phase 4): ClamAV `clamd` marks `scan_status` and quarantines
  infected files. RAM is the constraint: the official docs say **3 GiB minimum, 4 GiB
  preferred**, and memory briefly doubles on signature reload
  ([ClamAV docs](https://docs.clamav.net/manual/Installing/Docker.html),
  [Arch wiki](https://wiki.archlinux.org/title/ClamAV)). The VM's RAM is unknown.

### 5.6 Tenant isolation and authorization

- Every blob row is RLS-scoped. Storage keys are prefixed `t/<tenant>/`. Handlers resolve the
  key **from the RLS-filtered row**, never from client input.
- **Dedup is per tenant only.** Cross-tenant dedup would turn "already have it" into an
  oracle: tenant A could learn whether tenant B holds a given file.
- Read access today is **tenant-wide**. That matches the current server model, where RBAC is
  planned but not built (ARCHITECTURE §7). `sensitive` is enforced client-side only until
  server RBAC lands, and that must be stated in the UI copy and in the trust map.
- RLS is `ENABLE`, not `FORCE` (HANDOFF trust map), so the owner role bypasses it. Upload
  and download must use `withTenant()` on the app pool, never `ownerQ`.

### 5.7 Offline behaviour and local quotas

- **Browser quotas** ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria)):

  | Browser | Per-origin limit |
  |---|---|
  | Chrome/Edge | Up to 60% of disk |
  | Firefox, best-effort | min(10% of disk, 10 GiB group limit) |
  | Firefox, persisted | 50% of disk |
  | Safari 17+ | About 60% of disk for browser apps; about 15% for other WebKit-embedding apps |

  Eviction is all-or-nothing per origin, LRU, and skips persisted origins. **Safari ITP
  deletes script-written storage after 7 days without user interaction.** An unsynced
  attachment on Safari is therefore at real risk. Upload aggressively, and warn when
  `persistState() !== 'persisted'` (`src/data/persist.ts`).
- **Local budget.** Default 500 MB per device, shown with `navigator.storage.estimate()`.
  Only `uploaded` or `cached` blobs are LRU-evicted. Thumbnails and metadata stay.
  **`staged` and `pending` blobs are never evicted.** Catch `QuotaExceededError` and say
  which files did not fit.
- **Electron** uses the same IndexedDB under the pinned `userData` directory. A native
  file-store option via IPC (`userData/attachments/<sha>`) can come later. It is not needed
  for v1.
- **Offline, everything works except** provider pickers, viewing a file whose bytes were
  never downloaded (shows "available when online"), and signed links.

### 5.8 Audit trail

- Every attach, remove and caption edit is an outbox row with `user` and `operator` stamps.
  It appears in the existing Edit Log and in the server `change_log`, which ARCHITECTURE §8
  makes tamper-evident when the hash chain lands.
- **Removal is always soft** (`removedAt`, `removedBy`, `removedReason`). The UI hides
  removed files by default and the Edit Log shows them. This is the Companies (Accounts)
  Rules audit-trail stance applied to evidence.
- The SHA-256 in the metadata makes a swapped file detectable: same name, different hash.
  The audit pack's index carries the hashes.

### 5.9 Printing, exporting, backup

- **Voucher print** (`src/ui/print/PrintVoucher.tsx`, `PagedDoc.tsx`) gets an opt-in
  "Print attachments" toggle. Images are appended as annexure pages. PDFs are listed by name
  and hash. Rasterising PDFs into the print needs pdf.js (Apache-2.0) and is a later
  optional step.
- **Audit pack.** A zip containing the existing register export, the files, and
  `index.csv` (`voucher no, date, type, party, file, sha256`). A small zip library is
  needed, for example fflate (MIT), as a **new npm dependency**.
- **Backup format.** Keep `tallyweb-backup-v2` JSON. Metadata rides in it automatically, with
  a flag `attachmentsIncluded: false`. Add an optional companion `.zip` (`backup.json` plus
  `blobs/<sha256>`). On restore, missing blobs show "file not in this backup, fetch from
  cloud". No change to the format tag.

### 5.10 Provider integrations (phase 3), pick and copy

**Design principle: provider tokens stay in the browser.**
- The backend never stores Google or Microsoft refresh tokens.
- Bytes are downloaded client-side with a short-lived token and fed into the **same ingest
  pipeline** as a local file.
- Result: no server-side OAuth secrets, no restricted scopes, and no CASA.

- **Google (web):**
  1. The Google Identity Services token client requests `drive.file`.
  2. The Picker opens with `setAppId(<project number>)` and an API key restricted to the app
     origin.
  3. On pick, the client calls Drive `files.get?alt=media`. Google-native Docs and Sheets
     need `files.export` to PDF.
  4. The client stores `source:{via:'gdrive', providerId, webUrl}`.
- **Google (Electron):** a Desktop-app OAuth client with a loopback redirect, and the
  system-browser Picker flow (`drive.file` only, per the Picker overview).
- **OneDrive / SharePoint (web):**
  1. MSAL.js popup opens Picker v8 in a popup.
  2. Token scopes are `OneDrive.ReadOnly` for personal accounts, or SharePoint
     `MyFiles.Read` for work and school accounts.
  3. A separate Graph or SharePoint read fetches
     `@sharePoint.endpoint/drives/{driveId}/items/{id}/content`, because picker tokens
     cannot be reused for Graph.
- **"Save to Drive"** (exporting an audit pack to the user's Drive) fits `drive.file`. It is a
  later nicety.
- **Link-only** (phase 1): store the URL and recognise known hosts (`drive.google.com`,
  `docs.google.com`, `onedrive.live.com`, `1drv.ms`, `*.sharepoint.com`). Label it
  **"Linked: not stored or backed up by Comsky Books"**. Never fetch or scrape public Drive
  links server-side.

### 5.11 Comsky Drive seam

Define one interface the picker UI calls:

```ts
interface DocumentProvider {
  id: 'gdrive' | 'onedrive' | 'comsky-drive';
  pick(): Promise<{ name, mime, bytes: Blob, providerId, webUrl }[]>;
}
```

Comsky Drive becomes a third implementation once it exposes an API and shares identity with
Books (today Books has its own auth in `backend/src/auth.js`).

Another option is for Drive to *adopt* Books' attachment store, with the same Comsky Cloud
bucket layout. In that case "attachments" are simply Drive files with an owner reference.
That is an owner-level product decision (section 10).

---

## 6. Security, privacy and legal notes

### 6.1 Security

- **TLS first.** Without it, attachments travel in clear: PAN and Aadhaar copies, cancelled
  cheques, salary slips. Recommendation:
  - **Do not enable cloud upload of attachments for real customers on the HTTP deploy.**
    Local attach and links can ship on HTTP.
  - TLS is Phase B item 1 in HANDOFF §2. `books.comhard.in:80/443` belong to another app, so
  it would be a DNS-01 certificate on `:1036`.
- **Same-origin user content is the biggest web risk.**
  - The SPA, API and any served file share `books.comhard.in:1036`, and access and refresh
    tokens live in `localStorage`.
  - An uploaded HTML or SVG served inline, or turned into a `blob:` URL, would run with
    access to those tokens.
  - Controls: type allow-list with magic-byte sniffing (client **and** server), `nosniff`,
    `Content-Disposition: attachment` for everything except PDF and images, and no `blob:`
    URLs for non-allow-listed types.
  - Later: serve user content from a **separate origin** (for example a `files.` host).
  - Test whether a `Content-Security-Policy: sandbox` header breaks Chrome's PDF viewer
    before adding it. This was not verified.
- **Malware.** The allow-list is the first wall. ClamAV is the second, and it needs RAM.
  Infected files are quarantined (`scan_status`), never silently deleted. They are evidence
  too.
- **Integrity.** The server recomputes SHA-256 on upload, and the content address makes
  tampering visible.
- **Quotas.** They are enforced server-side before bytes are read (`Content-Length`) and
  re-checked after. A full VM disk would take down sync for every tenant.
- **At rest.** VM disk encryption status is unknown (open question). On S3, use
  server-side encryption. Per-tenant envelope keys (ARCHITECTURE §8) are a later step; they
  also enable crypto-shredding for erasure.

### 6.2 DPDP Act 2023 / Rules 2025

- **Roles.** For customers' party and employee data inside attachments, the tenant is the
  **Data Fiduciary** and Comhard is a **Data Processor**. Contracts (terms or DPA) must say
  so. For Comhard's own users, Comhard is the Fiduciary.
- **Security safeguards and logs.** Encryption in transit (TLS), access control, and access
  logs kept for at least one year. Log downloads and signed-link issuance as audit events.
- **Erasure vs retention.**
  - Books evidence is retained under GST, Companies Act and Income-tax law, which overrides
    erasure.
  - Documents with no statutory basis can be hard-deleted on a verified request. Examples:
    an ex-employee's Aadhaar copy after the statutory payroll period, or a lead's brochure.
  - The deletion propagates as an erasure tombstone through the outbox so devices evict local
    copies. This is the honest local-first limit already stated in ARCHITECTURE §8.
- **Sensitive identity documents.**
  - Default the `sensitive` tag for the `pan`, `aadhaar`, `bank` and `cheque` tags and for
    employee attachments.
  - Show a hint to prefer **masked Aadhaar** copies. UIDAI rules on storing Aadhaar copies
    were **not researched here** and need counsel.
- **Cross-border.**
  - Storage must be in India for the Companies (Accounts) Rules regardless of DPDP.
  - Cloud OCR or LLM processing (section 9) may run outside India. That needs tenant opt-in
    and a DPA.
  - ARCHITECTURE §8 describes DPDP's cross-border model as a negative list. That was not
    re-verified here.
- **Timing.** The main obligations bite around **May 2027**. Building to them now is cheaper
  than retrofitting.

### 6.3 Statutory consequences for product rules

- **Link-only attachments are not evidence we hold.** A Companies Act customer relying only
  on links would not meet "daily backup on servers in India" for those papers. The UI must
  say so, and "pick and copy" (option E) should be the default provider behaviour.
- **Retention default:** 8 years from the end of the owner's FY. Removal is soft, and purge
  happens only after retention.
- **Produce on demand (CGST Rule 57):** this is the audit-pack export.

### 6.4 Google and Microsoft policy

- Google: comply with the Google API Services User Data Policy. **Whether its Limited Use
  terms apply to `drive.file` data was not verified.** Brand verification needs a homepage
  and privacy policy on a Search-Console-verified domain.
- The consent screen will show the product name (Comsky Books) and the domain. The legal
  entity in the privacy policy is Comhard Technologies Pvt Ltd, per the CLAUDE.md branding
  rule.
- Microsoft: the publisher domain must match the Partner Center email domain.

---

## 7. What can be built without credentials, and what cannot

| Capability | Needs external credentials or contract? | Notes |
|---|---|---|
| Attach file, paste, drag, camera (web), view, remove, captions | **No** | Phase 1 |
| Link-only attachments (Drive / OneDrive URLs) | **No** | Phase 1. Clearly labelled not backed up. |
| Vouchers-without-documents report, Day Book glyph, audit pack, backup zip | **No** | Phase 1 |
| Cloud sync of blobs to the **VM disk** | **No** (owner's VM SSH, which the owner already supplies for deploys) | Phase 2. Production use needs TLS plus off-box backup. |
| Cloud sync to **Comsky Cloud S3 / AWS Mumbai** | **Yes**: bucket plus access keys (and pricing for Comsky Cloud) | Phase 2b. Same adapter. |
| E-invoice signed-QR decode, PDF text-layer extraction to draft | **No** | Phase 5a. The NIC public key for signature verification is published (per [Masters India](https://www.mastersindia.co/gst/signed-qr-code-e-invoicing-system/), in NIC's QR procedure document); it is not a credential. Decoding works without verification. |
| Google Drive picker / copy | **Yes**: Google Cloud project, OAuth client, API key, consent screen, Search-Console-verified domain, brand verification, **HTTPS** | Phase 3. No CASA if `drive.file` only. $0. |
| OneDrive / SharePoint picker / copy | **Yes**: Entra app registration, SPA redirect on **HTTPS**, publisher verification (Partner One ID); customers' admins may need to consent | Phase 3. $0 registration (Partner Center fees not verified). |
| ClamAV scanning | No credentials; **needs ~3–4 GiB RAM** | Phase 4 |
| Cloud OCR or LLM extraction | **Yes**: vendor account, API key, DPA, tenant consent | Phase 5b |
| Email-in document inbox | **Yes**: DNS (MX) for a mail subdomain plus an inbound mail service or server | Phase 6 |
| Comsky Drive integration | **Blocked**: the product does not exist yet | Phase 6+ |

---

## 8. Phased plan

| Phase | Scope | Effort (eng-days) | Depends on |
|---|---|---|---|
| **0. Prerequisites** | TLS on a real domain (HANDOFF Phase B #1). Decide the blob storage location. Free disk and RAM numbers for the VM. | Tracked in Phase B | Owner |
| **1. Local MVP** (ships on the current HTTP deploy) | Dexie v8 (`attachments`, `attachmentBlobs`). Ingest pipeline (sniff, downscale, hash with `sha256Pure` fallback). `Alt+U` panel on vouchers and masters with draft-id staging. Clipboard paste, drag-drop, camera input. Viewer. Link-only. `writeGuard` rules. Undo origin tag. Day Book glyph. **Vouchers without Supporting Documents** report (Screen union, router, crumb, MenuScreen, CommandPalette, GoTo, per the new-screen checklist). Audit pack zip. Backup companion zip. `?` sheet. Engine assertions (sniffing, hashing, staging, eviction rules). | **8–11** | — |
| **2. Cloud sync** | `007_attachments.sql`. `attachments.js` endpoints with streaming upload and hash verify. `blobstore.js` (FS then S3). Plan `max_storage_mb` enforced. Signed links. Client `pushBlobs()` with byte budget, lazy fetch, LRU eviction, pending states. Usage screen. Admin-portal storage column. Nightly **off-box blob backup**. Retention/GC job (dry-run first). Logo migration off base64. | **10–14** | Phase 1. TLS before real customers. |
| **3. Google Drive + OneDrive** | Provider seam. GIS + Picker (web). Desktop-loopback flow (Electron). MSAL + Picker v8 (web). Google-native export to PDF. Docs/privacy page updates. | **7–10** eng + **external lead time** for brand and publisher verification (not verified; plan on 1–3 weeks) | TLS. Owner-created Google and Microsoft registrations. |
| **4. Hardening** | ClamAV worker. Separate user-content origin. `sensitive` enforcement server-side (with RBAC). DPDP erasure tombstones. Access-log retention ≥1 year. Restore drill that includes blobs. | **6–9** | RAM. RBAC work. |
| **5a. Capture, credential-free** | QR decode (zxing-wasm or jsQR, to be chosen) and JWT payload parse with optional NIC-key verify. PDF text layer (pdf.js) plus GSTIN, date and amount heuristics. **Draft Transactions** screen to a pre-filled VoucherScreen. Party matched by GSTIN. Ledgers from ghost recall. | **6–8** | Phase 1 |
| **5b. Capture, cloud extraction** | Extraction adapter (LLM vision or a document-AI vendor) with a structured JSON schema. Tenant opt-in. Usage metering as a paid add-on. | **8–12** + DPA | Phase 5a. Vendor account. |
| **6. Later** | Email-in inbox per company. "Save to Drive" export. Comsky Drive provider. Attach-to-email when invoices are emailed. | TBD | DNS/mail. Drive API. |

**MVP = Phases 1 + 2 ≈ 18–25 engineer-days (4–5 weeks).** Following the HANDOFF §4
method, Phase 1 splits cleanly into a hand-built foundation (types, Dexie, store, the
VoucherScreen hook) plus disjoint new files: panel, viewer, report, audit pack. That suits
the parallel fan-out. The VoucherScreen integration is hand-done, sequentially.

---

## 9. OCR of attached bills into draft vouchers (later phase)

**Benchmarks to beat.** Suvit and Vyapar TaxOne publish **50–80%** first-pass capture. Docs
by Ira gives 100 free pages, then paid packs. Zoho charges ₹499/month per 50 scans. The bar
is "pre-fill most fields, human confirms". **Never auto-post.** Drafts live in their own
table, not as Optional vouchers: Optional already has register semantics that would be
polluted.

| Option | Credentials | Cost | Quality on Indian GST bills | Offline | Notes |
|---|---|---|---|---|---|
| **E-invoice signed QR decode** | None | Free | Exact for the fields it carries | Yes | The QR is a JWT signed by the IRP. Its payload carries SellerGstin, BuyerGstin, DocNo, DocTyp, DocDt, TotInvVal, ItemCnt, MainHsnCode, Irn (and IrnDt) ([ClearTax](https://docs.cleartax.in/cleartax-docs/e-invoicing-api/learn-e-invoicing-api-basics)). It covers only suppliers above the e-invoicing threshold, which has been AATO **> ₹5 crore since 1 Aug 2023** ([EY](https://www.ey.com/en_in/technical/alerts-hub/2023/05/cbic-lowers-turnover-threshold-for-e-invoicing-to-inr5-crores-with-effect-from-1-august-2023)), a large share of SMB purchase bills. **Best first step.** |
| **PDF text layer + heuristics** | None | Free | Good for digital PDFs; useless for photos | Yes | pdf.js plus regexes (GSTIN pattern, dates, "Total"). |
| **Tesseract.js on device** | None | Free | Weak on tables and layouts | Yes | Fallback only. |
| **AWS Textract AnalyzeExpense** | AWS account | **$0.01/page** first 1M; 100 pages/month free for 3 months ([AWS](https://aws.amazon.com/textract/pricing/), US West price; Mumbai availability not confirmed on that page) | Generic invoices | No | |
| **Google Document AI Invoice Parser** | GCP account | ~**$0.10 per 10-page block** (secondary: [source](https://invoicedataextraction.com/blog/google-document-ai-invoice-parser-pricing); the official [pricing page](https://cloud.google.com/document-ai/pricing) could not be read) | Generic invoices | No | India region availability not verified |
| **Azure Document Intelligence (prebuilt invoice)** | Azure account | ~**$10 per 1,000 pages** (secondary: [source](https://aiproductivity.ai/pricing/azure-document-intelligence); official page showed placeholders) | Generic invoices | No | |
| **LLM vision with structured output** (for example Claude) | API key | Claude Haiku 5.5 **$0.10 / $0.50** per M input/output tokens; Claude Sonnet 5.5 **$2 / $10** (Anthropic's current model table, cached 2026-10-06; check Anthropic's pricing page before budgeting). Rough estimate: a 1-page bill image of about 1.5–2k tokens (≈ width×height/750) plus prompt and about 1k output tokens comes to **≈ $0.001/page on Haiku 5.5** and **≈ $0.015/page on Sonnet 5.5**. Validate with `count_tokens` on real bills. | Handles Indian layouts, HSN tables and GST splits well. **Must be validated on real bills before any claim.** | No | Native PDF input. A JSON schema output maps directly to `Voucher`. Batch API halves cost for non-urgent queues. India inference residency was not verified. |

**Recommended path.**
1. **Phase 5a** (QR plus PDF text) needs no credentials and gives exact data for e-invoices.
2. **Phase 5b** adds one cloud extractor behind an adapter, chosen by a bake-off on about 200
   real anonymised bills from the consultants. Measure field-level accuracy, cost per bill,
   and latency.
3. Charge it as a metered add-on, using Zoho's ₹499 per 50 scans as the price anchor.
4. Reuse **ghost recall** (CLAUDE.md Phase 20) to fill ledgers, items and cost centres from
   the matched supplier's last voucher. OCR only needs the header: party, number, date,
   amount and the tax split.

---

## 10. Open questions for the owner

1. **TLS and domain.** When does HTTPS land, and on which host? It gates every provider
   integration and cloud upload of identity documents. Should the OAuth consent screens show
   `comhard.in` or a `comsky.ai` host? Moving the app host conflicts with the CLAUDE.md
   "identity ≠ display name" rule for `books.comhard.in`.
2. **Blob storage location.** VM disk first (how much free disk? is it encrypted?), or go
   straight to **Comsky Cloud S3** (internal price? presigned URLs, CORS and versioning
   supported?), or AWS Mumbai? Where does the off-box daily backup go: Comsky Backup?
3. **Comsky Drive.** Is Drive meant to *become* the document store for Books, sharing a
   bucket layout and identity, or to be just another picker source? Who owns the timeline
   and the API?
4. **Quotas and pricing.** What storage per plan (for example Starter 2 GB, Pro 10 GB,
   Business 50 GB)? Is OCR a metered add-on? Note that this would be the first **enforced**
   plan limit.
5. **Retention and locked periods.** Is 8 years from FY end the default? May users *add*
   attachments to vouchers in a locked period? Removal there would stay refused.
6. **Sensitive documents.** Should Aadhaar or PAN card copies be allowed at all? Should
   masking be required? This needs counsel's view under DPDP and UIDAI rules.
7. **Who registers the apps?** Which Google account and Workspace owns the Cloud project?
   Which Entra tenant hosts the app? Is Comhard already in the Microsoft AI Cloud Partner
   Program (a Partner One ID is needed for publisher verification)?
8. **Link-only attachments for Companies Act clients.** Allow them, warn on them, or require
   "copy" mode?
9. **AI processing outside India.** Is it acceptable with tenant opt-in? Which vendor do you
   prefer? Can the consultants supply about 200 real bills for a bake-off?
10. **Limits.** Are 10 MB per file and 20 per object right for your users (scanned
    multi-page PDFs)? Is HEIC from iPhones required, or is convert-on-ingest acceptable?
11. **Mobile.** Is web-app camera capture enough for v1, or is a native app on the roadmap
    (see the approvals research)?
12. **VM RAM.** Can the VM spare about 3–4 GiB for ClamAV, or should scanning run elsewhere?

---

## Sources

All web sources were checked 2026-10-15.

**Comsky:** [comsky.ai](https://comsky.ai/) · [Drive](https://comsky.ai/drive) ·
[Cloud](https://comsky.ai/cloud) · [Backup](https://comsky.ai/backup).
`products.comsky.in` and `tally.comsky.in` returned no DNS record.

**Competitors:**
- Tally: [7.0 notes](https://help.tallysolutions.com/release-notes-tallyprime-7-0) ·
  [7.0 update](https://tallysolutions.com/darpan/product-update-tallyprime-7-0/) ·
  [Docs by Ira](https://help.tallysolutions.com/release-notes-docs-by-ira-tallyprime) ·
  [Ira article](https://tallysolutions.com/smart-moves/expert-insights/how-tallyprimes-latest-ai-feature-docs-by-ira-helps-businesses-reduce-manual-accounting-work/)
- [BUSY](https://busy.in/faqs/how-do-i-upload-documents-like-invoices-and-bills-on-busy-answerid-56043.md)
- Zoho: [limits](https://zstatic.zohostatic.com/books/kb/invoices/multiple-attachments.md) ·
  [Documents](https://www.zoho.com/in/books/help/documents/documents.html)
- [QuickBooks](https://quickbooks.intuit.com/learn-support/en-us/help-articles/attachments-in-quickbooks-online/00/261783)
- Xero (unofficial): [ideas](https://productideas.xero.com/forums/967115-invoices-quotes/suggestions/46678708-invoices-increase-attachment-size) ·
  [ApprovalMax](https://support.approvalmax.com/en/articles/414533-what-file-types-can-be-uploaded-via-approvalmax-capture)
- [Vyapar TaxOne](https://taxone.vyapar.com/help/articles/ocr-purchase) · [Suvit](https://help.suvit.io/articles/ocr-purchase) ·
  [LOGIC ERP](https://kb.logicerp.com/logic-erp-version-release-notes/november-2020/accounts/attach-documents-option-added-in-setup-account-voucher-and-its-sub-categories)

**Google:**
- [Drive scopes](https://developers.google.com/workspace/drive/api/guides/api-specific-auth)
- Picker: [overview](https://developers.google.com/workspace/drive/picker/guides/overview) ·
  [web](https://developers.google.com/workspace/drive/picker/guides/web-picker)
- Verification: [overview](https://support.google.com/cloud/answer/13463073) ·
  [brand requirements](https://support.google.com/cloud/answer/13464321) ·
  [OAuth origins](https://support.google.com/cloud/answer/15549257) ·
  [loopback](https://developers.google.com/identity/protocols/oauth2/resources/loopback-migration)
- CASA pricing: [Switch Labs](https://www.switchlabs.dev/post/casa-tier-2-tier-3-security-review-providers-pricing-and-the-cheapest-option) ·
  [Leviathan](https://leviathansecurity.com/programs/google-casa-cloud-application-security-assessment)

**Microsoft:**
- [File Picker v8](https://learn.microsoft.com/en-us/onedrive/developer/controls/file-pickers/)
- Publisher verification: [overview](https://learn.microsoft.com/en-ca/entra/identity-platform/publisher-verification-overview) ·
  [how-to](https://learn.microsoft.com/ga-ie/entra/identity-platform/mark-app-as-publisher-verified)
- [Redirect URIs](https://learn.microsoft.com/entra/identity-platform/reply-url)

**Storage:**
- [MDN quotas](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria)
- R2: [pricing](https://developers.cloudflare.com/r2/pricing/) ·
  [data location](https://developers.cloudflare.com/r2/reference/data-location/)
- [S3 Mumbai (reseller)](https://precisiontech.in/cloud/amazon-aws-cloud/aws-pricing/)
- MinIO: [Pigsty](https://blog.vonng.com/en/db/minio-resurrect/) ·
  [RAGFlow](https://github.com/infiniflow/ragflow/issues/13840)
- ClamAV: [docs](https://docs.clamav.net/manual/Installing/Docker.html) ·
  [Arch](https://wiki.archlinux.org/title/ClamAV)
- [Sage on links (via Cleverence)](https://www.cleverence.com/amp/articles/sage-documentation/about-file-and-link-attachments-on-records-sage-4927/)

**Law:**
- Companies (Accounts) Rules: [KNAV](https://in.knavcpa.com/insights/flash-alert-mca-amends-the-companies-accounts-rules-2014-on-back-ups-of-the-books-of-accounts-for-indian-companies) ·
  [TaxHeal](https://www.taxheal.com/companies-accounts-fourth-amendment-rules-2022.html)
- CGST: [s.36](https://www.knowyourgst.com/gstlaw/cgst-act/36-period-of-retention-of-accounts-36/) ·
  [Rule 57](https://gstzen.in/a/generation-and-maintenance-of-electronic-records-cgst-rule-57.html)
- Income-tax Act 2025: [TaxHeal](https://www.taxheal.com/?p=115206) · [myHQ](https://myhq.in/blog/maintenance-of-books-of-accounts-in-india)
- DPDP Rules: [PIB](https://static.pib.gov.in/WriteReadData/specificdocs/documents/2025/nov/doc20251117695301.pdf) ·
  [Khaitan](https://www.khaitanco.com/sites/default/files/2025-11/ERGO%20-%20Digital%20Personal%20%20Data%20Protection%20Rules%20-%2015%20November%202025.pdf) ·
  [KSK](https://ksandk.com/data-protection-and-data-privacy/data-retention-deletion-dpdp-act-india/)
- E-invoice: [threshold (EY)](https://www.ey.com/en_in/technical/alerts-hub/2023/05/cbic-lowers-turnover-threshold-for-e-invoicing-to-inr5-crores-with-effect-from-1-august-2023) ·
  QR: [ClearTax](https://docs.cleartax.in/cleartax-docs/e-invoicing-api/learn-e-invoicing-api-basics) ·
  [Masters India](https://www.mastersindia.co/gst/signed-qr-code-e-invoicing-system/)

**OCR pricing:**
- [AWS Textract](https://aws.amazon.com/textract/pricing/)
- Google Document AI: [official (unreadable)](https://cloud.google.com/document-ai/pricing) ·
  [secondary](https://invoicedataextraction.com/blog/google-document-ai-invoice-parser-pricing)
- [Azure DI (secondary)](https://aiproductivity.ai/pricing/azure-document-intelligence)
- Claude: Anthropic model and pricing table (cached 2026-10-06)

**Codebase:**
- Client data and sync: `src/types.ts` · `src/data/db.ts` · `src/sync/sync.ts` ·
  `src/data/persist.ts` · `src/core/vault.ts` · `src/polyfills.ts`
- UI: `src/ui/VoucherScreen.tsx` · `src/ui/CompanyCreate.tsx` · `src/ui/print/*`
- Backend: `backend/src/{server,sync,rest,db,realtime,tenancy}.js` ·
  `backend/sql/00{1,2,3,6}_*.sql` · `backend/deploy/backup.sh`
- Shell and docs: `electron/main.cjs` · `docs/ARCHITECTURE.md` §§7–9 ·
  `docs/HANDOFF.md` §2 · `CLAUDE.md`
