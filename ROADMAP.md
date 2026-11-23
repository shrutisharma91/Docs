# Comsky Books — Product Roadmap

Hybrid (offline + online) SaaS: a faithful TallyPrime replacement that expands beyond Tally. See [FEATURE-PARITY.md](FEATURE-PARITY.md) for the full gap list; live board in the Kanban artifact.

## Now — next 12 features (ranked)

1. **Export report (Excel/PDF/CSV/JSON)** (platform · P0/L) — Zero export today is a dealbreaker; accountants must get Excel/PDF/CSV out of every report before they trust the tool.
2. **Report printing (Alt+P / preview)** (reports · P0/L) — Reports are unprintable; printing is table-stakes for statements, registers and outstanding, and unblocks report-export work.
3. **Ctrl-F Auto Column (comparative multi-period)** (reports · P0/XL) — Comparative multi-period columns (month/quarter/year) are the single most-used Tally MIS feature and differentiate a credible replacement.
4. **F12 Configure (per-report configuration)** (reports · P0/XL) — The F12 configuration engine underpins detailed/condensed, %, scale, filter and sort — one platform investment that upgrades every report at once.
5. **Import from XML (masters & vouchers)** (platform · P0/XL) — Migration from existing Tally installs is the top adoption blocker; without masters+voucher import nobody switches.
6. **GST returns JSON/CSV export for portal** (gst · P0/L) — GSTR-1/3B portal JSON turns on-screen returns into actual filing, the daily reason Indian SMBs run the software.
7. **Security control master toggle** (admin · P0/M) — Users/roles/rights gate every multi-user, accountant-plus-staff and compliance sale; nothing exists today.
8. **Edit Log / audit trail (old-vs-new, user)** (admin · P1/L) — Edit-log with user + old/new is now mandatory under Indian company law (Rule 11(g)) and a hard procurement requirement.
9. **Batches with mfg/expiry dates** (inventory · P1/XL) — Batches with mfg/expiry unlock pharma/FMCG/food verticals — a large segment simply cannot use the product without them.
10. **Per-item valuation method selection** (inventory · P1/M) — FIFO + per-item costing method is expected accounting behaviour; WAC-only produces wrong COGS/stock value for many businesses.
11. **Price levels & price lists** (inventory · P1/XL) — Price levels/lists with auto-fill remove manual rate typing on every invoice line — a daily time sink for distributors/retail.
12. **e-Invoice IRN generation + signed QR** (gst · P1/XL) — e-Invoice IRN+QR is legally required above the turnover threshold; without it invoices are non-compliant for a growing share of firms.

## Beyond Tally — SaaS epics

These are the wedge Tally cannot match natively (no-code custom fields replace TDL; cloud enables APIs, collaboration, multi-tenant).

| Epic | Effort | Why it matters | Scope |
|--|--|--|--|
| **Open REST APIs + API keys** | XL | An integration surface is the platform moat Tally lacks natively; it makes Comsky Books the system-of-record other apps build on and drives sticky ecosystem revenue. | Versioned REST for masters/vouchers/reports, scoped API keys, OAuth2 client-credentials, rate limits, pagination, idempotency keys, OpenAPI docs + sandbox. |
| **Webhooks / event streaming** | L | Push events (voucher.created, stock.low, payment.received) let customers automate workflows and integrators react in real time — table-stakes for modern SaaS. | Event bus, subscription management, signed payloads (HMAC), retries with backoff + dead-letter, delivery logs and replay UI. |
| **Custom fields (UDF) on any object + in reports** | XL | Replaces Tally's TDL developer barrier with no-code user-defined fields, making the product self-serviceable and defensible against verticalized competitors. | UDF schema per object (ledger/item/voucher/employee), typed fields + validation, render in forms, filter/sort/column them in reports and exports, API exposure. |
| **Team collaboration (comments/mentions/tasks/approvals)** | L | Cloud-native collaboration (approve a voucher, @mention an accountant, assign follow-ups) is something desktop Tally structurally cannot offer. | Threaded comments on vouchers/masters, @mentions + notifications, task assignment, maker-checker approval workflow with audit ties. |
| **Multi-tenant secure scalable backend** | XL | The current single-account SQLite sync server cannot serve many orgs securely; a real multi-tenant backend is the prerequisite for every other SaaS epic. | Tenant isolation + row-level security, Postgres migration, horizontal scaling, encryption at rest/in transit, backups/DR, SSO/SAML, audit logging, SOC2 posture. |
| **Billing & subscriptions** | L | Directly monetizes the SaaS with recurring revenue and self-serve growth, replacing Tally's perpetual-license + TSS model. | Plan/tier management, seat + usage metering, Stripe/Razorpay, trials, dunning, invoices/GST on our own billing, entitlement gating tied to feature flags. |
| **Mobile apps (iOS/Android)** | XL | Owners want on-the-go dashboards, invoice creation and approvals; mobile is a decisive differentiator and drives daily engagement. | React Native app sharing core logic, offline-first sync, biometric login, invoice/receipt capture, dashboards, approvals + push notifications. |
| **HRMS (beyond payroll)** | XL | Expands ACV by owning the full employee lifecycle on top of existing payroll, competing with standalone HR suites. | Leave & attendance management, employee self-service portal, onboarding/offboarding, documents, expense claims, PF/ESI/PT/IT statutory built on the payroll engine. |
| **CRM / sales pipeline** | L | Connecting leads-to-invoice keeps sales and finance in one system, increasing stickiness and cross-sell versus point CRMs. | Contacts/leads/opportunities, pipeline stages, quotations flowing into sales orders/invoices, activity timeline, email integration, sales dashboards. |
| **Analytics / BI dashboards + scheduled insights** | L | Cloud data enables live, shareable dashboards and proactive alerts that a desktop ledger tool cannot, raising perceived value for owners/CFOs. | Configurable KPI dashboards, drill-through, saved views, scheduled email/WhatsApp digests, anomaly alerts, embed/share links. |

## Architecture (design before build)

- **Design doc: [`docs/ARCHITECTURE.md`](ARCHITECTURE.md)** — the multi-tenant cloud-backend
  design is written (1730 lines: tenancy/RLS, Postgres storage, sync/conflict, auth, security/DPDP,
  scale/ops, migration + SaaS-epic hooks, decision log, risk register, phased rollout). **This
  gates every online SaaS epic — build against it, don't design ad hoc.**
- Headline choices (see the doc's Decision Log): shared-schema **pool + Postgres RLS** on a
  `SET LOCAL app.tenant_id` GUC, silo-promotion for whale tenants; hybrid `jsonb`-doc + extracted
  columns over an append-only `change_log` (evolves today's `changes` table); local-first Dexie +
  outbox preserved (cloud = system of record); domain-aware conflict resolution (doc-atomic
  vouchers by `rev`, per-field masters, delete-dominance) replacing blind LWW; **server-assigned
  gap-free voucher numbers**; Argon2id + short EdDSA JWT + rotating refresh; India-region + DPDP
  crypto-shred. Rollout is phased P1→P4+ behind a versioned protocol so existing clients keep working.

## Live board

Interactive Kanban (shareable): https://claude.ai/code/artifact/206bd236-7d83-40c2-af5c-1b9ea8d31a2c
