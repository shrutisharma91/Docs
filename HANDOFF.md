# Comsky Books — Developer Handoff

Everything needed to continue this project in a **different Claude / Claude Code account**
without losing context. Start with `/CLAUDE.md` (auto-loaded), then this file.

---

## 1. How to move this to another Claude account (do this)

The project **folder + git history** is the source of truth. The previous developer's
memory in `~/.claude/projects/.../memory/` is **outside the repo and will NOT transfer** —
that is why all durable context has been copied into the repo (`CLAUDE.md`, `docs/`).

**Recommended path — push to a git remote:**
```bash
# in this folder, on the old machine/account:
git remote add origin <your-git-url>      # e.g. a private GitHub repo
git push -u origin main                    # ships full history + all commits

# on the new machine/account:
git clone <your-git-url> tallyreplica
cd tallyreplica && npm install
npm run dev                                # verify it runs
```
Then open the folder in the new account's Claude Code. **`CLAUDE.md` auto-loads** — that
plus this file rebuilds full context. (Do NOT run `/init`; a curated `CLAUDE.md` already exists.)

**If you can't use a remote:** copy the whole folder EXCEPT `node_modules/`, `dist/`,
`release/`, `server/data/` (all regenerable). Keep `.git/` so history survives. Then
`npm install` on the other side.

**Things that do NOT transfer (and what to do):**
- **`~/.claude` memory** → already folded into `CLAUDE.md` + this file. Optionally, in the
  new account, tell Claude "save the project status to memory" to re-seed its local memory.
- **The claude.ai Kanban Artifact** (board) is bound to the old account. Regenerate a fresh
  one: `python3 scripts/gen-kanban.py` (writes `docs/kanban.html`), then ask Claude to
  publish `docs/kanban.html` as an Artifact. Data lives in `docs/parity-data.json`.
- **MCP connectors** (Notion/Linear/Slack/etc.) need re-authorising in the new account
  (claude.ai connector settings, or `claude mcp`).
- **Scratchpad / workflow transcripts** are ephemeral; nothing important lives there.

## 2. Current state & resume point — Cloud SaaS (updated 2026-07-12; Phase 24 is §2a, the left menu §2b, Phase 26 §2c, Phase 27 §2d, Phase 28 §2e, Phase 29 §2f, the MCP server §2g, Phase 30 §2h)

> **State at 2026-10-25 (read this before anything below).** `main` is `1d22e84`. **Phase 30 (custom fields v2, reporting by field, stock by field, the mouse Back control; §2h) is merged into `main` and is NOT DEPLOYED.**
> The VM still runs the Phase 29 deploy of 2026-10-23 (`687c062`, §2f): nothing from Phase 30 is on it and no customer has used it. Its backend half (JavaScript only, no migration) must be deployed BEFORE or WITH its web app, and the owner
> must run the deploy smoke on the VM (§2h has the order, the rollback and the cleanup). Also merged and wired to no screen: the batch / serial / IMEI, scanning and label engines (`5aee765`, §2h). The MCP server is committed (`d7846ce`), not deployed (§2g).
> `origin` is a bare repo on the VM (`ssh://ubuntu@151.243.40.208:1306/home/ubuntu/git/comsky-books.git`) and is 7 commits behind `main`; pushing needs the owner's SSH password, which is never written anywhere.

> **If you are a fresh Claude in a new account: read this section first.** It is the latest
> state and supersedes older dates below. Two tracks live in this one repo:
> **(A) the product** — the local-first, keyboard-first TallyPrime-parity accounting SPA +
> Electron in `src/` (~73% weighted parity: 73.5% after Phase 24, 73.2% after Phase 29 (parity wave 3, built 2026-10-22, committed `2cee555`,
> **DEPLOYED 2026-10-23, web app and backend, BETA**: §2f, which holds the deploy record and rollback), no P0 missing; Phases 25, 26, 27 and 28 are
> beyond-Tally work and moved no parity row (Phase 27, the Settings window and configurable shortcuts, and Phase 28,
> Dashboard v2, are committed together as `5c94342` and their **web app is DEPLOYED (2026-10-21)**; Phase 27's
> **backend rule went live on 2026-10-23** in the same rsync as Phase 29's backend, so the VM now runs the `687c062` backend code, no longer `a0fffd6`: §2d, §2f; Phase 28 is
> front-end only: §2e; Phase 30, §2h, is beyond-Tally work too, moved no parity row, and is merged but NOT deployed); data in
> IndexedDB/Dexie v8; runs fully offline). **(B) the cloud SaaS backend** in `backend/` + admin portal in `admin/` —
> multi-tenant sync + signup/trials/subscriptions + admin. Cloud design: `docs/ARCHITECTURE.md`
> (§13 has the live phase status). This section is track B.

**Live deployment (owner's VM "Internal-VM7"):** Ubuntu 22.04, IP `151.243.40.208`, SSH port
`1306`, user `ubuntu`. Node 22 + Postgres 16. Only ports **1306 (ssh) / 1036 (app+API) / 1038
(admin)** reach this box — `books.comhard.in:80/443` is a *different* app (Mattermost).
- **Customer app:** `http://books.comhard.in:1036/comhard/books/` (obscure path via env
  `APP_PREFIX`; root `/` → 404; the API stays at the origin root). systemd `comhard-backend`.
- **Admin portal:** `http://books.comhard.in:1038` — systemd `comhard-admin`; login
  `admin@comhard.in`. HTTP only (no TLS yet).
- **SECRETS ARE NOT IN THE REPO** (correct): the VM SSH password, the admin password, and the
  DB passwords live on the box at `/home/ubuntu/comhard/.env.db` and `backend/.env`. **The
  owner (Akshay) pastes the SSH + admin passwords into the session when resuming deploy work**
  — same as how this was set up. Don't commit them.

**Built + verified this run, all committed to `main` (per-milestone commits; read
`docs/ARCHITECTURE.md` §13 "Build status"):**
- **P1** Postgres backend (Fastify + `pg`) behind the identical v1 sync wire; Argon2id + JWT
  access + rotating refresh (reuse-detection); rate limiting.
- **P2** tenancy + **Postgres RLS isolation** (`tenant`/`membership`/`company`; per-request
  `app.tenant_id` GUC via `withTenant()`); `/me`, `/tenants`, `/auth/switch`.
- **P3** server-authoritative voucher numbers (`POST /sync/next-number`, gap/dupe-free) + SSE
  realtime (`GET /sync/events?access=`). *(delta `op:'patch'` deferred — client is full-doc.)*
- **P4** scoped **API keys** + **REST `/v1/:collection`** + HMAC-signed **webhooks**.
- **SaaS** signup → **7-day trial**, plans (Starter/Pro/Business), `subscribe`/`cancel`,
  Razorpay adapter in **MOCK mode**, entitlement **402 read-only lockout** on expiry, invoices.
- **Admin** metrics / tenant search+detail / extend-trial / set-plan / cancel / **impersonate**
  (audited) + a standalone Vite portal (`admin/`).
- **Client** billing UI (`src/ui/billing/Subscription.tsx` + `TrialBanner`), same-origin sync
  default, `crypto.randomUUID` polyfill for HTTP (`src/polyfills.ts`), persistent-storage
  request + non-durable warning (`src/data/persist.ts`).
- **Ops** nightly `pg_dump` cron (`backend/deploy/backup.sh`).

**TRUST MAP — demo-grade / NOT production (carry forward, do not assume otherwise):**
- Payments are **MOCK** (no live Razorpay keys). e-Invoice IRN is mock; GST JSON is
  portal-shaped, **not filing-grade**.
- **HTTP only, no TLS.** `crypto.randomUUID` + durable IndexedDB are polyfilled/degraded on
  HTTP; the real fix is **HTTPS** (secure cookies + these APIs come free). TLS options:
  DNS-01 cert (needs a comhard.in DNS API token) served on `:1036`, or provider routes 443→VM.
- **Single VM, single Postgres.** Rate-limiter + SSE hub are **in-memory** (don't survive a
  restart or a 2nd instance). RLS is `ENABLE` (owner bypasses), not `FORCE`.
- **No CI/CD, no monitoring/alerting, no integration/e2e/load tests.** Only pure-engine unit
  tests + adversarial code reviews. **Backup restore has never been tested; no PITR.**
- Designed-but-not-built: delta `op:'patch'`, P4 server report-materializer, all P4+ scale.

**Deploy recipe** (from a machine that has the repo; `sshpass` needed, SSH askpass can flake
→ retry the rsync):
```bash
export SSHPASS='<vm-ssh-password-from-owner>'; unset SSH_ASKPASS
SSHBASE="ssh -p 1306 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o PreferredAuthentications=password -o PubkeyAuthentication=no"
# backend:
rsync -az --exclude node_modules --exclude .env -e "sshpass -e $SSHBASE" backend/ ubuntu@151.243.40.208:/home/ubuntu/comhard/backend/
sshpass -e $SSHBASE ubuntu@151.243.40.208 'cd /home/ubuntu/comhard/backend && npm run migrate && sudo systemctl restart comhard-backend'   # sudo via SUDO_ASKPASS echoing the pw
# SPA:  npm run build  → rsync dist/ → /home/ubuntu/comhard/app  (served dynamically, no restart needed)
# admin: cd admin && npm run build → rsync admin/dist → /home/ubuntu/comhard/admin/dist → restart comhard-admin
```
**Verify it's alive:** `curl http://books.comhard.in:1036/health` → `{ok:true}`;
`curl -sI http://books.comhard.in:1036/comhard/books/` → 200 html.

**Resume / next steps** — this is the **Phase B (productionization) handoff-to-engineers**
backlog, in priority order: TLS/HTTPS → live Razorpay + GST-compliant invoicing → CI/CD →
monitoring/alerting → backup **restore drill** + PITR → load/soak test → external security
review → then delta-patch / report-materializer / horizontal scale (P4+). (Track A: continue
Tally parity via the fan-out method in §4 if desired.) Staffing/handoff advice was discussed
with the owner: start with 1 senior full-stack (Node+Postgres+RLS) owner + fractional
DevOps-security + fractional QA; audit-first, don't blind-trust the AI-built code.

## 2a. Phase 24: committed and DEPLOYED (2026-10-19)

**State.** Phase 24 is the light theme plus two parity waves: 63.8% to 73.5% weighted (258 covered,
44 partial, 79 missing of 381; first published as "74%", which summed 260 covered rows; an unaudited
upper figure, see `docs/research/ROADMAP-ADDENDUM.md`).
Commits `947bdc2` (theme + wave 1) and `a0fffd6` (wave 2) are on `main` and pushed to the VM git
remote (same box as production: not an off-site copy). Gate at the end of wave 2: `npx tsc --noEmit`
0 errors, `npm run build` green, `npm run assert` 63 suites / 20,907 assertions, `npm run shots`
449 screens in light, dark and light-comfortable with zero console errors, `npm run perf` 15/15.

**Deployed on 2026-10-19 (in this order):**
1. Pre-deploy `pg_dump` to `/home/ubuntu/comhard/backups/pre-phase24-*.dump` (2.2 MB, 137 TOC
   entries; a restore has still never been tested). Rollback copies: `/home/ubuntu/comhard/backend.prev-phase23/`
   and `app.prev-phase23/`.
2. Backend rsync, `npm run migrate` (applies `backend/sql/007_sync_conflicts.sql`: `sync_conflict`,
   `sync_device`, `idx_change_doc`; RLS enabled with policies; the app role can use both tables because
   default privileges for `comhard_app` exist on the VM, set at DB setup and NOT in the sql files),
   then `systemctl restart comhard-backend`. `package.json` was unchanged, so no `npm install`.
3. SPA bundle (`dist/` rsync): live bundle hash matches the local build, all 12 font files 200.
4. Smoke tests against the live API: login, `/sync/pull`, `/sync/conflicts` (401 without auth; the
   owner gets the empty report), weak password refused on `/auth/register`, wrong current password
   refused on `/auth/change-password` (nothing changed), `/v1` read.

**Facts found while deploying:** the JWT secret on the VM is custom (96 chars); the `comhard-admin`
service is INACTIVE (the admin portal on :1038 is not running; start it only if wanted); disk 48% used.

**Still to do:** a **Windows x64 installer** was rebuilt 2026-10-19 with the Electron backup bridge
(`electron/main.cjs`, `electron/preload.cjs`; unsigned, never run on Windows, bridge verified only
against a mocked electron: run it once in a real window). **No Mac installer exists** in `release/`
(only stale 2026-08 blockmaps); a Mac build needs ad-hoc signing at minimum. The trust map in section 2
is otherwise unchanged: HTTP only, mock payments, single VM, untested restore.

## 2b. Phase 25: left menu (built, reviewed, committed `4cdac82` and DEPLOYED 2026-10-19)

**Phase 26 (§2c) replaced this tree with menu v2 (12 groups, 64 rows, a `+` per row; deployed 2026-10-20); the
108-row tree described here was the live build from 2026-10-19 to 2026-10-20, and the rail's behaviour, width, storage and
full-screen view below are unchanged.** The left rail has two views, a module **Menu** (default; 108 rows in priority order, a filter box, the
Tally key on the rows that have one) and the old **Quick access**, and it hides for a full-screen view
with `Ctrl+\` (`⌘\` on a Mac), the breadcrumb's Hide menu button or the hamburger. Front-end only: no
backend change, migration or Dexie bump, nothing synced (three per-device keys: `TW_NAV_VIEW`,
`TW_NAV_GROUPS`, `TW_NAV_COLLAPSED`), so a release is the SPA rsync in section 2 plus a new Electron
installer. **Deployed 2026-10-19**: rollback copy `/home/ubuntu/comhard/app.prev-phase25/`; live bundle `index-etLBEIuk.js`; the live URL was shot in headless Chrome (menu renders, 0 console errors). The Windows installer rebuilt earlier that day does NOT contain it. Files: (`src/ui/nav/`, `src/ui/navTree.ts`,
`src/state/navStore.ts`, `src/core/navPrefs.ts`, `src/styles/navtree.css`, the `sidenav` assert suite,
`docs/design/left-nav-spec.md`); gate: tsc 0, build green, 64 suites / 22,481 assertions, perf 15/15,
478 screens clean. Still to do: press `Ctrl+\` for real on Windows, Linux, Firefox, Safari and
in the Electron window with focus in a voucher field (only headless Chrome on macOS has been used);
update the rail-width rows of `docs/design/light-theme-spec.md` (still 186px; now 212px, 0 hidden).
Menu rows live in `navTree.ts` and are mirrored in `scripts/assert/sidenav.assert.ts`.

## 2c. Phase 26: Zoho-style UX, wave A (built and reviewed 2026-10-20; committed `c0d9aa3` and DEPLOYED 2026-10-20)

**State.** The owner's request of 2026-10-19 (Zoho-Books-style navigation and reports), built as one wave: menu v2, a
universal quick-create, a `+ New` page header, a navigation stack that stops piling up, row filters, saved and shared views,
and a Reports Center with Create New Report. It is **beyond-Tally work: no parity row moved** (the two nearest rows, `report-range-filter` and
`report-value-filter`, stay partial because the filter is on five report screens, not the main statements). It is
committed (`c0d9aa3`, with the behaviour-preserving scaffolds of waves B and D) and DEPLOYED 2026-10-20 (SPA only, no backend change;
rollback copy `/home/ubuntu/comhard/app.prev-wavea/`; live bundle `index-DA5yYonE.js`; eight wave-A recipes pass against the live URL with zero
console errors); no customer has used any of it. **Another session's work (a Dashboard rework: `src/core/plFigures.ts`,
`scripts/shots/golden.mjs`) had started to land uncommitted in the same tree by 2026-10-20 17:20, so Phase 26 was committed by
named files, never `git add -A`; that Dashboard rework is Dashboard v2 (§2e, committed in `5c94342`; `plFigures.ts` itself had already gone into `c0d9aa3`).** Spec and design: `docs/design/zoho-ux-spec.md` (1,448 lines, Appendices A to D;
it supersedes the tree of `docs/design/left-nav-spec.md`), with `docs/design/ux-analysis-2026-10.md` and
`docs/research/zoho-books-reference.md`, `zoho-dashboard.md`, `pipedrive-list-ux.md` and `wave-a2-plan.md` behind it.

**What it is, and where it lives.**
- **Menu v2** (`src/ui/navTree.ts`, `src/ui/nav/`): 12 groups, 64 rows, 26 `+`, 10 Gateway footer links; a group label opens that
  module's list, the chevron only expands. Pinned on top, Saved views between Reports and Taxes.
- **One create registry** (`src/ui/quickCreate.ts`: 28 creates, `QC_PAGES` with 33 page entries) behind the rail `+`, the page
  `+ New` / `...` menu (`src/ui/pageHeader.tsx`, `src/ui/header/`) and the universal `+` in the top bar (`src/ui/qc/`, `Alt+Q`).
  **One chord registry** (`src/core/shortcutRegistry.ts`): the `?` sheet and Favourite Shortcuts' locked keys are generated from it.
- **Navigation stack** (`src/core/navPlan.ts` pure policy, `src/ui/navigate.ts`, `src/ui/navGuard.tsx`, `src/state/dirty.ts`,
  `src/state/navGuard.ts`, `src/state/stackId.ts`): decide on real unsaved work, replace when clean, pop to a screen already in
  the stack, ask "Leave this entry?" when dirty, peeks capped at 6, crumb shows four segments.
- **Row filter** (`src/core/reportFilter.ts`, `reportFilterFields.ts`, `reportRows.ts`, `src/ui/rowFilter.ts`,
  `src/ui/header/FilterBar.tsx` and `FilterPanel.tsx`; `Alt+F12` range, `Ctrl+F12` value) on the Day Book, every voucher
  register, Ledger Vouchers and Receivables/Payables. **Saved/shared views** (`src/core/savedViews.ts`, `src/ui/reportViews/`,
  `src/ui/header/ViewsTitle.tsx`).
- **Reports Center and Create New Report** (`src/ui/reportsCenter/`, catalogue in `catalogue.ts`: 99 reports, 14 categories;
  `src/state/reportVisits.ts`). New screens `reportsCenter`, `customReport`, and `settings` (Wave B stub: `PlannedScreen`,
  `SCREEN_AREA 'admin'`, **nothing links to it**; Phase 27 (§2d) filled it in).

**Data and deploy impact.** Front-end only: no backend change, no migration, no Dexie bump. One new synced field: the optional
`Company.reportViews` (at most 40 company-wide views), written only through `saveCompany` and the write guard, so it rides the
ordinary company-row sync; per the builder an older client carries it through a pull (a mixed old/new client sync was not
reported tested). Personal views, the last-visited list, and the menu/rail state stay per device. A release is the SPA rsync of
section 2 plus a new Electron installer. (It was deployed 2026-10-20 as the SPA rsync; the deviations below are the ones to read before changing it.)

**Gate (2026-10-20).** From the integration gate's own reports, except the assertion run, which the docs pass repeated:
`npx tsc --noEmit` 0 errors; `npm run build` green; `npm run assert` **72 suites, 27,088 assertions, all
green** (re-run by the docs pass; the new or rewritten suites are `zoho-scaffold` 771, `navstack` 550, `quickcreate` 477,
`reportfilter` 227, `reportfilter-rows` 61, `reportscenter` 1,910, `savedviews-filter` 79, `crosscut` 308, `sidenav` 1,576);
`npm run perf` 15/15 gated checks (12 ungated reference rows are over their line at 250k vouchers and flagged WATCH, for example
`rpt.gst` 2,026 ms; no earlier run to compare); `npm run shots` **534 screens, three full sweeps (light compact, dark compact,
light comfortable) 534/534 with zero console errors**, and a fourth, light compact with the rail hidden, 489/534 with zero console
errors: the 45 misses are recipes that drive the rail or press `Ctrl+\` themselves, which cannot work when it is already hidden
(each passes in the first three). Mutation checks: the builders broke their own fixes on purpose (24, 29 and 21 mutants in
scratch copies; the suites caught all but a few judged equivalent or redundant, and one survivor got a new test). Everything was
checked in headless Chrome on macOS only: real `Alt+Q`, `Alt+F12`, `Ctrl+F12` and `Ctrl+\` presses on Windows, Linux, Firefox,
Safari and Electron are unchecked, and **no live TallyPrime, portal or second synced device was involved**.

**Deliberate deviations from the spec (each pinned by an assertion).** (1) The left-menu `+` is `requestNav(list, 'dest')` then the form
pushed on top, so Esc returns to the list (spec 9.2), where plain `'dest'` (spec 1.3) would drop it; the universal `+` and Favourite slots
stay plain `'dest'`. (2) Three footer labels were shortened because `visual.assert` measured them cut: "Bills & ageing...", "More
reports...", "Statutory reports...". (3) The mega menu is `min(900px, 96vw)`, not 860 ("Payment Received" and "Manufacturing Journal"
were cut). (4) Rows with a voucher create carry that voucher in `also` (keying a Sales voucher keeps Invoices lit). (5) `navTree`
`DRILL_FEATURE` gained the 22 screens that left the rail, so pins and saved views of them honour their company feature. (6) The Reports
Center draws its OWN "Create New Report" button, not the page header's actions slot (its `QC_PAGES` entry is unused by the page), and
opens a report with source `'page'` (a push), so Esc and Cancel come back to the Center. (7) The row-filter sanitiser KEEPS a condition
naming a field the report no longer has (it matches nothing) instead of dropping it: dropping one condition of an AND would silently
widen a saved view. (8) `Alt+F12` opens on the first NUMBER field (Amount), not the registry's first number-or-date. (9) `visual.assert`'s
186px-evidence threshold went 20 to 8. Crumb words are NOT renamed (the recipes match them); the tooltip carries the whole path.

**Shared files edited beyond the spec's list, all surgical.** `AppShell` keys each stacked screen by identity (`state/stackId.ts`)
because a collapse removes entries from the middle and an index key would hand a surviving screen's component to another screen;
`AppShell` reads `state/qcStore` so screens are inert while the quick-create menu is open; `state/dirty.ts` holds a SET of probes per
slot (a nested form must not hide the screen under it); `hr/leaveapply.tsx` opens with its auto-filled `days` so an untouched Leave
form is clean; `core/shortcutRegistry.ts` gained `chordOf()` so only the host spells `Alt+Q`; `reportViews/Launcher.tsx` ends with
`requestNav(target, 'goto')` so a report already open is popped to.

**Known gaps and next steps.**
- The row filter is on five report screens; every other report needs one registry entry (`core/reportFilterFields.ts`) and one
  `useRowFilter` call. Not on Trial Balance, Group Summary, Balance Sheet, P&L, Stock Summary, GST, TDS or payroll.
- A pristine voucher with a HELD party (Alt+K batch constants) still counts as dirty, so a rail click asks.
- `ui/navTree.ts` `featureOf` knows only rail rows: a report pinned from the Reports Center that has no rail row is not hidden from
  Pinned when its company feature is switched off (one-line fix: `?? entryForScreen(scr)?.needs?.[0]`).
- Reports Center keys (Space, S, N, C, E, /) are bare screen-scoped keys, so they are not in the chord registry; its Print/Export
  prints column headings blank (they are buttons) and is not offered as a report. Search reads at most 8 words.
- The Reports Center's Create New Report is a saved setup of an existing report, not a free-form builder; scheduled reports and
  advanced analytics are not built. Views to a single role, and a keyboard-only way to pin from the views dropdown (Alt+V then P
  does it), are not built.
- Wave B (`docs/design/zoho-ux-spec.md` section 8: configurable shortcut buttons and the admin Settings window for theme, accent,
  fonts and size) was reserved here; it is built, reviewed, committed (`5c94342`) and its web app deployed 2026-10-21 in Phase 27 (§2d).
- Harness, not app: a 534-screen sweep needs `--timeout 3600` (the default 600 s kills it) and another agent's `pkill` can kill the
  shared Chromes; `qc-click`'s hover step is best-effort in headless Chrome; one `sidenav-flow` run failed once in "keyboard" and
  passed on every rerun.
- **Docs still to refresh (Phase 26 is deployed):** `docs/design/light-theme-spec.md` rail-width rows (still 186px; the rail is 212px, 0 hidden).

## 2d. Phase 27: Settings window + configurable shortcuts, wave B (built and reviewed 2026-10-20; committed `5c94342`, web app DEPLOYED 2026-10-21; backend half DEPLOYED 2026-10-23 with Phase 29's backend)

**State.** The owner's last request of 2026-10-19: shortcut buttons on the right or top that anyone can change instead of
hard-coded ones, and an admin window where the workspace administrator changes theme, colour, fonts and text size. Built as
one wave: an architect pass (spec plus a scaffold, 75 suites green), two builders on disjoint files (A: Appearance and the
workspace appearance and policy panels; B: the key translation layer, the Shortcuts tab, the rail and top-bar layouts), then a
fix round after four review lenses (keyboard safety and defaults identity, rights and sync, the visual matrix, brick-proofing
and UX) confirmed 35 findings, all fixed except two minor items the fix report names as open (that list was not in the text
this docs pass was given: read the fix agent's transcript). It is **beyond-Tally work: no parity row moved** (two rows under
`beyond`, `b-settings` and `b-shortcuts`). **Committed as `5c94342` (2026-10-21, one commit with Dashboard v2) and its web
app DEPLOYED to the VM on 2026-10-21** (live bundle `index-L4gMljR9.js` and `index-iQb2mpm8.css`; rollback copy
`/home/ubuntu/comhard/app.prev-wavebd`; the live URL was driven in headless Chrome and the Settings recipes `st-fit-settings` and
`st-look-settings-150` passed with zero console errors), BETA: **no customer has used any of it.** **The backend
half was deployed on 2026-10-23** (`backend/src/conflict.js` `UNVERSIONED_PROTECTED.companies = ['ui']` and a 4-line `sync.js` edit, no migration), because the whole
`backend/` folder was rsynced for Phase 29 (§2f); the VM now runs the `687c062` backend code, no longer `a0fffd6`. The rule itself was NOT exercised on the VM (see "To check" below). Spec and design: `docs/design/settings-spec.md` (Appendix G lists what the fix round supersedes in the earlier sections
and says where the two disagree).

**Wave B and Dashboard v2 were built in one working tree and committed together as `5c94342` (241 paths), so the by-hunk split
described next was never needed.** `git status -uall` showed 235 changed paths at the final gate: 108 are wave B, 119 are
**Dashboard v2** (a different session's work: `src/ui/dash/` with 63
files, `src/core/dash*.ts`, `src/ui/modern/Dashboard.tsx`, `DashCustomise.tsx`, `dashStore.ts`, `useDashLayout.ts`, the
deleted `dashWidgets.tsx`, `src/styles/dashviz.css`, `dashwidgets.css`, `dashcustom.css`, `scripts/shots/recipes/dash*.mjs`
and the `dash-*`, `dashboard-*`, `dashindex`, `dashscale`, `plfigures` and `viz-tokens` suites), 2 are research docs, and
**five files carry edits from both** (they would have had to be taken by hunk (`git add -p`) for a one-wave commit):
`src/styles.css`, `src/types.ts`, `src/core/shortcutRegistry.ts`, `src/ui/overlays.tsx`,
`scripts/assert/visual.assert.ts` (this list said six, with `src/ui/modern/CommandPalette.tsx`; its diff is wave B only, checked
2026-10-21; Dashboard v2 is §2e). The per-file split (235 paths) was written to the session scratchpad
(`wb-final/files-split.txt`) which is ephemeral; re-derive it by path and diff content if it is gone. Wave B's files: the new
`src/ui/settings/` (16 files), `src/core/{uiPrefs,appearance,shortcutPrefs,railLayout,chordMap,settingsChord,policyWrite}.ts`,
`src/state/{uiPrefs,companyUi,policyMirror}.ts`, `src/ui/fontScale.ts`, eleven new stylesheets in `src/styles/` (`accent`,
`appearance`, `appearance-fit`, `contrast`, `fontface`, `motion`, `railtop`, `settings`, `settings-look`, `shortcuts`,
`uiscale`), `docs/design/settings-spec.md`, the suites `uiprefs`, `uiscope`, `appearance`, `shortcutprefs`,
`shortcutprefs-mac` with `scripts/assert/env-browser.ts` and their mutation lists, `scripts/gen/uiscale.mjs`, the recipes
`settings-appearance.mjs` and `settings-shortcuts.mjs`, the flows `settings-appearance-flow.mjs` and
`settings-shortcuts-flow.mjs`, the Go To golden `scripts/shots/golden/goto.capture.mjs` and `goto.ranking.json`, and edits to
`index.html`, `src/keys.ts`, `src/main.tsx`, `src/state/theme.ts`, `src/state/navStore.ts`, `src/core/access.ts`,
`src/core/syncGuard.ts`, `src/ui/{AppShell,MenuScreen,ThemeControls,VoucherConfig,VoucherScreen,forms,masters,icons,navTree,
favouritesCore,reportColumns,reportConfig,widgets}.ts*`, `src/ui/{admin,gst,header,nav,company,banking,onboarding}/` files
(font sizes, key labels and the Preferences row) and 8 stylesheets for font sizes, plus `backend/src/conflict.js` and `sync.js`.

**What it is.**
- **Settings window** (`src/ui/admin/SettingsScreen.tsx`, the shell; tabs from the registry `src/ui/settings/tabs.ts`):
  **Appearance** (a person's own look, with a live preview), **Shortcuts** (a person's own keys, right-hand buttons, top
  bar, export and import as JSON), **Workspace** (the administrator's defaults and locks, drawn as panels from
  `workspacePanels.ts`: appearance, shortcuts, policy; read-only for everyone else, which is how an operator learns why a
  setting is locked). `SCREEN_AREA settings` is **`open`**: everyone changes their own; the administrator's part is judged by
  `canAdminister` and again by the write guard on the company row. Ways in (none can be moved by a setting): `Ctrl+,` / `Cmd+,`
  (fixed; works inside a field; not behind an overlay, a gate, or on key repeat), the top-bar gear, Go To (a late command),
  the Gateway Company menu (key Q), and the left menu's Preferences row (the one group with seven rows).
- **Appearance:** theme Light / Dark / Same as my computer (a `prefers-color-scheme` listener); eight accent presets (teal,
  blue, indigo, violet, plum, berry, green, graphite), each re-valuing only the accent family; font IBM Plex Sans / System /
  Ledger (Plex Mono); text size 90 / 100 / 110 / 125 / 150 % by `--ui-k` tokens scoped to `#root` (no CSS zoom); reduce
  motion; high contrast (neutrals only). The first frame is painted from `TW_UI_FIRST` by `index.html` before any bundle
  loads. Printed paper is untouched (`--paper-font`).
- **Layers, nearest wins:** locked by the administrator, personal (`TW_UI_PERSONAL`, keyed `device|<operator>`, the operator
  only while security is on), company default (`Company.ui`), built-in. Text size, contrast and motion are defaults only,
  never locks. Keys are never lockable (they depend on a keyboard); the standard rail and top-bar layouts are.
- **Company policy:** `Company.ui` is optional and synced (no Dexie bump, no migration). The ONE writer is
  `src/state/companyUi.ts` `writeCompanyUi` (a read-modify-write of the stored row behind the write guard, with a revision
  counter planned by `src/core/policyWrite.ts`); an alteration carries the field; a device that remembers a policy an older
  app version erased keeps applying it and offers Restore (`src/state/policyMirror.ts`, `detectErased`). Export, import
  (rejected whole on the first problem) and reset (No armed) are in the Workspace policy panel.
- **Shortcuts:** one translation layer in `src/keys.ts` (pressed chord to default chord through a Proxy over the real event),
  so the ~120 key sites are unchanged; `keyLabel` and `proseKey()` print the person's key everywhere. 25 rail actions are
  rebindable plus aliases for Go To, the menu toggle, quick create and help; the default is to ADD a second key; turning
  the original off is an explicit per-row switch; the rules live in `src/core/shortcutPrefs.ts` and the catalogue in
  `src/core/shortcutRegistry.ts` (`REBINDABLE`, `RESERVED`). The right-hand rail and top bar take a layout (`railLayout.ts`:
  hide, rename, reorder; hiding never disables a key); the top bar is now a shedding group (items fold under `...`, the gear
  never moves).
- **The way back:** the four routes above; **Reset all my settings** (clears appearance, keys, buttons, top bar, the two
  legacy theme keys, `TW_NAV_VIEW` and the first-frame snapshot; asks first, No armed); **`?uireset=1`** before first paint;
  invalid imports are rejected whole; a hostile stored table cannot turn off Ctrl+K, Enter, Esc, `Ctrl+,` or F8.
- **Server** (`backend/src/conflict.js`, `sync.js`): `UNVERSIONED_PROTECTED.companies = ['ui']` keeps the current `ui` against
  a put that states no base revision (an older app version), recorded as a `protected-fields` mismatch whose wording says
  "Workspace settings change refused". It is NOT added to `PROTECTED_FIELDS` (that rule would revert a cosmetic edit from an
  up-to-date administrator one revision behind). A current client writing from a stale base is an ordinary mismatch.

**Data and deploy impact.** Front-end plus **a backend change**: two files, no migration, no new
dependency, no Dexie bump. **Deploy status (2026-10-21):** the SPA is DEPLOYED (rsync of `dist/` to `/home/ubuntu/comhard/app`,
rollback copy `/home/ubuntu/comhard/app.prev-wavebd`; bundle `index-L4gMljR9.js` and `index-iQb2mpm8.css`; the Settings and
Dashboard recipes above passed against the live URL); the backend was NOT deployed that day, so a new client ran against the
`a0fffd6` backend, which simply lacked the protection. **Backend DONE 2026-10-23:** the release was finished by Phase 29's deploy (§2f): pre-deploy `pg_dump`,
rollback copy `backend.prev-wave3`, rsync of the whole `backend/` folder from a clean export of `2cee555` (tree `687c062`), `npm run migrate` (it applied Phase 29's
`008_voucher_series.sql`; wave B itself has no migration), `systemctl restart comhard-backend`. The desktop app is a separate release: the SPA deploy does not
reach it, and an installer built from `5c94342` is not part of this deploy. (Deploying the backend first would have been the safer
order, because the protection then takes effect for older clients at once; the SPA went out first.) **To check on the live backend
(NOT done: it needs a signed-in write and no test account was created on the live host):** push a
company row that carries no `ui` and no base from a throw-away tenant, then pull; the current `ui` must still be there and the
Sync Mismatch Report must show "Workspace settings change refused". The server rule is live but unexercised; the client defence (revision, per-device mirror,
Restore) still stands in front of it.

**Gate (2026-10-20, one fixed working tree, later committed as `5c94342`; the tree was fingerprinted before and after the gate and was identical).**
`npx tsc --noEmit` 0 errors; `npm run build` green (the main chunk is 2,496 kB, a warning that is not new); `npm run assert`
**84 suites, 42,488 assertions, all green** (re-run by the docs pass: the per-suite counts in the gate's log sum to 42,488; its
summary line says 41,598, a parsing slip: summing the second word of each line reads the two-word suite name
"inventory valuation: 890" as 0); `npm run perf -- --enforce` 15/15 (13 reference-only rows are over their line and
never fail; the largest are `rpt.gst@250k` 1,163 ms against a 1,000 ms reference and the new `dash.v2.all@250k` 762 ms);
`npm run shots` **642 screens in
three sweeps (light compact, dark compact, light comfortable), each 642/642 with zero console errors** (without `--strict`;
the Appearance and Shortcuts recipes were run with `--strict` in all four scheme and density combinations: 29 and 12 screens).
A **576-capture matrix** (Settings Appearance, Shortcuts and Workspace tabs, Gateway, Day Book, Trial Balance, Balance Sheet,
Dashboard and a Sales voucher, 8 accents x 2 schemes x 90 / 100 / 125 / 150 %, compact, IBM Plex Sans, 1440x900): 576/576
captured, 0 console errors, 0 overflow or cut-label failures, 0 contrast failures (171,448 text elements; the Settings pairs'
lowest ratio is 5.36 in light and 7.86 in dark), and **one failure class, below**. The fix round also ran a 96-state light
fit matrix (4 widths x 4 sizes x 3 faces x 2 densities) and a 36-state dark subset, clean after one fix. Real-key flows in
throw-away profiles: `settings-appearance-flow` 67 checks and `settings-shortcuts-flow` 121 checks (the latter in light
compact, dark comfortable and light comfortable). Mutation lists (deliberate breakages of the new suites): `uiprefs` 31,
`uiscope` 7, `shortcutprefs` 35, `appearance` 35, `visual` 15, all killed. Defaults identity: with nothing chosen, removing
the appearance stylesheet changes no pixel on Gateway, Day Book, Balance Sheet, Dashboard or a Sales voucher in light compact
and dark comfortable, and the printed voucher's face, size and ink are identical at every size; the Go To golden of 103 queries
matches HEAD.

**One real defect, open: Dashboard at 150 % text size.** In the Receivables and Payables cards (all eight colours, both
schemes; 125 % and below are clean) the card title is drawn under the `+ New` split button and the grey "as at ... (today)"
line wraps one word per line. It is Dashboard v2's card header (`src/ui/dash/charts/ChartFrame.tsx`; `.dv-titles` has
`min-width: 0` in `src/styles/dashviz.css` while the action buttons do not shrink), exposed by wave B's text size. The
`st-fit-dashboard` recipe checks overflow, ellipsis and row pitch only, so it stays green; fix it by wrapping the actions
below the title when narrow, and give that recipe a text-over-text check. **Still open after Dashboard v2's fix round (§2e,
checked 2026-10-21): the two CSS rules are unchanged and the Dashboard gate swept at 100 %.** Evidence picture in the session scratchpad:
`wb-matrix/dark/dark-teal-150-dashboard.png` (ephemeral). Nobody edited a repo file for it.

**Known gaps and next steps.**
- **Not built:** the "My buttons" block (Favourite Shortcuts drawn as extra rail buttons, off by default; spec P2). Rail and
  top-bar layouts are global by button id, not per screen kind. No synced per-user layer (a person's choices are per device and
  move by export and import; a synced layer needs a per-user table). No free-colour accent (a validated derive-and-adjust
  picker is specified in the spec, 2.3.4). Inter is not shipped (not installed; the rupee sign cannot be proven from metadata).
- **Prose:** 47 user-facing sentences (for example "Press F2 to change the period", "Alt+D delete" hints) still name a rebindable
  key by its standard name. They stay true while the standard key works and mislead only if a person turns it off. The ceiling
  in `shortcutprefs.assert.ts` section 11b is 47: new prose must use `proseKey()` / `keyLabel()`, and the number may only fall.
- **Text size reach:** 135 inline literal `fontSize` values in rarely used screens (ApiDocs, leave balance, Form 16A, Webhooks,
  OperatorSignIn's remainder, the cheque editor and others) are unscaled; `appearance.assert` holds the count at 135 so it can
  only fall. Printed paper and the cheque editor must stay unscaled. Below about 1280 px wide at 125 to 150 % the Balance Sheet
  and Trial Balance wrap long names onto two lines (taller rows, nothing cut).
- **Accepted, not fixed:** high contrast does not deepen accent-filled buttons (5.8 to 6.4:1 white on the accent in light;
  AA, not 7:1); on a voucher the rail is taller than its column at 125 % and above, so it scrolls (keys work, Esc and Accept
  stay where they were); the first frame (`TW_UI_FIRST`) is device-wide, so for an instant before a company opens the sign-in
  gate can show the last operator's look.
- **Checked in headless Chrome on macOS only.** Real key presses on Windows, Linux, Firefox, Safari and Electron are unchecked.
  Electron 31 (Chromium 126) cannot be driven here (the binary is not installed in `node_modules`). Whether Safari and macOS
  Chrome give the page `Cmd+,` first is unverified (three other routes exist). Chromium's handling of `Ctrl+F4` in a tab is
  unverified (it is excluded as a second Close Tab spelling). Option+digit producing a glyph on a real Mac keyboard is covered
  only at event level (`shortcutprefs-mac`). With security ON, a non-administrator's company-defaults save being refused was
  exercised in a browser for the Appearance panel and by the suites and source rules for the Shortcuts panel; a signed-in
  non-admin operator was not driven through the Shortcuts panel in a browser.
- **The contrast recipe's parser reads `rgb()` only** (`scripts/shots/recipes/settings-appearance.mjs` `CONTRAST_JS`); a future
  pair that uses `color-mix()` would be measured against the wrong background. None of its eight pairs does today.
- **Shared infrastructure edited, for the reviewer:** `scripts/assert/mutations/run.mjs` (now copies `index.html` and
  `package.json`, links `node_modules`, and has `MUT_DEBUG` to expose a crash posing as a kill), `scripts/assert/uiprefs.assert.ts`
  and `env-browser.ts` (an in-memory browser environment so `theme.ts` is import-safe under Node), `src/core/uiPrefs.ts`,
  `src/ui/admin/SettingsScreen.tsx` (`useNarrow`), `src/ui/icons.tsx` and `src/styles/navtree.css` (icons and the menu filter box
  follow the text size), and surgical edits to `CommandPalette.tsx`, `navTree.ts`, `widgets.tsx`, `overlays.tsx`, `MenuScreen.tsx`,
  `OperatorSignIn.tsx`, `SyncConflicts.tsx`, `reportConfig.tsx`, `FilterPanel.tsx`, `Snippets.tsx`, `brsImport.tsx` and `index.html`.
  Two reviewed edits changed other suites on purpose: `sidenav.assert.ts` (the registry and rules files only NAME `Ctrl+\` as
  data; three buttons now own Enter and Space) and `zoho-scaffold.assert.ts` (settings rights `open`; positive checks replace
  "nothing links to Settings"), and the literal menu table in `sidenav.assert.ts` is now 65 rows (menu v2's 64 plus the
  Preferences row).
- **Docs refreshed 2026-10-21 (commit and SPA deploy):** `CLAUDE.md` Phase 27, `COMSKY_BOOKS_PRODUCT.md` §4.9 and §4.11 status
  cells, the two `beyond` notes in `docs/parity-data.json` (the `"status": "undeployed"` pill is gone), and `docs/kanban.html`
  was regenerated. **Refreshed 2026-10-23 after the backend deploy:** the "backend half NOT deployed" sentences were corrected (this section,
  `CLAUDE.md` Phase 27 and Known limitations, `COMSKY_BOOKS_PRODUCT.md` §1 and the §4.11 row, the `b-settings` note, §7 below).
  The light-theme spec's rail-width rows are still 186 px (see §2b).

## 2e. Phase 28: Dashboard v2 (built and reviewed 2026-10-20; committed `5c94342`, web app DEPLOYED 2026-10-21)

**State.** A rebuild of the Dashboard (wave A2, the Dashboard half of the owner's Zoho-style request of 2026-10-19), built as a scaffold
(contracts, registry, perf split, two suites), three lanes on disjoint files (engine and charts; board and edit mode; widgets),
a fix round after review (3 blockers, 8 groups of major findings, about 30 minors, all fixed except one recorded decision) and a
final gate. Each lane was cut off by a usage limit once and resumed by a second builder who audited the first one's files before
finishing: that audit found a vacuous source scan, stale mutation entries and three chart bugs, so do the same after any cut-off.
It is **beyond-Tally work: no parity row moved** (one new `beyond` row, `b-dashboard-v2`, which makes 40; `b-dashviews` stays as
what the Phase 24 build had live until 2026-10-21). **Committed as `5c94342` (2026-10-21, with wave B) and the SPA DEPLOYED to the
VM on 2026-10-21** (bundle `index-L4gMljR9.js` and `index-iQb2mpm8.css`; rollback copy `/home/ubuntu/comhard/app.prev-wavebd`;
the live URL was driven in headless Chrome and the recipes `dash2-default`, `dash2-widgets-all`, `dash2-edit-board` and
`dash2-period-menu` passed with zero console errors); BETA, no customer has used it. Design and
contracts, with the owner's decisions D2, D3, D4, D8, D9 and D12 and the file ownership: `docs/design/dashboard-v2-contracts.md`
(behind it `docs/research/zoho-dashboard.md` and `wave-a2-plan.md`).

**The working tree held two waves, and they were committed together as `5c94342` (see §2d).** `git status -uall` showed 241 paths
at the final gate: 120 are Dashboard v2, 116 are wave B, and **five files carry edits from both** (a one-wave commit would have taken them by hunk, `git add -p`): `src/types.ts`,
`src/styles.css`, `src/ui/overlays.tsx`, `src/core/shortcutRegistry.ts`, `scripts/assert/visual.assert.ts`. (§2d and `CLAUDE.md` once
said six: `CommandPalette.tsx` is wave B only, checked by diff on 2026-10-21.) The classifier script and file list were in the
session scratchpad and are ephemeral; re-derive by path and diff content.

**What it is, and where it lives.**
- **Engine** (pure; no UI, store, React or Dexie import, checked by `dash-scaffold`): `src/core/dashIndex.ts` (one pass over the
  voucher log, typed arrays, an accessor API), `dashBuckets.ts` (bucket plan, `asOn`), `dashScale.ts` (chart maths), `dashPeriod.ts`,
  `dashGeometry.ts`, `dashCfg.ts`, `dashKeys.ts`, and `plFigures.ts` (the P&L arithmetic shared with the Profit & Loss and Ratio
  Analysis screens; already committed in `c0d9aa3`). `dashboardData.ts` stays as the reference engine (`computeDashMetrics`); nothing
  in the app calls it, the widget suites reconcile against it.
- **Charts:** `src/ui/dash/charts/` (Columns, StackBar, RankBars, Sparkline, ChartFrame, Tip), `src/styles/dashviz.css` and
  `light/dashviz.css`; series colours are the `--viz-1..4` and `--viz-age-1..4` tokens in `src/styles.css`, whose validator output is
  recorded in `scripts/assert/viz-tokens.record.json` (re-record with `viz-tokens.record.mjs` when `--panel` or a token changes).
- **Widgets:** `src/ui/dash/widgets/` (`registry.ts` is the catalogue of 24 ids; 22 implemented, one file each, 12 pure compute
  files in `compute/`; `env.ts` is the single door that guards rights and passes and memoises lazy passes). `watchlist` and
  `highlights` are catalogued only.
- **Board and edit mode:** `src/ui/modern/Dashboard.tsx`, `src/ui/dash/` (`useBoard`, `useBoardResults` (waves), `PeriodMenu`,
  `drill`, `skeleton`, `edit/`, `gallery/`), `DashCustomise.tsx`, `dashStore.ts`, `useDashLayout.ts`, and the layout reader
  `src/core/dashboardLayout.ts`. **Deleted:** `src/ui/modern/dashWidgets.tsx` (nothing imported it).
- **Tests:** suites `dash-scaffold`, `dashboard-board`, `dashboard-custom`, `dashboard-v2`, `dashindex`, `dashscale`, `plfigures`,
  `viz-tokens`, with a mutation list each under `scripts/assert/mutations/`; recipes `dash-charts`, `dash2-widgets`,
  `dashboard-board`, `dashboard-custom`, `dashboard-fix` in `scripts/shots/recipes/`; perf rows `dash.index`, `dash.bills`,
  `dash.v2.all` (`scripts/perf/passes.dash.ts`, mirrors in `mirrors.dash.mjs`: whoever moves a mirrored line re-points its row).

**Pinned behaviour that changed on purpose** (the owner's decision D4 and the fix round; `dashboard-custom.assert.ts` was rewritten
and its header lists them; the goldens `dash-tiles` and `dash-tiles-rich` were re-recorded for them, the six `plfigures.*` goldens
are identical):
1. **A new default board.** A person with NO saved layout now gets twelve widgets: Today, Get started, Receivables, Payables, Cash
   flow, Income and expense, Cash & Bank, Top expenses, Overdue invoices, GST payable, Recent Vouchers, Stock alerts (Quick Entry
   off). Before: the Phase 24 default of tiles. A saved layout is untouched, and a widget added after it was saved arrives OFF.
2. **As-at stamps.** Balances (receivables, payables, cash, stock, TDS) carry the date they are as at (the As at switch: Today or
   Period end, per operator); the hero no longer says "FY"; the Cash flow Opening and Closing are dated.
3. **Catalogue 24, not 15.** A stored size a widget does not offer reads as its default (a saved trend at S reads L). Receivables,
   Payables and Cash are cards that keep a Small tile face; their headline no longer needs bill-wise (the split does; Overdue and
   Overdue invoices still do). Board rows are packed so no row ends in a hole.
4. **Words and drills.** The GST tile's words say "for the open period" (decision D8: words only, the period it builds is
   unchanged). Sales, Purchases and the trend open the Group Summary of Sales / Purchase Accounts, not the register (the register
   totals GST-inclusive values; the foot says "excl. GST"). The Today count reads "Posted vouchers today".
5. **A real edit mode.** F12 Customize is a board edit mode (drag with a drop marker, an Add widgets gallery, Space/arrow/Shift
   moves, per-widget settings, Cancel changes, a 24-widget limit); F2 is the Period control; the F-key column stays on the
   Dashboard.
6. **Unchanged on purpose:** the Dashboard is still refused to Data Entry operators (D2), the Cash Flow Summary report is not
   changed (D3), no Dexie bump, no new screen, no menu or shortcut change.

**Data and deploy impact.** Front-end only: **no backend change** (no file under `backend/` belongs to
Dashboard v2; the two there are wave B's), no migration, no dependency, no Dexie bump (data stays v8). The release is the SPA rsync
of §2 (rollback copy `/home/ubuntu/comhard/app.prev-wavebd`), and it was the **same build that carries wave B**, deployed on
2026-10-21; Dashboard v2 needs no backend deploy (wave B's backend rule is still pending, §2d). No Dashboard-only tree was ever
built or gated: the combined tree is the one that shipped. **Older clients:** the layout (`Company.dashboard`, and the
per-operator store) gains an optional per-item `cfg` and `v: 2` when a layout carries settings. The new reader keeps every unknown
field. **The Phase 24 reader (live until 2026-10-21) keeps an unknown widget id but writes it back as `{ id, on, size }`**, so an un-updated app
(a web tab not reloaded, an Electron installer not rebuilt) that saves the layout drops the settings of a newer build's widgets
(Top expenses' Top N and "include purchases" go back to their defaults); nothing else is lost and nothing throws. **Checked after
the deploy (2026-10-21):** `dash2-default`, `dash2-widgets-all`, `dash2-edit-board` and `dash2-period-menu` against the live URL in
headless Chrome, zero console errors. **Not reported done:** the check on a tenant with a Phase 24 saved layout (the saved layout
must be unchanged and the new widgets off), and the rest of the Dashboard recipes (`dashfix-*`, `dc-*`; `scripts/shots/README.md`)
against the live URL.

**Gate (2026-10-20 to 21, one fixed tree, later committed as `5c94342`: a content hash of all 241 paths was identical at start, before the sweeps and at the end).**
`npx tsc --noEmit` 0 errors; `npm run build` green (main chunk 2,496 kB, the old warning); `npm run assert` **84 suites, 42,488
assertions, all green** (Dashboard suites: `dash-scaffold` 417, `dashboard-board` 678, `dashboard-custom` 469, `dashboard-v2`
1,304, `dashindex` 2,730, `dashscale` 174, `plfigures` 134, `viz-tokens` 212, `quickcreate` 494); mutation lists, all killed:
`dash-scaffold` 42, `dashboard-v2` 60, `dashindex` 64, `dashscale` 36, `dashboard-board` 94, `dashboard-custom` 36, `viz-tokens` 27;
`npm run perf -- --enforce` 15/15 gated, no mirror drift; `npm run shots` **642 screens in three sweeps (light compact, dark
compact, light comfortable), 642/642 each, zero console errors with `--strict`**, including all 69 dashboard-family screens. The
earlier dark sweep that failed 100+ screens was an interrupted run, not a code defect. A before/after picture page (the Dashboard at
`HEAD fcfafaa` against the working tree, light and dark, edit mode, the gallery, table twins, an empty company) was made in the
session scratchpad (`wd-before-after.html`, ephemeral). **Bookkeeping:** the "41,598" in the Phase 27 gate's summary line is not a
transposition (§2d, `CLAUDE.md` and the product doc said it was): summing the second word of each log line reads the two-word suite
name "inventory valuation: 890" as 0, and 890 is the whole difference.

**Performance, read honestly (250,000 vouchers, Apple M4, p50 ms, pure engine time, interleaved A/B, machine load about 3.5).**
Old default board (`dash.kpis` at HEAD) **350**. New: index pass `dash.index` **64**, bills `dash.bills` 80, **everything the default
board lists (`dash.v2.all`) 717 to 720**. At 50,000: 64 against 125. So the default board costs about **twice** the old one in engine
time and about 1.8 times the plan's 400 ms target (`zoho-dashboard.md` 3.9). Other runs of `dash.v2.all` read 658 (fix round), 762
(the wave B gate's run) and 338 (the lane's own run, before the GSTR-3B was widened from one month to the open period).
Why: GST payable is default-on and builds the whole open period (the whole financial year by default, D8 corrected only its words);
that GSTR-3B wave is about 400 ms of the 717, and a filing-month build would save about 350 ms. The board computes in waves
(`useBoardResults.ts`: index widgets synchronously before the first paint, then bills, stock, GSTR-3B, vouchers, each in its own
task), so the first paint carries about 64 ms and the longest single block is the GSTR-3B wave; that is **read from the code, not
measured in a browser** (no time-to-paint measurement exists). Nothing gates `dash.v2.all` (every Dashboard row is a reference row,
decision D13 not made). **The header comment of `useBoardResults.ts` still says the whole board is "about 400 ms (index 55, bills
153, stock 124, a GSTR-3B 69)"; the harness does not reproduce it** (a comment fix, deliberately not made in a docs pass).

**Known gaps and next steps.**
- **Open defect, not fixed: the Dashboard at 150 % text size** (§2d): the card header's `.dv-titles { min-width: 0 }` and
  `.dv-actions { flex: none }` are unchanged and the Dashboard gate swept at 100 %. Fix as §2d says, then re-run the 576-capture
  matrix or at least the Dashboard at 125 and 150 % in all eight accents.
- **Deferred by decision (D12):** view widgets (a saved report as a widget: no `vw-<id>` ids, no `cfg.src`) and the Overdue preset
  wait for the Pipedrive-style grid and `core/listQuery.ts`; the Overdue invoices list is bespoke and every line is marked
  `REPLACE-WITH-LISTQUERY` (grep it). Drill `filter` (`writeReportFilter`) is not built because nothing needs it yet. Also not
  built: Account watchlist and Highlights (catalogued, no implementation), the filing-month GST card, "sort Get started first",
  "collapse empty cards", scheduled digests, anomaly alerts and share links.
- **Decided, not changed:** the Bills Receivable / Payable reports (`ui/reports/outstanding.tsx`, Phase 26's) still decide a
  bill's side by the sign, while the card uses the ledger's group, so on a book with an advance on the other side the report can
  list a row the card calls unapplied (documented in `dashboard-v2` and `dashboard-board`). A card cannot carry its as-at into a
  report, so it says what it left out ("N vouchers are dated after today and left out here; the report counts them"). AltGr-typed
  characters do not reach the gallery search box (the favourites suite forbids reading Ctrl+Alt by hand).
- **Small:** Get started's Hide is a per-device `localStorage` flag (`TW_DASH_SETUP_HIDE_<owner>`), not company data and not the
  layout; the operator-without-Administration refusal of C/X (company layout) rests on `writeProblem` and no checked-in recipe
  drives it (the admin path was driven once with a scratch recipe, then deleted); the rail-hiding rule
  `.dash.dash-editing + .btnpanel` assumes the Dashboard keeps its panel directly after `.dash`.
- **Checked in headless Chrome on macOS only.** Real touch input (the board uses pointer events and `touch-action: none`; headless
  Chrome sent mouse events), a screen-reader pass over the live region, forced-colours rendering, and real F-key and Ctrl presses on
  Windows, Linux, Firefox, Safari and Electron are unchecked. The Settings team's `--viz-on-muted` re-value in `contrast.css` has
  no high-contrast pair in `visual.assert.ts` (only the standard schemes).
- **Docs refreshed 2026-10-21 (commit and SPA deploy):** `CLAUDE.md` Phase 28, `COMSKY_BOOKS_PRODUCT.md` (the Dashboard v2 rows,
  the preamble and the Phase 28 paragraph), `docs/parity-data.json` (`"status": "undeployed"` dropped from `b-dashboard-v2`, the epic
  note updated) and `python3 scripts/gen-kanban.py` was re-run. Nothing about Dashboard v2 is pending deploy.

## 2f. Phase 29: parity wave 3 (built 2026-10-21 and 22, three of five clusters reviewed; COMMITTED as `2cee555` on 2026-10-22; DEPLOYED to the VM on 2026-10-23, web app and backend, BETA; blocker 1 (the perf mirror) is fixed in that commit)

**State.** Five clusters, built in the working tree on top of `83d9876` (the final gate counted 199 paths: 77 modified, 122 untracked) and since committed as `2cee555` (192 paths) and deployed (2026-10-23). It is the
second parity wave since Phase 17. **Parity moved 258 / 44 / 79 (73.5%) to 250 / 58 / 73 (73.2%)**: six rows up and nine DOWN from covered to partial, because the
build showed those nine had been scored on a first slice (`docs/parity-data.json`, `docs/FEATURE-PARITY.md`; `docs/research/tally-parity-recheck-2026-10.md` section (h)
lists which pending items it addressed). Six P0 rows are partial now (`report-export` joined the five). **Review: three of the five clusters (cancel and party terms, e-way and dispatch, export and share) went through an adversarial review and a fix pass (14, 6 and 17 confirmed
findings, fixed at the root; the cancel pass reported one of its 14 fixes as only partial and its open-issues text was not in the record this section was written from, so re-read that pass before trusting "all fixed"); single entry and server numbering had their builders' own verification and mutation lists and no separate review pass in the reports this section was written from, so
both are now deployed, so give those two a review. Beta: never used by a customer, checked in headless Chrome on macOS only; nothing compared with a live TallyPrime, the GST portal, the IRP, the e-way bill portal, a UPI app, WhatsApp or real Excel.**

**The five clusters, with the file map.**
1. **Cancel (Alt+X) and party terms.** A cancelled voucher is a SHELL (`Voucher.cancelled` plus `optional: true`, no lines, party or flags), so every engine excludes it
   for free; `core/voucherCancel.ts`, `src/ui/cancel/`, `sortVouchers` in the store seals it against an older client; `docsIssued` and the GSTR-1 `doc_issue` count it;
   the register (`ui/reports/cancelled.tsx`) shows Cancelled (number kept) and Deleted. Party ledger TAIL fields (credit limit, interest rate and style, default price
   level; `ui/partyTermsFields.ts`, `core/partyTerms.ts`), the credit-limit engine and the Credit Limits report (`core/creditLimit.ts`, `ui/credit/`), the Alt+Z price
   level (`core/priceLevel.ts`, `ui/priceLevel/`), the rewritten Interest Calculation report (`core/interestCalc.ts`), `ui/popupInView.ts` (a pick-list popup hanging
   from a scrolled form). Client half of the cloud rule: `sync/cancelNotice.ts`, `core/syncGuard.ts`.
2. **Single entry** for Payment, Receipt and Contra, opt-in: `core/singleEntry.ts`, `core/singleEntryWalk.ts`, `ui/vch/singleEntry.tsx`, `styles/singleEntry.css`.
3. **Export and share.** A real .xlsx writer and reader with no dependency (`core/xlsx.ts`, `xlsxRead.ts`, `zip.ts`, `inflate.ts`, `reportSheet.ts`), the mapped
   import (`core/importMap.ts`, `ui/admin/MappedImport.tsx`), the Share panel (`core/share.ts`, `ui/share/`, Alt+M), a static UPI QR (`core/upi.ts`), bulk print
   (`core/bulkPrint.ts`, `ui/print/BulkPrint.tsx`), Alt+F9 PDF (`ui/pdfOut.ts`) and the Electron bridge `electron/exportBridge.cjs` (`pdf:save`, `share:open`; `main.cjs`
   and `preload.cjs` edited).
4. **E-way bill and dispatch** (offline): `core/dispatch.ts`, `core/ewb.ts`, `ewbImport.ts`, `ewbRecord.ts`, `ewbState.ts`, `data/ewbWrite.ts`, `ui/dispatch/` (Alt+S),
   `ui/gst/ewayPanels.tsx`, `ewayRegister.tsx` (no left-rail row: the sidenav suite pins 65 rows), the INV-01 builder's DispDtls, ShipDtls and EwbDtls.
5. **Server voucher numbering**: `src/sync/numbering.ts`, `core/vchNumbering.ts`, and the BACKEND `backend/sql/008_voucher_series.sql`, `backend/src/numbering.js`,
   the `/sync/next-number` v2, `/claim` and `/release` routes in `backend/src/sync.js`, and a cloud rule in `backend/src/conflict.js` (a cancelled voucher stays cancelled
   against a write that has not seen the cancellation; the refused edit goes to the Sync Mismatch Report). **No Dexie bump, no new dependency.**

**Gate (2026-10-22, the tree as it stands, fingerprinted before and after):** `npx tsc --noEmit` 0 errors; `npm run build` green; `npm run assert` **102 suites, 50,266
assertions, all green** (18 new suites: bulkprint, cancel-fixes, dispatch, dispatch-form, einvoice-dispatch, ewb, ewb-import, exportbridge, importmap, interest, party-terms,
popup-inview, share, singleentry, upi, vchnumbering, voucher-cancel, xlsx; singleentry alone is 4,501); every mutation list killed (singleentry 48, vchnumbering 78, xlsx 71,
importmap 89, share 62, exportbridge 47, ewb 115, ewb-import 72, dispatch 50, dispatch-form 28, einvoice-dispatch 46, voucher-cancel 34, cancel-fixes 34, party-terms 37,
interest 13, syncconflict 17, upi 26, bulkprint 18, popup-inview 8); **697 screens in three sweeps** (light compact, dark compact, light comfortable), zero console errors in
2,091 captures (one screen died of ENOSPC and passed alone; 642 before this wave); new real-key flows `cancel-party-flow` 77, `single-entry-flow` 111, `eway-flow` 87,
`export-share-flow` 105, `numbering-flow` 64; 40 SQL assertions on a real Postgres 17 for migration 008. **`npm run perf -- --enforce` exits 2: see blocker 1.**

**BEFORE COMMITTING (blockers and hygiene; kept as the record: blocker 1 is fixed in `2cee555` and that commit was made by path; items 3 and 4 are standing hygiene for this shared machine).**
1. **The perf harness has one stale mirror.** `vch.altWalk` in `scripts/perf/mirrors.mjs` pins `[...st.vouchers].sort(...)` in `VoucherScreen.tsx`, but PageUp/PageDown now call
   `adjacentVoucher()` in the new `src/core/voucherWalk.ts` (`[...vouchers].sort(walkCompare)`), so the row measures a copy of code the app no longer has and `--enforce` fails.
   Re-point the entry to `src/core/voucherWalk.ts` and `altWalkPass` in `scripts/perf/passes.ts` to `walkCompare`, then re-run `npm run perf -- --enforce` on a QUIET machine:
   at load 25 to 33 `rpt.balanceSheet` failed twice (p95 374 ms at 50k against 240; 1,632 ms at 250k against 1,200) and a clean `git archive` of `83d9876` failed the same way;
   at load 13 all 15 gated rows passed (balance sheet headroom is only 1.5x at 250k, so it trips first).
2. **Do not `git add -A`.** The tree also holds other workstreams that are NOT part of this wave: `mcp/` (a separate package with its own `node_modules`), `backend/src/mcpRoute.js`,
   `docs/design/mcp-server-spec.md`, `docs/design/back-nav-spec.md`, `docs/design/custom-fields-spec.md` and about 18 new files under `docs/research/` (back-nav, batch/serial,
   custom fields, MCP, mobile, POS, tracking, UDF, Zoho reporting tags), plus `.claude/settings.local.json`. Add by explicit path.
3. **Free disk space first.** The Data volume sat at 99 to 100% through the whole gate (free space swung between about 0.5 and 1.4 GB); ENOSPC failed a sweep screen, a Chrome start,
   an app boot and a full assert and mutation run. Headless Chrome profiles grow about 110 MB per 12 minutes each. A 554 MB `com.google.GoogleUpdater` unpacker directory in the user temp dir and old scratchpad
   sweep directories are not this wave's: clear them. If space hits zero, git and Dexie writes fail for everyone.
4. Never `pkill` by pattern on this shared machine: `pkill -f vite` stopped the shared dev server on 5173 and `pkill -f user-data-dir=.*comsky-shots-` killed other builders'
   harness Chromes (both caused spurious failures in other clusters).

**DEPLOY ORDER (the owner supplied the VM secrets; recipe in section 2). Steps 1 to 4 were DONE on 2026-10-23 at 00:40 IST (the VM clock read 2026-10-22 19:10), on the owner's explicit
instruction, from commit `2cee555`; the tree that went out is `687c062`, which only adds doc edits. Step 3's signed-in smoke test was NOT done; step 5 (the Electron rebuild) is still TODO.**
Either order is safe, because an older server makes the new client number locally and flag nothing, but this order closes the cloud rule first.
1. **DONE.** Pre-deploy `pg_dump` to `/home/ubuntu/comhard/backups/pre-wave3-20261022-190958.dump` (2.2 MB, 154 TOC entries; a restore has never been tested) and a rollback copy of the
   old backend, `/home/ubuntu/comhard/backend.prev-wave3`. The backend was rsynced from a clean git export of the commit (not from the working tree, so the untracked
   `backend/src/mcpRoute.js` was NOT shipped); `package.json` was unchanged, so no `npm install`. **This backend deploy also shipped wave B's cloud rule**
   (`UNVERSIONED_PROTECTED.companies = ['ui']`, section 2d): it is live now, and the backend is the `687c062` code, no longer `a0fffd6`.
2. **DONE.** `npm run migrate` printed "migrate: applying 008_voucher_series.sql ... ok" (`voucher_series`, `voucher_number_issue`, RLS on `app.tenant_id`; idempotent, additive,
   `voucher_counter` untouched); `comhard-backend` restarted, active, `/health` answered `{ok:true}`.
3. **PARTLY DONE (unauthenticated checks only).** Against the live host: `POST /sync/next-number`, `/sync/next-number/claim` and `/sync/next-number/release` answer 401 without a token (the routes exist),
   `GET /v1/vouchers` answers 401 without a key, `GET /sync/conflicts` answers 401 without a token, and the SPA index answers 200. **NOT done:** every signed-in flow, because no test account was created on the live host and no real
   second device was used. Still to run once (the Fastify wrapper `numberCall` was only syntax-checked and source-asserted; the SQL half ran on Postgres 17): two reserves with
   the same key return the same number; a claim of a held number answers `taken`; a trial-locked tenant answers 402; and a push of an edit to a cancelled voucher from a
   stale base is set aside (Sync Mismatch Report row) and answered with the shell. Use two real devices once: the two-device proofs so far ran on simulated devices.
4. **DONE.** SPA built from the clean export (bundle `index-Dq3GTmR1.js` and `index-D20CkhkO.css`, identical to the bundle the 2026-10-22 gate ran), rollback copy `/home/ubuntu/comhard/app.prev-wave3`,
   rsynced to `/home/ubuntu/comhard/app`; the live URL `http://151.243.40.208:1036/comhard/books/` serves it (both assets 200). Eleven of 19 screens were driven in headless Chrome on the LIVE URL with
   `--strict` and zero console errors: welcome, gateway, dashboard, bulk-entry, brs-import-xlsx, cancel-reason-prompt, voucher-dispatch-panel, voucher-dispatch-filled, single-entry-start, single-entry-receipt,
   single-entry-contra. The other 8 (cancelled-voucher-view, cancelled-register, credit-limits-board, voucher-price-level-picker, interest-report, eway-register, eway-register-detail,
   print-invoice-dispatch) failed ONLY because the harness seeds those screens by importing the dev-server module `/src/state/store.ts`, which a production bundle does not contain: a harness limit,
   not an app error (the same screens passed 697/697 in three sweeps on the dev server in the gate). `comhard-admin` is still inactive.
5. **ELECTRON REBUILD (still TODO).** `electron/main.cjs`, `preload.cjs` and the new `exportBridge.cjs` changed (`pdf:save` through `webContents.printToPDF`, `share:open`). The Electron binary is
   NOT installed in the build environment, so the real PDF write and save dialog were **never run**: only a fake electron and a fake file system. Run `npm run electron:build` (Windows
   .exe and Mac .dmg) on real machines, then press Alt+F9 on a report, a voucher print preview and a bulk print, and check a share link. `appId`, the `userData` pin and `contextIsolation`
   are asserted unchanged. The Windows installer has still never run on Windows and is unsigned.
6. **DONE 2026-10-23.** Docs after the deploy: every touched parity row's note now says "deployed 2026-10-23 (BETA; authenticated cloud flows not exercised on the VM)"
   (`docs/parity-data.json`, `docs/kanban.html` regenerated), and `CLAUDE.md` (Phase 29, Phase 27's backend half, Known limitations), `COMSKY_BOOKS_PRODUCT.md` (section 4.15 and the rows it names),
   `docs/FEATURE-PARITY.md` and this section were corrected.

**Rollback (2026-10-23 deploy).** What exists on the VM: the previous web app at `/home/ubuntu/comhard/app.prev-wave3`; the previous backend code at
`/home/ubuntu/comhard/backend.prev-wave3`; and the pre-deploy database dump `/home/ubuntu/comhard/backups/pre-wave3-20261022-190958.dump` (`pg_dump`, 2.2 MB, 154 TOC entries; **a restore from it
has never been tested**). Rolling the backend back would also take wave B's `Company.ui` rule off again; migration 008 was applied to the database and nothing here records an undo for it.
The web rollback copy from the 2026-10-21 deploy was `/home/ubuntu/comhard/app.prev-wavebd`.

**What is NOT done (the honest list; each is also in its parity row).**
- **Cancel:** no un-cancel (as in Tally; Ctrl+Z straight after is the only way back, two steps for a reverse-charge purchase whose Journal goes with it), Alt+X is not wired on the
  Ledger Vouchers drill, bulk selections cancel one at a time, Tally XML import skips ISCANCELLED vouchers (counted) and export omits cancelled ones (counted). SelectPopup in
  `ui/widgets.tsx` now repositions only when its top would be clipped by a scrolled form: a reviewer should glance at other scrolled selects (voucher screens, Budgets).
- **Credit limit** covers item invoices only (no blocking mode, no Journal, Payment, Receipt or Sales-as-voucher check; Bulk Entry only warns). **Interest:** no F11 switch, grace
  period, settled-late bills or bank-date basis; rate and style on party ledgers only. **Price levels:** no dated or group lists, slabs, discount or Alt+R; items already on the voucher
  keep their rate when the level changes; Bulk Entry and the Alt+C overlay take no level or terms.
- **Single entry:** the DEFAULT stays the Dr/Cr grid (TallyPrime defaults to single entry), so the row stays partial; the Account side for Contra and Receipt is this build's
  convention, cited in `core/singleEntry.ts` and not compared with a live TallyPrime; Ctrl+H and Alt+V were pressed in headless Chrome only.
- **Export and share:** the .xlsx was checked with `unzip -t`, `xmllint` and this app's own reader, never opened in real Excel, LibreOffice or Google Sheets; `Web Share` with a
  file and the desktop `share:open` were not driven for real (the harness has neither); the WhatsApp `wa.me` rule is from vendor guides quoting a help page that could not be
  fetched; the UPI link format is cited from a third-party copy of NPCI's 2017 draft (the NPCI file could not be fetched, `tr` is deliberately not written: re-check it before
  launch); the mapped import makes ONE two-ledger voucher per row (no item invoices; `.xls` is not read); Import Bank Statement still does NOT read .xlsx (the reader exists in
  `core/xlsxRead.ts` and is not wired there); a restricted-role mapped import was asserted but not driven in the browser; JSON, XML, HTML and image export are not built.
- **E-way bill and dispatch:** the official portal documents (`docs.ewaybillgst.gov.in`) answered 403 every time, Rule 138 was read from secondary texts, the bulk file's key names
  and version tag (1.0.0421) come from ERPNext's generator and a Vayana sample (SCHEMA?), 8 rules are flagged in `EWB_UNCERTAIN` and shown on the register, nothing was uploaded to the
  portal, no QR is drawn (the payload is undocumented), the ship-to GSTIN has no known bulk-file key, a State's own intra-State limit is unknown, every purchase over Rs 50,000
  reads "Details needed" even when the supplier will generate the bill, and the Tally XML export does not carry the consignee. The party master's multiple addresses are not built.
- **Server numbering:** RCM and Rule 37 journals (`data/rcmWrite.ts`, `data/itcWrite.ts`) and bank-statement import (`ui/banking/brsImportWrite.ts`) still number from the local count (two
  lines each: `markIssuedLocally()` before the write, `claimIssuedSoon()` after); Bulk Entry, the Stock Journal, payroll and disbursement save provisional and are claimed at sync, so a
  collision there is flagged, not prevented; claims are one request each, at most 250 per sync (a batch endpoint would speed a 1,000-row block); an explicit Renumber changes the voucher
  number only (its New Ref bill and an order's Order No. keep the old name, because later receipts may already settle against it); a provisional voucher whose device never syncs again
  stays flagged elsewhere with no age limit, escalation or Data Check line, and the badge shows only on the alteration header, the undo toast, the mismatch report and the print notice
  (not in the Day Book); a tab closed between the cloud's answer and the local write burns one number (a gap, never a duplicate); Tally's Auto (manual override) numbering and multiple
  named series are not built.
- **Team members and invites: assessed, not built.** Exists: `membership(tenant, user, role)`, per-tenant RLS, `/auth/switch`, `/me`, changes already stamped with the account (two accounts
  in one tenant would sync and share one numbering ledger). Missing: any way to add a second account (`/auth/register` always makes a new tenant and trial: no invite table, accept
  flow, member list, role change or removal); any e-mail sender in `backend/` (and the site is HTTP only, so an invite link would travel in clear: TLS first); `max_users` is display-only;
  server authorisation is tenant-wide (every member reads and writes every company's change log; the per-company and per-role `cids` scope in ARCHITECTURE.md is not built, so inviting an
  accountant or auditor today gives them everything). Order and size: TLS and an e-mail provider (an owner decision: SES, Resend or Postmark, plus DNS) about 1 to 2 days; invite table,
  4 to 5 endpoints, token hashing, expiry, rate limit and `max_users` about 2 to 3 days; a Team screen about 2 days; per-company and per-role server enforcement 1 to 2 weeks and it must land
  before anyone outside the owner's firm is invited.
- **Working notes for the next builder.** Each cluster was cut off by the usage limit at least once and resumed by a builder who audited the predecessor's files first; that audit found
  39 surviving mutations in the share, importmap and exportbridge lists, a stale mutation in the cancel and e-way lists, a source scan that scanned nothing, and a flake with a real cause
  (a clash flagged the voucher before it wrote its report row). Do the same audit after any cut-off. New gotchas from this wave are in `CLAUDE.md` under Phase 29.

## 2g. MCP server: let a customer's AI tool read their own books (built and reviewed 2026-10-22 to 23; COMMITTED as `d7846ce`, NOT released, NOT deployed, never tried in a live AI client)

**State.** Own folder `mcp/` (own `package.json`, `tsconfig.json` and gate; the root `tsc` and `npm run assert` never see it) plus one backend plugin, `backend/src/mcpRoute.js`. **Zero edits to `src/`**: figures come from the
app's own engines, bundled read-only (52 files pinned by SHA-256 in `mcp/engine-lock.json`, recorded at `687c062`; read the CLAUDE.md "MCP server GOTCHAs" before editing an engine; **the lock has been stale since `1d22e84`: 9 pinned files changed, see §2h**). Package `@comhard/books-mcp` 1.1.0, private,
unpublished. Read: spec `docs/design/mcp-server-spec.md` (sections 0, 7, 8), research `docs/research/mcp-*.md`, customer guide `mcp/README.md`, operator guide `mcp/DEPLOY.md`, `mcp/CHANGELOG.md`. Two lanes. **Core:** the stdio
connector, 16 tools (14 read, 2 draft-write), three data sources (the AI gateway `/mcp/gw`, the plain `/v1` REST API, a company backup file). **Remote:** the plugin, dormant unless `MCP_ENABLED=1` (POST `/mcp`, the gateway
routes, key approval, tenant switch, kill switch, audit, four tables made by an embedded DDL at start-up). **Nothing is on the VM:** the 2026-10-23 backend deploy used a clean git export (so no `mcpRoute.js`), and
`backend/src/server.js` has no registration line (`grep mcp backend/src/server.js` is empty; checked again 2026-10-25). Now that `mcpRoute.js` is committed, the NEXT clean-export deploy of `backend/` WILL copy it to the VM: inert while nothing registers it (§2h step 1). **Review:** `mcp/CHANGELOG.md` records a three-lens round (44 findings) fixed or left open and the spec records a separate 44-finding
design review; I could not tell whether they are one set. The remote lane's fix report ends with its own open list, which was not in the record this section was written from: re-read that pass before trusting "all fixed".

**Defaults, and deviations.** Read only by default. Writes (impossible on any server yet): preview then confirm; the token is an integrity check, NOT a human approval and NOT single-use (spec D5; an earlier brief said
single-use); every draft is Optional (promote with Alt+L or Ctrl+L); ceilings 25 open drafts, 10 an hour, 50 a day, Rs 1 crore a day; replay-first idempotency stops doubles. An MCP key is a **read-scope** key approved by an
owner or administrator with an explicit company list (a Write key is refused). No ledger creation, no TDS or TCS entries, `MCP_WRITE_KINDS` has no default (first release `receipt,payment,contra,journal`), invoice drafts off
until an accountant says whether a number gap from a deleted draft is acceptable. **The owner has answered none of the spec's 17 questions; the build proceeded on its defaults** (section 8.1: TLS later and a tunnel now,
no plan gating, reads stay open after a trial, masked identifiers, package unpublished, OAuth after stage 3). **Deviations a builder made, to reverse if you disagree:** (1) a snapshot page costs 1 token, not the spec's 10
(`mcp/src/http/limits.ts`, `GATEWAY_COST.snapshot`; at 10 a first load of a 79-voucher company took 51.1 s; a 50k book still needs about 12 s at the default burst of 40, so raise `MCP_RATE_BURST`); (2) the DDL and the wave-3
tier check run in `onReady`, not at registration (`server.js` registers the plugin at line 51 and runs `await migrate()` at line 75); (3) the flag is `MCP_ENABLED` (an earlier brief said `MCP_HTTP_ENABLED`); (4) the key
is read from `X-API-Key` only, so a bearer-only client cannot use it; (5) `/mcp` routes log no query string (`backend/src/logging.js` redacts only five names).

**Run and test** (from `mcp/`; `npm ci` first; Node 20 or later, only 25.2.1 has been run):
```bash
npm run typecheck && npm test     # tsc; the 41 core suites (skips test/http*)
npm run test:remote               # the 15 remote-lane suites; http-pg (real Postgres) skips unless the pg driver and Postgres binaries exist: MCP_PG_MODULE=<pg lib/index.js> node test/http-run.mjs http-pg.test
npm run drift && npm run smoke    # engine lock, import door, secrets, brand guards; the built stdio bundle as a real process
npm run mutate                    # 201 core mutations (about 7 minutes); node test/http-mutate.mjs runs the remote lane's 133
npm run gate                      # all of it under one tree fingerprint (--quick, --skip-mutants); npm run release prints the SHA-256 pins and publishes nothing
```
**Evidence (2026-10-23, fingerprint `c057afa0fd4d6353` before and after):** mcp `tsc` 0 (root `tsc` 0); drift clean; purity scan 52 engine files, 0 impure; core 41 suites / 112,131 assertions; remote 15 / 4,153; stdio smoke 112; mutations 201
of 201 and 133 of 133 killed; `npm audit` 0. `gate --skip-mutants` ran green with the perf budgets relaxed four times (load about 10). The real-Postgres suite ran green on Postgres 14 and 17 and in the fix pass (pg 8.16.0
installed `--no-save`, then pruned) and was a SKIP in the final gate. **Never run:** any live client (Claude Desktop, Code, Cursor, VS Code, claude.ai, ChatGPT), Node 20 or 22, TLS, or the plugin inside the real backend
(`backend/node_modules` is not installed in this checkout, so `realDeps()` against the real `db.js`, `tokens.js`, `subscription.js` and `platform.js` was read, not run).

**DEPLOY ORDER (none of it done; do not set `MCP_ENABLED` early).**
1. **TLS and a hostname first**, for anything remote and before any customer key (trust map in section 2). The plugin and the connector both refuse plain HTTP to a non-loopback address. Until then: the owner's own tenant over an SSH
   tunnel, with `MCP_ALLOWED_HOSTS` listing exactly the tunnel's `localhost:PORT`.
2. **Finish what the flag needs.** The bundle exports no Postgres data layer (`createPgDataSource(backend)` or `createPgRuntime(backend)`; contract in the `mcpRoute.js` header and `mcp/DEPLOY.md`), so with `MCP_ENABLED=1` the backend
   REFUSES TO START (tested, deliberate); `MCP_WRITES_ENABLED=1` also refuses (no Postgres write store). Writes also need wave 3's numbering: deployed since 2026-10-23, never exercised signed in (section 2f).
3. **Before any customer key:** seam **R5** (`redactForRead` in `backend/src/restPolicy.js` strips four fields and still returns a legacy plaintext `vaultPin` or `pin` to any read key, with or without MCP; patch text in
   `mcp/DEPLOY.md`); seam **K3** recommended (`/v1` honours no MCP company list or expiry) or the owner's written acceptance; the isolation decision (default: reads in a separate process that holds no secret, slice R-iso,
   NOT BUILT); counsel's reading of the README wording. R6 (REST delete ghost) and S2 (refuse forged `mcp:` device ids) if cheap.
4. **Seam S1**, directly after `await app.register(restRoutes);` in `backend/src/server.js`: `await app.register((await import('./mcpRoute.js')).mcpPlugin); // Comsky MCP: dormant unless MCP_ENABLED=1`.
5. **Ship.** `cd mcp && npm ci && npm run build`, then `npm run release` for the hashes. `mcp/dist/` is git-ignored, so the clean-export rsync of `backend/` used on 2026-10-23 does NOT carry the bundle: rsync `mcp/dist/http.mjs`
   (and `worker.mjs` once one exists) to `/home/ubuntu/comhard/mcp/dist/` too, and rebuild just before pinning.
6. **Env in `backend/.env` on the VM** (never in git): `MCP_ENABLED=1`, `MCP_TOKEN_SECRET` (`openssl rand -base64 48`), `MCP_BUNDLE_SHA256`, `MCP_ALLOWED_HOSTS`, `MCP_TRUSTED_PROXIES` once TLS is in front; `JWT_SECRET` must be 32+ characters
   (the VM's is 96). Then `INSERT INTO mcp_tenant(tenant_id, allowed)` for the owner's tenant only, restart `comhard-backend`, and check `GET /mcp/gw/info` answers 401 without a key and 426 from another machine over plain HTTP.
7. **Stages** (spec 7.6): owner's tenant read only over a tunnel; drafts on that tenant; two or three design partners read only (only after step 3); partners who ask for drafts; everyone. **Quickest stop:** `POST /mcp/gw/kill`
   (owner or admin bearer), then unset `MCP_ENABLED` and restart; revoke the key in the app.

**NOT done.** Every live client (the README snippets follow vendor docs, not a run); Node 20 and 22; the Postgres data layer, the compute worker (`mcp/src/host/`) and the write commit (`write/inProcess.ts`) are not written, and
`mcp/src/httpMain.ts`, `remote-smoke`, the 250,000-voucher EXPLAIN record and the wiring smoke against the real `db.js` are not built or never run; OAuth and so claude.ai, phone and ChatGPT (a header key may work for an
organisation with Anthropic's `static_headers` beta: unverified); the `.mcpb` never built; no LICENSE, nothing on npm, the name has not cleared a trademark search; the app seams A1 to A3 (an `MCP: <key>` mark in the Edit Log, an
"AI access" approval screen, the disclosure text at key creation) are not made, so approving a key is `curl` today; auth cache, rate buckets and the kill epoch are per process (right for the single VM); three spec mutants
(`W9` commit outside the tenant lock, `W17` device id left as `api`, `W24b` ceilings counted in memory) are recorded NOT_APPLICABLE because that code is in the unbuilt half, so the production versions are mutation-tested by neither
lane; the app's `round2` rounds an exact half-paisa tie down (4.725 becomes 4.72) and drafts match it on purpose: a chartered accountant should confirm, and also the TDS section numbering after the Income-tax Act 2025; the DPDP
position of sending books to an AI vendor is for counsel.

**Hygiene.** Add by explicit path, never `git add -A`: the MCP paths are committed since `d7846ce` (`mcp/` 181 files; `node_modules`, `dist`, `.cache`, `*.tgz` and `.release` are ignored by `mcp/.gitignore`; `backend/src/mcpRoute.js`,
`docs/design/mcp-server-spec.md` and the three `docs/research/mcp-*.md`), but the tree still holds other workstreams' untracked research and design files (POS, mobile, custom-field research), which belong to them. `gate --with-ci` wipes `node_modules`. The disk ran at 98 to 100% for the whole build
(stale `cb-mcp-*`, `mcp-pg-*` and `mcp-http-*` temp dirs were cleared by hand). A mutation can look killed while its suite is already failing on the clean tree: run the suite on the unmutated tree first.

## 2h. Phase 30: custom fields v2, reporting by field, stock by field and the mouse Back control (built and reviewed 2026-10-24; MERGED into `main` as `1d22e84` on 2026-10-25; NOT DEPLOYED, BETA)

**State.** Built in an isolated worktree (`.wt-cf`, branch `cf-back-nav`, cut from the wave 3 snapshot `baa0c45`), reviewed by five lenses with sceptic verification, fixed (four fix clusters of 5, 5, 20 and 9 findings, each fixed at the root and proved by a
suite that fails on the old behaviour), squash-rebased onto `main` (`5aee765`: wave 3 as committed, the `d941298` perf-mirror fix and the merge of the batch/serial/scan/label engine branch), fixed once more after the rebase, and
FAST-FORWARDED into `main` as **`1d22e84`** on 2026-10-25 (357 files, +53,699 / -1,781). **NOT deployed: the VM runs `687c062` (§2f); no customer has used it; checked in headless Chrome on macOS only; nothing compared with a live TallyPrime, the GST portal or
the VM.** Parity is unchanged (250 / 58 / 73, 73.2%): five `beyond` rows (`b-custom-fields`, `b-entry-rules`, `b-field-reports`, `b-stock-variants`, `b-back-control`) that the board shows as BUILT, NOT DEPLOYED. `tdl-customization` stays missing (a no-code field is not TDL).
**Read first:** `docs/design/cf-back-nav-build-notes.md` (the source of this section: slices built and not built, deviations, release gates G1 and G2, deploy order, the rebase and its fix round; its sections 0 to 11 describe the tree BEFORE the rebase and its
section 12 the rebased one; the merge-gate below is newer than the "Not done" list of its section 12), then `docs/design/custom-fields-spec.md` (with the owner addendum of 2026-10-22) and `docs/design/back-nav-spec.md`. The five research notes behind them
(`docs/research/back-nav-audit.md`, `custom-fields-code-audit.md`, `tally-udf-and-competitors.md`, `zoho-custom-fields.md`, `zoho-reporting-tags.md`) are still UNTRACKED in `main` at the time of writing: commit them by explicit path.

**What it built** (no Dexie bump, no SQL migration, no dependency, no new chord, `electron/` untouched; a company with no custom field and a person who changes no setting see what they saw before, asserted by reference in `customfields-identity` and others):
- **Custom fields v2** [`b-custom-fields`, `b-entry-rules`]: fields on ledgers, stock items, voucher headers (a strip under Narration), item lines (0 to 2 inline columns by measured width, the rest on a sub-row), employees and leads; 12 types plus a lookup (masters only);
  scope, defaults, inheritance from the party or item, print switches, hide, tombstone delete, v1 conversion, starter sets, Fill blanks (batch undo), custom columns in the mapped Excel/CSV importer and Bulk Entry. **Entry rules** (Settings > Workspace): an administrator
  chooses whether a mandatory field binds an OLD voucher or master (new entries only, the default, from a date, always), per company and per field, with a consequence count. Files: `src/core/customFields.ts`, `entryPolicy.ts`, `entryPolicyWrite.ts`, `customFieldEdit.ts`,
  `customFieldColumns.ts`, `customFieldFill.ts`, `lineFields.ts`, `voucherFields.ts`; `src/ui/admin/CustomFields.tsx` and `ui/admin/cf/`; `ui/settings/panels/EntryRulesPanel.tsx`, `state/entryPolicyWriter.ts`, `entryPolicyMirror.ts`; `ui/vch/CustomLineCol.tsx`, `ui/udf.ts`.
- **Analysis by field** [`b-field-reports`]: five screens in a new LAST Reports Center category (106 reports in 15 categories; no left-rail or Gateway row on purpose; the rail has one new row, "Custom Fields": 66 rows, 27 `+` creates): Summary by field, Income and expense by field
  (ties to the Profit and Loss nett), Bills by field and Balances by field (tie to the GROUP balance), Stock by field; banded dimensions; the row filter on the Day Book, voucher registers, Ledger Vouchers and Bills offers the company's fields. Files: `core/fieldSummary.ts`,
  `fieldBills.ts`, `fieldPl.ts`, `fieldPlStock.ts`, `fieldAttribution.ts`, `fieldReportView.ts`, `fieldBands.ts`, `fieldDimension.ts`, `customFilterFields.ts`; `ui/reports/field*.tsx`, `ui/cf/useCustomFilterFields.ts`.
- **Stock by variant** [`b-stock-variants`]: a line field marked "Tracks stock" keeps stock by Colour, Size or any variant: `core/variantCore.ts`, `stockByField.ts`, `variantEntry.ts`, `stockAgeing.ts`, `byVariant` in `stockPositions`, the count group (`InventoryLine.countPart`, merged once in `stockMovements`),
  the screen `ui/reports/fieldStock.tsx` (size-run grid, godown level), a Stock Summary group mode, the variant on the voucher, Physical Stock by variant, Opening by variant, "Split Not set", Tally XML export merging. One pooled cost per item.
- **Back control** [`b-back-control`]: `planBack` (`core/navPlan.ts`), `state/trail.ts`, `requestBack` (`ui/navigate.ts`), `ui/backNav.ts`, `ui/BackButton.tsx` in the top bar; `Box.onClose` required and a Close on every dialog (`ui/boxClose.ts`, `styles/boxclose.css`); the rail's Esc button
  runs `runEscape` (`keys.ts`); the busy registry (`registerBusy`, `useBusy` in `state/dirty.ts`) so Back refuses during an Accept; the rail's Prev/Next; cursor and scroll restore in `ReportShell`.
- **Server half** (JavaScript only): `backend/src/conflict.js` (capability- and omission-keyed rules `CAP_KEPT` for `entryPolicy`, `OMIT_KEPT` for `ui`, the custom-value carry), `sync.js` (the push body moved out), new `histories.js`, `pushRun.js`, `mismatch.js`, `restWrite.js`, `rest.js` (REST keeps what an older
  writer omits); client `src/sync/caps.ts` (`UDF2_BUILD = true`) and `core/syncDiff.ts`. The server announces `udf2` in its pull answers (`SERVER_CAPS`). Smoke scripts: `scripts/smoke/customfields-deploy-smoke.mjs`, `udf-scenario.mjs`, `shipped-caps.mjs`; `scripts/pg/customfields-pg.mjs`.

**Merge record.** `git rebase --onto main wave3-wip cf-back-nav` onto `5aee765` stopped on 15 files (the plan had counted 13); the two specs were kept as the branch's versions, every other conflict was resolved with both intents kept. Re-applied BY HAND because the code moved: `backend/src/sync.js` (the branch's side taken; wave 3's
`|| needsCancelCheck(c)` lives on in `histories.js` as a named import), `backend/src/conflict.js` `classifyPush` (wave 3's rule 1b, a cancelled voucher stays cancelled, FIRST; then the custom-value rule 1c and the two company-policy rules 1d), `src/core/importMap.ts` (main's hardened version with the custom-column edits laid on it),
`VoucherScreen.tsx` (`walkVoucher` calls wave 3's `adjacentVoucher`), `forms.tsx` (`tailGroup` / `groupEnd` beside the custom flags). Suite pins that each pinned a literal the other branch had changed were re-pointed. The review of the rebased tree asked one question, "do the wave 3 fix hunks survive?", and answered yes (67 of 98 files byte-identical to `main`,
31 read hunk by hunk, the moved server code driven through the real `runPush` and `loadHistories`); it found one major finding about a harness recipe (`ledger-tail-reachable` had been loosened and would have passed even if the party-form tail regressed: it is `main`'s again) and four minor ones (five stale mutations, a coin-toss in `numbering-flow`, two stale spec status lines, the viewport leak), all fixed.

**Gate (2026-10-25, merge-gate on the exact tree, now `main`):** `npx tsc --noEmit` 0; `npm run build` green; `npm run assert` **162 suites / 168,165 assertions, all green** (all 125 suites `main` had are present, 37 new ones, 15,241 assertions); `npm run perf -- --enforce` exit 0 (15/15 gated); `npm run shots` 947/947 in each of three sweeps
(light compact, dark compact, light comfortable), zero console errors under `--strict`. Earlier gates on the pre-rebase tree: 43 new mutation lists all killed (CF-1 alone 193 mutations); the deploy smoke's SQL half on a throwaway Postgres 17 (`scripts/pg/customfields-pg.mjs`). **Never run:** the smoke on the VM, Postgres 16, Node 22, a real older shipped build against the real
server, real keys on Windows, Linux, Firefox, Safari or Electron, touch, a screen reader, a time-to-paint measurement. The perf figures at 250,000 vouchers came from a busy machine.

**DEPLOY ORDER (none of it done; the owner supplies the VM SSH and database passwords at session start and they are never written to a file; recipe in §2).** The VM runs `687c062` (§2f). Against that tree the backend differs by 12 files and `backend/package.json` and `backend/sql` are untouched (no `npm install`, no migration; `npm run migrate` is harmless).
1. **Backend first, in the same window as the web app.** `pg_dump` first (the 2026-10-23 dump is `/home/ubuntu/comhard/backups/pre-wave3-20261022-190958.dump`; a restore has never been tested), keep a rollback copy of the live `backend/`, rsync `backend/` from a CLEAN git export of `main` (never the working tree), restart `comhard-backend`, check
   `/health`. The 12 files are the seven JavaScript files of this phase (`conflict.js`, `sync.js`, `rest.js`, `histories.js`, `pushRun.js`, `mismatch.js`, `restWrite.js`) and, from other commits, `mcpRoute.js` (inert: `server.js` does not register it, so unlike the 2026-10-23 deploy it WILL be on the VM; exclude it if you want the VM to match that state) and
   `unitChain.js`, `unitKey.js` with their `.d.ts` (the batch engines' shared code; nothing in `backend/` imports them).
2. **The owner runs the deploy smoke once, before releasing any custom field to a synced company:** `node scripts/smoke/customfields-deploy-smoke.mjs --url http://books.comhard.in:1036 --yes`. It registers its own throwaway account and tenant (a 7-day trial) on the live host, walks the scenario over the real HTTP routes (a current and an older app push the same voucher; a definition deleted by
   an older editor; a REST re-post; two company-row pushes, one stating `udf2` and one not), prints PASS or FAIL per step, exits 0 only if all pass, and prints the SQL that removes its tenant (run as the owner role, tenant documents first: `change_log` has no foreign key to tenant). It refuses to run if `src/sync/caps.ts` says the build states nothing. It will be the first signed-in
   traffic on the VM; the same throwaway account could also serve the wave 3 signed-in checks that §2f step 3 lists as not done (a suggestion, not part of the smoke). Optionally run `scripts/pg/customfields-pg.mjs` against a scratch Postgres 16, never the production database.
3. **Then the web app, with NO gap** (build from the clean export, rsync `dist/` to `/home/ubuntu/comhard/app`, keep a rollback copy; record the bundle names here afterwards). Between the backend and a web app that states `udf2`, the server treats today's shipped build as an older app, and today's v1 master form clears a custom value by omission, so the server puts a cleared value back on ledgers and
   items that carry v1 custom values (the corrected document is handed back and a protected-fields row is recorded; nothing is lost, but a person cannot clear a value until the app updates). An un-reloaded web tab or an Electron install that was not rebuilt behaves as an older app afterwards.
4. **Rebuild the Electron installers** (also still owed since wave 3's PDF bridge, never run on a real Electron; `electron/` is unchanged by Phase 30).
5. **Release gates (spec section 17):** G1, an owner beta (CF-0, CF-14, CF-1, CF-6a deployed and smoked, CF-2, CF-4a, CF-4b-lite): met in code once step 2 passes. G2, the customer release (G1 plus CF-3, CF-4b, CF-5 and CF-9 for a company that keeps stock by a variant): NOT met, CF-4b is missing. The editor refuses a field that needs `udf2` while the stored server caps lack it
   (`needsServerCap`), which is the safety net, not the plan. **A count by variant must not reach a customer before the build carrying CF-9a runs everywhere that company's stock is read** (an older build reads a count by variant last-line-wins). Run the smoke's cleanup SQL when done.
6. **Rollback:** the web and backend rollback copies; there is no migration to undo. A rolled-back backend ignores `caps` (harmless); `Company.entryPolicy` written by new clients stays in the company document as an unknown field, which an older client's resave can erase (the client mirror and Restore are the defence, as for `ui`).
7. **Docs after the deploy:** the five `beyond` rows lose their `undeployed` status in `docs/parity-data.json` (then `python3 scripts/gen-kanban.py`), and `CLAUDE.md`, `COMSKY_BOOKS_PRODUCT.md`, `docs/FEATURE-PARITY.md` and this section say deployed, with what was and was not checked.

**NOT built (all optional or gated in the spec).** CF-4b, the same field as a column, header filter and SORT in the listing reports through the report grid: there is no grid (B-grid S1 and S2), and this is the only slice between the build and G2; until then a field works as a filter on four reports and as a dimension on five analysis screens. CF-6b (`backend/src/udfRules.js`, the
server validating against the company's definitions: REST cannot warn about a blank variant or a mandatory field). CF-9e (per-variant costing, gated on owner question 18), CF-9f (variant-aware manufacture: the Manufacturing Journal books Not set and warns), CF-10 (Tally UDF import and export, gated on three real exports), CF-11 (field-level access), CF-12 (employee and lead fields in the
payroll and CRM reports), CF-5b (Layout B), B-5 (an Esc Quit button on the Gateway submenus, only if the owner says so), B-6 (the browser's Back, the thumb button, a swipe, `beforeunload`; back-nav questions Q3 and Q8), B-4's grid half. **Partial:** profit by a tracking line field is engine-complete but the screen passes no reader (wire the per-variant opening and closing value into
`ui/reports/fieldPl.tsx`); a lookup is on masters only and cannot group; saved views on the analysis screens keep F12 settings only; CSV Import Data and the legacy CSV importers have no custom column and no "booked as Not set" preview; Fill blanks covers voucher-header fields only; stock by field costs by the item's pooled value; the Employees and Leads lists have no row filter.

**Open defects and risks.** A six-line-field invoice costs about 1.7x the keystroke latency (7.7 ms p50 against 4.4; memoising the item rows was judged a risky edit of the voucher grid); `stockPositions` with a variant resolver measured 1.17 to 1.67x the plain call at 250,000 vouchers on a busy machine (plan: at most 1.3x; re-measure on an idle one); the editor's `usageOf` costs 690 ms at 250,000 vouchers
(memoised, editor only); on a master form a select's open popup covers the Yes/No buttons of its own row; the Alt+D Delete buttons on the employee and lead forms have no click handler; a lookup list has no row cap; 13 of the 2,447 batch-engine mutation kills are crashes that prove nothing about assertions (see the batch-engines paragraph below). The mixed-version cases (an older app resaving an employee or lead on the real
server) are covered by mock-server suites only. Owner questions still open (spec section 18): 4 (field-level access), 6 (Tally UDF import), 18 (per-variant costing), 21 (employee and lead fields in reports); back-nav Q3 and Q8.

**The MCP package is out of step.** 9 of the 52 files pinned in `mcp/engine-lock.json` changed in `1d22e84` (`src/types.ts` and `src/core/` `backupMigrate`, `integrity`, `inventory`, `openingStock`, `splitCompany`, `syncGuard`, `valuation`, `voucherCancel`; found by comparing SHA-256, not by running the drift script). `cd mcp && npm run drift` therefore fails its engine-lock check until the reconciliation suites pass again and
`node scripts/drift.mjs --write-lock` rewrites the lock; do that before the next MCP release (§2g). Phase 30 did not do it, and its notes do not mention the lock.

**Also merged in this window: the batch, serial/IMEI, scanning and label engines (`5aee765`, branch `trk-batch-serial`, built 2026-10-24; NOT parity, no `beyond` row; wired to NO screen).** Four pure slices of new files, spec `docs/design/batch-serial-labels-spec.md` (36 decisions, 28 owner questions with defaults), build notes `docs/design/batch-serial-labels-build-a-notes.md`: T-0a (unit, batch, stock-cell and rule engines:
`src/trackTypes.ts`, `core/unitId.ts`, `unitChain.ts`, `batches.ts`, `trackRules.ts`, `stockCell.ts`; the shared chain code `backend/src/unitChain.js` and `unitKey.js`), T-5a (scan detector, resolver, scale labels, GS1 reader), T-6a (EAN-13/8, UPC-A, Code 128, Code 39, ITF, GS1-128 encoders, QR through the existing `core/qr.ts`) and T-6b (label model, layout, Legal Metrology and FSSAI warnings,
a TSPL raw-job emitter; ZPL is refused). 23 suites, 101,461 assertions, 2,447 mutations killed; nothing imports it, so no screen, key or number changed (asserted). **Not built:** T-0b (the integration into `types.ts`, `inventory.ts`, `valuation.ts`, `openingStock.ts`, the write guard and the MCP engine lock; its gate, CF-9a committed, reviewed and green on a rebased `main`, is met by `1d22e84`: re-read spec 15.3),
T-1 to T-4, T-5b, T-6c, T-6d, the T-6s printed spike (needs the owner's printers), T-7a, T-7b, T-8, T-9. Known costs and risks (build notes section 7): the unit model takes about 1.5 s at 235k serial rows against the spec's T-0b budgets, raw ids are normalised at the doors and not inside `eventsOf`, and a few spec numbers rest on secondary sources.

**Hygiene.** Add by explicit path, never `git add -A`. `origin` is 7 commits behind `main`; a push needs the owner's SSH password. The worktrees `.wt-cf` (`cf-back-nav`) and `.wt-bs` (`trk-batch-serial`) are merged into `main` and were left in place; `.wt-grid` (`report-grid`) was cut from `1d22e84` and had no commits of its own when this was written (its name suggests the report grid that CF-4b needs; nothing in the record states its scope).
The disk sat at 98 to 100 percent through the whole Phase 30 build (each headless Chrome profile grows about 55 MB a minute in Code Cache; the mutation runner now clones copy-on-write); free space before any sweep. Never `pkill` by pattern on this machine. New gotchas are in `CLAUDE.md` under Phase 30.

## 3. First 15 minutes in the new account

1. `npm install && npm run build` — confirm 0 tsc errors + build passes (baseline health).
2. `npm run dev`, open http://localhost:5173. First run shows the theme chooser → pick one.
   Create a company (Ctrl+A to accept forms), enter a Payment/Sales voucher, open Balance
   Sheet. Press `Alt+G` (Classic) or `⌘K` (Modern) — the command palette lists everything.
3. Skim `docs/FEATURE-PARITY.md` (what's done/partial/missing) and `docs/ROADMAP.md`.
4. Read the "Conventions & gotchas" in `CLAUDE.md` before editing.

## 4. How this was built (methodology that works here)

The productive loop, repeated per phase:
1. **Foundation (do by hand):** extend `types.ts`, `db.ts` (Dexie version + migration),
   `store.ts` (Screen routes + collections + `checkDelete`), add menu/palette/GoTo entries,
   and create **stub files** (`export const X = () => <ComingSoon .../>`).
2. **Parallel fan-out (Workflow tool):** spawn agents that each own **exactly one NEW file**
   (disjoint ownership = no merge collisions). Give each the patterns to copy
   (`docs/SPEC-V2.md`, `masters.tsx`, `reports.tsx`) and hard rules (touch only your file,
   tsc must pass). 5–6 agents/phase is the sweet spot.
3. **Integrate:** full `tsc` + `build`, fix any cross-file glue, verify key screens live in
   the preview, commit.
4. **Cross-cutting features** (voucher screen, engines, shared `reports.tsx`) are done **by
   hand, sequentially** — never parallelised (they'd collide).
5. **Adversarial review (Workflow):** finder agents by dimension → 2-refuter verification →
   fix only confirmed findings. This caught the biggest bug (optional-voucher leak).

**Gotcha with the Kanban refresh:** flipping feature status by keyword match OVER-CLAIMS
(e.g. `sTANdard` matches `tan`, e-way ≠ e-invoice). Always hand-correct the flips in
`parity-data.json` before regenerating. The `covered/partial/missing` counts in the board
are curated, not automatic.

**Watch for:** session rate-limits can kill fan-out agents mid-response — always check which
files actually landed (grep for `ComingSoon`) vs the reported failures, and finish any
dropped agent's file by hand.

## 5. Phase history (all on `main`, commit-by-commit)

MVP (accounting core) → v2 (inventory/GST/billwise/cost-centres/TDS/payroll/orders/banking/
multi-company/backup/sync/printing, 25+ reports) → Modern theme + ⌘K palette → parity audit
(381-feature gap list + board) → report print/export → comparative columns → audit log,
import, exceptions, security, price levels, budgets, GST JSON, F12 engine, FIFO/batches →
e-Invoice/e-Way, TDS 26Q, payroll statutory, manufacturing/BOM, interest → cheque, GSTR-2B,
profitability, reminders, reorder, cost analysis, credit-limit → optional/post-dated
vouchers, PAN/TAN → adversarial review + fixes. Then Phases 6–12 (TCS/TDS, banking, GST
validation, GSTR-1 B2CL/CDNR, GSTR-3B full, payroll depth w/ Dexie v5) → **~56% weighted
parity, every P0 closed.** Then the **cloud SaaS track (P1–P4 + trials/subscriptions +
admin), deployed to the VM — see §2.** Phases 13–17 closed more P1/P2 gaps; Phases 18–23 built
what Tally does not have (CRM/HR, open API, ghost recall, undo, bulk entry, the Six Keystrokes look)
and moved parity only by corrections; **Phase 24 (§2a) moved it to ~73.5%**, with a 10-cluster wave 1
and an 11-cluster wave 2, each followed by adversarial review or a fix pass. Phase 25 (§2b, the left menu,
deployed) and Phase 26 (§2c, the Zoho-style UX wave, built, reviewed and deployed 2026-10-20) are beyond-Tally work and
moved no parity row; so is Phase 27 (§2d, the Settings window and configurable shortcuts, wave B: built and reviewed
2026-10-20, committed `5c94342`, web app deployed 2026-10-21, backend half deployed 2026-10-23) and Phase 28 (§2e, Dashboard v2: 22 widgets,
charts, as-at figures, a new default board; built and reviewed 2026-10-20, committed `5c94342`, deployed 2026-10-21, front-end only). **Phase 29 (§2f, parity wave 3: cancel keeping the number,
party credit limit / interest / price level, single-entry layout, .xlsx and PDF and share, e-way bill and dispatch, server voucher numbers) is the second parity wave: 73.5% to 73.2% because nine
overstated rows were re-judged; built 2026-10-22, three of five clusters also reviewed, committed `2cee555`, deployed 2026-10-23 (web app and backend, BETA).** The MCP server (§2g) was committed as `d7846ce`. **Phase 30 (§2h: custom fields v2, entry rules, five Analysis by field reports,
stock by a variant field, the Back control and a Close on every dialog) is beyond-Tally work and moved no parity row; built and reviewed in a worktree 2026-10-24, merged into `main` as `1d22e84` on 2026-10-25, NOT DEPLOYED.** The batch / serial / scan / label engines were merged the same day as `5aee765` (pure new files, wired to no screen).

## 6. Roadmap — three tracks (pick per priority)

1. **Finish Tally parity:** what is left is listed with reasons in
   `docs/research/ROADMAP-ADDENDUM.md` (job-work chain, multiple/nested BOM + by-products, voucher
   class, multi-currency, payroll income tax and salary revisions, eTDS/FVU, plus the partial rows
   that need a real TallyPrime / portal / bank file to close). Same fan-out method; take the more
   conservative of a builder's status and its fix pass.
   (Phase 29, §2f, took the cancel, single-entry, export and share, e-way and numbering rows as far as they go without a live file or a contract; what is left of each is in §2f.)
2. **Beyond-Tally SaaS (the moat):** Custom Fields / UDF (v1 shipped client-side; **v2 with reports by field and stock by variant is merged but not deployed, §2h**),
   Open REST APIs + API keys (**shipped, P4**), webhooks (**shipped, P4**), billing/trials
   (**shipped, SaaS/M3**), team collaboration (comments/mentions/approvals — not built), HRMS,
   CRM, mobile. Remaining epics build on the deployed backend (§2).
   (The MCP server, §2g, lets a customer's own AI tool read their books: committed, not deployed; its deploy order starts with TLS. The batch / serial / label engines, §2h, are merged and not wired to any screen; their integration, T-0b onwards, is the next step there.
   The custom-field slice that is still missing for a customer release is CF-4b, the field as a grid column and sort, which waits for the report grid.)
3. **Architecture / cloud backend:** `docs/ARCHITECTURE.md` is **written AND P1–P4 are built +
   deployed** (multi-tenant RLS, sync/conflict, auth, server voucher numbers, REST/webhooks) —
   see §2 + §13 of that doc. What remains is **Phase B productionization** (TLS, live payments,
   CI/CD, monitoring, restore drill, load test, security review) + the P4+ scale items —
   the engineer handoff work listed in §2 "Resume / next steps".

## 7. Known limitations (carry these forward — see also the Trust Map in §2)

- F12 report-config engine is on the main statements, registers and Stock Summary; GST returns,
  TDS, payroll statements and about 35 other report files are not on it yet (one `useF12` call each).
  The Phase 26 row filter (Alt+F12 / Ctrl+F12) is narrower still: the Day Book, voucher registers, Ledger Vouchers and
  Receivables/Payables only (§2c).
- Phase 27 (§2d) is committed (`5c94342`) and its web app deployed (2026-10-21); its backend half (`backend/src/conflict.js`,
  `sync.js`) was deployed on 2026-10-23 with Phase 29's backend and has not been exercised on the VM (it needs a signed-in write). At 150 % text size
  the Dashboard's Receivables and Payables card titles sit under the `+ New` button (§2d, still open after §2e).
- Phase 28 (§2e) is committed and deployed (2026-10-21; front-end only; BETA). Its default board costs about twice the old one in
  engine time at 250,000 vouchers (717 against 350 ms, ungated); view widgets and the Overdue preset wait for `core/listQuery.ts`;
  an un-updated client that saves a dashboard layout drops the per-widget settings of a newer build.
- e-Invoice: IRNs are never generated here (portal results are imported, QR printed; the mock IRN is
  sample-only and watermarked; no IRP/GSP call). GST JSON is portal-shaped, not filing-grade;
  payroll statutory uses standard rates (not per-state configurable).
- Voucher numbering is per-device sequential (two offline devices can collide) when offline or in the local app; signed in and online, Phase 29's client (§2f) asks the cloud for the number on Accept and flags
  a clash instead of renumbering it. The backend half (`008_voucher_series.sql`, `numbering.js`) has been on the VM since 2026-10-23 but no signed-in reserve, claim or clash has been exercised there, and RCM / Rule 37 journals and bank-statement import still number locally.
- Phase 29 (§2f) is built (three of five clusters also reviewed), committed (`2cee555`) and deployed (2026-10-23, web app and backend, BETA); its desktop PDF write and the Electron share bridge were never run (the installers are NOT rebuilt), the e-way bill file was never
  uploaded to the portal, and the stale perf mirror (`vch.altWalk`) was re-pointed before the commit.
- Phase 30 (§2h) is merged into `main` (`1d22e84`, 2026-10-25) and NOT deployed (BETA; headless Chrome on macOS only). Its backend half (JavaScript, no migration) goes before or with the web app and the owner runs the deploy smoke first. A custom field is not yet a grid column or a sort (CF-4b), the server does not validate custom values (CF-6b),
  stock by variant has one pooled cost per item and no variant-aware manufacture, a lookup is on masters only, there is no Tally UDF import and no field-level access, the browser's Back and the thumb button are not wired (B-6), and `mcp/engine-lock.json` is stale until the MCP package is re-reconciled. The batch / serial / label engines have no screen.
- Backup-restore now queues deletes for rows the file lacks (restore means replace), so restoring an
  older backup on a synced device deletes newer cloud records; the question names the count. A
  Receipt Note followed by an unlinked Purchase invoice can still double-count stock.
