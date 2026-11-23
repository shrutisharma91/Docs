# Comsky Books: light theme spec ("Calm canvas")

> **Status:** design spec, ready for four implementation agents. **Author:** design lead (read-only pass: nothing under `src/`, `scripts/` or `electron/` was edited; this is the one file created). **Date:** 2026-10-12.
> **Scope:** the **light colour scheme only**. The dark scheme, every printed document, every key, the menu tree, field order, compact row pitch and the 152px button rail are **frozen** and must come out bit-identical.

---

## 0. The decision

**Direction.** *Calm canvas*: a cool-neutral canvas (`#f2f4f6`) carrying white, 1px-bordered, 8px-radius cards; IBM Plex Sans throughout with tabular figures and quiet sentence-case labels; the brand teal reserved for action, focus and the keyboard cursor; and colour carried by small, opaque, pastel semantic chips instead of by panel tints.
**Why this one.** Every trait that makes the light UI read as a 1990s ERP (one monochrome green-grey, edge-to-edge hairline panes, mono ALL-CAPS tracked labels, typewriter figures, a mint selection band, dotted fields, double-underlined totals) is a *finish* decision, not a layout one, so it can be removed with zero change to keys, positions, row pitch or rail width, and because it ships as a light-scoped overlay the dark scheme and the printed paper cannot move.

| Changes (light only) | Does **not** change |
|---|---|
| Canvas + white cards (a visible gap of canvas around every screen, 1px border, soft shadow) | Any shortcut, menu item, menu order, field order, F-key rail contents |
| Neutral cool palette, one accent, pastel chips, softer selection | Dark scheme (`:root[data-scheme='dark']` block, every dark-computed style) |
| Plex Sans for everything but keycaps; sentence-case 11-12px labels; sans tabular figures | Printed invoices, vouchers, reports, cheques (`print.css`, `.pv-doc`, `.pr-doc`) |
| 4/6/8/12px radii, light elevation, boxed form fields, quiet grid cells | Compact row pitch (<= 21px), `--row-y/--row-x/--head-y/--field-y/--pad`, type-scale tokens, `--topbar-h`, `--statusbar-h` |
| Gateway as two cards, sidebar/rail/bars sitting on the canvas | Grid `186px 1fr 152px`, the rail's key + label metrics |
| Accent re-used sparingly; brick-red hotkeys become underlined ink/teal | Brand name (`src/brand.ts`), IBM Plex as the only family, no new fonts |

---

## 1. Evidence: what I looked at and measured

1. **Looked** at Gateway, Day Book, Trial Balance, Balance Sheet, GSTR-1, a ledger register, Sales voucher (F8, with the ledger popup, place-of-supply list, ghost recall and the Quit prompt), a master create form, the Ctrl+K palette, the F1 key sheet, the Dashboard, Getting Started, Bulk Entry, Reminder Letters and the Period prompt at 1440x900, light, compact; plus Dashboard and Trial Balance/GSTR-1 in comfortable. (The pane's `zoom` action does not magnify, so detail was read by scaling the page with a temporary CSS transform and screenshotting regions.)
2. **Read** `src/styles.css` (all 1757 lines), `styles/{report,voucher,sidebar,onboarding,feedback,favourites,keyhelp,bulk,print}.css`, `docs/design/six-keystrokes-mockup.html`, `src/brand.ts`, `scripts/assert/visual.assert.ts` + `visual.baseline.json`, `reportTools.ts` (how reports are cloned for print), `AppShell.tsx`, `MenuScreen.tsx`.
3. **Prototyped this spec** as a throw-away stylesheet served from a temporary local directory and injected into the live page (never committed). Everything in Appendix A is that stylesheet, so it has been *seen*, not just reasoned about. Measured with it on vs off, at 1440x900:
   * compact report row pitch **20.875px both ways** (limit 21);
   * `.btnpanel` **152px**, and on the 21-button Sales-voucher rail **0 labels truncated or newly wrapped**;
   * **dark scheme: 0 of 160** computed-style fingerprints changed (light: 160 of 160 changed, so the probe is sensitive);
   * IBM Plex Sans digits are already tabular (`1111` and `0000` both measure 0.6em, and `font-variant-numeric` makes no difference), so dropping Plex Mono for figures loses no alignment and makes rupee columns ~13% narrower.
4. **Ran the real `scripts/assert/visual.assert.ts`** against a scratch copy of the repo carrying the proposed tokens plus the assert extension in section 5.5: **706 assertions pass** (684 existing + 22 new light-only pairs), dark unchanged.

---

## 2. Diagnosis: why the light scheme reads "Tally-era ERP"

I checked the premise before accepting it: the canvas is **not** sepia or cream. The tokens are a green-tinted grey (`--bg #eef1ee`, hue 120deg; `--panel #fbfcfb`; `--panel2 #f3f6f4`, hue 140deg) and the whole product reads as one colour. The old feel comes from these specific, citable traits:

1. **The canvas is never visible, so nothing is "raised".** The token ramp exists (`--bg` < `--panel2` < `--panel` < `--raised`) but the layout never lets the canvas show: `.screen`, `.gw-left`, `.tmenu-outer`, `.navrail`, `.btnpanel` are flush, edge-to-edge panes separated only by `1px solid var(--border-lt)` (`.navrail` border-right, `.btnpanel` border-left, `.gw-left` border-right, `.statusbar`/`.topbar` rules). A content surface has radius 0 and margin 0. Only `.dash` (the Dashboard) is built from cards, which is why it is the one screen that already looks modern. The chrome (`--panel2`) sits only 1.05:1 from the canvas, and the sheet 1.11:1, with no gap between them.
2. **One hue everywhere.** Canvas, chrome, selection (`--sel #cfeae4`, sat 39%), headers (`--tint #e4efeb`), hover (`--hover #e6f1ed`), focused-field fill (`--field-focus #e6f1ef`), borders (`--border #c2ccc5`, `--border-lt #d7ded9`) and the ink (`--ink #14201b`, green-black) are all the same green. The teal accent (`#0b6e62`) has nothing to contrast against, so it cannot *mean* anything.
3. **Mono ALL-CAPS tracked micro-labels as the whole typographic voice.** 35 `text-transform: uppercase` declarations (31 outside `print.css`) and 31 tracked `letter-spacing: 0.06-0.14em` ones, nearly all in Plex Mono 10px: `table.tt th`, `table.vt-entries th`, `table.inv-items th`, `.subpanel th`, `.fsection`, `.inv-section-title`, `.tmenu-section`, `.btnpanel::before`, `.sb-sec`, `.sb-today-head`, `.popup .pt`, `.subpanel .pt`, `.kpi .lbl`, `.cmdp-grp-lbl`, `.ksheet-grp-lbl`, `.ob-eyebrow`, `.ob-keys-h`, `.subs-secttl`, `.bulk-card .k`, `.bulk-stat .l`, `.fav-lk-grp h3`, `.fav-pick-t`, `.chip`, `.subs-badge`, `.plan-tag`, `.fav-lock-tag`. It is the register of a DOS report generator.
4. **Typewriter figures.** `--num` is IBM Plex Mono and 68 rules use it: `td.r`, `.kpi .val`, `.plan-price`, `.vch-date input`, `.frow input.num`, `.amt` inputs, `.sb-today-n`, popup balances. Monospaced rupee columns with `letter-spacing: -0.01em` look like a green-screen printout and are 13% wider than the tabular sans figures that do the same alignment job.
5. **Dotted-underline fields on a bare sheet.** The global `input` rule paints `--dotted` (a 1px-on/1px-off gradient) and nothing else, and `.frow` lays labels out as `label : value` with a `.sep` colon column. It is a printed form being typed on; there is no field affordance, and focus is a mint fill plus a 2px bar.
6. **Full-row mint selection band plus a hairline under every row.** `table.tt tr.sel td { background: var(--sel) }` (mint, 1.23:1 on the sheet), `.tmenu-item.sel`, `.popup .pitem.sel`, `.cmdp-item.sel`, `.ksheet-row.sel` all add an `inset 2px 0 0 var(--accent)` bar (9 occurrences) on top; `table.tt td { border-bottom }` rules every row. Reads as a spreadsheet cursor, not a list.
7. **Accounting-print furniture on screen.** `table.tt tr.total td { border-top: 1px solid var(--total-line); border-bottom: 3px double var(--total-line) }`, `.scr-body table.tt tr[data-rs='total'] td { border-top: 2px }`, `.vch-totals td`, with `--total-line #1a2621` (near-black). Right for paper (and kept there); on screen it is a ledger book.
8. **Duplicated, centred report titles.** `.rpt-title`/`.rpt-sub` are `text-align: center` and repeat the title already in `.scr-head .t` (the Day Book screenshot shows "Day Book" twice). Comfortable density already hides `.rpt-title`; compact never did. The centred, tracked-out `.tmenu` Gateway has the same printed-page feel.
9. **Bevelled keycaps and a labelled "Button Bar".** `.pbtn .pk, kbd, .kcap ...` use `--sunk` grey, `border-bottom-width: 2px` and mono caps; `.btnpanel::before` prints "Button Bar" in 10px tracked mono caps.
10. **Brick-red mnemonics.** `--hot #a83a30` underlined (`.hk`, `.topbar .titem u`): the Windows-menu accelerator look, and a second hue that appears nowhere else in the product.
11. **Square corners and stripe callouts.** `--radius 4px`, `--radius-lg 5px`, 0 on content surfaces; 12 left-stripe accents. The *callout* ones (`.kpi::before`, `.subs-status`, `.ob-note`, `.bulk-done`, `.acceptbar.fb-accept`) are the 2015 admin-template tell; the *state* stripes (`.sev-warn/.sev-crit`, `.pk-tbl [data-pk]`, `table.bulkgrid tr.bad/.iffy`) carry meaning and stay.
12. **Shadows borrowed from dark.** `--shadow-3: 0 22px 60px rgba(0,0,0,0.40)` and `--overlay-bg rgba(10,16,14,0.45)` make every light modal look like a dark-theme artefact.
13. **Cream strips top and bottom.** `.ob-banner` and `.storagewarn` are 10% amber over `--panel`, `.trialbar` is the mint `--tint`: full-bleed beige/mint bars on every screen.
14. **Chrome fused to content.** Top bar, rail and status bar are all `--panel2` with 1px rules against the sheet, so the window reads as a Win32 form with a menu bar rather than an app on a canvas.

**Trait to fix, at a glance**

| # | Trait | Fix | Where |
|---|---|---|---|
| 1 | canvas never visible | canvas gap + 1px border + radius + shadow on every screen and on the Gateway | 6, 7.3, 7.6 |
| 2 | one green hue | cool neutral ramp, one accent, pastel chips | 5.2, 5.3 |
| 3 | mono caps labels | L1/L2 sentence-case sans recipes | 6, 7.x |
| 4 | typewriter figures | `--num` to Plex Sans (tabular); mono kept for keys and code | 5.1, 9.1 |
| 5 | dotted fields | boxed form fields; quiet grid cells | 7.8, 7.9 |
| 6 | mint band + bar everywhere | soft wash; accent edge only on table cursors, caret on menus | 6, 7.7, 7.10, 7.11 |
| 7 | double-rule totals | single rule + soft fill on screen (paper keeps its own) | 7.7 |
| 8 | duplicate centred titles | hide `.rpt-title`, left-align `.rpt-sub` | 7.6 |
| 9 | bevelled keycaps | flat white keycaps, same mono metrics | 7.4 |
| 10 | brick-red mnemonics | ink letter, teal underline | 7.1, 7.3 |
| 11 | square corners, stripes | 4/6/8/12px radii; callout stripes become tinted cards | 5.1, 7.10-7.16 |
| 12 | dark shadows | `--shadow-1/2/3` and scrim re-valued for light | 5.2 |
| 13 | cream strips | inset rounded chip-toned strips | 7.5 |
| 14 | fused chrome | chrome sits on the canvas | 7.1, 7.2, 7.4, 7.5 |

---

## 3. Principles

1. **Cards on canvas.** One neutral canvas, one white surface, three elevations. A card needs a *visible gap* of canvas, a 1px border and a soft shadow; palette alone cannot raise it (old sheet vs canvas was already 1.11:1).
2. **Neutral first; teal means intent.** Accent only for: primary buttons, focus, the keyboard cursor, links/mnemonics, the brand mark, active-state glyphs. Never for headers, row bands, idle borders or hover. Data colour comes from chips.
3. **Quiet labels, loud data.** Labels are sentence-case 11-12px `--dim`; figures are 13.5px `--ink` Plex Sans, tabular. Mono survives only on keycaps, the typed palette alias and code.
4. **Same grid, softer lines.** No geometry changes. Lines get lighter, selection gets gentler, but the keyboard cursor stays findable (tint **plus** the 2px accent edge in tables; tint **plus** the `›` caret in menus).
5. **Boxed where you type once, quiet where you type fast.** Form fields get a real box (inset ring, zero layout cost); cells in dense grids are quiet until hovered or focused.
6. **AA is a floor, proven in numbers.** Every colour token pair is in `visual.assert.ts`; no text sits on a translucent surface; the one `color-mix` tint (voucher badges) is verified (>= 4.56:1).
7. **Motion is feedback.** <= 120ms (`--dur 90ms`, `--dur-slow 120ms` unchanged), never between a keystroke and its character; no hover transforms.
8. **Light-only by construction.** An overlay layer, scoped, wrapped in `@media screen`, rooted so the print clone can never match. Dark and print are guaranteed by structure, then checked by probe.

---

## 4. How it ships: the overlay layer

### 4.1 Why an overlay and not an edit-in-place
Dark must not change and four agents must not touch the same file. A **light-scoped overlay** gives both: new files, loaded after `styles.css`, every rule scoped to the light scheme, and a dark scheme that never sees them. The old light rules stay in place as the fallback and as the proof of "what changed". (When dark is restyled later, fold the overlay into the base rules and delete the prefix; that is the promotion path.)

### 4.2 The prefix
Every overlay selector starts with the literal

```
html:root[data-scheme='light']
```
Throughout this document and in Appendix A it is abbreviated **`@L`**. Implementers write it expanded; `sed "s/@L/html:root[data-scheme='light']/g"` turns the appendix into a starting file. `@L[data-density='comfortable']` expands to `html:root[data-scheme='light'][data-density='comfortable']`.
`index.html` sets `data-scheme` before first paint, so the attribute is always present.

### 4.3 The hard rules (reviewers fail the work for any of these)
1. **Wrap each file's rules in one `@media screen { ... }`.** Print media then ignores the overlay entirely.
2. **Anything a report clone can contain must be rooted under `.screen` or `.scr-body`** (`table.tt`, `.chip`, `.hdr2`, `.total`, `.rpt*`, `.twoside`, `.hscroll`, `.amt-*`, `.overdue`, `.dimtext`, `.sev-*`). `reportTools.ts` clones the *children* of `.scr-body` into `.print-root` (a body-level host that is also laid out under *screen* media while `fitToPage` measures it), so a bare `table.tt th` overlay rule would out-specify `print.css` and change printed reports and the orientation measurement.
3. **Re-point `--num` / `--mono` in exactly one place** (the scoped token block, 5.1), never inside a rule, and never touch `--font`. `print.css` pins `--num` back to mono with the one-line edit in 9.3, so a printed report keeps its Plex Mono chips and `print.css`'s `--pp-mono: var(--mono)` keeps resolving to Plex Mono.
4. **Mirror the base selector** you are overriding, with the `@L` prefix in front. The prefix adds (0,2,1), so an overlay rule always out-ranks the base rule it mirrors, in any source order. If the base rule starts with `:root[data-density='comfortable']`, the overlay must start `@L[data-density='comfortable']` instead (the density rules in styles.css section 25 and keyhelp.css's last two rules are the only ones).
5. **No equal-specificity ties between two overlay rules.** Generic field rules use `:where(:not([type=...]))` (specificity (0,2,2)) so component rules (palette, Go To, key sheet, grid cells) always win regardless of file order.
6. **Do not touch** the metrics the 152px rail was solved for: `.btnpanel` padding/gap, `.pbtn` padding/gap/font-size/line-height, `.pbtn .pk` font-size/padding/min-width/border-left/right, `.main` columns. The assert reads them from the *base* rules and cannot see an overlay that breaks them (measured with Plex advance widths: the worst static pair is 134.4 of 137px).
7. **Never remove or rename** a custom property or class (`visual.baseline.json`: 87 properties, 259 classes). Only re-value or add.
8. **Do not change `content:`** of any pseudo-element except `.btnpanel::before` ('Button Bar' to 'Button bar'), and do not hide an element that carries text another screen reads (`.rpt-sub` stays in the DOM; only `.rpt-title` is `display: none`, as comfortable density already does).

### 4.4 Files
```
src/styles/light.css            4 @imports, nothing else
src/styles/light/shell.css      Agent 1
src/styles/light/reports.css    Agent 2
src/styles/light/entry.css      Agent 3
src/styles/light/overlays.css   Agent 4
src/styles.css                  Agent 1 only, token blocks only (section 5)
src/main.tsx                    one added import line (Step 0)
```
`main.tsx`: `import './styles/light.css';` directly after `import './styles.css';`. Specificity, not order, decides every conflict, so bundler ordering is irrelevant.

---

## 5. Tokens

### 5.1 How the token edit is made (`src/styles.css`, Agent 1 only, nothing else in the file)

1. **Re-value the palette tokens** (44 listed in 5.2; 40 actually change value) in the existing light block (`:root { color-scheme: light; ... }`, section 4 "LIGHT PALETTE") exactly as in 5.2. All of them are overridden by the dark block, so dark is unaffected (script-checked by `token-leak.mjs`, 10.1: *tokens changed in `:root` that dark does not override = 0*).
2. **Add one new block** directly after the dark palette block (before the "Accent tones" comment):

```css
/* 5b. LIGHT-ONLY tokens. Anything dark does not override MUST live here, never in :root:
   writing a hex into :root would be inherited by dark (the visual assert caught it: dark --kbd-ink
   fell to 1.38:1 in my trial). New light-only tokens are defined here too. */
:root[data-scheme='light'] {
  --kbd-bg: #ffffff;  --kbd-edge: #cfd5dc;  --kbd-ink: #2b3440;
  --radius-sm: 4px;  --radius: 6px;  --radius-lg: 8px;  --radius-xl: 12px;
  /* figures are Plex Sans (tabular by design); the literal mono stack stays reachable as --mono */
  --num: var(--font);
  --mono: 'IBM Plex Mono', ui-monospace, 'SF Mono', Menlo, Consolas, 'Roboto Mono', monospace;
  --frame-gap: 8px;                 /* the visible canvas gap around a screen card */
  --hover-canvas: #e8ecf0;          /* row hover on the CANVAS (sidebar, rail); --hover is for white */
  --chip-green-fg: #14633a;  --chip-green-bg: #e2f4e9;
  --chip-red-fg: #a52d22;    --chip-red-bg: #fde7e4;
  --chip-amber-fg: #7a4a00;  --chip-amber-bg: #fcefd0;
  --chip-blue-fg: #1f4fa8;   --chip-blue-bg: #e3ecfd;
  --chip-violet-fg: #5136b8; --chip-violet-bg: #ebe6fc;
  --chip-teal-fg: #0b6358;   --chip-teal-bg: #dcf2ee;
  --chip-slate-fg: #475262;  --chip-slate-bg: #eceff3;
}
/* comfortable sets its own radii at the same specificity as the block above; this one wins on weight, not order */
:root[data-scheme='light'][data-density='comfortable'] {
  --frame-gap: 12px;
  --radius-sm: 4px;  --radius: 6px;  --radius-lg: 8px;  --radius-xl: 12px;
}
```
3. **Do not edit** the density blocks, the dark block, the second `:root` block (`--tone-green/red/blue/violet/amber`; unchanged and verified in 5.6), the type scale, `--sp-*`, `--dur*`, `--ease`.
4. **Aliases dark inherits stay untouched in `:root`**: `--line --line2 --rule --fg --ink-3 --red --err-bg --dotted --shadow --kpi-tone --vt-tone`. They re-resolve from the re-valued tokens on their own.
5. **The one rule to remember:** a token dark does not override may only be re-valued inside `:root[data-scheme='light']`.

### 5.2 Every existing light token, old to new

`Scope` = where the new value is written. Unchanged rows are listed so nothing is ambiguous.

| Token | Old | New | Scope | Why |
|---|---|---|---|---|
| `--bg` | `#eef1ee` | `#f2f4f6` | `:root` | canvas: cool neutral. White vs canvas is 1.10:1 (the old sheet vs canvas was 1.11:1); what changes is that the canvas is now VISIBLE around every card |
| `--panel` | `#fbfcfb` | `#ffffff` | `:root` | the card/sheet: pure white (was #fbfcfb). Not the real problem: the layout never showed the canvas around it |
| `--panel2` | `#f3f6f4` | `#f7f8fa` | `:root` | soft fill INSIDE a card: ghost block, bulk cards, popup heads. No longer the chrome colour (chrome sits on --bg) |
| `--raised` | `#ffffff` | `#ffffff` | `:root` |  |
| `--sunk` | `#e6ebe7` | `#eceff3` | `:root` | recessed: segmented tracks, scope chips, progress track. Keycaps no longer use it (see --kbd-bg) |
| `--field` | `#ffffff` | `#ffffff` | `:root` |  |
| `--field-focus` | `#e6f1ef` | `#f3faf9` | `:root` | barely-there teal wash for bulk drop zone / focused grid cell |
| `--btn-bg` | `#f7f9f8` | `#ffffff` | `:root` |  |
| `--border` | `#c2ccc5` | `#d5dae1` | `:root` | control edge (buttons, popups, keycap lower edge). Was #c2ccc5, the heaviest line in the old UI |
| `--border-lt` | `#d7ded9` | `#e6e9ed` | `:root` | hairline: card edge, row separators |
| `--line` | `var(--border-lt)` | unchanged | `:root` (alias, unchanged) |  |
| `--line2` | `var(--border-lt)` | unchanged | `:root` (alias, unchanged) |  |
| `--rule` | `var(--border)` | unchanged | `:root` (alias, unchanged) |  |
| `--total-line` | `#1a2621` | `#2a3340` | `:root` | only the voucher Total row still uses it; report totals use --border |
| `--ink` | `#14201b` | `#19202a` | `:root` | cool near-black (was green-black) |
| `--fg` | `var(--ink)` | unchanged | `:root` (alias, unchanged) |  |
| `--ink-2` | `#46544d` | `#475262` | `:root` | secondary text |
| `--dim` | `#586761` | `#5a6676` | `:root` | muted text. Hierarchy unchanged: ink > ink-2 > dim |
| `--ink-3` | `var(--dim)` | unchanged | `:root` (alias, unchanged) |  |
| `--ghost` | `#5f6e67` | `#626d7d` | `:root` | ghost recall / placeholders |
| `--accent` | `#0b6e62` | `#0b7266` | `:root` | brand teal kept; #0b6e62 to #0b7266, a hair lighter, still 5.11:1 on its own soft tint (lighter fails 4.5:1 there) |
| `--accent-soft` | `rgba(11, 110, 98, 0.10)` | `rgba(11, 114, 102, 0.09)` | `:root` | selected / hot fill |
| `--on-accent` | `#ffffff` | `#ffffff` | `:root` |  |
| `--maroon` | `#0b6e62` | `#0b7266` | `:root` | historical name, = accent |
| `--btn-key` | `#0b6e62` | `#0b7266` | `:root` | = accent |
| `--hot` | `#a83a30` | `#0b7266` | `:root` | was brick red #a83a30. Hotkey letter is now accent + underline: one accent hue in the chrome, not two |
| `--sel` | `#cfeae4` | `#e3f1ee` | `:root` | cursor row: soft teal wash (was #cfeae4 mint band). The 2px accent edge on the first cell is what finds it at a glance |
| `--sel-strong` | `#a9d9d0` | `#d4ece7` | `:root` | pressed / selected option |
| `--hover` | `#e6f1ed` | `#f1f4f7` | `:root` | hover on a WHITE surface |
| `--tint` | `#e4efeb` | `#edf6f4` | `:root` | sub-header rows (hdr2), trial bar, ob-note |
| `--bar` | `#f3f6f4` | `#f2f4f6` | `:root` | status bar = canvas |
| `--dr` | `#3a4e7e` | `#2c4a9e` | `:root` | debit figures (blue-indigo) |
| `--cr` | `#146b3e` | `#137544` | `:root` | credit figures |
| `--green` | `#1b7a4a` | `#1a7f4b` | `:root` | solid positive fills |
| `--neg` | `#ab4036` | `#b0372c` | `:root` | negative figures |
| `--red` | `var(--neg)` | unchanged | `:root` (alias, unchanged) |  |
| `--warn` | `#8a5200` | `#8a5300` | `:root` | warning text |
| `--warn-bg` | `rgba(138, 82, 0, 0.11)` | `rgba(138, 83, 0, 0.10)` | `:root` | translucent warning wash |
| `--crit` | `#ab4036` | `#b0372c` | `:root` | error text |
| `--crit-bg` | `rgba(171, 64, 54, 0.11)` | `rgba(176, 55, 44, 0.09)` | `:root` | translucent error wash |
| `--err-bg` | `var(--crit-bg)` | unchanged | `:root` (alias, unchanged) |  |
| `--ok-bg` | `rgba(20, 107, 62, 0.11)` | `rgba(19, 117, 68, 0.10)` | `:root` | translucent ok wash |
| `--danger-bg` | `#ab4036` | `#b0372c` | `:root` | solid error band |
| `--danger-fg` | `#ffffff` | `#ffffff` | `:root` |  |
| `--focus` | `#0b6e62` | `#0b7266` | `:root` |  |
| `--focus-ring` | `0 0 0 2px rgba(11, 110, 98, 0.30)` | `0 0 0 3px rgba(11, 114, 102, 0.22)` | `:root` | 3px soft halo (was 2px at .30); the 1px accent edge inside carries the 3:1 |
| `--kbd-bg` | `var(--sunk)` | `#ffffff` | `:root[data-scheme='light']` | white keycap face |
| `--kbd-edge` | `var(--border)` | `#cfd5dc` | `:root[data-scheme='light']` | keycap edge |
| `--kbd-ink` | `var(--ink)` | `#2b3440` | `:root[data-scheme='light']` | keycap text |
| `--field-line` | `#7f8e88` | `#848fa0` | `:root` | resting field edge. 3.27:1 on white, 3.08:1 on --panel2 (WCAG 1.4.11). Used as an inset 1px ring, not a dotted underline |
| `--dotted` | `repeating-linear-gradient(90deg, var(--field-line) 0 1px, transparent 1px 2px)` | unchanged | `:root` (alias, unchanged) |  |
| `--shadow-1` | `0 1px 2px rgba(20, 32, 27, 0.05)` | `0 1px 2px rgba(16, 24, 40, 0.05)` | `:root` | card |
| `--shadow-2` | `0 1px 2px rgba(20, 32, 27, 0.05), 0 10px 30px -14px rgba(20, 32, 27, 0.22)` | `0 1px 2px rgba(16, 24, 40, 0.06), 0 6px 16px -4px rgba(16, 24, 40, 0.12)` | `:root` | popup / accept bar / menus |
| `--shadow-3` | `0 22px 60px rgba(0, 0, 0, 0.40), 0 2px 8px rgba(20, 32, 27, 0.10)` | `0 2px 6px rgba(16, 24, 40, 0.08), 0 20px 48px -12px rgba(16, 24, 40, 0.28)` | `:root` | palette, modals, toast. Was 0.40-alpha black |
| `--shadow` | `var(--shadow-2)` | unchanged | `:root` (alias, unchanged) |  |
| `--overlay-bg` | `rgba(10, 16, 14, 0.45)` | `rgba(16, 24, 40, 0.34)` | `:root` | scrim, lighter |
| `--kpi-tone` | `var(--accent)` | unchanged | `:root` (alias, unchanged) |  |
| `--vt-tone` | `var(--accent)` | unchanged | `:root` (alias, unchanged) |  |
| `--radius-sm` | `3px` | `4px` | `:root[data-scheme='light']` | keycaps, chips |
| `--radius` | `4px` | `6px` | `:root[data-scheme='light']` | fields, buttons, rows |
| `--radius-lg` | `5px` | `8px` | `:root[data-scheme='light']` | cards |
| `--radius-xl` | `6px` | `12px` | `:root[data-scheme='light']` | palette, key sheet, modals |
| `--num` | `'IBM Plex Mono', ui-monospace, 'SF Mono', Menlo, Consolas, 'Roboto Mono', monospace` | `var(--font)` | `:root[data-scheme='light']` | FIGURE FACE. Was Plex Mono (the typewriter ledger look); now Plex Sans, whose digits are tabular already (verified: 1111 and 0000 both 0.6em). Keycaps/code re-assert mono via --mono |
| `--mono` | `var(--num)` | `'IBM Plex Mono', ui-monospace, 'SF Mono', Menlo, Consolas, 'Roboto Mono', monospace` | `:root[data-scheme='light']` | was an alias of --num. Now the literal mono stack, so print.css (--pp-mono: var(--mono)) and keycaps keep Plex Mono |

Unchanged and not repeated: the type scale (`--font --sans --fs-* --lh*`), density metrics (`--row-y --row-x --head-y --field-y --pad --topbar-h --statusbar-h`), spacing (`--sp-1..8`), motion (`--dur --dur-slow --ease`), the five `--tone-*` colours.

### 5.3 New tokens (all in the scoped block)

| Token | Value | Use |
|---|---|---|
| `--frame-gap` | `8px` (comfortable `12px`) | canvas gap around the screen card, the Gateway cards, the banners |
| `--hover-canvas` | `#e8ecf0` | hover on the sidebar, rail, top-bar items (surfaces that sit on the canvas) |
| `--chip-green-fg/-bg` | `#14633a` / `#e2f4e9` | paid, active, delivered, credit |
| `--chip-red-fg/-bg` | `#a52d22` / `#fde7e4` | overdue, failed, revoked, locked |
| `--chip-amber-fg/-bg` | `#7a4a00` / `#fcefd0` | pending, partial, write scope, sample, storage warning |
| `--chip-blue-fg/-bg` | `#1f4fa8` / `#e3ecfd` | informational, debit, scopes |
| `--chip-violet-fg/-bg` | `#5136b8` / `#ebe6fc` | post-dated, journal, payroll |
| `--chip-teal-fg/-bg` | `#0b6358` / `#dcf2ee` | current, selected plan, trial bar |
| `--chip-slate-fg/-bg` | `#475262` / `#eceff3` | neutral: closed, inactive, "due" |

Chips are **opaque** pairs on purpose: a translucent tint picks up the colour of the row it sits on (a marked or cursor row dropped the old `Optional` pill to 3.77:1; the fix in `report.css` is exactly this idea).

### 5.4 Contrast: every pair, before and after

Computed with the assert's own functions (`over`, `lin`, `lum`) against the proposed tokens. All pass; most improve or hold. Tight ones to leave alone: `--ghost` on `accent-soft/panel` 4.62, `--accent` on `accent-soft/bg` 4.66, `--accent` on `sel-strong` 4.69, `--cr` on `ok-bg/panel` 4.99.

| foreground on surface | min | before | after | where it lands |
|---|---|---|---|---|
| `--ink` on `panel` | 4.5 | 16.31 | **16.39** | body on the sheet |
| `--ink` on `panel2` | 4.5 | 15.41 | **15.42** | body on chrome |
| `--ink` on `raised` | 4.5 | 16.77 | **16.39** | body in popups |
| `--ink` on `bg` | 4.5 | 14.74 | **14.86** | body on the canvas |
| `--ink` on `sunk` | 4.5 | 13.90 | **14.21** | keycap / chip text |
| `--ink` on `sel` | 4.5 | 13.21 | **14.11** | selected row |
| `--ink` on `sel-strong` | 4.5 | 10.80 | **13.23** | selected option |
| `--ink` on `hover` | 4.5 | 14.51 | **14.85** | hovered row |
| `--ink` on `tint` | 4.5 | 14.25 | **14.90** | tinted header |
| `--ink-2` on `panel` | 4.5 | 7.75 | **7.92** | secondary text |
| `--ink-2` on `panel2` | 4.5 | 7.32 | **7.45** | rail labels |
| `--ink-2` on `bar` | 4.5 | 7.32 | **7.18** | status crumb |
| `--dim` on `panel` | 4.5 | 5.79 | **5.84** | muted labels on the sheet |
| `--dim` on `panel2` | 4.5 | 5.47 | **5.49** | muted labels on chrome (eyebrows, button bar) |
| `--dim` on `raised` | 4.5 | 5.95 | **5.84** | muted text in popups |
| `--dim` on `bg` | 4.5 | 5.23 | **5.30** | muted text on the canvas |
| `--dim` on `sunk` | 4.5 | 4.93 | **5.06** | muted text on a chip |
| `--dim` on `bar` | 4.5 | 5.47 | **5.30** | status bar |
| `--dim` on `hover` | 4.5 | 5.15 | **5.29** | muted text under the pointer |
| `--dim` on `tint` | 4.5 | 5.06 | **5.31** | muted text on a tinted header |
| `--dim` on `accent-soft/panel` | 4.5 | 5.01 | **5.14** | muted text on a focused field |
| `--ghost` on `panel` | 4.5 | 5.22 | **5.25** | ghost rows / placeholder on the sheet |
| `--ghost` on `panel2` | 4.5 | 4.93 | **4.94** | ghost-recall block |
| `--ghost` on `raised` | 4.5 | 5.37 | **5.25** | placeholder in a popup |
| `--ghost` on `field` | 4.5 | 5.37 | **5.25** | placeholder in a field |
| `--ghost` on `accent-soft/panel` | 4.5 | 4.52 | **4.62** | placeholder in a focused field |
| `--accent` on `panel` | 4.5 | 5.97 | **5.81** | accent text |
| `--accent` on `panel2` | 4.5 | 5.64 | **5.47** | brand in the top bar |
| `--accent` on `raised` | 4.5 | 6.14 | **5.81** | accent in popups |
| `--accent` on `sel` | 4.5 | 4.84 | **5.00** | menu cursor row |
| `--accent` on `hover` | 4.5 | 5.31 | **5.27** | hovered accent text |
| `--accent` on `tint` | 4.5 | 5.22 | **5.28** | tinted header accent |
| `--accent` on `accent-soft/panel` | 4.5 | 5.17 | **5.11** | voucher chip / focused field |
| `--accent` on `accent-soft/panel2` | 4.5 | 4.90 | **4.83** | hot key in the rail / selected sidebar row |
| `--accent` on `accent-soft/raised` | 4.5 | 5.31 | **5.11** | selected palette row |
| `--on-accent` on `accent` | 4.5 | 6.14 | **5.81** | text on a solid accent fill |
| `--hot` on `panel` | 4.5 | 6.16 | **5.81** | menu hotkey letter |
| `--hot` on `panel2` | 4.5 | 5.83 | **5.47** | top-bar hotkey letter |
| `--hot` on `sel` | 4.5 | 4.99 | **5.00** | hotkey letter on the cursor row |
| `--hot` on `hover` | 4.5 | 5.48 | **5.27** | hotkey letter under the pointer |
| `--kbd-ink` on `kbd-bg` | 4.5 | 13.90 | **12.59** | keycap |
| `--kbd-ink` on `panel` | 4.5 | 16.31 | **12.59** | keycap in the rail |
| `--bg` on `ink` | 4.5 | 14.74 | **14.86** | undo toast: --bg text on the --ink chip |
| `--warn` on `panel` | 4.5 | 6.21 | **6.33** | warning text |
| `--warn` on `warn-bg/panel` | 4.5 | 5.30 | **5.47** | warning chip |
| `--crit` on `panel` | 4.5 | 5.80 | **6.12** | error text |
| `--crit` on `crit-bg/panel` | 4.5 | 4.94 | **5.34** | error chip |
| `--cr` on `panel` | 4.5 | 6.38 | **5.74** | credit figures |
| `--cr` on `ok-bg/panel` | 4.5 | 5.42 | **4.99** | ok chip |
| `--dr` on `panel` | 4.5 | 7.94 | **8.14** | debit figures |
| `--danger-fg` on `danger-bg` | 4.5 | 5.97 | **6.12** | error band |
| `--field-line` on `panel` | 3 | 3.33 | **3.27** | resting field underline on the sheet |
| `--field-line` on `panel2` | 3 | 3.15 | **3.08** | resting field underline on chrome |
| `--focus` on `panel` | 3 | 5.97 | **5.81** | focus underline |
| `--focus` on `accent-soft/panel` | 3 | 5.17 | **5.11** | focus underline on its own fill |

LIGHT-ONLY (new, checked in light only):

| foreground on surface | min | after | where it lands |
|---|---|---|---|
| `--dim` on `sel` | 4.5 | **5.03** | muted text on the cursor row |
| `--ink-2` on `sel` | 4.5 | **6.82** | secondary text on the cursor row |
| `--ink-2` on `tint` | 4.5 | **7.20** | secondary on tinted band |
| `--accent` on `sel-strong` | 4.5 | **4.69** | accent on pressed row |
| `--ink` on `hover-canvas` | 4.5 | **13.80** | canvas row hover |
| `--dim` on `hover-canvas` | 4.5 | **4.92** | muted on canvas hover |
| `--accent` on `hover-canvas` | 4.5 | **4.90** | accent on canvas hover |
| `--ink-2` on `bg` | 4.5 | **7.18** | secondary on canvas |
| `--accent` on `bg` | 4.5 | **5.27** | brand/links on canvas |
| `--hot` on `bg` | 4.5 | **5.27** | top-bar hotkey letter on canvas |
| `--warn` on `bg` | 4.5 | **5.74** | warning on canvas |
| `--crit` on `bg` | 4.5 | **5.55** | error on canvas |
| `--focus` on `bg` | 3 | **5.27** | focus ring on canvas |
| `--accent` on `accent-soft/bg` | 4.5 | **4.66** | hot rail key on canvas |
| `--ink` on `accent-soft/bg` | 4.5 | **13.14** | ink on soft accent over canvas |
| `--chip-green-fg` on `chip-green-bg` | 4.5 | **6.38** | chip green |
| `--chip-red-fg` on `chip-red-bg` | 4.5 | **5.91** | chip red |
| `--chip-amber-fg` on `chip-amber-bg` | 4.5 | **6.55** | chip amber |
| `--chip-blue-fg` on `chip-blue-bg` | 4.5 | **6.45** | chip blue |
| `--chip-violet-fg` on `chip-violet-bg` | 4.5 | **6.72** | chip violet |
| `--chip-teal-fg` on `chip-teal-bg` | 4.5 | **6.12** | chip teal |
| `--chip-slate-fg` on `chip-slate-bg` | 4.5 | **6.87** | chip slate |

### 5.5 The assert extension (Agent 1; extend, never relax)

Chips and `--hover-canvas` are consumed only by light-scoped rules, so their pairs are checked in light only. In `scripts/assert/visual.assert.ts`: read the scoped block as a third token table, make the light lookup prefer it, and add a light-only pair loop. Exact diff (verified green: 706 assertions):

```diff
@@ const light = tokenTable(...)
 const light = tokenTable((s) => s === ':root' || s === ':root, :root[data-density=\'compact\']');
+const lightScoped = tokenTable((s) => s === ':root[data-scheme=\'light\']');
 const dark = tokenTable((s) => s === ':root[data-scheme=\'dark\']');
@@ function val(...)
-  const raw = (scheme === 'dark' ? dark[name] ?? light[name] : light[name]);
+  const raw = (scheme === 'dark' ? dark[name] ?? light[name] : lightScoped[name] ?? light[name]);
@@ after the existing PAIRS loop, before "// ── 2a. density"
+const LIGHT_PAIRS: [string, string, number, string][] = [
+  ['dim', 'sel', 4.5, 'muted text on the cursor row'],
+  ['ink-2', 'sel', 4.5, 'secondary text on the cursor row'],
+  ['ink-2', 'tint', 4.5, 'secondary on tinted band'],
+  ['accent', 'sel-strong', 4.5, 'accent on pressed row'],
+  ['ink', 'hover-canvas', 4.5, 'canvas row hover'],
+  ['dim', 'hover-canvas', 4.5, 'muted on canvas hover'],
+  ['accent', 'hover-canvas', 4.5, 'accent on canvas hover'],
+  ['ink-2', 'bg', 4.5, 'secondary on canvas'],
+  ['accent', 'bg', 4.5, 'brand / links on canvas'],
+  ['hot', 'bg', 4.5, 'top-bar hotkey letter on canvas'],
+  ['warn', 'bg', 4.5, 'warning on canvas'],
+  ['crit', 'bg', 4.5, 'error on canvas'],
+  ['focus', 'bg', 3, 'focus ring on canvas'],
+  ['accent', 'accent-soft/bg', 4.5, 'hot rail key on canvas'],
+  ['ink', 'accent-soft/bg', 4.5, 'ink on soft accent over canvas'],
+  ['chip-green-fg', 'chip-green-bg', 4.5, 'chip green'],
+  ['chip-red-fg', 'chip-red-bg', 4.5, 'chip red'],
+  ['chip-amber-fg', 'chip-amber-bg', 4.5, 'chip amber'],
+  ['chip-blue-fg', 'chip-blue-bg', 4.5, 'chip blue'],
+  ['chip-violet-fg', 'chip-violet-bg', 4.5, 'chip violet'],
+  ['chip-teal-fg', 'chip-teal-bg', 4.5, 'chip teal'],
+  ['chip-slate-fg', 'chip-slate-bg', 4.5, 'chip slate'],
+];
+for (const [fg, bg, min, where] of LIGHT_PAIRS) {
+  let ratio = 0;
+  try { ratio = contrast(over(color(val('--' + fg, 'light')), surface(bg, 'light')), surface(bg, 'light')); } catch (e) { ratio = -1; }
+  check(ratio >= min, `light-only: --${fg} on ${bg} >= ${min} (${where})`, +ratio.toFixed(2));
+}
```
Why a separate light-only loop: the existing loop checks every pair in *both* schemes, and the new tokens deliberately have no dark value. (If the first diff line is omitted, dark would fall through to the light table for the `--kbd-*` trio and the assert would fail dark: that fall-through is the trap section 5.1 warns about.)

### 5.6 Overlay-only colours the assert cannot see (computed, must stay >= these)

| Colour | On | Ratio | Note |
|---|---|---|---|
| `.vbadge` text, `--tone-green` `#17784c` | `color-mix(tone 12%, #fff)` | 4.64 | |
| `--tone-red` `#b8412f` | same | 4.60 | |
| `--tone-blue` `#2a63bd` | same | 4.90 | |
| `--tone-violet` `#5d4bd0` | same | 5.21 | |
| `--tone-amber` `#8f6206` | same | 4.56 | the floor: do not raise the tint above 12% |
| `.kpi-ic` glyph, any tone | same tint | >= 4.56 | non-text needs 3:1 |
| keycap edge `#cfd5dc`, card edge `--border-lt` | white | 1.48 / 1.22 | decorative: the glyph/content identifies the component (keycap text 12.59:1) |
| cursor edge `--accent` | white | 5.81 | non-text >= 3 |
| `--field-line` field ring | white / `--panel2` | 3.27 / 3.08 | WCAG 1.4.11 component boundary |
| `--dim` / `--ink-2` on `chip-green/amber/red/teal-bg` | (subscription status cards) | 4.93-5.11 / 6.69-6.94 | worst is `--dim` on `chip-red-bg` 4.93 |
| table-cell inputs | n/a | n/a | **documented exception**: no resting boundary in dense grids; the cell is identified by column header + row, hover well, and a 5.81:1 inset focus ring |

---

## 6. System decisions

**Type.** IBM Plex Sans 400/500/600 only (the bundled weights; 650/700/750/800 in the old CSS snap to 600, so never write a weight above 600 in new rules). Body 13.5px compact / 14px comfortable (unchanged). Two label recipes, nothing else:
 * **L1 section label**: `font-family: var(--font); font-size: var(--fs-xs); font-weight: 600; letter-spacing: 0; text-transform: none; color: var(--dim)`.
 * **L2 column head**: `font-family: var(--font); font-size: var(--fs-sm); font-weight: 500; letter-spacing: 0; text-transform: none; color: var(--dim)`.
 Authored case is kept (CSS cannot sentence-case "GST" or "HSN"), so most heads read Title Case; that is fine and is a copy edit, not a CSS one.
 **Figures**: Plex Sans via `--num`, tabular by design, `letter-spacing: 0`. KPI figure 25px/600/-0.02em. **Mono** survives on exactly: keycaps (`--mono`), the typed alias in the palette `(rt)`, `code`/`<kbd>`-like tokens in snippets and API docs.

**Accent.** Keep the brand teal; it is not defined in `src/brand.ts` (that file holds the name only) and the problem was never its hue, it was a green canvas, a mint band and mint headers around it. `--accent` moves `#0b6e62` to `#0b7266`: the lightest teal that still clears 4.5:1 on its own soft tint (5.11), so it is a hair brighter without a contrast regression. Teal is *not* made more vivid: white-on-teal text needs L <= ~0.18, so vividness comes from pastel tints, not from the accent. Where teal appears: primary buttons (`.dash-primary .subs-cta .plan-btn .bulk-cta`, the armed answer `.acceptbar .opt.sel`), focus ring, the table cursor edge, links and mnemonic letters, the brand mark, active glyphs, progress fill. Where it does not: headers, row bands, idle borders, hover.

**Shape.** Radius 4 (keycaps) / 6 (fields, buttons, list rows) / 8 (cards, popups) / 12 (palette, key sheet, modals). Table rows stay square inside the card (the selection band is edge to edge). Chips and badges are pills (`999px`).

**Elevation.** L0 canvas. L1 card = `1px --border-lt` + `--shadow-1`. L2 popover (popup, accept bar, chord hint, sub-panel header) = `--shadow-2`. L3 modal / palette / toast = `--shadow-3` over a `--overlay-bg` scrim (`backdrop-filter: blur(3px)` stays where it already is). Always border **and** shadow, never border alone, never a shadow heavier than `--shadow-3`.

**Selection and focus.**
 * Table cursor: `--sel` wash **plus** the existing 2px accent bar on the first cell (kept; it is what finds the cursor after a glance away).
 * List cursor (menu, sidebar, popup, palette, key sheet): `--sel` wash, radius 6-8, **no** bar (menus keep the `›` caret and accent text).
 * Boxed field focus: `inset 0 0 0 1px var(--accent), var(--focus-ring)`. Grid cell focus: `inset 0 0 0 1.5px var(--accent)`. Buttons and links: the existing `outline: 2px solid var(--focus); outline-offset: 2px` (untouched).
 * Forced-colors: boxed inputs get `border: 1px solid CanvasText` (the box-shadow is stripped in that mode); the base file's own forced-colors rules stay.

**Chips and badges.** Seven pastel tones (5.3). Semantic map: `.chip.ok` green, `.warn` amber, `.crit` red, `.due` slate (existing classes keep their names and meanings); new opt-in classes `.info` blue, `.accent` teal, `.violet`, `.dr` blue, `.cr` green, `.slab-0/5/12/18/28` = slate/teal/blue/violet/amber. Voucher-type badges (`.vbadge`) keep the per-base tone mapping already in `modern/Dashboard.tsx` (`BASE_TONE`: sales/receipt/credit-note green, purchase/payment/debit-note red, contra blue, journal/payroll violet) as a 12% tint, no border.

**Buttons.** Primary: solid `--accent`, `--on-accent` text (5.81:1), radius 6, `--shadow-1`, hover `filter: brightness(1.07)` (existing). Secondary: white `--btn-bg`, `1px --border`, radius 6, `--shadow-1`, hover `--hover`. No new classes; the existing ones (`.fav-btn .ob-btn .bulk-ghost .sb-cta .subs-cancel-btn .quick button .themectl .tb`) get the secondary look from tokens plus one `box-shadow` each.

**Icons.** Keep the stroke set (24 grid, 1.6 stroke, `currentColor`): `--dim` at rest, `--accent` when active or selected; KPI icons sit in a 27px tinted tile (`.kpi-ic`). No fills, no emoji.

**Motion.** Hover/focus transitions use the existing `--dur`/`--dur-slow` (<= 120ms). Remove the KPI hover lift (`transform: translateY(-2px)`); a raised card does not need to move. `prefers-reduced-motion` handling in base stays.

**Density.** Comfortable is the same card system: `--frame-gap 12px`, radii identical, and the density rules that currently paint `.screen` back to `--bg` and heads to `--bg` are overridden with the `[data-density='comfortable']` form (reports.css). Known pre-existing issue, not made worse: comfortable's 24px cell padding wraps the widest reports (GSTR-1 at 1440px); the 12px card gap costs 12px more. Optional: `padding-inline: var(--sp-4)` on comfortable `table.tt td/th` (Agent 2).

---

## 7. Surface by surface (screen only; paper never changes)

Format per surface: **what it should look like**, the **selectors** and the **owner file** (section 8). "Token-only" means the re-valued tokens already do the work and no overlay rule is needed; do not add one. Appendix A is the reference CSS for everything marked (A).

### 7.1 Top bar: `shell.css` (A)
Sits **on the canvas**: no fill, no bottom rule, `--topbar-h` frozen (28 / 34px). Brand: a 9px `--accent` rounded mark (CSS `::before`) then "Comsky Books" in `--ink` 600 (it was accent-coloured text on a grey bar). Menu items (`K: Company ...`): words `--dim`, the mnemonic letter `--ink` with a **1.5px accent underline** at 3px offset (the underline is how the menu is read; the brick red goes), hover = `--hover-canvas` pill, disabled stay at 0.7 opacity. Go To pill: white, `1px --border`, radius 6, `--shadow-1`, the `⌘K` keycap on `--sunk` with no shadow. Theme/density buttons: the same white secondary pill. Selectors: `.topbar .brand .titem u .navtoggle .gotopill .gk .themectl .tb`.

### 7.2 Sidebar: `shell.css` (A)
On the canvas, width 186 frozen, no right rule. `.sb-today` stays a white card (token-only) with `--shadow-1`. Section labels `.sb-sec .sb-today-head` use L1. Rows: radius 6 (token), hover `--hover-canvas`; the **active row is a white pill** (`--panel`, `1px --border-lt`, `--shadow-1`), glyph `--accent`, text `--ink` 600; the 2px accent rail (`.sb-row.on::before`) is removed (the pill says "here", and the table cursor is the only thing that gets a bar). Star appears on hover/focus only (unchanged). Empty state, foot, focus outlines: token-only.

### 7.3 Gateway: `shell.css` (A)
Two **content-sized cards** instead of two full-height panes: `.gw-left` (period / company / counts) and `.tmenu-outer` (the menu). Cards align to the top (`align-items: start`) so a short menu does not become a tall empty rectangle. **Menu placement is untouched**: still the right-hand column, same width, same order, same single letters; text stays centred (centring is the Gateway's identity and not a "finish"). Sub-menus (Create / Alter) render an *empty* `.gw-left`; hide it with `visibility: hidden` on `:empty` so the menu does not jump to column 1 (verified). Title: 15px/600 `--ink`, hairline under it, the 28px accent bar under the title removed. Section labels: L1, centred, flanked by `--border-lt` hairlines (token-only). Items: radius 6; hover `--hover`; cursor = `--sel` wash + the `›` caret + accent 600 text, **no** inset bar; mnemonic letter = accent + 1px underline (`--hot` now equals accent). `.gw-block` rows: figures 15px/600 sans.

### 7.4 F-key button rail: `shell.css` (A)
On the canvas, no left rule; **width and every key/label metric frozen**. Header "Button bar" (sentence case; the only `content:` edit) in L1. Rows: radius 6 (token), hover `--hover-canvas`, the armed/active key (`.pbtn.hot`, "Use last", "Held (n)") = `--accent-soft` + accent text (token-only, 4.83:1 on soft-over-panel2, **4.66:1 on soft-over-canvas**, both in the assert). Keycap: white face (`--kbd-bg #fff`), `1px --kbd-edge`, **flat** (bottom border 1px + a 1px `rgba(16,24,40,.07)` drop), Plex Mono 600 10.5px (unchanged face, so the width budget holds). Group separators `.pbtn.gap::before`: `--border-lt` (token-only). Disabled keys 0.45 opacity (unchanged).

### 7.5 Status bar, banners, toast anchor: `shell.css` (A) and `overlays.css` (A4 for the lane)
Status bar on the canvas, no top rule; crumb `--ink-2` 500, rest `--dim`. `.storagewarn` / `.trialbar` become **inset rounded strips** (`margin: 0 var(--frame-gap) 4px`, radius 6) in chip-amber / chip-teal (`.warn` amber, `.lock` red) instead of full-bleed 10%-amber bars. `.ob-banner` (sample company) = white strip with a hairline, the "Sample" chip becomes a sentence-case amber pill. The feedback lane keeps its anchor, offsets and `--lane-lift*` logic untouched (7.11).

### 7.6 The screen card and report shell: `reports.css` (A)
**Every** screen (`.content > .screen`) is a card: `margin: 4px var(--frame-gap) var(--frame-gap) 0`, `1px --border-lt`, radius 8, `--shadow-1`, white. Left margin 0 because the sidebar already has its own gap; the 4px top matches the Gateway. `overflow: hidden` (base) clips content to the radius, and the shadow is deliberately tight (<= 2px spread) because `.content` is `overflow: hidden` and the 4/8px margin is all the room it has. Card header `.scr-head`: unchanged geometry (`6px 16px 5px`), title 15px/600, company name `--dim` right, the filter bar left of it. `.rpt-title` is hidden in compact (the card header already says it; comfortable has done this for a release) and `.rpt-sub` is left-aligned as the period subtitle. `.rpt-sub` and `.rpt-title` stay in the DOM and in the print clone's reach.

### 7.7 Report tables: `reports.css` (A)
 * **Head**: L2 recipe, `--panel` background (sticky, must stay opaque), a single `inset 0 -1px 0 var(--border)` underline (the old top rule goes).
 * **Rows**: hairline `--border-lt` (token-only), pitch unchanged (`padding: var(--row-y) var(--row-x) calc(var(--row-y) - 1px)` is asserted), hover `--hover`, cursor `--sel` + the 2px accent bar on the first cell (base rule, untouched), drill cap `↵` re-asserted mono and drawn as a keycap.
 * **Group / sub-header** (`tr.grp`, `tr.hdr2`): 600 weight; `hdr2` on `--panel2` with `--ink-2` text (was mint with teal text).
 * **Totals** (`tr.total`, `[data-rs='total']`): a 1px `--border` rule above, `--panel2` fill, 600, **no double underline** on screen (print keeps it: `print.css` re-states its own).
 * **Negatives** `[data-neg]` `--crit`; Dr/Cr figures `.amt-dr` `.amt-cr` token-only.
 * **Two-sided statements** `.twoside`: hairline divider (token-only).
 * **Filter** `.rf-in`, `.rf-hint`, `.rf-selbar`, `.rf-empty`, skeleton `.rf-skel-bar`: token-only; only `.rf-kbd` needs the mono re-assert.
 * **Registers**: marked-row tint (13% accent), tick box, `Optional` pill: the pill loses its caps/tracking (`text-transform: none`); the tint and the 3px marked-row bar are *state*, they stay.
 * **Chips / badges**: section 6 + Appendix A. Existing markup (CRM, webhooks, API keys, Bulk Entry, ageing) improves with no change.

### 7.8 Voucher entry: `entry.css` (A)
 * Header: title 17px/600, the voucher-number chip = `--accent-soft` fill + accent text, radius 6 (token-only; the number stays a chip).
 * **Header and party rows (`.frow`)**: labels `--dim` 13px, **no colon column** (`.frow .sep` hidden), fields are **boxed**: white, `inset 0 0 0 1px var(--field-line)` ring, radius 6, `padding: 1px 7px` (height unchanged, so compact rows do not grow). Focus: accent ring + 3px halo.
 * **Item and ledger grids** (`table.vt-entries`, `table.inv-items`, `.subpanel`, and every other `td input`): cells are **quiet**: transparent, no resting guide (the dotted rule is gone; column heads and row structure identify the cell), `--hover` well on cell hover, white cell with an inset 1.5px accent ring on focus. The entry row (`tr:has(input:focus)`) keeps its 7% accent tint and 2px bar (base, untouched). Heads use L2. This is the one WCAG 1.4.11 exception in the spec and it is called out in 5.6.
 * **Totals** `.vch-totals`: top rule `--border` (was near-black `--total-line`).
 * **Ghost recall / deltas / pin bar / calc chip** (voucher.css, styles.css): the dashed recall block keeps its dash (it means "not yours yet") on `--panel2`; its inner heads use L2-small; `.gh-tag/.gh-badge/.gh-d` are pills on translucent washes (token-only); the held-constants chips `.pb-chip` hover border accent (token-only); `.calchint` is already a small white popover.
 * **Sub-panels** (`.subpanel`): card on `--shadow-3`, radius 8 (token), header band `--panel2` with an L1 label.
 * **Narration** input is a boxed field; `.snip-open` chip is amber (token-only).

### 7.9 Master forms: `entry.css` (A)
Same boxed fields; `.frow label` keeps its 220px column (160px in `.inv-party`), section headings (`.fsection`) become L1 with 20px above instead of a mono eyebrow (comfortable's `color: accent` override stays: it is a section label, accent is allowed there). `.form` max-width 760px stays (do not widen; the Ledger form is four fields in a 1100px card and that is fine). Read-only values `.frow .static` `--ink-2`. Custom-field "Additional Details" inherits.

### 7.10 Popups, modals, accept bar, error band: `overlays.css` (A)
 * **List-of-ledgers popup** `.popup`: card, radius 8, `--shadow-2`, header L1 on `--panel2`; row cursor `--sel` + ink 500, **no inset bar**, balance column `.sub` in sans; "+ Create" row stays italic accent.
 * **Modal** `.box` (period prompt, confirmations): white, radius 12, `--shadow-3`, title row 13.5px/600 on `--panel2` (token-only), scrim `--overlay-bg` (lighter).
 * **Accept bar** `.acceptbar`: card radius 8, `--shadow-2`; the armed answer is a solid accent chip with `--on-accent` (5.81:1) and the focus ring; its old 3px accent left stripe becomes a 1px border (`.acceptbar.fb-accept`).
 * **Error band** `.errband`: an inset rounded strip (`margin: 0 var(--frame-gap) var(--frame-gap)`, radius 6) in `--danger-bg` / `--danger-fg` (6.12:1) instead of a full-bleed bar.

### 7.11 Palette, Go To, key sheet, chord hint, toast: `overlays.css` (A)
 * **Ctrl+K palette** `.cmdp`: already the closest to the target. Radius 12, `--shadow-3`; input row without a field box (palette/key-sheet/Go To inputs are explicitly un-boxed); group labels L1; row radius 8, cursor `--sel` (no bar), label stays ink 500 (not teal), the matched characters `<b>` accent 600; the right-hand scope chip is a sentence-case `--sunk` pill; the typed alias `(rt)` stays **mono**; keys on the right are keycaps.
 * **Key sheet** `.ksheet`: radius 12, group labels L1 (sticky on `--raised`), row cursor `--sel`, keycaps flat.
 * **Chord hint** `.kchord`: white popover (token-only).
 * **Toast / feedback lane** `.sl-card`: stays the **inverted** `--ink` chip (it must never look like the page); radius 6 and `--shadow-3` come from tokens; the error card keeps its 3px crit left edge (that one carries severity); `.sl-key` re-asserted mono.

### 7.12 Dashboard: `overlays.css` (A)
Already cards on canvas; finish it. KPI card: padding 16, **no 3px colour stripe**, label sentence-case 12px/500 `--dim`, figure Plex Sans 600 25px -0.02em, the icon in a 27px tinted tile (`color: var(--kpi-tone)`, 12% tint), hover = `--shadow-2` + `--border` (no lift). Quick-entry buttons: secondary buttons. `.vbadge` pills in Recent Vouchers (reports.css). Hero "New Sales Invoice" is the primary button.

### 7.13 Onboarding, Welcome, Getting Started: `overlays.css` (A) (banner/chip in `shell.css`)
Welcome is already cards on canvas; it picks up tokens, radius, and L1 for `.ob-eyebrow` / `.ob-keys-h`. `.ob-card-foot` dotted rule becomes a solid hairline; `.ob-ico` tile stays `--accent-soft`; `.ob-note` loses its 3px left stripe (1px border); the progress track `.ob-prog-bar` is `--sunk` with an accent fill; step cards `.ob-step-btn` white with a hairline, active step `--sel` + accent border. Keycaps `.ob-kbd` re-asserted mono. The confirmation dialog `.ob-dialog` is a modal (7.10).

### 7.14 Subscription / billing: `overlays.css` (A)
`.subs-status` loses the 3px stripe for a tinted card (`--tint` for trial, chip-amber for warn, chip-red for lock, 1px border); `.subs-badge` and `.plan-tag` lose caps/tracking; `.plan-card` white, radius 8, selected = `1px --accent` + soft ring (token-only); `.plan-price` Plex Sans 600; `.subs-cta .plan-btn` primary.

### 7.15 Favourites: `overlays.css` (A)
`.fav-lock-tag` becomes a sentence-case pill; `.fav-pick` is a modal (radius 12, `--shadow-3`); `.fav-opt.sel` = `--sel` (and loses the inset bar via the list rule in 6); `.fav-opt-s` sunk pill; `.fav-btn` secondary; `.fav-lk-grp h3` and `.fav-pick-t` use L1. Keycaps are the shared recipe.

### 7.16 Bulk entry: `entry.css` (A)
Tokens do most of it. `.bulk-card .k` and `.bulk-stat .l` use L1; `table.bulkgrid th` L2 on `--panel2`; `.bulk-key` and `.bulk-cta .kk` re-asserted mono; `.bulk-ghost` secondary; `.bulk-cta` primary (its `color: var(--bg)` on accent still passes, 5.27:1); `.bulk-paste` keeps its dashed drop-zone (a modern idiom, and it means "drop here"), `.bulk-done` loses the 3px green stripe for a hairline card. State stripes on grid rows (`tr.bad`, `tr.iffy`) stay.

### 7.17 Print-preview chrome: `overlays.css` (A)
Only the screen chrome around the paper: `.pv-stage` is the canvas (token-only), the sheet shadow is softened (`0 0 0 1px rgba(16,24,40,.08), 0 8px 24px -8px rgba(16,24,40,.18)`), `.pv-terms-ed` and `.pv-bank-*` are cards/lists from tokens. **`.pv-doc`, `.pv-sheet` internals, `.pr-doc`, `.print-report`, `.print-preview` content: do not touch.**

### 7.18 Base: `shell.css` (A) and tokens
`body` canvas (token), scrollbars (token: thumb `--border`), `::selection` (`--sel-strong`, token), `:where(button,a,...):focus-visible` outline (untouched), `.hk` letter (accent via `--hot`, underlined, unchanged rule). Forced-colors fallbacks from the base file stay.

---

## 8. Parallel plan: who owns which file and selectors

### 8.0 Step 0 (orchestrator, before fan-out, five lines, no design judgement)
1. Create `src/styles/light.css` containing exactly: `@import './light/shell.css'; @import './light/reports.css'; @import './light/entry.css'; @import './light/overlays.css';`
2. Create the four `src/styles/light/*.css` stubs, each: a one-line header comment + `@media screen {\n}`.
3. In `src/main.tsx` add `import './styles/light.css';` after `import './styles.css';`.
The agents share one working tree and one dev server, so Agent 1's token block is visible to everyone the moment it is saved. **Agent 1 lands the token blocks (5.1) first, in the first ~10 minutes**; the others write rules meanwhile and verify visually once tokens exist (until then `var(--chip-*)` is undefined and a chip is transparent: that is expected, not a bug).

### 8.1 Ownership (no selector appears in two files; if you need one you do not own, tell the orchestrator, never add it to your file)

| Agent | Owns (files) | Owns (selector families) |
|---|---|---|
| **1 Foundation + shell** | `src/styles.css` (5.1 blocks only), `src/styles/light.css`, `src/styles/light/shell.css`, `src/main.tsx` (the import), `src/styles/print.css` (**one line**, 9.3), `scripts/assert/visual.assert.ts` (5.5, extend only), `src/ui/admin/{ApiDocs,Webhooks,ApiKeys}.tsx` (the 9.1 `var(--num)` to `var(--mono)` rename only) | `.app .topbar .brand .titem .spacer .gotopill .gi .gt .gk .navtoggle .themectl .tb .main .navrail .nsec .nitem .nfoot` (dead rules: leave) **`.sb-*`** (sidebar.css), `.btnpanel .pbtn .pk .plabel .gap .hot` and the `.btnpanel .pbtn[title...]` rules in voucher.css, `.statusbar .crumb .sync-dot`, `.trialbar .storagewarn .storagewarn-x`, `.gateway .gw-* .tmenu* .tmenu-outer .quit`, `kbd .kcap .helpgrid .k` (the shared keycap recipe), `.hk`, the hold-Alt reveal rules in keyhelp.css, `.ob-banner .ob-chip .ob-link .ob-frame` (SampleBanner), base `body ::selection` scrollbars |
| **2 Reports** | `src/styles/light/reports.css` | `.content > .screen .scr-head .scr-body .scr-skel`, `.rpt* table.tt .r .total .hdr2 .grp .sev-* .amt-* .overdue .indent1 .dimtext .twoside .hscroll`, **`.chip .chiprow .vbadge`**, all of `report.css` (`.rf-*`, `[data-neg]`, `[data-rs]`, `.pk-tbl`, `.vt-opt`, `.rf-selbar`), all skeletons (`.ss-*`, `.rf-skel*`), the comfortable-density rules for the above |
| **3 Entry** | `src/styles/light/entry.css` | global `input / textarea / select` field treatment, `.form .frow .sep .wide .num .static .fsection`, `.inv-* .vch-* table.vt-entries table.inv-items .drcr .amt .qty .rate .gdn`, `.subpanel .pbody`, `.pinbar .pb-*`, `.ghost-recall .gh-* .ghost-deltas`, `.calcwrap .calchint`, `.snip*`, all of `bulk.css` (`.bulk*`, `table.bulkgrid`) |
| **4 Overlays** | `src/styles/light/overlays.css` | `.popup .plist .pitem .pt(popup) .create`, `.overlay .box .bt .bb`, `.acceptbar .q .opt` + `.acceptbar.fb-accept`, `.errband`, `.goto`, **`.cmdp*`**, **`.ksheet* .kchord*`**, `.statuslane .sl-* .fb-inline`, all `.ob-*` except banner/chip/link/frame, **`.fav-*`**, `.subs-* .plan-* `, **`.dash* .kpi* .quick .li-* .dash-primary`**, `.print-preview`, screen chrome in `print.css` (`.pv-stage .pv-terms-ed .pv-bank-*`, expressed in overlays.css, **not** by editing print.css) |

Dead rules (no TSX user, verified by grep): `.mshell .msidebar .msb-* .mmain .mtopbar .msearch .mtb-* .mcontent .mstatus`, `.navrail .nsec/.nitem/.nfoot`, `.helpgrid`. Leave them (baseline classes must keep a rule); do not style them.

Selectors that use `var(--num)` for a key, code or typed token and must **re-assert `font-family: var(--mono)`** in their own owner's file, because `--num` is now sans: Agent 1 `.gotopill .gk .pbtn .pk kbd .kcap .sb-kbd .helpgrid .k`; Agent 2 `.rf-kbd`, the drill-cap `::after`; Agent 3 `.bulk-key .bulk-cta .kk .snip code`; Agent 4 `.sl-key .ob-kbd .kchord-opts b .cmdp-label .cmdp-hint`.

### 8.2 Order and handoff
Parallel; no ordering except Agent 1's tokens first. When all four are done: Agent 1 (or the orchestrator) runs the section 10 gates in order. Commit only your own paths (`git add <path>`, never `-A`: the tree is shared). One commit per agent; message ends with the attribution line the harness gives.

---

## 9. Markup and non-CSS changes

**Required for acceptance: 9.1 and the first line of 9.3 (the print pin that keeps printed reports Plex Mono).** Everything else in this spec is CSS plus the five lines of Step 0. The list below is what CSS cannot do, ranked: 9.1 and 9.3 required, 9.2 strongly recommended, 9.4-9.5 optional (the `.vbadge` print reset in 9.3 is needed only if 9.4 is done).

### 9.1 Required (tiny, mechanical)
 * `src/main.tsx`: the import line (Step 0).
 * `scripts/assert/visual.assert.ts`: the 5.5 extension (extend, never relax).
 * **Code and identifiers must stay monospaced.** `--num` is now Plex Sans in light, and ten inline styles use it for things that must not become proportional (an API key where `l`/`I`/`1` and `O`/`0` must be distinguishable, base URLs, paths, request bodies). CSS cannot override an inline style, so change `'var(--num)'` to `'var(--mono)'` (a pure rename; in dark `--mono` is `var(--num)`, so dark is identical) at: `src/ui/admin/ApiDocs.tsx:92,102,112,321,347,380`, `src/ui/admin/Webhooks.tsx:357`, `src/ui/admin/ApiKeys.tsx:167,347,385`. (Agent 1, because it is a consequence of the `--num` re-point.) `einvoice.tsx` already uses the literal `monospace`.

### 9.2 Strongly recommended (fixes pre-existing AA failures; the only items that change dark, so they are the owner's call)
 * **Inline status chips with hardcoded colours.** `src/ui/tds/quarterly.tsx:44-58` (`StatusChip`: `#d05050` on its tint = **3.38:1**, `#4090c8` = **2.94:1**) and `src/ui/tds/tcs206c.tsx:39-50` (`CrossedChip`: `#4a9a55` = **2.89:1**) fail AA in light. Replace with `<span className="chip crit|accent|due|ok">`. Also `quarterly.tsx:247` and `tcs206c.tsx:144` (`color: '#d05050'` body text, **4.25:1**) to `color: 'var(--crit)'`. `ImportData.tsx:552` and `company/screens.tsx:305,495` use `#2e7d32` (5.13:1, passes; swap to `var(--green)` only for consistency).
 * `src/ui/reports/ageing.tsx:164`: the config popover has an inline `boxShadow: '0 4px 18px rgba(0,0,0,.4)'` that CSS cannot override. Change to `'var(--shadow-2)'`.

### 9.3 `src/styles/print.css`: one line (Agent 1; this is what keeps printed reports Plex Mono)
In the `.print-root .pr-doc { ... }` token block, after `--pp-mono`, add:
```css
  --num: var(--pp-mono);   /* the screen's figure face is sans in the light scheme; paper keeps Plex Mono */
```
If 9.4 is done, also add `.print-root .pr-doc .vbadge { background: none; border: 0; padding: 0; border-radius: 0; font: inherit; color: inherit; }`.

### 9.4 Recommended polish (colourful-but-restrained data screens)
The voucher-type column is plain text everywhere except the Dashboard. Render it as a badge in the three registers that show it: `src/ui/reports.tsx:710` (Day Book), `reports.tsx:1039` (Ledger Vouchers), `src/ui/reports/vchregisters.tsx:127` (registers): `<span className="vbadge" style={{ ['--vt-tone' as string]: toneFor(v.base) }}>{vtName(v)}</span>` with `BASE_TONE` moved out of `modern/Dashboard.tsx:24` into a shared module. `td.vt-opt::after` (the Optional pill), CSV export and print all read text and keep working. Needs the 9.3 `.vbadge` print reset.

### 9.5 Optional
 * `electron/main.cjs:30` `backgroundColor: '#cdd9e0'` to `'#f2f4f6'` (first-frame colour of the light canvas; today it flashes a blue-grey).
 * Copy: authored Title Case heads ("Vch Type", "Debit Amount") could become sentence case ("Vch type"); CSS cannot do this safely (acronyms).
 * `docs`: add a gotcha to `CLAUDE.md` "Conventions": *light-scheme finish lives in `src/styles/light/*.css` under `html:root[data-scheme='light']` inside `@media screen`; never put a hex into `:root` for a token dark does not override.* Update the baseline note when the overlay is folded into base.

---

## 10. Acceptance checklist (what the reviewers run)

Run in this order; stop at the first red. Items marked **[probe]** have a snippet below. All screen checks are at **1440x900, light, compact** unless stated; repeat 10.5 in comfortable.

### 10.1 Hard gates (mechanical)
- [ ] `npx tsc --noEmit` is **0 errors**; `npm run build` passes (Vite inlines `light.css` imports).
- [ ] `npm run assert` is green, **all 15 suites** (only `visual` reads CSS; it must show the 22 new light-only pairs and 0 failures).
- [ ] **Overlay lint** **[probe]**: `node lint-overlay.mjs src/styles/light/*.css` (each file is one `@media screen { }`; every selector starts `html:root[data-scheme='light']`; no `--font/--num/--mono` declarations; no motion > 150ms).
- [ ] **Dark isolation, static** **[probe]**: `node token-leak.mjs <main styles.css> src/styles.css` (dark block byte-identical; every `:root` value that changed is overridden by dark; density blocks untouched).
- [ ] `git diff --stat main` touches only: `src/styles.css`, `src/styles/light.css`, `src/styles/light/*.css`, `src/main.tsx` (1 line), `src/styles/print.css` (1-2 lines, 9.3), `scripts/assert/visual.assert.ts`, any 9.2/9.4 files if the owner opted in, and the ten `var(--num)` to `var(--mono)` renames in `ApiDocs.tsx`, `Webhooks.tsx`, `ApiKeys.tsx` (9.1). **No other `.ts/.tsx`**, so no key, route, menu or order change is possible.
- [ ] `visual.baseline.json`: no custom property or class removed (the suite checks this; confirm it ran).

### 10.2 Dark and print are untouched
- [ ] **Dark fingerprint** **[probe]**: on `main` and on the branch, in `data-scheme='dark'`, on every screen in 10.6, `__fp()` returns identical maps (**0 differing keys**). If `npm run shots` is present: `npm run shots -- --scheme dark --out /tmp/dark-main` vs `/tmp/dark-branch` are byte-identical (`cmp`) (shoot `main` twice first to prove the harness is deterministic).
- [ ] **Print**: in **light**, print to PDF (before / after) the Day Book, Trial Balance, GSTR-1 (wide, landscape), a Sales invoice, a Payment voucher, a cheque and the Reminder letters; page images are identical (same orientation, same zoom, double-underlined totals, Plex Mono chips). Same in dark.
- [ ] `grep -c "^@media screen" src/styles/light/*.css` is 1 per file; `grep -rn "pr-doc\|pv-doc\|pv-sheet\|print-report\|print-root" src/styles/light/` is empty (except the single `.pv-stage .pv-doc .pv-sheet` shadow line in `overlays.css`, which is screen chrome inside `@media screen`).

### 10.3 Frozen geometry **[probe]**
- [ ] `.main` columns read `186px <n>px 152px` on every screen; `.btnpanel` is exactly 152px.
- [ ] Compact report pitch **<= 21px** on Day Book, Trial Balance, GSTR-1, Ledger Vouchers (measured 20.875px with the prototype).
- [ ] Rail: **no `.pbtn` with `scrollWidth > clientWidth`**, no newly wrapped label, on the 21-key Sales voucher rail, Physical Stock voucher and GSTR-1 (its JSON-export key wraps to two lines by design, as before). Worst static pair stays <= 135 of 137px.
- [ ] No overlay rule sets `padding`, `gap`, `font-size`, `min-width` or `border-left/right-width` on `.pbtn`, `.pbtn .pk`, `.btnpanel`; none sets `--row-*`, `--head-y`, `--field-y`, `--topbar-h`.
- [ ] Keyboard parity: tab/arrow/Enter/Esc/F-keys/Ctrl+K/`?`/Alt+G behave exactly as on `main` (no `.ts/.tsx` diff makes this structural; spot-check the palette, a voucher, a report cursor).

### 10.4 Contrast and states
- [ ] Every pair in 5.4 passes in the assert; 5.6 values are not lowered (`.vbadge` tint stays 12%).
- [ ] Focus is visible and >= 3:1 on: boxed field, grid cell, button, link, table cursor (edge), menu cursor (caret + wash), sidebar item, palette row, rail key, Accept prompt answer, checkbox.
- [ ] Forced-colors (`emulate forced-colors: active`): boxed inputs show a border; focus shows an outline.
- [ ] Hover and cursor never fight: hovering the cursor row keeps the cursor look (`tr:hover:not(.sel)` rule in base is untouched); a marked row (`data-pk`) that is also the cursor row keeps >= 4.5:1 text.
- [ ] `prefers-reduced-motion: reduce`: nothing animates (base block untouched).

### 10.5 Surface-by-surface (look, then compare to 7.x)
| Surface | Must be true |
|---|---|
| Top bar | on canvas, no rule; brand mark + ink name; mnemonic letters ink with teal underline; Go To pill white with border and shadow |
| Sidebar | on canvas; Today is a white card; active row is a white pill with no accent rail; labels sentence case |
| Gateway | two content-sized white cards; menu still the right column, same order and letters; cursor = wash + `›`; sub-menus show no empty left card and the menu has not moved |
| Rail | on canvas; "Button bar"; flat white keycaps; armed key soft teal; 152px |
| Screen card | every screen has a 4/8px canvas gap, 1px border, 8px radius, tight shadow; no screen is edge to edge; `.errband` and `.acceptbar` sit inside it |
| Reports | no centred title, no duplicate title; heads 12px/500 dim, no caps, no tracking; cursor = wash + bar; totals single rule + soft fill, no double rule; negatives red; figures sans |
| Voucher | boxed header fields with accent focus ring; no `:` column; grid cells quiet, hover well, ringed focus; entry row tinted; total rule light |
| Master forms | boxed fields; section labels sentence case |
| Popup / modal / accept | white, rounded, soft shadow; list cursor wash, no bar; scrim light |
| Palette / key sheet | radius 12, un-boxed input, wash cursor, sentence-case group labels, pill scope chips, alias mono |
| Toast | still the inverted ink chip |
| Dashboard | no coloured stripes; tinted icon tiles; sans figures |
| Onboarding / billing / favourites / bulk | no 3px green/teal stripes (state stripes on grid rows stay); L1 labels; pills sentence case |
| Print preview chrome | canvas stage, paper sheet unchanged |
- [ ] **No remaining mono ALL-CAPS tracked label** anywhere on screen in light: `[...document.querySelectorAll('#root *')].filter(e => { const c = getComputedStyle(e); return e.children.length === 0 && e.textContent.trim() && c.textTransform === 'uppercase' && !e.closest('.print-root') }).length === 0` (allow-list: `.cmdp-input kbd`, the palette's scope chip, which stays an uppercase keycap).
- [ ] **No remaining mono text outside keycaps/alias/code**: the same sweep for `fontFamily` starting `"IBM Plex Mono"` returns only `kbd`, `.kcap`, `.pk`, `.gk`, `.sb-kbd`, `.ob-kbd`, `.sl-key`, `.rf-kbd`, `.bulk-key`, `.kk`, `.gh-key`, `.cmdp-label .cmdp-hint`, `code`, the `↵` drill cap, `.kchord-opts b`, and the ten inline code/identifier spans renamed in 9.1.

### 10.6 Screens for the probes
Gateway; Create menu; Day Book; Trial Balance; Balance Sheet; GSTR-1; Ledger Vouchers (Sharma Coaching Classes); Sales voucher (F8) with the ledger popup open; Sales voucher after Enter on the party (ghost recall + place-of-supply list); Quit prompt; Ledger Creation; Period prompt (F2); Ctrl+K palette with a query; F1 key sheet; Dashboard; Getting Started; Bulk Entry; Sales Pipeline (chips); Favourite Shortcuts; Billing/Subscription; Reminder Letters; sample-company banner; storage-warning strip.

### 10.7 Probes (paste into the page; they are what I used)

**Dark fingerprint** (returns `{ 'tag.classes#n': hash }`; run on `main` and on the branch with `data-scheme='dark'`, then diff):
```js
window.__fp = () => {
  const P = ['color','background-color','background-image','border-top-color','border-right-color','border-bottom-color','border-left-color',
    'border-top-width','border-bottom-width','border-top-left-radius','box-shadow','font-family','font-size','font-weight','font-style',
    'letter-spacing','text-transform','text-decoration-line','padding','margin','width','height','opacity','outline-style','display',
    'position','gap','text-align'];
  const h = (s) => { let x = 5381; for (let i = 0; i < s.length; i++) x = ((x << 5) + x + s.charCodeAt(i)) | 0; return (x >>> 0).toString(36); };
  const out = {}, seen = {};
  for (const e of document.querySelectorAll('#root *')) {
    if (!e.getClientRects().length) continue;
    const key = e.tagName.toLowerCase() + '.' + [...e.classList].sort().join('.');
    seen[key] = (seen[key] || 0) + 1; if (seen[key] > 4) continue;
    const c = getComputedStyle(e);
    out[key + '#' + seen[key]] = h(P.map((p) => c.getPropertyValue(p)).join('|'));
  }
  return out;
};
// dark: document.documentElement.setAttribute('data-scheme','dark'); JSON.stringify(__fp())  -> save; diff the two JSON files
```
Sensitivity check I ran on the prototype: light on vs off = 160/160 keys differ; dark on vs off = **0/160**.

**Geometry** (Day Book, Trial Balance, GSTR-1; and the voucher for the rail):
```js
const vis = (s) => [...document.querySelectorAll(s)].filter((e) => e.getClientRects().length);
const rows = [...vis('.scr-body')[0].querySelectorAll('table.tt tbody tr')].slice(2, 22);
const pitch = (rows.at(-1).getBoundingClientRect().top - rows[0].getBoundingClientRect().top) / (rows.length - 1);   // <= 21
const bp = vis('.btnpanel')[0];
({ pitch, panelW: bp.getBoundingClientRect().width,                                   // 152
   railOverflow: [...bp.querySelectorAll('.pbtn')].filter((b) => b.scrollWidth > b.clientWidth + 0.5).map((b) => b.textContent),   // []
   cols: getComputedStyle(vis('.main')[0]).gridTemplateColumns })                      // 186px <n>px 152px
```
(`vis()` matters: the shell keeps every earlier screen mounted under `display: none`, so a bare `querySelector` finds a hidden one.)

**Overlay lint** (save as `lint-overlay.mjs`; the version I tested):
```js
import { readFileSync } from 'node:fs';
const PREFIX = "html:root[data-scheme='light']";
const bad = [];
const splitTop = (p) => { const out = []; let d = 0, cur = ''; for (const ch of p) { if ('(['.includes(ch)) d++; if (')]'.includes(ch)) d--; if (ch === ',' && d === 0) { out.push(cur); cur = ''; } else cur += ch; } out.push(cur); return out; };
for (const file of process.argv.slice(2)) {
  const css = readFileSync(file, 'utf8').replace(/\/\*[\s\S]*?\*\//g, '');
  const top = css.trim();
  if (!/^@media screen\s*\{/.test(top) || !top.endsWith('}')) bad.push(`${file}: whole file must be one @media screen { ... }`);
  const stack = []; let i = 0;
  while (i < css.length) {
    const open = css.indexOf('{', i), close = css.indexOf('}', i);
    if (open === -1 && close === -1) break;
    if (open !== -1 && (close === -1 || open < close)) {
      const prelude = css.slice(i, open).trim();
      if (prelude.startsWith('@')) stack.push(prelude);
      else { stack.push('rule'); for (const sel of splitTop(prelude)) if (!sel.trim().startsWith(PREFIX)) bad.push(`${file}: selector without the light prefix: ${sel.trim().slice(0, 90)}`); }
      i = open + 1;
    } else { stack.pop(); i = close + 1; }
  }
  if (stack.length) bad.push(`${file}: unbalanced braces`);
  for (const m of css.matchAll(/(--font|--num|--mono)\s*:/g)) bad.push(`${file}: declares ${m[1]}`);
  for (const m of css.matchAll(/(transition|animation)[^;{}]*?(\d+(?:\.\d+)?)(ms|s)\b/g)) { const ms = m[3] === 's' ? +m[2] * 1000 : +m[2]; if (ms > 150) bad.push(`${file}: motion over 150ms: ${m[0].slice(0, 60)}`); }
}
if (bad.length) { console.log(bad.join('\n')); process.exit(1); }
console.log('overlay lint: ok');
```
Run: `node lint-overlay.mjs src/styles/light/*.css` (passes on Appendix A; fails, as it should, on a bare `.bad {}` and on a 300ms transition).

**Token leak** (save as `token-leak.mjs`; `git show main:src/styles.css > /tmp/before.css`):
```js
import { readFileSync } from 'node:fs';
const read = (f) => readFileSync(f, 'utf8').replace(/\/\*[\s\S]*?\*\//g, '');
const parse = (css) => [...css.matchAll(/([^{}]+)\{([^{}]*)\}/g)].map((m) => [m[1].trim().replace(/\s+/g, ' '), m[2]]);
const toks = (blocks, pick) => { const t = {}; for (const [s, b] of blocks) if (pick(s)) for (const m of b.matchAll(/(--[\w-]+)\s*:\s*([^;]+);/g)) t[m[1]] = m[2].trim().replace(/\s+/g, ' '); return t; };
const [before, after] = [parse(read(process.argv[2])), parse(read(process.argv[3]))];
const isRoot = (s) => s === ':root' || s === ":root, :root[data-density='compact']" || s === ":root[data-density='comfortable']";
const isDark = (s) => s === ":root[data-scheme='dark']";
const rootB = toks(before, isRoot), rootA = toks(after, isRoot), dark = toks(after, isDark), problems = [];
if (JSON.stringify(toks(before, isDark)) !== JSON.stringify(dark)) problems.push('the dark block changed');
for (const [k, v] of Object.entries(rootA)) if (rootB[k] !== v && !(k in dark)) problems.push(`${k} changed in :root but dark does not override it`);
for (const sel of [":root, :root[data-density='compact']", ":root[data-density='comfortable']"])
  if (JSON.stringify(toks(before, (s) => s === sel)) !== JSON.stringify(toks(after, (s) => s === sel))) problems.push(`density block changed: ${sel}`);
if (problems.length) { console.log(problems.join('\n')); process.exit(1); }
console.log('token-leak: ok');
```
Negative control I ran: putting `--kbd-ink: #2b3440` into `:root` fails with "changed in :root but dark does not override it".

---

## 11. Risks and open decisions

1. **Specificity is the whole mechanism.** An overlay rule that does not mirror its base selector, or that ties another overlay rule, will lose or flip silently. Mitigation: 4.3 (4, 5), the lint, and the `:where()` generic field rule. The only known tie-prone pair is the generic field rule vs `.cmdp-input input` / `.goto input` / `.ksheet-head input`; the generic rule is `:where()`-wrapped for that reason.
2. **Print leakage.** The way the print clone works (children of `.scr-body` under a body-level host, measured under screen media) means a bare `table.tt` or `.chip` overlay rule changes printed reports. Mitigated by `.screen` rooting, `@media screen`, and the `--num` pin; verified by the print comparison in 10.2.
3. **`.rpt-title` hidden in compact.** Comfortable has done this for a release, so every `ReportShell` screen has a card header title, but a bespoke screen with `.rpt-title` and no `.scr-head .t` would lose its only title. Reviewer: open every screen in 10.6 and any report reachable from the Gateway "Display More Reports"; there are 20 `rpt-title` call sites.
4. **Grid cells have no resting boundary** (the one WCAG 1.4.11 exception, 5.6). It is deliberate (a ruled grid at 20px pitch is the noisiest thing in the product) and is offset by hover well, header columns and a 5.81:1 focus ring. If the owner prefers strict compliance, set `td input` resting to `box-shadow: inset 0 -1px 0 var(--field-line)` (a solid underline, same zero layout cost).
5. **Accent is still teal.** I kept it deliberately (section 6). If the owner wants a bolder brand move, change only `--accent`, `--maroon`, `--btn-key`, `--hot`, `--focus`, `--accent-soft`, `--focus-ring` and the `--chip-teal-*` pair, then re-run the assert; nothing else references the hue.
6. **Gateway `.gw-left:empty`** relies on React rendering no child for sub-menus; if a future change renders whitespace or a placeholder there, an empty card would appear. Safer long-term fix is a modifier class in `MenuScreen.tsx` (not needed now).
7. **Inline-styled colours** in TSX (9.2) cannot be themed by CSS; the TDS chips fail AA today (2.89-3.38:1) and are the only places the new palette cannot reach.
8. **Browser support**: `:where()`, `:has()` (already used), `color-mix()` (already used), `:not(<list>)`; all within Electron 31 / Chromium 126 and current Safari.
9. **Shared working tree.** Four agents edit one checkout; disjoint files make this safe, `git add -A` does not (it is how another agent's work gets committed under yours).
10. **Comfortable density** was only eyeballed on Dashboard, Trial Balance and GSTR-1; its wide-report wrapping predates this work.

### Out of scope (do not do)
Dark restyle; any layout, order, key or copy change beyond `'Button bar'`; new fonts or font weights; the printed documents; renaming anything; removing the old light rules; changing the Electron shell beyond the optional first-frame colour.

---

## Appendix A: reference CSS (validated)

These four files are **exactly** the stylesheet I injected into the live page and measured (section 1). Each is a drop-in starting point for its owner's file: expand `@L` with `sed "s/@L/html:root[data-scheme='light']/g"`, then extend it to cover every selector family the owner has in 8.1 using the same recipes (L1/L2 labels, chip tokens, `.screen` rooting, mono re-asserts). Things the re-valued tokens already do are intentionally **not** restated. Parsed clean by esbuild; the expanded files pass the section 10.7 lint.

### A.1 `src/styles/light/shell.css` (Agent 1)

```css
/* src/styles/light/shell.css — Agent 1. Top bar, sidebar, button bar, status bar, banners, Gateway, keycaps.
   Everything inside @media screen; every selector starts with @L (see section 4). */
@media screen {

/* ── canvas: the chrome has no surface of its own, it IS the canvas ───────────────────────── */
@L .topbar { background: transparent; border-bottom: 0; }
@L .navrail { background: transparent; border-right: 0; }
@L .btnpanel { background: transparent; border-left: 0; }
@L .statusbar { background: transparent; border-top: 0; }

/* ── top bar ──────────────────────────────────────────────────────────────────────────────── */
@L .topbar .brand { color: var(--ink); letter-spacing: -0.01em; }
@L .topbar .brand::before {
  content: ''; display: inline-block; width: 9px; height: 9px; margin-right: 7px;
  border-radius: 3px; background: var(--accent);
}
/* the mnemonic letter keeps its underline (it is how the menu is read) but loses the red */
@L .topbar .titem u { color: var(--ink); text-decoration-color: var(--accent); text-decoration-thickness: 1.5px; text-underline-offset: 3px; }
@L .topbar .titem:not(.dis):hover { background: var(--hover-canvas); }
@L .navtoggle:hover { background: var(--hover-canvas); }
@L .gotopill { background: var(--panel); border-color: var(--border); border-radius: var(--radius); box-shadow: var(--shadow-1); }
@L .themectl .tb { background: var(--panel); border-color: var(--border); box-shadow: var(--shadow-1); }
@L .themectl .tb:hover { background: var(--hover); }

/* ── keycaps: one recipe, flat, white, mono (the ONLY mono left in the product) ───────────── */
@L .gotopill .gk, @L .pbtn .pk, @L kbd, @L .helpgrid .k, @L .kcap, @L .sb-kbd {
  font-family: var(--mono);
  border-bottom-width: 1px;
  box-shadow: 0 1px 0 rgba(16, 24, 40, 0.07);
}
@L .gotopill .gk { box-shadow: none; }

/* ── labels: one sentence-case recipe replaces the mono ALL-CAPS eyebrow ──────────────────── */
@L .btnpanel::before, @L .tmenu-section, @L .sb-today-head, @L .sb-sec, @L .gw-block h4 {
  font-family: var(--font); font-size: var(--fs-xs); font-weight: 600;
  letter-spacing: 0; text-transform: none; color: var(--dim);
}
@L .btnpanel::before { content: 'Button bar'; }

/* ── sidebar (sidebar.css selectors) ─────────────────────────────────────────────────────── */
@L .sb-today { box-shadow: var(--shadow-1); }
@L .sb-row:hover { background: var(--hover-canvas); }
@L .sb-row.on { background: var(--panel); border-color: var(--border-lt); box-shadow: var(--shadow-1); }
@L .sb-row.on::before { display: none; }                 /* the accent rail: a white pill says "here" */
@L .sb-row.on .sb-item { color: var(--ink); }
@L .sb-row.on .sb-gl { color: var(--accent); }

/* ── F-key rail. Metrics are LOCKED (152px): no padding / gap / font-size / min-width here ── */
@L .pbtn { border-radius: var(--radius); }
@L .pbtn:not(.dis):hover { background: var(--hover-canvas); }

@L .sb-cta { box-shadow: var(--shadow-1); }

/* ── banners ──────────────────────────────────────────────────────────────────────────────── */
@L .storagewarn, @L .trialbar { margin: 0 var(--frame-gap) 4px; border: 0; border-radius: var(--radius); }
@L .storagewarn { background: var(--chip-amber-bg); color: var(--chip-amber-fg); }
@L .trialbar { background: var(--chip-teal-bg); color: var(--chip-teal-fg); }
@L .trialbar.warn { background: var(--chip-amber-bg); color: var(--chip-amber-fg); }
@L .trialbar.lock { background: var(--chip-red-bg); color: var(--chip-red-fg); }
@L .trialbar:hover, @L .storagewarn:hover { filter: none; box-shadow: var(--shadow-1); }
@L .ob-banner { background: var(--panel); border-bottom: 1px solid var(--border-lt); }
@L .ob-chip {
  font-family: var(--font); font-size: var(--fs-xs); font-weight: 600; letter-spacing: 0; text-transform: none;
  padding: 1px 9px; border: 0; border-radius: 999px; background: var(--chip-amber-bg); color: var(--chip-amber-fg);
}

/* ── Gateway: two cards, not two panes ───────────────────────────────────────────────────── */
@L .gateway { align-items: start; gap: var(--frame-gap); padding: 4px var(--frame-gap) var(--frame-gap) 0; }
@L .gw-left, @L .tmenu-outer {
  align-self: start; max-height: 100%;
  background: var(--panel); border: 1px solid var(--border-lt); border-radius: var(--radius-lg); box-shadow: var(--shadow-1);
}
@L .gw-left:empty { visibility: hidden; }                /* sub-menus: keep the menu in column 2, draw no empty card */
@L .tmenu-title { background: transparent; }
@L .tmenu-title::after { display: none; }                /* the accent underline bar under the title */
@L .tmenu-item { border-radius: var(--radius); }
@L .tmenu-item.sel { background: var(--sel); box-shadow: none; }   /* the › caret + accent text carry "cursor" */

}
```

### A.2 `src/styles/light/reports.css` (Agent 2)

```css
/* src/styles/light/reports.css — Agent 2. The screen CARD (every screen), report tables, chips, badges, report.css.
   Rooted at .screen / .scr-body so the print clone (which sits under neither) can never match. */
@media screen {

/* ── the card every screen lives in ──────────────────────────────────────────────────────── */
@L .content > .screen {
  margin: 4px var(--frame-gap) var(--frame-gap) 0;
  border: 1px solid var(--border-lt); border-radius: var(--radius-lg); box-shadow: var(--shadow-1);
}
@L .screen .rpt-title { display: none; }                 /* the card header already says it (comfortable density already does this) */
@L .screen .rpt-sub { text-align: left; padding: 8px var(--sp-4) 2px; }

/* ── tables ───────────────────────────────────────────────────────────────────────────────── */
@L .screen table.tt th {
  font-family: var(--font); font-size: var(--fs-sm); font-weight: 500; letter-spacing: 0; text-transform: none;
  background: var(--panel); box-shadow: inset 0 -1px 0 var(--border);
}
@L .screen table.tt td.r { letter-spacing: 0; }
@L .screen table.tt tr.total td { background: var(--panel2); border-top: 1px solid var(--border); border-bottom: 1px solid var(--border-lt); }
@L .scr-body table.tt tr[data-rs='total'] td { border-top: 1px solid var(--border); }
@L .screen table.tt tr.hdr2 td { background: var(--panel2); color: var(--ink-2); }
@L .scr-body table.tt td.vt-opt::after { text-transform: none; letter-spacing: 0; font-weight: 500; }
@L .scr-body.rf-shell[data-drill] table.tt tbody tr.sel:not([data-rs]) td:last-child::after { font-family: var(--mono); }
@L .rf-kbd { font-family: var(--mono); }

/* ── chips + badges: pastel, opaque, sentence-case ───────────────────────────────────────── */
@L .screen .chip { font-family: var(--font); font-size: var(--fs-xs); font-weight: 500; letter-spacing: 0; text-transform: none; padding: 1px 8px; }
@L .screen .chip.ok    { color: var(--chip-green-fg);  background: var(--chip-green-bg); }
@L .screen .chip.warn  { color: var(--chip-amber-fg);  background: var(--chip-amber-bg); }
@L .screen .chip.crit  { color: var(--chip-red-fg);    background: var(--chip-red-bg); }
@L .screen .chip.due   { color: var(--chip-slate-fg);  background: var(--chip-slate-bg); }
@L .screen .chip.info  { color: var(--chip-blue-fg);   background: var(--chip-blue-bg); }
@L .screen .chip.accent{ color: var(--chip-teal-fg);   background: var(--chip-teal-bg); }
@L .screen .chip.violet{ color: var(--chip-violet-fg); background: var(--chip-violet-bg); }
/* opt-in classes for markup that wants them (nothing renders them yet): Dr/Cr suffix, GST slab */
@L .screen .chip.dr { color: var(--chip-blue-fg); background: var(--chip-blue-bg); }
@L .screen .chip.cr { color: var(--chip-green-fg); background: var(--chip-green-bg); }
@L .screen .chip.slab-0  { color: var(--chip-slate-fg);  background: var(--chip-slate-bg); }
@L .screen .chip.slab-5  { color: var(--chip-teal-fg);   background: var(--chip-teal-bg); }
@L .screen .chip.slab-12 { color: var(--chip-blue-fg);   background: var(--chip-blue-bg); }
@L .screen .chip.slab-18 { color: var(--chip-violet-fg); background: var(--chip-violet-bg); }
@L .screen .chip.slab-28 { color: var(--chip-amber-fg);  background: var(--chip-amber-bg); }
@L .screen .vbadge, @L .dash .vbadge {
  font-family: var(--font); font-weight: 500; letter-spacing: 0; border: 0; border-radius: 999px; padding: 1px 9px;
  background: color-mix(in srgb, var(--vt-tone) 12%, #fff);
}

/* ── comfortable density: same card, the density rules must not paint it back to canvas ──── */
@L[data-density='comfortable'] .content > .screen { background: var(--panel); }
@L[data-density='comfortable'] .scr-head { background: transparent; }
@L[data-density='comfortable'] .screen .rpt-sub { padding: var(--sp-3) var(--sp-6) var(--sp-2); }   /* compact's 8/16/2 must not win in comfortable */
@L[data-density='comfortable'] .screen table.tt th { background: var(--panel); box-shadow: inset 0 -1px 0 var(--border); }
@L[data-density='comfortable'] .screen table.tt tr.total td { border-top: 1px solid var(--border); border-bottom: 1px solid var(--border-lt); }

}
```

### A.3 `src/styles/light/entry.css` (Agent 3)

```css
/* src/styles/light/entry.css — Agent 3. Fields, master forms, voucher entry, sub-panels, ghost recall, snippets, bulk entry. */
@media screen {

/* ── fields. Form fields are BOXED (inset ring: no layout cost, the dense grids stay pixel-exact) ── */
@L input:where(:not([type='checkbox'], [type='radio'], [type='range'], [type='color'], [type='file'], [type='button'], [type='submit'], [type='reset'], [type='image'], [type='hidden'])) {
  background-image: none; background-color: var(--field); border-radius: var(--radius);
  box-shadow: inset 0 0 0 1px var(--field-line); padding: 1px 7px;
}
@L input:where(:not([type='checkbox'], [type='radio'], [type='range'], [type='color'], [type='file'], [type='button'], [type='submit'], [type='reset'], [type='image'], [type='hidden'])):focus {
  background-image: none; background-color: var(--field); box-shadow: inset 0 0 0 1px var(--accent), var(--focus-ring);
}
/* a field inside a table cell is a GRID CELL: quiet until hovered, ringed (inside the cell) on focus */
@L td input:where(:not([type='checkbox'], [type='radio'], [type='range'], [type='color'], [type='file'], [type='button'], [type='submit'], [type='reset'], [type='image'], [type='hidden'])) {
  background-color: transparent; box-shadow: none;
}
@L td:hover input:where(:not([type='checkbox'], [type='radio'], [type='range'], [type='color'], [type='file'], [type='button'], [type='submit'], [type='reset'], [type='image'], [type='hidden'])):not(:focus) {
  background-color: var(--hover);
}
@L td input:where(:not([type='checkbox'], [type='radio'], [type='range'], [type='color'], [type='file'], [type='button'], [type='submit'], [type='reset'], [type='image'], [type='hidden'])):focus {
  background-color: var(--field); box-shadow: inset 0 0 0 1.5px var(--accent);
}
@media (forced-colors: active) {
  @L input:where(:not([type='checkbox'], [type='radio'], [type='range'], [type='color'], [type='file'], [type='button'], [type='submit'], [type='reset'], [type='image'], [type='hidden'])) {
    border: 1px solid CanvasText;
  }
}

/* ── master forms ─────────────────────────────────────────────────────────────────────────── */
@L .frow .sep { display: none; }                         /* the ':' column belongs to the printed form */
@L .fsection {
  font-family: var(--font); font-size: var(--fs-xs); font-weight: 600; letter-spacing: 0; text-transform: none;
  margin-top: var(--sp-5);
}

/* ── voucher entry ────────────────────────────────────────────────────────────────────────── */
@L table.vt-entries th, @L table.inv-items th {
  font-family: var(--font); font-size: var(--fs-sm); font-weight: 500; letter-spacing: 0; text-transform: none;
  background: var(--panel); box-shadow: inset 0 -1px 0 var(--border);
}
@L .vch-totals td { border-top-color: var(--border); }
@L .inv-section-title, @L .subpanel .pt, @L .snip-preview .snip-k {
  font-family: var(--font); font-size: var(--fs-xs); font-weight: 600; letter-spacing: 0; text-transform: none; color: var(--dim);
}
@L .subpanel th {
  font-family: var(--font); font-size: var(--fs-xs); font-weight: 500; letter-spacing: 0; text-transform: none;
}
@L .screen .ghost-recall .gh-rows th {
  font-family: var(--font); font-size: var(--fs-xs); font-weight: 500; letter-spacing: 0; text-transform: none;
}
@L .snip code, @L .snip-help code, @L .snip-preview code { font-family: var(--mono); }

/* ── bulk entry ───────────────────────────────────────────────────────────────────────────── */
@L .bulk-card .k, @L .bulk-stat .l {
  font-family: var(--font); font-size: var(--fs-xs); font-weight: 600; letter-spacing: 0; text-transform: none;
}
@L table.bulkgrid th { background: var(--panel2); font-weight: 500; }
@L .bulk-key, @L .bulk-cta .kk { font-family: var(--mono); }
@L .bulk-ghost { box-shadow: var(--shadow-1); }
@L .bulk-done { border-left: 1px solid var(--border-lt); }   /* stripe callout -> plain card */

}
```

### A.4 `src/styles/light/overlays.css` (Agent 4)

```css
/* src/styles/light/overlays.css — Agent 4. Popups, modals, accept bar, palette, key sheet, feedback lane,
   onboarding, favourites, subscription, dashboard, print-preview CHROME (never the paper). */
@media screen {

/* ── popups / sub-menus / modals ─────────────────────────────────────────────────────────── */
@L .popup .pt {
  font-family: var(--font); font-size: var(--fs-xs); font-weight: 600; letter-spacing: 0; text-transform: none; color: var(--dim);
}
@L .popup .pitem.sel { background: var(--sel); color: var(--ink); font-weight: 500; box-shadow: none; }
@L .popup .pitem.sel .sub { color: var(--ink-2); }
@L .popup .pitem .sub { font-family: var(--font); }
@L .acceptbar.fb-accept { border-left: 1px solid var(--border-lt); }       /* stripe callout -> plain card */
@L .errband { margin: 0 var(--frame-gap) var(--frame-gap); border-radius: var(--radius); }

/* ── Ctrl+K palette, Go To, key sheet ────────────────────────────────────────────────────── */
@L .cmdp-grp-lbl, @L .ksheet-grp-lbl, @L .fav-pick-t, @L .fav-lk-grp h3 {
  font-family: var(--font); font-size: var(--fs-xs); font-weight: 600; letter-spacing: 0; text-transform: none; color: var(--dim);
}
@L .cmdp-item { border-radius: var(--radius-lg); }
@L .cmdp-item.sel { background: var(--sel); box-shadow: none; }
@L .cmdp-item.sel .cmdp-label { color: var(--ink); }
@L .cmdp-item > .cmdp-hint {
  font-family: var(--font); font-size: var(--fs-xs); font-weight: 500; letter-spacing: 0; text-transform: none;
  border-radius: 999px; background: var(--sunk);
}
@L .cmdp-label .cmdp-hint { font-family: var(--mono); }           /* the typed alias reads as input */
@L .cmdp-input input, @L .ksheet-head input, @L .goto input { background-color: transparent; box-shadow: none; border-radius: 0; }
@L .cmdp-input input:focus, @L .ksheet-head input:focus, @L .goto input:focus { background-color: transparent; box-shadow: none; }
@L .ksheet-row { border-radius: var(--radius); }
@L .ksheet-row.sel { background: var(--sel); box-shadow: none; }
@L .fav-opt.sel { background: var(--sel); box-shadow: none; }
@L .sl-key, @L .ob-kbd, @L .kchord-opts b { font-family: var(--mono); }

/* ── onboarding, subscription ────────────────────────────────────────────────────────────── */
@L .ob-eyebrow, @L .ob-keys-h, @L .subs-secttl {
  font-family: var(--font); font-weight: 600; letter-spacing: 0; text-transform: none;
}
@L .ob-card-foot { border-top-style: solid; border-top-color: var(--border-lt); }
@L .ob-note, @L .subs-status { border-left: 1px solid var(--border-lt); }   /* stripe callouts -> tinted cards */
@L .subs-status.ok   { background: var(--chip-green-bg); }
@L .subs-status.warn { background: var(--chip-amber-bg); }
@L .subs-status.lock { background: var(--chip-red-bg); }
@L .subs-badge, @L .plan-tag { font-family: var(--font); letter-spacing: 0; text-transform: none; font-weight: 600; }
@L .fav-lock-tag { font-family: var(--font); letter-spacing: 0; text-transform: none; font-weight: 600; border-radius: 999px; }

@L .fav-btn, @L .ob-btn, @L .subs-cancel-btn { box-shadow: var(--shadow-1); }

/* ── dashboard ────────────────────────────────────────────────────────────────────────────── */
@L .kpi { padding: var(--sp-4); }
@L .kpi::before { display: none; }                       /* the 3px colour stripe is the old admin-template tell */
@L .kpi .lbl { font-family: var(--font); font-size: var(--fs-sm); font-weight: 500; letter-spacing: 0; text-transform: none; }
@L .kpi .val { font-weight: 600; letter-spacing: -0.02em; }
@L .kpi-ic {
  box-sizing: content-box; width: 17px; height: 17px; padding: 5px; border-radius: 8px; opacity: 1;
  color: var(--kpi-tone, var(--accent)); background: color-mix(in srgb, var(--kpi-tone, var(--accent)) 12%, #fff);
}
@L .kpi.click:hover { transform: none; box-shadow: var(--shadow-2); border-color: var(--border); }
@L .quick button { border-color: var(--border); background: var(--panel); box-shadow: var(--shadow-1); }
@L .quick button:hover { background: var(--hover); border-color: var(--border); }
@L .dash-primary { box-shadow: var(--shadow-1); }

/* ── print-preview chrome (the sheet itself is paper and never changes) ──────────────────── */
@L .pv-stage .pv-doc .pv-sheet { box-shadow: 0 0 0 1px rgba(16, 24, 40, 0.08), 0 8px 24px -8px rgba(16, 24, 40, 0.18); }

}
```

## Appendix B: Step 0 files

`src/styles/light.css`
```css
@import './light/shell.css';
@import './light/reports.css';
@import './light/entry.css';
@import './light/overlays.css';
```
Each stub in `src/styles/light/`:
```css
/* src/styles/light/<name>.css, <owner>. Light scheme only; see docs/design/light-theme-spec.md section 4. */
@media screen {
}
```
`src/main.tsx` (after `import './styles.css';`):
```ts
import './styles/light.css';
```

## Appendix C: how to rebuild the contrast numbers

On a scratch copy of the repo (never the working tree): apply the 5.1 token edits to its `src/styles.css`, apply the 5.5 diff to its `scripts/assert/visual.assert.ts`, then `VISUAL_VERBOSE=1 node scripts/assert/run.mjs visual`. `light*` lines are the light-only pairs. Expected: `visual: 706 assertions passed`.
