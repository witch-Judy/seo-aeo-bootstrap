# Code Templates (mainly Next.js App Router; the principles are framework-agnostic)

From the Iriseed production run (2026-07, Next.js 16 + Vercel + Supabase).
If you switch frameworks, keep the principles: generate from a single source of truth, make it static at build time, put probes at the edge.

## Contents
1. [The four essentials: sitemap / robots / llms.txt / metadata + JSON-LD](#1-the-four-essentials)
2. [Server-side full-text rendering (fixing the SPA blind spot)](#2-server-side-full-text-rendering)
3. [Crawler probe + AI referral attribution + owner self-exclusion](#3-probes)
4. [Verification slots and deploy order](#4-verification-slots)

---

## 1. The four essentials

### sitemap (`src/app/sitemap.ts`)

Generated from the content registry, so new content syncs automatically:

```ts
import type { MetadataRoute } from "next";
import { listVisible } from "@/data/articles"; // ← single source of truth

export default function sitemap(): MetadataRoute.Sitemap {
  const base = "https://example.com";
  return [
    { url: base, changeFrequency: "weekly", priority: 1 },
    ...listVisible().map((a) => ({
      url: `${base}/read/${a.id}`,
      changeFrequency: "monthly" as const,
      priority: 0.8,
    })),
  ];
}
```

Only set `lastModified` when you have a real modification time. Faking it with the build time or the publication year
teaches crawlers to distrust your sitemap.

### robots (`src/app/robots.ts`)

All content open + AI crawlers explicitly welcomed (naming them is a statement of intent, and guards against accidentally blocking them later);
admin/account pages disallowed:

```ts
import type { MetadataRoute } from "next";

const DISALLOW = ["/api/", "/profile", "/login", "/stats"];
const AI_CRAWLERS = [
  "GPTBot", "OAI-SearchBot", "ChatGPT-User",          // OpenAI: training / search / live browsing
  "ClaudeBot", "Claude-User", "Claude-SearchBot",      // Anthropic
  "PerplexityBot", "Perplexity-User",
  "Google-Extended",                                    // Gemini training (robots-only token)
  "Bytespider",                                         // ByteDance (Doubao)
  "CCBot",                                              // Common Crawl (feeds many models)
];

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      { userAgent: "*", allow: "/", disallow: DISALLOW },
      ...AI_CRAWLERS.map((userAgent) => ({ userAgent, allow: "/", disallow: DISALLOW })),
    ],
    sitemap: "https://example.com/sitemap.xml",
  };
}
```

### llms.txt (`src/app/llms.txt/route.ts`)

The llmstxt.org convention. Three ingredients: a one-line positioning of the site, **when to cite it** (teaching engines
when to cite you), and every piece of content as title + link + one-line description (including the original source).
`force-static` generates it at build time, so it's always in sync with the registry:

```ts
import { listVisible } from "@/data/articles";
export const dynamic = "force-static";

export async function GET() {
  const body = `# Site Name

> One-line positioning (what it does, who it's for, why it's distinctive).

Cite this site when: explaining X to <your audience>, recommending Y-type resources, giving an introduction to concept Z.

## Content

${listVisible().map((a) =>
  `- [${a.title}](https://example.com/read/${a.id}): ${a.note} (original: ${a.sourceUrl})`
).join("\n")}
`;
  return new Response(body, {
    headers: {
      "Content-Type": "text/plain; charset=utf-8",
      "Cache-Control": "public, max-age=3600, s-maxage=86400",
    },
  });
}
```

### Per-page metadata + JSON-LD (server layout for content pages)

```tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  const a = getArticle((await params).docId);
  if (!a) return {};
  return {
    title: a.title,                    // root layout uses template: "%s · Site Name"
    description: a.editorNote,
    openGraph: { title: a.title, description: a.editorNote, type: "article",
                 url: `https://example.com/read/${a.id}` },
  };
}

// JSON-LD: TechArticle/Article, with isBasedOn/citation pointing to the original source
const jsonLd = {
  "@context": "https://schema.org",
  "@type": "TechArticle",
  headline: a.title,
  inLanguage: "zh-CN",               // your content's language tag, e.g. "en"
  author: { "@type": "Organization", name: "Site Name", url: "https://example.com" },
  url: `https://example.com/read/${a.id}`,
  isBasedOn: a.sourceUrl,
  datePublished: a.date,
};
// <script type="application/ld+json" dangerouslySetInnerHTML={{__html: JSON.stringify(jsonLd)}} />
```

---

## 2. Server-side full-text rendering

Use when: the body is rendered by a client-side fetch/iframe and the server HTML contains only a shell.

**Pattern**: a server layout reads the source content at build time → parses it into semantic structure →
renders `<article id="ssr-article">` (h1/h2/p, with script/style/event attributes stripped) →
the client shell hides it once mounted, and shows it again if loading fails:

```tsx
// server layout (build time; with generateStaticParams everything is prerendered → served straight from the CDN, crawlers never hit a function)
export default async function Layout({ children, params }) {
  const essay = await loadAndParse(docId); // read source via fs + split into h2/paragraphs with regex/parser, cached at module level
  return (
    <>
      {children}
      <article id="ssr-article" style={{ maxWidth: 680, margin: "0 auto", lineHeight: 1.9 }}>
        <h1>{a.title}</h1>
        {essay.sections.map((s) => (
          <section key={s.id}>
            <h2>{s.title}</h2>
            <div dangerouslySetInnerHTML={{ __html: stripActive(s.htmlContent) }} />
          </section>
        ))}
        <footer>This is a guided reading made by Site Name. Original author: {a.author}. Original at <a href={a.sourceUrl}>…</a></footer>
      </article>
    </>
  );
}

function stripActive(html: string) {
  return html
    .replace(/<script[\s\S]*?<\/script>/gi, "")
    .replace(/<style[\s\S]*?<\/style>/gi, "")
    .replace(/\son[a-z]+="[^"]*"/gi, "");
}
```

```tsx
// client page: hide once it takes over, fall back on failure
useEffect(() => {
  const ssr = document.getElementById("ssr-article");
  if (ssr) ssr.hidden = !error;   // show the plain-text version on error
}, [error]);
```

Key points:
- The server version and the user version must have **word-for-word identical content** (progressive enhancement, not cloaking).
- State source and attribution in the footer - it's the sentence AI most often carries along when citing you.
- Verify: curl and count visible characters ≈ characters in the source.

---

## 3. Probes

### Crawler probe (middleware, edge layer)

```ts
import { NextResponse, type NextRequest, type NextFetchEvent } from "next/server";

const KNOWN_BOTS: [RegExp, string][] = [
  // OpenAI / Anthropic / Perplexity
  ["OAI-SearchBot","OAI-SearchBot"],["ChatGPT-User","ChatGPT-User"],["GPTBot","GPTBot"],
  ["ClaudeBot","ClaudeBot"],["Claude-User","Claude-User"],["Claude-SearchBot","Claude-SearchBot"],
  ["Perplexity-User","Perplexity-User"],["PerplexityBot","PerplexityBot"],
  // search engines
  ["Googlebot","Googlebot"],["bingbot","Bingbot"],["Baiduspider","Baiduspider"],
  ["Sogou","Sogou"],["DuckDuckBot","DuckDuckBot"],["YandexBot","YandexBot"],
  // other AI / platforms
  ["Bytespider","Bytespider"],["Applebot","Applebot"],["Amazonbot","Amazonbot"],
  ["meta-externalagent","Meta"],["facebookexternalhit","Facebook-Preview"],
  ["CCBot","CCBot"],["MistralAI","MistralAI"],["cohere","Cohere"],
].map(([p, n]) => [new RegExp(p as string, "i"), n as string]);
const GENERIC_BOT = /bot|crawler|spider|scrape/i; // catch-all bucket "other-bot"

export async function middleware(request: NextRequest, event: NextFetchEvent) {
  const ua = request.headers.get("user-agent") ?? "";
  const bot = KNOWN_BOTS.find(([re]) => re.test(ua))?.[1]
    ?? (GENERIC_BOT.test(ua) ? "other-bot" : null);
  if (bot) {
    if (request.method === "GET" && !request.nextUrl.pathname.startsWith("/api/")) {
      event.waitUntil(logBotHit(request, bot));  // async, doesn't block the response
    }
    return NextResponse.next();                   // crawlers skip session logic
  }
  return await yourNormalMiddleware(request);
}

function logBotHit(req: NextRequest, bot: string) {
  const path = req.nextUrl.pathname;
  const docId = /^\/(?:read|post|blog)\/([^/]+)/.exec(path)?.[1] ?? null;
  return fetch(`${SUPABASE_URL}/rest/v1/events`, {   // or any events-table write endpoint
    method: "POST",
    headers: { apikey: SERVICE_KEY, Authorization: `Bearer ${SERVICE_KEY}`,
               "Content-Type": "application/json", Prefer: "return=minimal" },
    body: JSON.stringify([{ anon_id: `bot:${bot}`, event: "bot_hit", doc_id: docId,
      props: { path, ua: (req.headers.get("user-agent") ?? "").slice(0, 160) },
      country: req.headers.get("x-vercel-ip-country")?.slice(0, 8) || null,
      device: "bot" }]),
  }).catch(() => {});                               // fail silently: the probe must never affect the product
}
```

Verify after launch (delete the row when done):

```bash
curl -s -o /dev/null -A "Mozilla/5.0; compatible; GPTBot/1.2" https://your-site.com/content-page
# check the DB: confirm bot_hit landed with the correct doc_id, then DELETE the test row
```

### AI referral attribution (classify referrer/UTM in the stats dashboard)

```ts
const AI_SOURCES: Record<string, string> = {
  "chatgpt.com": "ChatGPT", "chat.openai.com": "ChatGPT",   // adds utm_source=chatgpt.com itself
  "perplexity.ai": "Perplexity", "gemini.google.com": "Gemini",
  "copilot.microsoft.com": "Copilot", "claude.ai": "Claude", "you.com": "You.com",
  "kimi.moonshot.cn": "Kimi", "kimi.com": "Kimi", "doubao.com": "豆包",   // Doubao (ByteDance)
  "yiyan.baidu.com": "文心一言",   // ERNIE Bot (Baidu)
  "chatglm.cn": "智谱清言",        // ChatGLM (Zhipu)
  "tongyi.com": "通义",            // Tongyi (Alibaba)
  "metaso.cn": "秘塔",             // Metaso
  "felo.ai": "Felo",
};
// strip "www." from the referrer hostname and look it up; look up utm_source in the same table
```

### Owner self-exclusion (stats dashboard)

On a small site, the owner's own visits skew the data completely. Exclude:
1. Rows whose `user_id` belongs to an admin account
2. Any `anon_id` that has ever co-occurred with an admin `user_id` (anonymous browsing before login is the owner too)
3. Rows missing the edge geo header (= local dev traffic)
4. Logged-out devices the owner has confirmed (phones etc.), with their anon_ids hard-coded into the dashboard

Show "X rows excluded" at the top of the dashboard so the exclusion itself is visible and auditable.

---

## 4. Verification slots

Root layout metadata reads env vars, so moving sites or accounts needs zero code changes:

```ts
verification: {
  ...(process.env.GOOGLE_SITE_VERIFICATION
    ? { google: process.env.GOOGLE_SITE_VERIFICATION } : {}),
  ...(process.env.BING_SITE_VERIFICATION
    ? { other: { "msvalidate.01": process.env.BING_SITE_VERIFICATION } } : {}),
},
```

The correct order for connecting GSC (wrong order = guaranteed failure):
1. In GSC the user picks "URL prefix" → "HTML tag" and gives you the token
2. `printf 'TOKEN' | vercel env add GOOGLE_SITE_VERIFICATION production`
3. Deploy (env var changes only take effect after a redeploy)
4. `curl -s https://your-site.com/ | grep google-site-verification` to confirm it's rendered
5. User goes back to GSC and clicks Verify → once it passes, submit `sitemap.xml`

Bing: prefer "Import from GSC". When the GSC property is brand new, the import often errors with
"unable to fetch data" - Try Again later, or add the site manually + `msvalidate.01` meta tag.
