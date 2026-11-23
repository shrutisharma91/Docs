# Comsky Books: the Settings window (Wave B), architecture and spec

> **Status:** design spec plus the scaffold it describes, ready for two builders working in parallel. **Author:** architect pass, 2026-10-20. **Scope:** the owner's last request of 2026-10-19: *shortcut buttons on the right-hand side or top that anyone can change, not hard-coded; and an admin settings window where the workspace administrator can change the theme, colour, fonts and font size.*
> **Reads before this:** `docs/design/zoho-ux-spec.md` (section 8 reserved Wave B; section 5 here says what this supersedes), `docs/design/light-theme-spec.md` (tokens, the overlay rules, the contrast method), `docs/research/zoho-books-reference.md` (what Zoho lets an admin change), `CLAUDE.md`.
> **Frozen and untouched:** every key, label, order and layout of the app at HEAD. **Nothing changes for a person who never opens Settings** (section 1.1 says how that is asserted). Operator rights and the write guard are untouched; settings are NOT books data and never write a voucher or a master; printed paper never changes; fonts are bundled, no CDN; no new dependency.
> **Evidence tags:** **[C]** read from code in this session (file named); **[V]** verified by running something (tsc, the assertion suites, a scratch probe in a throw-away headless Chrome); **[R]** from the research documents; **[J]** my judgement. The experiments behind the decisions are in section A.

---

## 0. The decisions, in one table

| # | Question | Decision | Section |
|---|---|---|---|
| 1 | The window | Screen `settings`, a **tab list from a registry** (`ui/settings/tabs.ts`): **Appearance** (a person's own look), **Shortcuts** (a person's own keys, right-hand buttons and top bar), **Workspace** (the administrator's defaults and locks, read-only for everyone else). `SCREEN_AREA 'settings'` is **`open`**, not `admin`: everyone changes their own; the administrator's part is judged inside (`canAdminister`) **and at the write** (`TABLE_AREA companies = admin`). | 1 |
| 2 | Ways in | Top-bar **gear** (fixed, icon only, right of the theme buttons), **Ctrl+, / Cmd+,** (free, registered, fixed, works inside a field), Go To "Settings", the Gateway Company menu (Q), Favourite Shortcuts. The left-menu row ("Preferences") is added at integration. | 1.4 |
| 3 | Theme | Light / Dark / **Same as my computer** (a `prefers-color-scheme` listener). Density stays Compact / Comfortable. | 2.1 |
| 4 | Accent | **Eight curated presets** (teal = today's, plus blue, indigo, violet, plum, berry, green, graphite). Each re-values **only the accent family** (13 light tokens, 11 dark) in both schemes. **All 141 pairs of the real `visual.assert` matrix pass for every preset** [V]. **No free hex in this wave** (a validated colour picker is a documented later step). | 2.3 |
| 5 | Fonts | **IBM Plex Sans** (default), **System font**, **Ledger (IBM Plex Mono)**. **Inter is not shipped**: it is not installed, the rupee sign cannot be proven from metadata, and the width budgets were solved for Plex. Figures keep the figure face; paper keeps `--paper-font`. | 2.4 |
| 6 | Text size | **90 / 100 / 110 / 125 / 150 %** by **tokens** (`--ui-k` multiplied into the type scale, row metrics, spacing, the two fixed columns and the left menu), **not CSS `zoom`**. Measured: row pitch scales exactly (20.88 to 26.09 to 31.31), no rail label or menu label clips, no page overflow at 1440, 1280 and 1024 [V]. The **top bar** must shed items instead of overflowing: that was the only thing that broke at 125 / 150 %. | 2.5 |
| 7 | Motion, contrast | **Reduce motion** (Normal / Reduce / Same as my computer) and **High contrast** (neutrals only, every text pair >= 7:1 on 13 surfaces, both schemes [V]). | 2.6 |
| 8 | Where values live | Layers, nearest wins: **locked by the administrator**, **personal** (localStorage `TW_UI_PERSONAL`, per company and operator), **company default** (`Company.ui`, synced, optional, no Dexie bump), **built-in**. The first frame is painted from `TW_UI_FIRST` by `index.html` before any bundle loads. | 3 |
| 9 | The company row | `Company.ui` is written by **one function** (`state/companyUi.ts`), as a read-modify-write of that one field on the freshest row, behind the write guard, with a revision counter. An alteration carries it (`alteredCompany` spreads). A device that remembers a policy an older app version erased offers to restore it. The server's `PROTECTED_FIELDS` is **not** widened (reasons in 3.4). | 3.4, 3.5 |
| 10 | Shortcut customisation | **One translation layer at the keydown level**: a per-person table `pressed chord -> the default chord`, applied in `keys.ts` before any handler, so **handlers keep seeing the defaults** and nothing in ~120 key sites changes. Labels follow through the one printer `keyLabel`. A person **adds a second key** for an action (default) and may **turn the original off**. | 4 |
| 11 | What is rebindable | The rail family: F2, Alt+F2, F3, Alt+F3, F4 to F10, Alt+F5, Alt+F6, Alt+F7, Ctrl+F7, F11, F12, Alt+C, Alt+D, Alt+P, Alt+N, Alt+K, Alt+2, Alt+B (**25 actions**), plus an **extra alias** (never a rebind) for Go To, the menu toggle, quick create and help. Everything else is **reserved** (section 4.3). | 4.2 to 4.4 |
| 12 | Rail and top bar | A button is identified by its **default chord** (`Btn.id ?? Btn.k`); a layout is `{hidden, order, labels}` applied by a pure function, per person and as a company default, lockable. **Hiding never disables a key.** The top bar's eight items get the same. | 4.5, 4.6 |
| 13 | The way back | Settings is always reachable by four routes none of which a setting can move; **"Reset all my settings"** (with a confirmation that arms No) clears every personal layer; **`?uireset=1`** clears them before first paint; an import of invalid JSON is **rejected whole**; the reserved keys cannot be rebound, so Go To always works. | 4.8 |
| 14 | Proof | `scripts/assert/uiprefs.assert.ts` (done, 196 assertions, 16 deliberate mutations each killed) is the contract; the builders extend `visual.assert.ts` and add suites (section 5). | 5 |

---

## A. What was verified

1. **The starting point is green [V].** `npx tsc --noEmit` 0 errors; `npm run assert` 72 suites, all green, before any change. After the scaffold: 75 suites at the time of writing (`uiprefs`, and two suites other agents added meanwhile), all green; `npm run build` passes.
2. **Where chords are declared [C]** (the audit the owner asked for): **69** `<ButtonPanel>` sites build `Btn { k, label, onClick }` lists; about **120** key-hook sites (`useScreenKeys`, `useKeys`, `useFallbackKeys`; 122 matches including their three definitions), of which **111** compare `combo(e)` against a literal; hosts that read the raw event (`App.tsx`: Ctrl+K, Ctrl+P, Ctrl+E and the generic F-key swallow; `NavChordHost` Ctrl+\\ via `isNavCollapseChord`; `QuickCreateHost` Alt+Q; `FavouritesHost` Ctrl+Alt+1..9; `KeyHelpHost` `?` and the G prefix; the new `SettingsChordHost`); about **26** element-level `onKeyDown` handlers and **2** `addEventListener('keydown')` on an input (the report filter and the views input), all of which own only typing keys, Enter, arrows and Esc; the top bar's `TOP_ITEMS` (display only: the letters are not bound); the Gateway's plain `hk` letters; `core/shortcutRegistry.ts` (now 4 entries, generating the `?` sheet and the locked list); the palette's `TALLY_KEYS` keycaps; `safeKey` pairs (5 call sites); about **95** user-facing strings that name a key in prose. **120** distinct chord literals (a bare key, or a key with Ctrl, Alt or Shift) exist in `src/`. **No handler reads F-keys or Alt chords from the raw event except through `combo()`**: grep for `e.key === 'F`, `.code === 'F` and direct `altKey` reads finds none outside `keys.ts`, `navPrefs.ts` (the menu chord) and typing-key gates. That is what makes a translation layer at the dispatcher sufficient.
3. **Zoom versus tokens, measured [V].** In headless Chrome 155 with `html { zoom: 1.5 }`: row pitch scales (20.88 to 30.81) but the layout viewport shrinks to 960 css px, the **top bar's min-content (1561px) overflows**, the whole `.app` grows to 1683px, **the F-key rail lands at x 1455 on a 1440px screen** (off-screen) and a hit test on its first button fails. Electron 31 is Chromium 126, which predates the standardised `zoom` (128): its geometry and `vh` semantics are different and **cannot be driven here** (the Electron binary is not installed in `node_modules`: `dist/` holds only `LICENSE` and `version`). A mechanism that cannot be verified on the shipped desktop build was rejected; tokens use only ordinary lengths and behave the same in every engine.
4. **The token mechanism, measured with the exact proposal [V]** (section 2.5; compact density; Gateway, Sales voucher with 23 rail buttons, Day Book, Trial Balance; viewports 1440x900, 1280x720, 1024x768; sizes 100, 125, 150): with the tokens scoped to `#root`, the two grid columns and `.pk` min-width scaled, and the top bar's items in a shrinkable, clipped group: **0 page overflow in all 36 combinations**, the rail ends exactly at the window edge (x + width = viewport width), **0 rail labels clipped, 0 menu labels ellipsised**, report row pitch **20.88 / 26.09 / 31.31**. Without the top-bar change the same proposal overflows at 150 % on 1440 and at 125 % on 1280.
5. **The accent matrix [V].** The real `PAIRS` (both schemes) and `LIGHT_PAIRS` tables of `visual.assert.ts` were run against each of the seven candidate presets: **141 pairs each, 0 failures**; the tightest pairs are listed in Appendix A.
6. **High contrast [V].** 13 surfaces x 4 text tokens, both schemes, computed (Appendix D).
7. **Print isolation [C].** `print.css` builds the paper's face as `--pp-font: var(--font, 'IBM Plex Sans')` (voucher and cloned report): **changing `--font` would change printed invoices and the on-screen preview**. `.print-root` (the print host, created by `reportTools.ts` and the voucher, cheque, bill-of-supply and reminder screens) is appended to `document.body`, **outside `#root`**. Hence: appearance overrides are scoped to `#root`, and `--paper-font` pins the paper's face (2.7).
8. **The company row [C].** `alteredCompany` spreads every field the form does not own, so a new `Company` field survives an alteration with no edit (asserted by `voucherconfig.assert.ts` reading the `Company` keys from `types.ts`, and again in `uiprefs.assert.ts`). `PROTECTED_FIELDS` is **server-side** (`backend/src/conflict.js`: `companies: vault, vaultLegacy, security`; `users: pinHash`); there is no such constant in the client. The writing guard for the company row is `TABLE_AREA.companies = 'admin'` through `writeProblem` / `putDoc`.
9. **Fonts [C].** Only `@fontsource/ibm-plex-sans` and `ibm-plex-mono` are installed. The registry has `@fontsource/inter` (checked with `npm view`), but a package's `unicode-range` says which code points a slice *may* carry, not whether the glyph exists. The repository's own proof for the rupee sign reads the `cmap` of the shipped `.woff` (`visual.assert.ts`, "latin-ext carries the rupee sign U+20B9"): any new face must pass that before it ships.
10. **The translation trick works on real events [V].** In headless Chrome, a `Proxy` over a real `KeyboardEvent` that overrides `key`, `code`, `ctrlKey`, `metaKey`, `altKey` and `shiftKey`: `combo(proxy)` reads `F8`, `Alt+D`, `Ctrl+F7` and `Ctrl+\\` for a pressed `Alt+9`; `isNavCollapseChord(proxy)` is true for the Ctrl+\\ default; `proxy instanceof KeyboardEvent`; `proxy.preventDefault()` and `stopPropagation()` reach the real event (`defaultPrevented` is true); a macOS Option+D arriving as the glyph `∂` still reads `Alt+D`; 200,000 lookups (build the pressed chord, `Map.get`) take about 21 ms in total, about 0.1 microsecond each.
11. **The scaffold works end to end [V].** A scratch probe in a throw-away headless Chrome (nothing saved, never the shared pane): `Ctrl+,` opens Settings from the Gateway and from inside a voucher field (Esc returns to the voucher); the gear opens it; arrow keys move between tabs with focus following; "Reset all my settings" asks first with **No** armed; the Gateway Company menu `Q` and Go To "settings" reach it (Settings first, Company Features second); the `<html>` attributes at rest are exactly `lang`, `data-scheme`, `data-density`; zero console errors.

---

## 1. The Settings window

### 1.1 What "a person who never opens Settings sees no change" means, and how it is asserted
Every value of every setting is optional in every layer, and **the built-in layer is the app as it is at HEAD**: light, compact, teal, IBM Plex Sans, 100 %, normal motion, standard contrast, the Menu view, the standard rail and top bar, the standard keys. In code (`src/core/uiPrefs.ts`):
- `BUILT_IN` is frozen and asserted equal to those eight values;
- `resolveAppearance(undefined, undefined)` is `BUILT_IN`;
- `htmlAttrsFor(BUILT_IN)` sets **only** `data-scheme` and `data-density` (the two attributes `<html>` carries today) and **no** attribute of the new family (`data-accent`, `data-font`, `data-ui-size`, `data-motion`, `data-contrast`): every new CSS rule is keyed on one of those, so for such a person **no new rule can match**;
- an empty rail layout, an empty top layout and an empty key table are identities (section 4 asserts the same reference comes back);
- the company row has no `ui` field at all until an administrator saves one (`serialiseCompanyUi(empty, 0)` is `undefined`).

### 1.2 The screen and its registries
`ui/admin/SettingsScreen.tsx` is the **shell** and does not name any tab. It draws `visibleTabs(admin)` from `ui/settings/tabs.ts`:

| Tab | Who fills it | Holds |
|---|---|---|
| `appearance` | Builder A | theme, density, accent, font, size, motion, contrast: a person's OWN look, a live preview, the lock the administrator may have set shown beside each control ("Set by your administrator") |
| `shortcuts` | Builder B | the keys table (record, add an alias, turn the original off, reset one or all), the right-hand buttons, the top bar, export / import as JSON, a pointer to Favourite Shortcuts |
| `workspace` | both, by panel | the administrator's part, drawn as **panels** from `ui/settings/workspacePanels.ts`: `appearance` (A: defaults, locks, the menu's default view), `shortcuts` (B: the standard rail and top bar, lock), `policy` (A: export, import, reset, restore). Read-only for everyone who may not administer, which is how an operator learns why a setting is locked. |

A future tab (notifications, numbering, print) is **one entry in `tabs.ts` and one file**; a future administrator section is one entry in `workspacePanels.ts`. The three placeholders in the scaffold are `SettingsPlaceholder` sections of a real screen: not `ComingSoon` and not `PlannedScreen` (their allow-lists in `sidenav.assert` and `zoho-scaffold.assert` are untouched).

**What a tab can ask of the shell** (`ui/settings/tabCtx.ts`): `useTabEsc(fn)` offers Esc to the tab first (an open recorder, a field with text to clear); `useTabButtons(buttons)` adds buttons to the right-hand rail above Quit. A tab registers **no global key handler** and never toggles `active`; its own keys use `useScreenKeys` (top-gated). The shell's Esc handler is itself a `useScreenKeys` and asks the tab first.

**Keyboard model of the shell.** The tab list is a vertical `role="tablist"` with a roving `tabIndex`; focus lands on the active tab when Settings opens; `Up`/`Down`/`Left`/`Right`/`Home`/`End` move between tabs (focus follows, activation is automatic) and stop there (React's `onKeyDown`, so the screen's key stack never sees them); `Tab` enters the panel; `Esc` leaves after the tab has had its turn. The last tab used is remembered for the session. The rail shows `Esc Quit`. Enter and Space on the list's own buttons stay theirs (`nav/ownKeys.ts`).

### 1.3 Who may open it, who may change what
`SCREEN_AREA.settings` changes from the scaffold's `'admin'` to **`'open'`** (decision 1; the old comment is replaced). Reasons:
- Zoho's own appearance setting is an administrator's; this product's owner asked for both an admin window **and** buttons "anyone" can change. A personal theme, font size and set of keys on one person's device is not company data and nothing the write guard exists for;
- the administrator's part is a write to the **company row**. `TABLE_AREA.companies` is `'admin'`, and `writeProblem` is asked **before** the write (`state/companyUi.ts`) and `putDoc` asks again. So with operator security ON a non-administrator who gets past the UI (a stale tab, a console) is refused by the same guard that protects every company setting. With security OFF everyone is the administrator, as for every company setting today.
- `canAdminister(ctx)` (`core/uiPrefs.ts`) is `!securityOn(ctx) || hasArea(operator.role, 'admin')` (a Full-access role has every area). It is passed to every tab and panel as `props.admin`; a panel renders its controls disabled when it is false. It is **not** the security boundary (the write guard is); it is what stops the UI offering what would be refused.
- With security on and nobody signed in the sign-in gate is on top and Settings cannot be reached (`SIGNED_OUT_OK` is unchanged).

### 1.4 Entry points (additive, none of which a setting can move)
| Entry | Where | State |
|---|---|---|
| **Gear** | `ThemeControls.tsx`: after the two toggles, icon only (the bar does not grow), `aria-label="Settings"`, a title that carries the chord. `onMouseDown` keeps the cursor where it was (a voucher field), `onKeyDown={ownActivate}` keeps Enter and Space for the button. A peek (`requestNav(..., 'goto')`): Esc comes back. It is **not part of any list a person can hide**. | done |
| **`Ctrl+,` / `Cmd+,`** | Registry entry `settings` (live, not configurable), host `ui/settings/SettingsChordHost.tsx` (one permanent `useFallbackKeys`, gates inside), match `core/settingsChord.ts` (Ctrl or Cmd, `e.key === ','`, **no Alt, no Shift**). Free: grep found no use of `'Ctrl+,'` or `Comma` in `src/`; not in `BROWSER_RESERVED`; the conventional "Preferences" chord (macOS, VS Code incl. the web build, Slack). **Works inside a field** (it types nothing and Settings opens on top of the entry) but not behind an overlay, a gate or a swallowing modal, and not while the key repeats. Listed on the `?` sheet and in Favourite Shortcuts' locked keys by the registry. On a layout where the comma key types a letter (Cyrillic, Turkish) it does not fire; the gear and Go To are the ways in there. Whether Safari or macOS Chrome gives the page the first chance at `Cmd+,` **[unverified]**: that is why it is one of four routes. | done |
| **Go To** | `Settings` (category Company) with aliases (`settings` as an exact code, plus appearance, theme, dark mode, accent, font size, zoom, shortcuts, keys, remap, button bar, top bar, preferences, high contrast, reduce motion...). `Company Features (F11)` already carried the alias `settings`; both are alias-exact, the shorter title wins a tie (`paletteRank`), so `settings` lands on this window with Company Features second and Sync Settings third [V]. The bare word `keys` is deliberately **not** an alias (an alias is a code and beats every title: it would put this window above API Keys, a change to a query that works today); `shortcuts` is, and lists Favourite Shortcuts second [V]. The keycap shows `Ctrl+,` (read from the registry). Also in Favourite Shortcuts' list (the palette drift guard needs both). | done |
| **Gateway** | Company menu (Alt+F3) Productivity section: `Settings`, key **Q** (every letter of the word is a key elsewhere in that menu, so it takes the one left over; not underlined, like Split Company Data's Z). Appended: nothing above moves. | done |
| **Left menu row** | **Integration step, not in the scaffold** (it edits the 64-row literal table of `sidenav.assert.ts`, which wave A's reviewers are still changing). Add one row to the `settings` group of `navTree.ts`: `{ id: 'settings.prefs', label: 'Preferences', to: { s: 'settings' }, aliases: ['appearance','theme','colour','font','size','shortcuts','keys','customise'] }` (a short label: the 212px column fits 144px of label, and "Appearance & Shortcuts" does not), then in `sidenav.assert.ts`: the `SPEC_ROWS` table, the eight `64` literals to `65`, the two "only Switch Company" expectations (an accounting-only role now also sees `settings.prefs`, because the screen is open; the signed-out and unresolved-operator cases are unchanged), and in `zoho-scaffold.assert.ts` the line "no left-menu row opens Settings yet". The group's own click target stays `companyAlter` (Company Details): the module's home does not change. | integration |
| **Quick-create (`+`)** | Not an entry: Settings creates nothing (`quickcreate.assert` already asserts that). | n/a |

### 1.5 The scaffold, and the contract it leaves
Delivered (all `tsc` 0, `npm run assert` green):
`core/uiPrefs.ts` (types, option catalogues, readers, precedence, the page-attribute map, the company policy codec, a person's store codec), `core/settingsChord.ts`, `state/uiPrefs.ts` (the browser glue: `usePersonalUi`, `updatePersonalUi`, `resetPersonalUi`, `useUiOwner`, `useCompanyUi`), `state/companyUi.ts` (`writeCompanyUi`, the ONE writer), `ui/admin/SettingsScreen.tsx` (the shell), `ui/settings/{tabs.ts, tabCtx.ts, Placeholder.tsx, AppearanceTab.tsx, ShortcutsTab.tsx, WorkspaceTab.tsx, workspacePanels.ts, SettingsChordHost.tsx, panels/*}`, `styles/settings.css` (tokens only), `styles/accent.css` (the validated preset tokens, **not yet imported**), `types.ts` (`Company.ui`, `CompanyUiStored`), `core/access.ts` (`settings: 'open'`), `core/shortcutRegistry.ts` (the `settings` entry), the entries of 1.4, and `scripts/assert/uiprefs.assert.ts`.
The two suites that named the stub (`zoho-scaffold`: its rights line and its "nothing links to Settings yet" lines) were changed deliberately, with the reason in the message.

---

## 2. Appearance

### 2.1 Theme and density
`ColorScheme` stays the two values the CSS knows (`light` | `dark`, `data-scheme`). A person's CHOICE has three: `light`, `dark`, `system`. `state/theme.ts` (Builder A) resolves `system` through `matchMedia('(prefers-color-scheme: dark)')` with a `change` listener, writes the **resolved** value to `data-scheme` (so every existing rule, light overlay and dark block works unchanged), and keeps the choice in the store. The top-bar toggle stays binary: it sets an explicit `light` or `dark` (never returns to `system`); while the choice is `system` it shows the resolved word and its title says "following your computer". Density is unchanged (`compact` | `comfortable`). `htmlAttrsFor(resolved, env)` in `core/uiPrefs.ts` is the **only** mapping from a choice to attributes; `theme.ts` applies it, `index.html` mirrors it (2.7) and the suite checks the mirror against it.

### 2.2 The cascade of the appearance files (Builder A)
All new rules are keyed on the new `<html>` attributes and **scoped to `#root`** (2.7). One file each, imported from `main.tsx` after `styles/light.css`, in this order (later wins an equal-specificity tie, and HC must win the focus ring):

```
styles/accent.css      data-accent        (done, Appendix A)
styles/fontface.css    data-font          (--font and --sans only)
styles/uiscale.css     data-ui-size       (generated: Appendix B)
styles/motion.css      data-motion
styles/contrast.css    data-contrast      (Appendix D)
```
plus a `#root { font-size: var(--fs); line-height: var(--lh); font-family: var(--font); color: var(--ink) }` rule **inside `uiscale.css` and `fontface.css` only** (a `body` rule outside `#root` would not follow a re-valued token; this was found in the measurement: without it row pitch was 22.19 instead of 26.09 [V]).

### 2.3 Accent colour: eight curated presets, none free
**Why a set and not a picker.** Each preset is checked against the whole matrix; a picked hex would need the whole derivation (washes, the dark counterpart, the on-accent) at runtime. A validated picker is a **later step** with a contract here (2.3.4), not part of this wave.

**2.3.1 What a preset may touch** (and the suite enforces it: a token outside these lists in `accent.css` fails):

| Scheme | Tokens | Why |
|---|---|---|
| light | `--accent --maroon --btn-key --hot --focus --accent-soft --focus-ring` | the accent family named in light spec section 11 risk 5 |
| light | `--sel --sel-strong --tint --field-focus` | **four washes that carry the hue**: the teal ones are `#e3f1ee #d4ece7 #edf6f4 #f3faf9`. The light spec said "nothing else references the hue"; that is not literally true, and a blue accent over a mint cursor row reads two-coloured. Re-hued (same saturation and lightness), proven by the same matrix. If the owner wants the strict list, drop these four and re-run: the pairs involving them are already in `PAIRS`. |
| light | `--chip-teal-fg --chip-teal-bg` | the "current / selected" chip follows the accent |
| dark | `--accent --maroon --btn-key --focus --accent-soft --focus-ring --on-accent` | `--on-accent` is dark ink on the bright dark accent, so it moves with it |
| dark | `--sel --sel-strong --tint --field-focus` | as above |

Never touched: the neutrals, the semantic colours (a red or amber accent would read as "something is wrong": `--crit` is hue 5, `--warn` hue 36, so there is none; green and blue are nearer to `--cr`/`--green` and `--dr` and are offered with that in mind), `--hot` in dark (it is the dark scheme's own warm hotkey tone, as for teal), `--on-accent` in light (white).

**2.3.2 The eight, with the numbers** (full tokens: `src/styles/accent.css`; the CSS is generated by the derivation below and must not be edited without re-running the matrix). Ratios are WCAG 2.x contrast.

| Preset | Hue | Light accent | Dark accent | Dark on-accent | Light: accent on panel = white on accent | Dark: accent on panel | Dark: on-accent on accent | Tightest light pair | Tightest dark pair |
|---|---|---|---|---|---|---|---|---|---|
| **teal** (default; no rule) | 174 | `#0b7266` | `#3ddbc6` | `#04211d` | 5.81 | 10.0 | 9.8 | `accent / sel-strong` 4.69 | (baseline) |
| blue | 212 | `#1b61b2` | `#85b4e9` | `#05111e` | 6.18 | 8.02 | 8.78 | ghost / accent-soft on panel 4.60 | same, 4.97 |
| indigo | 238 | `#464ad1` | `#a8abf0` | `#05061e` | 6.64 | 8.06 | 9.27 | ghost / accent-soft on panel 4.58 | same, 4.97 |
| violet | 264 | `#743dc7` | `#c0a3ec` | `#0f051e` | 6.49 | 8.01 | 9.14 | ghost / accent-soft on panel 4.58 | same, 4.98 |
| plum | 298 | `#9c2ea0` | `#e296e5` | `#1e051e` | 6.34 | 8.04 | 8.88 | ghost / accent-soft on panel 4.57 | same, 5.01 |
| berry | 332 | `#ad2966` | `#eb97be` | `#1e0511` | 6.39 | 8.02 | 8.94 | ghost / accent-soft on panel 4.55 | same, 5.01 |
| green | 146 | `#12763d` | `#2fc972` | `#051e10` | 5.70 | 8.03 | 8.10 | accent / accent-soft on bg 4.57 | `dim / sel` 4.75 |
| graphite | 214 | `#52637a` | `#a7b1bf` | `#05101e` | 6.13 | 8.00 | 8.81 | ghost / accent-soft on panel 4.64 | same, 4.96 |

Every preset: **141 pairs checked, 0 below its minimum**; every pair a preset touches has at least 0.05 to spare over its minimum (4.5 text, 3 non-text). The "tightest" are existing pairs that the preset's own `--accent-soft` or washes enter (`ghost` on a focused field, `accent` on the pressed row `--sel-strong`).

**2.3.3 The derivation** (so the assertion can recompute it, and so a ninth preset is one line): the light accent is the lightest colour of its hue (saturation per preset) with white on it >= 5.5 and every touched pair >= min + 0.05; the dark accent is the darkest with accent on `--panel` >= 8 and every touched pair >= min + 0.05, with `--on-accent` the same hue at lightness 0.07; the four washes are the teal ones re-hued; `--accent-soft` is the accent at alpha 0.09 (light) / 0.13 (dark); `--focus-ring` is 3px at 0.22 (light) / 2px at 0.34 (dark); `--hot` = `--accent` in light.

**2.3.4 The assertion matrix (Builder A, in `visual.assert.ts`; extend, never relax).** Parse `accent.css` (top-level rules, no `@media` wrapper, so the existing reader works); for each `(preset, scheme)` build an override table over the real token table (`val()` prefers it) and run `PAIRS` (that scheme) and, for light, `LIGHT_PAIRS`; assert: both scheme rules exist for each of the seven; teal has none; only the whitelisted tokens appear; every affected pair >= min + 0.05, every unaffected pair >= min; `--accent-soft` alpha in `{0.09, 0.13}`; the light swatches of `core/uiPrefs.ts ACCENTS` equal each preset's light `--accent`. **A free hex later** is accepted only through `deriveAccent(hex) -> { light, dark } | reject`, a pure function that runs this same derivation, adjusts lightness until every pair passes and says "adjusted for readability" (never silently), and is asserted over a sample of hundreds of hexes.

### 2.4 Fonts
| Option | `--font` | Figures (`--num`) | Why |
|---|---|---|---|
| **IBM Plex Sans** (default; no attribute) | unchanged | unchanged | what it is |
| **System font** (`data-font="system"`) | `system-ui, -apple-system, 'Segoe UI Variable Text', 'Segoe UI', Roboto, 'Helvetica Neue', Arial, 'Noto Sans', sans-serif` | **unchanged** | zero bytes; carries the rupee sign on SF, Segoe UI (Windows 8 onward), Roboto and Noto; width differs (SF is about 3 % wider than Plex regular, Segoe narrower: the rail wraps to two lines rather than clipping, and the measurement of 2.5 shows labels have room) |
| **Ledger (monospace)** (`data-font="ledger"`) | `'IBM Plex Mono', ui-monospace, 'SF Mono', Menlo, Consolas, monospace` | mono too (in light `--num` is set to `var(--font)`; in dark it is already mono) | already bundled with the rupee sign proven in `latin-ext` (all three weights); the look of a printed ledger; about 18 % wider text, so it is the choice most likely to wrap rail labels: the matrix below measures it |

**Figures stay tabular and keep their face.** A custom property that references `var(--font)` is substituted where it is declared: light's `--num: var(--font)` is already the Plex stack by the time `#root` re-values `--font`, so figures stay Plex Sans with no extra rule for **System**; **Ledger** re-declares `--num` in light only. `font-variant-numeric: tabular-nums` is already on `body` and the form controls, which is what keeps columns aligned in a face whose digits are proportional by default. Mono stays on keycaps and code (`--mono` is never touched).

**Paper never follows.** `print.css` `--pp-font` (two declarations, voucher and report) is changed from `var(--font, 'IBM Plex Sans')` to `var(--paper-font, 'IBM Plex Sans')`, where `--paper-font` is a new token in `styles.css` `:root` holding the original stack, and `--font` becomes `var(--paper-font)` (same computed value: byte-identical at rest, asserted). The scoping to `#root` already keeps `.print-root` clear; the pin is for the on-screen **preview** of a voucher, which lives in `#root`.

**Inter and any further face.** Not in this wave: (1) not installed; (2) the rupee proof is a cmap read, not metadata; (3) the 152px rail and 212px menu were solved with Plex advance widths. To add one later: one entry in the font catalogue, its `@font-face` rules **with `unicode-range` and both `latin` and `latin-ext`** in `fonts.css`, the `visual.assert` cmap proof for U+20B9, and the rail and menu width arithmetic re-run with that face. Nothing loads from a network.

**The width matrix (Builder A, `visual.assert.ts`).** The existing rail and menu arithmetic (`need()` / `wd()`) is parametrised over `(font, size)`: for Plex it is as today; for Ledger the advance is constant (`upm` 600 / 1000 per em); for System it is measured with a conservative 1.06 factor on Plex Sans (documented as such, because the OS face is not readable at test time). The assertion fails when a **static** rail pair or menu row would need more than 2 lines (the rail clamps at 2) or an ellipsis (the menu). It is a floor, like the existing one; the harness measurement (5.3) is the proof.

### 2.5 Text size: tokens, not `zoom`
**Decision.** Five steps, `data-ui-size` = `90 | 110 | 125 | 150` (100 sets nothing). One generated file, `styles/uiscale.css`, with one block per step and density (`html[data-ui-size='125'][data-density='compact'] #root { ... }`, comfortable likewise) that re-values:
1. the **type scale**: `--fs-2xs --fs-key --fs-xs --fs-sm --fs --fs-md --fs-lg --fs-xl --fs-2xl`;
2. the **row and chrome metrics**: `--row-y --row-x --head-y --field-y --pad --topbar-h --statusbar-h`;
3. the **spacing grid**: `--sp-1 .. --sp-8` (it is density-independent on purpose, but a 4px step beside 20px text looks cramped);
4. the **left menu's own tokens**: `--nv-w --nv-fs --nv-pad --nv-ic --nv-gap --nv-act`;
5. the **two fixed columns**, as rules (the base rules stay literal: `visual.assert` reads `186px 1fr 152px` from them): `html[data-ui-size='N'] .main { grid-template-columns: 186k 1fr 152k }`, `.app.nv-wide:not(.navhid) .main { var(--nv-w) 1fr 152k }`, `.app.navhid .main { 0 1fr 152k }`, and `.pbtn .pk { min-width: 26k }`.
Every value is `round(base * k, 3)px` with `k = size / 100` and `base` the density's own value (Appendix B has the tables). `--ui-k` is also set to `k` so a literal length can be written `calc(12px * var(--ui-k, 1))`, which is **identical at 100 %** (the fallback) and follows at any other: the 22 literal `font-size` px declarations in `styles/*.css` outside print are converted that way by Builder A; the **170 inline `fontSize: N` in TSX** (VoucherScreen, ApiDocs, leavebalance and others) are converted only in the high-traffic screens (VoucherScreen, forms, report shells, Dashboard) through one helper, and the rest are **documented as unscaled** with a ratchet count in the suite.

**Floors (scale up only).** Chrome heights and hit targets never go below today's: `--topbar-h >= 28px`, `--statusbar-h >= 24px`, `--nv-act`, `--qc-btn` and `--ph-h >= 24px` (WCAG 2.5.8). At 90 % they are `max(floor, base * 0.9)`.

**Why not `zoom`** (evidence in A.3): one mechanism for every engine; no `vh` hazard (`.app { height: 100vh }` stays right); no JS geometry surprises for the popups that anchor on `getBoundingClientRect` (the quick-create menu, the views dropdown, the filter panel); testable in the harness; `zoom` would be one rule, but it cannot be verified on Electron 31.

**Compact row pitch** at 100 % is untouched (20.875px; the existing assertion stays); at other steps it is `20.875k` (assert: `fs * lh + 2 * row-y == 20.875 * k`, within a pixel). The F-key rail is `152k` and the left menu `212k`; labels wrap at two lines in the rail and ellipsise in the menu exactly as today, and the ratio of text to box is the same as at 100 %, which is why nothing clips.

**The top bar must shed (Builder B owns the markup).** The one thing that broke at 125 and 150 % in the measurement is the top bar's min-content (brand, eight items, Go To pill, `+`, theme buttons, gear: about 1,044px at 100 %, 1,300 at 125, 1,560 at 150): it pushed `.app` wider than the window and the F-key rail off the right edge. Fix, required at **every** size (it also removes the existing overflow below 960px): the eight labelled items sit in a `.topitems` group (`display: flex; flex: 0 1 auto; min-width: 0; overflow: hidden`), with `.topbar`, `.gotopill`, `.statusbar` and `.app` given `min-width: 0` and the brand, `+`, theme buttons and gear `flex: none`. Items that do not fit are clipped and are listed in a **`...` overflow button** (a `ResizeObserver` on the group decides which; the list reuses the keyboard-driven popup) so a mouse user never loses Print or Export. The prototype in A.4 used exactly this CSS.

### 2.6 Motion and contrast
**Reduce motion** (`data-motion="reduce"`, set by Reduce, or by Same as my computer when `prefers-reduced-motion` matches; **Normal ignores the computer**, because that is what the app does today and the default is frozen): tokens `--dur` and `--dur-slow` to `0ms`, and `#root *, #root *::before, #root *::after { animation-duration: 0.01ms; animation-iteration-count: 1; transition-duration: 0.01ms; scroll-behavior: auto }` (specificity (1,1,1) beats any class rule, so no `!important`).
**High contrast** (`data-contrast="high"`): **neutrals only**, in both schemes (Appendix D): `--ink --ink-2 --dim --ghost --border --border-lt --field-line --total-line` and `--focus-ring: 0 0 0 3px var(--focus)`. Every text token is **>= 7:1 on all 13 surfaces** (panel, panel2, raised, bg, sunk, sel, sel-strong, hover, tint, accent-soft over panel and panel2, field, bar), borders >= 3:1. The accent family is not touched (accent text stays >= 4.5 as the matrix proves per preset). `contrast.css` is imported **after** `accent.css`, so its `--focus-ring` wins the equal-specificity tie.

### 2.7 First paint, and why paper is untouched
- **Attributes on `<html>`**, rules scoped to `#root`: every screen, dialog and the palette render inside `#root`; the print hosts do not (they are appended to `document.body`). So a printout can never take a screen colour, face or size, and nothing at all needs `@media screen`.
- **`index.html`'s inline script** (runs before React; Builder A extends the existing one): reads `TW_UI_FIRST` (a JSON snapshot of the **choices**, written by `theme.ts` after every change, shape `FirstPaint` in `core/uiPrefs.ts`), falls back to the legacy `TW_THEME_SCHEME` / `TW_THEME_DENSITY` (today's keys, still written by the top-bar toggles), evaluates `system` with `matchMedia`, sets `data-scheme` and `data-density` as today and sets the other attribute **only for a known non-default value**. `?uireset=1` in the address clears every `TW_UI_*` and `TW_THEME_*` key and removes the parameter with `history.replaceState` before anything paints. The whole script is in a `try`: blocked storage means the built-in look. The suite runs the script in a `vm` sandbox against a table of snapshots and compares the attributes with `htmlAttrsFor` (one source of truth).
- **The comfortable / compact and scheme toggles** write both the personal store and the legacy keys; **a reset** clears both (`resetPersonalUi`), or an old dark choice would come straight back as the "personal" one.

---

## 3. Workspace policy

### 3.1 Layers and precedence
For each appearance field, nearest wins:
1. **locked** by the administrator: the company's value, or **the built-in one if the administrator set none** (a lock never leaves a person on a stale personal choice);
2. **personal**: this person's choice on this device;
3. **company default**: `Company.ui.defaults`;
4. **built-in**.
Two legacy facts feed layer 2: the presence of `TW_THEME_SCHEME` / `TW_THEME_DENSITY` on a device **is** a personal choice (today they are only written when someone toggles), and the presence of `TW_NAV_VIEW` is a personal menu-view choice; the policy's `navView` applies to a device that has never chosen. `resolveAppearance(personal, company)` returns `{ value, source, locked }` per field so the UI can say "Set by your administrator" or "Your choice". Locks exist for the eight appearance fields and for `rail` and `top` (the standard button layouts); **keys are never lockable** (they depend on a person's keyboard).

### 3.2 Where each layer is stored
| Layer | Storage | Key | Notes |
|---|---|---|---|
| personal | `localStorage` | `TW_UI_PERSONAL` = `{ "<company>|<operator>": PersonalUi }` | one blob for appearance, keys, rail, top; owner = company + operator **only while security is on** (a shared PC gives each person their own); capped (64 owners, 64 entries a list, 2 MB); read defensively; every access in `try`; works with storage blocked for the session; `storage` events follow another tab |
| first frame | `localStorage` | `TW_UI_FIRST` | the resolved choices (2.7) |
| company | the **company row** | `Company.ui` (optional, synced) | 3.3 |
| legacy | `localStorage` | `TW_THEME_SCHEME`, `TW_THEME_DENSITY`, `TW_NAV_VIEW` | unchanged meaning |

A person's choices are a convenience on their device, not company data: not written through the change log, not subject to the write guard (which guards the books), exactly as the Dashboard's personal layout (`ui/modern/dashStore.ts`). Moving between devices is by export / import (4.7); a synced per-user layer is a later step (it needs a per-user table).

### 3.3 `Company.ui`
```ts
interface CompanyUiStored { v: number; rev?: number; [key: string]: unknown }   // types.ts (a leaf: loosely typed)
// as read by core/uiPrefs.ts sanitiseCompanyUi:
interface CompanyUi { rev; defaults: UiOverrides; locks: LockId[]; rail: RailLayout|null; top: TopLayout|null; extras; defaultsExtra; locksExtra }
```
Optional and additive: **no Dexie bump, no table, no migration**; a company without it is exactly what it was. The reader never throws and drops what it cannot trust (a wrong value, an unknown id, an over-long label, a control character, a duplicate); **what this build does not know is kept** (`extras`, `defaultsExtra`, `locksExtra`) and written back untouched, so an older build saving does not erase a newer build's setting (the Dashboard layout's rule). An alteration carries it: `alteredCompany` spreads, asserted in `uiprefs.assert.ts`.

### 3.4 The one writer, and why the server list is not widened
`state/companyUi.ts` `writeCompanyUi(update)`:
- reads the **freshest** company row from the store and builds the row with the pure `withCompanyUi(company, ui)` (`core/uiPrefs.ts`: `{ ...rest, ui }`, or the field removed): it never rebuilds the company, so the passcode vault, the security record, print, dashboard, GST history all pass through exactly as they are (the suite proves every other field stays the same object, and that mutating the helper to drop the vault, or the writer to build a fresh object, is caught);
- asks `writeProblem({ table: 'companies', ... })` first: Administration while security is on; `putDoc` asks again;
- writes **nothing when nothing changed** (so opening a panel never bumps a revision or syncs a row), and bumps `ui.rev` on every real save, an emptied policy included (`{ v, rev }`, never a deleted key);
- returns `null` or the reason, for the caller's own error band.
Both builders write the policy **only** through it; the suite fails if any other file calls `serialiseCompanyUi`.
**`PROTECTED_FIELDS` (server) is not widened**, decided on the code: its rule keeps the *current* value for a write that is **stale or unversioned** and raises a conflict whose copy says "Passcode / security change refused". A cosmetic policy saved by an up-to-date administrator whose `base` is one revision behind (another device re-saved the row meanwhile) would be silently reverted under that wording. The right server rule is narrower: protect `ui` only against an **unversioned** write (an older client sends no `base`). That is a six-line change in `backend/src/conflict.js` plus its suite and a VM deploy, which needs secrets only the owner has: **recommended, not done, and not required** for this wave because of 3.5. The overwritten version is kept by the Sync Mismatch Report either way.

### 3.5 An older app version erases the field (the vault-overwrite lesson, client side)
The exposure: a device with a copy of the company row from before the policy existed re-saves the row (an address edit) and the push replaces the whole row. Defence in depth, all additive:
1. every current client carries the field (3.3);
2. the policy has `rev`; each device mirrors the last non-empty policy it saw in `TW_UI_COMPANY_MIRROR:<cid>`; a row that arrives **without** `ui` while the mirror has `rev >= 1` is "erased by an older version" (a deliberate reset writes `{ v, rev }`, so absence is never a reset). That device **keeps applying the mirror** (so nobody's screen flashes back) and the Workspace > policy panel offers **Restore** (one `writeCompanyUi`), for an administrator; everyone else keeps seeing the mirror;
3. the pure detector `detectErased(mirror, storedRow)` is asserted (absent after rev 2 is erased; `{v, rev: 3}` is a reset; a lower `rev` than the mirror's is an older copy, not a newer policy).
Builder A owns this (the panel and the detector).

### 3.6 Export, import, reset (company policy)
Export: `{ "app": "comsky-books", "kind": "ui-policy", "v": 1, "ui": { ...CompanyUiStored } }`. Import (admin only): parse (<= 100 KB), check `kind` and `v`, sanitise, and require that the input **survives sanitising unchanged** in every known field (otherwise it is rejected **whole**, with the first thing that would have been dropped: "defaults.accent: 'red' is not an accent"). Nothing is applied on a failed import. "Reset the workspace policy" asks first (No armed), then writes the empty policy at the next revision. The sample company: allowed (it never syncs).

### 3.7 Multi-device and multi-operator
A pull of the company row from another device changes `company.ui`; `useCompanyUi` re-renders; the resolved appearance re-applies. Two administrators saving at once: last write wins and the loser's version is kept in the Sync Mismatch Report (an `edit-edit` on `companies`), as for any company setting. An operator signing in changes the owner, hence the personal layer, hence the look.

---

## 4. Shortcut customisation

### 4.1 The mechanism: a translation layer at the keydown level
**Rule: handlers keep seeing the defaults.** About 120 key sites compare `combo(e)` against literals; none changes. `keys.ts` `installKeyboard()` gains one step before the stack loop:

```ts
let table: ChordTable | null = null;                 // setChordTable(): null / empty = identity: ZERO cost (one null check)
window.addEventListener('keydown', (e) => {
  const ev = table ? translate(e, table) : e;        // identity returns the SAME event object
  if (ev === null) { swallowInert(e); return; }      // a default the person turned OFF: dead, and the browser does not get it either
  for (let i = stack.length - 1; i >= 0; i--) if (stack[i](ev)) { e.preventDefault(); e.stopPropagation(); return; }
  if (ev !== e) e.preventDefault();                  // the person's own key never falls through to the browser (Ctrl+S would save the page)
});
function translate(e, t) {
  if (e.ctrlKey && e.altKey) return e;               // AltGr (Windows) and the favourite-slot family: NEVER matched, NEVER inert
  const hit = t.byPressed.get(chordOfEvent(e));      // O(1): 'Alt+9' -> 'F8'
  if (hit) return translatedEvent(e, hit);           // a Proxy over the real event: key/code/ctrlKey/metaKey/altKey/shiftKey of the DEFAULT chord; every method bound to the real event
  return t.inert.has(plainCombo(e)) ? null : e;      // plainCombo ignores Shift, as combo() does (Shift+F8 has always been F8)
}
```
`chordOfEvent` is `combo()` plus Shift (so `Alt+Shift+9` can be told from `Alt+9`; the target grammar bans Shift in v1, so it only decides the lookup). `translatedEvent` sets the fields `combo()` and the nav-chord matchers read, so `combo(proxy) === 'F8'` and `isNavCollapseChord(proxy)` work; it is allocated **only on a remapped press**. Why a Proxy and not a synthetic `KeyboardEvent`: a re-dispatched event is seen by every other listener, loses `isTrusted`, and cannot share `preventDefault` with the real one. Why the dispatcher and not `combo()`: some handlers read `e.key` / `e.ctrlKey` directly (`App.tsx` Ctrl+K, P, E; the nav chord), so translating inside `combo()` would miss them.

**Inside text fields the remap behaves exactly like the original**, by construction: the handler that receives the default chord applies its own gate. F-key handlers fire in a field (as F8 always has); the Wave A chords that refuse a field (Alt+Q) still refuse it. A remapped press that no handler wants is still `preventDefault`ed (the person bound that key). **AltGr never matches**: Ctrl and Alt together are passed through untouched, and no target may contain both. On a Mac, Option+letter types a symbol, so the recorder warns before accepting one.

**Performance:** `Map.get` on a short string per keydown when a table exists (measured: about 0.1 microsecond a press, A.10); nothing when none does; the table is rebuilt only when the choice changes. The suite times 200,000 translations.

### 4.2 What is rebindable (a list of rail KEYS, each with a stable id), and what is not
`core/shortcutRegistry.ts` gains `REBINDABLE: { id, chord, label, does, group }[]` (Builder B; Appendix C). **The unit is the KEY of the rail, not one screen's use of it**, and that is stated on the row: the same key is used for the same kind of thing on many screens and for a different thing on some (F5 is Payment on entry screens and Detailed / Condensed on a statement; Alt+N is After accept on a voucher, New Snippet, Add Line, New Role; Alt+B is Batch on an item line and the Basis picker on a statement). A person's key moves **all** of a key's uses together, which is what "my key for what F5 does where I am" means, and `does` lists the uses so nobody is surprised. A person's key is stored against the stable **id** (`ChordPrefs[id]`), so a later change of a default cannot silently re-point it.

| Group | Actions (id: default chord) |
|---|---|
| Date, period, company | `date`: F2 (Date on an entry, Period on a report), `period`: Alt+F2, `company`: F3, `companyMenu`: Alt+F3, `features`: F11, `configure`: F12 |
| Vouchers | `vch:Contra` F4, `vch:Payment` F5, `vch:Receipt` F6, `vch:Journal` F7, `vch:Sales` F8, `vch:Purchase` F9, `vch:DebitNote` Alt+F5, `vch:CreditNote` Alt+F6, `vch:StockJournal` Alt+F7, `vch:PhysicalStock` Ctrl+F7 (**derived from `VCH_BASES.fkey`**, so the list cannot drift), `otherVouchers` F10 |
| While entering | `newLedger` Alt+C, `delete` Alt+D, `print` Alt+P, `afterAccept` Alt+N (also New Snippet, Add Line, New Role), `holdField` Alt+K, `duplicate` Alt+2, `batch` Alt+B (also the Basis picker on a statement) |
| Launchers (**alias only**) | `goTo` (Alt+G / Ctrl+K), `menuToggle` (Ctrl+\\), `quickCreate` (Alt+Q), `help` (F1): a person may add **one more key** for each; the reserved chords themselves keep working and are never moved or turned off |

That is **25 rebindable actions and 4 launchers**. Every other chord a rail shows is listed as `RAIL_FIXED` with its reason: screen-specific letter mnemonics used on one or two screens (Alt+A, Alt+Y, Alt+S, Alt+O, Alt+I, Alt+F, Alt+R), platform alias pairs (`safeKey`: Ctrl+L / Alt+L, Ctrl+T / Alt+T, Ctrl+H / Alt+V, Ctrl+J / Alt+J, and the recall pair Alt+R / Ctrl+Space), Ctrl+R, Ctrl+P, Ctrl+B, Ctrl+Z, Ctrl+V and the Wave A chords (`NEW_SHORTCUTS`, `configurable: false`). **A drift guard** scans `src/` for every `Btn` key literal and every chord literal and fails when one is in none of `REBINDABLE`, `RAIL_FIXED`, `RESERVED` or the registry (so a new chord cannot be forgotten, as `zoho-scaffold` does for the Wave A ones).

**This supersedes `zoho-ux-spec.md` section 8.2 and 8.3** for the rail's keys: that section reserved Wave B on the assumption that "configurable shortcut buttons" meant which optional actions sit on the rail, and ruled out moving a voucher F-key. The owner's request is explicit ("anyone ... change these shortcuts"), and the brief of this wave lists "the F-key rail voucher keys" as rebindable. What stays true: **the defaults do not move** (nothing changes until a person opts in), the registry's `configurable` flag stays `false` for every Wave A chord, and no key changes meaning for anyone who did not change it.

### 4.3 Reserved: never rebound, never taken
| Reserved | Why |
|---|---|
| Esc, Enter, Tab, Shift+Tab, Space, Backspace, Delete, the arrows, Home, End, PgUp, PgDn | the keyboard layer's own vocabulary; printable keys type |
| every plain printable key, digit and letter, and Shift+letter | typing; menu mnemonics; the left menu's letters |
| Ctrl+A (Accept), Ctrl+Z (undo), Ctrl+C, Ctrl+V, Ctrl+X | editing; the app's own meaning |
| Ctrl+K and Alt+G (Go To), Ctrl+\\ (the menu), Alt+Q (quick create), F1 and `?` (help), **Ctrl+,** (Settings), Ctrl+. (dismiss), Ctrl+Alt+1..9 (favourite slots), Ctrl+Space (recall), Ctrl+Enter, Alt+V (views), Alt+F12 and Ctrl+F12 (filters), Ctrl+F and `/` (find) | the launchers and Wave A chords: fixed so Go To and Settings always work |
| `BROWSER_RESERVED` (Ctrl+T, N, W, L, and Q, H, M on a Mac), **Alt+F4**, Ctrl+Tab, Ctrl+PageUp/Down, Ctrl+Shift+anything, Alt+Tab; in a browser tab also **Ctrl+digit** (tab switching) | a page cannot cancel them, or the browser owns them; the desktop app lifts the browser-only ones |
| any chord containing **both Ctrl and Alt** | AltGr on Windows, the favourite-slot family |
| any chord containing **Shift** in v1 | combo() does not encode Shift; `Shift+F8` has always been F8; claiming it would change an accidental alias. Revisit with evidence. |
| F5, F11, F12 as **targets** are allowed | they are shipped as Payment, Features and Configure today and `App.tsx` swallows every F-key with `preventDefault`; Electron has no menu (`Menu.setApplicationMenu(null)`), so there is no accelerator to collide with. Decided on that evidence, not on a general belief about reload. |

**Target grammar (what a person may choose):** `F2`..`F12` with no modifier, `Ctrl+`, or `Alt+`; or a letter or digit with exactly one of `Ctrl+` / `Alt+`; **not reserved, not browser-owned, not taken**. Pool (before the platform exclusions): 105 candidates, **44 free today** [V] (Ctrl+F2..F6, F8..F11; Alt+F8..F11; Ctrl+D G I O S U X Y; Ctrl+0..9; Alt+M U X Z 0 1 3..9), and a chord an action has vacated (turned off) becomes free. After the browser-only exclusions (Ctrl+digits, and Ctrl+D G I O S U X Y) the pool is **26**, 25 without Alt+U (held for the attachments panel; Alt+M, X, Z were kept in reserve by Wave A). Enough for most people's handful of changes; every key a person turns off also becomes free.

**Taken** is judged against EVERY registered chord: `NEW_SHORTCUTS`, `REBINDABLE` (unless that action turned its original off), `RAIL_FIXED`, `RESERVED`, the favourite slots, and a person's other custom keys. The data source is the checked-in lists, kept honest by the drift guard of 4.2, never by a runtime scan.

### 4.4 A person's key: add a second one (default) or move it
A `ChordPref` is `{ to, off? }`. **Default: the new key is added and the original keeps working** (the superset rule of Phase 20: accepting more is never a regression, accepting less is). **"Turn the original off"** is a separate switch per row, and the only way a default goes dead. The label everywhere is the person's key; the tooltip says "(also F8)" while the original is live. **Swap semantics:** assigning a chord that is another action's default is refused as "taken" unless that action has turned its original off (then it is free): to swap F8 and F9 a person turns both off and gives each the other's key, which is two explicit steps and visible in the table. (A one-click swap would silently remove a default.) A conflict message names the owner and the remedy: "Ctrl+F4 is already Receipt. Turn Receipt's original key off first, or pick another key."

### 4.5 Showing it: one printer
`keys.ts keyLabel(k)` is the **one printer** of a chord (the rail keycap, tooltips, the `?` sheet's `capLabel`, the left menu's dim keycaps, the palette, Favourite Shortcuts' locked list all go through it). It gains `displayChord(k)`: a leaf, DOM-free map (`core/chordMap.ts`, installed by `setChordTable`) that returns the person's key for a default chord that has one; identity otherwise, so every suite that runs under Node (`env-mac`, `env-win`) sees the defaults. Consequences, all asserted: the rail, the `?` sheet (`KeyCaps` lists the person's key first and the original as "or F8" while it is live), tooltips and menus follow with **no per-screen edit**; `keyHaystack` indexes both spellings, so the sheet's search finds a row by either; the hold-Alt reveal reads the keycap text. **Known limit:** about 95 user-facing strings name a key in prose ("set it (F2) before filing", "Company Features (F11)" as a Go To label that doubles as an alias identity). They keep the default name; the rail, the sheet and the tooltips are where keys are taught and they are dynamic. Builder B converts the handful that matter and lists the rest in the suite as a ratchet.

### 4.6 The right-hand rail and the top bar
**Identity.** A rail button is identified by its **default chord as the screen declares it** (`Btn.id ?? Btn.k`: `'F2'`, `'Alt+F2'`, `'Ctrl+F7'`). No screen's `Btn` list changes (69 sites untouched), one choice follows the same key across every screen, and the Wave A reservation `Btn.id` / `movable` stays available for a button that needs another identity. A top-bar item is one of eight ids (`company data exchange import export share print help`).

**The layout** (`RailLayout`, `TopLayout` in `core/uiPrefs.ts`, applied by pure functions in `core/railLayout.ts`, Builder B):
- **hide**: remove by id. **Never disables the key** (the `?` sheet still lists it). `Esc` and `Ctrl+A` cannot be hidden (Quit and Accept). A hidden button's `gap` (the hairline that marks a group) passes to the next visible one.
- **reorder**: the listed buttons are re-sorted **among the positions they already occupy**, so unlisted buttons and group breaks never move and a reorder can never push a Tally key out of its group.
- **relabel**: keyed `id|original label` (`F2|Date`), so it renames exactly the button that says that (F2 reads "Date" on a voucher and "Period" on a report); a label is 1 to 24 characters, no control characters, and may not equal the original. A longer label wraps (the rail clamps at two lines).
- **identity**: an empty layout returns **the same array object** (asserted by reference).
The top bar's items hide, reorder and rename the same way; the Go To pill, the `+`, the theme buttons and the **gear** are not in the list and never hide. The top bar is made data-driven (`TOP_ITEMS` stays the built-in list; the group and its overflow button are 2.5).
**Per screen kind or global?** Global, by id (decided): the Tally keys appear on 69 rails with the same meaning, so one choice is one place to change; per-screen layouts would be 69 lists to keep. A per-kind override is a later step.
**Priority.** P1: keys, hide, reorder, top bar, export / import, reset. P2 (after P1's acceptance): rail relabel (needs a catalogue of `(id, label)` pairs, kept honest by a scan of `k:` / `label:` literals), and a **"My buttons" block**: the assigned Favourite Shortcuts (Ctrl+Alt+1..9) drawn as extra buttons at the foot of the rail (the "Owner preset" idea of the Wave A spec), off by default.

**Company defaults and locks.** The Workspace > `shortcuts` panel (B) writes the company's standard `rail` and `top` and the `rail` / `top` locks through `writeCompanyUi`. **Keys are not a company setting.** Precedence for a layout is as for appearance (locked, personal, company, standard); a personal layout replaces the company's wholesale (section-level, not merged).

### 4.7 Storage, export, import
Keys, rail and top bar are parts of the personal blob (3.2). **Export** (Shortcuts tab; also in the company panel for the company layouts): `{ "app": "comsky-books", "kind": "ui-settings", "v": 1, "keys": {...}, "rail": {...}, "top": {...}, "appearance": {...} }`. **Import validation is all-or-nothing:** parse (<= 100 KB), check `kind` / `v`, then every section must survive its sanitiser **unchanged** and `chordRules.validate` must accept the key table **on this platform** (a chord that is reserved here, or taken, rejects the file with that reason); nothing is applied on failure and the message names the first offender. Chords are stored in the canonical `Ctrl` / `Alt` spelling (`Ctrl` is the Command key on a Mac), so a file moves between Windows, Mac and the browser; labels adapt through `keyLabel`. On load, a stored table is validated against the **current** rules and an entry that has since become reserved or taken (a build added a chord) is dropped with a note in the Shortcuts tab, never silently kept.

### 4.8 The way back, in full
A customisation can never strand a person, because **the things that get you back are not things a setting can touch**:
1. **Settings is reachable four ways** that no setting moves: the gear (fixed, not in any hideable list), `Ctrl+,` (reserved, translation-exempt), Go To (`Ctrl+K` / `Alt+G` are reserved, so the palette always works and finds "Settings"), the Gateway's Company menu (`Q`, a plain letter). Even if every F-key were remapped to nothing, Go To works.
2. **"Reset all my settings"** in the shell (confirmation, No armed first) clears the personal blob, the legacy theme keys and the first-frame snapshot in one call (`resetPersonalUi`); each tab has its own "Reset ..." (keys, buttons, top bar, appearance), and each key row has a reset.
3. **`?uireset=1`** on the address clears every `TW_UI_*` / `TW_THEME_*` key before first paint (the browser). The desktop app has no address bar: the gear, `Ctrl+,` and Go To are its way.
4. **Invalid input is rejected whole**: a bad import changes nothing; a corrupt stored blob reads as empty; a stored key that is no longer allowed is dropped on load.
5. **The translation layer is skipped entirely** when the table is empty, and a throw while building it is caught and means "identity".
6. The appearance cannot hide the way back: the gear is a fixed 33 x 21px button in the top bar at every size (it is `flex: none`, outside the clipped group), Settings has no setting that hides a keycap, and nothing hides the Esc button.

### 4.9 The recorder
"Change" opens a small modal (`useKeys`, on top of the stack, so it owns the keyboard): it **captures** the next chord with the same function the translation layer uses (`chordOfEvent`), ignores bare modifiers, shows the chord and a verdict beside it (free; or the reason: reserved, used by your browser, taken by X, contains Ctrl and Alt, Option types a symbol on a Mac), **Enter confirms** (only when valid), **Esc cancels**, **Backspace / Delete remove this action's key**. Those four keys are reserved, so they can never be recorded. The modal announces the verdict in an `aria-live` region, and the whole flow is reachable and finishable by keyboard.

---

## 5. Contracts, assertions, acceptance, risks

### 5.1 Ownership (disjoint files)
| | **Builder A: appearance** | **Builder B: keys, rail, top bar** |
|---|---|---|
| Tabs / panels | `ui/settings/AppearanceTab.tsx`, `WorkspaceTab.tsx` is the shell's (done); panels `AppearanceDefaultsPanel.tsx`, `PolicyPanel.tsx` | `ui/settings/ShortcutsTab.tsx`, panel `ShortcutDefaultsPanel.tsx`, `ui/settings/KeyRecorder.tsx` (new) |
| Core (pure) | `core/appearance.ts` (new: the font and size catalogues, `detectErased`, `deriveAccent` later), `core/uiPrefs.ts` additions | `core/chordRules.ts`, `core/chordMap.ts`, `core/railLayout.ts` (new), `core/shortcutRegistry.ts` (append `REBINDABLE`, `RAIL_FIXED`, `RESERVED`) |
| State | `state/theme.ts` (choice, `system`, listeners, `TW_UI_FIRST`, legacy read-through), `state/navStore.ts` (policy `navView` when no device choice) | `keys.ts` (`setChordTable`, `translate`, `keyLabel` -> `displayChord`), `state/uiPrefs.ts` additions only for keys |
| UI | `ui/ThemeControls.tsx` (the toggles' wiring only; the gear is done), `index.html` | `ui/widgets.tsx` (`ButtonPanel`: `applyRail`; `TopBar`: data-driven items, `.topitems`, overflow `...`), `ui/overlays.tsx` (`KeyCaps` lists the person's key first and the original while live) |
| CSS | `styles/accent.css` (done) wire-up, `fontface.css`, `uiscale.css`, `motion.css`, `contrast.css`, the 22 literal font sizes, `styles.css` (`--paper-font`, `--font` alias), `print.css` (the two `--pp-font` lines) | `styles/settings.css` additions for its tabs; the top bar rules (`.topitems`, `.topbar` min-width) |
| Assertions | `visual.assert.ts` extension (section 5.2), new `appearance.assert.ts` | new `chordrules.assert.ts`, `raillayout.assert.ts`, `keytranslate.assert.ts` |
| Recipes (`scripts/shots/recipes/`) | `settings-appearance.mjs` | `settings-shortcuts.mjs` |
| **Shared, one writer each** | `state/companyUi.ts` is the ONE writer of `Company.ui` for both; `core/uiPrefs.ts` is edited only by appending and only with its suite updated; `ui/settings/tabs.ts` and `workspacePanels.ts` are the architect's and nobody edits them |

**Rules for both.** Re-read a shared hot file immediately before each edit and make small surgical `Edit`s; never rewrite one. New chords are appended to the registry, never to `overlays.tsx` or `LOCKED_KEYS`. No `git add / commit / stash / checkout / reset`. No new dependency. Tokens only in CSS; no `display: none` on a grid item; no uppercase or tracked labels. `docs/parity-data.json` and `CLAUDE.md` are not edited by the builders. A new screen or table is not needed by this work.
**Cycles.** `ui/widgets.tsx` imports `ThemeControls` (which imports `navigate`); nothing under `ui/settings/` may be imported by `widgets.tsx` at module level.

### 5.2 Assertion plan
| Suite | State | Proves |
|---|---|---|
| `uiprefs.assert.ts` (196) | **done** | identity (defaults frozen; no new attribute at rest; empty layers change nothing), junk and prototype-pollution safety of every reader, the policy codec (round trip, `rev`, unknown keys kept, an older build saving), `alteredCompany` carries `ui`, precedence incl. locks, the personal store (owners, caps, size), `canAdminister` for off / admin / full / accountant / nobody, the Settings chord truth table (no Alt, no Shift), the registry entry, the shell and registries, ONE writer, no books write, tokens-only CSS |
| `visual.assert.ts` | **A extends** | the accent matrix of 2.3.4; high-contrast pairs (13 surfaces x 4 tokens x 2 schemes >= 7, borders >= 3); `uiscale.css` equals the generator's output for every step and density, compact pitch `== 20.875k` and `<= 21` at 100 %, floors (28 / 24 / 24), `.main` columns `186k / 152k`; the font width matrix of 2.4; `--paper-font` equals the old `--font` literal and is what both `--pp-font` use; no `accent` / `uiscale` / `fontface` / `contrast` rule touches `.print-root`, `.pv-doc` or `.pr-doc`; every rule of those files starts `html[data-...]` and ends in `#root`; **baseline names never removed** |
| `appearance.assert.ts` | **A** | the `index.html` script run in a `vm` sandbox against a table of snapshots equals `htmlAttrsFor` (incl. `system` with each `matchMedia` answer, a corrupt snapshot, blocked storage, `?uireset=1`); `theme.ts` listener and legacy fallback by source scan; `detectErased`; the panels are read-only without `props.admin` (source scan); a locked field's control is disabled |
| `chordrules.assert.ts` | **B** | the grammar, every reserved chord, browser-reserved per platform, AltGr, Shift refused, taken vs vacated, the alias-only launchers, **the drift guard** (every chord literal and every `Btn` key in `src/` is classified), `validate` on import (whole-or-nothing, the first reason), a stored table re-validated on load |
| `keytranslate.assert.ts` | **B** | an empty table passes **the same event object**; a remapped press reaches handlers as the default chord (`combo(proxy)`, `isNavCollapseChord(proxy)`); the original stays live in alias mode and is inert (and `preventDefault`ed) when turned off; Shift+F8 inert when F8 is; AltGr untouched; a remap that no handler wants is still `preventDefault`ed; 200,000 translations under a fixed budget; `keyLabel` is the identity with an empty table under `env-mac` and `env-win` |
| `raillayout.assert.ts` | **B** | identity by reference; hide, never-hide, gap transfer; reorder among occupied positions; keyed relabel; fuzz against random lists |
| `zoho-scaffold.assert.ts` | done (two lines changed) | the registry rules still hold with the `settings` entry |

### 5.3 Recipes and the sweep
`npm run shots` only (never the shared pane; recipes stop short of Accept; anything that must write is checked with a scratch script in a throw-away profile). **A:** `settings-appearance`: for scheme x density x each size x each accent: open Settings, the Gateway, a report, a voucher; measure page overflow, rail labels clipped, menu labels ellipsised, row pitch, and take the screenshot. **B:** `settings-shortcuts`: remap F8 to Alt+9 and press it on the Gateway and inside a voucher field; turn F9 off and press it; swap two; record a reserved and a taken chord; reorder and hide; export, break the JSON, import; "Reset all". Untouched screens must be **pixel-identical** to the current sweep (the attribute set at rest proves nothing new can match).

### 5.4 Acceptance checklist (reviewers run it; each item names its proof)
**Frozen things are frozen.**
- [ ] With nothing stored the `<html>` attributes are `lang`, `data-scheme`, `data-density` only, and the whole sweep is pixel-identical (`uiprefs.assert`, `npm run shots`).
- [ ] Every key, label, order and layout at HEAD: the rail on every screen that has one, the Gateway, the `?` sheet rows, the Go To order (the screenshot sweep and the existing suites).
- [ ] Defaults frozen in code: `BUILT_IN`, the first-frame script, `--font` computed value (`uiprefs`, `appearance`, `visual`).
- [ ] The compact row pitch is `<= 21px` at 100 % (`visual.assert`); the 152px rail and 212px menu are unchanged at 100 %.
- [ ] Paper is untouched: printed voucher, report and cheque are byte-identical with every font, size, accent and contrast (`visual.assert` scope and `--paper-font` checks; one print comparison in the sweep).
- [ ] No Tally wording; no hex outside the token files (`accent.css`, `styles.css`) and the cheque editor's existing one; no new dependency; fonts local.
**The owner's list.**
- [ ] **Appearance:** Light / Dark / Same as my computer (change the OS scheme and the app follows without a reload), density, eight accents (each in light, dark, compact, comfortable), three fonts (rupee signs render in the chosen face), five sizes, reduce motion, high contrast; each applies immediately and survives a reload with **no flash** (the first frame is correct).
- [ ] **Shortcut buttons:** a person can add a key, turn the original off, reset one or all, hide, reorder and (P2) rename rail buttons and top-bar items; the top bar sheds into `...` at 150 % on a 1280px window; labels everywhere show the new key (rail, tooltip, `?` sheet, palette).
- [ ] **Admin window:** the Workspace tab sets company defaults and locks (theme, accent, font, size, density, menu view, standard rail and top bar); a locked setting is disabled for everyone else with the reason; a non-administrator sees the tab read-only; with operator security ON a non-administrator's save is **refused by the write guard** (not only by the UI).
**The way back.**
- [ ] The reserved keys cannot be rebound or taken (every one of 4.3 by a test); remapping every F-key to nothing leaves Go To, Settings (`Ctrl+,`, the gear, the Company menu) and Esc working.
- [ ] `Reset all my settings` restores the workspace defaults, legacy theme keys included; `?uireset=1` restores before first paint; an invalid import changes nothing; a stored key that became reserved is dropped on load.
- [ ] AltGr (Ctrl+Alt) never matches a custom key (a Windows AltGr+letter test); inside a text field a remapped key behaves like the original.
**Rights and sync.**
- [ ] `Company.ui` survives Alter Company and an older-client re-save of the row on a device that holds the mirror (the Workspace panel offers Restore); two administrators saving at once leave the loser in the Sync Mismatch Report; `writeCompanyUi` never touches `vault`, `security` or any other field.
- [ ] Operator rights and the write guard are untouched (`security-access.assert` unchanged and green); Settings never writes a voucher or a master (`uiprefs.assert` source scan).
**Gates.** `npx tsc --noEmit` 0; `npm run build`; `npm run assert` all green (75 suites at the time of writing, plus the builders'); `npm run perf` 15/15; `npm run shots` over all screens in light and dark, compact and comfortable, zero console errors, `--strict`.

### 5.5 Risks and open decisions (the default taken is stated)
1. **`Cmd+,` on Mac browsers [unverified].** If Safari or macOS Chrome keeps it for the browser's own Settings, the chord does not reach the page; the gear, Go To and the Company menu are the other three routes, and Electron has no menu. Default: ship it; the suite tests only what the page receives.
2. **The top bar shedding is a behaviour change at every size** (below 960px it already overflowed). It is required for 125 / 150 %. Default: ship at all sizes, with the `...` button so nothing becomes unreachable.
3. **Four washes are re-hued with the accent** (2.3.1), beyond "only the accent family". Default: yes; a strict reading is a four-token deletion plus a re-run.
4. **Inline `fontSize` (170 sites) and icon pixel sizes do not scale.** Beside 20px text a 13px icon looks small at 150 %. Default: convert the high-traffic screens and the 22 CSS literals, ratchet the rest, accept the icons (an `.ic svg` size rule is a cheap follow-up).
5. **`Shift` is not allowed in a custom key** in v1 (combo() cannot see it). Default: refuse; revisit with a dispatcher that reads `shiftKey` for custom targets only.
6. **No custom hex** in this wave. Default: presets; the picker's contract is 2.3.4.
7. **A personal layout is per device**, not synced. A person on two machines exports and imports. Default: ship that; a per-user synced layer is a later step (needs a table).
8. **Non-Latin keyboard layouts**: `combo()` reads `e.key`, so on a Cyrillic or Devanagari layout even `Ctrl+A` is a different string today. A custom key recorded there is recorded in that layout's spelling. Pre-existing and out of scope; the recorder says what it saw.
9. **Prose that names a key** (about 95 strings) keeps the default name. Default: the rail, the sheet and tooltips are dynamic; a ratchet keeps the count from growing.
10. **Server `PROTECTED_FIELDS` is not widened** (3.4). Default: client defence in depth (3.5). Recommended later: protect `ui` against **unversioned** writes only.
11. **Electron `zoom`** is not used, so there is no `Ctrl+=` zoom in the desktop app; the size setting is its zoom. A `Ctrl+=` / `Ctrl+-` / `Ctrl+0` that steps the size (desktop only) is a one-line follow-up.
12. **The Workspace tab is visible to non-administrators, read-only.** It is how a locked setting explains itself. Default: visible.

---

## Appendix A. Accent presets
The complete, validated token sets are in **`src/styles/accent.css`** (14 rules: seven presets x two schemes; teal has none). The table in 2.3.2 is its summary. The derivation of 2.3.3 and the matrix of 2.3.4 reproduce it. Light rule shape (blue shown):
```css
html[data-accent='blue'][data-scheme='light'] #root {
  --accent: #1b61b2; --maroon: #1b61b2; --btn-key: #1b61b2; --hot: #1b61b2; --focus: #1b61b2;
  --accent-soft: rgba(27, 97, 178, 0.09); --focus-ring: 0 0 0 3px rgba(27, 97, 178, 0.22);
  --sel: #e3eaf1; --sel-strong: #d4dfec; --tint: #edf1f6; --field-focus: #f3f6fa;
  --chip-teal-bg: #dce6f2; --chip-teal-fg: #0b3463;
}
html[data-accent='blue'][data-scheme='dark'] #root {
  --accent: #85b4e9; --maroon: #85b4e9; --btn-key: #85b4e9; --focus: #85b4e9;
  --accent-soft: rgba(133, 180, 233, 0.13); --focus-ring: 0 0 0 2px rgba(133, 180, 233, 0.34); --on-accent: #05111e;
  --sel: #1c2a3b; --sel-strong: #273a50; --tint: #171c21; --field-focus: #1f262d;
}
```

## Appendix B. Size tables (generated: `value = round(base * k, 3)px`, `k = size / 100`)
Compact bases: `--fs-2xs 10, --fs-key 10.5, --fs-xs 11, --fs-sm 12, --fs 13.5, --fs-md 15, --fs-lg 17, --fs-xl 20, --fs-2xl 25, --row-y 2, --row-x 10, --head-y 4, --field-y 2.5, --pad 8, --topbar-h 28, --statusbar-h 24`. Comfortable bases: `10.5, 11, 11.5, 12.5, 14, 16, 19, 23, 29, 6, 13, 8, 6, 14, 34, 26`. Shared: `--sp-1..8 = 4 8 12 16 20 24 32 40`, `--nv-w 212, --nv-fs 13, --nv-pad 5, --nv-ic 16, --nv-gap 7, --nv-act 24`. Columns `186 / nv-w 212 / rail 152`, `.pk` min-width `26`. Floors: `--topbar-h >= 28`, `--statusbar-h >= 24`, `--nv-act`, `--qc-btn`, `--ph-h >= 24`.

| Step | `--fs` compact / comfortable | `--row-y` | compact row pitch | `--nv-w` | rail | `--sp-2` |
|---|---|---|---|---|---|---|
| 90 | 12.15 / 12.6 | 1.8 / 5.4 | 18.79 | 190.8 | 136.8 | 7.2 |
| 100 | 13.5 / 14 (no block) | 2 / 6 | 20.875 | 212 | 152 | 8 |
| 110 | 14.85 / 15.4 | 2.2 / 6.6 | 22.96 | 233.2 | 167.2 | 8.8 |
| 125 | 16.875 / 17.5 | 2.5 / 7.5 | 26.09 | 265 | 190 | 10 |
| 150 | 20.25 / 21 | 3 / 9 | 31.31 | 318 | 228 | 12 |

(Measured in the harness: 20.88, 26.09, 31.31 at 100, 125, 150 [V].)

## Appendix C. The rebindable catalogue, the fixed lists, the file shapes (Builder B)
```ts
// core/shortcutRegistry.ts (append; never edit the Wave A entries)
export interface RebindableAction { id: string; chord: string; label: string; /** what the key does, per kind of screen: written so a person sees every use that will move with it */ does: string; group: 'dates' | 'vouchers' | 'entry' | 'launcher'; aliasOnly?: true }
export const REBINDABLE: readonly RebindableAction[] = [
  { id: 'date', chord: 'F2', label: 'Change the date', does: 'the voucher date; the period on a report', group: 'dates' },
  { id: 'period', chord: 'Alt+F2', label: 'Change the period', does: 'the period of a report or the Gateway', group: 'dates' },
  { id: 'company', chord: 'F3', label: 'Select company', does: 'opens the company list from anywhere', group: 'dates' },
  { id: 'companyMenu', chord: 'Alt+F3', label: 'Company menu', does: 'opens the Company menu from anywhere', group: 'dates' },
  { id: 'features', chord: 'F11', label: 'Company features', does: 'opens the features list', group: 'dates' },
  { id: 'configure', chord: 'F12', label: 'Configure this screen', does: 'a report\'s options; the voucher configuration on an item voucher', group: 'dates' },
  // vouchers: one per VCH_BASES entry that has an fkey, id `vch:${base}`, `does` from BASE_LABELS (F5 also reads Detailed / Condensed on a
  // statement; F4 .. F9 on a menu open that voucher, on a voucher they switch its type); plus F10 (Other vouchers / Change Vch Type)
  { id: 'newLedger', chord: 'Alt+C', label: 'New ledger on the fly', does: 'creates a ledger from the field you are in', group: 'entry' },
  { id: 'delete', chord: 'Alt+D', label: 'Delete', does: 'deletes the voucher, the line or the level you are on', group: 'entry' },
  { id: 'print', chord: 'Alt+P', label: 'Print', does: 'prints the voucher or the report', group: 'entry' },
  { id: 'afterAccept', chord: 'Alt+N', label: 'After accept', does: 'after Accept on a voucher; New Snippet; Add Line on a budget; New Role or User', group: 'entry' },
  { id: 'holdField', chord: 'Alt+K', label: 'Hold this field', does: 'keeps this field for the next voucher', group: 'entry' },
  { id: 'duplicate', chord: 'Alt+2', label: 'Duplicate', does: 'copies the voucher being altered', group: 'entry' },
  { id: 'batch', chord: 'Alt+B', label: 'Batch', does: 'batch allocation on an item line; the basis of values on a statement', group: 'entry' },
  { id: 'goTo', chord: 'Alt+G', label: 'Go To', does: 'jump to anything', group: 'launcher', aliasOnly: true },
  { id: 'menuToggle', chord: 'Ctrl+\\', label: 'Hide or show the left menu', does: 'full-screen view', group: 'launcher', aliasOnly: true },
  { id: 'quickCreate', chord: 'Alt+Q', label: 'Quick create', does: 'the + menu', group: 'launcher', aliasOnly: true },
  { id: 'help', chord: 'F1', label: 'Key sheet', does: 'the list of keys', group: 'launcher', aliasOnly: true },
];
```
`core/chordRules.ts` (pure): `parseChord`, `targetProblem(chord, platform, desktop)`, `validateChordPrefs(prefs, ctx) -> { ok, table } | { ok: false, problems }`, `buildChordTable(prefs) -> { byPressed, inert, display }`, all taking the platform as a parameter so the suite runs both. `core/chordMap.ts` (leaf, no DOM): `setDisplayMap`, `displayChord`, `alsoChord`. The key table is installed on company open, operator change and every edit; **it is not installed without a company** (nothing personal is kept then).

## Appendix D. High contrast (neutrals only; measured on 13 surfaces)
| Token | Light | Dark |
|---|---|---|
| `--ink` | `#05080c` | `#ffffff` |
| `--ink-2` | `#1b2330` | `#f1f6f3` |
| `--dim` | `#2c3644` | `#e6eee9` |
| `--ghost` | `#3a4554` | `#dde6e1` |
| `--border` | `#6f7b8c` | `#8da198` |
| `--border-lt` | `#869299` | `#62746b` |
| `--field-line` | `#566274` | `#a2b4ab` |
| `--total-line` | `#05080c` | `#ffffff` |
| `--focus-ring` | `0 0 0 3px var(--focus)` | `0 0 0 3px var(--focus)` |

Worst case per text token across panel, panel2, raised, bg, sunk, sel, sel-strong, hover, tint, accent-soft over panel and panel2, field and bar: light `ink 16.20, ink-2 12.75, dim 9.87, ghost 7.85`; dark `ink 9.01, ink-2 8.25, dim 7.63, ghost 7.07` (all on `--sel-strong`); `--field-line` 6.19 / 5.82 (light on panel / panel2) and 7.98 / 8.14 (dark); `--border` 4.30 light, 6.35 dark; `--border-lt` 3.19 light, 3.50 dark. All >= 7 for text, >= 3 for boundaries.

## Appendix E. `index.html`, the first frame (Builder A extends the existing script)
```html
<script>
  try {
    var d = document.documentElement, L = localStorage;
    if (/[?&]uireset=1(&|$)/.test(location.search)) {                       // the way back from anything
      var gone = []; for (var i = 0; i < L.length; i++) { var k = L.key(i); if (/^TW_(UI_|THEME_)/.test(k)) gone.push(k); }
      gone.forEach(function (k) { L.removeItem(k); });
      history.replaceState(null, '', location.pathname + location.search.replace(/([?&])uireset=1(&|$)/, '$1').replace(/[?&]$/, '') + location.hash);
    }
    var f = {}; try { f = JSON.parse(L.getItem('TW_UI_FIRST') || '{}') || {}; } catch (e) {}
    var s = f.scheme || L.getItem('TW_THEME_SCHEME');
    if (s === 'system') s = matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
    d.setAttribute('data-scheme', s === 'dark' ? 'dark' : 'light');
    d.setAttribute('data-density', (f.density || L.getItem('TW_THEME_DENSITY')) === 'comfortable' ? 'comfortable' : 'compact');
    // each of the rest ONLY for a known non-default value (list mirrored from core/uiPrefs.ts; the suite compares the result with htmlAttrsFor)
  } catch (e) { /* storage blocked: theme.ts sets the built-in look once it runs */ }
</script>
```

## Appendix F. What this supersedes in `zoho-ux-spec.md` section 8
| There | Here |
|---|---|
| 8.1 `SCREEN_AREA 'admin'`, "nothing links to it" | `'open'`; reachable from the gear, `Ctrl+,`, Go To, the Gateway; the left-menu row at integration |
| 8.2 registry `configurable` flips, "never a voucher F-key" | the registry's `configurable` stays `false` for every Wave A chord; rebinding is a **separate table of actions** (`REBINDABLE`) the owner's request now includes the voucher keys in; defaults unchanged |
| 8.3 "hide or reorder only the optional second block" | hide, reorder and rename any button except Quit and Accept, as an **opt-in** layout; hiding never disables a key; defaults unchanged |
| 8.4 "Wave B moves Light / Compact into Settings" | the two toggles stay (frozen); the gear is added beside them |
| 8.4 palette not a picker; bundled fonts; size interacts with the compact row | as decided: eight presets; Plex, System, Ledger; tokens with a measured pitch |

## Appendix G. The cross-cutting fix round (2026-10-20): what changed after the reviews, and what it supersedes above

Four review lenses (keyboard safety and defaults identity, rights and sync, the visual matrix, brick-proofing and UX) confirmed the findings below; each is fixed at its root, with a failing-before assertion where one can be written (`uiprefs`, `uiscope`, `shortcutprefs`, `syncconflict`, `appearance`) and the harness probes in the report. Where this appendix and a section above disagree, this appendix wins.

| Topic | Before | Now |
|---|---|---|
| **Go To** (1.4) | The Settings entry carried about 30 aliases and ranked like any command, so `s`, `set`, `key`, `font`, `customize` and 28 more queries moved their first row. | The entry is a **late** command (the mechanism of the "All <label>" rows): it ranks behind every ordinary match and can never move one. The words still find it (`theme`, `appearance`, `preferences`, `shortcuts`...); `settings` still opens Company Features first. Golden: `scripts/shots/golden/goto.capture.mjs`, recorded from the commit before Settings existed, 103 queries, identical. |
| **Left menu** (1.4) | No row. | `settings.prefs` "Preferences", appended to the Settings group (the one group with seven rows). |
| **Who owns a personal choice** (3.2) | `<company>\|<operator>`: size, contrast, accent, font, keys and layouts vanished on switching company; the legacy light / dark, density and menu-view keys were device-wide and became every operator's choice under security. | `device\|<operator>` (the operator only while security is on). A table written the old way is **folded** into the device-wide owner when read (the owner already there wins, then the most recently written company). With security on an operator's choices are their own blob only: the legacy keys are not written and not read, the menu view is a blob field, and the workspace default reaches an operator who never chose. The table keeps the **newest** 64 owners (the oldest are dropped on save, so what is reported saved loads back). |
| **Reset** (4.8) | Left `TW_NAV_VIEW`. | `Reset all my settings` and `?uireset=1` clear it too, and the live menu follows without a reload. |
| **The company row** (3.4, 3.5) | The writer rebuilt the row from the copy this tab loaded and numbered the save from the row's revision. | **Read-modify-write of the STORED row** (`updateDoc`, one transaction), numbered **above everything this device has seen** (the row's and the remembered one), and a Reset or Restore on an `erased` / `older` device is written even if the row "has nothing to change". The decision is the pure `core/policyWrite.ts planPolicyWrite`. |
| **What is drawn** (3.5) | The rail, the top bar and the Shortcuts tab read the raw row, so an erased policy lost its rail layout and locks on every device at once. | They read the policy **this device applies** (`useEffectivePolicy`: the remembered one when the row was erased or is older), as the appearance locks already did. Only the administrator's forms read the raw row. |
| **The administrator's form** | An untouched form never adopted a policy saved elsewhere, and Save wrote the stale form over it. | The form keeps what it started from and folds in a change field by field (`rebaseDraft`); Save writes only what this person changed, over the freshest policy. |
| **Locks** (3.1) | Any appearance field. | **Text size, contrast and motion are defaults only** (`DEFAULTS_ONLY`): a lock would take away the one control a low-vision or motion-sensitive person has. A stored lock of them is kept as an unknown one and enforces nothing. |
| **Server** (3.4, risk 10) | `PROTECTED_FIELDS` untouched, `ui` erasable by an older client. | `UNVERSIONED_PROTECTED.companies = ['ui']` in `backend/src/conflict.js`: the current `ui` is kept against a write that states **no base** (an older app version), recorded as a `protected-fields` mismatch whose words say "workspace settings" (`kindTitleOf`, `onlyPolicyKept`). A current client from a stale base is an ordinary mismatch. **Needs a backend deploy** (`conflict.js`, `sync.js`). |
| **Shift** (4.1) | A turned-off Alt+K or Alt+B also killed Alt+Shift+K (release every hold) and Alt+Shift+B (the basis picker's reverse step), and a person's key plus Shift did nothing. | `SHIFT_AWARE`: those two chords keep their Shift variant live, and the person's key plus Shift is translated to it. The pins bar and the key sheet name the release chord the way the person spells it. |
| **Mac Option** (4.1) | A turned-off Alt+letter was swallowed in every box, which also killed the glyph Option+letter types. | Dead as a shortcut, still typed as a symbol (a press whose `key` is not the letter on the key is not prevented). |
| **Reserved** (4.3) | Ctrl+Y was offered; on a Mac so were Command+D, O and Y, which also take Control+D, O and Y (delete forward, open line, yank). | Ctrl+Y is reserved (Redo); the three are refused on a Mac (`MAC_FIELD_EDITING`). |
| **Names** (4.6) | Zero-width, bidirectional and filler characters were kept. | `cleanLabel` removes every Unicode format character, C1 control and blank filler before the length and "same as the original" checks, and a name needs a letter or a digit. |
| **Unknown settings** (3.3) | Carried at any size. | At most 16 KB in all, four levels deep; the remembered copy is the sanitised policy, so an oversized row cannot switch the safety net off. |
| **Fit** (2.5) | The keys table collapsed between 110 % and the window width; below 1100px it hid the key columns; the Go To pill shrank to nothing; a clipped top-bar item was left half drawn; the preview, a report header and a dialog's placeholders were cut. | The keys table and the two button lists are **containers** (never the whole tab: a container traps fixed overlays) and a row stacks when it has no room; the pill has a floor; a clipped item is not drawn; the others wrap, scroll or grow with the text. Measured over 1440 / 1366 / 1280 / 1024 x 100 / 110 / 125 / 150 % x three faces x two densities, light and a dark subset: no cut figure, no half item, no spill. |
| **Keyboard and pointer** | A row of the button editors held a Tab stop per name box and button (150 presses to the way back); the recorder had no buttons. | One Tab stop a row (16 presses to "Reset all my settings"); the recorder has Use, Remove and Cancel and closes on a click behind it. |

**Accepted, not fixed:** high contrast does not deepen accent-filled buttons (5.8 to 6.4:1 white on the accent in light; AA, not 7:1; spec 2.6 says neutrals only); on a voucher the rail is taller than its column at 125 % and above, so it scrolls (the keys work, and Esc and Accept are where they were); the first frame (`TW_UI_FIRST`) is device-wide, so for an instant before the company opens the sign-in gate can show the last operator's look; the sync-time `ui` protection exists only once the backend is deployed.
