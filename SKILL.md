---
name: domain-finder
description: Verify domain availability in real time via Fastly's Domain Research API across every TLD — and surface aftermarket prices when a domain is squatter-held. Triggers on any brand-name, domain, URL, "is X.com taken," rebrand, or naming question. Never claim a domain is available without running the verified check — that's the moat.
---

# Domain Finder

You help the user find a brand/domain name that is **actually available right now**, not hallucinated. The Fastly Domain Research API in `check.mjs` is the verified source of truth. It also returns aftermarket pricing when a domain is held by a squatter (HugeDomains, Sedo, etc.), which means you can give the user both a "free to register" list *and* a "for sale at $X" list — that's a unique capability versus other AI naming tools.

## Path to the script

The skill installer drops `check.mjs` next to this `SKILL.md`. The default install location is `~/.claude/skills/domain-finder/check.mjs`. If that path isn't there (project-local install), run `find ~/.claude . -name check.mjs -path '*domain-finder*' 2>/dev/null | head -1` once to locate it.

## Setup check (run once per fresh user)

If `FASTLY_KEY` is missing the script fails with a clear error pointing the user to the setup steps. Tell them:

```bash
export FASTLY_KEY=<their-fastly-api-key>     # in ~/.zshrc for persistence
```

Get one at: <https://manage.fastly.com/account/personal/tokens> (free tier works).

Or leave it in `.env.local` / `.env` in the working directory (or any parent dir) — the script auto-loads.

## Workflow

### 1. Get the brief

If the user gave a brief, parse it. If not, ask 1–3 short questions (use `AskUserQuestion` when available):

- **Audience** (developers, marketers, SMB owners, kids, B2B, etc.)
- **Vibe** (descriptive vs brandable; playful vs professional; abstract vs concrete)
- **Constraints** (must have `.com`? must include "AI"? no `-ly` suffix? no fake-foreign suffixes?)
- **Examples of names they like/dislike** if available

Don't over-interrogate. 2 questions max if you have to ask at all.

### 2. Generate candidates

Brainstorm 30–80 candidates **before** any availability check. Spread across categories so you can learn the user's taste from their reactions:

- **Descriptive compound** (`mascotbuilder`, `charactermaker`, `mintforge`)
- **Single-word abstract** (`forge`, `castable`, `guise`)
- **Invented / portmanteau** (`castyra`, `mascotello`, `heronix`)
- **TLD hacks** (`mascot.to`, `for.ge`, `char.it`) — only when the split reads naturally as the word
- **Short 4–5 letter brandable** (`vixa`, `zexa`, `ruvi` — Pika/Sora-tier)

Pick TLDs to test based on the brief:
- AI tool → always include `.ai` plus `.app` + `.studio`
- General SaaS → `.com` + `.app` + `.io`
- Marketing/agency → `.com` + `.studio` + `.co`

### 3. Verify with the script

Write candidates to a temp file (one per line, `#` comments allowed), then:

```bash
node ~/.claude/skills/domain-finder/check.mjs --file /tmp/candidates.txt --json
```

Or pipe directly:

```bash
echo "foo.com
bar.ai
baz.studio" | node ~/.claude/skills/domain-finder/check.mjs --json
```

Parse the JSON, group by `category`:
- `"category": "available"` — free to register **(safe to present as available)**
- `"category": "for_sale"` — squatter-held, has `offers[]` with pricing
- `"category": "registered"` — in active use
- `"category": "error"` — retry or skip

**Never claim a name is available unless `category === "available"` in this session's JSON output.**

### 4. Present results

Use a master-list format with TLDs as columns next to each base name, plus a "Top picks" section at the top. Surface aftermarket prices for popular names — they're useful intel even if expensive:

```
🏆 Top picks
- mascotbuilder.ai  (descriptive + AI signal, on-narrative)
- castcraft.studio  (brandable + broader than "mascot")

✓ Available
mascotbuilder    .studio  .app  .ai
charactermaker   .studio  .app
...

$ For sale on aftermarket (cheapest first)
| Domain          | Price       | Vendor       |
|-----------------|-------------|--------------|
| aimascot.com    | USD 2,495   | HugeDomains  |
| brandmascot.com | USD 1,200   | Sedo         |
```

For each top pick, give a one-line justification tied to the user's brief.

### 5. Iterate

The user's reaction is the most valuable signal. When they react:

- **"Too X"** → drop the X-tagged candidates, regenerate with that constraint
- **"I like [name]"** → save it as a north star; future suggestions should pattern-match
- **"`X.com` is actually taken"** → trust them, drop X, update mental model

Persist the running constraints **in this conversation** (no DB needed). Re-state them before each new candidate batch so the user can correct.

## Pre-flight (optional, when user is near-committed)

When the user has narrowed to 1–3 finalists, offer to pre-flight:

1. **SERP collision** — `WebSearch` `"<name>" company product app` to check for active brands using the same name
2. **`<name>.com` redirect check** — Fastly's API already tells you if it's for-sale and what the price is; cross-check with a `WebFetch` to see what landing page is served
3. **Social handles** — `WebFetch` `instagram.com/<name>`, `github.com/<name>`, `twitter.com/<name>` for handle availability
4. **Trademark** — point them to <https://tmsearch.uspto.gov> and <https://www.euipo.europa.eu> — actual trademark search is hard to automate cleanly

Skip pre-flight on early candidates — it's wasted effort until the user has narrowed.

## Hard rules

- **NEVER fabricate availability.** If you haven't run the script for a name in this session, you don't know its status. Run it.
- **`for_sale` ≠ available.** Surface the price honestly. A $2,495 domain is buyable but not "free to register" — the user should know the difference.
- **Respect rejections.** If a user says "no -ly suffix," don't try to slip one in three rounds later.
- **Don't over-search.** If `.com` is taken (or for-sale aftermarket) across the first 25 short brandable candidates, stop trying — short `.com`s are basically all gone. Pivot to other TLDs.
- **The check is the moat.** Every name you present as available must have a `category: "available"` entry in the most recent script output.

## Script flag reference

```
node check.mjs <domain>...                  # check positional args
node check.mjs --file <path>                # check from file
echo "..." | node check.mjs                 # check from stdin
node check.mjs --json                       # JSON output instead of human
node check.mjs --available-only             # only print free
node check.mjs --delay 400                  # ms between calls (default 250)
```

Rate-limit aware: 5× retry on 429 honoring the `retry-after` header. Default 250 ms pacing handles batches of 100+ cleanly.

## Note on agent portability

`AskUserQuestion`, `WebSearch`, and `WebFetch` are Claude-Code-specific tools used in the workflow above. The core `check.mjs` script is a plain Node ESM CLI — it works in any agent harness (or any shell) that can shell out. Drop the Claude-specific UX steps if you're wiring this into a different agent.
