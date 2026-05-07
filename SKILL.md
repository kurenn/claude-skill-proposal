---
name: proposal
description: Generate beautiful single-page client proposals as a document-format replacement for PDFs and Google Docs. Persists answers, fetches brand colors from the client's site (with WCAG contrast warnings), generates a Tailwind-built single-page HTML, and runs the `/critique` design skill via the Skill tool to grade and fix the output. Includes dynamic OG images for unfurls and version-history snapshots on revise. Use when the user says "create a proposal", "client proposal", "scope of work", "SOW", or "pitch document". Use `/proposal --revise <slug>` to update an existing proposal, `/proposal --clone <slug>` to start a new one from an existing discovery, or `/proposal --meetings <url|path>,...` to draft answers from Circleback transcripts and/or local meeting notes.
argument-hint: [client name | --revise <slug> | --clone <slug>] [--meetings <url|path>,...]
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
| `/proposal --meetings <src>[,<src>...]` | New-proposal flow with discovery answers drafted from one or more transcripts. Each chunk presents the draft and waits for seller confirmation/edits before saving. |
| `/proposal "Acme Corp" --meetings <src>,<src>` | Same as above, with client name pre-filled. |
| `/proposal --revise <slug> --meetings <src>,...` | Revise existing proposal using new meeting context. Diffs the saved discovery against extracted transcript answers and asks the seller which sections to update. |

**`<src>` accepts**:
- **Circleback URL** — e.g., `https://app.circleback.ai/meetings/<id>`. Resolved via the Circleback MCP connector (`ReadMeetings` + `GetTranscriptsForMeetings`). The connector must be authorized in the user's environment.
- **Local file path** — `.txt`, `.md`, `.vtt`, `.srt`, or `.json` (Circleback export format). Use this for offline notes or transcripts from other tools.

**Combinability rules**:
- `--meetings` IS allowed with: a positional client name, `--revise <slug>`.
- `--meetings` is NOT allowed with `--clone`. If both are passed, abort with: *"`--clone` and `--meetings` are not compatible. Use `--meetings` with `--revise` to layer new context onto an existing proposal, or omit `--clone` to start fresh from transcripts."*

## The flow

```
0. Bootstrap working dir         → scaffold vercel.json, assets, api/og.tsx
1. Load seller-defaults.json     → auto-fill seller's company info (skip questions)
1.5 Transcript ingestion         → (only if --meetings) fetch + normalize sources,
                                   pre-extract candidate answers w/ provenance,
                                   flag cross-meeting conflicts
2. Discovery (3 chunks)          → Setup batch → Story (1-by-1) → Scope (1-by-1)
                                   (with --meetings: each chunk runs as
                                   draft → seller review → confirm)
3. Brand color extraction        → multi-source w/ WCAG flag, present candidates
4. Generate single-page HTML     → adapt template.html with discovery JSON
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

## STEP 1.5 — Transcript ingestion (only if `--meetings` was passed)

If the user did not pass `--meetings`, skip this step entirely and proceed to Step 2 unchanged.

### 1.5a — Resolve every source

For each comma-separated value in `--meetings`:

| Pattern | How to resolve |
|---|---|
| Starts with `http://` or `https://` and host contains `circleback` | Extract the meeting ID from the URL path. Call `ReadMeetings` with that ID for metadata (date, participants, title). Call `GetTranscriptsForMeetings` for the full transcript. |
| Resolves to an existing local file (`.txt`, `.md`, `.vtt`, `.srt`, `.json`) | Read directly. For `.vtt`/`.srt`, strip timestamps and speaker tags into plain prose. For `.json`, expect Circleback export schema (`participants`, `transcript`, `meeting_date`); fall back to dumping all string values if shape is unknown. |
| Anything else | Fail loudly for that single source — print: *"Could not resolve `<value>`. Expected a Circleback URL or a local file path (.txt, .md, .vtt, .srt, .json)."* |

If **every** source fails, abort the skill before discovery. If **some** succeed, continue with the survivors and tell the seller which were dropped.

**Circleback connector failure**: if `ReadMeetings` returns auth errors, tell the seller: *"The Circleback connector isn't authorized in this environment. Run the connector setup or paste the transcript as a local file."*

### 1.5b — Normalize into a unified extraction context

Build an in-memory list of normalized sources:

```
[
  {
    "source_type": "circleback" | "file",
    "ref": "<original url or path>",
    "meeting_date": "YYYY-MM-DD" | null,
    "participants": ["..."] | null,
    "transcript": "<plain text>"
  },
  ...
]
```

Sort by `meeting_date` ascending so "most recent" is unambiguous in conflict resolution.

### 1.5c — Pre-extract candidate answers

Read all transcripts together and extract candidate answers for every discovery field, with provenance:

| Discovery field | What to look for |
|---|---|
| `client.name`, `client.website` | Mentions of the buyer's company; URLs |
| `champion.name`, `champion.title`, `champion.email` | Speaker introductions, signature blocks, "I'm <name>, <title> at..." |
| `champion.stakeholders` | "I'll need to run this by...", "the CFO won't approve unless...", "legal will want to see..." |
| `situation.problem_in_buyer_words` | The buyer (not the seller) describing pain. **Verbatim, not paraphrased.** |
| `situation.buyer_quote` + `buyer_quote_source` | A single line that captures the problem in the buyer's voice. Must include the source meeting. |
| `outcome.metrics` | Specific numbers the buyer wants (% lift, time-to-X, $ saved). |
| `approach.phases` | Phasing the seller proposed and the buyer reacted to. |
| `deliverables.items` | Concrete artifacts named in the call. |
| `timeline.total_duration`, `milestones`, `dependencies` | Dates discussed, blockers raised. |
| `investment.model`, `tiers[].headline_price`, `payment_terms` | Pricing the seller floated and the buyer's reaction. |
| `risks` | Hesitations the buyer voiced + any mitigation the seller offered. |

For every extracted candidate, attach:
- `source`: which meeting (URL or filename) it came from
- `confidence`: `high` (direct verbatim quote), `medium` (clear paraphrase), `low` (inferred from context)
- `quote_span`: the exact transcript text (required for `situation.buyer_quote` — used as the pull quote in the template)

If a field has no candidate, leave it empty. **Do not fabricate from a single passing mention.**

### 1.5d — Conflict scan

Compare candidates across sources for these high-stakes fields:
- `investment.tiers[].headline_price`
- `timeline.total_duration` and milestone dates
- `approach.phases` (count and names)
- `champion.stakeholders` (different sets of stakeholders implies the deal shape changed)
- `client.name` and `project.codename`

If two sources disagree, record a conflict entry:

```
{
  "field": "investment.tiers[0].headline_price",
  "candidates": [
    { "value": "$48,000 fixed", "source": "circleback:...", "date": "2026-04-22" },
    { "value": "$52,000 + 10% retainer", "source": "circleback:...", "date": "2026-05-15" }
  ]
}
```

Conflicts are surfaced to the seller in Step 2, **not** auto-resolved by recency.

### 1.5e — Cross-meeting client-mismatch check

If extracted `client.name` candidates disagree across sources (different companies), stop and ask the seller: *"These transcripts reference different clients (`X`, `Y`). Which proposal are we drafting? Or did you mean to pass these as separate proposals?"* Do not continue until clarified.

### 1.5f — Persist sources for audit

When `discovery.json` is eventually written in Step 2, include a top-level `sources` array (see schema). The proposal HTML itself does **not** display source info — provenance stays in the JSON for the seller's audit trail only.

## STEP 2 — Discovery (three chunks)

### When `--meetings` was passed: confirmation-per-chunk mode

Instead of asking each question, run each chunk as **draft → seller review → confirm**:

1. Present the pre-extracted candidate answers for the chunk in a compact, scannable format. Show the `source` and `confidence` next to each value. For high-stakes fields with conflicts (from Step 1.5d), surface every candidate side-by-side and ask the seller to pick.
2. Quote `situation.buyer_quote` verbatim with its source attribution — this is the pull quote in the template, it cannot be paraphrased.
3. The seller can: **accept**, **edit a specific field**, **reject and ask normally**, or **add fields the transcript missed**.
4. For any field with no candidate (or only `low` confidence), ask the seller normally — same questions as the no-meetings flow.
5. Only when the seller confirms the chunk is correct, write its fields to `discovery.json` and proceed to the next chunk.

**The B3 gate still fires.** If no `high`-confidence buyer-language quote about the problem can be extracted from the transcripts, stop and tell the seller: *"The transcripts don't contain a clean problem quote in the buyer's own words. Paste one (or go get one). The proposal will read generic without it."* Do not let `--meetings` bypass this gate.

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
- If no website yet, default to `#0F172A` slate primary.
- Brand color is an accent, not a hero. Used only on: pull-quote left border, large outcome metrics, milestone dates, ✓ checkmarks, mitigation eyebrow, pricing headline, footer year. Everything else is grayscale.

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

### --revise + --meetings

When invoked as `/proposal --revise <slug> --meetings <src>,...`:

1. Read `proposals/<slug>/discovery.json`
2. Run **STEP 1.5** transcript ingestion on the new sources
3. Diff: for each discovery field, compare the saved value against the new candidate. Build a list of:
   - **Contradictions** — saved value disagrees with a `high`-confidence transcript candidate
   - **Extensions** — saved value is empty, new candidate has content
   - **Stable** — saved value matches or no new candidate exists
4. Present the diff section-by-section with sources, e.g.:
   ```
   Pricing
     was:  $48,000 fixed
     new:  $52,000 + 10% retainer  (circleback:..., 2026-05-15, high confidence)
     Apply? (Y / n / edit)
   ```
5. Update only the fields the seller approves
6. Append the new sources to `sources[]` in `discovery.json` (don't overwrite — append, so the audit trail spans the full revision history)
7. Snapshot, bump version, regenerate, run critique (same as standard `--revise` from step 5 onward)

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

**`--clone` is not combinable with `--meetings`.** If both flags are passed, abort with the error message in the Argument routing table. The two intents are different: clone replicates prior work; `--meetings` re-interviews from new context. To start fresh from transcripts for a similar engagement, run `/proposal --meetings <src>,...` and edit the drafts.

## Anti-patterns — refuse on sight

- Generating a proposal without discovery — push back, the output will fail
- Accepting B3 as paraphrased corporate-speak — go get the actual quote
- Adding "About Us" / "Our Values" / "Our Approach to Excellence" sections
- Using brand color as the page background or primary surface — accent only
- "Contact for pricing" — surface the number
- Stock imagery, gradient hero, "passionate" language
- **Skipping `/critique` or simulating it instead of invoking** — Step 6 is REQUIRED via real Skill tool call
- Adding back any active button (Accept, Share) — this skill ships read-only documents
- **Fabricating discovery answers from a single passing transcript mention** — when in doubt, leave the field empty and ask the seller. `--meetings` does not lower the specificity bar.
- **Letting `--meetings` bypass the B3 buyer-quote gate** — if the transcript has no clean problem quote in the buyer's voice, stop and tell the seller. Paraphrased corporate-speak still kills the proposal.
- **Auto-resolving cross-meeting conflicts by recency without asking** — surface every conflict to the seller (Step 1.5d).

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
