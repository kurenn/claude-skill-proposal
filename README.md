# claude-skill-proposal

A Claude Code skill that turns a sales conversation into a beautiful, single-page client proposal — branded, restrained, deployable. Document-format replacement for PDFs and Google Docs.

```
/proposal                                    # full new flow (discovery → render → critique)
/proposal "Acme Corp"                        # pre-fill the client name
/proposal --meetings <url|path>,...          # draft discovery from Circleback transcripts or meeting notes
/proposal --revise <slug>                    # update an existing proposal, snapshot prior version
/proposal --revise <slug> --meetings <url>   # revise with new meeting context
/proposal --clone <slug>                     # start a new proposal from an existing one
```

## What it does

Walks the seller through a 3-chunk discovery interview (setup → buyer story → scope), optionally sourcing answers from Circleback transcripts or meeting notes via the `--meetings` flag. Extracts brand colors from the client's website (with WCAG contrast warnings), generates a Tailwind-built single-page HTML proposal from a canonical template, runs the `/critique` design skill to grade and fix the output, and deploys to Vercel.

The proposal itself is **read-only** — there are no live buttons, no acceptance tracking, no analytics. The seller emails or pastes a URL; the buyer reads it like a beautifully-typeset PDF and replies through the existing email thread.

Designed around one principle: **a proposal does not close a deal — the conversation does**. The proposal is the artifact your champion hands to people who weren't in the room (CFO, skeptical co-founder, procurement). Every section earns its place by pre-empting an objection the champion will face.

### New in this version

- **Transcript ingestion** — pass `--meetings <circleback-url|local-file>,...` to draft discovery answers from prior conversations instead of re-interviewing the seller end-to-end. Supports Circleback URLs (via the MCP connector) and local meeting notes (`.txt`, `.md`, `.vtt`, `.srt`, `.json`). Surfaces cross-meeting conflicts for the seller to resolve. Works standalone or with `--revise` to layer new context onto existing proposals.

## Install

The skill lives at `~/.claude/skills/proposal/`. Two ways to install:

**Clone (recommended — easy to update):**

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/kurenn/claude-skill-proposal.git ~/.claude/skills/proposal
```

**Or download a tarball:**

```bash
mkdir -p ~/.claude/skills/proposal
curl -L https://github.com/kurenn/claude-skill-proposal/archive/refs/heads/main.tar.gz \
  | tar -xz --strip-components=1 -C ~/.claude/skills/proposal
```

Restart Claude Code (or start a new session) and the skill will appear as `/proposal` in your available skills list.

## Update

```bash
cd ~/.claude/skills/proposal && git pull
```

Your local `seller-defaults.json` (gitignored) and any generated proposals in working directories outside the skill are untouched.

## Dependencies

| Dependency | Required? | Purpose |
|---|---|---|
| [Claude Code](https://claude.com/claude-code) | Required | The harness that runs skills |
| [impeccable.style](https://impeccable.style) — `/critique` skill | **Required** | Step 6 of the flow invokes the `/critique` skill via the Skill tool to grade visual hierarchy, information architecture, and emotional resonance, then iteratively applies fixes. Without it, the skill will fall back to a manual rubric, but the output quality drops noticeably. Install via [impeccable.style](https://impeccable.style). |
| Circleback MCP connector | Optional | Required only if using `--meetings` with Circleback URLs. If you only pass local file paths (`.md`, `.txt`, `.vtt`, `.srt`, `.json`), no connector is needed. |
| [Vercel CLI](https://vercel.com/docs/cli) | Optional | For one-command deploy of generated proposals. Install with `npm i -g vercel` |
| Node.js + Tailwind CSS | Optional | Only needed if you edit `template.html` and want to rebuild `vercel-starter/assets/tailwind.css`. The compiled CSS is shipped — most users never need this. |

The flow gracefully degrades: without Vercel CLI you can still generate proposals locally; without `/critique` you can still ship, just with weaker design quality.

## Customizing

| File | What it controls |
|---|---|
| `template.html` | The proposal's HTML structure, sections, and Tailwind classes |
| `tailwind-input.css` | Custom CSS layered on top of Tailwind base |
| `examples/*.html` | Reference proposals across industries (SaaS, agency, services). Read these to anchor your taste before editing the template. |
| `vercel-starter/vercel.json` | Deploy config: clean URLs, security headers, asset caching |
| `vercel-starter/api/og.tsx` | Dynamic 1200×630 Open Graph image for Slack/email unfurls |
| `discovery-schema.json` | Canonical schema for `proposals/<slug>/discovery.json` — what the discovery interview captures, including an audit trail of transcript sources (for `--meetings`) |

### Rebuilding the precompiled CSS

If you edit `template.html` and add Tailwind utilities not already present in the compiled bundle, regenerate it:

```bash
cd ~/.claude/skills/proposal
npm install
npx tailwindcss -i tailwind-input.css -o vercel-starter/assets/tailwind.css --minify
```

## How the discovery interview works

The skill asks **three chunks** of questions, one at a time within each chunk:

1. **Setup** (one batch) — client, industry, project codename, your company info (auto-loaded from `seller-defaults.json` after the first run)
2. **Story** (one question per turn) — champion, stakeholders, the buyer's exact words about their problem, the outcome they're buying, hesitations
3. **Scope** (one question per turn) — phased approach, deliverables, timeline, pricing, proof points, risks, next-step

Answers persist to `proposals/<slug>/discovery.json` so you can `--revise` later without re-interviewing.

## Quality bar

A shipping-ready proposal hits all of these:

- All template placeholders filled (no leftover `{{...}}` tokens)
- Banned-word scan returns zero hits (no "synergy", "leverage", "passionate about", etc.)
- The buyer's actual words appear in the situation section (even when sourced from `--meetings` transcripts)
- Pricing is on the page (not "contact us")
- Each section ties to a specific objection it pre-empts for the champion
- `/critique` rates every dimension 8+/10
- Brand color passes WCAG AA contrast on its given context (the extractor flags this)
- Mobile view at 375px doesn't break the timeline or pricing block
- If `--meetings` was used, cross-meeting conflicts are surfaced and resolved by the seller

## License

MIT. See [LICENSE](LICENSE).

## Credits

The discovery question structure was sketched in collaboration with the [`/meditate`](https://github.com/anthropics/claude-code/tree/main/skills) skill across 5 layered analysis passes. The `/critique` design pass uses [impeccable.style](https://impeccable.style) skills.
