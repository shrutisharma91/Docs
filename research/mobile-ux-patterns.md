# Mobile UX patterns for Comsky Books: platform rules, how others do it, India constraints, and a design brief for owners and salespeople

> **Status:** research and design brief. Nothing here is built and this document changes no code. Written 2026-10-22 against `main`
> at `83d9876`; the working tree also carries about 160 uncommitted paths (parity wave 3, the MCP server build). A fact about an
> uncommitted file is marked as such and may move.
> **The ask (owner, verbatim intent):** "research well, and develop great mobile apps, considering also business owners and sales
> people using the app, in mind."
> **What this is:** the UX layer: platform rules, competitor structure, India-specific constraints, interaction design for entry
> and reports, navigation, a screen inventory with key interactions, and principles. **What it is not:** the persona and job analysis
> ([mobile-personas-and-jobs.md](mobile-personas-and-jobs.md), sections 2 to 8 and 12), the backend audit
> ([mobile-backend-readiness.md](mobile-backend-readiness.md), items B1 to B12) or the market survey
> ([mobile-market.md](mobile-market.md)). All three were written in parallel; this document cites them instead of repeating them, and
> uses their findings as fixed inputs. Where they overlap with this one (a PWA first, HTTPS first, one 2.74 MB script, the rep
> holding a projection), they agree.
> **Classification:** mobile is a *Beyond Tally* capability. Track it under `beyond` in `docs/parity-data.json`, never in the parity
> percentage (CLAUDE.md, "Phases 18-21").
> **Evidence tags:** **[W]** a page I fetched and read (the fetch tool summarised it; I paraphrase and never copy) · **[Ws]** a search
> summary only, the page itself not read · **[C]** read in this repo (`file:line`) · **[M]** measured by me in this session, method in
> Appendix B · **[J]** my judgement or arithmetic from cited inputs, not evidence · **[unverified]** stated but not checked. URLs are
> in Appendix A.
> **No user was observed, no competitor account was used, no real device was driven, and no store screenshot was viewed** (section 1).

---

## 0. TL;DR

1. **Build the phone app as ONE installable web app (PWA) with two role shells, an Owner shell and a Field shell, inside the same
   codebase, and add a store wrapper later only when a named trigger fires** (section 6). Reasons that are checked: the pure engines in
   `src/core` and the sync client already run in a browser; Android carries 93.9% of Indian mobile web traffic [W] and Chrome on
   Android has the PWA features we need (install prompt, push with action buttons, Background Sync, a barcode detector) [W]; the same
   app is about 0.83 MiB of gzipped script today against 12 to 580 MB for the native apps in this market [M, W]. Where it is weak
   (iOS: no install prompt event, no push action buttons, no Background Sync; both: the OS text size is not applied by default, and I
   could not confirm any home-screen widget support) the document lists a fix or an accepted cost. This is the opposite of the old roadmap line, which said "React Native app" [C `docs/ROADMAP.md:32`];
   section 6.2 says why that line should be reopened.
2. **Three platform gaps decide the order, and none of them is a screen:** HTTPS (a plain-HTTP origin cannot register a service
   worker, so no install, no offline shell, no push, no camera) [W, C]; a shell that fits a phone (today it scrolls sideways below
   938 px) [C]; and per-person identity plus a server-built data scope for reps (today every member's phone holds the whole ledger and
   the operator PIN hashes) [C, personas section 2, readiness section 4]. The UX below assumes those are being closed.
3. **The owner's phone is a pulse, a decision and a chase.** Of Dashboard v2's 22 implemented widgets, 11 translate as they are, 6 become
   two-line rows with swipe actions, 3 charts need a tap-to-select redesign (they use hover and keyboard focus today), and 2 are
   replaced (section 10.2). The owner Home is: Cash & Bank with an "as at" stamp, Receivables with its overdue split, a "Needs you"
   feed (approvals first), Today, and a short overdue list.
4. **The rep's phone is an order, a collection, a visit and a party card, and it never invoices in v1.** A Sales Order is a non-posting
   voucher (`accounting: false`, no stock effect) [C `src/types.ts:35`], so it is safe to create offline; the invoice follows at the
   desk. A receipt taken in the field is **pending verification**, because a UPI deep link gives the phone no payment status (on
   iOS a custom-scheme link reports only that an app opened; gateways confirm on the server) [Ws]. The party picker shows
   outstanding, overdue and credit headroom inline.
5. **Entry on a phone is not a smaller grid.** It is a stack of line cards with a stepper, a sticky total in lakh and crore, a review
   sheet that shows the figure in words and the credit projection, an autosaved draft and an undo toast (section 9). The voucher screen
   is 4,119 lines and builds a voucher's entries inline [C `src/ui/VoucherScreen.tsx:2540`, `:2685-2702`], so a phone entry screen first
   needs that composition extracted into `src/core` (one `composeSalesOrder` and one `composeReceipt`), or it will fork the books.
6. **Money guardrails are specific, not a generic "Are you sure?"**: live figure in lakh and crore under the field, words on review,
   a magnitude check against the party's history ("usually ₹12,500, this is 100 times"), a duplicate check, a cash-limit check at the
   ₹2 lakh line [Ws], confirmation only for the irreversible, undo for everything else (section 9.7). NN/g and Apple's alert rules both
   say a reflexive confirmation protects nobody [W].
7. **India facts that change the design, each measured or sourced:** Plex Sans figures are tabular by default and **about 13%
   narrower than Plex Mono** for lakh and crore amounts, so figures use Sans on a phone [M]; the rupee sign exists only in the latin-ext file
   [M, C]; `Intl.NumberFormat` is unsafe for regional Indian locales (Tamil prints "million", Kannada groups in thousands, Marathi and
   Bengali flip to native digits, English prints "12KCr") so the app keeps its own formatter in every language [M]; Plex has Devanagari
   as a separate family (77 KB a weight) and no Tamil, Telugu, Bengali or Gujarati [W, Ws] (section 5.3).
8. **Performance: the app ships one 2.74 MB script (873 KB gzipped, 683 KB brotli) and zero lazy imports** [M, C]. It was 345 KB
   gzipped when `docs/PERF.md` was written [C `docs/PERF.md:331`]. For a 75th-percentile Indian phone a JS-heavy page has about
   0.62 MiB of JavaScript for a 3-second load and 1.15 MiB for 5 seconds [W]. The Field shell needs its own entry chunk with a hard
   budget (proposed 350 KB gzipped, [J]); today's bundle fails that on any phone.
9. **Web-platform traps that native apps do not have, all verified:** Chrome on Android does not apply the OS text size to web pages by
   default; iOS Safari applies it only to text set with a system text style [W]; iOS zooms into any input below 16 px [W]. So a
   person who raised the phone's text size sees no change in our web app unless we build the control. The app's Settings already has
   a text size control to 150% [C]; a phone needs 200% (WCAG 1.4.4, Apple's 200%) [W] and a 16 px floor for body and inputs.
10. **WhatsApp is the channel, and one tap versus two decides the flow.** Text with a link opens WhatsApp on a prefilled number in one
    tap (`wa.me`, already built) [C]; a PDF needs the Web Share API with files, which lets the person pick the contact (two taps);
    and the web build cannot make a PDF at all (the browser keeps its print dialog) [C `src/ui/pdfOut.ts:1-9`]. The phone needs a
    PDF-or-image path (section 5.5): an image for receipts and orders first, a server PDF and a hosted link later.
11. **Sign-in: the owner's desktop shows a QR with a confirm code; the phone scans it; the owner approves a named device; a rep joins
    from a WhatsApp invite link. Nobody types an API key** (API keys are tenant-wide and for integrators [C
    `backend/src/platform.js:83`]). The flow has the shape of the OAuth device grant (RFC 8628) and must carry its mitigations plus
    the QR-login ones (QRLJacking): a confirm prompt with device detail, a short expiry, single use [W].
12. **Keeping the Superhuman feel on touch** means four things: a search-first entry (the Go To ranker, unchanged), recents and
    pins on the empty state, swipe actions that are never the only path, and a contextual create button. Superhuman's own phone app
    replaced keyboard shortcuts with gestures and reach-friendly placement [W]; Apple and Android both say a gesture must have a
    visible alternative [W] (section 11).
13. **First spike (proposed, section 16.4):** a phone Home, a Party card and a Collect form over the existing store behind a flag,
    installable from a TLS tunnel on one real A-class Android, measured for bundle, load and tap latency, and shown to five people.

---

## 1. Method, and what I could not verify

| Item | What happened | Effect on this document |
|---|---|---|
| Apple Human Interface Guidelines HTML pages | Return only a title to a fetcher | Read through the JSON mirror of each page at `developer.apple.com/tutorials/data/design/human-interface-guidelines/<page>.json` [W] |
| Material 3 (m3.material.io) pages | Return only a title (script-rendered) | Material rules come from Android's own design pages, Google's accessibility help and the Material Components for Android docs on GitHub [W]; the M3 spec text itself is not read |
| Google Play store pages (Zoho Books, Pipedrive, Khatabook, Xero) | Returned a truncated placeholder | No Play ratings, download counts or Play descriptions. Android size and languages are unknown |
| Apple App Store listings | Read through the iTunes Search and Lookup API, India storefront (`country=in`), for version, file size, languages, rating counts and the description | Competitor size and language figures are the **iOS download size** from the listing, not an Android size; the descriptions are vendor text [W] |
| **Store screenshots and apps** | Not viewed (images cannot be read by the fetcher; saving them needs a download I did not make) | **Layouts of competitor home screens, pickers and quick-create are inferred from vendor text and help pages, not seen.** Section 4 says which claims are which. Nothing here says "Vyapar's picker looks like X" |
| Any competitor account or device | None used | No hands-on view of any picker, offline behaviour or speed |
| Real devices | None driven | Every performance, contrast-in-sun and touch claim for our own app is a projection or a computed number [M, J] |
| Opensignal, NPCI, incometaxindia.gov.in, some Salesforce pages | 403 or empty | Network and tax figures come from search summaries [Ws] |
| The uncommitted tree | A parity gate and an MCP build were editing it | `dist/` was built at 21:05 on 2026-10-21 from that tree; byte counts are indicative |

---

## 2. What exists today, seen from a phone

Identity, sync and TLS are covered in depth elsewhere; the rows marked "see" give the one-line consequence and the pointer. The rest
are UI facts this document needs.

| # | Fact | Evidence | Consequence for the phone |
|---|---|---|---|
| 1 | **No manifest, no service worker, no PWA plugin.** `index.html` holds a viewport tag only; `vite.config.ts` has the React plugin and nothing else; there is no `public/` directory. | `index.html:5`; `vite.config.ts:1-10` | Nothing installs or runs offline from the web today. |
| 2 | **The live host is plain HTTP**, and a service worker, push, camera, share and WebAuthn need a secure context. | `docs/HANDOFF.md` trust map; [W] MDN Secure Contexts; readiness section 9 | HTTPS first (readiness B1). See. |
| 3 | **The shell has a 938 px floor** (212 rail + 574 content + 152 F-key rail); at 390 px the rail is 54% of the screen and the sample banner wraps to four lines. The Dashboard board collapses by container query at 780, 640 and 460 px, but the shell around it does not. | `docs/design/ux-analysis-2026-10.md:18`, `:274-279`; `src/styles/dashcustom.css:25-31`; `src/styles.css:583`, `:587`, `:1544` | A phone view is a new shell. The board's own collapse rules are reusable. |
| 4 | **Type is built in px, at 13.5 px (compact) or 14 px (comfortable) for body**, and rows are about 21 px tall. The text-size setting runs 90 to 150% through `--ui-k`. | `src/styles.css:65`, `:90-120`; CLAUDE.md "Phase 27" | Below the 16 px floor iOS needs for inputs and the 200% WCAG asks for; see section 3.3. |
| 5 | **Hover is a feature.** Chart tooltips open on `pointerenter` and a click drills; labels use `title`; the rail's `+` and star reveal on hover (one `@media (hover: none)` rule exists, for the rail only). | `src/ui/dash/charts/Columns.tsx:190-195`; `src/ui/dash/charts/Tip.tsx:1-6`; `src/styles/navtree.css:176`; `docs/design/ux-analysis-2026-10.md:270` | On a touch screen a tap would show the tip and drill in the same gesture [J, from the code; not run on a device]. Charts need a select-then-drill model. |
| 6 | **One script chunk: 2,743,605 bytes raw, 872,823 gzipped, 683,446 brotli; CSS 291,416 raw, 47,860 gzipped; zero `lazy()` or dynamic route imports.** `docs/PERF.md` quoted 1,166 kB raw and 345 kB gzipped earlier. | [M] on `dist/assets/index-D16gwGz7.js`; `grep -rn "lazy(" src` finds nothing; `docs/PERF.md:331` | The bundle has grown 2.5 times since PERF.md. A phone shell must not load the desktop's 99 reports to show a Home. |
| 7 | **No i18n layer.** `<html lang="en">`; strings are literals; `Intl` appears in a few admin screens only, always `en-IN`. | `index.html:2`; search of `src/` for `i18n`, `useTranslation` (none) | Vernacular is new work; see 5.4. |
| 8 | **Fonts:** self-hosted IBM Plex Sans (400, 500, 600) and Plex Mono, **latin and latin-ext only**; the rupee sign U+20B9 is in latin-ext only; the font stack falls through to OS faces for Devanagari, Tamil and others. | `src/styles/fonts.css:1-40`; `src/styles.css:20-37`; [M] | See 5.2 and 5.3. |
| 9 | **Sharing exists and sends nothing by itself:** `wa.me/<digits>?text=` (text capped), `mailto:`, a UPI link and QR; on the web the "PDF" is the browser's print dialog. No `navigator.share` call exists. | `src/core/share.ts:7-12`, `:124`; `src/core/upi.ts:7-12`; `src/ui/share/SharePanel.tsx:31-44`; `src/ui/pdfOut.ts:1-9`; search of `src/` for `navigator.share` (none) | WhatsApp text and link work on a phone today; a file does not. |
| 10 | **Entry logic that is pure:** ghost recall (`findRecall`), credit exposure and its reasoned override, price levels (`sellingRate`), single-entry Receipt and Payment projection, `amountInWords`, Indian grouping `fmtAmt`. **Entry logic that is not:** a voucher's entries (party leg, tax rows, TDS and TCS, Round Off ledger) are composed inside `VoucherScreen.tsx` `save`, and Bulk Entry re-implements a subset in a UI file. | `src/core/recall.ts:48`; `src/core/creditLimit.ts:1-20`; `src/core/singleEntry.ts:1-30`; `src/core/money.ts:2-14`, `:196`, `:210`; `src/ui/VoucherScreen.tsx:2540`, `:2685-2702`; `src/ui/admin/BulkEntry.tsx:1-60` | A phone order or receipt screen needs the composition in core first (section 9.10). Receipts are the easy one: the single-entry shape is already pure. |
| 11 | **Data the field needs and does not have:** no barcode on `StockItem` (only `alias` and `partNo`); no salesperson on a voucher (only free-text `Lead.owner`); only `LeadActivity` for visits (prospects, not customers); Sales Order and Receipt exist. | `src/types.ts:488-527`, `:571-600`, `:35`; `docs/research/pipedrive-list-ux.md:468`; personas section 2 row 8 | Barcode is a new `StockItem` field, appended last (the frozen field order rule [C CLAUDE.md Phase 24 GOTCHAs]); a visit is a new document. |
| 12 | **Reusable speed assets:** the Go To ranker with strict tiers and a 16 ms per-keystroke budget; recents and pins in `state/recents.ts`; one quick-create registry of 28 creates; the widget registry with rights, features, drill and as-at clocks; eight validated accents; a high-contrast mode with every text token at 7:1 or more; SSE live sync (`src/sync/realtime.ts`). | `src/ui/paletteRank.ts:1-30`; `docs/PERF.md:74-80`; `src/state/recents.ts:34-42`; `src/ui/quickCreate.ts:64-98`; `src/ui/dash/widgets/registry.ts:32-72`; `src/styles/accent.css`; `src/styles/contrast.css:1-12`; `src/sync/realtime.ts:1-30` | The "feel" is already built as engines and tokens; a phone shell composes them. |
| 13 | **Whole-tenant, unredacted, uncompressed pull; one login per tenant; rights on the client only; tokens in `localStorage`.** | `backend/src/sync.js:343-378`; `backend/src/auth.js:14-38`; `src/core/access.ts:10-14`; `src/sync/sync.ts:37-39`; readiness sections 3 to 5 | A rep must hold a projection, not the books. See. |

---

## 3. Platform rules, distilled

Apple's rules are read from the Human Interface Guidelines through their JSON mirror [W]; Android's from developer.android.com,
Google's accessibility help and the Material Components for Android docs [W]; web rules from WCAG, MDN, web.dev and WebKit [W].
"We adopt" is my reading [J].

### 3.1 Navigation

| Rule | Source | We adopt |
|---|---|---|
| A tab bar is for navigating between sections, not for actions; keep it visible; keep tabs few; avoid an overflow "More" tab; do not disable or hide a tab, explain an empty section instead; one word per label; use badges only for critical information | Apple HIG tab bars [W] | Five tabs at most, per role; **no More tab**; create is not a tab (a contextual button instead); badge only for approvals and unsent items |
| Android navigation bar holds three to five destinations; with four or more only the selected label shows by default; badges can be a dot | Material Components for Android, Bottom Navigation [W] | Show all labels (we have five short words); a dot for "something waits" |
| A floating action button marks the screen's primary action; a medium FAB is the first recommendation in compact windows; a standard one is 56 dp | Material Components FAB doc [W] | One contextual create button on Home and lists (Owner: Create; Field: New order); 56 px; mirrored for left-handed users |
| Window size classes: one pane when compact; adapt navigation by size | Android adapt-layout [W]; Apple adaptivity (size classes, not device types) [W] | The phone shell is the compact class; a tablet class later reuses it with a rail (section 8) |
| Draw edge to edge; keep touch targets out of the gesture-navigation insets; the system may add a scrim behind three-button navigation | Android system bars [W] | Tab bar and FAB sit above `env(safe-area-inset-bottom)`; no target in the bottom gesture strip [J on the CSS] |
| Do not override system gestures; offer a visible control and treat an edge swipe as an accelerator | Apple gestures [W] | Back is a visible button plus the system back; no custom edge swipes |

### 3.2 Touch targets and spacing

| Rule | Source | We adopt |
|---|---|---|
| iOS default control size 44 by 44 pt, minimum 28; about 12 pt around bezelled controls, 24 pt around bezel-less ones | Apple HIG accessibility [W] | 44 px floor |
| Android: 48 by 48 dp minimum, 8 dp apart, about 9 mm on any screen; the touch area may exceed the drawn icon | Google Accessibility Help [W]; Android accessibility [W] | **48 px for rows, buttons and steppers; 12 px gaps**; hit areas expanded past small icons |
| WCAG 2.2 AA asks 24 by 24 CSS px, with a spacing exception; the 44 px figure is the stricter enhanced criterion | W3C Understanding 2.5.8 [W]; 2.5.5 [Ws] | 24 px is a legal floor, never used for a money control |
| About 1 cm square, 2 mm apart; a thumb impact area averages 2.5 cm | NN/g [W] | Same direction |
| 49% of observed touches were one-handed (67% right thumb, 33% left); grip changes every few seconds with the task; the author's 2017 note says reach diagrams were wrong and people mostly touch the centre | Hoober, UXmatters [W] | Primary actions in the lower half, **but no risky action hidden in a far corner**; a left-handed switch mirrors the create button |

### 3.3 Type and text size

| Rule | Source | We adopt |
|---|---|---|
| iOS body is 17 pt (Large), 11 pt is the minimum; thin and light weights are hard to read; people must be able to enlarge text at least 200% | Apple HIG typography and accessibility [W] | Body 16 px (inputs 16 px or more), amounts in lists 17 px; **Plex weights 400, 500, 600 only** (already true [C `fonts.css`]) |
| Android: sizes in sp, body 12 sp or more | Android accessibility [W] | Floor 12 px for any secondary text |
| WCAG 1.4.4: text resizes to 200% with nothing lost; a viewport that blocks zoom works against it | W3C [W] | **Never set `user-scalable=no` or `maximum-scale`**; support 200% |
| iOS Safari zooms into an input whose font is 15 px or smaller | CSS-Tricks (older article; re-check on device) [W] | Inputs at 16 px or more |
| **Chrome on Android does not apply the OS text size to web pages by default** (it has its own slider; an opt-in `meta text-scale` is behind a flag in Canary). **iOS Safari applies the OS size only to text set with a system text style** (`font: -apple-system-body`). Firefox for Android scales. About 37% of Android and 34% of iOS users have changed their system text scale (one author's figure, [Ws]) | Adrian Roselli, Feb 2026 [W]; WebKit [W]; search summary [Ws] | An **in-app text size to 200%** (today 90 to 150% [C]); opt in to `meta text-scale` when it ships; set the iOS system text style on the root and re-set `font-family` after it [J, to be tried]; relative units in the phone shell |
| Let layout reflow, not truncate, at large sizes | Apple Layout [W] | Rows grow; labels wrap; amounts switch to compact form if they cannot fit (5.2) |

### 3.4 Colour, contrast, dark mode

| Rule | Source | We adopt |
|---|---|---|
| AA: 4.5:1 for text up to 17 pt, 3:1 for large or bold text; Android also wants 3:1 for non-text parts | Apple HIG accessibility [W]; Android accessibility [W] | Existing tokens already pass (ink 16.4:1, dim 5.8:1, accent 5.8:1 on the sheet [M]) |
| Dark mode: aim for 7:1 on small custom text; use base and elevated background sets; **avoid an app-specific appearance setting, because people expect the system to decide** | Apple HIG dark mode [W] | On a phone the default is **"same as my phone"**; the manual override stays but is not a first-run question |
| Dark theme exists on Android 10+, saves power by an amount that depends on the screen technology, and helps low-vision and low-light use | Android [W] | Ship dark from day one |
| Do not carry meaning in colour alone; use a second cue | Android accessibility [W]; Dashboard v2 already prints words with every colour [C `docs/research/zoho-dashboard.md` section 3.11] | Keep it |

### 3.5 Lists, swipe, sheets

| Rule | Source | We adopt |
|---|---|---|
| Keep row text short; a middle ellipsis keeps both ends of a clipped name; a selected row that opens a view stays highlighted | Apple lists and tables [W] | Party names: two lines, then a middle ellipsis |
| A swipe must have a visible equivalent; a custom gesture is never the only way | Apple gestures [W]; Android accessibility (a trash button beside swipe-to-delete) [W] | Every swipe action is also a row button or in the row's menu |
| Sheets: medium and large detents, a grabber, swipe down to dismiss, confirm before discarding unsaved work, Cancel on the leading side and Done on the trailing, **never a sheet from a sheet** | Apple sheets [W]; Material bottom sheet (standard vs modal, a drag handle with a 48 dp target) [W] | Bottom sheets for short choices and the review step; **pickers inside a form are full-screen routes, not sheets**, so nothing nests |
| A snackbar gives brief feedback, holds one action such as Undo, shows one at a time, sits above the FAB and tab bar | Material Snackbar [W] | The undo toast already exists on desktop [C `src/styles/feedback.css`]; the phone version has a button |

### 3.6 Forms and number entry

| Rule | Source | We adopt |
|---|---|---|
| Show the keyboard that fits the content; validate at the right moment (digits-only fields tell the person at once); a clear button at the trailing end; a label that stays visible (a placeholder disappears) | Apple text fields, entering data [W] | As written |
| For numbers use `inputmode="numeric"` or `"decimal"`, **not `type="number"`** (scroll wheels change it, feedback is poor); a decimal keypad may have no minus key, so do not rely on typing a sign; prefix symbols sit outside the input and validation must accept a typed prefix; turn spellcheck off for codes and names | GOV.UK Design System [W]; MDN inputmode [W] | Money and quantity: `inputmode="decimal"`, `autocomplete="off"`, `spellcheck="false"`; **sign by an explicit control** (Dr/Cr, increase/decrease), never a typed minus; `inputmode="none"` on a field a hardware barcode scanner types into (so the keyboard does not pop up) [W MDN] |
| Fill with defaults and recents; offer choices over typing; do not let people continue without required data | Apple entering data [W] | Party picker with recents, ghost recall, price levels (section 9) |

### 3.7 Feedback, errors, undo

| Rule | Source | We adopt |
|---|---|---|
| Alerts interrupt: use them for an uncommon destructive action that cannot be undone, never for common undoable ones, **never at launch**; title says what happened; buttons name the result (verbs), never Yes or No; no default affirmative | Apple alerts [W]; NN/g confirmation dialogs (people stop reading them; name the item; no pre-selected Yes) [W] | Section 9.7 |
| Show feedback close to the thing, use several channels, confirm important actions only (people expect success) | Apple feedback [W] | Inline state chips; a toast only for saves and sends |
| Undo: do not redefine shake-to-undo; label the undo with the operation; batch related changes | Apple undo and redo [W] | Toast Undo for 10 s [J]; no shake handler |
| Show local data immediately; make an online-only write's offline state visible (disable, a message); queue the rest | Android offline-first [W] | The state vocabulary in 8.4 |

### 3.8 Search, notifications, quick actions, widgets, permissions

| Rule | Source | We adopt |
|---|---|---|
| Search gets a primary place (its own tab or a bottom toolbar); show recents before typing; one clear scope | Apple searching [W] | A Search entry on every shell, empty state = Pinned, Recent, Today |
| Notifications: concise; up to four action buttons on iOS; generic text when previews are hidden; nothing sensitive; do not repeat; badge only for unread; do not notify while the app is open | Apple notifications [W] | Private lock-screen text by default (also the personas doc's rule [C personas section 6.6]) |
| **Web push action buttons: Chrome Android yes; Safari iOS no** | MDN compat data, `Notification.actions` [W] | Android: Approve and Reject as buttons; iOS: tap opens the card |
| Home Screen quick actions: up to four, short titles, static or dynamic | Apple [W] | Manifest `shortcuts` on Android (MDN lists the member; support "limited" [W], Chrome Android per general knowledge [unverified]); none on iOS PWAs [unverified] |
| Widgets are glanceable, deep-link to the content, are not real time, and say when they last refreshed | Apple widgets [W] | A native-shell item (section 6); Zoho Books ships Android and iOS widgets [W] |
| Ask permission in context, with one sentence on why; keep the app useful if refused; collect only what you need | Apple privacy and onboarding [W] | Camera on the first scan, microphone on the first voice use, notifications after the first approval, location only at check-in (and only if the owner enables it) |

### 3.9 Haptics, onboarding, loading

| Rule | Source | We adopt |
|---|---|---|
| Haptics: success, warning, error, impact, selection; short; tied to a cause; optional; the app must work without them | Apple haptics [W] | A short success pulse on save and an error pulse on a block, behind a setting. The Vibration API is "limited availability" per MDN, and iOS Safari does not have it (general knowledge, [unverified]); treat haptics as Android-only polish |
| Onboarding short, optional, in context; defer setup; sign-in only when needed; offer passkeys; offer account deletion in the app | Apple onboarding and managing accounts [W]; App Store 5.1.1(v) [W] | Section 13 |
| Show something immediately; placeholders, not a blank; determinate progress when it is known | Apple loading [W] | Skeletons already exist on the board [C `src/ui/dash/skeleton.ts`]; cached Home on cold start |

---

## 4. How others structure a phone app

Features, prices and roles are in the personas doc (section 4.1); this section is about **structure and interaction**, and is honest
about evidence: I read vendor text and help pages, not screens (section 1).

### 4.1 What each product's phone app is, as far as published

| Product | Home and structure | Quick-create, entry, pickers | Collections and reminders | Offline | Size, languages (iOS listing, India) | Evidence |
|---|---|---|---|---|---|---|
| **Zoho Books** | Dashboard tiles: bank summary (cash on hand, payments clearing, bank balance), total receivables, overdue invoices and bills; a reports tab that includes credit limits; customer view with contact, transactions and credit | Estimates, invoices, recurring invoices, bills, expenses; voice invoice on iOS; scan receipts, barcodes and business cards; approvals by push | Accept payments; client portal for invoices and statements | Not mentioned on its page; a community thread is inconclusive | 453 MB; 26 languages including Hindi, Gujarati, Kannada, Marathi, Tamil, Telugu; 4.79 from 6,022 | [W] Zoho page and listing; [Ws] offline |
| **Zoho Daybook** (ledger) | Dashboard summaries and reports over a cash and credit ledger | Cash, credit, contacts | Reminders by SMS or WhatsApp | "100% offline" with automatic sync | 22.6 MB; 24 languages, Hindi the only Indic one | [W] listing |
| **Vyapar** | Billing and inventory app with roles including Salesman | Sale flow: party (or blank for cash sale), invoice details, items with quantity, price and discount, payment type, save, share. Barcode by camera, Bluetooth or USB scanner | Reminders and updates by SMS or WhatsApp; WhatsApp invoice | Offline-first locally; its own desktop guide says to turn sync off to bill offline (personas doc, section 4.1) | 235.8 MB; English and Hindi only; 3.50 from 1,122 | [W] listing; [Ws] vendor pages |
| **Khatabook** | The **party** is the unit: a ledger per customer and supplier, each entry with a note and attachments, a cashbook | Add customer, add credit or debit | Free SMS updates; reminders; calling a customer from the app; UPI and QR collection | Local save; offline behaviour unclear | 69.8 MB; **13 languages** (Assamese, Bengali, Gujarati, Hindi, Kannada, Malayalam, Marathi, Odia, Punjabi, Tamil, Telugu, English, Indonesian); 4.72 from 123,015 | [W] listing; [Ws] |
| **OkCredit** | Customer and supplier balances | Voice entry (the CEO's own statement); PDF statements | WhatsApp or SMS reminders | Offline | 98.4 MB; 10 languages; 4.85 from 4,478 | [W] listing; the CEO interview as cited in the personas doc (section 3) |
| **Pipedrive** | Agenda-oriented: a smart agenda view for planning activities | Log calls; start calls and emails in one tap; move deals; add notes and activities | n/a | **Move deals and add notes and activities offline; syncs when connected** | 83.1 MB; 23 languages | [W] listing and mobile page |
| **Zoho CRM** | Home is **activity-first**: the day's appointments, meetings, tasks and calls, with a calendar to change the day | Check-in at a customer (logged into the record's notes); voice notes saved as audio or converted to text, also on call logs; nearby customers within a radius | n/a | **Local database: view, add, change, delete records offline**; which modules depends on the edition | 581 MB; 27 languages including Hindi | [W] help and listing |
| **QuickBooks** | Dashboard of a cash-flow widget, a to-do feed that flags overdue invoices, bank balances and recent transactions; reports include P&L and a mileage log | Receipt photo to expense; connect bank and categorise | n/a | Not described | n/a | [W] Intuit help; layout of tabs conflicts across sources [Ws] |
| **Xero** | Quotes and invoices with a "viewed" status; bank reconciliation with suggestions; receipt upload with assistance | Tap to Pay on iPhone in five countries | n/a | Not described | 385.7 MB; English only (US storefront listing) | [W] listing; a Xero ideas thread says there is no Balance Sheet on mobile and no plan for one [Ws] |
| **Marg eOrder** (rep) and **eOwner** (owner) | Two separate apps | eOrder: carts editable offline or online, bill-by-bill collection, outstanding and PDC, live stock, route monitoring | eOwner: collections, outstanding, PDC, GPS location of salesmen, temporary credit limits | Offline cart | **12 MB** and 49 MB; English only | [W] listings; features beyond these are in personas section 4.1 |
| **Livekeeping** (companion to a desktop accounting package) | Dashboard comparing sales with the previous week or month; multi-company | Voucher entry, physical stock | Party-wise outstanding; GST invoices shared on WhatsApp | Not described | 86.7 MB; English; 4.66 from 1,818 | [W] listing |
| **Superhuman** (speed benchmark, e-mail) | Triage by swipe; split inbox switched at the bottom | Search by pulling down; the command menu by pulling down then swiping right; swipes and the triage bar are customisable | n/a | n/a | n/a | [W] vendor blog; [Ws] help |

### 4.2 Patterns worth taking **[J]**

1. **The owner's home is balances plus a to-do feed** (Zoho Books, QuickBooks): cash, what is owed, what is overdue, and a short list of things that need a decision.
2. **The party is the unit of work for Indian small businesses** (Khatabook, OkCredit, Zoho Daybook): one card per party with a balance, a reminder and an entry button. Our Party card should be the most polished screen in the app.
3. **A rep's home is activity-first** (Zoho CRM, Pipedrive): today's follow-ups, not a dashboard.
4. **Check-in, voice note and "nearby" are the field-CRM trio** (Zoho CRM). Check-in and voice note are cheap; "nearby" needs location the owner may not want (readiness decision 7).
5. **Offline means "edit locally, sync later" with explicit scope** (Pipedrive, Zoho CRM): the scope is deals and notes, not the whole company.
6. **ERP vendors split the owner and rep apps; CRMs and newer apps use one app with roles** (Marg versus Pipedrive and Vyapar's roles). Section 7 decides.
7. **Footprint matters**: native Indian small-business apps run 12 to 236 MB on iOS, and the global suites 385 to 581 MB [W]. A PWA's first load is about 1 MiB today (script 0.83 MiB gzipped, style 47 KB, fonts about 120 KB) [M].
8. **Speed on touch is gestures plus search**, not menus (Superhuman) [W].
9. **Languages are a real differentiator for owners and reps**: Khatabook 13, OkCredit 10, Zoho Books 26, while Vyapar, the Tally-companion apps and Marg's apps list English (and Hindi) [W]. Reps read the local language first (personas section 3).

### 4.3 Failure modes to design against

- **An offline mode that quietly skips the credit check** or tells people to turn sync off before billing (personas doc, sections 4.1 and 4.2).
- **A desktop report squeezed onto a phone.** Xero's mobile app has no Balance Sheet and Xero says it has no plan for one [Ws]: an omission rather than a squeeze, which is the right stance for the long tail of our 99 reports.
- **A rep app that is the owner app with fewer buttons** (it still carries the whole book; section 7 and readiness section 4).
- **English-only labels for field staff.**

### 4.4 What I could not verify

How any product's party and item pickers look and behave; offline behaviour of Zoho Books, QuickBooks and Xero; Android size and ratings; whether Zoho's reports tab works offline. The right way to close these is a day with six apps installed on one Android phone and a screen recording of the same three tasks (an order for a known party, a receipt against a bill, "who owes me") [J].

---

## 5. India: what it does to the interface

### 5.1 Devices, network and a performance budget

| Fact | Evidence | Reading |
|---|---|---|
| Android held 93.89% of Indian mobile OS web traffic in September 2026, iOS 6.05% | StatCounter [W] | Android Chrome first; iOS is owners and managers |
| Budget for a 75th-percentile user: target device a low-end Android (Galaxy A24 4G class), 9 Mbps down and 100 ms round trip; for a 3-second load about 0.62 MiB of JavaScript in a JS-heavy page, for 5 seconds about 1.15 MiB; **India's own 75th percentile is about 6.2 Mbps and 85 ms**; emerging-market services should anchor on the 90th or 95th percentile | Alex Russell, Performance Inequality Gap 2026 [W] | Field shell first-load JS target 350 KB gzipped, owner shell 500 KB [J]; today 873 KB for everything [M] |
| Interaction to Next Paint: good is 200 ms or less at the 75th percentile | web.dev [W] | Tap-to-feedback target 100 ms, INP at or below 200 ms [J] |
| Per-keystroke picker filtering must stay at or under 16 ms (one frame); measured on an M4 Mac, not a phone | `docs/PERF.md:74-80` [C] | The perf harness's `--cpu-factor` is a what-if only; the phone number is unmeasured |
| The largest unmeasured cost is DOM size: no virtualisation anywhere; `SelectPopup` renders every match | `docs/PERF.md:331-336` [C] | Phone pickers and lists must window their rows (50 at a time) |
| Phone sales in the second quarter of 2026: the US$100-200 band is the largest; brands that restrict background work dominate | IDC and Counterpoint figures as read in the personas doc, section 3 [C]; I did not re-read the analysts | Design for 3 to 4 GB of RAM; sync on open, not on a timer |
| Dashboard engine time at 250,000 vouchers is about 718 ms on the dev machine; on a phone it is a multiple nobody has measured | CLAUDE.md Phase 28 [C]; `--cpu-factor=4` is the harness's own projection | An owner pulse for a big book comes from the server (readiness section 5.6), not from the phone's engines |

### 5.2 Figures: lakh and crore, tabular, and why Mono is wrong on a phone

Measured from the installed font files and from Node 25.2.1 (ICU 78.1) [M]:

| Finding | Number |
|---|---|
| Plex Sans (latin subset) digit advance, all ten digits, weights 400 and 600 | **600 units per 1,000 em: tabular by default**; comma and period are 272 and 299; the subset exposes no `tnum` feature and needs none |
| Plex Mono digit, comma and period advance | 600 each |
| The rupee sign in the latin-ext subset | present, advance 605 (it is absent from the latin subset) |
| `₹12,34,56,789.00` set in Sans against Mono | 8,293 against 9,600 units: **Sans is 13.6% narrower**; at 28 px that is 232 px against 269 px; `₹12,34,567.00`: 6,821 against 7,800 (12.5% narrower) |
| The same figure at 28 px and 150% text size (42 px) | 348 px in Sans: wider than a 328 px column on a 360 px phone |

Consequences:

1. **Figures are set in Plex Sans on the phone in both schemes.** Today the light scheme already uses Sans for `--num` and the dark scheme uses Mono [C `src/styles.css:44`, `:325`]; the phone makes them the same.
2. Because Plex's digits are tabular only when Plex loads, **always add `font-variant-numeric: tabular-nums`** so the fallback face (first load, data saver) also aligns [J].
3. **Hero figures need a fit rule:** at large text sizes or for 11-digit amounts, show a compact form ("₹12.35 Cr") with the exact figure one tap away or in the label for screen readers; list rows keep exact paise.
4. **Never trust `Intl` for Indian regional locales.** Outputs on ICU 78.1:

| Locale and option | Output for the test number | Problem |
|---|---|---|
| `en-IN`, grouping, 1234567.891 | `12,34,567.891` | Correct |
| `en-IN`, compact short, 1,234,567 / 12,345,678 / 123,456,789,012 | `12L` / `1.2Cr` / `12KCr` | The last is a form nobody writes |
| `en-IN`, compact long | `12 lakh` / `1.2 crore` | Correct |
| `hi-IN`, compact short / long, 12,345,678 | `1.2 क॰` / `1.2 करोड़` | The short crore is an obscure abbreviation |
| `ta-IN`, compact long, 12,345,678 | `12 மில்லியன்` (million) | Wrong scale for India |
| `kn-IN`, grouping | `1,234,567.891` | Western grouping |
| `mr-IN` and `bn-IN`, grouping | `१२,३४,५६७.८९१` and `১২,৩৪,৫৬৭.৮৯১` | Native digits by default |

   The repo's own `fmtAmt` groups correctly with no locale [C `src/core/money.ts:2-14`]. **One formatter for every language; digits chosen explicitly; a house compact form** (below 1 lakh exact, then `₹5.6 L`, `₹4.56 Cr`, and above 100 Cr `₹1,234 Cr`).
5. **Which digits.** Bank SMS, UPI apps and printed invoices use Latin digits, so the default in every language is Latin digits; native digits are an opt-in setting [J]. A person who chose Hindi still reads `₹12,500`.
6. Words: `amountInWords` and `numToWordsIndian` exist [C `money.ts:196`, `:210`] and become the review sheet's echo (9.7).
7. Line height: Indic scripts carry marks above and below the line, and the compact `--lh: 1.25` would clip them [J]; the phone uses 1.4 to 1.5 and never applies tracked uppercase to Indic text.

### 5.3 Scripts and fonts

| Script | What the shipped fonts do | Options and sizes |
|---|---|---|
| Latin and the rupee sign | Plex Sans latin plus latin-ext in three weights: six files, 16 to 24 KB each, about 120 KB; Plex Mono six more files of 13 to 16 KB, about 86 KB [M, from `dist/assets`] | Keep Sans; on a phone leave the Mono files out of the first load |
| Devanagari | Not in the Plex files the app loads; falls to the OS face [C `styles.css:20-37`] | IBM Plex Sans Devanagari exists as a separate family, 7 weights, OFL, **77 to 79 KB per weight** for the Devanagari subset in woff2 (400, 500, 600) [W]; Noto Sans Devanagari is **50 KB at 400 and 54 KB at 600** [W] |
| Tamil, Telugu, Bengali, Gujarati, Kannada, Malayalam, Gurmukhi | Plex does not cover them (IBM lists Bengali and Tamil as in progress; Telugu and Gujarati not mentioned) [Ws]; falls to the OS face | Noto Sans per script, woff2 at weight 400 and 600: Tamil 15 and 16 KB, Telugu 36 and 39, Bengali 44 and 48, Gujarati 37 and 39, Kannada 28 and 30, Malayalam 24 and 26, Gurmukhi 11 and 12 [W, jsDelivr file listing of `@fontsource` 5.3.0] |

Recommendation **[J]**: **do not bundle Indic fonts in the first phone release.** Party names that arrive in any script already
render through the OS face, which is how the app has worked [C]. Test three real phones (Samsung, Xiaomi, vivo) with a Devanagari, a
Tamil and a Telugu party name; if any shows boxes or a wrong face, add the Noto file for that script, loaded by `unicode-range` so it
costs nothing for people who never see the script. **When the UI itself gets a Hindi translation** (5.4), bundle one Devanagari family
at two weights, loaded only when the language is Hindi: Noto (about 104 KB) if bytes decide, Plex Sans Devanagari (about 157 KB) if
visual match decides. A mixed Latin and Devanagari line looks best when both come from the Plex family [J].

### 5.4 Language and voice

- **No i18n exists** [C]. A phone shell is the cheapest place to start, because its strings are new and short. Rule: **labels are one or two words and verbs name results** (the same rule Apple gives for button titles [W]), which also makes translation cheap.
- **Order of languages** [J]: Hindi first (it is on Zoho Books, Vyapar and Khatabook's lists), then Marathi, Gujarati and Tamil by the first customers' geography; reps before owners.
- **Transliteration search.** Party and item names are typed in Latin by most users even when the person speaks Hindi; the picker should find "sharma" for a Devanagari "शर्मा" through the existing `alias` field, which is searched in every picker [C `src/types.ts:292`, `:492`]. Generating the alias automatically needs a transliteration step [J].
- **Voice.** Three levels, cheapest first. (1) The keyboard's microphone key in any text field, free, language by keyboard; Gboard had Hindi voice input by 2017, and whether Hindi works offline on a given phone is **not confirmed** [Ws]. (2) An in-app microphone on search and notes through the Web Speech API: Chrome on Android has it under the prefixed name since version 33 and Safari on iOS since 14.1; Chrome sends the audio to a server, so it needs a signal [W]; hide the button when offline. (3) A spoken order ("20 kg cheeni, 5 peti tel for Sharma") parsed into lines needs a language model and a review step; it is a later item that the MCP work may supply, and it must never submit by itself [J]. A voice **note** (audio attached to a visit or order) is simpler and is a shipped pattern in Zoho CRM [W].

### 5.5 WhatsApp: one tap or two

| Need | Mechanism | Taps | Constraint |
|---|---|---|---|
| A reminder or a statement summary to a party | `wa.me/<number>?text=<text>` with the party's number, built in `src/core/share.ts` [C `:124`] | 1 (WhatsApp opens on that chat; the person presses Send) | Text only; the official WhatsApp page could not be fetched, so the number rule rests on the vendor guides the repo cites [C `share.ts:7-12`] |
| A document (order confirmation, receipt, invoice, statement) | Web Share API with `files`: `canShare` with files is in Chrome Android 76, Safari iOS 14, Samsung Internet 11, not Firefox Android [W MDN compat data] | 2 (pick WhatsApp, then the contact; the number cannot be prefilled) | Needs a secure context and a tap [W]; needs a file |
| A document with the number prefilled | A short text with a **secure link** to a hosted copy, sent by `wa.me` | 1 | Needs a link service (the party portal's public link, `docs/research/customer-vendor-portal.md`) |
| Automatic messages | WhatsApp Business Platform; free replies inside 24 hours of a customer message, templates outside it [W Meta docs] | 0 | Needs a Meta business account and templates; not v1 |

**The missing piece is the file.** A browser cannot write a PDF from a page; the app keeps the print dialog and says so [C
`src/ui/pdfOut.ts:1-9`], and there is no `navigator.share` call [C]. Options **[J]**: (a) render the existing print markup to an
**image** in the page (small, shows inline in WhatsApp, good for receipts and order slips); (b) a server-rendered PDF (needs headless
rendering on the VM, which the party-portal research already assumes); (c) the hosted link. Order: image first (phase P1), link with
the portal, PDF last. Do not promise "PDF to WhatsApp" before one of these exists.

### 5.6 UPI and collections on a phone

- **The link** is `upi://pay?pa=<VPA>&pn=<name>&am=<amount>&tn=<note>&cu=INR`; the repo builds it, prints it as a QR, and leaves out the merchant-only fields on purpose [C `src/core/upi.ts:7-30`]. It is off unless the owner turns it on and uses the owner's own VPA [C].
- **Collect requests are going away.** NPCI told providers to stop person-to-person collect requests from 1 October 2025 [Ws, W MediaNama]; for merchants, one gateway blocked collect on Android and desktop from 28 February 2026 and keeps it temporarily on iOS, steering merchants to intent on mobile and QR on desktop [W Cashfree]. So a rep cannot rely on "send a request to the customer's UPI id"; **the customer scans the company's QR, or taps an intent link**.
- **The phone never learns the result.** On iOS a custom-scheme link reports only whether an app opened, not the payment status; gateways confirm by a server check [Ws]. A rep's "customer paid" is therefore a **claim**: the receipt is saved as pending verification until the bank credit is matched (personas section 8.6 sets the cash controls; the live-bank-feeds research supplies the match [C `docs/research/live-bank-feeds.md`]).
- **Limits.** Person-to-person UPI stays at ₹1 lakh a day; higher per-transaction limits (₹5 lakh) apply to listed merchant categories and verified merchants [Ws]. A ₹3 lakh B2B collection may not fit; the collect screen offers "bank transfer" with only the bank details the owner chose to publish (the repo's rule: never print bank details the owner did not choose [C CLAUDE.md, Phase 20-21 GOTCHAs]).
- **Cash at ₹2 lakh.** Receiving cash of ₹2 lakh or more from one person in a day, in one transaction or for one event is barred for the receiver, with a penalty equal to the amount [Ws]. Section numbering changes under the Income-tax Act 2025 (personas section 0 puts it at s.186 [Ws]); a CA must confirm before it is coded. The collect screen warns at 80% of the line and blocks at the line, per party per day [J]. No such guard exists in the repo today [C, search of `src/core` and `VoucherScreen.tsx`].
- **Show the QR large and keep the screen on** while it is displayed [J]; the Screen Wake Lock API is not checked here [unverified].

### 5.7 Sunlight, one hand, data saver

- **Contrast in sunlight.** WCAG 7:1 may not be enough outdoors: a 2018 paper on ambient light and text recognition suggests 18:1 for high illumination (abstract only, [Ws]); simulated-sunlight tests of phones show effective screen contrast collapsing to about 1.2 to 3.3 in 2012-era devices [Ws]. Our tokens, computed [M]: ink on the sheet 16.4:1 (standard) and **20.1:1 in high-contrast mode**; secondary `--dim` 5.8:1 standard and **12.2:1** high; accent 5.8:1; the thin field underline 3.3:1. So: **figures, party names and primary buttons are `--ink`, never `--dim`**; the high-contrast mode (already built, every text token at 7:1 or more [C `src/styles/contrast.css:1-12`]) gets a **one-tap "Sunlight" switch** in the top-bar menu and in the quick settings sheet [J]. Fills, not outlines, for primary buttons (white on teal is 5.8:1 [M]).
- **One hand.** 49% of observed touches were one-handed [W]; keep primary actions in the lower half, make the create button mirrorable, and let the keyboard-covered field stay visible (Android's layout guidance says to move the active input above the keyboard [W]).
- **Data saver.** `navigator.connection.saveData` exists only in Chromium-based Android browsers; Safari and Firefox Android do not have it [W MDN compat data]. So an **in-app Data saver switch**: no automatic photo upload on mobile data, no images, sync of the daily pack only on Wi-Fi by default [J].
- **OTP and identity constraints.** SMS needs DLT registration of the sender and each template in India [Ws]; WhatsApp authentication messages cost roughly ₹0.11 to ₹0.15 each, sources disagree [Ws]; e-mail needs a sender we do not have [C `docs/research/README.md`]. WebOTP exists on Chrome Android only (84 and later) [W]. This is why section 13 leads with the QR link and the invite link, not an OTP.

---

## 6. Technology: a PWA first, wrappers when triggered

### 6.1 What each route can and cannot do (verified rows only)

| Capability | PWA on Android Chrome | PWA on iOS Safari | Wrapped web app or native |
|---|---|---|---|
| Install | Manifest, 192 and 512 px icons, HTTPS; an install prompt event [W web.dev] | Add to Home Screen by hand; **no install prompt event** [W MDN compat] | Store install |
| Push | Web push, with action buttons [W] | iOS and iPadOS 16.4+, **only from a Home Screen web app**, permission after a tap, **no action buttons** [W WebKit, MDN] | FCM and APNs, up to four actions on iOS [W HIG] |
| Background Sync | Chrome and Samsung Internet from 49 [W] | **None** [W] | WorkManager, but OEM skins kill background work (personas section 3) |
| Storage durability | `navigator.storage.persist()` approved or denied by engagement, no prompt [W MDN] | A Home Screen web app keeps its own days-of-use count and is not expected to be purged; **script-written storage of a Safari tab can be deleted after 7 days without interaction** [W WebKit]; quotas about 60% of disk for browser and Home Screen apps, 15% for other apps [W] | **Capacitor: IndexedDB can be evicted by the OS; move to SQLite or the Preferences API** [W Capacitor docs] |
| Camera barcode | `BarcodeDetector` from Chrome 83 on Android [W] | Behind a flag since Safari 17 [W] | Native scanner libraries [unverified] |
| Voice | `webkitSpeechRecognition` since 33, server-based [W] | Prefixed, since 14.1 [W] | Native speech APIs |
| Share a file | Chrome 76 [W] | Safari 14 [W] | Native share sheet |
| Contacts | Chrome Android 80; Samsung Internet partial and removed in 22 [W] | Only behind a preference flag [W] | Native |
| Respect the OS text size | **No by default** (own slider; `meta text-scale` is opt-in and not stable) [W] | Only with system text styles [W] | Yes |
| Home-screen shortcuts, widgets | Manifest `shortcuts` (limited support, Android per general knowledge [unverified]); no widgets [unverified] | Neither [unverified] | Quick actions (four on iOS) and widgets [W HIG] |
| OTP autofill | WebOTP, Chrome Android 84 [W] | Not in the compat data [W] | Native SMS APIs [unverified] |
| Store presence and trust | None; a Trusted Web Activity can put the same site in Google Play (Chrome 72+, verified by Digital Asset Links, same developer) [W] | None; Apple rejects apps that are only a repackaged website (4.2) [W] | Yes; account deletion in-app (5.1.1(v)) [W] |
| First-load weight | **0.83 MiB gzipped today, to be cut** [M] | Same | 12 to 581 MB on the iOS listings [W] |

### 6.2 Options

| Option | For | Against | Verdict |
|---|---|---|---|
| **A. PWA, one codebase, two role shells** | Reuses `src/core`, the Dexie outbox, the sync client, the conflict and numbering code and the tokens; no store review; ships with the web deploy; first load is a link | Needs HTTPS and a lean shell; iOS push only from an installed app, no iOS Background Sync; no widgets; no OS text scaling; no store presence | **Do first** |
| **B. A Trusted Web Activity for Google Play** | Play Store presence (trust and discovery for Indian small-business buyers) with almost no extra code; updates ship from the web [W] | Android only; the host app cannot read the site's storage; Chrome 72+ required [W] | **Add when the owner wants a Play listing**; weeks, not months [J] |
| **C. Capacitor around the same web app** | Adds native plugins (scanner, push, biometrics, contacts) and the App Store; reuses the web client | iOS review wants more than a repackaged site [W]; IndexedDB in a WebView can be evicted, so the outbox must move to SQLite [W]; CORS origin changes (readiness 10.4) | **Only if** iOS owners need reliable push or the store, or Android scanning proves flaky |
| **D. React Native or Flutter rewrite** | Native feel, OEM-friendly background work | A second UI codebase; no IndexedDB, so the outbox, base revisions, conflict store and numbering are reimplemented (readiness 10.4); `src/core` can be shared, the UI and `src/data` cannot | **Not now.** The old roadmap line [C `docs/ROADMAP.md:32`] predates the offline-first engine and the PWA evidence; reopen it |

### 6.3 Triggers that justify a wrapper **[J]**

1. A buyer asks for a Play Store or App Store listing before they will install (→ B, then C).
2. iOS owners must receive approvals reliably and will not add the app to the Home Screen (→ C).
3. The camera barcode path is unreliable on the target phones (→ C or a WASM decoder first).
4. Reps need Bluetooth printing, call-log capture or background location, and the PWA path fails the test (→ C).
5. Text scaling and widgets become a stated customer need (→ C).

### 6.4 Making the web app light enough

- **An entry chunk per shell.** `main.tsx` for the desktop, a phone entry for each role, shared core; report screens, payroll, GST returns and the 99-report catalogue load only on the desktop shell. Budgets: Field 350 KB gzipped first-load JS, Owner 500 KB, later screens 100 KB each [J, from the Russell table in 5.1].
- **Service worker app shell** cache-first; data network-first with a cached fallback for the owner pulse and stale-while-revalidate for the party list [W web.dev offline cookbook].
- **Persistent storage:** request it after the first successful sync; show "not safe" on iOS when the app is not installed [J, personas section 7.5 says the same].
- **Windowed lists** (50 rows at a time) and a trimmed picker index; no mount of every ledger on open [C PERF.md].
- **Measure on the reference device:** a Galaxy A-class 4G phone on a throttled profile of about 6 Mbps and 85 ms, Lighthouse mobile preset, INP from the web-vitals library [W, J].

---

## 7. One app or two?

| Evidence | Reading |
|---|---|
| Vyapar: one app, role presets including Salesman, invite by phone or e-mail, a 4-digit passcode per person, multi-device sync is a paid feature [Ws] | One app with roles is workable and understood in this market |
| Marg: **two apps**, eOrder (12 MB) for the rep and eOwner (49 MB) for the owner [W] | The ERP-vendor pattern; the rep app is small by construction |
| Pipedrive, Zoho CRM: one app for everyone; offline limited to what is stored on the device (Zoho: the modules depend on the edition) [W] | Modern pattern |
| Zoho: Books and CRM are separate apps because they are separate products [W] | Not a rep-versus-owner split |
| Our own data: a rep must hold a projection, an owner may hold the books (personas section 8, readiness 5.6) | The data scope differs, not the codebase |

**Decision [J]: one codebase, one installable app, two shells chosen by the signed-in role.** The membership role (owner, admin,
member) plus a field capability decides which shell opens; an owner can open "Field view" to see what a rep sees (it is also the
fastest way to test the rep experience). Why: one sign-in and one install for an owner who also sells; one place to fix bugs; the
Field shell excludes reports, dashboard and payroll code, so it stays small enough for a 3 GB phone (6.4). **What would flip it to
two store listings:** a Play listing that must be named for reps ("Comsky Field") or a customer's device management that wants a
separate package. Both are a build target away (a second Trusted Web Activity pointing at the Field entry), not a rewrite.

---

## 8. Navigation model

### 8.1 Two shells

```
OWNER SHELL                                         FIELD SHELL
top bar:  Company v | period chip   search  sync·  me     top bar:  day state   search  sync·  me
tabs:     Home | Parties | Books | Reports | Inbox         tabs:     Today | Parties | Orders | Collect | Leads or Stock
create:   one contextual button (Create)                   create:   one contextual button (New order)
```

- **Five tabs at most, no More** (Apple and Material both say so [W]). The Field shell's fifth tab is **Leads** for a company that uses the CRM feature and **Stock** otherwise; the set is chosen by company features and role at sign-in, never disabled afterwards (Apple: do not hide or disable tabs [W]).
- **Create is not a tab.** It is one 56 px button that changes with the screen: Owner Home offers the create sheet (the 28-entry quick-create registry, filtered by rights [C `src/ui/quickCreate.ts:64-98`]); a Party card offers "Receive payment" and "New order"; the Field shell's default is New order.
- **Search is the global command**, reached from the top bar and by pulling down on Home (the Superhuman gesture [W]), with a visible icon so the gesture is never the only way.
- **The 12-group menu** (Pinned, Dashboard, Sales ... Settings [C CLAUDE.md]) is not rebuilt as a drawer. It becomes the Search empty state's "Browse" list (full screen, same rows, same rights and feature gating) for the long tail.

### 8.2 Stack, sheets, back

- A **push** stack inside each tab, remembered per tab. The desktop's navigation rules (replace when clean, ask when dirty, never pile up screens [C `src/core/navPlan.ts`, CLAUDE.md Phase 26]) still hold, with **one change: a phone draft autosaves, so leaving a half-entered order never prompts** (9.6). The prompt remains for the review sheet's discard.
- **Sheets** for short choices (period, filters, reasons, the review step); detents medium and large, a grabber, Cancel leading and Done trailing, swipe down confirms before discarding work [W]. **No sheet opens a sheet**: the party and item pickers are full-screen routes.
- **Back:** the visible back button plus the system back; each pushed route adds a history entry so the Android back gesture and the iOS edge swipe return to the previous screen instead of leaving the app [J]. We never rely on a custom edge swipe [W].
- **Deep links** from a notification, a WhatsApp link or a shortcut open the target inside the right tab with a real back stack ("Approval 4821" → Inbox → Home), and a signed-out deep link keeps its destination through sign-in [J].
- **Screens without a phone treatment** (most of the 99 reports, GST returns, masters beyond party and item) are not routable in the phone shell. They show one card, "This screen needs a wider display", with two actions: copy a link, and (where it exists) share as a file. This takes the same stance as Xero's mobile app, which omits the Balance Sheet [Ws], and avoids the 938 px floor.

### 8.3 Tablet and keyboards

A tablet or a phone with a Bluetooth keyboard uses the **desktop shell** when the width reaches the desktop's floor; in the middle widths the phone shell gains a rail instead of tabs (Android switches between a navigation bar and a rail by window size class [W]; the breakpoints, compact below 600 dp and medium from 600 to 839 dp, are general knowledge and [unverified]). The keyboard layer is untouched for `(pointer: fine)`: keycaps, the F-key rail and Go To stay as they are. On `(pointer: coarse)` the keycaps and the hold-Alt hints do not render [J].

### 8.4 The state vocabulary (the offline contract made visible)

Every figure carries an **as at** stamp (Dashboard v2's rule [C CLAUDE.md Phase 28 GOTCHAs]); every item the phone holds carries one of six states, shown as a small chip and an icon (never colour alone):

| State | Meaning | Where it shows |
|---|---|---|
| **Draft** | Typed on this phone, not submitted | Orders, Today ("1 draft") |
| **Queued** | Submitted, waiting for a signal | Orders, Collect, sync sheet |
| **Sent** | The server has it | Orders |
| **Held** | The server needs a person (credit, discount, approval) | Orders, Inbox, notification |
| **Rejected** | A person said no, with a reason | Orders, notification |
| **Provisional** | Numbered on the phone, number may change at sync | Receipts, orders |

The **sync dot** in the top bar (green, amber, red) opens a sheet: last sync time, N unsent, size of the pack, a retry button, and what is stuck and why. It replaces the desktop's status bar [C]. Android's offline-first guidance asks for exactly this: local data first, an explicit error state, and a visible offline state for online-only writes [W].

---

## 9. Entry on a phone

The principle: **the screen asks one thing at a time, shows what the person needs to decide, and makes the dangerous thing hard and the
safe thing fast.** Sections 9.1 to 9.8 are for the Sales Order and the Receipt; the same pieces carry an Expense or a Quick Invoice
later.

### 9.1 Party picker (a full-screen route)

- **Opens on recents**, not on the whole list, then a search box with the keyboard up. Each row: the name (two lines, middle ellipsis), the city, and on the right the **outstanding with Dr or Cr and, when overdue, the overdue amount in the warning colour with its words** ("₹40,000 due · 12 days late"). A **credit headroom chip** ("₹60,000 left", "over limit") comes from the credit engine [C `src/core/creditLimit.ts`].
- **Sort is stable** (recents by last use, then name): the Go To rule that the top row never jumps under a blended score applies [C `src/ui/paletteRank.ts:1-14`].
- **Finds by alias and phone digits**, and by Latin spelling of a Devanagari name (5.4).
- **Right of the row, a phone icon** (call) and a WhatsApp icon: the two most common next moves from a party list in this market.
- **Fresh data:** the figures come from the pack and say "as at 09:12 plus 1 unsent" (readiness 5.6).
- **Add a party** is "New customer" at the top of the list: a short form (name, phone, city or GSTIN with the checksum check [C `src/core/gstin.ts`]) that creates a **draft party** for the office to complete (personas section 7.6, rule 7).

### 9.2 Item picker (a full-screen route)

- **Opens on "Usually for this party"** (items from the party's last orders), then recents, then search. Each row: name, unit, **rate for this party's price level** with the level's name in small words, **stock status** ("in stock", "low", "out" or an exact figure per the owner's setting; personas section 7.6, rule 6), and a thumbnail only if the owner has photos (not v1).
- **Barcode:** a camera button opens a scanner on the first press (permission in context). On Android Chrome the page can use `BarcodeDetector` (Chrome 83) [W]; elsewhere a WASM decoder library would be needed [unverified: not evaluated]. Hardware scanners that type like a keyboard (Vyapar lists Bluetooth and USB scanners [Ws]) need `inputmode="none"` on the scan field so the on-screen keyboard stays down [W MDN]. **The data model needs a `barcode` field on `StockItem`** (several codes per item for packs; today only `alias` and `partNo` exist [C `src/types.ts:492-493`]), appended after the last field and kept out of the middle of any habitual Enter walk [C CLAUDE.md Phase 24 GOTCHAs], with an O(1) lookup built at load.
- **Window the list** (50 rows) and never mount every item (the measured gap in PERF.md [C]).
- **Tap adds one unit and shows the line card**; a second tap on the same item in the picker increments (the picker stays open for a run of items, with a "Done (3)" button).

### 9.3 Lines: quantity, rate, discount and tax, all visible

Each line is a card, not a table row:

```
Sugar M30 25 kg bag                         [stepper  -  5  +]
₹1,420 x 5 = ₹7,100        Retail level
Disc 2%  -₹142 | GST 5%  +₹348              Line ₹7,306
```

- **Stepper** buttons are 48 px with long-press repeat; a tap on the number types it (`inputmode="decimal"`, or `numeric` for a unit that takes no fraction; unit suffix rules from `parseQty`, which deliberately never reads lakh or crore magnitudes on a quantity [C `src/core/money.ts:162`, CLAUDE.md Phase 20 GOTCHAs]). Alternate units show both ("2 box = 24 Nos") [C `Unit.baseUnitId`, `StockItem.altUnitId`].
- **Rate and discount are visible and editable only within the role's cap**; an edit above the cap becomes an approval request, not an error (personas section 7.7). The rate shows where it came from.
- **Tax is always visible:** per line, GST rate and amount; the footer shows the total **including** GST and, in the review sheet, CGST and SGST or IGST by place of supply (the existing `isIntraState` decides [C `src/core/gst.ts:9`]).
- **Swipe left on a line deletes with an Undo toast; a row button does the same** (Apple and Android both require the visible alternative [W]).
- **Sticky footer:** "5 lines · ₹1,24,500 incl. GST · Review". The figure is in lakh grouping with tabular figures; if it cannot fit at the chosen text size it switches to the compact form (5.2).

### 9.4 Ghost recall: "same as last time"

The engine already exists: it finds the party's latest comparable voucher (not optional, not the one being edited, not dated after this one) and re-derives it against today's masters, with deltas for changed prices, changed GST and missing items [C `src/core/recall.ts:19-50`]. On a phone:

- After the party is chosen, a **card above the lines**: "Last order · 12 Sep · 6 lines · ₹18,400 · 3rd time this quarter" with a **Repeat** button and a line of differences ("2 prices changed, 1 item discontinued").
- Repeat fills the lines in **ghost style** (italic, dim, a dashed edge, as on the desktop and in the mockup [C `docs/design/six-keystrokes-mockup.html` `.gh`]) and each carries its delta badge; the person confirms a line by touching its stepper or a "Confirm all" button. **A recalled value that skips a normal prompt must be held until it has been through it** (the Phase 20-21 gotcha [C CLAUDE.md]).
- Ghost recall is the strongest "fast order" feature we have, and none of the competitor texts I read (section 4) describes an equivalent; it is on the critical path to the 60-second order target (personas section 7.3).

### 9.5 Voice and dictation

Level 1 (the keyboard's microphone) is free everywhere there is a text field; the review sheet's narration and the visit note use it. Level 2 (an in-app microphone on search and notes) is Chrome Android and Safari iOS only and needs a signal (5.4). Level 3 (a spoken order) is deferred. **A voice note** recorded in the page is stored as an attachment and uploaded when online (the attachments design [C `docs/research/attachments-drives.md`]); it is the cheap, honest version of "voice order" and works offline [J].

### 9.6 Drafts and undo

- **Autosave a draft on every change** (300 ms debounce) to local storage that is not the outbox, and restore it on resume. Android may kill a background tab on a low-memory phone, so a half-typed order must survive; this is the one place where the phone differs from the desktop's "Leave this entry?" prompt [C `src/state/dirty.ts`] [J].
- **A draft is a state** (8.4) shown on Today; it is deleted on submit or on explicit discard.
- **Undo after submit:** a toast "Order SO-1042 queued · Undo" for 10 seconds withdraws it from the queue. Once the server has it, "Withdraw" is a visible action on the order until it is accepted [C `src/core/undo.ts` is the desktop mechanism; the phone reuses the outbox]. No shake-to-undo (Apple says not to redefine system gestures [W]).

### 9.7 Guardrails against a fat finger with money

| Risk | Guard | Basis |
|---|---|---|
| A zero too many (12,500 typed as 1,25,000) | **Live echo under the field** in lakh grouping and, in the review sheet, the **amount in words** ("One lakh twenty-five thousand rupees only"). The figure is shown twice in two forms, and words are hard to mistype the same way | `fmtAmt`, `amountInWords` exist [C `money.ts:2-14`, `:210`] |
| A figure wildly out of pattern | **Magnitude check against history:** compare a line quantity with the party's usual for that item, and an order total or receipt with the party's median; above a threshold (proposed 10 times, or more than 5 times the rep's largest) show one inline sentence, "Usually ₹12,500. This is 100 times that." and require a second tap | [J]; extends the existing overridable warnings (negative cash, negative stock, duplicate number, credit limit) [C `BulkEntry.tsx:1-60`] |
| A wrong decimal | Unit-aware: no fraction for "Nos"; a decimal comma or point typed in a whole-unit field is refused with the reason | `parseQty`, unit rules [C] |
| Saving twice | **Duplicate check:** same party, same total, within 15 minutes or within the same draft session: "Same as order SO-1041, sent 4 minutes ago. Send another?" with the verbs "Send another" and "Do not send" | [J]; the repo has a duplicate voucher-number warning [C CLAUDE.md Phase 6] |
| Credit breach | **Projection in the review sheet:** "Outstanding ₹1.2 L + this order ₹1.25 L = ₹2.45 L; limit ₹2 L". For a rep, the button is "Ask owner to approve", never "Continue anyway" (the desktop override with a reason stays an owner action) | `creditProjection` [C `src/core/creditLimit.ts`]; personas section 7.7 |
| A cash receipt at the legal line | **Warn at 80%, block at 100%** of ₹2 lakh per party per day for cash; point to UPI or a cheque | [Ws] (CA to confirm); no guard exists today [C] |
| A receipt larger than the bill, or negative | Allocation cannot exceed the open bills' sum (the excess is "on account", shown in words); no typed sign | `openBillsFor`, single-entry shape [C] |
| "Customer paid by UPI" tapped without payment | The receipt is **Provisional, pending verification**, never reduces the party's reported outstanding until matched; the phone shows "₹10,000 collected, awaiting bank" | [Ws] (no status on iOS); approvals design (`optional` bit) [C `docs/research/approvals-workflows.md:18-22`] |
| A reflexive confirmation | **Confirm only the irreversible or the large**; the sheet names the figure and the party and its buttons are verbs, **no default**; everything else is undone by the toast | NN/g; Apple alerts [W] |
| An accidental swipe delete | A swipe never deletes without an Undo toast; destructive swipe is not a full-swipe action | Apple gestures, Material snackbar [W] |
| Fat finger on a stepper | Steppers 48 px with 12 px between; a long-press repeat that accelerates but caps; the total updates under the thumb | Section 3.2 |
| Wrong party | The party name and its outstanding are large in the review sheet's header; a location-based "you are at another stop" warning is not part of v1 | [J] |

### 9.8 Money input spec

One component for amounts and quantities: a text field with `inputmode="decimal"`, `autocomplete="off"`, `spellcheck="false"`,
`enterkeyhint="next"` (or `done` on the last field) [W]; a `₹` prefix **outside** the input and tolerated if typed [W GOV.UK]; the live
echo beneath in lakh grouping; shorthand accepted as on the desktop (`5L`, `1.2cr`, `50000/-`; accepting more is never a regression,
`(1000)` is negative; a quantity never takes lakh or crore) [C `src/core/money.ts:92-135`, CLAUDE.md Phase 20-21 GOTCHAs]; sign by a
visible Dr/Cr or increase/decrease control, **never a typed minus** because the decimal keypad may not have one [W]. A custom in-app
keypad was considered and **rejected**: it would break dictation, assistive technology and the system's number keypad that people
already know; a row of **"00", "000", "L", "Cr" chips above the keyboard** is a cheap speed-up to try in the spike [J].

### 9.9 Collect: the receipt flow

1. From a party card or the Collect tab: **bills list** with the oldest first, each with its due date, days late and balance; a "suggest allocation" button fills oldest-first; the person can set an amount per bill. (The bill-wise engine and the single-entry projection exist [C `src/core/billwise.ts`, `src/core/singleEntry.ts`].)
2. **Mode:** Cash, UPI, Cheque, Bank transfer. Each shows only what it needs (a cheque asks for number and date; UPI shows the QR; bank transfer shows only owner-chosen details).
3. **Amount** with the echo and words; the cash guard at 80% and 100% of ₹2 lakh.
4. **UPI:** the QR is large with the amount and the invoice number in the note (`am`, `tn`); "Customer paid" saves a **Provisional** receipt with the UTR if the customer reads it out; the owner's bank match confirms (5.6).
5. **Receipt to the party:** an image of the receipt by one tap through `wa.me` with a link, or two taps through the share sheet (5.5); the wording says "received, subject to realisation" for non-cash [J, CA to confirm].
6. **Offline:** works fully; the receipt is Queued/Provisional with a local number and is claimed on return (the numbering claim exists [C `backend/src/numbering.js:1-30`]).

### 9.10 What the code must do first

Extract, with the assertion suites that exist for the desktop pinned in place: `composeSalesOrder(lines, party, masters)` and `composeReceipt(allocation, account)` as pure functions in `src/core`, called by `VoucherScreen`'s `save`, by Bulk Entry and by the phone. Until then the phone would be a third place that knows how a voucher is built [C `VoucherScreen.tsx:2540`, `:2685-2702`; `BulkEntry.tsx`]. The master-resave gotcha applies: a writer that rebuilds a document from fields it knows drops the rest, so the phone spreads the stored document first [C CLAUDE.md Phase 24 GOTCHAs].

---

## 10. Reports and the dashboard on a small screen

### 10.1 Rules

1. **Summary first.** A hero figure, one sentence in plain words, then the detail on demand. Apple says to let people reveal detail gradually and to size a small chart for glanceable use or as a preview that opens a larger one [W].
2. **Every number keeps its "as at" and its window**, and equals the report it opens on the desktop to the paisa (the Phase 28 blocker rule [C CLAUDE.md]). The phone's drill opens the phone version of the same report from the same engine.
3. **Charts have at most six marks, a selected-mark callout and a table twin.** Select by tap (the callout shows value and name), drill by a second tap on the callout or a "View" row. The existing kit already ships a table twin and keyboard focus; it needs the select-first model (today a tap shows the tip and drills at once [C `Columns.tsx:190-195`; J]). Apple's chart guidance also asks for accessible chart labels and a descriptive headline [W].
4. **Lists beat charts for action.** The overdue list, the party list and the approvals feed are rows with swipe actions, not charts.
5. **Wide tables become cards**, or a sticky first column with horizontal scroll as the fallback for the long tail; never a 938 px page.
6. **Sharing** a report is a file or an image through the share sheet (5.5); the xlsx writer already exists [C `src/core/xlsx.ts`].

### 10.2 The 22 implemented widgets on a phone

Counts: 11 translate as they are, 6 become two-line rows with actions, 3 need a chart redesign, 2 are replaced. Source: the registry rows (form, sizes, engine, drill) [C `src/ui/dash/widgets/registry.ts:32-200`]; the "Phone treatment" is my judgement.

| Widget (id) | Form today | Phone treatment | Notes |
|---|---|---|---|
| **A. Translates as it is (11)** | | | |
| Today (`today`) | strip | **Home, pulse strip**: sales, money in, money out today | Owner Home row 2 |
| Cash & Bank (`cash`) | card S/M/L | **Home hero**: total with an as-at stamp; tap opens accounts | The first thing the owner reads [personas 6.4 job 1] |
| Receivables (`receivables`) | card | **Home card**: headline plus the overdue split bar; segments are 48 px targets | The bar opens the Collect list filtered to that bucket instead of the Bills Receivable report (a deliberate change from the desktop's drill); the list's total must equal the segment, the Phase 28 rule |
| Payables (`payables`) | card | Home card, secondary position | Flipped sign shows Cr or Dr in words, as today [C `ArCard.tsx`] |
| Overdue (`overdue`) | tile | Folded into the Receivables card | Redundant on a phone |
| Net profit (`profit`) | tile | Optional Home tile | Owner choice |
| Sales (period) (`sales`) | tile | Optional Home tile; drills to the Group Summary as now | Same figure as the report (Phase 28 rule) |
| Purchases (period) (`purchases`) | tile | Optional tile | |
| Stock Value (`stock`) | tile | Optional tile | |
| TDS to deposit (`tdsPayable`) | tile | Optional tile | |
| GST payable (`gstPayable`) | tile | **Home tile with the due date** | Today it says "for the open period"; the filing-month card with statutory dates is deferred [C CLAUDE.md]; the phone owner wants the date |
| **B. Becomes two-line rows with actions (6)** | | | |
| Overdue invoices (`overdueList`) | list | **The Collect tab's list and Home's top three**: party, bill, days late, amount; swipe: Call, WhatsApp reminder, Record receipt | The single most valuable list; today bespoke, marked `REPLACE-WITH-LISTQUERY` [C `OverdueList.tsx:4`] |
| Top customers (`topCustomers`) | list | Ranked rows (name, amount, share bar) | Owner-only |
| Top items (`topItems`) | list | Ranked rows | |
| Top expenses (`topExpenses`) | ranked bars | Ranked rows, five | Phone-friendly already |
| Stock alerts (`stockAlerts`) | list | "Needs you" feed rows | Owner and store |
| Recent vouchers (`recent`) | list | The Books tab's feed (grouped by day) and Home's "Recent" | Approvals join the feed |
| **C. Chart redesign (3)** | | | |
| Cash flow (`cashflow`) | chart M/L | Six monthly columns, tap selects, a "net this month" sentence | Remove hover tips |
| Income and expense (`incomeExpense`) | chart | One net-profit headline plus two stacked bars for the last three months | |
| Sales and purchase trend (`trend`) | chart L | A sparkline in the Sales tile plus a six-month column view in Reports | |
| **D. Replaced (2)** | | | |
| Get started (`setup`) | strip | Replaced by the first-run link flow (section 13) | |
| Quick Entry and Reports (`quick`) | card | Replaced by the create button, the quick-actions menu and Search | |

Two catalogued widgets with no implementation (`watchlist`, `highlights`) stay off. Rights and features gate each widget exactly as on the desktop through `widgetVisible` [C `registry.ts:200+`]; **a widget that is hidden, denied or under another screen is never computed** (the Phase 28 rule) matters more on a phone, where compute is slow [C CLAUDE.md].

**Default owner Home (top to bottom):** Cash & Bank hero → Today strip → "Needs you" (approvals, promises due, stock alerts) → Receivables with Overdue top three → Payables → GST payable tile → Cash flow. This is the Zoho Books and QuickBooks pattern (cash, owed, overdue, to-do) [W], with approvals first, which none of the pages I read describes.

### 10.3 The long tail of reports

| Desktop report | Phone form |
|---|---|
| Day Book, registers | A feed grouped by day, type chips, a voucher detail sheet |
| Ledger vouchers | Statement cards with a running balance; share as image |
| Outstandings and ageing | Party cards that expand to bills; the ageing bar on top |
| Trial Balance, Balance Sheet, P&L | Collapsible groups with sticky Dr and Cr totals; the condensed view by default (the Detailed and Condensed switch exists [C CLAUDE.md Phase 14]) |
| Stock Summary | Item cards (quantity, value) with search |
| GSTR-3B summary | Summary tiles, due date, share; **GSTR-1 JSON and other filing exports stay on the desktop** |
| Cost centres, payroll, TDS registers, e-invoice register | Desktop only through P1 to P3 |

---

## 11. Keeping the keyboard-first feel on touch

| Desktop speed feature | Touch equivalent | Notes |
|---|---|---|
| Go To (`Alt+G`, `Ctrl+K`) with strict-tier ranking and a stable top row | **Search**: top-bar icon and pull-down on Home; the same `rankPalette`; results typed as party, item, voucher, report, action | The 16 ms keystroke budget [C PERF.md:74-80] applies; windowed results |
| Quick access: Today, Pinned, Recent | **Search's empty state**: Pinned, Recent, Today's counts [C `state/recents.ts`] | Pins are per device today; a synced pin list is a small server addition |
| F-key rail (152 px) | A bottom action bar of at most three buttons on entry screens and a "More" sheet for the rest | Never hides a button's function [C CLAUDE.md Phase 27] |
| `Alt+Q` universal plus | **The contextual create button** (8.1) | Same registry, same rights |
| Ghost recall | The "Repeat last" card (9.4) | |
| Bulk Entry (paste a block from a sheet) | **Share to Comsky** from WhatsApp or the gallery (Web Share Target on Android, [unverified: the manifest member's support was not confirmed, compat data returned 404]); creates a draft or an attachment | A later item |
| Hold-Alt hints, keycaps | Touch-and-hold on a row shows its actions (Apple: touch and hold reveals more [W]); keycaps hidden on `(pointer: coarse)` | A gesture is never the only path |
| Hotkey letters | Swipe actions plus visible row buttons | |
| Undo `Ctrl+Z` | Undo toast with a button | 3.7 |
| Saved views and Reports Center stars | A star on a report card; "My reports" at the top of Reports | |

**Notification actions.** Android Chrome can show action buttons on a web push (Approve, Reject; the cap is the browser's) and iOS cannot [W]. The notification is generic by default ("An approval is waiting") and the card opens at two taps from the lock screen (personas section 6.7).

**Quick actions on the icon.** At most four (Apple's cap [W]); manifest `shortcuts` on Android [W]; proposed: Owner: Approvals, Cash today, Search, Create; Field: New order, Collect, Search, Today.

**Budgets [J]:** tap to visible feedback 100 ms; INP at or under 200 ms at the 75th percentile on the reference phone [W]; cached Home on cold start in under two seconds with a stale stamp (personas section 0 target); a known party's order of five lines in 60 seconds offline (personas section 0 target).

---

## 12. The existing design language on a phone

Source of truth: `docs/design/six-keystrokes-mockup.html`, `src/styles.css`, `src/styles/accent.css`, `src/styles/contrast.css`,
`src/styles/light/*` [C].

| Token or rule | Desktop today | Phone | Why |
|---|---|---|---|
| Base size | 13.5 px compact, 14 px comfortable; rows about 21 px [C `styles.css:65`, `:90-120`] | Body 16 px; list amount 17 px; hero 28 to 32 px; one density (comfortable); text size 100 to 200% | 16 px input floor, 17 pt body, 200% [W] |
| Control size | n/a | 48 px rows and buttons, 44 px absolute floor, 12 px gaps | 3.2 |
| Row height | 21 px | 56 to 72 px two-line rows | Touch |
| Figures | Plex Mono in dark, Plex Sans in light | **Plex Sans in both**, `tabular-nums` set | About 13% narrower [M] |
| Colour | Teal `#0b7266` (light) and `#3ddbc6` (dark); eight validated accents; neutrals checked against 13 surfaces [C `accent.css`, `contrast.css`] | Same tokens, same accents; **default "same as my phone"** | HIG dark mode [W] |
| Sunlight | High contrast mode exists in Settings | **A one-tap "Sunlight" switch** that sets it | 5.7 |
| Radii | 3 to 6 px (light scheme 4 to 12) [C `styles.css:323`] | Keep the family: 8 px cards, 12 px sheets, 999 px chips | Same voice |
| Elevation | `--bg`, `--panel2`, `--panel`, `--raised`, `--field` [C `styles.css:161-166`] | Same; a sheet is `--raised` over a scrim | Apple base and elevated sets [W] |
| Accent presets, Settings window | Eight presets, text size, font, contrast | Same settings on a phone-sized Settings (text size to 200%, language, hand side, data saver, sunlight, app lock, devices) | Opt-in and asserted to change nothing when unset [C CLAUDE.md Phase 27] |
| Keycaps, F-key rail, left rail | Everywhere | Hidden on `(pointer: coarse)`; rail becomes tabs; F-keys become the action bar | 8.1, 8.3 |
| Ghost recall look | Italic, dim, dashed block, delta chip | The same, bigger | Mockup |
| Mono eyebrow caps | `fs-2xs` tracked uppercase labels | **Not on Indic text**; 12 px minimum elsewhere | 5.2 |
| Motion | 90 to 120 ms feedback | Same; `prefers-reduced-motion` honoured [C `styles.css:503`] | |
| Fonts | Self-hosted Plex, latin and latin-ext | Sans only on first load (about 120 KB for three weights, latin and latin-ext) | 5.3, 6.4 |

New CSS keys on `@media (pointer: coarse)` and `(hover: none)`, and on the shell attribute, so that **a person who never opens the
phone shell sees exactly the desktop** (Phase 27's opt-in rule) [C CLAUDE.md]. Implementation notes to verify on a device **[J]**:
`viewport-fit=cover` with `env(safe-area-inset-*)`, dynamic viewport height units, `touch-action: manipulation` on buttons,
`overscroll-behavior: contain` on scroll panes so pull-to-refresh is ours, a `theme-color` per scheme.

---

## 13. First run and sign-in

### 13.1 Principles

- **A salesperson never types an API key, a server address or a company id.** Keys are tenant-wide with only read and write scopes and are for integrations [C `backend/src/platform.js:83-89`, `src/ui/admin/ApiKeys.tsx`]. A phone gets a **device session** in its own token family (the family and its revocation already exist [C `backend/src/tokens.js`]).
- **Ask nothing until it is needed** (Apple onboarding, privacy, accounts [W]): no permission prompt on launch; camera at the first scan; microphone at the first voice use; notifications after the first approval; location only if the owner enabled check-ins.
- **Show what is happening**: pack size, progress, what will be on the phone, and what will not (cost, other reps' parties).
- **Never block use on a download**: the owner sees the pulse first (a small server answer); the rep's pack streams in the background and the party and item pickers open as soon as masters are in (resumable by cursor [C readiness 5.3]).
- **Offer deletion and sign-out in the app** (App Store 5.1.1(v) for any store build [W]).

### 13.2 Flows

| Flow | Steps | Notes |
|---|---|---|
| **A. Owner, already has an account** | Open the link, sign in with e-mail and password (exists [C `backend/src/auth.js:40-52`]), choose the company, see the Home pulse; install prompt after the first approval or the third visit, not on arrival; notification permission after the first approval | An owner without a company yet is told to start on a computer, with a link to send themselves |
| **B. Link a phone from the desktop (QR)** | Desktop, signed in as owner or admin: Team and Devices → Add a phone. It shows a **QR plus a 6-digit confirm code**, valid for about 5 minutes and **single use**. The phone's camera app opens the link; the page shows "Link this phone to Acme Traders as Ravi (Field)? Code on your computer: 483 291". After the person confirms the code, the **desktop shows the device name, model and rough place and asks Approve**. The phone then receives its own tokens, picks the shell by role and starts the pack | Shape of the OAuth device grant (RFC 8628): a long unguessable device code on the phone, a short user code, the complete link for the QR, polling with a 5-second minimum and back-off, short life, rate-limited guesses [W]. **QR login is open to a phishing relay (QRLJacking)**, so: a confirm prompt with session details, a quick expiry, one use, HTTPS with HSTS, no reuse of the scanner for anything else [W OWASP]. The owner is notified of every link. No endpoint for this exists [C readiness section 3.5 lists the invite design] |
| **C. A rep joins by invite** | Owner: Team → Invite (name, mobile number, role Field, parties scope, discount cap). The app builds a **WhatsApp message with a join link** through the existing `wa.me` builder [C `share.ts`]. The rep opens it, sees the company name and role, sets a **PIN** (the policy checker exists [C `src/core/passwordPolicy.ts`]; Vyapar uses 4-digit passcodes [Ws]), and the owner's phone shows "Ravi wants to join: Approve". Then the pack | Works for a person with no e-mail. The server-side person, role and scope are readiness items B2, B3, B5 [C readiness section 12.2] |
| **D. Returning** | App lock with the device's biometrics through WebAuthn where available, else the PIN; auto-lock after a short idle; a hide-amounts switch on the owner Home | WebAuthn needs HTTPS [W]; Apple prefers passkeys to passwords [W] |
| **E. Lost phone** | Owner: Team and Devices → Revoke; the family's refresh and access tokens die; cached sensitive figures expire by the owner's setting; unsent items are exported or shared first (readiness 5.7) | A device list needs the `sync_device` table's label and last-seen to be exposed [C `backend/src/sync.js:164-171`] |
| **F. First order, first receipt** | A sample party and item offered in a "Try it" mode that writes nothing to the books; a coach mark on the stepper; nothing else | Apple: onboarding short, optional, in context [W] |

**What the QR flow deliberately does not do:** it never carries an e-mail, a password or an API key; it does not let a phone mint
another device's link; and the desktop approval is not optional for a rep.

---

## 14. Screen inventory

Phases (section 16): **P1** owner read-only, **P2** owner actions and approvals, **P3** field MVP, **P4** later. "Builds on" is what
exists in the repo [C]. Offline column: **Cache** = last-known, stamped; **Full** = works without a signal.

### 14.1 Owner shell

| ID | Screen | What it shows | Key interactions | Offline | Builds on | Phase |
|---|---|---|---|---|---|---|
| O1 | Sign-in and link | E-mail sign-in; QR link; company choice; pack/pulse progress | Scan, confirm code, approve on desktop; install prompt late | n/a | `auth.js`; device grant design | P1 (sign-in); QR link with O14 in P2 |
| O2 | **Home** | Cash & Bank hero with as-at; Today strip; Needs you (approvals from P2); Receivables with overdue split; Payables; GST payable with due date; Cash flow | Pull to refresh; tap a card to drill; period chip (sheet); **hide amounts**; long-press a figure to copy the exact amount; swipe a feed row to snooze | Cache, stale chip after a set age | Widget registry, engines, `dashPeriod` | P1 |
| O3 | Cash & Bank | Accounts, balances, unreconciled count and last feed time when bank feeds exist | Tap an account for its statement | Cache | `cashBankBook`; live-bank-feeds design | P1 |
| O4 | **Collect list** (Receivables) | Ageing chips (Current, 1-30, 31-60, 61-90, 90+); party rows with overdue amount and days | **Swipe: Call, WhatsApp reminder, Record receipt**; chip filters; sort by oldest or largest; note a promise-to-pay date | Cache; reminder composes offline | `outstandingBills`, `share.ts`, billwise | P1 |
| O5 | **Party card** | Name, call and WhatsApp buttons, balance (Dr/Cr), credit limit bar, overdue; tabs Bills, Statement, Activity; last five vouchers | Receive payment, New order, Share statement (image), add a note or promise | Cache | Ledger, `creditLimit`, `exposureOf`, `recall` | P1 |
| O6 | Parties | Customers and suppliers, search-first, filters (overdue, over limit, no order in 30 days) | Row actions as O4 | Cache | Ledgers, saved views | P1 |
| O7 | **Inbox** | Approvals, promises due, bounced items, sync mismatches, system messages; day digest | Approve and Reject buttons on the card (step-up above a limit), reason sheet, Hold; deep links from push | Cached list; decisions need a connection ("needs a connection", never "saved") | Approvals design [C `approvals-workflows.md` section 6.4]; mismatch report | P2 |
| O8 | Books | Day Book feed by day with type chips; voucher detail sheet (read-only; share image; cancel with reason for the owner) | Search, filter chips, tap for detail | Cache | `daybook`, `voucherCancel` | P1 |
| O9 | Reports | The curated ten (P&L, Balance Sheet condensed, Trial Balance grouped, Cash and Bank book, Outstanding, Stock Summary, GST summary, Sales by customer, Sales by item, Expenses); "My reports" | Period chip; share as image or xlsx; star | Cache | Reports Center catalogue, `plFigures` | P1 |
| O10 | **Create sheet** | The quick-create registry filtered by rights (Invoice, Payment received, Payment made, Order, Lead, Party, Expense) | Tap to open the phone form for that type; unsupported types are absent, not greyed | Full for those with a form | `quickCreate.ts` | P2 |
| O11 | Receive payment | The collect flow (9.9) | Allocation, mode, QR, receipt share | Full | `singleEntry`, `upi`, `share` | P2 |
| O12 | Quick expense | Amount, ledger (recents), narration, photo of the bill | One screen, one save, undo | Full | Payment voucher; attachments design | P2 |
| O13 | Search | Pinned, Recent, Today; typed results by kind; Browse (the 12-group list) | Type, tap; Browse for the long tail | Cache | `rankPalette`, `recents`, `navTree` | P1 |
| O14 | Team and Devices | Members, roles, invites, devices, revoke | Add a phone (QR), invite by WhatsApp, revoke | Needs a connection | readiness B2, B3 | P2 |
| O15 | Settings | Language, text size to 200%, theme (follow phone), Sunlight, hand side, data saver, app lock, notification preferences, privacy of lock-screen text | Each opt-in, none changes anything until chosen | n/a | Settings window tabs [C] | P1 |
| O16 | Notification content | Approval waiting; day digest; limit exceeded or cash at the legal line; party 30/60/90 overdue; GST due in 3 and 1 days; large receipt; stale device | Android action buttons; iOS tap; five a day at most | n/a | Personas section 6.8 | P2 |

### 14.2 Field shell

| ID | Screen | What it shows | Key interactions | Offline | Builds on | Phase |
|---|---|---|---|---|---|---|
| S1 | **Join and pack** | Invite or QR; set PIN; what will be downloaded (size); permissions in context | Approve on the owner's phone; resume if interrupted | n/a, then Full | Flow B and C | P3 |
| S2 | **Today** | Day state; follow-ups due (from `nextFollowUp`); parties not ordered in 30 days; top overdue to collect; drafts and unsent items; totals for orders and collections | Start day and end day; tap a follow-up to open the party card; one-tap Call or WhatsApp | Full | `Lead.nextFollowUp`, derived lists | P3 |
| S3 | Parties | Search-first, recents; each row has outstanding, overdue, headroom chip, last order date | Filters (overdue, over limit, dormant); row actions: Call, WhatsApp, New order, Collect | Full (pack) | Pack, `creditLimit` | P3 |
| S4 | **Party card** (field) | Balance, overdue bills oldest first, credit headroom, last five orders with **Repeat**, phone buttons, visit log, notes | Collect, New order, Log visit; no purchase cost, no other reps' data | Full (stamped) | Ledger, `recall` | P3 |
| S5 | **New order** | The flow in 9.1 to 9.7: party, lines, review | Repeat last; stepper; barcode; swipe delete with undo; review sheet with words, credit projection, delivery date, voice note | Full | `recall`, `priceLevel`, `creditLimit`, `composeSalesOrder` (new) | P3 |
| S6 | **Collect** | Bills, allocation, mode, amount, QR | 9.9 | Full | `singleEntry`, `upi`, `billwise` | P3 |
| S7 | Log visit | Outcome chips (ordered, no order and why, closed, follow up), note or voice note, next follow-up date, optional photo and location stamp | Under five seconds for the common case | Full | New `PartyVisit` document; `LeadActivity` pattern | P3 |
| S8 | Orders (mine) | Status chips (Draft, Queued, Sent, Held, Rejected, Accepted, Invoiced); a held order says why | Withdraw before acceptance; repeat; share image | Full | Sales Order voucher, state vocabulary | P3 |
| S9 | Leads (CRM tenants) | Stage lists; follow-up dates | Swipe: Call, Log, Advance stage; add by business-card photo later | Full | `crm/leads.tsx`, `Lead`, `LeadActivity` | P3 |
| S10 | Stock check | Item search with scan; status by the owner's rule; rate by price level; last sold to this party | Scan or type | Cache with a stale flag | Stock positions, `priceLevel` | P3 |
| S11 | Sync and storage | Last sync, N unsent, pack size, errors, clear data | Retry; export unsent as a share; wipe | Full | State vocabulary | P3 |
| S12 | Day close | Expected cash against handed over; unsent items; today's orders and collections | Confirm hand-over; share the summary | Full | Personas section 8.6 (a rep cash ledger is a new object) | P3 |
| S13 | Van or counter invoice | Quick invoice with barcode, held bills, Bluetooth printing | Needs a per-device invoice series (readiness section 6) and, for a company above the e-invoice threshold (₹5 Cr [C `docs/research/einvoice-gsp.md:71`]), a connection, because the IRN comes from the portal [J] | Partial | Readiness section 6 | P4 |

### 14.3 Shared components

Search; the state chip and the sync sheet; the as-at stamp; the money field and echo; the party and item pickers (full-screen
routes); the review sheet; the undo toast; the share sheet and the image renderer; the QR display; the reason sheet; the empty,
offline and error cards; the "needs a wider display" card.

---

## 15. Design principles **[J]**

1. **Answer first, ask never.** The owner opens to the answer; the rep opens to the day. No tour, no permission prompt, no setup question before the first useful screen.
2. **The party is the centre.** Balance, overdue, headroom, last order, and the two next moves (collect, order) sit on one card.
3. **Name the state of everything the phone holds.** As at for figures; Draft, Queued, Sent, Held, Rejected, Provisional for items; nothing is "saved" unless it is.
4. **A figure on the phone equals the report it opens**, same window, same date, to the paisa.
5. **Make the safe thing fast and the dangerous thing specific.** Undo for the reversible, a sentence with the figure and verb buttons for the irreversible, a magnitude check for the improbable.
6. **A gesture is never the only way.** Every swipe has a button; every long-press has a menu; the keyboard layer stays on tablets with keyboards.
7. **Touch is 48 px; text is 16 px; the user's size is law.** Support 200%, never block zoom, never rely on the OS to scale web text.
8. **Figures are Plex Sans, tabular, in lakh and crore, in every language.** One formatter; Latin digits by default; a compact form that never lies; exact paise in lists.
9. **Short words, local words.** One or two words per label, verbs for results, Hindi first; the party's own script displays untouched.
10. **WhatsApp and UPI close the loop; nothing posts without a person.** One tap to a prefilled chat; a receipt from the field is a claim until the bank agrees.
11. **Offline is the normal case for the rep and a safety net for the owner.** Full for a shift, honest about staleness, never a silent skip of the credit check.
12. **A phone holds the least it needs.** A projection, not the books; cost and other reps' customers never arrive.
13. **Light first, light always.** A shell per role, a byte budget enforced in the build, windowed lists, no Mono on the first load.
14. **Same voice, new body.** Teal, Plex, ghost recall, as-at stamps and the quiet radii carry over; keycaps and rails do not.
15. **Sunlight and one hand are test conditions, not edge cases.** Test outdoors with the Sunlight switch and with one thumb.

---

## 16. Phasing, effort, risks, open questions, first spike

### 16.1 Phases (client work only, engineer-weeks, plus or minus 50%, **[J]**)

| Phase | Content | Needs from the backend (readiness items) | Client weeks |
|---|---|---|---|
| **P0 Foundations** | HTTPS; phone shell at 360 to 430 px; manifest, icons, service-worker app shell; entry chunks with a byte budget; phone sizes added to the screenshot harness (`scripts/shots`); `composeSalesOrder` and `composeReceipt` extracted with their suites; the money field and the formatter and the state vocabulary | B1 | 4 to 6 |
| **P1 Owner read-only** | O1 to O6, O8, O9, O13, O15: Home, Cash, Collect list, Party card, Books, ten reports, Search, Settings; share an image | B1; small books compute locally, big books need B7 pulse | 6 to 8 |
| **P2 Owner actions** | O7, O10 to O12, O14, O16: Inbox and approvals, Create, Receive payment, Quick expense, Team and Devices, push opt-in | B2, B3, B4, B9; approvals server (README S-pieces) | 5 to 7 |
| **P3 Field MVP** | S1 to S12: pack, Today, Party card, New order, Collect, Visit, Orders, Stock, Sync, Day close; draft autosave; barcode field and scanner; Hindi plus one regional language | B2 to B8; the field pack and read models | 12 to 16 |
| **P4 Later** | Van and counter invoicing, Bluetooth printing, voice order, Share to Comsky, TWA for Play, Capacitor if triggered, widgets, a server PDF and the hosted link | B11, portal, GSP | Per item |

Client total through P3 is about **27 to 37 engineer-weeks**, on top of the backend's 15 to 23 weeks of which 45 to 65 days overlap
platform work already planned (readiness section 12.2). These are judgement figures with no measured velocity behind them.

### 16.2 Risks

| Risk | Mitigation |
|---|---|
| iOS PWA limits (no install prompt, push only from the Home Screen, no Background Sync) | Owners on iOS use pull-to-refresh and the digest; the "install me" card explains the gain; a Capacitor path is on the triggers list |
| OEM skins kill background work on most Indian phones (personas section 3) | Sync on open and on foreground; a visible N-unsent badge; push is a hint |
| 3 to 4 GB phones and a 2.7 MB bundle | Entry chunks, windowed lists, a budget gate in the build |
| Indic text rendering on real devices | The three-phone test in 5.3 before any font is bundled |
| A QR-login relay attack | The confirm prompt, desktop approval, expiry, single use, owner notification (13.2) |
| A rep's phone holds more than it should | Section 2 row 13; the field pack and the server read gate before any rep ships (readiness B5 to B7) |
| Two shells drift | One component library, one formatter, one state vocabulary, one screenshot harness |
| Scope creep into a field-force product (beats, targets, route optimisation) | Personas section 4.2 says these are crowded and a later bet |
| Store-only trust demands | The triggers in 6.3; a TWA is weeks of work |
| The PDF gap | Image first, link with the portal, server PDF last (5.5) |

### 16.3 Decisions needed from the owner

1. **Is the PWA-first route accepted**, and may the old React Native roadmap line be reopened?
2. **The first segment:** distributors with field reps, or owners who only want the pulse? It decides P1 or P3 first.
3. **Do reps use company phones or their own?** It decides tracking, wipe and the app-lock stance.
4. **Languages:** Hindi plus which regional language first?
5. **Play Store listing wanted early?** (a Trusted Web Activity, 6.2 option B)
6. **Image or PDF for receipts and orders** while a server PDF does not exist (5.5)?
7. **Cash:** is a ₹2 lakh warn-and-block acceptable, and which CA signs off the wording?
8. **Sunlight and text size:** accept a one-tap Sunlight mode and a 200% text size as v1 requirements?
9. **Barcode:** is a `barcode` field on `StockItem` acceptable now (appended last, Alt-key grid, import mapping)?
10. **Location at check-in:** off by default, as the readiness doc proposes?

### 16.4 Recommended first spike (bounded, not started)

**Goal:** prove the phone shell on the real store with no backend change. **Scope:** behind a flag (`?shell=m`), the sample company and
a 2,000-voucher synthetic book: (1) a manifest, icons and a service-worker app shell, installed on one real Samsung A-class Android
through a TLS tunnel (not production); (2) three screens: owner Home (Cash & Bank, Today, Receivables, three overdue rows), Party card,
Collect (Receipt through the single-entry shape); (3) a separate entry chunk and its gzipped size; (4) a measurement run on that phone
on a throttled profile; (5) five sessions: three owners, two reps. **Acceptance [J]:** no horizontal scroll at 360 px and at 200% text;
cached Home in 2 seconds or less on the reference phone; three taps from Home to a party's overdue bills; a receipt against one bill
in 30 seconds or less; INP at or under 200 ms; the phone entry chunk at or under 350 KB gzipped; zero console errors; contrast checked in
both schemes and Sunlight; five people finish the three tasks unaided. **It does not decide** the data scope for reps, the PDF route,
or the wrapper. It adds a phone shell folder, a manifest and their tests, and edits only the entry points that must load them.

---

## Appendix A. Sources

All paraphrased; none copied. Tags as in the header.

**Apple Human Interface Guidelines [W]** (read through the JSON mirror `https://developer.apple.com/tutorials/data/design/human-interface-guidelines/<page>.json`; the HTML pages are `https://developer.apple.com/design/human-interface-guidelines/<page>`): tab-bars, layout, accessibility, typography, sheets, text-fields, lists-and-tables, playing-haptics, dark-mode, charting-data, entering-data, searching, onboarding, managing-accounts, home-screen-quick-actions, notifications, widgets, loading, feedback, alerts, gestures, undo-and-redo, privacy. App Store Review Guidelines (4.2, 4.7, 5.1.1(v)): https://developer.apple.com/app-store/review/guidelines/ [W].

**Android and Material [W]:** https://developer.android.com/design/ui/mobile/guides/foundations/accessibility · …/foundations/system-bars · …/layout-and-content/layout-basics · …/layout-and-content/adapt-layout · https://support.google.com/accessibility/android/answer/7101858 · https://developer.android.com/topic/architecture/data-layer/offline-first · https://developer.android.com/guide/topics/ui/look-and-feel/darktheme · Material Components for Android docs on GitHub (BottomNavigation, Snackbar, BottomSheet, FloatingActionButton): https://github.com/material-components/material-components-android/tree/master/docs/components · Trusted Web Activities: https://developer.android.com/develop/ui/views/layout/webapps/trusted-web-activities. m3.material.io pages could not be read.

**Web platform [W]:** WCAG 2.2 Understanding 2.5.8 https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html and 1.4.4 https://www.w3.org/WAI/WCAG22/Understanding/resize-text.html · https://web.dev/articles/install-criteria · https://web.dev/articles/offline-cookbook · https://web.dev/articles/inp · MDN: Secure Contexts, Storage quotas and eviction criteria, SpeechRecognition, Navigator.share, Vibration API, Contact Picker API, manifest `shortcuts`, `inputmode`, `Intl.NumberFormat` · MDN browser-compat-data (raw JSON): BarcodeDetector, SyncManager, OTPCredential, NetworkInformation, Notification, BeforeInstallPromptEvent, Navigator (`canShare`, `contacts`), SpeechRecognition · WebKit: https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/ , https://webkit.org/blog/14403/updates-to-storage-policy/ , https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/ , https://webkit.org/blog/3709/using-the-system-font-in-web-content/ · https://adrianroselli.com/2026/02/honoring-mobile-os-text-size.html · https://css-tricks.com/16px-or-larger-text-prevents-ios-form-zoom/ · https://capacitorjs.com/docs/guides/storage · https://design-system.service.gov.uk/components/text-input/

**Security and sign-in [W]:** RFC 8628 https://datatracker.ietf.org/doc/html/rfc8628 · OWASP QRLJacking https://community.owasp.org/attacks/Qrljacking

**UX research [W, Ws]:** NN/g touch targets https://www.nngroup.com/articles/touch-target-size/ and confirmation dialogs https://www.nngroup.com/articles/confirmation-dialog/ · Hoober, UXmatters https://www.uxmatters.com/mt/archives/2013/02/how-do-users-really-hold-mobile-devices.php · Pignoni, "Effects of ambient illumination on text recognition for UI development", CVCS 2018 (abstract only) [Ws] · web.dev Flipkart case study https://web.dev/case-studies/flipkart (2016; 63% on 2G, about 3x less data than the app) [W]

**Competitors [W unless noted]:** https://www.zoho.com/books/accounting-mobile-apps/ · App Store listings through `https://itunes.apple.com/search?term=<name>&country=in&entity=software` and `…/lookup?id=<id>&country=in` (Zoho Books 710446064, Zoho Invoice 413017364, Zoho Daybook 6744957391, Zoho CRM 444908810, Vyapar 6478382307, Khatabook 1488204139, OkCredit 1488748286, Pipedrive 921456160, Xero 441880705, Marg eOrder 1631309291, Marg eOwner 6476924466, Livekeeping 1451277319) · https://www.pipedrive.com/en/features/mobile-crm · Zoho CRM iPhone overview https://help.zoho.com/portal/en/kb/crm/crm-mobile/apple/getting-started/articles/zoho-crm-iphone-an-overview · Intuit help https://quickbooks.intuit.com/learn-support/en-us/help-articles/compare-mobile-app-features/00/239565 · Superhuman https://blog.superhuman.com/superhuman-for-iphone-now-faster-than-ever/ · [Ws] Vyapar, Khatabook, Xero and QuickBooks navigation, Superhuman help, Zoho offline threads.

**India [W unless noted]:** StatCounter https://gs.statcounter.com/os-market-share/mobile/india · Alex Russell https://infrequently.org/2025/11/performance-inequality-gap-2026/ · fonts: jsDelivr package listings for `@fontsource/noto-sans-*` and `@fontsource/ibm-plex-sans-devanagari` 5.3.0, https://fontsource.org/fonts/ibm-plex-sans-devanagari [W]; IBM Plex script coverage [Ws] · NPCI collect and limits [Ws]; https://www.cashfree.com/docs/payments/manage/payment-methods/upi-collect [W]; https://www.medianama.com/2025/08/223-npci-p2p-collect-payments-oct-1-what-it-means/ [W]; iOS UPI app schemes and status [Ws] · WhatsApp Business Platform windows and templates https://developers.facebook.com/docs/whatsapp/cloud-api/guides/send-messages [W] · DLT and WhatsApp OTP pricing [Ws] · cash receipts at ₹2 lakh [Ws] · Opensignal and device-market summaries [Ws] · Gboard Hindi voice input (India blog, 2017) [Ws].

**In this repo [C]:** `CLAUDE.md`; `docs/HANDOFF.md`; `docs/PERF.md`; `docs/design/ux-analysis-2026-10.md`; `docs/design/six-keystrokes-mockup.html`; `docs/research/{approvals-workflows,customer-vendor-portal,live-bank-feeds,attachments-drives,zoho-dashboard,pipedrive-list-ux,mobile-personas-and-jobs,mobile-backend-readiness,mobile-market}.md`; the source files cited inline.

## Appendix B. How the measurements were made

- **Bundle** [M]: `gzip -9` and `brotli -q 11` on `dist/assets/index-D16gwGz7.js` and `index-Bad99vsL.css`, built 2026-10-21 21:05 from the then-current working tree. A rebuild will differ.
- **Plex figures** [M]: a short Node script parsed the `.woff` files in `node_modules/@fontsource/ibm-plex-sans` and `ibm-plex-mono` (table directory, `cmap` formats 4 and 12, `hmtx`, GSUB feature tags) and read the advance width of each digit, comma, period and U+20B9; widths in the table are sums of advances over 1,000 units per em. No font was modified.
- **Contrast** [M]: the WCAG relative-luminance formula applied to the token hex values in `src/styles.css`, `contrast.css` and `accent.css`.
- **`Intl` outputs** [M]: Node 25.2.1, ICU 78.1, `Intl.NumberFormat` for `en-IN`, `hi-IN`, `ta-IN`, `kn-IN`, `mr-IN`, `bn-IN`, `gu-IN`, `te-IN`. Other engines and ICU versions may differ, which is itself the argument for one in-house formatter.
- **Sync sizes** are not measured or argued here. They are in the readiness doc's section 5.2 (50,000 vouchers = 102 pages, 45.6 MB raw, about 4.6 MB gzipped), which is why this document speaks of a field pack and a pulse and not of a replica of the books.
- **Nothing in this document was run on a phone.**
