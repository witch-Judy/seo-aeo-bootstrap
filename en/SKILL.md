---
name: seo-aeo-bootstrap
description: Build a complete SEO/AEO (AI answer engine optimization) setup for any website from scratch - diagnose what crawlers can actually see, add server-side full-text rendering, ship the sitemap/robots.txt/llms.txt/metadata essentials, deploy a crawler probe and AI referral attribution, register Google Search Console and Bing Webmaster Tools, and monitor results in a verifiable way. Use when the user mentions SEO, AEO, GEO, indexing, getting cited by AI, "why can't Google/ChatGPT find my site", Search Console, sitemap, robots.txt, llms.txt, crawlers, optimizing a newly launched site, or wanting to see how much traffic (AI referral traffic) AI engines send - even if the user mentions only one of these pieces, load this skill and check the whole chain.
---

# SEO/AEO Bootstrap Playbook

Take a site from "invisible to engines" to "cited, attributable, and measurably working".
This playbook was applied end to end on Iriseed (iriseed.com, Next.js + Vercel + Supabase);
every code template comes from that production run.

## Mental model (read this before touching anything)

**AEO has a three-link causal chain, and each link uses a different kind of data:**

| Link | Question | Kind of data |
|---|---|---|
| ① Crawl | Did AI/search crawlers fetch the page? | **Census** - your own probe logs every fetch |
| ② Display | Did the content show up in AI answers / search results? | Only official consoles are trustworthy (GSC, Bing); everything else is sampling and guesswork |
| ③ Referral | Did anyone click through after seeing it? | **Census** - referrer/UTM logs every click |

The two ends (①③) are censuses: you can collect 100%-reliable data yourself. The middle (②) only counts official numbers.
Third-party "AEO score" products work by sampling engine APIs with a question set they picked - the questions are theirs,
answers are non-deterministic, the API is not the real user interface, and there is no control group. Don't buy a score; bracket ② between ① and ③.
Details in `references/measurement.md`.

**Two lessons from production:**
- Instrument early, not late. Iriseed had 13 signed-up users before instrumentation went live; what they read and where they got stuck is gone for good. Ship the probes before you promote, not after.
- Put UTM parameters on every outbound link you post. Otherwise traffic arrives and you can't tell where it came from.

## Phase 0 - Diagnose (10 minutes; diagnose before you prescribe)

**The single most important check: how much text does a crawler that doesn't run JS actually see?**
GPTBot, ClaudeBot, PerplexityBot, Baidu and Sogou do not execute JavaScript.
If the site is an SPA / client-side rendered, they may see nothing but a title and a one-line summary.

```bash
curl -s https://your-site.com/some-content-page | python3 -c "
import sys, re
html = sys.stdin.read()
text = re.sub(r'<script[\s\S]*?</script>', '', html)
text = re.sub(r'<[^>]+>', ' ', text)
text = re.sub(r'\s+', ' ', text)
print('Total HTML length:', len(html), '| Crawler-visible text:', len(text))
print(text[:400])
"
```

Visible text far smaller than the page's real content → the critical fix is Phase 2; everything else waits.

Other checks (does it exist, and is it any good):
- `/sitemap.xml` (covers every content page? updates automatically when content changes?)
- `/robots.txt` (accidentally blocking AI crawlers? plenty of templates default to Disallow everything)
- `/llms.txt` (the llmstxt.org convention: a site manual for AI engines)
- Are `<title>` / `description` / OG tags generated per page?
- Is there JSON-LD structured data?
- Is the site registered in GSC / Bing Webmaster Tools?

Report findings ranked as "critical → missing → could be better", not as a flat list.

## Phase 1 - The four essentials

sitemap, robots.txt, llms.txt, per-page metadata + JSON-LD.
Core principle: **generate all four from one content registry** (a database, a CMS API, or a single source of truth
like `articles.ts`), so publishing new content updates all four automatically and nothing rots. A hand-maintained list is stale by week two.

In robots.txt, explicitly welcome AI crawlers (GPTBot/ClaudeBot/PerplexityBot/Bytespider etc.; full list in code-patterns).
In llms.txt, spell out "when to cite this site" - you are teaching AI engines when they should cite you.

Code templates (mainly Next.js, with framework-agnostic principles): `references/code-patterns.md` section 1.

## Phase 2 - The critical fix: full text visible to crawlers

If Phase 0 shows the body text is not in the server-rendered HTML, this is the top priority.
Every other AEO effort is wasted until it's fixed - an engine can only cite you if it can read you.

Options, cheapest first:
1. The framework already supports SSR/SSG → move content fetching out of a client-side `useEffect` and into the server / build step.
2. Content lives inside an iframe or a client-rendered shell → have the server also render a semantic full-text copy (h1/h2/p) that is
   **word-for-word identical to the interactive version**; hide it once the client shell mounts, and show it again as a fallback if loading fails.
3. Pure static hosting where you can't change rendering → at minimum, bake the full text into the build output.

**Content parity rule**: the server version and the user version must match word for word. That's progressive enhancement,
not cloaking. Serving crawlers different content is cloaking, and it gets penalized.

Verify after the change: run the Phase 0 curl command and compare the character count against the source - they should be roughly equal
(Iriseed, measured: 192 → 4,292 visible characters, vs 4,329 in the source).

Implementation template: `references/code-patterns.md` section 2.

## Phase 3 - Probes (the self-built half of monitoring)

Three probes, all writing to your own events table (a census, no third-party dependency):

1. **Crawler probe**: middleware / edge layer identifies known crawlers by UA and asynchronously records
   `bot_hit` (crawler name, path, country). Key points: never block the response, fail silently,
   skip session logic for crawler requests. Crawlers don't run JS, so front-end analytics are completely blind to them -
   this layer has to live on the server/edge.
2. **AI referral attribution**: front-end tracking captures referrer + UTM and classifies them against an engine list
   (chatgpt.com, perplexity.ai, Kimi, Doubao (ByteDance's AI assistant) and 15+ others).
   ChatGPT's outbound links already carry `utm_source=chatgpt.com`.
3. **Owner self-exclusion**: the stats dashboard excludes the site owner by default (logged-in user_id + linked anonymous ids +
   local dev traffic). On a small site, the owner's own visits will skew the data completely.

Verify the probes right after launch: curl once with a spoofed crawler UA, confirm the row landed, then delete the test row.
Implementation templates and bot/engine lists: `references/code-patterns.md` section 3.

## Phase 4 - Official consoles

1. **Reserve verification slots in code**: metadata reads the `GOOGLE_SITE_VERIFICATION` /
   `BING_SITE_VERIFICATION` env vars (template in code-patterns section 4).
   Moving sites or accounts later means changing env vars, not code.
2. **GSC**: the user creates a "URL prefix" property at search.google.com/search-console →
   picks "HTML tag" to get the token → you store it in the hosting platform's env vars → trigger a redeploy →
   **confirm the meta tag is rendered in production** (curl | grep) → user clicks Verify → submit sitemap.xml.
   Get the order wrong and it will fail: deploy first, verify second.
3. **Bing Webmaster Tools**: prefer "Import from GSC" (one click). If the GSC property is too new, the import errors with
   "unable to fetch data" - that's normal; retry later, or add the site manually + meta tag verification.
   Bing's **AI Performance (BETA)** report = official citation data for Copilot / Bing AI surfaces.
   It's currently the only official AEO tile, but it represents Microsoft only, not all AI.

## Phase 5 - Reading the data, and cadence

- Launch day: a `/stats`-style dashboard shows crawler activity immediately; GSC/Bing data starts accumulating after 1-2 days.
- **The first signal worth reading**: which pages crawlers fetch. Those are what AI engines consider the most citable content on your site.
- Weekly: crawler fetch trend, whether AI referrals are still at zero, GSC impression queries (how people actually search their way to you).
- Blind spot: zero-click citations (AI used your content but the user didn't click). The fix is DIY sample probing -
  take the real questions each page answers, freeze the question set, ask each engine at a fixed cadence, and record whether you're cited.
  Design notes in `references/measurement.md`. Let the crawler probe run for two weeks before you start this.

## Content layer: make the content worth citing

Infrastructure decides whether you *can* be read; how you write decides whether you *get* cited.
Concrete techniques for writing and revising (answer first, self-contained paragraphs, question-style headings, specific entities,
proprietary data, E-E-A-T signals, etc.): `references/content-tips.md`.
For content sites, this matters as much as the infrastructure.

## Done checklist

```bash
# 1. Full text visible to crawlers (number should be close to the source)
curl -s https://your-site.com/content-page | python3 -c "... (Phase 0 script)"
# 2. The essentials are live
curl -sI https://your-site.com/sitemap.xml | head -1
curl -s https://your-site.com/robots.txt | head -5
curl -s https://your-site.com/llms.txt | head -5
# 3. Probe is alive (spoof the UA, check the DB, delete the row afterwards)
curl -s -A "GPTBot/1.2" https://your-site.com/content-page -o /dev/null
# 4. Verification tag is rendered
curl -s https://your-site.com/ | grep -o 'google-site-verification[^>]*'
```

All green + sitemap submitted to GSC/Bing = shipped. After that it's waiting for data and following Phase 5.
