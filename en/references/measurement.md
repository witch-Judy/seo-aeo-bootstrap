# Measurement: What to Trust, What Not to, and How to Verify It Yourself

## Why third-party "AEO scores" can't be trusted

Every AEO tool on the market (visibility scores, "up X% after optimization") works the same way:
**pick a batch of questions, periodically ask them through each AI engine's API, and count how often your brand/domain shows up in the answers.**
It's a sampled survey, not backend data from any engine. OpenAI, Anthropic and Perplexity
have no public publisher console - anyone claiming "official AEO data" for them is making it up.

Four weaknesses that can't be fixed:
1. **They pick the questions** - choose different questions and you get a different score.
2. **Answers are non-deterministic** - the same question gets a different answer each time, so the score jitters by nature.
3. **API ≠ real interface** - login state, memory and region all change what real users see.
4. **No control group** - "up Y% after optimization" can't distinguish the optimization working from a model update or sampling noise.

## Trustworthy data comes in three layers

| Layer | Data | Kind | Source |
|---|---|---|---|
| Crawl | Which crawler, which page, when | Census | Self-built bot probe (see code-patterns section 3) |
| Display | Impressions/clicks/queries, citations on AI surfaces | Official | GSC "Performance"; Bing "Search Performance" + "AI Performance (BETA)" |
| Referral | Real visits sent by AI engines | Census | referrer/UTM attribution (chatgpt.com adds utm_source itself) |

**How to read Bing's numbers**: AI Performance covers only Microsoft surfaces (Copilot, Bing AI).
It's currently the only official AEO tile, but it's just Microsoft's piece, not the whole wall.
ChatGPT used to rely on Bing's index; OpenAI now has its own crawler and index (OAI-SearchBot),
so Bing's numbers are no longer a proxy for ChatGPT.

**Bracketing**: when an engine has no official numbers - if its crawler fetches you frequently (①) and its referrals
are growing (③), confidence that it's citing you is high. Nobody needs to certify it.

## Blind spot: zero-click citations

The AI told the user what your content says, but the user didn't click the link - the referral probe can't see it.
This is the only part that requires actively asking, and it's the reason paid tools exist.
But self-testing a single site is more faithful than any tool, because you know better than anyone what to ask.

### Designing DIY sample probing

- **Question set**: 1-2 questions per page that the page genuinely answers (in your target readers' own words,
  not SEO keywords). Once set, freeze it - changing questions = changing the ruler, and the trend becomes meaningless.
- **Engine set**: ChatGPT (with search on), Perplexity, Gemini, Kimi, Doubao, and whatever else you're targeting.
- **Cadence**: once a week, same day, same time.
- **Record**: for each (question × engine), three values: did it mention your domain/brand,
  did it give a link, and what position among the sources. Store it in a table and chart the trend.
- **Interpretation discipline**: this is still sampling - single-week swings mean nothing; only look at trends over 4+ weeks.
  Sample size = questions × engines; when it's small, don't draw conclusions.
- **Timing**: let the crawler probe run for two weeks first. If crawlers haven't fetched you yet, probing will be all zeros - wasted effort.

## When to read what

- **Day 0**: probes are visible immediately. Spoof a UA once to confirm they're alive.
- **Day 1-2**: GSC/Bing start showing data.
- **First signal**: which pages crawlers fetch = what AI engines consider your most citable content,
  and a guide for what to write next.
- **Weekly**: crawler trend, whether AI referrals have broken zero, GSC queries (how people actually search their way to you).
- **Monthly**: DIY probing trend; does the content getting cited match the content you want cited?

## Lessons log (Iriseed, 2026-07)

- What the 13 users who signed up before instrumentation went live actually looked at can never be recovered - **probes before promotion**.
- The owner accounted for 546 of 777 events; without excluding them you'd reach the opposite conclusion
  ("mostly desktop" was a dev-traffic artifact; real traffic was 99% mobile).
- A Google referrer can also be the owner - device-level exclusion has to be confirmed, not inferred from login state alone.
