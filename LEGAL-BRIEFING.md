# Legal Briefing — IP Risk for a Tally-Adjacent India-Market Accounting SaaS

> **Prepared as background research, NOT legal advice. Verify with qualified Indian IP
> counsel before launch.** This document is a founder's meeting-prep memo for a session with
> an Indian IP litigator. It cites public sources and flags every claim that could not be
> verified. Case citations marked **[verify]** must be pulled from the primary record before
> being relied on. Do not treat any statement here as a legal opinion.
>
> **Date:** 2026-07-09 · **Prepared for:** Akshay (owner) · **Product:** clean-room web/desktop
> accounting app; "Classic" mode mirrors Tally/TallyPrime keyboard workflow & information
> layout · **Working name at the time of writing:** "TallyWeb". **Since actioned:** the
> product now ships as **"Comsky Books"** (company: Comhard Technologies Pvt Ltd) — the
> recommendation in §1 below is done, and "TallyWeb" survives here only as the historical
> subject of that analysis. A trademark search on the new name is still outstanding.

---

## Executive summary

1. **The product name is the single biggest legal risk, by a wide margin.** "TallyWeb" (or
   any name containing "Tally") uses a competitor's registered mark inside your own brand.
   That is squarely what trademark infringement (Trade Marks Act 1999 s.29) and passing off
   are designed to stop. **Recommendation: rename to something with zero "Tally" content
   before any public use, filings, or spend on the mark.** Every serious competitor
   (Busy, Vyapar, Marg, Zoho) avoids "Tally" in its name — none is an accident.

2. **Mirroring the keyboard workflow and information layout is the most defensible part of
   the plan.** Under India's idea–expression dichotomy (*R.G. Anand v. Delux Films*, 1978),
   copyright protects expression, not ideas, methods or functional systems. Keystroke
   sequences and menu/command flow are close to "method of operation" — the exact category
   US courts held uncopyrightable in *Lotus v. Borland* (persuasive-only in India). Risk here
   is **low-to-moderate**, and lower still because your code is clean-room.

3. **Pixel-exact cloning of Tally's distinctive on-screen "get-up" is a separate, higher
   risk** than copying the workflow. Copying *functional layout* is defensible; slavishly
   reproducing Tally's distinctive colours, screen ornamentation and visual identity can
   expose you to **copyright (UI as part of a literary work / design) and passing off**.
   Reproduce the *workflow*, not the *skin*.

4. **Clean-room, from-scratch code is a strong asset — keep the paper trail.** Independent
   creation is a complete answer to copyright copying claims. Document that no Tally source,
   binaries, decompilation, or file-format reverse-engineering fed the build.

5. **"Import from Tally" / "alternative to Tally" / "migrate from Tally" marketing is
   normal and defensible** as nominative / honest comparative use (s.30(1), s.30(2)(d),
   s.35), *provided* you use the plain word (not Tally's logo/stylised mark), use it only as
   far as necessary, add a non-affiliation disclaimer, and never disparage. This is exactly
   what Busy, Vyapar, Marg and Zoho already do in public, apparently without being sued.

6. **Tally Solutions' *actual* India IP-litigation history could not be verified** in public
   sources for this memo (only its strong anti-piracy messaging and a certified-partner
   programme are visible). Because Akshay works at a Tally reseller/partner, **check whether
   the partner agreement contains non-compete, non-solicitation or IP terms** that bind him
   personally — that contractual risk may matter more than statutory IP risk. This is a
   priority question for counsel.

7. **"Tally" is a registered mark (at least the TALLY.ERP 9 family, Class 9) and a strong,
   reputed brand, but does not appear on IP India's published "well-known marks" list.**
   A court could still treat it as well-known *within accounting software*. Assume strong
   protection in-field; the safe path does not depend on the well-known question.

8. **Headline recommendation:** rename off "Tally"; ship the workflow, not the skin; keep the
   clean-room evidence; market migration/comparison in plain, non-disparaging, disclaimed
   language. Do these and the residual risk is manageable and in line with what the whole
   market already does.

---

## Risk matrix

| Element | Risk level | Primary legal basis | Mitigation |
|---|---|---|---|
| **Product name containing "Tally" ("TallyWeb")** | **High** | TM infringement s.29(1)/(2); dilution s.29(4); passing off | **Rename entirely off "Tally."** No "Tally" in brand, logo, app name, domain, store listing, or company name. |
| **Tally marks/logo used inside the UI** (menu labels, splash, help text) | **High** | s.29 use "in the course of trade"; s.29(4) dilution | Use only generic/functional labels. Never display Tally's word-mark styling or logo in-app. Nominative references (docs/marketing) only per guardrails. |
| **Pixel-exact trade dress** (Tally's exact colours, screen get-up, distinctive visual identity) | **Moderate–High** | Copyright (UI/artistic + program "look"); passing off (get-up); possibly design | Reproduce *functional* layout only. Differentiate palette, typography, chrome, branding. Your "Modern" theme is the hedge — keep it prominent. |
| **Keyboard flow & information workflow** (keystrokes, command sequence, field order) | **Low–Moderate** | Idea–expression dichotomy (*R.G. Anand*); method-of-operation (persuasive: *Lotus v. Borland*) | Frame as functional interoperability. Document independent design. Avoid copying non-functional expressive flourishes. |
| **Clean-room, from-scratch code** | **Low** | Copyright (literal + SSO, *Maraekat*) — but independent creation is a defence | Keep clean-room records: no Tally source/binaries/decompilation; no copied file formats/schemas beyond what's needed for interoperability. |
| **Comparative / migration marketing** ("import from Tally", "alternative to Tally") | **Low–Moderate** | Nominative fair use; s.30(1), s.30(2)(d), s.35, s.29(8) (disparagement) | Plain word only, minimal, add "not affiliated with / not endorsed by Tally Solutions" disclaimer, keep claims true, never denigrate. |
| **Reseller/partner contractual terms binding the founder** | **Unknown — verify** | Contract law (Tally partner agreement) | Pull and review the actual agreement for non-compete/IP/confidentiality clauses. Priority counsel item. |

---

## 1. Indian copyright — idea/expression and software UI / "look and feel"

**Idea–expression dichotomy (the foundation).** The Supreme Court in **R.G. Anand v. M/s
Delux Films, AIR 1978 SC 1613**, held there is **no copyright in an idea, subject-matter,
theme, plot or basic method** — infringement is confined to copying the *form, manner,
arrangement and expression* of the idea. The tests it laid down: (a) **substantial
similarity** in the mode of expression, and (b) the **lay-observer test** — whether an
ordinary viewer, seeing both works, gets the "unmistakable impression" that one is a copy.
Where the theme is the same but the presentation differs, there is no infringement. [1]

**Application to your workflow.** A keyboard-driven data-entry flow — which key does what,
the sequence of prompts, the order of fields — sits close to the *idea / method / function*
end of the spectrum, not the protected *expression* end. This is the strongest doctrinal
support for the "Classic mode mirrors the workflow" strategy.

**Software is a "literary work."** Under the **Copyright Act 1957, s.2(o)** a computer
programme is a literary work, protected under s.13/s.14. So the *code* is protected, and (per
below) so, arguably, is its "structure, sequence and organisation" and some UI expression —
but only against actual *copying*, not against independent re-creation of the same function.

**The one on-point Indian software case: Maraekat Infotech Ltd. v. Naylesh V. Kothari,
2016 SCC OnLine Bom 2369** (Bombay HC, Gupte J.). The court held that in software copyright
disputes, courts should compare not only literal code but also **overall similarity in
design features and program structure**, extending protection to a program's **"structure,
sequence and organisation" (SSO)**. [2][3] **Important caveats for us:** (i) this was a case
of **former employees allegedly copying the plaintiff's actual software** (EON-Exchange →
iExchangeForex) — i.e. real copying by insiders, not independent competitive re-creation;
(ii) SSO protection cuts *for* Tally only if you actually copied Tally's program — which a
clean-room build does not. It does show Indian courts will look past literal code to
"look and structure," so **avoid making your Classic build feel like a byte-for-byte reskin.**

**"Look and feel" / GUI is genuinely unsettled in India.** Commentary is consistent that
**India lacks clear jurisprudence on GUI "look and feel"**; the Copyright Act has no
GUI-specific provision, and copyright tends to reach only **exact or near-exact copying**,
so minor modifications typically avoid liability. [3] Indian analysis borrows US filters —
the **abstraction–filtration–comparison** test (*Computer Associates v. Altai*) and the
rejection of broad look-and-feel protection (*Apple v. Microsoft*) — to strip out functional
and public-domain elements before comparing. [3] **Bottom line: there is no Indian case
squarely holding that mirroring a competitor's UI workflow, via independent code, infringes
copyright.** Reasoning by analogy from *R.G. Anand* + the functional-filtering approach,
the *workflow* is low risk; a *pixel-identical visual clone* is where copyright/get-up
exposure rises.

**Design law is the more established route for GUI — and it runs against a copier, not for
you.** GUIs in India are increasingly analysed under the **Designs Act 2000** (Locarno
**Class 14-04**, "Screen Displays and Icons"; and **Class 32**, "graphic symbols/GUI," added
by the Design (Amendment) Rules 2019). Registrability has been contested — the Design Office
**rejected Amazon's GUI application** (GUI not a "finished article," only visible when powered,
"functional") — but recent decisions (e.g. **UST Global**, and a 2026 Calcutta HC ruling
recognising GUI layout/iconography/colour as design subject-matter **[verify — very recent,
confirm citation and holding]**) point toward design protection for *ornamental* GUI.
[3][4][5] **Relevance:** if Tally holds (or obtains) a **registered design** in specific
TallyPrime screen ornamentation, copying that exact ornamentation is a design-infringement
risk. Copying *functional layout* is not. **Counsel should check IP India's design register
for Tally screen-display registrations.** [verify]

**US persuasive authority (persuasive only in India — flag as such):**
- **Lotus Development Corp. v. Borland International, 49 F.3d 807 (1st Cir. 1995), aff'd by an
  equally divided Court, 516 U.S. 233 (1996):** a spreadsheet's **menu command hierarchy is an
  uncopyrightable "method of operation"** (17 U.S.C. §102(b)); anyone may build on it. Because
  SCOTUS split 4–4, it set no nationwide US precedent, but the First Circuit reasoning is the
  best articulation of why *command/keystroke systems* are outside copyright. **Directly
  analogous to a keyboard-workflow clone — but US, persuasive-only in India.** [6][7]
- **Google LLC v. Oracle America, 593 U.S. 1 / 141 S. Ct. 1163 (2021):** reimplementing the
  Java API's **declaring code and its structure/organisation was fair use**; the Court
  *declined* to decide whether the API was copyrightable at all, calling reuse that lets users
  deploy existing skills "transformative." **Analogous to letting Tally-trained users carry
  their muscle memory across — but US fair-use doctrine ≠ Indian "fair dealing" (s.52), which
  is narrower and enumerated; persuasive-only.** [8][9]

**Net copyright assessment:** workflow-mirroring via clean-room code is **low-to-moderate**
risk; pixel-exact visual cloning is **moderate-to-high**. The idea–expression line, the
near-exact-copying threshold for UI, and independent creation all favour the workflow strategy.

---

## 2. Indian trademark — well-known marks, infringement, dilution, trade dress / passing off

**Statutory frame (Trade Marks Act 1999).**
- **s.29(1)–(2):** infringement where a person uses, in the course of trade, a mark identical
  or deceptively similar to a registered mark for identical/similar goods, causing a
  likelihood of confusion. Putting "Tally" in your product name for accounting software is
  the textbook fact pattern.
- **s.29(4): dilution** — use of an identical/similar mark even on *dissimilar* goods can
  infringe where the registered mark has a **reputation in India** and use, without due
  cause, takes **unfair advantage of / is detrimental to** its distinctive character or repute.
  In **ITC Ltd v. Philip Morris Products SA** (Delhi HC) the court set out four elements:
  (i) diluting mark identical/similar to the senior mark; (ii) senior mark has a reputation in
  India; (iii) use without due cause; (iv) use takes unfair advantage of / harms distinctiveness
  or repute. The Delhi HC also indicated the reputation for dilution should extend across the
  country. **[verify precise citation — commonly cited as 2010 SCC OnLine Del / (2010) 42 PTC 572
  (Del)]**. [10][11]
- **s.2(1)(zg):** defines a **well-known trade mark**; **s.11(2)/(6)–(9):** protect well-known
  marks even across classes and list the factors and (Rule 124, TM Rules 2017) the formal
  declaration route (Form TM-M, fee ₹1,00,000). [12][13]

**Is "Tally" registered / well-known?**
- **Registered: yes (at least in part).** Public trademark listings show **TALLY.ERP 9**
  filed **16 March 2009**, in Class 9 ("calculating machines, data processing equipment and
  computers"), registered, proprietor **Bharat Goenka**. [14] The broader "Tally" / "TallyPrime"
  family is used extensively as a brand. **Counsel should pull the full register from
  tmrsearch.ipindia.gov.in** to confirm the exact live registrations (word mark "TALLY" per se,
  "TallyPrime," logo marks, classes 9 and 42). **[verify — full register pull needed]** [15]
- **Well-known (formal): apparently not.** A search of **IP India's published well-known
  marks list** (latest public list dated 10.10.2022) did **not** surface "Tally"; the list
  features marks like TATA, AMUL, GOOGLE, APPLE, SONY, etc. [16][17] **Caveat:** a mark can be
  found well-known by a court on the facts even without formal listing, and Tally is plainly a
  dominant, reputed brand in Indian accounting software — so assume **strong in-field
  protection** regardless. The safe strategy does not turn on this question.

**Trade dress / passing off (classical trinity).** Beyond registration, Tally can sue in
**passing off**, which protects unregistered get-up/goodwill. The classical trinity (from
*Reckitt & Colman v. Borden* — the UK "Jif Lemon" case, followed in India): (i) **goodwill /
reputation** in the get-up; (ii) **misrepresentation** by the defendant leading the public to
believe its goods are the claimant's / associated; (iii) **damage**. **Relevance:** a
pixel-faithful clone of Tally's distinctive screen "get-up" plus a "Tally"-containing name
could support a passing-off claim (users think it's a Tally product / endorsed). Mirroring
*functional* workflow, under a clearly different brand and visual identity, undercuts the
misrepresentation limb. **[verify precise Indian passing-off citations with counsel; trinity
is well-settled.]**

**Net trademark assessment:** name/logo use = **high**; in-UI mark use = **high**; functional
trade-dress mirroring under a distinct brand = **moderate**; distinctiveness of your own brand
is the main lever you control.

---

## 3. Nominative & comparative use — when "import from Tally" / "alternative to Tally" is lawful

**The permission exists in statute and case law — but it is bounded.**
- **s.30(1):** use is not infringement if it is in accordance with **honest practices** in
  industrial/commercial matters and does not, without due cause, take unfair advantage of or
  harm the mark's distinctiveness/repute.
- **s.30(2)(d):** use is permitted to indicate that goods/services are **adapted to form part
  of, or be compatible with,** the trademark owner's product — i.e. genuine
  compatibility/interoperability statements. This is the provision that most directly supports
  "compatible with Tally data / imports Tally data."
- **s.35:** saves bona fide descriptive use.
- **s.29(8):** advertising *using* another's mark **infringes** if it is not honest practice,
  is detrimental to distinctiveness/repute, or **takes unfair advantage** — the disparagement
  hook.

**Nominative fair use test (adopted in India).** The **Madras High Court in Consim Info Pvt.
Ltd. v. Google India Pvt. Ltd.** adopted the US three-part test: (1) **necessity** — the
product/service is not readily identifiable without using the mark; (2) **minimalism** — use
only so much of the mark as reasonably necessary; (3) **no false endorsement** — do not
suggest sponsorship/endorsement by the mark owner. [18]

**Two cautionary Indian cases on *how* you reference the mark:**
- **Hawkins Cookers Ltd. v. Murugan Enterprises** (Delhi HC, Division Bench): printing
  "HAWKINS" prominently (red colour) on gasket packaging **exceeded** nominative use — the
  undue emphasis suggested endorsement. Lesson: reference must be **understated and factual**,
  not prominent branding. [18]
- **Prius Auto Industries v. Toyota** (Delhi HC): even for genuine compatibility, the court
  required a **different font**, use only to indicate compatibility, and **clear attribution**
  to the accessory-maker. Lesson: differentiate typography, and attribute. [18]

**Comparative advertising — puffery allowed, disparagement not.** Indian courts consistently
hold you **may** claim your product is the best / better and make honest comparisons, but you
**may not disparage or denigrate** the competitor's product or mark: [19][20]
- **Reckitt & Colman v. M.P. Ramachandran** (Calcutta HC, 1999): a trader may call his own
  goods best, and even say they are better than a rival's, but **may not say the rival's goods
  are bad**. **[verify cite — 1999 PTC (19) 741 (Cal)]**
- **PepsiCo v. Hindustan Coca-Cola** (Delhi HC, 2003): the **manner** matters — ridiculing/
  denigrating a competitor's product is disparagement.
- **Dabur India v. Wipro** (Delhi HC, 2006): actionable disparagement must effectively
  **border on defamation**.
- **Hindustan Unilever v. Reckitt Benckiser** (Calcutta HC): comparison allowed without
  commenting adversely on the rival's qualities; must not harm the rival's mark's repute.
- **Colgate-Palmolive v. Hindustan Unilever:** a degree of implicit disparagement is tolerated
  so long as the ad stays within puffery.

**Practical takeaway:** "**Import your data from Tally in one click**," "**a modern
alternative to Tally**," "**built for Tally users**" are defensible *if* (a) true, (b) the
plain word "Tally" only — never the logo/stylised mark, (c) minimal and not turned into your
own branding, (d) carry a **"not affiliated with, or endorsed by, Tally Solutions Pvt. Ltd."**
disclaimer, and (e) never claim Tally is bad/defective. This is precisely the register the
whole market uses (Section 5).

---

## 4. Tally Solutions' enforcement posture — what could and could not be verified

**Could NOT verify (flagged):** No specific reported IP litigation by **Tally Solutions Pvt.
Ltd.** in India — against resellers, competitors, "Tally-compatible" claimants, or name/mark
misusers — surfaced in the public searches run for this memo. This is an **absence of found
evidence, not evidence of absence**; Tally may have sent cease-and-desist letters, filed
oppositions, or litigated without prominent online reporting. **Counsel should run court-record
and TTAB/opposition searches (IP India, e-Courts, HC dockets) for "Tally Solutions" as
plaintiff/opponent.** [verify — priority research item] [21][22]

**What is visible:**
- A pervasive ecosystem of **anti-piracy / "don't use cracked Tally" messaging** (from Tally
  partners and resellers), indicating Tally actively discourages piracy and cracked builds —
  consistent with a company protective of its software rights, though this is about *piracy of
  Tally's own product*, not about competitors or interoperability. [23][21]
- A formal **certified-partner programme** with terms and conditions — meaning **contractual**
  obligations likely bind partners/resellers. [24]

**Direct implication for the founder.** Because Akshay works at a **Tally reseller/partner**,
the most concrete near-term risk may be **contractual, not statutory**: partner agreements can
carry **non-compete, non-solicitation, confidentiality and IP-assignment** clauses, and using
knowledge/relationships from that role to launch a competing product can trigger them. **This
is arguably the highest-value question to take to counsel** — it is specific, actionable, and
independent of the trademark/copyright analysis. **[verify — pull and review the actual
partner/employment agreement.]**

---

## 5. What competitors actually do (Busy / Vyapar / Marg / Zoho Books)

The established Tally competitors provide a **real-world calibration**: they all (i) avoid
"Tally" in their product name, (ii) freely publish **"migrate from Tally" / "Tally
alternative"** marketing that names Tally by the plain word, and (iii) appear **not to have
been sued by Tally** for doing so (no such suits found in public search — [verify]).

| Product | "Tally" in name? | Tally-migration / comparison claim? | UI resemblance to Tally | Known Tally disputes |
|---|---|---|---|---|
| **Busy** (Busy Infotech) | No | **Yes, prominent.** "Migrate to BUSY from Tally"; ships a **"Tally2BUSY"** utility; names Tally with no disclaimer. [25] | Own Windows UI; not a Tally clone (own conventions) | None found [verify] |
| **Vyapar** | No | **Yes, aggressive.** Dedicated **"Tally alternative"** landing page; "Import Tally data … one-click"; comparison page naming Tally; "best Tally alternative." No disclaimer seen. [26][27] | Own modern/mobile-first UI; deliberately *unlike* Tally (positions as friendlier) | None found [verify] |
| **Marg ERP / MargBooks** | No | **Yes.** "MargBooks vs Tally Solutions" comparison; Tally→Marg migration tooling. [28][29] | Own dense desktop UI; not a Tally clone | None found [verify] |
| **Zoho Books** | No | **Yes, official.** Zoho help page: **"Migrate from Tally to Zoho Books"** (chart of accounts, opening balances, etc.). [30] | Wholly different cloud/web UI | None found [verify] |

**Key insight for the meeting:** the market has **normalised naming Tally for migration and
comparison** — that behaviour is low-risk and you can point to it. The behaviour **no one**
engages in is **putting "Tally" in the product/brand name** or shipping a literal visual
clone. Your plan matches the market on the safe axis (migration/comparison marketing;
workflow familiarity) and is the **only** one flirting with the risky axis (the "TallyWeb"
name). Renaming brings you fully in line with proven, unchallenged industry practice.

*Note: "UI resemblance" above is a light-touch observation, not a forensic comparison; none of
these competitors attempts Tally's pixel-faithful "Classic" get-up the way this product's
Classic mode does — that is a differentiator worth being deliberate about.*

---

## 6. Recommended launch guardrails

**DO**
- **Rename off "Tally" completely** — brand, logo, app name, domain, company name, store
  listings, package/binary names, social handles. Clear the new name via a TM search before
  spend.
- **Keep and prominently offer the "Modern" theme** as the default face of the product; make
  "Classic" an opt-in familiarity mode, described in **functional** terms ("keyboard-first
  data entry," "fast ledger navigation"), not as "the Tally look."
- **Reproduce workflow, not skin.** Match keystrokes/field order/command flow (functional);
  differentiate palette, typography, window chrome, iconography, splash, and all branding.
- **Preserve clean-room evidence:** written record that the code is original; no Tally source,
  binaries, decompilation, or SDK; no copied database schemas/file formats beyond the minimum
  needed for genuine data-import interoperability. Keep design docs dated.
- **Market migration/comparison in plain, disclaimed language:** "imports data from Tally,"
  "an alternative to Tally," "built for Tally users." Add **"[Product] is not affiliated with,
  sponsored by, or endorsed by Tally Solutions Pvt. Ltd. 'Tally' is a trademark of its
  respective owner."**
- **Use only the plain word "Tally"** in such references — never Tally's logo, stylised
  word-mark, colours, or screenshots of Tally.
- **Pull and review the reseller/partner agreement** for non-compete/IP/confidentiality terms
  *before* launch; get counsel's read.
- **File your own trademark** for the new name (Classes 9 and 42) early.
- **Have counsel run the Tally register + design register + litigation-history searches**
  flagged throughout.

**DON'T**
- **Don't ship any name/logo/domain containing "Tally"** or a near-variant ("Tallly,"
  "iTally," "TallyGo," etc.).
- **Don't display Tally's marks, logo, or product screenshots inside the app** or in ads.
- **Don't reproduce Tally's exact colour scheme / distinctive get-up pixel-for-pixel** — copy
  the workflow, not the visual identity.
- **Don't imply affiliation, endorsement, partnership, or "official" compatibility.**
- **Don't disparage Tally** ("Tally is outdated/buggy/insecure") — honest comparison only.
- **Don't decompile, reverse-engineer Tally binaries, or copy Tally source / file formats**
  beyond what's genuinely necessary for import interoperability — and document that boundary.
- **Don't use knowledge, customer lists, or materials obtained via the reseller relationship**
  in ways the partner agreement forbids.
- **Don't rely on US cases (Lotus, Google v. Oracle) as if they bind Indian courts** — they
  are persuasive only.

---

## 7. Open questions for the IP litigator

1. **Name clearance:** Given "Tally" is registered (Class 9; confirm Class 42 and word-mark
   "TALLY"/"TallyPrime"), how far must a new name sit from "Tally"? Any risk in taglines like
   "for Tally users"?
2. **Register pull:** What exactly does Tally Solutions hold — word marks, logo marks,
   **registered designs** over TallyPrime screens, classes, and are any live/opposed? (IP India
   TM + Design registers.)
3. **Well-known status:** Has Tally ever been *declared* well-known, or asserted well-known in
   any proceeding? Does it matter here given clear in-field reputation?
4. **Litigation history:** Any record of Tally Solutions as plaintiff/opponent — against
   competitors, "Tally-compatible" vendors, resellers, or name misusers? C&D letter patterns?
5. **Reseller/partner agreement:** Does Akshay's partner/employment agreement contain
   non-compete, non-solicit, confidentiality, or IP-assignment clauses that reach this product?
   Enforceability of any non-compete post-termination under Indian law (s.27 Contract Act)?
6. **Classic mode:** Is the pixel-faithful "Classic" get-up worth the incremental
   passing-off/design/copyright exposure, or should it be visually differentiated while keeping
   the keyboard workflow?
7. **Migration marketing:** Exact safe wording and disclaimer for "import from Tally" /
   "alternative to Tally," in light of s.29(8), s.30, and the *Hawkins*/*Prius* limits on how
   prominently the mark may appear.
8. **Data-format interoperability:** Legal exposure from reading/writing Tally's data/export
   formats (XML etc.) for import — is that protected interoperability or a risk?
9. **Copyright scope:** Given *Maraekat* (SSO) and the near-exact-copying threshold for UI,
   how much visual/structural distance is "enough" for the Classic mode?
10. **Design law:** If Tally holds registered GUI designs, which specific screen elements must
    we avoid reproducing?

---

## 8. Sources

*Primary sources (statutes, judgments, official registers) preferred; reputable secondary
(law-firm notes, Lexology/Mondaq, academic) used where primary was not directly retrievable.
Items marked **[verify]** in the text still require confirmation against the primary record.*

1. *R.G. Anand v. Delux Films*, AIR 1978 SC 1613 — Wikipedia summary: https://en.wikipedia.org/wiki/RG_Anand_v._Deluxe_Films ; case note: https://lawbhoomi.com/r-g-anand-v-m-s-deluxe-films-and-ors/
2. *Maraekat Infotech Ltd. v. Naylesh V. Kothari*, 2016 SCC OnLine Bom 2369 — CaseMine: https://www.casemine.com/judgement/in/58117eda2713e179478c4db1
3. "Protection of Graphical User Interfaces in India: Copyright or Design?" — Tech Law Forum @ NALSAR: https://techlawforum.nalsar.ac.in/protection-of-graphical-user-interfaces-in-india-copyright-or-design/
4. "The GUI Conundrum: Finding The Optimal IP Protection In India" — Mondaq: https://www.mondaq.com/india/copyright/1331998/the-gui-conundrum-finding-the-optimal-ip-protection-in-india
5. "GUIs as registrable designs in India: From UST Global to the 2026 clarification" — Asia IP: https://asiaiplaw.com/section/ip-analysts/guis-as-registrable-designs-in-india-from-ust-global-to-the-2026-clarification
6. *Lotus Dev. Corp. v. Borland Int'l*, 49 F.3d 807 (1st Cir. 1995), aff'd 516 U.S. 233 (1996) — Wikipedia: https://en.wikipedia.org/wiki/Lotus_Dev._Corp._v._Borland_Int%27l,_Inc. ; text (BitLaw): https://www.bitlaw.com/source/cases/copyright/Lotus.html
7. *Lotus v. Borland* — Oyez: https://www.oyez.org/cases/1995/94-2003
8. *Google LLC v. Oracle America*, 593 U.S. 1 (2021) — opinion: https://www.supremecourt.gov/opinions/20pdf/18-956_d18f.pdf
9. *Google v. Oracle* — Wikipedia: https://en.wikipedia.org/wiki/Google_LLC_v._Oracle_America,_Inc. ; U.S. Copyright Office summary: https://www.copyright.gov/fair-use/summaries/google-llc-oracle-am-inc-2021.pdf
10. "Beyond Boundaries: How Section 29(4) Protects Trademarks with Reputation" (ITC v. Philip Morris four-element test) — Legacy Partners: https://legacypartners.in/insights-and-research/beyond-boundaries-how-section-294-protects-trademarks-with-reputation
11. "Trademark Infringement v/s Trademark Dilution" — Khurana & Khurana: https://www.khuranaandkhurana.com/trademark-infringement-v-s-trademark-dilution-the-legal-power-of-being-well-known
12. Section 29, Trade Marks Act 1999 — iPleaders: https://blog.ipleaders.in/section-29-of-trade-marks-act-1999/
13. "All You Need To Know About Well-Known Trademarks" — Mondaq: https://www.mondaq.com/india/trademark/808148/all-you-need-to-know-about-well-known-trademarks
14. TALLY.ERP 9 trademark application detail (filed 16 Mar 2009, Class 9, Bharat Goenka) — Company Vakil: https://www.companyvakil.com/trademarksearch/TALLY.ERP-9/1287841
15. IP India Trade Marks Public Search (to pull the full Tally register): https://tmrsearch.ipindia.gov.in/tmrpublicsearch/
16. IP India — Well-Known Trademarks page: http://www.ipindia.gov.in/pages/trade-marks/publications/well-known-trademarks
17. IP India — "List of Well-Known Trade Marks as of 10.10.2022" (PDF; Tally not listed): https://ipindia.gov.in/writereaddata/Portal/Images/pdf/List_of_Well-Known_Trade_Marks_as_of_10.10.2022.pdf
18. "Doctrine of 'Nominative Fair Use' under Trademark Law" (Consim Info v. Google; Hawkins v. Murugan; Prius v. Toyota; s.30(2)(d)) — AZB & Partners: https://www.azbpartners.com/bank/doctrine-of-nominative-fair-use-under-trademark-law/
19. "Comparative Advertising: Drawing the Fine Line Between Puffery and Disparagement" (Reckitt & Colman v. Ramachandran; PepsiCo v. HCC; Dabur v. Wipro; HUL v. Reckitt) — Anand and Anand: https://www.anandandanand.com/news-insights/comparative-advertising-drawing-the-fine-line-between-puffery-and-disparagement/
20. "Doctrine of nominative fair use under Trademark law" — Mondaq: https://www.mondaq.com/india/trademark/879442/doctrine-of-nominative-fair-use-under-trademark-law
21. "Fighting Digital Piracy and Copyright Infringement in India" — Lexology: https://www.lexology.com/library/detail.aspx?g=610f3619-d8cb-4995-bde7-e168aa75072c
22. "Copyright piracy and cybercrime: enforcement challenges in India" — WIPO Magazine: https://www.wipo.int/en/web/wipo-magazine/articles/copyright-piracy-and-cybercrime-enforcement-challenges-in-india-42977
23. Tally Solutions — Wikipedia (company background, products): https://en.wikipedia.org/wiki/Tally_Solutions
24. Tally Certified Partners programme (partner terms exist): https://tallysolutions.com/global/partner-with-us/
25. BUSY — "Migrate to BUSY from Tally" (Tally2BUSY utility): https://busy.in/migration/ ; https://busy.in/tally-to-busy-data-transfer-tool/
26. Vyapar — "Tally Alternative" landing page: https://vyaparapp.in/free/tally-alternative
27. Vyapar — comparison with other apps (names Tally; "import/switch"): https://vyaparapp.in/comparison-with-other-apps
28. MargBooks vs Tally Solutions comparison: https://margbooks.com/margbooks-vs-tallysolutions.html
29. Marg ERP 9+ vs TallyPrime comparison — Software Advice: https://www.softwareadvice.com/erp/marg-erp-9-plus-profile/vs/tally/
30. Zoho Books — "Migrate from Tally to Zoho Books" (official help): https://www.zoho.com/ae/books/help/migration/tally-to-zoho-books.html
