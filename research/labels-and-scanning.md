# Label printing and barcode scanning for an Indian accounting and inventory app: standards, printers, rules, scanners, and what this repository needs

> **Status:** research and judgement only. Nothing here is built and no source file was changed. Written 2026-10-23 against the main tree
> (`/Users/Akshay/tallyreplica`, HEAD `7074445`) and, read-only, the worktree `.wt-cf` (custom-fields spec with its addendum of 2026-10-22, and the Back-control spec).
> **The owner's request (intent):** "batch tracking of Tally is weak; Busy's customers appreciate its batch management; mobile shops track IMEI; label printing and
> scanning are required and Tally is weak there too; build these well." This document is the **label printing and scanning** half. Batch and serial design is in
> `docs/research/batch-serial-tracking.md`, `batch-serial-code-audit.md` and `tracking-verticals-india.md`; this document does not repeat them and says where it adds to them.
> **Evidence tags.** **[W key]** a page I fetched and read (the key points at the source list in section 9). **[Ws key]** a search summary only; the page itself was not read.
> **[C file:line]** read in this repository. **[M]** computed or measured by me in a throw-away script (the scripts lived in the session scratchpad and are not repo code).
> **[J]** judgement. **[unverified]** I could not confirm it. Every external fact is paraphrased; nothing is quoted at more than a few words. Statutory facts carry the weakest tag I could
> honestly give them.
> **What I could not do (the full list is section 8):** no printer, no scanner and no phone were available, so every statement about how hardware behaves is from a manual, a vendor page or a
> forum summary, never from a test. I have no live Busy, Marg, Vyapar or TallyPrime. GS1's own specification pages answered 403 or were too large to fetch, so GS1 facts come from GS1 India,
> GS1 national offices, the GS1 application-identifier dataset and independent libraries' documentation. I did not run the app (the dev server on :5173 is not mine), so the behaviour of a
> scanner burst in the voucher screen is read from code, not measured.
> **Contents:** 0 Read this first; 1 Symbologies and standards; 2 Label media, printers, printer languages and print paths; 3 Label content and the Indian rules; 4 The label design experience; 5 Barcode scanning; 6 The repository today; 7 Recommendations; 8 What I could not verify; 9 Sources.

---

## 0. Read this first

1. **Labels are buildable from parts this repository already has, and the hard part is the printer, not the barcode.** A pure QR encoder with an independent decoder test exists
   [C `src/core/qr.ts:1-12`], a millimetre-exact custom page precedent exists (the cheque leaf) [C `src/ui/banking/chequeLeaf.tsx:169-173`], and a bridge pattern for untrusted-renderer output exists
   [C `electron/exportBridge.cjs:1-20`]. What is missing is a 1-D encoder, a label layout engine, a label-size page path, and any way to choose a printer or print silently [C greps in section 6].
2. **A 1-D barcode encoder is small, and I proved it can be checked.** I wrote throw-away EAN-13 and Code 128 encoders (about 60 lines) and the macOS Vision framework, an independent decoder, read
   seven codes correctly at both 4 and 2 pixels per module, including a 15-digit IMEI in Code 128 and a weighing-scale style EAN-13 [M, section 1.9]. The repository's own habit (a pure `src/core`
   file asserted against an independent decoder, as `qr.ts` is) fits. A dependency does not look necessary for EAN, UPC, Code 128, Code 39 or QR (two of those families were tested, the rest are the same kind of work [J]); it is needed, or a few hundred written lines, only for DataMatrix.
3. **Three symbology facts decide most of the design.** (a) A shop's own labels do not need GS1 membership, but must never use the 890 block (only GS1 India issues it) [W gs1india-home]; the
   restricted ranges (020-029, 040-049, 200-299) exist for in-store use [W wiki-gs1-codes]. (b) A GS1 DataMatrix or GS1 QR carries batch, expiry and serial in one scan using application identifiers
   10, 17 and 21, and variable-length fields are cut by the ASCII group separator, which **a keyboard cannot type** [W gs1-ai; Ws gs1-dm-fnc1; W cleverence-gs]. (c) India's top-300 drug brands carry a barcode or
   QR with batch, dates and licence number since 1 August 2023, but the rule does **not** require GS1 and leaves the product code to the maker [W cdsco-faq; W drugs-rules, rule 96(6)-(7)].
4. **On the printer side, "driver or raw" is the real fork, and the app can do both in Electron and only the first in a browser.** A browser reaches a label printer only through the print dialog and
   its driver (exact `@page` size, margins zero, scale 100%, then whatever the driver does) [W mdn-page-size; Ws chrome-label-print]. Electron 31 adds silent print to a named printer with a custom page size
   [W electron-v31-webcontents], and a main-process raw path (TCP port 9100, or the Windows spooler's RAW data type [W ms-rawprint]) that sends the printer's own language (TSPL for TSC, the one language I read in full; ZPL for Zebra; whether the TVS and Godex models speak either is unconfirmed) [W tspl; W zebra-zd220]. Chromium browsers can reach a printer over WebUSB, Web Serial or Web Bluetooth, but only on HTTPS, and Safari and Firefox cannot
   (Firefox has Web Serial only, by the caniuse page I read) [W caniuse-usb, caniuse-serial, caniuse-bluetooth].
5. **The most robust label output is a 1-bit bitmap rendered by the app at the printer's own resolution and sent as graphics**, because it prints the rupee sign and Indian-script text that the printers'
   built-in fonts and code pages do not offer (the TSPL code-page table has no Indian script [W tspl]; Zebra's ZPL printers advertise Unicode and open-type fonts but still need a suitable font loaded [W zebra-zd220]), and because it makes the preview and the sticker the same pixels [J]. Native printer barcodes
   are the alternative for crisp 1-D bars; both are described in sections 2.3 and 2.4.
6. **The legal line for labels is simple to state and easy to cross.** Pack declarations (MRP inclusive of all taxes, net quantity, the maker's, packer's or importer's name and address, month and year,
   consumer-care details) are the **packer's** duty, and a trader who repacks goods is a packer [W lm-book, rules 2(g), 4(1), 6]. A shop's own price or barcode sticker is permitted for other
   declarations provided it does not cover the maker's, and a lower-MRP sticker may not cover the original [W lm-book, rule 6(3)-(4)]; a sale above the printed MRP is barred and nobody may obliterate or alter
   it [W lm-book, rule 18(2), 18(5)]. So the app prints shop labels freely and offers a declaration template only for a user who is a packer or importer, never claims compliance, and never prints
   a number a regulator issues (a HUID is generated by BIS's server and laser-marked at a hallmarking centre [W bis-blog]).
7. **Scanning mostly works today by accident, and fails silently where it matters.** A keyboard-wedge scanner types the code and an Enter; in the voucher item field a code stored in the item's
   alias or part number selects the item [C `VoucherScreen.tsx:2862-2867`, `:3223`]. An unknown code does nothing and the next scan appends to it; a burst that lands in Narration ends in Accept
   [C `:3367`]; on a menu or report a burst is a string of hotkeys [C `keys.ts:11-34`]. The fix is not a detector first: it is a **scan target model** (only a field that wants a scan takes one; anywhere else a
   burst is swallowed and reported), an exact-lookup resolver, and an unknown-code message. A timing detector is the second step and must be opt-in [J, Phase 27 rule in CLAUDE.md].
8. **Wedge scanners are layout-dependent and the app can defend itself.** The scanner sends key positions, the operating system turns them into characters through whatever layout is active, and Indian shops
   switch to Hindi or another script layout often [J, unverified]. Decoding from `KeyboardEvent.code` (the physical key) plus the Shift state, with a setup screen that shows what arrived, removes the dependence on the
   OS layout for the usual US-configured scanner; this is a design proposal, not something I could test [J]. Two scanner modes break a keyboard-first app outright: Alt-plus-numeric-keypad "keypad emulation"
   (the app's Alt chords would fire) and a group-separator sent as Ctrl+] [W zebra-keypad; W gs1parser; W cleverence-gs].
9. **Camera scanning needs HTTPS, and the live site is HTTP only** [C `docs/HANDOFF.md:63`, `:90`]. `BarcodeDetector` has limited availability (partial in Chromium, off by default in Safari, absent in Firefox) [W caniuse-barcode; W mdn-barcode];
   a WebAssembly decoder fills the gap (about 0.94 MB for the reader build of zxing-wasm [W jsdelivr-zxingwasm]). A handheld wedge scanner beats a camera for a counter doing 300 bills a day; the camera is for
   a phone doing a stock-take or a lookup [J].
10. **Recommended order** (section 7): fixed label templates with pure encoders and the browser print path first (works on web and Electron, no hardware code); the scan-target model and the unknown-code message
    next (no hardware code either); then the Electron printer bridge (list, silent print, raw TCP and spooler); then GS1 and DataMatrix; camera and mobile last, after TLS.

---

## 1. Symbologies and standards

### 1.1 The families, and which one a shop should reach for

| Symbology | What it holds | Size facts | Where it appears in Indian trade | Check | Source |
|---|---|---|---|---|---|
| **EAN-13** (GTIN-13), **UPC-A**, **EAN-8** | 13, 12 or 8 digits (the last is a check digit). UPC-A is an EAN-13 with a leading zero | EAN-13 is 95 modules: start guard 3, left six digits 42, centre guard 5, right six digits 42, end guard 3; the first digit is not drawn but chosen by the pattern of the left six | every retail pack; weighing-scale labels; books (ISBN-13 is an EAN-13 beginning 978 or 979) | mod 10, weights 3 and 1 from the right | [W wiki-ean; W wiki-gs1-codes] |
| **EAN/UPC add-on** (2 and 5 digits) | a periodical's issue number, or a suggested price | drawn beside the main symbol | books and magazines | add-on's own | [W wiki-ean]; printers take them as `EAN13+2` and `EAN13+5` [W tspl] |
| **Code 128** | all 128 ASCII characters; Set C packs two digits per symbol | 11 modules per symbol, the stop is 13; quiet zone at least 10 modules each side | shop item and batch codes; IMEI and serial numbers; GS1-128 is Code 128 with FNC1 first | mod 103, weighted by position (always present) | [W wiki-c128] |
| **Code 39** | 43 characters plus `*` as start and stop | low density (wide) | older systems, industrial | none built in; optional mod 43 | [W wiki-c39] |
| **ITF-14** | 14 digits (a GTIN-14) | bearer bars | outer cartons | GTIN check | [W tspl lists ITF14, EAN14] |
| **QR Code** (ISO/IEC 18004) | up to 7,089 digits, 4,296 alphanumerics or 2,953 bytes; 21 to 177 modules a side | error correction L about 7%, M 15%, Q 25%, H 30% | UPI links, e-invoice QR, URLs, phone-scannable labels | Reed-Solomon | [W denso-qr]; Denso Wave has waived its patent rights for standardised codes [W denso-qr] |
| **Data Matrix** (ECC 200, ISO/IEC 16022) | up to 3,116 digits or 2,335 alphanumerics; 10x10 to 144x144 square, 8x18 to 16x48 rectangular | small and dense; a 14x14 symbol holds 16 digits (so a 15-digit IMEI fits) | small parts, electronics, pharma (GS1 DataMatrix) | Reed-Solomon, recovers up to about 30% damage | [W wiki-dm]; the 14x14 capacity from the GS1 DataMatrix guideline's table [Ws gs1-dm-table] |
| **GS1-128**, **GS1 DataMatrix**, **GS1 QR** | a string of GS1 application identifiers (section 1.3) | as the carrier | cartons, pharma packs, some FMCG | per element | [W gs1-ai; W biip-sym] |
| **GS1 Digital Link QR** | a web address that embeds GTIN, batch and serial (`/01/<gtin>/10/<batch>/21/<serial>`) | a normal QR | brands moving to consumer-facing 2-D codes (a retail "2-D at point of sale" deadline of 2027 is claimed by QR vendors) | none | [Ws gs1-digital-link]; GS1 India lists Digital Link and DataMatrix among the standards it supports [W gs1india-home] |

**Judgement [J].** The "where it appears" column is my reading of the trade unless a source is named. For a shop's own item, batch and unit labels: **Code 128 (or EAN-13 where a GTIN exists) plus a QR or DataMatrix for compact units**. Code 39 and ITF are only to be *read*, never
the first thing the app prints. GS1 symbologies matter far more for **reading** (pharma strips, cartons, imports) than for printing in Indian trade, because only manufacturers and importers print them.

### 1.2 GS1, GS1 India and the 890 prefix

- **890 is India's prefix in GS1's table**; GS1 India says it is the only body in India authorised to issue barcodes beginning 890 and warns about unauthorised resellers (it mentions a Delhi court order
  against one) [W gs1india-home; W wiki-gs1-codes]. GS1 India's blogs say every product made in India carries 890; GS1 Global's own page, read only through a search summary, says the prefix names the
  member organisation that issued the number and **does not prove where the product was made** [W gs1india-ean13; Ws gs1-prefix-search]. The app must not use the prefix to decide "made in India".
- **A GTIN is company prefix plus an item reference plus a check digit**; the company prefix is variable length, so a short prefix leaves more digits for items; in India item references are managed
  through GS1 India's DataKart product repository [W gs1india-ean13; W gs1india-home]. A registration yields an allotment letter with the prefix in about 7 to 10 working days according to GS1 India's guidance, as a search summary
  gave it [Ws gs1-prefix-search]; fees were not on the pages I read [W gs1india-home].
- **Ranges for in-store use.** 020-029 and 200-299 are restricted circulation numbers whose meaning is set by the local GS1 office (the variable-weight and price-embedded labels of section 1.5); 040-049 is for use
  inside one company [W wiki-gs1-codes]. A library's reference gives the same split for 13-digit codes (02 and 20-29 geographic, 04 company-only) [W biip-rcn].
- **What this means for the app [J].** (1) A shop that has no GS1 membership numbers its own stock in the **040-049** block (company use), and the app never generates an 890 number for it. (2) A *manufacturer*
  who holds a prefix can use a **GTIN builder** (prefix, item reference, computed check digit) and a cheque against collisions; that is a small, safe feature. (3) Two shops that sell to each other should
  not rely on 04x codes meaning the same thing in both books; the registry keeps the shop's code and the supplier's GTIN as separate entries (the barcode registry of `tracking-verticals-india.md` section 6.5).
  (4) Reading is unaffected: any EAN or UPC from any country scans the same way.

### 1.3 Application identifiers, and why a keyboard scanner breaks them

The identifiers a pharma, FMCG or electronics pack uses, as read from GS1's machine-readable dataset (the first 100,000 of 546,468 characters, which covered every row below) [W gs1-ai]:

| AI | Meaning | Shape | Needs a separator when not last? |
|---|---|---|---|
| 00 | SSCC (a logistic unit) | 18 digits | no |
| **01** | **GTIN** | 14 digits (an EAN-13 becomes a GTIN-14 by a leading 0; the check digit is unchanged [M]) | no |
| **10** | **Batch or lot** | 1 to 20 characters from a printable set (letters, digits and a set of punctuation) | **yes** |
| 11 | Production date | YYMMDD | no |
| 15 | Best before | YYMMDD | no |
| **17** | **Use by or expiry** | YYMMDD | no |
| **21** | **Serial number** | 1 to 20 characters | **yes** |
| 240 | Additional item identification | 1 to 30 characters | **yes** |
| 310n | Net weight in kg | 6 digits; the fourth digit n (0 to 5) is the number of decimals | no |

(A price and a count have identifiers too; the dataset reached only part of its length, so I give none I did not read. The day value in the date pattern accepts `00`; what GS1 intends `00` to mean I could not confirm
[unverified].)

- **How a string is cut.** An element ends at the end of the data, at the fixed length its identifier defines, or at the group separator (ASCII 29, "GS", the FNC1 character); a parser therefore needs the whole length
  table and must reject an unknown identifier rather than guess [W gs1parser]. Putting the variable-length fields last avoids most separators [J].
- **FNC1 has two jobs.** In first position it is only a flag that says "this is GS1 data": a scanner that is configured to report it prefixes the symbology identifier (`]d2` for GS1 DataMatrix, `]C1` for GS1-128) and does
  **not** send a GS. An FNC1 in a later position is sent as GS [Ws gs1-dm-fnc1; W biip-sym].
- **A keyboard cannot type GS.** A keyboard-emulating scanner substitutes a control sequence (canonically Ctrl+] ) or drops the character, and some makers replace it with Tab or a space without saying so
  [W gs1parser; W cleverence-gs; Ws gs1-dm-fnc1]. Serial (virtual COM) mode passes the byte through, and scanners have a setting to send it as ASCII 29 [W cleverence-gs]. So a GS1 reader in this app has to
  accept **three** spellings: the real GS character, the key chord, and the *absence* of a separator (cut by length, which fails for a variable field followed by another), and must say which it assumed [J].
- **Worked example [M].** An EAN-13 `8901234567890` as a GTIN-14 is `08901234567890`. A pack with expiry 31 March 2027 and batch `B2410A` is the element string `0108901234567890` `17270331` `10B2410A`; because the
  batch comes last no separator is needed. Adding a serial after the batch needs a GS between them: `10B2410A<GS>21<serial>`. (The numbers are invented; `890123456789` is not an allotted prefix.)
- **Expiry on Indian medicines is a month and year that runs to the last day of the month** [W drugs-rules, rule 96 explanation], so a reader that fills an expiry date from AI 17 and a writer that builds AI 17 from a
  month-and-year batch expiry should both use the **last day of the month** [J].

### 1.4 Check-digit rules, in one place

| Code | Rule | Worked example | Source |
|---|---|---|---|
| EAN-13, UPC-A, EAN-8, GTIN-14, SSCC | multiply the data digits by 3 and 1 alternately starting with 3 at the right-most data digit; the check digit brings the sum to a multiple of 10 | `890123456789` gives 0, so `8901234567890`; `400638133393` gives 1: the worked example in Wikipedia's article, `4006381333931`, whose weighted sum of 89 gives check digit 1 | [W wiki-ean]; [M] |
| IMEI | Luhn over 15 digits (8-digit type code, 6-digit serial, 1 check digit); consecutive IMEIs do **not** differ by one | `490154203237518` passes | `batch-serial-tracking.md` section 3.1 [R]; [M] |
| Code 128 | start value plus each symbol's value times its position, modulo 103; always present | Set C `123456` is start 105, then 12, 34, 56, check 44, stop 106 | [W wiki-c128]; [M] |
| Code 39 | none; optional sum of character values modulo 43 | | [W wiki-c39] |
| Scale label | the EAN-13 check over the full 12 digits, recomputed whenever the weight or price changes (Odoo's example recalculates it); some national schemes add a separate price check digit | `2100123012503` (item 00123, 1.250 kg) | [W odoo-nomenclature; W gs1se-rcn]; [M] |

One validator per family, never one shared "check digit" function: using the EAN rule on an IMEI, or the reverse, passes or fails the wrong codes (the sibling research makes the same point).
On a **scan**, a bad EAN check digit is a hard reject (the scanner would not have sent it); on a **typed** alias it is a soft warning, because today's aliases are free text [J].

### 1.5 Weighing-scale and price-embedded labels (restricted circulation numbers)

- **What the standard fixes and what it leaves open.** GS1 reserves the prefixes (02 and 20-29 for a region) and says the local GS1 office defines the structure; the same libraries that decode them keep a **per-region
  profile** and an option to switch the embedded check digit off because some firms use it for something else [W biip-rcn; W gs1se-rcn].
- **Two real layouts that disagree.** Sweden: a prefix starting 2 whose second digit says what the value is (0 to 2: price in kronor with two, one or no decimals; 3 to 5: weight in kg with three, two or one
  decimals), then a 2 to 3 digit item number, the value, and a final check digit [W gs1se-rcn]. Odoo's default weight rule: prefix 21, five item digits, five weight digits (two whole, three decimal), then the check digit, and the
  item's own barcode holds `00000` where the weight goes [W odoo-nomenclature]. **No Indian scale maker's layout was found** (Digi and Essae manuals did not surface) [Ws scale-search; unverified].
- **Design [J].** A **scale profile** per shop (or per scale): prefix digits, item-digit count, value-digit count, what the value is (weight in kg with n decimals, price in rupees or paise, a count), and whether the final digit is the
  EAN check. The setup is "scan one printed label, type what it weighs, the app shows the split and asks you to confirm", with the Odoo and Swedish layouts as two built-in test vectors. A label whose item digits match no
  item is "unknown", never a guess. The item stores its **scale code** (the digits the scale prints), a field of the barcode registry.
- **Legal angle.** A retailer who weighs goods for a customer in front of them is not packing; a retailer who pre-packs loose goods is a packer with the declarations of section 3 [W lm-book, rules 2(g), 4(1); contested edges in
  the closing note of section 3.1].

### 1.6 Symbology identifiers (what a scanner may put in front of the data)

A scanner can be set to prefix each read with a three-character identifier that names the symbology, so software need not guess [W biip-sym; Ws gs1-dm-fnc1]:

| Identifier | Meaning | Implies GS1 data |
|---|---|---|
| `]E0` | EAN-13, UPC-A or UPC-E | yes |
| `]E4` | EAN-8 | yes |
| `]C1` | GS1-128 | yes |
| `]d2` | GS1 DataMatrix | yes |
| `]Q3` | GS1 QR Code | yes |
| `]d1` and other modifier digits of `]C`, `]d`, `]Q` | the same carriers without the GS1 flag (a source gives `]d1` for plain Data Matrix) [Ws gs1-dm-fnc1] | no |
| `]A` plus a digit | Code 39 | no |

The reader must accept the same code **with and without** the prefix (a shop will not switch the setting on), and treat a prefix as authoritative when present (a `]d2` or `]C1` read is GS1 by declaration, a bare 13-digit
string is not) [J]. The Zebra programming reference lists the controlling parameters under names such as "transmit code ID", prefix and suffix values, "Enter key" and "Tab key" [W zebra-ds4678-nav].

### 1.7 Sizes, dots and what fits on a label

GS1's size guide for a consumer-item EAN-13: module (X-dimension) minimum 0.264 mm, recommended 0.330 mm, maximum 0.660 mm; bar height minimum 18.28 mm, recommended 22.85 mm; quiet zone 11 modules on the left and
7 on the right (113 modules in all with the quiet zones) [W gs1se-size; Ws gs1-size-search]. A 203 dpi printer has 8 dots per millimetre and a 300 dpi printer about 11.8 (the TSC manual rounds to 12; 600 dpi is 24) [W tspl];
Zebra's own specification gives the nominal dot as 0.125 mm at 203 dpi and 0.084 by 0.099 mm at 300 dpi [Ws zebra-print-specs]. Bars should be a **whole number of dots** (a forum answer puts the steps at about 5 mil at 203 dpi) or the printer
rounds them silently [Ws idautomation-dots].

Widths I computed for common cases (module counts include quiet zones) [M]:

| Code | Modules | 2 dots at 203 dpi (X 0.250 mm) | 3 dots at 203 dpi (X 0.375 mm) | 4 dots at 300 dpi (X 0.339 mm) |
|---|---|---|---|---|
| EAN-13 | 113 | 28.3 mm | 42.4 mm | 38.3 mm |
| Code 128, 15-digit IMEI (Set C pairs, one switch to Set B for the last digit) | 154 | 38.5 mm | 57.8 mm | 52.2 mm |
| Code 128, 12 digits in Set C | 121 | 30.3 mm | 45.4 mm | 41.0 mm |
| Code 128, 12 mixed characters (Set B) | 187 | 46.8 mm | 70.1 mm | 63.3 mm |
| QR version 1 (21 modules plus a 4-module quiet zone each side = 29) | 29 | 7.3 mm | 10.9 mm | 9.8 mm |
| Data Matrix 14x14 (plus a 1-module quiet zone each side = 16) | 16 | 4.0 mm | 6.0 mm | 5.4 mm |

Reading it [M, J]: **a 15-digit IMEI in Code 128 does not fit a 38 mm label at 2 dots (38.5 mm) and fits a 50 mm label; the same IMEI as a Data Matrix is about 4 to 6 mm square.** A retail EAN-13 at two
dots (0.25 mm) is a little below GS1's own minimum module (0.264 mm), which matters for a pack that must scan at a stranger's till and not for a shelf label scanned by the shop's own scanner. The 25 mm tall label cannot carry GS1's 18 mm minimum bar
height with a name and price; shelf labels use bars of 8 to 12 mm, which hand scanners read but GS1 does not bless [J]. A 3-up roll of 38 mm labels needs about 114 mm of media, more than the 104 mm print width of the desktop
4-inch printers I read about (Zebra ZD220 104 mm [W zebra-zd220]; Godex EZ2250i 104 mm and TVS LP 46 Neo 104 to 108 mm [Ws godex-ez2250i; Ws tvs-lp46]); a layout engine must know **the printer's print width, the number across and the gap**, not only the label size.

### 1.8 Generating codes in the browser and in Electron

| Option | What it is | Size and licence | SVG | Accuracy evidence | Fit for this repository |
|---|---|---|---|---|---|
| **Pure TypeScript in `src/core`** (the repo's habit: `qr.ts`, `xlsx.ts`, `zip.ts`, `inflate.ts`) | EAN-8/13, UPC-A/E, Code 128 (sets A, B, C with switching), Code 39, ITF as small functions that return a string of module widths; QR already exists | a few hundred lines in all; no licence question; no dependency (`package.json` has none for barcodes or PDF [C `package.json`]) | yes: a module array to one SVG path, as `qrSvgPath` and `QrSvg` do [C `src/ui/print/QrCode.tsx:1-33`] | write an **independent** decoder or use one; see 1.9 | **Best fit.** Deterministic, asserted, and the same module array feeds a bitmap or a printer command |
| **JsBarcode** | 1-D generator | MIT; all formats about 60.8 KB minified (60,817 bytes) [W jsbarcode; W jsdelivr-jsbarcode] | yes, plus canvas | no GS1-128, DataMatrix or QR [W jsbarcode]; "widely used" is my impression [J] | acceptable if a dependency is wanted for 1-D only; but the two 1-D encoders I wrote were about 60 lines [M] |
| **bwip-js** | a JavaScript port of the Barcode Writer in Pure PostScript (BWIPP): about 111 types including GS1-128, GS1 DataMatrix, GS1 QR, rectangular DM, Digital Link variants | MIT [W bwip-js-license]; version 4.11.4 dated 2026-08-19; the minified browser build is about 1.11 MB (1,112,458 bytes), the file listing shows an ESM file of about 198 KB and a separate 2.1 MB core module (`bwipp.mjs`); how a bundler combines them I did not check [W bwip-js; W jsdelivr-bwipjs] | yes (`toSVG()` is synchronous) | the reference most others are checked against | **Heavy for runtime; excellent as a test oracle** (a devDependency that encodes the same input and is compared module for module), or as a lazily loaded chunk if DataMatrix and GS1 are wanted before they are written |
| **node-qrcode** | QR generator | MIT; levels L, M, Q, H; mixed numeric, alphanumeric and byte segments chosen automatically; browser bundle supports evergreen browsers [W node-qrcode] | yes | | not needed: `qr.ts` exists. Its limits: byte mode only, levels L and M [C `src/core/qr.ts:12`, `:30-44`], no GS1 mode indicator, so a GS1 QR (and mixed-mode compaction) would need extending it; a **Digital Link QR is just a URL** and works today |
| **Printer-native barcodes** | the printer draws the code from a command (TSPL `BARCODE`, `QRCODE`, `DMATRIX`; ZPL `^BC`, `^BE`, `^BQ`, `^BX`) | no code in the app | n/a | the printer's firmware; bars are whole dots by construction | Electron raw path only; the preview still needs the app's own drawing, so the two can disagree and a test vector per symbology is needed |

**Judgement [J].** Write the 1-D encoders (small, checkable, no dependency), keep `qr.ts` and extend it to level Q and numeric/alphanumeric segments only if a label needs them, and **decide DataMatrix when the GS1
slice starts**: it is a few hundred lines of Reed-Solomon over GF(256), the placement pattern and C40/ASCII/digit-pair encodation, with FNC1 as codeword 232 for GS1 data (TSPL shows the printer side: `~1` is FNC1, `~]` is GS
[W tspl]). If it is not written, load bwip-js lazily for that one screen. Either way bwip-js belongs in the **test** dependencies as the oracle.

### 1.9 What I actually measured

A throw-away script in the session scratchpad (not repo code) encoded EAN-13 and Code 128 (a simple optimiser: Set C for digit runs of four or more, Set B otherwise, with the switch codes) into module strings; a second
throw-away Swift program drew them as pixel-exact bars with an 11-module quiet zone and asked **macOS Vision's barcode detector** to read them [M]:

| Input | Encoded as | Read back at 4 px per module | Read back at 2 px per module |
|---|---|---|---|
| EAN-13 `8901234567890` (an invented number) | 95 modules | exact | exact |
| EAN-13 `2100123012503` (a scale-style label) | 95 modules | exact | exact |
| EAN-13 `4006381333931` (Wikipedia's worked example) | 95 modules | exact | exact |
| Code 128 `123456` | values 105, 12, 34, 56, 44, 106; 68 modules | exact | exact |
| Code 128 `490154203237518` (an IMEI) | values 105, 49, 1, 54, 20, 32, 37, 51, 100, 24, 93, 106; 134 modules | exact | exact |
| Code 128 `MOB-SAM-A15-128GB-BLK` | 266 modules | exact | exact |
| Code 128 `B2410A-17` | 134 modules | exact | exact |

What this shows: the encoders are short; the Code 128 pattern rows I wrote from memory are right **for the 29 symbol values these seven codes happen to use** (an independent decoder rejects a wrong row, but the other 78 rows were not exercised, so a build must encode and decode every one of the 107 values); and **a decoder exists on a developer's Mac for every symbology a label needs**
(Vision reads EAN, Code 128, Code 39, QR, Data Matrix and more [unverified for the full list; the run covered two families]). What it does **not** show: anything about a printer, ribbon, a real scan at distance,
or a hand scanner's tolerance. The repository's test harness already uses this pattern for QR (an independently written decoder, and Apple's QR detector when `qr.ts` was written [C `src/core/qr.ts:1-12`]).

---

## 2. Label media, printers, printer languages and the print paths

### 2.1 Media

**A4 sheet labels (laser or inkjet).** Indian shops and the apps they use work with pre-cut A4 sheets: Vyapar's guide names 24-up and 40-up self-adhesive sheets [W vyapar-barcode]; Odoo ships PDF grids (2 x 7 with price,
4 x 7, 4 x 12) [W odoo-labels]; Zoho's template editor takes a sheet size or custom dimensions, page margins, rows and columns and label margins [W zoho-labels]. The grids I could confirm from retail listings are European
A4 standards, and I found no Indian-supplier list [Ws avery-grids; Ws indian-sheets]:

| Grid (per A4 sheet) | Label size | Source |
|---|---|---|
| 21-up (3 by 7) | 63.5 x 38.1 mm | Avery L7160 listings [Ws avery-grids] |
| 24-up (3 by 8) | about 63.5 x 33.9 mm | Avery L7159 listings (a retailer rounds it to 63 x 34) [Ws avery-grids] |
| 40-up (4 by 10) | 45.7 x 25.4 mm (a catalogue citing Avery L7654), or 48.5 x 25.4 mm (a German listing) | [Ws indian-sheets] |
| 44-up | about 48 x 25.4 mm | [Ws indian-sheets] |
| 65-up (5 by 13) | 38.1 x 21.2 mm | Avery L7651 listings [Ws avery-grids] |

So the sheet screen takes **free grid parameters** (page, top and left margin, columns, rows, label width and height, column and row gap, the start cell for a part-used sheet) with those rows as presets, not a fixed list. A desktop printer cannot print to the
sheet edge; the unprintable border is a property of the printer that I did not find a figure for [unverified], so the first test print on a real sheet measures it (the cheque leaf's offset calibration is the precedent, section 6).

**Thermal rolls.** Sizes named by an Indian-fashion vendor blog (a vendor, so a pointer rather than market data): 25 x 15 mm (small accessories, jewellery), 40 x 30 mm (jewellery tags), 50 x 25 mm (clothing, including sarees and lehengas),
75 x 50 mm (shoes, outerwear), 100 x 150 mm (shipping) [W commmerce-sizes]. The owner's list (50 x 25, 38 x 25, 100 x 50) adds 38 x 25 and 100 x 50 [J, unverified for market share]. Rolls come one, two or three labels across; the
horizontal width of that stock decides whether a printer's print head can use it (section 1.7). Three facts every vendor page repeats: the printer's label size must match the template, the roll must be loaded right, and a printer
that skips labels needs **calibrating** [W busy-faq5; W busy-labels; W zoho-labels]. TSPL has `SIZE`, `GAP` (gap sensing) and `BLINE` (black-mark sensing) for this [W tspl]. Direct-thermal paper fades with heat and time and
transfer printing uses a ribbon; the LP 46 Neo lists both methods [Ws tvs-lp46]. A durable sticker (a phone's warranty, a pharmacy shelf) is therefore a thermal-transfer job [J, unverified].

### 2.2 Printers common in India, and what I could confirm

| Printer | Language | Resolution and width | Notes | Source |
|---|---|---|---|---|
| **TSC TE244** and **TTP-244 Pro** | TSC's TSPL family; the 2025 TSC manual covers the TSC barcode-printer series; **per-model support varies**: it lists the TDP-643 Plus, TTP-243, TTP-342, TTP-244ME, TTP-342M and TTP-248M as lacking the `QRCODE` command, and some of them as lacking ITF-14, EAN-14 and MSI | 203 dpi, 6 inches a second | Indian listings show roughly 9,850 to 13,899 rupees (prices vary by seller and date) | [W tspl; Ws tse244-listings] |
| **TVS LP 46 Neo** | **not confirmed**: no listing or brochure I could reach states TSPL, ZPL or EPL emulation | 203 dpi, 150 mm/s, print width 104 to 108 mm; USB and serial; direct thermal and transfer; 300 m ribbon | an Indian maker; check its programmer's manual and driver before promising raw support | [Ws tvs-lp46] |
| **Zebra ZD220, ZD230** | ZPL II, EPL 2, XML (Zebra's own sheet for the ZD220); ZD230 adds Link-OS in reseller listings | 203 dpi; media 25.4 to 112 mm; print width 104 mm; USB | the standard "replace a legacy printer" model; a vendor blog names the ZD421 and TSC TE200 for fashion chains | [W zebra-zd220; Ws zd230; W commmerce-sizes] |
| **Godex EZ-2250i** (and the G500) | EZPL, GEPL, GZPL with automatic switching (EZ-2250i); the G500's languages **not confirmed** | 203 dpi (EZ-2350i is 300), print width 104 mm; Ethernet, USB, RS-232, USB host | semi-industrial | [Ws godex-ez2250i] |
| **Brother QL series** (QL-800, QL-820NWB) | its own raster command set, ESC/P and P-touch templates; **b-PAC SDK is for Windows**, with iOS and Android SDKs | die-cut and continuous tape up to 62 mm for the QL-8xx line [unverified] | useful for small shelf and address labels | [Ws brother-ql] |
| **Epson TM and other receipt printers** | ESC/POS | 58 or 80 mm paper | receipts, not labels; listed so the two jobs are not confused | [W node-thermal-printer] |
| **Bluetooth label printers** (for example a Phomemo M110 at a list price near 6,999 rupees, sold with 40 x 30 mm rolls) | usually driven through the maker's own app; the protocol is vendor-specific | about 203 dpi | the cheapest mobile route; open protocols exist only for some | [Ws phomemo-price; unverified] |

I could not confirm language support for the TVS and the Godex G500, and I have no test unit of anything. The first deployment question for an owner is "which printer do your customers already own", and the answer decides whether the raw path is TSPL, ZPL or only a driver.

### 2.3 Printer languages, from the manuals

| Language | Makers | What a label job looks like | Native barcodes | Text and the rupee sign |
|---|---|---|---|---|
| **TSPL / TSPL2** | TSC, and a family of look-alikes | `SIZE`, `GAP`, `DIRECTION`, `CLS`, then `TEXT`, `BARCODE`, `QRCODE`, `DMATRIX`, `BITMAP`, then `PRINT m,n` (m sets, n copies of each) | `128`, `128M`, `EAN128`, `39`, `93`, `EAN13` with add-on 2 or 5, `EAN8`, `UPCA`, `UPCE`, `ITF14`, `EAN14`, `CODA` and more; `QRCODE` (levels L, M, Q, H, model 1 or 2, auto or manual encode); `DMATRIX` (ECC 200, 10x10 to 144x144, rectangular option, FNC1 as `~1`, GS as `~]`) | built-in fonts and a `CODEPAGE` table of Western, Greek, Cyrillic, Hebrew, Arabic, Turkish and East-Asian pages plus UTF-8; **no Indian-script page**; a TrueType font can be downloaded [W tspl] |
| **ZPL II** | Zebra and many compatibles | `^XA` ... `^XZ` with `^FO` (position), `^BY` (module width in dots 1 to 10, ratio 2.0 to 3.0, default height), `^BC` Code 128, `^BE` EAN-13, `^BQ` QR, `^BX` Data Matrix, `^FD` data, `^PQ` quantity (up to 99,999,999) | `^B0` Aztec through `^BZ` Postnet: Code 11, 39, 93, 128, EAN-8, EAN-13, UPC-A/E, ITF, PDF417, MaxiCode and more | `^CI28` selects UTF-8 (firmware V60.14 or later), and the printer needs a Unicode TrueType font for non-Latin text; bitmapped fonts do not fully support it; Zebra's ZD220 sheet advertises native open-type font support and Unicode multi-language printing, which still needs a suitable (for example Devanagari) font loaded [unverified] [W zebra-by; W zebra-pq; W zebra-ci; W zebra-barcode-list; W zebra-zd220] |
| **EPL / EPL2** | Zebra (older), Eltron | line-oriented commands, mostly retired | basic 1-D | the ZD220 still carries five resident EPL2 fonts [W zebra-zd220] |
| **EZPL / GEPL / GZPL** | Godex | its own and emulations of the others | yes | check per model [Ws godex-ez2250i] |
| **CPCL** | Zebra mobile printers | for handheld belt printers | yes | out of scope here |
| **ESC/POS** | Epson, Star-compatibles | receipts: text, QR, images, cutter, drawer kick | QR, a few 1-D | code-page based; images for anything else [W node-thermal-printer] |
| **Brother raster** | Brother QL, TD | the host sends a raster image and commands | none (the host draws) | anything the host can draw [Ws brother-ql] |
| **Driver (GDI)** | everyone | the operating system renders the page to the printer's resolution | none | whatever the app's fonts can draw; the print dialog or Electron's print API drives it |

A small TSPL job for a 50 x 25 mm label (200 dots tall at 8 dots per mm), written from the syntax in the TSC manual and **untested on any printer** [J, built from W tspl]:

```
SIZE 50 mm,25 mm
GAP 3 mm,0
DIRECTION 1
CLS
TEXT 8,8,"2",0,1,1,"Samsung A15 128GB"
BARCODE 8,40,"128",56,1,0,2,2,"490154203237518"
TEXT 8,150,"2",0,1,1,"MRP Rs.17999 incl. of all taxes"
PRINT 3,1
```

(`PRINT 3,1` prints three sets of one copy; the bar is 56 dots, 7 mm, with two-dot modules, so the IMEI of section 1.7 takes 308 of the 400 dots. Whether `128` with narrow 2 and wide 2 is the right pairing for Code 128 I took from the manual's table [W tspl].)
A ZPL equivalent from the commands above, also untested and with `^PW` and `^LL` taken from general knowledge rather than a page I read [J, unverified]:

```
^XA
^PW400
^LL200
^FO8,8^A0N,24,24^FDSamsung A15 128GB^FS
^BY2,3,56
^FO8,40^BCN,56,Y,N,N^FD490154203237518^FS
^PQ3
^XZ
```

**The bitmap alternative.** TSPL's `BITMAP X,Y,width,height,mode,data` draws a 1-bit image (width in bytes, height in dots, mode overwrite, OR or XOR) [W tspl]; ZPL has an equivalent graphic-field command that I did not read [unverified]. A label rendered by the app at the printer's own resolution is
small: a 50 x 25 mm label at 203 dpi is 400 x 200 dots, 10,000 bytes; a 100 x 50 mm label at 300 dpi is 1,181 x 591 dots, about 87 KB; a 100 x 150 mm shipping label at 203 dpi is 800 x 1,200 dots, 120 KB [M]. That is trivial on a network or USB link and slow
only over a serial cable. The cost is that **the printer's own sharp bars and fast text are bypassed**, so the module width must be whole dots (the layout engine's job) and the bitmap polarity and byte order must be proved once per language on a real printer [unverified].

### 2.4 The print paths, one by one

**Path 1: the browser print dialog.** The page sets `@page { size: 50mm 25mm; margin: 0 }` (MDN lists lengths as legal for `size`, and the descriptor is Baseline since December 2024 [W mdn-page-size]), prints, and the person confirms in the dialog.
- *What the repository already does:* `PrintLeaf` injects exactly that rule only while a cheque is printing and says Chromium honours a custom size for PDF and for printers that accept it, while a driver that insists on its own stock still needs "actual size" [C `src/ui/banking/chequeLeaf.tsx:164-173`]. The named pages that exist are A4, A5 and Letter, each a separate `@page`
  because a page rule cannot read a custom property [C `src/styles/print.css:34-43`, comment at `:35-36`]; a label size is therefore an **injected style**, not a new named page.
- *Limits (anecdotal, [Ws chrome-label-print]):* margins and "headers and footers" must be off, scale 100%, the label's paper size picked in the driver, Chrome can ignore an `@page` size that leaves no content area, and its margin settings do not persist between prints. A printout is N pages: 500 labels
  is a 500-page job in one dialog, and a dialog interrupts the counter every time.
- *Bars must be SVG fills, not CSS backgrounds*: "Background graphics" is off by default and a filled CSS area vanishes [C `print.css:17-19`]; `QrSvg` already draws SVG paths for this reason [C `QrCode.tsx:5-33`]. SVG fills print with that option off in
  Chromium as far as I know [J, unverified]; the test sheet decides.
- *Feasible:* **now, on the HTTP site and in Electron.** It is the only path available to a plain browser on HTTP.

**Path 2: Chrome's kiosk-printing switch.** Vendor help pages for POS products say adding `--kiosk-printing` to Chrome's shortcut prints straight to the default printer with no dialog, on Windows; one says macOS does not support it, and several say behaviour changed between Chrome
versions [Ws kiosk-printing]. It is a deployment trick for a shop's counter PC, not a feature, and the app can only document it [J].

**Path 3: Electron `webContents.print`.** The Electron 31 documentation lists these options: `silent` (no dialog; an empty `deviceName` means the system default printer), `deviceName` (the system name, for example `Brother_QL_820NWB`, **not** the display name), `pageSize` (a named size or an object with height and width),
`margins` (with a `none` type), `copies`, `dpi`, `scaleFactor`, `landscape`, `printBackground` (default false), and a callback giving success and a failure reason; custom sizes must be at least 353 microns on a side, and `getPrintersAsync()` returns each printer's name, display name, description, status, default flag and options [W electron-v31-webcontents; W electron-printer-info].
- *Not stated on the pages I read:* the units of the `pageSize` object (the 353 figure suggests microns [J]), and whether `print()` honours a CSS `@page` size the way `printToPDF({ preferCSSPageSize: true })` does (the PDF bridge relies on that [C `electron/exportBridge.cjs:108-114`]). **Both need a test on the packaged Electron 31 binary**, which the Phase 29 notes say was never run for the PDF bridge either [C CLAUDE.md, Phase 29].
- *`printToPDF`:* the bridge already writes the mounted print host to a PDF at the CSS page size (`preferCSSPageSize: true`) [C `electron/exportBridge.cjs:108-114`; W electron-v31-webcontents]; a label PDF can therefore be saved and printed from any PDF viewer, a fallback for a shop whose driver mangles the dialog path. It is not silent and not a label workflow [J].
- *Fit:* a label printer installed with its driver. No raw code. The driver resamples, so bars can alias (section 1.7). Silent, named printer, many copies in one job.

**Path 4: Electron raw to the printer.** Three transports from the main process, none needing a browser permission:
- **TCP port 9100** to a network printer with Node's built-in `net`: the usual raw port; `node-thermal-printer` documents `tcp://host:9100` for receipt printers and RawBT's page mentions port 9100 for Ethernet or Wi-Fi printers [W node-thermal-printer; Ws rawbt]. Zebra, TSC and Godex network models are the natural targets [J, unverified per model].
- **The Windows spooler with the RAW data type**: Microsoft documents the sequence `OpenPrinter`, `StartDocPrinter` with a document type of "RAW", `StartPagePrinter`, `WritePrinter`, then the closing calls, which hands the bytes to a GDI-driver printer unmodified [W ms-rawprint]. From Node this needs a native module or a helper process. `node-printer` (MIT) exposes `printDirect` with RAW and `getPrinters`, but its page shows Node and node-webkit versions from a decade ago, 118 open issues and no release list, so it is a risk with Electron 31 [W node-printer]; a small PowerShell or C# helper that calls the spooler is the alternative [J].
- **USB direct (libusb) or a COM port.** USB direct on Windows normally needs a replacement WinUSB driver, which removes the vendor driver the shop relies on [J, unverified]; a COM port suits older serial printers (`\\.\COM1` in `node-thermal-printer`'s notes) [W node-thermal-printer]. macOS and Linux have CUPS; whether a raw queue (`lp -o raw`) still behaves as it used to I could not confirm [unverified].
- *Fit:* the best output and the only fast path for hundreds of labels; needs a **printer profile** (language, dpi, label size and gap, print width, offsets) that no browser path needs.

**Path 5: WebUSB, Web Serial, Web Bluetooth and WebHID from a Chromium browser.** All need HTTPS (or `localhost`), a user gesture and a device chooser; grants persist per device (`getDevices`, `getPorts`) [W mdn-usb; W mdn-serial; W mdn-bluetooth].
- Support by the caniuse pages I read: WebUSB Chrome 61, Edge 79, Opera 48, Samsung Internet 8.2, none in Firefox or Safari; Web Serial Chrome and Edge 89, Opera 76, Firefox 151, none in Safari; Web Bluetooth Chrome 56, Edge 79, Opera 43, Samsung 6.2, none in Firefox or Safari (an extension can add it on iOS, not counted) [W caniuse-usb; W caniuse-serial; W caniuse-bluetooth]. The same pages gave Chrome for Android a bare "156", which I do not trust as the whole story (Android Chrome has had Web Bluetooth and WebUSB for years in my understanding); **check the table directly** [unverified].
- **Web Serial reaches Bluetooth Classic printers that Windows has paired**, because it covers devices that appear as a virtual serial port (USB CDC and Bluetooth SPP) [W mdn-serial]; Web Bluetooth is Low Energy only and has no Classic profile [W mdn-bluetooth]. Many cheap thermal printers advertise a Low Energy write service (one forum reports service 18F0 with characteristic 2AF1, others use 18F1), so a BLE route exists for some and is device-specific [Ws ble-uuids].
- **In Electron**, `navigator.serial.requestPort()` and its siblings do nothing useful until the main process answers the `select-serial-port`, `select-usb-device` or `select-hid-device` events with a device id; without a device-permission handler, grants live in memory only [W electron-session].
- *Fit:* the only raw path for a **pure web build on a desktop Chromium**, once the site has TLS. Not for Safari, Firefox (serial only), or iOS.

**Path 6: a local agent (QZ Tray).** A helper app that pages reach over a local WebSocket; it prints raw jobs (ZPL, EPL, ESC/POS, ESC/P, DPL, PGL, SBPL) and pixel jobs (PDF, HTML, images), talks to USB, serial and HID, runs on Windows, macOS and Linux, and prints silently only if the messages are digitally signed [W qz-docs]. The licence was not stated on the pages I read [unverified].
Odoo's community answers send Zebra users to an IoT Box or a QZ-based module for the same reason [Ws odoo-print]. *Fit:* only if the **web** build must print raw before the Electron build is adopted; it adds an install, a certificate and a support burden per shop PC [J].

**Path 7: mobile.** An Android phone prints through Android's print service (printer vendors ship plug-ins; a "print service" app such as RawBT presents Bluetooth, USB and network thermal printers as a destination, and its page says it can be used from a website [Ws rawbt]) or through Web Bluetooth to a Low Energy printer. iOS Safari has none of WebUSB, Web Serial or Web Bluetooth [W caniuse-usb; W caniuse-serial; W caniuse-bluetooth], so
an iPhone prints through AirPrint, the vendor's app, or a native shell [J]. The mobile research reached the same place: build a thin native shell only if the field pilot shows that scanning, printing or background sync is a must-have [R `mobile-personas-and-jobs.md:803-804`].

### 2.5 Which path is feasible where

| Capability | Web on HTTP (today) | Web on HTTPS | Electron (Windows, Mac) | Android Chrome or PWA | Native shell (Capacitor) | iOS Safari |
|---|---|---|---|---|---|---|
| A4 sheet labels through the print dialog | yes | yes | yes | yes, through the system print sheet | yes | yes, through the share or print sheet |
| A roll label through a driver, `@page` size, dialog | yes (manual dialog choices) | yes | yes | clumsy | via a plug-in | only an AirPrint-capable printer |
| Silent print to a chosen printer | no (kiosk flag on Windows) | no | **yes** (`print`, `getPrintersAsync`) | no | via a plug-in | no |
| Raw TSPL or ZPL | no | desktop Chromium only (WebUSB, Web Serial); a prompt per device | **yes** (TCP 9100, spooler RAW, serial) | BLE printers only (Web Bluetooth) | **yes** with a plug-in | no |
| Bitmap graphics to a raw printer | no | as raw | yes | BLE | yes | no |
| Bluetooth Classic label printer | no | Web Serial if paired on the PC | Web Serial (paired) or a serial port | no | yes | no |
| Camera scanning | **no** (needs a secure context) | yes | needs a check and a permission handler (section 5.4) | yes | yes | yes (a WebAssembly decoder; native `BarcodeDetector` is off by default in Safari) |

### 2.6 How the existing export bridge is built, and what a label bridge copies

The desktop app already has the pattern for a **renderer that is not trusted** [C `electron/exportBridge.cjs:1-20`]:
- `main.cjs` defines `trusted(e)` as "the sender is the main window's own contents" [C `electron/main.cjs:61-62`] and wires each handler from a pure factory in `exportBridge.cjs` with its collaborators passed in (`isTrusted`, `getWin`, `showSave`, `fs`), so a test runs the same function against fakes [C `main.cjs:144-155`; `exportBridge.cjs:78-90`].
- A request is **validated, not repaired**: unknown keys, wrong types and values outside a list are refused with a reason [C `exportBridge.cjs:44-53`]; the renderer names a file *name* and options, never a path; one save at a time (`busy`) [C `:80-88`]; the file is written beside and renamed [C `:117-120`].
- `preload.cjs` re-sanitises the same options before the IPC (a second check) and exposes only `tallyweb.pdf.save` and `tallyweb.share.open` [C `electron/preload.cjs:30-41`]. `BACKUP_NAME_RE` is mirrored between `main.cjs` and `core/backupPolicy.ts` with a comment to keep them identical [C `main.cjs:33-34`].

A label bridge follows it [J]:
- `labels.printers()` returns `{ name, displayName, isDefault, status }` from `getPrintersAsync()` (nothing else about the machine).
- `labels.print(req)` takes **a printer name that must equal a name in the live list** (main re-queries, so a renderer cannot name an arbitrary device), a kind (`page` for a driver print of the mounted host, `raw` for a command stream), the label size in millimetres inside bounds (say 15 to 300), a copy count with a ceiling, and for `raw` a payload string.
- **The raw payload is the dangerous part**: a printer language can change the printer itself. The TSPL manual's command list includes `INITIALPRINTER` (restore the printer's settings to defaults), `MENU`, `EXPORT`, `DOWNLOAD` and the immediate commands that start with ESC and `!` (one restarts the printer, one cancels a pause, one disables the immediate commands) [W tspl]; ZPL has format and font downloads [unverified, not read]. The main process therefore accepts only **a short allow-list of command names (`SIZE`, `GAP`, `DIRECTION`, `CLS`, `TEXT`, `BARCODE`, `QRCODE`, `DMATRIX`, `BITMAP`, `BAR`, `BOX`, `PRINT` for TSPL, and the matching few for ZPL), a printable-ASCII charset plus CR and LF (and the bitmap's binary block inside a length-checked `BITMAP`), and a size ceiling**, refuses everything else, and the renderer builds the string with a pure `src/core` function the test also runs. The allow-list is mirrored in both places and asserted identical, as `BACKUP_NAME_RE` is.
- **TSPL can report its own status.** `<ESC>!?` returns one byte at any time, over RS-232, USB and Ethernet: normal, head open, paper jam, out of paper, out of ribbon (and combinations), paused, printing, or other error [W tspl]. A raw TSPL path can therefore say "the printer is out of paper" before and after a job, which no driver path can; that is a reason to prefer raw TSPL on Electron where the printer allows it [J].
- One job at a time; the answer is `{ ok, bytes, how }` and the screen says **"sent to the printer"**, never "printed": a raw write succeeding does not mean a label came out (the Phase 29 rule "Share means open, never send" is the same honesty applied to output [C CLAUDE.md, Phase 29 gotchas]).
- Packaging: any change here ships only with a **rebuilt installer** and a real printer test.

---

## 3. Label content and the Indian rules

**How to read this section.** It reports what the rules say, from primary text where I could get it, so the label designer can offer the right fields and the right warnings. It is **not legal advice and the app must not
read as if it were**: it prints what the owner chose, flags what a rule seems to require, and never says "compliant". The field catalogue by label kind (A to G) is in `tracking-verticals-india.md` section 6.6; this section adds the rule text and
the "who is responsible" line that catalogue lacked. Sources read in full or in the named part: the Department of Consumer Affairs' consolidated Packaged Commodities Rules (56 pages, text extracted locally; it runs to the 2022 amendments) [W lm-book];
FSSAI's compendium of the Labelling and Display Regulations, Version V of 13 October 2022 (42 pages) [W fssai-comp]; the CDSCO consolidated Drugs Rules (963 pages, September 2024 file; Rules 94 to 97 read) and CDSCO's FAQ on the QR-code rule [W drugs-rules; W cdsco-faq]; BIS's hallmarking pages [W bis-blog; W bis-faq].

### 3.1 Legal Metrology (Packaged Commodities) Rules, 2011 [W lm-book]

| Rule | What it says (paraphrased) | What it means for the app |
|---|---|---|
| **2(g)** | a *packer* is anyone who pre-packs any commodity, in a bottle, tin, wrapper or otherwise, in units suitable for sale wholesale or retail | **a trader who repacks bulk goods into sale packs is a packer**; the Rule 6 declarations become that trader's duty |
| **2(m)** | the retail sale price is the maximum price at which a packaged commodity may be sold, **inclusive of all taxes** (substituted by the 2017 amendment, in force from 1 January 2018) | the declaration states the price as a maximum retail price inclusive of all taxes; an app that stores a tax-exclusive rate cannot derive the MRP (the POS audit's finding [R `pos-code-audit.md` 1.3]) |
| **3** | the retail-package chapter does not apply to packages above 25 kg or 25 litres, to cement, fertiliser and farm produce in bags above 50 kg, or to goods meant for industrial or institutional buyers | a carton or bulk sack label is a different kind of label |
| **4(1), 4(2)** | nobody may pre-pack for sale unless the package carries the declarations; where several retail packages are grouped as one promotional pack, **each** package must comply | |
| **6(1)(a)** | name and address of the manufacturer; where the manufacturer is not the packer, both; for an imported package, the importer; a company named with no qualifier is presumed to be the manufacturer | the template has "Mfd by", "Packed by" and "Imported by" as separate, owner-chosen lines |
| **6(1)(aa)** | country of origin for an imported product | |
| **6(1)(b), (c), (f)** | common or generic name; net quantity in the standard unit (or the number); dimensions where size matters | |
| **6(1)(d)** | the month and year in which the commodity is manufactured. The consolidated book shows the words "or pre-packed or imported" struck out from 1 October 2022, yet leaves the older "pre-packed" wording in other rules | **what a packer or importer prints now is not settled by the text I read**; the template lets the owner choose the wording (Mfg, Pkd, Imp) and says the rule's wording changed in 2022 |
| **6(1)(da)** | "best before" or "use by" for a commodity that can become unfit | |
| **6(1)(e)** | the retail sale price, stated as a maximum retail price inclusive of all taxes, in Indian currency. The book's earlier illustrations used "Rs." and forms like "MRP Rs. ... incl. of all taxes" and a rounding rule; the 2022 substitution (in force 1 October 2022) replaced the illustrations and the rounding wording with the words "in Indian currency" | **keep the words "MRP" and "inclusive of all taxes" fixed**; never offer an MRP field without them |
| **6(2)** | name, address, telephone number and e-mail of the person or office to contact for consumer complaints | |
| **6(3), 6(4)** | individual stickers may not be used to alter or to make a required declaration, **except** a sticker carrying a revised **lower** MRP, which must not cover the maker's own MRP; stickers are allowed for any *other* declaration | a shop's price or barcode sticker is "another declaration": **the layout must keep it off the maker's printing** (a placement warning, not an enforcement) |
| **6(4A)** | a manufacturer, packer or importer may additionally print a barcode, GTIN or QR code | the shop's code beside the maker's declarations is allowed; the maker's is theirs |
| **6(9)** | on an imported package a label may be affixed to make the declarations | an importer's declaration label is a real use case (label kind D) |
| **6(11)** | a **unit sale price** in rupees to two decimals (per g or kg, cm or m, ml or litre, or per unit) except where the retail price equals the unit price; the book dates it from 1 October 2022 and carries a statement that no prosecution would be started over the declaration from 1 April 2022 under the 2021 amendment | the template computes the unit price from MRP and net quantity and lets the owner decline when equal |
| **7** | letters at least 1 mm high; the minimum height of numerals and letters rises with the area of the principal display panel (in bands) | the pack-label designer enforces a floor of 1 mm and warns below the band for the panel area. **My text extraction of the amended table is scrambled, so I give no figures; read Table I of the gazette before coding the bands** |
| **8, 9** | declarations on the principal display panel with clear space round the quantity (at least the numeral height above and below, twice that left and right); legible and prominent; the MRP and net-quantity numerals in a colour that contrasts with the background | black on white, and a clear-space rule in the designer |
| **18(2)** | nobody, retailer or packer or importer, may sell a packed commodity above its retail sale price | a sales line rate above a stored MRP deserves a warning (needs MRP stored; today it is not [R POS audit]) |
| **18(5)** | nobody may obliterate, smudge or alter the retail sale price printed by the maker, packer or importer | **the shop's own sticker must never cover it**; this is the single most important layout rule for kind A labels |
| **18(7)** | retailers of packaged goods sold by weight or volume must keep an electronic weighing machine of at least class III accuracy with a printed receipt, for customers to check | no app work |

The Act's definition of a pre-packaged commodity (a commodity placed in a package without the purchaser present so that its contents have a pre-determined quantity) is in section 2(l); a 2016 law-firm note says the edges were litigated and a
three-judge Supreme Court reference was then pending, so **whether a particular repacking counts is not for the software to decide** [Ws lm-act]. A search summary also mentions a Legal Metrology amendment of 2023 with a no-prosecution date of 1 January 2024
that I could not reconcile from the results, so the consolidated book above is current only to the 2022 amendments [Ws lm-amend; unverified].

### 3.2 FSSAI (Labelling and Display) Regulations, 2020 [W fssai-comp]

| Regulation | What it says (paraphrased) | Label consequence |
|---|---|---|
| **4(5)** | declarations in English or in Hindi in Devanagari script; other languages may be added but must not contradict | a bilingual template; the label engine must draw Devanagari (a bitmap path or a font, section 2.3) |
| **4(7)** | content clear, prominent, indelible and legible | |
| **5(7)** | the FSSAI logo and the 14-digit licence number on the label (the compendium's example shows 14 digits), in a colour that contrasts with the background; the brand owner's number, and the manufacturer's or packer's if different; the importer's for imports | a licence-number field on the company, checked for the 14-digit shape the example shows [J] |
| **5(8)** | net quantity, retail sale price and consumer-care details **follow the Legal Metrology rules** | the same fields as 3.1 |
| **5(9)** | a batch, code or lot number must be declared | the batch field |
| **5(10)** | "date of manufacture or packaging" and "expiry or use by" must be declared; "best before" may be added; day, month and year as DD/MM/YY for a shelf life up to three months, otherwise month and year (an amendment lets DD/MM/YY be used for the longer shelf lives too); month names as capital letters with at least three letters; the two dates grouped at one place; storage conditions when validity depends on them | the date formatter has two modes by shelf life and a month-name option |
| **8(1)** | a package of 100 square centimetres or less is exempt from the ingredients, batch number, nutrition, licence number and logo declarations (given on the multi-unit pack instead); one under 30 square centimetres is exempt from the manufacture and use-by dates | **a small sticker label is not a statutory label**; the designer shows the package area and the exemption it implies |
| **8(3)** | for a shelf life of seven days or less the manufacture date may be omitted but the expiry must show | |

The compendium is a reference, not the law; FSSAI's amendments page, seen only through a search summary, lists five amendments with the latest dated 24 March 2026, which I did not read [Ws fssai-search; unverified].
A separate FSSAI order made the 14-digit number mandatory on bills and invoices from 1 October 2021, which belongs to the invoice, not the label [Ws fssai-search].

### 3.3 Drugs and Cosmetics Rules, 1945 [W drugs-rules; W cdsco-faq]

- **Rule 96(1)** (the maker's label on the innermost container and every covering, printed or written in **indelible** ink): the drug's name (the proper name at least two font sizes larger than a brand name, with exceptions); net contents; active-ingredient content; the manufacturer's name and the address of the premises; a **distinctive batch number** preceded by "Batch No.", "B No.", "Batch", "Lot No." or "Lot"; the **manufacturing licence number** preceded by
  "Manufacturing Licence Number", "Mfg. Lic. No." or "M.L."; the manufacture and expiry dates (for a drug outside Schedule P the expiry may not exceed sixty months from manufacture); a red-box warning for the scheduled categories; the import licence number and importer for imports; the marketer's name and address where one exists.
  I searched the consolidated text for a price clause and found none in Rule 96 [W drugs-rules]; where an MRP comes from for medicines (the general packaged-commodity rules, price-control orders) I did not read [unverified].
- **Expiry convention.** Rule 96's explanation: an expiry is a month and year, it means the drug is recommended up to the **last day of that month**, and it is preceded by the words "Expiry date" [W drugs-rules, rule 96].
- **Rule 96(6)-(7), the QR rule for the top 300 brands** (Schedule H2, notification GSR 823(E) of 17 November 2022, in force 1 August 2023): the manufacturer prints or affixes a **barcode or QR code** on the primary label (or the secondary label if space is short) that stores a unique product identification code, the proper and generic name, the brand name, the manufacturer's name and address, the batch number, the manufacturing date, the expiry date and the manufacturing licence number. CDSCO's FAQ adds that the **unique product code is whatever the manufacturer's own procedure says** and that any batch manufactured on or after 1 August 2023 is covered [W cdsco-faq; W drugs-rules]. **No GS1 structure is required**, so a pharmacy's reader must handle a maker's own string as well as GS1 ones [J]. Rule 96(5) separately requires a QR on every active pharmaceutical ingredient with a longer list of data (the rule text is in the same file) [W drugs-rules]. Newer reports of an extension to vaccines, narcotics, anti-cancer drugs and antimicrobials were not checked against a notification [Ws cdsco-qr-search; unverified].
- **Rule 94(2), a dispensed medicine.** A medicine made up ready for treatment and supplied on a doctor's prescription is outside Rules 96 to 101 if its label carries the supplier's name and address, the patient's name and the quantity, the serial number of the entry in the prescription register, the dose for internal use, and "for external use only" where it applies [W drugs-rules, rule 94(2)]. That is the **pharmacy dispense label** (kind G); it is the dispenser's own label and an inventory app may print it.
- **What the app must not do:** re-print the maker's batch, dates or licence number onto a maker's pack (those are the maker's and the rule asks for indelible printing); a **rack or shelf tag** with batch and expiry for the pharmacy's own use is an internal tag and is fine [J].

### 3.4 BIS hallmarking and the HUID [W bis-blog; W bis-faq]

- Each hallmarked article carries three marks: the BIS logo, the purity mark (carat and fineness) and a **six-character alphanumeric HUID**, a code generated by BIS's server and **laser-marked by a BIS-recognised assaying and hallmarking centre**; the three-mark system began on 1 July 2021, replacing a four-mark one. Only BIS-registered jewellers send articles to a centre; buyers check a HUID in the BIS Care app. Covered grades are 14, 18, 20, 22, 23 and 24 carat under IS 1417 [W bis-blog].
- The FAQ page I read lists what a jeweller's **bill** must show (each hallmarked article, net precious-metal weight, carat and fineness, the hallmarking charge, and a note that purity can be verified at a centre) and that each item of a pair and each detachable part is hallmarked; it did not mention the HUID, so I could not confirm any rule about printing it on a bill or tag [W bis-faq]. A search summary of the same FAQ said it describes the HUID; the pages disagree and I trust the fetched one [Ws bis-search].
- **Consequence [J].** The HUID is an attribute of the **unit** (the tag), entered or scanned *after* hallmarking, never generated by the app; the jewellery tag can print it as text (it is six characters and not a barcode in anything I read). A tag template with fields for tag number, gross, stone and net weight, purity and HUID is a normal shop tag.

### 3.5 Who prints what, and what the app may do

| Label kind (from `tracking-verticals-india.md` 6.6) | Legally on the hook | What the app may print | Guardrails |
|---|---|---|---|
| **A. Shelf or price tag** | the retailer for what it states | name, barcode, **selling price**, optionally the maker's MRP and a unit price | never covers the maker's declarations (6(3), 18(5)); an MRP shown carries the locked wording; selling price should not exceed MRP (18(2)) |
| **B. Garment or footwear tag** | the brand or packer for MRP and declarations; a retailer for its own tag | style, size, colour, barcode, MRP, price code | an own-brand garment makes the owner the maker; the declaration template is available |
| **C. Weighed or in-store-packed label** | the packer (the shop) when it pre-packs; the scale for a weighed loose item | the scale prints its own; for a pre-pack, the Rule 6 and FSSAI fields | unit price, packed-on date and the "packer" line are required fields in the template |
| **D. Statutory pack label** | the **packer, importer or manufacturer** | every Rule 6, FSSAI or Drugs field the owner fills | the app flags missing fields, locks the MRP wording and the 1 mm floor, never fills a date or a number silently, never says "compliant" |
| **E. Jewellery tag** | the jeweller | tag, weights, purity, HUID (recorded), barcode | HUID entered after hallmarking, never generated |
| **F. Unit or serial sticker** | the shop | IMEI or serial barcode, model, price, warranty end | no statute; IMEI check digit is the data's own rule (section 1.4) |
| **G. Pharmacy dispense label** | the dispenser (Rule 94(2)) | supplier name and address, patient, quantity, register serial number, dose, external-use words | not the maker's pack; no maker's data re-printed as the maker's |

### 3.6 Language, currency and the rupee sign

FSSAI wants English or Hindi in Devanagari on food labels [W fssai-comp, 4(5)]; Legal Metrology's earlier illustrations (replaced in 2022 by the words "in Indian currency") wrote the price as "Rs." or "MRP Rs. ... incl. of all taxes" [W lm-book, 6(1)(e)]. The TSPL and ZPL built-in fonts and code pages offer no Indian script and, in the TSPL
table, no rupee glyph (the code pages listed are Western, Greek, Cyrillic, Hebrew, Arabic, Turkish and East-Asian ones, plus UTF-8 [W tspl]); a printer-font job should say "Rs." and a **bitmap job** (or a driver print, where the app already loads IBM Plex for the rupee sign
[C `src/ui/print/printFonts.ts:1-25`]) can draw the sign and Devanagari. This is one more reason section 2.3 prefers the bitmap route for statutory labels [J].

---

## 4. The label design experience

### 4.1 How the products people compare against do it

| Product | Design model | Data source | Sizes and printers | Where it is weak (as the pages show it) | Source |
|---|---|---|---|---|---|
| **TallyPrime** | none: the help article is about **scanning**; the code lives in the stock item's Alias field and a scan at the item-name field picks the item; the article says nothing about printing labels; partners sell add-ons | alias | n/a | no label output of its own; one code per item | [W tally-help]; add-ons [R `batch-serial-tracking.md` 8.3] |
| **Busy** | an add-on: Add-On menu, *Item Barcode Printing*, *Configure Barcode Labels*, pick a format, *Configure Format*: barcode type and print settings (default printer, paper type, top and left margins), label height and width, then fields from a panel: barcode, QR, data field (item name, alias, price, MRP), image, plain text | the item master, a voucher, a current-stock list ("print barcodes from current stock status": pick item, format and date), or an Excel sheet (map label fields to columns, give path, rows and sheet) | the printer must be default, with the right driver, the roll loaded and the printer calibrated; a barcode value must be 5 to 20 characters; a type *Item+Batch No.* carries a batch | the FAQ is full of **skipped labels, misaligned and too-small barcodes** (every fix is "match the label size, calibrate, adjust margins"); barcodes find items but **cannot create them**; design is a chargeable partner service [R `batch-serial-tracking.md` 8.3] | [W busy-labels; W busy-faq3; W busy-faq5] |
| **Marg ERP** | a report: *Daily Reports, Inventory Reports, Barcode Printing, Selected Batch* prints batch-wise barcodes and labels; the retail edition is said to take SKU, serial or batch codes and custom label design; "Scan & Bill" for strips and bottles | batch and item | I could read only the first step of the help article | not verified beyond a menu path and marketing | [W marg-care (first step only); Ws marg-search] |
| **Vyapar** | a utility: *Utilities, Barcode Generator*: choose printer type (a thermal printer for stickers, or a laser printer for A4 sheets, with 24-up and 40-up named) and label size, pick an item (its **item code** is the barcode, or type a manufacturer code as the item code), enter the number of labels and a header and two text lines, add more items, preview, generate, print or save | the item master | no label-size list published | one code per item, no batch or unit label; barcode generation is a paid-plan feature per the sibling research | [W vyapar-barcode; R `batch-serial-tracking.md` 8.3] |
| **Zoho Inventory** | a **template editor**: page properties (a label-sheet size or custom dimensions, margins, rows and columns), template properties (label margins, font, barcode type such as Code 128, which fields: item name, SKU, rate, organisation), *Apply Recommended Barcode Size*, a label preview and a page preview; two default templates (linear, QR); labels for individual batch or serial numbers when tracking is on | item, bulk selection (capped), batch, serial | the template's paper size and label dimensions must match what the printer is set to | a cap on bulk selection; a community report that a Zebra desk printer would not print; no label at receiving | [W zoho-labels; R `batch-serial-tracking.md` 8.3] |
| **Odoo** | a choice of **fixed formats** on the operation type: PDF grids (2x7 with price, 4x7, 4x12) and ZPL for Zebra, printed by hand from a delivery or automatically at validation; ZPL can be edited as a view in developer mode; location barcodes as a PDF; an unknown scanned product can be looked up in a barcode database and created | product, lot or serial | direct Zebra printing needs an IoT Box or a third-party QZ module | the design is a developer task | [W odoo-labels; W odoo-software; Ws odoo-print] |
| **BarTender** (the specialist tool) | a drag-and-drop designer with a wizard, saved templates, a library of over 400 ready barcode components, database connectors (Excel, Access, Google Sheets, MySQL, JSON), ERP integration by file or web-service or database triggers, an automation edition that prints on events | any | any printer with a driver; **Windows only**; edition-gated | heavy, priced, separate from the books | [Ws bartender] (vendor and reseller pages) |

### 4.2 What the comparison says

1. **Everyone's label is an item label.** A label born from a receipt (one per unit or per quantity just booked, carrying the batch, expiry and MRP just typed) is not what any of them lead with (the sibling research reached the same finding [R `batch-serial-tracking.md` 8.3]). That is the room to be better.
2. **The pain is calibration and alignment, not layout.** Much of Busy's barcode FAQ is about skipped labels, misalignment and barcodes too small [W busy-faq5; W busy-faq3]. A **test label, a per-printer offset and a "scan it back" check are worth more than a designer** [J].
3. **Design is gated.** Paid at Busy, a developer task at Odoo, capped at Zoho. A small set of **good fixed layouts a person can adjust field by field** is what a shop owner actually uses [J].
4. **Fixed templates plus a field picker with a live preview is the model that a keyboard-first app can do well.** A free drag-and-drop designer needs hit-testing, snapping, alignment, stacking order, text fitting and undo; it is a product of its own, and BarTender exists for the shops that need it (the app can export a CSV or Excel a designer can read, which Busy's and Zoho's users already do [W busy-faq3; Ws zoho-export]) [J].
5. **The repository already has the right editor shape.** The cheque layout editor is a keyboard-driven grid of numeric cells (position, size) with a dashed focus frame on the sample, a **test sheet** (a 10 mm ruler grid printed at "actual size" to hold against a real leaf), and a **calibration shift** saved per bank [C `src/ui/banking/chequeLayoutEditor.tsx:200-260`; `src/types.ts:916-931`]. A label field editor is the same machine with different cells.

### 4.3 A label is data, rendered by one layout engine

One pure function turns a **template plus a row of data** into a list of drawing operations in millimetres; three renderers consume it: an SVG for the preview and the browser print, a 1-bit bitmap at the printer's dpi for raw graphics, and (optionally) native printer commands. A sketch of the shape, **not code to paste** [J]:

```ts
interface LabelTemplate {
  id: string; name: string; kind: 'shelf' | 'unit' | 'batch' | 'hangtag' | 'jewel' | 'pack' | 'dispense' | 'location';
  stock: { kind: 'roll'; wMm: number; hMm: number; across: number; gapXMm: number; gapYMm: number; printWidthMm?: number }
       | { kind: 'sheet'; page: 'A4'; cols: number; rows: number; marginTopMm: number; marginLeftMm: number; pitchXMm: number; pitchYMm: number; labelWMm: number; labelHMm: number };
  fields: Array<
    | { t: 'text'; src: string; xMm: number; yMm: number; wMm: number; hMm: number; sizePt: number; bold?: boolean; align?: 'l' | 'c' | 'r'; fit: 'shrink' | 'clip' | 'wrap'; maxLines?: number }
    | { t: 'barcode'; sym: 'ean13' | 'code128' | 'code39' | 'qr' | 'dm'; src: string; xMm: number; yMm: number; wMm: number; hMm: number; moduleDots?: number; hri: boolean }
    | { t: 'box' | 'line' | 'image'; xMm: number; yMm: number; wMm: number; hMm: number }
  >;
}
// a printer PROFILE (language, dpi, print width, offsets, gap sensing) is separate from the template
```

Reasons for each choice [J]: millimetres are what the owner measures and what the cheque editor already uses; `fit` is how a long Hindi product name does not overflow a 38 mm label; the printer **profile** (offset, dpi, language) is separate so one template prints on a TSC at the counter and a Zebra in the godown; `moduleDots` lets the engine round bars to whole dots for the chosen dpi (section 1.7). The text drawing for the bitmap path needs a text-fitting step that measures glyphs in the browser (canvas), which the print engine already does for reports
(`fitToPage` measures with the real face [C `src/ui/reportTools.ts:234-256`]).

### 4.4 The starter library, by business

Each template has a size, the fields it carries and the code it prints. **Sizes are starting points to confirm with five shops per vertical on real stock** (the sibling research makes the same ask [R `tracking-verticals-india.md` 6.6]).

| Template | Stock | Fields | Code | Note |
|---|---|---|---|---|
| **Shelf price label** (retail, grocery, hardware) | roll 50 x 25 or 38 x 25; or A4 40-up | name (two lines), selling price large, optional MRP and unit price, rack | Code 128 of the item code, or the item's EAN-13 | kind A; never covers a maker's price (3.1) |
| **Phone unit sticker** (kind F) | roll 50 x 25 or 50 x 30 | model, storage, colour, **IMEI 1** barcode and text, warranty end; a second sticker (or a second code) for **IMEI 2** | Code 128 (a 15-digit IMEI is 38.5 mm at two dots) or Data Matrix 14x14 | dual-SIM phones carry two IMEIs; the label prints one per unit by default |
| **Electronics carton label** | roll 100 x 50 | model, colour, serial or both IMEIs, date received | Code 128 and QR | |
| **Pharmacy shelf or batch label** | roll 50 x 25, or A4 24-up | name, **batch**, **expiry (MM/YY)**, MRP, rack | Code 128 of item plus batch | an internal tag (3.3); expiry on a month's last day |
| **Pharmacy dispense label** (kind G) | roll 75 x 50 or an A5 sheet | the Rule 94(2) fields: supplier, patient, quantity, register serial number, dose, external-use words | none | the dispenser's own label |
| **Revised-MRP sticker** | roll 38 x 25 | "MRP Rs. __ (revised, incl. of all taxes)" | none | allowed only as a **lower** MRP and not covering the original (6(3)); a warning says so |
| **Garment hang tag** | roll 50 x 25 (clothing) or 75 x 50 (shoes) [W commmerce-sizes] | brand, style, size, colour, MRP, price code | EAN-13 or Code 128 of the **variant** code | one code per size and colour; variant keys by option id so a renamed colour keeps its code [R `tracking-verticals-india.md` 6.5; C `.wt-cf/docs/design/custom-fields-spec.md:571,582` (per-variant barcodes are explicitly not in the custom-fields slices)] |
| **Hardware rack or bin label** | roll 75 x 50 or A4 12-up | large name, code, location | QR and Code 128 | |
| **Jewellery tag** (kind E) | roll 25 x 15 or 40 x 30 [W commmerce-sizes] | tag number, gross, stone and net weight, purity, **HUID** (recorded after hallmarking), making or price code | Code 128 or a small Data Matrix | small stock: 25 mm is 200 dots at 203 dpi |
| **Statutory pack label** (kind D) | roll 60 x 40 up to 100 x 60 | every 3.1 to 3.3 field, with the unit price, FSSAI number and licence fields | EAN-13 if the owner holds a GS1 prefix, else none | only for a packer or importer; declaration warnings (3.5) |
| **Book price sticker** | roll 38 x 25 | ISBN barcode, price | EAN-13 (978 or 979) with an EAN-5 price add-on [W wiki-ean; W wiki-gs1-codes] | |
| **Location label** | roll 50 x 25 or A4 | godown, rack, bin | QR and Code 128 | used by stock-take scanning (section 5.5) |

### 4.5 Which template prints, how many, and where it came from

- **Choosing the template** [J]: item first, then the item's stock group, then the voucher type or purpose, then the company default, then the built-in set for the company's business type. Storage is the open question: a **new table** (a Dexie version bump and a synced collection, which every sync, undo and backup path then handles) or a field on the **company document** like `reportViews`, which brings the Phase 27 older-client protection duty (a revision, a local mirror, a server keep-current rule for a write that states no base [C CLAUDE.md, Phase 27 gotchas]). On the server side a new table is accepted by `/sync/push` without a deploy, but the backend's `DATA_TABLES` list (used to accept conflict reports) must be extended and a suite fails when it drifts from the client's list [C `backend/src/conflict.js:74-87`]. The build spec decides; the research only records that both are real costs.
- **Copies = quantity received.** On a purchase line the default is the received quantity (per piece or per pack by the unit chosen), one label **per serial** for a serial-tracked item, one per batch line for a batch-tracked item; the person can override every row. A raw job sends `PRINT sets,copies` (TSPL) or `^PQ` (ZPL) so 500 identical labels are one job, not 500 pages [W tspl; W zebra-pq]; the browser path pays 500 pages and must say so.
- **Bulk by selection.** The Phase 26 selection bar already sends a chosen set of vouchers to bulk print with the `I` key and a ceiling of 100 [C `src/core/bulkPrint.ts:14`; `src/ui/reports/selectionBar.tsx:167`]; a label queue is the same gesture over stock items, batches or units, with an editable quantity column and a larger ceiling for raw jobs. Whatever the ceiling, a job over it is **refused whole with the reason**, never cut (the bulk-print rule).
- **A label is also an entry point, not a menu.** Offer it where the units appear: after Accept on a purchase (an opt-in voucher-type flag next to `printAfterSave` [C `src/types.ts:543`; `VoucherScreen.tsx:2826-2829`], off by default), from a batch list, a serial list, Stock Query, and a stock-item selection. Chords go into the registry, none chosen here.
- **Reprint** is from a **label log** (who, when, template, printer, count, the voucher or unit it came from): a unit's label is **reprinted, never re-minted**, because the IMEI is the same.
- **Dialogs follow the Back-control rules**: the label dialog is a `Box` with `onClose` and a visible Close, and registers a dirty probe while it holds an edited queue [C `.wt-cf/docs/design/back-nav-spec.md` rules B5 and B12, lines 27 and 34].

### 4.6 Preview, calibration and proof

A label that does not scan is worse than none. The proof chain [J]:
1. The preview is the **same drawing** that prints (the layout engine's output), at 1:1 with a dots ruler.
2. A **test label** per printer profile: the ruler grid, EAN-13 and Code 128 at two and three dots, a QR and a Data Matrix, the printer's own sense of size (a frame at the nominal edge), and a sentence telling the person to print at **actual size, margins none, scale 100%** for a driver path.
3. **A per-profile offset** (X and Y in millimetres, may be negative) and a **start cell** for a part-used sheet, saved on the profile, exactly as the cheque leaf saves its calibration shift [C `src/types.ts:928-929`].
4. **"Scan it back"**: after the test label, the dialog asks the person to scan it with the shop's own scanner, shows what the scanner delivered (raw characters, the symbology identifier if present, the terminator) and compares it with what was meant. This catches a bar that is too thin, the wrong layout, a missing separator and a dead terminator in one step, and it doubles as the scanner setup screen of section 5.
5. Honest output wording: the raw path says **"sent to the printer"** and, for TSPL, may add the printer's own status byte (out of paper, head open) [W tspl `<ESC>!?`]; it never says "printed".

---

## 5. Barcode scanning

### 5.1 How a scan reaches the app

| Device | What the app receives | Notes | Reaches |
|---|---|---|---|
| **USB or Bluetooth scanner in keyboard (HID) mode** (the default of nearly every hand scanner) | key events: the characters of the code, then a terminator, usually Enter and sometimes Tab | the operating system turns key positions into characters with the **active keyboard layout**; a Bluetooth scanner paired to a phone types into the focused field the same way | every browser, Electron, an Android or iOS browser |
| **Serial mode** (USB CDC, "virtual COM") or **Bluetooth SPP** | bytes, exactly as sent, so a GS character and any prefix survive | needs a serial reader | Electron (serial), Chromium over HTTPS (Web Serial: it covers virtual serial ports from USB CDC and Bluetooth SPP [W mdn-serial]) |
| **HID POS / vendor reports** | reports through WebHID or a vendor driver | model-specific | Chromium over HTTPS (WebHID), Electron (`select-hid-device`) [W electron-session]; **unverified per model** |
| **Camera** (phone, tablet or webcam) | decoded strings from the Barcode Detection API or a WebAssembly decoder | needs a secure context | section 5.4 |
| **A phone as the desk's scanner** (the phone decodes and relays to the desktop session) | a string over the app's own sync or realtime channel | not researched | a later option once the site has TLS [J] |

The first row is where almost all of the work is, because it is the only one that needs no permission, no driver and no TLS, and it is the one a keyboard-first app handles worst.

### 5.2 Wedge scanners: what goes wrong, and the defence

| Trap | What happens | Evidence | Defence |
|---|---|---|---|
| **Terminator** | Enter by default; some scanners send Tab; in this app Tab on the item field is the browser's own Tab and lands on Narration (the custom-fields spec records the same, and adds a Tab branch only when a company has line fields) | onScan.js treats keys 9 and 13 as scan-end by default [W onscan]; [C `.wt-cf/docs/design/custom-fields-spec.md:992`] | accept **Enter or Tab** as the end; the setup screen records which this scanner sends |
| **Prefix and symbology identifier** | `]E0`, `]C1`, `]d2` or a control character arrives before the data (section 1.6) | [W biip-sym; W zebra-ds4678-nav] | strip a known prefix, keep it as a hint, accept codes with or without it |
| **Layout mismatch** | the scanner's layout must match the computer's; symptoms are swapped letters (Y and Z on a German layout) and digits turned to symbols (AZERTY) | Odoo's troubleshooting page calls a layout mismatch the main cause of barcode errors and gives a French-Belgian example [W odoo-troubleshooting]; [Ws layout-search] | decode from the **physical key** (`KeyboardEvent.code`) plus Shift for a scanner left on US, or from `key` when the scanner is set to the computer's layout: a per-device **scanner setting** chosen by the setup scan [J] |
| **Indian input languages** | a shop that switches to Hindi, Marathi or another script layout may get script characters, or a Hindi input method may swallow the keys (an IME marks them as composing) | not researched [J, unverified] | the same `code`-based decode ignores the layout, and the setup screen shows "what arrived" so a bad day is diagnosable |
| **Caps Lock** | a scanner that sends capitals by Shift can arrive lower-case when Caps Lock is on (the browser inverts the case) | Zebra lists "Caps Lock override" and "convert case" among its USB keyboard parameters [W zebra-ds4678-nav] | `code` plus Shift gives the case the scanner meant; the setup scan includes upper and lower case |
| **Dead keys and AltGr** | on an international layout an apostrophe or quote is a dead key that composes with the next letter; AltGr reports as Ctrl+Alt | the key-translation layer never rewrites a Ctrl+Alt press, so an AltGr-typed character reaches the field unchanged [C `src/keys.ts:89-91`] (the favourite slots, Ctrl+Alt+1 to 9, are the one family of Ctrl+Alt chords the app binds [C CLAUDE.md, Phase 23]); `Dead` is recognised for the Option key on a Mac [C `keys.ts` `typesSymbol`] | `code`-based decoding does not compose; punctuation in codes is rare in Indian trade |
| **Alt plus numeric keypad ("keypad emulation")** | the scanner sends every character as Alt held while digits are typed on the keypad (the letter A as Alt, then 0 6 5); some Zebra models ship this enabled, others disabled | [W zebra-keypad; Ws zebra-keypad-defaults] | **this mode makes every digit a chord in this app**: `combo()` reads the keypad digit with Alt held as `Alt+<digit>`, because it recovers a physical key only for `Digit` codes, not `Numpad` ones [C `keys.ts:206-222`], and the app binds `Alt+2` (Duplicate, and others), `Alt+8` and `Alt+9` [C `core/shortcutRegistry.ts:240`; grep of `'Alt+2'` shows 11 uses]. A scan containing a 2 on a screen that binds `Alt+2` could duplicate a voucher. The scan layer must **recognise Alt plus keypad digits as a character and swallow them**, and the setup screen must name the mode and tell the person to turn it off [J] |
| **The group separator (GS)** | a keyboard cannot type it; the scanner sends Ctrl+] or a layout-dependent variant, or drops it, or swaps it for Tab or space; a German-layout setting is reported to send Ctrl and a plus sign, which many browsers read as zoom | [W gs1parser; W cleverence-gs; Ws gs1-dm-fnc1] | map Ctrl+] to GS inside a scan, accept a missing separator with a visible "assumed", prefer serial mode for GS1 work, and say so in the setup screen |
| **Speed** | a scanner types about 5 to 10 characters per frame [J, unverified]; the voucher screen re-renders per keystroke and, at 120,000 vouchers, each keystroke costs about 66 ms, so one 13-digit scan is about 0.86 s of work | [R `pos-code-audit.md` 1.3 (computed there from a measured per-keystroke cost; I did not re-run it)] | a scan target takes the burst as **one change**, not thirteen renders; a burst test with synthetic keys at 2 to 5 ms spacing is a gate [R `batch-serial-code-audit.md` 3.14] |
| **Unknown characters and delay** | characters absent from the layout may be dropped or sent anyway; some scanners and old PCs need a delay between keystrokes | Zebra's "barcodes with unknown characters" (send by default) and "keystroke delay" (none by default) [Ws zebra-defaults] | the timing threshold is **learned** from the setup scan, never hard-coded |
| **A burst on the wrong control** | in Narration the terminator is Accept; on the Accept bar Enter answers Yes; on a menu or report letters are hotkeys; on a list the text filters | [C `VoucherScreen.tsx:3367`; `src/ui/widgets.tsx:426` (list filter); `keys.ts:11-34`; the walk table in `batch-serial-code-audit.md` 3.14] | the **scan target model** of 5.3 |

### 5.3 Telling a scan from typing, and where to put it

**The measured defaults of a well-known browser library** (the numbers a design can start from): it waits 100 ms after a key to decide whether a scan has ended, expects about 30 ms or less per character on average, ignores anything shorter than 6 characters, and ends on Enter or Tab; it can ignore scans while a chosen element has focus and can also read scans that arrive as a paste [W onscan]. Humans type at well over 100 ms per key and a fast typist at 60 ms or more [J]; the gap is wide enough for timing to work but narrow enough that a slow scanner or a loaded machine crosses it, hence a learned threshold.

**Three tiers, cheapest first [J]:**
1. **Scan targets, no timing.** A field that *wants* a scan (a serial grid cell, the stock-take box, the label "scan it back" box, the item name field of a voucher line) simply takes the characters and finishes on the terminator. This is how the item field already behaves for a code typed into the alias: the popup filters by substring and Enter picks the first match [C `VoucherScreen.tsx:2862-2867`, `:3211-3223`]. What it lacks is an **exact** lookup and a message for no match.
2. **The scan target model.** Only a scan target takes a scan. Anywhere else a burst is swallowed and reported ("Not here: scan into an item field"), instead of firing hotkeys, answering Accept or appending to a field. The detector that does this sits **at the keydown layer before the handler stack**, in the same place as `installAltReveal`, which already observes every key in the capture phase without consuming [C `src/keys.ts:303-330`]; the Phase 27 key translation is another layer at that point [C `keys.ts:11-34`, `:89`]. It cannot retract the first character that has already reached a handler, so a bare-letter screen needs either a **scan-ready mode** (bare letters inert while it is on) or a short hold of the first key (a latency tax on hotkeys); both are opt-in [J].
3. **Opt-in, asserted to change nothing when unset.** The Phase 27 rule applies word for word: customisation is opt-in and "nothing chosen" is the app as it was, **asserted by identity** (the dispatcher hands every handler the same event object when no table exists) [C CLAUDE.md, Phase 27 gotchas]. The scan layer is therefore off until a person turns on *I use a barcode scanner* on that device, stored like the other per-device choices (for example the personal-settings table `TW_UI_PERSONAL`, keyed by device and operator), and the suite proves that with it off the event object is unchanged. Chords it adds go in `core/shortcutRegistry.ts` and respect the reserved list.

**What to build first** is tier 1 plus the exact-lookup resolver and the unknown-code message, which fix the silent failure today with no key-layer change at all; the key layer follows once there is a scanner to test against [J].

### 5.4 Camera scanning

| Piece | Fact | Source |
|---|---|---|
| **Browser API** | the Barcode Detection API detects Aztec, Code 128, Code 39, Code 93, Codabar, Data Matrix, EAN-13, EAN-8, ITF, PDF417, QR and UPC-A and E in images and video frames; secure contexts only; MDN marks it experimental with limited availability; `getSupportedFormats()` says what this browser can do | [W mdn-barcode] |
| **Support** | Chrome and Edge partial from 83, Opera partial, Samsung Internet from 13, Firefox none, **Safari disabled by default** from 17; the page gave Chrome for Android only as "156", which I do not trust as the whole story | [W caniuse-barcode] |
| **WebAssembly decoder (ponyfill)** | `barcode-detector` (MIT) wraps ZXing-C++ compiled to WebAssembly behind the same API, offers group names such as `gs1_codes`, and **fetches its `.wasm` from a CDN by default** (it can be pointed at a local path for offline use or a content-security policy) | [W barcode-detector] |
| **Size** | `zxing-wasm` 2.2.0: reader wasm 943,898 bytes, full 1,379,928, writer 616,975; the reader JS about 35 KB | [W jsdelivr-zxingwasm] |
| **ZXing-js and html5-qrcode** | both Apache-2.0 and both in **maintenance mode**; ZXing-js's page recommends a commercial library for teams wanting active maintenance; html5-qrcode's author says pull requests will not be merged for now | [W zxing-js; W html5-qrcode] |
| **The deployment** | the live site is **HTTP only**, so a browser will not give it the camera or this API [C `docs/HANDOFF.md:63`, `:90`]; the mobile research reached the same conclusion [R `mobile-personas-and-jobs.md:112`] | |
| **Electron** | the packaged app loads `file://` [C `electron/main.cjs:214`], which Chromium should count as a secure context (a sibling note says so [Ws]; **test `window.isSecureContext` in the packaged build**); `main.cjs` installs no permission handler [C grep of `electron/main.cjs`], Electron's page does not say what the default is when none is set, and a Mac build needs the camera usage text in `Info.plist` and a call to ask for access [W electron-session; W electron-systemprefs]; the build config has no `extendInfo` [C `package.json` `build.mac`] | |

**When the camera is the right tool [J].** Occasional work on a phone or tablet: a stock-take walk, a price or stock lookup, receiving in a godown, reading a QR from a pharma strip. **Not** a counter doing hundreds of bills a day: a handheld wedge scanner reads in a fraction of a second without aiming, focus or light, and costs a few thousand rupees. The camera path should be built for the phone shell (mobile track) after TLS, behind one `scan()` interface whose other implementation is the wedge target, so a screen asks for "a scan" and does not care where it came from.

### 5.5 The scan-driven workflows that matter

| # | Workflow | What a scan does | Rules |
|---|---|---|---|
| 1 | **Scan to add a voucher line** | resolves to an item, a batch, a variant or a unit and adds a line with the item's rate; a repeat scan of the same non-serial item **adds one to that line's quantity** | Odoo's adjustments and transfers raise the count on each scan [Ws odoo-adjust]; a repeat of a **serial or IMEI never increments**, it is refused as a duplicate; a barcode that fits several live batches opens the batch picker with the proposed batch selected (sibling 8.2); Enter after a scan stays in the item field so the next scan lands in a new line; Busy can read a quantity out of a barcode on a physical stock voucher [Ws busy-physical] |
| 2 | **Scan into the serial or IMEI grid** | one scan, one row; the IMEI's check digit and length are tested; duplicates inside the grid, in stock, and already sold are refused with the reason | quantity equals the serial count; a dual-SIM phone takes two IMEIs per unit (sibling 3.1) |
| 3 | **Scan to find** | opens the item's stock query, history or vouchers | the palette already finds an item by alias or part number [C `src/ui/modern/CommandPalette.tsx:425`] and Stock Query's picker matches the same fields [C `src/ui/inventory/stockquery.tsx:43`], so a code typed into the alias already works there; a price-check mode that shows price, stock and batches without a voucher is a small screen |
| 4 | **Stock-take (Physical Stock) by scanning** | location scan, then item or unit scans raise counts; a **difference list** against the books at the end | none of the Indian products I read does this (Busy gives advice only [R `batch-serial-tracking.md` 11.2]); Odoo's adjustment screen raises counts per scan [Ws odoo-adjust]; the engine counts per (item, godown) only today and the custom-fields slice adds a `countPart` for counts made in parts [R `batch-serial-code-audit.md` 3.2; C `.wt-cf/docs/design/custom-fields-spec.md` AD.3.3] |
| 5 | **Receive against a purchase** | a GS1 or maker code fills item, batch, expiry and serial; a shop code fills the item | the batch and expiry still pass the batch rules; an unknown code offers to create the item with the code attached |
| 6 | **Unknown code to a new item** | opens the stock item form with the code pre-filled as a registry entry | Odoo can look an unknown product up in a barcode database and create it [W odoo-software]; Busy's FAQ says a scan can only find an existing item and cannot create one [W busy-faq5]; for India a public lookup is a privacy and accuracy question, so the first version is "create here with the code" [J] |
| 7 | **Picking and packing check** | each scanned item ticks a line of the Delivery Note or order; a surplus or a wrong item is flagged | there is no link from a Delivery Note to its invoice today [R `batch-serial-tracking.md` 9.1 row 7], so the check is against the note |
| 8 | **Returns and exchanges** | scanning a unit shows its sale, buyer and warranty and refuses a unit never sold | the sibling's unit rules (`batch-serial-tracking.md` section 7.1) |
| 9 | **Godown transfer or Stock Journal** | scanned rows fill the lines of the source godown | the transfer must carry the batch (sibling constraint 1) |
| 10 | **Reprint** | scanning an old label opens the label dialog for that unit or batch | the log of 4.5 |

### 5.6 Error handling

| Case | Behaviour | Message to the person |
|---|---|---|
| **Unknown code** (no registry, alias or part-number hit) | nothing is added and **nothing is appended to the field**; the field clears and keeps focus | "Code 8901234567890 is not in your items. Create it (key), or search" (today it is silent and the next scan appends [C `batch-serial-code-audit.md` 3.3]) |
| **Same unit scanned twice** | refused; the first row is highlighted | "This IMEI is already on line 3" |
| **Same item, not serial** | quantity +1 on the existing line (or a new line, a per-user choice) | the line flashes |
| **Wrong EAN check digit** | a scanner does not send one, so this is a typed or pasted code: soft warning, kept | "That code's check digit is wrong (should end in 0)" |
| **IMEI fails Luhn or length** | warn and allow only with a reason | "Not a valid IMEI (check digit)" |
| **Mixed symbologies in one field** | resolved by the order in sibling 8.2; the resolved kind is shown as a chip: unit, batch, variant, item, scale, GS1 | "Read as: GS1 batch B2410A, expiry 31-Mar-2027" |
| **GS1 parse trouble** | an unknown identifier is rejected with the raw text shown; a variable-length field followed by another with no separator is read by the shortest rule and flagged "no separator found" | "Could not split this code; showing what was read" |
| **Partial read** (under the minimum length, or no terminator within about 100 to 150 ms [W onscan]) | discarded with a short tone | none, or "incomplete scan" |
| **Burst off-target** | swallowed | "Scan into an item field" |
| **Burst while the Accept bar is up** | ignored; it never answers Yes | none |
| **One code on two items** | a picker with both; a Data Check row lists duplicate codes | "This code belongs to two items" |
| **Expired or held batch scanned** | the sibling's expiry rules apply at the line, not at the scan | per sibling R11 and R12 |

### 5.7 How the competitors handle scanning

| Product | Scanning | Weak spot (from the pages read) | Source |
|---|---|---|---|
| **TallyPrime** | the cursor is on the Name of Item field; scan; the item whose **Alias** matches is chosen; scan several one at a time, then Ctrl+A; the alias holds the code (a 16-digit number according to the help page) | one code per item, no batches or units, no setup help | [W tally-help] |
| **Busy** | per POS voucher series, a "barcode scan" option for picking items and a "parameterized BCN" option so items can be scanned directly; connect the scanner by USB or Bluetooth; serial numbers handled by the series; the FAQ's first advice is **"test the scanner in Excel first"** (that is, the wedge); a quantity can be read out of a barcode on a physical stock voucher | barcodes find items and cannot create them; no scanner diagnostic; stock-take by scanning is advice only [R `batch-serial-tracking.md` 11.2] | [W busy-faq3; W busy-faq5; Ws busy-physical] |
| **Marg ERP** | "Scan & Bill" for strips and bottles; scanning batch numbers directly is claimed | marketing-level claims, nothing verified | [Ws marg-search] |
| **Vyapar** | barcode scanning at billing; generation is plan-gated per the sibling research | | [R `batch-serial-tracking.md` 8.3] |
| **Zoho Inventory** | a serial scanned into its field | cap on bulk, limited labels | [R `batch-serial-tracking.md` 8.3] |
| **Odoo Barcode** | a keyboard-wedge design that needs the scanner's layout to match and, on Zebra mobile devices, "send characters as events"; repeat scans raise the count; a lot or serial is scanned after the product; GS1 nomenclature with one rule per application identifier, where a variable-length field not last needs the GS character (`\x1D`) in the pattern; an unknown product can be looked up and created | the mobile app has drawn complaints that its scanner does not scan [R `mobile-market.md:175`]; GS1 needs developer mode and a rule per field; no Indian-pack handling | [W odoo-troubleshooting; W odoo-gs1-nomenclature; Ws odoo-adjust; W odoo-software] |

**Where they are all weak [J]:** none offers a **scanner setup and diagnostic screen**; none that I read validates an IMEI at scan; none combines a unit scan with a duplicate check across devices; and none that I could confirm turns a pharma strip's QR into a filled-in receipt line.

---

## 6. The repository today: what exists, what a scan does, and what a build must respect

All greps were run from the repo root on 2026-10-23 over `src` and `electron` (`*.ts`, `*.tsx`, `*.cjs`, `*.mjs`, `*.js`, `*.css`), case-insensitive. Line numbers are of the tree at HEAD `7074445` with the working tree as it stood (the tracked changes in the working tree at the time were `.claude/settings.local.json` and the sibling research file `batch-serial-tracking.md`).

### 6.1 What does not exist (each a grep with the count it returned)

| Looked for | Matches |
|---|---|
| `barcode`, `gtin`, `imei`, `code128`, `ean13`, `upc-a` | **0** each: no field, type, encoder or word anywhere in `src` or `electron`; every earlier hit of "QR code" is the e-invoice or the UPI code |
| `BarcodeDetector`, `getUserMedia` | **0** |
| `navigator.usb`, `navigator.serial`, `navigator.hid`, `navigator.bluetooth` | **0** |
| `getPrintersAsync`, `getPrinters`, `deviceName`, `silent:` | **0**: the Electron bridge cannot list a printer or print silently |
| `setPermissionRequestHandler`, `setDevicePermissionHandler` | **0**: no permission handling in `electron/main.cjs` |
| `kiosk` | **0** |
| a label-size page, a sheet grid, a thermal-receipt width | none: print pages are A4, A5 and Letter [C `src/core/printConfig.ts:32`; `src/styles/print.css:34-43`; bridge list `electron/exportBridge.cjs:24`, `electron/preload.cjs:36`] |
| a barcode or print library | none: `package.json` lists `dexie`, `react`, `react-dom`, `zustand` and two Plex font packages as dependencies [C `package.json`] |

### 6.2 What exists that labels and scanning build on

| Piece | Where | Why it matters |
|---|---|---|
| **A pure QR encoder with an independent decoder test** | `src/core/qr.ts` (versions 1 to 40, byte mode, levels L and M, its tables checked against the standard's capacities; an independently written decoder reads a symbol back out of the matrix; Apple's QR detector read the symbols when it was written) [C `qr.ts:1-12`, `:30-44`]; drawn by `QrSvg` as one SVG path, black on white, quiet zone included, sized in millimetres [C `src/ui/print/QrCode.tsx:1-33`] | the pattern to copy for every other symbology; a Digital Link QR already works |
| **UPI QR on the invoice** | `src/ui/print/PrintVoucher.tsx:257-264` (26 mm, level M) | proves a small QR prints from the print host |
| **A millimetre-exact custom page** | `PrintLeaf` injects `@page { size: Wmm Hmm; margin: 0 }` only while printing [C `src/ui/banking/chequeLeaf.tsx:164-173`] | the label page precedent |
| **A calibrated field editor and a test sheet** | `chequeLayoutEditor.tsx` (cells, dashed focus frame, "Print test sheet" at actual size) and the `ChequeLayout` type (`offsetX`, `offsetY`, per-field positions) [C `src/ui/banking/chequeLayoutEditor.tsx:200-260`; `src/types.ts:916-931`] | the label editor precedent |
| **The print host** | a DOM node outside `#root` that `window.print()` snapshots: vouchers [C `src/ui/print/PrintVoucher.tsx:608`, `:676`], bulk print [C `src/ui/print/BulkPrint.tsx:128-131`], reports [C `src/ui/reportTools.ts:275`, `:303`]; background graphics are off by default and a filled area vanishes [C `print.css:17-19`] | a label host is one more host |
| **Fonts before print** | `ensurePrintFonts()` loads IBM Plex first so the rupee sign prints [C `src/ui/print/printFonts.ts:1-25`] | the driver path already draws the rupee sign; a raw text path would not |
| **A PDF bridge and a share bridge, written for an untrusted renderer** | `electron/exportBridge.cjs`, wired in `main.cjs:144-155`, exposed by `preload.cjs:30-41`, tested by `scripts/assert/exportbridge.assert.ts` against a fake `electron` | the pattern a label bridge copies (section 2.6) |
| **Bulk print with a ceiling and a refuse-whole rule** | `src/core/bulkPrint.ts:10-14` (`MAX_BULK = 100`), `src/ui/reports/selectionBar.tsx:167` | the bulk-by-selection precedent |
| **A print-after-save hook** | `VoucherType.printAfterSave` [C `src/types.ts:543`]; `VoucherScreen.tsx:2816`, `:2826-2829` | the shape of a "print labels after Accept" flag |
| **A text-safe spreadsheet writer** | `src/core/xlsx.ts` writes **every string as a shared string**, whatever it starts with, so a 13-digit EAN or a 15-digit IMEI exported as text stays text [C `xlsx.ts:11`]; `csvOut.ts` defuses `= + - @` but a CSV of long digit strings still opens as numbers in a spreadsheet [C `src/core/csvOut.ts:1-40`] | label data can round-trip through Excel, and CSV needs a text-typing note |

### 6.3 Where a barcode lives today, and what finds it

- `StockItem.alias` and `StockItem.partNo` are two optional strings, "searchable in every picker" [C `src/types.ts:491-493`]; the stock item form has them as `Alias` and `Part no.` [C `src/ui/inventory/masters.tsx:377-378`]. The Dexie store for stock items is `'id, cid'`: **neither is indexed** [C `src/data/db.ts:129`, `:214`, `:238`], so any lookup is over the in-memory array; an exact barcode lookup should use a `Map` built once per change of items or registry [J].
- **Who reads them:** the voucher item popup filters by substring on name, alias and part number [C `src/ui/VoucherScreen.tsx:2862-2868`]; the master pickers and Stock Query use `PopupItem.alt` (alias and part number joined) [C `src/ui/widgets.tsx:356-362`; `src/ui/masters.tsx:859`; `src/ui/inventory/stockquery.tsx:43`]; the palette indexes both as aliases (prefix match) [C `src/ui/modern/CommandPalette.tsx:425`]; **Bulk Entry resolves a row's item by exact normalised name, then alias or part number, first one wins and a later duplicate is silently ignored** [C `src/ui/admin/BulkEntry.tsx:269-276`]; the Tally XML import and export carry `PARTNUMBER` [C `src/core/tallyImport.ts:972-973`; `src/core/tallyExport.ts:484`].
- **What that means for scanning:** one code per item, a substring match (so a partial or wrong-digit scan can match nothing or something), a silent winner when two items share a code, no pack-size or batch or unit code, and no way to say "this code is a pack of 12".

### 6.4 What a wedge scanner's burst does today

1. **One `keydown` listener on `window`** runs a last-in-first-out stack of handlers; a handler returns true to consume, and printable keys are not consumed by default, so they reach the focused field [C `src/keys.ts:11-34`]. A person's own key table is applied first when one exists, by a Proxy over the real event [C `keys.ts:44-125`].
2. **In the voucher item name field** the characters become the text, the popup filters on them, and Enter calls `pop.pick(popSel)`, which picks the first match [C `VoucherScreen.tsx:3211-3223`]; with no match `pickItem` returns on an empty list and nothing says so [C `:2047-2049`]; the **next scan appends to the leftover text** [R `batch-serial-code-audit.md` 3.3].
3. **In quantity, rate or discount** the text is read as an arithmetic expression and a "= value" hint is drawn on every keystroke; a 13-digit code is a 13-digit quantity [R `batch-serial-code-audit.md` 3.14; C `VoucherScreen.tsx:148-154`].
4. **In Narration** Space, Tab and Enter first try a snippet expansion and Enter then **accepts the voucher** [C `VoucherScreen.tsx:3361-3367`].
5. **On the Accept bar** Y, N and Enter answer it [C `src/ui/widgets.tsx:470-480`]; **on a list or picker** a printable key extends the filter [C `widgets.tsx:426`].
5a. **In the Ctrl+K palette** the search input is focused, a typed or scanned code filters the item rows (alias and part number are indexed as prefix aliases) and Enter opens the first row, Item Vouchers; a miss shows "No matches for ..." in the list [C `src/ui/modern/CommandPalette.tsx:425`, `:547-554`, `:569-575`]. It is the one place today where an unknown code is **reported**, and it works for a scan only after the palette is opened.
6. **On a screen with no input** (a report, a menu) letters are hotkeys and Enter activates the highlighted row [C `keys.ts:11-34`; Gateway and report key tables].
7. **Where an input has focus the bare-letter global bindings stand down**: the `?` sheet and the G chord are dead inside inputs (`isTypingTarget`) [C `keys.ts:435-439`], which is why typing a narration does not navigate and why a scan **into an input** is safe from them.
8. **Alt keys:** a lone Alt press starts a 300 ms "reveal" timer and any other key cancels it [C `keys.ts:279`, `:303-330`]; Alt plus a keypad digit reads as `Alt+<digit>` [C `keys.ts:206-222`], the keypad-emulation hazard of 5.2.
9. **Not measured:** how many keystrokes per frame the voucher screen takes at the scanner's pace; the sibling audit asks for a synthetic-key burst test as a gate [R `batch-serial-code-audit.md` 3.14].

### 6.5 What the in-flight work imposes

| Work in flight | What it means for labels and scanning | Source |
|---|---|---|
| **Custom fields and CF-9** (stock by a line-level "tracks stock" variant key beside godown and batch) | per-variant barcodes and SKUs are **explicitly not in** the custom-fields slices ("they need their own slices if asked"), so the registry owns them; a variant's identity is an **option id**, so a label's code survives a rename; the Tab contract on the item grid is being changed (a Tab branch behind `lineFieldCount > 0`), so any scan terminator of Tab must be tested against it; counts made in parts get a `countPart` flag the stock-take-by-scan must reuse | [C `.wt-cf/docs/design/custom-fields-spec.md:571`, `:582`, `:992`; AD.3.3] |
| **The Back control** | every new dialog (label dialog, scanner setup, queue) is a `Box` with `onClose` and a visible Close, and registers a dirty probe while it holds an edited queue; an Accept in flight blocks Back | [C `.wt-cf/docs/design/back-nav-spec.md` B5 (line 27), B12 (line 34)] |
| **Batch and serial design (siblings)** | the registry, the resolution order, the unit and batch rules and the 13 constraints are theirs; labels and scanning consume them | `batch-serial-tracking.md` sections 8 and 9; `tracking-verticals-india.md` 6.5 and 6.6 |
| **POS research** | a POS needs the burst captured out of the per-keystroke React path (finding 5) and a thermal receipt path; the label bridge and the receipt bridge are the same bridge | [R `pos-code-audit.md` 7 and 8] |
| **Mobile research** | camera scanning and Bluetooth printing are Chromium-on-Android features, and a thin native shell is justified only if a pilot demands them; TLS first | [R `mobile-personas-and-jobs.md:112`, `:757`, `:803-804`] |

### 6.6 Project rules a build of this must follow (from CLAUDE.md)

- **New screens** go through the new-screen checklist: the `Screen` union, `router.tsx` route and crumb, `MenuScreen.tsx`, the palette and `overlays.tsx` Go To, a `SCREEN_AREA` entry (the area is what the screen *writes*), a `state/dirty.ts` probe if it holds typed work, a row in the Reports Center catalogue for a report (a label log or a scan reconciliation), and `ui/quickCreate.ts` for any create action [C CLAUDE.md, Conventions].
- **New master fields go last** on the stock item form or in an Alt-key grid, never before Opening Balance, and the whole Enter walk is asserted with the new keys removed [C CLAUDE.md, Phase 24 gotchas]. A `barcode` entry is a registry, not another form field.
- **Customisation is opt-in and asserted to change nothing when unset** (the scan layer, a label default, any setting) [C CLAUDE.md, Phase 27 gotchas].
- **Defaults are frozen**: no existing key's meaning changes; a new chord is a row in `core/shortcutRegistry.ts` before it ships [C CLAUDE.md, Conventions].
- **An Electron IPC takes a name and options, never a path or a free command string** [C CLAUDE.md, Phase 29 gotchas; `exportBridge.cjs:1-20`].
- **Never invent a number a regulator issues** (an IRN, an e-way bill number; here a HUID) [C CLAUDE.md, Phase 29 gotchas].
- **A source-scanning assertion needs a guard that it scanned something**, and a **perf mirror is a copy of source text** that goes stale (a scan or label perf row must be re-pointed when code moves) [C CLAUDE.md, Phase 28 and 29 gotchas].
- **Do not reword a screen as if something happened that did not**: "sent to the printer", never "printed" [C CLAUDE.md, Phase 29: "Share means open, never send"].
- **Cap length before any pattern test and scan linearly**: a GS1 parser and a pasted code both take untrusted text [C CLAUDE.md, Phase 24 gotchas].

---

## 7. Recommendations [J]

### 7.1 On one page

1. **Build labels as data plus one layout engine**, with a small library of fixed templates a person adjusts field by field (size, fields, offset), a live preview that is the same drawing that prints, a test label and a "scan it back" check. No free drag-and-drop designer in the first release.
2. **Write the 1-D encoders yourselves** (EAN/UPC, Code 128, Code 39, ITF-14, the GTIN builder and the check-digit functions), in `src/core`, asserted against an independent oracle (bwip-js as a test-only dependency, macOS Vision as a manual recipe); keep `qr.ts`; decide DataMatrix when the GS1 slice starts. No runtime barcode dependency now.
3. **Ship the browser print path first** (A4 sheet grid, and a roll label through an injected `@page` size), because it works on the HTTP site and in Electron with no hardware code. Say plainly in the dialog what the driver may still do.
4. **Fix the silent scan failure before building a detector**: an exact-lookup resolver over a registry (the sibling design), the unknown-code message, and a scan-target model. Then add an **opt-in** key-layer scan detector and a **scanner setup screen** that shows what actually arrived.
5. **In Electron, add a label bridge in two steps**: (a) printer list and silent print with a chosen printer and page size, (b) raw TSPL and ZPL over TCP 9100 and the Windows spooler, with a printer profile, a bitmap path for rupee and Devanagari text, and the TSPL status byte. Copy the export bridge's rules: name and options never a path or a free command string, a command allow-list, one job at a time, "sent to the printer" not "printed".
6. **Treat statutory pack labels as the owner's responsibility and ship them last**: shelf, unit, batch, hang-tag, jewellery and dispense labels first; the declaration template for a packer or importer afterwards, with locked wording and warnings, and never the word "compliant".
7. **Do not build** a public barcode lookup, a free designer, a multi-printer fleet manager, scale integration over a serial cable, GS1 minting for shops, or camera scanning on the HTTP site.
8. **Hardware first.** The first thing to do with any of this is to put the owner's own scanner and printer through section 7.5. A desk study cannot settle layout, driver, ribbon or Bluetooth behaviour.

### 7.2 Slices, in order

Sizes are my guess at relative effort (S under a week of one builder, M about two, L more) against this repository's pace [J]; the dependency column is what must exist first.

| Slice | What | Files (new unless named) | Needs | Size | Runs on |
|---|---|---|---|---|---|
| **LB-0** | pure encoders and check digits: EAN-8/13, UPC-A/E, Code 128 (sets A, B, C), Code 39, ITF-14; GTIN builder; IMEI Luhn; scale-profile parse; GS1 element-string build and parse (no 2-D); module array to SVG path and to a dot bitmap; the suite with bwip-js as oracle, plus a mutation list | `src/core/barcode/*.ts`, `scripts/assert/barcode.assert.ts` | none | S to M | all |
| **LB-1** | label template type, the library of section 4.4, the layout engine, the SVG and bitmap renderers, text fitting | `src/core/labelLayout.ts`, `src/core/labelTemplates.ts`, `src/ui/labels/render.tsx` | LB-0 | M | all |
| **LB-2** | the printing UI: sheet grid and roll dialogs, injected `@page`, label host, test label, per-profile offset and start cell, the label log, entry points (after Accept flag, batch and serial lists, selection), the new-screen checklist items | `src/ui/labels/*`, a flag on `VoucherType`, `print.css` | LB-1 and the batch/serial lists | M to L | all (browser dialog) |
| **LB-3** | the registry resolver and the unknown-code message; a scan target for the item field, the serial cell, the stock-take box | `src/core/scanResolve.ts`; edits to `VoucherScreen.tsx` (single writer) | the sibling registry | M | all |
| **LB-4** | the opt-in key-layer scan detector, the scan-target model, the scanner setup screen (layout mode, terminator, GS handling, learned timing, Alt-keypad detection), the burst test as a gate | `src/keys.ts` (single writer), `src/ui/settings/*` | LB-3 | M | all |
| **LB-5** | the Electron label bridge: printer list, silent print, TCP 9100, spooler RAW helper, TSPL status poll, the allow-list validator and its mirror, fake-electron tests, **a real printer test** | `electron/labelBridge.cjs`, `main.cjs`, `preload.cjs` | LB-2 | M to L | Electron |
| **LB-6** | GS1 and Data Matrix: DataMatrix encoder (or a lazy bwip-js chunk), GS1-128, the pharma scan-to-fill (item, batch, expiry, serial), the maker-QR string handling for the top-300 rule | `src/core/barcode/datamatrix.ts` or a chunk, `src/core/gs1.ts` | LB-3 and the batch/serial engine | M to L | all |
| **LB-7** | camera scan and the web raw path: `scan()` interface with a camera implementation (the WebAssembly ponyfill, self-hosted), Web Serial and WebUSB printing | `src/ui/scan/camera.tsx` | **TLS on the site**, the phone shell decision | M | Chromium on HTTPS, phones |
| **LB-8** | a free designer | | pilots that ask for it | L | all |

LB-0 and LB-1 are new files with no collision risk and can start in parallel with anything. `VoucherScreen.tsx` and `keys.ts` are single-writer files (the sibling audit's list), so LB-3 and LB-4 queue behind whoever holds them. No slice needs a backend deploy to work; storing templates as a new table touches the server only through the `DATA_TABLES` list used for conflict reports (4.5).

### 7.3 What is feasible where (the short form of section 2.5)

- **Web build on today's HTTP site:** fixed labels on A4 sheets and rolls through the print dialog; wedge scanning; the scan-target model and exact lookup. **Not:** silent print, raw printing, camera, Web Serial or WebUSB.
- **Web build after TLS:** adds the camera (with the WebAssembly ponyfill where `BarcodeDetector` is missing) and, on desktop Chromium only, raw printing over Web Serial or WebUSB with a prompt per device. Safari has none of those and Firefox has Web Serial only.
- **Electron (Windows and Mac):** everything in the first bullet plus **silent print to a named printer, TCP and spooler raw printing, TSPL status, serial scanners**; the desktop app is where the label work becomes pleasant and where the highest value per effort is (it owns the keys, the printer and the permissions).
- **Android (browser or PWA):** wedge scanning with a Bluetooth scanner, camera scanning after TLS, printing through the print sheet or a vendor app or Web Bluetooth to a Low Energy printer; a native shell is the honest answer for Classic Bluetooth and TSPL.
- **iOS:** camera scanning (WebAssembly decoder), AirPrint or a vendor app for printing; no raw printing from a web page.

### 7.4 What I would not build first, and why

- **A public barcode lookup:** an accuracy and privacy question with no Indian source I could confirm; "create here with the code" covers the need.
- **A free designer:** its cost is that of a product; the fixed library plus field nudging covers the shops I can imagine, and an Excel export covers the shops that outgrow it.
- **GS1 minting for shops:** only GS1 India issues 890 numbers; a shop's own numbers use the company block. A GTIN builder for a manufacturer who holds a prefix is a small, separate feature.
- **Scale integration over RS-232:** no Indian scale protocol was found; a **scale profile** that reads the labels the scale already prints covers the grocery case.
- **Statutory pack labels in the first release:** the only labels with legal exposure; they need the owner's own legal reading (section 3) and a pilot with a packer.

### 7.5 What the owner should test with real hardware (before a build is sized)

Bring the scanner and printer a typical customer owns, a phone, and these stocks: 50 x 25, 38 x 25 and 100 x 50 rolls, an A4 40-up sheet, and a jewellery tag.
1. **Scanner self-test outside the app** (a plain text editor): scan an EAN-13, a Code 128 with capitals, lower case and a hyphen, and (if you have one) a pharma GS1 DataMatrix. Note: the terminator (Enter or Tab), any prefix, whether capitals survive, whether the group separator arrives and as what, the speed. Repeat with the computer's input language set to Hindi.
2. **The same scans inside the app** at: the voucher item field with a code stored in Alias, Narration, the Accept bar, a report, and Go To. Write down every surprise (this is the table of 6.4 made real).
3. **The scanner's modes:** switch it to serial (virtual COM) and note whether the separator now arrives; check whether "keypad emulation" is on (the characters appear slowly with Alt flicker) and turn it off.
4. **A calibration label through the browser dialog**: the ruler grid, an EAN-13 and a Code 128 at two and three dots, a QR and a Data Matrix, on each stock. Record margins, the driver's paper-size list, whether `@page` size was honoured, any scaling, and **scan every printed code** with the shop's scanner and a phone.
5. **The same through Electron's silent print**, then **raw TSPL or ZPL** to the same printer, then a **bitmap** label with the rupee sign and a Devanagari name. Compare bars under a loupe or by scan success at the smallest module.
6. **Out of paper, head open, ribbon out**: does the printer report it (TSPL's status byte), and what does a driver path say?
7. **A Bluetooth printer and a phone**: which route works (the maker's app, a print service, Web Bluetooth) and what survives a lost connection.
8. **A real batch of 200 labels**: time, jams, skipped labels, alignment drift across the roll.
Five shops per business type (mobile, pharmacy, garments, grocery, jewellery) is the sample the sibling research asks for; a shop that runs a label printer today has its own templates, and those, photographed, are the best requirements document [R `tracking-verticals-india.md` 8.2].

### 7.6 Decisions for the owner, each with the default I would take

| # | Decision | Default | Why |
|---|---|---|---|
| 1 | Which printers and scanners do your customers already own? | ask ten; assume 203 dpi TSC, TVS and Zebra desk printers and keyboard-wedge scanners | the language, the size set and the first printer test follow |
| 2 | Raw printing in Electron: now or after the driver path? | driver path first, raw TSPL second | the driver path needs no profile and no allow-list; raw gives status and speed |
| 3 | Where do label templates live? | a new synced table (an additive Dexie bump) | a field on the company document brings the older-client protection duty of Phase 27; a new table also needs the backend's `DATA_TABLES` list extended |
| 4 | Mint internal EAN-13 numbers for shop items? | yes, in 040-049, with a collision check; never 890 | 890 is GS1 India's; the range is for in-company use |
| 5 | Statutory pack labels in v1? | no; after the first release and a packer pilot | legal exposure; the owner's own reading of section 3 |
| 6 | A free designer? | no | cost of a product; fixed templates and nudging cover the shops I can imagine |
| 7 | A public lookup for unknown barcodes? | no | accuracy and privacy; "create here" covers it |
| 8 | Camera scanning on the HTTP site? | not possible; plan it with TLS and the phone shell | a browser will not allow it |
| 9 | Is hardware (printers, labels, scanners) a channel? | an owner's business question, not researched here | the owner works at a Tally reseller; a certified printer list could be a selling point |

---

## 8. What I could not verify (read this before relying on any claim)

1. **No hardware.** No scanner, label printer or phone was available. Everything about how a wedge scanner behaves, how a driver resamples bars, how a printer reports status, and how Bluetooth pairs is from a manual, a vendor page or a forum summary. Beyond arithmetic and text extraction, the only thing I *ran* was the encoder experiment (section 1.9), which proves the encodings are valid, not that any printer or scanner will cope.
2. **GS1's own documents.** The General Specifications, the GS1 DataMatrix guideline and GS1's company-prefix page answered 403 or were too large to fetch. GS1 facts come from GS1 India, GS1 Sweden's guides, GS1's application-identifier dataset (read for the identifiers in 1.3 only), Wikipedia, independent libraries' documentation and search summaries. In particular the meaning of a `00` day in AI 17, the GS1 rules for what to do when a separator is missing, the exact DataMatrix capacity table and GS1's own scale-label statements are **not** read at source [unverified].
3. **Indian scale and printer facts.** No Indian weighing-scale maker's label layout was found; the language support of the TVS LP 46 Neo and the Godex G500 was not confirmed; TSC TE244 emulations were not confirmed; prices are listings of unknown date.
4. **The statutory texts, and their currency.** The Legal Metrology rules were read in the Consumer Affairs consolidated book, which runs to the 2022 amendments; the text extraction of its amended Table I of letter heights was scrambled and I give no figures; a 2023 amendment mentioned by search summaries was not read. FSSAI was read in the Version V compendium of 13 October 2022 (later amendments exist and were not read). The Drugs Rules were read in CDSCO's consolidated file (September 2024) for Rules 94 to 97; I found no price clause in Rule 96 and did not look for MRP rules for medicines elsewhere. The BIS FAQ page I fetched did not mention the HUID, and a search summary said it does; I relied on BIS's blog. **None of this is legal advice and no lawyer or Legal Metrology officer was consulted.**
5. **Printing behaviour of Chromium and Electron.** I did not run Electron 31 or Chrome against a printer. Whether `webContents.print` honours a CSS `@page` size, the units of its `pageSize` object, how drivers treat custom sizes, whether SVG fills print with background graphics off, and what the kiosk-printing flag does today are from documentation or anecdote [unverified].
6. **Browser support figures.** The caniuse pages as summarised gave Chrome for Android a bare "156" for WebUSB, Web Bluetooth, Web Serial and `BarcodeDetector`, which looks like an artefact of the summary; read the tables directly. `file://` being a secure context in Electron is from a sibling note, not tested.
7. **Competitors.** No live Busy, Marg, Vyapar, Zoho or Odoo. Marg's help article was read for its first step only (a 630,000-character page). What each product "cannot do" is "not found in the pages I read", which is weaker than "cannot".
8. **Scanner-vendor facts** come from Zebra's programming-reference menus and parameter pages and from third-party notes (Odoo, onScan.js, a Russian integrator's article, library documentation); I could not read a Honeywell, Datalogic or Indian-brand manual, and the keyboard-layout, Caps Lock and input-language behaviours of an Indian shop's PC are **untested hypotheses**.
9. **The voucher screen's burst behaviour is read, not measured**: no keystroke-rate test was run (the dev server is not mine).
10. **Costs and sizes** in 7.2 are my relative guesses; the library sizes in 1.8 are file-listing sizes of specific versions, not bundle measurements after tree-shaking.
11. **The sibling documents** (`batch-serial-tracking.md`, `batch-serial-code-audit.md`, `tracking-verticals-india.md`, `pos-code-audit.md`, `mobile-*.md`) were read and are cited, not re-verified; where I disagree or add, the text says so. The custom-fields spec in `.wt-cf` is a draft that its author marks unreviewed and was still changing.

---

## 9. Sources

Tags: **[W]** a page fetched and read (in full unless the entry says otherwise); **[Ws]** a search summary only, the page was not read; **[R]** read in a sibling document of this workstream, not re-read by me; **[C]** the repository. Binary documents (PDF) were returned by the fetch tool as files, and I read them by extracting their text locally with a small macOS PDFKit program kept in the session scratchpad. Everything is paraphrased. Pages fetched on 2026-10-22 and 2026-10-23.

### 9.1 Symbologies and GS1

- `gs1-ai` https://ref.gs1.org/ai/GS1_Application_Identifiers.jsonld [W, the first 100,000 of 546,468 characters: AIs 00 to 310n]
- `wiki-gs1-codes` https://en.wikipedia.org/wiki/List_of_GS1_country_codes [W]
- `wiki-ean` https://en.wikipedia.org/wiki/International_Article_Number [W]
- `wiki-c128` https://en.wikipedia.org/wiki/Code_128 [W]; `wiki-c39` https://en.wikipedia.org/wiki/Code_39 [W]; `wiki-dm` https://en.wikipedia.org/wiki/Data_Matrix [W]
- `denso-qr` https://www.qrcode.com/en/about/standards.html [W]
- `gs1se-size` https://gs1.se/en/guides/how-to-guides/size-guide/ [W]; `gs1se-rcn` https://gs1.se/en/support/how-do-i-create-my-variable-weight-numbers/ [W] and https://gs1.se/en/guides/how-to-guides/barcode-label-items-of-varying-weight/ [W]
- `biip-rcn` https://biip.readthedocs.io/en/latest/reference/rcn/ [W]; `biip-sym` https://biip.readthedocs.io/latest/reference/symbology/ [W]
- `gs1india-home` https://www.gs1india.org/ [W]; `gs1india-ean13` https://www.gs1india.org/blog/ean-13-the-barcode-number [W]
- `gs1-prefix-search` a search for GS1 India and the 890 prefix, whose results included https://www.gs1india.org/blog/all-you-need-to-know-about-barcoding and https://www.gs1.org/standards/id-keys/company-prefix (the latter answered 403 when fetched) [Ws]
- `gs1-size-search` a search for EAN-13 dimensions (GS1 UK, GS1 Canada's EAN-UPC symbol reference PDF, which the fetch tool returned as unreadable binary, and a GS1 change notice) [Ws]
- `gs1-dm-fnc1` a search on GS1 DataMatrix FNC1 and the group separator, results including https://www.packagingdigest.com/coding/gs1-datamatrix-fnc1-versus-gs-variable-length-field-separator-character (403 when fetched) [Ws]
- `gs1-dm-table` a search summary of the GS1 DataMatrix guideline's capacity table, https://gs1.org/standards/gs1-datamatrix-guideline/25 (403 when fetched) [Ws]
- `gs1-digital-link` searches on GS1 Digital Link, results including https://www.unitag.io/blog/gs1-digital-link-vs-qr-code-what-brands-and-printers-need-to-know-before-sunrise-2027/ [Ws]
- `gs1parser` https://unpkg.com/gs1-barcode-parser-mod@1.0.7/README.md [W]
- `cleverence-gs` https://www.cleverence.ru/articles/markirovka-tovarov/-simvol-gs-i-fnc1-v-data-matrix-nastroyka-skanera-i-datakollektora-dlya-1s/ [W, in Russian]
- `scale-search` a search for Indian weighing-scale label formats (Digi, Essae) that found no format [Ws]

### 9.2 Libraries

- `bwip-js` https://github.com/metafloor/bwip-js [W]; `bwip-js-license` https://raw.githubusercontent.com/metafloor/bwip-js/master/LICENSE [W]; `jsdelivr-bwipjs` https://data.jsdelivr.com/v1/package/npm/bwip-js@4.11.4/flat [W]
- `jsbarcode` https://github.com/lindell/JsBarcode [W]; `jsdelivr-jsbarcode` https://data.jsdelivr.com/v1/package/npm/jsbarcode@3.11.6/flat [W]
- `node-qrcode` https://github.com/soldair/node-qrcode [W]
- `jsdelivr-zxingwasm` https://data.jsdelivr.com/v1/package/npm/zxing-wasm@2.2.0/flat [W]; `barcode-detector` https://github.com/Sec-ant/barcode-detector [W]
- `zxing-js` https://github.com/zxing-js/library [W]; `html5-qrcode` https://github.com/mebjas/html5-qrcode [W]
- `onscan` https://github.com/axenox/onscan.js [W]

### 9.3 Printers, printer languages and print paths

- `tspl` https://fs.tscprinters.com/system/files/31-0000001-00_tspl_tspl2_programming_3_0.pdf [W, TSC's TSPL/TSPL2 programming manual, 446 pages, copyright 2025; read from the extracted text]
- `zebra-by` https://docs.zebra.com/us/en/printers/software/zpl-pg/c-zpl-zpl-commands/r-zpl-by.html [W]; `zebra-pq` .../r-zpl-pq.html [W]; `zebra-ci` .../r-zpl-ci.html [W] (all three read from the second half of long pages); `zebra-barcode-list` https://docs.zebra.com/jp/ja/printers/mobile/zq610-620-630-plus-ug/r-zq6x0plus-specifications/r-zq6x0plus-zpl-font-and-bar-code-specifications-and-commands.html [W]; the `^BC` description and the ZPL guide's location from a search (https://docs.zebra.com/us/en/printers/software/zpl-pg/c-zpl-zpl-commands/r-zpl-bc.html) [Ws]
- `zebra-zd220` https://www.zebra.com/content/dam/zebra_dam/en/spec-sheets/zd220-spec-sheet-en-us.pdf [W]; `zd230` a search summary of reseller listings [Ws]
- `tvs-lp46` search results for the TVS LP 46 Neo (the maker's pages redirected to https://www.tvselectronics.in/ which carried no product detail) [Ws]; `tse244-listings` Indian listings for the TSC TE244 and TTP-244 Pro [Ws]; `godex-ez2250i` listings and a brochure for the Godex EZ-2250i [Ws]; `brother-ql` search results including https://download.brother.com/welcome/docp100278/cv_ql800_eng_raster_101.pdf [Ws]; `phomemo-price` a price tracker for the Phomemo M110 [Ws]
- `zebra-print-specs` a search summary of https://docs.zebra.com/us/en/printers/industrial/zt510-ug/specifications/printing-specifications.html [Ws]; `idautomation-dots` https://support.idautomation.com/Barcode-Components/Barcodes-are-different-sizes-when-printing-to-printers-with-different-resolutions/_146 [Ws]
- `electron-v31-webcontents` https://raw.githubusercontent.com/electron/electron/v31.4.0/docs/api/web-contents.md [W]; `electron-printer-info` https://raw.githubusercontent.com/electron/electron/v31.4.0/docs/api/structures/printer-info.md [W]; `electron-session` https://www.electronjs.org/docs/latest/api/session [W, the first 100,000 of 111,611 characters; the "latest" docs, not v31]; `electron-systemprefs` https://www.electronjs.org/docs/latest/api/system-preferences [W]
- `ms-rawprint` https://learn.microsoft.com/en-us/windows/win32/printdocs/sending-data-directly-to-a-printer [W]
- `node-printer` https://github.com/tojocky/node-printer [W]; `node-thermal-printer` https://github.com/Klemen1337/node-thermal-printer [W]
- `qz-docs` https://qz.io/docs/ [W] (its pricing page and the GitHub README did not state the licence)
- `mdn-page-size` https://developer.mozilla.org/en-US/docs/Web/CSS/@page/size [W, the first 100,000 characters]
- `chrome-label-print` searches on Chrome and custom label sizes (a veterinary-software help article, Mozilla bug 1209265, Chromium issue 40783463) [Ws]; `kiosk-printing` vendor help pages on Chrome's kiosk-printing flag (for example https://pikr.helpscoutdocs.com/article/58-disable-print-previews-of-shipping-labels-in-chrome) [Ws]
- `mdn-serial` https://developer.mozilla.org/en-US/docs/Web/API/Web_Serial_API [W]; `mdn-usb` https://developer.mozilla.org/en-US/docs/Web/API/WebUSB_API [W]; `mdn-bluetooth` https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API [W]; `mdn-barcode` https://developer.mozilla.org/en-US/docs/Web/API/Barcode_Detection_API [W]
- `caniuse-usb` https://caniuse.com/webusb [W]; `caniuse-serial` https://caniuse.com/web-serial [W]; `caniuse-bluetooth` https://caniuse.com/web-bluetooth [W]; `caniuse-barcode` https://caniuse.com/mdn-api_barcodedetector [W]; the Chrome developer pages https://developer.chrome.com/docs/capabilities/usb and https://developer.chrome.com/docs/capabilities/serial were read and said nothing on printers or protected classes
- `rawbt` a search on the RawBT Android print service (https://www.apkmirror.com/apk/402d/rawbt-print-service and the developer's pages) [Ws]; `ble-uuids` a search on BLE thermal printers (a forum thread and the `node-escpos` driver notes) [Ws]

### 9.4 Scanners

- `zebra-keypad` https://docs.zebra.com/us/en/scanners/general/sm72-ig/usb-interface/usb-parameter-defaults/keypad-emulation.html [W]; `zebra-keypad-defaults` and `zebra-defaults` search summaries of Zebra's USB parameter tables (the SM72, DS4678 and LS2208 pages) [Ws]
- `zebra-ds4678-nav` https://docs.zebra.com/us/en/scanners/general/ds4-series/ds4678-prg/usb-interface/usb-parameter-defaults.html and .../standard-parameter-defaults.html [W, navigation and parameter names only; the tables did not come through]
- `layout-search` a search on keyboard-layout mismatch for wedge scanners (help-centre articles) [Ws]
- `odoo-troubleshooting` https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/barcode/setup/device_troubleshooting.html [W]

### 9.5 Indian rules

- `lm-book` https://consumeraffairs.gov.in/public/upload/admin/cmsfiles/whatsnews/Book_on_Legal_Metrology_Packaged_Commodities_Rules,2011_with_all_amendments_whatsnews.pdf [W, the Department of Consumer Affairs' consolidated book, 56 pages, read from the extracted text]; `lm-act` and `lm-amend` search summaries (a law-firm note on the Act's definition and amendments of 2021 to 2023) [Ws]
- `fssai-comp` https://www.fssai.gov.in/upload/uploadfiles/files/Compendium_Labelling_Display_17_10_2022.pdf [W, Version V of 13 October 2022, 42 pages, read from the extracted text]; `fssai-search` search summaries including FSSAI's amendments page [Ws]
- `drugs-rules` https://cdsco.gov.in/opencms/resources/UploadCDSCOWeb/2022/drug_rules/Drugs%20Rules%201945_2024%2009.pdf [W, 963 pages, read from the extracted text for Rules 94 to 97]; `cdsco-faq` https://cdsco.gov.in/opencms/resources/UploadCDSCOWeb/2018/UploadPublic_NoticesFiles/Final%20FAQs%20on%20QR%20code%2021.07.2023.pdf [W, 2 pages]; `cdsco-qr-search` a search summary on the QR rule and its extension [Ws]
- `bis-blog` https://services.bis.gov.in/php/BIS_2.0/BISBlog/?p=560 [W]; `bis-faq` https://www.bis.gov.in/hallmarking-overview/hallmarking-faqs/hallmarking-faq [W]; `bis-search` a search summary on the HUID [Ws]

### 9.6 Competitors and market

- `busy-labels` https://busy.in/faqs/barcode-qr-code-lable-printing/barcode [W]; `busy-faq3` https://busy.in/faqs/barcode-qr-code-lable-printing/barcode/3/ [W]; `busy-faq5` https://busy.in/faqs/barcode-qr-code-lable-printing/barcode/5/ [W]; `busy-physical` a search summary of https://busy.in/faqs/can-you-use-barcodes-or-scanning-devices-to-expedite-the-physical-stock-voucher-entry-process-answerid-41899/ [Ws]
- `tally-help` https://help.tallysolutions.com/?p=193523 [W]
- `marg-care` https://care.margcompusoft.com/margerp/inventory-reports/4185/1/inftrees [W, the first step only of a 630,000-character page]; `marg-search` search summaries of Marg's pages [Ws]
- `vyapar-barcode` https://api1.vyaparapp.in/guides/how-to-generate-a-barcode-in-vyapar [W]
- `zoho-labels` https://www.zoho.com/en-ng/inventory/help/items/qrcode-generation.html [W]; `zoho-export` a vendor blog found by search [Ws]
- `odoo-nomenclature` https://www.odoo.com/documentation/19.0/applications/inventory_and_mrp/barcode/operations/barcode_nomenclature.html [W]; `odoo-gs1-nomenclature` https://www.odoo.com/documentation/19.0/applications/inventory_and_mrp/barcode/operations/gs1_nomenclature.html [W]; `odoo-labels` https://www.odoo.com/documentation/18.0/applications/inventory_and_mrp/inventory/shipping_receiving/setup_configuration/print_on_validation.html [W]; `odoo-software` https://www.odoo.com/documentation/19.0/applications/inventory_and_mrp/barcode/setup/software.html [W]; `odoo-adjust` and `odoo-print` search summaries of Odoo's adjustment and transfer pages and its forum [Ws]
- `commmerce-sizes` https://commmerce.com/blog/smart-barcode-label-printers-indian-fashion-chains-size-guide-2026/ [W, a vendor blog]; `avery-grids` and `indian-sheets` search summaries of retail listings for A4 label sheets [Ws]; `bartender` search summaries of reseller and vendor pages [Ws]

### 9.7 This repository and its siblings

[C] `src/keys.ts`, `src/ui/VoucherScreen.tsx`, `src/ui/widgets.tsx`, `src/ui/masters.tsx`, `src/ui/inventory/masters.tsx`, `src/ui/inventory/stockquery.tsx`, `src/ui/admin/BulkEntry.tsx`, `src/ui/modern/CommandPalette.tsx`, `src/ui/print/*`, `src/ui/reportTools.ts`, `src/ui/banking/chequeLeaf.tsx`, `src/ui/banking/chequeLayoutEditor.tsx`, `src/core/qr.ts`, `src/core/upi.ts`, `src/core/printConfig.ts`, `src/core/bulkPrint.ts`, `src/core/xlsx.ts`, `src/core/csvOut.ts`, `src/styles/print.css`, `src/data/db.ts`, `src/types.ts`, `electron/main.cjs`, `electron/preload.cjs`, `electron/exportBridge.cjs`, `package.json`, `CLAUDE.md`, `docs/HANDOFF.md`, and, read-only, `.wt-cf/docs/design/custom-fields-spec.md` and `.wt-cf/docs/design/back-nav-spec.md`.
[R] `docs/research/batch-serial-tracking.md`, `batch-serial-code-audit.md`, `tracking-verticals-india.md`, `pos-code-audit.md`, `mobile-market.md`, `mobile-personas-and-jobs.md`.

**The throw-away work** (not repo code, kept in the session scratchpad): the arithmetic of sections 1.4, 1.7 and 2.3; the EAN-13 and Code 128 encoders and the macOS Vision decode run of section 1.9; the PDF text extraction used for the TSC manual, the Zebra datasheet, the Legal Metrology book, the FSSAI compendium, the CDSCO FAQ and the Drugs Rules.
