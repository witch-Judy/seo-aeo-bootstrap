# Content-Layer Tips: Make Your Content Worth Citing

Infrastructure (SKILL.md Phases 1-4) decides whether engines **can read** you; this file is about whether they **want to cite** you.
Use it as a checklist when writing new content or revising old content.

## AEO: write to be excerpted

AI engines excerpt at the **paragraph** level. A great article as a whole doesn't help; you need paragraphs that work on their own when lifted out.

1. **Answer first.** The first paragraph of every H2 section states the conclusion/definition directly; explanation and setup come after.
   When an AI retrieves that section, the first paragraph is what it lifts - make it self-contained, independent of context
   (references like "it" or "as mentioned above" break the excerpt).
2. **Question-style headings.** Use the exact questions your readers actually ask as H2/H3s ("How does a diffusion model remove noise?"),
   not noun phrases ("Denoising mechanism"). AI retrieval is fundamentally question-to-question matching.
3. **One idea per paragraph.** Split anything longer than 5 lines. A paragraph that mixes two ideas can't be excerpted for either.
4. **Concrete entities and numbers.** "Fast" → "about 75 seconds"; "lots of users" → "43 unique visitors".
   Sentences with specific numbers, dates and named entities are far more likely to be cited than sentences built on adjectives.
5. **Only proprietary material forces a citation.** First-party data, your own experiments, a distinctive methodology, original charts -
   an AI paraphrasing common knowledge doesn't need to cite anyone, but paraphrasing "facts only you have" means it has to name you.
   This is the highest-leverage item in the content layer.
6. **Structured facts.** Use tables and lists for anything comparative or enumerable; AI engines excerpt structured content
   more accurately and cite it more often.
7. **Bylines and sources.** State author/organization, date and references clearly (E-E-A-T signals).
   Link to real sources when you cite others - engines treat you the way you treat sources.
8. **Use llms.txt to teach engines how to use you.** Write "Cite this site when: ..." explicitly.
   Put a one-line self-description in the page footer ("This is a Y made by X; original at Z") -
   AI often carries that sentence along verbatim, which amounts to a built-in ad slot.

## SEO: write to be found

1. **Title = who it's for + what they get.** "Transformers Explained" loses to
   "For people who never took linear algebra: work through attention by hand". Audience words are your long-tail search terms.
2. **One search intent per page.** One article answers one question. To cover a second question, write a second article
   and interlink them. Mixing intents on one page dilutes the ranking for each.
3. **Long tail first.** A new site can't beat big sites on head terms ("AI tutorial"); specific questions
   ("What do RLHF annotators actually do?") have less competition, sharper intent and better conversion. Build up the long tail before going after head terms.
4. **Descriptive internal anchor text.** "Click here" carries no information; "interactive explanation of attention"
   tells both users and engines what's on the other end. Interlink related content into a web so authority flows within the site.
5. **Don't leave og:image empty.** Shares on social/messaging apps without an image get dramatically fewer clicks - even more so
   in the Chinese ecosystem (WeChat, Xiaohongshu (RedNote)). A single site-wide default brand image beats nothing.
6. **Update old content.** When you make substantive changes, update dateModified. Pages that are kept up to date
   generally outrank one-and-done posts.
7. **Speed and mobile.** Real traffic to Chinese-language content sites is almost entirely mobile (Iriseed measured 99%).
   Mobile above-the-fold beats everything on desktop.
8. **Backlinks are still the strongest signal.** Citable assets (interactive explainers, original data, tools)
   naturally attract backlinks - the flip side of AEO item 5.

## China ecosystem notes

- **Baidu and Sogou (which powers WeChat Search) don't execute JS** - server-side full-text rendering is just as critical for Chinese SEO.
- **ByteDance's Bytespider is what feeds Doubao** (ByteDance's AI assistant), the biggest source of Chinese AI citations. Don't block it in robots.txt.
- In-app sharing previews on Xiaohongshu (RedNote) and WeChat depend on OG tags (their crawlers don't execute JS either), so og:title/og:image
  must be in the server-rendered HTML.

## Anti-patterns (penalized or wasted effort)

- Showing crawlers different content than users (cloaking) - the server-side full text must match the user version word for word.
- Keyword stuffing, hidden keyword text - stopped working in the 2010s; now it's a net negative.
- Buying backlinks, link-exchange schemes.
- Passing off build time as lastModified - crawlers learn to distrust your sitemap.
- Mass-generating hundreds of thin pages at once - in the AI era engines are even more sensitive to that "bulk-generated" feel;
  few and distinctive beats many and mediocre.
