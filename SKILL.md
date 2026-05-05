---
name: proposal
description: Generate beautiful single-page client proposals as a document-format replacement for PDFs and Google Docs. Persists answers, fetches brand colors from the client's site (with WCAG contrast warnings), generates a Tailwind-built single-page HTML, and runs the `/critique` design skill via the Skill tool to grade and fix the output. Includes dynamic OG images for unfurls and version-history snapshots on revise. Use when the user says "create a proposal", "client proposal", "scope of work", "SOW", or "pitch document". Use `/proposal --revise <slug>` to update an existing proposal, `/proposal --clone <slug>` to start a new one from an existing discovery, or `/proposal --theme <name>` to swap the visual theme (editorial, technical, minimal, icalia). The `--icalia` shorthand applies the Icalia Labs house theme — also the automatic fallback when the client has no brand identity of their own.
argument-hint: [client name | --revise <slug> | --clone <slug>] [--theme editorial|technical|minimal|icalia | --icalia]
---

# /proposal

Generate single-page client proposals as a beautiful, branded HTML document — the replacement for PDF / Google Doc proposals. Static, no live actions, just a polished read.

## Operating principle

A proposal does not close a deal — the conversation does. The proposal is the artifact your champion hands to people who weren't in the room. Its job is to **reduce career risk for the champion** by giving them ammunition to defend the choice. Every section earns its place by pre-empting an objection the champion will face.

This skill is **document-replacement**, not a SaaS product. There are no live buttons, no accept tracking, no view analytics. The output is a beautifully formatted page deployed to a URL the seller emails or pastes in Slack. The buyer reads it the way they'd read a PDF and replies through the existing email thread.

If the seller can't answer the discovery questions in the buyer's own words, the proposal won't save the deal. Push back. Don't generate from a thin brief.

## Skill assets (in `~/.claude/skills/proposal/`)

```
SKILL.md                          ← you are here
template.html                     ← the single-page proposal template
tailwind-input.css                ← Tailwind source for precompiled CSS
tailwind.config.js                ← content paths for JIT
discovery-schema.json             ← canonical schema for discovery.json
extract-colors.sh                 ← multi-source brand color extractor (WCAG-aware)
seller-defaults.json              ← persisted seller info — auto-loaded if present
themes/
  ├── editorial.json              ← default theme (Fraunces + Inter, white)
  ├── technical.json              ← dark + IBM Plex + JetBrains Mono numerals
  ├── minimal.json                ← Inter only, neutral grays, sentence-case eyebrows
  └── icalia.json                 ← Icalia Labs house theme — Inter Black, navy + red, fixed left accent bar; auto-fallback when buyer has no brand
examples/
  ├── saas-acme.html              ← reference: SaaS implementation pattern
  ├── agency-globex.html          ← reference: agency/creative pattern
  └── services-initech.html       ← reference: professional services pattern
vercel-starter/
  ├── vercel.json                 ← clean URLs + security headers + asset caching
  ├── index.html                  ← private root landing page
  ├── package.json                ← @vercel/og dependency for OG image route
  ├── assets/tailwind.css         ← precompiled, production-ready
  └── api/
      └── og.tsx                  ← dynamic OG image route (only runtime asset)
```

**Read `examples/*.html` before generating.** They are the canonical reference for what a good filled-in proposal looks like across industries.

## When to use / when NOT to

- ✅ Sending a proposal, scope of work, SOW, quote, or pitch document
- ✅ User has had a discovery call and needs to put it in writing
- ❌ Marketing landing page → `/copy-board` or `/prototype`
- ❌ Internal planning docs → write directly
- ❌ Multi-page deck → this skill is single-page only by design
- ❌ User has not had any conversation with the prospect → tell them to have one first

## Argument routing

| Invocation | Behavior |
|---|---|
| `/proposal` | Full new-proposal flow (Steps 0–7) |
| `/proposal Acme Corp` | Same, with client name pre-filled |
| `/proposal --revise <slug>` | Load `proposals/<slug>/discovery.json`, ask what's changed, snapshot current to `versions/v{n}.html`, regenerate, redeploy |
| `/proposal --clone <slug>` | Load `proposals/<slug>/discovery.json` as a starting point, ask what's different, save to a NEW slug |
| `/proposal --theme <name>` | Apply a visual theme at render time. `<name>` is one of `editorial` (default), `technical`, `minimal`, or `icalia`. Combinable with all of the above. |
| `/proposal --icalia` | Shorthand for `--theme icalia`. Applies the Icalia Labs house theme (Inter Black display, Trebuchet eyebrows, navy + red, fixed left accent bar). Also the **automatic fallback** when the buyer has no brand identity surfaced during Step 3 (no extractable site, or seller declines all candidates). |

## The flow

```
0. Bootstrap working dir         → scaffold vercel.json, assets, api/og.tsx
1. Load seller-defaults.json     → auto-fill seller's company info (skip questions)
1.6 Theme selection              → resolve --theme arg (or default editorial),
                                   load themes/<name>.json, remember on project.theme
2. Discovery (3 chunks)          → Setup batch → Story (1-by-1) → Scope (1-by-1)
3. Brand color extraction        → multi-source w/ WCAG flag, present candidates
                                   (compare against theme.page_bg, NOT always white)
4. Generate single-page HTML     → adapt template.html with discovery JSON +
                                   inject theme fonts link + theme css block
5. Self-check + tone enforcement → strip banned words, regenerate-and-rank
6. /critique design pass         → invoke critique skill VIA Skill TOOL (REQUIRED)
7. Review with seller            → iterate
8. Deploy to Vercel              → preview then prod via /vercel:deploy
```

## STEP 0 — Bootstrap

If the user's CWD does not contain `vercel.json`, scaffold from the starter:

```bash
SKILL_DIR="$HOME/.claude/skills/proposal"
cp "$SKILL_DIR/vercel-starter/vercel.json" ./vercel.json
cp "$SKILL_DIR/vercel-starter/index.html" ./index.html
cp "$SKILL_DIR/vercel-starter/package.json" ./package.json
mkdir -p ./assets ./api
cp "$SKILL_DIR/vercel-starter/assets/tailwind.css" ./assets/tailwind.css
cp "$SKILL_DIR/vercel-starter/api/og.tsx" ./api/og.tsx
```

Add `.gitignore` if missing: `.vercel/`, `.DS_Store`, `node_modules/`.

## STEP 1 — Load seller defaults

Before any discovery questions, check for `~/.claude/skills/proposal/seller-defaults.json`:

- **If it exists**: load it. Show the seller a one-line summary (`"Defaults loaded for Vertex Studio · Owen Rivera · owen@vertex.studio. Use these? (Y/n)"`). If yes, skip the seller-info questions in Chunk A. If no, ask normally and offer to update the defaults at the end.
- **If it doesn't exist**: ask the seller-info questions normally. After the proposal is done, ask: *"Save these as defaults for next time? (Y/n)"* — if yes, write `~/.claude/skills/proposal/seller-defaults.json` with company name, tagline, seller name, seller title, seller email.

Schema for `seller-defaults.json`:
```json
{
  "company": {
    "name": "Vertex Studio",
    "tagline": "Conversion-driven product design for B2B fintech.",
    "seller_name": "Owen Rivera",
    "seller_title": "Partner",
    "seller_email": "owen@vertex.studio"
  }
}
```

## STEP 1.6 — Theme selection

Themes control the **visual layer only** — typography, color scale, eyebrow treatment, page background. They do not change section structure, section order, the discovery questions, or any anti-pattern. Section #N is still section #N regardless of theme.

### Resolve the theme

1. If `--icalia` shorthand was passed → resolve to `icalia`.
2. Else if `--theme <name>` was passed → use that name. Validate against the enum below.
3. Else if `--revise` was used and `proposals/<slug>/discovery.json` already has `project.theme` → reuse it.
4. Else if Step 3 (brand color extraction) returns **no usable brand** for the buyer (no website, or seller declines all candidates and there is no prior `project.theme`) → fall back to `icalia`. The Icalia theme defines `default_brand_color` `#CC3239` so the doc still has an accent without one being supplied. Tell the seller: *"No brand surfaced for the buyer — falling back to the `icalia` house theme. Pass `--theme editorial` or `--theme minimal` to override."*
5. Else → default to `editorial`.

Valid theme names (v1):

| Name | Look | Best for |
|---|---|---|
| `editorial` *(default)* | Fraunces serif headlines + Inter body, white page, slate text, brand color as quiet accent. | Generalist — founders, execs, most B2B buyers. The current/legacy look. |
| `technical` | IBM Plex Sans + JetBrains Mono numerals on a deep slate background (`#0B0F19`). Tabular monospace prices and dates. Reads like internal engineering documentation. | SaaS, dev-tools, implementation deals where the buyer is technical (CTO, eng-lead, platform). |
| `minimal` | Inter for both display and body — no serif anywhere. Neutral-gray scale (instead of slate), sentence-case eyebrows (instead of uppercase tracked), tighter display tracking. | Design-conscious modern startups. When the proposal itself should feel minimal. |
| `icalia` | Inter Black (900) display, Trebuchet MS eyebrow caps, white page with deep navy (`#1C2333`) inset surfaces and Icalia red (`#CC3239`) brand accent. Fixed 4px left red accent bar runs the full viewport as a continuous brand signal. Default brand color when none is supplied. | Icalia Labs house look. Buyers without brand identity (pre-launch, stealth, no website). When the seller wants the Icalia visual register (executive, restrained, factual). |

If the user passes an unknown `--theme`, abort with: *"`--theme <name>` not recognized. Available themes: `editorial`, `technical`, `minimal`, `icalia`. Pass one of these or omit the flag for the editorial default."*

### Load the theme spec

```bash
THEMES_DIR="$HOME/.claude/skills/proposal/themes"
cat "$THEMES_DIR/<name>.json"
```

Each theme JSON exposes:
- `page_bg` — used by Step 3 for the WCAG contrast pass
- `fonts_link_html` — `<link>` markup for the theme's Google Fonts
- `css` — a CSS string scoped to `body[data-theme="<name>"]` selectors that overrides the editorial defaults
- `brand_color_targets` — descriptive list of where `--brand` is applied (kept consistent across themes for content parity)
- `wcag_note` *(technical only)* — guidance on the dark-background contrast check

### Persist the choice

Set `project.theme` on `discovery.json` so:
- `--revise` re-renders with the same theme by default (no need to repass `--theme`)
- The audit trail shows which theme was used for each version

### Industry-aware default suggestion (advisory only — do not auto-apply)

If `--theme` was not passed, after Chunk A captures `industry`, the skill MAY suggest a non-default theme based on this map and ask the seller — but never silently override:

| Industry | Suggested theme |
|---|---|
| `saas`, `implementation` | `technical` worth offering |
| `agency_creative`, `ecommerce` | `minimal` worth offering |
| Buyer has no brand / no website / pre-launch | `icalia` worth offering |
| Everything else | `editorial` |

The phrasing should be a single line: *"Industry is SaaS — want to use the `technical` theme (dark, IBM Plex, mono numerals) instead of the default editorial look? (Y/n)"*. If the seller declines, stay on `editorial`.

## STEP 2 — Discovery (three chunks)

### Chunk A — Setup (one batch, 4 questions)

If `seller-defaults.json` was loaded and confirmed, skip A4.

```
A1. Client company name + their website URL
A2. Industry — pick one: saas | agency_creative | professional_services
    | consulting | implementation | ecommerce | other
A3. Project codename / working title.
A4. (skip if seller-defaults loaded) Your company info — name, tagline,
    your name + title, contact email.
```

After A: kick off brand extraction (Step 3) in parallel with Chunk B.

### Chunk B — Story (ONE QUESTION PER TURN)

**Critical**: ask Chunk B questions one at a time, waiting for the seller's reply before asking the next. This is the difference between a chat and a tax form.

```
B1. Champion: name, title, email of the person on the buyer side advocating
    for this. They are who the proposal serves.
B2. Stakeholders who must say yes — list each (CFO, CTO, board, legal,
    procurement) and the likely objection from each.
B3. The problem — IN THE BUYER'S WORDS. Paste the actual quote from
    the call/email. If they don't have one, say "I don't have a quote"
    and the skill will push back.
B4. Buyer quote — a single line + its source (will appear as a pull quote).
B5. The outcome they're buying — measurable. 3 specific metrics if possible.
B6. Hesitations they voiced. Cost? Timeline? "We tried this before"?
B7. What's already been agreed in conversation. What's still soft.
```

**Gate**: If B3 is paraphrased corporate-speak, stop and tell the seller: *"The proposal will read generic. Go get the actual quote, then come back."* Then wait. Don't continue.

After B: ask `"Ready for the scope chunk? (Y/n)"`. Do not advance until confirmed.

### Chunk C — Scope (ONE QUESTION PER TURN)

Same one-at-a-time discipline as Chunk B. Industry-aware emphasis based on A2.

```
C1. Approach — 2 to 5 phases. Each: name, duration, what happens, why.
C2. Deliverables — concrete, countable. 5 to 12 items.
C3. Out of scope — what's explicitly NOT included.
C4. Timeline — total duration + 4 to 8 milestones with dates.
    Plus dependencies that could slip.
C5. Pricing — model (fixed | T&M | retainer | tiered | subscription),
    headline price (already formatted with the currency symbol),
    payment terms, scope-change policy.
C6. Proof points — 2 to 3 named case studies.
C7. Risks — top 2-3 honest risks + the mitigation MECHANISM (not a hope).
C8. Next-step paragraph — what's the simple, dated invitation? (No CTA
    button — proposal is read-only. Just text + email + valid-until date.)
C9. Validity period — most proposals say 14-30 days. Ask the seller to pick.
    NOTE: this is informational only — the page displays "Valid until <date>"
    but doesn't enforce it. Buyers can still read the page after that date.
C10. Optional T&C appendix? If yes: IP ownership, confidentiality,
    termination, governing law.
```

After C: persist all answers to `proposals/<slug>/discovery.json` matching `discovery-schema.json`. Slug = `<client-kebab>-<8-char-token>` (token from `openssl rand -hex 4`).

## STEP 3 — Brand color extraction

```bash
bash $HOME/.claude/skills/proposal/extract-colors.sh <url>
```

The script returns 5–10 candidates with WCAG contrast flags:

```
#0B5FFF   rgb(11, 95, 255)    css-freq      ✓ safe for white CTA text
#F4F6FA   rgb(244, 246, 250)  css-freq      ⚠ too light for white CTA text — pair with dark text
#1C2333   rgb(28, 35, 51)     css-freq      ✓ safe for white CTA text
```

**Rules:**
- Only pick a `✓ safe` candidate as PRIMARY (used on the brand button is gone, but emphasis numbers and price still inherit `--brand` color).
- If the seller insists on a `⚠` color, use it as ACCENT only, never as PRIMARY.
- If no website yet (or no usable brand surfaces), and the active theme is `icalia`, default to `#CC3239` (Icalia red) as primary and `#1C2333` (navy) as the inset surface — both already declared by the theme's `default_brand_color` and `--icalia-navy` CSS variable.
- If no website yet and a non-Icalia theme is active, default to `#0F172A` slate primary (or `#FFFFFF` text accent for the `technical` theme).
- The unbranded-buyer fallback automatically promotes the active theme to `icalia` (see Step 1.6), so this path is rare.
- Brand color is an accent, not a hero. Used only on: pull-quote left border, large outcome metrics, milestone dates, ✓ checkmarks, mitigation eyebrow, pricing headline, footer year. Everything else is grayscale.

**Theme-aware contrast**: the WCAG check must compare the chosen brand color against the **theme's `page_bg`**, not always white. For the `technical` theme (`#0B0F19`), a hex like `#0B5FFF` that's "✓ safe on white" may fail on dark — re-test against the theme background and prefer brighter, higher-luminance candidates. Each theme JSON exposes `page_bg` for this lookup; the `technical.json` theme also includes a `wcag_note` with explicit guidance.

Save to `discovery.json` under `brand`.

## STEP 4 — Generate the HTML

Read `template.html` and adapt with the `discovery.json` answers. **Do not rewrite from scratch.**

**Required sections, in order**:

| # | Section | Pre-empts |
|---|---------|-----------|
| 1 | Header (sticky) — company, "Proposal for X", v{n}, valid-until, email link | "Is this still current?" |
| 2 | Cover — prepared for, by, date, headline, summary, key metadata grid | "Is this for me?" |
| 3 | Situation — problem in their words + pull quote | "Do they understand us?" |
| 4 | Outcome — 3 measurable results | "What does success look like?" |
| 5 | Approach — phased with the *thinking* per phase | "Is this risky?" |
| 6 | Deliverables — concrete + out-of-scope box | "What am I getting?" |
| 7 | Timeline — milestones + dependencies | "When, what could slip?" |
| 8 | Investment — pricing surfaced, with includes list | "Is this fair?" |
| 9 | Why us — 2-3 named proof points with metrics | "Can they do this?" |
| 10 | Risks & mitigations — explicit | "What could go wrong?" |
| 11 | Next step — text only ("Reply to <email> by <date> to confirm") | "What do I do?" |
| 12 | (Optional) T&C appendix — collapsible | Procurement asks |
| 13 | Footer — company, contact, validity, reference | "Who do I reach?" |

**No CTA button. No share button. No accept form.** This is a read-only document. The next-step section ends with one sentence: *"Reply to seller@email.com by 2026-05-23 to confirm."*

**Key template placeholders to fill** (full list in `template.html`):
- All `{{...}}` text fields
- `{{LANG}}` from `locale.language` (English-only for now)
- `{{BRAND_PRIMARY_HEX}}` for `--brand` CSS variable
- `{{BRAND_HEX_NOHASH}}` for OG image route param
- `{{*_ENC}}` URL-encoded versions for OG meta + mailto subjects
- `{{PROPOSAL_VERSION}}` (defaults to 1, increments on `--revise`)
- `{{#if TERMS_INCLUDE}} ... {{/if}}` block — keep or strip based on `terms.include`
- **Theme placeholders (from Step 1.6)**:
  - `{{THEME_NAME}}` — value of `project.theme` (e.g., `editorial`, `technical`, `minimal`). Lands on `<body data-theme="...">`.
  - `{{THEME_FONTS_LINK_HTML}}` — paste verbatim from `themes/<name>.json#fonts_link_html`. Replaces the legacy hardcoded Google Fonts `<link>`.
  - `{{THEME_STYLES_CSS}}` — paste verbatim from `themes/<name>.json#css` into the inline `<style>` block. For `editorial`, this is intentionally empty (the base template + tailwind.css already provide the editorial defaults).

**Output path**: `proposals/<slug>/index.html` plus `proposals/<slug>/discovery.json`.

**Tone rules** (enforced in self-check, Step 5):

- **Banned words** — strip on sight: *revolutionary, best-in-class, cutting-edge, world-class, leverage, synergy, passionate about, unlock, unleash, supercharge, seamless, frictionless, effortless, game-changer, holistic, robust* (when used vaguely).
- **No marketing voice.** Confident, factual, measured.
- **Mirror the buyer's language** from B3/B4.
- **Numbers over adjectives.** Short sentences.
- **No "About Us" / "Our Values" sections.** The seller is not the subject.

## STEP 5 — Self-check + regenerate-and-rank pass

Run through the output before showing the seller:

1. **Banned-word scan**: grep the generated HTML. If hits, regenerate the offending sentence.
2. **Specificity check**: every section must contain at least one specific number, name, or quote. If a section is all adjectives, regenerate from `discovery.json`.
3. **Mirror check**: situation section must include at least one phrase from B3/B4.
4. **Regenerate-and-rank**: for the cover headline and the next-step headline, generate 3 candidate variants. Pick the most specific (with a number, date, or buyer language). Discard the others.
5. Report what you stripped/changed in a 3-line summary.

## STEP 6 — `/critique` design pass (REQUIRED, real Skill invocation)

This is the highest-leverage step. **Actually invoke the `/critique` skill — do NOT simulate it.**

```
1. Open the proposal:
     open proposals/<slug>/index.html
   (or use /browse to take a screenshot for the critique input)

2. Invoke the critique skill VIA THE SKILL TOOL — not by writing an
   imitation of what /critique would say:

     Skill(skill="critique", args="proposals/<slug>/index.html — focus on
     visual hierarchy, information architecture, emotional resonance,
     restraint, and whether a sophisticated B2B buyer would find this
     credible. Rate each dimension 0-10.")

3. Apply the fixes the critique returns. For each issue rated <8/10:
   - Read the specific feedback
   - Edit the relevant section of the HTML
   - Move to the next issue

4. Re-invoke /critique on the revised file. Iterate until all dimensions
   are 8+/10 OR you've completed 2 critique rounds.

5. If critique surfaces deeper template problems (broken layout,
   accessibility violations), also invoke /design-review on the live
   preview deploy in Step 8.
```

**Why this matters**: simulating /critique by pattern-matching what it might say produces shallow improvements. The actual skill has its own rubric and surfaces issues Claude wouldn't notice without invocation. Spending 30 seconds on a real Skill call is the difference between a 7/10 proposal and a 9/10 one.

**Don't skip on `--revise`** — content changes can break visual balance.

## STEP 7 — Review with the seller

Show the seller:
1. The file path: `proposals/<slug>/index.html`
2. A summary of what `/critique` flagged and how you fixed it
3. Open in browser to review (`open ...`)
4. Offer iteration: "tighten the situation," "more aggressive timeline," "swap proof points"

Do not push to deploy until the seller has read end-to-end.

If the seller hasn't already saved seller defaults, **ask now**: *"Save your company info as defaults for next time? (Y/n)"*

## STEP 8 — Deploy to Vercel

Only when the seller says "ship", "deploy", "publish".

**Preferred path**: invoke `/vercel:deploy`.

```
/vercel:deploy            # preview
/vercel:deploy prod       # production
```

**Manual fallback**:
```bash
command -v vercel || echo "Run: npm i -g vercel && vercel login"
vercel deploy --yes              # preview
vercel deploy --prod --yes       # production
```

After deploy, present the URL exactly:

```
Proposal live at:
  https://[project].vercel.app/proposals/<slug>

Send the champion this URL. They can paste it in email or Slack —
the unfurl shows a generated OG image with the project + client name.
```

## --revise mode

When invoked as `/proposal --revise <slug>`:

1. Read `proposals/<slug>/discovery.json`
2. Show the seller a section-by-section summary of current values
3. Ask: *"What sections do you want to change? List them (e.g., pricing, timeline, proof points)."*
4. Update only the named sections in `discovery.json`
5. **Snapshot current**: `mv proposals/<slug>/index.html proposals/<slug>/versions/v<current>.html` (creating `versions/` if needed)
6. Bump `project.version` (1 → 2)
7. Re-generate `index.html` from the updated JSON as v{n+1}
8. **Re-run STEP 6 critique pass** (real Skill invocation)
9. Confirm and offer to redeploy via `/vercel:deploy prod`

The URL stays stable. The champion's link still works — they get the updated content next page-load. Older versions are preserved at `proposals/<slug>/versions/v1.html`, `v2.html`, etc. for audit.

## --clone mode

When invoked as `/proposal --clone <source-slug>`:

1. Read `proposals/<source-slug>/discovery.json`
2. Show the seller the source proposal's key fields (client, project codename, price, phases, deliverables count) in a brief summary
3. Ask: *"What's different from the source? List the sections to change. Most commonly: client name, champion, situation, brand colors, proof points, dates."*
4. For each named section, ask the new values (one question per turn)
5. Generate a new `<new-slug>` from the new client name + fresh 8-char token
6. Save updated discovery to `proposals/<new-slug>/discovery.json`
7. Generate `proposals/<new-slug>/index.html` from the new discovery
8. **Run STEP 6 critique pass** on the new file
9. Standard review + deploy

The source proposal is left untouched. Cloning is the right move when you're sending similar work to a different client and 60–80% of the discovery answers carry over.

## Anti-patterns — refuse on sight

- Generating a proposal without discovery — push back, the output will fail
- Accepting B3 as paraphrased corporate-speak — go get the actual quote
- Adding "About Us" / "Our Values" / "Our Approach to Excellence" sections
- Using brand color as the page background or primary surface — accent only
- "Contact for pricing" — surface the number
- Stock imagery, gradient hero, "passionate" language
- **Skipping `/critique` or simulating it instead of invoking** — Step 6 is REQUIRED via real Skill tool call
- Adding back any active button (Accept, Share) — this skill ships read-only documents
- **Editing theme JSON files inline to "tweak this one proposal"** — themes are shared assets. If a one-off variation is needed, copy the theme to a local file and pass its path; do not mutate `themes/*.json` for a single deal.
- **Letting a theme change section structure or copy** — themes are visual-only. The 13 sections, the discovery questions, the B3 gate, the banned-word list, and the anti-patterns all stay constant across themes.
- **Picking `--theme technical` without re-testing the brand color against `#0B0F19`** — the WCAG pass in Step 3 must use the theme's `page_bg`, not always white.

## Quality bar

A v2 proposal is shipping-ready when:
- [ ] All `{{...}}` placeholders are filled (no leftover template tokens)
- [ ] Banned-word scan returns zero hits
- [ ] Buyer's actual words appear in the situation section
- [ ] Pricing is on the page (not "contact us")
- [ ] Each section ties to a specific objection it pre-empts
- [ ] `/critique` (real invocation) rates every dimension 8+/10
- [ ] Brand color passed WCAG contrast for use on its given context
- [ ] OG image renders correctly when URL is unfurled
- [ ] Mobile view at 375px doesn't break the timeline or pricing block

## Final reminder

The discovery questions are the IP. The HTML is the wrapper. Spend most of the time on Steps 2 and 6. The skill is interview + critique, not template generation.

This is a **document replacement** — not a SaaS product. Static, read-only, beautiful. Reply paths happen in email. Acceptance happens off-page.
