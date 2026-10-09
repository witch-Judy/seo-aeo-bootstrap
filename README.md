# seo-aeo-bootstrap

A Claude Code skill that takes a website from "search engines and AI assistants can't see it" to "crawled, cited, attributed, and measurable" — SEO plus AEO/GEO (answer-engine optimisation) in one pass.

It is the playbook I wrote after doing this by hand on [Iriseed](https://iriseed.com), a Chinese interactive-explainer site (Next.js + Vercel + Supabase), then reused on a startup's marketing site. The code templates come from the Iriseed deployment, not from a blog post.

[中文说明](README.zh.md)

## The one check that matters most

Most AI crawlers (GPTBot, ClaudeBot, PerplexityBot) and the big Chinese search engines (Baidu, Sogou) **don't execute JavaScript**. If your content renders client-side, they see a title and a one-line description.

```bash
curl -s https://your-site.com/some-article | python3 -c "
import sys, re
h = sys.stdin.read()
t = re.sub(r'\s+', ' ', re.sub(r'<[^>]+>', ' ', re.sub(r'<script[\s\S]*?</script>', '', h)))
print('HTML:', len(h), '| text a crawler can see:', len(t))"
```

On Iriseed this printed **192 characters** for an article whose source was 4,329. After server-rendering a word-for-word semantic copy of the interactive article (progressive enhancement, not cloaking), it printed **4,292**. Nothing else in AEO matters until this number is right.

## What the skill does

| Phase | What happens |
|---|---|
| 0. Diagnose | Measure crawler-visible text, check sitemap / robots / llms.txt / metadata / JSON-LD / console setup, and rank findings as *critical → missing → nice-to-have* |
| 1. Foundations | sitemap, robots (explicitly welcoming AI crawlers), llms.txt, per-page metadata + JSON-LD — all generated from **one content registry** so they never drift |
| 2. Full text for crawlers | Server-side rendering of the real content, with a verification step |
| 3. Probes | Edge middleware that logs 23 search and AI crawlers, referral attribution for 15+ AI engines (ChatGPT, Perplexity, Gemini, Claude, Kimi, Doubao…), and owner-traffic exclusion |
| 4. Official consoles | Google Search Console and Bing Webmaster wired through env vars, in the order that actually works |
| 5. Read the data | What to look at on day 1, week 1, month 1 — and how to cover the zero-click blind spot |

Plus a content guide on writing paragraphs that AI engines can lift and cite, and a section on the Chinese ecosystem (Baidu, Bytespider/Doubao, WeChat and Xiaohongshu link previews).

## The measurement stance

Third-party "AEO visibility scores" ask a chosen set of questions to model APIs and count mentions. The questions are theirs, answers are non-deterministic, the API isn't what users see, and there's no control group.

This skill measures the two ends you *can* observe completely and squeezes the middle:

1. **Crawl** — your own probe logs every crawler hit (census)
2. **Display** — only official consoles count (GSC, Bing incl. its AI Performance report)
3. **Referral** — referrer/UTM attribution logs every click from an AI engine (census)

If an engine's crawler keeps fetching a page and its referrals are rising, it is citing you — no score needed.

## Lessons that cost real data

- Instrument **before** you promote. Iriseed had 13 sign-ups before analytics shipped; what they read is gone forever.
- Exclude yourself. On a small site the owner produced 546 of the first 777 events and flipped the conclusion ("mostly desktop" was really 99% mobile).
- Every outbound link gets a UTM, or the traffic arrives unlabelled.

## Install

```bash
git clone https://github.com/witch-Judy/seo-aeo-bootstrap
cp -R seo-aeo-bootstrap/en ~/.claude/skills/seo-aeo-bootstrap      # English
# or: cp -R seo-aeo-bootstrap/zh ~/.claude/skills/seo-aeo-bootstrap  # 中文
```

Then ask Claude Code things like *"why can't ChatGPT find my site?"* or *"set up SEO and llms.txt for this Next.js app"*. Templates target Next.js App Router; the principles (single source of truth, static generation, probes at the edge) carry over to other stacks.

## Layout

```
en/  SKILL.md + references/ (code-patterns, measurement, content-tips)   English
zh/  same, original Chinese version
```

## License

MIT
