# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page DJ link-in-bio site for Shray ("shrayway", Chicago), live at **https://shrayway.com**. It replaces a Linktree. The audience is overwhelmingly **phones arriving from his Instagram bio** — that constraint drives most decisions here (page weight, tap targets, small-screen testing).

The page has three jobs: sell/RSVP the next show, show off recorded live sets, and look clean without reading as corporate or corny.

**No build step, no dependencies, no framework, no tests.** Three files do everything:

| File | Role |
|---|---|
| `public/index.html` | The whole template — CSS and a vanilla-JS renderer in one file |
| `public/config.json` | All content. The only file that changes day to day |
| `public/banner.jpg` | Header photo (plus generated icons and `og.jpg` beside it) |

`PLAN.MD` at the repo root is the fuller spec and decision log. **It is gitignored on purpose** — it holds candid notes and unannounced-gig details, and the owner chose to keep it out of the public repo. Read it for context; don't commit it.

## Commands

```bash
# Local preview — MUST be served over http.
# file:// blocks fetch('config.json') and the page renders nothing.
cd public && python3 -m http.server 8000      # → http://localhost:8000

# Validate content before committing (a bad comma breaks the whole page)
python3 -m json.tool public/config.json

# Syntax-check the inline template script
python3 -c "import re;h=open('public/index.html').read();s=[x for x in re.findall(r'<script[^>]*>(.*?)</script>',h,re.S) if x.strip()][0];open('/tmp/chk.js','w').write(s)" && node --check /tmp/chk.js

# Confirm the deploy would upload only the site files (see "public/" below)
npx wrangler@4 deploy --dry-run    # expect "Read N files from .../public"

# Deploy: push to main. Cloudflare builds automatically, ~30s.
git push

# Verify a deploy actually landed (live should equal local byte-for-byte)
curl -sL https://shrayway.com/ | md5 ; md5 -q public/index.html
```

## Architecture

`index.html` fetches `config.json` at runtime and builds the DOM in JS. There is no server, no templating, no hydration — just `render(config)` appending elements.

**The gigs array is the single source of truth.** A gig is added once; the template decides whether and where it appears. Nothing is ever moved between sections by hand.

Every gig has a **cutoff: 5:00 AM local, the day after its `date`** (`CUTOFF_HOUR` in `index.html`). This is deliberate and must not be "fixed" to midnight: sets run past midnight, and a 2 AM set belongs to the night before. Tickets stay buyable mid-show.

| Condition | Where it appears |
|---|---|
| Before cutoff, `pinned: true` | "up next" — a full hero ticket card |
| Before cutoff, nothing pinned | the soonest gig becomes the lone hero |
| Before cutoff, not the hero | "up next" — compact row |
| Past cutoff, has `setUrl` | **live sets** (YouTube thumbnail auto-pulled) |
| Past cutoff, no `setUrl` | **nowhere** — it drops off the site |

There is no "past gigs" résumé section by design. A recording is the only thing that keeps a past gig on the page.

### Gig fields

Only `date` and `venue` are required.

| Field | Effect |
|---|---|
| `date` | `YYYY-MM-DD`. Drives all placement. For a gig running past midnight, use the date the night *starts* |
| `title` | Shown big; `venue` drops to the meta line. Omit and the venue is the headline |
| `venue`, `meta` | Supporting detail line |
| `doors` | e.g. `"8PM"` → renders `DOORS 8PM` on the ticket stub |
| `setTime` | When he's actually on. Separate from `doors` because it's usually known only the week of — a one-field late addition |
| `price` | `"free"` → green tag + "rsvp" CTA; anything else → "tickets" |
| `cta` | Overrides the derived CTA label |
| `url` | Ticket/RSVP link |
| `setUrl` | Recording. Promotes a past gig into **live sets** |
| `pinned` | Full hero card. **Multiple pinned gigs each get one at equal height** — for co-headline weekends |
| `series` | When *every* upcoming gig shares it, the name appears as an uppercase tag beside "up next" and cards get "night one / night two" instead of "next show / coming up". Reverts automatically once they pass |
| `teaser` | Booked but unannounced: renders the date only, name behind a redaction bar. The hidden text never enters the DOM, so it can't be read from page source. A teaser is never the hero |

`config.city` is **no longer read** by the template (the banner's city marker was removed) — it's a leftover key.

## Constraints that will bite you

These are all deliberate and were each arrived at the hard way.

- **`public/` exists so Wrangler doesn't publish `.git`.** Wrangler uploads its assets directory verbatim, and Cloudflare clones the repo before deploying. Pointing `assets.directory` at the repo root would serve the entire commit history at `shrayway.com/.git/`. Never move the site files back to root; run the dry-run above if you touch `wrangler.jsonc`.
- **`.pinned` must keep `padding: 0`.** The hero is a ticket whose date stub sits flush against the card edge. A stale small-screen rule once re-added padding and detached the stub on phones.
- **No gradients on the ticket card.** An earlier gradient wash read as a generic AI-generated component. Flat fills only.
- **`<title>`, description and all `og:`/`twitter:` tags are static in `<head>` — never set them from JS.** Crawlers (iMessage, WhatsApp, Instagram) don't run JavaScript, so scripted tags produce a blank share preview. Scripting them also silently overwrites hand edits. `og:image` must stay an **absolute** URL.
- **No `<button>` or `<a>` nested inside another `<a>`.** The hero card and some contact rows are themselves links. Adding a nested interactive element (a copy button, a "venue → maps" link) requires restructuring the row or using a stretched-link overlay.
- **Tap targets are 44px minimum.** Social icons and copy buttons hit this via padding/`::after` overlays that don't change their visual size.
- **Animations start at `opacity: 0`.** A global `prefers-reduced-motion` block forces `opacity: 1 !important` so content can never be stranded invisible.
- **Text contrast was measured, not eyeballed.** All styles pass WCAG AA against the *composited* violet-on-ink backgrounds. `--bone-faint` is tuned to the edge of that; darkening it will fail the smallest type.

## Deploy pipeline

GitHub `shraysrivastava/shrayway` (public) → Cloudflare **Workers** (not Pages) via `wrangler.jsonc` → **shrayway.com** (domain registered at Cloudflare, custom domain attached to the Worker).

- Pushing to `main` triggers the deploy. No manual step.
- `shrayway.com/index.html` returns empty — Cloudflare canonicalizes to `shrayway.com/`. Always verify against the root URL.
- `www.shrayway.com` is **not attached** — only the apex resolves.

## Verifying UI changes

There's no test suite, so changes are verified by rendering. What has worked:

- Headless Chrome screenshots: `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu --no-sandbox --hide-scrollbars --window-size=390,900 --virtual-time-budget=9000 --screenshot=out.png http://localhost:8000/`
- **Chrome pins a 500px layout viewport for `--dump-dom` and ignores `--window-size`.** To test real phone widths, load the page in fixed-width `<iframe>`s from a temporary harness page in `public/` (delete it afterward) and measure via `contentDocument`.
- Always check **320px** as well as 390/460 — that's where wrapping and the stub width break first.
- Screenshots taken too early look blank because of the staggered fade-ins. Confirm against the rendered DOM before concluding a deploy failed.

## Working style

Content changes come through Claude, not a CMS — the owner asks rather than editing JSON himself, and self-serve editing tooling has been explicitly rejected. Direction is consistently **toward simplification**: a "past gigs" section, a site-wide Halloween theme, a decorative barcode and an "admit one" rail were all built and then deliberately removed. Prefer cutting over adding.
